# 审计日志 & 配额处理 流程梳理

## 1. 整体架构概览

Filestash 采用 **插件化 + Hook 注册机制** 实现审计日志与配额/容量限制功能。核心的衔接点是通过 `IAuthorisation` 中间件链在 **Controller 层与 Backend 层之间**注入横切逻辑。

```
HTTP 请求 → 中间件链 → Controller (ctrl/)
                            ↓
                AuthorisationMiddleware 链 (审计触发点)
                            ↓
                     Backend 插件 (plg_backend_*)
                  (配额/容量限制的实际执行)
                            ↓
                 Workflow Trigger 事件总线
                            ↓
              IAuditPlugin 查询接口 (audit 查询)
```

---

## 2. 审计日志（操作记录）完整流程

### 2.1 接口定义

在 `server/common/types.go:65-71` 定义了审计插件接口：

```go
type IAuditPlugin interface {
    Query(ctx *App, searchParams map[string]string) (AuditQueryResult, error)
}
type AuditQueryResult struct {
    Form       *Form  `json:"form"`    // 查询表单定义
    RenderHTML string `json:"render"`  // 渲染后的 HTML
}
```

### 2.2 插件注册机制

通过 `server/common/plugin.go:185-193` 的 Hook 注册：

```go
var audit IAuditPlugin

func (this Register) AuditEngine(a IAuditPlugin) {
    audit = a
}
func (this Get) AuditEngine() IAuditPlugin {
    return audit
}
```

**默认实现**：`server/model/audit.go:7-9` 注册了 `SimpleAudit`，它只是一个占位符，提示需要安装真正的审计插件：

```go
func init() {
    Hooks.Register.AuditEngine(SimpleAudit{})
}
```

### 2.3 操作记录的触发链路

操作记录不是直接写入数据库，而是通过 **Workflow Trigger 事件总线** 分发。核心代码在 `server/pkg/workflow/trigger/fileaction.go`。

#### Step 1: 注册 WorkflowTrigger 并注入 AuthorisationMiddleware

```go
// fileaction.go:15-17
func init() {
    Hooks.Register.WorkflowTrigger(&FileEventTrigger{})
}

// fileaction.go:86-88 - 关键：注册授权中间件来捕获所有操作
func (this *FileEventTrigger) Init() (chan ITriggerEvent, error) {
    Hooks.Register.AuthorisationMiddleware(hookAuthorisation{})
    return fileaction_event, nil
}
```

#### Step 2: hookAuthorisation 拦截所有文件操作

`hookAuthorisation` 实现了 `IAuthorisation` 接口（8 种操作），每种操作都会调用 `processFileAction`：

| 方法 | 事件类型 | 说明 |
|------|----------|------|
| `Ls(ctx, path)` | `ls` | 列目录 |
| `Cat(ctx, path)` | `cat` | 读文件/下载 |
| `Mkdir(ctx, path)` | `mkdir` | 创建目录 |
| `Rm(ctx, path)` | `rm` | 删除 |
| `Mv(ctx, from, to)` | `mv` | 移动/重命名 |
| `Save(ctx, path)` | `stat` | 保存（注意：event 名是 stat） |
| `Touch(ctx, path)` | `touch` | 创建空文件 |
| `Stat(ctx, path)` | `save` | 状态检查（注意：event 名是 save） |

#### Step 3: processFileAction 发布事件

```go
// fileaction.go:91-98
func processFileAction(ctx *App, params map[string]string) {
    // 关键开关：AUDIT 上下文标志
    if ctx.Context.Value("AUDIT") == false {
        return
    }
    if err := TriggerEvents(fileaction_event, fileaction_name, 
        fileactionCallback(params)); err != nil {
        Log.Error("[workflow] trigger=event step=triggerEvents err=%s", err.Error())
    }
}
```

#### Step 4: AUDIT 开关的特殊处理

在 `server/ctrl/files.go:99-126` 的 `FileLs` 中，为了探测权限会先临时关闭 AUDIT 标志：

```go
for _, auth := range Hooks.Get.AuthorisationMiddleware() {
    if err = auth.Ls(ctx, path); err != nil { ... }
    
    // 关闭 AUDIT 标志，避免权限探测产生审计记录
    ctx.Context = context.WithValue(ctx.Context, "AUDIT", false)
    
    if err = auth.Mkdir(ctx, path); err != nil { perms.CanCreateDirectory = NewBool(false) }
    if err = auth.Touch(ctx, path); err != nil { perms.CanCreateFile = NewBool(false) }
    if err = auth.Mv(ctx, path, path); err != nil { perms.CanRename = NewBool(false); perms.CanMove = NewBool(false) }
    if err = auth.Save(ctx, path); err != nil { perms.CanUpload = NewBool(false) }
    if err = auth.Rm(ctx, path); err != nil { perms.CanDelete = NewBool(false) }
    if err = auth.Cat(ctx, path); err != nil { perms.CanSee = NewBool(false) }
    
    // 恢复 AUDIT 标志
    ctx.Context = context.WithValue(ctx.Context, "AUDIT", nil)
}
```

