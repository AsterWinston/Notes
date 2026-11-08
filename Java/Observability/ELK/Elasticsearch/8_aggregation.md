# 1. 聚合三大类

聚合（Aggregation）是 Elasticsearch 在"搜索命中列表"之外的第二种答案形态：它不关心具体是哪几篇文档，而关心**分组与统计**——按状态分组的订单数、按天分组的平均延迟、按用户去重的 UV。理解聚合的关键是先建立"三大类"的框架感：**Bucket 聚合负责分组，Metric 聚合负责对每组算一个数，Pipeline 聚合负责对已经算出来的数再算一次**。任何复杂报表都能拆成这三层的组合。

```text
Aggregations
  ├── Bucket (桶聚合)     → 分组
  ├── Metric (指标聚合)    → 统计
  ├── Pipeline (管道聚合)  → 对聚合结果再聚合
  └── Matrix (矩阵聚合)    → 多字段协方差（实验特性）
```

| 类别 | 回答的问题 | 代表算子 | 成本特征 |
|------|-----------|---------|---------|
| Bucket | 分成哪些组 | terms、date_histogram、range、histogram、composite | 桶数量决定内存，深嵌套指数级放大 |
| Metric | 每组算出什么数 | avg、sum、max、stats、percentiles、cardinality | 单值累加器，成本与文档数线性相关 |
| Pipeline | 对结果再做运算 | avg_bucket、derivative、cum_sum、bucket_sort | 只在 reduce 阶段运行，几乎不碰文档 |

日常开发按前三类即可；Matrix（matrix_stats）是 experimental，极少用到。

# 2. Bucket 聚合

Bucket 聚合的本质是**把命中集合按某个维度切成若干子集**，每个子集就是一个"桶"，桶有自己的 doc_count，并且可以携带子聚合。所有 Bucket 聚合的开销都由"桶的数量"决定，设计分组维度时先问"这个维度基数有多大"是第一课。

