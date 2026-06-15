# 共享链接权限系统分析文档

## 一、系统架构总览

Filestash 的共享链接权限系统采用**三层验证 + 五级权限**架构，实现了从凭证签发、身份验证、动作授予到过期回收的完整权限生命周期管理。

```
                    ┌─────────────────────┐
                    │   凭证签发层 (Auth) │
                    │  - Session 加密存储  │
                    │  - Proof Cookie     │
                    └─────────┬───────────┘
                              │
                    ┌─────────▼───────────┐
                    │  身份验证层 (Verify) │
                    │  - 密码(bcrypt)      │
                    │  - 邮箱验证码        │
                    │  - HTTP Basic Auth  │
                    └─────────┬───────────┘
                              │
                    ┌─────────▼───────────┐
                    │  动作授予层 (Permit) │
                    │  - CanRead / CanWrite│
                    │  - CanUpload         │
                    │  - CanShare          │
                    │  - CanManageOwn      │
                    └─────────┬───────────┘
                              │
                    ┌─────────▼───────────┐
                    │ 过期回收层 (Expire)  │
                    │  - 时间戳校验        │
                    │  - 数据库清理        │
                    │  - Cookie 失效       │
                    └─────────────────────┘
```

核心代码分布：
- 数据模型：`server/model/share.go`、`server/common/types.go:180-204`
- 中间件验证：`server/middleware/session.go`
- 控制器逻辑：`server/ctrl/share.go`、`server/ctrl/files.go`
- 权限判定：`server/model/permissions.go`
- 路由配置：`server/routes.go:70-79`

---

## 二、凭证签发机制

### 2.1 密钥派生体系

所有密钥均从主密钥 `SECRET_KEY` 通过 SHA-256 哈希派生，实现密钥用途隔离：

**文件**：`server/common/constants.go`

| 派生密钥 | 用途 |
|---------|------|
| `SECRET_KEY_DERIVATE_FOR_USER` | 加密/解密 Session Cookie（包含后端连接凭证） |
| `SECRET_KEY_DERIVATE_FOR_PROOF` | 加密/解密 Proof Cookie（共享访问验证状态） |
| `SECRET_KEY_DERIVATE_FOR_ADMIN` | 加密/解密管理员会话 |
| `SECRET_KEY_DERIVATE_FOR_HASH` | 生成邮箱哈希校验码（网络驱动器认证） |

### 2.2 加密算法

采用 **AES-256-GCM** 对称加密 + **Zlib** 压缩的组合：

**文件**：`server/common/crypto.go`

```go
func EncryptString(secret string, data string) (string, error) {
    d, _ := compress([]byte(data))              // Zlib 压缩
    d, _ = EncryptAESGCM([]byte(secret), d)     // AES-GCM 加密
    return base64.URLEncoding.EncodeToString(d), nil  // URL安全Base64
}
```

### 2.3 Share 凭证签发流程

当用户创建共享链接时，创建者的完整 Session（含后端连接凭证）被加密后存入 Share 记录的 `auth` 字段：

**文件**：`server/ctrl/share.go:35-94`

```go
func ShareUpsert(ctx *App, res http.ResponseWriter, req *http.Request) {
    share_id := mux.Vars(req)["share"]
    s := Share{
        Id: share_id,
        Auth: func() string {
            if ctx.Share.Id == "" {
                // 普通用户创建：收集所有 auth Cookie 拼接
                str := ""
                index := 0
                for {
                    cookie, err := req.Cookie(CookieName(index))
                    if err != nil { break }
                    index++
                    str += cookie.Value
                }
                return str
            }
            // 嵌套共享场景：复用父共享的 auth 凭证
            return ctx.Share.Auth
        }(),
        Backend: func() string {
            if ctx.Share.Id == "" {
                return GenerateID(ctx.Session)  // 生成后端唯一标识
            }
            return ctx.Share.Backend
        }(),
        Path: /* 拼接完整路径，考虑父共享嵌套 */,
        // ... 权限位、密码、过期时间等
    }
    model.ShareUpsert(&s)
}
```

**签发关键点**：
1. **Auth 字段**：存储创建者的加密 Session，包含后端类型、用户名、密码、路径等完整连接信息
2. **Backend 字段**：通过 `GenerateID(ctx.Session)` 生成，用于标识共享归属
3. **嵌套共享**：通过共享链接再创建子共享时，直接复用父共享的 `auth` 凭证
4. **密码处理**：使用 bcrypt 哈希存储，`PASSWORD_DUMMY` 占位符用于前端回显

