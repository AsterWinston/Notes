# 1. 可观测性概述

## 1.1 可观测性的三大支柱

在云原生生态中，可观测性（Observability）通常被定义为三大支柱：

```
┌─────────────────────────────────────────────────────┐
│                    可观测性                           │
│                                                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │  Metrics │  │  Logging │  │  Tracing │           │
│  │  (指标)   │  │  (日志)   │  │  (追踪)   │           │
│  ├──────────┤  ├──────────┤  ├──────────┤           │
│  │ 回答:     │  │ 回答:     │  │ 回答:     │           │
│  │ "发生什么" │  │ "为什么    │  │ "在哪儿    │           │
│  │          │  │  发生"    │  │  发生"    │           │
│  ├──────────┤  ├──────────┤  ├──────────┤           │
│  │ 工具:     │  │ 工具:     │  │ 工具:     │           │
│  │ Prometheus│  │ EFK/ELK   │  │ Jaeger    │           │
│  │ Grafana  │  │ Loki      │  │ Zipkin    │           │
│  └──────────┘  └──────────┘  └──────────┘           │
└─────────────────────────────────────────────────────┘
```

本章聚焦于 Metrics 和核心监控体系，日志和追踪在后续章节展开。

## 1.2 Kubernetes 监控的层次

```
层次 1: 基础设施层
├── 节点资源: CPU, Memory, Disk, Network
├── etcd 状态: 延迟、存储大小、Raft 状态
└── kubelet/kube-proxy 状态

层次 2: Kubernetes 对象层
├── 集群状态: 节点数、Pod 数、Namespace 数
├── 控制器状态: Deployment 就绪副本、Job 完成情况
└── 调度器状态: 调度延迟、Pending Pod 数量

层次 3: 应用层
├── 应用指标: QPS、延迟、错误率
├── 业务指标: 订单数、用户注册数
└── 依赖状态: 数据库连接池、缓存命中率
```

# 2. Prometheus 体系

## 2.1 为什么选择 Prometheus

Prometheus 是 CNCF 继 Kubernetes 之后的第二个毕业项目，已成为云原生监控的事实标准。它的核心设计理念与 Kubernetes 高度契合：

**Pull 模型**：Prometheus 主动从目标抓取指标（而非等待推送），这与 Kubernetes 的服务发现机制天然结合——Prometheus 通过 Kubernetes API 自动发现需要监控的 Pod 和 Service。

**多维数据模型**：指标由 metric name + labels（键值对）组成，可以灵活地聚合和过滤。

**PromQL**：强大的查询语言，支持聚合、计算、预测等复杂操作。

**不依赖外部存储**：Prometheus 本身是一个时序数据库，可以直接存储和查询指标数据。

## 2.2 Prometheus 架构

```
┌─────────────────────────────────────────────────────────┐
│                     Prometheus 生态                       │
│                                                           │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐             │
│  │ Short-lived│  │  Node    │   │  Service │             │
│  │  Jobs     │   │ Exporter │   │ Discovery│             │
│  └─────┬─────┘   └────┬─────┘   └────┬─────┘             │
│        │ Pushgateway  │              │                     │
│        │    ┌─────────┘              │                     │
│        ▼    ▼                        ▼                     │
│  ┌──────────────────────────────────────┐                │
│  │           Prometheus Server           │                │
│  │  ┌────────┐ ┌────────┐ ┌──────────┐ │                │
│  │  │Retrieval│→│Storage │→│PromQL    │ │                │
│  │  └────────┘ └────────┘ └──────────┘ │                │
│  └──────────────────┬───────────────────┘                │
│                     │                                      │
│                     ▼                                      │
│  ┌──────────────────────────────────────┐                │
│  │              Grafana                  │                │
│  │         (可视化仪表盘)                 │                │
│  └──────────────────────────────────────┘                │
│                                                           │
│  ┌──────────────────────────────────────┐                │
│  │          Alertmanager                 │                │
│  │  (告警管理：分组/抑制/路由)             │                │
│  └──────────────────────────────────────┘                │
│                     │                                      │
│                     ▼                                      │
│          Email / Slack / PagerDuty / Webhook              │
└─────────────────────────────────────────────────────────┘
```

## 2.3 Metrics Server

Metrics Server 是 Kubernetes 官方维护的轻量级指标收集器，是 HPA 和 `kubectl top` 的数据来源：

```bash
# 安装 Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# 查看节点资源使用
kubectl top nodes

# 查看 Pod 资源使用
kubectl top pods -n production
kubectl top pods -A --sort-by=cpu
kubectl top pods -A --sort-by=memory
```

**Metrics Server vs Prometheus**：
- Metrics Server：轻量，只提供 CPU/Memory 的当前值，不保存历史数据，HPA 必需
- Prometheus：功能全，支持历史数据、复杂查询、告警、多数据源

两者互补——Metrics Server 用于 Kubernetes 内部（HPA），Prometheus 用于运维监控。

## 2.4 kube-state-metrics

kube-state-metrics 是一个关键组件，它监听 Kubernetes API Server 并生成关于资源状态的指标（如 Deployment 的副本数、Pod 的 Restart Count 等）。这些是"对象状态"指标，而非 cAdvisor 提供的"资源使用"指标。

