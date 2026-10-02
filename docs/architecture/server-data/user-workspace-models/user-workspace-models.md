# 用户与工作区模型（user-workspace-models）

> 本文是 `server-data` 域下的叶子子系统文档。域级总览见 `../server-data.md`。
> 本文展开 User/Workspace/WorkspaceUser 等模型封装与关系，不展开其他业务模型。
>
> 源码基准：anything-llm v1.16.2，commit `128a0157`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 用户 CRUD | create/update/delete/get/where/count | `server/models/user.js:108,157,302,272,312,292` |
| 用户字段校验 | 用户名/角色/密码复杂度/配额校验后落库 | `user.js:115-135` |
| 敏感字段过滤 | `filterFields` 去掉密码等返回前端 | `user.js:136` |
| 聊天配额 | `canSendChat(user)` 24h 消息上限 | `user.js:364` |
| 工作区创建 | `new(name, creatorId, additionalFields)` | `server/models/workspace.js:199` |
| 工作区查询 | get/getWithUser/whereWithUser(s)（多用户带权限过滤） | `workspace.js:369,297,418,447` |
| 工作区更新/删除 | update/delete/trackChange | `workspace.js:247,392,512` |
| 工作区成员管理 | workspaceUsers/updateUsers（绑/解成员） | `workspace.js:468,501` |
| 成员关系表 | WorkspaceUser createMany/create/get/count/delete | `server/models/workspaceUsers.js:4,26,45,60,83,93` |
| 提示词历史 | workspace.promptHistory/deletePromptHistory | `workspace.js:623-655` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `User.create({username,password,role,...})` | `user.js:108` | 校验+bcrypt 哈希+创建用户 |
| `User.get/clause` / `_get` | `user.js:272,282` | 按条件查用户 |
| `User.canSendChat(user)` | `user.js:364` | 24h 配额判定 |
| `Workspace.new(name, creatorId, fields)` | `workspace.js:199` | 创建工作区 |
| `Workspace.getWithUser(user, clause)` | `workspace.js:297` | 多用户模式下按权限查工作区 |
| `Workspace.updateUsers(workspaceId, userIds)` | `workspace.js:501` | 重设工作区成员 |
| `WorkspaceUser.createMany(userId, workspaceIds)` | `workspaceUsers.js:4` | 批量绑成员 |

## 3. 关键调用链

**链 A：创建用户（`user.js:108-141`）**
1. `checkPasswordComplexity` 不通过直接返回 `{user:null,error}`（`user.js:115-118`）。
2. `validations.username/role/bio/dailyMessageLimit` 规范化字段（`user.js:122-133`）。
3. `bcrypt.hashSync(password, 10)` 哈希后 `prisma.users.create`（`user.js:124-135`）。
4. 返回 `filterFields(user)` 去掉敏感字段；异常 catch 后格式化错误信息（`user.js:136-140`）。

**链 B：创建工作区并绑成员（`workspace.js:199`）**
1. `Workspace.new` 写 workspaces 行。
2. 创建后 `WorkspaceUser.createMany` 把创建者/指定用户绑入 workspace_users。

**链 C：多用户按权限查工作区（`workspace.js:297`）**
1. `getWithUser(user, clause)` 联表 workspace_users 过滤出该用户可见的工作区。

![用户与工作区模型架构图](user-workspace-models-architecture.html)
![创建工作区时序](user-workspace-models-sequence.html)

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| 用户默认角色 | `default` | `user.js:111` |
| bcrypt cost | 10 | `user.js:125` |
| suspended 字段 | 0/1 标记封禁 | `schema.prisma:67` |
| Workspace.chatMode | 默认 `chat` | `schema.prisma:136` |
| topN/similarityThreshold | 默认 4 / 0.25 | `schema.prisma:132,135` |

## 5. 错误与重试语义

- 密码复杂度/用户名校验失败：返回错误对象，不抛异常，由端点转 400。
- `User.create` catch 后 `_identifyErrorAndFormatMessage` 把唯一约束等转可读信息（`user.js:137-139`）。
- 查询不到返回 null，由端点决定 404。
- 无重试；DB 错误直接上抛端点。

## 6. 并发细节

- 模型方法为 async，单请求独立 await Prisma。
- `filterFields` 为同步纯函数。
- 多对多成员关系由 workspace_users 表表达，`updateUsers` 全量重绑。
- 无应用锁；SQLite 串行写。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- User/Workspace/WorkspaceUser 模型封装、字段校验、配额、成员关系。

**Out-of-Scope（不在本仓库源码内）**
- 聊天记录/线程/嵌入配置等模型（见其余 server-data 叶子）。
- 向量库中工作区向量的删除/重建（ai-integrations）。

## 8. 与相邻子系统交互

- 本叶子 → prisma-schema：经 PrismaClient 访问 users/workspaces/workspace_users 表。
- 本叶子 → server-api：被 auth/admin/chat/workspace 端点调用。
- 本叶子 → document-vector-models：workspace 一对多 workspace_documents。

## 9. 语言专项适配口径（TS-JS 口径）

- **capability seam 分组**：模型文件是 Provider（数据访问），端点是 Consumer；按表归属分组。
- **图型**：architecture（表关系拓扑）+ sequence（创建工作区的表写入顺序）。
- **外部边界**：SQLite 经 Prisma 访问；bcrypt 为 Node 原生扩展。
- **部署维度**：模型随 server 包；数据存于 storage/anythingllm.db。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 用户工作区架构图 | `user-workspace-models-architecture.html` | architecture | showcase |
| 创建工作区时序图 | `user-workspace-models-sequence.html` | sequence | showcase |

- 未生成 dataflow/lifecycle：无数据管道；用户封禁态为简单布尔字段，不单列状态机。
- JSON IR 位于 `json/` 目录。
