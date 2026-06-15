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

## 十、前端鉴权 Token 透传链路

预览文件时，身份凭证需要从前端一路穿透到后端存储。Filestash 采用**多通道、多策略**的鉴权透传方案，适配不同的客户端场景（AJAX / `<video>` / `<img>` / iframe / 第三方服务回源）。

### 10.1 Token 的来源与前端存储

**位置**：`public/assets/model/session.js:13`

`window.BEARER_TOKEN` 是前端全局的 token 容器，在三个时机被设置：

1. **会话创建时**：`createSession()` 从登录响应的 `responseHeaders.bearer` 获取
   ```javascript
   // session.js:26
   if (responseHeaders.bearer) window.BEARER_TOKEN = responseHeaders.bearer;
   ```

2. **会话恢复时**：`initSession()` 从 `/api/session` 的 `result.authorization` 获取
   ```javascript
   // session.js:12-14
   rxjs.tap(({ authorization }) => {
       if (authorization) window.BEARER_TOKEN = authorization;
   }),
   ```

3. **URL Hash 传递时**：iframe 嵌入场景通过 `#bearer=xxx` 传入
   ```javascript
   // session.js:41-45
   const token = new URLSearchParams(location.hash.replace(...)).get("bearer");
   if (token) window.BEARER_TOKEN = token;
   ```

### 10.2 AJAX 请求：Authorization Header 自动注入

**位置**：`public/assets/lib/ajax.js:9-13`

所有通过 `ajax()` 发出的请求自动附加 Bearer Token：

```javascript
// ajax.js:9-13
if (!opts.headers) opts.headers = {};
opts.headers["X-Requested-With"] = "XmlHttpRequest";
opts.headers["X-Request-ID"] = traceID();
if (window.BEARER_TOKEN) opts.headers["Authorization"] = `Bearer ${window.BEARER_TOKEN}`;
```

适用场景：文本编辑器、PDF.js、图片元数据查询等通过 JS fetch 的预览器。

### 10.3 浏览器原生元素：Cookie 分片策略

`<video>`、`<audio>`、`<img>`、`<embed>`、`<iframe>` 等**原生 HTML 元素**无法自定义 HTTP Header，必须走 Cookie 通道。

#### 10.3.1 后端：分片 Cookie 提取

**位置**：`server/middleware/session.go:160` → `_extractAuthorization()`

后端使用**四级 fallback 策略**提取 token，优先级从高到低：

```go
func _extractAuthorization(req *http.Request) (token string) {
    // 策略1：分片 Cookie（应对 4KB Cookie 限制）
    for index := 0; ; index++ {
        cookie, err := req.Cookie(CookieName(index))
        if err != nil { break }
        token += cookie.Value  // 按序号拼接
    }
    if token != "" { return token }

    // 策略2：Authorization Header (Bearer)
    authHeader := req.Header.Get("Authorization")
    if strings.HasPrefix(authHeader, "Bearer ") {
        return strings.TrimPrefix(authHeader, "Bearer ")
    }

    // 策略3：URL Query Param (?authorization=xxx)
    if auth := req.URL.Query().Get("authorization"); auth != "" {
        return auth
    }

    // 策略4：HTTP Basic Auth (username = "authorization")
    if u, p, ok := req.BasicAuth(); ok && u == "authorization" {
        return p
    }
    return ""
}
```

#### 10.3.2 为什么要用分片 Cookie？

单 Cookie 有 4KB 大小限制，而 Filestash 的 session 可能包含后端凭证（如 S3 key、OAuth token 等），体积可能超过限制。通过 `CookieName(0)`、`CookieName(1)`、`CookieName(2)`…… 拆分成多片 cookie 存储，后端按序拼接还原。

### 10.4 第三方服务回源：Token 注入与代理

WOPI / OnlyOffice 等**第三方服务**需要回源拉取文件，此时凭证需要通过服务端注入或 URL 参数传递。

| 方案 | 代表插件 | 实现方式 |
|------|---------|---------|
| **URL Query 参数** | WOPI | `wopiSRC?access_token=<authorization>`，WOPI 服务端回源时带上 |
| **服务端 Cache Key** | OnlyOffice | 预生成 `key = Hash(content+userId+path)`，存在 `onlyoffice_cache`，服务端通过 key 间接找到 path 和 Cat() |
| **path64 编码** | WOPI | path/base64 编码在 URL 路径中，配合 access_token 鉴权 |

以 WOPI 为例（`plg_editor_wopi/handler.go:277`）：
```go
wopiSRC += GenerateID(map[string]string{
    "id":   GenerateID(ctx.Session),  // session ID
    "path": fullpath,                 // 文件路径
})
wopiSRC += "::" + base64.StdEncoding.EncodeToString([]byte(fullpath))
```
`wopiToCommonAPI` 中间件解析 `path64`，将 `access_token` 映射为 `authorization` query param，再走正常鉴权链路。

### 10.5 鉴权透传全景图

