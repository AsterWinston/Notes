
# 1. 为什么需要 PQ

Logstash 默认的 Memory Queue **存活在 JVM 堆内**，事件从 Input 进入队列、被 Filter Worker 取走、再交给 Output，全程都是内存对象。这意味着：进程崩溃、OOM 被 kill、容器被驱逐——队列里排的事件和 Worker 手里未 flush 的批次会**瞬间全部消失**。即使上游是 Filebeat 也不能兜底：Filebeat 只保证"成功交给 Logstash 的 TCP 连接"，交出去之后注册表偏移就前进了，Logstash 侧再丢它不会重发。
```text
Memory Queue（默认）：
  Input → Memory Queue → Filter → Output
               │
               ▼ 进程重启/崩溃
           数据丢失 ✗

Persistent Queue：
  Input → Persistent Queue (磁盘) → Filter → Output
               │
               ▼ 进程重启
           数据从 Checkpoint 恢复 ✓
```
PQ（Persistent Queue，持久化队列）的思路很直接：**把 Input 与 Filter 之间的缓冲层从内存搬到磁盘**，事件序列化后追加写（append-only）落到页文件，落盘确认后 Input 才认为写入成功；重启时从 Checkpoint 位点继续消费。它把 Logstash 从"单点故障即丢数据"变成"单点故障只是延迟"。代价是吞吐下降、磁盘 IO 增加，以及多了一套需要监控的磁盘目录。

## 1.1 Memory Queue 与 Persistent Queue 对比

| 维度 | Memory Queue（默认） | Persistent Queue |
|------|---------------------|------------------|
| 存储位置 | JVM 堆内 | 磁盘页文件（`path.queue`） |
| 进程崩溃/重启 | 队列内数据全丢 | 从 Checkpoint 恢复 |
| 可缓冲时长 | 受堆大小限制，分钟级 | 受 `queue.max_bytes` 限制，可到小时级 |
| 吞吐 | 高（纯内存拷贝） | 约损失 10%-15%（序列化+磁盘写） |
| GC 压力 | 事件对象在堆内，量大时 GC 明显 | 事件不落堆，堆压力小 |
| 批读取 | Worker 直接从内存队列取批 | 默认先读入内存缓冲，`queue.drain` 可改直读磁盘 |
| 运维成本 | 低 | 要监控队列目录磁盘水位 |
| 适用 | 开发调试、可容忍丢失的日志 | 生产、日志即业务数据（审计、计费） |

> **设计考量**：PQ 改变的是"在途数据的持久性"，不改变投递语义。事件仍然按 **至少一次（at-least-once）** 投递——崩溃恢复时最后一个未 Checkpoint 的批次会被重放，下游若要求不重复，必须自己保证幂等（写入 ES 时显式指定 `_id` 是最常用的手段）。

# 2. PQ 配置

```yaml
# logstash.yml（也可按管道写进 pipelines.yml，粒度更细）
queue.type: persisted               # memory（默认）/ persisted
path.queue: /var/lib/logstash/queue # 队列目录，需提前创建并授权给 logstash 用户
queue.max_bytes: 2gb                # PQ 最大磁盘占用（默认 1024mb）
queue.max_events: 0                 # 不限制事件数（0 = 仅限制字节）
queue.checkpoint.writes: 1024       # 每 1024 次写入做 checkpoint（默认值）
queue.checkpoint.interval: 1000     # 强制 checkpoint 间隔(ms)，与 writes 先到先触发
queue.checkpoint.retry: true        # 磁盘写失败时重试 checkpoint 而不是直接崩
queue.page_size: 64mb               # 单个页文件大小（默认 64mb）
queue.drain: false                  # true = Filter 直接从磁盘读，重启零丢失但吞吐更低
```

各参数的关系值得捋一遍：`queue.max_bytes` 是**总闸门**，队列写满后不再接受新事件（表现为背压，见 5.1）；`queue.page_size` 决定文件切换粒度，完全消费完的旧页会被删除回收，所以 `path.queue` 目录的实际占用在 `0 ~ max_bytes` 之间浮动，不是预分配的。`queue.drain: true` 是"持久性优先"开关：默认模式下队列内容会预读进一批内存缓冲，崩溃时这一小段缓冲仍会丢；drain 模式砍掉这层缓冲、每次直接读盘，吞吐再降一截，配合 `checkpoint.writes=1` 可做到接近"零丢失重启"。

