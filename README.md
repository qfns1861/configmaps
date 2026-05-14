# configmaps

> CI/CD 配套仓库 · Helm Values / ArgoCD 配置 / App 配置

本仓库是 CI/CD 流水线的配套配置仓库，集中管理所有集群的 **Helm Chart values**、**ArgoCD ApplicationSet** 以及 **应用级别参数**。与 `charts` 仓库配合使用，实现 GitOps 部署。

---

## 目录结构

```
configmaps/
├── cicd.yaml                  # 项目标识与 CI/CD 基础参数
├── default-config.yaml        # 全局平台级 ConfigMap（ArgoCD 命名空间）
│
├── default/                   # 默认（基线）配置，所有集群共享
│   ├── cluster.yaml           # default 集群环境变量
│   ├── telegraf/
│   │   ├── k8s.yaml           # telegraf K8s / ArgoCD 部署参数
│   │   └── cfg.yaml           # telegraf 采集器配置
│   └── categraf/
│       ├── k8s.yaml           # categraf K8s / ArgoCD 部署参数
│       └── cfg.yaml           # categraf 采集器配置
│
└── hf01/                      # hf01 集群专属覆盖配置
    ├── cluster.yaml           # hf01 集群环境变量
    ├── hf01-apps.yaml         # ArgoCD ApplicationSet（hf01）
    ├── telegraf/
    │   ├── k8s.yaml           # 覆盖：副本数、镜像、Cilium 等
    │   └── cfg.yaml           # 覆盖：采集器配置
    └── categraf/
        ├── k8s.yaml           # 覆盖：副本数、镜像版本
        └── cfg.yaml           # 覆盖：采集器配置
```

---

## 配置分层机制

ArgoCD 在部署时按以下顺序**依次加载并合并** values 文件，**后加载的优先级更高**：

```
default/<app>/k8s.yaml          ← 基线 K8s 参数
default/<app>/cfg.yaml          ← 基线采集器配置
default/cluster.yaml            ← 基线集群环境变量
<cluster>/<app>/k8s.yaml        ← 集群专属 K8s 参数覆盖
<cluster>/<app>/cfg.yaml        ← 集群专属采集器配置覆盖
<cluster>/cluster.yaml          ← 集群专属环境变量覆盖
```

> 此合并顺序由 `hf01/hf01-apps.yaml` 中的 `spec.sources[0].helm.valueFiles` 定义。

---

## 全局平台配置（default-config.yaml）

`default-config.yaml` 以 Kubernetes ConfigMap 形式部署到 `argo` 命名空间，提供基础设施级别的全局参数：

| 参数 | 值 | 说明 |
|---|---|---|
| `argo_server` | `http://argo.qfns.com:31549` | Argo Workflows 服务地址 |
| `argocd_server` | `argocd-server.argocd.svc.cluster.local` | ArgoCD 内部服务地址 |
| `gitlab_url` | `http://192.168.16.128` | GitLab 地址 |
| `helm_version` | `3.16.4` | Helm 工具版本 |
| `kube_versions` | `1.28.0, 1.32.0` | 支持的 K8s 版本范围 |
| `image_python_git` | `python-git:1.0.0` | 通用 Python+Git 镜像 |
| `image_helm` | `alpine/helm:3.16.4` | Helm 执行镜像 |
| `image_argocd` | `argocd-python:v3.3.4` | ArgoCD 操作镜像 |

---

## 集群清单

### default

| 字段 | 值 |
|---|---|
| 集群标识 | `default` |
| 集群名称 | `default` |
| 集群类型 | `dmz` |
| 命名空间 | `default` |
| K8s 版本 | `1.32` |
| 时区 | `UTC` |
| DNS | `qfns.cicd.com` |
| 内网 IP | `10.0.0.1` |
| 部署应用 | `telegraf` |
| 中间件 | PostgreSQL ✅ / Redis ✅ / Kafka ✅ / Prometheus ✅ / VictoriaLogs ✅ |

### hf01

| 字段 | 值 |
|---|---|
| 集群标识 | `hf01` |
| 集群名称 | `dmz-hf01` |
| 集群类型 | `dmz` |
| 命名空间 | `hf01` |
| K8s 版本 | `1.32` |
| 时区 | `UTC` |
| DNS | `siyi.cicd.com` |
| 内网 IP | `10.0.0.1` |
| 部署应用 | `telegraf`、`categraf` |
| 特性 | Canary ✅ / Vector ✅ / VictoriaLogs ✅ / Kafka ✅ |

---

## ArgoCD ApplicationSet（hf01）

文件：`hf01/hf01-apps.yaml`

