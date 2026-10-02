# cloud-deployments 云部署模板（cloud-deployments）

> 本文是 `deployment-ops` 域下的叶子子系统文档。域级总览见 `../deployment-ops.md`。
> 本文只展开**各云平台部署模板**：基于同一 Docker 镜像在 AWS/GCP/K8s/Helm/DigitalOcean/OpenShift/HuggingFace 等环境的编排定义；
> 镜像构建见 `docker-build`。
>
> 源码基准：`cloud-deployments/`，commit `128a015`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| AWS CloudFormation | JSON 模板一键起 EC2 实例 | `cloud-deployments/aws/cloudformation/cloudformation_create_anythingllm.json`、`DEPLOY.md`、`aws_https_instructions.md` |
| GCP Deployment Manager | YAML 模板部署 | `cloud-deployments/gcp/deployment/gcp_deploy_anything_llm.yaml`、`DEPLOY.md` |
| 原生 K8s 清单 | PV/PVC/Deployment/Service，EBS 持久卷，UID/GID 1000 | `cloud-deployments/k8/manifest.yaml:1-40` |
| Helm Chart | 完整 Chart（deployment/service/pvc/configmap/ingress/httproute/serviceaccount） | `cloud-deployments/helm/charts/anythingllm/`（Chart.yaml/values.yaml/templates/*） |
| DigitalOcean | Terraform 配置 | `cloud-deployments/digitalocean/terraform/` |
| OpenShift | 专用 Dockerfile 与 entrypoint | `cloud-deployments/openshift/`（Dockerfile/docker-entrypoint.sh/README.md） |
| HuggingFace Spaces | Space 的 Dockerfile | `cloud-deployments/huggingface-spaces/Dockerfile` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| CloudFormation JSON | `aws/cloudformation/*.json` | AWS 资源编排模板 |
| GCP DM YAML | `gcp/deployment/*.yaml` | GCP 资源编排 |
| K8s Manifest | `k8/manifest.yaml` | PV/PVC/Deployment/Service |
| Helm Chart | `helm/charts/anythingllm/` | 参数化 K8s 部署包 |
| OpenShift Dockerfile | `openshift/Dockerfile` | OpenShift 适配镜像 |

## 3. 关键调用链

**链 1：K8s 原生部署**
1. 模板定义 `PersistentVolume`（gp2/EBS，5Gi，ext4，zone us-east-1c）（`k8/manifest.yaml:1-20`）。
2. `PersistentVolumeClaim`（ReadWriteOnce，5Gi）（`k8/manifest.yaml:22-33`）。
3. `apps/v1` Deployment 拉起容器，挂载 PVC，以 UID/GID 1000 运行。
4. Service 暴露 3001 端口。

**链 2：Helm 部署**
1. `helm/charts/anythingllm/templates/` 含 deployment/service/pvc/configmap/ingress/httproute/serviceaccount/extra-objects。
2. `values.yaml` 参数化镜像、端口、持久卷。
3. `helm install` 渲染模板 apply。

**链 3：AWS/GCP 一键部署**
1. 用户在 AWS CloudFormation 控制台上传 JSON 模板，填参数（volumeID、namespace 等）。
2. 模板创建 EC2 实例并拉取 anythingllm 镜像运行。
3. 按 `aws_https_instructions.md` 配置 HTTPS。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---|---|---|
| 持久卷大小 | 5Gi，ReadWriteOnce | `k8/manifest.yaml:13-16/28-31` |
| UID/GID | 1000（与镜像运行用户一致） | `k8/manifest.yaml:4-5` |
| 可用区 | us-east-1c | `k8/manifest.yaml:19` |
| 端口 | 3001 | 各模板 Service |
| Helm values | 镜像 tag、存储、ingress | `helm/charts/anythingllm/values.yaml` |

## 5. 错误与重试语义

- 模板本身不含重试逻辑；部署失败由云编排器（CloudFormation/Helm）回滚。
- 容器内健康检查失败由编排器重调度（见 docker-build 叶子）。
- 持久卷未绑定导致启动失败属运维配置错误，模板通过占位符提示。

## 6. 并发细节

- 各云模板独立，无跨模板并发。
- K8s Deployment 副本数由 values 控制；单实例默认。
- server 与 collector 同容器并行（见 docker-build）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `cloud-deployments/` 全部模板与说明文档

**Out-of-Scope（不在本仓库源码内）**
- AWS/GCP/DO/OpenShift/HuggingFace 云平台本身、其控制面与计费，为外部云服务
- 镜像内容见 docker-build 叶子
- 业务代码见各域叶子

## 8. 与相邻子系统交互

- **docker-build → cloud-deployments**：所有模板引用同一 Docker 镜像。
- **cloud-deployments → 云平台**：在外部云基础设施上编排运行（不在本仓库源码内）。
- **cloud-deployments → 终端用户**：暴露 3001 端口提供 Web/API 服务。

## 9. 语言专项适配口径（JavaScript / TS-JS）

- **部署维度**：本叶子是 IaC 模板集合，部署维度用包发布 + 镜像引用表达，不涉及单二进制。
- **图型**：architecture（模板→镜像→云拓扑）+ sequence（部署编排流程）；无事件流管道与单实体状态机。
- **外部边界**：标注外部云平台（AWS/GCP/K8s/Helm 控制面）为外部系统，不在本仓库源码内。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 云部署模板架构图 | `cloud-deployments-architecture.html` | architecture | **showcase** |
| 云平台部署流程时序 | `cloud-deployments-sequence.html` | sequence | **showcase** |

- JSON IR 源文件：`json/cloud-deployments-architecture.json`、`json/cloud-deployments-sequence.json`。
- 省略说明：本叶子未生成 dataflow/lifecycle/workflow 图——部署是单次编排动作（已用 sequence 表达），无 ETL 管道或单实体状态机语义，按资源节省原则省略。
