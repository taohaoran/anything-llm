# 文本切分器与 Token 定价（text-splitting-pricing）

> 本文是 `ai-integrations` 域下的叶子子系统文档。域级总览见 `../ai-integrations.md`。
> 本文只展开"文档文本切分（chunking）、token 计数与模型定价估算"，不展开具体 LLM/向量库适配。
>
> 源码基准：anything-llm v1.16.2，commit `128a0157`，纯 JavaScript ESM。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 通用文本切分器 | 按 chunkSize/chunkOverlap 把文档切成带元数据的 chunk | `server/utils/TextSplitter/index.js:26` `TextSplitter` |
| 切分配置 | chunkPrefix / chunkSize(默认1000) / chunkOverlap(默认20) / chunkHeaderMeta | `TextSplitter/index.js:31-40` 构造函数 |
| 文档元数据结构 | DocumentMetadata typedef（id/url/title/docAuthor/description/docSource/chunkSource/published/wordCount/pageContent/token_count_estimate） | `TextSplitter/index.js:2-19` |
| Token 精确计数 | 基于 js-tiktoken 的 TokenManager 单例，按模型选 encoding | `server/utils/helpers/tiktoken.js:11` `TokenManager` |
| 模型定价查询 | 按 provider/model 查输入/输出 token 单价（USD per 1M tokens） | `server/utils/helpers/modelPricing/index.js:1` |
| 免费 provider 识别 | ollama/lmstudio/localai 等本地 provider 定价为 0 | `modelPricing/index.js:26` `FREE_PROVIDERS` |
| 成本估算 | 按 prompt/completion token 数算 CostBreakdown | `modelPricing/index.js:13` `CostBreakdown` typedef |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `TextSplitter` | `TextSplitter/index.js:26` | 切分器类；`#setSplitter(config)` 内部选切分策略；输出带 DocumentMetadata 的 chunk 数组 |
| `TokenManager` | `helpers/tiktoken.js:11` | 单例（按 model 缓存 encoder）；`#getEncodingFromModel(model)` 选 tiktoken 编码；负责精确 tokenize 与 chat history 反向 tokenize |
| `ModelCost` / `CostBreakdown` | `modelPricing/index.js:3,13` | 单价与成本明细 typedef |
| `FREE_PROVIDERS` | `modelPricing/index.js:26` | 本地免费 provider 白名单 |

## 3. 关键调用链

1. **文档切分**：collector/server 拿到纯文本后 `new TextSplitter({chunkSize, chunkOverlap, chunkHeaderMeta})`（`TextSplitter/index.js:31`）→ 切分输出每个 chunk 带 `pageContent` 与 `token_count_estimate` → 交给嵌入引擎向量化（见 embedding 叶子）。
2. **Token 计数**：聊天构造 prompt 时 `TokenManager` 单例（`tiktoken.js:13`）按当前 model 选 encoder → 精确计算 prompt token 数，用于判断是否超上下文窗口、触发 `compressMessages`。
3. **成本估算**：LLMPerformanceMonitor 拿到 usage（prompt_tokens/completion_tokens）→ modelPricing 查单价 → 算 CostBreakdown 记录。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| chunkSize | 默认 1000 | `TextSplitter/index.js:34` |
| chunkOverlap | 默认 20 | `TextSplitter/index.js:35` |
| chunkPrefix / chunkHeaderMeta | 空 / null | `TextSplitter/index.js:33,36` |
| TokenManager model | 默认 `gpt-3.5-turbo` | `tiktoken.js:12` |

## 5. 错误与重试语义

- TokenManager 单例按 model 缓存（`tiktoken.js:13`），切 model 时重建 encoder。
- 定价查不到时 fallback（cache_read 回退 input、reasoning 回退 output，见 `modelPricing/index.js:8-11` typedef 注释）。
- 本层无网络调用（定价数据本地 JSON），无重试。

## 6. 并发细节

- TokenManager 静态单例（`tiktoken.js:13`），跨请求复用 encoder（encoder 创建昂贵）。
- TextSplitter 无状态实例，可并发安全使用。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `server/utils/TextSplitter/index.js`
- `server/utils/helpers/tiktoken.js`、`modelPricing/index.js`

**Out-of-Scope（不在本仓库源码内）**
- `js-tiktoken` npm 包及其 BPE 编码表
- 模型定价数据源（models.dev 约定，JSON 随仓库或远程）

## 8. 与相邻子系统交互

- 上游 → 本叶子：collector/file-conversion 切分纯文本；DocumentManager 入库。
- 本叶子 → 下游：切分后的 chunk 交给 embedding-engines 向量化；token 计数喂给 llm-providers 的 prompt window 判断与压缩。

## 9. 语言专项适配口径（TS/JS 口径）

- **capability seam 分组**：本叶子是工具/支撑库型，归入系统级机制；TextSplitter 为无状态工厂类，TokenManager 为单例。
- **图类型侧重**：dataflow（文档文本 → 切分 → chunk → 向量化 管道）+ architecture（切分/计数/定价三件套静态结构）。
- **外部边界**：js-tiktoken npm 包。
- **部署维度**：随 server 进程，无独立服务。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 切分/计数/定价架构 | `text-splitting-pricing-architecture.html` | architecture | standard（showcase 回退 standard；render 退出码 0） |
| 文档切分数据流 | `text-splitting-pricing-dataflow.html` | dataflow | showcase（初版边标签与节点重叠，改用 bottom-channel 路由后一次通过） |

JSON IR 源文件位于 `json/` 目录。