```
 前端场景               传递方式                 后端接收策略
─────────────────  ───────────────────────  ──────────────────────────
 AJAX / fetch       Authorization Header       策略2: Bearer Header
 <video>/<audio>    Cookie (分片)               策略1: 分片 Cookie
 <img>/<embed>      Cookie (分片)               策略1: 分片 Cookie
 URL 直接访问       Cookie / Query Param        策略1 / 策略3
 WOPI 回源          access_token + path64       wopiToCommonAPI → 策略3
 OnlyOffice 回源    key (服务端 cache)          查 onlyoffice_cache
 iframe (appframe)  Cookie + postMessage        策略1 + JS 通信
```

---

## 十一、扩展插件运行时沙箱与安全隔离

扩展插件（机制 B，zip 格式）在前端运行时，Filestash 采用**多层防御**的隔离策略：CSP 内容安全策略 + COOP/COEP 跨域隔离 + 动态 import 作用域。

### 11.1 第一层：CSP（Content-Security-Policy）

**位置**：`server/ctrl/files.go:406-408`

所有通过 `CatHandler` 下载的文件（包括插件静态资源）响应头中默认带 CSP：

```
Content-Security-Policy: default-src 'none';
                        img-src 'self';
                        media-src 'self';
                        style-src 'unsafe-inline';
                        font-src data:;
                        script-src-elem 'self'
```

| 指令 | 值 | 含义 |
|------|-----|------|
| `default-src` | `'none'` | 默认拒绝所有资源加载（最严格） |
| `img-src` | `'self'` | 图片只能从同源加载 |
| `media-src` | `'self'` | 音视频只能从同源加载 |
| `style-src` | `'unsafe-inline'` | 允许内联样式 |
| `font-src` | `data:` | 允许 data URI 字体 |
| `script-src-elem` | `'self'` | `<script>` 只能从同源加载 |

可通过管理员配置 `features.protection.disable_csp = true` 关闭（`files.go:56-67`），但不推荐。

### 11.2 第二层：COOP + COEP（跨域隔离头）

**位置**：`server/plugin/plg_application_office/middleware.go:13-14`

`plg_application_office`（Lowa / WASM 版 LibreOffice）插件通过中间件注入额外安全头：

```go
head.Set("Cross-Origin-Opener-Policy", "same-origin")
head.Set("Cross-Origin-Embedder-Policy", "require-corp")
```

**为什么办公插件需要这两个头？**

- `COOP: same-origin`：将顶级浏览上下文与跨域文档隔离，防止 window.opener 攻击
- `COEP: require-corp`：要求所有子资源必须显式声明 CORP 头，否则被拦截
- **核心目的**：启用 `SharedArrayBuffer`，WASM 多线程渲染（soffice.js / LibreOffice WASM 构建需要 SAB 进行线程间通信）

> 这两个头是针对特定插件的（通过 middleware 注册），不是全局设置。其他预览器（图片、PDF 等）不会携带。

### 11.3 第三层：动态 import 的作用域隔离

扩展插件的 entrypoint 通过 `import(new URL(url, import.meta.url).href)` 动态加载（`model/plugin.js:26`）：

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

**隔离特点**：
- ES Module 有自己的作用域，不会污染全局变量
- 但可以访问 `window`、`document` 等浏览器 API（不是真正的沙箱）
- 插件代码与主应用在**同一个浏览上下文**中运行

### 11.4 PDF.js 的独立沙箱

`pdf.sandbox.js` 是 PDF.js 自带的计算沙箱，用于隔离 PDF 内容解析中可能的恶意脚本：

- 在 Web Worker 中运行
- 仅通过 postMessage 与主线程通信
- 阻止对 DOM 的直接访问

这是 PDF.js 库自带的安全机制，不是 Filestash 实现的。

### 11.5 插件安全层级总结

| 层级 | 机制 | 作用域 | 强度 |
|------|------|--------|------|
| L1 | CSP 内容安全策略 | 所有文件下载响应 | 中（阻止 inline script 等） |
| L2 | COOP + COEP | 办公插件（WASM） | 中高（启用跨源隔离 + SAB） |
| L3 | ES Module 作用域 | 所有前端扩展插件 | 低（仅变量隔离） |
| L4 | Worker 沙箱 | PDF.js 计算 | 高（无法访问 DOM） |

---

## 十二、跨域资源加载

Filestash 的预览链路中存在多处跨域交互：iframe 嵌入第三方办公服务、CDN 静态资源、Chromecast 投屏等。

### 12.1 appframe 宿主的跨域 iframe 通信

**位置**：`public/assets/pages/viewerpage/application_iframe.js:20-44`

`appframe` 用 iframe 嵌入外部服务（OnlyOffice / WOPI），通过 `postMessage` 跨域通信：

```javascript
// application_iframe.js:25-44
window.addEventListener("message", (event) => {
    try { msg = JSON.parse(event.data); } catch(e) { return; }
    switch (msg.type) {
    case "error":
        throw new Error(msg.msg);
    case "notify::error":
    case "notify::info":
    case "notify::success":
        // 调用全局 notification 组件展示
        ...
    }
});
```

**方向**：iframe 内部 → `window.parent` → Filestash 父窗口
**安全**：`event.origin` 未做严格校验（使用 `"*"`），因为办公服务域名由用户配置。

### 12.2 WOPI 双向跨域

WOPI 场景存在**两层跨域**：

