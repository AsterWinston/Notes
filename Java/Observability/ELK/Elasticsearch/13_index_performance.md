# 1. 写入的成本模型

先把"一次 index 请求里 ES 到底干了什么"想清楚，所有写入优化都是从这张账本上砍项：

```text
一次写入的开销构成:
  网络/HTTP + 请求解析
  → Ingest Pipeline(如果有)
  → 路由计算, 转发到主分片
  → 版本检查(get split: 有无 seq_no 冲突)
  → translog 追加(+按 durability fsync)      ← IO 大头
  → 解析文档、按 mapping 推断字段类型
  → Lucene addDocument: 分词 + 构建倒排/doc_values/norms  ← CPU 大头
  → 并行复制到各副本(同样的活再干一遍)        ← 网络 + CPU 翻倍
  → (异步) refresh 建段 / merge 合并段          ← 后台 IO/CPU, 与写争资源
```

**结论：写入优化的本质是"摊薄固定开销"（bulk 摊掉每文档的网络、fsync、副本握手）、"延后可见性"（refresh）、"减少复制份数"（副本）、"减少索引结构"（mapping 选项）四条路。** 单文档 PUT 的延迟大头不是 Lucene，而是往返与 fsync，第一条军规：永远用 bulk。

# 2. Bulk 优化

```bash
# 最佳实践
# 1. 每批 5-15MB(字节数, 不是条数!)
# 2. 文档数 500-1000 起步, 按压测调整
# 3. 多线程并行提交
# 4. 检查 response errors, 处理 429

# ndjson 格式: action 行 + 文档行交替
POST /_bulk
{"index":{"_index":"orders","_id":"1"}}
{"order_no":"ORD001","amount":100.0}
{"index":{"_index":"orders","_id":"2"}}
{"order_no":"ORD002","amount":200.0}
```

**bulk 的调参单位是"批的字节数"而不是条数**：小文档 1000 条才 300KB 远远不够，大文档（5KB/条）1000 条正好 5MB。**起点 5-15MB，逐步加压找拐点**——加大批量直到吞吐不再上升或出现 429/延迟陡增，回退一档即该集群最优批。别迷信"越大越好"：过大批次会在协调节点攒整个响应、在分片上挤占 write 线程池更久，还放大失败重试的代价。

并行策略与背压处理同样关键：

```text
提交端模型:
  N 个 worker 线程 ── 每线程持有 1 个在途 bulk ──▶ 集群
  在途 bulk 总数 ≈ write 线程数 × 节点数(起步), 超过会排队进 write queue
  write queue 满(默认 10000) → 429 es_rejected_execution_exception
       → 客户端必须退避重试(指数退避 + jitter), 不能盲目重放
```

```java
// Java 新版客户端(elasticsearch-java 8.x): 攒批按字节提交, 必须逐条检查失败
BulkResponse resp = esClient.bulk(bulkReq, o -> o);   // bulkReq 内已按 ~8MB 切批
if (resp.errors()) {                                   // 顶层 200 也可能有条目级失败
    resp.items().stream().filter(it -> it.error() != null)
        .forEach(it -> retryOrDeadLetter(it));         // 429 重放, 4xx 进死信
}
```

> **注意**：bulk 响应 **HTTP 200 不代表全部成功**——必须检查 `items[].error`。429（rejected）与 503 是可重放的临时错误，400（mapping 冲突、字段超长）重放一万次也不会成功，要落死信队列。只盯着 200 是最常见的静默丢数据原因。

**设计考量**：

- **同一批尽量同索引/同分片**：混多索引不会报错，但协调节点要拆路由，失去批量转发红利；租户型写入可按 `_routing` 分组攒批；
- **自定义 `_id` 触发版本检查**（先查旧文档、写 tombstone），比自动 ID 慢且有段膨胀：**可重放场景用业务键当 ID 换幂等，纯追加日志用自动 ID 换吞吐**；
- **重试必须幂等**：固定 `_id`，或接受"重复写覆盖同 ID"的语义。

# 3. Refresh Interval

```json
# 批量导入时关闭 refresh
PUT /orders/_settings
{ "index.refresh_interval": "-1" }

# 导入完成后恢复并手动刷新
PUT /orders/_settings
{ "index.refresh_interval": "30s" }
POST /orders/_refresh
```

