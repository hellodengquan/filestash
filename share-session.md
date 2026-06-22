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

---

## 九、链接撤销与已建立会话的失效路径

### 9.1 撤销入口

```
DELETE /api/share/{share}  →  ShareDelete
```

中间件链：`ApiHeaders → SecureHeaders → SecureOrigin → CanManageShare → PluginInjector`

前端 `modal_share.js:97-103` 中，用户点击删除按钮后调用 `DELETE /api/share/{id}`，删除成功后从前端 `state.links` 数组中移除该条目。

### 9.2 ShareDelete 控制器 (`server/ctrl/share.go:96-104`)

```go
func ShareDelete(ctx *App, res http.ResponseWriter, req *http.Request) {
    share_target := mux.Vars(req)["share"]
    if err := model.ShareDelete(share_target); err != nil {
        Log.Debug("share::delete '%s'", err.Error())
        SendErrorResult(res, err)
        return
    }
    SendSuccessResult(res, nil)
}
```

### 9.3 ShareDelete 模型层 (`server/model/share.go:141-148`)

```go
func ShareDelete(id string) error {
    stmt, err := DB.Prepare("DELETE FROM Share WHERE id = ?")
    if err != nil { return err }
    _, err = stmt.Exec(id)
    return err
}
```

仅执行一条 `DELETE FROM Share WHERE id = ?`，**不做任何额外清理**：不清除客户端的 Proof cookie，不吊销已建立的 Backend 连接缓存。

### 9.4 已建立会话的失效路径——"无状态即时失效"模型

Filestash 的共享会话**没有服务端 session 状态**，因此不存在"吊销会话"的操作。失效完全依赖每次请求时的实时校验：

```
已持有 Proof cookie 的用户继续访问 /s/{share_id}
         │
         ▼
   SessionStart → _extractShare(req)
         │
         │  1. _extractShareId(req) → 提取 share_id
         │  2. model.ShareGet(share_id)
         │     → DB 查询：DELETE 后已无此记录
         │     → 返回 ErrNotFound
         │  3. _extractShare 收到 err != nil
         │     → 返回 Share{}, nil（空 Share，不报错！）
         │
         ▼
   _extractSession(req, ctx)
         │
         │  ctx.Share.Id == ""（空 Share）
         │  → 走普通登录分支
         │  → 需要有效的 Authorization cookie/token
         │  → 匿名访问者没有 Authorization → session 为空 map
         │
         ▼
   _extractBackend(req, ctx)
         │
         │  model.NewBackend(ctx, ctx.Session)
         │  → 空 session 无法建立后端连接 → err
         │
         ▼
   SessionStart 返回 ErrNotAuthorized (401)
```

**关键发现**：`_extractShare` 中 `ShareGet` 返回错误时（包括 `ErrNotFound`），函数返回的是 `Share{}, nil`——**空 Share 且无错误**，而非将错误传递出去。这意味着撤销后的链接不会触发显式的"链接已删除"提示，而是静默降级为需要正常登录的状态。

### 9.5 Proof cookie 的残留与无害性

链接被撤销后，用户浏览器中仍残留加密的 `"proof"` cookie（有效期 30 天），但这不构成安全风险：

1. **Proof 无独立效力**：Proof cookie 只存储"已通过哪些验证"的记录，它必须与 DB 中存在的 Share 记录配合使用
2. **ShareGet 是硬依赖**：`_extractShare` 首先从 DB 获取 Share 记录，若记录不存在，后续的 Proof 匹配逻辑完全不会执行
3. **无回收机制**：服务端不会主动清除客户端的 Proof cookie，但 cookie 会在 30 天后自然过期

### 9.6 已缓存的 Backend 连接

`model.NewBackend` 内部可能有连接缓存，但每次请求都经过 `_extractShare → ShareGet` 的实时 DB 查询，因此：

