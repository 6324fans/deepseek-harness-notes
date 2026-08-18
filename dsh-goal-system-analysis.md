# DSH Goal 长目标系统实现深度分析

> 本文档基于 `@deepseek-ai/dsh-goal@0.1.0-rc.6`、`@deepseek-ai/dsh-goal-round-driver@0.1.0-rc.6`、`@deepseek-ai/dsh-tool-goal@0.1.0-rc.6`、`@deepseek-ai/dsh-command-goal@0.1.0-rc.6` 四个包的 `lib/` 产物源码分析。所有行号引用对应编译后的 `lib/index.js` 等文件（bundle 内嵌了 fold.js/runtime.js 等 region）。

---

## 0. 架构总览

DSH 的 Goal 系统是一个 **事件溯源（event-sourced）的同一会话长目标系统**，由四个职责分离的包组成：

| 包 | 职责 | 核心导出 |
|---|---|---|
| `dsh-goal` | 领域核心：事件溯源状态、CAS 变更、进程级 continuation 激活 | `GoalService`、`foldGoal`、`decodeGoalChange` |
| `dsh-goal-round-driver` | 自动 continuation 轮次驱动器（race-fenced） | `apply`、`renderGoalRoundPrompt` |
| `dsh-tool-goal` | 模型可调用的 `get_goal`/`create_goal`/`update_goal` 工具 | `apply`、`Config` |
| `dsh-command-goal` | 人类可用的 `/goal` 斜杠命令 | `apply` |

> 🔑 **设计原则**：持久化状态（durable phase/revision）与进程级状态（activation armed/disarmed）严格分离。phase 是可重放的、跨进程一致的；activation 是进程本地的、从不持久化。

依赖关系（基于 package.json 的 peerDependencies）：

```
dsh-command-goal ──┐
dsh-tool-goal ─────┼──► dsh-goal (核心领域)
dsh-goal-round-driver ┘
```

`dsh-goal` 自身依赖 `dsh-session`（事件日志）、`dsh-agent`（agent 注册表）、`dsh-invariants`（不变量守护）、`dsh-typert-protocol`（远程边界）、`dsh-session-projection`（投影注册）。

---

## 1. Goal 的数据模型

### 1.1 类型层次

> 🔑 数据模型分为三层：**Ref（身份）→ Snapshot（持久快照）→ View（运行时视图）**，外加投影层 Projection。

#### GoalId — 身份标识（品牌类型）

`dsh-goal/lib/types/types.d.ts:13-14` / `runtime.js:10-12`

```typescript
export type GoalId = Branded<'GoalId'>;
```

`GoalId` 是 `string & { [BRAND]: 'GoalId' }` 的品牌类型，运行时就是普通字符串，编译时携带品牌以防止类型混用。构造器 `GoalId(id)` 是恒等函数（`runtime.js:10-12`），仅做品牌标注。实际 id 格式为 `goal-${randomUUID()}`（`index.js:573`）。

#### GoalRef — CAS（Compare-And-Set）身份

`dsh-goal/lib/types/types.d.ts:16-21`

```typescript
export interface GoalRef {
    readonly id: GoalId;
    readonly revision: number;  // 正数；每次持久变更自增
}
```

> 🔑 `GoalRef` 是乐观并发控制的核心：所有变更操作（edit/pause/resume/complete/block/clear）都必须传入期望的当前 `{id, revision}`，服务端验证精确匹配后才允许推进到 `revision+1`。见 `expectCurrent()`（`index.js:689-694`）。

#### GoalSnapshot — 持久快照

`dsh-goal/lib/types/types.d.ts:46-55`

```typescript
export interface GoalSnapshot extends GoalRef {
    readonly objective: string;        // 人类请求的完成目标
    readonly phase: GoalPhase;         // 持久生命周期阶段
    readonly blockedReason?: GoalBlockReason;  // 仅 phase='blocked' 时存在
    readonly maxGoalRounds: number;    // 总准许轮次上限
}
```

每次非 clear 的 goal 变更都写入完整快照（whole-value rule）。`blockedReason` 与 phase 强绑定：phase='blocked' 时必须存在，其他 phase 时不存在。

#### GoalView — 运行时视图

`dsh-goal/lib/types/types.d.ts:59-68`

```typescript
export interface GoalView extends GoalSnapshot {
    readonly roundsStarted: number;     // 当前已准许的最高轮次号
    readonly createdAt: number;         // create 变更的时间戳（epoch ms）
    readonly updatedAt: number;         // 最近变更的时间戳（epoch ms）
    readonly activation: GoalActivation; // 进程级；从不持久化
}
```

> 🔑 `GoalView` 是 `GoalService.get()`/`create()` 等方法返回给调用者的完整视图。它合并了持久快照（Snapshot）、从 session 日志派生的计数器（roundsStarted/时间戳）、以及进程级激活状态（activation）。前两者可重放，后者不可。

#### GoalProjection — 会话投影

`dsh-goal/lib/types/types.d.ts:75-94`

```typescript
export interface GoalProjection {
    readonly goal: GoalSnapshot;
    readonly roundsStarted: number;
    readonly createdAt: number;
    readonly updatedAt: number;
}

// 注册到 SessionProjectionMap
declare module '@deepseek-ai/dsh-session-projection/types' {
    interface SessionProjectionMap {
        goal: GoalProjection | null;  // null = create 前 / clear 后
    }
}
```

投影是 last-wins fold 的结果（`applyGoalProjection`，`index.js:376-391`）：每个 `goal/change` 事件携带完整后置状态，所以折叠就是"取最后一个"。注意投影**故意省略 activation**——投影只反映持久 phase。

#### GoalChangeMeta — 持久变更负载

`dsh-goal/lib/types/domain.d.ts:13-32`

```typescript
export type GoalOperation = 'create' | 'edit' | 'pause' | 'resume' | 'complete' | 'block' | 'clear';

export interface GoalSnapshotChangeMeta {  // 非 clear 变更
    readonly kind: 'goal/change';
    readonly version: 1;
    readonly operation: Exclude<GoalOperation, 'clear'>;
    readonly goal: GoalSnapshot;
    readonly roundsStarted: number;
    readonly createdAt: number;
    readonly updatedAt: number;
}

export interface GoalClearChangeMeta {  // clear 墓碑
    readonly kind: 'goal/change';
    readonly version: 1;
    readonly operation: 'clear';
    readonly cleared: GoalRef;
    readonly clearedAt: number;
}

export type GoalChangeMeta = GoalSnapshotChangeMeta | GoalClearChangeMeta;
```

