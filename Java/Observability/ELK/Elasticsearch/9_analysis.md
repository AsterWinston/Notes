# 1. Analyzer 三组件

**分析器（Analyzer）是文本与倒排索引之间的翻译层**：文档写入时，`text` 字段的原始字符串要先经过分析器切成一个个词项（Term），才能进入倒排索引；搜索时，查询串也要走一遍（可能不同的）分析器，双方词项在同一个"词项空间"里相遇才可能命中。**分词不一致是"明明有这条数据却搜不到"的头号原因**，所以理解分析器的内部结构比背配置更重要。

```text
Analyzer (分析器)
  ├── CharFilter (字符过滤器)  → 预处理字符(HTML 标签去除、表情符号替换)
  ├── Tokenizer (分词器)      → 将文本切分成词项(Term)
  └── TokenFilter (词项过滤器) → 词项转换(小写、停用词、同义词)
```

三段流水线严格按顺序执行：**先 CharFilter（可以是多个）改写原始字符流，再 Tokenizer 切成词项流（一个分析器有且只有一个），最后 TokenFilter（多个）逐个改写词项**。CharFilter 作用于切分前，所以它能改"字形"；TokenFilter 作用于切分后，只能改"词形"，不能把两个词合成一个再切开（同义词/多词项过滤器是特例，它们能发出多个位置重叠的词项）。

```json
POST /_analyze
{
  "analyzer": "standard",
  "text": "iPhone 15 Pro is awesome!"
}

# 返回:
{ "tokens": [
  { "token": "iphone", "position": 0 },
  { "token": "15",     "position": 1 },
  { "token": "pro",    "position": 2 },
  { "token": "is",     "position": 3 },
  { "token": "awesome","position": 4 }
]}
```

`_analyze` API 是分析器世界的调试器：token 是最终词项，position 记录词序（match_phrase 靠它判断相邻性），position_length/start_offset/end_offset 在返回里能进一步揭示同义词展开与字符位置。**排查询问顺序永远是：先 _analyze 看索引侧切出什么，再 _analyze 看搜索侧切出什么，两边对不上就是分词问题，与评分、索引数据都无关。**

# 2. 内置分析器

| 分析器 | 说明 | 适用场景 |
|--------|------|---------|
| `standard` | 默认，Unicode 文本分割算法（UAX#29），按词边界切分并转小写 | 通用英文/混排 |
| `simple` | 非字母字符切分，转小写 | 简单英文，数字连写会被拆开（如 A1 → a、1） |
| `whitespace` | 仅按空格切分，不转小写 | 标签串、ID 串 |
| `keyword` | 不分词，整体作为词项 | 精确匹配、枚举 |
| `stop` | standard + 内置停用词表 | 英文检索降噪 |
| `pattern` | 正则分词 | 自定义分隔符（如日志按 `,\s+` 切） |
| `ik_smart` | IK 中文最简分词 | 中文搜索 |
| `ik_max_word` | IK 中文最大分词 | 中文索引 |
| `pinyin` | 汉字转拼音 | 拼音搜索、首字母搜索 |
| `ngram`/`edge_ngram`（tokenizer） | 滑窗切碎 | 搜索建议、模糊匹配 |

两个容易误判的点：其一，**standard 分析器对中文是"单字切分"**——它按 Unicode 词边界工作，而中日韩文本没有空格边界，结果是"中华人民共和国"切成逐字 term。单字索引能"凑巧"支持任意子串查询，但完全丧失词义单元：搜"中国"能命中，搜"华人"也能命中（跨词边界的假匹配），短语和排序质量都很差——这就是中文必须装 IK 等分词插件的根本原因。其二，**keyword 与 text 的分野决定字段能力**：keyword 不切词，所以精确 term 查询、聚合、排序都行，但无法全文搜索；两者常以 `fields` 多字段形式共存（keyword + text 双形态）。

# 3. IK 中文分词

IK（infinilabs/analysis-ik）是中文场景事实上的标准插件，安装必须**集群每个节点都装且版本严格一致**，滚动重启后生效：

```bash
# 安装 IK 插件
elasticsearch-plugin install \
  https://github.com/infinilabs/analysis-ik/releases/download/v8.11.0/elasticsearch-analysis-ik-8.11.0.zip

# 验证:每个节点都应能看到 analysis-ik
GET /_cat/plugins?v
```

```json
# ik_smart — 粗粒度
POST /_analyze
{
  "analyzer": "ik_smart",
  "text": "中华人民共和国国歌"
}
// ["中华人民共和国", "国歌"]

# ik_max_word — 细粒度
POST /_analyze
{
  "analyzer": "ik_max_word",
  "text": "中华人民共和国国歌"
}
// ["中华人民共和国", "中华人民", "中华", "华人", "人民共和国", "人民", "共和", "国歌"]
```

