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

---

## 7. 异步索引机制（Searchable Indexing）详解

### 7.1 触发路径一：用户操作驱动（被动索引（FileHook

| 触发源 | 代码位置 | 调用链 |
|--------|---------|-------|
| 用户浏览目录 | `server/ctrl/files.go:99-105` | `FileLs` → `for auth := range Hooks.Get.AuthorisationMiddleware() { auth.Ls(ctx, path) }` → `FileHook.Ls` → `go Daemon.HintLs`

### 7.2 触发路径二：工作流触发（主动索引

`server/plugin/plg_search_sqlitefts/workflow/index.go` 定义了 `StepIndexer` 工作流动作，管理员可在后台配置定时/手动触发全量索引：

```go
func (this StepIndexer) Execute(params, input) (map[string]string, error) {
    // 1. 解密 token → session
    // 2. NewBackend(app, session) 建立独立 backend
    // 3. GetCrawler(app, true) 取/建 Crawler
    // 4. heap.Push(&crwlr.FoldersUnknown, 根路径)
    // 5. 启动 N 个 goroutine 并发：DiscoverPop → Backend.Ls → DiscoverPush，直到 FoldersUnknown 清空
    //    用 sync.Cond 协调 inflight 计数
}
```

工作流路径与后台常驻后台爬虫的区别：工作流是"跑完整个目录树，不按时间片轮转，直到全部索引所有内容；常驻爬虫按时间片切分（CYCLE_TIME 秒）轮换阶段，避免长时间占用。

### 7.3 触发路径三：搜索时惰性触发

`server/plugin/plg_search_sqlitefts/query.go:24-29`

用户搜索时，把当前搜索 `path` 也推入探索队列：

```go
heap.Push(&crwlr.FoldersUnknown, &Document{Path: path, ...})
```

这是一种"搜索即索引"的惰性策略：用户搜什么路径，后台下次时间片就优先爬什么路径。

### 7.4 目录探索优先队列（HeapDoc）排序规则

`server/plugin/plg_search_sqlitefts/crawler/types.go:30-61`

```go
type HeapDoc []*Document

func (h HeapDoc) Less(i, j int) bool {
    if h[i].Priority != 0 || h[j].Priority != 0 {
        return h[i].Priority < h[j].Priority  // 显式 Priority 优先
    }
    // 否则按"距 InitialPath 的相对深度越浅越优先
    scoreA := len(strings.Split(h[i].Path, "/")) / len(strings.Split(h[i].InitialPath, "/"))
    scoreB := len(strings.Split(h[j].Path, "/")) / len(strings.Split(h[j].InitialPath, "/"))
    return scoreA < scoreB
}
```

优先队列上限：`MAX_HEAP_SIZE = 100000`，超过则静默丢弃。

---

## 8. 跨 backend 联合 query

### 8.1 真实的"跨多 backend"的真实含义

代码里**没有单请求内并发查询多个 backend**机制。每个 HTTP 请求绑定一个 backend（由 session 决定。

**多 backend 共存体现在：

1. **多用户多 session 场景**：每个用户 session 不同 → `GenerateID(session) 不同 → 不同 Crawler → 不同 SQLite 文件
2. **同一用户多连接场景**：同一用户分别登录到不同存储类型不同 session["type"] 不同 → 不同 Crawler
3. **共享索引模式**：`SEARCH_SHARED_INDEX=true` 时所有用户共用同一个 `fts.sql`，但仍按 session 区分 backend

### 8.2 Crawler ID 生成算法

`server/common/crypto.go:193-218` 的 `GenerateID(session)`：

```go
func GenerateID(params map[string]string) string {
    orderedKeys := sort(所有 session key)
    for key := range orderedKeys {
        switch key {
        case "password", "path", "session", "timestamp":
            // 排除这些字段
        default:
            p += key + "=>" + params[key] + ", "
        }
    }
    p += "salt=>" + SECRET_KEY
    return Hash(p, 20)
}
```

**同路径不同 backend 的语义由 ID 由 session 中除 password/path/session/timestamp 之外所有字段（包括 type/hostname/url 等）联合哈希生成。两个 session 只要连接参数（不同 → 不同 ID → 不同 Crawler/索引。

---

## 9. 同路径不同 backend 的语义

### 9.1 路径是**相对 backend 根的逻辑路径

```
backend 根可能完全不同物理存储上：
- backend = S3：`/docs/report.pdf → S3 bucket 内的路径
- backend = local：`/docs/report.pdf → 本地文件系统路径

### 9.2 chroot 路径修正机制

