
# 1. 概述

Logstash 是 Elastic Stack 的**数据处理管道**，负责从多种来源采集数据，进行转换和丰富，然后输出到目标存储。是"ELK"中的"L"。

```
┌──────────────────────────────────────────────────────┐
│                   Logstash Pipeline                   │
│                                                       │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐       │
│  │  Input   │───→│  Filter  │───→│  Output  │       │
│  │ (输入)    │    │  (过滤)   │    │ (输出)    │       │
│  └──────────┘    └──────────┘    └──────────┘       │
│       │                │                │             │
│  Filebeat          Grok             Elasticsearch    │
│  Kafka             Mutate           Kafka            │
│  Redis             JSON             File             │
│  JDBC              GeoIP            S3               │
│  HTTP              Date             Email            │
└──────────────────────────────────────────────────────┘
```

# 2. ELK 中的角色

```
传统 ELK 架构:
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Filebeat │───→│ Logstash │───→│Elasticsearch│──→│ Kibana   │
│ (采集)    │    │ (处理)    │    │  (存储)    │    │ (可视化)  │
└──────────┘    └──────────┘    └──────────┘    └──────────┘

Logstash 在中间负责：
  - 数据解析（非结构化 → 结构化）
  - 数据丰富（GeoIP、字典映射）
  - 数据过滤（丢弃无效数据）
  - 数据路由（不同数据到不同索引）
```

# 3. 核心能力

| 能力 | 说明 | 示例 |
|------|------|------|
| **多源采集** | 200+ Input 插件 | Beats/Kafka/JDBC/Redis/File |
| **数据转换** | 50+ Filter 插件 | Grok 正则/Mutate 字段/GeoIP |
| **多目标输出** | 50+ Output 插件 | ES/Kafka/S3/File |
| **持久化** | Persistent Queue + DLQ | 防止数据丢失 |
| **多管道** | 一进程多管道 | 按数据流分离处理 |

# 4. vs Fluentd vs Fluent Bit vs Vector

| 维度 | Logstash | Fluentd | Fluent Bit | Vector |
|------|----------|---------|-----------|--------|
| **语言** | JRuby (JVM) | C + Ruby | C | Rust |
| **内存占用** | 高 (JVM 500MB+) | 中 (~50MB) | 极低 (~5MB) | 低 (~20MB) |
| **插件生态** | ★★★★★ (200+) | ★★★★☆ | ★★★☆☆ | ★★★☆☆ |
| **处理能力** | 高（重量处理） | 中 | 低（轻量） | 高 |
| **吞吐量** | 中（JVM限制） | 高 | 极高 | 极高 |
| **ELK 集成** | ★★★★★ 原生 | ★★★☆☆ | ★★★★☆ | ★★★☆☆ |
| **适用场景** | 重量 ETL | 通用采集 | 边缘/容器 | 高性能管道 |

**选型建议**：重量处理和复杂 ETL 用 Logstash；轻量采集和边缘设备用 Fluent Bit；K8s 环境用 Fluentd/Vector。

# 5. 适用与不适用场景

```
✓ 适用：
  - 非结构化日志解析（Nginx/Apache/自定义日志 → JSON）
  - 数据丰富和转换（GeoIP、字段重命名、类型转换）
  - 多源数据汇聚后分类输出到不同索引
  - Kafka → ES 的数据管道
  - 定时数据库查询同步到 ES

✗ 不适用：
  - 简单日志采集（直接用 Filebeat → ES）
  - 边缘/容器环境（资源开销大，用 Fluent Bit）
  - 超低延迟场景（JVM GC 可能引发延迟抖动）
  - 只需采集不需处理的场景（Filebeat 直接输出）
```

# 6. Logstash 8.x 变化

| 特性 | 说明 |
|------|------|
| **Java Execution Engine** | 8.0 默认 Java 引擎（弃用 Ruby），管道执行性能提升 |
| **ECS Compatibility** | 默认启用 Elastic Common Schema 字段兼容 |
| **Security ON by Default** | 8.x 默认开启安全，需配置 ES 认证 |
| **Pipeline-to-Pipeline** | 管道间通信（进程内高效传输） |
| **Native Plugin 增加** | 更多 Java 原生插件替换 Ruby 插件 |

# 7. 一个最小的 Logstash 管道

```ruby
# logstash.conf
input {
  stdin { }
}

filter {
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }
  }
  date {
    match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]
  }
  geoip {
    source => "clientip"
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "nginx-access-%{+YYYY.MM.dd}"
  }
  stdout { codec => rubydebug }
}
```

```bash
# 运行
bin/logstash -f logstash.conf
```
