# DSH (DeepSeek Harness) 沙箱与文件系统实现深度分析

> 本文档基于对 `@deepseek-ai/dsh-*` 各包 `lib/index.js` 与 `lib/types/*.d.ts` 的逐行阅读，覆盖文件沙箱、bash 沙箱、approval 机制、permission presets、文件系统后端、沙箱拦截点、一次性升级机制与完整交互流程。所有行号引用对应已编译产物（`node_modules/@deepseek-ai/<pkg>/lib/`）。

---

## 0. 总体架构与分层

DSH 的沙箱与文件系统是一个**分层、单一事实源（single-source-of-truth）**的设计，关键包及其职责：

| 层 | 包 | 角色 | Cordis 服务名 |
|---|---|---|---|
| 抽象接口 | `dsh-fs` | 文件系统 Service Definition（`FileSystem` 抽象类、`FsError`、类型词汇表） | `ctx.fs`（抽象） |
| 抽象接口 | `dsh-sandbox` | 进程沙箱 Service Definition（`SandboxProvider` 抽象类）+ 升级词汇表 + writable roots | `ctx.sandbox`（抽象） |
| 策略层 | `dsh-sandbox-policy` | 沙箱模式决策中枢（`ctx.sandboxPolicy`），三种模式 + 会话级 override | `ctx.sandboxPolicy` |
| 审批层 | `dsh-user-approval` | 用户审批服务（`ctx.approval`），ask/never 策略 | `ctx.approval` |
| 预设层 | `dsh-permission-presets` | 将 sandbox 模式 + approval 策略打包成"权限预设" | `ctx.permissionPresets` |
| FS 拦截层 | `dsh-fs-sandbox` | **沙箱化的 FS 后端**，包装 `fs-local`，在写操作前做策略判定 | `ctx.fs`（拦截实现） |
| FS 后端 | `dsh-fs-local` | 底层文件 IO（realpath、原子写、edit 临界区、Win32 ACL） | `ctx.fs`（原始实现） |
| 进程后端 | `dsh-sandbox-local` | 平台原生进程沙箱（bwrap/Landlock/Seatbelt/Windows-ACL） | `ctx.sandbox`（本地实现） |
| Bash 拦截层 | `dsh-bash-sandbox` | 沙箱化的 bash 执行器，包装 `bash-local`，将 argv 经 `ctx.sandbox.confine()` 包裹 | `ctx.shell`（拦截实现） |
| Bash 后端 | `dsh-bash-local` | 底层 bash 子进程 spawn | `ctx.shell`（原始实现） |
| 工具层 | `dsh-tool-fs` | 面向 LLM 的 read/write/edit 工具，拥有 escalation 升级编排 | （工具注册） |
| 工具层 | `dsh-tool-bash` | 面向 LLM 的 bash 工具，拥有 escalation 升级编排 | （工具注册） |
| 观察策略 | `dsh-fs-observation-policy` | "写前必先读"事件策略（注册 `fs/*` 事件监听） | （事件插件，无服务） |

> 🔑 **核心分层原则**：`dsh-sandbox` / `dsh-fs` 是纯抽象 Service Definition（不含实现）；`dsh-sandbox-local` / `dsh-fs-local` 是本地实现；`dsh-fs-sandbox` / `dsh-bash-sandbox` 是**拦截层**（继承本地实现，在写/执行前插入策略判定）；`dsh-tool-fs` / `dsh-tool-bash` 是**工具层**，负责把 `sandbox_permissions` 参数翻译成一次性升级请求并通过 `ctx.approval` 审批。

### cordis.patch.yml 的装配（`dsh-base/cordis.patch.yml`）

```yaml
# 第 169-205 行 —— 沙箱与权限的装配
- id: sandbox
  name: '@deepseek-ai/dsh-sandbox-local'          # 本地进程沙箱后端

- id: sandbox-policy
  name: '@deepseek-ai/dsh-sandbox-policy'
  config:
    mode: !!js process.env.DSH_PERMISSION_MODE ?? 'workspace-write'   # 第175行
    workspaceRoot: !!js process.cwd()                                 # 第176行

- id: bash-sandbox
  name: '@deepseek-ai/dsh-bash-sandbox'           # 替代 bash-local，注册为 ctx.shell
  disabled: !!js process.platform === 'win32'
  config:
    timeoutMs: 60000

- id: approval
  name: '@deepseek-ai/dsh-user-approval'
  config:
    policy: !!js "(... === 'danger-full-access') ? 'never' : 'ask'"   # 第191行

- id: permission
  name: '@deepseek-ai/dsh-permission-presets'
  config:
    presets:                                       # 第196-205行
      read-only:          { sandbox: read-only, approval: ask }
      workspace-write:    { sandbox: workspace-write, approval: ask }
      danger-full-access: { sandbox: danger-full-access, approval: never }

# 第 443-444 行 —— 沙箱化 FS 后端
- id: fs-sandbox
  name: '@deepseek-ai/dsh-fs-sandbox'             # 替代 fs-local，注册为 ctx.fs
```

> 🔑 **装配关键点**：`bash-sandbox`（而非 `bash-local`）注册为 `ctx.shell`；`fs-sandbox`（而非 `fs-local`）注册为 `ctx.fs`。工具层（`tool-bash`/`tool-fs`）完全无感——它们只调用 `ctx.shell` / `ctx.fs`，拦截在服务层完成。默认模式由环境变量 `DSH_PERMISSION_MODE` 控制（默认 `workspace-write`）。

---

## 1. 文件沙箱模式 — read-only / workspace-write / danger-full-access

### 1.1 模式定义

三种模式定义在 `dsh-sandbox/lib/types/index.d.ts` 第 19 行：

```typescript
export type SandboxMode = 'read-only' | 'workspace-write' | 'danger-full-access';
```

`SandboxExecutionPolicy`（第 27-40 行）是**每次调用（per-call）解析出的完整策略**：

```typescript
export interface SandboxExecutionPolicy {
    mode: SandboxMode;          // 文件效果模式
    workspaceRoot: string;      // workspace-write 下可写的绝对根目录
    sessionId?: SessionId;      // 调用会话的标识（Windows-ACL 用于隔离）
}
```

### 1.2 三种模式的语义（`dsh-sandbox/lib/types/index.d.ts` 第 14-18 行注释 + `dsh-sandbox/lib/index.js`）

| 模式 | 可写范围 | 说明 |
|---|---|---|
| `read-only` | 仅 `/dev/null` 等必需 sink | 拒绝一切文件变更 |
| `workspace-write` | workspace root + 平台 temp 区（`/tmp` + `os.tmpdir()`） | 允许工作区和临时目录写入 |
| `danger-full-access` | 无限制 | 绕过一切隔离 |

> 🔑 **writableRoots 函数**（`dsh-sandbox/lib/index.js` 第 154-161 行）是三种模式可写根的**唯一事实源**：

```javascript
function writableRoots(policy) {
    if (policy.mode !== "workspace-write") return [];   // read-only / danger 都返回空
    return [...new Set([
        policy.workspaceRoot,
        "/tmp",
        tmpdir()
    ].map(canonicalPath))];   // canonical = realpath，解析符号链接
}
```

`read-only` 返回空数组（无可写根）；`danger-full-access` 不走此函数（直接放行）。**只有 `workspace-write` 产生可写根列表**，且 `workspace-write` 之外的两个模式都不调用 `writableRoots`——`danger-full-access` 在更早处短路返回，`read-only` 直接抛错。

### 1.3 canonicalPath — 符号链接规范化（第 138-144 行）

