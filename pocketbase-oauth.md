# PocketBase OAuth2 第三方登录协作流程分析

## 一、整体架构概览

PocketBase 的 OAuth2 登录流程横跨三个主要阶段，涉及以下核心文件的协作：

| 阶段 | 功能 | 核心文件 |
|------|------|----------|
| 初始化 | 获取可用 provider 列表、生成 state 和 PKCE 参数 | [record_auth_methods.go](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_methods.go) |
| Provider 回调 | 接收第三方回调、通过 Realtime 转发 code、state 校验 | [record_auth_with_oauth2_redirect.go](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_with_oauth2_redirect.go) |
| 账号绑定与会话落地 | 交换 token、获取用户信息、创建/关联账号、生成 JWT | [record_auth_with_oauth2.go](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_with_oauth2.go)、[record_helpers.go](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_helpers.go) |

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
- **Handler**: [recordAuthMethods](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_methods.go#L83-L179)
- **路由注册**: [bindRecordAuthApi](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth.go#L22-L24)

### 关键处理逻辑

1. **Provider 配置加载**：遍历集合配置中的 `collection.OAuth2.Providers`，对每个 provider 调用 `config.InitProvider()` 进行初始化。

2. **State 生成**：每个 provider 生成一个 30 字符的随机 `state`：
   ```go
   State: security.RandomString(30)
   ```
   [record_auth_methods.go:140](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_methods.go#L140)

3. **PKCE 支持**：对于支持 PKCE 的 provider（如 `provider.PKCE() == true`）：
   - 生成 43 字符的 `codeVerifier`
   - 计算 S256 哈希得到 `codeChallenge`
   - 将 `code_challenge` 和 `code_challenge_method=S256` 拼接到授权 URL

   [record_auth_methods.go:156-164](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_methods.go#L156-L164)

4. **构建授权 URL**：调用 `provider.BuildAuthURL(state, opts...)`，最终由 [BaseProvider.BuildAuthURL](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/tools/auth/base_provider.go#L151-L153) 调用标准 `oauth2.Config.AuthCodeURL` 生成。

5. **Apple 特殊处理**：Apple 的授权 URL 额外附加 `response_mode=form_post` 参数，使其以 POST 方式回调。

   [record_auth_methods.go:150-154](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_methods.go#L150-L154)

### Provider 接口抽象

所有第三方登录实现统一遵循 [Provider](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/tools/auth/auth.go#L34-L131) 接口，核心方法包括：
- `BuildAuthURL(state, opts...)` — 构建授权页 URL
- `FetchToken(code, opts...)` — 用 code 换取 token
- `FetchAuthUser(token)` — 获取标准化的用户信息

基础实现位于 [BaseProvider](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/tools/auth/base_provider.go#L14-L28)，具体 provider（Google、GitHub 等）通过组合它实现差异化逻辑。

---

## 三、阶段 2：Provider 回调 —— oauth2-redirect 与 Realtime 转发

### 路由入口
- **URL**: `GET /api/oauth2-redirect` 和 `POST /api/oauth2-redirect`
- **Handler**: [oauth2SubscriptionRedirect](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_with_oauth2_redirect.go#L31-L101)
- **路由注册**: [bindRecordAuthApi](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth.go#L12-L18)

> 💡 **设计亮点**：PocketBase 没有直接在回调 URL 中完成登录，而是通过 **Realtime（WebSocket）信道** 将 code 转发给前端，由前端再主动调用 `auth-with-oauth2` 完成登录。这样避免了回调 URL 与后端 session 耦合，天然支持 SPA 架构。

### 回调处理流程

1. **参数提取**（支持 GET query 和 POST form-body）：
   - `state`：对应 auth-methods 返回的随机值，同时也是 Realtime clientId
   - `code`：授权码
   - `error`：错误信息
   - `user`：仅 Apple 返回，包含姓名

   [record_auth_with_oauth2_redirect.go:32-44](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_with_oauth2_redirect.go#L32-L44)

2. **State 校验（Realtime 客户端查找）**：
   - `state` 参数为空 → 直接失败
   - 通过 `e.App.SubscriptionsBroker().ClientById(data.State)` 查找 Realtime 客户端
   - 校验客户端是否订阅了 `@oauth2` topic
   - **IP 一致性校验**：对比初始化 Realtime 连接的 IP 与当前回调请求 IP，防止 XSRF

   [record_auth_with_oauth2_redirect.go:46-66](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_with_oauth2_redirect.go#L46-L66)

3. **Apple 姓名临时存储**：
   - Apple 只在首次回调的 `user` 字段中返回姓名，且后续无法通过 userinfo 接口获取
   - 将解析到的姓名以 `@redirect_name_{code}` 为 key 存入内存 Store，1 分钟后自动过期
   - 在后续 `auth-with-oauth2` 阶段取出使用

   [parseAndStoreAppleRedirectName](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_with_oauth2_redirect.go#L140-L180)

4. **Realtime 消息转发**：
   - 将 `{state, code, error}` 序列化为 JSON
   - 通过 `client.Send(msg)` 发送到 `@oauth2` topic
   - 前端通过 Realtime 连接收到消息后，提取 code 并调用 `auth-with-oauth2`

   [record_auth_with_oauth2_redirect.go:82-93](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_with_oauth2_redirect.go#L82-L93)

5. **页面重定向**：
   - 成功 → 重定向到 `../_/#/auth/oauth2-redirect-success`
   - 失败 → 重定向到 `../_/#/auth/oauth2-redirect-failure`

---

## 四、阶段 3：Token 交换与用户信息获取

### 路由入口
- **URL**: `POST /api/collections/{collection}/auth-with-oauth2`
- **Handler**: [recordAuthWithOAuth2](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_with_oauth2.go#L30-L173)
- **请求体**: `provider`, `code`, `codeVerifier`, `redirectURL`, `createData`

### 表单定义
[recordOAuth2LoginForm](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_with_oauth2.go#L177-L220)

### Token 交换流程

1. **Provider 初始化**：从集合配置中找到对应 provider 配置并初始化：
   ```go
   providerConfig, ok := collection.OAuth2.GetProviderConfig(form.Provider)
   provider, err := providerConfig.InitProvider()
   ```
   [record_auth_with_oauth2.go:66-74](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_with_oauth2.go#L66-L74)

2. **PKCE 处理**：若 provider 支持 PKCE，将 `code_verifier` 加入 token 交换参数：
   ```go
   if provider.PKCE() {
       opts = append(opts, oauth2.SetAuthURLParam("code_verifier", form.CodeVerifier))
   }
   ```
   [record_auth_with_oauth2.go:84-86](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_with_oauth2.go#L84-L86)

3. **获取 Access Token**：调用 `provider.FetchToken(form.Code, opts...)`，最终由 [BaseProvider.FetchToken](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/tools/auth/base_provider.go#L156-L158) 通过 `oauth2.Config.Exchange` 完成。

4. **获取用户信息**：调用 `provider.FetchAuthUser(token)`，返回标准化的 [AuthUser](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/tools/auth/auth.go#L142-L164) 结构：
   ```go
   type AuthUser struct {
       Id           string         // 第三方用户唯一 ID
       Name         string         // 显示名称
       Username     string         // 用户名
       AvatarURL    string         // 头像 URL
       Email        string         // 已验证的邮箱
       AccessToken  string
       RefreshToken string
       RawUser      map[string]any // 原始响应
   }
   ```

5. **Apple 姓名补全**：如果是 Apple 登录且 `authUser.Name` 为空，从 Store 中取出之前暂存的姓名：
   ```go
   nameKey := oauth2RedirectAppleNameStoreKeyPrefix + form.Code
   name, ok := e.App.Store().Get(nameKey).(string)
   ```
   [record_auth_with_oauth2.go:102-111](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_with_oauth2.go#L102-L111)

---

## 五、阶段 4：账号绑定逻辑

账号绑定的核心位于 [oauth2Submit](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_with_oauth2.go#L259-L402)，在事务 `e.App.RunInTransaction` 中执行，确保原子性。

### 5.1 已有账号查找（三级匹配策略）

在 `oauth2Submit` 之前，[recordAuthWithOAuth2](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_with_oauth2.go#L113-L140) 按以下优先级定位 `authRecord`：

| 优先级 | 匹配方式 | 代码位置 |
|--------|----------|----------|
| 1 | 通过 `_externalAuths` 表按 `(collectionRef, provider, providerId)` 精确查找 | [record_auth_with_oauth2.go:116-123](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_with_oauth2.go#L116-L123) |
| 2 | 使用当前已登录的 fallbackAuthRecord（已登录状态下绑定新 provider） | [record_auth_with_oauth2.go:131-133](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_with_oauth2.go#L131-L133) |
| 3 | 通过 OAuth2 返回的邮箱查找已有 auth record | [record_auth_with_oauth2.go:134-139](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_with_oauth2.go#L134-L139) |

> ⚠️ **安全注意**：第 3 级的邮箱匹配存在被恶意预注册抢占的风险，PocketBase 在后续处理中做了防护。

### 5.2 新用户创建流程（`authRecord == nil`）

[oauth2Submit](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_with_oauth2.go#L261-L333)

1. **字段映射**：根据集合配置的 `OAuth2.MappedFields` 将 OAuth2 用户信息映射到 record 字段：
   - `id` ← `authUser.Id`
   - `name` ← `authUser.Name`
   - `username` ← `authUser.Username`（附加唯一性和格式校验）
   - `avatarURL` ← 若映射字段是 File 类型则下载头像，否则存 URL 字符串
   - `email` ← `authUser.Email`

   [record_auth_with_oauth2.go:280-318](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_with_oauth2.go#L280-L318)

2. **内部请求创建 Record**：通过 `sendOAuth2RecordCreateRequest` 发起内部 POST 请求到 `/api/collections/{name}/records`，经过完整的 CRUD 钩子链路：
   ```go
   ir := &core.InternalRequest{
       Method: http.MethodPost,
       URL:    "/api/collections/" + e.Collection.Name + "/records",
       Body:   payload,
   }
   ```
   [sendOAuth2RecordCreateRequest](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_with_oauth2.go#L404-L426)

3. **自动标记已验证**：如果 record 邮箱与 OAuth2 邮箱一致，自动设为 `verified=true`。

### 5.3 已有用户安全加固（`authRecord != nil`）

[oauth2Submit](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_with_oauth2.go#L334-L385)

针对**账号预注册劫持**（攻击者提前用受害者邮箱注册未验证账号）的防护措施：

1. **密码重置**：若 record 未验证且不是当前登录用户，设置随机密码：
   ```go
   if !isLoggedAuthRecord && !e.Record.Verified() {
       e.Record.SetRandomPassword()  // 30 字符随机密码 + 刷新 tokenKey
   }
   ```
   [record_auth_with_oauth2.go:344-347](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_with_oauth2.go#L344-L347)

2. **清除旧 OAuth2 关联**：删除该 record 所有旧的 ExternalAuth 关联，防止攻击者已绑定其他恶意 provider：
   ```go
   if !e.Record.Verified() {
       txApp.DeleteAllExternalAuthsByRecord(e.Record)
       optExternalAuth = nil
   }
   ```
   [record_auth_with_oauth2.go:357-363](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_with_oauth2.go#L357-L363)

3. **邮箱补全与验证升级**：
   - record 邮箱为空 → 用 OAuth2 邮箱填充
   - record 未验证且邮箱匹配 → 标记为已验证

### 5.4 建立 ExternalAuth 关联

[oauth2Submit](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_with_oauth2.go#L388-L398)

如果没有已存在的 ExternalAuth 记录，创建新的关联：

```go
optExternalAuth = core.NewExternalAuth(txApp)
optExternalAuth.SetCollectionRef(e.Record.Collection().Id)
optExternalAuth.SetRecordRef(e.Record.Id)
optExternalAuth.SetProvider(e.ProviderName)
optExternalAuth.SetProviderId(e.OAuth2User.Id)
txApp.Save(optExternalAuth)
```

**数据模型**：[ExternalAuth](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/core/external_auth_model.go#L22-L119)

| 字段 | 说明 |
|------|------|
| `collectionRef` | 所属 auth 集合 ID |
| `recordRef` | 关联的用户 Record ID |
| `provider` | OAuth2 provider 名称 (e.g. "google") |
| `providerId` | 第三方平台的用户唯一 ID |
| `created` / `updated` | 时间戳 |

**验证升级时的级联清理钩子**：当 record 从未验证升级为已验证时，自动清除所有 ExternalAuth 并刷新 tokenKey（[registerExternalAuthHooks](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/core/external_auth_model.go#L141-L177)）。

---

## 六、阶段 5：会话落地机制

### 6.1 Auth Token 生成

会话落地的入口是 [RecordAuthResponse](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_helpers.go#L36-L43)：

```go
token, tokenErr := authRecord.NewAuthToken()
```

[NewAuthToken](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/core/record_tokens.go#L47-L49) 最终调用 `newAuthToken`：

```go
func (m *Record) newAuthToken(duration time.Duration, refreshable bool) (string, error) {
    key := (m.TokenKey() + m.Collection().AuthToken.Secret)
    claims := jwt.MapClaims{
        TokenClaimType:         TokenTypeAuth,    // "auth"
        TokenClaimId:           m.Id,
        TokenClaimCollectionId: m.Collection().Id,
        TokenClaimRefreshable:  refreshable,      // true
    }
    return security.NewJWT(claims, key, duration)
}
```

[record_tokens.go:51-73](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/core/record_tokens.go#L51-L73)

**JWT 签名密钥构成**：`record.tokenKey + collection.authToken.secret`
- `tokenKey` 是每个用户 Record 独有的随机字段（[FieldNameTokenKey](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/core/record_model_auth.go#L35-L48)）
- 修改密码时会自动刷新 `tokenKey`，使旧 token 全部失效

### 6.2 通用 Auth 响应处理

[recordAuthResponse](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_helpers.go#L45-L144) 执行以下步骤：

1. **MFA 检查**（[checkMFA](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_helpers.go#L189-L256)）：
   - 若集合启用了 MFA 且符合 MFA Rule，则要求二次认证
   - 首次调用：创建 MFA record，返回 `mfaId`，HTTP 401
   - 二次调用：验证 mfaId 和不同的 authMethod

2. **Record 数据富化**：
   - 对超级用户：取消所有隐藏字段
   - 暴露当前认证用户的邮箱（`IgnoreEmailVisibility(true)`）
   - 根据 `?expand=` 参数展开关联记录

3. **登录告警（Auth Alert）**：若集合配置了 `AuthAlert.Enabled`：
   - 生成 IP + UserAgent 的 MD5 指纹
   - 新指纹首次登录时发送邮件告警
   - 维护最近 5 个登录来源（[maxAuthOrigins](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_helpers.go#L587-L664)）

4. **最终响应**：
   ```json
   {
       "token":  "eyJhbGciOiJIUzI1NiIs...",
       "record": { /* 用户 record */ },
       "meta":   { /* OAuth2 user info + isNew flag */ }
   }
   ```
   [record_helpers.go:127-L142](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_helpers.go#L127-L142)

> 💡 PocketBase **不使用服务端 session 和 cookie**。会话完全由前端保存 JWT token，每次请求通过 `Authorization` 头携带。

### 6.3 后续请求中的认证中间件

每次请求由 [loadAuthToken](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/middlewares.go#L184-L209) 中间件处理：

```go
token := getAuthTokenFromRequest(e)  // 从 Authorization 头提取，支持 Bearer 前缀
if token != "" {
    record, err := e.App.FindAuthRecordByToken(token, core.TokenTypeAuth)
    if record != nil {
        e.Auth = record  // 挂载到请求上下文
    }
}
```

[middlewares.go:211-221](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/middlewares.go#L211-L221)

---

## 七、事件钩子（Hook）体系

OAuth2 流程暴露两个关键钩子，供开发者自定义扩展：

### 7.1 OnRecordAuthWithOAuth2Request

**触发时机**：Token 交换成功、账号定位完成后，在 `oauth2Submit` 之前。

**事件数据**：[RecordAuthWithOAuth2RequestEvent](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/core/events.go#L561-L571)

| 字段 | 说明 |
|------|------|
| `ProviderName` | e.g. "google" |
| `ProviderClient` | Provider 接口实例，可调用额外 API |
| `Record` | 定位到的用户（可能为 nil，表示新用户） |
| `OAuth2User` | 标准化的第三方用户信息 |
| `CreateData` | 前端传入的额外创建数据 |
| `IsNewRecord` | 是否新用户 |

**典型用法**：
- 修改 `e.Record` 自定义账号查找逻辑
- 修改 `e.CreateData` 注入额外字段
- 在绑定前执行自定义校验

注册位置：[app.OnRecordAuthWithOAuth2Request()](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/core/base.go#L1075-L1077)

### 7.2 OnRecordAuthRequest

**触发时机**：所有认证方式（password/oauth2/otp/refresh）成功后，在返回响应之前。

**事件数据**：包含 `Token`、`Record`、`AuthMethod`（值为 `"oauth2"`）、`Meta`。

可用于：自定义 token 格式、审计日志、额外的响应头注入。

---

## 八、完整协作时序图

```
前端                     PocketBase                    Provider
 │                         │                             │
 │── GET auth-methods ────▶│                             │
 │◀── providers + state ───│                             │
 │                         │                             │
 │── 跳转 consent URL ──────────────────────────────────▶│
 │                         │◀── 用户授权 ─────────────────│
 │                         │── GET/POST oauth2-redirect ──│
 │                         │   (state, code)              │
 │                         │  校验 state Realtime client   │
 │◀── Realtime @oauth2 ────│                             │
 │    {code, state}         │                             │
 │                         │                             │
 │── POST auth-with-oauth2 ──────────────────────────────│
 │   {provider, code, ...}  │                             │
 │                         │── FetchToken(code) ─────────▶│
 │                         │◀── access_token ─────────────│
 │                         │── FetchAuthUser(token) ─────▶│
 │                         │◀── 用户信息 ─────────────────│
 │                         │                             │
 │                         │  查找/创建 authRecord         │
 │                         │  事务：                       │
 │                         │   • 新用户：映射字段+创建     │
 │                         │   • 旧用户：安全加固          │
 │                         │   • 创建 ExternalAuth 关联    │
 │                         │   • 生成 JWT token           │
 │◀── {token, record, meta}│                             │
 │                         │                             │
 │── 请求 API (Auth header)│                             │
 │◀── loadAuthToken 中间件 │                             │
 │     校验 JWT → e.Auth    │                             │
```

---

## 九、关键安全设计总结

| 安全点 | 实现位置 | 说明 |
|--------|----------|------|
| **PKCE** | [record_auth_methods.go:156-164](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_methods.go#L156-L164) | 支持 S256 模式，防止授权码被截获 |
| **State + Realtime 绑定** | [record_auth_with_oauth2_redirect.go:46-66](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_with_oauth2_redirect.go#L46-L66) | state 作为 Realtime clientId，附加 IP 校验防 XSRF |
| **防预注册劫持** | [record_auth_with_oauth2.go:344-363](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_with_oauth2.go#L344-L363) | 未验证账号自动重置密码、清除旧 OAuth2 关联 |
| **验证升级清理** | [external_auth_model.go:141-177](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/core/external_auth_model.go#L141-L177) | 从未验证→已验证时清除所有 ExternalAuth 并刷新 tokenKey |
| **安全下载头像** | [record_auth_with_oauth2.go:438-511](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/apis/record_auth_with_oauth2.go#L438-L511) | `safeHTTPClient` 禁止访问内网/回环 IP，防止 SSRF |
| **每用户独立 tokenKey** | [record_tokens.go:56](file:///d:/fz/0601/solo-dogfeeding/code/156-pocketbase/core/record_tokens.go#L56) | 密码修改或验证升级时使所有旧 token 失效 |
