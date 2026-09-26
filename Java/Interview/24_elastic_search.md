# 1. 知识点
- **Cluster（集群）**：由一个或多个 Elasticsearch 节点组成，通过集群名称 `cluster.name` 唯一标识。
- **Index（索引）**：具有相似特征的文档集合（类比关系型数据库的表），在 ES 7.0+ 中一个索引只能有一个 `_doc` Type。
- **Shard（分片）**：索引被水平划分为多个分片，分 Primary 主分片（索引创建时确定，不可修改）和 Replica 副本分片（可动态调整）。分片分布在不同节点。
- **Document（文档）**：ES 中最小的数据单元，JSON 格式，每个文档有唯一的 `_id`。
- **Mapping（映射）**：定义索引中字段的类型和分词方式（类比数据库的 Schema）。ES 8.0+ 默认开启安全认证。
- **Analyzer（分词器）**：由 Character Filter（字符过滤器，预处理文本）、Tokenizer（分词器，将文本拆分为词条）、Token Filter（词条过滤器，处理词条如大小写转换/同义词）三部分组成。
- **倒排索引**：核心数据结构，将文档中的词条（Term）映射到文档 ID 列表。通过 Term Dictionary（存储所有 Term，使用 FST 压缩）和 Term Index（FST 数据结构，内存驻留，快速定位到 Term Dictionary）检索。
- **Term Dictionary**：存储索引中所有词条的有序字典，以 FST（Finite State Transducer）结构存储在磁盘上。
- **Term Index**：FST 结构的词典索引，驻留在内存中，通过前缀匹配快速定位 Term Dictionary 中对应的 Block。
- **Doc Values**：列式存储结构，用于排序、聚合等操作。文档写入时建立，存储在磁盘上。
- **Stored Fields**：原始文档字段存储，按文档 ID 读取（`_source` 字段启用时存储原始 JSON）。
- **Segment（段）**：倒排索引的最小存储单元，不可变。新文档写入生成新 Segment，多个 Segment 在后台定期合并（Force Merge）。
- **Master 节点**：负责集群管理（创建/删除索引、分配分片、节点管理），不负责数据读写。
- **Data 节点**：存储数据、执行数据操作（CRUD、搜索、聚合）。
- **Coordinator 节点**：协调节点，接收客户端请求，路由到对应数据节点汇聚结果返回。每个节点默认都是 Coordinator。
- **写入流程**：请求 → 协调节点 → 路由到主分片所在节点 → 写入内存 Buffer + 写入 Translog（WAL 预写日志）→ 返回成功。内存 Buffer 满或 Refresh 触发 → 生成新 Segment 写入 OS Cache 可搜索。Translog 达到阈值触发 Flush → Segment 提交到磁盘（持久化）。
- **Refresh**：默认每 1s 将内存 Buffer 中的数据生成新 Segment（只读）写入 OS Cache，此时数据变为可搜索。`refresh_interval` 可调整。
- **Flush**：将 OS Cache 中的 Segment 通过 `fsync` 刷入磁盘，同时清除 Translog。默认 30 分钟或 Translog 达到 512MB 触发。
- **Translog**：写入先记录 Translog 保证数据可靠性。默认每 5s 或每次写入后（`index.translog.durability=request`）刷盘。
- **读取流程**：查询 → 协调节点 → 并行请求各分片（通过快照避免并发冲突）→ 分片返回文档 ID 和排序值 → 协调节点排序聚合 → 根据 ID 重新请求完整文档数据 → 返回客户端。
- **Query DSL**：Elasticsearch 的查询语言，包括 `match`（全文搜索）、`term`（精确匹配）、`match_phrase`（短语匹配）、`multi_match`（多字段匹配）、`bool`（组合查询，must/filter/should/must_not）、`range`（范围查询）。
- **聚合（Aggregation）**：分 Bucket Aggregation（分组，类似 GROUP BY，如 terms/histogram）和 Metric Aggregation（度量，类似 COUNT/SUM/AVG）。
- **IK 分词器**：中文分词插件，`ik_smart`（粗粒度分词）和 `ik_max_word`（细粒度分词），支持自定义词典扩展。
- **Nested（嵌套）**：用于存储对象数组，独立文档索引，保持对象内字段关联，查询时用 `nested query`。
- **Join 类型**：父子关系文档，同索引中多类型关联，查询用 `has_child` / `has_parent`。ES 官方建议用扁平化宽表或应用侧关联替代 Join。
- **调优**：
  - `refresh_interval` 设置更长（如 30s），减轻写入负担。
  - 合理设计分片数量（单个分片 10~50GB 为宜，集群分片总数不宜过大）。
  - 禁用 Dynamic Mapping（或限制 `dynamic: strict`）避免字段爆炸。
  - 合理设置副本数（写入量高时可 `number_of_replicas=0` 后续再增）。
  - 对不搜索的字段禁用 `index` 属性。
  - 使用 SSD 磁盘。
  - 定时 Force Merge 只读索引合并 Segment（业务低峰期进行）。
