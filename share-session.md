# Filestash 共享链接全链路分析

## 一、数据模型

### 1.1 Share 结构体 (`server/common/types.go:180-194`)

```go
type Share struct {
    Id           string  `json:"id"`              // 共享链接 ID（URL 路径参数 /s/{id}）
    Backend      string  `json:"-"`               // 所属后端的唯一标识（= GenerateID(创建者Session)）
    Auth         string  `json:"auth,omitempty"`  // 加密后的创建者凭证，用于恢复后端连接
    Path         string  `json:"path"`            // 共享的文件/目录路径
    Password     *string `json:"password,omitempty"` // bcrypt 哈希后的密码
    Users        *string `json:"users,omitempty"`    // 允许的邮箱列表（逗号分隔，支持 * 通配符）
    Expire       *int64  `json:"expire,omitempty"`   // 过期时间（Unix 毫秒时间戳）
    Url          *string `json:"url,omitempty"`      // 重定向 URL
    CanShare     bool    `json:"can_share"`          // 被分享者是否可再分享
    CanManageOwn bool    `json:"can_manage_own"`     // 被分享者是否可管理自己的分享
    CanRead      bool    `json:"can_read"`           // 是否可读
    CanWrite     bool    `json:"can_write"`          // 是否可写
    CanUpload    bool    `json:"can_upload"`         // 是否可上传
}
```

### 1.2 数据库表 (`server/model/index.go:20-24`)

```
Location(backend VARCHAR(16), path VARCHAR(512), PK(backend, path))
Share(id VARCHAR(64) PK, related_backend, related_path, params JSON, auth VARCHAR(4093))
Verification(key VARCHAR(512), code VARCHAR(4), expire DATETIME = now+10min)
```

`params` 列以 JSON 存储 Password / Users / Expire / Url / CanShare / CanManageOwn / CanRead / CanWrite / CanUpload。

### 1.3 Proof 结构体 (`server/model/share.go:20-26`)

```go
type Proof struct {
    Id      string  `json:"id"`                // Hash(Key+"::"+Value, 20)
    Key     string  `json:"key"`               // "password" | "email" | "code"
    Value   string  `json:"-"`                 // 实际值（密码哈希 / 邮箱 / 验证码）
    Message *string `json:"message,omitempty"` // 提示信息
    Error   *string `json:"error,omitempty"`   // 错误信息
}
```

---

## 二、共享链接创建

### 2.1 入口路由 (`server/routes.go:78-79`)

```
POST /api/share/{share}  →  ShareUpsert
```

中间件链：`ApiHeaders → SecureHeaders → SecureOrigin → BodyParser → CanManageShare → PluginInjector`

### 2.2 CanManageShare 中间件 (`server/middleware/session.go:96-158`)

创建/修改共享链接前，必须通过 **CanManageShare** 中间件的权限审查：

1. **新建场景**（`ShareGet(share_id)` 返回 `ErrNotFound`）：直接委托给 `SessionStart` 中间件，确保请求者是已登录用户即可
2. **修改已有链接**：
   - **场景 1**：请求者是链接的原始创建者 → `s.Backend == GenerateID(ctx.Session)` → 放行
   - **场景 2**：请求者不是原始创建者 → 重新提取 Share 信息，判断 `s.CanShare == true` 且当前会话的 `GenerateID` 匹配 `s.Backend` → 才放行

### 2.3 ShareUpsert 控制器 (`server/ctrl/share.go:35-94`)

构建 `Share` 对象的关键字段来源：

| 字段 | 来源逻辑 |
|------|----------|
| `Id` | URL 路径参数 `{share}` |
| `Auth` | 若 `ctx.Share.Id != ""`（分享者再分享），取 `ctx.Share.Auth`；否则拼接所有 `CookieName(index)` cookie 值（即创建者的加密凭证） |
| `Backend` | 若 `ctx.Share.Id != ""`，取 `ctx.Share.Backend`；否则取 `GenerateID(ctx.Session)`（创建者的后端指纹） |
| `Path` | 若 `ctx.Share.Id != ""`，以 `ctx.Share.Path` 为左路径；否则以 `ctx.Session["path"]` 为左路径；右侧拼上前端提交的 `path` |
| `Password` | 前端 `body.password`，经过 bcrypt 哈希后存储 |
| `Expire` | 前端 `body.expire`（Unix 毫秒时间戳） |
| `CanRead/CanWrite/CanUpload/CanShare/CanManageOwn` | 前端 body 直接传入 |

### 2.4 ShareUpsert 模型层 (`server/model/share.go:85-139`)