```
┌──────────────┐     postMessage      ┌──────────────────────┐
│  Filestash   │ ◄──────────────────► │  WOPI iframe (同域)  │
│  (父窗口)    │                      │  /api/wopi/iframe    │
└──────────────┘                      └──────────┬───────────┘
                                                  │
                                                  │ HTTP 请求
                                                  ▼
                                          ┌──────────────────┐
                                          │ WOPI Server      │
                                          │ (第三方，跨域)    │
                                          └──────────────────┘
```

- **第一层**（父窗口 ↔ /api/wopi/iframe）：同域，无跨域问题
- **第二层**（WOPI 服务回源 Filestash）：跨域 HTTP，但服务端到服务端，不受浏览器 CORS 限制

### 12.3 Chromecast 投屏的跨域处理

**位置**：`public/assets/model/chromecast.js:38`

Chromecast 需要从 Google Cast SDK 加载，且播放地址需要可公开访问：

```javascript
// chromecast.js:38-39
if (!window.BEARER_TOKEN) throw new Error("Invalid account");
// TODO: it would be much much nicer to set the authorization from an HTTP header
```

当前限制：Chromecast 无法携带 Authorization Header，需要通过 URL 参数传递 token（或使用公共分享链接）。

### 12.4 CORS 配置

Filestash 后端**不主动设置** `Access-Control-Allow-Origin` 头。API 均为同源调用，跨域场景通过以下方式解决：

| 场景 | 解决方案 |
|------|---------|
| 第三方 iframe 嵌入 | appframe + postMessage |
| 静态资源 CDN | 同源 / 构建时打包到 `assets/` |
| 服务端到服务端调用 | 不受 CORS 限制（WOPI/OnlyOffice 回源） |
| Chromecast | URL 参数携带 token（TODO 优化） |

---

## 十三、大文件预览：流式传输与缓存策略

视频、PDF、大图片等大文件的预览依赖 HTTP Range Request（分块请求）和服务端缓存。

### 13.1 HTTP Range Request（字节范围请求）

**位置**：`server/ctrl/files.go:360-438`

Filestash 完整实现了 RFC 7233 的 Range 请求支持，使 `<video>`、`<audio>`、PDF.js 等客户端可以**按需拉取片段**，无需下载整个文件。

#### 13.1.1 请求处理流程

```
客户端发送 Range: bytes=100-199
            │
            ▼
    解析 Range 头（支持多段，逗号分隔）
            │
            ▼
    后端是否支持 Seek？
       ├─ 是 → 直接 Seek 到起始位置 → LimitReader
       └─ 否 → 下载整个文件到 tmp 缓存 → 从缓存 Seek
            │
            ▼
    响应 206 Partial Content
    Content-Range: bytes 100-199/12345
    Content-Length: 100
```

#### 13.1.2 两种后端适配策略

```go
// files.go:316-358
if req.Header.Get("range") != "" && needToCreateCache == true {
    if obj, ok := file.(io.Seeker); ok == true {
        // 策略1：后端原生支持 Seek（如本地文件系统）
        size, _ := obj.Seek(0, io.SeekEnd)
        obj.Seek(0, io.SeekStart)
        contentLength = size
    } else {
        // 策略2：后端不支持 Seek（如 S3 / FTP）→ 下载到本地 tmp 缓存
        tmpPath := GetAbsolutePath(TMP_PATH, "file_"+QuickString(20)+".dat")
        f, _ := os.OpenFile(tmpPath, os.O_RDWR|os.O_CREATE, os.ModePerm)
        file_cache.Set(ctx.Session, tmpPath)  // 写入 session 级缓存
        io.Copy(f, file)  // 完整下载一次
        // 后续 range 请求直接走缓存文件
    }
}
```

### 13.2 Session 级文件缓存

**位置**：`server/ctrl/files.go:37` → `file_cache AppCache`

`file_cache` 是基于 `hashstructure` 的 session 级缓存：

- **底层实现**：`github.com/patrickmn/go-cache`
- **保留时间**：5 分钟（`cache.go:52`）
- **清理间隔**：10 分钟
- **Key**：`hashstructure.Hash(ctx.Session)` — 以整个 session map 为哈希输入
- **Value**：临时文件路径（`/tmp/filestash/file_xxxxx.dat`）

**过期自动清理**：
```go
// files.go:69-71
file_cache.OnEvict(func(key string, value interface{}) {
    os.RemoveAll(filepath.Join(GetAbsolutePath(TMP_PATH), key))
})
```

### 13.3 流式分块传输

**位置**：`server/ctrl/files.go:414-438`

即使非 Range 请求，文件也通过 `io.CopyBuffer` 流式传输：

```go
buf := make([]byte, size*1024)  // 缓冲区大小
if f, ok := file.(io.ReadSeeker); ok && len(ranges) > 0 {
    // Range 请求：Seek + LimitReader
    f.Seek(ranges[0][0], io.SeekStart)
    header.Set("Content-Range", fmt.Sprintf("bytes %d-%d/%d", ...))
    res.WriteHeader(http.StatusPartialContent)
    io.CopyBuffer(res, io.LimitReader(f, ranges[0][1]-ranges[0][0]+1), buf)
} else {
    // 普通请求：整个文件流式输出
    io.CopyBuffer(res, file, buf)
}
```

