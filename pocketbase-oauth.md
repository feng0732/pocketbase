# PocketBase OAuth2 第三方登录协作流程分析

## 一、整体架构概览

PocketBase 的 OAuth2 登录流程横跨三个主要阶段，涉及以下核心文件的协作：

| 阶段 | 功能 | 核心文件 |
|------|------|----------|
| 初始化 | 获取可用 provider 列表、生成 state 和 PKCE 参数 | `apis/record_auth_methods.go` |
| Provider 回调 | 接收第三方回调、通过 Realtime 转发 code、state 校验 | `apis/record_auth_with_oauth2_redirect.go` |
| 账号绑定与会话落地 | 交换 token、获取用户信息、创建/关联账号、生成 JWT | `apis/record_auth_with_oauth2.go`、`apis/record_helpers.go` |

```
┌─────────────┐     ┌────────────────┐     ┌───────────────────┐
│  前端客户端  │────▶│  auth-methods  │────▶│  获取 provider 列表 │
└─────────────┘     └────────────────┘     └───────────────────┘
       │                                                    │
       │  用户跳转至第三方 consent 页面                       │
       ▼                                                    ▼
┌─────────────┐     ┌────────────────┐     ┌───────────────────┐
│  Provider   │────▶│ oauth2-redirect│────▶│  Realtime 转发 code │
└─────────────┘     └────────────────┘     └───────────────────┘
       │                                                    │
       │  前端携带 code 调用                                 │
       ▼                                                    ▼
┌─────────────┐     ┌────────────────┐     ┌───────────────────┐
│  前端客户端  │────▶│auth-with-oauth2│────▶│  绑定账号+生成会话  │
└─────────────┘     └────────────────┘     └───────────────────┘
```

---

## 二、阶段 1：OAuth2 初始化 —— 获取可用 Provider

### 路由入口
- **URL**: `GET /api/collections/{collection}/auth-methods`
- **Handler**: `apis/record_auth_methods.go` 中的 `recordAuthMethods`
- **路由注册**: `apis/record_auth.go` 中的 `bindRecordAuthApi`

### 关键处理逻辑

1. **Provider 配置加载**：遍历集合配置中的 `collection.OAuth2.Providers`，对每个 provider 调用 `config.InitProvider()` 进行初始化。

2. **State 生成**：每个 provider 生成一个 30 字符的随机 `state`：
   ```go
   State: security.RandomString(30)
   ```

3. **PKCE 支持**：对于支持 PKCE 的 provider（如 `provider.PKCE() == true`）：
   - 生成 43 字符的 `codeVerifier`
   - 计算 S256 哈希得到 `codeChallenge`
   - 将 `code_challenge` 和 `code_challenge_method=S256` 拼接到授权 URL

4. **构建授权 URL**：调用 `provider.BuildAuthURL(state, opts...)`，最终由 `tools/auth/base_provider.go` 中的 `BaseProvider.BuildAuthURL` 调用标准 `oauth2.Config.AuthCodeURL` 生成。

5. **Apple 特殊处理**：Apple 的授权 URL 额外附加 `response_mode=form_post` 参数，使其以 POST 方式回调。

### Provider 接口抽象

所有第三方登录实现统一遵循 `tools/auth/auth.go` 中的 `Provider` 接口，核心方法包括：
- `BuildAuthURL(state, opts...)` — 构建授权页 URL
- `FetchToken(code, opts...)` — 用 code 换取 token
- `FetchAuthUser(token)` — 获取标准化的用户信息

基础实现位于 `tools/auth/base_provider.go` 中的 `BaseProvider`，具体 provider（Google、GitHub 等）通过组合它实现差异化逻辑。

---

## 三、阶段 2：Provider 回调 —— oauth2-redirect 与 Realtime 转发

### 路由入口
- **URL**: `GET /api/oauth2-redirect` 和 `POST /api/oauth2-redirect`
- **Handler**: `apis/record_auth_with_oauth2_redirect.go` 中的 `oauth2SubscriptionRedirect`
- **路由注册**: `apis/record_auth.go` 中的 `bindRecordAuthApi`

> 💡 **设计亮点**：PocketBase 没有直接在回调 URL 中完成登录，而是通过 **Realtime（WebSocket）信道** 将 code 转发给前端，由前端再主动调用 `auth-with-oauth2` 完成登录。这样避免了回调 URL 与后端 session 耦合，天然支持 SPA 架构。

### 回调处理流程

1. **参数提取**（支持 GET query 和 POST form-body）：
   - `state`：对应 auth-methods 返回的随机值，同时也是 Realtime clientId
   - `code`：授权码
   - `error`：错误信息
   - `user`：仅 Apple 返回，包含姓名

2. **State 校验（Realtime 客户端查找）**：
   - `state` 参数为空 → 直接失败
   - 通过 `e.App.SubscriptionsBroker().ClientById(data.State)` 查找 Realtime 客户端
   - 校验客户端是否订阅了 `@oauth2` topic
   - **IP 一致性校验**：对比初始化 Realtime 连接的 IP 与当前回调请求 IP，防止 XSRF

3. **Apple 姓名临时存储**：
   - Apple 只在首次回调的 `user` 字段中返回姓名，且后续无法通过 userinfo 接口获取
   - 将解析到的姓名以 `@redirect_name_{code}` 为 key 存入内存 Store，1 分钟后自动过期
   - 在后续 `auth-with-oauth2` 阶段取出使用

4. **Realtime 消息转发**：
   - 将 `{state, code, error}` 序列化为 JSON
   - 通过 `client.Send(msg)` 发送到 `@oauth2` topic
   - 前端通过 Realtime 连接收到消息后，提取 code 并调用 `auth-with-oauth2`

5. **页面重定向**：
   - 成功 → 重定向到 `../_/#/auth/oauth2-redirect-success`
   - 失败 → 重定向到 `../_/#/auth/oauth2-redirect-failure`

