# PocketBase 记录验证逻辑详解

本文档对照源代码，按模块梳理 PocketBase 在 Record 创建与更新时的字段验证流程，覆盖默认值、关联校验、局部更新和覆盖语义。所有结论均附带仓库相对路径 + 行号的代码依据。

---

## 一、整体调用链路

```
HTTP 请求
  ↓
apis/record_crud.go: recordCreate / recordUpdate
  ├─ 权限检查 (CreateRule / UpdateRule / ManageRule)
  ├─ recordDataFromRequest: 解析 body → 应用 modifiers → 合并上传文件 → 删除非 superuser 的 hidden 字段
  └─ forms/record_upsert.go: RecordUpsert.Load → Submit
       ├─ validateFormFields()        (仅 Auth: email/verified/password 的权限校验)
       └─ app.SaveWithContext(record)
            ↓
       core/db.go: save() → create() / update()
            ├─ OnModelCreate / OnModelUpdate → OnRecordCreate / OnRecordUpdate (callFieldInterceptors)
            ├─ ValidateWithContext → OnModelValidate → OnRecordValidate (callFieldInterceptors)
            │   └─ onRecordValidate: 遍历 collection.Fields 调用 field.ValidateValue()
            └─ OnModelCreateExecute / OnModelUpdateExecute → OnRecordCreateExecute / OnRecordUpdateExecute
                └─ callFieldInterceptors:
                     ├─ AutodateField: 自动设置时间戳
                     ├─ FileField: 执行文件上传
                     ├─ onRecordSaveExecute: Auth 跨集合 ID 唯一性、TokenKey 刷新
                     └─ DBExport → INSERT/UPDATE SQL
            ↓
       OnModelAfterCreateSuccess / OnModelAfterUpdateSuccess
            ├─ PasswordField: 清空明文密码
            └─ FileField: 删除被替换的旧文件
```

### 关键入口代码依据

