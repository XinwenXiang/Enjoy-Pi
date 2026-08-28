# 00 · Agent-loop 零基础

## 1. 先看普通 Chatbot

普通 Chatbot 通常只调用模型一次：

```text
用户问题 → LLM → 文本答案 → 结束
```

模型不会真的读取文件、执行 shell 或查询实时天气。它只是根据输入生成下一段内容。

## 2. Agent 为什么需要 loop

Agent 给模型一组工具定义。模型一次响应可以选择：

- 直接给最终答案；或
- 返回一个 tool call，说明想调用哪个工具、参数是什么。

程序看到 tool call 后，真正执行函数，把 tool result 加进消息，再调用一次模型：

```text
用户任务
  ↓
LLM 决策
  ├─ 没有 tool call → 最终答案 → 结束
  └─ 有 tool call
       ↓
     程序执行工具
       ↓
     tool result 加入 messages
       ↓
     再调用 LLM ─────────────┘
```

关键点：**不是 LLM 自己在循环，是 harness/应用程序在循环调用 LLM。** LLM 每次只做一次决策。

## 3. 天气例子

用户问：“北京现在天气怎么样？”

1. User message：问题进入 `messages`。
2. Assistant message：模型请求 `get_weather({ city: "Beijing" })`。
3. Tool result：程序执行真实函数，得到 `28°C, sunny`，再加入 `messages`。
4. Assistant message：模型看到结果，生成最终自然语言答案。
5. 最终响应没有 tool call，loop 退出。

## 4. 最小伪代码

```ts
const messages = [userMessage];

while (true) {
  const response = await llm(messages, tools);
  messages.push(response);

  const calls = response.toolCalls;
  if (calls.length === 0) {
    return response; // 最终答案
  }

  for (const call of calls) {
    const result = await runTool(call);
    messages.push(result);
  }
}
```

Pi 增加了流式输出、事件、并行工具、参数校验、steering/follow-up、abort 和错误处理，但核心骨架仍是这个循环。

## 5. 对应到 Pi 源码

| 伪代码 | Pi 源码 |
|---|---|
| `while (...)` | `packages/agent/src/agent-loop.ts → runLoop()` |
| `llm(...)` | `streamAssistantResponse()` |
| 找 tool calls | `message.content.filter(c => c.type === "toolCall")` |
| `runTool(...)` | `executeToolCalls()` |
| 保存 tool result | `currentContext.messages.push(result)` |
| 没有更多任务时结束 | `agent_end` |

## 6. 内层循环与外层循环

`runLoop()` 中有两层循环：

- **内层循环**：处理模型产生的工具调用，以及用户在运行中插入的 steering message。
- **外层循环**：模型本来要结束时，再检查 follow-up queue；如果有，就重新进入内层循环。

先只理解工具调用主线，再读队列逻辑。不要一开始同时理解所有分支。
