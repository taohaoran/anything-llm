# chat-interface 聊天界面（chat-interface）

> 本文是 `frontend` 域下的叶子子系统文档。域级总览见 `../frontend.md`。
> 本文只展开**工作区聊天界面**：消息列表渲染、SSE 流式输出、输入框与附件、工作区/线程切换、agent 会话 WebSocket 移交；
> 路由外壳与鉴权见 `app-shell`，工作区文档管理见 `workspace-admin-ui`。
>
> 源码基准：`frontend/`，commit `128a015`，JavaScript（ESM，TS-JS 口径）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 工作区加载 | 按 URL `slug` 拉取工作区详情、建议消息、agent 命令可用性，Slack 式过渡保留旧聊天 | `frontend/src/pages/WorkspaceChat/index.jsx:27-72` |
| 密码工作区门禁 | 进入需密码的工作区先弹 `PasswordModal` | `pages/WorkspaceChat/index.jsx:12-17`、`components/Modals/Password` |
| 消息提交 | 组装 user 消息 + 空 pending assistant，处理附件与空工作区自动建线程 | `components/WorkspaceChat/ChatContainer/index.jsx:100-149` |
| SSE 流式请求 | `@microsoft/fetch-event-source` POST `/workspace/:slug/stream-chat`，AbortController 中止 | `frontend/src/models/workspace.js:161-232` |
| 消息类型分发 | `handleChat` 按 `type`（modelRouteNotification/imageGenerationPending/abort/statusResponse/...）更新历史 | `frontend/src/utils/chat/index.js:7-90` |
| 中止生成 | 监听 `ABORT_STREAM_EVENT`，`ctrl.abort()` 并回 `stopGeneration` | `models/workspace.js:168-172`、`utils/chat/index.js:4` |
| agent 会话移交 | `@agent:` 时关闭 HTTP 流、改走 WebSocket；`agentEventLoadingState` 控制发送/停止钮 | `utils/chat/agent.js:45-54`、`utils/chat/index.js:58-71` |
| 附件拖拽上传 | DnDWrapper 提供 `files/parseAttachments`，拖拽清空事件 | `components/WorkspaceChat/ChatContainer/DnDWrapper`、`ChatContainer/index.jsx:54` |
| 语音输入 | `react-speech-recognition`，提交时结束 STT | `ChatContainer/index.jsx:82-84`、`:143-154` |
| 侧栏 | ChatSidebar（历史线程）、SourcesSidebar（来源引用）、MemoriesSidebar（记忆） | `components/WorkspaceChat/ChatContainer/` |
| 输入框草稿持久化 | `usePromptInputStorage` 按线程保存/清空草稿 | `hooks/usePromptInputStorage.js`、`ChatContainer/index.jsx:108` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `ChatContainer({workspace, threadSlug, knownHistory})` | `ChatContainer/index.jsx:43` | 聊天状态中枢：`chatHistory/loadingResponse/websocket/socketId` |
| `handleChat(chatResult, setLoadingResponse, setChatHistory, ...)` | `utils/chat/index.js:7` | SSE 消息分发器，按 `type` 生产新消息并入历史 |
| `Workspace.streamChat({slug}, message, handleChat, attachments)` | `models/workspace.js:161` | SSE 客户端封装，回调 `onopen/onmessage/onerror` |
| `handleSocketResponse(socket, event, setChatHistory)` | `utils/chat/agent.js:88` | agent WebSocket 事件分发 |
| `agentEventLoadingState(data)` | `utils/chat/agent.js:45` | 由 socket 事件推导 loading 三态（true/false/null） |
| `websocketURI()` | `utils/chat/agent.js:82` | 由 location/`VITE_API_BASE` 推导 `ws/wss` 地址 |
| `usePasswordModal()` | `pages/WorkspaceChat/index.jsx:12` | 工作区密码门禁状态 |
| `PROMPT_INPUT_EVENT` / `ABORT_STREAM_EVENT` | `ChatContainer/index.jsx:6-7`、`utils/chat/index.js:4` | 跨组件自定义事件通道 |

