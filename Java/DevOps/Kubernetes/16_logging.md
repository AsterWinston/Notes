# 1.Kubernetes 日志体系概述

## 1.1 日志的层次与挑战

在 Kubernetes 中，日志分布在多个层面：

```
层面 1: 应用日志
├── 业务日志: 请求日志、操作日志、业务事件
└── 框架日志: Spring Boot、Django、Express 等框架输出

层面 2: Kubernetes 组件日志
├── kubelet 日志: 节点管理、容器生命周期
├── kube-apiserver 日志: API 请求审计
├── kube-scheduler 日志: 调度决策
└── kube-controller-manager 日志: 控制器活动

层面 3: 基础设施日志
├── 容器运行时日志: containerd/cri-o 日志
├── 内核日志: dmesg、OOM 事件
└── 节点系统日志: systemd journal、syslog
```

核心挑战：
- **Pod 是短暂的**：Pod 删除后，其容器日志随之消失（除非有集中式日志收集）
- **分布性**：日志分散在数十到数千个 Pod 中，需要集中式存储和查询
- **格式不统一**：不同应用、不同语言输出的日志格式各异，需要标准化处理
- **大规模集群**：每天可能产生 TB 级日志，需要高效的收集、传输和存储方案

## 1.2 日志架构模式

Kubernetes 推荐使用**节点级日志代理**（Node-level Logging Agent）模式：

```
┌──────────────────────────────────────────────────┐
│                   Worker Node                      │
│                                                    │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐            │
│  │  Pod A  │ │  Pod B  │ │  Pod C  │            │
│  │ stdout  │ │ stdout  │ │ file    │            │
│  │ stderr  │ │ stderr  │ │ /var/log│            │
│  └────┬────┘ └────┬────┘ └───┬─────┘            │
│       │           │          │                    │
│       │  stdout/stderr → /var/log/containers/     │
│       │           │          │                    │
│       ▼           ▼          ▼                    │
│  ┌────────────────────────────────┐              │
│  │      Logging Agent (DaemonSet)  │              │
│  │   Fluentd / Fluent Bit / Vector │              │
│  └───────────────┬────────────────┘              │
│                  │                                 │
└──────────────────┼─────────────────────────────────┘
                   │
                   ▼
        ┌──────────────────────┐
        │   Central Log Store   │
        │  Elasticsearch / Loki │
        │  / ClickHouse / S3    │
        └──────────┬───────────┘
                   │
                   ▼
        ┌──────────────────────┐
        │   Visualization       │
        │  Kibana / Grafana    │
        └──────────────────────┘
```

# 2.日志收集与处理

## 2.1 kubectl logs 的原理

```bash
# kubectl logs 从 kubelet 获取日志
kubectl logs <pod-name> -n production
kubectl logs <pod-name> -c <container-name>  # 多容器 Pod 指定容器
kubectl logs <pod-name> --previous           # 查看上一个容器的日志
kubectl logs <pod-name> --tail=100 -f        # 流式查看
kubectl logs <pod-name> --since=1h           # 过去 1 小时的日志
kubectl logs -l app=my-app --all-containers  # 按标签查看所有容器日志
```

`kubectl logs` 的交互流程：
```
kubectl → API Server → kubelet (Log endpoint) → 读取 /var/log/containers/<pod>_<ns>_<container>*.log
```

日志文件的存储位置由容器运行时决定：
- **containerd**：`/var/log/pods/<namespace>_<pod>_<uid>/<container>/`
- **CRI-O**：`/var/log/pods/<namespace>_<pod>_<uid>/<container>/`
- **Docker (旧版)**：`/var/lib/docker/containers/<container-id>/`

## 2.2 Fluent Bit / Fluentd

Fluent Bit 是 C 语言编写的高性能日志转发器，Fluentd 是 Ruby 编写的全功能日志收集器。生产环境通常使用 Fluent Bit（轻量、低资源）作为节点代理，Fluentd 用于中央聚合。

```yaml
# Fluent Bit DaemonSet 配置
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:2.2
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: fluent-bit-config
          mountPath: /fluent-bit/etc/
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: fluent-bit-config
        configMap:
          name: fluent-bit-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         5
        Log_Level     info
        Daemon        off

    [INPUT]
        Name          tail
        Path          /var/log/containers/*.log
        Parser        docker
        Tag           kube.*
        Refresh_Interval 5
        Mem_Buf_Limit 5MB

    [FILTER]
        Name          kubernetes
        Match         kube.*
        Kube_URL      https://kubernetes.default.svc:443
        Merge_Log     On

    [OUTPUT]
        Name          es
        Match         *
        Host          elasticsearch.logging.svc
        Port          9200
        Index         k8s-logs
        Type          _doc
```

