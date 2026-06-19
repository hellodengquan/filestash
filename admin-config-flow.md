# Admin 后台配置注入链路详解

本文档详细说明 Filestash 项目中 Admin 后台与配置注入的完整链路，包括配置 Schema 定义、热更新机制和敏感字段处理。

## 目录

- [1. 配置 Schema 定义](#1-配置-schema-定义)
- [2. 配置热更新机制](#2-配置热更新机制)
- [3. 敏感字段处理](#3-敏感字段处理)
- [4. Admin 后台配置注入链路](#4-admin-后台配置注入链路)
- [5. 配置版本迁移与 Schema 演化兼容](#5-配置版本迁移与-schema-演化兼容)
- [6. 热更新失败回滚机制](#6-热更新失败回滚机制)
- [7. 多 Admin 并发冲突解决路径](#7-多-admin-并发冲突解决路径)
- [8. 配置变更通知给在线 Admin 的机制](#8-配置变更通知给在线-admin-的机制)
- [9. 敏感字段在审计场景下的明文恢复路径](#9-敏感字段在审计场景下的明文恢复路径)
- [10. 配置变更在审计记录中的字段对比展示路径](#10-配置变更在审计记录中的字段对比展示路径)
- [11. 多层级配置合并冲突时的优先级决策](#11-多层级配置合并冲突时的优先级决策)
- [12. 关键代码位置索引](#12-关键代码位置索引)

---

## 1. 配置 Schema 定义

### 1.1 核心数据结构

配置系统的核心数据结构定义在 `server/common/config.go` 中，采用**树形嵌套结构**：

```
Configuration
├── Form []Form          # 配置表单分组
│   └── Form
│       ├── Title string    # 分组名称
│       ├── Form []Form     # 子分组（嵌套）
│       └── Elmnts []FormElement  # 配置项列表
└── Conn []map[string]any   # 后端连接配置
```

`FormElement` 是单个配置项的定义，包含以下关键字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `Name` | string | 配置项名称（作为 JSON key） |
| `Type` | string | 配置类型（text/number/password/boolean/select/bcrypt/enable/hidden/long_text 等） |
| `Default` | interface{} | 默认值 |
| `Value` | interface{} | 当前值 |
| `Required` | bool | 是否必填 |
| `ReadOnly` | bool | 是否只读 |
| `Pattern` | string | 正则校验模式 |
| `Opts` | []string | 下拉选项（select 类型） |
| `Target` | []string | 联动目标（enable 类型控制其他字段显隐） |
| `Description` | string | 描述说明 |

### 1.2 Schema 定义方式

配置 Schema 有两种定义方式：

#### 方式一：核心配置硬编码

在 `NewConfiguration()` 函数中预定义核心配置分组：

- **general**：通用设置（端口、主机名、密钥、编辑器、上传配置等）
- **features**：功能开关（API、分享、保护机制等）
- **log**：日志配置
- **email**：邮件服务配置
- **auth**：认证配置（admin 密码）

代码位置：`server/common/config.go:62-143`

#### 方式二：插件动态注册

插件可通过 `Schema()` 方法动态扩展配置：

```go
// 示例：plg_editor_codemirror/config.go
Config.Get("features.collaborative.enable").Schema(func(f *FormElement) *FormElement {
    f.Name = "enable"
    f.Type = "enable"
    f.Description = "Enable/Disable collaborative editing"
    f.Default = true
    return f
}).Bool()
```

当访问的配置路径不存在时，系统会自动创建 `hidden` 类型的配置项，插件通过 `Schema()` 方法可以将其"提升"为正式配置项。

### 1.3 配置访问方式

使用**点分隔路径**访问配置项，通过 `Config.Get("path.to.key")` 获取 `ConfigElement`，然后调用类型转换方法：

```go
// 读取配置
Config.Get("general.secret_key").String()
Config.Get("general.upload_pool_size").Int()
Config.Get("features.share.enable").Bool()

// 设置配置
Config.Get("general.host").Set("files.example.com")
```

配置系统内部使用 `sync.Map` 作为缓存，加速路径查找。

---

## 2. 配置热更新机制

### 2.1 热更新流程

配置支持**运行时热更新**，无需重启服务。完整流程如下：

```
前端修改配置
    ↓
POST /admin/api/config
    ↓
PrivateConfigUpdateHandler (server/ctrl/config.go)
    ├─ SaveConfig(b)  → 写入文件（含加密处理）
    └─ Config.Load()  → 重新加载配置到内存
        ├─ 从文件读取配置（config_state.go:LoadConfig）
        ├─ 解密敏感字段
        ├─ Hydration: 将 JSON 数据映射到 Form 结构
        ├─ 清空缓存 (cache.Clear())
        ├─ 更新日志级别 (Log.SetVisibility)
        └─ 触发 OnConfig 钩子 (Hooks.Get.OnConfig())
```

### 2.2 配置加载 (Load)

`Configuration.Load()` 方法负责将磁盘上的 JSON 配置文件加载到内存结构中：

1. **读取文件**：调用 `LoadConfig()` 读取 `config.json`（自动解密敏感字段）
2. **提取连接**：解析 `connections` 数组到 `Config.Conn`
3. **扁平化 JSON**：将嵌套 JSON 转换为 `path.key → value` 的扁平 map
4. **Hydration**：遍历扁平 map，通过 `Get(path)` 找到对应的 `FormElement`，设置 `Value`
5. **清空缓存**：`cache.Clear()` 使缓存失效
6. **更新日志**：根据新配置设置日志级别
7. **触发钩子**：调用所有注册的 `OnConfig` 回调函数

代码位置：`server/common/config.go:186-219`

### 2.3 配置保存 (Save)

`Configuration.Save()` 方法将内存配置持久化到磁盘：

1. **序列化**：通过 `formToJSON()` 将 Form 树序列化为 JSON（只包含 Value）
2. **合并 connections**：将 `Conn` 数组添加到 JSON 中
3. **美化输出**：`PrettyPrint()` 格式化 JSON
4. **加密保存**：调用 `SaveConfig()` 写入文件（自动加密敏感字段）

代码位置：`server/common/config.go:262-284`

### 2.4 配置变更钩子 (OnConfig)

插件可以注册配置变更回调，在配置热更新时收到通知：

```go
// 注册回调
Hooks.Register.OnConfig(func() {
    // 配置变更后的处理逻辑
})

// 触发时机：Config.Load() 末尾
for _, fn := range Hooks.Get.OnConfig() {
    fn()
}
```

代码位置：`server/common/plugin.go:286-294`

---

## 3. 敏感字段处理

### 3.1 三层敏感字段处理机制

Filestash 对敏感字段有三层处理机制：

| 层级 | 处理方式 | 适用场景 |
|------|----------|----------|
| 存储层 | AES-GCM 加密 | 配置文件中的敏感字段 |
| 传输层 | bcrypt 哈希 / 掩码 | admin 密码、分享密码 |
| 展示层 | 密码输入框 / 只读 | 前端表单展示 |

### 3.2 存储层：配置文件加密

在 `server/common/config_state.go` 中定义了需要加密的配置路径：

```go
var configKeysToEncrypt []string = []string{
    "middleware.identity_provider.params",
    "middleware.attribute_mapping.params",
}
```

**加密流程**（保存时）：
1. 读取配置 JSON 字符串
2. 遍历 `configKeysToEncrypt` 列表
3. 对每个路径的值使用 AES-GCM 加密
4. 加密密钥：`Hash(CONFIG_SECRET, 16)`，其中 `CONFIG_SECRET` 环境变量优先，否则派生于 `general.secret_key`
5. 将加密后的值写回 JSON
6. 写入配置文件

**解密流程**（加载时）：
1. 读取配置文件
2. 初始化密钥派生
3. 遍历 `configKeysToEncrypt` 列表
4. 对每个路径的密文使用 AES-GCM 解密
5. 将明文值写回内存配置

代码位置：`server/common/config_state.go:24-113`

### 3.3 传输层：密码哈希与掩码

#### bcrypt 类型（admin 密码）

Admin 密码使用 bcrypt 哈希存储，**永远不以明文形式存在**：

- **设置密码**：前端使用 `bcrypt.js` 对密码进行哈希（`ctrl_setup.js:91`）
- **存储内容**：配置文件中存储的是 bcrypt 哈希值（60 字符）
- **验证密码**：后端使用 `bcrypt.CompareHashAndPassword()` 验证（`server/ctrl/admin.go:60`）
- **前端展示**：`bcrypt` 类型渲染为**只读**密码输入框（`public/assets/components/form.js:191-203`）

#### PASSWORD_DUMMY 掩码

在某些场景（如分享链接）中，密码会被替换为占位符 `{{PASSWORD}}`：

```go
const PASSWORD_DUMMY = "{{PASSWORD}}"

// 序列化时掩码
func (s *Share) MarshalJSON() ([]byte, error) {
    p := Share{
        Password: func(pass *string) *string {
            if pass != nil {
                return NewString(PASSWORD_DUMMY)
            }
            return nil
        }(s.Password),
        // ...
    }
    return json.Marshal(p)
}

// 保存时识别掩码，保留原值
if *p.Password == PASSWORD_DUMMY {
    if s, err := ShareGet(p.Id); err != nil {
        p.Password = s.Password
    }
}
```

代码位置：`server/common/types.go:178,206-228`，`server/model/share.go:87-90`

### 3.4 展示层：前端密码输入

前端对不同类型的密码字段有不同的渲染方式：

1. **password 类型**：普通密码输入框，带眼睛图标可切换明/密文
2. **bcrypt 类型**：只读密码输入框（用于 admin 密码展示）
3. **自动完成禁用**：密码字段设置 `autocomplete="off"`，并禁用密码管理器（LastPass/1Password/Bitwarden 等）

代码位置：`public/assets/components/form.js:152-178`（password 类型），`public/assets/components/form.js:191-203`（bcrypt 类型）

### 3.5 密钥派生

系统使用 `general.secret_key` 作为主密钥，通过 `InitSecretDerivate()` 派生出多个子密钥：

- `SECRET_KEY_DERIVATE_FOR_ADMIN`：admin session 加密
- `SECRET_KEY_DERIVATE_FOR_PROOF`：配置文件加密
- 等等

初始化时机：
1. 配置加载时（`config_state.go:45`）
2. 配置初始化后（`config.go:259`）

代码位置：`server/common/constants.go:72-73`

---

## 4. Admin 后台配置注入链路

### 4.1 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                        前端 (Admin 后台)                      │
│                                                              │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐ │
│  │ ctrl_setup.js│     │ctrl_settings │     │ ctrl_storage │ │
│  │  (设置向导)   │     │  (设置页面)   │     │  (存储配置)   │ │
│  └──────┬───────┘     └──────┬───────┘     └──────┬───────┘ │
│         │                    │                    │         │
│         └────────────────────┼────────────────────┘         │
│                              │                              │
│                    ┌─────────▼─────────┐                    │
│                    │   model_config.js │                    │
│                    │  (admin 配置模型)  │                    │
│                    └─────────┬─────────┘                    │
└──────────────────────────────┼──────────────────────────────┘
                               │ HTTP
┌──────────────────────────────┼──────────────────────────────┐
│                        后端 (Go Server)                       │
│                              │                              │
│                    ┌─────────▼─────────┐                    │
│                    │  /admin/api/config│                    │
│                    │   (AdminOnly)     │                    │
│                    └─────────┬─────────┘                    │
│                              │                              │
│              ┌───────────────┴───────────────┐              │
│              │                               │              │
│    ┌─────────▼─────────┐         ┌──────────▼──────────┐   │
│    │ PrivateConfigHandler│         │PrivateConfigUpdate │   │
│    │     (GET)          │         │      (POST)         │   │
│    └─────────┬─────────┘         └──────────┬──────────┘   │
│              │                               │              │
│              │                        ┌──────▼───────┐      │
│              │                        │  SaveConfig  │      │
│              │                        │  (写入磁盘)   │      │
│              │                        └──────┬───────┘      │
│              │                               │              │
│              └───────────────┬───────────────┘              │
│                              │                              │
│                    ┌─────────▼─────────┐                    │
│                    │  Config (内存)     │                    │
│                    │  Configuration{}   │                    │
│                    └─────────┬─────────┘                    │
│                              │                              │
│                    ┌─────────▼─────────┐                    │
│                    │  Hooks.OnConfig   │                    │
│                    │  (配置变更通知)    │                    │
│                    └───────────────────┘                    │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 前端链路详解

#### 4.2.1 配置获取

**入口**：`public/assets/pages/adminpage/model_config.js`

```javascript
const config$ = isSaving$.pipe(
    rxjs.filter((loading) => !loading),
    rxjs.switchMapTo(ajax({
        url: "admin/api/config",  // GET 请求
        method: "GET",
        responseType: "json"
    })),
    rxjs.map((res) => res.responseJSON.result),
    rxjs.shareReplay(1),  // 缓存最近一次结果
);
```

特点：
- 使用 RxJS 响应式编程
- `shareReplay(1)` 实现多订阅者共享同一份数据
- 保存期间暂停获取新配置

#### 4.2.2 配置保存

**保存管道**：`public/assets/pages/adminpage/model_config.js`

```javascript
export function save() {
    return rxjs.pipe(
        rxjs.tap(() => isSaving$.next(true)),  // 标记保存中
        rxjs.debounceTime(800),                // 防抖 800ms
        rxjs.mergeMap((formData) => ajax({
            url: "admin/api/config",           // POST 请求
            method: "POST",
            responseType: "json",
            body: formData,
        })),
        rxjs.tap(() => isSaving$.next(false)), // 保存完成
        rxjs.catchError((err) => {
            isSaving$.next(false);
            return rxjs.throwError(err);
        }),
    );
}
```

特点：
- **防抖保存**：800ms 防抖，避免频繁请求
- **状态管理**：通过 `isSaving$` 管理保存状态
- **错误处理**：保存失败时重置状态

#### 4.2.3 设置页面 (ctrl_settings.js)

设置页面的工作流程：

1. **加载配置**：调用 `getAdminConfig()` 获取配置
2. **过滤显示**：`reshapeConfigBeforeDisplay` 移除 `constant`、`middleware`、`connections`
3. **生成表单**：使用 `createForm()` 根据 schema 动态生成表单
4. **监听变更**：`useForm$()` 监听所有 input 的 input 事件
5. **防抖保存**：250ms 防抖后应用变更，再走 800ms 保存防抖
6. **保存前重组**：`reshapeConfigBeforeSave` 将 middleware 和 connections 加回配置

```javascript
// 保存前的配置重组逻辑
const reshapeConfigBeforeSave = rxjs.pipe(
    rxjs.mergeMap((configWithMissingKeys) => getAdminConfig().pipe(
        rxjs.first(),
        rxjs.map((config) => {
            configWithMissingKeys["middleware"] = config["middleware"];
            return configWithMissingKeys;
        }),
        formObjToJSON$(),  // 将表单结构转换为纯 JSON
    )),
    rxjs.mergeMap((adminConfig) => getConfig().pipe(
        rxjs.first(),
        rxjs.map((publicConfig) => {
            adminConfig["connections"] = publicConfig["connections"];
            return adminConfig;
        }),
    )),
);
```

代码位置：`public/assets/pages/adminpage/ctrl_settings.js:78-97`

#### 4.2.4 表单项渲染

`public/assets/lib/form.js` + `public/assets/components/form.js` 负责根据 schema 渲染表单：

- **非叶子节点**：渲染为分组（fieldset / 自定义分组）
- **叶子节点**：根据 `type` 渲染不同的输入控件
- **enable 类型**：特殊的开关类型，可通过 `target` 控制其他字段的显隐

### 4.3 后端链路详解

#### 4.3.1 路由注册

配置相关的 API 路由定义在 `server/routes.go` 中：

| 路由 | 方法 | 处理器 | 权限 | 说明 |
|------|------|--------|------|------|
| `/admin/api/config` | GET | `PrivateConfigHandler` | AdminOnly | 获取完整配置 |
| `/admin/api/config` | POST | `PrivateConfigUpdateHandler` | AdminOnly | 更新配置 |
| `/api/config` | GET | `PublicConfigHandler` | 公开 | 获取公开配置 |

代码位置：`server/routes.go:41-42,97`

#### 4.3.2 获取完整配置 (GET)

`PrivateConfigHandler` 直接返回 `Config` 对象的 JSON 序列化：

```go
func PrivateConfigHandler(ctx *App, res http.ResponseWriter, req *http.Request) {
    SendSuccessResult(res, &Config)
}
```

`Configuration.MarshalJSON()` 自定义序列化逻辑，会额外添加 `constant` 分组（包含运行时信息如 user、license）。

代码位置：`server/ctrl/config.go:11-13`，`server/common/config.go:488-504`

#### 4.3.3 更新配置 (POST)

`PrivateConfigUpdateHandler` 处理配置更新：

```go
func PrivateConfigUpdateHandler(ctx *App, res http.ResponseWriter, req *http.Request) {
    b, _ := io.ReadAll(req.Body)  // 读取请求体
    if err := SaveConfig(b); err != nil {  // 保存到文件（含加密）
        SendErrorResult(res, err)
        return
    }
    Config.Load()  // 重新加载到内存
    SendSuccessResult(res, nil)
}
```

**关键点**：
1. 直接将前端发来的 JSON 写入文件（不经过内存结构）
2. 保存后立即调用 `Config.Load()` 热更新内存配置
3. 内存配置更新后自动触发 `OnConfig` 钩子

代码位置：`server/ctrl/config.go:15-23`

#### 4.3.4 公开配置导出

`PublicConfigHandler` 返回经过滤的公开配置：

```go
func PublicConfigHandler(ctx *App, res http.ResponseWriter, req *http.Request) {
    cfg := Config.Export()  // 导出公开字段
    SendSuccessResultWithEtagAndGzip(res, req, cfg)
}
```

`Config.Export()` 只暴露必要的前端配置（如编辑器、上传配置、功能开关等），不包含敏感信息。

代码位置：`server/ctrl/config.go:25-28`，`server/common/config.go:286-362`

### 4.4 配置初始化流程

服务启动时的配置初始化流程：

```
main.go
  ↓
InitConfig()
  ├─ NewConfiguration()  # 创建默认配置结构（schema）
  ├─ Config.Load()       # 从文件加载配置（解密敏感字段）
  │   └─ LoadConfig()    # 读取 + 解密
  └─ Config.Initialise() # 初始化处理
      ├─ 从环境变量 ADMIN_PASSWORD 设置 admin 密码
      ├─ 从环境变量 APPLICATION_URL 设置 host
      ├─ 自动生成 secret_key（如果为空）
      ├─ 保存配置（如果有变更）
      └─ InitSecretDerivate()  # 初始化密钥派生
```

代码位置：`server/common/config.go:53-60`，`server/common/config.go:241-260`

---

## 5. 配置版本迁移与 Schema 演化兼容

### 5.1 核心设计理念："惰性迁移 + 前向兼容"

Filestash 没有采用显式的版本号迁移脚本，而是采用**"惰性迁移 + 前向兼容"**的设计哲学。所有兼容逻辑都是"代码即迁移"，在代码执行过程中动态完成 schema 演化。

### 5.2 自动创建机制（前向兼容）

**挂载点**：`Configuration.Get()` 方法中的 `traverse` 递归函数

```go
// server/common/config.go:369-397
var traverse func(forms *[]Form, path []string) *FormElement
traverse = func(forms *[]Form, path []string) *FormElement {
    // ...
    // 2) `formElement` does not exist, let's create it.
    (*forms)[i].Elmnts = append(currentForm.Elmnts, 
        FormElement{Name: path[1], Type: "hidden"})  // ← 自动创建
    return &(*forms)[i].Elmnts[len(currentForm.Elmnts)]
    // ...
    // append a new `form` if the current key doesn't exist
    *forms = append(*forms, Form{Title: path[0]})  // ← 自动创建分组
    return traverse(forms, path)
}
```

**工作原理**：
1. 当新版本代码访问 `Config.Get("new.feature.enable")` 时
2. 如果路径不存在，系统自动创建 `Type: "hidden"` 的配置项
3. 旧版本配置文件中不存在的字段，在新版本中可以安全访问
4. 自动创建的配置项默认值为 `nil`，读取时会返回 `FormElement.Default`

**兼容保证**：新版本代码可以无错误地读取旧版本配置文件。

代码位置：`server/common/config.go:385-396`

### 5.3 Schema 提升机制（插件扩展）

**挂载点**：`ConfigElement.Schema()` 方法

```go
// server/common/config.go:405-409
func (this *ConfigElement) Schema(fn func(*FormElement) *FormElement) *ConfigElement {
    fn(this.currentElement)  // ← 插件修改元数据
    this.cfg.cache.Clear()    // ← 清除缓存
    return this
}
```

**典型用法**：插件在初始化时将自动创建的 `hidden` 配置项"提升"为正式配置项：

```go
// 插件 init() 函数中调用
Config.Get("features.collaborative.enable").Schema(func(f *FormElement) *FormElement {
    f.Name = "enable"
    f.Type = "enable"           // ← 从 hidden 改为 enable
    f.Description = "Enable/Disable collaborative editing"
    f.Default = true
    return f
}).Bool()
```

**迁移时机**：插件加载时（服务启动或插件初始化）

代码位置：`server/common/config.go:405-409`，示例见 `server/plugin/plg_editor_codemirror/config.go:7-18`

### 5.4 默认值回退机制

**挂载点**：`ConfigElement.Interface()` 方法

```go
// server/common/config.go:475-486
func (this *ConfigElement) Interface() interface{} {
    // ...
    if el.Value == nil {
        return el.Default  // ← 值为 nil 时返回默认值
    }
    return el.Value
}
```

**工作原理**：
- 旧配置文件中不存在的字段，`Value` 为 `nil`
- 读取时自动回退到 `Default` 值
- 保证即使配置文件中没有该字段，系统也能正常工作

代码位置：`server/common/config.go:475-486`

### 5.5 环境变量覆盖机制

**挂载点**：`Configuration.Initialise()` 方法

```go
// server/common/config.go:241-260
func (this *Configuration) Initialise() {
    shouldSave := false
    if env := os.Getenv("ADMIN_PASSWORD"); env != "" {
        shouldSave = true
        this.Get("auth.admin").Set(env)  // ← 环境变量覆盖
    }
    if env := os.Getenv("APPLICATION_URL"); env != "" {
        shouldSave = true
        _ = this.Get("general.host").Set(env).String()
    }
    if this.Get("general.secret_key").String() == "" {
        shouldSave = true
        key := RandomString(16)          // ← 自动生成缺失字段
        this.Get("general.secret_key").Set(key)
    }
    if shouldSave {
        this.Save()  // ← 保存迁移后的配置
    }
    InitSecretDerivate(this.Get("general.secret_key").String())
}
```

**迁移时机**：服务启动时（`InitConfig()` → `Initialise()`）

**兼容场景**：
- 从环境变量注入配置（容器化部署常用）
- 自动生成缺失的必填字段（如 `secret_key`）
- 迁移完成后自动保存到磁盘

代码位置：`server/common/config.go:241-260`

### 5.6 泛型默认值函数

**挂载点**：`defaultValue[T]()` 泛型函数

```go
// server/common/config.go:506-522
func defaultValue[T string | int | bool](dval T, envName string) T {
    if val := os.Getenv(envName); val != "" {
        // 类型转换...
        return any(val).(T)  // ← 环境变量优先
    }
    return dval  // ← 编译时默认值
}
```

**用法示例**：
```go
FormElement{Name: "port", Type: "number", 
    Default: defaultValue(8334, "FILESTASH_PORT"), ...}
```

代码位置：`server/common/config.go:506-522`

### 5.7 配置数据迁移挂载点汇总

| 挂载点 | 触发时机 | 用途 |
|--------|----------|------|
| `Get()` traverse 自动创建 | 访问不存在的配置时 | 前向兼容，自动创建缺失字段 |
| `Schema()` 方法 | 插件初始化时 | 插件扩展配置 schema |
| `Interface()` 默认值回退 | 读取配置时 | 缺失字段返回默认值 |
| `Initialise()` 方法 | 服务启动时 | 环境变量注入、自动生成必填字段 |
| `OnConfig` 钩子 | 配置热更新后 | 插件响应配置变更 |
| `config_state.go` 可替换 | 构建时 | 替换整个配置存储层（S3、自定义加密等） |

> **特殊说明**：`config_state.go` 文件头部有警告，说明该文件可以在构建时被插件生成器替换，以支持 S3 存储、自定义加密等场景。这是最高层级的兼容扩展点。

代码位置：`server/common/config_state.go:3-14`

---

## 6. 热更新失败回滚机制

### 6.1 现状分析："部分回滚 + 无完整回滚"

Filestash 的热更新没有完整的事务性回滚机制，而是采用**"分段错误处理"**的策略。不同阶段的失败有不同的处理方式。

### 6.2 热更新的三个阶段

```
前端发送 JSON
    ↓
阶段 1: SaveConfig(b) → 写入磁盘
    ↓ (成功)
阶段 2: Config.Load() → 重新加载到内存
    ↓ (成功)
阶段 3: Hooks.OnConfig() → 通知插件
```

### 6.3 各阶段失败处理

#### 阶段 1：写入磁盘失败

**代码**：`server/ctrl/config.go:15-23`

```go
func PrivateConfigUpdateHandler(ctx *App, res http.ResponseWriter, req *http.Request) {
    b, _ := io.ReadAll(req.Body)
    if err := SaveConfig(b); err != nil {  // ← 阶段 1 失败
        SendErrorResult(res, err)          // ← 返回错误
        return                             // ← 直接返回，不执行后续步骤
    }
    Config.Load()
    SendSuccessResult(res, nil)
}
```

**回滚效果**：✅ **完全回滚**
- 磁盘文件保持原样（写入失败意味着没有修改）
- 内存配置保持原样（没有调用 `Load()`）
- 返回错误给前端，用户可以重试

代码位置：`server/ctrl/config.go:17-19`

#### 阶段 2：加载到内存失败

**代码**：`server/common/config.go:186-219`

```go
func (this *Configuration) Load() error {
    cFile, err := LoadConfig()  // ← 从磁盘读取（可能失败）
    if err != nil {
        Log.Error("config::load %s", err)
        return err
    }
    
    // Hydration 过程（逐个字段设置）
    for path, value := range flattenJSON("", raw) {
        el := this.Get(path)
        if el.currentElement != nil && el.currentElement.Value != value {
            el.currentElement.Value = value  // ← 逐个修改内存
        }
    }
    
    this.cache.Clear()
    Log.SetVisibility(this.Get("log.level").String())
    for _, fn := range Hooks.Get.OnConfig() {
        fn()  // ← 触发钩子（可能失败）
    }
    return nil
}
```

**回滚效果**：⚠️ **部分回滚 / 无回滚**
- 如果 `LoadConfig()` 失败（文件损坏、解密失败）：
  - 磁盘文件已被修改（阶段 1 成功）
  - 内存配置**部分修改**（Hydration 是逐个字段进行的，中途失败会导致不一致）
  - 返回错误，但内存已处于不一致状态
- 如果 Hydration 中途失败：
  - 部分字段已更新，部分字段未更新
  - 没有快照/回滚机制恢复
- 如果 `OnConfig` 钩子失败：
  - 内存配置已更新
  - 部分插件收到通知，部分未收到
  - 没有补偿机制

**风险场景**：
1. 写入了损坏的 JSON 到磁盘，下次服务启动失败
2. 解密失败导致敏感字段丢失
3. Hydration 中断导致内存配置不完整

代码位置：`server/common/config.go:186-219`

#### 阶段 3：插件钩子失败

**代码**：`server/common/config.go:215-217`

```go
for _, fn := range Hooks.Get.OnConfig() {
    fn()  // ← 插件钩子执行，panic 会导致整个服务崩溃？
}
```

**回滚效果**：❌ **无回滚**
- 钩子函数没有错误返回值
- 如果钩子 panic，整个服务可能崩溃
- 已执行的钩子没有回滚逻辑

### 6.4 内存操作的原子性保证

虽然没有完整回滚，但内存配置的**单个操作**是线程安全的：

```go
// server/common/config.go:16-18
type Configuration struct {
    mu    sync.RWMutex  // ← 读写锁保护
    cache sync.Map
    Form  []Form
    Conn  []map[string]any
}
```

**锁的使用**：
- `Load()` 中的 Hydration：通过 `Get()` 间接获取写锁
- `Save()` 中的序列化：使用读锁 `RLock()`
- `Set()` 方法：使用写锁 `Lock()`
- `Get()` 方法：使用写锁 `Lock()`（遍历树时需要）

**风险**：`Load()` 不是原子操作，它会多次调用 `Get()`（每次获取锁），期间其他 goroutine 可能读取到部分更新的配置。

### 6.5 现有防护机制

1. **文件权限保护**：`init()` 函数确保配置文件权限为 `0660`，目录为 `0770`

```go
// server/common/config_state.go:115-123
func init() {
    Hooks.Register.Onload(func() {
        if err := os.Chmod(GetAbsolutePath(CONFIG_PATH), 0770); ...
        if err := os.Chmod(GetAbsolutePath(CONFIG_PATH, "config.json"), 0660); ...
    })
}
```

2. **JSON 格式验证**：`SaveConfig()` 使用 `PrettyPrint()` 间接验证 JSON 格式
3. **加密容错**：解密失败时记录警告但继续加载（`CONFIG_ENCRYPT=false` 时跳过解密）

```go
// server/common/config_state.go:53-57
t, err := DecryptString(Hash(key, 16), p)
if err != nil {
    if !defaultValue(true, "CONFIG_ENCRYPT") {
        break  // ← 加密被禁用时跳过
    }
    Log.Warning(...)
    continue  // ← 记录警告，继续加载其他字段
}
```

### 6.6 建议的改进方向

当前实现存在数据丢失风险，建议添加：

1. **写入前备份**：保存新配置前创建 `config.json.bak`
2. **原子写入**：先写入 `config.json.tmp`，成功后 rename
3. **快照回滚**：`Load()` 前创建内存快照，失败时恢复
4. **钩子容错**：使用 `recover()` 捕获钩子 panic，确保所有钩子都能执行

---

## 7. 多 Admin 并发冲突解决路径

### 7.1 现状分析："最后写入者获胜"（Last Write Wins）

Filestash 没有显式的并发冲突检测和解决机制，而是依赖**"防抖 + 锁 + 最后写入者获胜"**的策略。

### 7.2 前端层面的冲突避免

#### 双重防抖机制

```javascript
// ctrl_settings.js:56-66
effect(init$.pipe(
    useForm$(() => qsa($container, "[data-bind=\"form\"] [name]")),
    rxjs.debounceTime(250),          // ← 第一重：输入防抖 250ms
    rxjs.mergeMap((formState) => ...),
    reshapeConfigBeforeSave,
    saveConfig(),                     // ← 第二重：保存防抖 800ms
    rxjs.catchError(ctrlError()),
));
```

```javascript
// model_config.js:29-44
export function save() {
    return rxjs.pipe(
        rxjs.tap(() => isSaving$.next(true)),
        rxjs.debounceTime(800),        // ← 保存防抖 800ms
        rxjs.mergeMap((formData) => ajax(...)),
        rxjs.tap(() => isSaving$.next(false)),
    );
}
```

**效果**：
- 单个用户快速输入不会触发多次保存
- 总计约 1 秒的防抖窗口，减少并发冲突概率

#### 保存状态锁

```javascript
// model_config.js:4-15
const isSaving$ = new rxjs.BehaviorSubject(false);

const config$ = isSaving$.pipe(
    rxjs.filter((loading) => !loading),  // ← 保存中不获取新配置
    rxjs.switchMapTo(ajax(...)),
    rxjs.shareReplay(1),
);
```

**效果**：
- 保存期间阻止新的配置获取
- 单个 tab 内不会同时有多个保存请求
- 但**多个 tab / 多个用户**之间没有协调

代码位置：`public/assets/pages/adminpage/model_config.js:4-15`

### 7.3 后端层面的冲突避免

#### 内存读写锁

```go
// server/common/config.go:16-18
type Configuration struct {
    mu sync.RWMutex  // ← 保护内存结构
}
```

**锁的使用场景**：
| 操作 | 锁类型 | 说明 |
|------|--------|------|
| `Get(path)` | 写锁 | 可能需要创建新节点 |
| `Set(value)` | 写锁 | 修改配置值 |
| `Save()` | 读锁 | 序列化内存结构 |
| `Load()` | 多次写锁 | 每个字段 `Get()` 都会获取 |

**局限性**：
- 只保护内存结构，不保护磁盘文件
- `Load()` 过程中会多次获取/释放锁，不是原子操作
- 多个请求可以同时进入 `PrivateConfigUpdateHandler`

#### 缺失的并发控制

后端**没有**以下机制：

1. **文件锁（flock）**：多个进程同时写入可能导致文件损坏
2. **乐观锁（ETag / If-Match）**：无法检测"读取-修改-写入"之间的冲突
3. **版本号**：配置文件没有版本字段，无法检测过期修改
4. **差异合并**：后写入者直接覆盖前写入者的所有修改

### 7.4 实际并发场景分析

#### 场景 1：同一用户多个 tab

```
Tab A: GET /admin/api/config → 获取版本 1
Tab B: GET /admin/api/config → 获取版本 1
Tab A: 修改字段 X → POST（覆盖整个配置）
Tab B: 修改字段 Y → POST（覆盖整个配置，X 的修改丢失！）
```

**结果**：Tab A 的修改被 Tab B 覆盖，数据丢失。

#### 场景 2：两个不同 admin 同时修改

```
Admin A: 修改 log.level = DEBUG
Admin B: 修改 general.host = files.example.com
Admin A 的 POST 先到达 → 保存 A 的修改
Admin B 的 POST 后到达 → 保存 B 的修改（A 的修改被覆盖！）
```

**结果**：Admin A 的修改丢失。

#### 场景 3：保存 + 读取并发

```
Goroutine A: Config.Load() （正在 Hydration，已更新部分字段）
Goroutine B: Config.Get("log.level").String() （可能读到新值）
Goroutine C: Config.Get("general.host").String() （可能读到旧值）
```

**结果**：短暂的不一致窗口，读取到部分更新的配置。

### 7.5 为什么"最后写入者获胜"可以接受

虽然存在数据丢失风险，但在 Filestash 的场景下这是可接受的权衡：

1. **低频修改**：配置修改是低频操作，并发概率低
2. **管理员数量少**：通常只有 1-2 个管理员
3. **配置字段独立**：大部分场景下管理员修改不同字段
4. **影响范围有限**：配置修改后可以重新设置

### 7.6 建议的改进方向

如果需要更强的并发保证，可以添加：

1. **ETag 乐观锁**：
   ```
   GET /admin/api/config → 返回 ETag: "v123"
   POST /admin/api/config → 携带 If-Match: "v123"
   服务端校验 ETag，不匹配返回 412 Precondition Failed
   ```

2. **部分更新 API**：
   ```
   PATCH /admin/api/config/general/host
   Body: { "value": "files.example.com" }
   只更新单个字段，减少冲突概率
   ```

3. **文件锁**：使用 `syscall.Flock()` 防止多进程写入

4. **变更审计**：记录配置变更历史，便于回滚

---

## 8. 配置变更通知给在线 Admin 的机制

### 8.1 核心结论：无实时推送，依赖拉取模式

Filestash **没有** WebSocket、SSE 或任何实时推送机制来通知在线 admin 配置变更。所有配置更新都采用**"拉取模式"**，即：
- 保存者自己能看到更新（因为保存后重新拉取）
- 其他在线 admin 只能在下次拉取时看到更新

### 8.2 前端配置拉取机制

#### 单例冷 Observable 设计

```javascript
// public/assets/pages/adminpage/model_config.js:6-15
const config$ = isSaving$.pipe(
    rxjs.filter((loading) => !loading),  // ← 保存中不获取
    rxjs.switchMapTo(ajax({               // ← 每次订阅触发新请求
        url: "admin/api/config",
        method: "GET",
        responseType: "json"
    })),
    rxjs.map((res) => res.responseJSON.result),
    rxjs.shareReplay(1),                  // ← 缓存最近一次结果
);
```

**关键特性**：
1. **冷 Observable**：`ajax()` 是冷 Observable，**每次新订阅才会触发 HTTP 请求**
2. **缓存复用**：`shareReplay(1)` 让多个订阅者共享同一份最近数据
3. **保存互斥**：`isSaving$` 过滤确保保存期间不会发起新的获取请求

**代码解读**：
- `config$` 本身不会主动轮询，它是被动的
- 只有当有新的订阅者（或保存结束后重新订阅）才会拉取新配置
- `shareReplay(1)` 确保同一页面内多次订阅不会重复请求

代码位置：`public/assets/pages/adminpage/model_config.js:6-15`

#### 保存后自动刷新机制

```javascript
// public/assets/pages/adminpage/model_config.js:29-44
export function save() {
    return rxjs.pipe(
        rxjs.tap(() => isSaving$.next(true)),
        rxjs.debounceTime(800),
        rxjs.mergeMap((formData) => ajax({
            url: "admin/api/config",
            method: "POST",
            // ...
        })),
        rxjs.tap(() => isSaving$.next(false)),  // ← 保存完成，isSaving$ = false
        // ...
    );
}
```

**刷新流程**：
1. 保存开始 → `isSaving$.next(true)`
2. 保存完成 → `isSaving$.next(false)`
3. `config$` 管道中的 `filter((loading) => !loading)` 让值通过
4. `switchMapTo(ajax(...))` 触发新的 GET 请求
5. `shareReplay(1)` 将新配置推送给所有订阅者

**效果**：保存者自己的页面会自动获取新配置并刷新。

代码位置：`public/assets/pages/adminpage/model_config.js:29-44`

### 8.3 页面首次加载拉取

```javascript
// public/assets/pages/adminpage/ctrl_settings.js:27-30
const config$ = getAdminConfig().pipe(
    rxjs.first(),  // ← 只取第一个值然后 complete
    reshapeConfigBeforeDisplay,
);
```

```javascript
// public/assets/pages/adminpage/model_config.js:25-27
export function get() {
    return config$;  // ← 返回可观察对象，订阅触发拉取
}
```

**关键点**：`rxjs.first()` 确保页面只在加载时拉取一次配置，之后不会自动刷新。

### 8.4 其他在线 Admin 的通知盲区

**场景**：Admin A 和 Admin B 同时打开设置页面

```
Admin A: 打开设置页面 → GET /admin/api/config → 配置 v1
Admin B: 打开设置页面 → GET /admin/api/config → 配置 v1
Admin A: 修改 log.level = DEBUG → POST → 配置变为 v2
Admin A: isSaving$ 从 true→false → 触发 GET → 获取配置 v2 ✅
Admin B: 页面继续显示配置 v1 ❌（无任何通知）
```

**Admin B 的刷新方式**：
1. 手动刷新页面 F5
2. 离开设置页面再进入
3. 修改自己的配置（触发保存→拉取流程）
4. 每 30 秒的 session 轮询 **不会** 触发配置刷新

### 8.5 Session 轮询机制（不触发配置刷新）

```javascript
// public/assets/pages/adminpage/model_admin_session.js:6-19
const adminSession$ = rxjs.merge(
    sessionSubject$,
    rxjs.merge(
        rxjs.interval(30000),  // ← 每 30 秒轮询一次
        rxjs.fromEvent(document, "visibilitychange").pipe(
            rxjs.filter(() => !document.hidden)  // ← 页面可见时也轮询
        ),
    ).pipe(
        rxjs.startWith(null),
        rxjs.mergeMap(() => ajax({ url: "admin/api/session", ... })),
        rxjs.map(({ responseJSON }) => responseJSON.result),
    )
).pipe(
    rxjs.distinctUntilChanged(),  // ← session 不变时不触发
    rxjs.shareReplay(1)
);
```

**重要**：这个 30 秒轮询只检查 admin session 是否有效，**不会**拉取配置更新。配置变更不会触发任何通知。

代码位置：`public/assets/pages/adminpage/model_admin_session.js:6-19`

### 8.6 配置变更通知挂载点汇总

| 挂载点 | 通知范围 | 机制 | 延迟 |
|--------|----------|------|------|
| 保存者页面自动刷新 | 仅保存者自己 | `isSaving$` true→false 触发 GET | 保存完成后立即 |
| 页面加载拉取 | 所有进入页面的 admin | `rxjs.first()` 单次拉取 | 页面加载时 |
| 重新进入设置页面 | 导航到该页面的 admin | 新订阅触发 GET | 页面切换时 |
| Session 轮询 | ❌ 无配置通知 | 只检查 session 有效性 | N/A |
| WebSocket/SSE | ❌ 不存在 | 无实时推送 | N/A |

### 8.7 改进建议

如果需要实时通知所有在线 admin，可以添加：

1. **WebSocket 广播**：配置保存后通过 WebSocket 向所有在线 admin 推送变更事件
2. **长轮询**：`/admin/api/config` 支持长轮询，配置变更时立即返回
3. **版本号检测**：前端定时（如 10 秒）轮询配置版本号，发现变更后拉取完整配置
4. **EventSource (SSE)**：服务端推送配置变更事件

---

## 9. 敏感字段在审计场景下的明文恢复路径

### 9.1 敏感字段分类与存储方式

Filestash 有三类敏感信息，存储方式不同，审计时的恢复能力也不同：

| 敏感信息类型 | 存储方式 | 审计时能否恢复明文 |
|-------------|----------|-------------------|
| 中间件参数（identity_provider.params、attribute_mapping.params） | AES-GCM 加密 | ✅ 可以恢复 |
| Admin 密码（auth.admin） | bcrypt 哈希 | ❌ 不可恢复 |
| 分享链接密码（share.password） | bcrypt 哈希 | ❌ 不可恢复 |

### 9.2 AES-GCM 加密字段的明文恢复路径

#### 完整恢复链路

```
审计插件调用 Config.Get("middleware.identity_provider.params")
    ↓
Configuration.Get(path)  // server/common/config.go:364
    ↓
返回 FormElement（Value 是明文）
    ↓
审计插件读取 .String() 获取明文
```

**为什么可以直接获取明文？**

因为配置在加载到内存时已经完成了解密：

```go
// server/common/config_state.go:40-65 (LoadConfig)
func LoadConfig() ([]byte, error) {
    // ...
    // 1. 从磁盘读取加密的 JSON
    configStr, err := os.ReadFile(...)
    
    // 2. 初始化解密密钥
    var key string = CONFIG_SECRET
    if key == "" {
        key = Config.Get("general.secret_key").String()
    }
    
    // 3. 解密所有敏感字段
    for _, jsonPathWithEncryptedData := range configKeysToEncrypt {
        p := gjson.Get(configStr, jsonPathWithEncryptedData).String()
        // 🔑 解密！
        t, err := DecryptString(Hash(key, 16), p)  // ← AES-GCM 解密
        // ...
        configStr, err = sjson.Set(configStr, jsonPathWithEncryptedData, t)
    }
    
    return []byte(configStr), nil  // ← 返回解密后的 JSON
}
```

代码位置：`server/common/config_state.go:48-65`

#### 解密算法详解

```go
// server/common/crypto.go:27-53
func EncryptString(secret string, data string) (string, error) {
    d, _ := compress([]byte(data))               // ← zlib 压缩
    d, _ = EncryptAESGCM([]byte(secret), d)      // ← AES-256-GCM 加密
    return base64.URLEncoding.EncodeToString(d), nil
}

func DecryptString(secret string, data string) (string, error) {
    d, _ := base64.URLEncoding.DecodeString(data)  // ← Base64 解码
    d, _ = DecryptAESGCM([]byte(secret), d)        // ← AES-256-GCM 解密
    d, _ = decompress(d)                           // ← zlib 解压
    return string(d), nil
}
```

**密钥派生**：
- 主密钥：优先使用环境变量 `CONFIG_SECRET`，否则使用 `general.secret_key`
- 加密密钥：`Hash(key, 16)` → SHA-256 哈希后取前 16 字节
- 算法：AES-256-GCM（12 字节 nonce，16 字节认证标签）

代码位置：`server/common/crypto.go:27-53,128-159`

#### 内存中明文的生命周期

```go
// server/common/config.go:186-219 (Load)
func (this *Configuration) Load() error {
    // 1. 从文件读取（已解密）
    cFile, err := LoadConfig()
    
    // 2. 扁平化 JSON
    raw := map[string]any{}
    json.Unmarshal(cFile, &raw)
    
    // 3. Hydration: 明文写入内存
    for path, value := range flattenJSON("", raw) {
        el := this.Get(path)
        if el.currentElement != nil {
            el.currentElement.Value = value  // ← 明文存入内存
        }
    }
    
    return nil
}
```

**明文在内存中**：
- `Configuration.Form[].Elmnts[].Value` 字段存储明文
- 只要服务运行，敏感字段明文就存在于内存中
- 通过 `Config.Get("path").String()` 可以随时读取明文

#### 审计插件获取明文的完整路径

```go
// 审计插件实现 IAuditPlugin 接口
type IAuditPlugin interface {
    Query(ctx *App, searchParams map[string]string) (AuditQueryResult, error)
}

// 在 Query 方法中可以直接访问明文
func (myAudit MyAudit) Query(ctx *App, params map[string]string) (AuditQueryResult, error) {
    // ✅ 直接获取中间件参数明文
    idpParams := Config.Get("middleware.identity_provider.params").String()
    attrParams := Config.Get("middleware.attribute_mapping.params").String()
    
    // 审计逻辑...
    
    return AuditQueryResult{...}, nil
}
```

**注册审计插件**：
```go
// server/common/plugin.go:185-193
var audit IAuditPlugin

func (this Register) AuditEngine(a IAuditPlugin) {
    audit = a
}

func (this Get) AuditEngine() IAuditPlugin {
    return audit
}
```

**默认实现**：`server/model/audit.go` 中的 `SimpleAudit` 只是占位符，提示需要安装审计插件。

### 9.3 bcrypt 哈希字段的不可恢复性

#### Admin 密码

```go
// server/ctrl/admin.go:57-62
func AdminAuthenticationHandler(...) {
    // 从配置中读取 bcrypt 哈希
    adminPassword := Config.Get("auth.admin").String()  // ← 存储的是哈希，不是明文
    
    // 验证密码（只能比较，不能解密）
    if err := bcrypt.CompareHashAndPassword(
        []byte(adminPassword), 
        []byte(password)
    ); err != nil {
        SendErrorResult(res, ErrAuthenticationFailed)
        return
    }
}
```

**特点**：
- 存储的是 60 字符的 bcrypt 哈希（`$2a$10$...`）
- 前端在设置时就用 bcrypt.js 哈希（`ctrl_setup.js:91`）
- 明文密码**从未**进入后端内存（除了验证时的短暂存在）
- 审计时**无法**恢复 admin 密码明文

代码位置：`server/ctrl/admin.go:57-62`

#### 分享链接密码

```go
// server/common/types.go:178-228
const PASSWORD_DUMMY = "{{PASSWORD}}"

// 序列化时掩码
func (s *Share) MarshalJSON() ([]byte, error) {
    p := Share{
        Password: func(pass *string) *string {
            if pass != nil {
                return NewString(PASSWORD_DUMMY)  // ← 替换为占位符
            }
            return nil
        }(s.Password),
        // ...
    }
    return json.Marshal(p)
}
```

**特点**：
- 分享密码存储在后端（可能是 bcrypt 或明文，取决于后端实现）
- 序列化为 JSON 时被替换为 `{{PASSWORD}}` 掩码
- 审计时如果直接访问数据库可能获得哈希值，但同样无法恢复明文

### 9.4 明文恢复权限边界

| 角色 | 能否获取敏感字段明文 | 途径 |
|------|---------------------|------|
| 前端 Admin UI | ❌ 不能 | 前端只获取配置 schema，敏感字段被加密/哈希 |
| 后端插件 | ✅ 可以 | `Config.Get("path").String()` 直接读取内存明文 |
| 审计插件 | ✅ 可以 | 作为后端插件，享有同等权限 |
| 配置文件 | ❌ 不能 | 加密存储，需要密钥解密 |
| 内存 dump | ✅ 可以 | 明文存在于 Go 堆内存中 |

### 9.5 审计时的明文恢复代码核对表

#### 核对路径 1：加密字段能正常解密

**代码核对**：
1. ✅ `LoadConfig()` 中调用 `DecryptString()` 解密所有 `configKeysToEncrypt` 路径
2. ✅ 解密密钥正确派生：`Hash(CONFIG_SECRET || secret_key, 16)`
3. ✅ 解密后的明文写入内存 `FormElement.Value`
4. ✅ `Config.Get()` 能正确找到该路径并返回 `Value`

**可能的失败点**：
- `CONFIG_SECRET` 环境变量变更导致密钥不匹配
- `general.secret_key` 被重置
- 配置文件损坏

#### 核对路径 2：审计插件有权限访问

**代码核对**：
1. ✅ `AuditEngine.Query()` 运行在服务端上下文
2. ✅ 没有权限检查限制审计插件访问 `Config.Get()`
3. ✅ `IAuditPlugin` 接口没有对敏感字段的访问限制
4. ✅ 默认 `SimpleAudit` 不访问敏感字段，但自定义插件可以

**风险点**：恶意审计插件可以窃取所有敏感字段明文

#### 核对路径 3：bcrypt 字段不可恢复

**代码核对**：
1. ✅ `auth.admin` 存储的是 bcrypt 哈希，不是明文
2. ✅ 前端设置密码时就已哈希（`ctrl_setup.js:91`）
3. ✅ 后端只使用 `bcrypt.CompareHashAndPassword()` 验证
4. ✅ 没有任何代码路径能从哈希恢复明文

---

## 10. 配置变更在审计记录中的字段对比展示路径

### 10.1 核心结论：无内置字段对比，依赖审计插件扩展

Filestash **没有**内置的配置变更字段对比（diff）功能。配置变更的审计记录完全依赖于**第三方审计插件**的实现。

### 10.2 审计插件接口定义

```go
// server/common/types.go:65-71
type IAuditPlugin interface {
    Query(ctx *App, searchParams map[string]string) (AuditQueryResult, error)
}

type AuditQueryResult struct {
    Form       *Form  `json:"form"`   // 查询表单 schema
    RenderHTML string `json:"render"` // 审计结果渲染 HTML
}
```

**设计意图**：
- `Form` 字段允许审计插件定义自己的查询条件表单（日期范围、操作类型等）
- `RenderHTML` 字段允许审计插件完全自定义展示内容，包括字段对比 diff
- 核心系统不干预审计数据的存储和展示格式

代码位置：`server/common/types.go:65-71`

### 10.3 默认审计实现（占位符）

```go
// server/model/audit.go:58-75
type SimpleAudit struct{}

func (this SimpleAudit) Query(ctx *App, searchParams map[string]string) (AuditQueryResult, error) {
    return AuditQueryResult{
        Form: &AuditForm,
        RenderHTML: `<style>
            #alert-audit-missing{
                background: var(--error); color: var(--super-light);
                padding: 15px 15px;
                border-radius: 2px;
                margin-top: 15px;
            }
        </style>
        <div id="alert-audit-missing">
            You need to install an audit plugin to use this
        </div>`,
    }, nil
}
```

**说明**：默认的 `SimpleAudit` 只是提示用户需要安装审计插件，不记录任何数据。

代码位置：`server/model/audit.go:58-75`

### 10.4 审计 API 入口

```go
// server/ctrl/admin.go:138-158
func FetchAuditHandler(ctx *App, res http.ResponseWriter, req *http.Request) {
    plg := Hooks.Get.AuditEngine()
    if plg == nil {
        SendErrorResult(res, ErrNotImplemented)
        return
    }
    searchParams := map[string]string{}
    _get := req.URL.Query()
    for key, element := range _get {
        if len(element) == 0 {
            continue
        }
        searchParams[key] = element[0]
    }
    result, err := plg.Query(ctx, searchParams)
    if err != nil {
        SendErrorResult(res, err)
        return
    }
    SendSuccessResult(res, result)
}
```

**前端调用**：
```javascript
// public/assets/pages/adminpage/model_audit.js:6-13
export function get(searchParams = new URLSearchParams()) {
    return ajax({
        url: "admin/api/audit?" + searchParams.toString(),
        responseType: "json"
    }).pipe(
        rxjs.map(({ responseJSON }) => responseJSON.result)
    );
}
```

### 10.5 配置变更的审计记录挂载点

#### 挂载点 1：PrivateConfigUpdateHandler（后端保存时）

```go
// server/ctrl/config.go:15-23
func PrivateConfigUpdateHandler(ctx *App, res http.ResponseWriter, req *http.Request) {
    b, _ := io.ReadAll(req.Body)  // 新配置 JSON
    if err := SaveConfig(b); err != nil {
        SendErrorResult(res, err)
        return
    }
    Config.Load()  // 加载新配置
    SendSuccessResult(res, nil)
}
```

**插件可以在这里注入审计**：
- 方法：注册 `OnConfig` 钩子，在配置加载后记录变更
- 挑战：没有旧值可用（`Load()` 已经覆盖了内存）
- 解决方案：插件需要自己保存上一次配置的快照

#### 挂载点 2：Set() 方法（程序化修改时）

```go
// server/common/config.go:429-444
func (this *ConfigElement) Set(value interface{}) *ConfigElement {
    if this.currentElement == nil {
        return this
    }
    this.cfg.mu.Lock()
    changed := this.currentElement.Value != value  // ← 可以在这里获取旧值
    if changed {
        this.currentElement.Value = value
        this.cfg.cache.Clear()
    }
    this.cfg.mu.Unlock()
    if changed {
        this.cfg.Save()  // ← 保存到磁盘
    }
    return this
}
```

**插件可以在这里注入审计**：
- 方法：`changed` 变量可以检测到值变化
- 优势：可以获取到旧值和新值进行对比
- 局限：`Set()` 是公共方法，所有代码都可以调用，审计插件需要用 AOP 方式拦截

#### 挂载点 3：前端保存流程（用户交互时）

```javascript
// public/assets/pages/adminpage/ctrl_settings.js:56-66
effect(init$.pipe(
    useForm$(() => qsa($container, "[data-bind=\"form\"] [name]")),
    rxjs.debounceTime(250),           // ← 250ms 输入防抖
    rxjs.mergeMap((formState) => config$.pipe(
        rxjs.first(),
        rxjs.map((formSpec) => mutateForm(formSpec, formState)),  // ← 应用变更到 formSpec
    )),
    reshapeConfigBeforeSave,  // ← 重组配置，添加 middleware/connections
    saveConfig(),             // ← 发送到后端
    rxjs.catchError(ctrlError()),
));
```

**前端可以在这里注入审计**：
- 方法：在 `saveConfig()` 之前比较 `formState` 和原始 `formSpec`
- 优势：前端可以精确知道哪个字段被修改了
- 局限：前端数据不可信，后端需要二次校验

### 10.6 字段对比的实现路径（审计插件视角）

如果要实现配置变更的字段对比，审计插件需要：

#### 步骤 1：在 OnConfig 钩子中捕获配置快照

```go
// 审计插件示例代码
var lastConfigSnapshot map[string]any

func init() {
    Hooks.Register.OnConfig(func() {
        // 1. 序列化当前配置
        currentConfig := serializeConfig(Config)
        
        // 2. 如果有上一次快照，计算 diff
        if lastConfigSnapshot != nil {
            diff := computeDiff(lastConfigSnapshot, currentConfig)
            if len(diff) > 0 {
                // 3. 记录审计日志
                recordAuditLog("config_change", diff)
            }
        }
        
        // 4. 保存当前快照
        lastConfigSnapshot = currentConfig
    })
}
```

#### 步骤 2：使用 flattenJSON 进行路径级对比

```go
// server/common/config.go:221-238
func flattenJSON(prefix string, m map[string]any) map[string]any {
    out := map[string]any{}
    for k, v := range m {
        key := k
        if prefix != "" {
            key = prefix + "." + k
        }
        switch val := v.(type) {
        case map[string]any:
            for nk, nv := range flattenJSON(key, val) {
                out[nk] = nv
            }
        default:
            out[key] = v
        }
    }
    return out
}
```

**对比逻辑**：
```go
oldFlat := flattenJSON("", oldConfig)
newFlat := flattenJSON("", newConfig)

for path, oldVal := range oldFlat {
    newVal, exists := newFlat[path]
    if !exists {
        // 字段被删除
        diff[path] = map[string]any{"op": "remove", "old": oldVal}
    } else if oldVal != newVal {
        // 字段被修改
        diff[path] = map[string]any{"op": "modify", "old": oldVal, "new": newVal}
    }
}

for path, newVal := range newFlat {
    if _, exists := oldFlat[path]; !exists {
        // 新增字段
        diff[path] = map[string]any{"op": "add", "new": newVal}
    }
}
```

#### 步骤 3：前端展示字段对比

审计插件可以在 `RenderHTML` 中返回包含 diff 展示的 HTML：

```html
<!-- 示例 diff 展示 -->
<div class="config-diff">
    <h3>配置变更详情</h3>
    <table>
        <tr><th>字段路径</th><th>操作</th><th>旧值</th><th>新值</th></tr>
        <tr class="diff-modify">
            <td>log.level</td>
            <td>修改</td>
            <td class="old-value">INFO</td>
            <td class="new-value">DEBUG</td>
        </tr>
        <tr class="diff-add">
            <td>features.new_feature.enable</td>
            <td>新增</td>
            <td></td>
            <td class="new-value">true</td>
        </tr>
    </table>
</div>
```

### 10.7 现有代码中的 diff 能力

项目中只有一处 diff 相关代码：**编辑器的 diff 模式**

```javascript
// public/assets/pages/viewerpage/application_editor/diff.js
import "../../../lib/vendor/codemirror/mode/diff/diff.js";
window.CodeMirror.__mode = "diff";
export default window.CodeMirror;
```

这是用于文件内容 diff 展示的 CodeMirror 模式，**不用于配置变更对比**。

### 10.8 字段对比展示路径汇总

| 层级 | 对比能力 | 代码位置 | 备注 |
|------|----------|----------|------|
| 后端核心 | ❌ 无内置 | - | 需要审计插件扩展 |
| OnConfig 钩子 | ⚠️ 可扩展 | `server/common/plugin.go:286-294` | 无旧值，需自行快照 |
| Set() 方法 | ⚠️ 可扩展 | `server/common/config.go:429-444` | 有旧值，但需 AOP 拦截 |
| 前端保存流程 | ⚠️ 可扩展 | `public/assets/pages/adminpage/ctrl_settings.js:56-66` | 有表单状态，但不可信 |
| flattenJSON | ✅ 可用 | `server/common/config.go:221-238` | 可用于路径级 diff 计算 |
| 审计插件接口 | ✅ 完全自定义 | `server/common/types.go:65-71` | RenderHTML 可展示任意内容 |
| 编辑器 diff 模式 | ❌ 不适用 | `public/assets/pages/viewerpage/application_editor/diff.js` | 仅用于文件内容 |

---

## 11. 多层级配置合并冲突时的优先级决策

### 11.1 核心结论：四级配置源，高优先级覆盖低优先级

Filestash 的配置系统有**四个层级**的配置源，按照优先级从高到低排列：

| 优先级 | 配置源 | 时机 | 代码位置 |
|--------|--------|------|----------|
| 1（最高） | `Config.Set()` 程序化设置 | 运行时 | `server/common/config.go:429-444` |
| 2 | 环境变量（`Initialise()` 中） | 服务启动 | `server/common/config.go:243-249` |
| 3 | 配置文件（`config.json`） | 每次 `Load()` | `server/common/config_state.go:40-65` |
| 4（最低） | 默认值（Schema 定义） | 每次读取 | `server/common/config.go:475-486` |

**决策规则**：高优先级的配置值完全覆盖低优先级的值，没有合并逻辑。

### 11.2 优先级决策代码核对

#### 优先级 1：`Set()` 程序化设置（最高优先级）

```go
// server/common/config.go:429-444
func (this *ConfigElement) Set(value interface{}) *ConfigElement {
    // ...
    changed := this.currentElement.Value != value
    if changed {
        this.currentElement.Value = value  // ← 直接覆盖 Value 字段
        this.cfg.cache.Clear()
    }
    // ...
    if changed {
        this.cfg.Save()  // ← 保存到配置文件
    }
    return this
}
```

**特点**：
- 直接修改 `FormElement.Value`
- 自动保存到配置文件（会持久化）
- 优先级最高，会覆盖所有其他来源

**典型使用场景**：
```go
// Initialise() 中设置环境变量
if env := os.Getenv("ADMIN_PASSWORD"); env != "" {
    this.Get("auth.admin").Set(env)  // ← 程序化设置
}

// 自动生成缺失字段
if this.Get("general.secret_key").String() == "" {
    key := RandomString(16)
    this.Get("general.secret_key").Set(key)  // ← 程序化设置
}
```

代码位置：`server/common/config.go:429-444`

#### 优先级 2：环境变量（`Initialise()` 中）

```go
// server/common/config.go:241-260
func (this *Configuration) Initialise() {
    shouldSave := false
    if env := os.Getenv("ADMIN_PASSWORD"); env != "" {
        shouldSave = true
        this.Get("auth.admin").Set(env)  // ← 通过 Set() 设置，优先级 1
    }
    if env := os.Getenv("APPLICATION_URL"); env != "" {
        shouldSave = true
        _ = this.Get("general.host").Set(env).String()  // ← 通过 Set() 设置
    }
    // ...
    if shouldSave {
        this.Save()  // ← 保存到配置文件
    }
    // ...
}
```

**注意**：环境变量不是独立的层级，而是通过 `Set()` 方法设置的，所以实际优先级等同于 `Set()`。

#### 优先级 2 变体：`defaultValue[T]()` 编译时环境变量

```go
// server/common/config.go:506-522
func defaultValue[T string | int | bool](dval T, envName string) T {
    if val := os.Getenv(envName); val != "" {
        switch any(dval).(type) {
        case int:
            if n, err := strconv.Atoi(val); err == nil {
                return any(n).(T)
            }
        case bool:
            if b, err := strconv.ParseBool(val); err == nil {
                return any(b).(T)
            }
        default:
            return any(val).(T)
        }
    }
    return dval
}
```

**使用方式**：
```go
// Schema 定义时
FormElement{
    Name: "port",
    Type: "number",
    Default: defaultValue(8334, "FILESTASH_PORT"),  // ← 环境变量优先
    // ...
}

FormElement{
    Name: "level",
    Type: "select",
    Default: defaultValue("INFO", "LOG_LEVEL"),  // ← 环境变量优先
    // ...
}
```

**优先级说明**：
- 这个环境变量在**编译时**作为 `Default` 值的一部分
- 优先级低于配置文件（因为配置文件会设置 `Value`，而 `Interface()` 会优先返回 `Value`）
- 所以实际优先级是：`Set()` > 配置文件 > `defaultValue()` 环境变量 > 硬编码默认值

代码位置：`server/common/config.go:506-522`

#### 优先级 3：配置文件（`config.json`）

```go
// server/common/config.go:186-211
func (this *Configuration) Load() error {
    // ...
    // 从配置文件加载
    cFile, err := LoadConfig()  // ← 读取并解密配置文件
    
    // 扁平化 JSON
    raw := map[string]any{}
    json.Unmarshal(cFile, &raw)
    
    // Hydration: 覆盖 Value 字段
    for path, value := range flattenJSON("", raw) {
        el := this.Get(path)
        if el.currentElement != nil && el.currentElement.Value != value {
            el.currentElement.Value = value  // ← 覆盖 Value
        }
    }
    // ...
}
```

**特点**：
- 覆盖 `FormElement.Value` 字段
- 每次 `Load()` 都会重新加载
- 优先级高于 `Default`，低于 `Set()`

#### 优先级 4：默认值（Schema 定义）

```go
// server/common/config.go:475-486
func (this *ConfigElement) Interface() interface{} {
    if this.currentElement == nil {
        return nil
    }
    this.cfg.mu.RLock()
    el := *this.currentElement
    this.cfg.mu.RUnlock()
    if el.Value == nil {
        return el.Default  // ← Value 为 nil 时返回 Default
    }
    return el.Value  // ← 优先返回 Value
}
```

**决策点**：
- `Value != nil` → 返回 `Value`（来自配置文件或 `Set()`）
- `Value == nil` → 返回 `Default`（来自 Schema 定义，可能包含 `defaultValue()` 环境变量）

代码位置：`server/common/config.go:475-486`

### 11.3 完整优先级决策树

```
Config.Get("path").String()
    ↓
ConfigElement.Interface()
    ├─ Value != nil ?
    │   ├─ YES → 返回 Value
    │   │   ├─ 来源 1：Set() 程序化设置
    │   │   └─ 来源 2：配置文件 Load()
    │   └─ NO → 返回 Default
    │       ├─ Default 来自 defaultValue() ?
    │       │   ├─ YES → 环境变量存在 ? 返回环境变量值
    │       │   │                   : 返回硬编码默认值
    │       │   └─ NO → 返回硬编码默认值
    │       └─ 来源：Schema 定义
    └─ currentElement == nil → 返回 nil
```

### 11.4 冲突场景与决策结果

#### 场景 1：配置文件 vs 环境变量（defaultValue）

```bash
# 环境变量
export LOG_LEVEL=DEBUG
```

```go
// Schema 定义
FormElement{
    Name: "level",
    Type: "select",
    Default: defaultValue("INFO", "LOG_LEVEL"),  // Default = "DEBUG"
}
```

```json
// config.json
{
    "log": {
        "level": "WARNING"  // Value = "WARNING"
    }
}
```

**结果**：`Value = "WARNING"` 覆盖 `Default = "DEBUG"` → 返回 `"WARNING"`

#### 场景 2：Set() vs 配置文件

```go
// 启动时 Initialise()
if env := os.Getenv("APPLICATION_URL"); env != "" {
    this.Get("general.host").Set(env)  // Set() 设置 Value = "https://example.com"
}
```

```json
// config.json
{
    "general": {
        "host": "http://localhost:8334"  // 配置文件中的值
    }
}
```

**结果**：
1. `Load()` 从配置文件加载 → `Value = "http://localhost:8334"`
2. `Initialise()` 调用 `Set()` → `Value = "https://example.com"`
3. 最终返回 `"https://example.com"`（`Set()` 优先级更高）

#### 场景 3：多个 Set() 调用

```go
// 插件 A
Config.Get("general.upload_pool_size").Set(10)

// 插件 B
Config.Get("general.upload_pool_size").Set(20)
```

**结果**：最后调用的 `Set()` 生效 → 返回 `20`

**冲突日志**：
```go
// server/common/config.go:418-421
shouldSave := this.currentElement.Default == nil
if shouldSave {
    this.currentElement.Default = value
} else if this.currentElement.Default != value {
    Log.Debug("Attempt to set multiple default config value => %+v", this.currentElement)
}
```

> 注意：这个日志是针对 `Default` 字段多次设置的警告，不是针对 `Value` 字段。

### 11.5 特殊层级：`constant` 虚拟分组

```go
// server/common/config.go:488-504
func (this Configuration) MarshalJSON() ([]byte, error) {
    return Form{
        Form: append(this.Form, Form{
            Title: "constant",
            Elmnts: []FormElement{
                {Name: "version", Type: "text", ReadOnly: true, Value: APP_VERSION},
                {Name: "user", Type: "boolean", ReadOnly: true, Value: username},
                {Name: "license", Type: "text", ReadOnly: true, Value: LICENSE},
            },
        }),
    }.MarshalJSON()
}
```

**特点**：
- `constant` 分组只在序列化到 JSON 时才存在
- 字段都是 `ReadOnly: true`
- 优先级最高（直接硬编码在内存中，无法通过配置文件或环境变量修改）

### 11.6 导出配置的优先级（`Export()`）

```go
// server/common/config.go:286-362
func (this *Configuration) Export() interface{} {
    return struct {
        Editor                  string `json:"editor"`
        // ...
        UploadPoolSize          int    `json:"upload_pool_size"`
        // ...
    }{
        Editor:                  this.Get("general.editor").String(),  // ← 走正常优先级
        UploadPoolSize:          this.Get("general.upload_pool_size").Int(),
        // ...
    }
}
```

**说明**：`Export()` 方法通过 `Get()` 读取配置，所以自动遵循优先级规则。

### 11.7 冲突处理的代码核对表

| 冲突场景 | 决策代码 | 结果 |
|----------|----------|------|
| 配置文件 vs 默认值 | `Interface()` Value 优先 | 配置文件胜出 |
| Set() vs 配置文件 | `Set()` 直接覆盖 Value | Set() 胜出 |
| Set() vs Set() | 后调用的覆盖先调用的 | 后调用者胜出 |
| defaultValue() 环境变量 vs 配置文件 | Value 优先于 Default | 配置文件胜出 |
| defaultValue() 环境变量 vs 硬编码默认 | defaultValue() 内部判断 | 环境变量胜出 |
| constant 分组 vs 所有 | 序列化时硬编码 | constant 胜出 |

---

## 12. 关键代码位置索引

| 功能模块 | 文件路径 | 关键行号 |
|----------|----------|----------|
| 配置核心结构 | `server/common/config.go` | 14-52 |
| 默认 Schema 定义 | `server/common/config.go` | 62-143 |
| 配置加载 (Load) | `server/common/config.go` | 186-219 |
| 配置保存 (Save) | `server/common/config.go` | 262-284 |
| 配置访问 (Get/Set) | `server/common/config.go` | 364-486 |
| 配置文件加密/解密 | `server/common/config_state.go` | 24-113 |
| 配置加密字段列表 | `server/common/config_state.go` | 24-27 |
| OnConfig 钩子 | `server/common/plugin.go` | 286-294 |
| Admin 配置 API | `server/ctrl/config.go` | 11-28 |
| Admin 认证 | `server/ctrl/admin.go` | 16-84 |
| 路由注册 | `server/routes.go` | 41-42, 97 |
| PASSWORD_DUMMY | `server/common/types.go` | 178 |
| 前端 Admin 配置模型 | `public/assets/pages/adminpage/model_config.js` | 1-45 |
| 前端设置页面 | `public/assets/pages/adminpage/ctrl_settings.js` | 1-97 |
| 前端表单渲染 | `public/assets/components/form.js` | 45-329 |
| 前端表单逻辑 | `public/assets/lib/form.js` | 1-124 |
| 设置向导 | `public/assets/pages/adminpage/ctrl_setup.js` | 1-213 |
| **配置迁移兼容** | | |
| Get() 自动创建机制 | `server/common/config.go` | 385-396 |
| Schema() 提升方法 | `server/common/config.go` | 405-409 |
| Interface() 默认值回退 | `server/common/config.go` | 475-486 |
| Initialise() 初始化 | `server/common/config.go` | 241-260 |
| defaultValue 泛型函数 | `server/common/config.go` | 506-522 |
| config_state 可替换警告 | `server/common/config_state.go` | 3-14 |
| **热更新回滚** | | |
| PrivateConfigUpdateHandler | `server/ctrl/config.go` | 15-23 |
| Configuration.Load() 错误处理 | `server/common/config.go` | 186-219 |
| 配置文件权限保护 | `server/common/config_state.go` | 115-123 |
| 解密容错处理 | `server/common/config_state.go` | 53-57 |
| **并发冲突** | | |
| 前端双重防抖 | `public/assets/pages/adminpage/ctrl_settings.js` | 56-66 |
| 前端保存防抖 | `public/assets/pages/adminpage/model_config.js` | 29-44 |
| 前端保存状态锁 | `public/assets/pages/adminpage/model_config.js` | 4-15 |
| 后端读写锁 | `server/common/config.go` | 16-18 |
| Configuration.mu 锁使用 | `server/common/config.go` | 263, 398, 415, 433 |
| **配置变更通知** | | |
| 前端 config$ 单例 | `public/assets/pages/adminpage/model_config.js` | 6-15 |
| 保存后刷新机制 | `public/assets/pages/adminpage/model_config.js` | 29-44 |
| 页面单次拉取 | `public/assets/pages/adminpage/ctrl_settings.js` | 27-30 |
| Session 轮询 | `public/assets/pages/adminpage/model_admin_session.js` | 6-19 |
| **敏感字段恢复** | | |
| LoadConfig 解密 | `server/common/config_state.go` | 48-65 |
| EncryptString/DecryptString | `server/common/crypto.go` | 27-53 |
| AES-GCM 加密实现 | `server/common/crypto.go` | 128-159 |
| Hash 密钥派生 | `server/common/crypto.go` | 55-59 |
| 审计插件接口 | `server/common/types.go` | 65-71 |
| 审计引擎注册 | `server/common/plugin.go` | 185-193 |
| 默认审计实现 | `server/model/audit.go` | 58-75 |
| Admin 密码验证 | `server/ctrl/admin.go` | 57-62 |
| PASSWORD_DUMMY 掩码 | `server/common/types.go` | 178 |
| FetchAuditHandler API | `server/ctrl/admin.go` | 138-158 |
| **配置变更审计对比** | | |
| IAuditPlugin 接口 | `server/common/types.go` | 65-71 |
| FetchAuditHandler | `server/ctrl/admin.go` | 138-158 |
| SimpleAudit 默认实现 | `server/model/audit.go` | 58-75 |
| flattenJSON 工具函数 | `server/common/config.go` | 221-238 |
| 前端审计模型 | `public/assets/pages/adminpage/model_audit.js` | 1-21 |
| 前端审计控制器 | `public/assets/pages/adminpage/ctrl_activity_audit.js` | 1-66 |
| **配置优先级决策** | | |
| ConfigElement.Set() | `server/common/config.go` | 429-444 |
| ConfigElement.Interface() | `server/common/config.go` | 475-486 |
| Configuration.Initialise() | `server/common/config.go` | 241-260 |
| defaultValue[T]() 泛型函数 | `server/common/config.go` | 506-522 |
| Configuration.MarshalJSON() | `server/common/config.go` | 488-504 |
| Configuration.Load() | `server/common/config.go` | 186-219 |
| Configuration.Export() | `server/common/config.go` | 286-362 |
| Default 多次设置警告 | `server/common/config.go` | 418-421 |