- 即使 Backend 对象在缓存中存活，`_extractShare` 的 DB 查询失败会阻断整个请求链
- 缓存的 Backend 连接仅在 Share 记录仍然存在时才能被复用

### 9.7 过期导致的失效路径

与撤销不同，过期**不会**删除 DB 记录，而是通过 `Share.IsValid()` 判定：

```go
// server/common/types.go:196-204
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

过期的处理链路：

```
_extractShare → ShareGet(成功，记录仍在) → s.IsValid() → err="Link has expired"
→ _extractShare 返回 Share{}, NewError("Link has expired", 410)
→ SessionStart 返回 410 错误给客户端
```

`ShareVerifyProof` 中也调用 `s.IsValid()`（`server/ctrl/share.go:141-145`），确保过期链接无法继续验证新 Proof。

### 9.8 前端失效处理 (`public/assets/pages/ctrl_sharepage.js`)

前端 `ctrl_sharepage.js:84-97` 中的 `verify` 函数在每次加载分享页面时首先调用 `POST /api/share/{shareID}/proof`（body 为 null）：

- 若服务端返回 `key=""`（无待验证 Proof） → 进入 `"done"` 状态 → 跳转到文件浏览页
- 若服务端返回 `key="password"/"email"/"code"` → 显示对应的验证表单
- 若服务端返回错误（链接已删除 → 401，链接已过期 → 410） → 被 `ctrlError` 捕获 → 显示错误页面

### 9.9 撤销/过期失效路径对比

| 维度 | 链接撤销（DELETE） | 链接过期（Expire） |
|------|-------------------|-------------------|
| DB 记录 | 被删除 | 仍存在 |
| `_extractShare` 行为 | `ShareGet` 返回 `ErrNotFound` → 返回空 Share | `ShareGet` 成功 → `IsValid()` 返回 410 错误 |
| HTTP 状态码 | 401（Not authorised） | 410（Gone） |
| 客户端体验 | 静默降级为登录页 | 显式"Link has expired"错误 |
| Proof cookie | 残留但无害 | 残留但无用（IsValid 阻断在先） |
| 可逆性 | 不可逆（需重新创建 Share） | 不可逆（需更新 Expire 字段） |

---

## 十、附密码场景下的暴力破解防护机制

### 10.1 防护层次总览

```
                  请求进入
                    │
         ┌──────────┼──────────┐
         ▼          ▼          ▼
   第1层: HTTP    第2层: Proof  第3层: bcrypt
   RateLimiter   验证逻辑内    计算成本
   (全局限流)    sleep 延迟    (算法固有)
```

### 10.2 第1层：HTTP 全局速率限制 (`server/middleware/http.go:107-121`)

```go
var limiter = rate.NewLimiter(10, 1000)

func RateLimiter(fn HandlerFunc) HandlerFunc {
    return HandlerFunc(func(ctx *App, res http.ResponseWriter, req *http.Request) {
        if limiter.Allow() == false {
            Log.Warning("middleware::http::ratelimit too many requests")
            SendErrorResult(res, NewError(http.StatusText(429), 429))
            return
        }
        fn(ctx, res, req)
    })
}
```

- 使用 `golang.org/x/time/rate` 令牌桶算法
- **参数**：速率 10 QPS，桶容量 1000
- **作用域**：**全局单例**（非 per-IP），所有限流共享同一个 limiter
- **覆盖路由**：`POST /api/session`（登录）和 `POST /admin/api/session`（管理员登录）

**关键发现**：`POST /api/share/{share}/proof`（Proof 验证端点）的中间件链中**没有 RateLimiter**：

```
// server/routes.go:75
share.HandleFunc("/{share}/proof", NewMiddlewareChain(ShareVerifyProof,
    []Middleware{ApiHeaders, SecureHeaders, SecureOrigin, BodyParser, PluginInjector}))
