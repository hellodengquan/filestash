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
            <iframe style="width:100%;height:100%" src="${url}" scrolling="no"></iframe>
        </div>
    `);
    render($page);
    // ... postMessage 通信处理
}
```

机制 A（XDGOpen）典型返回 `["appframe", {endpoint: "..."}]` 使用此宿主。

**iframe ↔ 父窗口 postMessage 协议**（`application_iframe.js:20-44`）：
- iframe 通过 `window.parent.postMessage(JSON.stringify({type, msg}), "*")` 向父窗口发消息
- 父窗口监听 `message` 事件，解析并分发：
  - `type: "error"` → 抛出异常
  - `type: "notify::error/info/success"` → 调用全局 `notification` 组件

---

#### 6.3.2 `skeleton`（骨架宿主）详解

**用途**：为扩展插件提供统一的渲染容器（menubar + 内容区 + 加载态），插件只需关注业务渲染。

##### 6.3.2.1 宿主渲染结构

```
┌─ component_skeletonviewer ─────────────────────────────┐
│  <component-menubar>          ← 顶部菜单栏              │
│    ├── .titlebar (文件名)                             │
│    └── .action-item (按钮区，由插件 .add() 追加)       │
│                                                          │
│  .component_skeleton_container  ← 插件渲染挂载点       │
│    (加载期间显示 <component-loader>)                   │
└──────────────────────────────────────────────────────────┘
```

**代码位置**：`application_skeleton.js:12-21`

```javascript
const $page = createElement(`
    <div class="component_skeletonviewer">
        <component-menubar filename="${safe(getFilename())}" class="${!hasMenubar && "hidden"}"></component-menubar>
        <div class="component_skeleton_container flex"></div>
    </div>
`);
const $menubar = renderMenubar(qs($page, "component-menubar"));
const $container = qs($page, ".component_skeleton_container");
```

##### 6.3.2.2 宿主注入给插件的上下文

skeleton 宿主调用插件 loader 时传入的参数结构：

```javascript
// application_skeleton.js:22-29
effect(rxjs.from(loadPlugin(mime)).pipe(
    rxjs.mergeMap((loader) => {
        const opts = {
            mime,                  // 文件 MIME 类型
            acl$,                  // 权限流 (GET/PUT/POST)
            getFilename,           // () => string，获取文件名
            getDownloadUrl,        // (withName?) => string，获取下载 URL
            $menubar,              // ComponentMenubar 实例，插件可 .add(button)
        };
        if (!loader) {
            componentDownloader(render, opts); // 失败回退：下载页
            return rxjs.EMPTY;
        }
        return rxjs.from(loader(createRender($container), opts));
    }),
));
```

##### 6.3.2.3 插件 loader 契约（接口规范）

Zip 插件的 entrypoint（如 `loader_docx.js`、`loader_lowa.js`）必须默认导出一个渲染函数：

```javascript
export default async function(render, opts) {
    // render:       createRender($container) 的返回值 → 向内容区挂载 DOM
    // opts:         { mime, acl$, getFilename, getDownloadUrl, $menubar }
    //   - opts.$menubar.add($button) 可向菜单栏追加按钮
}
```

两个典型插件的实现：

| 插件 | entrypoint | 渲染方式 | 交互 |
|------|-----------|---------|------|
| `plg_application_docxjs` | `loader_docx.js` | 调用 `window.docx.renderAsync(arrayBuffer, $page)` | 向 $menubar 追加下载按钮；失败回退下载页 |
| `plg_application_office` | `loader_lowa.js` | Emscripten WASM + `<canvas>` (soffice.js) | 向 $menubar 追加粗体/斜体/对齐/字号等工具栏；通过 `port.postMessage` 与 WASM 通信；保存到后端 |

以 `loader_docx.js:11-24` 为例：
```javascript
export default async function(render, { getDownloadUrl, getFilename, $menubar, acl$ }) {
    const $page = createElement(`<div class="component_docx"></div>`);
    render($page);
    $menubar.add(buttonDownload(getDownloadUrl()));   // 向菜单栏加按钮

    const removeLoader = createLoader($page);
    effect(ajax({ url: getDownloadUrl(), responseType: "arraybuffer" }).pipe(
        removeLoader,
        rxjs.mergeMap(async ({ response }) => renderDocx(response, $page)),
        rxjs.catchError(() => ctrlDownloader(render, { acl$, getFilename, getDownloadUrl, hasMenubar: false })),
    ));
}
```

