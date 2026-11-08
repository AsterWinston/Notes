本篇两部分：第一部分是可扫读的高频考点清单；第二部分 15 题按面试真实问法展开，覆盖倒排索引、写入流程、refresh/flush/translog、深分页、聚合误差、分片规划、脑裂、ILM 等必考项。

# 1. 知识点

- **核心概念** — Cluster（多节点组成的集群）> Node（一个 ES 进程）> Index（文档集合，类比表）> Shard（Lucene 实例，Primary+Replica，水平扩展单位）> Document（JSON 数据单元）
- **倒排索引** — 词项(term)→  Posting List(docId 列表) 的映射，由 FST(词典，常驻内存) + 倒排表(磁盘，跳表/PFOR 压缩)组成；全文检索 O(词项定位+链表求交)，不用扫全部文档；正排是 docId→字段值
- **text vs keyword** — text 分词后进倒排，用于 match 全文检索，不能直接排序/聚合；keyword 整值不分词，用于 term 精确匹配、聚合、排序；两者兼顾用 multi-fields（text + keyword 子字段，ignore_above 256）
- **写入流程** — 协调节点路由 `shard = hash(routing) % number_of_primary_shards` → 主分片写内存 buffer + translog → 并行转发 replica，全部成功才返回 → refresh 默认 1s 生成新 segment 才可搜 → flush 落盘 fsync
- **读取流程** — Query phase：协调节点广播各相关分片，每分片返回本地 top `from+size` 的 (docId, score, sort value)，协调节点全局归并；Fetch phase：只对最终 Top N 按其所属分片取完整文档，拼装返回
- **refresh/flush** — refresh：buffer 生成 segment 进 OS cache 可搜索，轻量，默认 1s；flush：写 Lucene commit + fsync 并截断 translog，默认 translog 超过 flush_threshold_size（512MB）触发（5.x 之前另有 30 分钟周期触发）；"可搜索"取决于 refresh，"不丢数据"取决于 translog
- **translog** — ES 层的 WAL：每次写 append，默认 `index` 同步持久化（等 fsync）；节点崩溃后按 translog 重放恢复未 flush 的写；批量导入可临时 async + 加大阈值换吞吐
- **NRT** — 近实时：写成功 ≠ 立刻可搜，延迟 = refresh_interval（默认 1s）；`GET /_doc` 走 realtime 直查不受影响；读写一致性用 `refresh=wait_for` 保证
- **doc_values** — 列式正排结构，写入时构建，排序/聚合/script 用它，靠 OS cache 而非堆；text 默认没有 doc_values，对 text 排序聚合会触发 fielddata 进堆（危险）；`"doc_values": false` 可对不聚合字段省磁盘
- **fielddata** — text 字段排序/聚合时把词典加载进堆的产物，默认禁用；`fielddata > 0` 即报警信号，改 keyword 子字段是根治办法
- **深分页** — from+size 每分片取 from+size 条到协调节点合并，1 万分片就是 100 亿×分片数；max_result_window 默认 10000；解法：search_after + PIT（实时翻页）、scroll（导出）、多分片并行滚动
- **terms 聚合误差** — 各分片独立算 top-N 再合并，文档分布不均时全局词条可能落榜；`error_rate`（默认 1.5%）为期望误差上限，增大 size/shard_size 换准；精确需求用 composite 聚合翻页
- **分片规划** — 单分片搜索型 30-50GB、日志型 50-100GB；每 GB 堆 ≤ 20 分片，单节点 ≤ 600；主分数不可改，改只能 reindex/shrink/split；分片过多 → master 压力 + 堆元数据开销
- **脑裂** — 7.7 前 Zen1 靠 `minimum_master_nodes = N/2+1` 防；8.x voting configuration + 多数派选举，机制上杜绝双主，少数侧自动停摆；运维口诀：3 专职 master、独立故障域、下线先加 voting exclusion
- **query cache** — 缓存 filter 子句的 bitset（按段），对不变条件+低变化数据有效；match 全文查询不进；命中率看 `_stats/query_cache`
- **ILM** — hot(写+rollover) → warm(shrink+forcemerge) → cold(read_only、低配) → frozen(searchable snapshot 挂对象存储) → delete；别名收口读写，策略挂索引模板
- **routing** — `hash(routing) % 主分片数` 决定文档归属；自定义 routing 可让同租户/同用户数据进同分片加速查询，代价是热点与扩缩容复杂
- **副本** — 写都走主分片再转发副本；副本防单点不防误删（误删实时同步）；防灾靠 snapshot（段级增量、引用计数），低 RPO 跨集群靠 CCR
- **段(segment)** — Lucene 不可变小文件，删除只打标记，merge 合并小段并物理回收；搜索成本随段数上涨，warm 阶段 forcemerge 到 1 段
- **bulk 调优** — 5-15MB/批、并发随吞吐曲线找拐点；导入期 replica=0、refresh_interval=-1、translog async，导完恢复 + forcemerge
- **ES vs Solr** — 新项目选 ES：原生分布式（Solr 需 ZooKeeper）、ELK 生态、社区活跃度高一个量级；两者都是 Lucene 内核

