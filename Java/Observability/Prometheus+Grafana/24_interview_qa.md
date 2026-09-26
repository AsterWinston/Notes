
# 1. 基础概念

## 1.1 Q1：Prometheus 是什么？它的核心设计理念是什么？

Prometheus 是一个开源的系统监控和告警工具集，核心设计理念包括：

- **Pull 模型**：Server 主动拉取指标，而非 Agent 推送
- **多维数据模型**：基于时间序列，通过 Metric Name + Key/Value 标签定义
- **PromQL 查询语言**：强大的函数式查询语言
- **自研 TSDB**：高效的时序数据本地存储
- **告警分离**：Prometheus Server 负责评估规则，Alertmanager 负责通知

## 1.2 Q2：Prometheus 的数据模型是怎样的？

```
<metric_name>{<label_name>=<label_value>, ...}
```

- **Metric Name**：标识一类度量，如 `http_requests_total`
- **Labels**：维度标识，Key/Value 对，如 `{method="GET", status="200"}`
- **Samples**：数据点，包含 Timestamp（毫秒精度）和 Value（float64）
- **Time Series**：由 Metric Name + Labels 唯一标识的样本序列

## 1.3 Q3：Prometheus 的四种 Metric 类型是什么？

| 类型 | 特点 | 适用场景 |
|------|------|----------|
| Counter | 只增不减（可重置） | 请求计数、错误计数 |
| Gauge | 可增可减 | 内存使用、温度、当前连接数 |
| Histogram | 分桶统计，服务端算分位数 | 延迟分布、响应大小 |
| Summary | 预计算分位数，客户端算分位数 | 不需要聚合的分位数场景 |

## 1.4 Q4：Prometheus 为什么不使用 Push 模型而使用 Pull 模型？

| 对比 | Pull | Push |
|------|------|------|
| 故障检测 | 采集失败可立即感知目标是否存活 | 需要心跳机制 |
| 控制权 | Server 控制采集频率，避免被淹没 | Agent 控制发送频率 |
| 安全性 | 只需 Server 访问目标端口 | 需要 Agent 访问 Server |
| 扩展性 | 通过服务发现自动发现目标 | 需要注册中心 |

可访问性限制的场景可通过 Pushgateway 或 Remote Write 绕过。

# 2. PromQL

## 2.1 Q5：rate() 和 irate() 的区别？

| 特性 | rate() | irate() |
|------|--------|---------|
| 计算方式 | 窗口首尾差值 ÷ 总时间 | 最后两个点的差值 ÷ 间隔 |
| 平滑度 | 平滑，适合长期趋势 | 毛刺多，适合短时尖峰 |
| Counter 重置 | 自动处理 | 自动处理 |
| 推荐场景 | 告警、Dashboard | 短时突发检测 |

## 2.2 Q6：increase() 与 rate() 的关系？

`increase(metric[5m])` 等价于 `rate(metric[5m]) * 300`（时间窗口 5 分钟的秒数）。二者在 PromQL 内部实现逻辑相同，只是 rate 返回每秒平均速率，increase 返回窗口内的总增量。

## 2.3 Q7：histogram_quantile() 如何工作？有什么注意事项？

函数基于 Histogram 的桶计数通过**线性插值**估算分位数。

注意事项：

1. 必须配合 `rate()` 使用（对 `_bucket` 取 rate 得到有效分布）
2. Bucket 设计影响精度：在典型值附近应设置更密集的 Bucket
3. `+Inf` 桶必须存在
4. 分位数在数据稀疏时可能不准确

```promql
# 正确用法
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# Bucket 设计建议
buckets: [0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10, +Inf]
```

## 2.4 Q8：什么是高基数（Cardinality）？为什么需要避免？

高基数指的是某个标签的值组合过多，导致时间序列数量爆炸。

**影响：**

- 内存占用增加（每个序列需要维护索引和 Chunk）
- 查询性能下降（需要扫描更多序列）
- 磁盘空间膨胀

**示例：**

