# 管理与系统 API（admin-system-api）

> 本文是 `server-api` 域下的叶子子系统文档。域级总览见 `../server-api.md`。
> 本文展开系统设置、用户管理、工作区管理、API 密钥与开发者管理端点，不展开登录鉴权中间件本身（见 auth-authz-api）。
>
> 源码基准：anything-llm v1.16.2，commit `128a0157`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 用户列表 | `GET /admin/users` | `server/endpoints/admin.js:39` |
| 新建用户 | `POST /admin/users/new`，bcrypt 哈希密码 | `admin.js:53` |
| 更新/删除用户 | `POST /admin/user/:id`、`DELETE /admin/user/:id` | `admin.js:89,130` |
| 邀请管理 | 列表/新建/删除 `/admin/invites`、`/admin/invite/new`、`/admin/invite/:id` | `admin.js:163,177,209` |
| 工作区管理 | 列表/新建/更新成员/删除 `/admin/workspaces*` | `admin.js:229-332` |
| 系统偏好 | `GET /admin/system-preferences-for`、`POST /admin/system-preferences` | `admin.js:332,463` |
| API 密钥管理 | 列表/生成/删除 `/admin/api-keys*` | `admin.js:500,520,544` |
| 开发者管理 API | `/v1/admin/*` 经 validApiKey 镜像管理能力 | `server/endpoints/api/admin/index.js:15-735` |
| 开发者用户管理 | `/v1/users`、`/v1/users/:id/issue-auth-token`（SSO 发令牌） | `server/endpoints/api/userManagement/index.js:12,68` |
| 首次引导 | `GET/POST /onboarding`、`GET /setup-complete` | `server/endpoints/system.js:98,108,118` |
| 系统元信息 | `/env-dump`、`/system/logo`、`/system/footer-data`、`/system/support-email`、`/system/custom-app-name` | `system.js:91,727,761,773,789` |
| 多用户模式查询 | `/system/multi-user-mode` | `system.js:717` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `adminEndpoints(app)` | `admin.js` | 注册 `/admin/*` 管理路由 |
| `apiAdminEndpoints(router)` | `endpoints/api/admin/index.js` | 注册 `/v1/admin/*` 开发者路由（validApiKey） |
| `User` 模型 CRUD | `server/models/user.js` | 用户创建/更新/删除/过滤敏感字段 |
| `Invite` 模型 | `server/models/invite.js` | 邀请码发放与校验 |
| `ApiKey` 模型 | `server/models/apiKeys.js` | 开发者密钥生成与校验 |
| `SystemSettings` | `server/models/systemSettings.js` | 系统偏好/多用户模式读写 |
| `updateENV` / `dumpENV` | `server/utils/helpers/updateENV.js` | 写 .env 配置 |

## 3. 关键调用链

**链 A：管理员创建用户（`admin.js:53`、`api/admin/index.js:85`）**
1. 路由经 `flexUserRoleValid([admin])` 守卫（`admin.js` 顶部守卫）。
2. 取请求体用户名/密码/角色，`bcrypt.hashSync` 哈希后 `User.create`。
3. 写 EventLogs 审计事件，返回更新后的用户列表。

**链 B：系统偏好保存（`admin.js:463`）**
1. admin 提交偏好键值；`SystemSettings.write`/相关配置写入 DB 与 `.env`（经 `updateENV`）。
2. 部分偏好变更触发运行时重载（如 LLM/向量库配置，见 auth-settings-models）。

**链 C：开发者经 API Key 管理（`api/admin/index.js`）**
1. 所有 `/v1/admin/*` 先过 `validApiKey`（`api/admin/index.js:15` 等）。
2. 业务逻辑与 Web 管理面复用同一组模型，仅入口与鉴权不同。

![管理与系统 API 架构图](admin-system-api-architecture.html)
![管理员创建用户时序](admin-system-api-sequence.html)

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| 角色要求 | 用户/工作区/偏好管理需 admin；部分 manager 可上传 | `admin.js` 守卫、`multiUserProtected.js` |
| 多用户模式 | 由 SystemSettings 决定是否开放 `/admin/*` | `system.js:717`、`api/admin/index.js:15` |
| `JWT_EXPIRY` | SSO issue-auth-token 签发临时会话令牌有效期 | `api/userManagement/index.js:68` |
| 引导状态 | `markOnboarded()` 后 `/setup-complete` 为真 | `utils/boot/markOnboarded.js` |

## 5. 错误与重试语义

- 用户已存在/密码不合规：400/500 返回错误信息，不自动重试。
- 角色不足：`flexUserRoleValid`/`strictMultiUserRoleValid` 直接 401。
- API Key 缺失/非法：`validApiKey` 返回 403 "No valid api key found"。
- 写 .env 失败：捕获后 500 返回。
- 本叶子无重试；审计事件失败不阻塞主响应。

## 6. 并发细节

- 每个管理请求独立 await 模型 CRUD，无共享锁。
- `updateENV` 写 .env 文件为同步磁盘 I/O，管理面低频操作可接受。
- 遥测/EventLogs 串行 await 写在响应前。
- 无后台任务；管理操作即时生效。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- admin.js、api/admin、api/userManagement、system.js 引导与元信息端点。

**Out-of-Scope（不在本仓库源码内）**
- LLM/向量库配置的运行时加载与校验（auth-settings-models、ai-integrations）。
- 邮件发送邀请（外部邮件服务，不在本仓库源码内）。
- 用户头像/logo 二进制处理（utils/files，归支撑库）。

## 8. 与相邻子系统交互

- 本叶子 → auth-authz-api：复用角色守卫、validApiKey、makeJWT（SSO 发令牌）。
- 本叶子 → server-data：读写 User/Workspace/Invite/ApiKey/SystemSettings。
- 本叶子 → ext-channels-api：系统偏好影响外部渠道开关。
- 上游：管理后台 SPA、开发者 API 客户端。

## 9. 语言专项适配口径（TS-JS 口径）

- **capability seam 分组**：双入口（Web 管理面 + 开发者 OpenAPI）复用同一组模型 Provider，是典型的"同能力两入口"seam。
- **图型**：architecture（双入口 + 模型拓扑）+ sequence（创建用户请求链）。
- **外部边界**：开发者 API 客户端标 external；.env 文件与 SQLite 为 Node/存储边界。
- **部署维度**：管理端点随 server 单进程；开发者 OpenAPI 经 Swagger 自动生成文档。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 管理系统架构图 | `admin-system-api-architecture.html` | architecture | showcase |
| 创建用户时序图 | `admin-system-api-sequence.html` | sequence | showcase |

- 未生成 dataflow/lifecycle：无数据管道与单实体状态机。
- JSON IR 位于 `json/` 目录。
