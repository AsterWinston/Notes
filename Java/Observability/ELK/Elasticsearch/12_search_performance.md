# 1. 查询优化的两个层面

搜索性能问题只有两种成因：**做了不必要的工作**（算了分数、扫了不该扫的文档），或者**必要的工作做得太贵**（段太多、分片太多、深分页、通配展开）。优化也必须两层同时下手——DSL 写法（少干活）与集群/索引结构（把活变便宜）。只在 DSL 上抠细节而不解决"单分片 100GB"这种结构问题，是典型的无效优化。

## 1.1 Filter 物化与缓存

```json
GET /orders/_search
{
  "query": {
    "bool": {
      "must":   [{ "match": { "title": "iPhone" } }],
      "filter": [
        { "term": { "status": "active" } },
        { "range": { "created_at": { "gte": "now-7d/d" } } }
      ]
    }
  }
}
```

**filter 上下文做三件事：不算分、可缓存、可用 bitset 求交**。它把条件编译成 Roaring Bitmap（每个命中文档一个 bit），多个 filter 之间是位运算 AND，而不是逐文档评分。**把不需要相关性的条件全部挪进 filter 是收益最直接的改写**——"状态、租户、时间范围"这三个业务必带条件放在 must 里，等于每个候选文档白算一次 BM25。

## 1.2 Profile API

```json
GET /orders/_search
{
  "profile": true,
  "query": { "match": { "title": "iPhone" } }
}

# 配套诊断: 看查询被编译成什么, 不执行
GET /orders/_validate/query?explain=true&rewrite=true
```

profile 返回按分片、按层级的耗时树：`type`（query/rewrite/build_scorer/next_reader）、`description`（实际执行的 Lucene 查询对象）、`time_in_nanos`。**用法是"读 description 而不是只看数字"**：同一句 match,description 里是 `TermQuery` 还是 `MultiTermQuery` 完全决定了要不要继续优化；`build_scorer` 占比高说明候选文档太多，`rewrite` 高说明查询构造本身贵（典型是 wildcard 展开）。

> **行业实践**：排查慢查询固定动线：`_validate?explain` 看查询编译成什么 → profile 看候选文档数是否正常 → "扫了很多无关文档"就改索引结构（排序/分区/字段类型），"候选数正常但耗时长"才看段数与缓存。先看候选数，后看耗时，顺序反了就会做无效优化。

# 2. 缓存体系

| 缓存类型 | 位置 | 缓存内容 | 失效条件 | 开关 |
|---------|------|---------|---------|------|
| **Node Query Cache** | 节点级 | filter 子句的 bitset（按 segment） | segment 变化（force merge/删除即失效） | `indices.cache.filter.size`（默认堆 10%） |
| **Shard Request Cache** | 分片级 | 整个请求的结果（top hits + 聚合） | 分片 refresh | `index.requests.cache.enable: true` |
| **文件系统缓存** | OS | Lucene 段文件页 | OS LRU | 靠"堆一半 + 剩给 page cache"保证 |
| **Field Data Cache** | 节点级 | fielddata（正排内存结构） | 手动 clear / 段变化 | 默认禁用（text 需显式开） |

```json
# 查看缓存统计
GET /_nodes/stats/indices/query_cache,request_cache,fielddata
```

**Shard Request Cache 的适用面比想象的窄**：它缓存"请求 + 分片"粒度的完整响应，默认 1s refresh 让频繁写入的索引每秒失效一次。真正划算的是 Kibana 大盘里**重复执行、结果可容忍秒级滞后、计算极贵**的聚合请求：给这类索引开 `index.requests.cache.enable`，并在请求里用 `now-1h/h` 这类**对齐到整点的时间表达式**——相对时间不 round，缓存 key 每次都不一样，命中率归零。

> **注意**：query cache 命中率低是常态而非异常——filter bitset 按 segment 存储，写入产生的每个新 segment 都要重算。指望"加缓存解决慢查询"经常是缘木求鱼，ES 的读性能主要靠**文件系统缓存 + 段设计 + 减少候选文档**，缓存只是锦上添花。

# 3. 排序、字段裁剪与分片设计

## 3.1 关闭评分

```json
GET /orders/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "status": "active" } }
      ]
    }
  },
  "sort": [{ "created_at": "desc" }]
}
```

