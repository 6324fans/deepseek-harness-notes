# DSH Bash/命令执行链 源码分析报告

> 分析对象：`@deepseek-ai/dsh-bash-local`、`@deepseek-ai/dsh-bash-sandbox`、`@deepseek-ai/dsh-tool-bash` 三个包，并辅以依赖包 `@deepseek-ai/dsh-sandbox`、`@deepseek-ai/dsh-sandbox-policy`、`@deepseek-ai/dsh-shell` 的类型定义。

---

## 0. 三层架构总览

DSH 的命令执行采用**三层职责分离**设计，自上而下为：

| 层 | 包 | 角色 | 关键注册 |
|---|---|---|---|
| 工具层（LLM 入口） | `dsh-tool-bash` | 注册名为 `bash` 的工具，面向模型；负责入参校验、审批升级（escalation）、输出渲染、后台作业 | `ctx.tools` |
| 沙箱层（拦截/审批） | `dsh-bash-sandbox` | 注册为 `ctx.shell`，替换 local 执行器；把 `bash -c` 包裹成受限 argv，做拒绝判定与 runner 失败判定 | `ctx.shell` |
| 执行层（底层后端） | `dsh-bash-local` | 通过 `ctx.subprocess` 真正 spawn 子进程；管理超时、输出捕获、进程组生命周期 | `ctx.subprocess` |

> 🔑 **继承关系**：`SandboxBashExecutor extends LocalBashExecutor`（`dsh-bash-sandbox` 第110行）。沙箱层不是"包装"而是"子类化"——它复用 local 的全部进程机制（超时、输出、kill），只在 `run`/`start` 之前把 argv 换成沙箱 runner 的 argv，并在结算（settlement）时附加 `sandbox` 事实。

**典型前台调用链**（行号见各包小节）：

```
模型调用 bash 工具
  └─ dsh-tool-bash execute()            [tool-bash 第386行]
       ├─ validateBashArgs()            [tool-bash 第119行]
       ├─ resolveSandboxPolicy()        [tool-bash 第226行]  ← ctx.sandboxPolicy.resolve(session)
       ├─ approveBashEscalation()?      [tool-bash 第238行]  ← 仅当传了 sandbox_permissions
       ├─ ctx.shell.resolve(request)    [tool-bash 第429行]  ← SandboxBashExecutor.resolve() 注入 sandboxPolicy
       └─ ctx.shell.run(spec)           [tool-bash 第429行]  ← SandboxBashExecutor.run()
            ├─ confine(command, policy) [bash-sandbox 第228行] ← ctx.sandbox.confine(["bash","-c",cmd], policy)
            ├─ super.runArgv(spec, confined.argv)             ← LocalBashExecutor.runArgv() [bash-local 第228行]
            │    └─ ctx.subprocess.spawn(spawnSpec(...))      [bash-local 第236行]
            ├─ classifyRunnerFailure()  [bash-sandbox 第165行]
            └─ return {...result, sandbox:{mode,denied,enforcement}}  [bash-sandbox 第167行]
```

---

## 1. `@deepseek-ai/dsh-bash-local`（底层执行后端）

**文件**：`lib/index.js`（333 行）、`lib/types/index.d.ts`（115 行）

### 1.1 核心导出

`lib/index.js` 第333行：
```js
export { ENV_OVERRIDES, LocalBashExecutor, LocalBashExecutor as default, assertServiceableBashConfig };
```

- `LocalBashExecutor`（默认导出）：`ctx.shell` 能力的本地实现，继承 `ShellExecutor`（来自 `dsh-shell`）。
- `ENV_OVERRIDES`：模型友好的环境变量覆盖。
- `assertServiceableBashConfig`：配置合法性校验。

### 1.2 关键常量与函数

#### `ENV_OVERRIDES`（第81–86行）

```js
const ENV_OVERRIDES = {
    NO_COLOR: "1",       // 禁用颜色
    TERM: "dumb",        // 哑终端（与 Claude Code 一致）
    PAGER: "cat",        // 禁用分页器
    GIT_PAGER: "cat"
};
```

> 🔑 注释（第74–80行）说明这是"bash-tool 策略"——先合并进 spawn 的显式 env，因此**受信调用者自己的 env 条目仍能覆盖它**；subprocess 服务还独立做凭据清洗。合并顺序见 `spawnSpec`（第195–199行）：`{...ENV_OVERRIDES, ...spec.env, ...spec.dshEnv}`，即 `dshEnv`（托管变量）最后合并、优先级最高。

#### 默认预算（第88、90行）

- `DEFAULT_GRACE_MS = 3e3`（3 秒，SIGTERM→SIGKILL 宽限，注释称与 OpenCode 一致）
- `DEFAULT_MAX_SPILL_BYTES = 64 * 1024 * 1024`（64 MiB，每流溢出文件上限）

#### `finalOutput(reader)`（第92–99行）

把 collect-mode 读取器投影成最终 `CollectedOutput` 形状：`text`、`truncated`（= `read.lossy`）、可选 `spillPath`。

#### `assertServiceableBashConfig(config)`（第111–119行）

schema 表达不了"正且有限"以及 `graceMs` 的 timer 上限，所以在写入处拒绝：逐字段 `assertPositiveFinite`（timeoutMs、maxTimeoutMs、maxOutputBytes、maxSpillBytes、graceMs），并要求 `graceMs <= MAX_TIMER_DELAY_MS`。

### 1.3 `LocalBashExecutor` 类（第127–331行）

#### 静态成员

- `static inject = ["subprocess"]`（第128行）：依赖 `ctx.subprocess` 服务。
- `static Config`（第129–136行）schema：
  - `cwd: string`
  - `timeoutMs: number`（默认 `12e4` = 120 秒）
  - `maxTimeoutMs: number`（默认 `6e5` = 600 秒）
  - `maxOutputBytes: number`（默认 `64e3` = 64 KiB，**内存内**每流上限，超出溢出到文件）
  - `maxSpillBytes: number`（默认 64 MiB）
  - `graceMs: number`（默认 3 秒）

#### 构造函数（第143–155行）

