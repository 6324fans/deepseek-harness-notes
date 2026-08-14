# DeepSeek Harness 学习笔记

基于 `deepseek-ai/deepseek-harness` 源码整理的中文学习资料，重点分析 Agent Loop、工具调用、上下文管理、Hook 与 Steering 等核心机制。

当前版本还包含固定提交复现方法、`turn-stopping` 与 steering 时序、Session 持久化检查点、工具副作用恢复、安全威胁模型、可运行插件实验和核心不变量清单。

源码分析基准：`47f943859bef60e4160492346772ded9b24f765a`

## 阅读入口

[DeepSeek Harness 源码学习指南](./deepseek-harness-learning-guide.zh.md)

## 上游项目

- [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness)