1. **密码处理**：若密码为 `PASSWORD_DUMMY`（`"{{PASSWORD}}"`），保留旧密码；否则 bcrypt 哈希
2. **Location 表**：先 `INSERT INTO Location(backend, path)`，若冲突（已存在）则忽略
3. **Share 表**：`INSERT ... ON CONFLICT(id) DO UPDATE SET ...`（upsert 语义）
4. `params` 字段仅序列化白名单字段（排除 Auth 等敏感信息）

---

## 三、Token / Proof 校验

共享链接的"token"并非传统 JWT，而是基于 **Proof（凭证证明）** 的多步验证机制。

### 3.1 匿名访问（无密码无用户限制）

当 `Share.Password == nil && Share.Users == nil` 时，`ShareProofGetRequired` 返回空切片 → `remainingProof` 长度为 0 → 直接通过。即**匿名可访问**。

### 3.2 Proof 验证入口路由

```
POST /api/share/{share}/proof  →  ShareVerifyProof
```

中间件链：`ApiHeaders → SecureHeaders → SecureOrigin → BodyParser → PluginInjector`（**无** SessionStart，即不要求已登录）

### 3.3 ShareVerifyProof 控制器 (`server/ctrl/share.go:106-213`)

完整验证流程：

```
1) 从 DB 读取 Share 记录
2) 获取已验证的 Proof（从 cookie "proof" 中解密）
3) 获取需要验证的 Proof（Share.Password → "password"; Share.Users → "email"）
4) 校验上下文合法性（Proof 数量 ≤ 20；Share 未过期）
5) 调用 model.ShareProofVerifier 验证当前提交的 Proof
6) 将新验证的 Proof 追加到已验证列表
7) 计算剩余待验证 Proof = required - verified
8) 将已验证 Proof 加密后写入 cookie "proof"（有效期 30 天）
9) 若仍有剩余 Proof → 返回下一个待验证项
   若全部通过 → 返回 Share 的 id/path/can_read/can_write/can_upload
```

### 3.4 Proof 验证细节 (`server/model/share.go:150-261`)

| Proof Key | 验证方式 |
|-----------|----------|
| `password` | `bcrypt.CompareHashAndPassword(存储哈希, 提交明文)`；失败则 sleep 1s 防暴力破解 |
| `email` | 先匹配 `Share.Users` 列表（支持 `*` 通配符域名）；匹配成功后生成 4 位验证码，存入 Verification 表（10 分钟有效），发送邮件；返回 key="code" |
| `code` | 在 Verification 表中查找验证码；成功后删除该记录并返回 key="email", value=邮箱地址 |

### 3.5 Proof 持久化 (`server/model/share.go:291-309`)

已验证的 Proof 存储在名为 `"proof"` 的加密 cookie 中：

- 加密密钥：`SECRET_KEY_DERIVATE_FOR_PROOF` = `Hash("PROOF_" + SECRET_KEY, len(SECRET_KEY))`
- 加密方式：`EncryptString(密钥, JSON(verifiedProof))`
- 读取时：`DecryptString(密钥, cookie值) → JSON反序列化 → []Proof`
- Cookie 有效期：30 天
- HttpOnly + SameSite=None + Secure

### 3.6 Proof 等价判断 (`server/model/share.go:341-354`)

```go
func shareProofAreEquivalent(ref Proof, p Proof) bool {
    if ref.Key != p.Key { return false }
    if ref.Value != "" && ref.Value == p.Value { return true }
    // 对于逗号分隔的值（如 Users 列表），逐段比较
    for _, chunk := range strings.Split(ref.Value, ",") {
        if p.Id == Hash(ref.Key+"::"+chunk, 20) { return true }
    }
    return false
}
```

---

## 四、会话权限合成（SessionStart 中间件）

### 4.1 SessionStart 中间件入口 (`server/middleware/session.go:57-83`)

每个请求进入时依次执行：

```
1) _extractShare(req)   → 填充 ctx.Share
2) _extractAuthorization(req) → 填充 ctx.Authorization
3) _extractSession(req, ctx)  → 填充 ctx.Session
4) _extractBackend(req, ctx)  → 填充 ctx.Backend
```

### 4.2 _extractShare (`server/middleware/session.go:202-260`)

核心逻辑——将共享链接"折叠"为临时会话：

```
1) 从 URL 参数 /mux 变量提取 share_id
2) 若 share_id 为空 → 返回空 Share（非共享访问）
3) 检查配置 features.share.enable 是否开启
4) 从 DB 获取 Share 记录
5) 调用 s.IsValid() 检查过期
6) 获取已验证的 Proof（从 cookie 或 Basic Auth header 解析）：
   - cookie "proof" → 解密 → []Proof
   - Authorization: Basic base64(user:pass) → 解析出 username/password
     - username 格式 "email[hash]" → 校验 hash → 匹配 Share.Users
     - password → bcrypt 比对匹配 Share.Password
   - 匹配成功则追加到 verifiedProof
7) 计算剩余 Proof = required - verified
8) 若 remainingProof 不为空 → 返回错误 "Unauthorized Shared space"
9) 全部通过 → 返回 Share 对象，设置到 ctx.Share
```

