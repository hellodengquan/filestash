# 上传分片与流控代码链路分析

## 一、整体架构概览

Filestash 的上传系统采用 **前端分片 + 后端流式合并** 的架构，基于 [TUS 协议](https://tus.io/protocols/resumable-upload) 实现断点续传。整体链路横跨前后端，涉及以下核心模块：

```
┌───────────────────────────────────────────────────────────────────────────┐
│                              前端 (Browser)                                │
│  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────────────┐  │
│  │  componentFile  │→→│  workers$ (任务  │→→│  workerImplFile (执行   │  │
│  │  zone / FAB     │   │   队列调度)     │   │   器: 分片+TUS协议)    │  │
│  └─────────────────┘   └─────────────────┘   └────────────┬────────────┘  │
│                                                           │               │
│  ┌─────────────────┐   ┌─────────────────┐                │               │
│  │  componentUpload│←←│  executeHttp (XHR│←←───────────────┘               │
│  │  Queue (UI展示) │   │   进度/速度)    │                                │
│  └─────────────────┘   └─────────────────┘                                │
└───────────────────────────────────────┬───────────────────────────────────┘
                                        │ HTTP (TUS Protocol)
                                        ↓
┌───────────────────────────────────────────────────────────────────────────┐
│                              后端 (Go Server)                              │
│  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────────────┐  │
│  │  FileSave (路由 │→→│ chunkedUploadCache│→→│  chunkedUpload (流合并器)│  │
│  │   分发)         │   │  (AppCache 24h)  │   │  io.Pipe + Backend.Save│  │
│  └─────────────────┘   └─────────────────┘   └─────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────┘
```

### 关键配置项

| 配置项 | 位置 | 默认值 | 说明 |
|--------|------|--------|------|
| `upload_chunk_size` | `server/common/config.go:80` | 0 (MB) | 0=不分片，>0 启用分片 |
| `upload_pool_size` | `server/common/config.go:79` | 15 | 最大并行上传数（后端保留，前端硬编码 MAX_WORKERS=4） |
| `MAX_WORKERS` | `public/.../ctrl_upload.js:100` | 4 | 前端并行工作线程数 |

---

## 二、断点续传逻辑（TUS 协议实现）

断点续传是整个分片上传中最复杂的部分。Filestash 实现了 TUS 协议的核心子集，核心思路是：**用服务端缓存保存上传上下文（offset + stream），前端通过 HEAD 查询恢复进度**。

### 2.1 断点续传的完整生命周期

```
                ┌─────────────────────────────────────────┐
                │         首次上传 / 重试上传              │
                └───────────────┬─────────────────────────┘
                                │
        ┌───────────────────────▼───────────────────────┐
        │ 1. HEAD /api/files/save?path=xxx              │  ← 断点查询
        │    Headers: Tus-Resumable: 1.0.0              │
        └───────────────────────┬───────────────────────┘
                                │
          ┌─────────────────────┴─────────────────────┐
          │ 命中缓存?                                  │
          └──────┬───────────────────────┬────────────┘
                 │ Yes                   │ No
                 ▼                       ▼
    ┌────────────────────────┐  ┌───────────────────────────────┐
    │ 读取 offset / length   │  │ 2. POST /api/files/save       │
    │ 从 cacheKey 获取       │  │    Headers:                    │
    │ uploadURL = apiURL     │  │    Upload-Length: <filesize>   │
    │ offset = X             │  │    Tus-Resumable: 1.0.0       │
    └───────────┬────────────┘  └───────────────┬───────────────┘
                │                                │
                │                       ┌────────▼────────┐
                │                       │ 创建 chunkedUpload│
                │                       │ → io.Pipe()      │
                │                       │ → 启动 goroutine │
                │                       │   调用 Backend.Save│
                │                       │ → 写入 cache     │
                │                       └────────┬────────┘
                │                                │
                └──────────────────┬─────────────┘
                                   ▼
                ┌─────────────────────────────────────────┐
                │ 3. 循环 PATCH 分片                       │
                │    for i in [ceil(offset/chunkSize), N)  │
                │    PATCH <uploadURL>                     │
                │      Upload-Offset: current_offset       │
                │      Content-Type: application/offset+...│
                │      Body: file.slice(offset, offset+sz) │
                └───────────────────┬─────────────────────┘
                                    │
                    ┌───────────────▼───────────────┐
                    │ 每片完成后 offset += chunkSize │
                    │ 更新 chunkedUpload.offset      │
                    └───────────────┬───────────────┘
                                    │
              ┌─────────────────────▼─────────────────────┐
              │ offset == totalSize ?                     │
              └──────┬────────────────────────┬───────────┘
                     │ Yes                    │ No (中断→下次继续)
                     ▼                        ▼
        ┌────────────────────┐      ┌────────────────────┐
        │ Close() pipeWriter │      │ 保留 cache (24h)   │
        │ 等待 goroutine     │      │ 下次 HEAD 恢复     │
        │ Backend.Save 返回  │      └────────────────────┘
        │ → Done, 删除 cache │
        └────────────────────┘
```

### 2.2 前端断点恢复实现

**代码位置**: `public/assets/pages/filespage/ctrl_upload.js:363-466` (`prepareJob` 方法)

#### 关键步骤解析

**Step 1: 断点查询（HEAD 请求）** - `ctrl_upload.js:397-412`
```javascript
try {
    const resp = await executeHttp.call(this, apiURL, {
        method: "HEAD",
        headers: { ...tusHeaders },
        body: null,
        progress: () => {},
        speed,
    });
    // 关键校验: 确保是同一个文件（大小匹配）
    if (file.size === parseInt(resp.headers["upload-length"])) {
        const tmp = parseInt(resp.headers["upload-offset"]);
        if (tmp > 0) {
            offset = tmp;              // 恢复断点位置
            uploadURL = apiURL;         // 复用原 URL
        }
    }
} catch (err) {}  // 失败静默降级为全新上传
```

**Step 2: 新建上传（POST 请求）** - `ctrl_upload.js:413-432`

当 `offset === 0`（首次上传或断点恢复失败）时，创建新的上传会话：

```javascript
const resp = await executeHttp.call(this, apiURL, {
    method: "POST",
    headers: {
        ...tusHeaders,
        "Upload-Length": file.size,  // 声明文件总大小
    },
    body: null,
    ...
});
uploadURL = resp.headers.location;   // 服务端返回的上传会话 URL
```

**Step 3: 分片循环上传（PATCH 请求）** - `ctrl_upload.js:434-459`

**断点续传的精髓在此行**:
```javascript
// 从断点位置开始，而不是从 0 开始！
for (let i = Math.ceil(offset / chunkSize); i < numberOfChunks; i++) {
```

分片进度映射到全局进度的算法：
```javascript
progress: (p) => {
    const start = Math.ceil(100 * offset / file.size);           // 该片起始百分比
    const end = Math.ceil(100 * Math.min(file.size, offset + chunkSize) / file.size);
    progress(Math.floor(start + (end - start) * p / 100));       // 线性插值到全局
}
```

### 2.3 后端断点状态管理

**代码位置**: `server/ctrl/files.go:477-734`

#### 缓存键设计 - `files.go:543-546`
```go
cacheKey := map[string]string{
    "path":    path,       // 文件目标路径
    "session": GenerateID(ctx.Session),  // 会话ID（区分不同用户）
}
```

#### AppCache 配置 - `files.go:687-689`
```go
func initChunkedUploader() {
    chunkedUploadCache = NewAppCache(60*24, 1)  // 保留 24 小时，1 分钟清理
    ...
}
```
> **关键保障**: 断点状态最多保留 **24 小时**，超时自动清理并关闭管道（触发合并失败逻辑）。

#### HEAD 查询断点 - `files.go:554-567`
```go
if proto == "tus" && req.Method == http.MethodHead {
    c := chunkedUploadCache.Get(cacheKey)
    if c == nil {
        SendErrorResult(res, ErrNotFound)  // 404 → 前端降级为全新上传
        return
    }
    offset, length := c.(*chunkedUpload).Meta()
    h.Set("Upload-Offset", fmt.Sprintf("%d", offset))  // 返回断点位置
    h.Set("Upload-Length", fmt.Sprintf("%d", length))
    res.WriteHeader(http.StatusNoContent)
    return
}
```

#### Offset 一致性校验 - `files.go:630-635`
```go
initialOffset, totalSize := uploader.Meta()
if initialOffset != requestOffset {
    // 前端声明的 offset 与服务端记录不一致 → 拒绝
    SendErrorResult(res, ErrNotValid)
    return
}
```

---

## 三、进度反馈机制

进度反馈是一个 **三层级联更新** 系统：从 XHR 底层事件 → 单文件进度 → 全局队列聚合。

### 3.1 整体数据流向

```
XMLHttpRequest.upload.onprogress
        │ (每触发一次)
        ▼
executeHttp() 内部计算
  ├─ percent = loaded / total → 调用 progress(percent)
  └─ avgSpeed (滑动窗口5s)  → 调用 speed(bytesPerSec)
        │
        ├──────────────────────────────────────────┐
        │                                          │
        ▼                                          ▼
workerImplFile 的回调                       workerImplFile 的回调
  (分片时做线性插值修正)                        (直接透传)
        │                                          │
        ▼                                          ▼
updateDOMTaskProgress()                    updateDOMTaskSpeed()
  → 单文件百分比显示                          → 单文件速度显示
        │                                          │
        │                                          ▼
        │                                  updateDOMGlobalSpeed()
        │                                    → MAX_WORKERS 速度累加
        │                                    → 节流: 500ms 更新一次
        ▼
updateDOMWithStatus("doing"/"done"/"error")
  → 文件行状态色切换
  → 按钮（停止/重试）替换
```

### 3.2 底层：XHR 进度与速度采集

**代码位置**: `public/assets/pages/filespage/ctrl_upload.js:527-585`

```javascript
xhr.upload.onprogress = (e) => {
    if (!e.lengthComputable) return;
    const percent = Math.floor(100 * e.loaded / e.total);
    progress(percent);

    // ===== 速度计算: 滑动窗口平均算法 =====
    prevProgress.push(e);  // 保留最近的进度事件
    if (prevProgress.length === 1) return;  // 单点无法计算速度

    let avgSpeed = 0;
    for (let i = 1; i < prevProgress.length; i++) {
        const p1 = prevProgress[i];
        const pm1 = prevProgress[i-1];
        avgSpeed += (p1.loaded - pm1.loaded) / ((p1.timeStamp - pm1.timeStamp) / 1000);
    }
    avgSpeed = avgSpeed / (prevProgress.length - 1);  // 算术平均
    speed(avgSpeed);

    // 窗口老化: 超过 5 秒的样本丢弃
    if (e.timeStamp - prevProgress[0].timeStamp > 5000) {
        prevProgress.shift();
    }
};
```

**设计考量**:
- 不使用瞬时速度（波动大），而是 **多点窗口平均**
- 窗口边界：**5 秒滑动窗口**，兼顾平滑性与灵敏度

### 3.3 中层：分片上传的进度插值

**代码位置**: `public/assets/pages/filespage/ctrl_upload.js:451-455`

当使用分片上传时，XHR 的 `loaded/total` 只针对当前分片，需要映射到文件全局进度：

```javascript
const start = Math.ceil(100 * offset / file.size);           // 当前分片起始百分比
const end = Math.ceil(100 * Math.min(file.size, offset + chunkSize) / file.size);
progress(Math.floor(start + (end - start) * p / 100));       // 线性插值
```

**示例**: 100MB 文件，chunkSize=20MB，上传第 3 片 (40-60MB)
- `start = 40%`, `end = 60%`
- 若当前片传了 50% (`p=50`)，则全局进度 = `40 + (60-40)*50/100 = 50%`

### 3.4 上层：全局 UI 聚合

**代码位置**: `public/assets/pages/filespage/ctrl_upload.js:183-193`

全局速度聚合（4 个 worker 的速度累加）：
```javascript
const updateDOMGlobalSpeed = (function(workersSpeed) {
    let last = 0;
    return (nworker, currentWorkerSpeed) => {
        workersSpeed[nworker] = currentWorkerSpeed;  // 更新对应槽位
        if (new Date().getTime() - last <= 500) return;  // 500ms 节流
        last = new Date().getTime();
        const speed = workersSpeed.reduce((acc, el) => acc + el, 0);  // 求和
        // 渲染到 UI
        ...
    };
}(new Array(MAX_WORKERS).fill(0)));
```

**任务状态机**: `updateDOMWithStatus()` - `ctrl_upload.js:195-257`

| 状态 | UI 表现 | 触发时机 |
|------|---------|----------|
| `todo` | 灰色等待 | 入队时 |
| `doing` | 蓝色进度条 + 停止按钮 | worker 取出任务时 |
| `done` | 绿色完成 + 清除按钮 | `exec.run()` 成功时 |
| `error` | 红色错误 + 重试按钮 | 异常捕获时 |

---

## 四、合并失败处理

"合并" 在本系统中不是单独的步骤，而是指 **所有分片写入 io.Pipe 后，Backend.Save 将流式数据持久化** 的过程。失败可能发生在多个层面。

### 4.1 数据流式合并模型

**代码位置**: `server/ctrl/files.go:672-685`

```go
func createChunkedUploader(save func(path string, file io.Reader) error, path string, size uint64) *chunkedUpload {
    r, w := io.Pipe()          // 创建管道
    done := make(chan error, 1)
    go func() {
        done <- save(path, r)  // 启动 goroutine: 从管道读 → 写入后端
    }()
    return &chunkedUpload{
        fn:     save,
        stream: w,             // PATCH 请求写入此端
        done:   done,          // Backend.Save 结果信号
        offset: 0,
        size:   size,
    };
}
```

**数据流示意**:
```
PATCH #1 Body ──┐
PATCH #2 Body ──┼──► io.PipeWriter ───────► io.PipeReader ──────► Backend.Save(path, reader)
PATCH #3 Body ──┘    (Next() 写入)          (goroutine 消费)         (最终持久化)
                                                          │
                                                          └──► err → done channel
```

### 4.2 失败场景分类与处理链路

#### 场景 1: 单分片传输失败（网络问题）

**发生位置**: `ctrl_upload.js:447-465` (前端 PATCH 请求抛出异常)

```javascript
try {
    for (let i = Math.ceil(offset/chunkSize); i < numberOfChunks; i++) {
        ...
        await executeHttp.call(this, uploadURL, { ... });  // ← 这里抛异常
        offset += chunkSize;
    }
    virtual.afterSuccess();
} catch (err) {
    virtual.afterError();           // ← UI 层清理
    if (err === ABORT_ERROR) return; // 用户主动取消，不再重试
    throw err;                       // ← 冒泡给上层 → UI 显示"重试"按钮
}
```

**处理结果**:
- 服务端 `chunkedUploadCache` **保留**（24h 内有效）
- 服务端 `offset` 停留在最后一次成功写入的值
- 前端 UI 进入 `error` 状态，用户点击 **重试** 时：
  1. 重新执行 `prepareJob()`
  2. 先 HEAD 查询恢复 offset
  3. 从断点处继续上传（参见 2.2 断点恢复）

---

#### 场景 2: Backend.Save 流式合并失败（最终写入失败）

**触发时机**: 所有分片都成功写入 Pipe，但 Backend.Save 在消费管道时出错（如后端存储不可用、权限不足、磁盘满等）

**代码位置**: `server/ctrl/files.go:656-663`

```go
} else if newOffset == totalSize {
    if err := uploader.Close(); err != nil {  // ← Close() 会阻塞等待 done channel
        Log.Debug("files::save::tus action=uploader.close err=%s", err.Error())
        SendErrorResult(res, ErrNotValid)  // ← 返回 400 给前端
        return
    }
    chunkedUploadCache.Del(cacheKey)
}
```

**Close 内部实现**: `files.go:721-728`
```go
func (this *chunkedUpload) Close() error {
    this.stream.Close()   // 先关闭写端 → 让 Backend.Save 的 Read 遇到 EOF
    err := <-this.done    // 阻塞等待 Backend.Save 的 goroutine 返回结果
    this.once.Do(func() { close(this.done) })
    return err            // ← Backend.Save 的错误原样返回
}
```

**前端表现**: `ctrl_upload.js:447-465` 中最后一片的 `executeHttp` 会收到 HTTP 400，进入 catch 分支，与场景 1 表现相同。

> **⚠️ 但有本质区别**: Close() 失败后 `chunkedUploadCache.Del()` **没有被调用**吗？看代码：
> ```go
> if err := uploader.Close(); err != nil {
>     SendErrorResult(res, ErrNotValid)
>     return  // ← 直接 return，cache 没删！
> }
> chunkedUploadCache.Del(cacheKey)  // 只有 Close 成功才删
> ```
> 
> 实际上 **cache 不会被立即删除**，但 Close() 已关闭了 stream。下次重试时 HEAD 会命中缓存但 offset 等于总大小，进入边界情况。这是一个 **潜在的状态不一致问题**。

---

#### 场景 3: 缓存过期被自动清理

**触发时机**: 用户中断上传超过 24 小时

**代码位置**: `files.go:687-699`

```go
chunkedUploadCache.OnEvict(func(key string, value interface{}) {
    c := value.(*chunkedUpload)
    if c == nil { return }
    if err := c.Close(); err != nil {  // ← 被驱逐时强制 Close
        Log.Warning("ctrl::files::chunked::cleanup action=close err=%s", err.Error())
        return
    }
})
```

**后果**:
- `stream.Close()` 关闭管道 → Backend.Save 的管道读取返回 `io.EOF` 或 `io.ErrClosedPipe`
- Backend.Save 收到 **不完整数据流**，通常会报错（取决于具体后端实现）
- 缓存被删除，下次上传 **无法断点续传**，只能从头开始

---

#### 场景 4: 用户主动取消上传

**前端链路**: `ctrl_upload.js:209-213`
```javascript
$stop.onclick = () => {
    cancel();                              // 调用 exec.cancel()
    $task.removeAttribute("data-status");
    ...
};
// cancel 实现 (ctrl_upload.js:345-348)
cancel() {
    if (this.xhr) assert.type(this.xhr, XMLHttpRequest).abort();
    this.xhr = null;
}
```

```javascript
// executeHttp 中
xhr.upload.onabort = () => reject(ABORT_ERROR);

// prepareJob 的 catch
if (err === ABORT_ERROR) return;  // ← 不 throw，不触发重试 UI
```

**后端状态**: 
- 当前正在处理的 PATCH 请求被中断，`io.Copy` 返回错误
- 但之前已成功写入的分片数据 **保留在管道缓冲区中**（由 offset 记录）
- 如果服务端 chunkedUpload 还在（24h 内），**刷新页面后仍可断点续传**

---

#### 场景 5: Offset 断言溢出（安全边界）

**代码位置**: `files.go:650-655`

```go
newOffset, _ := uploader.Meta()
if newOffset > totalSize {
    uploader.Close()
    chunkedUploadCache.Del(cacheKey)
    Log.Warning("files::save::tus path=%s err=assert_offset ...", ...)
    SendErrorResult(res, NewError("aborted - offset larger than total size", 403))
    return
}
```

**触发条件**: 恶意客户端构造请求导致 offset 越界，或并发写入竞争
**处理策略**: 强制关闭 + 删除缓存 + 记录警告 + 拒绝请求（安全优先）

---

### 4.3 重试机制（用户侧）

**代码位置**: `ctrl_upload.js:240-249`

```javascript
$retry.onclick = async() => {
    executeMutation("todo");     // UI 重置为等待
    executeMutation("doing");    // UI 切换为进行中
    try {
        await exec.retry();      // 调用 Worker 的 retry()
        executeMutation("done"); // 成功
    } catch (err) {
        executeMutation("error"); // 失败，继续显示重试
    }
};
```

`retry()` 的实现（`ctrl_upload.js:356-359`）：
```javascript
this.retry = () => {
    virtual.before();    // 重新添加虚拟文件到列表
    return executeJob(); // 完整重新执行 prepareJob → 含断点查询
};
```

---

## 五、流控与并发调度

### 5.1 前端工作池调度

**代码位置**: `ctrl_upload.js:259-325`

```javascript
const MAX_WORKERS = 4;                        // 最多 4 个并行上传
let tasks = [];                                // 全局任务池
const reservations = new Array(MAX_WORKERS).fill(false);  // 槽位占用标记

// 新任务到达时
workers$.subscribe(async({ tasks: newTasks, loading = false }) => {
    tasks = tasks.concat(newTasks);            // 追加到任务池
    while (!$page.classList.contains("hidden")) {
        const nworker = reservations.indexOf(false);  // 找空闲槽位
        if (nworker === -1) break;             // 无空闲 → 等待
        reservations[nworker] = true;           // 标记占用
        // 启动 worker（永不失败循环）
        noFailureAllowed(processWorkerQueue.bind(null, nworker))
            .then(() => reservations[nworker] = false);
    }
});
```

**调度循环**: `processWorkerQueue(nworker)` - `ctrl_upload.js:261-304`
1. `nextTask(tasks)` → 找第一个 `ready()` 为 true 的任务
2. 检查 DOM 中无同路径的运行中任务（防重复）
3. 通过 `exec = task.exec(...)` 创建执行器实例
4. `await exec.run(task)` → 阻塞直到完成/失败
5. 循环直到任务池为空

**依赖调度（目录上传）**: `ready()` 函数 - `ctrl_upload.js:698-710`
- 拖拽上传目录时使用 BFS 遍历，子文件/子目录的 `ready()` 会阻塞直到 **父目录创建完成**
- 确保 `/a/b/c.txt` 不会在 `/a/b/` 创建之前就开始上传

### 5.2 后端并发控制

后端 **无显式的上传并发限制**，依赖：
1. 前端 `MAX_WORKERS=4` 限制了单个客户端的并发
2. Go HTTP Server 的连接并发上限
3. 具体后端存储的客户端连接池配置（如 S3 SDK 的 maxConns）

---

## 七、多 Backend 分片合并的差异化路径

所有 backend 都通过统一的 `IBackend.Save(path string, file io.Reader) error` 接口接入，但内部实现差异巨大，导致分片合并（即流式消费 io.Reader）的行为完全不同。

### 7.1 IBackend 接口与 Backend 注册机制

**统一接口定义** (`server/common/types.go:13-24**:
```go
type IBackend interface {
    ...
    Save(path string, file io.Reader) error  // 所有 backend 必须实现
    ...
}
```

**注册机制** (`server/common/backend.go:21-26`:
```go
func (d *Driver) Register(name string, driver IBackend) {
    d.ds[name] = driver
}
// 各 backend 在各自 `init()` 中调用，例如：
// plg_backend_s3: Backend.Register("s3", S3Backend{})
// plg_backend_ftp: Backend.Register("ftp", Ftp{})
// plg_backend_local: Backend.Register("local", &Local{...})
```

**TUS 创建时绑定 backend 来源** (`server/ctrl/files.go:578-585`
```go
ctx.Context = context.Background()
b, err := ctx.Backend.Init(ctx.Session, ctx)  // 用当前会话重建 backend 实例
if err != nil { ... }
uploader := createChunkedUploader(b.Save, path, size)  // ← 绑定具体 backend 的 Save 方法
```

### 7.2 Local Backend：直接文件写入磁盘

**代码位置**: `server/plugin/plg_backend_local/index.go:124-134
```go
func (this Local) Save(path string, content io.Reader) error {
    f, err := SafeOsOpenFile(path, os.O_WRONLY|os.O_CREATE|os.O_TRUNC, 0664)
    if err != nil {
        return err
    }
    if _, err = io.Copy(f, content); err != nil {  // ← 直接从 io.PipeReader → 本地文件
        f.Close()
        return err
    }
    return f.Close()
}
```

| 特性 | 说明 |
|--------|------|
| **流式特性 | 纯流式 `io.Copy`，零额外缓冲 |
| **内存占用 | 仅内核 pipe buffer（默认 64KB） |
| **断点影响 | pipe 自带背压：写端写入快于磁盘消费时，PATCH 阻塞 |
| **失败行为 | io.Copy 出错立即返回，文件已写部分落盘但不清理（需手动删除） |

### 7.3 S3 Backend：SDK 内置再分片（Multipart Upload）

**代码位置**: `server/plugin/plg_backend_s3/index.go:569-587`
```go
func (this S3Backend) Save(path string, file io.Reader) error {
    p := this.path(path)
    if p.bucket == "" { return ErrNotValid }
    uploader := s3manager.NewUploader(this.createSession(p.bucket))
    input := s3manager.UploadInput{
        Body:        file,   // ← io.PipeReader
        Bucket:      aws.String(p.bucket),
        Key:         aws.String(p.path),
        ContentType: aws.String(GetMimeType(path)),
        ...
    }
    _, err := uploader.UploadWithContext(this.app.Context, &input)
    return err
}
```

**关键差异**：AWS SDK 的 `s3manager.Uploader` 内部使用 **Multipart Upload** 机制：

| 特性 | 说明 |
|--------|------|
| **SDK 分片策略** | Uploader 内部默认 5MB 分块（`DefaultUploadConcurrency=5）并发上传 |
| **内存占用** | 需要至少缓冲多个分片缓冲） 至少缓冲 5×5MB=25MB） |
| **两级分片 | 前端分片 → Pipe → S3 SDK 再 前端 chunk 再分片） → S3 Multipart |
| **断点粒度 |
| **失败行为 | 任一分片失败触发 abort multipart upload，不完整文件不会可见） |
| **完成时机 | SDK 内部调用 CompleteMultipartUpload 才真正存在 |

> **⚠️ 注意两层分片**：前端 chunk_size=20MB，SDK 内部再拆成 4 个 5MB 的 multipart part 上传到 S3。

### 7.4 FTP Backend：FTP 协议流模式）

**代码位置**: `server/plugin/plg_backend_ftp/index.go:363-367
```go
func (f Ftp) Save(path string, file io.Reader) (err error) {
    return f.Execute(func(client *goftp.Client) error {
        return client.Store(path, file)  // goftp 的 Store 方法
    })
}
```

**Execute 包装器** (`plg_backend_ftp/index.go:373-399
```go
func (f Ftp) Execute(fn func(*goftp.Client) error) error {
    err := fn(f.client)
    ftpErr, ok := err.(goftp.Error)
    if !ok { return err }
    code := ftpErr.Code()
    if code == 421 || (code == 0 && err.Error() == "EOF reading response: EOF" {
        // 连接断开 → 自动重连重试
        f.Close()
        b, initErr := f.Init(f.p, &App{Context: f.ctx})
        return b.(*Ftp).Execute(fn)
    }
    ...
}
```

| 特性 | 说明 |
|--------|------|
| **传输模式 | FTP `STOR` 命令，数据流 |
| **连接池 | `ConnectionsPerHost` 可配置（默认 5） |
| **失败重试 | 421/EOF 自动重连后重新执行 Execute(fn) **但会丢失已读流数据 |
| **致命问题 | ⚠️ FTP 的 `client.Store` 从 `io.Reader` 消费后，连接断开 → 重连后再次消费同一个 Reader，此时 pipe 已读位置已前进，重传的数据永久丢失 |

### 7.5 三种 Backend 对比表

| 维度 | Local | S3 | FTP |
|------|-------|-----|-----|
| **Save 调用方 | 内核 write() → 内核 pipe buffer 64KB) | AWS SDK Uploader (5MB 缓冲池 | goftp STOR 数据流 |
| **内存占用 | 低（≤ 高（多分片并发） | 中（流 |
| **再分片** | 无 | 有（SDK 内部再拆成 5MB） | 无 |
| **失败原子性** | 写入一半可见） | 原子（Complete 前不可见） | 部分写入（服务端临时文件） |
| **断点续传 | 依赖服务端 offset） | 依赖 TUS cache | 依赖 TUS cache |
| **断连重传 | pipe 阻塞 | 连接断开由 SDK 管 | 自动重连但有丢数据风险 |

---

## 八、断点续传状态的存储位置与权限校验挂载点

### 8.1 断点状态存储：纯内存，不落盘

**结论：断点续传状态 100% 存储在 Go 进程内存中，不写入磁盘。

#### 存储结构**：`server/ctrl/files.go:477

```go
var chunkedUploadCache AppCache  // 包级全局变量
```

**初始化** `server/ctrl/files.go:687-700
```go
func initChunkedUploader() {
    chunkedUploadCache = NewAppCache(60*24, 1)  // retention=24h, cleanup=1min
    chunkedUploadCache.OnEvict(func(key string, value interface{}) {
        c := value.(*chunkedUpload)
        c.Close()  // 被驱逐时关闭 pipe
    })
}
```

**存储介质**：`server/common/cache.go:51-63
```go
func NewAppCache(arg ...time.Duration) AppCache {
    c := AppCache{}
    c.Cache = cache.New(retention*time.Minute, cleanup*time.Minute)
    // ← 基于 "github.com/patrickmn/go-cache" → 纯内存 KV，纯内存
    return c
}
```

#### chunkedUpload 结构体**：`server/ctrl/files.go:702-710
```go
type chunkedUpload struct {
    fn     func(path string, file io.Reader) error  // backend.Save
    stream *io.PipeWriter      // ← 写端
    offset uint64              // ← 已写字节数（断点位置）
    size   uint64              // ← 文件总大小
    done   chan error         // ← Backend.Save 返回值信号
    once   sync.Once
    mu     sync.Mutex        // ← offset 读写锁
}
```

#### 与磁盘的关系：
```
                   进程内存中
┌─────────────────────────────────────┐
│  chunkedUploadCache            │
│  ┌────────────────────────┐  │
│  │ map[hash(key)]          │  │
│  │   - offset =1234567     │  │
│  │   - *io.PipeWriter    │  │
│  │   - done channel       │  │
│  │   - size        │  │
│  └────────────────────────┘  │
└──────────────┬──────────────────────┘
           │ io.Pipe()
           │
           ▼
    ┌────────────────┐
    │ io.PipeReader │────► Backend.Save(Reader
    └────────────────┘
           │
           ▼
  各 Backend 各自写入各自的目标存储
  (Local 磁盘 / S3 / FTP 服务器)
```

**关键注意事项**：
1. **进程重启 = 所有断点状态全部丢失**。进程重启后 cache 清空，用户必须重新上传
2. **`TMP_PATH`（`data/cache/`）用于 range request 缓存下载分片上传
3. **多实例部署**：用户请求哈希到同一台实例，断点续传只能在同一实例恢复

### 8.2 权限校验：三层校验挂载点

上传请求经过三层权限拦截，全部 **跨所有 backend 生效：

#### Layer 1: Controller 层硬编码权限（基础能力级权限

**代码位置**：`server/ctrl/files.go:492-522

```go
func FileSave(ctx *App, res http.ResponseWriter, req *http.Request) {
    // ① 权限层1：编辑权限
    if model.CanEdit(ctx) == false {
        if model.CanUpload(ctx) == false {
            SendErrorResult(res, ErrPermissionDenied)
            return
        }
        // 无编辑权限但有上传权限 → 禁止覆盖
        root, filename := SplitPath(path)
        entries, err := ctx.Backend.Ls(root)
        for _, e := range entries {
            if e.Name() == filename {
                SendErrorResult(res, ErrConflict)  // 文件已存在 → 409
                return
            }
        }
    }
    ...
}
```

#### Layer 2: Authorisation Middleware（插件可插拔）

**挂载接口**：`server/common/types.go:32-41`
```go
type IAuthorisation interface {
    ...
    Save(ctx *App, path string) error
    ...
}
```

**调用点**：`server/ctrl/files.go:516-522
```go
// 所有已注册的 Auth 插件
for _, auth := range Hooks.Get.AuthorisationMiddleware() {
    if err = auth.Save(ctx, path); err != nil {
        SendErrorResult(res, ErrNotAuthorized)
        return
    }
}
```

**注册机制**：`server/common/plugin.go:141-149
```go
var authorisation_middleware []IAuthorisation

func (this Register) AuthorisationMiddleware(a IAuthorisation) {
    authorisation_middleware = append(authorisation_middleware, a)
}
```

**已注册的插件示例**（grep）：
- `plg_authorisation_example
- `plg_search_sqlitefts` - 搜索爬虫文件
- ...

#### Layer 3: Backend 自声明 Meta（能力级 ACL）

**接口**：`server/common/types.go` - 各 backend 可选实现
```go
// 非 IBackend 接口本身不是强制
// 但 backend 可以声明
```

**示例**：
- **S3**：`plg_backend_s3/index.go:182-192`
  - 根路径 `/` → `CanUpload: false`
- **FTP**：`plg_backend_ftp/index.go:239-251
  - 匿名用户 `acl == "r"` → `CanUpload: false`

**调用链**：`server/ctrl/files.go:492`
`
通过 `model.CanUpload(ctx)` 在 FileLs 中调用，FileSave 中直接调用）

#### 权限校验完整调用链路图
```
前端发起请求
    │
    ▼
FileSave() handler
    │
    ├──► Layer 1: model.CanEdit()/CanUpload()
    │         └─ ctx.Share.CanUpload (分享链接场景)
    │
    ├──► Layer 2: for _, auth := range Hooks.Get.AuthorisationMiddleware()
    │         auth.Save(ctx, path)
    │         └─► plg_authorisation_*
    │
    ├──► TUS HEAD/POST/PATCH
    │
    └──► Backend.Save() 的 Save() 的 Meta 本身
              └─► backend 具体 backend 的 FTP 后端本身的 ACL/S3 bucket policy...
```

#### TUS 各 HTTP Method 的权限覆盖情况

| TUS Method | Layer 1 CanEdit/Upload) | Layer 2 AuthMiddleware.Save) | Backend 实际执行 |
|-------------|-----------------------|------------------------|-----------------|
| **OPTIONS** | ❌ 不校验 | ❌ 不校验 | - |
| **HEAD** (断点查询) | ❌ 不校验 | ❌ 不校验 | 读 cache |
| **POST** (创建会话) | ✅ 校验 | ✅ 校验 | - |
| **PATCH** (传分片) | ❌ 不校验 | ❌ 不校验 | 写 pipe 写入 |

> **⚠️ 安全隐患**：HEAD 和 PATCH 不经过 Layer 1/Layer 2 权限校验。攻击者只要拿到有效的 session 就可以 PATCH 继续写入（cacheKey 包含 path + session，攻击者可写。

---

## 九、合并失败时残留分片的 GC 路径

Filestash 的「残留分片」分两个层面：**进程内部的 TUS 缓存清理**和**后端存储介质的残留数据清理**。两者的 GC 路径完全不同，前者完善，后者因 backend 而异。

### 9.1 GC 触发点全景图

```
                  合并失败触发源
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
   用户主动取消    进程异常崩溃      24h 超时驱逐
          │              │              │
          ▼              ▼              ▼
   xhr.abort()       进程内存清零    chunkedUploadCache
   → XHR abort                         .OnEvict 回调
          │                              │
          ▼                              ▼
  PATCH io.Copy 返回 err           c.Close() → stream.Close()
          │                              │
          └──────┬───────────────────────┘
                 ▼
         TUS 内存状态清理（AppCache 删除）
                 │
                 ▼
         Backend.Save 的 Reader 收到 EOF/ErrPipe
                 │
       ┌─────────┴──────────┬────────────┬───────────────┐
       ▼                    ▼            ▼               ▼
     Local                S3         FTP/SFTP      Backblaze/Azure
  (文件残留磁盘)     (Multipart残留)  (部分STORE)    (Block残留)
       │                    │            │               │
       ▼                    ▼            ▼               ▼
   无自动清理        SDK内失败自动Abort  服务端策略      SDK 清理
   需用户手动        但进程崩溃残留      依赖服务端      依赖策略
```

### 9.2 层 1：TUS 缓存的 GC（进程内）

**触发点 1：正常 Close 流程（成功/显式失败）** - `server/ctrl/files.go:656-663`
```go
} else if newOffset == totalSize {
    if err := uploader.Close(); err != nil {
        SendErrorResult(res, ErrNotValid)
        return
    }
    chunkedUploadCache.Del(cacheKey)  // ← 成功：主动从缓存删除
}
```

**触发点 2：Offset 越界（安全保护）** - `server/ctrl/files.go:650-655`
```go
if newOffset > totalSize {
    uploader.Close()                       // ← 关闭管道
    chunkedUploadCache.Del(cacheKey)       // ← 立即从缓存删除
    SendErrorResult(res, NewError("aborted - offset larger than total size", 403))
    return
}
```

**触发点 3：OnEvict 驱逐回调（24h 超时/程序清理）** - `server/ctrl/files.go:689-699`
```go
chunkedUploadCache.OnEvict(func(key string, value interface{}) {
    c := value.(*chunkedUpload)
    if c == nil { return }
    if err := c.Close(); err != nil {        // ← 被驱逐时强制 Close
        Log.Warning("ctrl::files::chunked::cleanup action=close err=%s", err.Error())
        return
    }
})
```

**触发点 4：服务启动时清理 TMP_PATH** - `server/common/constants.go:54-55`
```go
os.RemoveAll(GetAbsolutePath(TMP_PATH))    // ← 清理下载缓存
os.MkdirAll(GetAbsolutePath(TMP_PATH), os.ModePerm)
```
> 注意：`TMP_PATH` 只存 **下载 range** 缓存（`file_cache`），**不存上传分片**。上传分片 100% 在内存 pipe 中。

### 9.3 层 2：各 Backend 存储残留的 GC 路径

#### 9.3.1 Local Backend：半写入文件永久残留

**残留场景**：
1. 上传 1GB 文件，传到 600MB 时用户取消 / 进程崩溃 / 网络中断
2. `SafeOsOpenFile` 已经创建了目标文件（`O_CREATE|O_TRUNC`）
3. `io.Copy(f, content)` 写了 600MB，返回错误，`f.Close()` 执行

**代码路径** (`server/plugin/plg_backend_local/index.go:124-134`)：
```go
func (this Local) Save(path string, content io.Reader) error {
    f, err := SafeOsOpenFile(path, os.O_WRONLY|os.O_CREATE|os.O_TRUNC, 0664)
    // ↑ 注意：O_TRUNC 会先截断已有同名文件！
    if err != nil { return err }
    if _, err = io.Copy(f, content); err != nil {
        f.Close()           // ← 关闭已写 600MB 的文件
        return err
    }
    return f.Close()
}
```

**GC 状态**：
- ❌ **无自动清理**，600MB 截断文件永久停留在目标路径
- ⚠️ 更糟：`O_TRUNC` 在 Save 入口就清空了原文件，失败后原文件已丢失
- 用户视角：目标目录出现一个「大小不符」的同名文件，下次重试上传会再次 `O_TRUNC` 覆盖它

#### 9.3.2 S3 Backend：SDK 内部两级保护 + 进程崩溃的盲区

**第一级：s3manager.Uploader 的正常失败路径**

AWS SDK 的 `Uploader` 内部使用 Multipart Upload，失败时会自动 Abort：

```
Uploader.UploadWithContext(ctx, &input)
    │
    ├──► CreateMultipartUpload  ← 创建 UploadId
    │
    ├──► 并发 UploadPart (默认 5×5MB 缓冲)
    │       ├── Part #1 OK
    │       ├── Part #2 OK
    │       └── Part #3 失败
    │            │
    │            └──► AbortMultipartUpload  ← SDK 内部自动调用
    │                 （清理 S3 端已上传的 Part）
    │
    └──► 错误返回给调用方
```

**第二级：进程崩溃 / SIGKILL 的盲区**

如果在 `CreateMultipartUpload` 成功、`CompleteMultipartUpload` 成功之前进程崩溃：
- SDK 内部的 `defer` 来不及执行
- **S3 端残留 Incomplete Multipart Upload**（所有已上传 Part 永久存在）
- ❌ **Filestash 无任何清理逻辑**

**运维侧补救方案**：需要在 S3 Bucket 上配置 Lifecycle Rule：
```json
{
  "Rule": {
    "Filter": { "Prefix": "" },
    "Status": "Enabled",
    "AbortIncompleteMultipartUpload": {
      "DaysAfterInitiation": 7
    }
  }
}
```

#### 9.3.3 FTP Backend：部分 STOR 文件 + 重连丢数据风险

**残留场景 1：STOR 中途断开**
```
client.Store(path, reader)
    ├──► USER/PASS → PASV → STOR path
    ├──► Data Socket Open
    ├──► 写入 600MB 数据
    └──► 连接断开（421 / 网络中断）
         │
         └──► FTP 服务端行为：
              • 大多数服务器（vsftpd/proftpd）：保留已写 600MB 的临时文件
              • 下次同路径 STOR 重新开始（覆盖）
```

**残留场景 2：Execute 自动重连的副作用** - `plg_backend_ftp/index.go:373-390`
```go
code := ftpErr.Code()
if code == 421 || (code == 0 && err.Error() == "error reading response: EOF") {
    f.Close()
    FtpCache.Set(f.p, nil)
    b, initErr := f.Init(f.p, &App{Context: f.ctx})
    return b.(*Ftp).Execute(fn)   // ← 重新执行同一个 Store 函数
}
```

**⚠️ 致命问题**：
- 第一次 `client.Store(path, reader)` 已经消费了 reader 的前 600MB
- 重连后再次调用 `client.Store(path, reader)`，reader 的 Read 位置无法回退
- 结果：只上传了剩余 400MB → **文件缺失开头 600MB，永久损坏**
- GC：损坏文件残留，无自动清理

#### 9.3.4 Backblaze B2：整体上传模式，无中间残留

**代码位置**：`server/plugin/plg_backend_backblaze/index.go:401-459`
```go
func (this Backblaze) Save(path string, file io.Reader) error {
    // Step 1: b2_get_upload_url 获取上传 URL
    // Step 2: 从 io.Reader 读 → HTTP POST 到 uploadUrl
    res, err := this.requestWithSHA1(
        "POST", resBody.UploadUrl, file,
        map[string]string{
            "Authorization":       resBody.Token,
            "X-Bz-File-Name":      url.PathEscape(p.B2Path),
            "Content-Type":        GetMimeType(p.B2Path),
            "X-Bz-Content-Sha1":   sha,  // ← 整体 SHA1
        }, totalSize,
    )
}
```

**残留特性**：
- 一次性流式 POST，后端在 SHA1 校验通过前 **不暴露文件**
- 中途断开：B2 服务端丢弃已接收数据，**无残留**
- 进程崩溃：同上，无任何残留
- ✅ GC 最干净

#### 9.3.5 Azure Blob：UploadStream 的 Block Blob 自动管理

**代码位置**：`server/plugin/plg_backend_azure/index.go:331-338`
```go
func (this *AzureBlob) Save(path string, file io.Reader) error {
    _, err := this.client.UploadStream(
        this.ctx, ap.containerName, ap.blobName, file, nil,
    )
    return err
}
```

**残留特性**：
- Azure SDK `UploadStream` 内部使用 Block Blob 分块（StageBlock + CommitBlockList）
- 失败时 SDK 自动清理未提交的 Block
- 进程崩溃：未提交 Block 在 7 天后被 Azure 自动 GC（服务端策略）
- ✅ 后端自带 GC 兜底

### 9.4 各 Backend 残留 GC 对比表

| Backend | 残留类型 | 正常失败清理 | 进程崩溃残留 | GC 兜底机制 |
|---------|---------|-------------|-------------|------------|
| **Local** | 目标文件磁盘残留 | ❌ 保留已写部分 | ❌ 永久保留 | ❌ 无，需用户手动 |
| **S3** | Incomplete Multipart | ✅ SDK 自动 Abort | ⚠️ 永久保留 Part | ⚠️ 需配置 Bucket Lifecycle |
| **FTP** | 目标文件残留 / 损坏 | ⚠️ 保留部分内容 | ❌ 永久保留 | ❌ 依赖 FTP 服务端策略 |
| **Backblaze** | 无残留 | ✅ 整体校验 | ✅ 无残留 | ✅ 服务端丢弃不完整数据 |
| **Azure** | 未提交 Block | ✅ SDK 清理 | ✅ 7 天后 Azure 清 | ✅ Azure 服务端策略 |

---

## 十、多 Backend 并发上传的限流策略挂载点

Filestash 的并发限流是 **多层分散式** 的，没有统一中央调度器，而是前端、HTTP 层、各 backend SDK 各自为政。

### 10.1 限流层次全景图

```
           用户上传 N 个文件
                  │
       ┌──────────┴──────────┐
       ▼                     ▼
【前端限流】MAX_WORKERS=4   多标签页/多用户并发
  (per-browser 槽位)           │
       │                    │
       └──────┬─────────────┘
              ▼
       【HTTP层限流】RateLimiter (全局令牌桶: 10/s, 突发1000)
              │
              ▼
       FileSave 路由 (无并发上限)
              │
    ┌─────────┼──────────┬───────────────┐
    ▼         ▼          ▼               ▼
【S3并发】  【FTP并发】 【Local并发】   【其他 Backend】
 threadSize  ConnectionsPerHost   无限制      各自 SDK 限制
 (50 默认)    (默认5)
    │           │              │
    ▼           ▼              ▼
 S3 SDK      FTP 连接池      OS write()
 Uploader    (goftp)        (系统限制)
 PartSize=5MB
 Concurrency=5
```

### 10.2 Layer 1：前端上传池（per-browser）

**代码位置**：`public/assets/pages/filespage/ctrl_upload.js:99-100,259-325`
```javascript
const MAX_WORKERS = 4;  // ← 硬编码，不读取后端 upload_pool_size 配置
const reservations = new Array(MAX_WORKERS).fill(false);

// 调度循环
while (tasks.length > 0) {
    const nworker = reservations.indexOf(false);
    if (nworker === -1) break;     // 4 个槽位全满 → 等待
    reservations[nworker] = true;
    processWorkerQueue(nworker);   // 启动 worker
}
```

**特性**：
- 单浏览器实例最多 **4 个文件同时上传**（不分大小）
- `MAX_WORKERS` 是硬编码，**不读取** 后端的 `upload_pool_size=15` 配置
- 多标签页之间 **不共享** 槽位（每个标签页独立 4 个）

### 10.3 Layer 2：HTTP API RateLimiter（全局）

**代码位置**：`server/middleware/http.go:107-121`
```go
var limiter = rate.NewLimiter(10, 1000)  // ← 令牌桶：10/s 速率, 1000 突发容量

func RateLimiter(fn HandlerFunc) HandlerFunc {
    return HandlerFunc(func(ctx *App, res http.ResponseWriter, req *http.Request) {
        if limiter.Allow() == false {
            Log.Warning("middleware::http::ratelimit too many requests")
            SendErrorResult(res, NewError(..., http.StatusTooManyRequests))
            return
        }
        fn(ctx, res, req)
    })
}
```

**挂载位置（routes.go）**：
| 路由 | 是否挂载 RateLimiter |
|------|---------------------|
| `/api/session` (POST 登录) | ✅ |
| `/admin/api/session` (POST) | ✅ |
| **`/api/files/save`** (上传) | ❌ **未挂载** |
| `/api/files/cat` (下载) | ❌ |
| `/api/files/*` (其他) | ❌ |

> **关键发现**：上传 API `/api/files/save` 和 `/api/files/cat` **没有** 挂 RateLimiter！令牌桶只保护登录接口。上传路径无全局 QPS 限制。

### 10.4 Layer 3：S3 Backend 内部并发控制

**两个维度的并发控制**：

#### 维度 A：`threadSize` 参数（Rm/Mv 批量操作）

**代码位置**：`server/plugin/plg_backend_s3/index.go:87-97,379-399`
```go
threadSize, err := strconv.Atoi(params["number_thread"])
if err != nil {
    threadSize = 50                    // ← 默认 50 个 goroutine
} else if threadSize > 5000 || threadSize < 1 {
    threadSize = 2
}

// 删除操作中的应用
jobChan := make(chan S3Path, this.threadSize)
errChan := make(chan error, this.threadSize)
for i := 1; i <= this.threadSize; i++ {   // ← 启动 threadSize 个 worker
    wg.Add(1)
    go func() {
        for spath := range jobChan {
            client.DeleteObjectWithContext(ctx, ...)
        }
    }()
}
```

- 只作用于 `Rm()`（递归删除）和 `Mv()`（目录移动）
- **不影响 Save()**

#### 维度 B：AWS SDK Uploader 的分片并发

**代码位置**：S3 SDK `s3manager.Uploader` 默认参数
```go
uploader := s3manager.NewUploader(session)
// Uploader 内部默认值：
//   PartSize:       5 * 1024 * 1024    // 5MB 每片
//   Concurrency:    5                   // 5 个 goroutine 并发传 part
```

**整体并发模型**：
```
1 个用户文件上传（S3 + 分片）
    │
    ├──► 前端 MAX_WORKERS 允许（4 个槽位之一）
    │
    ├──► HTTP 层：无 RateLimiter 限制
    │
    └──► s3manager.Uploader
         ├── 从 io.PipeReader 读取
         ├── 缓冲 5 × 5MB = 25MB 内存
         └── 5 个并发 S3 PutPart 请求
```

**4 文件并行的总并发**：4 × 5 = 20 个并发 S3 PutPart 请求，内存占用至少 4 × 25MB = 100MB

### 10.5 Layer 4：FTP Backend 连接池

**代码位置**：`server/plugin/plg_backend_ftp/index.go:80-85,98-103`
```go
conn := 5   // 默认 5 个连接
if params["conn"] != "" {
    if i, err := strconv.Atoi(params["conn"]); err == nil && i > 0 {
        conn = i
    }
}

cfg := goftp.Config{
    ConnectionsPerHost: conn,    // ← goftp SDK 连接池大小
    Timeout:            timeout,
    ...
}
client, err := goftp.DialConfig(cfg, hostname)
```

**特性**：
- 同一 FTP backend（同用户+同主机）共享 `FtpCache`，所有 Save/Ls 操作竞争 `conn` 个连接
- 示例：4 个文件同时上传 + 2 个 Ls 查询 → 竞争 5 个连接，形成自然限流

### 10.6 Layer 5：`upload_pool_size` 配置（名存实亡）

**配置定义**：`server/common/config.go:79,298,323`
```go
// 管理后台可配置
FormElement{Name: "upload_pool_size", Type: "number", Default: 15, ...}

// 反序列化
UploadPoolSize int `json:"upload_pool_size"`

// 读取
UploadPoolSize: this.Get("general.upload_pool_size").Int(),
```

**实际使用**：全局 grep `UploadPoolSize` / `upload_pool_size`
- 配置项被完整保存到 JSON、导出到前端 `/api/config`
- ❌ **后端代码中没有任何地方实际使用 `Config.UploadPoolSize` 做并发控制**
- ❌ **前端 `MAX_WORKERS` 也不读取此配置（硬编码 4）**
- 结论：这是一个 **预留但未实现** 的配置项

### 10.7 各 Backend 并发限流挂载点总结表

| 限流层级 | 挂载点代码位置 | 控制对象 | 默认值 | 是否实际生效 |
|---------|--------------|---------|--------|------------|
| 前端上传池 | `ctrl_upload.js:99` | per-browser 文件数 | 4 | ✅ 生效 |
| HTTP 令牌桶 | `middleware/http.go:107` | 全局 API QPS | 10/s | ⚠️ 只保护登录，不保护上传 |
| S3 threadSize | `plg_backend_s3/index.go:89` | Rm/Mv 操作并发 | 50 | ✅ 仅删除/移动 |
| S3 Uploader Concurrency | SDK 默认 | Save 的 Part 并发 | 5 | ✅ 内置生效 |
| FTP ConnectionsPerHost | `plg_backend_ftp/index.go:81` | FTP 连接池 | 5 | ✅ 生效 |
| Local I/O | 无代码 | 系统 write() | 无限制 | ❌ 直接穿透 |
| upload_pool_size | `config.go:79` | 配置项 | 15 | ❌ 未接入实际代码 |

---

## 十一、多 Backend 故障切换时半完成分片归属判定

Filestash 所谓「故障切换」在实际代码中表现为：**HTTP 请求跨多次传输之间 backend 参数变更**（用户重新登录到不同账号、后端配置变更、分享链接 session 变更等）。归属判定完全依赖 cacheKey 的 hash 生成规则，不依赖 backend_id。

### 11.1 归属判定的核心：cacheKey 生成规则

**代码位置**: `server/ctrl/files.go:543-546`, `server/common/crypto.go:193-218`

```go
// FileSave 中的 cacheKey (files.go:543-546)
cacheKey := map[string]string{
    "path":    path,
    "session": GenerateID(ctx.Session),   // ← 归属判定的核心
}
```

**GenerateID 的 hash 算法**（`crypto.go:193-218`）：
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
        case "password":  // ← 故意排除
        case "path":      // ← 故意排除
        case "session":   // ← 故意排除
        case "timestamp": // ← 故意排除（登录时间戳）
        default:
            if val := params[key]; val != "" {
                p += key + "=>" + params[key] + ", "
            }
        }
    }
    p += "salt=>" + SECRET_KEY  // ← 绑定服务端密钥
    return Hash(p, 20)          // ← 20 位 SHA1 截断
}
```

### 11.2 Session map 中的字段来源

Session 是 `map[string]string`，存储于加密 Cookie 中。字段因 backend 类型而异：

| Backend | Session 中典型字段 |
|---------|------------------|
| **S3** | `type`, `access_key_id`, `secret_access_key`, `endpoint`, `region`, `path` |
| **FTP** | `type`, `hostname`, `port`, `username`, `path`, `tls` |
| **Local** | `type`, `path` |
| **分享链接** | `type`, `hostname`, `username`, `path`（来自 `Share.Auth` 解密） |

### 11.3 归属判定矩阵

GenerateID 故意排除了 `password/path/session/timestamp` 四个字段，因此判定规则如下：

| 场景 | 可变参数 | hash 是否相同 | cacheKey 是否命中 | 断点续传效果 |
|------|---------|-------------|------------------|------------|
| **同一用户同路径续传** | 无 | ✅ 相同 | ✅ 命中 | ✅ 正常断点恢复 |
| **密码变更**（同账号） | `password` | ✅ 相同 | ✅ 命中 | ✅ 断点恢复（password 被排除） |
| **登录时间不同**（Cookie 刷新） | `timestamp` | ✅ 相同 | ✅ 命中 | ✅ 断点恢复（timestamp 被排除） |
| **切不同 S3 Region 的 bucket** | `region` | ❌ 不同 | ❌ 未命中 | ❌ 无法续传 |
| **切不同 S3 Access Key**（同 bucket） | `access_key_id` | ❌ 不同 | ❌ 未命中 | ❌ 无法续传 |
| **S3 endpoint 变更** | `endpoint` | ❌ 不同 | ❌ 未命中 | ❌ 无法续传 |
| **FTP 切不同用户**（同主机） | `username` | ❌ 不同 | ❌ 未命中 | ❌ 无法续传 |
| **FTP 切不同主机**（同用户） | `hostname` | ❌ 不同 | ❌ 未命中 | ❌ 无法续传 |
| **目标路径不同** | `path` | ✅ 相同（path 被排除） | ❌ 未命中 | ❌ cacheKey.path 不同 |
| **切不同类型 backend**（S3→FTP） | `type` | ❌ 不同 | ❌ 未命中 | ❌ 无法续传 |
| **同分享链接不同用户打开** | - | ✅ 相同 | ✅ 命中 | ✅ 断点恢复 |

### 11.4 完整归属判定链路

```
每个请求到达 FileSave
        │
        ▼
