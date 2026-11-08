# 1. 查询上下文 vs 过滤上下文

Query DSL 是 ES 的"SQL"，但它的第一个设计分野与 SQL 完全不同：**同一个条件放在不同上下文里，会走两条不同的执行路径**。

| 上下文 | 用途 | 评分 | 缓存 |
|--------|------|------|------|
| **Query** | 相关性搜索 | ✓ 计算 _score | ✗ |
| **Filter** | 精确筛选 | ✗ 无评分 | ✓ 自动缓存 |

```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "iPhone" } }        // 查询上下文，评分
      ],
      "filter": [
        { "term": { "status": "active" } },        // 过滤上下文，无评分，缓存
        { "range": { "price": { "gte": 1000 } } }
      ]
    }
  }
}
```

两条路径的差别在物理层：**query 上下文**要回答"这篇文档像不像"，需要访问词项频率、文档长度等评分因子，逐条算 BM25 得分，且不同查询串的答案不同、缓存无意义；**filter 上下文**只回答"符合/不符合"，命中结果是一个 bitset（每文档一位），可以直接用 Roaring Bitmap 做集合交并——跳过全部评分计算，且这个 bitset 与查询串绑定后可跨请求复用（节点级缓存，LRU 淘汰）。range、term、exists 这类结构天然没有"相关性"可言，放进 filter 既能省评分 CPU 又能吃到缓存；反过来把 `term` 塞进 must 不会报错，但它会为每个命中算一次毫无意义的常数得分，还放弃缓存。

**默认规则**一句话：**"是不是"进 filter，"像不像"进 query**。上例中"iPhone"是用户输入要按相关性排序，进 must；"在售"和"价格≥1000"是硬条件，进 filter。写熟练之后你会发现真实业务查询里 80% 的条件都属于后者。

> **行业实践**：日志检索的 DSL 几乎都是"bool.filter 套一切等值/范围条件 + 极少量 must 全文条件"的形态；监控面板背后那些重复执行的过滤条件（服务名、时间窗口）正因 filter 缓存而便宜。若一个 term 条件区分度极低（如恒真的环境标记），它连 filter 都不必放——在应用层拼查询时就剔除。

# 2. 全文查询（match 家族）

match 家族的共同前提：**查询串先过字段自己的 analyzer 分词，再去倒排索引找 term**。

```json
// match — 分词后匹配（默认 OR 逻辑）
{ "match": { "title": "iPhone 15 Pro" } }

// match — 改 AND / 最少命中数，收紧召回
{ "match": { "title": { "query": "iPhone 15 Pro", "operator": "and" } } }
{ "match": { "title": { "query": "iPhone 15 Pro", "minimum_should_match": "75%" } } }

// match_phrase — 短语匹配（保持顺序与相邻）
{ "match_phrase": { "title": "iPhone 15" } }

// match_phrase_prefix — 前缀短语（搜索建议/自动补全）
{ "match_phrase_prefix": { "title": "iPh" } }

// multi_match — 多字段查询
{
  "multi_match": {
    "query": "iPhone",
    "fields": ["title^3", "description"]  // title 权重 3 倍
  }
}

// query_string — 类 Lucene 语法（面向懂语法的用户/内部工具）
{ "query_string": { "query": "title:iPhone AND price:[1000 TO 2000]" } }
```

逐层拆开机制：

- **match 的默认 operator 是 OR**：`"iPhone 15 Pro"` 被分成三个词，**含任意一个就算命中**，命中词越多得分越高。这解释了新手最常见的困惑"搜 iPhone 15 为什么返回一堆无关 iPhone"——需要收紧就改 `operator: and` 或用 `minimum_should_match`（支持 `75%`、`<3`、`3<90%` 这类阶梯语法，是搜索质量调优的第一杠杆）。
- **match_phrase 不是"字符串相等"**：它要求分词后的 term 序列按顺序相邻（底层用 position 信息比对），所以 `"15 iPhone"` 不会命中 `"iPhone 15 Pro"`，而 match（OR）会。`slop: 1` 允许间隔一个词，用于容错。
- **match_phrase_prefix** 把短语最后一个词放宽为前缀匹配，是搜索框下拉提示的老方案；数据量大时它需要扫 term 字典，性能有限，正经补全走 `completion` 类型或 `edge_ngram`（第 9 篇）。
- **multi_match** 对每个字段独立执行 match 再合并得分——`best_fields`（默认，取最高分字段）与 `most_fields`（各字段得分求和，适合一个字段内容有冗余备份的容错场景）两种策略解决的是"多字段如何合并相关性"这一本质难题；`cross_fields` 则把多字段视作一个大字段做 term 级组合（"标题有 iPhone 描述有 15" 也能命中）。
- **query_string / simple_query_string** 功能最强也最危险：直接暴露语法给用户输入时，未转义的 `AND`、`~`、`*` 会改变整个查询形态，恶意/畸形输入还可能造出昂贵查询。面向普通用户一律包装成 match，query_string 留给内部运维工具。

