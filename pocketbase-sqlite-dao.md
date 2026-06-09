# PocketBase SQLite 嵌入与 DAO 抽象边界分析

本文档从代码实现角度梳理 PocketBase 中 SQLite 数据库嵌入层与 DAO（数据访问对象）抽象层之间的边界设计，重点覆盖**连接池**、**事务封装**、**读写拆分**和**并发写入**四大核心机制，所有参数和行为均基于源代码核实。

---

## 一、整体架构分层

PocketBase 的数据访问层采用**两层架构**，通过 `dbx.Builder` 接口解耦：

| 层次 | 职责 | 核心文件 |
|------|------|----------|
| **SQLite 嵌入层** | 驱动加载、连接池管理、WAL 模式配置、PRAGMA 参数设置 | [db_connect.go](core/db_connect.go)、[base.go](core/base.go) |
| **DAO 抽象层** | 模型 CRUD、事务生命周期、Hook 系统、读写路由、锁重试 | [db.go](core/db.go)、[db_tx.go](core/db_tx.go)、[db_builder.go](core/db_builder.go)、[db_retry.go](core/db_retry.go) |

依赖关系：`github.com/pocketbase/dbx v1.12.0`（查询构建器）+ `modernc.org/sqlite v1.52.0`（纯 Go SQLite 驱动）。

```
用户代码 → App.Save() / Delete() / ModelQuery() / RunInTransaction()
               ↓
         ┌── DAO 抽象层 ────────────────────────────────┐
         │  Hook 生命周期 · 事务封装 · 读写路由 · 重试   │
         └───────────────────────┬───────────────────────┘
                                 ↓
         ┌── dbx.Builder 接口 ──────────────────────────┐
         │  dualDBBuilder（非事务） / *dbx.Tx（事务）    │
         └───────────────────────┬───────────────────────┘
                                 ↓
         ┌── SQLite 嵌入层 ─────────────────────────────┐
         │  连接池 · WAL · modernc.org/sqlite 驱动       │
         └───────────────────────────────────────────────┘
```

---

## 二、连接池设计：双池分离

### 2.1 双连接池策略

SQLite 在 WAL 模式下虽支持"一读一写"并发，但同一时刻仍只允许一个写事务。PocketBase 采用**双池策略**最小化 `SQLITE_BUSY`：

- **Concurrent Pool（并发读池）**：多连接，专用于 SELECT 查询
- **Nonconcurrent Pool（非并发写池）**：**强制单连接**，串行化所有写入

### 2.2 连接池初始化代码

