# Filestash 插件生命周期：注册 → 加载 → 钩子 → 路由 → 卸载/热重启

> 以代码执行顺序为主线，梳理隔离边界、配置注入与错误回收的实现细节。

---

## 一、总体架构概览

Filestash 的插件系统分为**两条互补的加载管线**：

| 管线 | 入口 | 插件形式 | 加载时机 |
|------|------|----------|----------|
| **编译期内置插件** | `server/plugin/index.go` 侧加载（`_ import`） | Go 源码包 | Go `init()` 阶段，进程启动前 |
| **运行期外部插件** | `server/pkg/extension/discovery.go` | `.zip` 包（含 `manifest.json` + 资源 + 可选 WASM） | `main()` 显式调用 `extension.Discovery()` |

两条管线共享同一套**钩子注册表** `common.Hooks`，最终在 `cmd/main.go:Run()` 中统一完成生命周期编排。

---

## 二、启动时序与代码链路

下面是 `cmd/main.go:Run()` 的完整执行顺序，标注了每一步对应的源文件：

```
1. InitLogger()                          → common/log.go
2. InitConfig()                          → common/config.go          —— 配置加载
3. extension.Discovery()                 → pkg/extension/discovery.go —— 外部插件发现
4. ctrl.InitPluginList(...)              → ctrl/plugin.go            —— 插件列表初始化
5. workflow.Init()                       → pkg/workflow/index.go     —— 工作流引擎初始化
6. Hooks.Get.Starter() 安全检查           → common/plugin.go          —— 必须存在 starter
7. Hooks.Get.Onload() 逐个调用           → common/plugin.go:270      —— 所有插件的 Onload 回调
8. Hooks.Get.HttpEndpoint() 注册路由      → common/plugin.go:64       —— 插件自定义 HTTP 端点
9. server.Build(router)                  → server/routes.go          —— 核心业务路由
10. server.PluginRoutes(router)          → server/routes.go:158      —— 前端覆盖路由 + xdg-open
11. server.CatchAll(router)              → server/routes.go:120      —— 兜底路由
12. Hooks.Get.Starter()(ctx, router)     → plg_starter_http          —— 启动 HTTP 服务器
13. Hooks.Get.OnQuit() 逐个调用          → common/plugin.go:277      —— 进程退出清理
```

### 关键细节

- **步骤 2 中**，`Config.Load()` 末尾会触发 `Hooks.Get.OnConfig()` 回调（`common/config.go:215`），即配置变更钩子。
- **步骤 3** 在步骤 2 之后，确保外部插件发现时配置已就绪。
- **步骤 7** 是所有插件"二次初始化"的时机——`init()` 仅注册钩子，`Onload` 才执行依赖配置的逻辑。
- **步骤 12** 阻塞运行，直到收到 `SIGTERM`/`SIGINT` 信号（`withSignal()` 函数）。

---

## 三、编译期内置插件的注册

### 3.1 注册机制

```
server/plugin/index.go  —— 通过 _ import 把所有内置插件拉入编译
    ↓
各插件的 init() 函数  —— Go 运行时在 main() 前自动调用
    ↓
调用 Hooks.Register.Xxx() 或 Backend.Register()  —— 写入全局注册表
```

### 3.2 两大注册目标

| 注册目标 | 数据结构 | 典型调用 | 源文件位置 |
|----------|----------|----------|-----------|
| `Hooks.Register.*` | `common/plugin.go` 中的包级变量 | `Hooks.Register.AuthenticationMiddleware("admin", Admin{})` | `plugin.go:129` |
| `Backend.Register(name, driver)` | `common/backend.go` 中的 `Driver.ds map[string]IBackend` | `Backend.Register("s3", S3Backend{})` | `backend.go:21` |

### 3.3 内置插件分类与注册示例

**存储后端（Backend）** — 注册到 `Backend` 全局 Driver：

```go
// server/plugin/plg_backend_s3/index.go:38
func init() {
    Backend.Register("s3", S3Backend{})
}
```

