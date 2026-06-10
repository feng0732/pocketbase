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

### 2.3 HooksWatch 热重载机制

[plugins/jsvm/jsvm.go#L365-L463](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/jsvm.go#L365-L463) 中 `watchHooks()` 的实现：

#### 核心机制：非增量更新，而是全进程重启

HooksWatch **不会**对已注册的路由或 Hook 做任何增量修改。当 `pb_hooks` 目录下文件发生变化时，它触发的是**整个应用进程的重启**。

```
watchHooks()
├── 创建 fsnotify.Watcher
├── 递归添加 pb_hooks 下所有非隐藏、非 node_modules 子目录
│   └── filepath.WalkDir → watcher.Add(path)
├── 注册 OnTerminate 钩子：关闭 watcher + 防抖定时器
└── 启动 goroutine 监听事件
    ├── 收到 watcher.Events:
    │   ├── 启动/重置 50ms 防抖定时器
    │   └── 定时器触发后：
    │       ├── Windows: 打印黄色警告 "File xxx changed, please restart the app manually"
    │       │   （因为 Windows 不支持 execve 替换进程）
    │       └── 非 Windows: color.Yellow("restarting...") → p.app.Restart()
    └── 收到 watcher.Errors: 打印红色错误
```

#### app.Restart() 的效果

调用 `app.Restart()` 后：
1. 当前进程通过 `syscall.Exec`（类 Unix）替换自身为新的进程镜像
2. 新进程从头执行完整启动流程：
   - `NewWithConfig()` → `core.NewBaseApp()` → 重新创建所有 Hook 实例
   - `jsvm.Register()` → `registerHooks()` → 重新扫描 `pb_hooks`、重建 Loader/Executor VM、重新执行所有 JS 文件
   - `serve` 命令 → `apis.Serve()` → `apis.NewRouter()` → `OnServe.Trigger()` → 重新注册所有路由
3. 旧进程中已注册到 Hook 上的 JS 回调、VM 池中的 goja.Program、已编译的路由等全部随进程销毁

**关键结论**：HooksWatch 不做路由卸载、Hook 解绑或 VM 热替换，它依赖操作系统级别的进程重启来达到"热重载"效果。这意味着：
- Windows 用户必须手动重启进程，无法自动热重载
- 已建立的 HTTP 连接会被中断
- Cron 任务会重新调度

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

### 4.5 routerAdd/routerUse 与 on* hook 的执行上下文差异

PocketBase JSVM 中有三类 JS 回调注册方式，它们的 `$app` 注入机制、执行时机和上下文环境完全不同。

#### 三类注册方式对比

| 维度 | `routerAdd` / `routerUse` | `onRecord*` / `onModel*` 等 on* hook | `cronAdd` |
|------|--------------------------|--------------------------------------|-----------|
| 绑定函数 | `routerBinds()` | `hooksBinds()` | `cronBinds()` |
| 代码位置 | [binds.go#L149-L179](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/binds.go#L149-L179) | [binds.go#L42-L102](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/binds.go#L42-L102) | [binds.go#L104-L147](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/binds.go#L104-L147) |
| 注册时机 | Loader VM 执行 JS 时 | Loader VM 执行 JS 时 | Loader VM 执行 JS 时 |
| 执行时机 | HTTP 请求到达时 | 业务事件触发时（如记录创建） | Cron 定时器触发时 |
| 包装函数 | `wrapHandlerFunc` / `wrapMiddlewares` | 反射生成的 `reflect.MakeFunc` | `app.Cron().Add()` 内联函数 |

#### $app 注入方式的差异

这是三类回调最关键的区别：

##### 1. routerAdd/routerUse：Go 侧显式注入 e.App

[plugins/jsvm/binds.go#L193-L207](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/binds.go#L193-L207) 中的 `wrapHandlerFunc`：

```go
wrappedHandler := func(e *core.RequestEvent) error {
    return executors.run(func(executor *goja.Runtime) error {
        // Go 侧在执行前显式设置 $app = RequestEvent.App
        executor.Set("$app", e.App)       // ← 关键差异 1
        executor.Set("__args", []any{e})
        res, err := executor.RunProgram(pr)
        executor.Set("__args", goja.Undefined())
        // ...
    })
}
```

`wrapMiddlewares` 中对所有中间件的处理完全相同，见 [binds.go#L248-L263](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/binds.go#L248-L263) 和 [binds.go#L268-L283](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/binds.go#L268-L283)。

**用户 JS 侧**：
```javascript
routerAdd("GET", "/api/test", (e) => {
    // $app 由 Go 侧预先设置为 e.App
    // e.app 同时也可用（RequestEvent 的字段）
    console.log($app === e.app)  // true
    return e.json(200, {})
})
```

##### 2. on* hook：JS 侧包装函数从事件对象提取

[plugins/jsvm/binds.go#L60-L63](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/binds.go#L60-L63) 中的 `hooksBinds`：

```go
loader.Set(jsName, func(callback string, tags ...string) {
    // ★ 关键：在 JS 源码层面注入 $app 赋值
    callback = `function(e) { $app = e.app; return (` + callback + `).call(undefined, e) }`
    //                 ↑ 在 JS 执行时自己从 e.app 提取
    
    pr := goja.MustCompile(defaultScriptPath, 
        "{("+callback+").apply(undefined, __args)}", true)
    
    // ... 反射获取 hookInstance ...
    hookBindFunc.Call([]reflect.Value{handler})  // handler 内：
    // └─ executor.Set("$app", goja.Undefined())  ← 先设为 undefined！
    //    executor.Set("__args", handlerArgs)
    //    executor.RunProgram(pr)  // → 执行包装后的 JS，JS 内部 $app = e.app
    //    executor.Set("__args", goja.Undefined())
})
```

**用户 JS 侧**：
```javascript
onRecordAfterCreateSuccess((e) => {
    // 执行流程：
    // 1. Go 侧 executor.Set("$app", undefined)
    // 2. Go 侧 executor.Set("__args", [RecordEvent])
    // 3. JS 侧：包装函数执行 → $app = e.app（从事件对象赋值）
    // 4. JS 侧：调用用户 callback
    console.log($app === e.app)  // true，但赋值时机不同
})
```

**与 routerAdd 的本质区别**：
- routerAdd：Go 侧在 RunProgram **之前**用 `executor.Set("$app", e.App)` 注入
- on* hook：Go 侧先 `executor.Set("$app", undefined)`，然后靠**编译到 JS 源码里的赋值语句** `$app = e.app` 在 JS 运行时注入

这个差异的原因是：on* hook 的事件类型很多（RecordEvent、ModelEvent、CollectionEvent、MailerEvent 等），Go 侧反射生成的通用 handler 不知道具体事件类型，无法在 Go 层面统一提取 `.App` 字段，所以把赋值逻辑下沉到了 JS 侧。

##### 3. cronAdd：完全不注入 $app

[plugins/jsvm/binds.go#L105-L121](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/binds.go#L105-L121)：

```go
cronAdd := func(jobId, cronExpr, handler string) {
    pr := goja.MustCompile(defaultScriptPath, "{("+handler+").apply(undefined)}", true)
    //                                                               ↑ 无参数！
    
    err := app.Cron().Add(jobId, cronExpr, func() {
        err := executors.run(func(executor *goja.Runtime) error {
            // ★ 完全没有 Set("$app", ...)！
            // 也没有 Set("__args", ...)
            _, err := executor.RunProgram(pr)
            return err
        })
        // ...
    })
}
```

**用户 JS 侧**：
```javascript
cronAdd("myJob", "* * * * *", () => {
    // $app 使用的是 sharedBinds 中设置的默认全局值（启动时的 App 实例）
    // 没有 e 参数，无法从事件对象获取
    // cronAdd/cronRemove 函数在 executor VM 中也被绑定
})
```

注意 `cronBinds` 还额外修改了 executors 的 factory，将 `cronAdd` / `cronRemove` 注入到**所有 executor VM**中，见 [binds.go#L133-L146](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/binds.go#L133-L146)。

#### sharedBinds 中的默认 $app

[plugins/jsvm/jsvm.go#L288-L312](file:///d:/fz/0601/solo-dogfeeding/code/161-pocketbase/plugins/jsvm/jsvm.go#L288-L312) 中所有 VM 创建时都会执行：

```go
sharedBinds := func(vm *goja.Runtime) {
    // ...
    vm.Set("$app", p.app)  // ← 启动时的 App 实例作为默认值
    vm.Set("$template", templateRegistry)
    vm.Set("__hooks", absHooksDir)
}
```

这个默认值在不同场景下的有效性：
- **routerAdd/routerUse**：每次执行前被 `executor.Set("$app", e.App)` 覆盖
- **on* hook**：每次执行前被 `executor.Set("$app", undefined)` 覆盖，之后 JS 包装函数又从 `e.app` 重新赋值
- **cronAdd**：使用这个默认值（因为 Cron handler 中不修改 $app）

#### 执行上下文差异总结

```
┌─────────────────────────────────────────────────────────────────┐
│                   VM 初始状态（sharedBinds）                     │
│  $app = p.app（启动时实例）                                      │
│  __args = undefined                                             │
│  require / $http / $os 等全局就绪                                │
└──────────────────────┬──────────────────────────────────────────┘
                       │
          ┌────────────┼─────────────────┐
          ▼            ▼                 ▼
     routerAdd     on* hook          cronAdd
     routerUse
          │            │                 │
          ▼            ▼                 ▼
  Set("$app", e.App)  Set("$app",       不修改 $app
  Set("__args", [e])  undefined)        不设置 __args
                       Set("__args",
                       [event])
          │            │                 │
          ▼            ▼                 ▼
  RunProgram(pr)   RunProgram(pr)     RunProgram(pr)
  └─ 用户 JS 直接   └─ 包装 JS 先执行   └─ 用户 JS 直接执行
     执行             $app = e.app
                     └─ 再执行用户 JS
          │            │                 │
          ▼            ▼                 ▼
  Set("__args",     Set("__args",       (无清理)
  undefined)        undefined)
```

### 4.6 模块系统共享

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
