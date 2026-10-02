# 文档管道（document-pipeline）域总览

> 本域包含 6 个叶子子系统；各叶子详情见对应文档。
> 源码基准：anything-llm v1.16.2，commit `128a0157`，纯 JavaScript ESM。

## 1. 域职责

本域负责"把任意来源的文档（上传文件/网页链接/纯文本/音频/外部知识库）解析为纯文本并入库待嵌入"。核心是 **collector 独立 Node Express 服务**（`collector/`，独立 package，name=anything-llm-document-collector），它与 server 主进程通过 HTTP + 文件系统（hotdir）交互：server 上传文件到 hotdir 后 POST 通知 collector，collector 解析成 JSON documents 回传，server 再向量化入向量库。

核心代码路径：`collector/index.js`（Express 入口）、`collector/processSingleFile/`（格式转换）、`collector/processLink/`（链接抓取）、`collector/processRawText/`、`collector/convertAudioToWav/`、`collector/extensions/`（扩展端点）、`collector/hotdir/`（临时目录）、`server/utils/DocumentManager/`、`server/jobs/`。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序图 / 数据流图 | 职责一句话 |
|------|------|--------|-------------------|-----------|
| collector-server-core | [collector-server-core.md](collector-server-core/collector-server-core.md) | [架构图](collector-server-core/collector-server-core-architecture.html) | [时序图](collector-server-core/collector-server-core-sequence.html) | collector Express 服务入口、路由、签名校验、任务接收编排 |
| file-conversion | [file-conversion.md](file-conversion/file-conversion.md) | [架构图](file-conversion/file-conversion-architecture.html) | [数据流](file-conversion/file-conversion-dataflow.html) | PDF/DOCX/PPTX/Excel/EPub/图片/mbox 等格式解析为纯文本 |
| link-scraping | [link-scraping.md](link-scraping/link-scraping.md) | — | [数据流](link-scraping/link-scraping-dataflow.html) · [时序图](link-scraping/link-scraping-sequence.html) | 网页 URL 抓取、HTML→Markdown/纯文本提取 |
| raw-text-audio | [raw-text-audio.md](raw-text-audio/raw-text-audio.md) | [架构图](raw-text-audio/raw-text-audio-architecture.html) | [数据流](raw-text-audio/raw-text-audio-dataflow.html) | 纯文本直接入库 + 音频 ffmpeg 转 16kHz WAV |
| collector-extensions-hotdir | [collector-extensions-hotdir.md](collector-extensions-hotdir/collector-extensions-hotdir.md) | [架构图](collector-extensions-hotdir/collector-extensions-hotdir-architecture.html) | [时序图](collector-extensions-hotdir/collector-extensions-hotdir-sequence.html) | resync/repo/youtube/confluence 扩展端点 + hotdir 临时目录 |
| document-manager-jobs | [document-manager-jobs.md](document-manager-jobs/document-manager-jobs.md) | [架构图](document-manager-jobs/document-manager-jobs-architecture.html) | [时序图](document-manager-jobs/document-manager-jobs-sequence.html) | server 端 DocumentManager 文档管理 + Bree 后台任务/嵌入 worker |

## 3. 域级机制细节

- **进程边界**：collector 是独立进程（可独立部署），server 通过 `CollectorApi`（`server/utils/collectorApi.js`）HTTP 调用；请求经 `verifyPayloadIntegrity` 签名校验。
- **统一响应约定**：所有 collector 路由即使失败也返回 HTTP 200 + `{success:boolean, reason:string, documents:[]}`，由 server 端按 success 字段判断。
- **hotdir 临时目录**：上传文件先落 `collector/hotdir/`（WATCH_DIRECTORY），处理完 trashFile 删除；collector 启动时 `wipeCollectorStorage()` 清空。
- **路径安全**：所有文件操作经 `path.normalize` + `isWithin` 校验，防 `..` 目录穿越。
- **嵌入 worker 隔离**：`server/jobs/embedding-worker.js` 是独立 child_process，native embedding 模型 OOM 只杀 worker 不影响主 server；通过 IPC 消息（embed/doc_starting/chunk_progress/doc_complete/all_complete）回报进度。

## 4. 域级图

本域共 12 张叶子图（6 叶 × 2 张），覆盖 architecture/sequence/dataflow 三种图型。link-scraping 叶子以 dataflow+sequence 为主（无独立静态组件拓扑，架构与 collector-server-core 重叠，按资源节省原则省略第 1 张 architecture 图）。
