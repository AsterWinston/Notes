# 1.Helm 概述

## 1.1 为什么需要 Helm

在没有 Helm 的情况下，部署一个典型的应用需要管理多个 Kubernetes YAML 文件：Deployment、Service、ConfigMap、Secret、Ingress、PVC 等等。这些文件之间存在大量重复的配置信息（应用名称、镜像版本、端口号等），在不同环境（dev/staging/prod）部署时需要手动修改这些值，既繁琐又容易出错。

Helm 被称为"Kubernetes 的包管理器"，解决了以下核心问题：

**模板化部署**：不再维护几十个几乎相同的 YAML 文件，而是维护一个模板 + 一个 values 文件。环境差异通过不同的 values 文件覆盖。

**版本管理**：Helm 维护了每次部署的版本记录（Release），支持一键回滚到任意历史版本。

**依赖管理**：一个应用可能依赖于 MySQL、Redis、Kafka 等中间件。Helm 的 Chart 依赖机制允许你声明这些依赖并自动安装。

**分享与复用**：Helm Chart 可以通过 Helm Repository 或 OCI Registry 分享，社区维护了数千个 Chart 供直接使用。

## 1.2 Helm 的核心概念

```
Chart（包）
  └── 一个包含 Kubernetes 资源模板的压缩包
      类似于 apt 的 .deb 或 yum 的 .rpm

Repository（仓库）
  └── 存储和分享 Chart 的地方
      类似于 Docker Registry 或 Maven Central

Release（发布）
  └── Chart 在集群中的一次安装实例
      同一个 Chart 可以安装多次，每次都是一个独立的 Release

Revision（版本）
  └── Release 的每次升级产生一个新的 Revision
      用于回滚和历史记录
```

### Helm 架构（v3 简化）

```
┌─────────────┐
│  helm CLI   │
└──────┬──────┘
       │ Kubernetes API
       ▼
┌─────────────┐
│ API Server  │
└─────────────┘
       │
       ▼
    etcd (Release 信息以 Secret 存储)
```

Helm v3 相比 v2 最大的变化是**移除了 Tiller**——不再需要在集群中部署服务端组件，helm CLI 直接通过 kubeconfig 与 API Server 交互，使用与 kubectl 相同的认证和授权机制。

# 2.Helm Chart 结构

## 2.1 脚手架创建

```bash
helm create my-chart

# 生成的目录结构
my-chart/
├── Chart.yaml            # Chart 的元数据
├── values.yaml           # 默认配置值
├── values.schema.json    # values 的 JSON Schema 验证（可选）
├── charts/               # 子 Chart 依赖
├── templates/            # Kubernetes 资源模板
│   ├── NOTES.txt         # 安装后的帮助提示
│   ├── _helpers.tpl      # 模板辅助函数
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── serviceaccount.yaml
│   └── tests/            # 测试资源
│       └── test-connection.yaml
├── .helmignore           # 打包时忽略的文件
├── crds/                 # CRD 定义（无模板）
└── README.md
```

## 2.2 Chart.yaml 详解

```yaml
apiVersion: v2                    # Chart API 版本（v1 已废弃）
name: my-app                      # Chart 名称
description: My application       # 描述
type: application                 # application 或 library（库 Chart）
version: 1.2.3                    # Chart 版本（SemVer）
appVersion: "2.0.0"               # 应用版本
kubeVersion: ">=1.25.0"           # 兼容的 Kubernetes 版本
dependencies:                     # 依赖的其他 Chart
  - name: mysql
    version: "9.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: mysql.enabled      # 条件导入
    tags:                         # 标签控制
      - database
  - name: redis
    version: "17.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
maintainers:                      # 维护者信息
  - name: Team Platform
    email: platform@example.com
icon: https://example.com/icon.png
keywords:
  - web
  - api
annotations:                      # 自定义注解
  category: Application
```

## 2.3 values.yaml 的角色

`values.yaml` 是 Chart 中最重要的文件——它定义了模板中所有变量的默认值。用户通过 `--set` 或 `-f` 覆盖这些值。

```yaml
# values.yaml - 默认配置
replicaCount: 3

image:
  repository: nginx
  tag: "1.25"
  pullPolicy: IfNotPresent

nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  name: ""

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: false
  className: "nginx"
  hosts:
    - host: myapp.local
      paths:
        - path: /
          pathType: Prefix
  tls: []

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

nodeSelector: {}
tolerations: []
affinity: {}

# 环境相关配置
config:
  logLevel: "info"
  maxConnections: "1000"

secrets:
  dbPassword: ""
```

## 2.4 templates/ 模板语法

