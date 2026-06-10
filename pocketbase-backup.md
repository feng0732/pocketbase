# PocketBase 备份与恢复工作流代码梳理

## 一、总体架构

备份与恢复功能的代码分布在以下核心模块：

| 模块层级 | 主要文件 | 职责 |
|---------|---------|------|
| API 层 | [apis/backup.go](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/apis/backup.go) | HTTP 路由注册、请求鉴权、响应处理 |
| API 层 | [apis/backup_create.go](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/apis/backup_create.go) | 创建备份的表单校验与请求处理 |
| API 层 | [apis/backup_upload.go](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/apis/backup_upload.go) | 备份文件上传的表单校验与请求处理 |
| 核心层 | [core/base_backup.go](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base_backup.go) | 备份创建、恢复、定时自动备份的核心业务逻辑 |
| 核心层 | [core/base.go](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base.go) | 备份文件系统初始化、进程重启、启动清理 |
| 核心层 | [core/events.go](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/events.go) | BackupEvent 事件结构体定义 |
| 核心层 | [core/db_tx.go](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/db_tx.go) | RunInTransaction / AuxRunInTransaction 事务实现 |
| 工具层 | [tools/archive/create.go](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/tools/archive/create.go) | ZIP 压缩归档实现 |
| 工具层 | [tools/archive/extract.go](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/tools/archive/extract.go) | ZIP 解压提取实现 |
| 工具层 | [tools/osutils/dir.go](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/tools/osutils/dir.go) | 目录内容移动（原子替换核心，含局部回滚） |

---

## 二、数据导出流程（CreateBackup）

### 2.1 调用链边界

```
HTTP POST /api/backups
        │
        ▼
apis/backup_create.go: backupCreate()
        │  ├─ 并发检查：StoreKeyActiveBackup
        │  ├─ 表单校验：backupCreateForm.validate()
        │  └─ 名称唯一性检查
        ▼
core/base_backup.go: CreateBackup()
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
tools/archive/create.go: Create()
        │  └─ zipAddFS() 遍历 pb_data，排除指定目录
        ▼
存储层：本地 pb_data/backups/ 或 S3
```

### 2.2 关键边界点

