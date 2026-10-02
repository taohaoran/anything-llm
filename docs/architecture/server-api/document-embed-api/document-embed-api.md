# 文档上传与嵌入 API（document-embed-api）

> 本文是 `server-api` 域下的叶子子系统文档。域级总览见 `../server-api.md`。
> 本文展开文档上传/链接抓取/嵌入与反嵌入/文件夹管理端点，不展开文档解析管道本身（见 document-pipeline 域）。
>
> 源码基准：anything-llm v1.16.2，commit `128a0157`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 创建文档文件夹 | `POST /document/create-folder`，admin/manager 角色，路径越界校验 | `server/endpoints/document.js:14-42` |
| 移动文档文件 | `POST /document/move-files`，已嵌入工作区的文件禁止移动 | `document.js:44-108` |
| 工作区文件上传 | `POST /workspace/:slug/upload`，multer 接收后交 Collector 处理 | `server/endpoints/workspaces.js:110-177` |
| 链接上传 | `POST /workspace/:slug/upload-link`，交 Collector 抓取链接 | `workspaces.js:179-220` |
| 更新嵌入 | `POST /workspace/:slug/update-embeddings`，adds/deletes 增删向量 | `workspaces.js:222-286` |
| 上传并嵌入 | `POST /workspace/:slug/upload-and-embed` | `workspaces.js:784` |
| 移除并反嵌入 | `POST /workspace/:slug/remove-and-unembed` | `workspaces.js:862` |
| 嵌入进度/队列 | `GET embed-progress`、`GET embed-queue` | `workspaces.js:979,1010` |
| 路径安全校验 | `normalizePath` + `isWithin` 防目录穿越 | `document.js:20-22,59-66` |
| 收集服务探活 | `Collector.online()` 离线则拒绝上传 | `workspaces.js:133-144,186-197` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `documentEndpoints(app)` | `document.js:12` | 注册文件夹/移动端点 |
| `CollectorApi` | `server/utils/collectorApi/index.js` | 调用外部文档收集服务（processDocument/processLink/online） |
| `Document.addDocuments(workspace, adds, userId)` | `server/models/documents.js` | 把待嵌入文件加入工作区并触发嵌入 |
| `Document.removeDocuments(workspace, deletes, userId)` | `models/documents.js` | 从工作区移除并反嵌入 |
| `Document.where({docpath:{in}})` | `models/documents.js` | 批量查文档是否已嵌入 |
| `handleFileUpload`（multer） | `utils/files/multer.js` | multipart 文件落盘中间件 |
| `embedFiles / isNativeEmbedder` | `server/utils/EmbeddingWorkerManager.js` | 原生嵌入器的独立异步嵌入通道 |

## 3. 关键调用链

**链 A：文件上传并处理（`workspaces.js:110-177`）**
1. 路由 `[validatedRequest, flexUserRoleValid([admin,manager]), handleFileUpload]`（`workspaces.js:112-116`）。
2. multer 落盘后取 `originalname`、`folderName`、`metadata`（注意 multipart 字段顺序，文本字段须在文件之前）（`workspaces.js:120-131`）。
3. `Collector.online()` 探活，离线则 500 拒绝（`workspaces.js:133-144`）。
4. `Collector.processDocument(originalname, metadata)` 交外部收集服务解析（`workspaces.js:146-153`）。
5. 若指定 folderName，`moveProcessedDocsToFolder` 移入目标文件夹；写遥测与 EventLogs（`workspaces.js:157-170`）。

**链 B：更新工作区嵌入（`workspaces.js:222-286`）**
1. 按 slug 取当前工作区（多用户带 `getWithUser`）（`workspaces.js:230-237`）。
2. 先 `Document.removeDocuments(workspace, deletes, userId)` 反嵌入删除项（`workspaces.js:239-243`）。
3. 若 `isNativeEmbedder()` 且有 adds，走 `embedFiles(...)` 原生嵌入工作器（`workspaces.js:250-264`）；否则走 `Document.addDocuments`，返回 `failedToEmbed` 与 errors 汇总（`workspaces.js:266-280`）。

**链 C：移动文件保护（`document.js:44-108`）**
1. 取所有待移动 `from` 路径，`Document.where({docpath:{in}})` 查出已嵌入文档（`document.js:50-53`）。
2. 过滤出未嵌入的 `moveableFiles` 才执行 `fs.rename`；已嵌入文件跳过并在响应中提示数量（`document.js:54-93`）。