---

## 四、阶段 3：auth-with-oauth2 全流程门禁详解

### 路由入口
- **URL**: `POST /api/collections/{collection}/auth-with-oauth2`
- **Handler**: `apis/record_auth_with_oauth2.go` 中的 `recordAuthWithOAuth2`
- **请求体**: `provider`, `code`, `codeVerifier`, `redirectURL`, `createData`

从请求进入到会话返回，共经过 **7 层门禁校验**。下面逐一展开：

---

### 门禁 1：集合级基础校验

进入 handler 后立即执行的三道前置检查：

```go
collection, err := findAuthCollection(e)      // 1. 必须是 auth 类型集合
if !collection.OAuth2.Enabled {                // 2. 集合必须启用 OAuth2
    return e.ForbiddenError("The collection is not configured to allow OAuth2 authentication.", nil)
}
if e.Auth != nil && e.Auth.Collection().Id == collection.Id {
    fallbackAuthRecord = e.Auth                 // 3. 已登录用户可作为 fallback 绑定新 provider
}
```

**代码位置**：`apis/record_auth_with_oauth2.go` 第 31-43 行

---

### 门禁 2：表单参数校验（含 redirectURL）

表单结构体定义在 `apis/record_auth_with_oauth2.go` 中的 `recordOAuth2LoginForm`：

```go
type recordOAuth2LoginForm struct {
    collection   *core.Collection
    CreateData   map[string]any
    Provider     string   // 必填，长度 ≤ 100
    Code         string   // 必填
    CodeVerifier string
    RedirectURL  string   // 必填（新字段，推荐）
    RedirectUrl  string   // 已废弃（旧字段，兼容 v0.22 之前版本）
}
```

**字段 tag**：
- `RedirectURL` → `form:"redirectURL" json:"redirectURL"`
- `RedirectUrl` → `form:"redirectUrl" json:"redirectUrl"`

#### `redirectUrl` → `redirectURL` 的兼容处理

在 `e.BindBody(form)` 解析完请求体之后、`form.validate()` 校验之前，执行了一次旧字段到新字段的兼容迁移：

```go
if form.RedirectUrl != "" && form.RedirectURL == "" {
    e.App.Logger().Warn("[recordAuthWithOAuth2] redirectUrl body param is deprecated and will be removed in the future. Please replace it with redirectURL.")
    form.RedirectURL = form.RedirectUrl
}
```

**代码位置**：`apis/record_auth_with_oauth2.go` 第 53-56 行

> 💡 **兼容策略**：
> - 仅当旧字段有值、新字段为空时才做迁移，避免新字段值被覆盖
> - 迁移同时打一条 Warn 日志，提示开发者尽快切换到新字段
> - `validate()` 只校验 `RedirectURL` 非空，旧字段不参与校验

校验逻辑：

```go
func (form *recordOAuth2LoginForm) validate() error {
    return validation.ValidateStruct(form,
        validation.Field(&form.Provider, validation.Required, validation.Length(0, 100), validation.By(form.checkProviderName)),
        validation.Field(&form.Code, validation.Required),
        validation.Field(&form.RedirectURL, validation.Required), // 仅非空校验，不做白名单匹配
    )
}

func (form *recordOAuth2LoginForm) checkProviderName(value any) error {
    name, _ := value.(string)
    _, ok := form.collection.OAuth2.GetProviderConfig(name)
    if !ok {
        return validation.NewError("validation_invalid_provider", "Provider with name {{.name}} is missing or is not enabled.")
    }
    return nil
}
```

**代码位置**：`apis/record_auth_with_oauth2.go` 第 177-220 行

> ⚠️ **关于 redirectURL 的校验说明**：
> - PocketBase **服务端不做 redirectURL 白名单匹配**，仅校验其非空。
> - `redirectURL` 直接通过 `provider.SetRedirectURL(form.RedirectURL)` 传递给 oauth2 配置。
> - 真正的 redirectURL 一致性校验由 **第三方 Provider 服务端** 在 token 交换阶段完成（OAuth2 规范要求 code 换取 token 时 redirectURL 必须与授权请求完全一致）。
> - 这是一种合理的职责分离：PocketBase 信任各 OAuth2 Provider 自身对回调地址的校验。

---

### 门禁 3：Provider 初始化与 Token 交换

```go
// 从集合配置中查找 provider，找不到直接 500
providerConfig, ok := collection.OAuth2.GetProviderConfig(form.Provider)
provider, err := providerConfig.InitProvider()

provider.SetContext(ctx)                                    // 30s 超时上下文
provider.SetRedirectURL(form.RedirectURL)                   // 设置回调地址

var opts []oauth2.AuthCodeOption
if provider.PKCE() {
    opts = append(opts, oauth2.SetAuthURLParam("code_verifier", form.CodeVerifier))
}

token, err := provider.FetchToken(form.Code, opts...)       // 向 Provider 换取 access_token
authUser, err := provider.FetchAuthUser(token)              // 获取标准化用户信息
```

**代码位置**：`apis/record_auth_with_oauth2.go` 第 65-98 行

- `InitProvider` 位于 `core/collection_model_auth_options.go` 第 504-543 行，负责将集合中配置的 `ClientId`、`ClientSecret`、各 URL 注入 provider 实例。
- `FetchToken` / `FetchAuthUser` 的基础实现位于 `tools/auth/base_provider.go`，最终委托 `golang.org/x/oauth2` 标准库完成 HTTP 调用。

---

### 门禁 4：超级用户集合禁注册

在账号绑定的核心函数 `oauth2Submit` 中（`apis/record_auth_with_oauth2.go` 第 259-402 行），当判定需要**创建新用户**时，第一道防线就是禁止超级用户通过 OAuth2 自助注册：

```go
if e.Record == nil {
    // extra check to prevent creating a superuser record via OAuth2
    if e.Collection.Name == core.CollectionNameSuperusers {
        return errors.New("superusers are not allowed to sign-up with OAuth2")
    }
    // ... 后续新用户创建逻辑
}
```

