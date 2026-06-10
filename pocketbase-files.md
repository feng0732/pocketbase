# PocketBase 文件上传与存储驱动流程分析

> 代码引用统一使用仓库相对路径，如 `core/field_file.go#L241-L307`

---

## 一、整体架构概览

```
HTTP Request (multipart/form-data)
         │
         ▼
┌───────────────────────────────────────────┐
│  apis/record_crud.go                      │
│  ├─ recordDataFromRequest()               │  ── 从请求提取文件并合并普通字段
│  └─ extractUploadedFiles()                │  ── 识别 fieldName / +field / field+ 三种 key
└─────────────────────┬─────────────────────┘
                      │
                      ▼
┌───────────────────────────────────────────┐
│  forms/record_upsert.go                   │
│  └─ Load(data)                            │  ── 将数据加载到 Record，触发 ReplaceModifiers
└─────────────────────┬─────────────────────┘
                      │
                      ▼
┌───────────────────────────────────────────┐
│  core/record_model.go                     │
│  ├─ ReplaceModifiers()                    │  ── 解析 +field / field+ / field- 修饰符
│  └─ SetIfFieldExists() → FindSetter()     │  ── 按字段类型路由到具体 setter
└─────────────────────┬─────────────────────┘
                      │
                      ▼
┌───────────────────────────────────────────┐
│  core/field_file.go                       │
│  ├─ ValidateValue()                       │  ── 6 层校验: 必填/数量/格式/大小/MIME 等
│  └─ Intercept()                           │  ── 生命周期拦截: 上传→DB写入→清理旧文件
└─────────────────────┬─────────────────────┘
                      │
                      ▼
┌───────────────────────────────────────────┐
│  tools/filesystem/filesystem.go           │
│  └─ System (统一接口封装)                  │  ── UploadFile / Delete / Serve / CreateThumb 等
└─────────────────────┬─────────────────────┘
                      │
                      ▼
┌───────────────────────────────────────────┐
│  tools/filesystem/blob/                   │
│  ├─ driver.go  (Driver 接口, 8 个方法)    │
│  └─ bucket.go  (线程安全封装 + 错误包装)   │
└───────────────┬───────────────────────────┘
                │
      ┌─────────┴─────────┐
      ▼                   ▼
┌───────────────┐  ┌───────────────┐
│internal/      │  │internal/      │
│fileblob/      │  │s3blob/        │  ── 本地文件系统 / S3 兼容存储
│(临时文件+rename│  │(自研 HTTP S3  │
│ 原子写入)     │  │ 客户端)       │
└───────────────┘  └───────────────┘
```

---

## 二、文件数据结构与命名

### 2.1 File 结构体

定义于 `tools/filesystem/file.go#L29-L34`

```go
type File struct {
    Reader       FileReader  // 统一的文件内容读取接口
    Name         string      // 规范化后的存储名 (含随机后缀)
    OriginalName string      // 用户上传的原始文件名
    Size         int64       // 文件字节数
}
```

**FileReader 多态实现**：
- `MultipartReader` — 来自 HTTP `multipart/form-data`（`tools/filesystem/file.go#L131-L138`）
- `PathReader` — 来自本地文件路径（`tools/filesystem/file.go#L145-L152`）
- `BytesReader` — 来自内存字节数组（`tools/filesystem/file.go#L159-L175`）
- `openFuncAsReader` — 来自函数闭包，用于"已有文件重新包装为可上传"场景（`tools/filesystem/file.go#L181-L187`）

### 2.2 文件名规范化算法

`tools/filesystem/file.go#L195-L236` 中 `normalizeName()` 的步骤：

1. **超长截断**：原始文件名 >300 字符时，只保留末尾 300 字符
2. **扩展名处理**：
   - 支持双后缀如 `.tar.gz`（通过 `extractExtension()` 实现，`tools/filesystem/file.go#L248-L262`）
   - 扩展名过滤非法字符，无效扩展名通过 MIME 嗅探重新检测
   - 扩展名最长 20 字符
3. **主体名规范化**：
   - 转 `Snakecase`，去除首尾点号
   - 长度 <3 追加 10 位随机字符；长度 >100 截断
4. **最终格式**：`{cleanName}_{10位随机字母数字}{.ext}`

