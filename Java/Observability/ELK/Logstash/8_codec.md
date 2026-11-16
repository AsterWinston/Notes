# 1. 概述

Codec（编码/解码器）负责在 Input/Output 阶段对数据进行序列化和反序列化。

```text
Input:  原始字节流 → Codec.decode() → Event
Output: Event → Codec.encode() → 格式化输出
```

**codec 是"字节流与事件流之间的翻译官"，插在 I/O 插件的边缘执行**。这个位置选择并非偶然：解码发生在数据进入队列之前（filter worker 拿到的永远是 Event），编码发生在写出目的地之前（output 插件只管发字节）。于是 input/output 插件得以只关心"传输"（连接、确认、批量、重试），把"格式"这件千变万化的事外包给可组合的 codec——一个 tcp input 能接收 JSON、纯文本还是 Graphite 格式，全靠换一行 codec 配置。

三个结构性认知：**codec 不属于 filter 家族**，它处理的是"字节边界"问题——一条事件在流里从哪结束（line 按换行切、json_lines 按换行+完整性切、multiline 按正则归属切），这决定了为什么多行合并只能发生在 codec 层；**codec 有状态**，line/multiline 必须攒够一条完整事件才产出，输入残缺时会把内容缓存在插件实例里（每个 input 连接一份解码器实例，tcp 多客户端互不串线）；**codec 失败也打标记**——`_jsonparsefailure` 之类在 input 阶段就出现，事件不丢但内容以原始形态留在 `message`。

```text
input {
  kafka {
    codec => json          # 每条 Kafka 消息按 JSON 解码成一个 Event
  }
  file {
    codec => multiline { pattern => "^\s+at\s" what => "previous" }   # 堆栈合并
  }
}

output {
  file  { codec => line { format => "%{message}" } }   # 回写纯文本
  kafka { codec => json_lines }                        # 转发时序列化
}
```

# 2. 常用 Codec

## 2.1 json / json_lines

```text
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

两者区别在于**对"消息边界"的假设**：`json` 假设每次交付的数据恰好是一个完整 JSON 对象（Kafka 一条消息、UDP 一个包），解码器直接 parse 整段；`json_lines` 假设输入是"每行一个 JSON"的流（NDJSON，文件、TCP 流），按换行切分后逐段 parse，**未闭合的行会被扣在缓冲区等下一段数据拼接**——所以 `json_lines` 能正确处理"一行 JSON 被 TCP 拆成两个包到达"，`json` 配在 TCP 上则必然把半截 JSON 打成一堆 `_jsonparsefailure`。

> **注意**：json codec 解析成功后，JSON 的键**原样成为 Event 顶层字段**——包括 `@timestamp` 键会直接覆盖事件时间（好事，日志自带时间就不用 date filter 了），也包括恶意/畸形数据里的 `@timestamp: "abc"` 会引发难查的"事件消失"（非法时间戳文档进 ES 后被过滤）。不可信来源上先 `target` 隔离（codec 做不到，得改用 plain + filter json `{ target => ... }`）是更稳的姿态。

## 2.2 plain / line

```text
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

**plain 与 line 的区别只有一条：line 会按分隔符（默认 `\n`）切分流，plain 把"每次交付的数据"整个当一个事件**。对 file input 两者表现一致（input 已按行读）；对 tcp/udp，要注意各插件的 codec 默认值不同（tcp 默认是 `json_lines`，udp 默认是 `json`，file 默认是 `plain`）——"TCP 发一条不带换行的 JSON，事件永远不产生"正是 codec 默认值与换行边界假设造成的悬案。排查 input"没数据"时，第一问永远是：**边界切对了吗？**（换行符有没有、编码是不是 UTF-8——`charset => "GBK"` 是中文 Windows 日志的常客。）

## 2.3 multiline — 多行合并

multiline codec 是 Logstash 最有代表性的边缘能力：**把物理上多行、逻辑上同一条的日志合成一个 Event**。参数组合决定行为：

| 参数 | 取值 | 语义 |
|------|------|------|
| `pattern` | 正则 | 对每行执行一次匹配 |
| `negate` | true/false | 是否对匹配结果取反 |
| `what` | previous / next | 不匹配时（若 negate=true）续接到上一事件 / 归入下一事件 |
| `pattern_line_anchor` | start / end | 只看行首/行尾（8.x 起支持） |
| `max_lines` | 数字 | 单事件最多合并多少行（防毒内容撑爆内存） |
| `auto_flush_interval` | 秒 | 流空闲多久后强制输出缓冲中的事件 |