## 3. 关键调用链

**链 1：发送一条普通消息并流式渲染**
1. 用户在 PromptInput 提交 → `ChatContainer.handleSubmit`（`ChatContainer/index.jsx:100`）读取输入框 `PROMPT_INPUT_ID`（`:102-104`）。
2. 空工作区且无历史时先 `Workspace.threads.new(slug)` 建线程并 `navigate` 到线程 URL（`:112-124`）；否则组装 `prevChatHistory = [...chatHistory, userMsg, pendingAssistant]`（`:127-141`），`setChatHistory` 渲染（`:146`）。
3. 调 `Workspace.streamChat`（`models/workspace.js:161`）：建 `AbortController`（`:162`），注册 `ABORT_STREAM_EVENT` 监听（`:172`）。
4. `fetchEventSource` POST `/workspace/:slug/stream-chat`（`:175`）；`onopen` 对 4xx（非 429）发 `type:"abort"` 错误消息并 abort（`:184-198`）。
5. 每个 `onmessage` 用 `safeJsonParse` 解析后回调 `handleChat(chatResult)`（`:212-214`）；`handleChat` 按 type 把内容写进历史并 `setChatHistory`（`utils/chat/index.js:73-90`）。
6. 完成/`close` 时 `setLoadingResponse(false)`（`utils/chat/index.js:62`）；`finally` 移除 abort 监听（`models/workspace.js:230`）。

**链 2：停止生成**
1. 用户点停止钮 → 广播 `ABORT_STREAM_EVENT`（`utils/chat/index.js:4`）。
2. `streamChat` 内 `onAbortStream` 执行 `ctrl.abort()` 并回 `handleChat({type:"stopGeneration"})`（`models/workspace.js:168-171`）。
3. `handleChat` 收到 abort 关闭 pending 消息、复位 loading（`utils/chat/index.js:58-62`）。

**链 3：agent 会话从 HTTP 移交 WebSocket**
1. SSE 下发 `@agent:` 开头的 statusResponse → `handleChat` 直接 return（不落历史、不关 loading，因 agent WS 接管）（`utils/chat/index.js:70-71`）。
2. ChatContainer 据 `AGENT_SESSION_START` 建立 WebSocket，`websocketURI()` 推导地址（`agent.js:82-86`）。
3. 每个 socket 事件经 `handleSocketResponse`（`agent.js:88`）分发；`agentEventLoadingState` 判定是否仍在工作（`agent.js:45-54`）：`WAITING_ON_INPUT/toolApprovalRequest` 等→显示发送钮，被动事件→不改 loading。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---|---|---|
| `openWhenHidden: true` | 页面隐藏时仍保持 SSE 连接 | `models/workspace.js:180` |
| `VITE_API_BASE` | 推导 WebSocket 主机 | `agent.js:85` |
| `PENDING_HOME_MESSAGE` | sessionStorage 暂存新建线程后的待发消息 | `ChatContainer/index.jsx:115`、`utils/constants.js` |
| `LAST_VISITED_WORKSPACE` | localStorage 记录最近访问工作区 | `pages/WorkspaceChat/index.jsx:55-61` |
| `AGENT_AWAITING_USER_EVENTS` | 暂停等用户输入的事件白名单 | `agent.js:15-22` |

## 5. 错误与重试语义

- **SSE 4xx（非 429）**：立即发 `type:"abort"` 错误消息渲染到历史，`ctrl.abort()` 终止，**不自动重试**（`models/workspace.js:184-198`）。
- **onerror**：发 abort 错误消息后 abort 并 `throw new Error()` 终止 fetch-event-source 自动重连（`:216-227`）——刻意不重连，避免重复发问。
- **JSON 解析失败**：`safeJsonParse(msg.data, null)` 丢弃坏帧（`:213`）。
- **agent socket 失败**：`wssFailure` 事件走 `handledEvents` 分支提示用户（`agent.js:67-80`）。
- **引用（citations）乱序到达**：按 uuid 在模块级 `bufferedCitations` Map 缓冲，待消息出现后回填（`agent.js:56-66`）。

