# AI 集成（ai-integrations）域总览

> 本域包含 5 个叶子子系统；各叶子详情见对应文档。
> 源码基准：anything-llm v1.16.2，commit `128a0157`，纯 JavaScript ESM（Node >= 18）。

## 1. 域职责

本域是 anything-llm 的"能力接缝（capability seam）"层：通过统一的工厂函数 + duck-typing 接口，把 40+ 家 LLM、14 种嵌入引擎、10 类向量库、多模态 STT/TTS/图像生成以及文本切分/Token 定价等能力适配成系统内部一致的调用接口。所有 provider 均按环境变量（`LLM_PROVIDER`/`EMBEDDING_ENGINE`/`VECTOR_DB`/`STT_PROVIDER`/`TTS_PROVIDER`/`IMAGE_GEN_PROVIDER`）懒加载 require，避免启动期加载全部第三方 SDK。

核心代码路径：`server/utils/helpers/index.js`（三个工厂函数 `getLLMProvider`/`getVectorDbClass`/`getEmbeddingEngineSelection`）、`server/utils/AiProviders/`、`EmbeddingEngines/`、`vectorDbProviders/`、`ImageGenerators/`、`SpeechToText/`、`TextToSpeech/`、`TextSplitter/`。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序图 / 数据流图 | 职责一句话 |
|------|------|--------|-------------------|-----------|
| llm-providers | [llm-providers.md](llm-providers/llm-providers.md) | [架构图](llm-providers/llm-providers-architecture.html) | [时序图](llm-providers/llm-providers-sequence.html) | 40+ LLM 提供商工厂、统一聊天/流式接口、动态模型路由 |
| embedding-engines-rerankers | [embedding-engines-rerankers.md](embedding-engines-rerankers/embedding-engines-rerankers.md) | [架构图](embedding-engines-rerankers/embedding-engines-rerankers-architecture.html) | [数据流](embedding-engines-rerankers/embedding-engines-rerankers-dataflow.html) | 14 种嵌入引擎适配 + 本地 ONNX 重排序器 |
| vector-db-providers | [vector-db-providers.md](vector-db-providers/vector-db-providers.md) | [架构图](vector-db-providers/vector-db-providers-architecture.html) | [时序图](vector-db-providers/vector-db-providers-sequence.html) | 10 类向量库适配、抽象基类、集合/相似度检索 |
| multimodal-stt-tts | [multimodal-stt-tts.md](multimodal-stt-tts/multimodal-stt-tts.md) | [架构图](multimodal-stt-tts/multimodal-stt-tts-architecture.html) | [时序图](multimodal-stt-tts/multimodal-stt-tts-sequence.html) | 图像生成、STT、TTS 三类多模态工厂 |
| text-splitting-pricing | [text-splitting-pricing.md](text-splitting-pricing/text-splitting-pricing.md) | [架构图](text-splitting-pricing/text-splitting-pricing-architecture.html) | [数据流](text-splitting-pricing/text-splitting-pricing-dataflow.html) | 文本切分、tiktoken Token 计数、模型定价估算 |

## 3. 域级机制细节

- **统一工厂模式**：三类核心工厂（LLM/Embedder/VectorDB）均在 `server/utils/helpers/index.js` 用 switch-case 按环境变量 require + new，default 分支提供安全回退（向量库未知值回退 LanceDB，嵌入未知值回退 NativeEmbedder）。
- **懒加载 require**：每个 case 分支内 `require("../AiProviders/<x>")`，启动期不加载任何 provider SDK，按运行时选择才加载。
- **duck-typing 接口**：LLM/Embedder 无显式基类（靠 JSDoc `@returns` 与方法约定）；VectorDB 有真正的 abstract 基类 `vectorDbProviders/base.js`，未实现方法默认 throw。
- **注入关系**：`getLLMProvider()` 内部调 `getEmbeddingEngineSelection()` 把 embedder 注入每个 LLM 实例（`helpers/index.js:138`）。
- **本地模型单例复用**：NativeEmbedder/NativeEmbeddingReranker 用静态 Map 复用 ONNX session，避免重复加载与内存泄漏（onnxruntime-node dispose 为空操作）。

## 4. 域级图

本域共 10 张叶子图（5 叶 × 2 张），覆盖 architecture/sequence/dataflow 三种图型。
