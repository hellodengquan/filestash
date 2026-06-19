# Admin 后台配置注入链路详解

本文档详细说明 Filestash 项目中 Admin 后台与配置注入的完整链路，包括配置 Schema 定义、热更新机制和敏感字段处理。

## 目录

- [1. 配置 Schema 定义](#1-配置-schema-定义)
- [2. 配置热更新机制](#2-配置热更新机制)
- [3. 敏感字段处理](#3-敏感字段处理)
- [4. Admin 后台配置注入链路](#4-admin-后台配置注入链路)
- [5. 关键代码位置索引](#5-关键代码位置索引)

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

## 5. 关键代码位置索引

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
