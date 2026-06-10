# PocketBase 自定义路由与脚本扩展代码链路分析

本文档深入分析 PocketBase 中自定义路由注册、HTTP 请求接入处理以及 JS 脚本运行隔离的完整代码链路。

---

## 一、整体架构概览

PocketBase 的扩展体系分为三层：

```
┌─────────────────────────────────────────────────────────┐
│                    用户扩展层                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │  Go 原生扩展  │  │ pb_hooks/*.js│  │ pb_migrations│   │
│  │ (直接调用API) │  │  JS 脚本钩子 │  │  JS 迁移脚本 │   │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘   │
└─────────┼─────────────────┼──────────────────┼───────────┘
          │                 │                  │
┌─────────▼─────────────────▼──────────────────▼───────────┐
│                    Hook 注册层                             │
│  OnServe / OnModel* / OnRecord* / On*Request 等钩子系统    │
└────────────────────────────┬──────────────────────────────┘
                             │
┌────────────────────────────▼──────────────────────────────┐
│                    路由与请求层                             │
│  tools/router.Router → http.ServeMux → HTTP Server         │
└────────────────────────────┬──────────────────────────────┘
                             │
┌────────────────────────────▼──────────────────────────────┐
│                  JSVM 运行隔离层                            │
│  goja.Runtime 池化 → 独立执行上下文 → 错误隔离              │
└───────────────────────────────────────────────────────────┘
```

---

## 二、扩展注册链路

### 2.1 Go 原生扩展注册

#### 入口：应用启动

从 [pocketbase.go](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/pocketbase.go) 的 `NewWithConfig()` 开始：