**Java 堆栈是 `negate + previous` 的教科书用例**——"凡是不以时间戳开头的行，都是上一行的续行"：

```text
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

```text
# 合并后的一条事件：
# {
#   "message" => "2024-01-15 10:30:00.123 ERROR 12345 --- [main] c.e.Service : db down
# java.sql.SQLException: Connection refused
#     at com.example.dao.UserDao.query(UserDao.java:42)
#     at com.example.Service.checkout(Service.java:88)
# Caused by: java.net.ConnectException: Connection refused
#     at ...",
#   "tags" => ["multiline"]
# }
```

被合并的事件会带上 `multiline` tag，供下游识别。**auto_flush_interval 是 multiline 的生命线**：一个事件的"下一条不以时间戳开头的行"迟迟不来时（日志尾部、低频应用），缓冲里的事件永远不会输出——`auto_flush_interval => 5` 表示"5 秒没有新行进来就强制交货"，K8s/容器日志采集几乎必配，否则最后一个异常永远看不见。

> **警告**：multiline 的语义是**跨行重组**，它假设"属于同一事件的行连续到达"。多进程写同一个日志文件（行交错）、`cat 文件1 文件2` 混合喂入时，两个堆栈会互相粘连成"科学怪人事件"。解法优先级：**Filebeat 侧的 multiline processors 更好**（按文件隔离状态、有 `match_once` 与超时机制，且把 CPU 留在边缘）；Logstash 端仅适合单来源顺序流；能改应用日志（JSON + 堆栈进单字段）就彻底绕开。

## 2.4 rubydebug（调试）

```text
output {
  stdout { codec => rubydebug }
}
```

rubydebug 是**只用于 output 端的调试 codec**：以 Ruby inspect 风格打印事件（`"field" => "value"` 树形缩进 + ANSI 配色），加 `metadata => true`（8.x 参数名 `properties`/`metadata`，视版本）还能显示平时不可见的 `@metadata` 树——排查"字段到底进了哪棵树"时它是唯一能看见元数据全貌的窗口。生产禁用（格式化开销大）。

## 2.5 csv / format 等补充件

`csv` codec 按表头映射解码（`columns => ["ip","ts","status"]` 或读首行动态取列名），把 CSV 文件/消息直接变成结构化字段，省掉 dissect；输出侧的 `line { format => "%{[clientip]} %{[response]} %{+YYYY-MM-dd}" }` 支持字段插值与日期宏，定制任意文本协议回写；`edgeshop`/`graphite`/`msgpack` 等长尾 codec 各司协议，`msgpack` 值得记住——它是"高性能二进制 JSON"，应用用 msgpack 写日志时 input codec 一行接好且比 json 省 CPU。

# 3. 二进制 Codec

## 3.1 avro

```text
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

```text
input {
  kafka {
    codec => protobuf {
      class_name => "com.example.EventProto"
      include_path => "/usr/share/logstash/proto"
    }
  }
}
```

**avro codec 按本地 schema 文件（.avsc）解码二进制消息**，适合"生产者用 avro SDK + schema registry 管理"的静态契约场景；schema 变更要同步更新本地文件，否则解码失败。**protobuf codec（logstash-codec-protobuf 插件）需要 `.proto` 定义与编译后的 Java 类路径**，配置 `class_name/include_path/recursive_fields`，比 avro 笨重——这是 Logstash 复用 Java 生态的又一次体现（代价是 JVM 里加载协议类、类冲突风险）。两者都不支持 Confluent Schema Registry 的动态拉取，重度 registry 用户的替代方案是**把解码前移到生产者/流处理层，Logstash 只处理 JSON/文本**，或者接受"schema 文件手工同步"的运维成本。

# 4. Codec vs Filter

| 场景 | 推荐 |
|------|------|
| 简单 JSON 解析 | Codec `json`（Input 阶段） |
| 复杂解析 + 字段操作 | Filter `json` + `mutate` |
| 多行合并 | Codec `multiline`（Input 阶段）或 Filebeat 端处理 |
| 二进制解码 | Codec `avro` / `protobuf` |