**当查询里没有任何"文本相关性"需求时，整条查询应该只由 filter 组成**：全 filter 的查询在 Lucene 层退化成 bitset 收集，可触发 constant score 优化。`"track_total_hits": false`（不数精确总数）与 `"track_scores": false` 是两个免费提速开关——**大索引上的 `track_total_hits: true` 是很多人忽略的隐形成本**，它强制扫完全部匹配文档才能给出 total。fetch 阶段同理：`_source` 白名单只取必要字段，大文本（正文、堆栈）用 excludes 排除、详情页再取，能把取回成本砍掉一半以上。

## 3.2 用时间/分区裁剪分片

**最有效的一条读优化是"少扫分片"**：协调节点向 N 个分片扇出，每个分片的固定开销（打开 searcher、构造 weight、维护 size 堆、序列化结果）相加就是尾延迟——**查询延迟由最慢的分片决定**，所以分片数越多、越不均，尾延迟越差。手段：① 按时间索引 + `@timestamp` 范围过滤，让 ES 在 routing 阶段剪掉整个索引（只查相关的 `logs-2024.01.15`）；② 用 `preference`/`routing` 缩小分片集；③ 用 index sorting 让 Lucene 在段级提前终止收集。

> **注意**："1 主 1 副 × 20 分片"在 6 节点集群上是 120 个分片，搜索请求要扇出到 20 个分片复制组。经验值：**单分片 10-50GB，搜索型索引每分片 2000 万文档以内，总"能同时被一个查询命中"的分片数控制在几十以内**。宁可索引数多（按天）靠剪枝少扫，也不要一个巨型索引被全量扇出。

# 4. 深分页解决方案

深分页的根源在 §1 提到的 query-then-fetch：每个分片必须收集 `from+size` 条再全局归并，内存与 CPU 都随偏移线性膨胀——这就是 `index.max_result_window: 10000` 保护存在的原因（限制的是**分片上排序队列大小**，不是"最多能返回 1 万条"）。

## 4.1 from+size（默认，有限制）

```json
GET /orders/_search
{
  "from": 10000,
  "size": 20
}
# 限制: index.max_result_window = 10000 (from+size 超过即报 search_phase_execution_exception)
```

它的合理区间就是"人类会点的页码"：前几页。调大 max_result_window 能治标（不报错），但每往深走一页都更慢更吃内存——没有任何业务应该依赖第 5000 页，把它当性能反模式看待。

## 4.2 Scroll（遍历全量，已不推荐）

```json
# 创建 scroll
GET /orders/_search?scroll=5m
{ "size": 1000, "query": { "match_all": {} } }

# 后续翻页
POST /_search/scroll
{ "scroll": "5m", "scroll_id": "DXF1ZXJ5..." }

# 清理(不显式清理会一直占资源直到超时)
DELETE /_search/scroll
{ "scroll_id": "DXF1ZXJ5..." }
```

**scroll 是一个"搜索上下文的快照"**：首次请求为每个分片建立一份 searcher 视图，后续翻页沿"当前游标位置"继续读，不需要重排前面的结果，所以深翻页成本是常数级。代价有两条：一是快照语义，滚动期间新建/更新的文档看不到；二是 **scroll 上下文在分片上驻留**，每个都在吃堆与文件句柄，并发几十个全量导出就可能把节点拖垮。官方定位已明确：**scroll 只用于一次性全量导出（reindex 内部就用它），用户交互式翻页请一律 PIT + search_after**。

## 4.3 Search After（推荐，实时）

```json
GET /orders/_search
{
  "size": 20,
  "query": { "match": { "status": "active" } },
  "sort": [{ "created_at": "asc" }, { "_id": "asc" }],   // 必须带唯一字段做 tiebreaker
  "search_after": [1705302000000, "order_id_xxx"]         // 上一页最后一条的 sort 值
}
```

search_after 的机制是**"给定排序值，只要排在它之后的记录"**，不维护任何服务端状态，每次请求独立执行，天然水平扩展。限制有三条：① **只能顺序前进，不能跳页**；② 排序必须确定且唯一，要加 `_id` 或 `_shard_doc` 做 tiebreaker;③ 翻页期间写入导致排序值漂移时会漏/重，需要 PIT 固定视图。

## 4.4 PIT + search_after（8.x 官方推荐）