**两种模式的取舍就是召回率与准确率的取舍**：ik_max_word 把所有可能的切分组合都发出来，索引里词项多、倒排链长（写入慢、占空间），但用户无论输入"中华""人民"还是"共和国"都能命中；ik_smart 做最粗切分，词项少、索引干净，但索引了"中华人民共和国国歌"整体后，搜"华人"就命不中。中文搜索引擎的经典组合是**索引用 max_word、搜索用 smart**（§6 展开）。

IK 内置词典之外还支持自定义三类词库，更新时 IK 会做增量热加载：

```bash
# config/analysis-ik/custom.dic —— 主扩展词典(需重启)
iPhone 15 Pro
MacBook Pro
乔布斯

# remote ext dict
# IKAnalyzer.cfg.xml
<entry key="remote_ext_dict">http://my-server/dict.txt</entry>
```

```xml
<!-- IKAnalyzer.cfg.xml 关键配置 -->
<properties>
    <entry key="ext_dict">custom.dic</entry>                       <!-- 本地扩展词,重启生效 -->
    <entry key="remote_ext_dict">http://dict-svc/new_word.txt</entry> <!-- 远程热更新,轮询 Last-Modified/ETag -->
    <entry key="ext_stopwords">stopword.dic</entry>                 <!-- 自定义停用词 -->
</properties>
```

> **行业实践**：远程词典要求 HTTP 响应带 `Last-Modified` 或 `ETag` 头，IK 节点每 60 秒轮询，内容变化才拉取——所以"新词秒级生效"是错觉，规划热词系统（如电商新品名当天可搜）要把这个轮询周期和"该词只影响新文档、存量需 reindex"这两件事一起算进排期。

**设计考量**：

- 词典放 NFS/共享盘会被各节点缓存不一致坑，推荐独立词典服务 + 版本化 URL；
- 扩展词命中后**旧文档不会自动重新分词**，补数据只能 reindex 或 `_update_by_query`；
- 分词粒度争议（如"西安市"切出"西安市/西安/市"）优先通过自定义词 + 停用词消歧，而不是换分词器。

# 4. 自定义分析器

内置分析器覆盖不了时，就手工组装三个组件。自定义分析器定义在 `settings.analysis` 下，先声明零件（char_filter/tokenizer/filter），再用 `type: custom` 的 analyzer 把它们串起来：

```json
PUT /shop
{
  "settings": {
    "analysis": {
      "char_filter": {
        "my_char_filter": {
          "type": "mapping",
          "mappings": ["＆ => &", "￥ => ¥"]
        }
      },
      "tokenizer": {
        "my_tokenizer": {
          "type": "pattern",
          "pattern": "[,;\\s]+"
        }
      },
      "filter": {
        "my_stop": {
          "type": "stop",
          "stopwords": ["的", "了", "是"]
        }
      },
      "analyzer": {
        "my_analyzer": {
          "type": "custom",
          "char_filter": ["html_strip", "my_char_filter"],
          "tokenizer": "ik_max_word",
          "filter": ["lowercase", "my_stop"]
        }
      }
    }
  }
}
```

这个例子里 `my_analyzer` 同时用了内置零件（`html_strip`、`lowercase`）和自定义零件，这正是自定义分析器的常见形态——**大多数需求不是从零发明分词器，而是给现成分词器加过滤链**。常用 filter 家族：`lowercase`/`uppercase`（大小写归一）、`stop`（停用词）、`synonym`/`synonym_graph`（同义词）、`stemmer`（词干化，running→run）、`phonetic`（音标近似）、`trim`/`keyword_marker`（保留指定词不切）、`word_delimiter_graph`（按驼峰/连字符拆词，如 JSON 类技术名）。

**索引设置不可在开启的索引上修改**：改 analysis 必须先 close 索引、改完 open（或 reindex 到新索引），而分析器变更意味着存量倒排全部作废，close/open 只改"以后"不救"过去"——生产上普遍用"别名 + 新索引 reindex"完成分词器升级。

# 5. 同义词

同义词让"iPhone"与"苹果手机"互相可达，机制上是在 TokenFilter 阶段把输入词替换/扩展成多个重叠词项：

```json
PUT /shop
{
  "settings": {
    "analysis": {
      "filter": {
        "synonym_filter": {
          "type": "synonym",
          "synonyms": [
            "iPhone,苹果手机",
            "MacBook,苹果笔记本",
            "js, javascript"
          ],
          "synonyms_path": "analysis/synonyms.txt"
        }
      },
      "analyzer": {
        "synonym_analyzer": {
          "tokenizer": "ik_max_word",
          "filter": ["synonym_filter"]
        }
      }
    }
  }
}
```

同义词语法有两种方向：`a, b, c` 是**等价组**（双向互通），`a, b => c` 是**显式映射**（搜 a、b 都改写为 c,c 不会反向命中 a——适合"别名归一"）。Solr 格式（`s, => syn`）也支持。**多词同义词（"智能电视，tv"）会把词项放在同一 position 上展开**，因此对 phrase 查询与 highlight 的行为会变复杂，8.x 建议一律使用 `synonym_graph` filter（处理多词更正确）。

