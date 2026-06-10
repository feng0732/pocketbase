# PocketBase jsvm require 相对路径解析与缓存 key 深度分析

本文沿着代码追踪 `defaultScriptPath`、`RunScript`、`MustCompile`、`getCurrentModulePath`、`__hooks` 五个关键要素，说明 Loader VM 与 Executor VM 加载同一模块时的路径解析差异、缓存命中逻辑，以及由此产生的"看似意外，实则必然"的行为。

---

## 一、defaultScriptPath：一切路径解析的锚点

### 1.1 定义与初始化

[defaultScriptPath](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/jsvm.go#L41-L57) 的完整定义：

```go
var defaultScriptPath = "pb.js"

func init() {
    // For backward compatibility and consistency with the Go exposed
    // methods that operate with relative paths (e.g. `$os.writeFile`),
    // we define the "current JS module" as if it is a file in the current working directory
    // (the filename itself doesn't really matter and in our case the hook handlers are executed as separate "programs").
    //
    // This is necessary for `require(module)` to properly traverse parents node_modules (goja_nodejs#95).
    cwd, err := os.Getwd()
    if err != nil {
        color.Yellow("Failed to retrieve the current working directory: %v", err)
    } else {
        defaultScriptPath = filepath.Join(cwd, defaultScriptPath)
    }
}
```

**关键事实**：
- 最终值形如：`D:\fz\0601\solo-dogfeeding\code\160-pocketbase\pb.js`
- `pb.js` 是一个**虚拟文件名**——磁盘上并不存在这个文件
- 它的目录部分（即 `cwd`）是所有 JS 代码相对路径的**统一解析基准**
- 注释明确说明：这样设计是为了让 `require()` 能正确向上遍历查找 `node_modules/`

### 1.2 为什么需要虚拟锚点文件

Node.js 的 `require()` 算法在解析时，会从**当前模块所在目录**开始，逐级向上查找 `node_modules/`。如果 goja_nodejs 无法确定一个"当前模块路径"，它就不知道从哪个目录开始向上遍历。

PocketBase 的 Hook 脚本并非真正的 JS 模块文件（它们通过字符串传入，不是从磁盘加载），因此必须**伪造**一个模块文件路径作为起点，而 `{cwd}/pb.js` 就是这个伪造的起点。

---

## 二、getCurrentModulePath：goja_nodejs 的调用栈魔法

### 2.1 实现原理

来自 goja_nodejs `require/module.go`（commit `1f56ff5bcf14`）：

```go
func (r *RequireModule) getCurrentModulePath() string {
    var buf [2]goja.StackFrame
    frames := r.runtime.CaptureCallStack(2, buf[:0])
    if len(frames) < 2 {
        return "."
    }
    return filepath.Dir(frames[1].SrcName())
}
```

`CaptureCallStack(2, buf[:0])` 捕获当前 JS 调用栈，最多 2 帧：
- `frames[0]`：`require()` 函数自身
- `frames[1]`：**调用 require 的那一行代码所在的脚本/函数**
- `frames[1].SrcName()`：该脚本的"源文件名"

`SrcName()` 返回的值正是在 `RunScript(path, ...)`、`MustCompile(path, ...)`、`RunProgram(prg)` 中传入的 `path` 参数。

### 2.2 解析流程总览

```
JS 代码: require("./lib/utils")
    │
    ▼
goja 内部调用 RequireModule.require()
    │
    ▼
getCurrentModulePath()
    │  CaptureCallStack → frames[1].SrcName()
    │  → 返回的是传入 RunScript/MustCompile 的 path
    │  → 例如 "D:/project/pb.js"
    ▼
filepath.Dir(srcName)
    │
    ▼
baseDir = "D:/project"   ← 相对路径将从这里解析
    │
    ▼
resolvePath(baseDir, "./lib/utils")
    │
    ▼
最终绝对路径 = "D:/project/lib/utils.js"
```

---

## 三、Loader VM：RunScript 与 Hook 文件加载

### 3.1 Loader VM 中 Hook 文件的执行方式

[registerHooks()](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/jsvm.go#L328-L347) 中：

```go
for file, content := range files {
    func() {
        // ...
        _, err := loader.RunScript(defaultScriptPath, string(content))
        //                                 ↑↑↑
        //                   注意：无论 file 是 a.pb.js 还是 b.pb.js，
        //                   RunScript 的第一个参数永远是 defaultScriptPath！
    }()
}
```

**极端重要**：即使实际读取的文件是 `pb_hooks/posts.pb.js`、`pb_hooks/users.pb.js` 等不同文件，`RunScript` 的 `path` 参数**永远是 `defaultScriptPath`**（即 `{cwd}/pb.js`）。

### 3.2 对 Loader VM 中 require 的影响

假设项目结构：

```
D:/project/
├── pb_hooks/
│   ├── main.pb.js
│   └── lib/
│       └── utils.js
└── node_modules/
    └── lodash/
```

在 `pb_hooks/main.pb.js` 顶层执行时写：

```js
// pb_hooks/main.pb.js 顶层代码
const utils = require("./lib/utils");    // (1)
const lodash = require("lodash");        // (2)
```

**解析过程 (1)**：
- `frames[1].SrcName()` = `D:/project/pb.js`（来自 RunScript 的第一个参数）
- `baseDir` = `D:/project`
- 解析结果 = `D:/project/lib/utils.js`（**不是** `D:/project/pb_hooks/lib/utils.js`）
- **文件不存在，报错**

**解析过程 (2)**：
- `baseDir` = `D:/project`
- 从 `D:/project/node_modules/lodash` 查找
- **成功**（因为 node_modules 就在 cwd 下）

### 3.3 Loader VM 中 require pb_hooks 内模块的正确写法

由于相对路径 `./` 锚定在 `{cwd}` 而非 `pb_hooks/`，用户有两种方式正确引用 Hook 目录内的模块：

**方式 A：相对 cwd 写路径**
```js
const utils = require("./pb_hooks/lib/utils");  // 显式写 pb_hooks/
```

**方式 B：使用 `__hooks` 变量**
```js
const utils = require(__hooks + "/lib/utils");  // __hooks 是绝对路径
```

---

## 四、`__hooks` 变量：PocketBase 提供的显式锚点

### 4.1 定义与注入时机

在 [registerHooks()](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/jsvm.go#L273-L276) 中计算，通过 [sharedBinds()](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/jsvm.go#L288-L312) 注入每个 VM：

```go
absHooksDir, err := filepath.Abs(p.config.HooksDir)  // "D:/project/pb_hooks"
// ...
sharedBinds := func(vm *goja.Runtime) {
    // ...
    vm.Set("$template", templateRegistry)
    vm.Set("__hooks", absHooksDir)   // 注入为字符串常量
    // ...
}
```

`__hooks` 的值是**绝对路径字符串**，指向 `pb_hooks/` 目录。

### 4.2 `__hooks` 的双重作用

1. **作为 require 路径的前缀**：
   ```js
   require(__hooks + "/lib/validator.js")
   // → require("D:/project/pb_hooks/lib/validator.js")
   ```
   这让用户可以**精确**定位到 Hook 目录下的模块，无需关心 defaultScriptPath 指向哪里。

2. **作为文件操作的路径基准**：
   ```js
   const data = $os.readFile(__hooks + "/data/config.json");
   ```
   与 Go 侧暴露的 `$os.*` 方法配合使用（这些 Go 方法本身使用 `filepath.Join` 处理路径）。

### 4.3 Migrations VM 中的一致性

Migrations 使用独立 VM（不共享 Hooks 的 `requireRegistry`），但同样注入 `__hooks`：

```go
// registerMigrations():
vm.Set("$template", templateRegistry)
vm.Set("__hooks", absHooksDir)
```

注意 Migrations VM 的 `defaultScriptPath` 与 Hooks 相同，因此相对路径解析行为一致——`require("./foo")` 也从 cwd 解析。

---

## 五、Executor VM：MustCompile、RunProgram 与回调执行

### 5.1 回调代码的编译路径

Hook 回调在注册时（Loader VM 内）被预编译为 `*goja.Program`。以 [`hooksBinds()`](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/binds.go#L60-L63) 为例：

```go
loader.Set(jsName, func(callback string, tags ...string) {
    // 包装：注入 $app 覆写逻辑
    callback = `function(e) { $app = e.app; return (` + callback + `).call(undefined, e) }`
    // 预编译
    pr := goja.MustCompile(defaultScriptPath, "{("+callback+").apply(undefined, __args)}", true)
    //                                    ↑↑↑
    //                          同样使用 defaultScriptPath 作为路径！
    // ...
})
```

`goja.MustCompile(path, source, strict)` 的第一个参数 `path` 会被嵌入到编译产物 `*goja.Program` 中，作为代码的"源标识"。当 `executor.RunProgram(pr)` 执行时，该 Program 的所有栈帧 `SrcName()` 都返回这个嵌入的路径。

### 5.2 Executor VM 中 require 的解析基准

Executor VM 执行回调时的调用栈：

```
executor.RunProgram(pr)
    │
    └─ { (function(e) { ... }).apply(undefined, __args) }   ← SrcName = {cwd}/pb.js
           │
           └─ 用户回调函数体
                  │
                  └─ require("./lib/utils")   ← getCurrentModulePath() 向上找调用者
                         │
                         └─ frames[1].SrcName() = {cwd}/pb.js
                         │
                         └─ baseDir = {cwd}
```

结论：**Executor VM 中 `require()` 的解析基准与 Loader VM 完全相同**——都是 `{cwd}`。两种 VM 中执行同一行 `require("./foo")`，解析出的绝对路径完全一致。

### 5.3 其他路径下的编译锚点

PocketBase 中所有预编译回调都统一使用 `defaultScriptPath`：

| 位置 | 代码 | 路径参数 |
|------|------|----------|
| Hook 注册 | [binds.go:63](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/binds.go#L63) | `defaultScriptPath` |
| Cron Job | [binds.go:106](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/binds.go#L106) | `defaultScriptPath` |
| Router Handler | [binds.go:191](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/binds.go#L191) | `defaultScriptPath` |
| Middleware | [binds.go:243](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/binds.go#L243) | `defaultScriptPath` |
| Middleware (string) | [binds.go:265](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/binds.go#L265) | `defaultScriptPath` |

这是有意为之的统一设计，确保无论 JS 代码在 Loader VM 顶层执行还是在 Executor VM 中作为回调执行，相对路径解析行为保持一致。

---

## 六、缓存 key 的构成：绝对路径是唯一标识

### 6.1 编译缓存 key

来自 goja_nodejs `Registry.getCompiledSource()`：

```go
func (r *Registry) getCompiledSource(p string) (*goja.Program, error) {
    r.Lock()
    defer r.Unlock()

    prg := r.compiled[p]   // key = 绝对路径字符串 p
    // ...
    r.compiled[p] = prg
    return prg, nil
}
```

`r.compiled` 的 key 是**已解析的绝对路径**（传入 `getCompiledSource` 之前已通过 `resolvePath` 归一化）。

### 6.2 Module 缓存 key

来自 goja_nodejs `RequireModule.loadModule()`：

```go
func (r *RequireModule) loadModule(path string) (*goja.Object, error) {
    module := r.modules[path]   // key = 同上，已解析的绝对路径
    // ...
    r.modules[path] = module
    return module, nil
}
```

两层缓存使用**完全相同的 key**——解析后的绝对路径字符串。

### 6.3 路径归一化

`resolvePath` 在 Windows 上会做驱动器字母大小写归一化、斜杠方向归一化等处理，确保：
- `D:\project\lib\utils.js`
- `d:/project/lib/utils.js`
- `D:/project/./lib/utils.js`

最终都归一化为同一字符串，保证缓存命中。

---

## 七、Loader VM 与 Executor VM 加载同一模块的完整对比

假设项目结构：
```
D:/project/
├── pb_hooks/
│   ├── main.pb.js
│   └── lib/
│       └── validator.js
└── node_modules/
    └── lodash/
```

用户在 `pb_hooks/main.pb.js` 中写：

```js
// Loader VM 顶层执行
const v1 = require(__hooks + "/lib/validator");

onRecordCreate("posts", (e) => {
    // Executor VM 内执行
    const v2 = require(__hooks + "/lib/validator");
    const v3 = require(__hooks + "/lib/validator");
    return e.next();
});
```

### 7.1 时间线分析

```
T1: Loader VM 顶层执行 require(__hooks + "/lib/validator")
    │
    ├─ resolvePath: __hooks 是 "D:/project/pb_hooks"
    │              → 拼接得 "D:/project/pb_hooks/lib/validator.js"
    │
    ├─ Registry.compiled 查无此 key
    │   → 读文件、编译 → 存入 Registry.compiled["D:/project/pb_hooks/lib/validator.js"]
    │
    └─ Loader_RequireModule.modules 查无此 key
        → 新建 module 对象 → 执行 Program → 存入 Loader_RequireModule.modules[...]

T2: 首次 HTTP 请求到达，Executor VM-A 执行 onRecordCreate 回调
    │
    ├─ 回调内 require(__hooks + "/lib/validator")
    │
    ├─ 解析出相同绝对路径 "D:/project/pb_hooks/lib/validator.js"
    │
    ├─ Registry.compiled 命中 ✅（T1 时编译好的 *goja.Program）
    │   → 跳过 parse/compile，直接复用字节码
    │
    └─ ExecutorA_RequireModule.modules 查无此 key
        → 新建独立 module 对象 → 执行 Program → 存入 ExecutorA_RequireModule.modules[...]

T3: 同一回调内第二次 require（v3）
    │
    ├─ 同 ExecutorA_RequireModule.modules 命中 ✅
    │   → 直接返回 T2 时创建的 module.exports
    │
    └─ 共享同一个 module 实例及其闭包状态

T4: 第二个 HTTP 请求到达，Executor VM-B 执行同一回调
    │
    ├─ Registry.compiled 命中 ✅（仍复用 T1 时的编译缓存）
    │
    └─ ExecutorB_RequireModule.modules 查无此 key（不同 VM）
        → 新建又一个独立 module 对象 → 执行 Program → 存入 ExecutorB_RequireModule.modules[...]
```

### 7.2 缓存命中矩阵

| 缓存层级 | T1 Loader VM | T2 Executor VM-A | T3 同 VM-A 二次 require | T4 Executor VM-B |
|----------|-------------|------------------|------------------------|------------------|
| Registry.compiled（编译缓存） | ❌ → 编译后存入 | ✅ 命中 | ✅ 命中 | ✅ 命中 |
| per-VM RequireModule.modules | ❌ → 加载后存入 | ❌ → 加载后存入 | ✅ 命中 | ❌ → 加载后存入 |
| 闭包状态（模块内变量） | 创建独立副本 | 创建独立副本 | 共享 VM-A 的副本 | 创建又一个独立副本 |

### 7.3 关键结论

1. **编译缓存 100% 共享**：Loader VM 首次 require 触发编译后，所有 Executor VM 的所有后续 require 都复用字节码，parse/compile 成本只付一次。

2. **Module 对象与闭包 per-VM 独立**：N 个 Executor VM + 1 个 Loader VM 会产生最多 N+1 份独立的 module 实例和闭包状态。

3. **路径解析完全一致**：由于 Loader 和 Executor 都以 `{cwd}/pb.js` 为 SrcName 锚点，同一相对路径在两类 VM 中解析出的绝对路径完全相同，编译缓存必定命中。

4. **`__hooks` 是最可靠的写法**：它绕开了"相对路径基准在哪里"的所有歧义，直接使用绝对路径，在任何 VM、任何执行上下文中结果一致。

---

## 八、路径解析与缓存全景图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              进程级共享                                        │
│                                                                              │
│  require.Registry                                                             │
│    └─ compiled map["D:/project/pb_hooks/lib/validator.js"] → *goja.Program    │
│       ▲                                                                      │
│       │ 所有 VM 共享                                                           │
│       │                                                                      │
├───────┼──────────────────────────────────────────────────────────────────────┤
│       │                    VM 级（per-Runtime 独立）                           │
│       │                                                                      │
│  Loader VM              Executor VM-A            Executor VM-B               │
│    │                      │                        │                         │
│    ├─ RequireModule       ├─ RequireModule          ├─ RequireModule          │
│    │   └─ modules         │   └─ modules            │   └─ modules            │
│    │       └─ [abs_path]  │       └─ [abs_path]    │       └─ [abs_path]     │
│    │          → module_L  │          → module_A    │          → module_B     │
│    │                      │                        │                         │
│    └─ getCurrentModule    └─ getCurrentModule      └─ getCurrentModule       │
│         │ SrcName =            │ SrcName =              │ SrcName =          │
│         │ {cwd}/pb.js          │ {cwd}/pb.js            │ {cwd}/pb.js        │
│         │                      │                        │                    │
│         └─ baseDir = cwd       └─ baseDir = cwd         └─ baseDir = cwd     │
│                                                                              │
│  所有 VM 注入:                                                                 │
│    __hooks = "D:/project/pb_hooks"  (字符串常量)                               │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 九、常见陷阱与最佳实践

| 陷阱 | 现象 | 正确做法 |
|------|------|----------|
| `require("./lib/utils")` 找不到文件 | 相对 `{cwd}` 而非 `pb_hooks/` 解析 | 用 `require(__hooks + "/lib/utils")` 或 `require("./pb_hooks/lib/utils")` |
| 以为 Loader VM 顶层 require 会影响 Executor VM | 两者 module 缓存独立，各有各的实例 | 用 `$app.store()` 共享可变状态 |
| 同一模块在不同请求中状态不一致 | 请求落到不同 Executor VM，读到不同闭包副本 | 纯函数模块无状态；有状态用 `$app.store()` |
| Migrations 中 require 不到 Hook 目录模块 | Migrations VM 的 defaultScriptPath 也是 cwd，相对路径行为一致 | Migrations 同样可用 `__hooks + "/..."` 引用 pb_hooks 下模块 |
| 迁移文件中 `require("../pb_hooks/...")` 行为异常 | 相对路径基准是 cwd 而非迁移文件所在目录 | 使用绝对路径或 `__hooks` |
