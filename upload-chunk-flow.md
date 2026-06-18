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

## 九、关键文件索引

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
| `server/plugin/plg_backend_local/index.go` | Local Backend Save 实现 | 124-134 |
| `server/plugin/plg_backend_s3/index.go` | S3 Backend Save 实现（s3manager） | 569-587 |
| `server/plugin/plg_backend_ftp/index.go` | FTP Backend Save + Execute 重连 | 363-399 |
| `public/assets/pages/filespage/model_virtual_layer.js` | UI 虚拟文件层（loading 状态管理） | 全程 |
| ↳ `save()` | 文件保存前后的 UI 状态切换 | 136-179 |
