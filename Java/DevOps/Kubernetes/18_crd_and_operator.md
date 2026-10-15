# 1. CRD 与 Operator 概述

## 1.1 Kubernetes 的扩展性哲学

Kubernetes 的核心设计理念之一是"平台平台"（Platform for Platforms）——它不仅是一个容器编排工具，更是一个构建分布式平台的基础框架。这种可扩展性通过以下机制实现：

```
扩展层级 1: 内置资源
  Pod, Service, Deployment, ConfigMap, ...
  → 大多数应用足够

扩展层级 2: 聚合 API (API Aggregation)
  metrics.k8s.io, custom.metrics.k8s.io
  → 扩展 API Server，为现有 API 增加子资源

扩展层级 3: CRD (Custom Resource Definition)
  PostgresCluster, KafkaTopic, Certificate, ...
  → 定义自己的 API 资源类型

扩展层级 4: Operator 模式
  CRD + 自定义控制器 = Operator
  → 用代码自动化管理应用的完整生命周期
```

## 1.2 CRD 是什么

CRD（Custom Resource Definition）允许你在 Kubernetes 中定义自己的 API 资源类型。一旦定义了 CRD，你就可以像使用 Deployment、Service 一样使用自定义资源。

```yaml
# 定义 CRD
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.example.com       # name = <plural>.<group>
spec:
  group: example.com                # API 组
  names:
    kind: Database                  # CamelCase 单数
    listKind: DatabaseList
    plural: databases               # 小写复数（URL 中使用）
    singular: database              # 小写单数
    shortNames:                     # 缩写
    - db
  scope: Namespaced                 # Namespaced 或 Cluster
  versions:
  - name: v1
    served: true
    storage: true                   # 只有一个版本可以是 storage
    schema:
      openAPIV3Schema:              # OpenAPI v3 Schema 验证
        type: object
        properties:
          spec:
            type: object
            properties:
              engine:
                type: string
                enum: ["mysql", "postgresql", "mongodb"]
              version:
                type: string
                pattern: '^\d+\.\d+$'
              replicas:
                type: integer
                minimum: 1
                maximum: 10
              storage:
                type: string
            required: ["engine", "version"]
          status:
            type: object
            properties:
              phase:
                type: string
                enum: ["Pending", "Creating", "Running", "Failed"]
              endpoint:
                type: string
    subresources:
      status: {}                    # 启用 /status 子资源
      scale:                        # 启用 /scale 子资源（可选）
        specReplicasPath: .spec.replicas
        statusReplicasPath: .status.replicas
    additionalPrinterColumns:       # kubectl get 时的额外列
    - name: Engine
      type: string
      jsonPath: .spec.engine
    - name: Version
      type: string
      jsonPath: .spec.version
    - name: Status
      type: string
      jsonPath: .status.phase
```

### 1.2.1 安装 CRD 后

```bash
# 创建自定义资源
kubectl apply -f - <<EOF
apiVersion: example.com/v1
kind: Database
metadata:
  name: my-postgres
spec:
  engine: postgresql
  version: "15.0"
  replicas: 3
  storage: 100Gi
EOF

# 使用 kubectl 操作
kubectl get databases
# NAME          ENGINE      VERSION   STATUS
# my-postgres   postgresql  15.0      Pending

kubectl describe database my-postgres
kubectl get database my-postgres -o yaml
```

## 1.3 Operator 是什么

**Operator = CRD + 自定义控制器 + 运维知识代码化**

Operator 模式由 CoreOS 在 2016 年提出，核心思想是：**让软件自己管理自己**。它将人类运维专家的操作知识（安装、配置、升级、备份、故障恢复）编码为控制器逻辑，实现自动化运维。

```
传统模式（人类操作）:
  DBA 收到需求 → 手动安装 PostgreSQL → 配置主从复制 → 配置备份
  → 监控运行状态 → 故障时手动恢复 → 手动升级版本

Operator 模式（自动化）:
  用户创建 PostgresCluster CR → Operator 自动安装 PostgreSQL
  → 自动配置复制 → 自动备份 → 自动监控 → 自动故障恢复 → 自动升级
```

