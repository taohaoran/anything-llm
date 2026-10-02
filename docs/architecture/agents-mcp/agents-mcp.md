# agents-mcp 域总览

> 本域覆盖 anything-llm 的 **Agent（aibitat）运行时生态**：从对话循环内核、内置工具、Agent Flows 工作流，到 MCP 外部工具桥接、长期记忆与外部渠道（telegram / 后台队列 / 推送）。
>
> 源码基准：anything-llm v1.16.2，commit `128a015`，纯 JavaScript ESM。

## 域职责

本域解决“如何让 LLM 不只是聊天，而是能调用工具、执行多步工作流、连接外部 MCP server、并通过 telegram 等渠道触达用户”的问题。核心是 `aibitat` 这张“agent 对话图”：它把 LLM provider、工具函数 Map、事件总线组织成一个可递归推进的对话循环。

## 叶子索引

| 叶子 | 职责 | 文档 | 图 |
|---|---|---|---|
| agent-runtime | AIbitat 会话图、递归对话循环、工具分发、思考链、中断/重试 | [agent-runtime.md](agent-runtime/agent-runtime.md) | [架构](agent-runtime/agent-runtime-architecture.html) · [状态机](agent-runtime/agent-runtime-lifecycle.html) |
| agent-tools | 内置工具插件（网页搜索/RAG/文件/图像/SQL/计划任务）注册与执行 | [agent-tools.md](agent-tools/agent-tools.md) | [架构](agent-tools/agent-tools-architecture.html) · [时序](agent-tools/agent-tools-sequence.html) |
| agent-flows-engine | Agent Flows 线性步骤流水线（变量替换、directOutput 短路） | [agent-flows-engine.md](agent-flows-engine/agent-flows-engine.md) | [架构](agent-flows-engine/agent-flows-engine-architecture.html) · [数据流](agent-flows-engine/agent-flows-engine-dataflow.html) |
| mcp-bridge | MCP 客户端桥接：管理外部 MCP server、把其工具登记进 aibitat | [mcp-bridge.md](mcp-bridge/mcp-bridge.md) | [架构](mcp-bridge/mcp-bridge-architecture.html) · [时序](mcp-bridge/mcp-bridge-sequence.html) |
| memory-external-channels | 长期记忆、BackgroundWorkers 队列、PushNotifications、telegramBot | [memory-external-channels.md](memory-external-channels/memory-external-channels.md) | [架构](memory-external-channels/memory-external-channels-architecture.html) · [时序](memory-external-channels/memory-external-channels-sequence.html) |

## 域级机制细节

- **统一插件模型**：所有能力（内置工具、MCP 工具、chat-history、websocket、Agent Flows）都通过 `aibitat.use(plugin)` 挂载，插件 `setup(aibitat)` 内用 `aibitat.function({...})` 登记工具，统一进入 `functions` Map 供 LLM functionCall。
- **递归工具循环**：一轮 `reply` 内，provider 返回 functionCall → 执行工具 → 结果回灌 → 再推理，直到无 functionCall 或达 `maxToolCalls`（默认 10）。
- **取消传播**：会话级 `AbortController`，`abort()` 中止在途 provider 请求并让递归循环提前退出。
- **重要事实校正**：Agent Flows 是**线性步骤数组**，不是有向图/边/条件分支（详见该叶子第 0 节）。