所有后端必须实现 `IBackend` 接口（`common/types.go:13`），包含 `Init`、`Ls`、`Cat`、`Save` 等方法。`LoginForm()` 返回的 `Form` 结构既定义了连接参数 Schema，也用于前端渲染登录表单。

**认证中间件（AuthenticationMiddleware）** — 注册到 `Hooks.Register`：

```go
// server/plugin/plg_authenticate_admin/index.go:13
func init() {
    Hooks.Register.AuthenticationMiddleware("admin", Admin{})
}
```

必须实现 `IAuthentication` 接口（`common/types.go:26`），包含 `Setup()`、`EntryPoint()`、`Callback()` 三个方法。

**HTTP 端点（HttpEndpoint）** — 注册自定义路由：

```go
// server/plugin/plg_handler_mcp/index.go:23
Hooks.Register.HttpEndpoint(func(r *mux.Router) error {
    r.HandleFunc("/sse", NewMiddlewareChain(srv.sseHandler, m))
    // ...
    return nil
})
```

**启动器（Starter）** — 注册服务器启动函数：

```go
// server/plugin/plg_starter_http/index.go:14
func init() {
    Hooks.Register.Starter(Start)
}
```

Starter 是**单例**——`starter_process` 变量只保留最后一个注册值（`plugin.go:110`），不像其他钩子是追加模式。

---

## 四、运行期外部插件的发现与加载

### 4.1 发现流程

```
extension.Discovery()                           // discovery.go:15
    ↓ os.ReadDir(PLUGIN_PATH)                   // 读取 state/plugins/ 目录
    ↓ 过滤 .zip 文件
    ↓ initModule(fname)                         // discovery.go:112
        ↓ zip.OpenReader(fname)
        ↓ 在 zip 中查找 manifest.json
        ↓ json.Decode → PluginImpl{Author, Version, Modules[]}
    ↓ 遍历 impl.Modules，按 type 分发：
        ├─ "css"          → Hooks.Register.CSS(string(b))
        ├─ "patch"        → Hooks.Register.StaticPatch(b)
        ├─ "favicon"      → Hooks.Register.Favicon(b)
        ├─ "middleware"   → adapter.MiddlewareExtension(b) → Hooks.Register.Middleware(m)
        ├─ "workflow::action" → adapter.WorkflowActionExtension(b) → Hooks.Register.WorkflowAction(a)
        ├─ "xdg-open"    → 仅记录到 plugins map，不立即注册钩子
        └─ 其他           → 返回 ErrNotImplemented
    ↓ plugins[name] = impl                     // 记入全局插件注册表
```

**源文件**：`server/pkg/extension/discovery.go`

### 4.2 manifest.json 格式

```json
{
    "author": "Filestash Pty Ltd",
    "version": "v0.0",
    "modules": [
        {
            "type": "middleware",
            "entrypoint": "middleware.wasm"
        },
        {
            "type": "xdg-open",
            "mime": "application/word",
            "entrypoint": "loader_lowa.js",
            "application": "skeleton"
        }
    ]
}
```

一个 zip 可以包含**多个 module**，每个 module 有独立 type 和 entrypoint。

### 4.3 WASM 运行时隔离

外部 `middleware` 和 `workflow::action` 类型的模块使用 WASM 沙箱执行：

```
adapter.MiddlewareExtension(wasmBytes)                    // adapter/middleware.go:20
    ↓ runtime.New(wasm, middlewareExports())              // adapter/runtime/runtime.go:21
        ↓ wazero.NewRuntime(ctx)                          —— 创建纯 Go WASM 运行时
        ↓ wasi_snapshot_preview1.MustInstantiate(...)      —— 注入 WASI 支持
        ↓ WithExports(build)                              —— 注册宿主函数
            ↓ NewHostModuleBuilder(wrt, "env")            —— 构建 "env" 模块
            ↓ b.Export("req_method", fn)                  —— 逐个导出宿主 API
            ↓ b.Instantiate(ctx)                          —— 实例化宿主模块
        ↓ wrt.CompileModule(ctx, wasm)                    —— 编译 WASM 二进制
        ↓ wrt.InstantiateModule(ctx, compiled, ...)       —— 实例化插件模块
    ↓ 返回 *Runtime{wrt, ctx, mod}
```

