# 工作区聊天 API（workspace-chat-api）

> 本文是 `server-api` 域下的叶子子系统文档。域级总览见 `../server-api.md`。
> 本文展开工作区聊天的流式响应、会话历史与会话线程端点，不展开文档上传嵌入（见 document-embed-api）。
>
> 源码基准：anything-llm v1.16.2，commit `128a0157`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 工作区流式聊天 | `POST /workspace/:slug/stream-chat`，SSE 推送 LLM 增量 | `server/endpoints/chat.js:23-103` |
| 会话线程流式聊天 | `POST /workspace/:slug/thread/:threadSlug/stream-chat`，带线程上下文与自动重命名 | `chat.js:105-209` |
| 工作区/线程校验中间件 | `validWorkspaceSlug`、`validWorkspaceAndThreadSlug`，注入 `locals.workspace/thread` | `server/utils/middleware/validWorkspace.js:6,28` |
| 聊天配额 | 多用户模式下 `User.canSendChat(user)` 检查 24h 消息上限 | `chat.js:50,137` |
| SSE 响应头装配 | `text/event-stream`、`Cache-Control:no-cache`、`Connection:keep-alive`、flushHeaders | `chat.js:44-48` |
| 聊天历史读取 | `GET /workspace/:slug/chats` | `server/endpoints/workspaces.js:414` |
| 删除聊天 | `POST delete-chats`、`delete-edited-chats` | `workspaces.js:441,472` |
| 编辑/反馈/置顶 | `update-chat`、`chat-feedback/:chatId`、`update-pin` | `workspaces.js:496,538,609` |
| TTS 朗读 | `POST /workspace/:slug/tts/:chatId` | `workspaces.js:636` |
| 建议消息 | GET/POST `suggested-messages` | `workspaces.js:562,580` |
| 提示词历史 | GET/POST/DELETE `prompt-history` | `workspaces.js:891,911,932` |
| 会话线程管理 | 线程 CRUD、fork、自动重命名 | `server/endpoints/workspaceThreads.js:26-219`、`chat.js:160-174` |
| 流式响应块工具 | `writeResponseChunk` 统一写 SSE 块 | `server/utils/helpers/chat/responses.js` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `chatEndpoints(app)` | `chat.js:20` | 注册两个 stream-chat 路由 |
| `validWorkspaceSlug` / `validWorkspaceAndThreadSlug` | `validWorkspace.js:6,28` | 按 slug 预取工作区/线程到 locals |
| `streamChatWithWorkspace(response, workspace, message, chatMode, user, thread, attachments)` | `server/utils/chats/stream.js:20` | 流式聊天核心（委托给 AI 集成域） |
| `writeResponseChunk(response, chunk)` | `utils/helpers/chat/responses.js` | 写单个 SSE 块 |
| `User.canSendChat(user)` | `models/user.js` | 多用户 24h 配额判定 |
| `WorkspaceThread.autoRenameThread({...onRename})` | `models/workspaceThread.js` | 线程首答后自动重命名，回调推 `rename_thread` 事件 |

## 3. 关键调用链

**链 A：工作区流式聊天（`chat.js:23-103`）**
1. 路由经 `[validatedRequest, flexUserRoleValid([ROLES.all]), validWorkspaceSlug]`（`chat.js:25`）。
2. 取 `userFromSession`、`reqBody` 的 `{message, attachments}`；空消息直接返回 `type:"abort"` 块（`chat.js:28-42`）。
3. 设置 SSE 响应头并 `flushHeaders()`（`chat.js:44-48`）。
4. 多用户模式 `User.canSendChat` 配额超限则写 abort 块返回（`chat.js:50-60`）。
5. `streamChatWithWorkspace(response, workspace, message, workspace.chatMode, user, null, attachments)` 执行检索+流式补全（`chat.js:62-70`）。
6. 结束后发遥测 `sent_chat`、写 EventLogs，`response.end()`（`chat.js:71-89`）；异常时写 abort 块并 end（`chat.js:90-101`）。

**链 B：线程聊天与自动重命名（`chat.js:105-209`）**
1. 经 `validWorkspaceAndThreadSlug` 同时注入 workspace 与 thread（`chat.js:107-117`）。
2. 流式聊天后调用 `WorkspaceThread.autoRenameThread`，重命名时通过 `onRename` 回调写 `{action:"rename_thread", thread:{slug,name}}` 特殊 SSE 块推给前端（`chat.js:160-174`）。