SessionStart 中间件 (session.go:57-82)
  ├──► _extractSession(req, ctx)
  │       ├──► 分享链接场景: ctx.Share.Auth 解密 → session map
  │       └──► 普通登录场景: Authorization Cookie 解密 → session map
  │               ├──► DecryptString(SECRET_KEY_DERIVATE_FOR_USER, auth)
  │               └──► json.Unmarshal → map[type,hostname,username,...]
  │
  └──► _extractBackend(req, ctx)
          └──► model.NewBackend(ctx, session)   (model/files.go:9-51)
                  ├──► isAllowed() ← 匹配 Config.Conn 白名单
                  │       ├──► 检查 type
                  │       ├──► 检查 hostname/path
                  │       └──► 检查 url
                  └──► Backend.Get(session["type"]).Init(session, ctx)
        │
        ▼
FileSave 内部 (files.go:479+)
        │
        ├──► cacheKey.path = PathBuilder(ctx, query.path)  ← 目标路径规范化
        │
        ├──► cacheKey.session = GenerateID(ctx.Session)
        │       ├──► 排序 session 所有 key
        │       ├──► 排除: password / path / session / timestamp
        │       ├──► 拼接: key1=>val1, key2=>val2, ...
        │       ├──► 追加: salt=>SECRET_KEY
        │       └──► SHA1 → 20 位 hash
        │
        ├──► HEAD:  chunkedUploadCache.Get(cacheKey) → 返回 offset
        ├──► POST:  chunkedUploadCache.Set(cacheKey, uploader)
        └──► PATCH: chunkedUploadCache.Get(cacheKey) → Next(reader)
