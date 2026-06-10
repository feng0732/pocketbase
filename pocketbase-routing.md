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

### 3.4 默认全局中间件执行顺序（精确 Priority 值）

`Hook.Bind()` 注册中间件时使用 `sort.SliceStable` 按 `Priority` 升序排序（值越小越先执行）。所有中间件的 `Priority` 基准值为 `DefaultRateLimitMiddlewarePriority = -1000`（见 [apis/middlewares_rate_limit.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/middlewares_rate_limit.go#L16)）。

完整执行顺序（洋葱 Before 阶段，从外到内）：

| Priority | 中间件 | ID | 定义位置 | 作用 |
|----------|--------|----|----------|------|
| **-99999** | wwwRedirect | pbWWWRedirect | [middlewares.go:227-250](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/middlewares.go#L227-L250) | www → non-www 域名重定向 |
| **-1041** | CORS | pbCors | [middlewares_cors.go:27-28](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/middlewares_cors.go#L27-L28) | 跨域响应头（在 activityLogger 之前使 OPTIONS 预检不计入日志和限流） |
| **-1040** | activityLogger | pbActivityLogger | [middlewares.go:34](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/middlewares.go#L34) `=-1000-40` | 请求日志（最外层包装，记录完整执行时间） |
| **-1030** | panicRecover | pbPanicRecover | [middlewares.go:39](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/middlewares.go#L39) `=-1000-30` | panic 捕获恢复（在 activityLogger 内部，使 panic 也能被记录） |
| **-1020** | loadAuthToken | pbLoadAuthToken | [middlewares.go:42](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/middlewares.go#L42) `=-1000-20` | 解析 Authorization 头，写入 `e.Auth`（**鉴权加载，不强制报错**） |
| **-1015** | superuserIPsWhitelist | pbSuperuserIPsWhitelist | [middlewares.go:45](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/middlewares.go#L45) `=-1020+5` | 超级用户 IP 白名单检查（**必须在 loadAuthToken 之后**，依赖 `e.Auth` 是否为超级用户） |
| **-1010** | securityHeaders | pbSecurityHeaders | [middlewares.go:48](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/middlewares.go#L48) `=-1000-10` | 设置 X-XSS-Protection、X-Content-Type-Options、X-Frame-Options |
| **-1000** | rateLimit | pbRateLimit | [middlewares_rate_limit.go:16](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/middlewares_rate_limit.go#L16) | 全局限速（在 securityHeaders 之后执行，**即使被限流也带有安全响应头**；同时依赖 `e.Auth` 区分 guest/auth 受众） |
| **-990** | BodyLimit | pbBodyLimit | [middlewares_body_limit.go:18](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/middlewares_body_limit.go#L18) `=-1000+10` | 请求体大小限制（最内层全局中间件，超限直接返回 413） |

#### 洋葱模型完整调用链图示

```
wwwRedirect.Before ─┐
   CORS.Before ─────┤
 activityLogger.Before ─┤  (记录 __execStart 时间戳)
  panicRecover.Before ─┤  (defer recover 就位)
loadAuthToken.Before ──┤  (解析 Token → e.Auth)
superuserIPsWhitelist.Before  (检查 e.Auth 是否超管 + IP 白名单)
securityHeaders.Before ─┤  (写安全响应头)
  rateLimit.Before ────┤  (检查速率)
   BodyLimit.Before ───┤  (包装 limitedReader)
          │
        Route Action (业务 handler: e.JSON/e.NoContent/返回 error)
          │
   BodyLimit.After ────┘  (无额外逻辑，error 原样向上冒泡)
  rateLimit.After ────┘  (无额外逻辑)
securityHeaders.After ──┘  (无额外逻辑)
superuserIPsWhitelist.After  (无额外逻辑)
loadAuthToken.After ───┘  (无额外逻辑)
  panicRecover.After ───┤  (若捕获 panic → 替换为 500 ApiError 向上冒泡)
 activityLogger.After ─┤  (logRequest(e, err) 写日志 → 原样 return err)
   CORS.After ─────────┤  (无额外逻辑)
wwwRedirect.After ─────┘
          │
          ▼
   Hook.Trigger() 最终 return err
          │
          ▼
   ErrorHandler(resp, req, err) 写 JSON 错误响应
```

**关键设计细节：**
- `activityLogger`（-1040）比 `panicRecover`（-1030）更早执行 Before，意味着日志包裹了 panic 恢复——即使内部 panic，也能被记录
- `loadAuthToken`（-1020）必须在 `superuserIPsWhitelist`（-1015）之前，因为后者需要读取 `e.Auth` 判断是否为超级用户
- `rateLimit`（-1000）在 `securityHeaders`（-1010）之后，确保被 429 限流的响应也带有安全头；同时 rateLimit 内部会根据 `e.Auth` 是否为 nil 选择 guest/auth 受众规则
- `BodyLimit`（-990）是最内层全局中间件，因为它需要在解析 body 之前生效

### 3.5 中间件解绑与排除

RouterGroup 和 Route 都支持 `Unbind(middlewareIds...)`，解绑时会：
1. 从自身 Middlewares 列表移除
2. 记录到 `excludedMiddlewares` map，父级同名中间件在链构建时也会被跳过

示例（[apis/record_crud.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/record_crud.go#L28)）：

```go
subGroup := rg.Group("/collections/{collection}/records").Unbind(DefaultRateLimitMiddlewareId)
```

### 3.6 两套限流机制：collectionPathRateLimit vs checkCollectionRateLimit

PocketBase 有两种限流使用方式，分别对应不同场景：

| 维度 | collectionPathRateLimit | checkCollectionRateLimit |
|------|-------------------------|-------------------------|
| **类型** | 中间件（`*hook.Handler`），可通过 `.Bind()` 绑定到路由 | 普通函数，直接在 handler 内部调用 |
| **所在文件** | [apis/middlewares_rate_limit.go:54-75](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/middlewares_rate_limit.go#L54-L75) | [apis/middlewares_rate_limit.go:81-108](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/middlewares_rate_limit.go#L81-L108) |
| **collection 参数来源** | 从 URL path 中通过 `e.Request.PathValue(collectionPathParam)` 动态解析 | 由调用方作为参数直接传入已解析的 `*core.Collection` |
| **返回值** | 返回 `e.Next()` 或 NotFoundError（解析 collection 失败） | 返回 `error`（nil 或 429 TooManyRequestsError） |
| **是否调用 e.Next()** | 是（作为中间件必须调用） | 否（纯函数检查） |
| **使用场景** | 认证类路由（不会被 Batch 复用） | CRUD 路由（会被 Batch API 复用） |

**调用关系：collectionPathRateLimit 内部调用 checkCollectionRateLimit**

```go
// collectionPathRateLimit（中间件）
func collectionPathRateLimit(collectionPathParam string, baseTags ...string) *hook.Handler[*core.RequestEvent] {
    return &hook.Handler[*core.RequestEvent]{
        Id:       DefaultRateLimitMiddlewareId,
        Priority: DefaultRateLimitMiddlewarePriority,
        Func: func(e *core.RequestEvent) error {
            // 第1步：从 path 解析 collection
            collection, err := e.App.FindCachedCollectionByNameOrId(
                e.Request.PathValue(collectionPathParam),
            )
            if err != nil {
                return e.NotFoundError("Missing or invalid collection context.", err)
            }

            // 第2步：委托给 checkCollectionRateLimit 执行真正的限流检查
            if err := checkCollectionRateLimit(e, collection, baseTags...); err != nil {
                return err
            }

            return e.Next()  // 第3步：中间件必须调用 Next
        },
    }
}
```

**为什么 CRUD 路由不使用中间件形式？**

[apis/record_crud.go:26-27](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/record_crud.go#L26-L27) 注释说明：
> "the rate limiter is inlined because some of the crud actions are also used in the batch APIs"

因为 CRUD handler 会被 [Batch API](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/batch.go#L37-L71) 直接复用（如 `recordCreate(false, next)`、`recordUpdate(false, next)`），如果限流是路由中间件：
- 直接 HTTP 请求：中间件会执行，正常限流 ✓
- Batch 内部调用：不经过 HTTP 路由和中间件链，无法限流 ✗

所以 CRUD 必须采用 **handler 内联调用** 的方式，确保无论直接请求还是 Batch 复用都能触发限流。

### 3.7 CRUD 路由中间件链的实际绑定代码

[apis/record_crud.go:25-34](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/record_crud.go#L25-L34)：

```go
func bindRecordCrudApi(app core.App, rg *router.RouterGroup[*core.RequestEvent]) {
    // 第1步：创建子分组，解绑全局限速中间件（因为要改成内联调用）
    subGroup := rg.Group("/collections/{collection}/records").
        Unbind(DefaultRateLimitMiddlewareId)

    // 第2步：注册路由（每条路由可独立绑定自己的中间件）
    subGroup.GET("", recordsList)                                              // GET /api/collections/{collection}/records
    subGroup.GET("/{id}", recordView)                                          // GET /api/collections/{collection}/records/{id}
    subGroup.POST("", recordCreate(true, nil)).Bind(dynamicCollectionBodyLimit(""))   // POST + 集合级 body limit
    subGroup.PATCH("/{id}", recordUpdate(true, nil)).Bind(dynamicCollectionBodyLimit("")) // PATCH + 集合级 body limit
    subGroup.DELETE("/{id}", recordDelete(true, nil))                          // DELETE
}
```

对比认证类路由的绑定方式（[apis/record_auth.go:22-66](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/record_auth.go#L22-L66)）：

```go
// 认证路由使用中间件形式的限流（因为不会被 Batch 复用）
sub.POST("/auth-with-password", recordAuthWithPassword).Bind(
    collectionPathRateLimit("", "authWithPassword", "auth"),  // 中间件限流
)
```

**CRUD POST 创建路由的完整中间件链（从外到内）**：

| 层级 | 来源 | Priority | 中间件/处理 |
|------|------|----------|------------|
| 全局 | pbRouter.Bind() | -99999 | wwwRedirect |
| 全局 | pbRouter.Bind() | -1041 | CORS |
| 全局 | pbRouter.Bind() | -1040 | activityLogger |
| 全局 | pbRouter.Bind() | -1030 | panicRecover |
| 全局 | pbRouter.Bind() | -1020 | loadAuthToken |
| 全局 | pbRouter.Bind() | -1015 | superuserIPsWhitelist |
| 全局 | pbRouter.Bind() | -1010 | securityHeaders |
| ~~全局~~ | ~~pbRouter.Bind()~~ | ~~-1000~~ | ~~rateLimit~~ **（被 Unbind 解绑）** |
| 全局 | pbRouter.Bind() | -990 | BodyLimit（默认 32MB） |
| 路由级 | subGroup.POST(...).Bind(...) | -990 | dynamicCollectionBodyLimit（按文件字段累加） |
| — | — | — | **recordCreate Action**（handler 内联 checkCollectionRateLimit） |

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

### 6.0 错误返回完整路径（从 Handler 到 ErrorHandler）

以路由 Action 返回一个 `e.UnauthorizedError(...)` 为例，错误从产生到写入 HTTP 响应要经过以下层层传递：

```
阶段 A：业务层产生错误
───────────────────────────────────────────────────────────────────
recordCreate Action
  │  if err := form.Submit(); err != nil {
  │      return firstApiError(err, e.BadRequestError("Failed to create record", err))
  │  }
  ▼
return *router.ApiError  ← 错误对象诞生

阶段 B：Hook 链逐层冒泡（洋葱 After 阶段，从内到外）
───────────────────────────────────────────────────────────────────
BodyLimit.After           err := e.Next()  →  return err   (原样透传)
rateLimit.After           err := e.Next()  →  return err   (原样透传)
securityHeaders.After     err := e.Next()  →  return err   (原样透传)
superuserIPsWhitelist.After  err := e.Next() → return err  (原样透传)
loadAuthToken.After       err := e.Next()  →  return err   (原样透传)
panicRecover.After
  │  defer recover() { ... }  ← 如果是 panic 在此被捕获转为 ApiError
  │  err := e.Next()
  ▼  return err   (panic 已被替换为 500 ApiError，普通 err 原样透传)
activityLogger.After
  │  err := e.Next()
  │  logRequest(e, err)  ← 写日志（不吞错误）
  ▼  return err          ← 继续向上冒泡
CORS.After                err := e.Next()  →  return err   (原样透传)
wwwRedirect.After         err := e.Next()  →  return err   (原样透传)

阶段 C：Hook.Trigger 返回给 Router
───────────────────────────────────────────────────────────────────
Hook.Trigger(event, v.Action)
  │  // 最外层的 event.Next() 执行完毕
  ▼  return err   ← Hook 链的最终返回值

阶段 D：BuildMux 注册的 Handler 捕获
───────────────────────────────────────────────────────────────────
[tools/router/router.go:130-151]
  │  event, cleanupFunc := r.eventFactory(resp, req)
  │  err := routeHook.Trigger(event, v.Action)
  │  if err != nil {
  ▼      ErrorHandler(resp, req, err)   ← 统一错误入口
  │  }
  │  if cleanupFunc != nil { cleanupFunc() }

阶段 E：ErrorHandler 写回 HTTP 响应
───────────────────────────────────────────────────────────────────
[tools/router/router.go:160-183]
  1. err == nil? → return（无事发生）
  2. resp.Written() == true? → return（业务已写过响应，不再覆盖）
  3. Content-Type 为空 → 设置为 application/json
  4. apiErr := ToApiError(err)   ← 归一化
  5. resp.WriteHeader(apiErr.Status)   ← 写状态码
  6. 非 HEAD 请求 → json.NewEncoder(resp).Encode(apiErr)   ← 写 JSON Body
```

**关键点：**
- 绝大多数中间件在 After 阶段不修改 error，直接 `return err` 原样冒泡
- 只有 `panicRecover` 会修改 error（把 panic 转为 500 ApiError）
- `activityLogger` 会读 error 写日志，但不吞掉
- `ErrorHandler` 有"响应已写则跳过"的守卫，避免业务已写 `e.JSON(200, ...)` 后又被错误覆盖

---

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

## 7. CRUD 请求在中间件、处理器、错误处理器间的实际处理过程

以 `POST /api/collections/users/records` 创建记录请求为例，完整走读每个阶段：

### 7.1 阶段一：HTTP 路由层（BuildMux 注册的 handler）

[tools/router/router.go:130-151](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/tools/router/router.go#L130-L151)

```
HTTP 请求到达
   │
   ├─ 包装 ResponseWriter → ResponseWriter{written:false, status:0}
   ├─ 包装 Request.Body → RereadableReadCloser（支持重复读取）
   ├─ EventFactory 创建 RequestEvent{App, Response, Request, Auth:nil}
   │
   ├─ 构建 routeHook：
   │   ├─ 父分组中间件（按 Priority 排序，排除已 Unbind 的）
   │   ├─ 当前路由中间件（dynamicCollectionBodyLimit）
   │   └─ Action = recordCreate(true, nil)
   │
   ├─ err := routeHook.Trigger(event, v.Action)  ← 进入中间件链
   │
   └─ if err != nil { ErrorHandler(resp, req, err) }  ← 错误处理入口
```

### 7.2 阶段二：全局中间件链（Hook 洋葱 Before 阶段，从外到内）

```
wwwRedirect (-99999)
  ├─ 检查是否需要 www 前缀重定向
  └─ e.Next()
     │
CORS (-1041)
  ├─ 处理 OPTIONS 预检请求（若是直接写响应 return nil）
  ├─ 设置 Access-Control-* 响应头
  └─ e.Next()
     │
activityLogger (-1040)
  ├─ e.Set(__execStart, time.Now())  ← 记录起始时间
  └─ err := e.Next()
     │     └─ 后续所有中间件+Action 都在此函数内执行
     │
panicRecover (-1030)
  ├─ defer func() { if r := recover(); ... { err = 500 ApiError } }()
  └─ e.Next()
     │
loadAuthToken (-1020)
  ├─ 从 Authorization 头提取 Token
  ├─ 调用 app.FindAuthRecordByToken() 解析
  ├─ 成功则 e.Auth = record，失败不报错（Auth 保持 nil）
  └─ e.Next()
     │
superuserIPsWhitelist (-1015)
  ├─ 若 e.Auth != nil 且是超管集合，检查 e.RealIP() 是否在 Settings.SuperuserIPs 白名单
  ├─ 不在白名单 → return e.ForbiddenError(...)
  └─ e.Next()
     │
securityHeaders (-1010)
  ├─ 写 X-Content-Type-Options、X-XSS-Protection、X-Frame-Options
  └─ e.Next()
     │
【全局 rateLimit (-1000) 已被 Unbind，跳过】
     │
BodyLimit (-990)
  ├─ 检查 Content-Length > DefaultMaxBodySize(32MB)
  ├─ 超限 → return ErrRequestEntityTooLarge (413)
  ├─ 包装 e.Request.Body = limitedReader（流式读取时也会检查）
  └─ e.Next()
     │
```

### 7.3 阶段三：路由级中间件 + 处理器（recordCreate Action）

```
dynamicCollectionBodyLimit (-990)  ← 路由级 .Bind()
  ├─ 从 path 解析 collection
  ├─ 遍历 collection.Fields，累加 File 等字段的 MaxBodySize
  ├─ 再次 applyBodyLimit（可能比 32MB 更大）
  └─ e.Next()
     │
recordCreate Action (handler)  ← [apis/record_crud.go:209-390]
  │
  ├─ Step 1: 解析 collection
  │     e.App.FindCachedCollectionByNameOrId(pathValue("collection"))
  │     失败 → return e.NotFoundError(...)
  │     是 View 集合 → return e.BadRequestError(...)
  │
  ├─ Step 2: 限流检查（内联函数，不是中间件！）
  │     err = checkCollectionRateLimit(e, collection, "create")
  │     超限 → return e.TooManyRequestsError(...)  ← 429 直接冒泡
  │
  ├─ Step 3: 解析 RequestInfo（Query/Headers/Body/Auth/Method/Context）
  │     requestInfo, err := e.RequestInfo()
  │     err → return firstApiError(err, e.BadRequestError("", err))
  │
  ├─ Step 4: API 规则级鉴权
  │     !hasSuperuserAuth && collection.CreateRule == nil
  │       → return e.ForbiddenError("Only superusers...")
  │
  ├─ Step 5: 构造 Record 和 Form，加载数据
  │     record := core.NewRecord(collection)
  │     data, err := recordDataFromRequest(e, record)
  │     form := forms.NewRecordUpsert(app, record)
  │     form.Load(data)
  │
  ├─ Step 6: 验证 CreateRule（非超管且规则非空）
  │     构造 dummyRecord + WITH 子查询，将规则解析为 SQL WHERE
  │     查询不存在 → return e.BadRequestError("create rule failure")
  │
  ├─ Step 7: 触发 OnRecordCreateRequest Hook 链（用户可扩展）
  │     event := new(core.RecordRequestEvent)
  │     event.RequestEvent = e
  │     hookErr := app.OnRecordCreateRequest().Trigger(event, func(e *core.RecordRequestEvent) error {
  │         // Hook 的最内层 Action
  │         ├─ form.Submit()  ← 数据库事务内保存记录
  │         │     失败 → return firstApiError(err, e.BadRequestError(...))
  │         ├─ EnrichRecord()  ← 计算 expand 等
  │         ├─ execAfterSuccessTx(responseWriteAfterTx, ...)
  │         │     └─ e.JSON(200, record)  ← 写响应，written=true
  │         └─ optFinalizer（Batch API 场景使用）
  │     })
  │     hookErr != nil → return hookErr  ← Hook 链中产生的错误冒泡
  │
  └─ Step 8: return nil  ← 正常结束
```

### 7.4 阶段四：Hook 洋葱 After 阶段（错误冒泡，从内到外）

假设 Step 2 触发了限流，返回了 429 `*ApiError`：

```
recordCreate Action
  └─ return TooManyRequestsError (429 *ApiError)
        │
dynamicCollectionBodyLimit.After
  └─ err := e.Next() → return err  ← 原样透传
        │
BodyLimit.After
  └─ err := e.Next() → return err  ← 原样透传
        │
securityHeaders.After
  └─ err := e.Next() → return err  ← 原样透传
        │
superuserIPsWhitelist.After
  └─ err := e.Next() → return err  ← 原样透传
        │
loadAuthToken.After
  └─ err := e.Next() → return err  ← 原样透传
        │
panicRecover.After
  └─ err := e.Next() → return err  ← 非 panic，原样透传
        │
activityLogger.After
  ├─ err := e.Next()  ← 收到 429 ApiError
  ├─ logRequest(e, err)
  │     ├─ status = 429（从 ApiError.Status 读取，因为 resp.Written 还是 false）
  │     ├─ attrs = [..., "error", message, "details", rawData]
  │     └─ app.Logger().Error("POST /api/collections/...", attrs...)
  └─ return err  ← 不吞错误，继续向上冒泡
        │
CORS.After
  └─ err := e.Next() → return err  ← 原样透传
        │
wwwRedirect.After
  └─ err := e.Next() → return err  ← 原样透传
        │
Hook.Trigger()
  └─ return err  ← Hook 链最终返回 429 ApiError
```

### 7.5 阶段五：ErrorHandler 写回响应

[tools/router/router.go:160-183](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/tools/router/router.go#L160-L183)

```
BuildMux handler 收到 err
   │
   └─ ErrorHandler(resp, req, err):
        │
        ├─ 1. err != nil ✓
        ├─ 2. resp.Written() = false  ✓（handler 没写过响应）
        ├─ 3. Content-Type 为空 → 设置 "application/json"
        ├─ 4. apiErr := ToApiError(err)  → 已是 *ApiError，直接返回
        ├─ 5. resp.WriteHeader(429)  → 标记 written=true, status=429
        └─ 6. json.Encode({
                "status": 429,
                "message": "",
                "data": {}
             })
```

### 7.6 正常成功路径与错误路径的分支对比

| 节点 | 成功路径（CreateRule 通过） | 错误路径（限流触发） |
|------|---------------------------|---------------------|
| recordCreate Step 2 checkCollectionRateLimit | return nil，继续 | return 429 ApiError，**直接退出** |
| 后续 Step 3-8 | 全部执行，最终 e.JSON(200, record) | **不执行** |
| resp.Written() | true（Step 7 e.JSON 写入） | false（handler 未写响应） |
| activityLogger 日志级别 | Info | Error |
| ErrorHandler 执行 | **跳过**（Written=true） | 执行，写 429 JSON |
| HTTP 响应状态码 | 200（e.JSON 设置） | 429（ErrorHandler 设置） |

### 7.7 限流 429 错误返回的精确内容核准

以 `POST /api/collections/users/records` 触发限流为例，从错误产生到最终响应和日志的每一步代码追踪：

#### 第一步：checkRateLimit 产生错误

[apis/middlewares_rate_limit.go:184-186](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/middlewares_rate_limit.go#L184-L186)

```go
if !rt.isAllowed(key) {
    return e.TooManyRequestsError("", errors.New("triggered rate limit rule: "+rule.String()))
}
```

传入参数：
- `message = ""`（空字符串）
- `rawErrData = errors.New("triggered rate limit rule: " + rule.String())`

其中 `rule.String()` 是 RateLimitRule 的 JSON 序列化（[core/settings_model.go:766-773](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/core/settings_model.go#L766-L773)），典型值如：
```
{"label":"users:create","audience":"","duration":60,"maxRequests":60}
```

#### 第二步：NewTooManyRequestsError → NewApiError 构造 ApiError

[tools/router/error.go:111-131](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/tools/router/error.go#L111-L131)

```go
func NewTooManyRequestsError(message string, rawErrData any) *ApiError {
    if message == "" {
        message = "Too Many Requests."   // 空 message 用默认值
    }
    return NewApiError(http.StatusTooManyRequests, message, rawErrData)
}

func NewApiError(status int, message string, rawErrData any) *ApiError {
    if message == "" {
        message = http.StatusText(status)
    }
    return &ApiError{
        rawData: rawErrData,                                    // 保存原始 error，仅用于日志
        Data:    safeErrorsData(rawErrData),                    // 公开的 data 字段（安全转换）
        Status:  status,                                        // 429
        Message: strings.TrimSpace(inflector.Sentenize(message)), // "Too Many Requests."（已有句号，Sentenize 原样返回）
    }
}
```

`safeErrorsData(rawErrData)` 对 `errors.New(...)` 的处理（[tools/router/error.go:151-160](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/tools/router/error.go#L151-L160)）：
```go
case error:
    validationErrors := validation.Errors{}
    if errors.As(v, &validationErrors) {
        return resolveSafeErrorsData(validationErrors)
    }
    return map[string]any{}  // ← 普通 error 返回空对象（不暴露内部细节）
```

**构造完成的 ApiError 内存结构**：
```go
&ApiError{
    rawData: errors.New("triggered rate limit rule: {\"label\":\"users:create\",\"audience\":\"\",\"duration\":60,\"maxRequests\":60}"),
    Data:    map[string]any{},    // 空对象！不向客户端暴露内部错误信息
    Status:  429,
    Message: "Too Many Requests.",
}
```

#### 第三步：ErrorHandler 写回 HTTP 响应

[tools/router/router.go:160-183](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/tools/router/router.go#L160-L183)

```go
func ErrorHandler(resp http.ResponseWriter, req *http.Request, err error) {
    // ...
    apiErr := ToApiError(err)          // 已是 ApiError，原样返回
    resp.WriteHeader(apiErr.Status)    // 写 HTTP 429
    if req.Method != http.MethodHead {
        json.NewEncoder(resp).Encode(apiErr)  // JSON 编码 ApiError 的导出字段
    }
}
```

**最终 HTTP 响应（完整原始字节）**：

```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Date: Wed, 10 Jun 2026 10:00:00 GMT
Content-Length: 59

{"status":429,"message":"Too Many Requests.","data":{}}
```

关键点：
- `data` 字段是空对象 `{}`，不向客户端暴露 `triggered rate limit rule: ...` 的内部信息
- `message` 是 Sentenize 处理后的 `"Too Many Requests."`（首字母大写 + 句号结尾）
- 安全响应头（X-Content-Type-Options 等）已由 securityHeaders 中间件在 Before 阶段写入

#### 第四步：activityLogger 记录的日志字段

[apis/middlewares.go:394-418](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/middlewares.go#L394-L418)

```go
if err != nil {
    apiErr, isPlainApiError := err.(*router.ApiError)
    if isPlainApiError || errors.As(err, &apiErr) {
        if status == 0 {                          // Written=false，status 还没写
            status = apiErr.Status                 // 从 ApiError.Status 读取 = 429
        }
        var errMsg string
        if isPlainApiError {                        // 是直接的 *ApiError（非包装）
            errMsg = apiErr.Message                  // = "Too Many Requests."
        } else { ... }
        attrs = append(attrs,
            slog.String("error", errMsg),           // "error": "Too Many Requests."
            slog.Any("details", apiErr.RawData()),   // "details": 原始 error.Error() 字符串
        )
    }
}
```

**最终写入的日志（slog Error 级别）**：

```
level=ERROR
msg="POST /api/collections/users/records"
type=request
execTime=1.234
url=/api/collections/users/records
method=POST
status=429
error="Too Many Requests."
details="triggered rate limit rule: {\"label\":\"users:create\",\"audience\":\"\",\"duration\":60,\"maxRequests\":60}"
referer=""
userAgent="curl/8.0.0"
auth=""
userIP="192.168.1.100"
remoteIP="192.168.1.100"
```

关键点：
- 日志的 `error` 字段是公开的 Message（与响应一致）
- 日志的 `details` 字段包含内部的 `triggered rate limit rule: ...`，用于排查触发了哪条规则
- 这个 `details` **不会**出现在 HTTP 响应中，只在服务端日志里

---

## 8. 完整请求流程示例

以 `POST /api/collections/users/records` 创建记录为例。注意 `/collections/{collection}/records` 分组通过 `Unbind(DefaultRateLimitMiddlewareId)` 解绑了全局 `rateLimit` 中间件，改在每个 handler **内部内联调用 `checkCollectionRateLimit`**（因为 CRUD handler 会被 Batch API 复用，无法走中间件链）。

```
1. http.Server 接收请求，交给 ServeMux
2. 匹配 pattern: "POST /api/collections/{collection}/records"
3. 包装 ResponseWriter → ResponseWriter{written, status}
4. 包装 Request.Body → RereadableReadCloser（支持重复读取）
5. EventFactory 创建 RequestEvent{App, Response, Request}
6. Hook.Trigger 启动中间件链 + recordCreate Action（按 Priority 从小到大执行 Before）：

   ┌─ wwwRedirect (-99999): 检查 www 前缀 → e.Next()
   │   ┌─ CORS (-1041): 设置跨域头 → e.Next()
   │   │   ┌─ activityLogger (-1040): Set(__execStart, time.Now()) → e.Next()
   │   │   │   ┌─ panicRecover (-1030): defer recover() 就位 → e.Next()
   │   │   │   │   ┌─ loadAuthToken (-1020): 解析 Authorization 头 → e.Auth → e.Next()
   │   │   │   │   │   ┌─ superuserIPsWhitelist (-1015): 若 e.Auth 是超管则检查 IP → e.Next()
   │   │   │   │   │   │   ┌─ securityHeaders (-1010): 写 X-XSS-Protection 等头 → e.Next()
   │   │   │   │   │   │   │   ┌─ 【全局 rateLimit(-1000) 已被 Unbind，跳过】
   │   │   │   │   │   │   │   │   ┌─ BodyLimit (-990): 检查 Content-Length，包装 limitedReader → e.Next()
   │   │   │   │   │   │   │   │   │   ┌─ dynamicCollectionBodyLimit (-990): 集合字段累加 body 上限 → e.Next()
   │   │   │   │   │   │   │   │   │   │   └─ recordCreate Action (路由 handler)
   │   │   │   │   │   │   │   │   │   │       ├─ 查找 collection
   │   │   │   │   │   │   │   │   │   │       ├─ ✅ 内联 checkCollectionRateLimit (限流在此，非中间件)
   │   │   │   │   │   │   │   │   │   │       ├─ 解析 RequestInfo
   │   │   │   │   │   │   │   │   │   │       ├─ 检查 CreateRule 权限（API 规则级鉴权）
   │   │   │   │   │   │   │   │   │   │       ├─ 创建 RecordRequestEvent
   │   │   │   │   │   │   │   │   │   │       ├─ OnRecordCreateRequest Hook 链（含用户自定义扩展）
   │   │   │   │   │   │   │   │   │   │       ├─ form.Submit() 保存记录
   │   │   │   │   │   │   │   │   │   │       ├─ 成功: e.JSON(200, record) 写响应 → Written=true
   │   │   │   │   │   │   │   │   │   │       └─ 失败: return firstApiError(err, e.BadRequestError(...))
   │   │   │   │   │   │   │   │   │   └─ dynamicCollectionBodyLimit After: return err（原样透传）
   │   │   │   │   │   │   │   │   └─ BodyLimit After: return err（原样透传）
   │   │   │   │   │   │   │   └─ securityHeaders After: return err（原样透传）
   │   │   │   │   │   │   └─ superuserIPsWhitelist After: return err（原样透传）
   │   │   │   │   │   └─ loadAuthToken After: return err（原样透传）
   │   │   │   │   └─ panicRecover After: 若捕获 panic → 转为 500 ApiError；否则 return err
   │   │   │   └─ activityLogger After: logRequest(e, err) 写访问日志 → return err
   │   │   └─ CORS After: return err（原样透传）
   │   └─ wwwRedirect After: return err（原样透传）

7. Hook.Trigger() 返回最终 err
8. BuildMux handler 判断：
   - err == nil 且 Written==true → 正常结束（Action 已通过 e.JSON 写响应）
   - err != nil 且 Written==false → 调用 ErrorHandler(resp, req, err) 写 JSON 错误
   - err != nil 但 Written==true → ErrorHandler 直接跳过（业务已写）
```

#### 错误场景完整走读（CreateRule 校验失败）

```
recordCreate Action
  └─ return e.ForbiddenError("CreateRule not satisfied", ruleErr)
        │
        ▼
  Hook 链 After 阶段逐层冒泡（所有中间件原样 return err）
        │
        ▼
  activityLogger.After:
    ├─ logRequest(e, err) → 从 ApiError 中提取 Message/RawData 写入 slog
    └─ return err
        │
        ▼
  Hook.Trigger() return err
        │
        ▼
  BuildMux handler:
    └─ if err != nil { ErrorHandler(resp, req, err) }
          │
          ▼
     ErrorHandler:
       1. getWritten(resp) → false（业务未写响应）
       2. Content-Type = "application/json"
       3. apiErr := ToApiError(err) → 已是 ApiError，直接返回
       4. resp.WriteHeader(403)
       5. json.Encode({status:403, message:"...", data:{...}})
```

---

## 9. 关键文件索引

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
| [apis/middlewares.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/middlewares.go) | 内置中间件（鉴权、日志、panic、安全头、www 重定向等） |
| [apis/middlewares_rate_limit.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/middlewares_rate_limit.go) | rateLimit / collectionPathRateLimit / checkCollectionRateLimit 三套限流 |
| [apis/middlewares_body_limit.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/middlewares_body_limit.go) | BodyLimit / dynamicCollectionBodyLimit 请求体大小限制 |
| [apis/middlewares_cors.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/middlewares_cors.go) | CORS 跨域中间件 |
| [apis/record_crud.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/record_crud.go) | bindRecordCrudApi 路由绑定、recordsList/recordView/recordCreate/recordUpdate/recordDelete handler |
| [apis/record_auth.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/record_auth.go) | bindRecordAuthApi 路由绑定、认证类 handler、使用 collectionPathRateLimit 中间件限流 |
| [apis/record_helpers.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/record_helpers.go) | firstApiError 等辅助函数 |
| [apis/batch.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/apis/batch.go) | Batch API，直接复用 CRUD handler（因此 CRUD 限流必须内联而非中间件） |
| [core/event_request.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/core/event_request.go) | RequestEvent、RequestInfo |
| [core/events.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/core/events.go) | 所有应用级事件类型（RecordRequestEvent 等） |
| [core/record_query.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/core/record_query.go) | FindAuthRecordByToken Token 验证 |
| [core/record_tokens.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/core/record_tokens.go) | Token 生成 |
| [tools/security/jwt.go](file:///d:/fz/0601/solo-dogfeeding/code/154-pocketbase/tools/security/jwt.go) | JWT 解析与生成 |
