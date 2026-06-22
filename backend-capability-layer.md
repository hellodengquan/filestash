# Filestash 存储后端能力抽象层分析

## 1. 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                        HTTP Controller                       │
│  (ctrl/files.go: FileLs / FileCat / FileMv / FileRm / ...)  │
└──────────────────────────┬──────────────────────────────────┘
                           │ ctx.Backend (IBackend 接口)
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                  Model 层 (model/files.go)                   │
│  - NewBackend(): Config.Conn 白名单校验 → 不匹配返回 Nothing{} │
│  - GetHome(): Home() 接口探测 + 路径修正                       │
│  - CanRead/CanEdit/CanUpload/CanShare 权限开关                │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                Driver 注册中心 (common/backend.go)            │
│  Driver.ds: map[string]IBackend                              │
│  - Register(name, IBackend)                                  │
│  - Get(name) -> IBackend (未知类型返回 Nothing{})            │
└──────────────────────────┬──────────────────────────────────┘
                           │
    ┌──────────────────────┼──────────────────────┐
    ▼                      ▼                      ▼
┌──────────┐         ┌──────────┐          ┌──────────┐
│   S3     │         │   FTP    │          │  SFTP    │
│plg_backen│         │plg_backen│          │plg_backen│
│  d_s3    │         │  d_ftp   │          │  d_sftp  │
└──────────┘         └──────────┘          └──────────┘
    ... (20+ 后端插件: WebDAV / Git / GDrive / Dropbox / Local ...)
```

---

## 2. 核心接口约束 (IBackend)

定义位置：`server/common/types.go:13-24`

```go
type IBackend interface {
    Init(params map[string]string, app *App) (IBackend, error)  // 初始化 & 连接
    Ls(path string) ([]os.FileInfo, error)                      // 列目录
    Stat(path string) (os.FileInfo, error)                      // 获取文件元信息
    Cat(path string) (io.ReadCloser, error)                     // 读文件
    Mkdir(path string) error                                    // 建目录
    Rm(path string) error                                       // 删除
    Mv(from string, to string) error                            // 重命名/移动
    Save(path string, file io.Reader) error                     // 写文件
    Touch(path string) error                                    // 创建空文件
    LoginForm() Form                                            // 登录表单描述
}
```

### 接口契约的刚性与弹性

| 方法 | 约束刚性 | 典型降级路径 |
|------|---------|-------------|
| `Init` | **刚性** - 失败即无法使用后端 | FTP: 尝试 ftp → ftps::implicit → ftps::explicit 多种连接模式 |
| `Ls` | **刚性** - 核心只读操作 | 部分后端返回空 `[]FileInfo{}` 而非错误 |
| `Stat` | **弹性** - 多个后端 `ErrNotImplemented` | `ctrl/files.go:FileCat` 中 HEAD 请求时先 `Stat` 失败则 fallback 到 `Cat` |
| `Cat` | **半刚性** - 读场景核心 | S3: 加密失败自动降级为非加密读取 |
| `Mkdir` | **弹性** | S3 根目录禁止建文件/目录 |
| `Rm` | **弹性** | Dav (CardDAV/CalDAV) 仅支持 2 层深度删除 |
| `Mv` | **弹性** - 差异最大 | Dav: 通过 Cat+Save+Rm 模拟；S3: Bucket 级不支持重命名 |
| `Save` | **半刚性** | Git: 写入后自动 commit & push |
| `Touch` | **弹性** | 多数后端用 `Save(path, "")` 模拟 |

---

## 3. 可选扩展接口（鸭子类型能力探测）

Filestash **不**使用接口嵌入来声明可选能力，而是在调用方用 Go 类型断言做运行时探测。

### 3.1 Meta 接口 —— 能力声明

**探测位置**: `server/ctrl/files.go:96-98` 和 `server/ctrl/files.go:451-453`

```go
// 不是 IBackend 的正式成员，而是通过断言检查
if obj, ok := ctx.Backend.(interface{ Meta(path string) Metadata }); ok {
    perms = obj.Meta(path)
}
```

**Metadata 结构** (`server/common/types.go:164-176`):

```go
type Metadata struct {
    CanSee             *bool      // 是否可查看
    CanCreateFile      *bool      // 能否创建文件
    CanCreateDirectory *bool      // 能否创建目录
    CanRename          *bool      // 能否重命名
    CanMove            *bool      // 能否移动
    CanUpload          *bool      // 能否上传
    CanDelete          *bool      // 能否删除
    CanShare           *bool      // 能否分享
    HideExtension      *bool      // 是否隐藏扩展名
    RefreshOnCreate    *bool      // 创建后是否刷新
}
```

> **关键设计**: 使用 `*bool` 三态（nil / true / false）—— nil 表示"未声明，使用默认许可"。

#### 各后端 Meta 声明对比

| 后端 | 路径条件 | 声明的能力约束 | 文件位置 |
|------|---------|--------------|---------|
| **S3** | `/` (根) | `CanCreateFile=false, CanRename=false, CanMove=false, CanUpload=false` | `plg_backend_s3/index.go:182-192` |
| **FTP** | ACL == "r" (匿名模式) | 所有写能力全部 `false` | `plg_backend_ftp/index.go:239-251` |
| **Dav (CardDAV/CalDAV)** | 全局 | `CanMove=false, HideExtension=true` | `plg_backend_dav/index.go:344-363` |
| **Dav** | `/` (根) | `CanCreateFile=false, CanCreateDirectory=true, CanRename=false, CanUpload=false, RefreshOnCreate=false` | 同上 |
| **Dav** | 非根 | `CanCreateFile=true, CanCreateDirectory=false, CanRename=true, CanUpload=true, RefreshOnCreate=true` | 同上 |
| **Local** | —— | 无 Meta 实现 → 全部走默认 | —— |
| **SFTP** | —— | 无 Meta 实现 → 全部走默认 | —— |
| **WebDAV** | —— | 无 Meta 实现 → 全部走默认 | —— |
| **Git** | —— | 无 Meta 实现 → 全部走默认 | —— |
| **Dropbox** | —— | 无 Meta 实现 → 全部走默认 | —— |
| **GDrive** | —— | 无 Meta 实现 → 全部走默认 | —— |

### 3.2 Home 接口 —— 工作目录探测

**探测位置**: `server/model/files.go:58-63`

```go
if obj, ok := b.(interface{ Home() (string, error) }); ok {
    tmp, err := obj.Home()   // 优先用后端提供的 Home
    ...
} else if _, err := b.Ls(base); err != nil {
    return base, err         // 降级：用 Ls 探测 base 是否有效
}
```

| 后端 | Home() 实现 | 默认 fallback |
|------|------------|-------------|
| FTP | `client.Getwd()` | 登录后服务器默认 CWD |
| SFTP | `SFTPClient.Getwd()` | 同上 |
| Local | `os.UserHomeDir()` → `user.Current().HomeDir` → `"/"` | 三级 fallback |
| S3 / WebDAV / Git / GDrive / Dropbox | **无实现** | 走 `Ls(base)` 探测路径有效性 |

### 3.3 OAuth 接口 —— 认证协议探测

**探测位置**（在前端/会话层）: 查找 `OAuthURL()` / `OAuthToken()` 方法

| 后端 | OAuthURL() | OAuthToken() |
|------|-----------|-------------|
| Dropbox | ✓ (dropbox.com/oauth2) | 无（前端拿 token） |
| GDrive | ✓ (google Endpoint) | ✓（Exchange code → token） |
| S3 / FTP / SFTP / WebDAV / Git | ✗（用户手动填凭据） | ✗ |

---

## 4. Init 阶段的连接策略与降级

### 4.1 FTP: 连接模式自动探测链

位置：`server/plugin/plg_backend_ftp/index.go:87-173`

```
hostname 前缀判断
   │
   ├─ ftp://      → 只尝试明文 FTP
   ├─ ftps://     → 先 implicit TLS，再 explicit TLS
   └─ (无前缀)    → 完整三级策略:
                     1. ftp (明文)
                        └─ 5s 超时拨号 → ReadDir("/") 验证 → 重新 60s 长连接
                     2. ftps::implicit
                        └─ 60s TLS 拨号 + TLSImplicit 模式 + ReadDir 验证
                     3. ftps::explicit
                        └─ 5s 拨号 + AUTH TLS 握手 → ReadDir 验证 → 重连 60s
