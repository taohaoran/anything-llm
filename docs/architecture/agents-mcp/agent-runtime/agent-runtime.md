# Agent 运行时（agent-runtime）

> 本文是 `agents-mcp` 域下的叶子子系统文档。域级总览见 `../agents-mcp.md`。
> 本文展开 **AIbitat 对话图运行时**（会话状态机、递归对话循环、工具分发、思考链），不重复展开
> 内置工具集（见 `../agent-tools/agent-tools.md`）、Agent Flows 引擎（见 `../agent-flows-engine/agent-flows-engine.md`）、
> MCP 桥接（见 `../mcp-bridge/mcp-bridge.md`）。
>
> 源码基准：anything-llm v1.16.2，commit `128a015`，主语言为纯 JavaScript ESM。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| AIbitat 会话类 | 管理 agent 之间对话图的核心类，持有 agents/channels/functions 三张 Map 与聊天历史 | `server/utils/agents/aibitat/index.js:15`（`class AIbitat`） |
| 递归对话循环 | `chat(route)` 递归驱动节点间来回对话，按 reply 文本决定 terminate/interrupt/继续 | `aibitat/index.js:604`（`async chat`） |
| 节点回复 | `reply(route)` 组装 system+history 消息、选择 provider、路由模型、决定流式/同步执行 | `aibitat/index.js:888`（`async reply`） |
| 工具调用循环 | `handleExecution`/`handleAsyncExecution` 递归执行 `functionCall`，结果回灌消息后再次推理 | `aibitat/index.js:1196`、`:1032` |
| 组（channel）路由 | channel（群）内用 `selectNext` 选下一个发言节点，按轮次上限终止 | `aibitat/index.js:716`（`selectNext`）、`:610` |
| 中断/续聊/重试 | `interrupt` 暂停等用户输入、`continue(feedback)` 恢复、`retry()` 重试失败轮 | `aibitat/index.js:473`、`:1348`、`:1390` |
| 中止传播 | `_aborted` 标志 + 会话级 `AbortController`，在循环边界与 provider 调用间传播取消 | `aibitat/index.js:41`、`:430`（`abort`） |
| 插件挂载 | `use(plugin)` 注册插件，插件 `setup(aibitat)` 内调 `aibitat.function()` 登记工具 | `aibitat/index.js:168`、`:1546` |
| 引用/附件缓冲 | 工具执行期间缓冲 citations、工具图片附件、澄清问卷，在响应定稿时冲刷 | `aibitat/index.js:64`、`:72`、`:81` |
| Provider 工厂 | `getProviderForConfig` 按 provider 名实例化对应 LLM provider | `aibitat/index.js:1437` |
| AgentHandler 工厂 | 校验调用、组装 AIbitat、挂载 websocket/chat-history 插件、加载自定义 agent 与插件 | `server/utils/agents/index.js:22`（`class AgentHandler`）、`:849`（`createAIbitat`） |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `AIbitat` | `aibitat/index.js:15` | 会话主体；事件总线 `emitter`（EventEmitter），暴露 onStart/onMessage/onError/onToolCallResult/onAbort/onTerminate/onInterrupt 钩子 |
| `AgentHandler` | `agents/index.js:22` | 单次 agent 调用的工厂与上下文容器（invocation、provider、模型路由、解析文件上下文注入） |
| `AgentProviderInstance`（JSDoc typedef） | `aibitat/providers/ai-provider.js` | provider 能力接口：`complete(messages, functions)`、`stream(...)`、`supportsAgentStreaming`、`isModelLoaded()`、`resetCumulativeUsage()` |
| `FunctionConfig`（插件工具） | `aibitat/index.js:1546` | `{ name, description, parameters(JSON Schema), handler(args), caller }`，存入 `functions` Map |
| `APIError` | `aibitat/error.js` | provider 错误包装，`chat`/`reply` 捕获后转 `newError` 状态 |
| `ToolReranker` | `aibitat/utils/toolReranker.js` | 智能技能选择：按 user prompt 对候选工具重排裁剪 |
| `defaultMaxToolCalls()` | `aibitat/index.js:87` | 读 `AGENT_MAX_TOOL_CALLS`，默认单轮最多链式调用 10 次工具 |

## 3. 关键调用链

**链 1：一次 agent 提问的完整驱动**