**refresh 的代价不是 fsync，而是"造小段"**：每秒一次的 refresh 在高速写入下源源不断产生小段，tiered merge 要花数倍 IO/CPU 合回去——**段合并风暴才是 refresh_interval=1s 拖垮吞吐的真凶**。导入期设 `-1`（停止自动 refresh，新数据要手动 `_refresh` 或恢复设置后才可搜索），写完恢复，吞吐提升 2-5 倍就是这么来的。

对常驻日志索引，常用折中是 5s 或 30s：牺牲一点可搜索延迟，换来段的"老龄化"，merge 压力显著下降。注意 **searchidle** 机制（30 秒无查询自动暂停 refresh，8.x 默认开启）：日志索引若没人查，refresh 本来就会自动暂停——"关了 refresh 吞吐也没涨多少"是正常现象，不是配置没生效。

# 4. Translog 优化

```json
# 批量导入时: 异步 + 增大阈值
PUT /orders/_settings
{
  "index.translog.durability": "async",     // 异步刷盘
  "index.translog.sync_interval": "30s",    // 30s fsync 一次
  "index.translog.flush_threshold_size": "1gb"  // 攒到 1GB 才 flush
}

# 导入完成后恢复
PUT /orders/_settings
{
  "index.translog.durability": "request"    // 每次请求 fsync(默认, 安全)
}
```

默认 `durability: request` 下，每个（含 bulk 里的每个）写请求都在返回前 fsync translog——**bulk 整批共享一次 fsync，这也是"必须用 bulk"的又一硬件层理由**。改成 `async` 后 fsync 交给 `sync_interval` 定时器，bulk 之间不再等盘，机械盘上收益尤其明显；风险窗口 = 一个 sync_interval 的数据，宕机（进程死）不丢、断电才丢。

`flush_threshold_size` 的作用链：translog 攒到阈值 → flush（段 fsync + commit point + 截断 translog）→ 崩溃后要重放的日志变短 → 分片恢复更快。调大到 1GB 减少 flush 频率，但故障恢复时要重放更多日志——这是"日常吞吐 vs 恢复时长"的旋钮，不是纯性能开关。

> **注意**：网上大量 5.x/6.x 教程的配置键在 8.x 已废弃或改名，改 settings 前先对照当前版本文档，别整段抄旧博客。

# 5. 副本策略

```json
# 批量导入前: 关闭副本
PUT /orders/_settings
{ "index.number_of_replicas": 0 }

# 导入完成: 添加副本(自动同步)
PUT /orders/_settings
{ "index.number_of_replicas": 1 }

# 等待副本就绪
GET /_cluster/health/orders?wait_for_status=green&timeout=60s
```

**N 个副本 = 每份数据被解析、分词、建倒排 N+1 遍**。导入期置 0，写完再置回，新副本直接从已建好的段做文件级复制（跳过 Lucene 索引构建），比边写边同步便宜得多——**"先 0 副本灌数据、后加副本"是初始化大索引的标配流程**，同时省掉导入期的副本确认往返。副本写失败不会拖垮写入：该副本被标记 stale，主分片继续服务，恢复后自动追平。

**设计考量**：

- 读多写少的检索索引：副本加到 2，顺便扩大查询并行度（副本也参与搜索）；
- 写多的日志索引：1 副本封顶，查询吞吐不足优先加节点、加索引并行度，而不是加副本；
- 导入期置 0 期间集群无冗余，节点故障 = 数据全丢，所以**只在"可重放数据源"（Kafka/对象存储备份）存在时这么干**。

# 6. Mapping 与动态映射治理

写入路径上每个字段都要经历"有没有映射 → 没有则动态推断类型 → 更新 mapping → mapping 变更要广播到全集群并刷新每个分片"这条链，**动态映射（dynamic mapping）在字段不可控的数据上是性能与稳定双重风险**。

## 6.1 Mapping 爆炸（mapping explosion）

最典型的成因是"把用户/租户/日期当字段名"：

```json
// 危险写法: 每条文档动态创建新字段
PUT /metrics/_doc/1
{ "user_1001": { "click": 10 }, "2024-01-15": { "uv": 99 } }
// 一天之后: 字段数破万, 集群状态膨胀, 每次写入广播 mapping, 查询规划变慢
```

防护手段一套配齐：