**缓冲区大小**可配置（`general.buffer_size`）：
| 配置 | 大小 | 适用场景 |
|------|------|---------|
| `small` | 32 KB | 低内存环境 |
| `medium` | 128 KB | 默认 |
| `large` | 2 MB | 高带宽大文件 |

### 13.4 不同预览器的流式行为

| 预览器 | 流式方式 | 是否使用 Range | 缓存策略 |
|--------|---------|---------------|---------|
| **Video (HLS)** | HLS.js 自适应码率 | 是（HLS 分片） | 客户端缓冲 20s（`maxMaxBufferLength: 20`） |
| **Video (原生)** | `<video>` 原生流式 | 是（浏览器自动 Range） | 服务端 file_cache（不支持 Seek 的后端） |
| **Audio** | `<audio>` 原生流式 | 是 | 同 video |
| **PDF.js** | 按需拉取页面 | 是（部分浏览器/场景） | 客户端内存缓存 |
| **PDF (原生)** | `<embed>` 流式渲染 | 视浏览器而定 | 浏览器缓存 |
| **图片** | 渐进式加载 | 否（一般完整加载） | 浏览器 HTTP 缓存 |
| **文本编辑器** | 一次性加载 | 否 | 内存 |
| **3D 模型** | 一次性加载 | 否 | 内存 |

### 13.5 视频 HLS 流式细节

**位置**：`public/assets/pages/viewerpage/application_video.js:137-160`

```javascript
const hls = new Hls({
    debug: !!new URLSearchParams(location.search).get("debug"),
    manifestLoadPolicy: loadPolicy,
    maxMaxBufferLength: 20,  // 最大缓冲 20 秒
});
hls.loadSource(sources[i].src);
hls.attachMedia($video);
```

如果不是 HLS 流（`application/x-mpegURL`），则降级为原生 `<source>` 标签，交给浏览器处理 Range 请求。

---

## 十四、Service Worker 离线缓存

> **注意**：截至当前代码，SW 路由已注册但前端未主动 `navigator.serviceWorker.register()`，仅在全局错误兜底时调用 `unregister()` 清理旧 SW。预留了 `/sw.js` 路由（`server/routes.go:105`）指向 `ServeFile("/assets/")`，但仓库中无 `public/assets/sw.js` 源文件。以下分析现有 SW 基础设施和预留设计。

### 14.1 SW 路由注册

**位置**：`server/routes.go:105`

```go
r.HandleFunc(WithBase("/sw.js"), http.HandlerFunc(NewMiddlewareChain(ServeFile("/assets/"), middlewares))).Methods("GET")
```

将 `/sw.js` 映射到静态文件服务器根目录 `/assets/`，使得 SW 能被部署到网站根路径从而获得最大作用域。

### 14.2 错误兜底：SW 清理机制

**位置**：`public/assets/boot/ctrl_boot_frontoffice.js:57-63`

`window.onerror` 全局错误处理中包含 SW 清理逻辑：

```javascript
window.onerror = function(msg, url, lineNo, colNo, error) {
    report(msg, error, url, lineNo, colNo);
    $error(msg);
    if ("serviceWorker" in navigator) navigator.serviceWorker
        .getRegistrations()
        .then((registrations) => {
            for (const registration of registrations) {
                registration.unregister();
            }
        });
};
```

**设计意图**：如果旧版本 SW 导致错误，自动注销所有注册，防止离线缓存的旧版本代码持续引发问题。

### 14.3 现有离线策略现状

当前项目**未启用主动 SW 注册**，离线能力依赖：
1. 浏览器标准 HTTP 缓存（Cache-Control / ETag / Last-Modified）
2. 服务端 Session 级文件缓存（`file_cache`，见第十三章）
3. 视频 HLS.js 客户端缓冲（20s）

---

## 十五、Range 请求分段处理详解

Filestash 完整实现 RFC 7233 HTTP Range Request，支持大文件（视频、PDF、音频）的**渐进式下载**。核心在 `FileCatHandler`（`server/ctrl/files.go:193`）。

### 15.1 处理流程全景

```
客户端: Range: bytes=1048576-2097151
    │
    ▼
┌─ ① needToCreateCache 判断 ─────────────┐
│   if Range 头存在 && 后端不支持 Seek?     │
│     是 → ② 下载整个文件到 tmp 缓存        │
│     否 → ④ 直接使用后端 ReadSeeker       │
└───────────────┬─────────────────────────┘
                │
                ▼
┌─ ③ 解析 Range 头 ─────────────────────┐
│   支持多段: bytes=0-100,200-300         │
│   支持省略端点: bytes=-500 (尾部500B)   │
│   ranges = [[1048576, 2097151]]         │
└───────────────┬─────────────────────────┘
                │
                ▼
┌─ ⑤ 响应设置 ─────────────────────────┐
│   Accept-Ranges: bytes                 │
│   Content-Range: bytes 1048576-2097151/10485760 │
│   Content-Length: 1048576              │
│   HTTP 206 Partial Content             │
└───────────────┬─────────────────────────┘
                │
                ▼
┌─ ⑥ 流式分块输出 ─────────────────────┐
│   Seek(range[0][0])                   │
│   io.LimitReader + io.CopyBuffer       │
│   缓冲区: 32KB / 128KB / 2MB (可配)    │
└───────────────────────────────────────┘
```