搜索时路径处理：

1. **入参**：`ctrl.PathBuilder(ctx, req.URL.Query().Get("path")`（`server/ctrl/files.go:1104-1117`

```go
basePath := filepath.Join(session["path"], userQueryPath))
if strings.HasPrefix(basePath, session["path"]) == false {
    return ErrFilesystemError  // 禁止跳出 chroot
}
```

2. **出参**：`FPath: "/" + strings.TrimPrefix(原始绝对路径, session["path"])
```

用户只能看到相对于 chroot 后的相对路径。

### 9.3 sqlitefts 路径前缀范围查询

`server/plugin/plg_search_sqlitefts/indexer/query.go:14-19`

```sql
WHERE path > ? AND path < ?   -- 即 [path, path~) 利用字符串排序实现前缀匹配
```

利用 SQLite 的 path 索引做前缀扫描，高效限定搜索范围。

---

## 10. 全文搜索引擎对接

### 10.1 文件内容提取 pipeline

`server/plugin/plg_search_sqlitefts/crawler/phase_utils.go:14-33`

```
updateFile(path, backend, tx)
    ├─ backend.Cat(path)           ← 从 backend 取原始字节流
    ├─ converter.Convert(path, reader)  ← 按 MIME 类型转纯文本
    │     ├─ text/plain / text/org / text/markdown / application/x-form → textify.Txt
    │     ├─ application/pdf → textify.PDF
    │     └─ application/excel / word / powerpoint → textify.Office
    │     └─ 其他类型 → 返回 ErrNotImplemented（只索引文件名
    └─ tx.FileContentUpdate(path, convertedText)  ← 写入 FTS5 content 列
    └─ tx.IndexTimeUpdate(path, now)      ← 标记已索引
```

### 10.2 FTS5 虚拟表与触发器

`server/plugin/plg_search_sqlitefts/indexer/index.go:82-93`

```sql
CREATE VIRTUAL TABLE file_index USING fts5(
    path UNINDEXED,        -- path 不参与分词索引
    filename,              -- 文件名
    filetype,              -- 文件扩展名
    content,               -- 全文内容
    tokenize = 'porter'    -- porter 词干提取（英文）
);

-- 3 个触发器：file 表的增删改 → 自动同步 file_index 的元数据列
```

### 10.3 可索引扩展名与大小限制

`server/plugin/plg_search_sqlitefts/config/configuration.go`

- `INDEXING_EXT`：默认 `org,txt,docx,pdf,md,form,xlsx,pptx`
- `MAX_INDEXING_FSIZE`：默认 512MB
- `SEARCH_EXCLUSION`：默认排除 `node_modules,bower_components,.cache,.npm,.git`

---

## 11. 统一权限过滤

### 11.1 搜索前：整体权限检查

`server/ctrl/search.go:16`

```go
if model.CanRead(ctx) == false {
    SendErrorResult(res, ErrPermissionDenied)
    return
}
```

`server/model/permissions.go:7-12`

```go
func CanRead(ctx *App) bool {
    if ctx.Share.Id != "" {       // 共享链接场景
        return ctx.Share.CanRead   // 按共享链接授权
    }
    return true                   // 登录用户默认有读权限
}
```

### 11.2 文件操作级别的授权中间件链

`server/ctrl/files.go:99-104`（以 `FileLs` 为例：

```go
for _, auth := range Hooks.Get.AuthorisationMiddleware() {
    if err = auth.Ls(ctx, path); err != nil {
        SendErrorResult(res, err)
        return
    }
}
```

`FileHook` 就注册在这条链上——它**不做真正授权检查**（永远返回 nil），只异步触发 Hint。真正的权限过滤由其他 `IAuthorisation` 插件（如 `plg_authorisation_example` 自定义实现。

### 11.3 搜索结果的结果本身不做逐文件权限

搜索结果本身不做逐文件权限检查——结果过滤——只做整体 `CanRead` 检查。搜索结果直接从索引里返回所有匹配结果，依赖：
- chroot 路径前缀范围（path < ? AND path < ?）天然隔离
- 共享链接场景下 `ctx.Session["path"]` 前缀裁剪

---

## 12. 分页与排序抽象

### 12.1 后端不支持分页——全部返回

`ISearch.Query()` 返回完整 `[]IFile`，无 `limit/offset` 参数。sqlitefts 查询内 `LIMIT 50000` 是硬编码上限，非用户可控分页。

### 12.2 前端防抖 + 虚拟滚动分页

`public/assets/pages/filespage/ctrl_filesystem.js:79-92`

