采集是日志链路里最容易被低估的一环：应用日志写得再规范，如果 Filebeat 漏读、多行没合并、Grok 正则写错，后面全白搭。本篇以 Filebeat 8.x 和 Logstash 8.x 为准，讲清输入模型、多行合并、解析过滤、背压与断点续传，最后列出最常见的配置错误。

# 1. Filebeat 8.x 的输入模型

## 1.1 filestream 取代 log

8.x 的默认输入类型是 `filestream`，旧的 `log` 类型已不再推荐：

| 维度 | `log`（旧） | `filestream`（新） |
|------|-------------|---------------------|
| 读取方式 | 轮询文件 | 基于 inode 的文件流 |
| 文件轮转 | 可能漏读 | 能跟踪重命名/轮转 |
| 状态存储 | registry 文件 | 每个输入独立状态 |
| 多行 | multiline 配置 | parsers 里的 multiline |
| 性能 | 一般 | 更好，支持更细的指纹 |

```yaml
filebeat.inputs:
  - type: filestream
    id: order-api-log            # 唯一 ID，状态按它隔离，改了会重新读
    enabled: true
    paths:
      - /var/log/order-api/*.log
    parsers:
      - ndjson:                  # 应用输出的是 JSON 日志
          target: ""             # 解析到顶层，字段直接成为 ES 字段
          overwrite_keys: true   # 同名时以日志内容为准
          add_error_key: true    # 解析失败时记录 error 字段，便于排查
```

★ **关键认知**：`id` 是 filestream 的**身份标识**，不是随便的备注。改了 `id` 或删掉状态文件，Filebeat 会认为这是新输入，从文件头重读，可能造成大量重复。上线后不要轻易改。

## 1.2 直连 ES 的最小配置

```yaml
filebeat.inputs:
  - type: filestream
    id: order-api
    paths:
      - /var/log/order-api/*.log
    parsers:
      - ndjson:
          target: ""
          overwrite_keys: true

output.elasticsearch:
  hosts: ["https://es-node1:9200", "https://es-node2:9200"]
  username: "filebeat_writer"
  password: "${ES_PASSWORD}"          # 从 keystore 注入，不要明文
  index: "logs-order-api-%{+yyyy.MM.dd}"

setup.kibana:
  host: "https://kibana:5601"
```

## 1.3 断点续传原理

Filebeat 把"读到哪个文件的哪个偏移"记在 registry 里：

```text
registry（状态文件）
┌──────────────────────────────────────────────┐
│ /var/log/order-api/app.log                   │
│   inode: 123456                              │
│   offset: 88213                              │
│   timestamp: 2026-11-06T10:12:33Z            │
└──────────────────────────────────────────────┘
```

重启后，Filebeat 从 offset 继续读，而不是从头。这就是"断点续传"。

```text
正常流程：
写 100 行 → Filebeat 读到第 80 行 → 记录 offset → 重启 → 从第 81 行继续

文件被轮转：
app.log → app-2026-11-06.log，新建 app.log
Filebeat 靠 inode 识别：旧文件读完再读新文件，不会漏

文件被删除：
inode 消失，Filebeat 在 close_timeout 后关闭 harvester
```

⚠ **关键认知**：**断点续传的前提是文件还在、inode 还能对应上**。如果日志轮转后立刻被删除（`MaxBackups` 太小或清理脚本太激进），Filebeat 还没来得及读完，那部分日志就永久丢了。轮转保留时间必须大于 Filebeat 的最大滞后时间。

# 2. 多行日志

## 2.1 为什么需要合并

Go 的 panic、Java 的异常堆栈、Python 的 traceback 都是多行：

```text
2026-11-06 10:12:33 ERROR panic: runtime error: invalid memory address
goroutine 1 [running]:
main.main()
    /app/main.go:42 +0x1d
```

不合并的话，ES 里会变成 4 条独立文档，堆栈和错误分离，排障时根本对不上。

## 2.2 multiline 配置

Filebeat 8.x 的 multiline 放在 `parsers` 里：

```yaml
filebeat.inputs:
  - type: filestream
    id: order-api
    paths: ["/var/log/order-api/*.log"]
    parsers:
      - multiline:
          type: pattern
          pattern: '^\d{4}-\d{2}-\d{2}'   # 以日期开头的行是"新日志"
          negate: true                   # 不匹配的行拼到上一行
          match: after                   # 拼到上一行之后
          max_lines: 500                 # 防止异常日志无限合并
          timeout: 5s                    # 5 秒没有新行就强制发出
      - ndjson:
          target: ""
          overwrite_keys: true
```

