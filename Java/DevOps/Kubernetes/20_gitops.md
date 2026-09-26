# 1.GitOps 概述

## 1.1 什么是 GitOps

GitOps 是一套以 Git 为核心的操作模型，由 Weaveworks 在 2017 年提出。其核心思想极其简洁：**Git 仓库是集群期望状态的唯一真理源（Single Source of Truth），集群的实际状态由自动化工具持续向 Git 中的期望状态收敛。**

```
传统 CI/CD 模式（Push）:
  开发者 → git push → CI 构建镜像 → CI 执行 kubectl apply → 集群变更
  
  问题：CI 拥有集群写权限；部署操作分散在 Pipeline 中；集群状态与 Git 可能漂移

GitOps 模式（Pull）:
  开发者 → git push → CI 构建镜像 + 更新 Manifest 仓库
  
  ┌────────────────────────────────┐
  │        Git Repository          │  真理源
  │  deployment.yaml               │
  │  configmap.yaml                │
  │  service.yaml                  │
  └──────────────┬─────────────────┘
                 │ 定期 Pull + 对比
                 ▼
  ┌────────────────────────────────┐
  │      GitOps Agent              │
  │    (ArgoCD / FluxCD)           │
  │                                 │
  │  持续执行:                      │
  │  1. 获取 Git 中的期望状态       │
  │  2. 获取集群中的实际状态        │
  │  3. 如果不同 → 调谐集群向 Git 收敛│
  └──────────────┬─────────────────┘
                 │ Apply / Sync
                 ▼
        ┌────────────────┐
        │  K8s Cluster   │
        └────────────────┘
```

### 核心原则

1. **声明式（Declarative）**：用 YAML/JSON 描述期望状态，而非编写操作脚本
2. **版本化与不可变（Versioned & Immutable）**：所有变更通过 Git 提交记录，可审计、可回滚
3. **自动拉取（Pulled Automatically）**：Agent 从 Git 拉取期望状态并自动应用到集群
4. **持续调谐（Continuously Reconciled）**：Agent 持续监控集群状态，自动修复漂移

## 1.2 GitOps vs 传统 CI/CD

| 维度 | Push 模式（传统 CI/CD） | Pull 模式（GitOps） |
|------|----------------------|-------------------|
| **触发方式** | CI 主动推送（kubectl apply） | Agent 主动拉取（Git → Cluster） |
| **集群访问** | CI 需要集群写权限（安全隐患） | Agent 在集群内运行，无需外部凭证 |
| **状态漂移** | 手动操作导致漂移，难以发现 | Agent 自动检测并修复漂移 |
| **回滚** | 手动执行回滚任务 | git revert + 自动同步 |
| **审计** | CI 日志分散 | Git 历史即审计日志 |
| **多集群** | 每个集群需配置 CI | Agent 部署到每个集群即可 |
| **灾难恢复** | 重建集群 + 恢复备份 + 重新部署 | bootstrap 集群 + 自动从 Git 同步所有资源 |

# 2.ArgoCD

## 2.1 ArgoCD 架构

ArgoCD 是 CNCF 毕业的 GitOps 工具，使用 Go 语言开发。其架构设计清晰：

```
┌─────────────────────────────────────────────────────┐
│                    ArgoCD                            │
│                                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │ API Server   │  │  Repository  │  │ Application │ │
│  │ (REST/gRPC)   │  │   Server     │  │ Controller  │ │
│  │              │  │ (Git 仓库管理) │  │ (调谐引擎)   │ │
│  └──────────────┘  └──────────────┘  └──────┬─────┘ │
│                                             │        │
│                                 ┌───────────▼──────┐ │
│                                 │  Redis Cache     │ │
│                                 └──────────────────┘ │
└─────────────────────────────────────────────────────┘
              │                        │
              ▼                        ▼
     ┌─────────────┐          ┌─────────────┐
     │  Git Repo    │          │ K8s Cluster  │
     │ (期望状态)    │          │  (实际状态)    │
     └─────────────┘          └─────────────┘
```

### Application CRD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/my-app-config
    targetRevision: main              # Git 分支/tag/commit
    path: overlays/production         # Kubernetes 清单目录
    # 或使用 Helm
    # helm:
    #   valueFiles:
    #   - values-production.yaml
  destination:
    server: https://kubernetes.default.svc  # 目标集群
    namespace: production
  syncPolicy:
    automated:                        # 自动同步
      prune: true                     # 自动删除 Git 中已移除的资源
      selfHeal: true                  # 自动修复集群中的手动修改
    syncOptions:
    - CreateNamespace=true            # 自动创建 Namespace
    - PruneLast=true                  # 删除资源后再创建新资源
  ignoreDifferences:                  # 忽略的差异
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas                  # 忽略 HPA 导致的 replicas 变化
  revisionHistoryLimit: 10            # 保留的历史版本数
