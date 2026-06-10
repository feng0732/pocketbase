# PocketBase SSE 实时订阅实现脉络

## 整体架构概览

PocketBase 的实时订阅系统基于 **SSE（Server-Sent Events）** 协议，采用「Broker-Client」模式，核心组件分布在三个层级：

| 层级 | 职责 | 关键文件 |
|------|------|----------|
| API 层 | HTTP 连接管理、订阅请求处理、事件广播 | [realtime.go](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go) |
| 订阅中间层 | Client/Broker 抽象、消息通道、订阅存储 | [broker.go](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/tools/subscriptions/broker.go)、[client.go](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/tools/subscriptions/client.go)、[message.go](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/tools/subscriptions/message.go) |
| 应用核心层 | Broker 生命周期、事件 Hook、数据模型事件 | [base.go](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/core/base.go)、[events.go](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/core/events.go) |

---

## 一、订阅注册流程

订阅注册分为 **两个独立的 HTTP 请求**：先建立 SSE 长连接获得 `clientId`，再通过 POST 请求绑定具体订阅主题。

### 1.1 SSE 连接建立（GET /api/realtime）

入口函数为 [realtimeConnect](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L43-L166)，执行步骤如下：

**步骤 1：准备 SSE 响应头**

```go
e.Response.Header().Set("Content-Type", "text/event-stream")
e.Response.Header().Set("Cache-Control", "no-store")
e.Response.Header().Set("X-Accel-Buffering", "no")  // 禁用 Nginx 缓冲
```

同时通过 `http.NewResponseController` 禁用写入超时，防止长连接被服务器主动切断。

**步骤 2：创建 Client 实例并注册到 Broker**

```go
connectEvent.Client = subscriptions.NewDefaultClient()  // 生成 40 字符随机 ID
ce.App.SubscriptionsBroker().Register(ce.Client)        // 存入 Broker 的 store
defer func() {
    e.App.SubscriptionsBroker().Unregister(ce.Client.Id())  // 函数退出时自动注销
}()
```

