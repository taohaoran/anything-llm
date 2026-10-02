# 外部渠道接入 API（ext-channels-api）

> 本文是 `server-api` 域下的叶子子系统文档。域级总览见 `../server-api.md`。
> 本文展开嵌入 widget、外部连接器、Telegram、WebPush、移动设备等对外渠道端点，不展开聊天核心（见 workspace-chat-api）。
>
> 源码基准：anything-llm v1.16.2，commit `128a0157`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 嵌入 widget 流式聊天 | `POST /embed/:embedId/stream-chat`，SSE 流式 | `server/endpoints/embed/index.js:19-67` |
| 嵌入历史读取 | `GET /embed/:embedId/:sessionId` | `embed/index.js:69-90` |
| 嵌入历史清除 | `DELETE /embed/:embedId/:sessionId` | `embed/index.js:92-107` |
| 嵌入配置管理 | `/embeds` CRUD（admin） | `server/endpoints/embedManagement.js:19-116` |
| 嵌入守卫 | `validEmbedConfig`、`canRespond`、`setConnectionMeta` | `server/utils/middleware/embedMiddleware.js:9,43,22` |
| 外部文档连接器 | `/ext/:repo_platform/branches`、`/repo`、`/ext/youtube/transcript`、`/ext/confluence`、`/ext/website-depth`、`/ext/drupalwiki`、`/ext/obsidian/vault` | `server/endpoints/extensions/index.js:15-174` |
| Telegram 配置与连接 | `/telegram/config`、`/connect`、`/disconnect`、`/status` | `server/endpoints/telegram.js:18-179` |
| Telegram 用户审批 | pending/approved/approve/deny/revoke | `telegram.js:197-272` |
| WebPush 订阅 | `/web-push/subscribe`、`/web-push/pubkey` | `server/endpoints/webPush.js:9,21` |
| 移动设备管理 | `/mobile/devices`、`/connect-info`、`/register`、`/send/:command`、`/auth` | `server/endpoints/mobile/index.js:20-147` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `embeddedEndpoints(app)` | `embed/index.js:16` | 注册对外 embed 聊天端点 |
| `validEmbedConfig` / `canRespond` / `setConnectionMeta` | `embedMiddleware.js:9,43,22` | embed 配置校验、是否允许响应、连接元数据 |
| `streamChatWithForEmbed(response, embed, message, sessionId, overrides)` | `server/utils/chats/embed.js` | 嵌入 widget 专用流式聊天 |
| `EmbedChats.forEmbedByUser` / `markHistoryInvalid` | `server/models/embedChats.js` | 嵌入会话历史读写 |
| `extensionEndpoints(app)` | `endpoints/extensions/index.js` | 注册 `/ext/*` 连接器 |
| `telegramEndpoints(app)` | `endpoints/telegram.js` | Telegram 配置与用户审批 |

## 3. 关键调用链

**链 A：嵌入 widget 流式聊天（`embed/index.js:19-67`）**
1. 路由经 `[validEmbedConfig, setConnectionMeta, canRespond]`（`embed/index.js:21`）。
2. 取 `sessionId/message` 及可选 override（prompt/model/temperature/username）（`embed/index.js:25-33`）。
3. 设置 SSE 响应头并 `flushHeaders()`（`embed/index.js:35-39`）。
4. `streamChatWithForEmbed(response, embed, message, sessionId, overrides)` 执行聊天（`embed/index.js:41-46`）。
5. 结束后发遥测 `embed_sent_chat` 并 `response.end()`；异常写 abort 块（`embed/index.js:47-65`）。

**链 B：外部连接器抓取（`extensions/index.js`）**
1. `/ext/*` 路由把请求转发给 collector/对应外部服务（如 GitHub branches、Confluence、YouTube transcript）。
2. 返回可嵌入文档列表，供用户加入工作区。

![外部渠道接入架构图](ext-channels-api-architecture.html)
![嵌入 widget 聊天时序](ext-channels-api-sequence.html)

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| embed overrides | 允许 widget 传 prompt/model/temperature/username 覆盖默认 | `embed/index.js:28-33` |
| embed 守卫 | `canRespond` 判断该 embed 是否允许响应该请求 | `embedMiddleware.js:43` |
| Telegram bot | 由 `/telegram/connect` 配置 token 并启用 | `telegram.js:76` |
| WebPush | `/web-push/pubkey` 返回公钥供前端订阅 | `webPush.js:21` |

## 5. 错误与重试语义

- embed 异常：写 abort 块 `{type:"abort", error:e.message}` 后 end（`embed/index.js:54-65`）。
- 历史读取失败：500；清除历史 `markHistoryInvalid` 失败 500。
- 连接器失败：转发 collector 错误信息。
- Telegram 用户审批：未批准用户消息被拒（见 telegramBot，agents-mcp 域）。
- 本叶子不做外部 API 重试；连接器抓取重试在 collector 内。

## 6. 并发细节

- embed 聊天为 SSE 长连接，与工作区聊天同构。
- Telegram/WebPush/移动推送为事件驱动：bot 与推送服务在后台运行（见 agents-mcp 域），本叶子只提供配置与设备管理端点。
- 无共享锁；embed 会话按 embedId+sessionId 隔离。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- embed widget 聊天/历史、embed 配置管理、外部连接器、Telegram 配置/审批、WebPush、移动设备端点。

**Out-of-Scope（不在本仓库源码内）**
- 嵌入 widget 前端（embed 子模块，未 checkout）。
- Telegram bot 运行时（telegramBot，agents-mcp 域）。
- 外部 GitHub/Confluence/YouTube 等服务（不在本仓库源码内）。
- 浏览器扩展（browser-extension 子模块，未 checkout）。

## 8. 与相邻子系统交互

- 本叶子 → workspace-chat-api：embed 聊天复用流式聊天核心（streamChatWithForEmbed）。
- 本叶子 → server-data：读写 EmbedChats、EmbedConfig、MobileDevice。
- 本叶子 → document-pipeline：外部连接器经 collector 抓取文档。
- 本叶子 → agents-mcp：Telegram bot、PushNotifications 后台服务。
- 上游：嵌入 widget、Telegram 用户、移动 App、外部系统。

## 9. 语言专项适配口径（TS-JS 口径）

- **capability seam 分组**：各渠道端点是 Consumer，转发到聊天核心与 collector；embedMiddleware 是横切守卫。
- **图型**：architecture（多渠道接入拓扑）+ sequence（embed widget SSE 请求链）。SSE/Webhook 是 JS 事件驱动热点。
- **外部边界**：图中标 `external` 的 Telegram 用户、移动设备、外部服务；Webhook/WebSocket 是 Node HTTP 边界。
- **部署维度**：多渠道经同一 server 进程；Telegram webhook、WebPush 需公网可达（部署侧）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 外部渠道架构图 | `ext-channels-api-architecture.html` | architecture | showcase |
| 嵌入 widget 时序图 | `ext-channels-api-sequence.html` | sequence | showcase |

- 未生成 dataflow/lifecycle：连接器抓取管道在 document-pipeline 域展开；无单实体状态机。
- JSON IR 位于 `json/` 目录。