**密码哈希存储**：
**文件**：`server/model/share.go:86-95`

```go
func ShareUpsert(p *Share) error {
    if p.Password != nil {
        if *p.Password == PASSWORD_DUMMY {
            // 前端回传占位符，保留原密码
            if s, err := ShareGet(p.Id); err == nil {
                p.Password = s.Password
            }
        } else {
            // 新密码：bcrypt 哈希
            hashedPassword, _ := bcrypt.GenerateFromPassword([]byte(*p.Password), bcrypt.DefaultCost)
            p.Password = NewString(string(hashedPassword))
        }
    }
    // ... 写入数据库
}
```

### 2.4 Backend ID 生成算法

用于唯一标识某个后端连接会话：

**文件**：`server/common/crypto.go`

```go
func GenerateID(params map[string]string) string {
    p := ""
    for _, key := range sortedKeys(params) {
        switch key {
        case "password", "path", "session", "timestamp": // 排除敏感/易变字段
        default:
            if val := params[key]; val != "" {
                p += key + "=>" + val + ", "
            }
        }
    }
    p += "salt=>" + SECRET_KEY  // 混入主密钥防止伪造
    return Hash(p, 20)
}
```

此 ID 用于：
1. 关联 Share 记录与创建者的后端（`related_backend` 字段）
2. 判断共享管理权归属（创建者校验）

### 2.5 数据库存储结构

**文件**：`server/model/index.go:20-33`

```sql
CREATE TABLE IF NOT EXISTS Location(
    backend VARCHAR(16), 
    path VARCHAR(512), 
    PRIMARY KEY(backend, path)
);

CREATE TABLE IF NOT EXISTS Share(
    id VARCHAR(64) PRIMARY KEY,
    related_backend VARCHAR(16),
    related_path VARCHAR(512),
    params JSON,          -- 密码哈希、邮箱列表、过期时间、权限位等
    auth VARCHAR(4093) NOT NULL,  -- 加密的创建者 Session
    FOREIGN KEY (related_backend, related_path) 
        REFERENCES Location(backend, path) 
        ON UPDATE CASCADE ON DELETE CASCADE
);

CREATE TABLE IF NOT EXISTS Verification(
    key VARCHAR(512), 
    code VARCHAR(4), 
    expire DATETIME DEFAULT (datetime('now', '+10 minutes'))
);
```

---

## 三、身份验证与凭证验证

### 3.1 共享请求的验证链路

所有请求经过 `SessionStart` 中间件，其中 `_extractShare` 函数完成共享上下文提取与验证：

**文件**：`server/middleware/session.go:202-260`

```go
func _extractShare(req *http.Request) (Share, error) {
    // 1. 提取 share_id（URL query 或 path variable）
    share_id := _extractShareId(req)
    if share_id == "" { return Share{}, nil }
    
    // 2. 检查共享功能是否启用
    if Config.Get("features.share.enable").Bool() == false {
        return Share{}, NewError("Feature isn't enabled", 405)
    }

    // 3. 从数据库读取 Share 记录
    s, err := model.ShareGet(share_id)
    if err != nil { return Share{}, nil }
    
    // 4. 校验是否过期
    if err = s.IsValid(); err != nil {
        return Share{}, err
    }

    // 5. 获取已验证凭证（从 Proof Cookie）
    var verifiedProof []model.Proof = model.ShareProofGetAlreadyVerified(req)
    
    // 6. 支持 HTTP Basic Auth（WebDAV 场景）
    username, password := parseBasicAuth(req.Header.Get("Authorization"))
    if s.Users != nil && username != "" {
        if v, ok := model.ShareProofVerifierEmail(*s.Users, username); ok {
            verifiedProof = append(verifiedProof, model.Proof{Key: "email", Value: v})
        }
    }
    if s.Password != nil && password != "" {
        if v, ok := model.ShareProofVerifierPassword(*s.Password, password); ok {
            verifiedProof = append(verifiedProof, model.Proof{Key: "password", Value: v})
        }
    }

    // 7. 计算剩余需要验证的凭证
    requiredProof := model.ShareProofGetRequired(s)
    remainingProof := model.ShareProofCalculateRemainings(requiredProof, verifiedProof)
    if len(remainingProof) != 0 {
        return Share{}, NewError("Unauthorized Shared space", 400)
    }
    return s, nil  // 验证通过
}
```

