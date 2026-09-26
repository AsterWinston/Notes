
# 1. Grok — 正则解析

```ruby
filter {
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }
  }
}
```

详见 `6_grok.md`。

# 2. Mutate — 字段操作

```ruby
filter {
  mutate {
    # 重命名
    rename => { "old_field" => "new_field" }

    # 删除
    remove_field => ["@version", "message", "beat", "host", "input", "tags"]

    # 替换
    replace => { "env" => "production" }

    # 类型转换
    convert => { "response" => "integer" }
    convert => { "bytes" => "integer" }
    convert => { "duration" => "float" }

    # 值合并
    coerce => { "bytes" => 0 }        # 空值 → 0

    # 复制
    copy => { "clientip" => "src_ip" }

    # 字符串操作
    gsub => [ "message", "\\r\\n", " " ]
    strip => ["message", "user_agent"]
    lowercase => ["method"]
    uppercase => ["country"]
  }
}
```

# 3. Date — 时间字段

```ruby
filter {
  date {
    match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z", "ISO8601" ]
    target => "@timestamp"            # 替换默认时间字段
    timezone => "Asia/Shanghai"
  }
}
```

**重要性**：`@timestamp` 是 ES 的默认时间字段，不正确设置将导致 Kibana 时间显示错误。

# 4. JSON — 解析 JSON

```ruby
filter {
  json {
    source => "message"               # 从哪个字段解析
    target => "parsed"                # 输出到哪个字段（默认根级别）
    skip_on_invalid_json => true
  }
}

# 输入: '{"user":"alice","action":"login","status":200}'
# 输出: { "user" => "alice", "action" => "login", "status" => 200 }
```

# 5. Dissect — 分隔符解析

Dissect 比 Grok 更快（无需正则），适合分隔符明确的结构化日志：

```ruby
filter {
  dissect {
    mapping => {
      "message" => "%{ts} [%{level}] %{class} - %{msg}"
    }
  }
}

# 输入: "2024-01-15 10:30:00.123 [INFO] com.example.Service - user created"
# 输出:
# { "ts" => "2024-01-15 10:30:00.123",
#   "level" => "INFO",
#   "class" => "com.example.Service",
#   "msg" => "user created" }
```

## 5.1 Dissect 高级

```ruby
# 跳过字段: %{}
# 重复分隔符: %{+order}
dissect {
  mapping => {
    "message" => "%{ts} %{+ts} [%{level}] %{logger} - %{msg}"
  }
}
# 输入: "2024-01-15 10:30:00.123 [INFO] com.example.Service - hello world"
# ts => "2024-01-15 10:30:00.123"
# level => "INFO"
```

# 6. GeoIP — IP 地理位置

```ruby
filter {
  geoip {
    source => "clientip"              # IP 字段
    target => "geoip"                 # 输出目标
    database => "/usr/share/logstash/GeoLite2-City.mmdb"
    fields => ["city_name", "country_name", "location", "region_name"]
  }
}
```

# 7. Translate — 字典映射

```ruby
filter {
  translate {
    field => "status_code"
    destination => "status_desc"
    dictionary => {
      "200" => "OK"
      "301" => "Moved Permanently"
      "404" => "Not Found"
      "500" => "Internal Server Error"
    }
    fallback => "Unknown"
  }
}
```

# 8. Ruby — 自定义脚本

```ruby
filter {
  ruby {
    code => '
      # 自定义复杂逻辑
      if event.get("amount")
        event.set("tax", event.get("amount").to_f * 0.13)
        event.set("total", event.get("amount").to_f + event.get("tax"))
      end
    '
  }
}
```

# 9. 条件执行

```ruby
filter {
  # 仅在匹配时执行
  if [type] == "nginx" {
    grok { match => { "message" => "%{COMBINEDAPACHELOG}" } }
  }

  if "error" in [message] or "ERROR" in [message] {
    mutate { add_tag => ["error"] }
  }

  if [response] =~ /^5\d\d$/ {
    mutate { add_field => { "alert" => "true" } }
  }
}
```

# 10. 错误处理

```ruby
filter {
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }
    tag_on_failure => ["_grokparsefailure"]
  }

  # 只处理成功解析的事件
  if "_grokparsefailure" not in [tags] {
    date { match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ] }
  }
}
```
