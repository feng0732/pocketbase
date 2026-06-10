# PocketBase 文件上传与存储驱动流程分析

## 一、整体架构概览

```
HTTP Request
     │
     ▼
┌─────────────────────┐
│  record_crud.go     │  ── 提取 multipart 文件，转换为 *filesystem.File
│  extractUploadedFiles
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  record_upsert.go   │  ── Form.Load() 加载文件到 Record
│  Load(data)         │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  field_file.go      │  ── FileField 拦截器: 校验 → 上传 → DB 写入 → 清理旧文件
│  Intercept()        │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  filesystem.go      │  ── System 统一接口封装 (上传/删除/列表/缩略图等)
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  blob/driver.go     │  ── Driver 接口抽象 (Attributes/List/Read/Write/Delete/Copy)
│  blob/bucket.go     │
└─────────┬───────────┘
          │
    ┌─────┴─────┐
    ▼           ▼
┌─────────┐  ┌─────────┐
│fileblob │  │ s3blob  │  ── 具体驱动实现: 本地文件系统 / S3 兼容存储
│(本地)    │  │(S3)     │
└─────────┘  └─────────┘
```

---

## 二、文件数据结构

### 2.1 File 结构体（上传文件载体）

定义于 [file.go](file:///d:/fz/0601/solo-dogfeeding/code/163-pocketbase/tools/filesystem/file.go#L29-L34)

```go
type File struct {
    Reader       FileReader  // 文件内容读取器接口
    Name         string      // 规范化后的存储名 (含随机后缀)
    OriginalName string      // 原始文件名
    Size         int64       // 文件大小
}
```

**FileReader 接口** 定义了统一的 `Open()` 方法，支持多种来源：
- `MultipartReader` - 来自 HTTP multipart/form-data
- `PathReader` - 来自本地文件路径
- `BytesReader` - 来自内存字节数组
- `openFuncAsReader` - 来自自定义函数闭包

### 2.2 文件名规范化

`normalizeName()` [file.go#L195-L236](file:///d:/fz/0601/solo-dogfeeding/code/163-pocketbase/tools/filesystem/file.go#L195-L236)

规则：
1. 截取超长文件名（>300 字符取后 300）
2. 提取扩展名（支持 `.tar.gz` 双后缀），无效扩展名通过 MIME 检测
3. 主体名转为 Snakecase，过短追加随机字符，过长截断
4. 最终格式：`{cleanName}_{10位随机字符}{.ext}`

示例：`My Report.PDF` → `my_report_abc123def4.pdf`

---

## 三、文件校验流程

### 3.1 校验入口

`FileField.ValidateValue()` [field_file.go#L241-L307](file:///d:/fz/0601/solo-dogfeeding/code/163-pocketbase/core/field_file.go#L241-L307)

在 Record 保存时由字段校验系统自动调用。

### 3.2 校验层级

| 校验项 | 实现位置 | 说明 |
|--------|----------|------|
| 必填校验 | [field_file.go#L243-L248](file:///d:/fz/0601/solo-dogfeeding/code/163-pocketbase/core/field_file.go#L243-L248) | Required=true 且文件列表为空时报错 |
| 禁止纯字符串新文件 | [field_file.go#L250-L269](file:///d:/fz/0601/solo-dogfeeding/code/163-pocketbase/core/field_file.go#L250-L269) | 新上传必须是 `*filesystem.File`，不能只传文件名字符串 |
| 文件数量上限 | [field_file.go#L271-L275](file:///d:/fz/0601/solo-dogfeeding/code/163-pocketbase/core/field_file.go#L271-L275) | MaxSelect 控制，单文件默认 1 |
| 文件名长度/格式 | [field_file.go#L282-L289](file:///d:/fz/0601/solo-dogfeeding/code/163-pocketbase/core/field_file.go#L282-L289) | 1-150 字符，匹配 `looseFilenameRegex` |
| 文件大小 | [validators/file.go#L18-L41](file:///d:/fz/0601/solo-dogfeeding/code/163-pocketbase/core/validators/file.go#L18-L41) | `UploadedFileSize()`，默认 5MB |
| MIME 类型 | [validators/file.go#L50-L96](file:///d:/fz/0601/solo-dogfeeding/code/163-pocketbase/core/validators/file.go#L50-L96) | `UploadedFileMimeType()`，通过 `mimetype.DetectReader` 检测真实内容类型 |

### 3.3 MIME 类型检测特点

不是靠扩展名判断，而是通过读取文件内容头部字节进行嗅探（`gabriel-vasile/mimetype` 库），更安全可靠，防止伪装扩展名。

---

## 四、文件上传 API 入口

### 4.1 Record 创建/更新时的文件上传

文件并非独立 API，而是伴随 Record 的创建/更新一起提交（multipart/form-data）。

#### 路由注册
[record_crud.go#L27-L34](file:///d:/fz/0601/solo-dogfeeding/code/163-pocketbase/apis/record_crud.go#L27-L34)
```
POST   /api/collections/{collection}/records    ── 创建
PATCH  /api/collections/{collection}/records/{id} ── 更新
```

#### 文件提取流程

`extractUploadedFiles()` [record_crud.go#L690-L727](file:///d:/fz/0601/solo-dogfeeding/code/163-pocketbase/apis/record_crud.go#L690-L727)

1. 检查 Content-Type 是否为 `multipart/form-data`
2. 遍历 Collection 的所有 FileField
3. 尝试从请求中读取 3 种 key 格式：
   - `fieldName` - 直接设置（替换全部）
   - `+fieldName` - 前置追加
   - `fieldName+` - 后置追加
4. 将 `multipart.FileHeader` 转为 `*filesystem.File`（自动规范化文件名）

`recordDataFromRequest()` [record_crud.go#L631-L688](file:///d:/fz/0601/solo-dogfeeding/code/163-pocketbase/apis/record_crud.go#L631-L688) 合并文件与普通字段数据。

### 4.2 文件下载 API

[apis/file.go#L25-L46](file:///d:/fz/0601/solo-dogfeeding/code/163-pocketbase/apis/file.go#L25-L46)

```
GET /api/files/{collection}/{recordId}/{filename}?thumb=WxH&token=xxx
```

**受保护文件**（FileField.Protected=true）需要 `?token=` 参数，token 通过：
```
POST /api/files/token  ── 需认证
```
生成短期文件访问令牌。

**缩略图生成**（懒加载 + singleflight 去重 + 信号量限流）：
- 仅对图片类型生成（`image/png`, `image/jpg`, `image/jpeg`, `image/gif`, `image/webp`）
- 尺寸格式：`WxH`, `WxHt`, `WxHb`, `WxHf`, `0xH`, `Wx0`
- 存储路径：`{recordBasePath}/thumbs_{filename}/{thumbSize}_{filename}`

---

## 五、文件保存流程（核心）

核心机制是 `FileField.Intercept()` [field_file.go#L336-L404](file:///d:/fz/0601/solo-dogfeeding/code/163-pocketbase/core/field_file.go#L336-L404)，它实现了 `RecordInterceptor` 接口，在 Record 生命周期的不同节点插入文件操作。

### 5.1 拦截器生命周期

```
InterceptorActionCreateExecute / UpdateExecute
         │
         ├── 1. processFilesToUpload()  ── 先把文件传到存储
         │                                   (失败则全部回滚)
         │
         ├── 2. actionFunc()           ── 执行 DB 写入 (Record 保存)
         │
         ├── 失败 → afterRecordExecuteFailure()  ── 删除已上传的文件
         │
         └── 成功 → rememberFilesToDelete()  ── 标记需要删除的旧文件
                   afterRecordExecuteSuccess() ── 将 File 对象替换为文件名
                              │
                              ▼
              InterceptorActionAfterCreate / AfterUpdate
                              │
                              └── processFilesToDelete() ── 异步删除旧文件
```

### 5.2 文件上传执行

`processFilesToUpload()` [field_file.go#L512-L552](file:///d:/fz/0601/solo-dogfeeding/code/163-pocketbase/core/field_file.go#L512-L552)

```go
for _, upload := range uploads {
    path := record.BaseFilesPath() + "/" + upload.Name
    if err := fsys.UploadFile(upload, path); err == nil {
        succeeded = append(succeeded, upload.Name)
    } else {
        // 遇到第一个错误即停止，不允许部分上传
        // 然后删除已成功上传的文件做回滚
        deleteFilesByNamesList(succeeded)
        return error
    }
}
```

**存储路径规则**：`{collectionId}/{recordId}/{normalizedFilename}`

### 5.3 事务一致性保障

| 阶段 | 失败处理 |
|------|----------|
| 文件上传中 | 立即停止，删除已上传成功的文件 |
| DB 写入中（事务内）| `afterRecordExecuteFailure` 删除所有新上传文件 |
| DB 提交后失败 | `InterceptorActionAfterCreate/UpdateError` 清理新文件 + 空目录 |
| 成功后旧文件删除 | 失败仅记录日志，不影响 Record 状态（最终一致） |

关键设计：**先上传文件，再写 DB**。这样 DB 回滚时只需删文件，不会出现 DB 有记录但文件不存在的情况。

### 5.4 旧文件清理

`rememberFilesToDelete()` 在 DB 写入成功后，对比新旧值差异，将待删除的文件名存入 Record 的内部临时字段。

`processFilesToDelete()` 在 AfterCreate/AfterUpdate 钩子中真正执行删除，包括：
- 删除主文件
- 删除关联的缩略图目录（`DeletePrefix("thumbs_{filename}/")`）

---

## 六、存储驱动抽象与切换

### 6.1 Driver 接口定义

[blob/driver.go#L39-L97](file:///d:/fz/0601/solo-dogfeeding/code/163-pocketbase/tools/filesystem/blob/driver.go#L39-L97)

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

设计意图：
- 参考 gocloud.dev/blob 的精简版，保持 API 兼容
- 所有操作以 `key`（虚拟路径）为核心，屏蔽底层差异
- `NormalizeError` 统一错误类型（如将各驱动的"不存在"统一为 `blob.ErrNotFound`）

### 6.2 Bucket 封装层

[blob/bucket.go](file:///d:/fz/0601/solo-dogfeeding/code/163-pocketbase/tools/filesystem/blob/bucket.go)

`Bucket` 持有具体 Driver，提供更友好的 API：
- 线程安全（`sync.RWMutex` 保护 closed 状态）
- UTF-8 Key 校验
- Metadata 键名强制小写，保证跨驱动一致性
- 错误统一包装（含 key 信息便于调试）
- Reader/Writer 资源泄漏检测（runtime finalizer 警告未 Close）

### 6.3 System 业务封装

[filesystem.go](file:///d:/fz/0601/solo-dogfeeding/code/163-pocketbase/tools/filesystem/filesystem.go)

`System` 是对 Bucket 的进一步封装，面向业务场景：

| 方法 | 用途 |
|------|------|
| `NewLocal(dir)` | 初始化本地存储 |
| `NewS3(...)` | 初始化 S3 兼容存储 |
| `UploadFile(file, key)` | 上传 File 对象，自动检测 MIME + 保存原始文件名元数据 |
| `UploadMultipart(fh, key)` | 直接上传 multipart |
| `Serve(res, req, key, name)` | HTTP 文件服务（含 Range、缓存头、Content-Disposition） |
| `CreateThumb(...)` | 图片缩略图生成与存储 |
| `DeletePrefix(prefix)` | 批量删除（如整个 thumbs_xxx 目录） |
| `GetReuploadableFile(key, preserveName)` | 将已有文件包装为可重新上传的 File（用于复制场景） |

### 6.4 本地驱动 (fileblob)

[fileblob.go](file:///d:/fz/0601/solo-dogfeeding/code/163-pocketbase/tools/filesystem/internal/fileblob/fileblob.go)

**写入策略**（防部分写）：
1. `createTemp()` 创建临时文件（`.{timestamp}.tmp` 后缀）
2. 写入临时文件同时计算 MD5
3. `Close()` 时：
   - 写入 `.attrs` sidecar 文件（存储 ContentType、Metadata、MD5 等）
   - `os.Rename()` 原子替换为目标路径
   - 清理临时文件

**Key 转义**：
- 控制字符 0-31 → `__0x<hex>__`
- Windows 非法字符 `<>:|?*` 转义
- `../`、`//`、末尾 `/` 的斜杠转义
- `/` → `os.PathSeparator`

### 6.5 S3 驱动 (s3blob)

[s3blob.go](file:///d:/fz/0601/solo-dogfeeding/code/163-pocketbase/tools/filesystem/internal/s3blob/s3blob.go)

- 内部封装自研 `s3.S3` HTTP 客户端（非 AWS SDK）
- 支持 `UsePathStyle`（兼容 MinIO、阿里 OSS 等非 AWS S3）
- Metadata 使用 URL + Hex 双层转义
- ETag 转换为 MD5（用于完整性校验）

### 6.6 驱动切换机制

**切换入口**：`BaseApp.NewFilesystem()` [base.go#L715-L729](file:///d:/fz/0601/solo-dogfeeding/code/163-pocketbase/core/base.go#L715-L729)

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
    // fallback to local filesystem
    return filesystem.NewLocal(filepath.Join(app.DataDir(), LocalStorageDirName))
}
```

**决策逻辑**：
1. 检查 `app.settings.S3.Enabled` 是否为 true
2. 是 → 创建 S3 驱动实例
3. 否 → 回退到本地文件系统（`pb_data/storage/` 目录）

**配置来源**：Settings 存储在数据库中，可通过 Admin UI 或 API 动态修改。每次调用 `NewFilesystem()` 都基于最新设置判断，无需重启。

**备份存储独立配置**：`NewBackupsFilesystem()` 使用 `settings.Backups.S3`，可与常规文件使用不同的存储桶/账号。

---

## 七、关键设计总结

1. **文件名随机化**：所有上传文件自动追加 10 位随机后缀，既防止冲突又实现一定的"不可枚举"安全。

2. **先文件后 DB**：上传成功才写 DB，DB 失败回滚文件，避免孤儿记录。

3. **拦截器模式**：FileField 通过 `RecordInterceptor` 将文件逻辑与 Record 生命周期深度绑定，调用方无需关心文件操作细节。

4. **Driver 接口隔离**：`blob.Driver` 接口极小（8 个方法），新增存储后端只需实现该接口，上层 System/Bucket 完全复用。

5. **原子写入**：本地驱动通过临时文件 + rename 保证写入原子性，防止崩溃产生损坏文件。

6. **最终一致性**：旧文件删除在 DB 提交后异步执行，删除失败不影响主流程，通过日志可追溯补偿。
