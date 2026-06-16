# Filestash 搜索流水线（Search Pipeline）

本文档按代码执行顺序梳理 Filestash 跨多存储后端的文件搜索机制，覆盖三个核心问题：
1. 插件如何上报可搜内容（索引构建）
2. 查询如何拆分到不同 backend
3. 结果如何合并、去重、排序

---

## 0. 核心接口与插件注册

### 0.1 ISearch 接口契约

所有搜索引擎必须实现 `ISearch` 接口（定义在 `server/common/types.go:48-50`）：

```go
type ISearch interface {
    Query(ctx App, basePath string, term string) ([]IFile, error)
}
```

返回值 `[]IFile` 中的 `IFile` 是扩展了 `Path()` 方法的 `os.FileInfo`：

```go
type IFile interface {
    os.FileInfo
    Path() string
}
```

### 0.2 搜索插件注册机制

所有插件通过 `server/plugin/index.go` 的空导入触发 `init()`。搜索插件使用 Hook 注册：

```go
// server/common/plugin.go:158-166
var search ISearch

func (this Register) SearchEngine(s ISearch) {
    search = s
}
func (this Get) SearchEngine() ISearch {
    return search
}
```

> **注意**：全局只有一个 `search` 变量，后注册的插件会覆盖先注册的。
> `server/plugin/index.go` 中默认导入了 `plg_search_stateless`；`plg_search_sqlitefts` 需单独启用。

### 0.3 Backend 注册机制

所有存储后端通过全局 `Backend` Driver 注册（`server/common/backend.go:11-38`）：

```go
var Backend = NewDriver()  // Driver{ds: map[string]IBackend{}}

// 在每个 plg_backend_* 的 init() 中
Backend.Register("local", &Local{...})
Backend.Register("s3", &S3{...})
Backend.Register("sftp", &SFTP{...})
// ... 20+ 种 backend
```

`IBackend` 接口定义了文件系统的核心操作：`Init / Ls / Stat / Cat / Mkdir / Rm / Mv / Save / Touch`。

---

## 1. HTTP 请求入口与 Session-Backend 绑定

### 1.1 路由定义

搜索 API 路由在 `server/routes.go:68`：

```go
files.HandleFunc("/search", NewMiddlewareChain(FileSearch, middlewares)).Methods("GET")
```

中间件链：`ApiHeaders → SecureHeaders → SecureOrigin → SessionStart → LoggedInOnly → PluginInjector`

### 1.2 SessionStart 建立 App 上下文（关键）

`server/middleware/session.go:57-83` 的 `SessionStart` 是请求进入业务逻辑前的关键步骤：

```
_extractShare()       → 解析共享链接
_extractAuthorization → 从 cookie/Authorization header 取 token
_extractSession()     → 解密 token，得到 session map（含 type, hostname, path 等）
_extractBackend()     → 根据 session["type"] 初始化具体 IBackend 实例
```

`_extractBackend()` 调用 `model.NewBackend(ctx, session)`（`server/model/files.go:9-51`）：

1. `isAllowed()` 检查 session 参数是否匹配管理员在 `Config.Conn` 中配置的允许连接
2. `Backend.Get(conn["type"])` 从全局 Driver 取出对应后端 driver
3. 调用 `driver.Init(conn, ctx)` 返回初始化好的 `IBackend` 实例

> **重要结论**：每个 HTTP 请求的 `App.Backend` 是单一实例，由用户登录时的 session 决定。
> **不存在单请求跨多 backend 并行查询**。"跨多存储后端"体现在：不同用户/会话绑定不同 backend，每个 backend 有独立的搜索索引。

### 1.3 ctrl.FileSearch 控制器

`server/ctrl/search.go:10-54`：

```go
func FileSearch(ctx *App, res http.ResponseWriter, req *http.Request) {
    path := ...    // 从 query 参数取，默认 "/"
    q := ...       // 搜索关键词
    if model.CanRead(ctx) == false { ... }

    searchEngine := Hooks.Get.SearchEngine()  // 取注册的 ISearch 实现
    searchResults, err = searchEngine.Query(*ctx, path, q)

    // 如果配置了 chroot，修正返回路径
    if ctx.Session["path"] != "" {
        for i := 0; i < len(searchResults); i++ {
            searchResults[i] = File{
                FName: searchResults[i].Name(),
                FSize: searchResults[i].Size(),
                FType: ...,
                FPath: "/" + strings.TrimPrefix(searchResults[i].Path(), ctx.Session["path"]),
            }
        }
    }
    SendSuccessResults(res, searchResults)
}
```

