# 认证与授权 API（auth-authz-api）

> 本文是 `server-api` 域下的叶子子系统文档。域级总览见 `../server-api.md`。
> 本文展开登录/注册/JWT/会话/多用户权限与各鉴权中间件，不展开用户 CRUD 管理（见 admin-system-api）。
>
> 源码基准：anything-llm v1.16.2，commit `128a0157`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 统一会话鉴权中间件 | `validatedRequest`：按单/多用户模式分流校验 Bearer JWT | `server/utils/middleware/validatedRequest.js:8` |
| 单用户密码模式校验 | JWT 内嵌加密后的密码串，正则匹配 + bcrypt 比对环境变量 AUTH_TOKEN | `validatedRequest.js:33-66` |
| 多用户会话校验 | decodeJWT 取 `id` → `User.get` → 检查 suspended，注入 `response.locals.user` | `validatedRequest.js:70-104` |
| 开发者 API Key 校验 | `validApiKey`：Bearer 密钥查 ApiKey 表，注入 multiUserMode | `server/utils/middleware/validApiKey.js:4-34` |
| 角色授权中间件 | `flexUserRoleValid`/`strictMultiUserRoleValid`/`isSingleUserMode`/`isMultiUserSetup`，角色 admin/manager/default | `server/utils/middleware/multiUserProtected.js` |
| JWT 签发与验签 | `makeJWT`（默认 30d）、`decodeJWT`（失败返回空对象） | `server/utils/http/index.js:25,60` |
| 登录请求令牌 | `POST /request-token`：单用户/多用户双分支登录 | `server/endpoints/system.js:198-349` |
| Simple SSO 临时令牌 | `GET /request-token/sso/simple`，由 `simpleSSOEnabled` 守卫 | `server/endpoints/system.js:351-358`、`utils/middleware/simpleSSOEnabled.js` |
| 邀请注册 | `/invite/:code` 领取邀请码注册 | `server/endpoints/invite.js:12` |
| 开发者认证探活 | `GET /v1/auth` 带 validApiKey 返回 `{authenticated:true}` | `server/endpoints/api/auth/index.js:8-29` |
| 登录审计 | 失败登录（用户名错/密码错/封禁）写 EventLogs 与遥测 | `system.js:218-284` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `validatedRequest(req,res,next)` | `validatedRequest.js:8` | 会话鉴权主中间件，模式分流 |
| `validateMultiUserRequest` | `validatedRequest.js:70` | 多用户会话校验子函数 |
| `validApiKey` | `validApiKey.js:4` | 开发者 API Key 校验 |
| `flexUserRoleValid(allowedRoles)` / `strictMultiUserRoleValid(allowedRoles)` | `multiUserProtected.js:44,24` | 角色授权工厂，返回中间件 |
| `ROLES = {all,admin,manager,default}` | `multiUserProtected.js:3-8` | 角色枚举 |
| `makeJWT(info, expiry)` / `decodeJWT(token)` | `http/index.js:25,60` | JWT 签发/验签 |
| `userFromSession(req,res)` | `http/index.js:39` | 从请求取当前多用户用户（优先 locals 缓存） |
| `simpleSSOEnabled` / `simpleSSOLoginDisabled` | `simpleSSOEnabled.js:5,46` | SSO 开关中间件与判断函数 |

## 3. 关键调用链

**链 A：多用户登录签发令牌（`system.js:198-310`）**
1. `POST /request-token` 取 `{username,password}`；若多用户模式且 SSO 禁登录则 403（`system.js:202-212`）。
2. `User._get({username})` 查用户；不存在则记 `failed_login_invalid_username` 事件返回 `[001]`（`system.js:215-233`）。
3. `bcrypt.compareSync(password, existingUser.password)` 比对失败记 `failed_login_invalid_password` 返回 `[002]`（`system.js:235-251`）。
4. `existingUser.suspended` 为真记 `failed_login_account_suspended` 返回 `[004]`（`system.js:253-269`）。
5. 通过后 `makeJWT({id,username}, JWT_EXPIRY)` 签发会话令牌；首登未见过恢复码则生成恢复码一并返回（`system.js:288-309`）。

**链 B：受保护请求穿过鉴权（`validatedRequest.js:8-104`）**
1. 先 `SystemSettings.isMultiUserMode()` 并写入 `response.locals.multiUserMode`（`validatedRequest.js:9-11`）。
2. 多用户走 `validateMultiUserRequest`：取 Bearer → decodeJWT → `User.get({id})` → suspended 检查 → 注入 `locals.user`（`validatedRequest.js:70-104`）。
3. 单用户：开发态或未配 AUTH_TOKEN/JWT_SECRET 时直接放行；否则校验 JWT 中加密的 `p` 字段与 AUTH_TOKEN 的 bcrypt 哈希（`validatedRequest.js:18-66`）。
4. 通过后下游 `flexUserRoleValid` 按角色放行或返回 401。

