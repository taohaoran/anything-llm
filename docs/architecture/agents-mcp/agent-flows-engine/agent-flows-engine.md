# Agent Flows 引擎（agent-flows-engine）

> 本文是 `agents-mcp` 域下的叶子子系统文档。域级总览见 `../agents-mcp.md`。
>
> 源码基准：anything-llm v1.16.2，commit `128a015`，纯 JavaScript ESM。

## 0. 重要事实校正（与常见假设的差异）

任务设想 Agent Flows 是“多步骤工作流编排、节点与边、条件分支”。**实读源码后确认：本仓库的 Agent Flows 是一个线性步骤数组流水线，不是有向图（无节点-边拓扑、无运行时条件分支）。**

- 流程定义存为 `flowsDir/<slug>/flow.json`，其核心是 `config.steps: []`（一个有序数组）。
- `FlowExecutor.executeFlow` 用 `for (let i = 0; i < steps.length; i++)` **按数组下标顺序**依次执行，没有 DAG、没有边、没有网关节点。
- “分支”能力实际由两处实现：① `llm-instruction` 步骤把数据交给 LLM 做变换/决策；② 任一步骤 `directOutput: true` 时立即短路返回、不再执行后续步骤。除此之外没有显式 if/else 节点。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 流程 CRUD | `AgentFlows` 静态类：list/get/create/update/delete，按 slug 管理目录与 JSON | `server/utils/agentFlows/index.js` |
| 流程加载 | `loadFlow(slug)` 读 `flowsDir/<slug>/flow.json` 并 `JSON.parse` | `agentFlows/index.js` |
| 顺序执行器 | `FlowExecutor.executeFlow(flow, inputs, context)` 线性跑 steps | `agentFlows/executor.js` |
| 变量替换 | 步骤 prompt/参数里的 `${varName}` 用累积 variables 替换 | `executor.js`（`replaceVariables`） |
| start 块初始化 | 第一步类型必须为 `start`，把 inputs 写入 variables | `executor.js` |
| apiCall 步骤 | 发 HTTP 请求，响应体存入变量 | `executors/api-call.js` |
| llmInstruction 步骤 | 用当前 provider/模型对 variables 做指令式处理，结果存变量 | `executors/llm-instruction.js` |
| webScraping 步骤 | 抓取 URL 内容存入变量 | `executors/web-scraping.js` |
| 直出短路 | 步骤 `directOutput:true` 时跳过后续步骤直接返回 | `executor.js` |
| 流程类型注册 | `FlowTypes` 枚举允许的步骤类型 | `agentFlows/flowTypes.js` |
| 作为 Agent 工具 | 流程可登记为 aibitat 工具被 LLM 触发 | `agents/index.js` 挂载处 |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `AgentFlows`（静态类） | `agentFlows/index.js` | 流程定义的持久化（文件系统 CRUD） |
| `FlowExecutor` | `agentFlows/executor.js` | 执行一次流程实例，维护 `variables` 累积对象 |
| `FlowTypes` | `agentFlows/flowTypes.js` | 步骤类型枚举：start/apiCall/llmInstruction/webScraping 等 |
| 步骤执行器 | `executors/*.js` | 每种 step type 一个实现，接收 `{step, variables, context}` 返回字符串结果 |
| `replaceVariables` | `executor.js` | 把字符串里 `${var}` 替换为 variables 中现值 |

## 3. 关键调用链

**链 1：一次流程执行**

1. 调用方（API 或 aibitat 工具）`AgentFlows.loadFlow(slug)` 取 `{config: {steps}}`。
2. `new FlowExecutor().executeFlow(flow, inputs, context)`（`executor.js`）初始化 `variables = {...inputs}`。
3. 校验第一步必须是 `start`（`executor.js` start 块处理）。
4. `for` 循环遍历 `config.steps`：`replaceVariables(step, variables)` → 按 `step.type` switch 到对应 executor → 把返回结果写入 `variables[step.outputVar]`。
5. 若该步 `directOutput === true`，立即 `return result` 短路，跳过后续步骤。
6. 循环结束返回最终 variables 或最后一步结果；结果回灌宿主 aibitat。

**链 2：llm-instruction 步骤内部**

- `executors/llm-instruction.js`：用 context 提供的 provider/model，把 variables 拼成消息，`provider.complete(messages)`，文本结果存入 `variables[step.outputVar]`。

## 4. 配置项

| 配置 | 行为 | 位置 |
|---|---|---|
| `config.steps[]` | 有序步骤数组，顺序即执行顺序 | `flow.json` |
| `step.type` | 必须在 `FlowTypes` 枚举内，未知类型抛错 | `flowTypes.js`、`executor.js` |
| `step.outputVar` | 该步结果写入 variables 的键名 | 各 executor |
| `step.directOutput` | `true` 时短路返回 | `executor.js` |
| `${varName}` | 模板变量，执行时从 variables 取值 | `replaceVariables` |

## 5. 错误与重试语义

- 未知 step type：抛错终止整个流程（无自动重试）。
- 第一步不是 `start`：执行器报错。
- apiCall/llmInstruction 内部错误向上抛出，中断循环；调用方负责处理。
- 无重试/退避策略。

## 6. 并发细节

- 单流程实例内步骤**串行 for 循环**，无并发。
- 多流程实例之间相互独立，共享 `flowsDir` 只读加载。
- 无锁；variables 在单实例内顺序读写。

## 7. 系统边界

**In-Scope**
- `server/utils/agentFlows/`：CRUD、执行器、各步骤 executor、类型枚举

**Out-of-Scope**
- 流程在前端可视化编辑器的拖拽编排 UI → frontend 域
- LLM 推理本身 → ai-integrations 域 / `agent-runtime`
- 外部 HTTP 目标（apiCall 调用的第三方 API）→ 外部系统，不在本仓库源码内

## 8. 与相邻子系统交互

- 上游：API 路由（`/api/workspaces/{slug}/agent-flows`）CRUD；aibitat 把已启用流程登记为工具。
- 下游：`llmInstruction` → LLM provider；`apiCall` → 外部 HTTP；`webScraping` → 外部网页。
- 结果回 `agent-runtime`（aibitat）继续对话或直接直出给用户。

## 9. 语言专项适配口径（TS/JS 口径）

- **capability seam 分组**：Service Definition（`AgentFlows` 持久化 + `FlowTypes`）/ Provider（各 step executor）/ Consumer（`FlowExecutor` 串联）。
- **图类型**：architecture（组件拓扑）+ dataflow（线性步骤→变量累积→输出的数据管道语义，最贴切）。
- **外部边界**：外部 HTTP API、LLM provider。
- **省略 workflow/lifecycle/sequence**：本叶子虽“按顺序执行步骤”，但无多角色泳道、无审批回退，故不用 workflow；无单实体状态机（状态机属 agent-runtime）；调用链简单线性，dataflow 已表达，不另画 sequence。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| Flows 引擎组件拓扑 | `agent-flows-engine-architecture.html` | architecture | showcase |
| 执行数据流 | `agent-flows-engine-dataflow.html` | dataflow | standard |

- 两图 render 退出码 0、HTML 非空。
- **降档披露**：dataflow 目标 showcase 落 standard。初版因双向 vars↔step 回边造成 `clean-flow/edge-through-node` 与标签重叠；采取的修复：把两行步骤合并为单行、去掉双向回边的重复标签，改为单向虚线表示“下一步再取变量”。
- JSON IR 位于 `json/`。
