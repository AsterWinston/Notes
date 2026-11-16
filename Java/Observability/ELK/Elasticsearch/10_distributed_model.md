# 1. 写入流程详解

Elasticsearch 的分布式能力建立在两个基本动作上：**把数据切开（分片）和把数据复制（副本）**。理解集群模型，最好的切入点是跟着一条文档走完一次写入——它会依次经过协调节点的路由计算、主分片的写日志与内存缓冲、副本的并行复制，最后才返回 ACK。这条链路上的每一步都对应着一个可观测、可调优、也会出故障的环节。

```text
Client → Node 1 (Coordinating)
  │
  │ Step 1: 路由计算
  │   shard = hash(_routing) % num_primary_shards
  │   shard = hash("1") % 3 = 1 → Primary 1 on Node 2
  │
  ├── Step 2: 转发到 Primary Shard (Node 2)
  │     │
  │     │ a. 写入 Translog (同步磁盘)
  │     │ b. 写入 Memory Buffer
  │     │ c. 转发到 Replica (并行)
  │     │
  │     ├──→ Replica on Node 3
  │     │     └── a. 写入 Translog, b. 写入 Memory Buffer
  │     │
  │     └── 等待副本确认 (wait_for_active_shards)
  │
  └── Step 3: 返回 Client (ACK)
```

**路由公式 `shard = hash(_routing) % number_of_primary_shards` 是整个模型的地基**。默认 `_routing` 取文档 `_id`，所以同一个 ID 永远落到同一个主分片，这保证了"改同一条数据总是打到同一份"。**公式里参与取模的是主分片数而不是总分片数**——这条规则决定了主分片数一旦确定就不可更改：改了模数，所有历史文档的路由结果全部错位，存量数据再也查不到。想按业务维度控制分布（比如同一租户的数据放同一分片以享受本地查询），写入时显式传 `routing` 参数即可，代价是跨租户查询要扫全部分片。

副本分片永远不会被协调节点直接写。客户端只和主分片发生写入关系，副本是主分片的"被动镜像"，这样避免了多写者并发带来的版本冲突与去重成本。也正因如此，主分片迁移期间该分片不可写，而副本迁移不影响写入。

## 1.1 wait_for_active_shards

```json
PUT /orders/_doc/1?wait_for_active_shards=all
{ "amount": 100.0 }

# all: 等待所有副本确认(最安全,最慢)
# 1: 仅主分片(默认,最快,最不安全)
# 2: 主 + 1 个副本确认
```

**默认值是 1，即只等主分片写完就返回，副本复制是异步的**——这一点常被误解。副本没追上时集群会短暂"分片之间不一致"，依赖副本读到的旧数据称为 stale read；如果主分片在副本落地前挂掉，未复制的那部分写入会随主分片一起丢失（只能从 translog 恢复，前提是新主能拿到那份 translog）。把 `wait_for_active_shards` 调到 `all` 或具体数字，等价于用延迟换持久性：副本所在节点故障时，写入会直接超时报错而不是静默降级。

> **注意**：ES 没有提供跨文档事务与"quorum 读写"语义，`wait_for_active_shards` 只影响单次请求的确认时机，不要把它当成 CAP 里的强一致开关。真正不能丢的数据（订单、账务）应该有外部可靠来源（Kafka/数据库）+ 可重放的 `_id` 幂等写入，靠"可重试"而不是靠"等多久"来保证不丢。

## 1.2 近实时（NRT）的含义

写入 ACK 返回时，文档**已经持久化但不一定可被搜索**。ES 官方把这种特性称为 near real time，默认延迟约 1 秒（refresh 周期）。原因是倒排索引不可原地修改，新文档要等到 refresh 生成新 segment 后才进入搜索视野。这个设计取舍是明确的：**换到的是极高的写入吞吐，付出的是"写完立刻查会查不到"**。

业务代码里最常见的踩坑就是"写完立刻读"：创建订单后马上跳转列表页，列表查不到刚建的单。解决方案按代价排序：① 前端用本地状态直接展示（推荐）；② 写入时 `?refresh=true`（单条可以，批量是灾难）；③ 用 `_mget`/`GET _doc` 按 ID 直接读——**按 ID 读取是实时的，不受 refresh 影响**，因为它走 translog + 内存 buffer 的实时查找路径。

