# Agent 与 MCP API（agent-mcp-api）

> 本文是 `server-api` 域下的叶子子系统文档。域级总览见 `../server-api.md`。
> 本文展开 Agent 调用 WebSocket、MCP 服务管理、Agent Flow 与技能白名单端点，不展开 agent 运行时/aibitat 内部（见 agents-mcp 域）。
>
> 源码基准：anything-llm v1.16.2，commit `128a0157`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| Agent 调用 WebSocket | `WS /agent-invocation/:uuid`，长连接跑 agent loop | `server/endpoints/agentWebsocket.js:24` |
| Socket 消息分发 | `relayToSocket` 处理工具切换/反馈/批准/澄清/bail 命令 | `agentWebsocket.js:12-24` |
| 会话中止 | socket close 时 `aibitat.abort()` 取消在途 LLM 请求 | `agentWebsocket.js:38-44` |
| MCP 服务列表 | `GET /mcp-servers/list`（admin） | `server/endpoints/mcpServers.js:35` |
| MCP 强制重载 | `GET /mcp-servers/force-reload` | `mcpServers.js:12` |
| MCP 启停/删除 | `POST /mcp-servers/toggle`、`/delete` | `mcpServers.js:55,78` |
| MCP 工具抑制 | `POST /mcp-servers/toggle-tool` | `mcpServers.js:99` |
| Agent Flow 管理 | save/list/get/delete/toggle `/agent-flows/*` | `server/endpoints/agentFlows.js:14-168` |
| 技能可用性探测 | filesystem/image-gen 等 `is-available` | `server/endpoints/agentSkillWhitelist.js:13-49` |
| 技能白名单 | `POST /agent-skills/whitelist/add` | `agentSkillWhitelist.js:66` |
| Agent 产物托管 | `/agent-skills/generated-files/:filename`、生成图片 | `server/endpoints/agentFileServer.js:30,96` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `agentWebsocket(app)` | `agentWebsocket.js:26` | 注册 WS agent 调用路由 |
| `relayToSocket(message)` | `agentWebsocket.js:12` | socket 入站消息分发到各 handler |
| `AgentHandler({uuid}).init()` / `createAIbitat` / `startAgentCluster` | `server/utils/agents/` | agent 会话生命周期（运行时见 agents-mcp 域） |
| `MCPCompatibilityLayer` | `server/utils/MCP/` | MCP 服务管理与工具调用兼容层 |
| `WorkspaceAgentInvocation.close(uuid)` | `server/models/workspaceAgentInvocation.js` | 关闭调用记录 |
| `WEBSOCKET_BAIL_COMMANDS` | `utils/agents/aibitat/plugins/websocket.js` | 中止命令白名单 |

## 3. 关键调用链

**链 A：建立 Agent WebSocket 会话（`agentWebsocket.js:24-65`）**
1. 浏览器连接 `/agent-invocation/:uuid`（`agentWebsocket.js:26`）。
2. `new AgentHandler({uuid}).init()` 加载 invocation；不存在则 `socket.close()` 返回（`agentWebsocket.js:29-35`）。
3. 注册 `socket.on("message", relayToSocket)` 与 `close` 处理器（`agentWebsocket.js:37-44`）。
4. 发遥测后 `createAIbitat({socket})`，若 socket 已关则不启动；再 `startAgentCluster()`（`agentWebsocket.js:56-63`）。
5. 异常时 `socket.send({type:"wssFailure"})` 并关闭（`agentWebsocket.js:66-69`）。

**链 B：运行中消息分发（`agentWebsocket.js:12-24`）**
1. 入站消息先 `String(message)` 归一化 Buffer。
2. 依次尝试 `handleToolToggle` → `handleFeedback` → `handleToolApproval` → `handleClarificationResponse`，命中即返回；否则 `checkBailCommand` 检测 bail 词并 abort。

**链 C：MCP 服务管理（`mcpServers.js`）**
1. 所有 `/mcp-servers/*` 经 `[validatedRequest, flexUserRoleValid([admin])]`。
2. 每次 `new MCPCompatibilityLayer()` 调用 list/reload/toggle/delete/toggleTool，结果序列化为 JSON 返回。