# 2. 面试题

## 2.1 什么是 Elasticsearch？和传统关系型数据库有什么本质区别？

ES 是基于 Lucene 的分布式搜索与分析引擎，存储 JSON 文档、用 REST API 交互。和 MySQL 的本质区别在四个层面：索引结构上，ES 用倒排索引服务"按内容找文档"，B+Tree 服务"按主键/二级键找行"；扩展方式上，ES 靠分片原生水平拆分；一致性上，ES 不支持多文档事务、写入近实时，MySQL 是 ACID 强一致；查询模型上，ES 擅长全文检索、聚合分析，MySQL 擅长精确 CRUD 与关联查询。两者是互补不是替代：业务库保交易，ES 做搜索/日志/报表，通过 binlog 同步（canal）保持一致。

## 2.2 倒排索引的原理是什么？为什么它检索快，更新却慢？

倒排索引把"文档→词"翻转为"词→文档列表"：分词器把文本切成词项，词典（FST，前缀共享、常驻内存）记录词项位置，倒排表存该词项出现的 docId 序列及其词频、位置，docId 列表有序便于跳表二分与位图求交，"iPhone AND 15"就是两条 posting list 求交集。查询只与词表规模相关而与文档总量几乎无关，所以快。慢的一面来自不可变性：更新一条文档 = 旧文档段里标记删除 + 写新段，删改积累导致段膨胀、查询扫无效文档，要靠 merge 整理——这就是 Lucene"读快写慢"的根源，也是 ES 近实时、不适合当主存储的原因。

## 2.3 讲一下 ES 写入一条文档的完整流程。

1) 请求到协调节点，计算 `shard = hash(_routing) % number_of_primary_shards` 定位主分片；2) 转发到主分片节点，先写内存 buffer、同时 append 一条 translog（默认等 fsync 才算成功，保证崩溃不丢）；3) 主分片把文档并行转发给所有副本，全部 ack 后返回成功；4) 约每 1 秒 refresh：buffer 生成新 segment 进入 OS cache，此刻起才可被搜索——所以 ES 叫近实时；5) translog 超阈值触发 flush：segment fsync + Lucene commit + 截断 translog。"写入成功"由 translog 保证持久，"可搜索"由 refresh 决定，两者解耦是理解 ES 写入的关键。

> **注意**：主分片转发副本时，若个别副本节点故障，ES 不会拖住整个请求——该副本被踢出重新分配，主分片照常 ack；但若已无可用副本分配目标（如所有节点磁盘到 high watermark），写请求会等待或失败。

## 2.4 refresh、flush、translog 三者是什么关系？分别什么时候用？

| 操作 | 作用 | 默认触发 | 代价 |
|------|------|----------|------|
| refresh | buffer→segment，数据可搜索 | 1s | 轻，不 fsync |
| flush | Lucene commit + fsync + 截断 translog | translog 超 512MB | 重 |
| translog | 每次写 append，崩溃后重放找回 | 每次写同步（index 模式） | fsync 延迟 |

refresh 解决"可搜索"，translog 解决"不丢数据"，flush 解决"translog 瘦身与正式提交"，也可 `POST /_flush` 手动触发。批量导入时把 refresh_interval 设 -1、translog 改 async，吞吐大幅提升，代价是窗口内可搜索性与持久性都下降，导完必须恢复设置。加分句：refresh 是"对搜索可见"，flush 是"对重启安全"，translog 是两者之间的桥。

