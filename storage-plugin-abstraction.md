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

## 9. 插件加载失败时的备选逻辑与回退机制

### 9.1 多级回退链设计

Filestash 的回退机制不是"自动切换到本地存储"，而是**分层降级**，从特定后端 → 空实现 → 前端报错。完整回退链如下：

```
后端认证/加载失败
    ↓
┌─────────────────────────────────────────────┐
│ Level 1: isAllowed() 配置检查失败          │
│   → 返回 Nothing{} + ErrNotAllowed         │
├─────────────────────────────────────────────┤
│ Level 2: Driver.Get(type) 找不到后端类型   │
│   → 返回 Nothing{} （静默降级）            │
├─────────────────────────────────────────────┤
│ Level 3: Init() 认证失败/连接超时          │
│   → 返回 nil + err （SessionAuthenticate  │
│     返回 401 给前端，前端提示"认证失败"）   │
├─────────────────────────────────────────────┤
│ Level 4: 运行时操作失败（Ls/Cat/Save...）  │
│   → Nothing 实现返回 ErrNotImplemented    │
│     或空数组，前端显示空目录/报错           │
└─────────────────────────────────────────────┘
```

### 9.2 Level 1 & 2：Driver 层的静默降级

`Driver.Get()` 在后端类型不存在或被显式禁用时，不报错而是返回 `Nothing{}` 空实现，定义于 [server/common/backend.go:28-34](server/common/backend.go#L28-L34)：

```go
const BACKEND_NIL = "_nothing_"

func (d *Driver) Get(name string) IBackend {
    b := d.ds[name]
    if b == nil || name == BACKEND_NIL {
        // 关键：不返回 error，返回空实现实现静默降级
        return Nothing{}
    }
    return b
}
```

**触发场景**：
- 用户请求的后端类型未被编译进程序（未在 `server/plugin/index.go` 导入）
- 管理员在 `isAllowed()` 检查中显式返回 `BACKEND_NIL`
- 后端插件 `init()` 注册因 panic 失败（Go 的 init panic 会导致程序直接退出，无回退）

### 9.3 Level 3：Init() 失败时的前端提示

`model.NewBackend()` 在 `isAllowed()` 检查失败时显式使用 `BACKEND_NIL`，定义于 [server/model/files.go:44-49](server/model/files.go#L44-L49)：

```go
func NewBackend(ctx *App, conn map[string]string) (IBackend, error) {
    isAllowed := func() bool { /* 配置检查 */ }
    
    if isAllowed() == false {
        // 显式降级到 Nothing，同时返回 ErrNotAllowed 供上层处理
        return Backend.Get(BACKEND_NIL), ErrNotAllowed
    }
    
    return Backend.Get(conn["type"]).Init(conn, ctx)
}
```

**SessionAuthenticate 控制器处理**（[server/ctrl/session.go:52-62](server/ctrl/session.go#L52-L62)）：
```go
func SessionAuthenticate(ctx *App, res http.ResponseWriter, req *http.Request) {
    backend, err := model.NewBackend(ctx, session)
    if err != nil {
        // 将错误直接返回给前端，前端展示"认证失败"
        SendErrorResult(res, err)
        return
    }
    // 认证成功，下发加密 Cookie
}
```

**运营注意**：
- **无自动切换到本地存储的逻辑**。后端加载/认证失败时，用户停留在登录页，提示具体错误信息。
- 若需"默认本地存储"行为，需通过配置 `connections` 预设 local 后端作为唯一可用选项，或修改前端逻辑在认证失败时跳转到 local 登录表单。

### 9.4 Level 4：Nothing 空实现的运行时语义

`Nothing` 结构体的各方法返回值设计确保了前端不会崩溃，定义于 [server/common/backend.go:40-79](server/common/backend.go#L40-L79)：

```go
type Nothing struct{}

func (b Nothing) Ls(path string) ([]os.FileInfo, error) {
    return []os.FileInfo{}, nil     // 返回空数组，前端显示空目录
}
func (b Nothing) Stat(path string) (os.FileInfo, error) {
    return nil, ErrNotFound         // 返回 404，前端提示不存在
}
func (b Nothing) Cat(path string) (io.ReadCloser, error) {
    return NewReadCloserFromReader(strings.NewReader("")), ErrNotImplemented
}
func (b Nothing) Mkdir(path string) error { return ErrNotImplemented }
func (b Nothing) Rm(path string) error    { return ErrNotImplemented }
func (b Nothing) Mv(from, to string) error { return ErrNotImplemented }
func (b Nothing) Save(path string, file io.Reader) error { return ErrNotImplemented }
func (b Nothing) Touch(path string) error  { return ErrNotImplemented }
```

**运营含义**：
- `Ls` 返回空数组 → 用户看到一个看似正常的空目录，可能误认为后端无文件（需配合日志排查）
- 写操作统一返回 `ErrNotImplemented` → 用户操作被拒绝
- 没有数据写入本地存储作为"暂存区"的逻辑，所有失败都是显式报错或空结果

### 9.5 插件 import 失败的极端场景

由于存储后端通过 Go 空白导入 `_ "plg_backend_*"` 在编译时链接：
- **编译时缺失**：如果 `server/plugin/index.go` 中注释掉某后端 import，该后端从注册表里消失，用户请求时触发 Level 2 降级到 `Nothing{}`
- **init() 中 panic**：Go 的 `init()` 函数 panic 会导致整个程序退出，不会跳过该插件继续加载其他。因此各后端 `init()` 仅做 `Backend.Register()`（无法 panic 的操作），不做网络调用或文件 IO
- **外部 Wasm 插件**：仅影响 middleware/workflow 等扩展点，不影响存储后端的可用性

## 10. 插件间共享凭证的加密实现及密钥更新流程

### 10.1 凭证的存储位置与加密范围

Filestash 的凭证分散在三个位置，使用不同密钥加密：

| 凭证类型 | 存储位置 | 加密密钥 | 定义位置 |
|---------|---------|---------|---------|
| **用户会话凭证** | 浏览器 Cookie（`auth`, `auth1`...） | `SECRET_KEY_DERIVATE_FOR_USER` | [server/ctrl/session.go:96](server/ctrl/session.go#L96) |
| **管理员会话凭证** | 浏览器 Cookie（`admin`） | `SECRET_KEY_DERIVATE_FOR_ADMIN` | [server/ctrl/admin.go:67](server/ctrl/admin.go#L67) |
| **中间件配置凭证** | `state/config/config.json` | `SECRET_KEY_DERIVATE_FOR_PROOF` | [server/common/config_state.go:24-27](server/common/config_state.go#L24-L27) |
| **共享链接证明** | 浏览器 Cookie（`proof`） | `SECRET_KEY_DERIVATE_FOR_PROOF` | [server/model/share.go:303](server/model/share.go#L303) |
| **会话签名** | HTTP Header | `SECRET_KEY_DERIVATE_FOR_SIGNATURE` | [server/ctrl/session.go:380](server/ctrl/session.go#L380) |

### 10.2 配置文件中的凭证加密

`configKeysToEncrypt` 定义了需要在 `config.json` 中加密的字段路径，定义于 [server/common/config_state.go:24-27](server/common/config_state.go#L24-L27)：

```go
var configKeysToEncrypt []string = []string{
    "middleware.identity_provider.params",  // SSO/OAuth 客户端密钥
    "middleware.attribute_mapping.params",  // LDAP/AD 查询密码
}
```

**加载时解密**（[server/common/config_state.go:43-68](server/common/config_state.go#L43-L68)）：
```go
func LoadConfig() ([]byte, error) {
    // ... 读取文件 ...
    
    // 步骤1: 先从明文 secret_key 初始化派生密钥
    if os.Getenv("CONFIG_SECRET") == "" {
        InitSecretDerivate(gjson.Get(configStr, "general.secret_key").String())
    }
    
    // 步骤2: 用 PROOF 派生密钥解密配置中的敏感字段
    key := defaultValue(SECRET_KEY_DERIVATE_FOR_PROOF, "CONFIG_SECRET")
    for _, jsonPathWithEncryptedData := range configKeysToEncrypt {
        p := gjson.Get(configStr, jsonPathWithEncryptedData).String()
        if p == "" { continue }
        t, err := DecryptString(Hash(key, 16), p)  // 二次哈希缩短密钥长度
        if err != nil {
            Log.Warning("cannot decrypt config path '%s': %s", ...)
            continue  // 解密失败跳过，不阻塞启动
        }
        configStr, _ = sjson.Set(configStr, jsonPathWithEncryptedData, t)
    }
    return []byte(configStr), nil
}
```

**保存时加密**（[server/common/config_state.go:71-113](server/common/config_state.go#L71-L113)）：
```go
func SaveConfig(v []byte) error {
    // ...
    key := defaultValue(SECRET_KEY_DERIVATE_FOR_PROOF, "CONFIG_SECRET")
    for _, jsonPathWithEncryptedData := range configKeysToEncrypt {
        p := gjson.Get(configStr, jsonPathWithEncryptedData).String()
        if p == "" { continue }
        t, err := EncryptString(Hash(key, 16), p)  // 与解密相同的密钥派生
        // ...
    }
    // ...
}
```

### 10.3 会话凭证的加密流程

用户 Session（包含后端密码、Access Key 等）通过以下流程加密：

```
明文 session JSON (含 password/access_key)
    ↓ json.Marshal
原始字节
    ↓ zlib 压缩 (crypto.go:28)
压缩字节
    ↓ AES-256-GCM 加密，密钥 = SECRET_KEY_DERIVATE_FOR_USER (crypto.go:32)
    ↓ nonce(12) + ciphertext + tag
密文
    ↓ base64.URLEncoding (crypto.go:36)
URL 安全字符串
    ↓ 按 3800 字节分片
auth, auth1, auth2, ... Cookie
```

**加密算法细节**（[server/common/crypto.go:128-164](server/common/crypto.go#L128-L164)）：
```go
func EncryptAESGCM(key []byte, plaintext []byte) ([]byte, error) {
    c, _ := aes.NewCipher(key)
    gcm, _ := cipher.NewGCM(c)
    nonce := GCMNonce.Next()  // 全局递增 nonce，带 sync.Mutex 保护
    // 输出格式: nonce || ciphertext || auth_tag
    return gcm.Seal(nonce, nonce, plaintext, nil), nil
}
```

### 10.4 主密钥初始化流程

`Configuration.Initialise()` 在服务启动时生成/加载主密钥，定义于 [server/common/config.go:241-260](server/common/config.go#L241-L260)：

```go
func (this *Configuration) Initialise() {
    shouldSave := false
    
    // ... 环境变量覆盖 ...
    
    // 步骤1: 首次启动时生成 16 字节随机密钥
    if this.Get("general.secret_key").String() == "" {
        shouldSave = true
        key := RandomString(16)         // 从加密安全的 rand.Reader 生成
        this.Get("general.secret_key").Set(key)
    }
    
    if shouldSave {
        this.Save()                      // 持久化到 config.json
    }
    
    // 步骤2: 从主密钥派生所有子密钥
    InitSecretDerivate(this.Get("general.secret_key").String())
}
```

### 10.5 密钥更新流程（手动操作）

当前代码库**没有自动密钥轮换机制**。手动更换主密钥的操作步骤如下：

```
步骤1: 通知所有用户密钥即将轮换（现有会话将全部失效）
步骤2: 管理员登录后台，记录当前 connections 配置（不含密码）
步骤3: 停止服务，备份 state/config/config.json
步骤4: 编辑 config.json，设置新的 general.secret_key（16字符以上）
步骤5: 清空 config.json 中所有已加密字段（会用旧密钥加密，新密钥无法解密）：
       - middleware.identity_provider.params
       - middleware.attribute_mapping.params
步骤6: 删除所有 state/db/ 下的持久化会话（如果有）
步骤7: 重启服务
步骤8: 管理员重新配置 SSO/LDAP 等中间件参数（会用新密钥加密）
步骤9: 所有用户需重新登录（旧 Cookie 无法解密）
```

**密钥更新影响面**：
| 数据类型 | 是否失效 | 恢复方式 |
|---------|---------|---------|
| 已登录用户的 Session Cookie | 是 | 用户重新登录 |
| 管理员 Session Cookie | 是 | 管理员重新登录 |
| 共享链接（无密码） | 不受影响 | 通过独立的 share_id 识别 |
| 共享链接（有密码） | 是 | 需要重新创建或重新输入密码（密码哈希独立） |
| `config.json` 中的中间件参数 | 是 | 重新配置并保存 |
| 已缓存的后端连接 | 是 | 自动重建，用户无感 |
| 文件内容/元数据 | 不受影响 | 不依赖主密钥加密 |

### 10.6 OAuth 凭证的特殊处理

Google Drive、Dropbox 等支持 OAuth 的后端，Access Token/Refresh Token 经过 `OAuthToken` 方法补充到 session 后，随其他字段一起加密存储在 Cookie 中：

定义于 [server/ctrl/session.go:65-80](server/ctrl/session.go#L65-L80)：
```go
if obj, ok := backend.(interface {
    OAuthToken(*map[string]interface{}) error
}); ok {
    if err := obj.OAuthToken(&ctx.Body); err != nil {
        SendErrorResult(res, NewError("Can't authenticate (OAuth error)", 401))
        return
    }
    // OAuth 成功后，session 包含 access_token/refresh_token，
    // 这些 token 会被整个 session JSON 一起加密存储
    session = model.MapStringInterfaceToMapStringString(ctx.Body)
    backend, err = model.NewBackend(ctx, session)
}
```

**存储链**：
```
OAuth 回调 → OAuthToken() 填充 access_token/refresh_token
    → session map 完整序列化
    → EncryptString(USER_KEY, JSON)
    → 分片写入 auth Cookie
```

## 11. 多后端同时挂载时的文件名冲突与命名空间隔离

### 11.1 架构前提：单会话单后端

Filestash 的核心设计是**每个浏览器会话只关联一个后端类型和一个根路径**，不是多后端聚合文件管理器。因此不存在真正意义上的"同时挂载多个后端并合并命名空间"的场景。

**会话结构**（[server/ctrl/session.go:21-26](server/ctrl/session.go#L21-L26)）：
```go
type Session struct {
    Home          *string `json:"home,omitempty"`
    IsAuth        bool    `json:"is_authenticated"`
    Backend       string  `json:"backendID"`   // 单个后端 ID
    Authorization string  `json:"authorization,omitempty"`
}
```

每个 HTTP 请求的 `ctx.Backend` 是**单一** `IBackend` 实例，不是数组或映射。

### 11.2 命名空间隔离的四层设计

尽管不支持多后端聚合，但 Filestash 通过四层机制确保单个后端实例内的路径/文件不串扰：

```
┌──────────────────────────────────────────────────────┐
│ Layer 1: 会话级 Path 前缀约束                        │
│   ctx.Session["path"] 作为所有操作的强制前缀        │
│   PathBuilder() 确保 HasPrefix 成立                  │
├──────────────────────────────────────────────────────┤
│ Layer 2: 配置级 isAllowed() 范围校验                 │
│   NewBackend() 检查连接参数在 Admin 配置范围内      │
├──────────────────────────────────────────────────────┤
│ Layer 3: 后端级内部路径映射                           │
│   单后端多共享/多存储桶时，通过第一层虚拟目录隔离    │
├──────────────────────────────────────────────────────┤
│ Layer 4: 本地存储级 Chroot 目录                       │
│   TmpStorage/URL 后端为每个用户创建独立的文件系统根  │
└──────────────────────────────────────────────────────┘
```

### 11.3 Layer 1：PathBuilder 的前缀约束

定义于 [server/ctrl/files.go:1104-1117](server/ctrl/files.go#L1104-L1117)：

```go
func PathBuilder(ctx *App, path string) (string, error) {
    if path == "" {
        return "", NewError("No path available", 400)
    }
    sessionPath := ctx.Session["path"]
    // 将会话根路径 + 用户请求路径拼接
    basePath := filepath.ToSlash(filepath.Join(sessionPath, path))
    if path[len(path)-1:] == "/" && basePath != "/" {
        basePath += "/"
    }
    // 关键校验：防止通过 ../../ 越界
    if strings.HasPrefix(basePath, ctx.Session["path"]) == false {
        return "", ErrFilesystemError
    }
    return basePath, nil
}
```

**越权示例**：
```
会话 path = "/bucket/userA/"
用户请求 path = "/../userB/secret.txt"
拼接后 = "/bucket/userB/secret.txt"
HasPrefix("/bucket/userA/") = false → 返回 ErrFilesystemError
```

### 11.4 Layer 2：配置级白名单校验

`model.NewBackend()` 中的 `isAllowed()` 确保用户只能连接 Admin 在配置中预设的后端和路径范围：

```go
// 路径范围检查
if val, ok := d["path"]; ok == true {
    configPath := val.(string)
    // 用户的 session path 必须以 configPath 为前缀
    if strings.HasPrefix(conn["path"], configPath) == false {
        continue  // 不匹配，该配置项不可用
    }
}

// 主机/URL 范围检查
if val, ok := d["hostname"]; ok == true {
    if conn["hostname"] != val.(string) { continue }
}
```

### 11.5 Layer 3：单后端内部的虚拟命名空间

部分后端（Samba、S3、CardDAV）在一个连接下有多个"桶/共享/集合"，通过第一层虚拟目录实现内部隔离。

**Samba 多共享挂载**（[server/plugin/plg_backend_samba/index.go:35-38](server/plugin/plg_backend_samba/index.go#L35-L38)）：
```go
type Samba struct {
    session *smb2.Session
    share   map[string]*smb2.Share  // 多个共享名 → Share 句柄映射
}
```

**Ls 方法的命名空间分发**（[server/plugin/plg_backend_samba/index.go:170-194](server/plugin/plg_backend_samba/index.go#L170-L194)）：
```go
func (smb Samba) Ls(path string) ([]os.FileInfo, error) {
    // 根路径 "/" 时，返回所有可用的共享名作为虚拟子目录
    if path == "/" {
        f := make([]os.FileInfo, 0)
        for key, _ := range smb.share {
            f = append(f, File{
                FName: key,      // "documents", "videos", "backups"...
                FType: "directory",
            })
        }
        return f, nil
    }
    // 非根路径，解析出共享名 + 相对路径
    share, path, err := smb.toSambaPath(path)
    // path = "/documents/report.pdf"
    // → share = smb.share["documents"], path = "/report.pdf"
    dir, err := share.Open(path)
    return dir.Readdir(-1), nil
}
```

**S3 多桶模式** 类似：根目录列出所有配置的 bucket，进入 bucket 后才进行实际 S3 API 调用。

**命名冲突处理**：
- Samba：共享名由服务器管理员命名，不会重名
- S3：Bucket 名全局唯一，不会冲突
- 如果后端列表中有两个同名共享（例如通过不同协议挂载同名目录），**无自动重命名机制**，后注册的会覆盖先注册的

### 11.6 Layer 4：本地 Chroot 目录隔离

TmpStorage 和 URL 下载后端为每个用户创建完全独立的文件系统根：

**TmpStorage 隔离**（[server/plugin/plg_backend_tmp/index.go:13-51](server/plugin/plg_backend_tmp/index.go#L13-L51)）：
```go
const FILESTASH_DIRECTORY = "/tmp/filestash_tmp/"

func (this TmpStorage) Init(params map[string]string, app *App) (IBackend, error) {
    // userID 必须匹配正则 [a-zA-Z0-9]*，防止路径注入
    if regexp.MustCompile(`^[a-zA-Z0-9]*$`).MatchString(params["userID"]) == false {
        return nil, ErrAuthenticationFailed
    }
    this.userID = params["userID"]
    // 每个 user 的 chroot = /tmp/filestash_tmp/{userID}/
    root, err := this.fullpath("/")
    os.MkdirAll(root, 0755)
    return &this, nil
}

func (this TmpStorage) fullpath(path string) (string, error) {
    // 内部通过 userID 拼接真实路径
    p := filepath.Join(FILESTASH_DIRECTORY, this.userID, path)
    // 二次检查是否逃逸出用户目录
    if strings.HasPrefix(p, filepath.Join(FILESTASH_DIRECTORY, this.userID)) == false {
        Log.Warning("plg_backend_tmp::chroot attempt to circumvent chroot via path[%s]", path)
        return "", ErrFilesystemError
    }
    return p, nil
}
```

**ChrootCache 自动清理**（[server/plugin/plg_backend_tmp/index.go:22-28](server/plugin/plg_backend_tmp/index.go#L22-L28)）：
```go
ChrootCache.OnEvict(func(key string, value interface{}) {
    chroot := value.(string)
    // 用户 30 天不活动后，自动删除其临时目录
    if strings.HasPrefix(chroot, FILESTASH_DIRECTORY) {
        os.RemoveAll(chroot)
    }
})
```

### 11.7 跨用户文件名冲突的场景分析

| 场景 | 隔离机制 | 潜在风险 |
|------|---------|---------|
| **Local 后端多用户使用同一路径前缀** | PathBuilder HasPrefix 检查 | 若 Admin 配置 path="/data/shared/"，所有用户读写同一目录，**会有文件名冲突**。需由上层业务解决（如按用户名建子目录） |
| **Local 后端用户 A path="/data/a/"，用户 B path="/data/b/"** | PathBuilder 独立前缀 + 系统文件权限 | 安全隔离，无冲突。但需确保 OS 级目录权限正确设置 |
| **S3 同 bucket 不同用户** | `session["path"] = "/bucket/userA/"` | PathBuilder 前缀隔离，除非手工构造路径绕过 |
| **Samba 同服务器不同共享** | 第一层虚拟目录（共享名）+ `toSambaPath()` 路由 | 天然隔离。跨共享 Mv 需两个共享分别 Open，由 Samba 插件内部处理 |
| **TmpStorage 不同 userID** | 独立 Chroot 目录 `/tmp/filestash_tmp/{id}/` | 完全隔离，缓存驱逐自动清理 |

### 11.8 跨后端操作的边界

当前架构**不支持跨后端的文件操作**（如从 S3 直接 Mv 到 FTP）：

- `Mv(from, to)` 的 `from` 和 `to` 必须在同一个 `ctx.Backend` 下
- 跨后端传输需要前端分别调用 `Cat` 下载 + `Save` 上传
- 因此不会出现跨后端文件名冲突问题

## 12. 插件操作的审计日志与权限链路

### 12.1 三层权限检查链路

每个文件操作经过**三层**权限检查，层层递进：

```
HTTP 请求
    ↓
Layer 1: 粗粒度权限 (CanRead / CanEdit / CanUpload / CanShare)
    ↓ 基于 Share 链接属性或默认 true
Layer 2: 授权中间件链 (AuthorisationMiddleware)
    ↓ 按注册顺序逐个调用 auth.Ls()/auth.Cat()/...
    ↓ 任意一个返回 error 即拒绝
Layer 3: 后端级 ACL (IBackend 内部实现)
    ↓
实际执行操作
```

#### Layer 1：粗粒度权限

定义于 [server/model/permissions.go:7-33](server/model/permissions.go#L7-L33)：

```go
func CanRead(ctx *App) bool {
    if ctx.Share.Id != "" {
        return ctx.Share.CanRead  // 共享链接有显式权限则遵循
    }
    return true  // 已登录用户默认可读
}

func CanEdit(ctx *App) bool {
    if ctx.Share.Id != "" {
        return ctx.Share.CanWrite
    }
    return true
}
```

**调用时机**：在控制器最开始检查，如 [server/ctrl/files.go:80](server/ctrl/files.go#L80)：
```go
func FileLs(ctx *App, res http.ResponseWriter, req *http.Request) {
    if model.CanRead(ctx) == false {
        SendErrorResult(res, ErrPermissionDenied)
        return
    }
    // ...
}
```

#### Layer 2：授权中间件链

`IAuthorisation` 接口支持按方法级细粒度控制，定义于 [server/common/types.go:36-45](server/common/types.go#L36-L45)：

```go
type IAuthorisation interface {
    Ls(ctx *App, path string) error
    Cat(ctx *App, path string) error
    Mkdir(ctx *App, path string) error
    Rm(ctx *App, path string) error
    Mv(ctx *App, from string, to string) error
    Save(ctx *App, path string) error
    Touch(ctx *App, path string) error
}
```

**调用链**（以 `FileLs` 为例，[server/ctrl/files.go:99-103](server/ctrl/files.go#L99-L103)）：
```go
for _, auth := range Hooks.Get.AuthorisationMiddleware() {
    if err = auth.Ls(ctx, path); err != nil {
        SendErrorResult(res, err)
        return
    }
}
```

**注册顺序 = 执行顺序**：
- 插件通过 `Hooks.Register.AuthorisationMiddleware(impl)` 注册
- 按 `init()` 执行顺序（即 `server/plugin/index.go` 中的 import 顺序）添加到 slice
- 执行时按 slice 顺序遍历，**短路语义**：第一个返回 error 的中间件终止整条链

**示例：plg_authorisation_example**（[server/plugin/plg_authorisation_example/index.go:13-45](server/plugin/plg_authorisation_example/index.go#L13-L45)）：
```go
func (this AuthM) Ls(ctx *App, path string) error {
    Log.Stdout("LS %+v", ctx.Session)  // 审计日志
    return nil                          // 放行
}
func (this AuthM) Mkdir(ctx *App, path string) error {
    Log.Stdout("MKDIR %+v", ctx.Session)
    return ErrNotAllowed  // 拒绝所有 Mkdir 操作
}
```

#### Layer 3：后端级 ACL

部分后端（如 NFS4、S3）在 `Init()` 或操作内部进行额外的 ACL 检查：
- NFS4：支持 `ACL4_SUPPORT_AUDIT_ACL` 和 `ACE4_SYSTEM_AUDIT_ACE_TYPE` 系统级审计
- S3：IAM 权限由 AWS SDK 在 API 调用时检查，失败以 error 形式返回
- Local：依赖操作系统文件权限

### 12.2 审计日志的三类埋点

Filestash 有三类审计日志输出，覆盖不同场景：

#### 1) 认证/登出审计日志

在 `SessionAuthenticate` 和 `SessionLogout` 中通过 `Log.Stdout` 输出，定义于 [server/ctrl/session.go:60-179](server/ctrl/session.go#L60-L179)：

```go
// 认证失败
Log.Stdout("AUDIT action[fail] backend[%s] user[%s] target[%s]", 
    session["type"], backendID(session), ip(req))

// 认证成功
Log.Stdout("AUDIT action[login] backend[%s] user[%s] target[%s]", 
    session["type"], username(session), ip(req))

// 登出
Log.Stdout("AUDIT action[logout] backend[%s] user[%s] target[%s]", 
    ctx.Session["type"], username(ctx.Session), ip(req))
```

#### 2) 授权中间件审计

授权中间件可以自行输出审计日志，如 `plg_authorisation_example`：
```go
func (this AuthM) Ls(ctx *App, path string) error {
    Log.Stdout("LS %+v", ctx.Session)
    return nil
}
```

#### 3) 可插拔审计引擎

通过 `IAuditPlugin` 接口支持完整审计查询，定义于 [server/common/types.go:65-67](server/common/types.go#L65-L67)：

```go
type IAuditPlugin interface {
    Query(ctx *App, searchParams map[string]string) (AuditQueryResult, error)
}
```

**默认实现 `SimpleAudit`**（[server/model/audit.go:58-75](server/model/audit.go#L58-L75)）：
```go
type SimpleAudit struct{}

func (this SimpleAudit) Query(ctx *App, searchParams map[string]string) (AuditQueryResult, error) {
    return AuditQueryResult{
        Form: &AuditForm,
        RenderHTML: `<div id="alert-audit-missing">
            You need to install an audit plugin to use this
        </div>`,
    }, nil
}
```

**审计查询表单字段**（[server/model/audit.go:11-56](server/model/audit.go#L11-L56)）：
- `date from` / `date to`：时间范围过滤
- `action`：操作类型（rename、list、download、create_folder、remove、move、save_file、create_file）
- `path`：文件路径
- `backend`：后端类型
- `session`：会话 ID
- `share`：共享链接 ID
- `user`：用户名
- `target`：操作目标

**查询端点**（[server/routes.go:48](server/routes.go#L48)）：
```go
admin.HandleFunc("/audit", NewMiddlewareChain(FetchAuditHandler, middlewares)).Methods("GET")
```

### 12.3 AUDIT Context 标记与旁路

`ctx.Context` 中的 `AUDIT` key 用于标记某些操作是否需要审计：

```go
// 禁用审计（如内部递归调用）
ctx.Context = context.WithValue(ctx.Context, "AUDIT", false)

// 恢复审计
ctx.Context = context.WithValue(ctx.Context, "AUDIT", nil)
```

**使用场景**（[server/ctrl/files.go:105-125](server/ctrl/files.go#L105-L125)）：
```go
// 读取父目录元数据时禁用审计，避免产生大量 noise
ctx.Context = context.WithValue(ctx.Context, "AUDIT", false)
parent, err := ctx.Backend.Stat(abspath)
ctx.Context = context.WithValue(ctx.Context, "AUDIT", nil)
```

**检查点**：
- 搜索爬虫（`plg_search_sqlitefts`）：索引文件时跳过审计
- Workflow 触发器（`pkg/workflow/trigger/fileaction.go`）：自动化操作跳过审计

### 12.4 权限链路的短路语义

| 层级 | 失败行为 | 错误返回 |
|------|---------|---------|
| CanRead/CanEdit | 立即终止 | `ErrPermissionDenied` (403) |
| AuthorisationMiddleware | 按顺序，第一个失败即终止 | 中间件返回的 error（可自定义） |
| 后端 ACL | 操作失败返回 error | 后端 SDK 或系统返回的 error |

**示例：FileCat 完整权限检查**（[server/ctrl/files.go:208-225](server/ctrl/files.go#L208-L225)）：
```go
func FileCat(ctx *App, res http.ResponseWriter, req *http.Request) {
    // Layer 1: 粗粒度
    if model.CanRead(ctx) == false {
        SendErrorResult(res, ErrPermissionDenied)
        return
    }
    // ... 路径处理 ...
    
    // Layer 2: 授权中间件链
    for _, auth := range Hooks.Get.AuthorisationMiddleware() {
        if err = auth.Cat(ctx, path); err != nil {
            SendErrorResult(res, err)
            return
        }
    }
    
    // Layer 3: 后端实际操作（可能触发后端 ACL）
    file, err = ctx.Backend.Cat(path)
}
```

## 13. 插件间的级联依赖问题处理

### 13.1 插件初始化的两个阶段

Filestash 的插件初始化分为**两个独立阶段**，解决不同类型的依赖：

```
编译/启动阶段 1: init() 函数执行
    顺序 = server/plugin/index.go 中的 import 顺序
    职责：接口注册（Backend.Register、Hooks.Register.Xxx）
    禁止：网络调用、文件 IO、依赖其他插件的初始化结果
    
启动阶段 2: Onload 回调执行（main.go:34-36）
    顺序 = Hooks.Register.Onload() 的注册顺序
    职责：数据库初始化、表创建、配置加载
    可以：访问其他插件的注册结果
```

### 13.2 阶段 1：init() 的顺序依赖

所有存储后端和大多数插件通过 `init()` 注册，执行顺序由 **Go import 顺序**决定。

`server/plugin/index.go` 中的 import 顺序即执行顺序（[server/plugin/index.go:3-46](server/plugin/index.go#L3-L46)）：

```go
import (
    _ "github.com/mickael-kerjean/filestash/server/plugin/plg_backend_local"   // 1
    _ "github.com/mickael-kerjean/filestash/server/plugin/plg_backend_s3"      // 2
    _ "github.com/mickael-kerjean/filestash/server/plugin/plg_backend_sftp"    // 3
    _ "github.com/mickael-kerjean/filestash/server/plugin/plg_backend_ftp"     // 4
    _ "github.com/mickael-kerjean/filestash/server/plugin/plg_backend_dav"     // 5
    // ...
    _ "github.com/mickael-kerjean/filestash/server/plugin/plg_search_sqlitefts" // 较后
    _ "github.com/mickael-kerjean/filestash/server/plugin/plg_authorisation_example" // 最后
)
```

**依赖保障**：
- **存储后端插件之间无依赖**：每个后端独立注册到 `Driver.ds` map，互不影响
- **功能插件可以依赖后端接口**：在 `init()` 中只能注册，不能调用后端 `Init()`
- **基础插件排在前**：后端、认证、授权排在 import 列表前面
- **扩展插件排在后**：搜索、缩略图、工作流等排在后面

**循环依赖风险**：
- 由于 `init()` 只能做注册，不能互相调用，因此**不存在循环依赖问题**
- 如果 A 和 B 互相需要对方在 `init()` 中注册，Go 的 import 机制会先完成所有 import 再执行 init()，因此注册是全局可见的

### 13.3 阶段 2：Onload 回调与顺序控制

`Onload` 回调在所有 `init()` 完成后、HTTP 服务启动前执行，定义于 [cmd/main.go:34-36](cmd/main.go#L34-L36)：

```go
// main.go 启动流程
check(extension.Discovery(), "Plugin Discovery failed")  // 先发现外部插件
check(workflow.Init(), "Workflow init failed")           // 工作流初始化
for _, fn := range Hooks.Get.Onload() {                  // 执行所有 Onload 回调
    fn()
}
for _, obj := range Hooks.Get.HttpEndpoint() {           // 注册 HTTP 端点
    obj(router)
}
Hooks.Get.Starter()(withSignal(), router)                // 启动 HTTP 服务
```

**Onload 注册机制**（[server/common/plugin.go:268-275](server/common/plugin.go#L268-L275)）：
```go
var afterload []func()

func (this Register) Onload(fn func()) {
    afterload = append(afterload, fn)  // 按调用顺序 append
}

func (this Get) Onload() []func() {
    return afterload  // 返回整个 slice 顺序执行
}
```

**典型 Onload 用例**：

1. **数据库表初始化**（[server/plugin/plg_widget_recent/db.go:13](server/plugin/plg_widget_recent/db.go#L13)）：
```go
Hooks.Register.Onload(func() {
    db := GetDB()
    db.Exec(`CREATE TABLE IF NOT EXISTS "recent" (
        "id" INTEGER PRIMARY KEY AUTOINCREMENT,
        "path" TEXT, "user_id" TEXT, "created_at" DATETIME
    )`)
})
```

2. **配置校验**（[server/plugin/plg_starter_http2/index.go:34](server/plugin/plg_starter_http2/index.go#L34)）：
```go
Hooks.Register.Onload(func() {
    if Config.Get("general.force_ssl").Bool() {
        // 检查证书配置
    }
})
```

3. **依赖其他插件**（[server/plugin/plg_search_sqlitefts/index.go:13](server/plugin/plg_search_sqlitefts/index.go#L13)）：
```go
Hooks.Register.Onload(register)  // 确保 SearchEngine 注册完成后再启动爬虫

func register() {
    // 此时可以安全调用 Hooks.Get.SearchEngine()
    daemon := &CrawlerDaemon{}
    Hooks.Register.AuthorisationMiddleware(crawler.FileHook{Daemon: daemon})
    daemon.Start()
}
```

### 13.4 特殊依赖场景的处理

#### Starter 插件：强制依赖检查

Starter（HTTP 服务器）是唯一有强制依赖检查的插件，定义于 [cmd/main.go:31-33](cmd/main.go#L31-L33)：

```go
if Hooks.Get.Starter() == nil {
    check(ErrNotFound, "Missing starter plugin. err=%s")  // 直接退出程序
}
```

如果没有任何插件调用 `Hooks.Register.Starter(fn)`，程序无法启动。

#### Workflow 排序：Order 字段

Workflow Trigger 和 Action 支持显式 `Order` 字段控制执行顺序，定义于 [server/common/plugin.go:341-343](server/common/plugin.go#L341-L343)：

```go
func (this Register) WorkflowTrigger(t ITrigger) {
    workflow_triggers = append(workflow_triggers, t)
    sort.Slice(workflow_triggers, func(i, j int) bool {
        return workflow_triggers[i].Manifest().Order < workflow_triggers[j].Manifest().Order
    })
}
```

插件在 `Manifest()` 中返回 Order 整数：
```go
func (this FileActionTrigger) Manifest() Manifest {
    return Manifest{
        Order: 10,  // 数字越小越先执行
        ID:    "file_action",
    }
}
```

#### 目录服务：单例覆盖

`DirectoryService` 采用**后注册覆盖先注册**策略，定义于 [server/common/plugin.go:363-369](server/common/plugin.go#L363-L369)：

```go
var directory IDirectoryService

func (this Register) DirectoryService(d IDirectoryService) {
    directory = d  // 直接赋值，后注册覆盖先注册
}
```

如果有多个 LDAP/AD 插件，最后 import 的生效。

### 13.5 级联失败的处理策略

| 依赖类型 | 失败处理 | 影响范围 |
|---------|---------|---------|
| **存储后端 init() panic** | Go 运行时直接崩溃 | 程序无法启动 |
| **存储后端 init() 正常注册** | 无失败（仅 map 赋值） | 无影响 |
| **外部 Wasm 插件加载失败** | 记录日志 + `continue` | 仅该插件不可用 |
| **Onload 回调 panic** | Go 运行时崩溃 | 程序无法启动 |
| **Onload 回调返回 error** | 无（Onload 无返回值，需自行捕获） | 取决于具体逻辑 |
| **缺失 Starter 插件** | 显式 `check(ErrNotFound)` | 程序无法启动 |
| **Workflow Init 失败** | `check(err)` 退出 | 程序无法启动 |

**最佳实践**：
- `init()` 中只做 `Hooks.Register.Xxx()` 等无副作用操作
- 可能失败的操作（网络、文件 IO）放在 `Onload` 中，自行捕获 panic
- 依赖顺序通过 `import` 顺序和 `Order` 字段显式控制
- 避免在 `init()` 中创建 goroutine 或打开资源

## 14. 插件运行时的性能监控与慢请求追踪

### 14.1 HTTP 层性能监控

`telemetry` 中间件在每个 HTTP 请求结束时记录性能指标，定义于 [server/middleware/telemetry.go:15-106](server/middleware/telemetry.go#L15-L106)：

**日志条目结构**：
```go
type LogEntry struct {
    Host       string  `json:"host"`
    Method     string  `json:"method"`
    RequestURI string  `json:"pathname"`
    Status     int     `json:"status"`
    Duration   float64 `json:"responseTime"`  // 毫秒
    Backend    string  `json:"backend"`       // s3, sftp, local...
    Share      string  `json:"share"`         // 共享链接 ID
    Session    string  `json:"session"`       // 会话哈希
    RequestID  string  `json:"requestID"`     // X-Request-ID
    Ip         string  `json:"ip"`
    UserAgent  string  `json:"userAgent"`
    Version    string  `json:"version"`       // 应用版本
}
```

**输出控制**（[server/middleware/telemetry.go:82-91](server/middleware/telemetry.go#L82-L91)）：
```go
if Config.Get("log.telemetry").Bool() {
    telemetry.Record(point)  // 保存到内存，定期批量上报
}
if Config.Get("log.enable").Bool() {
    tid := ""
    if point.RequestID != "" && Config.Get("log.level").String() == "DEBUG" {
        tid = "trace=" + point.RequestID  // DEBUG 级别输出 trace ID
    }
    Log.Stdout("HTTP %3d %3s %6.1fms %s %s", 
        point.Status, point.Method, point.Duration, limit(point.RequestURI, 200), tid)
}
```

**批量遥测上报**（[server/middleware/telemetry.go:108-132](server/middleware/telemetry.go#L108-L132)）：
```go
func (this *Telemetry) Flush() {
    // 批量 POST 到 https://downloads.filestash.app/event
    // 用于统计使用情况，可通过 log.telemetry = false 关闭
}
```

### 14.2 OpenTracing 兼容的分布式追踪

Filestash 实现了轻量级 OpenTracing 兼容的追踪框架，核心在 `server/pkg/tracer/`。

#### 追踪核心接口

定义于 [server/pkg/tracer/types.go:1-20](server/pkg/tracer/types.go#L1-L20)：

```go
type ITracer = func(TraceContext, string, SpanOptions) ISpan

type ISpan interface {
    SetError(error)    // 标记错误
    Close()            // 结束 span
    TraceContext() TraceContext
}

type SpanOptions struct {
    Kind       string            // "SERVER" / "CLIENT"
    Service    string            // "sftp", "s3", "http"
    Attributes map[string]string // 自定义标签
}
```

**注册机制**（[server/common/plugin.go:198-199](server/common/plugin.go#L198-L199)）：
```go
func (this Register) Tracer(t ITracer) {
    tracer.Register(t)
}
```

默认实现是 `Nop()` 空操作，不产生任何开销：
```go
func StartSpan(parent TraceContext, name string, opts SpanOptions) ISpan {
    if tracer == nil {
        return Nop()  // 无插件注册时，零开销
    }
    return tracer(parent, name, opts)
}
```

#### SFTP 后端的追踪埋点

SFTP 后端通过装饰器模式完整追踪每个协议操作，定义于 [server/plugin/plg_backend_sftp/tracing.go:13-124](server/plugin/plg_backend_sftp/tracing.go#L13-L124)：

```go
type tracedClient struct {
    *sftp.Client
    app      *App
    hostname string
    username string
}

func (t *tracedClient) ReadDir(path string) ([]os.FileInfo, error) {
    span := NewSpan(t.app, "ReadDir", connAttrs(t, map[string]string{
        "sftp.path":   path,
        "sftp.packet": "SSH_FXP_READDIR",
    }))
    defer span.Close()
    
    files, err := t.Client.ReadDir(path)  // 实际调用
    span.SetError(err)                    // 标记错误
    return files, err
}
```

**文件级追踪**（[server/plugin/plg_backend_sftp/tracing.go:48-62](server/plugin/plg_backend_sftp/tracing.go#L48-L62)）：
```go
type tracedFile struct {
    *sftp.File
    client *tracedClient
    path   string
    span   tracer.ISpan  // 整个文件生命周期的 span
}

func (t *tracedFile) Close() error {
    t.span.Close()  // 结束文件读写 span
    // 额外创建 Close 操作 span
    closeSpan := NewSpan(t.client.app, "Close", ...)
    err := t.File.Close()
    closeSpan.SetError(err)
    closeSpan.Close()
    return err
}
```

**追踪上下文传递**：
```go
func NewSpan(app *App, name string, attrs map[string]string) tracer.ISpan {
    // 从 HTTP 请求 context 中提取 TraceContext
    return tracer.StartSpan(tracer.TraceFromContext(app.Context), name, ...)
}
```

#### S3 后端的追踪埋点

S3 通过 HTTP `RoundTripper` 拦截器实现追踪，定义于 [server/plugin/plg_backend_s3/utils.go:8-45](server/plugin/plg_backend_s3/utils.go#L8-L45)：

```go
type S3Transport struct {
    Transport http.RoundTripper
}

func (t S3Transport) RoundTrip(r *http.Request) (*http.Response, error) {
    var span tracer.ISpan
    if r.Context() != nil {
        opts := tracer.SpanOptions{
            Kind:    tracer.KindClient,
            Service: "s3",
            Attributes: map[string]string{
                "s3.operation": r.Operation.Name,  // ListObjects, PutObject...
                "s3.bucket":    r.URL.Host,
                "s3.key":       r.URL.Path,
            },
        }
        span = tracer.StartSpan(tracer.TraceFromContext(r.Context()), 
            r.Operation.Name, opts)
    }
    
    resp, err := t.Transport.RoundTrip(r)
    if span != nil {
        span.SetError(err)
        span.Close()
    }
    return resp, err
}
```

#### HTTP 客户端统一追踪

所有后端共享的 `HTTPClient` 自动集成追踪，定义于 [server/common/default.go:57-59](server/common/default.go#L57-L59)：

```go
return &http.Client{
    Transport: tracer.NewTransport(cfg.traceService, 
        NewTransformedTransport(cfg.transport)),
    Timeout: 5 * time.Hour,
}
```

`tracer.NewTransport` 为所有 HTTP 请求自动创建 span。

### 14.3 慢请求识别与排查

#### 超时配置分层

| 层级 | 超时设置 | 作用 |
|------|---------|------|
| **TCP 连接** | 10s | [server/common/default.go:45](server/common/default.go#L45) |
| **TLS 握手** | 5s | [server/common/default.go:48](server/common/default.go#L48) |
| **响应头** | 60s | [server/common/default.go:50](server/common/default.go#L50) |
| **空闲连接** | 60s | [server/common/default.go:49](server/common/default.go#L49) |
| **总请求** | 5 小时 | [server/common/default.go:58](server/common/default.go#L58) |
| **SFTP 操作** | 依赖 TCP 超时 | 无应用层超时，通过 context 取消 |

#### 慢请求排查路径

当用户报告慢操作时，按以下路径排查：

```
1. 检查 HTTP 日志 → 找到 Duration > 阈值的请求
   grep "HTTP.* [0-9]{4,}\.ms" app.log
   
2. 启用 DEBUG 日志 → 获取 X-Request-ID
   log.level = DEBUG
   
3. 启用 Tracer 插件 → 查看分布式追踪详情
   - 注册 tracer 到 Jaeger/Zipkin
   - 分析各子 span 耗时（ReadDir vs Open vs Read 等）
   
4. 后端特定排查：
   SFTP: 查看各 SSH_FXP_* 操作耗时
   S3:   查看 S3 API 调用耗时 + 重试次数
   FTP:  查看连接建立 + 数据通道耗时
```

#### 上下文取消机制

每个请求的 `ctx.Context` 会在连接断开时自动取消，所有阻塞操作应该监听 `Done()`：

```go
// SFTP 缓存引用计数示例
go func() {
    <-app.Context.Done()  // 客户端断开或超时
    d.wg.Done()            // 释放连接引用
}()
```

### 14.4 性能监控的可扩展性

| 监控维度 | 实现方式 | 扩展点 |
|---------|---------|--------|
| **HTTP 请求指标** | Telemetry 中间件 | 通过 `log.telemetry` 配置关闭/开启 |
| **分布式追踪** | OpenTracing 兼容框架 | 注册自定义 `Tracer` 到 Jaeger/Zipkin |
| **后端操作指标** | 各后端插件自行埋点 | SFTP/S3 已完整埋点，其他后端可参考实现 |
| **自定义指标** | 插件自行实现 | 通过 `Hooks.Register.Middleware()` 添加 |
| **告警** | 无内置 | 基于日志/追踪数据在外部系统配置 |

## 15. 完整调用链示例

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
    ├→ Layer 1: model.CanRead(ctx) → true
    ├→ PathBuilder(ctx, "/documents") → "/bucket/documents"
    ├→ Layer 2: auth[0].Ls(ctx, path) → nil
    │   Layer 2: auth[1].Ls(ctx, path) → nil (遍历所有授权中间件)
    ├→ Layer 3: ctx.Backend.Ls("/bucket/documents")
    │   └→ S3Transport.RoundTrip()
    │       ├→ NewSpan("ListObjectsV2", {s3.bucket, s3.key})
    │       ├→ AWS SDK 实际调用
    │       └→ span.Close()
    ├→ 结果转换为 FileInfo 数组
    └→ SendSuccessResultsWithMetadata(...)
    ↓
Telemetry 中间件
    └→ Log.Stdout("HTTP 200 GET 123.4ms /api/ls?path=/documents")
```

## 16. 相关文件速查表

| 文件 | 职责 |
|------|------|
| [server/common/types.go](server/common/types.go) | `IBackend`、`IAuthorisation`、`IAuditPlugin` 接口定义 |
| [server/common/backend.go](server/common/backend.go) | `Driver` 注册中心、`Nothing` 空实现、降级回退 |
| [server/common/plugin.go](server/common/plugin.go) | `Hooks` 扩展机制、Onload/OnQuit/Workflow 排序 |
| [server/common/crypto.go](server/common/crypto.go) | AES-GCM 加密、zlib 压缩、会话 ID、Nonce 生成器 |
| [server/common/constants.go](server/common/constants.go) | 密钥分层派生、Cookie 路径配置 |
| [server/common/config.go](server/common/config.go) | Configuration.Initialise() 主密钥初始化 |
| [server/common/config_state.go](server/common/config_state.go) | 配置文件加解密、configKeysToEncrypt |
| [server/common/cache.go](server/common/cache.go) | 会话绑定缓存、并发安全 |
| [server/common/default.go](server/common/default.go) | HTTP 客户端、TLS 标准化配置、超时设置 |
| [server/plugin/index.go](server/plugin/index.go) | 所有插件导入入口（编译时链接，决定 init 顺序） |
| [server/model/files.go](server/model/files.go) | `NewBackend()` 工厂、isAllowed 配置校验 |
| [server/model/permissions.go](server/model/permissions.go) | CanRead/CanEdit/CanUpload 粗粒度权限 |
| [server/model/audit.go](server/model/audit.go) | SimpleAudit 默认实现、AuditForm 查询表单 |
| [server/middleware/session.go](server/middleware/session.go) | 中间件注入 `ctx.Backend`、会话解密 |
| [server/middleware/telemetry.go](server/middleware/telemetry.go) | HTTP 请求性能监控、日志输出、遥测上报 |
| [server/pkg/tracer/](server/pkg/tracer/) | 分布式追踪框架（OpenTracing 兼容） |
| [server/ctrl/session.go](server/ctrl/session.go) | 会话认证、Cookie 分片、OAuthToken、登录登出审计 |
| [server/ctrl/admin.go](server/ctrl/admin.go) | 管理员 Cookie 加密、Backend.Drivers()、/audit 端点 |
| [server/ctrl/files.go](server/ctrl/files.go) | 控制器方法分发、授权中间件调用、PathBuilder |
| [server/plugin/plg_backend_sftp/tracing.go](server/plugin/plg_backend_sftp/tracing.go) | SFTP 全操作追踪埋点（装饰器模式） |
| [server/plugin/plg_backend_s3/utils.go](server/plugin/plg_backend_s3/utils.go) | S3 HTTP RoundTripper 追踪埋点 |
| [server/plugin/plg_authorisation_example/](server/plugin/plg_authorisation_example/) | 授权中间件示例（审计日志+权限控制） |
| [server/pkg/extension/discovery.go](server/pkg/extension/discovery.go) | 外部 Wasm 插件发现与加载、失败隔离 |
| [server/pkg/extension/adapter/runtime/](server/pkg/extension/adapter/runtime/) | Wasm 沙箱运行时、互斥锁保护 |
| [server/plugin/plg_backend_local/](server/plugin/plg_backend_local/) | Local 后端实现（Admin 密码认证） |
| [server/plugin/plg_backend_tmp/](server/plugin/plg_backend_tmp/) | TmpStorage 用户 Chroot 目录隔离 |
| [server/plugin/plg_backend_samba/](server/plugin/plg_backend_samba/) | Samba 多共享虚拟命名空间 |
| [server/plugin/plg_backend_*/index.go](server/plugin/) | 各后端具体实现 |
| [cmd/main.go](cmd/main.go) | 主程序启动流程、Onload 执行顺序、Starter 依赖检查 |
| [go.mod](go.mod) | 外部依赖版本锁定 |
