# docker-build Docker 构建与编排（docker-build）

> 本文是 `deployment-ops` 域下的叶子子系统文档。域级总览见 `../deployment-ops.md`。
> 本文只展开**镜像构建与容器编排**：多阶段 Dockerfile、docker-compose、entrypoint 三进程拉起、健康检查；
> 云平台部署模板见 `cloud-deployments`，国际化见 `i18n-locales`。
>
> 源码基准：`docker/`，commit `128a015`，构建产物为容器镜像。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 基础镜像 | `ubuntu:noble-20251013` 作为 base | `docker/Dockerfile:2` |
| 架构适配阶段 | `build-arm64` / `build-amd64` 分别装系统依赖、Node 18、yarn、uvx 0.6.10 | `Dockerfile:8-72`、`:77-126` |
| 非 root 运行用户 | `ARG_UID/ARG_GID`（默认 1000）创建 `anythingllm` 用户，HOME=/app | `Dockerfile:5-6`、`:42-46`、`:111-115` |
| arm64 Chromium 补丁 | 手动装兼容 chromedriver 修复 Puppeteer | `Dockerfile:63-70` |
| 前端构建阶段 | `node:18-slim` + `--platform=$BUILDPLATFORM` 原生编译避 QEMU esbuild 崩溃 | `Dockerfile:141-147` |
| 后端依赖阶段 | server `yarn install --production`，collector 生产依赖 | `Dockerfile:151-161` |
| 产物合并 | `production-build` 把前端 `dist` 拷入 `server/public` | `Dockerfile:167-169` |
| 运行时环境 | `NODE_ENV=production`、`ANYTHING_LLM_RUNTIME=docker`、`DEPLOYMENT_VERSION=1.16.2` | `Dockerfile:172-174` |
| 健康检查 | `HEALTHCHECK` 每分钟 curl `/api/ping` | `Dockerfile:177-178`、`docker-healthcheck.sh:5-14` |
| 容器入口 | entrypoint 并行拉起 server 与 collector，`wait -n` | `docker-entrypoint.sh:19-29` |
| Compose 编排 | 单服务 `anything-llm`，端口 3001，挂载 storage/hotdir/outputs/.env | `docker/docker-compose.yml:7-31` |
| VEX 漏洞声明 | 4 条 CVE 豁免声明 | `docker/vex/*.vex.json` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `Dockerfile` 多阶段 | `docker/Dockerfile` | base→arch→frontend-build/backend-build→production-build |
| `docker-entrypoint.sh` | `docker/docker-entrypoint.sh:19` | prisma migrate + 双进程拉起 |
| `docker-healthcheck.sh` | `docker/docker-healthcheck.sh:5` | curl `/api/ping` 探活 |
| `docker-compose.yml` | `docker/docker-compose.yml:8` | 单服务编排与卷/端口映射 |
| `.env.example` | `docker/.env.example` | 全部环境变量样例 |

## 3. 关键调用链

**链 1：镜像构建流水线**
1. `FROM ubuntu:noble AS base`（`Dockerfile:2`）。
2. 按 `TARGETARCH` 选 `build-arm64` 或 `build-amd64`（`:8/:77`），装系统依赖、Node 18、yarn、uvx（`:15-38`）。
3. `FROM build-${TARGETARCH} AS build` 合并（`:131`）。
4. `frontend-build` 用 `--platform=$BUILDPLATFORM` 在原生架构 `yarn build`（`:141-146`）。
5. `backend-build` 拷 server 源码 `yarn install --production`（`:151-154`），再装 collector 生产依赖（`:158-161`）。
6. `production-build` `COPY --from=frontend-build /app/frontend/dist /app/server/public`（`:169`），设环境变量与 HEALTHCHECK（`:172-178`）。

**链 2：容器启动**
1. `ENTRYPOINT ["/bin/bash","/usr/local/bin/docker-entrypoint.sh"]`（`Dockerfile:182`）。
2. entrypoint 检查 `STORAGE_DIR` 未设则告警（`docker-entrypoint.sh:4-17`）。
3. `cd /app/server && npx prisma generate && npx prisma migrate deploy && node index.js`（`:20-25`）。
4. 后台 `node /app/collector/index.js`（`:27`）；`wait -n` 任一进程退出即 `exit $?`（`:28-29`）。

