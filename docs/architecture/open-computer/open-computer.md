# open-computer 域总览

> 本域覆盖 anything-llm 仓库内相对独立的子产品 **open-computer**：一个让 AI agent 真正操作一台桌面 VM（浏览器、桌面应用、bash）的系统，由 CLI、VM 编排、服务层三部分组成。
>
> 源码基准：anything-llm v1.16.2，commit `128a015`。CLI 为 TypeScript（tsx 运行），服务层为 JavaScript。

## 域职责

open-computer 解决“给 agent 一台真实可操作的计算机”的问题：CLI 拉起一个隔离 VM，VM 内跑 interface-service，service 驱动一个 pi agent 子进程，通过一组 MCP 扩展（浏览器 CDP、可见 bash、桌面应用、向用户提问）让 agent 点击/输入/截图来完成任务。它与 anything-llm 主 server 相对独立，是仓库内的子产品。

## 叶子索引

| 叶子 | 职责 | 文档 | 图 |
|---|---|---|---|
| oc-cli | `oc` 命令行入口与子命令分发（tsx/TS） | [oc-cli.md](oc-cli/oc-cli.md) | [架构](oc-cli/oc-cli-architecture.html) · [时序](oc-cli/oc-cli-sequence.html) |
| oc-services | interface-service：HTTP 服务、pi 进程、会话超管、能力扩展 | [oc-services.md](oc-services/oc-services.md) | [架构](oc-services/oc-services-architecture.html) · [数据流](oc-services/oc-services-dataflow.html) |
| oc-master | VM 生命周期编排：创建/启动/停止 VM、SSH 通道、状态文件 | [oc-master.md](oc-master/oc-master.md) | [架构](oc-master/oc-master-architecture.html) · [状态机](oc-master/oc-master-lifecycle.html) |

## 域级机制细节

- **三层结构**：CLI（oc-cli）→ VM 编排（oc-master/vm.ts）→ 服务层（oc-services/interface-service）。
- **agent 驱动方式**：service 以子进程拉起第三方 `pi agent`，经 stdin RPC 驱动；agent 通过 MCP 形态的扩展工具操作桌面。
- **与主 server 的边界**：open-computer 不接入主 `server/` 的 aibitat；它是独立子产品，VM/pi agent 均为外部依赖。
