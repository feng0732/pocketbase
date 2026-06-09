# PocketBase SQLite 嵌入与 DAO 抽象边界分析

本文档从代码实现角度梳理 PocketBase 中 SQLite 数据库嵌入层与 DAO（数据访问对象）抽象层之间的边界设计，重点覆盖连接池、事务封装、读写拆分和并发写入四大核心机制。

---

## 一、整体架构分层

PocketBase 的数据访问层采用**两层架构**：

| 层次 | 职责 | 核心文件 |
|------|------|----------|
| **SQLite 嵌入层** | 驱动加载、连接池管理、WAL 模式配置、SQL 执行 | [db_connect.go](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/db_connect.go)、[base.go](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/base.go) |
| **DAO 抽象层** | 模型 CRUD、事务管理、Hook 生命周期、读写路由 | [db.go](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/db.go)、[db_tx.go](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/db_tx.go)、[db_builder.go](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/db_builder.go)、[db_retry.go](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/db_retry.go) |

两层通过 `dbx.Builder` 接口解耦：嵌入层提供 `*dbx.DB` 和 `*dbx.Tx` 实例，抽象层面向接口编程。

```
用户代码 → App.Save()/Delete()/ModelQuery()
               ↓
         DAO 抽象层（Hook、事务、重试、读写路由）
               ↓
         dbx.Builder 接口（dualDBBuilder / *dbx.Tx）
               ↓
    SQLite 嵌入层（连接池、WAL、modernc.org/sqlite）
```

---

## 二、连接池设计：双池分离

### 2.1 为什么需要双连接池

SQLite 的写锁是数据库级别的（即使在 WAL 模式下，同一时刻只允许一个写事务）。PocketBase 采用**双池策略**来最小化 `SQLITE_BUSY` 错误：

- **Concurrent Pool（并发池）**：多连接，用于读操作（SELECT）
- **Nonconcurrent Pool（非并发池）**：单连接，用于写操作（INSERT/UPDATE/DELETE/DDL）

### 2.2 连接池初始化