```yaml
# kube-state-metrics 提供的典型指标
kube_deployment_status_replicas_ready     # Deployment 就绪副本数
kube_pod_status_phase                     # Pod 状态 (Running/Pending/Failed)
kube_node_status_condition                # 节点状态
kube_persistentvolume_status_phase        # PV 状态
kube_job_status_failed                    # Job 失败次数
kube_statefulset_status_replicas_ready    # StatefulSet 就绪副本数
```

### 2.4.1 关键告警规则示例

```yaml
# PrometheusRule: 关键告警
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: kubernetes-alerts
spec:
  groups:
  - name: kubernetes-apps
    rules:
    # Pod 频繁重启
    - alert: PodCrashLooping
      expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} is crash looping"

    # Deployment 副本不可用
    - alert: DeploymentReplicasNotAvailable
      expr: |
        kube_deployment_status_replicas_available 
        / kube_deployment_status_replicas 
        < 1
      for: 10m
      labels:
        severity: critical

    # 节点 NotReady
    - alert: NodeNotReady
      expr: kube_node_status_condition{condition="Ready",status="true"} == 0
      for: 5m
      labels:
        severity: critical

    # PersistentVolume 使用率过高
    - alert: PersistentVolumeFillingUp
      expr: |
        (kubelet_volume_stats_available_bytes 
        / kubelet_volume_stats_capacity_bytes) 
        < 0.1
      for: 5m
      labels:
        severity: warning

    # 节点内存压力
    - alert: NodeMemoryPressure
      expr: kube_node_status_condition{condition="MemoryPressure",status="true"} == 1
      for: 5m
      labels:
        severity: warning
```

## 2.5 Node Exporter

Node Exporter 以 DaemonSet 方式部署在每个节点上，暴露节点级别的硬件和内核指标：

```
常见指标：
node_cpu_seconds_total              # CPU 使用时间
node_memory_MemAvailable_bytes      # 可用内存
node_disk_io_time_seconds_total     # 磁盘 I/O 时间
node_network_receive_bytes_total    # 网络接收字节
node_filesystem_avail_bytes         # 文件系统可用空间
node_load1 / node_load5 / node_load15  # 系统负载
```

### 2.5.1 资源使用 PromQL 示例

```promql
# CPU 使用率（百分比）
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# 内存使用率（百分比）
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# 磁盘 I/O 利用率
rate(node_disk_io_time_seconds_total{device!=""}[5m]) * 100

# Pod CPU 使用量（按 Namespace 聚合）
sum(rate(container_cpu_usage_seconds_total{namespace="production"}[5m])) by (pod)

# Pod 内存使用量（按 Namespace 聚合）
sum(container_memory_working_set_bytes{namespace="production"}) by (pod)
```

# 3. Prometheus Operator

## 3.1 核心 CRD

Prometheus Operator 将 Prometheus 生态的配置抽象为 Kubernetes CRD：

| CRD | 作用 |
|-----|------|
| `Prometheus` | 定义 Prometheus 实例的配置 |
| `ServiceMonitor` | 声明式定义要监控的 Service |
| `PodMonitor` | 对没有 Service 的 Pod 进行监控 |
| `PrometheusRule` | 定义告警规则 |
| `AlertmanagerConfig` | 定义 Alertmanager 路由配置 |

## 3.2 ServiceMonitor 配置

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-monitor
  namespace: production
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
  - port: metrics           # Service 中定义的端口名
    interval: 30s           # 抓取间隔
    path: /metrics          # 指标路径
    scheme: http
    relabelings:            # 重新标记
    - sourceLabels: [__meta_kubernetes_pod_node_name]
      targetLabel: node
  namespaceSelector:
    matchNames:
    - production
```

## 3.3 Prometheus 实例配置

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: main
  namespace: monitoring
spec:
  serviceAccountName: prometheus
  replicas: 2                               # 高可用
  retention: 15d                            # 数据保留 15 天
  storage:
    volumeClaimTemplate:                    # 持久化存储
      spec:
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 100Gi
  resources:
    requests:
      memory: 2Gi
      cpu: 500m
  serviceMonitorSelector:                   # 选择要监控的 ServiceMonitor
    matchLabels:
      team: platform
  ruleSelector:                             # 选择告警规则
    matchLabels:
      role: alert-rules
  alerting:
    alertmanagers:
    - namespace: monitoring
      name: alertmanager
      port: web
```

# 4. Grafana 可视化

## 4.1 数据源与仪表盘

```yaml
# Grafana 数据源配置 (通过 ConfigMap 或 UI)
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
data:
  datasources.yaml: |
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus-operated.monitoring.svc:9090
      access: proxy
      isDefault: true
```

## 4.2 常用 Kubernetes 仪表盘

| 仪表盘 ID（grafana.com） | 名称 | 内容 |
|--------------------------|------|------|
| 315 | Kubernetes Cluster | 集群级别概览 |
| 6417 | Kubernetes Cluster (Prometheus Operator) | 集群详细指标 |
| 8588 | Kubernetes Deployment | Deployment 监控 |
| 747 | Kubernetes Pod | Pod 级别指标 |
| 7249 | Kubernetes Node | 节点详情 |

