# Filestash 会话机制与多账号切换实现分析

## 一、整体架构概览

Filestash 采用**无状态服务端 + 客户端加密 Cookie** 的会话模型。服务端不持久化用户会话数据，所有凭据均加密后存储在用户浏览器的 Cookie 中；各后端插件使用进程内 `AppCache` 对连接对象做短期缓存以提升性能。

核心数据结构在 `server/common/app.go:7-15`：

```go
type App struct {
    Backend       IBackend          // 当前请求的后端连接实例
    Body          map[string]interface{}
    Session       map[string]string // 解密后的会话参数（type/username/password/path/timestamp 等）
    Share         Share             // 共享链接上下文
    Context       context.Context
    Authorization string            // 原始加密 token 字符串
    Languages     []string
}
```

---

## 二、会话存储位置

### 2.1 客户端：加密 Cookie（主存储）

所有会话数据存储在用户浏览器的 Cookie 中，路径作用域为 `/api/`。

**相关常量** `server/common/constants.go:11-18`：
```go
COOKIE_NAME_AUTH  = "auth"
COOKIE_NAME_PROOF = "proof"
COOKIE_NAME_ADMIN = "admin"
COOKIE_PATH       = "/api/"
COOKIE_PATH_ADMIN = "/admin/api/"
```

**分片存储机制**（当凭据较大时）：
Cookie 有体积限制（单 cookie ~4KB），Filestash 实现了自动分片。`server/common/utils.go:97-102`：
```go
func CookieName(idx int) string {
    if idx == 0 {
        return COOKIE_NAME_AUTH      // "auth"
    }
    return COOKIE_NAME_AUTH + strconv.Itoa(idx)  // "auth1", "auth2", ...
}
```
分片写入逻辑见 `server/ctrl/session.go:102-124`，每片最大 3800 字节。

**加密流程** `server/common/crypto.go:27-53`：
```
明文会话(JSON) → zlib 压缩 → AES-256-GCM 加密 → Base64 URL 编码 → Cookie Value
```
密钥派生 `server/common/constants.go:72-79`：
```go
SECRET_KEY_DERIVATE_FOR_USER = Hash("USER_"+SECRET_KEY, len(SECRET_KEY))
```

### 2.2 服务端：进程内连接缓存（性能优化）

每个后端插件独立维护一个 `AppCache`，核心实现在 `server/common/cache.go:11-63`。它使用 `patrickmn/go-cache` + 结构哈希作为 key：

| 后端插件 | 缓存变量 | 保留时长 | 位置 |
|---|---|---|---|
| SFTP | `SftpCache` | 1 分钟 | `server/plugin/plg_backend_sftp/index.go:27` |
| Samba | `SambaCache` | 30 分钟 | `server/plugin/plg_backend_samba/index.go:21` |
| S3 | `S3Cache` | 2 分钟 | `server/plugin/plg_backend_s3/index.go:39` |
| TmpStorage | `ChrootCache` | 30 天 | `server/plugin/plg_backend_tmp/index.go:22` |

缓存 key 由 `hashstructure.Hash(session_params)` 生成，即会话参数字典的哈希值。

### 2.3 数据库：仅存储共享链接元数据

SQLite 数据库 `state/db/share.sql` 仅保存共享链接（Share）信息，不保存用户会话。表结构在 `server/model/index.go:20-33`：
- `Location(backend, path)` - 位置信息
- `Share(id, related_backend, related_path, params, auth)` - 共享链接，其中 `auth` 字段存加密的会话快照

---

## 三、不同后端账号隔离机制

### 3.1 会话唯一标识：GenerateID

`server/common/crypto.go:193-218` 定义了会话 ID 生成算法：

```go
func GenerateID(params map[string]string) string {
    p := ""
    orderedKeys := make([]string, len(params))
    // ... 排序所有 key ...
    for _, key := range orderedKeys {
        switch key {
        case "password":   // 排除
        case "path":       // 排除
        case "session":    // 排除
        case "timestamp":  // 排除
        default:
            if val := params[key]; val != "" {
                p += key + "=>" + params[key] + ", "
            }
        }
    }
    p += "salt=>" + SECRET_KEY  // 混入服务端密钥防篡改
    return Hash(p, 20)
}
```

**关键点**：密码、路径、时间戳不参与 ID 计算，这意味着：
- 同一用户（相同 username/hostname 等）用不同密码登录 → **同一 ID**（但密码错误会在 Init 阶段被拒绝）
- 同一用户连接同一后端的不同 path → **同一 ID**
- 不同用户 → **不同 ID**（保证隔离）

### 3.2 后端 ID 与路径绑定

完整的 Backend ID 还会绑定路径，定义在 `server/ctrl/session.go:509-511`：
```go
func backendID(session map[string]string) string {
    return Hash(GenerateID(session)+session["path"], 20)
}
```
这用于前端区分同一账号下的不同路径挂载点。

### 3.3 连接白名单校验

创建后端连接前必须通过配置白名单校验，`server/model/files.go:9-51`：
```go
func NewBackend(ctx *App, conn map[string]string) (IBackend, error) {
    isAllowed := func() bool {
        // 遍历 Config.Conn，匹配 type + hostname + path + url
        // 只有管理员配置过的后端才允许连接
    }
    if isAllowed() == false {
        return Backend.Get(BACKEND_NIL), ErrNotAllowed
    }
    return Backend.Get(conn["type"]).Init(conn, ctx)
}
```
这防止了黑客使用 Filestash 连接配置外的任意后端。

### 3.4 共享链接归属校验

在 `server/middleware/session.go:96-157` 的 `CanManageShare` 中验证：
```go
if s.Backend == GenerateID(ctx.Session) {
    // 当前会话的用户 == 创建共享链接的用户 → 允许操作
}
```
确保用户只能管理自己创建的共享链接。

### 3.5 各后端插件级隔离

每个后端在 `Init(params)` 时把凭据封装在自己的结构体中，例如：
- **WebDAV** `server/plugin/plg_backend_webdav/index.go:34-50`：`WebDavParams{url, username, password, path}`
- **S3** `server/plugin/plg_backend_s3/index.go:42-101`：`aws.Config` + `params`
- **SFTP** `server/plugin/plg_backend_sftp/index.go:44-79`：从 `SftpCache` 按 `params` 哈希取出 `*ssh.Client`

不同账号的连接对象在 `AppCache` 中以不同哈希 key 存储，互不干扰。

---

## 四、切换账号时凭据更新流程

### 4.1 前端发起切换

入口在 `public/assets/pages/connectpage/ctrl_form.js:129-221`。

用户在连接页面选择不同存储后端 / 输入不同账号密码 → 表单提交 → 调用 `createSession(formData)`（`public/assets/model/session.js:18-30`）：
```javascript
export function createSession(authenticationRequest) {
    return ajax({
        method: "POST",
        url: withShare("api/session"),
        body: authenticationRequest,
        responseType: "json",
    })
}
```

### 4.2 后端验证并签发新 Cookie

`POST /api/session` 路由在 `server/routes.go:23-32`，中间件链不包含 `SessionStart`（因为是登录接口），最终执行 `SessionAuthenticate` `server/ctrl/session.go:52-135`：

```
Step 1: 注入时间戳 → ctx.Body["timestamp"] = time.Now()
Step 2: 构造 session map，确保 path 以 "/" 结尾
Step 3: model.NewBackend() → 调用具体后端的 Init() 验证凭据
        ├─ S3: 构造 aws.Config，不立即发起网络请求
        ├─ SFTP: 建立 SSH 连接 + SFTP 子系统
        ├─ Local: bcrypt 校验管理员密码
        └─ 其它后端各有不同
Step 4: 若支持 OAuth（OAuthToken 接口），获取并注入 access_token
Step 5: model.GetHome() 验证至少能访问 home 目录
Step 6: json.Marshal(session) → 序列化
Step 7: EncryptString(SECRET_KEY_DERIVATE_FOR_USER, ...) → 加密
Step 8: 分片写入 Set-Cookie（auth, auth1, auth2, ...）
        MaxAge = 60 * general.cookie_timeout（分钟）
Step 9: 返回 { is_authenticated, home, backendID, authorization }
```

### 4.3 浏览器端覆盖更新

浏览器收到多个 `Set-Cookie: auth=...; MaxAge=...` 响应头后，自动覆盖原有同名 Cookie。前端收到响应后：
- 若开启 chromecast，把 `authorization` token 存入 `window.BEARER_TOKEN`
- 跳转到 `/files{home}`

### 4.4 旧连接的清理

旧账号的后端连接对象保留在各自插件的 `AppCache` 中，不会被主动销毁，而是依赖缓存 TTL 过期：
- SFTP：1 分钟后通过 `OnEvict` 回调关闭 `ssh.Client`
- Samba：30 分钟后卸载共享
- 等等

这是权衡设计：如果每次切换账号都同步关闭旧连接，可能因后端 TCP 关闭握手导致登出卡顿（参见 `SessionLogout` 中的 goroutine 优化注释 `server/ctrl/session.go:138-152`）。

---

## 五、超时与强制下线的处理路径

### 5.1 Cookie 相对超时（可配置）

来源：`general.cookie_timeout` 配置项，默认 `60 * 24 * 7 = 10080` 分钟（1 周）`server/common/config.go:84`。

写入位置：`server/ctrl/session.go:115`
```go
MaxAge: 60 * Config.Get("general.cookie_timeout").Int(),
```
浏览器根据 `Max-Age` 自动删除过期 Cookie。

### 5.2 Session 绝对超时（硬编码 1 年）

解密后强制检查，`server/middleware/session.go:306-313`：
```go
t, err := time.Parse(time.RFC3339, session["timestamp"])
if err != nil {
    return session, ErrNotAuthorized
} else if t.Add(24 * 365 * time.Hour).Before(time.Now()) {
    // 登录时间超过 1 年 → 强制失效
    return session, ErrNotAuthorized
}
```

### 5.3 管理员会话超时（AdminToken）

管理员后台独立使用 `AdminToken`，有效期 24 小时 `server/common/token.go:16-35`：
```go
type AdminToken struct {
    Claim  string    `json:"token"`  // "ADMIN"
    Expire time.Time `json:"time"`   // time.Now() + 24h
}
func (this AdminToken) IsValid() bool {
    return this.Expire.Sub(time.Now()) > 0
}
```
存储在 `admin` Cookie（路径 `/admin/api/`）。

### 5.4 共享链接过期

`Share.IsValid()` `server/common/types.go:196-204`：
```go
func (s Share) IsValid() error {
    if s.Expire != nil {
        now := time.Now().UnixNano() / 1000000
        if now > *s.Expire {
            return NewError("Link has expired", 410)
        }
    }
    return nil
}
```
每次通过共享链接访问时校验。

### 5.5 密钥变更导致全体下线

所有会话加密依赖 `SECRET_KEY`。若管理员更改 `general.secret_key`：
- `InitSecretDerivate()` 重新派生 `SECRET_KEY_DERIVATE_FOR_USER`
- 所有现有 Cookie 解密失败 → `server/middleware/session.go:298-301` 返回 `ErrNotAuthorized`
- 同时触发 `RecoverFromBadCookie()` `server/common/recovery.go:10-21` 发送 `Set-Cookie: auth=; MaxAge=-1` 清除客户端坏 Cookie

