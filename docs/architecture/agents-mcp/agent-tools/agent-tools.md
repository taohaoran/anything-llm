# Agent 工具集（agent-tools）

> 本文是 `agents-mcp` 域下的叶子子系统文档。域级总览见 `../agents-mcp.md`。
> 本文展开 **aibitat 内置工具插件的注册与执行**，不重复展开运行时对话循环（见 `../agent-runtime/agent-runtime.md`）。
>
> 源码基准：anything-llm v1.16.2，commit `128a015`，纯 JavaScript ESM。

## 1. 功能清单

工具以“插件”形式实现：每个插件导出 `{ name, startupConfig, plugin() }`，`plugin()` 返回对象，其 `setup(aibitat)` 内调用 `aibitat.function({...})` 把工具登记进会话的 `functions` Map（注册机制见 `agent-runtime` 叶子）。

| 工具组 | 能力 | 源码路径 |
|---|---|---|
| 网页搜索 | `web-browsing`：按系统设置选 Google CSE / Brave / 等引擎实时搜索 | `aibitat/plugins/web-browsing.js:6`（`search` 在 `:66`） |
| 网页抓取 | `web-scraping`：抓取指定 URL 正文 | `aibitat/plugins/web-scraping.js` |
| RAG 记忆检索 | `rag-memory`：对工作区文档/向量库 search，或 store 长期记忆；带 Deduplicator | `aibitat/plugins/memory.js:7` |
| 文件系统 | `filesystem`：read/edit/list/search/copy-file 等子工具 | `aibitat/plugins/filesystem/lib.js`（787 行）、`read-text-file.js` 等 |
| 创建文件 | `create-files`：把内容落成文件 | `aibitat/plugins/create-files/index.js` |
| 图像生成 | `generate-image`：调图像生成模型 | `aibitat/plugins/generate-image.js` |
| 图表 | `rechart`：生成图表 | `aibitat/plugins/rechart.js` |
| SQL Agent | `sql-agent`：对数据库执行只读查询 | `aibitat/plugins/sql-agent/index.js` |
| 计划任务 | `create-scheduled-job`：创建带完整 agent 能力的 cron 任务 | `aibitat/plugins/create-scheduled-job/index.js` |
| 用户追问 | `request-user-input`：向用户提澄清问题（问卷） | `aibitat/plugins/request-user-input.js` |
| 文档摘要 | `docSummarizer`：摘要长文档 | `aibitat/plugins/summarize.js` |
| 邮箱/日历 | `gmail`、`outlook`、`google-calendar`：第三方账户集成 | 各 `*/index.js`、`*/lib.js` |
| 运行时插件 | `websocket`（流式回前端）、`chat-history`（落库）、`http-socket`、`cli`、`model-router-cooldown`、`router-classifier` | `aibitat/plugins/` |
| 插件目录聚合 | 汇总全部插件并按 slug 别名导出 | `aibitat/plugins/index.js:19` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| 插件对象 `{ name, startupConfig, plugin() }` | 各 `plugins/*.js` | 每个工具插件的统一形态；`startupConfig.params` 声明所需启动参数 |
| `aibitat.function(functionConfig)` | `aibitat/index.js:1546` | 登记工具；`functionConfig = { name, description, examples, parameters(JSON Schema), handler(args), super, controller?, isMCPTool? }` |
| `handler(args)` | 各插件内 | LLM 决定调用后实际执行的 async 函数，返回字符串结果回灌给模型 |
| `Deduplicator` | `aibitat/utils/dedupe.js` | rag-memory 工具去重同一查询 |
| `ToolReranker` | `aibitat/utils/toolReranker.js` | 工具级重排，超出 topN 时裁剪工具列表 |

## 3. 关键调用链

**链 1：工具挂载**

1. `AgentHandler.#attachPlugins()`（`agents/index.js`）按工作区配置挑选需要的插件，对每个插件 `aibitat.use(plugin.plugin(runtimeArgs))`。
2. 插件 `setup(aibitat)` 被回调（`aibitat/index.js:168` 的 `use`），内部 `aibitat.function({...})` 把工具写入 `functions` Map（`aibitat/index.js:1547`）。
3. 以 `web-browsing` 为例：`setup` 里登记 `name:"web-browsing"`、JSON Schema 参数 `{query:string}`、`handler` 内调 `this.search(query)`（`web-browsing.js:15`、`:52`、`:66`）。

**链 2：工具被 LLM 调用**

1. `reply()` 把当前 agent 可用工具 `functions` 取齐（`aibitat/index.js:921`），可选 `ToolReranker.rerank(userPrompt, functions)`（`:926`）裁剪。
2. provider 返回 `functionCall{name, arguments}` 后，`handleExecution` 从 `functions.get(name)` 取 `fn`，设 `fn.caller`，`await fn.handler(args)`（`aibitat/index.js:1260-1272`）。
3. 结果作为 `{role:"function"}` 回灌递归；期间工具可 `addToolAttachment` 注入图片、`addCitation` 累积引用，最终随响应冲刷。

