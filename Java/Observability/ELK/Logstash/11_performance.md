
# 1. 调优全景图

Logstash 的吞吐问题**几乎总是单点瓶颈**：Input 拿数据、Queue 缓冲、Filter 加工、Output 写入四段中，最慢的一段决定整条管道的上限。调优的正确姿势是先定位瓶颈段再对症下药，而不是把 batch、workers、堆内存一起往上加——那样只会让 GC 和内存风险叠加，吞吐却没动。

```text
性能调优维度:
┌────────────┬────────────┬────────────┬────────────┐
│  JVM 层     │  管道层     │  Filter 层  │  Output 层  │
├────────────┼────────────┼────────────┼────────────┤
│ 堆内存/GC   │ Worker 数   │ Grok 优化   │ ES 批量     │
│ 堆外内存    │ Batch 大小  │ Dissect 替代│ 连接池      │
│ 启动参数    │ Queue 类型  │ 条件过滤    │ 重试策略    │
└────────────┴────────────┴────────────┴────────────┘
```

一个判断框架：**Input 段慢**表现为 `events.in` 速率上不去而 workers 空闲；**Filter 段慢**表现为 CPU 高、queue 堆积、out 滞后于 in；**Output 段慢**表现为 queue 持续堆积、CPU 有余量、bulk 请求延迟高或 429/503 上升。三类处置手段完全不同，混用会互相掩盖。

# 2. JVM 调优

```bash
# config/jvm.options
-Xms2g -Xmx2g          # 堆内存相同（避免 resize 开销）
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:+DisableExplicitGC

# 堆外内存（Direct Memory）
# Kafka Input 使用堆外内存
-XX:MaxDirectMemorySize=512m
```

**堆大小的选择依据是"在途事件总量"，不是"日志总量"**。同时在管道里的事件数 ≈ `pipeline.batch.size × pipeline.workers × 管道数`，再乘上单个事件的平均大小和 filter 期间的字段膨胀（geoip、大量 mutate 会让事件膨胀 2-5 倍），剩下的给 ES 输出缓冲、PQ 内存预读缓冲和元数据。常见的 8 workers × 250 batch ≈ 2000 条在途，堆 2g 足够；堆给得过大（比如 16g）反而有害——Full GC 扫描空间更大、停顿更长。**`-Xms` 与 `-Xmx` 相同**是为了让 GC 行为可预测；另一条上限是**不要把堆设过 31g 左右**，超过后压缩指针（Compressed Oops）失效。

> **注意**：8.x 默认随包 JDK 已使用 G1GC，`-XX:+UseG1GC` 属于显式声明而非必须；`-XX:+DisableExplicitGC` 要谨慎——Direct Buffer 回收依赖 `System.gc()` 触发，禁用后 Kafka/NIO 用满堆外内存可能出现 OOM: Direct buffer memory。堆外内存合理预期是几百 MB，`jcmd <pid> VM.native_memory` 可看真实占用。

# 3. 管道 Workers 与 Batch

```yaml
# logstash.yml
pipeline.workers: 4            # Worker 线程（= CPU 核数）
pipeline.batch.size: 125       # 每批事件数（默认 125）
pipeline.batch.delay: 50       # 批等待时间(ms)
```

```text
Workers 计算:
  Workers = CPU 核心数 × 1-2
  过多 → 上下文切换开销 > 并行收益
  过少 → 无法充分利用 CPU

Batch 计算:
  batch.size 太小 → 频繁刷新 Output（ES 写入压力大）
  batch.size 太大 → 内存占用高，延迟增加
```

这两个参数是**耦合**的，要一起看。`workers` 决定"同时有多少批在飞"，`batch.size` 决定"每批装多少"，二者乘积决定在途事件数与内存需求。

`pipeline.batch.delay` 常被误解，它实际是**取批时的最大等待毫秒数**：worker 从队列 poll 时，在 delay 内凑不满 `batch.size` 就拿已有的量开工，是延迟与批量效率的旋钮——**低流量高时效**调小（10-20ms），**高吞吐**调大（50-100ms）。

