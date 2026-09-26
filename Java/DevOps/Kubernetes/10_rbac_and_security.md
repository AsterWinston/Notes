# 1.Kubernetes 安全体系概览

## 1.1 安全模型的四个层面

Kubernetes 的安全模型遵循"纵深防御"（Defense in Depth）原则，从外到内分为四个层面：

```
┌─────────────────────────────────────────────────────────────┐
│  层面 1: 网络边界安全                                         │
│  - TLS 加密通信 (所有组件间)                                    │
│  - NetworkPolicy (Pod 间网络隔离)                              │
│  - Ingress/Egress 控制                                       │
├─────────────────────────────────────────────────────────────┤
│  层面 2: API 访问安全 (本章核心)                                │
│  - 认证 (Authentication): 确认"你是谁"                        │
│  - 授权 (Authorization/RBAC): 决定"你能做什么"                 │
│  - 准入控制 (Admission Control): 验证和修改请求                 │
├─────────────────────────────────────────────────────────────┤
│  层面 3: Pod 安全                                             │
│  - SecurityContext (运行用户、权限、只读文件系统)                 │
│  - Pod Security Standards (PSS)                              │
│  - Seccomp / AppArmor / SELinux                              │
├─────────────────────────────────────────────────────────────┤
│  层面 4: 数据安全                                             │
│  - Secret 加密存储                                            │
│  - etcd 静态加密 (Encryption at Rest)                         │
│  - 镜像签名与验证                                              │
└─────────────────────────────────────────────────────────────┘
```

### API 请求的完整处理链

每个 API 请求在到达 etcd 之前，依次经过三个安全检查：

```
kubectl / client-go 请求
        │
        ▼
┌───────────────────┐
│  1. Authentication│  验证身份
│     (认证)         │  "我是谁？"
│                   │  X509证书 / Bearer Token / OIDC / Webhook
└───────┬───────────┘
        │ 请求携带用户身份
        ▼
┌───────────────────┐
│  2. Authorization │  检查权限
│     (授权/RBAC)    │  "我能做这个操作吗？"
│                   │  Role/ClusterRole + RoleBinding/ClusterRoleBinding
└───────┬───────────┘
        │ 权限检查通过
        ▼
┌───────────────────┐
│  3. Admission      │  修改和验证
│     Control        │  Mutating Webhook → Schema Validation → Validating Webhook
│     (准入控制)      │
└───────┬───────────┘
        │ 准入通过
        ▼
    etcd (持久化)
```

## 1.2 为什么 RBAC 是 Kubernetes 授权的事实标准

Kubernetes 早期支持多种授权模式：ABAC（基于属性的访问控制）、RBAC（基于角色的访问控制）、Node Authorizer、Webhook。RBAC 在 v1.8 中 GA 并迅速成为事实标准，原因如下：

1. **最小权限原则（Least Privilege）**：RBAC 天然支持细粒度权限控制，可以为每个用户/ServiceAccount 精确指定可以操作的资源和操作类型
2. **声明式管理**：Role 和 RoleBinding 是标准的 Kubernetes API 对象，支持 kubectl、GitOps 等标准工具
3. **可审计**：权限关系在集群中以对象形式明确存储，可以随时审查"谁有什么权限"
4. **动态调整**：修改 Role 或 RoleBinding 立即生效，无需重启任何组件
5. **多租户友好**：Namespace 级别的 Role 天然支持多租户隔离

# 2.认证 (Authentication)

## 2.1 认证策略概览

Kubernetes 不管理用户，但会验证用户的身份凭证。支持的认证策略：

| 认证方式 | 适用对象 | 说明 |
|---------|---------|------|
| X509 客户端证书 | 集群组件（kubelet、scheduler 等） | 双向 TLS 认证，最安全 |
| Bearer Token | 用户、ServiceAccount | HTTP Authorization: Bearer <token> |
| Bootstrap Token | 新节点加入集群 | 临时令牌，用于 TLS Bootstrap |
| ServiceAccount Token | Pod 内应用 | 自动挂载到 Pod 中 |
| OpenID Connect (OIDC) | 企业用户 | 对接 LDAP/AD/OAuth2 等身份系统 |
| Webhook Token | 自定义认证 | 外部认证服务 |
| Authenticating Proxy | 已有代理认证 | 如通过 Nginx 反向代理统一认证 |

