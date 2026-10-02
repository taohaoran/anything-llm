# DocumentManager 与后台任务调度（document-manager-jobs）

> 本文是 `document-pipeline` 域下的叶子子系统文档。域级总览见 `../document-pipeline.md`。
> 本文只展开 server 端 DocumentManager（文档读取/pin/上下文组装）与 `server/jobs/` 后台任务调度。
>
> 源码基准：anything-llm v1.16.2，commit `128a0157`，纯 JavaScript ESM。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| DocumentManager | 工作区级文档管理：读取 pinned 文档、组装上下文 | `server/utils/DocumentManager/index.js:9` `class DocumentManager` |
| pinned 文档 | 读取工作区置顶文档并按 maxTokens 截断 | `DocumentManager/index.js:20-55` |
| 文档存储路径 | 生产用 `STORAGE_DIR/documents`，开发用 `server/storage/documents` | `DocumentManager/index.js:3-7` |
| 嵌入 worker | 独立子进程跑文档嵌入循环，OOM 只杀 worker 不影响主服务 | `server/jobs/embedding-worker.js:1-25` |
| IPC 进度协议 | parent↔worker 传 embed/add_files/batch_starting/chunk_progress/doc_complete 等 | `embedding-worker.js:11-22` |
| 同步监控文档 | `sync-watched-documents.js` 处理 stale 同步队列 | `server/jobs/sync-watched-documents.js:11` |
| 清理任务 | cleanup-generated-files / cleanup-generated-images / cleanup-orphan-documents | `server/jobs/cleanup-*.js` |
| 记忆抽取 | `extract-memories.js` 从对话抽记忆 | `server/jobs/extract-memories.js` |
| 定时任务 | `run-scheduled-job.js` 跑 cron 调度任务 | `server/jobs/run-scheduled-job.js` |
| Telegram 聊天 | `handle-telegram-chat.js` | `server/jobs/handle-telegram-chat.js` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `DocumentManager({workspace, maxTokens})` | `DocumentManager/index.js:9` | 构造；读 pinned docs、组装上下文 |
| embedding-worker（child process） | `server/jobs/embedding-worker.js` | 独立进程嵌入；通过 process IPC 与父进程通信 |
| `EmbeddingWorkerManager`（父侧） | BackgroundService/Bree | 按需 spawn worker，接收 IPC 进度 |
| `DocumentSyncQueue`/`DocumentSyncRun`/`DocumentVectors`/`Document` | `server/models/` | Prisma 模型封装 |
| `CollectorApi` | `server/utils/collectorApi.js` | server 调 collector 的 HTTP 客户端 |

## 3. 关键调用链

1. **嵌入任务调度**：server 端用户上传文档 → 写 collector hotdir → `EmbeddingWorkerManager` spawn `embedding-worker.js` 子进程（`embedding-worker.js:1` 注释）→ parent 发 `{type:"embed", files, workspaceSlug,...}`（`:12`）→ worker 逐个文档：调 collector 解析 → 嵌入 → 写向量库 → 回报 `doc_starting/chunk_progress/doc_complete`（`:15-19`）→ 全部完成发 `all_complete`（`:22`）。
2. **监控文档同步**：`sync-watched-documents.js` 启动（`:11`）→ 查 `DocumentSyncQueue.staleDocumentQueues()`（`:15`）→ 无队列则退出（`:16-18`）→ `new CollectorApi()` 探活（`:20-22`）→ 对 stale 文档重新抓源并更新向量。
3. **聊天时文档上下文**：聊天请求 → `new DocumentManager({workspace})` → `pinnedDocuments()`（`DocumentManager/index.js:20`）查 Prisma → 逐个 `fs.readFileSync` 解析 JSON → 按 maxTokens 截断 → 作为 contextTexts 传给 LLM。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `STORAGE_DIR` | 生产文档存储根 | `DocumentManager/index.js:6` |
| `maxTokens` | DocumentManager 上下文 token 上限，默认 Infinity | `DocumentManager/index.js:11` |
| Bree/BackgroundService | 后台任务调度器（cron/队列） | `server/utils/BackgroundWorkers/` |

## 5. 错误与重试语义

- **OOM 隔离**：嵌入 worker 是独立子进程，native embedding 模型 OOM 只杀 worker，主 server 存活（`embedding-worker.js:1-4` 注释）。
- worker 处理失败发 `doc_failed` 消息（`:20`），不中断后续文档。
- collector 不可达 → sync 任务记日志退出（`sync-watched-documents.js:21-22`）。
- pinned 文档路径越界 → `isWithin` 检查后跳过（`DocumentManager/index.js:38-41`）。

## 6. 并发细节

- **子进程隔离**：嵌入在 child_process 跑，IPC 双向消息；worker 内文件串行处理，运行中可接收 `add_files` 追加（`embedding-worker.js:13`）。
- Bree 调度后台任务并发跑；每个 job 是独立脚本。
- DocumentManager 读 pinned 文档用 `for await` 串行读文件。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `server/utils/DocumentManager/`、`server/jobs/`、`server/models/`（Prisma 封装）

**Out-of-Scope（不在本仓库源码内）**
- Bree/BackgroundWorkers 调度框架（node_modules）
- Prisma/PostgreSQL 数据库进程
- collector 进程（独立，见 collector-server-core）

## 8. 与相邻子系统交互

- 上游 → 本叶子：聊天请求用 DocumentManager 组上下文；admin 触发嵌入/同步任务。
- 本叶子 → 下游：`CollectorApi` HTTP 调 collector（document-pipeline 域）；`getVectorDbClass()` 写向量（ai-integrations 域）；Prisma 读写文档/向量记录。

## 9. 语言专项适配口径（TS/JS 口径）

- **capability seam 分组**：DocumentManager 为无状态类；jobs 为 Bree 注册的独立脚本。
- **图类型侧重**：sequence（嵌入 worker IPC 时序）+ architecture（DocumentManager+jobs+collector 边界）。
- **外部边界**：PostgreSQL、collector 子进程、Node child_process。
- **部署维度**：jobs 随 server 进程；嵌入 worker 为 child_process。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| DocumentManager 与任务架构 | `document-manager-jobs-architecture.html` | architecture | showcase（初版 jobs→worker 竖边方向不合规，删去该隐式边后通过） |
| 嵌入 worker IPC 时序 | `document-manager-jobs-sequence.html` | sequence | showcase（一次通过） |

JSON IR 源文件位于 `json/` 目录。