示例：`My Annual Report.FINAL.PDF` → `my_annual_report_final_abc123def4.pdf`

---

## 三、请求数据进入记录：字段合并与修饰符解析

这是理解文件上传落地最关键的一步。整个流程分为两个阶段：**从 HTTP 请求提取** → **应用修饰符到 Record**。

### 3.1 阶段一：`recordDataFromRequest()` 提取与合并

位于 `apis/record_crud.go#L631-L688`

```
HTTP Request Body (multipart/form-data 或 JSON)
         │
         ├─ 1. record.ReplaceModifiers(info.Body)
         │      └── 先对普通字段（非文件）应用修饰符，如 number+、select- 等
         │
         ├─ 2. extractUploadedFiles()
         │      └── 遍历 Collection 的所有 FileField
         │          对每个字段尝试读取 3 种 key:
         │            - "avatar"     (直接替换)
         │            - "+avatar"    (前置追加)
         │            - "avatar+"    (后置追加)
         │          将 multipart.FileHeader 转为 *filesystem.File
         │
         └─ 3. 合并文件与普通字段值
                └── 关键逻辑 (apis/record_crud.go#L650-L669):
                    如果请求同时包含:
                      info.Body["avatar"] = ["old1.jpg"]   (普通表单字段)
                      multipart["avatar+"] = [newFile]      (上传文件)
                    且 key 不带 +/- 修饰符 → 先把已有字符串值放入列表，再追加上传文件
                    最终 result["avatar"] = ["old1.jpg", newFile]
```

**重要细节**：只有当 key 是裸字段名（不是 `+field`、`field+`、`field-`）且 `info.Body` 中已有值时，才会执行"已有值 + 新上传文件"的合并。修饰符 key 不会触发合并，直接以修饰符语义为准。

### 3.2 阶段二：`ReplaceModifiers()` 修饰符解析

位于 `core/record_model.go#L1360-L1390`

修饰符系统是 PocketBase 的通用机制，不仅文件字段支持，Number、Select、Relation 等字段也支持。

**执行流程**：

```go
func (m *Record) ReplaceModifiers(data map[string]any) map[string]any {
    dataCopy := maps.Clone(data)
    recordCopy := m.Fresh()   // 创建 Record 临时副本，加载当前 DB 中已有值

    // 按 key 长度排序（短的先处理），确保可预测结果
    sortedDataKeys = sortKeysByLength(data)

    for k := range sortedDataKeys {
        // 对临时副本 SetIfFieldExists，触发 FindSetter → 具体 setter
        field := recordCopy.SetIfFieldExists(k, data[k])

        if field != nil {
            delete(dataCopy, k)                              // 删掉修饰符 key
            dataCopy[field.GetName()] = recordCopy.Get(...)   // 存入解析后的最终值
        }
    }
    return dataCopy
}
```

**关键设计**：
- 不在原 Record 上操作，而是 `Fresh()` 创建一个带有当前 DB 值的临时副本（`core/record_model.go#L634-L650`），确保修饰符基于"旧值"计算，且不产生副作用
- **只按 key 字符串长度升序排序**：`sort.SliceStable(sortedDataKeys, func(i, j int) bool { return len(key[i]) < len(key[j]) })`（`core/record_model.go#L1374-L1376`）
- **相同长度 key 不保证先后顺序**：Go 的 map 遍历顺序是随机的，`sort.SliceStable` 对长度相等的元素保持其原始遍历顺序不变，因此长度相同的修饰符处理顺序不确定
- 只有长度不同的 key 之间才有确定顺序（短 key 先处理）

### 3.3 `SetIfFieldExists()` → `FindSetter()` 路由

位于 `core/record_model.go#L867-L887`

```go
func (m *Record) SetIfFieldExists(key string, value any) Field {
    for _, field := range m.Collection().Fields {
        if ff, ok := field.(SetterFinder); ok {
            setter := ff.FindSetter(key)   // 每个字段类型自己决定支持哪些修饰符
            if setter != nil {
                setter(m, value)
                return field
            }
        }
        // fallback: key == field.Name 时，走 PrepareValue + SetRaw
    }
    return nil
}
```