### 1.3.1 典型的 Operator 能力阶梯

| 等级 | 能力 | 说明 |
|------|------|------|
| Level 1 | 基本安装 | 自动创建 Deployment、Service、ConfigMap |
| Level 2 | 无缝升级 | 处理版本升级、Schema 迁移 |
| Level 3 | 完整生命周期 | 备份、恢复、故障转移、扩缩容 |
| Level 4 | 深度洞察 | 指标收集、告警、性能分析 |
| Level 5 | 自动导航 | 自动调优、自动扩缩容、自动问题修复 |

# 2. CRD 深入

## 2.1 API 版本管理

CRD 支持多版本共存，不同版本之间可以自动转换：

```yaml
spec:
  versions:
  - name: v1alpha1       # 早期 Alpha 版本
    served: true         # 仍可通过 API 访问
    storage: false       # 不再存储此版本
    schema:
      openAPIV3Schema: {...}
    subresources:
      status: {}
  - name: v1beta1        # Beta 版本
    served: true
    storage: false
    schema:
      openAPIV3Schema: {...}
  - name: v1             # 稳定版本
    served: true
    storage: true        # 存储版本
    schema:
      openAPIV3Schema: {...}
  conversion:            # 版本转换策略
    strategy: Webhook    # None / Webhook
    webhook:
      clientConfig:
        service:
          name: conversion-webhook
          namespace: default
          path: /convert
    conversionReviewVersions: ["v1"]
```

### 2.1.1 版本升级的场景

```
用户创建 v1alpha1 的资源:
kubectl apply -f db-v1alpha1.yaml

API Server:
1. 接收 v1alpha1 对象
2. 调用 Conversion Webhook 转换为 v1 (storage 版本)
3. 以 v1 格式存入 etcd

用户查询 v1beta1 的资源:
kubectl get db my-db -o yaml

API Server:
1. 从 etcd 读取 v1 格式
2. 调用 Conversion Webhook 转换为 v1beta1
3. 返回 v1beta1 格式给用户
```

## 2.2 验证与默认值

### 2.2.1 声明式验证 (Schema)

```yaml
schema:
  openAPIV3Schema:
    type: object
    required: ["spec"]
    properties:
      spec:
        type: object
        required: ["replicas"]
        properties:
          replicas:
            type: integer
            minimum: 1
            maximum: 100
          image:
            type: string
            pattern: '^[a-zA-Z0-9./\-_]+:[a-zA-Z0-9.\-_]+$'
          resources:
            type: object
            properties:
              cpu:
                type: string
                pattern: '^\d+m?$'
              memory:
                type: string
                pattern: '^\d+(Ki|Mi|Gi)$'
          env:
            type: array
            items:
              type: object
              required: ["name"]
              properties:
                name:
                  type: string
                value:
                  type: string
          tolerations:
            type: array
            x-kubernetes-preserve-unknown-fields: true   # 允许未知字段
```

### 2.2.2 验证 Webhook (更灵活的验证)

对于 Schema 无法表达的复杂验证逻辑（如跨字段验证），使用 Validating Webhook：

```go
// 示例：验证 replicas 必须为奇数（某些分布式系统的要求）
func validateDatabase(db *Database) error {
    if db.Spec.Replicas%2 == 0 {
        return fmt.Errorf("replicas must be odd number for quorum-based systems")
    }
    return nil
}
```

## 2.3 自定义列 (Printer Columns)

```yaml
additionalPrinterColumns:
- name: Replicas
  type: integer
  jsonPath: .spec.replicas
- name: Ready
  type: integer
  jsonPath: .status.readyReplicas
- name: Status
  type: string
  jsonPath: .status.phase
- name: Age
  type: date
  jsonPath: .metadata.creationTimestamp
- name: Endpoint
  type: string
  jsonPath: .status.endpoint
  priority: 1              # 默认隐藏，-o wide 时显示
```