**边界 1：并发互斥锁**
- 位置：[core/base_backup.go#L45-L50](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base_backup.go#L45-L50)
- 机制：通过 `app.Store().Has(StoreKeyActiveBackup)` 检查是否有进行中的备份/恢复操作
- 作用：防止并发备份或备份与恢复同时执行导致的数据不一致

**边界 2：事务写阻塞**
- 位置：[core/base_backup.go#L83-L92](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base_backup.go#L83-L92)
- 机制：嵌套调用 `RunInTransaction` + `AuxRunInTransaction`
- 实现细节：[core/db_tx.go#L14-L49](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/db_tx.go#L14-L49) 中使用 `NonconcurrentDB` 连接开启 SQLite 事务
- 作用：利用 SQLite 事务锁，临时阻塞其他数据库写入，确保归档期间 data.db 不会被修改

**边界 3：WAL Checkpoint（详细含义）**
- 位置：[core/base_backup.go#L87-L88](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base_backup.go#L87-L88)
- 执行语句：`PRAGMA wal_checkpoint(TRUNCATE)`

**WAL（Write-Ahead Logging）机制背景：**
SQLite 默认使用 rollback journal，WAL 模式是替代方案：
- 写操作不直接修改 `data.db`，而是追加写入 `data.db-wal` 文件
- 读操作读取 `data.db` + `data.db-wal` 合并后的视图
- 当 WAL 文件积累到一定阈值（默认 1000 页），自动触发 Checkpoint 将 WAL 页写回 `data.db`

**`wal_checkpoint(TRUNCATE)` 的具体含义：**

| 模式 | 行为 |
|------|------|
| `PASSIVE` | 不阻塞读写，尽可能把已提交的 WAL 页写回主库，不截断 WAL |
| `FULL` | 阻塞新写，将所有 WAL 页写回主库，同步到磁盘后返回 |
| `RESTART` | 同 FULL，但之后将 WAL 头重置，后续写入从 WAL 开头开始覆盖 |
| **`TRUNCATE`** | **阻塞新写，将所有 WAL 页写回主库 fsync 持久化，然后将 WAL 文件截断为 0 字节** |

**为什么备份时必须执行：**
1. 确保 `data.db` 本身是完整的最新状态（而不是依赖 WAL 回放）
2. 截断 `data.db-wal` 为空文件，避免备份包里同时包含 `data.db` 和大体积 WAL
3. 如果不执行，备份出来的 `data.db` 可能缺失最后一批已提交事务，必须配合对应的 WAL 才能完整恢复
4. 代码注释说明：`errors are ignored because it is not that important and the PRAGMA may not be supported by the used driver`

**边界 4：目录排除列表**
- 位置：[core/base_backup.go#L57-L63](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base_backup.go#L57-L63)
- 备份时排除的 `pb_data` 根目录条目（常量定义见 [core/base.go#L40-L46](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base.go#L40-L46)）：
  - `backups` (`LocalBackupsDirName`) — 备份目录自身，避免无限递归打包已有备份
  - `.pb_temp_to_delete` (`LocalTempDirName`) — 临时目录，下次 Bootstrap 会自动删除整个目录
  - `.notify` (`LocalNotifyDirName`) — 多实例间运行时状态同步目录，属于运行时临时数据
  - `.autocert_cache` (`LocalAutocertCacheDirName`) — Let's Encrypt 证书缓存，可重新申请
  - `lost+found` (`lostFoundDirName`) — ext 等文件系统 fsck 产生的孤儿文件目录

**边界 5：存储层抽象**
- 位置：[core/base.go#L739-L749](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base.go#L739-L749)
- 机制：`NewBackupsFilesystem()` 根据 `Settings().Backups.S3` 配置返回 S3 或本地文件系统
- 本地存储路径：`pb_data/backups/`

### 2.3 备份文件命名规则
- 位置：[core/base_backup.go#L410-L422](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base_backup.go#L410-L422)
- 格式：`{prefix}{app_name}_{YYYYMMDDHHMMSS}.zip`
- 自动备份前缀：`@auto_pb_backup_`
- 手动备份前缀：`pb_backup_`
- app_name 超过 50 字符会被截断

---

## 三、压缩归档流程（archive 包）

### 3.1 Create — 压缩归档

```
tools/archive/create.go: Create(srcDir, destZipPath, exclude...)
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
- 位置：[tools/archive/create.go#L56-L61](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/tools/archive/create.go#L56-L61)
- 两种匹配方式：
  1. 精确匹配：`ignore == name`
  2. 目录前缀匹配：`clean(name) + "/"` 以 `clean(ignore) + "/"` 开头
- 注意：只在遍历路径上排除，不会递归进入被排除的子目录（WalkDir 本身的行为）

**边界 2：压缩级别**
- 位置：[tools/archive/create.go#L31-L33](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/tools/archive/create.go#L31-L33)
- 使用 `flate.BestSpeed`（级别 1）而非默认 `DefaultCompression`（级别 6），优先保证备份速度

**边界 3：错误清理**
- 位置：[tools/archive/create.go#L36-L39](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/tools/archive/create.go#L36-L39)
- 若压缩过程出错，使用 `errors.Join` 聚合：关闭 writer、关闭文件、删除不完整的 zip

### 3.2 Extract — 解压提取

```
tools/archive/extract.go: Extract(srcZipPath, destDir)
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
- 位置：[tools/archive/extract.go#L42-L45](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/tools/archive/extract.go#L42-L45)
- 检查解压后路径是否仍在目标目录内，防止恶意 zip 通过 `../../etc/passwd` 等相对路径写入任意位置

**边界 2：仅处理常规文件**
- 位置：[tools/archive/extract.go#L54-L74](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/tools/archive/extract.go#L54-L74)
- 只提取 **目录** 和 **普通文件**（`Mode().IsRegular()`）
- 符号链接、命名管道、套接字、设备文件等非常规文件会被静默跳过

---

## 四、恢复覆盖流程（RestoreBackup）

### 4.1 调用链边界

```
HTTP POST /api/backups/{key}/restore
        │
        ▼
apis/backup.go: backupRestore()
        │  ├─ 并发检查
        │  ├─ 校验备份文件存在
        │  └─ routine.FireAndForget — 异步执行（先返回 204 No Content）
        │        └─ time.Sleep(1s) — 等待 HTTP 响应写出后再开始实际恢复
        ▼
core/base_backup.go: RestoreBackup()
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
        │  │           ├─ Step A: MoveDirContent(pb_data → oldTempDataDir, exclude)
        │  │           └─ Step B: MoveDirContent(extractedDataDir → pb_data, exclude)
        │  │
        │  ├─ 定义 revertDataDirChanges() 回滚函数（仅 Restart 失败时调用）
        │  │
        │  └─ e.App.Restart() — execve 替换当前进程
        │        └─ 若重启失败，调用 revertDataDirChanges() 回滚，回滚失败则 panic
        ▼
core/base.go: Restart()
        └─ execve(execPath, os.Args, os.Environ()) — UNIX 进程替换
```

### 4.2 关键边界点

**边界 1：平台限制**
- 位置：[core/base_backup.go#L171-L173](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base_backup.go#L171-L173)
- 恢复功能仅支持 UNIX 系统，Windows 直接返回错误
- 原因：依赖 `execve` 系统调用进行原子进程替换（Windows 无此机制）

**边界 2：本地 vs S3 解压差异**
- 位置：[core/base_backup.go#L198-L243](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base_backup.go#L198-L243)
- S3：先下载 blob 到临时 zip 文件，再解压（blob.Reader 不实现 `ReaderAt`，而 `zip.OpenReader` 需要随机访问）
- 本地：直接读取 `pb_data/backups/` 下的 zip 文件路径传给 `archive.Extract`，避免额外磁盘拷贝

**边界 3：恢复前完整性校验**
- 位置：[core/base_backup.go#L246-L249](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base_backup.go#L246-L249)
- 解压后必须存在 `data.db` 文件，否则视为无效备份直接返回错误
- 此时 `pb_data` 尚未被触碰，无任何副作用

**边界 4：原子替换（核心）**
- 位置：[core/base_backup.go#L253-L272](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base_backup.go#L253-L272)
- 使用 `osutils.MoveDirContent` 进行两步原子移动：
  - **Step A**：将当前 `pb_data` 内容（排除列表）移到 `oldTempDataDir`
  - **Step B**：将 `extractedDataDir` 内容移到 `pb_data`
- 整个过程包裹在两层 Transaction 中，阻塞数据库写入

**边界 5：失败回滚机制（见下方 §4.3 详细分析）**

**边界 6：恢复时的排除列表**
- 位置：[core/base_backup.go#L168](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base_backup.go#L168)
- 比备份时少排除 `.notify`：
  - `backups` — 保留现有备份不被覆盖
  - `.pb_temp_to_delete` — 临时目录，含本次恢复的 old / extracted 数据
  - `.autocert_cache` — HTTPS 证书缓存
  - `lost+found` — 文件系统修复目录

### 4.3 恢复覆盖失败边界深度分析

恢复流程中存在 **两层回滚**，触发时机完全不同，必须严格区分：

```
┌───────────────────────────────────────────────────────────────────────┐
│                     RestoreBackup 失败场景全景                         │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  Phase 1: 准备阶段（解压前）                                            │
│  ├─ 并发检查 / 平台检查 / 临时目录创建 / 文件系统初始化 失败            │
│  ├─ 备份文件不存在                                                      │
│  └─ 后果：pb_data 完全未触碰，直接返回错误                               │
│                                                                       │
│  Phase 2: 解压阶段                                                     │
│  ├─ S3 下载失败 / 解压失败 / data.db 缺失                               │
│  ├─ 清理：defer os.RemoveAll(extractedDataDir) 自动执行                 │
│  └─ 后果：pb_data 完全未触碰，仅临时目录有垃圾（下次启动清理）           │
│                                                                       │
│  Phase 3: 原子替换阶段（关键！）                                        │
│  │                                                                     │
│  ├─ Step A 失败（MoveDirContent pb_data → oldTempDataDir）             │
│  │   ├─ MoveDirContent 内部 tryRollback() 局部回滚                      │
│  │   ├─ 已搬走的条目被 Rename 回 pb_data                                 │
│  │   ├─ 新创建的 oldTempDataDir 被尝试删除                               │
│  │   └─ 后果：pb_data 完整如初，replaceErr 被返回                       │
│  │                                                                     │
│  ├─ Step A 成功，Step B 失败（MoveDirContent extracted → pb_data）      │
│  │   ├─ MoveDirContent 内部 tryRollback() 只回滚 Step B 已搬走的条目     │
│  │   ├─ ⚠️ Step A 的结果 **不会被自动回滚**                              │
│  │   ├─ 此时磁盘状态：                                                  │
│  │   │   ├── pb_data/  = 排除列表中的条目 + Step B 部分成功的备份数据    │
│  │   │   ├── oldTempDataDir/  = Step A 搬走的原 pb_data 完整数据        │
│  │   │   └── extractedDataDir/  = 剩余未移动的备份数据                  │
│  │   ├─ replaceErr 被直接 return                                        │
│  │   ├─ extractedDataDir 通过 defer 删除                                │
│  │   ├─ ⚠️ oldTempDataDir **没有 defer 删除**                           │
│  │   └─ 后果：pb_data 处于中间不一致状态！                               │
│  │        原数据完整躺在 .pb_temp_to_delete/old_pb_data_* 中            │
│  │        需要手动恢复，或下次启动会被误删（.pb_temp_to_delete 整体清空） │
│  │                                                                     │
│  Phase 4: 重启阶段                                                     │
│  ├─ Step A、Step B 均成功，App.Restart() 失败                           │
│  ├─ 调用 revertDataDirChanges() 真正回滚：                               │
│  │   ├─ MoveDirContent(pb_data → extractedDataDir)  — 退回备份数据     │
│  │   └─ MoveDirContent(oldTempDataDir → pb_data)   — 还原原数据        │
│  ├─ 若回滚也失败 → panic(fmt.Errorf) 终止进程                            │
│  └─ 后果：要么 pb_data 恢复为原状态，要么进程崩溃                         │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

**两层回滚的本质区别：**

| 回滚层级 | 实现位置 | 触发时机 | 回滚范围 |
|---------|---------|---------|---------|
| 局部回滚 `tryRollback()` | [tools/osutils/dir.go#L37-L54](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/tools/osutils/dir.go#L37-L54) | 单次 `MoveDirContent` 内部某条 `os.Rename` 失败时 | 仅回滚本次 `MoveDirContent` 已成功移动的条目，不影响其他操作 |
| 全局回滚 `revertDataDirChanges()` | [core/base_backup.go#L274-L288](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base_backup.go#L274-L288) | 仅在 `App.Restart()` 返回错误时调用 | 反向执行两步完整的 `MoveDirContent`，将 pb_data 恢复到恢复前状态 |

**关键结论：**
- Step A 成功 + Step B 失败这个场景是**危险窗口**：原数据已搬走但没有触发全局回滚
- 这个窗口的存在是因为 `replaceErr` 直接 return，没有走 `revertDataDirChanges` 逻辑
- `oldTempDataDir` 没有独立的 `defer os.RemoveAll`，依赖于上层 `LocalTempDirName` 目录在下次 Bootstrap 时整体删除（见 [core/base.go#L431](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base.go#L431)）
- 下次 Bootstrap 时 `os.RemoveAll(pb_data/.pb_temp_to_delete)` 会把原数据一并删除，如果 Step B 失败后未手动干预，原数据永久丢失

---

## 五、MoveDirContent 原子移动细节

### 5.1 实现机制

位置：[tools/osutils/dir.go#L20-L79](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/tools/osutils/dir.go#L20-L79)

```
MoveDirContent(src, dest, rootExclude...)
        │
        ├─ os.ReadDir(src) — 读取源目录条目
        ├─ os.Mkdir(dest) — 创建目标目录（若不存在，记录 manuallyCreatedDestDir）
        │
        ├─ 遍历每个条目：
        │     ├─ 跳过 rootExclude 中的根级条目
        │     ├─ os.Rename(oldPath, newPath) — 同分区内原子重命名
        │     ├─ 成功：记录 old→new 到 moved map
        │     └─ 失败：调用 tryRollback()
        │           ├─ 遍历 moved map，反向 os.Rename(new, old)
        │           ├─ 若 dest 是本次新建且全部回滚成功，尝试 os.Remove(dest)
        │           └─ 聚合所有错误后返回
        │
        └─ 全部成功：返回 nil
```

### 5.2 关键特性

1. **根级排除**：`rootExclude` 只匹配源目录根下的直接条目名称（basename），不递归进入子目录
2. **原子性粒度**：单文件 `os.Rename` 在同分区是原子的，但整个 `MoveDirContent` 不是事务性的——它通过回滚补偿来实现"最终原子"
3. **跨设备限制**：要求 src 和 dest 在同一文件系统（`os.Rename` 不支持跨分区），这也是临时目录必须放在 `pb_data` 内的原因
4. **不删除源目录**：只移动源目录的内容，源目录本身保留

---

## 六、自动备份（Cron）

位置：[core/base_backup.go#L304-L408](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/core/base_backup.go#L304-L408)

### 6.1 触发时机

- `OnBootstrap` 事件：应用启动时加载定时任务
- `OnSettingsReload` 事件：配置变更时重新加载/移除定时任务

### 6.2 保留策略

- 通过 `Settings().Backups.CronMaxKeep` 限制自动备份数量
- 超过限制时，按 `ModTime` 降序排序，删除最旧的自动备份
- 仅处理前缀为 `@auto_pb_backup_` 的备份文件（不影响手动备份）
- 备份失败时会给所有超管发送系统告警邮件

---

## 七、API 端点汇总

所有端点定义在 [apis/backup.go#L17-L25](file:///d:/fz/0601/solo-dogfeeding/code/164-pocketbase/apis/backup.go#L17-L25)：

| 方法 | 路径 | 权限 | 说明 |
|-----|------|------|------|
| GET | `/api/backups` | Superuser | 列出所有备份文件 |
| POST | `/api/backups` | Superuser | 创建新备份 |
| POST | `/api/backups/upload` | Superuser | 上传备份文件（MIME: application/zip） |
| GET | `/api/backups/{key}` | File Token | 下载备份文件（需 superuser file token） |
| DELETE | `/api/backups/{key}` | Superuser | 删除备份文件（不能删除正在使用的备份） |
| POST | `/api/backups/{key}/restore` | Superuser | 从指定备份恢复（异步执行，先返回 204） |

---

## 八、数据流向全景图

```
┌──────────────────────────────────────────────────────────────────────┐
│                        备份流程（CreateBackup）                       │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Transaction 写阻塞期间：                                             │
│                                                                      │
│  pb_data/                     .pb_temp_to_delete/         backups/   │
│  ├── data.db   ─────────────►  pb_backup_XXXXXX.zip  ────►  xxx.zip  │
│  ├── data.db-wal  (WAL 截断)                           (本地/S3)     │
│  ├── storage/                                                         │
│  ├── backups/              (排除)                                     │
│  ├── .pb_temp_to_delete/   (排除)                                     │
│  ├── .notify/              (排除)                                     │
│  ├── .autocert_cache/      (排除)                                     │
│  └── lost+found/           (排除)                                     │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                       恢复流程（RestoreBackup）                       │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  第一步：解压（pb_data 未触碰）                                        │
│  backups/xxx.zip ──► .pb_temp_to_delete/pb_restore_XXXXXXXX/         │
│                           ├── data.db  (校验存在)                     │
│                           └── ...                                     │
│                                                                      │
│  第二步：Step A（原数据迁出）                                          │
│  pb_data/{data.db,storage,...}  ──► .pb_temp_to_delete/              │
│  (排除项留在原地)                        old_pb_data_XXXXXXXX/        │
│                                          ├── data.db                 │
│                                          └── ...                     │
│                                                                      │
│  第三步：Step B（备份数据迁入）                                        │
│  .pb_temp_to_delete/                                                  │
│    pb_restore_XXXXXXXX/*  ──►  pb_data/                              │
│    (排除项留在原地)         (排除项保留不动)                           │
│                                                                      │
│  第四步：execve 重启进程（成功路径）                                    │
│     或 Restart 失败 → revertDataDirChanges() 完整回滚                  │
│     或 Step B 中途失败 → 仅 Step B 局部回滚（⚠️ 原数据留在 old 目录）    │
│                                                                      │
│  下次启动：os.RemoveAll(.pb_temp_to_delete) 清理全部临时数据            │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```