**关键点**：`_extractShare` 同时支持浏览器 cookie 验证和 WebDAV Basic Auth 验证。

### 4.3 _extractSession (`server/middleware/session.go:262-315`)

**共享链接场景**（`ctx.Share.Id != ""`）：

```
1) 解密 ctx.Share.Auth → 得到创建者的 session map
   （Auth 字段保存的是创建者登录时加密的凭证，用 SECRET_KEY_DERIVATE_FOR_USER 加密）
2) 设置 session["path"]：
   - 若 Share.Path 是目录 → session["path"] = ctx.Share.Path（chroot 根）
   - 若 Share.Path 是文件 → session["path"] = 去掉文件名的目录路径
     （且必须验证请求路径是 Share.Path 的后缀，防止路径遍历）
3) 返回 session（即创建者的凭证 + Share 限定的路径）
```

**普通登录场景**（`ctx.Share.Id == ""`）：

```
1) 解密 ctx.Authorization（cookie 或 Bearer token）→ session map
2) 校验 session["timestamp"] 不超过 365 天
3) 返回 session
```

### 4.4 权限合成总结

共享链接的"临时会话叠加在原有权限上"的实际含义：

1. **身份**：使用创建者的凭证（Auth 字段解密 → 创建者的 session）来建立后端连接
2. **路径**：用 `Share.Path` 替换/覆盖 session["path"]，实现路径 chroot
3. **权限**：通过 `ctx.Share.CanRead/CanWrite/CanUpload/CanShare` 限制操作，而非使用创建者原本的完整权限

也就是说：**后端连接继承创建者的身份，但操作权限被 Share 对象的三元组（CanRead, CanWrite, CanUpload）严格限制**。

---

## 五、文件 IO 时的最终权限判定

### 5.1 权限检查函数 (`server/model/permissions.go:1-33`)

```go
func CanRead(ctx)   bool { if ctx.Share.Id != "" { return ctx.Share.CanRead }   return true }
func CanEdit(ctx)   bool { if ctx.Share.Id != "" { return ctx.Share.CanWrite }  return true }
func CanUpload(ctx) bool { if ctx.Share.Id != "" { return ctx.Share.CanUpload } return true }
func CanShare(ctx)  bool { if ctx.Share.Id != "" { return ctx.Share.CanShare }  return true }
```

**判定规则**：若当前是共享会话（`ctx.Share.Id != ""`），权限完全由 Share 对象的布尔字段决定；否则（普通登录会话）一律返回 `true`。

### 5.2 各文件操作的权限判定矩阵

| 操作 | API 路由 | 主要权限检查 | 额外限制 |
|------|----------|-------------|----------|
| 列目录 (Ls) | `GET /api/files/ls` | `CanRead(ctx)` | 若不可读但可上传 → 返回空列表；还会调用 AuthorisationMiddleware 探测各项子权限 |
| 读文件 (Cat) | `GET /api/files/cat` | `CanRead(ctx)` | + AuthorisationMiddleware.Cat/Stat |
| 下载 (Downloader) | `GET /api/files/zip` | `CanRead(ctx)` | + AuthorisationMiddleware.Ls/Cat |
| 保存/上传 (Save) | `POST /api/files/cat` | `CanEdit(ctx)` ‖ `CanUpload(ctx)` | 仅可上传时，禁止覆盖已有文件；+ AuthorisationMiddleware.Save |
| 创建目录 (Mkdir) | `POST /api/files/mkdir` | `CanUpload(ctx)` | + AuthorisationMiddleware.Mkdir |
| 创建文件 (Touch) | `POST /api/files/touch` | `CanUpload(ctx)` | + AuthorisationMiddleware.Touch |
| 重命名/移动 (Mv) | `POST /api/files/mv` | `CanEdit(ctx)` | + AuthorisationMiddleware.Mv |
| 删除 (Rm) | `POST /api/files/rm` | `CanEdit(ctx)` | + AuthorisationMiddleware.Rm |
| 解压 (Extract) | `POST /api/files/unzip` | `CanRead(ctx)` | + AuthorisationMiddleware.Mkdir/Save |

### 5.3 FileLs 中的细粒度权限计算 (`server/ctrl/files.go:79-191`)

列目录时权限最为复杂，会进行多层计算：