### 3.2 Proof 验证机制

Proof 系统支持两种验证方式，可**组合使用**（需同时满足）：

#### 方式一：密码验证

**文件**：`server/model/share.go:263-268`

```go
func ShareProofVerifierPassword(hashed string, given string) (string, bool) {
    // 使用 bcrypt 进行慢哈希比对，防止暴力破解
    if err := bcrypt.CompareHashAndPassword([]byte(hashed), []byte(given)); err != nil {
        return "", false
    }
    return hashed, true
}
```

安全措施：
- 密码使用 **bcrypt** 算法存储（默认 cost）
- 验证失败后故意 **Sleep 1秒** 防止爆破（`server/model/share.go:160`）
- 密码哈希值本身作为 Proof 的 Value，用于后续等价性判断

#### 方式二：邮箱白名单验证

支持**精确匹配**和**通配符后缀匹配**：

**文件**：`server/model/share.go:269-289`

```go
func ShareProofVerifierEmail(users string, wanted string) (string, bool) {
    for _, possibleUser := range strings.Split(users, ",") {
        possibleUser = strings.Trim(possibleUser, " ")
        if wanted == possibleUser {  // 精确匹配
            return possibleUser, true
        } else if possibleUser[0:1] == "*" {  // 通配符：*@company.com
            if strings.HasSuffix(wanted, strings.TrimPrefix(possibleUser, "*")) {
                return possibleUser, true
            }
        }
    }
    return "", false
}
```

邮箱验证**两步流程**：
1. **提交邮箱** → 系统生成 4 位随机验证码 → 发送验证邮件（`server/model/share.go:166-229`）
2. **提交验证码** → 系统从 `Verification` 表查询 → 匹配后立即删除（一次性使用）

验证码存储特性：
- **有效期**：10 分钟（数据库默认值）
- **一次性**：使用后立即删除（`server/model/share.go:252-255`）
- **自动清理**：后台 goroutine 每 6 小时清理过期记录

### 3.3 Proof 验证控制器

**文件**：`server/ctrl/share.go:106-213`

```go
func ShareVerifyProof(ctx *App, res http.ResponseWriter, req *http.Request) {
    // 1. 初始化上下文
    s, _ := model.ShareGet(share_id)
    submittedProof := model.Proof{Key: type, Value: value}
    verifiedProof := model.ShareProofGetAlreadyVerified(req)
    requiredProof := model.ShareProofGetRequired(s)

    // 2. 防滥用检查
    if len(verifiedProof) > 20 || len(requiredProof) > 20 {
        // 强制清空 Proof Cookie
        http.SetCookie(res, &http.Cookie{
            Name: COOKIE_NAME_PROOF, Value: "", MaxAge: -1, Path: COOKIE_PATH,
        })
        return
    }
    
    // 3. 验证共享链接有效性
    if err := s.IsValid(); err != nil {
        SendErrorResult(res, err)
        return
    }

    // 4. 处理提交的 Proof
    submittedProof, err = model.ShareProofVerifier(s, submittedProof)
    if err != nil {
        submittedProof.Error = NewString(err.Error())
        SendSuccessResult(res, submittedProof)
        return
    }
    
    // 5. 邮箱验证码特殊处理（发送后返回提示）
    if submittedProof.Key == "code" {
        submittedProof.Value = ""
        submittedProof.Message = NewString("We've sent you a message with a verification code")
        SendSuccessResult(res, submittedProof)
        return
    }

    // 6. 将验证通过的 Proof 加入已验证列表
    if submittedProof.Key != "" {
        submittedProof.Id = Hash(submittedProof.Key+"::"+submittedProof.Value, 20)
        // 去重检查
        if alreadyExist == false {
            verifiedProof = append(verifiedProof, submittedProof)
        }
    }

    // 7. 计算剩余需要验证的 Proof
    remainingProof = model.ShareProofCalculateRemainings(requiredProof, verifiedProof)

    // 8. 持久化 Proof 到 Cookie
    cookie := http.Cookie{
        Name: COOKIE_NAME_PROOF,
        Value: EncryptString(SECRET_KEY_DERIVATE_FOR_PROOF, json.Marshal(verifiedProof)),
        Path:     COOKIE_PATH,
        MaxAge:   60 * 60 * 24 * 30,  // 30 天
        HttpOnly: true,
        SameSite: http.SameSiteNoneMode,
        Secure:   true,
    }
    http.SetCookie(res, &cookie)

    // 9. 返回结果
    if len(remainingProof) > 0 {
        SendSuccessResult(res, remainingProof[0])  // 返回下一个需要验证的 Proof
        return
    }
    // 全部验证通过 → 返回权限信息
    SendSuccessResult(res, struct {
        Id, Path string
        CanRead, CanWrite, CanUpload bool
    }{s.Id, s.Path, s.CanRead, s.CanWrite, s.CanUpload})
}
```