```

**失败回退**: 全部模式失败 → `ErrAuthenticationFailed`

### 4.2 SFTP: 认证方式自动探测链

位置：`server/plugin/plg_backend_sftp/index.go:128-152`

```
密码字段内容判断
   │
   ├─ 匹配 PEM 私钥格式 (-----BEGIN ...-----)
   │     └─ SSH PublicKey Auth（带 passphrase 或不带）
   │
   └─ 普通字符串
         └─ 同时提交 2 种认证:
             1. ssh.Password(password)
             2. ssh.KeyboardInteractive(用 password 回答所有 challenge)
```

**额外降级**: 用户名大小写重试验证（`index.go:174-179`）

### 4.3 S3: 凭据链与区域默认值 + Bucket 区域探测

位置：`server/plugin/plg_backend_s3/index.go:46-99` 与 `utils.go:57-77`

| 参数 | 默认值 / 降级策略 |
|------|------------------|
| region (Init阶段) | `"us-east-1"`，若 endpoint 含 `.cloudflarestorage.com` → `"auto"` |
| region (运行时) | 首次访问某 Bucket 时调用 `GetBucketLocation` 自动探测，结果缓存至 `S3Cache` (key = `{bucket, params}`) |
| credentials | ① Static AK/SK → ② STS AssumeRole → ③ EnvProvider → ④ RemoteCred (EC2/ECS IAM) |
| number_thread | 缺省 = 50；超出 [1, 5000] 范围 → 强制 = 2 |
| encryption_key | 长度校验：必须 32 字符（否则 400 错误） |

> **二级探测**: 用户可不在连接参数中填 region，系统在 `createSession(bucket)` 时按需调用 `GetBucketLocation` API 查询该 Bucket 实际所在区域并动态切换 config.Region。

### 4.4 Git: 默认值填充

位置：`server/plugin/plg_backend_git/index.go:73-91`

| 参数 | 默认值 |
|------|-------|
| branch | `"master"` |
| commit message | `"{action} ({filename}): {path}"` |
| authorName / committerName | `APPNAME` (Filestash) |
| authorEmail / committerEmail | `"https://filestash.app"` (非白标时) |

---

## 5. 控制器层的权限与能力归并

位置：`server/ctrl/files.go:FileLs:79-144`

**能力归并顺序**（后者覆盖前者，`false` 为不可逆降级）：

```
1. 后端 Meta() 声明          → perms (初始)
        │
        ▼
2. IAuthorisation 中间件逐个试调用   → 失败则把对应能力置 false
   - auth.Ls()   失败 → 拦截返回
   - auth.Mkdir() 失败 → CanCreateDirectory=false
   - auth.Touch() 失败 → CanCreateFile=false
   - auth.Mv()    失败 → CanRename=false, CanMove=false
   - auth.Save()  失败 → CanUpload=false
   - auth.Rm()    失败 → CanDelete=false
   - auth.Cat()   失败 → CanSee=false
        │
        ▼
3. model.CanEdit(ctx) == false → 全部写能力强制 false
        │
        ▼
4. model.CanUpload(ctx) == false → 上传相关能力强制 false
        │
        ▼
5. model.CanShare(ctx) == false → CanShare 强制 false
```

### 权限来源（`model/permissions.go`）

| 函数 | Share 链接场景 | 正常登录场景 |
|------|---------------|-------------|
| `CanRead` | `ctx.Share.CanRead` | `true` |
| `CanEdit` | `ctx.Share.CanWrite` | `true` |
| `CanUpload` | `ctx.Share.CanUpload` | `true` |
| `CanShare` | `ctx.Share.CanShare` | `true` |

---

## 6. 典型能力降级与回退路径

### 6.1 Range 请求（断点续传）降级

位置：`server/ctrl/files.go:312-358`

```
客户端发送 Range Header
   │
   ├─ 已有缓存文件 → 直接本地 io.Seeker
   │
   └─ 后端 Cat 返回的 ReadCloser
         │
         ├─ 实现了 io.Seeker? ──Yes──► 直接 seek & 返回部分内容
         │
         └─ No (大多数 HTTP 后端: S3/WebDAV/GDrive)
               │
               ▼
           降级路径:
           1. 创建本地临时文件 /tmp/file_{rand}.dat
           2. io.Copy 把后端内容全量下载到本地
           3. 关闭源 stream，打开本地文件
           4. 对本地文件执行 seek，返回 Range 片段
           5. 写入 file_cache（session 级缓存）
```

### 6.2 Cat 加密降级 + Glacier 离线文件处理（S3 特有）

位置：`server/plugin/plg_backend_s3/index.go:320-339` 与 `index.go:230-239`

```
用户配置了 encryption_key？
   │
   Yes ──► 所有 Cat 请求带上 SSE-C 头
   │         │
   │         ├─ 成功: 返回解密内容
   │         │
   │         ├─ InvalidRequest + encryption 字样错误
   │         │        │
   │         │        ▼
   │         │    静默降级: 去掉加密头重试
   │         │    (兼容未加密的已有文件)
   │         │
   │         ├─ InvalidArgument + "secret key was invalid"
   │         │        └─► 返回明确错误: "This file is encrypted file, you need the correct key!"
   │         │
   │         ├─ AccessDenied → ErrNotAllowed
   │         │
   │         └─ InvalidObjectState → ErrNotReachable (Glacier 离线文件)
   │
   No ──► 普通 GetObject
```

**Glacier 离线文件标记**: `Ls` 时检测到 `StorageClass == "GLACIER"`，返回的 `File.Offline = true`。前端据此展示"离线"标识。
`FileDownloader`（打包下载）遇到 `ErrNotReachable` 时跳过该文件继续处理其他文件，不中断整个 zip。

### 6.3 Mv 能力的三层实现

| 实现策略 | 后端 | 原理 |
|---------|------|------|
| **原子 Rename** | Local / SFTP / FTP | 底层 `rename()` / `RNFR+RNTO` / `SSH_FXP_RENAME` |
| **Copy + Delete** | S3 / WebDAV / GDrive / Dropbox | `CopyObject` + `DeleteObject`（文件级）；目录级需递归遍历 |
| **Read + Write + Delete** (完全模拟) | Dav (CardDAV/CalDAV) | `Cat` 读内容 → 解析修改 FN/SUMMARY 字段 → `Save` 写新路径 → `Rm` 删旧路径 |

> **S3 特殊限制**: Bucket 级（path == "/"）`Mv` 直接返回 `ErrNotImplemented`
> 参考: `plg_backend_s3/index.go:443-446`

### 6.4 删除 (Rm) 递归实现

| 后端 | 递归删除策略 |
|------|-------------|
| Local | `os.RemoveAll` (系统调用) |
| SFTP | 手动递归 `ReadDir` → 逐个文件 `Remove` → 子目录递归 → `RemoveDirectory` |
| FTP | 手动递归 + `goftp.Error` code < 300 视为成功（bsftp 兼容） |
| S3 | `ListObjectsV2Pages` 分页 + 多线程 `DeleteObject` 池 → 最后 `DeleteBucket` (若需) |
| WebDAV | 依赖服务端 `DELETE` 集合是否支持递归 |
| Git | `SafeOsRemoveAll` 本地工作区 + 自动 commit & push |
| Dav (CardDAV/CalDAV) | **禁止深度 > 2** → 返回 `ErrNotValid` |

### 6.5 FTP Execute 错误码归一化 & 自动重连

位置：`server/plugin/plg_backend_ftp/index.go:373-399`

```
goftp.Error 分类:
   │
   ├─ 0 < code < 300 → 视为成功 (bsftp rm 返回 200 OK 的兼容 hack)
   │
   ├─ code == 421 / (code==0 && "EOF") → 连接失效自动重连
   │     └─ Close → cache 清除 → 重新 Init → 递归 Execute 原调用
   │
   └─ code == 550 → 二次语义判断:
         ├─ message == "permission denied" → ErrPermissionDenied
         └─ 其他                         → ErrNotFound