# 3. Operator 开发

## 3.1 Kubebuilder 脚手架

Kubebuilder 是 Kubernetes SIG 维护的 Operator 开发框架，提供完整的开发脚手架：

```bash
# 创建项目
kubebuilder init --domain example.com --repo github.com/example/my-operator

# 创建 API（生成 CRD）
kubebuilder create api \
  --group database \
  --version v1 \
  --kind Database \
  --resource true \      # 生成 CRD
  --controller true      # 生成 Controller

# 创建 Webhook
kubebuilder create webhook \
  --group database \
  --version v1 \
  --kind Database \
  --defaulting \         # Mutating Webhook (Defaulting)
  --programmatic-validation  # Validating Webhook
```

### 3.1.1 生成的项目结构

```
my-operator/
├── api/
│   └── v1/
│       ├── database_types.go          # CRD 类型定义
│       ├── database_webhook.go        # Webhook 逻辑
│       ├── groupversion_info.go       # GVK 定义
│       └── zz_generated.deepcopy.go   # 自动生成的 DeepCopy
├── internal/
│   └── controller/
│       ├── database_controller.go     # Controller 逻辑
│       └── suite_test.go              # 测试
├── config/
│   ├── crd/                           # CRD YAML
│   ├── rbac/                          # RBAC 配置
│   └── manager/                       # Manager 部署
├── Makefile
└── go.mod
```

## 3.2 Operator 核心代码

### 3.2.1 类型定义 (database_types.go)

```go
package v1

import (
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// DatabaseSpec 定义期望状态
type DatabaseSpec struct {
    // +kubebuilder:validation:Enum=mysql;postgresql;mongodb
    Engine string `json:"engine"`

    Version string `json:"version"`

    // +kubebuilder:validation:Minimum=1
    // +kubebuilder:validation:Maximum=10
    Replicas int32 `json:"replicas"`

    // +optional
    Storage string `json:"storage,omitempty"`

    // +optional
    Resources *corev1.ResourceRequirements `json:"resources,omitempty"`

    // +optional
    Config map[string]string `json:"config,omitempty"`
}

// DatabaseStatus 定义观察到的状态
type DatabaseStatus struct {
    Phase          string `json:"phase,omitempty"`
    ReadyReplicas  int32  `json:"readyReplicas,omitempty"`
    Endpoint       string `json:"endpoint,omitempty"`
    Conditions     []metav1.Condition `json:"conditions,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Engine",type="string",JSONPath=".spec.engine"
// +kubebuilder:printcolumn:name="Status",type="string",JSONPath=".status.phase"
type Database struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    Spec   DatabaseSpec   `json:"spec,omitempty"`
    Status DatabaseStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true
type DatabaseList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items []Database `json:"items"`
}
```

### 3.2.2 控制器逻辑 (database_controller.go)

```go
func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx)

    // 1. 获取 Database 资源
    var db databasev1.Database
    if err := r.Get(ctx, req.NamespacedName, &db); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 2. 检查是否被删除（执行清理）
    if !db.DeletionTimestamp.IsZero() {
        return r.handleDeletion(ctx, &db)
    }

    // 3. 添加 Finalizer
    if !containsString(db.Finalizers, databaseFinalizer) {
        db.Finalizers = append(db.Finalizers, databaseFinalizer)
        if err := r.Update(ctx, &db); err != nil {
            return ctrl.Result{}, err
        }
    }

    // 4. 调谐各子资源
    // 创建/更新 Service
    if err := r.reconcileService(ctx, &db); err != nil {
        return ctrl.Result{}, err
    }

    // 创建/更新 StatefulSet
    if err := r.reconcileStatefulSet(ctx, &db); err != nil {
        return ctrl.Result{}, err
    }

    // 创建/更新 ConfigMap
    if err := r.reconcileConfigMap(ctx, &db); err != nil {
        return ctrl.Result{}, err
    }

    // 5. 更新 Status
    if err := r.updateStatus(ctx, &db); err != nil {
        return ctrl.Result{}, err
    }

    // 6. 返回（可设置 RequeueAfter 实现周期性检查）
    return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
}