## 2.2 ServiceAccount 深度解析

ServiceAccount 是 Kubernetes 中给**应用程序（Pod）**使用的身份，而非人类用户。

### ServiceAccount、Token、Secret 的关系

```
Namespace: default
  │
  ├── ServiceAccount: default (自动创建)
  │     ├── Secret: default-token-xxxxx (自动创建的 Token Secret, 1.22-)
  │     │   ├── ca.crt (集群 CA 证书)
  │     │   ├── namespace (命名空间)
  │     │   └── token (JWT Token)
  │     │
  │     └── 所有 Pod 默认使用此 ServiceAccount
  │
  ├── ServiceAccount: my-app (手动创建)
  │     ├── Secret: my-app-token-xxxxx
  │     └── 指定 serviceAccountName: my-app 的 Pod 使用
  │
  └── ServiceAccount: ci-deployer (CI/CD 使用)
```

### ServiceAccount 的 Token 演进

Kubernetes 的 ServiceAccount Token 经历了几次重要变化：

**阶段一（K8s 1.21 前）**：
- 创建 ServiceAccount 时自动创建一个永不过期的 Secret 存储 Token
- Pod 通过挂载 Secret 获得 Token
- 问题：Token 永不过期，泄露后风险极高

**阶段二（K8s 1.21-1.23）**：
- 引入 `TokenRequest` API 和 `BoundServiceAccountToken` 特性门控
- 推荐使用 projected volume 生成时效性 Token
- 旧的自动 Secret 仍然创建

**阶段三（K8s 1.24+）**：
- 默认不再自动创建 ServiceAccount Token Secret
- 推荐使用 `TokenRequest` API 获取有时效的 Token
- 使用 `projected` 卷注入 Token

```yaml
# 推荐的 Token 注入方式（K8s 1.21+）
apiVersion: v1
kind: Pod
metadata:
  name: sa-demo
spec:
  serviceAccountName: my-app
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: token
      mountPath: /var/run/secrets/tokens
  volumes:
  - name: token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 7200       # 2小时过期
          audience: api                  # 目标受众
```

### 手动创建 ServiceAccount 和绑定

```bash
# 创建 ServiceAccount
kubectl create serviceaccount my-app -n production

# 创建带有时效的 Token（v1.24+）
kubectl create token my-app -n production --duration=1h

# 手动创建 ServiceAccount Token Secret（v1.24+，如需长期 Token）
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: my-app-token
  namespace: production
  annotations:
    kubernetes.io/service-account.name: my-app
type: kubernetes.io/service-account-token
EOF
```

### Pod 使用 ServiceAccount

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  serviceAccountName: my-app           # 指定 SA
  automountServiceAccountToken: true   # 是否自动挂载 Token（默认 true）
  containers:
  - name: app
    image: nginx
```

Pod 内部访问 API Server 的方式：
```bash
# 容器内自动挂载的路径
ls /var/run/secrets/kubernetes.io/serviceaccount/
# ca.crt  namespace  token

# 使用 token 访问 API Server
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl -k -H "Authorization: Bearer $TOKEN" https://kubernetes.default.svc/api/v1/namespaces/default/pods
```

# 3.RBAC 授权体系

## 3.1 RBAC 的四大核心资源

```
Role                    ClusterRole
(Namespace 级)           (Cluster 级)
     │                       │
     │ 绑定                  │ 绑定
     ▼                       ▼
RoleBinding             ClusterRoleBinding
(授予 NS 内权限)          (授予集群级权限)
     │                       │
     │ 授予                  │ 授予
     ▼                       ▼
Subject (主体)
  - User (人类用户)
  - Group (用户组)
  - ServiceAccount (应用身份)
