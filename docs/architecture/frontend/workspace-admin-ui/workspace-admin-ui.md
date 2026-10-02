# workspace-admin-ui 工作区管理界面（workspace-admin-ui）

> 本文是 `frontend` 域下的叶子子系统文档。域级总览见 `../frontend.md`。
> 本文只展开**工作区级管理界面**：工作区设置 Tab 页（外观/聊天/向量库/成员/Agent）、自动保存表单、文档上传与解析；
> 系统级设置（LLM/向量库/用户）见 `settings-console-models`，聊天界面见 `chat-interface`。
>
> 源码基准：`frontend/`，commit `128a015`，JavaScript（ESM，TS-JS 口径）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 设置外壳与 Tab 路由 | 表驱动 `TABS` 映射五个设置页，按 URL `tab` 参数渲染 | `frontend/src/pages/WorkspaceSettings/index.jsx:28-34`、`:79` |
| 工作区加载 | 按 slug 拉工作区详情 + `System.keys()` 取向量库类型 + 建议消息 | `WorkspaceSettings/index.jsx:55-75` |
| 密码门禁 | 受保护工作区先弹 `PasswordModal` | `WorkspaceSettings/index.jsx:37-42` |
| 外观与名称 | 工作区名称、建议聊天消息、删除工作区（删除保护） | `pages/WorkspaceSettings/GeneralAppearance/`（WorkspaceName/SuggestedChatMessages/DeleteWorkspace） |
| 聊天设置 | 聊天模式、提示词、温度、LLM 选择、历史设置 | `pages/WorkspaceSettings/ChatSettings/` |
| 向量库设置 | 向量数展示、检索模式、最大上下文片段、相似度阈值、重置库、向量库标识 | `pages/WorkspaceSettings/VectorDatabase/index.jsx:11-40` |
| 成员管理 | 成员列表、加成员弹窗（仅 admin/manager 可见） | `pages/WorkspaceSettings/Members/`（index/AddMemberModal/WorkspaceMemberRow） |
| Agent 配置 | 工作区级 Agent 行为配置 | `pages/WorkspaceSettings/AgentConfig/` |
| 自动保存表单 | `AutosaveForm` 防抖提交，`castToType` 按字段类型转换 | `VectorDatabase/index.jsx:13-24`、`components/AutosaveForm` |
| 文档上传/解析 | `uploadFile`/`parseFile`/`uploadLink`/`getParsedFiles` | `frontend/src/models/workspace.js:271-312` |
| 文档置顶 | `setPinForDocument` 更新文档置顶状态 | `models/workspace.js:349-354` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `TABS`（对象映射） | `WorkspaceSettings/index.jsx:28` | tab key → 页面组件 |
| `WorkspaceSettings()` / `ShowWorkspaceChat()` | `index.jsx:36/47` | 设置外壳与数据加载 |
| `TabItem` | `index.jsx:133` | NavLink 页签，按角色 `visible` 控制 |
| `AutosaveForm` | `components/AutosaveForm` | 防抖自动提交表单容器 |
| `Workspace.update(slug, data)` | `models/workspace.js`（被 `VectorDatabase/index.jsx:19` 调用） | 更新工作区配置 |
| `Workspace.uploadFile/parseFile/uploadLink` | `models/workspace.js:271/281/303` | 文档上传/解析/链接抓取 |
| `castToType(key, value)` | `utils/types.js` | 表单字符串按字段语义转 number/bool |

## 3. 关键调用链

**链 1：向量库设置自动保存**
1. 用户在 VectorDatabase 页编辑字段 → `AutosaveForm` 防抖触发 `onSave(formEl)`（`VectorDatabase/index.jsx:13`）。
2. `handleUpdate` 用 `new FormData(formEl)` 收集字段，逐字段 `castToType` 转换（`:15-17`）。
3. 调 `Workspace.update(workspace.slug, data)`（`:19`）；失败 `showToast(\`Error: ${message}\`)` 并返回 `false`（`:20-22`）；成功返回 `true` 保持静默。

**链 2：进入工作区设置**
1. 路由 `/workspace/:slug/settings/:tab` 经 `ManagerRoute` 放行（`main.jsx:39-46`）。
2. `WorkspaceSettings` 先 `usePasswordModal()` 门禁（`index.jsx:37`）；通过后 `ShowWorkspaceChat` 按 slug 拉数据（`:55-75`）。
3. `TabContent = TABS[tab]`（`:79`）渲染对应页签，传入 `slug/workspace/deletionProtected`（`:122-126`）。