校验 config，记 `this.source`（当前权威配置来源），并通过 `installSettingsSection`（来自 `dsh-settings`）注册到 `SHELL_SETTINGS_NAMESPACE`，支持运行时重载配置——但**重载只换 source，不影响已在跑的进程**（进程在 composition teardown 时仍被 kill 并 join）。

#### `get config`（第140–142行）

返回 `this.source()`，即当前权威配置。

#### `resolve(request)`（第163–178行）— 请求→规格

> 🔑 这是把"模型请求"变成"完整规格"的步骤，`run`/`start` 收到的是**已填好默认值、已封顶**的 spec，永不再次 default。

- `timeoutMs = clampTimeout(request.timeoutMs, this.config.timeoutMs, this.config.maxTimeoutMs, ...)`（第164行）：用 `dsh-timeout` 的 `clampTimeout` 把调用方传入的超时夹在 `[config.timeoutMs, config.maxTimeoutMs]` 区间。
- `workdir = request.workdir ?? this.config.cwd ?? process.cwd()`（第169行）。
- `sandboxPolicy: request.sandboxPolicy`（第176行）：**原样透传**——local 执行器不消费它，只是搬运，让子类（沙箱层）能读到。

#### `spawnSpec(spec, argv, stdoutMaxBytes, signal)`（第180–201行）— 映射到 subprocess spawn

```js
{
    argv,
    cwd: spec.workdir,
    stdio: {
        stdin: spec.stdin !== void 0 ? { data: spec.stdin } : "ignore",
        stdout: collect(stdoutMaxBytes),     // 有界捕获 + 溢出
        stderr: collect(this.config.maxOutputBytes)
    },
    graceMs: this.config.graceMs,
    signal,
    env: { ...ENV_OVERRIDES, ...spec.env, ...spec.dshEnv }
}
```

其中 `collect(maxBytes) = { maxBytes, spill: { maxBytes: this.config.maxSpillBytes } }`（第181–184行）。

#### `run(spec)`（第213–219行）— 前台执行

直接调用 `runArgv(spec, ["bash", "-c", spec.command])`（第214–218行）。

> 🔑 **注意**：`run` 用的是 `["bash","-c",command]` 这个 argv。沙箱层会**替换**这个 argv（见第2.3节）。

#### `runArgv(spec, argv)`（第228–255行）— 前台生命周期核心

这是**子类复用的边界方法**。逻辑：

1. 第235行：`deadline(spec.signal, spec.timeoutMs, "BASH_TIMEOUT")`——用 `dsh-timeout` 创建一个融合了调用方 `AbortSignal` 与超时的 fused deadline（disposable，用 `__addDisposableResource` 注册到 `env_1`，第234–253行的 try/catch/finally 用 `__disposeResources` 保证释放）。
2. 第236行：`this.ctx.subprocess.spawn(this.spawnSpec(spec, argv, spec.stdoutMaxBytes, d.signal))`——真正 spawn。
3. 第237行：`await handle.done`。
4. 第238行：`LocalBashExecutor.collected(handle)` 取出 collect 读取器。
5. 第239行：`timedOut = timeoutOf(d.signal, "BASH_TIMEOUT") !== void 0`。
6. 第240行：`aborted = d.signal.aborted && !timedOut`（超时与 abort 互斥——fused deadline 只报"首个原因"）。
7. 第241–248行：返回 `{...outcome, timedOut, aborted, timeoutMs, stdout: finalOutput(...), stderr: finalOutput(...)}`。

> 🔑 **timeoutMs 如何用**：不是用 `setTimeout`，而是 `deadline()` 产出一个 `AbortSignal`，喂给 `ctx.subprocess.spawn` 的 `signal` 字段（第194行）。subprocess 服务在信号触发时走 SIGTERM→SIGKILL 升级（`graceMs`）。`timedOut`/`aborted` 的区分由 `timeoutOf` 完成。

#### `start(spec)` / `startArgv(spec, argv)`（第256–319行）— 后台执行

`start` 调 `startArgv(spec, ["bash","-c",spec.command])`。`startArgv` 构造 `ShellProcess` 句柄（第283–318行）：

- `status: "running"`（第284行）。
- `done` Promise（第287–296行）：
  - resolve 分支：根据 `signal?.aborted` 或 `outcome.signal !== null` 判 `killed`/`completed`，盖 exitCode/signal，调 `this.onProcessDone(proc, stderr, false)`。
  - reject 分支：`status = "killed"`，记 `spawnFailureNote`，调 `this.onProcessDone(proc, note, true, error)`。
- `readOutput()`（第297–310行）：**消费式**增量读取（stdoutOffset/stderrOffset 推进），把 stderr 放进 `[stderr]` 段，返回 `{delta, lossy, stdoutSpillPath?, stderrSpillPath?}`。
- `kill()`（第311–316行）：幂等，调 `running.terminate()`。

> 🔑 后台进程**无超时**（`startArgv` 第273行用 `spec.signal` 而非 deadline），与 `dsh-shell` 类型注释一致："background processes have no executor timeout"。

#### `onProcessDone(...)`（第330行）— 结算钩子

```js
onProcessDone(_proc, _stderr, _spawnFailed, _spawnError) {}
```

> 🔑 基类**空实现**，留给子类（沙箱层）覆盖。在 `done` Promise resolve/reject **之后、`done` 真正 settle 之前**被调用，让子类能附挂执行事实（如 `sandbox`）。这是沙箱层注入 `proc.sandbox` 的挂载点。

### 1.4 数据结构（`lib/types/index.d.ts`）

- `Config`（第29–42行）：所有字段可选。
- `ResolvedConfig`（第44行）：`Required<Omit<Config,'cwd'>> & Pick<Config,'cwd'>`。
- `LocalBashExecutor`（第61–113行）：`resolve`、私有 `spawnSpec`/`collected`、`run`、`runArgv`、`start`、`startArgv`、`onProcessDone`。

---

## 2. `@deepseek-ai/dsh-bash-sandbox`（沙箱拦截/审批层）

**文件**：`lib/index.js`（237 行）、`lib/types/index.d.ts`（67 行）、`lib/types/helpers.d.ts`（55 行）、`lib/types/invariant.d.ts`（16 行）

### 2.1 核心导出

`lib/index.js` 第237行：
```js
export { SandboxBashExecutor, SandboxBashExecutor as default };
```