// reconcileStatefulSet 创建或更新 StatefulSet
func (r *DatabaseReconciler) reconcileStatefulSet(ctx context.Context, db *databasev1.Database) error {
    sts := &appsv1.StatefulSet{}
    err := r.Get(ctx, types.NamespacedName{
        Name:      db.Name,
        Namespace: db.Namespace,
    }, sts)

    if errors.IsNotFound(err) {
        // StatefulSet 不存在，创建
        sts = r.buildStatefulSet(db)
        return r.Create(ctx, sts)
    }
    if err != nil {
        return err
    }

    // StatefulSet 存在，检查是否需要更新
    if *sts.Spec.Replicas != db.Spec.Replicas {
        sts.Spec.Replicas = &db.Spec.Replicas
        return r.Update(ctx, sts)
    }

    return nil
}

// updateStatus 更新状态
func (r *DatabaseReconciler) updateStatus(ctx context.Context, db *databasev1.Database) error {
    // 检查 StatefulSet 就绪状况
    sts := &appsv1.StatefulSet{}
    if err := r.Get(ctx, types.NamespacedName{
        Name: db.Name, Namespace: db.Namespace,
    }, sts); err != nil {
        return err
    }

    db.Status.ReadyReplicas = sts.Status.ReadyReplicas

    if sts.Status.ReadyReplicas == db.Spec.Replicas {
        db.Status.Phase = "Running"
        db.Status.Endpoint = fmt.Sprintf("%s.%s.svc.cluster.local", db.Name, db.Namespace)
    } else {
        db.Status.Phase = "Creating"
    }

    return r.Status().Update(ctx, db)
}
```

### 3.2.3 RBAC 标记

```go
// +kubebuilder:rbac:groups=database.example.com,resources=databases,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=database.example.com,resources=databases/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=apps,resources=statefulsets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=services,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=configmaps,verbs=get;list;watch;create;update;patch;delete

func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // ...
}
```

## 3.3 Webhook 实现

### 3.3.1 Defaulting Webhook（设置默认值）

```go
// +kubebuilder:webhook:path=/mutate-database-example-com-v1-database,mutating=true,failurePolicy=fail,groups=database.example.com,resources=databases,verbs=create;update,versions=v1,name=mdatabase.kb.io,sideEffects=None,admissionReviewVersions=v1

func (d *Database) Default() {
    if d.Spec.Replicas == 0 {
        d.Spec.Replicas = 1
    }
    if d.Spec.Storage == "" {
        d.Spec.Storage = "10Gi"
    }
    if d.Spec.Resources == nil {
        d.Spec.Resources = &corev1.ResourceRequirements{
            Requests: corev1.ResourceList{
                corev1.ResourceCPU:    resource.MustParse("500m"),
                corev1.ResourceMemory: resource.MustParse("512Mi"),
            },
        }
    }
}
```

### 3.3.2 Validating Webhook（验证逻辑）

```go
// +kubebuilder:webhook:path=/validate-database-example-com-v1-database,mutating=false,failurePolicy=fail,groups=database.example.com,resources=databases,verbs=create;update,versions=v1,name=vdatabase.kb.io,sideEffects=None,admissionReviewVersions=v1

func (d *Database) ValidateCreate() error {
    return d.validate()
}