```javascript
function canonicalPath(path) {
    try { return realpathSync.native(path); }
    catch { return path; }   // 路径不存在时原样返回（保守：匹配不到任何根）
}
```

> 🔑 在 macOS 上 `/tmp` 实际是 `/private/tmp`，若不 canonical 化，按拼写授权的根会匹配不到任何路径。`canonicalPath` 确保 Seatbelt 授予的根与 fs fence 的遏制检查匹配的是**同一个解析后路径**。

---

## 2. sandbox-policy 如何决定文件操作是否允许

`dsh-sandbox-policy` 是沙箱模式的**决策中枢**，注册为 `ctx.sandboxPolicy`。

### 2.1 SandboxPolicyService（`dsh-sandbox-policy/lib/index.js` 第 101-154 行）

```javascript
var SandboxPolicyService = class extends Service {
    static Config = z.object({
        mode: z.union(["read-only", "workspace-write", "danger-full-access"]).default("read-only"),
        workspaceRoot: z.string()
    });
    defaultMode;       // 部署默认模式（第111行）
    workspaceRoot;     // workspace-write 的回退根（第113行）

    constructor(ctx, config) {
        super(ctx, "sandboxPolicy");
        this.defaultMode = config.mode;
        this.workspaceRoot = resolveWorkspaceRoot(config.workspaceRoot ?? process.cwd());
        // 第118-127行：注入 systemPrompt，将当前策略渲染进 runtime-context 快照
        ctx.inject(["systemPrompt"], (scope) => {
            scope.systemPrompt.context({
                name: "sandbox:policy",
                order: 110,
                text: (context) => {
                    const session = context.agent?.session;
                    return session === void 0 ? "" : renderPolicyContext(this.resolve({ session }));
                }
            });
        });
    }
```

### 2.2 resolve() — 每次调用的策略解析（第 138-145 行）

> 🔑 **这是整个沙箱决策的核心入口**。优先级从高到低：

```javascript
resolve(request = {}) {
    const { session } = request;
    return {
        mode: request.mode ??                                  // 1. 显式批准的升级模式（最高优先级）
              (session === void 0 ? void 0 : this.overrideOf(session)) ??  // 2. 会话级 override（sandbox/mode 事件）
              this.defaultMode,                                // 3. 部署默认模式（最低）
        workspaceRoot: resolveWorkspaceRoot(session?.header.cwd ?? this.workspaceRoot),
        ...session === void 0 ? {} : { sessionId: session.id }
    };
}
```

**优先级链**：`approved escalation mode` > `session override (sandbox/mode 事件)` > `deployment default`。

### 2.3 会话级 override — session 日志即存储（第 5-56 行）

```javascript
// 第 39-44 行：fold 函数 —— 从事件日志倒序找最后一个 sandbox/mode 事件
function effectiveSandboxMode(events) {
    for (let index = events.length - 1; index >= 0; index -= 1) {
        const event = events[index];
        if (event.type === "sandbox/mode") return event.data.mode;
    }
}

// 第 54-56 行：唯一的写入路径 —— 追加一个 sandbox/mode 事件
function setSandboxMode(session, mode) {
    session.append("sandbox/mode", { mode });
}
```

> 🔑 **会话级状态用事件日志存储，无外部 config store**：`effective = fold(events) ?? 部署默认`。重启后重放日志即恢复状态；两个会话看不到彼此的状态。这与 `approval/policy` 事件的设计完全对称。事件类型定义在 `dsh-sandbox-policy/lib/types/session-mode.d.ts` 第 22-37 行（`SessionEventMap` 扩展了 `sandbox/mode` 事件，可携带 `source: 'delegation'` 标记用于子代理种子化）。

### 2.4 renderPolicyContext — 系统提示渲染（第 83-94 行）

根据模式向 LLM 注入不同的上下文文本（即你在 runtime context 中看到的 "Current DSH file policy: workspace-write..."）。这部分文本在每次模型请求前由 `systemPrompt.context()` 注入。

### 2.5 谁调用 sandboxPolicy.resolve()？

- **fs-sandbox 的 `checkedTarget()`**（见第 4 节）—— 每次写/编辑前调用
- **bash-sandbox 的 `resolve()`**（见第 3 节）—— 每次 bash 执行前调用
- **tool-fs 的 `FsSandboxController.resolvePolicy()`** —— 工具层解析策略后传入
- **tool-bash 的 `resolveSandboxPolicy()`** —— 工具层解析策略后传入

> 🔑 **enforcing filesystem、one-shot bash、terminal backends 读取的是同一个 `resolve()` 结果**——这保证了 bash 和 fs 不会出现"写工具不能写 /tmp 但 bash 可以"的不对称。

---

## 3. bash 沙箱 — 命令执行如何被沙箱拦截

`dsh-bash-sandbox` 注册为 `ctx.shell`，**替代** `dsh-bash-local`。它继承 `LocalBashExecutor`，在执行前将 argv 经 `ctx.sandbox.confine()` 包裹。

### 3.1 SandboxBashExecutor（`dsh-bash-sandbox/lib/index.js` 第 110-235 行）

```javascript
var SandboxBashExecutor = class extends LocalBashExecutor {
    static inject = ["subprocess", "sandbox", "sandboxPolicy"];   // 第111-115行
    mode;

    constructor(ctx, config) {
        super(ctx, config);
        this.mode = ctx.sandboxPolicy.defaultMode;   // 第126行
    }
    get sandboxMode() { return this.mode; }           // 第129行 —— 能力事实

    // 第137-142行：把 per-call 策略盖印到 spec 上
    resolve(request) {
        return {
            ...super.resolve(request),
            sandboxPolicy: request.sandboxPolicy ?? this.ctx.sandboxPolicy.resolve()
        };
    }
```

### 3.2 run() — 前台执行拦截（第 143-175 行）

> 🔑 **这是 bash 沙箱的核心拦截逻辑**：

```javascript
async run(spec) {
    const policy = spec.sandboxPolicy;
    const { mode } = policy;

    // 第146-152行：danger-full-access 直接放行，不包裹
    if (mode === "danger-full-access") return {
        ...await super.run(spec),
        sandbox: { mode, denied: false }
    };

    // 第153-156行：非 danger 模式，将命令经 ctx.sandbox.confine() 包裹
    const confined = this.confine(spec.command, { ...policy, mode });
    let result;
    try {
        result = await this.runArgv(spec, confined.argv);   // 执行包裹后的 argv
    } catch (error) {
        if (spec.signal?.aborted === true) spec.signal.throwIfAborted();
        // 第162行：runner 启动失败 → SANDBOX_UNAVAILABLE（fail-closed）
        if (isRunnerSpawnFailure(error, confined.argv[0], spec.workdir))
            throw new SandboxUnavailableError(mode, String(error));
        throw error;
    }

    // 第165-166行：runner 自身失败（非命令失败）→ SANDBOX_UNAVAILABLE
    const runnerFailure = classifyRunnerFailure(result.exitCode, result.stderr.text, confined.runnerFailureRules);
    if (runnerFailure !== void 0) throw new SandboxUnavailableError(mode, runnerFailure.detail);

    return {
        ...result,
        sandbox: {
            mode,
            denied: classifyDenial(result, confined.denialSignatures),   // 第171行：根据 stderr 判断是否被沙箱拒绝
            enforcement: confined.enforcement
        }
    };
}
```

### 3.3 confine() — 命令包裹（第 228-234 行）

```javascript
confine(command, policy) {
    return this.ctx.sandbox.confine(["bash", "-c", command], policy);
    // 把 shell 源码包成 ["bash", "-c", command] 再交给沙箱 provider
}
```