### 3.4 Proof Cookie 持久化与等价性判断

验证通过的 Proof 被加密后存入 Cookie：

**文件**：`server/model/share.go:291-309`

```go
func ShareProofGetAlreadyVerified(req *http.Request) []Proof {
    c, _ := req.Cookie(COOKIE_NAME_PROOF)
    if c == nil { return []Proof{} }
    if len(c.Value) > 500 { return []Proof{} }  // 防膨胀
    j, err := DecryptString(SECRET_KEY_DERIVATE_FOR_PROOF, c.Value)
    if err != nil { return []Proof{} }
    var p []Proof
    json.Unmarshal([]byte(j), &p)
    return p
}
```

**Proof 等价性判断**（判断已验证的 Proof 是否满足某一需求）：

**文件**：`server/model/share.go:341-354`

```go
func shareProofAreEquivalent(ref Proof, p Proof) bool {
    if ref.Key != p.Key { return false }
    // 密码：直接比对哈希值
    if ref.Value != "" && ref.Value == p.Value { return true }
    // 邮箱：通过 ID 比对，ID = Hash("email::" + 邮箱地址, 20)
    for _, chunk := range strings.Split(ref.Value, ",") {
        chunk = strings.Trim(chunk, " ")
        if p.Id == Hash(ref.Key+"::"+chunk, 20) {
            return true
        }
    }
    return false
}
```

### 3.5 WebDAV 网络驱动器认证

当通过 WebDAV 挂载时，使用 HTTP Basic Auth，用户名格式为：

```
email[hash]
```

其中 `hash = Hash(email + SECRET_KEY_DERIVATE_FOR_HASH, 10)`，防止伪造用户名。

**文件**：`server/middleware/session.go:222-242`、`server/model/share.go:654-656`

---

## 四、动作授予与权限判定

### 4.1 权限位定义

Share 结构体定义了 5 个布尔权限位：

**文件**：`server/common/types.go:180-194`

```go
type Share struct {
    Id           string  `json:"id"`
    Backend      string  `json:"-"`        // 创建者后端ID（不序列化到前端）
    Auth         string  `json:"auth,omitempty"`
    Path         string  `json:"path"`
    Password     *string `json:"password,omitempty"`
    Users        *string `json:"users,omitempty"`
    Expire       *int64  `json:"expire,omitempty"`
    CanShare     bool    `json:"can_share"`      // 能否再共享（创建子共享）
    CanManageOwn bool    `json:"can_manage_own"` // 能否管理自己创建的共享（预留）
    CanRead      bool    `json:"can_read"`       // 能否读取文件
    CanWrite     bool    `json:"can_write"`      // 能否编辑/覆盖/删除
    CanUpload    bool    `json:"can_upload"`     // 能否上传（不覆盖已有文件）
}
```

> **注意**：`CanManageOwn` 权限在数据模型中已定义，但当前代码中未实际使用，属于预留字段。

### 4.2 权限判定核心函数

**文件**：`server/model/permissions.go:1-33`

```go
func CanRead(ctx *App) bool {
    if ctx.Share.Id != "" { return ctx.Share.CanRead }
    return true  // 非共享上下文：默认允许
}

func CanEdit(ctx *App) bool {
    if ctx.Share.Id != "" { return ctx.Share.CanWrite }
    return true
}

func CanUpload(ctx *App) bool {
    if ctx.Share.Id != "" { return ctx.Share.CanUpload }
    return true
}

func CanShare(ctx *App) bool {
    if ctx.Share.Id != "" { return ctx.Share.CanShare }
    return true
}
```

**设计原则**：非共享上下文默认全部允许，共享上下文严格按权限位判定。

### 4.3 前端角色与权限映射

前端提供三种预设角色，对应不同权限组合：

**文件**：`public/assets/pages/filespage/modal_share.js:329-357`

| 角色 | CanRead | CanWrite | CanUpload | 说明 |
|-----|---------|----------|-----------|------|
| viewer | ✅ | ❌ | ❌ | 仅查看 |
| editor | ✅ | ✅ | ✅ | 完全编辑 |
| uploader | ❌ | ❌ | ✅ | 仅上传（看不到文件） |