```

### Application 的状态同步

ArgoCD 每 3 分钟（可配置）检查一次 Git 仓库和集群状态的差异：

```
Git (期望)           Cluster (实际)         状态
────────────────────────────────────────────────
replicas: 3          replicas: 3           Synced (已同步)
replicas: 3          replicas: 2           OutOfSync (差一个副本)
image: nginx:1.25    image: nginx:1.24     OutOfSync (镜像版本不同)
config: {key: v2}    config: {key: v1}     OutOfSync (配置不同)
(secret不存在)        secret存在            OutOfSync (需被 prune)
```

## 2.2 ArgoCD 安装与使用

```bash
# 安装 ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 暴露服务（NodePort/Ingress/Port Forward）
kubectl port-forward svc/argocd-server -n argocd 8080:443

# 获取初始密码
argocd admin initial-password -n argocd

# 登录
argocd login localhost:8080 --insecure

# 创建 Application
argocd app create my-app \
  --repo https://github.com/myorg/my-app-config.git \
  --path overlays/production \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace production \
  --sync-policy automated \
  --auto-prune \
  --self-heal

# 查看状态
argocd app get my-app
argocd app list

# 手动同步
argocd app sync my-app

# 回滚
argocd app rollback my-app

# 查看差异（Diff）
argocd app diff my-app

# 查看事件
argocd app history my-app
```

## 2.3 App of Apps 模式

在大型组织中，使用单个 Application 管理所有资源不可行。App of Apps 模式使用一个"根 Application"管理其他 Application：

```yaml
# 根 Application：管理所有子 Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/myorg/argocd-apps
    targetRevision: main
    path: apps/
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
```

```
argocd-apps/
├── apps/
│   ├── team-a/
│   │   ├── frontend-app.yaml
│   │   └── backend-app.yaml
│   ├── team-b/
│   │   └── data-service.yaml
│   └── infrastructure/
│       ├── ingress-nginx.yaml
│       └── cert-manager.yaml
└── projects/
    ├── team-a-project.yaml
    └── team-b-project.yaml
```

## 2.4 ApplicationSet：模板化部署

ApplicationSet 允许用模板生成多个 Application，适合多集群、多环境场景：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook
spec:
  generators:
  - list:                              # 列表生成器
      elements:
      - cluster: dev
        url: https://dev-cluster.example.com
      - cluster: staging
        url: https://staging-cluster.example.com
      - cluster: prod
        url: https://prod-cluster.example.com
  template:
    metadata:
      name: '{{cluster}}-guestbook'
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/guestbook-config
        targetRevision: main
        path: overlays/{{cluster}}
      destination:
        server: '{{url}}'
        namespace: guestbook
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

支持的生成器：
- `list`：静态列表
- `git`：Git 仓库中目录/文件变化自动生成
- `cluster`：自动发现托管集群
- `matrix`：组合多个生成器
- `merge`：合并生成结果
- `pullRequest`：为每个 Pull Request 创建临时环境

## 2.5 ArgoCD 安全与权限

```yaml
# ArgoCD Project：定义权限边界
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-a
  namespace: argocd