这在配置描述中有明确警告 `server/common/config.go:71`：
> *"Update this settings will invalidate existing user sessions and shared links"*

### 5.6 主动登出（Logout）

**前端**：访问 `/logout` → `public/assets/pages/ctrl_logout.js:10-18` 调用 `deleteSession()` → `DELETE /api/session`

**后端** `server/ctrl/session.go:137-181`：
```
Step 1: goroutine 异步关闭后端连接（避免阻塞登出响应）
        - SessionTry 重新从 Cookie 提取会话和 Backend
        - 若 Backend 实现 Close() 接口则调用（释放 TCP/SSH 连接）
Step 2: 循环删除所有分片 auth Cookie（auth, auth1, ..., MaxAge=-1）
Step 3: 删除 admin Cookie
Step 4: 删除 proof Cookie
Step 5: 记录登出审计日志
```

### 5.7 解密失败 / 坏 Cookie 恢复

`SessionStart` 中间件中 `_extractSession` 解密失败时 `server/middleware/session.go:66-70`：
```go
if ctx.Session, err = _extractSession(req, ctx); err != nil {
    RecoverFromBadCookie(res)   // 写 Set-Cookie 删除坏 auth
    SendErrorResult(res, err)   // 返回 401
    return
}
```

### 5.8 MCP Handler 的 Token 体系

MCP（Model Context Protocol）插件有独立的 OAuth2 风格 token 机制 `server/plugin/plg_handler_mcp/handler_auth.go`：
- `DEFAULT_TOKEN_EXPIRY = 3600`（1 小时）
- `DEFAULT_SECRET_EXPIRY = 30 * 24 * 3600`（30 天）
- 通过 `EncryptString(KEY_FOR_CODE, ctx.Authorization)` 将现有会话封装为 authorization code

---

## 六、请求生命周期中的会话处理链路

每个 API 请求经过以下中间件链（以 `/api/files/ls` 为例）：

```
HTTP Request
    │
    ▼
ApiHeaders, SecureHeaders, SecureOrigin （安全头）
    │
    ▼
SessionStart  server/middleware/session.go:57-83
    ├─ _extractShare(req)             → 解析共享链接（如有）
    ├─ _extractAuthorization(req)     → 从 Cookie / Header / Query 读取加密 token
    │                                   优先级：分片 Cookie > Bearer Header > ?authorization= > Basic Auth
    ├─ _extractSession(req, ctx)      → 解密 token → map[string]string
    │                                   共享链接走 ctx.Share.Auth 分支
    │                                   正常登录走 ctx.Authorization 分支
    └─ _extractBackend(req, ctx)      → model.NewBackend() 初始化后端连接
    │                                   优先从各自 AppCache 取
    │
    ▼
LoggedInOnly  server/middleware/session.go:18-26
    └─ 检查 ctx.Backend != nil && ctx.Session != nil
    │
    ▼
PluginInjector （插件注入中间件）
    │
    ▼
实际 Handler （如 FilesLs）
```

---

## 七、关键文件索引

| 功能 | 文件路径 |
|---|---|
| 会话中间件（提取/校验） | `server/middleware/session.go` |
| 会话控制器（登录/登出） | `server/ctrl/session.go` |
| 请求上下文定义 | `server/common/app.go` |
| 加密/解密/GenerateID | `server/common/crypto.go` |
| Cookie 名称与密钥派生 | `server/common/constants.go` |
| 后端连接缓存 | `server/common/cache.go` |
| 后端工厂与白名单 | `server/model/files.go` |
| 后端接口与类型 | `server/common/types.go` |
| 坏 Cookie 恢复 | `server/common/recovery.go` |
| 配置（cookie_timeout 等） | `server/common/config.go` |
| 前端会话模型 | `public/assets/model/session.js` |
| 前端连接页控制器 | `public/assets/pages/connectpage/ctrl_form.js` |
| 前端登出控制器 | `public/assets/pages/ctrl_logout.js` |
| MCP 会话类型 | `server/plugin/plg_handler_mcp/types/session.go` |
| MCP OAuth 处理 | `server/plugin/plg_handler_mcp/handler_auth.go` |

---

## 八、并发场景下的会话语义深度分析

### 8.1 核心前提：请求级会话快照模型

在深入并发场景之前，必须先理解 Filestash 会话机制的**基本设计原则**：

**会话身份在请求开始时一次性提取，请求生命周期内保持不变。**

每个 HTTP 请求进入 `SessionStart` 中间件时 `server/middleware/session.go:57-83`：
1. 从 `req.Cookie()` 读取 Cookie → 解密 → 存入 `ctx.Session`
2. 根据 session params 初始化 Backend → 存入 `ctx.Backend`
3. 后续整个 handler 执行过程中，身份和连接都基于这个 ctx 副本

这意味着：
- Cookie 是**输入**，只在请求开头读一次
- Set-Cookie 是**输出**，只影响后续请求
- 正在处理中的请求不会因为 Cookie 变化而"中途变身份"

### 8.2 场景一：注销请求 vs 正在进行的业务请求

#### 8.2.1 时序分析

考虑如下并发场景：

```
时间轴 →
  │
  ├─ 请求 A（文件下载）发起 → SessionStart 提取旧身份 → 开始传输数据
  │
  ├─ 请求 B（DELETE /api/session 注销）发起
  │    └─ 中间件链：ApiHeaders → SecureHeaders → SecureOrigin → PluginInjector
  │       （注意：不包含 SessionStart！见 server/routes.go:28-29）
  │
  ├─ 请求 B 执行 SessionLogout:
  │    ├─ 启动 goroutine 异步关连接
  │    ├─ 写 Set-Cookie: auth=; MaxAge=-1
  │    └─ 返回 200 OK
  │
  └─ 请求 A 继续传输... 传输完成返回 200 OK
```

#### 8.2.2 关键结论

| 问题 | 答案 | 代码依据 |
|---|---|---|
| 注销会中断正在进行的请求吗？ | **不会** | `SessionStart` 在请求开头一次性提取身份到 ctx，后续不再检查 Cookie |
| 正在进行的请求返回 401 吗？ | **不会，按旧身份正常处理** | handler 执行不依赖 Cookie，只依赖 ctx 中的 session 副本 |
| 是"先删 Session 还是读取时被中断"？ | 都不是。两者完全独立，互不影响 | 注销请求不走 SessionStart；业务请求身份在开头已确定 |

#### 8.2.3 为什么注销不走 SessionStart？

查看路由配置 `server/routes.go:28-29`：
```go
middlewares = []Middleware{ApiHeaders, SecureHeaders, SecureOrigin, PluginInjector}
session.HandleFunc("", NewMiddlewareChain(SessionLogout, middlewares)).Methods("DELETE")
```

**设计意图**：
1. 注销操作应该总是成功（即使 Cookie 损坏/过期也应该能登出）
2. 如果走 SessionStart，Cookie 损坏时会直接返回 401，用户反而"登不出去"
3. 注销只需要从 `req` 读 Cookie 名来决定删几片，不需要解密

`SessionLogout` 中读取 Cookie 的方式是按序号遍历 `server/ctrl/session.go:153-166`：
```go
index := 0
for {
    _, err := req.Cookie(CookieName(index))
    if err != nil { break }
    http.SetCookie(res, applyCookieRules(&http.Cookie{
        Name: CookieName(index), Value: "", MaxAge: -1, Path: COOKIE_PATH,
    }, req))
    index++
}
```

### 8.3 场景二：账号切换时的并发竞态

#### 8.3.1 同时读写 Cookie 的竞态窗口

**场景**：用户在标签页 A 正在下载大文件（旧身份），同时在标签页 B 提交登录表单切换到新账号。

```
浏览器端                      服务端
   │                           │
   ├─ 请求A (旧Cookie) ────────►
   │                           ├─ SessionStart 提取旧身份
   │                           └─ 开始下载...
   │
   ├─ POST /api/session ───────►
   │    （新账号密码）           ├─ 验证凭据
   │                           ├─ 生成新加密会话
   │                           └─ Set-Cookie: auth=新值
   │
   ◄──── 200 OK + 新Cookie ────┤
   │
   ├─ 请求C (新Cookie) ────────►  → 新身份处理
   │
   └─ 请求A继续下载中...         └─ 仍按旧身份传输
```

#### 8.3.2 竞态分析

**请求级隔离**：
- 请求 A 发起时携带旧 Cookie，服务端在 `SessionStart` 中提取身份
- 即使之后 Cookie 被覆盖/删除，请求 A 的 ctx 里已经有了旧身份
- **正在进行的请求不会中途切换身份**

**后续请求生效**：
- 浏览器收到 `Set-Cookie` 后更新本地 Cookie 存储
- 之后发起的新请求会携带新 Cookie → 新身份
- 但如果在登录请求返回前就发起了新请求，那些请求仍带旧 Cookie

#### 8.3.3 浏览器端的并发注意事项

浏览器对 Cookie 的读写是**单线程原子操作**（由浏览器内核管理），但：
- 多个标签页共享同一 Cookie Jar
- 标签页 A 正在发送请求时（请求头已发出），标签页 B 的 Set-Cookie 不会影响这个已发出的请求
- 已发出的请求的响应中的 Set-Cookie 会更新 Cookie Jar，影响之后的请求

### 8.4 场景三：AppCache 异步清理窗口的安全边界

这是最值得深入分析的场景。

#### 8.4.1 AppCache 的 key 计算

`server/common/cache.go:16-36`：
```go
func (a *AppCache) Get(key interface{}) interface{} {
    hash, err := hashstructure.Hash(key, nil)  // 对整个 map 做哈希
    // ...
    value, found := a.Cache.Get(fmt.Sprintf("%d", hash))
    return value
}

func (a *AppCache) Set(key map[string]string, value interface{}) {
    hash, err := hashstructure.Hash(key, nil)
    a.Cache.Set(fmt.Sprint(hash), value, cache.DefaultExpiration)
}
```

**关键点**：`hashstructure.Hash(key, nil)` 对**整个** `map[string]string` 做哈希，包括 **password、path、timestamp** 等所有字段。

这与 `GenerateID` 不同（GenerateID 排除 password/path/timestamp）。AppCache 的 key 是**全量参数哈希**。

#### 8.4.2 切换账号后旧连接还在吗？

**答案**：在 TTL 过期之前，旧连接对象仍然保留在 AppCache 中。

各后端的缓存保留时长：

| 后端 | 缓存时长 | 切换账号后旧连接存活时间 |
|---|---|---|
| SFTP | 1 分钟 | 最多约 1 分钟 |
| S3 (Region缓存) | 2 分钟 | 最多约 2 分钟（仅缓存 Region，不缓存连接） |
| Samba | 30 分钟 | 最多约 30 分钟 |
| NFS | 取决于配置 | 取决于配置 |
| Git | 10 分钟 | 最多约 10 分钟 |
| FTP | 30 秒 | 最多约 30 秒 |
| WebDAV | 无缓存 | 0（每次新建连接） |
| Backblaze | 5 分钟 | 最多约 5 分钟 |