### 4.4 各 API 对应的权限要求

| API 路由 | 方法 | 所需权限 | 备注 |
|---------|------|---------|------|
| `/api/files/ls` | GET | CanRead | 无 CanRead 但有 CanUpload 时返回空列表 |
| `/api/files/cat` | GET/HEAD | CanRead | 含文件读取、缩略图、下载 |
| `/api/files/zip` | GET | CanRead | ZIP 打包下载 |
| `/api/files/unzip` | POST | CanRead + CanUpload | 解压需创建目录和写入文件 |
| `/api/files/save` | POST/PATCH | CanEdit \|\| CanUpload | CanEdit=覆盖；仅 CanUpload 时禁止覆盖 |
| `/api/files/mv` | POST | CanEdit | 移动/重命名 |
| `/api/files/rm` | POST | CanEdit | 删除 |
| `/api/files/mkdir` | POST | CanUpload | 创建目录 |
| `/api/files/touch` | POST | CanUpload | 创建空文件 |
| `/api/files/search` | GET | CanRead | 搜索 |
| `/api/share` | GET | 需创建者身份 | 共享列表查询 |
| `/api/share/{id}` | POST | CanManageShare | 创建/更新共享 |
| `/api/share/{id}` | DELETE | CanManageShare | 删除共享 |
| `/api/share/{id}/proof` | POST | 公开 | 验证凭证 |

### 4.5 WebDAV 权限映射

WebDAV 接口（`/s/{share_id}` 路径）仅在共享上下文下可用，权限映射如下：

**文件**：`server/ctrl/webdav.go:13-52`

| WebDAV 方法 | 所需权限 | 说明 |
|------------|---------|------|
| OPTIONS / HEAD / GET | CanRead | 读取文件和属性 |
| PROPFIND | CanRead | 属性查询（目录列表） |
| MKCOL / DELETE / COPY / MOVE / PROPPATCH | CanWrite | 修改操作 |
| PUT | CanWrite && CanUpload | 文件上传 |
| LOCK / UNLOCK | CanWrite && CanUpload | 文件锁操作 |

```go
func WebdavHandler(ctx *App, res http.ResponseWriter, req *http.Request) {
    if ctx.Share.Id == "" {
        http.NotFound(res, req)  // 非共享上下文不可用
        return
    }
    canRead := model.CanRead(ctx)
    canWrite := model.CanEdit(ctx)
    canUpload := model.CanUpload(ctx)
    switch req.Method {
    case "OPTIONS", "HEAD", "GET":
        if canRead == false { /* 403 */ }
    case "MKCOL", "DELETE", "COPY", "MOVE", "PROPPATCH":
        if canWrite == false { /* 403 */ }
    case "PROPFIND":
        if canRead == false { /* 403 */ }
    case "PUT", "LOCK", "UNLOCK":
        if canWrite == false || canUpload == false { /* 403 */ }
    }
    // ... 调用 webdav Handler
}
```

### 4.6 公共站点处理器权限

当启用 `plg_handler_site` 插件时，`/public/{share}/` 路径提供静态站点访问：

**文件**：`server/plugin/plg_handler_site/index.go:35-83`

```go
func SiteHandler(app *App, w http.ResponseWriter, r *http.Request) {
    if app.Backend == nil {
        SendErrorResult(w, ErrNotFound)
        return
    }
    if model.CanRead(app) == false {
        SendErrorResult(w, ErrPermissionDenied)
        return
    }
    // ... 读取并返回文件
}
```

- 仅需 `CanRead` 权限
- 自动索引（autoindex）功能也需通过 `CanRead` 检查
- 目录访问时若存在 `index.html` 则自动返回该文件

### 4.7 文件列表中的细粒度权限探测

LS 接口不仅返回文件列表，还在 Metadata 中返回前端可用的操作权限：

**文件**：`server/ctrl/files.go:79-191`