---

## 2. 插件如何上报可搜内容（索引构建）

Filestash 内置两种搜索引擎，索引构建方式不同。

### 2.1 plg_search_stateless（无状态搜索）

**无索引**。每次查询实时遍历目录树，不持久化任何数据。因此不存在"上报可搜内容"这一步。

### 2.2 plg_search_sqlitefts（SQLite 全文搜索）

这是重点。它通过"文件操作钩子 + 后台爬虫 + SQLite FTS5"三件套构建索引。

#### 2.2.1 插件初始化

`server/plugin/plg_search_sqlitefts/index.go:12-35`：

```go
func init() {
    Hooks.Register.Onload(register)
}

func register() {
    enabled := config.SEARCH_ENABLE()
    daemon := crawler.NewDaemon(enabled)

    // 注册搜索引擎
    Hooks.Register.SearchEngine(SearchEngine{Daemon: &daemon})
    // 注册授权中间件（用于监听文件操作 → 触发索引更新）
    Hooks.Register.AuthorisationMiddleware(crawler.FileHook{Daemon: &daemon})
    // 注册工作流动作（管理员可手动触发索引）
    Hooks.Register.WorkflowAction(workflow.StepIndexer{Daemon: &daemon})

    // 启动 N 个后台爬虫 goroutine
    if enabled {
        for i := 0; i < config.SEARCH_PROCESS_PAR(); i++ {
            go func() {
                for {
                    crwlr := daemon.NextCrawler()  // 轮询取下一个 crawler
                    if crwlr == nil { time.Sleep(5 * time.Second); continue }
                    crwlr.Run()
                }
            }()
        }
    }
}
```

#### 2.2.2 文件操作钩子（FileHook）——内容上报入口

`server/plugin/plg_search_sqlitefts/crawler/events.go`：

`FileHook` 实现了 `IAuthorisation` 接口，作为授权中间件链的一环。每次用户执行文件操作时，它**异步**向 Daemon 发送 Hint：

| 用户操作 | Hint 调用 | 作用 |
|---------|----------|------|
| `Ls(path)` | `go Daemon.HintLs(ctx, path)` | 把该目录加入待探索队列 |
| `Cat(path)` | `go Daemon.HintLs(ctx, dir(path))` | 更新文件所在目录 |
| `Mkdir(path)` | `go Daemon.HintLs(ctx, dir(path))` + `HintLs(ctx, path)` | 新目录需要被索引 |
| `Rm(path)` | `go Daemon.HintRm(ctx, path)` | 从索引中删除该路径 |
| `Mv(from, to)` | `HintRm(from)` + `HintLs(to)` + `HintLs(dir(to))` | 旧路径删、新路径加 |
| `Save(path)` / `Touch(path)` | `HintLs(dir(path))` + `HintFile(path)` | 文件需要重新提取内容 |

`FileHook.record(ctx)` 会检查 `ctx.Context.Value("AUDIT")` 标志，避免对内部请求重复索引。

#### 2.2.3 Daemon 管理多个 Crawler（多 backend 隔离）

`server/plugin/plg_search_sqlitefts/crawler/daemon.go`：

```go
type daemonState struct {
    discovery bool
    idx       []Crawler   // 每个 session/backend 对应一个 Crawler
    n         int         // 轮询游标
    mu        sync.RWMutex
}

type Crawler struct {
    Id             string        // = GenerateID(session)，按用户会话隔离
    FoldersUnknown HeapDoc       // 待探索目录的优先队列
    CurrentPhase   string        // EXPLORE / INDEXING / MAINTAIN / PAUSE
    Backend        IBackend      // 该 Crawler 绑定的具体存储后端
    State          indexer.Index // SQLite 索引实例（每个 backend 一个 .sql 文件）
    mu             sync.Mutex
}
```

**Daemon 如何管理多 backend**：

