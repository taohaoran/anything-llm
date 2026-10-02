# MCP 桥接（mcp-bridge）

> 本文是 `agents-mcp` 域下的叶子子系统文档。域级总览见 `../agents-mcp.md`。
> 本文展开 anything-llm 作为 **MCP（Model Context Protocol）客户端** 的桥接层：管理外部 MCP server 进程、把其工具登记进 aibitat。
>
> 源码基准：anything-llm v1.16.2，commit `128a015`，纯 JavaScript ESM。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 客户端门面 | `MCP` 类：按 server 配置创建/复用 MCP client，供 aibitat 挂载 | `server/utils/MCP/index.js` |
| 超管（hypervisor） | `MCPHypervisor` 单例：负责连接生命周期、工具列举、注册表 | `server/utils/MCP/hypervisor/index.js` |
| 引导启动 | `bootMCPServers`：启动时批量连接所有启用的 MCP server | `hypervisor/index.js:493` |
| 单 server 启动 | `startMCPServer(config)`：按 transport 建 client 并 connect | `hypervisor/index.js:207` |
| 连接方式 | 支持 `stdio`（spawn 子进程）与 `SSE`/HTTP（远程）两种 transport | `hypervisor/index.js`（transport 判断） |
| 工具列举与登记 | `listTools` 后把每个 MCP 工具包装成 `aibitat.function` 登记 | `hypervisor/index.js` |
| 工具调用 | aibitat 调用时经 client `tools/call` 转发给 MCP server | `MCP/index.js` |
| 清理 | `pruneMCPServer`：关闭 client、断开、从注册表移除 | `hypervisor/index.js` |
| 工具中断 | 每个 MCP 工具 handler 自带 `AbortController`，可单独取消 | `MCP/index.js` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `MCP`（门面类） | `MCP/index.js` | 对外暴露 `getToolsForServer`/`callTool` 等，内部委托 hypervisor |
| `MCPHypervisor` | `hypervisor/index.js` | 单例，持有已连接 client 表与 tool 注册表 |
| `Client`（`@modelcontextprotocol/sdk`） | 外部 SDK | MCP 协议客户端，提供 `connect/listTools/callTool`（不在本仓库源码内） |
| transport | stdio 子进程 / SSE HTTP | 两种 server 连接方式 |

## 3. 关键调用链

**链 1：引导阶段连接 MCP server**

1. 服务启动 → `MCPHypervisor.bootMCPServers()`（`hypervisor/index.js:493`）读取已启用 server 配置。
2. 对每个配置 `startMCPServer(config)`（`:207`）：按 `config.transport` 选择 stdio 或 SSE，new `Client`，`connect()`。
3. 连接成功后 `client.listTools()` 取工具清单，逐个包装成 aibitat 可调用函数登记。
4. 单个 server 失败不阻塞其他 server（错误记入日志，继续引导下一个）。

**链 2：LLM 调用一个 MCP 工具**

1. aibitat `functions` Map 里某工具 `isMCPTool=true`，其 handler 内部调 `MCP.callTool(serverName, toolName, args)`。
2. hypervisor 从 client 表取对应 client，`client.callTool({name, arguments})`。
3. MCP server 返回结果，经门面回给 aibitat 作为 `{role:"function"}` 消息回灌。
4. 调用方可用该工具 handler 的 `AbortController` 中途取消。

## 4. 配置项

| 配置 | 行为 | 位置 |
|---|---|---|
| server 配置 `transport` | `stdio` 或 `sse`/`http` | `startMCPServer` |
| server 配置 `command`/`args`/`env` | stdio 模式 spawn 子进程的命令行 | stdio transport |
| server 配置 `url` | SSE/HTTP 模式的远端地址 | SSE transport |
| 启用/禁用 | 是否纳入 `bootMCPServers` | server 记录 |

## 5. 错误与重试语义

- 单个 MCP server 连接失败：引导阶段记日志跳过，不影响其他 server 与主服务启动。
- 工具调用失败：错误作为字符串结果回灌给 LLM，由模型决定如何告知用户。
- `pruneMCPServer`：关闭 client 时尽量优雅断开，异常吞掉避免崩溃。
- 无自动重连/退避；重连靠重启或重新引导。

## 6. 并发细节

- 每个 MCP server 一个 client 实例；工具调用之间靠 Node 事件循环异步。
- 每个 MCP 工具 handler 带独立 `AbortController`，取消只中断该次调用。
- 子进程（stdio）由 hypervisor 持有句柄，`prune` 时 kill。
- 共享注册表为内存 Map，单写多读。

## 7. 系统边界

**In-Scope**
- `server/utils/MCP/`：门面、hypervisor、连接生命周期、工具包装

**Out-of-Scope**
- MCP SDK 本身（`@modelcontextprotocol/sdk`）→ 外部依赖，不在本仓库源码内
- 具体 MCP server 进程实现 → 外部程序，不在本仓库源码内
- aibitat 工具注册机制 → `agent-runtime` 叶子
- 内置工具实现 → `agent-tools` 叶子

## 8. 与相邻子系统交互

- 上游：`agent-runtime`（aibitat 挂载 MCP 插件时经 `MCP` 门面取工具）。
- 下游：MCP client → 外部 MCP server（stdio 子进程 / 远程 HTTP）。
- 侧向：启动引导（server 入口）调 `bootMCPServers`。

## 9. 语言专项适配口径（TS/JS 口径）

- **capability seam 分组**：Service Definition（MCP 门面 + hypervisor 注册表）/ Provider（各外部 MCP server）/ Consumer（aibitat 把工具当内置函数调用）。
- **图类型**：architecture（桥接拓扑）+ sequence（引导与调用时序，典型异步 RPC）。
- **外部边界**：MCP SDK、外部 MCP server 进程、远程 HTTP。
- **并发**：async RPC + AbortController，无锁。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| MCP 桥接拓扑 | `mcp-bridge-architecture.html` | architecture | standard |
| 引导与工具调用时序 | `mcp-bridge-sequence.html` | sequence | showcase |

- 两图 render 退出码 0、HTML 非空。
- **降档披露**：架构图目标 showcase 落 standard。失败检查为 `clean-flow/endpoint-side-direction`（client 到子进程的垂直边被误判为水平）；采取的修复：显式 `fromSide:bottom`/`toSide:top` 并对垂直边加 `labelDy`。
- **省略 dataflow/lifecycle**：本叶子是 RPC 桥接，无数据管道（数据流动已并入 sequence）、无单实体状态机（连接生命周期较简单），按资源节省原则省略。
- JSON IR 位于 `json/`。
