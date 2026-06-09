# PocketBase 关系字段与级联规则代码深度分析

## 一、核心数据结构

### 1.1 RelationField 关系字段定义

关系字段定义在 [field_relation.go](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/field_relation.go) 中：

```go
type RelationField struct {
    Name          string `form:"name" json:"name"`
    Id            string `form:"id" json:"id"`
    System        bool   `form:"system" json:"system"`
    Hidden        bool   `form:"hidden" json:"hidden"`
    Presentable   bool   `form:"presentable" json:"presentable"`
    Help          string `form:"help" json:"help"`
    CollectionId  string `form:"collectionId" json:"collectionId"`  // 关联的目标集合ID
    CascadeDelete bool   `form:"cascadeDelete" json:"cascadeDelete"` // 级联删除标志
    MinSelect     int    `form:"minSelect" json:"minSelect"`       // 最小关联数量
    MaxSelect     int    `form:"maxSelect" json:"maxSelect"`       // 最大关联数量
    Required      bool   `form:"required" json:"required"`         // 是否必填
}
```

关键属性说明：
- **CollectionId**: 指向被关联集合的 ID，关系字段的核心指向
- **CascadeDelete**: 当所有关联记录被删除时，是否自动删除当前记录
- **MinSelect/MaxSelect**: 控制单选/多选（MaxSelect > 1 时为多选）
- **Required**: 关系字段是否必须有值

### 1.2 字段值存储格式

