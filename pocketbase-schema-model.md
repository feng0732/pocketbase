# PocketBase 集合 Schema 动态建模代码链路分析

## 概述

PocketBase 的集合（Collection）Schema 系统是一个运行时动态建模框架，允许用户通过 API/Admin UI 定义、修改数据结构，并自动同步到底层 SQLite 数据库。核心链路涉及四层：API 层 → 验证层 → Hook 执行层 → DB 同步层。

```
HTTP 请求 (apis/collection.go)
    ↓
请求事件 → OnCollectionCreateRequest / OnCollectionUpdateRequest
    ↓
App.Save(Collection) → 触发 Model 事件钩子
    ↓
OnCollectionValidate (core/collection_validate.go)
    ↓
OnCollectionSave (core/collection_model.go L800-L844)
    ↓
OnCollectionSaveExecute (core/collection_model.go L846-L932)
    ↓
SyncRecordTableSchema (core/collection_record_table_sync.go)
    ↓
实际 SQL 执行：CreateTable / AddColumn / RenameColumn / DropColumn / CreateIndex / DropIndex
```

---

## 一、Collection 模型核心定义

### 1.1 三种集合类型

定义在 [collection_model.go](file:///d:/fz/0601/solo-dogfeeding/code/149-pocketbase/core/collection_model.go#L23-L27)：

| 类型 | 常量 | 用途 |
|------|------|------|
| base | `CollectionTypeBase` | 普通数据表，存储业务记录 |
| auth | `CollectionTypeAuth` | 用户认证表，内置密码/邮箱/Token 等系统字段 |
| view | `CollectionTypeView` | SQL 视图，字段从查询自动推导，不存实际数据 |

### 1.2 核心结构体

[collection_model.go L352-L386](file:///d:/fz/0601/solo-dogfeeding/code/149-pocketbase/core/collection_model.go#L352-L386)

```go
type baseCollection struct {
    BaseModel
    Name       string                  // 表名，同时也是 SQLite 表名
    Type       string                  // base/auth/view
    Fields     FieldsList              // 字段定义列表（JSON 序列化存储）
    Indexes    types.JSONArray[string] // CREATE INDEX 语句数组
    ListRule   *string                 // API 访问规则
    ViewRule   *string
    CreateRule *string
    UpdateRule *string
    DeleteRule *string
    RawOptions types.JSONRaw           // auth/view 类型的专属配置
    System     bool                    // 系统集合标记（不可删/改名）
}

type Collection struct {
    baseCollection
    collectionAuthOptions  // auth 类型配置：OAuth2、密码策略、Token 配置等
    collectionViewOptions  // view 类型配置：ViewQuery SQL 语句
}
```

**存储机制**：Collection 元数据存储在 `_collections` 系统表中，`Fields` 和 `Indexes` 字段以 JSON 字符串形式持久化。

### 1.3 工厂方法

[collection_model.go L388-L462](file:///d:/fz/0601/solo-dogfeeding/code/149-pocketbase/core/collection_model.go#L388-L462)

- `NewCollection(typ, name)` — 根据类型分派
- `NewBaseCollection(name)` — 初始化 `id` 系统字段
- `NewAuthCollection(name)` — 初始化 `id, password, tokenKey, email, emailVisibility, verified` 系统字段及对应唯一索引
- `NewViewCollection(name)` — 不初始化字段，字段由 SQL 查询推导

ID 自动生成规则：`"pbc_" + crc32(Type + Name)`，冲突时追加数字后缀。

---

## 二、字段类型系统

### 2.1 Field 接口

[field.go L69-L113](file:///d:/fz/0601/solo-dogfeeding/code/149-pocketbase/core/field.go#L69-L113)

每个字段类型必须实现以下核心方法：

| 方法 | 作用 |
|------|------|
| `Type() string` | 返回字段类型标识（如 "text", "number"） |
| `ColumnType(app App) string` | 返回 SQLite 列定义 DDL |
| `PrepareValue(record, raw) (any, error)` | 将原始输入规范化为 Go 值 |
| `ValidateValue(ctx, app, record) error` | 运行时值校验 |
| `ValidateSettings(ctx, app, collection) error` | Schema 定义时的字段配置校验 |

扩展接口：
- `MultiValuer` — 是否支持多值（通过 `MaxSelect > 1` 判断，仅 3 种字段实现）
- `DriverValuer` — 导出数据库存储值
- `SetterFinder` / `GetterFinder` — 自定义字段访问器（支持 `field+`, `:autogenerate` 等修饰符）
- `RecordInterceptor` — 拦截 Record 的 CRUD 生命周期
- `MaxBodySizeCalculator` — 计算字段值最大请求体大小

### 2.2 字段注册表

[field.go L63-L67](file:///d:/fz/0601/solo-dogfeeding/code/149-pocketbase/core/field.go#L63-L67)

```go
var Fields = map[string]FieldFactoryFunc{}

// 各 field_*.go 的 init() 中注册，例如 field_text.go L18-L22:
func init() {
    Fields[FieldTypeText] = func() Field { return &TextField{} }
}
```

### 2.3 全部 14 种字段类型与 SQLite 列映射

根据 `Grep` 核对 `Fields[...]` 注册语句，共 **14 种** 字段类型（不是 15 种）：

| 字段类型常量值 | Go 结构体 | SQLite 列类型 | 默认值 | MultiValuer |
|----------------|-----------|--------------|--------|-------------|
| `text` | `TextField` | TEXT | `''`（PK 时为 `'r'||lower(hex(randomblob(7)))`） | 否 |
| `number` | `NumberField` | NUMERIC | `0` | 否 |
| `bool` | `BoolField` | BOOLEAN | `FALSE` | 否 |
| `email` | `EmailField` | TEXT | `''` | 否 |
| `url` | `URLField` | TEXT | `''` | 否 |
| `date` | `DateField` | TEXT | `''` | 否 |
| `autodate` | `AutodateField` | TEXT | `''` | 否 |
| `editor` | `EditorField` | TEXT | `''` | 否 |
| `password` | `PasswordField` | TEXT | `''` | 否 |
| `json` | `JSONField` | JSON | `NULL` | 否 |
| `geoPoint` | `GeoPointField` | JSON | `'{"lon":0,"lat":0}'` | 否 |
| `select` | `SelectField` | TEXT 或 JSON | `''` 或 `'[]'`（MaxSelect>1 时） | **是** |
| `relation` | `RelationField` | TEXT 或 JSON | `''` 或 `'[]'`（MaxSelect>1 时） | **是** |
| `file` | `FileField` | TEXT 或 JSON | `''` 或 `'[]'`（MaxSelect>1 时） | **是** |

**核对依据**：
- 14 个 `Fields[...]` 注册语句，参见各 `field_*.go` 的 `init()` 函数
- 仅 3 个字段声明实现 `MultiValuer`：[field_select.go L23](file:///d:/fz/0601/solo-dogfeeding/code/149-pocketbase/core/field_select.go#L23)、[field_relation.go L23](file:///d:/fz/0601/solo-dogfeeding/code/149-pocketbase/core/field_relation.go#L23)、[field_file.go L39](file:///d:/fz/0601/solo-dogfeeding/code/149-pocketbase/core/field_file.go#L39)
- `geoPoint` 类型名是驼峰（常量 `FieldTypeGeoPoint = "geoPoint"`），不是下划线分隔
- `GeoPointField.ColumnType()` 返回 ``JSON DEFAULT '{"lon":0,"lat":0}' NOT NULL``，lon 在前、lat 在后

多值切换规则在 [collection_record_table_sync.go L155-L298](file:///d:/fz/0601/solo-dogfeeding/code/149-pocketbase/core/collection_record_table_sync.go#L155-L298) 中处理（见第四节）。

### 2.4 FieldsList 容器

[fields_list.go](file:///d:/fz/0601/solo-dogfeeding/code/149-pocketbase/core/fields_list.go)

核心能力：
- **按 ID 或 Name 查找**：`GetById(id)`, `GetByName(name)`
- **智能 Add**（L118-L122）：按 ID 匹配替换，无 ID 时按 Name 匹配并复用原 ID，避免字段改名时丢失数据
- **JSON 反序列化**（L316-L332）：先读取 `type` 字段，从 `Fields` 注册表取工厂函数创建具体类型，再反序列化剩余属性
- **ID 自动生成**（L224-L236）：`type + crc32(name)`，冲突追加数字后缀

---

## 三、验证层：onCollectionValidate

[collection_validate.go](file:///d:/fz/0601/solo-dogfeeding/code/149-pocketbase/core/collection_validate.go)

通过 `OnCollectionValidate` Hook 触发（Priority=99），执行 `collectionValidator.run()`。

### 3.1 基础合法性校验

| 校验项 | 位置 | 说明 |
|--------|------|------|
| Id 唯一性 | L82-L94 | 新建时校验，已有集合禁止改 ID |
| System 标记 | L96-L98 | 禁止修改 system 标记 |
| Type | L99-L108 | 必须为 base/auth/view，禁止改类型 |
| Name | L109-L117 | 正则 `^\w+$`，全局唯一（不区分大小写），不与已有表名/集合 ID 冲突 |
| 规则语法 | L131-L157 | 通过 `search.FilterData` 编译规则表达式 |

### 3.2 Fields 校验

- `checkFieldDuplicates`（L245-L288）：ID 和 Name 不重复（Name 大小写不敏感）
- `checkMinFields`（L361-L423）：必须有 `id` PK 字段；auth 集合必须包含 password/tokenKey/email/emailVisibility/verified 系统字段
- `ensureNoSystemFieldsChange`（L425-L448）：系统字段不可删除或改名
- `ensureNoFieldsTypeChange`（L220-L243）：字段类型一旦创建不可更改（按 Field ID 匹配）
- `checkReservedAuthKeys`（L333-L359）：auth 集合禁止使用 `passwordConfirm`, `oldPassword` 等保留字段名
- `checkFieldValidators`（L290-L309）：逐个调用每个字段的 `ValidateSettings()`

### 3.3 Indexes 校验

[collection_validate.go L525-L687](file:///d:/fz/0601/solo-dogfeeding/code/149-pocketbase/core/collection_validate.go#L525-L687)

1. 视图集合禁止定义索引
2. 使用 `dbutils.ParseIndex()` 解析 SQL，必须合法
3. Index 名称全局唯一（跨集合）
4. Index 定义（排除名称/表名后）不重复
5. 系统字段上的唯一索引不可删除或修改
6. auth 集合必须有 tokenKey 和 email 的唯一索引（由 initTokenKeyField/initEmailField 自动注入）

---

## 四、运行时表同步：SyncRecordTableSchema

[collection_record_table_sync.go](file:///d:/fz/0601/solo-dogfeeding/code/149-pocketbase/core/collection_record_table_sync.go)

在 `OnCollectionSaveExecute` Hook 的事务末尾调用（Priority=99，最接近 DB 操作）。

### 4.1 新建集合（oldCollection == nil）

[collection_record_table_sync.go L28-L47](file:///d:/fz/0601/solo-dogfeeding/code/149-pocketbase/core/collection_record_table_sync.go#L28-L47)

```go
cols := make(map[string]string)
for _, field := range fields {
    cols[field.GetName()] = field.ColumnType(app)
}
txApp.DB().CreateTable(tableName, cols).Execute()
createCollectionIndexes(txApp, newCollection)
```

### 4.2 更新集合（oldCollection != nil）

[collection_record_table_sync.go L49-L140](file:///d:/fz/0601/solo-dogfeeding/code/149-pocketbase/core/collection_record_table_sync.go#L49-L140)

执行顺序：

1. **判断是否需要索引更新**：表名变更 / Fields 序列化变更 / Indexes 序列化变更 → 需要
2. **删除旧索引**：`dropCollectionIndexes()` 逐个 `DROP INDEX IF EXISTS`
3. **表改名**（如有）：`RenameTable`
4. **删除列**：遍历旧字段，按 ID 匹配不到的列执行 `DropColumn`
5. **新增/改名列** — 使用临时列名避免冲突：
   - 新字段（ID 未匹配）：`AddColumn` 临时名 → 最后 `RenameColumn` 到真名
   - 改名字段（ID 匹配但 Name 不同）：`RenameColumn` 旧名→临时名 → 最后 `RenameColumn` 临时名→新名
6. **处理单值/多值切换**：`normalizeSingleVsMultipleFieldChanges()`（见下节）
7. **创建新索引**：`createCollectionIndexes()`
8. **优化**：事务外执行 `PRAGMA optimize`

### 4.3 单值 ↔ 多值 数据迁移（normalizeSingleVsMultipleFieldChanges）

[collection_record_table_sync.go L155-L298](file:///d:/fz/0601/solo-dogfeeding/code/149-pocketbase/core/collection_record_table_sync.go#L155-L298)

**触发条件**：仅对实现了 `MultiValuer` 接口的 3 种字段（Select/Relation/File）生效，当 `oldField.MaxSelect > 1` 与 `newField.MaxSelect > 1` 结果不同时触发。即：
- 单→多：MaxSelect 从 ≤1 改为 >1（列类型从 TEXT → JSON）
- 多→单：MaxSelect 从 >1 改为 ≤1（列类型从 JSON → TEXT）

**执行步骤（逐字段处理，每个字段独立完成以下流程）**：

1. **临时删除所有视图**（L184-L203）：从 `sqlite_master` 查出所有 CREATE VIEW 语句并保存，然后逐个 `DeleteView`，避免后续列改名时的引用约束错误（作为 `writable_schema` PRAGMA 的替代方案）

2. **列改名**（L205-L212）：原列名 `originalName` → `oldTempName = "_" + originalName + 随机5字符`

3. **新建列**（L214-L218）：用 `newField.ColumnType(app)` 在 `originalName` 位置重新 AddColumn
   - 单→多：列类型从 `TEXT DEFAULT '' NOT NULL` 变为 `JSON DEFAULT '[]' NOT NULL`
   - 多→单：列类型从 `JSON DEFAULT '[]' NOT NULL` 变为 `TEXT DEFAULT '' NOT NULL`

4. **数据转换 SQL**（L220-L273）：

   **单→多（L222-L245）**：
   ```sql
   UPDATE {table} set {newCol} = (
       CASE
           WHEN COALESCE({oldTempCol}, '') = ''
           THEN '[]'                                    -- 空值或 NULL → 空数组
           ELSE (
               CASE
                   WHEN json_valid({oldTempCol}) AND json_type({oldTempCol}) == 'array'
                   THEN {oldTempCol}                       -- 已经是合法 JSON 数组 → 原值
                   ELSE json_array({oldTempCol})            -- 其他 → 包装成单元素数组
               END
           )
       END
   )
   ```

   **多→单（L246-L273）**：
   ```sql
   UPDATE {table} set {newCol} = (
       CASE
           WHEN COALESCE({oldTempCol}, '[]') = '[]'
           THEN ''                                        -- 空数组或 NULL → 空字符串
           ELSE (
               CASE
                   WHEN json_valid({oldTempCol}) AND json_type({oldTempCol}) == 'array'
                   THEN COALESCE(json_extract({oldTempCol}, '$[#-1]'), '')  -- 取最后一个元素
                   ELSE {oldTempCol}                       -- 非数组 → 原值
               END
           )
       END
   )
   ```
   注意：多→单时，FileField 的实际文件对象不会被删除，需通过自定义迁移手动处理。

5. **删除旧列**（L281-L285）：`DropColumn(oldTempName)`

6. **恢复视图**（L287-L293）：重新执行步骤 1 保存的所有 CREATE VIEW SQL

### 4.4 删除集合

[collection_model.go L686-L743](file:///d:/fz/0601/solo-dogfeeding/code/149-pocketbase/core/collection_model.go#L686-L743)

`OnCollectionDeleteExecute` Hook：
1. 检查是否为 System 集合（禁止删除）
2. 检查是否有其他集合的 Relation 字段引用本集合
3. 事务内：`DeleteTable` 或 `DeleteView`
4. 重新保存引用本集合字段的所有视图（检查依赖）

---

## 五、索引同步

### 5.1 索引解析与构建

[tools/dbutils/index.go](file:///d:/fz/0601/solo-dogfeeding/code/149-pocketbase/tools/dbutils/index.go)

```go
type Index struct {
    SchemaName string
    IndexName  string
    TableName  string
    Columns    []IndexColumn // {Name, Collate, Sort}
    Where      string        // 部分索引条件
    Unique     bool
    Optional   bool          // IF NOT EXISTS
}
```

- `ParseIndex(sql)` — 正则解析 `CREATE [UNIQUE] INDEX [IF NOT EXISTS] name ON table (cols) [WHERE cond]`
- `Build()` — 反向生成规范化 SQL（标识符统一用反引号包裹）
- `FindSingleColumnUniqueIndex()` — 查找某列上的单列唯一索引

### 5.2 索引创建流程

[collection_record_table_sync.go L325-L366](file:///d:/fz/0601/solo-dogfeeding/code/149-pocketbase/core/collection_record_table_sync.go#L325-L366)

`createCollectionIndexes()` 遍历 `collection.Indexes`：
1. `ParseIndex` 解析
2. 强制覆盖 `TableName = collection.Name`（与集合名保持同步）
3. 验证合法性，不合法返回 `validation.Errors`
4. 执行 `parsed.Build()` 生成 SQL 并执行

### 5.3 系统自动注入索引

Auth 集合初始化时自动注入（collection_model.go）：
- `initTokenKeyField`（L998-L1027）：`CREATE UNIQUE INDEX idx_tokenKey_xxx ON collection (tokenKey)`
- `initEmailField`（L1029-L1054）：`CREATE UNIQUE INDEX idx_email_xxx ON collection (email) WHERE email != ''`

---

## 六、数据迁移系统

### 6.1 迁移运行器

[core/migrations_runner.go](file:///d:/fz/0601/solo-dogfeeding/code/149-pocketbase/core/migrations_runner.go)

核心方法：
- `Up()` — 执行所有未应用迁移，双重事务包装（AuxRunInTransaction → RunInTransaction）
- `Down(n)` — 回退最近 n 条迁移
- `RemoveMissingAppliedMigrations()` — 清理已不存在的迁移记录

迁移表 `_migrations` 结构：`file VARCHAR(255) PRIMARY KEY, applied INTEGER`

### 6.2 迁移列表

[core/migrations_list.go](file:///d:/fz/0601/solo-dogfeeding/code/149-pocketbase/core/migrations_list.go)

```go
type Migration struct {
    Up               func(txApp App) error
    Down             func(txApp App) error
    File             string                       // 文件名，用于排序和去重
    ReapplyCondition func(txApp App, runner *MigrationsRunner, fileName string) (bool, error)
}
```

两个全局列表：
- `SystemMigrations` — 框架内置迁移（见 migrations/ 目录）
- `AppMigrations` — 用户自定义迁移

### 6.3 自动迁移（Automigrate）

[plugins/migratecmd/automigrate.go](file:///d:/fz/0601/solo-dogfeeding/code/149-pocketbase/plugins/migratecmd/automigrate.go)

`automigrateOnCollectionChange` 监听 Collection CRUD 请求：
1. 执行实际保存操作（`e.Next()`）
2. 对比新旧 Collection 状态
3. 生成 Go 或 JS 模板（`goDiffTemplate` / `jsDiffTemplate`）
4. 写入迁移文件：`{timestamp}_{action}_{collectionName}.{go|js}`
5. 同时向 `_migrations` 表插入记录（标记为已应用，避免下次 Up 重复执行）

---

## 七、完整代码链路图

```
用户操作: Admin UI / API
    │
    ▼
[apis/collection.go]
  collectionCreate() / collectionUpdate() / collectionDelete()
    │
    ▼ 触发请求事件
OnCollectionCreateRequest / OnCollectionUpdateRequest / OnCollectionDeleteRequest
    │
    ▼
app.Save(collection) 或 app.Delete(collection)
    │
    ▼ 触发 Model 层事件（collection_model.go L33-L348 注册的系统钩子）
OnModelValidate → OnCollectionValidate
    │
    ▼ [collection_validate.go]
  collectionValidator.run()
    ├─ 基础属性校验（Id/Type/Name/System/Rules）
    ├─ Fields 校验（重复/最小集合/类型不变/系统字段保护）
    └─ Indexes 校验（解析/全局唯一/系统唯一索引保护）
    │
    ▼
OnModelCreate → OnCollectionCreate → onCollectionSave() [collection_model.go L800-L844]
    ├─ 设置默认 Type / Created / Updated
    ├─ 重建 FieldsList（规范化默认 ID）
    ├─ initDefaultFields() — 确保系统字段存在
    ├─ updateGeneratedIdIfExists() — 自动生成冲突 ID
    └─ 规范化 Indexes 的 TableName
    │
    ▼
OnCollectionCreateExecute → onCollectionSaveExecute() [collection_model.go L846-L932]
    ├─ View 集合: DeleteView(旧名) + SaveView(新名) + CreateViewFields 推导字段
    ├─ 保存 Collection 元数据到 _collections 表
    └─ SyncRecordTableSchema(new, old)
         │
         ▼ [collection_record_table_sync.go]
         ├─ 新建: CreateTable + CreateIndexes
         └─ 更新: DropIndexes → RenameTable → DropColumns → Add/RenameColumns
                 → normalizeSingleVsMultipleFieldChanges → CreateIndexes
    │
    ▼
OnCollectionAfterCreateSuccess → ReloadCachedCollections()
    │
    ▼
Automigrate 插件（如启用）: 生成迁移代码文件
```

---

## 八、关键设计要点

1. **Field ID 作为稳定标识**：字段改名通过 ID 匹配识别，而不是 Name，避免数据丢失。这也是 `FieldsList.add()` 中无 ID 时按 Name 匹配并复用原 ID 的原因。

2. **临时列名策略**：列新增/改名采用"临时名→真名"两步法，处理 `A↔B` 互换等冲突场景。单值/多值切换时临时列名前缀为 `_` + 原名。

3. **SQLite 功能规避**：
   - 不直接 ALTER COLUMN 类型，而是 "改名旧列→建新列→数据转换→删旧列"
   - 修改列时临时删除所有视图，完成后恢复
   - 视图删除/恢复是逐字段执行的（每次处理一个 MultiValuer 字段变更都会循环一次）

4. **MultiValuer 仅 3 种字段**：SelectField、RelationField、FileField 通过 `MaxSelect > 1` 判断是否多值。其他字段（包括 json、geoPoint）不支持单/多值切换。

5. **Hook 分层执行**：
   - Priority < 0：用户自定义 Hook 先执行
   - Priority > 0（99）：系统 Hook 后执行，确保验证/同步在用户逻辑之后
   - SaveExecute 在 Priority 99（最靠近 DB），最小化事务锁时间

6. **索引容错**：验证阶段不检查表名（`TableName = "validator"`），实际创建时强制覆盖为当前集合名，允许部分修改集合名时索引定义不更新。

7. **双重缓存**：`ReloadCachedCollections()` 在成功和失败时都触发，失败时回滚到之前的缓存状态。