### 2.4 Workflow 事件分发

`server/pkg/workflow/trigger/index.go:24-47` 的 `TriggerEvents`：

1. 查询所有配置了 `event` trigger 的 workflow
2. 通过 callback 匹配 event 类型和 path 模式
3. 将匹配的事件发布到 `fileaction_event` channel

`server/pkg/workflow/index.go:21-61` 的 `Init()` 中：
- 启动 goroutine 监听 trigger channel
- 创建 Job 并放入工作队列
- worker 池异步执行关联的 Action（如发邮件、调用 API 等）

### 2.5 审计日志的查询接口

**路由**：`server/routes.go:48` → `GET /admin/api/audit`

```go
admin.HandleFunc("/audit", NewMiddlewareChain(FetchAuditHandler, middlewares)).Methods("GET")
```

**Handler**：`server/ctrl/admin.go:138-158`

```go
func FetchAuditHandler(ctx *App, res http.ResponseWriter, req *http.Request) {
    plg := Hooks.Get.AuditEngine()
    if plg == nil {
        SendErrorResult(res, ErrNotImplemented)
        return
    }
    // 将查询参数传给审计插件
    searchParams := map[string]string{}
    for key, element := range req.URL.Query() {
        if len(element) > 0 {
            searchParams[key] = element[0]
        }
    }
    result, err := plg.Query(ctx, searchParams)
    // ... 返回结果
}
```

**前端**：`public/assets/pages/adminpage/model_audit.js` → `ctrl_activity_audit.js`

- 查询 `GET /admin/api/audit?action=xxx&path=xxx&...`
- 渲染插件返回的 Form 和 HTML

### 2.6 审计 Form 支持的查询字段

`server/model/audit.go:11-56` 定义了默认的搜索表单字段：

- `date from` / `date to` - 时间范围
- `action` - 操作类型（rename/list/download/create_folder/remove/move/save_file/create_file）
- `path` - 路径
- `backend` - 后端类型
- `session` - 会话 ID
- `share` - 共享链接 ID
- `user` - 用户
- `target` - 目标路径

---

## 3. 配额 / 容量限制 完整流程

### 3.1 配额限制的执行层级

Filestash 的配额限制不是在核心框架层统一执行，而是 **分层处理**：

| 层级 | 位置 | 限制类型 | 说明 |
|------|------|----------|------|
| **后端存储层** | `plg_backend_*/index.go` | 空间配额/磁盘满 | 由具体后端返回错误 |
| **协议层** | NFS4/SFTP 协议 | 配额属性 | 协议内置配额字段 |
| **应用层** | `ctrl/files.go` | 上传并发/分块 | 上传池大小、分块大小 |
| **访问层** | `model/files.go` | 连接白名单 | 只允许连接配置中的后端 |

### 3.2 后端存储层：各插件的配额处理

#### SFTP 后端（示例）

`server/plugin/plg_backend_sftp/index.go:395-397` 错误映射：

```go
case 14:
    return NewError("No space left", 400)   // 磁盘空间不足
case 15:
    return NewError("Quota exceeded", 400)  // 配额超限
```

#### NFS4 后端（协议级配额）

NFS4 协议内置配额属性（`plg_backend_nfs4/repo/internal/nfs4.go:743-747`）：

```go
const FATTR4_QUOTA_AVAIL_HARD = 38  // 硬配额上限
const FATTR4_QUOTA_AVAIL_SOFT = 39  // 软配额上限
const FATTR4_QUOTA_USED         = 40  // 已用空间
```

NFS4 错误码：`NFS4ERR_DQUOT = 69` 表示硬配额达到。

#### Local 后端

`plg_backend_local/index.go` 直接调用系统 API，配额由操作系统文件系统强制执行，错误会透传给上层。

### 3.3 应用层：上传相关限制

配置定义在 `server/common/config.go:79-80`：

```go
FormElement{Name: "upload_pool_size", Type: "number", Default: 15, 
    Description: "Maximum number of files upload in parallel. Default: 15"},
FormElement{Name: "upload_chunk_size", Type: "number", Default: 0, 
    Description: "Size of Chunks for Uploads in MB."},
```

