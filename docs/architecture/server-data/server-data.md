# server-data 域总览

> 本文是 `server-data` 域的总览文档。本域覆盖 anything-llm 服务端数据访问层：Prisma schema、用户/工作区/文档/向量/配置/任务等模型封装。
>
> 源码基准：anything-llm v1.16.2，commit `128a0157`。主语言 JavaScript（并入 TS-JS 口径）。

## 域职责

`server-data` 域是服务端的数据访问层，负责：

- Prisma schema 定义、迁移与 SQLite 连接单例。
- User/Workspace/WorkspaceUser 模型与多对多关系。
- Document/DocumentVector/WorkspaceParsedFiles 等文档与向量映射模型。
- 系统设置、API 密钥、邀请、临时认证令牌等配置存储模型。
- 定时任务、任务运行记录、记忆、事件日志等后台模型。

本域只封装 CRUD 与字段校验，不做 HTTP 编排（属 server-api），也不做向量库/LLM 的实际连接（属 ai-integrations）。

## 叶子索引

| 叶子 | 职责 | 文档 |
|---|---|---|
| prisma-schema | Prisma schema、迁移、SQLite 连接 | [prisma-schema.md](./prisma-schema/prisma-schema.md) |
| user-workspace-models | User/Workspace/WorkspaceUser 模型与关系 | [user-workspace-models.md](./user-workspace-models/user-workspace-models.md) |
| document-vector-models | Document/DocumentVector/解析文件模型 | [document-vector-models.md](./document-vector-models/document-vector-models.md) |
| auth-settings-models | 系统设置/API 密钥/邀请/令牌模型 | [auth-settings-models.md](./auth-settings-models/auth-settings-models.md) |
| jobs-memory-models | 定时任务/运行记录/记忆/事件日志模型 | [jobs-memory-models.md](./jobs-memory-models/jobs-memory-models.md) |

## 域级机制细节

- **单例连接**：`server/utils/prisma/index.js` 导出共享 PrismaClient，全部模型复用。
- **键值配置**：`system_settings` 用 label/value 键值表存全部系统配置，`protectedFields` 限制敏感键不可经 API 改。
- **关系与外键**：用户/工作区/文档/记忆之间用 Prisma 外键关联（多对多用 workspace_users 中间表）；document_vectors 无外键，按 docId 字符串关联外部向量库。
- **事务**：批量写（bulkInsert、记忆替换、任务 start）用 `prisma.$transaction` 保证原子。
- **状态机**：scheduled_job_runs 有明确状态（queued/running/completed/failed/timed_out），由模型方法显式推进。