**链 3：工具失败自恢复**

- 工具内部 `handler` 普遍 try/catch，把错误转成自然语言返回（如 `web-browsing.js:56` 返回 `There was an error ... Let the user know: ...`），让模型继续而非崩溃。
- 工具未注册时回灌 `"Function not found. Try again."` 让模型纠正（`aibitat/index.js:1242`）。

## 4. 配置项

| 配置 / flag | 默认 / 行为 | 位置 |
|---|---|---|
| `agent_search_provider` | 决定 web-browsing 用哪个搜索引擎（默认 unknown） | `web-browsing.js:68` 读 `SystemSettings` |
| `AGENT_MAX_TOOL_CALLS` | 单轮链式工具调用上限，默认 10 | `aibitat/index.js:88` |
| 智能技能选择（ToolReranker） | 工具数超阈值时按 prompt 重排裁剪，省最多 80% token | `aibitat/index.js:926` |
| 各插件 `startupConfig.params` | 声明挂载时需要的运行参数（如 socket、userId） | 各插件 `startupConfig` |
| 工具中途开关 | `aibitat.toggleAgentTool` / `removeFunction(name)` 运行时启停工具 | `aibitat/index.js:1557`、`agents/index.js:872` |

## 5. 错误与重试语义

- 工具 `handler` 内部捕获异常并转为文本错误返回，模型据此向用户说明，不抛出崩溃。
- 工具未注册（`!fn`）→ 回灌错误消息后继续递归（`aibitat/index.js:1242`）。
- 达 `maxToolCalls` 后清空工具列表，强制生成最终回复（`aibitat/index.js:1231`）。
- 无工具级自动重试；重试靠模型在下一轮重新决策。

## 6. 并发细节

- 工具 `handler` 在上层 `handleExecution` 递归链中**串行 await**，一次只执行一个工具。
- 工具可通过 `addToolAttachment`/`addCitation`/`addToolAttachment` 写入会话级缓冲（`aibitat/index.js:293`、`:216`），由响应定稿时统一冲刷，避免并发写入竞争。
- MCP 工具每个实例自带 `AbortController`（`mcp-bridge` 叶子），可单独中断。
- Node 单线程事件循环；耗时工具（网页抓取、SQL）内部用 async I/O，不阻塞事件循环。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `server/utils/agents/aibitat/plugins/` 全部内置工具插件及其子工具
- 工具注册机制 `aibitat.function()`（运行时侧）

**Out-of-Scope**
- 各工具依赖的外部服务（Google CSE、Brave、Gmail/Outlook/Google Calendar API、图像生成模型）→ 外部系统，不在本仓库源码内
- MCP 外部 server 提供的工具 → `mcp-bridge` 叶子
- Agent Flows 作为“工具”被 LLM 调用 → `agent-flows-engine` 叶子
- 向量库检索实现 → ai-integrations / document-pipeline 域

## 8. 与相邻子系统交互

- 上游：`agent-runtime`（AIbitat）在每轮 `reply` 把 `functions` 传给 provider，并在 `functionCall` 时回调工具 `handler`。
- 下游：工具 → 外部 API（搜索/邮件/日历/图像）、向量库、文件系统、DocumentManager。
- 侧向：rag-memory 工具 → ai-integrations 向量检索；create-scheduled-job → BackgroundWorkers 域。

## 9. 语言专项适配口径（TS/JS 口径）

- **capability seam 分组**：按 Service Definition（`aibitat.function()` 工具注册接口）/ Provider（各插件 `handler` 实现）/ Consumer（AIbitat 运行时按 `functionCall` 调用 handler）分组。JS 无类型，工具 schema 以 JSON Schema（`parameters`）声明给 LLM。
- **图类型**：architecture（工具注册拓扑）+ sequence（一次工具调用往返时序，典型 JS 异步/事件链）。
- **外部边界**：外部搜索/邮箱/日历 API、文件系统、向量库。
- **并发**：async 串行递归 + 会话级缓冲，无锁。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 工具注册与执行拓扑 | `agent-tools-architecture.html` | architecture | standard |
| 工具调用执行时序 | `agent-tools-sequence.html` | sequence | showcase |

- 两图 render 退出码 0、HTML 非空。
- **降档披露**：架构图目标 showcase 落 standard。失败检查为 `clean-flow/endpoint-side-direction`（底部工具组向上连 registry 的斜边）与短横边标签重叠；采取的修复：删除工具组到 registry 的三条向上边（工具组已用 region 分组表达）、去掉过短横边上的文字标签。sequence 图一次通过 showcase。
- **省略 dataflow/lifecycle/workflow**：本叶子无数据管道（数据流动已并入 sequence）、无单实体状态机（状态机属 agent-runtime）、无多角色审批流程，按资源节省原则省略。
- JSON IR 位于 `json/`。