```

这意味着 Proof 验证接口**不受 HTTP 层速率限制保护**，暴力破解防护完全依赖第2层和第3层。

### 10.3 第2层：验证逻辑内的 sleep 延迟 (`server/model/share.go:150-164`)

```go
func ShareProofVerifier(s Share, proof Proof) (Proof, error) {
    p := proof

    if proof.Key == "password" {
        if s.Password == nil {
            return p, NewError("No password required", 400)
        }

        v, ok := ShareProofVerifierPassword(*s.Password, proof.Value)
        if ok == false {
            time.Sleep(1000 * time.Millisecond)  // ← 失败后强制等待 1 秒
            return p, ErrInvalidPassword
        }
        p.Value = v
    }

    if proof.Key == "email" {
        // ...
        v, ok := ShareProofVerifierEmail(*s.Users, proof.Value)
        if ok == false {
            time.Sleep(1000 * time.Millisecond)  // ← 邮箱匹配失败也等待 1 秒
            return p, ErrNotAuthorized
        }
        // ...
    }
    // ...
}
```

- **延迟策略**：密码或邮箱验证失败时，`time.Sleep(1s)` 阻塞当前 goroutine
- **效果**：单次密码尝试至少耗时 1 秒（加上 bcrypt 计算时间），将理论最大尝试速率限制在约 1 次/秒
- **局限**：
  - 这是**协程级**阻塞，不影响其他并发请求
  - 攻击者可通过大量并发连接绕过单连接限速
  - 没有递增延迟（如指数退避），也没有锁定机制

### 10.4 第3层：bcrypt 计算成本 (`server/model/share.go:92-93`)

```go
hashedPassword, _ := bcrypt.GenerateFromPassword([]byte(*p.Password), bcrypt.DefaultCost)
```

```go
func ShareProofVerifierPassword(hashed string, given string) (string, bool) {
    if err := bcrypt.CompareHashAndPassword([]byte(hashed), []byte(given)); err != nil {
        return "", false
    }
    return hashed, true
}
```

- **cost 因子**：`bcrypt.DefaultCost = 10`，即 2^10 = 1024 轮迭代
- **单次验证耗时**：在现代 CPU 上约 70-100ms
- **安全效果**：即使没有 sleep 延迟，纯 bcrypt 的计算成本也将暴力尝试速率限制在约 10-14 次/秒/核心
- **与 sleep 叠加**：每次失败尝试 = bcrypt 计算（~100ms）+ sleep（1000ms）≈ 1.1 秒

### 10.5 Proof 数量溢出保护 (`server/ctrl/share.go:129-140`)

```go
if len(verifiedProof) > 20 || len(requiredProof) > 20 {
    http.SetCookie(res, &http.Cookie{
        Name:   COOKIE_NAME_PROOF,
        Value:  "",
        MaxAge: -1,
        Path:   COOKIE_PATH,
    })
    Log.Debug("share::verify::validate 'proof issue' ...")
    SendErrorResult(res, ErrNotValid)
    return
}
```

- 限制已验证 Proof 和所需 Proof 的数量均不超过 20
- 超出则**立即清除 Proof cookie**（`MaxAge: -1`），迫使攻击者从头开始验证
- 这防止了通过不断追加伪造 Proof 来进行填充攻击

### 10.6 Proof cookie 长度限制 (`server/model/share.go:300-301`)

```go
func ShareProofGetAlreadyVerified(req *http.Request) []Proof {
    // ...
    if len(cookieValue) > 500 {
        return p  // cookie 过长则忽略，返回空 Proof 列表
    }
    // ...
}
```

- Proof cookie 解密前先检查长度，超过 500 字节则直接丢弃
- 防止恶意构造的超长 cookie 耗尽解密资源

### 10.7 验证码防护 (`server/model/share.go:178-258`)

邮箱验证流程中的防护：

1. **一次性使用**：验证码被成功使用后立即 `DELETE FROM Verification WHERE code = ?`（`share.go:252-255`）
2. **时间限制**：Verification 表中 `expire DATETIME DEFAULT (datetime('now', '+10 minutes'))`，查询时 `WHERE expire > datetime('now')`（`share.go:233`）
3. **自动清理**：后台每 6 小时执行 `DELETE FROM Verification WHERE expire < datetime('now')`（`model/index.go:41-46`）
4. **短验证码**：仅 4 位随机字符（`RandomString(4)`），本身存在碰撞风险，但结合10分钟有效期和邮件投递门槛，实际威胁有限

### 10.8 前端交互层的反馈 (`public/assets/pages/ctrl_sharepage.js:111-138`)

```javascript
// STEP2: attempt to login
rxjs.switchMap((creds) => verify(render, { shareID, body: creds, setState }).pipe(
    rxjs.catchError((err) => {
        if (err instanceof AjaxError) {
            switch (err.code()) {
            case "INTERNAL_SERVER_ERROR":
                return rxjs.throwError(err);
            case "FORBIDDEN":
                return rxjs.of(false);  // ← 密码错误返回 false
            }
        }
        // ...
    })
)),
// STEP3: update the UI when authentication fails
rxjs.filter((ok) => !ok),
rxjs.mapTo(["name", "arrow_right"]), applyMutation(qs($page, "component-icon"), "setAttribute"),
rxjs.mapTo(""), stateMutation(qs($page, "input"), "value"),
rxjs.mapTo(["error"]), applyMutation(qs($page, ".input_group"), "classList", "add"),
rxjs.delay(300), applyMutation(qs($page, ".input_group"), "classList", "remove"),
```

- 密码错误时前端显示 "error" CSS 类（红色边框闪烁 300ms），然后清空输入框
- **无尝试次数限制**：前端不限制最大重试次数，不显示剩余次数
- **无验证码/CAPTCHA**：多次失败后不触发人机验证
- **无账户锁定**：不锁定 Share 链接

### 10.9 暴力破解防护综合评估

| 防护层 | 机制 | 强度 | 局限 |
|--------|------|------|------|
| bcrypt 计算成本 | cost=10, ~100ms/次 | 中 | GPU 加速仍可达数千次/秒 |
| sleep 延迟 | 失败后 1s | 中 | 仅阻塞单协程，并发可绕过 |
| Proof 数量限制 | >20 则清 cookie | 低 | 主要防填充攻击，非防暴力破解 |
| cookie 长度限制 | >500 字节忽略 | 低 | 防资源耗尽，非防暴力破解 |
| HTTP RateLimiter | 10 QPS / 桶容量 1000 | 高 | **但未覆盖 Proof 验证端点** |
| 验证码 (email) | 10 分钟有效，一次性 | 中 | 仅限 email+code 场景 |
| 前端限制 | 无 | 低 | 无重试次数限制、无 CAPTCHA |

**主要安全缺口**：Proof 验证端点（`POST /api/share/{share}/proof`）缺少 per-IP 速率限制。攻击者可以并发方式绕过 sleep 延迟，利用 GPU 加速 bcrypt 破解，理论上在合理时间内尝试大量密码。

**改进建议**：
1. 为 Proof 验证端点增加 per-IP 速率限制（如 5 次/分钟/IP）
2. 实现指数退避：连续失败后 sleep 时间递增（1s → 2s → 4s → ...）
3. 引入 CAPTCHA 机制：连续 N 次失败后要求人机验证
4. 增加审计日志：记录密码验证失败事件，便于异常检测

---

## 十一、链接过期边界问题

### 11.1 IsValid() 边界定义

```go
// server/common/types.go:196-204
func (s Share) IsValid() error {
    if s.Expire != nil {
        now := time.Now().UnixNano() / 1000000  // 当前时间，毫秒精度
        if now > *s.Expire {                    // 严格大于 → 过期
            return NewError("Link has expired", 410)
        }
    }
    return nil
}
```

**边界语义**：`now > expire` 为严格大于，`now == expire` 时仍有效。过期时间是**闭区间右开**语义：`[创建时间, 过期时间)`。

### 11.2 过期检查的两个关键位置

| 检查位置 | 代码行 | 检查时机 | 影响范围 |
|----------|--------|----------|----------|
| `_extractShare` | `server/middleware/session.go:217` | 请求进入 SessionStart 中间件时 | 所有共享链接访问（ls/cat/save/zip 等） |
| `ShareVerifyProof` | `server/ctrl/share.go:141-145` | 用户提交 Proof 验证时 | Proof 验证流程本身 |

**关键观察**：`IsValid()` 检查**只发生在请求的入口处**（中间件层），在 IO 过程中不会再次检查。

### 11.3 过期时间点前开始的请求

场景：用户在 T0 时刻（T0 < expire）发起请求，请求处理时间较长，在 T1 时刻（T1 > expire）才完成。

```
   T0                    T_expire                 T1
   │───────────────────────│──────────────────────│
   发起请求               过期时间               请求完成
              请求处理在过期时间点之后继续