`SetterFinder` 接口定义于 `core/field.go#L122-L136`，每个字段实现 `FindSetter(key)` 返回匹配的 `SetterFunc`。

### 3.4 FileField 支持的 4 种修饰符

`core/field_file.go#L662-L675` 中 `FindSetter()`，以字段名 `documents` 为例分析 key 长度：

| key 模式 | key 长度 | 触发函数 | 行为 |
|----------|----------|----------|------|
| `"documents"` | 9（最短，一定最先处理） | `setValue()` | 完全替换为新值（字符串+`*File` 混合） |
| `"+documents"` | 10（与下面两行相同） | `prependValue()` | 在已有文件列表**头部**插入新文件 |
| `"documents+"` | 10（与上/下行相同） | `appendValue()` | 在已有文件列表**尾部**追加新文件 |
| `"documents-"` | 10（与上面两行相同） | `subtractValue()` | 从已有列表中**删除**指定文件名 |

> **关键结论**：`+documents`、`documents+`、`documents-` 三者 key 长度相同（均为 `len(fieldName)+1`），它们之间的处理顺序不保证，取决于 Go map 的随机遍历顺序。只有裸字段名 `documents`（长度 `len(fieldName)`）最短，确定最先执行。
>
> 该模式也适用于 NumberField（`number+`/`number-` 长度相同）、SelectField、RelationField —— 它们的 FindSetter 都遵循同样的前后缀命名规则。

具体实现（`core/field_file.go#L677-L712`）：

```go
// 前置追加: 新值 + 旧值
func (f *FileField) prependValue(record *Record, toPrepend any) {
    files := f.toSliceValue(record.GetRaw(f.Name))
    prepends := f.toSliceValue(toPrepend)
    if len(prepends) > 0 {
        files = append(prepends, files...)
    }
    f.setValue(record, files)
}

// 后置追加: 旧值 + 新值
func (f *FileField) appendValue(record *Record, toAppend any) {
    files := f.toSliceValue(record.GetRaw(f.Name))
    appends := f.toSliceValue(toAppend)
    if len(appends) > 0 {
        files = append(files, appends...)
    }
    f.setValue(record, files)
}

// 删除: 旧值 - 待删除值
func (f *FileField) subtractValue(record *Record, toRemove any) {
    files := f.excludeFiles(
        f.toSliceValue(record.GetRaw(f.Name)),
        f.toSliceValue(toRemove),
    )
    f.setValue(record, files)
}
```

### 3.5 FileField 值的两种形态：纯字符串与 `*filesystem.File`

在进入修饰符处理阶段，FileField 的值列表中可能混合两种类型的元素（`core/field_file.go#L775-L797` 的 `extractPlainStrings` 和 `extractUploadableFiles` 区分）：

| 类型 | 来源 | 含义 |
|------|------|------|
| `string` | 旧值（DB 中已有）或客户端重新提交的已有文件名 | 引用已存在的文件，只需保存文件名 |
| `*filesystem.File` | multipart 上传，由 `extractUploadedFiles()` 转换（`apis/record_crud.go#L690-L727`）| 新上传的文件，包含 Reader、原始文件名、已规范化的存储名 |

**关键点**：新上传的 `*filesystem.File` 在 `extractUploadedFiles` 阶段就已经完成了文件名规范化（`tools/filesystem/file.go#L195-L236` 的 `normalizeName()`），文件名格式为 `{cleanName}_{10位随机字符}{.ext}`，不是用户上传时的原始文件名。

### 3.6 修饰符与校验的协作：为什么不能凭空新增纯字符串

`ReplaceModifiers` 阶段只做列表组装，不做安全校验。真正的安全检查在 `FileField.ValidateValue()`（`core/field_file.go#L250-L269`）：

```
核心逻辑:  addedStrings = 新值中的纯字符串 - 旧值中的纯字符串

如果 addedStrings 非空 → 说明有人试图凭空"伪造"一个文件引用 → 报错
```

这意味着：
- ✅ `documents- = "old.jpg"` — 合法。删除的是已存在的文件名，不涉及新增
- ✅ `documents+ = [*filesystem.File]` — 合法。新上传的是 `*File` 类型，会走上传流程
- ❌ `documents+ = ["made_up.txt"]` — 非法。新增的是纯字符串，不在旧值中，会被 ValidateValue 拒绝
- ✅ 重新排列已有文件的顺序（纯字符串，但都在旧值中）— 合法

