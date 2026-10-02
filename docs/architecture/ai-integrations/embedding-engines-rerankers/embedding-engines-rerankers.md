# 嵌入引擎与重排序器适配（embedding-engines-rerankers）

> 本文是 `ai-integrations` 域下的叶子子系统文档。域级总览见 `../ai-integrations.md`。
> 本文只展开"嵌入引擎（embedding）与重排序器（reranker）的统一接口与工厂"，不展开 LLM 提供商（见 llm-providers）与向量库（见 vector-db-providers）。
>
> 源码基准：anything-llm v1.16.2，commit `128a0157`，纯 JavaScript ESM。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 嵌入引擎工厂 | 按 `EMBEDDING_ENGINE` 环境变量 switch-case 实例化对应嵌入器 | `server/utils/helpers/index.js:272` `getEmbeddingEngineSelection()` |
| 14 个嵌入引擎适配 | openai / azure / localai / ollama / native / lmstudio / cohere / voyageai / litellm / mistral / generic-openai / gemini / openrouter / lemonade | `server/utils/EmbeddingEngines/<engine>/index.js` |
| Native 本地嵌入器 | 用 onnxruntime-node + Xenova/all-MiniLM-L6-v2 本地推理，无需外部 API | `server/utils/EmbeddingEngines/native/index.js:7` `NativeEmbedder` |
| 分块嵌入并发控制 | `maxConcurrentChunks` 限制单次并发块数，`toChunks()` 分批，`reportEmbeddingProgress()` 上报进度 | `EmbeddingEngines/openAi/index.js:14,39`、`native/index.js:267` |
| 统一嵌入接口 | 每个嵌入器类实现 `embedTextInput(textInput)`（单条）与 `embedChunks(textChunks)`（批量） | `EmbeddingEngines/openAi/index.js:24,31` |
| 本地重排序器 | `NativeEmbeddingReranker` 用 Xenova/ms-marco-MiniLM-L-6-v2 对检索结果重排 | `server/utils/EmbeddingRerankers/native/index.js:6` |
| 模型缓存与懒加载 | ONNX pipeline 静态单例复用，模型下载到 `storage/models/`，失败回退 cdn.anythingllm.com | `native/index.js:13-15,37,43` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `getEmbeddingEngineSelection()` | `helpers/index.js:272` | 工厂：按 `EMBEDDING_ENGINE` require 并 new 对应类；default 走 NativeEmbedder |
| `OpenAiEmbedder` / `NativeEmbedder`（及 12 个同类） | `EmbeddingEngines/openAi/index.js:3`、`native/index.js:7` | duck-typing 统一接口：`embedTextInput(textInput)` → `{embedding:[...]}`；`embedChunks(textChunks)` → `{embeddings:[[...]]}` |
| `NativeEmbeddingReranker` | `EmbeddingRerankers/native/index.js:6` | 静态单例模型/tokenizer；`maxBatchSize=10`；对向量库召回的候选文档重排打分 |
| `toChunks(arr, size)` / `reportEmbeddingProgress()` | `server/utils/helpers.js` | 把大数组按并发上限切批；通过 SSE/进度回调上报嵌入进度 |
| `SUPPORTED_NATIVE_EMBEDDING_MODELS` | `EmbeddingEngines/native/constants.js` | 本地支持的嵌入模型清单（含 chunkPrefix/queryPrefix/maxConcurrentChunks） |

## 3. 关键调用链

1. **文档入库嵌入**：DocumentManager 对切分后的文本 chunk 调 `embedder.embedChunks(textChunks)`（`openAi/index.js:31`）→ `toChunks(textChunks, this.maxConcurrentChunks)`（`:39`）分批 → 每批调外部嵌入 API → 累加 `reportEmbeddingProgress()`（`native/index.js:292`）→ 返回 `{embeddings:[[...]]}` 交给向量库写入。
2. **查询时嵌入**：聊天主链路 `llm.embedTextInput(query)`（见 llm-providers 叶子，`openAi/index.js:293`）委托给 `this.embedder.embedTextInput()` → 返回单条向量 → 传给向量库 similaritySearch。
3. **检索后重排**：`LanceDb.rerankedSimilarityResponse()`（`vectorDbProviders/lance/index.js:99`）先做 similaritySearch 取 topN×放大倍数候选 → `new NativeEmbeddingReranker()`（`:108`）对候选打分重排 → 返回重排后 topN。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `EMBEDDING_ENGINE` | 未设置时 default → NativeEmbedder | `helpers/index.js:274,322` |
| `OPEN_AI_EMBEDDING_MODEL_PREF` 等 | 各引擎专属模型名 | 各 `EmbeddingEngines/<engine>/index.js` 构造函数 |
| NativeEmbedder 模型 | 默认 `Xenova/all-MiniLM-L6-v2` | `native/index.js:8` |
| `maxConcurrentChunks` | OpenAI=500；Native 按 modelInfo | `openAi/index.js:14`、`native/index.js:52` |
| Reranker 批次 | 静态 `maxBatchSize=10` | `EmbeddingRerankers/native/index.js:10` |
| `STORAGE_DIR` | 模型缓存目录根；未设置时用 `server/storage/models` | `native/index.js:43-47` |