---

## 七、扩展插件完整链路：从 Zip 到渲染

扩展插件（机制 B）是一个以 `.zip` 形式存在的完整模块，涉及后端发现、注册、导出、前端加载、skeleton 渲染 5 个阶段。

### 7.1 阶段一：插件发现与注册（后端启动时）

**入口**：`server/pkg/extension/discovery.go:15` → `Discovery()`

扫描 `PLUGIN_PATH` 目录下所有 `*.zip` 文件：

```go
// discovery.go:15-81
func Discovery() error {
    entries, _ := os.ReadDir(GetAbsolutePath(PLUGIN_PATH))
    for _, entry := range entries {
        if entry.IsDir() || !strings.HasSuffix(entry.Name(), ".zip") {
            continue
        }
        name, impl, err := initModule(entry.Name()) // 读取 manifest.json
        // 针对每个 module，按 type 分发到对应 Hooks：
        for i := 0; i < len(impl.Modules); i++ {
            switch impl.Modules[i]["type"] {
            case "css":         Hooks.Register.CSS(string(b))
            case "patch":       Hooks.Register.StaticPatch(b)
            case "favicon":     Hooks.Register.Favicon(b)
            case "middleware":  Hooks.Register.Middleware(adapter.MiddlewareExtension(b))
            case "workflow::action": Hooks.Register.WorkflowAction(...)
            case "xdg-open":    // noop：这里不处理，留待 /api/plugin 导出
            }
        }
        plugins[name] = impl
    }
}
```

**manifest.json 解析**：`discovery.go:112-141` → `initModule()`
- 打开 zip，读取根目录下 `manifest.json`
- 解析到 `PluginImpl{Author, Version, Modules[]}`

### 7.2 阶段二：/api/plugin 导出

**入口**：`server/ctrl/plugin.go:16` → `PluginExportHandler()`

遍历已注册插件，只筛选 `type: "xdg-open"` 的模块暴露给前端：

```go
// ctrl/plugin.go:16-33
func PluginExportHandler(ctx *App, res http.ResponseWriter, req *http.Request) {
    plgExports := map[string][]string{}
    for name, plg := range extension.All() {
        for _, module := range plg.Modules {
            if module["type"] == "xdg-open" {
                index := module["entrypoint"]
                if index == "" { index = "/index.js" }
                plgExports[module["mime"]] = []string{
                    module["application"],                                              // 宿主名，通常 "skeleton"
                    WithBase(JoinPath("/assets/"+BUILD_REF+"/plugin/",
                        filepath.Join(name+".zip", index))),                           // entrypoint 的可访问 URL
                }
            }
        }
    }
    SendSuccessResultWithEtagAndGzip(res, req, plgExports)
}
```

**返回格式示例**：
```json
{
  "application/word":  ["skeleton", "/assets/xxx/plugin/plg_application_office.zip/loader_lowa.js"],
  "application/excel": ["skeleton", "/assets/xxx/plugin/plg_application_office.zip/loader_lowa.js"]
}
```

### 7.3 阶段三：前端拉取插件映射

`public/assets/model/plugin.js` 在应用启动时调用 `init()`：

```javascript
// plugin.js:4-16
const plugin$ = ajax({
    url: "api/plugin",
    method: "GET",
    responseType: "json",
}).pipe(rxjs.map(({ responseJSON }) => responseJSON.result));

let plugins = {};

export async function init() {
    plugins = await plugin$.toPromise();
}
```

### 7.4 阶段四：分发命中 → 进入 skeleton 宿主

`opener()` 中（`mimetype.js:14`）：
```javascript
const p = getPlugin(mime);  // model/plugin.js:get()
if (p) return [p[0], { mime, ...p[1] }];
```
返回如 `["skeleton", { mime: "application/word" }]`，ctrl_viewerpage 动态加载 `application_skeleton.js`。

### 7.5 阶段五：skeleton 宿主动态加载插件 entrypoint

```javascript
// model/plugin.js:22-28
export async function load(mime) {
    const specs = plugins[mime];
    if (!specs) return null;
    const [, url] = specs;   // 取 entrypoint URL
    const module = await import(new URL(url, import.meta.url).href);
    return module.default;   // 返回插件的 default 渲染函数
}
```

插件静态资源通过 `PluginStaticHandler`（`server/routes.go:100`）从 zip 中按需读取并支持 gzip/brotli 压缩。

---

