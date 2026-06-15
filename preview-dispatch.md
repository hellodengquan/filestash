# 文件预览分发链路（Preview Dispatch Chain）

本文档从代码实现角度梳理 Filestash 中不同文件类型如何匹配到对应预览器的完整链路。

---

## 一、整体架构概览

预览分发是一个**后端提供元数据 + 前端执行分发**的双层架构：

```
┌─────────────────────────────────────────────────────────────────────┐
│                         后端 (Go)                                   │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────┐ │
│  │ mime.json     │  │ 插件注册系统  │  │ 配置/插件 API    │ │
│  │ (扩展名→MIME)  │  │ (XDGOpen等)  │  │ (/api/config     │ │
│  └──────┬────────┘  └──────┬───────┘  │  /api/plugin)    │ │
│         │生成代码             │注册        └────────┬───────────┘ │
│         ▼                    ▼                  ▲暴露              │
│  ┌──────────────────────────────────────────┼───────────────┐   │
│  │  common/mime.go + mime_generated.go       │               │   │
│  │  plugin.go (Hooks 系统)                  │               │   │
│  └──────────────────────────────────────────┼───────────────┘   │
└──────────────────────────────────────────────┼───────────────────┘─┘
                                           │ HTTP
┌───────────────────────────────────────────┼─────────────────────┐
│                 前端 (JS)               ▼                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  ① 获取 mime 映射 + 插件映射                     │  │
│  │  (config.js, plugin.js)                          │  │
│  └──────────────────────┬─────────────────────────────┘  │
│                         │                                   │  │
│                         ▼                                   │  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  ② opener() 分发函数                              │  │
│  │  (viewerpage/mimetype.js)                        │  │
│  └──────────────────────┬─────────────────────────────┘  │
│                         │                                   │  │
│                         ▼                                   │  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  ③ 加载对应 application_*.js 模块 (ctrl_viewerpage.js)  │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、MIME 类型定义与生成（后端）

### 2.1 数据源：`config/mime.json`

这是 MIME 类型的**唯一真相来源（Single Source of Truth）**，定义了文件扩展名到 MIME 类型的映射：

```json
{
  "jpg":  "image/jpeg",
  "pdf":  "application/pdf",
  "docx": "application/word",
  "md":   "text/markdown",
  ...
}
```

> **位置**：`config/mime.json:1-310`

### 2.2 代码生成：`server/generator/mime.go`

一个 Go 代码生成器（构建时运行）：

- 读取 `config/mime.json`
- 生成 `server/common/mime_generated.go`，在 `init()` 函数中填充全局 `MimeTypes` map

```go
// generator/mime.go:34-36
for key, value := range mTypes {
    fmt.Fprintf(&buf, "MimeTypes[\"%s\"] = \"%s\"\n", key, value)
}
```

### 2.3 运行时查询：`server/common/mime.go`

`GetMimeType(p string)` 是后端统一的 MIME 查询入口：

```go
// common/mime.go:11-22
func GetMimeType(p string) string {
    ext := filepath.Ext(p)       // 提取扩展名
    if ext != "" { ext = ext[1:] }
    ext = strings.ToLower(ext)  // 归一化为小写
    mType := MimeTypes[ext]
    if mType == "" {
        return "application/octet-stream" // 默认值
    }
    return mType
}
```

**默认值策略**：未知扩展名一律视为 `application/octet-stream`。

---

## 三、MIME 暴露给前端

### 3.1 配置导出：`server/common/config.go

`Configuration.Export()` 方法将 MIME 映射通过 `mime` 字段导出：

```go
// common/config.go:286-312
func (this *Configuration) Export() interface{} {
    return struct {
        ...
        MimeTypes map[string]string `json:"mime"`
        ...
    }{ ... }
}
```

### 3.2 配置 API：`server/ctrl/config.go`

`PublicConfigHandler` 处理 `/api/config` 请求：

```go
// ctrl/config.go:25-27
func PublicConfigHandler(ctx *App, res http.ResponseWriter, req *http.Request) {
    cfg := Config.Export()
    SendSuccessResultWithEtagAndGzip(res, req, cfg)
}
```

### 3.3 前端获取：`public/assets/model/config.js`

前端启动时通过 `/api/config` 拉取配置（含 MIME 映射）：