```

### 11.5 故障切换的边界场景分析

**场景 A：用户重新登录到同一后端（密码变更）**
- GenerateID：排除了 `password` → hash 不变
- cacheKey：`path` + 相同 session hash → 命中缓存
- **后果**：断点续传可用，上传继续 ✓

**场景 B：管理员修改了 SECRET_KEY**
```go
// crypto.go:216 → hash 输入变了
p += "salt=>" + SECRET_KEY  // 新密钥
```
- GenerateID：所有旧 session hash 失效
- cacheKey：**所有断点缓存无法命中**
- **后果**：所有进行中的上传必须从头开始

**场景 C：同路径不同账号同时上传**
```
用户A上传 /a.txt  → cacheKey = { "/a.txt", hash(sessionA) }
用户B上传 /a.txt  → cacheKey = { "/a.txt", hash(sessionB) }
```
- hash 不同 → 两个独立的 chunkedUpload 实例
- **后果**：互不干扰，各自 offset 独立管理 ✓

**场景 D：用户上传中 session 超时（>1 年）**
```go
// session.go:310-312
if t.Add(24 * 365 * time.Hour).Before(time.Now()) {
    return session, ErrNotAuthorized  // session 校验失败
}
```
- SessionStart 中间件返回 ErrNotAuthorized → **FileSave 根本不执行**
- 但旧缓存 chunkedUploadCache 中仍存在（直到 24h 超时）
- **后果**：缓存泄漏直到被 OnEvict 清理

### 11.6 归属判定的安全边界

**❌ 未校验 backend 对象一致性**：
```go
// files.go:578-585 (POST 创建时)
ctx.Context = context.Background()
b, err := ctx.Backend.Init(ctx.Session, ctx)  // ← 用当前 session 新建 backend
uploader := createChunkedUploader(b.Save, path, size)

