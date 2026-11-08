# 1. Output 层的定位与语义

**output 是管道的出口：把加工完的 Event 写到目的地，并决定"写失败怎么办"**。三个结构性事实决定了所有 output 的行为边界：**其一，一个管道可以挂多个 output，同一事件默认会发给所有匹配的 output**（"一进多出"是语言级能力，不是要靠路由拼的技巧）；**其二，event 进入 output 时 filter 阶段已经结束，队列中不再回退**——写不写得进去是 output 的责任；**其三，绝大多数 output 按 batch 提交**，配合 `pipeline.batch.size` 与插件自身的 flush 参数决定网络往返频率。

排障时最容易混淆的概念是"**重试**与**DLQ** 的分工"：可恢复失败（ES 暂时不可用、网络超时、429 限流）由 output 内置重试兜住，事件在内存/batch 层面等待；不可恢复失败（单文档 mapping 冲突）无法靠重试解决，进 dead_letter_queue 存档。两者都覆盖不了的场景（下游长期宕机），要靠前文 `2_architecture.md` 的持久化队列把压力顶回源头。

# 2. 输出插件全景

| 输出 | 定位 | 批量语义 | 失败处理 | 典型场景 |
|------|------|----------|----------|----------|
| **elasticsearch** | 主存储 | bulk 攒批（flush_size/idle_flush_time） | 重试 + DLQ | 日志检索 |
| **kafka** | 二次分发 | producer 异步批（linger 类似） | 内置 retries | 清洗后转发下游 |
| **file** | 落盘留底 | flush_interval | 写完即终点 | 失败样本、合规导出 |
| **s3** | 归档 | rotation（time/size/interval） | 本地缓冲重试 | 长期冷存储 |
| **stdout** | 调试 | 即时 | — | 开发验证 |
| **email / http** | 通知 | 逐条 | 有限重试 | 告警（须限流） |

# 3. Elasticsearch

```text
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

elasticsearch output 是全栈使用率最高的输出插件，核心参数分四组。**连接与安全组**：`hosts`（8.x 默认开启安全后 `https + user/password + cacert` 是三件套）、`sniffing`——生产必须 false，true 时通过 `_nodes/http` 自动发现集群节点并把数据写到发现结果上，容器/多网卡环境下发现地址不可达是经典断流事故。**写入语义组**：`index`（见第 4 节）、`action`（默认 `index`；日志类**建议显式 `create`**——撞 `_id` 时 create 返回 409 进 DLQ，index 则静默整文档覆盖）、`document_id`（幂等去重手段，代价是 ES 要查版本号、写入变重）、`routing`（同一实体钉到一个 shard，局部性更好）。**批量性能组**：`flush_size`（攒多少条发一次 bulk）与 `idle_flush_time`（低流量下最多再等 1 秒强制发送，保证可见延迟上限）。**生命周期组**：ILM 与 Data Stream，见下节。

## 3.1 Data Stream：8.x 的日志写入正解

**Data Stream 把"日志索引怎么滚动、别名怎么维护"整个收进 ES**：output 端只声明 `data_stream => true` + 三段命名（`type=logs` / `dataset=nginx` / `namespace=prod`，组合出 `logs-nginx-prod`），写入时 ES 自动落进最新 backing index、自动滚动、自动保留别名。它要求两件事提前就位：**index template 预先创建**（含 `data_stream: {}` 段、ECS 映射、ILM 策略绑定，可用 Kibana 或 `POST /_index_template/logs-nginx-prod`）与 **`action => create` 语义**（data stream 只接受 create）。对比手工 `%{+YYYY.MM.dd}` 按天索引：data stream 天然带 `@timestamp` 时区无歧义的 `event.ingested` 双时间轴、ILM 托管热冷删、查询别名永远存在，是日志类数据的官方推荐；手工命名仍适合"需要精确控制索引名、被外部系统直接引用"的场景。

# 4. 索引命名与模板

```text
output {
  elasticsearch {
    index => "%{[@metadata][biz]}-%{+YYYY.MM.dd}"      # 字段插值 + 时间宏
  }
}
```

`index` 参数支持**字段引用 + 日期表达式混排**：`%{+YYYY.MM.dd}` 取事件 `@timestamp`（UTC）按格式展开，`%{[field]}` 取事件字段——这组合就是"按业务分索引"的全部机制。**命名三原则**：**全小写**（ES 硬性限制，大写直接拒绝写入并报 invalid index name）；**前缀有层次**（`{类型/业务}-{日期}`，如 `nginx-access-2024.01.15`、`app-order-2024.01.15`）；**日期粒度匹配查询与生命周期**（天级滚动利于删除与快照，但短生命周期数据按小时甚至 12 小时更省 shard 数）。

> **注意**：**动态模板（dynamic template）与 ILM 必须在数据写入之前建好**，mapping 是由第一批数据决定的——第一行日志把 `status` 写成 `"200"` 字符串，该字段从此固定为 text，后续所有数值查询永远做不了，且**改 mapping 只能重建索引**。正确姿势：先用 index template 锁定 `number→long`、`boolean→boolean`，或让 Logstash 端 `mutate convert` 先把类型做对，再开管道。

```json
PUT _index_template/app-logs
{
  "index_patterns": ["app-logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1,
      "lifecycle.name": "logs-policy"
    },
    "mappings": {
      "dynamic_templates": [
        { "numbers_as_long": { "match_mapping_type": "string",
          "match": "*_ms", "mapping": { "type": "long" } } }
      ],
      "properties": {
        "response":  { "type": "integer" },
        "bytes":     { "type": "long" },
        "duration":  { "type": "float" }
      }
    }
  }
}
```

# 5. Kafka — 二次转发

```text
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