## 2.3 日志管道处理

```
原始文本日志
     │
     ▼
[Parser] 解析为结构化日志
     │
     ▼
[Filter/Kubernetes] 添加 Kubernetes 元数据
     │ (Pod名称, Namespace, Labels, Node名称)
     ▼
[Modify] 字段转换、删除敏感信息
     │
     ▼
[Router] 按 Namespace/Label 路由到不同目标
     │
     ├→ Elasticsearch (热数据, 7天)
     ├→ S3/OSS (冷归档, 90天)
     └→ Kafka (实时处理流)
```

# 3.EFK/ELK Stack

## 3.1 Elasticsearch + Fluentd + Kibana

```
Pod stdout/stderr
     │
     ▼
Fluentd DaemonSet (每个节点)  ← 收集 + 解析 + 添加元数据
     │
     ▼
Elasticsearch Cluster          ← 存储 + 索引 + 全文搜索
     │
     ▼
Kibana                         ← 可视化 + 搜索 + Dashboard
```

### Elasticsearch 部署要点

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: quickstart
spec:
  version: 8.12.0
  nodeSets:
  - name: default
    count: 3
    config:
      node.store.allow_mmap: false
    podTemplate:
      spec:
        containers:
        - name: elasticsearch
          resources:
            requests:
              memory: 4Gi
              cpu: 2
            limits:
              memory: 4Gi
              cpu: 2
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 200Gi
        storageClassName: fast-ssd
```

## 3.2 Kibana 查询示例

```
# 查看特定 Pod 的最新日志
kubernetes.pod_name: "my-app-7d5c8f-" AND kubernetes.namespace_name: "production"

# 错误日志
log: "*Error*" OR log: "*Exception*" 

# 按时间范围
@timestamp: [now-1h TO now]

# 聚合统计：每 5 分钟的错误数量
# 使用 Kibana 的 Visualization → Area Chart
# Y-axis: Count
# X-axis: @timestamp (5m interval)
# Filter: log: "*Error*"
```

# 4.Grafana Loki

## 4.1 Loki vs Elasticsearch

| 维度 | Elasticsearch | Loki |
|------|--------------|------|
| 索引方式 | 全文索引（索引所有内容） | 只索引标签（metadata），日志内容不索引 |
| 存储成本 | 高（索引膨胀 2-3x） | 低（压缩存储，约 1/3） |
| 查询性能 | 全文搜索极快 | 依赖标签过滤后扫描 |
| 资源需求 | 高（JVM 内存消耗） | 低（Go 编写，轻量） |
| 与 Prometheus 集成 | 需要额外配置 | 天然与 Grafana/Prometheus 标签模型一致 |

Loki 的设计哲学：**"不索引日志内容，只索引元数据标签"**。这大大降低了存储成本和运维复杂度。

## 4.2 Loki 架构

```
┌────────────┐   ┌──────────────┐
│ Promtail    │   │  Grafana Agent│  ← 日志收集代理（DaemonSet）
│ (每个Node)  │   │               │
└──────┬─────┘   └──────┬────────┘
       │                │
       ▼                ▼
┌──────────────────────────────────┐
│         Loki Distributor          │  ← 分发到 Ingester
└──────────────┬───────────────────┘
               ▼
┌──────────────────────────────────┐
│         Loki Ingester             │  ← 构建压缩后的 Chunks
└──────────────┬───────────────────┘
               ▼