> 🔑 沙箱层**不导出 helpers**——`helpers` 是内部模块（`#region lib/types/helpers.js`，第4行），仅在同文件内被 `SandboxBashExecutor` 使用。`.d.ts` 里有声明但 `index.js` 未 re-export，符合"内部"定位。

### 2.2 helpers 模块（第4–92行）— 拒绝/失败分类

这是回答"如何判定拒绝"的核心。

#### `EXECUTABLE_SPAWN_CODES`（第11行）

```js
const EXECUTABLE_SPAWN_CODES = new Set(["EACCES", "ENOENT"]);
```

Node spawn 失败码中能确凿表明"可执行文件解析/权限失败"的集合。

#### `isUsableWorkdir(path)`（第13–21行）

`statSync(path).isDirectory()` 且 `accessSync(path, constants.X_OK)`——判定调用方拥有的 cwd 是否可进入。

#### `isRunnerSpawnFailure(error, runnerProgram, workdir)`（第36–46行）

> 🔑 **关键归因逻辑**：判定一个 spawn 拒绝是否确凿是"沙箱 runner 本身启动失败"（而非命令失败）。规则：
> 1. `runnerProgram` 必须有值且 `workdir` 可用（第37行）——先排除 cwd 问题，否则无法把 ENOENT/EACCES 归因到 runner。
> 2. error 必须是对象（第38行）。
> 3. `code` 必须在 `EXECUTABLE_SPAWN_CODES` 中（第40行）。
> 4. 若 `path` 未定义：要求 `syscall === "spawn <runnerProgram>"`（第43行）；若 `path` 定义：要求 `path === runnerProgram` 且 `syscall` 为 `"spawn"` 或精确形式（第44–45行）。
>
> 注释（第22–35行）强调：workdir 在**分类时**检查而非与 spawn 原子检查，并发路径替换可能改变归因，但**不会允许一次未受限的执行**（fail-closed 原则）。

#### `classifyDenial(result, signatures)`（第53–55行）

调 `matchesSignature(result.exitCode, result.stderr.text, signatures)`——用后端的"拒绝方言"子串匹配 stderr。

#### `classifyRunnerFailure(exitCode, stderr, rules)`（第66–79行）

> 🔑 **结构化 runner 失败规则匹配**：对每条 `RunnerFailureRule`：
> 1. 要求 `exitCode` 非零非 null（第67行）。
> 2. 若规则有 `allowedExitCodes` 且不含当前 exitCode，跳过（第70行）。
> 3. 收集 `informationalLines`（小写化精确集合，第71行）和 `fatalSignatures`（小写化，第72行）。
> 4. 逐行扫描 stderr：跳过 informational 行（第75行），命中任一 fatal signature 则返回 `{detail: line}`（第76行）。
>
> 区分"runner 失败"（命令没跑）与"拒绝"（沙箱生效、拦下了命令）——前者抛 `SandboxUnavailableError`，后者只标记 `denied`。

#### `matchesSignature(exitCode, stderr, signatures)`（第87–91行）

非零退出 + stderr（小写）包含任一 signature（小写）→ true。信号终止（`exitCode === null`）不算拒绝。

### 2.3 `SandboxBashExecutor` 类（第110–235行）

```js
var SandboxBashExecutor = class extends LocalBashExecutor {
    static inject = ["subprocess", "sandbox", "sandboxPolicy"];  // 第111-115行
```

> 🔑 比 local 多注入 `sandbox`（`ctx.sandbox`，即 `SandboxProvider`）和 `sandboxPolicy`（`ctx.sandboxPolicy`，即 `SandboxPolicyService`）。

#### `mode`（第116行）与 `processFacts`（第123行）

- `mode = ctx.sandboxPolicy.defaultMode`（构造函数第126行）：部署默认模式。
- `processFacts = new Map()`：**逐进程**保存沙箱事实。注释（第117–122行）解释：多个并发调用可能使用不同 enforcement/dialect，共享一个"最新 wrap"值会把进程归错，所以必须 per-process 保留。

#### `get sandboxMode()`（第129–131行）

返回 `this.mode`——工具层读这个"能力事实"判断是否挂了沙箱执行器。

#### `resolve(request)`（第137–142行）

```js
resolve(request) {
    return {
        ...super.resolve(request),
        sandboxPolicy: request.sandboxPolicy ?? this.ctx.sandboxPolicy.resolve()
    };
}
```

> 🔑 **sandbox_permissions 如何传到沙箱层**：工具层在 `execute` 里把 resolved policy 放进 `request.sandboxPolicy`（见第3.4节），这里**优先用 request 自带的**；若调用方没传（直接调底层），回退到 `ctx.sandboxPolicy.resolve()`（部署策略）。`super.resolve` 已经填好 workdir/timeout，这里只补 `sandboxPolicy`。

#### `async run(spec)`（第143–175行）— 前台受限执行

```js
async run(spec) {
    const policy = spec.sandboxPolicy;
    const { mode } = policy;
    if (mode === "danger-full-access") return {              // 第146行
        ...await super.run(spec),
        sandbox: { mode, denied: false }
    };
    const confined = this.confine(spec.command, {...policy, mode});  // 第153行
    let result;
    try {
        result = await this.runArgv(spec, confined.argv);    // 第159行
    } catch (error) {
        if (spec.signal?.aborted === true) spec.signal.throwIfAborted();  // 第161行
        if (isRunnerSpawnFailure(error, confined.argv[0], spec.workdir))  // 第162行
            throw new SandboxUnavailableError(mode, String(error));
        throw error;
    }
    const runnerFailure = classifyRunnerFailure(             // 第165行
        result.exitCode, result.stderr.text, confined.runnerFailureRules);
    if (runnerFailure !== void 0) throw new SandboxUnavailableError(mode, runnerFailure.detail);  // 第166行
    return {
        ...result,
        sandbox: {
            mode,
            denied: classifyDenial(result, confined.denialSignatures),  // 第171行
            enforcement: confined.enforcement
        }
    };
}
```

