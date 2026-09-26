
# 1. Beats

```ruby
input {
  beats {
    port => 5044
    host => "0.0.0.0"
    ssl => true
    ssl_certificate => "/etc/logstash/certs/logstash.crt"
    ssl_key => "/etc/logstash/certs/logstash.key"
  }
}
```

# 2. File — 文件尾部读取

```ruby
input {
  file {
    path => "/var/log/app/*.log"
    start_position => "beginning"        # 首次从文件头读
    sincedb_path => "/var/lib/logstash/sincedb"
    stat_interval => 1                   # 文件变化检测间隔(秒)
    discover_interval => 15              # 新文件发现间隔(秒)
    mode => "tail"                       # tail / read
    delimiter => "\n"
  }
}
```

**sinceDB**：记录每个文件的读取位置（inode + offset），重启后从上次位置继续。

```bash
# sinceDB 文件内容（不可直接编辑）
cat /var/lib/logstash/sincedb
# 12345 0 0 1048576
# (inode) (maj) (min) (offset)
```

# 3. Kafka

```ruby
input {
  kafka {
    bootstrap_servers => "kafka1:9092,kafka2:9092,kafka3:9092"
    topics => ["app-logs", "nginx-logs"]
    group_id => "logstash-consumer"
    client_id => "logstash-01"
    auto_offset_reset => "latest"          # earliest / latest
    enable_auto_commit => true
    max_poll_records => 500
    security_protocol => "SASL_SSL"
    sasl_mechanism => "PLAIN"
    sasl_jaas_config => "org.apache.kafka.common.security.plain.PlainLoginModule required username='user' password='pass';"
    codec => "json"
  }
}
```

# 4. Redis

```ruby
input {
  redis {
    host => "localhost"
    port => 6379
    data_type => "list"                # list / channel / pattern_channel
    key => "logstash-queue"
    batch_count => 125                 # 每批读取数
  }
}
```

# 5. JDBC — 定时 SQL 查询

```ruby
input {
  jdbc {
    jdbc_connection_string => "jdbc:mysql://localhost:3306/orders"
    jdbc_user => "root"
    jdbc_password => "${DB_PASSWORD}"
    jdbc_driver_class => "com.mysql.cj.jdbc.Driver"
    jdbc_driver_library => "/usr/share/logstash/mysql-connector.jar"

    statement => "SELECT * FROM orders WHERE updated_at > :sql_last_value ORDER BY updated_at"
    schedule => "*/5 * * * *"         # 每 5 分钟
    use_column_value => true
    tracking_column => "updated_at"
    tracking_column_type => "timestamp"
    last_run_metadata_path => "/var/lib/logstash/.jdbc_last_run"
  }
}
```

# 6. HTTP Poller

```ruby
input {
  http_poller {
    urls => {
      api1 => {
        method => "GET"
        url => "https://api.example.com/orders/stats"
        headers => { "Authorization" => "Bearer ${API_TOKEN}" }
      }
    }
    request_timeout => 60
    schedule => { every => "1m" }
    codec => "json"
  }
}
```

# 7. Syslog

```ruby
input {
  syslog {
    port => 514
    type => "syslog"
    proxy_protocol => true
  }
}
```

# 8. TCP / UDP

```ruby
input {
  tcp {
    port => 5000
    codec => "json_lines"
    ssl_enable => true
    ssl_cert => "/etc/logstash/certs/server.crt"
    ssl_key => "/etc/logstash/certs/server.key"
  }

  udp {
    port => 5001
    buffer_size => 65536
  }
}
```

# 9. Input 插件选择建议

| 场景 | 推荐 Input |
|------|-----------|
| 日志文件采集 | Filebeat → Logstash Beats Input |
| 消息队列 | Kafka（高吞吐）/ Redis（轻量） |
| 数据库同步 | JDBC（定时）/ Debezium（CDC 实时） |
| HTTP API | HTTP Poller |
| 网络设备 | Syslog / TCP / UDP |
