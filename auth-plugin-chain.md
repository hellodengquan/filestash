# 登录鉴权插件链梳理

本文档从一次登录请求出发，逐环拆解 Filestash 的鉴权插件协作机制，厘清每一环的职责边界与数据流向。

---

## 一、全景概览

Filestash 的鉴权体系由三层独立职责串联而成：

```
用户请求 ──▶ HTTP 中间件链 ──▶ 身份认证插件 (IAuthentication) ──▶ 后端连接 (IBackend)
                 │                      │                              │
           请求校验 / 安全校验         身份校验 / 凭证验证            存储系统连通
```

- **HTTP 中间件链**：请求进入后的第一道关卡，负责协议层安全、速率限制、会话恢复。
- **身份认证插件 (Identity Provider)**：根据配置选中一个 `IAuthentication` 实现，完成用户身份验证。
- **后端连接 (IBackend)**：用认证结果去初始化存储后端，确认用户在存储系统中的合法性。

> 关键设计：同一时刻只激活一个身份认证插件，由配置 `middleware.identity_provider.type` 决定。

---

## 二、两条登录路径

Filestash 存在两条独立的登录入口，它们的中间件链和后续流程不同：

### 路径 A：前端表单直连登录 (POST /api/session)

```
POST /api/session
  │
  ├─ 1. ApiHeaders          → 设置 Content-Type: application/json
  ├─ 2. SecureHeaders       → 安全响应头 (HSTS, X-Content-Type-Options, X-XSS-Protection)
  ├─ 3. SecureOrigin        → Host 校验 + 请求来源合法性检查
  ├─ 4. RateLimiter         → 令牌桶限流 (10 req/s, burst 1000)
  ├─ 5. BodyParser          → 解析 JSON body → ctx.Body
  ├─ 6. PluginInjector      → 注入插件注册的中间件
  │
  └─ SessionAuthenticate    → 核心处理函数
```

**路由注册** (`server/routes.go:27`)：

```go
middlewares = []Middleware{ApiHeaders, SecureHeaders, SecureOrigin, RateLimiter, BodyParser, PluginInjector}
session.HandleFunc("", NewMiddlewareChain(SessionAuthenticate, middlewares)).Methods("POST")
```

**处理逻辑** (`server/ctrl/session.go:52-135`)：

1. 将 `ctx.Body` 加上 `timestamp` 和 `path` 字段，构成 session map
2. 调用 `model.NewBackend(ctx, session)` 尝试创建后端连接（即验证凭据 + 连通存储）
3. 若后端支持 OAuth，执行 `OAuthToken()` 完成令牌交换
4. 获取用户 home 目录
5. 将 session 序列化 → 用 `SECRET_KEY_DERIVATE_FOR_USER` 加密 → 写入分片 Cookie (`auth`, `auth0`, `auth1`, ...)
6. 返回 `{is_authenticated: true, home, backend, authorization}`

> 这条路径适用于前端页面直接提交表单的场景，用户在 UI 上选择后端类型后直接填写凭据。

### 路径 B：身份认证插件登录 (GET/POST /api/session/auth/)

```
GET/POST /api/session/auth/
  │
  ├─ 1. ApiHeaders          → 设置 Content-Type: application/json
  ├─ 2. SecureHeaders       → 安全响应头
  ├─ 3. PluginInjector      → 注入插件注册的中间件
  │
  └─ SessionAuthMiddleware  → 核心处理函数（编排整个插件认证流程）
```

**路由注册** (`server/routes.go:32`)：

```go
session.HandleFunc("/auth/", NewMiddlewareChain(SessionAuthMiddleware, middlewares)).Methods("GET", "POST")
```

**处理逻辑** (`server/ctrl/session.go:220-487`)，分四个步骤：

| 步骤 | 名称 | 职责 |
|------|------|------|
| Step 0 | 初始化 | 读取配置选中插件，解析 IdP 参数，收集 formData |
| Step 1 | EntryPoint | GET 请求 + `action=redirect` 时，调用插件的 `EntryPoint()` 展示登录页 / 重定向到外部 IdP |
| Step 2 | Callback | POST 请求或 OAuth 回调时，调用插件的 `Callback()` 验证凭据，返回用户属性 map |
| Step 3 | 属性映射 | 将 Callback 返回的用户属性通过 `middleware.attribute_mapping.params` 模板渲染为 session |
| Step 4 | 持久化 | session 加密后写入 Cookie，重定向到前端首页 |

> 这条路径适用于 SSO / LDAP / htpasswd 等需要独立登录页或外部 IdP 跳转的场景。

---

## 三、HTTP 中间件逐环解析

### 3.1 ApiHeaders

- **位置**：`server/middleware/http.go:14-24`
- **职责**：设置 `Content-Type: application/json`、`Cache-Control: no-cache`，透传 `X-Request-ID`
- **何时拦截**：不拦截，纯装饰型

### 3.2 SecureHeaders

- **位置**：`server/middleware/http.go:67-77`
- **职责**：强制 SSL 时设置 `Strict-Transport-Security`；始终设置 `X-Content-Type-Options: nosniff` 和 `X-XSS-Protection: 1; mode=block`
- **何时拦截**：不拦截，纯装饰型

### 3.3 SecureOrigin

- **位置**：`server/middleware/http.go:79-105`
- **职责**：
  1. 校验请求 Host 是否与 `general.host` 配置一致（防 Host 头注入）
  2. 校验请求来源：必须是 `X-Requested-With: XmlHttpRequest`（浏览器 XHR）或有 API 访问权限且无 Cookie 的请求
- **何时拦截**：Host 不匹配或来源非法时返回 403

### 3.4 RateLimiter

- **位置**：`server/middleware/http.go:107-121`
- **职责**：令牌桶限流，每秒 10 个请求，突发容量 1000
- **何时拦截**：超限时返回 429 Too Many Requests
- **局限**：全局限流器（非按用户/IP），高并发下可能误伤

### 3.5 BodyParser

- **位置**：`server/middleware/context.go:11-34`
- **职责**：将请求 body 解析为 `map[string]interface{}` 存入 `ctx.Body`
- **何时拦截**：JSON 解析失败时返回 400

### 3.6 SessionStart

- **位置**：`server/middleware/session.go:57-83`
- **职责**：从请求中恢复已有会话，填充 `ctx` 上下文：
  1. `_extractShare()` → 提取共享链接信息 → `ctx.Share`
  2. `_extractAuthorization()` → 从 Cookie / Authorization 头 / Query 参数 / Basic Auth 提取令牌 → `ctx.Authorization`
  3. `_extractSession()` → 解密 Authorization 令牌 → `ctx.Session`
  4. `_extractBackend()` → 用 session 参数创建后端连接 → `ctx.Backend`
  5. `_extractLanguages()` → 从 Accept-Language 提取语言偏好 → `ctx.Languages`
- **何时拦截**：会话解密失败、后端创建失败时返回 401/403

### 3.7 LoggedInOnly

- **位置**：`server/middleware/session.go:18-26`
- **职责**：守卫 — 确保请求已通过认证（`ctx.Backend != nil && ctx.Session != nil`）
- **何时拦截**：未登录时返回 403

### 3.8 AdminOnly

- **位置**：`server/middleware/session.go:28-55`
- **职责**：校验当前请求是否来自管理员，从 `Authorization` 头或 `admin` Cookie 中解密 `AdminToken`，验证 claim 和过期时间
- **何时拦截**：非管理员或令牌过期时返回 403

### 3.9 PluginInjector

- **位置**：`server/middleware/index.go:72-76`
- **职责**：将所有通过 `Hooks.Register.Middleware()` 注册的插件中间件注入到当前处理链中
- **何时拦截**：取决于具体插件中间件的实现

---

## 四、身份认证插件 (IAuthentication) 详解

### 4.1 接口契约

所有认证插件实现统一接口 (`server/common/types.go:26-30`)：

```go
type IAuthentication interface {
    Setup() Form                                                    // 返回管理后台配置表单
    EntryPoint(idpParams map[string]string, req, res) error         // 展示登录页 / 重定向到外部 IdP
    Callback(formData map[string]string, idpParams, res) (map[string]string, error)  // 验证凭据，返回用户属性
}
```

### 4.2 插件注册机制

每个插件在 `init()` 中通过 `Hooks.Register.AuthenticationMiddleware(id, impl)` 将自己注册到全局 map (`server/common/plugin.go:127-134`)：

```go
var authentication_middleware map[string]IAuthentication = make(map[string]IAuthentication, 0)
```

运行时，`SessionAuthMiddleware` 根据配置 `middleware.identity_provider.type` 的值，从 map 中选中对应插件。**同一时刻只有一个插件被选中**。

### 4.3 各插件职责

#### plg_authenticate_admin

| 维度 | 说明 |
|------|------|
| **注册 ID** | `"admin"` |
| **源码** | `server/plugin/plg_authenticate_admin/index.go` |
| **场景** | 仅允许管理员密码登录 |
| **EntryPoint** | 渲染一个密码输入框的 HTML 表单 |
| **Callback** | 用 bcrypt 校验用户输入密码与 `auth.admin` 配置值；成功返回 `{user: "admin", password: ...}` |
| **暴露属性** | `{{ .user }}` (= "admin"), `{{ .password }}` |

#### plg_authenticate_passthrough

| 维度 | 说明 |
|------|------|
| **注册 ID** | `"passthrough"` |
| **源码** | `server/plugin/plg_authenticate_passthrough/index.go` |
| **场景** | 不做身份验证，直接透传凭据到后端 |
| **EntryPoint** | 根据 `strategy` 配置渲染不同表单：`direct`（自动提交空表单）、`password_only`、`username_and_password` |
| **Callback** | 原样返回 `{user: formData["user"], password: formData["password"]}`，不做任何校验 |
| **暴露属性** | `{{ .user }}`, `{{ .password }}` |
| **适用** | 当后端本身负责认证（如 SFTP/FTP 用密码直连）时使用 |

#### plg_authenticate_local

| 维度 | 说明 |
|------|------|
| **注册 ID** | `"local"` |
| **源码** | `server/plugin/plg_authenticate_local/` |
| **场景** | Filestash 内置用户管理，支持 MFA (TOTP) |
| **EntryPoint** | 渲染邮箱+密码登录表单；若检测到 MFA Cookie 则渲染 TOTP 验证码输入页（含 QR 码绑定流程） |
| **Callback** | 从插件配置的 JSON DB 中读取用户列表，bcrypt 校验密码；若启用了 TOTP 则验证动态验证码；成功返回 `{user, password, bcrypt, role}` |
| **暴露属性** | `{{ .user }}`, `{{ .password }}`, `{{ .bcrypt }}`, `{{ .role }}` |
| **额外功能** | 注册了 `/admin/api/simple-user-management` 管理端点，提供用户 CRUD、密码重置邮件通知 |

#### plg_authenticate_htpasswd

| 维度 | 说明 |
|------|------|
| **注册 ID** | `"htpasswd"` |
| **源码** | `server/plugin/plg_authenticate_htpasswd/index.go` |
| **场景** | 使用 htpasswd / /etc/shadow 格式的用户名:密码哈希列表认证 |
| **EntryPoint** | 渲染用户名+密码登录表单 |
| **Callback** | 逐行解析 IdP 参数 `users` 中的 htpasswd 格式数据，匹配用户名后验证密码哈希（支持 `{SHA}`, `$apr1$`, `$1$`, `$5$`, `$6$`, `$2a$` 等多种格式）；成功返回 `{user, password, n}` |
| **暴露属性** | `{{ .user }}`, `{{ .password }}`, `{{ .n }}` |

#### plg_authenticate_ldap

| 维度 | 说明 |
|------|------|
| **注册 ID** | `"ldap"` |
| **源码** | `server/plugin/plg_authenticate_ldap/index.go` |
| **场景** | 企业 SSO — 委托 LDAP 目录验证身份（企业版功能） |
| **EntryPoint** | 重定向到 Filestash 购买页面（开源版未实现） |
| **Callback** | 返回 `ErrNotImplemented` |
| **配置项** | Hostname, Port, Bind DN, Bind DN Password, Base DN, Search Filter |

#### plg_authenticate_wordpress

| 维度 | 说明 |
|------|------|
| **注册 ID** | `"wordpress"` |
| **源码** | `server/plugin/plg_authenticate_wordpress/index.go` |
| **场景** | 使用 WordPress 站点的用户体系认证 |
| **EntryPoint** | 渲染用户名+密码登录表单 |
| **Callback** | 调用 WordPress XML-RPC API (`wp.getProfile`)，用用户凭据尝试获取 Profile；成功则返回 `{user, password, email, id, role}` |
| **暴露属性** | `{{ .user }}`, `{{ .email }}`, `{{ .roles }}`, `{{ .id }}`, `{{ .password }}` |

---

## 五、登录请求完整流转（路径 B 详解）

以一个典型的 SSO 登录为例（假设配置了 `middleware.identity_provider.type = "local"`）：

```
┌─────────────────────────────────────────────────────────────────────┐
│ 1. 用户访问 /api/session/auth/?action=redirect&label=S3&state=xxx  │
│    中间件链: ApiHeaders → SecureHeaders → PluginInjector             │
│    ↓                                                                │
│    SessionAuthMiddleware: Step0 选中 "local" 插件                    │
│                          Step1 调用 plugin.EntryPoint()              │
│    ↓                                                                │
│    返回 HTML 登录页 (邮箱 + 密码输入框)                              │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 2. 用户提交表单 POST /api/session/auth/                              │
│    formData = {user: "alice@test.com", password: "s3cret"}          │
│    ↓                                                                │
│    SessionAuthMiddleware: Step0 选中 "local" 插件                    │
│                          Step2 调用 plugin.Callback(formData, ...)  │
│    ↓                                                                │
│    插件验证: bcrypt 校验 → MFA 校验 → 返回 {user, password, role}    │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 3. 属性映射                                                          │
│    templateBind = Callback 返回值 + state 参数 + label 参数          │
│    ↓                                                                │
│    读取 middleware.attribute_mapping.params 配置                     │
│    例如: {"S3": {"type":"s3","access_key_id":"{{ .user }}",         │
│           "secret_access_key":"{{ .password }}"}}                   │
│    ↓                                                                │
│    模板渲染 → session = {type:"s3", access_key_id:"alice@test.com", │
│                         secret_access_key:"s3cret", timestamp:"..."}│
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 4. 后端连通性验证                                                     │
│    model.NewBackend(ctx, session) → 初始化 S3 Backend               │
│    ↓                                                                │
│    失败 → 重定向到 /?error=...                                       │
│    成功 → 继续                                                       │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 5. 会话持久化                                                         │
│    session → JSON → EncryptString(SECRET_KEY_DERIVATE_FOR_USER)     │
│    ↓                                                                │
│    写入 Cookie: auth=<encrypted>                                     │
│    清除 SSO 临时 Cookie (ssoref)                                     │
│    ↓                                                                │
│    重定向到前端首页 (302)                                             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 六、会话恢复流程（已登录用户的后续请求）

登录成功后，后续每个 API 请求走 `SessionStart` 中间件恢复会话：

```
请求 GET /api/files/ls
  │
  ├─ SessionStart
  │   ├─ _extractAuthorization()
  │   │   策略1: 拼接分片 Cookie (auth + auth0 + auth1 + ...)
  │   │   策略2: Authorization: Bearer <token>
  │   │   策略3: ?authorization=<token> 查询参数
  │   │   策略4: Basic Auth (用户名="authorization")
  │   │   → ctx.Authorization
  │   │
  │   ├─ _extractSession()
  │   │   DecryptString(SECRET_KEY_DERIVATE_FOR_USER, authorization)
  │   │   → JSON 反序列化 → ctx.Session
  │   │   → 校验 timestamp（超过365天过期）
  │   │
  │   └─ _extractBackend()
  │       model.NewBackend(ctx, ctx.Session) → ctx.Backend
  │
  ├─ LoggedInOnly → 校验 ctx.Backend && ctx.Session
  │
  └─ 业务处理函数