# 2. Refresh — 内存到可搜索

refresh 负责把"已写入但不可搜索"变成"可搜索"，它的成本模型决定了默认 1 秒这个值。

```text
Memory Buffer (不可搜索)
  │
  │ refresh (默认 1s)
  ▼
Segment (文件系统缓存, 可搜索)
  │
  │ flush (translog fsync + segment fsync)
  ▼
Segment (磁盘)

关键: refresh 只是 fsync 到文件系统缓存, 并不落盘
```

每次 refresh 会创建一个轻量级的新 segment，数据先驻留在**文件系统缓存（OS page cache）**里，靠操作系统在后台异步刷盘。所以 refresh 的开销是"构建一个小倒排段 + 打开 reader"，不是磁盘 IO——这就是 1 秒间隔能接受的原因，也是"关掉 refresh 能提速"的原理：省掉的是频繁小段构建与段合并压力，而不是 fsync。

```json
# 手动刷新
POST /orders/_refresh

# 批量导入时关闭
PUT /orders/_settings
{ "refresh_interval": "-1" }
```

> **行业实践**：日志类写入普遍把 `refresh_interval` 设成 `30s` 甚至 `-1`（导入期），搜索型业务保留 1s。判据是"用户能否容忍秒级延迟"与"每秒产生多少小 segment"——高频 refresh + 高基数 keyword 字段组合会让聚合变慢（global ordinals 反复重建，见第 8 篇）。

`searchidle` 是常被忽略的自动优化：索引连续 30 秒（`index.search.idle.after` 默认值）没有搜索请求，ES 会自动停掉它的 refresh，下一次搜索到来再恢复。所以"导入期关了 refresh 吞吐也没涨多少"是正常现象——空闲索引本来就不怎么 refresh。

# 3. Translog — 持久化保障

```text
Translog = Write-Ahead Log(与 MySQL Redo Log 同族)

作用: 防止数据丢失
  - 写入 Memory Buffer 前先写 Translog
  - 宕机后从 Translog 重放恢复

Flush 触发条件:
  - Translog 达到 index.translog.flush_threshold_size (默认 512MB)
  - 手动 POST /orders/_flush
  - 索引被 close 时自动 flush

Flush 过程:
  1. 先做一次 refresh, Memory Buffer → 新 Segment
  2. Segment fsync 到磁盘, commit point 落盘
  3. 截断(rotate) Translog
```

**Translog 是"倒排索引无法增量写"这一限制的补丁**。Lucene 的 segment 一旦生成就不可修改，引入 translog 后，写入只需顺序追加一条日志（廉价 IO），倒排段的构建推迟到 refresh/flush 时批量完成。恢复时的逻辑对称：重放最后一个 commit point 之后的 translog 记录。

```json
# 调整 Translog
PUT /orders/_settings
{
  "index.translog.durability": "async",   // 异步刷盘(性能高, 风险大)
  "index.translog.sync_interval": "5s",
  "index.translog.flush_threshold_size": "1gb"
}
```

`durability` 两个取值定义得很清楚：`request` 是每次写请求都 fsync translog（默认，最安全）；`async` 是按 `sync_interval` 周期 fsync，最坏丢失一个周期的数据。注意 **async 只是让 fsync 变慢，写入仍然是先落 translog 再返回**——它能扛进程崩溃和宕机（日志已在 OS 缓存），但扛不住整机断电/内核 panic（缓存未落盘）。

## 3.1 三者关系图

| 机制 | 触发 | 产物 | 影响可见性 | 影响持久性 |
|------|------|------|-----------|-----------|
| refresh | 周期（默认 1s）/手动 | 新 segment（page cache） | ✓ 变为可搜索 | ✗ 不 fsync |
| flush | translog 512MB/手动 | commit point + segment fsync | 顺带 refresh | ✓ 并可截断 translog |
| translog fsync | 每次请求（request）/周期（async） | 追加日志落盘 | ✗ | ✓ 崩溃恢复依据 |