**代码位置**：`apis/record_auth_with_oauth2.go` 第 261-266 行

> 🔒 **设计意图**：超级用户（`_superusers` 集合）是 PocketBase 的后台管理员身份，必须通过管理界面手动创建或命令行导入，严禁通过外部 OAuth2 自动注册产生，避免权限边界被突破。

---

### 门禁 5：账号查找策略（三级匹配）

在调用 `oauth2Submit` 之前，`recordAuthWithOAuth2` 按以下优先级定位 `authRecord`：

| 优先级 | 匹配方式 | 代码位置 |
|--------|----------|----------|
| 1 | 通过 `_externalAuths` 表按 `(collectionRef, provider, providerId)` 精确查找 | `apis/record_auth_with_oauth2.go` 第 116-123 行 |
| 2 | 使用当前已登录的 `fallbackAuthRecord`（已登录状态下绑定新 provider） | `apis/record_auth_with_oauth2.go` 第 131-133 行 |
| 3 | 通过 OAuth2 返回的邮箱查找已有 auth record | `apis/record_auth_with_oauth2.go` 第 134-139 行 |

> ⚠️ **安全注意**：第 3 级的邮箱匹配存在被恶意预注册抢占的风险，PocketBase 在后续处理中做了防护。

---

### 门禁 6：事件钩子 OnRecordAuthWithOAuth2Request

账号定位完成后，会触发可扩展的事件钩子，允许开发者在绑定前介入：

```go
event := new(core.RecordAuthWithOAuth2RequestEvent)
event.Record = authRecord
event.IsNewRecord = authRecord == nil
// ... 其他字段

return e.App.OnRecordAuthWithOAuth2Request().Trigger(event, func(e *core.RecordAuthWithOAuth2RequestEvent) error {
    if err := oauth2Submit(e, externalAuthRel); err != nil { ... }
    // ... 会话生成
})
```

**事件数据结构**：`core/events.go` 中的 `RecordAuthWithOAuth2RequestEvent`

| 字段 | 类型 | 说明 |
|------|------|------|
| `ProviderName` | `string` | e.g. "google" |
| `ProviderClient` | `Provider` | Provider 接口实例，可调用额外 API |
| `Record` | `*Record` | 定位到的用户（可能为 nil，表示新用户） |
| `OAuth2User` | `*AuthUser` | 标准化的第三方用户信息（结构体，有独立字段） |
| `CreateData` | `map[string]any` | 前端传入的额外创建数据 |
| `IsNewRecord` | `bool` | 是否新用户 |

> 🔍 **事件字段 vs 响应 meta 的区别**：
> - **事件字段**（本钩子）：`OAuth2User` 是 `*AuthUser` 结构体对象，`IsNewRecord` 是独立的 bool 字段，两者并列存在于事件对象上
> - **最终响应 meta**（HTTP 返回值）：是 `map[string]any` 扁平结构——把 `OAuth2User` 的 JSON 字段整体展开后，额外追加一个 `"isNew": true/false` 键值对（详见下文 meta 组装代码）

开发者可以在此钩子中：
- 修改 `e.Record` 自定义账号查找逻辑
- 修改 `e.CreateData` 注入额外字段
- 返回 error 中止登录流程

**注册入口**：`core/base.go` 中的 `app.OnRecordAuthWithOAuth2Request()`

---

### 门禁 7：RecordAuthResponse 会话返回阶段（内部 8 步）

账号绑定成功后，进入统一会话出口 `RecordAuthResponse`。这是所有认证方式（password / oauth2 / otp / refresh）共用的路径，内部按严格顺序执行 **8 个步骤**，任何一步失败都会提前返回错误响应，但 token 在第 1 步就已生成。

> ⚠️ **核心事实**：`token` 先生成，AuthRule、MFA 等校验在其后执行。即使后续校验失败（AuthRule 不通过、IP 不在白名单、MFA 要求二次认证），JWT token 也已在内存中签发完毕——只是不会写入 HTTP 响应返回给客户端。

#### 会话返回阶段内部执行顺序

```
RecordAuthResponse(e, authRecord, "oauth2", meta)
  │                                 │
  │                                 └─ meta 是扁平 map：
  │                                     {expiry, rawUser, id, name, username,
  │                                      avatarURL, accessToken, refreshToken,
  │                                      email, avatarUrl, isNew}
  │
  ├─ ① 生成 auth token                       ← 最先执行，失败直接 500
  │     authRecord.NewAuthToken()
  │
  └─ recordAuthResponse(e, authRecord, token, authMethod, meta)
        │
        ├─ ② 超级用户 IP 白名单校验           ← 仅 _superusers 集合生效
        │     SuperuserIPs 非空时严格匹配 e.RealIP()
        │
        ├─ ③ AuthRule 校验                    ← CanAccessRecord
        │
        ├─ ④ OnRecordAuthRequest 事件钩子
        │   └─ 钩子回调内（开发者 Written() 可提前返回）
        │        │
        │        ├─ ⑤ MFA 检查               ← 需要时返回 401 + mfaId
        │        │
        │        ├─ ⑥ 响应富化                ← Unhide / IgnoreEmailVisibility / expand
        │        │
        │        ├─ ⑦ 登录告警 AuthAlert      ← 新设备指纹发邮件
        │        │
        │        └─ ⑧ 最终响应 JSON           ← HTTP 200 {token, record, meta}
```

#### meta 的实际组装方式

在 `oauth2Submit` 成功返回后、调用 `RecordAuthResponse` 之前，meta 按以下逻辑构造：