### 15.2 后端 Seek 适配策略（步骤 ①~②）

```go
// files.go:316-358
if req.Header.Get("range") != "" && needToCreateCache == true {
    if obj, ok := file.(io.Seeker); ok == true {
        // 策略1：原生支持 Seek（如本地文件系统）
        size, _ := obj.Seek(0, io.SeekEnd)
        obj.Seek(0, io.SeekStart)
        contentLength = size
    } else {
        // 策略2：不支持 Seek（S3/FTP/SFTP 等）→ 完整下载到本地 tmp
        tmpPath := GetAbsolutePath(TMP_PATH, "file_"+QuickString(20)+".dat")
        f, _ := os.OpenFile(tmpPath, os.O_RDWR|os.O_CREATE, os.ModePerm)
        file_cache.Set(ctx.Session, tmpPath)  // 注册到 5 分钟 Session 缓存
        io.Copy(f, file)
        f.Sync()
        f.Close()
        file.Close()
        f, _ = os.OpenFile(tmpPath, os.O_RDONLY, os.ModePerm)
        // 后续所有 Range 请求都走本地缓存文件
    }
}
```

**关键设计**：`needToCreateCache` 在以下情况被设为 `true`：
- 请求携带 `range` 头（`files.go:268-270`）
- 即便是首次 Range 请求，对不支持 Seek 的后端也只下载一次

### 15.3 Range 头解析（步骤 ③）

```go
// files.go:360-383
ranges := make([][]int64, 0)
for _, r := range strings.Split(strings.TrimPrefix(req.Header.Get("range"), "bytes="), ",") {
    r = strings.TrimSpace(r)
    sides := strings.Split(r, "-")
    if start, err = strconv.ParseInt(sides[0], 10, 64); err != nil || start < 0 {
        start = 0  // 非法/缺失起点 → 从 0 开始
    }
    if end, err = strconv.ParseInt(sides[1], 10, 64); err != nil || end < start {
        end = contentLength - 1  // 非法/缺失终点 → 文件末尾
    }
    ranges = append(ranges, []int64{start, end})
}
```

**支持的 Range 格式**：
- `bytes=0-1023`：第 1 个 1KB
- `bytes=1024-`：从 1024 到末尾
- `bytes=-500`：最后 500 字节
- `bytes=0-100,200-300`：多段（当前代码只取 `ranges[0]`，其余丢弃）

> **注意**：当前实现只响应**第一段 Range**（`ranges[0]`），不返回 `multipart/byteranges`。

### 15.4 响应输出（步骤 ⑤~⑥）

```go
// files.go:412-438
header.Set("Accept-Ranges", "bytes")

if f, ok := file.(io.ReadSeeker); ok && len(ranges) > 0 {
    if _, err = f.Seek(ranges[0][0], io.SeekStart); err == nil {
        header.Set("Content-Range", fmt.Sprintf("bytes %d-%d/%d",
            ranges[0][0], ranges[0][1], contentLength))
        header.Set("Content-Length", fmt.Sprintf("%d",
            ranges[0][1]-ranges[0][0]+1))
        res.WriteHeader(http.StatusPartialContent)
        io.CopyBuffer(res, io.LimitReader(f,
            ranges[0][1]-ranges[0][0]+1), buf)
    } else {
        res.WriteHeader(http.StatusRequestedRangeNotSatisfiable)  // 416
    }
} else {
    io.CopyBuffer(res, file, buf)  // 非 Range 请求：整个文件流式输出
}
```

**缓冲区大小配置**（`general.buffer_size`）：
| 值 | 缓冲区 | 适用场景 |
|----|--------|---------|
| `small` | 32 KB | 低内存环境 |
| `medium` | 128 KB | **默认** |
| `large` | 2 MB | 高带宽大文件 |

---

## 十六、TUS 可恢复上传协议

TUS（Resumable Upload Protocol）实现位于后端 `FileSave`（`server/ctrl/files.go:479`）和前端 `workerImplFile`（`public/assets/pages/filespage/ctrl_upload.js:335`）。支持分片上传、断点续传、可选校验和。

### 16.1 协议支持能力

后端 OPTIONS 响应（`files.go:547-552`）：
```
Tus-Resumable: 1.0.0
Tus-Version: 1.0.0
Tus-Extension: creation,checksum
Tus-Checksum-Algorithm: sha1,crc32
```

### 16.2 后端状态机：5 个 HTTP 方法

| 方法 | 作用 | 关键响应头 |
|------|------|-----------|
| `OPTIONS` | 协商能力 | `Tus-Extension`, `Tus-Checksum-Algorithm` |
| `HEAD` | 查询上传进度 | `Upload-Offset`, `Upload-Length` |
| `POST` | 创建上传会话 | `201 Created`, `Location` |
| `PATCH` | 上传分片 | `204 No Content`, `Upload-Offset` |
| `POST`（无 Tus-Resumable） | 小文件直传 | `200 OK` |

