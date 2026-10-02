# anything-llm 系统级总览

> 基于 anything-llm 源码（v1.16.2，commit 128a015，MIT）深度分析产出。
> 覆盖系统级、8 个域、39 个叶子子系统的功能、问题域、系统边界与核心图表。
> 所有图表由 archify 渲染为自包含交互式 HTML。

## 1. 项目概述

**定位**（引用 README）：anything-llm 是一款 "all-in-one AI application"，把私有文档变成可聊天的机器人；支持自托管、多用户、多 LLM/向量库接入、自定义 Agent、MCP 兼容与可嵌入聊天 widget。

**架构范式**：纯 JavaScript（ESM，`"type":"module"`，Node >= 18）全栈单体应用，由四个可独立运行的组件构成——前端 SPA（React + Vite）、Server API（Express + Prisma）、Collector 服务（文档收集解析，独立进程）、Open Computer（Agent 虚拟电脑子产品）。数据层默认 SQLite + LanceDB，可插拔为 PostgreSQL + 各类商用向量库。

**代码规模**：

| 组件 | 源码文件数 | 总行数 | 说明 |
|---|---|---|---|
| server/ | 518 | 113,732 | Express API 服务端 |
| frontend/ | 611 | 139,187 | React + Vite SPA |
| collector/ | 73 | 14,028 | 文档收集解析服务 |
| open-computer/ | 53 | 10,916 | Agent 虚拟电脑子产品 |
| **合计** | **~1,255** | **~278k** | |

## 2. 功能总览

按领域列功能模块（标注域与叶子归属）：

| 功能域 | 核心能力 | 归属叶子 |
|---|---|---|
| API 服务 | Express 路由装配、JWT/Session 认证、工作区聊天（SSE 流式）、文档上传嵌入、系统管理、Agent/MCP 端点、外部渠道接入 | server-api 域 7 叶子 |
| 数据模型 | Prisma schema、用户/工作区模型、文档/向量模型、认证设置模型、任务/记忆模型 | server-data 域 5 叶子 |
| AI 集成 | 40+ LLM 提供商适配、嵌入/重排引擎、10+ 向量数据库提供商、多模态（图像/STT/TTS）、文本切分与定价 | ai-integrations 域 5 叶子 |
| 文档管道 | Collector 独立服务、文件格式转换、网页链接抓取、纯文本/音频处理、扩展与 hotdir、DocumentManager 与后台任务 | document-pipeline 域 6 叶子 |
| Agent/MCP | aibitat Agent 运行时、内置工具集、Agent Flows 线性步骤编排、MCP 桥接、记忆与外部渠道（telegram/PushNotifications） | agents-mcp 域 5 叶子 |
| 前端 | 应用外壳与路由、聊天界面（流式渲染+来源引用）、工作区管理 UI、认证与引导、设置控制台 | frontend 域 5 叶子 |
| Open Computer | CLI 入口、服务层（浏览器控制/OCR）、主控编排（VM 生命周期） | open-computer 域 3 叶子 |
| 部署运维 | Docker 多阶段构建、云平台部署模板、i18n 多语言 | deployment-ops 域 3 叶子 |

## 3. 解决的问题

| 用户痛点 | anything-llm 解法 |
|---|---|
| 私有文档无法被大模型利用 | 文档摄入管道（Collector 解析→切分→嵌入→向量库）+ RAG 检索增强生成 |
| LLM/向量库厂商锁定 | capability seam 设计：AiProviders / EmbeddingEngines / vectorDbProviders 各自为提供商注册表+工厂，支持 40+ LLM、10+ 向量库热插拔 |
| 单用户限制，团队无法协作 | 多用户实例、工作区隔离、角色权限、邀请机制 |
| Agent 工具生态封闭 | MCP 协议兼容，可接入外部 MCP 服务器；内置网页搜索/代码执行等工具 |
| 文档解析耗时阻塞主服务 | Collector 独立进程，通过 HTTP 与文件系统和 Server 解耦 |
| 部署复杂 | Docker 一键部署、多云平台模板、默认 SQLite+LanceDB 零外部依赖 |
| 无法嵌入第三方网站 | embed widget（git 子模块）+ 外部渠道 API（Slack/Telegram） |

## 4. 系统边界

### 上边界（用户/第三方接入）
- 浏览器用户通过前端 SPA 访问
- 第三方系统通过 Developer API（Bearer Key）、Slack/Telegram webhook、embed widget 接入
- MCP 客户端通过 MCP 协议连接本系统暴露的工具

### 下边界（基础设施）
- Node.js >= 18 运行时
- SQLite（默认）或 PostgreSQL（Prisma ORM）
- LanceDB（默认）或 Pinecone/Chroma/Qdrant/Milvus/Weaviate 等向量库
- 文件系统存储（documents/assets）
- Docker 容器运行时（生产部署）

### 内边界（本仓库 vs 扩展/外部）
- **本仓库源码内**：server/、frontend/、collector/、open-computer/、docker/、cloud-deployments/、locales/
- **不在本仓库源码内**：
  - `browser-extension/`、`embed/`：git 子模块，未 checkout（空目录）
  - 具体 LLM 云服务（OpenAI/Anthropic/Gemini 等）、向量库云服务（Pinecone 等）
  - Open Computer 中经 stdin RPC 拉起的 `pi` CLI、qemu/lima 虚拟化
  - 外部 MCP 服务器、Slack/Telegram 平台、Docker Hub

### 侧边界
- 多实例部署：每个 Docker 容器为独立实例，数据不跨实例共享
- 工作区隔离：文档、聊天历史、Agent 配置均按工作区隔离

### 不做什么
- 不提供模型训练/微调能力
- 不内置向量数据库计算引擎（依赖第三方 LanceDB 等）
- 不做移动端原生应用（前端为响应式 Web）
- 不实现多集群/联邦学习