```js
if (state.search) {
    return rxjs.timer(450).pipe(          // 450ms 防抖
        rxjs.switchMap(() => search(state.search)),  // 新请求取消旧请求
        ...
    )
}
```

`public/assets/pages/filespage/ctrl_filesystem.js:136-170` 虚拟滚动：

```js
VIRTUAL_SCROLL_MINIMUM_TRIGGER = 100
if (files.length > 100) {
    size = Math.min(files.length, BLOCK_SIZE * COLUMN_PER_ROW)
}
```

超过 100 条时只渲染可视区 + 上下一屏的 DOM。

### 12.3 排序：前端二次排序（后端只做后端相关性排序

| 引擎 | 后端排序 | 前端排序 |
|------|---------|---------|
| stateless | 目录探索 Score 优先（类 BFS 深度优先 + 启发式评分
| sqlitefts | FTS5 BM25 `rank` 排序
| 通用（非搜索场景 `sort` 时前端才做 `sort(files, type, order)，支持 name/date/size/type 四种排序
（搜索结果**不做前端排序**，保持后端返回的相关性顺序。

`public/assets/pages/filespage/helper.js:22-130` 前端排序规则：

- `sortByType`：目录优先 → 隐藏文件置底 → 扩展名 → 文件名
- `sortByName`：目录优先 → 隐藏文件置底 → 文件名
- `sortByDate`：修改时间
- `sortBySize`：目录优先 → 大小

---

## 13. 查询缓存与失效

### 13.1 无服务端查询缓存

搜索结果不做服务端缓存。每次请求都实时：
- stateless：实时 BFS 遍历
- sqlitefts：实时查 SQLite

### 13.2 索引失效机制（触发重索引）：

1. **文件变更触发失效**：
   - `HintFile` → `tx.IndexTimeClear(path)` → 下次 INDEXING 阶段重新提取内容

2. **定时失效**：
   `SEARCH_REINDEX`（默认 24 小时）MAINTAIN 阶段：

```go
// phase_maintain.go
tx.FindBefore(now - SEARCH_REINDEX 小时前)
```

找出超过 24h 的条目 → 重新 `updateFile`/updateFolder` 对比 backend 实际状态。

3. **用户操作触发失效**：

| 操作 | 失效动作 |
|------|---------|
| Save/Touch | IndexTimeClear → 重提取内容
| Rm | RemoveAll → 从索引删除
| Mv | 旧路径 RemoveAll + 新路径 HintLs

### 13.3 前端 ls 缓存

`public/assets/pages/filespage/model_files.js:84-100` 的 `ls()` 有 indexedDB 缓存（仅用于文件列表缓存），但 `search()` 无缓存。

---

## 14. 超时、部分失败与 Fallback

### 14.1 stateless 超时 graceful degrade

`server/plugin/plg_search_stateless/index.go:24-26`

```go
MAX_SEARCH_TIME := SEARCH_TIMEOUT()  // 默认 1500ms
for start := time.Now(); time.Since(start) < MAX_SEARCH_TIME; {
    // 探索一个目录一个目录地遍历
}
return files, nil  // 超时直接返回已找到结果，不报错
```

超时不是错误——返回已探索到部分结果。

### 14.2 sqlitefts 查询失败 fallback

`server/plugin/plg_search_sqlitefts/indexer/query.go:21-24`

```go
rows, err := this.db.Query(...)
if err != nil {
    Log.Warning("search::query DBQuery (%s)", err.Error())
    return files, ErrNotReachable  // 返回空结果 + ErrNotReachable
}
```

### 14.3 索引阶段的错误处理：