> 🔑 `version: 1` 是协议版本号（`GOAL_CHANGE_VERSION = 1`，`runtime.js:4`）。decode 时若 version 不符则 fail-loud（`fold.js:107-109`）。变更负载通过 `declare module '@deepseek-ai/dsh-session/types'` 注册为 `SessionEventMap['goal/change']`（`domain.d.ts:46-53`），即 goal 变更是 session 事件流的一等公民。

#### GoalMessageSource — 轮次消息归属

`dsh-goal/lib/types/domain.d.ts:33-45`

```typescript
export interface GoalMessageSource {
    readonly kind: 'goal';
    readonly goalId: GoalId;
    readonly revision: number;
    readonly round: number;  // 正数：准许的 continuation 轮次
}
```

注册到 `MessageSourceMap['goal']`（`domain.d.ts:41-45`）。round=0 的 goal 消息是"round-zero"（create 时注入的目标提示），round>0 是自动 continuation 轮次。driver 通过 `isGoalRoundSource()` 判定（`goal-round-driver/index.js:32-34`：`source.kind === "goal" && source.round > 0`）。

### 1.2 Fold 累加器

`dsh-goal/lib/types/fold.d.ts:6-13`

```typescript
export interface GoalFoldState {
    goal: GoalSnapshot | undefined;
    roundsStarted: number;
    createdAt: number | undefined;
    updatedAt: number | undefined;
    lastRef: GoalRef | undefined;
    seenGoalIds: Set<GoalSnapshot['id']>;  // 防止 id 复用
}
```

> 🔑 `seenGoalIds` 是一个防重 Set：create 时若 id 已被见过则 fail-loud（`fold.js:261-262`），确保同一 session 内不会有两个 goal 共用 id。注意 create 允许在 `phase='complete'` 后替换（`fold.js:259-260`），但新 goal 必须是新 id、revision=1、phase='active'、roundsStarted=0。

---

## 2. Goal 的状态机

### 2.1 Phase 集合

`dsh-goal/lib/index.js:40-45`（内联 fold region）

```javascript
const PHASES = new Set(["active", "paused", "blocked", "complete"]);
```

`GoalPhase = 'active' | 'paused' | 'blocked' | 'complete'`（`types.d.ts:37`）。

### 2.2 合法状态转移

状态转移规则在 `validateSnapshotTransition()`（`fold.js:169-222` / `index.js:176-212`）中严格校验。下表整理所有合法转移：

| 当前 phase | 操作 | 目标 phase | 前置条件 | activation |
|---|---|---|---|---|
| (无) | `create` | `active` | revision=1, roundsStarted=0, 新 id 或上一个 goal 已 complete | armed |
| `active` | `edit` | `active` | objective/maxGoalRounds 至少一项变化 | 保留当前 |
| `active` | `pause` | `paused` | — | disarmed |
| `active` | `complete` | `complete` | — | disarmed |
| `active` | `block` | `blocked` | 需 blockedReason | disarmed |
| `active` | `resume` | `active` | 仅当当前 disarmed（rearm 场景）；roundsStarted < maxGoalRounds | armed |
| `paused` | `resume` | `active` | roundsStarted < maxGoalRounds | armed |
| `blocked` | `resume` | `active` | roundsStarted < maxGoalRounds | armed |
| `blocked` | `complete` | `complete` | — | disarmed |
| `paused` | `complete` | `complete` | — | disarmed |
| `complete` | `create`(新) | `active` | 新 id, revision=1 | armed |
| 任意(非complete) | `clear` | (无) | 墓碑 revision+1 | disarmed |

> 🔑 **关键约束**：
> - `edit` 不能改变 phase 或 blockedReason（`fold.js:181-186`）。
> - `pause`/`resume`/`complete`/`block` 不能改变 objective 或 maxGoalRounds（`requireSameDefinition`，`fold.js:157-161`）。只有 `edit` 能改这两项。
> - `block` 只能从 `active` 转入（`fold.js:209-212`）。
> - `resume` 允许从 `active`（rearm 场景）、`paused`、`blocked` 转入 `active`，但**必须 round 预算未耗尽**（`fold.js:199`：`state.roundsStarted >= next.maxGoalRounds` 则失败）。
> - `complete` 不允许从 `complete` 再次 complete（`fold.js:205-207`）。

### 2.3 转移不变量（strict fold）

`validateSnapshotTransition()` 还校验三类不变量：

1. **revision 精确递增**（`requireNextRevision`，`fold.js:163-166`）：`next.revision === current.revision + 1` 且 `next.id === current.id`。
2. **计数器/时间戳保留**（`fold.js:175-179`）：非 create 变更必须保留 `createdAt`、`roundsStarted`，且 `updatedAt >= state.updatedAt`。
3. **phase-specific 字段集**（`decodeSnapshot`，`fold.js:73-78`）：blocked 快照必须有且仅有 `blockedReason,id,maxGoalRounds,objective,phase,revision`；其他 phase 必须有且仅有 `id,maxGoalRounds,objective,phase,revision`。

> 🔑 这些不变量有两层校验：**写侧**（GoalService 在 append 前构造合法变更）和**读侧/重放侧**（strict fold 在 decode 时 fail-loud）。后者由 `dsh-goal/invariant.js` 的 companion 插件独立验证每一个待发布事件（见第 7 节）。

---

## 3. Goal 的 CRUD — GoalService

`GoalService` 是 `TypertRemoteService` 的子类（`index.js:421-825`），通过 Cordis 注册为 `ctx.goals` 服务。

### 3.1 构造与配置

`index.js:513-532`

```javascript
static inject = ["agents"];
static Config = z.object({ defaultMaxGoalRounds: z.number().default(256) });
caches = new WeakMap();  // session -> cache

constructor(ctx, config = {}) {
    super(ctx, "goals");
    this.resolved = { defaultMaxGoalRounds: resolveMaxGoalRounds(config.defaultMaxGoalRounds ?? 256) };
    ctx.on("agent/session-start", ({ agent }) => {
        this.cache(agent.session).activation = "disarmed";  // 新 session 边界 disarm
    });
    ctx.inject(["sessionProjections"], (projectionCtx) => {
        projectionCtx.sessionProjections.register({
            key: "goal", schema: goalProjectionSchema, init: () => null,
            apply: applyGoalProjection, view: (state) => state, stateVersion: 4
        });
    });
}
```

> 🔑 **默认 maxGoalRounds = 256**。每个 agent session-start 边界会将 activation 设为 `disarmed`——这正是"session resume/fork 后需要人类显式 resume 才能 rearm"的实现机制。

### 3.2 读操作：get(agent)

`index.js:539-544`

```javascript
get(agent) {
    this.assertLive(agent);                    // 验证 agent 是注册表中的 live 实例
    const cache = this.cache(agent.session);   // 获取/创建 per-session 缓存
    this.sync(agent.session, cache);           // 增量同步新事件
    return this.view(cache);                   // 返回 GoalView | undefined
}
```

