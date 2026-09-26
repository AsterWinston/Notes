
# 1. 为什么需要服务发现

在动态基础设施中（Kubernetes、自动伸缩组、Consul 等环境下），目标实例会频繁启停、扩缩容。使用静态配置会导致：

- 每次实例变更都需要重启 Prometheus
- 无法自动处理扩缩容
- 增加运维负担

**服务发现机制**让 Prometheus 自动发现、添加、移除采集目标。

# 2. 静态服务发现

用于固定环境或测试场景：

```yaml
scrape_configs:
  - job_name: 'static-targets'
    static_configs:
      - targets: ['10.0.0.1:9090', '10.0.0.2:9090']
        labels:
          env: 'production'
      - targets: ['10.0.0.3:9090']
        labels:
          env: 'staging'
```

# 3. 文件服务发现

基于 JSON/YAML 文件实现动态发现，适合自定义自动化场景：

```yaml
scrape_configs:
  - job_name: 'file-sd'
    file_sd_configs:
      - files:
        - 'targets/*.json'
        - 'targets/*.yml'
        refresh_interval: 5m
```

## 3.1 targets JSON 格式

```json
[
  {
    "targets": ["10.0.0.1:9100", "10.0.0.2:9100"],
    "labels": {
      "env": "production",
      "role": "web"
    }
  },
  {
    "targets": ["10.0.0.3:9100"],
    "labels": {
      "env": "staging",
      "role": "db"
    }
  }
]
```

## 3.2 targets YAML 格式

```yaml
- targets:
  - "10.0.0.1:9100"
  - "10.0.0.2:9100"
  labels:
    env: production
    role: web

- targets:
  - "10.0.0.3:9100"
  labels:
    env: staging
    role: db
```

# 4. Consul 服务发现

## 4.1 配置

```yaml
scrape_configs:
  - job_name: 'consul-services'
    consul_sd_configs:
      - server: 'localhost:8500'
        datacenter: 'dc1'
        scheme: 'http'
        token: 'consul-token'
        services:
          - 'web'
          - 'api'
    relabel_configs:
      # 只保留携带 exporter 标签的服务
      - source_labels: [__meta_consul_tags]
        regex: '.*exporter.*'
        action: keep
      # 将 consul 服务地址映射为采集地址
      - source_labels: [__meta_consul_service_address]
        target_label: __address__
      - regex: __meta_consul_service_metadata_(.+)
        action: labelmap
```

## 4.2 Consul 元数据标签

| 标签 | 说明 |
|------|------|
| `__meta_consul_address` | Consul 节点地址 |
| `__meta_consul_dc` | 数据中心 |
| `__meta_consul_service_id` | 服务 ID |
| `__meta_consul_service` | 服务名 |
| `__meta_consul_service_address` | 服务地址 |
| `__meta_consul_service_port` | 服务端口 |
| `__meta_consul_tags` | 逗号分隔的 Tags |
| `__meta_consul_service_metadata_<key>` | 服务元数据 |

# 5. Kubernetes 服务发现

## 5.1 配置

```yaml
scrape_configs:
  # Node 级别指标
  - job_name: 'kubernetes-nodes'
    kubernetes_sd_configs:
      - role: node
    relabel_configs:
      - source_labels: [__address__]
        regex: '(.*):10250'
        replacement: '${1}:9100'
        target_label: __address__

  # Pod 级别指标
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: 'true'

  # Service 级别指标
  - job_name: 'kubernetes-services'
    kubernetes_sd_configs:
      - role: service
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_probe]
        action: keep
        regex: 'true'
```

## 5.2 Kubernetes 角色说明

| Role | 发现对象 | 典型用途 |
|------|----------|----------|
| `node` | Node 级别 | 采集 Node Exporter |
| `pod` | Pod 级别 | 采集应用指标 |
| `service` | Service 级别 | 采集 Service 后端 |
| `endpoints` | Endpoint 级别 | 采集 Endpoint 后端 |
| `endpointslice` | EndpointSlice 级别 | 大规模集群替代 endpoints |
| `ingress` | Ingress 级别 | 通过黑盒 Exporter 探测 |

## 5.3 Pod 级元数据标签

