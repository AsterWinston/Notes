
# 1. 管道安全

## 1.1 敏感字段脱敏

```ruby
filter {
  # 方式 1: 删除敏感字段
  if [phone] or [email] {
    mutate {
      remove_field => ["phone", "email", "password", "credit_card"]
    }
  }

  # 方式 2: 替换为掩码
  ruby {
    code => '
      if event.get("phone")
        phone = event.get("phone").to_s
        event.set("phone", phone[0..2] + "****" + phone[-4..-1])
      end
    '
  }

  # 方式 3: 仅保留 Hash（不可逆）
  mutate {
    gsub => [ "password", ".*", "******" ]
  }
}
```

# 2. ES 输出认证

```ruby
output {
  elasticsearch {
    hosts => ["https://es1:9200"]
    user => "logstash_writer"
    password => "${ES_PASSWORD}"
    ssl => true
    cacert => "/etc/logstash/certs/ca.crt"
    ssl_certificate_verification => true
  }
}
```

# 3. Keystore 密钥管理

```bash
# 创建 Keystore
bin/logstash-keystore create

# 添加密钥
bin/logstash-keystore add ES_PASSWORD
# Enter value: mysecret

# 列出
bin/logstash-keystore list

# 在配置中引用
output {
  elasticsearch {
    password => "${ES_PASSWORD}"     # 从 Keystore 读取
  }
}
```

**优势**：敏感信息不在配置文件中明文存储。

# 4. Monitoring UI

```
Kibana → Stack Monitoring → Logstash

监控面板:
  ├── Events Received / Emitted Rate
  ├── Pipeline Workers Count
  ├── Queue Events Count (堆积监控)
  ├── JVM Heap Usage
  ├── CPU Usage
  └── Output Plugin Latency
```

## 4.1 配置 Monitoring

```yaml
# logstash.yml
xpack.monitoring.enabled: true
xpack.monitoring.elasticsearch.hosts: ["https://es:9200"]
xpack.monitoring.elasticsearch.username: "logstash_system"
xpack.monitoring.elasticsearch.password: "${ES_PASSWORD}"
xpack.monitoring.elasticsearch.ssl.certificate_authority: "/etc/logstash/certs/ca.crt"
```

# 5. API 监控端点

```bash
# 节点信息
curl http://localhost:9600/_node

# 管道统计
curl http://localhost:9600/_node/stats/pipelines

# JVM 信息
curl http://localhost:9600/_node/stats/jvm

# 热线程
curl http://localhost:9600/_node/stats/hot_threads?human=true

# 插件信息
curl http://localhost:9600/_node/plugins
```

# 6. Dead Letter Queue 运维

```bash
# 查看 DLQ 文件
ls -la /var/lib/logstash/dead_letter_queue/

# 查看 DLQ 统计
curl http://localhost:9600/_node/stats | jq '.pipelines."main".dead_letter_queue'
```

```ruby
# 重放 DLQ
input { dead_letter_queue { path => "/var/lib/logstash/dead_letter_queue" } }
output { elasticsearch { ... } }
```

# 7. 审计日志

```yaml
# logstash.yml
log.format: json                    # JSON 格式日志
path.logs: /var/log/logstash
log.level: info

# 访问日志 (API 访问记录)
api.auth.type: basic
api.auth.basic.username: "admin"
api.auth.basic.password: "${API_PASSWORD}"
```

# 8. 生产安全检查清单

```
□ Keystore 存储所有密码（ES/ApI/Kafka）
□ 监控 ES 输出的认证错误
□ 敏感字段在管道中脱敏或删除
□ PQ 路径加密（如有合规要求）
□ API 端点认证开启（生产环境）
□ 定期检查 DLQ（发现下游问题）
□ 日志级别: info（生产）/ warn（稳定）
□ SSL/TLS 启用（Beats→Logstash、Logstash→ES）
```