---

### 3.7 修饰符组合的顺序不确定性

`+documents`、`documents+`、`documents-` 三者 key 长度相同（均为 `len(fieldName)+1`），处理顺序取决于 Go map 的随机遍历顺序，**不保证固定**。

#### 场景一：`documents+` 与 `documents-` 组合（真实上传场景）

这是最常见的场景：同时上传新文件并删除旧文件。

```
假设 DB 中 documents = ["old_a.jpg", "old_b.jpg"]

请求 multipart:
  documents+  = [ *filesystem.File{Name: "new_photo_abc123def4.jpg"} ]
                   新上传文件，文件名已规范化，带随机后缀
  documents-  = "old_a.jpg"   删除已有旧文件，纯字符串

两种可能的执行顺序，结果一致:

  先 + 后 -:
    1. documents+ → ["old_a.jpg", "old_b.jpg", new_photo_abc123def4.jpg]
    2. documents- → ["old_b.jpg", new_photo_abc123def4.jpg]

  先 - 后 +:
    1. documents- → ["old_b.jpg"]
    2. documents+ → ["old_b.jpg", new_photo_abc123def4.jpg]

最终结果相同: ["old_b.jpg", new_photo_abc123def4.jpg]
```

> **说明**：这个场景下顺序不影响结果，因为追加的新文件名（带随机后缀）和删除的旧文件名不会重叠。虽然结果一致，但处理顺序仍然是不确定的，不能依赖。

#### 场景二：重排 + 删除同一个已有文件（边界场景，结果不确定）

当两个修饰符操作**同一个已有文件**时，执行顺序会影响最终结果。这种情况通常发生在使用纯字符串文件名对已有文件进行重排的场景：

```
假设 DB 中 documents = ["old_a.jpg", "old_b.jpg"]

请求 body (JSON 或 form):
  +documents   = ["old_b.jpg"]    把已有文件移到最前面（头插）
  documents-   = "old_b.jpg"      删除已有文件

两种可能的执行顺序，结果不同:

  先 +documents 后 documents-:
    1. +documents  → ["old_b.jpg", "old_a.jpg", "old_b.jpg"]   （有重复）
    2. documents-  → ["old_a.jpg"]                              （删除所有 old_b.jpg）
    结果: ["old_a.jpg"]

  先 documents- 后 +documents:
    1. documents-  → ["old_a.jpg"]                              （删除 old_b.jpg）
    2. +documents  → ["old_b.jpg", "old_a.jpg"]                 （头插 old_b.jpg）
    结果: ["old_b.jpg", "old_a.jpg"]
```

> **说明**：`+documents = ["old_b.jpg"]` 中的 `old_b.jpg` 是纯字符串，但它在旧值中已存在，因此**不会**被 `ValidateValue` 拒绝（只拦"新增的"纯字符串，不拦已有的）。

> **实践建议**：
> - 真实场景中 `documents+`（新上传 `*File`）+ `documents-`（删旧文件字符串）的组合最常见，由于新旧文件名不重叠（新文件有随机后缀），结果通常一致，但顺序仍然是不确定的，不能依赖。
> - 对同一字段混合使用多个相同长度的修饰符时需谨慎。如果需要确定的行为，请在客户端组装完整列表后，直接使用裸字段名 `documents` 一次性提交。

---

## 四、文件校验流程

校验入口 `FileField.ValidateValue()` 位于 `core/field_file.go#L241-L307`，在 Record 保存时由 `onRecordValidate()`（`core/record_model.go#L1413-L1427`）统一触发。

### 4.1 校验层级详解