**中间件宿主 API**（`adapter/middleware.go:40-77`）：

| 导出函数 | 功能 | 方向 |
|----------|------|------|
| `req_method` | 读取请求方法 | 宿主 → WASM |
| `req_path` | 读取请求路径 | 宿主 → WASM |
| `req_header_get` | 读取请求头 | 宿主 → WASM |
| `req_body_read` | 读取请求体 | 宿主 → WASM |
| `resp_status` | 设置响应状态码 | WASM → 宿主 |
| `resp_header` | 设置响应头 | WASM → 宿主 |
| `resp_write` | 写入响应体 | WASM → 宿主 |
| `middleware_next` | 放行到下一中间件 | WASM → 宿主 |

**Workflow Action 宿主 API**（`adapter/workflow.go:33-63`）：

| 导出函数 | 功能 |
|----------|------|
| `workflow_manifest_set` | WASM 报告自己的 manifest |
| `workflow_params_get` | WASM 读取执行参数 |
| `workflow_input_get` | WASM 读取触发输入 |
| `workflow_output_set` | WASM 写出执行结果 |
| `workflow_error_set` | WASM 报告执行错误 |

---

## 五、事件钩子系统详解

### 5.1 钩子注册表一览

所有钩子定义在 `server/common/plugin.go`，通过 `Hooks.Register.*` 写入，`Hooks.Get.*` 读取：

| 钩子名 | 类型 | 注册模式 | 用途 |
|--------|------|----------|------|
| `ProcessFileContentBeforeSend` | `[]func(...)` | 追加 | 文件内容发送前处理（图片转码、安全过滤） |
| `HttpEndpoint` | `[]func(*mux.Router) error` | 追加 | 注册自定义 HTTP 路由 |
| `Static` | 委托 HttpEndpoint | 追加 | 静态资源覆盖 |
| `Starter` | `func(context.Context, *mux.Router)` | **单例覆盖** | 服务器启动入口 |
| `AuthenticationMiddleware` | `map[string]IAuthentication` | 按 ID 注册 | 身份认证插件 |
| `AuthorisationMiddleware` | `[]IAuthorisation` | 追加 | 权限校验插件 |
| `SearchEngine` | `ISearch` | **单例覆盖** | 搜索引擎 |
| `Thumbnailer` | `map[string]IThumbnailer` | 按 mimeType 注册 | 缩略图生成器 |
| `AuditEngine` | `IAuditPlugin` | **单例覆盖** | 审计引擎 |
| `Tracer` | `ITracer` | 委托 tracer 包 | 链路追踪 |
| `FrontendOverrides` | `[]string` | 追加 | 前端 JS 覆盖 URL |
| `XDGOpen` | `[]string` | 追加 | 文件类型打开器（JS 代码片段） |
| `CSS` | `map[string]string` | 按 ID 幂等注册 | 自定义样式表 |
| `Favicon` | `struct{binary, mime}` | **单例覆盖** | 站点图标 |
| `Onload` | `[]func()` | 追加 | 初始化回调 |
| `OnQuit` | `[]func()` | 追加 | 退出清理回调 |
| `OnConfig` | `[]func()` | 追加 | 配置变更回调 |
| `Middleware` | `[]func(HandlerFunc)HandlerFunc` | 追加 | 全局 HTTP 中间件 |
| `StaticPatch` | `map[string][]byte` | 按 ID 幂等注册 | 前端 JS 补丁 |
| `Metadata` | `IMetadata` | **单例覆盖** | 元数据服务 |
| `WorkflowTrigger` | `[]ITrigger` | 追加+排序 | 工作流触发器 |
| `WorkflowAction` | `[]IAction` | 追加+排序 | 工作流动作 |
| `DirectoryService` | `IDirectoryService` | **单例覆盖** | 目录服务 |

### 5.2 幂等注册机制

`CSS` 和 `StaticPatch` 钩子支持幂等注册——相同 ID 不会重复追加：

