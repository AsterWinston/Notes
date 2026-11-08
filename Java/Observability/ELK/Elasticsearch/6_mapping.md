# 1. 字段类型总览

Mapping 是索引的 schema，它声明的不只是"字段是什么类型"，而是**这个字段会被构造成哪几种物理结构、从而支持哪些操作**。同一个值写成 `text` 还是 `keyword`，决定它能被分词搜索还是聚合排序——mapping 设计是 ES 使用中回报最高的决策。

| 类型 | 说明 | 示例 |
|------|------|------|
| `text` | 全文检索字段（分词） | 商品描述 |
| `keyword` | 精确值字段（不分词） | 订单号、状态 |
| `long/float/double` | 数值 | 金额、数量 |
| `date` | 日期 | 创建时间 |
| `boolean` | 布尔 | 是否删除 |
| `object` | 嵌套 JSON 对象 | 地址 |
| `nested` | 独立索引的数组对象 | 订单明细 |
| `join` | 父子关联 | 订单+订单项 |
| `geo_point` | 地理坐标 | 经纬度 |
| `ip` | IP 地址 | 客户端 IP |
| `binary` | Base64 编码的二进制 | 文件内容 |
| `range` | 范围类型 | 价格区间 |

类型可以按"查询方式"归成四族，选型时先问自己这个字段服务于哪一族：

- **要"搜"的**：text（分词后建倒排，match 系列查询命中）——人写的自然语言内容。
- **要"过滤/统计"的**：keyword、数值、date、boolean、ip（值整体作为一个 term 建倒排，同时有 doc_values 支持排序聚合）——机器生成的编码、枚举、度量。
- **要"表达结构"的**：object、nested、join——文档内的一对多关系，第 4 节展开。
- **要"表达空间/区间"的**：geo_*、range——特殊编码换取专用查询算子。

一个高频误区是把数值字段全部用 text/keyword 存："手机号是字符串"的判断没错，但**金额、数量、评分一旦存成 keyword，range 查询与聚合就全废了**（keyword 的 range 按字典序比较，"9" > "10"）。date 同理：8.x 默认按 epoch_millis 解析，格式五花八门的日志要么统一 `format` 声明多种 pattern，要么在 ingest pipeline 里先归一化。

> **注意**：类型选错无法就地修改。已写入数据的字段改类型 = reindex 重建（第 4 篇零停机流程）；只有新增字段可以随时追加。动手写 mapping 之前，把每个字段将来要执行的查询（match？term？range？sort？aggs？）过一遍，比事后重建便宜两个数量级。

# 2. Dynamic Mapping

不显式定义 mapping 时，ES 根据 JSON 值自动推断字段类型：

```json
POST /test/_doc
{
  "name": "Alice",          // → text + keyword 子字段
  "age": 25,                 // → long
  "active": true,            // → boolean
  "created": "2024-01-01",  // → date
  "amount": 99.9             // → float
}
```

推断规则本身合理（string → text+keyword 子字段、整数 → long、浮点 → double、匹配 date 格式 → date），**风险在于它把 schema 的决定权交给了第一批到达的数据**。典型事故链：日志字段 `user_id` 先来一条数字 → 推断为 long；后来一条 `"U1001"` → 写入失败（无法转 long），或者更糟——先字符串后数字，字符串类型锁死后数字全部被 coerce 成字符串，聚合结果诡异。另一个经典：`amount` 首次出现是整数 `100` → long，后续 `99.9` 被截断成 `99` 或按类型报错，取决于 coerce 设置。

因此 dynamic mapping 的分级策略是：

```json
// 三档开关（索引级）
{ "mappings": { "dynamic": "true" } }     // 默认：自动推断并添加到 mapping
{ "mappings": { "dynamic": "strict" } }   // 未定义字段直接报错——业务核心索引推荐
{ "mappings": { "dynamic": false } }      // 未定义字段仍存入 _source 但不索引——日志兜底
```

**设计考量**：