**workers 的上限判断要看瓶颈类型**：CPU-bound（grok/ruby/json 密集）超过核数就是纯损耗；IO-bound（等 ES、等 Kafka）开到核数的 2-4 倍有意义，因为线程大部分时间在 park。但 workers 过大在 IO-bound 下也有新问题——ES 并发 bulk 过多触发线程池拒绝（429），看起来"CPU 不高但吞吐不涨"，此时要减 workers 或扩 ES。起步值：CPU-bound 用核数，IO-bound 用核数 × 2，压测微调。
> **设计考量**：多管道场景下 `pipeline.workers` 可按管道单独在 pipelines.yml 里设置，但所有管道的 workers **共享同一个进程 CPU 配额**。给采集管道 2、重加工管道 6 的意义是"相对优先级"，不是物理隔离；总量超过核数时仍然互相抢时间片。

# 4. Filter 优化

```ruby
# 1. 条件过滤（避免不必要处理）
if [type] == "nginx" {
  grok { match => { "message" => "%{COMBINEDAPACHELOG}" } }
}

# 2. Grok 锚定（^ 减少回溯）
grok { match => { "message" => "^%{COMBINEDAPACHELOG}" } }

# 3. Dissect 替代 Grok（分隔符明确时）
dissect { mapping => { "message" => "%{ts} [%{level}] %{msg}" } }

# 4. 提前删除不需要的字段
mutate { remove_field => ["@version", "beat", "input", "host"] }

# 5. 避免过多正则
# ✗: 5 个 Grok 规则
# ✓: 1 个 Grok + Mutate 字段操作
```

Grok 的性能问题不是"正则引擎慢"，而是**模式库拼接出的正则规模失控**。一个 `%{COMBINEDAPACHELOG}` 展开后是几十条子模式的合并，命中失败时 Oniguruma 需要回溯整串；无锚定还会让引擎在 message 的每个位置都尝试一次匹配。`^` 锚定把尝试起点固定为 0，失败即提前返回，是收益最大的单条优化；进一步可显式命名捕获（`%{IPORHOST:clientip}`）而不是把整段留给 `%{GREEDYDATA}`。

多个候选模式的正确写法是合并到一个块内：

```ruby
# ✗ 串行多个 grok 块：每条日志最坏情况跑多轮正则
grok { match => { "message" => "%{PATTERN_A}" } }
grok { match => { "message" => "%{PATTERN_B}" } }

# ✓ 单个 grok 块内多候选，命中即停（数组顺序 = 优先级，最常见的放前面）
grok {
  match => { "message" => [ "%{ACCESS_FORMAT}", "%{ERROR_FORMAT}" ] }
}
```

`remove_field` 在 filter 阶段看似没必要，但对下游有实际收益：字段越少，PQ 序列化字节越少、ES 的 `_source` 越小、mapping 字段爆炸风险越低。**耗时 filter 的顺序也影响吞吐**：`date`、`mutate` 这类轻量操作放前面，用条件把重活（geoip、ruby）限定到必要分支；`ruby` filter 是性能杀手，成本是每次调用跨 JRuby 与 Java 事件模型的数据搬运，能用 `mutate`/`translate`/`dissect` 表达的绝不写 ruby。

# 5. Output 优化

```ruby
output {
  elasticsearch {
    hosts => ["es1:9200", "es2:9200"]
    flush_size => 500               # 增大批量（旧版插件参数）
    idle_flush_time => 2            # 减少空刷新（旧版插件参数）
    sniffing => false               # 关闭嗅探（减少延迟）
    manage_template => false        # 手动管理模板（减少请求）
  }
}
```

8.x 中 elasticsearch output 的**批量大小主要由 `pipeline.batch.size` 决定**（一批 = 一个 bulk 请求），`flush_size` / `idle_flush_time` 属于早期版本的遗留参数，新插件里已不再是主要控制手段；真正值得调的是这几个：
```ruby
output {
  elasticsearch {
    hosts => ["https://es1:9200", "https://es2:9200"]
    index => "app-%{+YYYY.MM.dd}"
    http_compression => true        # 压缩 bulk 请求体，省带宽收益明显
    pool_max => 32                  # 每个主机的连接池上限（默认 32）
    sniffing => false               # 固定节点列表时关闭，去掉节点发现往返
    manage_template => false        # 模板/ILM 用 ILM API 或 CI 预置，避免每管道抢着写模板
    template_name => "app-log"
    pipeline => "app-ingest"        # 能把部分加工下沉到 ES ingest pipeline
    bulk_path => "/_bulk"
    action => "index"
  }
}
```