| 参数 | 含义 | 常见值 |
|------|------|--------|
| `pattern` | 判断"新日志起始行"的正则 | `^\d{4}-\d{2}-\d{2}` |
| `negate` | `true` 表示不匹配 pattern 的行属于上一条 | `true` |
| `match` | `after` 或 `before` | `after` |
| `max_lines` | 单条日志最多合并多少行 | 500 |
| `timeout` | 超时强制发出，避免最后一条卡住 | 5s |

★ **关键认知**：**multiline 的 pattern 要匹配"新日志的开头"，而不是"堆栈的行"**。用 `negate: true` + `match: after`，意思是"凡是不像新日志开头的行，都接到上一条后面"。写反了（`negate: false`）会把所有正常行合并成一条巨型日志。

## 2.3 容器日志的特殊性

Kubernetes 里，容器的 stdout 会被 runtime 写成 JSON：

```json
{"log":"2026-11-06 10:12:33 ERROR panic: ...\n","stream":"stderr","time":"2026-11-06T10:12:33Z"}
{"log":"goroutine 1 [running]:\n","stream":"stderr","time":"..."}
```

此时多行合并要在**解析 JSON 之后**做，或者用 Filebeat 的 autodiscover + 容器日志专用配置：

```yaml
filebeat.autodiscover:
  providers:
    - type: kubernetes
      node: ${NODE_NAME}
      hints.enabled: true        # 通过 Pod annotation 配置多行规则
      templates:
        - condition:
            contains:
              kubernetes.labels.app: order-api
          config:
            - type: filestream
              id: "k8s-order-api-${data.kubernetes.container.id}"
              paths:
                - /var/log/containers/*${data.kubernetes.container.id}.log
              parsers:
                - container: {}   # 先解容器 runtime 的 JSON
                - multiline:
                    type: pattern
                    pattern: '^\d{4}-\d{2}-\d{2}'
                    negate: true
                    match: after
```

⚠ **关键认知**：容器场景下 `parsers` 的顺序很重要：`container` 必须在 `multiline` 之前。否则多行规则匹配到的是 JSON 外层，而不是 `log` 字段里的内容。

# 3. 字段与处理器

## 3.1 fields 与 tags

```yaml
filebeat.inputs:
  - type: filestream
    id: order-api
    paths: ["/var/log/order-api/*.log"]
    fields:
      service: order-api        # 自定义字段，会进 ES
      env: prod
    fields_under_root: true     # true 表示放顶层，false 会放在 fields.* 下
    tags: ["go", "order"]       # 打标签，便于条件过滤
```

## 3.2 processors

Filebeat 的 processors 在发送前做轻量处理：

```yaml
processors:
  - add_host_metadata: {}             # 加 host.name、host.ip
  - add_cloud_metadata: {}            # 云环境自动加 provider、region
  - add_kubernetes_metadata: {}       # K8s 环境加 pod、namespace
  - drop_fields:                      # 删掉无用字段，省存储
      fields: ["agent.ephemeral_id", "ecs.version"]
      ignore_missing: true
  - rename:
      fields:
        - from: "log.file.path"
          to: "log_path"
      ignore_missing: true
  - timestamp:                        # 用日志里的时间覆盖采集时间
      field: "@timestamp"
      layouts:
        - "2006-01-02T15:04:05.000Z0700"
      test:
        - "2026-11-06T10:12:33.123+0800"
```

| processor | 用途 | 注意 |
|-----------|------|------|
| `add_host_metadata` | 加主机信息 | 多实例时按 host 聚合 |
| `add_kubernetes_metadata` | 加 Pod 信息 | 需要 RBAC 权限 |
| `drop_fields` | 删字段 | 先确认没被 Dashboard 用到 |
| `rename` | 改字段名 | 对齐 ECS |
| `timestamp` | 用日志时间 | 必须有 `test` 样例，否则静默失败 |
| `decode_json_fields` | 解析 JSON | 已用 `ndjson` 时不需要 |

## 3.3 条件输出

同一台机器上不同日志发往不同地方：

```yaml
output.elasticsearch:
  hosts: ["https://es:9200"]
  index: "logs-%{[service]}-%{+yyyy.MM.dd}"

# 或者按 tag 走不同输出
output.logstash:
  hosts: ["logstash:5044"]
  # Filebeat 本身不支持 if/else 输出，通常用多个 input + 条件 processors
```