> 🔑 **是否在执行前检查命令？否。** 沙箱层**不做命令字符串解析或预审批**——它不检查"这是不是文件写入类命令"。它做的是：
> 1. **danger-full-access 短路**（第146行）：直接 `super.run`（local 原生执行），返回 `denied: false`。这是"批准的全访问"模式。
> 2. **confine**（第153行）：把 `bash -c <command>` 整体交给 `ctx.sandbox.confine`，由沙箱后端（Seatbelt/Landlock/bwrap/Windows-ACL）把它包成受限 argv。**拦截发生在 OS 内核/runner 层**，不是在 JS 层解析命令。
> 3. **执行**（第159行）：用 local 的 `runArgv` 跑受限 argv。
> 4. **结算分类**：spawn 抛错→`isRunnerSpawnFailure` 判定是否 runner 启动失败（第162行）；跑完但非零→`classifyRunnerFailure` 判定是否 runner 失败（第165行）；否则用 `classifyDenial` 判定是否被拒绝（第171行）。
>
> 所以"文件写入类命令"的拦截是**后端在内核层做的**（如 Seatbelt 拒写、Landlock EACCES），JS 层只负责**识别后端报告的拒绝信号**（`denialSignatures`）。

#### `start(spec)`（第176–201行）— 后台受限执行

与 `run` 对称：danger-full-access 直接 `super.start`；否则 confine 后 `startArgv`，并把 `{mode, enforcement, denialSignatures, runnerFailureRules, runnerProgram, workdir}` 存进 `processFacts`（第192–199行），供 `onProcessDone` 用。

#### `onProcessDone(proc, stderr, spawnFailed, spawnError)`（第206–219行）— 后台结算

覆盖基类空实现。从 `processFacts` 取出事实（第207行），删除（第209行），计算 `runnerFailed`（第210行）：

- 若 `spawnFailed`：用 `isRunnerSpawnFailure(spawnError, runnerProgram, workdir)`。
- 否则：用 `classifyRunnerFailure(proc.exitCode, stderr, runnerFailureRules)`。

然后盖 `proc.sandbox`（第211–216行）：
```js
proc.sandbox = {
    mode: facts.mode,
    denied: !runnerFailed && matchesSignature(proc.exitCode, stderr, facts.denialSignatures),
    enforcement: facts.enforcement,
    ...runnerFailed ? { runnerFailed } : {}
};
```

> 🔑 **runner 失败时 `denied` 为 false**（因为 `!runnerFailed` 短路）——runner 失败意味着命令根本没跑，不是被拒。信号死亡（exitCode null）也不算拒绝（`matchesSignature` 第88行返回 false）。

#### `confine(command, policy)`（第228–234行）

```js
confine(command, policy) {
    return this.ctx.sandbox.confine(["bash", "-c", command], policy);
}
```

> 🔑 这是与 `dsh-sandbox` 的衔接点：把 shell 命令包成 `["bash","-c",command]` argv（**不是 shell 字符串**——`SandboxProvider.confine` 的契约要求 argv，见 `dsh-sandbox` 类型第133–135行注释），交给后端包装。返回 `ConfinedArgv`（含 `argv`、`enforcement`、`denialSignatures`、`runnerFailureRules`）。

### 2.4 与 sandbox-policy 的联动

> 🔑 **是否对每个命令做审批判定？是，但分两处**：
>
> 1. **策略解析**（每调用）：`resolve`（第140行）调 `ctx.sandboxPolicy.resolve()`，`SandboxPolicyService.resolve`（`dsh-sandbox-policy` 第75行）按优先级合并：显式批准 mode > session 的 `sandbox/mode` 事件 > 部署默认。返回 `SandboxExecutionPolicy {mode, workspaceRoot, sessionId?}`。
> 2. **升级审批**（仅当模型请求）：在**工具层**（`dsh-tool-bash`）做，不在沙箱层。沙箱层只消费已 resolved 的 policy。
>
> 沙箱层与 policy 的关系：policy 决定 mode（`read-only`/`workspace-write`/`danger-full-access`）和 workspaceRoot；沙箱后端用 mode 选 enforcement profile，用 workspaceRoot 算可写根（`dsh-sandbox` 的 `writableRoots`，第154–161行：`workspace-write` 允许 workspaceRoot + `/tmp` + `os.tmpdir()`）。

### 2.5 数据结构

- `Config = LocalConfig`（`index.d.ts` 第22行）：沙箱执行器复用 local 的配置旋钮；**策略不在这里**（注释第15–21行）。
- `SandboxBashExecutor`（第30–65行）：`inject`、`mode`、`processFacts`、`sandboxMode`、`resolve`、`run`、`start`、`onProcessDone`、私有 `confine`。
- `helpers.d.ts`：`isRunnerSpawnFailure`、`RunnerFailureMatch{detail}`、`classifyDenial`、`classifyRunnerFailure`、`matchesSignature`。
- `invariant.d.ts`：companion 插件 `bash-sandbox-invariant`（`name`、`inject`、`apply`）——包级不变量守护。

---

## 3. `@deepseek-ai/dsh-tool-bash`（LLM 工具层）

**文件**：`lib/index.js`（448 行）、`lib/types/index.d.ts`（22 行）、`lib/types/background.d.ts`（19 行）、`lib/types/render.d.ts`（38 行）、`lib/types/invariant.d.ts`（16 行）、`lib/invariant.js`（23 行）

### 3.1 核心导出

`lib/index.js` 第448行：
```js
export { Config, apply, inject, name };
```

- `name = "tool-bash"`（第110行）。
- `inject = ["tools", "shell", "systemPrompt", "shellEnv"]`（第111–116行）。
- `Config = z.object({ enableRunInBackground: z.boolean().default(true) })`（第118行）。
- `apply(ctx, config)`（第219行）：插件入口。

> 🔑 工具层**不直接依赖 sandbox / sandboxPolicy**——它通过 `ctx.shell.sandboxMode`（能力事实，第221行）探测是否挂了沙箱执行器，再按需 `ctx.get("sandboxPolicy")`（第223行）取策略服务。这是一种**能力探测式解耦**。

### 3.2 入参校验

#### `validateBashArgs(args)`（第119–124行）

```js
if (args.command.trim().length === 0) throw ...;          // 命令非空
if (args.description.trim().length === 0) throw ...;       // 描述非空
if (args.timeoutMs !== void 0 && (!Number.isFinite(...) || args.timeoutMs <= 0)) throw ...;  // 超时正有限
validateEscalationArgs(args.sandbox_permissions, args.justification);  // 第123行
```

