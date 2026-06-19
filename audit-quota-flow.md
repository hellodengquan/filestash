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

## 目录

- [2. 审计日志（操作记录）完整流程](#2-审计日志操作记录完整流程)
  - [2.2.1 衔接点 10：IAuditPlugin 与 AuthorisationMiddleware 的实现关系与注册流程](#221-衔接点-10iauditplugin-与-authorisationmiddleware-的实现关系与注册流程)
  - [2.4.1 衔接点 7：审计 log 记录持久化与查询索引路径](#241-衔接点-7审计-log-记录持久化与查询索引路径)
  - [2.4.2 衔接点 11：SimpleAudit 默认实现的全部行为（placeholder 之外）](#242-衔接点-11simpleaudit-默认实现的全部行为placeholder-之外)
  - [2.5 衔接点 1：Save 失败时审计日志如何落地、操作记录与失败结果对齐](#25-衔接点-1save-失败时审计日志如何落地操作记录与失败结果对齐)
  - [2.6.1 衔接点 8：审计检索 API 支持的过滤条件、按时间分页实现](#261-衔接点-8审计检索-api-支持的过滤条件按时间分页实现)
  - [2.6.2 衔接点 12：批量导出、异步聚合与归档路径](#262-衔接点-12批量导出异步聚合与归档路径)
- [3. 配额 / 容量限制 完整流程](#3-配额--容量限制-完整流程)
  - [3.2.1 衔接点 9：配额超限通知如何对接邮件 / Webhook / 消息总线](#321-衔接点-9配额超限通知如何对接邮件--webhook--消息总线)
  - [3.3.1 衔接点 3：大文件分块上传 mid-stream 配额检查与最终入库前二次校验](#331-衔接点-3大文件分块上传-mid-stream-配额检查与最终入库前二次校验)
  - [3.5 衔接点 2：同一用户多后端连接时容量限制的跨后端聚合路径](#35-衔接点-2同一用户多后端连接时容量限制的跨后端聚合路径)
  - [3.6 衔接点 4：TUS 断点续传已上传字节与配额对账、续传中途配额变化响应](#36-衔接点-4tus-断点续传已上传字节与配额对账续传中途配额变化响应)
  - [3.7 衔接点 5：插件热替换/reload 时正在进行的上传处理与资源回收](#37-衔接点-5插件热替换reload-时正在进行的上传处理与资源回收)
  - [3.8 衔接点 6：同一用户同时对多个后端写入的资源与计数协调](#38-衔接点-6同一用户同时对多个后端写入的资源与计数协调)
- [4. 后端插件与审计/配额的衔接机制](#4-后端插件与审计配额的衔接机制)
- [5. 关键文件索引](#5-关键文件索引)

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

### 2.2.1 衔接点 10：IAuditPlugin 与 AuthorisationMiddleware 的实现关系与注册流程

#### 2.2.1.1 两个接口的职责分离

审计体系由**两个完全独立的接口**协作完成，分别负责"写"和"读"：

| 接口 | 职责 | 方法数 | 注册 Hook | 调用时机 |
|------|------|--------|-----------|----------|
| `IAuthorisation` | **操作拦截（写入口）** | 8 个（Ls/Cat/Stat/Mkdir/Rm/Mv/Save/Touch） | `Hooks.Register.AuthorisationMiddleware()` | 每个文件操作**执行前** |
| `IAuditPlugin` | **审计查询（读出口）** | 1 个（Query） | `Hooks.Register.AuditEngine()` | 管理后台查询审计记录时 |

两个接口在类型系统上**没有任何继承或包含关系**，完全独立。

#### 2.2.1.2 为什么是两个接口而不是一个

设计意图是**关注点分离**：

1. `IAuthorisation` 是通用的权限/授权中间件，**不是专为审计设计的**：
   - 可以用来做权限控制（返回 error 阻止操作）
   - 可以用来做审计记录（返回 nil 但顺便记一笔）
   - 可以用来做事件触发（比如 workflow 的 fileevent trigger）

2. `IAuditPlugin` 是专属的查询接口：
   - 返回 `Form` 定义搜索条件
   - 返回 `RenderHTML` 渲染结果表格
   - 只在管理后台审计页面使用

一个完整的审计插件需要**同时注册到两个 Hook 上**：

```go
type FullAudit struct{}

// 实现 IAuthorisation → 写
func (this FullAudit) Ls(ctx *App, path string) error    { writeAudit("ls", path, ctx); return nil }
func (this FullAudit) Cat(ctx *App, path string) error   { writeAudit("cat", path, ctx); return nil }
func (this FullAudit) Save(ctx *App, path string) error  { writeAudit("save", path, ctx); return nil }
// ... 其余 5 个方法

// 实现 IAuditPlugin → 读
func (this FullAudit) Query(ctx *App, p map[string]string) (AuditQueryResult, error) {
    return AuditQueryResult{Form: &AuditForm, RenderHTML: renderTable(p)}, nil
}

func init() {
    // 同时注册两个 Hook
    Hooks.Register.AuthorisationMiddleware(FullAudit{})
    Hooks.Register.AuditEngine(FullAudit{})
}
```

#### 2.2.1.3 注册时序与流程

注册分两条线，在不同时间点完成：

**线 A：IAuthorisation（通过 Workflow Trigger 间接注册）**

```
进程启动 → init() 全部执行
    │
    ├─ model/audit.go init()
    │   └─ Hooks.Register.AuditEngine(SimpleAudit{})
    │         ↑ 只有 IAuditPlugin 在这里注册
    │
    ├─ workflow/trigger/fileaction.go init()
    │   └─ Hooks.Register.WorkflowTrigger(&FileEventTrigger{})
    │
    └─ 所有 plg_* 插件 init()
        └─ 各自注册自己的东西
            ↓
Onload 阶段 → 启动 HTTP server 之前
    │
    └─ workflow.Init()
        ├─ 遍历所有 WorkflowTrigger
        └─ 每个 trigger.Init()
            └─ FileEventTrigger.Init()
                └─ Hooks.Register.AuthorisationMiddleware(hookAuthorisation{})
                      ↑ 审计写入链的核心中间件在这里才注册
```

**注意**：`hookAuthorisation` 不是由审计插件注册的，而是由 `FileEventTrigger`（workflow 的 trigger）注册的。它的作用是把文件操作转化为 workflow 事件，不是直接写审计日志。

**线 B：IAuditPlugin（直接注册）**

```
进程启动 → model/audit.go init()
    └─ Hooks.Register.AuditEngine(SimpleAudit{})
       ↑ 默认占位实现

→ 如果安装了自定义审计插件 → 插件 init()
    └─ Hooks.Register.AuditEngine(MyAudit{})
       ↑ 覆盖默认的 SimpleAudit（因为 audit 是单例变量，后注册覆盖先注册）
```

**关键细节**：`audit` 是**单例**（`var audit IAuditPlugin`），后注册的会覆盖先注册的。而 `AuthorisationMiddleware` 是**切片**（`var authorisationMiddleware []IAuthorisation`），注册的都会保留，按顺序调用。

#### 2.2.1.4 调用链上的位置

```
HTTP 请求 → Controller (FileSave / FileLs / ...)
                  │
                  ▼
         for _, auth := range Hooks.Get.AuthorisationMiddleware() {
             auth.Save(ctx, path)  ← 写入端：IAuthorisation 链
             if err != nil { return 403 }
         }
                  │
                  ▼
         ctx.Backend.Save(path, reader)  ← 实际操作
                  │
                  ▼
管理后台 GET /admin/api/audit
    │
    └─ plg := Hooks.Get.AuditEngine()  ← 读取端：IAuditPlugin 单例
       result := plg.Query(ctx, searchParams)
```

#### 2.2.1.5 hookAuthorisation 的特殊地位

`hookAuthorisation`（`server/pkg/workflow/trigger/fileaction.go:19-59`）是**默认就有的** AuthorisationMiddleware，它不是审计插件，但它是审计记录的**间接源头**：

```go
type hookAuthorisation struct{}

func (this hookAuthorisation) Save(ctx *App, path string) error {
    processFileAction(ctx, map[string]string{"event": "stat", "path": path})
    return nil  // 永远返回 nil，不阻止操作
}
```

它的作用是**把每个文件操作发布为 workflow 事件**，不做持久化。如果用户配置了"文件事件触发 + 自定义 Action"的 workflow，就可以通过 Action 间接实现审计写入（比如 `run/api` 调外部服务写审计库）。

但这种方式有几个局限：
- 是**异步**的（workflow job 入队，worker 异步执行）
- 每个操作都要走一遍 workflow 匹配逻辑，有性能开销
- 无法获取操作的最终结果（成功/失败）
- 需要用户手动配置 workflow，不是开箱即用

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

### 2.4.1 衔接点 7：审计 log 记录持久化与查询索引路径

#### 2.4.1.1 默认实现：无持久化

核心框架**不提供审计日志的默认持久化存储**。`SimpleAudit`（`server/model/audit.go:58-75`）只是一个**占位符**：

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

打开管理后台的 Activity / Audit 页面，会看到红色提示框："You need to install an audit plugin to use this"。

#### 2.4.1.2 已有的持久化存储体系（可复用但默认不用于审计）

Filestash 核心自带一个 **SQLite 数据库**，在 `Onload` 中初始化（`server/model/index.go:12-38`）：

```go
var DB *sql.DB

func init() {
    Hooks.Register.Onload(func() {
        if DB, err = sql.Open("sqlite3", GetAbsolutePath(DB_PATH)+"/share.sql?_fk=true"); err != nil { ... }
    })
}
```

但 `share.sql` 中**只有 3 张表**，都不用于审计：

| 表名 | 用途 | 索引 |
|------|------|------|
| `Location(backend, path)` | 已共享的路径 | PRIMARY KEY(backend, path) |
| `Share(id, related_backend, related_path, params, auth)` | 共享链接 | FOREIGN KEY → Location |
| `Verification(key, code, expire)` | 邮箱验证码 | `idx_verification(code, expire)` |

还有 Workflow 体系的表（在 `server/pkg/workflow/model/db.go` 中初始化）：

| 表名 | 用途 | 索引 |
|------|------|------|
| `workflows(id, name, published, trigger, actions, created_at, updated_at)` | 工作流配置 | 无（按 name 匹配） |
| `jobs(id, related_workflow, status, input, steps, retries, created_at, started_at, finished_at)` | 工作流执行记录 | 无 |

**jobs 表是唯一接近"审计"的持久化记录**，但它：
- 只记录 workflow job 的执行（不是所有文件操作）
- input/steps 存 JSON，**没有结构化索引**
- 没有按时间、用户、路径的专门索引

#### 2.4.1.3 HTTP 访问日志的独立路径（Telemetry）

与审计日志平行的是 HTTP 访问日志链（`server/middleware/telemetry.go`）：

```go
var telemetry = Telemetry{Data: make([]LogEntry, 0)}  // 内存 buffer

type LogEntry struct {
    Host       string  `json:"host"`
    Method     string  `json:"method"`
    RequestURI string  `json:"pathname"`
    Status     int     `json:"status"`
    Duration   float64 `json:"responseTime"`
    Version    string  `json:"version"`
    Backend    string  `json:"backend"`    // session["type"]
    Share      string  `json:"share"`      // share.Id
    Session    string  `json:"session"`    // GenerateID(ctx.Session)
    RequestID  string  `json:"requestID"`  // X-Request-ID
}
```

**Telemetry 的去向**（两个独立分支）：

1. **内存 → 远端上报**：如果 `Config.Get("log.telemetry").Bool() == true`
   - `telemetry.Record(point)` 追加到 `[]LogEntry` slice（有 `sync.Mutex` 保护）
   - `telemetry.Flush()` 定时（或触发时）通过 HTTP POST 到 `https://downloads.filestash.app/event`
   - 发送后清空 slice
   - **这是遥测数据，不是审计日志，且上报到 Filestash 官方服务器**

2. **本地日志文件**：如果 `Config.Get("log.enable").Bool() == true`
   - 直接 `Log.Stdout("HTTP %3d %3s ...")` → 由 logger 写入 `LOG_PATH` 指定的文件
   - 格式是纯文本行，无结构化，无索引

**`/admin/api/logs` 接口**（`server/ctrl/admin.go:106-136`）读取的是这个本地日志文件：

```go
func FetchLogHandler(ctx *App, res http.ResponseWriter, req *http.Request) {
    file, _ := os.OpenFile(logpath, os.O_RDONLY, os.ModePerm)
    maxSize := req.URL.Query().Get("maxSize")
    if maxSize != "" {
        // 从文件尾部向前读 maxSize 字节，遇到换行符停止
    }
    // 返回纯文本，不是 JSON
}
```

这个接口**只支持 `maxSize`（倒序读取尾部 N 字节）**，**不支持按时间、用户、路径过滤**，也没有分页。

#### 2.4.1.4 自定义审计插件需要自建存储

一个完整的审计插件需要：

```go
type DBAuditPlugin struct{}

func init() {
    // 在 Onload 中初始化审计表
    Hooks.Register.Onload(func() {
        DB.Exec(`CREATE TABLE IF NOT EXISTS audit_log (
            id          INTEGER PRIMARY KEY AUTOINCREMENT,
            event       VARCHAR(32)     NOT NULL,
            path        VARCHAR(2048)   NOT NULL,
            target      VARCHAR(2048),
            backend     VARCHAR(16)     NOT NULL,
            session_id  VARCHAR(40)     NOT NULL,
            user_id     VARCHAR(128),
            share_id    VARCHAR(64),
            request_id  VARCHAR(32)     NOT NULL,
            status      VARCHAR(16)     DEFAULT 'PENDING',
            error_msg   VARCHAR(1024),
            created_at  DATETIME        DEFAULT CURRENT_TIMESTAMP,
            finished_at DATETIME
        )`)
        // 建索引
        DB.Exec(`CREATE INDEX idx_audit_event       ON audit_log(event)`)
        DB.Exec(`CREATE INDEX idx_audit_created_at  ON audit_log(created_at)`)
        DB.Exec(`CREATE INDEX idx_audit_user        ON audit_log(user_id, created_at)`)
        DB.Exec(`CREATE INDEX idx_audit_session     ON audit_log(session_id, created_at)`)
        DB.Exec(`CREATE INDEX idx_audit_path        ON audit_log(path)`)
    })

    // 注册为 AuthorisationMiddleware 来写入日志
    Hooks.Register.AuthorisationMiddleware(DBAuditPlugin{})
    // 注册为 AuditEngine 来提供查询
    Hooks.Register.AuditEngine(DBAuditPlugin{})
}

// 写入（操作前 PENDING）
func (this DBAuditPlugin) Save(ctx *App, path string) error {
    DB.Exec(`INSERT INTO audit_log
        (event, path, backend, session_id, user_id, share_id, request_id, status)
        VALUES ('save', ?, ?, ?, ?, ?, ?, 'PENDING')`,
        path, ctx.Session["type"], GenerateID(ctx.Session),
        getUser(ctx.Session), ctx.Share.Id, ctx.Context.Value("X-Request-ID"))
    return nil
}

// 查询（支持过滤与分页）
func (this DBAuditPlugin) Query(ctx *App, p map[string]string) (AuditQueryResult, error) {
    // 根据 p["date from"], p["date to"], p["action"], p["path"], ... 构造 WHERE 子句
    // 用 LIMIT / OFFSET 分页
    // 返回自定义的 RenderHTML 表格
}
```

#### 2.4.1.5 无现成索引路径总结

| 能力 | 核心框架默认 | 可复用体系 | 需自建 |
|------|-------------|-----------|--------|
| 审计日志持久化 | ❌ 无 | `share.sql` SQLite、Workflow `jobs` 表 | ✅ |
| 查询索引 | ❌ 无 | `jobs` 表无专门索引、telemetry 无索引 | ✅ |
| 按时间过滤 | ❌ 无 | 无（`/admin/api/logs` 只支持倒序 maxSize） | ✅ |
| 分页 | ❌ 无 | 无 | ✅ |
| 按用户/路径/后端/操作类型过滤 | ❌ 无 | Workflow `FindWorkflows` 有 trigger 过滤但不是审计表 | ✅ |

### 2.4.2 衔接点 11：SimpleAudit 默认实现的全部行为（placeholder 之外）

`SimpleAudit`（`server/model/audit.go:58-75`）虽然是占位符，但它**不是空的**。除了返回红色提示 HTML 之外，它还做了以下几件事：

#### 2.4.2.1 提供完整的搜索表单定义

`SimpleAudit.Query()` 返回的 `AuditQueryResult` 包含 `Form: &AuditForm`。`AuditForm`（`audit.go:11-56`）是一个包级变量，定义了**9 个搜索字段**的完整表单结构：

```go
var AuditForm Form = Form{
    Form: []Form{
        Form{
            Title: "search",
            Elmnts: []FormElement{
                {Name: "date from", Type: "datetime"},
                {Name: "date to",   Type: "datetime"},
                {
                    Name: "action", Type: "select",
                    Opts: []string{"", "rename", "list", "download",
                                   "create_folder", "remove", "move",
                                   "save_file", "create_file"},
                },
                {Name: "path",    Type: "text"},
                {Name: "backend", Type: "text"},
                {Name: "session", Type: "text"},
                {Name: "share",   Type: "text"},
                {Name: "user",    Type: "text"},
                {Name: "target",  Type: "text"},
            },
        },
    },
}
```

前端拿到这个 Form 后，会自动渲染成搜索栏。也就是说——**即使没有安装审计插件，管理后台的审计页面也有一个完整的搜索表单**，只是搜索结果区域是红色提示。

#### 2.4.2.2 定义了操作类型的标准枚举

`action` 字段的 `Opts` 定义了 8 种标准操作类型：`rename`、`list`、`download`、`create_folder`、`remove`、`move`、`save_file`、`create_file`。

这是**审计操作类型的"官方标准"**，自定义审计插件应该沿用这些枚举值以保持表单兼容。

**注意**：这里的操作类型枚举（`save_file`/`create_file`）与 `hookAuthorisation` 中的事件名（`stat`/`save`/`touch` 等）**不一致**：

| hookAuthorisation event | AuditForm action 选项 | 对应关系 |
|-------------------------|----------------------|----------|
| `ls` | `list` / `download` | 列表浏览/文件下载 |
| `cat` | `download` | 文件下载 |
| `stat` | `save_file` | 文件上传（Save 操作） |
| `save` | `create_file` | 文件创建？ |
| `mkdir` | `create_folder` | 目录创建 |
| `rm` | `remove` | 删除 |
| `mv` | `move` / `rename` | 移动/重命名 |
| `touch` | （无对应） | 创建空文件 |

**这是一个设计不一致点**：事件名和表单枚举不是一一对应的。`Save` 操作对应的 event 是 `stat`（`fileaction.go:47`），而 `Stat` 操作对应的 event 是 `save`（`fileaction.go:52`）— 看起来是写反了。

#### 2.4.2.3 渲染内联 CSS 样式

`RenderHTML` 中除了提示文字，还包含了一段内联的 `<style>`，用来定义 `#alert-audit-missing` 的红色背景样式，使用 CSS 变量（`--error`、`--super-light`）确保与主题色一致。

#### 2.4.2.4 忽略所有搜索参数

`SimpleAudit.Query()` 方法签名里有 `searchParams map[string]string` 参数，但函数体内**完全没有使用它**——无论你传什么过滤条件，都返回同样的占位 HTML。

```go
func (this SimpleAudit) Query(ctx *App, searchParams map[string]string) (AuditQueryResult, error) {
    // searchParams 完全没被使用
    return AuditQueryResult{...}, nil
}
```

#### 2.4.2.5 作为单例被注册

`SimpleAudit{}` 在 `init()` 中注册为默认 `AuditEngine`（`audit.go:7-9`）。因为 `audit` 是单例变量，**自定义审计插件的 init() 只要在 model/audit.go 之后执行，就能覆盖默认实现**。

由于 Go 的 `init()` 执行顺序是按包的导入顺序决定的，而 `server/plugin/index.go` 导入了所有插件包，且 `server/model` 包通常被更早导入，所以**自定义插件通常能覆盖 SimpleAudit**。

#### 2.4.2.6 SimpleAudit 不做的事（总结）

| 行为 | SimpleAudit 是否做 |
|------|-------------------|
| 返回搜索表单定义（9 个字段） | ✅ |
| 定义操作类型枚举 | ✅ |
| 返回占位提示 HTML + CSS | ✅ |
| 读取/写入任何持久化存储 | ❌ |
| 解析/使用搜索过滤参数 | ❌ |
| 按时间分页 | ❌ |
| 按用户/路径/后端过滤 | ❌ |
| 返回结构化数据（JSON） | ❌（只返回 HTML 字符串） |
| 批量导出 | ❌ |

### 2.5 衔接点 1：Save 失败时审计日志如何落地、操作记录与失败结果对齐

#### 2.5.1 两条独立的日志链

Filestash 有两条独立的日志链路，**默认设计没有将它们关联**：

| 链路 | 触发时机 | 执行方式 | 能拿到什么 | 位置 |
|------|----------|----------|-----------|------|
| **Audit Workflow 链** | `auth.Save()` 调用时（**操作前**） | 同步发布，异步执行 | 操作类型、路径、会话、用户 | `server/pkg/workflow/trigger/fileaction.go:91-98` |
| **HTTP Logger 链** | 请求处理完成后（**操作后**） | 异步 goroutine | HTTP 状态码、请求/响应详情 | `server/middleware/index.go:35` |

时序对比（以 `FileSave` 为例）：

```
T0: auth.Save(ctx, path) → processFileAction() → TriggerEvents()
    ↑ 这里已经发布了 audit event，但 Backend.Save 还没执行
    ↑ workflow job 已经创建，状态是 READY
T1: ctx.Backend.Save(path, req.Body) → 返回 ErrQuotaExceeded
    ↑ 配额超限，操作失败
T2: SendErrorResult(res, NewError(err.Error(), 403))
    ↑ 写响应，status=403
T3: go logger(&app, &resw, req)
    ↑ 这里能拿到 status=403，但 audit event 早已发布
```

#### 2.5.2 默认设计的核心问题

- Audit Workflow 链**在操作执行前**就发布了事件，Job 的 `input` 中只有操作元数据（event、path、session 等），没有最终执行结果
- HTTP Logger 链**在操作完成后**异步执行，能拿到 `resw.Status()`，但默认只写入 telemetry buffer，不与 audit 表关联
- 两条链通过 `X-Request-ID` 可以关联，但默认没有任何代码做这件事

#### 2.5.3 对齐机制（需要自定义插件实现）

要让操作记录包含成功/失败状态，自定义审计插件需要**同时注册两个 Hook**：

```go
type FullAuditPlugin struct {
    db *sql.DB
}

// 1. 注册为 AuthorisationMiddleware，在操作前写入 audit 表（状态=PENDING）
func (this FullAuditPlugin) Save(ctx *App, path string) error {
    requestID := ctx.Context.Value("X-Request-ID").(string)
    _, err := this.db.Exec(`
        INSERT INTO audit (request_id, event, path, session_id, user_id, status, created_at)
        VALUES (?, 'save', ?, ?, ?, 'PENDING', CURRENT_TIMESTAMP)
    `, requestID, path, GenerateID(ctx.Session), getUser(ctx.Session))
    return err
}
// ... 实现其他 7 个 IAuthorisation 方法

// 2. 注册为 Middleware，在请求完成后更新 audit 表的状态
func (this FullAuditPlugin) Middleware(next HandlerFunc) HandlerFunc {
    return HandlerFunc(func(ctx *App, res http.ResponseWriter, req *http.Request) {
        next(ctx, res, req)
        if obj, ok := res.(*middleware.ResponseWriter); ok {
            status := obj.Status()
            requestID := res.Header().Get("X-Request-ID")
            auditStatus := "SUCCESS"
            if status >= 400 {
                auditStatus = "FAILURE"
            }
            go func() {
                this.db.Exec(`
                    UPDATE audit SET status = ?, finished_at = CURRENT_TIMESTAMP
                    WHERE request_id = ? AND status = 'PENDING'
                `, auditStatus, requestID)
            }()
        }
    })
}

// 3. 注册为 IAuditPlugin 提供查询能力
func (this FullAuditPlugin) Query(ctx *App, params map[string]string) (AuditQueryResult, error) {
    // 查询 audit 表，支持按 status 过滤成功/失败记录
}

func init() {
    Hooks.Register.AuthorisationMiddleware(FullAuditPlugin{db: db})
    Hooks.Register.Middleware(FullAuditPlugin{db: db}.Middleware)
    Hooks.Register.AuditEngine(FullAuditPlugin{db: db})
}
```

#### 2.5.4 X-Request-ID 的生成位置

`server/middleware/telemetry.go:73-80` 中为每个请求生成唯一 ID：

```go
RequestID: func() string {
    if req.Header.Get("X-Request-ID") != "" {
        return req.Header.Get("X-Request-ID")
    }
    b := make([]byte, 16)
    rand.Read(b)
    return hex.EncodeToString(b)
}(),
```

这个 ID 会被放入 `LogEntry.RequestID`，并通过 `res.Header().Set("X-Request-ID", point.RequestID)` 返回给客户端。

### 2.6 审计日志的查询接口

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

### 2.6.1 衔接点 8：审计检索 API 支持的过滤条件、按时间分页实现

#### 2.6.1.1 框架层：纯透传，无过滤逻辑

框架层的 `FetchAuditHandler`（`server/ctrl/admin.go:138-158`）**完全透明地把 URL query 参数透传给插件**：

```go
searchParams := map[string]string{}
for key, element := range _get {
    if len(element) == 0 { continue }
    searchParams[key] = element[0]
}
result, err := plg.Query(ctx, searchParams)
```

**框架本身不做任何参数验证、SQL 构造、分页、索引选择**——所有逻辑完全由 `IAuditPlugin.Query()` 实现决定。

#### 2.6.1.2 前端表单参数 → URL query 的映射

前端 `ctrl_activity_audit.js:48-65` 中的逻辑：

```javascript
// 每次表单字段变化（防抖 1s 后）
const formData = new FormData($form);
const p = new URLSearchParams();
for (const [key, value] of formData.entries()) {
    if (!value) continue;
    // 表单字段名格式是 "search.date from" / "search.action" 等
    // 去掉 "search." 前缀后作为 query key
    p.set(key.replace(new RegExp("^search\."), ""), `${value}`);
}
// GET /admin/api/audit?date+from=xxx&action=save&path=%2Fdata...
```

**框架定义的表单字段**（`server/model/audit.go:11-56`）对应以下过滤条件：

| 表单字段 (FormElement.Name) | URL Query Key | 前端类型 | 说明 |
|----------------------------|---------------|----------|------|
| `search.date from` | `date from` | `datetime` | 起始时间 |
| `search.date to` | `date to` | `datetime` | 结束时间 |
| `search.action` | `action` | `select` | 操作类型：`rename`/`list`/`download`/`create_folder`/`remove`/`move`/`save_file`/`create_file` |
| `search.path` | `path` | `text` | 源路径（模糊匹配） |
| `search.backend` | `backend` | `text` | 后端类型（如 sftp/s3/local） |
| `search.session` | `session` | `text` | 会话 ID（= GenerateID(ctx.Session)） |
| `search.share` | `share` | `text` | 共享链接 ID（= Share.Id） |
| `search.user` | `user` | `text` | 用户标识 |
| `search.target` | `target` | `text` | 目标路径（rename/move 时的目标） |

**注意**：框架不校验这些值的格式，比如 `date from` 传非法字符串也会原样透传给插件。

#### 2.6.1.3 分页实现：完全缺失

核心框架**不提供任何分页支持**：

- URL 参数中**没有定义 `page`、`pageSize`、`limit`、`offset`、`cursor` 等字段**
- 前端 `AuditForm` 中没有分页控件字段
- 后端 `FetchAuditHandler` 不处理分页参数
- `AuditQueryResult` struct（`server/common/types.go:68-71`）只有 `Form *Form` 和 `RenderHTML string`，**没有 `total`、`pages`、`hasNext` 等元数据**

前端唯一的分页行为是：**表单变化时重新拉取全量数据**（通过 `rxjs.debounceTime(1000)` 防抖），完全靠插件渲染的 HTML 自己实现滚动/分页 UI。

#### 2.6.1.4 时间范围过滤：也完全缺失

虽然表单提供了 `date from` / `date to` 两个 `datetime` 字段，但：

- 框架不校验时间格式
- 框架不保证这两个值一定会被 `SimpleAudit` 使用（`SimpleAudit` 忽略所有参数，直接返回提示 HTML）
- 时间参数传值格式由浏览器 `datetime` input 决定（ISO 8601 字符串如 `2026-06-20T15:30`），但**后端没有解析逻辑**，完全依赖自定义插件解析

#### 2.6.1.5 完整的自定义分页 + 时间过滤实现示例

```go
// DBAuditPlugin.Query 的完整实现
func (this DBAuditPlugin) Query(ctx *App, p map[string]string) (AuditQueryResult, error) {
    // 1. 解析过滤条件
    where := []string{}
    args := []interface{}{}

    if v, ok := p["date from"]; ok && v != "" {
        where = append(where, "created_at >= ?")
        t, _ := time.Parse(time.RFC3339, v)
        args = append(args, t.UTC().Format("2006-01-02 15:04:05"))
    }
    if v, ok := p["date to"]; ok && v != "" {
        where = append(where, "created_at <= ?")
        t, _ := time.Parse(time.RFC3339, v)
        args = append(args, t.UTC().Format("2006-01-02 15:04:05"))
    }
    if v, ok := p["action"]; ok && v != "" {
        where = append(where, "event = ?")
        args = append(args, v)
    }
    if v, ok := p["path"]; ok && v != "" {
        where = append(where, "path LIKE ?")
        args = append(args, "%"+v+"%")
    }
    if v, ok := p["user"]; ok && v != "" {
        where = append(where, "user_id = ?")
        args = append(args, v)
    }
    // ... action, backend, session, share, target 同理

    // 2. 解析分页参数（插件自定义，不在 Form 定义中）
    page := 1
    pageSize := 50
    if v, ok := p["page"]; ok { page, _ = strconv.Atoi(v) }
    if v, ok := p["pageSize"]; ok { pageSize, _ = strconv.Atoi(v) }
    if page < 1 { page = 1 }
    if pageSize > 500 { pageSize = 500 }
    offset := (page - 1) * pageSize

    // 3. 查总数（用于分页控件显示）
    var total int
    countSQL := `SELECT COUNT(*) FROM audit_log`
    if len(where) > 0 {
        countSQL += " WHERE " + strings.Join(where, " AND ")
    }
    DB.QueryRow(countSQL, args...).Scan(&total)

    // 4. 查数据（用 created_at DESC 倒序，借助 idx_audit_created_at 索引）
    querySQL := `SELECT id, event, path, target, backend, user_id, status, created_at
        FROM audit_log`
    if len(where) > 0 {
        querySQL += " WHERE " + strings.Join(where, " AND ")
    }
    querySQL += " ORDER BY created_at DESC LIMIT ? OFFSET ?"
    args = append(args, pageSize, offset)

    rows, err := DB.Query(querySQL, args...)
    // ... 渲染成 RenderHTML 表格
}
```

#### 2.6.1.6 两种返回结果的路径对照

| 数据接口 | 路径 | 过滤能力 | 分页 | 结构化 |
|----------|------|---------|------|--------|
| `GET /admin/api/audit` | `FetchAuditHandler` → `IAuditPlugin.Query()` | 9 个字段（插件实现） | ❌ 无（需自建） | ❌ 返回 HTML，不是 JSON |
| `GET /admin/api/logs` | `FetchLogHandler` → 读本地日志文件 | 仅 `maxSize`（倒序尾部 N 字节） | ❌ 无 | ❌ 返回纯文本 |
| Workflow `GET /admin/api/workflow/{id}` | 管理后台看 Job History | 仅按 `related_workflow` 过滤 | ❌ 取最近 3000 条 | ✅ JSON |

**Workflow Job History**（`server/pkg/workflow/model/workflow.go:103-136`）是唯一有分页雏形的持久化查询：

```sql
SELECT COALESCE(JSON_GROUP_ARRAY(JSON_OBJECT(...)), JSON_ARRAY()) FROM (
    SELECT * FROM jobs j
    WHERE j.related_workflow = w.id
    ORDER BY j.created_at DESC
    LIMIT 3000    -- 硬编码 LIMIT，没有 OFFSET / page 控制
) j
```

### 2.6.2 衔接点 12：批量导出、异步聚合与归档路径

#### 2.6.2.1 核心结论：全部缺失

Filestash 核心框架**完全不提供**以下能力：

| 能力 | 框架默认 | 可复用体系 | 需自建 |
|------|---------|-----------|--------|
| 批量导出（CSV/JSON/XLSX） | ❌ 无 | 无 | ✅ |
| 异步聚合（按天/周/用户统计） | ❌ 无 | 无 | ✅ |
| 自动归档（历史数据迁移、冷存储） | ❌ 无 | 无 | ✅ |
| 数据保留策略（N 天后自动删除） | ❌ 无 | 无 | ✅ |
| 导出进度追踪 | ❌ 无 | 无 | ✅ |

`AuditQueryResult` 的唯一返回格式是 `Form + RenderHTML`（HTML 字符串），**没有 JSON 数据接口，没有流式下载接口**。

#### 2.6.2.2 为什么缺失——设计定位

审计查询接口的设计定位是**"管理后台页面渲染"**，不是"数据导出 API"：

1. **返回 HTML 不是 JSON**：`AuditQueryResult.RenderHTML` 是预渲染好的 HTML 表格，前端直接插入 DOM，不需要额外解析
2. **查询耦合表单定义**：返回结果同时包含 `Form` 结构，前端用同一个接口既拿表单定义又拿结果
3. **单页展示为主**：没有分页暗示着默认数据量不会太大，单页展示即可

#### 2.6.2.3 唯一沾边的"导出"：本地日志文件

`/admin/api/logs` 接口（`FetchLogHandler`）能下载纯文本日志，但那是**HTTP 访问日志**，不是审计日志：

- 格式：纯文本行（`2026/06/20 10:30:45 HTTP 200 GET /api/files/cat?path=...`）
- 不支持过滤：只有 `maxSize` 参数倒序读取尾部 N 字节
- 不是结构化数据：没法直接做统计分析

#### 2.6.2.4 唯一沾边的"聚合"：Workflow Job 统计

Workflow 的 `GetWorkflow` 查询中附带了 Job 统计（`workflow.go:103-136`）：

```sql
JSON_OBJECT(
    'id', w.id,
    'name', w.name,
    ...,
    'jobs', (
        SELECT COALESCE(JSON_GROUP_ARRAY(JSON_OBJECT(...)), JSON_ARRAY())
        FROM (
            SELECT * FROM jobs j
            WHERE j.related_workflow = w.id
            ORDER BY j.created_at DESC
            LIMIT 3000
        ) j
    )
)
```

但这是 workflow 执行历史，不是审计操作记录。而且是**一次性把 3000 条全部塞进 JSON**，不是流式分页。

#### 2.6.2.5 自定义批量导出方案

```go
// 注册一个新的 API Handler 用于导出
func init() {
    // 方式 1：通过 Middleware 拦截 /admin/api/audit/export 路径
    Hooks.Register.Middleware(exportMiddleware)

    // 方式 2：在 IAuditPlugin.Query 中识别特殊参数
    // 比如 searchParams["format"] == "csv" 时返回 CSV
}

func (this DBAuditPlugin) Query(ctx *App, p map[string]string) (AuditQueryResult, error) {
    // 如果请求格式是 csv，直接设置响应头返回 csv
    if p["format"] == "csv" {
        if w, ok := ctx.Context.Value("response_writer").(http.ResponseWriter); ok {
            w.Header().Set("Content-Type", "text/csv; charset=utf-8")
            w.Header().Set("Content-Disposition",
                "attachment; filename=audit_"+time.Now().Format("20060102")+".csv")
            // 流式写入 CSV
            rows, _ := DB.Query("SELECT ... FROM audit_log WHERE ...")
            fmt.Fprintf(w, "id,event,path,user,backend,status,created_at\n")
            for rows.Next() {
                var id int
                var event, path, user, backend, status, createdAt string
                rows.Scan(&id, &event, &path, &user, &backend, &status, &createdAt)
                fmt.Fprintf(w, "%d,%s,%s,%s,%s,%s,%s\n",
                    id, csvEscape(event), csvEscape(path),
                    csvEscape(user), csvEscape(backend),
                    csvEscape(status), createdAt)
            }
            return AuditQueryResult{}, nil
        }
    }
    // ... 正常 HTML 渲染
}
```

#### 2.6.2.6 自定义异步聚合与归档方案

```go
func init() {
    // 在 Onload 中启动聚合 + 归档的定时任务
    Hooks.Register.Onload(func() {
        go func() {
            ticker := time.NewTicker(24 * time.Hour)
            defer ticker.Stop()
            for range ticker.C {
                aggregateAuditLogs()   // 按天聚合
                archiveOldAuditLogs()  // 归档 90 天前的数据
                purgeExpiredLogs()     // 删除超过保留期的数据
            }
        }()
    })
}

func aggregateAuditLogs() {
    // 生成每日聚合表：audit_daily_stats(date, user_id, backend, event, count)
    DB.Exec(`
        INSERT OR REPLACE INTO audit_daily_stats (date, user_id, backend, event, count)
        SELECT DATE(created_at) as date, user_id, backend, event, COUNT(*)
        FROM audit_log
        WHERE created_at >= DATE('now', '-1 day')
        GROUP BY date, user_id, backend, event
    `)
}

func archiveOldAuditLogs() {
    // 把 90 天前的日志归档到另一张表或外部存储
    DB.Exec(`
        INSERT INTO audit_log_archive
        SELECT * FROM audit_log WHERE created_at < DATE('now', '-90 days')
    `)
    DB.Exec(`DELETE FROM audit_log WHERE created_at < DATE('now', '-90 days')`)
}
```

### 2.7 审计 Form 支持的查询字段

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

### 3.2.1 衔接点 9：配额超限通知如何对接邮件 / Webhook / 消息总线

#### 3.2.1.1 核心设计：配额超限本身不触发通知

**默认逻辑中，配额超限错误不会触发任何通知回调链**。

`Backend.Save()` 返回 `ErrQuotaExceeded` → `FileSave` 调用 `SendErrorResult(res, err)` → 返回 HTTP 4xx → 请求结束。

整个流程中：
- ❌ **没有专门的"配额超限事件"**（不同于文件操作事件）
- ❌ 没有 Hook 点允许插件注册 `OnQuotaExceeded` 回调
- ❌ 没有从错误响应中自动捕获配额错误并触发 workflow 的逻辑

**要让配额超限产生通知，只能通过现有的 Workflow 体系间接实现**，有两条路径：

#### 3.2.1.2 路径 A：通过 Workflow 自定义 Trigger + Condition

用户在管理后台创建一个新的 Workflow：

```
Trigger: event (文件事件)
Condition: event = "stat"  (对应 Save 操作)
            AND path LIKE "/quota-sensitive/*"
Action 1: run/api → 调用用户自建的配额检查服务（查询已用空间 + 预估本次大小）
            如果配额够 → 无操作
            如果配额不够 → 触发 Action 2
Action 2: notify/email → 发通知给管理员
```

**这不是真正的"配额超限触发"**，因为 Workflow 的 event trigger 是在 `auth.Save()` 时（操作前）触发的，此时 Backend.Save 还没执行，根本不知道会不会超限。

所以这种方式只能做**预检查通知**（"你要传的文件太大了"），做不到**真实超限通知**（"后端说配额不够了"）。

#### 3.2.1.3 路径 B：自定义 Middleware 捕获响应状态（真正的超限通知）

通过自定义 Middleware 包裹所有上传请求，在响应已经写入后检查是否有配额错误：

```go
type QuotaNotifier struct{}

func (this QuotaNotifier) Middleware(next HandlerFunc) HandlerFunc {
    return HandlerFunc(func(ctx *App, res http.ResponseWriter, req *http.Request) {
        // 只处理上传路径
        if req.URL.Path != "/api/files/save" {
            next(ctx, res, req)
            return
        }

        next(ctx, res, req)

        // 检查响应状态码
        rw, ok := res.(*middleware.ResponseWriter)
        if !ok || rw.Status() < 400 {
            return
        }

        // 检查响应体中是否包含配额错误消息
        // 注意：需要缓存响应体才能读取，这里省略缓存逻辑
        body := getResponseBody(rw)
        if !strings.Contains(body, "Quota exceeded") &&
           !strings.Contains(body, "No space left") &&
           !strings.Contains(body, "NFS4ERR_DQUOT") {
            return
        }

        // 触发通知
        user := getUser(ctx.Session)
        backend := ctx.Session["type"]
        path := req.URL.Query().Get("path")

        // 方式 1：直接发邮件
        go this.sendEmail(user, backend, path, body)

        // 方式 2：调 Webhook
        go this.callWebhook(map[string]string{
            "type":      "quota_exceeded",
            "user":      user,
            "backend":   backend,
            "path":      path,
            "timestamp": time.Now().UTC().Format(time.RFC3339),
        })

        // 方式 3：写入消息总线（通过事件总线）
        go workflow.TriggerEvents(quotaEventChannel, "quota_event",
            workflow.TriggerCallback(func(p map[string]string) bool {
                return true
            }))
    })
}

func init() {
    Hooks.Register.Middleware(QuotaNotifier{}.Middleware)
}
```

#### 3.2.1.4 现成的通知机制：Workflow Action

Filestash 内置了 3 种 Workflow Action（`server/pkg/workflow/actions/`），可以通过 Trigger 调用：

##### Action 1：`notify/email`（`actions/notify_email.go`）

```go
type ActionNotifyEmail struct{}

func (this *ActionNotifyEmail) Manifest() WorkflowSpecs {
    return WorkflowSpecs{
        Name:  "notify/email",
        Specs: Form{Elmnts: []FormElement{
            {Name: "email",   Type: "text"},
            {Name: "subject", Type: "text"},
            {Name: "message", Type: "long_text"},
        }},
    }
}

func (this *ActionNotifyEmail) Execute(params, input map[string]string) (map[string]string, error) {
    // 从全局配置读 SMTP 参数
    email := struct {
        Hostname: Config.Get("email.server").String()
        Port:     Config.Get("email.port").Int()
        Username: Config.Get("email.username").String()
        Password: Config.Get("email.password").String()
        From:     Config.Get("email.from").String()
        // 以下支持从 input 变量模板渲染
        To:      Render(params["email"], input)
        Subject: Render(params["subject"], input)
        Message: Render(params["message"], input)
    }{}
    m := gomail.NewMessage()
    m.SetHeader("From", email.From)
    m.SetHeader("To", email.To)
    m.SetBody("text/html", strings.ReplaceAll(email.Message, "\n", "<br>"))
    mail := gomail.NewDialer(email.Hostname, email.Port, email.Username, email.Password)
    return input, mail.DialAndSend(m)
}
```

**SMTP 配置来源**：`server/common/config.go` 中预定义了 `email.server` / `email.port` / `email.username` / `email.password` / `email.from` 配置项，管理员在后台配置。

##### Action 2：`run/api`（`actions/run_api.go`）

```go
type RunApi struct{}

func (this *RunApi) Execute(params, input map[string]string) (map[string]string, error) {
    // params.url / params.method / params.headers / params.body
    // 都支持用 {{变量名}} 模板从 input 渲染
    req, _ := http.NewRequest(
        params["method"],
        Render(params["url"], input),
        bytes.NewBufferString(Render(params["body"], input)))

    // headers 按行解析，每行 "Key: Value"
    for _, header := range strings.Split(Render(params["headers"], input), "\n") {
        if parts := strings.SplitN(strings.TrimSpace(header), ":", 2); len(parts) == 2 {
            req.Header.Add(strings.TrimSpace(parts[0]), strings.TrimSpace(parts[1]))
        }
    }
    if strings.Contains("POST/PUT/PATCH", params["method"]) {
        req.Header.Set("Content-Type", "application/json")
    }
    resp, err := HTTP.Do(req)
    if resp.StatusCode < 200 || resp.StatusCode >= 300 {
        return input, NewError(fmt.Sprintf("received status code is %d", resp.StatusCode), resp.StatusCode)
    }
    // 把响应体放到 output["http::response"] 供后续步骤使用
    output["http::status"] = string(resp.StatusCode)
    output["http::response"] = string(responseBody)
    return output, nil
}
```

这就是**Webhook** 能力。任何 URL 都可以调。

##### Action 3：`tools/debug`（`actions/tools_debug.go`）

```go
func (this *ToolsDebug) Execute(params, input map[string]string) (map[string]string, error) {
    Log.Info("[workflow] action=tools/debug input=%v", input)
    return input, nil
}
```

只是把 input 打到日志，用来调试 workflow 触发时的变量。

#### 3.2.1.5 变量模板渲染机制

`notify/email` 和 `run/api` 中的 `To/Subject/Message/URL/Body/Headers` 都通过 `Render(template, variables)` 处理（`actions/utils.go:7-13`）：

```go
func Render(templateText string, variables map[string]string) string {
    rendered, err := TmplExec(templateText, TmplParams(variables))
    if err != nil {
        return templateText
    }
    return rendered
}
```

`TmplExec` 支持 `{{event}}`、`{{path}}`、`{{user}}` 等变量替换，变量来源于 **workflow job input**（即 `fileactionCallback(params)` 返回的 map）。

#### 3.2.1.6 文件事件回调的 input 内容

`fileactionCallback`（`pkg/workflow/trigger/fileaction.go`）返回的 input 变量包含：

```go
fileactionCallback(params map[string]string) TriggerCallback {
    return func(p map[string]interface{}) bool {
        // 匹配：p["event"] == params["event"] && p["path"] ~= params["path"]
        // 然后发布时，p 就是 job 的 input，包含：
        //   event  = "ls" / "cat" / "mkdir" / "rm" / "mv" / "stat" / "touch" / "save"
        //   path   = 操作路径
        //   target = 目标路径（仅 mv 事件有）
        //   以及 ctx.Session 中的 user/username 等（取决于自定义 AuthorisationMiddleware）
    }
}
```

在 workflow 的 Action 中可以用 `{{event}}`、`{{path}}`、`{{target}}` 引用这些变量。

#### 3.2.1.7 完整配额超限通知回调链（自定义方案）

```
用户上传 POST /api/files/save
    │
    ▼
QuotaNotifier Middleware
    │
    ▼
auth.Save(ctx, path)
    ├─ processFileAction() → TriggerEvents(event trigger)
    │   └─ 如果配置了 event workflow → 创建 job → worker 池异步执行
    │       ├─ Action 1: run/api → POST 到自建配额检查服务
    │       │   Body: {"path": "{{path}}", "user": "{{user}}"}
    │       │   Response: {"quota": {"used": 98GB, "limit": 100GB}}
    │       ├─ Action 2: notify/email → Subject: "即将达到配额"
    │       └─ Action 3: run/api → POST 到 Slack/钉钉/消息总线
    │
    ▼
ctx.Backend.Save(path, reader) → 返回 ErrQuotaExceeded
    │
    ▼
SendErrorResult(res, 403, "Quota exceeded")
    │
    ▼
QuotaNotifier Middleware (after next())
    ├─ 检查 status=403 且 body 含 "Quota exceeded"
    └─ 异步通知：
        ├─ notify/email → admin@example.com
        ├─ run/api → POST 到 Slack Incoming Webhook
        │   {
        │     "text": "用户 alice 在 SFTP backend 上传 /data/report.csv
        │             时配额超限，已用 98GB/100GB"
        │   }
        └─ run/api → POST 到内部消息总线 /kafka/quota-topic
```

**总结：配额超限通知没有专门的 Hook，需要靠自定义 Middleware 拦截响应状态，或者靠 Workflow 做预检查通知。邮件和 Webhook 功能已经由 `notify/email` 和 `run/api` 两个内置 Workflow Action 提供。**

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

### 3.3.1 衔接点 3：大文件分块上传 mid-stream 配额检查与最终入库前二次校验

#### 3.3.1.1 TUS 分块上传的流式架构

分块上传的核心是 `io.Pipe` + goroutine 的流式设计（`server/ctrl/files.go:672-685`）：

```go
func createChunkedUploader(save func(path string, file io.Reader) error, 
                           path string, size uint64) *chunkedUpload {
    r, w := io.Pipe()
    done := make(chan error, 1)
    go func() {
        done <- save(path, r)  // 后台 goroutine 立即开始流式写入
    }()
    return &chunkedUpload{
        fn:     save,
        stream: w,
        done:   done,
        offset: 0,
        size:   size,
    }
}
```

完整时序：

```
TUS POST (创建会话):
  ├─ auth.Save(ctx, path)                ← ★ 审计/配额预检查触发点
  │                                      ← 这里可以拿到 Upload-Length 做预检查
  ├─ 解析 Upload-Length (totalSize)
  ├─ b, _ := ctx.Backend.Init(...)       ← 创建新的后端实例
  ├─ createChunkedUploader(b.Save, path, size)
  │   ├─ r, w := io.Pipe()
  │   └─ go func() { done <- save(path, r) }()  ← 立即启动流式写入
  └─ 返回 201 Created

TUS PATCH (上传第 N 块):
  ├─ 校验 Upload-Offset 匹配
  ├─ uploader.Next(reader)
  │   ├─ io.Copy(this.stream, body)      ← 数据通过 pipe 流向 save()
  │   │                                  ← 如果 save() 中途 quota exceeded，
  │   │                                  ← pipe 会返回错误，io.Copy 终止
  │   ├─ 累计 offset
  │   └─ 返回 err（如果有）
  ├─ 校验 checksum
  └─ 如果 newOffset == totalSize:
      ├─ uploader.Close()
      │   ├─ this.stream.Close()         ← 关闭写端，通知 save() 没有更多数据
      │   └─ err := <-this.done          ← ★ 等待 save() 最终返回值（二次校验点）
      └─ 删除缓存
```

#### 3.3.1.2 配额检查的三个可能位置

| 检查位置 | 时机 | 能拿到什么 | 怎么自定义 | 现有代码是否有 |
|----------|------|-----------|-----------|----------------|
| **预检查** | TUS POST 时，`auth.Save()` 中 | `Upload-Length`（文件总大小） | 自定义 `AuthorisationMiddleware.Save()`，从 req.Header 拿 `Upload-Length` | ❌ 没有 |
| **Mid-stream 检查** | TUS PATCH 时，`io.Copy` 过程中 | 实际写入的字节流 | 由 `Backend.Save()` 在流式写入时检查，通过 `io.Pipe` 反向传播错误 | ✅ 靠后端 |
| **最终校验** | `uploader.Close()` 等待 `done` channel | `save()` 的最终返回值 | 框架本身已有，但只是透传，没有额外校验 | ✅ 透传 |

#### 3.3.1.3 Mid-stream 错误传播机制

当 `Backend.Save()` 中途返回 quota exceeded 时：

```go
// 后台 goroutine 中的 save() 流式写入
func (sftp *Sftp) Save(path string, file io.Reader) error {
    for {
        buf := make([]byte, 32*1024)
        n, err := file.Read(buf)
        if n > 0 {
            // 写入到后端存储
            if _, err := sftp.client.WriteAt(buf[:n], offset); err != nil {
                // 比如 SFTP 返回 FX_QUOTA_EXCEEDED (15)
                if statusCode == 15 {
                    return NewError("Quota exceeded", 400)
                }
            }
        }
        if err == io.EOF {
            break
        }
    }
    return nil
}
```

当 `Save()` 返回错误时，`io.Pipe` 的读端会关闭，下一次 `io.Copy` 到写端时会收到 `io.ErrClosedPipe` 错误：

```go
// chunkedUpload.Next() 中的 io.Copy
func (this *chunkedUpload) Next(body io.ReadCloser) error {
    n, err := io.Copy(this.stream, body)
    // 如果 save() 已经返回错误并关闭了 pipe 读端，
    // 这里的 err 就是 io.ErrClosedPipe
    body.Close()
    this.mu.Lock()
    this.offset += uint64(n)
    this.mu.Unlock()
    return err
}
```

这个错误会一路返回到 Controller，最终以 `403 Forbidden` 返回给客户端。

#### 3.3.1.4 入库前二次校验的缺失

**框架层没有配额的二次校验**，只有以下校验：

1. `offset == totalSize` 校验（`files.go:649-655`）- 防止 offset 异常
2. `checksum` 校验（`files.go:645-647`）- 防止数据损坏
3. `uploader.Close()` 等待 `save()` 的最终返回值（`files.go:657-661`）- 只是透传后端的最终错误

**没有**：
- 已写入字节数的二次确认
- 文件最终大小与 `Upload-Length` 的一致性校验
- 配额的二次计算（used + newFileSize <= limit）

#### 3.3.1.5 如何自定义完整的配额检查链路

```go
type FullQuotaChecker struct{}

// 1. 预检查：在 TUS POST 时检查总大小
func (this FullQuotaChecker) Save(ctx *App, path string) error {
    // 注意：需要自定义 Middleware 把 req 放到 ctx 中
    req := ctx.Context.Value("http_request").(*http.Request)
    if _, ok := req.Header["Tus-Resumable"]; ok && req.Method == http.MethodPost {
        uploadLen, _ := strconv.ParseUint(req.Header.Get("Upload-Length"), 10, 0)
        used := GetUserBackendUsage(GenerateID(ctx.Session))
        limit := GetBackendQuota(GenerateID(ctx.Session))
        if used + int64(uploadLen) > limit {
            return NewError("Quota exceeded (pre-check)", 400)
        }
    }
    return nil
}

// 2. Mid-stream 检查：包装 Backend.Save，统计实际写入字节
func (this *QuotaBackend) Save(path string, r io.Reader) error {
    counter := &CountingReader{Reader: r}
    err := this.real.Save(path, counter)
    if err == nil {
        // 3. 入库前二次校验：实际写入大小与预期一致
        if counter.BytesRead != expectedSize {
            this.real.Rm(path) // 回滚
            return NewError("Size mismatch", 400)
        }
        AddUsedStorage(this.backendID, counter.BytesRead)
    }
    return err
}
```

#### 3.3.1.6 缓存过期的清理机制

`chunkedUploadCache` 有 24 小时 TTL，过期时会自动清理（`files.go:687-700`）：

```go
chunkedUploadCache.OnEvict(func(key string, value interface{}) {
    c := value.(*chunkedUpload)
    if err := c.Close(); err != nil {
        Log.Warning("ctrl::files::chunked::cleanup action=close err=%s", err.Error())
    }
})
```

这意味着：
- 24 小时内没有续传的上传会被自动取消
- `Close()` 会等待 `save()` 返回最终错误
- 配额超限的错误在这里也能被捕获

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

### 3.5 衔接点 2：同一用户多后端连接时容量限制的跨后端聚合路径

#### 3.5.1 核心结论

**Filestash 核心框架本身不做跨后端的容量聚合**。每个后端连接是独立的，配额/容量由各后端自行管理。

#### 3.5.2 身份标识体系

Filestash 有两套身份标识，粒度不同：

| 标识 | 生成方式 | 粒度 | 用途 |
|------|----------|------|------|
| `GenerateID(ctx.Session)` | 对 session 参数哈希 | 按**后端连接** | 区分不同的后端连接实例 |
| `getUser(ctx.Session)` | 从 session 提取 `user`/`username` 字段 | 按**用户** | 同一个用户在不同后端可能有相同/不同的用户名 |

`GenerateID` 的实现（`server/common/crypto.go:193-218`）：

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
        case "password":
        case "path":
        case "session":
        case "timestamp":
        default:
            if val := params[key]; val != "" {
                p += key + "=>" + params[key] + ", "
            }
        }
    }
    // 会包含 type、hostname、username 等字段
    // 所以不同的后端连接（即使同一个用户）会生成不同的 ID
    p += "salt=>" + SECRET_KEY
    return Hash(p, 20)
}
```

**关键**：`GenerateID` 会包含 `type`、`hostname`、`username` 等连接参数，所以**同一个用户挂 SFTP 和 S3 两个后端会生成两个不同的 ID**。

#### 3.5.3 跨后端聚合的缺失路径

核心框架中没有以下逻辑：

1. **没有按用户维度的 storage usage 表** - 所有的使用量都是各后端自己维护的
2. **没有统一的配额配置入口** - 配额是在后端存储系统中配置的，不是在 Filestash 中
3. **没有聚合查询 API** - 没有接口能返回"用户 A 在所有后端的总使用量"

在现有插件的实现中可以看到，`tenantID` 都是用 `GenerateID(ctx.Session)`：

- `plg_widget_recent/index.go:74` - `StoreRecent(GenerateID(ctx.Session), ...)`
- `plg_widget_description/handler.go:24` - `WHERE backend = ?` 用 `GenerateID(ctx.Session)`
- `plg_metadata_sqlite/index.go:34` - `tenantID := GenerateID(ctx.Session)`

这说明所有的插件数据都是按**后端连接**隔离的，不是按用户聚合的。

#### 3.5.4 如何自定义跨后端聚合

如果需要实现"用户总配额 = 所有后端使用量之和不能超过 X"，需要自定义插件：

```go
type CrossBackendQuota struct {
    db *sql.DB
}

func (this CrossBackendQuota) Save(ctx *App, path string) error {
    // 1. 获取用户标识（注意：不同后端 session 中的 user 字段可能不同）
    user := getUser(ctx.Session)
    if user == "" || user == "unknown" {
        return nil // 无法识别用户，跳过
    }

    // 2. 获取本次上传的大小（从 Upload-Length header 拿）
    // 注意：这需要在 Controller 层把 req 传到 ctx 中，或者自定义 Middleware
    // uploadSize := ctx.Context.Value("upload_size").(int64)

    // 3. 统计该用户所有后端连接的已用空间
    var totalUsed int64
    err := this.db.QueryRow(`
        SELECT SUM(bytes_used) 
        FROM storage_usage 
        WHERE user_id = ?
    `, user).Scan(&totalUsed)
    if err != nil {
        return err
    }

    // 4. 检查总配额
    var totalLimit int64 = 10 * 1024 * 1024 * 1024 // 10GB
    if totalUsed + uploadSize > totalLimit {
        return NewError("Total quota exceeded across all backends", 400)
    }

    return nil
}

// 5. 还需要一个定时任务或在操作成功后，同步各后端的使用量到 storage_usage 表
func syncBackendUsage(user string, backendType string, backendID string) {
    // 调用各后端的 Stat/Ls 递归计算使用量
    // 更新 storage_usage 表
}
```

#### 3.5.5 多后端连接的会话存储

`server/middleware/session.go` 中管理会话 cookie：

```go
func CookieName(idx int) string {
    if idx == 0 {
        return COOKIE_NAME_AUTH
    }
    return COOKIE_NAME_AUTH + strconv.Itoa(idx)
}
```

用户可以同时连接多个后端，每个后端的 session 存在不同的 cookie 中（`auth`、`auth1`、`auth2`...）。这就是"同一用户挂多个后端连接"的实现方式。

当用户切换后端时，`SessionStart` 中间件会从对应 cookie 中解密出 session map，`GenerateID(ctx.Session)` 就会生成对应后端连接的 ID。

### 3.6 衔接点 4：TUS 断点续传已上传字节与配额对账、续传中途配额变化响应

#### 3.6.1 已上传字节的状态存储

TUS 断点续传的中间状态**完全在内存中**，没有持久化。上传进度存储在 `chunkedUploadCache`（`server/ctrl/files.go:477`）：

```go
var chunkedUploadCache AppCache
```

`AppCache`（`server/common/cache.go:51-63`）底层使用 `patrickmn/go-cache`：

```go
func NewAppCache(arg ...time.Duration) AppCache {
    var retention time.Duration = 5
    var cleanup time.Duration = 10
    if len(arg) > 0 {
        retention = arg[0]
        if len(arg) > 1 {
            cleanup = arg[1]
        }
    }
    c := AppCache{}
    c.Cache = cache.New(retention*time.Minute, cleanup*time.Minute)
    return c
}
```

初始化时传入 `60*24, 1`（`files.go:688`），即 **retention = 1440 分钟 = 24 小时**，cleanup 间隔 = 1 分钟。

这意味着：
- **已上传字节偏移量只在内存中**，通过 `chunkedUpload.Meta()` 返回 `(offset, size)`
- 如果进程重启，所有进行中的上传状态丢失
- 客户端必须重新 `POST` 创建新的上传会话

#### 3.6.2 续传时的对账机制

客户端续传通过 `HEAD` 请求查询当前 offset（`files.go:554-567`）：

```go
if proto == "tus" && req.Method == http.MethodHead {
    c := chunkedUploadCache.Get(cacheKey)
    if c == nil {
        SendErrorResult(res, ErrNotFound)  // 会话已过期/丢失
        return
    }
    offset, length := c.(*chunkedUpload).Meta()
    h.Set("Upload-Offset", fmt.Sprintf("%d", offset))
    h.Set("Upload-Length", fmt.Sprintf("%d", length))
    res.WriteHeader(http.StatusNoContent)
}
```

**核心问题：offset 来自内存，不是来自后端存储**

```
客户端 HEAD /api/files/save?path=xxx
    → Upload-Offset: 5242880  (来自 chunkedUpload.offset)
    → Upload-Length: 10485760

但后端存储上该文件可能已经有 5242880 字节（正常）、
也可能只有 2097152 字节（中途 pipe 断开但部分数据已 flush）
→ 内存 offset 与后端实际写入量不对账
```

**代码中没有**：
- 向后端 `Stat()` 查询文件实际大小来校验 offset
- 比对 `Upload-Offset` 与后端实际文件大小
- offset 漂移的自动修正

#### 3.6.3 续传中途配额变化的响应

当配额在续传期间发生变化（如管理员调整了 SFTP 配额、其他用户占满了共享空间），Filestash 的响应完全依赖后端的即时行为：

**场景 1：PATCH 过程中后端返回配额错误**

`uploader.Next(reader)` → `io.Copy(this.stream, body)` → 数据通过 pipe 流入后台 `save()` goroutine → 后端写入时返回 quota exceeded → pipe 读端关闭 → `io.Copy` 收到 `io.ErrClosedPipe` → 返回 403。

这是**即时响应**，不需要额外代码。

**场景 2：两次 PATCH 之间配额变化**

两次 PATCH 之间，没有配额预检查。下一个 PATCH 的数据会直接流入 pipe，后端在写入时才会发现配额不足。

```
PATCH #3 完成 → offset=6MB → 客户端暂停
                    ↓
    此时管理员将配额从 10MB 降到 5MB
                    ↓
PATCH #4 开始 → io.Copy → save() 写入 → 后端返回 quota exceeded → 403
```

**代码中没有**：
- PATCH 请求开始时的配额预检查
- 两次 PATCH 之间的心跳/配额探测
- 配额变化的通知机制

#### 3.6.4 重复 POST 的处理

如果客户端在已有进行中上传时再次发 `POST`，旧的上传会被强制终止（`files.go:569-571`）：

```go
if proto == "tus" && req.Method == http.MethodPost {
    if c := chunkedUploadCache.Get(cacheKey); c != nil {
        chunkedUploadCache.Del(cacheKey)  // 触发 OnEvict → Close() → 等待 save() 返回
    }
    // 创建新的上传会话
}
```

但这里有一个**对账缺口**：`Del` 触发 `OnEvict`，`OnEvict` 会调用 `Close()`，`Close()` 会关闭 pipe 写端并等待 `save()` 返回。如果 `save()` 此时正在写入大量数据，`Close()` 会**阻塞等待**直到写入完成或出错。这段时间内新的 `createChunkedUploader` 中的 `b.Save(path, r)` 可能会与旧的 `save()` 产生**对同一文件的并发写入**，而后端的行为取决于具体实现（SFTP 会覆盖、S3 用 multipart 可能追加或冲突）。

#### 3.6.5 自定义配额对账方案

```go
type QuotaReconciler struct{}

func (this QuotaReconciler) Middleware(next HandlerFunc) HandlerFunc {
    return HandlerFunc(func(ctx *App, res http.ResponseWriter, req *http.Request) {
        // 仅在 TUS PATCH 之前做配额预检查
        if _, ok := req.Header["Tus-Resumable"]; ok && req.Method == http.MethodPatch {
            path := req.URL.Query().Get("path")
            // 1. 查询后端实际文件大小
            if info, err := ctx.Backend.Stat(path); err == nil {
                actualSize := info.Size()
                // 2. 查询当前配额
                quota := GetCurrentQuota(GenerateID(ctx.Session))
                // 3. 如果已经超限，直接拒绝
                if actualSize >= quota {
                    SendErrorResult(res, NewError("Quota already exceeded", 400))
                    return
                }
            }
        }
        next(ctx, res, req)
    })
}
```

### 3.7 衔接点 5：插件热替换/reload 时正在进行的上传处理与资源回收

#### 3.7.1 Filestash 没有运行时插件热替换

**核心结论：Filestash 不支持运行时插件热替换。** 所有插件通过 `init()` 在进程启动时注册（Go 语言的空白导入机制），注册后不可撤销、不可替换。

`server/plugin/index.go:48-50`：

```go
func init() {
    Hooks.Register.Onload(func() { Log.Debug("plugins loaded") })
}
```

所有 `plg_*` 包通过空白导入在编译时确定，运行时无法动态加载新插件。

#### 3.7.2 生命周期钩子

`server/common/plugin.go` 提供了四个生命周期钩子：

| 钩子 | 注册方法 | 调用时机 | 用途 |
|------|----------|----------|------|
| `Onload` | `Hooks.Register.Onload(fn)` | 进程启动后 | 初始化 DB、读取配置 |
| `OnQuit` | `Hooks.Register.OnQuit(fn)` | 进程退出前 | 清理资源 |
| `OnConfig` | `Hooks.Register.OnConfig(fn)` | 配置变更时 | 响应配置热更新 |
| `Middleware` | `Hooks.Register.Middleware(fn)` | 每次请求时 | 插入自定义中间件 |

**配置热更新**是唯一的"类热替换"机制。`OnConfig` 钩子在管理员通过 `/admin/api/config` 修改配置时触发：

```go
// server/common/plugin.go:286-294
var configChange []func()

func (this Register) OnConfig(fn func()) {
    configChange = append(configChange, fn)
}
func (this Get) OnConfig() []func() {
    return configChange
}
```

现有插件用 `OnConfig` 做的事情很有限——主要是切换 UI 补丁（`StaticPatch`）：

```go
// plg_widget_favourite/index.go:26-32
Hooks.Register.OnConfig(func() {
    if PluginEnable() {
        Hooks.Register.StaticPatch(PATCH, WithID("plg_widget_favourite"))
    } else {
        Hooks.Register.StaticPatch([]byte(""), WithID("plg_widget_favourite"))
    }
})
```

#### 3.7.3 进程重启（唯一的"热替换"方式）

当需要更换后端插件时，必须重启整个 Filestash 进程。重启流程：

```
1. SIGTERM / SIGINT 到达
2. 所有 Starter 注册的 HTTP Server 执行 ctx.Done()
   → srv.Shutdown(context.Background())  // 优雅关闭 HTTP 监听
   → 停止接受新连接，等待已有请求完成
3. Hooks.Get.OnQuit() 执行清理回调
4. 进程退出
5. 新进程启动 → init() 注册新插件 → Hooks.Get.Onload() 初始化
```

#### 3.7.4 进程重启对进行中上传的影响

**HTTP Server 的优雅关闭**（`plg_starter_http/index.go:25-35`）：

```go
go func() {
    ensureAppHasBooted(...)
    <-ctx.Done()
    srv.Shutdown(context.Background())  // 等待已有请求完成
}()
```

`http.Server.Shutdown()` 的行为：
- 停止接受新连接
- 等待已有请求处理完毕
- **没有超时限制**（`Shutdown(context.Background())` 传了 `Background` ctx）

但这对 TUS 分块上传有特殊影响：

| 上传类型 | 优雅关闭时的影响 |
|----------|------------------|
| 普通上传 `POST /api/files/save` | 正常完成，因为 `ctx.Backend.Save()` 是同步的 |
| TUS 上传 `POST`（创建会话） | 正常完成，只是创建了 pipe |
| TUS 上传 `PATCH`（传输数据） | 正常完成当前 chunk |
| TUS 上传的 pipe 后台 goroutine | ⚠️ **没有等待机制** |

**关键缺陷**：`chunkedUploadCache` 中的 pipe 后台 goroutine（`go func() { done <- save(path, r) }()`）不在 HTTP 请求的生命周期内。`Shutdown()` 等待的是 HTTP handler 返回，但 pipe 的 `save()` goroutine 可能还在运行。

当进程退出时：
1. HTTP handler 已经返回（`PATCH` 请求已经发了 204 No Content）
2. pipe 后台 goroutine 还在 `save(path, r)`
3. 进程退出 → goroutine 被强制杀死 → **数据可能只写了一半**

#### 3.7.5 缓存过期时的资源回收

`chunkedUploadCache` 的 `OnEvict` 回调（`files.go:687-700`）：

```go
chunkedUploadCache = NewAppCache(60*24, 1)
chunkedUploadCache.OnEvict(func(key string, value interface{}) {
    c := value.(*chunkedUpload)
    if c == nil {
        Log.Warning("ctrl::files::chunked::cleanup nil on close")
        return
    }
    if err := c.Close(); err != nil {
        Log.Warning("ctrl::files::chunked::cleanup action=close err=%s", err.Error())
        return
    }
})
```

`Close()` 的行为（`files.go:721-728`）：

```go
func (this *chunkedUpload) Close() error {
    this.stream.Close()       // 关闭 pipe 写端 → save() 会收到 io.EOF
    err := <-this.done        // 等待 save() 返回
    this.once.Do(func() {
        close(this.done)      // 关闭 channel
    })
    return err
}
```

这个回收机制在**正常过期**（24 小时无操作）时是完整的，但在**进程异常退出**时不生效。

#### 3.7.6 缺失的优雅关闭方案

```go
// 需要在 OnQuit 中主动关闭所有进行中的上传
func init() {
    Hooks.Register.OnQuit(func() {
        // 遍历 chunkedUploadCache 中的所有上传
        // 对每个执行 Close() 并等待完成
        // 设置超时避免无限等待
    })
}
```

但当前代码中 **没有任何插件注册了 `OnQuit` 钩子来处理上传清理**。搜索全代码库，`Hooks.Register.OnQuit` 没有任何调用方。

### 3.8 衔接点 6：同一用户同时对多个后端写入的资源与计数协调

#### 3.8.1 并发写入的架构支持

Filestash 的多后端连接架构天然支持并发写入——每个后端连接有独立的 `IBackend` 实例、独立的 session、独立的 `AuthorisationMiddleware` 链。

用户通过浏览器 Tab 1 写入 SFTP、Tab 2 写入 S3 时：

```
Tab 1: Cookie: auth=<sftp_session>
    → SessionStart 解密 → ctx.Session = {type: "sftp", hostname: "...", ...}
    → ctx.Backend = SFTPBackend{}
    → auth.Save(ctx, path) → processFileAction() → TriggerEvents()
    → ctx.Backend.Save(path, reader)

Tab 2: Cookie: auth1=<s3_session>
    → SessionStart 解密 → ctx.Session = {type: "s3", bucket: "...", ...}
    → ctx.Backend = S3Backend{}
    → auth.Save(ctx, path) → processFileAction() → TriggerEvents()
    → ctx.Backend.Save(path, reader)
```

两个请求完全独立，各自走自己的 Backend，没有共享状态。

#### 3.8.2 上传并发控制

**前端并发控制**：`upload_pool_size`（默认 15）在前端控制同时上传的文件数：

```go
// server/common/config.go:79
FormElement{Name: "upload_pool_size", Type: "number", Default: 15,
    Description: "Maximum number of files upload in parallel. Default: 15"},
```

但这是**全局限制**，不是按后端或按用户的限制。前端上传代码（`ctrl_upload.js`）维护一个本地的并发池，所有后端共享。

**后端没有并发控制**：`ctx.Backend.Save()` 是同步阻塞的，后端插件本身不做并发限制。

#### 3.8.3 chunkedUploadCache 的隔离性

cache key 的构成（`files.go:543-546`）：

```go
cacheKey := map[string]string{
    "path":    path,
    "session": GenerateID(ctx.Session),
}
```

`GenerateID(ctx.Session)` 包含后端类型、hostname 等参数（`crypto.go:193-218`），所以：
- 用户 A 写 SFTP `/data/file.txt` → cacheKey = hash({path: "/data/file.txt", session: "sftp_hash_1"})
- 用户 A 写 S3 `/data/file.txt` → cacheKey = hash({path: "/data/file.txt", session: "s3_hash_2"})

**不同后端的 TUS 上传完全隔离**，不会互相干扰。

#### 3.8.4 资源竞争的潜在问题

| 场景 | 是否有问题 | 说明 |
|------|-----------|------|
| 不同用户写不同后端 | ✅ 无问题 | 完全独立 |
| 同一用户写不同后端 | ✅ 无问题 | session 不同，cache key 不同 |
| 同一用户同一后端写不同路径 | ✅ 无问题 | path 不同，cache key 不同 |
| 同一用户同一后端写同一路径 | ⚠️ 有风险 | 后端并发写入同文件的语义不确定 |

**同一后端同一路径的并发写入**：如果用户在两个 Tab 同时上传同名文件到同一个 SFTP 服务器：

1. 两次 `POST` 会创建两个 `chunkedUpload`，但 **cache key 相同**
2. 第二个 `POST` 会执行 `chunkedUploadCache.Del(cacheKey)`，终止第一个上传
3. 第一个上传的 `OnEvict` → `Close()` → 等待 `save()` 返回
4. 然后创建新的 `chunkedUpload`，但此时 `save()` 可能还在写入旧数据

这就是 3.6.4 节提到的**并发写入**问题，后端行为取决于具体实现。

#### 3.8.5 计数协调的缺失

框架中**没有跨后端的写入计数协调**：

1. **没有全局写入计数器** - 不知道一个用户当前有多少个进行中的上传
2. **没有写入速率限制** - 不限制单位时间内的写入请求
3. **没有跨后端的资源汇总** - 不知道用户在所有后端的总带宽/存储使用量

如果需要实现"用户总并发上传数不超过 N"或"用户总写入带宽不超过 X MB/s"，需要自定义 `Middleware`：

```go
type ConcurrencyLimiter struct {
    mu       sync.Mutex
    counters map[string]int  // userID → 当前并发上传数
}

func (this *ConcurrencyLimiter) Middleware(next HandlerFunc) HandlerFunc {
    return HandlerFunc(func(ctx *App, res http.ResponseWriter, req *http.Request) {
        if req.Method != http.MethodPost && req.Method != http.MethodPatch {
            next(ctx, res, req)
            return
        }
        if _, ok := req.Header["Tus-Resumable"]; !ok {
            next(ctx, res, req)
            return
        }

        userID := getUser(ctx.Session)
        maxConcurrency := 5

        this.mu.Lock()
        if this.counters == nil {
            this.counters = make(map[string]int)
        }
        current := this.counters[userID]
        if current >= maxConcurrency {
            this.mu.Unlock()
            SendErrorResult(res, NewError("Too many concurrent uploads", 429))
            return
        }
        this.counters[userID] = current + 1
        this.mu.Unlock()

        next(ctx, res, req)

        this.mu.Lock()
        this.counters[userID]--
        this.mu.Unlock()
    })
}
```

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
| `server/common/types.go` | `IAuditPlugin`、`IAuthorisation`、`IBackend`、`IAction`、`ITrigger` 接口定义 |
| `server/common/plugin.go` | Hook 注册/获取机制（`Onload`/`OnQuit`/`OnConfig`/`Middleware`/`WorkflowAction`） |
| `server/common/backend.go` | Backend Driver 注册与获取 |
| `server/common/config.go` | 配置定义（上传并发、分块、SMTP 参数等） |
| `server/common/config_state.go` | 配置文件加载/保存，`Onload` 中设置文件权限 |
| `server/common/crypto.go` | `GenerateID()` 实现，按后端连接生成唯一标识 |
| `server/common/cache.go` | `AppCache` 实现（底层 `go-cache`），`NewAppCache`/`OnEvict` |
| `server/common/utils.go` | 通用工具函数 |
| `server/model/audit.go` | 默认 `SimpleAudit` 实现、`AuditForm` 搜索表单定义 |
| `server/model/files.go` | `NewBackend` 连接白名单检查 |
| `server/model/index.go` | `Onload` 初始化 SQLite DB（Share/Location/Verification 表） |
| `server/model/permissions.go` | `CanRead/CanEdit/CanUpload/CanShare` 权限函数 |
| `server/ctrl/files.go` | 文件操作 Controller，TUS 分块上传实现，`chunkedUploadCache` |
| `server/ctrl/admin.go` | `FetchAuditHandler` 审计查询、`FetchLogHandler` 本地日志读取 |
| `server/routes.go` | 路由注册，`/admin/api/audit`、`/admin/api/logs` 端点 |
| `server/middleware/index.go` | `NewMiddlewareChain` 中间件组装，`logger()` 调用点 |
| `server/middleware/session.go` | `SessionStart`，`_extractBackend` 初始化，多 cookie 管理 |
| `server/middleware/telemetry.go` | HTTP 访问日志记录（LogEntry、Telemetry、RequestID 生成、Flush 上报） |
| `server/pkg/workflow/index.go` | Workflow 初始化、worker 池、Job 执行 |
| `server/pkg/workflow/job.go` | `ExecuteJob`，Job 状态流转 |
| `server/pkg/workflow/action.go` | `ExecuteAction` 分发、`findAction` 查找注册的 IAction |
| `server/pkg/workflow/actions/notify_email.go` | `notify/email` Action，SMTP 邮件发送（gomail） |
| `server/pkg/workflow/actions/run_api.go` | `run/api` Action，Webhook/API 调用 |
| `server/pkg/workflow/actions/tools_debug.go` | `tools/debug` Action，日志输出 |
| `server/pkg/workflow/actions/utils.go` | `Render()` 模板变量渲染 |
| `server/pkg/workflow/model/workflow.go` | `FindWorkflows`、`AllWorkflows`、`GetWorkflow`（含 3000 条 Job History） |
| `server/pkg/workflow/model/job.go` | `CreateJob`、`NextJob`、`UpdateJob`，Job 持久化 |
| `server/pkg/workflow/trigger/fileaction.go` | ★ `hookAuthorisation` 审计触发核心，`fileactionCallback` 变量匹配 |
| `server/pkg/workflow/trigger/index.go` | `TriggerEvents` 事件分发 |
| `server/plugin/index.go` | 所有后端/认证插件的导入入口（编译时确定） |
| `server/plugin/plg_starter_http/index.go` | HTTP Server 启动，`Shutdown()` 优雅关闭 |
| `server/plugin/plg_authorisation_example/index.go` | 授权中间件示例 |
| `server/plugin/plg_backend_local/index.go` | Local 后端实现（配额靠 OS） |
| `server/plugin/plg_backend_sftp/index.go` | SFTP 后端（配额错误码映射 14→No space / 15→Quota exceeded） |
| `server/plugin/plg_widget_recent/index.go` | `getUser()` 实现示例，`GenerateID` 使用示例 |
| `server/plugin/plg_widget_favourite/index.go` | `OnConfig` 钩子使用示例（UI 补丁切换） |
| `server/plugin/plg_widget_description/utils.go` | 另一个 `getUser()` 实现示例 |
| `server/plugin/plg_metadata_sqlite/index.go` | `tenantID` 使用示例，按后端连接隔离 |
| `public/assets/pages/adminpage/model_audit.js` | 前端审计查询模型（`ajax GET admin/api/audit`） |
| `public/assets/pages/adminpage/ctrl_activity_audit.js` | 前端审计页面控制器（表单防抖、URL query 组装、HTML 渲染） |