```

---

## 七、授权插件 (IAuthorisation) — 请求级权限控制

认证（Authentication）确认"你是谁"，授权（Authorisation）决定"你能做什么"。

### 接口定义 (`server/common/types.go:32-41`)

```go
type IAuthorisation interface {
    Ls(ctx *App, path string) error
    Cat(ctx *App, path string) error
    Stat(ctx *App, path string) error
    Mkdir(ctx *App, path string) error
    Rm(ctx *App, path string) error
    Mv(ctx *App, from string, to string) error
    Save(ctx *App, path string) error
    Touch(ctx *App, path string) error
}
```

### 注册与调用

- 注册：`Hooks.Register.AuthorisationMiddleware(impl)` — 按注册顺序追加到 slice
- 调用位置：各文件操作 controller 中，在执行操作前逐一调用已注册的授权插件
- 内置示例：`plg_authorisation_example` — 仅日志输出，对写操作返回 `ErrNotAllowed`

### 内置权限模型

除了授权插件，还有两层权限控制：

1. **Share 权限** (`server/model/permissions.go`)：通过 `ctx.Share` 中的 `CanRead/CanWrite/CanUpload/CanShare` 字段控制共享链接的权限
2. **Admin 权限** (`server/middleware/session.go:AdminOnly`)：通过 `AdminToken` 验证管理员身份

---

## 八、密钥体系

| 密钥常量 | 用途 | 派生方式 |
|----------|------|----------|
| `SECRET_KEY` | 根密钥 | 启动时生成或从环境变量读取 |
| `SECRET_KEY_DERIVATE_FOR_USER` | 用户会话加密 | `Hash("USER_" + SECRET_KEY)` |
| `SECRET_KEY_DERIVATE_FOR_ADMIN` | 管理员令牌加密 | `Hash("ADMIN_" + SECRET_KEY)` |
| `SECRET_KEY_DERIVATE_FOR_PROOF` | 共享链接证明 | `Hash("PROOF_" + SECRET_KEY)` |
| `SECRET_KEY_DERIVATE_FOR_HASH` | 用户 ID 哈希 | `Hash("HASH_" + SECRET_KEY)` |
| `SECRET_KEY_DERIVATE_FOR_SIGNATURE` | State 签名 | `Hash("SGN_" + SECRET_KEY)` |

> 密钥派生逻辑位于 `server/common/constants.go:72-78`。

---

## 九、安全防护层

| 层 | 实现 | 说明 |
|----|------|------|
| 速率限制 | `RateLimiter` | 令牌桶 10/s burst 1000（全局共享） |
| Host 校验 | `SecureOrigin` | 防止 Host 头注入 |
| CSRF 防护 | Cookie `SameSite: Strict` / `HttpOnly: true` | 跨站请求无法携带 Cookie |
| 管理员限速 | `AdminSessionAuthenticate` 故意 sleep 1.5s | 暴力破解防护 |
| Cookie 分片 | 超过 3800 字节自动分片 | 避免浏览器 Cookie 大小限制 |
| 会话过期 | `timestamp` 字段 + 365 天有效期 | `server/middleware/session.go:310` |
| 扫描器陷阱 | `plg_security_scanner` | 对常见扫描路径返回 gzip 炸弹/XML 炸弹/重定向到攻击者自身 |
| State 签名 | `features.protection.signature` 配置 | 验证 SSO 回调中 state 参数的完整性 |

---

## 十、关键代码索引

| 文件 | 行号 | 职责 |
|------|------|------|
| `server/routes.go` | 19-33 | 路由与中间件链定义 |
| `server/middleware/index.go` | 21-37 | 中间件链组装 (`NewMiddlewareChain`) |
| `server/middleware/index.go` | 72-76 | 插件中间件注入 (`PluginInjector`) |
| `server/middleware/session.go` | 57-83 | 会话恢复 (`SessionStart`) |
| `server/middleware/session.go` | 160-188 | Authorization 令牌提取 (4 种策略) |
| `server/middleware/session.go` | 262-315 | 会话解密与过期校验 |
| `server/middleware/http.go` | 14-121 | HTTP 层中间件 (Headers, Secure, RateLimit) |
| `server/middleware/context.go` | 11-34 | Body 解析 |
| `server/ctrl/session.go` | 52-135 | 路径 A: 直连登录 (`SessionAuthenticate`) |
| `server/ctrl/session.go` | 220-487 | 路径 B: 插件登录 (`SessionAuthMiddleware`) |
| `server/common/plugin.go` | 127-134 | IAuthentication 插件注册 |
| `server/common/plugin.go` | 141-149 | IAuthorisation 插件注册 |
| `server/common/types.go` | 26-30 | IAuthentication 接口定义 |
| `server/common/types.go` | 32-41 | IAuthorisation 接口定义 |
| `server/common/constants.go` | 72-78 | 密钥派生 |
| `server/plugin/index.go` | 1-49 | 插件导入汇总 |

---

## 十一、鉴权失败的错误处理与日志审计

### 11.1 错误类型体系

所有鉴权相关的错误都实现了 `AppError` 接口（携带 message + HTTP 状态码），定义于 `server/common/error.go:15-31`：

| 常量 | 状态码 | 含义 | 触发场景 |
|------|--------|------|----------|
| `ErrNotAuthorized` | 401 | 未授权 | Authorization 解密失败、Cookie 篡改 |
| `ErrAuthenticationFailed` | 400 | 凭据校验失败 | 插件 Callback 返回失败 |
| `ErrPermissionDenied` | 403 | 权限不足 | LoggedInOnly / AdminOnly 拦截 |
| `ErrNotAllowed` | 403 | 操作被禁止 | SecureOrigin / 后端连接白名单不匹配 |
| `ErrInvalidPassword` | 403 | 密码错误 | 管理员登录、共享密码校验 |
| `ErrNotValid` | 405 | 参数非法 | Body 解析失败、IdP 配置错误 |
| `ErrNotReachable` | 502 | 后端不可达 | LDAP/Wordpress/SFTP 等远端超时 |

`SendErrorResult()` (`server/common/response.go:85-102`) 将错误包装为 `{status:"error", message:"..."}` JSON 响应，并根据 `err.Status()` 自动设置 HTTP 状态码。

### 11.2 各失败节点的日志输出路径

登录过程中每个关键失败点都有明确的日志埋点：

#### 路径 A（直连登录）失败埋点

```
SessionAuthenticate (/server/ctrl/session.go:52)
  ├─ model.NewBackend 失败
  │   ├─ Log.Debug("[auth] action=authenticate::newBackend err=%s")
  │   └─ Log.Stdout("AUDIT action[fail] backend[%s] user[%s] target[%s]") ← 审计日志
  ├─ OAuthToken 失败
  │   └─ Log.Debug("[auth] action=authenticate::oauthtoken err=%s")
  ├─ OAuth 后 NewBackend 再次失败
  │   ├─ Log.Debug("[auth] action=authenticate::oauth::newBackend err=%s")
  │   └─ Log.Stdout("AUDIT action[fail] backend[%s] user[%s] target[%s]") ← 审计日志
  └─ GetHome 失败
      └─ Log.Debug("[auth] action=authenticate::getHome err=%s")
```

#### 路径 B（插件登录）失败埋点

```
SessionAuthMiddleware (/server/ctrl/session.go:220)
  ├─ 插件未找到 / IdP 参数解析错误
  │   └─ 302 重定向到 /?error=...&trace=...
  ├─ plugin.EntryPoint 失败
  │   └─ Log.Error("entrypoint - %s")
  ├─ plugin.Callback 失败 (ErrAuthenticationFailed)
  │   ├─ Log.Warning("failed authentication - %s")
  │   └─ 303 重定向回登录页，通过 flash Cookie 回传错误
  ├─ plugin.Callback 其他失败
  │   └─ Log.Error("session::authMiddleware 'callback error - %s'")
  ├─ 属性映射模板渲染失败
  │   └─ Log.Warning("session::authMiddlware 'attribute mapping error' %s")
  ├─ State 签名校验失败
  │   └─ Log.Debug("callback signature is required, signature=%s")
  ├─ Backend 连接失败
  │   ├─ Log.Debug("session::authMiddleware 'backend connection failed %s'")
  │   └─ Log.Info("[auth] status=failed user=%s backend=%s::%s ip=%s err=%s") ← 审计日志
  └─ Session JSON 序列化失败
      └─ Log.Debug("session::authMiddleware 'session marshal error %+v'")
```

#### 成功审计日志

两条路径的成功场景都有审计输出：

- **路径 A 成功**：`Log.Stdout("AUDIT action[login] backend[%s] user[%s] target[%s]")` (`ctrl/session.go:128`)
- **路径 B 成功**：`Log.Info("[auth] status=success user=%s backend=%s::%s ip=%s")` (`ctrl/session.go:485`)
- **登出**：`Log.Stdout("AUDIT action[logout] ...")` (`ctrl/session.go:179`)

### 11.3 日志存储位置与级别

**日志初始化**：`server/common/log.go:16-24`，写入文件 `state/log/access.log`（路径由 `FILESTASH_PATH` 根目录决定）。

**日志级别控制**：通过 `log.level` 配置项切换，`Log.SetVisibility()` (`log.go:90-118`) 实现：

| 级别 | DEBUG | INFO | WARN | ERROR |
|------|-------|------|------|-------|
| `DEBUG` | ✓ | ✓ | ✓ | ✓ |
| `INFO` (默认) | ✗ | ✓ | ✓ | ✓ |
| `WARNING` | ✗ | ✗ | ✓ | ✓ |
| `ERROR` | ✗ | ✗ | ✗ | ✓ |

**Telemetry 遥测**：`server/middleware/telemetry.go:39-93` 每个请求结束时异步调用 `logger()` 记录：
- 当 `log.enable=true`：输出 `HTTP <status> <method> <duration>ms <path>` 格式的访问日志到 stdout + 文件
- 当 `log.telemetry=true`：将结构化的 LogEntry（含 IP、UA、Backend、Session、Share 等字段）每 10 秒批量 POST 到 `https://downloads.filestash.app/event`

### 11.4 审计插件扩展点

`IAuditPlugin` 接口 (`server/common/types.go:65-71`) 提供审计查询能力：

```go
type IAuditPlugin interface {
    Query(ctx *App, searchParams map[string]string) (AuditQueryResult, error)
}
```

- 注册方式：`Hooks.Register.AuditEngine(impl)` (`server/common/plugin.go:187-192`)
- 管理台入口：`Admin → Audit` 页面通过 `FetchAuditHandler` (`server/ctrl/admin.go:138-158`) 调用
- 默认实现：`SimpleAudit` (`server/model/audit.go:58-75`) 仅提示"需要安装审计插件"，无实际功能
- AuditQueryResult 返回一个搜索表单（按日期/操作类型/路径/后端/用户/IP 等维度）+ 自定义渲染 HTML

### 11.5 插件内失败的用户反馈机制

认证插件通过两种机制把错误信息传递给用户：

1. **flash Cookie**：设置一个 1 秒过期的 `flash` Cookie，登录页 HTML 中读取并渲染红色提示。例如 `admin` 插件 (`plg_authenticate_admin/index.go:75-80`)、`local` 插件 (`plg_authenticate_local/auth.go:186-192`)、`htpasswd` 插件 (`plg_authenticate_htpasswd/index.go:117-122`) 都使用此模式。
2. **URL Query 参数**：路径 B 中较重的错误通过 302 重定向附带 `?error=<msg>&trace=<detail>` 返回，由前端页面解析展示。

---

## 十二、Session 持久化：Cookie 与服务端协同

### 12.1 Session 的完整结构

登录成功后加密写入 Cookie 的明文内容是一个 JSON 对象，典型字段：

| 字段 | 来源 | 说明 |
|------|------|------|
| `type` | 属性映射 | 后端驱动名 (s3 / sftp / local 等) |
| `hostname` | 属性映射 | 远端主机地址 |
| `port` | 属性映射 | 远端端口 |
| `username` / `user` | 插件 Callback 返回 | 认证用户名 |
| `password` | 插件 Callback 返回 | 明文密码 (⚠️ 加密后存储) |
| `access_key_id` / `secret_access_key` | 属性映射 | S3 类凭据 |
| `path` | 系统强制补齐 | chroot 根路径，始终以 `/` 结尾 |
| `timestamp` | 登录时写入 | RFC3339 格式，用于判断过期 |
| `role` | local 插件 | 用户角色 |
| `bcrypt` | local 插件 | 密码哈希值 |
| `session` | 扩展会话字段 | 当 `general.extended_session=true` 时，将插件 Callback 原始 JSON 整个序列化存入 |

### 12.2 Cookie 写入流程（加密 → 分片 → 写入）

`SessionAuthenticate` (`server/ctrl/session.go:90-124`) 中的完整链路：

```
session map[string]string
  │
  ├─ json.Marshal → 原始 JSON 字节
  │
  ├─ EncryptString(SECRET_KEY_DERIVATE_FOR_USER, string)
  │   ├─ zlib 压缩                     ← 节省 Cookie 体积
  │   ├─ AES-256-GCM 加密              ← NonceGenerator 生成 12 字节 nonce
  │   └─ base64 URL 安全编码            ← 适合 Cookie 值
  │   → obfuscate string
  │
  └─ 分片循环 (每片 3800 字节)
      ├─ 第 0 片 → Cookie Name: "auth"
      ├─ 第 1 片 → Cookie Name: "auth1"
      ├─ 第 2 片 → Cookie Name: "auth2"
      └─ ...
         每条 Cookie 属性:
         ├─ HttpOnly: true
         ├─ SameSite: Strict (若 iframe 模式则为 None+Secure+Partitioned)
         ├─ MaxAge: 60 * general.cookie_timeout (分钟)
         └─ Path: /api/
```

**分片命名规则** (`server/common/utils.go:97-102`)：

```go
func CookieName(idx int) string {
    if idx == 0 { return COOKIE_NAME_AUTH /* "auth" */ }
    return COOKIE_NAME_AUTH + strconv.Itoa(idx) // "auth1", "auth2", ...
}
```

### 12.3 Cookie 读取流程（4 种策略 → 拼接 → 解密）

`_extractAuthorization()` (`server/middleware/session.go:160-188`) 按优先级尝试四种来源：

| 优先级 | 策略 | 说明 |
|--------|------|------|
| 1 | Cookie 分片拼接 | 循环读取 `auth`, `auth0`, `auth1`, ... 直到不存在，Value 首尾相接 |
| 2 | Authorization: Bearer | 截取 `Bearer ` 前缀之后的部分 |
| 3 | Query: `?authorization=` | URL 查询参数 |
| 4 | Basic Auth | 当用户名硬编码为 `"authorization"` 时，密码字段即 token |

