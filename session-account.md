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