```json
# 创建 PIT: 固定一个"搜索视图"
POST /orders/_pit?keep_alive=5m
// { "id": "gb1..." }

# 使用 PIT 搜索
GET /_search
{
  "size": 20,
  "pit": { "id": "gb1...", "keep_alive": "5m" },
  "query": { "match": { "status": "active" } },
  "sort": [{ "@timestamp": "asc" }, { "_shard_doc": "asc" }],
  "search_after": [1705302000000, "..."]
}

# 用完关闭(不关会按 keep_alive 占资源)
DELETE /_pit
{ "id": "gb1..." }
```

**PIT（Point In Time）补上了 search_after 缺的那一半：一致的分页视图**。创建 PIT 时锁定各分片的 segment 组合，后续所有请求都在同一份数据快照上分页，翻页期间不会因 refresh 导致文档漂移；keep_alive 是"滑动过期"（每次访问续期）而非绝对 TTL。PIT 上下文同样占用分片资源，但它是"轻量元数据 + 引用 segment"，比 scroll 的每分片游标便宜得多，并且支持聚合、script sort、跨页一致。`_shard_doc` 是 PIT 提供的最高效 tiebreaker（按 Lucene 内部文档号排序，无需读字段值）。

`search_after` 依赖排序值翻页、不维护服务端状态；而 **PIT + search_after 才是"翻页期间结果稳定"的完整方案**（Web 列表上一页/下一页、无限滚动 feed 的标准组合）。"导出全表/重建索引"用 scroll 或 reindex；"用户要跳页码"的需求，答案通常是改产品为"筛选条件收窄"（日期、状态、关键字）而不是把 max_result_window 调到 100 万。

# 5. 分页方案对比与慢日志

| 方案 | 适用场景 | 实时性 | 深度分页 | 跳页 | 服务端资源 |
|------|---------|--------|---------|------|-----------|
| from+size | 浅分页（前几页） | 实时 | ✗ ≤ max_result_window | ✓ | 无 |
| scroll | 全量导出 | 快照 | ✓ | ✗ | 重（每分片上下文） |
| search_after | 顺序翻页 | 实时 | ✓ | ✗ | 无 |
| PIT + search_after | 8.x 交互式搜索/分页 | 快照一致 | ✓ | ✗ | 轻 |

慢查询的定位应该常态化，而不是等投诉：

```json
PUT /orders/_settings
{
  "index.search.slowlog.threshold.query.warn": "10s",   // 查询阶段(收集)
  "index.search.slowlog.threshold.query.info": "5s",
  "index.search.slowlog.threshold.fetch.warn": "1s",    // fetch 阶段(取文档)
  "index.search.slowlog.level": "info",
  "index.search.slowlog.threshold.query.debug": "2s"
}
# 日志落在节点 logs 目录: elasticsearch_index_search_slowlog.json
```

query 阈值高、fetch 阈值低是有意的：**query 慢说明"筛得慢"（候选集或评分问题），fetch 慢说明"取得慢"（_source 太大、字段多、文档分散在冷存储）**。两类慢的处置方向完全不同，混在一起看就会误诊。

# 6. 前缀/模糊匹配：ngram vs wildcard

通配与"边打边搜"是最容易写出性能炸弹的两类查询。

```json
GET /products/_search
{ "query": { "wildcard": { "title": { "value": "*phone*", "rewrite": "constant_score" } } } }
```

**`*phone*` 这种"中缀通配"必须遍历该字段的全部词项**（词典级别扫描），代价与字段基数成正比，与命中数无关——这是倒排索引的结构性短板：倒排是"term → docs"，通配需要"哪些 term 匹配模式"。前缀 `phone*` 稍好（在有序词典上做一次区间扫描），但仍然是词典操作而非文档操作。

```json
PUT /products
{
  "settings": {
    "analysis": {
      "tokenizer": {
        "ngram_tokenizer": { "type": "ngram", "min_gram": 2, "max_gram": 4, "token_chars": ["letter"] }
      },
      "analyzer": {
        "my_ngram_analyzer": { "tokenizer": "ngram_tokenizer", "filter": ["lowercase"] }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "standard",
        "fields": {
          "ngram": { "type": "text", "analyzer": "my_ngram_analyzer" }   // 索引期切碎, 查询期整词匹配
        }
      }
    }
  }
}
```

