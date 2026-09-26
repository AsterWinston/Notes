
# 1. ELK 架构

```
┌──────────────────────────────────────────────────────┐
│                     ELK Stack                         │
│                                                       │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐        │
│  │ Filebeat │   │ Logstash │   │   APM    │        │
│  │ (采集)    │   │ (处理)    │   │  Agent   │        │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘        │
│       │              │              │                │
│       └──────────────┼──────────────┘                │
│                      ▼                                │
│              ┌──────────────┐                         │
│              │ Elasticsearch│ ← 存储 + 搜索 + 分析     │
│              └──────┬───────┘                         │
│                     │                                 │
│              ┌──────▼───────┐                         │
│              │    Kibana    │ ← 可视化                │
│              └──────────────┘                         │
└──────────────────────────────────────────────────────┘
```

# 2. Filebeat → ES

```yaml
# filebeat.yml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/app/*.log
    fields:
      app: order-service
      env: production

output.elasticsearch:
  hosts: ["es1:9200", "es2:9200", "es3:9200"]
  username: "beats_system"
  password: "${BEATS_PASSWORD}"
  index: "app-logs-%{+yyyy.MM.dd}"
```

# 3. Logstash 数据处理

```ruby
# logstash.conf
input {
  beats { port => 5044 }
}

filter {
  grok {
    match => { "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} %{GREEDYDATA:message}" }
  }
  date { match => ["timestamp", "ISO8601"] }
  mutate {
    remove_field => ["@version", "host"]
    add_field => { "processed_by" => "logstash" }
  }
}

output {
  elasticsearch {
    hosts => ["es:9200"]
    index => "processed-logs-%{+yyyy.MM.dd}"
  }
}
```

# 4. Kibana

## 4.1 Discover — 日志检索

```
Kibana → Discover
  - 选择 Index Pattern: app-logs-*
  - KQL 搜索: level:ERROR AND app:order-service
  - 时间过滤
  - 字段过滤
```

## 4.2 Dashboard — 可视化

```
常用面板:
  - 日志量趋势（折线图）
  - 错误率（饼图）
  - Top 10 慢接口（数据表）
  - P99 响应时间（折线图）
  - 按服务分布（柱状图）
```

## 4.3 Canvas — 大屏展示

创建可视化大屏展示实时业务指标。

## 4.4 Lens — 拖拽分析

拖拽字段即可生成图表，无需编写聚合 JSON。

# 5. APM 集成

```bash
# APM Server
docker run -d --name apm-server \
  -p 8200:8200 \
  docker.elastic.co/apm/apm-server:8.11.0

# Java Agent 启动参数
java -javaagent:elastic-apm-agent.jar \
  -Delastic.apm.service_name=order-service \
  -Delastic.apm.server_url=http://apm:8200 \
  -jar app.jar
```

# 6. Grafana ES 数据源

```
Grafana → Configuration → Data Sources → Elasticsearch
  URL: http://es:9200
  Index: orders-*
  Time field: created_at
```

**Grafana vs Kibana**：
- Kibana：ES 原生，功能全
- Grafana：跨数据源聚合（Prometheus + ES 同一面板）