1. `daemonState.HintLs(app, path)` 被调用时，先用 `GenerateID(app.Session)` 查找是否已有对应 Crawler
2. 若不存在且 `SEARCH_PROCESS_MAX` 未满，则：
   - 调用 `app.Backend.Init(app.Session, app)` 新建独立 backend 实例（避免占用用户请求连接）
   - 创建 `Crawler` + `indexer.NewIndex(idpath)`（SQLite 文件：`fts_<id>.sql` 或共享 `fts.sql`）
   - 把 `path` 推入 `FoldersUnknown` 优先队列
3. 后台 goroutine 通过 `daemonState.NextCrawler()` 轮询（round-robin）所有活跃 Crawler，依次调用 `crwlr.Run()`

#### 2.2.4 Crawler 四阶段状态机

`server/plugin/plg_search_sqlitefts/crawler/phase.go:60-70`：

```
EXPLORE → INDEXING → MAINTAIN → PAUSE → EXPLORE ...
```

每个阶段执行一个时间片（`CYCLE_TIME` 秒），时间到就切换到下一阶段，避免某个 backend 长期占用。

**阶段 1：EXPLORE（探索目录树）**
`crawler/phase_explore.go:15-103`：
1. `DiscoverPop()` 从 `FoldersUnknown` 堆顶弹出优先级最高的目录
2. 调 `this.Backend.Ls(doc.Path)` 拉取该目录内容
3. 对每个条目：
   - 目录：`dbInsert()` 写入 `file` 表 → 推入 `FoldersUnknown` 继续探索
   - 文件：`dbUpsert()` 写入 `file` 表（元数据，不含内容）
4. 对比数据库中该目录的旧条目，删除已不存在的文件

**阶段 2：INDEXING（提取文件内容）**
`crawler/phase_indexing.go:11-37`：
1. `tx.FindNew(maxSize, extensions)` 查询待索引文件：`type='file' AND size < MAX AND filetype IN (...) AND indexTime IS NULL`
2. 对每个文件：`updateFile(path, backend, tx)`（`crawler/phase_utils.go:14-33`）
   - `backend.Cat(path)` 取文件流
   - `converter.Convert(path, reader)` 转成纯文本（pdf/office/图片等）
   - `tx.FileContentUpdate(path, text)` 写入 `file_index` FTS5 虚拟表
   - 更新 `indexTime` 标记已索引

**阶段 3：MAINTAIN（增量维护）**
`crawler/phase_maintain.go`：检查旧索引文件，对比 backend 实际状态，做增量同步。

**阶段 4：PAUSE（暂停）**
`crawler/phase_pause.go`：休眠一个时间片，让出资源。

#### 2.2.5 SQLite 索引结构

`server/plugin/plg_search_sqlitefts/indexer/index.go:47-95`：

```sql
-- 文件元数据表
CREATE TABLE file(
    path VARCHAR(1024) PRIMARY KEY,
    filename VARCHAR(64),
    filetype VARCHAR(16),
    type VARCHAR(16),         -- 'file' or 'directory'
    parent VARCHAR(1024),
    size INTEGER,
    modTime timestamp,
    indexTime timestamp       -- NULL = 待索引内容
);

-- FTS5 全文索引虚拟表
CREATE VIRTUAL TABLE file_index USING fts5(
    path UNINDEXED,
    filename,
    filetype,
    content,
    tokenize = 'porter'
);

-- 触发器：file 表增删改自动同步 file_index 的元数据字段
CREATE TRIGGER after_file_insert ...
CREATE TRIGGER after_file_delete ...
CREATE TRIGGER after_file_update_path ...
```

---

## 3. 查询如何拆分到不同 backend

### 3.1 结论先行

**单请求只查一个 backend，无"拆分"动作。**

每个请求的 `App.Backend` 由 `SessionStart` 根据用户 session 确定，搜索请求只在这一个 backend 上执行。

"跨多存储后端"在搜索语境下的真实含义：
- 不同用户会话 → 不同 backend → 各自独立的 Crawler 和 SQLite 索引文件
- sqlitefts 的 `daemonState.idx[]` 数组同时维护多个 backend 的 Crawler，但单次 `Query()` 只命中其中一个