```

### 6.6 Stat 回退

| 后端 | Stat 能力 |
|------|----------|
| S3 | ✓ `HeadObject`；若 NotFound 且非文件 → 假设是目录（返回 FType=directory, FTime=-1） |
| Dropbox | ✗ `ErrNotImplemented` |
| GDrive | ✗ `ErrNotImplemented` |
| Dav | ✗ `ErrNotImplemented` |
| 其余 (Local/FTP/SFTP/WebDAV/Git) | ✓ 原生支持 |

> **调用方回退**: `FileCat` HEAD 请求中 `Stat` 失败不报错，而是继续用 `Cat` 尝试

### 6.7 Touch 回退

```
后端是否实现 Touch?
   ├─ Yes → Local/S3/FTP/SFTP (各有原生创建空文件方式)
   │
   └─ No → Dropbox/WebDAV/Dav/Git
           ▼
           统一回退: Save(path, strings.NewReader(""))
```

---

## 7. 错误码归一化层

统一错误定义: `server/common/error.go:15-31`

| 错误常量 | HTTP 码 | 典型场景 |
|---------|--------|---------|
| `ErrNotFound` | 404 | 资源不存在 |
| `ErrNotAllowed` / `ErrPermissionDenied` | 403 | ACL / 权限拒绝 |
| `ErrNotValid` | 405 | 不合法的操作（如 Dav 深层操作） |
| `ErrConflict` | 409 | 资源已存在 |
| `ErrNotReachable` | 502 | 无法建立连接（如 S3 Glacier 离线文件） |
| `ErrNotImplemented` | 501 | 后端不支持该方法 |
| `ErrFilesystemError` | 503 | 本地文件系统安全拦截（路径穿越等） |
| `ErrAuthenticationFailed` | 400 | 凭据验证失败 |
| `ErrTimeout` | 500 | Zip 下载/解压超时 |

### 各后端错误翻译实现

| 后端 | 归一化策略 | 文件位置 |
|------|-----------|---------|
| SFTP | `err()` 方法完整映射 0-31 号 SFTP 状态码 | `plg_backend_sftp/index.go:357-432` |
| FTP | `Execute()` 包装器分类 421/550 | `plg_backend_ftp/index.go:373-399` |
| S3 | 捕获 awserr.Error Code → `AccessDenied→ErrNotAllowed` / `InvalidObjectState→ErrNotReachable` / 加密错误语义 | `plg_backend_s3/index.go:320-339` |
| WebDAV | HTTP 状态码 + `HTTPFriendlyStatus()` 文本化 | `plg_backend_webdav/index.go` (多处) |
| Dav (Card/Cal) | 深度检测 → 超过 2 层直接 `ErrNotValid` | `plg_backend_dav/index.go:140-141` |
| Local | `SafeOs*` 系列包装 → `processError()` 剥 PathError/LinkError → `safePath()` 符号链接+路径穿越检测 → `ErrFilesystemError` | `common/files.go:79-164` |

#### Local 后端的安全文件操作层 (`common/files.go`)

Local / Git 后端并不直接调用 `os.OpenFile` / `os.Remove`，而是统一走 `SafeOs*` 系列函数，这是一层 **安全降级+防护**：

```
SafeOsOpenFile / SafeOsMkdir / SafeOsRemove / SafeOsRemoveAll / SafeOsRename
   │
   ├─ 1. safePath(path) 检查:
   │     ├─ filepath.EvalSymlinks 解析所有符号链接
   │     ├─ 若解析后路径 != filepath.Clean(原始路径) → 符号链接穿越 → ErrFilesystemError
   │     └─ 路径不存在时递归向父目录验证
   │
   ├─ 2. Linux 平台强制添加 O_NOFOLLOW flag (不跟随末尾符号链接)
   │
   └─ 3. processError 剥掉 *os.PathError / *os.LinkError 包装，暴露底层 errno
         └─ fs.ErrNotExist → 统一翻译为 ErrNotFound
```

---

## 8. 后端连接缓存与生命周期

AppCache 机制（`common/cache.go`）:

| 后端 | 缓存键 | 驱逐行为 |
|------|-------|---------|
| FTP | params (全量连接参数) | `wg.Wait()` 等待 in-flight 请求 → `Close()` 关闭 TCP |
| SFTP | params | 同上 + SSH/SFTP 双连接关闭 |
| Git | params | `os.RemoveAll` 清理本地 clone 目录 |
| Dav | params | 内存缓存（无连接关闭） |
| S3 / WebDAV / Dropbox / GDrive | **无缓存** —— 每次请求都新建 HTTP Client / SDK Session |

---

## 9. 后端能力矩阵总览

| 能力 | Local | S3 | FTP | SFTP | WebDAV | Git | GDrive | Dropbox | Dav(Card/Cal) |
|------|-------|----|-----|------|--------|-----|--------|---------|--------------|
| Ls | ✓ | ✓(Bucket 级+对象级) | ✓ | ✓ | ✓ | ✓(本地 clone) | ✓ | ✓ | ✓(仅 2 层) |
| Stat | ✓ | ✓(NotFound→目录推断) | ✓ | ✓ | ✓ | ✓ | ✗ | ✗ | ✗ |
| Cat | ✓ | ✓(加密降级) | ✓ | ✓ | ✓ | ✓ | ✓(格式导出降级) | ✓ | ✓ |
| Mkdir | ✓ | ✓(S3 根禁用) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓(根才允许) |
| Rm | ✓ | ✓(多线程递归) | ✓ | ✓ | ✓ | ✓(+commit+push) | ✓ | ✓ | ✓(≤2 层) |
| Mv | ✓(原子) | ✓(Copy+Del, Bucket级不行) | ✓(原子) | ✓(原子) | ✓(MOVE) | ✓(原子+commit) | ✓ | ✓ | ✗(用Cat+Save+Rm模拟) |
| Save | ✓ | ✓(分片 Uploader) | ✓ | ✓ | ✓ | ✓(+commit+push) | ✓ | ✓ | ✓(生成随机文件名) |
| Touch | ✓ | ✓ | ✓ | ✓ | Save("") 模拟 | ✓ | Save("") 模拟 | Save("") 模拟 | ✓(模板生成 VCARD/ICS) |
| Meta | — | 根禁用写 | ACL 只读设全 false | — | — | — | — | — | 详细分层控制 |
| Home | ✓(三级 fallback) | — | ✓ | ✓ | — | — | — | — | — |
| 连接缓存 | — | — | ✓(带重连) | ✓ | — | ✓(清理 clone) | — | — | ✓ |

---

## 10. 后端实例化链路：白名单校验 + 空后端兜底

完整的后端实例化链路 (`server/model/files.go:9-51` + `common/backend.go:28-34`):

```
HTTP 请求携带的 conn 参数 {type, hostname, path, ...}
   │
   ▼
model.NewBackend(): 与 Config.Conn 白名单逐条比对
   - type 必须匹配
   - hostname (若配置了) 必须完全相同
   - path (若配置了) 必须是配置 path 的子路径
   - url (若配置了) 必须完全相同
   │
   ├─ 不匹配 → Backend.Get(BACKEND_NIL) → 返回 Nothing{} + ErrNotAllowed
   │
   └─ 匹配 → Backend.Get(conn["type"]).Init(conn, ctx)
              │
              ├─ Init 成功 → 返回具体后端实例
              │
              └─ Driver.Get(name) 找不到注册 → 返回 Nothing{}
                          │
                          ▼
                   Nothing{}: 所有写操作返回 ErrNotImplemented
                              Ls 返回空数组
                              Stat 返回 ErrNotFound
