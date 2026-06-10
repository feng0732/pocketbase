# PocketBase 备份与恢复工作流代码梳理

## 一、总体架构

备份与恢复功能的代码分布在以下核心模块：

| 模块层级 | 主要文件 | 职责 |
|---------|---------|------|
| API 层 | [apis/backup.go](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/apis/backup.go) | HTTP 路由注册、请求鉴权、响应处理 |
| API 层 | [apis/backup_create.go](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/apis/backup_create.go) | 创建备份的表单校验与请求处理 |
| API 层 | [apis/backup_upload.go](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/apis/backup_upload.go) | 备份文件上传的表单校验与请求处理 |
| 核心层 | [core/base_backup.go](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base_backup.go) | 备份创建、恢复、定时自动备份的核心业务逻辑 |
| 工具层 | [tools/archive/create.go](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/tools/archive/create.go) | ZIP 压缩归档实现 |
| 工具层 | [tools/archive/extract.go](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/tools/archive/extract.go) | ZIP 解压提取实现 |
| 工具层 | [tools/osutils/dir.go](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/tools/osutils/dir.go) | 目录内容移动（原子替换核心） |

---

## 二、数据导出流程（CreateBackup）

### 2.1 调用链边界

```
HTTP POST /api/backups
        │
        ▼
[apis/backup_create.go] backupCreate()
        │  ├─ 并发检查：StoreKeyActiveBackup
        │  ├─ 表单校验：backupCreateForm.validate()
        │  └─ 名称唯一性检查
        ▼
[core/base_backup.go] CreateBackup()
        │  ├─ 触发 OnBackupCreate Hook
        │  ├─ 生成备份名称（若为空）
        │  ├─ 创建临时目录 pb_data/.pb_temp_to_delete
        │  ├─ RunInTransaction（阻塞写操作）
        │  │     └─ AuxRunInTransaction
        │  │           ├─ WAL Checkpoint: PRAGMA wal_checkpoint(TRUNCATE)
        │  │           └─ 调用 archive.Create()
        │  ├─ 通过 NewBackupsFilesystem() 持久化
        │  └─ 清理临时文件
        ▼
[tools/archive/create.go] Create()
        │  └─ zipAddFS() 遍历 pb_data，排除指定目录
        ▼
存储层：本地 pb_data/backups/ 或 S3
```

### 2.2 关键边界点