#### 8.4.3 安全分析：旧凭据能被利用吗？

**外部攻击者**：不能。
- 要从 AppCache 中取出连接，需要先通过 `SessionStart` 中间件
- `SessionStart` 需要从 Cookie 解密出 session params
- Cookie 已被删除/覆盖，攻击者无法获得旧凭据的明文
- 即使猜测 params，也需要构造出完全相同的 map（包括 password 的精确值）才能命中缓存 key

**同一浏览器的多标签页**：存在短暂的"并发存活窗口"。
- 标签页 A 已建立连接（旧身份），请求还在进行中
- 标签页 B 切换到新账号，Cookie 被更新
- 标签页 A 的**正在进行中的请求**继续使用旧连接正常工作
- 标签页 A 的**后续新请求**会携带新 Cookie → 新身份 → 新连接
- 旧连接在缓存中静静等待 TTL 过期，不会被新身份意外命中（因为 params 哈希不同）

#### 8.4.4 同一后端不同账号的缓存隔离

**问题**：用户在同一台 SFTP 服务器上从 user1 切换到 user2，缓存会串吗？

**答案**：不会串。

```
user1 的 params:  {type:sftp, hostname:example.com, username:user1, password:pass1, ...}
                → hash 为 H1
                → 缓存 key = "H1"

user2 的 params:  {type:sftp, hostname:example.com, username:user2, password:pass2, ...}
                → hash 为 H2
                → 缓存 key = "H2"

H1 ≠ H2 → 两个独立的缓存条目 → 互不干扰
```

因为 `hashstructure.Hash` 对整个 map 计算，username 不同 → 哈希不同 → 缓存条目不同。

#### 8.4.5 特殊场景：同一账号改密码

如果用户**没有注销**，而是直接用同一账号、**新密码**重新登录：

1. 旧密码对应的缓存条目（H_old）仍然存在，直到 TTL 过期
2. 新密码创建新的缓存条目（H_new）
3. 新请求使用新 Cookie → 新 params → H_new → 新连接
4. 旧连接在缓存中"静默过期"

这不会造成安全问题，因为旧连接仍然是同一个用户的合法连接（只是密码变更了，但已建立的 SSH/SFTP 连接不受密码变更影响）。

### 8.5 场景四：注销时 goroutine 异步关闭连接的细节

#### 8.5.1 代码回顾

`server/ctrl/session.go:138-152`：
```go
go func() {
    middleware.SessionTry(func(c *App, _res http.ResponseWriter, _req *http.Request) {
        if c.Backend != nil {
            if obj, ok := c.Backend.(interface{ Close() error }); ok {
                obj.Close()
            }
        }
    })(ctx, res, req)
}()
```

#### 8.5.2 执行时序

```
主 goroutine（请求处理）              后台 goroutine（关连接）
        │                                    │
        ├─ 启动 goroutine                     │
        ├─ 写 Set-Cookie 删除 auth             ├─ SessionTry 开始
        │                                      ├─ 从 req.Cookie() 读旧 Cookie
        ├─ SendSuccessResult 返回响应          ├─ 解密 → 提取 session
        │                                      ├─ NewBackend 初始化或从缓存取
        │                                      ├─ 调用 Backend.Close()
        ▼                                      ▼
   请求处理完成                          连接关闭完成
```

#### 8.5.3 并发安全分析

**关于 `req` 对象的使用**：
- 后台 goroutine 读取 `req.Cookie()` 时，主 goroutine 可能已经完成了请求处理
- 严格来说，Go 的 `net/http` 文档不保证 handler 返回后 `*http.Request` 仍然有效
- 但实际上，由于 `SessionLogout` 的中间件链不包含 SessionStart，主 goroutine 几乎不修改 req
- 且 Cookie 解析（`req.Cookie()`）是纯读取操作，没有副作用
- 实际风险很低，但这是一个理论上的并发安全瑕疵

**关于 `ctx` 对象的使用**：
- `SessionTry` 会修改传入的 ctx（设置 Session, Backend 等字段）
- 主 goroutine 在注销流程中也使用同一个 ctx
- 但由于注销流程不读 ctx.Session/ctx.Backend（本来就是空的），所以不会有数据竞争问题
- 不过这仍然不是良好的并发实践（两个 goroutine 写同一个结构体的不同字段，在 Go 内存模型下是有风险的）

#### 8.5.4 Close() 操作的影响

调用 `Backend.Close()` 会：
- SFTP：关闭 `ssh.Client` 和 `sftp.Client` `server/plugin/plg_backend_sftp/index.go:347-355`
- Samba：卸载 SMB 共享
- 等等

**注意**：这只会关闭 goroutine 中新创建/取出的那个连接实例。各后端插件独立实现了引用计数机制，用于跟踪活跃请求数。以 SFTP 为例 `server/plugin/plg_backend_sftp/index.go:62-78`：
```go
c := SftpCache.Get(params)
if c != nil {
    d := c.(*Sftp)
    d.wg.Add(1)          // 增加引用计数
    go func() {
        <-app.Context.Done()
        d.wg.Done()      // 请求结束后减少引用计数
    }()
    return d, nil
}
```

但 `Close()` 调用时**不检查 wg**，直接关闭连接。这意味着：
- 如果注销时调用 Close()，而此时有其他请求正在使用同一个缓存连接...
- 实际上不会发生，因为每个请求的 params 哈希相同 → 取出同一个连接对象
- 但 Close() 会直接关闭底层 socket，导致其他正在使用该连接的请求出错

**然而**，在注销场景下这通常不是问题：
- 注销的是当前用户，这个用户的其他并发请求本来就应该终止
- 而且 AppCache 的 key 包含 password 等所有字段，同一用户同一密码才会命中同一缓存
- 实际中用户很少会在下载文件的同时点注销

### 8.6 场景五：登录并发与多次建连

#### 8.6.1 重复登录请求

如果用户快速双击登录按钮，或者网络不好导致重试，会发生什么？

```
请求 1 (POST /api/session): 验证通过 → 建连接 → 写 Cookie
请求 2 (POST /api/session): 验证通过 → 建连接 → 写 Cookie
```

两次都会成功，都会：
- 建立新的后端连接（并缓存）
- 写入新的加密 Cookie（内容相同，因为凭据相同）
- 返回 200 OK

这不会造成正确性问题，只是：
- 可能建立了多余的连接（浪费后端资源）
- 第二个请求的 Cookie 会覆盖第一个，但内容相同所以没区别
- 缓存中可能有相同 key 的条目被覆盖（Set 操作是幂等的）

#### 8.6.2 不同账号的并发登录

如果两个请求用不同账号几乎同时登录：
- 各自建立自己的连接，缓存 key 不同 → 互不干扰
- 浏览器的 Cookie 会被"最后一个返回的"覆盖
- 用户最终看到的是最后完成的那个账号

这在单用户场景下没有意义，但在共享浏览器的极端场景下可能出现。

### 8.7 并发语义总结表

| 并发场景 | 行为 | 语义级别 |
|---|---|---|
| 注销 + 正在进行的请求 | 请求继续按旧身份执行，不受注销影响 | 请求级快照 |
| 账号切换 + 正在进行的请求 | 请求继续按旧身份执行，后续请求用新身份 | 请求级快照 |
| 账号切换后旧连接 | 留在 AppCache 中直到 TTL 过期，不会被新身份命中 | 缓存隔离 |
| 同一后端不同账号 | AppCache key 不同，完全隔离 | 哈希隔离 |
| 并发登录（同账号） | 都成功，可能建多余连接，最终结果一致 | 最终一致 |
| 并发登录（不同账号） | Cookie 被最后一个覆盖，服务端两边都有缓存 | 竞态但安全 |
| 密钥变更 + 进行中请求 | 请求继续执行，后续新请求全部失效 | 请求级快照 |

### 8.8 设计权衡与潜在风险

#### 8.8.1 设计优点

1. **无状态服务端**：不存会话数据，水平扩展容易
2. **请求级隔离**：每个请求身份确定，不会中途变，避免复杂的锁
3. **缓存性能**：AppCache + TTL 大幅减少建连开销
4. **登出快速**：异步关连接，用户体验好

#### 8.8.2 潜在风险点

1. **短窗口旧连接可被利用**：如果攻击者在用户登出前偷走了 Cookie，登出后 TTL 内仍可能通过缓存重放？
   - 实际上不行，因为要利用缓存必须先通过 SessionStart，而 SessionStart 需要解密 Cookie
   - Cookie 已经被用户删除了，攻击者拿不到

2. **没有服务端强制下线能力**：
   - 管理员无法踢出某个特定用户
   - 只能通过改 SECRET_KEY 全体下线（代价大）
   - 或者通过 Config.Conn 白名单移除某个后端（影响该后端所有用户）

3. **goroutine 中的 req 使用**：
   - `SessionLogout` 的后台 goroutine 在 handler 返回后可能仍在访问 `req`
   - 理论上不符合 Go net/http 的使用约定
   - 实际中因为只是读 Cookie，出问题概率很低

4. **Close() 不检查引用计数**：
   - SFTP 有 wg 但 Close() 不等待 wg
   - 如果并发情况下一个请求在关连接，另一个还在用，可能导致后者出错
   - 实际中因为同一用户的并发请求会复用同一缓存连接，注销时 Close() 会影响正在进行的请求

---

## 九、三条潜在风险的代码级触发路径核对

### 9.1 风险 1：handler 返回后后台 goroutine 仍持有 req 引用

#### 9.1.1 完整调用链路

```
DELETE /api/session
  → NewMiddlewareChain(SessionLogout, [ApiHeaders, SecureHeaders, SecureOrigin, PluginInjector])
    → f(&app, &resw, req)                    // 执行 SessionLogout
    → req.Body.Close()                         // ← 主 goroutine 关闭 Body
    → go logger(&app, &resw, req)              // ← 主 goroutine 再次用 req

SessionLogout 内部:
  → go func() {                                // ← 后台 goroutine 启动
      middleware.SessionTry(func(c, _res, _req) {
        c.Backend.Close()
      })(ctx, res, req)                        // ← 后台 goroutine 持有 req
    }()
  → ... Set-Cookie 循环 (也读 req.Cookie) ...
  → SendSuccessResult(res, nil)                // ← 主 goroutine 返回
```

#### 9.1.2 后台 goroutine 对 req 的具体操作

`SessionTry` 内部 `server/middleware/session.go:85-94` 按序执行：

| 步骤 | 代码位置 | 操作 | 是否安全 |
|---|---|---|---|
| `_extractShare(req)` | `session.go:87` | 读 `req.URL.Query().Get("share")` + `mux.Vars(req)["share"]` | **安全**：URL 和 Vars 在 req 生命周期内不会变 |
| `_extractAuthorization(req)` | `session.go:88` | 循环调用 `req.Cookie(CookieName(i))` | **有风险**：见下 |
| `_extractSession(req, ctx)` | `session.go:89` | 不直接读 req（读 ctx.Authorization） | **安全** |
| `_extractBackend(req, ctx)` | `session.go:90` | 不直接读 req（读 ctx.Session） | **安全** |

