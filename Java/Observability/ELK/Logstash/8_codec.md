
# 1. 概述

Codec（编码/解码器）负责在 Input/Output 阶段对数据进行序列化和反序列化。

```
Input:  原始字节流 → Codec.decode() → Event
Output: Event → Codec.encode() → 格式化输出
```

# 2. 常用 Codec

## 2.1 json / json_lines

```ruby
input {
  file {
    path => "/var/log/app/*.json"
    codec => "json"
  }

  kafka {
    topics => ["app-logs"]
    codec => "json"
  }
}
```

`json`：单条 JSON 记录（一个 JSON 对象 = 一个 Event）
`json_lines`：每行一个 JSON（适合流式数据）

## 2.2 plain / line

```ruby
input {
  file {
    path => "/var/log/app/*.log"
    codec => "plain"            # 默认，每行 = 一个 Event.message
  }

  tcp {
    port => 5000
    codec => "line"             # 每行 = 一个 Event
  }
}
```

## 2.3 multiline — 多行合并

```ruby
input {
  file {
    path => "/var/log/app/app.log"
    codec => multiline {
      pattern => "^\d{4}-\d{2}-\d{2}"     # 匹配行首的时间戳
      negate => true                       # 不匹配 → 续接到上一行
      what => "previous"                   # previous / next
      auto_flush_interval => 5
    }
  }
}

# 将 Java 异常堆栈合并为一个 Event
codec => multiline {
  pattern => "^\s+at\s"
  what => "previous"
}
```

## 2.4 rubydebug（调试）

```ruby
output {
  stdout { codec => rubydebug }
}
```

# 3. 二进制 Codec

## 3.1 avro

```ruby
input {
  kafka {
    topics => ["avro-events"]
    codec => avro {
      schema_uri => "/etc/logstash/schemas/event.avsc"
    }
  }
}
```

## 3.2 protobuf

```ruby
input {
  kafka {
    codec => protobuf {
      class_name => "com.example.EventProto"
      include_path => "/usr/share/logstash/proto"
    }
  }
}
```

# 4. Codec vs Filter

| 场景 | 推荐 |
|------|------|
| 简单 JSON 解析 | Codec `json`（Input 阶段） |
| 复杂解析 + 字段操作 | Filter `json` + `mutate` |
| 多行合并 | Codec `multiline`（Input 阶段）或 Filebeat 端处理 |
| 二进制解码 | Codec `avro` / `protobuf` |

# 5. Codec 选择建议

```
数据格式?
  ├── 纯文本（每行一条） → plain / line
  ├── JSON 格式 → json / json_lines
  ├── 多行日志（堆栈等） → multiline
  ├── 二进制序列化 → avro / protobuf
  └── 调试输出 → rubydebug
```