> **注意**：`queue.type` 属于启动期设置，`config.reload.automatic` 热加载**不会**应用它的变更，从 memory 切到 persisted 必须重启进程。另外队列目录不要放在 NFS 等共享网络盘上，fsync 语义不可靠，可能出现 checkpoint 与页文件不一致的损坏。

# 3. PQ 内部结构

```text
/var/lib/logstash/queue/
  ├── checkpoint             # Checkpoint 文件（标记已处理位置）
  └── page.0                 # Page File（64MB × N）
      page.1
      page.2
```

```text
每个 Page File 结构:
  ┌──────────────────────────────────────┐
  │ Header (序列号, 创建时间)             │
  ├──────────────────────────────────────┤
  │ Event 1 (序列化后的完整 Event)        │
  │ Event 2                              │
  │ ...                                  │
  │ Event N                              │
  └──────────────────────────────────────┘
```

结构上它是一个**单生产者多消费者的磁盘环形队列**：Input 线程只写"最后一页"（写满即轮转出新页），各 Filter Worker 从"最老的页"按批读取。事件会被完整序列化（含 `@timestamp`、`tags`、所有字段），估算 `queue.max_bytes` 要按序列化后的大小算。多管道场景下每条管道有各自独立的队列子目录；一页的全部事件被消费、且对应批次得到 Output 确认后，该页才允许删除。

# 4. Checkpoint 机制

```text
Checkpoint 记录:
  - 每个 Page File 的读取进度
  - 已确认写入 Page File 的序列号

写入流程:
  1. Event 追加到当前 Page File
  2. 达到 checkpoint.writes 阈值 → 更新 checkpoint
  3. 崩溃重启 → 从上次 checkpoint 位置恢复

注意:
  - checkpoint 不是每条都写（性能优化）
  - 崩溃可能丢失最后几个未 checkpoint 的事件
  - checkpoint.writes=1 每条都写（最安全，最慢）
```

Checkpoint 的本质是"**已经完整走完管道、被 Output 确认的序列号水位线**"。它由两个条件触发更新：写入次数达到 `queue.checkpoint.writes`，或距上次 checkpoint 超过 `queue.checkpoint.interval` 毫秒——**先到先触发**，所以低峰期不会因没攒够 1024 条就迟迟不推进位点。恢复流程：重启后读 checkpoint 文件，跳过水位线之前的已确认事件，从水位线之后继续投递，未确认的批次会被**重新完整执行一遍** Filter 和 Output。

由此有两个推论：崩溃损失上界是"最后一个 checkpoint 周期内已入队但未走完管道的事件"，窗口很小；恢复必然伴随少量重复，这正是 at-least-once 的来源。把 `checkpoint.writes` 调成 1 不会变成 exactly-once，只是把"丢"的窗口压到接近零。

> **注意**：checkpoint 推进意味着旧页可删除，但删除依赖 Output 确认。若 ES 网络抖动导致批次反复重试，队列位点会长时间不动、磁盘占用持续爬升——这是"队列满"最常见的成因，而不是流量真的翻倍。

# 5. PQ 性能影响

```text
性能基准（1000 events/s, 1KB/event）:
  Memory Queue: 100% (基准)
  PQ (checkpoint=1024): ~85%
  PQ (checkpoint=1): ~60%

建议:
  - 生产环境开启 PQ（数据安全 > 性能）
  - checkpoint.writes=1024（平衡性能与安全）
  - 使用 SSD 减少磁盘延迟
```

损失的来源有两块：**序列化/反序列化**（进出队列各一次，CPU 开销）和**磁盘顺序写**（SSD 上通常不构成瓶颈，机械盘上会明显拖尾）。上面的比例是"1KB 事件"的量级参考，事件越大，序列化占比越高。优化顺序：先用 SSD、再保持默认 checkpoint 参数、最后才动 `page_size`。`queue.drain: true` 额外牺牲吞吐换取重启窗口零丢失，除非日志承担计费/审计职责，一般不开。

