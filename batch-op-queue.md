# 批量操作队列执行机制分析

## 概述

批量复制、移动、删除操作的队列异步执行机制主要在**前端**实现，后端为简单的同步API处理。整个流程分为四个核心部分：

1. **前端发起** - 用户交互与请求构建
2. **后端入队** - API接收与同步处理（后端无队列）
3. **Worker执行** - 前端任务调度与并发控制
4. **进度推送** - 实时进度反馈与失败重试

---

## 1. 前端发起批量请求

### 1.1 文件选择机制

**核心文件**: `public/assets/pages/filespage/state_selection.js`

采用类似 Mac Finder 的选择模式，支持两种选择类型：

```javascript
// 数据结构
selection$ = [
    { "type": "anchor", "n": 0, "path": "/a.txt" },  // Cmd+点击
    { "type": "range",  "n": 5, "path": "/f.txt" },  // Shift+点击
]
```

**关键函数**:
- `addSelection({ shift, n, path, files })` - 添加选择项
- `expandSelection()` - 展开选择，返回完整路径数组
- `lengthSelection()` - 获取选中数量

### 1.2 批量操作入口

**核心文件**: `public/assets/pages/filespage/ctrl_submenu.js`

**批量删除** (ctrl_submenu.js:190-199):
```javascript
onClick(qs($page, `[data-action="delete"]`)).pipe(
    rxjs.mergeMap(() => {
        const paths = expandSelection().map(({ path }) => path);
        return rxjs.from(componentDelete(
            createModal(modalOpt), "remove",
        )).pipe(rxjs.mergeMap(() => {
            clearSelection();
            return rm(...paths);  // 传入所有路径
        }));
    }),
)
```

**单个删除** (ctrl_submenu.js:161-171):
```javascript
onClick(qs($page, `[data-action="delete"]`)).pipe(
    rxjs.mergeMap(() => {
        const path = expandSelection()[0].path;
        return rxjs.from(componentDelete(...)).pipe(
            rxjs.mergeMap(() => {
                clearSelection();
                clearCache(path);
                return rm(selection);
            })
        );
    }),
)
```

### 1.3 API请求层

**核心文件**: `public/assets/pages/filespage/model_files.js`

**批量删除** (model_files.js:62-69):
```javascript
export const rm = (...paths) => rxjs.forkJoin(paths.map((path) => ajax({
    url: withURLParams(`api/files/rm?path=${encodeURIComponent(path)}`),
    method: "POST",
    responseType: "json",
}))).pipe(
    handleSuccess(paths.length > 1 ? t("All Done!") : t("The file ...")),
    handleError,
);
```

**移动/重命名** (model_files.js:71-78):
```javascript
export const mv = (from, to) => ajax({
    url: withURLParams(`api/files/mv?from=${encodeURIComponent(from)}&to=${encodeURIComponent(to)}`),
    method: "POST",
    responseType: "json",
}).pipe(...);
```

> **关键点**: 使用 `rxjs.forkJoin` 并行发起多个HTTP请求，每个请求独立处理。

### 1.4 虚拟层（Virtual Layer）- UI 即时反馈

**核心文件**: `public/assets/pages/filespage/model_virtual_layer.js`

在API请求完成前就更新UI，提供即时视觉反馈：

```javascript
export function withVirtualLayer(ajax$, mutate) {
    mutate.before();           // 立即更新UI（显示loading）
    return ajax$.pipe(
        rxjs.tap((resp) => mutate.afterSuccess(resp)),  // 成功：更新缓存
        rxjs.catchError(mutate.afterError),             // 失败：回滚UI
    );
};
```

**删除操作的虚拟层流程** (model_virtual_layer.js:181-268):
1. `before()` - 给文件添加 `loading: true` 状态
2. `afterSuccess()` - 从列表和缓存中移除文件
3. `afterError()` - 移除loading状态，恢复原样

**移动操作的虚拟层流程** (model_virtual_layer.js:270-415):
- 同目录移动：修改文件名，显示loading
- 跨目录移动：源目录显示loading+删除标记，目标目录显示新文件loading

---

## 2. 后端入队机制

### 2.1 路由定义

**核心文件**: `server/routes.go`

```go
files.HandleFunc("/mv", NewMiddlewareChain(FileMv, middlewares)).Methods("POST")
files.HandleFunc("/rm", NewMiddlewareChain(FileRm, middlewares)).Methods("POST")
files.HandleFunc("/mkdir", NewMiddlewareChain(FileMkdir, middlewares)).Methods("POST")
files.HandleFunc("/touch", NewMiddlewareChain(FileTouch, middlewares)).Methods("POST")
files.HandleFunc("/save", NewMiddlewareChain(FileSave, middlewares)).Methods("POST", "PATCH")
```

### 2.2 控制器实现

**核心文件**: `server/ctrl/files.go`

**删除操作** (files.go:778-807):
```go
func FileRm(ctx *App, res http.ResponseWriter, req *http.Request) {
    // 1. 权限检查
    if model.CanEdit(ctx) == false {
        SendErrorResult(res, NewError("Permission denied", 403))
        return
    }
    // 2. 路径构建
    path, err := PathBuilder(ctx, req.URL.Query().Get("path"))
    // 3. 授权中间件检查
    for _, auth := range Hooks.Get.AuthorisationMiddleware() {
        if err = auth.Rm(ctx, path); err != nil {
            SendErrorResult(res, ErrNotAuthorized)
            return
        }
    }
    // 4. 执行删除（同步）
    err = ctx.Backend.Rm(path)
    if err != nil {
        SendErrorResult(res, err)
        return
    }
    SendSuccessResult(res, nil)
}
```

**移动操作** (files.go:736-776):
```go
func FileMv(ctx *App, res http.ResponseWriter, req *http.Request) {
    from, err := PathBuilder(ctx, req.URL.Query().Get("from"))
    to, err := PathBuilder(ctx, req.URL.Query().Get("to"))
    // 权限检查、授权检查...
    err = ctx.Backend.Mv(from, to)  // 同步执行
    SendSuccessResult(res, nil)
}
```

> **重要发现**: 后端**没有队列机制**，每个API请求是独立同步处理的。并发控制完全在前端实现。

---

## 3. Worker执行机制

Worker队列主要在**上传模块**实现，为批量上传提供并发控制。

### 3.1 核心数据结构

**核心文件**: `public/assets/pages/filespage/ctrl_upload.js`

```javascript
const MAX_WORKERS = 4;  // 最大并发数
const workers$ = new rxjs.BehaviorSubject({ tasks: [], size: null });

// 任务对象结构
task = {
    type: "file" | "directory",
    path: "/target/path.txt",
    exec: workerImplFile | workerImplDirectory,  // 执行器
    virtual: save(path, size),                   // 虚拟层对象
    done: false,
    ready: () => boolean,  // 任务是否就绪（处理依赖）
    file: () => Promise<File>,  // 延迟获取文件对象
}
```

### 3.2 任务入队

**处理文件列表** (ctrl_upload.js:587-647):
```javascript
async function processFiles(filelist) {
    const tasks = [];
    for (const currentFile of filelist) {
        const type = await detectFiletype(currentFile);
        let task = null;
        switch (type) {
        case "file":
            task = {
                type: "file",
                file: () => new Promise((resolve) => resolve(currentFile)),
                path,
                exec: workerImplFile,
                virtual: save(path, currentFile.size),
                done: false,
                ready: () => true,  // 文件无依赖
            };
            break;
        case "directory":
            task = {
                type: "directory",
                path: path + "/",
                exec: workerImplDirectory,
                virtual: mkdir(path),
                done: false,
                ready: () => true,
            };
            break;
        }
        task.virtual.before();  // 立即显示UI
        tasks.push(task);
    }
    return { tasks, size };
}
```

