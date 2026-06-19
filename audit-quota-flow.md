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
  - [2.5 衔接点 1：Save 失败时审计日志如何落地、操作记录与失败结果对齐](#25-衔接点-1save-失败时审计日志如何落地操作记录与失败结果对齐)
- [3. 配额 / 容量限制 完整流程](#3-配额--容量限制-完整流程)
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
| `server/common/types.go` | `IAuditPlugin`、`IAuthorisation`、`IBackend` 接口定义 |
| `server/common/plugin.go` | Hook 注册/获取机制（`Onload`/`OnQuit`/`OnConfig`/`Middleware`） |
| `server/common/backend.go` | Backend Driver 注册与获取 |
| `server/common/config.go` | 配置定义（上传并发、分块等） |
| `server/common/config_state.go` | 配置文件加载/保存，`Onload` 中设置文件权限 |
| `server/common/crypto.go` | `GenerateID()` 实现，按后端连接生成唯一标识 |
| `server/common/cache.go` | `AppCache` 实现（底层 `go-cache`），`NewAppCache`/`OnEvict` |
| `server/common/utils.go` | 通用工具函数 |
| `server/model/audit.go` | 默认 `SimpleAudit` 实现、AuditForm 定义 |
| `server/model/files.go` | `NewBackend` 连接白名单检查 |
| `server/model/index.go` | `Onload` 初始化 SQLite DB（Share/Location/Verification 表） |
| `server/model/permissions.go` | `CanRead/CanEdit/CanUpload/CanShare` 权限函数 |
| `server/ctrl/files.go` | 文件操作 Controller，TUS 分块上传实现，`chunkedUploadCache` |
| `server/ctrl/admin.go` | `FetchAuditHandler` 审计查询 Handler |
| `server/routes.go` | 路由注册，`/admin/api/audit` 端点 |
| `server/middleware/index.go` | `NewMiddlewareChain` 中间件组装，`logger()` 调用点 |
| `server/middleware/session.go` | `SessionStart`，`_extractBackend` 初始化，多 cookie 管理 |
| `server/middleware/telemetry.go` | HTTP 访问日志记录（LogEntry、RequestID 生成） |
| `server/pkg/workflow/index.go` | Workflow 初始化、worker 池、Job 执行 |
| `server/pkg/workflow/job.go` | `ExecuteJob`，Job 状态流转 |
| `server/pkg/workflow/model/job.go` | `CreateJob`、`NextJob`、`UpdateJob`，Job 持久化 |
| `server/pkg/workflow/trigger/fileaction.go` | ★ `hookAuthorisation` 审计触发核心 |
| `server/pkg/workflow/trigger/index.go` | `TriggerEvents` 事件分发 |
| `server/plugin/index.go` | 所有后端/认证插件的导入入口（编译时确定） |
| `server/plugin/plg_starter_http/index.go` | HTTP Server 启动，`Shutdown()` 优雅关闭 |
| `server/plugin/plg_authorisation_example/index.go` | 授权中间件示例 |
| `server/plugin/plg_backend_local/index.go` | Local 后端实现（配额靠 OS） |
| `server/plugin/plg_backend_sftp/index.go` | SFTP 后端（配额错误码映射） |
| `server/plugin/plg_widget_recent/index.go` | `getUser()` 实现示例，`GenerateID` 使用示例 |
| `server/plugin/plg_widget_favourite/index.go` | `OnConfig` 钩子使用示例（UI 补丁切换） |
| `server/plugin/plg_widget_description/utils.go` | 另一个 `getUser()` 实现示例 |
| `server/plugin/plg_metadata_sqlite/index.go` | `tenantID` 使用示例，按后端连接隔离 |
| `public/assets/pages/adminpage/model_audit.js` | 前端审计查询模型 |
| `public/assets/pages/adminpage/ctrl_activity_audit.js` | 前端审计页面控制器 |