## 5.1 queue.max_bytes、磁盘水位与背压

`queue.max_bytes` 是 PQ 的容量上限，行为模型是：**队列写满 → Input 的写入调用被阻塞 → 上游感受到背压**，而不是丢弃。这个"背压沿链路向上传导"的链条是理解 Logstash 过载行为的关键：

```text
ES 写不进去（mapping 爆炸 / 429 / 集群 red）
   ▼ Worker 批次无法确认，队列位点停滞
PQ 持续堆积，目录逼近 queue.max_bytes
   ▼ Input 阻塞
beats input 不再从 socket 读取
   ▼ TCP 接收缓冲区打满
Filebeat 内存队列满 → harvester 停止读新文件 → 采集延迟出现
（若上游是 Kafka input：暂停 poll，consumer lag 上涨，Kafka 替 LS 抗住堆积）
```

由此可以推出容量规划的"水位线"做法：`queue.max_bytes` 不是越大越好，它应当换算成"**能吸收多久的下游故障**"——峰值速率 × 可容忍故障时长 × 序列化后事件大小，再对照磁盘可用空间取保守值；同时给队列目录占用设两级告警（例如 75% 提醒、90% 紧急）。

> **行业实践**：中心集群用 "Filebeat → Kafka → Logstash → ES"，长时堆积交给 Kafka 分区（TB 级、可回放），Logstash 的 PQ 只作分钟级二次缓冲，`queue.max_bytes` 设 2-4gb 即可；直连 Beats 的中小部署才需要把 PQ 当主缓冲，按小时级故障容量配。

## 5.2 队列满的行为：backpressure 与 reject

要区分两种"满"：**PQ 满**没有"拒绝/丢弃事件"的选项，唯一行为就是阻塞输入（backpressure），这是刻意保守——管道层无权替用户决定丢数据；想"主动丢"只能在 Input/Filter 用条件降级（如高峰期只转发 error 级别）。而 **DLQ 满**是例外，提供了显式策略开关 `dead_letter_queue.queue.full.behavior: reject`（默认，拒绝新死信并记日志）或 `syslog`（降级写应用日志）。设计取向很清楚：队列满靠背压，死信满才允许丢弃。

# 6. 死信队列（DLQ）

```yaml
# logstash.yml
dead_letter_queue.enable: true
dead_letter_queue.max_bytes: 1gb
path.dead_letter_queue: /var/lib/logstash/dead_letter_queue
```

```text
Output 写入失败（如 ES 不可达）：
  1. 重试（retry）N 次
  2. 仍失败 → 写入 DLQ
  3. DLQ 存储原始 Event + 失败原因 + 时间戳
```

DLQ 解决的是 PQ 管不到的另一类丢失：事件**完整地走完了管道**，只是最后写 Output 失败（典型是字段类型与 mapping 冲突这类"毒数据"——ES 对整批 bulk 返回 400，重试不会成功）。开启 `dead_letter_queue.enable` 后，elasticsearch 输出插件会把这类永久失败的事件连同原因写入 DLQ 目录，不再让一颗毒丸卡死整个批次。有两个硬性前提常被忽略：**DLQ 依赖 `queue.type: persisted`**（复用 PQ 的磁盘队列基础设施，memory 队列下降级为只记日志）；DLQ 主要对 elasticsearch 这类支持死信的 output 插件生效。与 PQ 的分工一句话说清：**PQ 防"进程死"（在途事件丢失），DLQ 防"数据坏"（写不进下游的事件被扔掉）**。

## 6.1 DLQ 读回

```ruby
# dlq_read.conf
input {
  dead_letter_queue {
    path => "/var/lib/logstash/dead_letter_queue"
    commit_offsets => true        # 处理后标记位点，避免重复消费
    clean_consumed => true        # 处理完删除（运维语义，物理清理见下）
  }
}

filter {
  # 可选：修复 Event
  mutate { add_field => { "reprocessed" => true } }
}

output {
  elasticsearch {
    hosts => ["http://es:9200"]
    index => "recovered-%{+YYYY.MM.dd}"
  }
}
```