```

> **双层兜底**: ① 白名单不匹配时故意返回 Nothing{} 并带 ErrNotAllowed；② 未知后端名称时 Driver.Get 也返回 Nothing{}。两层都能保证上层永远拿到一个合法的 IBackend，不会出现 nil panic。

---

## 11. 关键设计洞察

1. **鸭子类型能力探测**: 可选能力（Meta/Home/OAuth）不嵌入 IBackend，而是在调用处用 `if obj, ok := b.(interface{ ... })` 做断言。好处是新增能力无需修改所有后端，坏处是能力清单散落在调用方。

2. **三态权限（nil/true/false）**: Metadata 用 `*bool` 而非 `bool`，nil 表示"不限制，交给上层决策"。这使得新增能力字段时老后端天然兼容（未声明 = nil = 走默认逻辑）。

3. **IAuthorisation 中间件的"试调用探测"**: 权限中间件不是声明式的，而是通过真正调用 `auth.Mkdir(path)` / `auth.Touch(path)` 等方法来探测某路径是否允许该操作 —— 失败则把对应能力位设为 false。这是一种"运行时探测"而非"静态声明"。

4. **错误码翻译层**: 每个后端把各自的 SDK/协议错误翻译成统一的 `ErrXxx` 哨兵值，这是抽象层最重要的"隐形适配"之一，控制器层完全不关心 S3 awserr vs SFTP 状态码 vs FTP code。

5. **Init 作为能力协商阶段**: Init 不只是"创建连接"，而是包含一系列的协商+降级：FTP 自动试 TLS 模式、SFTP 自动试多种 Auth 方法、S3 凭据链遍历、Local Home 三级 fallback。

6. **"尽力而为"的 Mv/Rm**: 跨后端语义差异最大的操作，提供三层模拟（原子 → 复制删除 → 读写删除），但代价是性能和一致性差距很大（S3 目录级 Mv 是 O(N) API 调用）。

7. **Git 后端的特殊地位**: Git 是唯一"写操作自带副作用"的后端（Save/Touch/Mv/Rm 都会 commit + push），也是唯一用本地工作区做中间层的后端（路径操作全部委托到临时目录的本地文件系统）。

8. **SafeOs* 安全层**: Local 和 Git 后端并不直接调用 os 包，而是通过 SafeOs* 系列函数执行路径安全校验（符号链接穿越检测 + O_NOFOLLOW），这是抽象层中容易被忽略的"能力降级+安全防护"双重角色。

---

## 12. 深度分析：能力降级默认值的取值策略

结论先行：**默认值采用「开放默认（opt-out）」而非「取最弱后端能力（交集）」**。即"未显式声明 = 允许"，然后通过多层叠加逐步收窄为 false。

### 12.1 Metadata 权限三态：nil = true（开放默认）

位置：`server/ctrl/files.go:443-475` (`FileAccess`) 与 `ctrl/files.go:95-144` (`FileLs`)

```go
// FileAccess: 判断是否允许 GET
if perms.CanSee == nil || *perms.CanSee == true {
    allowed = append(allowed, "GET")
}
// FileLs: perms := Metadata{}   // 零值全部为 nil → 初始视为全部允许
```

| `*bool` 值 | 语义 | 判定结果 |
|-----------|------|---------|
| `nil` | 后端未显式声明 → 交给上层决策 | 允许（等价 true） |
| `NewBool(true)` | 后端显式声明允许 | 允许 |
| `NewBool(false)` | 后端显式声明禁止 | 禁止 |

**关键洞察**：新增 `Metadata` 字段时，所有老后端天然兼容（都是 nil = 允许），不需要逐个改代码填充默认值。

### 12.2 权限位叠加顺序：单向收窄（true → false，不可逆）

`FileLs` 中权限被 4 层叠加，**一旦变 false 就再也变不回 true**：

```
第 0 层: perms = Metadata{}         ──► 全 nil = 全允许
           │
第 1 层: backend.Meta(path)         ──► 后端按路径显式置 false（如 S3 根、Dav 分层）
           │
第 2 层: IAuthorisation 中间件循环   ──► 试调用 Mkdir/Touch/Mv/Save/Rm/Cat
           │                           失败就把对应能力置 false
           │
第 3 层: model.CanEdit(ctx) == false ──► 批量置 CanCreateFile/Dir/Rename/Move/Delete/Upload = false
           │
第 4 层: model.CanUpload(ctx) == false ─► 进一步收窄
           │
第 5 层: model.CanShare(ctx) == false ──► CanShare 置 false
```

> 注意：第 2 层中间件使用的是"运行时试调用探测"，而非读取静态声明。

### 12.3 IAuthorisation 的"试调用探测"机制

位置：`server/ctrl/files.go:99-126`

```go
for _, auth := range Hooks.Get.AuthorisationMiddleware() {
    if err = auth.Ls(ctx, path); err != nil {          // 先查 Ls 权限（直接拒绝整个请求）
        return ErrNotAuthorized
    }
    if err = auth.Mkdir(ctx, path); err != nil {       // Mkdir 试调用
        perms.CanCreateDirectory = NewBool(false)      // 失败 → 置 false
    }
    if err = auth.Touch(ctx, path); err != nil {       // Touch 试调用
        perms.CanCreateFile = NewBool(false)           // 失败 → 置 false
    }
    if err = auth.Mv(ctx, path, path); err != nil {    // Mv 试调用
        perms.CanRename = NewBool(false)
        perms.CanMove = NewBool(false)
    }
    if err = auth.Save(ctx, path); err != nil {        // Save 试调用
        perms.CanUpload = NewBool(false)
    }
    if err = auth.Rm(ctx, path); err != nil {          // Rm 试调用
        perms.CanDelete = NewBool(false)
    }
    if err = auth.Cat(ctx, path); err != nil {         // Cat 试调用
        perms.CanSee = NewBool(false)
    }
}
```

**典型实现**：自带的审计和 Share 中间件就是基于此模式。路径穿越检测或分享链接过期时，试调用失败，对应权限位被置为 false。

### 12.4 Init 参数的默认值策略：每个后端自己的硬编码常识值

各后端 `Init` 中的默认值不是按"最弱后端"来取，而是按"该协议常见值"硬编码在各自实现里：

| 参数 | 后端 | 默认值 | 文件位置 |
|------|------|--------|---------|
| `hostname` | FTP | `"localhost"` | `plg_backend_ftp/index.go:70` |
| `port` | FTP | `"21"` | 同上 |
| `username` | FTP | `"anonymous"` | 同上 |
| `max_conn` | FTP | `"5"` | 同上 |
| `tls_mode` | FTP | 三级协商 `ftp → ftps::implicit → ftps::explicit` | `plg_backend_ftp/index.go:147-169` |
| `port` | SFTP | `"22"` | `plg_backend_sftp/index.go:76` |
| `region` | S3 | `"us-east-1"` | `plg_backend_s3/index.go:53` |
| `region` (Cloudflare R2) | S3 | 识别 endpoint → 改为 `"auto"` | `plg_backend_s3/index.go:50-56` |
| `region` (运行时) | S3 | `GetBucketLocation` 按需探测并缓存 | `plg_backend_s3/utils.go:57-77` |
| `number_thread` | S3 | `50`，范围越界强制为 `2` | `plg_backend_s3/index.go:60-68` |
| `credentials` | S3 | 四级链：AK/SK → AssumeRole → 环境变量 → EC2/ECS IAM | `plg_backend_s3/utils.go:21-47` |
| `branch` | Git | `"master"` | `plg_backend_git/index.go:81` |
| `commit` 消息模板 | Git | `"{action} {filename}"` | 同上 |
| `authorName/Email` | Git | `APP_NAME` / `"none@filestash"` | 同上 |
| `Home` | Local | 三级：`user.Current().HomeDir → os.Getenv("HOME") → os.Getenv("USERPROFILE")` | `plg_backend_local/index.go:60-81` |

**为什么不用"最弱后端交集"**：
- 各后端参数域完全不同（S3 有 region/threads，FTP 有 tls_mode），无法统一比较
- 能力矩阵的维度也不同（Dav 有深度限制，S3 有 Glacier offline），交集结果没意义
- 真正的"能力声明"在 Meta() 中按路径动态返回，而不是 Init 的参数默认值

### 12.5 Home 接口的默认值策略

位置：`server/model/files.go:53-69`

```go
// 策略：有实现就用，没有则退化为 "/"
func GetHome(b IBackend) string {
    if obj, ok := b.(interface{ Home() string }); ok {
        return obj.Home()
    }
    return "/"   // 默认：所有后端的虚拟根目录
}
```

Local 后端自己再做三级 fallback（见 §3.2）。

### 12.6 默认值策略设计权衡

| 设计选择 | 优点 | 缺点 |
|---------|------|------|
| nil=允许（开放默认） | 新增能力字段零侵入；老后端无需改代码即兼容；代码量少 | 容易忘记声明禁用；新后端初版往往过于宽松；权限漏洞风险 |
| 单向收窄（true→false） | 逻辑简单；避免中间件互相冲突 | 无法表达"某些路径放开"的反向授权 |
| 每个后端硬编码参数默认 | 按协议常识最直观 | 跨后端参数统一配置时无一致性保证 |

---

## 13. 深度分析：跨后端文件移动的事务一致性与错误处理路径

结论先行：**所有后端的 Mv 都不具备事务原子性**。除 Local/SFTP 的同分区原子 rename 外，其余都是两阶段 Copy+Delete 或 Cat+Save+Rm，**无回滚机制，不保证一致性**。

> 注意：此处"跨后端"指同一 IBackend 实例内部的跨目录/跨桶移动，不涉及从 S3 拖到 FTP 等跨实例场景（filestash 不支持直接跨实例移动）。

### 13.1 五层一致性等级总览

| 等级 | 实现方式 | 后端 | 原子性 | 失败后果 |
|------|---------|------|--------|---------|
| L1 系统原子 | 系统调用 `rename(2)` | Local | 同分区 ✓ 原子；跨分区 ✗ | 失败 = 0 步，源不动 |
| L2 协议原子 | SFTP/FTP 协议级 Rename | SFTP、FTP | 近似原子 | 失败 = 0 步 |
| L3 文件级 Copy+Delete | CopyObject → DeleteObject（两调） | S3 单文件、WebDAV、GDrive、Dropbox | ✗ 非原子 | Copy 成功但 Delete 失败 → **双份副本** |
| L4 目录级多线程 Copy+Delete | List → N*(Copy→Delete) 并发 + Cancel | S3 目录 | ✗ 非原子 | 部分成功部分失败；已复制的无法回滚 |
| L5 完全模拟 Cat+Save+Rm | Cat 读 → Save 写 → Rm 删 | Dav（CardDAV/CalDAV） | ✗ 非原子 | Save 成功但 Rm 失败 → **双份副本** |
| L6 本地操作+远端副作用 | 本地 rename → git commit → git push | Git | ✗ 非原子 | 本地变了但 push 失败 → **本地/远端不一致** |

### 13.2 L1/L2 原子实现：Local / SFTP / FTP

```go
// Local: server/plugin/plg_backend_local/index.go:127-140
func (this *LocalBackend) Mv(from string, to string) error {
    p1, _ := this.absPath(from)       // SafeOs* 已做路径安全校验
    p2, _ := this.absPath(to)
    return SafeOsRename(p1, p2)
}
```

- `SafeOsRename` → `os.Rename` → POSIX `rename(2)`
- **同分区**：内核保证原子性（要么全成要么全败）
- **跨分区**：`os.Rename` 会返回 `EXDEV`（cross-device link），**不会自动 fallback 为复制删除**，直接把错误抛出给用户

SFTP 的 `SFTPClient.Rename()` 和 FTP 的 `client.Rename(from, to)`（RNFR+RNTO 两阶段 FTP 命令）也是协议级近似原子操作。

### 13.3 L3 单文件 Copy+Delete：S3 / WebDAV / GDrive / Dropbox

以 S3 单文件移动为例，位置：`server/plugin/plg_backend_s3/index.go:435-474`

```go
// CASE 2: Rename/Move a file
input := &s3.CopyObjectInput{...}
_, err := client.CopyObjectWithContext(ctx, input)   // Step 1: 复制
if err != nil {
    return err                                        // Copy 失败 → 安全：源还在，目标不存在
}
_, err = client.DeleteObjectWithContext(ctx, &s3.DeleteObjectInput{...})  // Step 2: 删除源
return err                                            // ⚠ Delete 失败 → 目标已创建，源也还在 → 双份副本
```

**错误处理路径分析**：

| 阶段 | 失败点 | 后果 | 补救 |
|------|--------|------|------|
| Copy 之前 | 权限/网络/源不存在 | 0 步；一致 | 直接返回错误 |
| Copy 成功，Delete 之前 | 网络中断/权限 | 目标有，源有 → **双份** | **无补偿回滚** |
| Delete 执行中 | S3 服务端内部错 | 同上 → **双份** | **无补偿回滚** |
| Delete 成功 | — | 一致 | 正常 |

> **无 DeleteObject 回滚**：Delete 失败时不会回头去 Delete 新复制的目标。用户感知："移动成功了，但原文件还在"。

### 13.4 L4 目录级多线程 Copy+Delete：S3 目录移动

位置：`server/plugin/plg_backend_s3/index.go:475-546`

```
用户 Mv(/dirA/, /dirB/)
   │
   ▼