func (d *Database) validate() error {
    var allErrs field.ErrorList

    if d.Spec.Engine == "postgresql" && d.Spec.Version < "13.0" {
        allErrs = append(allErrs, field.Invalid(
            field.NewPath("spec").Child("version"),
            d.Spec.Version,
            "PostgreSQL version must be >= 13.0",
        ))
    }

    if len(allErrs) == 0 {
        return nil
    }
    return apierrors.NewInvalid(
        schema.GroupKind{Group: "database.example.com", Kind: "Database"},
        d.Name, allErrs,
    )
}
```

# 4. 生产实践

## 4.1 Operator 开发生命周期

```
阶段 1: 领域建模
├── 确定 CRD 的 spec（用户需要声明什么）
├── 确定 CRD 的 status（需要暴露什么状态）
└── 定义版本策略（v1alpha1 → v1beta1 → v1）

阶段 2: 开发控制器
├── 使用 Kubebuilder / Operator SDK 脚手架
├── 实现 Reconcile 循环
├── 处理 Finalizer（清理逻辑）
└── 编写单元测试和集成测试

阶段 3: 测试
├── 使用 envtest（本地 API Server）
├── 使用 kind 集群进行 e2e 测试
├── 混沌测试（模拟故障场景）
└── 升级测试（CRD 版本升级路径）

阶段 4: 打包与分发
├── 生成 Helm Chart
├── 发布到 OperatorHub / Artifact Hub
└── 提供 OLM (Operator Lifecycle Manager) 支持
```

## 4.2 常见模式

### 4.2.1 Finalizer 模式

```go
const databaseFinalizer = "database.example.com/finalizer"

func (r *DatabaseReconciler) handleDeletion(ctx context.Context, db *databasev1.Database) (ctrl.Result, error) {
    if !containsString(db.Finalizers, databaseFinalizer) {
        return ctrl.Result{}, nil
    }

    // 执行清理：删除备份、释放存储、通知依赖服务
    if err := r.cleanup(ctx, db); err != nil {
        return ctrl.Result{}, err
    }

    // 移除 Finalizer（允许 Kubernetes 删除资源）
    db.Finalizers = removeString(db.Finalizers, databaseFinalizer)
    if err := r.Update(ctx, db); err != nil {
        return ctrl.Result{}, err
    }

    return ctrl.Result{}, nil
}
```

### 4.2.2 OwnerReference 模式

```go
// 设置 OwnerReference，确保父资源删除时子资源被级联删除
func (r *DatabaseReconciler) buildStatefulSet(db *databasev1.Database) *appsv1.StatefulSet {
    sts := &appsv1.StatefulSet{
        ObjectMeta: metav1.ObjectMeta{
            Name:      db.Name,
            Namespace: db.Namespace,
        },
        Spec: appsv1.StatefulSetSpec{
            Replicas: &db.Spec.Replicas,
            // ...
        },
    }
    // 设置 OwnerReference
    ctrl.SetControllerReference(db, sts, r.Scheme)
    return sts
}
```

### 4.2.3 Status Conditions 模式

```go
func (r *DatabaseReconciler) setCondition(db *databasev1.Database, condType string, status metav1.ConditionStatus, reason, message string) {
    condition := metav1.Condition{
        Type:               condType,
        Status:             status,
        Reason:             reason,
        Message:            message,
        LastTransitionTime: metav1.Now(),
        ObservedGeneration: db.Generation,
    }
    meta.SetStatusCondition(&db.Status.Conditions, condition)
}
```

## 4.3 推荐的社区 Operator

| Operator | 管理对象 | 水平 |
|---------|---------|------|
| **Prometheus Operator** | Prometheus, Alertmanager, Grafana | Level 5 |
| **cert-manager** | TLS 证书（自动签发和续期） | Level 4 |
| **Strimzi** | Apache Kafka | Level 4 |
| **Zalando Postgres Operator** | PostgreSQL 集群 | Level 4 |
| **RabbitMQ Operator** | RabbitMQ 集群 | Level 3 |
| **Elastic Cloud on K8s (ECK)** | Elasticsearch, Kibana, APM | Level 4 |
| **VictoriaMetrics Operator** | VictoriaMetrics 集群 | Level 3 |