┌──────────────────────────────────┐
│       Object Store (S3/GCS/MinIO) │  ← 持久存储
└──────────────────────────────────┘
```

### Promtail 配置

```yaml
# Promtail DaemonSet 配置
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: promtail
spec:
  template:
    spec:
      containers:
      - name: promtail
        image: grafana/promtail:2.9
        args:
        - -config.file=/etc/promtail/promtail.yaml
        volumeMounts:
        - name: config
          mountPath: /etc/promtail
        - name: varlog
          mountPath: /var/log
        - name: pods
          mountPath: /var/log/pods
      volumes:
      - name: config
        configMap:
          name: promtail-config
      - name: varlog
        hostPath:
          path: /var/log
      - name: pods
        hostPath:
          path: /var/log/pods
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: promtail-config
data:
  promtail.yaml: |
    server:
      http_listen_port: 9080
    clients:
    - url: http://loki-gateway.logging.svc/loki/api/v1/push
    positions:
      filename: /tmp/positions.yaml
    scrape_configs:
    - job_name: kubernetes-pods
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_node_name]
        target_label: node
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
      - source_labels: [__meta_kubernetes_pod_container_name]
        target_label: container
      - replacement: /var/log/pods/$1/*.log
        separator: /
        source_labels: [__meta_kubernetes_pod_uid, __meta_kubernetes_pod_container_name]
        target_label: __path__
```

## 4.3 LogQL 查询语言

```logql
# 基础查询：查找特定 Pod 日志
{namespace="production", pod=~"my-app.+"} 
  |= "ERROR"

# 过滤包含特定关键词的日志
{namespace="production", app="api"} 
  |~ "(?i)error|exception|timeout"

# 聚合统计：每 5 分钟错误数量
rate({namespace="production"} 
  |= "ERROR" [5m])

# 按 namespace 聚合
sum by (namespace) (
  rate({namespace=~".+"} |= "ERROR" [5m])
)

# JSON 日志解析
{app="api"} 
  | json 
  | status >= 500 
  | line_format "{{.method}} {{.path}} → {{.status}}"
```

# 5.审计日志 (Audit Log)

## 5.1 API Server 审计

Kubernetes API Server 支持详细的审计日志，记录所有 API 请求：

```yaml
# 审计策略配置
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# 记录所有请求的元数据（默认）
- level: Metadata

# 记录 Secret 和 ConfigMap 的请求体和响应体
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets", "configmaps"]

# 不记录对 kube-system 中资源的只读请求
- level: None
  userGroups: ["system:authenticated"]
  namespaces: ["kube-system"]
  verbs: ["get", "list", "watch"]

# 记录所有 Pod 变更的请求体
- level: Request
  resources:
  - group: ""
    resources: ["pods"]
  verbs: ["create", "update", "patch", "delete"]
```

```bash
# kube-apiserver 日志目录
/var/log/kube-apiserver-audit.log
```

## 5.2 事件审计

Kubernetes Events 也值得定期归档分析：

```bash
# 查看集群级事件
kubectl get events -A --sort-by='.lastTimestamp'

# 导出事件到文件
kubectl get events -A -o json > events-$(date +%Y%m%d).json

# 使用 eventrouter 将事件转发到日志系统
# 部署 eventrouter Deployment，将 Events 转为日志输出
```

# 6.日志管理最佳实践

## 6.1 应用日志规范

```
推荐的日志格式 (JSON):
{
  "timestamp": "2024-01-15T10:30:00Z",
  "level": "ERROR",
  "logger": "com.example.OrderService",
  "message": "Failed to process order",
  "traceId": "abc123-def456",
  "spanId": "span-001",
  "context": {
    "orderId": "ORD-12345",
    "userId": "user-678",
    "duration": "150ms"
  }
}
```

### 关键实践

1. **使用结构化日志**：JSON 格式便于机器解析和聚合
2. **包含 Trace ID**：关联日志与分布式追踪
3. **输出到 stdout/stderr**：不要写入容器内文件（Pod 删除时丢失）
4. **控制日志级别**：生产环境用 INFO/WARN，排查时临时改为 DEBUG
5. **避免在日志中记录敏感信息**：密码、Token、个人身份信息等应脱敏

## 6.2 日志保留策略

```
热数据 (Elasticsearch/Loki): 7-14 天
  ├── 用于日常排查和实时告警
  │
温数据 (对象存储): 30-90 天
  ├── 用于周期性分析和审计
  │
冷数据 (归档存储): 90 天 - 1 年
  └── 用于合规和审计需求
```

### 存储估算

```
日日志量估算:
  每个 Pod 平均 100MB/天 × 100 个 Pod = 10GB/天
  保留 30 天 = 300GB (压缩后约 100GB)

Loki 场景:
  100GB (原始) → 约 30GB (压缩后)
  适合中小规模集群

ES 场景:
  100GB (原始) → 约 200GB (含索引)
  适合需要全文搜索的大型集群
```

## 6.3 常见问题诊断

```bash
# 容器日志不出现在 kubectl logs 中
# 原因：应用将日志写入文件而非 stdout/stderr
# 解决：配置应用输出到 stdout，或使用 Sidecar 读取文件并输出

# 日志收集器占用过多资源
# 解决：
# - 使用 Fluent Bit 替代 Fluentd（更轻量）
# - 限制日志速率
# - 使用 Loki 替代 ES（更省资源）

# 日志丢失
# 原因：容器重启时缓冲区未 flush
# 解决：确保应用的日志输出没有缓冲（Python: PYTHONUNBUFFERED=1, Go: os.Stderr.Sync()）
```
