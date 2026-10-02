# anything-llm 系统架构文档

> 基于 anything-llm 源码（v1.16.2，commit 128a015，MIT，纯 JavaScript ESM 全栈项目，约 1,255 个源码文件 / 27.8 万行）深度分析产出，
> 覆盖系统级、8 个域、39 个叶子子系统的功能、问题域、系统边界、架构图、时序图、数据流图与生命周期图。
> 所有图表由 archify 渲染为自包含交互式 HTML。全部文档为简体中文。

## 文档导航

### 系统级

| 文档 | 说明 | 图表 |
|------|------|------|
| [system-overview.md](system-overview.md) | 功能总览、解决的问题、系统边界、核心代码映射、语言适配口径 | [系统架构图](system-architecture.html) · [聊天时序图](system-chat-sequence.html) · [RAG 数据流图](system-rag-dataflow.html) · [Agent 生命周期图](system-agent-lifecycle.html) |

---

### server-api（API 服务域）—— 7 叶子

| 子系统 | 文档 | 架构图 | 第二图 | 职责一句话 |
|--------|------|--------|--------|-----------|
| 域总览 | [server-api.md](server-api/server-api.md) | — | — | Express API 路由、认证、聊天、文档、管理、Agent、外部渠道端点 |
| request-routing | [MD](server-api/request-routing/request-routing.md) | [架构图](server-api/request-routing/request-routing-architecture.html) | [时序图](server-api/request-routing/request-routing-sequence.html) | Express 路由装配与中间件链 |
| auth-authz-api | [MD](server-api/auth-authz-api/auth-authz-api.md) | [架构图](server-api/auth-authz-api/auth-authz-api-architecture.html) | [时序图](server-api/auth-authz-api/auth-authz-api-sequence.html) | 登录/注册/JWT/多用户权限 |
| workspace-chat-api | [MD](server-api/workspace-chat-api/workspace-chat-api.md) | [架构图](server-api/workspace-chat-api/workspace-chat-api-architecture.html) | [时序图](server-api/workspace-chat-api/workspace-chat-api-sequence.html) | 工作区聊天与 SSE 流式响应 |
| document-embed-api | [MD](server-api/document-embed-api/document-embed-api.md) | [架构图](server-api/document-embed-api/document-embed-api-architecture.html) | [数据流图](server-api/document-embed-api/document-embed-api-dataflow.html) | 文档上传/嵌入/向量库操作端点 |
| admin-system-api | [MD](server-api/admin-system-api/admin-system-api.md) | [架构图](server-api/admin-system-api/admin-system-api-architecture.html) | [时序图](server-api/admin-system-api/admin-system-api-sequence.html) | 系统设置/用户/工作区管理端点 |
| agent-mcp-api | [MD](server-api/agent-mcp-api/agent-mcp-api.md) | [架构图](server-api/agent-mcp-api/agent-mcp-api-architecture.html) | [时序图](server-api/agent-mcp-api/agent-mcp-api-sequence.html) | Agent 执行与 MCP 配置端点 |
| ext-channels-api | [MD](server-api/ext-channels-api/ext-channels-api.md) | [架构图](server-api/ext-channels-api/ext-channels-api-architecture.html) | [时序图](server-api/ext-channels-api/ext-channels-api-sequence.html) | Slack/Telegram/embed 外部渠道接入 |

### server-data（数据模型域）—— 5 叶子