**处理拖拽目录（BFS遍历）** (ctrl_upload.js:649-719):
```javascript
async function processItems(itemList) {
    const bfs = async(queue) => {
        const tasks = [];
        while (queue.length > 0) {
            const entry = queue.shift();
            // 广度优先遍历目录树...
            
            // 目录创建完成后才能上传子文件
            task.ready = () => {
                for (let i=0; i<tasks.length; i++) {
                    if (tasks[i].path === task.path) break;
                    else if (tasks[i].type === "file") continue;
                    // 检查父目录是否已完成
                    else if (isInDirectory(tasks[i].path, task.path)) {
                        if (tasks[i].done === false) return false;
                    }
                }
                return true;
            };
            tasks.push(task);
        }
        return { tasks, size };
    };
}
```

### 3.3 Worker池调度

**任务调度循环** (ctrl_upload.js:259-325):
```javascript
let tasks = [];  // 全局任务池
const reservations = new Array(MAX_WORKERS).fill(false);

// 监听新任务
workers$.subscribe(async({ tasks: newTasks, loading = false }) => {
    if (loading) return;
    tasks = tasks.concat(newTasks);  // 新任务加入池
    
    // 启动worker直到达到MAX_WORKERS
    while (!$page.classList.contains("hidden")) {
        const nworker = reservations.indexOf(false);
        if (nworker === -1) break;  // worker池已满
        reservations[nworker] = true;
        // 启动worker，完成后释放槽位
        noFailureAllowed(processWorkerQueue.bind(null, nworker))
            .then(() => reservations[nworker] = false);
    }
});
```

**Worker主循环** (ctrl_upload.js:261-304):
```javascript
const processWorkerQueue = async(nworker) => {
    while (tasks.length > 0) {
        updateDOMGlobalTitle($page, t("Running")+"...");

        // step1: 获取下一个就绪任务
        const task = nextTask(tasks);
        if (!task) {
            await new Promise((resolve) => setTimeout(resolve, 1000));
            continue;
        }

        // step2: 检查任务是否已在运行（去重）
        const $tasks = qsa($page, `[data-path="${task.path}"][data-status="running"]`);
        if ($tasks.length > 0) {
            await new Promise((resolve) => setTimeout(resolve, 1000));
            tasks.unshift(task);  // 重新放回队首
            continue;
        }

        // step3: 执行任务完整生命周期
        const $task = qs($page, `[data-path="${task.path}"]`);
        const exec = task.exec({
            progress: (progress) => updateDOMTaskProgress($task, formatPercent(progress)),
            speed: (speed) => {
                updateDOMTaskSpeed($task, speed);
                updateDOMGlobalSpeed(nworker, speed);
            },
        });
        updateDOMWithStatus($task, { exec, status: "doing", nworker });
        
        try {
            await exec.run(task);
            updateDOMWithStatus($task, { exec, status: "done", nworker });
        } catch (err) {
            updateDOMWithStatus($task, { exec, status: "error", nworker });
        }
        
        updateTotal.incrementCompleted();
        task.done = true;
    }
};
```

**任务选择器** (ctrl_upload.js:305-313):
```javascript
const nextTask = (tasks) => {
    for (let i=0; i<tasks.length; i++) {
        const possibleTask = tasks[i];
        if (!possibleTask.ready()) continue;  // 跳过未就绪任务
        tasks.splice(i, 1);  // 从池中移除
        return possibleTask;
    }
    return null;
};
```

---

## 4. 进度推送与失败重试

### 4.1 进度推送机制

**HTTP请求执行器** (ctrl_upload.js:527-585):
```javascript
function executeHttp(url, { method, headers, body, progress, speed }) {
    const xhr = new XMLHttpRequest();
    const prevProgress = [];  // 用于计算平均速度
    
    return new Promise((resolve, reject) => {
        xhr.open(method, forwardURLParams(url, ["share"]));
        xhr.setRequestHeader("X-Requested-With", "XmlHttpRequest");
        xhr.withCredentials = true;
        
        // 进度事件
        xhr.upload.onprogress = (e) => {
            if (!e.lengthComputable) return;
            const percent = Math.floor(100 * e.loaded / e.total);
            progress(percent);  // 推送进度
            
            // 计算上传速度（滑动窗口平均）
            prevProgress.push(e);
            let avgSpeed = 0;
            for (let i=1; i<prevProgress.length; i++) {
                const p1 = prevProgress[i];
                const pm1 = prevProgress[i-1];
                avgSpeed += (p1.loaded - pm1.loaded) / ((p1.timeStamp - pm1.timeStamp)/1000);
            }
            avgSpeed = avgSpeed / (prevProgress.length - 1);
            speed(avgSpeed);  // 推送速度
            
            // 滑动窗口：只保留最近5秒的数据
            if (e.timeStamp - prevProgress[0].timeStamp > 5000) {
                prevProgress.shift();
            }
        };
        
        xhr.upload.onabort = () => reject(ABORT_ERROR);
        xhr.onerror = (e) => reject(new AjaxError("failed", e, "FAILED"));
        xhr.onload = () => {
            if ([200, 201, 204].indexOf(xhr.status) === -1) {
                reject(new Error(xhr.statusText));
                return;
            }
            progress(100);
            resolve({ status: xhr.status, headers: parseHeaders(xhr) });
        };
        
        xhr.send(body);
    });
}
```

### 4.2 DOM状态更新

**核心函数**: `updateDOMWithStatus` (ctrl_upload.js:195-257)

```javascript
const updateDOMWithStatus = ($task, { status, exec, nworker }) => {
    const cancel = () => exec.cancel();
    const executeMutation = (status) => {
        switch (status) {
        case "todo":
            updateDOMGlobalTitle($page, t("Running") + "...");
            break;
            
        case "doing":
            const $stop = assert.type($iconStop.cloneNode(true), HTMLElement);
            updateDOMTaskProgress($task, formatPercent(0));
            $task.classList.remove("error_color");
            $task.classList.add("todo_color");
            $task.setAttribute("data-status", "running");
            $task.querySelector(".file_control").replaceChildren($stop);
            $stop.onclick = () => {
                cancel();
                $task.removeAttribute("data-status");
                $task.querySelector(".file_control").classList.add("hidden");
            };
            $close.addEventListener("click", cancel, { once: true });
            break;
            
        case "done":
            updateDOMGlobalTitle($page, t("Done"));
            updateDOMTaskProgress($task, t("Done"));
            updateDOMGlobalSpeed(nworker, 0);
            updateDOMTaskSpeed($task, 0);
            $task.removeAttribute("data-path");
            $task.removeAttribute("data-status");
            $task.classList.remove("todo_color");
            $task.querySelector(".file_control").classList.add("hidden");
            $close.removeEventListener("click", cancel);
            break;
            
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
            $task.querySelector(".file_control").firstElementChild.remove();
            $task.querySelector(".file_control").appendChild($retry);
            
            // 重试按钮逻辑
            $retry.onclick = async() => {
                executeMutation("todo");
                executeMutation("doing");
                try {
                    await exec.retry();
                    executeMutation("done");
                } catch (err) {
                    executeMutation("error");
                }
            };
            $close.removeEventListener("click", cancel);
            break;
        }
    };
    executeMutation(status);
};
```

### 4.3 执行器接口与实现

**执行器基类** (ctrl_upload.js:328-333):
```javascript
class IExecutor {
    contructor() {}
    cancel() { throw new Error("NOT_IMPLEMENTED"); }
    retry() { throw new Error("NOT_IMPLEMENTED"); }
    run() { throw new Error("NOT_IMPLEMENTED"); }
}
```