`assertLive`（`index.js:696-698`）通过引用相等（`this.ctx.agents.get(agent.id) !== agent`）而非 id 匹配来验证 live 身份——防止过期 agent 引用操作。

### 3.3 创建：create(agent, request)

`index.js:566-580`

```javascript
create(agent, request) {
    const spec = resolveCreateGoal(request, this.resolved.defaultMaxGoalRounds);
    const cache = this.prepareMutation(agent);
    const current = cache.state.goal;
    if (current !== void 0 && current.phase !== "complete")
        throw new GoalError(`goal "${current.id}" already exists with phase "${current.phase}"`, "GOAL_ALREADY_EXISTS");
    const now = Date.now();
    const goal = {
        id: GoalId(`goal-${randomUUID()}`),
        revision: 1, objective: spec.objective,
        phase: "active", maxGoalRounds: spec.maxGoalRounds
    };
    return this.commitSnapshot(agent, cache, "create", goal, 0, now, now, "armed");
}
```

> 🔑 **create 的前置条件**：当前无 goal，或当前 goal 已 `complete`（允许替换）。create 后 activation=`armed`，roundsStarted=0，createdAt=updatedAt=now。

`resolveCreateGoal`（`index.js:403-408`）规范化 objective（trim，非空）和 maxGoalRounds（正安全整数，默认 256）。

### 3.4 编辑：edit(agent, ref, request)

`index.js:588-599`

```javascript
edit(agent, ref, request) {
    const cache = this.prepareMutation(agent);
    const current = this.expectCurrent(cache, ref);  // CAS 验证
    if (request.objective === void 0 && request.maxGoalRounds === void 0)
        throw new GoalError("goal edit requires objective and/or maxGoalRounds", "GOAL_INVALID_EDIT");
    const goal = {
        ...current, revision: current.revision + 1,
        ...request.objective === void 0 ? {} : { objective: resolveObjective(request.objective) },
        ...request.maxGoalRounds === void 0 ? {} : { maxGoalRounds: resolveMaxGoalRounds(request.maxGoalRounds) }
    };
    return this.commitCurrent(agent, cache, "edit", goal, cache.activation);  // 保留当前 activation
}
```

edit 保留当前 activation（armed 仍 armed，disarmed 仍 disarmed），不改变 phase。

### 3.5 暂停：pause(agent, ref)

`index.js:606-608`

```javascript
pause(agent, ref) {
    return this.transition(agent, ref, "pause", ["active"], "paused", "disarmed");
}
```

通过共享的 `transition()` 辅助方法（`index.js:733-738`）：验证 current.phase ∈ allowed，构造 withPhase(current, phase)，commit 并设置 activation。pause 强制 disarmed。

### 3.6 恢复：resume(agent, ref)

`index.js:616-628`

```javascript
resume(agent, ref) {
    const cache = this.prepareMutation(agent);
    const current = this.expectCurrent(cache, ref);
    const resumable = ["active", "paused", "blocked"];
    if (!resumable.includes(current.phase)) throw this.transitionError(current, "resume", resumable);
    if (current.phase === "active" && cache.activation === "armed")
        throw new GoalError(`goal "${current.id}" is already active and armed`, "GOAL_INVALID_TRANSITION");
    if (cache.state.roundsStarted >= current.maxGoalRounds)
        throw new GoalError(`goal "${current.id}" exhausted ${current.maxGoalRounds} goal rounds; increase maxGoalRounds before resuming`, "GOAL_INVALID_TRANSITION");
    return this.commitCurrent(agent, cache, "resume", this.withPhase(current, "active"), "armed");
}
```

> 🔑 **resume 的三个前置条件**：
> 1. phase ∈ {active, paused, blocked}（complete 不可 resume）。
> 2. 若已 active+armed 则拒绝（幂等保护，避免重复 arm）。
> 3. **round 预算未耗尽**：`roundsStarted < maxGoalRounds`。耗尽则需先 edit 增加 maxGoalRounds。
> resume 后 activation=`armed`——这是唯一让 disarmed 的 active goal 重新获得自动 continuation 权限的途径。

### 3.7 完成：complete(agent, ref)

`index.js:635-641`

```javascript
complete(agent, ref) {
    return this.transition(agent, ref, "complete", ["active", "paused", "blocked"], "complete", "disarmed");
}
```

从 active/paused/blocked 转入 complete，强制 disarmed。

### 3.8 阻塞：block(agent, ref, reason)

`index.js:649-657`

```javascript
block(agent, ref, reason) {
    const cache = this.prepareMutation(agent);
    const current = this.expectCurrent(cache, ref);
    if (current.phase !== "active") throw this.transitionError(current, "block", ["active"]);
    return this.commitCurrent(agent, cache, "block", {
        ...this.withPhase(current, "blocked"),
        blockedReason: resolveBlockReason(reason)
    }, "disarmed");
}
```

block 只能从 active 转入 blocked，需 `blockedReason`（code + message），强制 disarmed。`resolveBlockReason`（`index.js:410-419`）校验 code 为 lower-kebab-case、message 非空并 trim。

### 3.9 清除：clear(agent, ref)

`index.js:664-680`

```javascript
clear(agent, ref) {
    const cache = this.prepareMutation(agent);
    const current = this.expectCurrent(cache, ref);
    const tombstone = { id: current.id, revision: current.revision + 1 };
    const change = {
        kind: "goal/change", version: 1, operation: "clear",
        cleared: tombstone, clearedAt: this.nextMutationTime(cache)
    };
    this.commit(agent, cache, change, "disarmed");
    return { ...tombstone };
}
```

> 🔑 **clear 与其他操作不同**：它不写快照，而是写一个 **墓碑（tombstone）**，记录 `{cleared: {id, revision}, clearedAt}`。fold 时遇到 clear 会将 goal 设为 undefined（`fold.js:240-256`）。clear 保留历史（事件仍在日志中），只是当前投影变 null。

### 3.10 提交流水线

所有变更最终通过 `commit()`（`index.js:775-794`）落地：

```javascript
commit(agent, cache, change, activation) {
    const ref = goalChangeRef(change);
    cache.pendingActivation = { seq: agent.session.seq, activation };  // 预登记激活意图
    try {
        agent.session.append("goal/change", change);  // 写入 session 事件日志（持久化）
        this.sync(agent.session, cache);               // 增量重放并确认激活
    } finally {
        cache.pendingActivation = void 0;
    }
    const goal = this.view(cache);
    const notification = { operation: change.operation, ref: {...ref}, ...goal === void 0 ? {} : { goal } };
    agentEvents(this.ctx, agent).emit("goal/changed", { change: notification });  // 通知监听者
}
```