## 5. 系统架构图说明

![系统架构图](system-architecture.html)

系统分为四层：

1. **客户端层**：用户浏览器通过 HTTP/SSE 访问前端 SPA（React + Vite）；外部渠道（Slack/Telegram/embed）通过 webhook/API 接入
2. **服务端层（本仓库核心）**：Server API（Express + Prisma）为核心枢纽，负责路由、认证、业务逻辑；Collector 服务独立进程处理文档解析；Agent 运行时（aibitat + Flows）执行智能体任务
3. **数据层**：SQLite（Prisma ORM）存储元数据、用户、会话、配置；向量数据库（默认 LanceDB）存储文档嵌入向量
4. **外部依赖**：LLM 提供商（40+ 模型）、嵌入/重排引擎、MCP/外部工具——均不在本仓库源码内

关键交互：
- 主路径 1（聊天）：用户→前端→Server API→LLM 提供商（流式响应）
- 主路径 2（文档摄入）：文档→Collector 服务→切分→嵌入引擎→向量数据库
- Server API 通过 Prisma 读写 SQLite，通过向量库提供商接口读写向量库
- Agent 运行时由 Server API 触发，通过 MCP 桥接查询外部工具

## 6. 核心时序图说明

![工作区聊天请求时序](system-chat-sequence.html)

工作区聊天请求（RAG + 流式响应）的完整调用链：

1. **用户输入**：用户在浏览器输入问题，前端 SPA 发送
2. **API 请求**：前端 `POST /workspace/:id/chat` 到 Server API
3. **向量检索**：Server API 将用户问题向量化后，向向量数据库检索 top-k 相似文档块
4. **构造提示词**：Server API 将检索到的文档块作为上下文，与系统提示词、聊天历史拼接，调用 LLM 提供商（stream 模式）
5. **流式返回**：LLM 提供商逐 token 流式返回，Server API 通过 SSE 逐块转发给前端
6. **前端渲染**：前端实时渲染流式消息，并附带来源引用
7. 会话历史异步保存到 SQLite（图中省略，详见 workspace-chat-api 叶子）

## 7. 系统数据流图说明

![文档摄入与 RAG 数据流](system-rag-dataflow.html)

数据流经五个阶段：

1. **数据源**：上传文件（PDF/DOCX/TXT 等）、网页链接、纯文本/音频
2. **收集解析**：Collector 服务接收任务，格式转换模块将各类文档解析为纯文本（音频转 WAV 后 STT）
3. **切分嵌入**：TextSplitter 将纯文本切分为 chunk，嵌入引擎向量化
4. **存储检索**：向量写入向量数据库；元数据（文档信息、工作区关联）写入 SQLite；聊天时从向量库检索 top-k，从 SQLite 取上下文
5. **生成输出**：LLM 推理生成回复，前端渲染流式输出

主路径（emphasis）：上传文件→Collector→TextSplitter→嵌入引擎→向量数据库→LLM→前端渲染。

## 8. Agent 执行生命周期说明

![Agent 执行会话生命周期](system-agent-lifecycle.html)

Agent 执行会话的状态流转：

1. **已创建**（start）：用户触发 Agent 任务，会话入队
2. **思考规划**（active，LLM 推理）：aibitat 运行时调用 LLM 推理下一步动作；若需工具则进入工具调用，若可直接回复则进入生成回复
3. **工具调用**（active，循环，tools lane）：执行内置工具或 MCP 工具，结果返回后回到思考规划，形成"思考→工具→结果→再思考"循环
4. **生成回复**（active）：LLM 生成最终回复
5. **已完成**（success）：输出结束，会话终止

失败与重试（图中省略，详见 agent-runtime 叶子）：推理出错或工具失败进入失败状态，可重试回到思考规划；超过最大迭代次数强制终止。

## 9. 语言适配口径

本项目主语言为 **JavaScript（ESM）**，按 archify skill 规则并入 **TS-JS 口径**：

- **capability seam 分组**：AiProviders / EmbeddingEngines / vectorDbProviders 各自为提供商注册表+工厂模式，是典型的 capability seam；纯 JS duck-typing，接口约定靠 JSDoc 与方法名（VectorDB 是唯一有真 abstract 基类的 seam）
- **图型侧重**：architecture + sequence + dataflow 为主图型；事件驱动链（EventEmitter / 回调 / Promise / SSE）进 sequence；Agent 循环状态机进 lifecycle
- **外部边界标注**：外部 LLM API / 向量库 API / 浏览器 API / Node 内置模块 / SQLite / Electron / Webhook / MCP 均标注"不在本仓库源码内"
- **部署维度**：用包发布 + 依赖图 + Docker 镜像而非单二进制；server/frontend/collector 各自独立 package.json
- open-computer/cli 内含 TypeScript，亦按 TS-JS 口径处理，不另读其他语言专项

## 10. 图表清单与质量档位

| 图表 | 类型 | 质量档位 | 说明 |
|---|---|---|---|
| [系统架构图](system-architecture.html) | architecture | **showcase** | 四层组件拓扑，10 组件 9 连接 |
| [聊天请求时序图](system-chat-sequence.html) | sequence | **showcase** | RAG + SSE 流式响应，5 参与者 8 消息 |
| [文档与 RAG 数据流图](system-rag-dataflow.html) | dataflow | standard | 五阶段管道，12 节点 10 流；标签间距未达 showcase |
| [Agent 执行生命周期图](system-agent-lifecycle.html) | lifecycle | standard | 5 状态 5 转换 + 工具循环；失败重试分支在 MD 文字说明 |

系统级图允许 standard 档（组件多、跨层连接复杂为预期内），所有 render 退出码 0、HTML 均约 800KB 自包含交互式。
