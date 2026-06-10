# PocketBase Admin UI 与静态资源嵌入流程分析

## 一、整体架构：三条资源返回路径

PocketBase Admin UI 挂载在 `/_/` 前缀下，但根据资源类型不同，实际上存在 **三条完全独立的代码执行路径**：

```
                              ┌───────────────────────────────────────────────────────────┐
                              │                    HTTP 路由 /_/*                          │
                              └───────────────────────────────────────────────────────────┘
                                                          │
                                                          │ 路由匹配（最具体优先）
                                                          ▼
              ┌───────────────────────────────┬───────────────────────────────┬──────────────────────────────────┐
              │  路径 A：嵌入打包静态资源        │  路径 B：扩展自有静态资源        │  路径 C：动态合并扩展入口脚本        │
              │                               │                               │                                   │
              │  典型 URL：                    │  典型 URL：                    │  典型 URL：                        │
              │    /_/assets/index-xxx.js     │    /_/extensions/ext1/a.png   │    /_/extensions.js               │
              │    /_/libs/tinymce/...        │    /_/extensions/ext2/style.css│                                   │
              │    /_/fonts/...               │                               │                                   │
              │    /_/images/...              │                               │                                   │
              │    /_/index.html (→ /_/)      │                               │                                   │
              │    /_/ (hash 路由页面)         │                               │                                   │
              │                               │                               │                                   │
              │  处理器：                      │  处理器：                      │  处理器：                          │
              │    Static(ui.DistDirFS,false) │    Static(ext.FS,false)        │    匿名函数（非 Static！）          │
              │                               │                               │                                   │
              │  文件来源：                    │  文件来源：                    │  文件来源：                        │
              │    Go embed 编译时嵌入          │    运行时注入的 ext.FS          │    运行时遍历所有扩展，              │
              │    ui/dist/ 目录                │    （每个扩展独立）              │    逐个 ext.FS.Open("main.js")      │
              │                               │                               │    动态拼接 IIFE 包裹              │
              │                               │                               │                                   │
              │  注册时机：                    │  注册时机：                    │  注册时机：                        │
              │    Serve() 中                  │    OnServe Hook                │    OnServe Hook                    │
              │    （OnServe 之前）              │    Priority 9999               │    Priority 9999                   │
              │                               │                               │                                   │
              │  内容发送：                    │  内容发送：                    │  内容发送：                        │
              │    http.ServeContent()         │    http.ServeContent()         │    re.Stream("text/javascript",buf)│
              │    (Range/协商缓存/Content-Type│    (Range/协商缓存/Content-Type│    (固定 Content-Type，不支持 Range)│
              └───────────────────────────────┴───────────────────────────────┴──────────────────────────────────┘
```

前端使用 Hash 路由（`/_/#/collections`），因此服务端 `indexFallback=false`，所有非 hash 直连 `/_/collections` 返回 404。

---

## 二、阶段 1：前端资源打包（Vite）

### 2.1 前端项目结构

前端代码位于 [ui/](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui) 目录：

| 目录/文件 | 说明 |
|-----------|------|
| [ui/src/](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/src) | 前端源代码（JS、CSS），使用 Shablon 模板引擎 |
| [ui/public/](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/public) | 静态资源（字体、图片、第三方库），构建时原样复制到 dist |
| [ui/index.html](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/index.html) | Vite 入口 HTML 模板 |
| [ui/vite.config.js](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/vite.config.js) | Vite 构建配置 |
| [ui/.env](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/.env) | 生产环境变量（含 `PB_BACKEND_URL = "../"`） |
| [ui/package.json](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/package.json) | npm 脚本与依赖 |

### 2.2 构建配置详解（关键）