- **集群健康状态**：Green（所有主分片和副本正常分配）、Yellow（所有主分片正常但部分副本未分配）、Red（部分主分片未分配，数据不完整）。
- **版本变化**：
  - 6.x：支持多 Type（已 Deprecated）。
  - 7.x：默认 `_doc` Type，废弃多 Type。
  - 8.x：移除 Type 概念，`_source` 仍保留；默认安全认证开启（TLS + 密码）。
  - 8.0+ 支持向量检索和 NLP 特性。
- **Bulk 批量操作**：批量导入/索引/更新/删除，减少网络开销。建议 batch 大小 5~15MB。
- **深度分页问题**：
  - `from+size`：性能随页数增大急剧下降（协调节点需在内存中排序前 from+size 条结果）。
  - `scroll`（游标）：生成搜索快照，适合大批量数据导出，不推荐用于现用户实时分页。
  - `search_after`：基于上一页排序值唯一标识下一页起始位置，性能效率高，适合实时分页和深度分页。
- **适用场景**：全文搜索（站内搜索/电商搜索）、ELK 日志分析（Elasticsearch + Logstash + Kibana）、APM（应用性能监控）、业务数据分析。

# 2. 面试题

## 2.1 Elasticsearch 的倒排索引是如何工作的？
倒排索引将文档"分词-文档ID"的正向关系反转，建立"词条→文档ID列表"的映射。查询时：
1. 对查询条件分词，得到多个词条（Term）。
2. 通过 Term Index（FST 结构，内存中）快速查找到 Term Dictionary 中词条所在位置。
3. 从 Term Dictionary 取出该词条的倒排列表（Posting List），即包含该词条的文档 ID 集合。
4. 对多个词条的倒排列表做交集/并集运算（通过跳表 SkipList 加速）。
5. 根据文档 ID 获取相关性评分（BM25 算法）和排序，返回结果。

倒排索引的不可变性：Segment 一旦写入不可修改，新增数据生成新 Segment，删除通过 .del 文件标记。

## 2.2 Elasticsearch 的写入流程是怎样的？
1. 客户端向任意节点发送写入（Index/Update）请求，该节点成为协调节点（Coordinator）。
2. 协调节点根据 `routing`（默认 `_id`）计算目标分片：`shard = hash(routing) % number_of_primary_shards`，路由到主分片所在节点。
3. 主分片节点：1) 数据写入内存 Buffer；2) 同时将数据写入 Translog（WAL 预写日志）；3) 若副本存在，并行转发写入请求给副本节点。
4. 所有副本写入成功后，主分片返回确认给协调节点。
5. 协调节点汇总结果返回给客户端。
6. 后台进程每 1s（默认 `refresh_interval`）将 Buffer 数据生成新 Segment 刷到 OS Cache，此时数据可被搜索。
7. Translog 达到阈值（30min 或 512MB）时执行 Flush，将 Segment fsync 到磁盘并清空 Translog。

## 2.3 Refresh 和 Flush 有什么区别？
| | Refresh | Flush |
|------|---------|-------|
| 作用 | 将内存 Buffer 数据生成新 Segment 写入 OS Cache | 将 OS Cache 中的 Segment fsync 到磁盘 |
| 数据可见性 | Refresh 后数据可被搜索 | 数据持久化到磁盘 |
| 触发周期 | 默认 1s（`refresh_interval`） | 默认 30min 或 Translog 512MB |
| 性能影响 | 轻量级（只写 OS Cache） | 较重量级（fsync 磁盘） |

## 2.4 Elasticsearch 的集群健康状态（Green/Yellow/Red）分别是什么含义？
- **Green**：所有主分片和副本分片都正常分配（集群完全健康）。
- **Yellow**：所有主分片正常分配，但部分副本分片未分配（例如只有 1 个节点但设置了 1 个副本）。数据完整，搜索正常，但存在单点丢失风险。
- **Red**：部分主分片未分配（节点宕机导致主分片丢失且无可用副本）。数据不完整，部分索引不可用。

## 2.5 Elasticsearch 如何解决深度分页问题？
- **问题**：`from+size` 方式深度分页时，协调节点需从各分片获取 `from+size` 条数据在内存中排序，越往后性能越差（内存和 CPU 开销大）。
- **Scroll**：生成数据快照（快照时间点的数据，后续变更不可见），保持游标遍历结果。适合一次取大量数据的场景（如数据导出），不支持实时分页。
- **search_after**：通过上一页最后一条数据的排序值作为搜索条件，跳过之前的数据，实时分页且无性能下降。推荐用于实时深度分页需求（如商品列表无限滚动）。
- 实践中：限制 `max_result_window`（默认 10000），用 `search_after` 替代深度 `from+size`。