![Agent 与 MCP API 架构图](agent-mcp-api-architecture.html)
![Agent 调用 WebSocket 时序](agent-mcp-api-sequence.html)

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| 角色要求 | MCP 管理全部需 admin | `mcpServers.js:14,37,57,80,101` |
| WebSocket bail 命令 | 命中 `WEBSOCKET_BAIL_COMMANDS` 即中止 | `agentWebsocket.js:50-57` |
| MCP 服务配置 | 由 MCPCompatibilityLayer 持久化管理（外部 MCP 服务清单） | `utils/MCP/` |

## 5. 错误与重试语义

- invocation 不存在：直接 `socket.close()`，无重试（`agentWebsocket.js:32-35`）。
- agent 会话异常：发 `wssFailure` 帧后关闭 socket；不自动重连（重连由前端负责）。
- MCP 管理端点异常：catch 后 500 返回 `{success:false, error}`，`servers:[]`（`mcpServers.js:24-31`）。
- socket 断开：`aibitat.abort()` 取消在途 LLM 请求并关闭调用记录，防止泄漏（`agentWebsocket.js:38-44`）。

## 6. 并发细节

- WebSocket 长连接：单 socket 内事件驱动，`relayToSocket` 同步分发入站消息；agent loop 在 aibitat 内异步跑，事件经 socket 回流。
- close 事件触发 abort，取消传播到在途 LLM 请求（`agentWebsocket.js:40-41`）。
- socket 可能在 aibitat 构建期间关闭：`startAgentCluster` 前检查 `socket.readyState !== socket.OPEN` 则不启动（`agentWebsocket.js:61-62`）。
- MCP 管理端点为短请求，无共享状态；`new MCPCompatibilityLayer()` 每次新建。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- agent WebSocket 端点、MCP 服务管理端点、agent flows/技能白名单/产物托管端点。

**Out-of-Scope（不在本仓库源码内）**
- aibitat agent 运行时、工具执行、MCP 客户端协议细节（agents-mcp 域）。
- 外部 MCP 服务进程（不在本仓库源码内）。
- Agent Flow 引擎执行（agents-mcp 域 agentFlows）。

## 8. 与相邻子系统交互

- 本叶子 → agents-mcp：调用 AgentHandler、MCPCompatibilityLayer、aibitat。
- 本叶子 → server-data：读写 WorkspaceAgentInvocation。
- 本叶子 → auth-authz-api：MCP 管理走 admin 角色守卫。
- 上游：浏览器经 WebSocket 与 HTTP 管理面交互。

## 9. 语言专项适配口径（TS-JS 口径）

- **capability seam 分组**：WebSocket 端点是 Consumer，AgentHandler/MCP 兼容层是 Provider。
- **图型**：architecture（组件拓扑）+ sequence（WebSocket 会话建立与消息分发，重点体现 JS 事件驱动的 socket.on 与 abort 取消）。WebSocket/回调是 JS 事件驱动热点，用 sequence 表达。
- **外部边界**：图中标 `external` 的 MCP 服务；WebSocket 是 Node/浏览器长连接边界。
- **部署维度**：WebSocket 挂在同一 HTTP server（express-ws）；反向代理需支持 WS 升级（部署侧）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| Agent/MCP 架构图 | `agent-mcp-api-architecture.html` | architecture | **standard** |
| Agent WS 时序图 | `agent-mcp-api-sequence.html` | sequence | showcase |

- 架构图落 standard 原因：showcase 校验 `clean-flow/endpoint-side-direction`（跨层连线端点方向）反复未过；采取的修复动作：按 M2 调整组件为两行布局、加 fromSide/toSide 后仍未达 showcase，按兜底策略降 standard 渲染（render 退出码 0，HTML 约 804KB）。
- 未生成 dataflow/lifecycle：agent loop 状态机在 agents-mcp 域展开；本叶子只画接入边界。
- JSON IR 位于 `json/` 目录。