`validateEscalationArgs`（`dsh-sandbox` 第50–54行）：`sandbox_permissions` 与 `justification` 必须成对出现，且 justification 非空句。

### 3.3 工具描述（`bashDescription`，第125–130行）

> 🔑 这是**注入到 LLM 的 bash 工具描述**（即本会话工具描述的来源）。`base`（第127行）说明：
> - 每次 `bash -c` 在**全新 shell** 中跑，cwd/变量/函数不跨调用持久——用 `workdir` 而非 `cd`。
> - 非零退出报 `[exit code: N]`。
> - 通过 `$DSH_*` 变量暴露 harness 环境事实。
> - 文件沙箱拒绝报 `[sandbox: file access denied under <mode> mode]`——是策略拒绝，不是 bug，**不要换方式重试**。
> - 长输出截断到尾部，全文存文件并报告路径。
>
> `escalationModes.length > 0` 时追加升级指导（第129行）：被拒后可**在同一轮**用 `sandbox_permissions`（最窄够用模式）+ 一句 `justification` 重试一次；若会话禁用审批提示则无此例外；被拒的升级对该命令是终局。

### 3.4 `apply()` 与 `execute()`（第219–445行）

#### 初始化（第219–258行）

- `backgroundEnabled = config.enableRunInBackground ?? true`（第220行）。
- `defaultMode = ctx.shell.sandboxMode`（第221行）：**探测能力**。若为 `undefined` → 没挂沙箱执行器。
- `escalationModes = defaultMode === void 0 ? [] : ESCALATION_TARGETS`（第222行）：`ESCALATION_TARGETS = ["workspace-write", "danger-full-access"]`（`dsh-sandbox` 第41行）。
- `sandboxPolicy = defaultMode === void 0 ? void 0 : ctx.get("sandboxPolicy")`（第223行）。
- 第224行**守卫**：若执行器 confine 但 `sandboxPolicy` 缺失 → throw（split composition fail）。
- `resolveSandboxPolicy(exec)`（第226行）：`sandboxPolicy?.resolve(exec.agent ? {session: exec.agent.session} : {})`——按调用会话解析策略。
- `approveBashEscalation(mode, justification, exec, standingPolicy)`（第238–253行）：
  - 第239行：`escalationModes.length === 0` → throw（无沙箱执行器不可升级）。
  - 第240行：`effectiveMode = standingPolicy.mode`。
  - 第241–252行：调 `approveEscalation({requestedMode, justification, effectiveMode, subject:"command"}, {approver: ctx.get("approval"), agent, callId, toolName:"bash", signal})`。

#### 工具注册（第259–445行）

`ctx.tools.register(defineTool({...}))`。工具名 `"bash"`（第260行）。

**参数 schema**（第262–296行）：
| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `command` | string | 是 | bash 命令 |
| `description` | string | 是 | 5–10 词主动语态描述 |
| `timeoutMs` | number | 否 | 超时；执行器套默认与上限 |
| `workdir` | string | 否 | 工作目录；相对路径按会话 workspace 解析 |
| `run_in_background` | boolean | 否 | 仅 `backgroundEnabled` 时暴露 |
| `sandbox_permissions` | string(enum) | 否 | 仅 `escalationModes.length>0` 时暴露；enum=`ESCALATION_TARGETS` |
| `justification` | string | 否 | 配合 sandbox_permissions |

> 🔑 **sandbox_permissions 参数如何传给沙箱层**的完整链路：
> 1. 模型传入 `sandbox_permissions` + `justification`。
> 2. `execute` 第389行：`approveBashEscalation(...)` → `approveEscalation`（`dsh-sandbox` 第92行）做**严格放宽校验**（`WIDER_MODES[effectiveMode].includes(requestedMode)`，第94行）+ 审批通道（`ctx.approval.request`）→ 返回批准的 mode（或 throw）。
> 3. 第390–393行：`policy = approvedMode === void 0 ? standingPolicy : {...standingPolicy, mode: approvedMode}`——把批准的 mode 盖到 standing policy 上。
> 4. 第401行：`...policy !== void 0 ? { sandboxPolicy: policy } : {}`——放进 `request`。
> 5. `ctx.shell.resolve(request)`（第429行）→ `SandboxBashExecutor.resolve`（第140行）取 `request.sandboxPolicy`。
> 6. `ctx.shell.run(spec)` → `SandboxBashExecutor.run`（第144行）读 `spec.sandboxPolicy.mode`。

#### `execute(args, exec)`（第386–442行）

```js
async execute(args, exec) {
    validateBashArgs(args);                                   // 第387行
    const standingPolicy = resolveSandboxPolicy(exec);         // 第388行
    const approvedMode = args.sandbox_permissions !== void 0 && args.justification !== void 0
        ? await approveBashEscalation(...)                     // 第389行
        : void 0;
    const policy = approvedMode === void 0 ? standingPolicy : {...standingPolicy, mode: approvedMode};  // 第390行
    const workdir = resolveWorkdir(args.workdir, exec, standingPolicy?.workspaceRoot);  // 第394行
    const dshEnv = ctx.shellEnv.collect(exec);                // 第395行
    const request = {                                         // 第396-402行
        command: args.command,
        ...workdir !== void 0 ? { workdir } : {},
        ...args.timeoutMs !== void 0 ? { timeoutMs: args.timeoutMs } : {},
        dshEnv,
        ...policy !== void 0 ? { sandboxPolicy: policy } : {}
    };
    if (args.run_in_background === true) { ... }              // 第403-428行 后台
    const result = await ctx.shell.run(ctx.shell.resolve({...request, signal: exec.signal}));  // 第429-432行 前台
    if (result.aborted) { throw HarnessError(TOOL_ABORTED); }  // 第433-437行
    return { kind: "foreground", ...canonicalBashResult(result) };  // 第438-441行
}
```

> 🔑 **前台执行**（第429–432行）：`ctx.shell.run(ctx.shell.resolve({...request, signal: exec.signal}))`。注意 `resolve` 与 `run` 分两步——`resolve` 填默认值，`run` 执行。`signal` 在前台才加（后台用作业取消）。
>
> 🔑 **aborted 处理**：若 `result.aborted`（调用方取消，非超时），抛 `TOOL_ABORTED`（第433–437行）。超时则不抛——超时是"命令结果"而非"基础设施错误"。