"清洗后的日志再回 Kafka 给下游实时消费"是 Logstash 作为流处理中间层的标准拓扑。`acks` 是可靠性与吞吐的跷跷板：`1`（leader 落盘即确认，默认）在吞吐与不丢之间折中；`all` 要求全部 ISR 确认，不丢但最慢，配合 producer 端 `retries`/`enable.idempotence` 才有意义；`0` 纯发即忘，仅限可丢遥测。`compression_type` 建议 lz4/snappy（日志文本压缩比高、CPU 便宜）。**分区路由**用默认哈希（`key` 参数指定字段则按 key 分区，同一实体保序）；`topic_id` 不存在时是否自动建 topic 取决于 broker 的 `auto.create.topics.enable`，生产建议预建。

# 6. File — 落盘归档

```text
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

file output 的当下用途主要是**失败留底与合规导出**：把 `_grokparsefailure` 或某业务线的原始日志按天落地（`path` 支持字段/日期插值），交给离线批处理或对象存储搬运。`codec => line { format => "%{message}" }` 让它输出纯文本行而非 JSON；`gzip` 直接压缩省盘。两个限制要知道：**没有原子轮转**（按日期插值依赖路径变化，跨天瞬间可能有两个进程句柄并存），以及**落盘即终点**——file output 写完不进任何队列，下游程序读文件的可靠性它不管。

# 7. stdout（调试用）

```text
output {
  stdout {
    codec => rubydebug               # 彩色格式化输出
    # codec => json                  # JSON 单行输出
  }
}
```

`rubydebug` 输出 Ruby Hash 风格（带 ANSI 颜色、含 `@metadata`），`json` 输出紧凑单行（便于管道给 jq）。stdout 的纪律只有一条：**验证完就从配置里删掉**——高吞吐下格式化与终端 IO 是实打实的 CPU 税。调试场景优先 `-e` 短管道，其次 `file` 输出落盘代替 stdout。

# 8. Email / HTTP 告警出口

```text
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

```text
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

```text
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

email/http/s3 三个"通知与归档"出口的共同特征：**都是非核心链路，绝不该影响主数据流**。`http` output 的 `format => "json"` 在 batch>1 时是 json_batch（数组体），单条逐发用 `format => "json"`+`http_method => "post"`；`retry_failed` 只对幂等方法（GET/HEAD/PUT/DELETE）生效，POST 默认不重试—— webhook 重试重复通知的后果要自己想。`s3` 是合规归档主力：按 `rotation_strategy`（size/interval/time）切文件、`non_standard_event` 之类失败可整批落盘重试。

> **行业实践**：告警出口选 email/http 要克制——**Logstash 管道自身可能挂，"用管道发告警"等于让故障源自己报病**。真正的告警闭环应建在 ES/Kibana（Watch/Alerting）或独立监控系统上；email output 里 `throttle_number` 必须设（否则每条事件一封邮件，SMTP 封号+邮箱爆炸双重事故）。

# 9. 路由：条件输出与多管道

