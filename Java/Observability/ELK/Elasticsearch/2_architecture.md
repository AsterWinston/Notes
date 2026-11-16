# 1. 概念层级

理解 ES 集群要从外向内建立四层心智模型：**Cluster 由 Node 组成，Node 承载 Shard，Shard 就是一个 Lucene 索引**。一切分布式行为（扩容、容错、路由）都发生在 Shard 这一层。

```text
Cluster (集群)
  └── Node (节点, 1-N)
        └── Index (索引, 1-N)
              ├── Settings (配置)
              ├── Mapping (字段定义)
              └── Shard (分片)
                    ├── Primary Shard (主分片, 1-N, 写入时定)
                    └── Replica Shard (副本分片, 0-N, 可动态调整)
```

这里最容易混淆的是"索引"的两义性：对使用者来说 Index 是一堆文档的逻辑集合（类似 MySQL 的表），但对内核来说**一个 Index 被物理拆成 N 个 Primary Shard，分散在不同节点上，每个 Shard 是一个完整独立的 Lucene 实例**。所以"查询一个索引"实际是"并发查询它的全部分片再归并"。第 4 篇的分片规划、第 12 篇的查询性能，全部建立在这一认识之上。

# 2. 节点角色

一个 ES 进程默认同时扮演多种角色，但生产集群要**按角色拆分机器**：

| 角色 | 配置 | 职责 |
|------|------|------|
| **Master-eligible** | `node.roles: [master]` | 集群管理（索引创建/删除、分片分配、发布集群状态） |
| **Data** | `node.roles: [data]` | 数据存储、CRUD、搜索、聚合（CPU/内存/磁盘重负载） |
| **Data hot/warm/frozen** | `node.roles: [data_hot, data_warm, data_frozen]` | 配合 ILM 分层存储 |
| **Ingest** | `node.roles: [ingest]` | 预处理管道（Pipeline），文档写入前执行脚本/解析 |
| **ML** | `node.roles: [ml]` | 机器学习任务 |
| **Coordinating** | `node.roles: []` | 纯协调：请求路由、结果归并（无状态，可水平加） |
| **Remote Cluster Client** | `node.roles: [remote_cluster_client]` | 跨集群搜索（CCS）/跨集群复制（CCR） |

```yaml
# 生产推荐节点分工（8.x 用 node.roles 取代 7.x 前的 node.master/node.data 开关）
# Master 节点（3 台，小巧：4C8G 足够，重点是稳定而非性能）
node.roles: [master]

# Data 节点（不参选 master，避免"既做管理又干活"导致 GC 停顿引发脑裂）
node.roles: [data, ingest]

# Coordinating-only 节点（所有角色关闭，专职扛客户端流量）
node.roles: []
```

**为什么 master 与 data 必须分开**：Master 维护集群状态（Cluster State）是一个**单线程串行的元数据事务**。若 master 同时承载大量数据分片，一次 Full GC 停顿几十秒就会被判定失联触发重选——每次选举冻结元数据变更，进而引发分片重分配风暴。小规格专用 master（3 台）+ 大规格 data 节点是标准答案。

**协调节点为什么值得独立**：搜索请求的 Query Phase 要向所有相关分片扇出、在内存维护 `分片数 × from+size` 的归并堆，聚合还要接收各分片中间结果做 reduce——这部分压力落在**接收请求的节点**上。客户端直连 data 节点时，协调开销会挤占本应用给 Lucene 的堆内存；独立 coordinating 层（无状态、加机器即扩容）是大集群标配。

> **注意**：8.x 里 `node.master: true` / `node.data: true` 这类旧开关已不推荐，统一使用 `node.roles` 列表；且**任何节点都可能成为协调节点**——客户端把请求打到哪，哪台就承担归并工作，"随机直连 data 节点"在大集群是要避免的反模式。

# 3. 分片模型

```text
Index: orders (3 主分片, 1 副本)
┌──────────────────────────────────────────────────────┐
│  Node A: Primary 0        Node B: Replica 0           │
│  Node B: Primary 1        Node C: Replica 1           │
│  Node C: Primary 2        Node A: Replica 2           │
└──────────────────────────────────────────────────────┘

写入路由公式: shard = hash(_routing) % num_primary_shards
（_routing 默认取文档 _id，可用 ?routing=xxx 覆盖）
```