### 3.2 stateless 查询路径

`server/plugin/plg_search_stateless/index.go:21-99`：

```go
func (this StatelessSearch) Query(app App, path string, keyword string) ([]IFile, error) {
    toVisit := []PathQuandidate{{path, 0}}  // 待访问目录队列（按 Score 排序）

    for start := time.Now(); time.Since(start) < MAX_SEARCH_TIME; {
        currentPath := toVisit[0]; toVisit = toVisit[1:]

        // 直接调当前 backend 的 Ls 接口
        f, err := app.Backend.Ls(currentPath.Path)

        for _, file := range f {
            // 文件名模式匹配
            if IsSearchQueryMatchingFilename(lower(name), lower(keyword)) {
                files = append(files, File{...})
            }
            // 子目录加入待访问队列，按 Score 插入（保持有序）
            if file.IsDir() {
                score = scoreBoostForPath(...) + scoreBoostForFilesInDirectory(...)
                      + scoreBoostOnDepth(...) + currentPath.Score
                insertSorted(toVisit, PathQuandidate{fullpath, score})
            }
        }
    }
    return files, nil
}
```

特点：
- 直接复用请求携带的 `app.Backend`
- BFS 但用 Score 优先队列替代普通队列，优先探索"更可能有结果"的目录
- 受 `SEARCH_TIMEOUT` 时间限制，到时直接返回已找到的结果

### 3.3 sqlitefts 查询路径

`server/plugin/plg_search_sqlitefts/query.go:19-31`：

```go
func (this SearchEngine) Query(app App, path string, keyword string) ([]IFile, error) {
    // 1. 按 session id 找到对应 Crawler（不存在则创建）
    crwlr, err := this.Daemon.GetCrawler(&app, true)

    // 2. 把当前 path 加入探索队列，提示后台索引该路径下的新内容
    heap.Push(&crwlr.FoldersUnknown, &Document{
        Type: "directory", Path: path, InitialPath: path, Name: filepath.Base(path),
    })

    // 3. 在该 backend 对应的 SQLite 索引中查询
    return crwlr.State.Search(path, keyword)
}
```

`indexer/query.go:11-40` 执行实际 SQL：

```sql
SELECT type, path, size, modTime
FROM file
WHERE path IN (
    SELECT path FROM file_index
    WHERE file_index MATCH ?           -- FTS5 全文匹配
      AND path > ? AND path < ?        -- 路径前缀范围：[path, path~)
    ORDER BY rank LIMIT 50000          -- FTS5 自带 rank 打分
)
```

关键词预处理：`regexp.MustCompile(`(\.|\-)`).ReplaceAllString(q, "\"$1\"")`
把 `.` 和 `-` 用引号包起来，避免被 FTS5 当作分词符。

---

## 4. 结果如何合并、去重、排序

### 4.1 stateless 引擎

| 环节 | 实现 | 说明 |
|------|------|------|
| **合并** | 无 | 单 backend，单一遍历过程 |
| **去重** | 隐式去重 | 路径作为天然唯一键；优先队列保证每个目录只被探索一次 |
| **排序** | 结果顺序 = 目录探索顺序 | 目录探索优先级由 `PathQuandidate.Score` 决定，评分规则见 `scoring.go` |

`scoring.go` 中的评分规则：
- `scoreBoostForPath(p)`：路径基名含 `document/project/home/note` 加 3 分；`node_modules` 减 100；点文件减 10
- `scoreBoostForFilesInDirectory(files)`：目录内含 `.org`(+2)、`.pdf/.doc/.docx/.md`(+1)，封顶 4 分
- `scoreBoostOnDepth(p)`：`-层级深度`，越深分越低

### 4.2 sqlitefts 引擎

| 环节 | 实现 | 说明 |
|------|------|------|
| **合并** | 无 | 单 backend，单条 SQL 查询 |
| **去重** | `file.path` 主键 | SQLite PRIMARY KEY 保证唯一；`ON CONFLICT` upsert 避免重复 |
| **排序** | FTS5 `rank` 排序 | SQL 中 `ORDER BY rank LIMIT 50000`，rank 由 FTS5 BM25 算法自动计算 |

### 4.3 通用后处理：chroot 路径修正