![认证鉴权体系架构图](auth-authz-api-architecture.html)
![多用户登录请求令牌时序](auth-authz-api-sequence.html)

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `JWT_SECRET` | 未设置时 makeJWT 抛错；单用户模式未设置则跳过鉴权放行 | `http/index.js:26`、`validatedRequest.js:20` |
| `AUTH_TOKEN` | 单用户模式的登录密码（bcrypt 哈希比对） | `validatedRequest.js:48,60`、`system.js:316` |
| `JWT_EXPIRY` | 登录令牌有效期，未传则 makeJWT 默认 `30d` | `system.js:290,340`、`http/index.js:25` |
| `SIMPLE_SSO_ENABLED` | 存在该环境变量才允许 SSO 临时令牌 | `simpleSSOEnabled.js:6` |
| `SIMPLE_SSO_NO_LOGIN` | 与 SSO 同存时禁用账密登录 | `simpleSSOEnabled.js:46` |
| 多用户模式开关 | 由 SystemSettings 表（`isMultiUserMode()`）决定，非 env | `validatedRequest.js:9` |

## 5. 错误与重试语义

- 登录失败分码返回：`[001]` 用户名不存在、`[002]` 密码错、`[003]` 单用户密码错、`[004]` 账户封禁、`[005]` 管理员禁用账密登录；均写 EventLogs 审计（`system.js:226-328`）。
- JWT 验签失败不抛错，`decodeJWT` catch 后返回空对象 `{p:null,id:null,username:null}`，由中间件判空返回 401（`http/index.js:60-65`）。
- 鉴权失败一律 401/403 直接响应，不进入业务；本叶子无重试（认证失败重试由前端/调用方负责）。
- suspended 用户在会话校验与登录两处都会被拒（`validatedRequest.js:93-99`、`system.js:253-269`）。

## 6. 并发细节

- 中间件为 `async` 函数，每次请求独立 `await` DB 查询（User.get、SystemSettings.isMultiUserMode），请求间无共享可变状态。
- `UserMetaCache.setFromRequest(request)`（`validatedRequest.js:67,104`）按请求缓存用户 locale，避免重复查询。
- bcrypt 比对为同步 CPU 密集调用（`bcryptjs.compareSync`），在事件循环上阻塞，密码哈希 cost 因子 10（`system.js:316`）。
- 无后台 goroutine/队列；鉴权状态通过 `response.locals` 在同一请求中间件链内传递。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- 鉴权/授权中间件、JWT 工具、登录/SSO/邀请端点、登录审计。

**Out-of-Scope（不在本仓库源码内）**
- 用户 CRUD、角色分配、用户管理端点（见 admin-system-api）。
- 密码恢复流程的完整 UI 与邮件发送（`utils/PasswordRecovery`，见 jobs-memory-models 相邻域）。
- bcryptjs、jsonwebtoken 等 npm 依赖（不在本仓库源码内）。
- OAuth/OIDC 外部 IdP 对接（本项目仅内置 Simple SSO 临时令牌，完整企业 IdP 不在本仓库源码内）。

## 8. 与相邻子系统交互

- 本叶子 → server-data：读写 `User`、`SystemSettings`、`ApiKey`、`EventLogs`、`TemporaryAuthToken` 模型（见 user-workspace-models、auth-settings-models）。
- 本叶子 → 各 API 叶子：作为路由守卫被 `validatedRequest`/`flexUserRoleValid` 等挂在受保护路由上。
- 上游：Web 客户端、开发者 API 客户端经 Authorization 头传 Bearer JWT/Key。
- 下游：调用 EncryptionManager 解密单用户模式 JWT 中的 `p` 字段。

## 9. 语言专项适配口径（TS-JS 口径）

- **capability seam 分组**：中间件是"横切关注点"（Service Definition），User/ApiKey 模型是 Provider，前端与 API 客户端是 Consumer；按导出的中间件工厂函数归组。
- **图型**：architecture（双轨鉴权组件拓扑）+ sequence（登录请求-响应链）；本叶子无数据管道、无单实体状态机（登录失败重试由调用方负责，不在服务端建模），故不补 dataflow/lifecycle。
- **外部边界**：图中标 `external` 的 API 客户端、npm 依赖 bcryptjs/jsonwebtoken、浏览器 Authorization 头；SQLite 经 Prisma 访问。
- **事件驱动**：登录审计经 EventLogs 异步写库，登录失败事件作为次要虚线消息进 sequence。
- **部署维度**：鉴权逻辑随 server 单进程部署；多用户模式与否由 DB 配置（SystemSettings）在运行期切换，非构建期区分。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 认证鉴权架构图 | `auth-authz-api-architecture.html` | architecture | **standard** |
| 登录时序图 | `auth-authz-api-sequence.html` | sequence | showcase |

- 架构图落 standard 原因：showcase 校验 `clean-flow/endpoint-side-direction`（跨层连线端点方向）未过；采取的修复动作：按 M2 删去低价值边、重排组件为两列布局后仍未达 showcase，按兜底策略降 standard 渲染（render 退出码 0，HTML 约 806KB）。
- 未生成 dataflow/lifecycle：无数据管道与单实体状态机语义。
- JSON IR 位于 `json/` 目录。