spec:
  description: "Team A 的项目"
  sourceRepos:
  - https://github.com/myorg/team-a-*
  destinations:
  - namespace: team-a-*
    server: https://kubernetes.default.svc
  - namespace: team-a-*
    server: https://staging-cluster.example.com
  clusterResourceWhitelist:
  - group: ''
    kind: Namespace
  - group: networking.k8s.io
    kind: NetworkPolicy
  namespaceResourceBlacklist:
  - group: ''
    kind: Secret          # 禁止 ArgoCD 管理 Secret
  roles:                   # 项目内 RBAC
  - name: developer
    policies:
    - p, proj:team-a:developer, applications, sync, team-a/*, allow
  - name: read-only
    policies:
    - p, proj:team-a:read-only, applications, get, team-a/*, allow
```

# 3.FluxCD

## 3.1 FluxCD 架构

FluxCD 由 Weaveworks 创建（GitOps 概念的提出者），现已捐赠给 CNCF（孵化阶段）。相比 ArgoCD，FluxCD 更加组件化和模块化：

```
┌──────────────────────────────────────────────────────────┐
│                      FluxCD                               │
│                                                            │
│  ┌───────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐ │
│  │ Source    │  │ Kustomize│  │ Helm     │  │ Notifi- │ │
│  │ Controller│  │ Controller│  │ Controller│  │ cation  │ │
│  │           │  │          │  │          │  │Controller│ │
│  │ Git/OCI   │  │ 应用 Kust│  │ 管理 Helm│  │ Slack/  │ │
│  │ 仓库管理  │  │ 资源     │  │ Releases │  │ Teams   │ │
│  └───────────┘  └──────────┘  └──────────┘  └─────────┘ │
│                                                            │
│  ┌───────────┐  ┌──────────────┐                          │
│  │ Image     │  │ Image Reflect│                          │
│  │ Automation│  │ Controller   │                          │
│  │ Controller│  │              │                          │
│  │           │  │ 监视镜像仓库  │                          │
│  │ 自动更新   │  │ 发现新版本    │                          │
│  │ Git 配置  │  │              │                          │
│  └───────────┘  └──────────────┘                          │
└──────────────────────────────────────────────────────────┘
```

### FluxCD 的核心 CRD

```yaml
# GitRepository：定义 Git 源
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 1m                    # 检查间隔
  url: https://github.com/myorg/my-app
  ref:
    branch: main
  secretRef:
    name: git-credentials         # Git 认证凭据

---
# Kustomization：应用 Kustomize 资源
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 5m
  path: ./overlays/production
  prune: true                     # 自动删除 Git 中移除的资源
  sourceRef:
    kind: GitRepository
    name: my-app
  targetNamespace: production
  healthChecks:                   # 同步后的健康检查
  - apiVersion: apps/v1
    kind: Deployment
    name: my-app
    namespace: production
```

## 3.2 FluxCD 安装与 Bootstrap

```bash
# 安装 Flux CLI
winget install FluxCD.Flux

# Bootstrap：从 Git 仓库引导集群
flux bootstrap github \
  --owner=myorg \
  --repository=fleet-infra \
  --branch=main \
  --path=./clusters/production \
  --personal

# Bootstrap 做了什么？
# 1. 在 Git 仓库中创建 flux-system/ 目录结构
# 2. 部署 FluxCD 组件到集群
# 3. 配置 FluxCD 从 Git 仓库同步
# 4. 输出 Deploy Key 供 Git 仓库认证
```

Bootstrap 之后的仓库结构（Monorepo）：

```
fleet-infra/
├── clusters/
│   ├── production/
│   │   ├── flux-system/              # FluxCD 自身配置
│   │   │   ├── gotk-components.yaml
│   │   │   ├── gotk-sync.yaml
│   │   │   └── kustomization.yaml
│   │   └── apps/
│   │       ├── team-a/
│   │       │   ├── frontend.yaml      # Kustomization
│   │       │   └── backend.yaml
│   │       └── team-b/
│   │           └── data-service.yaml
│   └── staging/
│       └── flux-system/
│           └── ...
├── infrastructure/
│   ├── ingress-nginx.yaml
│   └── cert-manager.yaml
└── apps/
    ├── base/
    │   └── guestbook/
    └── overlays/
        ├── production/
        └── staging/
```

## 3.3 Helm 集成

```yaml
# HelmRelease：以 GitOps 方式管理 Helm Chart
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
spec:
  interval: 5m
  chart:
    spec:
      chart: ingress-nginx
      version: "4.x"
      sourceRef:
        kind: HelmRepository
        name: ingress-nginx
        namespace: flux-system
  values:
    controller:
      service:
        type: LoadBalancer
      metrics:
        enabled: true
      replicaCount: 3
      resources:
        requests:
          cpu: 200m
          memory: 256Mi
```

## 3.4 自动镜像更新

FluxCD 的独特能力是**自动镜像版本更新**。它可以将镜像仓库中的新版本自动更新到 Git 仓库中：

```yaml
# ImageRepository：监视镜像仓库
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: my-app
spec:
  image: docker.io/myorg/my-app
  interval: 5m                    # 每 5 分钟检查新版本

---
# ImagePolicy：定义版本策略
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: my-app
spec:
  imageRepositoryRef:
    name: my-app
  policy:
    semver:                       # 语义版本策略
      range: 1.0.x                # 自动更新补丁版本

---
# 在 Deployment 中标记可被自动更新
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: app
        image: docker.io/myorg/my-app:1.0.0  # {"$imagepolicy": "flux-system:my-app"}
```

当镜像仓库出现新版本（如 `1.0.5`）时，FluxCD 自动：
1. 更新 Git 仓库中的 Deployment YAML
2. 提交一个包含版本变更的 commit
3. 集群通过 Kustomization 自动同步新配置

# 4.ArgoCD vs FluxCD

| 维度 | ArgoCD | FluxCD |
|------|--------|--------|
| **UI** | 丰富的 Web UI + CLI | 主要 CLI，Web UI 较弱 |
| **多租户** | Projects + RBAC | 通过 Kustomize namespace |
| **多集群** | ApplicationSet + 集群管理 | Kustomization + bootstrap |
| **Helm 管理** | 支持但不擅长 | 一流的 HelmRelease CRD |
| **自动镜像更新** | ArgoCD Image Updater（独立组件） | 内置 Image Automation |
| **配置管理** | 原生支持 Kustomize/Helm/Ksonnet | 原生支持 Kustomize/Helm |
| **安装复杂度** | 中 | 低（bootstrap 一条命令） |
| **通知** | 内置 Webhook/Websocket | Notification Controller |
| **哲学** | 一站式平台 | 模块化组件 |

### 选择建议

```
选择 ArgoCD：
├── 需要完善的 Web UI
├── 有严格的多租户隔离需求
├── 团队熟悉 Kubernetes 原生 CRD 管理
└── 需要 ApplicationSet 管理多环境/多集群

选择 FluxCD：
├── 偏好 CLI 操作和基础设施即代码
├── 深度使用 Helm（需要 HelmRelease 的高级能力）
├── 需要自动镜像版本更新（内置功能）
└── 希望最小化集群内安装的组件数量
```

# 5.生产实践

## 5.1 仓库组织策略

### Monorepo（单仓）

```
fleet-infra/
└── clusters/
    ├── production/
    └── staging/
```

多个集群/环境共享一个配置仓库。适合中小规模，配置复用方便。

### Repo-per-App（每应用一仓）

```
my-app/
├── .github/workflows/ci.yaml    # CI Pipeline
├── src/                         # 应用源代码
└── k8s/
    ├── base/
    └── overlays/
        ├── staging/
        └── production/
```

每个应用维护自己的 K8s 配置。适合微服务多团队协作，但管理多个仓库复杂。

### 推荐：混合模式

```
# 应用仓库（由开发团队管理）
my-app/
├── src/
├── Dockerfile
└── k8s/
    ├── base/
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── kustomization.yaml
    └── overlays/
        └── staging/
            └── kustomization.yaml  # 不包含 prod

# 基础设施仓库（由平台团队管理）
fleet-infra/
├── clusters/
│   └── production/
│       ├── apps/
│       │   └── my-app.yaml         # Kustomization 指向应用仓库
│       └── infrastructure/
└── overlays/
    └── production/
        └── my-app/
            └── kustomization.yaml  # 生产环境特定配置
```

## 5.2 密钥管理

GitOps 的核心挑战是：敏感信息不能存入 Git（即使加密也有风险）。推荐的方案：

| 方案 | 说明 |
|------|------|
| **Sealed Secrets** | 允许将加密的 Secret 提交到 Git，集群内解密 |
| **External Secrets Operator** | 从外部 KMS（AWS Secrets Manager/Vault）同步 Secret |
| **SOPS + Age** | 使用 OpenPGP/Age 加密 Secret 文件，提交到 Git |
| **Vault Sidecar** | Vault Agent 作为 Sidecar 注入 Secret |

```yaml
# External Secrets Operator 示例
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: db-secret
  data:
  - secretKey: password
    remoteRef:
      key: prod/database
      property: password
```

## 5.3 渐进式交付

结合 GitOps 实现金丝雀发布：

```yaml
# Argo Rollouts：Argo 生态的渐进式交付组件
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
spec:
  replicas: 5
  strategy:
    canary:
      steps:
      - setWeight: 20           # 20% 流量到金丝雀
      - pause: {duration: 5m}   # 等待 5 分钟观察
      - setWeight: 50
      - pause: {duration: 5m}
      - setWeight: 100
  template:
    spec:
      containers:
      - name: app
        image: myapp:v2
```

```yaml
# Flagger：与 FluxCD 集成
# 自动进行金丝雀发布分析
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: my-app
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  service:
    port: 80
  analysis:
    interval: 60s
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
    - name: request-duration
      thresholdRange:
        max: 500
    webhooks:
    - name: load-test
      url: http://flagger-loadtester.test/
      timeout: 5s
      metadata:
        cmd: "hey -z 1m -q 10 http://my-app-canary/"
```

## 5.4 灾难恢复

GitOps 的灾难恢复能力是其最大优势之一：

```
灾难场景：整个集群被删除

恢复步骤：
1. 创建新的 Kubernetes 集群
2. 运行 gitops bootstrap（ArgoCD/FluxCD 一键安装）
3. 等待自动化同步完成

结果：所有应用恢复到灾前状态，无需人工介入
只需：集群 API Endpoint + bootstrap 命令 = 完全恢复
```