拿到 `ctx.Authorization` 字符串后，`_extractSession()` (`session.go:262-315`) 进行：

1. **Base64 URL 解码** → 原始密文
2. **AES-GCM 解密** (key=`SECRET_KEY_DERIVATE_FOR_USER`) → 压缩数据
3. **zlib 解压** → JSON 明文
4. **JSON 反序列化** → `ctx.Session map[string]string`
5. **有效期校验**：`timestamp + 365 天 < 当前时间` → 返回 `ErrNotAuthorized`（过期 Cookie）

### 12.4 Cookie Max-Age 与服务端硬过期的双层控制

存在两层过期机制：

| 层 | 实现 | 位置 | 过期时间 | 作用域 |
|----|------|------|----------|--------|
| 浏览器层 | Cookie `Max-Age` 属性 | `ctrl/session.go:115` | `general.cookie_timeout` 分钟（配置项） | 浏览器自动删除 Cookie |
| 服务端层 | `timestamp` 字段硬校验 | `middleware/session.go:306-313` | 固定 365 天（24 * 365 小时） | 即使 Cookie 被篡改延长 Max-Age 也会被拒绝 |

> 注意：两者是**或**的关系。先到先失效。`cookie_timeout` 控制用户多久需重新登录，而 365 天硬上限兜底防止长期泄露的 Cookie 被利用。

### 12.5 服务端状态存储（SQLite）

除了 Cookie（客户端状态），服务端还在 `state/db/share.sql` SQLite 数据库中维护以下与鉴权相关的表：

#### Share 表（共享链接）

```sql
-- 由 model/index.go:24-25 创建
CREATE TABLE IF NOT EXISTS Share(
    id VARCHAR(64) PRIMARY KEY,
    related_backend VARCHAR(16),
    related_path VARCHAR(512),
    params JSON,                     -- 密码哈希、用户邮箱、过期时间、权限位
    auth VARCHAR(4093) NOT NULL,     -- 用 SECRET_KEY_DERIVATE_FOR_USER 加密的完整 session
    FOREIGN KEY (related_backend, related_path) REFERENCES Location ...
)
```

共享链接的 `auth` 字段其实就是一个持久化的 session，在 `_extractShare()` 中被解密后替代 Cookie 来源。

#### Verification 表（邮箱验证码）

```sql
CREATE TABLE IF NOT EXISTS Verification(
    key VARCHAR(512),                 -- "email::" + 用户邮箱
    code VARCHAR(4),                  -- 4 位随机验证码
    expire DATETIME DEFAULT datetime('now', '+10 minutes')
)
-- 自动清理: model/index.go:41-44, 每 6 小时执行一次 DELETE 过期记录
```

用于共享链接的"邮件证明"流程，`ShareProofVerifier` (`server/model/share.go:166-228`) 创建验证码、发送邮件并校验。

#### Location 表（路径索引）

```sql
CREATE TABLE IF NOT EXISTS Location(
    backend VARCHAR(16),
    path VARCHAR(512),
    PRIMARY KEY(backend, path)        -- 被 Share 表外键引用，用于级联清理
)
```

### 12.6 登出时的资源释放

`SessionLogout` (`server/ctrl/session.go:137-181`) 的完整清理流程：

```
DELETE /api/session
  │
  ├─ [goroutine 异步] 关闭后端连接 (释放 SFTP/SMB 等长连接)
  │   └─ if backend has Close() method → obj.Close()
  │
  ├─ 循环删除分片 Cookie: auth, auth1, auth2, ... (MaxAge=-1)
  ├─ 删除 Cookie: admin (MaxAge=-1)
  ├─ 删除 Cookie: proof (共享链接证明 Cookie, MaxAge=-1)
  │
  └─ 审计日志: AUDIT action[logout] ...
```

登出返回 200 成功响应是**同步**的，而后端连接 Close 是**异步 goroutine** 执行（`ctrl/session.go:138-152`）——原因是某些后端连接建立后需要再次鉴权才能 Close，可能耗时数秒，阻塞将导致登出体验差。

---

## 十三、多因素认证 (MFA/OTP) 的扩展点

### 13.1 现有实现：local 插件的 TOTP

`plg_authenticate_local` 是唯一内置 MFA 的认证插件。完整流程 (`server/plugin/plg_authenticate_local/auth.go`)：

```
用户访问 /api/session/auth/?action=redirect
  │
  ├─ EntryPoint()
  │   ├─ 检查 Cookie: mfa 是否已设？
  │   │   ├─ 有 → 渲染 TOTP 输入页
  │   │   │   ├─ 首次绑定用户：生成 TOTP Key → QR 码 PNG (200x200)
  │   │   │   └─ 老用户：仅显示验证码输入框
  │   │   └─ 无 → 渲染邮箱+密码登录页
  │   └─ 页面包含 <input type="hidden" name="session" value="加密的用户凭据">
  │
  ▼  用户提交密码 (POST)
  │
  ├─ Callback() — 第一阶段：密码校验
  │   ├─ bcrypt 比对密码哈希
  │   ├─ 检查 Disabled 标记
  │   │
  │   ├─ 若 IdP 配置 idpParams["mfa"] == "TOTP"
  │   │   ├─ 用户未绑定 MFA (users[i].MFA == "")
  │   │   │   └─ 暂存 formData["mfa"] 明文密钥 + formData["code"]
  │   │   └─ totp.Validate(requestedUser.Code, users[i].MFA)
  │   │       ├─ 成功 → 继续
  │   │       └─ 失败 → 设置 Cookie: mfa=<加密的User对象>
  │   │                 返回 ErrAuthenticationFailed
  │   │                 (EntryPoint 将从 mfa Cookie 还原用户状态并再次展示验证码)
  │   │
  │   └─ 首次绑定成功则 saveUsers() 持久化 MFA 密钥
  │
  └─ 返回 session: {user, password, bcrypt, role}
```

**关键设计 —— 两阶段 MFA 的状态传递**：
- 密码校验通过但 MFA 未通过时，不直接丢弃状态，而是把 `{Email, Password}` 加密后写入 `mfa` Cookie（1 秒有效期）
- 下次 EntryPoint 渲染时读取 `mfa` Cookie，用 `withMFA()` 解密还原出用户名，用户只需输入动态码，无需重输密码
- `User.EncryptedString()` (`auth.go:257-267`) 负责序列化 → 用 `SECRET_KEY_DERIVATE_FOR_USER` 加密

### 13.2 MFA 配置在鉴权链中的位置

MFA 开关不在全局配置中，而是作为 **IdP 参数** 存在：

```json
// middleware.identity_provider.params 配置
{
  "type": "local",
  "mfa": "TOTP",           // 空串 = 禁用；"TOTP" = 启用
  "notification_subject": "...",
  "notification_body": "...",
  "db": "JSON序列化的用户列表，含 MFA 字段"
}
```

用户的 MFA 密钥存储在 `local` 插件的内部数据结构中，每个 `User` 有 `MFA string` 字段（存储 TOTP Secret 的明文），被序列化后加密存入 `middleware.identity_provider.params.db`。

### 13.3 通用 MFA 扩展方案

其他认证插件要接入 MFA，可参考 local 插件的**三要素模式**：

| 要素 | 实现位置 | 作用 |
|------|----------|------|
| **配置开关** | `Setup() Form` 返回的配置表单中加入 select/checkbox | 管理员在后台选择 MFA 策略 |
| **状态暂存** | `Callback` 中校验第一因素失败时 → 写 Cookie 加密暂存 | 用户不用重输密码 |
| **多轮验证** | `EntryPoint` 根据 Cookie 判断当前处于哪个阶段 → 渲染对应页面 | 支持 2 轮以上的多因素流程 |
| **密钥持久化** | 利用插件自身配置结构或外部存储 | 存放用户级 MFA Secret |

通用的两因素实现模板：

```go
// EntryPoint 分支判断
func (this MyAuth) EntryPoint(idpParams, req, res) error {
    // 阶段 2: 从暂存 Cookie 恢复上下文，展示第二因素输入
    if stage2Cookie, err := req.Cookie("mfa_stage2"); err == nil {
        ctx := decryptStage2(stage2Cookie.Value)
        renderMFAPage(res, ctx)
        return nil
    }
    // 阶段 1: 展示用户名密码
    renderLoginPage(res)
    return nil
}

// Callback 分支判断
func (this MyAuth) Callback(formData, idpParams, res) (map[string]string, error) {
    // 第一因素验证
    if !verifyPrimary(formData) {
        return nil, ErrAuthenticationFailed
    }
    // 若需 MFA 且尚未通过第二阶段
    if idpParams["mfa"] != "" && !verifyMFA(formData) {
        setEncryptedCookie(res, "mfa_stage2", formData["user"]) // 暂存
        return nil, ErrAuthenticationFailed                    // 触发重新 EntryPoint
    }
    return userAttrs, nil
}
```

### 13.4 共享链接的 MFA 替代方案：Email 验证码

虽然共享链接本身不走 `IAuthentication` 插件链，但其证明机制 (`ShareProof`) 实现了另一种 MFA —— 邮箱验证码流程 (`server/model/share.go:150-260`)：

```
用户访问共享链接，要求提供邮箱证明
  │
  ├─ ShareProofVerifier(proof.Key="email")
  │   ├─ 校验邮箱在白名单内
  │   ├─ 生成 4 位随机验证码 → 写入 Verification 表 (10 分钟过期)
  │   ├─ 通过邮件配置 SMTP 发送 HTML 邮件
  │   └─ 返回 proof.Key="code" 提示前端展示输入框
  │
  ▼  用户提交 4 位验证码
  │
  ├─ ShareProofVerifier(proof.Key="code")
  │   ├─ 查询 Verification 表 WHERE code=? AND expire > now
  │   ├─ 匹配成功 → DELETE 该验证码（一次性使用）
  │   └─ 还原为 proof.Key="email", proof.Value=用户邮箱
  │
  └─ 将已验证的 Proof 加密写入 proof Cookie
```

邮件 SMTP 配置项：`email.server / email.port / email.username / email.password / email.from`。

### 13.5 State 签名与 MFA/SSO 的防篡改

OAuth/OIDC 等外部 IdP 回调的 `state` 参数（包含 label、next 跳转地址、签名属性）在 `SessionAuthMiddleware` 中进行了防篡改校验：

```go
// server/ctrl/session.go:368-391
for k, v := range stateStruct {
    if k == "signature" {
        signature = v
    }
    if slices.Contains(fields, k) { // features.protection.signature 配置列出的字段
        attributes += fmt.Sprintf("%s[%s] ", k, v)
    }
}
// attributes 字符串用 SECRET_KEY_DERIVATE_FOR_SIGNATURE 加密后必须等于 signature
v, err := DecryptString(SECRET_KEY_DERIVATE_FOR_SIGNATURE, signature)
if err != nil || attributes != v {
    // 302 重定向回首页，提示 Invalid Signature
}
```

该机制确保：即使用户拦截了回调请求并修改 state 中的敏感字段（如 `next` 跳转地址或 `nav` 路径），签名校验也会拒绝请求。

---

## 十四、补充代码索引

| 文件 | 行号 | 职责 |
|------|------|------|
| `server/common/error.go` | 8-94 | 错误体系 (AppError) 与预定义错误常量 |
| `server/common/response.go` | 85-102 | 错误响应序列化 `SendErrorResult` |
| `server/common/log.go` | 11-121 | 日志实现 (文件写入 + 级别控制) |
| `server/middleware/telemetry.go` | 13-133 | 访问日志 + 遥测数据收集与上报 |
| `server/ctrl/admin.go` | 138-158 | 审计插件查询入口 |
| `server/model/audit.go` | 1-75 | 默认审计引擎 (空实现) |
| `server/common/crypto.go` | 27-53 | `EncryptString` / `DecryptString` (zlib + AES-GCM + base64) |
| `server/common/crypto.go` | 128-164 | AES-256-GCM 加解密底层实现 |
| `server/common/crypto.go` | 240-266 | `NonceGenerator` — 计数器型 nonce |
| `server/common/utils.go` | 97-102 | Cookie 分片命名函数 |
| `server/common/recovery.go` | 7-21 | 坏 Cookie 恢复机制 (RecoverFromBadCookie) |
| `server/model/files.go` | 9-51 | `NewBackend` — 白名单校验 + 驱动初始化 |
| `server/model/index.go` | 12-44 | SQLite 表结构 (Share / Verification / Location) |
| `server/model/share.go` | 150-260 | 共享链接 Proof 验证（邮箱验证码） |
| `server/model/share.go` | 291-354 | Proof Cookie 读取与对比 |
| `server/plugin/plg_authenticate_local/auth.go` | 83-238 | local 插件 MFA 完整流程 (EntryPoint + Callback) |
| `server/plugin/plg_authenticate_local/auth.go` | 240-267 | `withMFA` / `EncryptedString` — MFA 暂存加解密 |
| `server/ctrl/session.go` | 361-391 | SSO State 签名校验 |

---

## 十五、登录限流与暴力破解防护

### 15.1 多层限流体系

Filestash 的暴力破解防护不是单一措施，而是由多个独立层叠加形成纵深防御：

```
请求进入
  │
  ├─ 层 1: 全局令牌桶限流 (RateLimiter)
  │   └─ 10 req/s, burst 1000, 全局共享
  │
  ├─ 层 2: 管理员登录故意延迟 (AdminSessionAuthenticate)
  │   └─ 每次尝试固定 sleep 1.5s
  │
  ├─ 层 3: 来源校验 (SecureOrigin)
  │   └─ Host 头 + X-Requested-With 校验
  │
  ├─ 层 4: 扫描器陷阱 (plg_security_scanner)
  │   └─ 已知攻击路径返回反制内容
  │
  └─ 层 5: Syncthing Basic Auth 延迟
      └─ 认证失败时 sleep 1s
```

### 15.2 全局令牌桶限流 (RateLimiter)

- **实现**：`server/middleware/http.go:107-121`
- **算法**：`golang.org/x/time/rate` 令牌桶，`rate.NewLimiter(10, 1000)`
- **参数**：每秒 10 个令牌，突发容量 1000
- **作用域**：**全局单一实例** — 所有 IP、所有用户共享同一个 limiter
- **拦截行为**：返回 `429 Too Many Requests`
- **日志**：`Log.Warning("middleware::http::ratelimit too many requests")`

**限流在路由链中的位置**：

| 路由 | 是否有 RateLimiter | 说明 |
|------|-------------------|------|
| `POST /api/session` (直连登录) | ✓ | 路径 A 核心登录入口 |
| `GET/POST /api/session/auth/` (插件登录) | ✗ | 路径 B 仅经过 ApiHeaders + SecureHeaders + PluginInjector |
| `POST /admin/api/session` (管理员登录) | ✓ | 管理员入口有独立限流 |
| 其他 API 路由 | 视路由而定 | `/api/share`、`/api/files` 等需要 SessionStart 的路由不含限流 |

> **注意**：路径 B (`/api/session/auth/`) 的中间件链中**没有 RateLimiter**，这意味着 SSO/插件登录不受全局令牌桶保护。如果 IdP 外部回调量大，理论上可以无限次触发 `plugin.Callback()`。

### 15.3 管理员登录的故意延迟

`AdminSessionAuthenticate` (`server/ctrl/admin.go:47-84`) 在执行任何密码校验之前强制等待 1.5 秒：