| # | 校验项 | 代码位置 | 说明 |
|---|--------|----------|------|
| 1 | 必填校验 | `core/field_file.go#L243-L248` | `Required=true` 且文件列表为空时返回 `validation.ErrRequired` |
| 2 | 禁止纯字符串新文件 | `core/field_file.go#L250-L269` | **安全关键点**：对比新旧值差异，新增的如果是纯字符串而非 `*filesystem.File`，报错。防止用户绕过上传直接伪造文件名 |
| 3 | 文件数量上限 | `core/field_file.go#L271-L275` | 超过 `MaxSelect`（单文件默认 1）报错 |
| 4 | 文件名格式 | `core/field_file.go#L282-L289` | 长度 1-150，匹配 `looseFilenameRegex = ^[^\./\\][^/\\]+$`（不能以点、斜杠开头，不能含路径分隔符）|
| 5 | 文件大小 | `core/validators/file.go#L18-L41` | `UploadedFileSize()`，默认 5MB（`DefaultFileFieldMaxSize`），超限报 `validation_file_size_limit` |
| 6 | MIME 类型 | `core/validators/file.go#L50-L96` | `UploadedFileMimeType()`，通过 `mimetype.DetectReader()` 读取文件内容头字节**真实嗅探**，不是靠扩展名判断 |

### 4.2 关于"禁止纯字符串新文件"的安全机制

```go
// core/field_file.go#L250-L269
oldExistingStrings := f.toSliceValue(f.getLatestOldValue(app, record))
existingStrings := list.ToInterfaceSlice(f.extractPlainStrings(files))
addedStrings := f.excludeFiles(existingStrings, oldExistingStrings)

if len(addedStrings) > 0 {
    // 新增的是纯字符串，不是 *filesystem.File → 拒绝
    return validation.NewError("validation_invalid_file", ...)
}
```

这条规则保证：新加入的文件必须经过上传流程（即类型为 `*filesystem.File`），用户不能直接 POST `{"documents": ["malicious.txt"]}` 来"伪造"一个服务器上并不存在的文件引用。

---

## 五、文件保存核心：拦截器生命周期

`FileField.Intercept()`（`core/field_file.go#L336-L404`）实现了 `RecordInterceptor` 接口，将文件操作深度嵌入 Record 的保存/删除事务流程。

### 5.1 完整执行时序

```
InterceptorActionCreateExecute / UpdateExecute
         │
         ├─ ① 取旧值: getLatestOldValue(app, record)
         │      └─ 非新建时从 DB 重新读取最新记录，避免并发更新丢失
         │
         ├─ ② processFilesToUpload(ctx, app, record)
         │      └─ 先于 DB 写入上传所有新文件
         │         失败 → 删除已上传的，立即 return error
         │
         ├─ ③ actionFunc()  ← 真正执行 DB 写入（INSERT / UPDATE）
         │      │
         │      ├─ 失败 → afterRecordExecuteFailure(ctx, app, record)
         │      │          └─ 删除所有新上传的文件（回滚存储）
         │      │
         │      └─ 成功
         │            │
         │            ├─ rememberFilesToDelete(app, record, oldValue)
         │            │    └─ 对比新旧值，将待删除的旧文件名暂存到
         │            │       Record 的 @pbInternal_deletedFilesPrefix_xxx 内部字段
         │            │
         │            └─ afterRecordExecuteSuccess(ctx, app, record)
         │                 └─ 将值中的 *filesystem.File 对象替换为纯字符串文件名
         │                    （因为文件已落盘，DB 只需存文件名）
         │                    已上传文件列表暂存到 @pbInternal_uploadedFilesPrefix_xxx
         │
InterceptorActionAfterCreate / AfterUpdate
         │
         └─ processFilesToDelete(ctx, app, record)
               └─ 读取 @pbInternal_deletedFilesPrefix_xxx
                  对每个文件名:
                    1. 删除主文件: fsys.Delete(record.BaseFilesPath() + "/" + name)
                    2. 删除缩略图: fsys.DeletePrefix("thumbs_" + name + "/")
                  失败仅记日志，不影响 Record 状态（最终一致）

InterceptorActionAfterCreateError / AfterUpdateError
         │
         └─ deleteNewlyUploadedFiles(ctx, app, record)
               └─ 读取 @pbInternal_uploadedFilesPrefix_xxx 删除新文件
                  新建失败额外尝试 deleteEmptyRecordDir()
```

### 5.2 文件上传执行：`processFilesToUpload()`

`core/field_file.go#L512-L552`

