# 1. JSON 与 JSONB

PostgreSQL 从 9.2 开始支持 `json`，9.4 引入 `jsonb`。两者名字只差一个字母，行为却完全不同，选错会直接影响查询能力与性能。

## 1.1 存储方式的差异

- **`json`**：以**原始文本**保存，保留键的顺序、空格、重复键。写入快，但每次查询都要重新解析文本。
- **`jsonb`**：写入时解析成**二进制结构**，删除重复键、不保留键顺序与空格。写入稍慢，但查询、索引、修改都高效。

```sql
SELECT '{"b":1, "a":2, "a":3}'::json;    -- 原样保留文本，含重复键
SELECT '{"b":1, "a":2, "a":3}'::jsonb;   -- 去重后只剩 a=3，键顺序可能变化
```

## 1.2 能力对比与选型

| 维度 | json | jsonb |
|------|------|-------|
| 存储 | 原始文本 | 解析后的二进制 |
| 键顺序 / 重复键 | 保留 | 不保留 / 后者覆盖前者 |
| 等值比较 | 不支持 | 支持 `=` |
| GIN 索引 / 包含查询 | 不支持 | 支持 |
| 修改函数 | 需整体重写 | `jsonb_set` 等 |

**结论很明确：业务字段一律用 `jsonb`**。`json` 只在"必须逐字节保留原始报文"的场景才有意义，例如审计日志、协议原文存档。

# 2. JSONB 的操作符与函数

## 2.1 取值

```sql
SELECT detail ->> 'channel' FROM orders;          -- app（text）
SELECT detail #>> '{address, city}' FROM orders;  -- 嵌套路径取 text
SELECT detail #>> '{items, 0, sku}' FROM orders;  -- 数组下标从 0 开始
```

`->` 和 `->>` 的区别是最常见的困惑点：需要继续做 JSON 操作时用 `->`；要拿字符串比较、拼接、放进普通列时用 `->>`。

## 2.2 存在性与包含

| 操作符 | 含义 | 示例 |
|--------|------|------|
| `@>` | 左侧是否包含右侧 | `detail @> '{"channel":"app"}'` |
| `<@` | 左侧是否被右侧包含 | `'{"a":1}'::jsonb <@ detail` |
| `?|` | 是否存在任意一个键 | `detail ?| array['a','b']` |
| `?&` | 是否同时存在所有键 | `detail ?& array['a','b']` |

```sql
SELECT * FROM orders WHERE detail @> '{"channel": "app", "amount": 120}';
SELECT * FROM orders WHERE detail ? 'coupon';
SELECT * FROM orders WHERE detail ?| ARRAY['coupon', 'gift'];
```

> **类型陷阱**：`@>` 比较的是 JSON 值的类型。`'{"amount": 100}'` 与 `'{"amount": "100"}'` 不相等。应用写入时务必保证数字就是数字、字符串就是字符串。

## 2.3 修改与展开

```sql
-- 合并：右侧覆盖左侧的同名键
SELECT '{"a":1,"b":2}'::jsonb || '{"b":3}'::jsonb;   -- {"a":1,"b":3}

-- 按路径设置值，第四个参数为 true 时自动创建缺失路径
SELECT jsonb_set(detail, '{channel}', '"web"', true) FROM orders;

-- 删除键；展开数组为多行
SELECT detail - 'coupon' FROM orders;
SELECT o.id, item ->> 'sku' AS sku, (item ->> 'qty')::int AS qty
FROM orders o, jsonb_array_elements(o.detail -> 'items') AS item;
```

注意：**JSONB 的修改没有"原地更新"**。即使只改一个键，`jsonb_set` 也会生成完整的新值，并触发整行更新，进而产生新行版本、写 WAL、可能重写 TOAST。对超大 JSON 频繁局部修改，代价很高。

# 3. JSONB 的索引

## 3.1 两种 GIN 操作类