### 16.3 chunkedUpload 数据结构

```go
// files.go:702-734
type chunkedUpload struct {
    fn     func(path string, file io.Reader) error  // Backend.Save
    stream *io.PipeWriter                            // 写入端
    offset uint64                                    // 已上传字节数
    size   uint64                                    // 总大小
    done   chan error                                // Backend.Save goroutine 返回值
    once   sync.Once                                 // done 只 close 一次
    mu     sync.Mutex                                // offset 读写锁
}
```

**设计模式**：`io.Pipe` + goroutine，一边通过 PATCH 向 `PipeWriter` 写，另一边 `Backend.Save` 从 `PipeReader` 读，零拷贝流式上传。

```go
// files.go:672-685
func createChunkedUploader(save func(...), path string, size uint64) *chunkedUpload {
    r, w := io.Pipe()
    done := make(chan error, 1)
    go func() {
        done <- save(path, r)  // 独立 goroutine 执行后端保存
    }()
    return &chunkedUpload{ stream: w, done: done, ... }
}
```

### 16.4 PATCH 分片上传流程

```go
// files.go:593-668
if proto == "tus" && req.Method == http.MethodPatch {
    // 校验 Content-Type: application/offset+octet-stream
    // 解析 Upload-Checksum（可选 sha1/crc32）
    // 解析 Upload-Offset

    uploader := chunkedUploadCache.Get(cacheKey).(*chunkedUpload)
    initialOffset, totalSize := uploader.Meta()

    if initialOffset != requestOffset {
        SendErrorResult(res, ErrNotValid)  // 偏移不匹配 → 拒绝
        return
    }

    // io.TeeReader 同时计算校验和
    reader := io.NopCloser(io.TeeReader(req.Body, hash))
    if err := uploader.Next(reader); err != nil { ... }

    // 校验和比对
    if expectedChecksum != hex.EncodeToString(hash.Sum(nil)) {
        SendErrorResult(res, NewError("Checksum Mismatch", 460))
        return
    }

    newOffset, _ := uploader.Meta()
    if newOffset == totalSize {
        uploader.Close()                 // 关闭 PipeWriter，触发 save() 返回
        chunkedUploadCache.Del(cacheKey) // 清理缓存
    }

    h.Set("Upload-Offset", fmt.Sprintf("%d", newOffset))
    res.WriteHeader(http.StatusNoContent)
}
```

### 16.5 上传缓存与清理

```go
// files.go:687-700
func initChunkedUploader() {
    chunkedUploadCache = NewAppCache(60*24, 1)  // 24h TTL，1min 清理
    chunkedUploadCache.OnEvict(func(key string, value interface{}) {
        c := value.(*chunkedUpload)
        c.Close()  // 缓存过期自动关闭 PipeWriter，终止上传
    })
}
```

**Cache Key 构成**：`{ path, session_hash }` — 同用户同路径恢复上传。

### 16.6 前端工作池与分片策略

**位置**：`public/assets/pages/filespage/ctrl_upload.js:100-326`

```
┌─ Upload Worker Pool (MAX_WORKERS = 4) ─┐
│  Worker 0  Worker 1  Worker 2  Worker 3 │
│    ▼         ▼         ▼         ▼      │
│  Task[] — 按序出队 — 并发执行            │
└────────────────────────────────────────┘
```

**前端分片决策**（`ctrl_upload.js:364-391`）：
```javascript
const chunkSize = getConfig("upload_chunk_size", 0) * 1024 * 1024;  // 默认 0 = 不分片
const numberOfChunks = Math.ceil(file.size / chunkSize);

if (chunkSize === 0 || numberOfChunks <= 1) {
    // 小文件：直接 POST body = file
} else {
    // 大文件：TUS 协议
    // 1. HEAD 查询是否已有上传会话
    // 2. POST 创建会话（Upload-Length: file.size）
    // 3. 循环 PATCH（Upload-Offset, Content-Type: application/offset+octet-stream）
    //    可选 Upload-Checksum: sha1 <hex>
}
```

**前端断点续传**（`ctrl_upload.js:397-412`）：
```javascript
const resp = await executeHttp.call(this, apiURL, { method: "HEAD", headers: tusHeaders });
if (file.size === parseInt(resp.headers["upload-length"])) {
    const tmp = parseInt(resp.headers["upload-offset"]);
    if (tmp > 0) { offset = tmp; uploadURL = apiURL; }  // 恢复之前的偏移
}
```

---

## 十七、Thumbnail 生成 Pipeline 与缓存

缩略图通过**插件钩子系统**实现，`FileCatHandler` 中触发。有两套机制：

| 机制 | 注册方式 | 适用 |
|------|---------|------|
| **`IThumbnailer` 接口** | `Hooks.Register.Thumbnailer(mimeType, handler)` | 视频缩略图（ffmpeg） |
| **`ProcessFileContentBeforeSend` 钩子** | `Hooks.Register.ProcessFileContentBeforeSend(fn)` | 图片转码/缩放（libvips/CGO） |

### 17.1 触发入口

**位置**：`server/ctrl/files.go:274-298`