**关键规则**：

- 主分片数索引创建时确定，不可修改——因为路由公式对 `num_primary_shards` 取模，改了模数同一文档就会算到另一个分片上，历史数据全部"查无此人"。想改只能 shrink/split/reindex（见第 4 篇）。
- 副本分片数可动态调整，`number_of_replicas` 随时 PUT `_settings` 即可。
- 主分片和它的副本**绝不在同一节点**——同节点存放只是浪费磁盘，节点宕机时一起丢，起不到容错作用。
- 每个分片是一个完整的 Lucene 实例，副本与主分片内容一致，但**各自独立执行读写**。

分片数量设计的取舍在第 4 篇详述，这里先记住两条铁律：**副本不是备份**（一次误删索引 API 会连副本一起删，备份靠 snapshot，见第 15 篇）；**分片不是越多越好**——每个分片有自己的 segment、堆内数据结构和集群状态元数据，几万个小分片让 master 发布状态都困难（small shards problem）。单分片建议控制在 10–50GB。

> **行业实践**：搜索请求默认主副分片都参与（轮询），写入只进主分片再复制副本；但"写完立刻要读到"的业务读写，不要指望副本解决一致性——副本复制是异步的，延迟毫秒级但非零。

# 4. 写入流程

一条 `PUT /index/_doc/1` 在集群内的完整旅程：

```text
Client → Node 1 (Coordinating)
  │
  │ 1. 解析请求，读取索引元数据，计算路由:
  │    shard = hash(_routing) % num_primary_shards
  │
  ├──→ Node 2 (Primary Shard)
  │      │
  │      │ 2. 版本检查(if_seq_no) → 写入 Lucene 内存 buffer + translog(fsync)
  │      │
  │      │ 3. 并行转发给所有副本(携带同一操作序列)
  │      │
  │      ├──→ Node 3 (Replica Shard)
  │      │     └── 4. 写 buffer + translog，完成后 ACK 主分片
  │      │
  │      └── 5. 收到全部副本 ACK 后回复协调节点
  │
  └── 6. 回复 Client
```

三个机制细节决定了 ES 的可靠性语义（其中协调节点缓存路由表，无需 master 参与即可投递请求，这也是 GET by ID 实时可读、search 近实时的原因）：

- **translog（事务日志）是写入持久性的锚点**。文档先进内存 buffer，同一操作立刻顺序追加写 translog；`index.translog.durability` 默认 `request`，即**每次请求都 fsync translog 后才返回**——"返回成功的文档断电不丢"，重启时靠 translog 重放恢复尚未 refresh 的 buffer。高吞吐日志场景可改为 `async`（约每 5s fsync 一次），换性能、赌最坏 5 秒数据。
- **写入默认等全部副本 ACK**，任何副本失败则整个请求失败；但主分片不会回滚——ES 用**分片级隔离**处理：出问题的副本被标记失败并重分配，一致性靠"每个分片各自重放 translog"保证。

## 4.1 Refresh — 数据可搜索

```text
Refresh: 内存 Buffer → 打开(Lucene Writer) → 生成新 Segment → 可被搜索

间隔: index.refresh_interval (默认 1s)
代价: 频繁 refresh → 小 segment 爆炸 → merge 压力 + 搜索要打开更多段
优化: 批量导入时设置 refresh_interval=-1（导入完成后恢复并手动 _refresh）
```

refresh 这个名字极易误导：它**不写磁盘、不刷数据**，只是把内存 buffer 里的文档交给 `IndexWriter` 生成新的（其实还在 OS page cache 里的）segment 并打开供搜索。所以 refresh 越频繁，小段越多，搜索时每个分片要读取的段文件越多，反而更慢。日志写入集群普遍把 refresh_interval 调到 30s–60s，用新鲜度换吞吐。

## 4.2 Flush — 持久化到磁盘

