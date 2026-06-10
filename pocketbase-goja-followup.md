# PocketBase goja JS Hooks 深度追踪分析（续）

本文深入分析上一文档未覆盖的四个关键问题：文件加载顺序的真正来源、Loader VM 的精确生命周期、多文件 Hook 的顺序风险、以及顶层状态保存的边界。

---

## 一、Hook 文件加载顺序的真正来源

### 1.1 代码注释与实际行为的矛盾

在 [registerMigrations()](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/jsvm.go#L183-L185) 和 [registerHooks()](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/jsvm.go#L237-L239) 的开头，注释都写着：

```go
// fetch all js migrations sorted by their filename
files, err := filesContent(...)
```

但这个注释具有严重的误导性——`filesContent()` 返回的是 **`map[string][]byte`**，而 Go 语言规范明确规定 **map 的迭代顺序是不确定的**。每次程序启动、甚至同一次运行中的两次迭代，顺序都可能不同。

### 1.2 `filesContent()` 的真实行为

[`filesContent()`](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/jsvm.go#L537-L570) 的执行流程：

```go
func filesContent(dirPath string, pattern string) (map[string][]byte, error) {
    files, err := os.ReadDir(dirPath)  // (1) os.ReadDir 返回已按文件名排序的 []DirEntry
    // ...
    result := map[string][]byte{}      // (2) 创建空 map
    for _, f := range files {          // (3) 按排序后的顺序遍历
        // ...
        result[f.Name()] = raw         // (4) 写入 map，顺序在此丢失
    }
    return result, nil                 // (5) 返回无序 map
}
```

关键点：
- 第 1 步 `os.ReadDir()` 确实按文件名排序返回条目（Go 标准库文档明确说明）
- 但第 4 步将排序后的数据写入 `map[string][]byte`，**排序信息永久丢失**
- 最终调用方用 `for file, content := range files` 迭代时，顺序完全随机

### 1.3 调用方的迭代方式

两个注册函数都直接使用 `for...range` 迭代 map：

**Migrations**（[jsvm.go:198](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/jsvm.go#L198)）：
```go
for file, content := range files {  // 每次启动顺序可能不同
    vm := goja.New()
    // ...
    _, err := vm.RunScript(defaultScriptPath, string(content))
}
```

**Hooks**（[jsvm.go:328](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/jsvm.go#L328)）：
```go
for file, content := range files {  // 每次启动顺序可能不同
    func() {
        // ...
        _, err := loader.RunScript(defaultScriptPath, string(content))
    }()
}
```

### 1.4 对 Migrations 的严重影响

Migrations 通常使用文件名前缀（如 `1620000000_initial.js`、`1630000000_add_email.js`）来表达依赖顺序。但由于 map 迭代无序，`migrate()` 函数被注册到 `core.AppMigrations` 的顺序是随机的。

注意：PocketBase 的 `core.AppMigrations` 内部**可能**在注册时按文件名重新排序（需要进一步确认），但在 JSVM 插件层面，这个顺序是不保证的。

### 1.5 结论

| 阶段 | 是否排序 | 依据 |
|------|----------|------|
| `os.ReadDir()` 读取目录 | ✅ 按文件名升序 | Go 标准库保证 |
| 存入 `map[string][]byte` 后 | ❌ 不确定 | Go map 迭代无顺序保证 |
| `registerMigrations()` 执行 | ❌ 不确定 | `for...range` 无序 map |
| `registerHooks()` 执行 | ❌ 不确定 | `for...range` 无序 map |

**实际顺序来源**：文件加载顺序完全由 Go runtime 的 map 迭代实现决定，不受文件名影响。同一份代码在不同机器、不同启动时刻、甚至同一进程两次运行之间，顺序都可能不同。

---

## 二、Loader VM 的完整生命周期

### 2.1 创建与初始化

Loader VM 在 [`registerHooks()`](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/jsvm.go#L321-L326) 中创建：

```go
// initialize the loader vm
loader := goja.New()                          // (1) 全新 goja.Runtime
sharedBinds(loader)                           // (2) 注入 BindCore/BindDbx/.../$app/$template/__hooks
hooksBinds(p.app, loader, executors)          // (3) 注入 onRecordCreate() 等注册函数
cronBinds(p.app, loader, executors)           // (4) 注入 cronAdd()/cronRemove()
routerBinds(p.app, loader, executors)         // (5) 注入 routerAdd()/routerUse()
```

`loader` 是 `registerHooks()` 函数的**局部变量**，不存储在 `plugin` 结构体或任何长期存活对象中。

### 2.2 活跃阶段：脚本加载循环

紧接着创建后，Loader VM 用于顺序执行所有 Hook 文件（[jsvm.go:328-347](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/jsvm.go#L328-L347)）：

```go
for file, content := range files {
    func() {
        defer func() { /* panic recovery */ }()
        _, err := loader.RunScript(defaultScriptPath, string(content))
        if err != nil {
            panic(err)
        }
    }()
}
```

此阶段 Loader VM 的 JS 全局状态（`globalThis`）是所有文件共享的——如果 a.pb.js 顶层 `globalThis.X = 123`，b.pb.js 顶层可以读到 `X`（前提是 a 先被 map 迭代到）。

### 2.3 "冻结"阶段：无引用可被 GC

`registerHooks()` 返回后：

```
registerHooks() 返回
    │
    ├─ loader 变量超出作用域
    ├─ 没有任何字段/全局变量持有 loader 的引用
    ├─ 因此 loader 在下一次 GC 时即可被回收
    │
    └─ 但注意：hooksBinds/cronBinds/routerBinds 中通过 loader.Set() 注入的
       Go 闭包（如 onRecordCreate 的实现）已随 loader.Set 被 goja.Runtime 内部
       持有引用。不过，由于这些闭包仅在 Loader VM 活跃时（即 Hook 文件顶层
       代码执行时）被调用，一旦加载完成它们就再也不会被触发了——因此
       loader 的可到达性取决于 Go GC 是否能追踪到这些闭包的反向引用。
```

**关键点**：Loader VM **不需要**存活到运行时阶段，因为：
- 所有 JS 回调都已被编译为独立的 `*goja.Program`（不绑定特定 Runtime）
- 回调的执行完全由 Executor Pool 负责
- 注册到 Go Hook 系统的是 Go 闭包，而非 goja 对象

### 2.4 特殊例外：cronBinds 的双向注入

[`cronBinds()`](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/binds.go#L104-L147) 有一个独特行为——它不仅将 `cronAdd`/`cronRemove` 注入 Loader VM，还替换 Executor Pool 的 factory 并为所有已预热 VM 注入这两个函数：

```go
// register the removal helper also in the executors to allow removing cron jobs from everywhere
oldFactory := executors.factory
executors.factory = func() *goja.Runtime {
    vm := oldFactory()
    vm.Set("cronAdd", cronAdd)
    vm.Set("cronRemove", cronRemove)
    return vm
}
for _, item := range executors.items {
    item.vm.Set("cronAdd", cronAdd)
    item.vm.Set("cronRemove", cronRemove)
}
```

这意味着 cron 管理函数可在 JS 回调执行时（Executor VM 内）动态调用，而不仅仅在 Loader 阶段。`cronAdd` 闭包内部引用了 `app.Cron()` 和 `executors`，但不引用 Loader VM——因此 Loader VM 的 GC 不受影响。

### 2.5 生命周期总览图

```
plugin.Init()
  │
  ├─ registerHooks() 被调用
  │   │
  │   ├─ loader := goja.New()        ← Loader VM 创建
  │   ├─ sharedBinds(loader)         ← 注入宿主绑定
  │   ├─ hooksBinds/cronBinds/...    ← 注入注册函数
  │   │
  │   ├─ for file, content := range files
  │   │   └─ loader.RunScript(...)   ← 活跃期：执行每个文件顶层代码
  │   │
  │   └─ registerHooks() 返回        ← loader 变量超出作用域
  │
  │                                  ← Loader VM 可被 GC（运行时阶段不再需要）
  │
  └─ App 启动，HTTP 请求到达
      │
      └─ Executor Pool 执行 JS 回调  ← 与 Loader VM 完全无关
```

---

## 三、多文件 Hook 的顺序风险

### 3.1 风险层级一：文件顶层代码的跨文件依赖

**场景**：
```js
// pb_hooks/utils.pb.js
function formatEmail(user) {  // 在 Loader VM 顶层定义
    return user.get("email").toLowerCase();
}
globalThis.formatEmail = formatEmail;
```

```js
// pb_hooks/handlers.pb.js
onRecordAfterUpdateRequest("users", (e) => {
    e.record.set("email", formatEmail(e.record));  // 引用 globalThis.formatEmail
    return e.next();
});
```

**风险分析**：
- 两个文件都在**同一个 Loader VM** 中顺序执行（不管 map 顺序如何，都是串行的）
- 如果 `utils.pb.js` 先被迭代到 → `formatEmail` 定义成功，`handlers.pb.js` 执行时可用
- 如果 `handlers.pb.js` 先被迭代到 → 顶层代码执行时 `onRecordAfterUpdateRequest()` 立即调用，JS 回调函数体（闭包）被编译但**不立即执行**，因此对 `formatEmail` 的引用是**延迟解析**的
- **但**：回调在 Executor VM 中执行时，Executor VM 的 globalThis 上**根本没有** `formatEmail`，因为 `utils.pb.js` 的顶层代码从未在 Executor VM 中运行过

**结论**：即使文件顺序碰巧正确，这种跨文件的顶层工具函数也**无法在 Hook 回调中使用**，因为回调在 Executor VM 中执行，而工具函数定义在 Loader VM 中。

### 3.2 风险层级二：同一 Hook 的多 Handler 执行顺序

假设两个不同文件注册了同一事件的 Hook：

```js
// pb_hooks/a.pb.js
onRecordCreate("posts", (e) => {   // Handler A
    console.log("A: before next");
    e.next();
    console.log("A: after next");
});
```

```js
// pb_hooks/b.pb.js
onRecordCreate("posts", (e) => {   // Handler B
    console.log("B: before next");
    e.next();
    console.log("B: after next");
});
```

[Hook.Bind()](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/tools/hook/hook.go#L65-L104) 的实现：

```go
// append new
if !exists {
    h.handlers = append(h.handlers, handler)
}
// sort handlers by Priority, preserving the original order of equal items
sort.SliceStable(h.handlers, func(i, j int) bool {
    return h.handlers[i].Priority < h.handlers[j].Priority
})
```

分析：
- JS 的 `onRecordCreate()` 没有 Priority 参数，两个 Handler 的 Priority 都是 0
- `sort.SliceStable` 在 Priority 相等时保持原注册顺序
- 注册顺序 = map 迭代到两个文件的顺序（不确定）
- 因此 Handler A 和 B 的执行顺序在每次启动时可能不同

Hook 的洋葱模型执行效果：
- 如果先注册 A 再注册 B → 执行序列：A:before → B:before → B:after → A:after
- 如果先注册 B 再注册 A → 执行序列：B:before → A:before → A:after → B:after

对于依赖执行顺序的逻辑（如 A 负责设置上下文、B 消费上下文），这会导致**偶发的 Heisenbug**。

### 3.3 风险层级三：Migrations 依赖顺序

Migrations 通常使用时间戳文件名表达依赖关系（如 `1620000000_init.js`、`1630000000_add_field.js`）。虽然 `core.AppMigrations.Register()` 内部可能做排序，但：

- JS 层面如果在迁移顶层直接调用了 `$app.*` 产生副作用（而非仅在 `migrate(up, down)` 中），则副作用顺序是随机的
- 不同迁移文件在独立 VM 中执行，无法通过 JS 全局变量共享状态（虽然通常不应该这么做）

### 3.4 风险层级四：Router 绑定顺序

`routerAdd()` 通过 `app.OnServe().BindFunc()` 延迟注册路由。多个文件注册路由时：

```js
// pb_hooks/api_v1.pb.js
routerAdd("GET", "/api/hello", (c) => { return c.json(200, {v: 1}) });
```

```js
// pb_hooks/api_v2.pb.js
routerAdd("GET", "/api/hello", (c) => { return c.json(200, {v: 2}) });
```

两条路由路径完全相同，后注册者会覆盖先注册者吗？取决于 `echo.Router` 的匹配策略——通常后者不覆盖前者，但匹配行为本身是不确定的。

### 3.5 风险总结与规避建议

| 风险类型 | 影响程度 | 确定性 | 规避方式 |
|----------|----------|--------|----------|
| 顶层工具函数跨文件不可用 | 高 | 100%（必然发生） | 使用 `require()` 模块系统（共享 `require.Registry`） |
| 同 Hook 多 Handler 执行顺序不确定 | 中 | 随启动变化 | 使用 `Priority` 参数（但 JS API 目前未暴露），或合并到单文件 |
| Migrations 顶层副作用顺序不确定 | 中/高 | 随启动变化 | 所有副作用仅限 `migrate(up, down)` 回调内部 |
| 相同路径路由注册冲突 | 低 | 随启动变化 | 确保路径唯一，避免跨文件定义相同路由 |

---

## 四、顶层状态保存的边界

### 4.1 三类 VM 中各状态的可访问性

| 状态来源 | 定义位置 | Loader VM 可访问 | Executor VM 可访问 | 跨 Executor 复用是否保留 |
|----------|----------|-----------------|-------------------|--------------------------|
| `$app` | `sharedBinds()` 注入 | ✅ | ✅（每次执行前被覆盖） | ❌（每次 Hook 执行前 `Set("$app", undefined)`，由 JS 包装内的 `e.app` 覆盖） |
| `$template` | `sharedBinds()` 注入 | ✅ | ✅ | ✅（永远不被修改，指向同一 Registry） |
| `__hooks` | `sharedBinds()` 注入 | ✅ | ✅ | ✅（永远不被修改，字符串常量） |
| `require.Registry` 模块缓存 | 跨 VM 共享 | ✅ | ✅ | ✅（所有 VM 共享同一 Registry 引用） |
| `cronAdd`/`cronRemove` | `cronBinds()` 特殊注入 | ✅ | ✅ | ✅（注入所有 Executor VM） |
| Loader VM 顶层 `var X = ...` | a.pb.js 顶层代码 | ✅ | ❌（Executor VM 从未执行过 a.pb.js） | N/A |
| Executor VM 内 `globalThis.X = ...` | Hook 回调执行时写入 | ❌ | ✅（对当前 VM） | ⚠️ **会泄漏**（池化复用，无重置） |
| `__args` | Hook 执行前注入 | ❌ | ✅（仅执行期间） | ❌（执行完显式 `Set("__args", undefined)`） |

### 4.2 Executor VM 池的状态泄漏：最关键的隐患

[`vmsPool.run()`](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/pool.go#L38-L73) 的核心逻辑：

```go
func (p *vmsPool) run(call func(vm *goja.Runtime) error) error {
    // ... 找到空闲 item 或创建临时 VM ...
    execErr := call(freeItem.vm)   // (1) 执行用户回调，可能修改 globalThis
    // "free" the vm
    freeItem.mux.Lock()
    freeItem.busy = false          // (2) 仅标记为空闲，不清理 VM 状态
    freeItem.mux.Unlock()
    return execErr
}
```

**池化复用的 VM 只重置 `busy` 标志，从不清理 JS 全局状态。**

对比 Hook 执行前后仅做了极有限的清理。以 `hooksBinds()` 中的包装为例（[binds.go:81-93](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/binds.go#L81-L93)）：

```go
err := executors.run(func(executor *goja.Runtime) error {
    executor.Set("$app", goja.Undefined())  // 清理 1
    executor.Set("__args", handlerArgs)     // 注入参数
    res, err := executor.RunProgram(pr)
    executor.Set("__args", goja.Undefined()) // 清理 2
    // check error...
    return normalizeException(err)
})
```

只清理了两个变量：
- `$app` → 设为 undefined（后续由 JS 包装 `$app = e.app` 覆写）
- `__args` → 执行完设为 undefined

**没有被清理的变量**：
- 用户 JS 回调中写入 `globalThis` 的任何自定义属性
- 用户 JS 在顶层代码中定义的变量（注：但 Executor VM 不执行 Hook 文件顶层代码，所以这条仅适用于通过 `require()` 加载的模块）
- 通过 `require()` 引入模块时产生的副作用

### 4.3 状态泄漏示例

```js
// pb_hooks/leak.pb.js
let counter = 0;  // 注意：这里是 Loader VM 顶层，不影响 Executor

onRecordCreate("posts", (e) => {
    if (globalThis.visitCount === undefined) {
        globalThis.visitCount = 0;  // 写入 Executor VM 的 globalThis
    }
    globalThis.visitCount++;
    console.log("visitCount =", globalThis.visitCount);
    return e.next();
});
```

假设池大小为 2，第 1、3、5 次请求使用 VM-A，第 2、4 次使用 VM-B：
- 请求 1（VM-A）：`visitCount` undefined → 初始化为 1，输出 1
- 请求 2（VM-B）：`visitCount` undefined → 初始化为 1，输出 1
- 请求 3（VM-A）：`visitCount` 已存在 → 2，输出 2
- 请求 4（VM-B）：`visitCount` 已存在 → 2，输出 2
- 请求 5（VM-A）：输出 3

**每个 VM 维护独立的计数器**，用户看到的是不确定的值（取决于请求被分配到哪个 VM）。如果用户用这种方式统计请求量，数据将严重失真。

### 4.4 正确的状态持久化方式

用户应通过宿主暴露的机制而非 JS 全局变量来保存状态：

```js
// 正确方式：使用 $app.Store()（所有 VM 共享同一 App 实例）
onRecordCreate("posts", (e) => {
    let count = $app.store().get("visitCount") || 0;
    count++;
    $app.store().set("visitCount", count);
    return e.next();
});
```

`$app.Store()` 返回的是 [`*store.Store[string, any]`](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/tools/store)，是进程级单例，在所有 VM 间共享且线程安全。

### 4.5 `require()` 模块的共享边界

所有 VM 共享同一个 `require.Registry`，通过 `require()` 加载的模块在所有 VM 中是**单例**：

```js
// pb_hooks/lib/counter.js (通过 require 加载)
let count = 0;
module.exports = { inc: () => ++count, get: () => count };
```

```js
// pb_hooks/hook.pb.js
const counter = require("./lib/counter.js");

onRecordCreate("posts", (e) => {
    counter.inc();  // 在所有 VM 间共享同一个模块实例
    console.log(counter.get());  // 值在所有请求间一致递增
    return e.next();
});
```

但注意：`require()` 加载的模块内部如果涉及 JS 对象闭包，这些闭包状态是被所有 VM 共享的——因为它们存储在 `require.Registry` 中，而不是某个特定 VM 的 `globalThis`。这意味着如果模块内部使用 `globalThis`，**读取到的是模块被首次加载时所在 VM 的 globalThis**（通常是某个 Executor VM，因为 Loader VM 顶层 require 也可用，但用户一般在回调中 require）。

### 4.6 状态隔离总览图

```
┌────────────────────────────────────────────────────────────────────────┐
│                         进程级（所有 VM 共享）                          │
│                                                                        │
│  core.App (p.app)                  require.Registry                   │
│    ├─ $app.Store()                   └─ 所有 require() 的模块           │
│    ├─ $app.DB() / Cron()                                                │
│    └─ $app.Cron()                                                      │
│                                                                        │
│  template.Registry ($template)                                          │
│  cronAdd / cronRemove 闭包（指向 app.Cron()）                           │
│                                                                        │
├────────────────────────────────────────────────────────────────────────┤
│                         VM 级（每个 goja.Runtime 独立）                  │
│                                                                        │
│  Executor VM A                     Executor VM B                       │
│    ├─ globalThis.* (可能泄漏)         ├─ globalThis.*                   │
│    ├─ $app (每次覆盖)                 ├─ $app                           │
│    └─ __args (执行后清理)              └─ __args                        │
│                                                                        │
│  Loader VM（可被 GC）                                                   │
│    └─ globalThis.*（仅加载阶段存在，执行阶段不使用）                      │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 五、关键发现汇总

1. **文件加载顺序完全不确定**：注释声称按文件名排序，但实际使用 `for...range` 迭代无序 map。每次启动 Hook 注册顺序可能不同。

2. **Loader VM 寿命极短**：`registerHooks()` 返回后即可被 GC。它的唯一作用是执行 Hook 文件的顶层代码并触发注册函数，运行时阶段完全不参与。

3. **顶层工具函数无法在回调中使用**：Loader VM 顶层定义的函数/变量存储在 Loader VM 的 `globalThis` 中，而回调在 Executor VM 中执行，两者完全隔离。必须用 `require()` 模块（共享 Registry）替代。

4. **Executor Pool 存在状态泄漏风险**：池化 VM 复用时只重置 `busy` 标志，不清理 JS 全局状态。用户在回调中写入 `globalThis` 的属性会在同一 VM 的后续请求中可见，且值取决于请求被调度到哪个 VM。

5. **安全的跨请求状态**：应使用 `$app.store()`（进程级、线程安全）或数据库，绝不能依赖 JS 全局变量。
