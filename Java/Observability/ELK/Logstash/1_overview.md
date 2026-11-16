# 1. 概述

**Logstash 是 Elastic Stack 的数据处理管道（Data Processing Pipeline）**，负责从多种来源采集数据，进行转换和丰富，然后输出到目标存储，是"ELK"中的"L"。这个定义里最容易被吞掉的是"处理"两个字：Logstash 从来不是单纯的搬运工，它的全部价值在于数据从"源头"到"存储"的中间那一段被**重新加工**——解析、清洗、富化、分流、路由。

一句话定位：**Logstash 是运行在 JVM 上的 ETL 工具**，上游对接几乎任意数据源（文件、消息队列、数据库、HTTP、syslog……），下游对接几乎任意存储，中间用一份声明式 DSL（input / filter / output 三段）描述逐条数据的加工逻辑。它处理的最小单位是**事件（Event）**——一条被抽象成 JSON 文档的记录，所有插件都在对这一个数据结构做增删改查。不管数据来自 Kafka 消息、MySQL 行还是 syslog 报文，进入管道后一律被规约成同一个 Event 结构，**所有插件只认这一个数据结构**——插件矩阵是乘法复用而不是排列爆炸，这是 Logstash 生态能积累 200+ 插件的根本原因。

很多团队的误区是把 Logstash 当"日志采集器"用，在每个业务机器上装一个 Logstash 去 tail 本地文件——这是对它最昂贵的用法。**正确分工是：边缘轻采集（Beats），中心重加工（Logstash）**——它应以少数集群节点的形式部署在中心位置，作为数据进入 Elasticsearch 之前的"加工厂"。

```text
┌──────────────────────────────────────────────────────┐
│                   Logstash Pipeline                   │
│                                                       │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐       │
│  │  Input   │───→│  Filter  │───→│  Output  │       │
│  │ (输入)    │    │  (过滤)   │    │ (输出)    │       │
│  └──────────┘    └──────────┘    └──────────┘       │
│       │                │                │             │
│  Filebeat          Grok             Elasticsearch    │
│  Kafka             Mutate           Kafka            │
│  Redis             JSON             File             │
│  JDBC              GeoIP            S3               │
│  HTTP              Date             Email            │
└──────────────────────────────────────────────────────┘
```

上图的三段之间完全由**插件**解耦：input 把外部世界变成 Event，filter 逐条改写 Event，output 把 Event 写到外部世界。三段各自水平扩展（input 线程、filter worker、output 线程池），互不阻塞，这是它架构灵活性的来源。

# 2. ELK 中的角色

```text
传统 ELK 架构:
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Filebeat │───→│ Logstash │───→│Elasticsearch│──→│ Kibana   │
│ (采集)    │    │ (处理)    │    │  (存储)    │    │ (可视化)  │
└──────────┘    └──────────┘    └──────────┘    └──────────┘

Logstash 在中间负责：
  - 数据解析（非结构化 → 结构化）
  - 数据丰富（GeoIP、字典映射）
  - 数据过滤（丢弃无效数据）
  - 数据路由（不同数据到不同索引）
```
在这个链路里四个组件职责边界很清楚：**Filebeat 管"送到"**（每台机器驻留，低资源搬运，带背压与断点续传），**Logstash 管"洗干净"**（杂乱文本变字段统一的事件），**Elasticsearch 管"存好查好"**，**Kibana 管"看"**。Logstash 是唯一"可插拔"的一环——抽掉它，Filebeat 直连 ES，系统照样运行，只是"洗"的活要么没人干，要么换人干：ES 自带的 **Ingest Node**（`_ingest/pipeline`，支持 grok、date、geoip、set 等处理器，配合 index template 的 `default_pipeline` 自动生效）。Ingest Node 的优势是零额外组件、随集群扩缩；劣势是处理器种类有限、无法消费 Kafka/JDBC、没有磁盘队列削峰、重加工与搜索写入抢同一节点 CPU。

## 2.1 什么时候可以去掉 L

**没有下面这些需求时，Logstash 就是可以省掉的**：日志格式单一且规整（应用直接输出 JSON，Filebeat 直连 ES 即可）；不需要跨源汇聚分流或二次转发；解析规则简单到 ES Ingest Pipeline 三四个 processor 就能写完；量级小到不值得多维护一个组件。反过来，只要出现"多源异构汇聚""重 ETL""需要削峰与落盘缓冲""出口不止 ES 一个"，Logstash 的价值就回来了。

> **行业实践**：中小团队最常见的起步形态是 Filebeat → Elasticsearch（配 ingest pipeline），等日志种类、数据源数量、解析复杂度上来之后，再在中间插入一层 Logstash 集群。不必从第一天就把"ELK 三个字母"装齐——L 是为复杂度服务的，不是为仪式感服务的。

# 3. 核心能力

| 能力 | 说明 | 示例 |
|------|------|------|
| **多源采集** | 200+ Input 插件 | Beats/Kafka/JDBC/Redis/File |
| **数据转换** | 50+ Filter 插件 | Grok 正则/Mutate 字段/GeoIP |
| **多目标输出** | 50+ Output 插件 | ES/Kafka/S3/File |
| **持久化** | Persistent Queue + DLQ | 防止数据丢失 |
| **多管道** | 一进程多管道 | 按数据流分离处理 |

