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
