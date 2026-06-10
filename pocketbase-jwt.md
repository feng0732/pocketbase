# PocketBase JWT 鉴权与 Token 派发源码分析

本文档沿着 **登录校验 → Token 签发 → 请求鉴权 → Token 续期** 这条关键路径，逐一剖析 PocketBase 中 JWT 鉴权体系的实现细节。

---

## 一、核心代码文件总览

| 模块 | 文件 | 职责 |
|---|---|---|
| JWT 底层工具 | [tools/security/jwt.go](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/tools/security/jwt.go) | JWT 的签发、解析（含验签/不验签两种） |
| Token 生成模型 | [core/record_tokens.go](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/record_tokens.go) | Record 级别各类 Token（auth/file/verification 等）的 claims 构造 |
| 密码登录接口 | [apis/record_auth_with_password.go](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/apis/record_auth_with_password.go) | `/api/collections/{collection}/auth-with-password` 处理逻辑 |
| Token 刷新接口 | [apis/record_auth_refresh.go](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/apis/record_auth_refresh.go) | `/api/collections/{collection}/auth-refresh` 处理逻辑 |
| 鉴权响应组装 | [apis/record_helpers.go](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/apis/record_helpers.go) | `RecordAuthResponse` / `recordAuthResponse` 统一返回 Token+Record |
| 中间件 | [apis/middlewares.go](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/apis/middlewares.go) | `loadAuthToken`、`RequireAuth` 等鉴权中间件 |
| Token 验签查库 | [core/record_query.go#L483-L540](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/record_query.go#L483-L540) | `FindAuthRecordByToken` 按 Token 反查 Record |
| 密码字段 | [core/field_password.go#L311-L325](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/field_password.go#L311-L325) | `PasswordFieldValue.Validate` 使用 bcrypt 校验密码 |
| 认证模型辅助 | [core/record_model_auth.go](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/record_model_auth.go) | Record 的 TokenKey、密码校验、邮箱等字段访问 |
| Collection Token 配置 | [core/collection_model_auth_options.go](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/collection_model_auth_options.go) | 各类 Token 的 Secret、Duration 默认值与校验 |
| 路由注册 | [apis/base.go](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/apis/base.go) | `NewRouter` 绑定全局中间件与 API 路由 |

---

## 二、JWT 底层：签名与解析

PocketBase 依赖第三方库 `github.com/golang-jwt/jwt/v5`，封装在 [tools/security/jwt.go](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/tools/security/jwt.go) 中，仅使用 **HS256**（HMAC-SHA256）对称加密算法。

### 2.1 签发 `NewJWT`

```go
// tools/security/jwt.go#L46-L58
func NewJWT(payload jwt.MapClaims, signingKey string, duration time.Duration) (string, error) {
    claims := jwt.MapClaims{
        "exp": time.Now().Add(duration).Unix(),
    }
    for k, v := range payload {
        claims[k] = v
    }
    return jwt.NewWithClaims(jwt.SigningMethodHS256, claims).SignedString([]byte(signingKey))
}
```

- `exp` 总是会被注入（取当前时间 + duration）。
- 调用方传入的 `payload` 若含同名 key 会覆盖默认 `exp`（注释中标注为待重构点）。
- `signingKey` 是字符串，最终转 `[]byte` 作为 HMAC 密钥。

### 2.2 解析并验签 `ParseJWT`

```go
// tools/security/jwt.go#L28-L43
func ParseJWT(token string, verificationKey string) (jwt.MapClaims, error) {
    parser := jwt.NewParser(jwt.WithValidMethods([]string{"HS256"}))
    parsedToken, err := parser.Parse(token, func(t *jwt.Token) (any, error) {
        return []byte(verificationKey), nil
    })
    // ...
    return claims, nil
}
```

关键点：**强制限定签名方法为 HS256**，防止 `alg=none` 等算法替换攻击。

### 2.3 仅校验时间 Claim `ParseUnverifiedJWT`

```go
// tools/security/jwt.go#L14-L25
func ParseUnverifiedJWT(token string) (jwt.MapClaims, error) {
    claims := jwt.MapClaims{}
    parser := &jwt.Parser{}
    _, _, err := parser.ParseUnverified(token, claims)
    if err == nil {
        err = jwt.NewValidator(jwt.WithIssuedAt()).Validate(claims)
    }
    return claims, err
}
```

**不验签**，只做 `exp`、`iat`、`nbf` 的时间校验。用在 Token 续期等场景——需要先读 payload 判断能否续期，再结合业务密钥做验签。

---

## 三、Token 的种类与 Claims 结构

在 [core/record_tokens.go#L12-L28](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/record_tokens.go#L12-L28) 中定义了 5 种 Token 类型和标准 Claim 字段：

```go
const (
    TokenTypeAuth          = "auth"           // 登录鉴权
    TokenTypeFile          = "file"           // 私有文件访问
    TokenTypeVerification  = "verification"   // 邮箱验证
    TokenTypePasswordReset = "passwordReset"  // 重置密码
    TokenTypeEmailChange   = "emailChange"    // 修改邮箱
)

const (
    TokenClaimId           = "id"            // Record ID
    TokenClaimType         = "type"          // Token 类型
    TokenClaimCollectionId = "collectionId"  // 所属 Collection ID
    TokenClaimEmail        = "email"
    TokenClaimNewEmail     = "newEmail"
    TokenClaimRefreshable  = "refreshable"   // 是否可被续期
)
```

### 3.1 签名密钥的构成——每个 Record 独立密钥

最关键的设计：**JWT 签名密钥 = `Record.tokenKey + Collection.token.Secret`**。

以 Auth Token 为例，见 [core/record_tokens.go#L51-L73](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/record_tokens.go#L51-L73)：

```go
func (m *Record) newAuthToken(duration time.Duration, refreshable bool) (string, error) {
    key := (m.TokenKey() + m.Collection().AuthToken.Secret)
    // ...
    claims := jwt.MapClaims{
        TokenClaimType:         TokenTypeAuth,
        TokenClaimId:           m.Id,
        TokenClaimCollectionId: m.Collection().Id,
        TokenClaimRefreshable:  refreshable,
    }
    return security.NewJWT(claims, key, duration)
}
```

- `Record.tokenKey` 是每条 auth record 自带的随机字符串字段（存在数据库里），用户改密码时会自动刷新。
- `Collection.AuthToken.Secret` 是 Collection 级别的随机密钥（默认 50 字符，见 [core/collection_model_auth_options.go#L71-L74](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/collection_model_auth_options.go#L71-L74)）。

**安全含义**：用户一旦修改密码 → `tokenKey` 刷新 → 之前签发的所有 Token 立刻失效（因为验签密钥对不上了）。这比纯服务端黑名单机制更轻量。

### 3.2 各类 Token 的默认有效期

见 [core/collection_model_auth_options.go#L47-L92](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/collection_model_auth_options.go#L47-L92) 的 `setDefaultAuthOptions`：

| Token 类型 | 默认有效期 |
|---|---|
| AuthToken | 432000s（5 天） |
| PasswordResetToken | 1800s（30 分钟） |
| EmailChangeToken | 1800s（30 分钟） |
| VerificationToken | 86400s（1 天） |
| FileToken | 180s（3 分钟） |

---

## 四、登录校验路径（密码登录）

入口路由由 [apis/record_auth.go#L31-L33](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/apis/record_auth.go#L31-L33) 注册：

```go
sub.POST("/auth-with-password", recordAuthWithPassword).Bind(
    collectionPathRateLimit("", "authWithPassword", "auth"),
)
```

处理函数位于 [apis/record_auth_with_password.go#L17-L98](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/apis/record_auth_with_password.go#L17-L98)。

### 4.1 整体流程

```
1. findAuthCollection(e)
   └─ 根据 URL 参数 {collection} 查 Collection，校验 IsAuth() && PasswordAuth.Enabled

2. 绑定并校验表单 authWithPasswordForm { identity, password, identityField? }

3. 按 identity 字段查找 Record
   ├─ 若指定了 identityField → 直接用该字段查
   └─ 否则遍历 Collection.PasswordAuth.IdentityFields（默认 ["email"]）逐个试
      └─ findRecordByIdentityField: 走带唯一索引的查询（支持 COLLATE NOCASE）

4. 触发 Hook OnRecordAuthWithPasswordRequest
   └─ 在 Hook 里做密码校验 + 生成响应
```

### 4.2 密码校验——bcrypt

`e.Record.ValidatePassword(e.Password)` 定义在 [core/record_model_auth.go#L78-L85](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/record_model_auth.go#L78-L85)，最终落到 [core/field_password.go#L317-L325](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/field_password.go#L317-L325)：

```go
func (pv PasswordFieldValue) Validate(pass string) bool {
    if pv.Hash == "" || pv.LastError != nil {
        return false
    }
    err := bcrypt.CompareHashAndPassword([]byte(pv.Hash), []byte(pass))
    return err == nil
}
```

### 4.3 反枚举攻击（防用户名穷举）

当用户不存在时，代码会做一次"假"的 bcrypt 校验，让"用户不存在"和"密码错误"两种情况耗时接近，避免通过响应时间判断用户名是否存在：

```go
// apis/record_auth_with_password.go#L87-L94
if e.Record == nil || !e.Record.ValidatePassword(e.Password) {
    if e.Record == nil {
        dummyPasswordCheck(e.App, e.Collection) // 用随机 record + 空密码跑一次 bcrypt
    }
    return e.BadRequestError("Failed to authenticate.", errors.New("invalid login credentials"))
}
```

错误消息统一为 `"Failed to authenticate."`，不区分是用户不存在还是密码错误。

### 4.4 生成 Token 返回

密码校验通过后调用 `RecordAuthResponse`，见 [apis/record_helpers.go#L36-L43](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/apis/record_helpers.go#L36-L43)：

```go
func RecordAuthResponse(e *core.RequestEvent, authRecord *core.Record, authMethod string, meta any) error {
    token, tokenErr := authRecord.NewAuthToken()  // 签发 JWT
    // ...
    return recordAuthResponse(e, authRecord, token, authMethod, meta)
}
```

`recordAuthResponse` 里会做：
1. 超级用户 IP 白名单校验
2. `AuthRule` 规则校验（比如要求 `verified = true`）
3. 触发 `OnRecordAuthRequest` Hook
4. MFA 二次校验（如有配置）
5. 登录告警邮件（新设备指纹）
6. 返回 JSON：`{ token, record, meta? }`

---

## 五、请求鉴权中间件路径

全局中间件在 [apis/base.go#L29-L37](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/apis/base.go#L29-L37) 注册：

```go
pbRouter.Bind(activityLogger())
pbRouter.Bind(panicRecover())
pbRouter.Bind(rateLimit())
pbRouter.Bind(loadAuthToken())       // ← 核心：解析 Token 并注入 e.Auth
pbRouter.Bind(superuserIPsWhitelist())
pbRouter.Bind(securityHeaders())
pbRouter.Bind(BodyLimit(DefaultMaxBodySize))
```

### 5.1 `loadAuthToken`——从 Header 提取并验证 Token

实现位于 [apis/middlewares.go#L184-L209](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/apis/middlewares.go#L184-L209)：

```go
func loadAuthToken() *hook.Handler[*core.RequestEvent] {
    return &hook.Handler[*core.RequestEvent]{
        Func: func(e *core.RequestEvent) error {
            if e.Auth != nil {
                return e.Next()  // 已有 Auth，跳过
            }
            token := getAuthTokenFromRequest(e)
            if token == "" {
                return e.Next()
            }
            record, err := e.App.FindAuthRecordByToken(token, core.TokenTypeAuth)
            if err != nil {
                e.App.Logger().Debug("loadAuthToken failure", "error", err)
            } else if record != nil {
                e.Auth = record  // ← 注入到请求上下文
            }
            return e.Next()
        },
    }
}
```

Token 从 `Authorization` 头取，支持带或不带 `Bearer ` 前缀，见 [apis/middlewares.go#L211-L221](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/apis/middlewares.go#L211-L221)。

**重要**：`loadAuthToken` **不直接返回错误**——Token 无效时只打 Debug 日志，`e.Auth` 保持为 nil。是否拒绝访问交由下游 `RequireAuth` 等中间件决定。这样方便用户自定义鉴权逻辑。

### 5.2 `FindAuthRecordByToken`——两步验证

核心实现在 [core/record_query.go#L483-L540](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/record_query.go#L483-L540)，分两步走：

**Step 1：不验签解析 claims，拿到 id、collectionId、type**

```go
unverifiedClaims, err := security.ParseUnverifiedJWT(token)
id, _ := unverifiedClaims[TokenClaimId].(string)
collectionId, _ := unverifiedClaims[TokenClaimCollectionId].(string)
tokenType, _ := unverifiedClaims[TokenClaimType].(string)
```

**Step 2：查库拿 Record → 组装对应类型的密钥 → 验签**

```go
record, err := app.FindRecordById(collectionId, id)
// ...
var baseTokenKey string
switch tokenType {
case TokenTypeAuth:
    baseTokenKey = record.Collection().AuthToken.Secret
case TokenTypeFile:
    baseTokenKey = record.Collection().FileToken.Secret
// ... 其他类型
}
secret := record.TokenKey() + baseTokenKey
_, err = security.ParseJWT(token, secret)  // 真正的 HS256 验签
```

这个设计的好处：**不需要在 JWT 里放用户敏感信息（如角色、权限）**，所有业务状态都在验签通过后从数据库实时加载。

### 5.3 `RequireAuth`——强制登录

定义在 [apis/middlewares.go#L84-L104](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/apis/middlewares.go#L84-L104)：

```go
func requireAuth(optCollectionNames ...string) func(*core.RequestEvent) error {
    return func(e *core.RequestEvent) error {
        if e.Auth == nil {
            return e.UnauthorizedError("The request requires valid record authorization token.", nil)
        }
        if len(optCollectionNames) > 0 && !slices.Contains(optCollectionNames, e.Auth.Collection().Name) {
            return e.ForbiddenError("The authorized record is not allowed to perform this action.", nil)
        }
        return e.Next()
    }
}
```

典型用法：`RequireAuth("_superusers", "users")` 限定只有特定 Collection 的登录用户能访问。

---

## 六、Token 续期（Refresh）路径

路由：[apis/record_auth.go#L26-L29](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/apis/record_auth.go#L26-L29)

```go
sub.POST("/auth-refresh", recordAuthRefresh).Bind(
    collectionPathRateLimit("", "authRefresh"),
    RequireSameCollectionContextAuth(""),  // 要求已登录且属于当前 Collection
)
```

处理逻辑见 [apis/record_auth_refresh.go#L9-L35](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/apis/record_auth_refresh.go#L9-L35)：

```go
func recordAuthRefresh(e *core.RequestEvent) error {
    record := e.Auth  // 已经由 loadAuthToken + RequireSameCollectionContextAuth 注入
    // ...
    return e.App.OnRecordAuthRefreshRequest().Trigger(event, func(e *core.RecordAuthRefreshRequestEvent) error {
        token := getAuthTokenFromRequest(e.RequestEvent)

        // 先不验签读取 claims，检查 refreshable 标记
        claims, _ := security.ParseUnverifiedJWT(token)
        if v, ok := claims[core.TokenClaimRefreshable]; ok && cast.ToBool(v) {
            token, tokenErr = e.Record.NewAuthToken()  // 签发新 Token
            // ...
        }
        // 不 refreshable 的 Token（如模拟令牌）直接复用原 Token 返回
        return recordAuthResponse(e.RequestEvent, e.Record, token, "", nil)
    })
}
```

### 续期的关键设计

1. **必须先通过完整鉴权**：`RequireSameCollectionContextAuth` 保证了请求带的旧 Token 是有效的（`loadAuthToken` 里已做过完整验签+查库）。
2. **`refreshable` claim**：普通登录 Token 此值为 `true`（见 [core/record_tokens.go#L65](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/record_tokens.go#L65)），而 `NewStaticAuthToken` 生成的静态 Token 此值为 `false`（见 [core/record_tokens.go#L42-L44](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/record_tokens.go#L42-L44)），无法续期。
3. **续期会签发全新 Token**——新的 `exp`、新的签名，旧 Token 不会被主动吊销（直到自然过期或用户改密码导致 `tokenKey` 变化）。

---

## 七、完整调用时序图

### 7.1 密码登录

```
POST /api/collections/users/auth-with-password
        │
        ▼
recordAuthWithPassword
  ├─ findAuthCollection          校验 Collection 存在且开启密码登录
  ├─ BindBody + validate          校验 identity/password 非空
  ├─ findRecordByIdentityField   按 email（或其他身份字段）+ 唯一索引查 Record
  └─ OnRecordAuthWithPasswordRequest Hook
      ├─ Record.ValidatePassword ──► bcrypt.CompareHashAndPassword
      │   └─ 失败时 dummyPasswordCheck 做反枚举
      └─ RecordAuthResponse
          ├─ Record.NewAuthToken ──► security.NewJWT(HS256, tokenKey+AuthToken.Secret, 5d)
          ├─ AuthRule 校验
          ├─ MFA 校验（如启用）
          ├─ authAlert 新设备邮件
          └─ JSON { token, record }
```

### 7.2 受保护请求

```
GET /api/collections/secrets/records
        │
        ▼
loadAuthToken (全局中间件)
  ├─ Authorization: <token> 或 Bearer <token>
  ├─ FindAuthRecordByToken
  │   ├─ ParseUnverifiedJWT ──► 取 id/collectionId/type
  │   ├─ FindRecordById ──► 查 DB
  │   └─ ParseJWT(token, record.TokenKey()+AuthToken.Secret) ──► HS256 验签
  └─ 成功 → e.Auth = record
        │
        ▼
RequireAuth() 等路由中间件
  └─ e.Auth == nil ? 401 : Next()
        │
        ▼
实际业务 Handler
```

### 7.3 Token 续期

```
POST /api/collections/users/auth-refresh
        │
        ▼
loadAuthToken ──► 完整验签通过 → e.Auth 注入
        │
        ▼
RequireSameCollectionContextAuth ──► 校验 Collection 匹配
        │
        ▼
recordAuthRefresh
  ├─ ParseUnverifiedJWT(token) ──► 检查 refreshable == true
  ├─ Record.NewAuthToken() ──► 签发新 Token（新 exp）
  └─ recordAuthResponse ──► 返回 { token: NEW_TOKEN, record }
```

---

## 八、安全设计要点总结

| 设计 | 位置 | 作用 |
|---|---|---|
| HS256 白名单 | [tools/security/jwt.go#L29](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/tools/security/jwt.go#L29) | 防 `alg=none` 攻击 |
| Record 级 tokenKey | [core/record_tokens.go#L56](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/record_tokens.go#L56) | 改密码即吊销所有旧 Token，无需服务端黑名单 |
| Collection 级 Secret | [core/collection_model_auth_options.go#L71-L90](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/collection_model_auth_options.go#L71-L90) | 每种用途独立密钥，隔离风险 |
| bcrypt 密码哈希 | [core/field_password.go#L322](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/field_password.go#L322) | 抗彩虹表、抗暴力破解 |
| dummyPasswordCheck | [apis/record_auth_with_password.go#L125-L136](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/apis/record_auth_with_password.go#L125-L136) | 用户枚举时序攻击防护 |
| 统一错误消息 | [apis/record_auth_with_password.go#L93](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/apis/record_auth_with_password.go#L93) | 不区分"用户不存在"与"密码错误" |
| Token 不从 URL 读取 | [apis/middlewares.go#L212](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/apis/middlewares.go#L212) | 避免 Token 出现在 Referer、日志中 |
| loadAuthToken 不直接报错 | [apis/middlewares.go#L200-L201](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/apis/middlewares.go#L200-L201) | 允许公开+私有混合路由，便于扩展自定义鉴权 |