`sniffing => false` 的收益是"少一次节点列表查询 + 写入目标固定"，代价是不能自动感知拓扑变化，生产上通常保持 false 并把 hosts 写成协调节点列表，由 ES 侧负责路由。

> **注意**：ES 写入吞吐的天花板通常在 **ES 侧而不是 Logstash 侧**——shard 数、副本数、refresh_interval、写入线程池队列。如果 bulk 延迟高且 ES 返回 429，加 Logstash 的 workers 只会制造更多拒绝，正确动作是加大 `refresh_interval`（如 30s）、减少副本、扩数据节点，同时在 Logstash 侧降低并发、依赖重试退避。

# 6. 慢管道定位

定位慢管道靠 Logstash 自带的 Stats API，核心关注 `events.in/out`（累计值要取差分）、`queue.events_count`、`plugins.filters[].events` 与 `bulk_requests` 几组指标：

```bash
# 1. 整体：哪个管道堆积、in/out 速率差最大
curl -s localhost:9600/_node/stats/pipelines?pretty \
  | jq '.pipelines | to_entries[] | {id:.key, in:.value.events.in, out:.value.events.out,
      qsize:.value.queue.events_count, qbytes:.value.queue.queue_size_in_bytes}'

# 2. 插件耗时：哪个 filter 是热点（看 each 与 collections 的比值）
curl -s localhost:9600/_node/stats/pipelines?pretty \
  | jq '.pipelines.main.plugins.filters[] | {name:.name, id:.id, events:.events}'

# 3. CPU 高但吞吐低：抓热点线程栈
curl -s "localhost:9600/_node/hot_threads?human=true&ignore_idle_threads=true&threads=20"
```

读法上：`queue.events_count` 与 `queue_size_in_bytes` 是堆积的"果"，`each`/`collections` 是"因"——先看果确定哪条管道堵，再按 each 排序找堵点插件。

| 指标 | 含义 | 异常判读 |
|------|------|----------|
| `events.in` vs `events.out` | 进入与流出计数（累计值，取差分算速率） | 差值持续扩大 = 有丢弃或严重堆积 |
| `events.filtered` | 各 filter 处理计数 | 某 filter 计数为 0 = 条件分支写错 |
| `queue.events_count` | 队列堆积事件数 | 持续增长 = 下游段跟不上；PQ 下 >10000 告警 |
| `queue.queue_size_in_bytes` | 队列字节数 | 逼近 `queue.max_bytes` = 背压即将传到上游 |
| `plugins.filters[].events.each` | 该插件单事件平均耗时（纳秒） | 直接指出热点 filter |
| `plugins.filters[].events.collections` | 该插件引起的 GC 次数 | 比值异常 = 插件在制造垃圾对象 |
| `outputs[].bulk_requests.size/retries` | bulk 批量与重试次数 | retries > 0 = ES 侧有压力或 mapping 冲突 |

> **行业实践**：把 `_node/stats` 用 logstash-exporter（Prometheus）采集进指标库，做速率差分告警（in/out 速率长期背离、`queue.events_count` 趋势上涨、`bulk_requests.retries` > 0），比人工看面板可靠得多。

# 7. 压测方法

压测的目的是**在可控条件下比较配置差异**，不是复现生产流量。用 generator 输入灌真实日志行、输出丢弃，只测 LS 自身处理能力：

```ruby
# 生成器压测：用真实日志行灌管道，输出丢弃，只测 LS 自身处理能力
input {
  generator {
    lines => [
      '10.0.0.1 - - [08/Nov/2026:10:30:00 +0800] "GET /api/user HTTP/1.1" 200 45',
      '10.0.0.2 - - [08/Nov/2026:10:30:01 +0800] "POST /api/order HTTP/1.1" 500 120'
    ]
    count  => -1            # -1 = 无限生成（配合 timeout 跑固定时长）
    metric => "message"
  }
}
filter {
  grok { match => { "message" => "^%{COMBINEDAPACHELOG}" } }
}
output {
  stdout { codec => nil }   # 丢弃输出，避免终端 IO 成为瓶颈
}
```

```bash
# 跑固定时长后取两次采样差分，除以间隔秒数 = eps
timeout 300 bin/logstash -f bench.conf > /dev/null &
a=$(curl -s localhost:9600/_node/stats/events | jq .events.out); sleep 60
b=$(curl -s localhost:9600/_node/stats/events | jq .events.out)
echo "$(( (b - a) / 60 )) eps"
```

