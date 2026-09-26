
# 1. Grok 原理

Grok 是 Logstash 的**正则表达式引擎**，将非结构化日志解析为结构化字段。

```
日志行: 192.168.1.1 - - [15/Jan/2024:10:30:00 +0800] "GET /api/user HTTP/1.1" 200 45

Grok Pattern: %{IP:clientip} %{USER:ident} %{USER:auth} \[%{HTTPDATE:timestamp}\] "%{WORD:method} %{DATA:request} HTTP/%{NUMBER:httpversion}" %{NUMBER:response} %{NUMBER:bytes}
```

# 2. 内置 Pattern

| Pattern | 匹配 | 示例输入 |
|---------|------|---------|
| `IP` | IPv4/IPv6 | 192.168.1.1 |
| `HOSTNAME` | 主机名 | web-server-01 |
| `WORD` | 字母数字 | GET |
| `INT` | 整数 | 200 |
| `NUMBER` | 数字 | 45, 3.14 |
| `DATA` | 非贪婪任意 | /api/user |
| `GREEDYDATA` | 贪婪任意 | 整行剩余内容 |
| `HTTPDATE` | HTTP 日期 | 15/Jan/2024:10:30:00 +0800 |
| `QUOTEDSTRING` | 带引号字符串 | "hello world" |
| `TIMESTAMP_ISO8601` | ISO8601 时间 | 2024-01-15T10:30:00.000Z |
| `NOTSPACE` | 无空格字符串 | active |
| `LOGLEVEL` | 日志级别 | INFO、ERROR、DEBUG |

# 3. 复合 Pattern

```
COMBINEDAPACHELOG:
  %{IP:clientip} %{USER:ident} %{USER:auth}
  \[%{HTTPDATE:timestamp}\] "%{WORD:verb} %{DATA:request} HTTP/%{NUMBER:httpversion}"
  %{NUMBER:response} %{NUMBER:bytes} "%{DATA:referrer}" "%{DATA:agent}"

COMMONAPACHELOG:
  %{IP:clientip} %{USER:ident} %{USER:auth}
  \[%{HTTPDATE:timestamp}\] "%{WORD:verb} %{DATA:request} HTTP/%{NUMBER:httpversion}"
  %{NUMBER:response} %{NUMBER:bytes}

JAVASTACKTRACEPART:
  at %{NOTSPACE:class}\.%{NOTSPACE:method}\(%{DATA:file}(?::%{INT:line})?\)
```

# 4. 常见场景

## 4.1 Spring Boot 日志

```ruby
grok {
  match => {
    "message" => "%{TIMESTAMP_ISO8601:log_time} %{LOGLEVEL:level} %{NUMBER:pid} --- \[%{DATA:thread}\] %{DATA:class} : %{GREEDYDATA:msg}"
  }
}
# 输入: 2024-01-15 10:30:00.123 INFO 12345 --- [main] com.example.App : Started in 3.5s
```

## 4.2 Nginx Error Log

```ruby
grok {
  match => {
    "message" => "%{DATA:timestamp} \[%{WORD:level}\] %{DATA:pid}#%{NUMBER:tid}: \*%{INT:cid} %{GREEDYDATA:msg}"
  }
}
```

## 4.3 自定义业务日志

```ruby
grok {
  match => {
    "message" => "%{TIMESTAMP_ISO8601:ts} \[%{WORD:service}\] %{WORD:action} userId=%{WORD:userId} amount=%{NUMBER:amount} status=%{WORD:status}"
  }
}
# 输入: 2024-01-15T10:30:00 [order] CREATE userId=U123 amount=99.9 status=SUCCESS
```

# 5. 自定义 Pattern

```bash
# patterns/custom.grok
ORDER_ID [A-Z]{3}\d{6}
CUSTOM_DATE \d{4}-\d{2}-\d{2}
CUSTOMER [a-z_]+
```

```ruby
filter {
  grok {
    patterns_dir => ["/etc/logstash/patterns"]
    match => { "message" => "%{CUSTOM_DATE:date} \[%{ORDER_ID:order_id}\] %{CUSTOMER:user} %{GREEDYDATA:msg}" }
  }
}
```

# 6. Grok Debugger

```
Kibana → Dev Tools → Grok Debugger
  输入 Sample Data + Grok Pattern → 查看解析结果

或在线工具: https://grokdebugger.com
```

# 7. 性能优化

```ruby
# ✗ 差: 没有锚定，引擎回溯
grok { match => { "message" => "%{DATA:field1} %{DATA:field2}" } }

# ✓ 好: 使用 ^ 锚定开头
grok { match => { "message" => "^%{DATA:field1} %{DATA:field2}" } }

# ✓ 好: 用 Dissect 替代 Grok (分隔符明确时)
dissect {
  mapping => { "message" => "%{field1} %{field2}" }
}

# ✓ 好: 先过滤再 Grok
if [type] == "nginx" {
  grok { ... }
}

# ✗ 差: 贪婪匹配过多
# %{GREEDYDATA} 只在最后使用一个
```

# 8. Grok vs Dissect

| 维度 | Grok | Dissect |
|------|------|---------|
| 原理 | 正则表达式 | 分隔符定位 |
| 灵活性 | ★★★★★ | ★★★☆☆ |
| 性能 | 中 | 高（5-10x faster） |
| 适用 | 复杂/不规则日志 | 分隔符明确的日志 |
| 学习曲线 | 中 | 低 |