配置导出到前端 `config.go:323-324`：

```go
UploadPoolSize:  this.Get("general.upload_pool_size").Int(),
UploadChunkSize: this.Get("general.upload_chunk_size").Int(),
```

前端使用：`public/assets/pages/filespage/ctrl_upload.js:364`

```javascript
const chunkSize = getConfig("upload_chunk_size", 0) * 1024 * 1024;
```

#### TUS 分块上传流程

`server/ctrl/files.go:542-669` 实现了 TUS 协议的断点续传：

1. `POST` - 创建上传会话，指定 `Upload-Length`
2. `PATCH` - 上传数据块，校验 `Upload-Offset`
3. `HEAD` - 查询当前上传进度
4. 完成后调用 `ctx.Backend.Save(path, reader)`

```go
// files.go:568-591 - 创建上传会话
if proto == "tus" && req.Method == http.MethodPost {
    size, _ := strconv.ParseUint(req.Header.Get("Upload-Length"), 10, 0)
    b, _ := ctx.Backend.Init(ctx.Session, ctx)
    uploader := createChunkedUploader(b.Save, path, size)
    chunkedUploadCache.Set(cacheKey, uploader)
}
```

### 3.4 访问层：后端连接白名单

`server/model/files.go:9-50` 的 `NewBackend` 中强制检查：

```go
isAllowed := func() bool {
    possibilities := make([]map[string]interface{}, 0)
    for i := 0; i < len(Config.Conn); i++ {
        d := Config.Conn[i]
        // 匹配 type + hostname/path/url
        if d["type"] != conn["type"] { continue }
        if val, ok := d["hostname"]; ok && val != conn["hostname"] { continue }
        if val, ok := d["path"]; ok {
            configPath := val.(string)
            if !strings.HasPrefix(conn["path"], configPath) { continue }
        }
        if val, ok := d["url"]; ok && val != conn["url"] { continue }
        possibilities = append(possibilities, Config.Conn[i])
    }
    return len(possibilities) > 0
}
if !isAllowed() {
    return Backend.Get(BACKEND_NIL), ErrNotAllowed
}
```

这防止用户绕过配置连接到任意后端。

---

## 4. 后端插件与审计/配额的衔接机制

### 4.1 IBackend 接口定义

所有后端必须实现 `server/common/types.go:13-24`：

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

### 4.2 Controller → AuthorisationMiddleware → Backend 的调用链

以 `FileSave` 上传为例 (`server/ctrl/files.go:479-539`)：

```
1. 权限检查：model.CanEdit / CanUpload
2. 路径构造：PathBuilder(ctx, path)
3. ★ AuthorisationMiddleware 链：
   for _, auth := range Hooks.Get.AuthorisationMiddleware() {
       auth.Save(ctx, path)          // ← 审计触发点
   }
4. 实际后端调用：ctx.Backend.Save(path, req.Body)  // ← 配额执行点
```

**关键：审计在调用 Backend 之前触发（通过 AuthorisationMiddleware），配额由 Backend 在实际执行时返回错误。**

### 4.3 插件注册与加载

`server/plugin/index.go` 通过空白导入加载所有插件：

```go
import (
    _ "github.com/mickael-kerjean/filestash/server/plugin/plg_backend_local"
    _ "github.com/mickael-kerjean/filestash/server/plugin/plg_backend_s3"
    _ "github.com/mickael-kerjean/filestash/server/plugin/plg_backend_sftp"
    // ... 20+ 种后端
)
```

每个后端在 `init()` 中注册到 `Backend` Driver：

```go
// plg_backend_local/index.go:11-13
func init() {
    Backend.Register("local", &Local{os.Getenv("LOCAL_BACKEND_SECRET")})
}
```

### 4.4 完整请求时序

```
用户发起请求 (eg: POST /api/files/save)
    │
    ▼
┌────────────────────────────┐
│  中间件链 (middleware/)     │
│  - SessionStart            │  提取 Session、Share、Backend
│  - LoggedInOnly            │  权限检查
│  - PluginInjector          │  注入插件自定义中间件
└────────────┬───────────────┘
             │
             ▼
┌────────────────────────────┐
│  Controller (ctrl/files.go)│
│  - CanEdit / CanUpload     │  Share 级权限
│  - PathBuilder             │  路径 chroot 检查
└────────────┬───────────────┘
             │
             ▼
┌──────────────────────────────────────┐
│  AuthorisationMiddleware 链          │
│  (按注册顺序调用)                     │
│  1. plg_authorisation_example        │  自定义权限规则
│  2. hookAuthorisation (fileaction)   │  ★ 审计触发点
│     └─ processFileAction()           │
│        └─ TriggerEvents() → workflow│
└────────────┬─────────────────────────┘
             │
             ▼
┌────────────────────────────┐
│  Backend 插件               │  ★ 配额/容量限制执行点
│  (plg_backend_*)            │
│  - Save(path, reader)       │  实际写入存储
│  - 返回 ErrQuotaExceeded    │  或底层错误
└────────────┬───────────────┘
             │
             ▼
┌────────────────────────────┐
│  HTTP 响应                  │
│  - 200 OK / 4xx 错误        │
│  - logger() 异步记录访问日志 │  (server/middleware/telemetry.go)
└────────────────────────────┘
```