```go
// 1. 创建空 map
meta := map[string]any{}

// 2. 把事件上的 *AuthUser 结构体先 Marshal 成 JSON，再 Unmarshal 到 map 中
//    相当于把 AuthUser 的所有 JSON 字段"扁平化"展开到 meta
rawOAuth2User, err := json.Marshal(e.OAuth2User)
err = json.Unmarshal(rawOAuth2User, &meta)

// 3. 额外追加一个 isNew 字段（注意：是 isNew，不是事件字段的 IsNewRecord）
meta["isNew"] = e.IsNewRecord

// 4. 传入 RecordAuthResponse
return RecordAuthResponse(e.RequestEvent, e.Record, core.MFAMethodOAuth2, meta)
```

**代码位置**：`apis/record_auth_with_oauth2.go` 第 159-171 行

**最终 HTTP 响应中 meta 的扁平结构**（来自 `AuthUser` JSON 字段 + isNew）：

| 键 | 来源 | 说明 |
|----|------|------|
| `expiry` | `AuthUser.Expiry` | token 过期时间 |
| `rawUser` | `AuthUser.RawUser` | Provider 返回的原始用户信息 map |
| `id` | `AuthUser.Id` | 第三方用户唯一 ID |
| `name` | `AuthUser.Name` | 显示名称 |
| `username` | `AuthUser.Username` | 用户名 |
| `avatarURL` | `AuthUser.AvatarURL` | 头像 URL |
| `accessToken` | `AuthUser.AccessToken` | 第三方 access_token |
| `refreshToken` | `AuthUser.RefreshToken` | 第三方 refresh_token |
| `email` | `AuthUser.Email` | 已验证邮箱 |
| `avatarUrl` | `AuthUser.AvatarUrl` | 已废弃字段，兼容 v0.22 |
| `isNew` | `event.IsNewRecord` | 是否为新注册用户 |

> 🔍 **关键区分**：
> - `OnRecordAuthWithOAuth2Request` 事件的字段是 **`OAuth2User`（*AuthUser 结构体）和 `IsNewRecord`（独立 bool）**，两者并列
> - 最终 HTTP 响应的 meta 是 **扁平化 map**，`OAuth2User` 被展开成多个顶级键，`IsNewRecord` 被重命名为小写的 `isNew`

#### ③ AuthRule 校验详解

`CanAccessRecord` 的实现位于 `core/record_query.go` 第 599-639 行：

```go
func (app *BaseApp) CanAccessRecord(record *Record, requestInfo *RequestInfo, accessRule *string) (bool, error) {
    // 超级用户在 requestInfo 层面直接放行（但 IP 白名单已在第②步单独校验）
    if requestInfo.HasSuperuserAuth() {
        return true, nil
    }
    // AuthRule = nil → 禁止任何人认证（仅超级用户可登录）
    if accessRule == nil {
        return false, nil
    }
    // AuthRule = "" → 空规则，任何人都可通过
    if *accessRule == "" {
        return true, nil
    }
    // AuthRule = "verified = true" 等表达式 → 走 SQL 过滤校验
    query := app.RecordQuery(record.Collection()).
        Select("(1)").
        AndWhere(dbx.HashExp{record.Collection().Name + ".id": record.Id})
    resolver := NewRecordFieldResolver(app, record.Collection(), requestInfo, true)
    expr, err := search.FilterData(*accessRule).BuildExpr(resolver)
    // ... 执行查询，存在记录即通过
    return exists > 0, nil
}
```

**AuthRule 的三种配置语义**（定义于 `core/collection_model_auth_options.go` 第 98-108 行）：

| AuthRule 值 | 含义 |
|-------------|------|
| `nil` | 彻底关闭该集合的认证（password/OAuth2/OTP 全部失效，仅超级用户可通过后台操作） |
| `""`（空字符串） | 不设额外限制，任何该集合用户只要通过凭证校验即可登录 |
| `"verified = true"` 等表达式 | 必须满足过滤规则才能登录（典型用例：只允许已验证邮箱的用户认证） |

校验不通过时直接返回 `403 Forbidden`（token 已生成但不返回）：

```go
ok, err := e.App.CanAccessRecord(authRecord, originalRequestInfo, authRecord.Collection().AuthRule)
if !ok {
    return firstApiError(err, e.ForbiddenError("The request doesn't satisfy the collection requirements to authenticate.", err))
}
```

**代码位置**：`apis/record_helpers.go` 第 58-61 行

> 💡 **注意**：这里传入的 `originalRequestInfo` 是**认证前**的请求上下文（`requestInfo.Auth` 仍为空或为旧登录态），确保 AuthRule 不会被当前正登录的用户身份 "self-fulfilling"。

关于会话返回阶段的 8 步完整详解（含 MFA、响应富化、AuthAlert 等），见第六节「Token 生成与会话落地全链路」。

---

## 五、账号绑定的事务内处理

账号绑定的核心位于 `apis/record_auth_with_oauth2.go` 中的 `oauth2Submit`，在事务 `e.App.RunInTransaction` 中执行，确保原子性。

### 5.1 新用户创建流程（`authRecord == nil`）

1. **字段映射**：根据集合配置的 `OAuth2.MappedFields` 将 OAuth2 用户信息映射到 record 字段：
   - `id` ← `authUser.Id`
   - `name` ← `authUser.Name`
   - `username` ← `authUser.Username`（附加唯一性和格式校验）
   - `avatarURL` ← 若映射字段是 File 类型则下载头像，否则存 URL 字符串
   - `email` ← `authUser.Email`

2. **内部请求创建 Record**：通过 `sendOAuth2RecordCreateRequest` 发起内部 POST 请求到 `/api/collections/{name}/records`，经过完整的 CRUD 钩子链路。

3. **自动标记已验证**：如果 record 邮箱与 OAuth2 邮箱一致，自动设为 `verified=true`。

### 5.2 已有用户安全加固（`authRecord != nil`）

针对**账号预注册劫持**（攻击者提前用受害者邮箱注册未验证账号）的防护措施：

