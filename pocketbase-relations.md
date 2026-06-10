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

### 4.1 onRecordDeleteExecute：事务边界划分

位于 [record_model.go#L1476-L1501](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/record_model.go#L1476-L1501)，这是级联删除的入口函数，也是事务边界的关键划分点：

```go
func onRecordDeleteExecute(e *RecordEvent) error {
    // ====== 事务外：只查集合Schema引用（不查数据） ======
    // note: the select is outside of the transaction to minimize
    // SQLITE_BUSY errors when mixing read&write in a single transaction
    refs, err := e.App.FindCachedCollectionReferences(e.Record.Collection())
    if err != nil {
        return err
    }

    originalApp := e.App
    // ====== 进入事务：所有数据读写都在这里 ======
    txErr := e.App.RunInTransaction(func(txApp App) error {
        e.App = txApp

        // 先删除主记录，再处理引用（防止 A<->B 互指导致递归死锁）
        if err := e.Next(); err != nil {
            return err
        }

        return cascadeRecordDelete(txApp, e.Record, refs)
    })
    e.App = originalApp

    return txErr
}
```

**事务边界要点**：

| 阶段 | 位置 | 做什么 | 为什么放事务外 |
|------|------|--------|----------------|
| 事务外 | `FindCachedCollectionReferences` | 从缓存里遍历所有集合Schema，找出**哪些集合的哪些 RelationField 字段定义**指向了被删记录所在的集合（返回 `map[*Collection][]Field`） | 只读缓存不涉及DB，避免SQLite同一事务内读写混合触发 SQLITE_BUSY |
| 事务内 | `e.Next()` + `cascadeRecordDelete` | 真正删除主记录、查询具体引用记录、处理级联逻辑 | 需要原子性：要么主记录和所有级联都成功，要么全部回滚 |

> 这里非常容易混淆：**事务外拿到的是「字段定义引用」，不是「引用记录」**。具体哪些记录引用了当前被删记录，是在事务内部通过 `cascadeRecordDelete` 里的 `app.RecordQuery(refCollection)` 去数据库查的。

### 4.2 事务外：FindCachedCollectionReferences

位于 [collection_query.go#L165-L188](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/collection_query.go#L165-L188)：

```go
func (app *BaseApp) FindCachedCollectionReferences(collection *Collection, excludeIds ...string) (map[*Collection][]Field, error) {
    collections, _ := app.Store().Get(StoreKeyCachedCollections).([]*Collection)
    result := map[*Collection][]Field{}
    for _, c := range collections {
        if slices.Contains(excludeIds, c.Id) { continue }
        for _, rawField := range c.Fields {
            f, ok := rawField.(*RelationField)
            if ok && f.CollectionId == collection.Id {  // 匹配字段定义上的CollectionId
                result[c] = append(result[c], f)         // 收集的是Field对象，不是记录
            }
        }
    }
    return result, nil
}
```

返回值 `map[*Collection][]Field` 的含义：**Schema层面——哪些集合的哪些 RelationField 字段定义指向了当前集合**。这只是告诉 cascadeRecordDelete "去哪些集合的哪些字段里查引用记录"，本身不返回任何数据记录。

### 4.3 事务内：cascadeRecordDelete

位于 [record_model.go#L1506-L1574](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/record_model.go#L1506-L1574)，这里做两件重要的过滤：**跳过视图集合** 和 **排除自引用场景下的被删记录自身**。

#### 4.3.1 跳过视图集合

```go
for _, refCollection := range sortedRefKeys {
    fields, ok := refs[refCollection]
    if !ok || refCollection.IsView() {
        continue // skip missing or view collections
    }
```

**为什么要跳过视图**：视图集合（View Collection）本质是一条 SQL `SELECT` 查询映射出来的虚拟表，不存储自己的数据记录，底层没有可操作的物理表。引用字段如果在视图里，指向关系的完整性由底层查询保证，不需要也不能对视图做 UPDATE/DELETE，所以直接跳过。

#### 4.3.2 同集合自引用排除被删记录自身（防御性过滤）

```go
query := app.RecordQuery(refCollection)

// ...（单选/多选查询条件）...

if refCollection.Id == mainRecord.Collection().Id {
    query.AndWhere(dbx.Not(dbx.HashExp{recordTableName + ".id": mainRecord.Id}))
}
```

**先看删除时序**：
```
事务内执行顺序：
  1. e.Next() → DELETE FROM categories WHERE id = 'A'  （先删主记录）
  2. cascadeRecordDelete → SELECT ... WHERE parent = 'A'  （后查引用）
```

按 SQLite 默认的 SERIALIZABLE 隔离级别，同一事务内删除后再查询，被删的A记录理论上已经不可见了，不会被查询结果包含。那为什么还要加这个 `NOT id = mainRecord.Id`？

**这是一个防御性过滤，设计意图有三层**：

1. **不依赖删除时序**：显式地表达"被删记录自身的自引用不纳入级联处理范围"的意图，不依赖 e.Next() 和查询之间的先后执行顺序。如果未来代码调整了删除和查询的顺序，这个条件依然能保证正确性。

2. **递归删除场景的兜底保护**：`deleteRefRecords` 里触发级联时调用的是 `app.Delete(refRecord)`，会完整走一遍删除流程。根据 [runInTransaction](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/db_tx.go#L25-L49) 的实现——如果当前 db 已经是 `*dbx.Tx`，嵌套的 `RunInTransaction` **不会另开 savepoint，而是直接复用当前事务执行回调**——这意味着：
   - 所有级联删除、引用记录更新都在**同一个顶层事务**里执行
   - 同事务内的写操作对后续查询是**立即可见**的（不会被隔离）
   - 当自引用环（A→B→C→A）在同一事务内递归处理时，显式排除当前被删记录可以避免对同一条记录的重复扫描和处理

3. **明确语义边界**：被删记录本身即将消失，它的字段值（包括自引用关系）已经没有任何维护价值。通过查询条件显式排除，可以避免对"一条即将消失的记录的关系字段"做任何无意义的读/写操作。

> 小结：不是"删了之后还能查到自己所以要排除"，而是"不管查不查得到，我都明确声明不处理自己"。这是防御性编程的典型做法。

#### 4.3.3 真正的引用记录查询（在事务内）

```go
if opt, ok := field.(MultiValuer); !ok || !opt.IsMultiple() {
    // 单选关系：字段值直接等于被删记录id
    query.AndWhere(dbx.HashExp{prefixedFieldName: mainRecord.Id})
} else {
    // 多选关系：用 SQLite JSON_EACH 把JSON数组拆成多行再匹配
    query.AndWhere(dbx.Exists(dbx.NewExp(fmt.Sprintf(
        `SELECT 1 FROM %s {{__je__}} WHERE [[__je__.value]]={:jevalue}`,
        dbutils.JSONEach(prefixedFieldName),
    ), dbx.Params{"jevalue": mainRecord.Id})))
}
```

查询结果按 4000 条一批分批取出，循环调用 `deleteRefRecords` 处理，直到无更多记录。

### 4.4 deleteRefRecords：级联决策

位于 [record_model.go#L1581-L1621](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/record_model.go#L1581-L1621)，这是级联规则的最终决策点：

```go
func deleteRefRecords(app App, mainRecord *Record, refRecords []*Record, field Field) error {
    relField, _ := field.(*RelationField)
    
    for _, refRecord := range refRecords {
        // 步骤1：从引用记录的字段值中移除被删记录的ID
        ids := refRecord.GetStringSlice(relField.Name)
        for i := len(ids) - 1; i >= 0; i-- {
            if ids[i] == mainRecord.Id {
                ids = append(ids[:i], ids[i+1:]...)
                break
            }
        }

        // 步骤2：级联删除——只有当"移除后关系为空"且CascadeDelete为true时触发
        if relField.CascadeDelete && len(ids) == 0 {
            if err := app.Delete(refRecord); err != nil {  // 递归走完整的Delete流程
                return err
            }
            continue
        }

        // 步骤3：必填保护——如果关系为空且字段是Required，报错回滚
        if relField.Required && len(ids) == 0 {
            return fmt.Errorf(
                "the record cannot be deleted because it is part of a required reference in record %s (%s collection)",
                refRecord.Id, refRecord.Collection().Name)
        }

        // 步骤4：普通情况——更新引用记录（跳过校验，因为其他引用可能也已失效）
        refRecord.Set(relField.Name, ids)
        if err := app.SaveNoValidate(refRecord); err != nil {
            return err
        }
    }
    return nil
}
```

> 注意步骤4用的是 `SaveNoValidate`：同一事务内可能已经删除了其他被引用的记录，如果再跑完整校验（ValidateValue 会去查所有关联记录是否存在）会因为引用已被删除而报错，所以这里跳过校验直接写库。

### 4.5 级联决策矩阵

| 场景 | CascadeDelete | Required | 移除被删ID后ids为空 | 结果 |
|------|---------------|----------|---------------------|------|
| 1 | true | 任意 | 是 | **递归级联删除**引用记录 |
| 2 | false | true | 是 | **报错回滚**（必填关系被破坏，不允许删除） |
| 3 | false | false | 是 | 清空引用字段，保存引用记录 |
| 4 | 任意 | 任意 | 否（还有其他关联） | 仅从关系列表中移除当前ID，保存引用记录 |

> **CascadeDelete 语义澄清**：不是"关联的记录被删了我就跟着删"，而是**"当我的这个关系字段里所有的关联都没了时，我自己也没有存在的意义了，把我也删掉"**。这也是为什么只有 `len(ids) == 0` 时才触发——多选关系下如果还有其他关联记录，不触发级联。

### 4.6 删除流程完整时序图

```
onRecordDeleteExecute
  │
  ├─ [事务外] FindCachedCollectionReferences
  │     └─ 遍历缓存中的所有集合Schema
  │     └─ 返回 map[*Collection][]Field（只是字段定义，不含记录）
  │
  └─ RunInTransaction ───────────────────────────────── 事务边界
        │
        ├─ e.Next() ──────────────────────────────── 先删主记录（防互指死锁）
        │     └─ DELETE FROM mainTable WHERE id = ?
        │
        └─ cascadeRecordDelete(txApp, record, refs)
              │
              ├─ 按集合名排序遍历
              │     │
              │     ├─ refCollection.IsView() → continue  （跳过视图）
              │     │
              │     └─ 遍历每个引用字段
              │           │
              │           ├─ 构造 RecordQuery
              │           │     ├─ 单选：WHERE field = record.Id
              │           │     └─ 多选：WHERE JSON_EACH(field) CONTAINS record.Id
              │           │
              │           ├─ [自引用场景] refColId == mainColId
              │           │     └─ AND NOT id = mainRecord.Id （排除自己）
              │           │
              │           ├─ 按4000条/批循环查询引用记录
              │           │
              │           └─ deleteRefRecords(app, mainRecord, batch, field)
              │                 │
              │                 ├─ 从ids里移除 mainRecord.Id
              │                 ├─ CascadeDelete && 空 → app.Delete(递归)
              │                 ├─ Required && 空 → return error
              │                 └─ 其他情况 → SaveNoValidate 更新引用记录
              │
              └─ 所有引用集合处理完成 → return nil（提交事务）
```

### 4.7 递归删除中的事务复用机制

递归级联删除时事务的真实行为是理解整个流程的关键。看 [runInTransaction](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/db_tx.go#L25-L49) 的核心实现：

```go
func (app *BaseApp) runInTransaction(db dbx.Builder, fn func(txApp App) error, isForAuxDB bool) error {
    switch txOrDB := db.(type) {
    case *dbx.Tx:
        // run as part of the already existing transaction
        return fn(app)   // ← 已经在事务里了，直接执行回调，不开新事务也不开 savepoint
    case *dbx.DB:
        var txApp *BaseApp
        txErr := txOrDB.Transactional(func(tx *dbx.Tx) error {  // ← 只有这里才开新事务
            txApp = app.createTxApp(tx, isForAuxDB)
            return fn(txApp)
        })
        // ... 处理 txApp.txInfo.runAfterFuncs
        return txErr
    }
}
```

**行为总结**：

| 调用场景 | db 实际类型 | 行为 |
|----------|-------------|------|
| 顶层 `onRecordDeleteExecute` 首次调用 | `*dbx.DB`（普通连接） | 开启真正的 SQLite 事务，创建 txApp（txInfo.parent 指向原 app） |
| 级联触发 `app.Delete(refRecord)` 后，其内部的 `RunInTransaction` | `*dbx.Tx`（已经在事务里） | **直接执行回调，复用当前事务**——既不开新事务，也不用 SAVEPOINT |

**对级联删除的实际影响**：

1. **所有写操作共属一个事务**：主记录删除、所有级联的引用记录删除/更新都在同一个顶层事务里。任何一步报错，全部一起回滚。

2. **写操作立即可见**：同一事务内的 DELETE/UPDATE 对后续的 SELECT 是立即可见的（没有事务隔离）。比如引用记录 A 删了之后，后续循环里的查询就查不到 A 了。

3. **OnComplete 回调的触发时机**：`txInfo.OnComplete` 注册的回调（比如 `OnModelAfterDeleteSuccess`）不会在每层嵌套完成时触发，而是全部累积到**最外层事务提交/回滚后**才统一执行（由最外层的 `txApp.txInfo.runAfterFuncs` 一次性调用）。

4. **为什么这样设计**：避免 savepoint 嵌套带来的性能开销和 SQLite 兼容性问题，同时保证级联操作的完整原子性。

### 4.8 数据库表同步

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
| [db_tx.go](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/db_tx.go) | RunInTransaction 实现、嵌套事务复用逻辑、TxAppInfo 回调管理 |
| [collection_query.go](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/collection_query.go) | FindCachedCollectionReferences 反向引用查找 |
| [collection_record_table_sync.go](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/core/collection_record_table_sync.go) | 表结构同步、单选/多选切换的数据迁移 |
| [forms/record_upsert.go](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/forms/record_upsert.go) | 记录创建/更新表单、数据加载与提交 |
| [apis/record_crud.go](file:///d:/fz/0601/solo-dogfeeding/code/153-pocketbase/apis/record_crud.go) | HTTP API Handler、修饰符解析、权限检查 |