**文件上传执行器** (ctrl_upload.js:335-468):
```javascript
function workerImplFile({ progress, speed }) {
    return new class Worker extends IExecutor {
        constructor() {
            super();
            this.xhr = null;
        }

        cancel() {
            if (this.xhr) assert.type(this.xhr, XMLHttpRequest).abort();
            this.xhr = null;
        }

        async run({ file, path, virtual }) {
            const _file = await file();
            const executeJob = () => this.prepareJob({ file: _file, path, virtual });
            // 绑定重试方法
            this.retry = () => {
                virtual.before();
                return executeJob();
            };
            return executeJob();
        }

        async prepareJob({ file, path, virtual }) {
            const chunkSize = getConfig("upload_chunk_size", 0) * 1024 * 1024;
            const numberOfChunks = Math.ceil(file.size / chunkSize);
            
            // Case1: 普通上传（单请求）
            if (chunkSize === 0 || numberOfChunks <= 1) {
                try {
                    await executeHttp.call(this, apiURL, {
                        method: "POST",
                        body: file,
                        progress,
                        speed,
                    });
                    virtual.afterSuccess();
                } catch (err) {
                    virtual.afterError();
                    if (err === ABORT_ERROR) return;
                    throw err;
                }
                return;
            }
            
            // Case2: 分块上传（TUS协议）
            const apiURL = toHref(`/api/files/save?path=${encodeURIComponent(path)}`);
            let uploadURL = "";
            let offset = 0;
            
            // TUS: 检查已上传进度（断点续传）
            try {
                const resp = await executeHttp.call(this, apiURL, {
                    method: "HEAD",
                    headers: { "Tus-Resumable": "1.0.0" },
                    progress: () => {},
                    speed,
                });
                if (file.size === parseInt(resp.headers["upload-length"])) {
                    offset = parseInt(resp.headers["upload-offset"]);
                    uploadURL = apiURL;
                }
            } catch (err) {}
            
            // TUS: 创建上传会话
            if (offset === 0) {
                const resp = await executeHttp.call(this, apiURL, {
                    method: "POST",
                    headers: {
                        "Tus-Resumable": "1.0.0",
                        "Upload-Length": file.size,
                    },
                    progress: () => {},
                    speed,
                });
                uploadURL = resp.headers.location;
            }
            
            // TUS: 上传分片
            try {
                for (let i=Math.ceil(offset/chunkSize); i<numberOfChunks; i++) {
                    if (this.xhr === null) break;
                    const chunk = file.slice(offset, offset + chunkSize);
                    const headers = {
                        "Tus-Resumable": "1.0.0",
                        "Content-Type": "application/offset+octet-stream",
                        "Upload-Offset": offset,
                    };
                    // 可选：校验和
                    if (TUS_CHECKSUM && crypto.subtle.digest) {
                        const hash = await crypto.subtle.digest("SHA-1", await chunk.arrayBuffer());
                        headers["Upload-Checksum"] = `sha1 ${toHex(hash)}`;
                    }
                    await executeHttp.call(this, uploadURL, {
                        method: "PATCH",
                        headers,
                        body: chunk,
                        progress: (p) => {
                            // 计算分片在整体中的进度
                            const start = Math.ceil(100 * offset / file.size);
                            const end = Math.ceil(100 * Math.min(file.size, offset + chunkSize) / file.size);
                            progress(Math.floor(start + (end - start) * p / 100));
                        },
                        speed,
                    });
                    offset += chunkSize;
                }
                virtual.afterSuccess();
            } catch (err) {
                virtual.afterError();
                if (err === ABORT_ERROR) return;
                throw err;
            }
        }
    }();
}
```

**目录创建执行器** (ctrl_upload.js:470-525):
```javascript
function workerImplDirectory({ progress }) {
    return new class Worker extends IExecutor {
        constructor() {
            super();
            this.xhr = null;
        }

        cancel() {
            assert.type(this.xhr, XMLHttpRequest).abort();
        }

        async run({ virtual, path }) {
            const executeJob = () => this.prepareJob({ virtual, path });
            this.retry = () => {
                virtual.before();
                return executeJob();
            };
            return executeJob();
        }

        async prepareJob({ virtual, path }) {
            let percent = 0;
            // 模拟进度（目录创建很快，用定时器模拟进度条）
            const id = setInterval(() => {
                percent += 10;
                if (percent >= 100) {
                    clearInterval(id);
                    return;
                }
                progress(percent);
            }, 100);
            
            try {
                await executeHttp.call(this, toHref(`/api/files/mkdir?path=${encodeURIComponent(path)}`), {
                    method: "POST",
                    progress,
                    speed: () => {},
                });
                clearInterval(id);
                progress(100);
                virtual.afterSuccess();
            } catch (err) {
                clearInterval(id);
                virtual.afterError();
                if (err === ABORT_ERROR) return;
                throw err;
            }
        }
    }();
}
```

### 4.4 后端TUS协议支持

**核心文件**: `server/ctrl/files.go` (FileSave函数, lines 479-670)

后端支持TUS（Resumable Upload）协议的核心方法：
- `OPTIONS` - 声明支持的扩展和校验算法
- `HEAD` - 查询已上传偏移量
- `POST` - 创建上传会话
- `PATCH` - 上传分片数据

```go
// 分块上传缓存（会话级）
var chunkedUploadCache AppCache

type chunkedUpload struct {
    fn     func(path string, file io.Reader) error
    stream *io.PipeWriter  // 流式写入后端
    offset uint64          // 当前偏移
    size   uint64          // 总大小
    done   chan error      // 完成信号
    once   sync.Once
    mu     sync.Mutex
}

// 创建上传器：通过io.Pipe实现流式上传
func createChunkedUploader(save func(path string, file io.Reader) error, path string, size uint64) *chunkedUpload {
    r, w := io.Pipe()
    done := make(chan error, 1)
    go func() {
        done <- save(path, r)  // 异步执行保存
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

---

## 整体串联流程图

```
前端用户交互
    ↓
[ctrl_submenu.js] 点击删除/移动按钮
    ↓
[state_selection.js] expandSelection() 获取选中路径
    ↓
[model_files.js] rm(...paths) / mv(from, to)
    │
    ├─→ [model_virtual_layer.js] withVirtualLayer()
    │       ├─ before() → 立即更新UI（显示loading）
    │       ├─ 成功 → afterSuccess() → 更新缓存
    │       └─ 失败 → afterError() → 回滚UI
    │
    └─→ rxjs.forkJoin 并行发起N个HTTP请求
            ↓
后端API处理（每个请求独立同步执行）
    ↓
[server/routes.go] 路由匹配
    ↓
[server/ctrl/files.go] FileRm / FileMv
    ├─ 权限检查
    ├─ 授权中间件
    └─ ctx.Backend.Rm/Mv（同步调用）
    ↓
返回JSON响应
    ↓
前端RxJS流处理
    ├─ 成功 → 更新缓存，刷新列表
    └─ 失败 → 显示错误通知，回滚虚拟层
```

---

## 关键设计洞察

1. **前端中心化队列**: 并发控制、进度管理、重试逻辑全部在前端实现，后端保持简单。

2. **虚拟层优化**: API请求发出前就更新UI，提供即时反馈，显著提升UX。

3. **依赖感知调度**: `ready()` 函数确保目录先创建完成后才能上传子文件。

4. **TUS断点续传**: 支持大文件分块上传，刷新页面后可继续上传。

5. **滑动窗口速度计算**: 使用最近5秒的进度数据计算平均速度，避免抖动。

6. **无状态后端**: 每个API独立无状态，便于水平扩展。

---

## 代码位置索引

| 模块 | 文件位置 | 关键行号 |
|------|----------|----------|
| 批量删除入口 | `public/assets/pages/filespage/ctrl_submenu.js` | 190-199 |
| API请求层 | `public/assets/pages/filespage/model_files.js` | 62-78 |
| 虚拟层 | `public/assets/pages/filespage/model_virtual_layer.js` | 36-42, 181-268 |
| Worker调度 | `public/assets/pages/filespage/ctrl_upload.js` | 259-325 |
| 进度推送 | `public/assets/pages/filespage/ctrl_upload.js` | 527-585 |
| 失败重试 | `public/assets/pages/filespage/ctrl_upload.js` | 195-257, 353-361 |
| TUS后端 | `server/ctrl/files.go` | 479-670 |
| 文件选择 | `public/assets/pages/filespage/state_selection.js` | 26-119 |

---

## 5. Worker 数量配置如何影响吞吐量与队列堆积

### 5.1 核心常量与调度器

**文件**: `public/assets/pages/filespage/ctrl_upload.js:100`

```javascript
const MAX_WORKERS = 4;
```

`MAX_WORKERS` 是硬编码常量，不可通过配置修改。它决定了同一时刻最多有多少个上传任务可以并发执行。

调度器使用一个长度为 `MAX_WORKERS` 的布尔数组 `reservations` 作为"槽位簿记"：

**文件**: `public/assets/pages/filespage/ctrl_upload.js:259-325`

```javascript
let tasks = [];                                              // 全局任务池
const reservations = new Array(MAX_WORKERS).fill(false);     // 槽位：false=空闲 true=占用