```go
func FileLs(ctx *App, res http.ResponseWriter, req *http.Request) {
    // 1. 基础权限过滤
    if model.CanRead(ctx) == false {
        if model.CanUpload(ctx) == false {
            SendErrorResult(res, ErrPermissionDenied)
            return
        }
        SendSuccessResults(res, make([]FileInfo, 0))  // 纯上传：空列表
        return
    }

    // 2. 通过 Authorisation 插件接口逐个探测权限
    perms := Metadata{}
    for _, auth := range Hooks.Get.AuthorisationMiddleware() {
        if err = auth.Ls(ctx, path); err != nil { /* ... */ }
        if err = auth.Mkdir(ctx, path); err != nil { perms.CanCreateDirectory = NewBool(false) }
        if err = auth.Touch(ctx, path); err != nil { perms.CanCreateFile = NewBool(false) }
        if err = auth.Mv(ctx, path, path); err != nil { perms.CanRename = false; perms.CanMove = false }
        if err = auth.Save(ctx, path); err != nil { perms.CanUpload = NewBool(false) }
        if err = auth.Rm(ctx, path); err != nil { perms.CanDelete = NewBool(false) }
        if err = auth.Cat(ctx, path); err != nil { perms.CanSee = NewBool(false) }
    }

    // 3. 共享权限覆盖（优先级高于插件探测）
    if model.CanEdit(ctx) == false {
        perms.CanCreateFile = NewBool(false)
        perms.CanCreateDirectory = NewBool(false)
        perms.CanRename = NewBool(false)
        perms.CanMove = NewBool(false)
        perms.CanDelete = NewBool(false)
        perms.CanUpload = NewBool(false)
    }
    if model.CanUpload(ctx) == false {
        perms.CanCreateDirectory = NewBool(false)
        perms.CanRename = NewBool(false)
        perms.CanMove = NewBool(false)
        perms.CanDelete = NewBool(false)
        perms.CanUpload = NewBool(false)
    }
    if model.CanShare(ctx) == false {
        perms.CanShare = NewBool(false)
    }
    // ... 返回文件列表和权限元数据
}
```

### 4.8 CanEdit 与 CanUpload 的区别

在 `FileSave` 中体现了关键差异：

**文件**：`server/ctrl/files.go:492-514`

```go
if model.CanEdit(ctx) == false {
    if model.CanUpload(ctx) == false {
        SendErrorResult(res, ErrPermissionDenied)
        return
    }
    // 仅 CanUpload：禁止覆盖已存在文件
    root, filename := SplitPath(path)
    entries, _ := ctx.Backend.Ls(root)
    for _, e := range entries {
        if e.Name() == filename {
            SendErrorResult(res, ErrConflict)  // HTTP 409 Conflict
            return
        }
    }
}
```

| 权限 | 可新建 | 可覆盖 | 可删除 | 可修改 |
|-----|-------|-------|-------|-------|
| CanEdit | ✅ | ✅ | ✅ | ✅ |
| CanUpload | ✅ | ❌ | ❌ | ❌ |

### 4.9 共享管理权判定（CanManageShare）

`CanManageShare` 中间件控制谁能修改/删除共享链接：

**文件**：`server/middleware/session.go:96-158`

```go
func CanManageShare(fn HandlerFunc) HandlerFunc {
    return HandlerFunc(func(ctx *App, res http.ResponseWriter, req *http.Request) {
        share_id := mux.Vars(req)["share"]
        s, err := model.ShareGet(share_id)
        
        if err == ErrNotFound {
            // 情况1：ID 尚未使用，任何登录用户都可创建
            SessionStart(fn)(ctx, res, req)
            return
        }

        // 情况2：原始创建者（通过 Backend ID 匹配判断）
        ctx.Share = Share{}
        ctx.Session, _ = _extractSession(req, ctx)
        if s.Backend == GenerateID(ctx.Session) {
            fn(ctx, res, req)
            return
        }

        // 情况3：非创建者但父共享授予了 CanShare 权限
        ctx.Share, _ = _extractShare(req)  // 提取父共享上下文
        ctx.Session, _ = _extractSession(req, ctx)
        if s.Backend == GenerateID(ctx.Session) && s.CanShare == true {
            fn(ctx, res, req)
            return
        }

        SendErrorResult(res, ErrPermissionDenied)
    })
}
```

**三层管理权限逻辑**：
1. **新 ID** → 任何已登录用户可占用创建
2. **创建者本人** → 通过 `s.Backend == GenerateID(ctx.Session)` 判断（同一后端连接）
3. **被授权的子用户** → 需同时满足：通过父共享访问 + 父共享 `CanShare=true`

### 4.10 路径隔离（Chroot）

共享链接访问时，Session 的 path 被强制限定在共享目标范围内，防止路径逃逸：

**文件**：`server/middleware/session.go:269-291`

