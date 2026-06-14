# 存储后端插件抽象与方法分发机制

## 概述

Filestash 通过插件化架构支持多种存储后端（Local、S3、FTP、Dropbox 等）。所有存储后端实现同一组 `IBackend` 接口，通过统一的注册和分发机制接入主程序。

## 核心架构分层

```
┌─────────────────────────────────────────────────────────┐
│                    HTTP 请求层                           │
│  (ctrl/files.go: FileLs, FileCat, FileSave, ...)        │
└───────────────────────────┬─────────────────────────────┘
                            │ ctx.Backend.Ls() / Cat() / ...
                            ▼
┌─────────────────────────────────────────────────────────┐
│                    中间件层                              │
│  (middleware/session.go: SessionStart)                  │
│  - _extractBackend() → model.NewBackend()               │
└───────────────────────────┬─────────────────────────────┘
                            │ Backend.Get(type).Init()
                            ▼
┌─────────────────────────────────────────────────────────┐
│                    驱动注册层                            │
│  (common/backend.go: Driver)                            │
│  - map[string]IBackend                                  │
│  - Register() / Get()                                   │
└───────────────────────────┬─────────────────────────────┘
                            │
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
        ┌───────┐       ┌───────┐       ┌───────┐
        │ Local │       │  S3   │       │  FTP  │  ...
        └───────┘       └───────┘       └───────┘
        (plg_backend_*)
```

## 1. 接口定义

### IBackend 接口