```sql
-- jsonb_ops：支持 @>、?、?|、?&，索引更大
CREATE INDEX idx_orders_detail ON orders USING gin (detail);

-- jsonb_path_ops：只支持 @>，但索引更小、查询更快
CREATE INDEX idx_orders_detail_path ON orders USING gin (detail jsonb_path_ops);
```

| 操作类 | 支持操作 | 体积 | 适用 |
|--------|---------|------|------|
| `jsonb_ops` | `@>`、`?`、`?|`、`?&` | 较大 | 需要键存在性判断 |
| `jsonb_path_ops` | 主要支持 `@>` | 较小 | 以包含查询为主 |

如果查询全是 `detail @> '{"channel":"app"}'` 这种形式，优先 `jsonb_path_ops`。只有需要 `? 'key'` 时才用默认的 `jsonb_ops`。

## 3.2 表达式索引与部分索引

当查询固定取某个路径并做等值/范围比较，表达式索引比 GIN 更小更快：

```sql
CREATE INDEX idx_orders_channel ON orders ((detail ->> 'channel'));

SELECT * FROM orders WHERE detail ->> 'channel' = 'app';

-- 只索引有 channel 的行
CREATE INDEX idx_orders_channel_partial
    ON orders ((detail ->> 'channel')) WHERE detail ? 'channel';
```

索引失效的常见写法：对表达式再套函数（`upper(detail ->> 'channel')`）、类型不匹配（数字与字符串混用）、用 `->>` 取路径却没有建表达式索引。

> **JDBC 特别注意**：`?` 在 JDBC 里是参数占位符，`WHERE detail ? 'coupon'` 会被驱动当成参数解析而报错。可以改用 `jsonb_exists(detail, 'coupon')`，或使用驱动支持的 `??` 转义。

## 3.3 什么时候不该用 JSONB

JSONB 适合**结构不固定、整体读写、查询路径相对固定**的扩展字段。以下场景应该用普通列或独立表：字段需要频繁更新且文档很大；需要对多个键做复杂 JOIN、聚合、约束；需要外键、唯一约束、`CHECK`；数据 schema 稳定，只是图省事不想建列。

> **行业实践**：把 JSONB 当作"schema 的补充"，而不是"逃避建模的工具"。核心业务字段仍然建普通列并加约束，只有变化频繁的扩展属性放进 JSONB。

# 4. 全文检索

PG 内置全文检索，核心是 `tsvector`（文档向量）与 `tsquery`（查询向量）。它适合中小规模关键词搜索，能省掉一套 Elasticsearch 的运维成本。

## 4.1 tsvector 与 tsquery

```sql
-- 文档被切词、转小写、去停用词，并记录位置
SELECT to_tsvector('english', 'PostgreSQL is a powerful relational database');
-- 'databas':7 'postgresql':1 'power':4 'relat':6

-- 查询向量支持 & | ! 和 <->（邻近）
SELECT to_tsquery('english', 'postgresql & database');
```

```text
原始文本
   │  解析器（parser）：按语言规则切词
   ▼
词元 ──▶ 词典：去停用词、词干化、同义词
   │
   ▼
tsvector：'postgresql':1 'databas':7 ──▶ GIN 倒排索引
```

## 4.2 查询构造函数

| 函数 | 输入 | 特点 |
|------|------|------|
| `to_tsquery` | 手写 `& | !` 表达式 | 灵活但需清洗输入 |
| `plainto_tsquery` | 普通文本 | 词之间按 AND 连接 |
| `phraseto_tsquery` | 短语 | 词之间按 `<->` 邻近连接 |
| `websearch_to_tsquery` | 搜索框语法 | 支持引号短语、`or`、`-` 排除，**最适合用户输入** |

```sql
SELECT to_tsvector('english', 'PostgreSQL database') @@
       websearch_to_tsquery('english', 'postgresql -mysql');
```

## 4.3 词典与配置

PG 内置多种文本搜索配置（`pg_ts_config`），常用的是 `simple`、`english`、`zh`（需扩展）。`simple` 只做小写化，不处理停用词和词干，适合中文或需要精确匹配的场景。查看可用配置用 `\dF`，查看 `english` 的词典链用 `SELECT * FROM pg_ts_config WHERE cfgname = 'english';`。