#### 9.1.3 `req.Cookie()` 的实际安全性

Go 标准库 `net/http` 中 `req.Cookie(name)` 的实现：
- 遍历 `req.Header["Cookie"]` 解析出目标 cookie
- 纯只读操作，不修改 req 的任何字段
- `req.Header` 在请求创建时设置，之后不会被 Go 运行时修改

**主 goroutine 对 req 的修改**：
- `NewMiddlewareChain` 在 handler 返回后执行 `req.Body.Close()`（`middleware/index.go:32-34`）
- 这只影响 `req.Body`，不影响 `req.Header` 或 `req.Cookie()`

**结论**：后台 goroutine 读 `req.Cookie()` 时主 goroutine 可能已经 Close 了 Body，但 `Cookie()` 不依赖 Body，只依赖 Header。Header 在整个请求生命周期内不变。

#### 9.1.4 真正的危险点：`res` 和 `ctx` 的并发写

| 对象 | 主 goroutine | 后台 goroutine | 冲突？ |
|---|---|---|---|
| `res` (ResponseWriter) | `SendSuccessResult` → WriteHeader + Write | `SessionTry` 内部不写 res（fn 只调 Close） | **无冲突** |
| `ctx` | 不修改（注销不走 SessionStart，ctx.Session 为空） | SessionTry 写 `ctx.Share`/`ctx.Authorization`/`ctx.Session`/`ctx.Backend` | **无冲突**（主不读这些字段） |
| `req` | `req.Body.Close()` + `req.Cookie()` | `req.Cookie()` | **理论竞态**但实际无影响（见上） |

#### 9.1.5 终态判定

**不会 panic，不会返回异常状态码。**

- 后台 goroutine 的 `SessionTry` 是容错版（所有 err 被 `_` 忽略）
- 即使 `_extractAuthorization` 解析失败，返回空字符串
- 即使 `_extractSession` 解密失败，返回空 map
- 即使 `_extractBackend` 失败，返回 nil Backend → `c.Backend != nil` 检查跳过 Close
- 唯一可能的外在表现：**Close() 没被调用**（如果 Cookie 在主 goroutine 中已被清除导致后台读不到），旧连接留在 AppCache 等 TTL 过期

#### 9.1.6 另一个真实问题

`NewMiddlewareChain` 在 handler 返回后执行 `req.Body.Close()` `middleware/index.go:32-34`。
但 `SessionLogout` 走的中间件链包含 `BodyParser` 吗？不包含 `server/routes.go:28-29`。
所以 Body 在 `SessionLogout` 中没被读，Close 一个未读的 Body 是安全的。

---

### 9.2 风险 2：Close() 不查引用计数时并发请求的错误传播

#### 9.2.1 触发场景

同一用户浏览器打开标签页 A（正在下载大文件）和标签页 B（点击注销），两者使用同一个 SFTP 连接（因为 params 哈希相同，命中同一个 AppCache 条目）。

```
时间轴 →
  │
  ├─ 请求 A (GET /api/files/cat?path=/bigfile)
  │    └─ SessionStart → SftpCache.Get(params) 命中缓存 → 取出 *Sftp{SSHClient, SFTPClient}
  │    └─ SFTPClient.OpenFile("/bigfile", O_RDONLY) → 开始读取
  │    └─ b.SFTPClient.ReadDir / OpenFile / ... 操作进行中
  │
  ├─ 请求 B (DELETE /api/session)
  │    └─ go func() {
  │           SessionTry → SftpCache.Get(params) 命中同一缓存
  │           → 取出同一个 *Sftp{SSHClient, SFTPClient}
  │           → SFTPClient.Close()    // 关闭 sftp.Client
  │           → SSHClient.Close()     // 关闭底层 ssh.Client
  │         }()
  │
  └─ 请求 A 的下一次 sftp read 调用...
       → 底层 SSH 连接已关闭
       → ??? 什么错误
```

#### 9.2.2 SFTP 后端：错误传播链路

**第一步**：`ssh.Client.Close()` 被调用（`server/plugin/plg_backend_sftp/index.go:349`）

这会关闭底层 TCP 连接，并设置 `ssh.Client` 内部状态为 closed。

**第二步**：请求 A 的下一次 SFTP 操作（如 `ReadDir`/`OpenFile`/`Read`）通过 `tracedClient` 调用 `sftp.Client` 方法

`tracedClient`（`server/plugin/plg_backend_sftp/tracing.go:20-110`）只是包装了 span，最终调用 `t.Client.XXX()`

**第三步**：`sftp.Client` 发送请求时发现底层连接已关闭

`pkg/sftp` 库中，当底层连接关闭后：
- 发送操作返回 `io.EOF`（如果 SSH channel 已关闭）
- 或 `*sftp.StatusError{Code: 4, msg: "Failure"}`（如果 SFTP 服务器返回错误）
- 或 `net: use of closed network connection`（如果 TCP socket 被关）

**第四步**：错误经过 `b.err(e)` 映射 `server/plugin/plg_backend_sftp/index.go:357-376`

```go
func (b Sftp) err(e error) error {
    f, ok := e.(*sftp.StatusError)
    if ok == false {
        if e == os.ErrNotExist {
            return ErrNotFound           // → 404
        }
        return e                         // ← 未经映射的原生错误
    }
    switch f.Code {
    case 0:  return nil
    case 1:  return NewError("There's nothing more to see", 404)
    case 2:  return NewError("Does not exist", 404)
    case 3:  return NewError("Permission denied", 403)
    case 4:  return NewError("Failure", 409)
    ...
    }
}
```

SSH 连接关闭产生的错误（`io.EOF` / `net: use of closed network connection`）**不是** `*sftp.StatusError`，也不等于 `os.ErrNotExist`，所以 `b.err()` 返回**未经映射的原生 error**。

**第五步**：ctrl 层调用 `SendErrorResult(res, err)` `server/common/response.go:85-102`

```go
obj, ok := err.(interface{ Status() int })
if ok == true {
    res.WriteHeader(obj.Status())   // 有 Status() 方法 → 用它的状态码
} else {
    res.WriteHeader(500)            // 没有 Status() 方法 → 500
}
```

`io.EOF` 和 `net.OpError` 都**没有** `Status() int` 方法 → **HTTP 500**。

对于 `*sftp.StatusError{Code: 4}` 的情况：`b.err()` 将其映射为 `NewError("Failure", 409)` → `AppError` 有 `Status()` 返回 409 → **HTTP 409**。

#### 9.2.3 各操作的具体错误表现

| 操作 | 被关连接后触发的 Go 错误 | 经过 b.err() 后 | 最终 HTTP 状态码 | 用户感知 |
|---|---|---|---|---|
| `Ls` (ReadDir) | `io.EOF` 或 `net.OpError` | 原样透传 | **500** | 列表加载失败 |
| `Cat` (OpenFile) | `io.EOF` 或 `ssh.ChannelClosed` | 原样透传 | **500** | 文件打开失败 |
| `Cat` 读取中 (Read) | `io.EOF` (来自 io.Copy) | **不经过 b.err()** | **连接直接断开** | 下载中断，响应截断 |
| `Save` (OpenFile+Write) | `io.EOF` 或 `net.OpError` | 原样透传 | **500** | 保存失败 |
| `Mv` (Rename) | `*sftp.StatusError{Code:4}` | `NewError("Failure", 409)` | **409** | 重命名冲突 |
| `Rm` (Remove) | `*sftp.StatusError{Code:4}` | `NewError("Failure", 409)` | **409** | 删除冲突 |
| `Stat` | `io.EOF` | 原样透传 | **500** | 文件信息获取失败 |

**特别注意 Cat 读取中的情况**：

`FileCat` handler（`server/ctrl/files.go`）中文件下载使用 `io.Copy(res, remoteFile)`，如果 Read 中途连接断开：
- `io.Copy` 返回 `io.EOF` 或 `io.ErrUnexpectedEOF`
- 但此时 HTTP 响应头已经写出去（200 OK + Content-Length），数据已经部分传输
- handler 无法再改状态码
- 客户端（浏览器）看到的是一个**不完整的下载**（Content-Length 不匹配）
- 浏览器通常会报"网络错误"或"下载中断"

#### 9.2.4 其他后端的表现

| 后端 | Close() 语义 | 被关闭后的并发请求表现 |
|---|---|---|
| **S3** | 无状态连接（每次 newSession），Close 未实现 | **不受影响**（S3 不缓存连接对象，只缓存 Region） |
| **WebDAV** | 无缓存，每次新建 HTTP 连接 | **不受影响**（无共享连接） |
| **Samba** | 关闭 SMB session | 后续操作返回 SMB 错误 → 映射为 500 |
| **FTP** | `f.client.Close()` 关闭控制连接 | 数据传输中断 → `goftp` 返回 `net.OpError` → 500 |
| **Git** | 清理临时 clone 目录 | 后续操作报 "repository not found" → 500 |

#### 9.2.5 与 OnEvict 的对比

注意 SFTP 的 `OnEvict` 回调 `server/plugin/plg_backend_sftp/index.go:28-41`：

```go
SftpCache.OnEvict(func(key string, value interface{}) {
    c := value.(*Sftp)
    c.wg.Wait()       // ← 等待所有引用释放
    c.Close()          // ← 然后才关闭
})
```

缓存 TTL 过期时的清理是**安全的**（等待 wg → 等所有请求完成再关）。问题只出现在 `SessionLogout` 主动调用 `Close()` 时，因为**不走 OnEvict**，不等待 wg。

#### 9.2.6 结论

**并发请求不会拿到 "closed" 或特定业务错误码。** 最常见的表现是：

- 元数据操作（Ls/Stat/Mkdir/Mv/Rm）→ **HTTP 500**（原生 Go error 未经映射）
- 下载操作（Cat）→ **响应截断**（已写 200 头，数据中途断开）
- 上传操作（Save）→ **HTTP 500**

前端无法区分"连接被注销关闭"和"网络故障/后端异常"，统一表现为操作失败。

---

### 9.3 风险 3：缺管理员粒度强制下线在多租户场景的数据穿透

#### 9.3.1 Filestash 的多租户模型

Filestash 的"租户"概念不是内置的，而是通过以下组合实现的：

1. **后端白名单** `Config.Conn`：限定可连接的存储后端（type/hostname/path/url）
2. **认证中间件插件** `AuthenticationMiddleware`：限制谁能登录（如 WordPress SSO、LDAP、passthrough）
3. **授权中间件插件** `AuthorisationMiddleware`：限制登录后能访问哪些路径
4. **共享链接** `Share`：带密码/过期时间的受限访问

**关键缺失**：没有"用户会话注册表"——服务端不知道当前有哪些活跃用户。

#### 9.3.2 管理员现有的踢人手段