```go
thumb := query.Get("thumbnail")
if thumb == "true" {
    fileMutation = true
    // Last-Modified / If-Modified-Since → 304 Not Modified
    for plgMType, plgHandler := range Hooks.Get.Thumbnailer() {
        if plgMType != mType { continue }           // 按 MIME 匹配
        file, err = plgHandler.Generate(file, ctx, &res, req)
        break  // 只取第一个匹配的 Thumbnailer
    }
}
// 然后执行 ProcessFileContentBeforeSend 钩子链
// （图片插件通过此钩子实现转码缩放）
```

**前端请求示例**（`filespage/thing.js:115`）：
```javascript
$img.src = "api/files/cat?path=" + encodeURIComponent(path) +
           "&thumbnail=true" + location.search.replace("?", "&");
```

### 17.2 图片缩略图：plg_image_light

**位置**：`server/plugin/plg_image_light/index.go:107-182`

注册 `ProcessFileContentBeforeSend` 钩子，通过 CGO 调用 libvips/libtranscode 进行缩放转码。

#### 处理 Pipeline

```
原始文件流
    │
    ▼
┌─ 1. MIME 过滤 ──────────────────────────┐
│   非 image/* | svg | x-icon → 跳过       │
│   thumbnail=false 且无 size → 跳过       │
│   thumbnail=false 且 gif → 跳过（保动图） │
└────────────┬─────────────────────────────┘
             ▼
┌─ 2. 构造 Transform 参数 ────────────────┐
│   thumbnail=true:  Size=300 Crop=true    │
│                   Quality=50 Exif=false   │
│                   Cache: 259200s (3d)     │
│   size=<N>:       Size=N Crop=false       │
│                   Quality=90 Exif=true    │
│                   Cache: 3600s (1h)       │
└────────────┬─────────────────────────────┘
             ▼
┌─ 3. 落盘（阻抗匹配） ──────────────────┐
│   io.Copy → /tmp/imagein_xxxxx.dat       │
│   (CGO 需要文件路径而非 Go io.Reader)    │
└────────────┬─────────────────────────────┘
             ▼
┌─ 4. RAW 预处理 ─────────────────────────┐
│   IsRaw(mType) → ExtractPreview()         │
│   (从 RAW 内嵌 JPEG 预览提取)             │
└────────────┬─────────────────────────────┘
             ▼
┌─ 5. 终态缩放 ──────────────────────────┐
│   CreateThumbnail(transform)             │
│   仅处理 jpeg/png/gif/tiff               │
└──────────────────────────────────────────┘
```

#### 可配置项

| 配置键 | 默认值 | 含义 |
|--------|--------|------|
| `features.image.enable_image` | `true` | 总开关 |
| `features.image.thumbnail_size` | `300` | 缩略图边长（px） |
| `features.image.thumbnail_quality` | `50` | 缩略图 JPEG 质量（0-100） |
| `features.image.thumbnail_caching` | `259200`（3天） | 浏览器缓存缩略图 |
| `features.image.image_quality` | `90` | 全图转码质量 |
| `features.image.image_caching` | `3600`（1h） | 浏览器缓存全图 |

**Cache-Control 输出**（`index.go:137-139`）：
```go
if query.Get("thumbnail") == "true" {
    (*res).Header().Set("Cache-Control", fmt.Sprintf("max-age=%d", thumb_caching()))
}
```

### 17.3 视频缩略图：plg_video_thumbnail（ffmpeg）

**位置**：`server/plugin/plg_video_thumbnail/index.go:34-80`

通过 `Hooks.Register.Thumbnailer("video/mp4", &ffmpegThumbnail{})` 注册，支持 `video/mp4`、`video/x-matroska`、`video/x-msvideo`。

#### 生成流程

```
GET /api/files/cat?path=/movie.mp4&thumbnail=true
    │
    ▼
┌─ 1. 缓存命中检测 ─────────────────────┐
│   cachePath = data/cache/video-thumbnail/ │
│               thumb_<session>_<hash(path)>.jpeg │
│   文件存在 → 直接返回                    │
└────────────┬─────────────────────────────┘
             │ 未命中
             ▼
┌─ 2. ffmpeg 命令执行 ─────────────────┐
│   ffmpeg                                 │
│     -headers "cookie: <session_cookies>" │  ← 鉴权
│     -skip_frame nokey                     │
│     -i http://127.0.0.1:<port>/api/files/cat?path=... │
│                                                ↑ 自调用拿源文件
│     -vf "thumbnail,scale=320:320:force_original_aspect_ratio=decrease" │
│     -frames:v 1                              │
│     -c:v mjpeg cachePath                     │
└────────────┬─────────────────────────────┘
             ▼
┌─ 3. 返回 + 清理 ───────────────────────┐
│   setHeader(res):                        │
│     Content-Type: image/jpeg             │
│     Cache-Control: max-age=2592000 (30d) │
│     ETag: base64(hash)                   │
│   plugin 启动时/缓存过期时 os.RemoveAll   │
└──────────────────────────────────────────┘
```