```json
PUT /metrics
{
  "settings": {
    "index.mapping.total_fields.limit": 1000,     // 字段总数上限(默认 1000)
    "index.mapping.depth.limit": 10,              // 嵌套深度上限
    "index.mapping.nested_fields.limit": 50,
    "index.mapping.field_name_length.limit": 100
  },
  "mappings": {
    "dynamic": false,                             // 未知字段: 存进 _source 但不索引(不报错)
    "dynamic_templates": [
      { "strings_as_keyword": {
          "match_mapping_type": "string",
          "mapping": { "type": "keyword", "ignore_above": 256 } } },
      { "disable_norms": {
          "match": "tags_*", "match_mapping_type": "string",
          "mapping": { "type": "keyword", "doc_values": false } } }
    ],
    "properties": {
      "values": { "type": "flattened" }           // 键值对集合场景: flattened 类型防爆炸
    }
  }
}
```

| dynamic 取值 | 行为 | 适用 |
|--------------|------|------|
| `true`（默认） | 新字段自动建索引，类型靠猜 | 字段集确定且少 |
| `false` | 新字段存进 _source、可 mget，但不可搜/不可聚合 | 日志类，已知字段才建索引 |
| `strict` | 出现未知字段整个文档报错 400 | 强 schema 业务数据 |

> **行业实践**：日志/指标场景一律 `"dynamic": "false"` + 显式 mapping 常用字段 + `dynamic_templates` 兜底字符串按 keyword；而**宽表型"每列一个字段"的建模（MAP 型业务数据）是 mapping 爆炸的头号来源，改用 `flattened` 类型或把键做成值（`{k: user_1001, v: 10}` 长表）才是正解**——"字段爆炸"和"文档爆炸"是两种规模问题，前者炸集群状态，后者只是大，别用撑爆 mapping 的方式解决容量问题。

字段类型本身也是性能账：不需要全文检索的字符串（状态、枚举、URL）用 `keyword` 而不是 `text`;range 查询与聚合一律走 `long`/`date` 等数值类型（BKD 树），别用 keyword 做字典序比较；`ignore_above` 对"超长但从不搜索"的字段（截图 base64、User-Agent）既防词项爆炸又省 CPU。

## 6.2 关闭不需要的结构

```json
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "norms": false,                  // 不做字段长度归一, 纯 filter 查询时可关
        "index_options": "freqs"         // 只要词项+词频, 省 positions 的写与读
      },
      "status": {
        "type": "keyword",
        "doc_values": false              // 不排序不聚合 → 省一份列式存储
      },
      "large_field": {
        "type": "text",
        "index": false                   // 只展示不检索: 不进倒排, 省写 CPU
      },
      "internal_id": {
        "type": "keyword",
        "index": false, "doc_values": true   // 只排序不搜索(常见于排序游标字段)
      }
    }
  }
}
```

**倒排、doc_values、norms、positions 每一样都在 addDocument 时消耗 CPU、在 merge 时消耗 IO**——只用于展示的正文关掉 `index`，不排序聚合的 keyword 关掉 `doc_values`，写入吞吐和磁盘能省 10-30%。禁用 `_source` 会失去 reindex/update/高亮能力，不推荐；`_all` 在 7.x 已彻底移除。

# 7. Indexing Buffer 与线程池

```json
# 增大索引缓冲区(相对堆的比例, 集群级动态设置)
PUT /_cluster/settings
{ "persistent": { "indices.memory.index_buffer_size": "30%" } }
```

`indices.memory.index_buffer_size`（默认 10%，节点级堆比例）控制**所有分片共享的 Lucene IndexWriter 内存 buffer**：buffer 越大，落盘的小段越少越大，merge 压力越小；但堆里可用作缓存与聚合熔断的空间同步缩小。**8GB 以下堆别动它**，31GB 大堆 + 写密集才值得调到 20-30%，改后盯 GC——buffer 调爆换来 full GC，吞吐反而断崖。

```bash
GET /_cat/thread_pool/write?v&h=node_name,active,queue,rejected
# rejected 持续增长 = 客户端并发/bulk 太大, 先降提交端压力, 不是先调大 queue
```

write 队列默认 10000，调大 queue 只会把拒绝变成更长的排队与更高内存，把 429 的背压信号"调没了"是负优化。

# 8. 段合并与 Index Sorting

## 8.1 合并策略