```go
if ctx.Share.Id != "" {
    str, _ = DecryptString(SECRET_KEY_DERIVATE_FOR_USER, ctx.Share.Auth)
    json.Unmarshal([]byte(str), &session)
    
    if IsDirectory(ctx.Share.Path) {
        // 目录共享：chroot 到该目录
        session["path"] = ctx.Share.Path
    } else {
        // 文件共享：仅允许访问该具体文件
        var path string = req.URL.Query().Get("path")
        if strings.HasSuffix(ctx.Share.Path, path) == false {
            return make(map[string]string), ErrPermissionDenied
        }
        session["path"] = strings.TrimSuffix(ctx.Share.Path, path) + "/"
    }
}
```

后续所有路径通过 `PathBuilder` 拼接时都会检查前缀：

**文件**：`server/ctrl/files.go:1104-1117`

```go
func PathBuilder(ctx *App, path string) (string, error) {
    sessionPath := ctx.Session["path"]
    basePath := filepath.ToSlash(filepath.Join(sessionPath, path))
    if path[len(path)-1:] == "/" && basePath != "/" {
        basePath += "/"
    }
    if strings.HasPrefix(basePath, ctx.Session["path"]) == false {
        return "", ErrFilesystemError  // 路径逃逸检测
    }
    return basePath, nil
}
```

---

## 五、过期回收机制

### 5.1 共享链接有效期校验

**文件**：`server/common/types.go:196-204`

```go
func (s Share) IsValid() error {
    if s.Expire != nil {
        now := time.Now().UnixNano() / 1000000  // 毫秒级 Unix 时间戳
        if now > *s.Expire {
            return NewError("Link has expired", 410)  // HTTP 410 Gone
        }
    }
    return nil
}
```

**校验时机**：
1. `_extractShare()` 提取共享时（`server/middleware/session.go:217`）
2. `ShareVerifyProof()` 验证凭证时（`server/ctrl/share.go:141`）

### 5.2 Proof Cookie 过期

| 属性 | 值 | 说明 |
|-----|----|------|
| **MaxAge** | 30 天 | 长期有效，无需频繁验证 |
| **HttpOnly** | true | 防止 XSS 窃取 |
| **SameSite** | None | 支持跨站嵌入 iframe |
| **Secure** | true | 仅 HTTPS 传输 |
| **大小限制** | 500 字节 | 超过则视为无效（防膨胀） |
| **数量限制** | 20 个 Proof | 超过则强制清空（防滥用） |

**文件**：`server/ctrl/share.go:180-193`、`server/model/share.go:300`、`server/ctrl/share.go:130`

### 5.3 Session 凭证过期

- 普通用户 Session Cookie 有独立过期时间（配置项控制）
- 创建者的 Session 被加密后存入数据库 `Share.auth` 字段，**无独立过期时间**
- 主密钥变更会导致所有现存共享的 `auth` 字段无法解密（`server/middleware/session.go:271-274` 返回 `ErrNotAuthorized`）

### 5.4 邮箱验证码自动清理

**文件**：`server/model/index.go:35-46`

```go
func init() {
    Hooks.Register.Onload(func() {
        // ... 初始化数据库
        go func() {
            autovacuum()  // 启动后台清理协程
        }()
    })
}

func autovacuum() {
    for {
        // 删除过期的验证码记录
        if stmt, err := DB.Prepare("DELETE FROM Verification WHERE expire < datetime('now')"); err == nil {
            stmt.Exec()
        }
        time.Sleep(6 * time.Hour)  // 每 6 小时执行一次
    }
}
```

此外，验证码在**成功使用后立即删除**（`server/model/share.go:252-255`），保证一次性使用。

### 5.5 显式撤销（删除共享）

**文件**：`server/ctrl/share.go:96-104`、`server/model/share.go:141-148`

```go
func ShareDelete(id string) error {
    stmt, _ := DB.Prepare("DELETE FROM Share WHERE id = ?")
    _, err := stmt.Exec(id)
    return err
}
```

删除后的影响：
- 后续 `_extractShare()` 查不到记录 → 返回空 Share → 无法建立共享上下文
- 用户已持有的 Proof Cookie **不会立即失效**，下次请求时因 Share 不存在而失败

### 5.6 数据库级联删除

Share 表通过外键关联到 Location 表，支持级联操作：

**文件**：`server/model/index.go:24`

```sql
FOREIGN KEY (related_backend, related_path) 
    REFERENCES Location(backend, path) 
    ON UPDATE CASCADE 
    ON DELETE CASCADE
```

