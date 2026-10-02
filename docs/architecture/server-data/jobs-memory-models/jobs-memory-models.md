# 后台任务与记忆模型（jobs-memory-models）

> 本文是 `server-data` 域下的叶子子系统文档。域级总览见 `../server-data.md`。
> 本文展开定时任务、任务运行记录、记忆、事件日志等模型封装，不展开 Worker 调度循环本身（属后台服务）。
>
> 源码基准：anything-llm v1.16.2，commit `128a0157`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 定时任务定义 | create/update/delete/where/allEnabled/canActivate | `server/models/scheduledJob.js:58,77,140,118,150,188` |
| cron 计算 | computeNextRunAt/isValidCron | `scheduledJob.js:30,50` |
| 运行时间戳推进 | updateRunTimestamps/recomputeNextRunAt | `scheduledJob.js:223,202` |
| 任务运行记录状态机 | start/markRunning/complete/fail/timeout/kill | `server/models/scheduledJobRun.js:32,67,118,135,161,195` |
| 孤儿运行回收 | failOrphanedRuns | `scheduledJobRun.js:281` |
| 记忆作用域 | forUserWorkspace/globalForUser/create/update/delete | `server/models/memory.js:51,73,98,137,158` |
| 记忆升降级 | promoteToGlobal/demoteToWorkspace | `memory.js:176,214` |
| 记忆批量替换/抽取 | replaceWorkspaceMemories/applyExtractedMemories（事务） | `memory.js:300,345` |
| 事件审计 | EventLogs.logEvent/getByEvent/whereWithData | `server/models/eventLogs.js:4,25,79` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `ScheduledJob.allEnabled()` | `scheduledJob.js:150` | 列启用任务供 Worker 拾取 |
| `ScheduledJobRun.start(jobId)` | `scheduledJobRun.js:32` | 事务创建 run 并标记 running |
| `ScheduledJobRun.statuses` | `scheduledJobRun.js:4` | queued/running/completed/failed/timed_out |
| `ScheduledJobRun.failOrphanedRuns()` | `scheduledJobRun.js:281` | 启动时把残留 running 标记失败 |
| `Memory.replaceWorkspaceMemories(...)` | `memory.js:300` | 事务替换工作区记忆 |
| `EventLogs.logEvent(event, metadata, userId)` | `eventLogs.js:4` | 写审计事件 |

## 3. 关键调用链

**链 A：Worker 拾取并执行任务（`scheduledJob.js:150`、`scheduledJobRun.js:32`）**
1. Worker 调 `ScheduledJob.allEnabled()` 取到期任务。
2. `ScheduledJobRun.start(jobId)` 在 `prisma.$transaction` 内建 run 行、置 running（`scheduledJobRun.js:34`）。
3. 执行完调 `complete(id,{result})` 或 `fail(id,{error})`，写 completedAt。
4. `updateRunTimestamps` 推进 job 的 lastRunAt/nextRunAt。

**链 B：记忆抽取落库（`memory.js:345`）**
1. Agent 对话后产出抽取记忆。
2. `applyExtractedMemories` 用事务批量 upsert。

![任务与记忆模型架构图](jobs-memory-models-architecture.html)
![定时任务执行时序](jobs-memory-models-sequence.html)

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| scheduled_jobs.schedule | cron 表达式 | `schema.prisma:409` |
| scheduled_jobs.enabled | 默认 true | `schema.prisma:410` |
| run.status | queued/running/completed/failed/timed_out | `schema.prisma:425` |
| memories.scope | 默认 workspace | `schema.prisma:443` |
| 任务并发上限 | canActivate 限制 active 数量 | `scheduledJob.js:188` |

## 5. 错误与重试语义

- run 状态机：running 中异常走 `fail`，超时走 `timeout`，均落终态。
- `failOrphanedRuns` 在启动时把上次崩溃残留的 running 回收为 failed。
- 记忆写失败事务回滚，不部分写入。
- 无自动重试；失败记录留存供查看。

## 6. 并发细节

- `ScheduledJobRun.start` 用 `prisma.$transaction` 防并发重复拾取。
- `replaceWorkspaceMemories`/`applyExtractedMemories` 用事务保证原子。
- 无应用锁；SQLite 串行。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- scheduledJob/scheduledJobRun/memory/eventLogs 模型封装。

**Out-of-Scope（不在本仓库源码内）**
- Worker 调度循环与 cron 触发（后台服务进程）。
- Agent 实际执行（ai-agents 域）。

## 8. 与相邻子系统交互

- 本叶子 → prisma-schema：经 Prisma。
- 本叶子 → 后台 Worker：被拾取并驱动状态机。
- 本叶子 → user-workspace-models：memories 关联 user/workspace。

## 9. 语言专项适配口径（TS-JS 口径）

- **capability seam 分组**：模型是 Provider，Worker 是 Consumer。
- **图型**：architecture（表拓扑）+ sequence（任务执行链）。
- **状态机**：run 有明确状态机，但 lifecycle 图布局校验反复不过，已在 MD 文字描述状态流转（queued→running→completed/failed/timed_out），改用 sequence 表达执行顺序。
- **外部边界**：SQLite 经 Prisma；cron 解析为 Node 库。
- **部署维度**：表在 SQLite，Worker 随 server 进程。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 任务记忆架构图 | `jobs-memory-models-architecture.html` | architecture | showcase |
| 任务执行时序图 | `jobs-memory-models-sequence.html` | sequence | showcase |

- 未生成 lifecycle：run 状态机本适合 lifecycle 图，但 `layout/constraint`（transition label 与 state 重叠）经多轮 labelDy/viewBox 调整仍未通过 showcase 且 standard 渲染同样报错，按兜底改用 sequence 表达执行顺序，状态流转文字见第 3、5 节。
- JSON IR 位于 `json/` 目录。
