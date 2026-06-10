# PocketBase MFA / OTP 代码走读

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
| `sentTo` | 验证码发送目标（通常是邮箱） |
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
| `Length` | `8` | 验证码位数（纯数字 `1234567890`） |
| `EmailTemplate` | 内置模板 | 发送邮件的主题和正文模板 |

---

## 3. OTP 全流程

### 3.1 请求 OTP（发邮件）
入口：[record_auth_otp_request.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/apis/record_auth_otp_request.go) → `recordRequestOTP`

```
用户 POST /api/collections/{collection}/auth/otp-request
  ├─ 参数：{ email }
  │
  ├─ 校验 collection.OTP.Enabled
  ├─ 通过 email 查找用户
  │   └─ 找不到：返回假的 otpId（200）+ 打日志，防邮箱枚举
  │
  ├─ 生成 6~8 位纯数字验证码
  │
  ├─ 限流（非 Dev 环境）：统计用户未过期 OTP 数量
  │   └─ > 9 个：复用最近一个（otps 按 created DESC 排序取 [0]），不再新发
  │
  ├─ 创建 OTP 记录（_otps 集合）
  │   ├─ SetCollectionRef / SetRecordRef
  │   ├─ SetPassword(验证码)    ← 内部做哈希存储
  │   └─ SetSentTo(email)
  │
  └─ 后台 goroutine 发邮件（routine.FireAndForget）
      ├─ 成功：返回 { otpId }
      └─ 失败：删除刚建的 OTP + 打日志
```

### 3.2 用 OTP 登录
入口：[record_auth_with_otp.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/apis/record_auth_with_otp.go) → `recordAuthWithOTP`

```
用户 POST /api/collections/{collection}/auth/otp
  ├─ 参数：{ otpId, password }
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
  │   ├─ 若用户未验证 且 sentTo == 用户邮箱：
  │   │   ├─ SetVerified(true)
  │   │   └─ 若 MFA 未启用：SetRandomPassword() 重置密码（防预劫持）
  │   │
  │   └─ 调用 RecordAuthResponse(..., MFAMethodOTP, ...)
  │          ↓ 进入通用 MFA 判断
```

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
│   ├─ MFA.Rule == "" → 默认返回 true（全体启用）
│   └─ MFA.Rule != "" → 用 search.FilterData 执行规则
│                         出错时返回 true（宁可要求 MFA，也不放松）
│   └─ 不需要 → return "", nil
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

### 4.3 `wantsMFA`：按规则筛选用户

```go
func wantsMFA(e *RequestEvent, record *Record) (bool, error)
```

- 空规则 → `true`（所有用户强制 MFA）
- 有规则时构造查询：`SELECT 1 FROM {collection} WHERE id=? AND ({rule})`
- 规则解析/执行**出错时默认返回 true**（安全优先：失败则收紧）

---

## 5. 失败处理与安全防护

### 5.1 OTP 失败处理
| 场景 | 行为 |
|------|------|
| OTP ID 不存在 / 已过期 / 密码错误 | 统一返回 `400 Invalid or expired OTP`，不区分原因，防枚举 |
| 速率限制触发（180s > 5 次） | 返回 `429 Too many attempts` |
| 邮件发送失败 | 删除已创建的 OTP 记录 + 打 Error 日志 |
| 找不到邮箱对应用户 | 返回假的 `otpId`（200 OK）+ 打日志，防邮箱枚举 |

### 5.2 MFA 失败处理
| 场景 | 行为 |
|------|------|
| MFA 记录不存在 / 过期 | 删除（如存在）+ `400 Invalid or expired MFA session` |
| MFA 不属于当前用户 / 集合 | `400 Invalid MFA session` |
| 二次认证用了和首次相同的方式 | `400 A different authentication method is required` |
| MFA.Rule 执行报错 | 视为需要 MFA，不跳过 |

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
┌──────────┐                    ┌────────────┐                    ┌──────────┐
│  Client  │                    │ PocketBase │                    │  Email   │
└────┬─────┘                    └─────┬──────┘                    └────┬─────┘
     │                                 │                              │
     │ 1. POST auth/otp-request        │                              │
     │    { email: "u@x.com" }         │                              │
     │────────────────────────────────>│                              │
     │                                 │ 生成 OTP（8 位数字）          │
     │                                 │ 保存 _otps                    │
     │                                 │─────────────────────────────>│ 发邮件
     │         200 { otpId: "abc" }    │                              │
     │<────────────────────────────────│                              │
     │                                 │                              │
     │ 2. POST auth/otp                │                              │
     │    { otpId: "abc", password }   │                              │
     │────────────────────────────────>│                              │
     │                                 │ 校验 OTP OK                  │
     │                                 │ 调用 RecordAuthResponse      │
     │                                 │   → checkMFA(OTP)            │
     │                                 │   → 首次认证，建 MFA 记录     │
     │    401 { mfaId: "mfa123" }      │                              │
     │<────────────────────────────────│                              │
     │                                 │                              │
     │ 3. POST auth-with-password      │                              │
     │    { identity, password,        │                              │
     │      mfaId: "mfa123" }          │                              │
     │────────────────────────────────>│                              │
     │                                 │ 校验密码 OK                  │
     │                                 │ 调用 RecordAuthResponse      │
     │                                 │   → checkMFA(Password, mfa123)│
     │                                 │     - MFA.Method=otp != password ✓│
     │                                 │     - 删除 MFA 记录           │
     │         200 { token, record }   │                              │
     │<────────────────────────────────│                              │
```

---

## 7. 关键文件索引

| 功能 | 文件 |
|------|------|
| MFA 模型定义 | [core/mfa_model.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/core/mfa_model.go) |
| MFA 查询 / 清理 | [core/mfa_query.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/core/mfa_query.go) |
| OTP 模型定义 | [core/otp_model.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/core/otp_model.go) |
| OTP 查询 / 清理 | [core/otp_query.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/core/otp_query.go) |
| 集合 MFA/OTP 配置 | [core/collection_model_auth_options.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/core/collection_model_auth_options.go) |
| checkMFA / wantsMFA | [apis/record_helpers.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/apis/record_helpers.go#L146-L256) |
| 密码认证（入口） | [apis/record_auth_with_password.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/apis/record_auth_with_password.go) |
| 请求 OTP | [apis/record_auth_otp_request.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/apis/record_auth_otp_request.go) |
| OTP 登录 | [apis/record_auth_with_otp.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/apis/record_auth_with_otp.go) |
| OAuth2 登录 | [apis/record_auth_with_oauth2.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/apis/record_auth_with_oauth2.go) |
| 用户密码方法 | [core/record_model_auth.go](file:///d:/fz/0601/solo-dogfeeding/code/157-pocketbase/core/record_model_auth.go) |