```json
# 导入完成后强制合并(仅限不再写入的索引!)
POST /orders/_forcemerge?max_num_segments=1

# 动态调整合并策略
PUT /orders/_settings
{
  "index.merge.policy.max_merged_segment": "5gb",
  "index.merge.scheduler.max_thread_count": 1
}
```

写入期与合并的平衡点：tiered 策略下 `max_merge_at_once`（默认 10）决定一次挑几个段、`max_merged_segment`（默认 5GB）决定段大小天花板——**日志索引可以把 max_merged_segment 提到 10-20GB 减少段数、利好查询，代价是单次 merge 更重**。机械盘节点 `max_thread_count: 1`（否则并发合并把 HDD 随机 IO 打满），SSD/NVMe 可放开或用默认（按核数）。`_forcemerge` 放在导入流程最后一步，且**先删掉过期/无谓数据再合并**（合并顺带物理清除 tombstone，先清理后合并才不白干）。

## 8.2 Index Sorting 的写入代价

```json
PUT /orders
{
  "settings": {
    "index.sort.field": ["tenant_id", "created_at"],
    "index.sort.order": ["asc", "desc"]
  }
}
```

索引排序（见 12 篇的读收益）在写入侧是**明确的负资产**：每篇文档插入 Lucene 时都要按排序键定位，近似"在有序数组中间插入"，写放大显著；租户/时间型日志索引按 `[@timestamp]` 单字段排序尚可接受，多字段 + 高基数的组合务必压测。**默认关闭，读收益（段级剪枝、排序提前终止）确实存在时才开**。

# 9. 批量导入 Checklist

```text
1. □ refresh_interval = -1(导入期)
2. □ number_of_replicas = 0(导入期, 前提: 数据源可重放)
3. □ translog: durability=async + sync_interval=30s + flush_threshold_size=1gb
4. □ mapping 预先建好, dynamic=false/strict, 关闭无用结构
5. □ Bulk 每批 5-15MB 起步, 压测找拐点; 检查 items[].error; 429 退避重试
6. □ 提交端并发 = 观察 write 队列不 rejected 的最大值
7. □ 导入完: 恢复 refresh/replicas/translog, 等 green
8. □ forcemerge max_num_segments=1(只读索引), 清理无用别名/快照空间预留
9. □ 全程监控: _cat/thread_pool/write、_nodes/stats/indices/merge、GC 日志、磁盘水位
```

# 10. 写入性能基准（示例形态）

```text
测试环境: 3 节点(8C32G, NVMe), 主分片=3, 副本=1, 文档 ~1KB JSON

阶段                          吞吐(docs/s)   备注
--------------------------------------------------------
裸跑(默认配置, 单线程 bulk 500)   5,000      refresh=1s, request fsync
+ 并发 8 线程, bulk 5MB           12,000      摊薄固定开销
+ refresh=-1, replica=0           25,000      消掉段风暴 + 复制
+ translog async                  30,000      NVMe 上边际收益小, HDD 上翻倍
```

数字只演示"每项优化攻掉账本上哪一项"，**换数据集/硬件结论都会变，照搬别人的基准数字做容量规划是错误做法**；按上表逐项累加压测、记录边际收益，砍掉收益最小且风险最大的项（比如断电窗口）。

# 11. 小结

| 主题 | 核心结论 |
|------|---------|
| bulk | 调参单位是字节（5-15MB 起步）；200 ≠ 全成功，items 级错误必须处理；429 退避重放 |
| refresh | 关它省的不是 fsync 而是小段→merge 风暴；导入期 -1，常驻日志 5-30s;searchidle 让空闲索引自动暂停 |
| translog | request 模式每请求 fsync（bulk 摊薄）；async 丢电窗口 = sync_interval;flush 阈值换的是恢复时长 |
| 副本 | 1 副本 = 索引工作 ×2；导入期置 0 后加副本走文件复制；置 0 窗口无冗余，数据源需可重放 |
| mapping | dynamic=false + dynamic_templates + total_fields.limit；把键当字段名 = 爆炸；flattened 救宽表 |
| 结构裁剪 | index/doc_values/norms/positions 每样都要付写成本，按需关闭 |
| buffer/合并 | 小堆别动 index_buffer;HDD merge 线程 1;forcemerge 只给只读索引、先清理后合并 |
| index sorting | 读优化、写惩罚，默认不开 |

写入侧就绪后，集群层面的"长期运转"问题（水位线、健康色、生命周期、扩缩容）是下一篇（14_cluster_management）的主题。
