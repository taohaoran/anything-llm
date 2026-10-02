# 纯文本与音频输入处理（raw-text-audio）

> 本文是 `document-pipeline` 域下的叶子子系统文档。域级总览见 `../document-pipeline.md`。
> 本文只展开"纯文本直接入库"与"音频文件转 WAV 预处理"。
>
> 源码基准：anything-llm v1.16.2，commit `128a0157`，纯 JavaScript ESM。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 纯文本入库 | `processRawText(textContent, metadata)` 构造文档对象并写 server documents | `collector/processRawText/index.js:62` |
| 元数据规范化 | `METADATA_KEYS.possible` 对 url/title/author/description/published 等做默认值与归一化 | `processRawText/index.js:18-58` |
| 音频转 WAV | `convertAudioToWav(filename)` 用 ffmpeg 转 16kHz 单声道 WAV | `collector/convertAudioToWav/index.js:10` |
| WAV 供 STT | 转好的 wav 后续喂给 Whisper STT | `collector/utils/WhisperProviders/ffmpeg.js` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `processRawText(textContent, metadata)` | `processRawText/index.js:62` | 校验非空→构造 DocumentMetadata→`writeToServerDocuments` |
| `METADATA_KEYS.possible` | `processRawText/index.js:18` | 各元数据字段的规范化函数集合 |
| `convertAudioToWav(filename)` | `convertAudioToWav/index.js:10` | ffmpeg 转码 |
| `FFMPEGWrapper` | `collector/utils/WhisperProviders/ffmpeg.js` | ffmpeg 子进程封装 |
| `stripAndSlug(input)` | `processRawText/index.js:8` | slugify 文件名 |

## 3. 关键调用链

1. **纯文本入库**：`/process-raw-text` 路由（`collector/index.js:179`）→ `processRawText(textContent, metadata)`（`processRawText/index.js:62`）→ 校验 textContent 非空（`:65-69`）→ 校验 title 非空（`:73-78`）→ 构造 data（含 id=uuid、pageContent、wordCount、token_count_estimate，`:82-96`）→ `writeToServerDocuments`（`:98`）。
2. **音频转码**：`/util/convert-audio-to-wav`（`collector/index.js:154`）→ `convertAudioToWav(filename)`（`convertAudioToWav/index.js:10`）→ 路径安全校验（`:20-25`）→ ffmpeg 转 16kHz mono → 返回 wavFilename。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| 音频目标格式 | 16kHz 单声道 WAV | `convertAudioToWav/index.js:8` 注释 |
| `published` 归一化 | 空/非法→当前时间戳 | `processRawText/index.js:46-58` |

## 5. 错误与重试语义

- textContent 空 → `{success:false, reason:"textContent was empty"}`（`processRawText/index.js:65-69`）。
- metadata.title 非字符串/空 → 失败（`:73-78`），因 title 派生 url/filename。
- 音频路径越界 → 拒绝（`convertAudioToWav/index.js:20-25`）。
- 源文件转码后 trash；wav 由调用方读取后 trash。

## 6. 并发细节

- ffmpeg 为子进程调用（`FFMPEGWrapper`），异步 await。
- 纯文本处理纯内存，无外部调用。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `collector/processRawText/`、`collector/convertAudioToWav/`、`collector/utils/WhisperProviders/`

**Out-of-Scope（不在本仓库源码内）**
- ffmpeg 系统二进制（外部进程）
- Whisper STT 模型（见 ai-integrations 的 STT 适配）

## 8. 与相邻子系统交互

- 上游 → 本叶子：collector-server-core 路由 `/process-raw-text`、`/util/convert-audio-to-wav`。
- 本叶子 → 下游：纯文本文档写 server documents → 向量化；wav 喂 STT 转写。

## 9. 语言专项适配口径（TS/JS 口径）

- **图类型侧重**：dataflow（文本→元数据规范化→文档；音频→ffmpeg→WAV）+ architecture。
- **外部边界**：ffmpeg 子进程。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 纯文本与音频处理架构 | `raw-text-audio-architecture.html` | architecture | showcase（一次通过） |
| 音频转码数据流 | `raw-text-audio-dataflow.html` | dataflow | showcase（一次通过） |

JSON IR 源文件位于 `json/` 目录。