1. **密码重置**：若 record 未验证且不是当前登录用户，设置随机密码 + 刷新 tokenKey：
   ```go
   if !isLoggedAuthRecord && !e.Record.Verified() {
       e.Record.SetRandomPassword()
   }
   ```

2. **清除旧 OAuth2 关联**：删除该 record 所有旧的 ExternalAuth 关联，防止攻击者已绑定其他恶意 provider：
   ```go
   if !e.Record.Verified() {
       txApp.DeleteAllExternalAuthsByRecord(e.Record)
       optExternalAuth = nil
   }
   ```

3. **邮箱补全与验证升级**：
   - record 邮箱为空 → 用 OAuth2 邮箱填充
   - record 未验证且邮箱匹配 → 标记为已验证

### 5.3 建立 ExternalAuth 关联

如果没有已存在的 ExternalAuth 记录，创建新的关联：

```go
optExternalAuth = core.NewExternalAuth(txApp)
optExternalAuth.SetCollectionRef(e.Record.Collection().Id)
optExternalAuth.SetRecordRef(e.Record.Id)
optExternalAuth.SetProvider(e.ProviderName)
optExternalAuth.SetProviderId(e.OAuth2User.Id)
txApp.Save(optExternalAuth)
```

**数据模型**：`core/external_auth_model.go` 中的 `ExternalAuth`

| 字段 | 说明 |
|------|------|
| `collectionRef` | 所属 auth 集合 ID |
| `recordRef` | 关联的用户 Record ID |
| `provider` | OAuth2 provider 名称 (e.g. "google") |
| `providerId` | 第三方平台的用户唯一 ID |
| `created` / `updated` | 时间戳 |

**验证升级时的级联清理钩子**：当 record 从未验证升级为已验证时，自动清除所有 ExternalAuth 并刷新 tokenKey（`core/external_auth_model.go` 中的 `registerExternalAuthHooks`）。

---

## 六、阶段 5：Token 生成与会话落地全链路

会话落地的入口是 `apis/record_helpers.go` 中的 `RecordAuthResponse`。这是所有认证方式（password / oauth2 / otp / refresh）共用的统一出口，内部执行顺序**严格如下**，任何一步失败都会提前返回：

---

### 6.1 执行顺序总览（代码级精确顺序）

```
RecordAuthResponse(e, authRecord, authMethod, meta)
│
├─ ① 生成 auth token                       ← 最先做，失败直接 500
│     authRecord.NewAuthToken()
│
└─ recordAuthResponse(e, authRecord, token, authMethod, meta)
      │
      ├─ ② 超级用户 IP 白名单校验          ← 仅 _superusers 集合生效
      │     authRecord.IsSuperuser() + isIPInList(Settings.SuperuserIPs)
      │
      ├─ ③ AuthRule 校验                   ← CanAccessRecord(AuthRule)
      │     nil → 全禁 / "" → 全通 / 表达式 → SQL 过滤
      │
      ├─ ④ OnRecordAuthRequest 事件        ← 构造 RecordAuthRequestEvent
      │     注入 token/record/authMethod/meta，开发者可介入
      │     │
      │     └─ 钩子回调内（如果 e.Written() 则提前返回）：
      │          │
      │          ├─ ⑤ MFA 检查
      │          │     checkMFA() → 需要二次认证则写入 401 + mfaId 并返回
      │          │
      │          ├─ ⑥ 响应富化
      │          │     triggerRecordEnrichHooks
      │          │       • 超级用户：Unhide 所有字段
      │          │       • IgnoreEmailVisibility(true)
      │          │       • 根据 ?expand= 参数展开关联记录
      │          │
      │          ├─ ⑦ 登录告警 AuthAlert
      │          │     authAlert() → 新设备 IP+UA 指纹首次登录时发邮件
      │          │
      │          └─ ⑧ 最终响应 JSON
      │                {token, record, meta}  → HTTP 200
      │
      └── (钩子外层结束)
```

---

### 6.2 步骤详解

#### ① 先生成 Token（最早执行，失败直接 500）

```go
func RecordAuthResponse(e *core.RequestEvent, authRecord *core.Record, authMethod string, meta any) error {
    token, tokenErr := authRecord.NewAuthToken()
    if tokenErr != nil {
        return e.InternalServerError("Failed to create auth token.", tokenErr)
    }
    return recordAuthResponse(e, authRecord, token, authMethod, meta)
}
```

**代码位置**：`apis/record_helpers.go` 第 36-43 行

> ⚠️ **注意顺序**：Token 生成是**所有后续校验的前置步骤**——即便后续 IP 白名单或 AuthRule 校验失败导致用户无法登录，token 也已经生成。这是因为 OnRecordAuthRequest 钩子需要在事件数据中携带完整 token，开发者可能在钩子中自定义分发逻辑。

`NewAuthToken` 定义于 `core/record_tokens.go` 第 47-49 行，最终调用 `newAuthToken`：

```go
func (m *Record) newAuthToken(duration time.Duration, refreshable bool) (string, error) {
    // 1. 只有 auth 集合的 record 才能签发
    if !m.Collection().IsAuth() {
        return "", ErrNotAuthRecord
    }

    // 2. 构造签名密钥 = 用户独立 tokenKey + 集合级 secret
    key := (m.TokenKey() + m.Collection().AuthToken.Secret)
    if key == "" {
        return "", ErrMissingSigningKey
    }

    // 3. 组装 JWT Claims
    claims := jwt.MapClaims{
        TokenClaimType:         TokenTypeAuth,    // "auth"
        TokenClaimId:           m.Id,             // 用户 record ID
        TokenClaimCollectionId: m.Collection().Id, // 所属集合 ID
        TokenClaimRefreshable:  refreshable,      // true: 可刷新
    }

    // 4. 默认有效期取集合配置（默认 5 天 = 432000 秒）
    if duration <= 0 {
        duration = m.Collection().AuthToken.DurationTime()
    }

    // 5. HS256 签名并签发
    return security.NewJWT(claims, key, duration)
}
```