同一格式在 codec 层与 filter 层往往都能解（json/csv），差别在**粒度与位置**。codec 处理**整条交付数据**，decode 失败标记在"输入完整性"层面（半截 JSON、错编码）；filter 处理**指定字段**，能嵌进 if 分支、能配 target 隔离、能按来源混用策略（同一 Kafka topic 里 JSON 与文本混杂时，codec 只能选一个，filter 可以逐条判断）。**推荐倾向**：来源格式单一且可信 → input codec（一次解析、worker 拿到的直接是结构化事件，还省掉 message 字段）；来源混杂、需要失败兜底、要控制落点 → plain/line codec + filter json/csv（多一次字符串解析的 CPU，换全部控制权）。

# 5. Codec 与 Input/Output 组合矩阵

```text
                plain    line    json    json_lines    multiline    备注
stdin           ✓        ✓       ✓       ✓             ✓          调试全家桶
file            ✓默认    ✓       慎用    ✓             ✓          json 配文件易踩半行，优先 json_lines
tcp             慎用     ✓       慎用    ✓默认         ✓          json 无边界假设，半包即失败
udp             ✓        ✗       ✓       ✓             ✗          无连接，按包=按事件
beats           默认plain(自带批结构)  json 慎用       ✓          Beat 协议已分帧，codec 只处理 message 内容
kafka           ✓        ✗       ✓默认   慎用          ✗          一条消息=一个边界，json/plain 二选一
http            ✓        ✗       ✓       ✗             ✗          一次请求体=一个交付
file(输出)      ✓        ✓       ✓       ✓             ✗          line+format 定制回写
kafka(输出)     ✓        ✗       ✓       ✓             ✗          json/json_lines 决定消息体形态
stdout          ✓        ✓       ✓       ✓             ✗          rubydebug 专用

图例：✓ 常用可用 | 慎用 | ✗ 不适用/无意义
```

矩阵里几条"看起来违反直觉"的组合值得点名：**kafka + multiline 无意义**（消息边界由 Kafka 协议划定，不存在跨消息续行）；**beats + json 慎用**（Beat 协议已把事件框好，Filebeat 送来的是含 `message` 的批帧，json codec 会试图解码整个协议体——JSON 日志应在 Filebeat 侧 `decode_json_fields` 或在 Logstash filter 层解析）；**tcp + json 慎用**（同 2.1 的半包问题，TCP 流上的 JSON 用 json_lines 或换协议）。

# 6. 排障手册：codec 相关五类症状

| 症状 | 根因 | 处方 |
|------|------|------|
| ES 里查不到、stdout 却有 | `@timestamp` 被畸形 JSON 键覆盖 | json codec 换 plain + filter json target |
| 最后一个堆栈永远不出现 | multiline 缓冲未刷新 | `auto_flush_interval` |
| 事件内容错位/科学怪人堆栈 | 多文件交错 + multiline | Filebeat 侧 multiline 或改 JSON 日志 |
| 中文日志乱码 | charset 非 UTF-8 | `charset => "GBK"` |
| 大量 `_jsonparsefailure` | codec 与消息边界不匹配 | 按第 5 节矩阵换 line/json_lines |

> **行业实践**：codec 参数不进 filter、不跨连接共享实例——tcp 场景下每个客户端连接持有独立解码器，一个连接的垃圾数据不会污染另一个连接的解析状态；但这意味着**恶意超长行可以吃满单连接缓冲**，`max_lines` 与 Filebeat 侧的 `max_bytes` 是配套防线。所有 codec 变更（哪怕只是 json→json_lines）都要走 `-t` + 样本回归，因为改的是"事件边界"这种最底层的语义。

# 7. 小结

codec 站在 I/O 插件的门口管两件事：**字节边界与格式转换**——input 侧 decode（字节→Event）、output 侧 encode（Event→字节），让传输插件与格式解耦。选型主线：纯文本 line/plain（注意二者对"交付"粒度的差异与默认值陷阱）、结构化 json/json_lines（有边界的选 json，无边界流选 json_lines）、跨行重组 multiline（negate+previous 合并 Java 堆栈，auto_flush_interval 保命，交错与多进程场景交给 Filebeat）、二进制 avro/protobuf（复用 Java 生态但放弃 schema registry 动态性）。codec 与 filter 的分工线在"整条交付 vs 指定字段、静态单一 vs 动态混杂"；rubydebug 只用于调试。所有 codec 故障的共同特征是不报错却"看不见数据"——半包、缓冲、charset、覆盖 `@timestamp`，排查顺序永远是：**先确认边界，再确认解码，最后才怀疑 filter**。