> 🔑 **bash 沙箱不解析命令内容**——它不检查"这是否是 `rm -rf`"。它把整个命令作为一个 `bash -c` 子进程，然后让**内核级沙箱**（Seatbelt/Landlock/bwrap）去限制这个子进程能写哪些文件。这是进程级隔离，不是命令级审计。

### 3.4 拒绝信号分类（helpers.js，第 53-91 行）

由于 bash 走的是内核沙箱，拒绝信息出现在 **stderr** 中（而非结构化错误码）。`classifyDenial()`（第 53-55 行）用 `matchesSignature()`（第 87-91 行）做**不区分大小写的子串匹配**：

```javascript
function classifyDenial(result, signatures) {
    return matchesSignature(result.exitCode, result.stderr.text, signatures);
}
function matchesSignature(exitCode, stderr, signatures) {
    if (exitCode === null || exitCode === 0) return false;   // 成功退出不算拒绝
    const lowered = stderr.toLowerCase();
    return signatures.some((s) => lowered.includes(s.toLowerCase()));
}
```

各后端的拒绝方言（`dsh-sandbox-local/lib/index.js` 第 208-218 行）：

| 后端 | denialSignatures（stderr 子串） |
|---|---|
| bwrap | `read-only file system` |
| landlock | `permission denied` |
| seatbelt | `operation not permitted` |
| windows-acl | `access is denied`, `access to the path`, `permission denied` |

### 3.5 后台进程拦截（start/onProcessDone，第 176-219 行）

后台进程（`run_in_background: true`）走 `start()`（第 176-201 行），同样包裹 argv，但把 per-process 沙箱事实存入 `processFacts` Map，在 `onProcessDone()`（第 206-219 行）进程结算时再分类拒绝信号。

> 🔑 **关键设计**：`processFacts` 是**每进程**而非全局的，因为不同重叠调用的后端可能不同（enforcement 和诊断方言可能不同），用全局最新值会错配分类。

### 3.6 runner 失败 vs 命令失败 vs 沙箱拒绝

三者必须区分（`dsh-bash-sandbox/lib/index.js` 第 47-91 行 + `dsh-sandbox-local/lib/index.js` 第 232-244 行）：

- **runner 失败**（沙箱程序本身没启动/崩溃）→ `SANDBOX_UNAVAILABLE`，命令根本没运行
- **沙箱拒绝**（命令运行了但内核拒绝了文件写入）→ `sandbox.denied = true`，命令非零退出
- **命令失败**（命令逻辑出错）→ 正常非零退出，`sandbox.denied = false`

`classifyRunnerFailure()`（第 66-79 行）用结构化规则（exit-code gate + informational lines 排除 + fatal signatures）区分 runner 失败，例如 Landlock 的 exit-125 + `landlock-run: ` 前缀。

---

## 4. 文件系统后端 — dsh-fs-local 如何实现读写

`dsh-fs-local` 是底层文件 IO 实现，注册为 `ctx.fs`（被 `fs-sandbox` 继承替代）。

### 4.1 抽象接口 — dsh-fs（`dsh-fs/lib/index.js` 第 58-75 行）

```javascript
var FileSystem = class extends Service {
    constructor(ctx) { super(ctx, "fs"); }
    get sandboxMode() {}   // 第74行：基类返回 undefined（不隔离）；fs-sandbox 覆写为部署默认
};
```

`FsError`（第 34-40 行）携带结构化错误码（`FsErrorCode` 类型见 `dsh-fs/lib/types/types.d.ts` 第 162 行）：

```typescript
export type FsErrorCode = 'FS_NOT_FOUND' | 'FS_NOT_DIRECTORY' | 'FS_NOT_TEXT' |
    'FS_NOT_REGULAR_FILE' | 'FS_TOO_LARGE' | 'FS_PERMISSION_DENIED' |
    'FS_SANDBOX_DENIED' | 'FS_IO_ERROR' | 'FS_STALE_VERSION' |
    'FS_NOT_OBSERVED' | 'FS_AMBIGUOUS_EDIT' | 'FS_EDIT_NOT_FOUND' | 'FS_ABORTED';
```

> 🔑 **`FS_SANDBOX_DENIED`** 是沙箱拒绝的专属错误码——与 bash 的 stderr 推断不同，fs fence 是**进程内**的，精确知道自己拒绝了什么。

### 4.2 LocalFileSystem（`dsh-fs-local/lib/index.js` 第 674-824 行）

```javascript
var LocalFileSystem = class extends FileSystem {
    static Config = z.object({
        cwd: z.string().default(process.cwd()),
        diffBasisMaxBytes: z.number().default(10 * 1024 * 1024)
    });
    locks = new Map();   // 第686行：per-targetKey 串行化锁
```

#### resolve()（第 704-712 行）

```javascript
async resolve(path, opts) {
    const local = await resolveLocalTarget(opts?.cwd ?? this.config.cwd, path);
    return { targetKey: local.targetKey, displayPath: local.displayPath };
}
```

> 🔑 `targetKey` 是 **realpath 身份**（符号链接已解析），`displayPath` 是**未解析的绝对路径**（用于展示）。这让别名共享 stale guard，且通过符号链接写入会更新其目标而非替换链接。

#### writeText() — 原子写入（第 779-797 行）

```javascript
async writeText(target, content, expected, signal) {
    return this.withLock(target.targetKey, async () => {   // 串行化
        const existing = await probe(target.targetKey);
        if (existing && existing.type !== "file") throw FsError("FS_NOT_REGULAR_FILE");
        // 第783-786行：版本守卫（CAS）
        if (expected?.kind === "replaceIfVersion") {
            if (!existing) throw FsError("FS_STALE_VERSION");
            if (existing.version !== expected.version) throw FsError("FS_STALE_VERSION");
        } else if (expected?.kind === "createIfAbsent" && existing)
            throw FsError("FS_NOT_OBSERVED");
        // 第787行：读取 before 内容用于 diff
        const before = existing && Buffer.byteLength(content) < this.config.diffBasisMaxBytes
            ? await readTextForDiff(...) : null;
        // 第788行：原子发布
        await writeFileAtomic(target.targetKey, content, existing?.mode, signal, ...);
        return { operation: existing ? "update" : "create", version, before, after };
    });
}
```

#### editText() — 字面量编辑临界区（第 798-815 行）

```javascript
async editText(target, edit, expected, signal) {
    return this.withLock(target.targetKey, async () => {
        const existing = await probe(target.targetKey);
        if (!existing) throw FsError("FS_STALE_VERSION");
        if (expected && existing.version !== expected.version) throw FsError("FS_STALE_VERSION");
        const original = await readForEdit(...);
        const edited = applyLiteralEdit(original.content, edit.oldString, edit.newString, edit.replaceAll, ...);
        await writeFileAtomic(target.targetKey, content, existing.mode, signal, ...);
        return { version, before: original.content, after: edited.content };
    });
}
```

#### 原子写入机制（fsio.js 注释，第 88-93 行 + win32.js 第 10-87 行）

> 🔑 写入采用 **stage-then-publish** 模式：在目标目录旁的私有兄弟目录中创建独占 owner-only 临时文件，写入并 fsync 后，通过 `rename`（POSIX）或 `ReplaceFileW`（Windows，第 83-86 行，保留被替换文件的 ACL）原子发布。

#### withLock — per-key FIFO 串行化（第 694-703 行）

