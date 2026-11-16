# 1. 事件处理流水线

Logstash 的一切围绕一个词：**Pipeline（管道）**。一个管道就是"input → filter → output"的一条完整数据流，进程可以同时运行多个管道（`pipelines.yml` 定义）。理解执行模型的关键是明白：**三段之间不是"一条线程从头跑到尾"，而是通过队列与批（batch）解耦的并发结构**——input 只管生产事件，filter+output 以 worker 为单位成批消费事件。

```text
Pipeline Worker Thread 1:
  Input Plugin ──→ Filter Chain ──→ Output Plugin
       │               │               │
  ┌────▼────┐    ┌─────▼─────┐    ┌────▼────┐
  │  Queue  │───→│  Grok     │───→│   ES    │
  │ (Memory │    │  Mutate   │    │  Kafka  │
  │  or PQ) │    │  GeoIP    │    │  File   │
  └─────────┘    └───────────┘    └─────────┘

Pipeline Worker Thread 2: (同上，并行处理)
Pipeline Worker Thread N: (同上)
```

图中的 filter chain 是**同一个 worker 线程串行执行的**：一个 batch 内的事件依次穿过 grok、mutate、geoip……所以"把昂贵的 filter 放前面还是放后面"会影响 worker 的占用形态。而**多个 worker 之间完全并行**，每个 worker 持有自己的 batch。output 阶段则由插件内部线程池（如 elasticsearch output 的 bulk 线程）异步化，worker 提交完 batch 就回去处理下一批。

# 2. 线程模型

```yaml
# logstash.yml
pipeline.workers: 4            # Worker 线程数（默认 CPU 核数）
pipeline.batch.size: 125       # 每批处理事件数
pipeline.batch.delay: 50       # 批处理等待时间(ms)
```

```text
Input Thread
  │
  │ 从数据源读取 → 封装为 Event → 放入 Queue
  │
  ▼
┌──────────────────────────────────────────────┐
│                  Queue                        │
│  (Memory Queue 或 Persistent Queue)           │
│  ┌──────┬──────┬──────┬──────┐              │
│  │Batch │Batch │Batch │Batch │              │
│  └──────┴──────┴──────┴──────┘              │
└──────────────────────────────────────────────┘
  │         │         │         │
  ▼         ▼         ▼         ▼
Worker 1  Worker 2  Worker 3  Worker 4
  │         │         │         │
  ▼         ▼         ▼         ▼
Filter → Output   (每个 Worker 独立处理)
```

线程数量由三个参数决定，各自对应不同的瓶颈。**`pipeline.workers`** 是 filter+output 的消费线程数，默认等于 CPU 核数：它是 CPU 密集（重 grok）时的扩展旋钮；纯 IO 密集时可以调大于核数（ES 慢、grok 轻时给 2× 核数），因为 worker 大量时间在等 output 的 future。**`pipeline.batch.size`** 是每个 worker 一次拉取的事件数：太小则 ES bulk 请求碎、每批固定开销摊不薄；太大则内存驻留多、GC 压力上升，且一批失败的粒度变大。**`pipeline.batch.delay`** 是队列不足时凑批的等待时间，低流量下用一点延迟换批量大小。

> **注意**：worker 数并非越多越好。每个 worker 都会实例化一份 filter 插件的**副本**（每个 grok 有独立的 pattern 缓存、每个 ruby filter 有独立的解释上下文），worker 翻倍意味着插件状态与内存占用也近似翻倍。判断 CPU 是否真被吃满，看 `_node/stats` 里 `events.in_flight`（队列中待处理事件数）与 `events.queued`：`in_flight` 持续高说明 output 慢、worker 在等；`queued` 持续高说明 worker 不够或 filter 太重。

## 2.1 中央队列与直写管道：两种拓扑

早期 Logstash 是"全局单管道 + 一条中央队列"的形态；8.x 的多管道架构下，**每个管道拥有独立的队列与 worker 组**，管道之间默认互不影响——A 管道被 ES 卡死不拖累 B 管道。管道之间还能用 `pipeline` input/output 插件（8.7+ 提供的 pipeline-to-pipeline）经进程内队列直接传递事件，省掉"管道 A 写 Kafka、管道 B 再消费 Kafka"的序列化与网络往返。

```text
# pipelines.yml —— 一进程双管道
- pipeline.id: nginx
  path.config: "/etc/logstash/conf.d/nginx/*.conf"
  pipeline.workers: 4
- pipeline.id: app-logs
  path.config: "/etc/logstash/conf.d/app/*.conf"
  pipeline.workers: 2
```

# 3. Event 模型

**Event 是管道里流动的唯一对象**，本质是一个包装了 JSON 文档的 Ruby/Java 混合体：有 `@timestamp`、`@version` 两个保留字段，其余是任意嵌套的键值树。