workers$.subscribe(async({ tasks: newTasks, loading = false }) => {
    if (loading) return;
    tasks = tasks.concat(newTasks);                          // 新任务入池

    while (!$page.classList.contains("hidden")) {
        const nworker = reservations.indexOf(false);         // 找第一个空闲槽位
        if (nworker === -1) break;                           // 池满，不再启动新 worker
        reservations[nworker] = true;                        // 占位
        noFailureAllowed(processWorkerQueue.bind(null, nworker))
            .then(() => reservations[nworker] = false);      // 完成后释放
    }
    reservations.fill(false);                                // 所有 worker 完成后重置
});
```

### 5.2 吞吐量模型

每个 worker 运行 `processWorkerQueue`，在 while 循环中**串行**消费任务：

**文件**: `public/assets/pages/filespage/ctrl_upload.js:261-304`

```javascript
const processWorkerQueue = async(nworker) => {
    while (tasks.length > 0) {
        const task = nextTask(tasks);        // 从池中取一个就绪任务
        if (!task) {
            await new Promise((resolve) => setTimeout(resolve, 1000));  // 无就绪任务则等1秒
            continue;
        }

        // 去重检查：同一文件不能有两个 worker 同时上传
        const $tasks = qsa($page, `[data-path="${task.path}"][data-status="running"]`);
        if ($tasks.length > 0) {
            await new Promise((resolve) => setTimeout(resolve, 1000));
            tasks.unshift(task);             // 放回队首
            continue;
        }

        const exec = task.exec({ progress, speed });
        updateDOMWithStatus($task, { exec, status: "doing", nworker });
        try {
            await exec.run(task);            // 阻塞直到当前任务完成
            updateDOMWithStatus($task, { exec, status: "done", nworker });
        } catch (err) {
            updateDOMWithStatus($task, { exec, status: "error", nworker });
        }
        updateTotal.incrementCompleted();
        task.done = true;
    }
};
```

**吞吐量分析**：

```
理论吞吐量 = MAX_WORKERS × 单 worker 处理速率

其中：
- 单 worker 处理速率 = 1 / (网络RTT + 传输时间 + 后端处理时间)
- 当 MAX_WORKERS=4 时，最多 4 个任务同时处于 "doing" 状态
- 其余任务在 tasks[] 中排队，状态为 "Waiting"
```

**不同 MAX_WORKERS 值的场景对比**：

| MAX_WORKERS | 并发数 | 10个文件总耗时（假设单文件10s） | 队列中等待数 | 网络带宽占用 |
|:-----------:|:------:|:------------------------------:|:------------:|:----------:|
| 1 | 1 | 100s | 9 | 低 |
| 2 | 2 | 50s | 8 | 中 |
| 4（当前值） | 4 | 30s | 6 | 高 |
| 8 | 8 | 20s | 2 | 饱和 |
| 16 | 16 | 10s | 0 | 可能超限 |

### 5.3 队列堆积的三个关键点

**关键点1：去重保护导致任务"弹回"**

当同一路径的任务正在执行时，新取出的同路径任务会被放回队首并等1秒：

```javascript
// ctrl_upload.js:273-278
const $tasks = qsa($page, `[data-path="${task.path}"][data-status="running"]`);
if ($tasks.length > 0) {
    await new Promise((resolve) => setTimeout(resolve, 1000));
    tasks.unshift(task);     // 放回队首，1秒后重试
    continue;
}
```

**影响**：如果 A 文件正在上传，B worker 又取到了 A，B 会空转1秒。在极端情况下，4 个 worker 可能都在等同一个文件，造成吞吐量骤降。

**关键点2：依赖感知调度导致的"饥饿等待"**

拖拽目录上传时，子文件的 `ready()` 会检查父目录是否已完成：

```javascript
// ctrl_upload.js:698-710
task.ready = () => {
    const isInDirectory = (filepath, folder) => folder.indexOf(filepath) === 0;
    for (let i=0; i<tasks.length; i++) {
        if (tasks[i].path === task.path) break;
        else if (tasks[i].type === "file") continue;
        else if (isInDirectory(tasks[i].path, task.path) === false) continue;
        if (tasks[i].done === false) return false;   // 父目录未完成 → 任务不就绪
    }
    return true;
};
```

**影响**：假设上传 `/docs/` 目录结构：
```
/docs/             → task[0], type=directory, ready=true
/docs/a.txt        → task[1], type=file, ready() => task[0].done
/docs/b.txt        → task[2], type=file, ready() => task[0].done
/docs/sub/         → task[3], type=directory, ready() => task[0].done
/docs/sub/c.txt    → task[4], type=file, ready() => task[3].done
```

如果 task[0]（目录 `/docs/`）的 mkdir 请求很慢，task[1]-task[4] 全部处于"不就绪"状态。即使 MAX_WORKERS=4，所有 worker 也会在 `nextTask` 中返回 null 后进入1秒轮询等待：

```javascript
// ctrl_upload.js:305-313
const nextTask = (tasks) => {
    for (let i=0; i<tasks.length; i++) {
        const possibleTask = tasks[i];
        if (!possibleTask.ready()) continue;    // 跳过不就绪任务
        tasks.splice(i, 1);
        return possibleTask;
    }
    return null;    // 无就绪任务
};
```

**关键点3：错误不阻塞队列**

`noFailureAllowed` 保证 worker 异常不会中断整个调度循环：

```javascript
// ctrl_upload.js:314
const noFailureAllowed = (fn) => fn().catch(() => noFailureAllowed(fn));
```

任务失败时，`updateDOMWithStatus` 将该任务标记为 "error" 状态并显示重试按钮，但 worker 不阻塞，继续处理下一个任务：

```javascript
// ctrl_upload.js:293-296
} catch (err) {
    updateDOMWithStatus($task, { exec, status: "error", nworker });
}
updateTotal.incrementCompleted();  // 无论成功失败都计数
task.done = true;                  // 标记完成，解除子任务依赖
```

> **注意**：`task.done = true` 在失败时也被设置为 true。这意味着依赖此任务的子任务会被放行，即使父任务（如 mkdir）实际失败了。子文件上传时可能因目录不存在而再次失败，用户需要手动重试。

### 5.4 全局速度统计的 worker 数量依赖

**文件**: `public/assets/pages/filespage/ctrl_upload.js:183-193`

```javascript
const updateDOMGlobalSpeed = (function(workersSpeed) {
    let last = 0;
    return (nworker, currentWorkerSpeed) => {
        workersSpeed[nworker] = currentWorkerSpeed;      // 按槽位号记录速度
        if (new Date().getTime() - last <= 500) return;  // 500ms 节流
        last = new Date().getTime();
        const speed = workersSpeed.reduce((acc, el) => acc + el, 0);  // 所有活跃 worker 速度求和
        const $speed = assert.type($page.firstElementChild?.nextElementSibling?.firstElementChild, HTMLElement);
        $speed.textContent = formatSpeed(speed);
    };
}(new Array(MAX_WORKERS).fill(0))));   // 初始化 MAX_WORKERS 个槽位
```

全局速度 = 所有活跃 worker 速度之和。`MAX_WORKERS` 决定了速度数组的长度。如果改为更大值，全局速度统计自然更准确（更多槽位）；但当前值 4 已足够覆盖浏览器对同一域名的 HTTP 并发连接数限制（通常 6 个），实际 4 个并发上传已接近浏览器网络栈的实用上限。

---

## 6. 网络断连后的进度推送恢复流程

### 6.1 项目中不存在 WebSocket

**重要发现**：本项目**没有使用 WebSocket 或 SSE** 进行进度推送。进度信息完全依赖浏览器原生 `XMLHttpRequest.upload.onprogress` 事件，这是一个**请求级**的回调机制，在请求中断后事件流自然终止。

因此，"断连后进度推送恢复"实际上包含两个层面：
1. **上传请求中断**：当前 XHR 请求失败，需要重新发起
2. **TUS 断点续传**：利用 TUS 协议在服务端保存的上传偏移量，从断点继续上传

### 6.2 网络断连时 XHR 的行为

**文件**: `public/assets/pages/filespage/ctrl_upload.js:527-585`

```javascript
function executeHttp(url, { method, headers, body, progress, speed }) {
    const xhr = new XMLHttpRequest();
    return new Promise((resolve, reject) => {
        // ...
        xhr.upload.onprogress = (e) => {       // ← 进度回调只在连接存活时触发
            if (!e.lengthComputable) return;
            const percent = Math.floor(100 * e.loaded / e.total);
            progress(percent);                  // 推送到 DOM
            // ... 速度计算
        };

        xhr.upload.onabort = () => reject(ABORT_ERROR);   // 用户取消
        xhr.onerror = (e) => reject(new AjaxError("failed", e, "FAILED"));  // ← 网络断连触发
        xhr.onload = () => { /* ... */ };
        xhr.send(body);
    });
}
```

**断连时的调用链**：

```
网络断开
    ↓