Broker 在 [base.go](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/core/base.go#L204) 中通过 `subscriptions.NewBroker()` 初始化，内部使用线程安全的 `store.Store[string, Client]` 维护所有连接客户端。

**步骤 3：发送 PB_CONNECT 握手消息**

服务端立即向客户端推送一条特殊事件，告知其 `clientId`：

```go
Message{
    Name: "PB_CONNECT",
    Data: []byte(`{"clientId":"` + ce.Client.Id() + `"}`),
}
```

消息格式由 [Message.WriteSSE](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/tools/subscriptions/message.go#L20-L37) 序列化为 SSE 标准格式：

```
id:abc123...
event:PB_CONNECT
data:{"clientId":"abc123..."}

```

**步骤 4：进入消息循环**

主循环通过 `select` 监听四个通道：

| 触发源 | 行为 |
|--------|------|
| `maxTimer.C`（默认 30 分钟） | 调用 `cancelRequest()` 终止连接 |
| `idleTimer.C`（默认 5 分钟无消息） | 调用 `cancelRequest()` 终止连接 |
| `ce.Client.Channel()` | 读取消息并写入 SSE 响应，重置 idleTimer |
| `ce.Request.Context().Done()` | 客户端断开，正常退出 |

### 1.2 订阅主题绑定（POST /api/realtime）

入口函数为 [realtimeSetSubscriptions](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L184-L249)。

**请求体结构：**

```go
type realtimeSubscribeForm struct {
    ClientId      string   `json:"clientId"`
    Subscriptions []string `json:"subscriptions"`
}
```

**安全校验（钩子触发前执行）：**

1. **Client 存在性校验**：通过 `Broker.ClientById(form.ClientId)` 查找
2. **IP 一致性校验**：防止 clientId 暴力破解，比对 `RealtimeClientIPKey` 存储的 IP
3. **Auth 升级校验**：只允许 guest→auth 的升级，已认证用户不可切换身份

**核心逻辑（在钩子链的末尾执行）：**

```go
e.Client.Set(RealtimeClientAuthKey, e.Auth)   // 更新认证状态
e.Client.Unsubscribe()                         // 先清空所有旧订阅（整体替换策略）
e.Client.Subscribe(e.Subscriptions...)         // 添加新订阅
```

### 1.3 Client 端订阅存储

[DefaultClient.Subscribe](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/tools/subscriptions/client.go#L154-L196) 方法的关键逻辑：

- 订阅主题支持携带 `options` 查询参数，格式如：
  ```
  collection/RECORD_ID?options={"query":{"filter":"status = true"},"headers":{"x-token":"abc"}}
  ```
- 解析后存储为 `map[string]SubscriptionOptions`，其中：
  - `Query` 保存 `filter`、`expand`、`fields` 等参数
  - `Headers` 的 key 会被转换为 snake_case（如 `X-Token` → `x_token`）

---

## 二、钩子触发与执行顺序

PocketBase 的 Hook 系统采用 **洋葱模型**（Onion Model），实现位于 [hook.go](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/tools/hook/hook.go#L146-L174)。`Trigger()` 方法从后向前构建 `nextFunc` 链，实际执行时 **优先级值越小越先执行**（升序排列）。每个 Handler 必须调用 `e.Next()` 才能进入下一层，否则会中断链。

### 2.1 连接请求（GET /api/realtime）的钩子链

路由定义在 [bindRealtimeApi](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L34-L41)：

```go
sub.GET("", realtimeConnect).Bind(SkipSuccessActivityLog())
```

**完整执行顺序（从外到内）：**

| 优先级 | 钩子/中间件 | ID | 说明 | 位置 |
|--------|------------|----|------|------|
| -1040 | `activityLogger()` | pbActivityLogger | 记录请求耗时、状态码等审计日志（成功时被跳过） | [middlewares.go:349](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/middlewares.go#L349) |
| -1030 | `panicRecover()` | pbPanicRecover | defer recover() 捕获 panic，转为 500 错误 | [middlewares.go:253](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/middlewares.go#L253) |
| -1020 | `loadAuthToken()` | pbLoadAuthToken | 从 Header/Cookie 解析 auth token，加载 `e.Auth` | [middlewares.go:42](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/middlewares.go#L42) |
| -1015 | `superuserIPsWhitelist()` | pbSuperuserIPsWhitelist | 超级用户 IP 白名单校验 | [middlewares.go:45](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/middlewares.go#L45) |
| -1010 | `securityHeaders()` | pbSecurityHeaders | 注入 CSP、X-Frame-Options 等安全头 | [middlewares.go:48](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/middlewares.go#L48) |
| -990 | `BodyLimit()` | — | 请求体大小限制（默认 20MB，GET 实际无 body） | [base.go:36](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/base.go#L36) |
| 0（默认） | `SkipSuccessActivityLog()` | pbSkipSuccessActivityLog | 设置 `__skipSuccessActivityLogger = true`，避免 SSE 长连接产生大量成功日志 | [middlewares.go:327](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/middlewares.go#L327) |
| — | **`realtimeConnect` handler** | — | 用户注册的 `OnRealtimeConnectRequest` 钩子 → 内嵌 handler（Broker.Register + 消息循环） | [realtime.go:43](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L43) |

**在 `realtimeConnect` 内部，还会嵌套触发以下应用级钩子：**

```
OnRealtimeConnectRequest
    ↓ e.Next()
    用户注册的自定义 Handler（可修改 IdleTimeout/MaxTimeout/Client）
    ↓ e.Next()
    内嵌 Action Handler：
      ├─ Broker.Register(client)
      ├─ defer Broker.Unregister(client.Id)
      ├─ 触发 OnRealtimeMessageSend（发送 PB_CONNECT）
      └─ for-select 消息循环（每条消息又触发 OnRealtimeMessageSend）
```

**应用级钩子定义位置：**
- `OnRealtimeConnectRequest()` — [base.go:1023](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/core/base.go#L1023)，事件结构体见 [events.go:445-469](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/core/events.go#L445-L469)
- `OnRealtimeMessageSend()` — [base.go:1027](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/core/base.go#L1027)，事件结构体见 [events.go:471-477](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/core/events.go#L471-L477)

> **重要注意**：`OnRealtimeConnectRequest` 中 `e.Next()` 之后的代码会在 **连接断开后** 才执行（因为内嵌 handler 是阻塞的消息循环）。开发者可利用这一点做连接级别的资源清理。

### 2.2 订阅请求（POST /api/realtime）的钩子链

路由定义：

```go
sub.POST("", realtimeSetSubscriptions)   // 没有 SkipSuccessActivityLog
```

**完整执行顺序（从外到内）：**

| 优先级 | 钩子/中间件 | ID | 说明 |
|--------|------------|----|------|
| -1040 | `activityLogger()` | pbActivityLogger | 记录请求（POST 不会被跳过，会产生审计日志） |
| -1030 | `panicRecover()` | pbPanicRecover | panic 恢复 |
| -1020 | `loadAuthToken()` | pbLoadAuthToken | 解析认证 token → `e.Auth` |
| -1015 | `superuserIPsWhitelist()` | pbSuperuserIPsWhitelist | 超级用户 IP 校验 |
| -1010 | `securityHeaders()` | pbSecurityHeaders | 安全响应头 |
| -990 | `BodyLimit()` | — | 请求体大小限制 |
| — | **`realtimeSetSubscriptions` handler** | — | 前置校验 → `OnRealtimeSubscribeRequest` 钩子链 → 内嵌 Action Handler |

**`realtimeSetSubscriptions` 内部执行流程：**

```
realtimeSetSubscriptions()
  │
  ├─ 1. BindBody & validate()（clientId 必填，subscriptions 最多 1000 条）
  │     ↓ 失败返回 400
  ├─ 2. Broker.ClientById(clientId)
  │     ↓ 失败返回 404
  ├─ 3. IP 一致性校验（clientIP vs e.RealIP()）
  │     ↓ 失败返回 400
  ├─ 4. Auth 升级校验（只允许 guest→auth）
  │     ↓ 失败返回 403
  │
  └─ 5. 构造 RealtimeSubscribeRequestEvent，触发 OnRealtimeSubscribeRequest：
          │
          ├─ 用户注册的自定义 Handler（可修改 Subscriptions 列表）
          │    ↓ e.Next()
          └─ 内嵌 Action Handler：
                ├─ client.Set(RealtimeClientAuthKey, e.Auth)
                ├─ client.Unsubscribe()         // 清空所有旧订阅
                ├─ client.Subscribe(subs...)    // 写入新订阅
                └─ execAfterSuccessTx() → 204 No Content
```

**应用级钩子定义位置：**
- `OnRealtimeSubscribeRequest()` — [base.go:1031](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/core/base.go#L1031)，事件结构体见 [events.go:479-485](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/core/events.go#L479-L485)

**关于 `execAfterSuccessTx`**：定义在 [record_helpers.go:564-583](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/record_helpers.go#L564-L583)，当处于事务中时，通过 `txInfo.OnComplete()` 延迟到事务提交成功后才返回 HTTP 响应，确保事务回滚时不会误导客户端认为订阅已生效。

---

## 三、连接请求钩子提前返回错误的清理边界分析

### 3.1 defer 清理链的分层结构

[realtimeConnect](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L43-L166) 函数中的 defer 不是同一个作用域注册的，而是分布在 **两个不同层级**。理解这一点是分析清理边界的关键：

```go
func realtimeConnect(e *core.RequestEvent) error {
    // ========== 层级 0：钩子触发之前 ==========
    writeDeadlineErr := rc.SetWriteDeadline(time.Time{})
    if writeDeadlineErr != nil {
        if !errors.Is(writeDeadlineErr, http.ErrNotSupported) {
            return e.InternalServerError(...)  // ← 极早期返回，无任何 defer 注册
        }
    }

    cancelCtx, cancelRequest := context.WithCancel(e.Request.Context())
    defer cancelRequest()                          // DEFER-A: 最外层作用域，最后执行
    e.Request = e.Request.Clone(cancelCtx)

    // ... 设置响应头、创建 Client 对象、设置 IP ...

    return e.App.OnRealtimeConnectRequest().Trigger(connectEvent,
        func(ce *core.RealtimeConnectRequestEvent) error {  // ← 内嵌 Action Handler
            // ========== 层级 1：钩子链最末端的内嵌 Action ==========
            ce.App.SubscriptionsBroker().Register(ce.Client)
            defer func() {
                e.App.SubscriptionsBroker().Unregister(ce.Client.Id())  // DEFER-B
            }()

            // ... 发送 PB_CONNECT ...

            maxTimer := time.NewTimer(ce.MaxTimeout)
            defer maxTimer.Stop()         // DEFER-C

            idleTimer := time.NewTimer(ce.IdleTimeout)
            defer idleTimer.Stop()        // DEFER-D

            // ... for-select 消息循环 ...
        })
}
```

**defer 执行顺序（LIFO 后进先出）**：

| defer | 注册作用域 | 执行顺序（函数返回时） | 作用 |
|-------|-----------|----------------------|------|
| DEFER-D `idleTimer.Stop()` | 内嵌 Action | 第 1 个执行 | 释放空闲定时器 |
| DEFER-C `maxTimer.Stop()` | 内嵌 Action | 第 2 个执行 | 释放最大生命周期定时器 |
| DEFER-B `Broker.Unregister()` | 内嵌 Action | 第 3 个执行 | 注销 Client + Discard channel |
| DEFER-A `cancelRequest()` | 外层函数 | 第 4 个执行（最后） | 取消请求上下文 |

> **核心差异**：DEFER-B/C/D 只有在 **内嵌 Action 被实际执行** 时才会被注册。如果用户钩子在 `e.Next()` 之前返回错误，这三个 defer 永远不会执行；而 DEFER-A 无论如何都会执行。

### 3.2 Hook 错误传播机制

PocketBase 的 Hook 采用洋葱模型，错误会沿着 Handler 调用链 **原路返回**。核心逻辑在 [hook.go:164-173](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/tools/hook/hook.go#L164-L173)：

```go
for i := len(handlers) - 1; i >= 0; i-- {
    old := event.nextFunc()
    event.setNextFunc(func() error {
        event.setNextFunc(old)
        return handlers[i](event)   // ← 每个 handler 的 return 值就是上一层 e.Next() 的返回值
    })
}
return event.Next()  // ← 整个 Trigger 的返回值就是最外层 handler 的返回值
```

如果某个 handler 不调用 `e.Next()` 就直接返回 error，那么其下游所有 handler（包括内嵌 Action）都不会执行，error 直接向上冒泡。

### 3.3 OnRealtimeConnectRequest 钩子的三种报错场景

#### 场景 A：e.Next() 之前返回错误（拒绝建立连接）

```go
app.OnRealtimeConnectRequest().BindFunc(func(e *core.RealtimeConnectRequestEvent) error {
    if someCondition {
        return e.ForbiddenError("connection not allowed", nil)  // ← 未调用 e.Next()
    }
    return e.Next()
})
```

**执行路径与清理状态**：

| 资源 | 状态 | 原因 |
|------|------|------|
| `cancelRequest()` | ✅ 执行 | DEFER-A 在外层作用域，一定执行 |
| `Broker.Register()` | ❌ 未执行 | 内嵌 Action 未被调用 |
| `Broker.Unregister()` | ❌ 未注册 | 内嵌 Action 未执行，DEFER-B 不存在 |
| Client 对象 | ✅ 已创建但未注册 | [realtime.go:71](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L71) 在钩子触发前创建，仅由 GC 回收 |
| Client.channel | ❌ 未被外部引用 | 未注册到 Broker，无 goroutine 会向其 Send |
| `maxTimer` / `idleTimer` | ❌ 未创建 | 内嵌 Action 未执行 |
| 响应状态码 | 403 Forbidden | error 原样向上传播 |
| activityLogger | ✅ **会记录错误日志** | `err != nil`，不受 SkipSuccessActivityLog 影响 |
| PB_CONNECT 消息 | ❌ 未发送 | 内嵌 Action 未执行 |

**资源泄漏风险：无**。Client 对象未注册到 Broker，channel 无写入者，最终由 GC 回收。

#### 场景 B：e.Next() 之后返回错误（连接建立后报错）

```go
app.OnRealtimeConnectRequest().BindFunc(func(e *core.RealtimeConnectRequestEvent) error {
    err := e.Next()  // ← 调用后内嵌 Action 已完整执行（或因断连退出）
    if err != nil {
        return err
    }
    // 连接已断开后执行的后置逻辑
    if somePostCheck {
        return errors.New("post-connection check failed")  // ← 连接断开后才返回错误
    }
    return nil
})
```

**执行路径与清理状态**：

| 资源 | 状态 | 原因 |
|------|------|------|
| `cancelRequest()` | ✅ 执行 | DEFER-A |
| `Broker.Register()` | ✅ 已执行 | 内嵌 Action 已调用 |
| `Broker.Unregister()` | ✅ 已执行 | DEFER-B 在函数返回时触发 |
| `client.Discard()` | ✅ 已调用 | Unregister 内部调用 |
| `maxTimer.Stop()` | ✅ 已执行 | DEFER-C |
| `idleTimer.Stop()` | ✅ 已执行 | DEFER-D |
| 响应状态码 | 取决于断开前是否已写入 | 若已写入 200 头则无法改变，未写入则按 error 设置 |
| activityLogger | ✅ **会记录错误日志** | `err != nil` |
| PB_CONNECT 消息 | 取决于断开时机 | 若断连发生在握手之前则未发送，否则已发送 |

> **重要注意**：`e.Next()` 返回后，连接实际上已经断开（因为内嵌 Action 是阻塞的消息循环）。此时返回 error 仅影响 **日志记录** 和 **未写入的响应状态码**，对连接本身和 Client 清理无影响。开发者可在此做连接级别的后置审计或指标上报。

#### 场景 C：SetWriteDeadline 极早期失败（钩子触发前）

位于 [realtime.go:47-49](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L47-L49)：

```go
writeDeadlineErr := rc.SetWriteDeadline(time.Time{})
if writeDeadlineErr != nil {
    if !errors.Is(writeDeadlineErr, http.ErrNotSupported) {
        return e.InternalServerError("Failed to initialize SSE connection.", writeDeadlineErr)
    }
}
```

这是唯一 **连 DEFER-A 都不会执行** 的返回路径（因为 `defer cancelRequest()` 注册在第 58 行，在这段检查之后）。不过此时尚未创建任何需要清理的资源（无 context、无 Client、无 Timer、无 Broker 注册），因此无泄漏风险。`http.ErrNotSupported` 会被降级为 Warn 日志而不返回错误。

### 3.4 内嵌 Action 内部不同阶段的失败差异

内嵌 Action 内部所有失败路径最终都 **返回 nil**（而非 error），这是 PocketBase 的设计选择：SSE 长连接的"断开"被视为正常情况而非异常。但不同失败点的清理范围不同：

| 失败点 | 代码位置 | 已注册的 defer | 清理结果 | 对客户端的可见性 |
|--------|----------|---------------|----------|-----------------|
| PB_CONNECT 写入/Flush 失败 | [realtime.go:100-106](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L100-L106) | DEFER-A + DEFER-B | Broker.Unregister + cancelRequest | 客户端只看到 TCP 断连，未收到 clientId |
| 定时器创建后消息循环中断 | 理论上不会失败 | DEFER-A + B + C + D | 全部清理 | 客户端已收到 clientId |
| 事件消息写入失败 | [realtime.go:145-151](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L145-L151) | DEFER-A + B + C + D | 全部清理 | 客户端已收到之前的消息 |
| maxTimer / idleTimer 超时 | [realtime.go:120-123](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L120-L123) | DEFER-A + B + C + D | 全部清理 | 正常强制断开 |
| Context 取消（客户端主动断开） | [realtime.go:156-162](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L156-L162) | DEFER-A + B + C + D | 全部清理 | 客户端主动关闭 |
| Channel 被外部 Discard 关闭 | [realtime.go:124-131](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L124-L131) | DEFER-A + B + C + D | 全部清理（Broker.Unregister 幂等） | 客户端看到断连 |

### 3.5 OnRealtimeMessageSend 钩子报错的两层差异

`OnRealtimeMessageSend` 在握手和每条事件发送时都会触发。钩子报错的影响取决于发生在 `e.Next()` 的哪一侧：

```go
connectMsgErr := ce.App.OnRealtimeMessageSend().Trigger(connectMsgEvent,
    func(me *core.RealtimeMessageEvent) error {
        err := me.Message.WriteSSE(me.Response, me.Client.Id())  // 实际写入
        if err != nil { return err }
        return me.Flush()                                        // 实际推送
    })
```

#### 子场景 1：e.Next() 之前返回错误（阻止消息发送）

```go
app.OnRealtimeMessageSend().BindFunc(func(e *core.RealtimeMessageEvent) error {
    if !allowed(e.Message.Name) {
        return errors.New("blocked")  // ← 未调用 e.Next()，WriteSSE 不执行
    }
    return e.Next()
})
```

- **WriteSSE/Flush**：不执行，消息未写入 Response
- **外层行为**：`connectMsgErr != nil` → 记录 Debug 日志 → `return nil` → 连接正常断开，defer 链正常执行
- **影响**：客户端收不到该条消息，连接被关闭（握手场景）或当前消息被跳过并关闭连接（事件场景）

#### 子场景 2：e.Next() 之后返回错误（消息已发送后报错）

```go
app.OnRealtimeMessageSend().BindFunc(func(e *core.RealtimeMessageEvent) error {
    err := e.Next()  // ← WriteSSE + Flush 已完成，字节已推送到客户端
    auditLog(e.Message)  // 后置审计
    return errors.New("post-check failed")
})
```

- **WriteSSE/Flush**：已执行，消息已到达客户端 TCP 缓冲区
- **外层行为**：`msgErr != nil` → 记录 Debug 日志 → `return nil` → 连接关闭
- **影响**：客户端实际上收到了消息，但服务端认为"发送失败"并关闭连接。这是 **最终一致性不一致窗口**：客户端可能基于已收到的消息做了操作，但服务端已断开连接。开发者应谨慎在此位置返回 error，若只需审计应返回 nil。

### 3.6 订阅请求钩子的清理边界对比

订阅请求 `OnRealtimeSubscribeRequest` 的资源模型与连接请求不同：所有订阅修改都在 **内存中** 操作（Client 的订阅 map 和认证状态 store），无外部资源需要 defer 释放。

#### 订阅场景 A：e.Next() 之前返回错误

```go
app.OnRealtimeSubscribeRequest().BindFunc(func(e *core.RealtimeSubscribeRequestEvent) error {
    if !validate(e.Subscriptions) {
        return e.BadRequestError("invalid subscription", nil)  // ← 未调用 e.Next()
    }
    return e.Next()
})
```

| 操作 | 是否执行 |
|------|----------|
| `client.Set(auth)` | ❌ 不执行，认证状态不更新 |
| `client.Unsubscribe()` | ❌ 不执行，旧订阅保留 |
| `client.Subscribe(subs)` | ❌ 不执行，新订阅不写入 |
| HTTP 响应 | 400 Bad Request |
| activityLogger | ✅ 记录错误日志 |

#### 订阅场景 B：e.Next() 之后返回错误

```go
app.OnRealtimeSubscribeRequest().BindFunc(func(e *core.RealtimeSubscribeRequestEvent) error {
    err := e.Next()  // ← 认证状态已更新、订阅已整体替换
    if someExternalCheck {
        return errors.New("external check failed")
    }
    return err
})
```

| 操作 | 是否执行 |
|------|----------|
| `client.Set(auth)` | ✅ 已执行 |
| `client.Unsubscribe()` | ✅ 已执行，旧订阅已清空 |
| `client.Subscribe(subs)` | ✅ 已执行，新订阅已写入 |
| HTTP 响应 | 取决于 `execAfterSuccessTx`：非事务模式下 error 直接返回 500；事务模式下 OnComplete 中才会写响应 |
| activityLogger | ✅ 记录错误日志 |

> **关键风险**：订阅修改是 **非事务性的内存操作**。即使外层处于数据库事务中，`client.Subscribe()` 的结果也不会随事务回滚而撤销。若在 `e.Next()` 之后返回 error，客户端会收到 500 错误认为订阅失败，但服务端内存中订阅 **实际上已经生效**，后续事件仍会推送到该客户端。开发者应避免在此位置返回会误导客户端的 error。

---

## 四、握手写入失败与事件写入失败的退出清理

### 4.1 核心清理机制概览

SSE 连接的所有退出路径最终都会走到同一个 **defer 清理链**，该链在 [realtimeConnect](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L57-L81) 中建立：

```go
cancelCtx, cancelRequest := context.WithCancel(e.Request.Context())
defer cancelRequest()                 // 第 1 层：取消请求上下文
e.Request = e.Request.Clone(cancelCtx)

return e.App.OnRealtimeConnectRequest().Trigger(connectEvent, func(ce *RealtimeConnectRequestEvent) error {
    ce.App.SubscriptionsBroker().Register(ce.Client)
    defer func() {
        e.App.SubscriptionsBroker().Unregister(ce.Client.Id())  // 第 2 层：Broker 注销
    }()
    // ... 消息循环 ...
})
```

两层 defer 的执行顺序为 **LIFO**（后进先出）：函数返回时先执行 `Broker.Unregister()`，再执行 `cancelRequest()`。

`Broker.Unregister()` 的内部实现见 [broker.go:58-65](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/tools/subscriptions/broker.go#L58-L65)：

```go
func (b *Broker) Unregister(clientId string) {
    client := b.store.Get(clientId)
    if client == nil {
        return
    }
    client.Discard()          // ① 关闭 channel，标记 isDiscarded=true
    b.store.Remove(clientId)  // ② 从全局注册表移除
}
```

### 4.2 握手（PB_CONNECT）写入失败的清理流程

握手消息写入位于 [realtime.go:86-107](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L86-L107)，包含两步操作：
1. `Message.WriteSSE(me.Response, me.Client.Id())` — 将 SSE 格式的字节写入 Response buffer
2. `me.Flush()` — 调用 `http.NewResponseController(e.Response).Flush()` 将 buffer 强制推送到客户端

**失败场景：** `WriteSSE` 或 `Flush` 任一返回 error（常见原因：客户端已断开连接、网络中断、ResponseWriter 被 hijack）。

**清理执行路径：**

```
OnRealtimeMessageSend Trigger 中 WriteSSE/Flush 返回错误
        ↓
connectMsgErr != nil，记录 Debug 日志：
  "Realtime connection closed (failed to deliver PB_CONNECT)"
        ↓
return nil （注意：不返回 error 给上层，避免触发 activityLogger 的错误日志）
        ↓
OnRealtimeConnectRequest 的 Trigger 结束，内嵌 handler 返回
        ↓
函数 realtimeConnect 开始执行 defer 链（LIFO 顺序）：
  ├─ ① defer Broker.Unregister(clientId)
  │     ├─ client.Discard() → close(client.channel)，isDiscarded=true
  │     └─ store.Remove(clientId) → 从 Broker 注册表删除
  └─ ② defer cancelRequest() → 取消请求上下文
        ↓
外层中间件继续 unwind：
  └─ activityLogger：因为返回 err=nil 且 skipSuccessActivityLog=true，不记录日志
```

**设计要点：**
- 握手失败时 `return nil` 而非 error，是有意为之：SSE 长连接的"断开"不应视为异常，避免污染错误日志
- `IsDiscarded()` 检查会阻止后续所有 `client.Send()` 调用（此时定时器尚未创建，尚无 pending 消息）

### 4.3 事件消息写入失败的清理流程

消息循环中的事件写入位于 [realtime.go:134-152](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L134-L152)，结构与握手完全一致，但处于 for-select 循环内。

**失败触发点：**
- `msg, ok := <-ce.Client.Channel()`：读取到消息后尝试写入 SSE
- `WriteSSE` / `Flush` 返回错误（客户端断连、TCP RST、连接超时等）

**清理执行路径：**

```
for-select 从 client.channel 读取到消息
        ↓
构造 RealtimeMessageEvent，触发 OnRealtimeMessageSend
        ↓
内嵌 handler 中 WriteSSE/Flush 出错，msgErr != nil
        ↓
记录 Debug 日志：
  "Realtime connection closed (failed to deliver message)"
        ↓
return nil → 退出 for-select 循环，退出内嵌 handler
        ↓
执行 defer 链：
  ├─ ① defer idleTimer.Stop()
  ├─ ② defer maxTimer.Stop()
  ├─ ③ defer Broker.Unregister(clientId)
  │     ├─ client.Discard() → close(channel)
  │     └─ store.Remove(clientId)
  └─ ④ defer cancelRequest()
```

**与握手失败的区别：**
- 多了两个定时器 defer（`idleTimer.Stop()` 和 `maxTimer.Stop()`），释放定时器资源避免内存泄漏
- 此时 `client.channel` 中可能还有未读消息，但 `Discard()` 关闭 channel 后，任何试图 `Send()` 的 goroutine 都会被 `recover()` 捕获（见 [client.go:274-285](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/tools/subscriptions/client.go#L274-L285)）

### 4.4 其他退出路径的清理对比

| 退出原因 | 触发位置 | return 值 | 清理行为 |
|----------|----------|-----------|----------|
| 握手写入失败 | [realtime.go:100-107](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L100-L107) | `nil` | defer 链正常执行 |
| 事件写入失败 | [realtime.go:145-152](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L145-L152) | `nil` | defer 链正常执行，定时器 Stop |
| maxTimer 超时（30min） | [realtime.go:120-121](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L120-L121) | 通过 `cancelRequest()` 间接触发 | cancelRequest → Context.Done() → 进入下一行 case |
| idleTimer 超时（5min） | [realtime.go:122-123](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L122-L123) | 同上 | 同上 |
| Channel 被外部关闭 | [realtime.go:124-132](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L124-L132) | `nil` | 日志标记 "closed channel"，defer 链执行 |
| 客户端主动断开 | [realtime.go:156-163](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L156-L163) | `nil` | 日志标记 "cancelled request"，defer 链执行 |
| OnRealtimeConnectRequest 钩子返回 error | 用户自定义 Handler | 非 nil error | defer 链仍执行，但 activityLogger 会记录错误日志 |

### 4.5 `DefaultClient.Send()` 的发送容错

位于 [client.go:274-285](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/tools/subscriptions/client.go#L274-L285)：

```go
func (c *DefaultClient) Send(m Message) {
    if c.IsDiscarded() {
        return                    // 第 1 道防线：已丢弃则直接返回
    }
    defer func() {
        recover()                 // 第 2 道防线：recover 兜底 channel 关闭竞态
    }()
    c.channel <- m                // 阻塞写入（无缓冲 channel）
}
```

**为何需要 recover？** 存在一个时间窗口：`IsDiscarded()` 检查返回 `false` 后，另一个 goroutine 可能立即调用 `Discard()` 关闭 channel，此时写入会 panic。`recover()` 确保 `Send()` 永远不会因竞态而崩溃。

---

## 五、事件过滤机制

事件广播触发点在 [bindRealtimeEvents](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L330-L527) 中绑定的多个数据模型 Hook 上，核心分发函数是 [realtimeBroadcastRecord](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L594-L773)。

### 5.1 事件触发源

| Hook | 动作 | 说明 |
|------|------|------|
| `OnModelAfterCreateSuccess` | create | 记录创建成功后广播 |
| `OnModelAfterUpdateSuccess` | update | 记录更新成功后广播 |
| `OnModelDelete` + `OnModelAfterDeleteSuccess` | delete | 采用「预缓存 + 事务后广播」两段式，避免事务回滚造成误通知 |

### 5.2 订阅主题匹配

广播时构建 6 种可能的订阅前缀进行匹配：

```go
subscriptionRuleMap := map[string]*string{
    (collection.Name + "/" + record.Id + "?"): collection.ViewRule,  // 单条记录（按集合名）
    (collection.Id   + "/" + record.Id + "?"): collection.ViewRule,  // 单条记录（按集合ID）
    (collection.Name + "/*?"):                 collection.ListRule,  // 通配符（按集合名）
    (collection.Id   + "/*?"):                 collection.ListRule,  // 通配符（按集合ID）
    (collection.Name + "?"):                   collection.ListRule,  // 向后兼容（旧格式）
    (collection.Id   + "?"):                   collection.ListRule,  // 向后兼容（旧格式）
}
```

匹配逻辑在 [DefaultClient.Subscriptions(prefixes...)](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/tools/subscriptions/client.go#L121-L149) 中：

```go
if strings.HasPrefix(s+"?", prefix) {
    result[s] = options
}
```

给每个订阅主题末尾追加 `?` 再匹配前缀，确保 `posts/abc` 不会错误匹配 `posts/abcd`。

### 5.3 两层权限过滤

**第一层：API 规则过滤（服务端强制）**

由 [realtimeCanAccessRecord](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L856-L902) 调用 `app.CanAccessRecord()` 执行：
- 单条记录订阅 → 校验集合的 `ViewRule`
- 通配符订阅 → 校验集合的 `ListRule`
- 规则为空字符串 → 禁止所有访问
- 规则为 `nil` → 公开访问（仅超级用户集合默认如此）

**第二层：客户端 filter 过滤（可选）**

如果订阅主题的 options 中带有 `filter` 查询参数，会额外执行数据库级别的过滤：

```go
filter := requestInfo.Query[search.FilterQueryParam]
if filter != "" {
    resolver := core.NewRecordFieldResolver(app, record.Collection(), requestInfo, false)
    expr, err := search.FilterData(filter).BuildExpr(resolver)
    q.AndWhere(expr)
    err = q.Limit(1).Row(&exists)
    return err == nil && exists > 0
}
```

这意味着即使事件已广播，若记录不满足客户端指定的 filter 表达式，该客户端也不会收到通知。

### 5.4 数据裁剪与增强

通过权限检查后，消息数据还会经过以下处理：

| 处理步骤 | 代码位置 | 说明 |
|----------|----------|------|
| Record 副本净化 | [realtime.go:656](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L656) | `record.Fresh()` 创建无 expand、无 unknown 字段的副本 |
| 超级用户字段显示 | [realtime.go:668-670](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L668-L670) | 超级用户订阅者可看到 hidden 字段 |
| expand 关系展开 | [realtime.go:675-688](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L675-L688) | 读取 options 中 `expand` 参数展开关联记录 |
| 邮箱可见性控制 | [realtime.go:692-697](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L692-L697) | 本人、超级用户、管理者可看到 auth 记录的 email |
| fields 字段筛选 | [realtime.go:718-733](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L718-L733) | 通过 `picker.Pick()` 按 `fields` 参数裁剪输出 |

最终消息结构：

```json
{
    "action": "create|update|delete",
    "record": { "...": "..." }
}
```

---

## 六、连接断开后的处理

### 6.1 正常断开的清理路径

连接断开会触发以下清理链条：

```
客户端关闭 / 超时 / Context 取消 / 写入失败
        ↓
realtimeConnect() 内嵌 handler return nil
        ↓
defer 链逆序执行（LIFO）：
  ① idleTimer.Stop()
  ② maxTimer.Stop()
  ③ Broker.Unregister(ce.Client.Id())  [realtime.go:79-81]
       ├─ client.Discard()  →  close(channel) + 标记 isDiscarded=true
       └─ store.Remove(clientId)  →  从 Broker 注册表删除
  ④ cancelRequest()  [realtime.go:58]
        ↓
外层中间件 unwind
```

[DefaultClient.Discard](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/tools/subscriptions/client.go#L252-L263) 是幂等的，多次调用不会 panic。关闭 channel 后，主循环中的 `case msg, ok := <-ce.Client.Channel()` 会收到 `ok=false`，也会退出函数，形成 **双重保障**。

### 6.2 消息发送时的容错

参见上文 4.5 节。

### 6.3 认证状态联动清理

当用户相关数据发生变化时，系统会主动清理所有关联客户端的认证缓存，防止权限提升后旧连接滥用。相关函数位于 [realtime.go:251-L328](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L251-L328)：

| 触发场景 | 绑定 Hook | 处理函数 | 行为 |
|----------|----------|----------|------|
| Auth 记录更新（密码修改等） | `OnModelAfterUpdateSuccess`（Priority=-99） | `realtimeUpdateClientsAuth` | TokenKey 变化则清除 auth，否则更新为最新 record |
| Auth 记录删除 | `OnModelAfterDeleteSuccess`（Priority=-99） | `realtimeUnsetClientsAuthByRecordModelOrProxy` | 清除该用户所有客户端的 auth |
| Auth 集合密钥变化 | `OnCollectionUpdate`（Priority=-99） | `realtimeUnsetClientsAuthByCollection` | 清除该集合下所有客户端的 auth |
| Auth 集合删除 | `OnCollectionAfterDeleteSuccess`（Priority=-99） | `realtimeUnsetClientsAuthByCollection` | 清除该集合下所有客户端的 auth |

这些函数都采用 **分块并发遍历**（`ChunkedClients` + `errgroup.Group`），每块 150 个客户端（`clientsChunkSize`），避免客户端数量大时阻塞。

**注意**：清理后客户端不会立即断开，而是保持未认证状态；下次发送 `POST /api/realtime` 时可重新认证。

### 6.4 Delete 事件的事务安全

Delete 操作采用三段式处理以避免事务回滚造成的误通知：

1. **预缓存阶段**（`OnModelDelete`，Priority=99，尽可能靠后执行）：
   - 调用 `realtimeBroadcastRecord(..., dryCache=true, ...)`
   - 消息暂存在各 Client 的 store 中，key 为 `delete/tableName/pk`
   - 不立即发送给客户端

2. **正式广播阶段**（`OnModelAfterDeleteSuccess`，Priority=-99）：
   - 调用 `realtimeBroadcastDryCacheKey` 遍历所有 Client
   - 取出暂存的消息，通过 `routine.FireAndForget` 异步调用 `client.Send()`
   - 清除 Client 上的缓存 key

3. **失败回滚阶段**（`OnModelAfterDeleteError`，Priority=-99）：
   - 调用 `realtimeUnsetDryCacheKey` 清除所有 Client 上的暂存消息
   - 不产生任何通知

---

## 七、完整数据流时序图

```
  客户端                          中间件层                          应用钩子层                          Handler 内核
    |                               |                                |                                   |
    |-- GET /api/realtime -------->|                                |                                   |
    |                               | activityLogger (start timer)   |                                   |
    |                               | panicRecover                    |                                   |
    |                               | loadAuthToken                   |                                   |
    |                               | superuserIPsWhitelist           |                                   |
    |                               | securityHeaders                 |                                   |
    |                               | BodyLimit                       |                                   |
    |                               | SkipSuccessActivityLog          |                                   |
    |                               |                                |-- OnRealtimeConnectRequest ------->|
    |                               |                                |  (用户自定义 Handler)              |
    |                               |                                |                                   |-- Broker.Register(client)
    |                               |                                |                                   |-- defer Broker.Unregister + cancelRequest
    |                               |                                |                       ┌---------->|-- OnRealtimeMessageSend
    |                               |                                |                       |           |   WriteSSE("PB_CONNECT")
    |<-- event:PB_CONNECT ----------|--------------------------------|-----------------------┘           |   Flush()
    |    data:{"clientId":"abc"}    |                                |                                   |
    |                               |                                |                                   |-- maxTimer/idleTimer 启动
    |                               |                                |                                   |-- for-select 循环开始
    |                               |                                |                                   |
    |-- POST /api/realtime -------->|                                |                                   |
    |   {clientId, subscriptions}   | activityLogger                  |                                   |
    |                               | panicRecover                    |                                   |
    |                               | loadAuthToken                   |                                   |
    |                               | ...                             |                                   |
    |                               |-- BindBody + validate           |                                   |
    |                               |-- ClientById + IP校验 + Auth校验|                                   |
    |                               |                                |-- OnRealtimeSubscribeRequest ---->|
    |                               |                                |  (用户自定义 Handler)              |
    |                               |                                |                                   |-- client.Set(auth)
    |                               |                                |                                   |-- client.Unsubscribe()
    |                               |                                |                                   |-- client.Subscribe(subs)
    |<-- 204 No Content ------------|--------------------------------|-----------------------------------|
    |                               |                                |                                   |
    |          (记录被修改)         |                                |                                   |
    |                               |                                |  OnModelAfterUpdateSuccess         |
    |                               |                                |    ↓ realtimeBroadcastRecord       |
    |                               |                                |    分块遍历 Clients                 |
    |                               |                                |    前缀匹配 + 权限校验 + filter     |
    |                               |                                |    expand/fields 裁剪              |
    |                               |                                |    client.Send(Message)            |
    |                               |                                |        ↓                           |
    |                               |                                |    写入 client.channel             |
    |                               |                                |        ↓                           |
    |                               |                                |   for-select 读到消息              |
    |                               |                                |        ↓                           |
    |                               |                                |-- OnRealtimeMessageSend ---------->|
    |                               |                                |                                   | WriteSSE + Flush
    |<-- event:posts/RECORD_ID -----|--------------------------------|-----------------------------------|
    |    data:{"action":"update",...}|                               |                                   |
    |                               |                                |                                   |
    |   (连接关闭或超时)            |                                |                                   |
    |                               |                                |                                   | for-select 退出
    |                               |                                |                                   |   ↓
    |                               |                                |                                   | defer 链执行:
    |                               |                                |                                   |   timers.Stop() → Unregister → cancelRequest
    |                               | activityLogger (skip success)   |                                   |
```

---

## 八、关键常量与配置

| 常量 | 值 | 含义 | 位置 |
|------|----|------|------|
| `clientsChunkSize` | 150 | 广播时每批处理的客户端数 | [realtime.go:26](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L26) |
| `IdleTimeout` | 5 分钟 | 无消息时自动断开时间 | [realtime.go:69](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L69) |
| `MaxTimeout` | 30 分钟 | 连接最长存活时间（强制重连） | [realtime.go:70](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L70) |
| Client ID 长度 | 40 | `security.RandomString(40)` | [client.go:94](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/tools/subscriptions/client.go#L94) |
| 订阅数量上限 | 1000 | 单个客户端最多订阅数 | [realtime.go:177](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L177) |
| 单主题长度上限 | 2500 | 单个订阅字符串最大长度 | [realtime.go:178](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L178) |
| `DefaultRateLimitMiddlewarePriority` | -1000 | 限流中间件优先级（其他中间件以此为基准偏移） | [middlewares_rate_limit.go:16](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/middlewares_rate_limit.go#L16) |