> 🔑 **pendingActivation 机制**：`commit` 在 append 前把期望的 activation 与"即将写入的 seq"绑定。`sync()` 重放该事件时（`index.js:718`），若事件的 seq 匹配 pendingActivation.seq，则采用 pending 的 activation；否则 disarm。这保证了：只有本进程发起的变更才携带激活意图，重放/其他来源的变更默认 disarm。

### 3.11 缓存与增量同步

`cache()`（`index.js:700-713`）：首次访问时 fold 全部 session.events 种子化缓存，activation=disarmed。

`sync()`（`index.js:715-721`）：增量消费 `session.events.slice(observedSeq)`，逐事件 applyGoalEvent，goal/change 事件时按 pendingActivation 协调 activation。

> 🔑 缓存是 `WeakMap<session, cache>`——session 被回收时缓存自动清理。observedSeq 跟踪已消费到哪个事件，避免重复 fold。

---

## 4. Goal Round Driver — 自动 continuation 轮次驱动

`dsh-goal-round-driver/lib/index.js` 是"自动续跑"的核心。它监听 agent 状态变化，在 goal active+armed 且 agent idle 时，自动注入下一轮 continuation 消息。

### 4.1 驱动器状态

`index.js:56-72`

```javascript
const states = new Map();  // agent -> driver state

function stateFor(agent) {
    const existing = states.get(agent);
    if (existing !== void 0) return existing;
    const state = {
        agent,
        attempt: void 0,        // 当前预约的轮次 {goalId, revision, round, messageId, content, phase, cancelled, stale}
        competingQueued: false, // 是否有竞争性消息排队
        needsCheckpoint: false, // 是否需要 durability checkpoint
        requested: false,       // 是否已请求 drive
        run: void 0,            // 当前序列化任务 Promise
        stopping: false         // 拆卸中
    };
    states.set(agent, state);
    return state;
}
```

### 4.2 核心驱动逻辑：drive()

`index.js:103-164`，这是续跑的主逻辑：

```javascript
async function drive(state) {
    const { agent } = state;
    if (!readyToDrive(state)) return;                    // 1. 就绪检查
    if (state.needsCheckpoint) {                         // 2. 持久化 checkpoint
        state.needsCheckpoint = false;
        try { await ctx.sessions.flush(agent.session); }
        catch (error) { ...; disarm(state); return; }
        if (!readyAfterCheckpoint(state)) return;
    }
    if (state.attempt !== void 0) {                      // 3. 上一轮已 admitted，预约下一轮
        state.attempt = void 0;
        state.needsCheckpoint = true;
        state.requested = true;
        return;
    }
    const goal = currentGoal(state);                     // 4. 读取当前 goal
    if (goal === void 0 || goal.phase !== "active" || goal.activation !== "armed") return;
    if (goal.roundsStarted >= goal.maxGoalRounds) {      // 5. 轮次耗尽 → 自动 block
        ctx.goals.block(agent, goalRef(goal), {
            code: "round-limit",
            message: `Goal reached its configured limit of ${goal.maxGoalRounds} rounds.`
        });
        return;
    }
    const round = goal.roundsStarted + 1;                // 6. 构造下一轮消息
    const content = renderGoalRoundPrompt(goal, round);
    const message = createUserMessage({ content, source: { kind: "goal", goalId: goal.id, revision: goal.revision, round } });
    state.attempt = { goalId: goal.id, revision: goal.revision, round, messageId: message.id, content, phase: "queued", cancelled: false, stale: false };
    try { agent.followup(message); }                     // 7. 排队消息
    catch (error) { ...; ctx.goals.block(..., { code: "queue-failed", ... }); }
}
```

> 🔑 **drive 的决策序列**：
> 1. **就绪检查** `readyToDrive`：fiber 空闲(state===2)、未 stopping、agent 仍 live、agent.status==='idle'、无 competingQueued。
> 2. **checkpoint**：若 needsCheckpoint（goal/changed 或上一轮完成后设置），先 `sessions.flush()` 确保持久化，再继续。
> 3. **轮次预约**：若 attempt 仍在（上一轮已 admitted），清空 attempt、标记需要 checkpoint、再次请求 drive——形成"每轮之间 checkpoint"的节奏。
> 4. **goal 检查**：仅当 phase='active' && activation='armed' 才续跑。
> 5. **轮次耗尽保护**：`roundsStarted >= maxGoalRounds` 时自动 block，code='round-limit'。
> 6. **构造 prompt**：`renderGoalRoundPrompt(goal, round)`。
> 7. **排队**：`agent.followup(message)` 将消息加入 inbox。

### 4.3 Continuation Prompt 渲染

`index.js:11-18`（`prompt.js` region）

```javascript
function renderGoalRoundPrompt(goal, round) {
    return [{
        type: "text",
        text: `<goal_round>
Objective: ${JSON.stringify(goal.objective)}
Round: ${round}/${goal.maxGoalRounds}

Continue working toward the objective in this same session. Treat the current workspace, tool results, and durable session state as authoritative; inspect them instead of assuming earlier narration is still current. Make concrete progress and verify the result. Before claiming completion, gather evidence that the whole objective is achieved, read the current goal, and mark it complete. If work remains, leave the goal active for the next round. Follow the configured goal-tool policy before reporting a blocker.
</goal_round>`
    }];
}
```

> 🔑 prompt 是**确定性函数**：给定相同 (goal, round) 必产生相同 content。这一点被 invariant companion 利用——它会重放 fold 重构 goalView，再用 `renderGoalRoundPrompt` 重新渲染，与事件中的 content 做 `isDeepStrictEqual` 比对（`goal-round-driver/invariant.js:48-54`）。任何篡改都会 fail-loud。

### 4.4 序列化驱动：requestDrive()

`index.js:166-199`

```javascript
function requestDrive(state) {
    if (state.stopping) return;
    state.requested = true;
    if (state.run !== void 0) return;               // 已有运行中的任务，合并请求
    let run;
    try {
        run = ctx.agents.withoutInitiator(async () => {
            while (state.requested && !state.stopping) {   // 自旋直到无新请求
                state.requested = false;
                try { await drive(state); }
                catch (error) { ...; disarm(state); }
            }
        });
    } catch (error) { ...; disarm(state); return; }
    state.run = run;
    const retire = () => {
        state.run = void 0;
        if (state.requested && !state.stopping) requestDrive(state);  // 有新请求则重启
    };
    run.then(retire, (error) => { ...; disarm(state); retire(); });
}
```

> 🔑 **coalesce 合并**：多次触发（goal/changed、agent/status idle 等）合并到单个序列化循环。`withoutInitiator` 确保驱动任务不持有 initiator 锁（避免与正在运行的 agent turn 冲突）。循环退出后若又有 requested，则重启——保证不丢触发。

### 4.5 事件监听与触发

