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

## 八、Token 失效与续期边界详解

PocketBase 没有传统的"Token 黑名单"机制，已签发 Token 是否有效完全取决于 **验签密钥是否匹配** + **exp 是否过期**。而验签密钥由两部分组成：`Record.tokenKey + Collection.AuthToken.Secret`。因此，分析失效边界就是追踪这两个值在什么场景下会发生变化，以及 `refreshable` Claim 如何控制续期。

### 8.1 Record 级 tokenKey 的刷新触发——`onRecordSaveExecute`

核心触发逻辑位于 [core/record_model.go#L1429-L1474](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/record_model.go#L1429-L1474)，在每次保存 Record（Create/Update）时的 Hook 中执行：

```go
func onRecordSaveExecute(e *RecordEvent) error {
    if e.Record.Collection().IsAuth() {
        if !e.Record.IsNew() {  // 只针对 Update，不针对 Create
            lastSavedRecord, err := e.App.FindRecordById(e.Record.Collection(), e.Record.Id)
            // ...
            // ensure that the token key is regenerated on password change or email change
            if lastSavedRecord.TokenKey() == e.Record.TokenKey() &&
                (lastSavedRecord.Get(FieldNamePassword) != e.Record.Get(FieldNamePassword) ||
                    lastSavedRecord.Email() != e.Record.Email()) {
                e.Record.RefreshTokenKey()
            }
        }
    }
    // ...
}
```

**触发条件（三个必须同时满足）：**

| 条件 | 说明 |
|---|---|
| `!e.Record.IsNew()` | 是已有 Record 的 Update，不是首次 Create |
| `lastSavedRecord.TokenKey() == e.Record.TokenKey()` | 本次保存还没有手动改过 tokenKey |
| 密码变化 **或** 邮箱变化 | 数据库里的 password hash / email 与新值不一致 |

`RefreshTokenKey()` 的实现见 [core/record_model_auth.go#L46-L48](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/record_model_auth.go#L46-L48)：

```go
func (m *Record) RefreshTokenKey() {
    m.Set(FieldNameTokenKey+autogenerateModifier, "")
}
```

它利用 `TextField` 的 `:autogenerate` 修饰符机制（定义见 [core/field_text.go#L383-L393](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/field_text.go#L383-L393)），在字段真正落库前生成随机字符串。

### 8.1.1 Collection 级 AuthToken.Secret 的刷新触发点

除了 Record 级 `tokenKey`，Collection 级的 `AuthToken.Secret` 也存在自动刷新机制。触发位置在 [core/collection_model.go#L846-L866](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/collection_model.go#L846-L866) 的 `onCollectionSaveExecute`：

```go
func onCollectionSaveExecute(e *CollectionEvent) error {
    // ...
    if !e.Collection.IsNew() {
        oldCollection, err := e.App.FindCachedCollectionByNameOrId(e.Collection.Id)
        // ...

        // invalidate previously issued auth tokens on auth rule change
        if oldCollection.AuthRule != e.Collection.AuthRule &&
            cast.ToString(oldCollection.AuthRule) != cast.ToString(e.Collection.AuthRule) {
            e.Collection.AuthToken.Secret = security.RandomString(50)
        }
    }
    // ... 事务内保存 Collection
}
```

**触发条件（三个条件必须同时满足）：**

| 条件 | 说明 |
|---|---|
| `!e.Collection.IsNew()` | 是已有 Collection 的 Update，不是首次 Create |
| ① `oldCollection.AuthRule != e.Collection.AuthRule` | 指针不同（`*string` 类型，两个 `*string` 变量比较的是内存地址） |
| ② `cast.ToString(old) != cast.ToString(new)` | **实际字符串值**也不同 |

条件①②是 **逻辑 AND** 关系——必须同时成立才刷新 Secret。

---

### `cast.ToString` 对 `*string` 的行为

`cast.ToString` 来自 `github.com/spf13/cast`，对 `*string` 的处理规则：
- `cast.ToString(nil)` → 返回 `""`（空字符串）
- `cast.ToString(&"")` → 返回 `""`（空字符串指针解引用后是空字符串）
- `cast.ToString(&"verified = true")` → 返回 `"verified = true"`

这意味着 `nil` 和 `&""`（指向空字符串的指针）在 `cast.ToString` 下**值相等**，都是 `""`。

---

### 全部 9 种边界组合的真值表

AuthRule 字段类型是 `*string`，在 [core/collection_model_auth_options.go#L109](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/collection_model_auth_options.go#L109) 中定义。以下是所有可能的变化组合：

| # | 旧值 | 新值 | ① 指针不同？ | ② `cast.ToString` 值不同？ | ① AND ② | Secret 刷新？ |
|---|---|---|---|---|---|---|
| 1 | `nil` | `nil` | ❌（均为 nil） | `""` vs `""` → ❌ | false | ❌ |
| 2 | `nil` | `&""` | ✅（nil vs 非 nil） | `""` vs `""` → ❌ | **false** | **❌ 不刷新** |
| 3 | `nil` | `&"verified = true"` | ✅ | `""` vs `"verified = true"` → ✅ | true | ✅ |
| 4 | `&""` | `nil` | ✅（非 nil vs nil） | `""` vs `""` → ❌ | **false** | **❌ 不刷新** |
| 5 | `&""` | `&""` | ❌（同指针） | `""` vs `""` → ❌ | false | ❌ |
| 6 | `&""` | `&"verified = true"` | ✅ | `""` vs `"verified = true"` → ✅ | true | ✅ |
| 7 | `&"verified = true"` | `nil` | ✅ | `"verified = true"` vs `""` → ✅ | true | ✅ |
| 8 | `&"verified = true"` | `&""` | ✅ | `"verified = true"` vs `""` → ✅ | true | ✅ |
| 9 | `&"verified = true"` | `&"role = 'admin'"` | ✅ | `"verified = true"` vs `"role = 'admin'"` → ✅ | true | ✅ |

**关键边界（#2 和 #4）：`nil` ↔ `&""` 互转不会触发 Secret 刷新。**

尽管从语义上看：
- `nil` = "完全禁止该 Collection 的认证"（代码注释：*disallow authentication altogether*）
- `&""` = "允许所有 auth record 认证"（空规则 = 无限制）

两者语义完全不同，但由于 `cast.ToString(nil) == cast.ToString(&"") == ""`，条件②始终为 false，AND 整体短路为 false，不会刷新 Secret。这是一个有意为之的边界优化：`nil ↔ ""` 之间的切换通常不会产生需要吊销的有效 Token（`nil` 时根本无法登录），故无需触发全局吊销。

---

**一旦触发**，AuthToken.Secret 被替换为全新随机字符串 → 该 Collection 下所有用户的所有已签发 Auth Token 在下次请求验签时全部失败（因为验签密钥 `record.TokenKey() + NEW_Secret` 与签发时用的 `record.TokenKey() + OLD_Secret` 不匹配）。

### 8.2 各类场景对 Token 有效性的影响

下面逐一分析各种用户操作对 **已签发 Token** 的影响：

#### ✅ 场景一：用户修改密码 → **所有旧 Token 立即失效**

触发路径：
1. 调用 `SetPassword("newPass")` → password 的 bcrypt hash 变化
2. `Save(record)` → 进入 `onRecordSaveExecute`
3. 检测到 `lastSavedRecord.Get(password) != e.Record.Get(password)` → 调用 `RefreshTokenKey()`
4. 新 tokenKey 写入数据库
5. 旧 Token 验签密钥 `旧tokenKey + Collection.Secret` ≠ 新密钥 → 全部失效

**附加清理**（仅密码变更时，不包含邮箱变更）：

- 清掉所有 MFA 会话：[core/mfa_model.go#L132-L157](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/mfa_model.go#L132-L157)
  ```go
  old := e.Record.Original().GetString(FieldNamePassword + ":hash")
  new := e.Record.GetString(FieldNamePassword + ":hash")
  if old != new {
      err = e.App.DeleteAllMFAsByRecord(e.Record)
  }
  ```
- 清掉所有 AuthOrigin（登录设备指纹）：[core/auth_origin_model.go#L114-L139](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/auth_origin_model.go#L114-L139)
  ```go
  if old != new {
      err = e.App.DeleteAllAuthOriginsByRecord(e.Record)
  }
  ```

这两个 Hook 使用 `e.Record.Original()` 读取变更前的 password hash 做对比，比 `onRecordSaveExecute` 更精准——只认密码变化，不认邮箱变化。

#### ✅ 场景二：用户变更邮箱（确认后）→ **所有旧 Token 立即失效**

变更邮箱分两步：

**Step 1：发起请求** `/api/collections/{collection}/request-email-change`
- 需要带当前登录 Token（已过 `RequireSameCollectionContextAuth`）
- 用户提交新邮箱 + 当前密码
- 系统生成 `TokenTypeEmailChange` 类型的 JWT（含 `newEmail` claim），发邮件到新邮箱
- **此时 tokenKey 不变**，旧 Token 仍然有效

**Step 2：确认邮箱** `/api/collections/{collection}/confirm-email-change` → 见 [apis/record_auth_email_change_confirm.go#L11-L52](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/apis/record_auth_email_change_confirm.go#L11-L52)

```go
e.Record.SetEmail(e.NewEmail)   // 改 email
e.Record.SetVerified(true)      // 顺便标记已验证
if err := e.App.Save(e.Record); err != nil { ... }
```

- `Save` 进入 `onRecordSaveExecute`
- 检测到 `lastSavedRecord.Email() != e.Record.Email()` → `RefreshTokenKey()`
- 旧 Token 全部失效

> 💡 **为什么改邮箱也要让 Token 失效？** 防止原邮箱被攻击者接管后仍持有有效 Token——邮箱作为身份恢复手段变更后，必须吊销所有会话。

#### ✅ 场景三：管理员修改 Collection 的 AuthToken.Secret → **该 Collection 下所有用户的所有 Token 立即失效**

验签密钥是 `record.TokenKey() + record.Collection().AuthToken.Secret`，后者是 Collection 级配置，存在数据库里。管理员在后台修改（或通过 API 更新 Collection）后：
- 后续签发新 Token 使用新 Secret
- 已有 Token 在下次请求走 `FindAuthRecordByToken` 验签时，用新密钥验证旧签名 → 失败
- **不需要改每个用户的 tokenKey**，整个 Collection 一次性全部失效

同样的逻辑适用于其他 Token 类型的 Secret：`PasswordResetToken.Secret`、`EmailChangeToken.Secret`、`VerificationToken.Secret`、`FileToken.Secret`——修改后对应的 Token 全部作废。

#### ✅ 场景四：修改 AuthRule（规则内容实际变化时）→ **该 Collection 所有用户的所有旧 Token 立即失效**

> ⚠️ **边界例外**：`nil` ↔ `&""`（禁止认证 ↔ 允许所有人）互转时**不会**触发 Secret 刷新（详见 8.1.1 节的 9 种组合真值表）。

**触发路径**（完全由框架自动处理，无需手动操作）：

1. 管理员通过 UI 或 API 修改 Collection 的 `AuthRule`（例如从 `""` 改为 `"verified = true"`，或 `"role = 'admin'"` 改为 `nil`）
2. `Save(collection)` → 进入 `onCollectionSaveExecute`（[core/collection_model.go#L846-L866](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/collection_model.go#L846-L866)）
3. 双重检查：`old != new`（指针）**AND** `cast.ToString(old) != cast.ToString(new)`（值）
4. 两者都为 true 时 → **自动执行** `e.Collection.AuthToken.Secret = security.RandomString(50)` 替换为全新随机密钥
5. 新的 `AuthToken.Secret` 随 Collection 配置写入数据库
6. 下一次任何用户请求进来时，`loadAuthToken` → `FindAuthRecordByToken`：
   ```go
   // core/record_query.go#L516-L518
   case TokenTypeAuth:
       baseTokenKey = record.Collection().AuthToken.Secret  // ← 读取的是 NEW Secret
   // ...
   secret := record.TokenKey() + baseTokenKey
   _, err = security.ParseJWT(token, secret)  // ← 旧 Token 用 OLD Secret 签发，验签失败 → return nil
   ```
7. `FindAuthRecordByToken` 返回 nil → `e.Auth` 为 nil → 下游 `RequireAuth` 返回 401

**为什么要这样设计？** AuthRule 代表"谁可以登录"的业务规则，一旦规则内容实际发生变化（如从放开改为仅已验证用户），必须保证所有已登录但不再符合新规则的用户被立刻踢下线。通过刷新 Collection Secret 可以一次性、原子性地吊销该 Collection 下**所有**已签发 Token，而无需逐条处理用户记录。

> 💡 AuthRule 的规则校验（`CanAccessRecord`）仍然存在于 [apis/record_helpers.go#L58-L61](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/apis/record_helpers.go#L58-L61)，但只在**签发新 Token**时（登录/刷新）执行。它是"准入检查"，而 Secret 刷新是"已准入用户的批量驱逐"——两层防线配合。

#### ❌ 场景五：单独修改 verified 状态 → **已签发 Token 不会立即失效**

改 verified 字段走正常的 Record Update，但 `onRecordSaveExecute` 只检测 password 和 email 变化，不检测 verified。所以 verified 变化 **不会触发 `RefreshTokenKey()`**，已签发 Token 的验签密钥不变 → Token 继续有效。

> ⚠️ **注意**：如果管理员同时修改了 `AuthRule`（例如加上 `"verified = true"`），那么会走**场景四**的路径——`onCollectionSaveExecute` 检测到 AuthRule 变化 → 自动刷新 `AuthToken.Secret` → 整个 Collection 所有用户的 Token 全部失效。也就是说，真正让用户下线的是 AuthRule 变更，而不是 verified 字段本身的变更。

但有一种特殊情况：**邮箱验证确认时如果 PasswordAuth 未启用**，见 [apis/record_auth_verification_confirm.go#L50-L53](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/apis/record_auth_verification_confirm.go#L50-L53)：

```go
if !e.Record.Collection().PasswordAuth.Enabled {
    e.Record.SetRandomPassword()  // ← 这里会触发 RefreshTokenKey()
}
```

`SetRandomPassword` 的实现在 [core/record_model_auth.go#L61-L73](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/record_model_auth.go#L61-L73)，内部显式调用了 `RefreshTokenKey()`。这种纯 OAuth2/OTP 用户在首次验证邮箱时，会因为生成随机密码而间接刷新 tokenKey。

#### ❌ 场景六：修改其他字段（name、avatar、自定义字段等）→ **Token 不变**

`onRecordSaveExecute` 只盯着 password 和 email 两个字段。

### 8.3 续期边界：`refreshable` Claim

普通 Auth Token 的 `refreshable = true`，静态 Token 的 `refreshable = false`。在 [apis/record_auth_refresh.go#L20-L34](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/apis/record_auth_refresh.go#L20-L34)：

```go
claims, _ := security.ParseUnverifiedJWT(token)
if v, ok := claims[core.TokenClaimRefreshable]; ok && cast.ToBool(v) {
    token, tokenErr = e.Record.NewAuthToken()  // 签发新 Token
}
// 如果 refreshable=false，直接复用原 token 返回（不签发新的）
```

**注意一个重要细节**：即使 `refreshable=false`，只要原 Token 还没过期、验签通过，请求依然能正常走——只是 `/auth-refresh` 接口不会给你换新 Token。静态 Token 通常用于 API Key 风格的长期访问凭证。

### 8.4 失效场景汇总表

| 操作 | 触发者 | tokenKey 变化？ | Collection Secret 变化？ | 已签发 Token 是否失效 |
|---|---|---|---|---|
| 用户修改密码 | 用户/管理员 | ✅ 刷新 | ❌ | ✅ **该用户全部失效** |
| 用户重置密码（忘记密码） | 用户 | ✅ 刷新 | ❌ | ✅ **该用户全部失效** |
| 邮箱变更（确认后） | 用户 | ✅ 刷新 | ❌ | ✅ **该用户全部失效** |
| 管理员改 Collection.AuthToken.Secret | 管理员 | ❌ | ✅ 变化 | ✅ **该 Collection 全部用户失效** |
| 管理员改 AuthRule（规则内容实际变化） | 管理员 | ❌ | ✅ **自动刷新** | ✅ **该 Collection 全部用户失效**（`nil ↔ ""` 互转除外） |
| 管理员改 verified 字段（单独改） | 管理员 | ❌ | ❌ | ❌ Token 继续有效（需配合 AuthRule 变更才会拦截） |
| 纯 OAuth2 用户首次邮箱验证 | 用户 | ✅* | ❌ | ✅* 因 SetRandomPassword 间接刷新 |
| 修改普通字段（name/avatar 等） | 用户 | ❌ | ❌ | ❌ Token 继续有效 |
| Token 自然过期 | 时间 | ❌ | ❌ | ✅ exp 校验失败 |

> * 仅在 PasswordAuth 未启用的邮箱验证确认场景下间接刷新

### 8.5 续期与失效的交互时序

```
用户 T0 登录 ──────────────────────────────────────────────► 签发 TokenA (exp=T0+5d, refreshable=true)
        │
        ├── T1 (T0+1d)  正常请求 ──► loadAuthToken 验签通过 ──► 200 OK
        │
        ├── T2 (T0+2d)  修改密码 ──► RefreshTokenKey() ──► TokenA 立即失效（仅该用户）
        │                              │
        │                              └── T2+1 用 TokenA 请求 ──► 验签密钥不匹配 ──► 401
        │
        ├── T2' (T0+2.5d) 管理员改 AuthRule ──► onCollectionSaveExecute 自动刷新 AuthToken.Secret
        │      (如从 "" 改为 "verified = true")        │
        │                                                └── 该 Collection ALL 用户 Token 全部失效
        │                                                     │
        │                                                     └── 任意用户下次请求 ──► 401
        │
        └── T3 (T0+3d)  /auth-refresh
               │
               ├─ 如果 TokenA 还在（假设没改密码、AuthRule 也没变）
               │    └─ ParseUnverifiedJWT 检查 refreshable=true ──► 签发 TokenB (exp=T3+5d)
               │
               └─ 如果 TokenA 已因改密码 / AuthRule 变更失效
                    └─ loadAuthToken 阶段就 401，根本到不了 auth-refresh handler
```

**关键区别：**
- 密码/邮箱变更 → 触发 `Record.tokenKey` 刷新 → **仅该用户** 的 Token 失效（粒度细）
- AuthRule 变更 → 触发 `Collection.AuthToken.Secret` 自动刷新 → **该 Collection 所有用户** 的 Token 失效（批量驱逐）
- 两种机制最终都会让 `FindAuthRecordByToken` 中的 `security.ParseJWT(token, record.TokenKey() + Secret)` 验签失败

---

## 九、安全设计要点总结

| 设计 | 位置 | 作用 |
|---|---|---|
| HS256 白名单 | [tools/security/jwt.go#L29](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/tools/security/jwt.go#L29) | 防 `alg=none` 攻击 |
| Record 级 tokenKey（密码/邮箱变更自动刷新） | [core/record_model.go#L1445-L1448](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/record_model.go#L1445-L1448) | 单用户级会话吊销：改密码/改邮箱即该用户所有旧 Token 失效，无需服务端黑名单 |
| Collection 级 AuthToken.Secret（AuthRule 变更自动刷新） | [core/collection_model.go#L861-L865](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/collection_model.go#L861-L865) | 全局级批量驱逐：改 AuthRule 时自动刷新 Secret → 该 Collection **所有用户**的所有 Token 立即失效 |
| Collection 级多用途独立 Secret | [core/collection_model_auth_options.go#L71-L90](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/collection_model_auth_options.go#L71-L90) | Auth / File / Verification / PasswordReset / EmailChange 各有独立密钥，风险隔离 |
| bcrypt 密码哈希 | [core/field_password.go#L322](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/field_password.go#L322) | 抗彩虹表、抗暴力破解 |
| dummyPasswordCheck | [apis/record_auth_with_password.go#L125-L136](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/apis/record_auth_with_password.go#L125-L136) | 用户枚举时序攻击防护 |
| 统一错误消息 | [apis/record_auth_with_password.go#L93](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/apis/record_auth_with_password.go#L93) | 不区分"用户不存在"与"密码错误" |
| Token 不从 URL 读取 | [apis/middlewares.go#L212](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/apis/middlewares.go#L212) | 避免 Token 出现在 Referer、日志中 |
| loadAuthToken 不直接报错 | [apis/middlewares.go#L200-L201](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/apis/middlewares.go#L200-L201) | 允许公开+私有混合路由，便于扩展自定义鉴权 |
| AuthRule 双重防线 | 签发时校验 ([apis/record_helpers.go#L58-L61](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/apis/record_helpers.go#L58-L61)) + 规则变更时全局吊销 ([core/collection_model.go#L861-L865](file:///d:/fz/0601/solo-dogfeeding/code/155-pocketbase/core/collection_model.go#L861-L865)) | 新登录用规则拦截准入，规则变更时用 Secret 刷新驱逐已登录用户 |