```go
func AdminSessionAuthenticate(ctx *App, res http.ResponseWriter, req *http.Request) {
    // Step 1: Deliberatly make the request slower to make hacking attempt harder for the attacker
    time.Sleep(1500 * time.Millisecond)

    // Step 2: Make sure current user has appropriate access
    admin := Config.Get("auth.admin").String()
    // ... bcrypt 校验 ...
}
```

- **作用**：将管理员密码暴力破解的速率限制在约 0.67 次/秒
- **不影响普通用户登录**：仅 `/admin/api/session` POST 走此逻辑
- **配合管理员令牌有效期**：`AdminToken` 有效期仅 24 小时 (`common/token.go:19`)，Cookie MaxAge 仅 1 小时 (`admin.go:76`)，即使窃取也很快失效

### 15.4 来源校验作为辅助防护

`SecureOrigin` (`server/middleware/http.go:79-105`) 拦截不合法的跨域请求：

1. **Host 校验**：`req.Host` 必须与 `general.host` 配置一致
   - 不匹配时：`Log.Error("Request coming from ... was blocked")` + 返回 403
   - `/admin/` 路径下仅 Warning 不拦截（管理台可能通过不同域名访问）
2. **XHR 校验**：请求必须携带 `X-Requested-With: XmlHttpRequest` 头
   - 豁免：API 模式 (`features.api.enable=true`) 且无 Cookie 的请求
   - 不满足时：`Log.Warning("Intrusion detection: %s - %s", IP, URL)` + 返回 403

这一层实际上阻止了：
- 大多数自动化脚本（不带 XHR 头的 curl/wget）
- Host 头注入攻击
- 跨域表单提交（浏览器不发 XHR 头的 `<form>` POST）

### 15.5 Syncthing 认证延迟

`plg_handler_syncthing` 的 `AuthBasic` 函数 (`server/plugin/plg_handler_syncthing/index.go:84-91`) 在 Basic Auth 失败时 sleep 1 秒：

```go
var notAuthorised = func(res http.ResponseWriter, req *http.Request) {
    time.Sleep(1 * time.Second)
    res.Header().Set("WWW-Authenticate", `Basic realm="User protect", charset="UTF-8"`)
    res.WriteHeader(http.StatusUnauthorized)
    // ...
}
```

### 15.6 WebDAV 黑名单（非 IP 黑名单）

`WebdavBlacklist` (`server/ctrl/webdav.go:67-134`) 是**文件名黑名单**，非 IP 黑名单。它在 WebDAV 共享链接链路 (`/s/{share}`) 中过滤 macOS 系统垃圾文件：

- **PUT/MKCOL 拦截**：`._*`、`.DS_Store`、`.localized` → 405 Method Not Allowed
- **PROPFIND 拦截**：上述 + `.ql_disablethumbnails`、`.ql_disablecache`、`.hidden`、`.Spotlight-V100`、`.metadata_never_index`、`Contents` 等 → 403 Forbidden

### 15.7 安全扫描器陷阱 (plg_security_scanner)

`plg_security_scanner` (`server/plugin/plg_security_scanner/index.go`) 注册了大量已知攻击路径的 handler，当扫描器触达这些路径时：

1. **记录攻击**：`Log.Info("Attack attempt %s %s %s", IP, URL, UserAgent)`
2. **随机反制**（概率分布）：
   - 5%：返回空响应，Content-Length 虚报 1000
   - 5%：返回十亿级大小的 XML（Billion Laughs Attack 反制）
   - 15%：返回 gzip 炸弹（10MB 解压为 10GB）
   - 10%：重定向回攻击者自身 IP
   - 5%：返回死循环 JS
   - 其余：XML 炸弹 / 伪装 JSON / geo 协议重定向

**配置开关**：`features.protection.enable`（默认 true）

### 15.8 当前防护的局限与缺失

| 层面 | 现状 | 风险 |
|------|------|------|
| IP 黑名单 | **不存在** | 无法封禁持续攻击的 IP |
| 按 IP 限流 | **不存在** | RateLimiter 全局共享，单 IP 可耗尽配额 |
| 按 用户/账号 限流 | **不存在** | 单个邮箱可无限次尝试登录 |
| 登录失败计数 | **不存在** | 无"第 N 次失败后锁定"机制 |
| CAPTCHA | **不存在** | 无法区分人与机器 |
| 路径 B 限流 | **不存在** | `/api/session/auth/` 链路无 RateLimiter |

> Filestash 当前的暴力破解防护主要依赖**延迟 + 校验**策略，而非**计数 + 封禁**策略。这在低流量场景下够用，但在公网暴露的高价值目标面前存在明显短板。

---

## 十六、CSRF / 跨域请求保护与鉴权链的协同

### 16.1 CSRF 防护机制全景

Filestash 不使用传统 CSRF Token，而是采用三层替代方案：

```
跨域请求进入
  │
  ├─ 层 1: Cookie SameSite=Strict
  │   └─ 浏览器级：跨站请求不携带 Cookie
  │
  ├─ 层 2: SecureOrigin XHR 头校验
  │   └─ 服务端级：无 X-Requested-With 头 → 403
  │
  └─ 层 3: Cookie HttpOnly=true
      └─ 浏览器级：JS 无法读取 Cookie 内容
```

### 16.2 Cookie SameSite 策略的分支逻辑

`applyCookieRules()` (`server/ctrl/session.go:489-502`) 根据是否启用 iframe 嵌入模式，选择不同的 SameSite 策略：

```
features.protection.iframe 配置
  │
  ├─ 为空（默认，未启用 iframe）
  │   └─ SameSite: Strict
  │       HttpOnly: true
  │       → 浏览器禁止跨站请求携带此 Cookie
  │       → 等效于 CSRF Token 的防护效果
  │
  └─ 非空（启用了 iframe 嵌入，值为允许的 origin）
      ├─ Referer 是 https:// 开头？
      │   ├─ 是 → SameSite: None
      │   │       Secure: true
      │   │       Partitioned: true (CHIPS)
      │   │       → 允许跨站携带 Cookie（iframe 场景必须）
      │   │       → 依赖 Partitioned 隔离不同顶级站点
      │   └─ 否 → SameSite: Strict (降级)
      │            + Log.Warning("non secure origin + iframe enabled")
      └─ 同时在响应中设置 bearer header:
          res.Header().Set("bearer", obfuscate)
          → iframe 通过 URL fragment 传递 #bearer=xxx
```

**iframe 模式下的特殊认证传递**：

当 `features.protection.iframe` 启用时，登录成功后除了设置 Cookie，还通过两种方式传递 token：

1. **响应 Header**：`res.Header().Set("bearer", obfuscate)` — 供 JS 读取
2. **重定向 URL Fragment**：`redirectURI += "#bearer=" + obfuscate` — 供 iframe 父页面读取

### 16.3 三类路由的 CORS/CSRF 策略差异

| 路由类别 | CORS 策略 | CSRF 防护 | 典型路由 |
|----------|-----------|-----------|----------|
| **公共只读 API** | `PublicCORS`: `Access-Control-Allow-Origin: *` | 无 Cookie 无需防护 | `/api/config`、`/api/backend`、`/api/plugin`、静态资源 |
| **用户认证 API** | 无 CORS 头 | SameSite=Strict + XHR 校验 | `/api/session`、`/api/session/auth/` |
| **管理员 API** | 无 CORS 头 | SameSite=Strict + AdminOnly 中间件 | `/admin/api/config`、`/admin/api/session` |

### 16.4 PublicCORS 中间件

`PublicCORS` (`server/middleware/http.go:35-47`) 仅用于**不需要 Cookie** 的公共端点：

```go
func PublicCORS(fn HandlerFunc) HandlerFunc {
    header.Set("Access-Control-Allow-Origin", "*")
    header.Set("Access-Control-Allow-Headers", "x-requested-with, x-request-id")
    if req.Method == http.MethodOptions {
        header.Set("Access-Control-Allow-Methods", "GET, OPTIONS")
        res.WriteHeader(http.StatusNoContent)
        return // OPTIONS 预检请求直接返回
    }
    fn(ctx, res, req)
}
```

**使用场景**：`/api/config`（前端获取公共配置）、`/api/plugin`（获取插件列表）、`/report`（用户反馈提交）、静态资源。

这些端点返回的是不涉及用户数据的信息，允许任何来源读取，因此 `Access-Control-Allow-Origin: *` 是安全的。

### 16.5 插件级 CORS — Site 和 MCP

#### Site 插件 (plg_handler_site)

`cors()` 中间件 (`server/plugin/plg_handler_site/middleware.go:12-32`) 提供受控 CORS：

- 配置项：`features.site.cors_allow_origins`
- 值为 `*`：允许所有来源
- 值为逗号分隔域名列表：仅允许匹配的 Origin
- 用于 `/public/{share}/` 路由，允许外部站点通过 JS API 嵌入 Filestash 共享文件

#### MCP 插件 (plg_handler_mcp)

`WithCORS()` (`server/plugin/plg_handler_mcp/utils/cors.go:9-16`) 提供 MCP 协议所需的 CORS：

```go
w.Header().Set("Access-Control-Allow-Origin", "*")
w.Header().Set("Access-Control-Allow-Headers", "mcp-protocol-version, Content-Type, Authorization")
```

MCP 端点（`/sse`、`/messages`）通过 `Authorization` 头传递 token，不依赖 Cookie，因此 `Allow-Origin: *` 不会造成 CSRF 风险。

### 16.6 SecureOrigin 与 CSRF 的协同

`SecureOrigin` 实际上承担了 CSRF 防护的核心职责。它的判断逻辑确保：

1. **浏览器发起的带 Cookie 请求**：必须带 `X-Requested-With: XmlHttpRequest` 头
   - 跨站 `<form>` POST 不带此头 → 被拦截
   - `fetch()` 跨域请求受 SameSite Cookie 限制 → Cookie 不发送 → 后续 SessionStart 失败
2. **API 客户端（无 Cookie）**：`features.api.enable=true` 且请求无 Cookie → 放行
   - API 客户端使用 `Authorization: Bearer` 头认证，不依赖 Cookie
3. **Host 校验**：防止 DNS 重绑定攻击（攻击者控制 DNS 指向自身服务器，但 Host 头不匹配）

### 16.7 X-Frame-Options 与 Clickjacking 防护

`IndexHeaders` (`server/middleware/http.go:49-65`) 设置：

- `features.protection.iframe` 为空时：`X-Frame-Options: DENY` — 禁止任何 iframe 嵌入
- `features.protection.iframe` 非空时：不设置 X-Frame-Options — 允许指定 origin 的 iframe 嵌入
- `Referrer-Policy: same-origin` — 阻止 Referrer 泄漏到外部

---

## 十七、账号锁定与解锁路径

### 17.1 现有锁定机制：local 插件的 Disabled 字段

`plg_authenticate_local` 的 `User` 模型 (`server/plugin/plg_authenticate_local/index.go:24-32`) 包含 `Disabled bool` 字段：

```go
type User struct {
    Email    string `json:"email"`
    Password string `json:"password"`
    Role     string `json:"role,omitempty"`
    Disabled bool   `json:"disabled,omitempty"`
    Code     string `json:"-"`
    MFA      string `json:"mfa,omitempty"`
}
```

**锁定检测** (`auth.go:185-194`)：

```go
if users[i].Disabled == true {
    http.SetCookie(res, &http.Cookie{
        Name:   "flash",
        Value:  "Account is disabled",
        MaxAge: 1,
        Path:   "/",
    })
    Log.Warning("plg_authentication_simple::auth action=authenticate email=%s err=disabled", users[i].Email)
    return nil, ErrAuthenticationFailed
}
```

**关键特征**：
- 锁定检查在密码校验**之后**、MFA 校验**之前**
- 这意味着即使密码正确，Disabled 账号也无法登录
- 日志明确区分了"密码错误"和"账号禁用"（后者有专门的 Warning）
- 用户看到的提示是"Account is disabled"，而非"Invalid password"

### 17.2 锁定/解锁操作路径

账号的锁定与解锁完全由管理员通过 `/admin/api/simple-user-management` 端点操作：

```
管理员操作锁定:
  POST /admin/api/simple-user-management
  ├─ 中间件: AdminOnly (验证管理员 Cookie/Token)
  ├─ 表单参数: email=xxx&password=xxx&role=xxx&disabled=on
  ├─ updateUser() 执行:
  │   ├─ 定位用户 (by email)
  │   ├─ 更新 Disabled = true
  │   └─ saveUsers() → 序列化为 JSON → 写入 middleware.identity_provider.params.db
  └─ 返回 303 重定向

管理员操作解锁:
  POST /admin/api/simple-user-management?email=xxx
  ├─ 表单参数: disabled 不传 (checkbox 未勾选 = Disabled = false)
  └─ 同上流程，Disabled = false

管理员删除用户:
  DELETE /admin/api/simple-user-management?email=xxx
  └─ removeUser() → 从数组中移除 → saveUsers()
```

### 17.3 用户数据存储与持久化

local 插件的所有用户数据（包括 Disabled 状态）存储在**配置文件**而非数据库中：

```
用户数据流:
  ┌──────────────────────────────────────┐
  │ middleware.identity_provider.params   │  ← JSON 配置字符串
  │ {                                    │
  │   "type": "local",                   │
  │   "mfa": "TOTP",                     │
  │   "db": "[                           │
  │     {\"email\":\"a@b.com\",          │
  │      \"password\":\"$2a$...\",        │
  │      \"disabled\":false,             │
  │      \"mfa\":\"JBSWY3DPEHPK3PXP\"},  │
  │     ...                              │
  │   ]"                                 │
  │ }                                    │
  └──────────────┬───────────────────────┘
                 │ savePluginData()
                 ▼
  ┌──────────────────────────────────────┐
  │ state/config/config.json             │  ← 磁盘持久化
  └──────────────────────────────────────┘
```

`savePluginData()` (`server/plugin/plg_authenticate_local/data.go:31-42`) 通过 `Config.Get(...).Set(string(b))` 将修改写回配置系统，配置系统负责持久化到磁盘。

### 17.4 自动锁定：当前不存在

当前代码中**没有**基于失败次数的自动锁定机制。以下是整个登录链中不存在的行为：

| 期望行为 | 是否存在 | 说明 |
|----------|----------|------|
| 连续 N 次失败后自动锁定账号 | ✗ | local 插件仅检查 `Disabled` 静态字段 |
| 连续 N 次失败后临时封禁 IP | ✗ | RateLimiter 不按 IP 区分 |
| 锁定后自动解锁（超时） | ✗ | 只有管理员手动解锁 |
| 锁定后通知用户 | ✗ | 仅管理员可见日志 |
| 登录失败计数器 | ✗ | 无任何计数逻辑 |

### 17.5 运营介入流程

当需要通过运营手段处理账号安全事件时，当前可用的操作路径：

#### 场景 1：用户账号被攻击者尝试暴力破解

```
1. 运营查看日志:
   GET /admin/api/log  (AdminOnly)
   → 检索 access.log 中的 "failed authentication" 和 "AUDIT action[fail]" 条目

2. 当前能做的:
   - 修改 auth.admin 密码（如果是管理员账号被攻击）
   - 在 IdP 侧修改用户密码（local 插件: 管理页面重置密码）
   - 禁用被攻击账号（设置 Disabled=true）

3. 当前做不到的:
   - 封禁攻击者 IP
   - 限制单 IP 登录频率
   - 查看结构化的登录失败统计
```

