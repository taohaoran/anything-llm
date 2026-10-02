# anything-llm 项目探查事实清单（facts.md）

> 由 MainAgent 于首次分析探查阶段产出，全部分片/叶子共用，禁止各自重扫仓库。
> 生成时间基准：2026-09-27；输出根：`<repo>/docs/architecture/`（仓库根无 docs/ 目录，按默认规则）。

## 1. 身份

- 仓库：Mintplex-Labs/anything-llm（本地路径 `/Users/thr/Documents/AllProjects/OpenSource/anything-llm`）
- 版本：v1.16.2（root package.json），当前 commit `128a0157`（"Read Anthropic replies from text blocks, not the first block (#6496)"）
- 定位：全栈 AI 应用，把私有文档变成聊天机器人；"all-in-one AI application"，可自托管、多用户、多 LLM/向量库接入
- 许可证：MIT
- 语言判定：**JavaScript 为主（ESM，`"type": "module"`，Node >= 18，本机 Node v22.23.2）**；open-computer/cli 含 TypeScript（tsconfig.json）
  → 按 skill 语言专项，**纯 JS 项目并入 TS-JS 口径，读 `references/language-ts.md`**

## 2. 规模（源码文件数与总行数，排除 node_modules/dist/build/vendor）

| 组件 | 源码文件数 | 总行数 |
|---|---|---|
| server/ | 518 | 113,732 |
| frontend/ | 611 | 139,187 |
| collector/ | 73 | 14,028 |
| open-computer/ | 53 | 10,916 |
| 合计 | ~1,255 | ~278k |

- 无 go.mod/pom.xml/build.gradle；组件各自独立 package.json（server=anything-llm-server、frontend=anything-llm-frontend、collector=anything-llm-document-collector；open-computer 无顶层 package.json，cli 子目录有 package.json）
- open-computer/cli 为 TS（src 含 vm.ts/ssh.ts/registry.ts/config.ts/commands/）

## 3. 结构（一层目录树）

```
anything-llm/
├── server/          # NodeJS Express API 服务端（endpoints/models/utils/jobs/middleware/prisma/storage/swagger/__tests__）
├── frontend/        # viteJS + React SPA（src/pages、components、hooks、contexts、models、utils、locales）
├── collector/       # NodeJS Express 文档收集/解析服务（processSingleFile/processLink/processRawText/convertAudioToWav/extensions/hotdir）
├── open-computer/   # 子产品：给 agent 的隔离虚拟电脑（cli/TS + services + master(iso/qemu/setup)）
├── docker/          # Docker 构建与部署说明
├── cloud-deployments/ # 各云平台部署模板
├── locales/         # i18n 语言包
├── images/、extras/、.github/、.devcontainer/
├── browser-extension/  # git 子模块（未 checkout，空目录）
└── embed/              # git 子模块（未 checkout，空目录）
```

- server 子目录：endpoints/（API 路由层）、models/（Prisma 模型封装）、utils/（业务逻辑：AiProviders/EmbeddingEngines/vectorDbProviders/agents/DocumentManager/MCP/...）、jobs/（后台任务）、middleware/、prisma/（schema）、storage/（models/documents/assets 资产目录）、swagger/
- frontend 子目录：pages/（Admin/GeneralSettings/Invite/Login/Main/OnboardingFlow/WorkspaceChat/WorkspaceSettings）、components/（LLMSelection/EmbeddingSelection/VectorDBSelection/Modals/Sidebar/...）、models/（前端模型）、hooks/utils/contexts/
- collector 子目录：processSingleFile/convert、processLink/convert+helpers、processRawText、convertAudioToWav、extensions（含 resync）、hotdir、middleware、utils、__tests__
- open-computer 子目录：cli/src（commands/config/registry/ssh/vm）、services（interface-service/memory-manager/extensions/public）、master（iso/qemu/setup）、scripts、assets

## 4. 功能清单（README Feature List 摘录）