**代码位置**：`core/record_tokens.go` 第 51-73 行

`security.NewJWT` 的实现位于 `tools/security/jwt.go` 第 46-58 行：

```go
func NewJWT(payload jwt.MapClaims, signingKey string, duration time.Duration) (string, error) {
    claims := jwt.MapClaims{
        "exp": time.Now().Add(duration).Unix(), // 自动注入过期时间
    }
    for k, v := range payload {
        claims[k] = v
    }
    // 固定使用 HS256 算法
    return jwt.NewWithClaims(jwt.SigningMethodHS256, claims).SignedString([]byte(signingKey))
}
```

**最终签发的 JWT 结构**：
```json
{
    "type": "auth",
    "id": "RECORD_ID",
    "collectionId": "COLLECTION_ID",
    "refreshable": true,
    "exp": 1718000000
}
```
使用密钥 `record.tokenKey + collection.authToken.secret` 进行 HS256 签名。

**密钥分层设计**：
- **`collection.authToken.secret`**：集合级密钥，创建集合时自动生成 50 字符随机串（`core/collection_model_auth_options.go` 第 71-74 行）
- **`record.tokenKey`**：用户级密钥，每个用户独立持有，修改密码或验证升级时自动刷新（`core/record_model_auth.go` 第 35-48 行）
- 两层拼接后作为签名密钥，确保：
  1. 修改密码 = 所有旧 token 立即失效
  2. 轮换集合级 secret = 全量用户登出

---

#### ② 超级用户 IP 白名单校验

```go
if authRecord.IsSuperuser() {
    allowedIPs := e.App.Settings().SuperuserIPs
    if len(allowedIPs) > 0 && !isIPInList(allowedIPs, e.RealIP()) {
        return e.ForbiddenError("", errors.New("superuser IP is not whitelisted"))
    }
}
```

**代码位置**：`apis/record_helpers.go` 第 46-51 行

**规则说明**：
- 仅对 `_superusers` 集合生效（`authRecord.IsSuperuser()` 为 true）
- 读取全局配置 `Settings.SuperuserIPs`（IP 字符串数组）
- **白名单为空 = 不做限制**，所有 IP 均可登录
- **白名单非空 = 严格匹配**，请求真实 IP（`e.RealIP()`，考虑 TrustedProxy 头）必须在列表中，否则返回 403
- 该检查早于 AuthRule，确保超级用户即使满足 AuthRule 也不能从非授权 IP 登录

---

#### ③ AuthRule 校验

```go
originalRequestInfo, err := e.RequestInfo()
// ...
ok, err := e.App.CanAccessRecord(authRecord, originalRequestInfo, authRecord.Collection().AuthRule)
if !ok {
    return firstApiError(err, e.ForbiddenError("The request doesn't satisfy the collection requirements to authenticate.", err))
}
```

**代码位置**：`apis/record_helpers.go` 第 53-61 行

`CanAccessRecord` 的实现位于 `core/record_query.go` 第 599-639 行：

```go
func (app *BaseApp) CanAccessRecord(record *Record, requestInfo *RequestInfo, accessRule *string) (bool, error) {
    // 超级用户在 requestInfo 层面直接放行（但此处在 recordAuthResponse 中已经单独做了 IP 白名单）
    if requestInfo.HasSuperuserAuth() {
        return true, nil
    }
    // AuthRule = nil → 禁止任何人认证（仅超级用户可登录）
    if accessRule == nil {
        return false, nil
    }
    // AuthRule = "" → 空规则，任何人都可通过
    if *accessRule == "" {
        return true, nil
    }
    // AuthRule = "verified = true" 等表达式 → 走 SQL 过滤校验
    query := app.RecordQuery(record.Collection()).
        Select("(1)").
        AndWhere(dbx.HashExp{record.Collection().Name + ".id": record.Id})
    resolver := NewRecordFieldResolver(app, record.Collection(), requestInfo, true)
    expr, err := search.FilterData(*accessRule).BuildExpr(resolver)
    // ... 执行查询，存在记录即通过
    return exists > 0, nil
}
```

**AuthRule 的三种配置语义**（定义于 `core/collection_model_auth_options.go` 第 98-108 行）：

| AuthRule 值 | 含义 |
|-------------|------|
| `nil` | 彻底关闭该集合的认证（password/OAuth2/OTP 全部失效，仅超级用户可通过后台操作） |
| `""`（空字符串） | 不设额外限制，任何该集合用户只要通过凭证校验即可登录 |
| `"verified = true"` 等表达式 | 必须满足过滤规则才能登录（典型用例：只允许已验证邮箱的用户认证） |

> 💡 **注意**：这里传入的 `originalRequestInfo` 是**认证前**的请求上下文（`requestInfo.Auth` 仍为空或为旧登录态），确保 AuthRule 不会被当前正登录的用户身份 "self-fulfilling"。

---

#### ④ OnRecordAuthRequest 事件钩子

```go
event := new(core.RecordAuthRequestEvent)
event.RequestEvent = e
event.Collection = authRecord.Collection()
event.Record = authRecord
event.Token = token
event.Meta = meta
event.AuthMethod = authMethod

return e.App.OnRecordAuthRequest().Trigger(event, func(e *core.RecordAuthRequestEvent) error {
    // ... ⑤⑥⑦⑧ 在钩子回调内执行
})
```

**代码位置**：`apis/record_helpers.go` 第 63-71 行

**事件数据结构**：`core/events.go` 中的 `RecordAuthRequestEvent`

| 字段 | 说明 |
|------|------|
| `Token` | 步骤①已生成的 JWT（开发者可替换） |
| `Record` | 当前认证用户（开发者可修改字段） |
| `AuthMethod` | 对于 OAuth2 固定为 `"oauth2"` |
| `Meta` | OAuth2 场景下为扁平化 map（AuthUser JSON 字段展开 + isNew 布尔字段），详见"meta 的实际组装方式" |