- `backend 某个目录 `Ls` 失败 → 跳过该目录，切到 PHASE_PAUSE，下次再试
- 单个文件 `Cat`/内容转换失败 → 跳过该文件，继续下一个
- SQLite constraint 冲突 → 视为已存在，检查 `ErrConstraint`，跳过
- 单个文件处理 panic → `defer recover()` 记录日志，不影响其他文件

`server/plugin/plg_search_sqlitefts/crawler/daemon.go:140-151` 中的 panic recover：

```go
defer func() {
    if r := recover(); r != nil {
        Log.Error("plg_search_sqlitefs::panic backend="%s" recover="%s"", name, r)
    }
}()
```

---

## 15. 完整补充关键文件速查表（补充）

| 文件 | 作用 |
|------|------|
| `server/common/types.go:43-50` | `IFile`、`ISearch` 接口定义 |
| `server/common/plugin.go:152-166` | SearchEngine Hook 注册/获取 |
| `server/common/backend.go` | 全局 Backend Driver，存储后端注册表 |
| `server/common/crypto.go:193-218` | `GenerateID()`，Crawler/索引 ID 生成 |
| `server/middleware/session.go:57-83` | SessionStart，建立 App.Backend 绑定 |
| `server/model/files.go:9-51` | `NewBackend()`，按 session 初始化具体 backend |
| `server/model/permissions.go` | `CanRead()`，统一读权限检查 |
| `server/ctrl/search.go` | HTTP 搜索控制器 |
| `server/ctrl/files.go:99-105` | AuthorisationMiddleware 授权链调用点 |
| `server/ctrl/files.go:1104-1117` | `PathBuilder()`，chroot 路径构建 |
| `server/plugin/plg_search_stateless/index.go` | 无状态搜索实现 |
| `server/plugin/plg_search_stateless/config.go` | `SEARCH_TIMEOUT` 配置（默认 1500ms |
| `server/plugin/plg_search_stateless/scoring.go` | 目录探索评分 + 文件名匹配算法 |
| `server/plugin/plg_search_sqlitefts/index.go` | sqlitefts 插件入口，注册 Hook，启动后台爬虫 |
| `server/plugin/plg_search_sqlitefts/config/configuration.go` | 所有搜索相关配置项 |
| `server/plugin/plg_search_sqlitefts/query.go` | sqlitefts 查询入口 |
| `server/plugin/plg_search_sqlitefts/crawler/daemon.go` | Daemon 多 Crawler（多 backend）管理 |
| `server/plugin/plg_search_sqlitefts/crawler/events.go` | FileHook 文件操作监听，触发 Hint |
| `server/plugin/plg_search_sqlitefts/crawler/phase.go` | 四阶段状态机调度 |
| `server/plugin/plg_search_sqlitefts/crawler/types.go` | `HeapDoc` 优先队列 + `Document` 结构 |
| `server/plugin/plg_search_sqlitefts/crawler/phase_explore.go` | EXPLORE 阶段：遍历目录树 |
| `server/plugin/plg_search_sqlitefts/crawler/phase_indexing.go` | INDEXING 阶段：提取文件内容 |
| `server/plugin/plg_search_sqlitefts/crawler/phase_maintain.go` | MAINTAIN 阶段：增量重索引 |
| `server/plugin/plg_search_sqlitefts/crawler/phase_utils.go` | `updateFile()` 内容转换与入库 |
| `server/plugin/plg_search_sqlitefts/crawler/phase_pause.go` | PAUSE 阶段 |
| `server/plugin/plg_search_sqlitefts/converter/index.go` | 按 MIME 分派 textify 转文本 |
| `server/plugin/plg_search_sqlitefts/indexer/index.go` | SQLite 表结构 + Manager 接口 |
| `server/plugin/plg_search_sqlitefts/indexer/query.go` | FTS5 查询 SQL |
| `server/plugin/plg_search_sqlitefts/indexer/error.go` | 索引错误类型 |
| `server/plugin/plg_search_sqlitefts/workflow/index.go` | 工作流主动索引 StepIndexer |
| `server/pkg/textify/` | txt/pdf/office 文本提取 |
| `server/routes.go:68` | `/api/files/search` 路由注册 |
| `public/assets/pages/filespage/model_files.js:160-165` | 前端 search() 函数 |
| `public/assets/pages/filespage/ctrl_filesystem.js:75-170` | 前端搜索防抖 + 虚拟滚动 |
| `public/assets/pages/filespage/helper.js:22-130` | 前端排序实现 |

---

## 16. 异步索引错误重试机制

### 16.1 重试策略设计：基于状态机的时间片轮转

代码中**没有显式的指数退避重试逻辑**，采用"时间片切分 + 状态机轮转"实现隐式重试：

| 错误场景 | 代码位置 | 重试行为 |
|---------|---------|---------|
| EXPLORE 阶段某目录 `Backend.Ls()` 失败 | `phase_explore.go:23-25` | `this.CurrentPhase = PHASE_PAUSE` → 休眠一个 CYCLE_TIME（默认 10s）→ 下一循环回到 EXPLORE 阶段再试 |
| INDEXING 阶段某文件 `Backend.Cat()` 失败 | `phase_utils.go:18-22` | `tx.RemoveAll(path)` 从索引删除该文件 → 该阶段循环终止（`return false`）→ 下一阶段继续 |
| INDEXING 阶段 SQL 查询失败 | `phase_indexing.go:12-16` | 记录 Warning 日志，`return false` → 本阶段退出，下一循环再试 |
| SQLite constraint 冲突（路径已存在） | `phase_explore.go:58-73` | 检查 `IndexTimeGet`：若在 SEARCH_REINDEX（默认 24h）内则跳过，否则 `IndexTimeUpdate` 后继续索引 |
| 单个文件 panic | `daemon.go:140-151` | `defer recover()` 捕获 panic → 记录 Error 日志 → 不影响其他文件 |
| `createCrawler()` 时 `Backend.Init()` 失败 | `daemon.go:128-133` | 记录 Warning，直接 return → 下次用户操作触发 Hint 时再试 |

### 16.2 重试代价控制：最大进程池 + LRU 淘汰

`daemonState.HintLs`（`daemon.go:117-126`）：

```go
search_process_max := SEARCH_PROCESS_MAX()  // 默认 5
if lenIdx > 0 && search_process_max > 0 && lenIdx > (search_process_max-1) {
    toDel := this.idx[0 : lenIdx-(search_process_max-1)]  // 淘汰最早的 N 个
    for i := range toDel {
        toDel[i].Close()  // 关闭 SQLite 连接 + Backend 连接
    }
    this.idx = this.idx[lenIdx-(search_process_max-1):]
}
```

LRU 策略：`idx[]` 数组按创建时间排序，超过 `SEARCH_PROCESS_MAX` 时淘汰最旧的 Crawler。下次需要时重新创建。

---

## 17. FederatedQueryBuilder 与 Query Plan 优化

### 17.1 结论：**不存在 FederatedQueryBuilder**

代码中没有这个类或类似概念。`ISearch` 接口极其精简：

```go
type ISearch interface {
    Query(ctx App, basePath string, term string) ([]IFile, error)
}
```

没有 query plan、没有多 backend 联合查询、没有谓词下推、没有 cost-based optimizer。

### 17.2 真实的查询优化点

| 引擎 | 优化手段 | 代码位置 |
|------|---------|---------|
| **stateless** | 用 Score 优先队列替代普通 BFS 队列，优先探索"更可能有结果"的目录，减少回溯 | `plg_search_stateless/scoring.go` |
| **stateless** | 1500ms 硬超时，避免无限遍历 | `plg_search_stateless/config.go` |
| **sqlitefts** | `path > ? AND path < ?` 利用 SQLite 主键索引做范围扫描，限定搜索前缀 | `indexer/query.go:14-19` |
| **sqlitefts** | FTS5 内置 `ORDER BY rank LIMIT 50000`，利用 BM25 相关性截断，避免全量排序 | `indexer/query.go:21` |
| **sqlitefts** | `LIMIT 50000` 硬编码上限，防止结果集过大撑爆内存 | `indexer/query.go:17` |
| **sqlitefts** | 关键词预处理：`regexp.MustCompile(`(\.|\-)`).ReplaceAllString(q, "\"$1\"")`，避免 `.` `-` 被 FTS5 当作分词符 | `indexer/query.go:11` |

### 17.3 扩展点：注释中提及的 ES/Solr 对接

`server/common/plugin.go:155` 注释：

> "The idea here is to enable different type of usage like leveraging elastic search or solr with custom stuff around it"

意味着架构上预留了扩展点：用户可以实现自己的 `ISearch` 插件，通过 `Hooks.Register.SearchEngine()` 注册后覆盖默认实现。在自定义插件中可以实现 FederatedQuery、ES 聚合查询等复杂逻辑。

---

## 18. 同 backend 同路径冲突解决

### 18.1 入队去重：HintLs 重复路径检测

`daemonState.HintLs`（`daemon.go:99-113`）：

```go
alreadyHasPath := false
for j := 0; j < len(this.idx[i].FoldersUnknown); j++ {
    if this.idx[i].FoldersUnknown[j].Path == path {
        alreadyHasPath = true
        break
    }
}
if alreadyHasPath == false {
    heap.Push(&this.idx[i].FoldersUnknown, &Document{...})
}
```

每次 `HintLs` 入队前遍历 `FoldersUnknown` 检查路径是否已存在，避免同一目录重复入队。

### 18.2 数据库层去重：SQLite PRIMARY KEY + ON CONFLICT

`server/plugin/plg_search_sqlitefts/indexer/index.go` 中 `FileCreate` 的 SQL：

```sql
INSERT OR REPLACE INTO file (path, filename, filetype, type, parent, size, modTime, indexTime)
VALUES (?, ?, ?, ?, ?, ?, ?, ?)
```

- `ON CONFLICT DO REPLACE`：同路径新记录覆盖旧记录
- `phase_explore.go:58-73` 中对 `ErrConstraint` 的特殊处理：若 `IndexTime` 在 24h 内则跳过索引，避免频繁更新

### 18.3 多 goroutine 并发冲突：Crawler.mu + daemonState.mu 双层锁

```go
// daemonState 级别的全局锁（管理 idx[] 数组增删）
type daemonState struct {
    mu sync.RWMutex  // 保护 idx[] 数组和 n 游标
    idx []Crawler
}