```ruby
# Logstash Event 结构
{
  "@timestamp" => 2024-01-15T10:30:00.000Z,
  "@version" => "1",
  "message" => "192.168.1.1 - - [15/Jan/2024:10:30:00 +0800] GET /api/user HTTP/1.1 200 45",
  "host" => "web-server-01",
  "path" => "/var/log/nginx/access.log"
}

# 经过 Grok + GeoIP 处理后:
{
  "@timestamp" => 2024-01-15T02:30:00.000Z,
  "clientip" => "192.168.1.1",
  "method" => "GET",
  "request" => "/api/user",
  "httpversion" => "1.1",
  "response" => 200,
  "bytes" => 45,
  "geoip": {
    "country_name" => "China",
    "city_name" => "Beijing",
    "location" => { "lat" => 39.9, "lon" => 116.4 }
  }
}
```

对照前后两个状态能读出事件模型的三条规则。其一，**input 只是把原始内容塞进 `message`**，字段是 filter"雕刻"出来的——`grok` 不替换 message，而是往里加键；`date` 会把解析出的时间**改写** `@timestamp`（注意处理后 `@timestamp` 从 10:30 变成了 02:30——日志里的 +0800 被换算成 UTC 存储，这是 ES 侧时间处理的根基）。其二，存在一个**不可见的 `@metadata` 树**：写进 `@metadata` 的字段（如路由用的 `@metadata[route]`）不会被任何 output 持久化，是"管道内变量"的官方出口，比用完就删的临时字段干净得多。其三，Event 附带 `tags` 数组作为状态标记通道（`_grokparsefailure` 等失败 tag 由此驱动后续条件分支）。

**设计考量**：字段引用语法 `[field][subfield]`（如 `[geoip][city_name]`）和 ECS 风格的 `geoip.city_name` 是同一棵树的不同写法；配置里出现点号引用时要小心——`"a.b" in field` 这种历史写法取的是**顶层键名叫 "a.b" 的字段**，而不是嵌套字段，两者混用是新手字段"取不到值"的头号原因。

# 4. Queue 机制

| Queue 类型 | 持久化 | 性能 | 适用场景 |
|-----------|--------|------|---------|
| **Memory**（默认） | ✗ 重启丢失 | 极高 | 开发/可容忍丢失 |
| **Persistent Queue** | ✓ 磁盘持久 | 中（有开销） | 生产环境 |

```yaml
# Queue 配置
queue.type: persisted           # persisted / memory
path.queue: /var/lib/logstash/queue
queue.max_bytes: 2gb            # PQ 最大磁盘
queue.checkpoint.writes: 1024   # 每 N 次写入做 checkpoint
```

队列的选型本质是**回答"丢不丢得起数据"**：内存队列下事件只在 JVM 堆里，进程崩溃、重启时未处理的事件全丢；持久队列（PQ）把事件序列化到磁盘页文件，input 确认"入盘"才算投递成功，代价是吞吐下降（实测常见 20%~50% 折损）与磁盘 IO 依赖。

## 4.1 Memory Queue

```text
Memory Queue: 纯内存，最快
  - 大小 = pipeline.batch.size × workers
  - 背压机制: 队列满 → 输入暂停
```

值得强调的是容量公式的含义：**内存队列不是无限深的水池**，容量近似"每个 worker 恰好持有一个 batch"——input 一次最多交出 `workers` 个 batch，必须等 worker 消费完才生产下一批。这就是背压（backpressure）：下游慢会原路把压力顶回 input，beats input 让 Filebeat 减速、file input 干脆暂停读文件。数据不会丢在队列里，但会"堵在源头"；源头若是会滚动删除的日志文件，堵太久就等于丢数据——这正是内存队列在生产里致命的场景。

## 4.2 Persistent Queue

```text
PQ 结构:
  ┌──────────────────────────────────┐
  │ Page File 1 (64MB)              │
  │  [Event1][Event2]...[EventN]    │
  ├──────────────────────────────────┤
  │ Page File 2 (64MB)              │
  ├──────────────────────────────────┤
  │ ...                              │
  ├──────────────────────────────────┤
  │ Checkpoint File                  │
  │  已处理的 Page + Offset          │
  └──────────────────────────────────┘
```

PQ 是一个单生产者单消费者的磁盘环形结构：事件按追加写进入 64MB 的 page file，worker 消费后推进 checkpoint（每 `queue.checkpoint.writes` 次写做一次），老 page 满了且全部确认后才删除。**容量语义要分清**：`queue.max_bytes`（默认 1024mb）控制的是 PQ 能占多少盘；PQ 写满时 input 被暂停，压力继续回推到源头。两个必配的运维项：队列目录（`path.queue`）要放独立磁盘并监控水位；checkpoint 间隔调小可降低崩溃后的重复量，但会增加 IO。

> **行业实践**：凡是"下游是 ES 且 ES 可能变慢（写入拒绝、mapping 冲突、集群黄红）"的生产管道，直接开 PQ；把 `queue.max_bytes` 设成"能容忍的故障时长 × 平均写入速率"，再对队列盘做 80% 告警。纯内存队列只留给开发调试和"日志可以丢"的边缘场景。

# 5. 死信队列（DLQ）

```yaml
# DLQ 配置
dead_letter_queue.enable: true
dead_letter_queue.max_bytes: 1gb
path.dead_letter_queue: /var/lib/logstash/dead_letter_queue
```