```go
fsys, _ := app.NewFilesystem()
defer fsys.Close()

for _, upload := range uploads {
    path := record.BaseFilesPath() + "/" + upload.Name
    // BaseFilesPath = collection.Id + "/" + record.Id
    // 见 core/record_model.go#L603-L609 和 core/collection_model.go#L470-L472

    if err := fsys.UploadFile(upload, path); err == nil {
        succeeded = append(succeeded, upload.Name)
    } else {
        // 遇第一个错误即停止，不允许部分成功
        deleteFilesByNamesList(ctx, app, record, succeeded)  // 回滚已上传的
        return fmt.Errorf("failed to upload all files: %w", ...)
    }
}
```

**存储路径规则**：`{collectionId}/{recordId}/{normalizedFilename}`

例如：`w7nqg9f1k0p2x34/rec_abc123def4/my_photo_a1b2c3d4e5.jpg`

### 5.3 事务一致性设计原则

| 阶段 | 失败后果 | 处理方式 |
|------|----------|----------|
| 文件上传中 | 已上传部分文件 | 立即停止，回滚已上传的 |
| DB 写入中（事务内） | 文件已上传但 DB 回滚 | `afterRecordExecuteFailure` 删除所有新上传 |
| DB 提交后事务外失败 | DB 已提交但后续处理异常 | `After*Error` 钩子尽力清理新文件和空目录 |
| 成功→旧文件删除失败 | 主流程已成功，旧文件残留 | 记 WARN 日志，靠人工或后续操作补偿（最终一致）|

核心原则：**先上传文件，再写 DB**。永远不会出现"DB 有记录但文件不存在"的情况，最坏只是有孤儿文件。

---

## 六、旧文件删除的触发条件汇总

旧文件删除有三条独立触发路径：

### 路径一：字段级更新时被替换/移除

触发条件：Record Update 操作中，FileField 的新值比旧值少了某些文件名。

涉及函数：
- `rememberFilesToDelete()` `core/field_file.go#L498-L510` — 计算差异并入队
- `processFilesToDelete()` `core/field_file.go#L476-L496` — AfterCreate/AfterUpdate 中执行

触发场景示例：
```
旧值: ["a.jpg", "b.jpg"]
新值: ["a.jpg"]            → b.jpg 被删
新值: ["c.jpg"]            → a.jpg, b.jpg 都被删
新值: ["a.jpg", "b.jpg"]   → 都不删
```

### 路径二：通过 `fieldName-` 修饰符显式删除

触发条件：请求中带 `documents-=a.jpg` 参数。

在 `ReplaceModifiers` 阶段就把文件名从值列表中移除，后续流程同路径一。

### 路径三：Record 或 Collection 被整体删除

触发条件：任何实现了 `FilesManager` 接口（`core/base.go#L50-L53`）的 Model 被删除。

实现于全局钩子 `registerBaseHooks()`（`core/base.go#L1289-L1348`）：

```go
app.OnModelAfterDeleteSuccess().Bind(&hook.Handler[*ModelEvent]{
    Id: "__pbFilesManagerDelete__",
    Func: func(e *ModelEvent) error {
        if m, ok := e.Model.(FilesManager); ok && m.BaseFilesPath() != "" && supportFiles(e.Model) {
            prefix := strings.TrimRight(m.BaseFilesPath(), "/") + "/"

            // 信号量控制并发，默认最多 2000 个同时删除任务
            deleteSem.Acquire(context.Background(), 1)

            // FireAndForget: 后台异步删除，不阻塞 DB 事务提交
            routine.FireAndForget(func() {
                defer deleteSem.Release(1)
                deletePrefix(prefix)  // fsys.DeletePrefix(collectionId/recordId/)
            })
        }
        return e.Next()
    },
    Priority: -99,
})
```

特点：
- **异步执行**：不阻塞删除事务，返回成功后后台慢慢删
- **信号量限流**：默认最多 2000 并发（`PB_FILES_DELETE_MAX_WORKERS` 可调）
- **整个目录级删除**：用 `DeletePrefix()` 批量删，比逐个删高效

---

## 七、文件下载与缩略图

### 7.1 下载路由

`apis/file.go#L25-L46`

```
GET /api/files/{collection}/{recordId}/{filename}
    ?thumb=WxH|WxHt|WxHb|WxHf|0xH|Wx0
    &token=xxx           (受保护文件必填)
    &download=true|false (强制 attachment)
```