```javascript
async withLock(targetKey, op) {
    const run = (this.locks.get(targetKey) ?? Promise.resolve()).then(op, op);
    this.locks.set(targetKey, run.then(() => void 0, () => void 0));
    try { return await run; }
    finally { if (this.locks.get(targetKey) === tail) this.locks.delete(targetKey); }
}
```

> 🔑 这让并发的 write/edit 对**同一文件**确定性有序：一个赢，其余看到新版本并以 `FS_STALE_VERSION` 拒绝。edit 的 read→match→rewrite 临界区因此无法交错。

### 4.3 Win32 平台差异（win32.js，第 10-87 行）

Windows 上用 koffi 懒加载 `advapi32.dll` / `kernel32.dll`，通过 `GetFileSecurityW`/`SetFileSecurityW` 读取并复制 DACL，通过 `ReplaceFileW` 做安全替换（保留 ACL 和元数据）。非 Windows 进程永不加载 Win32 库。

---

## 5. 沙箱拦截点 — 在工具执行的哪个环节拦截

### 5.1 FS 拦截层 — SandboxedFileSystem（`dsh-fs-sandbox/lib/index.js`）

`SandboxedFileSystem` **继承** `LocalFileSystem`，只覆写两个变更方法（`writeText`/`editText`），读操作**原样放行**：

```javascript
var SandboxedFileSystem = class extends LocalFileSystem {
    static inject = ["sandboxPolicy"];   // 第108行
    defaultMode;
    constructor(ctx, config) {
        super(ctx, config);
        this.defaultMode = ctx.sandboxPolicy.defaultMode;   // 第112行
    }
    get sandboxMode() { return this.defaultMode; }          // 第115行 —— 能力事实

    // 第129-131行：写操作先 fence 再委托
    async writeText(target, content, expected, signal, sandboxPolicy) {
        return super.writeText(await this.checkedTarget(target, sandboxPolicy), content, expected, signal);
    }
    // 第143-145行：编辑同理
    async editText(target, edit, expected, signal, sandboxPolicy) {
        return super.editText(await this.checkedTarget(target, sandboxPolicy), edit, expected, signal);
    }
```

#### checkedTarget() — 核心拦截点（第 157-170 行）

> 🔑 **这是文件沙箱的心脏**。它在**每次写/编辑**前执行策略判定：

```javascript
async checkedTarget(target, sandboxPolicy) {
    const policy = sandboxPolicy ?? this.ctx.sandboxPolicy.resolve();
    const { mode } = policy;

    if (mode === "danger-full-access") return target;   // 第160行：放行

    if (mode === "read-only")   // 第161行：直接拒绝
        throw new FsError(`cannot write "${target.displayPath}": file access denied under read-only mode`, "FS_SANDBOX_DENIED");

    // workspace-write：重新 canonical 化（防止 TOCTOU）+ 遏制检查
    const fresh = await this.resolve(target.displayPath);   // 第162行：NOW 重新 realpath
    let contained = false;
    for (const root of writableRoots(policy)) {              // 第164行：遍历可写根
        if (await isPathUnder(fresh.targetKey, root)) { contained = true; break; }
    }
    if (!contained)   // 第168行：不在任何可写根下 → 拒绝
        throw new FsError(`cannot write "${target.displayPath}": file access denied under workspace-write mode`, "FS_SANDBOX_DENIED");
    return fresh;   // 第169行：返回 fresh target（check-here-write-there 一致）
}
```

> 🔑 **TOCTOU 缓解**：`checkedTarget` 返回的是**重新 canonical 化后的 fresh target**（第 162 行 `this.resolve()`），而非调用者传入的 target。这样"检查的路径"就是"写入的路径"，消除了 check-here-write-there 的时间窗口。残留的 TOCTOU（遏制检查后、syscall 前的祖先符号链接被替换）被接受为此威胁模型的可接受残余。

#### isPathUnder — 遏制检查（containment.js，第 53-65 行）

```javascript
async function isPathUnder(path, root, caseSensitive = process.platform !== "win32") {
    if (isLexicallyUnder(path, root, caseSensitive)) return true;   // 快速词法路径
    // 慢路径：walk 祖先，比 filesystem identity（dev+ino）
    const rootInfo = await statIfPresent(root);
    if (!rootInfo) return false;
    let ancestor = path;
    while (true) {
        const ancestorInfo = await statIfPresent(ancestor);
        if (ancestorInfo && sameIdentity(ancestorInfo, rootInfo)) return true;
        const parent = dirname(ancestor);
        if (parent === ancestor) return false;
        ancestor = parent;
    }
}
```

> 🔑 词法快速路径处理正常 canonical 拼写；当拼写不同（Windows 8.3 短名/大小写别名）时，走祖先遍历 + `dev === right.dev && ino === right.ino` 身份比较（第 38-40 行），不弱化遏制为文本近似。

### 5.2 Bash 拦截层（见第 3 节）

bash 的拦截在 `SandboxBashExecutor.run()` / `start()` 中，通过 `ctx.sandbox.confine()` 包裹 argv。

### 5.3 拦截点总结

| 操作类型 | 拦截位置 | 拦截方式 |
|---|---|---|
| 文件写（write） | `SandboxedFileSystem.writeText` → `checkedTarget` | 进程内策略判定 + 遏制检查 |
| 文件编辑（edit） | `SandboxedFileSystem.editText` → `checkedTarget` | 同上 |
| 文件读（read/stat/listDir） | **不拦截**，原样放行 | 所有模式都允许读 |
| bash 命令 | `SandboxBashExecutor.run/start` → `ctx.sandbox.confine` | 内核级 argv 包裹 |

> 🔑 **fs fence 是策略检查（trusted code 对 model-controlled path），不是内核边界**；bash 是**内核边界**（untrusted code）。fs-sandbox 的注释（第 77-85 行）明确指出：操作是 seam 自己的（open/rename），只有 target path 不可信，所以 canonicalize-then-contain 是完整答案。不可信代码的内核级隔离是 `ctx.shell`（bash-sandbox）的职责。

---

## 6. approval 机制 — 何时需要用户批准、ask/never 策略

`dsh-user-approval` 注册为 `ctx.approval`，是审批能力的 Service Definition。

### 6.1 审批策略（`dsh-user-approval/lib/index.js` 第 36-40 行）

```javascript
const APPROVAL_POLICIES = ["ask", "never"];

// 第38行：never 策略的模型面提示
const NEVER_SENTENCE = "Approval prompts are disabled in this session: actions that require approval are rejected automatically — do not request sandbox escalation (do not set `sandbox_permissions`).";

// 第40行：ask 策略的提示
const ASK_SENTENCE = "Approval policy: ask. Operations that require approval may ask through the configured answerers; without an available answerer, the request fails closed.";
```

> 🔑 **`never`** = 确定性拒绝，从不提示任何人（适合 CI/无人值守）；**`ask`** = 委托给已组合的 answerer 链（如 Web UI 的用户点击），无 answerer 时 fail-closed 为 `unavailable`。

### 6.2 ApprovalService.request()（第 144-160 行）

> 🔑 **这是审批的核心入口**，被 escalation 机制调用：

```javascript
async request(req) {
    const session = req.agent.session;
    // 第146行：前置条件 —— 必须在开放的 turn 内（审计对必须 turn 包围）
    if (!hasOpenTurn(session.events))
        throw new Error("approval.request() outside an open turn...");
    const id = ApprovalRequestId(randomUUID());
    // 第148-153行：追加 approval/asked 审计事件
    session.append("approval/asked", { id, toolName: req.toolName, ...req.callId ? { callId } : {}, ...req.reason ? { reason } : {} });
    const outcome = await this.decide(req, session);    // 第154行：决策
    // 第155-158行：追加 approval/decided 审计事件
    session.append("approval/decided", { id, outcome });
    return outcome;
}
```