| 子系统 | 文档 | 架构图 | 第二图 | 职责一句话 |
|--------|------|--------|--------|-----------|
| 域总览 | [server-data.md](server-data/server-data.md) | — | — | Prisma schema 与全部数据模型 |
| prisma-schema | [MD](server-data/prisma-schema/prisma-schema.md) | [架构图](server-data/prisma-schema/prisma-schema-architecture.html) | [数据流图](server-data/prisma-schema/prisma-schema-dataflow.html) | Prisma schema 定义与 SQLite 连接 |
| user-workspace-models | [MD](server-data/user-workspace-models/user-workspace-models.md) | [架构图](server-data/user-workspace-models/user-workspace-models-architecture.html) | [时序图](server-data/user-workspace-models/user-workspace-models-sequence.html) | User/Workspace/WorkspaceUser 模型 |
| document-vector-models | [MD](server-data/document-vector-models/document-vector-models.md) | [架构图](server-data/document-vector-models/document-vector-models-architecture.html) | [数据流图](server-data/document-vector-models/document-vector-models-dataflow.html) | Document/DocumentChunk 等模型 |
| auth-settings-models | [MD](server-data/auth-settings-models/auth-settings-models.md) | [架构图](server-data/auth-settings-models/auth-settings-models-architecture.html) | [时序图](server-data/auth-settings-models/auth-settings-models-sequence.html) | 系统设置与认证配置存储模型 |
| jobs-memory-models | [MD](server-data/jobs-memory-models/jobs-memory-models.md) | [架构图](server-data/jobs-memory-models/jobs-memory-models-architecture.html) | [时序图](server-data/jobs-memory-models/jobs-memory-models-sequence.html) | 后台任务/记忆/通知模型 |

### ai-integrations（AI 集成域）—— 5 叶子

| 子系统 | 文档 | 架构图 | 第二图 | 职责一句话 |
|--------|------|--------|--------|-----------|
| 域总览 | [ai-integrations.md](ai-integrations/ai-integrations.md) | — | — | 多 LLM/嵌入/向量库/多模态/切分定价 |
| llm-providers | [MD](ai-integrations/llm-providers/llm-providers.md) | [架构图](ai-integrations/llm-providers/llm-providers-architecture.html) | [时序图](ai-integrations/llm-providers/llm-providers-sequence.html) | 40+ LLM 提供商统一调用接口 |
| embedding-engines-rerankers | [MD](ai-integrations/embedding-engines-rerankers/embedding-engines-rerankers.md) | [架构图](ai-integrations/embedding-engines-rerankers/embedding-engines-rerankers-architecture.html) | [数据流图](ai-integrations/embedding-engines-rerankers/embedding-engines-rerankers-dataflow.html) | 嵌入引擎与重排序器适配 |
| vector-db-providers | [MD](ai-integrations/vector-db-providers/vector-db-providers.md) | [架构图](ai-integrations/vector-db-providers/vector-db-providers-architecture.html) | [时序图](ai-integrations/vector-db-providers/vector-db-providers-sequence.html) | 10+ 向量数据库提供商适配 |
| multimodal-stt-tts | [MD](ai-integrations/multimodal-stt-tts/multimodal-stt-tts.md) | [架构图](ai-integrations/multimodal-stt-tts/multimodal-stt-tts-architecture.html) | [时序图](ai-integrations/multimodal-stt-tts/multimodal-stt-tts-sequence.html) | 图像生成/STT/TTS 多模态能力 |
| text-splitting-pricing | [MD](ai-integrations/text-splitting-pricing/text-splitting-pricing.md) | [架构图](ai-integrations/text-splitting-pricing/text-splitting-pricing-architecture.html) | [数据流图](ai-integrations/text-splitting-pricing/text-splitting-pricing-dataflow.html) | 文本切分/token 计数/定价计算 |

### document-pipeline（文档管道域）—— 6 叶子

| 子系统 | 文档 | 架构图 | 第二图 | 职责一句话 |
|--------|------|--------|--------|-----------|
| 域总览 | [document-pipeline.md](document-pipeline/document-pipeline.md) | — | — | Collector 服务与文档摄入全管道 |
| collector-server-core | [MD](document-pipeline/collector-server-core/collector-server-core.md) | [架构图](document-pipeline/collector-server-core/collector-server-core-architecture.html) | [时序图](document-pipeline/collector-server-core/collector-server-core-sequence.html) | Collector 独立服务核心与任务编排 |
| file-conversion | [MD](document-pipeline/file-conversion/file-conversion.md) | [架构图](document-pipeline/file-conversion/file-conversion-architecture.html) | [数据流图](document-pipeline/file-conversion/file-conversion-dataflow.html) | PDF/DOCX/PPTX 等格式转换 |
| link-scraping | [MD](document-pipeline/link-scraping/link-scraping.md) | [数据流图](document-pipeline/link-scraping/link-scraping-dataflow.html) | [时序图](document-pipeline/link-scraping/link-scraping-sequence.html) | 网页链接抓取与内容提取 |
| raw-text-audio | [MD](document-pipeline/raw-text-audio/raw-text-audio.md) | [架构图](document-pipeline/raw-text-audio/raw-text-audio-architecture.html) | [数据流图](document-pipeline/raw-text-audio/raw-text-audio-dataflow.html) | 纯文本与音频输入处理 |
| collector-extensions-hotdir | [MD](document-pipeline/collector-extensions-hotdir/collector-extensions-hotdir.md) | [架构图](document-pipeline/collector-extensions-hotdir/collector-extensions-hotdir-architecture.html) | [时序图](document-pipeline/collector-extensions-hotdir/collector-extensions-hotdir-sequence.html) | Collector 扩展机制与 hotdir 监控 |
| document-manager-jobs | [MD](document-pipeline/document-manager-jobs/document-manager-jobs.md) | [架构图](document-pipeline/document-manager-jobs/document-manager-jobs-architecture.html) | [时序图](document-pipeline/document-manager-jobs/document-manager-jobs-sequence.html) | DocumentManager 与后台任务调度 |