`index.js:200-274`，driver 注册了丰富的事件监听：

| 事件 | 处理 |
|---|---|
| `agent/error` | disarm（移除自动权限） |
| `agent/created` | stateFor(agent) 初始化 |
| `agent/disposed` | states.delete(agent) |
| `agent/session-start` | 重置 attempt/competingQueued/needsCheckpoint |
| `agent/status` (idle) | 检查 cancelled attempt → pause；requestDrive |
| `goal/changed` | needsCheckpoint=true; requestDrive |
| `agent/inbox/inserted` | 检测竞争消息；标记 attempt.stale |
| `agent/inbox/claimed` | 匹配的 attempt → phase='claimed' |
| `agent/inbox/discarded` | 匹配的 attempt → cancelled=true |
| `session/event` (user/message) | 匹配 attempt → phase='admitted' |
| `session/event` (turn/end max-tokens) | disarm |
| `session/event` (turn/end aborted) | claimed/admitted → cancelled；否则 disarm |
| `agent/pre-step` | **核心 race fence**（见下） |

### 4.6 Pre-step Race Fence

`index.js:281-339`，这是防止"过期预约消息进入 step"的最后一道闸门：

```javascript
ctx.on("agent/pre-step", async ({ agent, messages, signal }, next) => {
    const submitted = messages.find((message) => isGoalRoundSource(message.source));
    if (submitted === void 0) return next();       // 非 goal round 消息，放行
    const { content, source } = submitted;
    const state = stateFor(agent);
    let valid = false;
    try { valid = validReservation(state, content, source); }
    catch (error) { ...; disarm(state); }
    if (!valid) {                                   // 预约失效
        // 标记 stale，恢复其他 claimed 消息，重新 requestDrive
        ...
        return { kind: "reject" };
    }
    let decision;
    try { decision = await next(); }                // 放行进入 step
    catch (error) {
        if (signal.aborted) throw error;
        state.attempt = void 0; requestDrive(state); throw error;
    }
    if (signal.aborted) { ...; return decision; }
    if (decision.kind === "reject") {               // step 被拒绝 → block
        state.attempt = void 0;
        ctx.goals.block(agent, goalRef(goal), { code: "prompt-rejected", ... });
        return decision;
    }
    // post-decision 二次校验
    try { valid = validReservation(state, content, source); }
    catch (error) { ...; valid = false; }
    if (!valid) { ...; return { kind: "reject" }; }
    return decision;
});
```

`validReservation()`（`index.js:276-280`）的完整校验条件：

```javascript
return ctx.fiber.state === 2 && !state.stopping
    && attempt !== void 0 && attempt.phase === "claimed" && !attempt.stale
    && sameQueued(content, source, attempt)
    && goal !== void 0 && goal.id === source.goalId && goal.revision === source.revision
    && goal.phase === "active" && goal.activation === "armed"
    && source.round === goal.roundsStarted + 1;
```

> 🔑 **双重校验**（pre-decision 和 post-decision）确保：即使在 step 进入期间 goal 被外部改变（如人类 pause），也不会让过期轮次继续。失效时调用 `restoreOtherClaimed()`（`index.js:95-101`）恢复被挤占的其他 next-step 消息，避免丢消息。

### 4.7 自动 block 的 code

driver 在三种异常下自动 block：

| 场景 | code | 触发点 |
|---|---|---|
| 轮次耗尽 | `round-limit` | `index.js:126-129` |
| 排队失败 | `queue-failed` | `index.js:159-162` |
| pre-step 拒绝 | `prompt-rejected` | `index.js:319-322` |

---

## 5. Goal 的 Action 类型

### 5.1 工具层 action 枚举

`dsh-tool-goal/lib/index.js:110-116`

```javascript
const UPDATE_ACTIONS = ["edit", "pause", "resume", "complete", "blocked"];
```

> 🔑 注意工具层用 `blocked`（过去分词），而领域层 operation 用 `block`（动词）。工具层在 `update_goal` 的 `action` 参数枚举中暴露 `blocked`，内部映射到 `ctx.goals.block()`（`tool-goal/index.js:353`）。

### 5.2 领域层 operation 枚举

`dsh-goal/lib/types/domain.d.ts:12`

```typescript
export type GoalOperation = 'create' | 'edit' | 'pause' | 'resume' | 'complete' | 'block' | 'clear';
```

七个操作，其中 `create` 和 `clear` 不在 `update_goal` 工具的 action 枚举中（create 有独立工具，clear 仅命令可用）。

### 5.3 工具层 action 的权限矩阵

`dsh-tool-goal/lib/index.js:329-367`（update_goal execute）：

| action | 权限要求 | 额外参数 |
|---|---|---|
| `edit` | `requireDirectHuman`（直接人类回合） | objective 和/或 max_goal_rounds |
| `pause` | `requireDirectHuman` | 无 |
| `resume` | `requireDirectHuman` | 无 |
| `complete` | `completionAuthority`（直接人类 **或** 当前 goal round） | 无 |
| `blocked` | `completionAuthority`（直接人类 **或** 当前 goal round） | blocked_reason（必填） |

> 🔑 **权限分层**：`edit`/`pause`/`resume` 只允许直接人类操作（防止模型擅自改变目标定义或暂停/恢复）；`complete`/`blocked` 允许模型在自动 continuation 轮次中自主报告（这是自动续跑的终止机制）。

### 5.4 权限解析

`dsh-tool-goal/lib/types/authority.js`：

- `goalToolExecution()`（`authority.js:30-38`）：验证调用 agent 是 live、running、且是 currentInitiator，并定位当前 open turn。
- `hasDirectHumanInput()`（`authority.js:44-47`）：当前 root-agent turn 中是否有 `source.kind === 'user'` 的消息。
- `isMatchingGoalRound()`（`authority.js:49-51`）：当前 turn 是否是 goal 的精确 admitted round（goalId/revision/round 匹配）。
- `completionAuthority()`（`authority.js:67-75`）：优先 direct-human，否则 goal-round；都不满足则 reject。

### 5.5 命令层 action

`dsh-command-goal/lib/index.js:17-33`（`parseGoalCommand`）：

| 输入 | 解析为 | 说明 |
|---|---|---|
| 空 | `show` | 显示当前 goal |
| `clear` | `clear` | 清除 |
| `pause` | `pause` | 暂停 |
| `resume` | `resume` | 恢复 |
| `edit` | `invalid-edit` | 报错（需 objective） |
| `edit <text>` | `edit` | 编辑 objective |
| 其他任意文本 | `create` | 视为新 objective |

> 🔑 命令层用当前 goal 的 ref（`ctx.goals.get` 后取 id/revision），无需人类手动传 revision——命令是同步读-改，不存在 CAS 竞争。而工具层要求模型先 `get_goal` 再传 `goal_id`+`revision`，因为模型调用是异步的，可能有并发变更。

