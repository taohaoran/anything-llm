# deployment-ops 部署运维域（deployment-ops）

> 本域负责 anything-llm 的**镜像构建、容器编排、云平台部署模板与国际化语言包**（`docker/`、`cloud-deployments/`、`locales/`、`frontend/src/locales/`）。
> 域级总览：本文件；各叶子设计文档见下表。源码基准 commit `128a015`。

## 域职责

- Docker 构建：多阶段 Dockerfile（base→架构适配→前端构建→后端依赖→产物镜像）、docker-compose、entrypoint 双进程拉起、健康检查。
- 云部署模板：AWS CloudFormation、GCP Deployment Manager、原生 K8s、Helm Chart、DigitalOcean Terraform、OpenShift、HuggingFace Spaces。
- 国际化：i18next 初始化、语言检测、en 基准字典与 30+ 语言 common.js、翻译校验脚本。

## 叶子索引

| 叶子 | 职责 | 文档 | 图 |
|---|---|---|---|
| docker-build | Dockerfile 多阶段构建、docker-compose、镜像编排 | [docker-build/docker-build.md](docker-build/docker-build.md) | [架构](docker-build/docker-build-architecture.html) · [构建流](docker-build/docker-build-dataflow.html) |
| cloud-deployments | 各云平台（AWS/GCP/Azure/K8s/Helm 等）部署配置 | [cloud-deployments/cloud-deployments.md](cloud-deployments/cloud-deployments.md) | [架构](cloud-deployments/cloud-deployments-architecture.html) · [时序](cloud-deployments/cloud-deployments-sequence.html) |
| i18n-locales | 多语言 locale 文件、i18n 机制、语言切换 | [i18n-locales/i18n-locales.md](i18n-locales/i18n-locales.md) | [架构](i18n-locales/i18n-locales-architecture.html) · [时序](i18n-locales/i18n-locales-sequence.html) |

## 域级机制细节

- **单镜像多进程**：容器内 entrypoint 并行拉起 server（`:3001`）与 collector，`wait -n` 任一退出即终止；前端 `dist` 由 server 静态托管。
- **非 root 运行**：`ARG_UID/ARG_GID`（默认 1000）创建 `anythingllm` 用户；K8s/Helm 模板与镜像 UID 对齐。
- **构建优化**：前端用 `--platform=$BUILDPLATFORM` 原生编译避 QEMU；后端 `yarn install --production` 剥离 devDeps。
- **云模板复用同一镜像**：七类云平台模板均引用同一 Docker 镜像，仅编排层差异。
- **i18n 基准**：`en/common.js` 为键名 ground-truth，缺失键回退英文；`verify:translations` 在 CI 校验结构一致性。

## 图质量档位

本域 3 叶子各 2 图共 6 张，全部 showcase。
