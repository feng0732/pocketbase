# PocketBase 列表过滤 DSL 到 SQL 查询生成路径分析

本文档深入分析 PocketBase 过滤 DSL 从用户输入字符串到最终 SQL WHERE 子句的完整转换路径，重点阐明多值关系匹配的双重否定语义、标准运算符 vs `?` any 运算符的差异，并使用仓库相对路径确保文档可迁移。

> **路径约定**：文中所有代码引用均使用仓库相对路径（如 `./tools/search/filter.go`），在 GitHub、VS Code 等环境中可直接点击跳转。

---

## 一、整体架构与调用流程

### 1.1 入口层：HTTP API → Provider

用户列表查询请求首先进入 `recordsList` 处理器 [apis/record_crud.go](./apis/record_crud.go#L36-L128)：

```
HTTP GET /api/collections/{collection}/records?filter=...
  ↓
core.NewRecordFieldResolver(app, collection, requestInfo, allowHiddenFields)
  ↓
search.NewProvider(fieldsResolver).Query(query)
  ↓
Provider.ParseAndExec(urlQuery, &records)
```

`search.Provider` [tools/search/provider.go](./tools/search/provider.go#L64-L75) 是整个搜索管线的协调器，负责解析 URL query 参数、构建过滤表达式、调用 FieldResolver 注入 JOIN、并发执行 COUNT 和数据查询。

### 1.2 解析层：FilterData → fexpr AST

`FilterData.BuildExprWithLimit` [tools/search/filter.go](./tools/search/filter.go#L51-L105) 是过滤解析的核心入口：

```
原始 filter 字符串
  ↓
1. 替换占位符参数 {:name} → 安全引用后的值
2. 查 parsedFilterData 缓存（最多 500 条 LRU）
3. fexpr.Parse(raw) → 生成 []fexpr.ExprGroup AST
4. buildParsedFilterExpr(data, fieldResolver, &maxExpressions)
  ↓
dbx.Expression
```

使用第三方库 `github.com/ganigeorgiev/fexpr` 进行词法和语法分析，将 DSL 字符串解析为结构化的表达式组树（ExprGroup），支持：
- 比较运算符：`=` `!=` `>` `<` `>=` `<=` `~` (LIKE) `!~` (NOT LIKE)
- **any/optional 运算符**：`?=` `?!=` `?>` `?<` `?>=` `?<=` `?~` `?!~`
- 逻辑运算符：`&&` (AND) `||` (OR)
- 括号分组、字面量、标识符、函数调用（`geoDistance()`、`strftime()`）

### 1.3 表达式构建层：AST → SQL Expression

`buildParsedFilterExpr` [tools/search/filter.go](./tools/search/filter.go#L107-L153) 递归遍历 AST，对每个二元表达式调用 `resolveTokenizedExpr` → `resolveToken` → `buildResolversExpr` 组合成最终 SQL。

---

## 二、字段解析（Field Resolution）

### 2.1 解析器层次结构

PocketBase 采用两级 FieldResolver 设计：

| 解析器 | 文件 | 用途 |
|--------|------|------|
| `SimpleFieldResolver` | [tools/search/simple_field_resolver.go](./tools/search/simple_field_resolver.go) | 通用场景，仅支持白名单字段和 JSON 路径 |
| `RecordFieldResolver` | [core/record_field_resolver.go](./core/record_field_resolver.go) | Record 专用，支持关系、@request、@collection、修饰符等高级特性 |

### 2.2 字段名白名单校验

`RecordFieldResolver` 初始化时内置正则白名单 [core/record_field_resolver.go](./core/record_field_resolver.go#L97-L106)：

```go
[]string{
    `^\w+[\w\.\:]*$`,                          // 普通字段路径
    `^\@request\.context$`,                     // 请求上下文
    `^\@request\.method$`,                      // 请求方法
    `^\@request\.auth\.[\w\.\:]*\w+$`,          // 认证用户字段
    `^\@request\.body\.[\w\.\:]*\w+$`,          // 请求体字段
    `^\@request\.query\.[\w\.\:]*\w+$`,         // 查询参数
    `^\@request\.headers\.[\w\.\:]*\w+$`,       // 请求头
    `^\@collection\.\w+(\:\w+)?\.[\w\.\:]*\w+$`, // 跨集合引用
}
```

字段名首先通过 `list.ExistInSliceWithRegex` 校验，未命中则直接拒绝。

### 2.3 Runner 状态机解析

`RecordFieldResolver.Resolve` 委托给 `parseAndRun` [core/record_field_resolver_runner.go](./core/record_field_resolver_runner.go#L36-L43) → `runner.run()`，核心状态字段：

```go
type runner struct {
    activeProps                []string        // 待处理的属性路径段
    activeCollectionName       string          // 当前所在集合名
    activeTableAlias           string          // 当前表别名（用于 JOIN）
    withMultiMatch             bool            // 是否需要附加 MultiMatchSubquery
    multiMatch                 *MultiMatchSubquery
    multiMatchActiveTableAlias string
}
```

### 2.4 字段路径分派

`runner.run()` [core/record_field_resolver_runner.go](./core/record_field_resolver_runner.go#L61-L130) 根据路径首段分派：

| 前缀 | 处理函数 | 说明 |
|------|----------|------|
| `@collection` | `processCollectionField()` | 非关系型跨集合 JOIN |
| `@request.auth.` | `processRequestAuthField()` | 认证用户字段，可能 JOIN auth 表 |
| `@request.body.` + 关系字段 | `processRequestBodyRelationField()` | 请求体中的关系 ID 字段 |
| `@request.*` | `resolveStaticRequestField()` | 从静态请求信息取值 |
| 其他 | `processActiveProps()` | 普通/关系字段路径 |

### 2.5 修饰符（Modifiers）

字段末尾可用 `:` 附加修饰符，由 `splitModifier` [core/record_field_resolver.go](./core/record_field_resolver.go#L573-L591) 解析：

| 修饰符 | 适用字段 | 效果 |
|--------|----------|------|
| `:each` | 多值字段 | 通过 `json_each()` 展开数组为行，标识符变为 `[[jeAlias.value]]` |
| `:length` | 多值字段 | 返回 `JSON_ARRAY_LENGTH(column)` |
| `:lower` | 文本字段 | 包裹 `LOWER(identifier)` |
| `:isset` | `@request.*` | 返回 TRUE/FALSE 表示字段是否存在 |
| `:changed` | `@request.body.*` | 替换为 `isset=true && bodyField != originalField` 复合表达式 |

### 2.6 JSON / GeoPoint 字段处理

遇到 JSON 或 GeoPoint 字段时，剩余路径被当作 JSON path 处理：

```go
// a.b.c 其中 a 是 JSON 字段 → JSON_EXTRACT([[table.a]], '$.b.c')
dbutils.JSONExtract(r.activeTableAlias+"."+cleanFieldName, jsonPathStr)
```

`JSONExtract` [tools/dbutils/json.go](./tools/dbutils/json.go#L36-L51) 做了兼容性封装，即使列存的不是合法 JSON（如纯文本），也能通过临时包装对象正确提取：

```sql
CASE WHEN json_valid([[col]])
  THEN JSON_EXTRACT([[col]], '$path')
  ELSE JSON_EXTRACT(json_object('pb', [[col]]), '$.pb.path')
END
```

### 2.7 标识符宏与内置函数

- **宏**：`@now`、`@todayStart`、`@year`、`@weekday` 等 15 个时间相关宏在 `identifierMacros` [tools/search/identifier_macros.go](./tools/search/identifier_macros.go#L15-L135) 中注册。
- **函数**：`geoDistance()`（Haversine 距离）和 `strftime()` 在 `TokenFunctions` [tools/search/token_functions.go](./tools/search/token_functions.go#L13-L182) 中注册。函数参数同样经过 `resolveToken` 递归解析。

---

## 三、关系展开（Relation Expansion）与多值匹配语义

### 3.1 三种匹配模式的根本差异

PocketBase 的关系过滤存在三种截然不同的语义，是本分析最需要厘清的部分：

| 模式 | 示例 | 数学语义 | 触发条件 |
|------|------|----------|----------|
| **普通标量字段** | `title = 'Go'` | 直接比较，无量化 | 路径中无任何关系字段 |
| **ANY 匹配（存在性）** | `tags.name ?= 'Go'` | **∃** 至少一个关联值满足 | 运算符带 `?` 前缀（`?=` `?!=` `?>` 等） |
| **ALL 匹配（全称量化）** | `tags.name = 'Go'` | **∀** 所有关联值都满足 | 标准运算符 + 路径中存在多值关系或反向关系 |

代码层面的分歧点在 `buildResolversExpr` [tools/search/filter.go](./tools/search/filter.go#L209-L239)：

```go
// multi-match expressions
if !isAnyMatchOp(op) {           // 只有非 ? 运算符才进入
    if left.MultiMatchSubQuery != nil && right.MultiMatchSubQuery != nil {
        mm := &manyVsManyExpr{...}
        expr = dbx.Enclose(dbx.And(expr, mm))     // 附加全称量化约束
    } else if left.MultiMatchSubQuery != nil {
        mm := &manyVsOneExpr{...}
        expr = dbx.Enclose(dbx.And(expr, mm))     // 附加全称量化约束
    } else if right.MultiMatchSubQuery != nil {
        mm := &manyVsOneExpr{...}
        expr = dbx.Enclose(dbx.And(expr, mm))     // 附加全称量化约束
    }
}
```

`isAnyMatchOp` [tools/search/filter.go](./tools/search/filter.go#L450-L461) 对所有 `?` 前缀运算符返回 `true`，直接跳过 Multi-Match 附加逻辑。

### 3.2 正向关系 JOIN

#### 单值关系（MaxSelect ≤ 1）

```sql
LEFT JOIN [[related_table]] [[alias]] ON [[alias.id]] = [[current_table.a]]
```

单值关系不触发 `withMultiMatch`（因为一条记录最多关联一条）。

#### 多值关系（MaxSelect > 1）

关系值存储为 JSON 数组，需要两次 JOIN：

```sql
LEFT JOIN json_each(...) [[jeAlias]]      -- 展开 JSON 数组
LEFT JOIN [[related_table]] [[alias]] ON [[alias.id]] = [[jeAlias.value]]
```

`JSONEach` [tools/dbutils/json.go](./tools/dbutils/json.go#L10-L17) 做了归一化处理，兼容非 JSON 列。多值关系必然触发 `withMultiMatch = true`。

### 3.3 反向关系 JOIN

通过 `collectionName_via_fieldName` 语法访问反向引用，匹配正则 `^(\w+)_via_(\w+)$` [core/record_field_resolver_runner.go](./core/record_field_resolver_runner.go#L113-L115)。

**单值反向关系**：
```sql
LEFT JOIN [[posts]] [[alias]] ON [[alias.author]] = [[current_table.id]]
```

是否触发 Multi-Match 取决于反向字段是否有**唯一索引**：

```go
// core/record_field_resolver_runner.go:592-596
_, hasUniqueIndex := dbutils.FindSingleColumnUniqueIndex(backCollection.Indexes, backRelField.Name)
r.withMultiMatch = !hasUniqueIndex
```

如果目标集合上的关系字段有单列唯一索引（`demo4_via_rel_one_unique`），一条源记录最多被一条反向记录引用，不需要全称量化。

**多值反向关系**：反向字段本身是多值，用 `IN (SELECT json_each(...))` 连接，必然触发 Multi-Match。

### 3.4 Multi-Match 子查询：全称量化的双重否定实现

这是整个过滤系统最精巧也最容易误解的部分。当使用标准运算符（不带 `?`）过滤多值关系时，PocketBase 生成 **「直接表达式」 AND 「NOT EXISTS 子查询」** 的复合结构。

#### 3.4.1 结构总览

以 `self_rel_many.self_rel_one > true` 为例，最终 WHERE 条件为：

```sql
WHERE (
  -- ① 直接表达式（行级存在性过滤）
  ([[demo4_self_rel_many.self_rel_one]] > 1)

  AND

  -- ② 全称量化子查询（双重否定）
  (NOT EXISTS (
    SELECT 1
    FROM (
      -- MultiMatchSubquery：针对每条外层记录，取其所有关联值
      SELECT [[__mm_demo4_self_rel_many.self_rel_one]] as [[multiMatchValue]]
      FROM `demo4` `__mm_demo4`
      LEFT JOIN json_each(...) `__mm_demo4_self_rel_many_je`
      LEFT JOIN `demo4` `__mm_demo4_self_rel_many`
        ON [[__mm_demo4_self_rel_many.id]] = [[__mm_demo4_self_rel_many_je.value]]
      WHERE `__mm_demo4`.`id` = `demo4`.`id`   -- 关联到外层当前行
    ) {{__smTEST}}
    WHERE NOT ([[__smTEST.multiMatchValue]] > 1)   -- 找反例
  ))
)
```

#### 3.4.2 manyVsOneExpr：多对一全称量化

`manyVsOneExpr.Build` [tools/search/filter.go](./tools/search/filter.go#L689-L726) 的核心是双重否定：

```
NOT EXISTS (子查询 WHERE NOT(条件))
```

逻辑推导：
- 子查询返回当前父记录的**全部**关联值（不仅是 LEFT JOIN 匹配到的那些）
- `WHERE NOT(条件)` 筛选出**不满足条件**的关联值
- `NOT EXISTS` 断言不存在这样的反例 → **所有关联值都满足条件**

关键细节：`r1.AfterBuild = dbx.Not` 在内层表达式构建完成后取反，形成 NOT(条件)。

完整 SQL 模板：
```sql
NOT EXISTS (
    SELECT 1 FROM (<MultiMatchSubquery>) {{__smXXXXXXXX}}
    WHERE NOT (<multiMatchValue> <op> <otherOperand>)
)
```

其中 `MultiMatchSubquery` [tools/search/multi_match_subquery.go](./tools/search/multi_match_subquery.go) 是关联子查询——对每条外层记录独立执行，返回该记录通过完整关系链能到达的所有值。

#### 3.4.3 manyVsManyExpr：两侧都是多值关系

`manyVsManyExpr.Build` [tools/search/filter.go](./tools/search/filter.go#L632-L667) 处理两侧均为多值关系的场景，使用 LEFT JOIN + 双重否定：

```sql
NOT EXISTS (
    SELECT 1
    FROM (<leftSubQuery>) {{__mlXXXXXXXX}}
    LEFT JOIN (<rightSubQuery>) {{__mrXXXXXXXX}}
    WHERE NOT (left.multiMatchValue <op> right.multiMatchValue)
)
```

逻辑推导：
- `LEFT JOIN` 无 ON 子句 → 产生两个集合的笛卡尔积
- `WHERE NOT(条件)` 筛选出不满足比较条件的值对
- `NOT EXISTS` 断言不存在这样的反例对 → **左右集合中每个值两两之间都满足条件**

实际效果：对于 `=`，断言两集合的每个值都相等（即两集合都是同一值的重复）；对于 `>`，断言左集合的每个值都大于右集合的每个值。

#### 3.4.4 为什么需要直接表达式 + 全称量化的 AND？

直觉上只用 NOT EXISTS 就够了，但 PocketBase 同时保留了直接的 JOIN 行级表达式：

```go
expr = dbx.Enclose(dbx.And(expr, mm))   // expr 是直接比较，mm 是全称量化
```

原因是 LEFT JOIN 语义：
1. **直接表达式**处理 JOIN 产生的行，配合 `SELECT DISTINCT` 实现存在性（至少一个关联值匹配）
2. **全称量化**过滤掉"部分匹配但不全匹配"的记录
3. 两者 AND 后 = **所有关联值都满足条件，且关联非空**

如果关系为空（JSON 数组为 `[]`），LEFT JOIN 产生 NULL 行，直接表达式不匹配，记录被排除。

### 3.5 三种模式的 SQL 输出对比

以下测试均来自 [core/record_field_resolver_test.go](./core/record_field_resolver_test.go)，查询集合为 `demo4`：

| DSL 表达式 | 模式 | WHERE 子句核心 | 行为解释 |
|-----------|------|---------------|---------|
| `title > true` | 标量字段 | `[[demo4.title]] > 1` | 无 JOIN，直接列比较 |
| `self_rel_one.title > true` | 单值关系 + 标准运算符 | `[[demo4_self_rel_one.title]] > 1` | 单值关系不触发 Multi-Match，等价于 ANY |
| `self_rel_many.title ?> 'test'` | 多值关系 + ANY 运算符 | `[[demo4_self_rel_many.title]] = {:TEST}` | 仅 LEFT JOIN + DISTINCT，**存在至少一个匹配**即可返回 |
| `self_rel_many.title = 'test'` | 多值关系 + 标准运算符 | `( [[...]] = {:TEST} AND NOT EXISTS( WHERE NOT( [[...]] = {:TEST} ) ) )` | LEFT JOIN 做存在性 + NOT EXISTS 做全称量化 → **所有关联记录都必须匹配** |
| `self_rel_many.self_rel_one > true` | 嵌套多值关系 + 标准运算符 | `(直接表达式 AND NOT EXISTS(子查询 WHERE NOT(...)) )` | 完整关系链上的所有路径值都必须满足 |
| `demo4_via_rel_one_unique.id = true` | 反向单值 + 唯一索引 | `[[demo3_demo4_via_rel_one_unique.id]] = 1` | 唯一索引保证一对一，无需全称量化 |
| `demo4_via_rel_one_cascade.id = true` | 反向单值 + 无唯一索引 | `(直接表达式 AND NOT EXISTS(...))` | 多条反向记录可能指向同一目标 → 需要全称量化 |

### 3.6 特殊优化：单关系 ID 查找

当路径为 `user.id`（正向单关系且末段是 `id`）时，直接使用关系字段本身的值，不生成额外 JOIN [core/record_field_resolver_runner.go](./core/record_field_resolver_runner.go#L653-L657)：

```go
if !relField.IsMultiple() && i == totalProps-2 && r.activeProps[i+1] == FieldNameId {
    return r.finalizeActivePropsProcessing(collection, relField.Name, i)
    // 返回 [[current_table.user]] 而不是 [[alias.id]]
}
```

### 3.7 ListRule 级联

当 JOIN 目标集合有非空 ListRule 且 `allowHiddenFields=false` 时，通过 `updateQueryWithCollectionListRule` [core/record_field_resolver.go](./core/record_field_resolver.go#L160-L208) 将 ListRule 作为额外 AND 条件绑定到**最外层查询**：

```go
// 构造 id='' || (\nRULE\n) 包裹，保证空关系不影响其他 OR 分支
expr, _ := FilterData("id='' || (\n" + *c.ListRule + "\n)").BuildExpr(&cloneR)
query.AndWhere(expr)
```

这是**顶层绑定**（不在子查询内部），注释明确说明这是出于安全考虑——防止侧信道攻击通过 timing 差异推断受保护数据。

### 3.8 嵌套深度限制

`maxNestedRels = 6` [core/record_field_resolver_runner.go](./core/record_field_resolver_runner.go#L22)，超过直接报错，防止滥用关系链导致性能爆炸。

---

## 四、索引利用策略

### 4.1 NULL 比较优化：避免 COALESCE

`resolveEqualExpr` [tools/search/filter.go](./tools/search/filter.go#L328-L410) 是索引友好设计的核心：

**问题**：SQLite 中 `COALESCE(col, '') = ''` 会阻止索引使用（全表扫描），而 `col = '' OR col IS NULL` 可以走索引 seek。

**策略**（针对 `=` 和 `!=` 运算）：

| 场景 | 生成的 SQL |
|------|-----------|
| 任一侧是非空常量（如 `'abc'`、数字） | `col = 'abc'` 直接比较 |
| 一侧是空字符串/NULL | `(col = '' OR col IS NULL)` |
| 两侧都是可能空的列 | 回退到 `COALESCE(a, '') = COALESCE(b, '')` |
| JSON 字段（NullFallbackDisabled） | `a IS b` 或 `a IS NOT b` |

对于 `!=`，始终使用 `IS NOT` 而非 `<>`，因为 SQLite 中 `'value' <> nullableColumn` 在 nullableColumn 为 NULL 时返回 NULL 而非 TRUE。

### 4.2 唯一索引检测决定 Multi-Match

反向单值关系场景中，`FindSingleColumnUniqueIndex` [tools/dbutils/index.go](./tools/dbutils/index.go#L197-L208) 通过正则解析 `CREATE INDEX` 语句提取列名和 UNIQUE 标记。若目标字段有唯一索引，则跳过全称量化子查询。

### 4.3 去重策略：DISTINCT vs GROUP BY

`updateQueryWithDeduplicateConstraint` [core/record_field_resolver.go](./core/record_field_resolver.go#L210-L242) 目前直接用 `DISTINCT(true)`。

代码中注释提到最初尝试过在安全条件下用 `GROUP BY id`（单列 GROUP BY 比多列 DISTINCT 更高效），但发现会**阻止 ORDER BY 索引利用**——SQLite 的 GROUP BY 执行顺序可能导致它放弃使用 ORDER BY 相关索引（见 [discussion #7461](https://github.com/pocketbase/pocketbase/discussions/7461)）。

### 4.4 COUNT 优化：使用 _rowid_

非视图集合上，Provider 默认用 `_rowid_` 作为计数列 [apis/record_crud.go](./apis/record_crud.go#L83-L85)：

```go
if !collection.IsView() {
    searchProvider.CountCol("_rowid_")
}
```

SQLite 的 `_rowid_` 是内置主键别名，无需额外索引即可高效计数。

### 4.5 LIKE 通配符自动转义

`wrapLikeParams` [tools/search/filter.go](./tools/search/filter.go#L484-L498)：
- 用户未显式使用 `%` 时，自动包裹为 `%value%`（contains 语义）
- 自动转义 `\`、`%`、`_` 特殊字符（使用 `\` 作为 ESCAPE）

---

## 五、注入防护机制

### 5.1 参数化查询（基础防线）

所有用户提供的字面量绝不直接拼 SQL，一律走 dbx 参数绑定：

- **字符串/数字字面量**：`resolveToken` 遇到 `TokenText` / `TokenNumber`，生成 `{:tXXXXXXXX}` 占位符，值存入 `dbx.Params`
- **标识符宏**（`@now` 等）：同样用 `{:tXXXXXXXX}` 占位符
- **`@request.*` 静态值**：用 `{:fXXXXXXXXXX}` 占位符
- **表别名**：用 `security.PseudorandomString(8)` 生成随机后缀，同时避免别名冲突和可预测性

`security.PseudorandomString` 使用密码学安全随机源 [tools/security/random.go](./tools/security/random.go)。

### 5.2 占位符替换安全

FilterData 中 `{:name}` 形式的外部参数在 `BuildExprWithLimit` [tools/search/filter.go](./tools/search/filter.go#L59-L80) 预处理阶段替换：

```go
switch v := value.(type) {
case nil:
    replacement = "null"
case bool, float64, ...int types...:
    replacement = cast.ToString(v)    // 数字直接转字符串，无注入风险
default:
    replacement = strconv.Quote(cast.ToString(v)) // Go 标准库安全转义并加引号
}
raw = strings.ReplaceAll(raw, "{:"+key+"}", replacement)
```

注意：此阶段在 `fexpr.Parse` 之前执行，替换后的值是 DSL 语法的一部分（被 fexpr 重新词法分析），不是直接拼 SQL。

### 5.3 字段名校验

- **白名单正则**：所有标识符先通过 `RecordFieldResolver.allowedFields` 正则校验
- **隐藏字段**：非 superuser 且 `allowHiddenFields=false` 时，`field.GetHidden()` 的字段直接拒绝
- **邮箱字段特殊限制**：非 superuser 过滤 auth 集合的 email 字段时，自动附加 `emailVisibility = TRUE` 条件（`AfterBuild` 钩子 [core/record_field_resolver_runner.go](./core/record_field_resolver_runner.go#L823-L831)）
- **不存在的字段**：非 `@request.*` 路径返回错误，`@request.*` 优雅降级为 NULL

### 5.4 表达式资源限制

| 限制 | 常量 | 值 | 位置 |
|------|------|----|------|
| 最大表达式数量 | `DefaultFilterExprLimit` | 200 | [tools/search/provider.go](./tools/search/provider.go#L21) |
| 最大 filter 字符串长度 | `MaxFilterLength` | 3500 | [tools/search/provider.go](./tools/search/provider.go#L30) |
| 最大 sort 表达式数 | `DefaultSortExprLimit` | 8 | [tools/search/provider.go](./tools/search/provider.go#L24) |
| 最大 sort 字段长度 | `MaxSortFieldLength` | 255 | [tools/search/provider.go](./tools/search/provider.go#L33) |
| 最大嵌套关系深度 | `maxNestedRels` | 6 | [core/record_field_resolver_runner.go](./core/record_field_resolver_runner.go#L22) |
| strftime 最大参数 | — | 10 | [tools/search/token_functions.go](./tools/search/token_functions.go#L89-L91) |
| 展开查询限制 | — | 1000 条 | [core/record_query_expand.go](./core/record_query_expand.go#L108) |
| 每页最大数量 | `MaxPerPage` | 1000 | [tools/search/provider.go](./tools/search/provider.go#L27) |
| 解析缓存上限 | — | 500 | [tools/search/filter.go](./tools/search/filter.go#L102) |

### 5.5 ListRule 顶层绑定

如 3.7 节所述，关联集合的 ListRule 被绑定在最外层 `AND WHERE`，而非子查询内部。防止侧信道攻击。

### 5.6 定时攻击防护（Timing Attack Mitigation）

在 `recordsList` [apis/record_crud.go](./apis/record_crud.go#L104-L122) 中，当满足以下全部条件时触发随机延迟（0-500ms）：
1. 非 superuser 请求
2. 集合有非空 ListRule
3. 请求携带了 filter 参数
4. 返回结果为空
5. 该集合已触发 3 次/3秒 的限流

代码注释说明这不是完美防护，但配合网络延迟在实践中足以提高攻击门槛。真正敏感的字段（password、tokenKey）从根本上就不允许客户端过滤，且必要时使用**常数时间比较**。

### 5.7 列名引用规范

所有标识符（表名、列名、别名）统一使用 `[[...]]` 语法（PocketBase 的 dbx 方言），最终由 dbx 层转为数据库特定的引用方式（SQLite 用双引号），避免标识符注入。

---

## 六、完整调用链图示

```
用户输入: self_rel_many.title = 'test'  (demo4 集合)
│
├─ Provider.ParseAndExec()
│   ├─ Parse URL query
│   └─ Exec()
│       ├─ FilterData.BuildExprWithLimit()
│       │   ├─ fexpr.Parse() → AST: [Expr{Left:self_rel_many.title, Op:=, Right:'test'}]
│       │   └─ buildParsedFilterExpr()
│       │       └─ Expr: self_rel_many.title = 'test'
│       │           ├─ resolveToken("self_rel_many.title")
│       │           │   └─ RecordFieldResolver.Resolve()
│       │           │       └─ runner.run()
│       │           │           ├─ processActiveProps(["self_rel_many", "title"])
│       │           │           │   ├─ self_rel_many: 多值 RelationField
│       │           │           │   │   ├─ 注册 LEFT JOIN json_each(...) + LEFT JOIN demo4
│       │           │           │   │   ├─ 更新 activeTableAlias = "demo4_self_rel_many"
│       │           │           │   │   └─ 设置 withMultiMatch = true
│       │           │           │   └─ title: TextField (末段)
│       │           │           │       └─ 返回 "[[demo4_self_rel_many.title]]"
│       │           │           ├─ build MultiMatchSubquery (关联子查询)
│       │           │           └─ 返回 ResolverResult{
│       │           │                 Identifier: "[[demo4_self_rel_many.title]]",
│       │           │                 MultiMatchSubQuery: <关联子查询>
│       │           │              }
│       │           ├─ resolveToken("'test'") → "{:tXxXxXxXx}" Params: {"tXxXxXxXx": "test"}
│       │           └─ buildResolversExpr()
│       │               ├─ resolveEqualExpr(true, ...) → 直接表达式
│       │               ├─ isAnyMatchOp(=) → false → 进入 multi-match 分支
│       │               ├─ 构造 manyVsOneExpr{op:=, subQuery:..., otherOperand:'test'}
│       │               └─ dbx.And(直接表达式, manyVsOneExpr)
│       │                  → "[[demo4_self_rel_many.title]] = {:tXxXxXxXx}
│       │                     AND NOT EXISTS (SELECT 1 FROM (
│       │                       SELECT [[__mm_demo4_self_rel_many.title]] as multiMatchValue
│       │                       FROM demo4 __mm_demo4 LEFT JOIN json_each(...) ...
│       │                       WHERE __mm_demo4.id = demo4.id
│       │                     ) __smX WHERE NOT (__smX.multiMatchValue = {:tXxXxXxXx})
│       │                    )"
│       │
│       ├─ fieldsResolver.UpdateQuery(query)
│       │   ├─ 注入 LEFT JOIN json_each(...) + LEFT JOIN demo4
│       │   └─ 启用 DISTINCT(true)（因为有 JOIN）
│       │
│       ├─ [并发] countExec → COUNT(DISTINCT _rowid_)
│       └─ [并发] modelsExec → LIMIT + OFFSET 取出数据
│
└─ 返回 Result{Items, Page, PerPage, TotalItems, TotalPages}
```

---

## 七、关键文件索引

| 关注点 | 路径 |
|--------|------|
| 过滤 DSL 解析核心（表达式构建、相等比较、multi-match 双重否定） | [tools/search/filter.go](./tools/search/filter.go) |
| 搜索 Provider 协调器（Parse/Exec、资源限制） | [tools/search/provider.go](./tools/search/provider.go) |
| Token/函数解析（geoDistance、strftime） | [tools/search/token_functions.go](./tools/search/token_functions.go) |
| 标识符宏（@now、@year 等时间宏） | [tools/search/identifier_macros.go](./tools/search/identifier_macros.go) |
| 通用字段解析器（白名单 + JSON 路径） | [tools/search/simple_field_resolver.go](./tools/search/simple_field_resolver.go) |
| Multi-Match 关联子查询构建 | [tools/search/multi_match_subquery.go](./tools/search/multi_match_subquery.go) |
| Record 字段解析器主文件（白名单、ListRule 绑定、去重） | [core/record_field_resolver.go](./core/record_field_resolver.go) |
| Record 字段解析 Runner（状态机、关系 JOIN、修饰符、邮箱限制） | [core/record_field_resolver_runner.go](./core/record_field_resolver_runner.go) |
| 表达式替换工具 | [core/record_field_resolver_replace_expr.go](./core/record_field_resolver_replace_expr.go) |
| JSON 工具函数（JSONExtract、JSONEach 归一化） | [tools/dbutils/json.go](./tools/dbutils/json.go) |
| 索引解析工具（FindSingleColumnUniqueIndex） | [tools/dbutils/index.go](./tools/dbutils/index.go) |
| Record 查询构建 | [core/record_query.go](./core/record_query.go) |
| 关系展开（expand 参数处理） | [core/record_query_expand.go](./core/record_query_expand.go) |
| API 层列表入口（定时攻击防护、_rowid_ 计数） | [apis/record_crud.go](./apis/record_crud.go) |
| 词法分析器基础 | [tools/tokenizer/tokenizer.go](./tools/tokenizer/tokenizer.go) |
| 多值关系语义测试用例（any vs multi-match 对比） | [core/record_field_resolver_test.go](./core/record_field_resolver_test.go) |