#### 场景 2：用户忘记密码 / 账号被锁定

```
1. 用户无法自行解锁:
   - 没有自助"忘记密码"功能
   - 没有"联系管理员"入口

2. 运营解锁路径:
   管理员访问 /admin/simple-user-management
   ├─ 查找用户 → 修改密码 (POST)
   ├─ 或: 取消勾选 disabled (POST)
   └─ 可选: 发送邀请邮件 (仅新用户创建时自动触发)

3. 密码重置:
   - 管理员在用户管理页面设置新密码
   - 新密码经 bcrypt 哈希后存储
   - 若邮件配置完整，可通过自定义 notification 模板通知用户
```

#### 场景 3：管理员账号被入侵

```
1. 紧急措施:
   - 修改 auth.admin 配置（修改管理员密码哈希）
   - 需要直接编辑 state/config/config.json 或通过环境变量

2. AdminToken 失效:
   - AdminToken 有效期 24 小时 (token.go:19)
   - Cookie MaxAge 1 小时 (admin.go:76)
   - 修改密码后旧 token 仍有效直到过期（无主动吊销机制）
   - SECRET_KEY 变更会使所有现有 token 失效（密钥派生链断裂）
```

### 17.6 审计日志的运营可用性

管理员可通过两种方式查看安全事件：

1. **原始日志**：`GET /admin/api/log` — 返回 `state/log/access.log` 的纯文本内容
   - 包含所有 `AUDIT action[fail/login/logout]` 条目
   - 无结构化查询能力
2. **审计插件**：`GET /admin/api/audit` — 调用 `IAuditPlugin.Query()`
   - 默认实现 `SimpleAudit` 返回"需要安装审计插件"的提示
   - 企业版可提供按用户/IP/时间/操作的结构化审计查询

---

## 十八、补充代码索引（二）

| 文件 | 行号 | 职责 |
|------|------|------|
| `server/middleware/http.go` | 35-47 | `PublicCORS` — 公共端点 CORS 中间件 |
| `server/middleware/http.go` | 79-105 | `SecureOrigin` — Host 校验 + XHR 来源校验 |
| `server/middleware/http.go` | 107-121 | `RateLimiter` — 全局令牌桶限流 |
| `server/ctrl/admin.go` | 47-84 | `AdminSessionAuthenticate` — 管理员登录 (含 1.5s 延迟) |
| `server/ctrl/session.go` | 489-502 | `applyCookieRules` — Cookie SameSite/Secure/Partitioned 分支 |
| `server/ctrl/session.go` | 504-507 | `applyCookieSameSiteRule` — SSO Cookie SameSite 覆写 |
| `server/ctrl/webdav.go` | 67-134 | `WebdavBlacklist` — WebDAV macOS 垃圾文件过滤 |
| `server/common/token.go` | 11-35 | `AdminToken` 结构与有效期校验 |
| `server/common/constants.go` | 11-18 | Cookie 名称/路径常量 |
| `server/plugin/plg_security_scanner/index.go` | 21-264 | 扫描器陷阱 — 攻击路径注册与反制 |
| `server/plugin/plg_handler_site/middleware.go` | 12-48 | Site 插件 CORS + BasicAuth 中间件 |
| `server/plugin/plg_handler_site/config.go` | 44-56 | Site CORS 配置 (`features.site.cors_allow_origins`) |
| `server/plugin/plg_handler_mcp/utils/cors.go` | 9-16 | MCP 插件 CORS 中间件 |
| `server/plugin/plg_handler_syncthing/index.go` | 84-91 | Syncthing Basic Auth 延迟 |
| `server/plugin/plg_authenticate_local/index.go` | 24-32 | `User` 结构体 (含 Disabled/MFA 字段) |
| `server/plugin/plg_authenticate_local/service.go` | 12-67 | 用户 CRUD (createUser/updateUser/removeUser) |
| `server/plugin/plg_authenticate_local/data.go` | 18-52 | 插件配置读写 + isEnabled 检查 |
| `server/plugin/plg_authenticate_local/notify.go` | 32-71 | 邀请邮件发送 (sendInvitateMail) |

---

## 十九、第三方 SSO OAuth / SAML 回调失败处理

### 19.1 两套独立的 OAuth 链路

Filestash 中存在两条完全独立的 OAuth 流程，它们的回调失败处理路径完全不同：

| 链路 | 路由 | 场景 | 认证层级 |
|------|------|------|----------|
| **OAuth 后端驱动** | `GET /api/session/auth/{service}` | 用户选择 Google Drive / Dropbox 等后端时，使用 OAuth 授权该后端 | 后端级（存储系统授权） |
| **OAuth IdP 插件** | `GET/POST /api/session/auth/` | 配置了 `plg_authentication_openid` / `plg_authentication_saml` 等企业级 IdP | 身份级（用户是谁） |
| **MCP OAuth 服务器** | `GET /mcp/authorize` + `POST /mcp/token` | 作为 OAuth 授权服务器为 MCP 客户端发放令牌 | 服务端级（Filestash 是 IdP） |

### 19.2 OAuth 后端驱动的回调失败处理

**入口路由**：`GET /api/session/auth/{service}` → `SessionOAuthBackend` (`ctrl/session.go:183-218`)

**典型后端**：`gdrive`（Google Drive，标准 OAuth2 authorization code）、`dropbox`（Implicit Grant Flow）

**`SessionOAuthBackend` 的失败点**：

```
SessionOAuthBackend
  │
  ├─ [FAIL 1] NewBackend 失败
  │   场景: 缺少 ClientID / ClientSecret 配置
  │   处理: Log.Debug + SendErrorResult(res, err) → JSON 错误
  │   示例: GDrive "Missing Client Secret: Contact your admin" (502)
  │
  ├─ [FAIL 2] 后端不支持 OAuth
  │   场景: 调用非 OAuth 后端（如 S3/SFTP）的 OAuth 路由
  │   处理: Log.Debug + SendErrorResult(res, ErrNotSupported) → JSON 405
  │
  └─ [FAIL 3] OAuthURL 解析失败
      场景: 后端返回的授权 URL 格式异常
      处理: Log.Debug + SendErrorResult(res, ErrNotValid) → JSON 405
```

**成功时的跳转**：Accept 头包含 `text/html`（浏览器）→ 303 重定向到 IdP；API 调用 → 200 JSON 返回 URL。

**`state` 参数透传**：SessionOAuthBackend 将 `service::{next}` 拼接后写入 `state` 参数，`::` 作为分隔符（dropbox 不支持，另见下述）。

**`SessionAuthenticate` 中的 OAuth 令牌交换（路径 A）**：

```go
// ctrl/session.go:65-81
if obj, ok := backend.(interface{ OAuthToken(*map[string]interface{}) error }); ok {
    if err := obj.OAuthToken(&ctx.Body); err != nil {
        Log.Debug("[auth] action=authenticate::oauthtoken err=%s", ferror(err))
        SendErrorResult(res, NewError("Can't authenticate (OAuth error)", 401))
        return
    }
    // 再次 NewBackend：这次拿到了真正的 AccessToken
    backend, _ = model.NewBackend(ctx, ctx.Body)
    if backend == nil {
        Log.Debug("[auth] action=authenticate::oauth::newBackend err=%s", ...)
        Log.Stdout("AUDIT action[fail] ...") ← 审计日志
        SendErrorResult(res, ErrAuthenticationFailed)
        return
    }
}
```

#### GDrive 的 OAuthToken 失败点 (`plg_backend_gdrive/index.go:97-112`)：

```
OAuthToken
  │
  ├─ code 缺失 (ctx["code"] 不存在) → oauth2.Exchange 失败 → 返回错误
  ├─ code 无效 / 过期 / 被重放 → Google 返回 invalid_grant → 返回错误
  └─ scope 不足 → Google 返回 insufficient_scope → 返回错误
```

#### Dropbox 的特殊实现：

Dropbox 使用 `response_type=token`（**Implicit Grant Flow**），令牌直接通过 URL fragment 返回前端，后端完全不参与回调。因此：
- 不走 `OAuthToken` 流程
- 前端拿到 `access_token` 后自己组装 `POST /api/session` body
- 后端仅在 `Init` 中验证令牌有效性（第一次 API 调用失败才暴露）

### 19.3 IdP 插件 OAuth / SAML 的回调失败处理

**入口**：`GET/POST /api/session/auth/` → `SessionAuthMiddleware` (`ctrl/session.go:220-487`)

> **重要**：开源版中 `plg_authentication_saml` 和 `plg_authentication_openid` 仅在注释中提及 (`plugin.go:122-123`)，无实际实现。以下流程基于通用 `IAuthentication` 接口和 `SessionAuthMiddleware` 编排逻辑，适用于企业版接入这两个插件时的行为。

**SessionAuthMiddleware 的失败全景图（11 个失败节点）**：

```
SessionAuthMiddleware
  │
  ├─ [FAIL P0-1] 插件未找到 (identity_provider.type 为空/不匹配)
  │   → 307 重定向 /?error=Not%20Found&trace=middleware not found
  │
  ├─ [FAIL P0-2] POST Body 解析失败
  │   → 307 重定向 /?error=Not%20Valid&trace=parsing body - <err>
  │
  ├─ [FAIL P0-3] IdP 配置 JSON 反序列化失败
  │   → 307 重定向 /?error=Not%20Valid&trace=unpacking idp - <err>
  │
  ├─ [FAIL P0-4] IdP 参数模板渲染失败
  │   → 307 重定向 /?error=Not%20Valid&trace=idp - <err>
  │
  ├─ [FAIL P1-1] plugin.EntryPoint() 报错
  │   场景: 外部 IdP 不可达 (OAuth 授权端点超时) / SAML Metadata 拉取失败
  │   → Log.Error("entrypoint - %s")
  │   → 返回 200 OK + HTML 错误页 (Page(err.Error()))
  │
  ├─ [FAIL P2-1] plugin.Callback() 返回 ErrAuthenticationFailed
  │   场景: 用户拒绝授权 / 密码错误 / SAML Assertion 过期 / JWT 签名不匹配
  │   → Log.Warning("failed authentication - %s")
  │   → 303 重定向回 ?action=redirect（再次显示登录页，flash Cookie 提示）
  │
  ├─ [FAIL P2-2] plugin.Callback() 返回其他错误 且 未设置 Content-Type=text/html
  │   场景: IdP 返回 5xx / 网络中断 / 证书校验失败
  │   → Log.Error("callback error - %s")
  │   → 303 重定向 /?error=Not%20Allowed&trace=redirect request failed - <err>
  │
  ├─ [FAIL P2-3] plugin.Callback() 返回错误 但已写 response
  │   (插件内部处理了响应，如 SAML 的 POST Binding 表单自动提交)
  │   → 直接 return，不再后续处理
  │
  ├─ [FAIL P2-4] label/state 缺失 (ssoref Cookie 过期，且 URL 中无 label=)
  │   场景: 用户停留在登录页超过 10 分钟后提交
  │   → Log.Warning("callback_error err=missing_label url=%s")
  │   → 不拦截，但后续 attribute_mapping 找不到对应 label 将导致空 session
  │
  ├─ [FAIL P2-5] State 签名校验失败
  │   场景: 攻击者篡改 state 中的 next / nav / label 字段
  │   → Log.Debug("callback signature is required, signature=%s")
  │   → 307 重定向 /?error=Invalid%20Signature&trace=signature is not correct
  │
  ├─ [FAIL P3-1] Attribute Mapping 模板渲染失败
  │   场景: Callback 返回字段与 Go template 变量不匹配
  │   → Log.Warning("attribute mapping error %s")
  │   → 307 重定向 /?error=Not%20Valid&trace=mapping_error - <err>
  │
  ├─ [FAIL P3-2] model.NewBackend 失败 (凭据合法但后端不可达 / 白名单不匹配)
  │   场景: LDAP 验证通过但 SFTP 服务器宕机；属性映射字段缺失
  │   → Log.Debug("backend connection failed %s")
  │   → Log.Info("[auth] status=failed user=%s backend=%s::%s ip=%s err=%s") ← 审计日志
  │   → 307 重定向 /?error=<msg>&trace=backend error - <err>
  │       (若 err 是翻译错误则用 err.Error()，否则用 ErrNotValid)
  │
  ├─ [FAIL P4-1] session JSON Marshal 失败
  │   → Log.Debug("session marshal error %+v")
  │   → SendErrorResult(res, ErrNotValid) → JSON 405
  │
  └─ [FAIL P4-2] EncryptString 加密失败
      → Log.Debug("encryption error - %s")
      → SendErrorResult(res, ErrNotValid) → JSON 405
```

#### 错误分级与用户反馈策略

| 错误级别 | 错误类型 | 日志级别 | HTTP 响应 | 用户可见 |
|----------|----------|----------|-----------|----------|
| 用户误操作 | `ErrAuthenticationFailed` | Warning | 303 重定向回登录页 | 是 |
| 配置错误 | IdP JSON / 模板 / Mapping 错误 | Warning | 307 重定向首页 | 是（error query） |
| 安全事件 | State 签名失败 / 篡改 | Debug | 307 重定向首页 | 是（不提供细节） |
| 系统故障 | EntryPoint error / Callback 其他错误 | Error | 200 HTML 错误页 / 303 重定向 | 是 |
| 安全审计 | 后端连接失败 | Info + Debug | 307 重定向首页 | 是 |
| 内部异常 | JSON Marshal / Encrypt 失败 | Debug | JSON 405 | 否（前端未处理） |

#### ssoref Cookie 的 10 分钟过期机制

```go
// Step1: 在 EntryPoint 时写入，MaxAge 60*10 = 10 分钟
http.SetCookie(res, &http.Cookie{
    Name:   SSOCookieName /* "ssoref" */,
    Value:  label + "::" + state,
    MaxAge: 60 * 10,
    Path:   COOKIE_PATH,
})

// Step2: 在 Callback 阶段读取，提取 label + state
refCookie, _ := req.Cookie(SSOCookieName)
s := strings.SplitN(refCookie.Value, "::", 2)
```

**过期后果**：10 分钟内未完成 OAuth 回调 → ssoref Cookie 过期 → label/state 缺失 → 依赖 URL query 中的 label/state 作为回退；若两者都缺失则触发 `FAIL P2-4`，最终可能导致 attribute_mapping 找不到后端类型。

#### SAML 场景特有失败模式（预期）

基于 SAML 协议惯例和 `IAuthentication` 接口，企业版 `plg_authentication_saml` 可能的失败：

| 失败场景 | Callback 行为 | 对应用户感知 |
|----------|---------------|-------------|
| SAML Response 签名验证失败 | 返回 error + 自行设置 text/html | 插件内部渲染错误页 |
| Assertion 过期（NotOnOrAfter） | 返回 `ErrAuthenticationFailed` | 重定向登录页 + flash |
| Audience 不匹配（Recipient 错误） | 返回 error | 303 重定向首页 error query |
| InResponseTo 不匹配（重放攻击） | 返回 `ErrAuthenticationFailed` | 重定向登录页 + flash |
| IdP 证书过期 / 指纹不匹配 | 返回 error + Log.Error | 200 HTML 错误页 |

### 19.4 MCP OAuth 授权服务器的失败处理

MCP (`plg_handler_mcp`) 作为 OAuth 授权服务器，有独立的失败处理：

