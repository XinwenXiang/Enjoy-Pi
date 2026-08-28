# Pi Agent 源码学习指南（中文）

这是一套面向 Agent 开发初学者的个人学习材料，基于 `earendil-works/pi` 源码快照 `56f3f33`。

交互式网页教程：<https://pi-agent-harness-guide.leontine-xiang.chatgpt.site>

## 最重要的结论

当前 Pi Coding Agent 的真实主调用链是：

```text
AgentSession
  → Agent
    → runAgentLoop()
      → streamAssistantResponse()
      → executeToolCalls()
      → 下一轮 LLM 调用
```

`packages/agent/src/harness/agent-harness.ts` 代表正在建设的 durable harness 方向。当前很多公开方法仍会抛出 `HarnessNotImplemented`，所以阅读时必须区分：

- **今天已经运行的实现**：`Agent`、`agent-loop`、`AgentSession`
- **新一代目标设计与接口骨架**：`AgentHarness`、`packages/agent/docs/harness.md`

## 建议顺序

1. [00-agent-loop-basics.md](./00-agent-loop-basics.md)：完全从零理解 agent-loop。
2. [01-source-reading-map.md](./01-source-reading-map.md)：按一次请求拆读真实源码。
3. [02-durable-harness-design.md](./02-durable-harness-design.md)：理解可恢复 harness 的设计目标。

这些文件是个人学习材料，不属于 Pi 上游官方文档；遇到差异时，以仓库当前源码和官方文档为准。