```

处理逻辑：

```
GET /api/files/cat?path=large_file.iso
    │
    ▼
SessionStart → _extractShare
    │
    │  T0 < *s.Expire → IsValid() → nil → 通过
    │  ctx.Share 已设置，权限已合成
    │
    ▼
FileCat handler
    │
    │  1. CanRead(ctx) → ctx.Share.CanRead → true
    │  2. PathBuilder(ctx, path) → 路径校验通过
    │  3. ctx.Backend.Cat(path) → 返回 io.Reader
    │  4. io.Copy(w, f) → 流式写入 ResponseWriter
    │     ↑ 这个过程可能耗时数分钟到数小时
    │     ↑ 期间不会再次调用 IsValid()
    │
    ▼
   请求完成（即使 T1 > expire）
```

**结论**：过期检查是"**进门检查**"，一旦通过 SessionStart 中间件，后续 IO 过程不会再次校验过期时间。长耗时请求（如下载大文件、流式 ZIP 打包）可以在过期时间点之后继续完成。

### 11.4 过期时间点后发起的新请求

```
用户在 T1 > expire 时发起新请求
    │
    ▼
SessionStart → _extractShare
    │
    │  1. model.ShareGet(share_id) → 成功，记录仍在
    │  2. s.IsValid() → T1 > *s.Expire → err = "Link has expired"
    │  3. _extractShare 返回 Share{}, err
    │
    ▼
