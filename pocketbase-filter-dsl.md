# PocketBase 列表过滤 DSL 到 SQL 查询生成路径分析

本文档深入分析 PocketBase 过滤 DSL（Domain Specific Language）从用户输入字符串到最终 SQL WHERE 子句的完整转换路径，涵盖字段解析、关系展开、索引利用和注入防护四个核心维度。

---

## 一、整体架构与调用流程

### 1.1 入口层：HTTP API → Provider

用户的列表查询请求首先进入 [recordsList](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/apis/record_crud.go#L36-L128) 处理器：

```
HTTP GET /api/collections/{collection}/records?filter=...
  ↓
core.NewRecordFieldResolver(app, collection, requestInfo, true)
  ↓
search.NewProvider(fieldsResolver).Query(query)
  ↓
Provider.ParseAndExec(urlQuery, &records)
```

[search.Provider](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/search/provider.go#L64-L75) 是整个搜索管线的协调器，负责：
- 解析 URL query 参数（filter、sort、page、perPage）
- 构建过滤表达式并绑定到 dbx.SelectQuery
- 调用 FieldResolver 更新查询（注入 JOIN）
- 执行 COUNT 和数据查询（并发执行）

### 1.2 解析层：FilterData → fexpr AST

[FilterData.BuildExprWithLimit](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/search/filter.go#L51-L105) 是过滤解析的核心入口：

```
原始 filter 字符串
  ↓
1. 替换占位符参数 {:name} → 安全引用后的值
2. 查 parsedFilterData 缓存（最多 500 条）
3. fexpr.Parse(raw) → 生成 []fexpr.ExprGroup AST
4. buildParsedFilterExpr(data, fieldResolver, &maxExpressions)
  ↓
dbx.Expression
```

使用第三方库 `github.com/ganigeorgiev/fexpr` 进行词法和语法分析，将 DSL 字符串解析为结构化的表达式组树（ExprGroup），支持：
- 比较运算符：`=` `!=` `>` `<` `>=` `<=` `~` (LIKE) `!~` (NOT LIKE)
- 逻辑运算符：`&&` (AND) `||` (OR)
- 括号分组
- 字面量：字符串（`'text'`）、数字、布尔、NULL
- 标识符（字段名）
- 函数调用：`geoDistance()`、`strftime()`

### 1.3 表达式构建层：AST → SQL Expression

[buildParsedFilterExpr](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/search/filter.go#L107-L153) 递归遍历 AST：

```
foreach ExprGroup:
  switch group.Item:
    case fexpr.Expr:          → resolveTokenizedExpr()
    case fexpr.ExprGroup:     → 递归 buildParsedFilterExpr()
    case []fexpr.ExprGroup:   → 递归 buildParsedFilterExpr()
  ↓
用 AND/OR 连接各表达式，包裹为 concatExpr
```

单个二元表达式通过 [resolveTokenizedExpr](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/search/filter.go#L155-L167) 处理：分别解析左、右 operand（调用 `resolveToken`），然后通过 `buildResolversExpr` 组合成最终 SQL。

---

## 二、字段解析（Field Resolution）

### 2.1 解析器层次结构

PocketBase 采用两级 FieldResolver 设计：

| 解析器 | 文件 | 用途 |
|--------|------|------|
| `SimpleFieldResolver` | [simple_field_resolver.go](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/search/simple_field_resolver.go) | 通用场景，仅支持白名单字段和 JSON 路径 |
| `RecordFieldResolver` | [record_field_resolver.go](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/core/record_field_resolver.go) | Record 专用，支持关系、@request、@collection、修饰符等高级特性 |

### 2.2 字段名白名单校验

`RecordFieldResolver` 初始化时内置了正则白名单 [allowedFields](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/core/record_field_resolver.go#L97-L106)：

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

`RecordFieldResolver.Resolve` 委托给 [parseAndRun](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/core/record_field_resolver_runner.go#L36-L43) → `runner.run()`，后者是一个完整的状态机，核心状态字段：

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

[runner.run()](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/core/record_field_resolver_runner.go#L61-L130) 根据路径首段分派：

| 前缀 | 处理函数 | 说明 |
|------|----------|------|
| `@collection` | `processCollectionField()` | 非关系型跨集合 JOIN |
| `@request.auth.` | `processRequestAuthField()` | 认证用户字段，可能 JOIN auth 表 |
| `@request.body.` + 关系字段 | `processRequestBodyRelationField()` | 请求体中的关系 ID 字段 |
| `@request.*` | `resolveStaticRequestField()` | 从静态请求信息取值 |
| 其他 | `processActiveProps()` | 普通/关系字段路径 |

### 2.5 普通字段路径处理

[processActiveProps](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/core/record_field_resolver_runner.go#L441-L734) 逐段遍历路径：

1. **中间段**：必须是 RelationField，为其生成 LEFT JOIN（正向或反向），更新 `activeTableAlias`
2. **最后一段**：调用 `finalizeActivePropsProcessing`，根据字段类型和修饰符生成最终标识符

### 2.6 修饰符（Modifiers）

字段末尾可用 `:` 附加修饰符，由 [splitModifier](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/core/record_field_resolver.go#L573-L591) 解析：

| 修饰符 | 适用字段 | 效果 |
|--------|----------|------|
| `:each` | 多值字段（Relation、Select 等） | 通过 `json_each()` 将数组展开为行，字段标识符变为 `[[jeAlias.value]]` |
| `:length` | 多值字段 | 返回 `JSON_ARRAY_LENGTH(column)` |
| `:lower` | 文本字段 | 包裹 `LOWER(identifier)` |
| `:isset` | `@request.*` | 返回 TRUE/FALSE 表示字段是否存在 |
| `:changed` | `@request.body.*` | 替换为 `isset=true && bodyField != originalField` 复合表达式 |

### 2.7 JSON / GeoPoint 字段处理

遇到 JSON 或 GeoPoint 字段时，剩余路径被当作 JSON path 处理：

```go
// a.b.c 其中 a 是 JSON 字段 → JSON_EXTRACT([[table.a]], '$.b.c')
dbutils.JSONExtract(r.activeTableAlias+"."+cleanFieldName, jsonPathStr)
```

[JSONExtract](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/dbutils/json.go#L36-L51) 内部做了兼容性封装：

```sql
CASE WHEN json_valid([[col]])
  THEN JSON_EXTRACT([[col]], '$path')
  ELSE JSON_EXTRACT(json_object('pb', [[col]]), '$.pb.path')
END
```

即使列存的不是合法 JSON（如纯文本），也能通过临时包装对象正确提取。

### 2.8 标识符宏（Macros）

在 [resolveToken](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/search/filter.go#L261-L320) 中，`TokenIdentifier` 先检查 [identifierMacros](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/search/identifier_macros.go#L15-L135)：

| 宏 | 值 |
|----|----|
| `@now` | 当前 UTC 时间（DateTime 字符串） |
| `@todayStart` / `@todayEnd` | 今日起止时间 |
| `@year` / `@month` / `@day` / `@hour` / `@minute` / `@second` / `@weekday` | 当前时间各部分 |
| `@yesterday` / `@tomorrow` | 昨天/明天 |
| `@monthStart` / `@monthEnd` / `@yearStart` / `@yearEnd` | 时间段边界 |

### 2.9 内置函数

[TokenFunctions](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/search/token_functions.go#L13-L182) 注册了两个函数：

- **`geoDistance(lonA, latA, lonB, latB)`**：Haversine 公式计算两点距离（公里），转换为一大串数学运算 SQL
- **`strftime(format, [timeValue, modifiers...])`**：包装 SQLite strftime，自动应用 NULL 规范化，对多值关系附加 MultiMatchSubquery

函数调用的参数同样经过 `resolveToken` 递归解析，保证参数也是安全的标识符或参数化值。

---

## 三、关系展开（Relation Expansion）

### 3.1 正向关系 JOIN

`a.b.c` 中 `a` 是正向 RelationField：

**单值关系**（MaxSelect ≤ 1）：
```sql
LEFT JOIN [[related_table]] [[alias]] ON [[alias.id]] = [[current_table.a]]
```

**多值关系**（MaxSelect > 1）：关系值存储为 JSON 数组，需要两次 JOIN：
```sql
LEFT JOIN json_each(...) [[jeAlias]]      -- 展开 JSON 数组
LEFT JOIN [[related_table]] [[alias]] ON [[alias.id]] = [[jeAlias.value]]
```

[JSONEach](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/dbutils/json.go#L10-L17) 做了归一化处理，兼容非 JSON 列：
```sql
json_each(CASE WHEN iif(json_valid(col), json_type(col)='array', FALSE)
  THEN col ELSE json_array(col) END)
```

### 3.2 反向关系 JOIN

通过 `collectionName_via_fieldName` 语法访问反向引用。例如 `posts_via_author` 表示"所有指向当前记录的 posts"：

```go
// 匹配正则 ^(\w+)_via_(\w+)$
parts := viaRegex.FindStringSubmatch(prop)  // [ "posts_via_author", "posts", "author" ]
backCollection := loadCollection("posts")
backField      := backCollection.Fields.GetByName("author")  // 必须是 RelationField
```

**单值反向关系**：
```sql
LEFT JOIN [[posts]] [[alias]] ON [[alias.author]] = [[current_table.id]]
```

**多值反向关系**（author 字段是多值）：
```sql
LEFT JOIN [[posts]] [[alias]] ON [[current_table.id]] IN (
  SELECT [[je.value]] FROM json_each([[alias.author]]) [[je]]
)
```

### 3.3 特殊优化：单关系 ID 查找

当路径为 `user.id`（正向单关系且末段是 `id`）时，直接使用关系字段本身的值，不生成额外 JOIN：

```go
// record_field_resolver_runner.go:653-657
if !relField.IsMultiple() && i == totalProps-2 && r.activeProps[i+1] == FieldNameId {
    return r.finalizeActivePropsProcessing(collection, relField.Name, i)
    // 直接返回 [[current_table.user]] 而不是 [[alias.id]]
}
```

### 3.4 Multi-Match 子查询（核心机制）

当涉及多值关系时，简单的 JOIN + WHERE 会产生语义问题（例如"帖子只要有一个标签匹配即返回"vs"所有标签都必须匹配"）。PocketBase 通过 [MultiMatchSubquery](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/search/multi_match_subquery.go) 来保证"多值中存在一个满足条件"的语义。

#### 3.4.1 触发条件

`withMultiMatch = true` 在以下情况被设置：
- 正向多值关系字段
- 反向多值关系字段
- 反向单值关系但目标字段无唯一索引（可能多条记录指向同一目标）
- `@collection` 引用
- `:each` 修饰符作用于多值字段

#### 3.4.2 子查询结构

`MultiMatchSubquery.Build()` 生成：

```sql
SELECT [[resolvedValue]] as [[multiMatchValue]]
FROM [[base_table]] [[base_alias]]
LEFT JOIN ... (关系链)
WHERE [[base_alias.id]] = [[target_table_alias.id]]
```

这是一个**关联子查询**——外层查询的每条记录都会执行一次，返回该记录通过关系链能到达的所有值。

#### 3.4.3 多对一匹配：manyVsOneExpr

当只有一侧有 MultiMatchSubquery（例如 `tags.name = 'Go'`，tags 是多值关系）：

```sql
-- 除了正常的 WHERE [[alias.name]] = 'Go' 之外，附加：
NOT EXISTS (
  SELECT 1 FROM ([[MultiMatchSubquery]]) [[__sm_xxx]]
  WHERE NOT ([[__sm_xxx.multiMatchValue]] = 'Go')
)
```

含义："不存在一个关联值不满足条件" = "所有关联值都满足条件"…… 等等，不对。实际语义是——配合 LEFT JOIN 产生的 NULL 值，这个 NOT EXISTS 确保"至少有一个非 NULL 的匹配"。具体来说：

- LEFT JOIN 产生候选行（可能包含 NULL 关系）
- `expr` 处理普通比较（NULL 会按设计处理）
- `NOT EXISTS + NOT(条件)` 作为 AND 附加：保证当存在真实关联记录时，至少有一条满足条件

#### 3.4.4 多对多匹配：manyVsManyExpr

两侧都是多值关系（例如 `a.tags.name = b.categories.label`）：

```sql
NOT EXISTS (
  SELECT 1 FROM (...) [[__ml_xxx]]
  LEFT JOIN (...) [[__mr_xxx]]
  WHERE [[__ml_xxx.multiMatchValue]] = [[__mr_xxx.multiMatchValue]]
)
```

通过 LEFT JOIN + WHERE 检查两侧子查询结果的交集。

### 3.5 嵌套深度限制

[maxNestedRels = 6](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/core/record_field_resolver_runner.go#L22)，超过直接报错，防止滥用关系链导致性能爆炸。

### 3.6 ListRule 级联

当解析器 `allowHiddenFields = false` 且 JOIN 的目标集合有非空 ListRule 时，通过 [updateQueryWithCollectionListRule](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/core/record_field_resolver.go#L160-L208) 将 ListRule 作为额外 AND 条件绑定到最外层查询：

```go
// 构造 id='' || (\nRULE\n) 包裹，保证空关系不影响其他 OR 分支
expr, _ := FilterData("id='' || (\n" + *c.ListRule + "\n)").BuildExpr(&cloneR)
query.AndWhere(expr)
```

这是**顶层绑定**（不在子查询里），以避免侧信道攻击和数据泄露。

---

## 四、索引利用策略

### 4.1 NULL 比较优化：避免 COALESCE

[resolveEqualExpr](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/search/filter.go#L328-L410) 是索引友好设计的典范。

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

在反向单值关系场景中：

```go
// record_field_resolver_runner.go:592-596
_, hasUniqueIndex := dbutils.FindSingleColumnUniqueIndex(backCollection.Indexes, backRelField.Name)
r.withMultiMatch = !hasUniqueIndex
```

如果目标集合上的关系字段有单列唯一索引，说明一条源记录最多被一条反向记录引用，因此不需要 Multi-Match 检查。

[FindSingleColumnUniqueIndex](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/dbutils/index.go#L197-L208) 通过正则解析 `CREATE INDEX` 语句，提取列名和 UNIQUE 标记。

### 4.3 去重策略：DISTINCT vs GROUP BY

当有 JOIN 时需要去重，[updateQueryWithDeduplicateConstraint](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/core/record_field_resolver.go#L210-L242) 目前直接用 `DISTINCT(true)`。

代码中注释提到最初尝试过在安全条件下用 `GROUP BY id`（因为 GROUP BY 单列比 DISTINCT 多列更高效），但发现会**阻止 ORDER BY 索引利用**，故暂时禁用。GROUP BY 的执行顺序可能导致 SQLite 放弃使用 ORDER BY 相关索引（见 [discussion #7461](https://github.com/pocketbase/pocketbase/discussions/7461)）。

### 4.4 COUNT 优化：使用 _rowid_

在非视图集合上，Provider 默认用 `_rowid_` 作为计数列：

```go
// record_crud.go:83-85
if !collection.IsView() {
    searchProvider.CountCol("_rowid_")
}
```

SQLite 的 `_rowid_` 是内置主键别名，无需额外索引即可高效计数，避免了 "id" 字段需要覆盖索引的问题。

### 4.5 LIKE 通配符自动转义

[wrapLikeParams](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/search/filter.go#L484-L498)：
- 用户未显式使用 `%` 时，自动包裹为 `%value%`（contains 语义）
- 自动转义 `\`、`%`、`_` 特殊字符（使用 `\` 作为 ESCAPE）

生成的 LIKE 表达式：
```sql
[[col]] LIKE {:param} ESCAPE '\'
```

其中参数值已经过 `escapeUnescapedChars` 处理，特殊字符被前缀 `\` 转义。

---

## 五、注入防护机制

### 5.1 参数化查询（基础防线）

所有用户提供的字面量绝不直接拼 SQL，一律走 dbx 参数绑定：

- **字符串/数字字面量**：`resolveToken` 遇到 `TokenText` / `TokenNumber`，生成 `{:tXXXXXXXX}` 占位符，值存入 `dbx.Params`
- **标识符宏**（`@now` 等）：同样用 `{:tXXXXXXXX}` 占位符
- **`@request.*` 静态值**：用 `{:fXXXXXXXXXX}` 占位符
- **表别名**：用 `security.PseudorandomString(8)` 生成随机后缀，避免别名冲突同时也让攻击者难以预测

[security.PseudorandomString](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/security/random.go) 使用密码学安全随机源。

### 5.2 占位符替换安全

FilterData 中 `{:name}` 形式的外部参数在 [BuildExprWithLimit](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/search/filter.go#L59-L80) 预处理阶段替换：

```go
switch v := value.(type) {
case nil:
    replacement = "null"
case bool, float64, ...int types...:
    replacement = cast.ToString(v)  // 数字直接转字符串，无注入风险
default:
    replacement = strconv.Quote(cast.ToString(v))  // 用 Go 标准库安全转义并加引号
}
raw = strings.ReplaceAll(raw, "{:"+key+"}", replacement)
```

注意：此阶段在 `fexpr.Parse` 之前执行，替换后的值是 DSL 语法的一部分（被 fexpr 重新词法分析），不是直接拼 SQL。

### 5.3 字段名校验

- **白名单正则**：所有标识符先通过 `RecordFieldResolver.allowedFields` 正则校验
- **隐藏字段**：非 superuser 且 `allowHiddenFields=false` 时，`field.GetHidden()` 的字段直接拒绝
- **邮箱字段特殊限制**：非 superuser 过滤 auth 集合的 email 字段时，自动附加 `emailVisibility = TRUE` 条件（[AfterBuild 钩子](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/core/record_field_resolver_runner.go#L823-L831)）
- **不存在的字段**：非 `@request.*` 路径返回错误，`@request.*` 优雅降级为 NULL

### 5.4 表达式资源限制

| 限制 | 常量 | 值 | 位置 |
|------|------|----|------|
| 最大表达式数量 | `DefaultFilterExprLimit` | 200 | [provider.go](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/search/provider.go#L21) |
| 最大 filter 字符串长度 | `MaxFilterLength` | 3500 | [provider.go](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/search/provider.go#L30) |
| 最大 sort 表达式数 | `DefaultSortExprLimit` | 8 | [provider.go](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/search/provider.go#L24) |
| 最大 sort 字段长度 | `MaxSortFieldLength` | 255 | [provider.go](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/search/provider.go#L33) |
| 最大嵌套关系深度 | `maxNestedRels` | 6 | [record_field_resolver_runner.go](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/core/record_field_resolver_runner.go#L22) |
| strftime 最大参数 | — | 10 | [token_functions.go](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/search/token_functions.go#L89-L91) |
| 展开查询限制 | — | 1000 条 | [record_query_expand.go](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/core/record_query_expand.go#L108) |
| 每页最大数量 | `MaxPerPage` | 1000 | [provider.go](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/search/provider.go#L27) |
| 解析缓存上限 | — | 500 | [filter.go](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/search/filter.go#L102) |

### 5.5 ListRule 顶层绑定

如 3.6 节所述，关联集合的 ListRule 被绑定在最外层 `AND WHERE`，而非子查询内部。注释明确说明这是出于**安全考虑**：防止侧信道攻击通过 timing 差异推断受保护数据。

### 5.6 定时攻击防护（Timing Attack Mitigation）

在 [recordsList](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/apis/record_crud.go#L104-L122) 中，当满足以下全部条件时触发随机延迟（0-500ms）：

1. 非 superuser 请求
2. 集合有非空 ListRule
3. 请求携带了 filter 参数
4. 返回结果为空
5. 该集合已触发 3 次/3秒 的限流

代码注释也坦诚说明这不是完美防护，但配合网络延迟在实践中足以提高攻击门槛。真正敏感的字段（password、tokenKey）从根本上就不允许客户端过滤，且必要时使用**常数时间比较**。

### 5.7 列名引用规范

所有标识符（表名、列名、别名）统一使用 `[[...]]` 语法（PocketBase 的 dbx 方言），最终由 dbx 层转为数据库特定的引用方式（SQLite 用双引号），避免标识符注入。

---

## 六、完整调用链图示

```
用户输入: name ~ 'test' && @request.auth.id = author.id
│
├─ Provider.ParseAndExec()
│   ├─ Parse URL query: filter="name ~ 'test' && @request.auth.id = author.id"
│   └─ Exec()
│       ├─ FilterData.BuildExprWithLimit()
│       │   ├─ 替换 {:...} 占位符（此例无）
│       │   ├─ 查 parsedFilterData 缓存（miss）
│       │   ├─ fexpr.Parse() → AST:
│       │   │   [
│       │   │     {Item: Expr{Left:name, Op:~, Right:'test'}, Join: AND},
│       │   │     {Item: Expr{Left:@request.auth.id, Op:=, Right:author.id}}
│       │   │   ]
│       │   ├─ 存入缓存
│       │   └─ buildParsedFilterExpr()
│       │       ├─ Expr1: name ~ 'test'
│       │       │   ├─ resolveToken(name)
│       │       │   │   └─ RecordFieldResolver.Resolve("name")
│       │       │   │       └─ runner → "[[posts.name]]"
│       │       │   ├─ resolveToken('test')
│       │       │   │   └─ "{:tXxXxXxXx}" Params: {"tXxXxXxXx": "%test%"}
│       │       │   └─ buildResolversExpr → "posts.name LIKE {:tXxXxXxXx} ESCAPE '\\'"
│       │       │
│       │       └─ Expr2: @request.auth.id = author.id
│       │           ├─ resolveToken(@request.auth.id)
│       │           │   └─ runner → processRequestAuthField()
│       │           │       ├─ 注册 JOIN: users __auth_users ON __auth_users.id = {:userId}
│       │           │       └─ "[[__auth_users.id]]"
│       │           ├─ resolveToken(author.id)
│       │           │   └─ runner → processActiveProps()
│       │           │       ├─ author 是单值关系
│       │           │       ├─ id 优化 → 直接用 author 字段值
│       │           │       └─ "[[posts.author]]"
│       │           └─ buildResolversExpr → resolveEqualExpr(true, ...)
│       │               └─ 因 author 可能空 → "[[posts.author]] = [[__auth_users.id]]
│       │                   OR [[posts.author]] IS NULL"
│       │
│       ├─ modelsQuery.AndWhere(所有表达式)
│       ├─ fieldsResolver.UpdateQuery(query)
│       │   ├─ 注入 LEFT JOIN ...（注册的 auth users）
│       │   ├─ 若有 ListRule JOIN，注入其约束
│       │   └─ 若有关联 JOIN，启用 DISTINCT(true)
│       │
│       ├─ [并发] countExec → COUNT(DISTINCT _rowid_)
│       └─ [并发] modelsExec → LIMIT + OFFSET 取出数据
│
└─ 返回 Result{Items, Page, PerPage, TotalItems, TotalPages}
```

---

## 七、关键文件索引

| 关注点 | 文件路径 |
|--------|----------|
| 过滤 DSL 解析核心 | [tools/search/filter.go](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/search/filter.go) |
| 搜索 Provider 协调器 | [tools/search/provider.go](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/search/provider.go) |
| Token/函数解析 | [tools/search/token_functions.go](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/search/token_functions.go) |
| 标识符宏 | [tools/search/identifier_macros.go](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/search/identifier_macros.go) |
| 通用字段解析器 | [tools/search/simple_field_resolver.go](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/search/simple_field_resolver.go) |
| Multi-Match 子查询 | [tools/search/multi_match_subquery.go](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/search/multi_match_subquery.go) |
| Record 字段解析器主文件 | [core/record_field_resolver.go](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/core/record_field_resolver.go) |
| Record 字段解析 Runner | [core/record_field_resolver_runner.go](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/core/record_field_resolver_runner.go) |
| 表达式替换工具 | [core/record_field_resolver_replace_expr.go](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/core/record_field_resolver_replace_expr.go) |
| JSON 工具函数 | [tools/dbutils/json.go](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/dbutils/json.go) |
| 索引解析工具 | [tools/dbutils/index.go](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/dbutils/index.go) |
| Record 查询构建 | [core/record_query.go](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/core/record_query.go) |
| 关系展开（expand） | [core/record_query_expand.go](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/core/record_query_expand.go) |
| API 层列表入口 | [apis/record_crud.go](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/apis/record_crud.go) |
| 词法分析器基础 | [tools/tokenizer/tokenizer.go](file:///d:/fz/0601/solo-dogfeeding/code/152-pocketbase/tools/tokenizer/tokenizer.go) |