## 4.4 GIN 索引与排名

```sql
CREATE INDEX idx_product_search ON product USING gin (search_doc);

SELECT id, name, ts_rank(search_doc, q) AS rank,
       ts_headline('simple', description, q) AS snippet
FROM product, websearch_to_tsquery('simple', '索引 优化') q
WHERE search_doc @@ q
ORDER BY rank DESC
LIMIT 20;
```

`ts_rank` 考虑词频、位置与权重，可以用 `setweight` 给不同字段不同权重。更推荐用生成列（`GENERATED ALWAYS AS (...) STORED`）自动维护 `search_doc`，避免每次写入手动计算。

生成列要求表达式**不可变（immutable）**。`to_tsvector('simple', ...)` 的第一个参数是常量配置，满足要求；如果省略配置、依赖 `default_text_search_config`，则无法用于生成列或表达式索引。

# 5. 中文分词的边界

## 5.1 默认解析器对中文不友好

PG 自带的 parser 按空格和标点切词，不理解中文词语边界。结果通常是一整段中文被当成一个词元，导致搜"数据库"匹配不到"PostgreSQL 数据库优化"，`ts_rank` 也失去意义。

```sql
-- 默认配置下，中文往往得不到期望的词元
SELECT to_tsvector('simple', 'PostgreSQL 数据库性能优化');
```

## 5.2 zhparser 与 pg_jieba

`zhparser` 基于 SCWS，安装后可以创建中文文本搜索配置：

```sql
CREATE EXTENSION zhparser;
CREATE TEXT SEARCH CONFIGURATION zh (PARSER = zhparser);
ALTER TEXT SEARCH CONFIGURATION zh ADD MAPPING FOR n,v,a,i,e,l WITH simple;

SELECT to_tsvector('zh', 'PostgreSQL 数据库性能优化');
-- 'postgresql':1 '优化':4 '性能':3 '数据库':2
```

`pg_jieba` 基于结巴分词，安装后直接使用：

```sql
CREATE EXTENSION pg_jieba;
SELECT to_tsvector('jiebacfg', 'PostgreSQL 数据库性能优化');
```

## 5.3 方案对比

| 方案 | 优点 | 缺点 | 适用 |
|------|------|------|------|
| 默认 parser + `simple` | 零依赖 | 中文几乎不可用 | 英文、代码、精确短语 |
| `zhparser` | 中文分词较好，可配词性 | 需编译安装，词典需维护 | 中小规模中文搜索 |
| `pg_jieba` | 分词质量好，使用简单 | 需安装，升级需重新编译 | 中文内容为主的应用 |
| Elasticsearch | 分词、相关性、聚合、扩展完整 | 多一套系统与运维成本 | 大规模、复杂搜索 |

> **边界判断**：数据量在千万级以内、搜索以关键词过滤为主，PG 全文检索加中文分词完全够用；如果需要拼音纠错、同义词运营、复杂相关性排序、海量日志检索，尽早引入 Elasticsearch，不要硬扛。

## 5.4 pg_trgm 补充

对于"用户输入几个字，希望按子串匹配"的场景，`pg_trgm` 往往比全文检索更直观。安装扩展后建 `gin (name gin_trgm_ops)` 索引，即可让 `LIKE '%数据库%'` 走索引；代价是索引体积较大，且对长文本效果下降。

# 6. 一个完整实践

```sql
CREATE TABLE product (
    id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name        text NOT NULL,
    category    text NOT NULL,
    price       numeric(12,2) NOT NULL,
    attrs       jsonb NOT NULL DEFAULT '{}'::jsonb,
    description text,
    search_doc  tsvector GENERATED ALWAYS AS (
        setweight(to_tsvector('simple', coalesce(name, '')), 'A') ||
        setweight(to_tsvector('simple', coalesce(description, '')), 'B')
    ) STORED
);

CREATE INDEX idx_product_category_price ON product (category, price);
CREATE INDEX idx_product_attrs ON product USING gin (attrs jsonb_path_ops);
CREATE INDEX idx_product_search ON product USING gin (search_doc);
```

