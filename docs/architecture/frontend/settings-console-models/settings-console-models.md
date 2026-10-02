# settings-console-models 设置控制台与模型选择（settings-console-models）

> 本文是 `frontend` 域下的叶子子系统文档。域级总览见 `../frontend.md`。
> 本文只展开**系统级设置控制台**：SettingsSidebar 导航、LLM/向量库/Embedder/STT-TTS 偏好配置、用户与工作区管理、API 密钥、40+ provider 模型选择器；
> 工作区级设置见 `workspace-admin-ui`，登录引导见 `auth-onboarding-ui`。
>
> 源码基准：`frontend/`，commit `128a015`，JavaScript（ESM，TS-JS 口径）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 设置侧边导航 | `SettingsSidebar` + `MenuOption` 聚合全部设置页入口，按角色显示 | `frontend/src/components/SettingsSidebar/index.jsx:24-50` |
| LLM 偏好设置 | 选择默认 LLM provider 与模型 | `pages/GeneralSettings/LLMPreference/`、路由 `/settings/llm-preference`（`main.jsx:70-77`） |
| 多 provider 选项组件 | 40+ provider 专属表单（OpenAI/Anthropic/Ollama/Gemini/Bedrock 等） | `components/LLMSelection/`（40+ `*Options.jsx` + `LLMProviderOption`/`LLMItem`） |
| Embedder/切分偏好 | 嵌入模型与文本切分配置 | `pages/GeneralSettings/EmbeddingPreference/`、`EmbeddingTextSplitterPreference/` |
| 向量库偏好 | 系统级向量库选择与连接 | `pages/GeneralSettings/VectorDatabase/`、`components/VectorDBSelection/` |
| 图像生成/STT/TTS | 图像生成、转录、语音偏好 | `pages/GeneralSettings/ImageGenerationPreference/`、`TranscriptionPreference/`、`AudioPreference/` |
| 用户管理 | 用户列表/增删/角色 | `pages/Admin/Users/index.jsx:65` `fetchUsers`，路由 `ManagerRoute`（`main.jsx:325-330`） |
| 工作区管理 | 系统级工作区列表 | `pages/Admin/Workspaces/`（`main.js:331-339`） |
| API 密钥 | 开发者 API key 管理 | `pages/GeneralSettings/ApiKeys/`（`main.jsx:257-264`） |
| 模型路由 | Dynamic Model Routing 规则配置 | `pages/GeneralSettings/ModelRouters/`（`main.jsx:266-283`） |
| 安全/隐私/界面/品牌 | 安全策略、隐私数据、界面与品牌定制 | `pages/GeneralSettings/Security/`、`PrivacyAndData/`、`Settings/Interface/`、`Settings/Branding/` |
| 调度任务/外部连接 | Scheduled Jobs、Telegram、Mobile、CommunityHub | `pages/GeneralSettings/ScheduledJobs/`、`Connections/`、`CommunityHub/` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `SettingsSidebar()` | `SettingsSidebar/index.jsx:24` | 设置导航外壳 |
| `LLMProviderOption` / `LLMItem` | `components/LLMSelection/` | provider 列表项与单 provider 表单容器 |
| `*Options.jsx`（40+） | `components/LLMSelection/` | 各 LLM provider 的参数表单 |
| `VectorDBSelection` / `EmbeddingSelection` | `components/` | 向量库/嵌入模型选择器 |
| `System.*` | `models/system.js` | 系统设置读写 API（keys/onboarding/local-files 等） |
| `Admin/Users fetchUsers` | `Admin/Users/index.jsx:65` | 用户列表拉取 |

## 3. 关键调用链

**链 1：配置一个 LLM provider**
1. 管理员经 `AdminRoute` 进入 `/settings/llm-preference`（`main.jsx:70-77`）。
2. 页面渲染 `LLMSelection`，列出 `LLMProviderOption`；选中某 provider 渲染对应 `*Options.jsx` 表单（`components/LLMSelection/`）。
3. 用户填 API key/参数，`AutosaveForm` 防抖提交 → `models/system.js` 封装 POST 到后端系统配置接口。
4. 成功 toast；失败回显错误。

