# PocketBase API 请求流程梳理

本文从代码实现角度梳理 PocketBase 中 API 请求从进入到返回的完整流程，包括路由注册、中间件链、鉴权机制、上下文传递和错误处理。

---

## 1. 整体架构概览

PocketBase 的路由系统建立在标准库 `net/http.ServeMux` 之上，通过自研的 `tools/router` 包提供分组、中间件等能力，核心事件处理通过 `tools/hook` 包的 Hook 机制实现中间件链的洋葱模型执行。

```
客户端请求
    ↓
http.Server.Serve() (标准库)
    ↓
http.ServeMux (BuildMux 构建的路由表)
    ↓
包装 ResponseWriter / Request Body
    ↓
EventFactory 创建 RequestEvent
    ↓
Hook.Trigger() 触发中间件链 + 路由 Action
    ↓
ErrorHandler 统一错误处理
    ↓
响应返回
```

---

## 2. 路由注册与构建

### 2.1 入口：启动服务器

命令入口在 [cmd/serve.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/cmd/serve.go)，调用 `apis.Serve()`。

在 [apis/serve.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/serve.go#L51-L314) 的 `Serve()` 函数中：

1. 运行数据库迁移
2. 调用 `NewRouter(app)` 创建并初始化路由
3. 注册全局中间件（CORS、www 重定向等）
4. 通过 `OnServe` Hook 调用 `Router.BuildMux()` 构建标准 `http.Handler`
5. 启动 `http.Server` 监听请求

### 2.2 路由初始化

[apis/base.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/base.go#L19-L57) 中的 `NewRouter()` 完成路由的初始化：

```go
func NewRouter(app core.App) (*router.Router[*core.RequestEvent], error) {
    pbRouter := router.NewRouter(func(w http.ResponseWriter, r *http.Request) (*core.RequestEvent, router.EventCleanupFunc) {
        event := new(core.RequestEvent)
        event.Response = w
        event.Request = r
        event.App = app
        return event, nil
    })

    // 注册全局默认中间件
    pbRouter.Bind(activityLogger())
    pbRouter.Bind(panicRecover())
    pbRouter.Bind(rateLimit())
    pbRouter.Bind(loadAuthToken())
    pbRouter.Bind(superuserIPsWhitelist())
    pbRouter.Bind(securityHeaders())
    pbRouter.Bind(BodyLimit(DefaultMaxBodySize))

    // 注册 /api 分组路由
    apiGroup := pbRouter.Group("/api")
    bindSettingsApi(app, apiGroup)
    bindCollectionApi(app, apiGroup)
    bindRecordCrudApi(app, apiGroup)
    bindRecordAuthApi(app, apiGroup)
    // ... 更多 API 分组

    return pbRouter, nil
}
```

### 2.3 路由分组与注册

路由系统核心类型定义在 [tools/router/](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/tools/router)：

- `Router[T]` —— 顶层路由，持有 `RouterGroup` 和事件工厂函数
- `RouterGroup[T]` —— 路由分组，拥有前缀 Prefix、中间件列表和子节点（子分组或路由）
- `Route[T]` —— 具体路由，包含 Method、Path、Action 处理函数和路由级中间件

分组注册示例（[apis/record_auth.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/record_auth.go#L10-L69)）：

```go
func bindRecordAuthApi(app core.App, rg *router.RouterGroup[*core.RequestEvent]) {
    sub := rg.Group("/collections/{collection}")

    sub.GET("/auth-methods", recordAuthMethods).Bind(
        collectionPathRateLimit("", "listAuthMethods"),
    )

    sub.POST("/auth-refresh", recordAuthRefresh).Bind(
        collectionPathRateLimit("", "authRefresh"),
        RequireSameCollectionContextAuth(""),
    )
    // ...
}
```

### 2.4 BuildMux：构建标准 http.Handler

[tools/router/router.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/tools/router/router.go#L61-L158) 中 `BuildMux()` 递归遍历所有分组和路由，为每个路由注册标准 `http.ServeMux` handler：

核心逻辑：
1. 递归遍历 RouterGroup → RouterGroup / Route
2. 对每个 Route，拼接完整路径 pattern（如 `GET /api/collections/users/records`）
3. 按优先级顺序组装中间件链：父分组 → 当前分组 → 当前路由，同时跳过 `excludedMiddlewares`
4. 调用 `mux.HandleFunc(pattern, handler)` 注册到标准 ServeMux

handler 的实际逻辑：

```go
mux.HandleFunc(pattern, func(resp http.ResponseWriter, req *http.Request) {
    // 包装 ResponseWriter，追踪写入状态和状态码
    resp = &ResponseWriter{ResponseWriter: resp}

    // 包装 Request Body，允许重复读取
    body := &RereadableReadCloser{ReadCloser: req.Body}
    defer body.Close()
    req.Body = body

    // 通过事件工厂创建 RequestEvent
    event, cleanupFunc := r.eventFactory(resp, req)

    // 触发 Hook 链（中间件 + Action）
    err := routeHook.Trigger(event, v.Action)
    if err != nil {
        ErrorHandler(resp, req, err) // 统一错误处理
    }

    if cleanupFunc != nil {
        cleanupFunc()
    }
})
```

---

## 3. 中间件链：Hook 机制的洋葱模型

### 3.1 Hook 核心结构

[tools/hook/hook.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/tools/hook/hook.go) 定义了 `Hook[T]` 和 `Handler[T]`：

```go
type Handler[T Resolver] struct {
    Func     func(T) error  // 中间件/处理函数
    Id       string         // 唯一标识，可用于解绑
    Priority int            // 执行优先级，越小越先执行
}

type Hook[T Resolver] struct {
    handlers []*Handler[T]  // 按 Priority 排序的处理器列表
    mu       sync.RWMutex
}
```

### 3.2 Resolver 接口与 Event 基类

[tools/hook/event.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/tools/hook/event.go)：

```go
type Resolver interface {
    Next() error
    nextFunc() func() error
    setNextFunc(f func() error)
}

type Event struct {
    next func() error  // 指向链中的下一个函数
}

func (e *Event) Next() error {
    if e.next != nil {
        return e.next()
    }
    return nil
}
```

### 3.3 Hook.Trigger：洋葱链的构建

`Hook.Trigger()` 是核心，通过从后往前遍历 handlers，构建闭包形成调用链：

```go
func (h *Hook[T]) Trigger(event T, oneOffHandlerFuncs ...func(T) error) error {
    // 复制 handlers + 一次性 handlers
    handlers := make([]func(T) error, 0, len(h.handlers)+len(oneOffHandlerFuncs))
    for _, handler := range h.handlers {
        handlers = append(handlers, handler.Func)
    }
    handlers = append(handlers, oneOffHandlerFuncs...)

    event.setNextFunc(nil)

    // 从后往前构建链
    for i := len(handlers) - 1; i >= 0; i-- {
        i := i
        old := event.nextFunc()
        event.setNextFunc(func() error {
            event.setNextFunc(old)        // 执行前还原，用于 handler 内多次调用 Next()
            return handlers[i](event)     // 执行当前 handler
        })
    }

    return event.Next()  // 启动链
}
```

执行顺序示意（3 个中间件 + 1 个 Action）：

```
m1.Before → m2.Before → m3.Before → Action → m3.After → m2.After → m1.After
```

每个中间件通过调用 `e.Next()` 将控制权传递给链中的下一环，`Next()` 返回后执行后续逻辑。

### 3.4 默认全局中间件执行顺序

中间件按 `Priority` 排序执行（值越小越先执行），默认顺序定义在 [apis/middlewares.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/middlewares.go#L31-L56)：

| 优先级 | 中间件 | ID | 作用 |
|--------|--------|----|------|
| -99999 | wwwRedirect | pbWWWRedirect | www 域名重定向 |
| DefaultRateLimit - 40 | activityLogger | pbActivityLogger | 请求日志记录 |
| DefaultRateLimit - 30 | panicRecover | pbPanicRecover | panic 恢复 |
| DefaultRateLimit - 20 | loadAuthToken | pbLoadAuthToken | 加载认证 Token |
| DefaultRateLimit - 10 | securityHeaders | pbSecurityHeaders | 安全响应头 |
| Default | rateLimit | pbRateLimit | 速率限制 |
| DefaultLoadAuthToken + 5 | superuserIPsWhitelist | pbSuperuserIPsWhitelist | 超级用户 IP 白名单 |
| Default | BodyLimit | (匿名) | 请求体大小限制 |

> 注意：`activityLogger` 在最外层，能记录所有中间件执行时间和最终结果；`panicRecover` 次之，能捕获内部 panic。

### 3.5 中间件解绑与排除

RouterGroup 和 Route 都支持 `Unbind(middlewareIds...)`，解绑时会：
1. 从自身 Middlewares 列表移除
2. 记录到 `excludedMiddlewares` map，父级同名中间件在链构建时也会被跳过

示例（[apis/record_crud.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/record_crud.go#L28)）：

```go
subGroup := rg.Group("/collections/{collection}/records").Unbind(DefaultRateLimitMiddlewareId)
```

---

## 4. 鉴权流程

### 4.1 Token 加载：loadAuthToken 中间件

[apis/middlewares.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/middlewares.go#L174-L221) 中 `loadAuthToken()` 是**被动的全局中间件**，不强制鉴权，只负责尝试解析：

```go
func loadAuthToken() *hook.Handler[*core.RequestEvent] {
    return &hook.Handler[*core.RequestEvent]{
        Id:       DefaultLoadAuthTokenMiddlewareId,
        Priority: DefaultLoadAuthTokenMiddlewarePriority,
        Func: func(e *core.RequestEvent) error {
            if e.Auth != nil {
                return e.Next() // 已被其他中间件加载
            }

            token := getAuthTokenFromRequest(e)
            if token == "" {
                return e.Next() // 无 Token，跳过
            }

            // 尝试通过 Token 查找认证记录
            record, err := e.App.FindAuthRecordByToken(token, core.TokenTypeAuth)
            if err != nil {
                e.App.Logger().Debug("loadAuthToken failure", "error", err)
            } else if record != nil {
                e.Auth = record // 认证成功，存入事件
            }

            return e.Next()
        },
    }
}
```

Token 提取支持 `Authorization: xxx` 和 `Authorization: Bearer xxx` 两种格式。

### 4.2 Token 验证：FindAuthRecordByToken

[core/record_query.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/core/record_query.go#L483-L530) 中的 `FindAuthRecordByToken()`：

1. 调用 `security.ParseUnverifiedJWT()` 解析 JWT（不验证签名），提取 `id`、`collectionId`、`type` 等 claims
2. 验证 tokenType 是否在允许的 `validTypes` 中
3. 根据 collectionId 和 id 查询 Record
4. 确定对应类型的签名密钥（如 `AuthToken.Secret`），与 record 的 `TokenKey` 拼接
5. 调用 `security.ParseJWT(token, key)` **正式验证签名和有效期**

Token 生成在 [core/record_tokens.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/core/record_tokens.go)，使用 HS256 签名，密钥 = `record.TokenKey() + collection.XXXTToken.Secret`。

### 4.3 强制鉴权中间件

`loadAuthToken` 不报错，真正的鉴权通过路由级中间件强制要求：

| 中间件 | 作用 |
|--------|------|
| `RequireGuestOnly()` | 必须未登录 |
| `RequireAuth(optCollectionNames...)` | 必须已登录，可限制集合 |
| `RequireSuperuserAuth()` | 必须是超级用户（`_superusers` 集合） |
| `RequireSuperuserOrOwnerAuth(ownerIdPathParam)` | 超级用户或资源所有者本人 |
| `RequireSameCollectionContextAuth(collectionPathParam)` | 认证记录必须来自指定集合 |

示例（[apis/middlewares.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/middlewares.go#L84-L104)）：

```go
func RequireAuth(optCollectionNames ...string) *hook.Handler[*core.RequestEvent] {
    return &hook.Handler[*core.RequestEvent]{
        Id:   DefaultRequireAuthMiddlewareId,
        Func: func(e *core.RequestEvent) error {
            if e.Auth == nil {
                return e.UnauthorizedError("The request requires valid record authorization token.", nil)
            }
            if len(optCollectionNames) > 0 && !slices.Contains(optCollectionNames, e.Auth.Collection().Name) {
                return e.ForbiddenError("The authorized record is not allowed to perform this action.", nil)
            }
            return e.Next()
        },
    }
}
```

### 4.4 超级用户 IP 白名单

[apis/middlewares.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/middlewares.go#L305-L325) 的 `superuserIPsWhitelist()` 在 Token 加载后执行，如果当前是超级用户认证，会检查其 IP 是否在 `Settings.SuperuserIPs` 白名单中。

### 4.5 API 规则级鉴权

除了中间件层的粗粒度鉴权，CRUD API 还会在 handler 内根据 collection 的 `ListRule`/`ViewRule`/`CreateRule`/`UpdateRule`/`DeleteRule` 进行细粒度的规则过滤（见 [apis/record_crud.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/record_crud.go)），这些规则会被解析为 SQL WHERE 条件。

---

## 5. 上下文传递

### 5.1 RequestEvent：核心上下文载体

[core/event_request.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/core/event_request.go#L18-L29)：

```go
type RequestEvent struct {
    App               App            // 全局应用实例
    cachedRequestInfo *RequestInfo   // 请求信息缓存
    Auth              *Record        // 当前认证记录
    router.Event                     // 嵌入：Response、Request、Store
    mu                sync.Mutex
}
```

`RequestEvent` 嵌入了 `router.Event`，后者又包含：
- `Response http.ResponseWriter`（被包装为 ResponseWriter）
- `Request *http.Request`
- `data store.Store[string, any]`——通用键值存储

### 5.2 Event Store：中间件间数据共享

[tools/router/event.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/tools/router/event.go#L127-L150) 通过 `Get/Set/SetAll/GetAll` 在事件生命周期内传递数据：

```go
e.Set(requestEventKeyExecStart, time.Now())        // activityLogger 记录开始时间
e.Set(requestEventKeySkipSuccessActivityLog, true) // Static 处理中标记跳过成功日志
e.Set(RequestEventKeyInfoContext, "expand")        // 标记请求上下文类型
```

内置常量键：
- `__execStart` —— 请求起始时间
- `__skipSuccessActivityLogger` —— 是否跳过成功请求日志
- `pbLogMeta` —— 额外日志元数据
- `infoContext` —— RequestInfo 的 context 值

### 5.3 RequestInfo：结构化请求快照

[core/event_request.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/core/event_request.go#L167-L180)：

```go
type RequestInfo struct {
    Query   map[string]string  // URL 查询参数（首值）
    Headers map[string]string  // 请求头（首值，键转为 snake_case）
    Body    map[string]any     // 请求体解析结果
    Auth    *Record            // 当前认证记录
    Method  string
    Context string             // default/expand/realtime/batch/oauth2/...
}
```

通过 `e.RequestInfo()` 获取，结果会缓存（除 Auth 和 Context 每次刷新）。常用于 API 规则过滤（`@request.*`）。

### 5.4 App Hook 事件上下文

API handler 内部还会触发应用级事件，例如 `OnRecordCreateRequest`：

```go
event := new(core.RecordRequestEvent)
event.RequestEvent = e           // 嵌入原 RequestEvent
event.Collection = collection
event.Record = record

return e.App.OnRecordCreateRequest().Trigger(event, func(e *core.RecordRequestEvent) error {
    // 在 Hook 链中处理，最终写入响应
    return e.JSON(http.StatusOK, e.Record)
})
```

所有请求事件类型定义在 [core/events.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/core/events.go)。

---

## 6. 错误处理与返回

### 6.1 ApiError：统一错误结构

[tools/router/error.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/tools/router/error.go#L35-L47)：

```go
type ApiError struct {
    rawData any            // 原始错误（内部调试用）
    Data    map[string]any `json:"data"`    // 公开安全的错误详情
    Message string         `json:"message"` // 公开错误消息
    Status  int            `json:"status"`  // HTTP 状态码
}
```

便捷构造函数：
- `NewBadRequestError(message, rawData)` —— 400
- `NewUnauthorizedError(message, rawData)` —— 401
- `NewForbiddenError(message, rawData)` —— 403
- `NewNotFoundError(message, rawData)` —— 404
- `NewTooManyRequestsError(message, rawData)` —— 429
- `NewInternalServerError(message, rawData)` —— 500

RequestEvent 也提供了同名便捷方法（[tools/router/event.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/tools/router/event.go#L291-L320)），如 `e.BadRequestError(...)`。

### 6.2 安全错误数据：safeErrorsData

`ApiError.Data` 不会直接暴露 rawData，而是通过 `safeErrorsData()` 和 `resolveSafeErrorItem()` 转换：

- `validation.Errors`、`map[string]validation.Error`、`map[string]SafeErrorItem` 等会递归转换为 `{code, message, params?}` 结构
- 普通 `error` 返回空对象 `{}`
- `map[string]string` 直接返回

这样确保响应中只包含安全、可面向用户的错误信息。

### 6.3 ToApiError：错误归一化

[tools/router/error.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/tools/router/error.go#L133-L147)：

```go
func ToApiError(err error) *ApiError {
    var apiErr *ApiError
    if !errors.As(err, &apiErr) {
        if errors.Is(err, sql.ErrNoRows) || errors.Is(err, fs.ErrNotExist) {
            apiErr = NewNotFoundError("", err)
        } else {
            apiErr = NewBadRequestError("", err)
        }
    }
    return apiErr
}
```

非 ApiError 会被自动归一化：`sql.ErrNoRows`/`fs.ErrNotExist` → 404，其他 → 400。

### 6.4 ErrorHandler：统一错误写回

[tools/router/router.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/tools/router/router.go#L160-L183)：

```go
func ErrorHandler(resp http.ResponseWriter, req *http.Request, err error) {
    if err == nil {
        return
    }
    if ok, _ := getWritten(resp); ok {
        return // 响应已写，不再处理
    }

    header := resp.Header()
    if header.Get("Content-Type") == "" {
        header.Set("Content-Type", "application/json")
    }

    apiErr := ToApiError(err)
    resp.WriteHeader(apiErr.Status)

    if req.Method != http.MethodHead {
        json.NewEncoder(resp).Encode(apiErr)
    }
}
```

关键点：
- 如果 Response 已经写过（`Written() == true`），直接返回，避免重复写
- 默认 Content-Type 为 `application/json`
- 将 error 归一化为 ApiError 后 JSON 编码写入
- HEAD 请求不写 Body

### 6.5 firstApiError：冒泡首个 ApiError

[apis/record_helpers.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/record_helpers.go#L536-L562)：

```go
func firstApiError(errs ...error) *router.ApiError {
    for _, err := range errs {
        if err == nil { continue }
        if apiErr, ok := err.(*router.ApiError); ok {
            return apiErr
        }
        if errors.As(err, &apiErr) {
            return apiErr
        }
    }
    return router.NewInternalServerError("", errors.Join(errs...))
}
```

通常用于将底层错误（如校验错误、DB 错误）和上层包装的 ApiError 一起传入，优先返回已有的 ApiError，避免重复包装。

### 6.6 Panic 恢复

[apis/middlewares.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/middlewares.go#L252-L283) 的 `panicRecover()` 中间件捕获内部 panic（除 `http.ErrAbortHandler`），转为 500 ApiError，附带堆栈信息。

### 6.7 活动日志中的错误

`activityLogger` 在 `e.Next()` 返回后，会解析错误并记录到日志：ApiError 的 Message 和 RawData 会分别作为 `error` 和 `details` 字段输出。

---

## 7. 完整请求流程示例

以 `POST /api/collections/users/records` 创建记录为例：

```
1. http.Server 接收请求，交给 ServeMux
2. 匹配 pattern: "POST /api/collections/{collection}/records"
3. 包装 ResponseWriter → ResponseWriter{written, status}
4. 包装 Request.Body → RereadableReadCloser（支持重复读取）
5. EventFactory 创建 RequestEvent{App, Response, Request}
6. Hook.Trigger 启动中间件链 + recordCreate Action：

   ┌─ activityLogger: 记录 __execStart → e.Next()
   │   ┌─ panicRecover: defer recover → e.Next()
   │   │   ┌─ loadAuthToken: 解析 Authorization，写入 e.Auth → e.Next()
   │   │   │   ┌─ securityHeaders: 设置 X-XSS-Protection 等 → e.Next()
   │   │   │   │   ┌─ superuserIPsWhitelist: 超级用户 IP 检查 → e.Next()
   │   │   │   │   │   ┌─ BodyLimit: 检查请求体大小 → e.Next()
   │   │   │   │   │   │   ┌─ dynamicCollectionBodyLimit: 集合级 body limit → e.Next()
   │   │   │   │   │   │   │   └─ recordCreate Action (路由 handler)
   │   │   │   │   │   │   │       ├─ 查找 collection
   │   │   │   │   │   │   │       ├─ checkCollectionRateLimit
   │   │   │   │   │   │   │       ├─ 解析 RequestInfo
   │   │   │   │   │   │   │       ├─ 检查 CreateRule 权限
   │   │   │   │   │   │   │       ├─ 创建 RecordRequestEvent
   │   │   │   │   │   │   │       ├─ OnRecordCreateRequest Hook 链（含用户自定义扩展）
   │   │   │   │   │   │   │       ├─ form.Submit() 保存记录
   │   │   │   │   │   │   │       └─ e.JSON(200, record) 写响应
   │   │   │   │   │   │   └─ dynamicCollectionBodyLimit After（无操作）
   │   │   │   │   │   └─ BodyLimit After（无操作）
   │   │   │   │   └─ superuserIPsWhitelist After（无操作）
   │   │   │   └─ securityHeaders After（无操作）
   │   │   └─ loadAuthToken After（无操作）
   │   └─ panicRecover: 若 panic 捕获 → 返回 500
   └─ activityLogger: logRequest(e, err) 记录访问日志

7. 若无错误，响应已在 Action 中通过 e.JSON 写入
8. 若有错误，ErrorHandler 归一化后写入 JSON 错误响应
```

---

## 8. 关键文件索引

| 文件 | 作用 |
|------|------|
| [tools/router/router.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/tools/router/router.go) | Router 定义、BuildMux、ErrorHandler、ResponseWriter |
| [tools/router/group.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/tools/router/group.go) | RouterGroup 分组、中间件绑定/解绑、路由注册 |
| [tools/router/route.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/tools/router/route.go) | Route 定义、路由级中间件 |
| [tools/router/event.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/tools/router/event.go) | Event 基础结构、Store、响应写入方法、错误构造 |
| [tools/router/error.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/tools/router/error.go) | ApiError、ToApiError、安全错误转换 |
| [tools/hook/hook.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/tools/hook/hook.go) | Hook、Handler、Trigger 链构建 |
| [tools/hook/event.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/tools/hook/event.go) | Resolver 接口、Next 机制 |
| [apis/serve.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/serve.go) | 服务器启动、BuildMux 调用时机 |
| [apis/base.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/base.go) | NewRouter 初始化、EventFactory、全局中间件注册 |
| [apis/middlewares.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/middlewares.go) | 所有内置中间件（鉴权、日志、panic、安全头等） |
| [core/event_request.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/core/event_request.go) | RequestEvent、RequestInfo |
| [core/events.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/core/events.go) | 所有应用级事件类型 |
| [core/record_query.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/core/record_query.go) | FindAuthRecordByToken Token 验证 |
| [core/record_tokens.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/core/record_tokens.go) | Token 生成 |
| [tools/security/jwt.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/tools/security/jwt.go) | JWT 解析与生成 |
| [apis/record_crud.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/record_crud.go) | 记录 CRUD handler、API 规则应用 |
| [apis/record_helpers.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/record_helpers.go) | firstApiError 等辅助函数 |