**关键技巧**：ffmpeg 无法携带 Authorization Header，但可以通过 `-headers "cookie: ..."` 携带 Cookie。Filestash 后端鉴权支持 Cookie 通道（见第十章），因此 ffmpeg 通过自调用 `/api/files/cat` 即可获得原始视频流。

### 17.4 缩略图缓存层级总结

| 缓存层级 | 位置 | TTL | 触发条件 |
|---------|------|-----|---------|
| **L1 浏览器** | `Cache-Control: max-age` | 缩略图 3d，全图 1h，视频缩略图 30d | 相同 URL + HTTP 缓存 |
| **L2 磁盘（视频）** | `data/cache/video-thumbnail/thumb_*.jpeg` | 进程生命周期（启动时清理） | 同 session 同路径 |
| **L2 磁盘（Range）** | `/tmp/file_*.dat` | 5 min（Session 级） | Range 请求 + 后端不支持 Seek |
| **L3 If-Modified-Since** | `Last-Modified` 对比 | 文件修改即失效 | 同文件未修改 |
| **L4 ETag** | 校验和对比（部分场景） | 内容变化即失效 | ETag 不变 → 304 |

---

## 十八、关键代码索引

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
| Session 鉴权中间件 | `server/middleware/session.go:57` | `SessionStart()`, `_extractAuthorization()` |
| Token 前端存储 | `public/assets/model/session.js:13` | `window.BEARER_TOKEN` |
| AJAX 鉴权注入 | `public/assets/lib/ajax.js:9` | 自动附加 `Authorization: Bearer` |
| 文件下载（Cat） | `server/ctrl/files.go:200` | `FileCatHandler()` — Range 请求、缓存、流式传输 |
| Range 请求处理 | `server/ctrl/files.go:360` | Range 解析、206 Partial Content |
| Session 级文件缓存 | `server/ctrl/files.go:37` | `file_cache AppCache` (5 min TTL) |
| AppCache 实现 | `server/common/cache.go:11` | `AppCache` 结构体、`NewAppCache()` |
| CSP 响应头 | `server/ctrl/files.go:406` | `Content-Security-Policy` 设置 |
| CSP 开关配置 | `server/ctrl/files.go:56` | `disable_csp()` |
| COOP/COEP 中间件 | `server/plugin/plg_application_office/middleware.go:13` | `Cross-Origin-Opener-Policy`, `Cross-Origin-Embedder-Policy` |
| 视频播放器 | `public/assets/pages/viewerpage/application_video.js:24` | HLS.js + 原生 `<video>` |
| PDF 预览器 | `public/assets/pages/viewerpage/application_pdf.js:16` | 原生 `<embed>` / PDF.js 双模式 |
| Chromecast 投屏 | `public/assets/model/chromecast.js:38` | 跨域播放鉴权 TODO |
| SW 路由注册 | `server/routes.go:105` | `/sw.js` 路由映射 |
| SW 错误兜底 | `public/assets/boot/ctrl_boot_frontoffice.js:57` | `window.onerror` 自动注销 SW |
| Range 适配策略 | `server/ctrl/files.go:316` | Seek 支持检测 + tmp 缓存 |
| Range 头解析 | `server/ctrl/files.go:360` | `bytes=a-b,c-d` 多段解析 |
| Range 响应输出 | `server/ctrl/files.go:412` | 206 Partial Content + `io.CopyBuffer` |
| TUS 上传 handler | `server/ctrl/files.go:479` | `FileSave()` — 5 种 HTTP 方法状态机 |
| TUS chunkedUpload 结构体 | `server/ctrl/files.go:702` | `io.Pipe` + goroutine 流式上传 |
| TUS 缓存初始化 | `server/ctrl/files.go:687` | `initChunkedUploader()` — 24h TTL + OnEvict 清理 |
| 前端上传 Worker Pool | `public/assets/pages/filespage/ctrl_upload.js:100` | `MAX_WORKERS = 4` 并发上传 |
| 前端 TUS 分片实现 | `public/assets/pages/filespage/ctrl_upload.js:335` | `workerImplFile` — HEAD/POST/PATCH + 断点续传 |
| Thumbnailer 接口 | `server/common/types.go:61` | `IThumbnailer.Generate()` |
| Thumbnailer 注册中心 | `server/common/plugin.go:172` | `thumbnailer map[string]IThumbnailer` |
| 缩略图触发入口 | `server/ctrl/files.go:274` | `?thumbnail=true` → 遍历 Thumbnailer + ProcessFileContentBeforeSend |
| 图片缩略图 Pipeline | `server/plugin/plg_image_light/index.go:107` | `ProcessFileContentBeforeSend` → CGO libvips 缩放 |
| 图片缩略图 Transform | `server/plugin/plg_image_light/index.go:185` | `Transform` 结构体 |
| 视频缩略图 ffmpeg | `server/plugin/plg_video_thumbnail/index.go:47` | `Hooks.Register.Thumbnailer("video/mp4", ...)` |
| 视频缩略图缓存 | `server/plugin/plg_video_thumbnail/index.go:17` | `VideoCachePath = "data/cache/video-thumbnail/"` |
| 前端缩略图请求 | `public/assets/pages/filespage/thing.js:115` | `?path=...&thumbnail=true` |