## 2.1 Terms — 按值分组

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "by_status": {
      "terms": {
        "field": "status",
        "size": 10,               // 返回 top 10
        "order": { "_count": "desc" }
      }
    }
  }
}
```

**terms 聚合基于倒排词典（term dictionary）工作**，而不是逐文档读值：分片把出现过的每个 term 当作一个候选桶，统计其 doc_count，然后把局部 top 桶交给协调节点合并。这带来两个直接后果——它对**低基数枚举字段**（状态、类型、城市）极快，因为词典本身已在内存；它天然**有精度损失**，返回结果里的 `doc_count_error_upper_bound` 字段就是 ES 对"你可能漏了哪个桶、doc_count 可能差多少"给出的上界估计。

`size` 控制返回多少个桶，`order` 支持按 `_count`、`_key` 或子聚合的值排序。当业务需要"没有命中的桶也显示为 0"时，terms 做不到（只返回词典里真实存在的 term），需要"全量遍历桶"时应改用 composite——它像游标一样按排序键一页页返回桶，用上一页的 `after_key` 继续，全局有序、不漏不重，代价是不能按 doc_count 取 top。

> **行业实践**：线上做"热门分类 Top N"报表时，同时检查 `doc_count_error_upper_bound` 与 `sum_other_doc_count`。前者不为 0 且接近第 N 名差距时，说明排名可能失真，加大 `shard_size` 重查是标准处理动作。

## 2.2 Date Histogram — 时间直方图

```json
{
  "aggs": {
    "orders_over_time": {
      "date_histogram": {
        "field": "created_at",
        "fixed_interval": "1d",        // 固定间隔
        "calendar_interval": "month",  // 日历间隔
        "min_doc_count": 1,
        "extended_bounds": {
          "min": "2024-01-01",
          "max": "2024-01-31"
        }
      }
    }
  }
}
```

date_histogram 是时序分析的主力桶，机制上先按 interval 把时间轴切成连续的桶，再把每个文档的日期值"向下取整"放进对应桶。**fixed_interval 与 calendar_interval 的区别在跨夏令时和月末场景下才显现**：前者严格按 86400 秒切一格，后者按日历语义（"1 个月"在 2 月和 7 月长度不同），做报表统计几乎总是应选 calendar。时区通过 `time_zone` 指定，否则 UTC 会把"按天"的边界和北京时间错开 8 小时——这是时间报表最常见的坑。空桶默认不返回（`min_doc_count: 1`），配合 `extended_bounds` 可以把缺失的日期补出来，否则"断点"在折线图上会被误读为业务下滑而非数据缺失。

## 2.3 Range — 范围分桶

```json
{
  "aggs": {
    "price_ranges": {
      "range": {
        "field": "amount",
        "ranges": [
          { "to": 100 },
          { "from": 100, "to": 500 },
          { "from": 500, "to": 1000 },
          { "from": 1000 }
        ]
      }
    }
  }
}
```

range 对数值/日期字段做区间分组，边界是 **`from` 含、`to` 不含的闭开区间**，所以 `to:100` 与 `from:100` 恰好无缝衔接。与 histogram 的等距切分不同，range 的分桶边界完全由请求决定，因此**每个桶的判定彼此独立**——一个文档可能同时落入多个重叠区间，这既是灵活性，也是统计口径出错的高发点。

> **注意**：ranges 区间重叠是特性不是 bug，做"价格带分布"时务必让区间互斥；做"风险等级命中数"（一条记录可命中多条规则）时反而要故意利用重叠。写监控报表前先明确口径属于哪一类。

## 2.4 Histogram — 数值直方图

```json
{
  "aggs": {
    "price_histogram": {
      "histogram": {
        "field": "amount",
        "interval": 100
      }
    }
  }
}
```

histogram 按固定步长把数值轴切成等距桶，桶键为 `floor(value / interval) * interval`，可用 `offset` 平移边界。它和 date_histogram 是同一套实现，只是把"时间"换成"任意数值"。等距桶适合**分布形态分析**（延迟直方图、金额分布），对业务口径分档（9.9/99/199 价格带）既别扭又浪费桶，后者一律用 range。

# 3. Metric 聚合

Metric 聚合是桶内的"累加器"：对每个文档从 doc_values 里取出字段值，喂给一个数学模型，输出统计量。**大部分 metric 聚合的成本只与参与文档数有关、与桶数无关**，所以 avg/sum/min/max 可以非常便宜地挂在深层 bucket 下；真正要警惕的是 cardinality（内存）和 percentiles（维护值分布）这两种"重模型"。

```json
{
  "aggs": {
    "avg_price": { "avg": { "field": "amount" } },
    "max_price": { "max": { "field": "amount" } },
    "min_price": { "min": { "field": "amount" } },
    "sum_amount": { "sum": { "field": "amount" } },
    "stats": { "stats": { "field": "amount" } },
    "extended_stats": { "extended_stats": { "field": "amount" } },
    "cardinality": {                // 近似去重计数
      "cardinality": {
        "field": "user_id",
        "precision_threshold": 40000
      }
    },
    "percentiles": {                // 百分位
      "percentiles": {
        "field": "amount",
        "percents": [50, 95, 99]
      }
    }
  }
}
```

`stats` 一次返回 count/min/max/avg/sum,`extended_stats` 再加方差、标准差等——两者都比写五个独立 metric 聚合便宜，因为字段的列值只读一遍。

## 3.1 cardinality 与 HyperLogLog++

**cardinality 是近似的，这是设计决定而非实现缺陷**：精确去重要把所有值装进 Set，对 user_id 这种亿级字段内存直接爆炸；HyperLogLog++ 用概率数据结构把"去重计数"压成固定大小的寄存器数组，一亿 distinct 值也只用几十 KB。代价是结果有误差，误差率与 `precision_threshold` 相关：默认 3000、最大 40000，低于阈值时结果通常等同精确值，超过阈值后误差一般在 1%~2% 量级。

**设计考量**：

- **报表用 UV** 直接用 cardinality，误差对趋势分析几乎无影响；
- **对账/计费场景**不能用 cardinality，应走精确方案（数据库 distinct 或离线计算）；
- "分组各自去重且组数很多"时，每个桶都持有一份 HLL 结构，内存 = 桶数 × 单个结构大小，这是很多"聚合 OOM"的隐形来源。

percentiles 同理是近似算法（T-Digest，可选 HDR），它保留值分布的压缩表示而非原始值；`"tdigest": { "compression": 200 }` 可提高 P99 精度，但桶内存随之增长——监控大盘"P99 反复横跳"往往就是压缩率太低的信号。

# 4. 嵌套聚合

聚合树可以任意嵌套：外层桶切分数据，内层桶继续细分，叶子挂 metric，这就是 OLAP 里的 drill-down。

```json
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": { "field": "category" },
      "aggs": {                                     // 子聚合
        "avg_price": { "avg": { "field": "price" } },
        "top_products": {                            // 顶级命中
          "top_hits": {
            "size": 3,
            "_source": ["title", "price"],
            "sort": [{ "price": "desc" }]
          }
        }
      }
    }
  }
}
```

**成本模型是乘法而不是加法**：第一层 100 个桶 × 第二层 100 个桶 = 1 万个桶对象，每个桶都挂着累加器与排序状态；协调节点还要在 reduce 阶段把所有分片的桶树合并、按序截取，深嵌套大基数的请求是内存熔断（`circuit_breaking_exception`）的重灾区。`bucket_sort`、外层先 `filter` 收窄命中集，是控制这棵树的常用手段。

`top_hits` 是特殊叶子聚合：它在每个桶里再执行一次迷你搜索并返回**完整命中文档**，等于把每个桶的候选集缓存在内存里——桶多、size 大、文档肥三者叠加时开销极其可观，务必限制 `size` 并用 `_source` 过滤字段。另外 top_hits 的评分是分片内的，跨分片排序要靠外层 query 保证。

> **注意**：对对象数组（如订单里的商品行）做聚合，默认 `object` mapping 会把嵌套字段拉平，导致"商品 A 的价格 × 商品 B 的名称"这类错误组合；必须把字段定义为 `nested` 类型并用 `nested` 聚合进入子文档空间。nested 会为每个子文档建立独立的隐藏 Lucene 文档，父文档与子文档计数分离，这也是 nested 写入放大的根源。

# 5. Pipeline 聚合

Pipeline 聚合不读文档，它的输入是**其他聚合的输出**——在分片计算完成、协调节点 reduce 之后对这些结果做二次运算，回答"趋势的导数、桶间的平均、累计和"这类问题。

```json
{
  "aggs": {
    "orders_per_month": {
      "date_histogram": {
        "field": "created_at",
        "calendar_interval": "month"
      },
      "aggs": {
        "total": { "sum": { "field": "amount" } }
      }
    },
    "avg_monthly_revenue": {          // Pipeline: 基于父聚合
      "avg_bucket": {
        "buckets_path": "orders_per_month>total"
      }
    },
    "revenue_derivative": {           // Pipeline: 环比增长
      "derivative": {
        "buckets_path": "orders_per_month>total"
      }
    }
  }
}
```

`buckets_path` 是 Pipeline 聚合的"指针语法"：`>` 表示下钻一层，`<` 表示回到父级，`_count`/`_key` 是桶的内建属性。常用家族：`cum_sum`/`moving_fn`（累计与滑动窗口）、`derivative`（增长量）、`bucket_selector`/`bucket_sort`（按聚合值过滤和重排桶——**SQL 里 having 的语义，ES 只能在查询端用 bucket_selector 表达**）。

> **注意**：管道聚合运行在 reduce 阶段，意味着它拿到的桶可能已经过 terms 的 top-N 截断——对"缺失的桶"求平均会算错。需要完整桶集时把上游换成 composite 或调大 size，这是环比/同比报表对不上数的经典原因。

# 6. 聚合执行原理

```text
聚合执行流程:
  1. 协调节点解析聚合请求
  2. 分发到各分片并行计算
  3. 各分片返回局部聚合结果
  4. 协调节点合并返回最终结果