---

## 6. Blocked 条件 — 最小轮数与 blocked_reason

### 6.1 最小轮数阈值

`dsh-tool-goal/lib/index.js:109`

```javascript
const Config = z.object({ blockedAfterConsecutiveRounds: z.number().step(1).min(1).default(3) });
```

默认 **3 轮**。`resolveConfig()`（`index.js:193-197`）校验为正安全整数。

### 6.2 blocked 的前置条件

`dsh-tool-goal/lib/index.js:351-352`

```javascript
if (args.action === "blocked" && authority.kind === "goal-round"
    && authority.goal.roundsStarted < resolved.blockedAfterConsecutiveRounds)
    throw new HarnessError(
        `blocked requires at least ${resolved.blockedAfterConsecutiveRounds} consecutive goal rounds; current round is ${authority.goal.roundsStarted}`,
        "GOAL_TOOL_BLOCK_THRESHOLD"
    );
```

> 🔑 **最小轮数仅约束 goal-round 权限**：如果 authority 是 `direct-human`（人类直接操作），则不受最小轮数限制——人类可以随时 block。只有当模型在自动 continuation 中尝试 block 时，才要求 `roundsStarted >= blockedAfterConsecutiveRounds`（默认 3）。这防止模型在第一轮就轻易放弃。

### 6.3 blocked_reason 的结构

模型通过 `update_goal` 的 `blocked_reason` 参数传入**字符串**（`tool-goal/index.js:323-326`），工具层包装为：

```javascript
ctx.goals.block(execution.agent, ref, {
    code: "model-reported",        // 固定 code
    message: args.blocked_reason   // 模型提供的消息
});
```

领域层 `resolveBlockReason()`（`index.js:410-419`）校验：
- `code`：必须匹配 `^[a-z][a-z0-9]*(?:-[a-z0-9]+)*$`（lower-kebab-case）。
- `message`：非空，trim 后存储。

> 🔑 模型报告的 block 固定用 code=`model-reported`；driver 自动 block 用 `round-limit`/`queue-failed`/`prompt-rejected`。code 是机器可路由的分类，message 是人类可读的解释。

### 6.4 系统提示词中的 policy

`dsh-tool-goal/lib/index.js:189-191`（`guidance`）：

```
Mark blocked only after the same blocking condition persists for at least ${blockedAfter} consecutive goal rounds,
and report that concrete condition in blocked_reason; difficulty, uncertainty, or useful remaining work is not blocked.
```

> 🔑 policy 明确区分"阻塞"与"困难"：只有**同一阻塞条件持续 N 轮**才应 block。模型需在 blocked_reason 中描述具体阻塞条件，而非笼统的"太难了"。但注意：这是 prompt 层的软约束，硬约束只有轮数阈值。

### 6.5 Wrap-up 上下文注入

当 goal-round 权限的 `complete`/`blocked` 执行后，工具层通过 `exec.deferContext()` 注入 wrap-up 指令（`tool-goal/index.js:357-365`）：

`renderWrapupContext()`（`tool-goal/index.js:86-93`）生成 `<goal_complete>` 或 `<goal_blocked>` 指令，要求模型写一段面向用户的总结消息（成果、验证、后续建议），且**不再调用工具**。

> 🔑 这取代了"硬停止 turn"的旧设计：终端操作后不强制 turn 结束，而是让模型有机会向用户交代，然后自然结束。wrap-up 是 deferred context（notice form），在当前 step 之后注入。

---

## 7. Goal 与 Session 的关系 — 持久化与 Resume

### 7.1 事件溯源模型

> 🔑 Goal 状态**完全**由 session 事件日志决定。`GoalService` 不维护独立存储——所有变更通过 `agent.session.append("goal/change", change)`（`index.js:782`）写入 session 事件流。当前状态通过 fold 事件日志重建。

session 事件类型注册（`domain.d.ts:46-53`）：

```typescript
declare module '@deepseek-ai/dsh-session/types' {
    interface SessionEventMap {
        'goal/change': GoalChangeMeta;
    }
}
```

### 7.2 两层 Fold

**严格 fold（strict）** — `foldGoal()`（`fold.js:310-321`）：
- 用于 invariant companion、缓存种子化、round-driver invariant 重构。
- fail-loud：任何非法变更抛 Error。
- 维护 `seenGoalIds` Set。
- 状态是可变 `GoalFoldState`（含 Set）。

**投影 fold（projection-grade）** — `applyGoalProjection()`（`index.js:376-391`）：
- 注册为 session projection unit（`index.js:522-531`）。
- last-wins：取最后一个 goal/change 的完整值。
- 容错：decode 失败时返回原 state（不 fail），因为投影是缓存前置条件。
- 状态是纯 JSON（无 Set），符合 persisted-cache 要求。
- clear → null；其他 → `{goal, roundsStarted, createdAt, updatedAt}`。

> 🔑 **两层设计的原因**：投影用于持久化缓存（需纯 JSON、容错）；strict fold 用于正确性验证（需 fail-loud、完整不变量）。写侧（GoalService）保证写入合法；invariant companion 独立验证；投影仅做 last-wins 快照。职责分离。

### 7.3 轮次的持久化

`roundsStarted` **不是**通过 goal/change 事件递增的——它通过 `user/message` 事件中的 `GoalMessageSource.round` 字段推导。

`applyGoalEvent()`（`fold.js:283-304`）处理两种事件：
1. `goal/change` → applyGoalChange（更新快照）。
2. `user/message` → 若 source.kind==='goal'，校验 round 是"下一个准许轮次"（`source.round === state.roundsStarted + 1` 且 `<= maxGoalRounds`），然后 `state.roundsStarted = source.round`。

> 🔑 这意味着 **roundsStarted 由实际 admitted 的 user/message 事件决定**，而非 goal/change 快照中的 roundsStarted 字段。快照中的 roundsStarted 是"写入时的快照值"，但 fold 会用后续 user/message 事件覆盖它。round-driver 在 `drive()` 中构造 `source.round = goal.roundsStarted + 1` 的消息，当该消息被 session 接收为 user/message 事件时，fold 才推进 roundsStarted。

### 7.4 Resume 机制

> 🔑 **Resume 分两种语义**：

**A. Session resume/fork 后的 rearm**：
- 新进程加载 session 时，`cache()` 种子化 fold 全部事件，activation 设为 `disarmed`（`index.js:707`）。
- `agent/session-start` 事件也强制 disarm（`index.js:519-521`）。
- 此时 goal 的持久 phase 可能是 `active`，但 activation 是 `disarmed`——driver 不会自动续跑。
- 人类说"继续"时，模型调用 `update_goal action=resume`，`resume()` 检查 `current.phase === "active" && cache.activation === "armed"` 时拒绝（`index.js:625`），否则允许 rearm。

