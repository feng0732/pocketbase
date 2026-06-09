# PocketBase 记录验证逻辑详解

本文档对照源代码，详细说明 PocketBase 在记录（Record）创建和更新时的字段验证流程，包括默认值初始化、关联校验、局部更新和覆盖语义。

## 一、整体调用链路

```
HTTP 请求
  ↓
apis/record_crud.go: recordCreate / recordUpdate
  ├─ 权限检查 (CreateRule / UpdateRule / ManageRule)
  ├─ recordDataFromRequest: 解析 body，应用 modifiers，处理文件上传，过滤 hidden 字段
  └─ forms/record_upsert.go: RecordUpsert
       ├─ Load(data): 将数据载入 Record（保护 hidden 字段）
       ├─ Submit()
       │    ├─ validateFormFields(): Auth 集合的 password/email/verified 校验
       │    └─ app.SaveWithContext(record)
       │         ↓
       │    core/db.go: save() → create() / update()
       │         ├─ OnModelCreate / OnModelUpdate Hook
       │         │    └─ OnRecordCreate / OnRecordUpdate (callFieldInterceptors)
       │         ├─ ValidateWithContext: 触发 OnModelValidate → OnRecordValidate
       │         │    └─ callFieldInterceptors(validate)
       │         │         └─ onRecordValidate: 遍历所有字段调用 field.ValidateValue()
       │         └─ OnModelCreateExecute / OnModelUpdateExecute
       │              └─ callFieldInterceptors(createExecute/updateExecute)
       │                   ├─ AutodateField: 自动设置时间
       │                   ├─ FileField: 上传文件
       │                   ├─ PasswordField: 清除明文密码
       │                   └─ onRecordSaveExecute: Auth 跨集合 ID 唯一性、tokenKey 刷新
       └─ AfterCreateSuccess / AfterUpdateSuccess
            └─ FileField: 删除被替换的旧文件
```

