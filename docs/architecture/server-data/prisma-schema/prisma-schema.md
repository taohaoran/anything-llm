# Prisma Schema 与数据库连接（prisma-schema）

> 本文是 `server-data` 域下的叶子子系统文档。域级总览见 `../server-data.md`。
> 本文展开 Prisma schema 定义、迁移与数据库连接，不展开具体业务模型字段（见其余 server-data 叶子）。
>
> 源码基准：anything-llm v1.16.2，commit `128a0157`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 数据源声明 | 默认 SQLite，文件 `../storage/anythingllm.db` | `server/prisma/schema.prisma:14-17` |
| 客户端生成器 | `prisma-client-js` | `schema.prisma:1-3` |
| 可切 PostgreSQL | 注释模板，改 DATABASE_URL 后跑迁移 | `schema.prisma:5-12` |
| 32 个模型定义 | users/workspaces/workspace_chats 等业务表 | `schema.prisma:18-487` |
| 迁移历史 | 41 个迁移目录（2023-09 起） | `server/prisma/migrations/` |
| PrismaClient 单例 | 全局唯一实例，日志 error/info/warn | `server/utils/prisma/index.js:1-12` |
| 种子脚本 | `prisma/seed.js` | `server/prisma/seed.js` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `datasource db` | `schema.prisma:14` | 声明 provider 与连接串 |
| `PrismaClient`（单例） | `utils/prisma/index.js:8` | 类型化数据库访问入口 |
| 各 `model <table>` | `schema.prisma` | 表结构与关系定义 |
| 关系 `@relation(... onDelete: Cascade)` | 如 `recovery_codes:97` | 级联删除规则 |

## 3. 关键调用链

**链 A：业务请求到 SQL（运行时）**
1. 端点调用 `models/xxx.js` 封装方法（如 `User.get({id})`）。
2. 模型方法 `require("../utils/prisma")` 拿到单例 `prisma`，调用 `prisma.<table>.findUnique/...`。
3. Prisma Client 生成 SQL 发往 SQLite 文件（`storage/anythingllm.db`）。

**链 B：Schema 变更到建表（开发/部署）**
1. 修改 `schema.prisma` 后 `prisma migrate dev` 生成新迁移目录。
2. 部署时 `prisma migrate deploy` 按序执行 41 个迁移。
3. `prisma generate` 重新生成类型化 Client。

![Prisma 数据层架构图](prisma-schema-architecture.html)
![数据访问路径数据流](prisma-schema-dataflow.html)

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `datasource.provider` | `sqlite` | `schema.prisma:15` |
| `datasource.url` | `file:../storage/anythingllm.db` | `schema.prisma:16` |
| `DATABASE_URL` | 切 PostgreSQL 时使用 | `schema.prisma:11` |
| Prisma 日志 | `["error","info","warn"]`，可加 `query` 调试 | `utils/prisma/index.js:7` |

## 5. 错误与重试语义

- Prisma 查询错误直接抛给上层模型方法，由端点 catch 返回 500。
- SQLite 写锁/并发写冲突：Prisma 不自动重试；本项目单进程低频写，未实现重试。
- 迁移失败：部署期报错中止，不进入运行时。

## 6. 并发细节

- `PrismaClient` 为进程内单例，内部维护连接池；SQLite 为文件级锁，写串行化。
- 单 Node 进程所有请求共享同一 client 实例。
- 无应用层锁；级联删除由 `onDelete: Cascade` 在 DB 层保证。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- schema.prisma、migrations、PrismaClient 单例、seed 脚本。

**Out-of-Scope（不在本仓库源码内）**
- `@prisma/client` 生成的运行时与查询引擎（npm 依赖）。
- 外部 PostgreSQL 服务（切换部署时）。
- 向量数据库（独立于 Prisma，见 ai-integrations）。

## 8. 与相邻子系统交互

- server-data 域所有叶子（模型封装）都依赖本叶子的 PrismaClient 单例。
- server-api 端点经模型封装间接访问本层。

## 9. 语言专项适配口径（TS-JS 口径）

- **capability seam 分组**：本叶子是数据访问基础设施工厂（Service Definition），models/ 各文件是 Provider。
- **图型**：architecture（数据层拓扑）+ dataflow（请求→模型→client→SQLite 的数据访问管道）。
- **外部边界**：SQLite 文件为 Node 存储边界；PostgreSQL 为外部服务。
- **部署维度**：Prisma Client 随 server 包打包；SQLite 为单文件零依赖默认部署，PostgreSQL 需外部实例。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 数据层架构图 | `prisma-schema-architecture.html` | architecture | **standard** |
| 数据访问数据流 | `prisma-schema-dataflow.html` | dataflow | showcase |

- 架构图落 standard 原因：showcase 校验 `clean-flow/endpoint-side-direction`（竖向连线端点方向）未过；采取的修复动作：加 fromSide/toSide 后仍未达 showcase，按兜底降 standard（render 退出码 0）。
- 未生成 sequence/lifecycle：无单次请求多参与方消息交互、无单实体状态机。
- JSON IR 位于 `json/` 目录。