// files.go:623-628 (PATCH 写入时)
c := chunkedUploadCache.Get(cacheKey)  // ← 只看 cacheKey
uploader := c.(*chunkedUpload)         // ← 直接取 uploader.fn
uploader.Next(reader)                  // ← 写入到创建时的 backend
```

**安全隐患**：创建 uploader 时的 backend（`b.Save`）被闭包捕获。如果攻击者在 POST→PATCH 之间通过某种方式篡改 session（获取合法新 session 且 hash 碰撞），不会影响已创建的 uploader，仍然写入到原始 backend。但 cacheKey 相同意味着 session 内容等价，实际风险有限。

---

## 十二、上传失败后客户端续传 retry 的完整代码挂载点

Retry 链路是一个 **前后端深度耦合的状态机**，跨越 UI 层、Worker 调度层、TUS 协议层、Virtual Layer 层。

### 12.1 Retry 完整链路全景图

```
  用户点击 $retry 按钮
        │ (UI层)
        ▼
  updateDOMWithStatus($task, "error")
        │ ctrl_upload.js:240-249
        ▼
  exec.retry()   ← Worker.retry() 被覆盖为动态方法
        │
        │ ┌──────────────────────────────────────────┐
        │ │ run() 时动态绑定 (ctrl_upload.js:353-360) │
        │ │  this.retry = () => {                     │
        │ │     virtual.before();                     │
        │ │     return executeJob();  ← prepareJob    │
        │ │  }                                        │
        │ └──────────────────────────────────────────┘
        ▼
  virtual.before()    ← model_virtual_layer.js:149-154
        │
        │ stateAdd(virtualFiles$, basepath, { loading: true })
        │ statePop(mutationFiles$, basepath, filename)
        ▼
  prepareJob(file, path, virtual)  ← ctrl_upload.js:363-466
        │
        ├──► [Step 1] 分片策略判定
        │       chunkSize === 0 → 不分片，直接 POST
        │       numberOfChunks === 1 → 不分片，直接 POST
        │
        ├──► [Step 2] HEAD 断点查询
        │       executeHttp(HEAD /api/files/save?path=xxx)
        │       │ (Tus-Resumable: 1.0.0)
        │       │
        │       ├──► 命中 + upload-length === file.size
        │       │       offset = resp.headers["upload-offset"]
        │       │       uploadURL = apiURL  (复用)
        │       │
        │       └──► 404 / 大小不匹配
        │               offset = 0, uploadURL = ""
        │
        ├──► [Step 3] offset === 0 ? → POST 新建上传会话
        │       executeHttp(POST /api/files/save?path=xxx)
        │       │ (Upload-Length: file.size)
        │       │
        │       ├──► 成功: uploadURL = resp.headers.location
        │       └──► 失败: virtual.afterError() + throw err
        │
        └──► [Step 4] 循环 PATCH 分片上传
                for (i = Math.ceil(offset/chunkSize); i < numberOfChunks; i++)
                    executeHttp(PATCH uploadURL, { Upload-Offset: offset, Body: chunk })
                    │
                    ├──► 成功: offset += chunkSize
                    │
                    └──► 失败:
                          ├──► err === ABORT_ERROR → return (静默)
                          └──► 其他 err → virtual.afterError() + throw err
        │
        ▼ (所有分片成功后)
  virtual.afterSuccess()  ← model_virtual_layer.js:160-168
        │
        ├──► removeLoading(virtualFiles$, basepath, filename)
        ├──► fscache().update(basepath, ...)  ← 更新目录缓存
        └──► hooks.mutation.emit({ op: "save", path })
        │
        ▼
  updateDOMWithStatus($task, "done")  ← ctrl_upload.js:289-295
