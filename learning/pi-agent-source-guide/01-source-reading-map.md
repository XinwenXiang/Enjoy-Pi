# 01 · Pi 源码阅读地图

## 四层心智模型

| 层 | 主要位置 | 作用 |
|---|---|---|
| Model / Provider | `packages/ai` | 把不同模型供应商统一为流式调用接口 |
| Agent loop | `packages/agent/src/agent-loop.ts` | LLM ↔ tool 的最小控制循环 |
| Stateful Agent | `packages/agent/src/agent.ts` | messages、事件、abort、steer/follow-up queue |
| Application harness | `packages/coding-agent/src/core/agent-session.ts` | 持久化、skills、extensions、retry、compaction、system prompt |

## 跟一次 prompt

### 1. `AgentSession.prompt()`

文件：`packages/coding-agent/src/core/agent-session.ts`

依次观察：

1. extension command 是否提前处理；
2. input hook 是否拦截或变换输入；
3. skill 与 prompt template 如何展开；
4. 模型与认证如何校验；
5. `before_agent_start` 如何修改 messages/system prompt；
6. 最终如何进入 `_runAgentPrompt()`。

### 2. `Agent.prompt()`

文件：`packages/agent/src/agent.ts`

它负责：

- 阻止同时运行两个 prompt；
- 建立 `AbortController`；
- 复制当前 context；
- 把队列读取函数、hook、model 和工具配置交给 loop；
- 根据事件更新内存 state；
- 等待订阅者处理完 `agent_end` 后才真正 idle。

### 3. `runAgentLoop()` / `runLoop()`

文件：`packages/agent/src/agent-loop.ts`

主线：

```text
agent_start
→ turn_start
→ user message events
→ streamAssistantResponse
→ tool calls?
  → yes: executeToolCalls → tool results → next turn
  → no: check follow-up → agent_end
```

### 4. Tool 执行

先找这些函数：

- `executeToolCalls()`
- `executeToolCallsSequential()`
- `executeToolCallsParallel()`
- `prepareToolCall()`
- `executePreparedToolCall()`
- `finalizeExecutedToolCall()`

并行模式里，工具可以按完成速度产生事件，但写回对话上下文的 tool results 仍保持 assistant 原始 tool-call 顺序。

### 5. Session 收事件并持久化

回到 `AgentSession._handleAgentEvent()`：

- `message_end`：写入 `SessionManager`；
- `turn_end`：安全地刷新待插入 custom messages；
- `agent_end` 后：检查 retry、auto-compaction 与新队列消息；
- 同一批事件也会发给 extension 和 UI。

## 五站学习计划

1. `agent-loop.ts` + `types.ts`：画出最小循环。
2. `agent.ts`：分清 transcript state、streaming state 和 queues。
3. `agent-session.ts`：列出 prompt 进入 Agent 前的加工步骤。
4. `resource-loader.ts` + `extensions/runner.ts`：分清 tool、skill、extension、AGENTS.md。
5. `docs/harness.md` + `harness/agent-harness.ts`：学习 durable runtime，但不要误认成当前完整实现。