这份能力清单里真正构成护城河的是**中间那列 Filter**：grok 把非结构化文本还原成字段，date 纠正 `@timestamp`，geoip 把 IP 变成坐标，translate 把状态码翻译成业务语义，ruby filter 兜底定制逻辑。这套组合拳在"脏数据进、干净数据出"的场景里几乎没有平替。**可靠性能力（Persistent Queue + DLQ）则决定了它能不能进生产**：开启持久化队列后，事件写入磁盘队列才算完成 input 阶段，进程重启、下游阻塞期间数据落在盘上；写不进 ES 的"坏事件"进入死信队列存档、事后可回放。

# 4. vs Fluentd vs Fluent Bit vs Vector

| 维度 | Logstash | Fluentd | Fluent Bit | Vector |
|------|----------|---------|-----------|--------|
| **语言** | JRuby (JVM) | C + Ruby | C | Rust |
| **内存占用** | 高 (JVM 500MB+) | 中 (~50MB) | 极低 (~5MB) | 低 (~20MB) |
| **插件生态** | ★★★★★ (200+) | ★★★★☆ | ★★★☆☆ | ★★★☆☆ |
| **处理能力** | 高（重量处理） | 中 | 低（轻量） | 高 |
| **吞吐量** | 中（JVM限制） | 高 | 极高 | 极高 |
| **ELK 集成** | ★★★★★ 原生 | ★★★☆☆ | ★★★★☆ | ★★★☆☆ |
| **适用场景** | 重量 ETL | 通用采集 | 边缘/容器 | 高性能管道 |

**选型建议**：重量处理和复杂 ETL 用 Logstash；轻量采集和边缘设备用 Fluent Bit；K8s 环境用 Fluentd/Vector。

这四者的差异很大程度上是**实现语言决定的形态差异**：Fluent Bit 用 C 写成、5MB 内存能跑，但刻意保持"轻"，复杂加工能力有限；Fluentd 的 C 核心 + Ruby 插件取中庸值；Vector 用 Rust，性能上限最高、配置即代码（TOML/YAML），但生态年轻、面向 ES 的集成不如 Logstash 贴身；Logstash 用 JVM 换来复用 Java 生态（JDBC 驱动、Kafka 客户端都是现成的）和处理全面性，代价就是内存、启动与吞吐。

> **注意**：这几者不是互斥关系而是分层关系。生产里非常典型的组合是"边缘 Fluent Bit/Filebeat + 中心 Logstash + 下游 ES"：让 C/Rust 系代理负责海量机器上的廉价搬运，让 Logstash 只守着少数汇聚点做重加工——各花各的钱。

# 5. 适用与不适用场景

```text
✓ 适用：
  - 非结构化日志解析（Nginx/Apache/自定义日志 → JSON）
  - 数据丰富和转换（GeoIP、字段重命名、类型转换）
  - 多源数据汇聚后分类输出到不同索引
  - Kafka → ES 的数据管道
  - 定时数据库查询同步到 ES

✗ 不适用：
  - 简单日志采集（直接用 Filebeat → ES）
  - 边缘/容器环境（资源开销大，用 Fluent Bit）
  - 超低延迟场景（JVM GC 可能引发延迟抖动）
  - 只需采集不需处理的场景（Filebeat 直接输出）
```

这张"适用/不适用"清单的本质是一条判断线：**数据在传输途中是否需要"被重新理解"**。需要（把一行文本拆成字段、按内容路由、补外部信息），上 Logstash；不需要（结构化日志直通存储），让 Logstash 靠边站。另一个常被忽略的适用场景是"以 ES 为查询端的数据服务化"：把 MySQL 表通过 JDBC input 定时灌进 ES，让 ES 兼任只读查询引擎——这类"非日志 ETL"几乎没有其他组件比 Logstash 更省事。

## 5.1 Logstash vs Filebeat：重加工与轻采集

| 维度 | Filebeat（轻采集） | Logstash（重加工） |
|------|--------------------|--------------------|
| 定位 | 边缘代理：读文件、发事件 | 中心管道：解析、富化、路由 |
| 实现 | Go，单二进制 | JRuby/JVM |
| 单机内存 | 数十 MB 级 | 默认 1GB 堆起步 |
| 部署密度 | 每台主机/每个 Pod 一个 | 中心集群少数几台 |
| 处理能力 | 少量 processors（add_fields、decode_json 等） | grok/geoip/translate/ruby 全套 |
| 缓冲/断点 | 内存队列 + registry 偏移 | 持久化队列（PQ）落盘 + sincedb |
| 典型去向 | 直连 ES 或交给 Logstash | ES / Kafka / S3 / 多路输出 |

两者的衔接协议是 **Beat 协议**（Beats input，默认 5044 端口，TCP + 帧批 + 可选 TLS）：Filebeat 感知下游 Logstash 的背压（批没被 ACK 就暂停发送），实现"发送方限速"而非"接收方丢数据"。官方推荐的采集姿态——**Filebeat 做 Runner（每机一个、只管跑），Logstash 做加工厂（集中少数节点、只管重活）**。