```text
  写入 ──▶ translog(append) ──▶ in-flight segment(内存)
                │                        │
                │                  refresh(1s)
                │                        ▼
                │                 searcher 可见(可搜索)
           fsync(request)               │
                ▼                  flush(512MB)
            断电安全 ──────────▶ 段 fsync + commit point + 截断 translog
```

> **注意**：很多资料把 flush 描述成"内存满才触发"，这是过时说法。8.x 中 flush 主要由 translog 大小驱动，与 index buffer 下推成 segment 不是同一件事。

# 4. 读取流程再深入

```text
Client → Node 1 (Coordinating)
  │
  │ Step 1: Query Phase
  │   向所有相关分片发送查询
  │   各分片返回: (docId, _score, sort values) — 轻量
  │
  │ Step 2: Coordinate & Merge
  │   协调节点归并排序, 取全局 Top N 个 docId
  │
  │ Step 3: Fetch Phase
  │   向对应分片批量获取完整文档
  │   返回 Client
```

这就是经典的 **query-then-fetch 两阶段**。分成两阶段的原因是评分与排序必须全局决策：每个分片只知道自己数据的 top `from+size` 个候选（轻量元组），真正的 top N 只能在协调节点归并后确定，此时才值得去取完整文档。**由此推出一条实用结论：每个分片都要维护一个大小为 `from+size` 的优先队列，这就是深分页代价随偏移线性增长的来源**（详见 12_search_performance）。

Fetch 阶段默认按分片分组批量取回，查询与取回针对同一份 searcher 视图，不会出现"查到一半数据变了"的错乱。注意区分两条读路径：**搜索走 searcher，只看得见已 refresh 的段；按 ID 的 GET 是 realtime get，会直接查 translog，所以刚写入未可见的文档也能取到**——这就是"搜索查不到但 GET 查得到"的机制解释。

## 4.1 preference — 路由偏好

```json
GET /orders/_search?preference=_local
{ }

# _local: 优先本地分片(协调节点上有的分片)
# _only_local: 仅本地分片
# _preference: 用户自定义字符串, 相同字符串稳定路由到同一分片
# _shards:2,3: 指定分片号
# 也可以传 _primary / 具体节点名
```

preference 有两个正交用途：其一是**性能**——同一用户反复翻页时固定到同一分片，能持续命中该分片的 request cache/filter cache（代价是分片热点）；其二是**一致性缓解**——固定到刚写入的分片有时能让文档"看起来"已可见，但不是可靠保证，不要用。

> **注意**：preference 不改变结果正确性，只改变"谁来服务这次请求"。生产上更常见的是传**用户 ID / session ID 作为自定义 preference 字符串**，让同一用户稳定落到同一组分片；不要硬编码 `_shards:0,1`——分片拓扑一旦变化（迁址、reindex 后换索引），硬编码会让查询静默打错分片。

# 5. 分片分配策略

分配（Allocation）是主节点的工作：决定每个分片副本放在哪个节点上。三类指令的语义差别必须记清：

```json
PUT /orders/_settings
{
  "index.routing.allocation.include._tier": "hot",
  "index.routing.allocation.require.box_type": "ssd",
  "index.routing.allocation.exclude._name": "node-old",
  "index.routing.allocation.total_shards_per_node": 2
}
```

| 指令 | 语义 | 失败后果 | 典型用途 |
|------|------|---------|---------|
| `include` | 只能放到属性匹配 include 列表的节点（多值为 OR） | 无匹配 → 分片未分配 | 冷热分层 |
| `exclude` | 禁止放到匹配节点 | — | 节点下线（`exclude` 该节点名） |
| `require` | **必须**满足全部条件（多值为 AND） | 不满足 → 分片卡在 unassigned | 强制 SSD/机架 |
| `total_shards_per_node` | 该索引在此节点最多放几个分片 | 超上限不再分配 | 防止单节点分片爆炸 |

**优雅下线的标准做法是 exclude 而不是直接 stop**：`cluster.routing.allocation.exclude._name: node3` 让主节点主动把分片搬走，搬完再停进程，期间零红分片。`enable: primaries/replicas/all/new_primaries` 可精细控制分配开关，大版本升级前常设 `new_primaries` 阻止搬迁风暴。

## 5.1 分片分配感知