#### 后台作业（第403–428行）

```js
if (args.run_in_background === true) {
    if (!backgroundEnabled) throw ...;                        // 第404行
    const jobs = ctx.get("jobs");                             // 第405行
    if (jobs === void 0) throw ...;                           // 第406行 需要 dsh-jobs + dsh-tool-jobs
    if (exec.signal.aborted) { throw TOOL_ABORTED; }          // 第407-411行
    return {
        kind: "background",
        jobId: jobs.start({                                   // 第414行
            kind: "bash",
            label: args.command,
            ...exec.agent ? { owner: exec.agent } : {},
            run: () => {
                const proc = ctx.shell.start(ctx.shell.resolve(request));  // 第419行 ← start 而非 run
                return {
                    cancel: () => void proc.kill(),           // 第421行 ← 作业取消=kill 进程
                    done: proc.done.then(() => processOutcome(proc)),      // 第422行
                    readOutput: () => renderProcessRead(proc.readOutput(), proc.sandbox, escalationModes)  // 第423行
                };
            }
        })
    };
}
```

> 🔑 **background 作业机制**：
> - 用 `ctx.jobs.start` 注册一个 generic task（`dsh-jobs` 的 `JobController`）。
> - `run` 工厂在第419行调 `ctx.shell.start`（沙箱层 `start`，第176行）拿到 `ShellProcess`。
> - 返回三个方法：`cancel`（kill）、`done`（`proc.done` then `processOutcome`）、`readOutput`（`proc.readOutput` → `renderProcessRead`）。
> - **后台进程的取消走作业取消（`job_kill`→`cancel`→`proc.kill`），而非工具调用 signal**（注释第102–104行：id 返回后用作业取消而非 tool-call signal）。
> - `processOutcome`（第21–30行）：`killed`→`{status:"killed", detail:"signal: X"或"killed before exit"}`；否则 `completed` + `exit code: N`。非零退出报 completed 不报 failed。

### 3.5 输出渲染（render 模块，第32–99行）

#### `streamText(output)`（第39–42行）

若 `output.truncated`，追加 `\n[output truncated; full output: ${spillPath ?? "(unavailable)"}]`。

#### `renderResult(result, escalationModes=[])`（第54–74行）— 前台结果渲染

> 🔑 **渲染规则**（注释第43–53行）：
> - 非零退出**报错但不 isError**——模型决定如何反应；只有基础设施失败（spawn 错误/abort）才 isError。
> - body = stdout 文本；若 stderr 非空，加 `\n[stderr]\n<err>`（第58–61行）。
> - body 空则 `(no output)`（第62行）。
> - markers 数组（第63–73行）：
>   - `result.sandbox?.denied`（第64行）→ `sandboxDenialMarker(mode)`（`[sandbox: file access denied under <mode> mode]`）+ 若有升级模式则 `escalationHintMarker("command")`（第66行）。
>   - `result.timedOut`（第68行）→ `[timed out after ${timeoutMs}ms]`。
>   - `result.signal !== null`（第69行）→ `[killed by signal: ${signal}]`。
>   - 否则 `exitCode !== 0`（第70行）→ `[exit code: ${exitCode}]`。
> - markers 拼到 body 末尾，每行一个。

#### `renderProcessRead(read, sandbox, escalationModes=[])`（第85–98行）— 后台增量渲染

- `read.lossy`（第87行）→ 追加 `[some output was dropped from memory; full output: ...]`。
- `sandbox?.runnerFailed`（第91行）→ `[sandbox: the sandbox runner itself failed under <mode> mode — the command did not run; this is a sandbox problem, not a command failure]`。
- 否则 `sandbox?.denied`（第92行）→ denial marker + 升级 hint。

#### `canonicalBashResult(result)`（第185–206行）

把 executor DTO 拆成纯 JSON：`exitCode/signal/timedOut/aborted/timeoutMs/stdout/stderr`，可选 `sandbox{mode,denied,enforcement?,runnerFailed?}`。

#### 输出 schema（第297–385行）

`oneOf`：background（`{kind:"background", jobId}`）或 foreground（含完整退出/输出/sandbox 字段）。`render`（第381–384行）：background→`started background job ${jobId}`；foreground→`renderResult`。

### 3.6 展示（present）

- `presentBashCall`（第131–148行）：后台→`card:"generic"`；前台→`card:"terminal"`（含 cwd）。
- `presentBashResult`（第153–170行）：后台/isError→generic fenced console；前台→`parseExitStatus`（第164行，来自 `dsh-shell`）拆出 body + exit pill，返 `card:"terminal"`。

### 3.7 invariant companion（`lib/invariant.js`）

`tool-bash-invariant`（第8行），`inject: ["invariants"]`（第10行）。注释第12–15行：**无运行时不变量**——环境注册表自己在每次 mutation/read 校验所有权与值，不发布可交叉检查的快照。`install = () => {}`（第15行），`apply` 只 `ctx.invariants.register(PACKAGE_NAME, install)`（第21行）。

### 3.8 数据结构

- `Config{enableRunInBackground?}`（`index.d.ts` 第15–18行）。
- `processOutcome` 返回 `{status:'completed'|'killed', detail}`（`background.d.ts` 第15–18行）。
- `renderResult`/`renderProcessRead` 签名（`render.d.ts` 第19、30行）；re-export `parseExitStatus`/`ParsedExitStatus`（第37行）。

---

## 4. 依赖包衔接点

### 4.1 `@deepseek-ai/dsh-sandbox`（沙箱后端抽象 + 升级词表）