| 手段 | 代码路径 | 影响范围 | 粒度 |
|---|---|---|---|
| 改 `general.secret_key` | `config.go:71` | 全体用户全部失效 | **全量** |
| 删 `Config.Conn` 条目 | `model/files.go:9-51` | 该后端所有用户无法**新建**连接 | **后端级** |
| 改认证中间件配置 | `ctrl/session.go:220-236` | 新用户无法通过该中间件登录 | **认证方式级** |
| 删共享链接 | `model/share.go` | 使用该链接的**新**访问被拒 | **链接级** |

**以上均无法踢掉已持有有效 Cookie 的特定用户。**

#### 9.3.3 数据穿透路径分析

**场景**：多租户 SFTP 后端，不同用户分配不同 chroot path

```
用户 A: {type:sftp, hostname:sftp.example.com, username:tenant_a, path:/data/tenant_a/}
用户 B: {type:sftp, hostname:sftp.example.com, username:tenant_b, path:/data/tenant_b/}
```

**穿透路径 1：共享链接跨租户泄露**

`ShareUpsert` `server/ctrl/share.go:44-58` 创建共享链接时：
```go
s := Share{
    Auth: func() string {
        if ctx.Share.Id == "" {
            str := ""
            index := 0
            for {
                cookie, err := req.Cookie(CookieName(index))
                // ...拼接完整的加密会话 token...
                str += cookie.Value
            }
            return str  // ← 完整的用户 A 加密会话存入 Share.Auth
        }
        return ctx.Share.Auth
    }(),
}
```

这意味着共享链接的 `Auth` 字段存储的是**创建者完整的加密会话**，包含密码。如果用户 A 创建了指向 `/data/tenant_a/` 的共享链接并分享给用户 B，用户 B 通过该链接访问时：

- `_extractShare` 解密 `Share.Auth` → 得到用户 A 的完整 session params
- `_extractSession` 用这个 session 创建后端连接 → **以用户 A 的身份操作**
- `_extractBackend` 初始化 Backend 时 path 被 Share 的 path 覆盖 → 限制在用户 A 的 chroot 内

**跨租户穿透可能性**：如果 `Share.Path` 配置为 `/`（而非 `/data/tenant_a/`），或者 chroot 在 SFTP 服务器端未严格限制，用户 B 可以通过用户 A 的凭据访问用户 A 的数据。

但这是**设计如此**（共享=授权），不是漏洞。问题是管理员无法撤回已发出的共享链接所包含的凭据——只能删共享链接阻止新访问，已通过共享链接建立自己 Cookie 的人不受影响。

**穿透路径 2：Cookie 盗用无法单点清除**

假设用户 A 的 Cookie 被盗（XSS、中间人等），攻击者获得加密 token。管理员发现后：
- 无法只使用户 A 的 Cookie 失效
- 改 SECRET_KEY → 所有用户全部下线
- 删 SFTP 后端配置 → 阻止**新**连接，但 AppCache 中的旧连接仍在 TTL 内有效

**穿透路径 3：AppCache 跨请求复用**

同一 Filestash 实例上，如果用户 A 和用户 B 连接同一个 SFTP 服务器但用不同账号：
- 他们的连接在 AppCache 中是不同 key → 不互相影响
- 但如果**同一账号**被多人共享使用（如共享服务账号），AppCache 只建一个连接 → 所有人复用

```
用户 A 登录: {type:sftp, hostname:srv, username:shared, password:xxx}
用户 B 登录: {type:sftp, hostname:srv, username:shared, password:xxx}
→ params 哈希相同 → 命中同一 SftpCache 条目 → 共享同一个 SSH 连接
```

这不是漏洞，但在多租户场景下意味着：
- 用户 A 的文件操作和用户 B 的文件操作在同一个 SSH session 上交错
- SFTP 服务器端看到的都是同一个用户
- 审计日志无法区分是 A 还是 B

#### 9.3.4 管理员粒度强制下线的缺失带来的具体影响

| 场景 | 缺失能力 | 后果 |
|---|---|---|
| 用户密码泄露 | 无法使该用户现有 Cookie 失效 | 攻击者在 cookie_timeout 内（默认 1 周）持续访问 |
| 员工离职 | 无法远程清除其浏览器中的 Cookie | 前员工在 cookie_timeout 内可继续访问公司存储 |
| 共享链接滥用 | 无法使已通过链接创建 Cookie 的用户失效 | 只能删链接阻止新访问，已登录用户不受影响 |
| 合规审计 | 无法展示"已终止某用户所有会话" | 不满足某些合规要求（如 SOC2、GDPR 数据访问撤回） |
| APT 防御 | 无法快速隔离被入侵的会话 | 必须改 SECRET_KEY 影响全体用户，造成大面积服务中断 |

#### 9.3.5 根本原因

`_extractSession` `server/middleware/session.go:262-315` 的认证逻辑**完全自包含**：

```go
str, err = DecryptString(SECRET_KEY_DERIVATE_FOR_USER, ctx.Authorization)
// 只检查：1) 能否解密  2) timestamp 是否超过 1 年
// 不检查：1) 会话是否被管理员撤销  2) 用户是否仍被授权
```

没有"会话黑名单"或"令牌撤销列表"的概念。每个请求独立验证，不查询任何服务端状态。

#### 9.3.6 可行的缓解方案方向（代码层面）

1. **增加会话撤销表**：在 SQLite 中添加 `revoked_sessions` 表，`_extractSession` 解密后检查 `GenerateID(session)` 是否在撤销列表中
2. **利用 AuthorisationMiddleware 插件**：在每次请求时向外部认证服务（如 LDAP/IdP）验证用户是否仍有效，但当前插件只在 `Ls`/`Cat` 等操作上检查，不在 SessionStart 上检查
3. **缩短 cookie_timeout**：将默认 1 周缩短到数小时，缩小攻击窗口
4. **增加 session_id 概念**：在 session map 中嵌入随机 ID，管理员可按 ID 撤销

---

## 十、添加会话黑名单功能：最小改动方案代码核对

### 10.1 前置结论：StorageEngine 接口与现有存储结构

**Filestash 不存在 `StorageEngine` / `IStorage` 抽象接口。**

当前持久化层在 `server/model/index.go:10-38` 直接暴露全局变量 `DB *sql.DB`（SQLite3），所有 CRUD 操作在 `model` 包内直接使用 `DB.Prepare()` + `stmt.Exec()`。

现有数据表：
| 表名 | 用途 | 位置 |
|---|---|---|
| `Location` | 共享链接关联的后端+路径 | `index.go:20` |
| `Share` | 共享链接主体（含完整加密会话快照 `auth` 字段） | `index.go:24` |
| `Verification` | 邮件验证码临时表（带过期时间） | `index.go:28-32` |

**因此，无需新增接口，只需在 `server/model/index.go` 的 `init()` 中新增 `RevokedSession` 表，并在 `server/model/` 包新增查询/写入/删除函数。**

---

### 10.2 黑名单键的选择：`GenerateID` vs `session_id` vs `完整 params 哈希`

| 方案 | 粒度 | 优点 | 缺点 |
|---|---|---|---|
| **`GenerateID(session)`**（推荐用于"用户级踢人"） | **用户+后端**（排除 password/path/timestamp，含 username/hostname/type） | 同一用户所有路径所有 Cookie 全部失效，管理员只需知道用户名即可 | 无法精细到单个设备/单个会话；用户重新登录新密码同样被踢 |
| **新增随机 `session_id` 字段** | 单条会话（单设备单浏览器） | 最精细，可踢出单个泄露的会话 | 需要修改 SessionAuthenticate 写入 session_id，修改量略大 |
| **完整 params 哈希（AppCache key）** | 精确到密码+路径 | 与 AppCache key 对齐 | 同用户不同密码/路径各自独立，管理员难操作 |

**最小改动推荐**：以 `GenerateID(session)` 作为黑名单主键。理由：
1. 无需修改现有 Cookie 内容（不用加 session_id 字段）
2. 现有 `GenerateID()` 函数已存在于 `server/common/crypto.go:193-218`
3. 语义自然："踢出某个用户在某个后端的所有会话"

---

### 10.3 鉴权全流程需要同步修改的代码点

#### 10.3.1 数据层改动（server/model/）

**改动点 1：`server/model/index.go` 的 `init()` 内**

在现有 `CREATE TABLE IF NOT EXISTS Verification` 之后追加：
```sql
CREATE TABLE IF NOT EXISTS RevokedSession(
    gen_id VARCHAR(40) PRIMARY KEY,
    revoked_at DATETIME DEFAULT (datetime('now')),
    until DATETIME DEFAULT (datetime('now', '+365 days'))
)
CREATE INDEX IF NOT EXISTS idx_revoked_until ON RevokedSession(until)
```

并在 `autovacuum()`（`index.go:41-46`）中增加过期清理：
```go
if stmt, err := DB.Prepare("DELETE FROM RevokedSession WHERE until < datetime('now')"); err == nil {
    stmt.Exec()
}
```

**改动点 2：在 `server/model/share.go` 或新建 `server/model/session.go` 增加 3 个函数**

```go
// 检查是否在黑名单（每请求调用一次，需高性能）
func IsSessionRevoked(genID string) (bool, error)

// 管理员加入黑名单
func RevokeSession(genID string, durationHours int) error

// 管理员撤销黑名单（可选）
func UnrevokeSession(genID string) error
```

性能考虑：`IsSessionRevoked` 在 `SessionStart` 中每请求一次，建议用内存布隆过滤器或 `sync.Map` 做一级缓存，SQLite 查询兜底。

#### 10.3.2 鉴权中间件改动（server/middleware/session.go）

**改动点 3：`_extractSession` 函数末尾（`middleware/session.go:306-314`）**

在 timestamp 校验通过后、`return session, err` 之前插入黑名单检查：

```go
// 位置：在 timestamp 校验之后
if ctx.Share.Id == "" {   // 共享链接走另一套路径，不检查用户级黑名单
    genID := GenerateID(session)
    if revoked, err := model.IsSessionRevoked(genID); err != nil {
        return session, ErrInternal
    } else if revoked {
        Log.Warning("middleware::session 'revoked session gen_id=%s'", genID)
        RecoverFromBadCookie(res)   // ← 注意：_extractSession 不直接拿 res，需要调整签名
        return session, ErrSessionRevoked
    }
}
```

**问题**：`_extractSession` 当前签名是 `func _extractSession(req, ctx) (map, error)`，拿不到 `res`。两个选项：
- **选项 A（最小签名变更）**：给 `_extractSession` 加一个 `res http.ResponseWriter` 参数
- **选项 B（更干净）**：把黑名单校验移到 `SessionStart` 中，在 `_extractSession` 返回之后、`_extractBackend` 之前。

**推荐选项 B**，因为：
1. 不改 `_extractSession` 签名
2. 黑名单检查逻辑独立，未来容易替换为布隆过滤器

