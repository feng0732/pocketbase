# PocketBase MFA / OTP 代码走读（已校准）

## 1. 核心数据模型

### 1.1 MFA 模型
定义于 [mfa_model.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/core/mfa_model.go)

存储在系统集合 `_mfas`（`CollectionNameMFAs`）中，字段结构：

| 字段 | 说明 |
|------|------|
| `collectionRef` | 所属 auth 集合 ID |
| `recordRef` | 用户记录 ID |
| `method` | 已通过的认证方式：`password` / `oauth2` / `otp` |
| `created` / `updated` | 时间戳，用于判断过期 |

三种认证方式常量：
```go
MFAMethodPassword = "password"
MFAMethodOAuth2   = "oauth2"
MFAMethodOTP      = "otp"
```

### 1.2 OTP 模型
定义于 [otp_model.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/core/otp_model.go)

存储在系统集合 `_otps`（`CollectionNameOTPs`）中，字段结构：

| 字段 | 说明 |
|------|------|
| `collectionRef` | 所属 auth 集合 ID |
| `recordRef` | 用户记录 ID |
| `sentTo` | 验证码实际发送目标（邮件发送成功后才回填，详见 §3.1） |
| `password` | 验证码哈希值（和用户密码共用 `PasswordFieldValue`） |
| `created` / `updated` | 时间戳，用于判断过期 |

---

## 2. 集合级配置

定义于 [collection_model_auth_options.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/core/collection_model_auth_options.go)

### 2.1 MFA 配置 `MFAConfig`

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `Enabled` | `false` | 是否启用 MFA |
| `Duration` | `600` (10 分钟) | MFA 会话有效期（秒） |
| `Rule` | `""` | 可选过滤规则，只让满足条件的用户走 MFA；空表示全体用户 |

**约束**：启用 MFA 时必须至少启用 2 种认证方式（password / oauth2 / otp 任意组合）。

### 2.2 OTP 配置 `OTPConfig`

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `Enabled` | `false` | 是否启用 OTP 登录 |
| `Duration` | `180` (3 分钟) | 验证码有效期（秒） |
| `Length` | `8` | 验证码位数（纯数字 `1234567890`，启用时最小 4 位） |
| `EmailTemplate` | 内置模板 | 发送邮件的主题和正文模板 |

---

## 3. OTP 全流程

### 3.1 请求 OTP（发邮件）—— sentTo 写入时机校准
入口：[record_auth_otp_request.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/apis/record_auth_otp_request.go) → `recordRequestOTP`

```
用户 POST /api/collections/{collection}/auth/otp-request
  ├─ 参数：{ email }
  │
  ├─ 校验 collection.OTP.Enabled
  ├─ 通过 email 查找用户
  │   └─ 找不到：返回假的 otpId（200）+ 打日志，防邮箱枚举
  │
  ├─ 生成验证码（长度由 collection.OTP.Length 决定，默认 8 位，最小 4 位）
  │   代码位置：[record_auth_otp_request.go#L44](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/apis/record_auth_otp_request.go#L44)
  │   event.Password = security.RandomStringWithAlphabet(collection.OTP.Length, "1234567890")
  │
  ├─ 限流（非 Dev 环境）：统计用户未过期 OTP 数量
  │   └─ > 9 个：复用最近一个（otps 按 created DESC 排序取 [0]），不再新发
  │
  ├─ 创建 OTP 记录（_otps 集合）—— ⚠️ 注意：此时 sentTo 为空
  │   ├─ SetCollectionRef(e.Record.Collection().Id)
  │   ├─ SetRecordRef(e.Record.Id)
  │   ├─ SetPassword(e.Password)   ← 内部做哈希存储
  │   └─ ❌ 此处未调用 SetSentTo()
  │
  └─ 后台 goroutine 发邮件（routine.FireAndForget）
      └─ 调用 mails.SendRecordOTP()
          │
          ├─ 先构造邮件 Message，To = authRecord.Email()
          ├─ 触发 OnMailerRecordOTPSend Hook
          ├─ Mailer.Send(e.Message)   ← 实际发送 SMTP
          │
          ├─ ✅ 发送成功后，才回填 sentTo（关键路径）
          │   代码位置：[mails/record.go#L91-L121](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/mails/record.go#L91-L121)
          │   ├─ toAddress = e.Message.To[0].Address   ← 取实际收件地址
          │   ├─ FindOTPById(otpId)
          │   ├─ 若 otp.SentTo() != ""：跳过（幂等，已经发过）
          │   └─ otp.SetSentTo(toAddress) + Save(otp)
          │
          └─ 发送失败：删除刚建的 OTP + 打 Error 日志
```