```

### 四者关系

| 资源 | 作用域 | 说明 |
|------|--------|------|
| **Role** | Namespace | 定义在某个 Namespace 内的一组权限规则 |
| **ClusterRole** | Cluster | 定义集群级别的一组权限规则（也可用于 Namespace 级别授权） |
| **RoleBinding** | Namespace | 将 Role 绑定到 Subject，授权在特定 Namespace 内生效 |
| **ClusterRoleBinding** | Cluster | 将 ClusterRole 绑定到 Subject，授权在整个集群生效 |

## 3.2 Role 和 ClusterRole 详解

### Role：命名空间级权限集合

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
# 单条规则：允许 get、watch、list Pod
- apiGroups: [""]                   # 核心 API 组为空字符串
  resources: ["pods"]
  verbs: ["get", "watch", "list"]

# 允许读取 Pod 日志
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get", "list"]

# 允许读取 Service
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "watch", "list"]

# 允许读取 ConfigMap
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]

# 允许更新 Deployment (apps 组)
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "update", "patch"]

# 允许在所有 apiGroups 中操作某个资源（不推荐）
- apiGroups: ["*"]
  resources: ["events"]
  verbs: ["get", "list"]
```

**资源名称的白名单限制**：可以使用 `resourceNames` 限制只能操作特定名称的资源：

```yaml
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["app-config", "db-config"]   # 只能操作这两个 ConfigMap
  verbs: ["get", "update"]
```

### ClusterRole：集群级权限集合

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
# 读取节点信息（集群级资源）
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]

# 读取 PersistentVolume（集群级资源）
- apiGroups: [""]
  resources: ["persistentvolumes"]
  verbs: ["get", "list"]

# 读取所有 Namespace 的 Pod
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]

# 管理 RBAC 资源（集群级资源）
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["clusterroles", "clusterrolebindings"]
  verbs: ["get", "list"]
```

### 聚合 ClusterRole（Aggregated ClusterRole）

Kubernetes 支持通过标签聚合多个 ClusterRole 规则到一个 ClusterRole：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: view
  labels:
    rbac.authorization.k8s.io/aggregate-to-view: "true"
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch"]
```

所有带有 `rbac.authorization.k8s.io/aggregate-to-view` 标签的 ClusterRole 的规则会被自动合并到名为 `view` 的 ClusterRole 中。Kubernetes 内置的 `view`、`edit`、`admin` ClusterRole 都使用此机制。

### 内置的 ClusterRole

| 内置 ClusterRole | 权限范围 |
|-----------------|---------|
| `cluster-admin` | 超级管理员，拥有所有权限 |
| `admin` | Namespace 内管理员（需配合 RoleBinding 使用） |
| `edit` | 可以修改 Namespace 内大部分资源，不能操作 RBAC |
| `view` | 只读，可以查看 Namespace 内大部分资源 |

```bash
# 查看内置 ClusterRole
kubectl get clusterroles
kubectl describe clusterrole admin
```

## 3.3 verbs（操作动词）详解

RBAC 的 `verbs` 对应 HTTP 方法：

| Verb | HTTP 方法 | 说明 |
|------|-----------|------|
| `get` | GET | 获取单个资源 |
| `list` | GET | 获取资源列表 |
| `watch` | GET | Watch 资源变更 |
| `create` | POST | 创建资源 |
| `update` | PUT | 替换资源 |
| `patch` | PATCH | 部分更新资源 |
| `delete` | DELETE | 删除单个资源 |
| `deletecollection` | DELETE | 删除资源列表 |
| `*` | ALL | 所有操作 |
| `escalate` | POST | 创建/更新包含更高权限的 Role（仅 RBAC 资源） |
| `bind` | POST | 创建/更新 RoleBinding/ClusterRoleBinding |

**get / list / watch 的区别**：
- `get`：客户端指定要查询的资源名称
- `list`：客户端不指定名称，获取整个集合
- `watch`：通过长连接监听资源变更事件

注意：给用户 `create` 权限但不同时给 `get`/`list`，用户创建资源后可能无法查看。

# 4.RoleBinding 与 ClusterRoleBinding

## 4.1 RoleBinding：命名空间级绑定

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: production
subjects:
# 绑定给 ServiceAccount
- kind: ServiceAccount
  name: my-app
  namespace: production
# 绑定给用户
- kind: User
  name: alice
  apiGroup: rbac.authorization.k8s.io
# 绑定给用户组
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role                    # 也可以是 ClusterRole
  name: pod-reader
