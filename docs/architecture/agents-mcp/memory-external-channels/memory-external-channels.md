# 记忆系统与外部渠道（memory-external-channels）

> 本文是 `agents-mcp` 域下的叶子子系统文档。域级总览见 `../agents-mcp.md`。
> 本文把四块相对小但同属“agent 的记忆与外部触达”的子系统合并分析：长期记忆（memories）、
> 后台工作者队列（BackgroundWorkers）、推送通知（PushNotifications）、telegramBot 外部渠道。
>
> 源码基准：anything-llm v1.16.2，commit `128a015`，纯 JavaScript ESM。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 长期记忆存取 | `Memories` 类：把 agent 对话中的事实写入向量库长期记忆 | `server/utils/memories/index.js` |
| 后台任务队列 | `BackgroundWorkers`：基于队列的异步作业（telegram 对话、定时任务、记忆记录等） | `server/utils/BackgroundWorkers/index.js` |
| telegram 渠道 | `telegramBot`：轮询 Telegram，把消息入队、管理会话状态与工具审批 | `server/utils/telegramBot/index.js` |
| 推送通知 | `PushNotifications`：任务完成后经 webhook 推送 | `server/utils/PushNotifications/index.js` |
| 会话状态机 | telegram 维护每个 chat 的对话状态、待审批工具、澄清问卷 | `telegramBot/index.js` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `Memories` | `memories/index.js` | 长期记忆的写入/检索门面 |
| `BackgroundWorkers` | `BackgroundWorkers/index.js` | 队列与 worker 注册、作业调度 |
| `telegramBot` 模块 | `telegramBot/index.js` | Telegram Bot 实例、消息 handler、状态管理 |
| `PushNotifications` | `PushNotifications/index.js` | webhook 推送发送 |

## 3. 关键调用链

**链 1：telegram 入站消息**

1. 用户在 Telegram 发消息 → `telegramBot` 轮询收到 update（`telegramBot/index.js` handler）。
2. bot 把消息放入队列，`BackgroundWorkers` 起一个 `handle-telegram-chat` 作业。
3. worker 加载该 chat 的会话状态，构造 agent 调用 `AIbitat` 跑对话（复用 `agent-runtime`）。
4. agent 回复经 bot 发回 Telegram；期间需要用户确认时，bot 管理待审批状态，等用户回复后 `continue`。

**链 2：长期记忆写入**

- agent 对话中经 `rag-memory` 工具（见 `agent-tools`）或结束后 `Memories` 把值得记的事实写入向量库；下次对话可检索。

**链 3：任务完成推送**

- 后台作业完成 → `PushNotifications` 按配置 webhook 地址 POST 通知用户。

## 4. 配置项

| 配置 | 行为 | 位置 |
|---|---|---|
| Telegram Bot Token | 启用 telegram 渠道所需 | `telegramBot/index.js` |
| webhook URL | PushNotifications 推送目标 | `PushNotifications/index.js` |
| 队列并发/重试 | BackgroundWorkers 作业参数 | `BackgroundWorkers/index.js` |

## 5. 错误与重试语义

- telegram 单条消息处理失败不影响 bot 轮询循环；失败作业入重试或丢弃。
- 推送通知失败记日志，不阻塞主流程。
- 记忆写入失败不影响对话返回。

## 6. 并发细节

- BackgroundWorkers 队列 worker 并发消费作业，作业间隔离。
- telegram 用轮询（long polling），消息先入队再串行/并发处理，避免在轮询回调里跑长任务阻塞。
- 会话状态按 chatId 维护在内存 Map。
- Node 单线程事件循环，异步 I/O。

## 7. 系统边界

**In-Scope**
- `server/utils/memories/`、`BackgroundWorkers/`、`PushNotifications/`、`telegramBot/`

**Out-of-Scope**
- 向量库检索实现 → ai-integrations 域
- Telegram 服务器、webhook 接收方 → 外部系统，不在本仓库源码内
- agent 对话循环本身 → `agent-runtime`

## 8. 与相邻子系统交互

- 上游：Telegram（外部）→ telegramBot → BackgroundWorkers → `agent-runtime`。
- 下游：agent → memories（向量库）；完成 → PushNotifications → 外部 webhook。

## 9. 语言专项适配口径（TS/JS 口径）

- **capability seam 分组**：Service Definition（队列、会话状态机）/ Provider（各渠道适配）/ Consumer（agent 对话）。
- **图类型**：architecture（组件拓扑）+ sequence（telegram 入站异步链，典型事件驱动）。
- **外部边界**：Telegram API、webhook、向量库。
- **并发**：队列 worker + 轮询，async 无锁。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 记忆与外部渠道拓扑 | `memory-external-channels-architecture.html` | architecture | showcase |
| telegram 入站时序 | `memory-external-channels-sequence.html` | sequence | showcase |

- 两图 render 退出码 0、HTML 非空。
- **降档披露**：两图首轮未过 showcase（垂直边 `clean-flow/endpoint-side-direction`、participant 标签过宽、自消息 0px 跨度），修复后均达 showcase。修复动作：垂直边显式 `fromSide/toSide` + `labelDy`；participant 标签缩为短名；删除自消息自环。
- **省略 dataflow/lifecycle**：无数据管道（已并入 sequence）；telegram 会话状态机较碎，文字描述即可。
- JSON IR 位于 `json/`。