## 2.5 ES 为什么是近实时（NRT）？写入后立刻要能查到怎么办？

因为写先进 buffer，只有 refresh 生成 segment 后才进入搜索视野，默认 1s 一次，所以"写成功"和"搜得到"之间有最多一个 refresh 周期的窗口——这是为吞吐与段规模控制做的取舍。三种手段可消除等待：写请求带 `?refresh=wait_for`（挂起等到下次 refresh 再返回，兼顾吞吐与可见性）、`?refresh=true`（强制立即 refresh，重操作别滥用）、或 `GET /_doc/<id>` 直读（realtime get 不受 refresh 影响）。"下单后列表页必须能看到"这类体验用 wait_for，而不是把全局 refresh_interval 调成 100ms 拖垮集群。

## 2.6 text 和 keyword 有什么区别？一个字段既要全文检索又要聚合怎么办？

text 先分词再建倒排，支持 match 相关性检索，适合标题、正文、描述；keyword 整值入词典不分词，支持 term 精确匹配、排序、聚合，适合状态、枚举、ID、标签。同一字段两种用法用 multi-fields：

```json
"description": {
  "type": "text",
  "fields": { "keyword": { "type": "keyword", "ignore_above": 256 } }
}
```

检索打 `description`，聚合排序打 `description.keyword`。追问点：为什么 text 不能直接聚合——因为它在 doc_values 里不可用，排序/聚合会触发 fielddata 把整个词典装进堆，大字段直接打爆内存，所以默认禁止。

## 2.7 深分页问题为什么产生？有哪些解法？

`from + size` 要求每个分片各自取出 `from+size` 条（全局序不确定），协调节点合并后再截断——翻到第 10 万页时，100 个分片就是千万级文档在网络搬运，内存 CPU 双爆，所以默认 `max_result_window=10000` 拦住。解法分场景：① 用户翻页用 `search_after` + PIT：以上一页最后一条的排序值做游标，每分片只取 size 条，可深翻；② 全量导出用 scroll 或 async search + search_after（8.x 推荐）；③ 业务上限制页数（搜索产品通行做法）。search_after 的排序键必须加 `_shard_doc` 之类的唯一次级键保证全序，否则会漏/重。

## 2.8 terms 聚合结果准吗？为什么？

不完全准。聚合分两阶段：每个分片只统计并返回本地前 `size` 个桶（可用 `shard_size` 放大每分片收集量），协调节点再合并截断；全局高频词如果在某分片恰好排不进本地 top，就可能落榜或计数偏小。因此响应给出 `doc_count_error_upper_bound`（误差上界）和 `sum_other_doc_count`（未进桶文档计数）。误差受 `error_rate`（默认 1.5%）与 size/shard_size 影响，调大换准。要精确结果：换 composite 聚合游标翻页全量统计。

## 2.9 分片数怎么规划？分片过多或过少有什么后果？

经验值三条：单分片大小——搜索密集 30-50GB、日志密集 50-100GB；分片总量——每个 GB 堆内存不超过 20 个分片、单节点不超过 600；数量推导——按"保留期总数据量 ÷ 目标分片大小"。过多：每分片是独立 Lucene 实例，segment 元数据/global ordinals 常驻堆，查询建堆合并成本随分片数线性涨，master 状态膨胀、恢复慢——"千万条数据开 100 分片"是典型事故。过少：失去并行度，单分片迁移/重建耗时以小时计，且主分片数不可改，只能 reindex/shrink/split。日志类用"按天滚动 + 每天固定小分片数 + ILM shrink"，避免一次性赌对总量。

## 2.10 ES 怎么防止脑裂？

脑裂是网络分区下两侧各自选出 master 同时接受写入，合并时数据无声丢失。7.7 之前的 Zen1 靠人工纪律 `discovery.zen.minimum_master_nodes = N/2+1`（默认值 1，忘配即脑裂）。8.x 的 cluster formation + voting configuration 从机制上消灭了它：每轮选举带配置版本号，只有拿到**多数派投票**才能当 master——分区少数侧选不出主，最多集群停摆而不是双主写花。运维三件套仍然不能省：master-eligible 保持 3 台且独立故障域、计划下线节点先 `POST /_cluster/voting_config_exclusions`、别把数据节点设为 master-eligible。