**链 2：设置导航分级可见**
1. `SettingsSidebar` 用 `useUser()` 取角色（`SettingsSidebar/index.jsx:30`）。
2. 菜单项经 `MenuOption` 按 admin/manager 角色与 `CanViewChatHistoryProvider` 控制显隐。
3. 路由层再由 `AdminRoute`/`ManagerRoute` 兜底（`main.jsx`）。

**链 3：用户管理**
1. `Admin/Users` 挂载后 `fetchUsers()`（`Admin/Users/index.jsx:70`）拉用户列表。
2. 增删/改角色经 `models/admin.js` 调后端用户接口。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---|---|---|
| `MultiUserMode` | 决定用户管理页是否可用 | `models/system.js:61` keys |
| `CanViewChatHistory` | 控制聊天历史查看权限菜单项 | `SettingsSidebar/index.jsx:23` |
| provider 选项组件 | 40+ provider 各自的默认参数与 key 字段 | `components/LLMSelection/*Options.jsx` |
| 路由角色守卫 | AdminRoute=admin/单机，ManagerRoute=manager+ | `main.jsx` 各 settings 路由 |

## 5. 错误与重试语义

- **保存失败**：AutosaveForm 回显错误 toast，不自动重试。
- **拉取用户/设置失败**：`.catch(() => [])` 类回退，页面空态展示（`Admin/Users`）。
- **provider key 校验**：后端校验失败时表单回显错误；前端不做重试。
- 无指数退避；配置类操作均为单次请求。

## 6. 并发细节

- React 状态驱动：每个设置页独立 `useState`；`AutosaveForm` 防抖。
- `SettingsSidebar` 移动端抽屉用 `useState(showSidebar/showBgOverlay)` + `setTimeout` 300ms 延迟背景遮罩（`SettingsSidebar/index.jsx:38-49`）。
- 无后台轮询；应用版本号经 `useAppVersion` 拉取（`SettingsSidebar/index.jsx:28`）。
- 40+ provider 组件代码分割，按选中项懒渲染。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `frontend/src/pages/GeneralSettings/`、`pages/Admin/`、`components/SettingsSidebar/`、`components/LLMSelection/`、`components/EmbeddingSelection/`、`components/VectorDBSelection/`、`models/system.js`、`models/admin.js`

**Out-of-Scope（不在本仓库源码内）**
- 各 LLM/Embedder/向量库的实际调用与计费 → server/utils（ai-integrations 域）
- 用户/工作区/密钥的数据模型 → server/prisma（server-data 域）
- 第三方 LLM/向量库服务（OpenAI 等）为外部系统，不在本仓库源码内

## 8. 与相邻子系统交互

- **app-shell → settings-console-models**：`/settings/*` 经 `AdminRoute/ManagerRoute` 分级放行。
- **settings-console-models → chat-interface**：系统 LLM 偏好与模型选择器决定聊天默认模型。
- **settings-console-models → workspace-admin-ui**：工作区级配置继承/覆盖系统偏好。
- **settings-console-models → 后端**：经 `models/system.js`/`admin.js` 调 REST，外部 API 不在前端源码内。

## 9. 语言专项适配口径（JavaScript / TS-JS）

- **capability seam**：Provider（40+ `*Options.jsx` provider 表单，Service Definition）/ Consumer（设置页消费）/ Service（后端配置存储，外部）。这是典型的"多 provider 工厂"seam。
- **图型**：architecture（水平组件流）+ sequence（配置保存 Promise 链）；无事件流管道与单实体状态机。
- **外部边界**：外部 REST API、第三方 LLM/向量库服务（不在本仓库源码内）；不涉及 etcd/CRD。
- **部署维度**：Vite SPA 静态托管（见 docker-build 叶子）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 设置控制台架构图 | `settings-console-models-architecture.html` | architecture | **showcase** |
| 配置 LLM Provider 时序 | `settings-console-models-sequence.html` | sequence | **showcase** |

- JSON IR 源文件：`json/settings-console-models-architecture.json`、`json/settings-console-models-sequence.json`。
- 省略说明：本叶子未生成 dataflow/lifecycle/workflow 图——配置是表单保存链（已用 sequence 表达），无 ETL 管道、单实体状态机或多角色审批泳道语义，按资源节省原则省略。
