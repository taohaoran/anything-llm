# frontend 前端域（frontend）

> 本域是 anything-llm 的 Vite + React SPA（`frontend/`，纯 JavaScript ESM），负责用户交互、状态管理与后端 API 调用。
> 域级总览：本文件；各叶子设计文档见下表。源码基准 commit `128a015`。

## 域职责

- 应用外壳：路由表、Provider 上下文树、主题/会话全局状态、路由权限守卫、侧边导航。
- 聊天界面：消息列表、SSE 流式渲染、输入框与附件、工作区/线程切换、agent WebSocket 移交。
- 工作区管理 UI：工作区设置五个 Tab、自动保存表单、文档上传与解析触发。
- 认证与引导 UI：登录/SSO、单机/多用户密码表单、首次 onboarding 五步向导。
- 设置控制台：系统级 LLM/向量库/Embedder/STT-TTS 偏好、40+ provider 选择器、用户与工作区管理。

前端经 `frontend/src/utils/request.js` 统一注入 `Authorization/X-Timezone/X-Language` 请求头，所有数据经 `models/*.js` 调外部 REST/SSE API（后端不在本域源码内）。

## 叶子索引

| 叶子 | 职责 | 文档 | 图 |
|---|---|---|---|
| app-shell | 路由、布局、主题、全局状态、导航 | [app-shell/app-shell.md](app-shell/app-shell.md) | [架构](app-shell/app-shell-architecture.html) · [时序](app-shell/app-shell-sequence.html) |
| chat-interface | 消息列表、流式渲染、输入框、附件、工作区切换 | [chat-interface/chat-interface.md](chat-interface/chat-interface.md) | [架构](chat-interface/chat-interface-architecture.html) · [时序](chat-interface/chat-interface-sequence.html) |
| workspace-admin-ui | 工作区文档管理、向量库状态、工作区设置 | [workspace-admin-ui/workspace-admin-ui.md](workspace-admin-ui/workspace-admin-ui.md) | [架构](workspace-admin-ui/workspace-admin-ui-architecture.html) · [时序](workspace-admin-ui/workspace-admin-ui-sequence.html) |
| auth-onboarding-ui | 登录/注册/首次设置向导 | [auth-onboarding-ui/auth-onboarding-ui.md](auth-onboarding-ui/auth-onboarding-ui.md) | [架构](auth-onboarding-ui/auth-onboarding-ui-architecture.html) · [时序](auth-onboarding-ui/auth-onboarding-ui-sequence.html) |
| settings-console-models | 系统设置、LLM/向量库配置、用户管理、模型选择器 | [settings-console-models/settings-console-models.md](settings-console-models/settings-console-models.md) | [架构](settings-console-models/settings-console-models-architecture.html) · [时序](settings-console-models/settings-console-models-sequence.html) |

## 域级机制细节

- **路由分级**：`main.jsx` 用 `createBrowserRouter` 懒加载；`PrivateRoute/AdminRoute/ManagerRoute/SingleUserRoute` 按角色与单机/多用户模式放行（见 app-shell）。
- **状态管理**：以 React Context（Auth/Theme/Logo/Pfp/PWA/I18n）+ 局部 `useState` 为主，无全局状态库；跨组件用自定义事件（`PROMPT_INPUT_EVENT`/`ABORT_STREAM_EVENT`）避免高频重渲染。
- **流式交互**：普通 chat 走 `@microsoft/fetch-event-source` SSE；agent 阶段移交 WebSocket，由 `agentEventLoadingState` 三态控制发送/停止钮（见 chat-interface）。
- **JS 口径**：纯 JavaScript 并入 TS-JS 口径；类型线索来自 JSDoc `@typedef` 与 Context 导出；图型以 architecture + sequence + dataflow 为主。

## 图质量档位

本域 5 叶子各 2 图共 10 张：8 张 showcase，2 张 standard（app-shell 架构图、chat-interface 架构图，均因纵向桥接走线清档检查未达 showcase 阈值，已在对应叶子 MD 第 10 节披露修复动作，render 退出码均为 0）。
