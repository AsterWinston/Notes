
# 1. 什么是 Thanos

Thanos 是一个 **Prometheus 高可用和长期存储解决方案**，它基于 Prometheus 构建，通过 Sidecar 机制扩展 Prometheus 功能，提供**全局查询视图**、**无限数据保留**和**高可用性**。

Thanos 也是 CNCF 毕业项目，是目前最流行的 Prometheus 扩展方案之一。

# 2. 核心组件

```
                        +-------------+
                        |  Thanos     |
                        |  Query      |
                        +------+------+
                               |
          +--------------------+--------------------+
          |                    |                    |
    +-----v------+      +-----v------+     +------v-------+
    | Thanos     |      | Thanos     |     | Thanos       |
    | Sidecar    |      | Sidecar    |     | Store        |
    | (with      |      | (with      |     | Gateway      |
    | Prometheus)|      | Prometheus)|     | (Object Store|
    +-----+------+      +-----+------+     |  backed)     |
          |                    |            +--------------+
          |                    |                    |
          v                    v                    v
    +----------------------------------------------+
    |       Object Storage (S3/GCS/MinIO)          |
    +----------------------------------------------+
```

## 2.1 Sidecar

Sidecar 与 Prometheus 部署在同一 Pod/机器中：

- 将 Prometheus 的本地数据上传到对象存储
- 暴露 Store API 供 Thanos Query 查询
- 监控 Prometheus 健康状态

## 2.2 Query

无状态查询层，实现全局视图：

- 接收 PromQL 查询
- 向 Sidecar、Store Gateway 等后端发起查询
- 合并去重结果
- 实现时间戳去重（dedup）

## 2.3 Store Gateway

对象存储网关：

- 读取对象存储中的历史 Block
- 暴露 Store API
- 缓存元数据加速查询

## 2.4 Compactor

负责压缩和降采样对象存储中的数据：

- 合并小 Block 为大 Block
- 执行降采样（downsampling）
- 清理过期数据
- 修复损坏的 Block

## 2.5 Receive

接收 Remote Write 的写入层（可选组件）：

- 直接接收 Prometheus Remote Write
- 写入本地 TSDB
- 数据上传到对象存储
- 适用于 Push 模式或不部署 Sidecar 的场景

## 2.6 Rule

告警评估组件（可选）：

- 脱离 Prometheus 独立评估告警规则
- 基于全局数据做告警
- 支持 Recording Rule

# 3. 部署模式

## 3.1 模式一：Sidecar + 对象存储

最常用的部署方式，每个 Prometheus 实例旁部署一个 Sidecar。

```yaml
# docker-compose 示例
version: '3.8'
services:
  prometheus:
    image: prom/prometheus:v2.53.0
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=2d'  # 本地仅保留少量数据

  thanos-sidecar:
    image: quay.io/thanos/thanos:v0.35.0
    command:
      - 'sidecar'
      - '--tsdb.path=/prometheus'
      - '--prometheus.url=http://prometheus:9090'
      - '--objstore.config-file=/etc/thanos/objstore.yml'
    depends_on:
      - prometheus

  thanos-query:
    image: quay.io/thanos/thanos:v0.35.0
    command:
      - 'query'
      - '--store=thanos-sidecar:10901'
      - '--store=thanos-store:10901'
    ports:
      - "10902:10902"

  thanos-store:
    image: quay.io/thanos/thanos:v0.35.0
    command:
      - 'store'
      - '--objstore.config-file=/etc/thanos/objstore.yml'

  thanos-compactor:
    image: quay.io/thanos/thanos:v0.35.0
    command:
      - 'compact'
      - '--objstore.config-file=/etc/thanos/objstore.yml'
      - '--wait'
```

## 3.2 模式二：Receive 模式（无 Sidecar）

适用于无法部署 Sidecar 的场景（如托管 Prometheus）。

```
Prometheus -- Remote Write --> Thanos Receive --> Object Storage
                                                    |
                                              Thanos Query <-- Grafana
                                                    |
                                              Thanos Store
```

# 4. 对象存储配置

```yaml
# objstore.yml
type: S3
config:
  bucket: "thanos-storage"
  endpoint: "s3.amazonaws.com"
  region: "us-east-1"
  access_key: "AKIA..."
  secret_key: "..."
  insecure: false
  sigv4: false
```

## 4.1 支持的存储后端

