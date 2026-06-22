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