```text
Flush: 内存 Segment → OS 文件缓存 → fsync 磁盘, 然后 truncate translog

触发条件:
  - translog 大小达到 index.translog.flush_threshold_size (默认 512MB)
  - 达到 flush_threshold_period (默认 30 分钟)
  - 手动 POST /_flush
```

flush 才对应直觉里的"落盘"：落的是 **segment 文件**，落盘后 ES 截断（trim）已持久化对应的 translog 记录，防止其无限膨胀。**refresh 管可搜索性，flush 管崩溃恢复窗口，merge 管段的数量与大小**——这三个后台动作互相独立，是理解第 13 篇索引性能调优的前提。

# 5. 读取流程

```text
Client → Node 1 (Coordinating)
  │
  │ 1. Query Phase: 向所有相关分片并发发送查询
  │    每个分片返回: (docId, score, sort values) 本地 Top-N
  │
  │ 2. 协调节点对 分片数×N 条结果做全局归并排序，定出全局 Top-N 的 docId
  │
  │ 3. Fetch Phase: 只对"要返回的那 N 条"按分片分组，取完整文档(_source)
  │
  └── 4. 组装响应返回 Client
```

两阶段设计的原因是：**排序信息便宜，文档内容昂贵**。Query Phase 每个分片只回传几十字节的 docId+score；只有最终胜出的 N 条才进入 Fetch Phase 取 _source。由此可推出深分页的代价：`from=10000&size=10` 时每个分片必须先产生 10010 条本地结果，协调节点归并后丢弃 10000 条——默认 `max_result_window=10000` 就是为了拦住这种请求（翻深页用 search_after，滚动导出用 PIT + search_after，scroll 已不推荐）。

聚合的读取路径与 top-N 不同：各分片本地算出部分聚合结果，协调节点再 reduce 合并。因此**聚合天然近似**——terms 聚合每个分片先取自己的 size 个桶再全局排序，真实全局第一的桶若在某分片里排名靠后会被漏掉（`doc_count_error_upper_bound` 就是这个误差上界）。

# 6. 段合并（Segment Merge）

Lucene 每 refresh 生成一个 Segment，多个 Segment 需要合并：

```text
Segments: [S1] [S2] [S3] [S4] ...
               ↓ merge
          [S12]  [S3]  [S4] ...
               ↓ merge
          [S1234]  ← 一个更大的 Segment

合并策略: tiered (默认), 根据段大小分层合并
合并开销: CPU + IO；被标删的文档只在 merge 时真正移除
```

merge 同时干两件事：**减少段数量**（每段对应一组文件句柄和堆内开销，搜索要逐段执行再归并）与**物理清除删除标记**（update = delete + index，老文档留在原地直到 merge）。代价是重写大量数据，是 ES 最重的后台 IO 来源。

**设计考量**：

- **默认 tiered 策略**：把大小同数量级的段合并成更大段，控制总段数在 log 级别。可通过 `index.merge.policy.*` 调节（segments_per_tier、max_merged_segment 等），但绝大多数场景不值得动。
- **只对不再写入的索引 forcemerge**：日志按天滚动，昨天的索引已经只读，用 `POST /index/_forcemerge?max_num_segments=1` 把它压成一段，搜索与磁盘占用都显著受益；**正在写入的索引绝对不要 forcemerge**，会把热段冻成超大段，后续每次删除/更新都要重写它。
- **merge 限速**：`index.merge.scheduler.max_thread_count` 和 throttle 设置平衡"merge 完成快"与"别把磁盘打满"；HDD 上单线程，SSD 可以放心并发。

# 7. Master 选举

```text
# 选举流程（8.x 基于 Zen 2 / 内置协调算法，不再依赖外部组件）
1. Master-eligible 节点定期互相 ping，维护存活视图
2. 达到初始选举资格(initial election quorum)的节点开始拉票
3. 候选者向 compatible 节点收集投票，需获得 (eligible/2)+1 票
4. 胜出者成为 Master，发布 Cluster State 版本号，其余节点接受并应用

# 集群组建引导（首次启动必需）
discovery.seed_hosts: ["node1:9300", "node2:9300", "node3:9300"]
cluster.initial_master_nodes: ["node1", "node2", "node3"]
```