1. ListObjectsV2Pages 枚举 /dirA/ 下所有 N 个对象
   │
   ▼
2. 把 {源, 目标} key 对推入 jobChan
   │
   ▼
3. threadSize 个 goroutine 并发执行:
   for spath := range jobChan {
       CopyObject(spath[0] → spath[1])    // 复制 1 个对象
         ├─ 成功 → DeleteObject(spath[0])
         │              ├─ 成功 → 继续下一个
         │              └─ 失败 → cancel() + errChan <- err
         └─ 失败 → cancel() + errChan <- err
   }
   │
   ▼
4. 主线程等待所有 goroutine 完成
5. 遍历 errChan → 返回第一个错误
```

**context.Cancel 的作用**：不是"回滚"，只是"阻止新的 goroutine 继续干活"。已经完成 Copy 的 K 个文件不会被反向删除，已经 Delete 完成的更不会被恢复。

**可能出现的所有中间状态**：

| 场景 | N 总文件 | 已 Copy 完成 | 已 Delete 完成 | 状态 |
|------|---------|-------------|---------------|------|
| 全部成功 | 100 | 100 | 100 | ✓ 一致，dirB 有 100 个，dirA 空 |
| List 阶段失败 | 100 | 0 | 0 | ✓ 一致，dirA 仍有 100 个 |
| Copy 中途失败（第 30 个） | 100 | 30 | 0~29 | ⚠ dirB 有 30 个副本，dirA 有 (100 - delete_count) 个 → 数据重复 |
| Delete 中途失败（第 30 个） | 100 | 30 | 29 | ⚠ dirB 有 30 个，dirA 还有 71 个 → 数据重复 |
| 全部 Copy 成功但第 60 个 Delete 失败 | 100 | 100 | 59 | ⚠ dirB 有 100 个，dirA 还有 41 个 → 部分重复 |

**用户感知**：返回错误，但后端数据处于不一致状态。需要用户手工清理重复文件或重试（幂等性：Copy 成功的文件会因 key 已存在或直接覆盖而 OK，Delete 多次删同 key 也 OK）。

### 13.5 L5 完全模拟 Cat+Save+Rm：Dav（CardDAV/CalDAV）

位置：`server/plugin/plg_backend_dav/index.go:199-231`

```go
func (this Dav) Mv(from string, to string) error {
    if filepath.Dir(from) != filepath.Dir(to) {  // 限制：仅同目录内重命名
        return ErrNotValid
    }
    reader, err := this.Cat(from)                 // Step 1: 读源
    if err != nil { return err }
    d, _ := io.ReadAll(reader)

    content := 修改 FN/SUMMARY 字段（文件名变了，VCARD/ICS 内容里的标题也要同步）

    if err = this.Save(to, strings.NewReader(...)); err != nil {  // Step 2: 写目标
        return err
    }
    return this.Rm(from)                          // Step 3: 删源
}
```

**Dav 独有特性**：Step 2 `Save` 也不是覆盖写入，而是"生成随机 URN 写新资源 → 若有旧资源删旧的"（见下 §13.7）。所以 Mv 内部其实是 4 步：Cat → 生成新 URN 写 → Save 内部清理旧资源 → 删源 Mv 的源。

### 13.6 L6 本地原子 + 远端副作用：Git

位置：`server/plugin/plg_backend_git/index.go:260-278`

```go
func (g Git) Mv(from string, to string) error {
    SafeOsRename(fpath, tpath)                     // Step 1: 本地原子 rename
      │
      ├─ 失败 → 直接返回，一致
      │
      ▼
    message := g.git.message("move", from)
    g.git.save(message)                            // Step 2: git add + commit + push
      │
      ├─ w.Add(".")  失败 → 本地已改，git 未暂存 → 不一致
      ├─ w.Commit() 失败 → 本地已改，git 未提交 → 不一致
      └─ repo.Push() 失败 → 本地已改已提交，远端未更新 → 不一致
}
```

**Push 失败的关键不一致**：
- 用户下次重新 Mv/Save 时，**本地工作区还在**，会再次 `git.save()` → 把之前没 push 的 commit 和新的 commit 一起 push
- 这是"最终一致性"而非"强一致性"
- `Close()` 会 `os.RemoveAll(basePath)` 删除本地 clone，下次 Init 重新 clone → 之前未 push 的本地变更**永久丢失**

### 13.7 Dav.Save 的"伪覆盖"保护：先写新 URN 再删旧资源

位置：`server/plugin/plg_backend_dav/index.go:300-342`

CardDAV/CalDAV 的资源 URI 是 URN（如 `.../AG3T9K7P5Q.vcf`），不是用户看到的文件名（如 `张三.vcf`）。所以覆盖保存不是 PUT 同一个 URI，而是：

```
getResourceURI(旧路径) → URI of 已存在的同名文件 (如果有)
   │
   ▼