> **注意**：`zero_terms_query`（分词后无任何有效 term 时怎么办，比如停用词 "the"）默认 `none` 返回空结果，日志检索里常配成 `all`。另外 match 打在 keyword 字段上不报错但**不分词**——整个查询串当一个 term 精确比对，这是第 3 节 term/match 混淆的镜像问题。

# 3. 词项查询

词项（term-level）查询**跳过分词器**，把查询值原样与倒排索引里的 term 比对：

```json
// term — 精确匹配（keyword 类型或 text 的 .keyword 子字段）
{ "term": { "status": "active" } }

// terms — 多值匹配（任一命中）
{ "terms": { "category": ["电子产品", "图书"] } }

// range — 范围查询
{
  "range": {
    "price": {
      "gte": 1000,
      "lte": 5000
    }
  }
}

// date range 支持时区与数学表达式
{
  "range": {
    "created_at": { "gte": "now-7d/d", "lte": "now", "time_zone": "+08:00" }
  }
}

// exists — 字段存在
{ "exists": { "field": "description" } }

// prefix — 前缀查询
{ "prefix": { "order_no": "ORD" } }
```

**term vs match：ES 面试题第一名，也是线上事故第一名。** 判别方法只有一句话——**match 会把输入分词后去查，term 拿输入原样去查**。于是出现四象限：keyword 字段 + term = 正确（订单号、状态、枚举值精确匹配）；text 字段 + term = **查不到**（term "iPhone 15 Pro" 在倒排里不存在，那里只有 iphone/15/pro 三个碎词）；text 字段 + match = 正确（全文搜索）；keyword 字段 + match = 表面能用但语义错误（不分词，整串比对，且多词查询必挂）。

```text
             term("iphone 手机")          match("iphone 手机")
text 字段    ✗ 倒排里没有这个整串 term      ✓ 分成 iphone/手机 两个词去查
keyword 字段 ✓ 值本身不分词，整串比对        ⚠ 同样整串比对，但会诱导你写出分词幻觉
```

三个补充细节：**keyword 大小写敏感**——term 比对的是原样 term，"Active" ≠ "active"，入库前归一化（lowercase normalizer）比查询侧补救干净；**terms 是 OR**，没有 AND 语义，要"同时具备多值"得写多个 term 进 must（或看字段的 array 语义：一个 keyword 数组字段上，多个 term 的 must 表示"数组里同时含这几个值"）；**prefix/wildcard/regexp 是扫 term 字典的慢查询**，`prefix` 尚可（FST 结构支持前缀游走），`*` 开头的 wildcard 与 regexp 在大索引上会打满 CPU，生产上默认被 `search.allow_expensive_queries=false` 直接拦截——**leading wildcard 基本等于宣告这个查询模式需要换设计**（ngram 或 aggregations 字典）。

# 4. Bool 复合查询

```json
{
  "query": {
    "bool": {
      "must": [ ... ],           // AND, 参与评分
      "should": [                // OR, 参与评分
        { "match": { "title": "iPhone" } },
        { "match": { "title": "MacBook" } }
      ],
      "must_not": [ ... ],       // NOT, filter 上下文
      "filter": [ ... ],         // filter 上下文（不评分，缓存）
      "minimum_should_match": 1  // should 至少匹配 N 个
    }
  }
}
```

四个子句的评分归属必须记牢：**must/should 在 query 上下文计分，must_not/filter 在 filter 上下文不计分**。因此 bool 查询的总分 = 各 must 得分之和 + 命中的 should 得分之和（默认 sum，可用 `adjust_pure_negative` 等参数调整）——这带来一个反直觉结论：**多一个 must 条件，不满足它就不命中，满足它的文档得分却上升**。