### 4.5 扩展自定义审计插件的方式

1. 实现 `IAuditPlugin` 接口：
   ```go
   type MyAudit struct{}
   func (MyAudit) Query(ctx *App, params map[string]string) (AuditQueryResult, error) {
       // 查询数据库并返回结果
   }
   ```

2. 在 `init()` 中注册：
   ```go
   func init() {
       Hooks.Register.AuditEngine(MyAudit{})
   }
   ```

3. 同时注册 `AuthorisationMiddleware` 来捕获操作并写入数据库：
   ```go
   type AuditWriter struct{}
   func (AuditWriter) Ls(ctx *App, path string) error {
       // 写入审计表: INSERT INTO audit (event, path, user, time) VALUES ('ls', ...)
       return nil
   }
   // ... 实现其他 7 个方法
   func init() {
       Hooks.Register.AuthorisationMiddleware(AuditWriter{})
   }
   ```

### 4.6 扩展自定义配额限制的方式

**方式 1：自定义 AuthorisationMiddleware**（在操作前检查）：

```go
type QuotaChecker struct{}

func (QuotaChecker) Save(ctx *App, path string) error {
    userID := GenerateID(ctx.Session)
    used := GetUserUsedStorage(userID)
    limit := GetUserQuota(userID)
    if used >= limit {
        return NewError("Quota exceeded", 400)
    }
    return nil
}
```

**方式 2：包装 Backend**（在实际写入时精确计量）：

```go
// 在 Save 中包装 reader，统计实际字节数
func (b *QuotaBackend) Save(path string, r io.Reader) error {
    counter := &CountingReader{Reader: r}
    err := b.real.Save(path, counter)
    if err == nil {
        AddUsedStorage(b.userID, counter.BytesRead)
    }
    return err
}
```

---

## 5. 关键文件索引

| 文件路径 | 作用 |
|----------|------|
| `server/common/types.go` | `IAuditPlugin`、`IAuthorisation`、`IBackend` 接口定义 |
| `server/common/plugin.go` | Hook 注册/获取机制（`Hooks.Register.*` / `Hooks.Get.*`） |
| `server/common/backend.go` | Backend Driver 注册与获取 |
| `server/common/config.go` | 配置定义（上传并发、分块等） |
| `server/model/audit.go` | 默认 `SimpleAudit` 实现、AuditForm 定义 |
| `server/model/files.go` | `NewBackend` 连接白名单检查 |
| `server/model/permissions.go` | `CanRead/CanEdit/CanUpload/CanShare` 权限函数 |
| `server/ctrl/files.go` | 文件操作 Controller，AuthorisationMiddleware 调用点 |
| `server/ctrl/admin.go` | `FetchAuditHandler` 审计查询 Handler |
| `server/routes.go` | 路由注册，`/admin/api/audit` 端点 |
| `server/middleware/index.go` | `NewMiddlewareChain` 中间件组装，`logger()` 调用点 |
| `server/middleware/session.go` | `SessionStart`，`_extractBackend` 初始化 |
| `server/middleware/telemetry.go` | HTTP 访问日志记录（LogEntry） |
| `server/pkg/workflow/index.go` | Workflow 初始化、worker 池 |
| `server/pkg/workflow/trigger/fileaction.go` | ★ `hookAuthorisation` 审计触发核心 |
| `server/pkg/workflow/trigger/index.go` | `TriggerEvents` 事件分发 |
| `server/plugin/index.go` | 所有后端/认证插件的导入入口 |
| `server/plugin/plg_authorisation_example/index.go` | 授权中间件示例 |
| `server/plugin/plg_backend_local/index.go` | Local 后端实现（配额靠 OS） |
| `server/plugin/plg_backend_sftp/index.go` | SFTP 后端（配额错误码映射） |
| `public/assets/pages/adminpage/model_audit.js` | 前端审计查询模型 |
| `public/assets/pages/adminpage/ctrl_activity_audit.js` | 前端审计页面控制器 |