```

### 12.2 UI 层挂载点：$retry 按钮的事件绑定

**代码位置**：`public/assets/pages/filespage/ctrl_upload.js:227-251`

```javascript
case "error":
    const $retry = assert.type($iconRetry.cloneNode(true), HTMLElement);
    updateDOMGlobalTitle($page, t("Error"));
    updateDOMTaskProgress($task, t("Error"));
    updateDOMGlobalSpeed(nworker, 0);
    updateDOMTaskSpeed($task, 0);

    $task.removeAttribute("data-path");
    $task.removeAttribute("data-status");
    $task.classList.remove("todo_color");
    $task.classList.add("error_color");
    $task.firstElementChild.nextElementSibling.nextElementSibling.firstElementChild.remove();
    $task.firstElementChild.nextElementSibling.nextElementSibling.appendChild($retry);

    $retry.onclick = async () => {
        executeMutation("todo");     // UI: 标题 "Running..."
        executeMutation("doing");    // UI: 进度条 0%, 替换为 $stop
        try {
            await exec.retry();      // ← 核心调用：Worker 的 retry 方法
            executeMutation("done"); // UI: 显示 Done
        } catch (err) {
            executeMutation("error"); // UI: 继续显示 retry 按钮
        }
    };
```

**关键细节**：
- 只有 `processWorkerQueue` 的 `try/catch` 捕获到异常才会进入 `"error"` 状态（`ctrl_upload.js:293-294`）
- ABORT_ERROR（用户点击 $stop）被 `prepareJob` 的 catch 静默吞掉，**不会**进入 error 状态

### 12.3 Worker 层挂载点：run() 中动态覆盖 retry()

**代码位置**：`public/assets/pages/filespage/ctrl_upload.js:335-360`

```javascript
function workerImplFile({ progress, speed }) {
    return new class Worker extends IExecutor {
        constructor() {
            super();
            this.xhr = null;
        }

        cancel() { this.xhr?.abort(); }

        async run({ file, path, virtual }) {
            const _file = await file();  // ← 解析 File 对象引用（只执行一次）
            const executeJob = () => this.prepareJob({ file: _file, path, virtual });

            // ★ 关键：run() 执行时才覆盖 retry()
            // 捕获 _file、path、virtual 的闭包
            this.retry = () => {
                virtual.before();    // ← 重新添加 virtualFiles$ 标记
                return executeJob(); // ← 复用同一个 _file 对象
            };

            return executeJob();     // ← 首次执行
        }
    };
}
```

**为什么这样设计**：
1. `_file` 在首次 run() 时通过 `await file()` 计算一次，之后 retry 直接复用（避免重新读取拖拽列表）
2. `virtual` 对象也在闭包中捕获，retry 时同一个 SaveVL 实例继续工作
3. IExecutor 的基类 `retry() { throw "NOT_IMPLEMENTED" }` 是抽象方法，run() 执行前调用 retry 会崩溃

### 12.4 Virtual Layer 挂载点：before/afterSuccess/afterError

**代码位置**：`public/assets/pages/filespage/model_virtual_layer.js:145-177`

```javascript
export function save(path, size) {
    const [basepath, filename] = extractPath(path);
    const file = { name: filename, type: "file", size, time: new Date().getTime() };

    return new class SaveVL extends IVirtualLayer {
        before() {  // ← retry 时调用: 重新加入 UI
            stateAdd(virtualFiles$, basepath, { ...file, loading: true });
            statePop(mutationFiles$, basepath, filename);
        }

        async afterSuccess() {  // ← 成功时: 移除 loading, 刷新目录
            if (basepath === currentPath()) removeLoading(virtualFiles$, basepath, filename);
            await fscache().update(basepath, ({ files = [], ...rest }) => ({
                files: files.concat([file]), ...rest,
            }));
            hooks.mutation.emit({ op: "save", path: basepath });
        }

        async afterError() {  // ← 失败时: 从虚拟列表移除
            statePop(virtualFiles$, basepath, filename);
            return rxjs.EMPTY;
        }
    }();
}
```

**调用时机表**：

| 状态 | before() | afterSuccess() | afterError() |
|------|----------|---------------|--------------|
| **首次上传** | workers$ 入队前 | 所有 PATCH 成功后 | 任意 step 失败 |
| **retry 开始** | $retry.onclick 中 | retry 成功后 | retry 失败后 |
| **用户 cancel** | - | - | -（ABORT_ERROR 静默吞掉） |

### 12.5 TUS 协议层断点恢复挂载点

**前端 HEAD 查询**：`ctrl_upload.js:397-412`
```javascript
try {
    const resp = await executeHttp.call(this, apiURL, {
        method: "HEAD", headers: { ...tusHeaders },
    });
    // ★ 安全性校验：必须文件大小匹配
    if (file.size === parseInt(resp.headers["upload-length"])) {
        const tmp = parseInt(resp.headers["upload-offset"]);
        if (tmp > 0) {
            offset = tmp;          // ← 恢复断点
            uploadURL = apiURL;    // ← 复用 URL（POST 没调）
        }
    }
} catch (err) {}  // ← 空 catch：HEAD 失败静默降级为 offset=0 从头传
```

**后端 HEAD 处理**：`files.go:554-567`
```go
if proto == "tus" && req.Method == http.MethodHead {
    c := chunkedUploadCache.Get(cacheKey)
    if c == nil {
        SendErrorResult(res, ErrNotFound)  // 404 → 前端降级
        return
    }
    offset, length := c.(*chunkedUpload).Meta()
    h.Set("Upload-Offset", fmt.Sprintf("%d", offset))  // ← 返回断点
    h.Set("Upload-Length", fmt.Sprintf("%d", length))  // ← 返回总大小
    res.WriteHeader(http.StatusNoContent)
    return
}
```

**后端 POST 创建时的清理挂载**：`files.go:568-571`
```go
if proto == "tus" && req.Method == http.MethodPost {
    // ★ 关键：POST 新建前先删旧缓存
    if c := chunkedUploadCache.Get(cacheKey); c != nil {
        chunkedUploadCache.Del(cacheKey)
        // ↑ 旧缓存（及其中的 offset/stream）被丢弃，从头开始
    }
    ...
}
```
> 触发时机：HEAD 返回 404 或大小不匹配时，前端走 offset === 0 → POST。如果旧缓存还在（但大小不匹配），POST 会先清理。

### 12.6 各失败场景下的 Retry 行为

| 失败场景 | 前端 catch 路径 | virtual.afterError() | retry 时 HEAD 结果 | 是否真正断点续传 |
|---------|---------------|---------------------|------------------|--------------|
| **网络中断（单分片 PATCH）** | prepareJob → HTTP 错误 → catch throw | ✅ 调用 | ✅ 命中缓存，返回 offset | ✅ 从断点续传 |
| **Close() 失败（Backend.Save 合并错误）** | 最后一片 PATCH → 400 → throw | ✅ 调用 | ✅ 命中缓存（Close 失败但 cache 未删） | ⚠️ 返回 offset = totalSize，边界问题 |
| **24h 超时缓存被清理** | - | - | ❌ 404 → offset=0 | ❌ 从头传 |
| **SECRET_KEY 变更** | - | - | ❌ 404（cacheKey.session 变了） | ❌ 从头传 |
| **用户切换 backend 账号** | - | - | ❌ 404（cacheKey.session 变了） | ❌ 从头传 |
| **用户点击 $stop（abort）** | ABORT_ERROR → 静默 return | ❌ 不调用 | ✅ 命中缓存（如未超时） | ⚠️ 刷新页面后才可用，本次 UI 已移除 |
| **后端进程崩溃重启** | - | - | ❌ 404（内存清零） | ❌ 从头传 |

---

## 十三、多用户并发上传同一文件的去重策略代码挂载点

Filestash 的去重策略是 **分层防御 + 各管一段** 的组合，没有统一的全局去重调度器。每个层级（前端单实例、后端单用户、后端多用户）分别负责一部分范围。

### 13.1 去重层次全景图

```
 用户拖拽 N 个文件
        │
        ▼