注意:
  - terms 聚合可能有误差（每个分片返回 top N）
  - cardinality 是近似值（HyperLogLog++ 算法）
  - 聚合在 doc_values 上执行（列式存储）
```

逐段拆开看。**第 2 步里每个分片在自己的数据上独立构建聚合树**：Bucket 聚合逐文档把值路由进对应桶并触发子聚合，Metric 聚合在桶内累加——分片之间全程不通信，这是聚合能水平扩展的根本原因。**第 4 步的 reduce 才是精度与内存的分水岭**：协调节点把所有分片的局部结果按桶合并，terms 只保留了"各分片自己的 top N"参与合并，全局第 N+1 名的桶如果在任何分片都没进局部 top，它就永久消失了。

## 6.1 terms 的误差与 shard_size

为了降低截断误差，terms 每个分片实际收集 `shard_size` 个桶再截断给协调节点，默认约 `size × 1.5 + 10`。**加大 shard_size 能提高 top-N 的准确率，但只是概率上的改善——terms 聚合永远无法保证严格的全局 top N**（除非数据都在一个分片上）。误差上界 `doc_count_error_upper_bound` 的含义是：某个桶在没进局部 top 的分片里最多可能还藏着多少文档；它只在按 `_count` 排序时有意义，按子聚合排序时这个值并不反映"排名错误"的风险。

## 6.2 doc_values：聚合为什么快

**聚合和排序不读倒排索引，而是读 doc_values——按字段组织的列式正排存储**。写入时字段值按文档号顺序追加进列式结构，聚合器按列扫描、CPU 缓存友好；倒排擅长的是"给定 term 找文档"，两者互补。`doc_values: false` 能省磁盘，但让该字段失去排序/聚合能力。

keyword 字段的聚合依赖 **global ordinals**：分片首次聚合时把各 segment 的局部词典合并成全局有序编号表，后续桶统计就是数组下标访问。代价是 segment 变化后 ordinals 需要重建——高频写入 + 高基数 keyword 字段上跑 terms 聚合，表现为"聚合偶尔慢一大截"。

# 7. 聚合优化

```json
# 1. 限制桶数量
"terms": { "field": "status", "size": 10, "shard_size": 100 }