SessionStart 捕获错误 → SendErrorResult(res, err)
    │
    ▼
返回 410 Gone，body: {"error": "Link has expired"}
```

### 11.5 长连接 / 流式传输场景下的过期边界

#### 11.5.1 文件下载（FileCat / Downloader）

- **FileCat**（`/api/files/cat`）：仅在入口检查一次，流式传输过程中不检查
- **FileDownloader**（`/api/files/zip`）：仅在入口检查一次，ZIP 打包和流式传输过程中不检查

**重要配置**：`features.protection.zip_timeout`（默认 60 秒）限制 ZIP 打包总时长，与过期检查相互独立。

#### 11.5.2 分片上传（TUS 协议）

分片上传较为复杂，涉及多个 HTTP 请求：

```
POST   /api/files/cat?proto=tus   → 创建上传会话（入口检查 IsValid）
OPTIONS /api/files/cat?proto=tus  → 无 IsValid 检查
HEAD   /api/files/cat?proto=tus  → 查询偏移量
PATCH  /api/files/cat?proto=tus  → 上传分片
```

检查点分析：

1. **创建会话（POST）**：经过 SessionStart → 检查 IsValid → 必须在过期前调用
2. **上传分片（PATCH）**：也经过 SessionStart → **每次 PATCH 都会检查 IsValid**
3. **会话缓存**：`chunkedUploadCache` 按 `(path, GenerateID(ctx.Session))` 缓存，与过期检查独立

**分片上传的过期边界**：

| 场景 | 行为 |
|------|------|
| POST 创建会话在过期前，PATCH 分片也在过期前 | ✅ 正常上传 |
| POST 在过期前，第一个 PATCH 在过期前，后续 PATCH 在过期后 | ❌ 后续 PATCH 会失败（410），已上传分片留在缓存，1 天过期 |
| POST 在过期后 | ❌ 创建失败（410） |

#### 11.5.3 WebDAV 连接

WebDAV 是按请求（method）处理的，每个请求都经过 SessionStart 中间件：

```
PROPFIND /s/{share}/dir → SessionStart → IsValid()
GET      /s/{share}/file → SessionStart → IsValid()
PUT      /s/{share}/file → SessionStart → IsValid()
```

每个 WebDAV 请求都会独立检查过期时间。若 GET 请求下载文件耗时较长，行为与 FileCat 相同——进门后不再检查。

#### 11.5.4 WOPI / OnlyOffice 在线编辑

`plg_editor_wopi/handler.go:282` 和 `plg_editor_onlyoffice/index.go:331` 会将 `ctx.Share.Id` 附加到编辑会话 ID 中：

```go
// plg_editor_wopi/handler.go:282-283
if ctx.Share.Id != "" {
    wopiSRC += "::" + ctx.Share.Id
}
```

但编辑会话本身有自己的生命周期管理。当 Share 链接过期后，后续的 WOPI 回调请求经过 SessionStart 时会被拒绝，但已加载到前端的文档可能仍可编辑（只是无法保存）。

### 11.6 Proof cookie 在过期后的残留

Proof cookie 有效期 30 天，与 Share 过期时间独立。链接过期后：

1. 客户端仍持有有效的 Proof cookie（已通过密码/邮箱验证）
2. 但 `_extractShare` 中 `IsValid()` 检查先于 Proof 匹配执行
3. 因此即使 Proof cookie 有效，链接过期后也无法访问
4. Proof cookie 不会被自动清除，会在浏览器中残留直到 30 天后自然过期

### 11.7 过期边界问题总结

| 操作类型 | 过期检查点 | 过期后行为 |
|----------|------------|------------|
| 短请求（ls/mkdir/rm/mv） | 每次请求入口 | 立即 410 |
| 文件下载（cat） | 请求入口 | 已开始的下载可继续完成 |
| ZIP 打包下载 | 请求入口 | 已开始的打包可继续，受 zip_timeout 限制 |
| 分片上传（TUS） | 每次 PATCH 请求入口 | 过期后的 PATCH 会失败，已上传分片保留 |
| WebDAV | 每个方法请求入口 | 过期后的新请求失败，已开始的 GET 可继续 |
| 在线编辑（WOPI） | 每次回调请求入口 | 过期后无法保存，前端可能仍可查看 |
| Proof 验证 | ShareVerifyProof 入口 | 过期后无法继续验证流程 |

---

## 十二、多个链接覆盖同一文件时的权限冲突规则

### 12.1 数据模型层面——多对多关系

数据库设计天然支持一个文件/目录被多个 Share 链接引用：

```sql
-- server/model/index.go:20-24
CREATE TABLE IF NOT EXISTS Location(
    backend VARCHAR(16),
    path VARCHAR(512),
    CONSTRAINT pk_location PRIMARY KEY(backend, path)
);
CREATE TABLE IF NOT EXISTS Share(
    id VARCHAR(64) PRIMARY KEY,
    related_backend VARCHAR(16),
    related_path VARCHAR(512),
    params JSON,
    auth VARCHAR(4093) NOT NULL,
    FOREIGN KEY (related_backend, related_path)
        REFERENCES Location(backend, path)
        ON UPDATE CASCADE ON DELETE CASCADE
);
```

- `Location` 表记录所有被共享过的路径（唯一键：`backend + path`）
- `Share` 表通过外键关联到 `Location`，允许多个 Share 指向同一个 Location
- `ON DELETE CASCADE`：删除 Location 行会级联删除所有关联的 Share 行（但 Location 行不会主动删除）

### 12.2 ShareList 查询——前缀匹配，返回全部

```go
// server/model/share.go:28-46
func ShareList(backend string, path string) ([]Share, error) {
    stmt, err := DB.Prepare(
        "SELECT id, related_path, params FROM Share " +
        "WHERE related_backend = ? AND related_path LIKE ? || '%' ")
    // ...
    rows, err := stmt.Query(backend, path)
    // ...
}
```

**查询逻辑**：`related_path LIKE ? || '%'` 是前缀匹配。

当查询 `/a/b/` 时，返回：
- `/a/b/`（自身被共享）
- `/a/b/c/`（子目录被共享）
- `/a/b/c/file.txt`（子文件被共享）

这意味着：
1. 同一文件可能出现在多个父目录的 ShareList 结果中
2. 前端展示"已有共享链接"时，会显示所有覆盖该路径的链接（按前缀匹配）

### 12.3 访问时的链接选择——URL 决定一切

用户通过哪个共享链接访问，**完全由 URL 中的 `share_id` 决定**，这是最核心的规则：

```
/s/{share_id}/path/to/file
    ↑
    这个 share_id 决定了使用哪个 Share 记录的权限