```javascript
// model/config.js:4-10
const config$ = ajax({
    url: "api/config",
    method: "GET",
    responseType: "json",
}).pipe(rxjs.map(({ responseJSON }) => responseJSON.result));
```

---

## 四、预览器注册的三种机制

预览器（opener）的注册有**三种并行机制**，优先级从高到低依次生效：

| 机制 | 注册方式 | 典型使用场景 | 优先级 |
|------|----------|-------------|--------|
| **A: XDGOpen JS 注入 | Go 插件调用 `Hooks.Register.XDGOpen()` | OnlyOffice、WOPI 办公文档预览 | 最高（最先匹配） |
| **B: 扩展插件 manifest** | zip 插件 `manifest.json` 声明 | docxjs、office 等第三方预览器 | 中等 |
| **C: 前端硬编码规则** | `mimetype.js` 内部分发表 | 图片、视频、PDF 等内建预览 | 最低（兜底） |

---

### 机制 A：XDGOpen JS 代码注入（Go 原生插件）

#### A.1 注册接口：`server/common/plugin.go`

```go
// plugin.go:215-221
var xdg_open []string

func (this Register) XDGOpen(jsString string) {
    xdg_open = append(xdg_open, jsString)
}
```

#### A.2 典型插件注册：OnlyOffice

```go
// plg_editor_onlyoffice/index.go:194-201
Hooks.Register.XDGOpen(`
    if(mime === "application/word" || ... ||
       mime === "application/excel" || ... ||
       mime === "application/powerpoint" || ...) {
         return ["appframe", {"endpoint": "/api/onlyoffice/iframe"}];
       }
`)
```

每个 XDGOpen 注册的是**纯 JS 代码片段，在前端组成一个完整函数体。

#### A.3 注入前端：`server/routes.go → `/overrides/xdg-open.js`

```go
// routes.go:167-175
r.HandleFunc(WithBase("/overrides/xdg-open.js"), func(res http.ResponseWriter, req *http.Request) {
    res.Header().Set("Content-Type", GetMimeType(req.URL.String()))
    res.Write([]byte(`window.overrides["xdg-open"] = function(mime){`)
    openers := Hooks.Get.XDGOpen()
    for i := 0; i < len(openers); i++ {
        res.Write([]byte(openers[i]))
    }
    res.Write([]byte(`return null;}`))
})
```

最终生成的 JS 结构为：

```javascript
window.overrides["xdg-open"] = function(mime) {
    // [plugin1 的 JS 代码]
    // [plugin2 的 JS 代码]
    // ...
    return null;  // 所有插件都不匹配时返回 null
}
```

#### A.4 前端加载：`public/assets/boot/ctrl_boot_frontoffice.js

```javascript
// ctrl_boot_frontoffice.js:37-40
async function setup_xdg_open() {
    window.overrides = {};
    return loadJS(import.meta.url, toHref("/overrides/xdg-open.js"));
}
```

---

### 机制 B：扩展插件 manifest.json（Zip 插件）

#### B.1 manifest 声明示例

`plg_application_office/manifest.json:

```json
{
  "modules": [
    {
      "type": "xdg-open",
      "mime": "application/word",
      "entrypoint": "loader_lowa.js",
      "application": "skeleton"
    },
    {
      "type": "xdg-open",
      "mime": "application/excel",
      "entrypoint": "loader_lowa.js",
      "application": "skeleton"
    }
  ]
}
```

字段说明：
- `type: "xdg-open"`：声明这是一个预览器模块
- `mime`：要处理的 MIME 类型
- `entrypoint`：插件内的 JS 入口文件
- `application`：使用哪个内建 application 作为宿主（通常是 `skeleton`）

#### B.2 后端导出：`server/ctrl/plugin.go

`PluginExportHandler` 处理 `/api/plugin`：

```go
// ctrl/plugin.go:16-33
func PluginExportHandler(ctx *App, res http.ResponseWriter, req *http.Request) {
    plgExports := map[string][]string{}
    for name, plg := range extension.All() {
        for _, module := range plg.Modules {
            if module["type"] == "xdg-open" {
                plgExports[module["mime"]] = []string{
                    module["application"],
                    WithBase(JoinPath("/assets/.../plugin/", filepath.Join(name+".zip", index))),
                }
            }
        }
    }
    SendSuccessResultWithEtagAndGzip(res, req, plgExports)
}
```