- `SandboxMode = 'read-only' | 'workspace-write' | 'danger-full-access'`（`index.d.ts` 第19行）。
- `SandboxProvider`（`index.js` 第194行）抽象服务，`confine(argv, policy): ConfinedArgv`（`index.d.ts` 第140行）——后端实现（Seatbelt/Landlock/bwrap/Windows-ACL）把 argv 包成受限 argv，返回 `{argv, enforcement, denialSignatures, runnerFailureRules}`。
- `WIDER_MODES`（第29–32行）：严格放宽表。`read-only`→`[workspace-write, danger-full-access]`；`workspace-write`→`[danger-full-access]`。
- `ESCALATION_TARGETS = ["workspace-write", "danger-full-access"]`（第41行）。
- `approveEscalation`（第92–111行）：严格放宽检查（第94行）→ 审批通道（第97行 `approver.request`）→ outcome 映射（`allowed-once`→mode；`rejected`/`cancelled`/`unavailable`→throw）。
- `sandboxDenialMarker`（第63行）/`escalationHintMarker`（第75行）：模型可见的标记。
- `SandboxUnavailableError`（第182–187行）：runner 不可用时抛，带 `SANDBOX_UNAVAILABLE` 码。
- `writableRoots`（第154–161行）：`workspace-write` 允许 `workspaceRoot + /tmp + os.tmpdir()`（canonical 化）。

### 4.2 `@deepseek-ai/dsh-sandbox-policy`（策略之家 `ctx.sandboxPolicy`）

- `SandboxPolicyService`（`index.d.ts` 第59行）：`defaultMode`（部署默认，默认 `read-only`）、`workspaceRoot`。
- `resolve(request?)`（第75行）：显式批准 mode > session 的 `sandbox/mode` 事件 > 部署默认；session cwd 作 workspace-write 边界。
- 注释第11–17行强调：**fs、bash、terminal 后端读同一个 resolved policy**，执行器/provider 保持 session-free。

### 4.3 `@deepseek-ai/dsh-shell`（shell 能力缝与类型）

- `ShellExecRequest`（`types.d.ts` 第34行）：调用方请求（command + 可选 workdir/timeoutMs/signal/stdin/env/dshEnv/sandboxPolicy）。
- `ShellExecSpec`（第81行）：resolved 规格（workdir/timeoutMs/stdoutMaxBytes 必填）。
- `ShellRunResult`（第107行）：前台结果（exitCode/signal/timedOut/aborted/timeoutMs/stdout/stderr/sandbox?）。
- `ShellProcess`（第152行）：后台句柄（status/exitCode/signal/done/sandbox?/readOutput/kill）。
- `ShellSandboxInfo`（第18行）：`{mode, denied, enforcement?, runnerFailed?}`。
- `parseExitStatus`（`render.d.ts` 第38行）：逆解析 `[exit code: N]`/`[killed by signal: X]` 标记。

---

## 5. 关键问题集中回答

### Q1: bash 命令执行如何被沙箱拦截？

> 🔑 **不是 JS 层预解析命令**，而是**后端在 OS 层包裹整个 `bash -c` 进程**。`SandboxBashExecutor.confine`（`bash-sandbox` 第228行）把 `["bash","-c",command]` 交给 `ctx.sandbox.confine`，后端（如 macOS `sandbox-exec`、Linux Landlock/bwrap）返回一个受限 argv（如 `["sandbox-exec","-p",profile,"bash","-c",command]`）。这个 argv 由 local 的 `runArgv`/`startArgv` 真正 spawn。拦截由**内核/runner** 在文件系统调用层完成。

### Q2: 是否在执行前检查命令？是否拦截文件写入类命令？

> 🔑 **否，不预检命令字符串**。沙箱层不解析 `command` 内容，不区分"文件写入类"命令。它只做三件事：(1) mode 为 `danger-full-access` 时短路放行（`bash-sandbox` 第146行）；(2) 其他 mode 时 confine 后执行；(3) 执行后根据 stderr 信号分类（拒绝 / runner 失败 / 普通失败）。"文件写入拦截"是后端在内核层拒绝写操作（产生 EROFS/EACCES/EPERM），JS 层用 `denialSignatures`（后端方言子串）识别。

### Q3: timeoutMs 如何用？

> 🔑 由 **local 层**处理：`resolve`（`bash-local` 第164行）用 `clampTimeout` 把请求值夹在 `[config.timeoutMs=120s, config.maxTimeoutMs=600s]`；`runArgv`（第235行）用 `deadline(signal, timeoutMs, "BASH_TIMEOUT")` 产 fused `AbortSignal`，喂给 `ctx.subprocess.spawn` 的 `signal`。超时触发 SIGTERM→SIGKILL（`graceMs`）。**后台进程无超时**（`startArgv` 第273行用 `spec.signal` 而非 deadline）。

### Q4: helpers.d.ts 是什么辅助函数？

> 🔑 内部模块，提供**拒绝/失败分类**：`isUsableWorkdir`、`isRunnerSpawnFailure`（spawn 拒绝归因到 runner）、`classifyDenial`（后端拒绝方言匹配）、`classifyRunnerFailure`（结构化 runner 失败规则匹配）、`matchesSignature`（非零退出 + stderr 子串匹配）。仅被 `SandboxBashExecutor` 内部使用，未对外 re-export。

### Q5: 它和 sandbox-policy 如何联动？是否对每个命令做审批判定？

> 🔑 **每命令解析策略**：`SandboxBashExecutor.resolve`（第140行）调 `ctx.sandboxPolicy.resolve()` 拿 `{mode, workspaceRoot, sessionId}`。**升级审批**只在工具层（模型主动传 `sandbox_permissions`）触发，经 `approveEscalation` 做严格放宽 + 用户审批。沙箱层只消费已 resolved 的 policy，不直接做用户审批。普通命令（无升级请求）无审批提示，直接按 session/deployment mode 执行。

### Q6: sandbox_permissions 参数如何传递给沙箱层？

> 🔑 链路：模型传入 → `validateEscalationArgs`（tool-bash 第123行）→ `approveBashEscalation`（第238行）→ `approveEscalation`（dsh-sandbox 第92行，严格放宽 + `ctx.approval.request`）→ 返回批准 mode → `policy = {...standingPolicy, mode: approvedMode}`（第390行）→ `request.sandboxPolicy = policy`（第401行）→ `ctx.shell.resolve` 取 `request.sandboxPolicy`（bash-sandbox 第140行）→ `ctx.shell.run` 读 `spec.sandboxPolicy.mode`（第144行）。

### Q7: background 作业机制如何？

