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
| `server/common/crypto.go` | `GenerateID()` 实现，按后端连接生成唯一标识 |
| `server/common/utils.go` | 通用工具函数 |
| `server/model/audit.go` | 默认 `SimpleAudit` 实现、AuditForm 定义 |
| `server/model/files.go` | `NewBackend` 连接白名单检查 |
| `server/model/permissions.go` | `CanRead/CanEdit/CanUpload/CanShare` 权限函数 |
| `server/ctrl/files.go` | 文件操作 Controller，TUS 分块上传实现 |
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
| `server/plugin/index.go` | 所有后端/认证插件的导入入口 |
| `server/plugin/plg_authorisation_example/index.go` | 授权中间件示例 |
| `server/plugin/plg_backend_local/index.go` | Local 后端实现（配额靠 OS） |
| `server/plugin/plg_backend_sftp/index.go` | SFTP 后端（配额错误码映射） |
| `server/plugin/plg_widget_recent/index.go` | `getUser()` 实现示例，`GenerateID` 使用示例 |
| `server/plugin/plg_widget_description/utils.go` | 另一个 `getUser()` 实现示例 |
| `server/plugin/plg_metadata_sqlite/index.go` | `tenantID` 使用示例，按后端连接隔离 |
| `public/assets/pages/adminpage/model_audit.js` | 前端审计查询模型 |
| `public/assets/pages/adminpage/ctrl_activity_audit.js` | 前端审计页面控制器 |
