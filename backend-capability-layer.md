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