### agents-mcp（智能体域）—— 5 叶子

| 子系统 | 文档 | 架构图 | 第二图 | 职责一句话 |
|--------|------|--------|--------|-----------|
| 域总览 | [agents-mcp.md](agents-mcp/agents-mcp.md) | — | — | Agent 运行时/工具/Flows/MCP/记忆 |
| agent-runtime | [MD](agents-mcp/agent-runtime/agent-runtime.md) | [架构图](agents-mcp/agent-runtime/agent-runtime-architecture.html) | [生命周期图](agents-mcp/agent-runtime/agent-runtime-lifecycle.html) | aibitat Agent 运行时与对话循环 |
| agent-tools | [MD](agents-mcp/agent-tools/agent-tools.md) | [架构图](agents-mcp/agent-tools/agent-tools-architecture.html) | [时序图](agents-mcp/agent-tools/agent-tools-sequence.html) | 内置工具集注册与执行 |
| agent-flows-engine | [MD](agents-mcp/agent-flows-engine/agent-flows-engine.md) | [架构图](agents-mcp/agent-flows-engine/agent-flows-engine-architecture.html) | [数据流图](agents-mcp/agent-flows-engine/agent-flows-engine-dataflow.html) | Agent Flows 线性步骤编排引擎 |
| mcp-bridge | [MD](agents-mcp/mcp-bridge/mcp-bridge.md) | [架构图](agents-mcp/mcp-bridge/mcp-bridge-architecture.html) | [时序图](agents-mcp/mcp-bridge/mcp-bridge-sequence.html) | MCP 协议桥接与工具暴露 |
| memory-external-channels | [MD](agents-mcp/memory-external-channels/memory-external-channels.md) | [架构图](agents-mcp/memory-external-channels/memory-external-channels-architecture.html) | [时序图](agents-mcp/memory-external-channels/memory-external-channels-sequence.html) | 记忆系统/后台工作者/推送通知/telegramBot |

### frontend（前端域）—— 5 叶子

| 子系统 | 文档 | 架构图 | 第二图 | 职责一句话 |
|--------|------|--------|--------|-----------|
| 域总览 | [frontend.md](frontend/frontend.md) | — | — | React + Vite SPA 全部页面与组件 |
| app-shell | [MD](frontend/app-shell/app-shell.md) | [架构图](frontend/app-shell/app-shell-architecture.html) | [时序图](frontend/app-shell/app-shell-sequence.html) | 应用外壳/路由/布局/全局状态 |
| chat-interface | [MD](frontend/chat-interface/chat-interface.md) | [架构图](frontend/chat-interface/chat-interface-architecture.html) | [时序图](frontend/chat-interface/chat-interface-sequence.html) | 聊天界面/流式渲染/来源引用 |
| workspace-admin-ui | [MD](frontend/workspace-admin-ui/workspace-admin-ui.md) | [架构图](frontend/workspace-admin-ui/workspace-admin-ui-architecture.html) | [时序图](frontend/workspace-admin-ui/workspace-admin-ui-sequence.html) | 工作区管理 UI 与文档管理 |
| auth-onboarding-ui | [MD](frontend/auth-onboarding-ui/auth-onboarding-ui.md) | [架构图](frontend/auth-onboarding-ui/auth-onboarding-ui-architecture.html) | [时序图](frontend/auth-onboarding-ui/auth-onboarding-ui-sequence.html) | 认证与首次设置引导 UI |
| settings-console-models | [MD](frontend/settings-console-models/settings-console-models.md) | [架构图](frontend/settings-console-models/settings-console-models-architecture.html) | [时序图](frontend/settings-console-models/settings-console-models-sequence.html) | 设置控制台/LLM/向量库/用户管理 |