## 2.6 Elasticsearch 的 Master 节点和数据节点的职责分别是什么？
- **Master 节点**（`node.master: true`）：负责集群管理（索引创建/删除、分片分配、节点管理）、维护集群元数据。不负责数据查询和写入。配置 `node.data: false` 专做管理节点避免负载干扰。
- **Data 节点**（`node.data: true`）：存储数据，执行数据操作（CRUD、搜索、聚合）。
- **Coordinator 节点**：接收请求并路由，每个节点默认都是协调节点。集群较大时可设置专门的协调节点（`node.master: false, node.data: false`）分担协调负载。

## 2.7 Elasticsearch 的 Doc Values 是什么？与 `_source` 有何不同？
- **Doc Values**：列式存储结构的字段值，按文档 ID-字段值对存储。主要用于排序、聚合操作。在 Segment 写入时建立，存储在磁盘上，通过操作系统 PageCache 访问。
- **`_source`**：行式存储，按文档 ID 存储完整的原始 JSON。用于返回命中文档的完整内容。
- **区别**：查询返回文档走 `_source`，排序/聚合走 Doc Values。Doc Values 列式存储适合聚合场景而 `_source` 适合取原始返回。

## 2.8 Elasticsearch 的 Segment 合并（Force Merge）是什么？
Segment 是倒排索引的最小不可变单元。频繁写入会产生大量小 Segment，影响搜索性能（每次搜索需遍历所有 Segment）。ES 后台自动进行 Segment 合并（Minor/Major Merge），将小 Segment 合并为大 Segment，删除 .del 标记为删除的文档。

`_forcemerge` API 可手动触发合并，将 Segment 数量合并到指定 `max_num_segments`（常设为 1）。建议在低业务峰期对不再写入的只读索引执行。

## 2.9 Elasticsearch 如何做中文分词？IK 分词器的两种模式有什么区别？
Elasticsearch 本身对中文按字切分（效果差，单字词条），中文场景需安装 **IK 分词器**：
- **ik_smart**：粗粒度分词（最简粗分），如"南京市长江大桥" → ["南京市", "长江大桥"]。适合索引量大的业务场景。
- **ik_max_word**：细粒度分词（最大词量），如"南京市长江大桥" → ["南京市", "南京", "市长", "长江大桥", "长江", "大桥"]。搜索召回率更高但索引更大。

也支持 pinyin 分词（拼音插件），允许拼音搜索中文字段。

## 2.10 Elasticsearch 的索引数据如何做设计建模？
- **扁平化优先**：ES 不是一个关系型存储，建议将相关数据打平存入同一索引（宽表结构），避免 Join 和 Nested。
- **Nested 适用**：对象数组且需要维护数组中字段的关系做关联查询时使用（如订单下多个商品属性关联）。
- **Join 适用**：仅父子文档一对多且父文档更新频繁需要子文档级联更新的场景，ES 官方建议不要大量使用。
- **大索引设计**：对于时序数据（如日志），按时间切索引（如 `logs-2024.01`），方便归档和删除。
- **Mapping 设计**：根据字段用途设定合适的类型和分词器，不搜索的字段设为 `index: false`，不聚合的字段禁用 `doc_values`。

## 2.11 如何调优 Elasticsearch 的写入性能？
1. **加大 `refresh_interval`**：如设为 `30s` 或 `-1`（关闭自动刷新，批量导入完再恢复），减少 Segment 合并频率。
2. **批量写入（Bulk）**：使用 `_bulk` API 批量提交，调整 batch 大小（5~15MB）。
3. **暂减副本数**：写入高峰期将 `number_of_replicas` 设为 0，写入完成后恢复。
4. **合理分片数**：单节点分片数不超过节点 JVM 内存的 1/20（如 32G 堆内存不超过 640 个分片）。
5. **使用 SSD**：磁盘 I/O 是写性能的主要瓶颈，SSD 显著提升性能。
6. **增大 Translog flush 间隔**：为提升写入吞吐可适当增大 `index.translog.flush_threshold_size`。

## 2.12 Elasticsearch 和关系型数据库的概念对比？
| Elasticsearch | 关系型数据库 |
|--------------|----------|
| Cluster | 数据库集群 |
| Index | Database（数据库） |
| Type（7.x 废弃） | Table（表） |
| Document | Row（行） |
| Field | Column（列） |
| Mapping | Schema（表结构） |
| DSL | SQL |
| GET/PUT/POST/DELETE | SELECT/INSERT/UPDATE/DELETE |

## 2.13 ES 6.x → 7.x → 8.x 的变化有哪些？
| 版本 | 变化 |
|------|------|
| 6.x | 多 Type 支持但标记废弃 |
| 7.x | 默认 `_doc` Type，每个索引只有单一 Type；移除映射中的 Type 概念；Java High Level REST Client 引入 |
| 8.x | 彻底移除 Type；HLLRC 移除改用新的 Java API Client；安全功能默认开启（TLS + 密码）；支持 KNN 向量搜索 |
