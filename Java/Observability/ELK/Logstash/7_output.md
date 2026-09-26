
# 1. Elasticsearch

```ruby
output {
  elasticsearch {
    hosts => ["https://es1:9200", "https://es2:9200", "https://es3:9200"]
    index => "app-logs-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "${ES_PASSWORD}"
    ssl => true
    cacert => "/etc/logstash/certs/ca.crt"

    # 索引策略
    action => "create"               # create / index / update / delete
    document_id => "%{order_id}"     # 自定义 doc _id
    routing => "%{user_id}"          # 自定义 routing

    # Data Stream（8.x 推荐）
    data_stream => true
    data_stream_type => "logs"
    data_stream_dataset => "nginx"
    data_stream_namespace => "production"

    # ILM
    ilm_enabled => true
    ilm_policy => "logs-policy"
    ilm_rollover_alias => "app-logs"

    # 性能
    flush_size => 500                # 每批文档数
    idle_flush_time => 1             # 空闲刷新间隔(秒)
    sniffing => false                # 生产关闭 sniffing
  }
}
```

# 2. Kafka

```ruby
output {
  kafka {
    bootstrap_servers => "kafka1:9092,kafka2:9092,kafka3:9092"
    topic_id => "processed-logs"
    compression_type => "snappy"
    acks => "1"                      # 0 / 1 / all
    security_protocol => "SASL_SSL"
    sasl_mechanism => "PLAIN"
    sasl_jaas_config => "org.apache.kafka.common.security.plain.PlainLoginModule required username='user' password='pass';"
    codec => json
  }
}
```

# 3. File

```ruby
output {
  file {
    path => "/data/output/%{type}-%{+YYYY-MM-dd}.log"
    codec => line { format => "%{message}" }
    write_behavior => "append"       # append / overwrite
    gzip => true
    flush_interval => 5
  }
}
```

# 4. stdout（调试用）

```ruby
output {
  stdout {
    codec => rubydebug               # 彩色格式化输出
    # codec => json                  # JSON 单行输出
  }
}
```

# 5. Email

```ruby
output {
  email {
    to => "admin@example.com"
    from => "logstash@example.com"
    subject => "Alert: %{alert_type} at %{@timestamp}"
    body => "Event details: %{message}"
    address => "smtp.example.com"
    port => 587
    username => "${SMTP_USER}"
    password => "${SMTP_PASS}"
    use_tls => true
  }
}
```

# 6. HTTP

```ruby
output {
  http {
    url => "https://api.example.com/webhook"
    http_method => "post"
    format => "json"
    headers => { "Authorization" => "Bearer ${API_TOKEN}" }
    retry_failed => true
    retry_non_idempotent => false
  }
}
```

# 7. S3

```ruby
output {
  s3 {
    region => "us-east-1"
    bucket => "my-logs-bucket"
    prefix => "logstash/%{+YYYY/MM/dd}"
    codec => "json_lines"
    rotation_strategy => "time"
    time_file => 15                 # 每 15 分钟轮转
  }
}
```

# 8. 条件输出

```ruby
output {
  if [type] == "nginx" {
    elasticsearch { index => "nginx-%{+YYYY.MM.dd}" }
  } else if [type] == "app" {
    elasticsearch { index => "app-%{+YYYY.MM.dd}" }
  }

  if "error" in [tags] {
    email { to => "alert@example.com" subject => "[ERROR] Logstash Alert" }
  }

  if "_grokparsefailure" in [tags] {
    file { path => "/var/log/logstash/parse_failures.log" }
  }
}
```