根据 [ColumnType](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/field_relation.go#L155-L162) 方法：
- **单选关系** (`MaxSelect <= 1`)：`TEXT DEFAULT '' NOT NULL`，存储单个 ID 字符串
- **多选关系** (`MaxSelect > 1`)：`JSON DEFAULT '[]' NOT NULL`，存储 JSON 字符串数组

---

## 二、记录创建时关系字段的处理路径

### 2.1 完整调用链路

从 HTTP 请求到数据库持久化的完整流程：

```
recordCreate (API Handler)
  └─► recordDataFromRequest
        └─► record.ReplaceModifiers   // 处理 "field+"、"+field"、"field-" 修饰符
  └─► forms.NewRecordUpsert
  └─► form.Load(data)                 // 将数据加载到 Record
        └─► record.SetIfFieldExists
              └─► RelationField.FindSetter
                    └─► setValue / appendValue / prependValue / subtractValue
                          └─► RelationField.normalizeValue
  └─► form.Submit()
        └─► app.SaveWithContext
              └─► app.create
                    └─► OnModelCreate Hook
                          └─► OnRecordCreate Hook (model事件→record事件桥接)
                                └─► callFieldInterceptors(InterceptorActionCreate)
                    └─► app.ValidateWithContext
                          └─► OnModelValidate Hook
                                └─► OnRecordValidate Hook
                                      └─► callFieldInterceptors(InterceptorActionValidate)
                                            └─► onRecordValidate
                                                  └─► RelationField.ValidateValue  // ★ 关联校验
                    └─► OnModelCreateExecute Hook
                          └─► OnRecordCreateExecute Hook
                                └─► callFieldInterceptors(InterceptorActionCreateExecute)
                                      └─► onRecordSaveExecute
                                            └─► DB Insert
                                                  └─► Record.DBExport
                                                        └─► RelationField.DriverValue  // 值序列化
```

### 2.2 值归一化（normalizeValue）

位于 [field_relation.go#L169-L180](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/field_relation.go#L169-L180)：

```go
func (f *RelationField) normalizeValue(raw any) any {
    val := list.ToUniqueStringSlice(raw)  // 转为去重的字符串切片
    if !f.IsMultiple() {
        if len(val) > 0 {
            return val[len(val)-1]  // 单选：取最后一个值
        }
        return ""
    }
    return val  // 多选：返回切片
}
```

### 2.3 Setter 修饰符机制

RelationField 实现了 [SetterFinder](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/field.go#L126-L136) 接口，支持四种键模式：

| 键模式 | 方法 | 行为 |
|--------|------|------|
| `fieldName` | `setValue` | 直接覆盖设置 |
| `+fieldName` | `prependValue` | 在前面追加值 |
| `fieldName+` | `appendValue` | 在后面追加值 |
| `fieldName-` | `subtractValue` | 移除指定值 |

修饰符在 API 层通过 [Record.ReplaceModifiers](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/record_model.go#L1360-L1390) 被解析处理。

---

## 三、关联校验处理逻辑

### 3.1 校验触发时机

校验发生在 `SaveWithContext` → `ValidateWithContext` 流程中，核心是 [onRecordValidate](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/record_model.go#L1413-L1427) 函数：

```go
func onRecordValidate(e *RecordEvent) error {
    errs := validation.Errors{}
    for _, f := range e.Record.Collection().Fields {
        if err := f.ValidateValue(e.Context, e.App, e.Record); err != nil {
            errs[f.GetName()] = err
        }
    }
    if len(errs) > 0 {
        return errs
    }
    return e.Next()
}
```

### 3.2 RelationField.ValidateValue 详细分析

位于 [field_relation.go#L197-L237](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/field_relation.go#L197-L237)，校验步骤：

**第一步：空值与必填校验**
```go
ids := list.ToUniqueStringSlice(record.GetRaw(f.Name))
if len(ids) == 0 {
    if f.Required {
        return validation.ErrRequired
    }
    return nil // 非必填且为空，直接通过
}
```

**第二步：数量范围校验**
```go
if f.MinSelect > 0 && len(ids) < f.MinSelect {
    return validation.NewError("validation_not_enough_values", ...)
}
maxSelect := max(f.MaxSelect, 1)
if len(ids) > maxSelect {
    return validation.NewError("validation_too_many_values", ...)
}
```

**第三步：关联记录存在性校验（核心）**
```go
relCollection, err := app.FindCachedCollectionByNameOrId(f.CollectionId)
if err != nil {
    return validation.NewError("validation_missing_rel_collection", ...)
}

var total int
_ = app.ConcurrentDB().
    Select("count(*)").
    From(relCollection.Name).
    AndWhere(dbx.In("id", list.ToInterfaceSlice(ids)...)).
    Row(&total)
if total != len(ids) {
    return validation.NewError("validation_missing_rel_records", 
        "Failed to find all relation records with the provided ids")
}
```

> **关键点**：通过 `COUNT(*)` 查询比对提交的 ID 数量和数据库中实际存在的记录数量来校验关联有效性。

### 3.3 字段设置校验（ValidateSettings）

[field_relation.go#L240-L300](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/field_relation.go#L240-L300) 用于校验关系字段本身的配置合法性：

- `CollectionId` 不能为空且必须指向存在的集合
- 关系字段创建后 `CollectionId` 不可变更
- 非视图集合不能关联到视图集合（只有视图之间可以互相关联）
- `MinSelect <= MaxSelect`

---

## 四、删除记录时级联规则的影响处理

### 4.1 删除流程总览

```
recordDelete (API Handler)
  └─► e.App.Delete(record)
        └─► app.DeleteWithContext
              └─► OnModelDelete Hook
                    └─► OnRecordDelete Hook
                          └─► callFieldInterceptors(InterceptorActionDelete)
              └─► OnModelDeleteExecute Hook
                    └─► OnRecordDeleteExecute Hook
                          └─► callFieldInterceptors(InterceptorActionDeleteExecute)
                                └─► onRecordDeleteExecute   // ★ 级联核心
                                      ├─► FindCachedCollectionReferences  // 查找反向引用
                                      ├─► RunInTransaction
                                      │     ├─► e.Next()  // 先删除当前记录（防死锁）
                                      │     └─► cascadeRecordDelete       // ★ 处理级联
                                      │           └─► deleteRefRecords     // 处理每条引用
```

### 4.2 查找反向引用（FindCachedCollectionReferences）

位于 [collection_query.go#L165-L188](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/collection_query.go#L165-L188)：

```go
func (app *BaseApp) FindCachedCollectionReferences(collection *Collection, excludeIds ...string) (map[*Collection][]Field, error) {
    collections, _ := app.Store().Get(StoreKeyCachedCollections).([]*Collection)
    result := map[*Collection][]Field{}
    for _, c := range collections {
        if slices.Contains(excludeIds, c.Id) { continue }
        for _, rawField := range c.Fields {
            f, ok := rawField.(*RelationField)
            if ok && f.CollectionId == collection.Id {  // 找到所有指向当前集合的RelationField
                result[c] = append(result[c], f)
            }
        }
    }
    return result, nil
}
```

返回值 `map[*Collection][]Field` 的含义是：**哪些集合的哪些字段引用了当前被删除记录所在的集合**。

### 4.3 级联删除核心（cascadeRecordDelete）

位于 [record_model.go#L1506-L1574](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/record_model.go#L1506-L1574)：

**设计要点**：
- 在事务外先查询引用（避免 SQLITE_BUSY 读写冲突）
- 先删除主记录，再处理引用（防止 A↔B 互相引用导致死锁）
- 按集合名称排序保证处理顺序确定性

**查询引用记录的逻辑**：
```go
if opt, ok := field.(MultiValuer); !ok || !opt.IsMultiple() {
    // 单选关系：直接等值匹配
    query.AndWhere(dbx.HashExp{prefixedFieldName: mainRecord.Id})
} else {
    // 多选关系：用 JSON_EACH 拆解数组后匹配
    query.AndWhere(dbx.Exists(dbx.NewExp(fmt.Sprintf(
        `SELECT 1 FROM %s {{__je__}} WHERE [[__je__.value]]={:jevalue}`,
        dbutils.JSONEach(prefixedFieldName),
    ), dbx.Params{"jevalue": mainRecord.Id})))
}
```

批处理大小为 4000 条，循环处理直到无更多引用记录。

### 4.4 引用记录处理（deleteRefRecords）

位于 [record_model.go#L1581-L1621](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/record_model.go#L1581-L1621)，这是级联规则的最终决策点：

```go
func deleteRefRecords(app App, mainRecord *Record, refRecords []*Record, field Field) error {
    relField, _ := field.(*RelationField)
    
    for _, refRecord := range refRecords {
        // 步骤1：从引用记录中移除被删除记录的ID
        ids := refRecord.GetStringSlice(relField.Name)
        for i := len(ids) - 1; i >= 0; i-- {
            if ids[i] == mainRecord.Id {
                ids = append(ids[:i], ids[i+1:]...)
                break
            }
        }

        // 步骤2：判断级联删除
        if relField.CascadeDelete && len(ids) == 0 {
            if err := app.Delete(refRecord); err != nil {  // 递归删除
                return err
            }
            continue  // 引用记录已被删除，无需后续处理
        }

        // 步骤3：判断必填约束
        if relField.Required && len(ids) == 0 {
            return fmt.Errorf(
                "the record cannot be deleted because it is part of a required reference in record %s (%s collection)",
                refRecord.Id, refRecord.Collection().Name)
        }

        // 步骤4：普通情况——保存修改后的值（去除了被删除ID的引用）
        refRecord.Set(relField.Name, ids)
        if err := app.SaveNoValidate(refRecord); err != nil {
            return err
        }
    }
    return nil
}
```

### 4.5 级联决策矩阵

| 场景 | CascadeDelete | Required | 移除后ids为空 | 结果 |
|------|---------------|----------|---------------|------|
| 1 | true | 任意 | 是 | **级联删除**引用记录 |
| 2 | false | true | 是 | **报错阻止删除**（必填关系被破坏） |
| 3 | false | false | 是 | 清空引用字段，保存引用记录 |
| 4 | 任意 | 任意 | 否（还有其他关联） | 仅移除当前ID，保存引用记录 |

> **注意**：`CascadeDelete` 的语义是「当所有关联都没了就删自己」，而不是「删了关联就跟着删」。只有当移除被删ID后关系列表完全为空时才触发级联。

### 4.6 数据库表同步

[collection_record_table_sync.go](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/collection_record_table_sync.go) 负责集合结构变更时的表同步：

- [SyncRecordTableSchema](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/collection_record_table_sync.go#L21-L153)：创建/重命名/删除列，重建索引
- [normalizeSingleVsMultipleFieldChanges](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/collection_record_table_sync.go#L155-L298)：处理单选↔多选切换时的数据迁移
  - 单选→多选：将单值包装为 JSON 数组 `[value]`
  - 多选→单选：取数组最后一个元素作为值

---

## 五、Hook 与拦截器机制

### 5.1 Model 事件与 Record 事件桥接

PocketBase 使用两层事件系统：通用的 Model 事件和特化的 Record 事件。桥接逻辑在 [record_model.go#L55-L474](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/record_model.go#L55-L474) 的 `registerRecordHooks` 中注册。

以删除为例：
```
OnModelDelete (priority -99)
  └─► 转换为 OnRecordDelete
        └─► 执行完后通过 syncModelEventWithRecordEvent 同步状态
```

### 5.2 字段拦截器（RecordInterceptor）

[field.go#L171-L185](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/field.go#L171-L185) 定义了字段级别的拦截器接口：

```go
type RecordInterceptor interface {
    Intercept(
        ctx context.Context,
        app App,
        record *Record,
        actionName string,  // "validate"、"create"、"delete" 等
        actionFunc func() error,
    ) error
}
```

[callFieldInterceptors](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/record_model.go#L1394-L1411) 将集合所有字段的拦截器包装成洋葱模型：
```go
for _, field := range m.Collection().Fields {
    if f, ok := field.(RecordInterceptor); ok {
        oldfn := actionFunc
        actionFunc = func() error {
            return f.Intercept(ctx, app, m, actionName, oldfn)
        }
    }
}
return actionFunc()
```

虽然当前 `RelationField` 没有直接实现 `RecordInterceptor`（主要是文件字段在用），但这个框架为关系字段的扩展提供了可能。

---

## 六、关键文件索引

| 文件 | 职责 |
|------|------|
| [field_relation.go](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/field_relation.go) | RelationField 定义、值归一化、校验、Setter修饰符 |
| [field.go](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/field.go) | 字段通用接口、拦截器接口、拦截器Action常量 |
| [record_model.go](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/record_model.go) | Record模型、Hook桥接、级联删除(cascadeRecordDelete/deleteRefRecords) |
| [db.go](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/db.go) | Save/Delete 核心流程、事务处理、事件触发 |
| [collection_query.go](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/collection_query.go) | FindCachedCollectionReferences 反向引用查找 |
| [collection_record_table_sync.go](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/collection_record_table_sync.go) | 表结构同步、单选/多选切换的数据迁移 |
| [forms/record_upsert.go](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/forms/record_upsert.go) | 记录创建/更新表单、数据加载与提交 |
| [apis/record_crud.go](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/apis/record_crud.go) | HTTP API Handler、修饰符解析、权限检查 |
