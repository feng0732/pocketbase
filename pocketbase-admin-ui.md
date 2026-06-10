# PocketBase Admin UI 与静态资源嵌入流程分析

## 一、整体架构概览

PocketBase 的 Admin UI（超级用户管理后台）采用 **"前端打包 → Go embed 嵌入 → HTTP 静态服务"** 的三段式架构，最终实现**单二进制分发**——所有前端资源被编译进 Go 可执行文件中，无需额外部署。

完整流程链路：

```
[Vite 前端构建]  →  ui/dist/  →  [Go embed 编译时嵌入]  →  二进制文件
                                                         ↓
                                              [HTTP 路由 /_/*]
                                                         ↓
                                              [Static 处理器]
                                                         ↓
                                              [浏览器渲染 + 前端 Hash 路由]
```

**核心设计选择**：
- 前端使用 **Hash 路由**（`/_/#/collections`），而非 History API，因此服务端无需 SPA fallback
- Vite `base: "./"` 配合 `PB_BACKEND_URL = "../"`，确保资源和 API 路径使用相对引用
- 服务端 `indexFallback = false`，所有非 hash 路由的直连访问直接返回 404

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
    envPrefix: "PB",           // 只有 PB_ 开头的 env 变量会被注入前端
    base: "./",                // ★ 关键 1：资源引用使用相对路径 ./
    build: {
        chunkSizeWarningLimit: 1000,
        reportCompressedSize: false,
    },
    resolve: {
        alias: {
            "@": __dirname + "/src",
        },
    },
});
```

在 [ui/.env](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/.env#L1-L2) 中：

```
PB_BACKEND_URL = "../"      # ★ 关键 2：API 后端地址也用相对路径
```

#### 为什么必须用相对路径？

Admin UI 被挂载在 `/_/` 子路径下。如果使用绝对路径 `/assets/xxx.js`，浏览器会从域名根路径请求，导致 404（应该是 `/_/assets/xxx.js`）。使用 `./` 后，资源引用会基于当前页面 URL 的目录解析：

| 当前页面 URL | `<script src="./assets/a.js">` 解析结果 |
|-------------|-----------------------------------------|
| `http://host:8090/_/` | `http://host:8090/_/assets/a.js` ✓ |
| `http://host:8090/_/libs/tinymce/...` | `http://host:8090/_/libs/tinymce/assets/a.js`（tinymce 内部引用）|

同理，`PB_BACKEND_URL = "../"` 使得 SDK 发 API 请求时：
- 从 `/_/` 页面出发，`../` 解析到根路径 `/`
- 请求 `/api/collections` 最终变为 `http://host:8090/api/collections` ✓

### 2.3 入口 HTML 构建前后对比

| 源文件 [ui/index.html](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/index.html) | 构建后 [ui/dist/index.html](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/dist/index.html) |
|---|---|
| `<script src="/libs/shablon/shablon.iife.js">` | `<script src="./libs/shablon/shablon.iife.js">` |
| `<script type="module" src="/src/main.js">` | `<script type="module" src="./assets/index-V68uRsWE.js">` |
| - | `<link rel="stylesheet" href="./assets/index-BkwjA9HK.css">` |

### 2.4 构建命令

