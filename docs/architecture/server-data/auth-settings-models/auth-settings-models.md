# 系统设置与认证配置模型（auth-settings-models）

> 本文是 `server-data` 域下的叶子子系统文档。域级总览见 `../server-data.md`。
> 本文展开系统设置、API 密钥、邀请、临时认证令牌等配置存储模型，不展开业务用户模型。
>
> 源码基准：anything-llm v1.16.2，commit `128a0157`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 系统设置读写 | currentSettings/get/where/updateSettings | `server/models/systemSettings.js:455,654,673,689` |
| 设置校验钩子 | 各 preference 的 validations（memory/gmail/calendar/outlook 等） | `systemSettings.js:108-453` |
| 模式判定 | isMultiUserMode/memoriesEnabled/isOnboardingComplete | `systemSettings.js:754,764,795` |
| 引导完成 | markOnboardingComplete | `systemSettings.js:805` |
| 偏好键分组 | vectorDBPreferenceKeys/llmPreferenceKeys | `systemSettings.js:838,881` |
| 受保护字段 | protectedFields（multi_user_mode 等不可经 API 改） | `systemSettings.js:40` |
| API 密钥 | create（随机 secret）/get/delete/where | `server/models/apiKeys.js:12,31,51,61` |
| 邀请码 | create（随机 code）/deactivate/markClaimed | `server/models/invite.js:10,26,44` |
| 临时 SSO 令牌 | temporary_auth_tokens 模型 | `server/models/temporaryAuthToken.js` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `SystemSettings.currentSettings()` | `systemSettings.js:455` | 聚合全部配置 |
| `SystemSettings.getValueOrFallback(clause, fallback)` | `systemSettings.js:664` | 读单个键带默认值 |
| `SystemSettings.updateSettings(updates)` | `systemSettings.js:689` | 批量更新设置 |
| `SystemSettings.isMultiUserMode()` | `systemSettings.js:754` | 是否多用户模式 |
| `ApiKey.create(createdBy, name)` | `apiKeys.js:12` | 生成随机 secret 写密钥 |
| `Invite.create({createdByUserId, workspaceIds})` | `invite.js:10` | 生成邀请码 |
| `protectedFields` | `systemSettings.js:40` | 不可经 API 修改的键 |

## 3. 关键调用链

**链 A：读取多用户模式（`systemSettings.js:754`）**
1. 中间件 `validatedRequest` 调 `SystemSettings.isMultiUserMode()`。
2. 内部 `getValueOrFallback({label:"multi_user_mode"}, ...)` 查 system_settings 键值表。
3. 解析为布尔后写入 `response.locals.multiUserMode`。

**链 B：更新设置（`systemSettings.js:689`）**
1. 管理端点提交 updates，先过各 preference 的 validation 钩子。
2. 绕过 protectedFields 后 `_updateSettings` 写库。

![认证与配置模型架构图](auth-settings-models-architecture.html)
![读取系统设置时序](auth-settings-models-sequence.html)

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| system_settings 结构 | label 唯一 / value 字符串 | `schema.prisma:53-60` |
| protectedFields | multi_user_mode/hub_api_key/onboarding_complete | `systemSettings.js:40` |
| api_keys.secret | 随机唯一 | `apiKeys.js:7` |
| invites | 随机 code，关联 workspaceIds | `invite.js:5` |

## 5. 错误与重试语义

- 设置校验失败：validation 钩子返回错误，不写库。
- 密钥/邀请码生成失败：catch 返回错误对象。
- 无重试。

## 6. 并发细节

- 键值表读写为单行操作；updateSettings 批量写。
- 无锁；SQLite 串行写。
- `getValueOrFallback` 高频被中间件调用，结果缓存在 response.locals。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- systemSettings/apiKeys/invite/temporaryAuthToken 模型封装。

**Out-of-Scope（不在本仓库源码内）**
- LLM/向量库配置的运行时加载与连接测试（ai-integrations）。
- 邮件发送邀请（外部服务）。

## 8. 与相邻子系统交互

- 本叶子 → prisma-schema：经 Prisma。
- 本叶子 → auth-authz-api：isMultiUserMode 决定鉴权分支。
- 本叶子 → admin-system-api：管理端点读写这些配置。

## 9. 语言专项适配口径（TS-JS 口径）

- **capability seam 分组**：键值配置模型是 Provider；按"系统设置/密钥/邀请"归组。
- **图型**：architecture（表拓扑）+ sequence（读设置的调用链）。
- **外部边界**：SQLite 经 Prisma；随机 secret 用 Node crypto。
- **部署维度**：配置随 DB 持久化，重启保留。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 配置模型架构图 | `auth-settings-models-architecture.html` | architecture | showcase |
| 读设置时序图 | `auth-settings-models-sequence.html` | sequence | showcase |

- 未生成 dataflow/lifecycle：无数据管道；配置变更无单实体状态机。
- JSON IR 位于 `json/` 目录。