**B. 从 paused/blocked 恢复到 active**：
- phase 从 paused/blocked 转为 active，activation 设为 armed。
- 需 round 预算未耗尽。

### 7.5 不变量守护（Invariant Companion）

`dsh-goal/lib/invariant.js`：注册名为 `goal-invariant` 的 companion，对**每一个待发布的 session/event** 进行独立 strict fold 验证。

`install()`（`invariant.js:31-63`）：
- 用 `WeakMap<session, state>` 维护独立 fold 状态（与 GoalService 的缓存隔离）。
- `internal/dispatch` 阶段：clone 当前状态，apply 候选事件，staging 结果。
- `session/event` 阶段：确认 staging 匹配后提交；否则 fail。
- 这是在事件**发布前**验证，确保非法变更不会进入持久日志。

`dsh-goal-round-driver/lib/invariant.js`：注册 `goal-round-driver-invariant`，验证每个 goal round 的 user/message 事件的 content 与 `renderGoalRoundPrompt` 严格相等（防篡改）。

> 🔑 **双 invariant**：goal-invariant 守护状态机合法性；goal-round-driver-invariant 守护 continuation prompt 的确定性。两者都注册到 `ctx.invariants.register(PACKAGE_NAME, install)`，由 `dsh-invariants` 服务在事件发布前统一调度。

---

## 8. max_goal_rounds 限制

### 8.1 配置层级

| 层级 | 字段 | 默认值 | 位置 |
|---|---|---|---|
| 部署配置 | `defaultMaxGoalRounds` | 256 | `dsh-goal Config`（`index.js:513`） |
| 工具参数 | `max_goal_rounds`（create_goal） | 继承默认 | `tool-goal/index.js:278-281` |
| 工具参数 | `max_goal_rounds`（update_goal edit） | — | `tool-goal/index.js:319-322` |
| 持久快照 | `maxGoalRounds` | — | `GoalSnapshot` |

### 8.2 校验

`resolveMaxGoalRounds()`（`index.js:393-396`）：

```javascript
function resolveMaxGoalRounds(value) {
    if (!Number.isSafeInteger(value) || value < 1)
        throw new GoalError("maxGoalRounds must be a positive safe integer", "GOAL_INVALID_MAX_ROUNDS");
    return value;
}
```

必须是**正安全整数**（`Number.isSafeInteger` 且 `>= 1`）。

### 8.3 运行时强制

三处强制点：

1. **resume 时**（`index.js:626`）：`roundsStarted >= maxGoalRounds` 则拒绝，提示"increase maxGoalRounds before resuming"。
2. **fold 校验**（`fold.js:199, 299`）：resume 转移和 round 推进都校验 `round <= maxGoalRounds`。
3. **driver 自动 block**（`goal-round-driver/index.js:125-130`）：`roundsStarted >= maxGoalRounds` 时自动 block，code=`round-limit`。

> 🔑 **edit 可调整 maxGoalRounds**：即使 goal 耗尽轮次，人类可通过 `update_goal action=edit max_goal_rounds=N` 增加上限，然后 `resume` 继续。这是"续命"机制。注意 edit 不改变 phase，所以 blocked 的 goal 需先 edit 增加上限再 resume（resume 从 blocked→active）。

---

## 9. Goal 与 Agent Loop 的交互 — Arm/Disarm

### 9.1 Activation 模型

`GoalActivation = 'armed' | 'disarmed'`（`types.d.ts:57`）。

> 🔑 **activation 是进程级的、从不持久化的**。它只存在于 `GoalService.caches` 的 per-session cache 中。同一 session 在不同进程中的 activation 可能不同。持久层只记 phase。

### 9.2 Arm 的途径

只有以下操作会将 activation 设为 `armed`：
- `create()` → armed（`index.js:579`）
- `resume()` → armed（`index.js:627`）

### 9.3 Disarm 的途径

- `pause()` → disarmed
- `complete()` → disarmed
- `block()` → disarmed
- `clear()` → disarmed
- `disarm()` 方法（显式 disarm，不改 phase/revision）
- `agent/session-start` 事件（`index.js:519-521`）
- sync 重放非本进程发起的 goal/change（`index.js:718`）
- driver 异常时的 `disarm(state)`（`goal-round-driver/index.js:87-93`）

### 9.4 disarm() 方法

`index.js:552-558`

```javascript
disarm(agent) {
    this.assertLive(agent);
    const cache = this.cache(agent.session);
    this.sync(agent.session, cache);
    cache.activation = "disarmed";   // 仅改进程级状态
    return this.view(cache);
}
```

> 🔑 `disarm()` 是 driver 卸载前的清理操作（lifecycle owner 用），不改变持久 phase/revision。disarm 后，goal 仍 active（phase 不变），但 driver 不会续跑。人类 `resume` 可重新 arm。

### 9.5 driver 如何消费 activation

`drive()` 中的检查（`goal-round-driver/index.js:124`）：

```javascript
if (goal === void 0 || goal.phase !== "active" || goal.activation !== "armed") return;
```

只有 `phase='active' && activation='armed'` 才触发续跑。`readyToDrive()`（`index.js:79-81`）还要求 `agent.status === 'idle'`——即上一轮完全结束（turn end）后才开始下一轮。

### 9.6 goal/changed 事件驱动

`GoalService.commit()` 在 append 后 emit `goal/changed`（`index.js:793`）。driver 监听该事件（`goal-round-driver/index.js:234-238`）：

```javascript
ctx.on("goal/changed", ({ agent }) => {
    const state = stateFor(agent);
    state.needsCheckpoint = true;   // 标记需要持久化 checkpoint
    requestDrive(state);            // 请求驱动
});
```

> 🔑 **goal/changed 是续跑的主触发器之一**。create/resume 后立即触发 goal/changed → requestDrive → drive 在 agent idle 时注入第一轮。pause/block/complete 后也触发，但 drive 检查 phase/activation 后直接返回。

### 9.7 agent/status idle 触发

`goal-round-driver/index.js:216-233`：

```javascript
ctx.on("agent/status", ({ agent, status }) => {
    const state = stateFor(agent);
    if (status === "idle") {
        state.competingQueued = false;
        const attempt = state.attempt;
        const goal = currentGoal(state);
        // 若有 cancelled attempt 且 goal 仍 active+armed → pause（避免无限重试）
        if ((attempt?.phase === "queued" || attempt?.phase === "claimed" || attempt?.cancelled)
            && goal?.phase === "active" && goal.activation === "armed") {
            state.attempt = void 0;
            try { ctx.goals.pause(agent, goalRef(goal)); }
            catch (error) { ...; disarm(state); }
        }
        requestDrive(state);   // 请求下一轮
    }
});
```

