
# 1. 管道设计原则

```
✓ DO:
  - 一个源一条管道：Nginx / App / Syslog 分开
  - 先过滤再处理: if [type] == "nginx" 减少不必要的 Grok
  - 尽可能用 Dissect 代替 Grok（5-10x 性能）
  - 输出到不同索引：nginx-*, app-*, syslog-*
  - 给字段起有意义的名字

✗ DON'T:
  - 不要所有规则堆在一个管道
  - 不要在 Filter 中使用多个 Grok 匹配同一 message
  - 不要让 Grok 无锚定匹配（^ 开头）
  - 不要在生产用 stdout { codec => rubydebug }
```

# 2. Grok vs Dissect 选择

```
分隔符明确 + 结构固定?
  ├── 是 → Dissect（快 10 倍！）
  └── 否 → Grok（用 ^ 锚定）

示例:
  "2024-01-15 10:30:00 INFO com.example.Service - user created"
  → Dissect: "%{ts} %{level} %{class} - %{msg}"

  "192.168.1.1 - - [15/Jan/2024:10:30:00 +0800] \"GET /api/user HTTP/1.1\" 200 45"
  → Grok: COMBINEDAPACHELOG
```

# 3. 错误处理

```ruby
filter {
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }
    tag_on_failure => ["_grokparsefailure"]
  }

  # 仅处理成功的事件
  if "_grokparsefailure" not in [tags] {
    date { match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ] }
  }

  # 失败事件单独处理
  if "_grokparsefailure" in [tags] {
    mutate {
      add_tag => ["unparsed"]
      add_field => { "original_message" => "%{message}" }
    }
  }
}

output {
  if "unparsed" in [tags] {
    file {
      path => "/var/log/logstash/unparsed.log"
    }
  }
}
```

# 4. 配置管理

```bash
logstash/
  ├── config/
  │     ├── pipelines.yml           # 管道定义
  │     ├── logstash.yml            # 全局配置
  │     └── jvm.options             # JVM 参数
  ├── conf.d/
  │     ├── beats-input.conf        # Input
  │     ├── nginx-filter.conf       # Nginx 处理
  │     ├── app-filter.conf         # 应用处理
  │     └── es-output.conf          # Output
  └── patterns/
        └── custom.grok             # 自定义 Grok
```

```bash
# 配置校验（CI/CD）
bin/logstash -f /etc/logstash/conf.d --config.test_and_exit

# 版本管理
git init && git add . && git commit -m "Logstash config v1.0"
```

# 5. 生产配置

```yaml
# logstash.yml
pipeline.workers: 8
pipeline.batch.size: 250
pipeline.batch.delay: 25
queue.type: persisted
queue.max_bytes: 4gb
queue.checkpoint.writes: 1024
dead_letter_queue.enable: true
dead_letter_queue.max_bytes: 2gb
config.reload.automatic: false       # 生产建议手动重启
log.level: warn
```

# 6. 监控告警

```
必须告警:
  □ queue.events_count > 10000 (管道处理能力不足)
  □ DLQ 有新数据 (下游异常)
  □ events.out < events.in × 0.9 (10% 丢弃)
  □ JVM heap > 85%
  □ ES output 写入错误率 > 0
  □ Grok parse failure 率 > 5%

告警工具:
  - Kibana Alerting (ES query rule)
  - Prometheus + AlertManager (Logstash Exporter)
  - Script: curl localhost:9600/_node/stats | jq .queue.events_count
```

# 7. DON'T

```
✗ 不要在 Filter 中重复解析 (先 Grok 再 JSON 再 Grok)
✗ 不要用多个 GREEDYDATA 捕获
✗ 不要在循环中使用 ruby filter (性能杀手)
✗ 不要忘记设置 ES ILM (索引会无限增长)
✗ 不要用 Memory Queue 跑生产
✗ 不要让 pipeline.batch.size > 500 (内存风险)
✗ 不要忽略 DLQ 中的死信数据
```
