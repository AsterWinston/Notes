
# 1. 基础题

## 1.1 什么是 Elasticsearch？和传统数据库有什么区别？

ES 是基于 Lucene 的分布式搜索和分析引擎。核心区别：ES 使用倒排索引实现全文检索，通过分片水平扩展；传统数据库用 B+Tree，适合精确查询和事务。ES 不支持事务，是近实时（~1s）。

## 1.2 倒排索引是什么？

倒排索引将文档中的词项映射到文档 ID 列表。例如 "iPhone" → [doc1, doc5, doc8]。查询时直接通过词项定位文档，O(1) 查找，无需全表扫描。相当于书的索引页 vs 逐页翻找。

## 1.3 ES 的核心概念有哪些？

| 概念 | 说明 |
|------|------|
| Cluster | 集群，多节点组成 |
| Node | 节点，一个 ES 进程 |
| Index | 索引，文档集合（类比数据库） |
| Shard | 分片，Lucene 实例（Primary+Replica） |
| Document | 文档，JSON 数据单元 |

# 2. 原理题

## 2.1 ES 的写入流程是怎样的？

1. 协调节点计算路由：shard = hash(_routing) % num_primary_shards
2. 转发到主分片节点写入 Translog + Memory Buffer
3. 主分片并行转发到副本分片
4. 默认每 1s refresh（Memory Buffer → Segment，数据可搜索）
5. Translog 达到阈值或 30min 执行 flush（fsync 到磁盘）

## 2.2 ES 的读取流程是怎样的？

1. Query Phase：协调节点向所有相关分片发送查询，各分片返回 (docId, score, sort value)
2. 协调节点排序和分页
3. Fetch Phase：按最终排序结果取 Top N 个 docId，向对应分片获取完整文档
4. 返回给客户端

## 2.3 Refresh 和 Flush 有什么区别？

| 操作 | 作用 | 频率 | 代价 |
|------|------|------|------|
| Refresh | Memory Buffer → Segment（可搜索） | 1s（默认） | 轻量 |
| Flush | Segment → fsync 磁盘 + 清空 Translog | 30min 或 Translog 满 | 重量 |

## 2.4 text 和 keyword 有什么区别？

| 类型 | 分词 | 用途 |
|------|------|------|
| text | ✓ 分词，用于全文检索 | 描述、文章内容 |
| keyword | ✗ 不分词，完整匹配 | 订单号、状态、标签 |

# 3. 实践题

## 3.1 深分页问题如何解决？

- scroll：全量遍历（适合导出）
- search_after：实时翻页（推荐）
- PIT (Point In Time)：8.x 推荐
- 避免 from+size 超过 10000

## 3.2 如何优化批量写入？

1. 关闭 refresh（-1）
2. 关闭副本（replica=0）
3. Translog async + 增大阈值
4. Bulk 5-15MB/批
5. 写入完成后恢复设置 + forcemerge

## 3.3 集群变 Red 怎么办？

1. `_cluster/health` 确认状态
2. `_cat/shards` 找未分配分片
3. `_cluster/allocation/explain` 看原因
4. 常见：磁盘满（清理/扩容）、节点离线（恢复）、分片超限（增加节点）

# 4. 架构题

## 4.1 ES 如何保证高可用？

- 主分片 + 副本分片（多副本）
- Master 节点选举（> 2 个 master-eligible）
- 自动故障转移（节点宕机 → 副本提升为主）
- 快照备份（SLM 自动策略）

## 4.2 Hot-Warm-Cold 架构是什么？

按数据热度和时间将数据分布到不同硬件：
- Hot（SSD）：最近数据，高频读写
- Warm（HDD）：中等数据，只读
- Cold（归档）：历史数据，仅查询

通过 ILM 自动迁移，节省成本。

## 4.3 ES vs Solr 如何选择？

| 维度 | ES | Solr |
|------|-----|------|
| 分布式 | 原生 | 需 ZK |
| 实时性 | 近实时 | 近实时 |
| 社区 | ★★★★★ | ★★★☆☆ |
| 生态 | ELK | 较独立 |

新项目推荐 ES（生态更完善）。