```

**重要特性**：RoleBinding 可以引用 ClusterRole，将 ClusterRole 中的权限限制在特定命名空间内生效。例如，可以将集群级的 `view` ClusterRole 绑定到某个 Namespace，让用户在仅限于该 Namespace 拥有只读权限。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: view-binding
  namespace: team-a
subjects:
- kind: User
  name: bob
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole              # 引用了 ClusterRole
  name: view                     # 但权限仅限于 team-a Namespace
```

## 4.2 ClusterRoleBinding：集群级绑定

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-reader-binding
subjects:
- kind: Group
  name: ops-team
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: node-reader
```

ClusterRoleBinding 只能引用 ClusterRole，不能引用 Role。

## 4.3 Subject（主体）详解

Subject 是权限的授予对象，可以是以下类型：

```yaml
subjects:
# ServiceAccount（最常用）
- kind: ServiceAccount
  name: my-app
  namespace: production         # ServiceAccount 必须指定 Namespace

# 用户（通过 kubectl 的 context 指定）
- kind: User
  name: alice

# 用户组
- kind: Group
  name: developers
```

用户身份从哪里来：
- **X509 证书中的 CN**：`CN=alice` → 用户名为 `alice`
- **X509 证书中的 O**：`O=developers` → 用户组为 `developers`
- **Bearer Token**：通过 TokenReview API 验证身份
- **OIDC Token**：通过 JWT Claim 解析用户名和组

```bash
# 生成用户证书（实际中由管理员或 CA 签发）
openssl genrsa -out alice.key 2048
openssl req -new -key alice.key -out alice.csr -subj "/CN=alice/O=developers"
openssl x509 -req -in alice.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out alice.crt -days 365

# 配置 kubectl 使用用户证书
kubectl config set-credentials alice --client-certificate=alice.crt --client-key=alice.key
kubectl config set-context alice-context --cluster=my-cluster --user=alice
```

## 4.4 权限检查的底层原理

当 API Server 收到请求并完成认证后，Authorizer 链中的 RBAC Authorizer 执行权限检查：

```
请求: GET /api/v1/namespaces/production/pods/nginx-pod
认证用户: system:serviceaccount:production:my-app

RBAC 权限检查:
1. 查找所有引用了 my-app ServiceAccount 的 RoleBinding/ClusterRoleBinding
2. 获取对应的 Role/ClusterRole 的 rules
3. 检查是否有一条 rule 同时满足:
   - apiGroups 包含 ""
   - resources 包含 "pods"
   - verbs 包含 "get"
   - (如果是 RoleBinding) 请求的 namespace 匹配 RoleBinding 的 namespace
4. 通过 → 200 OK / 不通过 → 403 Forbidden
```

# 5.实战：多场景 RBAC 配置

## 5.1 场景一：开发者仅读权限

```yaml
# 允许开发者在指定 Namespace 只读
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-readonly
  namespace: dev
subjects:
- kind: Group
  name: dev-team
roleRef:
  kind: ClusterRole
  name: view    # 使用内置的 view ClusterRole
```

## 5.2 场景二：CI/CD Pipeline 部署权限

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: deployer
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer-role
  namespace: production
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list", "create", "update"]
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list", "create", "update", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deployer-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: deployer
  namespace: production
roleRef:
  kind: Role
  name: deployer-role
```

## 5.3 场景三：跨命名空间访问

```yaml
# 允许 monitoring 命名空间的 Prometheus 读取所有命名空间的 Pod 指标
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-reader
rules:
- apiGroups: [""]
  resources: ["pods", "services", "endpoints", "nodes"]
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]    # 非资源端点
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus-binding
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: prometheus-reader
```

## 5.4 场景四：Ingress Controller 最小权限

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: ingress-nginx
rules:
- apiGroups: [""]
  resources: ["configmaps", "pods", "secrets", "services", "endpoints"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses", "ingressclasses"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses/status"]
  verbs: ["update"]
- apiGroups: [""]
  resources: ["events"]
  verbs: ["create", "patch"]