## 八、appframe 宿主：WOPI 嵌入与 Handshake 流程

WOPI（Web Application Open Platform Interface）是微软 Office Online Server / LibreOffice Online / Collabra Online 使用的协议。Filestash 通过 `plg_editor_wopi` 插件接入。

### 8.1 WOPI 插件注册（机制 A：XDGOpen）

```go
// plg_editor_wopi/index.go:7-17
func init() {
    Hooks.Register.Onload(func() {
        server_url(); origin(); rewrite_url();
        if plugin_enable() {
            Hooks.Register.XDGOpen(WOPIOverrides)
        }
    });
    Hooks.Register.HttpEndpoint(WOPIRoutes);
}
```

`WOPIOverrides` 将 9 种办公文档 MIME 指向 `["appframe", {"endpoint": "/api/wopi/iframe"}]`：

```go
// plg_editor_wopi/handler.go:36-43
var WOPIOverrides = `
    if (mime === "application/word" || mime === "application/msword" ||
        mime === "application/vnd.oasis.opendocument.text" || ...) {
        return ["appframe", {"endpoint": "/api/wopi/iframe"}];
    }
`
```

### 8.2 WOPI 路由注册

```go
// plg_editor_wopi/handler.go:22-34
var WOPIRoutes = func(r *mux.Router) error {
    r.HandleFunc(WithBase("/api/wopi/iframe"), IframeContentHandler...).Methods("GET")
    r.HandleFunc(WithBase("/api/wopi/files/{path64}"), WOPIHandler_CheckFileInfo).Methods("GET")
    r.HandleFunc(WithBase("/api/wopi/files/{path64}/contents"), WOPIHandler_GetFile).Methods("GET")
    r.HandleFunc(WithBase("/api/wopi/files/{path64}/contents"), WOPIHandler_PutFile).Methods("POST")
    return nil;
}
```

### 8.3 WOPI 完整 Handshake 时序

```
 浏览器                        Filestash (Go)                WOPI 服务端 (LibreOffice Online)
    │                               │                               │
    │ 1. GET /view/path/to.docx     │                               │
    │ ──────────────────────────►  │                               │
    │                               │                               │
    │    appframe 渲染 iframe:      │                               │
    │    GET /api/wopi/iframe       │                               │
    │ ──────────────────────────►  │                               │
    │                               │ 2. GET /hosting/discovery     │
    │                               │ ────────────────────────────► │
    │                               │                               │  返回 XML（各扩展名对应的编辑 URL）
    │                               │ ◄──────────────────────────── │
    │                               │                               │
    │                               │ 3. 构造 WOPISrc 路径：
    │                               │    /api/wopi/files/{sessionId}::{b64(path)}[::{shareId}]
    │                               │                               │
    │ 4. 返回 POST 表单页面          │                               │
    │ ◄──────────────────────────  │                               │
    │    <form action={WOPI编辑URL} │                               │
    │      access_token=...>        │                               │
    │    <script>form.submit()</>   │                               │
    │                               │                               │
    │ 5. 自动 POST 到 WOPI 服务端    │                               │
    │ ───────────────────────────────────────────────────────────► │
    │                               │                               │
    │                               │ 6. WOPI 服务端回源：           │
    │                               │    GET /api/wopi/files/{id}    │ (CheckFileInfo)
    │                               │ ◄──────────────────────────── │
    │                               │ ────────────────────────────► │
    │                               │                               │
    │                               │ 7. GET /api/wopi/files/{id}/contents
    │                               │ ◄──────────────────────────── │
    │                               │    (读取原始文件流)            │
    │                               │ ────────────────────────────► │
    │                               │                               │
    │                               │                               │
    │ 8. postMessage Handshake       │                               │
    │ ◄─────────────────────────────────────────────────────────── │
    │    App_LoadingStatus /        │                               │
    │    Initialized → 回应 Host_PostmessageReady │                  │
    │ ───────────────────────────────────────────────────────────► │
    │                               │                               │
    │                               │ 9. 用户编辑 → 保存时：          │
    │                               │    POST /api/wopi/files/{id}/contents
    │                               │ ◄──────────────────────────── │
    │                               │    (写入后端存储)              │
```

#### 8.3.1 Step 1~3：IframeContentHandler 与 Discovery

`plg_editor_wopi/handler.go:139` → `IframeContentHandler`：