```
1) 基础检查：CanRead 不可读但 CanUpload 可上传 → 返回空列表（只允许上传的盲投场景）
2) 后端 Meta：若 Backend 实现了 Meta(path) → 获取后端侧权限元数据
3) AuthorisationMiddleware 探测：对每个操作（Mkdir/Touch/Mv/Save/Rm/Cat）做试调用，
   失败则将对应的 Metadata 字段设为 false
4) Share 权限叠加：
   - CanEdit==false → 清除创建文件/目录/重命名/移动/删除/上传
   - CanUpload==false → 清除创建目录/重命名/移动/删除/上传
   - CanShare==false → 清除分享权限
5) 最终返回文件列表 + Metadata（前端据此显示/隐藏操作按钮）
```

### 5.4 PathBuilder (`server/ctrl/files.go:1104-1117`)

所有文件操作都经过 `PathBuilder`，将相对路径拼接为绝对路径并校验：

```go
func PathBuilder(ctx *App, path string) (string, error) {
    sessionPath := ctx.Session["path"]
    basePath := filepath.Join(sessionPath, path)
    // 校验：结果路径必须以 session["path"] 为前缀（chroot 防逃逸）
    if !strings.HasPrefix(basePath, ctx.Session["path"]) {
        return "", ErrFilesystemError
    }
    return basePath, nil
}
```

共享场景中 `session["path"]` 已被 `_extractSession` 设置为 `Share.Path`，因此所有操作被限制在共享路径下。

### 5.5 WebDAV 权限映射 (`server/ctrl/webdav.go:13-60`)

共享链接也支持 WebDAV 协议（`/s/{share}`），权限映射如下：

| WebDAV 方法 | 权限要求 |
|-------------|----------|
| OPTIONS, HEAD, GET, PROPFIND | CanRead |
| MKCOL, DELETE, COPY, MOVE, PROPPATCH | CanEdit (CanWrite) |
| PUT, LOCK, UNLOCK | CanEdit + CanUpload |

---

## 六、完整请求流程图

```
用户访问 /s/{share_id}
         │
         ▼
   SessionStart 中间件
         │
    ┌────┴────┐
    ▼         ▼
_extractShare  _extractAuthorization
    │              │
    │  1. 从DB取Share记录
    │  2. 检查过期 IsValid()
    │  3. 从cookie/BASIC AUTH收集已验证Proof
    │  4. 计算remainingProof
    │  5. remainingProof != 0 → 拒绝
    │  6. remainingProof == 0 → 通过，ctx.Share = s
    │
    ▼
_extractSession
    │
    │  ctx.Share.Id != "" ?
    │     是 → 解密 Share.Auth → 创建者session
    │          设置 session["path"] = Share.Path (chroot)
    │     否 → 解密 Authorization → 普通用户session
    │
    ▼
_extractBackend
    │
    │  model.NewBackend(ctx, ctx.Session)
    │  → 用合成后的session建立后端连接
    │  （继承创建者的后端身份）
    │
    ▼
  业务 Handler（FileLs/FileCat/FileSave/...）
    │
    │  1. 调用 model.CanRead/CanEdit/CanUpload/CanShare
    │     → 读取 ctx.Share 的布尔字段判定权限
    │  2. PathBuilder → 校验路径不逃逸 chroot
    │  3. AuthorisationMiddleware → 插件级权限探测
    │  4. 执行后端操作
    │
    ▼
   返回结果
```

---

## 七、关键加密体系

| 密钥派生 | 用途 |
|----------|------|
| `SECRET_KEY_DERIVATE_FOR_USER = Hash("USER_" + SECRET_KEY)` | 加密/解密用户 session cookie（含 Share.Auth） |
| `SECRET_KEY_DERIVATE_FOR_PROOF = Hash("PROOF_" + SECRET_KEY)` | 加密/解密 Proof cookie |
| `SECRET_KEY_DERIVATE_FOR_ADMIN = Hash("ADMIN_" + SECRET_KEY)` | 加密/解密 Admin token |
| `SECRET_KEY_DERIVATE_FOR_HASH = Hash("HASH_" + SECRET_KEY)` | 网络驱动器用户名哈希 |

更换 `secret_key` 会导致所有现有 session、共享链接的 Auth 字段和 Proof cookie 失效。

---

## 八、安全设计要点

1. **密码存储**：bcrypt 哈希，序列化时替换为 `{{PASSWORD}}` 占位符，永不泄露原始哈希
2. **暴力破解防护**：密码/邮箱验证失败时 sleep 1 秒
3. **路径逃逸防护**：PathBuilder + `_extractSession` 双重校验，确保请求路径在 chroot 范围内
4. **文件级共享**：共享单个文件时，session["path"] 设为文件所在目录，但额外校验请求路径必须以 Share.Path 为后缀
5. **Proof cookie**：加密 + HttpOnly + SameSite=None + Secure，有效期 30 天
6. **验证码**：10 分钟有效期，一次性使用后立即删除
7. **链接过期**：`Share.IsValid()` 在每次 `_extractShare` 和 `ShareVerifyProof` 中调用，毫秒级精度校验