## 5. 错误与重试语义

- **NativeEmbedder 模型下载失败**：构造期检测 `modelDownloaded = fs.existsSync(modelPath)`（`native/index.js:49`）；下载失败回退 `#fallbackHost = cdn.anythingllm.com`（`:37`），注释声明该镜像"随时可能下线"。
- **外部嵌入 API 错误**：`embedChunks` 内分批调用，单批失败会向上抛出（OpenAI SDK 错误），本层不自动重试。
- **ONNX session 不可释放**：注释明确 onnxruntime-node 1.14 的 `dispose()` 是空操作，因此用静态 `#pipelines` Map 全局只建一次，避免内存泄漏（`native/index.js:11-15`）。
- **重排内存**：`maxBatchSize=10` 限制单次 batch，注释说明 onnxruntime-node 不把内存还给 OS，最大 batch 决定进程 RSS 峰值（`EmbeddingRerankers/native/index.js:7-8`）。

## 6. 并发细节

- 嵌入器实例通常每请求新建（`getEmbeddingEngineSelection()` 无缓存），但 NativeEmbedder 的 ONNX pipeline 是静态共享单例（`#pipelines` Map），跨请求复用。
- 批量嵌入用 `toChunks` 串行分批（非 Promise.all 并发），配合 `maxConcurrentChunks` 控制对外部 API 的速率与资源占用。
- Reranker 用静态 `#initializationPromise` 保证模型只初始化一次（`EmbeddingRerankers/native/index.js:14`）。
- 进度上报 `reportEmbeddingProgress` 通过事件/SSE 推给前端（与 collector/server 的进度通道对接）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `server/utils/helpers/index.js` 的 `getEmbeddingEngineSelection()` 工厂
- `server/utils/EmbeddingEngines/` 全部 14 个引擎适配
- `server/utils/EmbeddingRerankers/native/` 本地重排序器
- Native onnx 模型加载/缓存/回退逻辑

**Out-of-Scope（不在本仓库源码内）**
- OpenAI / Cohere / Voyage / Gemini 等外部嵌入云 API
- `onnxruntime-node` / `@xenova/transformers` npm 包（node_modules）
- HuggingFace 模型权重下载源（`huggingface.co`）与备用镜像 `cdn.anythingllm.com`
- 向量库存储与相似度检索实现（见 vector-db-providers 叶子）

## 8. 与相邻子系统交互

- 上游 → 本叶子：`getLLMProvider()`（llm-providers 工厂）把 embedder 注入每个 LLM 实例（`helpers/index.js:138`）；DocumentManager 入库时直接调 `embedder.embedChunks`。
- 本叶子 → 下游：各引擎 `require("openai")` 等 SDK → 外部嵌入 API；Native 引擎 `require("onnxruntime-node")` 本地推理。
- 本叶子 → 向量库：产出向量后交给 `getVectorDbClass()` 写入 / 检索（vector-db-providers 叶子）；Reranker 被 LanceDb 召回后调用。

## 9. 语言专项适配口径（TS/JS 口径）

- **capability seam 分组**：Service Definition = `getEmbeddingEngineSelection()` 工厂 + `embedTextInput/embedChunks` duck-typing 接口；Provider = `EmbeddingEngines/*` 14 个实现 + `EmbeddingRerankers/native`；Consumer = DocumentManager、LLM provider、检索链路。
- **图类型侧重**：architecture（工厂+多引擎静态拓扑）+ dataflow（文档 chunk → 分批 → 嵌入 → 向量 的数据管道）；不画 sequence（嵌入调用链与 llm-providers 重复）。
- **外部边界**：外部嵌入 API（OpenAI/Cohere/Voyage）、HuggingFace 模型下载、Node 内置 fs/path。
- **部署维度**：Native 嵌入/重排模型随 server 进程首次启动下载到 `storage/models/`；无独立包。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 嵌入引擎工厂与重排架构 | `embedding-engines-rerankers-architecture.html` | architecture | showcase（一次通过） |
| 文档嵌入数据流 | `embedding-engines-rerankers-dataflow.html` | dataflow | standard（初版 flow 缺 label 报 schema 错；补 label 后因边标签与节点重叠，删去进度/外部 API 两条次要边后 standard 通过；render 退出码 0） |

JSON IR 源文件位于 `json/` 目录。