1. 调用 `wopiDiscovery(ctx, path)` 获取 WOPI 编辑 URL
2. 渲染一个包含 `<form>+<iframe name="wopi_frame">` 的页面
3. 页面通过 `document.getElementById("wopi_form").submit()` 自动 POST 到 WOPI 服务端

`wopiDiscovery` 关键步骤（`handler.go:226-298`）：
- GET `{office_server}/hosting/discovery` 获取 XML 描述
- 按文件扩展名匹配 `<action ext="docx" urlsrc="...">`
- 构造 WOPISrc：`{filestash_server}/api/wopi/files/{id}::{b64(path)}[::{shareId}]`
- 拼到最终 URL：`{urlsrc}?WOPISrc=...&lang=...`

#### 8.3.2 Step 6~7：WOPI REST 三接口 + path64 解包

三个 REST 接口统一走 `WOPIExecute`（`handler.go:88-102`），通过中间件 `wopiToCommonAPI` 将 WOPI 风格参数转换为 Filestash 内部参数：

```go
// handler.go:104-137
func wopiToCommonAPI(fn HandlerFunc) HandlerFunc {
    extractInfo := func(encodedString string) (path string, shareID string) {
        tmp := strings.Split(encodedString, "::") // backendID::b64(path)::shareID
        bpath, _ := base64.StdEncoding.DecodeString(tmp[1])
        return string(bpath), (len(tmp) > 2 ? tmp[2] : "")
    }
    return HandlerFunc(func(ctx *App, res http.ResponseWriter, req *http.Request) {
        path, shareID := extractInfo(mux.Vars(req)["path64"])
        urlQuery := req.URL.Query()
        urlQuery.Set("path", path)
        if shareID != "" {
            urlQuery.Set("share", shareID)
            urlQuery.Del("access_key")
        } else {
            urlQuery.Set("authorization", urlQuery.Get("access_token"))
        }
        req.URL.RawQuery = urlQuery.Encode()
        fn(ctx, res, req); // 转到正常后端接口
    })
}
```

三接口职责：

| 路由 | 方法 | WOPI 语义 | 实现 |
|------|------|----------|------|
| `/api/wopi/files/{path64}` | GET | CheckFileInfo | 返回 `{BaseFileName, UserCanWrite, ...}` JSON |
| `/api/wopi/files/{path64}/contents` | GET | GetFile | `ctx.Backend.Cat(path)` 流式输出 |
| `/api/wopi/files/{path64}/contents` | POST | PutFile | `ctx.Backend.Save(path, body)` 写入 |

#### 8.3.3 Step 8：postMessage Handshake

WOPI iframe 通过 postMessage 与父窗口（Filestash 的 appframe）通信。`IframeContentHandler` 返回的 HTML 内嵌如下脚本：

```javascript
// handler.go:164-189 (template)
window.addEventListener("message", (event) => {
    let msg = JSON.parse(event.data);
    switch(msg.MessageId) {
    case "App_LoadingStatus":
        if (["Initialized", "Document_Loaded"].indexOf(msg.Values.Status) !== -1) {
            postChild({ MessageId: "Host_PostmessageReady" });  // ← 回应 WOPI 协议
            requestAnimationFrame(() => $iframe.classList.remove("hidden"));
            document.querySelector("component-loader").remove();
        }
        break;
    case "Action_Load_Resp":
        if (msg.Values.errorMsg) {
            postParent({ type: "error", msg: msg.Values.errorMsg });
        }
        break;
    }
});
```

收到 `Initialized` / `Document_Loaded` 后：
1. 向 WOPI 子 iframe 回复 `Host_PostmessageReady`（WOPI 标准要求）
2. 移除隐藏 class，淡入显示编辑器
3. 移除 loader

---

## 九、appframe 宿主：OnlyOffice 嵌入流程

OnlyOffice 通过 `plg_editor_onlyoffice` 插件接入，与 WOPI 不同的是：它不需要 discovery 步骤，直接用 DocsAPI 初始化编辑器，并通过 callback URL 异步保存。

### 9.1 OnlyOffice 插件注册（机制 A：XDGOpen）

```go
// plg_editor_onlyoffice/index.go:194-201
Hooks.Register.XDGOpen(`
    if(mime === "application/word" || mime === "application/msword" ||
       mime === "application/vnd.oasis.opendocument.text" || ...) {
          return ["appframe", {"endpoint": "/api/onlyoffice/iframe"}];
       }
`)
```

### 9.2 OnlyOffice 路由