┌─────────────────────────────────────────────────────┐
│ Layer 1: 前端 DOM 检查去重（per-browser）              │
│ ctrl_upload.js:272-278                                │
│ querySelectorAll([data-path=xxx][data-status=running])│
│ → 存在 → 等待 1s 放回队列头部                           │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│ Layer 2: 前端目录依赖树去重（BFS ready()）             │
│ ctrl_upload.js:694-710                                │
│ 父目录任务 done === false → ready() return false       │
│ → 保证 /a/b/c.txt 不在 /a/b/ 创建完成前上传              │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│ Layer 3: 后端权限层存在性检查（CanUpload 场景）         │
│ files.go:492-513                                      │
│ CanEdit=false && CanUpload=true →                     │
│   Ls(root) + 遍历 entries 检查同名                     │
│   存在 → ErrConflict(409)                              │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│ Layer 4: 后端 cacheKey 天然用户隔离（TUS POST）         │
│ files.go:543-546, 568-571                             │
│ cacheKey = { path, GenerateID(session) }              │
│ → 不同用户 session hash 不同                          │
│ → 同一用户二次 POST：先 Del 旧缓存再新建               │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│ Layer 5: 后端 offset 并发安全（单 TUS 实例内）          │
│ files.go:709, 715-717, 731-733                       │
│ chunkedUpload.mu sync.Mutex                           │
│ → 保护 offset 读写一致性                              │
│ → 不阻止并发 PATCH（需要客户端配合去重）               │
└─────────────────────────────────────────────────────┘
```

### 13.2 Layer 1：前端 DOM 检查去重

**代码位置**：`public/assets/pages/filespage/ctrl_upload.js:272-278`

```javascript
// processWorkerQueue 的 step2
const $tasks = qsa($page, `[data-path="${task.path}"][data-status="running"]`);
if ($tasks.length > 0) {
    await new Promise((resolve) => setTimeout(resolve, 1000));  // 等待 1s
    tasks.unshift(task);  // 放回队列头部（先进先出，下次继续尝试）
    continue;
}
```

**覆盖范围**：
- ✅ 同一浏览器同一标签页
- ✅ 防止 user 快速双击「上传」按钮导致重复任务
- ❌ 多标签页之间不共享 DOM（各自独立的 workers$ 队列）
- ❌ 多浏览器 / 多用户无效

**data-status 状态设置时机**（`ctrl_upload.js:207, 235`）：
```javascript
// 开始执行时
$task.setAttribute("data-status", "running");

