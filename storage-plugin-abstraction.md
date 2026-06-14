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

## 6. 完整调用链示例

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

## 7. 相关文件速查表

| 文件 | 职责 |
|------|------|
| [server/common/types.go](server/common/types.go) | `IBackend` 接口定义 |
| [server/common/backend.go](server/common/backend.go) | `Driver` 注册中心、`Nothing` 空实现 |
| [server/common/plugin.go](server/common/plugin.go) | `Hooks` 扩展机制 |
| [server/plugin/index.go](server/plugin/index.go) | 所有插件导入入口 |
| [server/model/files.go](server/model/files.go) | `NewBackend()` 工厂函数 |
| [server/middleware/session.go](server/middleware/session.go) | 中间件注入 `ctx.Backend` |
| [server/ctrl/files.go](server/ctrl/files.go) | 控制器方法分发 |
| [server/plugin/plg_backend_*/index.go](server/plugin/) | 各后端具体实现 |