```go
// plugin.go:226-235
func (this Register) CSS(stylesheet string, opts ...Option) {
    options := Options{}
    for _, opt := range opts {
        opt(&options)
    }
    if options.ID == "" {
        options.ID = QuickHash(stylesheet, 10)    // 自动生成 ID
    }
    cssOverride[options.ID] = stylesheet           // map 天然幂等
}
```

外部调用者可通过 `WithID("my-id")` Option 指定 ID，也可省略让系统按内容哈希生成。

### 5.3 生命周期回调的执行时机

```
OnConfig  → Config.Load() 末尾 (config.go:215)
           → 每次管理员保存配置时再次触发
Onload    → Run() 步骤 7 (main.go:34)
           → 仅进程启动时执行一次
OnQuit    → Run() 末尾 (main.go:47)
           → 收到 SIGTERM/SIGINT 后执行
```

---

## 六、请求路由与插件参与方式

### 6.1 中间件链

每个 HTTP 路由通过 `NewMiddlewareChain` 构建执行链（`middleware/index.go:21`）：

```go
func NewMiddlewareChain(fn HandlerFunc, m []Middleware) http.HandlerFunc {
    return func(res http.ResponseWriter, req *http.Request) {
        var resw ResponseWriter = NewResponseWriter(res)
        var f func(*App, http.ResponseWriter, *http.Request) = fn
        for i := len(m) - 1; i >= 0; i-- {
            f = m[i](f)          // 从后往前包裹
        }
        app := App{Context: req.Context()}
        f(&app, &resw, req)
        // ...
    }
}
```

**关键**：中间件按**逆序包裹**，执行时按**正序**运行（最前面的 Middleware 最先执行）。

### 6.2 PluginInjector — 全局插件中间件注入

几乎每条路由的中间件列表末尾都包含 `PluginInjector`（`middleware/index.go:72`）：

```go
func PluginInjector(fn HandlerFunc) HandlerFunc {
    for _, middleware := range Hooks.Get.Middleware() {
        fn = middleware(fn)          // 逐层包裹
    }
    return fn
}
```

这确保了所有通过 `Hooks.Register.Middleware` 注册的插件中间件（包括 WASM 中间件）会自动应用到每条路由。

### 6.3 典型路由的中间件链

以文件列表 API 为例（`server/routes.go:62`）：

```
GET /api/files/ls
  → ApiHeaders       (设置 Content-Type: application/json)
  → SecureHeaders    (HSTS, X-Content-Type-Options)
  → SessionStart     (提取 Share/Session/Backend)
  → LoggedInOnly     (检查登录状态)
  → PluginInjector   (注入所有 Hooks.Register.Middleware)
      → [WASM middleware 1]
      → [WASM middleware 2]
      → ...
  → FileLs           (实际处理函数)
```

### 6.4 插件自定义端点

通过 `Hooks.Register.HttpEndpoint` 注册的路由在 `Run()` 步骤 8 中注入（`main.go:37`）：

```go
for _, obj := range Hooks.Get.HttpEndpoint() {
    obj(router)
}
```

插件自行决定路由路径和中间件链。例如 MCP 插件（`plg_handler_mcp/index.go`）：

```go
Hooks.Register.HttpEndpoint(func(r *mux.Router) error {
    if !PluginEnable() { return nil }
    r.HandleFunc("/sse", NewMiddlewareChain(srv.sseHandler, []Middleware{WithCORS}))
    r.HandleFunc("/messages", NewMiddlewareChain(srv.messageHandler, []Middleware{WithCORS}))
    // ...
    return nil
})
```

### 6.5 前端插件路由

`server.PluginRoutes()` (routes.go:158) 处理前端覆盖：

1. **FrontendOverrides** — 为每个注册的 URL 创建默认处理器（返回占位注释）：
   ```go
   for _, obj := range Hooks.Get.FrontendOverrides() {
       r.HandleFunc(obj, func(res http.ResponseWriter, req *http.Request) {
           res.Write([]byte(fmt.Sprintf("/* Default '%s' */", obj)))
       })
   }
   ```