开发者可以在此钩子中：
- 修改 `e.Token` 替换为自定义 token 格式
- 修改 `e.Written()` 提前返回自定义响应（跳过后续 MFA/富化等步骤）
- 记录审计日志、注入额外响应头等

---

#### ⑤ MFA 检查（在钩子回调内）

```go
mfaId, err := checkMFA(e.RequestEvent, e.Record, e.AuthMethod)
// require additional authentication
if mfaId != "" {
    e.JSON(http.StatusUnauthorized, map[string]string{
        "mfaId": mfaId,
    })
    return ErrMFA
}
```

**代码位置**：`apis/record_helpers.go` 第 76-92 行

- 若集合未启用 MFA 或 `authMethod` 为空，直接跳过
- `wantsMFA()` 根据 `collection.MFA.Rule` 表达式判断当前用户是否需要 MFA
- 需要 MFA 时：创建 MFA record，返回 401 + `{"mfaId": "..."}`，由前端完成二次认证后再次调用

---

#### ⑥ 响应富化（在钩子回调内）

```go
// 创建浅拷贝，将当前登录用户挂载到 Auth 上下文中
requestInfo := *originalRequestInfo
requestInfo.Auth = e.Record

err = triggerRecordEnrichHooks(e.App, &requestInfo, []*core.Record{e.Record}, func() error {
    if e.Record.IsSuperuser() {
        e.Record.Unhide(e.Record.Collection().Fields.FieldNames()...)  // 超级用户：所有字段可见
    }
    e.Record.IgnoreEmailVisibility(true)                              // 总是暴露当前用户邮箱
    // expand 关联记录
    expands := strings.Split(e.Request.URL.Query().Get(expandQueryParam), ",")
    if len(expands) > 0 {
        e.App.ExpandRecord(e.Record, expands, expandFetch(e.App, &requestInfo))
    }
    return nil
})
```

**代码位置**：`apis/record_helpers.go` 第 94-119 行

> 关键设计：富化时使用**认证后**的 `requestInfo.Auth = e.Record`，这样 expand 子查询中可以正确解析 `@request.auth.*` 表达式。

---

#### ⑦ 登录告警 AuthAlert（在钩子回调内）

```go
if e.AuthMethod != "" && authRecord.Collection().AuthAlert.Enabled {
    if err = authAlert(e.RequestEvent, e.Record); err != nil {
        e.App.Logger().Warn("[recordAuthResponse] Failed to send login alert", "error", err)
    }
}
```

**代码位置**：`apis/record_helpers.go` 第 121-125 行

- 对 `authMethod` 为空的场景（如 refresh token）不发告警
- 根据 IP + UserAgent 计算 MD5 指纹
- 新指纹首次登录时异步发送告警邮件
- 维护最近 5 个登录来源记录（LRU 淘汰）

---

#### ⑧ 最终响应 JSON（在钩子回调内）

```go
result := struct {
    Meta   any          `json:"meta,omitempty"`
    Record *core.Record `json:"record"`
    Token  string       `json:"token"`
}{
    Token:  e.Token,
    Record: e.Record,
}
if e.Meta != nil {
    result.Meta = e.Meta
}

return execAfterSuccessTx(true, e.App, func() error {
    return e.JSON(http.StatusOK, result)
})
```

**代码位置**：`apis/record_helpers.go` 第 127-142 行

- OAuth2 场景下 `meta` 是**扁平 map**：`AuthUser` 的 JSON 字段整体展开（expiry, rawUser, id, name, username, avatarURL, accessToken, refreshToken, email, avatarUrl），额外追加 `"isNew": true/false`
- 通过 `execAfterSuccessTx` 确保响应只在数据库事务成功提交后才写入

> 💡 PocketBase **不使用服务端 session 和 cookie**。会话完全由前端保存 JWT token，每次请求通过 `Authorization` 头携带。

---

### 6.3 Token 验证过程（校验端）

后续每个请求到达时，由 `apis/middlewares.go` 中的 `loadAuthToken` 中间件处理：

```go
token := getAuthTokenFromRequest(e)  // 从 Authorization 头提取，支持 Bearer 前缀
if token != "" {
    record, err := e.App.FindAuthRecordByToken(token, core.TokenTypeAuth)
    if record != nil {
        e.Auth = record
    }
}
```

`FindAuthRecordByToken` 完整实现位于 `core/record_query.go` 第 483-540 行，包含 **6 步校验**：

| 步骤 | 校验内容 | 代码位置 |
|------|----------|----------|
| 1 | token 非空 | 第 484-486 行 |
| 2 | 解析未签名的 claims，检查必需字段 `id`、`collectionId`、`type` | 第 488-499 行 |
| 3 | 校验 token type 是否在白名单（只接受 `"auth"`） | 第 502-504 行 |
| 4 | 根据 claims 中的 `collectionId` 和 `id` 查询数据库，确认 record 存在且属 auth 集合 | 第 506-513 行 |
| 5 | 根据 token type 找到对应的集合级 secret，与 `record.TokenKey()` 拼接成签名密钥 | 第 515-531 行 |
| 6 | 调用 `security.ParseJWT` 进行 HS256 签名校验和 exp/iat/nbf 时效校验 | 第 533-537 行 |

`ParseJWT` 实现位于 `tools/security/jwt.go` 第 28-43 行：
```go
func ParseJWT(token string, verificationKey string) (jwt.MapClaims, error) {
    parser := jwt.NewParser(jwt.WithValidMethods([]string{"HS256"})) // 强制算法锁定，防 alg=none 攻击
    parsedToken, err := parser.Parse(token, func(t *jwt.Token) (any, error) {
        return []byte(verificationKey), nil
    })
    // ...
}
```

---

## 七、auth-with-oauth2 → 会话返回的完整门禁时序