XMLHttpRequest 触发 onerror
    ↓
reject(new AjaxError("failed", e, "FAILED"))
    ↓
executeHttp 的 Promise 被 reject
    ↓
调用方 catch 块执行
```

### 6.3 上传失败后的错误传播路径

以分块上传为例，跟踪从 XHR 失败到 UI 重试按钮的完整路径：

**Step 1: TUS 分片上传失败**

```javascript
// ctrl_upload.js:447-458
await executeHttp.call(this, uploadURL, {
    method: "PATCH",
    headers,
    body: chunk,
    progress: (p) => { /* ... */ },
    speed,
});
offset += chunkSize;
```

如果网络断开，`executeHttp` reject，控制流跳到 catch：

```javascript
// ctrl_upload.js:461-465
} catch (err) {
    virtual.afterError();              // 1. 虚拟层回滚UI
    if (err === ABORT_ERROR) return;   // 2. 用户主动取消则静默
    throw err;                          // 3. 网络错误向上抛出
}
```

**Step 2: workerImplFile.run() 抛出异常**

```javascript
// ctrl_upload.js:353-361
async run({ file, path, virtual }) {
    const _file = await file();
    const executeJob = () => this.prepareJob({ file: _file, path, virtual });
    this.retry = () => {
        virtual.before();
        return executeJob();
    };
    return executeJob();               // ← 异常从此处抛出
}
```

注意：`this.retry` 在 `run()` 中被绑定，**即使 run 失败，retry 方法仍然可用**。

**Step 3: processWorkerQueue 捕获异常**

```javascript
// ctrl_upload.js:290-295
try {
    await exec.run(task);
    updateDOMWithStatus($task, { exec, status: "done", nworker });
} catch (err) {
    updateDOMWithStatus($task, { exec, status: "error", nworker });  // ← 进入 error 状态
}
updateTotal.incrementCompleted();
task.done = true;
```

**Step 4: UI 显示重试按钮**

```javascript
// ctrl_upload.js:227-249
case "error":
    const $retry = assert.type($iconRetry.cloneNode(true), HTMLElement);
    updateDOMGlobalTitle($page, t("Error"));
    updateDOMTaskProgress($task, t("Error"));

    $task.classList.add("error_color");
    $task.querySelector(".file_control").appendChild($retry);

    $retry.onclick = async() => {
        executeMutation("todo");            // 1. 状态重置为待执行
        executeMutation("doing");           // 2. 状态切换为执行中
        try {
            await exec.retry();             // 3. 调用之前绑定的 retry 方法
            executeMutation("done");        // 4. 成功
        } catch (err) {
            executeMutation("error");       // 5. 再次失败，可继续重试
        }
    };
    break;
```

### 6.4 重试时 TUS 断点续传的恢复

用户点击重试按钮后，`exec.retry()` 被调用，其绑定的函数为：

```javascript
// ctrl_upload.js:356-359
this.retry = () => {
    virtual.before();       // 重新显示 loading 状态
    return executeJob();    // 重新执行 prepareJob
};
```

`prepareJob` 开头的 TUS HEAD 请求是**断点续传的关键**：

```javascript
// ctrl_upload.js:397-412
try {
    const resp = await executeHttp.call(this, apiURL, {
        method: "HEAD",                         // ← 查询服务端已上传偏移量
        headers: { ...tusHeaders },
        body: null,
        progress: () => {},
        speed,
    });
    if (file.size === parseInt(resp.headers["upload-length"])) {
        const tmp = parseInt(resp.headers["upload-offset"]);  // ← 获取已上传字节数
        if (tmp > 0) {
            offset = tmp;                        // ← 从断点继续
            uploadURL = apiURL;
        }
    }
} catch (err) {}    // HEAD 失败不阻塞，会创建新上传会话
```

**后端 HEAD 请求处理**：

**文件**: `server/ctrl/files.go:554-567`

```go
if proto == "tus" && req.Method == http.MethodHead {
    c := chunkedUploadCache.Get(cacheKey)
    if c == nil {
        SendErrorResult(res, ErrNotFound)
        return
    }
    offset, length := c.(*chunkedUpload).Meta()
    h.Set("Tus-Resumable", "1.0.0")
    h.Set("Upload-Offset", fmt.Sprintf("%d", offset))    // ← 返回已上传偏移量
    h.Set("Upload-Length", fmt.Sprintf("%d", length))     // ← 返回总大小
    h.Set("Cache-Control", "no-store")
    res.WriteHeader(http.StatusNoContent)
    return
}
```

**后端缓存的生命周期**：

**文件**: `server/ctrl/files.go:688-700`

```go
func initChunkedUploader() {
    chunkedUploadCache = NewAppCache(60*24, 1)  // 保留 60×24=1440 分钟 = 24 小时
    chunkedUploadCache.OnEvict(func(key string, value interface{}) {
        c := value.(*chunkedUpload)
        if err := c.Close(); err != nil {
            Log.Warning("ctrl::files::chunked::cleanup action=close err=%s", err.Error())
        }
    })
}
```

缓存 key 由路径和会话 ID 组成：

```go
// server/ctrl/files.go:543-546
cacheKey := map[string]string{
    "path":    path,
    "session": GenerateID(ctx.Session),
}
```

### 6.5 完整断连恢复时序

```
时间轴                    前端                              后端
─────────────────────────────────────────────────────────────────
t0    用户上传100MB文件，chunkSize=10MB
t1    POST /api/files/save              →     创建 chunkedUpload，设入 cache
      Tus-Resumable: 1.0.0                    Upload-Length: 104857600
      Upload-Length: 104857600           ←     201 Created, Location: /api/files/save?path=...
t2    PATCH 上传 chunk0 (0-10MB)        →     io.PipeWriter 写入 10MB
      Upload-Offset: 0
      progress(0%) → progress(8%) → progress(10%)
t3    PATCH 上传 chunk1 (10-20MB)       →     写入 20MB
      Upload-Offset: 10485760
      progress(10%) → progress(18%) → progress(20%)
      ...
t7    PATCH 上传 chunk5 (50-60MB)       →     写入 60MB
      Upload-Offset: 52428800
      progress(50%) → progress(58%)
      
      ╳ ═══ 网络断开 ═══ ╳
      
      XHR onerror → reject("FAILED")
      ↓
      virtual.afterError() → 回滚 UI
      ↓
      processWorkerQueue catch → updateDOMWithStatus("error")
      ↓
      UI: 显示 "Error" + 重试按钮
                                               offset=60MB 保存在 cache 中
                                               (24小时内有效)

t8    ╳ ═══ 网络恢复 ═══ ╳