1. **创建 BaseApp 实例**：[pocketbase.go#L128-L138](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/pocketbase.go#L128-L138)
   ```go
   pb.App = core.NewBaseApp(core.BaseAppConfig{...})
   ```

2. **初始化所有 Hook**：在 `NewBaseApp()` → `initHooks()` 中完成。见 [core/base.go#L234-L341](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/core/base.go#L234-L341)
   - App 生命周期钩子：`onBootstrap`、`onServe`、`onTerminate`
   - DB 模型钩子：`onModelValidate`、`onModelCreate`、`onModelUpdate`、`onModelDelete` 及其 Execute/AfterSuccess/AfterError 变体
   - Record 代理钩子：`onRecord*` 系列（通过 Model 钩子代理触发）
   - Collection 代理钩子：`onCollection*` 系列
   - API 请求钩子：`onRecord*Request`、`onCollection*Request`、`onSettings*Request` 等
   - Realtime、文件、邮件、认证等专用钩子

3. **注册系统基础钩子**：`registerBaseHooks()` 注册文件清理、Cron 启动、DB 优化等内置逻辑。见 [core/base.go#L1289-L1387](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/core/base.go#L1289-L1387)

#### 用户自定义注册

用户通过 `app.On*()` 方法注册扩展，例如：

```go
// 自定义路由
app.OnServe().Bind(&hook.Handler[*core.ServeEvent]{
    Func: func(e *core.ServeEvent) error {
        e.Router.GET("/api/hello", func(e *core.RequestEvent) error {
            return e.JSON(200, map[string]string{"msg": "hi"})
        })
        return e.Next()
    },
})

// 记录钩子
app.OnRecordAfterCreateSuccess("posts").BindFunc(func(e *core.RecordEvent) error {
    // ...
    return e.Next()
})
```

### 2.2 JSVM 脚本扩展注册

#### 插件注册入口

[plugins/jsvm/jsvm.go](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/jsvm.go) 中的 `Register()` 函数：

1. **注册迁移脚本**：`registerMigrations()` — 扫描 `pb_migrations/*.js`，每个文件独立创建一个 goja VM 执行
2. **注册应用钩子**：`registerHooks()` — 扫描 `pb_hooks/*.pb.js`

#### JS Hook 注册核心流程

[plugins/jsvm/jsvm.go#L236-L349](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/jsvm.go#L236-L349) 中 `registerHooks()` 的执行步骤：

```
registerHooks()
├── 扫描 pb_hooks 目录，匹配 *.pb.js / *.pb.ts 文件
├── 初始化文件监听器（HooksWatch=true 时）
├── 构建共享绑定函数 sharedBinds()
│   ├── require/console/process/buffer Node.js polyfill
│   ├── BindCore/BindDbx/BindSecurity/BindOS/BindFilepath
│   ├── BindHTTP/BindFilesystem/BindForms/BindMails/BindApis
│   └── 注入 $app、$template、__hooks 全局变量
├── 创建执行器 VM 池 executors = newPool(HooksPoolSize, factory)
│   └── 每个 VM 都执行 sharedBinds()
├── 创建 loader VM（单例，用于加载解析 JS）
│   ├── sharedBinds(loader)
│   ├── hooksBinds()   — 反射 core.App 所有 On* 方法，暴露为 JS 函数
│   ├── cronBinds()    — 暴露 cronAdd / cronRemove
│   └── routerBinds()  — 暴露 routerAdd / routerUse
└── 遍历每个 hook 文件
    └── loader.RunScript(defaultScriptPath, content)  → 执行 JS 代码
```

#### JS → Go 桥接：`routerBinds()`

[plugins/jsvm/binds.go#L149-L179](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/binds.go#L149-L179)

```javascript
// JS 侧
routerAdd("GET", "/api/hello", (e) => {
    return e.json(200, { msg: "hi" })
})
```

实际执行流程：
1. `wrapHandlerFunc()` 将 JS 函数编译为 `goja.Program`
2. 注册到 `app.OnServe()` 钩子中
3. OnServe 触发时调用 `e.Router.Route(method, path, wrappedHandler)`

#### JS → Go 桥接：`hooksBinds()`

[plugins/jsvm/binds.go#L42-L102](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/binds.go#L42-L102)

通过反射遍历 `core.App` 的所有 `On*` 方法（排除 `OnServe`），为每个方法生成一个 JS 函数：

```go
// 伪代码逻辑
for each method on app that starts with "On":
    jsName = toSnakeCase(methodName)  // OnRecordCreate → onRecordCreate
    loader.Set(jsName, func(callback string, tags ...string) {
        // 1. 将 JS callback 包装为 goja.Program
        // 2. 通过反射调用 app.OnXxx(tags...) 获取 TaggedHook 实例
        // 3. 构造 Go handler 函数：executors.run(...) → executor.RunProgram(pr)
        // 4. hookInstance.BindFunc(handler)
    })
```

---

## 三、请求接入与路由匹配流程

### 3.1 服务器启动链路

从 `serve` 命令到 HTTP 请求处理的完整路径：

```
cmd.NewServeCommand()            [cmd/serve.go]
    ↓ RunE
apis.Serve(app, config)          [apis/serve.go#L61-L314]
    ├── app.RunAllMigrations()
    ├── apis.NewRouter(app)      [apis/base.go#L19-L57]
    │   ├── 创建 router.Router，注入 eventFactory
    │   ├── 绑定全局中间件：activityLogger / panicRecover / rateLimit /
    │   │       loadAuthToken / superuserIPsWhitelist /
    │   │       securityHeaders / BodyLimit
    │   ├── 创建 /api 路由组，注册所有系统 API 路由
    │   └── bindUIExtensions()  — 注册 /_/extensions/* UI 扩展
    ├── 绑定 CORS 中间件
    ├── 注册 UI 静态路由 /_/{path...}
    ├── 构造 ServeEvent
    ├── app.OnServe().Trigger(serveEvent, finalizer)
    │   │                          ↑ 用户自定义路由在此注册！
    │   └── finalizer:
    │       ├── e.Router.BuildMux() → 生成 http.ServeMux
    │       ├── server.Handler = mux
    │       └── net.Listen + server.Serve()
    └── http.Server.Serve(listener)
```

### 3.2 Router 构建与 Mux 生成

[tools/router/router.go#L60-L158](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/tools/router/router.go#L60-L158) 中 `BuildMux()` → `loadMux()` 的核心逻辑：

#### 路由树结构

```
Router (根 RouterGroup)
├── children: [
│   ├── RouterGroup("/api")
│   │   ├── children: [Route("GET", "/collections", handler), ...]
│   │   └── Middlewares: [...]
│   ├── Route("GET", "/_/extensions.js", handler)
│   └── ...
]
└── Middlewares: [CORS, activityLogger, panicRecover, ...]
```

#### Mux 注册过程

对于每个 Route，递归向上收集所有父 Group 的 Middleware：

```go
// 伪代码
for each route in tree:
    routeHook = new(hook.Hook)
    
    // 1. 递归收集：祖先 Group 的 Middlewares → 当前 Group → 当前 Route
    for parentGroup in ancestors:
        for m in parentGroup.Middlewares:
            if not excluded: routeHook.Bind(m)
    for m in currentGroup.Middlewares:
        if not excluded: routeHook.Bind(m)
    for m in route.Middlewares:
        if not excluded: routeHook.Bind(m)
    
    // 2. 注册到标准库 ServeMux
    pattern = route.Method + " " + concat(prefixes) + route.Path
    // 例: "GET /api/collections"
    
    mux.HandleFunc(pattern, func(w, r) {
        w = &ResponseWriter{w}         // 追踪写入状态
        r.Body = &RereadableReadCloser{r.Body}  // 支持重复读取
        event, cleanup = eventFactory(w, r)    // 创建 RequestEvent
        
        // 3. 触发中间件 + handler 链
        err = routeHook.Trigger(event, route.Action)
        if err != nil: ErrorHandler(w, r, err)
    })
```

### 3.3 Hook 链执行机制

[tools/hook/hook.go](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/tools/hook/hook.go)

#### Handler 结构

```go
type Handler[T Resolver] struct {
    Func     func(T) error  // 处理函数
    Id       string         // 唯一标识（用于移除/去重）
    Priority int            // 执行优先级（越小越先执行）
}
```

#### Trigger 执行模型：洋葱模型

[tools/hook/hook.go#L153-L174](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/tools/hook/hook.go#L153-L174)

```go
func (h *Hook[T]) Trigger(event T, oneOffHandlerFuncs ...func(T) error) error {
    // handlers = [注册的handler列表] + [oneOff回调]
    
    // 从后往前构建嵌套调用链：
    // handler1 -> handler2 -> handler3 -> route.Action
    // 实际执行: handler1(e) { ... e.Next() → handler2(e) { ... e.Next() → ... } }
    for i := len(handlers) - 1; i >= 0; i-- {
        old := event.nextFunc()
        event.setNextFunc(func() error {
            event.setNextFunc(old)
            return handlers[i](event)
        })
    }
    
    return event.Next()  // 开始执行第一个
}
```

每个 handler 必须显式调用 `e.Next()` 才能继续链执行，否则链条中断。

#### TaggedHook：标签过滤

[tools/hook/tagged.go](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/tools/hook/tagged.go)

```go
// 带标签的注册（只针对 posts collection 触发）
app.OnRecordAfterCreateSuccess("posts").BindFunc(...)

// TaggedHook.Bind 包装了原始 Func：
handler.Func = func(e T) error {
    if h.CanTriggerOn(e.Tags()) {  // 检查标签匹配
        return fn(e)               // 匹配才执行用户逻辑
    }
    return e.Next()                // 否则直接跳过
}
```

Tag 来源：
- Record → Collection Id + Collection Name
- Collection → 自身 Id + Name
- Model → TableName

### 3.4 中间件排除机制

[tools/router/route.go#L45-L73](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/tools/router/route.go#L45-L73) 和 [tools/router/group.go#L73-L109](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/tools/router/group.go#L73-L109)

Route / RouterGroup 均维护 `excludedMiddlewares map[string]struct{}`：
- `Unbind(id)` 将中间件加入排除列表
- `loadMux()` 组装链时检查三层排除：父 Group → 当前 Group → 当前 Route

---

## 四、JS 脚本运行隔离机制

### 4.1 VM 池化设计

[plugins/jsvm/pool.go](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/pool.go)

```go
type vmsPool struct {
    factory func() *goja.Runtime
    items   []*poolItem  // 预创建的 VM
}

type poolItem struct {
    mux  sync.Mutex
    busy bool
    vm   *goja.Runtime
}
```

**池化策略**：
- 启动时根据 `HooksPoolSize` 预创建 N 个 goja.Runtime
- 每个 VM 都执行完整的 `sharedBinds()` 绑定
- 请求时从池获取空闲 VM，用完归还
- 池满时临时创建新 VM（不归池）

### 4.2 Loader VM 与 Executor VM 分离

```
┌──────────────────────────────────────────────┐
│               Loader VM (单例)                │
│  - 加载 pb_hooks/*.pb.js 文件                 │
│  - 解析 JS 中的 routerAdd / onRecordCreate 等 │
│  - 将用户回调编译为 goja.Program（预编译）     │
│  - 不参与实际请求执行                         │
└──────────────────────────────────────────────┘
           ↓ 产出 goja.Program
┌──────────────────────────────────────────────┐
│          Executor VM Pool (多个实例)          │
│  - 每个请求从池中获取一个 VM                  │
│  - 设置 $app = 当前请求的 App 实例            │
│  - 设置 __args = [RequestEvent, ...]          │
│  - RunProgram(precompiledProgram)             │
│  - 执行完毕清理 __args，归还 VM               │
└──────────────────────────────────────────────┘
```

关键代码见 [plugins/jsvm/binds.go#L181-L213](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/binds.go#L181-L213) 中 `wrapHandlerFunc`：

```go
wrappedHandler := func(e *core.RequestEvent) error {
    return executors.run(func(executor *goja.Runtime) error {
        executor.Set("$app", e.App)       // 注入当前请求上下文
        executor.Set("__args", []any{e})  // 注入事件参数
        res, err := executor.RunProgram(pr)
        executor.Set("__args", goja.Undefined())  // 清理
        
        if resErr := checkGojaValueForError(e.App, res); resErr != nil {
            return resErr
        }
        return normalizeException(err)
    })
}
```

### 4.3 全局绑定命名空间

所有 VM 共享相同的绑定策略，暴露以下全局对象：

| 命名空间 | 说明 | 绑定位置 |
|---------|------|---------|
| `$app` | 当前 App 实例（每个请求动态设置） | [binds.go#L305](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/binds.go#L305) |
| `$template` | 模板渲染注册表 | [binds.go#L306](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/binds.go#L306) |
| `$security.*` | MD5/SHA/JWT/加密等安全工具 | [binds.go#L709-L750](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/binds.go#L709-L750) |
| `$os.*` | 文件系统/进程/环境变量 | [binds.go#L806-L830](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/binds.go#L806-L830) |
| `$http.*` | HTTP 客户端请求 | [binds.go#L888-L1040](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/binds.go#L888-L1040) |
| `$apis.*` | 路由处理器/中间件工厂 | [binds.go#L845-L882](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/binds.go#L845-L882) |
| `$dbx.*` | SQL 查询表达式构建器 | [binds.go#L669-L690](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/binds.go#L669-L690) |
| `$filesystem.*` | S3/本地文件系统 | [binds.go#L756-L775](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/binds.go#L756-L775) |
| `$filepath.*` | 路径操作工具 | [binds.go#L781-L800](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/binds.go#L781-L800) |
| `$mails.*` | 邮件发送辅助 | [binds.go#L695-L704](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/binds.go#L695-L704) |
| `on*` | 反射生成的所有事件钩子函数 | [binds.go#L42-L102](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/binds.go#L42-L102) |
| `routerAdd / routerUse` | 自定义路由注册 | [binds.go#L149-L179](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/binds.go#L149-L179) |
| `cronAdd / cronRemove` | 定时任务 | [binds.go#L104-L147](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/binds.go#L104-L147) |
| `Record / Collection / Field*` | 模型构造器 | [binds.go#L465-L570](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/binds.go#L465-L570) |
| `Middleware` | 中间件构造器 | [binds.go#L593-L604](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/binds.go#L593-L604) |
| `require / console / process` | Node.js polyfill | goja_nodejs 库 |

### 4.4 错误隔离与异常规范化

[plugins/jsvm/binds.go#L1044-L1091](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/binds.go#L1044-L1091)

```go
// 1. 检查 JS 返回值是否是 Go error
checkGojaValueForError(app, value)
    ├── 若 value 是 error → 直接返回
    └── 若 value 是 Promise → 警告并尝试提取异常

// 2. 规范化 goja.Exception（JS 抛出的异常）
normalizeException(err)
    └── goja.Exception.Value().Export()
        ├── 若为 error 类型 → 直接返回
        └── 若为 map (goja.GoError) → 提取 value 字段的 error
```

此外，`registerHooks()` 在 Serve 阶段注册了全局异常规范化中间件：
```go
app.OnServe().BindFunc(func(e *core.ServeEvent) error {
    e.Router.BindFunc(p.normalizeServeExceptions)
    return e.Next()
})
```

### 4.5 模块系统共享

- `require.Registry` 在所有 VM 间共享（模块缓存全局共用）
- `template.Registry` 同样全局共享
- 每个 VM 独立调用 `registry.Enable(vm)` 绑定到自己的运行时

---

## 五、一次自定义路由请求的完整旅程

以 `routerAdd("GET", "/api/hello", handler)` 注册的路由为例，追踪请求处理全链路：

```
1. 用户访问 GET http://localhost:8090/api/hello
   ↓
2. net/http Server 接收，分发到 http.ServeMux
   ↓
3. ServeMux 匹配 "GET /api/hello"，调用 router.loadMux 注册的 HandleFunc
   ↓
4. 包装 ResponseWriter（追踪写入）和 Request.Body（支持重读）
   ↓
5. eventFactory 创建 core.RequestEvent
   ├── App = 当前 PocketBase 实例
   ├── Auth = 由 loadAuthToken 中间件填充
   └── 内嵌 router.Event { Response, Request, data store }
   ↓
6. 组装并执行 Hook 链
   ├── 全局中间件: activityLogger → panicRecover → rateLimit →
   │              loadAuthToken → superuserIPsWhitelist →
   │              securityHeaders → BodyLimit → normalizeServeExceptions
   ├── /api 组中间件（如有）
   └── 路由级中间件（如有）
   ↓
7. 到达 wrappedHandler（由 wrapHandlerFunc 生成）
   ├── 从 vmsPool 获取空闲 goja.Runtime
   ├── executor.Set("$app", e.App)
   ├── executor.Set("__args", []any{e})
   ├── executor.RunProgram(precompiledProgram)
   │   └── 执行用户 JS: (e) => e.json(200, { msg: "hi" })
   ├── 清理 __args
   ├── 归还 VM 到池
   └── 规范化返回的 error
   ↓
8. 若有 error，router.ErrorHandler 输出 JSON 错误响应
   若无 error，响应已由用户 handler 写入（e.json 内部调用 Response.Write）
```

---

## 六、关键数据结构索引

| 结构体 | 位置 | 职责 |
|--------|------|------|
| `PocketBase` | [pocketbase.go#L33-L44](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/pocketbase.go#L33-L44) | 应用启动器，CLI 命令入口 |
| `BaseApp` | [core/base.go#L75-L193](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/core/base.go#L75-L193) | App 接口实现，持有所有 Hook 实例 |
| `Router` | [tools/router/router.go#L44-L58](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/tools/router/router.go#L44-L58) | 路由注册器，构建 http.ServeMux |
| `RouterGroup` | [tools/router/group.go#L16-L22](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/tools/router/group.go#L16-L22) | 路由分组，共享前缀和中间件 |
| `Route` | [tools/router/route.go#L5-L12](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/tools/router/route.go#L5-L12) | 单个路由定义 |
| `Hook` | [tools/hook/hook.go#L54-L57](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/tools/hook/hook.go#L54-L57) | 通用事件钩子，洋葱模型执行 |
| `Handler` | [tools/hook/hook.go#L13-L32](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/tools/hook/hook.go#L13-L32) | 钩子处理器（带 Id 和 Priority） |
| `TaggedHook` | [tools/hook/tagged.go#L30-L34](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/tools/hook/tagged.go#L30-L34) | 带标签过滤的钩子代理 |
| `RequestEvent` | [core/event_request.go#L19-L29](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/core/event_request.go#L19-L29) | HTTP 请求事件（含 App、Auth、RequestInfo） |
| `ServeEvent` | [core/events.go#L105-L137](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/core/events.go#L105-L137) | 服务启动事件（含 Router、Server 引用） |
| `vmsPool` | [plugins/jsvm/pool.go#L16-L19](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/pool.go#L16-L19) | goja.Runtime 连接池 |
