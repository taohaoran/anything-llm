# open-computer 服务层（oc-services）

> 本文是 `open-computer` 域下的叶子子系统文档。域级总览见 `../open-computer.md`。
> 本文展开 `open-computer/services/interface-service/`：HTTP 服务、会话编排、扩展能力（浏览器控制、桌面操作、bash、OCR 等）。
>
> 源码基准：anything-llm v1.16.2，commit `128a015`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| HTTP 服务 | Express 服务入口，挂路由 | `open-computer/services/interface-service/index.js` |
| API 路由 | 会话/任务相关 REST 端点 | `services/interface-service/routes/api.js` |
| pi agent 子进程 | 以子进程方式拉起 pi agent，经 stdin RPC 驱动 | `services/interface-service/pi/process.js` |
| 会话超管 | `session/hypervisor.js` 管理会话状态 | `services/interface-service/session/hypervisor.js` |
| 浏览器控制 | browser-cdp 扩展：经 CDP 操控浏览器 | `services/extensions/browser-cdp/` |
| 可见 bash | visible-bash 扩展：在可见桌面执行命令 | `services/extensions/visible-bash/` |
| 桌面应用 | desktop-apps 扩展：打开/操作桌面应用 | `services/extensions/desktop-apps/` |
| 向用户提问 | ask-user 扩展：任务中向用户确认 | `services/extensions/ask-user/` |
| 其他扩展 | 截图/OCR 等能力 | `services/extensions/` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| Express app | `interface-service/index.js` | 服务实例，挂 routes/websocket |
| pi 进程 | `pi/process.js` | 子进程包装，stdin RPC 协议 |
| session hypervisor | `session/hypervisor.js` | 会话状态、任务生命周期 |
| 扩展（MCP 工具形态） | `extensions/*` | 暴露给 pi agent 的可调用工具 |

## 3. 关键调用链

**链 1：一次“让 agent 操作桌面”的循环**

1. HTTP 请求进入 → routes/api.js → interface-service 创建/复用会话（hypervisor）。
2. service 拉起 pi agent 子进程（`pi/process.js`），经 stdin RPC 下发任务。
3. pi agent 决策要调工具 → 经 MCP 扩展调用 browser-cdp/visible-bash/desktop-apps。
4. 扩展在 VM 桌面环境执行动作（点击/输入/截图），把观测结果（截图/文本）回给 pi。
5. pi 根据观测继续下一步，直到任务完成；结果经 service 返回。

## 4. 配置项

| 配置 | 行为 | 位置 |
|---|---|---|
| 服务端口 | interface-service 监听端口 | `config` |
| 扩展开关 | 启用哪些能力扩展 | extensions 加载处 |
| pi agent 路径/参数 | 子进程启动命令 | `pi/process.js` |

## 5. 错误与重试语义

- pi 子进程异常退出 → hypervisor 标记会话失败，HTTP 层返回错误。
- 工具调用失败 → 错误作为观测结果回给 pi，由 agent 决定重试或换策略。
- 无服务端自动重试；重试在 agent 推理循环内。

## 6. 并发细节

- pi agent 是独立子进程，service 经 stdin/stdout 异步通信。
- 多会话各自独立 hypervisor 状态。
- Node 事件循环异步 I/O。

## 7. 系统边界

**In-Scope**：`services/interface-service/`、`services/extensions/`

**Out-of-Scope**：pi agent CLI 本身（外部子进程，不在本仓库源码内）；qemu/lima VM 运行时；OS 桌面与浏览器二进制。

## 8. 与相邻子系统交互

- 上游：oc-cli（oc-master）拉起本服务；HTTP 客户端。
- 下游：pi agent 子进程、VM 内浏览器/桌面。

## 9. 语言专项适配口径（JS 口径）

- capability seam：Service Definition（Express app + routes）/ Provider（各扩展工具）/ Consumer（pi agent）。
- 图类型：architecture（拓扑）+ dataflow（agent→工具→环境→观测的数据循环）。
- 外部边界：pi agent 子进程、浏览器 CDP、OS。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 服务层拓扑 | `oc-services-architecture.html` | architecture | standard |
| 桌面操作数据流 | `oc-services-dataflow.html` | dataflow | showcase |

- 两图 render 退出码 0。
- **降档披露**：架构图落 standard（自动路由）；dataflow 初版因 return 边 back→pi 横穿全部节点与标签重叠失败，删除回路边（循环语义在第 3 节文字描述）后达 showcase。
- **省略 sequence/lifecycle**：调用链已在 architecture+dataflow 表达；无独立状态机。
- JSON IR 位于 `json/`。