```

```go
// server/middleware/session.go:202-205
func _extractShareId(req *http.Request) string {
    share_id := req.URL.Query().Get("share")
    if share_id == "" {
        if mux.Vars(req)["share"] != "" {
            share_id = mux.Vars(req)["share"]
        }
    }
    return share_id
}
```

优先级：
1. URL query 参数 `?share=xxx`
2. mux 路由变量 `/s/{share}`

**无冲突**：请求只能携带一个 `share_id`，同一时刻只应用一个 Share 链接的权限。

### 12.4 权限不合并——取当前链接的布尔值

`CanRead/CanEdit/CanUpload/CanShare` 四个权限函数的逻辑是：

```go
// server/model/permissions.go:1-33
func CanRead(ctx *App) bool {
    if ctx.Share.Id != "" {
        return ctx.Share.CanRead  // 只取当前 Share 的值，不与其他 Share 合并
    }
    return true
}
// CanEdit → ctx.Share.CanWrite
// CanUpload → ctx.Share.CanUpload
// CanShare → ctx.Share.CanShare
```

**核心规则**：权限只从 `ctx.Share`（当前 URL 指向的那个 Share 链接）读取，**永远不会**与其他覆盖同一文件的 Share 链接做合并（OR/AND）。

示例场景：

```
文件 /data/report.pdf 同时被两个链接共享：
  链接 A（id=abc123）: CanRead=true,  CanWrite=false, CanUpload=false, 密码保护
  链接 B（id=xyz789）: CanRead=true,  CanWrite=true,  CanUpload=true,  无密码

