
# 1. 事件处理流水线

```
Pipeline Worker Thread 1:
  Input Plugin ──→ Filter Chain ──→ Output Plugin
       │               │               │
  ┌────▼────┐    ┌─────▼─────┐    ┌────▼────┐
  │  Queue  │───→│  Grok     │───→│   ES    │
  │ (Memory │    │  Mutate   │    │  Kafka  │
  │  or PQ) │    │  GeoIP    │    │  File   │
  └─────────┘    └───────────┘    └─────────┘

Pipeline Worker Thread 2: (同上，并行处理)
Pipeline Worker Thread N: (同上)
```

# 2. 线程模型

```yaml
# logstash.yml
pipeline.workers: 4            # Worker 线程数（默认 CPU 核数）
pipeline.batch.size: 125       # 每批处理事件数
pipeline.batch.delay: 50       # 批处理等待时间(ms)
```

```
Input Thread
  │
  │ 从数据源读取 → 封装为 Event → 放入 Queue
  │
  ▼
┌──────────────────────────────────────────────┐
│                  Queue                        │
│  (Memory Queue 或 Persistent Queue)           │
│  ┌──────┬──────┬──────┬──────┐              │
│  │Batch │Batch │Batch │Batch │              │
│  └──────┴──────┴──────┴──────┘              │
└──────────────────────────────────────────────┘
  │         │         │         │
  ▼         ▼         ▼         ▼
Worker 1  Worker 2  Worker 3  Worker 4
  │         │         │         │
  ▼         ▼         ▼         ▼
Filter → Output   (每个 Worker 独立处理)
```

# 3. Event 模型

```ruby
# Logstash Event 结构
{
  "@timestamp" => 2024-01-15T10:30:00.000Z,
  "@version" => "1",
  "message" => "192.168.1.1 - - [15/Jan/2024:10:30:00 +0800] GET /api/user HTTP/1.1 200 45",
  "host" => "web-server-01",
  "path" => "/var/log/nginx/access.log"
}

# 经过 Grok + GeoIP 处理后:
{
  "@timestamp" => 2024-01-15T02:30:00.000Z,
  "clientip" => "192.168.1.1",
  "method" => "GET",
  "request" => "/api/user",
  "httpversion" => "1.1",
  "response" => 200,
  "bytes" => 45,
  "geoip": {
    "country_name" => "China",
    "city_name" => "Beijing",
    "location" => { "lat" => 39.9, "lon" => 116.4 }
  }
}
```

# 4. Queue 机制

| Queue 类型 | 持久化 | 性能 | 适用场景 |
|-----------|--------|------|---------|
| **Memory**（默认） | ✗ 重启丢失 | 极高 | 开发/可容忍丢失 |
| **Persistent Queue** | ✓ 磁盘持久 | 中（有开销） | 生产环境 |

```yaml
# Queue 配置
queue.type: persisted           # persisted / memory
path.queue: /var/lib/logstash/queue
queue.max_bytes: 2gb            # PQ 最大磁盘
queue.checkpoint.writes: 1024   # 每 N 次写入做 checkpoint
```

## 4.1 Memory Queue

```
Memory Queue: 纯内存，最快
  - 大小 = pipeline.batch.size × workers
  - 背压机制: 队列满 → 输入暂停
```

## 4.2 Persistent Queue

```
PQ 结构:
  ┌──────────────────────────────────┐
  │ Page File 1 (64MB)              │
  │  [Event1][Event2]...[EventN]    │
  ├──────────────────────────────────┤
  │ Page File 2 (64MB)              │
  ├──────────────────────────────────┤
  │ ...                              │
  ├──────────────────────────────────┤
  │ Checkpoint File                  │
  │  已处理的 Page + Offset          │
  └──────────────────────────────────┘
```

# 5. 死信队列（DLQ）

```yaml
# DLQ 配置
dead_letter_queue.enable: true
dead_letter_queue.max_bytes: 1gb
path.dead_letter_queue: /var/lib/logstash/dead_letter_queue
```

```
Pipeline 处理失败的事件 → 自动写入 DLQ
  保留原始 Event + 失败原因 + 时间戳
  可用 `input { dead_letter_queue {} }` 重新处理
```

# 6. Plugin 加载

```
Logstash 启动:
  1. 读取 pipeline 配置
  2. 动态加载需要的 Plugin (Ruby Gems / Java JARs)
  3. 每个 Plugin 在独立隔离环境中运行
  4. Input Thread → Queue → Filter Workers → Output

Plugin 目录:
  logstash-core/lib/logstash/inputs/    → 内置 Input
  logstash-core/lib/logstash/filters/   → 内置 Filter
  vendor/bundle/jruby/*/gems/           → 已安装插件
```

```bash
# 插件管理
bin/logstash-plugin list                    # 列出已安装
bin/logstash-plugin install logstash-input-kafka
bin/logstash-plugin update logstash-output-elasticsearch
```