位置：`SessionStart` 函数 `middleware/session.go:66-78`：
```go
if ctx.Session, err = _extractSession(req, ctx); err != nil {
    RecoverFromBadCookie(res)
    SendErrorResult(res, err)
    return
}
// ← 在这里插入黑名单检查
if ctx.Share.Id == "" && len(ctx.Session) > 0 {
    genID := GenerateID(ctx.Session)
    if revoked, err := model.IsSessionRevoked(genID); err == nil && revoked {
        RecoverFromBadCookie(res)
        SendErrorResult(res, ErrSessionRevoked)   // 需新增错误类型
        return
    }
}
```

**改动点 4：新增错误类型 `ErrSessionRevoked`**

在 `server/common/error.go` 或相关位置：
```go
var ErrSessionRevoked = NewError("This session has been revoked by an administrator", 401)
```
状态码选择 401（而非 403），因为语义是"认证失效，需要重新登录"，前端会自动跳转登录页。

#### 10.3.3 SessionTry 容错路径（server/middleware/session.go:85-94）

`SessionTry` 忽略所有错误（全部 `_`）。它用于 `SessionLogout` 的后台关连接 goroutine。

**判断：不需要改。** 理由：
- `SessionTry` 不处理真实业务请求，只是尝试关连接
- 即使是已撤销的会话，关连接操作本身不泄露数据，反而是安全的
- 如果加黑名单检查，可能导致 `SessionLogout` 的后台关连接跳过（因为此时 Cookie 可能已被清除，genID 都拿不到）

#### 10.3.4 共享链接路径（_extractSession 的 Share 分支）

**判断：不需要加黑名单检查。** 理由：
- 共享链接的 `ctx.Share.Auth` 是创建者的加密会话快照，通过 `DecryptString(SECRET_KEY_DERIVATE_FOR_USER, ctx.Share.Auth)` 解密
- 如果创建者的会话被撤销，共享链接也应该失效，因为共享链接本质上是"创建者身份的受限代理"
- 但共享链接有自己的 `Share.IsValid()` 机制（密码/过期时间），管理员删共享链接即可
- 如果业务要求"用户被踢出后其所有共享链接也失效"，可在 `_extractShare` 之后、`_extractSession` 之前检查创建者的 genID 是否在黑名单

**可选改动**（如果业务要求）：在 `Share` 表新增 `owner_gen_id VARCHAR(40)` 字段，创建时记录 `GenerateID(创建者的 session)`，在 `ShareGet` 返回时检查。超出最小改动范围。

#### 10.3.5 AdminOnly 中间件（server/middleware/session.go:28-55）

AdminOnly 独立使用 `AdminToken`，不经过 `SessionStart`，也不使用 `GenerateID`。

**判断：需要独立考虑，但属于管理员体系的另一个问题。** 如果只实现用户级踢人，暂时不需要动。

#### 10.3.6 新增管理后台 API 路由（server/routes.go）

在 `admin` 子路由（`routes.go:35-50`）新增：

```go
middlewares = []Middleware{ApiHeaders, AdminOnly, SecureOrigin, BodyParser, PluginInjector}
admin.HandleFunc("/session/revoke", NewMiddlewareChain(AdminSessionRevoke, middlewares)).Methods("POST")
admin.HandleFunc("/session/revoke/{gen_id}", NewMiddlewareChain(AdminSessionUnrevoke, middlewares)).Methods("DELETE")
admin.HandleFunc("/session/revoked", NewMiddlewareChain(AdminSessionRevokedList, middlewares)).Methods("GET")
```

对应的 handler 在 `server/ctrl/admin.go` 新增：
- `AdminSessionRevoke`：从 body 读 `gen_id` + `duration_hours`，调用 `model.RevokeSession`
- `AdminSessionUnrevoke`：从 path 读 gen_id，调用 `model.UnrevokeSession`
- `AdminSessionRevokedList`：列出黑名单内所有条目

**额外挑战**：管理员怎么知道 gen_id？GenerateID 需要知道 username/hostname/type 等字段。需要前端页面根据审计日志（现有 `/admin/api/audit`）推导，或在审计日志中直接记录 gen_id。

#### 10.3.7 审计日志改动（可选但推荐）

现有审计日志在 `server/ctrl/admin.go` 的 `FetchAuditHandler`。如果审计日志中不含 gen_id，管理员无法知道要踢谁。

**改动点**：在登录成功处（`ctrl/session.go` 的 `SessionAuthenticate` 末尾）写入审计日志时带上 gen_id。

---

### 10.4 AppCache 命中后是否需要黑名单校验？

#### 10.4.1 执行顺序回顾

```
SessionStart:
  _extractShare       → 解析共享链接
  _extractAuthorization → 读 Cookie / Bearer Header
  _extractSession     → 解密 + timestamp 校验  ← 【黑名单检查放在这里】
  _extractBackend     → model.NewBackend → Backend.Init
                         → 各后端插件的 AppCache.Get(params)  【AppCache 命中】
```

AppCache 读取发生在 `_extractBackend`（第 4 步），**在黑名单检查之后**。

#### 10.4.2 语义边界分析

**场景**：用户 A 被管理员加入黑名单。此时有一个用户 A 的并发请求 C 正在进行中：

| 时间 | 请求 C 的步骤 | 黑名单已生效？ | 行为 |
|---|---|---|---|
| T1 | `_extractSession` 解密 session | 否 | 解密通过 |
| T2 | 管理员将 gen_id_A 加入黑名单 | 是 | 已写入 DB |
| T3 | `_extractBackend` 从 SftpCache 取出旧连接 | 是（但 AppCache 不查） | 拿到连接对象 |
| T4 | handler 执行 Ls/Cat 等操作 | 是 | 操作正常完成 |

**在当前执行顺序下（黑名单在 `_extractSession` 之后立即检查），如果黑名单检查在 T1 通过，即使 T2 加入黑名单，T3-T4 仍正常执行**——这是请求级快照语义的固有特性，不是漏洞。

**但如果考虑极端情况**：将黑名单检查移到 AppCache 命中之后（即 `_extractBackend` 返回后），能否更早拦截？

**结论：不需要在 AppCache 命中后再加检查。** 理由 3 条：

**理由 1：请求级快照语义一致性**
- 一旦请求通过 `SessionStart`（`_extractSession` + `_extractBackend`），其身份应该在整个请求中保持一致
- 在 handler 执行中间突然改变身份会造成难以调试的部分成功/部分失败
- 当前设计保证：要么请求在 SessionStart 阶段就被拒绝（401），要么请求按已确立的身份完整执行

**理由 2：AppCache 是性能优化，不影响认证结果**
- AppCache 只是"有 params → 拿旧连接"的缓存，不会跳过认证
- 要命中 AppCache，必须先通过 `_extractSession` 的解密 + timestamp + 黑名单检查 → 然后才能带着正确的 params 走到 `Init`
- 攻击者如果绕过了 SessionStart，根本不需要 AppCache，直接能调任何后端操作

**理由 3：如果要加检查，位置应该在 SessionStart，不应该分散在各后端插件**
- 各后端插件的 `Init` 里读取 AppCache（SFTP 的 `SftpCache.Get`、Samba 的 `SambaCache.Get` 等）
- 在每个插件里都加黑名单检查 = 重复代码 + 容易遗漏
- 集中在 `SessionStart` 中 `_extractSession` 之后检查一次 = 所有后端自动受益
- 性能影响可忽略：SQLite 主键查询 + 可加布隆过滤器缓存

**唯一例外场景：长轮询 / SSE / 分块下载的超长时间请求**

如果有一个请求持续运行 30 分钟（如下载超大文件），在这 30 分钟内会话被撤销，这个请求不会被中断。这符合"请求级快照"语义。

如果业务需要"撤销后立即中断正在进行的长请求"，需要额外做：
1. 给 `App` 的 `ctx.Context` 加定时查询黑名单的包装（如 `context.WithCancel` + ticker）
2. 或者在 AppCache 命中时给后端连接注入"撤销检查钩子"
3. 超出最小改动范围

---

### 10.5 完整改动清单（最小方案）

| 序号 | 文件 | 改动内容 | 影响范围 |
|---|---|---|---|
| 1 | `server/model/index.go` | 新增 `RevokedSession` 表 DDL + 索引；`autovacuum` 加过期清理 | 数据初始化 |
| 2 | `server/model/session.go`（新建） | 实现 `IsSessionRevoked` / `RevokeSession` / `UnrevokeSession` / `RevokedSessionList` | 数据层 |
| 3 | `server/common/error.go` | 新增 `ErrSessionRevoked = NewError("...", 401)` | 错误定义 |
| 4 | `server/middleware/session.go` | `SessionStart` 中在 `_extractSession` 之后、`_extractBackend` 之前调用 `model.IsSessionRevoked`，返回时调 `RecoverFromBadCookie` + `SendErrorResult(res, ErrSessionRevoked)` | 核心拦截点 |
| 5 | `server/ctrl/admin.go` | 新增 `AdminSessionRevoke` / `AdminSessionUnrevoke` / `AdminSessionRevokedList` handler | 管理功能 |
| 6 | `server/routes.go` | 注册 3 条 `/admin/api/session/revoke*` 路由，加 `AdminOnly` 中间件 | 路由注册 |
| 7 | 前端 admin 页面（可选） | 新增会话管理 UI：展示审计日志中的 gen_id、提供撤销/恢复按钮 | 管理界面 |

**不改的文件**：
- `server/common/crypto.go`：`GenerateID` 直接复用
- `server/ctrl/session.go`：`SessionAuthenticate` / `SessionLogout` / `SessionGet` 均不改
- 各后端插件（sftp/s3/webdav/...）：无需修改
- `server/common/cache.go` / AppCache：无需修改
- `server/middleware/session.go` 中 `SessionTry` / `CanManageShare`：不改
- Share 相关代码：不改（超出最小范围）

---

### 10.6 安全语义保证

| 安全属性 | 是否满足 | 说明 |
|---|---|---|
| 撤销后新请求立即被拒 | ✅ | 下一次请求的 SessionStart → 黑名单检查 → 401 + 清 Cookie |
| 撤销后 AppCache 旧连接可被旧 Cookie 复用 | ❌（不允许） | 旧 Cookie 已无法通过 SessionStart，拿不到走到 AppCache 所需的明文 params |
| 撤销后进行中的长请求立即中断 | ⚠️（可选） | 最小方案按请求级快照，不中断；如需要可加 Context 包装 |
| 撤销同一用户不同密码的所有会话 | ✅ | GenerateID 排除 password，同 username 不管密码都命中同一黑名单 |
| 撤销后用户重新登录（改了密码） | ⚠️（需注意） | GenerateID 仍相同 → 新登录同样被踢。解决：Unrevoke，或限定 until 时长，或改用 session_id 方案 |
| 共享链接创建者被撤销后链接失效 | ❌（最小方案） | 共享链接走独立路径；如需要需改 Share 表加 owner_gen_id |
| 并发撤销/登录的一致性 | ✅ | SQLite 主键冲突保证同一 gen_id 不重复写入 |

---

## 十一、session_id + Revoked 方案上线协同分析

### 11.1 session_id 的写入位置与格式

#### 11.1.1 写入点 1：`SessionAuthenticate`（直接登录）

`server/ctrl/session.go:52-135`，在 `ctx.Body["timestamp"] = time.Now().Format(time.RFC3339)` 之后追加：

