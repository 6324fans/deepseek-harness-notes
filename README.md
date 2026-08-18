# DeepSeek Harness 学习笔记

基于 `deepseek-ai/deepseek-harness` 源码整理的中文学习资料，重点分析 Agent Loop、工具调用、上下文管理、Hook 与 Steering 等核心机制。

当前版本还包含固定提交复现方法、`turn-stopping` 与 steering 时序、Session 持久化检查点、工具副作用恢复、安全威胁模型、可运行插件实验和核心不变量清单。

源码分析基准：`@deepseek-ai/dsh@0.1.0-rc.6`

## 阅读入口

### 综合文档

- [DSH 实现深度解析（主文档）](./DSH-implementation-deep-dive.md) — 18 章，覆盖启动引导、Cordis 内核、动态 Prompt、缓存优化、工具系统、Session、LLM 适配层、子代理、Goal、Workflow、Web GUI、沙箱、Skill、Jobs 全部 14 个核心模块
- [DeepSeek Harness 源码学习指南](./deepseek-harness-learning-guide.zh.md) — Agent Loop、工具调用、上下文管理、Hook 与 Steering

### 专题深度分析

- [Session 会话系统](./dsh-session-analysis.md) — 事件溯源、Projection 投影、JSONL 持久化、Header 管理、创建/恢复/fork
- [Goal 长目标系统](./dsh-goal-system-analysis.md) — 状态机、Round Driver、权限矩阵、arm/disarm、事件溯源
- [沙箱与文件系统](./DSH-sandbox-fs-analysis.md) — 三模式沙箱、sandbox-policy 决策、Bash 拦截、Approval 机制、Permission Presets
- [Bash 沙箱分析](./dsh-bash-analysis-report.md) — Bash 命令执行沙箱拦截的交叉验证报告

## 文档关系

```
DSH-implementation-deep-dive.md（主文档，18 章概览）
  ├── dsh-session-analysis.md（Session 系统专题）
  ├── dsh-goal-system-analysis.md（Goal 系统专题）
  ├── DSH-sandbox-fs-analysis.md（沙箱与文件系统专题）
  └── dsh-bash-analysis-report.md（Bash 沙箱交叉验证）
```

主文档的 §9-§17 是各专题的浓缩版，专题文档包含完整的代码行号引用和时序图。

## 上游项目

- [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness)
