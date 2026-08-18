# DSH Session 会话系统实现深度分析

> **代码位置**: `/Users/jjq/.npm/_npx/1e7f6d9597241db0/node_modules/@deepseek-ai/dsh-session/`
> **版本**: `@deepseek-ai/dsh-session@0.1.0-rc.6`
> **持久化后端**: `@deepseek-ai/dsh-session-persistence-jsonl` + `@deepseek-ai/dsh-session-persistence`

本文档基于对 `lib/index.js`（打包产物，1886 行）、`lib/types/*.js`（源文件）、`lib/types/*.d.ts`（类型声明）、`lib/invariant.js`（不变量伴随插件）以及 `dsh-session-persistence-jsonl` 的逐行精读，给出可直接作为技术文档使用的结构化分析。

行号引用说明：`index.js:N` 指**打包产物** `lib/index.js` 的第 N 行；`types/index.js:N` 指**源文件** `lib/types/index.js` 的第 N 行（两者逻辑等价，源文件更易读）；`.d.ts:N` 指类型声明文件第 N 行。

---

## 目录

1. [架构总览](#1-架构总览)
2. [事件溯源（Event-Sourcing）机制](#2-事件溯源event-sourcing机制)
3. [Projection 投影机制 — Surface 与 deriveMessages](#3-projection-投影机制--surface-与-derivemessages)
4. [JSONL 持久化 — 如何存储到磁盘](#4-jsonl-持久化--如何存储到磁盘)
5. [Session 的事件类型（Event Types）](#5-session-的事件类型event-types)
6. [Header 管理 — requestHeader 与 foldRequestHeader](#6-header-管理--requestheader-与-foldrequestheader)
7. [Session 的创建、恢复（resume）、fork](#7-session-的创建恢复resumefork)
8. [SessionStore 的实现](#8-sessionstore-的实现)
9. [不变量系统（Invariant Companion）](#9-不变量系统invariant-companion)
10. [崩溃恢复（Crash Repair）](#10-崩溃恢复crash-repair)

---

## 1. 架构总览

DSH Session 系统采用**事件溯源（Event Sourcing）+ CQRS 风格投影**的架构。核心设计原则：

```
┌─────────────────────────────────────────────────────────────────┐
│                      事件溯源架构分层                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SessionStore (ctx.sessions)          ← 内存中的 Session 注册表   │
│   ├── create / prepare / enter / announce                        │
│   ├── fork                        ← 从 live session 前缀派生      │
│   ├── flush                       ← 持久化检查点入口              │
│   └── get / list                  ← 查询                          │
│                                                                 │
│  Session (plain class)                ← 事件溯源聚合根             │
│   ├── log: SessionEvent[]         ← append-only 事件日志（真相源）│
│   ├── surfaceManager              ← 增量 Surface 投影管理器       │
│   ├── append(type, data, opts)    ← 追加事件 + 同步通知           │
│   ├── deriveMessages()            ← 投影出 LLM 消息历史（缓存）   │
│   ├── requestHeader()             ← 折叠出当前请求头（缓存）      │
│   └── events / seq / id / header  ← 只读视图                     │
│                                                                 │
│  持久化插件 (dsh-session-persistence)  ← write-behind + flush     │
│   ├── 订阅 session/event          ← 异步缓冲写入                  │
│   ├── 订阅 session/flush          ← await 持久化检查点            │
│   └── dsh-session-persistence-jsonl ← JSONL 磁盘格式后端          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> 🔑 **核心设计决策**：`Session` 是一个**普通类**（非 Cordis Service），事件日志是唯一的真相源（source of truth），LLM 消息历史是**派生**出来的投影。持久化被有意排除在核心包之外——插件通过订阅 `session/event` 事件做 write-behind，在 `session/flush` 时排空缓冲。

关键代码声明（`types/index.js:1-7`）：

```js
/**
 * Event-sourced session service: append-only session log, in-memory store, and
 * the derived LLM message history. Persistence is a plugin concern (subscribe
 * to `session/event`, drain on `session/flush`).
 * @module @deepseek-ai/dsh-session
 */
```

---

## 2. 事件溯源（Event-Sourcing）机制

### 2.1 事件日志 — append-only 真相源

`Session` 类的核心是一个**只追加（append-only）**的事件数组 `log`（`index.js:1304` / `types/index.js:321`）：

```js
var Session = class Session {
    log = [];                                    // 私有 append-only 日志
    surfaceManager = new SurfaceManager(this.log); // 增量投影管理器，共享 log 引用
    ...
}
```

> 🔑 **关键契约**：`seq = log.length`。每个事件的序列号严格等于它在日志中的位置（从 0 开始的连续整数）。这是整个系统依赖的"连续性契约"（contiguity contract），见 `index.js:1401-1403`：

```js
/** The next event's sequence number — always the log length
 * (the `seq = log.length` contiguity contract). */
get seq() { return this.log.length; }
```

### 2.2 事件如何记录 — `append()` 方法

`append()` 是事件溯源的唯一写入入口（`index.js:1440-1480` / `types/index.js:484-533`）。其执行流程：

```
append(type, data, ...opts)
  │
  ├─ 1. 构造 surfaceMetadata（surfaceOp + sourceEventSeqs）
  ├─ 2. snapshotJsonValue(data)  ← 无损 JSON 验证 + 深拷贝（一次性遍历）
  │     └─ 失败则抛 "carries non-JSON-serializable data"
  ├─ 3. assertSupportedRequestHeader() ← 拒绝遗留格式
  ├─ 4. snapshotJsonValue(surfaceMetadata) ← 同样验证
  ├─ 5. 重入检测：entry?.appending → 抛 "cannot reenter"
  ├─ 6. 构造事件对象：
  │     { type, seq: log.length, time: Date.now(), data: dataSnapshot, ...surfaceMetadata }
  │     然后 deepFreeze() ← 深度冻结
  ├─ 7. surfaceManager.validateNext(event) ← 提交前验证 Surface 转换
  ├─ 8. 标记 appending = true
  ├─ 9. collectSessionCallbacks() ← 解析监听器快照（在 push 之前！）
  ├─ 10. this.log.push(event) ← 提交！
  ├─ 11. eventsSnapshot = undefined ← 使缓存失效
  ├─ 12. invokeContainedSessionObservers() ← 同步通知观察者（失败隔离）
  └─ 13. finally: appending = false; 若 detachRequested 则延迟 detach
```

> 🔑 **关键设计 1 — 快照优先**：监听器快照在 `log.push` **之前**解析（步骤 9 在步骤 10 前），但回调在 push **之后**执行。这意味着一个已提交的事件一定会被通知到已解析的监听器，观察者失败不会使已提交的 append 失败（per-listener containment）。

> 🔑 **关键设计 2 — 状态性 getter 防护**：`snapshotJsonValue` 用**一次性遍历**读取+验证+拷贝每个嵌套值，确保一个有状态的 getter 无法"验证时给一个值、存储时给另一个值"（`index.js:1432-1435` 注释）。

核心代码（`index.js:1440-1480`）：

```js
append(type, data, ...opts) {
    const surfaceOpts = opts[0];
    const surfaceMetadata = {
        ...surfaceOpts?.sourceEventSeqs === void 0 ? {} : { sourceEventSeqs: surfaceOpts.sourceEventSeqs },
        ...surfaceOpts?.surfaceOp === void 0 ? {} : { surfaceOp: surfaceOpts.surfaceOp }
    };
    const dataSnapshot = snapshotJsonValue(data);          // 无损JSON验证+detach
    if (dataSnapshot === void 0) throw new Error(`session event "${type}" carries non-JSON-serializable data`);
    ...
    const entry = attachments.get(this);
    if (entry?.appending) throw new Error("session append cannot reenter while another append is being published");
    const event = deepFreeze({
        type, seq: this.log.length, time: Date.now(),
        data: dataSnapshot, ...surfaceMetadataSnapshot
    });
    this.surfaceManager.validateNext(event);   // 提交前验证
    if (entry !== void 0) entry.appending = true;
    try {
        let callbacks;
        if (entry !== void 0) callbacks = collectSessionCallbacks(entry.emitCtx, [
            entry.carrier, "session/event", this, event
        ]);
        this.log.push(event);                  // 提交
        this.eventsSnapshot = void 0;          // 失效缓存
        if (callbacks !== void 0) invokeContainedSessionObservers(...);  // 通知
        return event;
    } finally {
        if (entry !== void 0) { entry.appending = false; ... }
    }
}
```

### 2.3 事件如何重放（Replay）

事件重放发生在三个场景：

**场景 A — 构造时种子重放**（`index.js:1370-1388` / `types/index.js:387-432`）：

构造函数接收 `seed`（一组已有事件），逐个验证后通过 `surfaceManager.validateNext()` 接入 Surface，然后 `log.push()`：

```js
constructor(id, seed, header, mode = "snapshot") {
    ...
    if (seed !== void 0) for (const [index, source] of seed.entries()) {
        const snapshot = mode === "restore" ? source : snapshotJsonValue(source);
        assertSessionEventEnvelope(snapshot, index);
        assertSupportedRequestHeader(snapshot.type, snapshot.data, ...);
        if (snapshot.seq !== index) throw new Error(`seed must be contiguous from 0`);
        try { this.surfaceManager.validateNext(snapshot); }
        catch (error) { throw new Error(`invalid seed event at index ${index}`); }
        this.log.push(mode === "restore" ? freezeRestoredObject(snapshot) : deepFreeze(snapshot));
    }
    this.firstLiveSeq = this.log.length;
    ...
}
```

> 🔑 **种子验证 = append 验证**：种子事件通过**与 `append` 完全相同的不变量**验证，确保 replay/fork 无法构造一个持久化后端无法存储的 live log。每个事件的 `data` 必须 JSON 可序列化，`seq` 必须从 0 连续。

**场景 B — Surface 折叠重放**（`surface.js` 中的 `foldSurface`，见第 3 节）。

**场景 C — 持久化恢复重放**（见第 7 节 `fromRestore`）。

### 2.4 `events` 快照与不可变性

`events` getter 返回缓存的冻结快照（`index.js:1397-1400`）：

```js
get events() {
    this.eventsSnapshot ??= Object.freeze([...this.log]);
    return this.eventsSnapshot;
}
```

快照在下次 append 时失效（`eventsSnapshot = void 0`）。返回的数组是 `Object.freeze` 的浅拷贝，而事件及其嵌套 data 在接受时已 `deepFreeze`，所以"既不能用 cast 也不能用普通 JavaScript 重写持久历史"。

### 2.5 事件发布机制 — 三类 Cordis 事件

SessionStore 通过 Cordis 事件总线发布三类事件（`.d.ts:32-76`）：

| 事件 | 模式 | 触发时机 | 语义 |
|------|------|----------|------|
| `session/created` | emit | `announce()` 时 | 创建公告；同步 throw 可否决并回滚 |
| `session/disposed` | emit | detach 时 | 销毁通知；listener 失败被日志隔离 |
| `session/event` | emit | `append()` 后 | post-commit fire-and-forget 追加通知 |
| `session/flush` | parallel | `flush()` 时 | await 持久化检查点；所有 listener 并行运行 |

> 🔑 **Scope 过滤**：所有事件都通过 `@deepseek-ai/dsh-scope` 做 agent-scoped 过滤——agent-scoped listener 只收到通过该 agent context 进入的 session 的事件。carrier 在 `enter()` 时由 `scopeTarget(session, scopeOf(this.ctx))` 捕获（`index.js:1691`）。

---

## 3. Projection 投影机制 — Surface 与 deriveMessages

Surface 是事件日志之上的**有序投影层**：一个产生 LLM 消息的事件的有序视图。原始 append-only 日志仍然是真相源（`surface.js:208-217`）。

> 🔑 **Surface 的核心意义**：不是所有事件都产生 LLM 消息（chunk、turn 边界、usage 等不产生）。Surface 只保留"产生消息的事件"的有序序列，并且支持 **replace 操作**（压缩/修剪时替换一段历史），而原始日志保持 append-only。

### 3.1 Surface 事件类型

只有三种事件类型有资格进入 Surface（`surface.js:219-223`）：

```js
const SURFACE_EVENT_TYPES = new Set(["user/message", "assistant/message", "tool/result"]);
```

### 3.2 SurfaceOp — 事件如何进入 Surface

每个 surface-eligible 事件**必须**声明 `surfaceOp`（`types.d.ts:388-392`）：

```ts
export type SurfaceOp = 'append' | {
    op: 'replace';
    start: number;  // 起始 seq（inclusive）
    end: number;    // 结束 seq（inclusive）
};
```

- **`'append'`**：追加到尾部——用户/助手/工具消息的正常路径
- **`{ op: 'replace', start, end }`**：替换从 `start` 到 `end`（都 inclusive）的 surface 节点。两者都必须是当前 surface 中存在的节点。`start === end` 替换单个节点。用于压缩（compaction）。

### 3.3 `deriveEventMessage` — 单事件投影规则

这是**每节点投影规则**的核心（`surface.js:278-287`）：

```js
function deriveEventMessage(event) {
    switch (event.type) {
        case "user/message": return event.data;
        case "assistant/message":
            if (event.data.message.content.length === 0) return null;  // 空内容不产生消息
            return event.data.message;
        case "tool/result": return event.data.message;
        default: return null;  // 非 surface 事件不产生消息
    }
}
```

> 🔑 **空内容 assistant/message**：`content.length === 0` 的 assistant/message 存在仅为了承载 `usage`（max-tokens 步），不进入派生历史。

### 3.4 SurfaceManager — 增量投影

`SurfaceManager`（`surface.js:457-511`）是增量有序 surface 视图和 append 边界验证器。它维护：

```js
var SurfaceManager = class {
    log;                    // 共享的日志引用
    baseSeq;                // 窗口第一个事件的绝对 seq
    _state = createFoldState();  // { nodes: [], replaceGeneration: 0 }
    _lastProcessedSeq;      // 最后处理的绝对 seq
    _pendingPlan;           // validateNext 验证过但尚未入 log 的候选
    ...
}
```

**`validateNext(event)`**（`surface.js:479-487`）——验证下一个候选但不修改已提交的 surface：

```js
validateNext(event) {
    if (this._lastProcessedSeq < this.baseSeq + this.log.length - 1) this._processDelta();
    const expectedSeq = this.baseSeq + this.log.length;
    this._pendingPlan = {
        event, expectedSeq,
        plan: planSurfaceEvent(this._state, event, expectedSeq, this.log, this.baseSeq)
    };
}
```

**`_processDelta()`**（`surface.js:499-510`）——增量折叠自上次访问以来追加的事件。它检查是否有 pending plan 匹配当前事件，匹配则复用已验证的 plan（避免重复验证）：

```js
_processDelta() {
    const tailSeq = this.baseSeq + this.log.length - 1;
    for (let seq = this._lastProcessedSeq + 1; seq <= tailSeq; seq++) {
        const event = this.log[index];
        const pending = this._pendingPlan;
        if (pending?.event === event && pending.expectedSeq === seq)
            applySurfacePlan(this._state, pending.plan);  // 复用已验证 plan
        else
            applySurfaceEvent(this._state, event, seq, this.log, this.baseSeq);
        ...
    }
}
```

### 3.5 Surface 折叠 — `foldSurface`（纯重放）

`foldSurface`（`surface.js:444-455`）是**离线纯重放**路径，将完整日志折叠为当前 surface 序列和替换历史：

```js
function foldSurface(events) {
    const state = createFoldState();
    const replacements = [];
    for (const [index, event] of events.entries()) {
        const replacement = applySurfaceEvent(state, event, index, events, 0);
        if (replacement !== void 0) replacements.push(replacement);
    }
    return { nodes: [...state.nodes], replacements };
}
```

### 3.6 Replace 操作的验证

Replace 操作有多重验证（`surface.js:396-418`）：

1. **`assertProvenance`**（`surface.js:320-337`）：`sourceEventSeqs` 必须包含**每个**被遮蔽的 surface 节点，且不能引用 >= 当前 seq 的事件，不能有重复
2. **`assertToolResultRewrite`**（`surface.js:369-395`）：`tool/result` 替换只能改 **content**，不能改其他字段——通过将 content 置 null 后做深度相等比较

```js
function assertToolResultRewrite(event, shadowedSeqs, events, baseSeq) {
    if (event.type !== "tool/result") return;
    if (shadowedSeqs.length !== 1) throw new Error("tool/result surface replacement must rewrite exactly one current node");
    // 比较 content 置 null 后的结构
    if (!isDeepEqualJson(originalRest, replacementRest)) throw new Error("tool/result surface replacement may change only content");
}
```

### 3.7 `deriveMessages()` — 派生 LLM 消息历史（缓存）

`Session.deriveMessages()`（`index.js:1539-1554`）将 Surface 折叠为 LLM 消息数组，带增量缓存：

```js
deriveMessages() {
    const surface = this.surface;
    const nodes = surface.nodes;
    const generation = surface.replaceGeneration;
    if (generation !== this.derivedGeneration) {  // replace 发生 → 重建
        this.derived = [];
        this.derivedNodes = 0;
        this.derivedGeneration = generation;
    }
    for (const seq of nodes.slice(this.derivedNodes)) {  // 只投影新节点
        const msg = this.deriveEventMessage(this.log[seq]);
        if (msg) this.derived.push(msg);
    }
    this.derivedNodes = nodes.length;
    return [...this.derived];  // 每次返回新数组（但 Message 对象共享且冻结）
}
```

> 🔑 **缓存策略**：
> - 每个 surface 节点**只投影一次**（首次见到时）
> - 一次调用成本 = O(新节点数)
> - surface rewrite（`replace`）触发 `replaceGeneration` 变化 → 完全重建
> - 返回的是新数组，但内部 `Message` 对象**共享**且**深度冻结**（复用已冻结的持久事件 data，无需二次深拷贝）

> 🔑 **append-origin vs replacement**：人类 transcript 应该投影 **append-origin** 事件（`isAppendSurfaceEvent`），而非 `session.surface`——因为已落地的 replacement 会遮蔽用户已看过的对话。模型侧消费者读 `session.surface`。

---

## 4. JSONL 持久化 — 如何存储到磁盘

> 🔑 **架构分层**：`dsh-session` 核心包**不实现持久化**。持久化分为两层：
> - `dsh-session-persistence`：协调器（`PersistenceCoordinator`），负责 write-behind 缓冲、flush、修复
> - `dsh-session-persistence-jsonl`：JSONL 磁盘格式后端

### 4.1 磁盘文件布局

```
<root>/
  <projectKey>/              ← projectKey(cwd)，如 --Users-jjq-Desktop-deepseek--
    <encodedSessionId>/      ← encodeSegment(sessionId)，路径安全编码
      session.jsonl          ← 或 session.jsonl.zstd（Zstandard 压缩）
```

路径构造函数（`dsh-session-persistence-jsonl/lib/index.js:133-158`）：

```js
function projectDir(root, cwd) {
    if (cwd === void 0) return join(root, "_no-cwd");
    return join(root, projectKey(cwd));
}
function sessionDir(root, cwd, id) {
    return join(projectDir(root, cwd), encodeSegment(id));
}
function logPath(root, cwd, id, compression) {
    return join(sessionDir(root, cwd, id), `session${logSuffix(compression)}`);
}
```

> 🔑 **路径安全**：`SessionId` 是未验证的 branded string，必须通过 `encodeSegment` 编码后才能用于路径——防止 `../` 遍历、NUL、分隔符注入（`index.js:84-96`）。

### 4.2 JSONL 文件格式

文件由**首行 header** + **后续事件行**组成：

**首行 — session header**（`index.js:36-49`）：

```json
{"type":"session","version":0,"id":"session-1","createdAt":1715000000000,"cwd":"/Users/jjq/Desktop","delegationDepth":0}
```

注意 `type: "session"` 是 bare tag（无斜杠），与事件 taxonomy 区分。`delegationDepth` 始终存在（默认 0）。

**后续行 — 事件**：每行一个 `JSON.stringify(event)`，或一个打包的 chunk row。

### 4.3 Chunk-Row 存储压缩

> 🔑 **问题**：Provider 流式输出 token 级 delta，一个会话可能存储数百个近乎相同的事件行，其 JSON 信封远大于载荷（实测约 56× 膨胀）。

`chunk-rows.js`（`index.js:727-1035`）将连续的同类、同 block 的 delta chunk 事件打包成**一个存储行**：

- `text-chunks`：打包 `text-delta` 运行
- `reasoning-chunks`：打包 `reasoning-delta` 运行
- `tool-call-chunks`：打包 `tool-call-delta` 运行

打包条件（`index.js:752`）：`MIN_RUN = 3`（少于 3 个不打包，因为信封开销与事件行相当）。

打包格式（`index.js:828-866`）：

```json
{
  "type": "text-chunks",
  "seq0": 10, "time0": 1715000000000,
  "data": { "turn": 1, "step": 1, "index": 0, "dt": [0, 1, 0], "texts": ["Hello", " ", "world"] }
}
```

- `seq0`/`time0`：第一个成员的 seq/time 锚点
- `dt`：成员间的时间间隔数组（长度 = 成员数 - 1），可为负（时钟回拨）
- 成员 k 重建为 seq = `seq0 + k`，time = `time0` + 前 k 个 dt 之和

> 🔑 **存储行 ≠ 事件**：存储行使用 bare type tag（无斜杠），**不进入** `Session.events`，没有 `SessionEventMap` 条目。编码器白名单精确形状——无法识别的就原样存储（只损失压缩，不损失数据）。解码器先验证再展开，畸形行抛错而非静默丢弃。

打包/解包 API（`index.js:877-1034`）：

```js
function packChunkRuns(events) { ... }     // 事件批 → 存储记录
function decodeStorageRecord(value) {       // 一行 JSONL → 事件数组
    if (tag !== "text-chunks" && tag !== "reasoning-chunks" && tag !== "tool-call-chunks")
        return [value];  // 非打包行，原样返回
    return expandRow(validateRow(value, tag));  // 验证 + 展开
}
```

### 4.4 写入路径 — write-behind

`dsh-session-persistence/lib/index.js:1132-1163` 注册写入路径：

```js
installWritePath() {
    ctx.on("session/created", (session) => { this.initFor(session); });
    ctx.on("session/event", (session, event) => {
        this.initFor(session).writes.enqueue(event);  // 异步缓冲
    });
    ctx.on("session/flush", (session) => this.flush(session));  // 检查点
    ctx.on("session/disposed", (session) => { this.retire(session); });
    ...
}
```

> 🔑 **热路径不阻塞 I/O**：`session/event` 只将事件入队（`writes.enqueue`），持久化插件异步缓冲。`append` 返回时事件已提交到内存 log，但可能尚未落盘。

### 4.5 Flush — 持久化检查点

`SessionStore.flush()`（`index.js:1787-1804`）是**唯一**的 flush 入口：

```js
async flush(session) {
    const { carrier } = this.liveEntryFor(session);
    const callbacks = collectSessionCallbacks(this.ctx, [carrier, "session/flush", session]);
    const results = await Promise.allSettled(callbacks.map(cb => {
        try { return cb(session); }
        catch (error) { return Promise.reject(error); }
    }));
    const failure = results.find(r => r.status === "rejected");
    if (failure !== void 0) throw failure.reason;
    return callbacks.length > 0;  // 是否有 listener 参与
}
```

> 🔑 **并行不否决**：`session/flush` 是 `parallel` 模式——所有 listener 都启动，调用等待全部 settle 后才报告失败（无 waterfall 否决）。返回是否有 listener 参与。

### 4.6 追加写入与崩溃安全

JSONL 后端的 `appendLines`（`jsonl/index.js:1200-1227`）：

```js
async appendLines(meta, events) {
    const content = await this.encodeEventBatch(events);  // JSON.stringify + 可能 zstd
    const path = logPath(this.root, meta.cwd, meta.id, this.compression);
    const handle = await open(path, "a");
    const { size: before } = await handle.stat();
    try {
        await handle.writeFile(content);
        await handle.sync();  // fsync
    } catch (error) {
        await this.rollbackAppend(path, before);  // 回滚到之前大小
        throw error;
    }
}
```

> 🔑 **崩溃安全**：写入失败时 `rollbackAppend` 将文件截断回之前大小（`truncate` + `sync`），因为留下部分字节会创建重复序列号。首次创建用 `writeSyncedTempFile` + `link`（原子发布）+ 目录 fsync。

### 4.7 读取路径 — SessionLogScanner

`SessionLogScanner`（`jsonl/index.js:205-301`）增量扫描 JSONL 事件记录：

- 首行单独解析为 header（`parseHeaderRecord`）
- 后续行逐行 `JSON.parse` + `decodeStorageRecord` 展开
- **撕裂尾部容忍**：最后一条无换行符的记录被视为 torn tail，在 `finish()` 时忽略
- **seq 连续性检查**：`event.seq !== this.events.length` 则标记 issue
- 返回 `{ meta, events, committedBytes }`——`committedBytes` 是安全截断偏移

---

## 5. Session 的事件类型（Event Types）

### 5.1 事件类型全集 — `SessionEventMap`

`SessionEventMap`（`types.d.ts:223-354`）定义了核心事件类型。以下是完整分类：

#### 对话核心事件（Surface-eligible，产生 LLM 消息）

| 事件类型 | data 结构 | Surface | 说明 |
|----------|-----------|---------|------|
| `user/message` | `UserMessage` | ✅ | 用户消息（直接 prompt / synthetic inject / goal round） |
| `assistant/message` | `{ turn, step, message: AssistantMessage, usage? }` | ✅ | 组装好的助手消息；携带 step 的 usage |
| `tool/result` | `{ turn, step, message: ToolResultMessage, error?, meta? }` | ✅ | 工具调用结果；`meta` 是工具私有展示载荷 |

#### 流式与边界事件（log-only，不进 Surface）

| 事件类型 | data 结构 | 说明 |
|----------|-----------|------|
| `turn/start` | `{ turn: number }` | 开启 turn |
| `turn/end` | `{ turn: number, reason: TurnEndReason }` | 关闭 turn，携带结束原因 |
| `step/start` | `{ turn, step }` | 开启 step（一次模型调用 + 工具执行） |
| `step/end` | `{ turn, step }` | 关闭 step |
| `assistant/chunk` | `{ turn, step, chunk: StreamChunk }` | 原始流式 chunk（token 级重放保真） |
| `tool/call` | `{ turn, step, callId, name, arguments }` | 模型请求的工具调用 |

#### 请求元数据事件

| 事件类型 | data 结构 | 说明 |
|----------|-----------|------|
| `request/header` | `{ header: EpochHeader, reason: RequestHeaderReason }` | 下一请求的完整 header 快照 |
| `request/context` | `RequestContext` | 路由元数据（provider/model/contextWindow） |
| `todo/write` | `{ todos: TodoItem[] }` | 整列表快照（last-write-wins） |
| `session/end-seed` | `Record<string, never>` | 构造种子结束标记（payload 为空） |

#### TurnEndReason — turn 结束原因（merge-extensible）

`TurnEndReasonMap`（`types.d.ts:135-167`）：

| kind | 说明 |
|------|------|
| `completed` | 正常完成 |
| `aborted` | 被取消（携带 `reason: AgentCancelCause`） |
| `blocked` | 阻塞 |
| `error` | 失败（携带 `error: LlmFailure`） |
| `max-tokens` | 达到输出 token 上限 |
| `interrupted` | 崩溃恢复时后端合成的关闭标记（loop 从不发出） |

### 5.2 已知事件类型目录 — `KNOWN_SESSION_EVENT_TYPES`

`known-event-types.js`（`index.js:1054-1099`）是**代码生成**的完整事件词汇表（由 `scripts/gen-persistence-catalog.ts` 生成）。包含 46 个事件类型：

```js
const KNOWN_SESSION_EVENT_TYPES = new Set([
    "agent-preset/selected", "agent/inbox/spliced", "approval/asked",
    "approval/decided", "approval/policy", "assistant/chunk", "assistant/message",
    "command/done", "command/run", "compaction/end", "compaction/prune",
    "compaction/start", "compaction/summary", "feedback/record", "goal/change",
    "hook/invoked", "hook/result", "llm/retry", "llm/retry-started",
    "permission/preset", "plan/mode", "request/context", "request/header",
    "sandbox/mode", "schedule/change", "session/end-seed", "session/title",
    "session/title-llm-request", "step/end", "step/start", "subagent/descriptor",
    "todo/write", "tool-workflow/agent-end", "tool-workflow/agent-start",
    "tool-workflow/run-end", "tool-workflow/run-start", "tool/call",
    "tool/code-dispatch", "tool/code-dispatch-start", "tool/result",
    "turn/end", "turn/start", "user/message", "web/deepseek-search-llm-request"
]);
```

> 🔑 **持久化读取拒绝未知类型**：持久化读取路径拒绝解释包含此集合外类型的日志，**除非**事件携带 `ignorable: true` 标记。这样的日志可能是更新的 harness 写的，静默跳过一个必需事件会重建错误的 session。

### 5.3 `ignorable` 标记 — 词汇增长的版本策略

`SessionEvent.ignorable`（`types.d.ts:428-438`）：

```ts
/**
 * Marks an event a reader may safely skip when it does not recognize `type`.
 * Absent means required: a reader meeting an unrecognized type without this
 * marker MUST refuse to reconstruct the session instead of silently dropping
 * the event.
 */
ignorable?: true;
```

> 🔑 **版本策略**：添加普通事件类型**不 bump 版本号**——`ignorable` 标记覆盖词汇增长。只有结构性变更（header 形状、事件信封、核心语义、Surface 机制）才 bump `SESSION_FORMAT_VERSION`。

### 5.4 事件信封结构

`SessionEvent`（`types.d.ts:420-452`）是一个基于 `type` 的判别联合（discriminated union）：

```ts
export type SessionEvent<T extends SessionEventType = SessionEventType> = {
    [K in SessionEventType]: {
        type: K;
        seq: number;       // 单调序列号
        time: number;      // Unix epoch ms
        data: SessionEventMap[K];
        ignorable?: true;
    } & (K extends SurfaceEventType ? {
        sourceEventSeqs?: number[];  // 引用的源事件 seq
        surfaceOp?: SurfaceOp;        // 如何进入 surface
    } : object);  // 非 surface 事件无这些字段
}[T];
```

> 🔑 **编译时强制**：`surfaceOp` 和 `sourceEventSeqs` 只存在于 `SurfaceEventType` 变体上。编译器在 `Session.append()` 调用点拒绝非 surface 类型（如 `turn/start`、`assistant/chunk`）传入这些字段。

---

## 6. Header 管理 — requestHeader 与 foldRequestHeader

### 6.1 EpochHeader — 请求头结构

`EpochHeader`（`types.d.ts:191-200`）：

```ts
export interface EpochHeader {
    config: LlmCallConfig;                         // provider, model, reasoning effort, 采样参数
    adapterDefaults?: LlmCallConfigAdapterDefaults; // 适配器生效的字段标记
    system?: string;                                // 渲染后的系统提示
    tools?: ToolSchema[];                           // 组装的工具 schema
}
```

### 6.2 request/header 事件

`request/header` 事件记录下一请求的**完整 canonical 快照**（`types.d.ts:322-325`）：

```ts
'request/header': {
    header: EpochHeader;
    reason: RequestHeaderReason;  // 'initial' | 'resume' | 'change'
};
```

- `'initial'`：日志的第一个 header（新对话）
- `'resume'`：loop 实例在已有 header 事件的日志上的第一个请求（进程重启、fork 种子）
- `'change'`：后续请求使用了不同 header

> 🔑 **遗留格式拒绝**：`request/header-delta`（遗留增量格式）和 `reason: "fallback"` 被拒绝（`index.js:1274-1277`）。

### 6.3 canonicalHeader — 规范化

`canonicalHeader`（`index.js:529-537` / `request-header.js`）将 header 规范化——空 system prompt 和空 tool list 变成缺失字段：

```js
function canonicalHeader(header) {
    return {
        config: header.config,
        ...adapterDefaults?.reasoningEffort === true || adapterDefaults?.maxTokens === true
            ? { adapterDefaults } : {},
        ...header.system !== void 0 && header.system.length > 0 ? { system: header.system } : {},
        ...header.tools !== void 0 && header.tools.length > 0 ? { tools: header.tools } : {}
    };
}
```

### 6.4 headerEquals — 字段相等比较

`headerEquals`（`index.js:548-553`）比较两个 canonical header：

```js
function headerEquals(a, b) {
    if (!callConfigEquals(a.config, b.config)
        || a.adapterDefaults?.reasoningEffort !== b.adapterDefaults?.reasoningEffort
        || a.adapterDefaults?.maxTokens !== b.adapterDefaults?.maxTokens
        || a.system !== b.system) return false;
    const at = a.tools ?? [], bt = b.tools ?? [];
    return at.length === bt.length && at.every((tool, i) => sameSchema(tool, bt[i]));
}
```

> 🔑 **用途**：loop 用 `headerEquals` 判断下一个请求的 header 是否与当前不同——不同才追加 `request/header` 事件（避免记录未变化的 header）。

### 6.5 foldRequestHeader — 离线重建

`foldRequestHeader`（`index.js:563-567`）是**纯离线重建路径**——从日志的 header 事件折叠出最后一次快照后的 `EpochHeader`：

```js
function foldRequestHeader(events, from) {
    let state = from;
    for (const event of events)
        if (event.type === "request/header") state = canonicalHeader(event.data.header);
    return state;  // 最后一个 header 快照，或 undefined
}
```

### 6.6 Session.requestHeader() — 增量缓存

`Session.requestHeader()`（`index.js:1493-1498`）是 `foldRequestHeader` 的**增量维护形式**——每个 header 事件只折叠一次：

```js
requestHeader() {
    if (this.headerFoldSeq < this.log.length) {
        this.headerFold = deepFreeze(foldRequestHeader(
            this.log.slice(this.headerFoldSeq), this.headerFold
        ));
        this.headerFoldSeq = this.log.length;
    }
    return this.headerFold;
}
```

> 🔑 **O(new events)**：每次读取成本仅为新增事件数，因为 `headerFoldSeq` 记录了已消费的日志位置。fold 结果 `deepFreeze` 后通过引用暴露——消费者原地修改会导致后续比较失同步，所以 mutation 会抛错。

### 6.7 requestContext — 路由元数据折叠

类似的增量模式用于 `request/context` 事件（`index.js:1508-1514`）：

```js
requestContext() {
    if (this.contextFoldSeq < this.log.length) {
        for (const event of this.log.slice(this.contextFoldSeq))
            if (event.type === "request/context") this.contextFold = deepFreeze({ ...event.data });
        this.contextFoldSeq = this.log.length;
    }
    return this.contextFold;
}
```

### 6.8 SessionHeader — 存储元数据

`SessionHeader`（`types.d.ts:40-78`）是**存储元数据**，独立于对话事件日志：

```ts
export interface SessionHeader {
    readonly version: number;           // 格式版本（当前 = 0）
    readonly id: SessionId;             // session id
    readonly createdAt: number;         // 创建时间（Unix ms）
    readonly cwd?: string;              // 绝对工作目录
    readonly parentSession?: SessionId; // fork 来源
    readonly seedLength?: number;       // 继承的前缀长度
    readonly origin?: 'subagent';       // 子 agent 来源标记
    readonly delegationDepth?: number;  // 委托深度（递归预算）
    readonly agentPreset?: string;      // agent preset id
}
```

> 🔑 **不在事件日志中**：`SessionHeader` 是存储关注点，不是可重放的对话状态。验证函数 `validateSessionHeader`（`types/index.js:25-66`）强制：`version === 0`、`id` 匹配、`createdAt` 非负安全整数、`cwd` 绝对路径等。

---

## 7. Session 的创建、恢复（resume）、fork

### 7.1 创建 — `Session.create()` 与 `SessionStore.create()`

**`Session.create(id, seed, header)`**（`index.js:1355-1357`）——静态工厂，创建 detached session：

```js
static create(id, seed, header) {
    return new Session(id, seed, header);  // mode = "snapshot"
}
```

**`SessionStore.create(id, options)`**（`index.js:1616-1623`）——创建绑定到调用 fiber 的 live session：

```js
create(id, options) {
    const session = this.prepare(id, options);
    this.ctx.effect(function* () {
        yield this.enter(session);     // 先进入（yield detach 用于回滚安全）
        this.announce(session);        // 后公告
    }.bind(this), "sessions.create()");
    return session;
}
```

> 🔑 **yield 顺序**：先 `yield this.enter(session)`（返回 detach disposer），再 `this.announce(session)`。如果 `session/created` listener 同步抛错，generator effect 会 dispose 已 yield 的 disposer（回滚 attach），而不是泄露 store entry。

### 7.2 prepare — 构造但不发布

`SessionStore.prepare(id, options)`（`index.js:1644-1666`）验证 id/cwd 并构造 `Session`，但**不进入 store**：

```js
prepare(id, options) {
    let sessionId;
    if (id === undefined) do sessionId = SessionId(`session-${++this.counter}`);
        while (this.store.has(sessionId));
    else sessionId = SessionId(id);
    if (this.store.has(sessionId)) throw new Error(`session "${sessionId}" already exists`);
    if (options?.seedSource === "persistence")
        return Session.fromRestore(sessionId, options.seed, options.meta);
    // 构造 header
    const header = { version: 0, id: sessionId, createdAt: meta?.createdAt ?? Date.now(), ... };
    return Session.create(sessionId, seed, header);
}
```

> 🔑 **persistence 路径**：`seedSource: 'persistence'` 时走 `Session.fromRestore`——所有权转移路径，不做二次序列化拷贝。

### 7.3 恢复（resume）— `Session.fromRestore()`

`Session.fromRestore(id, seed, header)`（`index.js:1367-1369`）——**持久化恢复**路径：

```js
static fromRestore(id, seed, header) {
    return new Session(id, seed, header, "restore");  // mode = "restore"
}
```

restore 模式与 snapshot 模式的区别（构造函数 `index.js:1370-1388`）：

| 方面 | snapshot 模式 | restore 模式 |
|------|--------------|--------------|
| header 验证 | `snapshotSessionHeader`（detach + freeze） | `validateRestoredSessionHeader`（原地 freeze） |
| 事件 detach | `snapshotJsonValue(source)` 深拷贝 | 直接使用 `source`（所有权转移） |
| 事件冻结 | `deepFreeze(snapshot)` | `freezeRestoredObject(snapshot)`（迭代式，不消耗调用栈） |

> 🔑 **所有权转移**：restore 模式要求调用方传入**新鲜 detached** 的图，不保留可变别名。验证后原地冻结（`freezeRestoredObject` 用迭代式 `Object.freeze` 避免递归栈溢出，`index.js:1176-1187`）。

**持久化协调器的恢复流程**（`dsh-session-persistence/lib/index.js:986-1017`）：

```js
async prepareCore(id) {
    const stored = await this.backend.loadStored(id);  // 从磁盘读取
    const { meta, events, revision, tornMarker } = stored;
    const storedEvents = adoptStoredEvents(events, id);  // 原地验证+冻结
    const closers = interruptedTurnClosers(storedEvents).map(adoptSessionEvent);  // 崩溃修复
    const balanced = [...storedEvents, ...closers];
    const session = this.ctx.sessions.prepare(id, {
        seed: balanced, meta, seedSource: "persistence"  // 走 fromRestore
    });
    return { inspection: ..., session, revision, sessionLength, tornMarker, closers };
}
```

### 7.4 session/end-seed — 种子边界标记

构造函数在种子末尾追加 `session/end-seed` 事件（`index.js:1387`）：

```js
if (seed !== void 0 && this.log.at(-1)?.type !== "session/end-seed")
    this.append("session/end-seed", {});
```

> 🔑 **`firstLiveSeq` vs `header.seedLength`**：
> - `firstLiveSeq`（`index.js:1346`）：**进程内**构造事实——构造种子长度。此 seq 之前的事件通过构造进入，**不发布**到 `session/event` firehose。
> - `header.seedLength`：**持久化**的 fork 血统边界——resume 的 session 构造种子是完整存储日志，但 header 保持原始 fork 值。
> - `session/end-seed` 事件是 `firstLiveSeq` 的**持久化投影**。读取存储历史的消费者应找**最后一个**此事件（重开未触碰的 session 不会重新标记）。

### 7.5 Fork — `SessionStore.fork()`

`fork(source, boundary, childSessionId)`（`index.js:1840-1852`）从 live session 的稳定前缀创建子 session：

```js
fork(source, boundary, childSessionId) {
    if (childSessionId !== void 0 && this.get(childSessionId) !== void 0)
        throw new SessionForkError(`already exists`, "SESSION_ALREADY_EXISTS");
    const liveSource = this._resolveForkSource(source);
    const seed = this._forkSeed(liveSource, boundary);
    return this.create(childSessionId, {
        seed,
        meta: {
            ...liveSource.header.cwd !== void 0 ? { cwd: liveSource.header.cwd } : {},
            parentSession: liveSource.id,
            seedLength: seed.length
        }
    });
}
```

**`_forkSeed`**（`index.js:1853-1872`）选择种子前缀并验证边界：

```js
_forkSeed(session, requestedBoundary) {
    const events = session.events;
    let boundary = requestedBoundary ?? events.at(-1)?.seq;
    ...
    // 验证：非负安全整数、存在、seq 连续
    // 关键：不能在 open turn 内 fork！
    const lastTurnBoundary = events.slice(0, boundary + 1)
        .findLast(event => event.type === "turn/start" || event.type === "turn/end");
    if (lastTurnBoundary?.type === "turn/start")
        throw new SessionForkError(`ends inside open turn`, "OPEN_TURN");
    return events.slice(0, boundary + 1);
}
```

> 🔑 **Fork 约束**：选定前缀可以以 turn 间事件结尾，但**不能在 open turn 内部**结束。这保证 fork 出的子 session 有一个 provider-valid 的 transcript。

**Fork 错误码**（`types/index.d.ts:278`）：

```ts
export type SessionForkErrorCode =
    | 'SESSION_NOT_FOUND'    // fork 源 id 未知
    | 'SESSION_NOT_LIVE'     // 不是 store 的 live 实例
    | 'SESSION_ALREADY_EXISTS' // 子 id 已占用
    | 'INVALID_BOUNDARY'     // 边界非连续 seq
    | 'OPEN_TURN';           // 前缀在 open turn 内结束
```

### 7.6 拆分生命周期 — prepare + enter + announce

> 🔑 **何时用拆分生命周期**：当 teardown 必须与另一资源**有序**时。`dsh-agent-loop` 用此模式，使最终 loop flush 先于 session detach——避免 racing sibling effects 在 driver 的关闭事件提交前移除发布钩子。

- `prepare(id, options)`：验证+构造，不发布
- `enter(session)`：碰撞检查 + 安装发布钩子 + 加入 store，返回 detach disposer。**不发** `session/created`
- `announce(session)`：发 `session/created`（仅一次）

`enter` 的延迟 detach 逻辑（`index.js:1710-1718`）：

```js
const detach = () => {
    if (!entered) return;
    entered = false;
    if (entry.announcing || entry.appending) {
        entry.detachRequested = true;  // 延迟到 dispatch unwind
        return;
    }
    entry.detach();
};
```

> 🔑 **延迟 detach**：在 `session/created` 同步 dispatch 或 append 发布期间请求 detach，会延迟到 dispatch unwind 后执行——确保配对的 disposal edge 正确发布。

---

## 8. SessionStore 的实现

### 8.1 类定义与 Cordis 集成

`SessionStore`（`index.js:1580` / `types/index.js:647`）继承 Cordis `Service`，注册为 `ctx.sessions`：

```js
var SessionStore = class extends Service {
    store = new Map();   // id → entry
    counter = 0;
    constructor(ctx) {
        super(ctx, "sessions");
        ctx.inject(["typert"], (typeCtx) => {
            typeCtx.typert.lookups.register("session", {
                parameter: "session", wire: "sessionId",
                hostTypeSymbol: "@deepseek-ai/dsh-session#Session",
                wireTypeSymbol: "@deepseek-ai/dsh-session/types#SessionId",
                resolve: sessionId => this.get(sessionId)
            });
        });
    }
    ...
}
```

> 🔑 **Typert 集成**：注册 `session` 类型查找，使 `@deepseek-ai/dsh-typert-protocol` 能从 `sessionId` 解析到 `Session` 对象。

### 8.2 entry 结构 — 模块私有附件

`attachments` 是模块私有的 `WeakMap`（`index.js:1294`），将 `Session` 映射到 entry，保持 `Session` 类**store-agnostic**：

```js
const attachments = new WeakMap();
// entry = {
//     id, session, carrier, emitCtx,
//     announced, announcing, appending,
//     detachRequested, detach: () => { this.detachEntered(entry); }
// }
```

> 🔑 **WeakMap 而非实例字段**：`Session` 是普通类，持久化插件可能创建 detached 实例（不在 store 中）。`attachments` 让"是否在 store 中"成为外部状态，`Session.append()` 通过 `attachments.get(this)` 检测——无 entry 则不发事件（detached session）。

### 8.3 liveEntryFor — 精确活性检查

`liveEntryFor`（`index.js:1806-1809`）确保操作的是**精确的 live entry**：

```js
liveEntryFor(session) {
    const entry = attachments.get(session);
    if (entry === void 0 || this.store.get(entry.id) !== entry)
        throw new Error(`session "${session.id}" is not live in this store`);
    return entry;
}
```

> 🔑 **stale 防护**：不仅检查 `attachments` 有 entry，还检查 `store.get(entry.id) === entry`——防止一个已被替换的 entry（同 id 新生命周期）被旧 capability 操作。

### 8.4 detachEntered — 精确移除

`detachEntered`（`index.js:1722-1728`）：

```js
detachEntered(entry) {
    entry.detachRequested = false;
    if (this.store.get(entry.id) !== entry) return;  // stale capability 无效
    this.store.delete(entry.id);
    attachments.delete(entry.session);
    if (entry.announced) this.emitDisposed(entry);  // 仅已公告的才发 disposed
}
```

> 🔑 **配对语义**：`session/disposed` 只对**已开始创建公告**的 entry 发出。未公告的 entry（如公告前 throw 回滚）不发 disposal edge。

### 8.5 完整 API 总结

| 方法 | 返回 | 说明 |
|------|------|------|
| `create(id?, options?)` | `Session` | 创建+进入+公告（便捷） |
| `prepare(id?, options?)` | `Session` | 构造不发布 |
| `enter(session)` | `() => void` | 进入 store，返回 detach disposer |
| `announce(session)` | `void` | 发 `session/created` |
| `flush(session)` | `Promise<boolean>` | await 持久化检查点 |
| `get(id)` | `Session \| undefined` | 查找 live session |
| `list()` | `Session[]` | 所有 live session（创建顺序） |
| `fork(source, boundary?, childSessionId?)` | `Session` | 从前缀派生子 session |

---

## 9. 不变量系统（Invariant Companion）

`lib/invariant.js`（168 行）是可选的 Cordis 伴随插件，注册包拥有的**关系型不变量**到 `ctx.invariants`。

### 9.1 验证的规则

`validateEvent`（`invariant.js:20-99`）验证每个事件：

1. **seq 单调递增**：`event.seq <= trace.lastSeq` → fail
2. **turn/step 包含**：`turn/start` 不能在 open turn 中；`step/*`、`assistant/*`、`tool/call` 必须在 open step 中
3. **工具调用配对**：`tool/result` 必须有同 step 的前置 `tool/call`（除非是 `TOOL_NOT_STARTED` 合成结果）
4. **核心执行事件 turn 包含**：`todo/write`、`request/header`、`request/context` 必须在 open turn 中

### 9.2 两阶段验证 — pre-commit staging

> 🔑 **关键设计**：不变量验证在事件**提交前**进行（通过 `internal/dispatch` 钩子），但状态转换在**提交后**应用（通过 `session/event` 钩子）。这确保一个验证通过但 commit 失败的事件不会污染 trace 状态。

```js
ctx.on("internal/dispatch", (_mode, eventName, args) => {
    if (eventName !== "session/event") return;
    const [session, event] = args;
    const trace = traceFor(session);
    const transition = validateEvent(trace, event, fail);  // 验证
    stagedTransitions.set(event, { session, trace, transition });  // 暂存
}, { global: true });

ctx.on("session/event", (session, event) => {
    const staged = stagedTransitions.get(event);
    if (staged === void 0 || staged.session !== session)
        return fail("session/event reached publication without matching pre-commit validation");
    stagedTransitions.delete(event);
    applyTransition(staged.trace, staged.transition);  // 应用
}, { global: true });
```

### 9.3 种子重放

`seedSession`（`invariant.js:130-135`）在 session 创建时重放所有已有事件建立 trace：

```js
const seedSession = (session) => {
    const trace = freshTrace();
    traces.set(session, trace);
    for (const event of session.events) applyTransition(trace, validateEvent(trace, event, fail));
    return trace;
};
```

---

## 10. 崩溃恢复（Crash Repair）

`lib/types/repair.js`（`index.js:606-725`）为中断的 session 日志提供崩溃恢复。

### 10.1 interruptedTurnClosers — 合成关闭事件

`interruptedTurnClosers(events)`（`index.js:626-725`）扫描日志，为未关闭的 turn 生成确定性合成事件：

```js
function interruptedTurnClosers(events) {
    let openTurn = null, openStep = null;
    const pendingCalls = new Map();
    // 扫描：追踪 open turn/step 和 pending tool calls
    for (const event of events) switch (event.type) {
        case "turn/start": openTurn = event.data.turn; pendingCalls.clear(); break;
        case "turn/end": openTurn = null; pendingCalls.clear(); break;
        case "step/start": openStep = event.data.step; break;
        case "step/end": pendingCalls.clear(); openStep = null; break;
        case "assistant/message":
            // 收集 tool-call blocks
            for (const block of event.data.message.content)
                if (block.type === "tool-call") pendingCalls.set(block.id, { step: ... });
            break;
        case "tool/call": // 标记 callSeq
        case "tool/result": // 从 pending 移除
    }
    // 生成 closers...
}
```

### 10.2 两种恢复场景

| 恢复码 | 场景 | 合成结果内容 |
|--------|------|-------------|
| `TOOL_NOT_STARTED` | 助手请求了工具但未记录 `tool/call` 开始 | "The tool call was interrupted before the Harness recorded it as started. Retry it if it is still needed." |
| `TOOL_OUTCOME_UNKNOWN` | 记录了 `tool/call` 但无结果 | "The tool call was interrupted after it was recorded, but no result was durably recorded. Its outcome is unknown. Decide whether to retry..." |

> 🔑 **确定性**：合成事件序列继续日志（`seq = last.seq + 1`），时间戳复用最后一个真实事件。未匹配的 call 先收到 error result，然后 `step/end`（如果 open），最后 `turn/end`（`reason: { kind: "interrupted" }`）。

### 10.3 恢复在持久化层的应用

`dsh-session-persistence/lib/index.js:995-996`：

```js
const closers = interruptedTurnClosers(storedEvents).map(adoptSessionEvent);
const balanced = [...storedEvents, ...closers];
```

恢复后如果有 closers，则提交修复到磁盘（`commitRepair`，`index.js:1025-1027`）。

---

## 附录 A：无损 JSON 工具（json.js）

`snapshotJsonValue`（`index.js:194-196`）和 `isJsonValue`（`index.js:204-206`）基于 `walkJsonValue`（`index.js:74-181`）——一个迭代式（非递归，避免栈溢出）的无损 JSON 验证器：

**接受**：普通数组、plain/null-prototype 对象、JSON 标量
**拒绝**：稀疏数组、循环引用、exotic 对象（Map/Set/Date/类实例）、`-0`、非有限数字、非字符串非可枚举键

> 🔑 **一次性遍历**：验证和拷贝在同一遍中完成，每个属性只读一次——有状态 getter 无法在验证和拷贝间提供不同值。`detach=true` 时物化分离快照，`detach=false` 时只验证。

---

## 附录 B：SessionPreparation — 未发布 Session 的所有权

`SessionPreparation`（`index.js:580-604` / `preparation.js`）用 `Symbol.dispose` 管理未发布 Session 的 provider 状态：

```js
var SessionPreparation = class {
    static create(session, options) { return new SessionPreparation(session, options ?? {}); }
    [Symbol.dispose]() {
        if (this.released) return;
        this.released = true;
        this.options.release?.();  // 释放 provider 状态
    }
};
```

> 🔑 **幂等释放**：disposal 同步且幂等。provider 决定 release 是返回 Session 到缓存还是丢弃；publication 可能在 disposal 前消费该状态，使 callback 成为 no-op。

---

## 附录 C：格式版本策略

`SESSION_FORMAT_VERSION = 0`（`index.js:37` / `types.js:32`）：

> 🔑 **版本策略核心**：版本是一个**单调整数**（无 major/minor 分割）。是否需要 bump 由**writer 发出什么**决定，而非 newer reader 能接受什么。只有结构性变更（header 形状、事件信封、核心语义、Surface 机制）才 bump。添加普通事件类型**不 bump**——`ignorable` 标记覆盖词汇增长。未发布期间固定为 0：不隐含兼容性，不兼容日志被拒绝，不提供迁移。

---

## 附录 D：关键文件索引

| 文件 | 行数 | 职责 |
|------|------|------|
| `lib/index.js` | 1886 | 打包产物（所有模块内联） |
| `lib/types/index.js` | 999 | 源文件：Session + SessionStore 类 |
| `lib/types/types.js` | 33 | SessionId brand + SESSION_FORMAT_VERSION |
| `lib/types/surface.js` | ~300 | Surface 投影层（browser-safe） |
| `lib/types/request-header.js` | ~50 | 请求头折叠重建 |
| `lib/types/json.js` | ~170 | 无损 JSON 验证/detach |
| `lib/types/chunk-rows.js` | ~310 | delta chunk 存储压缩编解码 |
| `lib/types/repair.js` | ~120 | 崩溃恢复合成事件 |
| `lib/types/preparation.js` | ~25 | 未发布 Session 所有权 |
| `lib/types/known-event-types.js` | ~50 | 生成的事件类型目录 |
| `lib/invariant.js` | 168 | 关系型不变量伴随插件 |

---

*本文档基于 `@deepseek-ai/dsh-session@0.1.0-rc.6` 源码逐行精读生成。*