t9    用户点击重试按钮
      ↓
      exec.retry()
      ↓
      virtual.before() → 重新显示 loading
      ↓
      prepareJob() 被重新调用
      ↓
      HEAD /api/files/save?path=...     →     从 cache 读取 chunkedUpload
      Tus-Resumable: 1.0.0                   Meta() → offset=62914560, size=104857600
                                            ←
      Upload-Offset: 62914560                204 No Content
      Upload-Length: 104857600
      ↓
      offset = 62914560 (60MB)               ← 断点续传！从第7个分片开始
      ↓
      for (i=6; i<10; i++) {
          PATCH chunk6 (60-70MB)       →     继续写入
          Upload-Offset: 62914560
          progress(60%) → progress(68%)
          ...
          PATCH chunk9 (90-100MB)      →     写入完成
          Upload-Offset: 94371840
          progress(92%) → progress(100%)
      }
      ↓
      virtual.afterSuccess()          →     chunkedUpload.Close() → cache 删除
      UI: "Done" ✓
```

### 6.6 断连恢复的边界情况

**情况1：服务端 cache 已过期（超过24小时）**

HEAD 请求返回 404，前端 catch 后静默忽略，`offset` 保持为 0：

```javascript
// ctrl_upload.js:397-412
try {
    const resp = await executeHttp.call(this, apiURL, { method: "HEAD", ... });
    // ... 解析 offset
} catch (err) {}    // HEAD 失败 → offset=0 → 从头开始
if (offset === 0) {
    // 创建新的 TUS 上传会话，从头上传
    const resp = await executeHttp.call(this, apiURL, { method: "POST", ... });
    uploadURL = resp.headers.location;
}
```

**情况2：普通上传（非 TUS）的断连恢复**

当 `upload_chunk_size` 配置为 0（默认值）或文件小于 chunkSize 时，走普通上传路径：

```javascript
// ctrl_upload.js:374-391
if (chunkSize === 0 || numberOfChunks === 0 || numberOfChunks === 1) {
    try {
        await executeHttp.call(this, apiURL, {
            method: "POST",
            body: file,        // ← 整个文件一次性上传
            progress,
            speed,
        });
        virtual.afterSuccess();
    } catch (err) {
        virtual.afterError();
        if (err === ABORT_ERROR) return;
        throw err;              // ← 失败后无断点续传，重试从头开始
    }
    return;
}
```

**普通上传断连后**：进度丢失，重试从头开始。没有 TUS 的偏移量查询机制，因为整个文件只有一个请求。

**情况3：批量删除/移动的网络断连**

批量删除使用 `rxjs.forkJoin` 并行发起，无队列、无进度条、无重试按钮：

```javascript
// model_files.js:62-69
export const rm = (...paths) => rxjs.forkJoin(paths.map((path) => ajax({
    url: withURLParams(`api/files/rm?path=${encodeURIComponent(path)}`),
    method: "POST",
    responseType: "json",
}))).pipe(
    handleSuccess(...),
    handleError,               // ← 失败只显示通知，无重试机制
);
```

网络断连时，`forkJoin` 中任意一个请求失败就会导致整个 observable 出错，`handleError` 仅弹出一个错误通知。

**情况4：ls 请求的离线降级**

`model_files.js` 中的 `ls` 函数有离线感知逻辑：

```javascript
// model_files.js:116-120
rxjs.merge(
    rxjs.of(navigator.onLine),              // 初始在线状态
    rxjs.fromEvent(window, "online"),       // 监听恢复
    rxjs.fromEvent(window, "offline"),      // 监听断开
)

// model_files.js:113
rxjs.catchError((err) => navigator.onLine ? rxjs.throwError(err) : rxjs.EMPTY),
// 在线时失败=真错误，离线时失败=静默忽略

// model_files.js:127-129
if (navigator.onLine) res["files"] = files;
else res["files"] = files.map((file) => file.type === "file" ? { ...file, offline: true } : file);
// 离线时给文件标记 offline: true，前端显示为不可操作
```

但这是文件列表查询的降级策略，**上传队列模块本身没有在线/离线事件监听**。网络断开时，上传队列只是通过 XHR 的 `onerror` 识别失败并显示重试按钮，需要用户手动触发重试。

### 6.7 总结：断连恢复策略对比

| 操作类型 | 进度推送机制 | 断连检测 | 断点续传 | 重试方式 |
|---------|------------|---------|---------|---------|
| 普通上传 | XHR onprogress | XHR onerror | ❌ 无 | 手动点击重试，从头开始 |
| TUS 分块上传 | XHR onprogress（分片级） | XHR onerror | ✅ HEAD 查偏移量 | 手动点击重试，从断点继续 |
| 批量删除 | 无进度条 | ajax 报错 | ❌ 无 | 仅通知，无重试 |
| 批量移动 | 无进度条 | ajax 报错 | ❌ 无 | 仅通知，无重试 |
| 目录创建 | setInterval 模拟 | XHR onerror | ❌ 无 | 手动点击重试 |
| 文件列表 | 无 | navigator.onLine | N/A | 自动：online 事件触发刷新 |

**核心结论**：本项目的进度推送不依赖 WebSocket/SSE，而是完全基于 XHR 的请求级事件。断连恢复依赖两个机制：(1) **用户手动重试**（点击重试按钮），(2) **TUS 协议的 HEAD 查询**（自动获取已上传偏移量）。没有自动重连和自动重试逻辑。

---

## 7. Worker 异常崩溃时 In-Progress 任务的接管转移路径

### 7.1 崩溃的定义与来源

在当前架构中，"worker 崩溃"有两种含义：

1. **JS 层异常**：`processWorkerQueue` 内部的 `await exec.run(task)` 抛出未预期的异常
2. **浏览器标签页崩溃**：整个页面进程终止（如 OOM、浏览器强制杀掉标签页）

两者的恢复路径截然不同。

### 7.2 JS 层异常：`noFailureAllowed` 自愈循环

**文件**: `public/assets/pages/filespage/ctrl_upload.js:314`

```javascript
const noFailureAllowed = (fn) => fn().catch(() => noFailureAllowed(fn));
```

这个递归函数是 worker 崩溃后的**第一道防线**。它的调用位置：

```javascript
// ctrl_upload.js:322
noFailureAllowed(processWorkerQueue.bind(null, nworker))
    .then(() => reservations[nworker] = false);
```

**完整恢复链路**：

```
processWorkerQueue 内部 await exec.run(task) 抛出异常
    ↓
processWorkerQueue 的 try-catch 捕获，走 "error" 分支
    ↓
updateDOMWithStatus($task, { exec, status: "error", nworker })
    ↓
task.done = true  ←  标记完成（即使失败）
    ↓
while 循环继续 → nextTask(tasks) → 取下一个任务
    ↓
（如果 while 循环自身也崩溃了呢？）
    ↓
noFailureAllowed 捕获 → 递归调用自身 → processWorkerQueue 重新启动
    ↓
reservations[nworker] = false 仅在 processWorkerQueue 正常退出时才执行
    → 如果 noFailureAllowed 递归重启，槽位仍然被占用
    → 重启后的 processWorkerQueue 继续处理 tasks[] 中的剩余任务