三点方法学要求：**变量只有一个**（一次只改 workers 或只改 batch）；**预热后再采样**（JRuby JIT 与正则编译需要十几秒，头一分钟数据偏低）；**瓶颈要外置排除**——压解析能力时输出用 `stdout { codec => nil }`，不要带 ES。

# 8. 硬件选型：CPU 还是 IO

| 瓶颈类型 | 典型管道特征 | 硬件优先项 | 参数配合 |
|----------|-------------|-----------|----------|
| CPU-bound | grok/ruby/json/geoip 密集，CPU 打满 | 更多核心、高主频 | workers = 核数；优化 grok |
| IO-bound | Kafka input + ES output 直通，CPU 空闲但吞吐低 | 网卡带宽、SSD | workers > 核数；http_compression |
| 内存敏感 | 事件大（堆栈日志、多字段）、batch 高 | 大堆（但不超 31g） | 降 batch；提前 remove_field |
| PQ 落盘重 | 开了 persisted 且下游常抖动 | SSD（顺序写 IOPS） | checkpoint 保持 1024 |

经验配比：**4C8G 是单条中型管道的起步规格**，8C16G 能稳定吃 2-3 万 eps 的 grok 负载。堆给物理内存的一半左右，剩下留给 OS page cache（PQ 页文件读写受益）与堆外缓冲。Logstash 是**无状态可水平扩展**的服务，单机目标是"单节点吞吐/成本比最高"——CPU 超过 70% 就应扩节点而不是继续加压。

# 9. 生产调优配置

```yaml
# 高性能配置
pipeline.workers: 8
pipeline.batch.size: 250
pipeline.batch.delay: 25
queue.type: persisted
queue.max_bytes: 4gb
queue.checkpoint.writes: 2048
```

# 10. 性能基准

```text
环境: 4C8G, Filebeat → Logstash (Grok+GeoIP) → ES

| 配置                 | 吞吐量     | CPU  |
|---------------------|-----------|------|
| Workers=2, Batch=125 | 5000 eps  | 45%  |
| Workers=4, Batch=250 | 12000 eps | 75%  |
| Workers=8, Batch=500 | 18000 eps | 90%  |

eps = events per second (每秒事件数)
```

量级参考（1KB 事件、中等复杂度，仅供容量规划粗估）：纯转发无加工可到 5-10 万 eps/4C；带 1-2 个 grok 是 1-3 万；带 geoip + translate + ruby 掉到几千。**跨量级下降基本都是正则与 ruby 造成的**。上面的表也说明"翻倍 workers ≠ 翻倍吞吐"，收益在 70%-80% CPU 之后迅速衰减。

# 11. 常见瓶颈

| 瓶颈 | 现象 | 解决 |
|------|------|------|
| Grok 正则回溯 | CPU 100%，吞吐低 | 锚定 ^、用 Dissect |
| ES 写入慢 | queue 堆积 | 增大 batch、扩 ES 节点、加 refresh_interval |
| JVM GC 频繁 | 吞吐波动大 | 增大堆、换 G1GC |
| PQ 磁盘慢 | 吞吐下降 | 换 SSD、增大 checkpoint 间隔 |
| ruby filter 滥用 | 单插件 each 耗时远超其他 | 改用 mutate/translate/dissect |
| workers 过多 | CPU 不高但吞吐不涨、ES 429 | 回落到核数×1-2，扩 ES |

# 12. 小结

| 层次 | 首要动作 | 常见误区 |
|------|----------|----------|
| 定位 | 先分清 in/queue/out 与插件 each 耗时，锁定瓶颈段 | 不定位就同时改多个参数 |
| JVM | 按在途事件数算堆；G1 + 固定 Xms/Xmx | 堆越大越好（停顿更长、压缩指针失效） |
| 管道 | workers 按瓶颈类型取核数或核数×2，batch 与内存一起调 | workers 超核数当万能药 |
| Filter | grok 锚定、候选数组、Dissect 替代、删 ruby | 串行多个 grok 块、GREEDYDATA 满天飞 |
| Output | http_compression、关 sniffing、批量对齐 batch.size | ES 有压力时继续加 Logstash 并发 |