// Crawler 级别的锁（保护单个 Crawler 的状态）
type Crawler struct {
    mu sync.Mutex    // 保护 FoldersUnknown、CurrentPhase、State
}
```

- `daemonState.mu`：RWMutex，读锁用于 `GetCrawler/NextCrawler`，写锁用于 `HintLs/createCrawler` 增删 idx
- `Crawler.mu`：每个 Crawler 独立锁，保护堆操作、状态切换、SQLite 事务

---

## 19. ES 索引同步延迟

### 19.1 结论：**不存在 ElasticSearch 插件**

代码中只有两种内置搜索引擎：
1. `plg_search_stateless`：无索引，实时遍历
2. `plg_search_sqlitefts`：SQLite FTS5 全文索引

没有 `plg_search_elasticsearch` 或类似插件。`plugin.go:155` 注释仅作为扩展思路提及，未实现。

### 19.2 sqlitefts 的索引延迟特性

虽然没有 ES，但 sqlitefts 有自己的延迟特征：

| 操作 | 延迟模型 | 延迟量级 |
|------|---------|---------|
| `Ls`/`Mkdir` 等目录操作 | 异步 Hint + 等待 EXPLORE 阶段时间片 | 0 ~ CYCLE_TIME*2（0~20s） |
| `Save`/`Touch` 等文件内容操作 | 异步 Hint + 等待 INDEXING 阶段时间片 | 0 ~ CYCLE_TIME*3（0~30s） |
| `Rm`/`Mv` 删除/移动 | 同步删除（`HintRm` 直接开事务 `RemoveAll`） | 毫秒级，立即生效 |
| 搜索时 lazy push | 推入队列后下次时间片才爬 | 0 ~ CYCLE_TIME（0~10s） |

> **关键区别**：删除操作是同步的，新增/修改是异步的。因为删除不需要走时间片，`HintRm` 直接 `GetCrawler → State.Change() → RemoveAll → Commit`。

### 19.3 索引延迟的用户感知

- 搜索结果可能不包含刚刚上传的文件（需要等下一轮 INDEXING）
- 但刚刚删除的文件不会出现在搜索结果中（立即删除）
- `SEARCH_REINDEX`（默认 24h）保证最终一致性

---

## 20. 权限缓存失效

### 20.1 后端权限：无缓存

`server/model/permissions.go:7-12`：

```go
func CanRead(ctx *App) bool {
    if ctx.Share.Id != "" {
        return ctx.Share.CanRead   // 直接从 ctx.Share 读，无缓存
    }
    return true
}
```

每次请求都实时计算，没有缓存层。`ctx.Share` 由 `_extractShare`（`middleware/session.go:17-28`）每次请求从数据库中重新读取。

### 20.2 前端文件列表缓存：基于路径前缀失效

`public/assets/pages/filespage/cache.js:63-77`：

```js
async remove(path, exact = true) {
    if (exact) {
        delete this.data[key];          // 精确删除
        return;
    }
    for (const k in this.data) {
        if (k.indexOf(key) === 0) {     // 前缀删除：删除该路径下所有缓存
            delete this.data[k];
        }
    }
}
```

失效时机（在 `ctrl_filesystem.js` 各操作回调中）：
- 上传文件后：`cache.remove(currentPath(), false)` → 删除当前目录及其所有子目录缓存
- 删除文件后：`cache.remove(filePath, true)` → 精确删除 + 父目录前缀删除
- 移动文件后：`cache.remove(fromPath, false)` + `cache.remove(toPath, false)` → 两端都删
- 重命名文件后：同上

### 20.3 搜索结果缓存：无缓存

前端 `search()` 函数（`model_files.js:160-165`）直接发 ajax，不走缓存。后端 `ISearch.Query()` 也没有缓存层。

---

## 21. 大 cursor 性能

### 21.1 结论：**不存在数据库 cursor**

搜索接口没有分页 cursor 概念。后端实现：

| 引擎 | 结果集处理 | 内存占用 |
|------|-----------|---------|
| stateless | 实时 BFS 遍历，匹配一条 append 一条到 slice，超时直接返回 | 与匹配到的文件数成正比，受 1500ms 限制 |
| sqlitefts | SQLite `Query` 一次性返回所有匹配行到 `[]IFile`，`LIMIT 50000` 硬限制 | 最多 50000 条记录的内存占用 |

### 21.2 前端虚拟滚动：对抗大结果集

`ctrl_filesystem.js:136-170`：

```js
const VIRTUAL_SCROLL_MINIMUM_TRIGGER = 100
if (files.length > 100) {
    size = Math.min(files.length, BLOCK_SIZE * COLUMN_PER_ROW)
}
```

超过 100 条时只渲染可视区 + 上下一屏的 DOM，通过 `ifscroll-before`/`ifscroll-after` 占位元素撑开滚动高度。

### 21.3 潜在性能点

- sqlitefts 单次查询最多 50000 条全部加载到内存，无流式返回
- 没有 `LIMIT/OFFSET` 参数，用户无法翻页
- stateless 超时后返回部分结果，可能漏掉深层文件

---

## 22. 缓存预热

### 22.1 结论：**无显式缓存预热机制**

没有 `warmup()`/`preload()`/`preheat()` 方法。但有三种隐式预热：

### 22.2 路径一：用户操作驱动的预热（HintLs）

`server/plugin/plg_search_sqlitefts/crawler/events.go:26-28`：

```go
func (this FileHook) Ls(ctx *App, path string) error {
    go this.Daemon.HintLs(ctx, path)  // 用户每浏览一个目录，后台就索引这个目录
    return nil
}
```

用户浏览哪里，后台索引哪里——"浏览即预热"。

### 22.3 路径二：搜索触发的预热

`server/plugin/plg_search_sqlitefts/query.go:24-29`：

```go
heap.Push(&crwlr.FoldersUnknown, &Document{Path: path, ...})
```

用户搜索某个路径时，该路径被推入探索队列，下次时间片优先爬取——"搜索即预热"。

### 22.4 路径三：工作流触发的全量预热

`server/plugin/plg_search_sqlitefts/workflow/index.go:48-135` 定义的 `StepIndexer` 工作流：

管理员可在后台配置：
- 手动触发：点击"Refresh Search Index"按钮
- 定时触发：配置 cron 表达式周期性全量爬取

工作流模式下，不按时间片轮转，启动 `SEARCH_PROCESS_PAR` 个 goroutine 并发跑完整个目录树，用 `sync.Cond` 协调 inflight 计数，直到 `FoldersUnknown` 为空。

---

## 23. 降级策略可见性

### 23.1 降级策略清单

| 降级场景 | 触发条件 | 降级行为 | 用户可见性 | 代码位置 |
|---------|---------|---------|-----------|---------|
| **stateless 超时降级** | `SEARCH_TIMEOUT`（默认 1500ms）到了 | 返回已探索到的部分结果 | **透明**：用户拿到部分结果，无任何提示 | `plg_search_stateless/index.go:24-26` |
| **sqlitefts 查询失败降级** | SQLite 查询出错（锁、损坏等） | 返回空 `[]IFile` + `ErrNotReachable` | **半透明**：用户看到空搜索结果，前端不弹错误 | `indexer/query.go:21-24` |
| **SEARCH_ENABLE=false 降级** | 管理员关闭全文搜索 | 回退到 stateless 引擎（仅文件名匹配） | **透明**：用户可能觉得搜索不准，但看不到提示 | `plg_search_sqlitefts/index.go:152-173` |
| **索引内容转换失败降级** | PDF/Office 解析失败、格式不支持 | 只索引文件名，不索引内容 | **透明**：该文件只能靠文件名搜到 | `phase_utils.go:25-27` |
| **单目录 Ls 失败降级** | 网络波动、权限不足导致某目录打不开 | 跳过该目录，切到 PAUSE 阶段 | **透明**：该目录下文件搜不到，无提示 | `phase_explore.go:23-25` |
| **前端 indexedDB 降级** | Firefox 隐私模式、浏览器不支持 | 回退到内存缓存（InMemoryCache） | **透明**：刷新页面缓存失效 | `cache.js:227-233` |
| **Crawler 池满降级** | 活跃 Crawler 超过 `SEARCH_PROCESS_MAX` | LRU 淘汰最早的 Crawler，关闭其 SQLite + Backend 连接 | **透明**：下次用到时重新创建，可能有延迟 | `daemon.go:117-126` |

### 23.2 唯一用户可见的降级提示

前端搜索框 placeholder 在 `enable_search=false` 时不会显示。`ctrl_submenu.js:240`：

```html
<button data-action="search" class=${getConfig("enable_search") ? "" : "hidden"}>
```

管理员关闭搜索功能时，搜索按钮直接隐藏——这是唯一可见的降级信号。

---

## 24. 完整关键文件速查表（最终版）

| 文件 | 作用 |
|------|------|
| `server/common/types.go:43-50` | `IFile`、`ISearch` 接口定义 |
| `server/common/plugin.go:141-166` | SearchEngine + AuthorisationMiddleware Hook |
| `server/common/backend.go` | 全局 Backend Driver，存储后端注册表 |
| `server/common/crypto.go:193-218` | `GenerateID()`，Crawler/索引 ID 生成 |
| `server/middleware/session.go:57-83` | SessionStart，建立 App.Backend 绑定 |
| `server/model/files.go:9-51` | `NewBackend()`，按 session 初始化具体 backend |
| `server/model/permissions.go` | `CanRead()`，统一读权限检查 |
| `server/ctrl/search.go` | HTTP 搜索控制器 |
| `server/ctrl/files.go:99-105` | AuthorisationMiddleware 授权链调用点 |
| `server/ctrl/files.go:1104-1117` | `PathBuilder()`，chroot 路径构建 |
| `server/plugin/plg_search_stateless/index.go` | 无状态搜索实现 |
| `server/plugin/plg_search_stateless/config.go` | `SEARCH_TIMEOUT` 配置（默认 1500ms） |
| `server/plugin/plg_search_stateless/scoring.go` | 目录探索评分 + 文件名匹配算法 |
| `server/plugin/plg_search_sqlitefts/index.go` | sqlitefts 插件入口，注册 Hook，启动后台爬虫 |
| `server/plugin/plg_search_sqlitefts/config/configuration.go` | 所有搜索相关配置项 |
| `server/plugin/plg_search_sqlitefts/query.go` | sqlitefts 查询入口 |
| `server/plugin/plg_search_sqlitefts/crawler/daemon.go` | Daemon 多 Crawler 管理 + LRU 淘汰 + panic recover |
| `server/plugin/plg_search_sqlitefts/crawler/events.go` | FileHook 文件操作监听，触发 Hint |
| `server/plugin/plg_search_sqlitefts/crawler/phase.go` | 四阶段状态机调度 |
| `server/plugin/plg_search_sqlitefts/crawler/types.go` | `HeapDoc` 优先队列 + `Document` 结构 |
| `server/plugin/plg_search_sqlitefts/crawler/phase_explore.go` | EXPLORE 阶段：遍历目录树 + 冲突重试 |
| `server/plugin/plg_search_sqlitefts/crawler/phase_indexing.go` | INDEXING 阶段：提取文件内容 |
| `server/plugin/plg_search_sqlitefts/crawler/phase_maintain.go` | MAINTAIN 阶段：增量重索引 |
| `server/plugin/plg_search_sqlitefts/crawler/phase_utils.go` | `updateFile()`/`updateFolder()` 同步逻辑 |
| `server/plugin/plg_search_sqlitefts/crawler/phase_pause.go` | PAUSE 阶段：错误后重试等待 |
| `server/plugin/plg_search_sqlitefts/converter/index.go` | 按 MIME 分派 textify 转文本 |
| `server/plugin/plg_search_sqlitefts/indexer/index.go` | SQLite 表结构 + Manager 接口 |
| `server/plugin/plg_search_sqlitefts/indexer/query.go` | FTS5 查询 SQL |
| `server/plugin/plg_search_sqlitefts/indexer/error.go` | 索引错误类型 |
| `server/plugin/plg_search_sqlitefts/workflow/index.go` | 工作流主动索引 StepIndexer |
| `server/plugin/plg_search_example/index.go` | 最简搜索插件示例 |
| `server/pkg/textify/` | txt/pdf/office 文本提取 |
| `server/routes.go:68` | `/api/files/search` 路由注册 |
| `public/assets/pages/filespage/model_files.js:160-165` | 前端 search() 函数 |
| `public/assets/pages/filespage/ctrl_filesystem.js:75-170` | 前端搜索防抖 + 虚拟滚动 |
| `public/assets/pages/filespage/helper.js:22-130` | 前端排序实现 |
| `public/assets/pages/filespage/cache.js` | 前端 ls 缓存（IndexedDB → 内存缓存降级） |
