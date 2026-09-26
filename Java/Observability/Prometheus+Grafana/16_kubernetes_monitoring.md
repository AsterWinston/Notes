
# 1. 监控体系概览

在 Kubernetes 环境中，Prometheus 通常配合 **kube-prometheus-stack**（原 Prometheus Operator）部署，提供开箱即用的监控方案。

## 1.1 K8s 监控层次

```
+--------------------------------------------------+
|                    Grafana                        |
|  Dashboard, Alerting, Explore                    |
+--------------------------------------------------+
                      |
+--------------------------------------------------+
|  Prometheus Server (+ Alertmanager)              |
|  + kube-state-metrics (K8s 对象状态)              |
|  + node-exporter (节点指标)                      |
|  + blackbox-exporter (网络探测)                  |
+--------------------------------------------------+
                      |
+--------------------------------------------------+
|  Kubernetes 数据来源                              |
|  + kubelet / cAdvisor (容器指标)                 |
|  + kube-apiserver / etcd (控制面指标)            |
|  + Pod 中应用指标 (/metrics)                     |
|  + Service / Endpoint 指标                       |
+--------------------------------------------------+
```

# 2. kube-prometheus-stack

## 2.1 Helm 安装

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --values values.yaml
```

## 2.2 组件清单

```yaml
# Chart 包含的组件
- prometheus-operator         # Prometheus Operator
- prometheus                  # Prometheus Server
- alertmanager                # Alertmanager
- node-exporter               # Node Exporter
- kube-state-metrics          # K8s 对象状态
- grafana                     # Grafana + 预置 Dashboard
- prometheus-adapter          # HPA 自定义指标适配器
```

# 3. 核心指标来源

## 3.1 kubelet / cAdvisor

kubelet 内置 cAdvisor，暴露容器运行时指标：

```yaml
scrape_configs:
  - job_name: 'kubelet'
    kubernetes_sd_configs:
      - role: node
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    authorization:
      credentials_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
```

```promql
# 容器 CPU 使用率
sum(rate(container_cpu_usage_seconds_total{container!="", pod!=""}[5m])) by (pod, container)

# 容器内存使用
sum(container_memory_working_set_bytes{container!="", pod!=""}) by (pod, container)

# 容器网络 IO
rate(container_network_receive_bytes_total{pod!=""}[5m])
rate(container_network_transmit_bytes_total{pod!=""}[5m])

# 容器文件系统
container_fs_usage_bytes{container!=""}
```

## 3.2 kube-state-metrics

kube-state-metrics 监控 K8s 对象状态，**不采集运行时的资源使用量**，而是采集对象元数据。

```promql
# Pod 状态分布
count(kube_pod_status_phase{phase="Running"}) by (namespace)
count(kube_pod_status_phase{phase="Pending"}) by (namespace)
count(kube_pod_status_phase{phase="Failed"}) by (namespace)

# Deployment 期望副本 vs 可用副本
kube_deployment_spec_replicas{deployment="my-app"}
kube_deployment_status_replicas_available{deployment="my-app"}

# Node 状态
kube_node_status_condition{condition="Ready", status="true"}

# 资源请求与限制
kube_pod_container_resource_requests{resource="cpu"}
kube_pod_container_resource_limits{resource="memory"}
```

| kube-state-metrics 指标 | 说明 |
|------------------------|------|
| `kube_pod_status_phase` | Pod 阶段（Running/Pending/Succeeded/Failed） |
| `kube_pod_info` | Pod 基本信息 |
| `kube_pod_container_status_restarts_total` | 容器重启次数 |
| `kube_node_status_capacity` | Node 资源容量 |
| `kube_node_status_allocatable` | Node 可分配资源 |
| `kube_deployment_status_replicas` | Deployment 副本状态 |
| `kube_service_status` | Service 状态 |
| `kube_namespace_status_phase` | Namespace 状态 |
| `kube_horizontalpodautoscaler_status_current_replicas` | HPA 当前副本数 |
| `kube_persistentvolumeclaim_status_phase` | PVC 状态 |

## 3.3 node-exporter（K8s 版本）

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    spec:
      hostNetwork: true
      hostPID: true
      containers:
        - name: node-exporter
          image: prom/node-exporter:v1.8.0
          args:
            - --path.rootfs=/host
          volumeMounts:
            - name: root
              mountPath: /host
              readOnly: true
      volumes:
        - name: root
          hostPath:
            path: /
      tolerations:
        - operator: Exists
```

