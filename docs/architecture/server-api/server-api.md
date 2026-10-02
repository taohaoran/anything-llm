# server-api 域总览

> 本文是 `server-api` 域的总览文档。本域覆盖 anything-llm 服务端 HTTP API 层：路由装配、鉴权、聊天、文档嵌入、管理、Agent/MCP、外部渠道接入。
>
> 源码基准：anything-llm v1.16.2，commit `128a0157`。主语言 JavaScript（并入 TS-JS 口径）。

## 域职责

`server-api` 域是 Express HTTP 服务的对外入口层，负责：

- 路由装配与中间件链（日志、CORS、限流、鉴权）。
- 登录/注册/JWT/Session 与多用户权限校验。
- 工作区聊天（含 SSE 流式响应）与会话历史。
- 文档上传/嵌入/删除/向量库操作端点。
- 系统设置、用户与工作区管理、统计端点。
- Agent 执行与 MCP 服务配置端点。
- 外部渠道（embed widget、Slack、Telegram、API、移动端推送）接入。

本域只做 HTTP 编排与参数校验，业务逻辑下沉到 ai-integrations 等域，数据访问下沉到 `server-data` 域。

## 叶子索引

| 叶子 | 职责 | 文档 |
|---|---|---|
| request-routing | Express 路由装配、中间件链、API 入口 | [request-routing.md](./request-routing/request-routing.md) |
| auth-authz-api | 登录/注册/JWT/Session/多用户权限中间件 | [auth-authz-api.md](./auth-authz-api/auth-authz-api.md) |
| workspace-chat-api | 工作区聊天、流式响应、会话历史 | [workspace-chat-api.md](./workspace-chat-api/workspace-chat-api.md) |
| document-embed-api | 文档上传/嵌入/删除/向量库端点 | [document-embed-api.md](./document-embed-api/document-embed-api.md) |
| admin-system-api | 系统设置/用户管理/工作区管理/统计 | [admin-system-api.md](./admin-system-api/admin-system-api.md) |
| agent-mcp-api | Agent 执行、MCP 服务配置端点 | [agent-mcp-api.md](./agent-mcp-api/agent-mcp-api.md) |
| ext-channels-api | 外部渠道（embed/Slack/Telegram/API）接入 | [ext-channels-api.md](./ext-channels-api/ext-channels-api.md) |

## 域级机制细节

- **统一入口**：`server/index.js` 装配 Express，挂载 `endpoints/` 下各路由与 `utils/middleware/` 中间件。
- **鉴权双轨**：单用户模式用登录 Session，多用户模式用 JWT + 用户角色（default/admin）+ 工作区成员校验（`multiUserProtected`、`validWorkspace`）。
- **流式响应**：聊天端点通过 SSE（Server-Sent Events）逐 token 推送，连接生命周期由端点管理。
- **外部渠道**：embed widget / Slack / Telegram 等各自挂载独立路由，复用核心 chat 逻辑但走独立鉴权（API key / 签名）。
