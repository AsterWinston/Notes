# 1. Input 的角色与总体形态

**input 插件的唯一职责是"把外部世界变成 Event 并塞进队列"**：它决定三件事——数据从哪来（协议/文件/队列）、一次投多少（是否批量）、以及源头的元信息往哪放（`host`、`path`、`@metadata`）。所有 input 共享一套通用参数：`add_field`、`tags`、`type`（已不推荐）、`codec`，以及 `ecs_compatibility`（8.x 起默认 `v8`，决定字段命名走 ECS 还是老式扁平）。

**一个管道可以声明多个 input**，它们并行运行、产出的事件汇入同一条 filter/output 链——这就是"多路复用"。复用的代价是**来源混流后必须靠字段区分**，而 input 本身不会自动打标签，需要手工补齐。下面的公共模式贯穿全篇：

```text
input {
  beats { port => 5044 }              # 应用/系统日志
  kafka {
    bootstrap_servers => "k1:9092"
    topics => ["business-events"]
    add_field => { "[metadata][source]" => "kafka" }
  }
}

filter {
  if [metadata][source] == "kafka" { ... }   # 按来源分流处理
}
```

> **注意**：`add_field` 写入的字段会一路带到 ES；如果只是想用来做条件判断，写成 `[metadata][source]`（进 `@metadata` 树）才不会被持久化。忘打标签导致"所有来源走同一条 grok、日志里满屏 `_grokparsefailure`"是 Logstash 事故榜第一名。

# 2. Beats — 生产首选入口

