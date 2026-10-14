# 03 · Pi 架构演进：为什么又引入 MCP？

更新：2026-10-14（北京时间）。核对上游 `earendil-works/pi` 快照 [`60ac3bcc3`](https://github.com/earendil-works/pi/tree/60ac3bcc3b8d6df34a44c0ed677a9eb9c74e52ac)。这是个人学习材料，代码示例中标为“示意”的工具名不代表 Pi 内置工具。

## 1. 先回答你的问题

**Pi 过去明确选择不内置 MCP；现在改变了这个产品选择。但它保留了小型 agent-core，将 MCP 做成可以替换或禁用的内置 extension。**

在 2026-09-11 的旧源码中，Coding Agent README 仍写着 **No MCP**，建议使用 CLI、Skills，或自己添加 MCP extension。2026-09-29 的提交 [`8562bcf`](https://github.com/earendil-works/pi/commit/8562bcf66a8eeefdf75ddd231ca9d7aa7d64f86e) 引入 codemode、tool search 和 MCP，并随 **v0.99.0** 发布。

作者在 [PR #10040](https://github.com/earendil-works/pi/pull/10040) 给了两层理由：codemode 的主要动力是一些模型（例如 Jev）更适合在 sandbox 中组合工具；对于 MCP，作者仍有保留，但认为部分用途合理，基础能力能够融入 Pi。该 PR 页面并未标记 merged；落地证据是上述提交与发布日志。

从实现看，Pi 同时处理了三个问题：外部服务如何接入、大量工具如何按需发现、多步调用如何通过代码组合。把这解释为“兼顾服务互通与上下文控制”是本教程的架构归纳，不是作者声称放弃了最小核心原则。

**MCP 本身是工具连接协议；它不会代替 agent-loop，也不会自动提供崩溃恢复。**

## 2. 把四个概念放在各自的位置

| 概念 | 它回答什么 | 天气助手里的作用 |
|---|---|---|
| Agent loop | 下一轮还要不要调用模型？ | 天气工具返回后，再让模型组织答案 |
| MCP | 怎样发现和调用另一进程/服务提供的能力？ | 从天气 MCP server 发现 `get_weather`，发送调用并接收结果 |
| Codemode | 怎样用一段代码组合多个工具、过滤结果？ | 并行查北京和上海，仅输出温度与天气摘要 |
| Durable Harness | 进程重启后怎样知道做到哪一步？ | 持久化输入、generation/tool task 与 checkpoint，再按恢复策略继续 |

AgentSession 会保存聊天历史，但“保存历史”不等于“每一步执行都能恢复”。MCP server 可以用本地 stdio，也可以用 Streamable HTTP；接了 MCP 并不代表必须部署远程服务。

## 3. 一张演进表：两条线并行

| 时间 / 版本 | 工具与正常 CLI 这条线 | 可恢复 runtime 这条线 | 读旧教程时要改什么 |
|---|---|---|---|
| 2025-11 至 2026-09-11 | 选择不内置 MCP；CLI/Skills/extensions 承载外部能力 | 原 `AgentHarness` 从设计骨架逐步成为可运行实验实现 | 初版“全是骨架”的结论只适用于当时快照 |
| 2026-08-28 · v0.84.4 | 下一轮前可先 compaction；`prepareNextTurn` 只在确实继续时执行 | 旧 Harness 在持续建设 | 收尾代码移到 `agent_end`，不能依赖最后一次 `prepareNextTurn` |
| 2026-09-18 | 普通 CLI 仍使用 `AgentSession → Agent → agent-loop` | Pico 移入独立 `packages/durable`，提交 [`0801601`](https://github.com/earendil-works/pi/commit/080160162164b6668f6c55b8bc1f335749f74eb3) | 独立 durable 包已经出现，不再只读早期设计文档 |
| 2026-09-21 · v0.87.0 | `shouldStopAfterTurn` 被 `finishTurn` 替代；加入 `prepareRequest` | durable 包继续发展 | 旧停止 hook 需要迁移 |
| 2026-09-29 · v0.99.0 | 内置 codemode、tool_search、MCP；增加通用工具 exposure 与嵌套调用 | durable 是另一条 runtime 路径 | 从“没有内置 MCP”改为“可替换的内置 extension” |
| 2026-10-01 · v1.0.0 | agent-core 收缩为 Agent、loop、proxy stream 与类型 | 从 agent-core 删除旧实验 Harness 和相关 exports；使用 `pi-durable` | `AgentHarness` import 和 `src/harness/` 阅读链接失效 |
| 2026-10-12 · v1.2.0 | `reachesTools` 与 namespace aliases 解耦 codemode/MCP；连接时不再枚举资源列表 | 独立 durable 包提供可执行 Harness、Conversation、Task，也支持 code mode / tool search | 大版本号不代表实验 durable API 已稳定；`pi mcp list --json` 输出也已改变 |

这些变化不是“Agent loop → MCP → durable”的替换关系。正常 CLI 的 loop、MCP 的工具接入和 durable 的恢复语义在不同层演进。

## 4. MCP 接入后，一次请求到底怎么走？

### 路径 A：模型直接调用 MCP 工具

前提：服务器或该工具配置 `exposure: "direct"`。以下 `weather/get_weather` 是示意 server/tool。

1. MCP extension 连接 server，通过 `listTools()` 取得名称、描述和 JSON Schema。
2. `createMcpToolDefinition()` 把它包装为 Pi 的工具，例如 `mcp__weather__get_weather`。
3. 工具被声明给模型。模型返回 tool call；这仍然是普通 agent-loop。
4. Pi 校验参数、执行 `tool_call` hook，然后工具 wrapper 调用 `McpClient.callTool()`。
5. stdio / HTTP transport 将请求送到 MCP server；**真正查询天气的是 server 的实现**。
6. `convertMcpResult()` 转换结果；执行 `tool_result` hook，结果进入对话，再调用模型。

MCP 工具进入 Pi 统一工具管线，已有 hooks 可以拦截它。协议不会替你决定业务审批规则；那些规则仍由应用/extension 实现。

### 路径 B：模型调用 codemode，由脚本调用 MCP 工具

当前 MCP 默认 exposure 是 `codemode`。服务器工具不会全部直接声明给模型，也不会全部塞进 codemode description。模型知道服务器摘要，脚本用 `searchTools()` 等能力找到所需声明。

示意脚本（必须先发现真实工具名与参数；不要直接复制假定的 schema）：

```js
// 第一次：发现能力，模型读到声明后再写调用脚本
text(await searchTools("current weather", { namespace: "weather" }));
```

```js
// 下一次：示意真实 server 恰好有这个名称及参数
const results = await Promise.allSettled([
  tools.mcp__weather__get_weather({ city: "Beijing" }),
  tools.mcp__weather__get_weather({ city: "Shanghai" }),
]);
for (const result of results) {
  if (result.status === "fulfilled") {
    const value = result.value;
    text(value.isError ? { error: value.content } : value.structuredContent ?? value.content);
  } else {
    text({ error: String(result.reason) });
  }
}
```

流程：模型产生 codemode 脚本 → QuickJS sandbox → Pi 的嵌套工具管线 → MCP client/server → 脚本整理结果 → codemode 输出进入对话 → 下一次模型回答。

脚本没有 Node、文件系统或网络 API；外部动作通过工具执行。嵌套调用仍经过校验和 hooks，并用 `parentToolCallId` 关联事件。MCP 的 `isError` 结果可能正常 resolve，脚本必须检查它，不能只依赖 `catch`。脚本失败也不会撤销已完成的外部调用。

Codemode 同样可以调用 `read`、`bash` 等非 MCP 工具；它不是 MCP 专用运行器。2026-10-12 的 [`24382be`](https://github.com/earendil-works/pi/commit/24382bedd03c61c0994628907a017b4491f8fa76) 进一步移除了 codemode 的 MCP 专用知识。

## 5. exposure：注册一个工具，不等于让模型直接看到它

以下是 **MCP server 配置**的四种模式，不要与通用 `ToolExposure` 的全部选项混用。

| MCP exposure | 模型怎样使用 | 适用学习例子 |
|---|---|---|
| `direct` | 立即看到工具定义，直接产生 tool call；也可从 codemode 调用 | 少量常用工具，最容易跟踪 loop |
| `codemode`（默认） | 脚本按需发现工具并调用；自动启用 codemode（除非关闭该设置） | 很多工具、并行查询、过滤大结果 |
| `deferred` | `tool_search` 加载工具声明后，模型直接调用；不要求 codemode | 工具很多，但希望保留普通 tool-call 方式 |
| `hidden` | 不可达；可配合单工具 override 开放少量能力 | 学习如何控制可用工具集合 |

`codemode-deferred` 现在只是 `codemode` 的兼容别名。MCP 的 `codemode` 配置在通用工具层映射为 `deferred`；MCP extension 再决定激活 codemode 还是 tool_search。因此读源码时要区分配置层与工具层。

先用一个只读工具、`direct` exposure 理解完整调用，再试 `codemode`。没有外部服务需求时，继续使用本地工具/CLI/Skills 即可。

## 6. 什么是真正的 breaking change？

MCP 的加入主要是新增能力和产品方向变化，不能把“加入 MCP”本身等同于“重写了 loop”。下面这些才有明确的迁移影响：

| 变化 | 以前的假设 | 现在该怎么做 |
|---|---|---|
| v0.84.4 · 下一轮准备 | 每个 turn 结束都调用 `prepareNextTurn` | 最终 turn 不调用；run 收尾监听 `agent_end` |
| v0.87.0 · 停止 hook | `shouldStopAfterTurn` 返回 boolean | 使用 `finishTurn` 返回 `{ action: "end" }`；保留 error/aborted 的 hard-exit 语义 |
| v1.0.0 · 删除旧 Harness exports | 从 `pi-agent-core` 导入 `AgentHarness` / `./harness/*` / `./node` | 阅读并按 `pi-durable` 当前 API 重写；不能只替换包名 |
| MCP adapter 共存 | 已装社区 `/mcp` extension，又添加内置 `mcp.json` | 注册 `/mcp` 的 extension 会替代内置 session MCP；shell `pi mcp` 仍用内置实现，不能假定两套配置同时工作 |
| SDK session | CLI 内置 MCP，所以 SDK 也自动连接 | SDK 不自动加载内置 extensions；显式加入 MCP，以及 exposure 所需 codemode/tool-search extension |
| v1.2.0 · 资源状态 JSON | `pi mcp list --json` 中 `resources` / `resourceTemplates` 是数量 | `resources: true` 表示支持资源；按需列举/读取，更新依赖旧计数的脚本 |

停止 hook 迁移示例：

```ts
// 旧：shouldStopAfterTurn: async (turn, signal) => shouldStop(turn, signal)
finishTurn: async (turn, signal) => {
  if (turn.message.stopReason === "error" || turn.message.stopReason === "aborted") return;
  return (await shouldStop(turn, signal)) ? { action: "end" } : undefined;
}
```

`finishTurn` 在 assistant/tool results 完成之后、`turn_end` 之前运行；正常 turn 的决定在 `turn_end` 后应用。`prepareRequest` 在每次 provider 请求前运行（含第一次）；`prepareNextTurn` 仅在确实要继续时运行。三者不是同一个 hook。

## 7. Durable 这条线：旧词与当前代码怎么对应？

旧资料中的 `Entry / Register / Usage ledger / Lane / Operation` 解释的是当时的运行时。当前独立 `pi-durable` 的入口是：

`Harness.open(storage, options, context)` → `root()` 获取 Conversation → `submit(input)` 持久化接收输入 → `pi.generation` task → `pi.tool` tasks → 后续 generation → Submission 完成。

| 旧阅读重点 | 当前阅读重点 | 注意 |
|---|---|---|
| Entry | 不可变 transcript Entry | 不必假设 entry 种类、格式完全兼容 |
| Registers / operation state | Document + Task checkpoint | 如 `pi.agent`、`pi.live`、`pi.inbox`、`pi.usage`；旧 `op.state/*` 不是当前教程入口 |
| Lane / Operation | Conversation / Submission / Task | 是帮助理解职责的类比，不是一对一 API 重命名 |
| Effect sandwich | 工具 intent 先 commit，再执行，再保存 outcome/checkpoint | 外部成功但结果未 commit 的不确定窗口仍存在 |

使用 SQLite/JSONL 等持久存储才能跨进程恢复；`MemoryStorage` 不会在进程退出后保留状态。中途崩溃的工具只有声明 `replay: "safe"` 才允许自动重跑，否则给模型 interrupted 结果；有外部副作用的工具仍需要自己的幂等性设计。

**普通 Pi CLI 仍是 `AgentSession + Agent + agent-loop`；独立 `pi-durable` 是可执行但 API 仍实验性的另一条路径。** MCP 的加入不证明 CLI 已接管这个 durable runtime。也不要再沿当前 main 的旧 `packages/agent/src/harness/`、Pico v3 或 `experimental/mini` 链接读代码。

## 8. 你下一次应该按这个顺序读

所有链接固定到本次核对快照，避免未来 main 又变化。

1. [agent-loop.ts](https://github.com/earendil-works/pi/blob/60ac3bcc3b8d6df34a44c0ed677a9eb9c74e52ac/packages/agent/src/agent-loop.ts)：找到 `runLoop`、`finishTurn`、`prepareNextTurn`。回答：MCP 结果回来后是谁决定下一轮？
2. [MCP extension index.ts](https://github.com/earendil-works/pi/blob/60ac3bcc3b8d6df34a44c0ed677a9eb9c74e52ac/packages/coding-agent/src/extensions/mcp/index.ts) 和 [tools.ts](https://github.com/earendil-works/pi/blob/60ac3bcc3b8d6df34a44c0ed677a9eb9c74e52ac/packages/coding-agent/src/extensions/mcp/tools.ts)：跟 `createMcpToolDefinition → callTool → convertMcpResult`。
3. [MCP client](https://github.com/earendil-works/pi/blob/60ac3bcc3b8d6df34a44c0ed677a9eb9c74e52ac/packages/mcp/src/client.ts)：分清连接、`listTools` 和实际 `callTool`。
4. [codemode tool.ts](https://github.com/earendil-works/pi/blob/60ac3bcc3b8d6df34a44c0ed677a9eb9c74e52ac/packages/coding-agent/src/extensions/codemode/tool.ts) 与 [codemode 文档](https://github.com/earendil-works/pi/blob/60ac3bcc3b8d6df34a44c0ed677a9eb9c74e52ac/packages/coding-agent/docs/codemode.md)：跟一次脚本如何进入嵌套工具管线。
5. 最后读 [pi-durable README](https://github.com/earendil-works/pi/blob/60ac3bcc3b8d6df34a44c0ed677a9eb9c74e52ac/packages/durable/README.md)、[harness.ts](https://github.com/earendil-works/pi/blob/60ac3bcc3b8d6df34a44c0ed677a9eb9c74e52ac/packages/durable/src/harness/harness.ts)、[generation.ts](https://github.com/earendil-works/pi/blob/60ac3bcc3b8d6df34a44c0ed677a9eb9c74e52ac/packages/durable/src/harness/generation.ts)、[tool.ts](https://github.com/earendil-works/pi/blob/60ac3bcc3b8d6df34a44c0ed677a9eb9c74e52ac/packages/durable/src/harness/tool.ts)。回答：工具已经对外成功，但结果未 commit 时，重启会怎样？

### 核对依据

- [旧 README（2026-09-11 快照）](https://github.com/earendil-works/pi/blob/71dca871bc80b6bc97be37f0ca3189399d651fff/packages/coding-agent/README.md)：当时的 No MCP 立场。
- [MCP / codemode 引入提交](https://github.com/earendil-works/pi/commit/8562bcf66a8eeefdf75ddd231ca9d7aa7d64f86e) 与 [PR #10040](https://github.com/earendil-works/pi/pull/10040)：实现与动机。
- [agent-core changelog](https://github.com/earendil-works/pi/blob/60ac3bcc3b8d6df34a44c0ed677a9eb9c74e52ac/packages/agent/CHANGELOG.md)：loop hooks、旧 Harness exports 删除。
- [coding-agent changelog](https://github.com/earendil-works/pi/blob/60ac3bcc3b8d6df34a44c0ed677a9eb9c74e52ac/packages/coding-agent/CHANGELOG.md)：v0.99.0、v1.2.0 与后续兼容行为。
- [MCP 文档](https://github.com/earendil-works/pi/blob/60ac3bcc3b8d6df34a44c0ed677a9eb9c74e52ac/packages/coding-agent/docs/mcp.md)：配置、exposure、extension 替换、SDK。

本次只更新学习资料与教程网页；不会自动把你的 fork 源码升级、执行 MCP 安装命令或迁移 session 数据。fork 中旧源码若与本章不同，请用以上固定上游链接对照。