- **类型**：`ApplicationSet`（`argoproj.io/v1alpha1`）
- **Generator**：Git 文件生成器，扫描 `hf01/**/k8s.yaml`，每个文件对应一个 Application
- **应用命名规则**：`<cluster>-<app>`（如 `hf01-telegraf`）
- **Chart 来源**：从 `k8s.yaml` 中的 `argo.chartRepo` + `argo.chart` + `argo.chartVersion` 动态读取
- **Values 来源**：Git 仓库 `configmaps.git`，分支由 `argo.valuesBranch` 指定（默认 `HEAD`）
- **部署目标**：`argo.server` 指定的集群，`argo.namespace` 指定的命名空间
- **同步策略**：`Replace=true`
- **版本保护**：CI/CD 临时修改的 `targetRevision`（chart 版本、values 分支）不被 ApplicationSet 覆盖

---

## 应用配置说明

### telegraf

**采集器**，负责指标采集并暴露 Prometheus `/metrics` 端点。

#### 默认配置（`default/telegraf/`）

| 参数 | 值 |
|---|---|
| Chart 版本 | `1.8.57` |
| 镜像 | `telegraf:1.32.1-alpine-cicd` |
| 副本数 | `1` |
| 命名空间 | `hf01` |
| Prometheus 端口 | `9273` |
| CPU Request/Limit | `250m` / `500m` |
| Memory Request/Limit | `64Mi` / `128Mi` |
| Service 类型 | `ClusterIP` |
| PDB minAvailable | `1` |
| Cilium | ❌ 禁用 |

采集项（`cfg.yaml`）：

| 采集项 | 说明 |
|---|---|
| `cpu` | 每核 + 总量，报告活跃状态 |
| `net_response` | 探测 PostgreSQL / Redis / Prometheus / VictoriaLogs / Kafka 可达性 |

#### hf01 覆盖（`hf01/telegraf/`）

| 参数 | 默认值 → 覆盖值 |
|---|---|
| Chart 版本 | `1.8.57` → `1.8.59` |
| 副本数 | `1` → `3` |
| Cilium | ❌ → ✅ 启用 |
| vector | ✅ → ❌ 禁用 |
| victoriaLogs | ✅ → ❌ 禁用 |
| canary | ❌ → ✅ 启用 |

---

### categraf

**采集器**，负责指标采集并将数据写入 Nightingale（夜莺）监控平台。

#### 默认配置（`default/categraf/`）

| 参数 | 值 |
|---|---|
| Chart 版本 | `1.0.0` |
| 镜像 | `categraf:latest` |
| 副本数 | `3` |
| 命名空间 | `hf01` |
| HTTP 指标端口 | `9100` |
| Service 类型 | `ClusterIP` |
| PDB minAvailable | `1` |
| Cilium | ✅ 启用 |
| Tolerations | `NoSchedule: Exists`（部署到所有节点） |
| RBAC | ✅ 自动创建，含 nodes/pods/services/ingresses 读权限 |
| ServiceAccount | `categraf-serviceaccount` |

采集配置（`cfg.yaml`）：

| 参数 | 值 |
|---|---|
| Writer URL | `http://nserver-service:17000/prometheus/v1/write` |
| 采集间隔 | `15s` |
| 精度 | `ms` |
| Hostname | `$HOSTNAME`（Pod 环境变量） |

#### hf01 覆盖（`hf01/categraf/`）

| 参数 | 默认值 → 覆盖值 |
|---|---|
| 镜像 | `categraf:latest` → `flashcatcloud/categraf:v0.5.4` |
| vector | ✅ → ❌ 禁用 |
| victoriaLogs | ✅ → ❌ 禁用 |
| canary | ❌ → ✅ 启用 |

---

## 新增集群 / 应用

### 新增集群

1. 在仓库根目录创建 `<cluster>/` 目录
2. 添加 `<cluster>/cluster.yaml`，填写集群元信息
3. 创建 `<cluster>/<cluster>-apps.yaml`（参考 `hf01/hf01-apps.yaml`）
4. 为每个应用创建 `<cluster>/<app>/k8s.yaml`，至少填写 `argo.*` 必填字段

### 新增应用

1. 在 `default/<app>/` 下创建 `k8s.yaml`（基线 K8s 参数）和 `cfg.yaml`（基线采集配置）
2. 如需集群专属配置，在 `<cluster>/<app>/` 下创建对应覆盖文件
3. 在对应集群的 `cluster.yaml` 中将应用名加入 `cluster.application` 列表

### k8s.yaml 必填字段

```yaml
argo:
  app: <应用名>          # ArgoCD Application 名称的一部分
  cluster: <集群名>      # 集群标识
  chart: <chart 名>      # Helm Chart 名称
  chartVersion: <版本>   # Helm Chart 版本号
  chartRepo: <仓库地址>  # Helm Chart 仓库 URL
  server: <K8s API>      # 目标集群 API Server 地址
```

---

## 相关仓库

| 仓库 | 地址 | 说明 |
|---|---|---|
| charts | `http://192.168.16.128/root/charts.git` | Helm Chart 源码 |
| configmaps | `http://192.168.16.128/root/configmaps.git` | 本仓库 |
| chart registry | `http://192.168.16.128:30380` | Helm Chart 仓库（Harbor） |
| ArgoCD | `http://argocd-server.argocd.svc.cluster.local` | GitOps 控制台 |
| GitLab | `http://192.168.16.128` | 代码托管 |