### 6.3 decide() — 策略 + answerer 瀑布（第 185-202 行）

```javascript
async decide(req, session) {
    const signal = req.signal;
    if (signal?.aborted) return "cancelled";                       // 信号已取消
    if (this.effectivePolicy(session) === "never") return "rejected";  // 第188行：never 直接拒绝
    // 第189行：answerer 瀑布（approval/request 事件），默认 fail-closed "unavailable"
    const answer = Promise.resolve().then(() =>
        this.ctx.waterfall(scopeTarget(this, req.agent), "approval/request", req, () => Promise.resolve("unavailable"))
    ).then((outcome) => OUTCOMES.includes(outcome) ? outcome : "unavailable", () => "unavailable");
    // 与 signal 竞态
    if (signal === void 0) return answer;
    return await new Promise((resolve) => {
        const onAbort = () => { signal.removeEventListener("abort", onAbort); resolve("cancelled"); };
        signal.addEventListener("abort", onAbort, { once: true });
        answer.then((outcome) => { signal.removeEventListener("abort", onAbort); resolve(outcome); });
    });
}
```

### 6.4 审批结果词汇表（第 29-34 行）

```javascript
const OUTCOMES = ["allowed-once", "rejected", "cancelled", "unavailable"];
```

- `allowed-once` —— **唯一授权**，仅对请求的那一个操作有效
- `rejected` —— 用户拒绝
- `cancelled` —— 信号中断
- `unavailable` —— 无可用 answerer / answerer 抛错 / 返回非词汇值（fail-closed 归一化）

> 🔑 **审批是阻塞的**：`request()` 是 `async`，会等待 answerer 链返回。在 Web UI 中这对应一个弹窗等待用户点击。审批结果只对**这一次操作**有效——不是会话级或永久授权。

### 6.5 会话级审批策略 override（第 49-79 行）

与 sandbox 模式完全对称：`effectiveApprovalPolicy()` fold 最后一个 `approval/policy` 事件（第 49-54 行），`setApprovalPolicy()` 追加事件（第 76-79 行）。

### 6.6 审计对（approval/asked + approval/decided）

> 🔑 每次审批都会在会话日志中留下一对审计事件（`asked` + `decided`，用相同 `id` 配对）。这要求在**开放 turn 内**调用（第 146 行），因为 turn 是持久日志的 commit/replay 边界——turn 之间的裸事件在 reload 时会被当作 crash-tail 垃圾丢弃。

### 6.7 策略切换通知（第 111-125 行）

`setPolicy()` 切换策略时，除了追加事件，还会向 agent 注入一条用户消息（`createUserMessage`），告知模型策略已变更——这样模型在下一步就知道当前策略。

---

## 7. permission presets — 预设的权限组合

`dsh-permission-presets` 将 sandbox 模式 + approval 策略**打包成用户可切换的"预设"**，注册为 `ctx.permissionPresets`。

### 7.1 预设表（`dsh-permission-presets/lib/index.js` 第 80-101 行）

```javascript
static Config = z.object({
    presets: z.dict(z.object({
        sandbox: z.union(SANDBOX_MODES).required(),
        approval: z.union(APPROVAL_POLICIES).required(),
        name: z.string(),
        description: z.string()
    })).default({
        "workspace-write": { sandbox: "workspace-write", approval: "ask", ... },
        "danger-full-access": { sandbox: "danger-full-access", approval: "never", ... }
    }),
    defaultPreset: z.string()
});
```

cordis.patch.yml 中定义了三个预设（第 196-205 行）：

| Preset | sandbox | approval | 含义 |
|---|---|---|---|
| `read-only` | read-only | ask | 只读，升级需审批 |
| `workspace-write` | workspace-write | ask | 工作区可写，升级需审批（**默认**） |
| `danger-full-access` | danger-full-access | never | 完全访问，不审批 |

### 7.2 preset 与独立旋钮的关系

> 🔑 **preset 不覆盖 sandbox-policy.mode 和 approval.policy——它通过各自的 canonical setter 写入它们**。`apply()`（第 272-278 行）：

```javascript
apply(session, name, setApproval) {
    const spec = this.resolve(name);
    if (this.current(session.events) !== name) session.append("permission/preset", { preset: name });
    const events = session.events;
    // 第276行：通过 setSandboxMode 写 sandbox/mode 事件
    if (spec.sandbox !== (effectiveSandboxMode(events) ?? this.ctx.shell.sandboxMode))
        setSandboxMode(session, spec.sandbox);
    // 第277行：通过 setApprovalPolicy 写 approval/policy 事件
    if (spec.approval !== (effectiveApprovalPolicy(events) ?? this.ctx.approval.config.policy ?? "ask"))
        setApproval(spec.approval);
}
```

**切换 preset = 追加 `permission/preset` 事件 + 追加 `sandbox/mode` 事件 + 追加 `approval/policy` 事件**。执行、提示渲染、重放都继续读取各自的旋钮 fold。preset 事件只保留用户意图（当两个 preset 共享 bundle 时）。

### 7.3 derive() — 从旋钮反推 preset（第 205-215 行）

```javascript
derive(state) {
    const sandbox = state.sandbox ?? this.ctx.shell.sandboxMode;
    const approval = state.approval ?? this.ctx.approval.config.policy ?? "ask";
    const matches = (spec) => spec.sandbox === sandbox && spec.approval === approval;
    if (state.preset !== null) { /* 上次选择仍匹配则优先 */ }
    for (const [name, spec] of Object.entries(this.presets)) if (matches(spec)) return name;
    return CUSTOM_PRESET;   // 第214行：无匹配 → "custom"
}
```

> 🔑 当有效旋钮值不匹配任何表项时返回 `"custom"`（第 23 行）——它永远不是切换目标或事件载荷，只是派生的"非预设"状态。

### 7.4 会话初始化钉扎（第 285-308 行）

`pinInitialPermission()` 在会话创建时（`session/created` 事件，第 131-134 行）填充缺失的权限事实：全新会话用用户默认 preset；已 seed 的会话保留有效旋钮值，只补缺失的持久事实。

### 7.5 前端与命令

- **读侧**：`permissions` session projection（第 143-152 行），用 zod schema 向客户端推送 select 选项
- **写侧**：`/permission <preset>` 命令（第 153-177 行）

---

## 8. sandbox_permissions 一次性升级机制

这是整个沙箱设计中最精巧的部分：当沙箱拒绝一个操作后，模型可以在**同一回合**内带 `sandbox_permissions` 参数重试，触发一次性审批升级。

### 8.1 升级词汇表（`dsh-sandbox/lib/index.js` 第 29-41 行）

```javascript
// 第29-32行：严格更宽表 —— 当前模式可升级到哪些模式
const WIDER_MODES = {
    "read-only": ["workspace-write", "danger-full-access"],
    "workspace-write": ["danger-full-access"]
};

// 第41行：闭集升级目标（read-only 是地板，无物升级到它）
const ESCALATION_TARGETS = ["workspace-write", "danger-full-access"];
```

> 🔑 **严格更宽（strictly wider）**：只能向更宽的模式升级，不能降级也不能平级。`read-only → workspace-write` ✓；`workspace-write → danger-full-access` ✓；`read-only → read-only` ✗（非更宽）；`workspace-write → read-only` ✗（更窄）。

### 8.2 参数配对验证（第 50-54 行）

