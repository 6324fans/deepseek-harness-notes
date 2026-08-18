# DeepSeek Harness (DSH) 实现深度解析

> 基于 `@deepseek-ai/dsh@0.1.0-rc.6` 源码逆向分析。所有结论均有代码行号佐证。

---

## 目录

1. [技术栈总览](#1-技术栈总览)
2. [启动与引导](#2-启动与引导)
3. [Cordis 框架内核](#3-cordis-框架内核)
4. [System Prompt 动态组装](#4-system-prompt-动态组装)
5. [动态 Prompt 与缓存优化](#5-动态-prompt-与缓存优化)
6. [工具系统](#6-工具系统)
7. [工具实时组装与缓存](#7-工具实时组装与缓存)
8. [Agent Loop](#8-agent-loop)
9. [会话系统 Session](#9-会话系统-session)
10. [LLM 适配层](#10-llm-适配层)
11. [子代理 Subagent](#11-子代理-subagent)
12. [Goal 长目标系统](#12-goal-长目标系统)
13. [Workflow 工作流编排](#13-workflow-工作流编排)
14. [Web GUI](#14-web-gui)
15. [沙箱与文件系统](#15-沙箱与文件系统)
16. [Skill 系统](#16-skill-系统)
17. [Jobs 后台任务](#17-jobs-后台任务)
18. [完整路线图](#18-完整路线图全部已展开)

---

## 1. 技术栈总览

### 不用 LangGraph / LangChain

在整个 `package.json` 依赖列表和 `node_modules` 树中，**没有任何 langgraph 或 langchain 的引用**。

### 底层基于 Cordis

核心技术栈：
- **`@deepseek-ai/cordis`** (v4.0.1) — Node.js 异步流程控制元框架（腾讯 Cordis 的 DeepSeek 分支）
- **`@deepseek-ai/cordis-plugin-*`** — hmr / include / timer / loader / group
- **`@deepseek-ai/dsh-*`** — DeepSeek 自研的 ~80 个插件包

### 架构路线对比

| 维度 | LangGraph | DSH |
|---|---|---|
| 语言生态 | Python | TypeScript / Node.js |
| 编排模型 | 有向图 / 状态机节点 | Cordis 插件体系 + 事件 waterfall |
| 运行形态 | 库 | CLI + 浏览器 Web GUI 的 agent harness |
| 工具系统 | 图节点 | 插件注册 + 作用域过滤 + 实时组装 |

---

## 2. 启动与引导

### 2.1 CLI 入口

一切始于 `@deepseek-ai/dsh/lib/bin.js`：

```js
// bin.js 第 129-152 行
const invocation = parseDshArgs(process.argv.slice(2), readVersion());
switch (invocation.mode) {
    case "profile":     → runProfile(...)
    case "plugin":      → runPlugin(...)
    case "dump-config": → runDumpConfig(...)
}
```

**重点：launcher 只管自己的几个 flag（`--profile`、`--patch`、`--dump-config`），其余原样透传给被启动的 profile 里的 app 插件解析。**

`dsh --profile web --resume abc` 中，`--resume abc` 是 web app 自己的参数。`dsh web` 是 `--profile web` 的硬编码别名。

### 2.2 分层环境加载

`loadLayeredEnv("dsh")` 建立三层环境快照：

```
process 继承环境（最高优先级）
    ↓ 补充（不覆盖）
project .env（当前工作目录）
    ↓ 补充（不覆盖）
user .env（$DSH_HOME/.env）
```

**重点：bootstrap-only 变量（`PATH`、`HOME`、`NODE_OPTIONS`、`HTTP_PROXY`、`DSH_*`、`XDG_*` 等）只允许从进程环境继承，禁止 .env 设置。** 这是为了防止配置文件劫持进程启动行为。

### 2.3 Profile 概念

Profile = `$DSH_HOME/profiles/<name>/` 目录：
- `package.json` — 声明 `dsh.profile.bundles`（bundle 包名列表）
- `cordis.patch.yml` — 用户自己的 patch 层
- `pnpm-workspace.yaml` — pnpm 配置
- `cordis.yml` — 空根配置（`[]`），仅给 Loader 锚定 baseUrl

Bundle = npm 包，`package.json` 里声明 `"dsh": { "bundle": { "patch": "./cordis.patch.yml" } }`。

### 2.4 Patch 层叠加（核心配置模型）

**整个配置不是"一个 config 文件"，而是"多层 patch 列表叠加在空根上"。**

叠加顺序（后者覆盖前者）：

```
空根 cordis.yml = []
    ↓ 第1层：bundle patches（按 dsh.profile.bundles 顺序）
    ↓ 第2层：profile 的 cordis.patch.yml（用户层）
    ↓ 第3层：home 层 $DSH_HOME/cordis.patch.yml（机器级偏好）
    ↓ 第4层：--patch 命令行 overlays
    ↓ 第5层：telemetry 开关
```

每个 patch 层是 YAML 数组，元素通过 `id` 定位目标行：

```yaml
# 覆盖
- id: sandbox-policy
  config:
    mode: danger-full-access

# 禁用
- id: session-telemetry-otel
  disabled: true

# 插入
- id: some-group
  insert:
    - { id: new-plugin, name: '@deepseek-ai/xxx' }
```

`applyEntryPatches`（`dsh-app-boot/lib/index.js` 第 57-106 行）是唯一叠加算法。dump-config 和实际 boot 用同一个函数，保证 `dsh --dump-config` 输出与实际启动一致。

**`dsh-base` 的 patch 是"一个大 insert"**，列出约 30 个核心插件行（timer、hmr、llm、session、agent、sandbox、approval、credentials 等）。

### 2.5 模块解析

- **双锚点**：bundle 先从 dsh 安装目录解析，再从 profile 目录解析
- **闭包符号链接**：`healProfilesModuleFallback` 对 dsh app 依赖闭包里的每个包建符号链接到 `$DSH_HOME/profiles/node_modules/`，使所有 in-box 插件全局可解析

### 2.6 Boot 流程

`boot()`（`dsh-app-boot/lib/index.js` 第 1166 行）：

```js
async function boot(binName, absoluteConfigPath, patches, prepare, bareModuleBaseUrl) {
    const ctx = new Context();                    // ① 创建 Cordis 根 Context
    ctx.baseUrl = ...;                             // ② 锚定 baseUrl
    ctx.provide("dshHomePath", dshHomePath);       // ③ 提供 home 路径服务
    await ctx.plugin(Loader);                      // ④ 安装 Cordis Loader
    await prepare?.(ctx);                          // ⑤ host 准备（注入 cmdline、env 快照）
    await mountRootInclude(ctx, absoluteConfigPath, patches, bareModuleBaseUrl);  // ⑥ 挂载根 Include
    await ctx.get("loader")?.await();              // ⑦ 等待插件树 settle
    await assertEntriesActivated(ctx, binName);    // ⑧ 审计：所有插件必须 ACTIVE
    return ctx;
}
```

**fail-loud 设计**：`installFailLoud` 在 boot 前装好 `unhandledRejection` 处理器，任何插件 init 失败 → 打印诊断 → `exit(1)`。

### 2.7 启动后收尾

- 挂载 HMR（依赖 timer）
- `watchUserPatches` — 监听 `cordis.patch.yml` 变化，热重载 patch 层
- `createProcessShutdown` — SIGINT(130)/SIGTERM(0) → dispose 整棵树（5 秒超时）→ exit

### 2.8 启动流程图

```
dsh --profile web --patch x.yml
  │
  ├─ loadLayeredEnv()  →  三层环境快照
  │
  ├─ composeProfile("web", ["x.yml"])
  │     ├─ prepareProfile → healProfilesModuleFallback + 重写空 cordis.yml
  │     ├─ loadProfile → 解析 bundle 列表，读每个 bundle 的 cordis.patch.yml
  │     ├─ 读 profile + home + --patch overlays
  │     └─ applyEntryPatches 叠加 → 有效 entry 列表
  │
  ├─ boot()
  │     ├─ new Context() + ctx.plugin(Loader)
  │     ├─ prepare: 注入 env 快照 + cmdline
  │     ├─ mountRootInclude: 根 config + patches → Loader 逐 entry 激活插件
  │     ├─ loader.await(): 等待插件树 settle
  │     └─ assertEntriesActivated: 审计全部 ACTIVE（fail-loud）
  │
  ├─ 挂 HMR + watchUserPatches
  ├─ installFailLoud
  └─ createProcessShutdown
        → 返回 { ctx, shutdown }
```

---

## 3. Cordis 框架内核

> Cordis 是 DSH 的地基。核心思想：**Context 是一个 Proxy，每个属性读取都是一次服务查找；插件是 Fiber（生命周期），通过 inject 声明依赖，服务可用才激活，服务消失自动卸载。**

源码位于 `@deepseek-ai/cordis/src/`，提供完整 TypeScript 源码。

### 3.1 Context：一个 Proxy

`context.ts` 第 42-84 行，`new Context()` 构造时返回的是 Proxy 而非原始 this：

```js
constructor() {
    this[symbols.isolate] = Object.create(null)    // 隔离 map
    this[symbols.intercept] = Object.create(null)  // 拦截 config map
    const self = new Proxy<this>(this, ReflectService.handler)  // ← Proxy
    this.root = self
    this.fiber = new Fiber(self, ..., null, ...)    // 根 fiber（uid=0, 永远 ACTIVE）
    this.reflect = new ReflectService(self)          // 服务解析层
    this.registry = new RegistryService(self)        // 插件注册表
    this.events = new EventsService(self)            // 事件总线
    this.logger = new LoggerService(self)            // 日志
    return self   // ← 返回 Proxy
}
```

Proxy 的 `get` trap（`reflect.ts` 第 136-171 行）决定 `ctx.xxx` 的读取行为：

```js
get: (target, prop, ctx) => {
    if (isSpecialProperty(prop)) return Reflect.get(target, prop, ctx)
    if (Reflect.has(target, prop)) return getTraceable(ctx, Reflect.get(target, prop, ctx))
    // 否则 → 服务查找：沿 fiber 父链向上查找 store
    return ctx.events.waterfall('internal/get', ctx, prop, error, () => {
        const key = target[symbols.isolate][prop]
        let fiber = ctx.fiber
        while (true) {
            const impl = fiber.store?.[prop]   // 当前 fiber 的 store 里有吗？
            if (impl) return impl.value         // 有 → 返回服务值
            if (prop in fiber.inject) throw error  // 声明了依赖但没找到 → 抛错
            if (!fiber.runtime) throw error       // 到根了 → 抛错
            fiber = fiber.parent.fiber            // 向上找
        }
    })
}
```

**所以 `ctx.systemPrompt`、`ctx.llm`、`ctx.session` 这些读取，本质是沿 fiber 父链向上查找谁提供了这个服务。**

### 3.2 Service：服务的基类

`Service` 基类（`service.ts` 第 42-58 行）在构造时自动注册：

```js
constructor(ctx, name) {
    self.ctx = ctx
    self.name = name
    self.ctx.reflect.provide(name, self, this[Service.check])  // ← 自动注册
    return self
}
```

DSH 的插件写法就是 `class SystemPrompt extends Service`，构造时调 `super(ctx, "systemPrompt")`。卸载时 `provide` 返回的 disposer 自动删除注册。

`provide` 实现（`reflect.ts` 第 277-305 行）关键点：
- 注册到 `this.store[key]`（key 是隔离 label）
- 写入 `fiber.store[name]`
- 如果 fiber 已 ACTIVE，调 `notify([name])` **唤醒所有依赖这个服务的 fiber**

### 3.3 Fiber：插件的生命周期

FiberState 枚举（`fiber.ts` 第 147-154 行）：

```
PENDING   — 等待依赖服务
LOADING   — 插件回调正在运行
ACTIVE    — 已加载，正在提供服务
FAILED    — 配置或启动抛错
DISPOSED  — 已移除，不能重启
UNLOADING — 正在运行 disposers 清理
```

**核心机制是 epoch（纪元）驱动的依赖追踪：**

`_refresh()`（第 611-623 行）计算依赖指纹：

```js
_refresh() {
    let epoch = ''
    for (const name of Object.keys(this.inject)) {
        const impl = this._store[name]
        if (!impl) { epoch = INACTIVE; break }  // 依赖缺失
        epoch += ':' + impl.fiber.uid            // 依赖满足 → 拼成 "uid1:uid2:..."
    }
    this._setEpoch(epoch)
}
```

`_setEpoch`（第 625-639 行）比较新旧 epoch：
- `INACTIVE → 有值`：依赖到齐 → `_reload()` → LOADING → 执行回调 → ACTIVE
- `有值 → INACTIVE`：依赖没了 → `_unload()` → UNLOADING → 跑 disposers → PENDING

**epoch 是依赖指纹——任意一个依赖的 fiber uid 变了（比如服务重启），epoch 变化触发重新加载。这就是服务提供者重启 → 依赖插件自动重载的热重载基础。**

### 3.4 effect()：副作用注册

`fiber.effect()` 注册"有副作用、需清理"的操作：

```js
ctx.on("event", listener)     // 注册事件监听，fiber 卸载时自动移除
ctx.provide("name", value)    // 注册服务，卸载时自动撤销
ctx.effect(() => { ...; return () => cleanup })  // 通用副作用
```

所有注册到 `fiber._disposables`，卸载时**逆序执行**。

### 3.5 事件系统：5 种分发模式

```
emit       — 同步触发，不等返回值
parallel   — 并发触发所有监听器，全部 await
serial     — 顺序 await，直到一个 bail
bail       — 同步顺序，直到一个返回非 null/false/undefined
waterfall  — 链式包裹，每个监听器包裹后面的链，最内层是 next 回调
```

**waterfall 是最核心的扩展机制**（`events.ts` 第 234-243 行）：

```js
waterfall(...args) {
    const cbs = this.dispatch('waterfall', args)
    const inner = args.pop()    // 最后一个参数是 next（默认行为）
    const next = () => {
        const cb = cbs.shift() ?? inner
        return cb(...args)
    }
    args.push(next)
    return next()   // 从最外层开始调
}
```

每个监听器收到 `(arg1, arg2, ..., next)`，可以：
- 调 `next(...)` → 继续链
- 不调 `next()` → **否决**，链终止

DSH 中的 waterfall 用例：
- `system-prompt/assemble` — 插件变换最终 prompt assembly
- `internal/update` — HMR 钩子否决配置更新
- `agent/pre-step` — reject 或改写要进入的消息
- `agent/request` — 改写 LLM 请求配置

**事件监听器自动绑定到 fiber 生命周期，且支持 context filter**（只有 `global: true` 跳过过滤）。

### 3.6 ScopedLayers：作用域分层

DSH 的很多注册表（systemPrompt 的 sections/contexts、tools）不是全局的，而是"全局层 + 作用域层"分层的：
- 全局层：所有 scope 可见
- scope 层：只在特定 fiber 作用域内可见，shadow 全局层同名条目
- `merge(scope)` 合并：先取全局层，再用 scope 链上的条目覆盖同名

### 3.7 plugin() 和 inject()

`ctx.plugin(plugin, config)`（`registry.ts` 第 316-336 行）：
1. 解析插件形态（function / class / `{apply}` 对象）
2. 复用或创建 Runtime 记录（按 callback 为 key）
3. `new Fiber(...)` 创建 fiber
4. 返回 fiber（可 `.await()` 等待激活）

`ctx.inject(deps, callback)` 是快捷方式——**依赖可用时跑回调，依赖变化时自动重跑**。

### 3.8 完整流转图

```
ctx.plugin(MyPlugin, config)
  → RegistryService.plugin()
    → new Fiber(parent, config, {llm, session, ...}, runtime)
      → state = PENDING
      → _refresh() 算 epoch
        → 依赖全齐？epoch = "uid1:uid2"
          → _reload() → state = LOADING → 执行回调 → state = ACTIVE
        → 依赖缺？epoch = INACTIVE
          → state = PENDING，等服务 notify

某个 Service 被 provide/撤销
  → reflect.notify([name])
    → 遍历所有 runtime 的所有 fiber
      → 如果 fiber.inject 含这个 name → _checkImpl + _refresh
        → epoch 变化 → _reload 或 _unload
```

### 3.9 Cordis 提供的能力总结

| 概念 | 作用 | DSH 怎么用 |
|---|---|---|
| Context (Proxy) | 属性读取=服务查找 | `ctx.llm`、`ctx.session`、`ctx.systemPrompt` |
| Service | 自动注册/注销服务 | 所有 DSH 插件 extends Service |
| Fiber | 插件生命周期+依赖追踪 | 每个插件实例一个 fiber，epoch 驱动 |
| inject | 声明依赖，服务可用才激活 | 插件声明 `inject: ["llm", "session"]` |
| effect | 副作用+自动清理 | `ctx.on`、`ctx.provide` 都是 effect |
| 事件 5 模式 | emit/parallel/serial/bail/waterfall | waterfall 是核心扩展点 |
| ScopedLayers | 全局+作用域分层注册 | systemPrompt sections、tools schema |
| notify | 服务变化唤醒依赖 | 服务重启→依赖插件自动重载 |

---

## 4. System Prompt 动态组装

> 源码：`@deepseek-ai/dsh-system-prompt/lib/index.js`

### 4.1 是否用了动态 system prompt？

**是的，用了动态 system prompt**，而且不是简单的字符串拼接，是一套"注册 + 分层作用域 + 事件变换 + 每步重组装"的完整引擎。

### 4.2 四类可注册输入

| 输入类型 | 作用 | 特点 |
|---|---|---|
| `sections` | 有序固定段落（基础人设/指令） | 静态文本或函数，可被 `complete` 段落短路 |
| `contexts` | 动态运行时上下文 | 按 `order` 排序，每次渲染；可被 `suppressRuntimeContext()` 抑制 |
| `variables` | 模板变量 | 插值进 section/context 文本 |
| `toolProviders` | 工具 schema 提供者 | 动态决定当前可用工具 |

### 4.3 组装流程

`assemble()`（第 240-288 行）每次模型步前重新拼装：

1. 合并全局层 + 作用域层的 variables/contexts/sections
2. contexts 按顺序排序并渲染（生成 `"Current runtime context..."` 快照）
3. 经过 Cordis `system-prompt/assemble` waterfall 事件，允许插件变换
4. `invariant.js` 插件做校验（段名/上下文名不能重复）

### 4.4 固定 sections 的 order

```js
// 构造时注册的基础 sections（第 166-175 行）
this.section({ name: "harness:identity", order: -100, text: "You are an AI agent powered by DeepSeek Harness." });
this.section({ name: PERSONA_SECTION, order: 0, text: config.persona ?? "" });
```

`addHarnessSourceSection`（第 1205-1213 行）注册 order=-99 的 checkout 路径段：
```js
systemPrompt.section({
    name: HARNESS_SOURCE_SECTION,
    order: -99,
    text: `The DeepSeek Harness implementation checkout is at ${sourceRoot}...`
});
```

### 4.5 渲染函数

```js
// 只渲染 sections（第 65-67 行）
function renderPrompt(assembly) {
    return assembly.sections.map(s => interpolate(s, assembly.variables, "section"))
        .filter(text => text.length > 0).join("\n\n");
}

// 渲染动态 context 快照（第 73-87 行）
function renderContextSnapshot(assembly) {
    const body = sections.map(s => s.text).join("\n\n");
    return `Current runtime context. This snapshot supersedes earlier runtime-context snapshots.\n\n${body}`;
}
```

### 4.6 动态部分的实际内容

从 system prompt 中可直观看到的动态注入内容：
- `Current runtime context. This snapshot supersedes earlier runtime-context snapshots.`
- `Current DSH file policy: workspace-write` / `Approval policy: ask`
- 工作目录路径、checkout 路径、GUI URL
- `<system-reminder>` 里的 `available_skills` 列表

---

## 5. 动态 Prompt 与缓存优化

> 核心矛盾：动态 system prompt 会破坏 prefix caching。DSH 用三层设计化解。

### 5.1 核心设计：把"动态"从 system prompt 里分离出去

在 `dsh-agent-loop/lib/index.js` 中，每次 step 分两步：

**第一步（第 611 行）：渲染 system prompt（只含 sections，不含动态 context）**
```js
const system = renderPrompt(assembly);  // 只渲染 sections
```

`renderPrompt()` 只拼接 `sections`，不包含 dynamic runtime context。这个 `system` 字符串跨 step 稳定——可缓存的 prefix。

**第二步（第 499-507 行）：动态 context 作为 user 消息追加**
```js
const sections = renderContextSections(assembly);
const context = this.runtimeContext.project(joinContextSections(sections), sections);
// 动态快照作为最后一条 user 消息
messages: context === void 0 ? claimed : [...claimed, context]
```

那段 `"Current runtime context..."` 是独立的 user 消息追加到消息流末尾，不在 system prompt 里。

### 5.2 RuntimeContextProjection：增量去重

`dsh-agent-loop/lib/index.js` 第 13-82 行：

```js
const CLEARED = "Current runtime context: none. Earlier runtime-context snapshots no longer apply.";

project(current, sections) {
    if (this.retained === void 0 && current.length === 0) return;       // 从来没有且现在也没有 → 不发
    const snapshot = current.length === 0 ? CLEARED : current;
    if (this.retained?.text === snapshot) return;                        // 和上次一样 → 不发！
    return createUserMessage({ content: [...], source: {...} });         // 只在变化时才追加
}
```

**如果这步的 runtime context 和上一步完全相同，不追加任何新消息 → 消息流不变 → prefix 完全命中缓存。**

### 5.3 缓存友好分层

| 内容 | 位置 | 稳定性 | 缓存效果 |
|---|---|---|---|
| 基础人设 / persona / 工具规范 | system prompt (sections) | 跨 step 稳定 | ✅ 完整命中 |
| 历史 turn (user/assistant/tool) | 消息流中段 | 只增不改 | ✅ 增量命中 |
| runtime context 快照 | 消息流末尾 user 消息 | 大多数 step 不变 | ✅ 不变时不追加 |

### 5.4 config 变化追踪

`dsh-llm/lib/types/call-config.js` 中 `callConfigEquals()` 逐字段比较 provider/model/temperature/maxTokens/stop，并记录 `request/header` 变更。

### 5.5 headerEquals：请求头去重

`dsh-session/lib/index.js` 第 548-553 行：

```js
function headerEquals(a, b) {
    if (!callConfigEquals(a.config, b.config) || ...) return false;
    if (a.system !== b.system) return false;                          // system 文本变了？
    const at = a.tools ?? [];
    const bt = b.tools ?? [];
    return at.length === bt.length && at.every((tool, i) => sameSchema(tool, bt[i]));
}
```

`buildRequest` 中只有 `!headerEquals(baseline, header)` 时才记录 `"change"`。

### 5.6 缓存失效场景

| 情况 | system 变? | tools 变? | 缓存影响 |
|---|---|---|---|
| 正常多步对话 | ❌ | ❌ | ✅ 前缀完全命中 |
| runtime context 变了 | ❌ | ❌ | ✅ 前缀命中（context 在末尾，去重） |
| 进入 plan mode | ✅ | ❌ | ⚠️ system 失效，tools 仍命中 |
| 插件 HMR 热重载 | 可能 | 可能 | ⚠️ 低频事件 |
| 切 provider/model | ✅ | 可能 | ⚠️ 完全不同 KV cache |

### 5.7 token meter 追踪缓存

`dsh-token-meter/lib/index.js` 第 200-228 行明确追踪 `cacheReadTokens` 和 `cacheWriteTokens`。provider adapter（如 `dsh-llm-pi-ai`）映射 provider 返回的 cache 字段。

---

## 6. 工具系统

> 源码：`@deepseek-ai/dsh-tools/lib/index.js`

### 6.1 工具是完全动态的——三层动态性

#### 第一层：插件级动态注册

每个工具包在 `apply` 回调里调用 `ctx.tools.register()`：

```js
// dsh-tool-fs/lib/index.js 第 333 行
ctx.tools.register(defineTool({
    name: "read",
    description: "Read a UTF-8 text file and return line-numbered content.",
    parameters: { file_path: {...}, offset: {...}, limit: {...} },
    output: { schema: {...}, render: ... },
}));
```

`register()` 返回 disposer，绑定到 fiber 生命周期——工具包卸载时自动撤销。

```js
// dsh-tools/lib/index.js 第 2755-2763 行
register(definition) {
    return this.layers.effect(this.ctx, (layer) => layer.tools.insert(name, definition),
        { label: "tools.register()" });
}
```

#### 第二层：作用域级可见性

`ToolRuntime` 用 `ScopedLayers` 管理工具（第 2571 行）：

```js
layers = new ScopedLayers((scope) => new ToolLayer(scope), () => {
    this.ctx.emit("tools/change");
});
```

`view(scope)`（第 2836 行）解析某 agent scope 实际看到哪些工具：

```js
view(scope) {
    const layers = this.layers.chainLayers(scope);
    const inherited = new Map(this.layers.global.tools.entries());  // 全局层
    for (const layer of layers) {
        // scope 链上的工具 shadow 全局同名工具
        for (const [name, def] of layer.tools.entries()) inherited.set(name, def);
    }
    const visible = new Map();
    for (const [name, def] of inherited) {
        // 每层都有 restrict 过滤器，全部 admits 才可见
        if (layers.every((layer) => layer.admits(name))) visible.set(name, def);
    }
    return { visible, knownNames, restrictableNames };
}
```

作用域级操作：
- `restrict({allow, deny})` — 给某 agent scope 加全局工具过滤器
- `presentAs("code"|"native"|"both")` — 切换呈现模式

**不同 agent preset 有不同工具集**，完全是配置驱动的动态组合。

#### 第三层：每步实时组装

`ToolRuntime` 构造时注册为 systemPrompt 的 toolProvider（第 2588 行）：

```js
constructor(ctx, config) {
    super(ctx, "tools");
    ctx.systemPrompt.tools((context) => this.wireSchemas(context.scope));
}
```

每次 `assemble()` 执行（第 249-280 行）：

```js
const providers = [...global.toolProviders, ...scopeLayers.flatMap(...)];
for (const provider of providers) {
    const result = provider(context);           // 调 ToolRuntime.wireSchemas(scope)
    collected.push(...result.schemas);
}
tools: orderTools(collected, this.toolOrder, knownNames)
```

### 6.2 工具包挂载方式

工具通过 agent preset 的 `cordis.yml` 配置挂载。以 `standard` preset 为例：

```yaml
- id: tool-bash
  name: '@deepseek-ai/dsh-tool-bash'
- id: tool-fs
  name: '@deepseek-ai/dsh-tool-fs'
- id: tool-fs-search
  name: '@deepseek-ai/dsh-tool-fs-search'
- id: tool-jobs
  name: '@deepseek-ai/dsh-tool-jobs'
- id: tool-skill
  name: '@deepseek-ai/dsh-tool-skill'
- id: tool-goal
  name: '@deepseek-ai/dsh-tool-goal'
- id: tool-subagent
  name: '@deepseek-ai/dsh-tool-subagent'
- id: tool-workflow
  name: '@deepseek-ai/dsh-tool-workflow'
# ...等约 15+ 个工具包
```

### 6.3 ToolRuntime 类

```js
var ToolRuntime = class extends Service {
    static inject = ["systemPrompt"];  // 依赖 systemPrompt 服务
    static Config = z.object({
        mode: z.union(["native", "code", "both"]).default("native"),
        maxParallelSubCalls: z.natural().min(1).default(10)
    });
    // ...
};
```

### 6.4 Code Mode

工具可以以原生 schema 或 Code Mode（生成 TypeScript/Python SDK）呈现：
- `presentAs("code")` — 让某 agent 的工具以 SDK 方式呈现
- `run_code` 是保留的传输工具名，不在注册层管理
- SDK 从注册的工具 schema 实时生成

### 6.5 工具调度

`TOOL_RUNTIME_SCHEDULER` 提供四阶段调度：
- `prepare` — 准备执行
- `dispatch` — 分派执行
- `finalize` — 结果处理
- `finish` — 完成

支持并行工具调用（`maxParallelSubCalls` 默认 10），独占调用形成 barrier。

---

## 7. 工具实时组装与缓存

### 7.1 核心结论

`assemble()` 每步都调，但"调了"不等于"变了"。有三道防线确保"大多数 step 组装出来的结果和上次完全一样"。

### 7.2 防线一：headerEquals 逐字节比较

```js
function headerEquals(a, b) {
    if (!callConfigEquals(a.config, b.config) || ...) return false;
    if (a.system !== b.system) return false;
    const at = a.tools ?? [];
    const bt = b.tools ?? [];
    return at.length === bt.length && at.every((tool, i) => sameSchema(tool, bt[i]));
    // sameSchema = JSON.stringify 比较
}
```

只有 `!headerEquals` 时才记录 `"change"`，多数 step 不记录。

### 7.3 防线二：工具列表运行时几乎不变

工具列表变化只在：插件 HMR、agent 切换 scope、动态 restrict/presentAs。正常对话中不发生 → 每步组装结果逐字节相同。

### 7.4 防线三：plan mode 保持工具列表不变

`agent.cordis.yml` 第 118 行：
> *"The tool catalog stays the same across modes for request-cache stability."*

进入 plan mode 时**不删工具**，用 system prompt 文字规则替代。保护 tools 前缀缓存。

### 7.5 system 和 tools 在请求中的位置

```js
request: markAgentLoopRequest(deepFreeze({
    ...header.config,                      // provider/model/temperature
    messages: boundaryMessages,            // 对话消息
    ...header.system ? { system: header.system } : {},  // system（独立字段）
    ...header.tools ? { tools: header.tools } : {},     // tools（独立字段）
    sessionId: this.session.id,
    signal
})),
```

system + tools 构成请求前缀（在 messages 之前）。只要不变，messages 只增不改 → 前缀缓存命中。

### 7.6 缓存追踪

`dsh-token-meter` 追踪 `cacheReadTokens` / `cacheWriteTokens`。`dsh-llm-pi-ai` 映射 provider 返回的 `cacheRead`/`cacheWrite`。

---

## 8. Agent Loop

> 源码：`@deepseek-ai/dsh-agent-loop/lib/index.js`

### 8.1 Turn / Step 结构

```
turn = 一次用户输入到 agent 回复完成的循环
  step = turn 内的一次 LLM 调用 + 工具执行
```

### 8.2 preStep（第 492-514 行）

```js
async preStep(target, position) {
    const claimed = this.inbox.claim(target, position.turn);
    const assembly = await this.loopCtx.systemPrompt.assemble(assembleContextFor(this, signal));
    const sections = renderContextSections(assembly);
    const context = this.runtimeContext.project(joinContextSections(sections), sections);
    const decision = await this.dispatch.waterfall("agent/pre-step", {
        messages: claimed, ...position, signal
    }, () => Promise.resolve({
        kind: "enter",
        messages: context === void 0 ? claimed : [...claimed, context]
    }));
    return { ...decision, assembly };
}
```

### 8.3 step（第 606-665 行）

```js
async step(assembly) {
    const system = renderPrompt(assembly);  // 只渲染 sections
    while (true) {
        const { request, preparedCall } = await this.buildRequest(
            turn, step, assembly.tools, system, this.session.deriveMessages(), signal
        );
        const stream = preparedCall?.stream(request) ?? this.loopCtx.llm.stream(request);
        // 流式接收 chunks
        for await (const chunk of stream) {
            assembler.push(chunk);
        }
        // 处理 finish: error / aborted / max-tokens / completed
        // 如果有 tool calls → executeToolCalls → 返回 null（继续循环）
        // 如果没有 tool calls → 返回 { kind: "completed" }
    }
}
```

### 8.4 buildRequest（第 670-739 行）

- 解析 provider/model 路由
- 跑 `agent/request` waterfall（允许改写 config）
- `prepareCall` 解析 adapter 默认值
- 构建 `canonicalHeader`（config + system + tools）
- `headerEquals` 对比，只在变化时记录
- 返回冻结的 request 对象

### 8.5 消息流结构

```
[system prompt（sections，稳定）]
[messages:
  user: "原始用户输入"
  assistant: "回复 + tool_calls"
  user: "tool result"
  assistant: "继续..."
  user: "Current runtime context snapshot..."  ← 末尾，按需追加
]
```

---

## 9. 会话系统 Session

> 源码：`@deepseek-ai/dsh-session`（lib/index.js 1886行 + 8个类型模块）、`dsh-session-persistence`、`dsh-session-persistence-jsonl`

### 9.1 事件溯源（Event-Sourced）机制

> 🔑 **Session 是一个 append-only event log**，所有状态变更都是不可变事件，`seq = log.length` 是连续性契约。

`Session.append()` 的 13 步流程：
1. 校验事件类型在 `KNOWN_SESSION_EVENT_TYPES`（46 种）中
2. 分配 `seq = events.length`
3. 构造完整事件信封（type/data/seq/createdAt/surfaceOp 等）
4. 快照优先：先更新 surface（如果事件是 surface-eligible）
5. 追加到 events 数组
6. 通知监听器（隔离 try-catch，一个监听器失败不影响其他）
7. 重入防护
8. 持久化触发（write-behind）
9. Typert 集成
10. 返回事件引用

种子重放验证：fork 子代理时，seed 事件前缀被验证为合法的连续 log。

### 9.2 Projection（投影）机制

> 🔑 **Surface 是 session event log 之上的有序视图**——只保留产生 LLM 消息的事件。

三种 surface-eligible 事件类型：
- `user/message` — 投影为 data
- `assistant/message` — 投影为 message（空内容返回 null）
- `tool/result` — 投影为 data.message

每个 surface-eligible 事件必须带 `surfaceOp`：
- `"append"` — 追加到 surface
- `{op: "replace", start, end}` — 位置替换（替换 [start, end] 区间）

`SurfaceManager` 做增量折叠 + 缓存：
- `foldSurface(events)` 完整回放 → `{nodes: [...seqs], replacements: [...]}`
- `deriveMessages()` O(new nodes) 缓存策略——只处理新增节点

### 9.3 JSONL 持久化

磁盘布局：`$DSH_HOME/sessions/<projectKey>/<encodedId>/session.jsonl`

文件格式：
- 第 1 行：header（JSON，含 session 元数据）
- 后续行：事件行（每行一个 JSON 事件）

> 🔑 **chunk-row 压缩**：实测 56× 膨胀（单事件单行）→ 打包成单行。`SessionLogScanner` 容忍撕裂的尾部行（崩溃恢复）。

write-behind 缓冲 + fsync + 崩溃回滚机制：
- 写入先进入内存缓冲
- 定时或定量 flush 到磁盘
- flush 时先写 `.tmp` 再 rename（原子写）
- 崩溃后恢复时扫描尾部，丢弃不完整行

### 9.4 事件类型

`SessionEventMap` 全集 46 种，按域分组：
- **turn/step**：`turn/start`、`turn/end`、`step/start`、`step/end`
- **chunk**：`assistant/chunk`（流式 token 增量）
- **message**：`user/message`、`assistant/message`
- **tool**：`tool/call`、`tool/result`
- **header**：`request/header`、`request/context`
- **todo**：`todo/update`
- **其他**：`llm/retry`、`session/end-seed`、`goal/change`、`schedule/change`、`subagent/descriptor` 等

`ignorable` 标记的版本策略：未知事件类型带 `ignorable: true` 可被旧版本安全跳过。

### 9.5 Header 管理

`EpochHeader` 结构：`{config, system, tools, adapterDefaults}`

- `canonicalHeader()` — 规范化（去空、排序）
- `headerEquals()` — 逐字段比较（config + system + tools 逐个 JSON.stringify）
- `foldRequestHeader(events)` — 离线重建（从事件流折叠出最新 header）
- `Session.requestHeader()` — 增量缓存（live 追踪）

### 9.6 创建、恢复（resume）、fork

| 操作 | 入口 | 特点 |
|---|---|---|
| 创建 | `Session.create()` | 新 session，header 为空 |
| 恢复 | `Session.fromRestore()` | 从持久化加载，snapshot 模式 |
| fork | 子代理继承 | seed = 父会话已完成 turn 的事件前缀 |

`session/end-seed` 事件标记 seed 边界。`firstLiveSeq` vs `seedLength` 区分种子事件和活跃事件。

fork 约束：`OPEN_TURN` — 如果父会话有未完成的 turn，不能 fork。

生命周期拆分：`prepare()` → `enter()` → `announce()`。

### 9.7 SessionStore

`SessionStore` 用 `WeakMap` 管理 attachments（图片等二进制资源）：
- entry 结构含 `liveEntryFor` 精确活性检查
- 延迟 detach（不立即释放，等 GC）
- Typert 集成（会话级类型注册表）

### 9.8 不变量系统

两阶段 pre-commit staging：
1. **staging 阶段**：校验事件合法性（类型、seq 连续性、字段完整性）
2. **commit 阶段**：追加到 log + 通知

崩溃恢复合成事件：`TOOL_NOT_STARTED` / `TOOL_OUTCOME_UNKNOWN` — 如果崩溃时有未完成的 tool call，恢复时合成这些事件保持 log 完整性。

---

## 10. LLM 适配层

> 源码：`@deepseek-ai/dsh-llm`、`dsh-llm-pi-ai`、`dsh-llm-retry`、`dsh-llm-deepseek`

### 10.1 核心架构

`LlmRuntime extends Service`，注册为 `ctx.llm`，维护三个 Map：

| Map | 键 | 值 | 用途 |
|---|---|---|---|
| `adapters` | provider route id | `{adapter, provider, retryPolicy}` | 活跃路由表 |
| `directory` | provider route id | `LlmConfigurableProvider` | 可配置 provider 目录（含休眠） |
| `discoveries` | settings namespace | discover 函数 | 模型发现（配置面板用） |

> 🔑 **All-or-nothing 注册**：`prepareRoutes()` 校验候选路由集 → `commitRoutes()` 在一个同步段完成删除旧 + 写入新（无 await，无观察者能看到空表）。返回的 handle 带 `replace()` 方法支持原子性路由切换。

### 10.2 请求生命周期

```
Agent Loop
  ├─ prepareCall(config, signal) → PreparedLlmCall
  │    ├─ registration(config.provider) → 查路由表
  │    ├─ resolveCallFor() → 解析模型能力 + 物化默认值
  │    └─ 返回 { config, retryPolicy, adapterDefaults, stream }
  │
  └─ prepared.stream(options) → AsyncIterable<StreamChunk>
       └─ ctx.waterfall("llm/stream", options, () => adapterStream())
            ├─ [llm-invariant] 流语法校验（global + prepend）
            ├─ [其他中间件] llm/stream 监听器
            └─ adapterStream() → 最终 adapter 边界
                 ├─ forAdapter() 剥离不归属的 replayState
                 ├─ adapter.stream() → adapter 流
                 └─ 异常 → terminal finish chunk
```

> 🔑 **HMR 防护**：`prepareCall` 捕获 registration 快照，`stream()` 始终用这个快照分派。即使 HMR 在准备和分派之间替换了 adapter，请求仍走旧注册。

### 10.3 StreamChunk 协议

Provider 中立的流式协议：

| chunk 类型 | 作用 |
|---|---|
| `block-start` | 开始一个内容块（text/reasoning/tool-call） |
| `text-delta` | 文本增量 |
| `reasoning-delta` | 推理增量 |
| `tool-call-delta` | 工具调用参数增量（原始 JSON 字符串） |
| `block-end` | 块结束（携带完整 block） |
| `usage` | token 用量 |
| `finish` | 终止（reason: completed/error/aborted/max-tokens） |

`BlockAssembler` 将 chunk 增量组装为消息：
- 容忍 delta-only 协议（无 block-start 时自动创建）
- 已关闭块的 delta 被静默忽略（malformed stream 保护）
- **max-tokens 截断时丢弃所有 tool-call block**（截断的参数不安全）

### 10.4 provider/model 解析

三层递进：
- `listModels(provider)` — 咨询性目录（不控制路由）
- `resolveModelInfo(provider, model)` — 精确元数据（context、maxTokens、reasoning）
- `resolveCallConfig(config)` — 校验 + 物化默认值

> 🔑 **无 clamping/aliasing**：不支持的 reasoning effort 直接拒绝（`UNSUPPORTED_REASONING_EFFORT`），不静默降级。

### 10.5 缓存与配置检测

`callConfigEquals()` 逐字段比较 provider/model/reasoningEffort/temperature/maxTokens/stop。

`deepFreeze()` 深冻结配置（迭代遍历 + WeakSet 循环检测），**跳过 AbortSignal**（实时取消通道不能 freeze）。

`AGENT_LOOP_REQUESTS` WeakSet 标记 agent loop 构建的请求——监听器只能读取不能重写。

### 10.6 Token 计量

```typescript
interface TokenUsage {
    inputTokens: number;        // 未缓存的输入
    outputTokens: number;
    cacheReadTokens?: number;   // 缓存命中
    cacheWriteTokens?: number;  // 缓存写入
    reasoningTokens?: number;
}
```

> 🔑 **不相交计数**：`inputTokens` 是未缓存输入；缓存命中单独报告。计费输入 = 三者之和。DeepSeek 的 `prompt_tokens` 折叠缓存命中，adapter 需减出来。

### 10.7 Replay State

> 🔑 **Harness content 是持久源**。replay state 只存重建 provider 原生消息所需的最小 provider-native 元数据（签名等）。

`forAdapter()` 是核心安全机制：replay state **只在与目标 adapter 同属时保留**。不同 adapter → 剥离 replayState → 降级为 `foreignAssistant`（`api: "dsh-foreign"`，不携带签名）。

### 10.8 Adapter 注册模式

| 特性 | deepseek-official | pi-ai |
|---|---|---|
| 路由数 | 单一固定 | 多路由（配置 dict） |
| 网络层 | 直接 fetch | 通过 pi-ai 库 |
| 协议 | DeepSeek 原生 | openai/anthropic 多协议 |
| 模型发现 | 内置目录 | `registerModelDiscovery` |
| 设置路径 | `settingsPath: []` | `settingsPath: ["providers", provider]` |

### 10.9 错误处理

`LlmError extends HarnessError`，携带冻结的 `failure` 快照（code/message/status/providerRetryAfterMs/requestId）。

> 🔑 **Route on code, never by parsing message**：`code` 是稳定的程序化失败类，路由逻辑基于 code。

可重试错误码：`EMPTY_RESPONSE`、`RATE_LIMIT`、`SERVER`、`TIMEOUT`、`TRANSPORT`（默认最多 2 次）。

重试策略：指数退避 + 对称抖动（`delay = min(initialDelay * 2^(retry-1) * jitter, maxDelay)`），`providerRetryAfterMs` 优先。

`dsh-llm-retry` 在 `agent/request-error` 扩展点上运行，重试事件先写入会话日志（durable before cancellable wait）。

### 10.10 归因头

每个 adapter 必须在每个 HTTP 请求中包含 `attributionHeaders()`（User-Agent: `deepseek-harness/<version>`），不可抑制。

---

## 11. 子代理 Subagent

> 源码：`dsh-subagent`、`dsh-tool-subagent`、`dsh-tool-subagent-control`

### 11.1 注册表

`SubagentRuntime extends Service`，注册为 `ctx.subagents`，**进程级单例**。多 Provider 共存于 `providers` Map。

`SubagentContinuationManager` 管理 continuable 子代理的驻留 Activation。

### 11.2 三种创建模式

| 模式 | 返回值 | 等待 | Task | 可后续交互 |
|---|---|---|---|---|
| Foreground | `{kind:"foreground", runId, output}` | ✅ | 无 | 否 |
| Background one-shot | `{kind:"background", jobId}` | ❌ | 有 | 否 |
| Background continuable | `{kind:"continuable", subagentId}` | ❌ | 无 | ✅ send_message |

### 11.3 subagent vs subagent_fork

> 🔑 区别由 Provider 的 `inheritsParentContext` 属性驱动：
- **subagent (spawn)**：`false`，fresh child，无 seed
- **subagent_fork**：`true`，seed = 父会话已完成 turn 的事件前缀

fork seed 可能包含祖先的 descriptor，投影 fold 在遇到每个 `subagent/descriptor` 时重置累积状态（last-wins）。

### 11.4 send_message 机制

`followup()` 三路路由：
1. Activation 不存在 → `coldResume`（从持久化 Session 重建）
2. Activation 正在销毁 → 等待 disposal 后重试
3. Activation 活跃 → `submitAdmitted`（直接提交到 inbox）

> 🔑 **send_message 返回 messageId，不是结果**。消息排入子代理的 FIFO turn 队列。如果子代理正在工作，消息等待当前 turn 完成后才处理。

`ChildLock` 确保每个 durable child 的操作线性化（Promise 链串行化）。

### 11.5 生命周期状态

内部状态：`running`（执行中或有未处理 inbox）/ `waiting`（有子代理在运行）/ `settled`（过渡态，触发 disposal）

模型面向状态：`running` / `idle`（内存中但不在工作）/ `ready`（仅持久化，可冷恢复）

`watchSettlement` 循环：`Promise.race([agent.whenIdle(), poke])` → 检查是否 settled → dispose → `notifySettlement`。

### 11.6 Scope 继承

`applyChildComposition` 三步：
1. join 父 preset（工具注册表 + prompt 段落）
2. 注册固定 delegation-scope 声明（runtime-context）
3. 应用子级 persona + toolFilter

> 🔑 **approval 固定为 'never'**：子代理的权限沙箱在委派边界被冻结。只继承父会话的显式 sandbox override，不继承 deployment 默认值或一次性授权。

深度单调递增：`resolveChildDepth = max(persistedDepth, runtimeDepth) + 1`。

### 11.7 结果通知

`notifySettlement` 是通知父级的**唯一正确位置**：
- 无条件通知（不考虑子代理是否曾 report）
- 通知消息 source 为 `{kind: "subagent-settled", form: "notice"}`
- 通知失败被记录并丢弃（不阻塞 disposal）
- busy owner → `inject`（注入下一步）；idle owner → `followup`（开启新 turn）

### 11.8 Cold Resume

> 🔑 cold resume **从不通过 subagent provider 分发**——持久化 Session 已持有初始前缀，descriptor 是完整的重建输入。只从 durable descriptor 重建子代理选项，既不恢复之前的 budget 也不继承父级当前的。

### 11.9 list_agents / interrupt_agent

`list_agents`：scope 选择（children/descendants），三级阶梯身份解析（live → 投影缓存 → cold read），过滤掉 one-shot 子代理。

`interrupt_agent`：同步授权、异步生效。`keepInbox: true` 保留已排队工作。不存在的目标是 accepted no-op。

---

## 12. Goal 长目标系统

> 源码：`dsh-goal`、`dsh-goal-round-driver`、`dsh-tool-goal`、`dsh-command-goal`

### 12.1 数据模型

三层类型层次：`GoalId`(Branded) → `GoalRef`(CAS {id, revision}) → `GoalSnapshot` → `GoalView`(含 activation)

事件负载 `GoalChangeMeta`：version=1，分快照变更和 clear 墓碑两类。

`GoalFoldState` 含 `seenGoalIds` Set 防止 id 复用。

### 12.2 状态机

4 phase：`active` / `paused` / `blocked` / `complete`

转移规则：
- `edit` 不能改 phase
- `pause/resume/complete/block` 不能改 objective
- `block` 仅从 `active` 转入
- `resume` 需 round 预算未耗尽

### 12.3 Round Driver

`drive()` 决策序列：就绪检查 → checkpoint → 轮次预约 → goal 检查 → 耗尽保护 → 构造 prompt → followup

`requestDrive` coalesce 序列化。`pre-step` 双重 `validReservation` 校验防止过期轮次进入 step。

> 🔑 cancelled attempt 自动 pause 保护，避免无限重试死循环。

### 12.4 权限矩阵

| Action | 需要权限 |
|---|---|
| edit / pause / resume | `direct-human`（人类直接请求） |
| complete / blocked | `goal-round`（允许自动续跑） |

### 12.5 Blocked 条件

`blockedAfterConsecutiveRounds` 默认 3，仅约束 `goal-round` 权限（人类不受限）。

模型报告固定 `code=model-reported`；driver 自动 block 用 `round-limit/queue-failed/prompt-rejected`。

### 12.6 Session 持久化

完全事件溯源，`roundsStarted` 由 user/message 事件的 `GoalMessageSource.round` 推导。

两层 fold：strict（fail-loud，invariant 用）vs projection（容错 last-wins，缓存用）。

### 12.7 max_goal_rounds

默认 256，三处强制：resume 检查、fold 校验、driver 自动 block。edit 可调整上限实现"续命"。

### 12.8 Agent Loop 交互

> 🔑 **最关键设计：持久 phase 与进程级 activation 分离**。
- 仅 `create/resume` arm activation
- `session-start` 强制 disarm
- driver 启动/卸载全 disarm
- **人类 `resume` 是 session 重启后唯一 rearm 途径**——平衡自主性与安全性

---

## 13. Workflow 工作流编排

> 源码：`dsh-tool-workflow`、`dsh-workflow-worker-thread`、`dsh-workflow`

### 13.1 三层分离架构

```
tool-workflow（模型契约：schema + run 生命周期）
  → dsh-workflow（抽象 seam：WorkflowEngine 基类）
    → worker-thread（实现：WorkerThreadWorkflowEngine）
```

> 🔑 换引擎不动模型接口。脚本在 `node:vm.createContext` 中编译执行。

### 13.2 脚本 API

5 个 frozen 全局 hook 注入 vm context：

| Hook | 作用 |
|---|---|
| `agent(prompt, opts?)` | 创建 subagent，返回 text 或 structured |
| `pipeline(items, ...stages)` | 每个 item 独立跑完整 stage 链（无 barrier） |
| `parallel(thunks)` | 并发所有 thunk + await 全部（barrier） |
| `phase(title)` | 设置当前 phase 标签（进度词汇） |
| `log(message)` | 叙述日志 |

### 13.3 agent() 调用链

```
脚本 agent(prompt)
  → ChildRpcBridge.startAgent() post ChildStart（RPC 到 host）
    → host WorkerRun.onChildStart → startChild()
      → this.subagents.start(provider, {...})  ★ 接合点
      ← post ChildStarted
    ← worker 拿到 childId
  → await run.result
    ← host 转发 child.result → post ChildSettled/ChildFailed
  → 返回 text 或 structured
```

> 🔑 **workflow 是编排层，不执行 LLM 调用/文件 IO/网络**。脚本只协调 agent 调用。

### 13.4 错误纪律

- `WorkflowError`（默认 fatal）→ 杀脚本
- 普通 child/stage 失败 → `null`（`.filter(Boolean)` 过滤）
- `AGENT_CAP`/`ITEM_CAP`/`UNSUPPORTED_OPTION`/`UNSUPPORTED_SCHEMA` → fatal

### 13.5 并发限制

- 自动并发：`min(16, max(1, cores-2))`
- 总量上限：默认 1000（runaway 后备）
- per-call item 上限：4096
- 同步超时：5s
- dispose grace：5s + terminate

### 13.6 Worker Thread 隔离

> 🔑 **不是安全边界**，是"事件循环隔离 + 强制终止 + 超时"。

- 环境清洗（无 ambient credentials）
- execArgv 清空
- vm context 隔离（无 Node.js API）
- realm 物化（值离开 vm 前转成纯 JSON）
- structured clone（workerData 和 postMessage）
- `worker.terminate()` 物理杀掉卡住脚本

### 13.7 exactly-once 语义

host 维护 agent ledger，保证每个 `agent-start` 恰好有一个 `agent-end`。worker 能说话时用其报告；不能说话时 host 合成 cancelled end。

---

## 14. Web GUI

> 源码：`dsh-web-app`、`dsh-host-webserver`、`dsh-host-apiproxy`、`dsh-host-frontend-static`、`dsh-client-*` 系列

### 14.1 启动流程

```
dsh web
  → CLI 解析 → profile 组合（dsh-base + dsh-web-app）
  → boot() → Cordis Loader 启动插件树
  → webStartup（inject cmdlineArgs）→ 解析 --host/--port → provide webStartup
  → webserver（inject webStartup）→ WebServer listen → provide webServer
  → web-runtime → resolveLanTrust → 挂 FrontendStatic → 注册 prompt section
  → api-gateway（inject 13服务）→ ApiProxyService
  → client-modules → 扫描 dsh.client 包 → 组合 graph → tapIndex 注入 __DSH_BOOT__
  → client-hmr → 轮询 bundle + /plugins/events SSE
  → client-connection → /api 路由 + WebSocket upgrade
  → loader.await() → assertEntriesActivated()
  → 打印 URL: dsh web: http://127.0.0.1:<port>
```

### 14.2 __DSH_BOOT__ 注入

> 🔑 index.html 本身不含 `__DSH_BOOT__`。这个 script 标签是**运行时由 host 注入**的。

`ClientModuleRegistry` 扫描所有 `dsh.client` 声明包，组合 boot graph（含 entries + rev），通过 `tapIndex` 在每次 index 响应时 `injectBootManifest(html, graph)` 插入 `<head>`。

### 14.3 前端引导

`AppWebEntry.run()`：
1. `parseBootManifest(__DSH_BOOT__)` → modules + plugins
2. `new ClientModuleSystem()` → 静态模块表（react/cordis/ui-primitives）
3. `loader.internal = moduleSystem` → 把 module system 注入为 Loader 的 internal contract
4. `loader.create({name})` 创建每个 graph entry → `loader.await()`
5. settled → React UI 渲染

### 14.4 Client-Plugin HMR

双端架构：
- **Node 半边**：interval stat-poll bundle 文件 → 发现变化 → `rebuilt()` → SSE 推送
- **Browser 半边**：`EventSource("/plugins/events")` → 收到 rebuilt frame → invalidate + prefetch + 销毁旧 fiber + 重新激活

> 🔑 HMR 链路只在 `pnpm run dev:web`（vite build --watch）时真正生效。生产构建无 watcher，链路空闲。

### 14.5 API Proxy

四象限 RPC 消息模型：`client-request` / `server-response` / `server-request`(push) / `rpcReceipt`

传输层：HTTP POST 上行 + WebSocket 下行 + SSE（测试同构）。

`toFetchHandler(api)` 把 ApiProxy 包装成纯 `fetch(Request)` 函数。`AbstractApiClient` 是 browser 侧客户端。`InProcessApiClient` 永不触网（测试/headless 同构点）。

> 🔑 信任围栏：`isTrustedApiRequest()` 防 DNS rebinding + 跨站。特权方法 pin loopback。

### 14.6 会话投影到前端

**MuxFrame**（session 级）：`session/event`（含 tool-event view）、`session/subscribed`、`session/jobs`、`session/projection`、`approval/*`、`question/*`

**HostFrame**（全局）：`host/session-added`、`host/workspace-changed`、`host/remote-event`

`events.mux()` 创建 FrameQueue → 基线回放 → 事件订阅 → `viewFor()` 投影 tool-event view → push 到 queue。

### 14.7 Agent Preset 呈现

host plane vs preset plane 归属决策：
- **移到 preset**：工具包、plan-mode、compaction（每个 agent 各自组装）
- **保留 host plane**：jobs/subagent registry（进程级单例）、skill registry（分层）、goal service、token meter

判据：一个 Service 若被 row 外部 realm 读取 → host plane；若只被 preset 内部消费 → preset plane。

---

## 15. 沙箱与文件系统

> 源码：`dsh-sandbox-local`、`dsh-fs-local`、`dsh-fs-sandbox`、`dsh-bash-sandbox`、`dsh-tool-bash`、`dsh-user-approval`、`dsh-permission-presets`

### 15.1 文件沙箱模式

三模式：`read-only` / `workspace-write` / `danger-full-access`

> 🔑 **`writableRoots()` 是唯一事实源**：Seatbelt（macOS）和 fs fence 共享，杜绝"bash 能写但 write 工具不能写"的不对称。

`canonicalPath` 做符号链接规范化（realpath），防止通过 symlink 逃逸。

### 15.2 sandbox-policy 决策

`resolve()` 三段优先级链：
1. **approved**（一次性升级授权）
2. **session override**（会话级覆盖）
3. **deployment default**（配置默认值）

会话日志即存储：fold 机制从 session events 重建当前策略状态。

### 15.3 Bash 沙箱拦截

`SandboxBashExecutor.run()` 通过 `ctx.sandbox.confine(["bash", "-c", command])` 包裹 argv。

> 🔑 **不解析命令内容**，靠内核拒绝 + stderr 方言分类。命令执行被沙箱限制在 writable roots 内。

### 15.4 文件系统后端

`LocalFileSystem` 实现：
- `resolve()` — realpath 身份
- `writeText()`/`editText()` — 原子写（stage-then-publish + per-key FIFO 锁）
- Win32 DACL 复制（保持文件权限继承）
- 版本令牌推导：`dev:ino:mtimeNs:ctimeNs`

### 15.5 沙箱拦截点

> 🔑 **`SandboxedFileSystem.checkedTarget()` 是 FS 心脏**：
- `read-only` → 直接拒绝写操作
- `workspace-write` → 重新 canonical + 遏制检查（路径在 writable roots 内？）
- `danger-full-access` → 放行
- 读操作不拦截

### 15.6 Approval 机制

双策略：`ask`（需用户批准）/ `never`（确定性拒绝）

`request()` 阻塞式审批。`never` 在 waterfall 派发前确定性拒绝。

审计对（asked + decided turn 包裹）。`allowed-once` 仅一次性授权。

### 15.7 Permission Presets

Preset 不覆盖旋钮，而是通过 `setSandboxMode`/`setApprovalPolicy` 规范 setter 写入。三旋钮并列持久化。`derive` 反推 `custom`。

预设组合：
- `read-only`：sandbox=read-only, approval=ask
- `workspace-write`：sandbox=workspace-write, approval=ask
- `danger-full-access`：sandbox=danger-full-access, approval=never

### 15.8 sandbox_permissions 一次性升级

`WIDER_MODES` 严格更宽阶梯：`read-only < workspace-write < danger-full-access`。

参数配对验证。`approveEscalation` 有序 fail-closed 序列。结构性 approver 解耦。

### 15.9 通用工具审批门控

> 🔑 **两条审批通道**：
1. `tools/pre-execute` 通用门控（插件可插拔，调度器在执行前调用 `serviceAsk`）
2. `sandbox_permissions` 升级审批（工具 execute 内部主动发起）

两者都最终走 `ctx.approval.request()`。

### 15.10 fail-closed 贯穿

- 无可用沙箱后端 → `SANDBOX_UNAVAILABLE`
- 无 answerer → `unavailable`
- runner 崩溃 → 拒绝执行
- 禁止静默 unconfined passthrough

---

## 16. Skill 系统

> 源码：`dsh-skill`、`dsh-skill-filesystem`、`dsh-tool-skill`

### 16.1 数据模型

三层递进：`SkillSummary` → `SkillCandidate`（带 rank/locator）→ `SkillDefinition`（含 content body）

> 🔑 目录列表只加载 Summary/Candidate（轻量），完整 body 仅在 `get()` 时按需加载。

name 严格 kebab-case：`/^[a-z0-9]+(?:-[a-z0-9]+)*$/`

`SkillInvocationPolicy`：`modelInvocable` / `userInvocable` 控制谁能调用。

### 16.2 分层注册表

`SkillRegistry` 用 `ScopedLayers` 实现 host + per-scope 结构，与工具注册表相同。

rank 优先级体系：
```
PROJECT_DSH(100) > PROJECT_AGENTS(200) > RUNTIME(250) > CUSTOM(300) > USER_DSH(400) > USER_AGENTS(500) > BUNDLED(600)
```

> 🔑 跨层时最近层直接覆盖，不走 rank。同层内 rank 决定冲突赢家。

### 16.3 Skill Filesystem

根目录发现按优先级：项目 `.dsh/skills` / `.agents/skills` → 自定义目录 → 用户 `~/.dsh/skills` / `~/.agents/skills` → 打包 skill。

skill 文件格式：YAML frontmatter + Markdown body。

frontmatter 字段：`name`、`description`、`whenToUse`、`disable-model-invocation`、`user-invocable`。

### 16.4 加载流程

1. `skill` 工具调用 → `isSkillName` 校验
2. `ctx.skills.list(lookup)` 获取摘要 → 找匹配 name
3. `isModelInvocable` 校验
4. `ctx.skills.get(name, lookup)` 加载完整 body
5. 再次 `isModelInvocable` 校验（防止 list 与 get 之间策略变更）
6. 返回 `{name, provider, content}`

### 16.5 available_skills 注入

> 🔑 **catalog 不是静态 system prompt，而是持久化的 user 消息**（`<system-reminder>` 形式），通过 SHA-256 digest 去重避免无谓重发。

`agent/pre-step` 监听器：
1. 可见性匹配检查（当前 agent 解析到的正是本插件注册的工具实例）
2. `ctx.skills.snapshot()` 获取目录快照
3. 不完整 → 不发
4. digest 与历史可见 digest 比较 → 相同则不发
5. 不同则渲染新 catalog 消息，替换或追加

### 16.6 HMR 热重载

`SkillWatchManager` 基于 chokidar + 引用计数 + microtask 防抖失效。

两种 watcher 模式：root 模式（chokidar depth:1）/ ancestor 模式（fs.watchFile 轮询）。

不健康 watcher 自愈：出错标记 unhealthy → `scheduleRewatch` 重试。

主机变更同步：`edit`/`write` 工具修改了被监视路径 → 立即失效。

---

## 17. Jobs 后台任务

> 源码：`dsh-jobs`（抽象 seam）、`dsh-jobs-local`（实现）、`dsh-tool-jobs`、`dsh-schedule`

### 17.1 数据模型

```typescript
type JobStatus = 'running' | 'stopping' | 'completed' | 'killed' | 'failed';
```

**producer/controller 分离**：
- `JobStart`（producer 提交）：`kind`、`label`、`run(): JobHooks`
- `JobHooks`（producer 返回）：`cancel()`、`done: Promise<JobOutcome>`、`readOutput?()`
- `JobSnapshot`（只读快照）：含 `reported` 字段（标记是否已报告为终态）

### 17.2 生命周期

```
running ──cancel──> stopping ──> killed (终态)
   ├─done(normal)──────> completed (终态)
   └─done(error)────────> failed (终态)
```

> 🔑 **first-wins 语义**：teardown 的强制失败也能覆盖迟到的 producer 正常 settle，保证只有一个终态记录。

### 17.3 后台执行

> 🔑 `dsh-tool-jobs` 注册的是 **controller**（声明"我能读和停 job"）。实际启动 job 的 producer 在别处。`start()` 要求有 controller 服务该 owner 才允许启动——**确保"能启动就一定能收集和停止"**。

并发限制：`maxConcurrentJobsPerOwner` 默认 10。

### 17.4 完成通知策略

> 🔑 **有策略的消息投递**：
- busy owner → 注入到下一步（`inject`）
- idle owner + wakeup 模式 → 开启新 turn（`followup`）
- 连续唤醒预算（默认 3）防止自激链
- `agent/inbox/claimed` 监听：用户真实输入打断时重置计数

### 17.5 Schedule 机制

`dsh-schedule` 提供三种定时规则：
- `after` — 延迟一次性
- `at` — 绝对时间一次性
- `every` — 固定频率循环（下限 300 秒）

> 🔑 **持久化基于 session event log**：所有变更是 `schedule/change` 事件，`foldScheduleEvents` 重放。跨 session 恢复/分叉可重放。

`session-local` 投递：提醒只在原始 session 存活时按时触发，否则变 overdue 直到 session 恢复。

---

## 18. 完整路线图（全部已展开）

| # | 点 | 关键词 | 状态 |
|---|---|---|---|
| 1 | 启动与引导 | dsh CLI → profile → patch 叠加 → Loader → 插件树 | ✅ |
| 2 | Cordis 框架内核 | Context / fiber / Service / 事件 / waterfall | ✅ |
| 3 | 会话系统 (Session) | event-sourced、projection、jsonl 持久化 | ✅ §9 |
| 4 | Agent 与 Agent Loop | turn / step / 消息流 | ✅ §8 |
| 5 | System Prompt 动态组装 | sections / contexts / variables | ✅ §4 |
| 6 | LLM 适配层 | provider / model / streaming / cache | ✅ §10 |
| 7 | 工具系统 (Tools) | 注册 / dispatch / schema | ✅ §6 |
| 8 | 子代理 (Subagent) | background / fork | ✅ §11 |
| 9 | Goal 系统 | 长目标 / round driver | ✅ §12 |
| 10 | Workflow | JS 脚本编排多 agent | ✅ §13 |
| 11 | Web GUI | web-app / HMR / client-plugin | ✅ §14 |
| 12 | 沙箱与文件系统 | sandbox / fs-local / approval | ✅ §15 |
| 13 | Skill 系统 | 加载 / filesystem | ✅ §16 |
| 14 | Jobs 后台任务 | worker-thread / schedule | ✅ §17 |

### 关键包索引

| 包 | 作用 |
|---|---|
| `@deepseek-ai/cordis` | 元框架（Context/fiber/Service/events） |
| `@deepseek-ai/cordis-plugin-loader` | 插件加载器 |
| `@deepseek-ai/cordis-plugin-hmr` | 热模块替换 |
| `@deepseek-ai/cordis-plugin-include` | YAML/JSON 配置 include |
| `@deepseek-ai/dsh-app-boot` | profile 解析、patch 叠加、boot |
| `@deepseek-ai/dsh-base` | 核心 patch 层（~30 个基础插件行） |
| `@deepseek-ai/dsh-system-prompt` | 动态 prompt 组装引擎 |
| `@deepseek-ai/dsh-agent-loop` | turn/step 循环、buildRequest |
| `@deepseek-ai/dsh-agent` | Agent 创建、inbox、dispatch |
| `@deepseek-ai/dsh-session` | 事件溯源会话、headerEquals |
| `@deepseek-ai/dsh-llm` | LLM 适配层、provider 路由 |
| `@deepseek-ai/dsh-llm-pi-ai` | pi-ai 多 provider adapter |
| `@deepseek-ai/dsh-tools` | ToolRuntime、defineTool、调度 |
| `@deepseek-ai/dsh-tool-fs` | read/write/edit 工具 |
| `@deepseek-ai/dsh-tool-bash` | bash 工具 |
| `@deepseek-ai/dsh-tool-subagent` | 子代理工具 |
| `@deepseek-ai/dsh-tool-workflow` | workflow 工具 |
| `@deepseek-ai/dsh-tool-goal` | goal 工具 |
| `@deepseek-ai/dsh-tool-ralph` | ralph 循环工具 |
| `@deepseek-ai/dsh-scope` | ScopedLayers 作用域抽象 |
| `@deepseek-ai/dsh-token-meter` | token 计量（含 cache 追踪） |
| `@deepseek-ai/dsh-sandbox-local` | 文件沙箱 |
| `@deepseek-ai/dsh-web-app` | Web GUI 应用 |
| `@deepseek-ai/dsh-skill` | Skill 系统 |
| `@deepseek-ai/dsh-jobs-local` | 后台任务 |

---

*文档生成于源码版本 `@deepseek-ai/dsh@0.1.0-rc.6`*
