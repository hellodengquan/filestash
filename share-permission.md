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

核心代码分布与锚点：

| 模块 | 文件 | 关键行号 |
|-----|------|---------|
| 密钥派生 | `server/common/constants.go` | L64-79 |
| 加密/哈希/随机 | `server/common/crypto.go` | L1-266 |
| Share 结构体 | `server/common/types.go` | L164-204 |
| App 上下文 | `server/common/app.go` | L7-15 |
| 数据模型 | `server/model/share.go` | L1-656 |
| 数据库建表 | `server/model/index.go` | L12-46 |
| 权限判定 | `server/model/permissions.go` | L1-33 |
| 中间件链执行 | `server/middleware/index.go` | L21-37 |
| 会话提取 | `server/middleware/session.go` | L57-331 |
| HTTP 中间件 | `server/middleware/http.go` | L1-129 |
| Body 解析 | `server/middleware/context.go` | L1-35 |
| 共享控制器 | `server/ctrl/share.go` | L1-213 |
| 文件控制器 | `server/ctrl/files.go` | L79-1117 |
| 会话控制器 | `server/ctrl/session.go` | L1-537 |
| WebDAV 处理 | `server/ctrl/webdav.go` | L1-80 |
| WebDAV 文件系统 | `server/model/webdav.go` | L33-134 |
| 路由配置 | `server/routes.go` | L19-176 |
| 前端共享模态框 | `public/assets/pages/filespage/modal_share.js` | L1-370 |
| 站点处理器插件 | `server/plugin/plg_handler_site/index.go` | L1-110 |
| 站点中间件 | `server/plugin/plg_handler_site/middleware.go` | L1-48 |
| 站点配置 | `server/plugin/plg_handler_site/config.go` | L1-56 |

---

## 二、凭证签发机制

### 2.1 主密钥初始化与派生调用链

主密钥 `SECRET_KEY` 在两处初始化，随后通过 `InitSecretDerivate` 派生所有子密钥：

**调用链 1 — 配置加载时**：`server/common/config_state.go:44-46`

```
LoadConfig()
  └─ gjson.Get(configStr, "general.secret_key").String()  // 从 config.json 读取
  └─ InitSecretDerivate(secret)                            // 派生子密钥
```

**调用链 2 — 运行时配置变更时**：`server/common/config.go:251-259`

```
Configuration.Initialise()
  └─ if secret_key == "":
  │     key := RandomString(16)     // 生成 16 位随机密钥
  │     this.Get("general.secret_key").Set(key)
  └─ InitSecretDerivate(this.Get("general.secret_key").String())
```

**`InitSecretDerivate` 完整逻辑**：`server/common/constants.go:72-79`

```go
func InitSecretDerivate(secret string) {
    SECRET_KEY = secret
    SECRET_KEY_DERIVATE_FOR_PROOF     = Hash("PROOF_"+SECRET_KEY, len(SECRET_KEY))
    SECRET_KEY_DERIVATE_FOR_ADMIN     = Hash("ADMIN_"+SECRET_KEY, len(SECRET_KEY))
    SECRET_KEY_DERIVATE_FOR_USER      = Hash("USER_"+SECRET_KEY, len(SECRET_KEY))
    SECRET_KEY_DERIVATE_FOR_HASH      = Hash("HASH_"+SECRET_KEY, len(SECRET_KEY))
    SECRET_KEY_DERIVATE_FOR_SIGNATURE = Hash("SGN_"+SECRET_KEY, len(SECRET_KEY))
}
```

每个派生密钥通过 `Hash()` → `sha256.New()` → `hashSize()` → `ReversedBaseChange()` 生成，长度等于主密钥长度（`len(SECRET_KEY)`），字符集为 `[a-zA-Z0-9]`。

| 派生密钥 | 前缀 | 用途 | 加密对象 |
|---------|------|------|---------|
| `SECRET_KEY_DERIVATE_FOR_USER` | `USER_` | Session Cookie / Share.auth 字段 | 后端连接凭证 |
| `SECRET_KEY_DERIVATE_FOR_PROOF` | `PROOF_` | Proof Cookie | 已验证凭证列表 |
| `SECRET_KEY_DERIVATE_FOR_ADMIN` | `ADMIN_` | 管理员会话 | 管理员 Token |
| `SECRET_KEY_DERIVATE_FOR_HASH` | `HASH_` | WebDAV 用户名哈希 | 防伪造校验码 |
| `SECRET_KEY_DERIVATE_FOR_SIGNATURE` | `SGN_` | SSO 签名验证 | 属性签名 |

### 2.2 加密算法与签名机制

**完整加密调用链**：`server/common/crypto.go:27-37`

```
EncryptString(secret, plaintext)
  ├─ compress([]byte(plaintext))            // L166-172: zlib.NewWriter 压缩
  ├─ EncryptAESGCM([]byte(secret), compressed)  // L128-144
  │     ├─ aes.NewCipher(key)               // AES-256 (key=32字节)
  │     ├─ cipher.NewGCM(block)             // GCM 模式
  │     ├─ GCMNonce.Next()                  // 生成 12 字节 nonce
  │     └─ gcm.Seal(nonce, nonce, pt, nil)  // 输出 = nonce || ciphertext || tag
  └─ base64.URLEncoding.EncodeToString(ciphertext)
```

**解密调用链**：`server/common/crypto.go:39-53`

```
DecryptString(secret, ciphertext)
  ├─ base64.URLEncoding.DecodeString(ciphertext)
  ├─ DecryptAESGCM([]byte(secret), raw)
  │     ├─ aes.NewCipher(key)
  │     ├─ cipher.NewGCM(block)
  │     ├─ 分离 nonce 和密文
  │     └─ gcm.Open(nil, nonce, ct, nil)    // 认证+解密
  └─ decompress(plaintext)                  // zlib 解压
```

**GCM Nonce 生成器**：`server/common/crypto.go:240-265`

```go
type NonceGenerator struct {
    current []byte   // 当前 nonce 状态
    count   int      // nonce 尺寸（12）
    *sync.Mutex
}

// 初始化：使用 crypto/rand 生成首个 nonce
func NewNonceGenerator(size int) NonceGenerator {
    firstNonce := make([]byte, size)
    io.ReadFull(rand.Reader, firstNonce)  // crypto/rand 安全随机源
    return NonceGenerator{firstNonce, size, &sync.Mutex{}}
}

// 递增：大端序 +1，避免 nonce 重复
func (this *NonceGenerator) Next() []byte {
    this.Lock()
    for i := len(this.current) - 1; i >= 0; i-- {
        if this.current[i] < 255 {
            this.current[i] += 1
            break
        }
        this.current[i] = 0  // 进位
    }
    newNonce := this.current
    this.Unlock()
    return newNonce
}
```

> **注意**：Nonce 生成采用**首随机 + 递增**策略。首次由 `crypto/rand` 生成真随机种子，后续通过大端序递增保证唯一性。Mutex 保证并发安全。AES-GCM 自身提供认证标签（16 字节），因此系统未使用独立的 HMAC 签名函数（`sign()` / `verify()` 在 `crypto.go:184-190` 中为空壳占位）。

**Hash 函数调用链**：`server/common/crypto.go:55-104`

```
Hash(str, n)
  ├─ sha256.New()
  ├─ hasher.Write([]byte(str))
  ├─ hasher.Sum(nil)                         // 32 字节 SHA-256 摘要
  └─ hashSize(digest, n)                     // 截取前 n 个字符
        └─ ReversedBaseChange(Letters, b[i]) // 字节值 → 自定义进制字符串
```

**随机源对比**：

