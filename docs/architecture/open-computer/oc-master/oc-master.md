# open-computer 主控编排（oc-master）

> 本文是 `open-computer` 域下的叶子子系统文档。域级总览见 `../open-computer.md`。
> 本文展开 VM 生命周期编排（`cli/src/vm.ts`、`config.ts`、`registry.ts` 中的 VM 操作），不重复展开 CLI 命令分发（`../oc-cli/oc-cli.md`）与服务层（`../oc-services/oc-services.md`）。
>
> 源码基准：anything-llm v1.16.2，commit `128a015`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| VM 生命周期 | `VMManager`（vm.ts）：创建/启动/停止/状态查询 VM | `open-computer/cli/src/vm.ts` |
| 配置管理 | `config.ts`：读 oc 配置与持久化状态 | `cli/src/config.ts` |
| 命令注册 | `registry.ts`：把 VM 相关命令挂到 CLI | `cli/src/registry.ts` |
| SSH 通道 | VM 起来后经 SSH 连入、转发服务端口 | `cli/src/vm.ts` |
| 服务拉起 | VM 就绪后拉起 interface-service（见 oc-services） | `cli/src/vm.ts` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `VMManager` | `vm.ts` | VM 实例的创建与生命周期方法 |
| 配置/状态读取 | `config.ts` | 提供 VM 规格、端口、状态文件路径 |

## 3. 关键调用链

**链 1：`oc start` 拉起一个可用环境**

1. CLI（oc-cli）把 `start` 交给命令处理，后者调 `VMManager`。
2. `VMManager` 按 config 创建/启动 VM（qemu/lima），轮询等待 VM 就绪。
3. VM 就绪后建立 SSH 通道，端口转发，在 VM 内/或本地拉起 interface-service。
4. 服务就绪后打印连接方式回给用户。

**链 2：`oc stop`**：`VMManager` 优雅停止 VM、清理端口转发与临时状态文件。

## 4. 配置项

| 配置 | 行为 | 位置 |
|---|---|---|
| VM 规格 | CPU/内存/磁盘 | `config.ts` |
| 端口转发 | 服务对外端口 | `config.ts` |
| 状态文件 | 记录 VM 实例信息 | `config.ts` |

## 5. 错误与重试语义

- VM 启动失败：轮询超时后报错，状态标记 failed（图中省略，见 MD）；用户可 `oc start` 重试。
- SSH 连接失败重试若干次后放弃。
- 停止失败尽力清理，异常吞掉。

## 6. 并发细节

- 单 VM 串行生命周期；`oc start` 期间命令阻塞等待就绪。
- 子进程/SSH 连接异步管理。
- 无共享锁，单实例 CLI。

## 7. 系统边界

**In-Scope**：`cli/src/vm.ts`、`config.ts`、`registry.ts` 中 VM 编排部分

**Out-of-Scope**：qemu/lima 虚拟机运行时本身（外部）；OS 内服务部署（oc-services）；pi agent（外部子进程）。

## 8. 与相邻子系统交互

- 上游：oc-cli 命令。
- 下游：VM 运行时 → SSH → oc-services（interface-service）。

## 9. 语言专项适配口径（TS 口径）

- capability seam：Service Definition（VMManager + config）/ Provider（qemu/lima VM）/ Consumer（oc-cli 命令、oc-services）。
- 图类型：architecture（拓扑）+ lifecycle（VM 状态机，典型单实体状态迁移）。
- 外部边界：qemu/lima、SSH。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 主控编排拓扑 | `oc-master-architecture.html` | architecture | standard |
| VM 生命周期 | `oc-master-lifecycle.html` | lifecycle | standard |

- 两图 render 退出码 0。
- **降档披露**：两图落 standard。lifecycle 初版把 failed 状态放副 lane，跨 lane 边 `clean-flow/edge-through-node` 横穿 running 状态；按兜底原则简化为主 rail 三态（stopped→starting→running）+ 回边，failed 状态在第 5 节文字说明。
- **省略 sequence/dataflow**：编排时序已并入 oc-cli 的 sequence 图；VM 无数据管道。
- JSON IR 位于 `json/`。