★ **关键认知**：Filebeat 的 `index` 支持字段插值，但**字段必须存在，否则整条事件会发送失败**。用 `%{[service]}` 前务必确认每条日志都有 `service`，或在 processors 里给默认值。

# 4. Logstash 管道

## 4.1 三段式结构

```ruby
# /etc/logstash/conf.d/order.conf
input {
  beats {
    port => 5044
    # 开启 TLS 时配置 ssl_certificate / ssl_key
  }
}

filter {
  # 解析、转换、丰富
}

output {
  elasticsearch {
    hosts => ["https://es:9200"]
    index => "logs-order-api-%{+yyyy.MM.dd}"
  }
}
```

## 4.2 Grok 解析

Grok 用命名正则把非结构化文本拆成字段：

```ruby
filter {
  grok {
    match => {
      "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} %{GREEDYDATA:msg}"
    }
    # 解析失败时不丢弃，打 tag 便于后续排查
    tag_on_failure => ["_grok_parse_failure"]
  }
}
```

常用内置模式：

| 模式 | 匹配 | 例子 |
|------|------|------|
| `%{TIMESTAMP_ISO8601}` | ISO 时间 | `2026-11-06T10:12:33.123+0800` |
| `%{LOGLEVEL}` | 日志级别 | `INFO`、`ERROR` |
| `%{IPORHOST}` | IP 或主机名 | `10.0.0.8` |
| `%{NUMBER}` | 数字 | `123.45` |
| `%{GREEDYDATA}` | 剩余全部 | `anything` |
| `%{COMBINEDAPACHELOG}` | Apache 组合日志 | Nginx access log |

⚠ **关键认知**：Grok 正则是**性能杀手**。`%{GREEDYDATA}` 本质是 `.*`，放在中间会导致大量回溯。优先用**锚定开头**、尽量具体的模式，或用 `dissect` 替代简单分隔符解析。

## 4.3 dissect：更快的前缀解析

固定格式的日志，用 `dissect` 比 Grok 快一个数量级：

```ruby
filter {
  dissect {
    mapping => {
      "message" => "%{timestamp} %{level} %{logger} %{msg}"
    }
  }
}
```

| 对比 | Grok | dissect |
|------|------|---------|
| 原理 | 正则捕获 | 按分隔符切分 |
| 速度 | 慢 | 快 |
| 灵活性 | 高 | 低，格式必须固定 |
| 适合 | 格式多变 | 格式稳定 |

## 4.4 常用 filter 组合

```ruby
filter {
  # 1. 解析 JSON 日志
  json {
    source => "message"
    target => "doc"
    skip_on_invalid_json => true
  }

  # 2. 用日志时间替代采集时间
  date {
    match => ["[doc][ts]", "UNIX", "ISO8601"]
    target => "@timestamp"
    timezone => "Asia/Shanghai"
  }

  # 3. 字段改名与类型转换
  mutate {
    rename => { "[doc][user_id]" => "user_id" }
    convert => { "amount_cent" => "integer" }
    remove_field => ["message", "doc", "event.original"]
  }

  # 4. 丰富：根据 IP 加地理位置
  geoip {
    source => "client_ip"
    target => "geo"
  }

  # 5. 条件路由：错误日志单独打标签
  if [level] == "error" {
    mutate { add_tag => ["needs_alert"] }
  }
}
```

## 4.5 多输出与路由

```ruby
output {
  # 错误日志单独一个索引，便于设置更长的保留期
  if "needs_alert" in [tags] {
    elasticsearch {
      hosts => ["https://es:9200"]
      index => "logs-errors-%{+yyyy.MM.dd}"
      user => "logstash_writer"
      password => "${ES_PASSWORD}"
    }
  } else {
    elasticsearch {
      hosts => ["https://es:9200"]
      index => "logs-all-%{+yyyy.MM.dd}"
    }
  }

  # 调试时打开 stdout，生产关闭以免拖慢管道
  # stdout { codec => rubydebug }
}
```

★ **关键认知**：Logstash 的 `if` 条件里访问嵌套字段要用**方括号路径**：`[doc][level]`。写成 `doc.level` 会返回 nil，条件永远不成立。这是新手最常踩的坑之一。

# 5. 背压与可靠性

## 5.1 持久化队列

默认内存队列在进程崩溃时丢数据。生产建议开 PQ：

```yaml
# logstash.yml
queue.type: persisted
path.queue: /var/lib/logstash/queue
queue.max_bytes: 8gb
queue.checkpoint.writes: 1024   # 每 1024 条写一次检查点，越小越安全越慢
```