- **业务文档索引（订单、商品）用 `strict`**：字段名打错（`quentity`）应当立刻报错暴露，而不是悄悄生成一个永远不会被查询的孤儿字段，多年后 mapping 里躺着几百个这种字段。
- **schema 未知的日志/埋点用 `false` 或 dynamic_templates**：保证可写入、核心字段可查，多余字段留在 _source 里备查。
- **default_dynamic_template 兜底**：8.x 内置规则会把字符串推断成 text+keyword 子字段，注意这正是"聚合慢"的伏笔——keyword 子字段有 `ignore_above: 256` 默认截断，超长的 JSON 串、base64 不参与聚合。

## 2.1 Dynamic Template

dynamic template 在"推断"这一步插入自定义规则，把猜测变成约定：

```json
PUT /orders
{
  "mappings": {
    "dynamic_templates": [
      {
        "strings_as_keyword": {
          "match_mapping_type": "string",
          "mapping": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      {
        "integers_as_integer": {
          "match_mapping_type": "long",
          "mapping": { "type": "integer" }
        }
      }
    ]
  }
}
```

模板按声明顺序匹配第一个命中的规则；`match_mapping_type` 匹配的是**推断出的类型**（long/string/double/date/boolean/object），还支持 `match`/`unmatch`（字段名 glob）与 `path_match`（全路径）做条件。两条高频规则：**字符串一律 keyword**（日志类索引不需要分词，省掉 text+keyword 双份索引，字段数与存储直接砍半——keyword 上的 `terms` 查询照样能做全文包含之外的绝大多数过滤）；**long 降级 integer**（省一半数值存储，订单号/时间戳/雪花 ID 这类可能超 2^31 的字段必须排除，否则写入直接报 out of range）。

> **行业实践**：日志集群的 mapping 治理三板斧——`dynamic: false`（或全量 ECS 模板）+ strings_as_keyword + 字段生命周期审计。Elastic Common Schema（ECS）预定义了字段命名（`host.name`、`event.duration`），接入 Beats/Agent 生态时按 ECS 定义 mapping，比自建 schema 省力得多。

# 3. 关键字段参数

类型之下还有一组开关，控制"这个字段到底建哪几种物理结构"：

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `index` | 是否索引（可搜索） | true |
| `doc_values` | 是否支持排序/聚合 | true |
| `norms` | 是否存储评分因子 | text: true, keyword: false |
| `store` | 是否单独存储 | false |
| `ignore_above` | keyword 超过此长度不索引 | 2147483647 |
| `null_value` | NULL 替代值 | null |
| `copy_to` | 复制到其他字段 | — |
| `coerce` | 类型强制转换 | true |

```json
{
  "email": {
    "type": "keyword",
    "ignore_above": 256     // 超过 256 字符不索引
  },
  "title": {
    "type": "text",
    "norms": false          // 不需要评分时关闭，节省空间
  },
  "remark": {
    "type": "text",
    "copy_to": "full_text"   // 多字段汇入统一搜索字段
  }
}
```

逐个说透取舍：

- **index vs doc_values**：`index: false` = 不进倒排、不能被搜索，但还能排序聚合（省空间，适合"只展示不参与检索"的长文本）；`doc_values: false` = 不能排序聚合脚本，但还能搜（适合 _source 已另有用途的大字段）。两者都关的字段等于纯存储在 _source 里。
- **norms**：BM25 评分需要"字段长度归一化因子"，每个 text 字段每段都要存它。日志的 message 字段若只做关键词过滤、从不按相关性排序，`norms: false` 可省下可观空间且写入更快。
- **ignore_above**：keyword 值超过阈值则**不进倒排**（doc_values 仍写）——防一个超长 URL/base64 把 term 字典撑爆。副作用常被忽略：被截断的值 `term` 查不到，但 `_source` 里原样在，肉眼"明明有这条"查询却 miss。
- **coerce**：true 时 `"123"` 自动转数字、`3.14` 转 long 截断成 3；false 时类型不符直接报错。**宁可报错也不要悄悄改值**——金融数据建议显式关 coerce。
- **null_value**：JSON 的 null 无法被搜索（等于字段缺失）；声明 `null_value: "unknown"` 后 term 查询可以命中这个替身值。