## 2.11 什么是 doc_values？和 fielddata 什么区别？为什么 ES 能聚合却不做 join？

doc_values 是写入时同步构建的**列式正排文件**（docId→值），存磁盘靠 OS cache 读取，排序/聚合/script field 都走它，keyword、数值、date、geo 默认开启；优势是不吃堆。fielddata 则是对**没有 doc_values 的 text 字段**做排序/聚合时把词典加载进 JVM 堆的内存结构，默认禁止且一开就危险。"不能 join"是设计取舍：doc_values/倒排都是分片内自治结构，跨分片关联没有分布式 join 的执行器——替代方案是写入时反范式化（nested/parent-child 有限支持）。

## 2.12 query cache 缓存什么？为什么我的 filter 没命中缓存？

query cache 缓存的是**filter 子句**（不打分条件）在每个段上的匹配结果 bitset，key 是 query + 段名。命中条件苛刻：段要不变或变化小、条件要重复出现。它不缓存全文打分查询；对高基数字段（用户 ID、每分钟都变的时间 range）缓存反而浪费内存。判断看 `_stats/query_cache` 的 hit_count/evictions。加分句：filter 走 bitset 缓存路径且不算分（比 must 快），所以"能 filter 就不要 must"。

## 2.13 说下 ILM 的设计：hot-warm-cold-frozen 各阶段做什么？

hot 承载写入与最新高频查询：SSD、rollover 按 `max_primary_shard_size=50GB/max_age` 滚动新索引；warm 停止写入做降本：shrink 合并分片、forcemerge 单段、降副本、迁 HDD；cold 极低频只读、更弱硬件；frozen 用 searchable snapshot 把段留在对象存储，本地只缓存访问到的块；delete 到期删除。策略通过 `_ilm/policy` 定义、索引模板挂载，写入端只认别名，整个生命周期靠 rollover 自动翻页。追问点：为什么 forcemerge 放 warm 不放 hot——对写入中的段做 merge 是重 IO 灾难；为什么 merge 后再快照——段不再变化，后续快照只增量上传新段。

## 2.14 ES 如何保证高可用？副本、快照、CCR 各自解决什么？

三层冗余三种目标：副本解决**单机故障**——节点挂了副本秒级提升主分片，服务不停，但它实时同步误删，防不了"数据被错误删除/损坏"；快照解决**灾备与回滚**——段级增量复制到对象存储，SLM 每日自动，能回到任意快照时间点；CCR 解决**低 RPO 异地热备**——操作日志级秒延迟复制，但它不是备份（误删照样同步）。再加选主多数派（防脑裂）、熔断限流（防雪崩）、协调节点隔故障域，构成完整高可用体系。答题收口：副本防"点"、快照防"错"、CCR 防"远"。

## 2.15 线上集群突然变 Red，你的排查步骤？

1) `GET _cluster/health?level=indices` 确定哪些索引 red，先看影响面；2) `GET _cat/shards?v&h=index,shard,state,unassigned.reason` 找未分配的主分片与原因；3) `GET _cluster/allocation/explain` 看分配器逐节点否决理由——最常见三类：磁盘到 high/flood watermark（清盘/扩容，并确认 `read_only_allow_delete` block 解除）、节点离线（恢复节点，等自动重新安置）、`ALLOCATION_FAILED`（失败重试达 `cluster.routing.allocation.max_retries`，默认 5 次；修掉根因如 mapping 冲突后 `POST _cluster/reroute?retry_failed=true`）；4) 若主分片数据确认已丢（磁盘物理损毁），从快照 restore 指定索引；5) 处置期间临时止血可把该索引副本置 0 或关闭非核心 red 索引让集群先 green。全程先读 explain 再动手，`allocate_empty_primary` 这类强制命令只有确认接受数据丢失才允许使用。

> **设计考量**：追问"为什么 disk watermark 会导致 red"时，要能讲出 85% 停止新分片分配、90% 迁出分片、95% 索引锁只读这条水位线阶梯，以及"日志类集群 red 多半是水位线没配告警、靠 flood stage 兜底"的运维教训——这比背命令更能体现生产经验。
