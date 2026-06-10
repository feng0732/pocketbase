# PocketBase goja JS Hooks 沙箱协作机制深度分析

## 一、整体架构概览

PocketBase 使用 [goja](https://github.com/dop251/goja) 作为纯 Go 实现的 JavaScript 引擎，通过 `plugins/jsvm/` 插件将 JS 沙箱与宿主 Go 代码深度整合。整体架构采用 **"双 VM 模型"**：

```
┌──────────────────────────────────────────────────────────────────┐
│                        PocketBase App (Go 宿主)                  │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │  Loader VM (goja.Runtime) — 仅在启动时运行 1 次            │   │
│  │  • 加载 pb_hooks/*.pb.js 脚本文件                           │   │
│  │  • 解析 on*() / routerAdd() / cronAdd() 等注册函数          │   │
│  │  • 将 JS 回调以预编译 Program 形式注入 Go Hook 系统         │   │
│  └───────────────────────────────────────────────────────────┘   │
│                              │                                    │
│                              ▼ 注册 Handler                       │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │  Executor VM Pool (vmsPool) — 并发执行 JS 回调             │   │
│  │  • N 个预热的 goja.Runtime 实例（默认 15）                  │   │
│  │  • 请求/Hook 触发时从池中借用 VM，执行后归还                │   │
│  │  • 池满时按需临时创建新 VM                                  │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                  │
│  Shared State (跨 VM 共享):                                       │
│  • require.Registry — 模块缓存                                    │
│  • template.Registry — HTML 模板缓存                              │
│  • core.App 实例引用                                              │
└──────────────────────────────────────────────────────────────────┘
```

核心代码入口：[jsvm.go](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/jsvm.go)

---

## 二、脚本加载机制

### 2.1 两类脚本：Hooks 与 Migrations

JSVM 插件加载两类独立的 JS 脚本，分别走不同的 VM 生命周期：

| 类型 | 目录 | 文件匹配 | VM 模式 |
|------|------|----------|---------|
| Hooks | `pb_hooks/` | `^.*(\.pb\.js|\.pb\.ts)$` | Loader VM + Executor Pool |
| Migrations | `pb_migrations/` | `^.*(\.js|\.ts)$` | 每文件独立一次性 VM |

配置定义见 [Config 结构体](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/jsvm.go#L59-L108)。

### 2.2 文件发现与读取

[`filesContent()`](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/jsvm.go#L537-L570) 负责扫描目录并按正则过滤文件：

```go
func filesContent(dirPath string, pattern string) (map[string][]byte, error) {
    files, _ := os.ReadDir(dirPath)
    var exp *regexp.Regexp
    if pattern != "" {
        exp, _ = regexp.Compile(pattern)
    }
    result := map[string][]byte{}
    for _, f := range files {
        if f.IsDir() || (exp != nil && !exp.MatchString(f.Name())) {
            continue
        }
        raw, _ := os.ReadFile(filepath.Join(dirPath, f.Name()))
        result[f.Name()] = raw
    }
    return result, nil
}
```

注意：只读取目录下的**直接子文件**，不递归子目录。

### 2.3 Migrations 加载流程

[`registerMigrations()`](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/jsvm.go#L183-L234)：

1. 每个迁移文件创建**全新独立 VM**（无池化）
2. 启用 Node.js 兼容模块：`require` / `console` / `process` / `buffer`
3. 注入宿主绑定：`BindCore()`、`BindDbx()`、`BindSecurity()`、`BindOS()` 等
4. 暴露全局 `migrate(up, down)` 函数，JS 调用后注册到 `core.AppMigrations`
5. 调用 `vm.RunScript(defaultScriptPath, content)` 一次性执行
6. 执行完成后该 VM 即被 GC 回收

关键设计：迁移脚本需要直接操作 App（如 `txApp.Save()`），因此每个迁移拥有独立 VM 且直接暴露完整 App 能力。

### 2.4 Hooks 加载流程

[`registerHooks()`](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/jsvm.go#L237-L350) 采用更复杂的双阶段设计：

#### 阶段 1：Loader VM 解析与注册

1. 创建单个 **Loader VM**，注入完整绑定（包括 `hooksBinds` / `cronBinds` / `routerBinds`）
2. 遍历所有 `.pb.js` 文件，依次 `loader.RunScript(defaultScriptPath, content)`
3. JS 顶层代码中调用的 `onRecordCreate()`、`routerAdd()`、`cronAdd()` 等并非立即执行回调，而是：
   - 将 JS 函数源码序列化为字符串
   - 用 `goja.MustCompile()` 预编译为 `*goja.Program`
   - 包装为 Go 闭包，通过反射注册到 `core.App` 的对应 Hook 槽位

以 `onModelUpdate` 为例，[`hooksBinds()`](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/binds.go#L43-L102) 中的核心逻辑：

```go
loader.Set(jsName, func(callback string, tags ...string) {
    // 构造最终执行代码：注入 $app 并调用用户回调
    callback = `function(e) { $app = e.app; return (` + callback + `).call(undefined, e) }`
    pr := goja.MustCompile(defaultScriptPath, "{("+callback+").apply(undefined, __args)}", true)

    // 通过反射获取 Go 端 Hook 实例（如 app.OnModelUpdate("demo1")）
    hookInstance := appValue.MethodByName(method.Name).Call(tagsAsValues)[0]
    hookBindFunc := hookInstance.MethodByName("BindFunc")

    // 构造 Go Handler：当事件触发时从 Executor Pool 借 VM 执行 Program
    handler := reflect.MakeFunc(handlerType, func(args []reflect.Value) (results []reflect.Value) {
        handlerArgs := make([]any, len(args))
        for i, arg := range args {
            handlerArgs[i] = arg.Interface()
        }
        err := executors.run(func(executor *goja.Runtime) error {
            executor.Set("$app", goja.Undefined())
            executor.Set("__args", handlerArgs)
            res, err := executor.RunProgram(pr)
            executor.Set("__args", goja.Undefined())
            if resErr := checkGojaValueForError(app, res); resErr != nil {
                return resErr
            }
            return normalizeException(err)
        })
        return []reflect.Value{reflect.ValueOf(&err).Elem()}
    })

    hookBindFunc.Call([]reflect.Value{handler})
})
```

#### 阶段 2：Executor Pool 并发执行

[`vmsPool`](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/pool.go) 管理 N 个预热的 Executor VM：

```go
type vmsPool struct {
    mux     sync.RWMutex
    factory func() *goja.Runtime
    items   []*poolItem  // 每个 item 带独立互斥锁
}
```

[`pool.run()`](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/pool.go#L38-L73) 的调度策略：

1. 遍历池内所有 item，对每个 item 加锁检查 `busy` 标志
2. 找到空闲 item 后标记 `busy = true`，解锁并执行回调
3. 执行完成后重新标记 `busy = false`
4. **若池全忙**：调用 `factory()` 创建临时 VM，用完即弃（不复用）

默认池大小由 `Config.HooksPoolSize` 控制，示例中配置为 15（见 [main.go](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/examples/base/main.go#L42-L48)）。

### 2.5 defaultScriptPath 的作用

[`init()`](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/jsvm.go#L43-L57) 中将 `defaultScriptPath` 设为 `{cwd}/pb.js`：

```go
cwd, _ := os.Getwd()
defaultScriptPath = filepath.Join(cwd, "pb.js")
```

这个路径是**虚拟的**（文件并不存在），主要用于：
- `vm.RunScript(defaultScriptPath, content)` 时给 goja 一个模块标识
- 让 `require()` 能正确从 cwd 向上查找 `node_modules`（解决 [goja_nodejs#95](https://github.com/dop251/goja_nodejs/issues/95)）
- 编译错误时提供有意义的文件名信息

### 2.6 热重载（HooksWatch）

当 `Config.HooksWatch = true` 时，[`watchHooks()`](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/jsvm.go#L369-L463) 使用 `fsnotify` 监听 `pb_hooks/` 目录变化：

- 去抖时间：50ms
- 非 Windows：调用 `app.Restart()` 重启整个进程（依赖 `execve`）
- Windows：仅打印黄色警告，要求手动重启
- 自动跳过 `.` 开头目录和 `node_modules`

---

## 三、权限边界与安全隔离

### 3.1 绑定暴露面总览

JS 沙箱通过一系列 `Bind*()` 函数选择性暴露 Go 宿主能力。所有绑定函数定义在 [binds.go](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/binds.go)。

| 绑定函数 | 暴露对象 | 能力范围 |
|----------|----------|----------|
| `BindCore()` | 全局 | 基础工具：`sleep`、`toString`、`toBytes`、`arrayOf`、`unmarshal`、各类构造器（`Record`、`Collection`、`DynamicModel`、`Middleware`、`ValidationError`、`Cookie`、`DateTime`、`Timezone` 等 41 项） |
| `BindDbx()` | `$dbx.*` | 数据库查询表达式构造：`exp`、`hashExp`、`and`、`or`、`in`、`like`、`between` 等 15 项 |
| `BindSecurity()` | `$security.*` | 加密与随机：`md5`、`sha256`、`hs256`、`randomString`、`createJWT`、`encrypt`、`decrypt` 等 16 项 |
| `BindOS()` | `$os.*` | **操作系统级**：`cmd`（执行 shell 命令）、`readFile`、`writeFile`、`mkdirAll`、`removeAll`、`exit`、`getenv` 等 20 项 |
| `BindFilepath()` | `$filepath.*` | 路径操作：`join`、`base`、`dir`、`walk`、`glob` 等 15 项 |
| `BindHTTP()` | `$http.*` + `FormData` | HTTP 客户端：`send` 方法（默认 120s 超时） + FormData 构造器 |
| `BindFilesystem()` | `$filesystem.*` | 文件系统抽象：`s3`、`local`、`fileFromPath`、`fileFromURL` 等 6 项 |
| `BindForms()` | 全局 | 表单构造器：`RecordUpsertForm`、`TestEmailSendForm` 等 4 项 |
| `BindMails()` | `$mails.*` | 邮件发送辅助：`sendRecordPasswordReset`、`sendRecordVerification` 等 5 项 |
| `BindApis()` | `$apis.*` + 全局 | HTTP API 中间件与错误：`requireAuth`、`static`、`recordAuthResponse`、`ApiError`、`NotFoundError` 等 8 个全局 + 11 个 `$apis` 项 |

**安全边界注意**：goja 自身是纯内存沙箱（无 syscall 能力），但 **PocketBase 通过 `BindOS()` 主动暴露了完整的操作系统访问**，包括执行任意 shell 命令（`$os.cmd`）和读写任意文件。这意味着 JS 脚本实际上拥有与宿主进程相同的 OS 权限，沙箱仅提供"语言隔离"而非"权限沙箱"。

### 3.2 Node.js 兼容层

每个 VM 启动时启用四个 goja_nodejs 模块：

```go
registry.Enable(vm)    // CommonJS require() 支持
console.Enable(vm)     // console.log / console.error
process.Enable(vm)     // process 对象（argv、env、cwd 等）
buffer.Enable(vm)      // Buffer 类型
```

其中 `require.Registry` 可以跨多个 VM 共享（见 [`registerHooks()`](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/jsvm.go#L285)），实现模块缓存的跨 VM 复用。

### 3.3 字段名映射：FieldMapper

[`FieldMapper`](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/mapper.go) 控制 Go struct 字段/方法在 JS 中的可见命名：

```go
func convertGoToJSName(name string) string {
    if v, ok := nameExceptions[name]; ok {
        return v  // 例如 "OAuth2" → "oauth2"
    }
    // 全大写如 "JSON" → "json"
    if len(name) == totalStartUppercase {
        return strings.ToLower(name)
    }
    // 连续大写如 "JSONField" → "jsonField"
    if totalStartUppercase > 1 {
        return strings.ToLower(name[0:totalStartUppercase-1]) + name[totalStartUppercase-1:]
    }
    // 普通如 "GetField" → "getField"
    if totalStartUppercase == 1 {
        return strings.ToLower(name[0:1]) + name[1:]
    }
    return name
}
```

goja 的 `FieldNameMapper` 机制本身就确保了 **Go 未导出字段（小写开头）在 JS 中完全不可访问**，这是一层天然的属性级隔离。

### 3.4 `$app` 的作用域覆写

在 Hook 回调执行时，系统会动态覆盖全局 `$app`：

```go
// 注册时注入到回调包装中
callback = `function(e) { $app = e.app; return (` + callback + `).call(undefined, e) }`

// 执行前在 Executor VM 中
executor.Set("$app", goja.Undefined())  // 先置 undefined，由 JS 包装内赋值
executor.Set("__args", handlerArgs)
res, err := executor.RunProgram(pr)
```

这意味着：
- Loader VM 中 `$app` 是原始 App 实例
- Executor VM 中每次 Hook 执行时 `$app` 被替换为事件对象上的 `e.app`（可能是事务 App 等上下文特定实例）
- 路由处理器中同样有此覆盖：`executor.Set("$app", e.App)`（见 [`wrapHandlerFunc()`](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/binds.go#L181-L213)）

### 3.5 异常归一化

[`normalizeException()`](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/binds.go#L1071-L1091) 负责将 JS 抛出的值还原为 Go error：

```go
func normalizeException(err error) error {
    jsException, ok := err.(*goja.Exception)
    if !ok {
        return err
    }
    switch v := jsException.Value().Export().(type) {
    case error:
        err = v  // JS 直接 throw 了 Go error 对象
    case map[string]any: // goja.GoError
        if vErr, ok := v["value"].(error); ok {
            err = vErr
        }
    }
    return err
}
```

[`checkGojaValueForError()`](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/binds.go#L1046-L1065) 额外处理回调返回值：若 JS 函数返回了 error 类型值（包括 Promise reject），也将其转为 Go error。

---

## 四、运行上下文（Runtime/VM 生命周期）

### 4.1 Migrations：一次性 VM

每个迁移文件创建独立 VM 的生命周期：

```
registerMigrations()
    │
    ├─ for file, content := range files
    │   ├─ vm := goja.New()                    ← 创建
    │   ├─ registry.Enable(vm) / console.Enable(vm)  ← Node.js 兼容层
    │   ├─ BindCore / BindDbx / ...            ← 宿主绑定
    │   ├─ vm.Set("migrate", ...)              ← 注册 migrate()
    │   ├─ vm.RunScript(defaultScriptPath, string(content))  ← 执行
    │   └─ (vm 超出作用域，等待 GC)             ← 销毁
```

Migrations VM **不共享** Executor Pool，也不参与 Hooks 注册流程。

### 4.2 Hooks：Loader VM + Executor Pool

```
registerHooks()
    │
    ├─ sharedBinds := func(vm) { ... }         ← 绑定闭包
    │
    ├─ executors := newPool(size, func() *goja.Runtime {
    │      executor := goja.New()               ← Executor VM 预热
    │      sharedBinds(executor)                ←   注入绑定
    │      return executor
    │  })
    │
    ├─ loader := goja.New()                     ← Loader VM 创建
    ├─ sharedBinds(loader)
    ├─ hooksBinds(app, loader, executors)       ← 注入 on*() 注册函数
    ├─ cronBinds(app, loader, executors)        ← 注入 cronAdd/Remove
    ├─ routerBinds(app, loader, executors)      ← 注入 routerAdd/Use
    │
    ├─ for file, content := range files
    │   └─ loader.RunScript(defaultScriptPath, string(content))  ← 解析注册
    │
    └─ (loader 保持引用，但不再 Run)             ← Loader VM "冻结"
```

**Executor VM 的请求级生命周期**（以 Hook 触发为例）：

```
HTTP 请求到达 → core.App.OnRecordBeforeCreate 触发
    │
    ├─ executors.run(func(executor *goja.Runtime) error {
    │   ├─ executor.Set("$app", undefined)       ← 清理
    │   ├─ executor.Set("__args", [event])       ← 注入参数
    │   ├─ executor.RunProgram(precompiledPrg)   ← 执行 JS 回调
    │   ├─ executor.Set("__args", undefined)     ← 清理参数
    │   └─ return normalizeException(err)
    └─ })
          │
          ├─ 池有空 → 归还 VM，标记 busy=false
          └─ 池全忙 → 临时 VM 用完即弃
```

### 4.3 跨 VM 共享状态

以下对象在 Loader VM 和所有 Executor VM 之间**共享引用**：

| 共享对象 | 创建位置 | 共享原因 |
|----------|----------|----------|
| `*require.Registry` | [jsvm.go:285](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/jsvm.go#L285) | require() 模块缓存，避免重复加载 |
| `*template.Registry` | [jsvm.go:286](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/jsvm.go#L286) | HTML 模板缓存 |
| `core.App` (p.app) | [jsvm.go:305](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/jsvm.go#L305) | PocketBase 应用实例，所有 DB/API 操作入口 |
| `absHooksDir` | [jsvm.go:307](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/jsvm.go#L307) | `__hooks` 全局变量，JS 中可获取 hooks 目录绝对路径 |

`Config.OnInit(vm)` 回调也在每个 VM 创建时执行，用户可通过它注入**自定义全局变量**。

### 4.4 VM 间状态不共享的部分

以下内容**每个 VM 独立拥有**：

- JS 全局变量（`globalThis` 上的属性）
- JS 闭包、对象引用、Prototype 链
- `goja.Runtime` 内部的堆状态

这就是为什么 Hooks 注册必须在 Loader VM 中完成——Executor VM 池中的每个实例都是"干净"的，不承载任何用户 JS 在顶层定义的变量。用户回调若需要持久化状态，必须通过 `$app.Store()` 等宿主机制而非 JS 全局变量。

---

## 五、宿主与沙箱协作的核心模式

### 5.1 反射式 Hook 绑定

[`hooksBinds()`](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/binds.go#L43-L102) 使用 Go 反射自动扫描 `core.App` 上所有 `On*` 开头的方法（排除 `OnServe`），为每个方法生成一个 JS 全局注册函数。

这种设计的优势：新增 Hook 事件无需修改 JSVM 代码，App 加一个 `OnXxx()` 方法就自动在 JS 中可用。

### 5.2 Program 预编译与延迟执行

```go
// 注册阶段（Loader VM）: 只编译，不执行
pr := goja.MustCompile(defaultScriptPath, "{("+callback+").apply(undefined, __args)}", true)

// 执行阶段（Executor VM）: 用预编译 Program 快速执行
res, err := executor.RunProgram(pr)
```

`goja.Program` 是编译后的字节码表示，可被任意数量的 `goja.Runtime` 重复执行，避免每次触发事件都重新 parse JS 源码。

### 5.3 参数注入：`__args` 约定

由于预编译 Program 无法在编译期捕获动态参数，系统采用"全局临时变量"模式：

```go
executor.Set("__args", handlerArgs)    // 执行前注入
res, err := executor.RunProgram(pr)    // JS 中 (...).apply(undefined, __args) 消费
executor.Set("__args", goja.Undefined())  // 执行后清理，防止泄漏
```

### 5.4 路由与中间件包装

[`routerBinds()`](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/binds.go#L149-L179) 注册的 `routerAdd()` 和 `routerUse()` 并不立即操作路由表，而是将操作包装到 `app.OnServe()` 中延迟执行——因为 `ServeEvent.Router` 在 App 启动后才可用。

[`wrapHandlerFunc()`](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/binds.go#L181-L213) 和 [`wrapMiddlewares()`](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/binds.go#L221-L290) 支持三种 handler 类型：
1. **原生 Go 函数** `func(*core.RequestEvent) error` — 直接透传
2. **Go `*hook.Handler`** — 直接透传
3. **JS 函数字符串 / `goja.FunctionCall`** — 走 Executor Pool 执行

### 5.5 Cron Job 的特殊处理

[`cronBinds()`](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/binds.go#L104-L147) 不仅将 `cronAdd` / `cronRemove` 注入 Loader VM，还额外注入到所有 Executor VM（包括池内已预热的和未来新建的）。这样 JS 回调在执行过程中也能动态增删定时任务。

### 5.6 DynamicModel：运行时生成结构体

[`newDynamicModel()`](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/binds.go#L1194-L1260) 是 JS 与 Go 类型系统协作的典型例子：

1. JS 传入 shape 描述（如 `{name: "", age: 0}`）
2. Go 使用 `reflect.StructOf()` 动态生成 struct 类型
3. 类型按 shape 哈希缓存，避免重复生成
4. 返回指向该动态 struct 的指针，可直接被 dbx 用于 ORM 查询

---

## 六、关键文件索引

| 文件 | 职责 |
|------|------|
| [plugins/jsvm/jsvm.go](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/jsvm.go) | 插件入口、Config、Migrations 注册、Hooks 注册、文件监听、类型声明生成 |
| [plugins/jsvm/binds.go](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/binds.go) | 所有 Bind*() 函数、Hook/路由/Cron 包装、异常归一化、DynamicModel、构造器辅助函数 |
| [plugins/jsvm/pool.go](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/pool.go) | Executor VM 池实现 vmsPool |
| [plugins/jsvm/mapper.go](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/mapper.go) | Go↔JS 命名转换 FieldMapper |
| [plugins/jsvm/form_data.go](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/form_data.go) | FormData 类型（供 $http.send 使用） |
| [plugins/jsvm/internal/types/types.go](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/plugins/jsvm/internal/types/types.go) | TypeScript 类型声明生成器（tygoja） |
| [tools/hook/hook.go](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/tools/hook/hook.go) | Go 侧 Hook 系统核心实现（Handler、Bind、Trigger 链） |
| [core/events.go](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/core/events.go) | 所有事件类型定义（BootstrapEvent、RecordEvent、RequestEvent 等） |
| [examples/base/main.go](file:///d:/fz/0601/solo-dogfeeding/code/160-pocketbase/examples/base/main.go) | 插件集成示例 |