```go
ctx.Body["session_id"] = RandomString(20)  // 使用 crypto/rand 的 RandomString
```

写入后 session map 结构变为：
```
{
  "type":      "sftp",
  "hostname":  "sftp.example.com",
  "username":  "alice",
  "password":  "secret",
  "path":      "/",
  "timestamp": "2026-06-18T10:30:00Z",
  "session_id":"aB3kX9mP2nQ8vR5wL7y",   ← 新增 20 字符随机串
  "session":   "{...}",                    ← SSO 扩展会话（仅 IdP 登录时）
}
```

#### 11.1.2 写入点 2：`SessionAuthMiddleware`（SSO/OAuth 登录）

`server/ctrl/session.go:427`，在 `mappingToUse["timestamp"] = time.Now().Format(time.RFC3339)` 之后追加：

```go
mappingToUse["session_id"] = RandomString(20)
```

两处写入确保所有登录路径（直接表单 + SSO 回调）都生成 session_id。

#### 11.1.3 对 Cookie 分片的影响

session_id 增加 33 字节（key "session_id" = 10 + value 20 + JSON 分隔符 ~3 = ~33）。当前加密后 Cookie 通常 < 1KB，距离 3800 字节分片阈值很远。**不增加分片数。**

#### 11.1.4 对 GenerateID 的影响

`GenerateID` `server/common/crypto.go:193-218` 排除 `"session"` 和 `"timestamp"`，但**不排除** `"session_id"`。需要补加：

```go
switch key {
case "password":
case "path":
case "session":
case "timestamp":
case "session_id":   // ← 新增，确保 GenerateID 不受随机 session_id 影响
```

如果不排除，同一用户每次登录 GenerateID 不同 → 无法按 GenerateID 聚合同一用户的所有会话。但在 session_id 方案下，我们用 session_id 作为黑名单主键，GenerateID 仍用于关联共享链接等场景，**必须保持稳定**。

---

### 11.2 旧设备何时被踢出

#### 11.2.1 踢出触发路径

管理员调用 `POST /admin/api/session/revoke`，body 为 `{ "session_id": "aB3kX9mP2nQ8vR5wL7y" }`。

执行时序：

```
管理员 → POST /admin/api/session/revoke → AdminOnly 中间件验证 → AdminSessionRevoke handler
  → model.RevokeSession(session_id, durationHours)
    → INSERT INTO RevokedSession(gen_id, revoked_at, until) VALUES(?, datetime('now'), ...)
```

#### 11.2.2 旧设备的下一次请求

```
旧设备 → GET /api/files/ls → SessionStart
  → _extractAuthorization → 从 Cookie 拼接加密 token
  → _extractSession → 解密 → json.Unmarshal → session["session_id"] = "aB3kX9mP2nQ8vR5wL7y"
  → 黑名单检查: model.IsSessionRevoked("aB3kX9mP2nQ8vR5wL7y")
    → SELECT COUNT(*) FROM RevokedSession WHERE gen_id = ? AND until > datetime('now')
    → 返回 true
  → RecoverFromBadCookie(res)  → Set-Cookie: auth=; MaxAge=-1
                                 → Set-Cookie: auth1=; MaxAge=-1
                                 → ...（逐片清除）
  → SendErrorResult(res, ErrSessionRevoked)  → HTTP 401
```

**旧设备在下次请求时被踢出，不是立即踢出。** 具体时间取决于旧设备的活跃度：
- 旧设备正在浏览文件 → 下次点击/刷新即被踢出（通常几秒内）
- 旧设备闲置 → 直到用户再次操作才被踢出
- 旧设备下载大文件中 → 当前请求不受影响，下次请求被踢出（请求级快照语义）

#### 11.2.3 与 GenerateID 方案的对比

| 维度 | GenerateID 方案 | session_id 方案 |
|---|---|---|
| 踢出粒度 | 同一用户同一后端所有设备 | 单设备单浏览器 |
| 旧设备踢出时机 | 同上 | 同上 |
| 新设备受影响？ | 是（同用户新登录也被踢） | **否**（新设备有新 session_id） |
| 管理员需知道什么 | username + hostname + type | session_id（从审计日志获取） |

---

### 11.3 单设备多 Cookie（分片）的清理顺序

#### 11.3.1 Cookie 分片写入顺序

`SessionAuthenticate` `server/ctrl/session.go:102-124` 写 Cookie：

```
循环 index=0, 1, 2, ...
  Set-Cookie: auth=前3800字节;   MaxAge=604800; Path=/api/
  Set-Cookie: auth1=后3800字节;  MaxAge=604800; Path=/api/
  Set-Cookie: auth2=剩余字节;    MaxAge=604800; Path=/api/
```

每个分片包含同一加密串的不同片段，**每个分片独立包含 session_id 的加密信息**（因为 session_id 在加密前的完整 JSON 中，不是分散在分片里）。

#### 11.3.2 Cookie 分片读取顺序

`_extractAuthorization` `server/middleware/session.go:160-173` 读 Cookie：

```go
index := 0
for {
    cookie, err := req.Cookie(CookieName(index))
    if err != nil { break }
    index++
    token += cookie.Value   // 按序拼接
}
```

**关键点**：如果只有 `auth1`（缺少 `auth`），`req.Cookie("auth")` 返回 err → 循环直接 break → token 为空 → 后续解密失败。分片是严格顺序的，缺首片则全废。

#### 11.3.3 黑名单踢出时的 Cookie 清理

`RecoverFromBadCookie` `server/common/recovery.go:10-21`：

```go
func RecoverFromBadCookie(res http.ResponseWriter) {
    index := 0
    for {
        _, err := req.Cookie(CookieName(index))  // ← 问题：此函数不接收 req
        ...
    }
}
```

实际上 `RecoverFromBadCookie` 的签名是 `func RecoverFromBadCookie(res http.ResponseWriter)`，它不读 req。让我确认其实际实现。

实际实现是逐序号删除固定名称的 Cookie，不依赖 req。这意味着**如果旧 Cookie 有 3 片（auth/auth1/auth2），而新 Cookie 只有 1 片（auth），清 Cookie 时可能只清了 auth，auth1/auth2 残留**。

#### 11.3.4 分片清理的完整方案

**改动点**：黑名单拦截时，需要清除所有可能的分片。由于不知道旧 Cookie 到底有几片，需要设置一个足够大的上限，或与 `SessionLogout` 保持一致（逐片尝试清除直到 req.Cookie 返回 err）。

在 `SessionStart` 的黑名单拦截点，我们持有 `req`，所以可以复用 `SessionLogout` 的逻辑：

```go
// 在黑名单拦截后
if revoked {
    // 清除所有分片 Cookie
    index := 0
    for {
        _, err := req.Cookie(CookieName(index))
        if err != nil { break }
        http.SetCookie(res, applyCookieRules(&http.Cookie{
            Name:   CookieName(index),
            Value:  "",
            MaxAge: -1,
            Path:   COOKIE_PATH,
        }, req))
        index++
    }
    SendErrorResult(res, ErrSessionRevoked)
    return
}
```

**清理顺序**：auth(0) → auth1 → auth2 → ... → 直到 req.Cookie 找不到下一个。这确保同一设备的所有分片被清除。

**残留风险**：如果在踢出请求之前，浏览器因为其他操作已经覆盖了部分分片（如新登录只写了 auth 而旧 Cookie 有 auth1），则 auth1 可能残留。但残留的 auth1 不完整，`_extractAuthorization` 读到它时因为缺首片，token 为空 → 不构成安全风险。

---

### 11.4 重新登录与强制登出时黑名单条目的生命周期

#### 11.4.1 场景矩阵

| 场景 | session_id 生成 | 黑名单影响 | 需要的处理 |
|---|---|---|---|
| 用户主动注销 | 不涉及（旧 session_id 仍在 Cookie 里直到浏览器删除） | 无 | 注销只清 Cookie，不涉及黑名单 |
| 用户主动注销后重新登录 | 生成新 session_id | 旧 session_id 不在黑名单 → 不影响 | 无需额外处理 |
| 管理员强制踢出某设备 | 不涉及 | 旧 session_id 被加入黑名单 | 已实现 |
| 管理员踢出后用户重新登录 | 生成新 session_id | 新 session_id 不在黑名单 → 正常登录 | 无需额外处理 ✅ |
| 用户改密码后重新登录 | 生成新 session_id | 旧 session_id 在黑名单（如果被踢）→ 旧设备被拒 | 无需额外处理 ✅ |
| 同一用户多设备 | 每设备独立 session_id | 只踢指定 session_id 的设备 | 精确控制 ✅ |

#### 11.4.2 关键优势：session_id 方案下重新登录不会被误杀

这是选择 session_id 而非 GenerateID 的核心理由。对比：

```
GenerateID 方案:
  管理员踢出 gen_id="abc123"（基于 username+hostname）
  用户改密码后重新登录 → GenerateID 仍为 "abc123" → 新会话也被拒绝 ❌
  必须先 Unrevoke → 用户才能登录 → 管理体验差

session_id 方案:
  管理员踢出 session_id="aB3kX9..."（旧设备的具体会话）
  用户重新登录 → 新 session_id="dF7mN2..." → 不在黑名单 → 正常登录 ✅
  无需 Unrevoke
```

#### 11.4.3 黑名单条目的过期机制

`RevokedSession` 表设计：

```sql
CREATE TABLE RevokedSession(
    session_id VARCHAR(40) PRIMARY KEY,
    gen_id     VARCHAR(40),          -- 保留 GenerateID 用于"按用户批量查看"
    revoked_at DATETIME DEFAULT (datetime('now')),
    until      DATETIME DEFAULT (datetime('now', '+365 days'))  -- 默认 1 年后自动过期
)
```

**过期语义**：
- `until` 不是"会话恢复时间"，而是"黑名单条目清理时间"
- 过期后条目被 `autovacuum` 删除，该 session_id 从黑名单消失
- 但此时旧 Cookie 早已过期（MaxAge = cookie_timeout，默认 1 周），不可能再用
- 所以过期清理是**安全的**：被踢的会话在 Cookie 过期后，黑名单条目已无存在必要

**`until` 应该设多长？**

| 策略 | until 值 | 理由 |
|---|---|---|
| 等于 cookie_timeout | `now + cookie_timeout` | 被踢的 Cookie 过期后黑名单条目即无意义，最省空间 |
| 固定 7 天 | `now + 7 days` | 兼顾安全与存储，覆盖绝大多数 cookie_timeout 配置 |
| 固定 1 年 | `now + 365 days` | 保守，审计追踪期长，但占用更多存储 |

**推荐**：`until = now + max(cookie_timeout, 7天)`。代码实现：

```go
func RevokeSession(sessionID, genID string) error {
    timeout := Config.Get("general.cookie_timeout").Int()  // 分钟
    untilHours := timeout / 60
    if untilHours < 168 { untilHours = 168 }  // 至少 7 天
    _, err := DB.Exec(
        "INSERT INTO RevokedSession(session_id, gen_id, until) VALUES(?, ?, datetime('now', '+' || ? || ' hours'))",
        sessionID, genID, untilHours,
    )
    return err
}
```

