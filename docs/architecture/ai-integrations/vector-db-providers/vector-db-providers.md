# 向量数据库提供商适配（vector-db-providers）

> 本文是 `ai-integrations` 域下的叶子子系统文档。域级总览见 `../ai-integrations.md`。
> 本文只展开"向量数据库提供商的统一接口、集合（namespace）管理与相似度检索"，不展开嵌入引擎（见 embedding-engines-rerankers）。
>
> 源码基准：anything-llm v1.16.2，commit `128a0157`，纯 JavaScript ESM。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 向量库工厂 | 按 `VECTOR_DB` 环境变量 switch-case 实例化对应向量库类 | `server/utils/helpers/index.js:87` `getVectorDbClass(getExactly)` |
| 10 个向量库适配 | pinecone / chroma / chromacloud / lancedb(默认) / weaviate / qdrant / milvus / zilliz / astra / pgvector | `server/utils/vectorDbProviders/<name>/index.js` |
| 抽象基类 | `VectorDatabase` 定义全部必须实现的方法契约，直接实例化抛错 | `server/utils/vectorDbProviders/base.js:6` |
| 集合/命名空间管理 | connect / heartbeat / totalVectors / namespaceCount / namespace / hasNamespace / namespaceExists / deleteVectorsInNamespace / updateOrCreateCollection | `base.js:21-`；lance 实现 `lance/index.js:37,62,249` |
| 文档写入 | addDocumentToNamespace（按 docId 写入向量+元数据） | `lance/index.js:312` |
| 相似度检索 | performSimilaritySearch / similarityResponse / rerankedSimilarityResponse | `lance/index.js:183,417,99` |
| 默认本地库 | LanceDB 默认（`lancedb`），文件型本地向量库 | `helpers/index.js:99,122-125` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `getVectorDbClass(getExactly)` | `helpers/index.js:87` | 工厂：按 `VECTOR_DB` 或入参实例化；未知值回退 LanceDB |
| `VectorDatabase`（抽象基类） | `vectorDbProviders/base.js:6` | 纯 virtual 基类；构造期阻止直接实例化；定义 connect/heartbeat/totalVectors/namespaceCount/namespace/hasNamespace/namespaceExists/addDocumentToNamespace/performSimilaritySearch 等接口 |
| `LanceDb extends VectorDatabase` | `vectorDbProviders/lance/index.js:17` | 默认实现；静态 `#connection` 单例连接；`updateOrCreateCollection` 建表 |
| `Pinecone`/`Chroma`/`Qdrant`/`Milvus`/`PGVector`/... | 各 `vectorDbProviders/<name>/index.js` | 各自 extends VectorDatabase，适配云服务 API |

## 3. 关键调用链

1. **文档入库**：DocumentManager 调 `vdb = getVectorDbClass()`（`helpers/index.js:87`）→ `vdb.connect()`（`lance/index.js:37`，静态单例复用）→ 对每条切分文本 `vdb.addDocumentToNamespace(namespace, data)`（`lance/index.js:312`）→ 内部 `updateOrCreateCollection`（`:249`）建/写表。
2. **相似度检索主路径**：聊天检索时 `vdb.performSimilaritySearch({queryEmbedding, namespace, topN})`（`lance/index.js:417`）→ `similarityResponse()`（`:183`）返回 topK 结果。
3. **重排检索**：`vdb.rerankedSimilarityResponse({...})`（`lance/index.js:99`）先 similaritySearch 取放大倍数候选 → `new NativeEmbeddingReranker()`（`:108`）打分重排 → 返回 topN。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `VECTOR_DB` | 默认 `lancedb`；未知值回退 LanceDB 并打 ENV ERROR | `helpers/index.js:88,122` |
| LanceDB uri | 本地文件路径（`storage/lancedb`） | `lance/index.js:21` 构造函数 |
| 各云库连接配置 | `PINECONE_ENVIRONMENT`/`PINECONE_API_KEY`、`CHROMA_ENDPOINT`、`QDRANT_HOST` 等 | 各 provider 构造函数 |
| `STORAGE_DIR` | 本地 LanceDB 与 pgvector 等本地存储根 | 构造函数读取 |

## 5. 错误与重试语义

- **未知 VECTOR_DB**：不抛错，打 `[ENV ERROR]` 后回退 LanceDB（`helpers/index.js:121-125`）。
- **基类直接实例化**：`if (this.constructor === VectorDatabase) throw`（`base.js:12`）。
- **未实现方法**：基类所有方法默认 `throw new Error("Must be implemented by provider")`（`base.js:21-23`），子类必须覆盖。
- 连接错误（如云向量库不可达）向上抛出由调用方（DocumentManager/chat）处理；本层不做重试。

## 6. 并发细节

- LanceDB 用静态 `#connection` 单例（`lance/index.js:19,38-39`），跨请求共享同一个 lancedb 连接，避免重复连接。
- 每次方法调用 `await this.connect()` 幂等（已连接直接返回）。
- 无显式锁；写入按 docId 追加/更新。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `helpers/index.js` 的 `getVectorDbClass()` 工厂
- `vectorDbProviders/base.js` 抽象基类
- `vectorDbProviders/*` 全部 10 个 provider 实现

**Out-of-Scope（不在本仓库源码内）**
- Pinecone / Chroma / Weaviate / Qdrant / Milvus / Zilliz / Astra 等云向量库服务端
- `@lancedb/lancedb` / `@pinecone-database/pinecone` 等 npm SDK
- pgvector 依赖的 PostgreSQL 数据库进程

## 8. 与相邻子系统交互

- 上游 → 本叶子：DocumentManager（入库写入）、聊天检索链路（similaritySearch）、endpoints（集合管理 API）调 `getVectorDbClass()`。
- 上游（嵌入叶子）：embedding-engines 产出的向量交给本叶子 `addDocumentToNamespace` 写入。
- 本叶子 → 下游：各 provider 调对应云 SDK → 外部向量库 API；LanceDB 本地文件读写。

## 9. 语言专项适配口径（TS/JS 口径）

- **capability seam 分组**：Service Definition = `VectorDatabase` 抽象基类（`base.js`，真正的 virtual 基类，区别于 LLM/Embedder 的 duck-typing）+ `getVectorDbClass()` 工厂；Provider = 10 个 extends VectorDatabase 实现；Consumer = DocumentManager、检索链路。
- **图类型侧重**：architecture（工厂+10 provider 静态拓扑）+ sequence（文档写入/检索调用链）。
- **外部边界**：外部云向量库 API、npm SDK、PostgreSQL。
- **部署维度**：默认 LanceDB 为本地文件型，零外部依赖；生产可切云向量库。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 向量库工厂与 provider 架构 | `vector-db-providers-architecture.html` | architecture | standard（showcase 因跨层连线较多回退 standard；render 退出码 0） |
| 文档写入与检索时序 | `vector-db-providers-sequence.html` | sequence | showcase（一次通过） |

JSON IR 源文件位于 `json/` 目录。