- Dynamic Model Routing：按规则自动把会话路由到最优 provider/model
- Automatic & User Managed Memories：LLM 记忆（工作区级）
- Scheduled Tasks：cron 周期任务/提示词，带完整 agent 能力
- Intelligent Skill Selection：无限工具 + 每查询最多省 80% token
- No-code AI Agent builder（Agent Flows）
- MCP 兼容（MCP servers 管理）
- 多模态支持（闭源+开源 LLM）
- 自定义 AI Agents（工作区内联网浏览等）
- 多用户实例与权限（Docker 版）
- 可嵌入网页的聊天 widget（embed 子模块，Docker 版）
- 多文档类型支持（PDF/TXT/DOCX 等，collector 管道）
- 聊天 UI：拖拽上传 + 来源引用
- 生产级云部署；全量开发者 API
- 支持的生态：LLM（OpenAI/Azure/Anthropic/Gemini/Ollama/LM Studio/DeepSeek/Mistral/Groq/Cohere/xAI/Moonshot/Minimax/Cerebras/本地 llama.cpp 等 40+）；Embedder（AnythingLLM Native/OpenAI/Gemini/Ollama/Cohere/Voyage 等）；向量库（LanceDB 默认/PGVector/Astra/Pinecone/Chroma/Weaviate/Qdrant/Milvus/Zilliz）；STT/TTS（浏览器内置/Piper/OpenAI/ElevenLabs）

## 5. 约束

- 仓库内无 AGENTS.md；用户级规则：**文件名/目录名一律英文短横线（-），禁止中文短横线**；输出语言一律简体中文（skill 硬性要求）
- 用户偏好：**架构分析产出必须为三层结构（顶层系统级文档区 + 各域目录 + 叶子），禁止扁平化输出**（与 skill Step 6 输出结构一致）
- 输出根：仓库根无 docs/ 目录 → `<repo>/docs/architecture/`；`_exploration/` 即本目录

## 6. 域与叶子盘点（两级拆分，39 叶子 / 8 域）

| 域 | 叶子（leaf） | 源码依据 |
|---|---|---|
| server-api（API 服务域） | request-routing / auth-authz-api / workspace-chat-api / document-embed-api / admin-system-api / agent-mcp-api / ext-channels-api（7） | server/index.js、endpoints/、middleware/ |
| server-data（数据模型域） | prisma-schema / user-workspace-models / document-vector-models / auth-settings-models / jobs-memory-models（5） | server/prisma、server/models/ |
| ai-integrations（AI 集成域） | llm-providers / embedding-engines-rerankers / vector-db-providers / multimodal-stt-tts / text-splitting-pricing（5） | server/utils/AiProviders、EmbeddingEngines、vectorDbProviders、ImageGenerators、SpeechToText、TextToSpeech、TextSplitter、helpers |
| document-pipeline（文档管道域） | collector-server-core / file-conversion / link-scraping / raw-text-audio / collector-extensions-hotdir / document-manager-jobs（6） | collector/ 全部、server/utils/DocumentManager、server/jobs/ |
| agents-mcp（智能体域） | agent-runtime / agent-tools / agent-flows-engine / mcp-bridge / memory-external-channels（5） | server/utils/agents（aibitat）、agentFlows、MCP、memories、BackgroundWorkers、PushNotifications、telegramBot |
| frontend（前端域） | app-shell / chat-interface / workspace-admin-ui / auth-onboarding-ui / settings-console-models（5） | frontend/src/ |
| open-computer（Open Computer 域） | oc-cli / oc-services / oc-master（3） | open-computer/ |
| deployment-ops（部署运维域） | docker-build / cloud-deployments / i18n-locales（3） | docker/、cloud-deployments/、locales/ |

- 归并支撑库：server/utils/logger、http、helpers、files、router、prisma 等归入所属叶子/系统级，不单独成篇
- 子模块 browser-extension/、embed/ 未 checkout（空目录），标注为"不在本仓库源码内"的第三方子模块，不拆叶子

## 7. 工具链

- archify CLI：`/tmp/archify-upstream/archify/bin/archify.mjs`（已就绪，Node ≥ 18，本机 v22）
- skill 目录：`/Users/thr/Library/Application Support/DoubaoWork/Profile 2/.doubaowork/agent_mode/workspace/.user_skills/archify-codebase-analysis`（scripts/render-diagram.sh、check-links.py、regression-check.py、references/ 全量在此）