| 函数 | 随机源 | 用途 | 代码位置 |
|-----|-------|------|---------|
| `RandomString(n)` | `crypto/rand`（安全） | 验证码、主密钥生成 | `crypto.go:106-118` |
| `QuickString(n)` | `math/rand`（非安全） | 非安全场景 | `crypto.go:120-126` |
| `NewNonceGenerator(size)` | `crypto/rand`（安全） | AES-GCM nonce | `crypto.go:246-251` |
| `NonceGenerator.Next()` | 确定性递增 | AES-GCM nonce | `crypto.go:253-265` |

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
                    cookie, err := req.Cookie(CookieName(index))  // CookieName: "auth", "auth1", "auth2", ...
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
        Path: func() string {
            leftPath := "/"
            rightPath := strings.TrimPrefix(NewStringFromInterface(ctx.Body["path"]), "/")
            if ctx.Share.Id != "" {
                leftPath = ctx.Share.Path   // 子共享基于父共享路径
            } else if ctx.Session["path"] != "" {
                leftPath = EnforceDirectory(ctx.Session["path"])
            }
            return leftPath + rightPath
        }(),
        Password:     NewStringpFromInterface(ctx.Body["password"]),
        Users:        NewStringpFromInterface(ctx.Body["users"]),
        Expire:       NewInt64pFromInterface(ctx.Body["expire"]),
        Url:          NewStringpFromInterface(ctx.Body["url"]),
        CanManageOwn: NewBoolFromInterface(ctx.Body["can_manage_own"]),
        CanShare:     NewBoolFromInterface(ctx.Body["can_share"]),
        CanRead:      NewBoolFromInterface(ctx.Body["can_read"]),
        CanWrite:     NewBoolFromInterface(ctx.Body["can_write"]),
        CanUpload:    NewBoolFromInterface(ctx.Body["can_upload"]),
    }
    model.ShareUpsert(&s)
}
```

**签发关键点**：
1. **Auth 字段**：存储创建者的加密 Session，包含后端类型、用户名、密码、路径等完整连接信息
2. **Backend 字段**：通过 `GenerateID(ctx.Session)` 生成，用于标识共享归属
3. **嵌套共享**：通过共享链接再创建子共享时，直接复用父共享的 `auth` 凭证；路径拼接基于父共享的 `ctx.Share.Path`
4. **密码处理**：使用 bcrypt 哈希存储，`PASSWORD_DUMMY` 占位符用于前端回显

**密码哈希存储**：
**文件**：`server/model/share.go:86-95`

```go
func ShareUpsert(p *Share) error {
    if p.Password != nil {
        if *p.Password == PASSWORD_DUMMY {
            if s, err := ShareGet(p.Id); err == nil {
                p.Password = s.Password  // 保留原密码
            }
        } else {
            hashedPassword, _ := bcrypt.GenerateFromPassword([]byte(*p.Password), bcrypt.DefaultCost)
            p.Password = NewString(string(hashedPassword))
        }
    }
    // INSERT INTO Location + INSERT/UPDATE Share
}
```

### 2.4 Backend ID 生成算法

**文件**：`server/common/crypto.go:193-218`

```go
func GenerateID(params map[string]string) string {
    p := ""
    orderedKeys := make([]string, len(params))
    for key, _ := range params {
        orderedKeys = append(orderedKeys, key)
    }
    sort.Strings(orderedKeys)  // 字典序排列保证确定性

    for _, key := range orderedKeys {
        switch key {
        case "password", "path", "session", "timestamp":  // 排除敏感/易变字段
        default:
            if val := params[key]; val != "" {
                p += key + "=>" + params[key] + ", "
            }
        }
    }
    if p == "" { return "na" }
    p += "salt=>" + SECRET_KEY  // 混入主密钥防止伪造
    return Hash(p, 20)          // SHA-256 → 20字符
}
```

此 ID 用于：
1. 关联 Share 记录与创建者的后端（`related_backend` 字段）
2. 判断共享管理权归属（创建者校验：`s.Backend == GenerateID(ctx.Session)`）

### 2.5 数据库存储结构

**文件**：`server/model/index.go:20-33`

```sql
CREATE TABLE IF NOT EXISTS Location(
    backend VARCHAR(16), 
    path VARCHAR(512), 
    CONSTRAINT pk_location PRIMARY KEY(backend, path)
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
CREATE INDEX idx_verification ON Verification(code, expire);
```

---

## 三、身份验证与凭证验证

### 3.1 共享请求的验证链路

所有请求经过 `SessionStart` 中间件，其中 `_extractShare` 函数完成共享上下文提取与验证：

**文件**：`server/middleware/session.go:202-260`

```go
func _extractShare(req *http.Request) (Share, error) {
    share_id := _extractShareId(req)         // URL query "share" 或 path variable
    if share_id == "" { return Share{}, nil }
    
    if Config.Get("features.share.enable").Bool() == false {
        return Share{}, NewError("Feature isn't enabled", 405)
    }

    s, err := model.ShareGet(share_id)
    if err != nil { return Share{}, nil }     // 查不到→空 Share（非共享请求）
    
    if err = s.IsValid(); err != nil {        // 过期校验
        return Share{}, err
    }

    // Proof-of-knowledge 一次校验（Cookie 中的已验证凭证）
    var verifiedProof []model.Proof = model.ShareProofGetAlreadyVerified(req)
    
    // WebDAV Basic Auth 二次校验
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

    requiredProof := model.ShareProofGetRequired(s)
    remainingProof := model.ShareProofCalculateRemainings(requiredProof, verifiedProof)
    if len(remainingProof) != 0 {
        return Share{}, NewError("Unauthorized Shared space", 400)
    }
    return s, nil
}
```

**`_extractShareId` 的两个来源**：`server/middleware/session.go:190-200`

```go
func _extractShareId(req *http.Request) string {
    share := req.URL.Query().Get("share")   // ?share=xxx
    if share != "" { return share }
    m := mux.Vars(req)["share"]             // /api/share/{share} 或 /s/{share}
    if m == "private" { return "" }          // "private" 保留字：忽略
    return m
}
```

### 3.2 Proof 验证机制

Proof 系统支持两种验证方式，可**组合使用**（需同时满足）：

#### 方式一：密码验证

**文件**：`server/model/share.go:263-268`

```go
func ShareProofVerifierPassword(hashed string, given string) (string, bool) {
    if err := bcrypt.CompareHashAndPassword([]byte(hashed), []byte(given)); err != nil {
        return "", false
    }
    return hashed, true  // 返回哈希值作为 Proof Value
}
```

安全措施：
- 密码使用 **bcrypt** 算法存储（默认 cost=10）
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
2. **提交验证码** → 系统从 `Verification` 表查询（需未过期）→ 匹配后立即删除（一次性使用）

验证码生成调用链：`RandomString(4)` → `crypto/rand.Read` → `[a-zA-Z0-9]` 字符集

### 3.3 Proof 验证控制器

**文件**：`server/ctrl/share.go:106-213`

完整流程：

1. **初始化上下文**：从数据库加载 Share、获取已验证/需要验证的 Proof
2. **防滥用检查**：verifiedProof > 20 或 requiredProof > 20 → 强制清空 Proof Cookie（`MaxAge: -1`）
3. **验证链接有效性**：`s.IsValid()` 过期校验
4. **处理提交的 Proof**：`ShareProofVerifier()` 执行密码/邮箱/验证码验证
5. **邮箱验证码特殊处理**：发送后返回提示，前端需再次提交验证码
6. **去重并追加**：`submittedProof.Id = Hash(Key+"::"+Value, 20)`，检查是否已存在
7. **计算剩余 Proof**：`ShareProofCalculateRemainings()`
8. **持久化到 Cookie**：`EncryptString(SECRET_KEY_DERIVATE_FOR_PROOF, json.Marshal(verifiedProof))`
9. **返回结果**：剩余 → 返回下一个需要验证的 Proof；全部通过 → 返回权限信息

**验证通过后返回的权限信息**：

```go
SendSuccessResult(res, struct {
    Id        string `json:"id"`
    Path      string `json:"path"`
    CanRead   bool   `json:"can_read"`
    CanWrite  bool   `json:"can_write"`
    CanUpload bool   `json:"can_upload"`
}{s.Id, s.Path, s.CanRead, s.CanWrite, s.CanUpload})
```

> 注意：前端获取此返回值后，在后续所有请求中通过 `?share=xxx` 参数标识共享上下文。

### 3.4 Proof Cookie 持久化与等价性判断

**文件**：`server/model/share.go:291-309`

```go
func ShareProofGetAlreadyVerified(req *http.Request) []Proof {
    c, _ := req.Cookie(COOKIE_NAME_PROOF)  // Cookie name = "proof"
    if c == nil { return []Proof{} }
    if len(c.Value) > 500 { return []Proof{} }  // 防膨胀攻击
    j, err := DecryptString(SECRET_KEY_DERIVATE_FOR_PROOF, c.Value)
    if err != nil { return []Proof{} }
    var p []Proof
    json.Unmarshal([]byte(j), &p)
    return p
}
```

**Proof 等价性判断**：`server/model/share.go:341-354`

```go
func shareProofAreEquivalent(ref Proof, p Proof) bool {
    if ref.Key != p.Key { return false }
    if ref.Value != "" && ref.Value == p.Value { return true }  // 密码：哈希值直接比对
    for _, chunk := range strings.Split(ref.Value, ",") {       // 邮箱：ID 比对
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

**用户名解析**：`server/middleware/session.go:222-242`

```go
username, password := func(authHeader string) (string, string) {
    decoded, _ := base64.StdEncoding.DecodeString(
        strings.TrimPrefix(authHeader, "Basic "),
    )
    s := bytes.Split(decoded, []byte(":"))
    usr := regexp.MustCompile(`^(.*)\[([0-9a-zA-Z]+)\]$`).FindStringSubmatch(string(s[0]))
    if len(usr) != 3 { return "", p }
    if Hash(usr[1]+SECRET_KEY_DERIVATE_FOR_HASH, 10) != usr[2] {
        return "", p  // 哈希不匹配→拒绝
    }
    return usr[1], p  // 返回真实邮箱
}(req.Header.Get("Authorization"))
```

**用户名编码**：`server/model/share.go:654-656`

```go
func networkDriveUsernameEnc(email string) string {
    return email + "[" + Hash(email+SECRET_KEY_DERIVATE_FOR_HASH, 10) + "]"
}
```

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
    Url          *string `json:"url,omitempty"`
    CanShare     bool    `json:"can_share"`      // 能否再共享（创建子共享）
    CanManageOwn bool    `json:"can_manage_own"` // 能否管理自己创建的共享（预留）
    CanRead      bool    `json:"can_read"`       // 能否读取文件
    CanWrite     bool    `json:"can_write"`      // 能否编辑/覆盖/删除
    CanUpload    bool    `json:"can_upload"`     // 能否上传（不覆盖已有文件）
}
```

> **注意**：`CanManageOwn` 权限在数据模型中已定义，但当前代码中未实际使用，属于预留字段。

前端 Metadata 结构体（返回给前端的权限元数据）：

**文件**：`server/common/types.go:164-176`

```go
type Metadata struct {
    CanSee             *bool      `json:"can_read,omitempty"`
    CanCreateFile      *bool      `json:"can_create_file,omitempty"`
    CanCreateDirectory *bool      `json:"can_create_directory,omitempty"`
    CanRename          *bool      `json:"can_rename,omitempty"`
    CanMove            *bool      `json:"can_move,omitempty"`
    CanUpload          *bool      `json:"can_upload,omitempty"`
    CanDelete          *bool      `json:"can_delete,omitempty"`
    CanShare           *bool      `json:"can_share,omitempty"`
    HideExtension      *bool      `json:"hide_extension,omitempty"`
    RefreshOnCreate    *bool      `json:"refresh_on_create,omitempty"`
    Expire             *time.Time `json:"-"`
}
```

### 4.2 权限判定核心函数

**文件**：`server/model/permissions.go:1-33`

```go
func CanRead(ctx *App) bool {
    if ctx.Share.Id != "" { return ctx.Share.CanRead }
    return true
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

**设计原则**：非共享上下文默认全部允许，共享上下文严格按权限位判定。判定依据是 `ctx.Share.Id` 是否非空——由 `SessionStart` 中间件的 `_extractShare()` 设置。

### 4.3 前端角色与权限映射

前端提供三种预设角色，对应不同权限组合：

**文件**：`public/assets/pages/filespage/modal_share.js:329-357`

| 角色 | CanRead | CanWrite | CanUpload | 说明 |
|-----|---------|----------|-----------|------|
| viewer | ✅ | ❌ | ❌ | 仅查看 |
| editor | ✅ | ✅ | ✅ | 完全编辑 |
| uploader | ❌ | ❌ | ✅ | 仅上传（看不到文件） |

**角色→权限映射**：`modal_share.js:329-346`

```javascript
function roleToShareObj(role) {
    return {
        can_read:  role === "viewer" || role === "editor",
        can_write: role === "editor",
        can_upload: role === "uploader" || role === "editor",
    };
}
```

**权限→角色反查**：`modal_share.js:348-357`

```javascript
function shareObjToRole({ can_read, can_write, can_upload }) {
    if (can_read && !can_write && !can_upload) return "viewer";
    if (!can_read && !can_write && can_upload) return "uploader";
    if (can_read && can_write && can_upload) return "editor";
    return undefined;  // 自定义权限组合无法映射到预设角色
}
```

### 4.4 各 API 对应的权限要求

| API 路由 | HTTP 方法 | 所需权限 | 代码锚点 |
|---------|----------|---------|---------|
| `/api/files/ls` | GET | CanRead（无则看 CanUpload） | `files.go:80-88` |
| `/api/files/cat` | GET/HEAD | CanRead | `files.go:208-212` |
| `/api/files/zip` | GET | CanRead | `files.go` 下载器 |
| `/api/files/unzip` | POST | CanRead + CanUpload | `files.go` 解压 |
| `/api/files/save` | POST/PATCH | CanEdit ∥ CanUpload | `files.go:492-514` |
| `/api/files/access` | OPTIONS | CanRead→GET, CanEdit→PUT, CanUpload→POST | `files.go:456-471` |
| `/api/files/mv` | POST | CanEdit | `files.go:737` |
| `/api/files/rm` | POST | CanEdit | `files.go:779` |
| `/api/files/mkdir` | POST | CanUpload | `files.go:810` |
| `/api/files/touch` | POST | CanUpload | `files.go:841` |
| `/api/files/search` | GET | CanRead | `search.go:16` |
| `/api/share` | GET | LoggedInOnly | `share.go:13` |
| `/api/share/{id}` | POST | CanManageShare | `share.go:35` |
| `/api/share/{id}` | DELETE | CanManageShare | `share.go:96` |
| `/api/share/{id}/proof` | POST | 公开 | `share.go:106` |

**`FileAccess` 权限探测函数**：`server/ctrl/files.go:443-475`

此函数根据权限位返回允许的 HTTP 方法列表（`Allow` 头），供前端预检：

```go
func FileAccess(ctx *App, res http.ResponseWriter, req *http.Request) {
    allowed := []string{}
    if model.CanRead(ctx) {
        if perms.CanSee == nil || *perms.CanSee == true {
            allowed = append(allowed, "GET")
        }
    }
    if model.CanEdit(ctx) {
        if (perms.CanCreateFile == nil || *perms.CanCreateFile == true) &&
            (perms.CanCreateDirectory == nil || *perms.CanCreateDirectory == true) {
            allowed = append(allowed, "PUT")
        }
    }
    if model.CanUpload(ctx) {
        if perms.CanUpload == nil || *perms.CanUpload == true {
            allowed = append(allowed, "POST")
        }
    }
    header.Set("Allow", strings.Join(allowed, ", "))
}
```

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

WebDAV 层面的路径隔离由 `WebdavFs.fullpath()` 实现：

**文件**：`server/model/webdav.go:125-134`

```go
func (this WebdavFs) fullpath(path string) string {
    p := filepath.Join(this.chroot, path)
    if strings.HasSuffix(path, "/") && !strings.HasSuffix(p, "/") {
        p += "/"
    }
    if strings.HasPrefix(p, this.chroot) == false {
        return ""  // 路径逃逸→返回空→os.ErrNotExist
    }
    return p
}
```

`chroot` 值来自 `ctx.Share.Path`，在 `WebdavHandler` 中传入 `NewWebdavFs(ctx.Backend, ctx.Share.Backend, ctx.Share.Path, req)`。

### 4.6 公共站点处理器权限

启用 `plg_handler_site` 插件时（`features.site.enable=true`），`/public/{share}/` 提供静态站点访问：

**文件**：`server/plugin/plg_handler_site/index.go:35-83`

- 仅需 `CanRead` 权限
- 自动索引（autoindex）：`features.site.autoindex=true` 时，目录无 `index.html` 则列出文件
- CORS：由 `features.site.cors_allow_origins` 控制，支持 `*` 或逗号分隔的 origin 列表
- 目录访问时若存在 `index.html` 则自动返回

站点列表页 `/public/` 需要 **管理员 Basic Auth**：

**文件**：`server/plugin/plg_handler_site/middleware.go:34-48`

```go
func basicAdmin(fn HandlerFunc) HandlerFunc {
    return HandlerFunc(func(ctx *App, w http.ResponseWriter, r *http.Request) {
        user, pass, ok := r.BasicAuth()
        if !ok || user != "admin" {
            w.Header().Set("WWW-Authenticate", `Basic realm="Restricted"`)
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        if err := bcrypt.CompareHashAndPassword(
            []byte(Config.Get("auth.admin").String()), []byte(pass),
        ); err != nil {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        fn(ctx, w, r)
    })
}
```

### 4.7 文件列表中的细粒度权限探测

LS 接口不仅返回文件列表，还在 Metadata 中返回前端可用的操作权限。权限计算采用**三层叠加**模型：

**文件**：`server/ctrl/files.go:79-144`

1. **后端插件探测**（Backend Auth Middleware）：逐个尝试操作，失败则标记不可用
2. **共享权限覆盖**（优先级高于插件）：根据 `CanEdit`/`CanUpload`/`CanShare` 批量禁用
3. **前端 Metadata 输出**：`*bool` 类型，`nil`=允许，`false`=禁止

**共享权限对 Metadata 的覆盖规则**：

| 共享权限位 | 禁用的 Metadata 字段 |
|-----------|---------------------|
| `CanEdit=false` | CanCreateFile, CanCreateDirectory, CanRename, CanMove, CanDelete, CanUpload |
| `CanUpload=false` | CanCreateDirectory, CanRename, CanMove, CanDelete, CanUpload |
| `CanShare=false` | CanShare |

> **继承规则**：共享权限是**目录级**的，作用于 `Share.Path` 及其所有子路径。没有按文件的细粒度权限——子目录/子文件继承父共享的全部权限位。子共享可以进一步限制（不可放宽），通过 `ShareUpsert` 中 `Path = parentShare.Path + rightPath` 拼接更深层路径实现。

### 4.8 CanEdit 与 CanUpload 的区别

**文件**：`server/ctrl/files.go:492-514`

| 权限 | 可新建 | 可覆盖 | 可删除 | 可修改 |
|-----|-------|-------|-------|-------|
| CanEdit | ✅ | ✅ | ✅ | ✅ |
| CanUpload | ✅ | ❌ | ❌ | ❌ |

仅 CanUpload 时，`FileSave` 会先 `Ls` 目标目录检查同名文件，存在则返回 HTTP 409 Conflict。

### 4.9 共享管理权判定（CanManageShare）

**文件**：`server/middleware/session.go:96-158`

**三层管理权限逻辑**：
1. **新 ID**（`ErrNotFound`）→ 任何已登录用户可占用创建
2. **创建者本人** → 通过 `s.Backend == GenerateID(ctx.Session)` 判断（同一后端连接）
3. **被授权的子用户** → 需同时满足：通过父共享访问 + 父共享 `CanShare=true`

**嵌套共享的权限继承**：

子共享创建时（`ctx.Share.Id != ""`），继承父共享的 `Auth` 和 `Backend`，路径基于父共享的 `ctx.Share.Path` 拼接。这意味着：
- 子共享的访问者使用父共享创建者的后端凭证
- 子共享权限**可以比父共享更严格**（例如父共享是 editor，子共享可以是 viewer）
- 子共享**不能超越**父共享权限（前端不阻止，但后端权限函数按各自 Share 记录独立判定）

### 4.10 路径隔离（Chroot）

共享链接访问时，Session 的 path 被强制限定在共享目标范围内：

**文件**：`server/middleware/session.go:269-291`

```go
if ctx.Share.Id != "" {
    str, _ = DecryptString(SECRET_KEY_DERIVATE_FOR_USER, ctx.Share.Auth)
    json.Unmarshal([]byte(str), &session)
    
    if IsDirectory(ctx.Share.Path) {
        session["path"] = ctx.Share.Path    // 目录共享：chroot 到该目录
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

`PathBuilder` 二次校验：`server/ctrl/files.go:1104-1117`

```go
func PathBuilder(ctx *App, path string) (string, error) {
    sessionPath := ctx.Session["path"]
    basePath := filepath.ToSlash(filepath.Join(sessionPath, path))
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

**校验时机**（两次 proof-of-knowledge 二次校验）：
1. `_extractShare()` 提取共享时（`server/middleware/session.go:217`）— 每次请求
2. `ShareVerifyProof()` 验证凭证时（`server/ctrl/share.go:141`）— 提交 Proof 时

这意味着即使 Proof Cookie 仍在有效期内，过期链接在步骤 1 即被拦截。

### 5.2 访客 Cookie 处理

共享链接访客涉及两类 Cookie：

**Proof Cookie**（已验证凭证）：

| 属性 | 值 | 代码位置 |
|-----|----|---------|
| Name | `"proof"` | `constants.go:13` |
| MaxAge | 30 天（2592000 秒） | `share.go:188` |
| HttpOnly | true | `share.go:190` |
| SameSite | None | `share.go:191` |
| Secure | true | `share.go:192` |
| Path | `/api/` | `constants.go:16` |
| 大小限制 | 500 字节（超过视为无效） | `share.go:300` |
| 数量限制 | 20 个 Proof（超过强制清空） | `share.go:130-137` |
| 加密密钥 | `SECRET_KEY_DERIVATE_FOR_PROOF` | `share.go:184` |

**Session Cookie**（非共享访客不持有）：

共享访客的"会话"不在 Cookie 中，而是通过 `?share=xxx` URL 参数 + Proof Cookie 组合标识。后端从 `Share.auth` 字段解密出创建者的 Session 来建立后端连接。

**登录清除 Proof Cookie**：`server/ctrl/session.go:167-178`

```go
func SessionLogout(ctx *App, res http.ResponseWriter, req *http.Request) {
    // 清除所有 auth Cookie
    index := 0
    for {
        _, err := req.Cookie(CookieName(index))
        if err != nil { break }
        http.SetCookie(res, &http.Cookie{
            Name: CookieName(index), Value: "", MaxAge: -1, Path: COOKIE_PATH,
        })
        index++
    }
    // 清除 admin Cookie
    http.SetCookie(res, &http.Cookie{
        Name: COOKIE_NAME_ADMIN, Value: "", MaxAge: -1, Path: COOKIE_PATH_ADMIN,
    })
    // 清除 Proof Cookie
    http.SetCookie(res, &http.Cookie{
        Name: COOKIE_NAME_PROOF, Value: "", MaxAge: -1, Path: COOKIE_PATH,
    })
}
```

**FileCat 中的下载标记 Cookie**：`server/ctrl/files.go:202-207`

```go
func FileCat(ctx *App, res http.ResponseWriter, req *http.Request) {
    http.SetCookie(res, &http.Cookie{
        Name:   "download",
        Value:  "",
        MaxAge: -1,    // 立即过期
        Path:   "/",
    })
    // ... 文件读取逻辑
}
```

### 5.3 Proof-of-knowledge 二次校验

共享链接的访问验证分两层：

**第一层（Cookie 内校验）**：`_extractShare()` 中的 `ShareProofGetAlreadyVerified()`

- 从 `proof` Cookie 解密出已验证的 Proof 列表
- 与 `ShareProofGetRequired()` 对比，计算剩余未验证项
- 全部通过 → 放行；否则 → 400 错误

**第二层（Basic Auth 旁路校验）**：`_extractShare()` 中的 `parseBasicAuth()`

- 适用于 WebDAV 场景（无浏览器 Cookie）
- 从 HTTP Authorization 头解析 username/password
- 分别调用 `ShareProofVerifierEmail()` 和 `ShareProofVerifierPassword()` 即时验证
- 验证通过的 Proof 合并到 `verifiedProof` 中

这种**双通道校验**设计确保：
- 浏览器用户：Proof Cookie 持久化，无需重复验证
- WebDAV 用户：每次请求通过 Basic Auth 重新验证

### 5.4 Session 凭证过期

- 普通用户 Session Cookie 过期时间由 `general.cookie_timeout` 配置控制（`server/ctrl/session.go:115`）
- Session 本身包含 `timestamp` 字段，`_extractSession` 校验不超过 365 天（`session.go:310`）
- 创建者的 Session 被加密后存入数据库 `Share.auth` 字段，**无独立过期时间**
- 主密钥变更会导致所有现存共享的 `auth` 字段无法解密（`session.go:270-274` 返回 `ErrNotAuthorized`）

### 5.5 邮箱验证码定时清理

**文件**：`server/model/index.go:35-46`

```go
func init() {
    Hooks.Register.Onload(func() {
        // ... 初始化数据库表
        go func() {
            autovacuum()
        }()
    })
}

func autovacuum() {
    if stmt, err := DB.Prepare("DELETE FROM Verification WHERE expire < datetime('now')"); err == nil {
        stmt.Exec()
    }
    time.Sleep(6 * time.Hour)  // 每 6 小时执行一次
}
```

> **注意**：原代码中 `autovacuum` 没有循环（仅执行一次+sleep），这是原始实现的缺陷。正确行为应为 `for { ...; time.Sleep(...) }`。

此外，验证码在**成功使用后立即删除**（`server/model/share.go:252-255`），保证一次性使用。

**Verification 表索引**：`server/model/index.go:30-32`

```sql
CREATE INDEX idx_verification ON Verification(code, expire)
```

此索引优化了验证码查询：`SELECT key FROM Verification WHERE code = ? AND expire > datetime('now')`

### 5.6 显式撤销（删除共享）

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
- 用户已持有的 Proof Cookie **不会立即失效**，下次请求时因 Share 不存在而自然失败
- WebDAV 缓存文件会在 `webdav_cache.OnEvict` 回调中清理

### 5.7 数据库级联删除

```sql
FOREIGN KEY (related_backend, related_path) 
    REFERENCES Location(backend, path) 
    ON UPDATE CASCADE ON DELETE CASCADE
```

当 Location（后端+路径组合）被删除时，相关的所有 Share 记录自动级联删除。

---

## 六、路由与中间件链

### 6.1 中间件链执行机制

**文件**：`server/middleware/index.go:21-37`

```go
func NewMiddlewareChain(fn HandlerFunc, m []Middleware) http.HandlerFunc {
    return func(res http.ResponseWriter, req *http.Request) {
        var f func(*App, http.ResponseWriter, *http.Request) = fn
        for i := len(m) - 1; i >= 0; i-- {  // 逆序包装
            f = m[i](f)
        }
        app := App{Context: req.Context()}  // 初始化空 App
        f(&app, &resw, req)
        req.Body.Close()
        go logger(&app, &resw, req)  // 异步日志
    }
}
```

执行顺序：**从右到左包装，从左到右执行**。例如 `[A, B, C]` → 请求流 `A→B→C→Handler→C→B→A`。

### 6.2 各中间件功能

**文件**：`server/middleware/http.go`

| 中间件 | 功能 | 代码位置 |
|-------|------|---------|
| `ApiHeaders` | Content-Type: json, Cache-Control: no-cache, 透传 X-Request-ID | L14-24 |
| `StaticHeaders` | Content-Type 按扩展名, Cache-Control: 30 天 | L26-33 |
| `IndexHeaders` | Content-Type: html, XSS 保护, X-Frame-Options, X-Powered-By | L49-65 |
| `SecureHeaders` | HSTS(强制SSL时), X-Content-Type-Options, X-XSS-Protection | L67-77 |
| `SecureOrigin` | Host 白名单校验, XHR/CSRF 检查（`X-Requested-With: XmlHttpRequest`）| L79-105 |
| `RateLimiter` | 全局令牌桶：10 req/s, burst=1000 | L107-121 |
| `PublicCORS` | Access-Control-Allow-Origin: *, OPTIONS 预检 | L35-47 |
| `BodyParser` | JSON body 解析到 `ctx.Body` | `context.go:11-35` |
| `SessionStart` | 提取 Share + Authorization + Session + Backend | `session.go:57-83` |
| `LoggedInOnly` | 检查 `ctx.Backend != nil && ctx.Session != nil` | `session.go:18-26` |
| `CanManageShare` | 共享管理权三层判定 | `session.go:96-158` |
| `PluginInjector` | 注入插件中间件链 | `index.go:72-76` |
| `WebdavBlacklist` | 过滤 macOS 系统文件 (.DS_Store, ._*等) | `webdav.go:67-80` |

### 6.3 各路径中间件链对比

| 路径 | 中间件链 | 是否限流 | 是否需登录 | 是否 CSRF |
|------|---------|---------|-----------|----------|
| `/api/share` GET | ApiHeaders→SecureHeaders→SecureOrigin→**SessionStart**→**LoggedInOnly**→PluginInjector | ❌ | ✅ | ✅ SecureOrigin |
| `/api/share/{id}` POST | ApiHeaders→SecureHeaders→SecureOrigin→BodyParser→**CanManageShare**→PluginInjector | ❌ | ✅(隐式) | ✅ |
| `/api/share/{id}` DELETE | ApiHeaders→SecureHeaders→SecureOrigin→**CanManageShare**→PluginInjector | ❌ | ✅(隐式) | ✅ |
| `/api/share/{id}/proof` POST | ApiHeaders→SecureHeaders→SecureOrigin→BodyParser→PluginInjector | ❌ | ❌ 公开 | ✅ |
| `/api/session` POST | ApiHeaders→SecureHeaders→SecureOrigin→**RateLimiter**→BodyParser→PluginInjector | ✅ | ❌ | ✅ |
| `/api/files/*` | ApiHeaders→SecureHeaders→[SecureOrigin→]SessionStart→LoggedInOnly→PluginInjector | ❌ | ✅ | ✅ |
| `/s/{share}` (WebDAV GET) | IndexHeaders→SecureHeaders→PluginInjector | ❌ | ❌ | ❌ |
| `/s/{share}` (WebDAV 其他) | **WebdavBlacklist**→**SessionStart**→PluginInjector | ❌ | ❌ | ❌ |
| `/public/{share}/` | **SessionStart**→SecureHeaders→cors | ❌ | ❌ | ❌ |
| `/public/` 列表 | SecureHeaders→**basicAdmin** | ❌ | ✅(Admin) | ❌ |

**关键差异**：

1. **Proof 接口无 SessionStart**：不提取 Session/Share，完全公开
2. **WebDAV 无 SecureOrigin**：WebDAV 客户端不发送 `X-Requested-With`，CSRF 防护由 Basic Auth 替代
3. **公共站点无 SecureOrigin**：允许跨域访问（有独立 CORS 中间件）
4. **限流仅在登录接口**：`RateLimiter` 仅用于 `/api/session` POST（登录）和管理员登录
5. **匿名访问限流**：共享 Proof 接口和公共站点均**无限流**，依赖 bcrypt 延迟和 Proof 数量限制防滥用

### 6.4 SecureOrigin 的 CSRF 防护逻辑

**文件**：`server/middleware/http.go:79-105`

```go
func SecureOrigin(fn HandlerFunc) HandlerFunc {
    return HandlerFunc(func(ctx *App, res http.ResponseWriter, req *http.Request) {
        // 1. Host 白名单
        if host := Config.Get("general.host").String(); host != "" {
            if req.Host != host { SendErrorResult(res, ErrNotAllowed); return }
        }
        // 2. CSRF 检查（三选一通过）
        if req.Header.Get("X-Requested-With") == "XmlHttpRequest" {  // 浏览器 XHR
            fn(ctx, res, req); return
        }
        if Config.Get("features.api.enable").Bool() && len(req.Cookies()) == 0 {  // API 模式
            fn(ctx, res, req); return
        }
        Log.Warning("Intrusion detection: %s - %s", RetrievePublicIp(req), req.URL.String())
        SendErrorResult(res, ErrNotAllowed)
    })
}
```

**WebDAV/公共站点无 CSRF 检查**的原因：这些接口不经过 `SecureOrigin`，而是依赖 Proof Cookie 加密 + Basic Auth 保证安全。

### 6.5 共享相关路由配置

**文件**：`server/routes.go:70-91`

```go
// API for Shared link
share := r.PathPrefix(WithBase("/api/share")).Subrouter()
middlewares = []Middleware{ApiHeaders, SecureHeaders, SecureOrigin, SessionStart, LoggedInOnly, PluginInjector}
share.HandleFunc("", NewMiddlewareChain(ShareList, middlewares)).Methods("GET")

middlewares = []Middleware{ApiHeaders, SecureHeaders, SecureOrigin, BodyParser, PluginInjector}
share.HandleFunc("/{share}/proof", NewMiddlewareChain(ShareVerifyProof, middlewares)).Methods("POST")

middlewares = []Middleware{ApiHeaders, SecureHeaders, SecureOrigin, CanManageShare, PluginInjector}
share.HandleFunc("/{share}", NewMiddlewareChain(ShareDelete, middlewares)).Methods("DELETE")

middlewares = []Middleware{ApiHeaders, SecureHeaders, SecureOrigin, BodyParser, CanManageShare, PluginInjector}
share.HandleFunc("/{share}", NewMiddlewareChain(ShareUpsert, middlewares)).Methods("POST")

// Webdav server / Shared Link
middlewares = []Middleware{IndexHeaders, SecureHeaders, PluginInjector}
r.HandleFunc(WithBase("/s/{share}"), NewMiddlewareChain(ServeFrontofficeHandler, middlewares)).Methods("GET")

middlewares = []Middleware{WebdavBlacklist, SessionStart, PluginInjector}
r.PathPrefix(WithBase("/s/{share}")).Handler(NewMiddlewareChain(WebdavHandler, middlewares))
```

---

## 七、时序图

### 7.1 共享链接创建时序

```
创建者浏览器          前端 modal_share.js        API Server              SQLite DB
    │                      │                        │                       │
    │  选择角色(viewer/    │                        │                       │
    │  editor/uploader)    │                        │                       │
    │─────────────────────>│                        │                       │
    │                      │  roleToShareObj()       │                       │
    │                      │  → {can_read,can_write, │                       │
    │                      │     can_upload}         │                       │
    │                      │                        │                       │
    │  填写高级选项        │                        │                       │
    │  (密码/邮箱/过期等)   │                        │                       │
    │─────────────────────>│                        │                       │
    │                      │  POST /api/share/{id}   │                       │
    │                      │  Body: {id, path,       │                       │
    │                      │   can_read, can_write,  │                       │
    │                      │   can_upload, can_share,│                       │
    │                      │   password?, users?,    │                       │
    │                      │   expire?}              │                       │
    │                      │───────────────────────>│                       │
    │                      │                        │  CanManageShare 中间件 │
    │                      │                        │  ├─ 若新ID: SessionStart│
    │                      │                        │  └─ 若已有: 创建者校验  │
    │                      │                        │                       │
    │                      │                        │  ShareUpsert()         │
    │                      │                        │  ├─ bcrypt 哈希密码    │
    │                      │                        │  ├─ 拼接 Auth=Cookie拼接│
    │                      │                        │  ├─ Backend=GenerateID │
    │                      │                        │  ├─ Path=基础路径+子路径│
    │                      │                        │  └─ UPSERT Share       │
    │                      │                        │──────────────────────>│
    │                      │                        │                       │
    │                      │    200 OK               │                       │
    │                      │<───────────────────────│                       │
    │  复制链接到剪贴板     │                        │                       │
    │<─────────────────────│                        │                       │
```

### 7.2 访客验证与访问时序

```
访客浏览器             API Server               SQLite DB           邮件服务器
    │                      │                        │                   │
    │  GET /s/{share_id}   │                        │                   │
    │─────────────────────>│                        │                   │
    │                      │  SessionStart 中间件    │                   │
    │                      │  ├─ _extractShareId()   │                   │
    │                      │  ├─ ShareGet(id)        │                   │
    │                      │  │─────────────────────>│                   │
    │                      │  │<────Share record────│                   │
    │                      │  ├─ IsValid() 过期校验   │                   │
    │                      │  ├─ ShareProofGetAlreadyVerified()          │
    │                      │  │  (从 proof Cookie)   │                   │
    │                      │  └─ 剩余Proof!=0 → 400 │                   │
    │                      │                        │                   │
    │  302 重定向到登录页    │                        │                   │
    │<─────────────────────│                        │                   │
    │                      │                        │                   │
    │  POST /api/share/{id}/proof                   │                   │
    │  {type:"password", value:"xxx"}                │                   │
    │─────────────────────>│                        │                   │
    │                      │  ShareVerifyProof()     │                   │
    │                      │  ├─ ShareGet(id)        │                   │
    │                      │  ├─ IsValid() 过期校验   │                   │
    │                      │  ├─ ShareProofVerifier() │                   │
    │                      │  │  └─ bcrypt.Compare()  │                   │
    │                      │  ├─ 追加到 verifiedProof │                   │
    │                      │  ├─ 计算剩余 Proof      │                   │
    │                      │  └─ 设置 proof Cookie    │                   │
    │                      │     (EncryptString)     │                   │
    │                      │                        │                   │
    │  若还需邮箱验证:       │                        │                   │
    │  {type:"email", value:"user@co.com"}           │                   │
    │─────────────────────>│                        │                   │
    │                      │  ShareProofVerifier()   │                   │
    │                      │  ├─ 匹配邮箱白名单      │                   │
    │                      │  ├─ 生成4位验证码        │                   │
    │                      │  ├─ INSERT Verification │                   │
    │                      │  │─────────────────────>│                   │
    │                      │  └─ 发送验证邮件         │                   │
    │                      │────────────────────────────────────────────>│
    │                      │                        │                   │
    │  {key:"code", value:"A3B2"}                    │                   │
    │─────────────────────>│                        │                   │
    │                      │  ShareProofVerifier()   │                   │
    │                      │  ├─ SELECT key FROM     │                   │
    │                      │  │  Verification WHERE  │                   │
    │                      │  │  code=? AND expire>now│                   │
    │                      │  │─────────────────────>│                   │
    │                      │  ├─ DELETE Verification │  (一次性消费)      │
    │                      │  │─────────────────────>│                   │
    │                      │  ├─ 追加 email Proof    │                   │
    │                      │  └─ 全部通过 → 返回权限  │                   │
    │                      │     {id, path, can_read,│                   │
    │                      │      can_write, can_upload}                  │
    │<─────────────────────│                        │                   │
    │                      │                        │                   │
    │  GET /api/files/ls?path=/&share={id}          │                   │
    │  (携带 proof Cookie)  │                        │                   │
    │─────────────────────>│                        │                   │
    │                      │  SessionStart 中间件    │                   │
    │                      │  ├─ _extractShare()     │                   │
    │                      │  │  ├─ ShareGet()       │                   │
    │                      │  │  ├─ IsValid()        │                   │
    │                      │  │  ├─ Proof Cookie→verifiedProof          │
    │                      │  │  └─ 剩余Proof==0 ✓   │                   │
    │                      │  ├─ _extractSession()   │                   │
    │                      │  │  └─ Decrypt auth→session│                │
    │                      │  └─ _extractBackend()   │                   │
    │                      │                        │                   │
    │                      │  FileLs()               │                   │
    │                      │  ├─ CanRead(ctx) → true │                   │
    │                      │  ├─ 权限探测+覆盖        │                   │
    │                      │  └─ 返回文件列表+Metadata│                   │
    │<─────────────────────│                        │                   │
```

### 7.3 嵌套共享创建时序

```
子用户(通过父共享访问)    API Server               SQLite DB
    │                      │                        │
    │  POST /api/share/{new_id}                     │
    │  share=parent_id     │                        │
    │  Body: {path, can_read, ...}                  │
    │─────────────────────>│                        │
    │                      │  CanManageShare 中间件  │
    │                      │  ├─ ShareGet(new_id)    │
    │                      │  │  → ErrNotFound       │
    │                      │  │  (新ID→走创建者路径)  │  ❌ 不是创建者!
    │                      │  │                      │
    │                      │  ├─ _extractSession()   │
    │                      │  │  (无Share上下文)      │
    │                      │  │  GenerateID(session)  │
    │                      │  │  != s.Backend         │
    │                      │  │                      │
    │                      │  ├─ _extractShare()     │
    │                      │  │  (提取父共享上下文)    │
    │                      │  │  ShareGet(parent_id)  │
    │                      │  │─────────────────────>│
    │                      │  │<──parent Share record│
    │                      │  │  parent.CanShare?     │
    │                      │  │                      │
    │                      │  ├─ _extractSession()   │
    │                      │  │  Decrypt(parent.Auth) │
    │                      │  │  GenerateID(session)  │
    │                      │  │  == s.Backend?        │
    │                      │  │  && parent.CanShare?  │
    │                      │  │                      │
    │                      │  └─ ✓ 通过 → 执行 Handler│
    │                      │                        │
    │                      │  ShareUpsert()          │
    │                      │  ├─ Auth = parent.Auth  │  (复用父共享凭证)
    │                      │  ├─ Backend = parent.Backend│(复用父后端ID)
    │                      │  ├─ Path = parent.Path + rightPath│
    │                      │  └─ UPSERT Share        │
    │                      │───────────────────────>│
    │                      │                        │
    │    200 OK            │                        │
    │<─────────────────────│                        │
```

---

## 八、安全设计要点总结

| 安全机制 | 实现方式 | 代码位置 |
|---------|---------|---------|
| **凭证加密** | AES-256-GCM + Zlib + 分用途派生密钥 | `crypto.go:27-53`、`constants.go:72-79` |
| **Nonce 唯一性** | crypto/rand 种子 + 大端序递增 + Mutex | `crypto.go:240-265` |
| **密钥隔离** | 5 种派生密钥用途隔离（USER/PROOF/ADMIN/HASH/SGN） | `constants.go:64-78` |
| **密码存储** | bcrypt 慢哈希 + 验证失败延迟 1s | `share.go:92-93`、`share.go:160-161` |
| **路径隔离** | Session path chroot + PathBuilder 逃逸检测 + WebdavFs.fullpath() | `session.go:276-290`、`files.go:1113`、`webdav.go:125-134` |
| **验证码** | 4 位 crypto/rand 随机数 + 10 分钟过期 + 一次性消费 + 6h 定时清理 | `share.go:183-186`、`share.go:252-255`、`index.go:41-46` |
| **Cookie 安全** | HttpOnly + SameSite=None + Secure + 大小/数量限制 | `share.go:188-192`、`share.go:300` |
| **CSRF 防护** | SecureOrigin 中间件（XHR 检查 + Host 白名单） | `http.go:79-105` |
| **嵌套共享** | 权限继承（父权限含 CanShare 才可再共享）+ Auth/Backend 复用 | `session.go:131-154`、`share.go:44-65` |
| **上传保护** | 仅 CanUpload 时禁止覆盖已有文件（409 Conflict） | `files.go:498-513` |
| **主密钥安全** | Auth 字段加密绑定主密钥，密钥变更则全失效 | `session.go:270-274` |
| **数据库级联** | Location 删除级联 Share 清理 | `index.go:24` |
| **防暴力破解** | 密码/邮箱验证失败后 sleep 1 秒 + bcrypt 慢哈希 | `share.go:160`、`share.go:173` |
| **WebDAV 用户名防伪造** | email + HMAC 哈希校验码 | `session.go:238`、`share.go:654-656` |
| **限流** | 全局令牌桶 10 req/s, burst=1000（仅登录接口） | `http.go:107-121` |
| **Proof 防膨胀** | Cookie ≤ 500 字节, Proof ≤ 20 个 | `share.go:300`、`share.go:130` |

---

## 九、设计意图深度分析

### 9.1 sign/verify 空壳函数与 AES-GCM 完整性兜底

**代码事实**：`server/common/crypto.go:184-190`

```go
func sign(something []byte) ([]byte, error) {
    return something, nil  // 空壳：原样返回，无签名
}
func verify(something []byte) ([]byte, error) {
    return something, nil  // 空壳：原样返回，无验证
}
```

**全局搜索确认**：`sign(` 和 `verify(` 在整个 Go 代码库中无任何调用方。这两个函数是**死代码**（dead code），从未被任何加密流程使用。

**为何可以留空而不破坏安全性**：

系统实际使用的加密调用链是 `EncryptString` → `EncryptAESGCM`，其中 AES-256-GCM 模式自身提供了**认证加密**（AEAD）：

1. **加密时**：`gcm.Seal(nonce, nonce, plaintext, nil)` → 输出 = `nonce || ciphertext || 16字节认证标签`
2. **解密时**：`gcm.Open(nil, nonce, ciphertext, nil)` → 认证标签验证失败则返回 `error`

AES-GCM 的 16 字节认证标签（GHASH）等价于 HMAC-SHA256 截断 128 位的安全强度，提供：
- **完整性**（Integrity）：密文或 nonce 被篡改 → 解密失败
- **认证性**（Authentication）：无密钥则无法生成有效标签
- **抗重放**（Implicit via nonce）：NonceGenerator 递增 + 首随机种子保证 nonce 唯一

**被空壳函数欺骗的攻击面分析**：

虽然 `sign`/`verify` 未被调用，但如果未来开发者误用这些函数作为独立签名层（而非使用 AES-GCM），将产生以下攻击面：

| 误用场景 | 攻击方式 | 后果 |
|---------|---------|------|
| `sign(ciphertext)` 作为独立签名 | 签名为空壳，攻击者可替换 ciphertext | 密文被替换，`verify` 不检测 |
| `sign(plaintext)` 在加密前签名 | 签名与明文一起加密，AES-GCM 仍保护完整性 | 低风险——冗余但安全 |
| `verify(decrypted)` 在解密后验证 | 空壳验证始终成功 | 若跳过 GCM 认证则完全失效 |

**风险评级**：当前为**零风险**（无调用方），但作为代码卫生问题应移除或标注 `// DEPRECATED: AES-GCM provides authentication`，防止未来误用。

**`sign`/`verify` 的设计意图推测**：

从函数签名 `(something []byte) ([]byte, error)` 和命名来看，这很可能是**早期设计预留的签名层**，原计划用于：
- 在 AES-GCM 之上叠加一层 HMAC 签名（Encrypt-then-MAC 双保险）
- 或者用于 SSO 场景的属性签名（`SECRET_KEY_DERIVATE_FOR_SIGNATURE` 已派生但未使用）

AES-GCM 作为认证加密被选定后，独立签名层变为冗余，函数体被清空但保留了接口定义。

### 9.2 autovacuum 缺失 for 循环与运维兜底

**代码事实**：`server/model/index.go:41-46`

```go
func autovacuum() {
    if stmt, err := DB.Prepare("DELETE FROM Verification WHERE expire < datetime('now')"); err == nil {
        stmt.Exec()
    }
    time.Sleep(6 * time.Hour)
}
```

**问题**：函数末尾 `time.Sleep` 后直接返回，`goroutine` 退出。这意味着 `autovacuum` **仅执行一次**，6 小时 sleep 后不再清理。

**与上游仓库的关联**：

上游仓库 `github.com/mickael-kerjean/filestash` 的 `server/model/index.go` 中，此函数在历次提交中始终使用 `time.Sleep` 而非 `for {}` 循环。此行为在 GitHub Issues 中未被单独报告为 bug（搜索 "autovacuum" / "verification cleanup" / "expired code" 无直接匹配 issue）。

**现行运维兜底**：

| 兜底机制 | 代码位置 | 说明 |
|---------|---------|------|
| **验证码一次性消费** | `share.go:252-255` | 使用后立即 `DELETE FROM Verification WHERE code = ?` |
| **验证码 SQL 过期过滤** | `share.go:234-240` | 查询时 `WHERE expire > datetime('now')` 自动跳过过期记录 |
| **Verification 表索引** | `index.go:30-32` | `CREATE INDEX idx_verification ON Verification(code, expire)` 保证查询性能 |
| **SQLite AUTOINCREMENT** | 无（仅 `DATETIME DEFAULT`） | 过期记录不占自增 ID，仅磁盘空间膨胀 |

**实际影响评估**：

- **低流量场景**：Verification 表体积极小（每条 < 1KB），即使不清理也几乎无影响
- **高流量场景**：未清理的过期记录持续累积，`Verification` 表膨胀导致查询性能退化
- **补救措施**：运维可配置 cron job 执行 `sqlite3 state/db/share.sql "DELETE FROM Verification WHERE expire < datetime('now');"` 或在应用重启时触发一次性清理（`init` 中的 `go autovacuum()` 在每次启动时执行一次）

**建议修复**：

```go
func autovacuum() {
    for {  // 添加 for 循环
        if stmt, err := DB.Prepare("DELETE FROM Verification WHERE expire < datetime('now')"); err == nil {
            stmt.Exec()
        }
        time.Sleep(6 * time.Hour)
    }
}
```

### 9.3 子共享放宽权限为何静默忽略而非抛错

**代码事实**：`server/ctrl/share.go:42-87` 和 `server/model/share.go:85-138`

`ShareUpsert` 控制器和模型层**均无任何父共享权限校验逻辑**。创建子共享时：

1. 前端提交 `{can_read: true, can_write: true, can_upload: true}` → 后端直接写入 `params` JSON
2. `CanManageShare` 中间件仅验证**管理权归属**（是否为创建者或被 CanShare 授权），不校验权限范围
3. `ShareUpsert` 模型层仅处理密码哈希和数据库写入，不校验权限逻辑

**为何选择"静默忽略"而非"抛错拒绝"**：

从代码设计意图推断，这是**有意为之的最小权限原则实践**：

```
子共享权限判定流程：

  子共享访问者请求文件操作
       │
  SessionStart 中间件
       │
  _extractShare() → 加载子共享的 Share 记录
       │
  _extractSession() → 从子共享的 Auth 解密出 session
       │
  权限函数（CanRead/CanEdit/CanUpload）
       │
  判定依据：子共享自身的权限位
       │
  ┌──────────────────────────────────────┐
  │ 即使子共享声明 can_write=true,       │
  │ 子共享的 Auth = 父共享的 Auth,       │
  │ 父共享的 session.path = 父共享路径,  │
  │ PathBuilder 限制在子共享路径内,      │
  │ 且后端插件可能进一步限制操作。       │
  │                                      │
  │ 实际效果：子共享声明放宽权限无害，   │
  │ 因为权限判定按各自 Share 记录独立进行│
  └──────────────────────────────────────┘
```

**关键洞察**：权限判定函数 `CanRead(ctx)`/`CanEdit(ctx)` 只看 `ctx.Share`（当前共享），**不回溯父共享**。因此：

- 子共享声明 `can_write=true` → 访客在此子共享下**确实可以编辑**
- 但子共享的 `Auth` 复用父共享凭证 → 后端操作以创建者身份执行
- 子共享的 `Path` = 父共享路径 + 子路径 → 访客被限制在更窄的目录树内

**放宽权限不会真正提升权限**的原因：

| 限制维度 | 机制 | 子共享能否突破 |
|---------|------|-------------|
| 路径范围 | `session["path"]` = 子共享 Path（更窄） | ❌ 不能访问父共享路径外的文件 |
| 后端权限 | `Auth` = 父共享凭证（后端权限相同） | ❌ 后端层面权限不变 |
| Proof 验证 | 子共享可设置独立密码/邮箱 | ✅ 可更宽松（但这也合理） |
| CanShare | 子共享的 CanShare 独立判定 | ✅ 子共享可声明 CanShare=true（需通过 CanManageShare 中间件，已在 scenario 2 中校验） |

**结论**：静默忽略而非抛错是一种**防御性设计**——子共享的权限声明在运行时按自身记录独立判定，放宽声明不影响安全性，拒绝则增加不必要的复杂度。

### 9.4 嵌套共享的深度上限与循环引用检测

**代码事实**：

在整个共享相关代码中（`session.go`、`share.go`、`ctrl/share.go`），**不存在**：

1. **嵌套深度上限**：无 `maxDepth` 变量或递归计数器
2. **循环引用检测**：无 `visited` 集合或 ID 链追踪
3. **父共享 ID 追溯**：Share 结构体无 `parent_id` 字段，无法从数据库层面追溯共享链

**嵌套共享的数据模型**：

```
Share 表中子共享记录：
  id = "child-share-id"
  related_backend = 父共享的 Backend（= 创建者的 GenerateID(session)）
  related_path = 父共享.Path + 子路径（更深层）
  auth = 父共享的 Auth（完整复制）
  params = 子共享独立的权限位
```

**为何无循环引用风险**：

循环引用需要 `share_A` 的 Auth/Path 来自 `share_B`，同时 `share_B` 的 Auth/Path 来自 `share_A`。在当前模型下这是**不可能的**：

1. **Auth 来源唯一**：所有嵌套共享的 `Auth` 最终追溯到**原始创建者的 Cookie**（非共享上下文时 `ctx.Share.Id == ""`），或直接复制父共享的 `Auth`
2. **Path 单调递增**：子共享 `Path = parentPath + subPath`，路径只能越来越深，不可能形成环
3. **Backend 不变**：所有嵌套层共享同一个 `Backend` ID（原始创建者的 `GenerateID(session)`）

```
深度嵌套示意（假设存在 3 层）：

原始创建者 Session → Share A (auth=创建者Cookie, path=/docs/)
                        │
                  Share B (auth=A.auth, path=/docs/project/)
                        │
                  Share C (auth=A.auth, path=/docs/project/release/)

每层嵌套仅导致：
  - Path 更深（/docs/ → /docs/project/ → /docs/project/release/）
  - 权限位更窄（可限制不可放宽实际权限）
  - Auth 始终 = 创建者 Cookie（无环可循）
```

**无深度上限的潜在问题**：

| 问题 | 严重度 | 说明 |
|------|-------|------|
| Share 记录膨胀 | 低 | 每个子共享一条记录，SQLite 可承受数万条 |
| Proof Cookie 膨胀 | 低 | Proof ≤ 20 个限制防滥用（`share.go:130`） |
| 权限管理复杂度 | 中 | 无 `parent_id`，无法级联删除子共享；删除父共享后子共享的 Auth 失效（同一密钥加密） |
| 前端 UX 混乱 | 中 | 前端 `modal_share.js` 无嵌套层级展示 |

**`Loop Detected` 错误码**：`server/common/error.go:174` 定义了 HTTP 508 `Loop Detected` 状态码，但当前共享代码中**未使用此错误码**。它可能为未来 WebDAV 循环引用检测预留。

---

## 十、§CODE 锚点命名规范

为便于通过 `grep` 快速定位代码，所有文档中的代码锚点统一采用 `§CODE_` 前缀 + 模块缩写 + 功能描述的命名规范。

### 10.1 命名规范

```
§CODE_{MODULE}_{FUNCTION}

MODULE 缩写：
  CRYPTO    = server/common/crypto.go
  CONST     = server/common/constants.go
  TYPES     = server/common/types.go
  APP       = server/common/app.go
  ERR       = server/common/error.go
  SMODEL    = server/model/share.go
  IMODEL    = server/model/index.go
  PMODEL    = server/model/permissions.go
  WMODEL    = server/model/webdav.go
  SESSION   = server/middleware/session.go
  HTTPMW    = server/middleware/http.go
  MWIDX     = server/middleware/index.go
  CTXMW     = server/middleware/context.go
  SCTRL     = server/ctrl/share.go
  FCTRL     = server/ctrl/files.go
  WCTRL     = server/ctrl/webdav.go
  ROUTES    = server/routes.go
  SITE      = server/plugin/plg_handler_site/index.go
  SITEMW    = server/plugin/plg_handler_site/middleware.go
  SITECFG   = server/plugin/plg_handler_site/config.go
  FRONT     = public/assets/pages/filespage/modal_share.js
```

### 10.2 完整锚点索引

使用方式：`grep -rn "§CODE_SMODEL_ShareUpsert" server/` 即可定位到对应代码段。

| 锚点 | 文件 | 行号 | 描述 |
|------|------|------|------|
| `§CODE_CONST_InitSecretDerivate` | `server/common/constants.go` | L72-79 | 主密钥派生 5 种子密钥 |
| `§CODE_CONST_SecretKeyVars` | `server/common/constants.go` | L64-70 | 密钥全局变量声明 |
| `§CODE_CONST_CookieNames` | `server/common/constants.go` | L12-17 | Cookie 名称与路径常量 |
| `§CODE_CRYPTO_EncryptString` | `server/common/crypto.go` | L27-37 | 加密调用链入口 |
| `§CODE_CRYPTO_DecryptString` | `server/common/crypto.go` | L39-53 | 解密调用链入口 |
| `§CODE_CRYPTO_EncryptAESGCM` | `server/common/crypto.go` | L128-144 | AES-GCM 加密实现 |
| `§CODE_CRYPTO_DecryptAESGCM` | `server/common/crypto.go` | L146-164 | AES-GCM 解密实现 |
| `§CODE_CRYPTO_NonceGenerator` | `server/common/crypto.go` | L240-265 | Nonce 生成器（首随机+递增） |
| `§CODE_CRYPTO_SignVerifyStubs` | `server/common/crypto.go` | L184-190 | sign/verify 空壳函数 |
| `§CODE_CRYPTO_GenerateID` | `server/common/crypto.go` | L193-218 | Backend ID 生成算法 |
| `§CODE_CRYPTO_Hash` | `server/common/crypto.go` | L55-58 | SHA-256 哈希截断 |
| `§CODE_CRYPTO_RandomString` | `server/common/crypto.go` | L106-118 | 安全随机字符串生成 |
| `§CODE_TYPES_ShareStruct` | `server/common/types.go` | L180-194 | Share 结构体（5 权限位） |
| `§CODE_TYPES_MetadataStruct` | `server/common/types.go` | L164-176 | 前端权限元数据结构 |
| `§CODE_TYPES_IsValid` | `server/common/types.go` | L196-204 | 共享过期校验方法 |
| `§CODE_SMODEL_ShareUpsert` | `server/model/share.go` | L85-138 | 共享记录创建/更新 |
| `§CODE_SMODEL_ShareGet` | `server/model/share.go` | L66-83 | 共享记录查询 |
| `§CODE_SMODEL_ShareDelete` | `server/model/share.go` | L141-148 | 共享记录删除 |
| `§CODE_SMODEL_ProofVerifierPassword` | `server/model/share.go` | L263-268 | bcrypt 密码验证 |
| `§CODE_SMODEL_ProofVerifierEmail` | `server/model/share.go` | L269-289 | 邮箱白名单验证 |
| `§CODE_SMODEL_ProofGetAlreadyVerified` | `server/model/share.go` | L291-309 | Proof Cookie 解密读取 |
| `§CODE_SMODEL_ProofAreEquivalent` | `server/model/share.go` | L341-354 | Proof 等价性判断 |
| `§CODE_SMODEL_NetworkDriveUsernameEnc` | `server/model/share.go` | L654-656 | WebDAV 用户名编码 |
| `§CODE_IMODEL_CreateTables` | `server/model/index.go` | L20-33 | 数据库建表 DDL |
| `§CODE_IMODEL_Autovacuum` | `server/model/index.go` | L41-46 | 验证码定时清理（缺陷：无 for 循环） |
| `§CODE_PMODEL_CanReadWrite` | `server/model/permissions.go` | L1-33 | 权限判定核心函数 |
| `§CODE_WMODEL_Fullpath` | `server/model/webdav.go` | L125-134 | WebDAV 路径隔离 |
| `§CODE_WMODEL_NewWebdavFs` | `server/model/webdav.go` | L42-49 | WebDAV 文件系统构造 |
| `§CODE_SESSION_SessionStart` | `server/middleware/session.go` | L57-93 | 会话启动中间件 |
| `§CODE_SESSION_CanManageShare` | `server/middleware/session.go` | L96-158 | 共享管理权三层判定 |
| `§CODE_SESSION_ExtractShareId` | `server/middleware/session.go` | L190-200 | 共享 ID 提取 |
| `§CODE_SESSION_ExtractShare` | `server/middleware/session.go` | L202-260 | 共享上下文提取与 Proof 校验 |
| `§CODE_SESSION_ExtractSession` | `server/middleware/session.go` | L262-315 | Session 解密与 Chroot |
| `§CODE_SESSION_BasicAuthParse` | `server/middleware/session.go` | L222-242 | WebDAV Basic Auth 解析 |
| `§CODE_HTTPMW_SecureOrigin` | `server/middleware/http.go` | L79-105 | CSRF 防护中间件 |
| `§CODE_HTTPMW_RateLimiter` | `server/middleware/http.go` | L107-121 | 令牌桶限流 |
| `§CODE_HTTPMW_SecureHeaders` | `server/middleware/http.go` | L67-77 | 安全响应头 |
| `§CODE_MWIDX_NewMiddlewareChain` | `server/middleware/index.go` | L21-37 | 中间件链执行引擎 |
| `§CODE_MWIDX_PluginInjector` | `server/middleware/index.go` | L72-76 | 插件中间件注入 |
| `§CODE_SCTRL_ShareUpsert` | `server/ctrl/share.go` | L35-94 | 共享创建/更新控制器 |
| `§CODE_SCTRL_ShareVerifyProof` | `server/ctrl/share.go` | L106-213 | Proof 验证控制器 |
| `§CODE_SCTRL_ShareDelete` | `server/ctrl/share.go` | L96-104 | 共享删除控制器 |
| `§CODE_FCTRL_FileLs` | `server/ctrl/files.go` | L79-191 | 文件列表 + 权限探测 |
| `§CODE_FCTRL_FileSave` | `server/ctrl/files.go` | L479-538 | 文件保存（CanEdit vs CanUpload） |
| `§CODE_FCTRL_FileAccess` | `server/ctrl/files.go` | L443-475 | HTTP 方法权限预检 |
| `§CODE_FCTRL_PathBuilder` | `server/ctrl/files.go` | L1104-1117 | 路径构建 + 逃逸检测 |
| `§CODE_WCTRL_WebdavHandler` | `server/ctrl/webdav.go` | L13-52 | WebDAV 权限分发 |
| `§CODE_ROUTES_ShareRoutes` | `server/routes.go` | L70-79 | 共享 API 路由配置 |
| `§CODE_ROUTES_WebdavRoutes` | `server/routes.go` | L87-91 | WebDAV 路由配置 |
| `§CODE_ROUTES_FileRoutes` | `server/routes.go` | L53-68 | 文件 API 路由配置 |
| `§CODE_SITE_SiteHandler` | `server/plugin/plg_handler_site/index.go` | L35-83 | 公共站点处理器 |
| `§CODE_SITE_RouteRegistration` | `server/plugin/plg_handler_site/index.go` | L17-32 | 站点路由注册 |
| `§CODE_SITEMW_BasicAdmin` | `server/plugin/plg_handler_site/middleware.go` | L34-48 | 管理员 Basic Auth |
| `§CODE_SITEMW_CORS` | `server/plugin/plg_handler_site/middleware.go` | L12-32 | 站点 CORS 中间件 |
| `§CODE_SITECFG_PluginEnable` | `server/plugin/plg_handler_site/config.go` | L16-28 | 站点功能开关配置 |
| `§CODE_ERR_LoopDetected` | `server/common/error.go` | L174 | HTTP 508 Loop Detected 定义 |
| `§CODE_FRONT_RoleMapping` | `public/assets/pages/filespage/modal_share.js` | L329-357 | 前端角色→权限映射 |