关键入口代码：
- API 层：[record_crud.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/apis/record_crud.go#L209-L529)
- 表单层：[record_upsert.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/forms/record_upsert.go#L283-L292)
- 核心保存：[db.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/db.go#L265-L448)
- 验证触发：[record_model.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/record_model.go#L1413-L1427)

---

## 二、默认值初始化

### 2.1 初始化时机

记录的默认值在 `NewRecord(collection)` 时通过每个字段的 `PrepareValue(record, nil)` 方法设置到 `originalData` 中。

关键代码：[record_model.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/record_model.go#L538-L561)

```go
func NewRecord(collection *Collection) *Record {
    record := &Record{...}
    for _, field := range collection.Fields {
        fieldName = field.GetName()
        if fieldName == FieldNameId {
            continue
        }
        value, _ := field.PrepareValue(record, nil)  // 传 nil 获得零值
        record.originalData[fieldName] = value
    }
    return record
}
```

### 2.2 各字段类型的零值

| 字段类型 | 零值 | 代码位置 |
|---------|------|---------|
| text (TextField) | `""` (空字符串) | [field_text.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_text.go#L174-L176) |
| number (NumberField) | `0` (float64) | [field_number.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_number.go#L129-L131) |
| bool (BoolField) | `false` | [field_bool.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_bool.go#L103-L106) |
| date (DateField) | 零值 `types.DateTime` | 类似 text |
| autodate (AutodateField) | 零值 `types.DateTime` | [field_autodate.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_autodate.go#L112-L116) |
| email (EmailField) | `""` | 类似 text |
| url (UrlField) | `""` | 类似 text |
| editor (EditorField) | `""` | 类似 text |
| json (JsonField) | `nil` / 对应类型零值 | - |
| geo_point (GeoPointField) | 零值 `types.GeoPoint` | - |
| password (PasswordField) | `&PasswordFieldValue{Hash: ""}` (空结构体) | [field_password.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_password.go#L157-L161) |
| relation (RelationField) | 单选: `""`, 多选: `[]string{}` | [field_relation.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_relation.go#L165-L180) |
| select (SelectField) | 单选: `""`, 多选: `[]string{}` | [field_select.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_select.go#L152-L167) |
| file (FileField) | 单选: `""`, 多选: `[]string{}` | [field_file.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_file.go#L202-L204) |

### 2.3 特殊自动默认值

#### TextField 的 AutogeneratePattern

当 `TextField.AutogeneratePattern` 非空且新记录该字段为空时，在 `InterceptorActionValidate` 和 `InterceptorActionCreate` 阶段自动生成随机值。

代码：[field_text.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_text.go#L348-L369)

```go
func (f *TextField) Intercept(ctx, app, record, actionName, actionFunc) error {
    switch actionName {
    case InterceptorActionValidate, InterceptorActionCreate:
        if f.AutogeneratePattern != "" && f.hasZeroValue(record) && record.IsNew() {
            v, _ := security.RandomStringByRegex(f.AutogeneratePattern)
            record.SetRaw(f.Name, v)
        }
    }
    return actionFunc()
}
```

#### AutodateField 的自动时间戳

在 `InterceptorActionCreateExecute`（创建）和 `InterceptorActionUpdateExecute`（更新）阶段，根据 `OnCreate` / `OnUpdate` 配置自动设置当前时间。

代码：[field_autodate.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_autodate.go#L164-L206)

```go
func (f *AutodateField) Intercept(...) error {
    switch actionName {
    case InterceptorActionCreateExecute:
        if f.OnCreate && 记录值未被手动修改 {
            record.SetRaw(f.Name, types.NowDateTime())
        }
    case InterceptorActionUpdateExecute:
        if f.OnUpdate && 记录值未被手动修改 {
            record.SetRaw(f.Name, types.NowDateTime())
        }
    }
}
```

注意：AutodateField 的 `FindSetter` 对字段名本身返回 `noopSetter`，即通过 `record.Set("created", xxx)` 无效，必须用 `record.SetRaw` 才能手动修改。

代码：[field_autodate.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_autodate.go#L153-L162)

---

## 三、字段值的设置与覆盖语义

### 3.1 Set / SetRaw / SetIfFieldExists 的区别

| 方法 | 作用范围 | 是否走字段规范化 | 代码位置 |
|-----|---------|--------------|---------|
| `SetRaw(key, value)` | 任意 key | 否，直接存储 | [record_model.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/record_model.go#L851-L857) |
| `SetIfFieldExists(key, value)` | 仅集合已知字段 | 是，走 SetterFinder 或 PrepareValue | [record_model.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/record_model.go#L867-L887) |
| `Set(key, value)` | 任意 key | 已知字段走规范化，自定义字段直接存储 | [record_model.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/record_model.go#L893-L904) |
| `Load(data)` | 批量调用 `Set` | 同上 | [record_model.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/record_model.go#L940-L945) |

#### SetIfFieldExists 的详细逻辑

```go
func (m *Record) SetIfFieldExists(key string, value any) Field {
    for _, field := range m.Collection().Fields {
        // 1. 先尝试 SetterFinder（支持 modifiers，如 "field+" / "+field" / "field-"）
        if ff, ok := field.(SetterFinder); ok {
            setter := ff.FindSetter(key)
            if setter != nil {
                setter(m, value)
                return field
            }
        }
        // 2. fallback: 如果 key 精确匹配字段名，用 PrepareValue 规范化
        if key == field.GetName() {
            value, _ = field.PrepareValue(m, value)
            m.SetRaw(key, value)
            return field
        }
    }
    return nil
}
```

### 3.2 RecordUpsert.Load 的字段保护机制

在 `forms/record_upsert.go` 的 `Load` 方法中，有针对 hidden 字段的特殊保护：

代码：[record_upsert.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/forms/record_upsert.go#L88-L126)

```go
func (form *RecordUpsert) Load(data map[string]any) {
    for k, v := range data {
        field := form.record.SetIfFieldExists(k, v)
        // 非 superuser 权限下，hidden 字段的值会被还原为 original
        if form.accessLevel != accessLevelSuperuser && 
           field != nil && 
           field.GetHidden() && 
           (!isAuth || field.GetName() != core.FieldNamePassword) {
            form.record.SetRaw(field.GetName(), 
                form.record.Original().GetRaw(field.GetName()))
        }
    }
}
```

**覆盖规则总结：**
- **Superuser 权限**：所有字段（包括 hidden）都可直接被覆盖
- **Manager 或 Default 权限**：hidden 字段提交的值会被忽略，自动恢复为原始值；唯一例外是 Auth 集合的 `password` 字段（即使 hidden 也允许修改）
- **API 层过滤**：在 `recordDataFromRequest` 中，非 superuser 请求的 hidden 字段在进入 Load 前已被从 data 中删除（password 除外）

代码：[record_crud.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/apis/record_crud.go#L673-L685)

---

## 四、局部更新（Modifiers 机制）

PocketBase 支持通过字段名的特殊后缀/前缀对多值字段和数字字段进行局部修改，而不需要先读取再整体写回。

### 4.1 支持 Modifiers 的字段类型

| 字段类型 | Modifier | 语义 | 代码位置 |
|---------|----------|------|---------|
| NumberField | `field+` | 加上指定值 | [field_number.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_number.go#L217-L221) |
| NumberField | `field-` | 减去指定值 | [field_number.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_number.go#L223-L227) |
| RelationField | `+field` | 前置追加 id | [field_relation.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_relation.go#L335-L344) |
| RelationField | `field+` | 后置追加 id | [field_relation.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_relation.go#L324-L333) |
| RelationField | `field-` | 删除指定 id | [field_relation.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_relation.go#L346-L355) |
| SelectField | `+field` / `field+` / `field-` | 同上 | [field_select.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_select.go#L229-L280) |
| FileField | `+field` / `field+` / `field-` | 同上 | [field_file.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_file.go#L661-L712) |
| TextField | `field:autogenerate` | 自动生成随机值 | [field_text.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_text.go#L383-L393) |

### 4.2 Modifiers 的解析时机

在 API 层 `recordDataFromRequest` 中调用 `record.ReplaceModifiers(info.Body)` 将修饰符键转换为实际字段值：

代码：[record_model.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/record_model.go#L1360-L1390)

```go
func (m *Record) ReplaceModifiers(data map[string]any) map[string]any {
    // 按 key 长度排序，确保先处理短的键
    sort.SliceStable(sortedDataKeys, func(i, j int) bool {
        return len(sortedDataKeys[i]) < len(sortedDataKeys[j])
    })
    for _, k := range sortedDataKeys {
        field := recordCopy.SetIfFieldExists(k, data[k])
        if field != nil {
            delete(dataCopy, k)  // 删除修饰符键
            dataCopy[field.GetName()] = recordCopy.Get(field.GetName())  // 存入规范化后的真实值
        }
    }
}
```

注意：update 请求需要先从数据库读取已有记录，这样 modifier 才能基于旧值进行增量计算。

### 4.3 PATCH + IgnoreUnchangedFields

Record 提供 `IgnoreUnchangedFields(state)` 方法，开启后 UPDATE SQL 只包含实际被修改的字段（通过与 `Original()` 比较）。

代码：[record_model.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/record_model.go#L1096-L1124)

```go
func (m *Record) DBExport(app App) (map[string]any, error) {
    result, _ := m.dbExport()
    if !m.IsNew() && m.ignoreUnchangedFields {
        oldResult, _ := m.Original().dbExport()
        for oldK, oldV := range oldResult {
            if oldK == idColumn { continue }
            newV, ok := result[oldK]
            if ok && areValuesEqual(newV, oldV) {
                delete(result, oldK)  // 删除未变化的字段
            }
        }
    }
    return result, nil
}
```

---

## 五、字段级校验（ValidateValue）

每个字段类型通过 `ValidateValue(ctx, app, record)` 方法进行值校验，在 `onRecordValidate` 中被统一调用。

代码：[record_model.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/record_model.go#L1413-L1427)

```go
func onRecordValidate(e *RecordEvent) error {
    errs := validation.Errors{}
    for _, f := range e.Record.Collection().Fields {
        if err := f.ValidateValue(e.Context, e.App, e.Record); err != nil {
            errs[f.GetName()] = err
        }
    }
    if len(errs) > 0 { return errs }
    return e.Next()
}
```

### 5.1 通用校验项（多数字段）

| 校验项 | 说明 |
|-------|------|
| Required | 值不能为空（各字段对"空"定义不同，见下） |
| 类型匹配 | 内部存储类型必须符合预期（如 number 是 float64） |
| 字段特有约束 | min/max、pattern、length 等 |

各字段对"空值"的定义：
- **text/email/url/editor/date/autodate**: `""` (空字符串)
- **number**: `0`
- **bool**: `false`（Required 时要求必须为 true）
- **relation/select/file 单选**: `""`
- **relation/select/file 多选**: `[]` 空切片
- **password**: `Hash == ""`

### 5.2 RelationField 关联校验

代码：[field_relation.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_relation.go#L198-L237)

```go
func (f *RelationField) ValidateValue(...) error {
    ids := list.ToUniqueStringSlice(record.GetRaw(f.Name))
    // 1. Required 检查
    // 2. MinSelect 检查（如配置）
    // 3. MaxSelect 检查（如配置）
    // 4. 数据库存在性检查：查询关联集合确认所有 id 都存在
    relCollection, _ := app.FindCachedCollectionByNameOrId(f.CollectionId)
    app.ConcurrentDB().Select("count(*)").From(relCollection.Name).
        AndWhere(dbx.In("id", list.ToInterfaceSlice(ids)...)).Row(&total)
    if total != len(ids) {
        return validation.NewError("validation_missing_rel_records", ...)
    }
}
```

关键点：
- 关联记录的存在性通过 `count(*)` 聚合查询校验
- 多值时要求全部 id 都存在（total == len(ids)）
- 关联目标集合不可用会返回 `validation_missing_rel_collection`

### 5.3 FileField 文件校验

代码：[field_file.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_file.go#L240-L307)

```go
func (f *FileField) ValidateValue(...) error {
    files := f.toSliceValue(record.GetRaw(f.Name))
    // 1. Required 检查
    // 2. 禁止新增纯字符串文件名（新文件必须是 *filesystem.File 对象）
    //    对比 original 中的旧文件名，找出新增的字符串，全部视为非法
    // 3. MaxSelect 检查
    // 4. 对每个待上传的 *filesystem.File：
    //    - 文件名长度 1-150
    //    - 文件名正则校验（不能以 . 开头、不能含 / \）
    //    - 文件大小 <= MaxSize（默认 5MB）
    //    - MIME 类型在白名单中（如配置）
}
```

关键点：
- **防混淆检查**：已存在的旧文件名（字符串）可以保留，但新增值必须是 `*filesystem.File` 对象，不允许直接提交任意字符串作为新文件名
- 上传大小默认 5MB（`DefaultFileFieldMaxSize`）

### 5.4 PasswordField 密码校验

代码：[field_password.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_password.go#L164-L211)

```go
func (f *PasswordField) ValidateValue(...) error {
    fp := record.GetRaw(f.Name).(*PasswordFieldValue)
    // 1. bcrypt 哈希错误检查
    // 2. Required 时 Hash 不能为空
    // 3. 仅当 Plain 非空时校验：
    //    - Min 长度（按 rune 计算）
    //    - Max 长度（默认 71，bcrypt 限制）
    //    - Pattern 正则（如配置）
}
```

关键点：
- 校验只针对明文密码（`Plain`），Hash 本身不校验长度/格式
- 使用 `record.Set("password", "123456")` 时会立即在 Setter 中哈希，`Plain` 和 `Hash` 都存入 `PasswordFieldValue`
- 校验通过后，在 `AfterCreate` / `AfterUpdate` 阶段 `Plain` 会被清空

### 5.5 TextField (含主键) 校验

代码：[field_text.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/field_text.go#L179-L220)

**主键（PrimaryKey）特殊校验：**
- 更新时不允许改变主键值
- 非默认 id pattern 时，大小写不敏感地检查重复
- 禁止包含特殊字符：`. / \ | " ' \` < > : ? * % $ \0 \t \n \r 空格`
- 禁止使用 Windows 保留名：CON, PRN, AUX, NUL, COM1-9, LPT1-9

### 5.6 Auth 记录的表单级额外校验

代码：[record_upsert.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/forms/record_upsert.go#L128-L196)

| 字段 | 校验规则 |
|-----|---------|
| email | 非 Manager/Superuser 更新时不能直接修改（必须等于 original.Email） |
| verified | 非 Manager/Superuser 时不能修改（必须等于 original.Verified） |
| password | 新建 / passwordConfirm 非空 / oldPassword 非空 时必填 |
| passwordConfirm | 同上条件时必填；且必须等于 password |
| oldPassword | 更新 + 无 Manager 权限 + 正在改密码 时必填，且必须匹配原密码哈希 |

### 5.7 Auth 跨集合 ID 唯一性

在 `onRecordSaveExecute` 中，Auth 记录的 ID 会与其他所有 Auth 集合做跨集合唯一性检查，避免多 Auth 集合配置规则错误时产生 ID 冲突。

代码：[record_model.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/record_model.go#L1429-L1474)

---

## 六、关联字段的级联删除行为

虽然不属于创建/更新验证，但与关联校验紧密相关，删除记录时对引用的处理：

代码：[record_model.go](file:///d:/fz/0601/solo-dogfeeding/code/151-pocketbase/core/record_model.go#L1576-L1621)

| 引用字段配置 | 被引用记录删除时的行为 |
|------------|---------------------|
| `CascadeDelete = true` 且 引用清空 | 级联删除引用记录 |
| `Required = true` 且 引用将清空 | 阻止删除，返回错误 |
| 其他情况 | 仅从引用字段中移除该 id，保存引用记录（SaveNoValidate，不触发校验避免其他已删引用导致的失败） |

---

## 七、完整校验时序（以 Update 为例）

```
1. APIs: recordUpdate
   ├─ FindRecordById: 从 DB 读取原始记录
   ├─ recordDataFromRequest
   │   ├─ ReplaceModifiers: "field+" → 实际值（基于原始记录）
   │   ├─ 合并上传文件
   │   └─ 删除非 superuser 的 hidden 字段（password 除外）
   └─ 根据 ManageRule 授予 ManagerAccess（如果有权限）

2. Forms: RecordUpsert.Load
   └─ SetIfFieldExists 逐一赋值，非 superuser 的 hidden 字段回滚为 original

3. Forms: RecordUpsert.Submit
   └─ validateFormFields (仅 Auth): email/verified/password 权限校验
   └─ app.SaveWithContext

4. Core: OnModelUpdate → OnRecordUpdate (callFieldInterceptors)
   ├─ TextField: AutogeneratePattern（仅 IsNew）
   └─ ... 其他字段的 Update interceptor

5. Core: OnModelValidate → OnRecordValidate (callFieldInterceptors)
   ├─ onRecordValidate: 遍历所有字段 ValidateValue()
   │   ├─ Required / Min / Max / Pattern 等
   │   ├─ RelationField: 关联记录存在性
   │   ├─ FileField: 新增文件合法性、大小、MIME
   │   └─ TextField(主键): ID 不可变性 + 格式
   └─ ... 其他字段的 Validate interceptor

6. Core: OnModelUpdateExecute → OnRecordUpdateExecute (callFieldInterceptors)
   ├─ AutodateField: OnUpdate=true 则设置当前时间
   ├─ FileField: 先上传文件，失败则回滚
   └─ onRecordSaveExecute
       ├─ Auth: 密码/邮箱变更时刷新 TokenKey
       ├─ Auth: 跨集合 ID 唯一性检查
       └─ DBExport → UPDATE SQL（IgnoreUnchangedFields 时只含变化字段）

7. Core: OnModelAfterUpdateSuccess → OnRecordAfterUpdateSuccess
   ├─ PasswordField: 清除 Plain 明文
   └─ FileField: 删除被替换的旧文件
```