## 6. 并发细节

- React 状态驱动：`chatHistory` 数组不可变更新（`setChatHistory([...])`）驱动重渲染；`chatHistoryRef` 配合 `useChatContainerQuickScroll` 处理快速滚动（`ChatContainer/index.jsx:55`）。
- `ResizeObserver` 监听输入框高度，动态调整历史区底部 padding（`:67-80`），卸载 disconnect。
- SSE 与 WebSocket 双通道并存：HTTP 流负责普通 chat，agent 阶段由 WS 接管，`getAgentSessionActive()` 避免 statusResponse 误关停止钮（`utils/chat/index.js:62`）。
- 自定义事件（`PROMPT_INPUT_EVENT`/`ABORT_STREAM_EVENT`/`THREAD_RENAME_EVENT`）绕过 props 下发，避免高频重渲染（`ChatContainer/index.jsx:92-98`）。
- 无后台轮询；线程重命名经 `THREAD_RENAME_EVENT` 广播到侧栏（`agent.js:93-101`）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `frontend/src/pages/WorkspaceChat/`、`components/WorkspaceChat/`、`utils/chat/`、`models/workspace.js`（streamChat/threads/suggestedMessages）
- PromptInput、DnDWrapper、ChatHistory、SourcesSidebar、MemoriesSidebar、ChatSidebar

**Out-of-Scope（不在本仓库源码内）**
- SSE/WebSocket 服务端实现、LLM 流式拼接、agent 执行循环 → `server/`（server-api / agents-mcp 域）
- `@microsoft/fetch-event-source`、`react-speech-recognition`、`react-device-detect` 为第三方 npm 包
- 浏览器 DOM/WebSocket/ResizeObserver 为浏览器 API
- 工作区文档上传解析 → collector 管道（document-pipeline 域）

## 8. 与相邻子系统交互

- **app-shell → chat-interface**：`/workspace/:slug` 路由经 `PrivateRoute` 放行后挂载本叶子；Sidebar 切换工作区触发 `slug` 参数变化重新加载。
- **chat-interface → settings-console-models**：`WorkspaceModelPicker` 选用工作区/系统模型；模型偏好配置在设置控制台。
- **chat-interface → 后端**：经 `models/workspace.js` 调 REST/SSE，外部 API 不在前端源码内。
- **chat-interface → workspace-admin-ui**：来源引用侧栏展示检索到的文档片段；文档本身管理在工作区设置页。

## 9. 语言专项适配口径（JavaScript / TS-JS）

- **capability seam**：Consumer（ChatContainer 消费流结果）/ Provider（`models/workspace.js` SSE 客户端、`utils/chat/*` 分发器）/ Service（后端 SSE/WS，外部）。
- **图型**：architecture（组件分层）+ sequence（发送→流式→中止/移交的 Promise/事件链，JS 事件驱动热点进 sequence）；本叶子无单实体状态机，agent 会话状态由三态函数表达而非 lifecycle。
- **外部边界**：标注外部 REST/SSE/WebSocket API 与浏览器 API（DOM/ResizeObserver/WebSocket）；不涉及 etcd/CRD。
- **部署维度**：Vite SPA 依赖 `@microsoft/fetch-event-source` 等 npm 包，构建产物静态托管（见 docker-build 叶子）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 聊天界面架构图 | `chat-interface-architecture.html` | architecture | **standard**（showcase 未过：同列节点间纵向桥接的 `endpoint-side-direction` 检查；已将 handleChat 分发器并入 stream→container 回调、重排为水平主路径并缩短重叠标签，render 退出码 0） |
| 发送消息与流式渲染时序 | `chat-interface-sequence.html` | sequence | **showcase** |

- JSON IR 源文件：`json/chat-interface-architecture.json`、`json/chat-interface-sequence.json`。
- 省略说明：本叶子未生成 dataflow/lifecycle/workflow 图——消息渲染是单向 SSE 回调链（已用 sequence 表达），无 ETL 管道、单实体状态机或多角色审批泳道，按资源节省原则省略。