### open-computer（Open Computer 域）—— 3 叶子

| 子系统 | 文档 | 架构图 | 第二图 | 职责一句话 |
|--------|------|--------|--------|-----------|
| 域总览 | [open-computer.md](open-computer/open-computer.md) | — | — | Agent 虚拟电脑子产品 |
| oc-cli | [MD](open-computer/oc-cli/oc-cli.md) | [架构图](open-computer/oc-cli/oc-cli-architecture.html) | [时序图](open-computer/oc-cli/oc-cli-sequence.html) | CLI 入口与命令分发（含 TS） |
| oc-services | [MD](open-computer/oc-services/oc-services.md) | [架构图](open-computer/oc-services/oc-services-architecture.html) | [数据流图](open-computer/oc-services/oc-services-dataflow.html) | 浏览器控制/屏幕操作/OCR 服务层 |
| oc-master | [MD](open-computer/oc-master/oc-master.md) | [架构图](open-computer/oc-master/oc-master-architecture.html) | [生命周期图](open-computer/oc-master/oc-master-lifecycle.html) | 主控编排与 VM 生命周期 |

### deployment-ops（部署运维域）—— 3 叶子

| 子系统 | 文档 | 架构图 | 第二图 | 职责一句话 |
|--------|------|--------|--------|-----------|
| 域总览 | [deployment-ops.md](deployment-ops/deployment-ops.md) | — | — | Docker 构建/云部署/i18n |
| docker-build | [MD](deployment-ops/docker-build/docker-build.md) | [架构图](deployment-ops/docker-build/docker-build-architecture.html) | [数据流图](deployment-ops/docker-build/docker-build-dataflow.html) | Docker 多阶段构建与镜像编排 |
| cloud-deployments | [MD](deployment-ops/cloud-deployments/cloud-deployments.md) | [架构图](deployment-ops/cloud-deployments/cloud-deployments-architecture.html) | [时序图](deployment-ops/cloud-deployments/cloud-deployments-sequence.html) | 各云平台部署模板 |
| i18n-locales | [MD](deployment-ops/i18n-locales/i18n-locales.md) | [架构图](deployment-ops/i18n-locales/i18n-locales-architecture.html) | [时序图](deployment-ops/i18n-locales/i18n-locales-sequence.html) | 多语言 locale 与 i18n 机制 |

---

## 产出统计

| 层级 | MD | HTML 图 | JSON IR | 说明 |
|---|---|---|---|---|
| 系统级 | 2（README + system-overview） | 4 | 4 | 架构/时序/数据流/生命周期 |
| 域总览 | 8 | — | — | 每域含叶子索引表 |
| 叶子 | 39 | 78 | 78 | 每叶子 ≥2 张图 |
| **合计** | **49** | **82** | **82** | |

## 覆盖范围与说明

- **覆盖**：8 域 39 叶子，覆盖 server/frontend/collector/open-computer 全部核心源码
- **未覆盖**：`browser-extension/`、`embed/` 为未 checkout 的 git 子模块（空目录），标注"不在本仓库源码内"，未拆叶子
- **质量档位**：系统级图 2 张 showcase + 2 张 standard；叶子级图目标 showcase，落 standard 的均在对应叶子 MD 第 10 节披露失败检查名与修复动作
- **事实校正**：Agent Flows 实为线性 `config.steps[]` 数组顺序执行，非节点-边条件分支图（详见 agent-flows-engine 叶子 MD）
- **语言适配口径**：纯 JavaScript ESM 项目并入 TS-JS 口径，各叶子 MD 第 9 节落写
- **探查事实**：`_exploration/facts.md` 保留全仓库盘点，供后续迭代分析复用