# 6. Logstash 8.x 变化

| 特性 | 说明 |
|------|------|
| **Java Execution Engine** | 8.0 默认 Java 引擎（弃用 Ruby），管道执行性能提升 |
| **ECS Compatibility** | 默认启用 Elastic Common Schema 字段兼容 |
| **Security ON by Default** | 8.x 默认开启安全，需配置 ES 认证 |
| **Pipeline-to-Pipeline** | 管道间通信（进程内高效传输） |
| **Native Plugin 增加** | 更多 Java 原生插件替换 Ruby 插件 |

逐条展开：**ECS Compatibility** 意味着 8.x 中各插件默认按 Elastic Common Schema 输出字段（比如地理信息进 `client.geo.*` 而不是裸 `geoip.*`）；**Security by Default** 指 8.x 安装包默认对 ES 通信启用 TLS + 账号认证，升级时最常见的"连不上 ES"都出在这里；**Pipeline-to-Pipeline**（8.7+）允许一个管道的输出经进程内队列路由给同进程的另一个管道，替代一部分"两条管道靠 Kafka 互联"的重型方案。

> **注意**：表中"Java Execution Engine/弃用 Ruby"的准确含义是——**JRuby 运行时依然是宿主**，被替换的是**插件层**：核心插件（beats、kafka、elasticsearch 等）陆续从 Ruby 重写为 Java 实现，管道执行的热路径更多落在 JVM Java 代码上。不要理解成"Logstash 不再是 Ruby 项目"：配置文件 DSL 依然是 Ruby 语法，插件生态依然有相当比例是 Ruby gems。

# 7. 一个最小的 Logstash 管道

```text
# logstash.conf
input {
  stdin { }
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
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "nginx-access-%{+YYYY.MM.dd}"
  }
  stdout { codec => rubydebug }
}
```

```bash
# 运行
bin/logstash -f logstash.conf
```

这 20 行覆盖了 Logstash 的全部核心概念：`stdin` 把一行键盘输入变成一个 Event（原始文本进 `message` 字段）；`grok` 用内置的 `%{COMBINEDAPACHELOG}` 模式把 message 拆成 `clientip`、`timestamp`、`method`、`response` 等字段；`date` 把拆出来的日志时间戳**纠正**为 `@timestamp`（否则事件时间是"进入管道的时间"而非"日志发生的时间"，Kibana 时间轴会整体漂移）；`geoip` 基于 `clientip` 补充地理位置块；output 阶段同一事件既写 ES（按天命名的 `nginx-access-YYYY.MM.dd` 索引）又用 `rubydebug` 打印到终端校验。运行前用 `bin/logstash -f logstash.conf -t` 校验配置合法性。

# 8. JRuby 实现的代价

选择 JVM/JRuby 换来的第一笔成本是**内存**：每个 Event 都是 JVM 堆上的对象，默认堆配置 1GB 起步，跑起来实际占用轻松数 GB；事件对象朝生夕灭、分配速率极高，堆给小就频繁 Full GC、吞吐骤降，给大则单次停顿变长——"吞吐还可以但延迟抖动不可控"是 JVM 管道的典型指纹。**第二笔成本是启动与变更**：JVM + JRuby 运行时初始化要几十秒，配置热重载需重新实例化全部插件、重新编译 grok 模式。**第三笔是吞吐天花板**：同等硬件下 Logstash 单节点吞吐明显低于 Vector/Fluent Bit，靠 `pipeline.workers` 多核并行和 batch 机制硬扛，单位吞吐的资源成本更高。缓解手段：堆给到 2~4GB（超过 31GB 会失去压缩指针收益）、加大 `pipeline.batch.size` 摊薄开销、把不该进中心管道的采集路径下沉给 Beats/Fluent Bit、优先选择已 Java 化的核心插件。

**设计考量**：明知代价仍选 JVM，是一次清醒的交换——**JDBC 驱动、Kafka 客户端、各类企业协议栈都是现成的 Java 生态资产**，用 JRuby 胶水把它们接进日志管道几乎零成本；而"单文件声明式 DSL + 逐条加工"的表达力，是纯 C/Rust 系轻代理难以提供的。Logstash 把自己定位成"ETL 引擎"而非"传输代理"，JVM 就是与之匹配的技术栈。

# 9. 小结

Logstash 是 Elastic Stack 中的**数据处理管道**：input/filter/output 三段插件化架构，把任意来源的数据规约成统一 Event，做解析、富化、路由后写往任意目的地。它是"ELK"里唯一可选的一环——**简单直通不需要它，复杂度上来时离不开它**。与 Fluent Bit/Fluentd/Vector 的差异本质是定位与实现语言的差异：Logstash 用 JVM 的重量换处理能力的全面和 Java 生态的复用，边缘轻采集永远优先交给 Go/C/Rust 系代理。8.x 的关键词是 ECS 默认、安全默认、核心插件 Java 化（JRuby 宿主地位不变）。判断是否引入 Logstash，只需回答一个问题：数据在到达存储之前，是否需要被"重新理解"。