返回结果格式：`{ [mime: [applicationName, entrypointUrl]`

#### B.3 前端获取：`public/assets/model/plugin.js`

```javascript
// model/plugin.js:18-20
export function get(mime) {
    return plugins[mime];  // 直接按 MIME 查询
}
```

---

### 机制 C：前端硬编码默认规则

当机制 A、B 都不命中时，使用内建硬编码规则（兜底）。

---

## 五、前端分发核心：`opener()` 函数

**位置**：`public/assets/pages/viewerpage/mimetype.js:3-42`

这是整个预览分发的**决策中枢**，执行流程如下：

```
输入: filename (文件名) + mimes (MIME 映射表)
    │
    ▼
┌─────────────────────────────┐
│ ① 计算 MIME 类型         │
│ getMimeType(file, mimes)    │
│ 按扩展名查 mimes，        │
│ 默认 "text/plain"          │
└────────────┬─────────────┘
             │
             ▼
┌─────────────────────────────┐
│ ② 机制 A：XDGOpen 插件   │
│ window.overrides["xdg-open"] │
│ (mime) → 非 null 直接返回│
└────────────┬─────────────┘
             │ null（未匹配）
             ▼
┌─────────────────────────────┐
│ ③ 机制 B：扩展插件      │
│ getPlugin(mime)           │
│ → 命中直接返回           │
└────────────┬─────────────┘
             │ undefined（未匹配）
             ▼
┌─────────────────────────────┐
│ ④ 机制 C：硬编码规则     │
│ 按 MIME 精确/前缀匹配       │
└────────────┬─────────────┘
             │
             ▼
      [openerName, options]
```

### 5.1 硬编码规则表（机制 C 详情）