> **注意**：`store` 在现代 ES 里几乎是"永远不要打开"的参数——_source 本身就是存储字段原文的，再单独 store 一份纯属重复。只有"关闭 _source 的极端省空间场景"才轮到它，而那种场景下 partial update、高亮、reindex 全都会坏掉，得不偿失。

# 4. text vs keyword：multi-field

同一个字段需要"既能全文搜、又能精确过滤/聚合"时，用 **fields 多字段映射**，一次写入派生多种索引结构：

```json
PUT /products
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "fields": {
          "keyword": { "type": "keyword", "ignore_above": 256 }
        }
      }
    }
  }
}

// 搜索用 text
GET /products/_search
{ "query": { "match": { "title": "iPhone" } } }

// 聚合/排序用子字段（注意 .keyword 后缀，写错前缀聚合直接报错）
GET /products/_search
{ "size": 0, "aggs": { "titles": { "terms": { "field": "title.keyword" } } } }
```

这是 ES 使用频率最高的"为什么"之一：**`text` 字段不能直接聚合排序**——它存的是分词后的 term，"iPhone 15 Pro" 已经被拆成三个词，聚合拿不到完整值；而 keyword 保留原值整体作为一个 term。`title` 与 `title.keyword` 是**同一原文的两套索引**，写一份、查两套，代价是索引体积膨胀，所以 dynamic mapping 对字符串默认就生成这套组合，业务 mapping 里要主动决定谁配这个双份。

中文场景在 text 之上还有分析器一层（IK 分词器、`ik_max_word` vs `ik_smart`），本篇不展开，见第 9 篇分析器；但 keyword 上的中文不受分词影响，"电子产品"整体匹配，这正是 category 类字段用 keyword 的原因。

# 5. Nested 类型

object 数组的处理方式是**扁平化**——所有子字段合并进父文档的同一批 term，导致跨元素串味。看 nested 的定义：

```json
PUT /orders
{
  "mappings": {
    "properties": {
      "items": {
        "type": "nested",    // 每个 item 独立索引
        "properties": {
          "name": { "type": "keyword" },
          "quantity": { "type": "integer" }
        }
      }
    }
  }
}

// 查询必须用 nested query
GET /orders/_search
{
  "query": {
    "nested": {
      "path": "items",
      "query": {
        "bool": {
          "must": [
            { "term": { "items.name": "iPhone" } },
            { "range": { "items.quantity": { "gt": 1 } } }
          ]
        }
      }
    }
  }
}
```

**Nested vs Object**：`object` 会把数组扁平化导致跨对象匹配；`nested` 每个元素独立文档。用一个例子钉死差异：文档 `items: [{name:"iPhone", quantity:2}, {name:"Case", quantity:5}]`，object 存储后 term 字典里 `name` 有 iPhone/Case、`quantity` 有 2/5，查询"iPhone 且数量>4"会**错误命中**（两个条件分别被不同元素满足）；nested 把每个元素存成隐藏的子文档，条件必须在同一子文档内同时成立。

代价也必须摆在桌面：nested 元素是**真实独立的 Lucene 文档**——100 个 items 的父文档 = 101 个文档，任何一次更新父文档都要**重写全部子文档**（无法单独改一个 item）；nested 数量有索引级上限（`nested_fields.limit`）。选型口诀：**需要"同一元素内多条件关联"才上 nested；只存不查的明细直接塞 _source；一对多且各自有生命周期，考虑拆成父子文档（join）或干脆两个索引应用层关联**。

# 6. Runtime Field

```json
PUT /orders/_mapping
{
  "runtime": {
    "day_of_week": {
      "type": "keyword",
      "script": {
        "source": "emit(doc['created_at'].value.dayOfWeek())"
      }
    }
  }
}
```

Runtime 字段（8.x 成熟）反转了"先索引后查询"的次序：不写入任何索引结构，查询时读 _source 现算。

