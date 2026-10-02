# 文档格式转换（file-conversion）

> 本文是 `document-pipeline` 域下的叶子子系统文档。域级总览见 `../document-pipeline.md`。
> 本文只展开"各类文档格式（PDF/DOCX/PPTX/Excel/EPub/图片/mbox/音频）解析为纯文本"的转换器集合。
>
> 源码基准：anything-llm v1.16.2，commit `128a0157`，纯 JavaScript ESM。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 文件类型分发 | `SUPPORTED_FILETYPE_CONVERTERS` 按扩展名映射到转换器；未知类型若为文本则按 .txt 处理 | `collector/processSingleFile/index.js:55-80`、`utils/constants.js` |
| PDF 解析 | asPDF 目录，解析 PDF 为文本 | `collector/processSingleFile/convert/asPDF/` |
| DOCX 解析 | asDocx | `collector/processSingleFile/convert/asDocx.js` |
| EPub 解析 | asEPub | `collector/processSingleFile/convert/asEPub.js` |
| 图片 OCR | asImage | `collector/processSingleFile/convert/asImage.js` |
| mbox 解析 | asMbox | `collector/processSingleFile/convert/asMbox.js` |
| Office MIME | asOfficeMime（PPTX 等） | `collector/processSingleFile/convert/asOfficeMime.js` |
| Xlsx 解析 | asXlsx | `collector/processSingleFile/convert/asXlsx.js` |
| 纯文本 | asTxt | `collector/processSingleFile/convert/asTxt.js` |
| 音频 | asAudio | `collector/processSingleFile/convert/asAudio.js` |
| 路径安全 | `normalizePath`/`isWithin` 防目录穿越；`trashFile` 删除失败文件 | `collector/utils/files.js`、`processSingleFile/index.js:31-39,73` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `processSingleFile(targetFilename, options, metadata)` | `processSingleFile/index.js:24` | 入口：路径校验→扩展名分发→调对应 converter→返回 documents |
| `SUPPORTED_FILETYPE_CONVERTERS` | `utils/constants.js` | 扩展名→转换器映射表 |
| 各 `asXxx.js` | `processSingleFile/convert/` | 统一约定：读文件→解析→返回 `{pageContent, ...metadata}` |
| `writeToServerDocuments({data, filename})` | `utils/files.js` | 把解析结果写成 JSON 文档存到 server documents 目录 |

## 3. 关键调用链

1. **PDF 处理**：`processSingleFile`（`processSingleFile/index.js:24`）→ 校验 `fullFilePath` 在 WATCH_DIRECTORY 内（`:31-39`）→ 取 `fileExtension`（`:55`）→ 查 `SUPPORTED_FILETYPE_CONVERTERS`（`:65`）→ require 对应 converter → 解析 → `writeToServerDocuments`。
2. **未知类型降级**：扩展名不在映射表时，若 `isTextType(fullFilePath)` 则当 .txt 处理（`:66-70`）；否则 trashFile 并返回不支持（`:73-78`）。
3. **parseOnly 模式**：`/parse` 路由传 `options.parseOnly=true`（`collector/index.js:88-92`），转换器不写 server documents。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `WATCH_DIRECTORY` | 上传文件根 | `utils/constants.js` |
| `RESERVED_FILES` | `["__HOTDIR__.md"]` 不处理 | `processSingleFile/index.js:13` |
| `options.absolutePath` | 内部用，跳过 WATCH_DIRECTORY 校验 | `processSingleFile/index.js:20` |

## 5. 错误与重试语义

- 文件不存在 → 返回 `{success:false, reason:"File does not exist"}`（`processSingleFile/index.js:48-53`）。
- 保留文件名（__HOTDIR__.md）拒绝处理（`:41-46`）。
- 不支持的非文本扩展名 → trashFile 删除临时文件（`:73`）。
- converter 内部异常由 `/process` 路由 catch 兜底（`collector/index.js:62-69`）。

## 6. 并发细节

- 单文件串行处理；无并发池。
- 解析大文件（PDF/音频）时占用内存/CPU，错误隔离靠 collector 独立进程。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `collector/processSingleFile/`、`collector/utils/files.js`、`utils/constants.js`

**Out-of-Scope（不在本仓库源码内）**
- `pdf-parse`/`mammoth`/`officeparser`/OCR 等第三方 npm 解析库
- ffmpeg 系统二进制（音频转码，见 raw-text-audio）

## 8. 与相邻子系统交互

- 上游 → 本叶子：collector-server-core 的 `/process`、`/parse` 路由。
- 本叶子 → 下游：解析产出纯文本+元数据 → 写 server documents 目录 → server 端向量化（embedding 叶子）。

## 9. 语言专项适配口径（TS/JS 口径）

- **capability seam 分组**：converter 注册表（`SUPPORTED_FILETYPE_CONVERTERS`）+ 各 asXxx 实现，是典型 seam。
- **图类型侧重**：dataflow（文件→按扩展名分发→各转换器→纯文本文档 管道）+ architecture。
- **外部边界**：第三方解析 npm 库、ffmpeg。
- **部署维度**：随 collector 进程。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 格式转换分发架构 | `file-conversion-architecture.html` | architecture | showcase（一次通过） |
| 文件解析数据流 | `file-conversion-dataflow.html` | dataflow | showcase（bottom-channel 路由一次通过） |

JSON IR 源文件位于 `json/` 目录。