权限检查流程（`apis/file.go#L108-L135`）：
1. `FileField.Protected == true` → 需 token
2. token 通过 `POST /api/files/token` 获取（需认证），token 内含用户身份
3. 用 token 对应身份 + 记录的 `viewRule` 做 `CanAccessRecord` 校验
4. 不通过直接 404（不告知"存在但无权"，防止枚举）

### 7.2 缩略图懒生成

`apis/file.go#L164-L200` + `createThumb()` `apis/file.go#L219-L243`

```
请求 /api/files/.../photo.jpg?thumb=300x300
  │
  ├─ 检查 fsys.Exists(thumbs_photo.jpg/300x300_photo.jpg)
  │     └─ 已存在 → 直接 Serve
  │
  └─ 不存在 → createThumb()
        │
        ├─ singleflight.Group(thumbPath): 同一张图的并发请求只生成一次
        ├─ semaphore.Weighted: 全局最多 CPU+2 个并发生成任务
        │     (PB_THUMBS_MAX_WORKERS 可调)
        ├─ context 超时: 默认 60s (PB_THUMBS_MAX_WAIT 可调)
        │
        └─ fsys.CreateThumb(originalPath, thumbPath, size)
              见 tools/filesystem/filesystem.go#L499-L587
              使用 disintegration/imaging 库
```

缩略图存储路径：`{recordBasePath}/thumbs_{filename}/{thumbSize}_{filename}`

---

## 八、存储驱动抽象与切换

### 8.1 三层封装架构

| 层级 | 文件 | 职责 |
|------|------|------|
| System | `tools/filesystem/filesystem.go` | 业务方法（UploadFile、Serve、CreateThumb…），屏蔽所有存储差异 |
| Bucket | `tools/filesystem/blob/bucket.go` | 线程安全包装 + UTF-8 校验 + Metadata 统一小写 + 错误归一化 |
| Driver | `tools/filesystem/blob/driver.go` | 8 个方法的最小接口，新驱动只需实现这 8 个 |

### 8.2 Driver 接口

`tools/filesystem/blob/driver.go#L39-L97`

```go
type Driver interface {
    NormalizeError(err error) error
    Attributes(ctx context.Context, key string) (*Attributes, error)
    ListPaged(ctx context.Context, opts *ListOptions) (*ListPage, error)
    NewRangeReader(ctx context.Context, key string, offset, length int64) (DriverReader, error)
    NewTypedWriter(ctx context.Context, key, contentType string, opts *WriterOptions) (DriverWriter, error)
    Copy(ctx context.Context, dstKey, srcKey string) error
    Delete(ctx context.Context, key string) error
    Close() error
}
```

### 8.3 本地驱动 (fileblob) 原子写入

`tools/filesystem/internal/fileblob/fileblob.go`

写入流程：
1. `createTemp()` 创建临时文件：`{target}.{nanoseconds_in_hex}.tmp`
   - 可通过 `NoTempDir=true` 决定放 `os.TempDir` 还是目标目录旁
2. 写入临时文件同时计算 MD5 hash
3. `Close()` 时：
   - 写 `.attrs` sidecar 文件（存 ContentType、Metadata、MD5 等）
   - `os.Rename()` 原子替换为目标路径
   - 清理临时文件

Key 转义（跨平台安全）：控制字符 0-31、`../`、`//`、Windows `<>:"|?*` 等 → `__0x<hex>__`

### 8.4 S3 驱动 (s3blob)

`tools/filesystem/internal/s3blob/s3blob.go`

- 自研 `s3.S3` HTTP 客户端（不依赖 AWS SDK，轻量）
- 支持 `UsePathStyle=true` 兼容 MinIO / 阿里 OSS / 腾讯 COS 等非 AWS S3 服务
- Metadata key/value 用 URL 编码 + Hex 双层转义
- ETag → MD5 转换用于完整性校验

### 8.5 驱动切换决策：`NewFilesystem()`

`core/base.go#L715-L729`