**优势**：无需重建索引即可添加字段，节省存储，mapping 变更零成本；**代价**：每个匹配文档都要执行脚本，**查询性能低于索引字段一到两个数量级**，文档量大时 CPU 会成为新瓶颈。**设计考量**：探索阶段（schema 还没想清楚）用 runtime 字段验证需求；验证出来的高频字段沉淀为索引字段；对已有索引"加一个低频过滤字段"这种改造收益比不上的场景，runtime 是正解——比如安全场景里从原始 log 现场解析出的临时 IOC 字段。

# 7. Mapping Explosion 防护

dynamic mapping + 高基数键值对（把 Map 直接序列化进文档）会让字段数指数膨胀，集群状态里每个字段的 term 字典与元数据都占 master 内存与每分片开销——这就是 mapping explosion，历史上大量"集群状态发布慢""fielddata 熔断"事故的源头：

```yaml
# elasticsearch.yml
indices.mapping.total_fields.limit: 1000       # 索引最大字段数
indices.mapping.depth.limit: 20                # 最大嵌套深度
indices.mapping.nested_fields.limit: 50        # 最大 nested 字段数
indices.mapping.nested_objects.limit: 10000    # 单个文档最大 nested 对象数
```

反模式与解法对照：

| 反模式 | 后果 | 解法 |
|--------|------|------|
| 把用户自定义标签 `{"tag_红": true, ...}` 平铺进文档 | 字段数随标签组合爆炸 | 收敛成 `tags: [keyword]` 数组 |
| 用 metric 名当字段名（`cpu_usage_host_001`） | 每主机一字段，索引即炸 | 改用文档行 + hostname keyword 维度 |
| 通配符 dynamic: true 全开 | 拼写错误字段永久残留 | strict/false + dynamic template |
| 超限后临时调大 limit | 把爆炸推迟并放大 | 修 schema，limit 只做保险丝 |

> **注意**：`total_fields.limit` 超限后写入报 `too many dynamic clauses`，此时既不能删字段（mapping 只增不减，除非 reindex），调大限制又只是饮鸩止渴。正确处置顺序：先止血（把新数据按上述方案收敛格式），再重建。监控上把 `GET /_mapping` 的字段数纳入巡检，比等报错体面得多。

与 mapping 体量相关的最后一个陷阱是 **fielddata**：对 text 字段做排序/聚合时，ES 会把倒排索引**反转加载进 JVM 堆**（fielddata cache），一个大 text 字段能瞬间打爆堆并触发熔断（`CircuitBreakingException`）。8.x 的正确姿势不是去调 `fielddata.cache_size`，而是**永远不要对 text 直接聚合**——改用 `.keyword` 子字段。"fielddata"三个字如今只应出现在报错信息里，作为历史教训记住即可。

# 8. 小结

| 主题 | 核心结论 |
|------|---------|
| 类型四族 | 搜（text）、滤/聚（keyword/数值/date）、结构（object/nested/join）、空间区间（geo/range），按查询方式选型 |
| Dynamic mapping | 业务索引 strict、日志索引 false 或模板收敛；首批数据决定类型的机制本身就是风险 |
| 核心开关 | index/doc_values 按用途裁剪；norms 不需评分就关；ignore_above 防爆但有截断盲区；store 基本别碰 |
| text vs keyword | text 分词后值已碎不能聚合排序，keyword 整体保留不能全文；.keyword 子字段一份原文两套索引 |
| nested | 独立子文档保证"元素内关联"，代价是全量重写；非必要不上 |
| runtime 字段 | 免重建加字段、省空间，查询性能低 1–2 个数量级，验证期利器 |
| mapping 爆炸 | 高基数键平铺即爆炸；limit 是保险丝不是方案，收敛格式 + 字段数巡检 |

mapping 定了"数据长什么样"，第 7 篇讲"怎么问"——Query DSL：match 家族与 term 的分工、bool 组合、query/filter 上下文的评分与缓存差异，以及 BM25 打分里最常见的几个误解。