**链 3：文档上传解析**
1. 前端 `Workspace.uploadFile(slug, formData)` POST `/workspace/:slug/upload`（`models/workspace.js:272`）。
2. `parseFile` POST `/workspace/:slug/parse`（`:282`）；`uploadLink` POST `/workspace/:slug/upload-link` 抓取链接（`:304`）。
3. 解析结果经 collector 管道（见 document-pipeline 域）异步入向量库，前端 `VectorCount` 刷新向量数。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---|---|---|
| `WorkspaceDeletionProtection` | 来自 `System.keys()`，true 时禁止删除工作区 | `WorkspaceSettings/index.jsx:71` |
| `VectorDB` | 来自 `System.keys()`，标识当前向量库类型 | `index.jsx:68` |
| 成员页签可见性 | `["admin","manager"].includes(user?.role)` | `index.jsx:113` |
| `cache: "no-cache"` | 建议消息接口禁用缓存 | `models/workspace.js:317` |

## 5. 错误与重试语义

- **自动保存失败**：`Workspace.update` 返回 `{workspace:null, message}` 时 toast 错误，表单保持用户输入不回滚（`VectorDatabase/index.jsx:20-22`）。
- **上传失败**：`uploadFile` 直接返回 `{response, data}`，由调用方据 `response.ok` 处理；本叶子不自动重试。
- **工作区不存在**：`Workspace.bySlug` 返回 null 时 `setLoading(false)` 退出加载（`WorkspaceSettings/index.jsx:59-62`）。
- 无指数退避；删除工作区受 `deletionProtected` 保护（`GeneralAppearance/DeleteWorkspace`）。

## 6. 并发细节

- React 状态驱动：`workspace/loading/deletionProtected` 三态；`useEffect([slug, tab])` 在切 tab 时重新拉数据（`index.jsx:75`）。
- `AutosaveForm` 内部防抖避免高频请求；表单值不可变更新。
- 移动端隐藏 Sidebar（`isMobile` 判断，`index.jsx:82`）。
- 无后台轮询；向量数 `VectorCount reload` 在上传完成后触发刷新。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `frontend/src/pages/WorkspaceSettings/`（五个 Tab 子目录）、`components/AutosaveForm`、`models/workspace.js`（update/upload/parse/pin）

**Out-of-Scope（不在本仓库源码内）**
- 文档解析/向量化管道 → collector/ 与 server utils（document-pipeline / ai-integrations 域）
- 向量库实际读写 → server/utils/vectorDbProviders
- 系统级 LLM/向量库/用户配置 → settings-console-models 叶子
- `react-device-detect` 为第三方 npm 包

## 8. 与相邻子系统交互

- **app-shell → workspace-admin-ui**：`/workspace/:slug/settings/:tab` 经 `ManagerRoute` 放行。
- **workspace-admin-ui → chat-interface**：返回聊天页按钮 `paths.workspace.chat(slug)`（`index.jsx:89`）；向量库设置影响聊天检索。
- **workspace-admin-ui → settings-console-models**：工作区 LLM 选择与系统 LLM 偏好联动。
- **workspace-admin-ui → 后端**：经 `models/workspace.js` 调 REST，外部 API 不在前端源码内；文档解析落 collector 管道。

## 9. 语言专项适配口径（JavaScript / TS-JS）

- **capability seam**：Service Definition（`TABS` 路由映射、`AutosaveForm` 容器）/ Consumer（五个 Tab 页面消费 workspace 对象）/ Provider（`models/workspace.js` API 封装）。
- **图型**：architecture（水平组件流）+ sequence（自动保存/上传 Promise 链）；无事件流管道与单实体状态机。
- **外部边界**：外部 REST API、浏览器 FormData/DOM；不涉及 etcd/CRD。
- **部署维度**：Vite SPA 静态托管（见 docker-build 叶子）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 工作区管理架构图 | `workspace-admin-ui-architecture.html` | architecture | **showcase** |
| 自动保存与上传时序 | `workspace-admin-ui-sequence.html` | sequence | **showcase** |

- JSON IR 源文件：`json/workspace-admin-ui-architecture.json`、`json/workspace-admin-ui-sequence.json`。
- 省略说明：本叶子未生成 dataflow/lifecycle/workflow 图——文档解析是异步后台管道（归 document-pipeline 域），前端仅触发与刷新，无 ETL 血缘/单实体状态机/多角色审批泳道语义，按资源节省原则省略。