主数据库 `data.db` 初始化见 [base.go#L1175-L1209](core/base.go#L1175-L1209)：

```go
func (app *BaseApp) initDataDB() error {
    dbPath := filepath.Join(app.DataDir(), "data.db")

    // ── 并发读池 ──
    concurrentDB, _ := app.config.DBConnect(dbPath)
    concurrentDB.DB().SetMaxOpenConns(app.config.DataMaxOpenConns) // 默认 120
    concurrentDB.DB().SetMaxIdleConns(app.config.DataMaxIdleConns) // 默认 15
    concurrentDB.DB().SetConnMaxIdleTime(3 * time.Minute)

    // ── 非并发写池（关键：仅 1 连接）──
    nonconcurrentDB, _ := app.config.DBConnect(dbPath)
    nonconcurrentDB.DB().SetMaxOpenConns(1)   // 所有写入在 Go 层面排队
    nonconcurrentDB.DB().SetMaxIdleConns(1)
    nonconcurrentDB.DB().SetConnMaxIdleTime(3 * time.Minute)

    app.concurrentDB = concurrentDB
    app.nonconcurrentDB = nonconcurrentDB
    return nil
}
```

辅助数据库 `auxiliary.db` 初始化见 [base.go#L1235-L1260](core/base.go#L1235-L1260)，逻辑完全相同。

### 2.3 连接池参数汇总

默认常量定义在 [base.go#L32-L37](core/base.go#L32-L37)：

| 参数 | data.db 默认值 | auxiliary.db 默认值 | 作用 |
|------|---------------|---------------------|------|
| `DataMaxOpenConns` | **120** | — | 并发读池最大活跃连接数 |
| `DataMaxIdleConns` | **15** | — | 并发读池最大空闲连接数 |
| `AuxMaxOpenConns` | — | **20** | 辅助库并发读池活跃连接数 |
| `AuxMaxIdleConns` | — | **3** | 辅助库并发读池空闲连接数 |
| 写池 `MaxOpenConns` | **1** | **1** | 所有写入强制串行（两个库都一样） |
| 写池 `MaxIdleConns` | **1** | **1** | 同上 |
| `ConnMaxIdleTime` | **3 分钟** | **3 分钟** | 空闲连接回收时间 |
| `QueryTimeout` | **30 秒** | **30 秒** | 单条查询超时 |

### 2.4 SQLite 驱动加载与 PRAGMA 配置

连接建立由 [db_connect.go](core/db_connect.go) 中的 `DefaultDBConnect` 负责：

```go
func DefaultDBConnect(dbPath string) (*dbx.DB, error) {
    // 注意：busy_timeout 必须第一个设置，因为启用 WAL 前就需要阻塞等待能力
    pragmas := "?" +
        "_pragma=busy_timeout(10000)" +
        "&_pragma=journal_mode(WAL)" +
        "&_pragma=journal_size_limit(200000000)" +
        "&_pragma=synchronous(NORMAL)" +
        "&_pragma=foreign_keys(ON)" +
        "&_pragma=temp_store(MEMORY)" +
        "&_pragma=cache_size(-32000)"

    return dbx.Open("sqlite", dbPath+pragmas)
}
```

各 PRAGMA 参数详解：

| PRAGMA | 值 | 准确含义 |
|--------|-----|---------|
| `busy_timeout` | **10000** | SQLite 驱动层面锁等待超时，单位毫秒（10 秒）。在锁释放前阻塞等待 |
| `journal_mode` | **WAL** | Write-Ahead Logging 模式，实现"多读单写"并发 |
| `journal_size_limit` | **200000000** | WAL 文件大小上限，200,000,000 字节 = 约 190.7 MB |
| `synchronous` | **NORMAL** | WAL 模式下推荐设置：仅在 checkpoint 时做 `fsync`，普通提交不等待 |
| `foreign_keys` | **ON** | 启用外键约束检查 |
| `temp_store` | **MEMORY** | 临时表和临时索引存储在内存而非磁盘 |
| `cache_size` | **-32000** | **页缓存大小**。负值表示以 **KB** 为单位：`abs(-32000) × 1024 ≈ 32 MB`。SQLite 内部会根据当前 `page_size`（默认 4096 字节）换算成实际缓存页数。正数则直接表示页数 |

> **cache_size 重点说明**：SQLite 记住的是缓存页数，而非字节数。因此先用负值指定 KB 数，SQLite 再根据实际页面大小换算为页数。如果后续改变了 `page_size`，缓存对应的内存量也会随之变化。

### 2.5 双数据库实例

PocketBase 维护两个物理 SQLite 文件（各自拥有独立的双连接池）：

| 数据库文件 | 用途 | 访问方法 |
|-----------|------|---------|
| `data.db` | 主业务数据：collections、records、settings、auth 等 | `DB()` / `ConcurrentDB()` / `NonconcurrentDB()` |
| `auxiliary.db` | 辅助数据：logs 等（注：曾使用过三字母辅助库名称，后因 Windows 保留字 "AUX" 改名） | `AuxDB()` / `AuxConcurrentDB()` / `AuxNonconcurrentDB()` |

---

## 三、读写拆分：dualDBBuilder 智能路由

### 3.1 自动路由入口

用户通常调用 `app.DB()` 获取 `dbx.Builder`。`DB()` 方法会判断当前状态并返回合适的实例，见 [base.go#L490-L500](core/base.go#L490-L500)：

```go
func (app *BaseApp) DB() dbx.Builder {
    // 事务中：concurrentDB == nonconcurrentDB == 同一个 *dbx.Tx
    // 此时直接返回事务对象，不走 dualDBBuilder
    if app.concurrentDB == app.nonconcurrentDB {
        return app.concurrentDB
    }
    // 非事务中：返回 dualDBBuilder 做读写路由
    return &dualDBBuilder{
        concurrentDB:    app.concurrentDB,
        nonconcurrentDB: app.nonconcurrentDB,
    }
}
```

### 3.2 dualDBBuilder 的方法级路由表

[db_builder.go](core/db_builder.go) 实现了 `dbx.Builder` 接口，将每个方法路由到不同连接池：

| 方法 | 路由目标 | 说明 |
|------|---------|------|
| `Select(cols ...string)` | concurrentDB | 读操作 |
| `Quote() / QueryBuilder() / GeneratePlaceholder()` | concurrentDB | 纯计算，不涉及 IO |
| `Model(data interface{})` | **nonconcurrentDB** | 模型级 CRUD（Insert/Update） |
| `Insert() / Upsert() / Update() / Delete()` | nonconcurrentDB | 写操作 |
| `CreateTable() / DropTable() / AddColumn() / CreateIndex()` 等 DDL | nonconcurrentDB | 写操作（DDL 一律走写池） |
| `NewQuery(sql string)` | **智能判断** | 见下文 |

### 3.3 NewQuery：SQL 前缀检测路由

对于原始 SQL 查询，通过检测前缀来路由，见 [db_builder.go#L151-L160](core/db_builder.go#L151-L160)：

```go
func (b *dualDBBuilder) NewQuery(str string) *dbx.Query {
    trimmed := trimLeftSpaces(str)
    // SELECT 或 WITH（CTE 公共表表达式）→ 读池
    if hasPrefixFold(trimmed, "SELECT") || hasPrefixFold(trimmed, "WITH") {
        return b.concurrentDB.NewQuery(str)
    }
    // 其他所有情况（INSERT / UPDATE / DELETE / PRAGMA / VACUUM 等）→ 写池
    return b.nonconcurrentDB.NewQuery(str)
}
```

> **设计取舍**：不处理 `WITH ... INSERT`、`INSERT ... RETURNING` 等复杂场景，这类场景较少见，避免过度复杂化。

### 3.4 DAO 层的显式池选择

在 DAO 层（[db.go](core/db.go)）中，各操作显式选择对应池：

| 操作 | 使用的池 | 代码位置 |
|------|---------|---------|
| `ModelQuery()`（模型 SELECT 查询） | ConcurrentDB | [db.go#L67-L69](core/db.go#L67-L69) |
| `Save()` → `create()`（INSERT） | NonconcurrentDB | [db.go#L290-L296](core/db.go#L290-L296) |
| `Save()` → `update()`（UPDATE） | NonconcurrentDB | [db.go#L385-L391](core/db.go#L385-L391) |
| `Delete()`（DELETE） | NonconcurrentDB | [db.go#L124-L130](core/db.go#L124-L130) |
| `validateRecordId()`（存在性检查 SELECT） | ConcurrentDB | [db.go#L487-L491](core/db.go#L487-L491) |
| `HasTable() / TableInfo() / TableColumns()`（元数据查询） | ConcurrentDB | [db_table.go#L101-L107](core/db_table.go#L101-L107) |
| `DeleteTable() / Vacuum()`（DDL / 维护操作） | NonconcurrentDB | [db_table.go#L90-L95](core/db_table.go#L90-L95) |

---

## 四、事务封装：App 浅克隆 + TxAppInfo

### 4.1 事务 API 入口

事务总是从**非并发写池**启动（确保同一事务内读写使用同一连接），见 [db_tx.go#L11-L23](core/db_tx.go#L11-L23)：

```go
// 主库事务
func (app *BaseApp) RunInTransaction(fn func(txApp App) error) error {
    return app.runInTransaction(app.NonconcurrentDB(), fn, false)
}

// 辅助库事务
func (app *BaseApp) AuxRunInTransaction(fn func(txApp App) error) error {
    return app.runInTransaction(app.AuxNonconcurrentDB(), fn, true)
}
```

### 4.2 嵌套事务的复用机制

`runInTransaction` 支持安全嵌套，核心逻辑是类型判断，见 [db_tx.go#L25-L49](core/db_tx.go#L25-L49)：

```go
func (app *BaseApp) runInTransaction(db dbx.Builder, fn func(txApp App) error, isForAuxDB bool) error {
    switch txOrDB := db.(type) {
    // ── 情况 A：已在事务中（db 是 *dbx.Tx）──
    case *dbx.Tx:
        // 直接复用现有事务，不开启新事务，不创建新 App 克隆
        // 嵌套的 OnComplete 回调会注册到同一个 txInfo
        return fn(app)

    // ── 情况 B：不在事务中（db 是 *dbx.DB）──
    case *dbx.DB:
        var txApp *BaseApp
        // 通过 dbx.Transactional 开启底层 SQLite 事务
        txErr := txOrDB.Transactional(func(tx *dbx.Tx) error {
            txApp = app.createTxApp(tx, isForAuxDB)
            return fn(txApp)
        })
        // 外层事务结束后，批量执行所有 OnComplete 回调
        if txApp != nil && txApp.txInfo != nil {
            afterFuncErr := txApp.txInfo.runAfterFuncs(txErr)
            if afterFuncErr != nil {
                return errors.Join(txErr, afterFuncErr)
            }
        }
        return txErr
    }
}
```

**嵌套行为总结**：
- 外层 `RunInTransaction`：真正开启 SQLite 事务，创建新 App 克隆
- 内层 `RunInTransaction`：仅做类型断言后直接执行，`fn` 接收的是外层的 `txApp`，`OnComplete` 回调注册到外层的 `txInfo`
- PocketBase 层不使用 SAVEPOINT（嵌套事务回滚点），嵌套完全复用外层事务

### 4.3 事务 App 的克隆机制

通过**浅拷贝 BaseApp** 来隔离事务状态，见 [db_tx.go#L51-L69](core/db_tx.go#L51-L69)：

```go
func (app *BaseApp) createTxApp(tx *dbx.Tx, isForAuxDB bool) *BaseApp {
    clone := *app  // 浅拷贝：Hook、Store、Logger、Cron 等引用共享

    // 核心替换：读写池都指向同一个 *dbx.Tx
    // 这样事务中的 app.DB() 判断 concurrentDB == nonconcurrentDB 成立
    if isForAuxDB {
        clone.auxConcurrentDB = tx
        clone.auxNonconcurrentDB = tx
    } else {
        clone.concurrentDB = tx
        clone.nonconcurrentDB = tx
    }

    // 每个事务拥有独立的 TxAppInfo
    clone.txInfo = &TxAppInfo{
        parent:     app,       // 指向父 App（非事务版本）
        isForAuxDB: isForAuxDB,
    }

    return &clone
}
```

### 4.4 TxAppInfo：事务完成回调注册表

`TxAppInfo` 是事务与 Hook 系统的桥梁，见 [db_tx.go#L71-L112](core/db_tx.go#L71-L112)：

```go
type TxAppInfo struct {
    parent     *BaseApp                      // 父 App（非事务）
    afterFuncs []func(txErr error) error     // 延迟回调列表
    mu         sync.Mutex                    // 并发安全（事务中可能并发注册回调）
    isForAuxDB bool
}

// 注册回调：事务结束（提交或回滚）后调用
func (a *TxAppInfo) OnComplete(fn func(txErr error) error) {
    a.mu.Lock()
    defer a.mu.Unlock()
    a.afterFuncs = append(a.afterFuncs, fn)
}

// 事务结束时调用所有回调（仅调用一次，调用后清空列表）
func (a *TxAppInfo) runAfterFuncs(txErr error) error {
    a.mu.Lock()
    defer a.mu.Unlock()
    var errs []error
    for _, call := range a.afterFuncs {
        if err := call(txErr); err != nil {
            errs = append(errs, err)
        }
    }
    a.afterFuncs = nil  // 清空，防止重复调用
    if len(errs) > 0 {
        return fmt.Errorf("transaction afterFunc errors: %w", errors.Join(errs...))
    }
    return nil
}
```

### 4.5 Hook 的延迟执行机制

事务中的 `OnModelAfterCreateSuccess` / `OnModelAfterDeleteError` 等副作用 Hook 不会立即执行，而是通过 `txInfo.OnComplete()` 延迟到事务真正结束。

以 `create` 操作为例，见 [db.go#L343-L363](core/db.go#L343-L363)：

```go
if app.txInfo != nil {
    // 在事务中 → 注册延迟回调
    app.txInfo.OnComplete(func(txErr error) error {
        // 恢复 event.App 为父 App（非事务版本），因为 txApp 即将销毁
        if app.txInfo != nil && app.txInfo.parent != nil {
            event.App = app.txInfo.parent
        }

        if txErr != nil {
            // 事务回滚 → 触发 Error Hook
            event.Model.MarkAsNew() // 恢复"新建"状态
            return app.OnModelAfterCreateError().Trigger(...)
        }
        // 事务提交 → 触发 Success Hook
        return app.OnModelAfterCreateSuccess().Trigger(event)
    })
} else {
    // 不在事务中 → 立即执行 Hook
    if err := event.App.OnModelAfterCreateSuccess().Trigger(event); err != nil {
        return err
    }
}
```

**设计意义**：确保发送邮件、删除文件、触发实时消息等副作用**仅在数据真正持久化后才发生**，避免事务回滚后出现不一致。

---

## 五、并发写入：三层防护 + 重试机制

### 5.1 三层防护架构

PocketBase 通过三层机制应对 SQLite 的并发写入限制：

| 层级 | 机制 | 位置 | 作用 |
|------|------|------|------|
| **L1** | 单连接写池 | `nonconcurrentDB.SetMaxOpenConns(1)` | Go 层面串行化所有写请求，避免竞争 |
| **L2** | `busy_timeout(10000)` | SQLite PRAGMA | 驱动层面最多等待 10 秒获取锁 |
| **L3** | 应用层指数退避重试 | `baseLockRetry()` | 检测 "database is locked"，应用层再次重试 |

### 5.2 重试逻辑详解

核心实现在 [db_retry.go](core/db_retry.go)：

```go
// 重试间隔表（毫秒）。注意：索引 0 的值 50ms 永远不会被使用，因为 attempt 从 1 开始
var defaultRetryIntervals = []int{50, 100, 150, 200, 300, 400, 500, 700, 1000}

const defaultMaxLockRetries = 12  // 最多重试 12 次

func baseLockRetry(op func(attempt int) error, maxRetries int) error {
    attempt := 1
Retry:
    err := op(attempt)

    // 仅当还有剩余重试次数时才检查是否需要重试
    if err != nil && attempt <= maxRetries {
        errStr := err.Error()
        // 通过错误文本匹配（不依赖驱动特定错误码，兼容 mattn/go-sqlite3 和 modernc）
        if strings.Contains(errStr, "database is locked") ||
           strings.Contains(errStr, "table is locked") {
            time.Sleep(getDefaultRetryInterval(attempt))
            attempt++
            goto Retry
        }
    }
    return err
}
```

### 5.3 重试间隔与总等待时间计算

`getDefaultRetryInterval` 的实现，见 [db_retry.go#L64-L70](core/db_retry.go#L64-L70)：

```go
func getDefaultRetryInterval(attempt int) time.Duration {
    if attempt < 0 || attempt > len(defaultRetryIntervals)-1 {
        // 超过数组长度 → 使用最后一个值（1000ms）
        return time.Duration(defaultRetryIntervals[len(defaultRetryIntervals)-1]) * time.Millisecond
    }
    return time.Duration(defaultRetryIntervals[attempt]) * time.Millisecond
}
```

**完整重试时间线**（attempt 从 1 开始，共 12 次重试 + 1 次初始执行 = 最多 13 次执行尝试）：

| 第几次失败 | attempt 值 | 等待间隔 | 累计等待 |
|-----------|-----------|---------|---------|
| 第 1 次失败后 | 1 | 100 ms | 100 ms |
| 第 2 次失败后 | 2 | 150 ms | 250 ms |
| 第 3 次失败后 | 3 | 200 ms | 450 ms |
| 第 4 次失败后 | 4 | 300 ms | 750 ms |
| 第 5 次失败后 | 5 | 400 ms | 1,150 ms |
| 第 6 次失败后 | 6 | 500 ms | 1,650 ms |
| 第 7 次失败后 | 7 | 700 ms | 2,350 ms |
| 第 8 次失败后 | 8 | 1,000 ms | 3,350 ms |
| 第 9 次失败后 | 9 | 1,000 ms | 4,350 ms |
| 第 10 次失败后 | 10 | 1,000 ms | 5,350 ms |
| 第 11 次失败后 | 11 | 1,000 ms | 6,350 ms |
| 第 12 次失败后 | 12 | 1,000 ms | 7,350 ms |

**总计最大纯等待时间：7,350 ms = 7.35 秒**。加上每次执行的耗时和 SQLite `busy_timeout` 的 10 秒等待，最坏情况下单次写入操作可能等待较长时间。

> 注意：`defaultRetryIntervals[0] = 50ms` **永远不会被使用**，因为 `attempt` 从 1 开始。

### 5.4 两种重试入口

#### (1) execLockRetry：查询执行 Hook（带超时 Context）

用于通过 `WithExecHook` 注入到查询构建器，自动附加 `QueryTimeout`（默认 30 秒），见 [db_retry.go#L20-L41](core/db_retry.go#L20-L41)：

```go
func execLockRetry(timeout time.Duration, maxRetries int) dbx.ExecHookFunc {
    return func(q *dbx.Query, op func() error) error {
        // 如果用户没设置 Context，自动附加超时
        if q.Context() == nil {
            cancelCtx, cancel := context.WithTimeout(context.Background(), timeout)
            defer func() {
                cancel()
                q.WithContext(nil) // 执行完重置，避免影响复用
            }()
            q.WithContext(cancelCtx)
        }
        execErr := baseLockRetry(func(attempt int) error {
            return op()
        }, maxRetries)
        // 错误附加 SQL 语句（方便排查）
        if execErr != nil && !errors.Is(execErr, sql.ErrNoRows) {
            execErr = fmt.Errorf("%w; failed query: %s", execErr, q.SQL())
        }
        return execErr
    }
}
```

在 `ModelQuery()` 中自动注入，见 [db.go#L83-L85](core/db.go#L83-L85)：

```go
func (app *BaseApp) modelQuery(db dbx.Builder, m Model) *dbx.SelectQuery {
    return db.Select(...).From(...).
        WithBuildHook(func(query *dbx.Query) {
            query.WithExecHook(execLockRetry(app.config.QueryTimeout, defaultMaxLockRetries))
        })
}
```

#### (2) baseLockRetry：DAO 层直接调用

所有写入操作（Save/Delete 等）显式调用，以 `Delete` 为例，见 [db.go#L132-L138](core/db.go#L132-L138)：

```go
return baseLockRetry(func(attempt int) error {
    _, err := db.Delete(e.Model.TableName(), dbx.HashExp{
        idColumn: pk,
    }).WithContext(e.Context).Execute()
    return err
}, defaultMaxLockRetries)
```

### 5.5 定期 WAL Checkpoint 与优化

后台 Cron 每天午夜（`0 0 * * *`）执行一次数据库维护，见 [base.go#L1360-L1375](core/base.go#L1360-L1375)：

```go
app.Cron().Add("__pbDBOptimize__", "0 0 * * *", func() {
    // TRUNCATE：将 WAL 内容合并到主数据库文件，并截断 WAL 文件
    app.NonconcurrentDB().NewQuery("PRAGMA wal_checkpoint(TRUNCATE)").Execute()
    app.AuxNonconcurrentDB().NewQuery("PRAGMA wal_checkpoint(TRUNCATE)").Execute()
    // optimize：运行 ANALYZE，更新查询规划器统计信息
    app.NonconcurrentDB().NewQuery("PRAGMA optimize").Execute()
})
```

---

## 六、DAO 层 CRUD 完整流程

### 6.1 Save 操作完整流程图

```
app.Save(model)
  └─ app.save(ctx, model, withValidations, isForAuxDB)
      ├─ model.IsNew() == true  →  create
      └─ model.IsNew() == false →  update
                                  ↓
              ┌─ OnModelCreate/Update Hook 触发
              │     ├─ 可选：ValidateWithContext（包含 PreValidator / PostValidator）
              │     └─ OnModelCreate/UpdateExecute Hook 触发
              │           └─ baseLockRetry {
              │                 ├─ 获取 NonconcurrentDB（写池）
              │                 ├─ 支持 DBExporter 自定义导出 或 db.Model().Insert/Update()
              │                 ├─ 最多 12 次重试（最长总等待 7.35s）
              │                 └─ 每次重试前检测 "database is locked" 文本
              │              }
              ├─ 立即失败？→ 触发 OnModelAfter*Error Hook（恢复模型状态）
              └─ 成功？
                   ├─ 在事务中？
                   │    └─ txInfo.OnComplete(...) 注册延迟回调
                   │         ├─ 提交后 → OnModelAfter*Success
                   │         └─ 回滚后 → OnModelAfter*Error + 恢复模型状态
                   └─ 不在事务中？
                        └─ 立即触发 OnModelAfter*Success Hook
```

### 6.2 Model 接口契约

DAO 层操作的所有实体必须实现 [Model](core/db_model.go#L6-L13) 接口：

```go
type Model interface {
    TableName() string          // 映射到的数据库表名
    PK() any                    // 当前主键值（用于 INSERT / UPDATE）
    LastSavedPK() any           // 上次成功持久化的主键值（用于 UPDATE WHERE 条件）
    IsNew() bool                // 为 true 走 INSERT，为 false 走 UPDATE
    MarkAsNew()                 // 标记为新建（如事务回滚后调用）
    MarkAsNotNew()              // 标记为已持久化（INSERT 成功后调用）
}
```

[BaseModel](core/db_model.go#L16-L59) 提供默认实现，通过 `lastSavedPK` 字段追踪持久化状态：

```go
type BaseModel struct {
    lastSavedPK string  // 未导出，仅内部使用
    Id          string  `db:"id" json:"id"`
}

func (m *BaseModel) IsNew() bool     { return m.lastSavedPK == "" }
func (m *BaseModel) MarkAsNew()      { m.lastSavedPK = "" }
func (m *BaseModel) MarkAsNotNew()   { m.lastSavedPK = m.Id }
func (m *BaseModel) LastSavedPK() any { return m.lastSavedPK }
func (m *BaseModel) PK() any          { return m.Id }
func (m *BaseModel) PostScan() error  { m.MarkAsNotNew(); return nil }
```

---

## 七、架构边界总结图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        用户代码 / API / 插件层                       │
│  Save() Delete() ModelQuery() RunInTransaction() NewQuery()         │
└──────────────────────────────────┬──────────────────────────────────┘
                                   │
┌──────────────────────────────────▼──────────────────────────────────┐
│                   DAO 抽象层  (core/*.go)                            │
│                                                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────────┐ │
│  │ Hook 系统    │  │ 事务 TxAppInfo│  │ 锁重试 baseLockRetry       │ │
│  │ (生命周期)    │  │ (延迟回调)    │  │ (12次/总7.35s等待)        │ │
│  └──────────────┘  └──────────────┘  └────────────────────────────┘ │
│                                                                       │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │              dualDBBuilder（读写路由）                           │  │
│  │   SELECT/WITH → concurrentDB(120)  其他 → nonconcurrentDB(1)   │  │
│  └───────────────────────────────┬────────────────────────────────┘  │
└──────────────────────────────────┼───────────────────────────────────┘
                                   │
┌──────────────────────────────────▼──────────────────────────────────┐
│                 SQLite 嵌入层 (dbx v1.12 + modernc v1.52)           │
│                                                                       │
│  ┌─────────────────────┐     ┌─────────────────────┐                 │
│  │  Concurrent Pool    │     │ Nonconcurrent Pool  │                 │
│  │  data.db: 120 连接   │     │  data.db: 1 连接    │                 │
│  │  auxiliary.db: 20 连接 │     │  auxiliary.db: 1 连接 │                 │
│  └──────────┬──────────┘     └──────────┬──────────┘                 │
│             │                           │                            │
│             └─────────────┬─────────────┘                            │
│                           │                                          │
│              ┌────────────▼────────────┐                             │
│              │  modernc.org/sqlite      │                             │
│              │  PRAGMA 配置:             │                             │
│              │  · busy_timeout=10s      │                             │
│              │  · journal_mode=WAL      │                             │
│              │  · synchronous=NORMAL    │                             │
│              │  · cache_size=-32000(32MB)│                             │
│              └────────────┬────────────┘                             │
└───────────────────────────┼──────────────────────────────────────────┘
                            │
                   ┌────────▼────────┐
                   │  data.db /       │
                   │  auxiliary.db    │
                   └─────────────────┘
```

---

## 八、关键设计决策与权衡汇总

| 决策 | 收益 | 代价 / 注意事项 |
|------|------|----------------|
| **双池分离（读 120 / 写 1）** | 读操作完全并发；写操作在 Go 层面排队，减少 SQLite 锁竞争 | 写入吞吐量受限于单连接；写池连接成为全局瓶颈 |
| **WAL + synchronous=NORMAL** | 读写不互斥；普通提交不做 `fsync`，性能好 | 极端崩溃（断电/OS 崩溃）下可能丢失最近已提交事务 |
| **busy_timeout + 应用层重试** | 双层防护，最大程度避免 `SQLITE_BUSY` | 应用层通过错误字符串匹配锁错误，不够"优雅"但跨驱动兼容 |
| **事务中克隆 App** | 事务状态（DB 引用、TxAppInfo）与父 App 隔离 | 浅拷贝共享 Hook、Store 等引用；`event.App` 在延迟回调中需手动恢复为 `parent` |
| **延迟 Hook（OnComplete）** | 邮件、文件删除、实时消息等副作用仅在事务提交后执行 | `OnComplete` 回调返回的 error 会被 Join 到最终结果，但不会回滚已提交事务 |
| **错误文本匹配检测锁** | 兼容 mattn/go-sqlite3 和 modernc.org/sqlite 两种驱动 | 依赖驱动错误消息不变；未来驱动更新文本可能导致检测失效 |
| **cache_size 负值（-32000）** | 以 KB 为单位声明目标缓存大小，比直接写页数更直观 | SQLite 最终按页数保存该设置；如果后续改变 `page_size`，实际内存量会随换算结果变化 |
| **嵌套事务不使用 Savepoint** | 实现简单；OnComplete 回调统一注册 | 无法单独回滚内层嵌套事务；内层失败直接影响外层整体 |