所有存储后端必须实现 `IBackend` 接口，定义于 [server/common/types.go:13-24](server/common/types.go#L13-L24)：

```go
type IBackend interface {
    Init(params map[string]string, app *App) (IBackend, error)
    Ls(path string) ([]os.FileInfo, error)
    Stat(path string) (os.FileInfo, error)
    Cat(path string) (io.ReadCloser, error)
    Mkdir(path string) error
    Rm(path string) error
    Mv(from string, to string) error
    Save(path string, file io.Reader) error
    Touch(path string) error
    LoginForm() Form
}
```

### 空实现（Nothing）

当后端不存在或不允许访问时，返回 `Nothing` 空实现，定义于 [server/common/backend.go:40-79](server/common/backend.go#L40-L79)：

```go
type Nothing struct{}

func (b Nothing) Ls(path string) ([]os.FileInfo, error) {
    return []os.FileInfo{}, nil
}
func (b Nothing) Cat(path string) (io.ReadCloser, error) {
    return NewReadCloserFromReader(strings.NewReader("")), ErrNotImplemented
}
// ... 其他方法均返回 ErrNotImplemented
```

## 2. 插件注册机制

### 2.1 插件导入入口

所有后端插件通过 `server/plugin/index.go` 统一导入，利用 Go 的 `init()` 函数自动注册：

```go
// server/plugin/index.go:3-46
import (
    _ "github.com/mickael-kerjean/filestash/server/plugin/plg_backend_local"
    _ "github.com/mickael-kerjean/filestash/server/plugin/plg_backend_s3"
    _ "github.com/mickael-kerjean/filestash/server/plugin/plg_backend_ftp"
    // ... 约 20+ 种后端
)
```

### 2.2 插件自注册

每个后端插件在自己的 `init()` 函数中调用 `Backend.Register()` 注册：

**示例：Local 后端** [server/plugin/plg_backend_local/index.go:11-13](server/plugin/plg_backend_local/index.go#L11-L13)
```go
func init() {
    Backend.Register("local", &Local{os.Getenv("LOCAL_BACKEND_SECRET")})
}
```

**示例：S3 后端** [server/plugin/plg_backend_s3/index.go:37-39](server/plugin/plg_backend_s3/index.go#L37-L39)
```go
func init() {
    Backend.Register("s3", S3Backend{})
    S3Cache = NewAppCache(2, 1)
}
```

### 2.3 驱动注册中心

`Driver` 结构体维护所有已注册的后端，定义于 [server/common/backend.go:17-38](server/common/backend.go#L17-L38)：

```go
type Driver struct {
    ds map[string]IBackend
}

func (d *Driver) Register(name string, driver IBackend) {
    if driver == nil {
        panic("backend: register invalid nil backend")
    }
    d.ds[name] = driver
}

func (d *Driver) Get(name string) IBackend {
    b := d.ds[name]
    if b == nil || name == BACKEND_NIL {
        return Nothing{}
    }
    return b
}
```

全局单例：
```go
var Backend = NewDriver()
```

## 3. 后端实例化流程

### 3.1 NewBackend 工厂函数

`model.NewBackend()` 负责根据 session 创建后端实例，定义于 [server/model/files.go:9-51](server/model/files.go#L9-L51)：

```go
func NewBackend(ctx *App, conn map[string]string) (IBackend, error) {
    // 安全检查：确保连接参数在配置允许的范围内
    isAllowed := func() bool {
        possibilities := make([]map[string]interface{}, 0)
        for i := 0; i < len(Config.Conn); i++ {
            d := Config.Conn[i]
            if d["type"] != conn["type"] {
                continue
            }
            // 检查 hostname、path、url 等参数匹配
            // ...
            possibilities = append(possibilities, Config.Conn[i])
        }
        return len(possibilities) > 0
    }

    if isAllowed() == false {
        return Backend.Get(BACKEND_NIL), ErrNotAllowed
    }
    
    // 关键：通过类型获取后端模板，调用 Init() 创建实例
    return Backend.Get(conn["type"]).Init(conn, ctx)
}
```

### 3.2 Init 方法的作用

每个后端的 `Init()` 方法负责：
1. 验证连接参数（密码、密钥等）
2. 创建并返回一个**新的后端实例**（包含连接状态）

**示例：Local 后端 Init** [server/plugin/plg_backend_local/index.go:19-29](server/plugin/plg_backend_local/index.go#L19-L29)
```go
func (this Local) Init(params map[string]string, app *App) (IBackend, error) {
    if err := bcrypt.CompareHashAndPassword(
        []byte(Config.Get("auth.admin").String()),
        []byte(params["password"]),
    ); err == nil {
        return &Local{}, nil
    }
    return nil, ErrAuthenticationFailed
}
```

**示例：S3 后端 Init** [server/plugin/plg_backend_s3/index.go:42-79](server/plugin/plg_backend_s3/index.go#L42-L79)
```go
func (this S3Backend) Init(params map[string]string, app *App) (IBackend, error) {
    // 配置 AWS 凭证、区域、端点等
    creds := []credentials.Provider{
        &credentials.StaticProvider{...},
        &credentials.EnvProvider{},
        // ...
    }
    config := &aws.Config{
        Credentials: credentials.NewChainCredentials(creds),
        Region:      aws.String(region),
    }
    return &S3Backend{
        config: config,
        params: params,
        app:    app,
    }, nil
}
```

## 4. 方法分发流程

### 4.1 中间件注入 Backend

请求经过 `SessionStart` 中间件时，`ctx.Backend` 被初始化，定义于 [server/middleware/session.go:57-82](server/middleware/session.go#L57-L82)：

```go
func SessionStart(fn HandlerFunc) HandlerFunc {
    return HandlerFunc(func(ctx *App, res http.ResponseWriter, req *http.Request) {
        // 1. 提取 share 信息
        ctx.Share, _ = _extractShare(req)
        // 2. 提取 authorization
        ctx.Authorization = _extractAuthorization(req)
        // 3. 提取 session（包含连接参数）
        ctx.Session, _ = _extractSession(req, ctx)
        // 4. 关键：创建后端实例
        ctx.Backend, _ = _extractBackend(req, ctx)
        
        fn(ctx, res, req)
    })
}

func _extractBackend(req *http.Request, ctx *App) (IBackend, error) {
    return model.NewBackend(ctx, ctx.Session)
}
```

### 4.2 控制器方法调用

控制器通过 `ctx.Backend` 调用具体后端方法。以 `FileLs` 为例，定义于 [server/ctrl/files.go:79-191](server/ctrl/files.go#L79-L191)：

```go
func FileLs(ctx *App, res http.ResponseWriter, req *http.Request) {
    // 权限检查
    if model.CanRead(ctx) == false {
        SendErrorResult(res, ErrPermissionDenied)
        return
    }
    
    // 路径处理
    path, err := PathBuilder(ctx, req.URL.Query().Get("path"))
    
    // 授权中间件检查
    for _, auth := range Hooks.Get.AuthorisationMiddleware() {
        if err = auth.Ls(ctx, path); err != nil {
            SendErrorResult(res, err)
            return
        }
    }
    
    // 关键：调用后端的 Ls 方法（多态分发）
    entries, err := ctx.Backend.Ls(path)
    if err != nil {
        SendErrorResult(res, err)
        return
    }
    
    // 结果处理和返回
    // ...
    SendSuccessResultsWithMetadata(res, files, perms)
}
```

### 4.3 其他方法分发示例

**FileCat** [server/ctrl/files.go:193-441](server/ctrl/files.go#L193-L441)：
```go
func FileCat(ctx *App, res http.ResponseWriter, req *http.Request) {
    // ... 权限检查、路径处理 ...
    file, err = ctx.Backend.Cat(path)  // 多态调用
    // ... 插件钩子处理（缩略图、转码等）...
    io.CopyBuffer(res, file, buf)
}
```

**FileSave** [server/ctrl/files.go:479-670](server/ctrl/files.go#L479-L670)：
```go
func FileSave(ctx *App, res http.ResponseWriter, req *http.Request) {
    // ... 权限检查、路径处理 ...
    err = ctx.Backend.Save(path, req.Body)  // 多态调用
    // ...
}
```

**FileMv** [server/ctrl/files.go:736-776](server/ctrl/files.go#L736-L776)：
```go
func FileMv(ctx *App, res http.ResponseWriter, req *http.Request) {
    // ...
    err = ctx.Backend.Mv(from, to)  // 多态调用
    // ...
}
```

## 5. 关键设计要点

### 5.1 模板-实例模式

注册到 `Driver` 的是**模板对象**（如 `S3Backend{}`），每次 `Init()` 调用返回一个**新实例**：

```
注册阶段：S3Backend{} → Driver.ds["s3"]
请求阶段：Driver.Get("s3").Init(params) → &S3Backend{config: ..., params: ...}
```

这种设计的好处：
- 模板对象是无状态的，可安全并发访问
- 每个请求/会话有独立的后端实例，状态隔离
- 支持连接参数动态绑定

### 5.2 安全边界

`model.NewBackend()` 中的 `isAllowed()` 检查确保：
- 用户不能连接到配置中未定义的后端
- 不能访问配置路径范围之外的目录
- 防止利用 Filestash 作为跳板攻击其他系统

### 5.3 扩展机制

通过 `Hooks` 机制支持横向扩展，定义于 [server/common/plugin.go](server/common/plugin.go)：

```go
// 授权中间件：在后端方法调用前执行
for _, auth := range Hooks.Get.AuthorisationMiddleware() {
    if err = auth.Ls(ctx, path); err != nil {
        return ErrNotAuthorized
    }
}

// 文件内容处理钩子：在发送给用户前处理
for _, obj := range Hooks.Get.ProcessFileContentBeforeSend() {
    f, changed, err := obj(file, ctx, &res, req)
    // ...
}
```

## 6. 插件间凭据与 Session 隔离机制

### 6.1 加密密钥分层派生

所有凭据加密使用分层密钥派生策略，定义于 [server/common/constants.go:58-79](server/common/constants.go#L58-L79)：

```go
func InitSecretDerivate(secret string) {
    SECRET_KEY = secret
    SECRET_KEY_DERIVATE_FOR_PROOF     = Hash("PROOF_"+SECRET_KEY, len(SECRET_KEY))
    SECRET_KEY_DERIVATE_FOR_ADMIN     = Hash("ADMIN_"+SECRET_KEY, len(SECRET_KEY))
    SECRET_KEY_DERIVATE_FOR_USER      = Hash("USER_"+SECRET_KEY, len(SECRET_KEY))
    SECRET_KEY_DERIVATE_FOR_HASH      = Hash("HASH_"+SECRET_KEY, len(SECRET_KEY))
    SECRET_KEY_DERIVATE_FOR_SIGNATURE = Hash("SGN_"+SECRET_KEY, len(SECRET_KEY))
}
```

**隔离要点**：
- 不同用途使用不同的派生密钥，防止密钥复用攻击
- 用户会话凭据使用 `SECRET_KEY_DERIVATE_FOR_USER` 加密
- 管理员会话使用 `SECRET_KEY_DERIVATE_FOR_ADMIN` 加密
- 共享链接证明使用 `SECRET_KEY_DERIVATE_FOR_PROOF` 加密

### 6.2 Session 加密与分片存储

用户会话经过 AES-GCM 加密后存储在 Cookie 中，定义于 [server/ctrl/session.go:52-124](server/ctrl/session.go#L52-L124)：

```go
func SessionAuthenticate(ctx *App, res http.ResponseWriter, req *http.Request) {
    // ... 后端认证 ...
    s, _ := json.Marshal(session)
    // AES-GCM 加密 + zlib 压缩
    obfuscate, err := EncryptString(SECRET_KEY_DERIVATE_FOR_USER, string(s))
    
    // Cookie 分片：超过 3800 字节时拆分为多个 Cookie
    value_limit := 3800
    index := 0
    for {
        // 分片存储到 auth, auth1, auth2, ...
        http.SetCookie(res, applyCookieRules(&http.Cookie{
            Name:   CookieName(index),  // "auth", "auth1", "auth2"...
            Value:  obfuscate[start:end],
            MaxAge: 60 * Config.Get("general.cookie_timeout").Int(),
            Path:   COOKIE_PATH,        // "/api/"
        }, req))
        // ...
    }
}
```

**加密算法**（[server/common/crypto.go:27-53](server/common/crypto.go#L27-L53)）：
```go
func EncryptString(secret string, data string) (string, error) {
    d, _ := compress([]byte(data))                // zlib 压缩
    d, _ = EncryptAESGCM([]byte(secret), d)       // AES-256-GCM 加密
    return base64.URLEncoding.EncodeToString(d), nil
}
```

### 6.3 Cookie 安全边界

`applyCookieRules` 函数为所有 Cookie 应用安全规则，定义于 [server/ctrl/session.go:489-502](server/ctrl/session.go#L489-L502)：

```go
func applyCookieRules(cookie *http.Cookie, req *http.Request) *http.Cookie {
    cookie.HttpOnly = true                          // 禁止 JS 访问
    cookie.SameSite = http.SameSiteStrictMode       // 严格跨站策略
    if Config.Get("features.protection.iframe").String() != "" {
        if f := req.Header.Get("Referer"); strings.HasPrefix(f, "https://") {
            cookie.Secure = true                    // 仅 HTTPS 传输
            cookie.SameSite = http.SameSiteNoneMode // iframe 兼容
            cookie.Partitioned = true               // 分区 Cookie
        }
    }
    return cookie
}
```

**Cookie 路径隔离**（[server/common/constants.go:12-16](server/common/constants.go#L12-L16)）：
- 普通用户 Cookie：`Path = "/api/"`
- 管理员 Cookie：`Path = "/admin/api/"`
- 防止不同权限级别的 Cookie 相互干扰

### 6.4 会话 ID 生成与隔离

`GenerateID` 函数生成唯一会话标识，排除敏感字段，定义于 [server/common/crypto.go:193-218](server/common/crypto.go#L193-L218)：

```go
func GenerateID(params map[string]string) string {
    p := ""
    orderedKeys := make([]string, len(params))
    for key, _ := range params {
        orderedKeys = append(orderedKeys, key)
    }
    sort.Strings(orderedKeys)

    for _, key := range orderedKeys {
        switch key {
        case "password":    // 排除密码
        case "path":        // 排除路径
        case "session":     // 排除会话
        case "timestamp":   // 排除时间戳
        default:
            if val := params[key]; val != "" {
                p += key + "=>" + params[key] + ", "
            }
        }
    }
    p += "salt=>" + SECRET_KEY  // 加盐防止碰撞
    return Hash(p, 20)
}
```

**后端 ID 生成**（[server/ctrl/session.go:509-511](server/ctrl/session.go#L509-L511)）：
```go
func backendID(session map[string]string) string {
    return Hash(GenerateID(session)+session["path"], 20)
}
```

## 7. 多用户多 Backend 同时挂载的串扰防护

### 7.1 后端实例按请求隔离

每个 HTTP 请求经过 `SessionStart` 中间件时创建独立的后端实例，定义于 [server/middleware/session.go:57-82](server/middleware/session.go#L57-L82)：

```go
func SessionStart(fn HandlerFunc) HandlerFunc {
    return HandlerFunc(func(ctx *App, res http.ResponseWriter, req *http.Request) {
        // 每个请求有独立的 App 上下文
        ctx.Session, _ = _extractSession(req, ctx)    // 独立 Session map
        ctx.Backend, _ = _extractBackend(req, ctx)     // 独立 Backend 实例
        fn(ctx, res, req)
    })
}
```

**App 上下文结构**（[server/common/app.go:7-15](server/common/app.go#L7-L15)）：
```go
type App struct {
    Backend       IBackend                // 每个请求独立的后端实例
    Session       map[string]string       // 每个请求独立的会话数据
    Context       context.Context         // 每个请求独立的上下文
    // ...
}
```

### 7.2 缓存键的会话绑定

`AppCache` 使用完整的 session 参数哈希作为缓存键，确保不同用户/后端的缓存不冲突，定义于 [server/common/cache.go:11-45](server/common/cache.go#L11-L45)：

```go
type AppCache struct {
    Cache *cache.Cache
    sync.Mutex
}

func (a *AppCache) Get(key interface{}) interface{} {
    // 使用 hashstructure 对整个 key（通常是 session map）哈希
    hash, err := hashstructure.Hash(key, nil)
    a.Lock()
    defer a.Unlock()
    value, found := a.Cache.Get(fmt.Sprintf("%d", hash))
    return value
}

func (a *AppCache) Set(key map[string]string, value interface{}) {
    hash, _ := hashstructure.Hash(key, nil)
    a.Cache.Set(fmt.Sprint(hash), value, cache.DefaultExpiration)
}
```

**示例：SFTP 缓存隔离**（[server/plugin/plg_backend_sftp/index.go:24-42](server/plugin/plg_backend_sftp/index.go#L24-L42)）：
```go
var SftpCache AppCache

func init() {
    SftpCache = NewAppCache(1, 1)  // 1分钟保留，1分钟清理
    SftpCache.OnEvict(func(key string, value interface{}) {
        c := value.(*Sftp)
        c.wg.Wait()  // 等待所有引用释放
        c.Close()    // 关闭连接
    })
}

func (s Sftp) Init(params map[string]string, app *App) (IBackend, error) {
    // 不同 params（不同用户/不同服务器）哈希不同，缓存不同
    if c := SftpCache.Get(params); c != nil {
        d := c.(*Sftp)
        d.wg.Add(1)  // 引用计数 +1
        go func() {
            <-app.Context.Done()  // 等待请求结束
            d.wg.Done()            // 引用计数 -1
        }()
        return d, nil
    }
    // ... 创建新连接并存入缓存
}
```

### 7.3 连接池的引用计数管理

FTP/SFTP 等有状态连接使用 `sync.WaitGroup` 进行引用计数，防止并发串扰，定义于 [server/plugin/plg_backend_ftp/index.go:19-45](server/plugin/plg_backend_ftp/index.go#L19-L45)：

```go
type Ftp struct {
    client *goftp.Client
    p      map[string]string
    wg     *sync.WaitGroup  // 引用计数器
    ctx    context.Context
}

func (f Ftp) Init(params map[string]string, app *App) (IBackend, error) {
    if c := FtpCache.Get(params); c != nil {
        d := c.(*Ftp)
        d.wg.Add(1)                    // 新请求引用 +1
        d.ctx = app.Context
        go func() {
            <-d.ctx.Done()             // 请求结束信号
            d.wg.Done()                 // 引用 -1
        }()
        return d, nil
    }
    // 新连接创建时初始化 WaitGroup
    backend := &Ftp{
        client: client,
        p:      params,
        wg:     &sync.WaitGroup{},      // 新的引用计数器
    }
    backend.wg.Add(1)
    FtpCache.Set(params, backend)
    return backend, nil
}
```

### 7.4 路径的 Chroot 约束

`PathBuilder` 确保所有路径操作被限制在会话的基础路径内，防止越权访问，定义于 [server/ctrl/files.go:1104-1117](server/ctrl/files.go#L1104-L1117)：

```go
func PathBuilder(ctx *App, path string) (string, error) {
    sessionPath := ctx.Session["path"]
    basePath := filepath.ToSlash(filepath.Join(sessionPath, path))
    if path[len(path)-1:] == "/" && basePath != "/" {
        basePath += "/"
    }
    // 关键：确保最终路径以 sessionPath 为前缀
    if strings.HasPrefix(basePath, ctx.Session["path"]) == false {
        return "", ErrFilesystemError
    }
    return basePath, nil
}
```

**配置层的路径约束**（[server/model/files.go:24-32](server/model/files.go#L24-L32)）：
```go
// 在 isAllowed() 检查中
if val, ok := d["path"]; ok == true {
    if val == nil {
        val = "/"
    }
    if configPath, ok := val.(string); ok == false {
        continue
    } else if strings.HasPrefix(conn["path"], configPath) == false {
        continue  // 用户路径不在配置允许的范围内
    }
}
```

### 7.5 并发安全设计

各后端插件在共享资源访问时使用互斥锁保护：

**Wasm 插件运行时**（[server/pkg/extension/adapter/runtime/runtime.go:13-55](server/pkg/extension/adapter/runtime/runtime.go#L13-L55)）：
```go
type Runtime struct {
    wrt wazero.Runtime
    ctx context.Context
    mu  sync.Mutex      // 保护 Wasm 模块实例
    mod api.Module
}

func (r *Runtime) Call(ctx context.Context, fnName string, key, val any) error {
    r.mu.Lock()          // 调用前加锁
    defer r.mu.Unlock()
    fn := r.mod.ExportedFunction(fnName)
    _, err := fn.Call(...)
    return err
}
```

**缓存访问**（[server/common/cache.go:11-14](server/common/cache.go#L11-L14)）：
```go
type AppCache struct {
    Cache *cache.Cache
    sync.Mutex        // 内嵌互斥锁
}
```

## 8. 插件热加载与外部依赖兼容性策略

### 8.1 插件分类与加载机制

Filestash 有两类插件，采用完全不同的加载策略：

| 插件类型 | 加载方式 | 热加载支持 | 示例 |
|---------|---------|-----------|------|
| 静态编译后端 | Go import 静态链接 | 不支持 | plg_backend_s3, plg_backend_ftp |
| 外部扩展插件 | Wasm 沙箱动态加载 | 启动时加载 | 自定义 middleware, workflow action |

#### 静态编译后端插件

所有存储后端通过 Go 空白导入静态链接，编译时确定，定义于 [server/plugin/index.go:3-46](server/plugin/index.go#L3-L46)：

```go
import (
    _ "github.com/mickael-kerjean/filestash/server/plugin/plg_backend_local"
    _ "github.com/mickael-kerjean/filestash/server/plugin/plg_backend_s3"
    _ "github.com/mickael-kerjean/filestash/server/plugin/plg_backend_ftp"
    // ... 20+ 后端全部在编译时链接
)
```

**特点**：
- 无运行时热替换能力，修改需要重新编译部署
- 类型安全，编译时检查接口实现
- 性能最优，无沙箱开销

#### 外部扩展插件

外部插件以 `.zip` 包形式放置在 `state/plugins/` 目录，启动时通过 Wasm 沙箱加载，定义于 [server/pkg/extension/discovery.go:15-81](server/pkg/extension/discovery.go#L15-L81)：

```go
func Discovery() error {
    entries, err := os.ReadDir(GetAbsolutePath(PLUGIN_PATH))
    for _, entry := range entries {
        if strings.HasSuffix(entry.Name(), ".zip") == false {
            continue
        }
        name, impl, err := initModule(fname)
        if err != nil {
            // 失败跳过，不影响其他插件和主程序
            Log.Error("could not initialise module name=%s err=%s", entry.Name(), err.Error())
            continue
        }
        // 根据模块类型注册到 Hooks
        for i := 0; i < len(impl.Modules); i++ {
            switch impl.Modules[i]["type"] {
            case "css":
                b, _ := GetPluginFile(name, impl.Modules[i]["entrypoint"])
                Hooks.Register.CSS(string(b))
            case "middleware":
                b, _ := GetPluginFile(name, impl.Modules[i]["entrypoint"])
                m, _ := adapter.MiddlewareExtension(b)  // 编译为 Wasm 模块
                Hooks.Register.Middleware(m)
            case "workflow::action":
                // ...
            }
        }
        plugins[name] = impl
    }
    return nil
}
```

### 8.2 插件加载的安全检查与失败回滚

**单插件失败隔离**：
```go
name, impl, err := initModule(fname)
if err != nil {
    // 仅记录错误并跳过该插件
    Log.Error("could not initialise module name=%s err=%s", entry.Name(), err.Error())
    continue  // 不 panic，不影响其他插件
}
```

**Wasm 沙箱安全边界**（[server/pkg/extension/adapter/runtime/runtime.go:21-44](server/pkg/extension/adapter/runtime/runtime.go#L21-L44)）：
```go
func New(wasm []byte, opts ...Option) (*Runtime, error) {
    ctx := context.Background()
    wrt := wazero.NewRuntime(ctx)                     // 创建独立 Wasm 运行时
    wasi_snapshot_preview1.MustInstantiate(ctx, wrt)  // 仅暴露 WASI 接口
    
    compiled, err := wrt.CompileModule(ctx, wasm)     // 编译验证
    if err != nil {
        wrt.Close(ctx)                                 // 失败时清理资源
        return nil, err
    }
    mod, err := wrt.InstantiateModule(ctx, compiled, wazero.NewModuleConfig())
    if err != nil {
        wrt.Close(ctx)                                 // 失败时清理资源
        return nil, err
    }
    return &Runtime{ctx: ctx, wrt: wrt, mod: mod}, nil
}
```

**失败回滚机制**：
1. 插件加载失败时调用 `wrt.Close(ctx)` 释放所有资源
2. 错误仅记录日志，不中断主程序启动
3. 已成功加载的插件不受影响
4. 无状态回滚（外部插件无持久化状态）

**无运行时热替换**：
- 当前设计仅在服务启动时执行一次 `Discovery()`
- 无动态加载/卸载 API
- 插件更新需要重启服务

### 8.3 外部依赖兼容性策略

项目通过以下策略管理 20+ 存储后端的 SDK 依赖：

#### 版本锁定

`go.mod` 中所有外部依赖使用精确版本号，定义于 [go.mod:5-51](go.mod#L5-L51)：

```go
require (
    github.com/Azure/azure-sdk-for-go/sdk/storage/azblob v1.6.4
    github.com/aws/aws-sdk-go v1.55.8                    # AWS S3 SDK
    github.com/go-git/go-git/v6 v6.0.0-alpha.3            # Git 后端
    github.com/go-ldap/ldap/v3 v3.4.13                    # LDAP 后端
    github.com/go-sql-driver/mysql v1.9.3                  # MySQL 后端
    github.com/h2non/bimg v1.1.9
    github.com/hirochachacha/go-smb2 v1.1.0                # Samba 后端
    github.com/lib/pq v1.12.3                              # PostgreSQL 后端
    github.com/mattn/go-sqlite3 v1.14.42                   # SQLite 后端
    github.com/mickael-kerjean/goftp v0.0.0-20260421114701-956d21f038b7  # FTP（自维护 fork）
    github.com/pkg/sftp v1.13.10                            # SFTP 客户端
    github.com/tidwall/gjson v1.18.0
    github.com/vmware/go-nfs-client v0.0.0-20190605212624-d43b92724c1b  # NFS 客户端
    golang.org/x/crypto v0.50.0                              # SSH/SFTP 基础
    golang.org/x/net v0.53.0                                 # WebDAV/HTTP 基础
    google.golang.org/api v0.276.0                           # Google Drive
    storj.io/uplink v1.14.0                                  # Storj 后端
)
```

#### 关键依赖版本策略

| 后端 | 依赖包 | 版本 | 策略说明 |
|------|--------|------|---------|
| **S3** | `github.com/aws/aws-sdk-go` | v1.55.8 | 使用 AWS SDK v1，稳定版本 |
| **SFTP** | `github.com/pkg/sftp` | v1.13.10 | 社区维护的标准 SFTP 库 |
| **FTP** | `github.com/mickael-kerjean/goftp` | fork 版本 | 自维护 fork，修复上游 bug |
| **WebDAV** | `golang.org/x/net` | v0.53.0 | 使用标准库 `net/http` + 自定义 XML 解析 |
| **Git** | `github.com/go-git/go-git/v6` | v6.0.0-alpha.3 | 使用纯 Go 实现，无需系统 git |
| **Azure Blob** | `github.com/Azure/azure-sdk-for-go/sdk/storage/azblob` | v1.6.4 | Azure 官方 SDK |
| **Google Drive** | `google.golang.org/api` | v0.276.0 | Google API 官方客户端 |
| **PostgreSQL** | `github.com/lib/pq` | v1.12.3 | 纯 Go PostgreSQL 驱动 |
| **MySQL** | `github.com/go-sql-driver/mysql` | v1.9.3 | 社区标准 MySQL 驱动 |
| **LDAP** | `github.com/go-ldap/ldap/v3` | v3.4.13 | 社区标准 LDAP 客户端 |
| **Samba** | `github.com/hirochachacha/go-smb2` | v1.1.0 | 纯 Go SMB2/3 实现 |

#### HTTP 客户端标准化

所有后端共享统一的 HTTP 客户端配置，确保行为一致，定义于 [server/common/default.go:42-61](server/common/default.go#L42-L61)：

```go
func HTTPClient(opts ...HTTPClientOption) *http.Client {
    cfg := &httpClientConfig{
        transport: &http.Transport{
            Dial: (&net.Dialer{
                Timeout:   10 * time.Second,     // 连接超时
                KeepAlive: 10 * time.Second,     // 保活时间
            }).Dial,
            TLSHandshakeTimeout:   5 * time.Second,   // TLS 握手超时
            IdleConnTimeout:       60 * time.Second,  // 空闲连接超时
            ResponseHeaderTimeout: 60 * time.Second,  // 响应头超时
        },
    }
    for _, opt := range opts {
        opt(cfg)
    }
    return &http.Client{
        Timeout:   5 * time.Hour,    // 总超时（长传输）
        Transport: tracer.NewTransport(cfg.traceService, 
                    NewTransformedTransport(cfg.transport)),
    }
}
```

**统一 User-Agent**（[server/common/default.go:93-104](server/common/default.go#L93-L104)）：
```go
type TransformedTransport struct {
    Orig http.RoundTripper
}

func (this *TransformedTransport) RoundTrip(req *http.Request) (*http.Response, error) {
    req.Header.Add("User-Agent", USER_AGENT)  // 统一请求标识
    return this.Orig.RoundTrip(req)
}
```

#### TLS 配置标准化

`DefaultTLSConfig` 提供安全的默认 TLS 配置，定义于 [server/common/default.go:76-91](server/common/default.go#L76-L91)：

```go
var DefaultTLSConfig = tls.Config{
    MinVersion: tls.VersionTLS12,                // 最低 TLS 1.2
    CipherSuites: []uint16{
        tls.TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,
        tls.TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
        tls.TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,
        tls.TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,
        tls.TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,
        tls.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
    },
    PreferServerCipherSuites: true,
    CurvePreferences: []tls.CurveID{
        tls.CurveP256,
        tls.X25519,
    },
}
```

#### 依赖维护策略

1. **关键依赖 Fork**：对于 FTP 等上游维护不活跃的依赖，使用自维护 Fork
2. **定期更新**：通过 `go get -u` 定期更新小版本
3. **测试覆盖**：各后端插件通过集成测试验证 SDK 兼容性
4. **间接依赖隔离**：使用 `go mod tidy` 确保仅保留必要依赖
5. **CVE 监控**：依赖 `go list -m -json all` 配合安全扫描工具监控漏洞

## 9. 完整调用链示例

以列出目录请求为例：

```
HTTP GET /api/ls?path=/documents
    ↓
SessionStart 中间件
    ├→ _extractSession() → 解密 session 获取 {type: "s3", access_key: "...", path: "/bucket"}
    ├→ _extractBackend() 
    │   └→ model.NewBackend(ctx, session)
    │       ├→ isAllowed() → 检查配置
    │       └→ Backend.Get("s3").Init(params, ctx) → 返回 *S3Backend
    └→ ctx.Backend = *S3Backend
    ↓
FileLs 控制器
    ├→ 权限检查 model.CanRead(ctx)
    ├→ PathBuilder(ctx, "/documents") → "/bucket/documents"
    ├→ 授权中间件检查 auth.Ls(ctx, path)
    ├→ ctx.Backend.Ls("/bucket/documents") → 调用 S3Backend.Ls()
    │   └→ S3 客户端调用 s3.ListObjectsV2(...)
    ├→ 结果转换为 FileInfo 数组
    └→ SendSuccessResultsWithMetadata(...)
```

## 10. 相关文件速查表

| 文件 | 职责 |
|------|------|
| [server/common/types.go](server/common/types.go) | `IBackend` 接口定义 |
| [server/common/backend.go](server/common/backend.go) | `Driver` 注册中心、`Nothing` 空实现 |
| [server/common/plugin.go](server/common/plugin.go) | `Hooks` 扩展机制 |
| [server/common/crypto.go](server/common/crypto.go) | 加密、哈希、会话 ID 生成 |
| [server/common/constants.go](server/common/constants.go) | 密钥派生、Cookie 配置 |
| [server/common/cache.go](server/common/cache.go) | 会话绑定缓存、并发安全 |
| [server/common/default.go](server/common/default.go) | HTTP 客户端、TLS 标准化配置 |
| [server/plugin/index.go](server/plugin/index.go) | 所有插件导入入口 |
| [server/model/files.go](server/model/files.go) | `NewBackend()` 工厂函数、安全检查 |
| [server/middleware/session.go](server/middleware/session.go) | 中间件注入 `ctx.Backend` |
| [server/ctrl/session.go](server/ctrl/session.go) | 会话认证、Cookie 管理、安全规则 |
| [server/ctrl/files.go](server/ctrl/files.go) | 控制器方法分发、路径约束 |
| [server/pkg/extension/discovery.go](server/pkg/extension/discovery.go) | 外部插件发现与加载 |
| [server/pkg/extension/adapter/runtime/](server/pkg/extension/adapter/runtime/) | Wasm 沙箱运行时 |
| [server/plugin/plg_backend_*/index.go](server/plugin/) | 各后端具体实现 |
| [go.mod](go.mod) | 外部依赖版本锁定 |