getCollectionURI(父路径) + RandomString(15) + .vcf/.ics  → 生成全新 URN
   │
   ▼
PUT 新 URN, If-None-Match: *   → 新写（防止覆盖）
   │
   ├─ 失败 → 返回错误，旧文件不动 ✓
   │
   ▼
如果之前查到旧资源存在 → DELETE 旧 URI
   │
   ├─ 失败 → 返回错误，但新文件已写入 → 两份不同 URN 的同一"文件名" ⚠
   │
   ▼
成功
```

> 这层设计是为了避免"PUT 覆盖写中间失败"导致旧内容被破坏。代价是 DELETE 旧 URI 失败时出现 URN 泄漏。

### 13.8 一致性问题的关键洞察

1. **无事务协调器**：整个后端抽象层没有 `BeginTx` / `Commit` / `Rollback` 接口，所有操作都是单向 fire-and-forget。

2. **幂等性是事实上的容错策略**：S3 CopyObject / DeleteObject 都是幂等的，用户重试能修正"双份副本"问题（目标 key 相同就覆盖，Delete 已不存在的 key 不报错）。重试是推荐的错误恢复方式。

3. **Dav 的三态路径映射**：用户可见路径（`/通讯录/张三.vcf`）与服务端 URN（`/users/u1/addrbook/AG3T9K7P5Q.vcf`）是解耦的。Save/Mv 会重映射 URN，这使得 Dav 后端的一致性问题更加隐蔽。

4. **Git 的"最终一致"风险窗口**：从本地 `SafeOsRename` 返回成功到 `git.push` 完成之间，存在不一致窗口。如果此时进程崩溃，且 `Close()` 删除了工作区，变更会永久丢失。

5. **S3 目录移动的数据重复**：CopyObject 是服务端异步的，对于大目录，网络中断时可能已完成上千次 Copy，这些重复数据只能靠用户手工清理或脚本 dedup。

---

## 14. 深度分析：客户端缓存层与服务器端缓存失效协同机制

### 14.1 缓存全景图：6 类独立缓存各司其职

整个后端能力层涉及 **6 类独立缓存**，分布在不同层级，TTL 和失效策略完全不同：

| 缓存 | 类型 | 作用 | TTL | 位置 |
|------|------|------|-----|------|
| `*Cache`（后端连接级） | AppCache | FTP/SFTP/Git/Dav 等长连接复用；S3 region；Tmp chroot 路径 | 默认 5 分钟（或自定义） | 各 backend plugin init() |
| `file_cache`（控制器级下载缓存） | AppCache | Range 请求时把后端非 seekable 流落盘到临时文件，支持多次 seek | 5 分钟（默认） | `ctrl/files.go:37` |
| `chunkedUploadCache`（控制器级上传缓存） | AppCache | TUS 断点续传的 uploader 状态，含 io.Pipe + offset | 24 小时（1440 分钟） | `ctrl/files.go:477` |
| `DavCache`（后端 Ls 结果缓存） | AppCache | CardDAV/CalDAV 的 PROPFIND 结果缓存，避免每次 Ls 重复请求 | 5 分钟（默认） | `plg_backend_dav/index.go:17` |
| `S3Cache`（后端 region 探测缓存） | AppCache | S3 GetBucketLocation 探测结果（按 {bucket, params} 哈希） | 5 分钟（默认） | `plg_backend_s3/index.go:27` |
| HTTP 协商缓存（协议级） | HTTP Header | Last-Modified / If-Modified-Since / ETag / If-None-Match | 由浏览器决定 | `ctrl/files.go` FileCat |

### 14.2 后端连接级缓存 `*Cache`：复用 + 驱逐关闭

以 FTP 缓存为例，位置：`server/plugin/plg_backend_ftp/index.go:19-38`

```go
var FtpCache AppCache

func init() {
    FtpCache = NewAppCache()
    FtpCache.OnEvict(func(key string, value interface{}) {
        if value == nil { return }
        f := value.(*Ftp)
        if f == nil { return }
        // 驱逐前优雅关闭 TCP 连接（等待 in-flight 请求完成）
        f.wg.Wait()
        f.Close()
    })
}
```

**Init 时缓存命中流程**：
```
用户请求携带的 params（全量连接参数哈希）
   │
   ▼
FtpCache.Get(params)
   ├─ 命中 → 返回已有的 Ftp 实例，跳过握手+TLS 协商
   └─ 未命中 → 走三级 TLS 协商 → FtpCache.Set(params, ftpInstance)
```

**显式失效触发点**（只有 FTP 的自动重连会主动 Del）：
```go
// FTP.Execute(): 遇到 421 / EOF 时自动重连
if code == 421 || (code == 0 && err.Error() == "EOF") {
    f.Close()
    FtpCache.Set(f.p, nil)   // 置 nil 触发 OnEvict 关闭旧连接
    b, _ := f.Init(f.p, ...)  // 重新建连
}
```

### 14.3 DavCache：写操作后的主动失效

Dav (CardDAV/CalDAV) 的 Ls 依赖耗时的 PROPFIND 请求，结果缓存 5 分钟。但任何写操作都必须主动失效，否则用户看不到自己刚创建的联系人：

位置：`server/plugin/plg_backend_dav/index.go` 多处

| 操作 | 失效时机 |
|------|---------|
| `Mkdir(path)` | 创建通讯录/日历成功后 → `DavCache.Del(this.params)` |
| `Rm(path)` | 删除资源成功后 → `DavCache.Del(this.params)` |
| `Save(path, reader)` | 写成功后自动隐式失效（URN 重映射，下次 Ls 直接看不到旧 URN） |

**失效粒度**：按整个后端实例 params 失效，而不是按 path。这意味着用户在一个通讯录里新建联系人，另一个通讯录的缓存也被清掉 —— 简单粗暴但正确。

### 14.4 `file_cache`：Range 请求的 Seek 能力补齐

位置：`server/ctrl/files.go:232-358` (FileCat)

问题背景：`IBackend.Cat()` 返回的是 `io.ReadCloser`，不保证能 `Seek`。但浏览器播放音视频、PDF 阅读器需要 HTTP Range（`bytes=start-end`），Range 必须对资源随机访问。

**协同流程**：

```
第 1 次请求（Range: bytes=1000-2000）:
   │
   ├─ file_cache.Get(ctx.Session) → nil（还没缓存）
   │
   ├─ 后端 Cat(path) → io.ReadCloser（不可 seek）
   │
   ├─ 检测到 !io.Seeker → 落到本地临时文件:
   │     tmpPath := /tmp/filestash/file_XXXXX.dat
   │     io.Copy(tmpFile, catReader)  // 全量下载到磁盘
   │     file_cache.Set(ctx.Session, tmpPath)  // key = 会话哈希
   │
   └─ 打开 tmpFile 作为 *os.File（可 seek）→ 读 Range 1000-2000 → 返回给浏览器


第 2 次请求（Range: bytes=5000-6000，同会话同文件）:
   │
   ├─ file_cache.Get(ctx.Session) → hit → tmpPath
   │
   ├─ os.OpenFile(tmpPath) → *os.File（可 seek）
   │
   └─ 直接读 Range 5000-6000 → 不再触发后端 Cat