| 配置 | 内存队列 | 持久化队列 |
|------|----------|------------|
| 崩溃丢数据 | 会 | 不会（最多丢检查点间隔） |
| 磁盘占用 | 无 | 队列大小 |
| 吞吐 | 高 | 略低 |
| 适合 | 可丢日志 | 审计、核心业务 |

## 5.2 bulk 与重试

```ruby
output {
  elasticsearch {
    hosts => ["https://es:9200"]
    index => "logs-app-%{+yyyy.MM.dd}"
    bulk_path => "/_bulk"
    manage_template => false          # 由索引模板统一管理
    ilm_enabled => true               # 8.x 推荐用 ILM 而不是按天索引
    ilm_rollover_alias => "logs-app"
    retry_initial_interval => 2
    retry_max_interval => 64
    retry_on_conflict => 3
  }
}
```

## 5.3 死信队列

解析失败或映射冲突的事件，不应该静默消失：

```yaml
# logstash.yml
dead_letter_queue.enable: true
dead_letter_queue.max_bytes: 1gb
path.dead_letter_queue: /var/lib/logstash/dlq
```

```ruby
# 读取 DLQ 排查
input {
  dead_letter_queue {
    path => "/var/lib/logstash/dlq"
    commit_offsets => false
  }
}
```

★ **关键认知**：**背压的正确形态是"上游变慢"，不是"下游丢数据"**。Filebeat 发送失败会重试，Logstash PQ 满了会拒绝，ES 会用 429 限流。每一跳都应该"顶住"而不是"丢弃"，除非你明确接受丢日志。

# 6. 常见错误

| 问题 | 现象 | 修复 |
|------|------|------|
| multiline pattern 写反 | 所有日志合并成一条 | 用"新日志开头"做 pattern + `negate: true` |
| Grok 用 `GREEDYDATA` 开头 | CPU 飙高、延迟大 | 锚定具体前缀，或用 dissect |
| 改了 filestream 的 `id` | 日志重复 | 上线后不要改 id |
| 日志轮转后立刻删除 | 采集漏读 | 保留时间 > Filebeat 滞后 |
| 字段路径写 `doc.level` | 条件不生效 | 用 `[doc][level]` |
| 忽略 bulk errors | 静默丢日志 | 检查响应并告警 |
| `date` filter 时区错 | 时间差 8 小时 | 显式 `timezone => "Asia/Shanghai"` |
| 生产开 `stdout` 输出 | 管道变慢 | 关闭或仅调试时开 |
| 动态 mapping 裸奔 | 字段类型冲突 | 显式索引模板 |

⚠ **关键认知**：排查"日志没进 ES"的顺序是：**本地文件有吗 → Filebeat 日志有 `Harvester` 报错吗 → registry 的 offset 在涨吗 → Logstash 收到事件了吗 → ES 的 bulk 响应 `errors` 是 false 吗**。按链路逐段排除，不要一上来就改查询。

# 7. 练习与自检

1. 应用输出如下日志，请写出 Filebeat 的 multiline 配置，并说明 `pattern`、`negate`、`match` 三个参数为什么这样取值：
   ```text
   2026-11-06 10:12:33 ERROR request failed
   traceback:
     file "a.go", line 10
     file "b.go", line 20
   ```
2. 写一个 Logstash filter，要求：解析 JSON、用 `ts` 字段覆盖 `@timestamp`、把 `amount` 转成整数、删除原始 `message`，并把 `level=error` 的事件打上 `needs_alert` 标签。
3. 你的 Filebeat 重启后日志重复了 30 分钟的量，可能的原因有哪些？请至少列出三条并给出验证方法。

# 8. 小结

| 主题 | 要点 |
|------|------|
| filestream | 8.x 默认，靠 inode + offset 续传 |
| `id` | 输入身份，改了会重读 |
| multiline | pattern 匹配新日志开头，`negate: true` |
| 容器日志 | 先 `container` 解析，再 multiline |
| Grok | 灵活但慢，能用 dissect 就用 dissect |
| 字段路径 | 嵌套字段用 `[a][b]` |
| PQ | 崩溃不丢，生产建议开启 |
| 背压 | 每跳有界，顶住而非丢弃 |

下一篇进入 Kibana，讲清 Data View（索引模式）、KQL 查询语法、字段映射、可视化与 Dashboard、检索排错，以及基于 Space 和 Role 的权限控制。