```go
func (app *BaseApp) NewFilesystem() (*filesystem.System, error) {
    if app.settings != nil && app.settings.S3.Enabled {
        return filesystem.NewS3(
            app.settings.S3.Bucket,
            app.settings.S3.Region,
            app.settings.S3.Endpoint,
            app.settings.S3.AccessKey,
            app.settings.S3.Secret,
            app.settings.S3.ForcePathStyle,
        )
    }
    return filesystem.NewLocal(filepath.Join(app.DataDir(), LocalStorageDirName))
}
```

**决策逻辑**：
1. 读取 `app.settings.S3.Enabled`（Settings 存在数据库中）
2. true → S3 驱动；false → 本地 `pb_data/storage/`
3. **每次调用都重新判断**，Settings 变更后立即生效，无需重启
4. 备份有独立配置：`NewBackupsFilesystem()` 读 `settings.Backups.S3`

---

## 九、`app.NewFilesystem()` 全量调用点

以下是生产代码中所有调用位置（排除 `_test.go` 文件）：

### 9.1 文件上传路径

| 调用位置 | 场景 |
|----------|------|
| `core/field_file.go#L522` | `processFilesToUpload()` — 新建/更新 Record 时上传新文件 |

### 9.2 文件下载与缩略图路径

| 调用位置 | 场景 |
|----------|------|
| `apis/file.go#L148` | `download()` handler — 文件下载 + 缩略图检查与懒生成 |

### 9.3 文件清理路径

| 调用位置 | 场景 |
|----------|------|
| `core/field_file.go#L455` | `deleteEmptyRecordDir()` — 新建 Record 失败后清理空目录 |
| `core/field_file.go#L586` | `deleteFilesByNamesList()` — 字段级旧文件删除、上传失败回滚 |
| `core/base.go#L1291` | `registerBaseHooks()` 的 OnModelAfterDeleteSuccess — Record/Collection 整体删除时异步批量删目录 |

调用关系可视化：

```
app.NewFilesystem()
  │
  ├─ 上传场景
  │   └─ core/field_file.go Intercept(CreateExecute/UpdateExecute)
  │        └─ processFilesToUpload()
  │
  ├─ 下载场景
  │   └─ apis/file.go download()
  │        ├─ Attributes() / GetReader() ── 读取原文件
  │        ├─ Exists() + CreateThumb()    ── 缩略图
  │        └─ Serve()                     ── HTTP 响应
  │
  └─ 清理场景
       ├─ core/field_file.go
       │    ├─ afterRecordExecuteFailure() → deleteFilesByNamesList()
       │    ├─ processFilesToDelete()      → deleteFilesByNamesList()
       │    ├─ AfterCreate/UpdateError     → deleteNewlyUploadedFiles() → deleteFilesByNamesList()
       │    └─ AfterCreateError (新建)     → deleteEmptyRecordDir()
       │
       └─ core/base.go registerBaseHooks()
            └─ OnModelAfterDeleteSuccess → DeletePrefix(collectionId/recordId/)  异步
```

---

## 十、关键设计总结

1. **文件名随机化**：每个上传文件自动追加 10 位随机后缀，既防冲突又实现"不可枚举"的轻量安全。

2. **修饰符通用机制**：`SetterFinder` / `FindSetter` 让每个字段类型自定义 key 模式（`+field`、`field+`、`field-`、`:autogenerate` 等），`ReplaceModifiers` **仅按 key 长度升序排序**——长度不同时有确定顺序（短 key 先处理），**相同长度修饰符的先后顺序不保证**（依赖 Go map 的随机遍历顺序）。

3. **先文件后 DB**：上传成功才写 DB，DB 失败回滚文件，从根源避免"DB 有引用但文件不存在"的脏状态。

4. **三级删除保障**：字段级差异检测 → 修饰符显式删除 → 模型级目录清理，覆盖所有文件生命周期。

5. **异步+限流的批量删除**：Record/Collection 删除用信号量+后台协程执行，避免大目录删除阻塞事务。

6. **Driver 接口极小化**：8 个方法即可新增存储后端，上层 System/Bucket 逻辑 100% 复用。

7. **本地写入原子性**：临时文件 + `os.Rename()` 确保不会出现半写入的损坏文件。

8. **动态驱动切换**：每次 `NewFilesystem()` 都重新读 Settings，S3 配置变更即时生效，无需重启服务。
