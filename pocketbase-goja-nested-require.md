# PocketBase jsvm 嵌套 require 路径解析深度分析

本文追踪被 `require()` 加载的模块内部再次 `require()` 时的完整路径解析流程，覆盖 `loadModuleFile`、模块包装函数、`__filename` / `__dirname`、`getCurrentModulePath` 的调用栈变化，以及与缓存 key、`__hooks` 的交互关系。

核心发现：**嵌套 require 的路径基准与顶层 require 完全不同。顶层 require 锚定在 `{cwd}/pb.js`，而模块内部 require 锚定在该模块文件的真实绝对路径**——这与 Node.js 的标准行为一致。

---

## 一、模块包装函数：`__filename` 与 `__dirname` 的注入方式

### 1.1 源码包装阶段

goja_nodejs 在 `Registry.getCompiledSource()`（`require/module.go`）中将用户 JS 源码包装为 CommonJS 模块函数：

```go
// goja_nodejs require/module.go
source := "(function(exports,require,module,__filename,__dirname){" + s + "\n})"
parsed, err := goja.Parse(p, source, parser.WithSourceMapLoader(r.srcLoader))
prg, err := goja.CompileAST(parsed, false)
```

注意：`goja.Parse(p, source, ...)` 的第一个参数 `p` 是模块文件的**绝对路径**。这个 `p` 被嵌入到 AST 中，作为该段代码所有栈帧的 `SrcName()`。

### 1.2 执行阶段：参数注入

`RequireModule.loadModuleFile()`（`require/resolve.go`）拿到编译后的 `*goja.Program` 后，在当前 Runtime 内执行并传入参数：

```go
// goja_nodejs require/resolve.go
func (r *RequireModule) loadModuleFile(path string, jsModule *goja.Object) error {
    prg, err := r.r.getCompiledSource(path)    // path = 模块文件的绝对路径
    f, err := r.runtime.RunProgram(prg)
    if call, ok := goja.AssertFunction(f); ok {
        jsExports := jsModule.Get("exports")
        jsRequire := r.runtime.Get("require")

        // 以参数形式传入 __filename 和 __dirname
        _, err = call(
            jsExports,              // arguments[0] = exports
            jsRequire,              // arguments[1] = require
            jsModule,               // arguments[2] = module
            r.runtime.ToValue(path),                    // arguments[3] = __filename（绝对路径）
            r.runtime.ToValue(filepath.Dir(path)),      // arguments[4] = __dirname（绝对路径的目录部分）
        )
    }
    return nil
}
```

**关键事实**：
- `__filename` = 模块文件的绝对路径（如 `D:/project/pb_hooks/lib/validator.js`）
- `__dirname` = 该模块所在目录（如 `D:/project/pb_hooks/lib`）
- 两者是**函数参数**，不是全局变量，但在模块函数作用域内可直接访问
- 它们的值完全由被加载模块的真实路径决定，与 `defaultScriptPath` 无关

---

## 二、嵌套 require 时 `getCurrentModulePath` 的调用栈分析

### 2.1 `getCurrentModulePath` 的实现回顾

```go
// goja_nodejs require/module.go
func (r *RequireModule) getCurrentModulePath() string {
    var buf [2]goja.StackFrame
    frames := r.runtime.CaptureCallStack(2, buf[:0])
    if len(frames) < 2 {
        return "."
    }
    return filepath.Dir(frames[1].SrcName())
}
```

核心逻辑：捕获调用栈的**第 2 帧**（`frames[1]`），取其 `SrcName()` 的目录部分。第 0 帧是 `require()` 函数自身，第 1 帧是**调用 `require()` 的那行代码所在的位置**。

### 2.2 顶层 require 的调用栈（锚定在 defaultScriptPath）

场景：在 `pb_hooks/main.pb.js` 顶层代码中写 `require("./foo")`，由 Loader VM 执行。