| 匹配条件 | 预览器 | 说明 |
|----------|--------|------|
| `type === "text"` | `"editor"` | 所有 text/* 系列 |
| `=== "application/pdf"` | `"pdf"` | PDF 专用 |
| `type === "image"` | `"image"` | 所有 image/* 系列 |
| `=== "application/javascript"` | `"editor"` | JS 代码 |
| `=== "application/xml"` | `"editor"` | XML |
| `=== "application/json"` | `"editor"` | JSON |
| `=== "application/x-perl"` | `"editor"` | Perl |
| `audio/wave\|mp3\|flac\|ogg\|mpeg` | `"audio"` | 音频文件 |
| `=== "application/x-form"` | `"form"` | 表单 |
| `=== "application/geo+json"` 等 | `"map"` | GIS 地图相关 |
| `type === "video"` \|\| `=== "application/ogg"` | `"video"` | 视频 |
| `=== "application/epub+zip"` | `"ebook"` | 电子书 |
| `=== "application/x-url"` | `"url"` | URL 快捷方式 |
| `type === "application"` && 非 `application/text` | `"download"` | 其他应用类型（下载） |
| **兜底 | `"editor"` | 最终兜底：所有剩余情况 |

> 示例代码：

```javascript
// mimetype.js:17-41
if (type === "text") {
    return ["editor", { mime }];
} else if (mime === "application/pdf") {
    return ["pdf", { mime }];
} else if (type === "image") {
    return ["image", { mime }];
}
// ... 更多规则 ...
return ["editor", { mime }];
```

---

## 六、预览器应用加载（ctrl_viewerpage.js）

**位置**：`public/assets/pages/ctrl_viewerpage.js:64-99`

### 6.1 主流程

```javascript
// ctrl_viewerpage.js:69-75
effect(rxjs.of(getConfig("mime", {})).pipe(
    rxjs.map((mimes) => opener(basename(getCurrentPath()), mimes)),
    rxjs.mergeMap(([openerName, opts]) =>
        rxjs.from(loadModuleWithMemory(openerName)).pipe(
            rxjs.switchMap(async (module) => {
                module.default(createRender($page), { ...opts, ... });
            })
        )
    ),
    rxjs.catchError(ctrlError()),
));
```

1. 从 config 中取 mime 映射
2. 用 `opener()` 计算 `[openerName, opts]`
3. 按名称动态加载模块
4. 调用模块的 `default()` 渲染函数

### 6.2 应用名称 → 模块映射表

| opener 名称 | 动态 import 的模块 | 功能 |
|---------------|---------------|------|
| `"editor"` | `application_editor.js` | CodeMirror 文本编辑器 |
| `"pdf"` | `application_pdf.js` | PDF.js 渲染 |
| `"image"` | `application_image.js` | 图片查看器（含分页） |
| `"download"` | `application_downloader.js` | 文件下载页 |
| `"form"` | `application_form.js` | 表单渲染器 |
| `"audio"` | `application_audio.js` | 音频播放器 |
| `"video"` | `application_video.js` | 视频播放器（含 HLS） |
| `"ebook"` | `application_ebook.js` | EPUB 阅读器 |
| `"3d"` | `application_3d.js` | Three.js 3D 模型查看 |
| `"appframe"` | `application_iframe.js` | iframe 包装器（OnlyOffice/WOPI） |
| `"map"` | `application_map.js` | Leaflet 地图 |
| `"url"` | `application_url.js` | URL 重定向 |
| `"table"` | `application_table.js` | 表格查看器 |
| `"skeleton"` | `application_skeleton.js` | 骨架加载器（扩展插件宿主） |

> 代码位置：`ctrl_viewerpage.js:19-52`

### 6.3 两种特殊宿主应用

#### 6.3.1 `appframe`（iframe 宿主）

**用途**：嵌入外部应用（如 OnlyOffice、WOPI 服务）

```javascript
// application_iframe.js:11-18
export default function(render, { endpoint = "" }) {
    const url = forwardURLParams(`${endpoint}?path=${encodeURIComponent(getCurrentPath())}`, ["share"]);
    const $page = createElement(`
        <div class="component_appframe">
            <iframe src="${url}"></iframe>
        </div>
    `);
    render($page);
    // ... postMessage 通信处理
}
```

机制 A（XDGOpen）典型返回 `["appframe", {endpoint: "..."}]` 使用此宿主。

#### 6.3.2 `skeleton`（骨架宿主）

**用途**：加载扩展插件提供的自定义渲染器

```javascript
// application_skeleton.js:11-34
export default function(render, { mime, ... }) {
    // ...
    effect(rxjs.from(loadPlugin(mime)).pipe(
        rxjs.mergeMap((loader) => {
            const opts = { mime, ... };
            if (!loader) {
                componentDownloader(...); // 插件加载失败回退到下载
                return rxjs.EMPTY;
            }
            return rxjs.from(loader(createRender($container), { $menubar, ...opts));
        }),
    )));
}
```

机制 B（扩展插件）典型返回 `["skeleton", { mime, ... }]` 使用此宿主。插件 JS 模块动态导入：

```javascript
// model/plugin.js:22-28
export async function load(mime) {
    const specs = plugins[mime];
    if (!specs) return null;
    const [, url] = specs;
    const module = await import(new URL(url, import.meta.url).href);
    return module.default;
}
```

---

## 七、关键代码索引

| 层级 | 文件 | 关键函数/结构 |
|------|------|-------------|
| MIME 数据源 | `config/mime.json` | 扩展名→MIME 映射表 |
| MIME 代码生成 | `server/generator/mime.go` | 生成 mime_generated.go |
| MIME 查询（后端） | `server/common/mime.go:11` | `GetMimeType()` |
| 插件注册中心 | `server/common/plugin.go:215` | `Register.XDGOpen() |
| XDGOpen 注入路由 | `server/routes.go:167` | `/overrides/xdg-open.js` |
| 扩展插件导出 | `server/ctrl/plugin.go:16` | `PluginExportHandler()` |
| 配置导出（含 MIME） | `server/common/config.go:286` | `Configuration.Export()` |
| 配置 API | `server/ctrl/config.go:25` | `PublicConfigHandler()` |
| 前端配置加载 | `public/assets/boot/ctrl_boot_frontoffice.js:37` | `setup_xdg_open()` |
| 前端配置模型 | `public/assets/model/config.js` | `get()` |
| 前端插件模型 | `public/assets/model/plugin.js` | `get()`, `load()` |
| **分发决策中心** | `public/assets/pages/viewerpage/mimetype.js:3` | **`opener()`** |
| 应用加载控制器 | `public/assets/pages/ctrl_viewerpage.js:19` | `loadModule()` |
| appframe 宿主 | `public/assets/pages/viewerpage/application_iframe.js:11` | iframe 包装器 |
| skeleton 宿主 | `public/assets/pages/viewerpage/application_skeleton.js:11` | 扩展插件渲染器宿主 |