Helm 使用 Go Template 语法，并扩展了大量内置函数：

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-chart.fullname" . }}
  labels:
    {{- include "my-chart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-chart.labels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "my-chart.serviceAccountName" . }}
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: {{ .Values.service.targetPort }}
        env:
        - name: LOG_LEVEL
          value: {{ .Values.config.logLevel | quote }}
        {{- with .Values.resources }}
        resources:
          {{- toYaml . | nindent 10 }}
        {{- end }}
```

### 关键模板语法

| 语法 | 说明 |
|------|------|
| `{{ .Values.key }}` | 引用 values.yaml 中的值 |
| `{{ .Release.Name }}` | Release 名称 |
| `{{ .Chart.Name }}` | Chart 名称 |
| `{{ .Release.Namespace }}` | Release 所在的 Namespace |
| `{{ include "template" . }}` | 调用命名模板 |
| `{{-` / `-}}` | 去除前后空白 |
| `\| quote` | 添加引号 |
| `\| nindent N` | 添加 N 个空格缩进 + 换行 |
| `\| indent N` | 添加 N 个空格缩进（无换行） |
| `{{- with ... }}...{{- end }}` | 条件块 |
| `{{- if ... }}...{{- end }}` | 条件判断 |
| `{{- range ... }}...{{- end }}` | 循环遍历 |

### _helpers.tpl 中的命名模板

```yaml
# templates/_helpers.tpl
{{- define "my-chart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{- define "my-chart.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end -}}

{{- define "my-chart.labels" -}}
helm.sh/chart: {{ include "my-chart.chart" . }}
{{ include "my-chart.selectorLabels" . }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end -}}
```

## 2.5 条件渲染与流程控制

```yaml
# 条件渲染
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
# ...
{{- end }}

# range 循环
{{- range .Values.ingress.hosts }}
- host: {{ .host }}
  http:
    paths:
    {{- range .paths }}
    - path: {{ .path }}
      pathType: {{ .pathType }}
      backend:
        service:
          name: {{ $.Values.service.name }}
          port:
            number: {{ $.Values.service.port }}
    {{- end }}
{{- end }}

# with 块（改变作用域）
{{- with .Values.nodeSelector }}
nodeSelector:
  {{- toYaml . | nindent 8 }}
{{- end }}

# 变量赋值
{{- $fullName := include "my-chart.fullname" . -}}

# 默认值
{{ .Values.image.pullPolicy | default "IfNotPresent" }}
```

# 3.安装和管理 Release

## 3.1 基础命令

```bash
# 安装 Chart
helm install my-release ./my-chart
helm install my-release ./my-chart -n production --create-namespace

# 使用自定义 values
helm install my-release ./my-chart -f custom-values.yaml
helm install my-release ./my-chart --set image.tag=1.26 --set replicaCount=5

# 查看 Release
helm list -n production
helm list -A                    # 所有命名空间
helm list --all                 # 包括已删除的 Release

# 查看 Release 状态和详情
helm status my-release -n production
helm get values my-release      # 查看当前生效的 values
helm get manifest my-release    # 查看生成的 Kubernetes 清单
helm get notes my-release       # 查看 NOTES.txt

# 查看 Release 历史
helm history my-release -n production

# 升级 Release
helm upgrade my-release ./my-chart -f new-values.yaml

# 回滚 Release
helm rollback my-release 2      # 回滚到 revision 2
helm rollback my-release        # 回滚到上一版本

# 卸载 Release
helm uninstall my-release -n production
helm uninstall my-release --keep-history  # 保留历史记录
```

## 3.2 安装干运行与调试

```bash
# 干运行：渲染模板但不安装
helm install my-release ./my-chart --dry-run

# 查看最终渲染的 YAML
helm template my-release ./my-chart -f values-prod.yaml

# 调试模式
helm install my-release ./my-chart --debug --dry-run

# 只渲染特定模板
helm template my-release ./my-chart -s templates/deployment.yaml
```

## 3.3 Values 的优先级覆盖

Values 有多个来源，按优先级从低到高：

```
1. Chart 内置的 values.yaml（最低优先级）
2. 父 Chart 的 values.yaml（如果是子 Chart）
3. 用户通过 -f 传入的 values 文件
4. 用户通过 --set 传入的键值对（最高优先级）
```

```bash
# 多层 values 覆盖
helm install my-release ./my-chart \
  -f values-base.yaml \       # 基础配置
  -f values-prod.yaml \       # 生产环境覆盖
  --set image.tag=1.26.0 \    # 最高优先级
  --set config.logLevel=debug
```

# 4.Chart 仓库与依赖

## 4.1 Helm Repository

```bash
# 添加仓库
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# 查看仓库列表
helm repo list

# 更新仓库索引
helm repo update

# 搜索 Chart
helm search repo nginx
helm search repo bitnami/nginx

# 安装仓库中的 Chart
helm install my-nginx bitnami/nginx --version 15.0.0
```

## 4.2 Chart 依赖管理

```yaml
# Chart.yaml 中声明依赖
dependencies:
- name: mysql
  version: "9.x.x"
  repository: "https://charts.bitnami.com/bitnami"
  condition: mysql.enabled
  tags:
    - database
- name: redis
  version: "17.x.x"
  repository: "https://charts.bitnami.com/bitnami"
  condition: redis.enabled
```

```bash
# 下载依赖 Chart 到 charts/ 目录
helm dependency update ./my-chart

# 列出依赖
helm dependency list ./my-chart

# 打包 Chart（包含依赖）
helm dependency update ./my-chart
helm package ./my-chart
```

## 4.3 子 Chart 的 Values 覆盖

```yaml
# 父 Chart 的 values.yaml 中覆盖子 Chart 的值
mysql:
  enabled: true
  auth:
    rootPassword: "MyR00tP@ss"
    database: myapp
    username: myapp
    password: secret

redis:
  enabled: false
  architecture: standalone
  auth:
    password: redis-secret
```

# 5.Helm Hooks

## 5.1 生命周期钩子

Helm 支持在 Release 生命周期的特定阶段执行操作：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    "helm.sh/hook": pre-upgrade    # 钩子类型
    "helm.sh/hook-delete-policy": before-hook-creation  # 清理策略
    "helm.sh/hook-weight": "5"     # 执行权重（从小到大）
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: migration
        image: myapp-migration:v1.0
```

### 钩子类型

| Hook | 触发时机 |
|------|---------|
| `pre-install` | 渲染模板之后、创建任何资源之前 |
| `post-install` | 所有资源创建之后 |
| `pre-delete` | 删除请求之后、删除任何资源之前 |
| `post-delete` | 所有资源删除之后 |
| `pre-upgrade` | 渲染模板之后、更新任何资源之前 |
| `post-upgrade` | 所有资源更新之后 |
| `pre-rollback` | 渲染模板之后、回滚任何资源之前 |
| `post-rollback` | 所有资源回滚之后 |
| `test` | `helm test` 命令执行时 |

### 删除策略

| 策略 | 行为 |
|------|------|
| `before-hook-creation` | 执行新钩子前删除旧钩子资源（默认） |
| `hook-succeeded` | 钩子成功后删除 |
| `hook-failed` | 钩子失败后删除 |

# 6.Helm 测试

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "my-chart.fullname" . }}-test-connection"
  labels:
    {{- include "my-chart.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
spec:
  containers:
  - name: wget
    image: busybox
    command: ['wget']
    args: [
      '{{ include "my-chart.fullname" . }}:{{ .Values.service.port }}'
    ]
  restartPolicy: Never
```

```bash
# 运行 Chart 测试
helm test my-release -n production

# 查看测试日志
kubectl logs my-release-test-connection -n production
```

# 7.Library Chart（库 Chart）

Library Chart 不包含实际资源模板，只提供可复用的命名模板：

```yaml
# Chart.yaml
apiVersion: v2
name: my-library
type: library
version: 1.0.0
```

```yaml
# templates/_labels.tpl
{{- define "my-library.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}
```

其他 Chart 可以通过依赖引用 Library Chart：

```yaml
# Chart.yaml
dependencies:
- name: my-library
  version: 1.0.0
  repository: file://../my-library
```

# 8.OCI Registry 支持

Helm v3.8+ 支持将 Chart 推送到 OCI 兼容的容器镜像仓库（如 Docker Hub、Harbor、AWS ECR）：

```bash
# 登录 OCI Registry
helm registry login harbor.example.com -u admin -p password

# 打包 Chart
helm package ./my-chart

# 推送到 OCI Registry
helm push my-chart-1.2.3.tgz oci://harbor.example.com/charts

# 从 OCI Registry 拉取
helm pull oci://harbor.example.com/charts/my-chart --version 1.2.3

# 直接从 OCI Registry 安装
helm install my-release oci://harbor.example.com/charts/my-chart --version 1.2.3
```

# 9.生产实践

## 9.1 Chart 设计最佳实践

1. **一个 Chart 一个应用**：不要在一个 Chart 中打包多个不相关的应用
2. **提供合理的默认值**：`values.yaml` 中的默认值应该能直接用于开发环境
3. **使用 _helpers.tpl**：提取重复的模板逻辑到命名模板中
4. **添加 NOTES.txt**：安装成功后输出有用的连接信息和使用说明
5. **为关键配置添加注释**：`values.yaml` 中的注释能极大帮助用户理解配置
6. **使用 values.schema.json**：为 values 提供 JSON Schema 验证，避免配置错误
7. **锁定依赖版本**：使用精确版本号（`1.2.3`）而非范围（`>=1.0.0`）

## 9.2 多环境管理策略

```bash
# 目录结构
my-app/
├── Chart.yaml
├── values.yaml              # 默认开发环境配置
├── values-staging.yaml      # 预发布环境覆盖
├── values-prod.yaml          # 生产环境覆盖
└── templates/
    └── ...

# 部署到不同环境
helm install my-app ./my-app -f values-staging.yaml -n staging
helm install my-app ./my-app -f values-prod.yaml -n production
```

## 9.3 安全建议

1. **不要在 values 中硬编码密码**：使用 `--set` 或外部 Secret 管理工具
2. **使用 `helm lint`**：在 CI/CD 中检查 Chart 的正确性
3. **Chart 签名验证**：使用 `helm verify` 验证 Chart 的来源
4. **RBAC 限制**：限制 Helm Release 的 ServiceAccount 权限

```bash
# Chart 检查
helm lint ./my-chart

# 安全扫描（使用第三方工具）
trivy config ./my-chart
```