// 成功/失败时清除
$task.removeAttribute("data-status");  // done/error 时
```

### 13.3 Layer 2：前端目录依赖树去重

**代码位置**：`public/assets/pages/filespage/ctrl_upload.js:694-710`

```javascript
ready = () => {
    for (let i=0; i<tasks.length; i++) {
        // 只检查当前任务之前的目录任务（BFS 顺序保证）
        if (tasks[i].path === task.path) break;
        else if (tasks[i].type === "file") continue;
        else if (isInDirectory(tasks[i].path, task.path) === false) continue;

        // 父目录的创建任务未完成 → 阻塞
        if (tasks[i].done === false) return false;
    }
    return true;
};
```

**覆盖范围**：
- ✅ 同一拖拽批次内的目录/文件依赖
- ✅ 防止「目录 /a/b/ 还没创建，文件 /a/b/c.txt 已经开始上传 → 404」
- ❌ 跨批次上传（先拖一批目录，再拖一批文件）

### 13.4 Layer 3：后端权限层存在性检查（409 Conflict）

**代码位置**：`server/ctrl/files.go:492-513`

```go
if model.CanEdit(ctx) == false {
    if model.CanUpload(ctx) == false {
        SendErrorResult(res, ErrPermissionDenied)
        return
    }
    // CanUpload 用户：禁止覆盖已有文件
    root, filename := SplitPath(path)
    entries, err := ctx.Backend.Ls(root)
    if err != nil {
        SendErrorResult(res, ErrPermissionDenied)
        return
    }
    for i := 0; i < len(entries); i++ {
        if entries[i].Name() == filename {
            Log.Debug("files::save action=permission_ls err=already_exist")
            SendErrorResult(res, ErrConflict)  // ← HTTP 409
            return
        }
    }
}
```

**⚠️ 关键限制**：
- 只对 **CanEdit=false, CanUpload=true** 的用户生效（典型场景：只允许上传、不允许修改的分享链接用户）
- 对管理员 / 所有者（CanEdit=true）**不做去重**，直接覆盖
- 存在 TOCTOU（Time-of-check to time-of-use）竞态：两个请求同时通过 Ls 检查 → 都进入 Save → 后者覆盖前者

### 13.5 Layer 4：后端 cacheKey 天然用户隔离 + POST 清理

**cacheKey 设计**（`files.go:543-546`）：
```go
cacheKey := map[string]string{
    "path":    path,
    "session": GenerateID(ctx.Session),  // ← 不同用户必然不同
}
```

**并发 POST 竞态**（`files.go:568-571`）：
```go
if proto == "tus" && req.Method == http.MethodPost {
    // ★ 关键：新建前先删旧缓存
    if c := chunkedUploadCache.Get(cacheKey); c != nil {
        chunkedUploadCache.Del(cacheKey)
        // ↑ 同一用户第二次 POST → 第一次的缓存/offset/pipe 全丢
    }
    ...
}
```

**并发场景分析**：

| 场景 | cacheKey 相同？ | 去重效果 |
|------|---------------|---------|
| **用户A + 用户B 传同一路径** | ❌ session 不同 | ❌ 两个独立 TUS 实例 → 最终 Save 谁后完成谁覆盖 |
| **同用户多标签页传同一路径** | ✅ session 相同 | ⚠️ 第二次 POST Del 第一次的缓存 → 第一次的 PATCH 收到 409 |
| **同用户同标签页双击上传** | ✅ session 相同 | ✅ Layer 1 DOM 检查先拦截 |

**PATCH 命中失败（缓存被第二次 POST 删了）**（`files.go:623-628`）：
```go
c := chunkedUploadCache.Get(cacheKey)
if c == nil {
    Log.Debug("files::save::tus action=backend_save step=cache_fetch_patch")
    SendErrorResult(res, NewError("Conflict", 409))  // ← 返回给前端
    return
}
```
前端收到 409 → `executeHttp` 的 `onload` 中 `xhr.status=409` 不在 [200,201,204] → reject Error → UI 进入 error 状态 → 显示 retry 按钮。

### 13.6 Layer 5：offset 的 Mutex 并发安全

**代码位置**：`server/ctrl/files.go:709, 712-733`

```go
type chunkedUpload struct {
    ...
    mu sync.Mutex     // ← 只保护 offset，不做全链路去重
}

func (this *chunkedUpload) Next(body io.ReadCloser) error {
    n, err := io.Copy(this.stream, body)
    body.Close()
    this.mu.Lock()
    this.offset += uint64(n)     // ← 并发写入时互斥
    this.mu.Unlock()
    return err
}

func (this *chunkedUpload) Meta() (uint64, uint64) {
    this.mu.Lock()
    defer this.mu.Unlock()
    return this.offset, this.size   // ← 读取时互斥
}
```

**能保护的范围**：
- ✅ 同一条 TUS 上传链路中，多个 goroutine 并发读写 offset 的数据一致性
- ❌ 不阻止同用户多标签页同时对同一路径进行 PATCH（需要 Layer 1 + Layer 4 配合）
- ❌ `io.Copy` 写入 pipe 的过程不受 mu 保护（同一时刻多 PATCH 并发写 stream 会导致数据交织）

> **实际上不会发生多 PATCH 并发写 stream**：因为前端分片是顺序 for 循环执行（`ctrl_upload.js:434`：`for (...; i<numberOfChunks; i++) { await executeHttp(...) }`），同一文件的 PATCH 天然串行。

### 13.7 多用户并发覆盖的竞态漏洞

**最终竞态窗口**：CanEdit=true 用户（管理员/所有者）的上传完全没有去重：

```
  时间轴 ─────────────────────────────────────────────►
  Admin A:  CanEdit=true → Layer3 跳过 → POST → PATCH...(offset 50%)
  Admin B:  CanEdit=true → Layer3 跳过 → POST → PATCH...(offset 50%)
                                    │
                                    ▼
                    最后一个 PATCH 完成的 Close()
                    → Backend.Save 的结果生效
                    → 另一个人的数据永久丢失
```

**代码证明**（`files.go:492-514` 之间的注释位置）：
- CanEdit=true → 直接跳过 500-513 行的存在性检查
- CanEdit=true 直接进入 516 行 AuthorisationMiddleware → 继续 TUS 流程

---

## 十四、上传超时的客户端断连处理路径

Filestash 的超时处理是 **「无主动超时 + 多层被动超时」** 的组合。整个上传链路 **没有任何地方设置主动的超时阈值**（没有 XHR timeout、没有 HTTP Server WriteTimeout、没有 per-request context deadline），完全依赖底层系统被动超时 + 24h 缓存驱逐兜底。

### 14.1 超时层级全景图

```
 用户发起分片上传 (PATCH /api/files/save)
        │
        │
        ▼
┌─────────────────────────────────────────────────────┐
│ Layer 1: 前端 XHR (无主动超时)                       │
│ ctrl_upload.js:527-585                                │
│ ✗ 未设置 xhr.timeout                                 │
│ ✗ onprogress 只算速度，无超时检测                     │
│ 处理: onerror / onabort → reject Promise             │
└──────────────────────┬──────────────────────────────┘
                       │ TCP 断连 / 浏览器内置超时 / 手动 abort
                       ▼
┌─────────────────────────────────────────────────────┐
│ Layer 2: Go HTTP Server (无 Read/Write 超时)          │
│ plg_starter_http/index.go:21-24                      │
│ plg_starter_https/index.go:23-29                     │
│ srv := &http.Server{                                 │
│     Addr:    fmt.Sprintf(":%d", port)                │
│     Handler: r                                       │
│     // ✗ ReadTimeout: 未设置                         │
│     // ✗ WriteTimeout: 未设置                        │
│     // ✗ IdleTimeout: 未设置                         │
│ }                                                    │
│ 处理: TCP keepalive 超时（内核默认 ~2h）              │
└──────────────────────┬──────────────────────────────┘
                       │ req.Body.Read() 返回 err
                       ▼
┌─────────────────────────────────────────────────────┐
│ Layer 3: FileSave PATCH handler                      │
│ files.go:636-644                                      │
│ io.Copy(this.stream, req.Body)                        │
│ → req.Body 读到 EOF/错误时自然返回                    │
│ → body.Close() 执行                                  │
│ → this.offset += n (只加成功字节)                    │
│ 处理: err 非 nil → HTTP 403 给前端                   │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│ Layer 4: io.Pipe 背压断连处理                        │
│ files.go:672-685 (createChunkedUploader)              │
│ Backend.Save goroutine 读得慢 → PATCH 写阻塞           │
│ 处理: 无超时，一直阻塞（依赖客户端 ABORT/浏览器超时）   │
└──────────────────────┬──────────────────────────────┘
                       │ 用户/浏览器主动断开 → stream.Close 触发
                       ▼