> **注意**：同义词 filter 放在**索引期**是灾难——索引时展开会把"手机"和"iPhone"都写进倒排，词典更新一次就要全量 reindex，且相关性被稀释。正确姿势是**只在 search_analyzer 上做同义词**：索引保持原始词项，搜索时把用户输入展开成多路查询，词典变更秒级生效（配合 `updateable: true` + reload_search_analyzers API）。

# 6. 搜索与索引分析分离

同一字段可以指定两套分析器，这是中文检索调优的核心开关：

```json
PUT /shop
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "ik_max_word",           // 索引时细粒度
        "search_analyzer": "ik_smart"         // 搜索时粗粒度
      }
    }
  }
}
```

**效果**：索引时最大分词提高召回率，搜索时最简分词提高准确率。展开讲机制：ik_max_word 让"华为智能手机"在索引里同时挂上"华为/智能/手机/智能手机/智能手机…"多条倒排链，用户任何粒度输入都撞得上；而 ik_smart 把查询"智能手机"保持为一个完整词项去匹配，避免被拆成"智能"+"手机"后OR 出一堆只含"手机"的无关商品——**索引侧做宽网、搜索侧做准星**，两侧不对称是有意设计而非配置失误。

同一思想在其他语言同样成立：英文搜索普遍**不**在索引期跑 stemmer/synonym，而是放到搜索期（甚至用 `match` 的 minimum_should_match 替代），原因与上文同义词一致——索引期转换不可逆、词典改动代价大。**判断口诀：凡是"会变的规则"（同义词、停用词、纠错），放搜索期；凡是"稳定的形态归一"（大小写、全半角），放索引期。**

> **行业实践**：电商搜索标配组合是 `analyzer: ik_max_word + search_analyzer: ik_smart + 搜索期 synonym_graph + 拼音子字段(title.pinyin)`，再用 `multi_match` 的 `most_fields`/`cross_fields` 把 title/brand/aliases 多字段并到一次查询里。

# 7. 停用词

停用词过滤"的/了/是"这类高频虚词，目的是缩短倒排链、减少噪声匹配：

```bash
# config/analysis/stopwords.txt
的
了
是
在
和
```

```json
{
  "filter": {
    "my_stop": {
      "type": "stop",
      "stopwords_path": "analysis/stopwords.txt"
    }
  }
}
```

机制上 stop filter 直接把命中词表的分词丢弃，且 position 留下空洞（后续词项不重新编号），这会影响 phrase 查询的相邻判断，所以**停用词表宁缺毋滥**：中文单字虚词大量出现在实体名里（"在上海"的"在"、"协和"的"和"），激进过滤会直接吃掉召回。停用词放在索引期还是搜索期同样值得权衡：放索引期省空间但不可逆；放搜索期等价于"查询里自动忽略这些词"，行为更接近可控的"查询降噪"。

> **注意**：`stop` filter 与 `analyzer: stop` 是两回事；而中文虚词过滤更常见的替代方案是不删词、靠 `minimum_should_match` 调权重——删词改的是"索引里有什么"，权重改的只是"排序"，出问题时后者远容易回滚。

# 8. _analyze 调试与排错流程

分析器问题的标准排查动线（每步都可复现）：

```json
# 1. 看某个索引字段的实际分词(自动取索引配置)
POST /shop/_analyze
{ "field": "title", "text": "iPhone 15 手机壳" }

# 2. 分别验证索引期与搜索期
POST /shop/_analyze
{ "field": "title", "text": "iPhone 15 手机壳", "analyzer": "ik_max_word" }
POST /shop/_analyze
{ "field": "title", "text": "iPhone 15 手机壳", "analyzer": "ik_smart" }

# 3. 结果对不上时,用 match_phrase 验证位置语义
GET /shop/_search
{ "query": { "match_phrase": { "title": "15 手机壳" } } }
```

**搜不到数据的排查顺序**：`_analyze` 两侧词项是否一致 → 一致则看 `term` 直查倒排是否存在 → 存在则查 query 类型（match vs term 用错字段类型）。

# 9. 小结

| 主题 | 核心结论 |
|------|---------|
| 流水线结构 | CharFilter（切分前改字符）→ Tokenizer（唯一）→ TokenFilter（切分后改词项） |
| 中文必装分词器 | standard 对中文单字切分，丢失词义单元；IK 是事实标准，全节点同版本安装 |
| ik_smart vs ik_max_word | smart 粗切提准确率，max_word 细切提召回；索引 max_word + 搜索 smart 是标配 |
| 同义词/停用词位置 | 可变规则放搜索期（updateable + reload），索引期转换不可逆且改动要 reindex |
| 词典热更新 | remote_ext_dict 轮询 Last-Modified/ETag，约 60s 生效且只影响新文档 |
| 调试入口 | `_analyze` + `field` 参数看真实配置，索引/搜索两侧词项对齐是一切排查的起点 |

分析器决定了"词项空间"长什么样；下一篇（10_distributed_model）把视角从单字段放大到整个集群——一条文档如何被路由到分片、副本如何同步、写入多久后可见。
