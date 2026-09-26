
# 1. 为什么需要远程存储

Prometheus 本地 TSDB 虽然高效，但受限于单节点资源，存在以下限制：

- **数据保留有限**：通常保留 15-30 天
- **存储容量受限**：受单机磁盘大小限制
- **高可用性差**：单节点故障导致数据丢失
- **全局视图缺失**：跨数据中心的联合查询困难

远程存储解决了这些问题，提供**长期存储**、**更大容量**和**高可用性**。

# 2. 远程存储架构

```
+----------------+     Remote Write      +------------------+
|  Prometheus A  |---------------------->|  Remote Storage  |
+----------------+   HTTP POST /write    |  (Thanos/        |
|  Prometheus B  |---------------------->|   VictoriaMetrics|
+----------------+                       |   /Cortex/       |
|  Prometheus C  |---------------------->|   InfluxDB)      |
+----------------+                       +------------------+
                                                    |
                                                    | Remote Read
                                                    v
                                            +------------------+
                                            |  Grafana         |
                                            +------------------+
```

# 3. 远程写（Remote Write）

## 3.1 配置

```yaml
remote_write:
  - url: "http://remote-storage:9201/api/v1/write"
    name: "primary-storage"
    # 基本认证
    basic_auth:
      username: "admin"
      password: "password"
    # 写入队列配置
    queue_config:
      capacity: 2500           # 队列容量
      max_samples_per_send: 500  # 每批最多样本数
      max_shards: 200          # 最大并发分片数
      min_shards: 1            # 最小分片数
      min_backoff: 30ms        # 重试最小等待
      max_backoff: 100ms      # 重试最大等待

  - url: "http://another-storage:9201/api/v1/write"
    # 发送 TLS
    tls_config:
      cert_file: /etc/prometheus/cert.pem
      key_file: /etc/prometheus/key.pem
    # 添加标签
    write_relabel_configs:
      - source_labels: [__name__]
        regex: "expensive_metric.*"
        action: drop
```

## 3.2 队列配置详解

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `capacity` | 2500 | 每个分片的队列容量 |
| `max_samples_per_send` | 500 | 每次发送的最大样本数 |
| `max_shards` | 200 | 最大并发分片数（自动调整） |
| `min_shards` | 1 | 最小分片数 |
| `max_backoff` | 5s | 最大重试间隔 |
| `min_backoff` | 30ms | 最小重试间隔 |

## 3.3 背压与重试机制

```
成功 → 减少 backoff
失败 → 增大 backoff（指数退避）
持续失败 → 增加分片数（提高吞吐量）
全部失败 → 缓存到磁盘（最终一致性）
```

# 4. 远程读（Remote Read）

## 4.1 配置

```yaml
remote_read:
  - url: "http://remote-storage:9201/api/v1/read"
    read_recent: true          # 是否也读取本地数据
    required_matchers:
      job: "prometheus"        # 只读取匹配的数据
    basic_auth:
      username: "admin"
      password: "password"
```

## 4.2 read_recent 说明

- `true`：查询时同时查本地和远程存储（推荐）
- `false`：只查远程存储（适用于完全替代本地存储的场景）

# 5. 后端存储对比

| 存储 | 优点 | 缺点 |
|------|------|------|
| Thanos | 完全兼容 PromQL，全局视图，无限存储 | 组件多，部署复杂 |
| VictoriaMetrics | 单二进制，高性能，资源占用低 | 部分 PromQL 不兼容 |
| Cortex | 多租户，水平扩展 | 部署复杂 |
| InfluxDB | 成熟稳定 | 需适配器转换协议 |
| TimescaleDB | 支持 SQL | 不兼容 PromQL，需适配器 |
| M3DB | 水平扩展，多租户 | 运维复杂 |

# 6. 适配器方案

对于不支持 Remote Write/Read 协议的存储后端，可以使用适配器。

## 6.1 InfluxDB 适配器

```bash
# prometheus-adapter for InfluxDB
docker run -d \
  -p 9201:9201 \
  -e INFLUXDB_ADDR=http://influxdb:8086 \
  -e INFLUXDB_DB=prometheus \
  prom/influxdb-adapter:latest
```

```yaml
remote_write:
  - url: "http://adapter:9201/write"

remote_read:
  - url: "http://adapter:9201/read"
```

# 7. 数据流控

## 7.1 过滤写入指标

```yaml
remote_write:
  - url: "http://remote-storage:9201/write"
    write_relabel_configs:
      # 只写入高价值指标
      - source_labels: [__name__]
        regex: "node_.*|http_request.*"
        action: keep
      # 删除高基数指标
      - source_labels: [__name__]
        regex: "node_network_.*"
        action: drop
```

## 7.2 限速写入

```yaml
remote_write:
  - url: "http://remote-storage:9201/write"
    queue_config:
      max_samples_per_send: 100   # 降低批大小限制吞吐
      max_shards: 10              # 限制并发数
```

# 8. 故障处理

## 8.1 写入失败

```promql
# 监控 Remote Write 失败次数
rate(prometheus_remote_storage_failed_samples_total[5m])

# 监控 Remote Write 延迟
prometheus_remote_storage_sent_batch_duration_seconds

# 监控队列情况
prometheus_remote_storage_shards  # 当前分片数
prometheus_remote_storage_queue_length  # 当前队列长度
prometheus_remote_storage_samples_in_total  # 写入总样本数
prometheus_remote_storage_dropped_samples_total  # 丢弃样本数
```

## 8.2 恢复策略

```
1. 临时故障：分片自动调整，队列缓存数据
2. 长时间故障：数据在 WAL 中保留（受 WAL 大小限制）
3. 严重故障：超出容量时数据被丢弃（丢弃计数增加）
```

# 9. 双写策略

## 9.1 同时写入两份存储

```yaml
remote_write:
  - url: "http://thanos-receive:19291/api/v1/write"
    name: "thanos"
  - url: "http://victoria-metrics:8428/api/v1/write"
    name: "victoria"
```

## 9.2 迁移策略

```
1. 同时写入新旧两套存储（双写）
2. 验证新存储数据完整性
3. 切换查询到新存储
4. 移除旧存储写入配置
```

# 10. 性能考量

## 10.1 写入性能

- Remote Write 默认使用 Snappy 压缩，带宽开销小
- 每个 Prometheus 实例建议写入不超过 50 万 samples/s
- 网络延迟增加会减少分片的有效性

## 10.2 读取性能

- Remote Read 不适合频繁的低延迟查询
- 建议使用本地存储处理常见 Dashboard 查询
- Grafana 查询时配置合理的时间范围和 Step