**边界 1：并发互斥锁**
- 位置：[base_backup.go:L45-L50](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base_backup.go#L45-L50)
- 机制：通过 `app.Store().Has(StoreKeyActiveBackup)` 检查是否有进行中的备份/恢复操作
- 作用：防止并发备份或备份与恢复同时执行导致的数据不一致

**边界 2：事务写阻塞**
- 位置：[base_backup.go:L83-L92](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base_backup.go#L83-L92)
- 机制：嵌套调用 `RunInTransaction` + `AuxRunInTransaction`
- 作用：利用 SQLite 的 NonconcurrentDB 连接，临时阻塞其他数据库写入，确保归档时数据一致性

**边界 3：WAL Checkpoint**
- 位置：[base_backup.go:L87-L88](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base_backup.go#L87-L88)
- 机制：执行 `PRAGMA wal_checkpoint(TRUNCATE)`
- 作用：将 WAL 日志中的变更写入主数据库文件并截断 WAL，避免备份包含未提交的 WAL 数据

**边界 4：目录排除列表**
- 位置：[base_backup.go:L57-L63](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base_backup.go#L57-L63)
- 备份时排除的 `pb_data` 根目录条目：
  - `backups` — 备份目录自身，避免无限递归
  - `.pb_temp_to_delete` — 临时目录，下次启动会自动删除
  - `.notify` — 跨实例同步目录
  - `.autocert_cache` — HTTPS 证书缓存
  - `lost+found` — 文件系统修复目录

**边界 5：存储层抽象**
- 位置：[base.go:L739-L749](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base.go#L739-L749)
- 机制：`NewBackupsFilesystem()` 根据配置返回 S3 或本地文件系统
- 本地存储路径：`pb_data/backups/`

### 2.3 备份文件命名规则
- 位置：[base_backup.go:L410-L422](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base_backup.go#L410-L422)
- 格式：`{prefix}{app_name}_{YYYYMMDDHHMMSS}.zip`
- 自动备份前缀：`@auto_pb_backup_`
- 手动备份前缀：`pb_backup_`

---

## 三、压缩归档流程（archive 包）

### 3.1 Create — 压缩归档

```
archive.Create(srcDir, destZipPath, exclude...)
        │
        ├─ os.MkdirAll(destDir)  — 创建目标目录
        ├─ os.Create(destZipPath) — 创建 zip 文件
        ├─ zip.NewWriter() + flate.BestSpeed — 注册快速压缩器
        └─ zipAddFS(os.DirFS(src), exclude...)
                │
                └─ fs.WalkDir 遍历源目录
                        ├─ 跳过目录（不添加空目录条目）
                        ├─ 检查排除路径（精确匹配或前缀匹配）
                        ├─ zip.FileInfoHeader() + Deflate 方法
                        └─ io.Copy 将文件内容写入 zip
```

**关键边界点：**

**边界 1：排除路径匹配逻辑**
- 位置：[create.go:L56-L61](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/tools/archive/create.go#L56-L61)
- 两种匹配方式：
  1. 精确匹配：`ignore == name`
  2. 目录前缀匹配：`clean(name) + "/"` 以 `clean(ignore) + "/"` 开头
- 注意：只在根目录层级排除，不会递归深入子目录排除

**边界 2：压缩级别**
- 位置：[create.go:L31-L33](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/tools/archive/create.go#L31-L33)
- 使用 `flate.BestSpeed` 而非默认压缩级别，优先保证备份速度

**边界 3：错误清理**
- 位置：[create.go:L36-L39](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/tools/archive/create.go#L36-L39)
- 若压缩过程出错，使用 `errors.Join` 聚合：关闭 writer、关闭文件、删除不完整的 zip

### 3.2 Extract — 解压提取

```
archive.Extract(srcZipPath, destDir)
        │
        ├─ zip.OpenReader() — 打开 zip
        ├─ Clean(dest) + "/" — 规范化目标路径（防 Zip Slip）
        └─ 遍历 zip.File，逐个 extractFile()
                │
                ├─ filepath.Join 拼接目标路径
                ├─ Zip Slip 检查：路径必须以 basePath 为前缀
                ├─ 目录：os.MkdirAll
                └─ 普通文件：
                    ├─ os.MkdirAll(filepath.Dir)
                    ├─ os.OpenFile(O_WRONLY|O_CREATE|O_TRUNC)
                    └─ io.Copy 写入内容
```

**关键边界点：**

**边界 1：Zip Slip 防护**
- 位置：[extract.go:L42-L45](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/tools/archive/extract.go#L42-L45)
- 检查解压后路径是否仍在目标目录内，防止恶意 zip 通过 `../` 路径穿越

**边界 2：仅处理常规文件**
- 位置：[extract.go:L54-L74](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/tools/archive/extract.go#L54-L74)
- 只提取 **目录** 和 **普通文件**
- 符号链接、命名管道、套接字等非常规文件会被静默跳过

---

## 四、恢复覆盖流程（RestoreBackup）

### 4.1 调用链边界

```
HTTP POST /api/backups/{key}/restore
        │
        ▼
[apis/backup.go] backupRestore()
        │  ├─ 并发检查
        │  ├─ 校验备份文件存在
        │  └─ routine.FireAndForget — 异步执行（先返回 204）
        │        └─ time.Sleep(1s) — 等待响应写出
        ▼
[core/base_backup.go] RestoreBackup()
        │  ├─ 触发 OnBackupRestore Hook
        │  ├─ 平台检查：Windows 不支持
        │  ├─ 创建临时目录 pb_data/.pb_temp_to_delete
        │  ├─ 获取备份文件系统
        │  │
        │  ├─ [分支 A] S3 存储：
        │  │     ├─ fsys.GetReader() 获取 blob 流
        │  │     ├─ os.CreateTemp 创建临时 zip
        │  │     ├─ io.Copy 写入临时文件
        │  │     └─ archive.Extract(tempZip, extractedDataDir)
        │  │
        │  ├─ [分支 B] 本地存储：
        │  │     └─ archive.Extract(pb_data/backups/{name}, extractedDataDir)
        │  │
        │  ├─ 校验：extractedDataDir/data.db 必须存在
        │  │
        │  ├─ RunInTransaction（写阻塞）
        │  │     └─ AuxRunInTransaction
        │  │           ├─ MoveDirContent(pb_data → oldTempDataDir, exclude)
        │  │           └─ MoveDirContent(extractedDataDir → pb_data, exclude)
        │  │
        │  ├─ 失败回滚函数 revertDataDirChanges()
        │  │
        │  └─ e.App.Restart() — 重启应用进程
        │        └─ 若重启失败，调用 revertDataDirChanges() 回滚
        ▼
[core/base.go] Restart()
        └─ execve(execPath, os.Args, os.Environ()) — UNIX 进程替换
```

### 4.2 关键边界点

**边界 1：平台限制**
- 位置：[base_backup.go:L171-L173](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base_backup.go#L171-L173)
- 恢复功能仅支持 UNIX 系统，Windows 直接返回错误
- 原因：依赖 `execve` 系统调用进行进程替换

**边界 2：本地 vs S3 解压差异**
- 位置：[base_backup.go:L198-L243](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base_backup.go#L198-L243)
- S3：先下载 blob 到临时 zip 文件，再解压（blob.Reader 不支持 ReaderAt）
- 本地：直接读取 `pb_data/backups/` 下的 zip 文件解压，避免额外拷贝

**边界 3：恢复前完整性校验**
- 位置：[base_backup.go:L246-L249](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base_backup.go#L246-L249)
- 解压后必须存在 `data.db` 文件，否则视为无效备份

**边界 4：原子替换（核心）**
- 位置：[base_backup.go:L253-L272](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base_backup.go#L253-L272)
- 使用 `osutils.MoveDirContent` 进行两步原子移动：
  1. 将当前 `pb_data` 内容（排除列表）移到 `oldTempDataDir`
  2. 将 `extractedDataDir` 内容移到 `pb_data`
- 整个过程包裹在两层 Transaction 中，阻塞数据库写入

**边界 5：失败回滚机制**
- 位置：[base_backup.go:L274-L288](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base_backup.go#L274-L288)
- `revertDataDirChanges()` 反向执行两步移动：
  1. `pb_data` → `extractedDataDir`
  2. `oldTempDataDir` → `pb_data`
- 若重启失败且回滚也失败，则直接 panic 防止数据损坏

**边界 6：恢复时的排除列表**
- 位置：[base_backup.go:L168](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base_backup.go#L168)
- 比备份时少排除 `.notify`：
  - `backups` — 保留现有备份
  - `.pb_temp_to_delete` — 临时目录
  - `.autocert_cache` — 证书缓存
  - `lost+found` — 文件系统修复目录

---

## 五、MoveDirContent 原子移动细节

### 5.1 实现机制

位置：[tools/osutils/dir.go:L20-L79](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/tools/osutils/dir.go#L20-L79)

```
MoveDirContent(src, dest, rootExclude...)
        │
        ├─ os.ReadDir(src) — 读取源目录条目
        ├─ os.Mkdir(dest) — 创建目标目录（若不存在）
        │
        ├─ 遍历每个条目：
        │     ├─ 跳过 rootExclude 中的根级条目
        │     ├─ os.Rename(oldPath, newPath) — 原子重命名
        │     └─ 记录已移动路径用于回滚
        │
        └─ 任一条目失败时 tryRollback()：
              └─ 反向 os.Rename 所有已移动条目
```

### 5.2 关键特性

1. **根级排除**：只排除源目录根下的直接条目，不递归子目录
2. **原子性保证**：使用 `os.Rename`（同分区内原子操作），失败时回滚所有已移动项
3. **跨设备限制**：要求 src 和 dest 在同一文件系统（避免 cross-device link 错误），这也是临时目录必须放在 `pb_data` 内的原因

---

## 六、自动备份（Cron）

位置：[base_backup.go:L304-L408](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base_backup.go#L304-L408)

### 6.1 触发时机

- `OnBootstrap` 事件：应用启动时加载定时任务
- `OnSettingsReload` 事件：配置变更时重新加载定时任务

### 6.2 保留策略

- 通过 `Settings().Backups.CronMaxKeep` 限制自动备份数量
- 超过限制时，按修改时间降序排序，删除最旧的自动备份
- 仅处理前缀为 `@auto_pb_backup_` 的备份文件

---

## 七、API 端点汇总

所有端点定义在 [apis/backup.go:L17-L25](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/apis/backup.go#L17-L25)：

| 方法 | 路径 | 权限 | 说明 |
|-----|------|------|------|
| GET | `/api/backups` | Superuser | 列出所有备份文件 |
| POST | `/api/backups` | Superuser | 创建新备份 |
| POST | `/api/backups/upload` | Superuser | 上传备份文件（MIME: application/zip） |
| GET | `/api/backups/{key}` | File Token | 下载备份文件（需 superuser file token） |
| DELETE | `/api/backups/{key}` | Superuser | 删除备份文件 |
| POST | `/api/backups/{key}/restore` | Superuser | 从指定备份恢复（异步执行） |

---

## 八、数据流向全景图

```
┌───────────────────────────────────────────────────────────────────┐
│                         备份流程（Create）                          │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│  pb_data/                    .pb_temp_to_delete/        backups/  │
│  ├── data.db                 ├── pb_backup_XXXXXX.zip ──►  xxx.zip│
│  ├── data.db-wal             │                           (本地/S3)│
│  ├── storage/                │                                    │
│  ├── backups/          (排除)│                                    │
│  ├── .pb_temp_to_delete/(排除)│                                    │
│  ├── .notify/           (排除)                                     │
│  ├── .autocert_cache/   (排除)                                     │
│  └── lost+found/        (排除)                                     │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────┐
│                         恢复流程（Restore）                         │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│  第一步：解压                                                      │
│  backups/xxx.zip ──► .pb_temp_to_delete/pb_restore_XXXXXXXX/      │
│                         ├── data.db  (校验存在)                    │
│                         └── ...                                    │
│                                                                   │
│  第二步：原子替换                                                  │
│  pb_data/*          ──► .pb_temp_to_delete/old_pb_data_XXXXXXXX/  │
│  (排除项不动)         (下次启动自动删除)                            │
│                                                                   │
│  pb_restore_XXXX/*  ──► pb_data/                                  │
│  (排除项不动)                                                     │
│                                                                   │
│  第三步：execve 重启进程                                            │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```