```javascript
function validateEscalationArgs(sandboxPermissions, justification) {
    if (sandboxPermissions !== void 0 && justification === void 0)
        throw new Error("invalid escalation: sandbox_permissions requires a justification");
    if (justification !== void 0 && sandboxPermissions === void 0)
        throw new Error("invalid escalation: justification is only valid together with sandbox_permissions");
    if (justification !== void 0 && justification.trim().length === 0)
        throw new Error("invalid justification: expected a non-empty sentence");
}
```

> 🔑 `sandbox_permissions` 和 `justification` **必须成对出现**——一个没有理由的审批请求，或一个不驱动任何操作的 reason，都是畸形请求。justification 必须是非空句子（面向用户的审批理由）。

### 8.3 模型面拒绝标记（第 63-77 行）

```javascript
function sandboxDenialMarker(mode) {
    return `[sandbox: file access denied under ${mode} mode]`;
}
function escalationHintMarker(subject) {   // subject = "command" | "operation"
    return `[sandbox: escalation available — retry this exact ${subject} once with sandbox_permissions (the narrowest wider mode that suffices) + justification; the approval prompt asks the user]`;
}
```

> 🔑 **两个 enforcing family（bash 和 fs）教授并报告同一词汇**，让模型无论是 bash 的内核 stderr 拒绝还是 fs fence 的结构化拒绝，都识别为相同的策略拒绝。拒绝标记后附带**同回合升级提示**——nudge 在决策点，让模型不必回忆工具描述。

### 8.4 approveEscalation() — 有序 fail-closed 序列（第 92-111 行）

> 🔑 **这是升级的核心编排，在任何执行之前运行**：

```javascript
async function approveEscalation(request, approval) {
    const { requestedMode: mode, effectiveMode, justification, subject } = request;

    // 第94行：1. 检查严格更宽
    if (!(WIDER_MODES[effectiveMode] ?? []).includes(mode))
        throw new Error(`sandbox escalation to "${mode}" is not strictly wider than this call's current "${effectiveMode}" mode`);

    // 第95行：2. 检查 approval 服务存在
    if (approval.approver === void 0)
        throw new Error(`sandbox escalation to "${mode}" requires approval, but no approval service is composed`);

    // 第96行：3. 检查 agent 存在（审批需路由到 agent）
    if (approval.agent === void 0)
        throw new Error(`sandbox escalation to "${mode}" requires approval, but the call has no agent to route it through`);

    // 第97-103行：4. 通过 approval channel 请求人类决策
    const outcome = await approval.approver.request({
        agent: approval.agent,
        toolName: approval.toolName,
        callId: approval.callId,
        reason: `escalate sandbox to ${mode}: ${justification}`,
        ...approval.signal ? { signal: approval.signal } : {}
    });

    // 第104-110行：5. 映射每个结果
    switch (outcome) {
        case "allowed-once": return mode;     // 唯一授权 —— 返回批准的模式
        case "rejected": throw new Error(`the user rejected escalating this ${subject} to "${mode}"`);
        case "cancelled": throw new Error(`approval for escalating to "${mode}" was cancelled`);
        case "unavailable": throw new Error(`sandbox escalation to "${mode}" requires approval, but no approval channel is available`);
        default: return assertNever(outcome, "EscalationOutcome");
    }
}
```

> 🔑 **非更宽请求永远不会提示人类**（第 94 行先抛错）。只有通过严格更宽检查的请求才进入审批通道。这防止了模型用升级参数做降级或平级"探测"。

### 8.5 关键设计：结构性 approval channel（escalation.d.ts 第 10-14 行注释）

`approveEscalation` 接收的是 `EscalationApprover`（一个**结构性函数形状**），而非 approval service 类型。工具层（拥有 agent、callId、toolName）闭包 `ctx.approval.request(...)` 并下传闭包——**这样 `dsh-sandbox` 包永远不依赖 approval 或 agent 包**。依赖方向是单向的：tool → sandbox（escalation）→ structural approver。

### 8.6 升级只对这一次调用有效

> 🔑 `approveEscalation` 返回的 `mode` 被**盖印到这一次调用**的 policy 上（`{ ...policy, mode: approvedMode }`），不影响会话的 standing mode。下一次调用仍然回到会话的有效模式。这是 "allowed-once" 的语义。

### 8.7 ESCALATION_TARGETS 的广告策略（第 34-40 行注释）

schema 的 enum 是 `ESCALATION_TARGETS`（闭集，恒为 `["workspace-write", "danger-full-access"]`），而非根据当前默认模式裁剪。原因：schema 是 registry 全局的，而有效模式是 per-call 真相——若按组合默认裁剪 enum，一个 `danger-full-access` 默认的广告会为空，而一个更窄切换的会话会被困住没有升级杠杆。严格更宽检查在**执行时**做，不在 schema 里固化。

---

## 9. 沙箱与 tool 执行的交互流程

### 9.1 FS 写操作完整流程（以 `write` 工具为例）

```
LLM 调用 write(file_path, content, sandbox_permissions?, justification?)
   │
   ▼
dsh-tool-fs: applyWriteTool 注册的 execute(args, exec)   [index.js 第654行]
   │
   ├─ 1. parseWriteArgs(args)                              [第655行]
   ├─ 2. sandbox.resolvePolicy("write", args, exec)        [第656行]
   │     │
   │     ├─ validateEscalationArgs(sandbox_permissions, justification)  [第1129行]
   │     ├─ standingPolicy = sandboxPolicy.resolve({ session })         [第1130行]
   │     ├─ if 无 sandbox_permissions → 返回 standingPolicy
   │     └─ if 有 → approveEscalation(...)                              [第1134行]
   │           ├─ 检查严格更宽
   │           ├─ ctx.approval.request(...)  ← 阻塞等用户审批
   │           └─ 返回 approvedMode 或抛错
   │     → 返回 { ...standingPolicy, mode: approvedMode }
   │
   ├─ 3. ctx.fs.resolve(filePath, { cwd: sandboxPolicy.workspaceRoot })  [第657行]
   ├─ 4. ctx.waterfall("fs/write-intent", target, exec)  ← 观察策略     [第658行]
   │     └─ fs-observation-policy 决定 createIfAbsent / replaceIfVersion
   │
   ├─ 5. ctx.fs.writeText(target, content, intent, signal, sandboxPolicy)  [第661行]
   │     │  ← 这里进入 SandboxedFileSystem（dsh-fs-sandbox）
   │     │
   │     ├─ checkedTarget(target, sandboxPolicy)              [第130行]
   │     │   ├─ mode == danger-full-access → 返回 target（放行）
   │     │   ├─ mode == read-only → 抛 FS_SANDBOX_DENIED
   │     │   ├─ mode == workspace-write:
   │     │   │   ├─ fresh = this.resolve(target.displayPath)  ← 重新 canonical
   │     │   │   ├─ for root in writableRoots(policy):
   │     │   │   │   └─ isPathUnder(fresh.targetKey, root) ← 遏制检查
   │     │   │   ├─ contained → 返回 fresh
   │     │   │   └─ !contained → 抛 FS_SANDBOX_DENIED
   │     │   └─ 返回 checked target
   │     │
   │     └─ super.writeText(checkedTarget, ...)  ← 进入 LocalFileSystem
   │         ├─ withLock(targetKey)  ← 串行化
   │         ├─ probe / 版本守卫
   │         └─ writeFileAtomic()  ← stage-then-publish 原子写
   │
   ├─ 6a. 成功 → ctx.emit("fs/observed", target, {kind:"present",version})  [第665行]
   │       → 返回 { path, operation, before, after }
   │
   └─ 6b. 失败 → catch:
           ├─ sandbox.mapError(error, sandboxPolicy)   [第663行]
           │   ├─ if FS_SANDBOX_DENIED → 包装成带 denial marker + hint 的 FsError
           │   └─ else → 原样透传
           └─ remediateFsError(...) → throw