选举设计上的关键变化：7.x 之前用 `discovery.zen.minimum_master_nodes = (master_eligible_nodes / 2) + 1` 手工配置防脑裂参数，**配错就双主**——历史上大量 ES 数据丢失事故的根源。Zen 2 把它改为**算法内置**：集群自动跟踪 master-eligible 节点数并维持 quorum（多数派），运维只需保证 **master 节点数配成奇数（3/5）**。

> **注意**：`cluster.initial_master_nodes` 是 bootstrap 一次性引导配置，节点全部起来后使命即结束；加新节点时把它写进去会导致新节点尝试组建"新集群"而不加入老集群，加机器的正确姿势是只配 `discovery.seed_hosts`。

Master 故障的表现分两种：短暂停顿触发重新选举，集群只是暂停元数据变更（读写不受影响）；**状态发布慢**则是大集群的慢性病——集群状态可达几十 MB，每次变更全节点广播会放大 master 单线程瓶颈，对策是控制索引与分片总量。

# 8. 倒排索引结构

最后把镜头拉到最小的存储单元：一个 Shard 内部，文档如何变成可搜索的结构。字段的处理方式由 mapping 决定（第 6 篇），核心是两类互补结构：

```text
原文档: { "title": "iPhone 15 Pro 手机", "price": 8999 }

title 经 analyzer 分词: [iphone, 15, pro, 手, 机]

倒排索引(用于搜索"谁包含这个词"):
┌──────────┬────────────────────────────┐
│ Term     │ Posting List (docId, tf…)  │
├──────────┼────────────────────────────┤
│ iphone   │ → [1]                       │
│ 15       │ → [1]                       │
│ pro      │ → [1]                       │
│ 手       │ → [1]                       │
└──────────┴────────────────────────────┘

doc_values(列式正排, 用于排序/聚合"这个文档的这个值是多少"):
┌──────────┬──────────┐
│ Doc      │ price    │
├──────────┼──────────┤
│ 1        │ 8999     │
└──────────┴──────────┘
```

- **倒排索引（inverted index）**回答"包含 term X 的文档有哪些"，是搜索的引擎：term 有序存放（FST 词典），posting list 压缩存储，多条件求交并极快。
- **doc_values** 是写入时额外构建的**按列正排存储**，回答"文档 i 的字段值是多少"，排序、聚合、脚本都靠它；这也是 text 不能排序（分词后值已碎）、keyword 可以的原因。
- **_source** 保存原始 JSON，供 fetch 返回、partial update 和高亮重算；它不用于检索。

三者对存储成本各出一份力，所以第 6 篇里"用不到的结构就关掉"（`index: false`、`doc_values: false`、`norms: false`）是 mapping 设计的第一性原则。

# 9. 小结

| 主题 | 核心结论 |
|------|---------|
| 概念层级 | Index 被物理拆成 Shard，每个 Shard 是独立 Lucene 实例，分布式行为全在 Shard 层 |
| 节点角色 | master/data 分离防 GC 引发脑裂；大集群独立 coordinating 层扛归并压力 |
| 分片规则 | 主分片数不可改（路由取模决定）；副本≠备份；分片大小 10–50GB，宁少勿多 |
| 写入链路 | 主分片 buffer+translog → 并行复制副本 → 全部 ACK 才返回；fsync 在 request 粒度 |
| 三后台动作 | refresh 管可搜索性（默认 1s，可调大）、flush 管落盘与 translog 截断、merge 管段数与清删 |
| 读取链路 | Query Phase 扇出取 docId+score，协调节点全局归并，Fetch Phase 只取 Top-N；深分页由此昂贵 |
| 选举 | Zen 2 多数派内置防脑裂，master 节点取奇数；initial_master_nodes 仅首启用 |
| 倒排结构 | 倒排索引负责搜、doc_values 负责排/聚、_source 负责还原文档 |

第 2 篇走完了数据在集群里的完整一生。第 3 篇回到工程起点：如何把这套集群装起来——zip/tar 与 Docker 部署、jvm.options 堆配置、elasticsearch.yml 关键参数，以及 8.x 默认安全体系的启用。
