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