```promql
# 高基数：标签包含唯一值
http_requests_total{user_id="12345"}  # user_id 有 100 万用户 → 100 万序列

# 低基数：标签取值有限
http_requests_total{tier="premium"}  # tier 只有取 3-5 个值 → 3-5 序列
```

# 3. 架构与部署

## 3.1 Q9：Prometheus 的本地存储（TSDB）如何工作？

- **2 小时 Block**：每 2 小时生成一个不可变的 Block
- **Gorilla 压缩**：时间戳 Delta-of-Delta 编码 + 值 XOR 编码，平均 1.37 字节/样本
- **倒排索引**：基于标签的倒排索引加速查询
- **WAL**：预写日志确保数据持久性
- **Compaction**：后台合并小 Block 减少文件数
- **Retention**：基于时间或大小的数据保留策略

## 3.2 Q10：如何实现 Prometheus 的高可用？

| 方案 | 说明 |
|------|------|
| 双实例 | 两个 Prometheus 采集相同目标，通过负载均衡或 Thanos 去重 |
| 联邦集群 | 分层结构，从下到上聚合数据 |
| Thanos | Sidecar 上传对象存储 + Query 全局查询 + 去重 |
| VictoriaMetrics | 高性能兼容 Prometheus 的单节点/集群方案 |

## 3.3 Q11：Alertmanager 的高可用如何实现？

Alertmanager 通过 **Gossip 协议**实现集群：

```bash
./alertmanager \
  --cluster.listen-address=0.0.0.0:9094 \
  --cluster.peer=am1:9094,am2:9094,am3:9094
```

特性：
- 告警去重：一条告警只发送一次通知
- 数据同步：静默、抑制等配置自动同步
- 故障转移：节点故障不影响告警发送

# 4. 工程实践

## 4.1 Q12：如何处理高基数问题？

1. **发现**：使用 Grafana 高基数分析 Dashboard 或查询 `count by (__name__) ({__name__=~".+"})`
2. **控制**：
   - Relabel 中删除高基数标签
   - 使用路径模板替代动态参数
   - 禁用不必要的 Collector
3. **预防**：
   - 标签值组合数控制在 10 万以内
   - 指标名 + 标签数不超过 10 个
   - 不使用用户 ID、Session ID 等作为标签

## 4.2 Q13：如何设计指标和标签？

**指标命名：**

```yaml
<namespace>_<name>_<unit>[_total]
```

**标签设计原则：**

- 标签值有限（如 method、status、job）
- 标签有业务意义
- 避免高基数（如 user_id、ip、email）
- 每个指标标签数 3-8 个

## 4.3 Q14：Grafana Dashboard 设计有哪些最佳实践？

1. 使用**模板变量**实现交互式过滤
2. 聚焦**黄金信号**：延迟、流量、错误、饱和度
3. 使用**时间范围选择器**提供灵活的时间窗口
4. Panel 数量控制在 20 个以内
5. 使用 Row 按逻辑分组
6. 添加 Panel 注释说明异常行为

## 4.4 Q15：什么是 Recording Rule？什么时候应该使用？

Recording Rule 是 Prometheus 的**预计算机制**，将复杂 PromQL 表达式定期计算结果存储为新的时间序列。

**使用场景：**

```yaml
# 1. 复杂且频繁查询的表达式
- record: instance:cpu_usage:avg5m
  expr: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100)

# 2. Dashboard 高频查询指标
- record: job:request_duration:p99
  expr: histogram_quantile(0.99, sum(rate(...)))

# 3. 跨多级聚合的中间结果
- record: instance:http_requests:rate5m
  expr: rate(http_requests_total[5m])
- record: job:http_requests:rate5m
  expr: sum(instance:http_requests:rate5m) by (job)
```

# 5. 监控策略

## 5.1 Q16：什么是 RED 方法？

RED 方法（源自 Google SRE 实践）：

| 指标 | 含义 | 示例 |
|------|------|------|
| Rate | 请求速率 | `rate(http_requests_total[5m])` |
| Errors | 错误率 | `rate(http_requests_total{status=~"5.."}[5m])` |
| Duration | 延迟 | `histogram_quantile(0.99, rate(...))` |

