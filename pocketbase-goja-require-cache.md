# goja_nodejs require.Registry 与 RequireModule 缓存边界深度分析

本文基于 goja_nodejs v0.0.0-20260212111938-1f56ff5bcf14 的源码，结合 PocketBase 的 jsvm 插件使用方式，精确校正 require 系统的三层缓存边界：编译缓存、模块 exports 缓存、闭包状态。

---

## 一、核心数据结构

### 1.1 Registry：跨 Runtime 共享的编译层

`Registry` 是进程级单例，可被多个 `goja.Runtime` 共享。PocketBase 在 [`registerHooks()`](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/jsvm.go#L285) 中创建一个共享实例：

```go
requireRegistry := new(require.Registry) // safe to be shared across multiple vms
```

其内部结构（来自 goja_nodejs require/module.go）：

```go
type Registry struct {
    sync.Mutex
    native   map[string]ModuleLoader
    compiled map[string]*js.Program   // 关键：编译缓存，key = 文件绝对路径

    srcLoader      SourceLoader
    pathResolver   PathResolver
    globalFolders  []string
}
```

- `compiled map[string]*goja.Program`：**全局编译缓存**。JS 源文件首次被 require 时解析并编译为 Program，后续所有 VM 复用同一份字节码。
- `sync.Mutex`：保护 `compiled` 和 `native` map 的并发访问。

### 1.2 RequireModule：per-Runtime 的模块实例层

调用 `Registry.Enable(runtime)` 时为**每个 Runtime 创建独立的 `RequireModule` 实例**（来自 goja_nodejs require/module.go）：

```go
func (r *Registry) Enable(runtime *js.Runtime) *RequireModule {
    rrt := &RequireModule{
        r:           r,              // 回指共享 Registry
        runtime:     runtime,        // 绑定到具体 goja.Runtime
        modules:     make(map[string]*js.Object),  // per-VM module 缓存
        nodeModules: make(map[string]*js.Object),  // per-VM node_modules 缓存
    }
    runtime.Set("require", rrt.require)
    return rrt
}
```

关键字段：
- `modules map[string]*goja.Object`：**当前 Runtime 内已加载的 module 对象**。每个 module 对象包含 `exports` 属性以及模块函数闭包状态。
- `nodeModules map[string]*goja.Object`：同上，但专门缓存从 `node_modules/` 解析的模块。

### 1.3 架构分层图

```
┌───────────────────────────────────────────────────────────────────────┐
│                      进程级（所有 VM 共享）                             │
│                                                                       │
│  require.Registry                                                     │
│    ├─ compiled map[path]*goja.Program   ← 编译缓存，只编译一次          │
│    ├─ native   map[path]ModuleLoader     ← 原生 Go 模块注册             │
│    └─ srcLoader / pathResolver            ← 文件读取与路径解析策略       │
│                                                                       │
├───────────────────────────────────────────────────────────────────────┤
│                      VM 级（每个 goja.Runtime 独立）                     │
│                                                                       │
│  Loader VM (goja.Runtime)         Executor VM-A (goja.Runtime)        │
│    ├─ RequireModule                    ├─ RequireModule                │
│    │   ├─ modules map[path]*Object     │   ├─ modules map[path]*Object │
│    │   │   └─ /path/to/m.js → moduleA  │   │   └─ /path/to/m.js → modA │
│    │   └─ nodeModules map[...]         │   └─ nodeModules map[...]     │
│    └─ ...                              └─ ...                          │
│                                                                       │
│  Executor VM-B (goja.Runtime)                                         │
│    ├─ RequireModule                                                    │
│    │   └─ modules map[path]*Object                                     │
│    │       └─ /path/to/m.js → modB  ← 独立于 modA 的另一个 module 实例  │
│    └─ ...                                                              │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 二、编译缓存（source → Program）的共享边界

### 2.1 缓存实现

[`Registry.getCompiledSource()`](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/jsvm.go) 的逻辑（来自 goja_nodejs）：

```go
func (r *Registry) getCompiledSource(p string) (*js.Program, error) {
    r.Lock()
    defer r.Unlock()

    prg := r.compiled[p]
    if prg == nil {
        buf, err := r.getSource(p)          // (1) 读文件
        s := string(buf)
        // JSON 文件特殊处理：包装为 module.exports = JSON.parse(...)
        if filepath.Ext(p) == ".json" {
            s = "module.exports = JSON.parse('" + template.JSEscapeString(s) + "')"
        }
        // (2) 包装为 Node.js CommonJS 模块函数
        source := "(function(exports,require,module,__filename,__dirname){" + s + "\n})"
        parsed, err := js.Parse(p, source, parser.WithSourceMapLoader(r.srcLoader))
        prg, err = js.CompileAST(parsed, false)
        if err == nil {
            if r.compiled == nil {
                r.compiled = make(map[string]*js.Program)
            }
            r.compiled[p] = prg   // (3) 存入全局缓存
        }
        return prg, err
    }
    return prg, nil
}
```

### 2.2 共享范围

| 维度 | 是否共享 | 说明 |
|------|----------|------|
| Loader VM ↔ Executor VM | ✅ 共享 | 共用同一个 `requireRegistry`，引用同一份 `compiled` map |
| Executor VM-A ↔ Executor VM-B | ✅ 共享 | 同上 |
| 线程安全 | ✅ 安全 | `r.Lock()` / `r.Unlock()` 互斥保护 |
| 缓存失效 | ❌ 无失效 | 进程生命周期内一旦编译即永久缓存（HooksWatch 触发的进程重启会清空） |

### 2.3 Program 的本质

`*goja.Program` 是**与 Runtime 无关的纯字节码表示**（相当于 Java class 文件或 .NET IL）。它不包含任何执行时状态，因此可以被任意数量的 Runtime 并发执行而互不干扰。这是编译缓存能安全共享的基础。

---

## 三、模块 exports 对象的共享边界

### 3.1 模块加载流程

[`RequireModule.loadModule()`](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/jsvm.go)（来自 goja_nodejs require/resolve.go）：

```go
func (r *RequireModule) loadModule(path string) (*js.Object, error) {
    module := r.modules[path]   // (1) 先查当前 VM 的 module 缓存
    if module == nil {
        module = r.createModuleObject()  // (2) 新建：{ exports: {} }
        r.modules[path] = module          // (3) 存入当前 VM 的缓存
        err := r.loadModuleFile(path, module)  // (4) 执行源码填充 exports
        if err != nil {
            module = nil
            delete(r.modules, path)
            if errors.Is(err, ModuleFileDoesNotExistError) { err = nil }
        }
        return module, err
    }
    return module, nil
}
```

步骤 1 是关键——它查的是 `RequireModule.modules`，而不是 `Registry` 上的全局 map。`RequireModule` 是 per-Runtime 的，因此 module 缓存也是 per-VM 的。

### 3.2 exports 对象的创建

[`RequireModule.loadModuleFile()`](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/jsvm.go) 执行已编译的 Program：

```go
func (r *RequireModule) loadModuleFile(path string, jsModule *js.Object) error {
    prg, err := r.r.getCompiledSource(path)  // 从 Registry 获取共享的编译缓存
    f, err := r.runtime.RunProgram(prg)       // 在当前 Runtime 中执行字节码
    if call, ok := js.AssertFunction(f); ok {
        jsExports := jsModule.Get("exports")   // 即 module.exports = {}
        jsRequire := r.runtime.Get("require")
        // 调用 CommonJS 包装函数：
        // function(exports, require, module, __filename, __dirname) { ... }
        _, err = call(jsExports, jsExports, jsRequire, jsModule,
            r.runtime.ToValue(path),
            r.runtime.ToValue(filepath.Dir(path)))
    }
    return nil
}
```

重要细节：`r.runtime.RunProgram(prg)` 在**当前 Runtime 内**执行字节码。虽然 `prg` 是共享的，但执行产生的函数对象、闭包、变量全部驻留在当前 Runtime 的 JS 堆中。

### 3.3 共享范围（校正结论）

| 维度 | 是否共享 | 说明 |
|------|----------|------|
| 同一 VM 内多次 require 同一模块 | ✅ 共享 | 命中 `RequireModule.modules`，返回同一个 `module.exports` 对象 |
| Loader VM require ↔ Executor VM require | ❌ **不共享** | 两个独立的 `RequireModule` 实例，各自有独立的 `modules` map |
| Executor VM-A ↔ Executor VM-B | ❌ **不共享** | 同上，每个池化 VM 有独立缓存 |
| 池化 VM 多次借还之间 | ✅ **保留** | VM 归还时不清理 `RequireModule.modules`，下次借用同一 VM 时仍可命中 |

**关键校正**：我在之前文档中提到 "require() 模块在所有 VM 间是单例"不够准确。更精确的说法是：
- **编译后的字节码**（`*goja.Program`）是进程级单例
- **执行后的 module 对象**（含 `exports`、闭包状态）是 per-VM 的，每个 Runtime 有自己的独立实例

### 3.4 对 PocketBase Executor Pool 的影响

PocketBase 的 Executor Pool 有 N 个预热 VM（默认 15）。假设一个模块 `counter.js`：

```js
// pb_hooks/lib/counter.js
let count = 0;
module.exports = { inc: () => ++count, get: () => count };
```

如果 3 个不同的 Executor VM 各自 require 了这个模块：

```
请求 1 → VM-A require("counter.js")
  → 未命中缓存，创建 moduleA{exports:{inc,get}}, countA=0 → inc() → countA=1
请求 2 → VM-B require("counter.js")
  → 未命中缓存，创建 moduleB{exports:{inc,get}}, countB=0 → inc() → countB=1
请求 3 → VM-A require("counter.js")
  → 命中缓存，返回 moduleA → inc() → countA=2
请求 4 → VM-C require("counter.js")
  → 未命中缓存，创建 moduleC{exports:{inc,get}}, countC=0 → inc() → countC=1
```

结果：3 个 VM 有 3 组独立的闭包状态，计数器值分别为 2、1、1。用户看到的值取决于请求被调度到哪个 VM。

---

## 四、闭包状态与 VM 的关联

### 4.1 闭包的绑定位置

模块源码被包装为：

```js
(function(exports, require, module, __filename, __dirname) {
    // 用户代码开始
    let count = 0;                   // (1) 在 Runtime 的栈/堆上创建变量
    module.exports = {
        inc: () => ++count,          // (2) 闭包捕获 count
        get: () => count             // (3) 闭包捕获 count
    };
    // 用户代码结束
})
```

当 `r.runtime.RunProgram(prg)` 执行时：
- `let count = 0` 在 **当前 Runtime 的 JS 堆**中分配
- 闭包 `() => ++count` 捕获该 Runtime 内的 `count` 引用
- 闭包函数对象本身也存储在该 Runtime 的堆中

因此，即使 `prg`（字节码）是共享的，每次在不同 Runtime 中执行都会产生**独立的变量实例和独立的闭包**。

### 4.2 闭包状态无法跨 VM 传递

因为 goja 的 JS 值（`goja.Value`、`goja.Object`）内部持有指向所属 Runtime 的指针，它们**不能跨 Runtime 使用**。如果尝试把 VM-A 的闭包或对象传给 VM-B，goja 会 panic。

这也是为什么 module 缓存必须是 per-VM 的——缓存的 `*goja.Object` 是 Runtime 绑定的，不能跨 VM 共享。

### 4.3 与 require 系统相关的闭包链

每次 require 形成的完整闭包链：

```
RequireModule (Go)
  └─ runtime (goja.Runtime)
      └─ modules map
          └─ "/path/to/m.js" → *goja.Object (module)
              ├─ exports: *goja.Object
              │   └─ "inc" → *goja.Object (function)
              │       └─ [[Environment]] → LexicalEnvironment
              │           └─ count: 1 (该 Runtime 堆上的 JS 值)
              └─ ...
```

整条链上的所有 JS 对象都归属于同一个 `goja.Runtime`，不可迁移。

---

## 五、PocketBase 场景下的具体表现

### 5.1 Loader VM 与 Executor VM 的隔离

PocketBase 的 Loader VM 和 Executor Pool 共享同一个 `require.Registry`（编译缓存），但各自有独立的 `RequireModule`（module 缓存）：

```go
// jsvm.go:285-312
requireRegistry := new(require.Registry)   // 进程级共享

sharedBinds := func(vm *goja.Runtime) {
    requireRegistry.Enable(vm)   // 每个 VM 都创建独立的 RequireModule
    // ...
}
```

这意味着：
- Loader VM 顶层 `require("./config.js")` 产生的 module 实例只存在于 Loader VM
- Executor VM 回调中 `require("./config.js")` 会重新加载并执行模块代码，产生独立的 module 实例
- 两者互不影响（除了共享已编译的 `*goja.Program`）

### 5.2 Loader VM 顶层 require 对 Executor VM 的影响

如果 Loader VM 顶层 require 了某个模块（例如为了获取工具函数在顶层注册 Hook）：

```js
// pb_hooks/utils.pb.js
const validator = require("./lib/validator.js");  // Loader VM 内执行

onRecordCreate("posts", (e) => {
    const v = require("./lib/validator.js");  // Executor VM 内重新 require
    v.validate(e.record);
    return e.next();
});
```

- Loader VM 内的 `validator` 和 Executor VM 内的 `validator` 是**独立的 module 实例**
- 但它们共用同一份编译过的 Program（避免重复解析源码）
- 如果 `validator.js` 顶层有副作用（如修改数据库），副作用会执行**至少两次**（一次在 Loader VM，N 次在 N 个首次 require 它的 Executor VM）

### 5.3 池化 VM 复用对 require 缓存的影响

[vmsPool.run()](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/pool.go#L38-L73) 在 VM 归还时**只重置 `busy` 标志**，不清理任何状态：

```go
execErr := call(freeItem.vm)
// "free" the vm
freeItem.mux.Lock()
freeItem.busy = false   // 仅标记为空闲
freeItem.mux.Unlock()
```

因此，`RequireModule.modules` 缓存和闭包状态在 VM 的整个生命周期内（从创建到进程退出）持续累积。这既是性能优化（避免重复 require），也是状态泄漏的来源。

### 5.4 require 模块的状态泄漏场景

```js
// pb_hooks/lib/db.js
const cache = {};  // 模块级缓存，per-VM 独立

module.exports = {
    get(key) { return cache[key]; },
    set(key, val) { cache[key] = val; },
};
```

```js
// pb_hooks/hooks.pb.js
onRecordCreate("posts", (e) => {
    const db = require("./lib/db.js");
    db.set("lastPost", e.record.get("title"));
    return e.next();
});

onRecordViewRequest("posts", (e) => {
    const db = require("./lib/db.js");
    const last = db.get("lastPost");  // 值取决于请求是否落到同一个 VM
    // ...
    return e.next();
});
```

如果请求 A（创建帖子）由 VM-3 处理，请求 B（读取 lastPost）也恰好由 VM-3 处理，则能读到值；否则返回 `undefined`。

**规避方式**：
- 使用 `$app.store()`（进程级、线程安全）替代模块内缓存
- 或确保 require 模块是**纯函数**（无内部可变状态）

---

## 六、三层缓存边界总结表

| 缓存层级 | 存储位置 | 共享范围 | 生命周期 | 线程安全 |
|----------|----------|----------|----------|----------|
| 源码编译缓存 `compiled map[path]*Program` | Registry（进程级） | 所有 VM 共享 | 进程启动到退出 | ✅ `sync.Mutex` 保护 |
| Module 对象缓存 `modules map[path]*Object` | RequireModule（per-VM） | 当前 VM 内部 | VM 创建到进程退出（池化 VM 永不清理） | ❌ VM 单线程执行无并发 |
| 闭包状态（模块内 let/var/const） | 各 Runtime 的 JS 堆 | 当前 module 实例内 | 与所在 module 对象同生命周期 | ❌ 同上 |
| Native 模块注册 `native map[path]ModuleLoader` | Registry（进程级） | 所有 VM 共享 | 进程启动到退出 | ✅ `sync.Mutex` 保护 |

---

## 七、关键结论

1. **编译缓存（`*goja.Program`）是唯一真正全局共享的**，它保证了同一 JS 文件在进程内只被 parse/compile 一次。

2. **Module 对象（含 exports）是 per-VM 的**。不同 VM require 同一 JS 文件会得到独立的 exports 对象和独立的闭包状态。

3. **池化 VM 复用导致 require 缓存持久化**。同一 VM 处理的多个请求之间共享 `RequireModule.modules`，模块内可变状态会在请求间泄漏。

4. **Loader VM require 与 Executor VM require 完全隔离**（除编译缓存外）。不要依赖 Loader VM 顶层 require 的副作用影响运行时行为。

5. **require 模块的最佳实践**：
   - 纯函数/工具模块：无状态，安全无隐患
   - 有状态模块：必须假设状态是 per-VM 的，不能用于跨请求共享
   - 跨请求共享：使用 `$app.store()`（Go 层 Store）或数据库