```sql
-- 类目 + 属性 + 关键词，按相关度排序
SELECT p.id, p.name, p.price, ts_rank(p.search_doc, q) AS rank
FROM product p, websearch_to_tsquery('simple', '轻量 键盘') q
WHERE p.category = 'electronics'
  AND p.attrs @> '{"wireless": true}'
  AND p.search_doc @@ q
ORDER BY rank DESC, p.price ASC
LIMIT 20;
```

三种索引各司其职：B-tree 处理类目与价格排序，GIN 处理 JSONB 包含，GIN 处理全文匹配。优化器可以选择先走哪个条件，再过滤其余条件。

# 7. 常见错误与排错

| 现象 | 根因 | 处理 |
|------|------|------|
| `operator does not exist: jsonb ? unknown` | JDBC 把 `?` 当参数占位符 | 改用 `jsonb_exists(col, 'key')` 或 `??` 转义 |
| `@>` 查不到数据 | JSON 值类型不匹配（数字 vs 字符串） | 统一写入类型，检查 `jsonb_typeof` |
| GIN 索引没被使用 | 查询用了 `->>` 等非索引操作 | 建表达式索引，或改写为 `@>` |
| 中文搜索无结果 | 默认 parser 不切词 | 安装 `zhparser`/`pg_jieba`，用对应配置 |
| 生成列报错 `not immutable` | 表达式依赖会话配置 | 显式传入配置名，如 `'simple'` |
| `jsonb_set` 后表膨胀快 | 每次修改都重写整个 JSON | 拆分高频字段为普通列，控制文档大小 |

```sql
SELECT jsonb_typeof(detail -> 'amount'), detail -> 'amount' FROM orders LIMIT 5;

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE detail @> '{"channel":"app"}';
```

# 8. 练习与自检

**练习一**：建 `product` 表，写入三条含 `attrs` JSONB 的记录，分别用 `->>`、`@>`、`?` 完成"取字段值""按属性过滤""判断键是否存在"。

**练习二**：为 `attrs @> '{"brand":"acme"}'` 建 GIN 索引，用 `EXPLAIN` 验证命中；再对比 `jsonb_ops` 与 `jsonb_path_ops` 的索引大小。

**练习三**：用 `websearch_to_tsquery` 实现支持引号短语与 `-` 排除的搜索，并用 `ts_rank` 排序。说明如果内容是中文，需要额外做什么。

**自检清单**：

- 能说清 `json` 与 `jsonb` 在存储、索引、查询上的差异；
- 能区分 `->` 与 `->>`，知道什么时候必须用后者；
- 能解释 `jsonb_ops` 与 `jsonb_path_ops` 的取舍；
- 能写出 `tsvector` 生成列与 GIN 索引的完整建表语句。

# 9. 小结

| 主题 | 核心结论 |
|------|---------|
| JSON vs JSONB | 业务字段用 JSONB，json 只用于原文存档 |
| 操作符 | `->` 保留结构，`->>` 取文本，`@>` 做包含，`?` 判断键 |
| JSONB 索引 | 默认 GIN 支持键判断，`jsonb_path_ops` 更小更快但只支持 `@>` |
| 全文检索 | `tsvector` + `tsquery` + GIN，`websearch_to_tsquery` 最适合用户输入 |
| 中文 | 默认 parser 不切词，需 `zhparser` 或 `pg_jieba` |
| 边界 | 大规模、复杂相关性搜索交给 Elasticsearch，PG 负责中小规模与事务内检索 |

JSONB 与全文检索让 PG 能覆盖一部分"文档库 + 搜索"的需求，但上限受单机资源与索引结构限制。下一篇换一个方向：**扩展机制**。PG 的强大很大程度上来自 `CREATE EXTENSION` 背后那套可插拔架构——统计、加密、空间、外部数据源，都可以通过扩展接进来。