```text
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

条件路由是"一个管道服务多业务"的基础语法（if 作用于 output 块级，同一事件可命中多分支）。当分支数超过 5~6 个、或不同业务对性能/隔离要求差异大时，应升级为**多管道架构**：按业务拆分 `pipelines.yml` 定义的独立管道（各自队列、workers、配置），再用 **pipeline-to-pipeline**（8.7+）做进程内分发：

```text
output {
  pipeline { send_to => "es_out" }        # 路由给同进程的另一个管道
}
```

**设计考量**：三种路由形态的边界——**index 参数插值**适合"同一目的地、命名不同"（最廉价，按字段拼索引名即可）；**条件 output** 适合"少量目的地 + 少量规则"；**多管道 + pipeline 输出** 适合"目的地异构（ES/S3/Kafka 各自特性）、需要隔离故障域与独立扩缩"。判断线：当 output 段的 if 嵌套需要注释才能读懂时，就该拆管道了。

# 10. 重试与 Dead Letter Queue

ES 拒绝批量写入时，elasticsearch output 按错误码分治：**429（es_rejected_execution，写入队列满）与 5xx/网络错误自动退避重试**，事件不丢，代价是管道吞吐下降、压力回推；**400（单文档 mapping 冲突）中只有冲突的那几条被剔除**，其余文档继续写。被剔除的坏文档默认打进日志后丢弃，开启 DLQ 才存档：

```yaml
# logstash.yml
dead_letter_queue.enable: true
dead_letter_queue.max_bytes: 1gb
dead_letter_queue.storage_policy: drop_newer   # 满了丢新还是丢旧
```

DLQ 的运维闭环三件套：**监控**（`_node/stats` 的 `dlq.queue_size_in_bytes` 与页文件数，配告警）、**回放**（第 2 篇的 `dead_letter_queue` input 专用管道，修好 mapping 后 `commit_on_success => true` 安全重放）、**清理**（回放完确认 `queue_size` 归零再关管道；长期不管 DLQ 等于磁盘上埋一颗"满了开始丢数据"的定时炸弹）。

> **注意**：DLQ 只捕获"文档级被拒绝"的失败，**不捕获网络断连、认证失败这类批级错误**（它们走重试路径）；也不覆盖 stdout/file/http 等不支持 DLQ 的插件。别把它当"输出失败保险箱"的全集。

# 11. 按业务分索引的完整实践

把前文机制串成一套可落地的组合：

```text
filter {
  # 1. 用来源信息确定业务域，放进 @metadata（不会污染文档）
  if [log][file][path] =~ /nginx/       { mutate { add_field => { "[@metadata][biz]" => "nginx-access" } } }
  else if [log][file][path] =~ /order/  { mutate { add_field => { "[@metadata][biz]" => "app-order" } } }
  else                                  { mutate { add_field => { "[@metadata][biz]" => "app-misc" } } }
}

output {
  elasticsearch {
    hosts   => ["https://es1:9200"]
    index   => "%{[@metadata][biz]}-%{+YYYY.MM.dd}"   # 2. 插值进索引名
    action  => "create"
  }
}
```

要点复盘：**业务域判定进 `@metadata`**（不入库、索引名里照样能用）；每类业务**预建同名 pattern 的 index template**（shards/replicas、ILM、字段 mapping 在第一批数据前锁死）；Kibana 按 `nginx-access-*`、`app-order-*` 建 index pattern，权限用 ES 的 index 级角色控制。

# 12. 小结

output 层的知识地图：**elasticsearch** 是主战场——安全三件套（https/账号/ca）、生产关 sniffing、日志用 create 语义、批量参数控制可见延迟、8.x 首选 Data Stream + ILM 托管滚动；**索引命名**靠 `%{[@metadata][biz]}-%{+YYYY.MM.dd}` 插值实现按业务分索引，但成败在**写数据之前**的 index template（mapping 被第一批数据决定，错了只能重建）。**kafka** 出口用 acks/compression/key 做可靠性与吞吐折中；**file/s3** 负责留底与归档；**email/http** 是非核心告警，克制使用且必须限流。可靠性分两层：可恢复错误靠内置重试 + 退避，不可恢复的文档级拒绝靠 DLQ 存档-监控-回放-清理闭环；长期宕机则由持久化队列兜住。路由能力分三层：索引插值 < 条件 output < 多管道 + pipeline-to-pipeline——规则复杂度的增长线，就是架构升级线。