读回操作要点：DLQ 内容已经序列化过，input 插件会原样还原事件，常见做法是把读回单独跑一个临时进程（`bin/logstash -f dlq_read.conf`），而不是常驻主管道；`commit_offsets => true` 必设，否则位点不推进、反复重读同一批。另外 DLQ input 只推进消费位点，**不会替你删除磁盘上的旧段文件**，物理清理要停管道、确认重放完毕后人工删目录内容。

> **注意**：DLQ 有数据本身就是告警信号而不是待办清单——如果每天产生死信说明 mapping/模板治理有问题，正确动作是修上游格式或索引模板，而不是天天重放。

# 7. PQ + DLQ 组合

```text
最佳实践:
  1. 生产环境开启 PQ（防止进程崩溃丢数据）
  2. 生产环境开启 DLQ（防止 ES 不可用丢数据）
  3. 监控 PQ 大小（超过阈值告警）
  4. 定期检查 DLQ（有数据说明下游异常）
```

把两者放进同一个事件生命周期里看，各自的位置一目了然：

```text
            ┌─────────────── 进程崩溃 → 从这里恢复（PQ 兜底）
            ▼
Input ──→ Persistent Queue ──→ Filter Workers ──→ Output
                                    │                 │
                                    │ tag 失败         │ 永久写失败（毒数据）
                                    ▼                 ▼
                              隔离索引/单独管道    Dead Letter Queue
                              （解析失败，不是丢失）  （重放工具读回）
```

三层防线对应三类故障：PQ 对进程级故障，DLQ 对输出级永久失败，tag/隔离索引对解析级失败。运维分别盯指标：PQ 看目录水位与堆积（`queue.events_count`），DLQ 看是否非空（稳态应为空），隔离索引看 parse failure 比例。

> **行业实践**：审计、计费类日志标配 PQ + DLQ，配合 ES `_id` 幂等写入；纯排障的应用日志可只开 PQ；debug 级可丢弃流量两者都不开，Filebeat 直接进 Kafka 更省资源。

# 8. 什么时候该开 PQ

判断标准不在"日志重不重要"，而在**这条链路的缓冲和重放能力由谁提供**：

| 场景 | 是否开 PQ | 理由 |
|------|-----------|------|
| Beats 直连 Logstash，日志不可丢 | 开 | 链路上只有 LS 一个缓冲点，丢了真丢 |
| 上游有 Kafka | 建议开但容量可小 | Kafka 抗长时堆积，PQ 做分钟级缓冲 |
| JDBC/File 定时批量入 ES | 开 | 重跑成本高，断点续传语义靠它 |
| 纯排障 debug 日志，量大可丢 | 不开 | 吞吐和成本优先 |

一个常被问到的边界："开了 PQ 是不是就万无一失"——不是，`path.queue` 所在磁盘成为新的单点，合规场景要把"重放上游原始日志"写进预案。

# 9. 小结

| 主题 | 关键结论 |
|------|----------|
| 定位 | PQ 把 Input→Filter 的缓冲从 JVM 堆搬到磁盘，进程崩溃后从 Checkpoint 恢复 |
| 语义 | at-least-once：崩溃窗口少量丢失（取决于 checkpoint），恢复少量重复（下游需幂等） |
| 核心参数 | `queue.type: persisted`、`queue.max_bytes`、`queue.checkpoint.writes`（默认 1024）、`queue.drain` |
| 满时行为 | PQ 满 → 阻塞 Input（背压沿 Beats/Kafka 向上传导）；DLQ 满 → `reject` / `syslog` 可选 |
| DLQ 关系 | PQ 防进程死、DLQ 防数据坏；DLQ 依赖 PQ 开启；读回用 `dead_letter_queue` input 单独跑 |
| 何时开 | 无 Kafka 缓冲且数据不可丢 → 开；有 Kafka → 小容量；可丢 → 不开 |