```

### 9.2 Bash 命令完整流程

```
LLM 调用 bash(command, description, sandbox_permissions?, justification?, ...)
   │
   ▼
dsh-tool-bash: execute(args, exec)   [index.js 第386行]
   │
   ├─ 1. validateBashArgs(args)                              [第387行]
   │     └─ validateEscalationArgs(sandbox_permissions, justification)  [第123行]
   │
   ├─ 2. standingPolicy = resolveSandboxPolicy(exec)         [第388行]
   │     └─ sandboxPolicy.resolve({ session: exec.agent.session })
   │
   ├─ 3. if sandbox_permissions → approveBashEscalation(...)  [第389行]
   │     ├─ 检查 escalationModes.length > 0
   │     ├─ approveEscalation({ requestedMode, justification, effectiveMode, subject:"command" }, ...)
   │     │   └─ ctx.approval.request(...)  ← 阻塞等用户审批
   │     └─ 返回 approvedMode
   │
   ├─ 4. policy = approvedMode ? {...standingPolicy, mode:approvedMode} : standingPolicy  [第390-393行]
   ├─ 5. resolveWorkdir(args.workdir, exec, standingPolicy?.workspaceRoot)  [第394行]
   ├─ 6. ctx.shellEnv.collect(exec)                          [第395行]
   │
   ├─ 7a. run_in_background → jobs.start(...)                [第403-427行]
   │      └─ ctx.shell.start(ctx.shell.resolve(request))
   │          └─ SandboxBashExecutor.start(spec)             [bash-sandbox 第176行]
   │              ├─ confine(command, policy)  ← ctx.sandbox.confine(["bash","-c",command], policy)
   │              ├─ startArgv(spec, confined.argv)
   │              └─ processFacts.set(proc, {...})  ← 存 per-process 事实
   │
   └─ 7b. 前台 → ctx.shell.run(ctx.shell.resolve({...request, signal}))  [第429行]
          └─ SandboxBashExecutor.run(spec)                   [bash-sandbox 第143行]
              ├─ mode == danger-full-access → super.run()（放行）       [第146行]
              ├─ confine(command, policy)  ← ctx.sandbox.confine(...)   [第153行]
              │   └─ LocalSandboxProvider.confine(argv, policy)
              │       ├─ selectRunner(mode)  ← 平台链 + 探针
              │       │   └─ bwrap / landlock / seatbelt / windows-acl
              │       └─ 返回 { argv: [runner, profile, "--", ...originalArgv], enforcement, denialSignatures, runnerFailureRules }
              ├─ runArgv(spec, confined.argv)  ← spawn 包裹后的 argv
              ├─ classifyRunnerFailure(...)  ← runner 自身失败？
              │   └─ 是 → throw SANDBOX_UNAVAILABLE
              └─ 返回 { ...result, sandbox: { mode, denied: classifyDenial(...), enforcement } }
                  └─ denied = matchesSignature(exitCode, stderr, denialSignatures)
   │
   ▼
渲染: renderResult(value, escalationModes)   [tool-bash 第54行]
   ├─ if sandbox.denied:
   │   ├─ sandboxDenialMarker(mode)  ← "[sandbox: file access denied under <mode> mode]"
   │   └─ escalationHintMarker("command")  ← "[sandbox: escalation available...]"
   └─ + [exit code: N] 等标记
```

### 9.3 完整升级时序图

```
LLM ──write(file, content)──▶ tool-fs.execute
                                │
                                ├─ resolvePolicy (无 sandbox_permissions)
                                │   → standingPolicy { mode: "workspace-write" }
                                │
                                ├─ ctx.fs.writeText(target, content, intent, signal, standingPolicy)
                                │   └─ SandboxedFileSystem.checkedTarget
                                │       └─ !contained → 抛 FS_SANDBOX_DENIED
                                │
                                ├─ sandbox.mapError(error, policy)
                                │   → FsError("[sandbox: file access denied under workspace-write mode]\n[sandbox: escalation available...]")
                                │
                                └─ throw → tool registry → isError result 给 LLM

LLM ◀──isError: "[sandbox: file access denied...]\n[sandbox: escalation available...]"

LLM ──write(file, content, sandbox_permissions="danger-full-access", justification="...")──▶ tool-fs.execute
                                                                                                │
                                                                                                ├─ resolvePolicy
                                                                                                │   ├─ validateEscalationArgs ✓
                                                                                                │   ├─ approveEscalation({
                                                                                                │   │     requestedMode: "danger-full-access",
                                                                                                │   │     effectiveMode: "workspace-write",
                                                                                                │   │     justification, subject:"operation"
                                                                                                │   │   }, { approver: ctx.approval, agent, callId, ... })
                                                                                                │   │   │
                                                                                                │   │   ├─ WIDER_MODES["workspace-write"].includes("danger-full-access") ✓
                                                                                                │   │   ├─ ctx.approval.request({...})
                                                                                                │   │   │   ├─ session.append("approval/asked", {...})
                                                                                                │   │   │   ├─ decide() → waterfall("approval/request", ...)
                                                                                                │   │   │   │   └─ [Web UI 弹窗等待用户]
                                                                                                │   │   │   │       └─ 用户点"允许" → "allowed-once"
                                                                                                │   │   │   └─ session.append("approval/decided", {outcome:"allowed-once"})
                                                                                                │   │   └─ 返回 "danger-full-access"
                                                                                                │   │
                                                                                                │   → 返回 { ...standingPolicy, mode: "danger-full-access" }
                                                                                                │
                                                                                                ├─ ctx.fs.writeText(target, content, intent, signal, {mode:"danger-full-access"})
                                                                                                │   └─ SandboxedFileSystem.checkedTarget
                                                                                                │       └─ mode == "danger-full-access" → 返回 target（放行）✓
                                                                                                │
                                                                                                └─ 成功 → { operation, version, before, after }
```

### 9.4 能力事实的诚实广告

> 🔑 工具层通过 `ctx.shell.sandboxMode` / `ctx.fs.sandboxMode` 读取**能力事实**（是否挂载了隔离后端），决定是否广告 escalation 字段：

```javascript
// dsh-tool-bash 第221-224行
const defaultMode = ctx.shell.sandboxMode;
const escalationModes = defaultMode === void 0 ? [] : ESCALATION_TARGETS;
// 只在隔离后端挂载时才把 sandbox_permissions/justification 加入 schema