1. `AgentHandler.createAIbitat()`（`agents/index.js:855`）`new AIbitat({provider, model, chats})`，挂 websocket、chat-history 插件，`#loadAgents()` + `#attachPlugins()`。
2. `startAgentCluster()`（`agents/index.js:949`）调 `aibitat.start({from: USER_AGENT, to: channel, content})`。
3. `start()`（`aibitat/index.js:584`）登记消息后 `chat({to, from})`。
4. `chat()`（`aibitat/index.js:604`）：若目标是 channel 则 `selectNext` 选节点并递归；否则 `reply(route)` 取回复文本。
5. `reply()`（`aibitat/index.js:888`）：取 agent 配置、格式化历史、可选 `ToolReranker.rerank`、按 `resolveRoute` 重选模型、`getProviderForConfig` 取 provider，按 `supportsAgentStreaming` 分派 `handleAsyncExecution` 或 `handleExecution`。
6. 返回文本后 `chat` 依据是否等于 `"TERMINATE"`/`"INTERRUPT"`、是否达 `maxRounds`，决定 `terminate`、`interrupt` 或继续 `chat(newChat)` 递归（`aibitat/index.js:673-694`）。

**链 2：工具调用递归（思考链）**

1. `handleExecution`（`aibitat/index.js:1220`）`providerInstance.complete(messages, functions)`。
2. 若返回 `completion.functionCall`（`aibitat/index.js:1228`）：从 `functions.get(name)` 取工具；未找到则回灌“Function not found”再递归；达 `maxToolCalls` 后清空 functions 列表（`:1231`）。
3. `await fn.handler(args)` 执行工具（`:1272`），发 `toolCallResult` 事件与 telemetry。
4. 工具结果作为 `{role:"function"}` 消息回灌，连同可能的图片附件（`:1309`）递归 `handleExecution(..., depth+1)`。
5. 无 `functionCall` 时冲刷 usage/citations 并返回 `textResponse`（`:1329-1336`）。

**链 3：中断-续聊**

`reply` 返回 `"INTERRUPT"` 或 agent 配置 `interrupt==="ALWAYS"` 时 `interrupt(newChat)`（`aibitat/index.js:687`）落一条 `state:"interrupt"` 的 chat；用户后续 `continue(feedback)`（`:1348`）弹出该条、把反馈作为新 user 消息继续 `chat`；失败轮可用 `retry()`（`:1390`）重发上一条。

## 4. 配置项

| 配置 / flag | 默认 / 行为 | 位置 |
|---|---|---|
| `maxRounds` | 默认 100，整次对话最大轮数 | `aibitat/index.js:109` |
| `maxToolCalls` | 默认 `AIbitat.defaultMaxToolCalls()`，读 `AGENT_MAX_TOOL_CALLS`，缺省 10 | `aibitat/index.js:87-92` |
| `interrupt` | 默认 `"NEVER"`；`"ALWAYS"` 让该节点每轮都中断等人确认 | `aibitat/index.js:108`、`:703` |
| `resolveRoute` 回调 | 挂载后每轮 `reply` 前重评模型路由（动态模型路由） | `aibitat/index.js:946`、`agents/index.js:880` |
| `fetchParsedFileContext` 回调 | 每轮把上传解析文件 + pinned docs 注入最后一条 user 消息 | `aibitat/index.js:896`、`agents/index.js:787` |
| `skipHandleExecution` | 流执行结果直接返回、不再链式工具调用（Agent Flows directOutput 用） | `aibitat/index.js:29`、`:1280` |
| `ToolReranker` 启用 | 工具数 > `defaultTopN` 时按 prompt 重排裁剪，省 token | `aibitat/index.js:926` |

## 5. 错误与重试语义

- provider 调用经 `#safeProviderCall`（`aibitat/index.js:994`）包装：用户主动 abort 时原样抛出；否则包成 `APIError("The agent model failed to respond: ...")`。
- `chat`/`reply` 捕获 `APIError` 后走 `newError(route, error)`（`:558`）落一条 `state:"error"` 的 chat，**不会崩溃进程**；用户可 `retry()` 重发。
- 工具未注册（`!fn`）不抛异常，而是回灌 `Function "<name>" not found. Try again.` 让 LLM 自我纠正（`:1242`）。
- 本地推理服务器加载模型慢时 `#reportModelLoading`（`:1011`）先推一条“Loading model…”的 introspect 再请求。
- 无自动指数退避重试；重试由 `retry()`（用户/上层触发）或 `maxToolCalls` 兜底（达上限后不再追加工具、强制生成最终回复）。

## 6. 并发细节

