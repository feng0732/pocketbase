# PocketBase 事件钩子总线深度解析

## 一、核心数据结构

### 1.1 基础接口与类型

整个钩子系统围绕以下核心类型构建，全部位于 `tools/hook` 包下：

| 类型 | 位置 | 作用 |
|------|------|------|
| `Resolver` 接口 | [event.go#L4-L11](file:///d:/fz/0601/solo-dogfeeding/code/159-pocketbase/tools/hook/event.go#L4-L11) | 所有事件必须实现的接口，提供 `Next()` 链式调用能力 |
| `Event` 结构体 | [event.go#L25-L45](file:///d:/fz/0601/solo-dogfeeding/code/159-pocketbase/tools/hook/event.go#L25-L45) | 基础事件结构体，自定义事件需嵌入它 |
| `Handler[T]` | [hook.go#L13-L32](file:///d:/fz/0601/solo-dogfeeding/code/159-pocketbase/tools/hook/hook.go#L13-L32) | 单个钩子处理器，包含执行函数、ID、优先级 |
| `Hook[T]` | [hook.go#L54-L57](file:///d:/fz/0601/solo-dogfeeding/code/159-pocketbase/tools/hook/hook.go#L54-L57) | 钩子队列管理器，线程安全地管理一组 Handler |
| `Tagger` 接口 | [tagged.go#L9-L13](file:///d:/fz/0601/solo-dogfeeding/code/159-pocketbase/tools/hook/tagged.go#L9-L13) | 支持按标签过滤的事件需实现此接口 |
| `TaggedHook[T]` | [tagged.go#L30-L34](file:///d:/fz/0601/solo-dogfeeding/code/159-pocketbase/tools/hook/tagged.go#L30-L34) | 带标签过滤的 Hook 代理 |

### 1.2 Handler 处理器结构

```go
type Handler[T Resolver] struct {
    Func     func(T) error   // 实际执行的处理函数，通常需要调用 e.Next() 推进链条
    Id       string          // 唯一标识，用于后续移除或替换
    Priority int             // 优先级，数值越小越先执行
}
```

**关键说明**：
- `Func` 内部如果想要后续 Handler 继续执行，需要显式调用 `e.Next()`
- 如果不调用 `e.Next()` 就 return（无论返回 nil 还是 error），前进方向的链条在此处终止
- 故意不调用 `e.Next()` 是合法行为（例如中间件拦截未授权的请求）

---

## 二、钩子注册机制

### 2.1 注册入口

`Hook[T]` 提供两种注册方式，参见 [hook.go#L59-L113](file:///d:/fz/0601/solo-dogfeeding/code/159-pocketbase/tools/hook/hook.go#L59-L113)：

| 方法 | 说明 |
|------|------|
| `Bind(handler *Handler[T])` | 完整注册，可指定 Id 和 Priority |
| `BindFunc(fn func(e T) error)` | 简化注册，自动生成 Id，Priority=0 |

### 2.2 注册逻辑详解

**Bind 方法执行流程**：

```
1. 加写锁 (sync.RWMutex)
2. 若 handler.Id 为空 → 自动生成 20 字符随机 ID
3. 若 handler.Id 已存在 → 替换原有 Handler（同 ID 不允许重复）
4. 若 handler.Id 不存在 → 追加到 handlers 切片
5. 按 Priority 稳定排序 (sort.SliceStable)
6. 返回 handler.Id
```

**关于排序**：使用 `sort.SliceStable` 按 Priority 升序排列。Priority 相同的 Handler 保持注册时的原始顺序（先注册先执行）。

### 2.3 注册示例

```go
h := &hook.Hook[*MyEvent]{}

// 方式一：简化注册
h.BindFunc(func(e *MyEvent) error {
    // Priority=0，ID 自动生成
    return e.Next()
})

// 方式二：完整注册
h.Bind(&hook.Handler[*MyEvent]{
    Id:       "my-custom-hook",
    Priority: -100,       // 更小的数 → 更早执行
    Func: func(e *MyEvent) error {
        return e.Next()
    },
})
```

### 2.4 钩子的移除

| 方法 | 位置 | 说明 |
|------|------|------|
| `Unbind(idsToRemove ...string)` | [hook.go#L116-L128](file:///d:/fz/0601/solo-dogfeeding/code/159-pocketbase/tools/hook/hook.go#L116-L128) | 按 ID 移除一个或多个 Handler |
| `UnbindAll()` | [hook.go#L131-L136](file:///d:/fz/0601/solo-dogfeeding/code/159-pocketbase/tools/hook/hook.go#L131-L136) | 清空所有已注册 Handler |

---

## 三、触发顺序与分派逻辑（核心）

### 3.1 Trigger 方法 —— 洋葱模型的构建

`Trigger` 是整个钩子系统的核心，参见 [hook.go#L146-L174](file:///d:/fz/0601/solo-dogfeeding/code/159-pocketbase/tools/hook/hook.go#L146-L174)。

**方法签名**：
```go
func (h *Hook[T]) Trigger(event T, oneOffHandlerFuncs ...func(T) error) error
```

**参数说明**：
- `event`：事件对象，必须实现 `Resolver` 接口（即嵌入 `hook.Event`）
- `oneOffHandlerFuncs`：可选的一次性处理函数，仅在本次触发时生效，追加到链尾

### 3.2 分派算法详解

这是最精妙的部分。钩子链采用 **洋葱模型（Onion Model / 中间件模式）**，通过 **倒序构建闭包链** 实现：

```go
// 1. 收集所有 handler 函数（已按 Priority 升序排列）
//    handlers = [h1, h2, h3, ...oneOffFuncs]

// 2. 从后往前倒序遍历，构建嵌套闭包
event.setNextFunc(nil)  // 初始 next = nil

for i := len(handlers) - 1; i >= 0; i-- {
    old := event.nextFunc()           // 保存当前 next
    event.setNextFunc(func() error {  // 新的 next = 包装当前 handler
        event.setNextFunc(old)        // 执行前恢复 "外层的 next"
        return handlers[i](event)     // 执行当前 handler
    })
}

// 3. 启动链条
return event.Next()
```

### 3.3 执行顺序可视化

假设有 3 个 Handler：`[A, B, C]`（按 Priority 排序），再加上 1 个 oneOff 函数 `D`：

```
注册/收集顺序:  A → B → C → D
倒序构建闭包:  从 D 开始，然后 C、B、A

最终形成的调用链:
  Next()
    └─> A(e)
          ├─ e.Next() 之前的代码（A 的前半部分）
          ├─ e.Next() ──> B(e)
          │                  ├─ e.Next() 之前的代码（B 的前半部分）
          │                  ├─ e.Next() ──> C(e)
          │                  │                  ├─ e.Next() 之前的代码（C 的前半部分）
          │                  │                  ├─ e.Next() ──> D(e)
          │                  │                  │                  ├─ 执行 D
          │                  │                  │                  └─ D 返回
          │                  │                  └─ e.Next() 之后的代码（C 的后半部分）
          │                  └─ e.Next() 之后的代码（B 的后半部分）
          └─ e.Next() 之后的代码（A 的后半部分）
```

**结论**：
- **e.Next() 之前**的代码（前进方向）按 Priority 升序执行（先注册/优先级高的先执行）
- **e.Next() 之后**的代码（返回方向）按 Priority 降序执行（先注册/优先级高的后执行）
- 某个 Handler 若不调用 `e.Next()` 就 return → **前进方向终止**，后续更内层的 Handler 不会被执行，但当前 Handler 及外层 Handler 的后半部分代码仍会沿返回方向继续执行
- 详细错误行为参见第五章"错误传播机制"

### 3.4 oneOff 处理函数的作用

`oneOffHandlerFuncs` 追加在常规 handlers 之后，通常用作：
- **最终业务逻辑**：例如数据库写入操作（见 [db.go#L117-L139](file:///d:/fz/0601/solo-dogfeeding/code/159-pocketbase/core/db.go#L117-L139)）
- **链式终结器**：确保链条最终落到实际业务上

---

## 四、标签（Tag）过滤机制

### 4.1 TaggedHook 的作用

`TaggedHook` 是 `Hook` 的代理层，允许按标签过滤触发。参见 [tagged.go#L28-L84](file:///d:/fz/0601/solo-dogfeeding/code/159-pocketbase/tools/hook/tagged.go#L28-L84)。

**典型应用场景**：
- `app.OnRecordCreate("posts")` —— 只监听 `posts` 集合的记录创建事件
- `app.OnModelDelete("users")` —— 只监听 `users` 表的模型删除事件

### 4.2 CanTriggerOn 判断逻辑

参见 [tagged.go#L40-L52](file:///d:/fz/0601/solo-dogfeeding/code/159-pocketbase/tools/hook/tagged.go#L40-L52)：

```go
func (h *TaggedHook[T]) CanTriggerOn(tagsToCheck []string) bool {
    if len(h.tags) == 0 {
        return true  // 无标签 → 匹配所有事件
    }
    // 有任一标签相交 → 触发
    for _, t := range tagsToCheck {
        if list.ExistInSlice(t, h.tags) {
            return true
        }
    }
    return false
}
```

### 4.3 Bind 的包装逻辑

`TaggedHook.Bind` 不直接注册用户函数，而是 **包装一层判断逻辑**：

```go
func (h *TaggedHook[T]) Bind(handler *Handler[T]) string {
    fn := handler.Func
    handler.Func = func(e T) error {
        if h.CanTriggerOn(e.Tags()) {
            return fn(e)           // 标签匹配 → 执行用户函数
        }
        return e.Next()            // 标签不匹配 → 跳过，继续链
    }
    return h.mainHook.Bind(handler)
}
```

**注意**：即使标签不匹配，也必须调用 `e.Next()` 以保证链条继续推进。

### 4.4 事件的 Tags 来源

不同事件类型的 `Tags()` 实现不同，参见 [events.go#L29-L80](file:///d:/fz/0601/solo-dogfeeding/code/159-pocketbase/core/events.go#L29-L80)：

| 事件类型 | Tags 返回值 |
|----------|-------------|
| `baseModelEventData` | 模型的 `TableName()`，或自定义 `HookTags()` |
| `baseRecordEventData` | `Record.HookTags()`（通常是集合 ID 和名称） |
| `baseCollectionEventData` | 集合的 `Id` 和 `Name` |

---

## 五、错误传播机制（修正版）

之前的说法"返回 error 立即终止链条"是不准确的。错误行为取决于 **error 是在 e.Next() 之前还是之后返回**，这和洋葱模型的双路径密切相关。

### 5.1 先理解：链条的两个方向

钩子链是 **洋葱模型**，存在两条路径：

```
前进方向（Next 之前）:  Handler A → Handler B → Handler C → oneOff
返回方向（Next 之后）:  Handler A ← Handler B ← Handler C ← oneOff
```

- **前进方向**：由 `e.Next()` 的调用驱动，从外到内
- **返回方向**：由函数 return 驱动，从内到外

**只有前进方向可以被"中断"**——只要某个 Handler 不调用 `e.Next()`，后续（更内层的）Handler 就不会被执行。而返回方向上的 Handler 已经在调用栈上了，它们的后半部分代码一定会执行（除非 panic）。

### 5.2 场景对比：Next 前返回 vs Next 后返回

假设有 3 个 Handler 按优先级排序为 `[A, B, C]`，我们分别在 B 的不同位置返回错误，看行为差异。

#### 场景 A：在 Next() 之前返回错误（前进方向中断）

```go
// Handler B
Func: func(e *Event) error {
    return errors.New("fail")  // 不调用 e.Next()，直接返回
}
```

执行路径：

```
A 前半 → e.Next()
  └─> B 直接返回 error（从未调用 e.Next()）
        └─> ❌ C 根本不会被执行
              ❌ oneOff 根本不会被执行
A 后半（return e.Next() → 收到 error）
```

**结论**：前进方向被阻断，B 之后的所有 Handler（C、oneOff）都不会执行。错误直接成为 `e.Next()` 的返回值向上冒泡。

#### 场景 B：在 Next() 之后返回错误（仅返回方向冒泡）

```go
// Handler B
Func: func(e *Event) error {
    e.Next()                    // 先调用 Next 推进链条
    return errors.New("fail")   // 然后返回错误
}
```

执行路径：

```
A 前半 → e.Next()
  └─> B 前半 → e.Next()
        └─> C 前半 → e.Next()
              └─> oneOff 执行完毕 ← return nil
        C 后半 ← return nil
  B 后半 ← return errors.New("fail")   // C 执行完了，B 才返回 error
A 后半 ← e.Next() 收到 error，向上冒泡
```

**结论**：
- **前进方向不受影响**：C 和 oneOff 都完整执行了（因为 B 在返回 error 之前已经调用了 `e.Next()`）
- **错误沿返回方向冒泡**：A 可以通过 `e.Next()` 的返回值捕获到这个 error
- 测试证据：[hook_test.go#L32](file:///d:/fz/0601/solo-dogfeeding/code/159-pocketbase/tools/hook/hook_test.go#L32) 的注释明确写着 `// error shouldn't stop the chain`，测试中 handler7 在 Next 后返回 error，但 handler8、handler9 仍然被执行。

### 5.3 Handler 对错误的三种处理方式

外层 Handler 可以通过 `e.Next()` 的返回值拿到内层错误，有三种处理策略：

```go
// 策略 1：直接透传（最常见）
Func: func(e *Event) error {
    // 前半逻辑
    return e.Next()  // 内层的 error 直接向上传
}

// 策略 2：捕获并处理错误
Func: func(e *Event) error {
    // 前半逻辑
    err := e.Next()
    if err != nil {
        log.Println("捕获到错误:", err)
        return nil    // 吞掉错误，外层不会知道
    }
    // 后半逻辑
    return nil
}

// 策略 3：包装错误后继续冒泡
Func: func(e *Event) error {
    // 前半逻辑
    err := e.Next()
    if err != nil {
        return fmt.Errorf("handler A 包裹: %w", err)
    }
    return nil
}
```

### 5.4 Event.Next() 的源码验证

参见 [event.go#L30-L35](file:///d:/fz/0601/solo-dogfeeding/code/159-pocketbase/tools/hook/event.go#L30-L35)：

```go
func (e *Event) Next() error {
    if e.next != nil {
        return e.next()  // 把内层 handler 的返回值直接交给外层
    }
    return nil
}
```

`Next()` 只是简单地把内层闭包的返回值透传出来。内层返回什么，外层的 `e.Next()` 就收到什么——nil 或 error。

### 5.5 Model CRUD 中的真实错误传播链

以 `app.Delete()` 为例，参见 [db.go#L110-L173](file:///d:/fz/0601/solo-dogfeeding/code/159-pocketbase/core/db.go#L110-L173)：

```go
// 最外层：用户调用 app.Delete()
deleteErr := app.OnModelDelete().Trigger(event, func(e *ModelEvent) error {
    // oneOff 的前半：检查 PK
    if pk == "" {
        return errors.New("no pk")  // Next 前返回 → 前进中断
    }
    // oneOff 内嵌套触发另一条钩子链
    return e.App.OnModelDeleteExecute().Trigger(event, func(e *ModelEvent) error {
        // 最内层 oneOff：执行 DB DELETE
        _, err := db.Delete(...).Execute()
        return err  // DB 错误沿返回方向冒泡
    })
})
```

错误传播路径：

```
1. DB DELETE 失败 → 最内层 oneOff 返回 err
2. OnModelDeleteExecute 的返回方向冒泡
3. OnModelDeleteExecute.Trigger() 返回 err
4. 外层 oneOff 收到 err，直接 return（策略 1：透传）
5. OnModelDelete 的返回方向冒泡
6. OnModelDelete.Trigger() 返回 err → 赋值给 deleteErr
```

**关键理解**：`Trigger()` 的返回值就是整个洋葱链最外层 handler 的最终返回值。

### 5.6 错误合并：AfterError 钩子的特殊处理

CRUD 失败后，会触发独立的 `OnModelAfter*Error` 钩子，它的错误会与原始错误合并，参见 [db.go#L141-L150](file:///d:/fz/0601/solo-dogfeeding/code/159-pocketbase/core/db.go#L141-L150)：

```go
if deleteErr != nil {
    errEvent := &ModelErrorEvent{ModelEvent: *event, Error: deleteErr}
    hookErr := app.OnModelAfterDeleteError().Trigger(errEvent)
    if hookErr != nil {
        return errors.Join(deleteErr, hookErr)  // 两个错误都保留
    }
    return deleteErr
}
```

这里是 **两条独立的钩子链**：
- 第一条链 `OnModelDelete` 负责实际操作，返回 `deleteErr`
- 第二条链 `OnModelAfterDeleteError` 负责错误后的回调，返回 `hookErr`
- 两个错误通过 `errors.Join` 合并后返回，互不覆盖

### 5.7 事务场景下的延迟触发

若操作在事务中（`app.txInfo != nil`），`AfterSuccess` / `AfterError` 钩子 **延迟到事务完成后** 才触发，参见 [db.go#L152-L167](file:///d:/fz/0601/solo-dogfeeding/code/159-pocketbase/core/db.go#L152-L167)：

```go
if app.txInfo != nil {
    app.txInfo.OnComplete(func(txErr error) error {
        if txErr != nil {
            // 事务回滚 → 触发 AfterError
            return app.OnModelAfterDeleteError().Trigger(...)
        }
        // 事务提交 → 触发 AfterSuccess
        return app.OnModelAfterDeleteSuccess().Trigger(event)
    })
}
```

这样确保只有真正提交成功的数据才触发 `AfterSuccess` 钩子。

---

## 六、Model 与 Record 的钩子代理关系

### 6.1 两层钩子体系

PocketBase 存在 **两层** 模型钩子：

| 层级 | 钩子示例 | 适用范围 |
|------|----------|----------|
| 通用 Model 层 | `OnModelCreate`、`OnModelUpdate`... | 所有 Model（Record、Collection、Settings 等） |
| 专用 Record 层 | `OnRecordCreate`、`OnRecordUpdate`... | 仅 Record 类型模型 |

### 6.2 代理桥接机制

Record 层钩子并非独立存在，而是通过 **系统内置 Handler** 桥接自 Model 层。参见 [record_model.go#L55-L288](file:///d:/fz/0601/solo-dogfeeding/code/159-pocketbase/core/record_model.go#L55-L288)。

以 `OnModelCreate → OnRecordCreate` 为例：

```go
app.OnModelCreate().Bind(&hook.Handler[*ModelEvent]{
    Id:       "__pbRecordSystemHook__",
    Priority: -99,  // 极低优先级 → 最早执行
    Func: func(me *ModelEvent) error {
        // 尝试转换为 RecordEvent
        if re, ok := newRecordEventFromModelEvent(me); ok {
            // 是 Record 类型 → 触发 Record 层钩子
            err := me.App.OnRecordCreate().Trigger(re, func(re *RecordEvent) error {
                syncModelEventWithRecordEvent(me, re)
                defer syncRecordEventWithModelEvent(re, me)
                return me.Next()  // 继续 Model 层链条
            })
            syncModelEventWithRecordEvent(me, re)
            return err
        }
        // 非 Record 类型 → 直接跳过 Record 层
        return me.Next()
    },
})
```

### 6.3 完整触发顺序（以 Record 创建为例）

```
App.Save(record)
  │
  └─> OnModelCreate.Trigger(...)
        │
        ├─ [系统桥接 Handler, Priority=-99]
        │     │
        │     └─> OnRecordCreate.Trigger(...)
        │           │
        │           ├─ OnRecordValidate.Trigger(...)  （如果启用验证）
        │           │
        │           ├─ OnRecordCreateExecute.Trigger(...)
        │           │     └─> 实际的 DB INSERT
        │           │
        │           └─> me.Next() 回到 Model 层
        │
        ├─ OnModelValidate.Trigger(...)  （如果启用验证）
        │
        ├─ OnModelCreateExecute.Trigger(...)
        │     └─> 实际的 DB INSERT
        │
        └─> 完成 → OnModelAfterCreateSuccess
                    │
                    └─ [系统桥接 Handler]
                          └─> OnRecordAfterCreateSuccess
```

**注意**：由于桥接 Handler 的 Priority=-99，它会 **最先执行**，因此 Record 层钩子总是运行在 Model 层用户自定义钩子 **之前**。

---

## 七、App 中所有钩子一览

### 7.1 应用生命周期钩子

| 钩子 | 事件类型 | 触发时机 |
|------|----------|----------|
| `OnBootstrap()` | `BootstrapEvent` | 应用初始化资源时 |
| `OnServe()` | `ServeEvent` | HTTP 服务器启动前 |
| `OnTerminate()` | `TerminateEvent` | 应用终止时（如 SIGTERM） |
| `OnBackupCreate()` | `BackupEvent` | 创建备份时 |
| `OnBackupRestore()` | `BackupEvent` | 恢复备份前 |
| `OnSettingsReload()` | `SettingsReloadEvent` | 配置重新加载时 |

### 7.2 Model 层钩子（通用）

| 钩子 | 事件类型 | 触发时机 |
|------|----------|----------|
| `OnModelValidate(tags...)` | `ModelEvent` | 模型验证时 |
| `OnModelCreate(tags...)` | `ModelEvent` | 模型创建时（包裹验证+DB写入） |
| `OnModelCreateExecute(tags...)` | `ModelEvent` | 验证通过后、INSERT 语句执行前 |
| `OnModelAfterCreateSuccess(tags...)` | `ModelEvent` | 成功持久化后（事务提交后） |
| `OnModelAfterCreateError(tags...)` | `ModelErrorEvent` | 创建失败时 |
| `OnModelUpdate(tags...)` | `ModelEvent` | 模型更新时 |
| `OnModelUpdateExecute(tags...)` | `ModelEvent` | 验证通过后、UPDATE 语句执行前 |
| `OnModelAfterUpdateSuccess(tags...)` | `ModelEvent` | 成功更新后 |
| `OnModelAfterUpdateError(tags...)` | `ModelErrorEvent` | 更新失败时 |
| `OnModelDelete(tags...)` | `ModelEvent` | 模型删除时 |
| `OnModelDeleteExecute(tags...)` | `ModelEvent` | DELETE 语句执行前 |
| `OnModelAfterDeleteSuccess(tags...)` | `ModelEvent` | 成功删除后 |
| `OnModelAfterDeleteError(tags...)` | `ModelErrorEvent` | 删除失败时 |

### 7.3 Record 层钩子（Record 专用）

与 Model 层一一对应：`OnRecordValidate`、`OnRecordCreate`、`OnRecordCreateExecute`、`OnRecordAfterCreateSuccess`、`OnRecordAfterCreateError`、`OnRecordUpdate`... 以及 `OnRecordEnrich`（记录富化时触发）。

### 7.4 Collection 层钩子（Collection 专用）

同样与 Model 层一一对应：`OnCollectionValidate`、`OnCollectionCreate`...

### 7.5 HTTP API 请求钩子

| 类别 | 示例钩子 |
|------|----------|
| Settings API | `OnSettingsListRequest`、`OnSettingsUpdateRequest` |
| File API | `OnFileTokenRequest`、`OnFileDownloadRequest` |
| Record Auth API | `OnRecordAuthRequest`、`OnRecordAuthWithPasswordRequest`、`OnRecordAuthWithOAuth2Request`、`OnRecordAuthRefreshRequest`、`OnRecordRequestPasswordResetRequest` 等 |
| Record CRUD API | `OnRecordsListRequest`、`OnRecordViewRequest`、`OnRecordCreateRequest`、`OnRecordUpdateRequest`、`OnRecordDeleteRequest` |
| Collection API | `OnCollectionsListRequest`、`OnCollectionViewRequest`、`OnCollectionCreateRequest`、`OnCollectionUpdateRequest`、`OnCollectionDeleteRequest`、`OnCollectionsImportRequest` |
| Realtime API | `OnRealtimeConnectRequest`、`OnRealtimeMessageSend`、`OnRealtimeSubscribeRequest` |
| Batch API | `OnBatchRequest` |

### 7.6 Mailer 钩子

| 钩子 | 触发时机 |
|------|----------|
| `OnMailerSend()` | 每次发送邮件时 |
| `OnMailerRecordAuthAlertSend(tags...)` | 发送登录预警邮件时 |
| `OnMailerRecordPasswordResetSend(tags...)` | 发送密码重置邮件时 |
| `OnMailerRecordVerificationSend(tags...)` | 发送邮箱验证邮件时 |
| `OnMailerRecordEmailChangeSend(tags...)` | 发送邮箱变更确认邮件时 |
| `OnMailerRecordOTPSend(tags...)` | 发送 OTP 邮件时 |

---

## 八、Router 中的钩子应用

PocketBase 的 HTTP 路由器也复用同一套钩子机制。参见 [tools/router/router.go#L90-L143](file:///d:/fz/0601/solo-dogfeeding/code/159-pocketbase/tools/router/router.go#L90-L143)。

### 8.1 路由匹配时的钩子聚合

当请求匹配到某个 Route 时，路由器会按以下顺序收集 Handler：

```
1. Router 级别的 middlewares
2. 逐级 Group 的 middlewares（从外到内）
3. Route 自身的 middlewares
```

全部聚合到一个临时 `routeHook` 中，然后触发：

```go
routeHook.Trigger(event, v.Action)
// v.Action 是路由的最终处理函数（相当于 oneOff）
```

这使得中间件也遵循 **洋葱模型**：Router 级 → Group 级 → Route 级 → Action → Route 级 → Group 级 → Router 级。

---

## 九、关键设计要点总结

| 特性 | 实现方式 | 代码位置 |
|------|----------|----------|
| 线程安全 | `sync.RWMutex` 保护 handlers 切片 | [hook.go#L56](file:///d:/fz/0601/solo-dogfeeding/code/159-pocketbase/tools/hook/hook.go#L56) |
| 执行顺序 | Priority 升序 + `sort.SliceStable` 保持注册顺序 | [hook.go#L98-L101](file:///d:/fz/0601/solo-dogfeeding/code/159-pocketbase/tools/hook/hook.go#L98-L101) |
| 链式调用 | 倒序构建闭包，通过 `event.next` 串联 | [hook.go#L164-L173](file:///d:/fz/0601/solo-dogfeeding/code/159-pocketbase/tools/hook/hook.go#L164-L173) |
| 标签过滤 | TaggedHook 包装 Func，不匹配时跳过 | [tagged.go#L58-L69](file:///d:/fz/0601/solo-dogfeeding/code/159-pocketbase/tools/hook/tagged.go#L58-L69) |
| 错误传播 | Next 前返回 error 中断前进方向；Next 后返回 error 仅沿返回方向冒泡；外层可捕获 `e.Next()` 返回值决定透传/吞掉/包装 | [event.go#L30-L35](file:///d:/fz/0601/solo-dogfeeding/code/159-pocketbase/tools/hook/event.go#L30-L35)、[hook_test.go#L32](file:///d:/fz/0601/solo-dogfeeding/code/159-pocketbase/tools/hook/hook_test.go#L32) |
| 事务延迟 | `txInfo.OnComplete` 回调中触发 After 钩子 | [db.go#L152-L167](file:///d:/fz/0601/solo-dogfeeding/code/159-pocketbase/core/db.go#L152-L167) |
| Model↔Record 桥接 | Priority=-99 的系统 Handler 做类型转换转发 | [record_model.go#L55-L288](file:///d:/fz/0601/solo-dogfeeding/code/159-pocketbase/core/record_model.go#L55-L288) |