**sentTo 写入时机的影响分析**：

| 场景 | sentTo 值 | 对后续流程的影响 |
|------|-----------|------------------|
| 邮件正在发送中，用户立刻用 OTP 登录 | `""`（空） | §3.2 中 `otpSentTo != ""` 条件不成立 → **不会**自动触发用户 verified 升级 |
| 邮件发送成功，用户用 OTP 登录 | 实际收件邮箱 | sentTo == 用户邮箱 → 自动 `SetVerified(true)`；若 MFA 未启用还会重置随机密码 |
| 邮件发送失败（SMTP 报错） | `""`（OTP 记录已被删除） | 用户拿不到验证码，无法登录 |
| Hook 篡改了 Message.To 地址（自定义发送渠道） | Hook 里改的目标地址 | 回填的是 Hook 修改后的地址，可能与用户邮箱不一致 → verified 不触发 |

### 3.2 用 OTP 登录
入口：[record_auth_with_otp.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/apis/record_auth_with_otp.go) → `recordAuthWithOTP`

```
用户 POST /api/collections/{collection}/auth/otp
  ├─ 参数：{ otpId, password }
  │     password 表单校验：长度 1~71（兼容 bcrypt 哈希长度，不限制最小位数）
  │
  ├─ 校验 collection.OTP.Enabled
  │
  ├─ 查找并校验 OTP（失败统一返回 "Invalid or expired OTP"，防枚举）
  │   ├─ FindOTPById(otpId)
  │   ├─ OTP.CollectionRef == collection.Id
  │   ├─ OTP.HasExpired(collection.OTP.DurationTime())
  │   └─ 找到关联用户 Record
  │
  ├─ 速率限制：@pb_otp_{recordId}
  │   └─ 180 秒内最多 5 次尝试，超过返回 429
  │
  ├─ OTP.ValidatePassword(password)  ← 比对验证码哈希
  │
  ├─ Hook: OnRecordAuthWithOTPRequest
  │   │
  │   ├─ 删除该 OTP 记录（用完即焚）
  │   │
  │   ├─ otpSentTo = e.OTP.SentTo()   ← 读取回填的发送目标
  │   │
  │   ├─ 自动 verified 升级的条件（三个同时满足）：
  │   │   ① !e.Record.Verified()       用户尚未验证
  │   │   ② otpSentTo != ""            sentTo 已成功回填（邮件发完了）
  │   │   ③ e.Record.Email() == otpSentTo   发送目标和用户邮箱一致
  │   │   → SetVerified(true)
  │   │   → 若 MFA 未启用：SetRandomPassword() 重置密码（防预劫持）
  │   │
  │   └─ 调用 RecordAuthResponse(..., MFAMethodOTP, ...)
  │          ↓ 进入通用 MFA 判断
```

**验证码长度配置的影响分析**：

- **启用阶段**：集合保存时校验 `OTP.Length`，启用 OTP 时要求 `>= 4`。配置合法后值直接存入集合配置。
- **生成阶段**：`RandomStringWithAlphabet(collection.OTP.Length, "1234567890")` 直接使用配置值生成纯数字串，**没有运行时二次校验**。如果有人手动改数据库把 Length 改成 3，会真的生成 3 位验证码。
- **校验阶段**：用户提交的 `password` 字段只做长度 `1~71` 校验，**不校验是否匹配配置的 Length**。这意味着无论配置是 4 位还是 8 位，用户输入任意长度数字都能进入哈希比对环节（当然错误密码会在 `ValidatePassword` 被拦截）。这是故意的——避免通过错误提示泄露验证码长度。