2. **XDGOpen** — 生成 `/overrides/xdg-open.js`，将所有 `xdg-open` 钩子合并为一段 JS：
   ```go
   r.HandleFunc(WithBase("/overrides/xdg-open.js"), func(...) {
       res.Write([]byte(`window.overrides["xdg-open"] = function(mime){`))
       for i := 0; i < len(openers); i++ {
           res.Write([]byte(openers[i]))
       }
       res.Write([]byte(`return null;}`))
   })
   ```

3. **插件静态资源** — 直接从 zip 中读取并返回：
   ```
   /assets/{BUILD_REF}/plugin/{name}.zip/{path}  → PluginStaticHandler (ctrl/plugin.go:35)
   /assets/plugin/{name}.zip                      → PluginDownloadHandler (ctrl/plugin.go:72)
   ```

### 6.6 插件导出 API

`GET /api/plugin` 路由（`ctrl/plugin.go:16`）向前端暴露所有 `xdg-open` 类型的插件信息：

```go
func PluginExportHandler(ctx *App, res http.ResponseWriter, req *http.Request) {
    plgExports := map[string][]string{}
    for name, plg := range extension.All() {
        for _, module := range plg.Modules {
            if module["type"] == "xdg-open" {
                plgExports[module["mime"]] = []string{
                    module["application"],
                    WithBase(JoinPath("/assets/"+BUILD_REF+"/plugin/", filepath.Join(name+".zip", index))),
                }
            }
        }
    }
    SendSuccessResultWithEtagAndGzip(res, req, plgExports)
}
```

前端 `public/assets/model/plugin.js` 消费此 API，按 MIME 类型动态加载插件 JS 模块：

```javascript
export function get(mime) { return plugins[mime]; }
export async function load(mime) {
    const specs = plugins[mime];
    const [, url] = specs;
    const module = await import(new URL(url, import.meta.url).href);
    return module.default;
}
```

---

## 七、隔离边界

### 7.1 内置插件 vs 外部插件

| 维度 | 内置插件 | 外部插件 |
|------|----------|----------|
| **代码运行空间** | 与主进程同一 Go 进程 | WASM 沙箱（wazero 纯 Go 运行时） |
| **内存访问** | 共享进程内存 | WASM 线性内存，通过 `IMemory` 接口拷贝传递 |
| **系统调用** | 无限制 | 仅 WASI + 宿主导出函数 |
| **崩溃影响** | 可 panic 整个进程 | WASM 错误返回 error，不崩溃主进程 |
| **注册方式** | `init()` + 直接调用 `Hooks.Register` | `manifest.json` 声明 + `Discovery()` 解析 |

### 7.2 WASM 沙箱的具体隔离机制

**运行时隔离**（`adapter/runtime/runtime.go`）：

1. **独立 Runtime 实例** — 每个外部插件创建独立的 `wazero.Runtime`，模块间不共享状态。
2. **内存拷贝语义** — `wazeroMem.Read()` 读取时做 `copy`，宿主和 WASM 不共享同一片内存：
   ```go
   // runtime/memory.go:17
   func (w wazeroMem) Read(ptr, length uint32) []byte {
       b, ok := w.m.Read(ptr, length)
       cp := make([]byte, len(b))
       copy(cp, b)           // 深拷贝，WASM 无法通过指针反写宿主
       return cp
   }
   ```
3. **互斥锁** — `Runtime.mu sync.Mutex` 保证 `Call()` 串行执行，避免并发调用同一模块：
   ```go
   // runtime/runtime.go:46
   func (r *Runtime) Call(ctx context.Context, fnName string, key, val any) error {
       r.mu.Lock()
       defer r.mu.Unlock()
       // ...
   }
   ```
4. **Context 传递状态** — 通过 `context.WithValue` 传递每次调用的状态对象，而非全局变量：
   ```go
   // adapter/middleware.go:29
   err := rt.Call(r.Context(), "middleware", middlewareKey{}, &middlewareState{r: r, w: w, next: &callNext})
   ```