- 单会话内 `chat` 是**递归 async 调用链**，同一时刻只有一条推理链在跑；工具 `handler` 内部可并发，但外层串行回灌。
- 取消传播：会话级 `abortController = new AbortController()`（`:48`），`abort()`（`:430`）置 `_aborted=true` 并 abort 所有在途 provider 请求；循环边界（`chat:605`、`handleExecution:1204`）与流式返回后（`:671`、`:1226`）都会检查 `_aborted` 提前退出，避免在 socket 关闭后悬挂 feedback 超时。
- `setMaxListeners(0, abortController.signal)`（`:130`）解除 EventEmitter 默认 11 监听器上限，因为每个 provider 请求都注册一次 abort 监听。
- 无锁/无临界区：`functions`、`agents`、`channels` 均为 Map，会话内单线程访问；`_pendingCitations`/`_toolAttachments` 为串行缓冲，工具执行后立即冲刷。
- 多会话之间彼此独立（每个 `AgentHandler` 自建 `AIbitat`），无线程共享状态。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `server/utils/agents/aibitat/`：会话图、对话循环、工具分发、事件钩子、provider 工厂入口
- `server/utils/agents/index.js`：`AgentHandler` 工厂、插件挂载编排
- `aibitat/providers/`：各 LLM provider 的 `complete/stream` 实现（provider 选择与封装属 ai-integrations 域）

**Out-of-Scope（不在本仓库源码内 / 相邻叶子）**
- 具体内置工具（网页搜索、代码执行、文件操作等）的实现 → `agent-tools` 叶子
- Agent Flows 的步骤编排 → `agent-flows-engine` 叶子
- 外部 MCP server 进程与协议 → `mcp-bridge` 叶子
- 具体 LLM 服务（OpenAI/Anthropic/Ollama 等）的 HTTP API → 外部系统，不在本仓库源码内
- 前端 WebSocket 协议帧格式 → frontend 域

## 8. 与相邻子系统交互

- 上游：API 层（workspace-chat-api）构造 `AgentHandler` → 调用 `createAIbitat` + `startAgentCluster`；前端 WebSocket 经 `websocket` 插件接收流式/introspect 事件。
- 下游：`AIbitat` → LLM provider（ai-integrations 域）；工具 `handler` → 各内置工具插件（`agent-tools`）；chat-history 插件 → Prisma 落 `workspace_chats`。
- 侧向：`resolveRoute` → 动态模型路由；`fetchParsedFileContext` → DocumentManager / WorkspaceParsedFiles（document-pipeline 域）。

## 9. 语言专项适配口径（TS/JS 口径）

- **capability seam 分组**：按 Service Definition（`AIbitat` 会话图 + provider 接口）/ Provider（`aibitat/providers/*` 各 LLM 适配）/ Consumer（各插件 `setup(aibitat)` 注册工具）分组。JS 无类型标注，以 `aibitat.function()` 导出与 JSDoc `@typedef AgentProviderInstance` 作为接口线索。
- **图类型**：以 architecture（组件拓扑）+ lifecycle（agent 对话循环状态机）为主；事件驱动（EventEmitter `emitter`、`onMessage`/`onToolCallResult` 回调、流式 `stream`）属典型 JS 异步热点，本叶子以文字化调用链 + lifecycle 表达，未单独出 sequence 图（见第 10 节省略原因）。
- **外部边界**：LLM provider HTTP API（外部）、Node 内置 `events`/`AbortController`、前端 WebSocket。
- **并发**：无 goroutine，用 async 递归 + AbortController 取消传播 + EventEmitter 事件，符合 Node 单线程事件循环模型。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 运行时组件拓扑 | `agent-runtime-architecture.html` | architecture | standard |
| 对话工具循环状态机 | `agent-runtime-lifecycle.html` | lifecycle | standard |

- 两图均 render 退出码 0、HTML 非空（约 800KB 自包含交互式）。
- **降档披露**：两图目标 showcase，实际落 standard。架构图失败检查为 `clean-flow/endpoint-side-direction` 与垂直连线标签重叠，采取的修复：改为两行式布局、垂直边加 `labelDy`、跨层边用 `orthogonal-v`。lifecycle 失败检查为 `clean-flow/edge-through-node`（long forward edge 穿越中间节点）与短边标签重叠，采取的修复：改用 `main` lane 相位轨、把工具循环放到副 lane、去掉回环边文字标签（循环语义在本文第 3 节文字描述）。
- **省略 sequence 图**：本单元未生成 sequence 图——工具调用往返的时序语义已由 lifecycle 状态机 + 第 3 节逐步调用链文字完整表达，再画 sequence 会与二者信息重复，按资源节省原则省略。中断/error 分支同样在第 3、5 节文字说明，未画入状态图（避免状态图边交叉）。
- JSON IR 源文件位于 `json/`。