| 后端 | type 值 | 说明 |
|------|---------|------|
| AWS S3 | `S3` | 兼容 S3 的存储 |
| GCS | `GCS` | Google Cloud Storage |
| Azure Storage | `AZURE` | Azure Blob Storage |
| Swift | `SWIFT` | OpenStack Swift |
| COS | `COS` | 腾讯云对象存储 |
| OSS | `OSS` | 阿里云对象存储 |
| MinIO | `S3` | 本地 S3 兼容存储 |
| Filesystem | `FILESYSTEM` | 本地文件系统（测试用） |

# 5. 全局查询与去重

## 5.1 查询流程

```
客户端查询: sum(rate(http_requests_total[5m]))

Thanos Query:
  1. 将查询广播到所有 Store API 后端
  2. 每个后端返回部分结果
  3. Query 合并结果
  4. 按时间戳去重（dedup）: 多副本保留最新值
  5. 执行 PromQL 聚合
  6. 返回最终结果
```

## 5.2 去重标记

```bash
thanos query \
  --store=sidecar-1:10901 \
  --store=sidecar-2:10901 \
  --query.replica-label=replica  # 指定去重标签
```

- `replica` 标签用于标识不同副本
- Thanos Query 自动按 `replica` 标签去重
- 同一时间戳的多副本数据，保留任意一个（值应相同）

# 6. 降采样（Downsampling）

降采样是 Thano 的 Compactor 提供的功能，将高分辨率数据降为低分辨率数据以加速长时间范围查询。

## 6.1 降采样级别

```bash
thanos compact \
  --retention.resolution-raw=30d       # 原始精度保留 30 天
  --retention.resolution-5m=180d       # 5 分钟精度保留 180 天
  --retention.resolution-1h=10y        # 1 小时精度保留 10 年
```

| 分辨率 | 保留时间 | 说明 |
|--------|----------|------|
| raw（原始） | 30 天 | 原始 Prometheus 数据 |
| 5m | 180 天 | 5 分钟聚合数据 |
| 1h | 10 年 | 1 小时聚合数据（长期归档） |

## 6.2 降采样规则

- Raw → 5m：保留每个 5 分钟窗口内的 min/max/avg/count
- 5m → 1h：保留每个 1 小时窗口内的 min/max/avg/count
- 使用 `aggregated` 标签标识降采样数据

## 6.3 查询行为

```promql
# 查询最近 1 天数据 → 使用 raw 数据
rate(http_requests_total[5m])

# 查询最近 1 年数据 → 自动使用降采样数据
rate(http_requests_total[5m] offset 1y)

# Thanos Query 会自动选择合适分辨率的数据
```

# 7. Thanos Rule

```bash
thanos rule \
  --data-dir /data/thanos-rule \
  --rule-file=/etc/thanos/rules/alerts.yml \
  --query=thanos-query:10902 \
  --alertmanagers.url=http://alertmanager:9093
```

## 7.1 与 Prometheus Rule 对比

| 特性 | Prometheus Rule | Thanos Rule |
|------|----------------|-------------|
| 数据范围 | 本地数据 | 全局数据 |
| 高可用 | 主备模式 | 可多副本运行 |
| 存储 | 本地 TSDB | 对象存储 |
| 依赖 | 无 | 需要 Query 组件 |

# 8. Thanos Query Frontend

查询前端组件，提供缓存和查询分片功能：

```bash
thanos query-frontend \
  --query-frontend.downstream-url=http://thanos-query:10902 \
  --query-range.split-interval=24h \
  --query-range.max-retries-per-request=5 \
  --labels.split-interval=24h \
  --labels.max-retries-per-request=5
```

# 9. 监控 Thanos

```promql
# Sidecar 数据上传状态
rate(thanos_sidecar_bucket_operations_total[5m])

# Store 网关查询延迟
thanos_store_bucket_last_successful_upload_time

# Query 查询延迟
thanos_query_queries_duration_seconds

# Compactor 运行状态
thanos_compact_halted  # 是否停止
thanos_compact_garbage_collected_total

# Store 门控查询
rate(thanos_store_bucket_touched_series_count[5m])
```

# 10. 常见架构模式

## 10.1 单数据中心

```
Prometheus + Sidecar → S3 → Thanos Store + Thanos Query → Grafana
```

## 10.2 多数据中心

```
DC1: Prometheus + Sidecar ─┐
                            ├── Thanos Query (Global) → Grafana
DC2: Prometheus + Sidecar ─┘
```

## 10.3 与 Kubernetes 集成

```yaml
# Prometheus Operator + Thanos
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: thanos-demo
spec:
  thanos:
    version: v0.35.0
    objectStorageConfig:
      key: thanos.yaml
      name: thanos-objstore-config
  retention: 2d
```