**链 3：健康检查**
1. `HEALTHCHECK` 每分钟执行 `docker-healthcheck.sh`（`Dockerfile:177`）。
2. `curl http://localhost:${SERVER_PORT:-3001}/api/ping`（`docker-healthcheck.sh:5`）；200 退出 0，否则退出 1（`:8-14`）。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---|---|---|
| `ARG_UID/ARG_GID` | 1000，运行用户与属组 | `Dockerfile:5-6`、`docker-compose.yml:14-15` |
| `SERVER_PORT` | 3001，server 监听与探活端口 | `docker-healthcheck.sh:5` |
| `STORAGE_DIR` | 未设则重启丢数据，entrypoint 告警 | `docker-entrypoint.sh:4` |
| `PUPPETEER_SKIP_CHROMIUM_DOWNLOAD`/`CHROME_PATH` | arm64 手动 Chromium 路径 | `Dockerfile:68-70` |
| 端口映射 | `3001:3001` | `docker-compose.yml:24-25` |
| 卷挂载 | `.env`、`server/storage`、`collector/hotdir`、`collector/outputs` | `docker-compose.yml:18-22` |
| `cap_add: SYS_ADMIN` | 容器权限 | `docker-compose.yml:16-17` |

## 5. 错误与重试语义

- **prisma migrate 失败**：entrypoint 中 `prisma migrate deploy` 失败会使 server 进程退出，`wait -n` 传播退出码，容器重启策略由编排决定（Dockerfile 内不重试）。
- **健康检查失败**：连续失败由 Docker 标记 unhealthy，镜像内不自动恢复。
- **STORAGE_DIR 未设**：仅告警不阻断启动（`docker-entrypoint.sh:4-17`），风险由运维承担。
- **任一进程退出**：`wait -n` 立即 `exit $?`，整容器退出（不做进程级重启）。

## 6. 并发细节

- **双进程并发**：server 与 collector 在同一容器内并行运行（`docker-entrypoint.sh:26-27`），`wait -n` 监听首个退出。
- 无锁/临界区；两进程通过文件系统（storage/hotdir/outputs）协作。
- 构建期前端用 `BUILDPLATFORM` 原生编译避免 QEMU 模拟下 esbuild 崩溃（`Dockerfile:138-141`）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `docker/Dockerfile`、`docker-compose.yml`、`docker-entrypoint.sh`、`docker-healthcheck.sh`、`.env.example`、`vex/`

**Out-of-Scope（不在本仓库源码内）**
- 基础镜像 `ubuntu:noble`、`node:18-slim`、Node/yarn/uvx 发行版为外部镜像与工具（不在本仓库源码内）
- Docker Hub、容器运行时、宿主机内核为外部基础设施
- server/collector/frontend 业务代码见各自域叶子
- 云平台部署模板 → cloud-deployments 叶子

## 8. 与相邻子系统交互

- **docker-build → server/collector/frontend**：构建产物把三组件打包进单镜像；server 静态托管前端 `dist`。
- **docker-build → cloud-deployments**：云模板基于本镜像部署。
- **docker-build → 宿主**：经 compose 暴露 3001 端口与持久卷。

## 9. 语言专项适配口径（JavaScript / TS-JS）

- **部署维度**：不适用单二进制；本叶子是 Vite SPA + Node server/collector 的多阶段镜像，依赖图以 npm/yarn 包与多阶段构建表达。
- **图型**：architecture（运行容器拓扑）+ dataflow（多阶段构建管道，天然 ETL 语义）；无单实体状态机与多角色泳道。
- **外部边界**：外部基础镜像、Docker Hub、容器运行时；不涉及 etcd/CRD。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 运行容器拓扑图 | `docker-build-architecture.html` | architecture | **showcase** |
| 多阶段构建流水线 | `docker-build-dataflow.html` | dataflow | **showcase** |

- JSON IR 源文件：`json/docker-build-architecture.json`、`json/docker-build-dataflow.json`。
- 省略说明：本叶子未生成 sequence/lifecycle/workflow 图——构建是线性阶段管道（已用 dataflow 表达），启动是脚本顺序执行（已在 MD 调用链说明），无多方消息时序或单实体状态机语义，按资源节省原则省略。