当 Location（后端+路径组合）被删除时，相关的所有 Share 记录自动级联删除。

---

## 六、路由与中间件链

### 6.1 共享相关路由配置

**文件**：`server/routes.go:70-79`

```go
share := r.PathPrefix(WithBase("/api/share")).Subrouter()

// 共享列表：需登录
middlewares = []Middleware{ApiHeaders, SecureHeaders, SecureOrigin, SessionStart, LoggedInOnly, PluginInjector}
share.HandleFunc("", NewMiddlewareChain(ShareList, middlewares)).Methods("GET")

// 验证 Proof：公开接口
middlewares = []Middleware{ApiHeaders, SecureHeaders, SecureOrigin, BodyParser, PluginInjector}
share.HandleFunc("/{share}/proof", NewMiddlewareChain(ShareVerifyProof, middlewares)).Methods("POST")

// 删除共享：需 CanManageShare 权限
middlewares = []Middleware{ApiHeaders, SecureHeaders, SecureOrigin, CanManageShare, PluginInjector}
share.HandleFunc("/{share}", NewMiddlewareChain(ShareDelete, middlewares)).Methods("DELETE")

// 创建/更新共享：需 CanManageShare 权限 + BodyParser
middlewares = []Middleware{ApiHeaders, SecureHeaders, SecureOrigin, BodyParser, CanManageShare, PluginInjector}
share.HandleFunc("/{share}", NewMiddlewareChain(ShareUpsert, middlewares)).Methods("POST")
```

### 6.2 文件 API 中间件链

**文件**：`server/routes.go:53-68`

所有文件 API 都经过 `SessionStart` 中间件，会自动提取共享上下文并应用权限限制。

### 6.3 WebDAV 路由

WebDAV 接口专门用于共享链接的网络驱动器访问：

**文件**：`server/routes.go:81-87`

```go
// WebDAV 接口（仅共享链接可用）
r.PathPrefix(WithBase("/s/{share}")).Handler(NewMiddlewareChain(
    WebdavHandler,
    []Middleware{SessionStart, WebdavBlacklist, PluginInjector},
))
```

- 必须通过共享链接访问（`ctx.Share.Id` 非空）
- 经过 `WebdavBlacklist` 中间件过滤 macOS 系统文件（`.DS_Store`、`._*` 等）
- 内部按 WebDAV 方法分别校验 `CanRead`/`CanWrite`/`CanUpload` 权限

### 6.4 公共站点处理器路由

启用 `plg_handler_site` 插件后提供静态站点访问：

**文件**：`server/plugin/plg_handler_site/index.go:22-25`

```go
r.PathPrefix("/public/{share}/").HandlerFunc(NewMiddlewareChain(
    SiteHandler,
    []Middleware{SessionStart, SecureHeaders, cors},
)).Methods("GET", "HEAD")
```

- 仅支持 GET 和 HEAD 方法
- 需通过 `SessionStart` 提取共享上下文并验证 `CanRead` 权限
- 用于将共享目录作为静态网站发布

---

## 七、安全设计要点总结

| 安全机制 | 实现方式 | 代码位置 |
|---------|---------|---------|
| **凭证加密** | AES-256-GCM + Zlib + 分用途派生密钥 | `crypto.go`、`constants.go` |
| **密码存储** | bcrypt 慢哈希 + 验证失败延迟 1s | `share.go:92-93`、`share.go:160-161` |
| **路径隔离** | Session path 强制 chroot + PathBuilder 逃逸检测 | `session.go:276-290`、`files.go:1113` |
| **验证码** | 4 位随机数 + 10 分钟过期 + 一次性消费 | `share.go:183-186`、`share.go:252-255` |
| **Cookie 安全** | HttpOnly + SameSite=None + Secure + 大小/数量限制 | `share.go:188-192`、`share.go:300` |
| **嵌套共享** | 权限继承（父权限含 CanShare 才可再共享） | `session.go:131-154` |
| **上传保护** | 仅 CanUpload 时禁止覆盖已有文件（409 Conflict） | `files.go:498-513` |
| **主密钥安全** | Auth 字段加密绑定主密钥，密钥变更则全失效 | `session.go:270-274` |
| **数据库级联** | Location 删除级联 Share 清理 | `index.go:24` |
| **防暴力破解** | 密码/邮箱验证失败后 sleep 1 秒 | `share.go:160`、`share.go:173` |