### 7.3 Backend 隔离

Backend 插件在创建连接时通过 `model.NewBackend()` 进行白名单校验（`model/files.go:9`）：

```go
func NewBackend(ctx *App, conn map[string]string) (IBackend, error) {
    isAllowed := func() bool {
        // 遍历 Config.Conn，检查 type/hostname/path/url 是否匹配
        // 只有配置文件中预先声明的连接才允许创建
    }
    if isAllowed() == false {
        return Backend.Get(BACKEND_NIL), ErrNotAllowed
    }
    return Backend.Get(conn["type"]).Init(conn, ctx)
}
```

这防止了用户利用 Filestash 作为代理访问未授权的存储后端。

---

## 八、配置注入

### 8.1 全局配置注入 — `Config.Get()`

所有插件（内置和外部）都通过 `Config.Get("path.to.key")` 读取配置：

```go
// plg_starter_http/index.go:20
port := Config.Get("general.port").Int()
```

`Config.Get()` 返回 `*ConfigElement`，支持链式调用 `.String()` / `.Int()` / `.Bool()` / `.Set()` / `.Schema()`。

### 8.2 Schema 注入 — 插件向配置系统注入新的配置项

插件可以在 `Onload` 或 `init()` 阶段通过 `Config.Get().Schema()` 动态添加配置字段：

```go
// plg_security_svg/index.go:16
disable_svg = func() bool {
    return Config.Get("features.protection.disable_svg").Schema(func(f *FormElement) *FormElement {
        if f == nil { f = &FormElement{} }
        f.Default = true
        f.Name = "disable_svg"
        f.Type = "boolean"
        f.Description = "Disable the display of SVG documents"
        return f
    }).Bool()
}
```

如果配置树中不存在该路径，`Schema()` 会自动创建。首次设置 `Default` 时会触发 `Config.Save()` 将新字段持久化。

### 8.3 连接参数注入 — `IBackend.Init(params, app)`

Backend 插件的连接参数通过 `Init` 方法注入：

```go
// plg_backend_s3/index.go:42
func (this S3Backend) Init(params map[string]string, app *App) (IBackend, error) {
    region := params["region"]
    // params 包含前端表单提交的所有字段
}
```

`params` 的来源链路：
1. 前端提交登录表单（类型+凭据）
2. `SessionAuthenticate`（`ctrl/session.go:52`）接收为 `ctx.Body`
3. `model.NewBackend(ctx, session)` 传入
4. `Backend.Get(conn["type"]).Init(conn, ctx)` 将 `conn` 作为 `params` 注入

### 8.4 认证中间件参数注入

认证插件的参数通过配置注入，路径为 `middleware.identity_provider.params`：

```go
// ctrl/session.go:269
idpParams := TmplParams(map[string]string{})
json.Unmarshal(
    []byte(Config.Get("middleware.identity_provider.params").String()),
    &idpParams,
)
```

`idpParams` 支持模板变量插值（`TmplExec`），管理员可以在配置中写 `{{ .client_id }}` 等模板表达式。

### 8.5 外部插件配置注入

WASM 插件不直接读取 `Config`，而是通过宿主导出函数间接获取：
- **Middleware 插件**：通过 `req_header_get` 读取请求头中的配置信息
- **Workflow Action 插件**：通过 `workflow_params_get` 获取执行参数

---

## 九、错误回收

### 9.1 启动阶段 — check() 立即退出

```go
// cmd/main.go:52
func check(err error, msg string) {
    if err == nil { return }
    Log.Error(msg, err.Error())
    os.Exit(1)
}
```

`InitLogger`、`InitConfig`、`Discovery`、`InitPluginList`、`workflow.Init` 任一失败，进程直接退出。

### 9.2 外部插件发现 — 单插件失败不阻断

```go
// discovery.go:28-31
for _, entry := range entries {
    name, impl, err := initModule(fname)
    if err != nil {
        Log.Error("could not initialise module name=%s err=%s", entry.Name(), err.Error())
        continue                    // 跳过失败的插件，继续加载下一个
    }
}
```