- apiGroups: ["discovery.k8s.io"]
  resources: ["endpointslices"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ingress-nginx
subjects:
- kind: ServiceAccount
  name: ingress-nginx
  namespace: ingress-nginx
roleRef:
  kind: ClusterRole
  name: ingress-nginx
```

# 6.准入控制 (Admission Control)

## 6.1 准入控制概述

准入控制是 API 请求处理链的最后一环——在请求通过认证和授权之后、持久化到 etcd 之前，拦截和修改请求。

准入控制分为两种类型：

**Mutating Admission Webhook**（变更型准入 Webhook）：
- 在对象持久化前，可以修改请求中的对象
- 例如：自动注入 Sidecar 容器（Istio）、自动添加标签、修改资源限制
- 按顺序调用，一个修改的结果对后续 Webhook 可见

**Validating Admission Webhook**（验证型准入 Webhook）：
- 在对象持久化前，验证对象的合法性
- 可以拒绝不符合规范的请求
- 例如：禁止使用 `latest` 标签的镜像、强制要求所有 Pod 设置资源限制

```
请求 → Mutating Webhook 1 → Mutating Webhook 2 → Schema Validation → Validating Webhook 1 → Validating Webhook 2 → etcd
```

## 6.2 内置准入控制器

Kubernetes 内置了多个准入控制器，默认启用的重要控制器包括：

| 准入控制器 | 作用 |
|-----------|------|
| `NamespaceLifecycle` | 阻止在正在删除的 Namespace 中创建资源 |
| `LimitRanger` | 为 Pod 设置默认资源请求和限制 |
| `ServiceAccount` | 自动为 Pod 分配 ServiceAccount |
| `PersistentVolumeClaimResize` | 验证 PVC 扩容请求 |
| `DefaultStorageClass` | 设置默认 StorageClass |
| `ResourceQuota` | 验证资源配额 |
| `MutatingAdmissionWebhook` | 调用变更型 Webhook |
| `ValidatingAdmissionWebhook` | 调用验证型 Webhook |
| `NodeRestriction` | 限制 kubelet 只能修改自己节点的资源 |
| `PodSecurity` | Pod Security Standards 强制执行（v1.25+，替代 PodSecurityPolicy） |

```bash
# 查看当前启用的准入控制器
kubectl -n kube-system describe pod kube-apiserver-<node> | grep enable-admission-plugins
```

## 6.3 自定义 Webhook 示例

### Mutating Webhook：自动添加资源限制

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: resource-limits-webhook
webhooks:
- name: resource-limits.example.com
  clientConfig:
    service:
      name: resource-limits-service
      namespace: webhooks
      path: "/mutate"
    caBundle: <CA_BUNDLE_BASE64>
  rules:
  - operations: ["CREATE", "UPDATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  namespaceSelector:
    matchLabels:
      environment: production
  admissionReviewVersions: ["v1"]
  sideEffects: None
  failurePolicy: Fail        # Fail/Ignore
```

### Validating Webhook：禁止使用 latest 镜像

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: image-policy-webhook
webhooks:
- name: image-policy.example.com
  clientConfig:
    service:
      name: image-policy-service
      namespace: webhooks
      path: "/validate"
    caBundle: <CA_BUNDLE_BASE64>
  rules:
  - operations: ["CREATE", "UPDATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  admissionReviewVersions: ["v1"]
  sideEffects: None
  failurePolicy: Fail
```

**failurePolicy** 决定 Webhook 不可用时的行为：
- `Fail`：Webhook 调用失败时拒绝请求（生产推荐，但不能让 Webhook 单点故障）
- `Ignore`：Webhook 调用失败时放行请求（但有安全风险）

## 6.4 常用准入控制工具

| 工具 | 类型 | 说明 |
|------|------|------|
| **OPA/Gatekeeper** | Validating | 基于 Rego 策略语言，CNCF 毕业项目 |
| **Kyverno** | Mutating + Validating | 策略即代码，使用 Kubernetes 原生 YAML 语法 |
| **Pod Security Admission** | Validating | 内置 Pod 安全标准（v1.25+ GA） |
| **KubeLinter** | 静态分析 | 在 CI/CD 中检查 YAML 安全配置 |

# 7.Pod 安全标准 (Pod Security Standards)

Kubernetes 1.25+ 使用 Pod Security Admission (PSA) 替代了已废弃的 PodSecurityPolicy (PSP)。

## 7.1 三种安全策略级别

| 级别 | 描述 | 适用场景 |
|------|------|---------|
| **Privileged** | 无限制，完全特权 | 系统组件（kube-system）、CI/CD |
| **Baseline** | 最小限制策略，防止已知的权限提升 | 普通应用（默认推荐） |
| **Restricted** | 严格限制，遵循 Pod 加固最佳实践 | 安全敏感应用 |

### Baseline 策略的关键限制

- 禁止 `hostNetwork`、`hostPID`、`hostIPC`
- 禁止 `hostPath` 卷
- 禁止特权容器（`privileged: true`）
- 禁止 `hostPorts`
- 限制 capabilities（不能添加 `NET_RAW` 等危险能力）
- 禁止 `procMount: Unmasked`

### Restricted 策略的额外限制

- 所有 `Baseline` 限制
- 要求 `runAsNonRoot: true`
- 要求 `seccompProfile.type: RuntimeDefault` 或 `Localhost`
- 限制卷类型（只能使用 ConfigMap、Secret、EmptyDir 等安全卷）

## 7.2 使用 Pod Security Admission

```yaml
# Namespace 级别设置安全策略
apiVersion: v1
kind: Namespace
metadata:
  name: secure-app
  labels:
    pod-security.kubernetes.io/enforce: baseline         # 强制执行，违反则拒绝
    pod-security.kubernetes.io/enforce-version: v1.30    # 以哪个 K8s 版本的标准检查
    pod-security.kubernetes.io/audit: restricted         # 审计模式，只记录不拒绝
    pod-security.kubernetes.io/warn: restricted          # 警告模式，返回警告但不拒绝
```

三种模式的行为：
- `enforce`：违反策略的 Pod 将被拒绝创建
- `audit`：违反策略的记录在审计日志中，Pod 仍可创建
- `warn`：违反策略的返回给用户警告信息，Pod 仍可创建

# 8.安全最佳实践

## 8.1 RBAC 最佳实践

1. **最小权限原则**：每个 ServiceAccount 只授予必要的最小权限。不要为了方便使用 `cluster-admin`
2. **每个应用独立的 ServiceAccount**：不要多个应用共享同一个 ServiceAccount
3. **定期审计权限**：使用 `kubectl auth can-i` 或工具审查权限
4. **限制 `escalate` 和 `bind` 权限**：这些权限允许用户提升自己的权限级别
5. **不要给 ServiceAccount 分配 `cluster-admin`**：除非它是集群管理组件
6. **限制 Secret 访问**：读取 Secret 的权限应严格限制

```bash
# 审计命令
# 检查当前用户(或SA)是否有某个权限
kubectl auth can-i create deployments --namespace production
kubectl auth can-i delete secrets --as system:serviceaccount:production:my-app

# 列出某个用户/SA 在某个 Namespace 的所有权限
kubectl auth can-i --list --namespace production --as system:serviceaccount:production:my-app
```

## 8.2 Pod 安全最佳实践

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  # Pod 级别安全设置
  securityContext:
    runAsNonRoot: true               # 必须非 root 运行
    seccompProfile:
      type: RuntimeDefault           # 使用运行时默认 Seccomp
  containers:
  - name: app
    image: myapp:v1.0
    securityContext:
      allowPrivilegeEscalation: false # 禁止权限提升
      runAsNonRoot: true
      privileged: false
      readOnlyRootFilesystem: true    # 只读根文件系统
      capabilities:
        drop:
        - ALL                         # 删除所有 capabilities
        add:
        - NET_BIND_SERVICE            # 只添加必需的
```

## 8.3 Secret 管理最佳实践

1. **使用 External Secrets Operator** 或 **Vault**：与外部密钥管理系统集成，避免手动管理 Secret
2. **启用 etcd 静态加密**：`--encryption-provider-config` 对 Secret 加密存储
3. **使用 Sealed Secrets**：允许将加密的 Secret 提交到 Git
4. **限制 Secret RBAC**：严格控制谁可以读取 Secret
5. **短生命周期 Token**：使用 projected ServiceAccountToken 替代默认的永久 Token