在 [ui/package.json](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/package.json#L5-L8) 中定义：

```json
"scripts": {
    "dev": "vite",                              // 开发模式，热重载
    "build": "dprint fmt && vite build"         // 生产构建，输出到 dist/
}
```

执行 `npm run build` 后，输出目录结构：

```
ui/dist/
├── index.html              # SPA 入口
├── assets/                 # 打包后的 JS/CSS（带 content hash）
│   ├── index-V68uRsWE.js
│   ├── index-BkwjA9HK.css
│   └── ...（按路由拆分的 chunk）
├── fonts/                  # 字体文件
├── images/                 # 图标、Logo、OAuth2 提供商图标
└── libs/                   # 第三方库（TinyMCE、uPlot、Prism、Shablon）
```

---

## 三、阶段 2：Go Embed 编译时嵌入

### 3.1 核心嵌入逻辑

嵌入由 [ui/embed.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/embed.go) 实现：

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

**关键技术点解析**：

1. **`//go:embed all:dist` 指令**：
   - `all:` 前缀确保 dist 目录下的所有文件（包括以 `_` 或 `.` 开头的文件）都被嵌入
   - Go 编译器在编译时读取 `ui/dist/` 目录内容，序列化进二进制
   - 必须导入 `embed` 包才能使用该指令

2. **`fs.Sub(distDir, "dist")` 的作用**：
   - 假设文件实际路径是 `dist/index.html`，在 `embed.FS` 中访问需要使用 `distDir.Open("dist/index.html")`
   - 通过 `fs.Sub` 创建一个子文件系统视图，使得 `DistDirFS.Open("index.html")` 即可访问
   - 简化后续 HTTP 处理器的路径映射

3. **构建标签 `//go:build !no_ui`**：
   - 默认构建（`go build`）时使用此文件，UI 被嵌入
   - 使用 `go build -tags no_ui` 时跳过此文件，改用 [ui/embed_no_ui.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/embed_no_ui.go)

### 3.2 无 UI 构建模式

[ui/embed_no_ui.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/embed_no_ui.go)：

```go
//go:build no_ui

package ui

import "io/fs"

// DistDirFS is deliberately not set to prevent bundling the UI with the binary.
var DistDirFS fs.FS     // 值为 nil
```

通过 `ui.DistDirFS == nil` 判断即可在运行时检测 UI 是否可用。

---

## 四、阶段 3：Hash 路由机制（前端）

### 4.1 为什么使用 Hash 路由

PocketBase Admin UI 使用 Hash 路由（如 `/_/#/collections`）而不是 History API 路由（如 `/_/collections`），原因在于：

- **Hash 部分 `#/...` 不会发送到服务器**——浏览器只发送 `/_/`，服务器始终返回同一个 `index.html`
- 前端 JS 通过监听 `hashchange` 事件，读取 `window.location.hash` 来决定渲染哪个页面
- **服务端无需实现 SPA fallback**——`indexFallback = false` 即可，简化了静态服务逻辑

### 4.2 前端路由实现

路由定义在 [ui/src/router.js](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/src/router.js#L117-L174)：

```js
const routeDefs = {};

// 注册路由（全部以 #/ 开头）
app.routes.guestOnly("#/login", pageSuperuserLogin);
app.routes.guestOnly("#/pbinstall/{token}", pageInstaller);
app.routes.superuserOnly("#/collections", pageCollections);
app.routes.superuserOnly("#/settings", pageApplicationSettings);
app.routes.superuserOnly("#/settings/sql", pageSQLConsole);
// ... 等等

export function initRouter() {
    destroyRouter = router(routeDefs, { fallbackPath: app.routes.fallbackPath });
}
// fallbackPath = "#/collections"（默认跳到集合列表页）
```

页面初始化流程在 [ui/src/main.js](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/src/main.js#L127-L147)：

1. 动态加载 `/_/extensions.js`（UI 扩展合并脚本）
2. 加载完成后 `app.store._ready = true`
3. watch 触发后调用 `initRouter()`
4. Shablon 的 `router()` 读取 `window.location.hash`，匹配路由定义并渲染页面
5. 如果 hash 为空或不匹配，fallback 到 `#/collections`

### 4.3 API 路径构建

在 [ui/src/pb.js](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/src/pb.js#L9-L14) 中初始化 PocketBase JS SDK：

```js
window.app.pb = new PocketBase(
    import.meta.env.PB_BACKEND_URL,   // 生产环境为 "../"
    new LocalAuthStore("__pb_superusers__" + currentPath),
);
```

SDK 的 `buildURL()` 方法会把 `../` 作为 base，结合 `/_/` 页面路径：
- `buildURL("/api/collections")` → 解析为 `/api/collections`（向上跳出 `/_/` 目录）
- `buildURL("/_/extensions.js")`（见 [main.js](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/src/main.js#L137-L139)）→ 解析为 `/_/extensions.js`

---

## 五、阶段 4：HTTP 路由注册与访问场景详解

### 5.1 路由注册入口

在 [apis/serve.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/apis/serve.go#L83-L99) 的 `Serve()` 函数中：

```go
if ui.DistDirFS != nil {
    pbRouter.GET("/_/{path...}", Static(ui.DistDirFS, false)).
        //                                       ↑ 注意：indexFallback = false
        BindFunc(func(e *core.RequestEvent) error {
            // 缓存控制：非开发模式、非根路径时设置 14 天缓存
            if !e.App.IsDev() &&
                e.Request.PathValue(StaticWildcardParam) != "" &&
                //                                    ↑ path 非空才缓存
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

**路由设计要点**：

| 项目 | 值 | 说明 |
|------|----|------|
| 路径 | `/_/{path...}` | Admin UI 挂载在 `/_/` 前缀下，与 API 路径 `/api/*` 分离 |
| 处理器 | `Static(ui.DistDirFS, false)` | 通用静态文件服务 |
| **indexFallback** | **`false`** | **不启用 SPA 路由 fallback**——因为前端使用 hash 路由 |
| 中间件 | 缓存控制 + CSP + Gzip | 生产环境非根路径资源缓存 14 天，启用 Gzip 压缩 |

### 5.2 URL 访问场景全景分析

路由模式 `GET /_/{path...}` 中的 `{path...}` 是 Go 1.22+ 的通配符语法，匹配 `/_/` 之后的所有路径段。通过 `e.Request.PathValue("path")` 取出匹配值。

以下是各种访问场景的完整分析：

---

#### ✅ 场景 A：正确访问——Hash 路由 `/_/#/collections`

```
用户输入: http://127.0.0.1:8090/_/#/collections
```

| 步骤 | 发生了什么 | 代码位置 |
|------|-----------|---------|
| A1 | 浏览器发起请求：**`GET /_/`**（`#/collections` 是 hash，不会发送） | 浏览器行为 |
| A2 | Go `http.ServeMux` 匹配路由 `/_/{path...}`，`PathValue("path") = ""` | 标准库 |
| A3 | Static 处理器：`filename = filepath.Clean("") = ""` | [apis/base.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/apis/base.go#L124) |
| A4 | `fs.Stat(fsys, "")` → 根目录存在，`fi.IsDir() = true` | |
| A5 | URL `/_/` 已以 `/` 结尾，不触发重定向 | [apis/base.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/apis/base.go#L146-L148) |
| A6 | `e.FileFS(fsys, "")` → 目录自动拼接 `index.html`，发送文件 | [tools/router/event.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/tools/router/event.go#L250-L262) |
| A7 | 缓存中间件：`path == ""` → **不设置** Cache-Control（HTML 不缓存） | [apis/serve.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/apis/serve.go#L86-L91) |
| A8 | 浏览器收到 index.html，解析 `<script src="./assets/...">` | |
| A9 | 相对路径 `./assets/index-xxx.js` → 请求 `GET /_/assets/index-xxx.js` | 浏览器行为 |
| A10 | 前端 JS 读取 `window.location.hash = "#/collections"`，渲染集合列表页 | [ui/src/router.js](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/src/router.js) |

**结果：200 OK ✅，页面正常渲染**

---

#### ✅ 场景 B：正确访问——根路径 `/_/` 或 `/_`

```
用户输入: http://127.0.0.1:8090/_/   (末尾带斜杠)
用户输入: http://127.0.0.1:8090/_    (末尾不带斜杠)
```

| URL | 行为 |
|-----|------|
| `/_/` | 同场景 A，直接返回 index.html，前端 hash fallback 到 `#/collections` |
| `/_` | Go `http.ServeMux` 匹配 `/_/{path...}`，`path=""`，`fi.IsDir()=true`，URL 不以 `/` 结尾 → **301 重定向到 `/_/`**，然后同上 |

---

#### ✅ 场景 C：正确访问——静态资源 `/_/assets/index-V68uRsWE.js`

```
浏览器自动请求: GET /_/assets/index-V68uRsWE.js
```

| 步骤 | 发生了什么 |
|------|-----------|
| C1 | `PathValue("path") = "assets/index-V68uRsWE.js"` |
| C2 | `filename = filepath.Clean("assets/index-V68uRsWE.js")` = `"assets/index-V68uRsWE.js"` |
| C3 | `fs.Stat(fsys, "assets/index-V68uRsWE.js")` → 存在，`fi.IsDir() = false` |
| C4 | URL 不以 `/` 结尾，不以 `index.html` 结尾 → 不重定向 |
| C5 | `e.FileFS(fsys, "assets/index-V68uRsWE.js")` → 直接发送文件 |
| C6 | 缓存中间件：`path != ""` 且非 dev → 设置 `Cache-Control: max-age=1209600...`（14 天）|

**结果：200 OK ✅，带强缓存**

同理，`/_/fonts/...`、`/_/images/...`、`/_/libs/...`、`/_/extensions.js` 都属于这类。

---

#### ✅ 场景 D：规范化重定向——`/_/index.html`

```
用户输入: http://127.0.0.1:8090/_/index.html
```

| 步骤 | 发生了什么 |
|------|-----------|
| D1 | `path = "index.html"` |
| D2 | `fs.Stat(fsys, "index.html")` → 存在，是文件 |
| D3 | URL 以 `index.html` 结尾 → **301 重定向到 `/_/`** |
| D4 | 浏览器请求 `/_/`，回到场景 B |

**结果：301 → 200 OK ✅**

---

#### ❌ 场景 E：404 错误——非 Hash 路径直连 `/_/collections`

```
用户直接输入或刷新: http://127.0.0.1:8090/_/collections
```

这是最容易混淆的场景。由于前端使用 hash 路由，**这个路径在服务端根本不存在对应的文件或目录**。

| 步骤 | 发生了什么 | 代码位置 |
|------|-----------|---------|
| E1 | `path = "collections"` | |
| E2 | `filename = filepath.Clean("collections")` = `"collections"` | [apis/base.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/apis/base.go#L124) |
| E3 | `fs.Stat(fsys, "collections")` → **文件不存在**，返回 err | [apis/base.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/apis/base.go#L136) |
| E4 | `indexFallback == false` && `filename != "index.html"` → **不 fallback** | [apis/base.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/apis/base.go#L137-L142) |
| E5 | 直接返回 `router.ErrFileNotFound` → HTTP **404** | |

**结果：404 Not Found ❌**

**用户需要访问的正确 URL 是 `/_/#/collections`（带 #）**。

---

#### ❌ 场景 F：404 错误——`/_/collections/`（末尾带斜杠）

```
用户输入: http://127.0.0.1:8090/_/collections/
```

| 步骤 | 发生了什么 |
|------|-----------|
| F1 | `path = "collections/"` |
| F2 | `filename = filepath.Clean("collections/")` = `"collections"`（Clean 去掉末尾斜杠）|
| F3 | `fs.Stat(fsys, "collections")` → **不存在** |
| F4 | `indexFallback == false` → 返回 404 |

**结果：404 Not Found ❌**

---

#### ❌ 场景 G：404 错误——不存在的资源

```
GET /_/assets/nonexist.js
GET /_/nonexist
GET /_/nonexist/
```

都因 `fs.Stat()` 失败且 `indexFallback=false` 返回 404。

---

### 5.3 访问场景汇总表

| 用户访问 URL | path 值 | filename | fs.Stat 结果 | 行为 | HTTP 状态 |
|-------------|---------|----------|-------------|------|----------|
| `/_/` | `""` | `""` | 根目录 ✓ | 目录 → index.html | 200 |
| `/_` | `""` | `""` | 根目录 ✓ | 301 重定向到 `/_/` | 301 |
| `/_/#/collections` | `""` | `""` | 根目录 ✓ | 返回 index.html，前端渲染 | 200 |
| `/_/#/settings/sql` | `""` | `""` | 根目录 ✓ | 返回 index.html，前端渲染 | 200 |
| `/_/index.html` | `"index.html"` | `"index.html"` | 文件 ✓ | 301 重定向到 `/_/` | 301 |
| `/_/assets/index-V68uRsWE.js` | `"assets/index-V68uRsWE.js"` | 同上 | 文件 ✓ | 直接返回，带 14 天缓存 | 200 |
| `/_/libs/tinymce/tinymce.min.js` | `"libs/tinymce/tinymce.min.js"` | 同上 | 文件 ✓ | 直接返回，带缓存 | 200 |
| `/_/extensions.js` | `"extensions.js"` | `"extensions.js"` | 文件 ✓（扩展脚本） | 直接返回 | 200 |
| **`/_/collections`** | `"collections"` | `"collections"` | **不存在** | **indexFallback=false → 404** | **404 ❌** |
| **`/_/settings`** | `"settings"` | `"settings"` | **不存在** | **indexFallback=false → 404** | **404 ❌** |
| `/_/collections/` | `"collections/"` | `"collections"` | **不存在** | indexFallback=false → 404 | 404 ❌ |
| `/_/assets/notexist.js` | `"assets/notexist.js"` | 同上 | **不存在** | 404 | 404 ❌ |

---

## 六、阶段 5：Static 静态文件处理器详解

### 6.1 Static 函数完整实现

[apis/base.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/apis/base.go#L93-L171)：

```go
func Static(fsys fs.FS, indexFallback bool) func(*core.RequestEvent) error {
    if fsys == nil {
        panic("Static: the provided fs.FS argument is nil")
    }

    return func(e *core.RequestEvent) error {
        // 1. 跳过成功访问日志（避免刷屏）
        if e.Get(requestEventKeySkipSuccessActivityLog) == nil {
            e.Set(requestEventKeySkipSuccessActivityLog, true)
        }

        // 2. 提取并规范化路径
        filename := e.Request.PathValue(StaticWildcardParam)  // "path"
        filename = filepath.ToSlash(filepath.Clean(strings.TrimPrefix(filename, "/")))

        // 3. 目录穿越防护（防御性检查，标准 fs 通常已处理）
        if len(filename) > 2 && filename[0] == '.' && filename[1] == '.' && (filename[2] == '/' || filename[2] == '\\') {
            if indexFallback && filename != router.IndexPage {
                return e.FileFS(fsys, router.IndexPage)
            }
            return router.ErrFileNotFound
        }

        // 4. 检查文件/目录是否存在
        fi, err := fs.Stat(fsys, filename)
        if err != nil {
            if indexFallback && filename != router.IndexPage {
                return e.FileFS(fsys, router.IndexPage)
            }
            return router.ErrFileNotFound
        }

        // 5. URL 规范化重定向
        if fi.IsDir() {
            if !strings.HasSuffix(e.Request.URL.Path, "/") {
                // 目录：确保以 / 结尾 → /test -> /test/
                return e.Redirect(http.StatusMovedPermanently, safeRedirectPath(e.Request.URL.Path+"/"))
            }
        } else {
            urlPath := e.Request.URL.Path
            if strings.HasSuffix(urlPath, "/") {
                // 文件但以 / 结尾 → 去掉尾部斜杠
                urlPath = strings.TrimRight(urlPath, "/")
                return e.Redirect(http.StatusMovedPermanently, safeRedirectPath(urlPath))
            } else if stripped, ok := strings.CutSuffix(urlPath, router.IndexPage); ok {
                // 显式 index.html → 重定向到目录
                return e.Redirect(http.StatusMovedPermanently, safeRedirectPath(stripped))
            }
        }

        // 6. 实际发送文件
        fileErr := e.FileFS(fsys, filename)

        // 7. SPA fallback（仅当 indexFallback=true 时；Admin UI 中此分支永远不触发）
        if fileErr != nil && indexFallback && filename != router.IndexPage && errors.Is(fileErr, router.ErrFileNotFound) {
            return e.FileFS(fsys, router.IndexPage)
        }

        return fileErr
    }
}
```

### 6.2 重定向规则汇总

| 请求路径 | 文件/目录类型 | 行为 | 示例 |
|----------|--------------|------|------|
| `/test`（无尾斜杠） | 目录 | 301 → `/test/` | `/_/assets` → `/_/assets/`（如果 assets 是目录） |
| `/test/`（有尾斜杠） | 文件 | 301 → `/test` | `/_/index.js/` → `/_/index.js` |
| `/test/index.html` | 文件 | 301 → `/test/` | `/_/index.html` → `/_/` |

### 6.3 FileFS 文件发送实现

最终由 [tools/router/event.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/tools/router/event.go#L234-L272) 的 `FileFS()` 完成文件输出：

```go
func (e *Event) FileFS(fsys fs.FS, filename string) error {
    f, err := fsys.Open(filename)
    if err != nil {
        return ErrFileNotFound
    }
    defer f.Close()

    fi, err := f.Stat()
    if err != nil {
        return err
    }

    // 如果是目录，自动拼接 index.html
    if fi.IsDir() {
        filename = filepath.ToSlash(filepath.Join(filename, IndexPage))
        f, err = fsys.Open(filename)
        // ...（错误处理与重新 Stat）
    }

    // 文件必须实现 io.ReadSeeker（embed.FS 返回的文件满足此要求）
    ff, ok := f.(io.ReadSeeker)
    if !ok {
        return errors.New("[FileFS] file does not implement io.ReadSeeker")
    }

    // 使用标准库的 ServeContent，自动处理：
    //   - Content-Type 推断（基于扩展名）
    //   - Range 请求（断点续传）
    //   - Last-Modified / If-Modified-Since（协商缓存）
    http.ServeContent(e.Response, e.Request, fi.Name(), fi.ModTime(), ff)

    return nil
}
```

---

## 七、阶段 6：首次启动的 Installer 机制

当系统中尚无超级用户时，PocketBase 会自动引导创建首个管理员。

在 [apis/installer.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/apis/installer.go) 中：

1. **判断是否需要安装程序**：`needInstallerSuperuser()` 检查超级用户表是否为空
2. **创建临时 installer 用户**：`findOrCreateInstallerSuperuser()` 创建一个邮箱为 `__pocketbase_installer@local.dev` 的临时超管
3. **生成 30 分钟 token** 并打开浏览器访问 `/_/#/pbinstall/{token}`（使用 hash 路由）
4. **前端路由匹配**：[ui/src/auth/pageInstaller.js](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/src/auth/pageInstaller.js) 处理 `#/pbinstall/{token}` hash 路由，用户创建真正的管理员后临时账户被清理

注意 Installer URL 使用的是 `/_/#/pbinstall/{token}`（hash 路由）而非 `/_/pbinstall/{token}`，避免 404。

---

## 八、阶段 7：UI 扩展机制

PocketBase 支持在运行时通过插件注入额外的 UI 资源，由 [apis/extensions.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/apis/extensions.go) 实现。

### 8.1 扩展资源路由

```go
func bindUIExtensions(app core.App) {
    if ui.DistDirFS == nil {
        return
    }

    app.OnServe().Bind(&hook.Handler[*core.ServeEvent]{
        Priority: 9999, // 尽可能靠后执行
        Func: func(se *core.ServeEvent) error {
            uiGroup := se.Router.Group("/_").
                Bind(缓存控制 + CSP + Gzip)

            // 每个扩展注册独立路由：/_/extensions/{name}/{path...}
            for _, ext := range se.UIExtensions {
                uiGroup.GET("/extensions/"+ext.Name+"/{path...}", Static(ext.FS, false))
            }

            // 合并所有扩展的 main.js
            uiGroup.GET("/extensions.js", func(re *core.RequestEvent) error {
                buf := new(bytes.Buffer)
                for _, ext := range se.UIExtensions {
                    // 将每个扩展的 main.js 包装在 (async function(){ ... })(); 中
                    _ = copyExtensionMainjs(buf, ext)
                }
                return re.Stream(200, "text/javascript", buf)
            })

            return se.Next()
        },
    })
}
```

### 8.2 main.js 合并策略

为避免多个扩展的全局作用域冲突，每个扩展的 `main.js` 被包装：

```js
await (async function(){
    /* ... 扩展的 main.js 原始内容 ... */
})();
```

使用 `await` 是为了支持顶层 `await` 语句。前端在 [ui/src/main.js](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/src/main.js#L137-L147) 中加载 `/_/extensions.js`。

---

## 九、完整调用时序图（以 `/_/#/collections` 为例）

```
用户在浏览器输入 http://127.0.0.1:8090/_/#/collections
        │
        │  浏览器行为：hash 部分 #/collections 不发送到服务器
        ▼
[HTTP 请求] GET /_/
        │
        ▼
[Go http.ServeMux 路由匹配] 匹配模式 "GET /_/{path...}"
  PathValue("path") = ""（空字符串）
        │
        ▼
[缓存控制中间件] path == "" → 不设置 Cache-Control（HTML 不缓存）
                  设置 Content-Security-Policy
        │
        ▼
[Gzip 中间件] 根据 Accept-Encoding 决定是否压缩
        │
        ▼
[Static 处理器 (indexFallback=false)]
  ├── filename = filepath.Clean("") = ""
  ├── fs.Stat(fsys, "") → 根目录，fi.IsDir() = true
  ├── URL "/_/" 已以 "/" 结尾 → 不重定向
  └── e.FileFS(fsys, "")
        │
        ▼
[FileFS]
  ├── fsys.Open("") → 打开根目录
  ├── fi.IsDir() == true
  ├── filename = filepath.Join("", "index.html") = "index.html"
  ├── fsys.Open("index.html") → 成功
  └── http.ServeContent(...) → 发送 HTML，200 OK
        │
        ▼
[浏览器解析 HTML]
  ├── 发现 <script src="./assets/index-V68uRsWE.js">
  ├── 当前页面 URL = http://host:8090/_/（目录）
  ├── 相对路径解析: ./assets/index-V68uRsWE.js → /_/assets/index-V68uRsWE.js
  └── 浏览器发起 GET /_/assets/index-V68uRsWE.js
        │
        ▼
[资源请求] GET /_/assets/index-V68uRsWE.js
  PathValue("path") = "assets/index-V68uRsWE.js"
  fs.Stat() → 文件存在
  缓存中间件：path != "" → 设置 Cache-Control: max-age=1209600
  http.ServeContent → 发送 JS 文件，200 OK
        │
        ▼
[前端 JS 执行]
  ├── 加载 /_/extensions.js（扩展脚本）
  ├── app.store._ready = true
  ├── initRouter() 被调用
  ├── window.location.hash = "#/collections"
  └── 路由匹配成功，渲染集合列表页面
```

---

## 十、关键文件索引

| 文件 | 职责 |
|------|------|
| [ui/embed.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/embed.go) | Go embed 嵌入 dist 目录 |
| [ui/embed_no_ui.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/embed_no_ui.go) | no_ui 标签下的空实现 |
| [ui/vite.config.js](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/vite.config.js) | Vite 构建配置（`base: "./"` 相对路径） |
| [ui/.env](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/.env) | 生产环境变量（`PB_BACKEND_URL = "../"`） |
| [ui/package.json](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/package.json) | npm build/dev 脚本 |
| [ui/src/router.js](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/src/router.js) | 前端 hash 路由定义 |
| [ui/src/main.js](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/src/main.js) | 前端入口，加载扩展，初始化路由 |
| [ui/src/pb.js](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/ui/src/pb.js) | JS SDK 初始化（使用相对路径 `PB_BACKEND_URL`） |
| [apis/serve.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/apis/serve.go) | 注册 `/_/{path...}` 路由与缓存/CSP/Gzip 中间件 |
| [apis/base.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/apis/base.go#L93-L171) | `Static()` 静态文件处理器（含 `indexFallback=false`） |
| [tools/router/event.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/tools/router/event.go#L234-L272) | `FileFS()` 实际发送文件（目录自动转 index.html） |
| [apis/extensions.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/apis/extensions.go) | UI 扩展资源路由与 main.js 合并 |
| [apis/installer.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/apis/installer.go) | 首次启动 Installer 引导流程（使用 hash 路由 URL） |
| [cmd/serve.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/cmd/serve.go) | serve CLI 命令入口 |
| [pocketbase.go](file:///d:/fz/0601/solo-dogfeeding/code/162-pocketbase/pocketbase.go) | PocketBase 入口，注册 serve 命令 |