在 [vite.config.js](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/vite.config.js#L1-L15) 中：

```js
import { defineConfig } from "vite";

export default defineConfig({
    envPrefix: "PB",
    base: "./",                // ★ 关键 1：资源引用使用相对路径 ./
    build: {
        chunkSizeWarningLimit: 1000,
        reportCompressedSize: false,
    },
    resolve: {
        alias: { "@": __dirname + "/src" },
    },
});
```

在 [ui/.env](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/.env#L1-L2) 中：

```
PB_BACKEND_URL = "../"      # ★ 关键 2：API 后端地址也用相对路径
```

#### 相对路径的必要性

Admin UI 挂载在 `/_/` 子路径下：

| 当前页面 URL | `<script src="./assets/a.js">` 解析 | `PB_BACKEND_URL="../"` + API `/api/collections` 解析 |
|-------------|------------------------------------|-----------------------------------------------------|
| `http://host:8090/_/` | `http://host:8090/_/assets/a.js` ✓ | `http://host:8090/api/collections` ✓ |

### 2.3 构建产物目录（走路径 A：嵌入静态）

执行 `npm run build` 后输出到 `ui/dist/`，其中所有文件都会被 Go embed 嵌入，最终走 **路径 A（Static 嵌入静态服务）**：

```
ui/dist/                          ← 被 //go:embed all:dist 整体嵌入
├── index.html                    ← 路径 A
├── assets/                       ← 路径 A（Vite 打包产物，带 content hash）
│   ├── index-V68uRsWE.js
│   ├── index-BkwjA9HK.css
│   └── ...（按路由拆分的 chunk）
├── fonts/                        ← 路径 A
├── images/                       ← 路径 A
└── libs/                         ← 路径 A（TinyMCE、uPlot、Prism、Shablon）
```

**注意：`/_/extensions.js` 不在 dist 目录中，它是运行时动态生成的，走路径 C。**

---

## 三、阶段 2：Go Embed 编译时嵌入（仅路径 A）

### 3.1 核心嵌入逻辑

[ui/embed.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/embed.go)：

```go
//go:build !no_ui
package ui

import (
    "embed"
    "io/fs"
)

//go:embed all:dist
var distDir embed.FS

// DistDirFS contains the embedded dist directory files (without the "dist" prefix)
var DistDirFS, _ = fs.Sub(distDir, "dist")
```

**关键技术点**：

1. **`//go:embed all:dist`**：Go 编译器在编译时读取 `ui/dist/` 目录内容，序列化进二进制。`all:` 前缀确保以 `_` 或 `.` 开头的文件也被嵌入。
2. **`fs.Sub(distDir, "dist")`**：`distDir.Open("dist/index.html")` 经 `fs.Sub` 包装后，调用方只需 `DistDirFS.Open("index.html")`。
3. **构建标签 `!no_ui`**：`go build` 默认嵌入；`go build -tags no_ui` 时使用 [ui/embed_no_ui.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/embed_no_ui.go)（`DistDirFS = nil`）。

---

## 四、阶段 3：前端 Hash 路由机制

### 4.1 为什么使用 Hash 路由

- **Hash 部分 `#/...` 不发送到服务器**：访问 `/_/#/collections`，浏览器实际只请求 `GET /_/`
- 前端 JS 读取 `window.location.hash` 渲染页面
- **服务端无需 SPA fallback**：`indexFallback = false`

### 4.2 前端路由定义

[ui/src/router.js](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/src/router.js#L117-L174)，所有路由均以 `#/` 开头：

```js
app.routes.guestOnly("#/login", pageSuperuserLogin);
app.routes.superuserOnly("#/collections", pageCollections);
app.routes.superuserOnly("#/settings", pageApplicationSettings);
// ...
// fallbackPath = "#/collections"（hash 为空时默认跳转）
```

### 4.3 前端加载流程（关键：动态加载 extensions.js）

[ui/src/main.js](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/src/main.js#L127-L147)：

```
1. 页面基础脚本执行完毕
2. await import(app.pb.buildURL("/_/extensions.js"))
   ↑ 请求 GET /_/extensions.js，走路径 C（动态合并处理器）
3. app.store._ready = true
4. initRouter() → 读 window.location.hash → 渲染对应页面
```

**也就是说，路径 C（扩展动态脚本）是前端页面加载流程的一部分，在路由初始化之前执行。**

---

## 五、阶段 4：HTTP 路由注册时序

三条路径的注册时机不同，这决定了它们的路由匹配优先级：

```
[called: Serve()]
   │
   ├── 1. NewRouter(app)
   │       │
   │       └── bindUIExtensions(app)
   │             │
   │             └── app.OnServe().Bind({Priority: 9999, Func: ...})
   │                    ↑ 仅仅绑定 Hook，不立即注册路由
   │
   ├── 2. pbRouter.GET("/_/{path...}", Static(ui.DistDirFS, false))
   │       ↑ 注册路径 A（嵌入打包静态资源）
   │         在 OnServe 触发之前
   │
   └── 3. app.OnServe().Trigger(serveEvent, func(e) {
             │
             ├── Hook 链执行（Priority 9999 最后执行）
             │       │
             │       └── bindUIExtensions 的 Func 被调用
             │             │
             │             ├── uiGroup = se.Router.Group("/_").Bind(缓存+CSP+Gzip)
             │             │
             │             ├── for ext in UIExtensions:
             │             │     uiGroup.GET("/extensions/"+ext.Name+"/{path...}", Static(ext.FS, false))
             │             │     ↑ 注册路径 B（各扩展自有静态资源）
             │             │
             │             └── uiGroup.GET("/extensions.js", 匿名处理器)
             │                   ↑ 注册路径 C（动态合并扩展入口脚本）
             │
             └── e.Router.BuildMux()
                   ↑ 所有路由（A+B+C）此时全部编译为最终 http.ServeMux
         })
```

**Go 1.22+ ServeMux 路由优先级（最具体匹配优先）**：

| 模式 | 匹配范围 | 优先级 |
|------|---------|--------|
| `GET /_/extensions.js`（路径 C） | 精确匹配一个文件 | **最高** |
| `GET /_/extensions/ext1/{path...}`（路径 B） | 特定扩展名的子路径 | **中** |
| `GET /_/{path...}`（路径 A） | `/_/` 下所有其他路径 | **最低** |

因此：
- 请求 `/_/extensions.js` 一定命中路径 C，不会被路径 A 捕获
- 请求 `/_/extensions/ext1/test.txt` 一定命中路径 B
- 请求 `/_/assets/a.js`、`/_/libs/...`、`/_/index.html` 等命中路径 A

---

## 六、阶段 5：统一访问场景表

以下是所有 `/_/` 前缀下 URL 的完整访问场景，**每条都明确标注命中哪条路径及完整代码调用链**：

| # | URL | 命中路径 | PathValue("path") | 完整代码执行链路 | HTTP 状态 |
|---|-----|---------|-------------------|-----------------|----------|
| A1 | `/_/#/collections` | **A** | `""`（hash 不发服务器） | `Static(ui.DistDirFS,false)`：`filename=""` → `fs.Stat`→根目录 → `FileFS` 拼 `index.html` → `http.ServeContent` | 200 |
| A2 | `/_/` | **A** | `""` | 同 A1 | 200 |
| A3 | `/_` | **A** | `""` | `Static`：`fi.IsDir=true`、URL 不以 `/` 结尾 → **301 重定向到 `/_/`** | 301 |
| A4 | `/_/index.html` | **A** | `"index.html"` | `Static`：`fs.Stat`→文件存在 → URL 以 `index.html` 结尾 → **301 重定向到 `/_/`** | 301 |
| A5 | `/_/assets/index-V68uRsWE.js` | **A** | `"assets/index-V68uRsWE.js"` | `Static`：`fs.Stat`→文件存在 → `FileFS` → `http.ServeContent`。中间件设 `Cache-Control: max-age=1209600`（path≠空） | 200 |
| A6 | `/_/libs/tinymce/tinymce.min.js` | **A** | `"libs/tinymce/tinymce.min.js"` | 同 A5（lib 也是 dist 嵌入文件） | 200 |
| A7 | `/_/fonts/roboto.woff2` | **A** | `"fonts/roboto.woff2"` | 同 A5 | 200 |
| A8 | `/_/images/logo.png` | **A** | `"images/logo.png"` | 同 A5 | 200 |
| A9 | `/_/collections`（非 hash ❌） | **A** | `"collections"` | `Static`：`fs.Stat(ui.DistDirFS,"collections")`→**不存在** → `indexFallback=false` → 返回 `ErrFileNotFound` | **404** |
| A10 | `/_/settings`（非 hash ❌） | **A** | `"settings"` | 同 A9 | **404** |
| A11 | `/_/assets/notexist.js` | **A** | `"assets/notexist.js"` | `fs.Stat`→不存在 → `indexFallback=false` → 404 | 404 |
| **C1** | **`/_/extensions.js`** | **C** | N/A（精确匹配） | 匿名处理器：`buf := new(bytes.Buffer)` → `for ext in UIExtensions: copyExtensionMainjs(buf, ext)`（逐个 `ext.FS.Open("main.js")` → 用 `await (async function(){...})();` 包裹 → `io.Copy` 到 buf） → `re.Stream(200, "text/javascript", buf)` | 200 |
| C2 | `/_/extensions.js`（无扩展） | **C** | N/A | 同 C1，但循环体无内容 → 返回空 Buffer（Content-Length: 0） | 200 |
| C3 | `/_/extensions.js`（`no_ui` 构建） | 无路由 | N/A | `ui.DistDirFS=nil` → `bindUIExtensions` 直接 return → 路径 C 未注册 → 回退到路径 A → `fs.Stat("extensions.js")` 不存在 → 404 | 404 |
| B1 | `/_/extensions/ext1/test.txt`（ext1 存在） | **B** | `"test.txt"`（在 ext1 组内） | `Static(ext1.FS, false)`：`fs.Stat(ext1.FS,"test.txt")`→存在 → `FileFS` → `http.ServeContent` | 200 |
| B2 | `/_/extensions/ext1/missing.txt`（ext1 存在） | **B** | `"missing.txt"` | `Static(ext1.FS, false)`：`fs.Stat`→不存在 → `indexFallback=false` → 404 | 404 |
| B3 | `/_/extensions/ext1/test.txt`（无任何扩展） | 无路由→回退 **A** | `"extensions/ext1/test.txt"` | 路径 B 未注册 → 命中路径 A → `fs.Stat(ui.DistDirFS, "extensions/ext1/test.txt")` → dist 中不存在 → 404 | 404 |
| B4 | `/_/extensions/ext2%20with%20spaces/a.css` | **B** | `"a.css"` | URL 解码后扩展名 "ext2 with spaces" 匹配 → `Static(ext.FS,false)` 正常处理 | 200 |

---

## 七、阶段 6：路径 A（嵌入静态服务）代码调用链

### 7.1 路由注册

[apis/serve.go:83-99](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/apis/serve.go#L83-L99)：

```go
if ui.DistDirFS != nil {
    pbRouter.GET("/_/{path...}", Static(ui.DistDirFS, false)).
        //                                       ↑ indexFallback = false
        BindFunc(func(e *core.RequestEvent) error {
            // path != ""（非 index.html）才设置 14 天强缓存
            if !e.App.IsDev() &&
                e.Request.PathValue(StaticWildcardParam) != "" &&
                e.Response.Header().Get("Cache-Control") == "" {
                e.Response.Header().Set("Cache-Control", "max-age=1209600, stale-while-revalidate=86400")
            }
            if e.Response.Header().Get("Content-Security-Policy") == "" {
                e.Response.Header().Set("Content-Security-Policy", defaultCSP)
            }
            return e.Next()
        }).
        Bind(Gzip())
}
```

### 7.2 Static 处理器完整执行流程

[apis/base.go:93-171](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/apis/base.go#L93-L171)：

```
Static(ui.DistDirFS, false) 返回闭包 → 被 http.ServeMux 调用
    │
    ├── 1. e.Set(requestEventKeySkipSuccessActivityLog, true)
    │       跳过成功访问日志
    │
    ├── 2. filename = filepath.Clean(PathValue("path"))
    │
    ├── 3. 目录穿越防护（防御性检查）
    │
    ├── 4. fi, err := fs.Stat(ui.DistDirFS, filename)
    │       │
    │       ├── err != nil（文件不存在）：
    │       │     indexFallback=false → 直接返回 ErrFileNotFound（404）
    │       │
    │       └── err == nil：
    │             │
    │             ├── fi.IsDir()：
    │             │     ├── URL 不以 "/" 结尾 → 301 重定向到 URL+"/"
    │             │     └── 已以 "/" 结尾 → 继续（不重定向）
    │             │
    │             └── !fi.IsDir()（是文件）：
    │                   ├── URL 以 "/" 结尾 → 301 去掉尾斜杠
    │                   └── URL 以 "index.html" 结尾 → 301 重定向到父目录
    │
    └── 5. fileErr := e.FileFS(ui.DistDirFS, filename)
            │
            └── 见 FileFS 调用链（下一节）
```

### 7.3 FileFS 文件发送（路径 A/B 共用）

[tools/router/event.go:234-272](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/tools/router/event.go#L234-L272)：

```
e.FileFS(fsys, filename)
    │
    ├── f := fsys.Open(filename)
    │
    ├── fi.IsDir() == true：
    │     filename = filepath.Join(filename, "index.html")
    │     重新 fsys.Open(filename)
    │
    ├── ff := f.(io.ReadSeeker)   // embed.FS 返回的文件满足此接口
    │
    └── http.ServeContent(Response, Request, fi.Name(), fi.ModTime(), ff)
          ↑ 标准库自动处理：
              - Content-Type 推断（扩展名）
              - Range 请求（断点续传）
              - Last-Modified / If-Modified-Since（协商缓存）
```

---

## 八、阶段 7：路径 C（动态合并扩展脚本）代码调用链

### 8.1 路由注册

[apis/extensions.go:52-66](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/apis/extensions.go#L52-L66)：

```go
uiGroup.GET("/extensions.js", func(re *core.RequestEvent) error {
    buf := new(bytes.Buffer)

    // ★ 每次请求都重新遍历所有扩展，重新读取并合并
    // note: don't cache in memory to allow previewing changes without restart
    for _, ext := range se.UIExtensions {
        err := copyExtensionMainjs(buf, ext)
        if err != nil {
            return re.InternalServerError("...", err)
        }
    }

    return re.Stream(200, "text/javascript", buf)
}).Bind(SkipSuccessActivityLog())
```

### 8.2 copyExtensionMainjs 合并逻辑

[apis/extensions.go:72-95](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/apis/extensions.go#L72-L95)：

```
copyExtensionMainjs(buf, ext)
    │
    ├── f, err := ext.FS.Open("main.js")
    │     │
    │     ├── os.ErrNotExist：return nil（跳过，不报错）
    │     └── 其他错误：return error
    │
    ├── buf.WriteString("await (async function(){")
    │     ↑ IIFE 包裹，隔离各扩展作用域
    │
    ├── io.Copy(buf, f)
    │     ↑ 将扩展 main.js 原样拷贝到 Buffer
    │
    └── buf.WriteString("})();")
          ↑ await 用于支持扩展内顶层 await
```

### 8.3 合并后响应示例

来自测试 [apis/extensions_test.go:75](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/apis/extensions_test.go#L75)：

```js
await (async function(){ext1_main})();await (async function(){ext3_main})();
```

### 8.4 路径 C 与路径 A 的 12 项差异

| 对比维度 | 路径 A：`/_/assets/index-xxx.js` | 路径 C：`/_/extensions.js` |
|---------|-------------------------------|--------------------------|
| 路由模式 | `GET /_/{path...}`（通配符） | `GET /_/extensions.js`（精确匹配） |
| 处理器 | `Static(ui.DistDirFS, false)` 返回的闭包 | 内联匿名函数 |
| 文件存在性检查 | `fs.Stat(fsys, "assets/index-xxx.js")` —— 精确查找 | 无文件存在性检查；扩展无 main.js 则跳过 |
| 文件读取 | `ui.DistDirFS.Open(filename)` 一次打开 | 循环 N 次 `ext.FS.Open("main.js")`（每个扩展各一次） |
| 目录处理 | 目录自动拼接 `index.html` | 不涉及目录概念 |
| 重定向逻辑 | 检查是否以 `/` 结尾、是否含 `index.html`，必要时 301 | 无重定向 |
| SPA Fallback | `indexFallback=false` → 404 | 不适用 |
| 内容发送方式 | `e.FileFS()` → `http.ServeContent()`（支持 Range、协商缓存、Content-Type 推断） | `re.Stream(200, "text/javascript", buf)`（固定 Content-Type，不支持 Range） |
| 内存缓存 | 文件在编译时嵌入二进制，常驻内存 | **每次请求重新读取并合并**（注释明确 "don't cache in memory to allow previewing changes without restart"） |
| 成功日志跳过 | Static 内部 `e.Set(requestEventKeySkipSuccessActivityLog, true)` | 显式 `.Bind(SkipSuccessActivityLog())` |
| 缓存头中间件 | 路由级别：仅 `path != ""` 时设置 Cache-Control | 组级别 uiGroup：无条件设置 Cache-Control（非 dev） |
| Gzip 中间件 | 绑定在路径 A 路由上 | 绑定在 uiGroup 上（路径 C 继承） |

---

## 九、阶段 8：路径 B（扩展自有静态资源）代码调用链

### 9.1 路由注册

[apis/extensions.go:42-49](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/apis/extensions.go#L42-L49)：

```go
for _, ext := range se.UIExtensions {
    if ext.Name == "" || ext.FS == nil {
        continue
    }
    // 路径模式：/_/extensions/{name}/{path...}
    // 文件系统：ext.FS（运行时注入，非 embed）
    uiGroup.GET("/extensions/"+ext.Name+"/{path...}", Static(ext.FS, false))
}
```

### 9.2 与路径 A 的异同

| 项目 | 路径 A（嵌入打包静态） | 路径 B（扩展自有静态） |
|------|---------------------|---------------------|
| Static 处理器 | ✓ 相同 | ✓ 相同 |
| indexFallback | false | false |
| FileFS → http.ServeContent | ✓ 相同 | ✓ 相同 |
| fs.FS 来源 | `ui.DistDirFS`（编译时 embed） | `ext.FS`（运行时注入任意 fs.FS） |
| 路由模式 | `/_/{path...}` | `/_/extensions/{name}/{path...}`（多一层命名空间） |
| 缓存头中间件 | 路由级（path≠空才设） | uiGroup 级（无条件） |

---

## 十、完整调用时序图

### 10.1 时序图 A：`/_/#/collections` 页面完整加载（路径 A + 路径 C）

```
用户输入: http://127.0.0.1:8090/_/#/collections
        │
        │  hash 部分 #/collections 不发送到服务器
        ▼
[HTTP 1] GET /_/
        │
        ├─ Go ServeMux 匹配：无更具体路由 → 命中路径 A
        │   模式: GET /_/{path...}
        │   PathValue("path") = ""
        │
        ├─ [路径 A] Static(ui.DistDirFS, false) 执行：
        │   ├── filename = filepath.Clean("") = ""
        │   ├── fs.Stat(ui.DistDirFS, "") → fi.IsDir() = true
        │   ├── URL "/_/" 已以 "/" 结尾 → 不重定向
        │   └── e.FileFS(ui.DistDirFS, "")
        │         ├── fsys.Open("") → 打开根目录
        │         ├── fi.IsDir() → 拼接 index.html
        │         ├── fsys.Open("index.html") → 成功
        │         └── http.ServeContent(...) → 发送 HTML
        │
        │   响应头：无 Cache-Control（path=""），有 CSP + Gzip
        │   响应体：index.html（200 OK）
        ▼
浏览器解析 index.html，发现以下资源引用：
        │
        ├── <script src="./libs/shablon/shablon.iife.js">
        │       → 相对路径解析为 GET /_/libs/shablon/shablon.iife.js
        │
        ├── <script type="module" src="./assets/index-V68uRsWE.js">
        │       → 相对路径解析为 GET /_/assets/index-V68uRsWE.js
        │
        └── <link rel="stylesheet" href="./assets/index-BkwjA9HK.css">
                → 相对路径解析为 GET /_/assets/index-BkwjA9HK.css
        │
        ▼
[HTTP 2~N] GET /_/assets/index-V68uRsWE.js 等（全部走路径 A）
        │
        ├─ 全部命中路径 A：GET /_/{path...}
        │   例如 PathValue("path") = "assets/index-V68uRsWE.js"
        │
        ├─ [路径 A] Static 执行：
        │   ├── filename = "assets/index-V68uRsWE.js"
        │   ├── fs.Stat(ui.DistDirFS, filename) → 存在（文件）
        │   ├── URL 不以 "/" 结尾，不以 "index.html" 结尾 → 不重定向
        │   └── e.FileFS(ui.DistDirFS, filename)
        │         └── http.ServeContent(...) → 发送文件
        │
        │   响应头：Cache-Control: max-age=1209600（path≠空），CSP，Gzip
        ▼
浏览器执行 JS，main.js 启动
        │
        └── await import(app.pb.buildURL("/_/extensions.js"))
                → GET /_/extensions.js
        │
        ▼
[HTTP N+1] GET /_/extensions.js（走路径 C）
        │
        ├─ Go ServeMux 匹配：精确模式 GET /_/extensions.js 优先
        │   （不命中路径 A 的通配符）
        │
        ├─ [路径 C] 匿名处理器执行：
        │   ├── buf := new(bytes.Buffer)
        │   ├── SkipSuccessActivityLog 已绑定
        │   │
        │   ├── for _, ext := range se.UIExtensions:
        │   │     └── copyExtensionMainjs(buf, ext):
        │   │           ├── ext.FS.Open("main.js")
        │   │           │     ├── 不存在 → skip（return nil）
        │   │           │     └── 存在 → 继续
        │   │           ├── buf.WriteString("await (async function(){")
        │   │           ├── io.Copy(buf, f)   ← 拷贝扩展 main.js 内容
        │   │           └── buf.WriteString("})();")
        │   │
        │   └── re.Stream(200, "text/javascript", buf)
        │
        │   响应头：uiGroup 中间件设 Cache-Control（非 dev）、CSP、Gzip
        │   响应体：await (async function(){ext1_main})();await (async function(){ext3_main})();
        ▼
扩展脚本执行完毕
        │
        ├── app.store._ready = true
        ├── initRouter() 被调用
        ├── window.location.hash = "#/collections"
        └── Shablon router 匹配 "#/collections" → 渲染集合列表页 ✓
```

### 10.2 时序图 B：`/_/extensions/ext1/test.txt`（路径 B）

```
GET /_/extensions/ext1/test.txt
        │
        ├─ Go ServeMux 匹配（最具体优先）：
        │   模式 1: GET /_/extensions/ext1/{path...}（路径 B）→ 优先 ✓
        │   模式 2: GET /_/{path...}（路径 A）→ 被跳过
        │
        │   PathValue("path") = "test.txt"
        │
        ├─ [路径 B] Static(ext1.FS, false) 执行：
        │   ├── filename = filepath.Clean("test.txt") = "test.txt"
        │   ├── fs.Stat(ext1.FS, "test.txt") → 存在（文件）
        │   ├── 不重定向
        │   └── e.FileFS(ext1.FS, "test.txt")
        │         └── http.ServeContent(...) → 发送 ext1 的 test.txt
        │
        │   响应头：uiGroup 中间件设 Cache-Control、CSP、Gzip
        │   响应体：ext1_txt（200 OK）
        ▼
200 OK ✓
```

### 10.3 时序图 C：`/_/collections` 非 hash 直连（路径 A → 404）

```
GET /_/collections（注意：没有 #，是错误 URL）
        │
        ├─ Go ServeMux 匹配：无更具体路由 → 命中路径 A
        │   模式: GET /_/{path...}
        │   PathValue("path") = "collections"
        │
        ├─ [路径 A] Static(ui.DistDirFS, false) 执行：
        │   ├── filename = filepath.Clean("collections") = "collections"
        │   ├── fs.Stat(ui.DistDirFS, "collections") → err != nil（不存在该文件/目录）
        │   ├── indexFallback == false → ★ 不回退到 index.html
        │   └── return router.ErrFileNotFound
        │
        ▼
404 Not Found ❌
正确的 URL 应该是 /_/#/collections（带 #），由前端 hash 路由处理
```

---

## 十一、首次启动 Installer 机制

当系统中尚无超级用户时，PocketBase 会自动引导创建首个管理员（[apis/installer.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/apis/installer.go)）：

1. `needInstallerSuperuser()` 检查超级用户表是否为空
2. 创建临时 installer 用户（邮箱 `__pocketbase_installer@local.dev`）
3. 生成 30 分钟 token，打开浏览器访问 **`/_/#/pbinstall/{token}`（hash 路由，避免 404）**
4. 前端 [ui/src/auth/pageInstaller.js](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/src/auth/pageInstaller.js) 处理 `#/pbinstall/{token}` hash 路由
5. 用户创建真正管理员后，临时账户被清理

---

## 十二、关键文件索引

| 文件 | 职责 |
|------|------|
| [ui/embed.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/embed.go) | Go embed 嵌入 dist 目录（路径 A 的文件来源） |
| [ui/embed_no_ui.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/embed_no_ui.go) | no_ui 标签下的空实现（路径 B/C 也不注册） |
| [ui/vite.config.js](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/vite.config.js) | Vite 构建配置（`base: "./"` 相对路径） |
| [ui/.env](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/.env) | 生产环境变量（`PB_BACKEND_URL = "../"`） |
| [ui/package.json](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/package.json) | npm build/dev 脚本 |
| [ui/src/router.js](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/src/router.js) | 前端 hash 路由定义（全部 #/ 开头） |
| [ui/src/main.js](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/src/main.js) | 前端入口，动态 import `/_/extensions.js`（触发路径 C），初始化路由 |
| [ui/src/pb.js](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/src/pb.js) | JS SDK 初始化（使用 `PB_BACKEND_URL` 相对路径） |
| [apis/serve.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/apis/serve.go#L83-L99) | 注册路径 A：`/_/{path...}` 路由与缓存/CSP/Gzip 中间件 |
| [apis/base.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/apis/base.go#L93-L171) | `Static()` 静态文件处理器（路径 A 和 B 共用，含 `indexFallback=false`） |
| [apis/extensions.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/apis/extensions.go) | 注册路径 B（扩展静态）和路径 C（动态合并 `extensions.js`） |
| [tools/router/event.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/tools/router/event.go#L234-L272) | `FileFS()` 实际发送文件（路径 A/B 共用，目录自动转 index.html，调用 http.ServeContent） |
| [apis/installer.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/apis/installer.go) | 首次启动 Installer 引导流程（使用 hash 路由 URL） |
| [apis/extensions_test.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/apis/extensions_test.go) | 路径 B/C 的测试用例（含合并结果断言） |
| [cmd/serve.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/cmd/serve.go) | serve CLI 命令入口 |
| [pocketbase.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/pocketbase.go) | PocketBase 入口，注册 serve 命令 |