# 4. PodMonitor 和 ServiceMonitor

Prometheus Operator 引入了 CRD（Custom Resource Definition）方式管理采集配置。

## 4.1 ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: my-app
  namespaceSelector:
    any: true    # 从所有命名空间匹配
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
      scheme: http
      relabelings:
        - sourceLabels: [__meta_kubernetes_pod_name]
          targetLabel: pod
```

## 4.2 PodMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  podMetricsEndpoints:
    - port: metrics
      interval: 15s
      path: /metrics
  namespaceSelector:
    any: true
```

## 4.3 ServiceMonitor vs PodMonitor

| 特性 | ServiceMonitor | PodMonitor |
|------|---------------|------------|
| 匹配对象 | Service | Pod |
| 适用场景 | Service 有稳定端点 | 直接采集 Pod IP |
| 端点多路 | 通过 Service 端口 | 直连 Pod |
| Service 代理 | 支持（K8s Service） | 不支持 |

# 5. 控制面监控

## 5.1 API Server 指标

```yaml
scrape_configs:
  - job_name: 'apiserver'
    kubernetes_sd_configs:
      - role: endpoints
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    authorization:
      credentials_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name]
        action: keep
        regex: default;kubernetes
```

```promql
# API Server 请求速率
rate(apiserver_request_total[5m])

# API Server 延迟
histogram_quantile(0.99, rate(apiserver_request_duration_seconds_bucket[5m]))

# 工作队列深度
apiserver_workqueue_depth
```

## 5.2 etcd 指标

etcd 通常部署在集群内，需要配置证书访问：

```promql
# etcd leader 变更
rate(etcd_server_leader_changes_seen_total[5m])

# etcd 请求延迟
histogram_quantile(0.99, rate(etcd_request_duration_seconds_bucket[5m]))

# etcd 磁盘写入延迟
histogram_quantile(0.99, rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m]))

# etcd 存储 db 大小
etcd_mvcc_db_total_size_in_bytes
```

# 6. 常见 K8s 告警规则

```yaml
groups:
  - name: kubernetes_alerts
    rules:
      - alert: PodCrashLooping
        expr: rate(kube_pod_container_status_restarts_total[15m]) * 60 > 5
        for: 5m
        annotations:
          summary: "Pod {{ $labels.pod }} is crash looping"

      - alert: PodPending
        expr: kube_pod_status_phase{phase="Pending"} > 0
        for: 5m
        annotations:
          summary: "Pod {{ $labels.pod }} is pending for 5 minutes"

      - alert: NodeNotReady
        expr: kube_node_status_condition{condition="Ready", status="true"} == 0
        for: 5m
        annotations:
          summary: "Node {{ $labels.node }} is not ready"

      - alert: PersistentVolumeFillingUp
        expr: kubelet_volume_stats_available_bytes / kubelet_volume_stats_capacity_bytes < 0.1
        for: 5m
        annotations:
          summary: "Volume {{ $labels.persistentvolumeclaim }} is filling up"

      - alert: HighCPURequested
        expr: sum(kube_pod_container_resource_requests{resource="cpu"}) by (node) / sum(kube_node_status_allocatable{resource="cpu"}) by (node) > 0.8
        for: 10m
        annotations:
          summary: "Node {{ $labels.node }} CPU requested > 80%"
```

# 7. HPA 自定义指标

使用 Prometheus Adapter 暴露自定义指标给 HPA：

```yaml
# prometheus-adapter 配置
rules:
  - seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
    resources:
      overrides:
        namespace: { resource: "namespace" }
        pod: { resource: "pod" }
    name:
      matches: '^(.*)_total$'
      as: '${1}_per_second'
    metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'
```

```yaml
# HPA 使用自定义指标
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: 100
```

# 8. 多集群监控

## 8.1 Thanos 全局视图

在每个 K8s 集群中部署 Thanos Sidecar + Prometheus，集中到中心 Thanos Query：

```yaml
# Cluster 1
prometheus + thanos-sidecar ─┐
                             ├── thanos-query (global)
# Cluster 2                    │
prometheus + thanos-sidecar ─┘ │
                               │
                         thanos-store ← S3 (对象存储）
```

## 8.2 Grafana 多数据源

```yaml
datasources:
  - name: Cluster A
    type: prometheus
    url: http://thanos-query-a:10902
  - name: Cluster B
    type: prometheus
    url: http://thanos-query-b:10902
```