```

**关键细节**：`noFailureAllowed` 保证的是**整个 worker 循环不会终止**，但它不能恢复**当前正在执行的那个任务**。

### 7.3 In-Progress 任务的命运：不可转移，只能重试

当一个 task 在 `exec.run(task)` 执行过程中出现异常（如 TUS 分片网络错误），该任务的恢复完全依赖 `exec.retry()` 方法。**没有其他 worker 会接管这个 in-progress 任务**。

原因分析：

**1. 任务已被从全局池中移除**

```javascript
// ctrl_upload.js:305-313
const nextTask = (tasks) => {
    for (let i=0; i<tasks.length; i++) {
        const possibleTask = tasks[i];
        if (!possibleTask.ready()) continue;
        tasks.splice(i, 1);    // ← 任务从 tasks[] 中移除
        return possibleTask;
    }
    return null;
};
```

任务被 `splice` 移出后，只存在于当前 worker 的局部变量 `task` 中，其他 worker 无法看到它。

**2. 执行器是即时创建的，与 worker 绑定**

```javascript
// ctrl_upload.js:282-288
const exec = task.exec({
    progress: (progress) => updateDOMTaskProgress($task, formatPercent(progress)),
    speed: (speed) => {
        updateDOMTaskSpeed($task, speed);
        updateDOMGlobalSpeed(nworker, speed);
    },
});
```

`exec` 实例绑定了特定 worker 的 `nworker` 编号和特定 `$task` DOM 元素。即使其他 worker 想接管，也**无法获取另一个 worker 创建的 `exec` 实例**。

**3. 重试按钮是唯一的恢复入口**

```javascript
// ctrl_upload.js:240-249
$retry.onclick = async() => {
    executeMutation("todo");
    executeMutation("doing");
    try {
        await exec.retry();        // ← 使用 run() 时绑定的 retry 方法
        executeMutation("done");
    } catch (err) {
        executeMutation("error");  // ← 失败可继续重试
    }
};
```

重试执行 `exec.retry()`，而不是把任务放回队列让其他 worker 接管。重试时仍然使用原始 `exec` 实例和原始 `nworker` 绑定的 DOM 更新回调。

### 7.4 In-Progress 任务失败后的完整状态转换

```
任务生命周期（失败路径）：

[nextTask 取出] ─→ task 从 tasks[] 移除
       │
       ↓
[exec = task.exec({...})] ─→ 创建执行器，绑定 nworker 和 DOM
       │
       ↓
[updateDOMWithStatus("doing")] ─→ UI: 显示进度条和停止按钮
       │
       ↓
[await exec.run(task)]
       │
       ├─ 成功 → updateDOMWithStatus("done") → task.done=true → 继续下一任务
       │
       └─ 异常 → updateDOMWithStatus("error")
                    │
                    ├─ UI: 显示 "Error" + 重试按钮
                    ├─ task.done = true ← 解除子任务依赖
                    ├─ updateTotal.incrementCompleted() ← 计数+1
                    └─ 继续下一任务（不阻塞 worker）

[用户点击重试]
       │
       ↓
[exec.retry()] ─→ virtual.before() + executeJob()
       │
       ├─ TUS: HEAD 查偏移量 → 从断点续传
       └─ 普通上传: 从头开始
```

### 7.5 浏览器标签页崩溃：无恢复机制

如果浏览器标签页整个崩溃（OOM、手动杀掉进程等）：

1. **前端状态全部丢失**：`tasks[]`、`reservations[]`、DOM 中的 `exec` 实例全部消失
2. **后端 TUS 会话保留**：`chunkedUploadCache` 中保存着 `io.PipeWriter` 和已上传偏移量，24 小时后自动过期
3. **用户重新打开页面**：上传队列为空，不会自动恢复之前未完成的任务
4. **TUS 缓存的后果**：后端的 `io.PipeWriter` 永远不会有写入者，后端的 `io.PipeReader` 永远不会有读取者。`save` goroutine 阻塞在 `io.Copy`，直到 cache 过期触发 `OnEvict` → `Close()` 关闭 Pipe

**后端 Pipe 泄露的防护**：

```go
// server/ctrl/files.go:688-700
func initChunkedUploader() {
    chunkedUploadCache = NewAppCache(60*24, 1)   // 24小时过期，每1分钟清理
    chunkedUploadCache.OnEvict(func(key string, value interface{}) {
        c := value.(*chunkedUpload)
        if c == nil { return }
        if err := c.Close(); err != nil {         // Close() 关闭 PipeWriter
            Log.Warning("ctrl::files::chunked::cleanup action=close err=%s", err.Error())
            return
        }
    })
}
```

```go
// server/ctrl/files.go:721-728
func (this *chunkedUpload) Close() error {
    this.stream.Close()      // ← 关闭 PipeWriter
    err := <-this.done       // ← 等待 save goroutine 返回
    this.once.Do(func() {
        close(this.done)
    })
    return err
}
```

当 PipeWriter 被关闭后，`io.Copy(stream, r)` 返回 `io.ErrClosedPipe`，save goroutine 退出，资源释放。

### 7.6 总结：In-Progress 任务接管路径对比

| 崩溃类型 | In-Progress 任务是否可恢复 | 恢复机制 | 数据损失 |
|---------|:----------------------:|---------|---------|
| exec.run() 网络异常 | ✅ | 重试按钮 → exec.retry() → TUS 断点续传 | 无（TUS）/ 从头（普通） |
| exec.run() 未知异常 | ✅ | 重试按钮 → exec.retry() | 可能丢进度 |
| processWorkerQueue 循环崩溃 | ✅ | noFailureAllowed 自动重启 worker | 当前任务重试按钮恢复 |
| DOM 异常导致 UI 不一致 | ⚠️ | 无自动恢复，任务可能卡在 doing 状态 | 进度丢失 |
| 浏览器标签页崩溃 | ❌ | 无前端恢复，后端 TUS cache 24h 后自动清理 | 全部丢失 |

---

## 8. 跨用户共享 Worker 池场景下的优先级调度

### 8.1 架构前提：Worker 池的隔离边界

**关键结论：本项目不存在跨用户共享的 Worker 池。**

Worker 池完全运行在浏览器前端，每个标签页有自己独立的 JS 运行时。需要区分三层隔离：

```
┌─────────────────────────────────────────────────────┐
│  服务器（Go 进程）                                     │
│                                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │
│  │ User A 请求  │  │ User B 请求  │  │ User C 请求  │  │
│  │ (session_1) │  │ (session_2) │  │ (session_3) │  │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  │
│         │                │                │          │
│         ↓                ↓                ↓          │
│  ┌──────────────────────────────────────────────┐   │
│  │        chunkedUploadCache（全局共享）           │   │
│  │  key: {path, session} → chunkedUpload        │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│  浏览器 User A                                        │
│  ┌──────────────────────────────────────────────┐   │
│  │  MAX_WORKERS=4 的独立 Worker 池               │   │
│  │  tasks[] = [A1, A2, A3, ...]                 │   │
│  │  reservations = [true, true, false, false]   │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│  浏览器 User B                                        │
│  ┌──────────────────────────────────────────────┐   │
│  │  MAX_WORKERS=4 的独立 Worker 池               │   │
│  │  tasks[] = [B1, B2, ...]                     │   │
│  │  reservations = [true, false, false, false]  │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

### 8.2 前端 Worker 池的隔离性分析

**workers$ 是模块级单例**：

```javascript
// ctrl_upload.js:16
const workers$ = new rxjs.BehaviorSubject({ tasks: [], size: null });
```

`workers$` 定义在模块顶层，但每个浏览器标签页有独立的 JS 运行时，因此：

- **同一用户同一标签页**：共享同一个 `workers$`，同一个 `tasks[]`
- **同一用户不同标签页**：各自有独立的 `workers$`，互不干扰
- **不同用户**：各自有独立的 `workers$`，完全隔离

**同标签页内的多入口共享**：

```javascript
// ctrl_upload.js:20-38
export default async function(render) {
    if (!document.querySelector(`[is="component_upload_queue"]`)) {
        const $queue = createElement(`<div is="component_upload_queue"></div>`);
        document.body.appendChild($queue);
        componentUploadQueue(createRender($queue), { workers$ });
    }

    effect(getPermission().pipe(
        rxjs.filter(() => calculatePermission(currentPath(), "upload")),
        rxjs.tap(() => {
            componentFilezone(createRender(...), { workers$ });
            componentUploadFAB(createRender(...), { workers$ });
        }),
    ));
}
```

上传队列组件（`componentUploadQueue`）在 DOM 中以 `component_upload_queue` 标记做单例检查，确保同一标签页只有一个队列实例。文件拖放区（`componentFilezone`）和 FAB 按钮（`componentUploadFAB`）都通过同一个 `workers$` 注入任务。

### 8.3 后端资源的跨用户竞争

虽然前端 Worker 池是隔离的，但后端的 `chunkedUploadCache` 是**进程级全局共享**的：

