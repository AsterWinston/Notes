
# 1. pipelines.yml

```yaml
# /etc/logstash/pipelines.yml

# 管道 1: 接收 Filebeat 数据，按类型路由到不同管道
- pipeline.id: beats-input
  path.config: "/etc/logstash/conf.d/beats-input.conf"
  pipeline.workers: 2

# 管道 2: Nginx 日志处理
- pipeline.id: nginx-processing
  path.config: "/etc/logstash/conf.d/nginx.conf"
  pipeline.workers: 4
  queue.type: persisted

# 管道 3: 应用日志处理
- pipeline.id: app-processing
  path.config: "/etc/logstash/conf.d/app.conf"
  pipeline.workers: 6

# 管道 4: 错误日志 -> 告警
- pipeline.id: error-alerting
  path.config: "/etc/logstash/conf.d/error.conf"
  pipeline.workers: 2
```

# 2. 模块化配置拆分

```
/etc/logstash/conf.d/
  ├── 01-input.conf        # Input 配置
  ├── 02-filter.conf       # Filter 配置
  └── 03-output.conf       # Output 配置
```

```ruby
# 01-input.conf
input {
  beats { port => 5044 }
}

# 02-filter.conf
filter {
  if [type] == "nginx" {
    grok { match => { "message" => "%{COMBINEDAPACHELOG}" } }
  }
}

# 03-output.conf
output {
  elasticsearch { hosts => ["http://es:9200"] }
}
```

Logstash 按字母顺序加载 `conf.d/*.conf`，合并为一个管道配置。

# 3. 条件分支（单管道内）

```ruby
input {
  beats { port => 5044 }
}

filter {
  if [type] == "nginx" {
    grok { match => { "message" => "%{COMBINEDAPACHELOG}" } }
    geoip { source => "clientip" }
    mutate { remove_field => ["message", "timestamp"] }
  }
  else if [type] == "app" {
    json { source => "message" }
  }
}

output {
  if [type] == "nginx" {
    elasticsearch {
      hosts => ["http://es:9200"]
      index => "nginx-%{+YYYY.MM.dd}"
    }
  }
  else if [type] == "app" {
    elasticsearch {
      hosts => ["http://es:9200"]
      index => "app-%{+YYYY.MM.dd}"
    }
  }

  if "error" in [tags] {
    email {
      to => "alert@example.com"
      subject => "[ERROR] %{type}"
    }
  }
}
```

# 4. 多管道 vs 条件分支

| 维度 | 多管道 (pipelines.yml) | 条件分支 (if-else) |
|------|----------------------|-------------------|
| 隔离性 | ★★★★★（独立 Workers/Queue） | ★★☆☆☆ |
| 运维复杂度 | 高 | 低 |
| 资源利用 | 可按需分配 Worker | 共享 Workers |
| 适用 | 大规模、多源异构数据 | 中小规模、同构数据 |

# 5. 程序化管道配置

```yaml
# pipelines.yml
- pipeline.id: prod
  pipeline.workers: 8
  pipeline.batch.size: 250
  path.config: "/etc/logstash/conf.d/prod.conf"

- pipeline.id: staging
  pipeline.workers: 2
  pipeline.batch.size: 50
  path.config: "/etc/logstash/conf.d/staging.conf"
```

不同环境的数据用不同管道处理，资源隔离。

# 6. 管道自动重载

```yaml
# logstash.yml
config.reload.automatic: true
config.reload.interval: 3s
```

修改配置文件后无需重启 Logstash，自动加载新配置。