```go
// plg_editor_onlyoffice/index.go:179-193
Hooks.Register.HttpEndpoint(func(r *mux.Router) error {
    oods := r.PathPrefix("/onlyoffice").Subrouter()
    oods.PathPrefix("/static/").HandlerFunc(StaticHandler)   // 反向代理 OnlyOffice 静态资源
    oods.HandleFunc("/event", OnlyOfficeEventHandler)         // OnlyOffice 保存回调
    oods.HandleFunc("/content", FetchContentHandler)          // OnlyOffice 拉取原始文件
    r.HandleFunc(COOKIE_PATH+"onlyoffice/iframe", IframeContentHandler)
    return nil
})
```

### 9.3 OnlyOffice 完整时序

```
 浏览器                         Filestash (Go)                       OnlyOffice Server
    │                               │                                      │
    │ 1. 进入 /view/xxx.docx         │                                      │
    │ ────────────────────────────► │                                      │
    │                               │                                      │
    │  appframe 加载 iframe:         │                                      │
    │  GET /api/onlyoffice/iframe    │                                      │
    │ ────────────────────────────► │                                      │
    │                               │ 2. Backend.Cat(path) 读文件流         │
    │                               │ 3. 计算 key = Hash(content+userId+path) │
    │                               │ 4. cache[key] = {path, Save, Cat}    │
    │                               │                                      │
    │ 5. 返回包含 DocsAPI 初始化 HTML  │                                      │
    │ ◄──────────────────────────── │                                      │
    │    <script src="/onlyoffice/static/.../api.js"></script>              │
    │    new DocsAPI.DocEditor("placeholder", config) │                     │
    │                               │                                      │
    │                               │ 6. 静态资源反向代理                   │
    │ ◄─────────────────────────────────────────────────────────────────── │
    │                               │                                      │
    │                               │ 7. OnlyOffice 拉取文件：              │
    │                               │    GET /onlyoffice/content?key=...    │
    │                               │ ◄──────────────────────────────────── │
    │                               │   从 cache 取 Cat()，回传文件流       │
    │                               │ ────────────────────────────────────► │
    │                               │                                      │
    │                               │ 8. 用户编辑后 OnlyOffice 回调保存：    │
    │                               │    POST /onlyoffice/event             │
    │                               │ ◄──────────────────────────────────── │
    │                               │   status=6 → 从 event.Url 下载新内容  │
    │                               │   → cache[key].Save(path, body)       │
    │                               │   → 200 {"error":0}                   │
    │                               │ ────────────────────────────────────► │
```

#### 9.3.1 IframeContentHandler：构造 DocsAPI 配置

`plg_editor_onlyoffice/index.go:255`

关键配置字段（通过 Go template 渲染到 HTML）：

```go
// index.go:408-493 模板渲染
tmpl.Execute(res, map[string]interface{}{
    "base":         filestashServerLocation,   // OnlyOffice 可访问到的 Filestash 地址
    "contentType":  contentType,               // "text" | "spreadsheet" | "presentation"
    "device":       oodsDevice,                // "desktop" | "mobile" | "embedded"
    "filename":     filename,
    "filetype":     filetype,                  // 扩展名
    "key":          key,                       // 文件唯一标识
    "mode":         oodsMode,                  // "view" | "edit"
    "userID":       userId,
    "userName":     username,
    "can_edit":     can_edit(),
    "can_download": can_download(),
    // ...
    "document.url":      "{{ .base }}/onlyoffice/content?key={{ .key }}",
    "editorConfig.callbackUrl": "{{ .base }}/onlyoffice/event",
})
```

#### 9.3.2 OnlyOfficeEventHandler：异步保存回调

OnlyOffice 服务端在文档状态变更时 POST JSON 到 `/onlyoffice/event`，`status` 字段含义：

| status | 含义 | Filestash 动作 |
|--------|------|--------------|
| 1 | 正在编辑 | - |
| 2 | 准备好保存 | - |
| 3 | 保存出错 | Warning 日志 |
| 4 | 关闭无修改 | - |
| 6 | **文档已保存** | GET `event.Url` 下载新文件 → `cache[key].Save(path, body)` 写回后端 |
| 7 | 强制保存出错 | Warning 日志 |

```go
// index.go:576-604 (status == 6)
case 6:
    saveObject, found := onlyoffice_cache.Get(event.Key)
    cData := saveObject.(*onlyOfficeCacheData)
    r, _ := http.NewRequest("GET", event.Url, nil)
    f, _ := HTTPClient().Do(r)
    cData.Save(cData.Path, f.Body)  // 写回后端存储
    res.Write([]byte(`{"error": 0}`))
```