```

**缓存键设计**：`ctx.Session`（整个会话对象哈希），而不是 path。这意味着**同一个会话同时打开两个文件时会互相覆盖缓存** —— 假设用户一个 tab 放视频，另一个 tab 放大 PDF，Range 请求会反复失效落盘。

**驱逐与清理**（`ctrl/files.go:69-71`）：
```go
file_cache.OnEvict(func(key string, value interface{}) {
    os.RemoveAll(filepath.Join(GetAbsolutePath(TMP_PATH), key))
})
```
5 分钟 TTL 到期后，自动删除临时 `.dat` 文件。

### 14.5 `chunkedUploadCache`：24 小时长 TTL + 驱逐即上传完成

位置：`server/ctrl/files.go:687-700`

```go
chunkedUploadCache = NewAppCache(60*24, 1)  // retention=24h, cleanup=1min
chunkedUploadCache.OnEvict(func(key string, value interface{}) {
    c := value.(*chunkedUpload)
    c.Close()   // 关闭 io.Pipe → 触发后端 Save 完成
})
```

**缓存键**：`{path: "/a/b/c.zip", session: GenerateID(ctx.Session)}` 哈希

**三种失效路径**：
1. **正常完成**：最后一个 PATCH 到达，`offset == totalSize` → `chunkedUploadCache.Del(cacheKey)` → 触发 OnEvict 关闭 Pipe → 后端 Save 返回
2. **用户主动 HEAD 查询发现不存在**：重新 POST 创建时，若缓存里已有旧 uploader，先 `chunkedUploadCache.Del()` 清掉
3. **24 小时超时驱逐**：用户传了一半断网超过 24 小时 → OnEvict 自动 Close Pipe → 后端 Save 用已收到的部分字节落盘（产生截断文件）

### 14.6 HTTP 协商缓存：Last-Modified / ETag

位置：`server/ctrl/files.go` FileCat 多处、`SendSuccessResultWithEtagAndGzip`

```
┌───────────────────────────────────────────────────────────────────┐
│                    缩略图（thumbnail=true）                        │
│                                                                   │
│  Stat → ModTime → Last-Modified 响应头                            │
│         │                                                         │
│         ▼                                                         │
│  下次请求带 If-Modified-Since                                     │
│         │                                                         │
│         ├─ 时间匹配 → 304 Not Modified（不生成缩略图）             │
│         └─ 不匹配 → 重新生成 + 更新 Last-Modified                  │
└───────────────────────────────────────────────────────────────────┘
┌───────────────────────────────────────────────────────────────────┐
│                    JSON API 响应                                   │
│                                                                   │
│  mode + JSON 内容做 hash → ETag 响应头                             │
│         │                                                         │
│         ▼                                                         │
│  下次请求带 If-None-Match                                          │
│         │                                                         │
│         ├─ 匹配 → 304 Not Modified（不返回 JSON body）            │
│         └─ 不匹配 → 返回新内容 + 新 ETag                           │
└───────────────────────────────────────────────────────────────────┘
```

**关键点**：HTTP 层缓存完全独立于后端 `*Cache`。后端 Ls 结果可能 5 分钟前已缓存（旧数据），HTTP 层仍然基于**这次 Ls 返回的旧数据**计算 ETag → 浏览器看到 304 但实际数据可能已过期。这是一个"嵌套缓存一致性"的弱设计。

### 14.7 S3Cache：Bucket Region 按需探测缓存

位置：`server/plugin/plg_backend_s3/utils.go:57-77`

```
用户访问 s3://mybucket/photo.jpg
   │
   ▼
createSession("mybucket"):
   newParams = {bucket: "mybucket", ...所有连接参数...}
   │
   ├─ S3Cache.Get(newParams) 命中 → 直接用缓存的 Region
   │
   └─ 未命中 → GetBucketLocation API 查询
              ├─ 成功 → this.config.Region = 查询结果
              ├─ 失败 → 保持默认 us-east-1
              └─ S3Cache.Set(newParams, this.config.Region)
```

**失效机制**：5 分钟 TTL 自动过期。**没有主动失效** —— Bucket 迁移 Region 的话最多 5 分钟后才会重新探测。

### 14.8 TmpStorage ChrootCache：用户临时目录持久化

位置：`server/plugin/plg_backend_tmp/index.go:18-30`

```go
ChrootCache = NewAppCache(60 * 24 * 30)  // 30 天！
ChrootCache.OnEvict(func(key string, value interface{}) {
    os.RemoveAll(chroot)   // 驱逐 = 删除用户的整个 /tmp/filestash_tmp/{userID}/
})
```

这是最长 TTL 的缓存。30 天不活动的用户，临时目录会被整体删除。

### 14.9 缓存协同的关键洞察

1. **缓存之间无联动**：`DavCache.Del()` 不会触发 HTTP ETag 重算；`file_cache` 的 5 分钟失效与 `FtpCache` 的 5 分钟失效独立计时。没有任何"写操作全局失效"的广播机制。

2. **写操作失效粒度极粗**：Dav 写操作清整个 params 级缓存（而不是 path 级），S3 完全不主动失效 Region 缓存。简单粗暴是设计原则。

3. **会话级 file_cache 的冲突**：缓存键是整个 session 哈希而非 path，同会话多文件 Range 请求互相覆盖 —— 是 bug 还是"够用就行"的有意设计，代码中没有注释。

4. **驱逐副作用**：`*Cache.OnEvict` 不只删内存条目，还会关 TCP 连接、删磁盘文件、完成截断上传。驱逐回调是"资源清理"而不仅是"缓存失效"。

---

## 15. 深度分析：大文件分片上传和断点续传的跨后端兼容

### 15.1 双层上传架构：TUS 协议层 + 后端 Save 能力层

filestash 采用 **双层架构**，把所有后端都兼容到分片上传：

```
┌───────────────────────────────────────────────────────────────────┐
│              TUS 协议层（ctrl/files.go FileSave）                  │
│                                                                   │
│  OPTIONS → 协商能力（creation, checksum: sha1/crc32）              │
│  POST    → 创建 uploader，建立 io.Pipe，启动 goroutine 调用        │
│            后端 Save(path, pipeReader)                             │
│  HEAD    → 查询当前 Upload-Offset / Upload-Length                 │
│  PATCH   → 追加 bytes 到 pipe；offset 对齐校验；可选校验和          │
│  完成    → offset == size → Close Pipe → 后端 Save 返回            │
│                                                                   │
│  存储: chunkedUploadCache (24h TTL) 存 uploader {pipe, offset}    │
└────────────────────────────────────┬──────────────────────────────┘
                                     │ io.Pipe (流式桥接)
                                     ▼
┌───────────────────────────────────────────────────────────────────┐
│              后端 Save 能力层（每个后端自己实现）                    │
│                                                                   │
│  S3:      s3manager.Uploader (SDK 自带分片并发，5MB part 默认)     │
│  FTP:     client.Store(path, reader) — 协议级流式写入             │
│  SFTP:    SFTPClient.OpenFile + io.Copy — 协议级流式写入          │
│  WebDAV:  HTTP PUT + body — 协议级流式写入                        │
│  Local:   os.OpenFile + io.Copy — 本地流式写入                    │
│  Git:     本地 Save → git add + commit + push                     │
│  Dav:     PUT 新 URN (If-None-Match:*) → DELETE 旧 URN            │
│  GDrive/Dropbox: SDK 文件 Create 接口                             │
└───────────────────────────────────────────────────────────────────┘
```

**核心设计思路**：TUS 层对后端完全屏蔽"分片"概念 —— 后端看到的就是一个普通的 `Save(path, io.Reader)`，Reader 另一端由 TUS 的 PATCH 请求源源不断喂数据。**后端不需要任何分片能力即可支持断点续传**。

### 15.2 TUS 协议状态机：完整时序

位置：`server/ctrl/files.go:479-670`

```
前端                                    服务端
  │                                       │
  │  OPTIONS /api/files/save              │
  │    Tus-Resumable: 1.0.0               │
  │ ─────────────────────────────────────►│
  │                                       │ 协商支持：creation, checksum
  │◄───────────────────────────────────── │ 算法：sha1, crc32
  │                                       │
  │  POST /api/files/save?path=/a.zip     │
  │    Tus-Resumable: 1.0.0               │
  │    Upload-Length: 1073741824          │  ← 1GB
  │ ─────────────────────────────────────►│
  │                                       │
  │                                       │ 1. 重新 Init 后端实例（新 context）
  │                                       │ 2. createChunkedUploader(b.Save, path, 1GB):
  │                                       │    - r, w := io.Pipe()
  │                                       │    - go func() { done <- b.Save(path, r) }()
  │                                       │ 3. chunkedUploadCache.Set(key, uploader)
  │                                       │    key = hash({path, session})
  │◄───────────────────────────────────── │
  │  201 Created                          │
  │  Location: /api/files/save?path=...   │
  │                                       │
  │  PATCH /api/files/save?path=/a.zip    │
  │    Tus-Resumable: 1.0.0               │
  │    Upload-Offset: 0                   │
  │    Upload-Checksum: sha1 XXXXX        │
  │    Content-Type: application/offset+octet-stream
  │    Body: [bytes 0~5MB]                │
  │ ─────────────────────────────────────►│
  │                                       │
  │                                       │ 1. 从 cache 取出 uploader
  │                                       │ 2. 校验 Upload-Offset == uploader.offset
  │                                       │ 3. io.Copy(uploader.stream, body)
  │                                       │ 4. 校验 sha1/crc32
  │                                       │ 5. offset += 5MB
  │◄───────────────────────────────────── │
  │  204 No Content                       │
  │  Upload-Offset: 5242880               │
  │                                       │
  │    ... (反复 PATCH, 每片 5MB) ...      │
  │                                       │
  │  PATCH ... 最后一片 (offset=1068MB)   │
  │ ─────────────────────────────────────►│
  │                                       │ 检测到 offset == 1GB
  │                                       │   uploader.stream.Close()
  │                                       │   等待 <-done（后端 Save 返回）
  │                                       │   chunkedUploadCache.Del(key)
  │◄───────────────────────────────────── │
  │  204 No Content                       │
  │                                       │
  │                                       │
  │  === 如果中途断网，前端恢复流程 ===      │
  │                                       │
  │  HEAD /api/files/save?path=/a.zip     │
  │    Tus-Resumable: 1.0.0               │
  │ ─────────────────────────────────────►│
  │                                       │ chunkedUploadCache.Get(key)
  │                                       │   → uploader.Meta(): {offset=100MB, size=1GB}
  │◄───────────────────────────────────── │
  │  204 No Content                       │
  │  Upload-Offset: 104857600             │
  │  Upload-Length: 1073741824            │
  │                                       │
  │  PATCH ... (从 100MB 续传)            │
  │ ─────────────────────────────────────►│  ...继续