## 5.2 Q17：什么是 USE 方法？

USE 方法（源自 Brendan Gregg）用于**资源监控**：

| 指标 | 含义 | 示例 |
|------|------|------|
| Utilization | 资源使用率 | CPU 使用率、磁盘使用率 |
| Saturation | 资源饱和度 | 队列长度、Load Average |
| Errors | 错误计数 | 磁盘 IO 错误、网络错误 |

## 5.3 Q18：如何监控 Kubernetes 集群？

监控体系分为四层：

1. **节点层**：Node Exporter（CPU、内存、磁盘、网络）
2. **容器层**：cAdvisor 指标（容器 CPU/内存/网络）
3. **K8s 对象层**：kube-state-metrics（Pod/Deployment/Node 状态）
4. **控制面层**：API Server / etcd / Controller Manager 指标

## 5.4 Q19：如何区分 Exporter 和 Pushgateway 的使用场景？

| 组件 | 适用场景 | 不适用场景 |
|------|----------|------------|
| Exporter | 长期运行的服务 | 批处理任务 |
| Pushgateway | 批处理任务、Cron Job | 长期运行的服务、大量指标 |

# 6. 高级话题

## 6.1 Q20：Prometheus 如何保证数据的可靠性？

- **WAL**：样本先写 WAL 再写入内存，崩溃后可重放 WAL 恢复数据
- **Checkpoint**：定期对 WAL 做检查点，减少恢复时间
- **Block 不可变性**：落盘后的 Block 只读，避免写入中损坏
- **Checksum**：Block 中每个 Chunk 都有校验和

## 6.2 Q21：什么是 OpenMetrics 标准？

OpenMetrics 是 CNCF 沙箱项目，旨在标准化 Prometheus 的指标暴露格式。它是 Prometheus 文本格式的超集，支持：

- 明确的 Metric 类型定义
- 时间戳支持
- 示例（Exemplar）支持

## 6.3 Q22：Thanos 的核心组件及其作用？

| 组件 | 作用 |
|------|------|
| Sidecar | 与 Prometheus 同部署，上传数据到对象存储 |
| Query | 全局查询层，合并去重多个 Store 后端 |
| Store Gateway | 读取对象存储中的历史 Block |
| Compactor | 压缩、降采样、清理对象存储中的数据 |
| Receive | 接收 Remote Write，替代 Sidecar |
| Rule | 基于全局数据的告警评估 |

## 6.4 Q23：Prometheus 的联邦集群（Federation）如何工作？

联邦允许一个 Prometheus Server 从另一个 Prometheus Server 采集聚合指标：

```yaml
scrape_configs:
  - job_name: 'federate'
    scrape_interval: 15s
    honor_labels: true
    metrics_path: '/federate'
    params:
      'match[]':
        - '{job="node"}'
        - '{__name__=~"job:.*"}'
    static_configs:
      - targets:
        - 'source-prometheus-1:9090'
        - 'source-prometheus-2:9090'
```

## 6.5 Q24：Prometheus 的 Remote Write 协议是如何工作的？

1. Prometheus 按 `queue_config` 配置将样本缓冲到分片中
2. 每个分片按批次将 Snappy 压缩的 Protobuf 数据发送到远程端点
3. 远程端点解压、解码、写入后端存储
4. 失败时自动重试（指数退避），成功时调整分片数以优化吞吐

## 6.6 Q25：如何调优 Prometheus 的查询性能？

1. **使用 Recording Rule** 预计算复杂查询
2. **缩小查询范围**：使用标签过滤、合理时间范围
3. **避免高基数查询**：先聚合再运算
4. **使用 Grafana 模板变量**减少查询量
5. **增加硬件资源**：CPU、内存、SSD 磁盘
6. **优化 `scrape_interval`**：变化慢的指标加大采集间隔
7. **减少不必要的指标**：在 Relabel 中丢弃