```go
// server/ctrl/files.go:477
var chunkedUploadCache AppCache
```

每个用户的 TUS 上传会话通过 cache key 隔离：

```go
// server/ctrl/files.go:543-546
cacheKey := map[string]string{
    "path":    path,
    "session": GenerateID(ctx.Session),   // ← session 隔离
}
```

`GenerateID` 基于 session 中的关键字段（type、host、user 等）生成唯一标识：

```go
// server/common/crypto.go:193-214
func GenerateID(params map[string]string) string {
    p := ""
    orderedKeys := make([]string, len(params))
    for key, _ := range params {
        orderedKeys = append(orderedKeys, key)
    }
    sort.Strings(orderedKeys)
    for _, key := range orderedKeys {
        switch key {
        case "password":    // 敏感字段不参与
        case "path":
        case "session":
        case "timestamp":
        default:
            if val := params[key]; val != "" {
                p += key + "=>" + params[key] + ", "
            }
        }
    }
    if p == "" { return "na" }
    return Hash(p, 20)
}
```

**跨用户竞争的关键点**：

1. **cache 容量无上限**：`NewAppCache(60*24, 1)` 只设了过期时间（24小时），没有容量限制。多用户并发上传时，cache 会无限增长直到内存耗尽
2. **无并发控制**：后端对同时进行的上传请求数没有信号量或速率限制
3. **无优先级**：所有请求按 HTTP 到达顺序处理，无优先级队列

### 8.4 跨标签页的协同：BroadcastChannel

项目中唯一的跨标签页通信机制是 `BroadcastChannel`：

```javascript
// model_files.js:82
const bc = new BroadcastChannel("filestash::ls::refresh");

// model_files.js:157
export const refresh = () => {
    window.dispatchEvent(new KeyboardEvent("keydown", { keyCode: 82 }));
    bc.postMessage(null);     // ← 通知其他标签页刷新文件列表
};
```

```javascript
// model_files.js:110
rxjs.fromEvent(bc, "message"),
```

**但 BroadcastChannel 不涉及上传队列**。它仅用于文件列表刷新的跨标签页同步。上传队列的 `workers$` 是完全独立的，同一用户在两个标签页上传文件会创建两个独立的 Worker 池。

### 8.5 实际的"跨用户调度"场景分析

尽管没有显式的跨用户 Worker 池，但以下场景等效于跨用户资源竞争：

**场景1：同一用户开多个标签页上传**

```
标签页 A: MAX_WORKERS=4, 上传 20 个文件
标签页 B: MAX_WORKERS=4, 上传 10 个文件
──────────────────────────────────────
浏览器对同一域名 HTTP 并发连接数限制 ≈ 6
实际并发 ≈ 6（不是 4+4=8）
两个标签页互相争抢连接，彼此降低吞吐量
```

浏览器层面，HTTP/1.1 对同一域名的并发连接数限制约为 6 个（Chrome 默认）。两个标签页各自有 4 个 worker，但浏览器总共只允许 6 个并发连接。这导致了隐式的"跨标签页调度"，由浏览器的连接管理器控制，无优先级可言。

**场景2：多用户同时上传到同一后端**

```
User A: 4 个并发上传请求 → 后端同步处理 → 存储后端 (S3/SFTP/...)
User B: 4 个并发上传请求 → 后端同步处理 → 存储后端 (S3/SFTP/...)
User C: 4 个并发上传请求 → 后端同步处理 → 存储后端 (S3/SFTP/...)
───────────────────────────────────────────────────────────
后端无并发控制，所有请求竞争存储后端 I/O
可能导致：存储后端 API 限流、连接池耗尽、内存压力
```

后端 `FileSave` 对请求的处理是同步阻塞的：

```go
// server/ctrl/files.go:530-539
if proto == "" && req.Method == http.MethodPost {
    err = ctx.Backend.Save(path, req.Body)   // ← 同步阻塞，直到写完
    req.Body.Close()
    // ...
    SendSuccessResult(res, nil)
    return
}
```

每个上传请求都会占用一个 Go HTTP handler goroutine，直到 `ctx.Backend.Save()` 返回。如果存储后端慢（如 S3 高延迟），大量并发请求会积累大量 goroutine。

**场景3：TUS 分块上传的全局资源竞争**

```go
// server/ctrl/files.go:672-684
func createChunkedUploader(save func(path string, file io.Reader) error, path string, size uint64) *chunkedUpload {
    r, w := io.Pipe()
    done := make(chan error, 1)
    go func() {
        done <- save(path, r)     // ← 每个 TUS 上传占一个 goroutine
    }()
    // ...
}
```

每个 TUS 上传会话占用：
- 1 个 `chunkedUpload` 结构体（内存）
- 1 个 `io.Pipe`（读写两端）
- 1 个 goroutine（执行 `save`）

这些资源在 cache 过期前一直占用，没有按用户配额回收。

### 8.6 调度优先级的现状

**前端**：FIFO（先进先出），无优先级

```javascript
// ctrl_upload.js:305-313
const nextTask = (tasks) => {
    for (let i=0; i<tasks.length; i++) {    // ← 按数组顺序遍历
        const possibleTask = tasks[i];
        if (!possibleTask.ready()) continue;
        tasks.splice(i, 1);
        return possibleTask;
    }
    return null;
};
```

任务严格按入队顺序（数组下标）调度，唯一的"优先级"来自 `ready()` 函数——目录创建任务天然比其子文件任务优先（因为子文件 `ready()` 返回 false 直到目录 `done`）。

**后端**：无调度，HTTP 请求按到达顺序处理

Go 的 `net/http` 包使用 goroutine-per-connection 模型，每个请求独立处理，没有全局队列或优先级。

### 8.7 跨用户调度的改进方向（代码层面的差距）

如果要实现跨用户优先级调度，需要以下改动，当前代码**均不存在**：

| 改动点 | 当前状态 | 所需改动 |
|-------|---------|---------|
| 前端任务优先级字段 | task 对象无 priority 字段 | 在 processFiles/processItems 中添加 `task.priority = 0` |
| nextTask 优先级排序 | 按数组下标 FIFO | 改为按 priority 排序后再遍历 |
| 后端请求速率限制 | 无 | 引入 `rate.Limiter` 或信号量 |
| 后端用户配额 | 无 per-user 限制 | 在 middleware 中实现 per-session 并发限制 |
| chunkedUploadCache 容量 | 无上限 | 添加 `cache.MaxItems()` 或 per-session 计数 |
| 跨标签页队列协调 | 无 | 使用 SharedWorker 或 BroadcastChannel 同步上传状态 |
| 后端优先级队列 | 无 | TUS 会话按用户优先级排队处理 |

**最接近"跨用户调度"的现有机制**：`chunkedUploadCache` 的 per-session 隔离（通过 `GenerateID(ctx.Session)` 作为 cache key），确保不同用户的 TUS 会话不会互相干扰——但这只是**隔离**，不是**调度**。

### 8.8 完整的调度层级总结

```
第1层：浏览器 HTTP 连接池
    └─ 同一域名最多 6 个并发连接（HTTP/1.1）
    └─ 无优先级，先到先得
    └─ 影响：同一用户多标签页上传时互相争抢

第2层：前端 Worker 池（per 标签页）
    └─ MAX_WORKERS = 4 个并发 worker
    └─ FIFO 调度（nextTask 按数组顺序）
    └─ 依赖感知：目录 → 子文件（隐式优先级）
    └─ 完全隔离：标签页之间无共享

第3层：后端 HTTP 处理（Go net/http）
    └─ goroutine-per-request
    └─ 无并发限制、无速率限制
    └─ 无优先级，同步阻塞处理

第4层：后端 TUS 会话缓存（全局共享）
    └─ chunkedUploadCache：per-session 隔离
    └─ 24 小时过期，无容量限制
    └─ 每个 TUS 会话占一个 goroutine + Pipe

第5层：存储后端（S3/SFTP/Samba/...）
    └─ 各自的连接池和速率限制
    └─ 最慢的瓶颈层
```