should 的语义陷阱在于它的角色随邻居变化：**当 bool 里存在任何 must 或 filter 时，should 变成纯加分项（可为 0 命中）**；只有当 bool 里既无 must 又无 filter 时，should 才自动变成"至少命中一个"（隐含 minimum_should_match=1）。这就是上例必须显式写 `minimum_should_match: 1` 的原因——加上 filter 条件后若不写它，"既不含 iPhone 也不含 MacBook"的文档照样会被查出来（只是分低）。把 should 当"必须至少满足其一"用的场景，永远显式声明。

嵌套 bool 可以表达任意复杂的逻辑（SQL 的 AND/OR/NOT 组合），但层级一深评分与可读性都遭殃；经验法则是把"过滤条件沉到最外层 filter 数组、只留一个 bool.should 处理业务分支"，DSL 会立刻清爽。

> **注意**：bool 里子句顺序不影响语义但影响性能——ES 会先执行**最便宜、最具排他性**的条件（bitset 交集的代价由稀疏程度决定）。所以高区分度的 term（如 tenant_id）放 filter 靠前是良好习惯；`must_not` 的代价是"全集减子集"，当被排除的部分很大时应反过来写正向条件。

# 5. 相关性评分

## 5.1 BM25（默认）

```text
score(q, d) = Σ IDF(qi) × TF(qi, d) / (TF(qi, d) + k1 × (1 - b + b × |d| / avgDL))

IDF: 逆文档频率（词越稀有权重越高）
TF: 词频（饱和控制，k1=1.2）
|d|: 文档长度（长度归一化，b=0.75）
```

BM25 是 TF-IDF 的改进版，两个改进点恰好对应两个常见误解：

- **TF 饱和**：词频对得分的贡献是对数式收敛的——出现 1 次和 100 次的差距远小于直觉。这解释了"某篇文档把关键词刷屏了为什么没有碾压全场"（防关键词堆砌是设计目标，k1 越大饱和越慢）。
- **IDF 按分片统计**：查询词的"稀有度"基于**它所在分片的语料**计算，而不是整个索引。所以同一个查询词在不同分片得分可能不一致，副本数据重分配后总分甚至可能轻微漂移——**跨分片的相关性在数学上不是严格可比的**，这是分布式搜索引擎的固有近似，也是"为什么我的排序偶尔怪"的根因之一。

## 5.2 explain API

```json
GET /products/_search
{
  "query": { "match": { "title": "iPhone" } },
  "explain": true
}

// 单文档打分拆解：排序诡异时的手术刀
GET /products/_explain/1
{ "query": { "match": { "title": "iPhone 15 Pro" } } }
```

explain 输出里逐层出现 `sum of` / `product of` / `weight(...)` / `query norm` 等节点，对应上面公式的展开。三个高频"打分误解"用它来验证：

1. **"分数越大越好"是错觉**：_score 没有可比量纲，12.7 分的文档 A 不比 8.2 分的文档 B "好 55%"，只有同一查询内的序有意义；跨查询比较分数毫无意义（所以不要拿分数绝对值做告警阈值）。
2. **字段短反而可能得分低**：长度归一化（norms）惩罚长文档，"iPhone"出现在标题和出现在一万字说明书正文里得分不同——若这个业务上不合理，解法是 `norms: false` 或 function_score 重排，而不是换查询。
3. **多字段命中不等于分数更高**：默认 multi_match(best_fields) 只取最高分字段，description 也命中不加一分；要"处处命中更好"就换 most_fields 或 sum 语义。

业务调权优先用函数而非改公式：`function_score`（field_value_factor 按热度/销量放大）、`boost` 参数（title^3）、`rescore`（粗排后对 top 千条重精排）覆盖 95% 的需求；调 k1/b 属于学术玩具，生产几乎不动。

> **行业实践**：搜索结果"不准"的排查顺序是——先 `_analyze` 看分词是否符预期（一半问题在此），再 `_explain` 看目标文档为什么分低，最后才 function_score 调权。跳过前两步直接堆 boost，只会得到谁也解释不了的排序玄学。

# 6. Search Template