在 [base.go#L1175-L1209](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/base.go#L1175-L1209) 的 `initDataDB()` 方法中完成初始化：

```go
func (app *BaseApp) initDataDB() error {
    dbPath := filepath.Join(app.DataDir(), "data.db")

    // 并发读池：默认 120 个活跃连接，15 个空闲连接
    concurrentDB, _ := app.config.DBConnect(dbPath)
    concurrentDB.DB().SetMaxOpenConns(app.config.DataMaxOpenConns) // 默认 120
    concurrentDB.DB().SetMaxIdleConns(app.config.DataMaxIdleConns) // 默认 15
    concurrentDB.DB().SetConnMaxIdleTime(3 * time.Minute)

    // 非并发写池：强制 1 个连接，串行化所有写入
    nonconcurrentDB, _ := app.config.DBConnect(dbPath)
    nonconcurrentDB.DB().SetMaxOpenConns(1)  // 关键：仅 1 连接
    nonconcurrentDB.DB().SetMaxIdleConns(1)
    nonconcurrentDB.DB().SetConnMaxIdleTime(3 * time.Minute)

    app.concurrentDB = concurrentDB
    app.nonconcurrentDB = nonconcurrentDB
    return nil
}
```

默认参数定义在 [base.go#L32-L37](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/base.go#L32-L37)：

| 参数 | data.db 默认值 | auxiliary.db 默认值 |
|------|---------------|---------------------|
| `MaxOpenConns`（并发池） | 120 | 20 |
| `MaxIdleConns`（并发池） | 15 | 3 |
| `MaxOpenConns`（非并发池） | 1 | 1 |
| `MaxIdleConns`（非并发池） | 1 | 1 |

### 2.3 SQLite 驱动与 PRAGMA 配置

连接建立由 [db_connect.go](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/db_connect.go) 中的 `DefaultDBConnect` 负责：

```go
func DefaultDBConnect(dbPath string) (*dbx.DB, error) {
    pragmas := "?_pragma=busy_timeout(10000)" +
        "&_pragma=journal_mode(WAL)" +
        "&_pragma=journal_size_limit(200000000)" +
        "&_pragma=synchronous(NORMAL)" +
        "&_pragma=foreign_keys(ON)" +
        "&_pragma=temp_store(MEMORY)" +
        "&_pragma=cache_size(-32000)"

    return dbx.Open("sqlite", dbPath+pragmas)
}
```

关键 PRAGMA 说明：

| PRAGMA | 值 | 作用 |
|--------|-----|------|
| `busy_timeout` | 10000ms | SQLite 层面的锁等待超时，配合应用层重试 |
| `journal_mode` | WAL | Write-Ahead Logging，读写不互斥 |
| `synchronous` | NORMAL | WAL 模式下的推荐设置，平衡性能与安全 |
| `journal_size_limit` | 200MB | 限制 WAL 文件大小，防止无限增长 |
| `foreign_keys` | ON | 启用外键约束 |
| `temp_store` | MEMORY | 临时表放内存 |
| `cache_size` | -32000 | 页缓存大小（约 256MB，负数表示 KB 数） |

> **注意**：`busy_timeout` 必须第一个设置，因为在启用 WAL 前就需要让连接具备阻塞等待能力。

### 2.4 双数据库实例

PocketBase 维护两个物理数据库文件，各自拥有独立的双连接池：

- **data.db**：主业务数据（collections、records、settings 等）
- **auxiliary.db**：辅助数据（logs、审计记录等，"aux" 因 Windows 保留字而改名）

对应访问方法：`DB()`/`ConcurrentDB()`/`NonconcurrentDB()` vs `AuxDB()`/`AuxConcurrentDB()`/`AuxNonconcurrentDB()`。

---

## 三、读写拆分：dualDBBuilder 路由层

### 3.1 自动路由机制

用户通常调用 `app.DB()` 获取 `dbx.Builder`，该方法返回 `dualDBBuilder` 实例（除非在事务中）。见 [base.go#L490-L500](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/base.go#L490-L500)：

```go
func (app *BaseApp) DB() dbx.Builder {
    // 事务中：concurrentDB == nonconcurrentDB == *dbx.Tx
    if app.concurrentDB == app.nonconcurrentDB {
        return app.concurrentDB
    }
    return &dualDBBuilder{
        concurrentDB:    app.concurrentDB,
        nonconcurrentDB: app.nonconcurrentDB,
    }
}
```

### 3.2 dualDBBuilder 的方法路由表

[db_builder.go](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/db_builder.go) 实现了 `dbx.Builder` 接口，将每个方法路由到不同的池：

| 方法 | 路由目标 | 说明 |
|------|---------|------|
| `Select()` | concurrentDB | 读操作 |
| `Model()` | nonconcurrentDB | 模型级写操作（Insert/Update） |
| `Insert()` | nonconcurrentDB | 写 |
| `Upsert()` | nonconcurrentDB | 写 |
| `Update()` | nonconcurrentDB | 写 |
| `Delete()` | nonconcurrentDB | 写 |
| `CreateTable()` 等 DDL | nonconcurrentDB | 写 |
| `Quote()` / `QueryBuilder()` 等 | concurrentDB | 纯计算，不涉及 IO |
| `NewQuery(sql)` | **智能判断** | 见下文 |

### 3.3 NewQuery 的 SQL 前缀检测

对于原始 SQL 查询，`dualDBBuilder` 通过检测 SQL 前缀决定路由，见 [db_builder.go#L151-L160](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/db_builder.go#L151-L160)：

```go
func (b *dualDBBuilder) NewQuery(str string) *dbx.Query {
    trimmed := trimLeftSpaces(str)
    // SELECT 或 WITH（CTE）→ 读池
    if hasPrefixFold(trimmed, "SELECT") || hasPrefixFold(trimmed, "WITH") {
        return b.concurrentDB.NewQuery(str)
    }
    // 其他 → 写池（INSERT/UPDATE/DELETE/PRAGMA 等）
    return b.nonconcurrentDB.NewQuery(str)
}
```

> **设计取舍**：不处理 `INSERT ... RETURNING`、`WITH ... INSERT` 等复杂 CTE 场景，因为这类场景较少见，避免过度复杂化。

### 3.4 DAO 层的显式池选择

在 DAO 层（[db.go](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/db.go)）中，各类操作显式选择池：

| 操作 | 使用的池 | 代码位置 |
|------|---------|---------|
| `ModelQuery()`（模型查询） | ConcurrentDB | [db.go#L67-L69](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/db.go#L67-L69) |
| `Save()`/create（写入） | NonconcurrentDB | [db.go#L290-L296](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/db.go#L290-L296) |
| `Save()`/update（更新） | NonconcurrentDB | [db.go#L385-L391](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/db.go#L385-L391) |
| `Delete()`（删除） | NonconcurrentDB | [db.go#L124-L130](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/db.go#L124-L130) |
| `validateRecordId()`（存在性检查） | ConcurrentDB | [db.go#L487-L491](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/db.go#L487-L491) |
| `HasTable()` / `TableInfo()` 等元数据 | ConcurrentDB | [db_table.go#L101-L107](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/db_table.go#L101-L107) |
| `DeleteTable()` / `Vacuum()` 等 DDL | NonconcurrentDB | [db_table.go#L90-L95](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/db_table.go#L90-L95) |

---

## 四、事务封装：App 克隆 + TxAppInfo

### 4.1 事务 API 入口

PocketBase 提供两个事务方法，见 [db_tx.go#L11-L23](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/db_tx.go#L11-L23)：

```go
func (app *BaseApp) RunInTransaction(fn func(txApp App) error) error {
    return app.runInTransaction(app.NonconcurrentDB(), fn, false)
}

func (app *BaseApp) AuxRunInTransaction(fn func(txApp App) error) error {
    return app.runInTransaction(app.AuxNonconcurrentDB(), fn, true)
}
```

**关键点**：事务总是从 `NonconcurrentDB`（单连接池）启动，确保同一事务内的读写操作使用同一连接。

### 4.2 嵌套事务处理

`runInTransaction` 支持安全嵌套，见 [db_tx.go#L25-L49](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/db_tx.go#L25-L49)：

```go
func (app *BaseApp) runInTransaction(db dbx.Builder, fn func(txApp App) error, isForAuxDB bool) error {
    switch txOrDB := db.(type) {
    case *dbx.Tx:
        // 已在事务中 → 直接复用，不开启新事务（Savepoint 机制由 dbx 底层处理）
        return fn(app)
    case *dbx.DB:
        var txApp *BaseApp
        txErr := txOrDB.Transactional(func(tx *dbx.Tx) error {
            // 创建事务专属 App 副本
            txApp = app.createTxApp(tx, isForAuxDB)
            return fn(txApp)
        })
        // 事务结束后执行所有 OnComplete 回调
        if txApp != nil && txApp.txInfo != nil {
            afterFuncErr := txApp.txInfo.runAfterFuncs(txErr)
            // ...
        }
        return txErr
    }
}
```

### 4.3 事务 App 的克隆机制

事务通过**浅拷贝 BaseApp** 来隔离状态，见 [db_tx.go#L51-L69](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/db_tx.go#L51-L69)：

```go
func (app *BaseApp) createTxApp(tx *dbx.Tx, isForAuxDB bool) *BaseApp {
    clone := *app  // 浅拷贝：共享 Hook、Store、Logger 等

    // 关键：将 concurrentDB 和 nonconcurrentDB 都指向同一个 *dbx.Tx
    if isForAuxDB {
        clone.auxConcurrentDB = tx
        clone.auxNonconcurrentDB = tx
    } else {
        clone.concurrentDB = tx
        clone.nonconcurrentDB = tx
    }

    // 每个事务拥有独立的 TxAppInfo
    clone.txInfo = &TxAppInfo{
        parent:     app,
        isForAuxDB: isForAuxDB,
    }

    return &clone
}
```

事务中 `app.DB()` 的判断逻辑：
```go
// 事务中 concurrentDB == nonconcurrentDB（都是同一个 *dbx.Tx）
// 因此 DB() 直接返回该 *dbx.Tx，跳过 dualDBBuilder
if app.concurrentDB == app.nonconcurrentDB {
    return app.concurrentDB
}
```

### 4.4 TxAppInfo：事务完成回调

`TxAppInfo` 用于注册事务完成后的回调（延迟 Hook 执行），见 [db_tx.go#L71-L112](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/db_tx.go#L71-L112)：

```go
type TxAppInfo struct {
    parent     *BaseApp
    afterFuncs []func(txErr error) error
    mu         sync.Mutex     // 并发安全
    isForAuxDB bool
}

func (a *TxAppInfo) OnComplete(fn func(txErr error) error) {
    a.mu.Lock()
    defer a.mu.Unlock()
    a.afterFuncs = append(a.afterFuncs, fn)
}
```

### 4.5 Hook 延迟执行机制

事务中的 `After*Success` / `After*Error` Hook 不会立即执行，而是通过 `txInfo.OnComplete()` 延迟到事务提交后。以 `create` 操作为例，见 [db.go#L343-L363](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/db.go#L343-L363)：

```go
if app.txInfo != nil {
    // 在事务中 → 延迟执行
    app.txInfo.OnComplete(func(txErr error) error {
        if txErr != nil {
            // 事务回滚 → 触发 Error Hook
            return app.OnModelAfterCreateError().Trigger(...)
        }
        // 事务提交 → 触发 Success Hook
        return app.OnModelAfterCreateSuccess().Trigger(event)
    })
} else {
    // 不在事务中 → 立即执行
    if err := event.App.OnModelAfterCreateSuccess().Trigger(event); err != nil {
        return err
    }
}
```

这种设计确保 Hook 中执行的副作用（如发送邮件、删除文件）只会在数据真正持久化后发生。

---

## 五、并发写入：锁重试 + 单连接串行化

### 5.1 三层防护机制

PocketBase 通过三层机制应对 SQLite 的并发写入限制：

| 层级 | 机制 | 位置 |
|------|------|------|
| L1 | 单连接写池（串行化所有写请求） | `nonconcurrentDB.SetMaxOpenConns(1)` |
| L2 | SQLite busy_timeout（驱动层等待） | `_pragma=busy_timeout(10000)` |
| L3 | 应用层指数退避重试 | `baseLockRetry` / `execLockRetry` |

### 5.2 应用层重试逻辑

重试核心实现在 [db_retry.go](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/db_retry.go)：

```go
const defaultMaxLockRetries = 12

var defaultRetryIntervals = []int{50, 100, 150, 200, 300, 400, 500, 700, 1000} // 毫秒

func baseLockRetry(op func(attempt int) error, maxRetries int) error {
    attempt := 1
Retry:
    err := op(attempt)
    if err != nil && attempt <= maxRetries {
        errStr := err.Error()
        // 检测 "database is locked" 或 "table is locked"
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

重试间隔采用递增策略（从 50ms 逐步增长到 1000ms），最多重试 12 次，总计最大等待约 5.35 秒。

### 5.3 两种重试入口

#### (1) execLockRetry：用于查询执行 Hook

`execLockRetry` 包装查询执行，自动附加超时 Context，见 [db_retry.go#L20-L41](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/db_retry.go#L20-L41)：

```go
func execLockRetry(timeout time.Duration, maxRetries int) dbx.ExecHookFunc {
    return func(q *dbx.Query, op func() error) error {
        if q.Context() == nil {
            cancelCtx, cancel := context.WithTimeout(context.Background(), timeout)
            defer cancel()
            q.WithContext(cancelCtx)
        }
        execErr := baseLockRetry(func(attempt int) error {
            return op()
        }, maxRetries)
        // ... 错误附加 SQL 信息
        return execErr
    }
}
```

在 `ModelQuery()` 中自动注入，见 [db.go#L83-L85](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/db.go#L83-L85)：

```go
func (app *BaseApp) modelQuery(db dbx.Builder, m Model) *dbx.SelectQuery {
    return db.Select(...).From(...).
        WithBuildHook(func(query *dbx.Query) {
            query.WithExecHook(execLockRetry(app.config.QueryTimeout, defaultMaxLockRetries))
        })
}
```

#### (2) baseLockRetry：用于 DAO 层写入

所有写入操作（create/update/delete）都显式调用 `baseLockRetry`，以 `Delete` 为例，见 [db.go#L132-L138](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/db.go#L132-L138)：

```go
return baseLockRetry(func(attempt int) error {
    _, err := db.Delete(e.Model.TableName(), dbx.HashExp{
        idColumn: pk,
    }).WithContext(e.Context).Execute()
    return err
}, defaultMaxLockRetries)
```

### 5.4 定期 WAL Checkpoint

后台 Cron 任务定期执行 WAL checkpoint，防止 WAL 文件无限膨胀，见 [base.go#L1360-L1375](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/base.go#L1360-L1375)：

```go
app.Cron().Add("__pbDBOptimize__", "0 0 * * *", func() {
    // TRUNCATE：将 WAL 内容合并到主数据库后截断 WAL
    app.NonconcurrentDB().NewQuery("PRAGMA wal_checkpoint(TRUNCATE)").Execute()
    app.AuxNonconcurrentDB().NewQuery("PRAGMA wal_checkpoint(TRUNCATE)").Execute()
    // ANALYZE：更新查询规划器统计信息
    app.NonconcurrentDB().NewQuery("PRAGMA optimize").Execute()
})
```

该 Cron 每天午夜（`0 0 * * *`）执行一次。

---

## 六、DAO 层 CRUD 完整流程

### 6.1 Save 操作流程

```
app.Save(model)
  └─ app.save(ctx, model, withValidations, isForAuxDB)
      ├─ model.IsNew() ? ── create ──┐
      └─ !model.IsNew() ? ── update ──┤
                                      ↓
                      ┌─ OnModelCreate/Update Hook
                      │     ├─ 验证（ValidateWithContext）
                      │     └─ OnModelCreate/UpdateExecute Hook
                      │           └─ baseLockRetry {
                      │                 ├─ 选择 NonconcurrentDB
                      │                 ├─ DBExporter 或 db.Model().Insert/Update()
                      │                 └─ 重试最多 12 次
                      │              }
                      ├─ 失败 → 立即触发 OnModelAfter*Error Hook
                      └─ 成功
                           ├─ 在事务中？
                           │    └─ txInfo.OnComplete(...) 延迟 Hook
                           └─ 不在事务中？
                                └─ 立即触发 OnModelAfter*Success Hook
```

### 6.2 Model 接口契约

DAO 层操作的所有实体必须实现 [Model](file:///d:/fz/0601/solo-dogfeeding/code/150-pocketbase/core/db_model.go#L6-L13) 接口：

```go
type Model interface {
    TableName() string          // 映射到数据库表名
    PK() any                    // 当前主键值
    LastSavedPK() any           // 上次持久化的主键值（用于 UPDATE WHERE）
    IsNew() bool                // 判断 INSERT vs UPDATE
    MarkAsNew()                 // 标记为新对象
    MarkAsNotNew()              // 标记为已持久化
}
```

`BaseModel` 提供了默认实现，使用 `lastSavedPK` 字段追踪持久化状态。

---

## 七、架构边界总结图

```
┌─────────────────────────────────────────────────────────────────────┐
│                          用户代码 / API 层                           │
│  app.Save()  app.Delete()  app.ModelQuery()  app.RunInTransaction() │
└──────────────────────────────────┬──────────────────────────────────┘
                                   │
┌──────────────────────────────────▼──────────────────────────────────┐
│                        DAO 抽象层 (core/*.go)                        │
│                                                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────────┐ │
│  │  Hook 系统   │  │  事务封装    │  │  重试 & 超时               │ │
│  │  (生命周期)  │  │  (TxAppInfo) │  │  (baseLockRetry)           │ │
│  └──────────────┘  └──────────────┘  └────────────────────────────┘ │
│                                                                       │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                  dualDBBuilder（读写路由）                      │  │
│  │   SELECT/WITH ──► concurrentDB    INSERT/UPDATE/DELETE ──► ... │  │
│  └───────────────────────────────┬────────────────────────────────┘  │
└──────────────────────────────────┼───────────────────────────────────┘
                                   │
┌──────────────────────────────────▼──────────────────────────────────┐
│                     SQLite 嵌入层 (dbx + modernc)                    │
│                                                                       │
│  ┌─────────────────────┐    ┌─────────────────────┐                  │
│  │  Concurrent Pool    │    │ Nonconcurrent Pool  │                  │
│  │  (data.db: 120 con) │    │  (data.db: 1 conn)  │                  │
│  └──────────┬──────────┘    └──────────┬──────────┘                  │
│             │                          │                             │
│             └──────────────┬───────────┘                             │
│                            │                                         │
│                  ┌─────────▼──────────┐                              │
│                  │  modernc.org/sqlite │                              │
│                  │  (WAL + busy_timeout)│                              │
│                  └─────────┬──────────┘                              │
└────────────────────────────┼─────────────────────────────────────────┘
                             │
                    ┌────────▼────────┐
                    │  data.db 文件    │
                    │  (SQLite 引擎)   │
                    └─────────────────┘
```

---

## 八、关键设计决策与权衡

| 决策 | 优点 | 代价 |
|------|------|------|
| **双池分离（120读/1写）** | 读操作完全并发，写操作避免锁竞争 | 写操作吞吐量受限于单连接 |
| **WAL + NORMAL 同步** | 读写不互斥，性能好 | 极端崩溃下可能丢失最近事务 |
| **应用层重试 + 错误文本匹配** | 不依赖驱动特定错误码，兼容性好 | 依赖字符串匹配，可能误判 |
| **事务中克隆 App** | 事务内状态隔离（DB 引用、OnComplete 回调） | 浅拷贝共享 Hook/Store，需使用者注意 |
| **延迟 Hook（OnComplete）** | 副作用（邮件、文件删除）仅在提交后执行 | Hook 执行失败不影响事务结果 |
| **错误字符串匹配锁检测** | 跨驱动兼容（mattn/go-sqlite3 vs modernc） | 不够"优雅"，依赖驱动错误消息不变 |