```text
Pipeline 处理失败的事件 → 自动写入 DLQ
  保留原始 Event + 失败原因 + 时间戳
  可用 `input { dead_letter_queue {} }` 重新处理
```

DLQ 解决的是队列解决不了的另一个问题：**不是"来不及写"，而是"写不进去"**。最典型的触发是 mapping 冲突——ES 里 `bytes` 已被动态映射成 long，下一条日志的 `bytes` 是 `"1.2e3"` 字符串，ES 以 `mapper_parsing_exception` 拒绝该文档。没有 DLQ 时该事件被 elasticsearch output 重试若干次后丢弃；开启 DLQ 后它带着失败原因原样封存，等修好之后用一个专门的回放管道重灌：

```text
input {
  dead_letter_queue {
    path => "/var/lib/logstash/dead_letter_queue"
    commit_on_success => true      # 回放成功才从 DLQ 删除
  }
}
output {
  elasticsearch { hosts => ["https://es1:9200"] }
}
```

> **注意**：DLQ 只对 elasticsearch、kafka 等"逐个文档级失败"的 output 生效，并且**默认关闭**——8.x 也依然要手工打开 `dead_letter_queue.enable`。它是最后一道保险丝，不是通用备份方案：`dead_letter_queue.max_bytes` 写满后按最旧优先**丢弃**，回放管道要尽早跑、跑完就关。

# 6. Plugin 加载

```text
Logstash 启动:
  1. 读取 pipeline 配置
  2. 动态加载需要的 Plugin (Ruby Gems / Java JARs)
  3. 每个 Plugin 在独立隔离环境中运行
  4. Input Thread → Queue → Filter Workers → Output

Plugin 目录:
  logstash-core/lib/logstash/inputs/    → 内置 Input
  logstash-core/lib/logstash/filters/   → 内置 Filter
  vendor/bundle/jruby/*/gems/           → 已安装插件
```

"独立隔离环境"是理解插件行为的关键：JRuby 给每个插件做类加载器隔离，Ruby 插件之间不共享全局状态；对 filter 而言还有一层——**每个 worker 线程持有 filter 插件的独立实例**，所以 ruby filter 里的实例变量是"每 worker 一份"而非全局单例，跨 worker 累计计数之类的想法在这里都会落空。input/output 通常只有单实例（并发由插件内部线程池提供）。

```bash
# 插件管理
bin/logstash-plugin list                    # 列出已安装
bin/logstash-plugin install logstash-input-kafka
bin/logstash-plugin update logstash-output-elasticsearch
```

> **注意**：`logstash-input-kafka`、`logstash-input-hawq` 等老 Ruby 插件在 8.x 已被官方废弃，kafka 的替代品是 **`logstash-integration-kafka`**（Java 实现，同时提供 kafka input/output 与 consumer 能力）。用 `bin/logstash-plugin list --verbose` 可以看到每个插件的版本与废弃警告；离线环境用 `bin/logstash-plugin prepare-offline-pack` 打离线包是标准做法。

# 7. 阻塞 IO 与线程池

插件按"是否阻塞 worker"分成两类，这决定了整套性能模型。**非阻塞类**：elasticsearch、kafka 等 Java 输出插件把 bulk 请求交给内部线程池异步执行，worker 只负责"提交 batch + 挂 future"，然后立刻消费下一批——IO 等待不占用 worker，这就是为什么 IO 重时可以开 `workers > CPU 核数`。**阻塞类**：file input 的读、JDBC input 的查询、老 Ruby 系 output（如早期的 http）在 worker 或 input 线程上同步等待，等 IO 就是干等。

```text
worker 生命周期（非阻塞 output）:
  取 batch → 串行跑 filter chain → 提交 bulk future → 立即取下一 batch
                                        │
                                output 线程池: 攒满 flush_size 或
                                idle_flush_time 到期 → 发 bulk 请求
```

**设计考量**：这套模型解释了 Logstash 两个经典现象——"ES 慢但 worker 空闲率高"（future 在 output 队列排队，真正卡的是 bulk 队列与 `in_flight`）与"grok 很重时 CPU 打满但吞吐上不去"（filter 串行同步，CPU 就是瓶颈，只能加 workers 或加机器）。调优第一步永远是先分清瓶颈类型：看 CPU 曲线与 `events.filter_duration_in_millis`、`events.queue_push_duration_in_millis` 的相对占比，再决定动 workers、动 batch size 还是动输出端。

# 8. 小结

Logstash 的执行模型可以压缩成一句话：**input 生产事件入队列，N 个 worker 成批拉取、串行过 filter 链、把输出异步甩给线程池**。`pipeline.workers/batch.size/batch.delay` 三个旋钮分别控制并行度、批量与凑批延迟；队列二选一——内存队列快但会丢，持久队列落盘保数据、用吞吐换安心；写不进 ES 的坏文档交给 DLQ 存档回放。Event 是一棵 JSON 树，`@timestamp` 可被 date 纠正、`@metadata` 是管道内变量、`tags` 承载失败状态。插件以 worker 为单位隔离实例化，阻塞与否决定 worker 该不该多开——先分清 CPU 瓶颈还是 IO 瓶颈，再谈调参。
