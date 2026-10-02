# 文档与向量模型（document-vector-models）

> 本文是 `server-data` 域下的叶子子系统文档。域级总览见 `../server-data.md`。
> 本文展开 Document/DocumentVector/WorkspaceParsedFiles 等模型封装，不展开嵌入与向量库调用（见 ai-integrations）。
>
> 源码基准：anything-llm v1.16.2，commit `128a0157`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 工作区文档查询 | forWorkspace/get/where | `server/models/documents.js:32,49,61` |
| 添加文档到工作区 | addDocuments（触发嵌入，返回 failedToEmbed） | `documents.js:83` |
| 从工作区移除 | removeDocuments（反嵌入） | `documents.js:204` |
| 文档删除/更新 | delete/update/_updateAll | `documents.js:39,254,274` |
| 文档内容读取 | content/contentByDocPath | `documents.js:286,295` |
| 可写字段白名单 | pinned/watched/lastUpdatedAt | `documents.js:10` |
| 向量记录批量插入 | DocumentVectors.bulkInsert（事务） | `server/models/vectors.js:7-31` |
| 向量记录查询/删除 | where/deleteForWorkspace/deleteIds | `vectors.js:33,46,62` |
| 解析文件记录 | workspace_parsed_files 表 | `server/models/workspaceParsedFiles.js` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `Document.addDocuments(workspace, adds, userId)` | `documents.js:83` | 把文件加入工作区并嵌入 |
| `Document.removeDocuments(workspace, removes, userId)` | `documents.js:204` | 移除并反嵌入 |
| `Document.forWorkspace(workspaceId)` | `documents.js:32` | 列工作区全部文档 |
| `DocumentVectors.bulkInsert(records)` | `vectors.js:7` | 事务批量写 docId↔vectorId 映射 |
| `DocumentVectors.deleteForWorkspace(workspaceId)` | `vectors.js:46` | 删工作区全部向量映射 |

## 3. 关键调用链

**链 A：嵌入并写映射（`documents.js:83`、`vectors.js:7`）**
1. `Document.addDocuments` 对每个文件走嵌入流程，拿到 vectorId 列表。
2. `DocumentVectors.bulkInsert` 用 `prisma.$transaction(inserts)` 批量写 document_vectors 行（`vectors.js:18-24`）。
3. 写 workspace_documents 行记录工作区文档归属。

**链 B：反嵌入（`documents.js:204`）**
1. `removeDocuments` 从向量库删除分块向量。
2. `DocumentVectors.deleteForWorkspace`/按 id 删 document_vectors 映射。
3. 删 workspace_documents 行。

![文档与向量模型架构图](document-vector-models-architecture.html)
![文档嵌入记录数据流](document-vector-models-dataflow.html)

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| workspace_documents.pinned/watched | 默认 false | `schema.prisma:36-37` |
| document_vectors | docId↔vectorId 映射，无外键 | `schema.prisma:113-119` |
| workspace_parsed_files.tokenCountEstimate | 默认 0 | `schema.prisma:386` |

## 5. 错误与重试语义

- `bulkInsert` 事务失败 catch 后返回 `{documentsInserted:0}`，不抛（`vectors.js:25-29`）。
- `where` 查询失败返回空数组（`vectors.js:42-44`）。
- addDocuments 返回 `failedToEmbed[]` 与 errors，不中断整体。
- 无重试；向量库删除失败上抛端点。

## 6. 并发细节

- bulkInsert 用 `prisma.$transaction` 保证原子。
- 模型方法 async，单请求独立。
- document_vectors 无外键约束（按 docId 字符串关联），跨表一致性靠应用层。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- documents/vectors/workspaceParsedFiles 模型封装。

**Out-of-Scope（不在本仓库源码内）**
- 向量库中实际向量的存储与检索（vectorDbProviders，ai-integrations）。
- 文档解析/切分（collector，document-pipeline 域）。

## 8. 与相邻子系统交互

- 本叶子 → prisma-schema：经 PrismaClient。
- 本叶子 → document-embed-api：被上传/嵌入端点调用。
- 本叶子 → user-workspace-models：workspace_documents 关联 workspaces。

## 9. 语言专项适配口径（TS-JS 口径）

- **capability seam 分组**：模型是 Provider，向量库是外部 Provider。
- **图型**：architecture（表关系）+ dataflow（嵌入记录写入管道）。
- **外部边界**：外部向量库标 external；向量本体不在 SQLite。
- **部署维度**：关系表在 SQLite，向量在外部向量库，二者分离。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 文档向量架构图 | `document-vector-models-architecture.html` | architecture | showcase |
| 嵌入数据流 | `document-vector-models-dataflow.html` | dataflow | **standard** |

- 数据流落 standard 原因：showcase 校验 `clean-flow/edge-through-node`（跨节点连线）未过；采取的修复动作：把向量索引节点下移到 row1、改 n4→n5 短连后仍未达 showcase，按兜底降 standard（render 退出码 0）。
- JSON IR 位于 `json/` 目录。