# 2. 使用 filter 缩小范围
{
  "query": { "range": { "created_at": { "gte": "2024-01-01" } } },
  "aggs": { ... }
}

# 3. execution_hint
"terms": { "field": "status", "execution_hint": "map" }
# map: 小 cardinality;global_ordinals: 大 cardinality（默认）
```

四条优化路径按优先级排列：

1. **先收窄命中集**：query 阶段的过滤决定聚合的输入行数，range/term 过滤放 query 而不是 filters 聚合里；
2. **控制桶的数量与形状**：terms 用 size + shard_size 组合，全量遍历用 composite，空桶交给 min_doc_count；
3. **选对执行策略**：`execution_hint: map` 跳过 global ordinals 构建，适合基数小（状态、布尔）且 segment 频繁变动的字段；默认 `global_ordinals` 适合高基数；`segment_ordinals` 适合 nested 聚合；
4. **能提前终止就提前终止**：`terminate_after` 限制单分片扫描文档数（近似结果，谨慎用于报表）。

> **注意**：filters 聚合与 query 里的 filter 语义不同——前者不作用于 query，只是桶间的分流，写"带过滤的聚合"时先分清要过滤输入还是过滤分组结果。

# 8. 小结

| 主题 | 核心结论 |
|------|---------|
| 三大类分工 | Bucket 分组、Metric 组内统计、Pipeline 对结果再运算；成本由"桶数 × 深度"决定 |
| terms 误差 | 分片各自取局部 top N,`doc_count_error_upper_bound` 给出上界；严格榜单用 composite |
| cardinality | HyperLogLog++,precision_threshold 上限 40000；对账场景禁用 |
| doc_values | 聚合走列式正排而非倒排；keyword 依赖 global ordinals，重建成本藏在"聚合偶发变慢"里 |
| top_hits | 每桶缓存一份完整命中，桶多+size 大+文档肥三者叠加必爆内存 |
| Pipeline | reduce 阶段运行，上游桶被截断时结果会静默错误 |

聚合这一篇偏"单请求怎么用"，下一篇（9_analysis）进入文本分析层——分词器决定了"内容能不能被搜到"，而聚合决定了"搜到之后怎么算"。