| 路由 | 失败点 | 处理 |
|------|--------|------|
| `GET /mcp/authorize` | response_type != code / client_id 空 / redirect_uri 空 | `http.Error` 400 纯文本 |
| `POST /mcp/token` | grant_type != authorization_code | `http.Error` 400 纯文本 |
| `POST /mcp/token` | authorization_code 解密失败 (被篡改/过期) | `http.Error` 400 "Invalid authorization code" |
| `GET /api/mcp` (`CallbackHandler`) | redirect_uri 缺失 | `SendErrorResult` JSON 错误 |
| `GET /api/mcp` | Authorization 加密失败 | `SendErrorResult` JSON 错误 |

---

## 二十、跨节点 Session 共享：存储选择与一致性策略

### 20.1 架构现状：无状态 + 客户端存储

**核心结论**：Filestash 默认是**纯无状态单节点设计**，不存在原生的跨节点 Session 共享机制。所有用户会话完全存储在**客户端 Cookie** 中，服务端不维护任何活跃 Session 列表。

| 组件 | 存储位置 | 跨节点可用 |
|------|----------|-----------|
| **用户认证 Session** | 客户端 Cookie (`auth` / `auth1` / ...) | ✓（任何节点都能解密，只要 SECRET_KEY 相同） |
| **Admin Token** | 客户端 Cookie (`admin`) + Authorization 头 | ✓（同上，加密依赖 SECRET_KEY） |
| **共享链接 Session** | 服务端 SQLite `Share.auth` 字段 | ✗（本地文件，不共享） |
| **邮箱验证码** | 服务端 SQLite `Verification` 表 | ✗（本地文件，不共享） |
| **SECRET_KEY** | `state/config/config.json` | ✓（需人工同步所有节点） |
| **IdP 用户 DB (local 插件)** | `middleware.identity_provider.params.db` (配置文件) | ✗（单节点修改后需配置同步） |
| **配置文件** | `state/config/config.json` | ✗（各节点独立，除非用共享存储） |

### 20.2 Session 数据在多节点场景下的天然一致性

由于 Session 是**客户端持有 + 服务端对称加密**的设计，不需要分布式 Session 存储：

```
            ┌─────────────────────────────────┐
            │   客户端 Cookie (auth, auth1..) │
            └────┬──────────────┬─────────────┘
                 │              │
      SECRET_KEY一致         SECRET_KEY一致
                 ▼              ▼
         ┌────────────┐  ┌────────────┐
         │  Node A    │  │  Node B    │
         │  Decrypt() │  │  Decrypt() │
         │  + 业务    │  │  + 业务    │
         └────┬───────┘  └────┬───────┘
              │               │
              └───────┬───────┘
                      ▼
           同一份明文 Session JSON
```

**解密依赖的全局共享要素**（所有节点必须完全一致）：

```
1. SECRET_KEY（根密钥）
   ├─ SECRET_KEY_DERIVATE_FOR_USER    ← Session 加密/解密
   ├─ SECRET_KEY_DERIVATE_FOR_ADMIN   ← Admin Token 加密/解密
   ├─ SECRET_KEY_DERIVATE_FOR_SIGNATURE ← State 签名校验
   └─ SECRET_KEY_DERIVATE_FOR_HASH    ← 用户 ID 哈希
   派生函数: Hash("USER_" + SECRET_KEY) 等 (common/constants.go:72-78)

2. general.cookie_timeout 配置       ← Cookie MaxAge 一致性

3. general.extended_session 配置      ← 是否附加扩展 session 字段
```

**配置不一致的后果**：

| 不一致项 | 现象 | 影响 |
|----------|------|------|
| SECRET_KEY 不同 | 节点 A 登录的用户请求节点 B → Decrypt 失败 → 401 | 节点间登录态不互通 |
| cookie_timeout 不同 | 节点 A 设 60 分钟，节点 B 设 1440 分钟 → Cookie MaxAge 不固定 | 用户体验异常，偶发掉线 |
| extended_session 不同 | 节点 A 生成的 session 含扩展字段，节点 B 生成的不含 | 插件可能读取不到扩展信息 |

### 20.3 多节点部署的关键约束与实践方案

#### 方案一：无状态水平扩展（推荐，完全客户端 Session）

```
        ┌─────────────┐
        │   Nginx/ALB │   (Session Affinity: None)
        └──────┬──────┘
               │  Round-Robin
     ┌─────────┼─────────┐
     ▼         ▼         ▼
  Node A    Node B    Node C
  (SECRET_KEY 完全一致)
  (config.json 同步)
```

**必须同步的文件/配置**：

| 文件 | 同步方式 | 变更频率 |
|------|----------|----------|
| `SECRET_KEY` (general.secret_key) | 启动时通过环境变量注入 `FILESTASH_SECRET_KEY`，或部署前预写 config.json | 永不（变更=所有用户掉线） |
| `state/config/config.json` | 共享存储挂载 (NFS/EFS) / ConfigMap + 热加载 / GitOps | 低（管理员操作变更） |
| `general.host` / SSL 证书 | 环境变量 / 配置管理工具 | 低 |
| `auth.admin` 密码哈希 | 同上 | 低 |

**仍不支持的场景**：
- ✗ 管理员主动吊销某个用户的 Session（只能等 Cookie 过期）
- ✗ 活跃 Session 列表查询（无法统计在线用户）
- ✗ 共享链接跨节点访问（Share 表存本地 SQLite）
- ✗ 邮箱验证码跨节点校验（Verification 表存本地 SQLite）

#### 方案二：共享 SQLite（简单共享场景）

将 `state/db/share.sql` 挂载到共享存储（NFS/EFS）：

```
        ┌──────────────────────────────────┐
        │        Load Balancer             │
        └────┬────────────┬────────────────┘
             │            │
        ┌────┴───┐   ┌────┴───┐
        │ Node A │   │ Node B │
        └────┬───┘   └────┬───┘
             │            │
             └──────┬─────┘
                    ▼
        ┌─────────────────────┐
        │  NFS: state/db/     │
        │  └ share.sql        │
        └─────────────────────┘
```

**注意**：SQLite 不支持多节点并发写，实际并发下可能出现 `database is locked` 错误。仅适用于共享链接创建/访问低并发场景。

#### 方案三：外部化 Session 存储（需要代码改造）

当前代码结构下，可通过**插件化改造**实现 Redis 集中式 Session 存储：

**改造点 1**：Session 持久化改为写入 Redis 而非 Cookie

```go
// 现有实现: SessionAuthenticate / SessionAuthMiddleware 末尾
obfuscate, _ := EncryptString(SECRET_KEY_DERIVATE_FOR_USER, string(s))
http.SetCookie(res, &http.Cookie{Name: "auth", Value: obfuscate, ...})

// 改造后:
sessionID := GenerateRandomID(32)
redisClient.Set(ctx, "session:"+sessionID, string(s), cookie_timeout)
http.SetCookie(res, &http.Cookie{Name: "sid", Value: sessionID, ...})
```

**改造点 2**：SessionStart 中间件改为 Redis 查询

```go
// 现有实现: _extractSession()
authorization := _extractAuthorization(req)
s, _ := DecryptString(SECRET_KEY_DERIVATE_FOR_USER, authorization)

// 改造后:
sid, _ := req.Cookie("sid")
s, _ := redisClient.Get(ctx, "session:"+sid.Value).Result()
```

**改造点 3**：增加 IStorage 接口 + Redis 插件实现

```go
type ISessionStore interface {
    Set(sessionID string, session []byte, ttl time.Duration) error
    Get(sessionID string) ([]byte, error)
    Delete(sessionID string) error          // 登出 / 吊销
    List() ([]SessionMeta, error)           // 在线列表
}
```

### 20.4 SECRET_KEY 轮换的一致性挑战

**问题**：SECRET_KEY 一旦更换，所有现有 Session Cookie 立即失效（解密失败 → ErrNotAuthorized）。

**影响范围**：
- 用户登录态（全部掉线）
- 共享链接（Share.auth 字段解密失败）
- Admin Token（管理员被登出）
- State 签名（SSO 回调全部校验失败）
- Proof Cookie（邮箱验证码的证明失效）

**无停机轮换策略（需代码支持）**：

当前代码未实现 Key ID 机制，轮换必须停机。可改造方案：

```
1. 所有加密输出前缀增加 <key_id>:<ciphertext>
2. 维护 <SECRET_KEY, KEY_ID> 映射表（如 Redis / 配置）
3. 解密时根据 key_id 选择旧密钥或新密钥
4. 部署双密钥版本 → 新 Cookie 用新密钥加密，旧 Cookie 用旧密钥解密后立即用新密钥重新加密
5. 确认所有用户都已刷新会话后，移除旧密钥
```

### 20.5 服务端数据的一致性：Share / Verification

即使 Session 本身无状态，服务端仍有两类需要跨节点的**共享状态**：

| 数据 | 单节点位置 | 跨节点一致性策略 |
|------|------------|-----------------|
| 共享链接元数据 | SQLite `Share` 表 | 方案 A: NFS 共享 SQLite (低并发)<br>方案 B: 改为 PostgreSQL/MySQL 外部 DB<br>方案 C: 仅使用相同节点 (LB sticky session) |
| 邮箱验证码 | SQLite `Verification` 表 | 同上 + 验证码有效期短 (10min)，不一致窗口较小 |
| IdP 用户列表 (local) | config.json | 配置同步（Zookeeper / Consul / 文件同步） |
| 审计日志 | state/log/access.log | 采集到集中式 ELK / Loki / ClickHouse |

---

## 二十一、密码重置与忘记密码端到端流程

### 21.1 现状概览：无自助式忘记密码

**核心结论**：Filestash（开源版）**不存在用户自助的"忘记密码"流程**。所有密码变更完全依赖管理员后台操作。

| 能力 | 是否存在 | 实现位置 |
|------|----------|----------|
| 用户自助申请重置链接 | ✗ | — |
| 邮箱验证码重置密码 | ✗ | — |
| 安全问题重置 | ✗ | — |
| 管理员后台重置密码 | ✓ | `UserManagementHandler` (`plg_authenticate_local/handler.go:47-72`) |
| 新用户创建时发送邀请邮件（含明文密码） | ✓ | `sendInvitateMail` (`notify.go:32-54`) |
| 管理员强制用户下次登录改密 | ✗ | — |
| 密码定期过期 | ✗ | — |

### 21.2 管理员重置密码的完整链路

#### 触发入口

管理员通过两种方式进入用户管理：

```
方式 1 (Web UI):
  GET /admin/simple-user-management
    → 走 UserManagementHandler GET 分支
    → 渲染用户列表 HTML 页面 (handler.html 模板)
    → 管理员点击某个用户的"编辑"
    → 显示该用户的 Email / Role / Disabled / 新密码 表单

方式 2 (API):
  GET /admin/api/simple-user-management (Accept: application/json)
    → SendSuccessResults(res, users) 返回所有用户的 JSON 数组
```

#### 密码重置操作流程

```
管理员提交表单 POST /admin/api/simple-user-management?email=<user@xxx>
  │
  ├─ 中间件: AdminOnly 校验管理员身份 (middleware/session.go:28-55)
  │   └─ 失败 → 403 Forbidden
  │
  ├─ UserManagementHandler 执行 updateUser()
  │   ├─ 参数提取:
  │   │   ├─ email    = URL query email (已存在用户)
  │   │   ├─ password = req.FormValue("password") (明文新密码)
  │   │   ├─ role     = req.FormValue("role")
  │   │   └─ disabled = req.FormValue("disabled") == "on"
  │   │
  │   ├─ formatPassword() 校验密码格式
  │   │   └─ 8 ≤ len ≤ 72，正则匹配
  │   │
  │   ├─ bcrypt.GenerateFromPassword(password, BCRYPT_DEFAULT_COST)
  │   │   └─ 成本因子 BCRYPT_DEFAULT_COST = 12
  │   │
  │   ├─ saveUsers() 写回配置
  │   │   └─ Config.Get("middleware.identity_provider.params").Set(serializedDB)
  │   │
  │   └─ 成功
  │
  ├─ 不发送通知邮件（只有 createUser 才触发 go sendInvitateMail）
  │   └─ 密码被重置后，用户不会收到任何邮件
  │
  └─ isAPI 判断:
      ├─ Accept: application/json → SendSuccessResult JSON 200
      └─ 否则 → 303 See Other 重定向回用户列表页
```

**关键代码**：
- 密码格式校验：`formatPassword` (`utils.go:20-37`)
- bcrypt 哈希：`formatPassword` 内部 `bcrypt.GenerateFromPassword(...)`
- 持久化：`savePluginData` (`data.go:31-42`)

### 21.3 新用户创建流程的密码下发（唯一自动发邮件的场景）

```
管理员 POST /admin/api/simple-user-management (不带 email query)
  │
  ├─ createUser() 判断: currentUser.Email == ""
  │   ├─ 验证邮箱唯一性 (重复邮箱报错)
  │   ├─ bcrypt 哈希密码
  │   └─ 追加到 users 数组
  │
  ├─ 异步 goroutine: sendInvitateMail(newUser)
  │   │
  │   ├─ isEmailSetup() 检查 SMTP 配置
  │   │   └─ email.from / server / port / username / password 必须全部配置
  │   │       不完整 → 静默失败 (return nil)
  │   │
  │   ├─ 检查邮件模板
  │   │   ├─ notification_subject 非空
  │   │   └─ notification_body 非空
  │   │       任一为空 → 静默失败 (return nil)
  │   │
  │   ├─ 模板渲染 withTemplate()
  │   │   ├─ 可用变量: {{.instance_url}} {{.user}} {{.password}} {{.role}}
  │   │   └─ ⚠️ 注意: {{.password}} 是**明文密码**！
  │   │       管理员在表单中填什么，邮件里就是什么
  │   │
  │   ├─ gomail SMTP 发送
  │   │   └─ TLS / STARTTLS 根据端口自动判断
  │   │
  │   ├─ 成功 → Log.Info("action=sent email=%s")
  │   └─ 失败 → return fmt.Errorf("cannot send mail - reason=%s")
  │            (goroutine 中的错误无人接收，仅能通过日志发现)
  │
  └─ 管理员页面同步返回成功 (不等待邮件发送结果)
```

> **安全注意**：邀请邮件中包含明文密码。如果 SMTP 通道使用 STARTTLS 端口 587 则加密传输；若使用 25 端口可能明文传输。企业使用前需确认邮件基础设施安全。

### 21.4 登录时的 bcrypt 密码验证

```
用户 POST /api/session/auth/?action=redirect 提交表单
  │
  └─ plg_authenticate_local Callback()
      │
      ├─ 遍历 users[i] 按 Email 匹配
      │
      ├─ bcrypt.CompareHashAndPassword(users[i].Password, []byte(formData["password"]))
      │   ├─ 成功 → 继续 (MFA 检查)
      │   └─ 失败 → flash Cookie "Invalid username or password" + ErrAuthenticationFailed
      │
      ├─ Disabled 检查 → 禁用账号 flash "Account is disabled"
      │
      └─ MFA 检查 (TOTP) → 失败则加密暂存 formData["mfa"] Cookie 回退
```

**bcrypt 成本因子**：`BCRYPT_DEFAULT_COST = 12`，**创建用户时固定**，后续修改密码时也使用同一常量。无法按用户/租户差异化成本。

### 21.5 密码验证失败的处理（与忘记密码的联系）

