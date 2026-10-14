# Pi Agent 源码学习指南（中文）

面向 Agent 开发初学者的个人学习材料。初版基于上游 `56f3f33`；本次于 2026-10-14 核对上游 [`60ac3bcc3`](https://github.com/earendil-works/pi/tree/60ac3bcc3b8d6df34a44c0ed677a9eb9c74e52ac)，修正过期入口并增加独立演进章节。

[交互式网页教程](https://pi-agent-harness-guide.leontine-xiang.chatgpt.site) · [直接看架构演进](https://pi-agent-harness-guide.leontine-xiang.chatgpt.site/#evolution)

## 现在应该记住的三件事

- 普通 Pi CLI 仍是 `AgentSession → Agent → runAgentLoop / runLoop → LLM / tools`。
- MCP 已在 v0.99.0 加入，作为可替换、可禁用的内置 extension；它连接外部工具，codemode 组合工具，agent-loop 仍负责循环。
- 旧 `packages/agent/src/harness/` 在 v1.0.0 删除。当前可恢复 runtime 的入口是独立的实验性 `packages/durable`，不是“AgentHarness 仍是骨架”。

## 建议顺序

1. [00-agent-loop-basics.md](./00-agent-loop-basics.md)：从零理解 tool call、tool result 和循环。
2. [01-source-reading-map.md](./01-source-reading-map.md)：拆读普通 CLI 的主调用链。
3. [03-architecture-evolution.md](./03-architecture-evolution.md)：独立章节；解释为什么引入 MCP、两条演进线、direct/codemode 调用、breaking changes 与当前源码入口。
4. [02-durable-harness-design.md](./02-durable-harness-design.md)：历史设计笔记；保留旧 Entry/Register/Lane/Operation 概念，带当前状态说明。要实现新项目，优先读 03 链接的 `pi-durable` 文档。

这些文件是个人学习材料，不属于上游官方文档。教程核对的是固定上游快照；本次没有将 fork 的 Pi 包代码升级到上游 main，也没有迁移任何 session 数据。