> 🔑 **cancelled attempt 的保护**：若上一轮被取消（aborted/discarded）而 goal 仍 active+armed，driver 不会无限重试，而是主动 `pause` goal——让人类介入决定是否 resume。这防止了"取消-重试-再取消"的死循环。

### 9.8 初始化时的 disarm

`goal-round-driver/index.js:340`：

```javascript
for (const agent of ctx.agents.list()) disarm(stateFor(agent));
```

driver 启动时对所有现存 agent disarm。结合 `agent/session-start` 的 disarm，确保：**只有显式 create/resume 才能建立 armed 状态**，进程重启不会意外续跑。

### 9.9 拆卸时的清理

`goal-round-driver/index.js:341-359`：

```javascript
yield async () => {
    const waits = [];
    for (const state of states.values()) {
        state.stopping = true;
        disarm(state);
        const attempt = state.attempt;
        if (attempt !== void 0) {
            attempt.stale = true;
            if (state.agent.status === "running") {
                state.agent.cancel({ kind: "parent" });   // 取消运行中的 agent
                waits.push(state.agent.whenIdle());
            }
        }
        if (state.run !== void 0) waits.push(state.run);
    }
    await Promise.allSettled(waits);
    states.clear();
};
```

> 🔑 driver 卸载时：disarm 所有 goal、标记 attempt stale、取消运行中的 agent turn、等待所有任务结束。这保证了 goal 不会在 driver 卸载后继续自动续跑（phase 仍可能 active，但 activation disarmed，需人类 resume）。

---

## 10. 完整时序图：一个 Goal 的生命周期

```
人类: "帮我完成一个大任务"
  │
  ├─► 模型调用 create_goal(objective, max_goal_rounds)
  │     │
  │     ├─ requireDirectHuman ✓
  │     ├─ ctx.goals.create() → goal/change(create) 事件, activation=armed
  │     ├─ emit goal/changed
  │     └─ 返回 GoalView
  │
  ├─► goal-round-driver 收到 goal/changed
  │     ├─ needsCheckpoint=true, requestDrive
  │     └─ agent idle 后 drive():
  │          ├─ roundsStarted(0) < maxGoalRounds ✓
  │          ├─ round=1, renderGoalRoundPrompt
  │          ├─ agent.followup(message)  ← 排队第1轮
  │          └─ attempt.phase=queued
  │
  ├─► agent claim 消息 → attempt.phase=claimed
  │     ├─ pre-step validReservation ✓
  │     ├─ turn 开始, user/message(goal round=1) admitted
  │     ├─ roundsStarted 推进到 1 (fold)
  │     └─ 模型工作... 调用工具... 
  │
  ├─► turn end → agent idle
  │     ├─ agent/status(idle) → requestDrive
  │     └─ drive():
  │          ├─ attempt 仍存在 → 清空, needsCheckpoint, requestDrive
  │          ├─ checkpoint: sessions.flush()
  │          └─ drive(): round=2, followup ← 排队第2轮
  │
  ├─► ... 重复直到目标完成 ...
  │
  ├─► 模型判断完成 → update_goal(action=complete)
  │     ├─ completionAuthority: goal-round ✓ (当前轮)
  │     ├─ ctx.goals.complete() → goal/change(complete), activation=disarmed
  │     ├─ deferContext(renderWrapupContext) ← 注入收尾指令
  │     └─ 模型写总结消息, turn end
  │
  └─► driver 收到 goal/changed(complete) → drive 检查 phase≠active → 返回
        （goal 终止）
```

**异常路径（block）**：
```
第4轮: 模型遇到阻塞 → update_goal(action=blocked, blocked_reason="...")
  ├─ authority=goal-round, roundsStarted(4) >= blockedAfterConsecutiveRounds(3) ✓
  ├─ ctx.goals.block() → phase=blocked, activation=disarmed
  └─ 人类介入: /goal resume 或 update_goal(resume) → 重新 armed
```

---

## 11. 错误码汇总

`dsh-goal/lib/types/domain.d.ts:75`

```typescript
export type GoalErrorCode =
    | 'GOAL_AGENT_NOT_LIVE'        // agent 不是注册表的 live 实例
    | 'GOAL_NOT_FOUND'             // 无当前 goal
    | 'GOAL_ALREADY_EXISTS'        // 已有非 complete 的 goal
    | 'GOAL_STALE_REVISION'        // CAS ref 不匹配当前 revision
    | 'GOAL_INVALID_OBJECTIVE'     // objective 空或非字符串
    | 'GOAL_INVALID_MAX_ROUNDS'    // maxGoalRounds 非正安全整数
    | 'GOAL_INVALID_BLOCK_REASON'  // block reason 格式非法
    | 'GOAL_INVALID_EDIT'          // edit 无可变字段
    | 'GOAL_INVALID_TRANSITION';   // 非法 phase 转移
```

工具层额外错误码（`HarnessError`）：
- `GOAL_TOOL_AUTHORITY_REQUIRED` / `GOAL_TOOL_DRIVER_REQUIRED` / `GOAL_TOOL_AGENT_REQUIRED`
- `GOAL_TOOL_INVALID_UPDATE`（参数与 action 不匹配）
- `GOAL_TOOL_BLOCK_THRESHOLD`（未达最小轮数）

---

## 12. 总结

DSH Goal 系统是一个精心设计的长目标系统，核心特点：

1. **事件溯源 + CAS**：状态由 session 事件日志决定，变更需乐观锁（ref），保证同一 session 跨进程一致。
2. **持久/进程分离**：phase 可重放，activation 进程级——session resume 后需人类显式 resume 才能 rearm 自动续跑。
3. **三层校验**：写侧（GoalService）、strict fold（invariant companion）、投影 fold（容错 last-wins）。
4. **race-fenced driver**：pre-step 双重校验、coalesce 序列化、cancelled 保护、checkpoint 节奏。
5. **权限分层**：edit/pause/resume 仅人类；complete/blocked 允许模型在自动轮次中报告，但 blocked 需最小轮数。
6. **确定性 prompt**：continuation prompt 是纯函数，invariant 验证未被篡改。
7. **四包职责分离**：领域核心（dsh-goal）、驱动器（round-driver）、模型工具（tool-goal）、人类命令（command-goal），通过 Cordis 事件和 Typert 远程边界解耦。

> 🔑 **最重要的设计决策**：将"目标是否继续"的权威性（activation）与"目标状态"（phase）分离。这使得进程崩溃/重启不会丢失目标状态，但也不会意外触发无人监督的自动续跑——必须有人类明确的 resume 信号才重新 arm。这是 DSH 在"自主性"与"安全性"之间的平衡点。