当 bcrypt 失败时，local 插件 auth.go 的处理：

```go
// auth.go:186-194
http.SetCookie(res, &http.Cookie{
    Name:   "flash",
    Value:  url.QueryEscape("Invalid username or password"),
    MaxAge: 1,
    Path:   "/",
})
Log.Warning("plg_authentication_simple::auth action=authenticate email=%s err=invalid", users[i].Email)
return nil, ErrAuthenticationFailed
```

**用户体验**：
- 不区分"用户不存在"和"密码错误"，统一提示 "Invalid username or password"（防用户名枚举）
- 不展示"忘记密码"链接（前端 UI 没有该入口）
- 不提示"请联系管理员"（用户无自助路径）

### 21.6 其他认证插件的密码重置能力对比

| 插件 | 用户自助改密 | 管理员重置 | 实现方式 |
|------|-------------|-----------|----------|
| `local` | ✗ | ✓ | 用户管理后台 /admin/simple-user-management |
| `admin` | ✗ | 需改配置 | 修改 `auth.admin` 配置值 (bcrypt 哈希) |
| `passthrough` | N/A | N/A | 不保存密码，直接透传后端 |
| `htpasswd` | ✗ | ✗ | 只读 htpasswd 格式，无写回能力 |
| `wordpress` | ✗ | ✗ | 委托 WordPress，密码不存本地 |
| `ldap` (企业版) | ✗ | ✗ | 委托 LDAP 服务器 |
| `openid` (企业版) | ✗ | ✗ | 委托 OpenID Provider |
| `saml` (企业版) | ✗ | ✗ | 委托 SAML IdP |

### 21.7 自助式"忘记密码"的可行实现方案（需扩展代码）

基于现有基础设施（邮件配置 + Verification 表 + share 证明机制），可按以下路径扩展：

```
Phase 1: 申请重置
  1. 用户登录页点击"忘记密码" → 输入邮箱
  2. 后端:
     a. 查询 local 插件 users 是否存在该邮箱
     b. 生成 4 位/6 位随机码
     c. 写入 Verification 表 (key="reset::<email>", code=xxx, expire=+30min)
     d. 调用 sendInvitateMail 发送模板邮件 (复用 SMTP 配置)
     e. 响应统一："如邮箱存在，重置邮件已发送"（防枚举）

Phase 2: 验证 + 改密
  1. 用户打开邮件链接，输入验证码 + 新密码
  2. 后端:
     a. SELECT * FROM Verification WHERE key="reset::<email>" AND code=? AND expire > NOW
     b. 匹配成功 → DELETE 该记录 (一次性)
     c. updateUser() → bcrypt 新密码哈希
     d. 清除所有分片 Cookie: auth, auth1, ... (强制所有设备登出)
     e. return 200 → 前端跳转登录页

Phase 3: 安全加固
  1. 频率限制：每邮箱每 30 分钟最多 3 次申请
  2. 验证码有效期：30 分钟
  3. 验证码错误次数上限：5 次后失效
  4. 强制改密后所有设备登出（清除 Session）
     → 通过轮换 SECRET_KEY 无法细粒度实现，需引入 Session ID 机制
  5. 同步审计日志：AUDIT action[password_reset] email=<x> ip=<y>
```

**可复用的现有组件**：
- 邮件发送：`sendInvitateMail` / `gomail` (`notify.go:32-54`)
- 临时验证码：`Verification` 表 + 自动清理任务 (`model/index.go:41-44`)
- 密码格式校验 + bcrypt：`formatPassword` (`utils.go:20-37`)
- 模板引擎：`withTemplate` (`notify.go:57-71`)
- 强制登出：`SessionLogout` 中的 Cookie 分片删除逻辑 (`ctrl/session.go:153-165`)

---

## 二十二、补充代码索引（三）

| 文件 | 行号 | 职责 |
|------|------|------|
| `server/ctrl/session.go` | 183-218 | `SessionOAuthBackend` — OAuth 后端驱动入口 |
| `server/ctrl/session.go` | 65-81 | `OAuthToken` 调用与错误处理（路径 A） |
| `server/ctrl/session.go` | 220-318 | `SessionAuthMiddleware` Step 0-1（初始化 + EntryPoint 错误处理） |
| `server/ctrl/session.go` | 319-455 | `SessionAuthMiddleware` Step 2-3（Callback 错误 + State 校验 + Backend 验证） |
| `server/ctrl/session.go` | 457-487 | `SessionAuthMiddleware` Step 4（序列化 + 加密失败处理） |
| `server/plugin/plg_backend_gdrive/index.go` | 93-112 | `GDrive.OAuthURL` / `OAuthToken` — 标准 Authorization Code |
| `server/plugin/plg_backend_dropbox/index.go` | 67-73 | `Dropbox.OAuthURL` — Implicit Grant Flow (response_type=token) |
| `server/plugin/plg_handler_mcp/handler_auth.go` | 66-107 | MCP 授权服务器 Authorize / Token 端点错误处理 |
| `server/plugin/plg_handler_mcp/handler_auth.go` | 138-155 | MCP `CallbackHandler` — Authorization Code 发放 |
| `server/plugin/plg_authenticate_local/utils.go` | 20-37 | `formatPassword` — 密码格式校验 + bcrypt 哈希 |
| `server/plugin/plg_authenticate_local/handler.go` | 47-73 | `UserManagementHandler` POST 分支（密码重置/创建） |
| `server/plugin/plg_authenticate_local/notify.go` | 13-30 | `isEmailSetup` — SMTP 配置完整性检查 |
| `server/plugin/plg_authenticate_local/notify.go` | 32-55 | `sendInvitateMail` — 邀请邮件发送（含明文密码） |
| `server/plugin/plg_authenticate_local/notify.go` | 57-71 | `withTemplate` — 邮件模板渲染 |
| `server/plugin/plg_authenticate_local/auth.go` | 186-194 | bcrypt 校验失败处理（错误提示模糊化） |
| `server/plugin/plg_authenticate_local/service.go` | 37-55 | `updateUser` / `createUser` — 持久化用户数据 |
| `server/model/index.go` | 33-44 | `Verification` 表定义 + 每 6 小时过期清理任务 |
| `server/common/config.go` | 71 | `secret_key` 配置定义（轮换影响所有 Session） |
| `server/common/constants.go` | 72-78 | SECRET_KEY 派生链（跨节点一致性核心） |

---

## 二十三、长期登录 Token（记住我）与刷新逻辑

### 23.1 现状：单一 Cookie 有效期，无"记住我"开关

**核心结论**：Filestash 不存在传统的"短期会话 Token + 长期 Refresh Token"双令牌架构，也没有"记住我"复选框。所有登录会话共享同一个 Cookie 有效期，由全局配置 `general.cookie_timeout` 统一控制。

| 设计要素 | 现状 | 典型企业级实现对比 |
|----------|------|-------------------|
| **Session Token** | Cookie `auth`，存储完整 Session JSON 加密 | 短期 JWT（15m-2h） |
| **Refresh Token** | 不存在 | 长期随机字符串（7d-30d），独立存储可吊销 |
| **记住我开关** | 不存在（前端登录页无复选框） | 用户可选，勾选则延长 Refresh Token 有效期 |
| **滑动过期** | 不存在（Cookie MaxAge 不刷新） | 每次请求刷新 token 有效期 |
| **强制吊销** | 不存在（只能等过期或改 SECRET_KEY） | Refresh Token 黑名单 / 主动删除 |

### 23.2 cookie_timeout 配置详解

**定义**：`server/common/config.go:84`

```go
FormElement{
    Name:        "cookie_timeout",
    Type:        "number",
    Default:     60 * 24 * 7,  // 10080 分钟 = 7 天
    Description: "Authentication Cookie expiration in minutes. Default: 60 * 24 * 7 = 1 week",
},
```

**设置生效点**（两处完全一致的逻辑）：

1. **路径 A（直连登录）**：`ctrl/session.go:115`
   ```go
   MaxAge: 60 * Config.Get("general.cookie_timeout").Int(),
   ```

2. **路径 B（插件登录）**：`ctrl/session.go:479`
   ```go
   MaxAge: 60 * Config.Get("general.cookie_timeout").Int(),
   ```

**注意**：单位是**分钟**，但代码中还会再乘以 60 转成秒传给 MaxAge。
- 默认值 10080 分钟 × 60 = 604800 秒 = 7 天
- 若管理员设 1 分钟，则实际 MaxAge = 60 秒

**管理员 Cookie 独立有效期**：`ctrl/admin.go:76` 中硬编码为 1 小时（3600 秒），不受 `cookie_timeout` 影响。

### 23.3 服务端硬过期的兜底（365 天上限）

即便 `cookie_timeout` 设得很大（比如 999999 分钟 ≈ 694 天），服务端在 `_extractSession` 中还有一层硬校验：

```go
// middleware/session.go:306-313
t, err := time.Parse(time.RFC3339, session["timestamp"])
if err != nil {
    return session, ErrNotAuthorized
}
if time.Since(t) > time.Hour*24*365 {
    return session, ErrNotAuthorized
}
```

**两层过期的关系**：

```
浏览器层 (Cookie MaxAge)
   ↓ 先到先失效
服务端层 (timestamp + 365 天)
```

因此任何 Session Cookie 的**最大理论有效期**是 365 天，无论 `cookie_timeout` 设多大。

### 23.4 无滑动刷新机制

当前实现中，**Cookie MaxAge 和 timestamp 都不会在后续请求中刷新**：

- `MaxAge` 在登录时写死，后续请求不更新 Cookie
- `timestamp` 在登录时写入 JSON，后续请求不修改 session 内容
- 因此用户无论每天多么活跃，登录 7 天后（默认）必然掉线，需要重新登录

**刷新前后对比**：

| 场景 | 当前行为 | 期望的滑动刷新行为 |
|------|----------|-------------------|
| 用户登录第 1 天 | MaxAge=604800, timestamp=T1 | MaxAge=604800, timestamp=T1 |
| 用户第 6 天访问 | MaxAge 不变 (剩余 1 天) | MaxAge 刷新为 604800 (剩余 7 天) |
| 用户第 7 天访问 | 强制登出 | 刷新后可继续使用 |

### 23.5 SessionGet 接口与会话状态检查

前端通过 `GET /api/session`（`SessionGet`，`ctrl/session.go:28-50`）轮询会话状态：

```go
type Session struct {
    Home          *string `json:"home,omitempty"`
    IsAuth        bool    `json:"is_authenticated"`
    Backend       string  `json:"backendID"`
    Authorization string  `json:"authorization,omitempty"`
}

func SessionGet(ctx *App, res http.ResponseWriter, req *http.Request) {
    r := Session{IsAuth: false}
    if ctx.Backend == nil {
        SendSuccessResult(res, r)
        return
    }
    home, err := model.GetHome(ctx.Backend, ctx.Session["path"])
    if err != nil {
        SendSuccessResult(res, r) // 后端不可达也返回未认证
        return
    }
    r.IsAuth = true
    r.Home = NewString(home)
    r.Backend = backendID(ctx.Session)
    // Chromecast 功能开启时返回完整 Authorization token
    if ctx.Share.Id == "" && Config.Get("features.protection.enable_chromecast").Bool() {
        r.Authorization = ctx.Authorization
    }
    SendSuccessResult(res, r)
}
```

**关键行为**：
- 后端连接失败（如 SFTP 服务端重启）会导致 `IsAuth=false`，前端自动跳转登录页
- 这是**被动的**会话检查，不是主动的 token 刷新
- `backendID` 由 `GenerateID(ctx.Session)` 计算（见下一章）

### 23.6 可实现的刷新机制扩展方案

基于现有基础设施，可按以下方式增加"记住我"和 token 刷新能力：

#### 方案 A：基于 Session 内容的滑动刷新（最小改动）

```go
// 在 SessionGet 或 SessionStart 中增加:
if time.Since(t) > time.Hour*24*6 { // 距离过期不足 1 天
    // 刷新 timestamp
    session["timestamp"] = time.Now().Format(time.RFC3339)
    s, _ := json.Marshal(session)
    obfuscate, _ := EncryptString(SECRET_KEY_DERIVATE_FOR_USER, string(s))
    // 重新分片写入 Cookie
    for index := 0; ; index++ {
        http.SetCookie(res, &http.Cookie{
            Name:   CookieName(index),
            Value:  obfuscate[index*3800 : end],
            MaxAge: 60 * Config.Get("general.cookie_timeout").Int(),
            // ...
        })
    }
}
```

**缺点**：每次刷新都会重写所有 Cookie 分片，带宽开销较大。

#### 方案 B：双令牌架构（标准 Refresh Token）

```
Phase 1: 登录时下发
  ├─ Access Token: Cookie (HttpOnly, 15m)
  └─ Refresh Token: 独立 Cookie (HttpOnly, 7d, 仅 /api/session/refresh 路由可见)

Phase 2: 正常 API 调用
  → 仅携带 Access Token

Phase 3: Access Token 过期 → 前端收到 401
  → 后台自动 POST /api/session/refresh
  → 服务端验证 Refresh Token（查 Redis 白名单）
  → 签发新的 Access Token + 可选轮换 Refresh Token
  → 返回 200，前端重试原请求
```

**需新增**：
- `POST /api/session/refresh` 路由
- Refresh Token 存储（Redis 白名单，支持主动吊销）
- 前端 axios 拦截器处理 401 → 自动刷新 → 重试原请求

#### 方案 C："记住我"复选框支持

在 `SessionAuthenticate` 和 `SessionAuthMiddleware` 中增加分支：

```go
rememberMe := ctx.Body["remember_me"].(bool)
if rememberMe {
    maxAge = 60 * 60 * 24 * 30 // 30 天
} else {
    maxAge = 60 * Config.Get("general.cookie_timeout").Int()
}
// 同时在 session JSON 中写入 remember_me 标志，用于服务端策略判断
```

---

## 二十四、登录设备管理：活跃会话与远程注销

### 24.1 现状：无设备概念，无法远程注销

**核心结论**：Filestash 开源版**不存在设备管理能力**。没有活跃会话列表、没有设备指纹、没有远程注销功能。管理员无法查看谁在线，也无法踢掉某个设备。

| 能力 | 现状 | 说明 |
|------|------|------|
| 活跃会话列表 | ✗ | 无服务端 Session 存储，无法统计 |
| 设备指纹 | 仅 `GenerateID`，非设备维度 | 按 Session 参数哈希，用于插件索引 |
| 远程注销指定设备 | ✗ | 无 Session ID 机制，无法定位特定会话 |
| 全局强制所有用户登出 | Δ | 改 SECRET_KEY 可实现，但代价太大 |
| 多端登录提示 | ✗ | 无设备数量限制和冲突检测 |

### 24.2 GenerateID：Session 身份标识（非设备标识）

`GenerateID` (`common/crypto.go:193-218`) 创建一个可区分不同 Session 的唯一 ID：

```go
func GenerateID(params map[string]string) string {
    orderedKeys := make([]string, len(params))
    for key := range params {
        orderedKeys = append(orderedKeys, key)
    }
    sort.Strings(orderedKeys)
    p := ""
    for _, key := range orderedKeys {
        switch key {
        case "password", "path", "session", "timestamp":
            // 忽略敏感和易变字段
        default:
            if val := params[key]; val != "" {
                p += key + "=>" + params[key] + ", "
            }
        }
    }
    if p == "" { return "na" }
    p += "salt=>" + SECRET_KEY
    return Hash(p, 20) // SHA1 截断前 20 字符
}
```