单个 zip 解析失败只记录日志，不影响其他插件。

### 9.3 WASM 调用错误回收

**Middleware 插件**（`adapter/middleware.go:26-36`）：

```go
err := rt.Call(r.Context(), "middleware", middlewareKey{}, &middlewareState{r: r, w: w, next: &callNext})
if errors.Is(err, runtime.ErrNoExport) || callNext {
    next(app, w, r)              // 导出函数不存在或插件调用了 next → 继续链
    return
} else if err != nil {
    log.Printf("middleware plugin call error: %v", err)  // 仅日志，不 panic
}
```

当 WASM 中间件出错时，**请求被静默中断**（不继续执行 next）。这是当前设计的一个取舍——插件失败不会泄露错误给用户，但也意味着后续中间件和业务逻辑不会执行。

**Workflow Action 插件**（`adapter/workflow.go:71-79`）：

```go
func (this *workflowActionState) Execute(params, input map[string]string) (map[string]string, error) {
    this.err = nil
    if err := this.rt.Call(context.Background(), "execute", workflowActionKey{}, this); err != nil {
        this.err = err            // 运行时错误优先
    }
    return this.output, this.err  // 将错误返回给工作流引擎
}
```

### 9.4 WASM 运行时错误类型

```go
// adapter/runtime/error.go
var ErrNoExport = errors.New("plugin: export not found")
```

当 WASM 模块缺少指定导出函数时，`Call()` 返回 `ErrNoExport`，中间件适配器对此做特殊处理（视为"放行"）。

### 9.5 Backend 错误回收

Backend 的错误通过 `IBackend` 接口方法返回 `error`，由上层统一转换为 `AppError`：

```go
// common/error.go
type AppError struct {
    message string
    status  int
}

func HTTPError(err error) AppError {
    // 将已知错误映射为标准 HTTP 状态码
}
```

响应通过 `SendErrorResult()` 写回客户端（`common/response.go:85`）：

```go
func SendErrorResult(res http.ResponseWriter, err error) {
    obj, ok := err.(interface{ Status() int })
    if ok { res.WriteHeader(obj.Status()) }
    else { res.WriteHeader(http.StatusInternalServerError) }
    encoder.Encode(APIErrorMessage{"error", m})
}
```

### 9.6 安全错误回收

`plg_security_svg`（`plugin/plg_security_svg/index.go`）展示了安全插件的错误回收模式：

```go
Hooks.Register.ProcessFileContentBeforeSend(func(reader io.ReadCloser, ctx *App, ...) (io.ReadCloser, bool, error) {
    if GetMimeType(req.URL.Query().Get("path")) != "image/svg+xml" {
        return reader, false, nil           // 非目标类型 → 原样放行
    }
    if disable_svg() == false {
        return reader, false, nil           // 功能关闭 → 原样放行
    }
    // 安全处理：设置 CSP 头 + 过滤 XML entity
    (*res).Header().Set("Content-Security-Policy", "script-src 'none'; default-src 'none'; img-src 'self'")
    txt, _ := io.ReadAll(reader)
    if regexp.MustCompile("(?is)entity").Match(txt) {
        txt = []byte("")                    // 检测到 XML 炸弹 → 清空内容
    }
    return NewReadCloserFromBytes(txt), true, nil
})
```

返回值 `(io.ReadCloser, bool, error)` 的含义：
- `bool = true`：插件已处理该文件，后续 `ProcessFileContentBeforeSend` 钩子不再执行
- `bool = false`：继续执行下一个钩子

---

## 十、卸载与热重启

### 10.1 优雅退出

进程收到信号后执行退出清理：

```go
// cmd/main.go:60-68
func withSignal() context.Context {
    ctx, cancel := context.WithCancel(context.Background())
    go func() {
        quit := make(chan os.Signal, 1)
        signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
        <-quit
        cancel()                     // 取消 context → Starter 中的 HTTP 服务器 Shutdown
    }()
    return ctx
}
```

随后执行所有 `OnQuit` 回调：

```go
// cmd/main.go:47
for _, fn := range Hooks.Get.OnQuit() {
    fn()
}
```