```text
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

Beats input 实现 **Beat 协议**（TCP 上的帧化批传输）：一个 Filebeat 连接一次送来**一批**事件，Logstash 逐条转成 Event 后回 ACK；Filebeat 收不到 ACK 就暂停该连接并保留数据——**背压天然成立**，这是 `beats` 优于裸 `tcp` 采集日志的根本原因（tcp input 没有应用层确认，写入失败即丢）。

Filebeat 送来的事件自带一棵结构化的元数据树，值得逐字段认识：
```json
{
  "@timestamp": "2024-01-15T02:30:00.000Z",
  "message": "192.168.1.1 - - [15/Jan/2024:10:30:00 +0800] \"GET /api HTTP/1.1\" 200 45",
  "ecs": { "version": "8.0.0" },
  "agent": { "name": "filebeat-8.11.0", "type": "filebeat", "hostname": "web-01", "ephemeral_id": "..." },
  "host": { "name": "web-01", "os": { "platform": "linux" }, "ip": ["10.0.0.5"] },
  "log": { "file": { "path": "/var/log/nginx/access.log" }, "offset": 1048576 },
  "input": { "type": "log" },
  "@metadata": { "beat": "filebeat", "type": "doc", "version": "8.11.0" }
}
```

两个实践结论：**`log.file.path` 是天然的分流键**（按文件路径判断"这是 nginx 还是应用日志"，比 `type` 可靠），`log.offset` 配合 path 可做重复数据定位；而 `agent`/`host`/`ecs` 这几棵树在入库前多半要瘦身——`mutate { remove_field => ["agent","ecs","input"] }` 能把文档体积压掉可观的一截，`host.name` 若与 ES 的 `host` 字段冲突还需改名。

**设计考量**：beats input 是高连接数的重灾区（每台机器一个 Filebeat = 一个长连接）。`host => "0.0.0.0"` + 前置 LB 时注意 TCP 空闲连接回收；要拿真实客户端 IP 需开 `proxy_protocol => true` 并让 LB 发送 HAProxy 协议，否则 `host` 只有 LB 的地址。

# 3. File — 文件尾部读取

```text
input {
  file {
    path => "/var/log/app/*.log"
    start_position => "beginning"        # 首次从文件头读
    sincedb_path => "/var/lib/logstash/sincedb"
    stat_interval => 1                   # 文件变化检测间隔(秒)
    discover_interval => 15              # 新文件发现间隔(秒)
    mode => "tail"                       # tail / read
    delimiter => "\n"                    # 行分隔（新版实际由 codec line 承担）
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

三条规则决定 file input 的行为边界。**其一，位置跟踪认 inode 不认文件名**：`inode` 变了就是"新文件"（日志轮转 rename+create 的场景没问题；`copytruncate` 轮转方式会让 inode 不变而内容截断，读位置可能越过新内容导致丢数据——**file input 要求轮转用 create 模式**）。**其二，`start_position => "beginning"` 只对 sincedb 里没有记录的文件生效**：反复测试时若不清 sincedb，第二次跑就"什么都不读"，把 `sincedb_path` 指到 `/dev/null` 可强行每次从头读（仅限测试）。**其三，`mode => "read"` 读完即关闭并移入 finished 状态**，适合历史文件批量灌入；默认的 `tail` 才适合常驻采集。

> **注意**：file input 读的是**本机磁盘**，容器里跑 Logstash 读宿主机日志要挂对路径且关心权限（`logstash` 用户读不到 `/var/log` 是高频故障）。生产环境用 file input 直读文件的场景正在被 Filebeat 取代——多机采集时 Filebeat 的 registry + 背压方案运维成本低得多；file input 的合理岗位是"本机日志单一来源"或"批量补历史数据"。

# 4. Kafka — 削峰主力

```text
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

**kafka input 是"以 Kafka 为缓冲带"架构的核心**：Filebeat/应用先把日志灌进 Kafka（承担海量写入与削峰），Logstash 集群以消费者组身份横向扩容消费——**每个 partition 同一时刻只被组内一个 Logstash 实例消费**，因此"实例数 ≤ partition 数"是容量铁律，多出的实例纯闲置。

参数里有三个容易配错：**`auto_offset_reset` 决定"没有提交位点时从哪开始"**，`latest` 会让新加入的实例完全跳过历史（补数据场景必须 `earliest`）；**`enable_auto_commit`** 默认 true 走 Kafka 的周期提交，崩溃时按最后一次提交重复消费（at-least-once）；**`max_partition_fetch_bytes`** 与消息大小要匹配，日志单条大于默认值时改消费端无效、得同步改 broker 的 `message.max.bytes`。

> **行业实践**：`codec => "json"` 只在消息体确定是 JSON 时用——一旦混入非 JSON 消息，json codec 会给事件打上 `_jsonparsefailure` 并把**原始字节**塞进 `message`，后续 grok 分支要能兜住；更稳的做法是 codec 用 `plain`，在 filter 里用 `json { source => "message" }` 逐条解析，解析失败可以按 tag 走 DLQ/单独索引，而不是在 input 阶段就污染数据。

# 5. Redis — 轻量队列

```text
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

Redis input 用 `BRPOP` 从 list 批量弹出（`batch_count` 一次最多取多少条），或用 SUBSCRIBE 消费 pub/sub 通道。`data_type` 的选择即语义的选择：**`list` 有持久化且"消息只消费一次"**（配合 RDB/AOF 可容忍重启），`channel`/`pattern_channel` 是**扇出广播**——多个 Logstash 实例都收到同一条消息，用来做"多实例同时订阅"而不是扩容吞吐，用错了就是数据翻倍的事故。作为 6.x 之前的"穷人版 Kafka"，Redis 方案在中等流量下仍常见；流量上来就该迁 Kafka。

# 6. JDBC — 定时 SQL 查询

```text
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

**JDBC input 把"数据库定时拉取 → ES"这件脏活压缩成 15 行配置**，核心机制是 `:sql_last_value` 占位符 + 增量记录：每次查询成功后，`tracking_column` 的最大值被写进 `last_run_metadata_path` 指向的文件，下次运行从该值继续（cron 语法由 `schedule` 控制，不填则只跑一次）。

**设计考量**：`use_column_value => true` 要求跟踪列存在且单调（`updated_at`、自增 `id`），并且必须配 `tracking_column_type`（`numeric`/`timestamp`）——类型不对会让比较逻辑静默失效、每次全表重拉。软删除、`deleted` 标记这类"库里删了 ES 里也要删"的需求 JDBC input 无法直接满足，得靠 `action => "delete"` 的输出侧配合或换 **Debezium/CDC** 路线。另外 JDBC input 是**阻塞式**的，一个大结果集查询会长时间占住这个 input 线程，`fetch_size`（游标批量）和分页 SQL 是控制单次内存驻留的旋钮；别用 `SELECT *` 拉大字段。

# 7. HTTP Poller

```text
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

`http_poller` 是**定时拉取型** input：按 schedule 请求若干命名 URL，响应体交给 codec（通常 json）解析成事件——适合把第三方 API、指标接口、字典服务的数据周期性灌进管道。两个必知细节：**响应的 HTTP 头、状态码会进 `@metadata`**（`@metadata[response_headers]`、`@metadata[response_code]`），排查"没数据"时先看是不是 4xx 被静默；cron 表达式 schedule 里**必须显式写时区**，否则按机器时区解释导致任务错峰。要接收**外部推来的 HTTP 请求**用另一个插件 `http` input，二者方向相反。

# 8. Syslog / TCP / UDP / HTTP / stdin

```text
input {
  syslog {
    port => 514
    type => "syslog"
    proxy_protocol => true
  }
}
```

`syslog` input 内置 RFC3164/5424 解析器，直接把 `<34>Jan 15 10:30:01 host sshd[123]: ...` 拆成 `priority/facility_code/timestamp/host/prog/pid/msg`——**这是少数"不需要 grok 就能结构化"的 input**。514 是特权端口，非 root 运行需要 `setcap 'cap_net_bind_service=+ep'` 或前端用 `syslog-ng/rsyslog` 转发到高端口。

```text
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

```text
input {
  http {
    port => 8080
    codec => "json"
  }
}
```

`tcp`/`udp`/`http` 三个通用入口的定位分别是：**tcp** 万能兜底（任何行式/JSON 客户端、log4j socket appender、网络设备），但**没有应用层 ACK**，客户端发出即认为成功，Logstash 挂了数据就丢——只适合"可容忍丢失"或上游自带重连缓冲的场景；**udp** 无连接、丢包常态化（`buffer_size` 调大接收缓冲只缓解不乱序），适合遥测打点类数据；**http** input 暴露 REST 端点，适合低频 webhook 与第三方直推（注意它会把每个请求体解析成事件，大 body 要限制 `max_body_size`）。`stdin` 只用于开发调试。

> **行业实践**：凡选 tcp/udp/http 作采集入口，就要在其上游自己补可靠性（应用侧本地磁盘队列 + 重试），或者承认"日志可丢"。**需要可靠传输的日志入口一律走 beats（或 kafka）**，这是 Logstash 官方生态设计的第一优先级。

# 9. 常见采集拓扑

```text
拓扑 A（推荐起步）: 边缘轻采集 + 中心重加工
┌────────────┐   ┌────────────┐   ┌────────────┐
│ Filebeat xN│──→│  Logstash  │──→│Elasticsearch│
└────────────┘   └────────────┘   └────────────┘

拓扑 B（削峰）: Kafka 缓冲带
Filebeat xN ──→ Kafka(cluster) ──→ Logstash xM ──→ ES
                                        └──→ Kafka(processed) ──→ 下游消费

拓扑 C（一进多出）: 同一条管道分流
                        ┌─ if nginx ─→ ES[nginx-*]
syslog/beats → filter ──┼─ if app ───→ ES[app-*]
                        └─ 全量 ─────→ S3(归档) / Kafka(实时)
```

三种拓扑的取舍差异在**可靠性与成本的落点**：A 最简单，但 Logstash 挂掉时压力顶回 Filebeat，日志滚动过快可能丢；B 用 Kafka 换"下游故障不丢数据 + 计算层随意伸缩"，代价是多维护一个集群；C 强调"一进多出"——同一份流既进 ES 检索，又转 Kafka 风控、转 S3 归档。

# 10. Input 插件选择建议

| 场景 | 推荐 Input |
|------|-----------|
| 日志文件采集 | Filebeat → Logstash Beats Input |
| 消息队列 | Kafka（高吞吐）/ Redis（轻量） |
| 数据库同步 | JDBC（定时）/ Debezium（CDC 实时） |
| HTTP API | HTTP Poller |
| 网络设备 | Syslog / TCP / UDP |
| 第三方推送 | HTTP input（低频）/ Kafka（高频） |
| 历史文件回灌 | file（mode=read）或 stdin + 管道 |
| 死信回放 | dead_letter_queue input |

**设计考量**：input 的选择顺序应当是"**先看可靠性要求，再看吞吐，最后看接入成本**"。可靠要求高 → beats/kafka（有确认语义）；吞吐极高、要多实例 → kafka（partition 即并行度）；协议特殊（网络设备、老应用 socket appender）→ tcp/udp/syslog 兜底但要接受丢包；数据库与 API 属"拉取型"，用 jdbc/http_poller 而不是硬塞进日志链路。

# 11. 小结

input 层的全部知识点可以压缩成一句话：**决定数据怎么进来，并负责让它"认得出来源"**。beats 是生产默认入口（带 ACK 与背压，`log.file.path`/`agent.*` 是分流与瘦身依据）；file input 靠 inode+offset 的 sincedb 续读，要求 create 模式轮转；kafka input 以 partition 数定并行度，`auto_offset_reset` 与 codec 选择是两大坑位；jdbc/http_poller 属定时拉取，增量靠 `:sql_last_value` + metadata 文件；tcp/udp/http 是万能但不可靠的兜底。多 input 并存时必须手工 `add_field`（写进 `@metadata` 更干净）打标签，否则 filter 层的 if 分支与最终索引都会混乱。选 input 的顺序：可靠性 → 吞吐 → 接入成本。