### 9.4 WOPI vs OnlyOffice 对比

| 维度 | WOPI | OnlyOffice |
|------|------|-----------|
| 文档加载 | WOPI REST (CheckFileInfo → GetFile) | DocsAPI + 预生成 key → `/onlyoffice/content` |
| 保存方式 | WOPI REST (PutFile，同步 HTTP) | 异步回调 `/onlyoffice/event` (status=6) |
| 发现机制 | `/hosting/discovery` XML 解析 | 无需（直接配置服务器地址） |
| 前端 Handshake | postMessage (`Host_PostmessageReady`) | DocsAPI.DocEditor 构造函数 |
| 认证 | access_token 注入 WOPISrc URL | key 在服务端 cache 中映射 path |
| 适用服务端 | LibreOffice Online, Collabra Online | OnlyOffice Docs |

---

## 十、关键代码索引

| 层级 | 文件 | 关键函数/结构 |
|------|------|-------------|
| MIME 数据源 | `config/mime.json` | 扩展名→MIME 映射表 |
| MIME 代码生成 | `server/generator/mime.go` | 生成 mime_generated.go |
| MIME 查询（后端） | `server/common/mime.go:11` | `GetMimeType()` |
| 插件注册中心 | `server/common/plugin.go:215` | `Register.XDGOpen()` |
| XDGOpen 注入路由 | `server/routes.go:167` | `/overrides/xdg-open.js` |
| 扩展插件导出 | `server/ctrl/plugin.go:16` | `PluginExportHandler()` |
| 扩展插件发现 | `server/pkg/extension/discovery.go:15` | `Discovery()`, `initModule()` |
| 扩展插件索引 | `server/pkg/extension/index.go` | `All()` → `plugins` map |
| 配置导出（含 MIME） | `server/common/config.go:286` | `Configuration.Export()` |
| 配置 API | `server/ctrl/config.go:25` | `PublicConfigHandler()` |
| 前端配置加载 | `public/assets/boot/ctrl_boot_frontoffice.js:37` | `setup_xdg_open()` |
| 前端配置模型 | `public/assets/model/config.js` | `get()` |
| 前端插件模型 | `public/assets/model/plugin.js` | `get()`, `load()`, `init()` |
| **分发决策中心** | `public/assets/pages/viewerpage/mimetype.js:3` | **`opener()`** |
| 应用加载控制器 | `public/assets/pages/ctrl_viewerpage.js:19` | `loadModule()` |
| appframe 宿主 | `public/assets/pages/viewerpage/application_iframe.js:11` | iframe 包装器 + postMessage 协议 |
| skeleton 宿主 | `public/assets/pages/viewerpage/application_skeleton.js:11` | 扩展插件渲染器宿主 |
| 菜单栏组件 | `public/assets/pages/viewerpage/component_menubar.js:10` | `ComponentMenubar`, `buttonDownload()`, `renderMenubar()` |
| 通用工具函数 | `public/assets/pages/viewerpage/common.js` | `getCurrentPath()`, `getDownloadUrl()`, `getFilename()` |
| 扩展插件示例 (docxjs) | `server/plugin/plg_application_docxjs/loader_docx.js:11` | 插件 loader 契约示例 |
| 扩展插件示例 (lowa) | `server/plugin/plg_application_office/loader_lowa.js:22` | WASM Canvas 办公文档编辑器 |
| WOPI 插件入口 | `server/plugin/plg_editor_wopi/index.go:7` | `init()` → XDGOpen 注册 + 路由绑定 |
| WOPI iframe/REST 处理器 | `server/plugin/plg_editor_wopi/handler.go:22` | `WOPIRoutes`, `IframeContentHandler`, `wopiDiscovery`, `wopiToCommonAPI` |
| WOPI 配置项 | `server/plugin/plg_editor_wopi/config.go:10` | `plugin_enable()`, `server_url()`, `origin()`, `rewrite_url()` |
| OnlyOffice 插件入口 | `server/plugin/plg_editor_onlyoffice/index.go:179` | 路由、XDGOpen、模板渲染 |
| OnlyOffice 保存回调 | `server/plugin/plg_editor_onlyoffice/index.go:576` | `OnlyOfficeEventHandler` (status=6 持久化) |