```
POST /collections/{collection}/auth-with-oauth2
          │
          ▼
    ┌───────────────────────────────┐
    │ 门禁 1: 集合级校验            │
    │  • auth 集合?                 │
    │  • OAuth2 已启用?             │
    │  • fallbackAuthRecord 提取    │
    └─────────────┬─────────────────┘
                  ▼
    ┌───────────────────────────────┐
    │ 门禁 2: 表单校验+兼容处理      │
    │  • BindBody 解析              │
    │  • redirectUrl→redirectURL    │
    │    (旧有值且新为空时迁移+Warn) │
    │  • validate()                 │
    │    - provider 必填+有效        │
    │    - code 必填                │
    │    - redirectURL 必填(仅非空) │
    └─────────────┬─────────────────┘
                  ▼
    ┌───────────────────────────────┐
    │ 门禁 3: Provider 交互         │
    │  • InitProvider (ClientId/Secret/URLs) │
    │  • SetRedirectURL + SetContext(30s)    │
    │  • PKCE 时附加 code_verifier            │
    │  • FetchToken(code) → access_token     │
    │  • FetchAuthUser(token) → AuthUser      │
    └─────────────┬─────────────────┘
                  ▼
    ┌───────────────────────────────┐
    │ 账号定位(三级匹配)             │
    │  1. ExternalAuth 精确查找      │
    │  2. 已登录 fallback            │
    │  3. 邮箱模糊匹配               │
    └─────────────┬─────────────────┘
                  ▼
    ┌───────────────────────────────┐
    │ 门禁 6: 事件钩子               │
    │ OnRecordAuthWithOAuth2Request │
    │ (可修改 Record/CreateData/    │
    │  返回 error 中止)             │
    └─────────────┬─────────────────┘
                  ▼
    ┌─────────────────────────────────────┐
    │ oauth2Submit (DB 事务)              │
    │                                      │
    │ 新用户分支:                          │
    │  ├─ 门禁 4: 禁止 _superusers 注册   │
    │  ├─ 字段映射 (id/name/email/avatar) │
    │  ├─ 内部请求创建 record             │
    │  └─ 邮箱匹配 → auto verified        │
    │                                      │
    │ 旧用户分支:                          │
    │  ├─ 未验证 → 重置随机密码           │
    │  ├─ 未验证 → 清除旧 ExternalAuth    │
    │  ├─ 补全空邮箱                      │
    │  └─ 邮箱匹配 → 升级 verified        │
    │                                      │
    │ 公共步骤:                            │
    │  └─ 创建 ExternalAuth 关联记录       │
    └─────────────┬───────────────────────┘
                  ▼
    ┌────────────────────────────────────────────────┐
    │ RecordAuthResponse —— 统一会话出口             │
    │  (所有认证方式共用此路径)                       │
    │                                                  │
    │  ① 先生成 Token (NewAuthToken → HS256)         │
    │     ↓ 失败直接 500                               │
    │  ② 超级用户 IP 白名单 (SuperuserIPs + RealIP)   │
    │     ↓ 不在白名单 → 403                           │
    │  ③ AuthRule 校验 (CanAccessRecord)              │
    │     ↓ nil→全禁/""→全通/表达式→SQL 过滤          │
    │  ④ OnRecordAuthRequest 事件                     │
    │     └─ 钩子回调内 (Written() 可提前返回)        │
    │          ⑤ MFA 检查                             │
    │             ↓ 需要二次认证 → 401 + mfaId         │
    │          ⑥ 响应富化                              │
    │             • 超级用户: Unhide 所有字段          │
    │             • IgnoreEmailVisibility(true)        │
    │             • ?expand= 展开关联记录              │
    │          ⑦ 登录告警 AuthAlert (IP+UA 指纹)       │
    │          ⑧ 响应 JSON {token, record, meta}       │
    └────────────────────────────────────────────────┘
```

---

## 八、后续请求中的认证中间件

每次请求由 `apis/middlewares.go` 中的 `loadAuthToken` 中间件处理，核心逻辑见 6.3 节。

---

## 九、关键安全设计总结

| 安全点 | 实现位置 | 说明 |
|--------|----------|------|
| **PKCE** | `apis/record_auth_methods.go` 第 156-164 行 | 支持 S256 模式，防止授权码被截获 |
| **State + Realtime 绑定** | `apis/record_auth_with_oauth2_redirect.go` 第 46-66 行 | state 作为 Realtime clientId，附加 IP 校验防 XSRF |
| **超级用户禁 OAuth2 注册** | `apis/record_auth_with_oauth2.go` 第 264-266 行 | 禁止 `_superusers` 集合通过 OAuth2 自动产生账号 |
| **超级用户 IP 白名单** | `apis/record_helpers.go` 第 46-51 行 | `Settings.SuperuserIPs` 非空时严格限制登录来源 IP |
| **AuthRule 三道门** | `core/record_query.go` 第 599-639 行 | nil 全禁 / "" 全通 / 表达式过滤 |
| **防预注册劫持** | `apis/record_auth_with_oauth2.go` 第 344-363 行 | 未验证账号自动重置密码、清除旧 OAuth2 关联 |
| **验证升级清理** | `core/external_auth_model.go` 第 141-177 行 | 从未验证→已验证时清除所有 ExternalAuth 并刷新 tokenKey |
| **安全下载头像** | `apis/record_auth_with_oauth2.go` 第 438-511 行 | `safeHTTPClient` 禁止访问内网/回环 IP，防止 SSRF |
| **每用户独立 tokenKey** | `core/record_tokens.go` 第 56 行 | 密码修改或验证升级时使所有旧 token 失效 |
| **JWT 算法锁定** | `tools/security/jwt.go` 第 29 行 | `jwt.WithValidMethods([]string{"HS256"})` 防止 alg=none 攻击 |
| **token 生成先于校验** | `apis/record_helpers.go` 第 37 行 | 保证 OnRecordAuthRequest 钩子能拿到完整 token，支持自定义分发逻辑 |