![文档嵌入 API 架构图](document-embed-api-architecture.html)
![文档上传嵌入数据流](document-embed-api-dataflow.html)

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| multer `limit` | 全局 bodyParser 3GB | `server/index.js:52` |
| collector 地址 | `CollectorApi` 经环境变量配置收集服务端口 | `utils/collectorApi/` |
| `isNativeEmbedder()` | 选用 AnythingLLM Native Embedder 时走独立 worker | `workspaces.js:250` |
| 角色要求 | 上传/嵌入/移动需 admin 或 manager | `workspaces.js:114,181,224`、`document.js:16,46` |

## 5. 错误与重试语义

- collector 离线：500 提示"Document processing API is not online"，文件不自动处理（`workspaces.js:135-144`）。
- `processDocument` 失败：返回 `{success:false, reason}`，500 带 reason（`workspaces.js:150-153`）。
- 移动文件部分失败：`Promise.all` 任一 reject 则整体 500 "Failed to move some files"；已嵌入文件不计入失败而是提示数量（`document.js:80-100`）。
- 嵌入失败：`Document.addDocuments` 返回 `failedToEmbed[]` 与 `errors[]`，端点汇总成 message 返回 200（`workspaces.js:266-280`），不抛 500。
- 路径穿越：`isWithin` 校验失败直接抛错/拒绝（`document.js:21,63-67`）。
- 本叶子不做上传重试；嵌入重试在 DocumentManager/嵌入工作器内。

## 6. 并发细节

- multer 单文件落盘为同步/回调 I/O；移动文件用 `Promise.all(movePromises)` 并发 `fs.rename`（`document.js:80`）。
- 原生嵌入器 `embedFiles` 为异步后台任务，端点立即返回更新后的 workspace（`workspaces.js:251-263`），嵌入进度经 `embed-progress`/`embed-queue` 轮询。
- 遥测/EventLogs 为 await 串行写在响应前。
- 无锁；文件系统操作靠 `fs.existsSync`/`isWithin` 做基本防护。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- document.js 文件夹/移动端点、workspaces.js 上传/链接/嵌入更新端点、CollectorApi 调用、路径安全校验。

**Out-of-Scope（不在本仓库源码内）**
- 文档解析/转换/切分管道本身（collector 服务，document-pipeline 域）。
- 向量库写入实现（vectorDbProviders，ai-integrations 域）。
- 外部 Collector 服务进程（独立 npm 包 anything-llm-document-collector，不在本 server 进程内）。
- 对象存储/S3 等外部存储（部署侧）。

## 8. 与相邻子系统交互

- 本叶子 → ext-channels-api：收集服务是独立进程，经 HTTP 通信。
- 本叶子 → server-data：读写 Document、Workspace 模型。
- 本叶子 → ai-integrations：嵌入工作器调用 EmbeddingEngine 与 vectorDbProviders。
- 本叶子 → auth-authz-api：依赖 validatedRequest 与角色中间件。
- 上游：Web 管理端上传文件/链接。

## 9. 语言专项适配口径（TS-JS 口径）

- **capability seam 分组**：端点是 Consumer，CollectorApi 是外部 Provider，Document 模型是本地 Provider。
- **图型**：architecture（组件拓扑）+ dataflow（上传→解析→落库→向量化→向量库的数据管道，最贴合 dataflow 语义）。
- **外部边界**：图中标 `external` 的 Collector 服务与向量数据库；multer 是 Node 中间件，文件系统是 Node fs 边界。
- **事件驱动**：原生嵌入器为异步后台任务，进度经轮询端点暴露，已在正文与 dataflow 体现。
- **部署维度**：server 与 collector 是两个独立 npm 进程，经 HTTP 通信。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 文档嵌入 API 架构图 | `document-embed-api-architecture.html` | architecture | showcase |
| 上传嵌入数据流 | `document-embed-api-dataflow.html` | dataflow | showcase |

- 未生成 sequence/lifecycle：上传-处理已由 dataflow 表达；文档无单实体状态机（嵌入队列状态在 jobs 域）。
- JSON IR 位于 `json/` 目录。
