
# 1. 基础题

## 1.1 Logstash 是什么？在 ELK 中扮演什么角色？

Logstash 是 ELK 中的数据收集和处理引擎。负责从多种来源采集数据（Input），进行解析和丰富（Filter），然后输出到目标存储（Output）。在 ELK 中承担 ETL（Extract-Transform-Load）角色。

## 1.2 Logstash 的管道模型是怎样的？

```
Input Plugin → Queue → Filter Chain → Output Plugin

Input: 从数据源采集 (Beats/Kafka/File/JDBC)
Queue: Memory Queue 或 Persistent Queue 缓冲
Filter: 解析和转换 (Grok/Mutate/GeoIP/Date)
Output: 写入目标 (ES/Kafka/File/S3)
```

## 1.3 Grok 和 Dissect 的区别？

| 维度 | Grok | Dissect |
|------|------|---------|
| 原理 | 正则表达式 | 分隔符定位 |
| 性能 | 中 | 高（5-10x faster） |
| 适用 | 复杂/不规则日志 | 分隔符明确的日志 |

# 2. 原理题

## 2.1 Persistent Queue 是如何工作的？

PQ 将事件序列化写入磁盘 Page File，通过 Checkpoint 标记已处理位置。崩溃重启后从 Checkpoint 恢复未处理的事件。默认每 1024 次写入做一次 Checkpoint（平衡性能与安全）。

## 2.2 Logstash 的线程模型是怎样的？

Input 线程读取数据 → 放入 Queue → N 个 Worker 线程（pipelines.workers）从 Queue 取批量事件 → 执行 Filter Chain → 发往 Output。Worker 数量 = CPU 核心数。

## 2.3 Dead Letter Queue 的作用？

当 Output 写入失败（如 ES 不可达），经过重试后仍失败的事件写入 DLQ（磁盘）。运维人员可用 `input { dead_letter_queue {} }` 重新处理。

# 3. 实践题

## 3.1 如何解析 Nginx 日志？

```ruby
input { beats { port => 5044 } }

filter {
  grok { match => { "message" => "%{COMBINEDAPACHELOG}" } }
  date { match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ] }
  geoip { source => "clientip" }
  mutate { remove_field => ["message", "timestamp"] }
}

output { elasticsearch { hosts => ["http://es:9200"] } }
```

## 3.2 Logstash 性能优化怎么做？

1. Grok 用 ^ 锚定，或改用 Dissect
2. 增大 `pipeline.batch.size`（250-500）
3. Workers = CPU 核心数
4. ES Output: flush_size=500, sniffing=false
5. 生产用 PQ + DLQ
6. JVM: G1GC, -Xms=-Xmx

## 3.3 Logstash vs Filebeat 怎么选？

- Filebeat：轻量采集，直接发 ES（日志格式已结构化）
- Logstash：重量处理，需要 Grok 解析非结构化日志
- 推荐：Filebeat → Logstash → ES（Filebeat 采集，Logstash 处理）

# 4. 架构题

## 4.1 Logstash 如何保证数据不丢失？

1. **Persistent Queue**：进程崩溃后从磁盘恢复
2. **Dead Letter Queue**：ES 不可达时暂存磁盘
3. **Output Retry**：写入失败自动重试
4. **Kafka Input**：Consumer Group offset 管理

## 4.2 什么场景用 Pipeline-to-Pipeline？

1. 同一机器上隔离不同类型数据处理（Nginx/App 独立 Pipeline）
2. 预处理 → 分发 → 多下游（一个 Beat 源分发到多个管道）
3. 资源隔离（高优先管道独立 Workers）

## 4.3 Logstash vs Fluentd 怎么选？

| 场景 | 推荐 |
|------|------|
| ELK 完整生态，复杂 ETL | Logstash |
| K8s 环境，轻量采集 | Fluentd / Fluent Bit |
| 高性能，资源敏感 | Vector |

混合架构：边缘 Fluent Bit → 中心 Logstash（各取所长）。
