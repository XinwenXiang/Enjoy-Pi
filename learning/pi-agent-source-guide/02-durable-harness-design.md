# 02 · Durable AgentHarness 设计（历史笔记）

**2026-10-14 状态修正：**本页保留初版 `56f3f33` 的设计解释。旧 Harness 后来已有可运行实现，并在 v1.0.0 从 agent-core 删除；当前入口是独立、仍实验性的 `packages/durable`。下面的 Registers、Lane、Operation 和 Usage ledger 是历史设计术语，不能当作当前 API。请先读 [03 · 架构演进](./03-architecture-evolution.md)。

## 为什么最小 Agent 不够

内存里的 loop 可以完成一次请求，但进程如果在工具执行一半时崩溃，重启后会遇到问题：

- 用户输入是否已经接受？
- 模型请求是否已经收费？
- 删除文件的工具是否已经执行？
- 能不能安全重试？
- 对话树当前 leaf 在哪里？

Durable harness 的目标是把“程序执行到哪里”也持久化。

## 三类权威存储

### Entries

不可变、write-once 的对话树节点：message、compaction、branch summary 或应用自定义 entry。

### Registers

保存可覆盖的当前状态，例如 facts、lane state、operation state 和尚未放进树的 pending entry。

### Usage ledger

只追加的 token 与成本记录。

设计约束：一个 durable payload 必须属于这三类之一，不能只藏在进程内的临时对象中。

## Lane 与 Operation

- **Lane**：对话树上的命名游标。每个 session 有 `main`，也可以有 Slack thread 或 subagent lane。
- **Operation**：lane 接受的一次工作单元，例如 run、compaction 或 navigation。
- `op.state/{operationId}` 保存完整的当前状态，相当于 durable program counter。

恢复时不是回放一串日志来猜位置，而是读取完整 `op.state`，根据 phase 继续。

## Effect sandwich

模型请求和真实工具调用属于外部 effect，使用两次提交包住：

```text
TX 1: 记录 intent，并预留输出 ID
      ↓
      执行模型请求 / 工具调用（不确定窗口）
      ↓
TX 2: 原子写入输出、usage 与下一 operation state
```

这样即使崩溃，也能知道最后确认提交到哪一步。

## 为什么仍不是 exactly-once

如果外部动作已经成功，但进程在 TX 2 前崩溃，harness 只知道它“可能执行过”。

因此工具需要声明恢复策略：

- `replay: "safe"`：例如只读查询，可以重新执行；
- `replay: "never"`：例如删除操作，不自动重放，写入一个中断错误结果继续。

这个设计明确管理不确定性，但不能凭空让外部系统具备 exactly-once 语义。

## 当时的实现状态与当前入口

初版快照中的 `AgentHarness` 公开方法有 `unavailable()` / `HarnessNotImplemented`。这个结论不再描述当前 main。

- 2026-09：旧 Harness 已推进为可运行实验 runtime；Pico 随后移入独立 durable 包。
- 2026-10-01 / v1.0.0：旧 `AgentHarness` 与 `./harness/*` 等 exports 从 agent-core 删除。
- 本次快照：使用 `@earendil-works/pi-durable` 的 `Harness / Conversation / Submission / Task / Document`。概念映射与路径见 03；这不是只改 import 名称即可完成的兼容迁移。
- 普通 Pi CLI 仍由 `AgentSession + Agent + agent-loop` 驱动。

[当前 pi-durable README](https://github.com/earendil-works/pi/blob/60ac3bcc3b8d6df34a44c0ed677a9eb9c74e52ac/packages/durable/README.md) · [历史 Harness 文档（固定旧快照）](https://github.com/earendil-works/pi/blob/56f3f33/packages/agent/docs/harness.md)