```yaml
# elasticsearch.yml
cluster.routing.allocation.awareness.attributes: rack_id
node.attr.rack_id: rack1          # 每个节点声明自己的属性
cluster.routing.allocation.awareness.force.rack_id.values: rack1,rack2
```

**效果**：主分片和副本不会分配到同一个 rack，提高可用性。awareness 的机制是"按属性值分组，同一分片集的副本尽量分散到不同组"，它只在节点数与分组数允许时生效。`force.values` 是关键区别项：不配 force 时，如果当前只有 rack1 的节点在线，主分片照样能分配到 rack1（集群可用但脆弱）；配了 force 后 rack2 必须有节点，否则分片保持未分配——**用可用性换"绝不妥协的容灾布局"**，机房容灾演练要求严格时选 force。

## 5.2 主分片数变更：reindex + 别名

因为路由取模的分母是主分片数，ES 不提供在线修改主分片数的能力，替代方案是"**新建索引 + reindex + 原子切别名**"：

```json
# 1. 新建更多主分片的索引
PUT /orders-v2
{ "settings": { "number_of_shards": 6, "number_of_replicas": 1 } }

# 2. reindex 迁移(slices=auto 由 ES 按分片并行切片)
POST /_reindex?slices=auto
{
  "source": { "index": "orders-v1", "size": 5000 },
  "dest":   { "index": "orders-v2" }
}

# 3. 原子切换别名
POST /_aliases
{
  "actions": [
    { "remove": { "index": "orders-v1", "alias": "orders" } },
    { "add":    { "index": "orders-v2", "alias": "orders" } }
  ]
}
```

# 6. 段合并策略

Lucene 段的不可变性意味着"删除只是标记、更新等于删除加新增"，所以段里会积累大量 tombstone 与冗余。**段合并（merge）就是把若干小段合成一个大段，顺带真正丢弃被删文档**，它是读性能的幕后功臣：段少了，一次查询要打开的倒排结构和 doc_values 文件就少了，过滤缓存与 global ordinals 也更稳定。

```json
# 强制合并到 1 个 segment(只读索引的读优化)
POST /orders/_forcemerge?max_num_segments=1

# 查看段信息
GET /_cat/segments/orders?v
```

默认合并策略是 `tiered`：**按段大小分层，优先合并大小相近的段**，用 `segments_per_tier`（默认 10）、`max_merged_segment`（默认 5GB）约束层内段数与合并产物大小。合并是 IO 与 CPU 密集操作，`index.merge.scheduler.max_thread_count` 按盘型设置（HDD 设 1,SSD 可放开）。

> **注意**：`_forcemerge` 只能用在**不再写入**的索引上。对活跃索引合并到 1 段，下一次写入又会立刻产生新段，同时合并期间会产生与索引等大的临时磁盘占用，还可能触发磁盘水位线导致集群只读。正确时机是 ILM 的 warm 阶段（见 14 篇）。

# 7. 小结

| 主题 | 核心结论 |
|------|---------|
| 写入路由 | `hash(_routing) % 主分片数`，分母不可变 → 主分片数不能改，只能 reindex + 别名 |
| 副本写入 | 只写主分片，副本异步镜像；`wait_for_active_shards` 默认 1，不是 quorum |
| 近实时 | 1s refresh 决定可见延迟；按 `_id` 读是实时的；写完立刻读用前端状态或 mget 兜 |
| refresh vs flush | refresh 造段进 page cache 只为可见；flush fsync 段 + commit point 并截断 translog 才管持久 |
| translog | WAL;request 每次 fsync,async 按 sync_interval，断电最多丢一个周期 |
| 读流程 | query-then-fetch 两阶段，分片各维护 from+size 堆 → 深分页线性变贵 |
| allocation | include=OR / require=AND / exclude 下线；awareness 分散机架，force 才绝不妥协 |
| 段合并 | tiered 按大小分层；删除靠合并回收；forcemerge 仅用于只读索引，注意临时磁盘 |

这一篇讲的是"数据在集群里怎么走"；下一篇（11_ingest_pipeline）把焦点放到写入链路的入口——文档在进入索引之前，如何在 ingest 层被解析与丰富。
