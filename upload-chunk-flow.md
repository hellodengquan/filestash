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

## 六、关键文件索引

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
| `public/assets/pages/filespage/model_virtual_layer.js` | UI 虚拟文件层（loading 状态管理） | 全程 |
| ↳ `save()` | 文件保存前后的 UI 状态切换 | 136-179 |