用户通过 /s/abc123/path/report.pdf 访问 → 使用链接 A 的权限（只读）
用户通过 /s/xyz789/path/report.pdf 访问 → 使用链接 B 的权限（读写+上传）
```

### 12.5 元数据探测中的多链接交互

在 `FileLs` 的细粒度权限计算中（`server/ctrl/files.go:79-191`），会通过 `AuthorisationMiddleware` 对每个操作做试调用探测。

**关键发现**：`AuthorisationMiddleware` 的 `ctx` 中只包含当前 Share，因此探测结果只反映当前 Share 的权限，不会考虑其他 Share 链接。

```
FileLs(ctx)
    ├── CanRead(ctx) → ctx.Share.CanRead
    ├── AuthorisationMiddleware.Ls/Mkdir/Touch/Save(...)
    │   └── 每个探测函数的 ctx 包含同一个 Share
    └── 最后叠加 Share 权限清除不可用操作
```

### 12.6 链接创建时的冲突处理

当对同一文件创建第二个共享链接时：

```go
// server/model/share.go:97-110
stmt, err := DB.Prepare("INSERT INTO Location(backend, path) VALUES($1, $2)")
_, err = stmt.Exec(p.Backend, p.Path)
if err != nil {
    throw := true
    if sqlite.IsConstraint(err) {
        throw = false  // 唯一键冲突 → 静默忽略
    }
    // ...
}
```

- `Location` 表插入冲突（该路径已有记录）→ 静默忽略，不报错
- `Share` 表使用 `ON CONFLICT(id) DO UPDATE`，若 `id` 相同则更新，若 `id` 不同则新增

**无去重逻辑**：对同一文件可以创建任意多个不同 `id` 的 Share 链接，彼此独立。

### 12.7 再分享（N 级分享）场景下的权限叠加

若用户 A 分享 `/a/` 给用户 B（链接 A，CanShare=true），用户 B 再分享 `/a/b/` 给用户 C（链接 B）：

| 维度 | 链接 A（原始分享） | 链接 B（再分享） |
|------|------------------|----------------|
| `Backend` | `GenerateID(A的session)` | `ctx.Share.Backend` = 链接 A 的 Backend（继承） |
| `Auth` | A 的加密凭证 | `ctx.Share.Auth` = 链接 A 的 Auth（继承） |
| `Path` | `/a/` | `/a/b/`（路径更窄） |
| 权限 | A 设置的 CanRead/Write/Upload | B 设置的 CanRead/Write/Upload |
| `CanShare` | true | B 可设置为 true 或 false |

**权限传播规则**：
- 后端身份链继承不变（都使用 A 的凭证）
- 路径只能收窄，不能扩大（子目录/文件）
- 再分享的权限可以比原始链接**更严**，但**不能更宽**——因为原始链接的 Share 仍在 `ctx.Share` 链中？

**重要澄清**：再分享时，`CanManageShare` 中间件的检查（`middleware/session.go:142-158`）确保：
- 请求者必须能访问到原始链接（即 Proof 验证通过）
- 原始链接的 `CanShare` 必须为 true

但创建出的新 Share 链接的权限**完全由创建者（B）在前端设置**，代码中没有强制"新权限必须是原权限的子集"的校验。这意味着存在一个**潜在的权限扩大风险**：B 可以将 A 分享的只读目录，以可写权限再次分享给 C。

### 12.8 前端展示的多链接列表

前端 `modal_share.js:130-185` 中，`ctrlListShares` 调用 `GET /api/share?path=xxx` 获取所有覆盖当前路径的共享链接，展示为列表供用户编辑/删除。

```
GET /api/share?path=/data/
    │
    ▼
ShareList(backend, "/data/")
    │
    ▼
SELECT ... WHERE related_path LIKE '/data/%'
    │
    ▼
返回所有覆盖 /data/ 及其子路径的 Share 链接
```

用户可以对列表中的任一链接进行编辑或删除，操作独立，互不影响。

### 12.9 多链接覆盖的权限冲突总结

| 场景 | 规则 |
|------|------|
| 访问时使用哪个链接的权限 | URL 中的 share_id 决定，无合并 |
| 多个链接权限如何合并 | 不合并，只使用当前 Share 的布尔值 |
| 同一文件能否创建多个链接 | 可以，彼此独立 |
| 再分享能否扩大权限 | 代码未强制限制，存在扩大风险 |
| 链接删除对其他链接的影响 | 无影响，独立删除 |
| FileLs 元数据探测 | 只基于当前 Share，不考虑其他链接 |
| Location 表冲突处理 | 唯一键冲突静默忽略 |

**设计哲学**：Share 链接是完全独立的访问令牌，每个链接有自己的权限配置和生命周期。访问时 URL 是唯一的权限来源，不存在"同一文件的多个链接权限取并集/交集"的逻辑。
