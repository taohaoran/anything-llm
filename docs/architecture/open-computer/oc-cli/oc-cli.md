# open-computer CLI 入口（oc-cli）

> 本文是 `open-computer` 域下的叶子子系统文档。域级总览见 `../open-computer.md`。
> 本文展开 `open-computer/cli/`（TypeScript，tsx 运行）的命令行入口与命令分发，不重复展开
> VM 编排（见 `../oc-master/oc-master.md`）与服务层（见 `../oc-services/oc-services.md`）。
>
> 源码基准：anything-llm v1.16.2，commit `128a015`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| CLI 入口 | `oc` 命令主入口，tsx 启动 | `open-computer/cli/src/index.ts` |
| 命令注册 | `registry.ts` 集中注册全部子命令 | `cli/src/registry.ts` |
| 命令基类 | `BaseCommand`：公共参数、上下文 | `cli/src/commands/base.ts` |
| 控制命令 | start/stop/status/connect 等子命令 | `cli/src/commands/control.ts` |
| 配置加载 | `config.ts`：读 oc 配置与状态 | `cli/src/config.ts` |
| VM 操作 | 调 `vm.ts` 创建/启动/停止 VM | `cli/src/vm.ts`（被 oc-master 复用） |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `oc` program | `index.ts` | commander 实例，挂全部子命令 |
| `registerCommands` | `registry.ts` | 把 commands/ 下各命令挂到 program |
| `BaseCommand` | `commands/base.ts` | 命令基类，提供 run() 模板 |
| `Control` 命令 | `commands/control.ts` | start/stop/status/connect 等具体命令 |

## 3. 关键调用链

**链 1：`oc start`**

1. 终端 `oc start` → `index.ts` 启动 commander。
2. `registry.ts` 分发到对应 command（`control.ts`）。
3. command 调 `vm.ts` 创建/启动 VM、拉起 interface 服务，打印连接方式。

## 4. 配置项

| 配置 | 行为 | 位置 |
|---|---|---|
| oc 配置文件 | VM 规格、服务端口等 | `config.ts` |
| 命令 flag | 各子命令自定义参数 | `commands/control.ts` |

## 5. 错误与重试语义

- 命令执行失败打印错误并退出非零码；无自动重试。
- VM 操作失败由 vm.ts（oc-master）抛出，命令层捕获打印。

## 6. 并发细节

- 单命令进程，命令退出即结束；无长驻并发。
- 异步 spawn VM/服务子进程。

## 7. 系统边界

**In-Scope**：`cli/src/`（入口、注册、命令、配置读取）

**Out-of-Scope**：VM 生命周期实现（`vm.ts` 归 oc-master）、interface-service（oc-services）、底层 qemu/lima（外部）。

## 8. 与相邻子系统交互

- 上游：终端用户。
- 下游：oc-master（vm.ts/config.ts）→ VM；oc-services（interface-service）。

## 9. 语言专项适配口径（TS 口径）

- 本叶子是 TS（cli 目录），按 TS/JS 口径：capability seam 分 Service Definition（program/registry）/ Command（各子命令）/ Orchestrator（vm/config）。
- 图类型：architecture（CLI 组件）+ sequence（一次命令执行链）。
- 外部边界：qemu/lima、终端 stdio。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| CLI 组件拓扑 | `oc-cli-architecture.html` | architecture | standard |
| oc start 时序 | `oc-cli-sequence.html` | sequence | showcase |

- 两图 render 退出码 0。
- **降档披露**：架构图落 standard（组件间连线自动路由）；sequence 首轮因 participant 标签“interface-service”过宽（>86px）失败，缩短为“服务层”后达 showcase。
- JSON IR 位于 `json/`。
