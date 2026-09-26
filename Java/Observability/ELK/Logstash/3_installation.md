
# 1. 环境要求

| 组件 | 要求 |
|------|------|
| JDK | 11+（8.x）/ 8+（7.x） |
| 内存 | 最小 1GB，推荐 2GB+ |
| OS | Linux/macOS/Windows |

# 2. Docker 部署

```bash
# Logstash 8.x
docker run -d \
  --name logstash \
  -v /path/to/logstash.conf:/usr/share/logstash/pipeline/logstash.conf \
  -v /path/to/logs:/var/log/app \
  docker.elastic.co/logstash/logstash:8.11.0
```

# 3. Linux 安装

```bash
# CentOS/RHEL
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
cat > /etc/yum.repos.d/logstash.repo << EOF
[logstash-8.x]
name=Logstash repository for 8.x
baseurl=https://artifacts.elastic.co/packages/8.x/yum
gpgcheck=1
enabled=1
EOF

yum install -y logstash
systemctl start logstash
```

# 4. Hello World — stdin → stdout

```ruby
# logstash.conf
input {
  stdin { }
}

output {
  stdout { codec => rubydebug }
}
```

```bash
bin/logstash -f logstash.conf

# 输入 "hello world"
# 输出:
# {
#   "message" => "hello world",
#   "@timestamp" => 2024-01-15T10:30:00.000Z,
#   "host" => "my-host",
#   "@version" => "1"
# }
```

# 5. 完整管道 — File → Grok → ES

```ruby
input {
  file {
    path => "/var/log/nginx/access.log"
    start_position => "beginning"
    sincedb_path => "/var/lib/logstash/sincedb"
  }
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
  mutate {
    remove_field => ["@version", "message", "timestamp"]
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "nginx-access-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "${ES_PASSWORD}"
  }
  stdout { codec => rubydebug }
}
```

# 6. logstash.yml 核心配置

```yaml
# 管道配置路径
path.config: /etc/logstash/conf.d/*.conf

# 数据路径
path.data: /var/lib/logstash
path.logs: /var/log/logstash

# 队列
queue.type: persisted
queue.max_bytes: 2gb
path.queue: /var/lib/logstash/queue

# 死信队列
dead_letter_queue.enable: true

# 管道性能
pipeline.workers: 4
pipeline.batch.size: 125
pipeline.batch.delay: 50

# JVM 配置 (jvm.options)
# -Xms1g -Xmx2g

# API 端点
api.enabled: true
api.http.host: "0.0.0.0"
api.http.port: 9600

# 自动重载
config.reload.automatic: true
config.reload.interval: 3s
```

# 7. 配置校验

```bash
# 校验语法（不实际运行）
bin/logstash -f pipeline.conf --config.test_and_exit

# 自动重载（修改配置后无需重启）
# logstash.yml: config.reload.automatic: true

# 查看 API
curl http://localhost:9600/_node/stats
curl http://localhost:9600/_node/pipelines
```

# 8. 环境变量

```ruby
# pipeline.conf 中使用环境变量
input {
  kafka {
    bootstrap_servers => "${KAFKA_BROKERS:localhost:9092}"
    topics => ["${TOPIC:default-topic}"]
  }
}

output {
  elasticsearch {
    hosts => ["${ES_HOST:localhost:9200}"]
    user => "${ES_USER:elastic}"
    password => "${ES_PASSWORD}"
  }
}
```