---

## 4. MFA 认证流程

核心逻辑位于 [record_helpers.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/apis/record_helpers.go)

所有认证方式（password / oauth2 / otp）最终都会汇入 `RecordAuthResponse` → `checkMFA`。

### 4.1 入口：三种认证方式的终点

| 认证方式 | 文件 | 调用点 |
|----------|------|--------|
| 密码 | [record_auth_with_password.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/apis/record_auth_with_password.go#L96) | `RecordAuthResponse(e, record, MFAMethodPassword, nil)` |
| OAuth2 | [record_auth_with_oauth2.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/apis/record_auth_with_oauth2.go#L171) | `RecordAuthResponse(e, record, MFAMethodOAuth2, meta)` |
| OTP | [record_auth_with_otp.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/apis/record_auth_with_otp.go#L100) | `RecordAuthResponse(e, record, MFAMethodOTP, nil)` |

### 4.2 `checkMFA` 真实走法

函数签名：`checkMFA(e, authRecord, currentAuthMethod) → (mfaId string, err error)`

```
checkMFA 执行步骤：
│
├─ 前置跳过（返回 "", nil）
│   ├─ collection.MFA.Enabled == false   ← 集合根本没开 MFA
│   └─ currentAuthMethod == ""           ← 调用方明确跳过（如内部 token 刷新）
│
├─ wantsMFA() 判断：该用户是否需要走 MFA
│   ├─ MFA.Rule == "" → 返回 (true, nil)（全体启用）
│   ├─ MFA.Rule != "" → 用 search.FilterData 执行规则
│   │   代码位置：[record_helpers.go#L148-L183](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/apis/record_helpers.go#L148-L183)
│   │   ├─ 规则解析失败 → (true, err)   ⚠️ 布尔值为 true，同时带 error
│   │   ├─ SQL 执行失败（非 ErrNoRows）→ (true, err)
│   │   └─ 正常返回 → (exists > 0, nil)
│   │
│   ├─ ⚠️ wantsMFA 返回 err 时的处理（关键校准）
│   │   代码位置：[record_helpers.go#L194-L197](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/apis/record_helpers.go#L194-L197)
│   │   ok, err := wantsMFA(e, authRecord)
│   │   if err != nil {
│   │       return "", e.BadRequestError("Failed to authenticate.",
│   │           fmt.Errorf("MFA rule failure: %w", err))
│   │   }
│   │   → HTTP 400，消息 "Failed to authenticate."
│   │   → **不会**进入 MFA 首次/二次认证分支，直接中断认证
│   │
│   └─ 不需要 MFA（ok == false）→ return "", nil
│
├─ 读取 mfaId：优先 URL query，其次请求 body
│
├─ 分支 A：首次认证（mfaId 为空）
│   ├─ 新建 MFA 记录（_mfas 集合）
│   │   ├─ SetCollectionRef / SetRecordRef
│   │   └─ SetMethod(currentAuthMethod)  ← 记录已通过哪种方式
│   ├─ Save(mfa)
│   └─ return mfa.Id, nil
│       ↓
│       外层 recordAuthResponse 会写 HTTP 401：{ "mfaId": "..." }
│       并返回 ErrMFA 哨兵错误
│
└─ 分支 B：二次认证（mfaId 非空）
    ├─ FindMFAById(mfaId)
    ├─ 定义 defer 风格的 deleteMFA() 闭包
    │
    ├─ 校验失败，删除 MFA 并返回错误：
    │   ├─ MFA 不存在 或 HasExpired(collection.MFA.DurationTime())
    │   │   → "Invalid or expired MFA session"
    │   ├─ MFA.RecordRef != authRecord.Id 或 collection 不匹配
    │   │   → "Invalid MFA session"
    │   └─ MFA.Method == currentAuthMethod
    │       → "A different authentication method is required"
    │         （必须用不同方式完成第二步）
    │
    ├─ 全部校验通过：deleteMFA() 删除会话
    └─ return "", nil  ← 空字符串表示 MFA 完成，继续发 token
```

### 4.3 MFA 规则执行失败的返回值与影响

**实际返回值（校准）**：

| 层级 | 函数 | 返回内容 |
|------|------|----------|
| 内层 | `wantsMFA` | `(true, error)` — 布尔值取安全默认 `true`，同时把规则错误返回 |
| 外层 | `checkMFA` | `("", *ApiError{Status: 400, Message: "Failed to authenticate."})` — `mfaId` 为空字符串，error 是包装了规则错误的 400 |
| 最外层 | `recordAuthResponse` | 直接把 400 error 返回给客户端，**不会**进入 MFA 首次/二次分支，也不会生成 token |

**对启用、校验、失败处理的影响**：

| 流程阶段 | 影响 |
|----------|------|
| **启用（保存阶段）** | `checkRule` 共享和认证阶段完全相同的 `BuildExpr` 代码。语法错误、字段名不符合白名单、引用字段不存在、**`@collection.xxx` 引用的集合不存在**，这 5 类在保存时就会被拦；只有字段类型与比较值不匹配、`@request.*` 值类型不兼容、保存后 schema 改动、数据库异常这几类运行时问题才会保存放行。详见 §4.5。 |
| **校验（认证阶段）** | 规则一旦在运行时解析/执行失败，**所有需要走 MFA 的登录都会被 400 拒绝**。即使是已完成第一步、带着合法 `mfaId` 来做第二步认证的用户也会被挡在外面。 |
| **失败处理** | 失败是"硬拒绝"而非"降级放行"。不会跳过 MFA 直接发 token，也不会创建新的 MFA 会话。客户端拿到的是和普通认证失败一样的 `Failed to authenticate.`，无法区分是密码错还是规则崩了。 |

> 安全设计意图：规则执行失败属于"状态不确定"，宁可错杀（拒绝所有认证）也不放过（跳过 MFA），防止规则被攻击者绕过。

### 4.4 `wantsMFA`：按规则筛选用户

```go
func wantsMFA(e *RequestEvent, record *Record) (bool, error)
```

- 空规则 → `(true, nil)`（所有用户强制 MFA）
- 有规则时构造查询：`SELECT 1 FROM {collection} WHERE id=? AND ({rule})`
- 规则解析/执行**出错时返回 `(true, err)`**（安全优先：失败则收紧）

### 4.5 MFA 规则两阶段校验对比：保存 vs 认证

MFA.Rule 的校验分两个阶段。两阶段**共享同一套字段解析和语法校验代码**，差异只在 RequestInfo 是否有真实值、以及是否执行真实 SQL。

---

#### 概览：两阶段共同 vs 独有（一次看清）

| 校验项 | 保存阶段 checkRule | 认证阶段 wantsMFA | 备注 |
|--------|-------------------|-------------------|------|
| 语法解析（括号、运算符） | ✅ | ✅ | 两阶段都调 `BuildExpr`，代码完全相同 |
| 字段名格式白名单正则 | ✅ | ✅ | 两阶段共用同一套 `allowedFields` |
| 引用字段是否存在于当前集合 | ✅ | ✅ | `processActiveProps` 中校验 |
| **`@collection.xxx` 引用的集合是否存在** | ✅ | ✅ | `processCollectionField → loadCollection`，找不到立刻报错 |
| `@collection.xxx.field` 引用字段是否存在 | ✅ | ✅ | `processActiveProps` 继续校验关联集合字段 |
| `allowHiddenFields=true` | ✅ | ✅ | 两阶段都传 true，`registerJoin` 不检查 ListRule |
| 空规则处理 | 直接跳过，不校验 | 直接返回 `(true, nil)` 全体走 MFA | 行为不同但都不会报错 |
| `RequestInfo` 有真实值 | ❌（空结构体） | ✅（auth/body/query/headers） | 导致 `@request.*` 值差异 |
| 执行真实 SQL | ❌ | ✅ | 认证阶段独有，类型不匹配等问题在此暴露 |
| `UpdateQuery` 处理 JOIN | ❌（不调用） | ✅（但 MFA 场景永不失败） | `allowHiddenFields=true` → `listRuleJoins` 为空 |
| 系统集合禁止改规则 | ✅ | — | 保存阶段独有的额外校验 |

---

#### 阶段一：集合保存时的 `checkRule`
入口：[collection_model_auth_options.go#L203-L219](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/core/collection_model_auth_options.go#L203-L219) → 调用 [collection_validate.go#L477-L503](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/core/collection_validate.go#L477-L503)

**触发条件**：`MFA.Enabled = true` **且** `MFA.Rule != ""`（空规则不经过 checkRule，直接跳过）

**实际执行代码**：
```go
r := NewRecordFieldResolver(validator.app, validator.new, &RequestInfo{}, true)
_, err := search.FilterData(vStr).BuildExpr(r)
```
参数说明：
- `&RequestInfo{}`：**空结构体**——auth=nil、body=nil、query=nil、headers=nil，所以 `@request.*` 字段解析时值为 NULL
- `allowHiddenFields=true`：关联集合 ListRule 检查被跳过
- 只调用 `BuildExpr`，**不执行 `UpdateQuery`、不执行任何 SQL**

额外校验：`ensureNoSystemRuleChange`——系统集合（`_superusers` 等）禁止修改 MFA.Rule

**保存阶段能拦截的所有问题（共 6 类）**：

| 类型 | 例子 | 报错链路 |
|------|------|----------|
| 语法错误（括号不匹配） | `(role = "admin"` | `BuildExpr` → 语法解析器 |
| 语法错误（非法运算符） | `role + "admin"` | `BuildExpr` → 语法解析器 |
| 字段名不符合白名单正则 | `@invalid.field = "x"` | runner 正则 `allowedFields` 校验 |
| 引用当前集合不存在的字段 | `nonexistent_field = "x"` | `processActiveProps` |
| **`@collection.xxx` 引用不存在的集合** | `@collection.nonexistent.id != null` | `processCollectionField` → `loadCollection` → 找不到返回 error |
| 系统集合改规则 | `_superusers` 改 MFA.Rule | `ensureNoSystemRuleChange` |

---

#### 阶段二：用户认证时的 `wantsMFA`
入口：[apis/record_helpers.go#L148-L183](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/apis/record_helpers.go#L148-L183)

**触发条件**：`MFA.Enabled = true`（空规则直接返回 `(true, nil)`，全体走 MFA）

**实际执行代码**：
```go
requestInfo, err := e.RequestInfo()                              // ① 真实请求上下文
resolver := core.NewRecordFieldResolver(e.App, record.Collection(), requestInfo, true)
expr, err := search.FilterData(rule).BuildExpr(resolver)        // ② 和保存阶段完全相同
err = resolver.UpdateQuery(query)                               // ③ MFA 场景永不失败
err = query.AndWhere(expr).Limit(1).Row(&exists)                // ④ 真实 SQL 执行
```

四个步骤的失败分析：

| 步骤 | 是否可能失败 | 失败场景 |
|------|-------------|----------|
| ① `RequestInfo` 提取 | ❌ 几乎不可能 | 由框架填充，失败属于框架级异常 |
| ② `BuildExpr` 语法/字段解析 | ⚠️ 保存后 schema 改动才会失败 | 保存时已校验，认证时此处失败只两种可能：<br>a) 保存后 `@collection.xxx` 引用的集合被删<br>b) 保存后 `@collection.xxx.field` 字段被删<br>c) 手动改数据库写入非法规则 |
| ③ `UpdateQuery` JOIN 处理 | ❌ MFA 场景永不失败 | `allowHiddenFields=true` → `registerJoin` 不执行 `registerRuleJoin` → `listRuleJoins` 为空 → 只有 `query.Distinct(true)`，无 error 返回值 |
| ④ 真实执行 SQL | ✅ **认证阶段独有的失败点** | 见下方「只在登录时暴露的问题」清单 |

---

#### 只在登录时才暴露的问题清单（共 5 类）

这些问题通过了保存阶段的所有校验，只有到用户认证执行真实 SQL 时才会显现：

| # | 类型 | 例子 | 保存时为什么放过 |
|---|------|------|------------------|
| 1 | 字段类型与比较值不匹配（弱类型 SQL 行为） | `email = 123`（字符串 vs 数字） | `BuildExpr` 不做类型检查，字段名合法就通过；SQLite 弱类型，是否报错取决于实际数据 |
| 2 | `@request.body.*` 值类型与运算符不兼容 | `@request.body.age > 18`（body 传 `"abc"`） | 保存时 `RequestInfo` 为空，`@request.body.age` 解析为 NULL，`NULL > 18` 语法完全合法；认证时真实字符串值触发 SQL 类型错误 |
| 3 | `@request.query.*` 值类型与运算符不兼容 | `@request.query.level >= 2`（query 传 `"vip"`） | 同上，保存时 query=nil，值为 NULL |
| 4 | 保存后 schema 被改动 | 保存时 `@collection.perm.level` 存在，之后 perm 集合被删或 level 字段被删 | 保存时 `loadCollection` / 字段解析都能通过，认证时再走 BuildExpr 就找不到了 |
| 5 | 数据库环境异常 | 表损坏、连接池耗尽、磁盘满、SQLite 锁超时 | 保存阶段不执行真实 SQL，无法感知 |

---

#### 配置排障汇总（已校准）

| 现象 | 可能原因 | 排查方向 |
|------|----------|----------|
| **保存集合时报错** `Invalid rule. Raw error: ...` | 6 类见上方「保存阶段能拦截的问题」：语法错、字段名非法、引用字段不存在、**`@collection` 集合不存在**、系统集合改规则 | 1. 检查括号/运算符语法<br>2. 确认字段在 auth 集合和 `@collection.*` 集合中都真实存在<br>3. `@collection.xxx` 时确认 xxx 集合已创建 |
| **保存成功，所有用户登录 400** `Failed to authenticate` | ① 保存后 `@collection.*` 引用的集合/字段被删<br>② 字段类型不匹配（`email = 123` 等）<br>③ `@request.*` 值类型与运算符不兼容<br>④ 数据库环境异常 | 1. 查日志 `MFA rule failure:` 前缀<br>2. 检查所有 `@collection.*` 引用的集合/字段是否还存在<br>3. **排障核心 SQL**（见下方） |
| **保存成功，仅部分用户 400** | 规则对特定行数据触发类型错误（如 `age > 18` 但某用户存 `"abc"`） | 对报错用户 id 执行排障核心 SQL，看哪列数据出问题；检查字段类型一致性 |
| **规则写了但没人走 MFA** | 规则逻辑写反（`1 = 0`、字段值全不匹配） | 执行排障核心 SQL，看 `exists` 是 0 还是 1；不属于失败，是规则逻辑问题 |
| **规则写了但所有用户都走 MFA** | ① MFA.Rule 为空（默认全体）<br>② 规则恒真（`1 = 1`） | 检查 MFA.Rule 是否为空字符串；执行排障核心 SQL 验证逻辑 |

> **排障核心 SQL**：wantsMFA 的等价 SQL 为
> ```sql
> SELECT 1 FROM {auth_table_name} WHERE id='{真实用户id}' AND ({你的规则}) LIMIT 1;
> ```
> 把替换变量填好直接在 SQLite 里跑，能最快区分是语法问题、类型问题、数据问题还是逻辑问题。

---

## 5. 失败处理与安全防护

### 5.1 OTP 失败处理
| 场景 | 行为 |
|------|------|
| OTP ID 不存在 / 已过期 / 密码错误 | 统一返回 `400 Invalid or expired OTP`，不区分原因，防枚举 |
| 速率限制触发（180s > 5 次） | 返回 `429 Too many attempts` |
| 邮件发送失败 | 删除已创建的 OTP 记录 + 打 Error 日志；sentTo 永远不会被回填 |
| 找不到邮箱对应用户 | 返回假的 `otpId`（200 OK）+ 打日志，防邮箱枚举 |
| 邮件发送慢，用户立刻登录 | sentTo 为空 → 不触发 verified 自动升级（但登录本身不受影响） |

### 5.2 MFA 失败处理
| 场景 | 行为 |
|------|------|
| MFA 记录不存在 / 过期 | 删除（如存在）+ `400 Invalid or expired MFA session` |
| MFA 不属于当前用户 / 集合 | `400 Invalid MFA session` |
| 二次认证用了和首次相同的方式 | `400 A different authentication method is required` |
| MFA.Rule 解析/执行失败 | `400 Failed to authenticate.` — 硬拒绝，不降级，不创建 MFA 会话 |

### 5.3 自动清理机制

#### 定时清理（每小时整点）
- `__pbMFACleanup__` cron → `DeleteExpiredMFAs()`
- `__pbOTPCleanup__` cron → `DeleteExpiredOTPs()`
- 遍历所有 auth 集合，按各自配置的 `Duration` 删除 `created < now - Duration` 的记录
- **即使功能被禁用也执行**，确保没有历史脏数据

#### 事件驱动清理
| 触发事件 | 清理动作 | 代码位置 |
|----------|----------|----------|
| 用户修改密码（password hash 变化） | 删除该用户所有 MFA 记录 | [mfa_model.go#L132-L157](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/core/mfa_model.go#L132-L157) |
| 用户 tokenKey 变化（密码重置等） | 删除该用户所有 OTP 记录 | [otp_model.go#L130-L153](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/core/otp_model.go#L130-L153) |
| OTP 认证成功 | 立即删除该 OTP（用完即焚） | [record_auth_with_otp.go#L73-L76](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/apis/record_auth_with_otp.go#L73-L76) |
| MFA 二次认证成功 / 失败 | 删除 MFA 会话 | [record_helpers.go#L232-L254](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/apis/record_helpers.go#L232-L254) |

---

## 6. 完整交互时序（MFA + OTP 组合）

```
┌──────────┐                    ┌──────────────┐                   ┌──────────┐
│  Client  │                    │  PocketBase  │                   │  Email   │
└────┬─────┘                    └──────┬───────┘                   └────┬─────┘
     │                                   │                             │
     │ 1. POST auth/otp-request          │                             │
     │    { email: "u@x.com" }           │                             │
     │──────────────────────────────────>│                             │
     │                                   │ 生成 N 位纯数字验证码         │
     │                                   │ 保存 _otps（sentTo=""）      │
     │                                   │────────────────────────────>│ 发邮件
     │         200 { otpId: "abc" }      │                             │
     │<──────────────────────────────────│                             │
     │                                   │                             │
     │                                   │ 邮件发送成功                  │
     │                                   │   → 回填 sentTo="u@x.com"    │
     │                                   │                             │
     │ 2. POST auth/otp                  │                             │
     │    { otpId: "abc", password }     │                             │
     │──────────────────────────────────>│                             │
     │                                   │ 校验 OTP OK                  │
     │                                   │ sentTo 已回填 → SetVerified  │
     │                                   │ 调用 RecordAuthResponse      │
     │                                   │   → checkMFA(OTP)            │
     │                                   │   → 首次认证，建 MFA 记录     │
     │    401 { mfaId: "mfa123" }        │                             │
     │<──────────────────────────────────│                             │
     │                                   │                             │
     │ 3. POST auth-with-password        │                             │
     │    { identity, password,          │                             │
     │      mfaId: "mfa123" }            │                             │
     │──────────────────────────────────>│                             │
     │                                   │ 校验密码 OK                  │
     │                                   │ 调用 RecordAuthResponse      │
     │                                   │   → checkMFA(Password,mfa123)│
     │                                   │     - MFA.Method=otp != pw ✓ │
     │                                   │     - 删除 MFA 记录           │
     │         200 { token, record }     │                             │
     │<──────────────────────────────────│                             │
```

---

## 7. 校准点汇总

| 校准项 | 旧描述（不准确） | 真实走法（已校准） | 影响 |
|--------|-----------------|-------------------|------|
| **sentTo 写入时机** | 创建 OTP 时同步写入 `SetSentTo(email)` | 邮件 SMTP 发送成功后，在 `SendRecordOTP` Hook 里异步回填，值来自 `Message.To[0].Address` | 邮件发送慢/失败时 sentTo 为空 → OTP 登录后不会自动 verified；Hook 改收件地址会改变 sentTo |
| **验证码长度生效** | 模糊描述"6~8 位" | 配置 `OTP.Length`（默认 8，最小 4）在生成时直接传给 `RandomStringWithAlphabet`；用户提交侧只限制 1~71 位不校验实际长度 | 改数据库能绕过最小 4 位限制；提交侧故意不校验长度以防枚举 |
| **MFA 规则失败返回** | "视为需要 MFA，不跳过" | 返回 HTTP 400 `Failed to authenticate.`，**不创建 MFA 会话，不发 token**，硬拒绝所有该集合的登录 | 规则写错会在运行时阻断所有用户登录；客户端无法区分是密码错还是规则崩了 |
| **MFA 规则两阶段校验** | "保存阶段不会报错" | 保存阶段 `checkRule`（空 RequestInfo + 只调 BuildExpr）能拦截语法错误、字段不存在、**`@collection.xxx` 引用不存在的集合**；只有字段类型不匹配、保存后 schema 改动、`@request.body.*` 值类型、数据库异常等才会到认证阶段 wantsMFA 才暴露 | 保存成功不代表规则可用；上线前用真实用户登录验证；查日志 `MFA rule failure:` 前缀；排障直接在 SQLite 跑 `SELECT 1 FROM {auth_table} WHERE id='...' AND ({规则})` |
| **`@collection` 不存在的报错环节** | "保存时放过，UpdateQuery 才报错" | **保存阶段 BuildExpr 就报错**（`processCollectionField` → `loadCollection` 找不到集合直接 return error）；和字段名校验属于同一层级 | 保存时提示 `Invalid rule` 就检查 `@collection.*` 引用的集合是否已创建 |
| **MFA 规则中 UpdateQuery** | "处理 @collection JOIN 时可能失败" | MFA 规则 `allowHiddenFields=true` → `registerJoin` 不执行 `registerRuleJoin` → `listRuleJoins` 为空 → `UpdateQuery` 中只有 `query.Distinct(true)`，**永远不会返回 error** | 不需要把 UpdateQuery 当作 MFA 规则的潜在失败点排查 |

---

## 8. 关键文件索引

| 功能 | 文件 |
|------|------|
| MFA 模型定义 | [core/mfa_model.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/core/mfa_model.go) |
| MFA 查询 / 清理 | [core/mfa_query.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/core/mfa_query.go) |
| OTP 模型定义 | [core/otp_model.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/core/otp_model.go) |
| OTP 查询 / 清理 | [core/otp_query.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/core/otp_query.go) |
| 集合 MFA/OTP 配置 | [core/collection_model_auth_options.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/core/collection_model_auth_options.go) |
| checkMFA / wantsMFA | [apis/record_helpers.go#L146-L256](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/apis/record_helpers.go#L146-L256) |
| **集合保存 checkRule 校验** | [core/collection_validate.go#L477-L503](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/core/collection_validate.go#L477-L503) |
| **MFA.Rule 保存时触发校验** | [core/collection_model_auth_options.go#L203-L219](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/core/collection_model_auth_options.go#L203-L219) |
| 密码认证（入口） | [apis/record_auth_with_password.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/apis/record_auth_with_password.go) |
| 请求 OTP（sentTo 创建处） | [apis/record_auth_otp_request.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/apis/record_auth_otp_request.go) |
| OTP 登录 | [apis/record_auth_with_otp.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/apis/record_auth_with_otp.go) |
| **sentTo 回填（关键）** | [mails/record.go#L52-L125](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/mails/record.go#L52-L125) |
| OAuth2 登录 | [apis/record_auth_with_oauth2.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/apis/record_auth_with_oauth2.go) |
| 用户密码方法 | [core/record_model_auth.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/core/record_model_auth.go) |