![工作区聊天 API 架构图](workspace-chat-api-architecture.html)
![工作区流式聊天请求时序](workspace-chat-api-sequence.html)

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `workspace.chatMode` | 聊天模式（chat/query 等），传入 streamChatWithWorkspace | `chat.js:66,153` |
| `user.dailyMessageLimit` | 多用户 24h 聊天配额，超限拒绝 | `chat.js:57` |
| `LLM_PROVIDER` / `EMBEDDING_ENGINE` / `VECTOR_DB` / `TTS_PROVIDER` | 仅用于遥测上报选型 | `chat.js:73-77` |
| SSE 响应头 | 固定 `text/event-stream` 等，非配置 | `chat.js:44-48` |

## 5. 错误与重试语义

- 空消息：400 + `{type:"abort", close:true, error:"Message is empty."}`（`chat.js:32-42`）。
- 配额超限：SSE abort 块提示 24h 上限，不抛错（`chat.js:50-60`）。
- 工作区/线程不存在：中间件直接 404（`validWorkspace.js:18,46`）。
- 流式过程异常：`catch` 写 abort 块 `{error:e.message}` 后 `response.end()`（`chat.js:90-101`）。
- 客户端主动中断：`clientAbortedHandler` 提前 resolve 退出 LLM 流（`responses.js:20-25`）。
- 本叶子不做 LLM 调用重试；检索/补全重试在 ai-integrations 域。

## 6. 并发细节

- SSE 长连接：`flushHeaders` 后连接保持，逐块 `writeResponseChunk` 推送，单请求期间事件循环被异步流式 I/O 让出。
- 线程自动重命名在流式结束后异步执行，其结果通过 SSE 回调块追加推送，不阻塞主响应。
- 遥测与 EventLogs 为 `await` 串行写在 `response.end()` 之前（`chat.js:71-88`）。
- 无共享内存锁；每个 SSE 响应对象独立。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- chat.js 两个 stream-chat 路由、workspaces.js 聊天历史/编辑/反馈/TTS/提示词历史、workspaceThreads.js 线程管理、validWorkspace 中间件、SSE 块工具。

**Out-of-Scope（不在本仓库源码内）**
- 检索增强、LLM 流式补全实现（ai-integrations 域）。
- 聊天记录 ORM 细节（server-data 域 WorkspaceChats 模型）。
- 前端聊天 UI（frontend 域）。
- 外部 LLM/向量库 API（不在本仓库源码内）。

## 8. 与相邻子系统交互

- 本叶子 → auth-authz-api：依赖 `validatedRequest`、`flexUserRoleValid`、`userFromSession`。
- 本叶子 → server-data：读写 Workspace、WorkspaceThread、WorkspaceChats、User（配额）。
- 本叶子 → ai-integrations：`streamChatWithWorkspace` 调用向量库检索与 LLM 补全。
- 本叶子 → agents-mcp：线程自动重命名、agent 命令可用性检测。
- 上游：Web 聊天端经 SSE 长连接消费块。

## 9. 语言专项适配口径（TS-JS 口径）

- **capability seam 分组**：端点是 Consumer（调用 streamChatWithWorkspace），validWorkspace 是横切中间件，Workspace/Chats 模型是 Provider。
- **图型**：architecture（组件拓扑）+ sequence（SSE 流式请求-响应时序，重点体现 JS 异步流式与 chunk 回流）。SSE/流式是 JS 事件驱动热点，用 sequence 的 return 消息表达增量回流。
- **外部边界**：图中标 `external` 的 LLM/向量库 API；SSE 是浏览器/Node HTTP 流式边界。
- **部署维度**：聊天端点随 server 单进程；SSE 长连接需要反向代理关闭缓冲（部署侧，不在本仓库源码内）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 聊天 API 架构图 | `workspace-chat-api-architecture.html` | architecture | showcase |
| 流式聊天时序图 | `workspace-chat-api-sequence.html` | sequence | showcase |

- 未生成 dataflow：检索增强管道本身在 ai-integrations 域展开，本叶子只画调用边界；未生成 lifecycle：聊天会话无显式状态机。
- JSON IR 位于 `json/` 目录。