### 10.2 WASM 运行时关闭

`Runtime.Close()` 释放 wazero 运行时资源：

```go
// adapter/runtime/runtime.go:57
func (r *Runtime) Close() {
    r.wrt.Close(r.ctx)
}
```

**注意**：当前代码中 `MiddlewareExtension` 和 `WorkflowActionExtension` 创建的 `Runtime` 实例**没有注册到 OnQuit**，意味着 WASM 运行时的清理依赖 Go GC 和进程退出。这不是内存泄漏（进程即将退出），但如果未来需要热重启则需补上。

### 10.3 热重启限制

**当前不支持真正的热重启**，原因：

1. **内置插件通过 `init()` + `_ import` 加载** — 这是 Go 编译期决定的，运行时无法卸载或重新加载。
2. **全局变量注册** — `Hooks.Register.*` 写入的是包级 `var`，没有提供 `Unregister` 或 `Reset` 方法。
3. **Starter 是单例** — `starter_process` 只有一个值，无法热替换 HTTP 服务器。
4. **外部插件无卸载机制** — `extension.Discovery()` 只有注册逻辑，没有对应的 `Undiscovery()` 或 `Reload()`。

**半热重启路径**（仅限配置热更新）：

```
管理员修改配置 → POST /admin/api/config
    → Config.Load() (config.go:186)
        → Hooks.Get.OnConfig() 回调触发
        → 所有注册了 OnConfig 的插件重新读取配置
```

例如 `plg_security_svg` 在 `Onload` 中闭包捕获了 `disable_svg` 函数，该函数每次调用都会通过 `Config.Get()` 读取最新配置值，因此配置变更可以立即生效。

---

## 十一、前端插件加载链路

```
浏览器加载页面
    → /api/config  → 获取全局配置（含 connections、auth middleware 列表等）
    → /api/plugin  → PluginExportHandler → 获取 xdg-open 插件映射
    → model/plugin.js:init()  → 缓存插件映射到 plugins 对象
    → 用户点击文件
    → plugin.js:get(mime)  → 查找对应插件
    → plugin.js:load(mime) → 动态 import() 插件 JS
        → /assets/{BUILD_REF}/plugin/{name}.zip/{entrypoint}
        → PluginStaticHandler 从 zip 中读取文件返回
    → 插件 JS 的 default export 初始化预览/编辑器
```

---

## 十二、关键代码位置索引

| 关注点 | 文件 | 行号 |
|--------|------|------|
| 插件钩子注册表 | `server/common/plugin.go` | 全文 |
| 钩子类型定义 | `server/common/types.go` | 13-120 |
| 启动编排 | `cmd/main.go` | 25-58 |
| 外部插件发现 | `server/pkg/extension/discovery.go` | 15-80 |
| WASM 运行时 | `server/pkg/extension/adapter/runtime/runtime.go` | 全文 |
| WASM 中间件适配 | `server/pkg/extension/adapter/middleware.go` | 全文 |
| WASM Workflow 适配 | `server/pkg/extension/adapter/workflow.go` | 全文 |
| 中间件链构建 | `server/middleware/index.go` | 21-37 |
| 插件中间件注入 | `server/middleware/index.go` | 72-76 |
| 核心路由 | `server/routes.go` | 19-176 |
| 插件路由 | `server/routes.go` | 158-176 |
| 配置系统 | `server/common/config.go` | 全文 |
| 配置变更钩子触发 | `server/common/config.go` | 215 |
| Backend Driver | `server/common/backend.go` | 全文 |
| Backend 创建与白名单 | `server/model/files.go` | 9-50 |
| 认证中间件调度 | `server/ctrl/session.go` | 220-487 |
| 错误类型体系 | `server/common/error.go` | 全文 |
| 响应封装 | `server/common/response.go` | 全文 |
| 前端插件模型 | `public/assets/model/plugin.js` | 全文 |
| 内置插件清单 | `server/plugin/index.go` | 全文 |
| 常量/路径 | `server/common/constants.go` | 全文 |