// dsh-tool-fs FsSandboxController 第1089-1094行
const defaultMode = ctx.fs.sandboxMode;
this.escalationModes = defaultMode === void 0 ? [] : ESCALATION_TARGETS;
```

`FileSystem.sandboxMode` 基类返回 `undefined`（`dsh-fs/lib/index.js` 第 74 行）；`SandboxedFileSystem` 覆写为部署默认模式（第 115 行）。若挂载的是裸 `fs-local`（不隔离），`sandboxMode` 为 `undefined`，escalation 字段不广告。

### 9.5 拒绝标记的双路径映射

| 路径 | 拒绝来源 | 拒绝形态 | 标记注入点 |
|---|---|---|---|
| FS | 进程内 fence | 结构化 `FS_SANDBOX_DENIED` FsError | `FsSandboxController.mapError()` 包装成带 marker 的 FsError |
| Bash | 内核 stderr | 非零退出 + stderr 子串匹配 | `renderResult()` 在输出末尾追加 marker |

两者都产出**相同的** `[sandbox: file access denied under <mode> mode]` 标记 + `[sandbox: escalation available...]` 提示，让模型统一识别。

---

## 附录 A：进程沙箱后端平台矩阵（dsh-sandbox-local）

| 平台 | 候选链（按优先级） | profile 构建器 | 探针 |
|---|---|---|---|
| Linux | `["bwrap", "landlock"]` | `bwrapProfileArgs` / `landlockProfileArgs` | bwrap: spawn `true`；landlock: `probe()` |
| macOS | `["seatbelt"]` | `seatbeltProfileArgs`（SBPL） | `sandbox-exec -p ... true` |
| Windows | `["windows-acl"]` | `windowsAclRunnerArgv` | restricted-token runner `cmd /c exit 0` |

- **选择逻辑**（第 480-502 行）：平台链唯一候选直接选（不探针）；多候选按链序探针仲裁；无候选/全不可用 → `unavailable`（fail-closed，抛 `SANDBOX_UNAVAILABLE`）
- **enforcement**（第 189-194 行）：bwrap/landlock/seatbelt 声称 `full`；windows-acl 声称 `partial`（WRITE_RESTRICTED 必须保留 Everyone + NTFS hard link 跨路径别名）
- **fail-closed 原则**（`SandboxProvider` 第 193-194 行注释）：`confine()` 必须返回 enforcing argv 或在 wrap/执行时 fail-closed；**禁止静默 unconfined passthrough**

## 附录 B：关键事件类型（session 日志）

| 事件类型 | 写入者 | 含义 |
|---|---|---|
| `sandbox/mode` | `setSandboxMode()` | 会话沙箱模式 override |
| `approval/policy` | `setApprovalPolicy()` | 会话审批策略 override |
| `approval/asked` | `ApprovalService.request()` | 审批提问（审计对上半） |
| `approval/decided` | `ApprovalService.request()` | 审批结果（审计对下半） |
| `permission/preset` | `PermissionPresetService.apply()` | 用户选择的 preset |
| `fs/observed` | `dsh-tool-fs` | 权威文件观察记录 |
| `fs/write-intent` | `dsh-tool-fs` waterfall | 写意图决策 |
| `fs/edit-intent` | `dsh-tool-fs` waterfall | 编辑意图决策 |

## 附录 C：fs-observation-policy — "写前必先读"策略

`dsh-fs-observation-policy`（97 行）是**事件插件**（无服务），注册三个 `fs/*` 监听器：

- `fs/observed`（第 92-94 行）：记录权威 present/absent 观察，keyed by WeakMap(owner=session) → Map(targetKey)
- `fs/write-intent`（第 90 行）：unseen/absent → `createIfAbsent`；present → `replaceIfVersion`
- `fs/edit-intent`（第 91 行）：unseen → 抛 `FS_NOT_OBSERVED`（"edit requires reading first"）；absent → 抛 `FS_NOT_FOUND`；present → 返回 version 作 CAS 基础

> 🔑 这就是为什么系统提示说"read an existing file first (the default fs-observation-policy requires it)"——edit 工具会通过 `fs/edit-intent` 瀑布查询观察状态，未观察过的文件直接拒绝编辑。

---

## 9.6 通用工具审批门控（tools/pre-execute）

除了 `sandbox_permissions` 升级审批，还有一条**通用工具审批通道**，位于 `dsh-tools` 的工具调度器中（`dsh-tools/lib/index.js` 第 3087-3121 行 `prepareExecution`）：

```javascript
// 第3098行：tools/pre-execute 瀑布 —— 可插拔的权限门控
const gate = await this.ctx.waterfall(carrier, "tools/pre-execute", exec,
    () => Promise.resolve({ kind: "allow" }));   // 默认放行

// 第3099-3102行：ask 门控走审批 seam
const askResolution = gate.kind === "ask"
    ? await this.serviceAsk(exec, gate)
    : { decision: gate, approvalCancelled: false };

const { decision } = askResolution;
// 第3109-3121行：deny 或 guardReason → 拒绝（isError 结果）
const denialReason = decision.kind === "allow"
    ? this.guardReason(exec)
    : decision.reason;
if (denialReason !== void 0) return next({ kind: "post-result", result: isErrorResult });
```

### serviceAsk()（第 3296-3347 行）— 审批 seam 的机会性消费

```javascript
async serviceAsk(exec, ask) {
    const approval = this.ctx.get("approval");           // 机会性获取
    if (approval === void 0) return { decision: { kind: "deny", reason: ... } };  // 无服务 → deny
    if (exec.agent === void 0) return { decision: { kind: "deny", reason: ... } }; // 无 agent → deny
    const outcome = await approval.request({ agent, toolName, callId, reason, signal });
    switch (outcome) {
        case "allowed-once": return { decision: { kind: "allow" } };
        case "rejected":     return { decision: { kind: "deny", reason: `the user rejected tool "${exec.name}"` } };
        case "cancelled":    return { decision: { kind: "deny", reason: `... was cancelled` }, approvalCancelled: true };
        case "unavailable":  return { decision: { kind: "deny", reason: `... no approval channel is available` } };
    }
}
```

> 🔑 **两条审批通道对比**：
>
> | 通道 | 触发方式 | 位置 | 适用场景 |
> |---|---|---|---|
> | **`tools/pre-execute` 通用门控** | 插件注册 `tools/pre-execute` 监听器返回 `{kind:"ask"}` | `dsh-tools` 调度器（第 3098 行） | 任何工具的执行前审批（如 hook、自定义权限插件） |
> | **`sandbox_permissions` 升级审批** | 模型在工具参数中传入 `sandbox_permissions` + `justification` | 工具层 `approveEscalation`（`dsh-sandbox` 第 92 行） | 沙箱拒绝后的一次性升级重试 |
>
> 两者都最终走 `ctx.approval.request()`，但触发点和编排不同。通用门控是**工具调度器**在执行前插入的可插拔瀑布；升级审批是**工具 execute 函数**内部、解析策略时主动发起的。

> 🔑 **`never` 策略下两条通道都 fail-closed**：`approval.request()` → `decide()` 第 188 行直接返回 `'rejected'`（不派发 waterfall）→ `serviceAsk` 映射为 `deny` + "the user rejected tool..."。这就是当前 runtime context 中"actions that require approval are rejected automatically"的完整路径。

---

## 总结：设计哲学

1. **单一事实源**：`writableRoots()` 一处定义，Seatbelt 和 fs fence 共享，杜绝 bash/fs 不对称
2. **fail-closed**：无可用后端 → 拒绝执行；无 answerer → `unavailable`；runner 崩溃 → `SANDBOX_UNAVAILABLE`
3. **per-call 策略**：策略不固定在 provider 上，每次调用解析；两个消费者可同时在不同策略下隔离
4. **严格更宽升级**：只能向更宽升级，非更宽请求不提示人类；升级只对一次调用有效
5. **日志即状态**：所有 override 用事件日志存储，重放即恢复，无外部 config store
6. **结构性解耦**：escalation 包用结构性 approver 形状，不依赖 approval/agent 包
7. **读放行写拦截**：fs 只拦变更（write/edit），读原样放行；所有模式都允许读
8. **进程内 fence vs 内核边界**：fs 是 trusted code 对 untrusted path 的策略检查；bash 是 untrusted code 的内核级隔离