## 4.3 自定义仪表盘 JSON 示例

```json
{
  "panels": [
    {
      "title": "Pod CPU Usage",
      "targets": [
        {
          "expr": "sum(rate(container_cpu_usage_seconds_total{namespace=\"$namespace\",pod=~\"$pod\"}[5m])) by (pod)",
          "legendFormat": "{{pod}}"
        }
      ],
      "gridPos": {"h": 8, "w": 12}
    }
  ],
  "templating": {
    "list": [
      {"name": "namespace", "type": "datasource", "query": "label_values(namespace)"},
      {"name": "pod", "type": "query", "query": "label_values(container_cpu_usage_seconds_total{namespace=\"$namespace\"}, pod)"}
    ]
  }
}
```

# 5. 金丝雀发布监控

## 5.1 RED 方法

对于微服务监控，推荐使用 RED 方法：

- **Rate (请求速率)**：每秒处理多少请求
- **Errors (错误率)**：失败的请求比例
- **Duration (延迟)**：请求处理的响应时间分布（P50/P95/P99）

```promql
# Rate: 每秒请求数
rate(http_requests_total[5m])

# Errors: 错误率
rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])

# Duration: P99 延迟
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

## 5.2 金丝雀发布对比监控

```promql
# 对比金丝雀版本和稳定版本的错误率
rate(http_requests_total{version="canary",status=~"5.."}[5m])
/
rate(http_requests_total{version="canary"}[5m])
# vs
rate(http_requests_total{version="stable",status=~"5.."}[5m])
/
rate(http_requests_total{version="stable"}[5m])
```

## 5.3 SLO/SLI 监控面板

```promql
# 可用性 SLI (错误预算消耗率)
1 - (
  sum(rate(http_requests_total{status!~"5.."}[30d]))
  /
  sum(rate(http_requests_total[30d]))
)

# 定义为 99.9% SLO
# 如果此值 > 0.999，说明过去 30 天满足 SLO
```

# 6. Alertmanager 告警

## 6.1 告警路由配置

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-main
type: Opaque
stringData:
  alertmanager.yaml: |
    global:
      smtp_smarthost: 'smtp.example.com:587'
      smtp_from: 'alertmanager@example.com'
    route:
      group_by: ['alertname', 'namespace']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      receiver: 'default'
      routes:
      - match:
          severity: critical
        receiver: 'pagerduty-critical'
      - match:
          severity: warning
        receiver: 'slack-warning'
    receivers:
    - name: 'default'
      email_configs:
      - to: 'ops@example.com'
    - name: 'pagerduty-critical'
      pagerduty_configs:
      - routing_key: '<PAGERDUTY_KEY>'
    - name: 'slack-warning'
      slack_configs:
      - api_url: '<SLACK_WEBHOOK_URL>'
        channel: '#k8s-alerts'
```

## 6.2 静默规则

```bash
# 创建静默（维护期间抑制告警）
curl -X POST http://alertmanager:9093/api/v2/silences \
  -H "Content-Type: application/json" \
  -d '{
    "matchers": [
      {"name": "namespace", "value": "production", "isRegex": false}
    ],
    "startsAt": "2024-01-15T02:00:00Z",
    "endsAt": "2024-01-15T04:00:00Z",
    "createdBy": "ops-team",
    "comment": "Planned maintenance window"
  }'
```

# 7. 生产实践

## 7.1 监控体系部署清单

```bash
# 推荐的监控组件（按优先级）
# 1. Metrics Server         - HPA 和 kubectl top 的依赖（必需）
# 2. kube-state-metrics     - Kubernetes 对象状态指标
# 3. Node Exporter           - 节点硬件指标
# 4. Prometheus Operator     - 简化 Prometheus 运维
# 5. Grafana                 - 可视化仪表盘
# 6. Alertmanager            - 告警管理
```

## 7.2 Prometheus 存储规划

```yaml
# 存储大小估算公式
# 存储 = 样本数/秒 × 样本大小(约 2 bytes) × 保留时间(秒)
# 
# 示例：1000 samples/s × 2 bytes × 15天 × 86400秒
#     = 1000 × 2 × 1296000
#     = 2.59GB (约 3GB 存储)

# 使用 Thanos 或 VictoriaMetrics 实现长期存储和高可用
```

## 7.3 生产告警清单

| 告警级别 | 告警名称 | 阈值 |
|---------|---------|------|
| Critical | NodeNotReady | 任意节点 NotReady > 5min |
| Critical | KubeAPIDown | API Server 不可达 |
| Critical | KubeletDown | kubelet 心跳停止 |
| Warning | PodCrashLooping | Pod 持续重启 > 15min |
| Warning | NodeMemoryPressure | 节点内存 < 10% |
| Warning | NodeDiskPressure | 节点磁盘 < 10% |
| Warning | PersistentVolumeFillingUp | PV 使用 > 90% |
| Info | HPAReachingLimit | HPA 接近上限 |