#### 11.4.4 用户主动注销时是否写入黑名单？

**不需要。** 理由：
- 主动注销 = 用户在浏览器中点"退出" → `SessionLogout` 清除所有分片 Cookie
- Cookie 被清除后，即使 session_id 不在黑名单，旧 Cookie 也无法再使用
- 写入黑名单是多余的，且会与"被强制踢出"混淆

**例外**：如果担心 Cookie 在网络层被抓包（如中间人），注销后应考虑将 session_id 写入黑名单，使被窃取的 Cookie 立即失效。但这属于安全加固，不是基本功能。

#### 11.4.5 审计日志中的 session_id 追溯

`SessionAuthenticate` `server/ctrl/session.go:128` 的审计日志：

```go
Log.Stdout("AUDIT action[login] backend[%s] user[%s] target[%s]", session["type"], username(session), ip(req))
```

需要扩展为：

```go
Log.Stdout("AUDIT action[login] backend[%s] user[%s] sid[%s] target[%s]", session["type"], username(session), session["session_id"], ip(req))
```

同样，`SessionLogout` `server/ctrl/session.go:179` 和管理员踢出操作也需要记录 session_id。

管理员从审计日志中提取 session_id，构造踢出请求。管理员 UI 可以提供"查看活跃会话 → 点击踢出"的操作。

---

### 11.5 黑名单查询与 AppCache TTL 的互动窗口

#### 11.5.1 完整时序图

```
             时间 →
             
设备 A (session_id=SID_A)           服务端                       AppCache
    │                                │                            │
    ├─ GET /api/files/ls ───────────►│                            │
    │                                ├─ SessionStart:             │
    │                                │  1. _extractAuthorization  │
    │                                │  2. _extractSession        │
    │                                │     → session_id = SID_A   │
    │                                │  3. IsSessionRevoked(SID_A)│
    │                                │     → false ✅             │
    │                                │  4. _extractBackend        │
    │                                │     → AppCache.Get(params) │
    │                                │     → 命中！取出旧 *Sftp   │
    │  ◄── 200 OK ──────────────────┤                            │
    │                                │                            │
    │         管理员踢出 SID_A        │                            │
    │                                │                            │
    │                                ├─ RevokeSession("SID_A")   │
    │                                │  → INSERT INTO RevokedSes  │
    │                                │                            │
    ├─ GET /api/files/cat ─────────►│                            │
    │                                ├─ SessionStart:             │
    │                                │  3. IsSessionRevoked(SID_A)│
    │                                │     → true ❌              │
    │                                │  → RecoverFromBadCookie    │
    │  ◄── 401 ErrSessionRevoked ───┤  → SendErrorResult        │
    │                                │                            │
    │                                │     AppCache 中旧连接:     │
    │                                │     *Sftp{SID_A的params}   │
    │                                │     仍在 TTL 内（SFTP=1分钟）│
    │                                │     但无请求能再取出它 ❌    │
    │                                │                            │
    │   ~1分钟后 TTL 过期             │                            │
    │                                │     OnEvict 触发:          │
    │                                │     wg.Wait() + Close()    │
    │                                │                            │
```

#### 11.5.2 互动窗口分析

**窗口定义**：从管理员踢出到 AppCache 中旧连接被 TTL 清理之间的时间段。

| 后端 | AppCache TTL | 互动窗口时长 | 窗口内旧连接可被利用？ |
|---|---|---|---|
| SFTP | 1 分钟 | ≤ 1 分钟 | **否** |
| S3 | 2 分钟 | ≤ 2 分钟 | **否** |
| Samba | 30 分钟 | ≤ 30 分钟 | **否** |
| FTP | 30 秒 | ≤ 30 秒 | **否** |
| WebDAV | 无缓存 | 0 | 不涉及 |

**为什么"否"？** 因为要从 AppCache 中取出连接对象，必须经过 `SessionStart` 的完整链路：
1. `_extractAuthorization` → 需要 Cookie 中有合法加密 token
2. `_extractSession` → 解密出 session map（含 session_id）
3. `IsSessionRevoked(session_id)` → 检查黑名单 → **被拦截**
4. `_extractBackend` → `model.NewBackend` → Backend.Init → AppCache.Get

步骤 3 拦截后请求不会到达步骤 4，AppCache 中的旧连接永远不会被已撤销的 session_id 取出。

#### 11.5.3 攻击者能否绕过黑名单直接命中 AppCache？

**不能。** AppCache 是进程内私有数据结构（`patrickmn/go-cache`），没有 HTTP 接口。攻击者无法直接调用 `SftpCache.Get()`。唯一的入口是 `model.NewBackend` → `Backend.Init` → `AppCache.Get`，而这条路径被 `SessionStart` 保护。

#### 11.5.4 AppCache 是否需要主动清理被撤销的条目？

**不需要，但可以优化。**

当前行为：被撤销会话对应的 AppCache 条目自然等待 TTL 过期 + OnEvict 清理。

如果希望踢出时立即释放资源（如关闭 SSH 连接），可在 `RevokeSession` 中追加一步：

```go
// 优化（可选）：踢出时主动清理 AppCache 中对应的连接
// 但问题是 RevokeSession 只有 session_id，没有完整 params → 无法计算 AppCache key
```

**困难**：AppCache key = `hashstructure.Hash(完整 params map)`，包含 password。黑名单只存 session_id，没有完整 params → 无法计算 AppCache key → 无法主动删除。

**解决路径**（超出最小方案）：
1. 在 `RevokedSession` 表额外存储加密的 params（占用空间）
2. 在 `RevokedSession` 表存储 AppCache key（需要 Init 时计算并持久化）
3. 不主动清理，依赖 TTL 自然过期（推荐）

#### 11.5.5 极端场景：TTL 内旧连接占用后端资源

Samba 的 TTL 长达 30 分钟。如果用户被踢出，SMB 共享在 30 分钟内不会被卸载。

**影响**：
- 后端 Samba 服务器上该用户的 SMB session 仍然活跃
- 如果后端有并发连接数限制，被踢用户的连接可能占位

**缓解方案**：
1. 缩短 Samba TTL（简单但增加连接重建频率）
2. 在 `RevokeSession` 时，遍历 `SambaCache` 检查每个条目的 session_id 是否匹配（需要 Samba 缓存条目中存储 session_id）
3. 定时任务扫描黑名单，对 AppCache 做反向清理（复杂）

**最小方案选择**：不处理，依赖 TTL 自然过期。30 分钟窗口在大多数场景下可接受。

---

### 11.6 session_id 方案下的完整改动清单

| 序号 | 文件 | 改动内容 | 与 GenerateID 方案的差异 |
|---|---|---|---|
| 1 | `server/model/index.go` | `RevokedSession` 表加 `session_id` 和 `gen_id` 双字段 | 增加 session_id 字段 |
| 2 | `server/model/session.go` | `IsSessionRevoked(sessionID)` / `RevokeSession(sessionID, genID)` / `UnrevokeSession(sessionID)` / `RevokedSessionList()` | 主键从 gen_id 变为 session_id |
| 3 | `server/common/error.go` | 新增 `ErrSessionRevoked` | 相同 |
| 4 | `server/common/crypto.go` | `GenerateID` 的 switch 增加 `case "session_id":` | 新增 |
| 5 | `server/ctrl/session.go` | `SessionAuthenticate` 第 53 行后加 `ctx.Body["session_id"] = RandomString(20)`；`SessionAuthMiddleware` 第 427 行后加 `mappingToUse["session_id"] = RandomString(20)` | 新增 |
| 6 | `server/middleware/session.go` | `SessionStart` 中在 `_extractSession` 之后、`_extractBackend` 之前加 `IsSessionRevoked` 检查 + 清分片 Cookie | 检查 key 从 gen_id 变为 session_id |
| 7 | `server/ctrl/admin.go` | 新增 3 个 handler | 相同 |
| 8 | `server/routes.go` | 注册 3 条路由 | 相同 |
| 9 | 审计日志 | `SessionAuthenticate` 和 `SessionLogout` 的 AUDIT 日志加 `sid[%s]` 字段 | 新增 |

**不改的文件**（与 GenerateID 方案相同）：
- 各后端插件 / AppCache / Share 相关 / SessionTry / CanManageShare

---

### 11.7 上线协同关键时间线

```
T0: 管理员调用 RevokeSession("SID_OLD", "gen_id_alice")
    → RevokedSession 表写入一行
    → 旧设备不知情，Cookie 仍在浏览器中

T0+ε: 旧设备发起下一次 HTTP 请求
    → SessionStart → _extractSession 解出 session_id = "SID_OLD"
    → IsSessionRevoked("SID_OLD") = true
    → RecoverFromBadCookie: Set-Cookie auth=; MaxAge=-1, auth1=; MaxAge=-1, ...
    → 返回 401 ErrSessionRevoked
    → 前端跳转登录页
    → 旧设备被踢出 ✅

T0+ε: 同一用户的新设备（session_id = "SID_NEW"）不受影响
    → IsSessionRevoked("SID_NEW") = false
    → 正常工作 ✅

T0 ~ T0+TTL: AppCache 中旧连接自然过期
    → SFTP: ~1 分钟后 OnEvict → wg.Wait() + Close()
    → Samba: ~30 分钟后
    → 此期间旧连接占用后端资源，但无法被任何请求利用

T0 + cookie_timeout: 旧 Cookie 的 MaxAge 到期
    → 即使浏览器未收到 RecoverFromBadCookie（如离线），Cookie 也自然失效
    → 黑名单条目仍有价值（防止 Cookie 被抓包重放）

T0 + until: 黑名单条目过期
    → autovacuum 清理
    → 此时旧 Cookie 早已过期（cookie_timeout << until）
    → 清理是安全的 ✅
```

---

### 11.8 并发场景核对

| 场景 | 时序 | 结果 |
|---|---|---|
| 踢出与登录并发 | RevokeSession 写入与 SessionAuthenticate 写 Cookie 同时进行 | 新登录获得新 session_id，不受旧 session_id 黑名单影响 ✅ |
| 踢出与文件操作并发 | RevokeSession 写入时，用户正在下载文件 | 当前请求不受影响（请求级快照），下次请求被拒 ✅ |
| 同一用户多设备同时踢出 | 管理员快速连续踢出 SID_A 和 SID_B | 两次 INSERT 不同主键，互不冲突 ✅ |
| 同一 session_id 重复踢出 | 管理员对同一设备踢两次 | SQLite 主键冲突 → INSERT OR IGNORE → 幂等 ✅ |
| 踢出后 AppCache Get 并发 | 请求 A 在黑名单检查后、AppCache.Get 前被踢出 | 黑名单检查已通过（T1 时刻为 false），请求 A 正常完成 ✅ |
| 踢出时 SessionTry 并发 | SessionLogout 的 goroutine 走 SessionTry 取旧连接 | SessionTry 忽略所有错误，不查黑名单，尝试关连接（安全） ✅ |