**计算逻辑**（按字母序拼接非敏感字段 + SECRET_KEY → 哈希）：
- 拼接顺序按 key 字母序，确保 map 遍历不确定时结果一致
- 排除字段：`password`（敏感）、`path`（用户 chroot 不影响身份）、`session`（扩展字段）、`timestamp`（时间戳每次不同）
- 加盐：SECRET_KEY，防止彩虹表反向推导凭据

**使用场景**：
- `plg_widget_recent`：按 `GenerateID` 存储最近访问文件（`backend_id` 维度）
- `plg_widget_favourite`：按 `GenerateID` 存储收藏夹
- `plg_widget_description`：按 `GenerateID` 存储文件描述
- `SessionGet`：返回 `backendID` 字段供前端使用

**关键局限**：
- 同一用户在不同设备用**相同凭据**登录 → **相同** `GenerateID`
  - 因为两者的 `username`、`hostname`、`type` 等字段完全相同
- 同一用户在不同设备用**不同凭据**登录 → **不同** `GenerateID`
  - 但这是凭据的不同，不是设备的不同
- `GenerateID` 不是设备 ID，无法区分"用户 A 的手机"和"用户 A 的笔记本"

### 24.3 backendID 辅助函数

`backendID` (`ctrl/session.go:129-137`) 对 `GenerateID` 做了简单包装：

```go
func backendID(session map[string]string) string {
    id := GenerateID(session)
    if id == "na" {
        if session["type"] != "" {
            return session["type"]
        }
        return ""
    }
    return id
}
```

若 Session 中所有字段都是排除字段（如 passthrough 模式），返回 `type` 作为兜底。

### 24.4 无 Session 列表的根本原因

Session 完全存储在**客户端 Cookie** 中，服务端不维护任何 Session 注册表：

```
[客户端]                [服务端]
  auth Cookie → 解密 → Session JSON
                          ↓
                    业务逻辑处理
                          ↓
                    响应返回
                          ↓
              (不存任何 Session 状态)
```

没有集中式的 Session 存储 → 没有活跃会话列表 → 无法远程注销。

### 24.5 设备管理扩展方案

要实现设备管理，需要引入**服务端 Session 注册表**：

#### Phase 1：引入 Session ID + 设备指纹

```
登录时生成:
  ├─ Session ID: 32 位随机字符串
  ├─ 设备指纹: Hash(User-Agent + IP + 可选客户端生成的 device_id)
  └─ 设备名称: 从 UA 解析 ("Chrome on macOS", "Safari on iOS" 等)
```

在 Session JSON 中新增字段：
```go
session["sid"] = "a1b2c3..."      // Session ID，每个设备唯一
session["device_name"] = "Chrome on macOS"
session["device_fp"] = "d4e5f6..." // 设备指纹
```

#### Phase 2：服务端 Session 注册表（Redis）

```go
// 存储结构: Hash
redis.HSet("sessions:"+userHash, sessionID, `{
    "sid": "...",
    "device_name": "...",
    "device_fp": "...",
    "login_time": "2026-06-14T10:00:00Z",
    "last_seen": "2026-06-14T10:30:00Z",
    "ip": "192.168.1.100",
    "user_agent": "Mozilla/5.0 ...",
    "is_current": false
}`)

// 每个 Session 单独设置 TTL = cookie_timeout
redis.Expire("sessions:"+userHash+":"+sessionID, cookieTimeout)
```

#### Phase 3：活跃会话列表 API

```
GET /admin/api/sessions?user=<email>
  ├─ 中间件: AdminOnly
  ├─ 查询 Redis: HGETALL sessions:<userHash>
  ├─ 遍历比对当前请求的 sid，标记 is_current=true
  └─ 返回 Session 列表 JSON
```

#### Phase 4：远程注销 API

```
DELETE /admin/api/sessions?sid=<session_id>
  ├─ 中间件: AdminOnly
  ├─ 校验 session_id 属于哪个用户
  ├─ redis.HDel("sessions:"+userHash, sessionID)
  ├─ redis.Del("revoke:"+sessionID)
  ├─ 在黑名单中加入 session_id (TTL = cookie_timeout)
  └─ 返回 200 OK

// 在 SessionStart 中间件增加吊销检查:
if redis.Exists("revoke:"+session["sid"]).Val() == 1 {
    return session, ErrNotAuthorized
}
```

#### Phase 5：用户级自助设备管理（可选）

```
GET /api/sessions → 当前用户所有设备列表
DELETE /api/sessions/:sid → 用户自己注销其他设备
```

### 24.6 全局强制登出的两种路径

| 方式 | 操作 | 影响范围 | 代价 |
|------|------|----------|------|
| **改 SECRET_KEY** | 修改 `general.secret_key` 配置 | 所有用户所有设备立刻掉线（解密失败） | 所有共享链接失效、所有需要重新登录 |
| **Redis 清理**（需改造） | `FLUSHDB` 或批量删除 `sessions:*` + 写入全局 `revoke_all` 标记 | 所有用户所有设备掉线 | 共享链接不受影响（不走 Session 注册表） |

---

## 二十五、管理员模拟用户登录（Impersonation）链路

### 25.1 现状：无原生 Impersonation 能力

**核心结论**：Filestash 开源版**不存在管理员模拟用户登录**的功能。没有 `actAs` 或 `sudo` 模式，管理员无法直接切换到用户视角排查问题。

| 能力 | 现状 | 说明 |
|------|------|------|
| 管理员切换到用户视角 | ✗ | 无 API，无 UI |
| Impersonation 审计日志 | ✗ | 无操作追踪 |
| 模拟状态标识（"正在以 XX 身份操作"） | ✗ | 无 UI 提示 |
| 退出模拟返回管理员身份 | ✗ | 无入口 |
| 禁止模拟特定用户 | ✗ | 无权限控制 |

### 25.2 为什么需要 Impersonation

典型业务场景：
1. **故障排查**：用户报告某个后端连接不上 / 文件看不到 → 管理员以用户身份登录复现
2. **配置验证**：管理员修改了 IdP 属性映射 → 以测试用户身份登录验证
3. **内容审核**：检查某个用户的文件操作是否合规
4. **客服支持**：远程协助用户完成操作

### 25.3 基于现有架构的 Impersonation 实现方案

需要新增一条独立的管理员路由，通过 AdminOnly 校验后直接生成目标用户的 Session Cookie。

#### Phase 1：新增 Impersonation API 路由

```go
// routes.go 新增:
admin.HandleFunc("/impersonate", NewMiddlewareChain(AdminImpersonate,
    []Middleware{ApiHeaders, AdminOnly, SecureOrigin, PluginInjector})).Methods("POST")
```

#### Phase 2：`AdminImpersonate` 处理函数（核心逻辑）

```go
func AdminImpersonate(ctx *App, res http.ResponseWriter, req *http.Request) {
    // 1. 校验管理员身份 (已由 AdminOnly 中间件保证)

    // 2. 提取目标用户标识
    targetEmail := ctx.Body["email"].(string)
    targetBackend := ctx.Body["backend"].(string) // 可选，指定模拟哪个后端

    // 3. 从对应认证插件获取目标用户的完整 Session 数据
    var session map[string]string
    var err error

    switch Config.Get("middleware.identity_provider.type").String() {
    case "local":
        // 从 local 插件的 users 数组中查找
        users := loadUsers() // 复用 plg_authenticate_local 的 loadUsers
        for _, u := range users {
            if u.Email == targetEmail {
                session = map[string]string{
                    "username": u.Email,
                    "user":     u.Email,
                    "role":     u.Role,
                    "bcrypt":   u.Password,
                    "type":     targetBackend, // 需要管理员指定后端类型
                    // 注意: 无法获取用户明文密码，passthrough 后端不可行
                }
                break
            }
        }
    case "ldap", "openid", "saml":
        // 企业版 IdP：查询用户属性映射模板，模拟渲染
        // 或者通过 API 从 IdP 获取用户属性
    default:
        SendErrorResult(res, NewError("Impersonation not supported for this IdP", 405))
        return
    }

    if session == nil {
        SendErrorResult(res, NewError("User not found", 404))
        return
    }

    // 4. 补齐 Session 系统字段
    session["timestamp"] = time.Now().Format(time.RFC3339)
    if session["path"] == "" {
        session["path"] = "/"
    }
    // 关键: 增加 impersonation 标记，用于审计和识别
    session["impersonated_by"] = "admin"
    session["impersonated_at"] = time.Now().Format(time.RFC3339)
    session["impersonated_from_ip"] = req.RemoteAddr

    // 5. 加密写入 Cookie（复用 SessionAuthenticate 的分片逻辑）
    s, _ := json.Marshal(session)
    obfuscate, _ := EncryptString(SECRET_KEY_DERIVATE_FOR_USER, string(s))
    // ... 分片写入 Cookie (同 ctrl/session.go:107-117)

    // 6. 审计日志
    Log.Stdout(fmt.Sprintf(
        "AUDIT action[impersonate] admin=%s target=%s ip=%s",
        GetUserFromAdminToken(ctx), targetEmail, req.RemoteAddr,
    ))

    // 7. 返回成功
    SendSuccessResult(res, map[string]interface{}{
        "status": "ok",
        "redirect": "/",
    })
}
```

#### Phase 3：Impersonation 状态的 UI 识别

前端通过 `GET /api/session` 检测 session 中的 `impersonated_by` 字段，在页面顶部显示蓝色提示条：
```
"您正在以 user@example.com 身份操作 | 退出模拟"
```

"退出模拟"按钮调用新 API `POST /admin/impersonate/stop`，清除当前 Cookie 并重新签发管理员 Cookie。

#### Phase 4：审计增强

在所有操作的审计日志中增加 `impersonated_by` 字段：
```
AUDIT action[ls] path=/Docs impersonated_by=admin@example.com user=user@example.com
```

### 25.4 安全约束与边界

Impersonation 是高权限操作，必须有严格的安全约束：

| 约束 | 实现方式 |
|------|----------|
| **仅超级管理员可使用** | AdminOnly 中间件确保只有管理员能调用 |
| **无法获取用户明文密码** | 对于 passthrough 后端（需要明文密码连接 SFTP），模拟时需要额外指定密码或使用匿名访问 |
| **Session 中保留模拟标记** | `impersonated_by` 字段永久保留在该 Session 中，无法伪造清除 |
| **审计不可抵赖** | 所有操作日志记录模拟者身份 |
| **操作隔离** | 模拟期间不允许修改管理员配置、不允许调用 `DELETE /admin/api/session`（修改管理员密码） |
| **禁止模拟其他管理员** | 目标用户 role == "admin" 时拒绝操作 |
| **模拟 Session 短期有效** | Impersonation 的 Session MaxAge 固定 1 小时，不继承全局 cookie_timeout |
| **密码哈希不可复用** | local 插件存储的是 bcrypt 哈希，模拟登录时如果后端需要密码验证（如 passthrough），需要特殊处理（见下） |

### 25.5 不同认证插件的 Impersonation 实现差异

| 插件 | 模拟可行性 | 实现要点 |
|------|-----------|----------|
| `local` | 高 | 从 users 数组读取用户属性，构造 session。但 bcrypt 哈希不能当明文密码用 |
| `admin` | N/A | 管理员自身就是超级用户，不需要模拟 |
| `passthrough` | 低 | 需要明文密码才能连接后端，管理员不知道用户密码 → 只能用于不需要凭证的后端 |
| `htpasswd` | 中 | 只读读取 htpasswd 文件，同 local 只有哈希 |
| `wordpress` | 中 | 通过 XML-RPC 重新获取用户，但需要用户密码 |
| `ldap` (企业版) | 高 | LDAP 管理员绑定后可读取任何用户属性，属性映射模板可正常渲染 |
| `openid` / `saml` (企业版) | 高 | 管理员可使用 Client Credentials Flow 获取目标用户的 token，或通过 IdP 管理员 API 模拟 |

### 25.6 关键难点：密码哈希 vs 明文密码

local 插件的用户密码是 bcrypt 哈希，无法还原为明文。这导致：

- ✅ **本地虚拟后端**：type=local（不需要连接外部服务）→ 可直接模拟
- ✅ **已知凭据后端**：S3 / AzureBlob 等在属性映射中硬编码凭据 → 可模拟
- ❌ **passthrough 后端**：需要用户明文密码连接 SFTP/FTP → 无法模拟（除非管理员额外输入密码）
- ❌ **用户特定 OAuth 后端**：用户自己的 OAuth token 不在 IdP 存储 → 无法模拟

### 25.7 退出模拟（Stop Impersonation）

需要提供 `POST /admin/impersonate/stop` API：

```go
func AdminStopImpersonate(ctx *App, res http.ResponseWriter, req *http.Request) {
    // 1. 清除用户 auth Cookie 分片
    for index := 0; ; index++ {
        if _, err := req.Cookie(CookieName(index)); err != nil {
            break
        }
        http.SetCookie(res, &http.Cookie{
            Name:   CookieName(index),
            Value:  "",
            MaxAge: -1,
            Path:   COOKIE_PATH,
        })
    }
    // 2. 管理员 Session 本身仍然有效（admin Cookie 未被清除）
    // 3. 返回重定向到 /admin
    // 4. 审计日志
    Log.Stdout(fmt.Sprintf("AUDIT action[impersonate_stop] admin=%s ip=%s", ...))
}
```

---

## 二十六、补充代码索引（四）

| 文件 | 行号 | 职责 |
|------|------|------|
| `server/common/config.go` | 84 | `cookie_timeout` 配置定义（默认 7 天，单位分钟） |
| `server/ctrl/session.go` | 21-26 | `Session` 响应结构体（IsAuth / Backend / Authorization） |
| `server/ctrl/session.go` | 28-50 | `SessionGet` — 前端轮询会话状态接口 |
| `server/ctrl/session.go` | 115 | 路径 A Cookie MaxAge 设置（`cookie_timeout` × 60） |
| `server/ctrl/session.go` | 479 | 路径 B Cookie MaxAge 设置（同上） |
| `server/ctrl/session.go` | 129-137 | `backendID` — GenerateID 包装函数 |
| `server/ctrl/admin.go` | 16-45 | `AdminSessionGet` — 管理员登录态检查 |
| `server/ctrl/admin.go` | 76 | 管理员 Cookie MaxAge 硬编码 1 小时（3600 秒） |
| `server/middleware/session.go` | 306-313 | Session timestamp 硬过期校验（365 天上限） |
| `server/common/crypto.go` | 193-218 | `GenerateID` — Session 身份哈希（排除 password/path/session/timestamp） |
| `server/common/crypto.go` | 220-240 | `GenerateMachineID` — 服务器机器标识（/etc/machine-id 等） |
| `server/plugin/plg_widget_recent/index.go` | 22,32,75,83 | `GenerateID` + `getUser` 用于最近文件索引 |
| `server/plugin/plg_widget_favourite/index.go` | 73-83 | `GenerateID` 用于收藏夹索引 |
| `server/plugin/plg_widget_description/handler.go` | 54 | `GenerateID` 用于文件描述索引 |
| `server/plugin/plg_authenticate_local/service.go` | 12-67 | `loadUsers` / `saveUsers` — local 插件用户数据读写（Impersonation 依赖） |