| 标签 | 说明 |
|------|------|
| `__meta_kubernetes_namespace` | Pod 所在 Namespace |
| `__meta_kubernetes_pod_name` | Pod 名称 |
| `__meta_kubernetes_pod_label_<label>` | Pod 标签 |
| `__meta_kubernetes_pod_annotation_<annotation>` | Pod 注解 |
| `__meta_kubernetes_pod_controller_kind` | 控制器类型（ReplicaSet/StatefulSet 等） |
| `__meta_kubernetes_pod_controller_name` | 控制器名称 |
| `__meta_kubernetes_pod_host_ip` | Pod 所在节点 IP |
| `__meta_kubernetes_pod_ip` | Pod IP |
| `__meta_kubernetes_pod_ready` | Pod 是否就绪 |
| `__meta_kubernetes_pod_phase` | Pod 阶段（Running/Pending 等） |

## 5.4 注解驱动的采集模式

在 Pod 上添加注解，Prometheus 自动采集：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  annotations:
    prometheus.io/scrape: 'true'
    prometheus.io/port: '8080'
    prometheus.io/path: '/metrics'
    prometheus.io/scheme: 'http'
spec:
  containers:
  - name: app
    image: my-app:latest
```

# 6. 其他服务发现方式

## 6.1 DNS 服务发现

```yaml
scrape_configs:
  - job_name: 'dns-sd'
    dns_sd_configs:
      - names:
        - '_prometheus._tcp.example.com'
        type: 'SRV'
        port: 9100
        refresh_interval: 30s
```

## 6.2 EC2 服务发现

```yaml
scrape_configs:
  - job_name: 'ec2'
    ec2_sd_configs:
      - region: us-east-1
        access_key: 'AKIA...'
        secret_key: '...'
        port: 9100
    relabel_configs:
      - source_labels: [__meta_ec2_tag_Name]
        target_label: instance_name
      - source_labels: [__meta_ec2_private_ip]
        target_label: __address__
```

## 6.3 Azure 服务发现

```yaml
scrape_configs:
  - job_name: 'azure-vms'
    azure_sd_configs:
      - subscription_id: '...'
        tenant_id: '...'
        client_id: '...'
        client_secret: '...'
        port: 9100
```

## 6.4 GCE 服务发现

```yaml
scrape_configs:
  - job_name: 'gce'
    gce_sd_configs:
      - project: 'my-project'
        zone: 'us-central1-a'
        port: 9100
```

## 6.5 OpenStack 服务发现

```yaml
scrape_configs:
  - job_name: 'openstack'
    openstack_sd_configs:
      - role: instance
        region: regionOne
        identity_endpoint: 'http://...'
```

# 7. Relabeling 机制

Relabeling 是 Prometheus 最强大的功能之一，用于**在采集前转换目标标签**。

## 7.1 Relabel 动作

| 动作 | 说明 |
|------|------|
| `replace` | 替换标签值（默认） |
| `keep` | 保留匹配的目标 |
| `drop` | 丢弃匹配的目标 |
| `hashmod` | 基于哈希取模 |
| `labelmap` | 将匹配的源标签映射为新标签 |
| `labeldrop` | 删除匹配的标签 |
| `labelkeep` | 只保留匹配的标签 |

## 7.2 常用 Relabel 场景

```yaml
relabel_configs:
  # 基于标签丢弃目标
  - source_labels: [__meta_kubernetes_pod_phase]
    regex: 'Succeeded|Failed'
    action: drop

  # 替换标签名
  - source_labels: [__meta_ec2_tag_Name]
    target_label: instance_name
    action: replace

  # 将元数据标签映射为普通标签
  - regex: __meta_kubernetes_pod_label_(.+)
    action: labelmap
    replacement: k8s_$1

  # 保留特定标签，删除其余
  - regex: '(job|instance|__.*)'
    action: labelkeep

  # 删除敏感标签
  - regex: 'password|secret|token'
    action: labeldrop
```

## 7.3 默认 Relabel 流程

```
__address__ ──> instance (默认 replace)
__scheme__  ──> scheme
__metrics_path__ ──> metrics_path
__param_<name> ──> params
```

## 7.4 注意事项

- Relabel 顺序影响结果：先执行 `scrape_configs` 中的 `relabel_configs`，再执行默认 relabel
- 所有 `__meta_*` 标签在默认 relabel 后会被自动删除
- `__` 开头的标签不会出现在最终时间序列中

# 8. 服务发现选择建议

| 环境 | 推荐方案 |
|------|----------|
| 裸金属 | 静态 + 文件发现 |
| 云主机 | EC2 / Azure / GCE SD |
| 容器化 | K8s SD + Pod 注解 |
| 有注册中心 | Consul SD |
| 混合环境 | 文件发现（配合 CMDB 或编排工具） |