> 🔑 `execute` 第403–428行：`ctx.jobs.start` 注册 generic task，`run` 工厂调 `ctx.shell.start`（沙箱层 `start`，非 `run`）拿 `ShellProcess`。返回 `{cancel: proc.kill, done: proc.done.then(processOutcome), readOutput: renderProcessRead(proc.readOutput())}`。**id 返回后取消走 `job_kill`→`cancel`→`proc.kill`**，非 tool-call signal。后台无超时。`processOutcome` 把 settled 进程映射为 `{completed|killed, detail}`。

### Q8: render.d.ts 如何渲染输出？

> 🔑 `renderResult`（前台）：stdout + `[stderr]` 段 + markers（denial/timeout/signal/exit）。非零退出报 `[exit code: N]` 不 isError。`renderProcessRead`（后台增量）：delta + lossy/sandbox 通知。`streamText` 追加截断溢出路径。`parseExitStatus`（来自 dsh-shell）逆解析标记供 terminal 展示拆 exit pill。

### Q9: dsh-bash-local 如何 spawn 子进程？

> 🔑 `runArgv`（第228行）/`startArgv`（第272行）调 `this.ctx.subprocess.spawn(this.spawnSpec(...))`。`spawnSpec`（第180行）构造 `{argv, cwd, stdio:{stdin/stdout collect/stderr collect}, graceMs, signal, env}`。前台用 fused deadline signal（第235行）；后台用 `spec.signal`（第273行）。输出有界捕获（`maxOutputBytes` 内存 + `maxSpillBytes` 溢出文件）。`done` resolve 后调 `onProcessDone` 钩子（基类空，沙箱层覆盖）。

---

## 6. 调用关系总图

```
┌─────────────────────────────────────────────────────────────────────┐
│  模型 (LLM) 调用 bash 工具                                            │
└──────────────────────────────┬──────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  dsh-tool-bash  (ctx.tools "bash")                                   │
│  • validateBashArgs          [119]   入参校验 + escalation 配对校验    │
│  • resolveSandboxPolicy      [226]   ctx.sandboxPolicy.resolve(session)│
│  • approveBashEscalation?    [238]   approveEscalation(严格放宽+审批)  │
│  • resolveWorkdir            [177]   workdir 解析（相对→session root）  │
│  • 构造 request{command,workdir,timeoutMs,dshEnv,sandboxPolicy}       │
│  • 前台: ctx.shell.run(ctx.shell.resolve({...request,signal})) [429] │
│  • 后台: ctx.jobs.start(run=>ctx.shell.start(ctx.shell.resolve)) [414]│
│  • 渲染: renderResult / renderProcessRead / canonicalBashResult      │
└──────────────────────────────┬──────────────────────────────────────┘
                               ▼  ctx.shell = SandboxBashExecutor
┌─────────────────────────────────────────────────────────────────────┐
│  dsh-bash-sandbox  (ctx.shell, extends LocalBashExecutor)            │
│  • resolve(spec)             [137]   注入 sandboxPolicy(优先 request)  │
│  • run(spec):                                                        │
│      - mode==danger-full-access → super.run + sandbox{denied:false}[146]│
│      - confine(command,policy) [228] ctx.sandbox.confine(["bash","-c",cmd])│
│      - runArgv(spec, confined.argv) [159]  ← 调父类                  │
│      - isRunnerSpawnFailure?   [162] → SandboxUnavailableError       │
│      - classifyRunnerFailure?  [165] → SandboxUnavailableError       │
│      - classifyDenial          [171] → sandbox.denied                │
│      - return {...result, sandbox:{mode,denied,enforcement}}         │
│  • start(spec)               [176]   后台同构, 存 processFacts        │
│  • onProcessDone             [206]   后台结算盖 proc.sandbox          │
└──────────────────────────────┬──────────────────────────────────────┘
                               ▼  super.runArgv / super.startArgv
┌─────────────────────────────────────────────────────────────────────┐
│  dsh-bash-local  (LocalBashExecutor, extends ShellExecutor)          │
│  • resolve(request)          [163]   clampTimeout + workdir default   │
│  • spawnSpec(spec,argv,...)  [180]   {argv,cwd,stdio,graceMs,signal,env}│
│  • runArgv(spec,argv)        [228]   deadline→ctx.subprocess.spawn    │
│  • startArgv(spec,argv)      [272]   ShellProcess{done,readOutput,kill}│
│  • onProcessDone()           [330]   空钩子(子类覆盖)                  │
└──────────────────────────────┬──────────────────────────────────────┘
                               ▼  ctx.subprocess.spawn
                        OS 子进程 (bash -c 或受限 runner argv)
```

**外部依赖衔接**：
- `ctx.sandbox`（`dsh-sandbox` `SandboxProvider`）：`confine` 返回受限 argv + 分类事实。
- `ctx.sandboxPolicy`（`dsh-sandbox-policy` `SandboxPolicyService`）：`resolve` 返回 `{mode, workspaceRoot, sessionId}`。
- `ctx.approval`（`dsh-user-approval`）：`request` 做用户审批（升级时）。
- `ctx.jobs`（`dsh-jobs`）：`start` 注册后台 generic task。
- `ctx.subprocess`（`dsh-subprocess-local`）：`spawn` 真正起进程。
- `ctx.shellEnv`（`dsh-shell-env`）：`collect` 收集 `$DSH_*` 托管变量。

---

## 7. 设计要点小结

1. **职责分离**：工具层管"模型契约 + 审批 + 渲染"，沙箱层管"confine + 拒绝分类",local 层管"spawn + 超时 + 输出"。
2. **fail-closed**：runner 不可用→`SandboxUnavailableError`（拒绝未受限执行）；`confine` 必须返回 enforcing argv 或失败。
3. **per-call policy**：policy 逐调用 resolved，不固化在 provider 上；并发调用可不同 mode。
4. **per-process facts**：后台进程的沙箱事实存 `processFacts` Map，避免并发归错。
5. **拒绝≠错误**：非零退出（含拒绝）是"命令结果"报 marker 不 isError；只有 spawn/abort 基础设施失败才 isError。
6. **严格放宽**：升级只能向更宽（`WIDER_MODES`），且需用户审批；非放宽请求不提示人。
7. **能力探测解耦**：工具层用 `ctx.shell.sandboxMode` 探测是否 confine，按需暴露 escalation 字段——同一工具插件兼容有/无沙箱的部署。
