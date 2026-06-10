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

**安全校验：**

1. **Client 存在性校验**：通过 `Broker.ClientById(form.ClientId)` 查找
2. **IP 一致性校验**：防止 clientId 暴力破解，比对 `RealtimeClientIPKey` 存储的 IP
3. **Auth 升级校验**：只允许 guest→auth 的升级，已认证用户不可切换身份

**核心逻辑：**

```go
e.Client.Unsubscribe()              // 先清空所有旧订阅（整体替换策略）
e.Client.Subscribe(e.Subscriptions...)  // 添加新订阅
e.Client.Set(RealtimeClientAuthKey, e.Auth)  // 更新认证状态
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

## 二、事件过滤机制

事件广播触发点在 [bindRealtimeEvents](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L330-L527) 中绑定的多个数据模型 Hook 上，核心分发函数是 [realtimeBroadcastRecord](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L594-L773)。

### 2.1 事件触发源

| Hook | 动作 | 说明 |
|------|------|------|
| `OnModelAfterCreateSuccess` | create | 记录创建成功后广播 |
| `OnModelAfterUpdateSuccess` | update | 记录更新成功后广播 |
| `OnModelDelete` + `OnModelAfterDeleteSuccess` | delete | 采用「预缓存 + 事务后广播」两段式，避免事务回滚造成误通知 |

### 2.2 订阅主题匹配

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

### 2.3 两层权限过滤

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

### 2.4 数据裁剪与增强

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

## 三、连接断开后的处理

### 3.1 正常断开的清理路径

连接断开会触发以下清理链条：

```
客户端关闭 / 超时 / Context 取消
        ↓
realtimeConnect() 函数 return
        ↓
defer Unregister(ce.Client.Id())  [realtime.go:79-81]
        ↓
Broker.Unregister()  [broker.go:58-65]
  ├─ client.Discard()  →  close(channel) 标记 isDiscarded=true
  └─ store.Remove(clientId)  →  从 Broker 注册表删除
```

[DefaultClient.Discard](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/tools/subscriptions/client.go#L252-L263) 是幂等的，多次调用不会 panic。关闭 channel 后，主循环中的 `case msg, ok := <-ce.Client.Channel()` 会收到 `ok=false`，也会退出函数，形成双重保障。

### 3.2 消息发送时的容错

[DefaultClient.Send](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/tools/subscriptions/client.go#L274-L285) 在写入 channel 前会检查 `IsDiscarded()`，并用 `recover()` 兜底：

```go
func (c *DefaultClient) Send(m Message) {
    if c.IsDiscarded() {
        return
    }
    defer func() {
        recover()  // 防止 channel 关闭瞬间的竞态导致 panic
    }()
    c.channel <- m
}
```

### 3.3 认证状态联动清理

当用户相关数据发生变化时，系统会主动清理所有关联客户端的认证缓存，防止权限提升后旧连接滥用。相关函数位于 [realtime.go:251-L328](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L251-L328)：

| 触发场景 | 处理函数 | 行为 |
|----------|----------|------|
| Auth 记录更新（如密码修改） | `realtimeUpdateClientsAuth` | TokenKey 变化则清除 auth，否则更新为最新 record |
| Auth 记录删除 | `realtimeUnsetClientsAuthByRecordModelOrProxy` | 清除该用户所有客户端的 auth |
| Auth 集合密钥变化 | `realtimeUnsetClientsAuthByCollection` | 清除该集合下所有客户端的 auth |
| Auth 集合删除 | `realtimeUnsetClientsAuthByCollection` | 清除该集合下所有客户端的 auth |

这些函数都采用 **分块并发遍历**（`ChunkedClients` + `errgroup.Group`），避免客户端数量大时阻塞。

### 3.4 Delete 事件的事务安全

Delete 操作采用两段式处理以避免事务回滚造成的误通知：

1. **预缓存阶段**（`OnModelDelete`，Priority=99，尽可能靠后）：
   - 调用 `realtimeBroadcastRecord(..., dryCache=true, ...)`
   - 消息暂存在 Client 的 store 中，不立即发送

2. **正式广播阶段**（`OnModelAfterDeleteSuccess`）：
   - 调用 `realtimeBroadcastDryCacheKey` 遍历所有 Client，取出暂存消息并发送

3. **失败回滚阶段**（`OnModelAfterDeleteError`）：
   - 调用 `realtimeUnsetDryCacheKey` 清除所有暂存消息

---

## 四、完整数据流时序图

```
  客户端                          服务端
    |                               |
    |-- GET /api/realtime --------->|  realtimeConnect()
    |                               |    - 创建 DefaultClient (ID=40随机串)
    |                               |    - Broker.Register(client)
    |                               |    - 发送 PB_CONNECT 事件
    |<-- event:PB_CONNECT ----------|
    |    data:{"clientId":"abc"}    |
    |                               |
    |-- POST /api/realtime -------->|  realtimeSetSubscriptions()
    |   {clientId, subscriptions}   |    - IP校验 + Auth校验
    |                               |    - client.Unsubscribe()
    |                               |    - client.Subscribe(topics...)
    |<-- 204 No Content ------------|
    |                               |
    |          (记录被修改)         |
    |                               |  OnModelAfterUpdateSuccess 触发
    |                               |  realtimeBroadcastRecord()
    |                               |    - 分块遍历所有 Client
    |                               |    - 按前缀匹配订阅主题
    |                               |    - CanAccessRecord 权限校验
    |                               |    - filter 二次过滤
    |                               |    - expand/fields 数据裁剪
    |                               |    - client.Send(Message)
    |                               |        ↓
    |                               |    写入 client.channel
    |                               |        ↓
    |                               |    realtimeConnect 主循环读取
    |<-- event:posts/RECORD_ID -----|
    |    data:{"action":"update",...}|
    |                               |
    |   (连接关闭或超时)            |
    |                               |  defer Broker.Unregister()
    |                               |    - client.Discard() → close(channel)
    |                               |    - 从 store 移除
```

---

## 五、关键常量与配置

| 常量 | 值 | 含义 | 位置 |
|------|----|------|------|
| `clientsChunkSize` | 150 | 广播时每批处理的客户端数 | [realtime.go:26](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L26) |
| `IdleTimeout` | 5 分钟 | 无消息时自动断开时间 | [realtime.go:69](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L69) |
| `MaxTimeout` | 30 分钟 | 连接最长存活时间（强制重连） | [realtime.go:70](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L70) |
| Client ID 长度 | 40 | `security.RandomString(40)` | [client.go:94](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/tools/subscriptions/client.go#L94) |
| 订阅数量上限 | 1000 | 单个客户端最多订阅数 | [realtime.go:177](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L177) |
| 单主题长度上限 | 2500 | 单个订阅字符串最大长度 | [realtime.go:178](file:///d:/fz/0601/solo-dogfeeding/code/158-pocketbase/apis/realtime.go#L178) |