| 维度 | wildcard/regexp | ngram（索引期切碎） | edge_ngram（前缀 ngram） |
|------|-----------------|-------------------|------------------------|
| 查询延迟 | 高且不稳定（随词典大小） | 低且稳定（等价 term 查询） | 低 |
| 索引膨胀 | 无 | 大（2-5 倍，gram 多） | 中 |
| 更新代价 | 低 | 高（改一个词要重切所有 gram） | 高 |
| 支持中缀匹配 | ✓ | ✓ | ✗（只前缀） |
| 典型用途 | 管理端偶发查询 | 全文"包含"搜索、拼音/错字 | 搜索建议、自动补全 |
| 相关性质量 | 差（需要 constant_score） | 需注意 phrase/minimum_should_match | 好（可配 `max_matches`） |

**取舍结论：高频用户输入的模糊匹配一律走 ngram 系（把成本搬到索引期），偶发/管理端查询用 wildcard 并接受它慢**。用 wildcard 时记得 `caseInsensitive: true` + `constant_score`（不评分可显著减少开销，8.x 起 wildcard 支持常量评分改写），并配合 `index.mapping.total_fields.limit` 与字段级 `norms: false` 控制面。

## 6.1 Search-as-you-type

```json
PUT /products
{
  "mappings": {
    "properties": {
      "title": {
        "type": "search_as_you_type"      // 自动生成子字段
      }
    }
  }
}

# 自动生成子字段: title / title._2gram / title._3gram / title._index_prefix

GET /products/_search
{
  "query": {
    "multi_match": {
      "query": "iph",
      "type": "bool_prefix",
      "fields": ["title", "title._2gram", "title._3gram"]
    }
  }
}
```

`search_as_you_type` 把"edge_ngram + 多字段 + bool_prefix 查询"这套正确姿势打包成一个字段类型，自动生成 `title` / `title._2gram` / `title._3gram` / `title._index_prefix` 子字段。**"最后一个词做前缀匹配、前面的词做整词匹配"的补全语义靠 `multi_match.type: bool_prefix` 表达**——比自己搭 edge_ngram 更推荐，因为不用手工维护索引/搜索期双 analyzer 配置。

# 7. 索引排序与 docvalues 排序

```json
PUT /orders
{
  "settings": {
    "index.sort.field": ["created_at", "status"],
    "index.sort.order": ["desc", "asc"]
  }
}
```

**index sorting 让每个段内的文档按指定顺序物理排列**，收益有两类：① 按该字段排序的查询不再需要全局堆排序，收集器"读够 size 条就停"（与 search_after 组合时特别明显）；② 带范围的 filter 能做**段级剪枝**——整个段的 min/max 与查询区间无交集时直接跳过。代价是**写入变贵**：每个文档插入都要按序定位，且段合并更重。因此它只适合"查询排序字段极其固定"的场景（时间序列、日志按 tenant+time），不要当成通用加速开关。排序字段本身：用 `doc_values`（默认开）而不是 fielddata；数值字段用 `long`/`date` 别用 `keyword`（字符串排序比较成本高）。

# 8. 小结

| 主题 | 核心结论 |
|------|---------|
| 优化分层 | 先减"候选文档数"（filter、分区、index sorting、剪枝），再谈缓存；只看耗时不看候选数会做错方向 |
| filter 三收益 | 不算分、可缓存、bitset 求交；租户/状态/时间条件必须放 filter |
| 免费开关 | `track_total_hits: false`、`_source` 白名单、`excludes` 大字段、`stored_fields: _none_` |
| 缓存 | request cache 只对"低频变更 + 贵聚合"划算（refresh 1s 会秒失效）；query cache 命中率低是常态 |
| 深分页 | from/size O（from+size） → search_after O（size） 但不可跳页；scroll 只用于导出 |
| 8.x 标配 | PIT + search_after + `_shard_doc` tiebreaker：一致视图 + 顺序翻页 + 低资源占用 |
| 模糊匹配 | 中缀 wildcard 扫全词典，高频场景改 ngram；补全用 search_as_you_type + bool_prefix |
| 慢日志 | query 阈值 vs fetch 阈值分别定位"筛得慢"与"取得慢" |

搜索侧的账算完了；下一篇（13_index_performance）转向写入侧——bulk 批量、refresh、translog、副本与 mapping 如何共同决定索引吞吐。