┌─────────────────────────────────────────────────────┐
│ Layer 5: TUS 缓存 24h 驱逐兜底                       │
│ files.go:687-699                                      │
│ chunkedUploadCache (retention=1440min, cleanup=1min) │
│ OnEvict 回调 → c.Close() → stream.Close() →          │
│   → Backend.Save 读 pipe 收到 ErrClosedPipe          │
│   → done channel 返回错误                            │
│ 处理: 泄漏的半完成上传最终被清理                     │
└─────────────────────────────────────────────────────┘
```

### 14.2 Layer 1：前端 XHR（无主动超时）

**代码位置**：`public/assets/pages/filespage/ctrl_upload.js:527-585`

```javascript
function executeHttp(url, { method, headers, body, progress, speed }) {
    const xhr = new XMLHttpRequest();
    this.xhr = xhr;
    return new Promise((resolve, reject) => {
        xhr.open(method, ...);
        // ★ 没有 xhr.timeout = XXX 这一行！
        // ★ 没有 xhr.ontimeout = ... 这一行！
        xhr.upload.onprogress = (e) => {
            ... // 只算 percent + speed
            // ★ 5 秒窗口只用于速度平均，不触发任何超时判定
            if (e.timeStamp - prevProgress[0].timeStamp > 5000) {
                prevProgress.shift();  // 仅窗口老化，不取消
            }
        };
        xhr.upload.onabort = () => reject(ABORT_ERROR);
        xhr.onerror = (e) => reject(new AjaxError("failed", e, "FAILED"));
        xhr.onload = () => { ... };
        xhr.send(body);
    });
}
```

**超时来源（全依赖外部）**：
- 浏览器内置 TCP 超时（通常 5-10 分钟无活动）
- 代理/负载均衡器的空闲超时（如 Nginx proxy_read_timeout 默认 60s）
- 用户点击 $stop → `cancel()` → `xhr.abort()` → `onabort` → `ABORT_ERROR`

**cancel 实现**（`ctrl_upload.js:345-348`）：
```javascript
cancel() {
    if (this.xhr) assert.type(this.xhr, XMLHttpRequest).abort();
    this.xhr = null;
}
```

### 14.3 Layer 2：Go HTTP Server（无 Read/Write 超时）

**代码位置**：`server/plugin/plg_starter_http/index.go:21-24`
```go
srv := &http.Server{
    Addr:         fmt.Sprintf(":%d", port),
    Handler:      r,
    // 三大关键超时全部未设置：
    // ReadTimeout:       未设置 → 无限
    // ReadHeaderTimeout: 未设置 → 无限
    // WriteTimeout:      未设置 → 无限
    // IdleTimeout:       未设置 → 无限
}
```

**HTTPS starter 同样未设置**（`plg_starter_https/index.go:23-29`）：
```go
srv := &http.Server{
    Addr:         fmt.Sprintf(":%d", port),
    Handler:      r,
    TLSNextProto: ...,
    TLSConfig:    ...,
    ErrorLog:     NewNilLogger(),
    // 同样无 Read/Write/Idle 超时
}
```

**实际影响**：
- PATCH 请求体慢速发送（1KB/s）→ 服务器一直等（不超时）
- 网络半断开（丢包 90%）→ 服务器一直阻塞在 `req.Body.Read()`
- 最终依赖：
  - TCP KeepAlive（`net.ListenConfig` 默认启用，通常 2 小时）
  - 操作系统 / 负载均衡器 / 浏览器的底层超时

### 14.4 Layer 3：FileSave PATCH handler（自然结束）

**代码位置**：`server/ctrl/files.go:636-644`

```go
reader := req.Body
if hash != nil {
    reader = io.NopCloser(io.TeeReader(req.Body, hash))
}
if err := uploader.Next(reader); err != nil {
    // Next 返回 err（可能是 req.Body 读取错误）
    Log.Debug("files::save::tus action=uploader.next path=%s err=%s", path, err.Error())
    SendErrorResult(res, NewError(err.Error(), 403))
    return
}
```

**Next 内部**（`files.go:712-719`）：
```go
func (this *chunkedUpload) Next(body io.ReadCloser) error {
    n, err := io.Copy(this.stream, body)
    body.Close()
    this.mu.Lock()
    this.offset += uint64(n)    // ← 只累加成功写入的字节
    this.mu.Unlock()
    return err                 // ← 原封不动返回错误
}
```

**断连时的字节一致性**：
- `io.Copy` 内部是 32KB 循环 Read + Write
- 当 `body.Read()` 返回错误（断连），`n` 只包含最后一次成功 copy 的字节数
- `offset += n` → 恰好是数据正确写入 pipe 的字节数
- **不会半字节计 offset**，断点续传时恰好从正确位置继续 ✓

### 14.5 Layer 4：io.Pipe 背压 + 断连锁定

**代码位置**：`server/ctrl/files.go:672-685`

```go
func createChunkedUploader(save func(path string, file io.Reader) error, path string, size uint64) *chunkedUpload {
    r, w := io.Pipe()
    done := make(chan error, 1)
    go func() {
        done <- save(path, r)  // ★ 独立 goroutine，无限阻塞也无人取消
    }()
    return &chunkedUpload{
        fn:     save,
        stream: w,
        done:   done,
        ...
    }
}
```

**背压场景**（慢速 Backend）：
```
客户端 100MB/s 写入 PATCH
        │
        ▼
io.PipeWriter (内存缓冲区)
        │ 64KB 满 → 阻塞 PATCH 的 io.Copy
        ▼
io.PipeReader
        │
        ▼
Backend.Save (FTP 慢速 500KB/s)
```
- 效果：PATCH 响应被拉长，但 PATCH handler 不会超时
- 客户端 `xhr.onprogress` 会看到 loaded/total 卡住（但不会触发超时）

**断连场景**（客户端断网 20 分钟恢复）：
1. 客户端断网 → `xhr.onerror` 或 TCP RST → Go 侧 `req.Body.Read()` 返回错误
2. `io.Copy` 返回错误 → `Next` 返回错误 → HTTP 403 响应（如果 socket 还能写）
3. 但 `save()` goroutine 继续等待 pipe 的更多数据
4. **没有人通知 goroutine 客户端已经断连** → pipe 保持打开
5. 最终依赖 Layer 5：24h 后 chunkedUploadCache.OnEvict → `stream.Close()` → goroutine 退出

### 14.6 Layer 5：TUS 缓存 24h 驱逐兜底

**代码位置**：`server/ctrl/files.go:687-699`

```go
func initChunkedUploader() {
    chunkedUploadCache = NewAppCache(60*24, 1)  // retention 24h, cleanup 1min
    chunkedUploadCache.OnEvict(func(key string, value interface{}) {
        c := value.(*chunkedUpload)
        if c == nil { return }
        if err := c.Close(); err != nil {       // ★ 强制关闭 pipe
            Log.Warning("ctrl::files::chunked::cleanup action=close err=%s", err.Error())
            return
        }
    })
}
```

**Close 的连锁反应**（`files.go:721-728`）：
```go
func (this *chunkedUpload) Close() error {
    this.stream.Close()       // ① 写端关闭 → PipeReader 收到 ErrClosedPipe
    err := <-this.done        // ② 等待 save() goroutine 退出
    this.once.Do(func() { close(this.done) })
    return err
}
```

**效果**：任何泄漏的上传会话（用户关浏览器、断网、TAB 崩溃）最多 24h 后：
- pipe 被关闭
- save() goroutine 退出（可能返回错误给 done channel）
- cache 条目删除
- 内存回收

### 14.7 各 Backend 的内部超时补充

| Backend | 内部超时配置 | 说明 |
|---------|------------|------|
| **FTP** | `plg_backend_ftp/index.go:92` `goftp.Config{Timeout: timeout}` | 控制 FTP 数据连接超时，默认值从 session params 中取 |
| **S3** | AWS SDK 默认 HTTP 客户端 | Go http.Client 默认无超时，SDK 内部有 per-request 重试（最多 3 次指数退避） |
| **Local** | 无 | 直接系统调用 `write()`，无应用层超时 |
| **Backblaze** | `HTTPClient()` 中的 Transport（`common/http.go`） | `DialContext` 30s，TLSHandshake 10s，但 ResponseHeaderTimeout 未设置 |
| **Azure** | SDK 内部策略 | Azure SDK 默认管道包含重试策略（4 次 800ms-60s 指数退避） |

---

## 十五、关键文件索引

| 文件 | 作用 | 核心行号 |
|------|------|----------|
| `public/assets/pages/filespage/ctrl_upload.js` | 前端上传控制器（分片、断点、UI） | 全程 |
| ↳ `componentUploadQueue()` | 队列调度 + 状态机 | 102-326 |
| ↳ `workerImplFile` | 文件上传执行器（TUS 协议实现） | 335-468 |
| ↳ `executeHttp()` | XHR 封装（进度/速度采集） | 527-585 |
| ↳ `processItems()` | 拖拽目录 BFS 遍历 + 依赖建模 | 649-719 |
| `server/ctrl/files.go` | 后端文件 API 路由 | 全程 |
| ↳ `FileSave()` | TUS 协议路由分发 | 479-670 |
| ↳ `chunkedUpload` 结构体 | 流合并器（Pipe + offset 管理） | 702-734 |
| ↳ `createChunkedUploader()` | 启动流式合并 goroutine | 672-685 |
| ↳ `initChunkedUploader()` | 缓存初始化（24h） | 687-700 |
| `server/common/cache.go` | AppCache 实现（基于 go-cache） | 11-77 |
| `server/common/config.go` | 上传配置项定义 | 79-80, 299, 324 |
| `server/common/types.go` | IBackend/IAuthorisation 接口定义 | 13-41 |
| `server/common/backend.go` | Backend Driver 注册与获取 | 11-38 |
| `server/common/plugin.go` | Hooks 注册机制（AuthMiddleware 等） | 138-149 |
| `server/common/constants.go` | TMP_PATH 等常量定义 | 20-56 |
| `server/model/permissions.go` | CanEdit/CanUpload 权限判定 | 7-33 |
| `server/routes.go` | API 路由注册（RateLimiter 挂载点） | 53-68 |
| `server/middleware/http.go` | RateLimiter 令牌桶实现 | 107-121 |
| `server/common/constants.go` | 启动时 TMP_PATH 清理逻辑 | 54-55 |
| `server/common/crypto.go` | GenerateID hash 算法（归属判定核心） | 193-218 |
| `server/middleware/session.go` | SessionStart 中间件、_extractSession/_extractBackend | 57-82, 262-319 |
| `server/model/files.go` | NewBackend 构造器（白名单校验+backend.Init） | 9-51 |
| `server/plugin/plg_starter_http/index.go` | HTTP Server 启动（无 Read/Write 超时设置） | 18-36 |
| `server/plugin/plg_starter_https/index.go` | HTTPS Server 启动（无 Read/Write 超时设置） | 18-55 |
| `server/plugin/plg_backend_local/index.go` | Local Backend Save 实现（O_TRUNC 风险） | 124-134 |
| `server/plugin/plg_backend_s3/index.go` | S3 Backend Save（s3manager）+ threadSize 控制 | 87-97, 569-587 |
| `server/plugin/plg_backend_ftp/index.go` | FTP Backend Save + Execute 自动重连（丢数据风险） | 80-103, 363-399 |
| `server/plugin/plg_backend_backblaze/index.go` | Backblaze Save（整体 SHA1 上传） | 401-459 |
| `server/plugin/plg_backend_azure/index.go` | Azure Save（UploadStream Block Blob） | 331-338 |
| `public/assets/pages/filespage/model_virtual_layer.js` | UI 虚拟文件层（loading 状态管理） | 全程 |
| ↳ `save()` | 文件保存前后的 UI 状态切换 | 136-179 |