`server/ctrl/search.go:35-52`：

若用户 session 配置了 `path`（即被 chroot 到子目录），遍历所有结果：
```go
FPath: "/" + strings.TrimPrefix(searchResults[i].Path(), ctx.Session["path"])
```
把绝对路径裁剪为相对于 chroot 根的路径，避免用户感知上层目录结构。

---

## 5. 完整时序图（按代码执行顺序）

```
用户浏览器
    │
    │ GET /api/files/search?path=/docs&q=hello
    ▼
gorilla/mux 路由 (routes.go:68)
    │
    ▼
中间件链执行
    ├─ ApiHeaders
    ├─ SecureHeaders
    ├─ SecureOrigin
    ├─ SessionStart (middleware/session.go:57)
    │     ├─ _extractAuthorization() → token
    │     ├─ _extractSession(token)  → session{type:"s3", hostname:"...", ...}
    │     └─ _extractBackend() → model.NewBackend() → Backend.Get("s3").Init() → IBackend 实例
    ├─ LoggedInOnly
    └─ PluginInjector
          └─ FileHook.Ls/Rm/... 等授权中间件（仅 sqlitefts，用于 Hint，不阻塞搜索）
    │
    ▼
ctrl.FileSearch (ctrl/search.go:10)
    ├─ Hooks.Get.SearchEngine() → ISearch 实例
    ├─ searchEngine.Query(ctx, "/docs", "hello")
    │     │
    │     ├─ [stateless] app.Backend.Ls() BFS + Score 排序 + 文件名匹配
    │     │         └─ return []IFile
    │     │
    │     └─ [sqlitefts] Daemon.GetCrawler(app) → Crawler{Backend, State(fts_xxx.sql)}
    │               ├─ heap.Push(FoldersUnknown, "/docs") （提示后台更新）
    │               └─ State.Search() → SQLite FTS5 MATCH query
    │                     └─ return []IFile (ORDER BY rank)
    │
    ├─ 若有 chroot：修正 FPath
    └─ SendSuccessResults(res, results)
          │
          ▼
用户浏览器（JSON 响应）
```

---

## 6. 关键文件速查表

| 文件 | 作用 |
|------|------|
| `server/common/types.go:43-50` | `IFile`、`ISearch` 接口定义 |
| `server/common/plugin.go:152-166` | SearchEngine Hook 注册/获取 |
| `server/common/backend.go` | 全局 Backend Driver，存储后端注册表 |
| `server/middleware/session.go:57-83` | SessionStart，建立 App.Backend 绑定 |
| `server/model/files.go:9-51` | `NewBackend()`，按 session 初始化具体 backend |
| `server/ctrl/search.go` | HTTP 搜索控制器 |
| `server/plugin/plg_search_stateless/index.go` | 无状态搜索实现 |
| `server/plugin/plg_search_stateless/scoring.go` | 目录探索评分 + 文件名匹配算法 |
| `server/plugin/plg_search_sqlitefts/index.go` | sqlitefts 插件入口，注册 Hook，启动后台爬虫 |
| `server/plugin/plg_search_sqlitefts/crawler/daemon.go` | Daemon 多 Crawler（多 backend）管理 |
| `server/plugin/plg_search_sqlitefts/crawler/events.go` | FileHook 文件操作监听，触发 Hint |
| `server/plugin/plg_search_sqlitefts/crawler/phase.go` | 四阶段状态机调度 |
| `server/plugin/plg_search_sqlitefts/crawler/phase_explore.go` | EXPLORE 阶段：遍历目录树 |
| `server/plugin/plg_search_sqlitefts/crawler/phase_indexing.go` | INDEXING 阶段：提取文件内容 |
| `server/plugin/plg_search_sqlitefts/crawler/phase_utils.go` | `updateFile()` 内容转换与入库 |
| `server/plugin/plg_search_sqlitefts/indexer/index.go` | SQLite 表结构 + Manager 接口 |
| `server/plugin/plg_search_sqlitefts/indexer/query.go` | FTS5 查询 SQL |
| `server/routes.go:68` | `/api/files/search` 路由注册 |
| `public/assets/pages/filespage/model_files.js:160-165` | 前端 `search()` 函数，发起搜索请求 |