```

### 15.3 兼容性对比：各后端 Save 与分片的适配

| 后端 | Save 实现方式 | 流式写入 | 分片由谁处理 | 最大文件限制 | 失败原子性 |
|------|-------------|---------|-------------|------------|----------|
| **S3** | `s3manager.Uploader`（aws-sdk-go） | ✓ | **SDK 自动分片**（默认 5MB part，并发 5） | 5TB（S3 限制） | 非原子（MPU 失败残留 parts，SDK 自动 Abort） |
| **Local** | `os.OpenFile` + `io.Copy` | ✓ | 操作系统 page cache | 磁盘限制 | ✓ 原子（O_TRUNC 覆盖） |
| **FTP** | `client.Store(path, reader)` | ✓ | FTP 协议数据连接流式传输 | 服务端依赖 | 非原子（半写入） |
| **SFTP** | `SFTPClient.OpenFile` + `io.Copy` | ✓ | SSH channel 流式传输 | 服务端依赖 | 非原子（半写入） |
| **WebDAV** | `HTTP PUT` body streamed | ✓ | HTTP chunked encoding | 服务端依赖 | 非原子（半写入） |
| **Git** | Local Save → git add → commit → push | ✓ 本地流式 | 本地文件系统 + Git | Git 仓库限制 | 非原子（§13.6 所述的最终一致） |
| **Dav** (Card/Cal) | PUT 新 URN（If-None-Match:*）→ DELETE 旧 | ✓（小文件） | HTTP 流式 | URN 映射机制限制 | 非原子（DELETE 失败双份副本） |
| **GDrive / Dropbox** | SDK 文件 Create | ✓ | SDK 内部 | 服务端限制 | 非原子 |

### 15.4 S3 后端的特殊地位：双层分片

S3 后端存在 **双层分片**，容易混淆：

```
第 1 层（TUS 层，服务端可控）:
  PATCH 1 (5MB) → Pipe → PATCH 2 (5MB) → Pipe → ...
  控制方：filestash 前端（chunk size 由 JS 决定）
  作用：断点续传

第 2 层（S3 SDK 层，服务端黑盒）:
  PipeReader → s3manager.Uploader →
    ├─ 缓冲 5MB → UploadPart #1
    ├─ 缓冲 5MB → UploadPart #2 （并发）
    └─ ...
    └─ 所有 Part 完成 → CompleteMultipartUpload
  控制方：aws-sdk-go（默认 PartSize=5MB, Concurrency=5）
  作用：绕开 S3 单 PUT 5GB 限制，支持大文件 + 并发加速
```

**双层分片的冲突**：
- TUS 层每片 5MB + SDK 层每片 5MB → 对齐
- 但如果前端修改 TUS chunk size（比如 1MB），SDK 内部仍然要缓冲到 5MB 才发 UploadPart，可能出现内存堆积
- `s3manager` 会自动判断文件大小：< 5MB 走普通 PutObject，≥ 5MB 自动走 MultipartUpload

### 15.5 后端无分片能力时的兼容：流式桥接的价值

**以 FTP 为例**：FTP `STOR` 命令根本没有分片概念，也没有断点续传（REST 命令扩展不普遍支持）。但通过 TUS + io.Pipe 架构：

```
前端 TUS PATCH (5MB 分片 1)
   │
   ▼
io.PipeWriter ← 写入 5MB
io.PipeReader → FTP client.Store(path, pipeReader)
   │                  │
   │                  └─ FTP STOR 持续读，有数据就发，没数据就等（阻塞）
   │
前端 TUS PATCH (5MB 分片 2，1 小时后来的，用户断网刚恢复)
   │
   ▼
io.PipeWriter ← 再写 5MB
   │
   ▼
FTP client.Store 被唤醒，继续发数据
   ...
```

**关键前提**：FTP 控制连接 + 数据连接在 24 小时（`chunkedUploadCache` TTL）内不被服务端踢掉。如果被踢，FTP `Execute` 有自动重连机制（§6.5），但 `Store` 流已断 → 只能从头开始。

### 15.6 错误处理路径与边界场景

位置：`server/ctrl/files.go:554-667`

| 错误场景 | 处理方式 | 后果 |
|---------|---------|------|
| HEAD 查询时 cache 中找不到 uploader | 返回 404 NotFound | 前端只能重新 POST 创建（从头传） |
| PATCH 时 cache miss | 返回 409 Conflict | 同上 |
| PATCH 的 Upload-Offset ≠ 服务端 offset | 返回 405 ErrNotValid | 前端应 HEAD 查询后从正确位置重传 |
| Upload-Checksum (sha1/crc32) 不匹配 | 返回 460 Checksum Mismatch | 这一片无效，重新传（offset 不变） |
| newOffset > totalSize | Close Pipe + Del cache + 403 | 异常，上传终止，远端可能有截断文件 |
| 24 小时 TTL 驱逐 | OnEvict → `c.Close()` → 关闭 Pipe | 远端 Save 落盘的是当前已接收部分（截断文件） |
| 后端 Save 返回错误 | Close 返回错误 → 从 PATCH 的 204 响应中体现？不，等 Close 时 PATCH 已经走了 | **错误丢失**：当前实现中 `uploader.Next()` 只看 Copy 错误，Save 的错误要到 Close 才暴露；如果是最后一个 PATCH，Close 错误会返回 405；否则静默存在 |

### 15.7 分片上传的关键洞察

1. **TUS 层是"万能胶水"**：通过 io.Pipe 把"前端分片异步喂数据"和"后端需要一个连续 Reader"桥接起来，对后端零侵入 —— 所有后端天然支持断点续传，不需要各自实现。

2. **S3 的双层分片冗余**：TUS 层分片+缓存实现断点续传，SDK 层分片实现大文件并发上传。两层各自独立，没有协调，存在边界条件（TUS 分片小于 SDK PartSize 时 SDK 内部缓冲堆积）。

3. **无服务端持久化**：`chunkedUploadCache` 纯内存（带 TTL），进程重启所有断点续传状态丢失 → 用户必须重新上传。

4. **长连接依赖**：FTP/SFTP 等长连接协议的 Store 在 Pipe 阻塞期间必须保持连接不被服务端超时关闭。FTP 自动重连只能在操作之间生效，不能在一个 Store 流中间恢复。

5. **错误丢失风险**：`chunkedUpload.done` channel 中的后端 Save 错误，只有最后一片 PATCH 触发 Close 时才会被检查；非最后一片时，即使后端 Save 已出错（如磁盘满、权限拒绝），后续 PATCH 仍然不断写 Pipe 并返回 204，直到 Close 才暴露问题 —— 用户传完了才发现失败。