```json
// 保存模板
POST /_scripts/product_search
{
  "script": {
    "lang": "mustache",
    "source": {
      "query": {
        "match": { "{{field}}": "{{value}}" }
      }
    }
  }
}

// 使用模板
GET /products/_search/template
{
  "id": "product_search",
  "params": { "field": "title", "value": "iPhone" }
}
```

Search Template 用 Mustache 把 DSL 变成带参数的字符串模板，价值不在省流量而在**治理**：查询逻辑集中管理（改一次全局生效，DSL 不必散落在各服务代码里）、参数注入天然防拼接错误、前端只传业务参数拿不到查询结构（防注入式滥用 query_string）。配合条件块还能做"参数缺省则不拼该条件"的动态查询：

```json
"filter": [
  {{#category}} { "term": { "category": "{{category}}" } }, {{/category}}
  {{#minPrice}} { "range": { "price": { "gte": {{minPrice}} } } }, {{/minPrice}}
]
```

局限也要说清：模板渲染发生在协调节点、每次请求都执行，复杂模板（大量条件分支 + 循环）的 CPU 开销可观；模板存在集群级别（`.scripts` 索引），版本管理需纳入发布流程。**简单场景够用，逻辑一复杂就回代码里拼 DSL**——模板解决的是"共享与治理"，不解决"编程能力"。

# 7. 搜索参数

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `size` | 每页数量 | 10 |
| `from` | 偏移量 | 0 |
| `_source` | 返回字段 | 全部 |
| `sort` | 排序 | `_score` |
| `timeout` | 超时 | 无 |
| `track_total_hits` | 精确总数 | true |

```json
GET /products/_search
{
  "size": 20,
  "from": 0,
  "_source": ["title", "price"],
  "sort": [
    { "price": "asc" },
    { "_score": "desc" }
  ],
  "timeout": "2s"
}
```

8.x 有一处默认值变化必须知道：**`track_total_hits` 默认不再是"无限精确"**——`true` 的实际含义是精确统计到 10000 条（返回 `10000` + `relation: "gte"`），要全量精确总数得显式 `true` 改 `null`/大数。原因是精确计数要求每分片扫完全部命中，在大索引上可能比取 top 10 还贵；绝大多数 UI 只需要"约 1 万+"。同理 `timeout` 是**软超时**（到期返回当前已收集的 incomplete 结果并标记 `_shards.failed`，不会杀查询），设置它只能防请求吊死，不能防资源浪费——压测里"timeout 500ms 还打出百万 QPS 的 ES 被打挂"并不违反直觉，各分片在超时窗口内照样跑满。

`sort` 一旦指定非 _score 字段，评分整体被跳过（省 BM25 计算，这是"只排序不打分"查询能进 filter 就能飞的原因）；`_source` 裁剪永远比返回全文档再应用层丢弃便宜（省 fetch 阶段的磁盘/网络读）；深分页的 search_after（`sort` 值作游标）与"列表下拉无限加载"场景天然契合——第 12 篇展开。

# 8. 小结

| 主题 | 核心结论 |
|------|---------|
| 两种上下文 | "是不是"进 filter（bitset+缓存，不评分），"像不像"进 query（算 BM25）；term 放进 must 白白丢缓存 |
| match 家族 | 默认 OR 召回宽，收紧靠 operator/minimum_should_match；match_phrase 靠 position 保序；query_string 慎对终端用户开放 |
| term vs match | term 原样比对、match 先分词——keyword+term、text+match 是唯一正确组合，其余象限全是坑 |
| bool | should 在有 must/filter 时只是加分项，必须显式 minimum_should_match；高区分度条件沉入 filter |
| BM25 | TF 饱和防堆词、IDF 按分片统计导致跨分片得分近似；_score 只有序有意义、没有可比量纲 |
| 调权路径 | _analyze → _explain → boost/function_score → 最后才动查询结构；k1/b 不要调 |
| 参数默认值 | track_total_hits 默认截 1 万、timeout 是软超时、非 _score 排序自动免评分 |

DSL 解决"单查询怎么表达"，第 8 篇进入"多维统计怎么表达"——聚合框架：Bucket/Metric/Pipeline 三层结构、doc_values 上的执行机制，以及 terms 聚合在分布式下的近似本质。