执行入口：[registerHooks()](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/jsvm.go#L342)

```go
_, err := loader.RunScript(defaultScriptPath, string(content))
//                                 ↑↑↑ = {cwd}/pb.js
```

调用栈：

```
帧 0: RequireModule.require()        SrcName = 内部（goja_nodejs 实现）
帧 1: 用户顶层代码中的 require() 调用  SrcName = {cwd}/pb.js  ← 来自 RunScript 的第一个参数
       └─ 这一行的"所属文件"被认为是 {cwd}/pb.js
```

结果：
- `frames[1].SrcName()` = `D:/project/pb.js`（虚拟文件）
- `filepath.Dir(...)` = `D:/project`（即 `cwd`）
- 相对路径 `./foo` 从 `D:/project` 解析 → 找 `D:/project/foo.js`

### 2.3 嵌套 require 的调用栈（锚定在模块真实路径）

场景：`pb_hooks/main.pb.js` 顶层 `require(__hooks + "/lib/validator")`，而 `validator.js` 内部又写了 `require("./helper")`。

执行入口链：

```
Loader VM: RunScript(defaultScriptPath, main.pb.js 源码)
  │
  └─ 顶层执行 require("D:/project/pb_hooks/lib/validator.js")
        │
        └─ loadModuleFile("D:/project/pb_hooks/lib/validator.js", moduleObj)
              │
              ├─ RunProgram(validator 的包装函数)
              │     → 包装函数的 SrcName = "D:/project/pb_hooks/lib/validator.js"
              │
              └─ validator.js 内部执行 require("./helper")
                    │
                    └─ getCurrentModulePath() 捕获调用栈
```

此时的调用栈：

```
帧 0: RequireModule.require()
帧 1: validator.js 中的 require("./helper") 调用
       └─ SrcName = "D:/project/pb_hooks/lib/validator.js"  ← 模块真实路径！
          （因为 validator.js 编译时 goja.Parse() 传入的 p 就是这个绝对路径）
```

结果：
- `frames[1].SrcName()` = `D:/project/pb_hooks/lib/validator.js`
- `filepath.Dir(...)` = `D:/project/pb_hooks/lib`（validator.js 所在目录）
- 相对路径 `./helper` 从 `D:/project/pb_hooks/lib` 解析 → 找 `D:/project/pb_hooks/lib/helper.js`

**这与 Node.js 标准行为完全一致**。

### 2.4 多层嵌套的递归行为

假设：
- `validator.js` require `./helper`
- `helper.js` require `./utils/string`
- `string.js` require `../config`

每层的 SrcName 和解析基准：

| require 调用位置 | frames[1].SrcName() | 解析基准 baseDir | `./xxx` 解析结果 |
|------------------|---------------------|------------------|-----------------|
| main.pb.js 顶层 | `D:/project/pb.js`（虚拟） | `D:/project` | `D:/project/xxx` |
| validator.js 内 | `D:/project/pb_hooks/lib/validator.js` | `D:/project/pb_hooks/lib` | `D:/project/pb_hooks/lib/xxx` |
| helper.js 内 | `D:/project/pb_hooks/lib/helper.js` | `D:/project/pb_hooks/lib` | `D:/project/pb_hooks/lib/xxx` |
| string.js 内 | `D:/project/pb_hooks/lib/utils/string.js` | `D:/project/pb_hooks/lib/utils` | `D:/project/pb_hooks/lib/utils/xxx` |

`../config` 从 `utils/` 向上解析到 `D:/project/pb_hooks/lib/config.js`，与 Node.js 行为一致。

---

## 三、顶层 require vs 嵌套 require：路径基准对比

### 3.1 一张表说明差异

| 维度 | 顶层 require（Hook 文件顶层/回调内） | 嵌套 require（模块内部） |
|------|------------------------------------|------------------------|
| SrcName 来源 | `RunScript`/`MustCompile` 的第一个参数 = `defaultScriptPath` | `goja.Parse(p, ...)` 的 `p` = 模块文件的**真实绝对路径** |
| SrcName 值 | `{cwd}/pb.js`（虚拟文件） | 如 `{cwd}/pb_hooks/lib/validator.js`（真实文件） |
| 解析基准 baseDir | `{cwd}` | **当前被加载模块所在目录** |
| 相对 `./foo` 解析到 | `{cwd}/foo.js` | `当前模块目录/foo.js` |
| `__filename` 可用性 | ❌（顶层代码不在模块函数作用域内） | ✅（作为模块函数参数注入） |
| `__dirname` 可用性 | ❌ | ✅ |
| `__hooks` 可用性 | ✅（全局变量） | ✅（全局变量） |

### 3.2 为什么顶层 require 的 SrcName 不是真实文件名

PocketBase 在 Loader VM 中加载所有 Hook 文件时：

```go
// jsvm.go:342
_, err := loader.RunScript(defaultScriptPath, string(content))
```

无论实际加载的是 `a.pb.js` 还是 `b.pb.js`，`RunScript` 的第一个参数始终是 `defaultScriptPath`（虚拟的 `{cwd}/pb.js`）。这是故意设计的，目的是让所有 Hook 文件的顶层 `require()` 都从 `{cwd}` 基准解析，避免因文件顺序不确定导致的路径歧义。

**但副作用是**：如果用户在 `pb_hooks/main.pb.js` 顶层写 `require("./lib/validator")`，会找 `{cwd}/lib/validator.js`（通常不存在），而不是 `{cwd}/pb_hooks/lib/validator.js`。

### 3.3 回调内 require 的行为

Hook 回调在 Executor VM 中执行时：

```go
// binds.go:63
pr := goja.MustCompile(defaultScriptPath, "{("+callback+").apply(undefined, __args)}", true)
// binds.go:84
res, err := executor.RunProgram(pr)
```

预编译和执行都使用 `defaultScriptPath` 作为路径标识。因此**回调内的顶层 require 与 Hook 文件顶层的 require 行为完全相同**——基准都是 `{cwd}`。

但如果回调内 `require("模块A")`，然后模块 A 内部再 `require("./模块B")`，则从模块 A 的真实目录解析。

---

## 四、`__hooks` 在嵌套 require 中的可用性与行为

### 4.1 `__hooks` 的注入方式

`__hooks` 是通过 [sharedBinds()](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/jsvm.go#L307) 注入每个 VM 的全局变量：

```go
sharedBinds := func(vm *goja.Runtime) {
    // ...
    vm.Set("__hooks", absHooksDir)  // absHooksDir = filepath.Abs(p.config.HooksDir)
    // ...
}
```

`vm.Set()` 写入的是 Runtime 的 `globalThis`，因此在所有 JS 代码中均可访问，包括：
- Hook 文件顶层代码（Loader VM 中）
- Hook 回调函数体（Executor VM 中）
- **任何被 require 加载的模块内部**（因为它们也在同一个 Runtime 中执行）

### 4.2 模块内部使用 `__hooks` 的效果

在 `pb_hooks/lib/validator.js` 中：

```js
// validator.js 内部
const config = require(__hooks + "/config/app.json");
```

解析过程：
- `__hooks` = `"D:/project/pb_hooks"`（从 globalThis 读取）
- 字符串拼接得到 `"D:/project/pb_hooks/config/app.json"`
- 绝对路径直接作为 key 查询 `Registry.compiled` 和 `RequireModule.modules`
- 不经过 `resolvePath` 的相对路径归一化，**结果与当前模块位置无关**

使用 `__hooks` 的优势：
- 路径确定性：无论从哪个模块、哪个 VM、哪个执行阶段调用，结果完全一致
- 不依赖调用栈捕获（`CaptureCallStack` 有性能开销，且在极端情况下可能栈深度不足）
- 语义清晰：明确指向 Hook 目录

### 4.3 `__dirname` 与 `__hooks` 的对比

```js
// validator.js 位于 D:/project/pb_hooks/lib/

// 写法 A: 使用 __dirname
const helper = require(__dirname + "/helper");  // → D:/project/pb_hooks/lib/helper.js
// 依赖 validator.js 的位置；如果未来移动文件，路径自动正确

// 写法 B: 使用 __hooks
const helper = require(__hooks + "/lib/helper"); // → D:/project/pb_hooks/lib/helper.js
// 显式锚定到 pb_hooks 根目录；移动文件后路径可能需要同步更新
```

两者等价但语义不同：
- `__dirname` 适合引用**同一模块子树内**的文件（符合 Node.js 惯例）
- `__hooks` 适合引用**相对于 pb_hooks 根目录**的文件（跨子树共享模块时更清晰）

---

## 五、缓存 key 在嵌套场景下的一致性

### 5.1 两种路径，同一条缓存 key

以下三种写法，虽然语法和解析过程完全不同，但最终都解析到同一个绝对路径，因此命中同一份缓存：

```js
// 位于 pb_hooks/lib/validator.js 内
// 目标文件：pb_hooks/lib/utils/string.js

// 写法 1：相对路径（Node.js 惯例）
const s1 = require("./utils/string");
// 基准 = D:/project/pb_hooks/lib
// 解析 = D:/project/pb_hooks/lib/utils/string.js

// 写法 2：使用 __dirname（等价于写法 1）
const s2 = require(__dirname + "/utils/string");
// → D:/project/pb_hooks/lib/utils/string.js

// 写法 3：使用 __hooks
const s3 = require(__hooks + "/lib/utils/string");
// → D:/project/pb_hooks/lib/utils/string.js
```

三种写法解析出的绝对路径完全一致，因此：
- `Registry.compiled["D:/project/pb_hooks/lib/utils/string.js"]` 命中同一份编译缓存 ✅
- 同一 VM 内的 `RequireModule.modules["D:/project/pb_hooks/lib/utils/string.js"]` 命中同一份 module 对象 ✅

### 5.2 跨 VM 缓存行为回顾

即使路径相同，不同 VM 的 module 缓存仍然独立：

```
Loader VM require("D:/project/pb_hooks/lib/validator.js")
  → 触发编译（如果首次）→ 存入 Registry.compiled[key]
  → 创建 module_L → 存入 Loader_RequireModule.modules[key]

Executor VM-A require(同一 key)
  → 编译缓存命中 ✅（复用 Loader VM 的编译结果）
  → 创建 module_A → 存入 ExecutorA_RequireModule.modules[key]

Executor VM-A 的 validator.js 内 require(同一 key，循环引用检测)
  → ExecutorA_RequireModule.modules[key] 命中 ✅（返回 module_A，防止死循环）

Executor VM-B require(同一 key)
  → 编译缓存仍命中 ✅
  → 创建 module_B → 存入 ExecutorB_RequireModule.modules[key]（独立实例）
```

goja_nodejs 通过在执行模块代码**之前**就将空的 `module` 对象存入 `RequireModule.modules` 来处理循环引用：

```go
// goja_nodejs require/resolve.go
func (r *RequireModule) loadModule(path string) (*goja.Object, error) {
    module := r.modules[path]
    if module == nil {
        module = r.createModuleObject()  // 先创建 { exports: {} }
        r.modules[path] = module          // 立即存入缓存，标记为"加载中"
        err := r.loadModuleFile(path, module)  // 再执行填充 exports
        // ...
    }
    return module, nil
}
```

如果模块 A require 模块 B，模块 B 又 require 模块 A，当 B require A 时：
- A 已在 `r.modules` 中（状态为"加载中"，`exports` 可能为空或部分填充）
- 直接返回 A 的 module 对象，不会递归加载 → 循环引用被打破

这与 Node.js 的循环引用处理策略一致。

---

## 六、完整路径解析流程图

```
用户代码调用 require(modpath)
    │
    ├─ 是否为绝对路径？（如 "D:/..." 或 "/" 开头）
    │   ├─ 是 → 直接使用 modpath 作为待解析路径，baseDir 无关
    │   └─ 否 → 继续
    │
    ├─ 是否以 "./" 或 "../" 开头？
    │   ├─ 是 → getCurrentModulePath() 捕获调用栈
    │   │        ├─ 帧 0: require() 自身
    │   │        └─ 帧 1: 调用 require 的代码
    │   │                  ├─ 如果在 Hook 顶层/回调内  → SrcName = {cwd}/pb.js   → baseDir = {cwd}
    │   │                  └─ 如果在被 require 的模块内 → SrcName = 模块绝对路径   → baseDir = 模块所在目录
    │   │
    │   │        resolvePath(baseDir, modpath) → 归一化为绝对路径
    │   │
    │   └─ 否（裸模块名，如 "lodash"）→ 从 baseDir 开始逐级向上查找 node_modules/
    │
    ▼
已解析的绝对路径 absPath
    │
    ├─ 查 RequireModule.modules[absPath] （per-VM 缓存）
    │   └─ 命中 → 直接返回 module.exports（可能加载中，用于循环引用）
    │
    ├─ 未命中，查 Registry.compiled[absPath] （全局编译缓存）
    │   ├─ 命中 → 复用 *goja.Program，跳过 parse/compile
    │   └─ 未命中 → 读文件 → 包装为 (function(exports,require,module,__filename,__dirname){...})
    │               → goja.Parse(absPath, source) → CompileAST → 存入 Registry.compiled[absPath]
    │
    ├─ 创建空 module 对象，存入 RequireModule.modules[absPath]（标记为加载中）
    │
    ├─ runtime.RunProgram(prg) → 得到包装函数 f
    │
    ├─ f(exports, require, module, absPath, filepath.Dir(absPath))
    │   │                        ↑__filename    ↑__dirname
    │   │
    │   └─ 用户模块代码执行
    │        └─ 内部再次 require() → 递归进入本流程
    │
    ▼
返回 module.exports
```

---

## 七、常见场景与预期行为

### 场景 1：顶层 require 相对路径找不到文件

```js
// pb_hooks/main.pb.js 顶层
const v = require("./lib/validator");  // ❌ 找 {cwd}/lib/validator.js，通常不存在
```

修正：
```js
const v = require(__hooks + "/lib/validator");  // ✅
const v = require("./pb_hooks/lib/validator");  // ✅（相对 {cwd}）
```

### 场景 2：模块内部 require 同级文件

```js
// pb_hooks/lib/validator.js 内部
const helper = require("./helper");  // ✅ 正确解析为 pb_hooks/lib/helper.js
```

这与 Node.js 行为一致，无需特殊处理。

### 场景 3：模块内部 require pb_hooks 根目录下的共享模块

```js
// pb_hooks/lib/utils/string.js 内，想引用 pb_hooks/shared/config.js
const cfg1 = require(__hooks + "/shared/config");  // ✅ 推荐，语义清晰
const cfg2 = require("../../shared/config");        // ✅ 也正确，但脆弱（移动文件需更新 ../）
```

### 场景 4：回调内 require

```js
onRecordCreate("posts", (e) => {
    // 回调顶层：基准 = {cwd}
    const a = require("./pb_hooks/lib/foo");  // ✅
    const b = require(__hooks + "/lib/foo");  // ✅
    const c = require("./lib/foo");           // ❌ 找 {cwd}/lib/foo.js

    // 但如果 require 的模块内部再次 require，基准变为该模块所在目录
    const validator = require(__hooks + "/lib/validator");
    // validator 内部的 require("./helper") 从 {cwd}/pb_hooks/lib 解析 ✅
    return e.next();
});
```

---

## 八、关键结论汇总

1. **顶层 require 与嵌套 require 的路径基准完全不同**：
   - 顶层（Hook 文件顶层/回调内）：锚定在虚拟的 `{cwd}/pb.js`，基准 = `{cwd}`
   - 嵌套（被 require 加载的模块内部）：锚定在模块真实绝对路径，基准 = 模块所在目录

2. **`__filename` 与 `__dirname` 只在模块内部可用**：它们是 CommonJS 包装函数的参数，顶层代码（不在模块函数作用域内）访问不到。

3. **`__hooks` 在任何位置均可使用**：作为全局变量注入 `globalThis`，无论是顶层代码、回调代码、还是任意深度的嵌套 require 内，都能读到相同的 `pb_hooks/` 绝对路径。使用它可以绕开所有路径基准歧义。

4. **缓存 key 始终是绝对路径**：无论用相对路径、`__dirname`、还是 `__hooks`，只要最终解析出的绝对路径相同，就命中同一份编译缓存和 module 缓存。

5. **循环引用处理**：goja_nodejs 在执行模块代码前就将空 module 对象存入缓存，与 Node.js 行为一致，能正确打破循环依赖。

6. **最稳妥的写法**：
   - 同一模块子树内引用：用相对路径（`./xxx`）或 `__dirname`，符合 Node.js 惯例
   - 跨子树、顶层代码、回调代码：用 `__hooks + "/..."`，路径最确定