| 环节 | 代码位置 |
|------|---------|
| API 入口 | [apis/record_crud.go#L209-L529](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/apis/record_crud.go#L209-L529) |
| Submit + app.SaveWithContext | [forms/record_upsert.go#L283-L292](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/forms/record_upsert.go#L283-L292) |
| save() 路由到 create/update | [core/db.go#L265-L271](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/db.go#L265-L271) |
| create() 触发 Hook 顺序 | [core/db.go#L273-L363](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/db.go#L273-L363) |
| onRecordValidate 遍历字段 | [core/record_model.go#L1413-L1427](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/record_model.go#L1413-L1427) |
| onRecordSaveExecute: Auth 跨集合 ID + TokenKey | [core/record_model.go#L1429-L1474](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/record_model.go#L1429-L1474) |

---

## 二、默认值初始化

### 2.1 初始化时机与入口

`NewRecord(collection)` 构造记录时，遍历集合所有字段，对每个字段调用 `PrepareValue(record, nil)`，把返回的零值填入 `originalData`。

代码依据：[core/record_model.go#L538-L561](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/record_model.go#L538-L561)

```go
func NewRecord(collection *Collection) *Record {
    record := &Record{...}
    for _, field := range collection.Fields {
        fieldName = field.GetName()
        if fieldName == FieldNameId { continue }
        value, _ := field.PrepareValue(record, nil)  // 传 nil → 零值
        record.originalData[fieldName] = value
    }
    return record
}
```

### 2.2 各字段的零值

| 字段类型 | 零值 | 代码依据 |
|---------|------|---------|
| text (TextField) | `""` | [core/field_text.go#L174-L176](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_text.go#L174-L176)（`cast.ToString(nil) → ""`） |
| number (NumberField) | `0`（float64） | [core/field_number.go#L128-L131](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_number.go#L128-L131) `cast.ToFloat64(nil) → 0` |
| bool (BoolField) | `false` | [core/field_bool.go#L103-L106](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_bool.go#L103-L106) `cast.ToBool(nil) → false` |
| date/autodate | 零值 `types.DateTime` | 类似 text，经 normalize 后为零值时间 |
| email/url/editor/json/geo_point | 对应类型零值 | 均走类似 `cast.ToXxx(nil)` 归一化 |
| password | `&PasswordFieldValue{Hash:"", Plain:""}` 空结构体 | [core/field_password.go#L157-L161](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_password.go#L157-L161) |
| relation 单选 / select 单选 / file 单选 | `""` | [core/field_relation.go#L164-L180](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_relation.go#L164-L180) `normalizeValue(nil)` 对单选返回 `""` |
| relation 多选 / select 多选 / file 多选 | `[]string{}` | 同上，多选返回空切片 |

### 2.3 自动生成默认值

#### (1) TextField.AutogeneratePattern

当 `TextField.AutogeneratePattern != ""` 且当前是新记录且字段为空时，在 `InterceptorActionValidate` 或 `InterceptorActionCreate` 阶段自动生成符合正则的随机字符串。

代码依据：[core/field_text.go#L348-L369](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_text.go#L348-L369)

```go
case InterceptorActionValidate, InterceptorActionCreate:
    if f.AutogeneratePattern != "" && f.hasZeroValue(record) && record.IsNew() {
        v, _ := security.RandomStringByRegex(f.AutogeneratePattern)
        record.SetRaw(f.Name, v)
    }
```

另外也支持通过 modifier `field:autogenerate` 触发同样的自动生成（拼接在 raw 值后面）：

代码依据：[core/field_text.go#L376-L397](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_text.go#L376-L397)

#### (2) AutodateField 自动时间戳

- **创建阶段**：`OnCreate=true` 时在 `InterceptorActionCreateExecute` 设置 `types.NowDateTime()`
- **更新阶段**：`OnUpdate=true` 时在 `InterceptorActionUpdateExecute` 设置 `types.NowDateTime()`
- 判断是否被手动修改：比较当前值和 `getLastKnownValue(record)`（优先看 `_autodate_lastKnown_prefix+name`，否则看 `Original()`），相等即表示未被手动改过，应自动更新。

代码依据：[core/field_autodate.go#L164-L206](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_autodate.go#L164-L206)

**关键点**：`AutodateField.FindSetter` 对字段名返回 `noopSetter`，即通过 `record.Set("created", xxx)` 无效，必须用 `SetRaw` 才能手动修改。

代码依据：[core/field_autodate.go#L153-L162](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_autodate.go#L153-L162)

```go
func (f *AutodateField) FindSetter(key string) SetterFunc {
    switch key {
    case f.Name:
        return noopSetter  // record.Set("created", ...) 被静默忽略
    default:
        return nil
    }
}
```

---

## 三、字段值设置与覆盖语义

### 3.1 Set / SetRaw / SetIfFieldExists 三者的区别

| 方法 | 是否走字段规范化 | 是否仅作用于已知字段 | 代码依据 |
|-----|--------------|-------------------|---------|
| `SetRaw(key, value)` | 否，直接存储 | 否，任意 key | [core/record_model.go#L851-L857](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/record_model.go#L851-L857) |
| `SetIfFieldExists(key, value)` | 是（先走 SetterFinder，fallback 到 PrepareValue） | 是 | [core/record_model.go#L867-L887](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/record_model.go#L867-L887) |
| `Set(key, value)` | 已知字段走规范化，自定义字段直接存 | 否 | [core/record_model.go#L893-L904](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/record_model.go#L893-L904) |
| `Load(data)` | 循环调用 `Set` | 否 | [core/record_model.go#L940-L945](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/record_model.go#L940-L945) |

### 3.2 SetIfFieldExists 详细执行流程

代码依据：[core/record_model.go#L867-L887](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/record_model.go#L867-L887)

```go
func (m *Record) SetIfFieldExists(key string, value any) Field {
    for _, field := range m.Collection().Fields {
        // 步骤 1: 先尝试 SetterFinder 接口（支持 modifiers，如 "field+" / "+field" / "field-"）
        if ff, ok := field.(SetterFinder); ok {
            setter := ff.FindSetter(key)
            if setter != nil {
                setter(m, value)
                return field
            }
        }
        // 步骤 2: fallback —— key 精确匹配字段名时，走 PrepareValue 规范化 + SetRaw
        if key == field.GetName() {
            value, _ = field.PrepareValue(m, value)
            m.SetRaw(key, value)
            return field
        }
    }
    return nil
}
```

**覆盖语义总结**：
- **SetterFinder 优先于 PrepareValue**：当 key 是 modifier（如 `tags+`）时，走 modifier setter，不走 PrepareValue；当 key 就是字段名时，字段的 `FindSetter(name)` 通常返回 `setValue`，内部会做归一化然后 `SetRaw`。
- **自定义字段**：`SetIfFieldExists` 返回 nil，`Set` 会 fallback 到 `SetRaw` 原样写入。

### 3.3 RecordUpsert.Load 的 hidden 字段保护机制

在表单层 `Load` 方法中，非 Superuser 提交的 hidden 字段值会被立即回滚为 `Original()` 的值。Auth 集合的 `password` 是唯一例外。

代码依据：[forms/record_upsert.go#L88-L126](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/forms/record_upsert.go#L88-L126)

```go
field := form.record.SetIfFieldExists(k, v)
// 条件: 非 superuser && 是字段 && 字段 hidden && (非Auth 或 不是 password)
if form.accessLevel != accessLevelSuperuser &&
   field != nil &&
   field.GetHidden() &&
   (!isAuth || field.GetName() != core.FieldNamePassword) {
    // 回滚为原始值
    form.record.SetRaw(field.GetName(), form.record.Original().GetRaw(field.GetName()))
}
```

此外在 API 层 `recordDataFromRequest` 中，非 superuser 请求的 hidden 字段在进入 Load 之前已被从 data 中删除（password 除外），形成双重保护。

代码依据：[apis/record_crud.go#L673-L685](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/apis/record_crud.go#L673-L685)

```go
if !info.HasSuperuserAuth() {
    for _, f := range record.Collection().Fields {
        if f.GetHidden() {
            if isAuth && f.GetName() == core.FieldNamePassword { continue }
            delete(result, f.GetName())
        }
    }
}
```

---

## 四、局部更新（Modifiers 机制）

### 4.1 解析时机

在 API 层 `recordDataFromRequest` 中调用 `record.ReplaceModifiers(info.Body)` 将带 modifier 的 key（如 `tags+`、`count-`）解析为实际字段值。处理完文件上传后还会再调用一次 `ReplaceModifiers`。

代码依据：[apis/record_crud.go#L631-L688](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/apis/record_crud.go#L631-L688)（见 L638 和 L668）

`ReplaceModifiers` 按 **key 长度由短到长** 排序，确保同一字段多个 modifier 时行为一致；基于 record 的当前值（需要先从 DB 读原记录）做增量计算。

代码依据：[core/record_model.go#L1360-L1390](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/record_model.go#L1360-L1390)

```go
sort.SliceStable(sortedDataKeys, func(i, j int) bool {
    return len(sortedDataKeys[i]) < len(sortedDataKeys[j])
})
for _, k := range sortedDataKeys {
    field := recordCopy.SetIfFieldExists(k, data[k])
    if field != nil {
        delete(dataCopy, k)                              // 删除 modifier 键
        dataCopy[field.GetName()] = recordCopy.Get(field.GetName()) // 存入规范化后的实际值
    }
}
```

### 4.2 各字段支持的 Modifier 汇总

| 字段类型 | Modifier | 语义 | 代码依据 |
|---------|----------|------|---------|
| NumberField | `field+` | 加上指定值（`val + cast.ToFloat64(raw)`） | [core/field_number.go#L200-L227](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_number.go#L200-L227) |
| NumberField | `field-` | 减去指定值（`val - cast.ToFloat64(raw)`） | 同上 |
| RelationField | `+field` | 前置追加 ID | [core/field_relation.go#L305-L355](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_relation.go#L305-L355) |
| RelationField | `field+` | 后置追加 ID | 同上 |
| RelationField | `field-` | 删除指定 ID（`list.SubtractSlice`） | 同上 |
| SelectField | `+field` / `field+` / `field-` | 同上 | [core/field_select.go#L229-L280](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_select.go#L229-L280) |
| FileField | `+field` / `field+` / `field-` | 同上（注意新增文件必须是 `*filesystem.File`） | [core/field_file.go#L661-L712](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_file.go#L661-L712) |
| TextField | `field:autogenerate` | 自动生成 `AutogeneratePattern` 随机值拼接到 raw | [core/field_text.go#L376-L397](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_text.go#L376-L397) |

### 4.3 PATCH + IgnoreUnchangedFields

Record 提供 `IgnoreUnchangedFields(state)` 方法，开启后 `DBExport` 只包含实际被修改的字段（与 `Original()` 做值相等比较后删掉未变化的 key，`id` 列除外）。

代码依据：[core/record_model.go#L1096-L1124](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/record_model.go#L1096-L1124)

```go
if !m.IsNew() && m.ignoreUnchangedFields {
    oldResult, _ := m.Original().dbExport()
    for oldK, oldV := range oldResult {
        if oldK == idColumn { continue }
        newV, ok := result[oldK]
        if ok && areValuesEqual(newV, oldV) {
            delete(result, oldK)
        }
    }
}
```

---

## 五、字段级校验（ValidateValue）

所有字段的 `ValidateValue(ctx, app, record)` 在 `onRecordValidate` 中被统一调用；每个字段校验失败的 error 被聚合到 `validation.Errors` map，key 为字段名。

代码依据：[core/record_model.go#L1413-L1427](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/record_model.go#L1413-L1427)

### 5.1 通用校验项

几乎所有字段都包含：

| 校验项 | 说明 |
|-------|------|
| 类型检查 | 内部类型断言（如 `record.GetRaw(f.Name).(float64)`），失败返回 `validators.ErrUnsupportedValueType` |
| Required | 字段值是否为"空"，各字段对"空"定义不同（见下表） |
| 字段特有约束 | min/max、pattern、length、MinSelect/MaxSelect、MIME 等 |

各字段对"空值"的定义：
- **text/email/url/editor/date/autodate**: `""`
- **number**: `0`
- **bool**: `false`（Required 时要求必须为 true）
- **relation/select/file 单选**: `""`
- **relation/select/file 多选**: `[]string{}` 空切片
- **password**: `Hash == ""`

### 5.2 RelationField 关联校验

代码依据：[core/field_relation.go#L197-L237](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_relation.go#L197-L237)

```
1. ids = ToUniqueStringSlice(value)
2. len(ids)==0 && Required  →  ErrRequired
3. MinSelect>0 && len(ids)<MinSelect  →  "validation_not_enough_values"
4. len(ids)>max(MaxSelect,1)          →  "validation_too_many_values"
5. 查关联集合 count(*)，total != len(ids)  →  "validation_missing_rel_records"
6. 关联集合查不到                     →  "validation_missing_rel_collection"
```

**关键点**：
- 去重后的所有 ID 都必须在目标集合中存在（`total == len(ids)`）
- 使用 `app.ConcurrentDB()` 读从库做存在性检查

### 5.3 FileField 文件校验

代码依据：[core/field_file.go#L240-L307](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_file.go#L240-L307)

```
1. files = toSliceValue(value)
2. len(files)==0 && Required  →  ErrRequired
3. 防混淆检查：对比 original 中的旧纯字符串文件名
   addedStrings = 新增的纯字符串（不在 oldExistingStrings 里的）
   len(addedStrings) > 0  →  "validation_invalid_file"（新文件必须是 *filesystem.File）
4. len(files) > MaxSelect  →  "validation_too_many_files"
5. 对每个 *filesystem.File：
   - 文件名长度 1-150
   - 文件名正则 looseFilenameRegex（不能以 . 开头、不能含 / \）
   - 文件大小 <= maxSize()（默认 DefaultFileFieldMaxSize=5MB）
   - 如配置了 MimeTypes，校验 MIME
```

**防混淆检查是核心安全机制**：客户端不能随便提交任意字符串冒充已上传的文件名，只有 `Original()` 中已存在的文件名才能保留为字符串形式，新增的值必须是 `*filesystem.File` 对象（由 `apis/record_crud.go` 的 `extractUploadedFiles` 解析 multipart 表单得来）。

### 5.4 PasswordField 密码校验

代码依据：[core/field_password.go#L163-L211](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_password.go#L163-L211)

```
1. 值必须是 *PasswordFieldValue，否则 ErrUnsupportedValueType
2. fp.LastError != nil（通常是 bcrypt 哈希失败）→ 返回该错误
3. Required && fp.Hash == ""  →  ErrRequired
4. fp.Plain == ""  →  直接 return nil（不做长度/Pattern 校验）
5. Plain 按 rune 计算长度：
   - < Min  →  "validation_min_text_constraint"
   - > Max（默认 71，bcrypt 限制）→  "validation_max_text_constraint"
6. Pattern != "" 且不匹配  →  "validation_invalid_format"
```

**关键流程**：
- `PasswordField.FindSetter` 在 `Set` 时会立即调用 `bcrypt.GenerateFromPassword` 同时保存 `Plain` 和 `Hash` 到 `PasswordFieldValue`
  代码依据：[core/field_password.go#L276-L304](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_password.go#L276-L304)
- 校验只针对 `Plain`，对 `Hash` 本身不校验格式
- `InterceptorActionAfterCreate/Update` 成功后清空 `Plain`
  代码依据：[core/field_password.go#L249-L258](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_password.go#L249-L258)

### 5.5 TextField（含主键）校验

代码依据：[core/field_text.go#L179-L277](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_text.go#L179-L277)

**主键（PrimaryKey=true）特殊校验**：
```
1. 非新记录（更新）:
   - oldVal != newVal  →  "validation_pk_change"（主键不可变）
   - oldVal != ""     →  直接 return nil（不再校验，允许从外部系统迁移过来的旧 ID）
2. 新记录且 Pattern != defaultLowercaseRecordIdPattern:
   - COLLATE NOCASE 查重 → "validation_pk_invalid"
3. 通用校验（主键和普通字段都执行）:
   - Required / PrimaryKey → ErrRequired
   - Min / Max 长度（按 rune，Max 默认 5000）
   - Pattern 正则（如配置）
4. 主键 + 非默认 Pattern 的额外限制:
   - 禁止字符: ./\|"'`<>?:*%$ \0 \t \n \r 空格
   - 禁止 Windows 保留名: CON, PRN, AUX, NUL, COM1-9, LPT1-9
```

### 5.6 Auth 记录的表单级额外校验

`RecordUpsert.validateFormFields()` 仅对 `IsAuth()` 集合生效。

代码依据：[forms/record_upsert.go#L128-L196](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/forms/record_upsert.go#L128-L196)

| 字段 | 校验条件 | 规则 |
|-----|---------|------|
| email | 更新 + 无 Manage 权限 | 必须等于 `original.Email()`（不能直接改邮箱） |
| verified | 无 Manage 权限 | 必须等于 `original.Verified()`（不能直接改验证状态） |
| password | 新记录 OR passwordConfirm!="" OR oldPassword!="" | Required |
| passwordConfirm | 同上 | Required + 必须等于 password |
| oldPassword | 更新 + 无 Manage 权限 + 正在改密码 | Required + 必须匹配原密码哈希（`checkOldPassword`） |

### 5.7 Auth 跨集合 ID 唯一性

`onRecordSaveExecute` 中对 Auth 记录，遍历所有 Auth 集合（除自身）检查 ID 冲突，防止多 Auth 集合时 API 规则出错导致的重复 ID。

代码依据：[core/record_model.go#L1445-L1461](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/record_model.go#L1445-L1461)

同时在同一阶段检测 password/email 是否变化，如果变化且 TokenKey 未被手动更新，则 `RefreshTokenKey()` 使旧 token 失效。

代码依据：[core/record_model.go#L1437-L1442](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/record_model.go#L1437-L1442)

---

## 六、完整校验时序（以 Update 为例）

```
1. APIs: recordUpdate (apis/record_crud.go#L400)
   ├─ FindRecordById: 从 DB 读取原记录（作为 Original()）
   ├─ recordDataFromRequest (apis/record_crud.go#L631-L688)
   │   ├─ ReplaceModifiers: "field+" 等 → 实际字段值（基于原记录）
   │   ├─ 合并 multipart 上传的 *filesystem.File
   │   └─ 删除非 superuser 的 hidden 字段（password 除外）
   └─ 满足 ManageRule 则授予 ManagerAccess

2. Forms: RecordUpsert.Load (forms/record_upsert.go#L88-L126)
   ├─ SetIfFieldExists 逐一赋值
   └─ 非 superuser 的 hidden 字段回滚为 Original() （password 除外）

3. Forms: RecordUpsert.Submit (forms/record_upsert.go#L283-L292)
   ├─ validateFormFields (Auth 专属): email/verified/password/oldPassword 权限校验
   └─ app.SaveWithContext → core/db.go save()

4. Core: OnModelUpdate → OnRecordUpdate (callFieldInterceptors)
   └─ 例：TextField: AutogeneratePattern（仅 IsNew 触发）

5. Core: ValidateWithContext → OnModelValidate → OnRecordValidate
   └─ onRecordValidate (core/record_model.go#L1413-L1427)
       └─ 遍历 collection.Fields，每字段调用 ValidateValue()
           ├─ Required / Min / Max / Pattern / Length
           ├─ RelationField: 关联记录存在性 (count(*))
           ├─ FileField: 新增文件必须是 *filesystem.File，大小/MIME/文件名
           ├─ PasswordField: Plain 长度和 Pattern
           └─ TextField(主键): PK 不可变性 + 大小写不敏感查重 + 保留字/字符

6. Core: OnModelUpdateExecute → OnRecordUpdateExecute (callFieldInterceptors)
   ├─ AutodateField: OnUpdate=true 且未被手动改 → 设置 Now()
   ├─ FileField: 先上传文件，失败回滚
   └─ onRecordSaveExecute (core/record_model.go#L1429-L1474)
       ├─ Auth: 密码/邮箱变更时刷新 TokenKey
       ├─ Auth: 跨集合 ID 唯一性检查
       └─ DBExport → UPDATE SQL（IgnoreUnchangedFields 时只含变化字段）

7. Core: OnModelAfterUpdateSuccess → OnRecordAfterUpdateSuccess
   ├─ PasswordField: fp.Plain = "" (清空明文)
   └─ FileField: 删除被替换的旧文件
```

---

## 七、关联字段的级联删除（补充）

虽不属于创建/更新验证，但与关联校验相关：删除记录时对引用字段的处理。

代码依据：[core/record_model.go#L1576-L1621](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/record_model.go#L1576-L1621)

| 引用字段配置 | 被引用记录删除时的行为 |
|------------|---------------------|
| `CascadeDelete=true` 且引用清空 | 级联删除引用记录 |
| `Required=true` 且引用将清空 | 阻止删除，返回错误 |
| 其他情况 | 从引用字段中移除该 id，`SaveNoValidate` 保存引用记录（跳过校验，避免其他已删引用导致连锁失败） |
