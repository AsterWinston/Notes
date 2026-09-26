# 1.Spring Data Elasticsearch 概述

## 1.1 Elasticsearch 简介

Elasticsearch（ES）是一个基于 Lucene 的分布式搜索和分析引擎：

- **近实时搜索**：数据写入后约 1 秒内即可被搜索
- **分布式架构**：自动分片和副本，支持水平扩展
- **RESTful API**：基于 HTTP/JSON 的接口设计
- **强大的全文搜索**：支持分词、同义词、拼音、模糊查询等
- **聚合分析**：提供强大的数据分析能力（类似 SQL GROUP BY）
- **高可用**：自动故障转移和数据恢复

## 1.2 Spring Data Elasticsearch 的作用

Spring Data Elasticsearch 是 Spring Data 家族中用于访问 Elasticsearch 的模块：

- 封装了 Elasticsearch 客户端的底层 API
- 提供 `ElasticsearchOperations` / `ElasticsearchRestTemplate` 模板类
- 提供 `ElasticsearchRepository` 编程模型（类似 JPA）
- 支持 `@Document` 实体映射和 `@Field` 字段映射
- 支持 `CriteriaQuery`、`StringQuery`、`NativeQuery` 多种查询方式
- 支持聚合（Aggregation）、高亮（Highlight）、排序（Sort）等高级功能
- 支持 Reactive（响应式）编程
- 与 Spring Boot 自动配置无缝集成

## 1.3 版本对应关系

| Spring Data ES | Elasticsearch | Spring Boot | Spring Framework |
|---------------|---------------|-------------|------------------|
| 5.0.x | 8.x | 3.0.x | 6.x |
| 5.1.x | 8.x | 3.1.x | 6.x |
| 5.2.x | 8.11+ | 3.2.x | 6.x |
| 5.3.x | 8.14+ | 3.3.x | 6.x |

**重要**：Spring Data Elasticsearch 版本必须与 ES 服务端版本兼容！查看官方兼容性矩阵。

# 2.快速入门

## 2.1 Maven 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
</dependency>
```

如果使用旧版 ES 7.x（不推荐，升级到 8.x）：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
</dependency>
<dependency>
    <groupId>co.elastic.clients</groupId>
    <artifactId>elasticsearch-java</artifactId>
    <version>8.14.3</version>
</dependency>
```

## 2.2 基础配置

```yaml
spring:
  elasticsearch:
    uris: http://localhost:9200          # 单节点
    # uris:                              # 多节点
    #   - http://es-node1:9200
    #   - http://es-node2:9200
    #   - http://es-node3:9200
    connection-timeout: 3s
    socket-timeout: 60s                 # 搜索超时
    username: elastic                   # ES 8.x 默认启用安全
    password: changeme

    # 高级配置（通过 Java Config 实现）
```

```java
@Configuration
public class ElasticsearchConfig {

    @Bean
    public ElasticsearchClientConfigurer customizer() {
        return new ElasticsearchClientConfigurer() {
            @Override
            public void configure(ClientConfigurationBuilder builder) {
                builder
                    .connectedTo("localhost:9200")
                    .usingSsl()
                    .withBasicAuth("elastic", "changeme")
                    .withClientConfigurer(
                        ElasticsearchClientConfigurer.createDefault())
                    .withConnectTimeout(Duration.ofSeconds(3))
                    .withSocketTimeout(Duration.ofSeconds(60));
            }
        };
    }
}
```

## 2.3 第一个 Elasticsearch 操作

```java
// 实体定义
@Document(indexName = "products")         // 索引名称
public class Product {

    @Id                                    // 文档 ID
    private String id;

    @Field(type = FieldType.Text, analyzer = "ik_max_word")
    private String name;

    @Field(type = FieldType.Keyword)
    private String category;

    @Field(type = FieldType.Double)
    private BigDecimal price;

    @Field(type = FieldType.Keyword)
    private List<String> tags;

    @Field(type = FieldType.Date,
           format = DateFormat.date_hour_minute_second)
    private LocalDateTime createTime;

    // getters and setters ...
}
```

```java
// Repository 接口
public interface ProductRepository
        extends ElasticsearchRepository<Product, String> {

    List<Product> findByName(String name);

    List<Product> findByNameContaining(String keyword);

    List<Product> findByCategoryAndPriceBetween(
        String category, BigDecimal min, BigDecimal max);
}
```

```java
@Service
public class ProductService {

    private final ProductRepository productRepository;
    private final ElasticsearchOperations operations;

    public ProductService(ProductRepository productRepository,
                          ElasticsearchOperations operations) {
        this.productRepository = productRepository;
        this.operations = operations;
    }

    public Product save(Product product) {
        return productRepository.save(product);  // 索引文档
    }

    public List<Product> searchByName(String keyword) {
        return productRepository.findByNameContaining(keyword);
    }

    public Optional<Product> findById(String id) {
        return productRepository.findById(id);
    }

    public void deleteById(String id) {
        productRepository.deleteById(id);
    }
}
```

# 3.实体映射详解

## 3.1 @Document 注解

```java
@Document(
    indexName = "products",              // 索引名（支持 SpEL 表达式）
    createIndex = true,                  // 是否在启动时自动创建索引
    dynamic = Dynamic.FALSE,             // 动态映射策略
    refreshInterval = "1s",              // 刷新间隔
    replicas = 1,                        // 副本数
    shards = 3,                          // 分片数
    writeTypeHint = WriteTypeHint.FALSE, // 是否写入 _class 字段
    versionType = VersionType.INTERNAL,  // 版本控制类型
    settingPath = "es-settings.json"     // 索引设置文件路径（classpath下）
)
public class Product {
    @Id
    private String id;
}
```

**索引命名支持 SpEL 表达式**：

```java
// 按日期分索引
@Document(indexName = "logs-#{T(java.time.LocalDate).now().toString()}")
public class LogEntry { }

// 按月份分索引（配合 Scheduled 自动创建）
@Document(indexName = "orders-#{T(java.time.format.DateTimeFormatter)
    .ofPattern('yyyy-MM').format(T(java.time.LocalDate).now())}")
public class Order { }
```

## 3.2 @Field 注解

```java
@Document(indexName = "articles")
public class Article {

    @Id
    private String id;

    // === 文本类型 ===
    @Field(type = FieldType.Text,        // 全文搜索，会被分词
           analyzer = "ik_max_word",     // 索引时分词器
           searchAnalyzer = "ik_smart",  // 搜索时分词器（可选）
           fielddata = false)            // 是否启用排序/聚合
    private String title;

    @Field(type = FieldType.Text,
           analyzer = "ik_max_word",
           index = false)                // 不索引（不可搜索）
    private String content;

    // === 关键词类型 ===
    @Field(type = FieldType.Keyword,    // 精确匹配，不分词
           normalizer = "lowercase")    // 标准化器
    private String category;

    @Field(type = FieldType.Keyword)
    private String author;

    // === 数值类型 ===
    @Field(type = FieldType.Integer)
    private Integer viewCount;

    @Field(type = FieldType.Double)
    private Double rating;

    @Field(type = FieldType.Long)
    private Long likeCount;

    // === 布尔类型 ===
    @Field(type = FieldType.Boolean)
    private Boolean published;

    // === 日期类型 ===
    @Field(type = FieldType.Date,
           format = DateFormat.date_hour_minute_second)
    private LocalDateTime publishTime;

    @Field(type = FieldType.Date,
           pattern = "yyyy-MM-dd")      // 自定义日期格式
    private LocalDate createDate;

    // === 嵌套对象 ===
    @Field(type = FieldType.Nested)     // 嵌套类型（独立文档）
    private List<Comment> comments;

    // === 对象类型 ===
    @Field(type = FieldType.Object)     // 对象类型（扁平化存储）
    private AuthorInfo authorInfo;

    // === 地理位置 ===
    @Field(type = FieldType.Text,
           analyzer = "ik_max_word")
    private String summary;

    @GeoPointField                      // 地理位置字段
    private GeoPoint location;

    // === 不作为_source存储但可搜索 ===
    @Field(type = FieldType.Keyword,
           store = true)                // 单独存储（不推荐，加大索引体积）
    private String field1;

    // === 不被索引也不被存储 ===
    @Field(type = FieldType.Keyword,
           index = false)
    private String field2;

    // === 忽略字段 ===
    @Transient                          // 完全由应用管理，不存入 ES
    private String tempField;
}
```

**FieldType 枚举详解**：

| FieldType | ES 映射类型 | 说明 | 常用场景 |
|-----------|-----------|------|---------|
| `Text` | text | 全文搜索，会被分词 | 标题、内容、描述 |
| `Keyword` | keyword | 精确匹配，不分词 | 分类、标签、ID |
| `Integer` | integer | 32位整数 | 数量、年龄 |
| `Long` | long | 64位整数 | 时间戳、ID |
| `Double` | double | 双精度浮点 | 价格、评分 |
| `Float` | float | 单精度浮点 | — |
| `Boolean` | boolean | 布尔值 | 状态标记 |
| `Date` | date | 日期 | 创建时间、更新时间 |
| `Object` | object | 扁平化嵌套对象 | 嵌套JSON |
| `Nested` | nested | 独立嵌套文档 | 多级嵌套关联查询 |
| `Geo_Point` | geo_point | 地理位置坐标 | LBS 相关 |
| `Binary` | binary | 二进制 | Base64 编码数据 |
| `Auto` | — | 自动推断类型 | 不推荐（不可控） |

## 3.3 关联与嵌套

### 3.3.1 Object 类型

```java
// 扁平化存储，JSON 查询可跨对象字段
public class Article {
    @Field(type = FieldType.Object)
    private AuthorInfo authorInfo;
}

public class AuthorInfo {
    private String name;
    private String email;
}

// 存储为：
// {
//   "authorInfo.name": "张三",
//   "authorInfo.email": "zhangsan@example.com"
// }
```

### 3.3.2 Nested 类型

```java
// 独立文档存储，保证嵌套对象的独立查询准确性
public class Article {
    @Field(type = FieldType.Nested)
    private List<Comment> comments;
}

public class Comment {
    private String author;
    private String content;
    private Integer likeCount;
}

// Nested 查询：
// 查询 comments 中 author="张三" AND content 包含"Spring" 的 article
// Object 类型会错误匹配（跨对象匹配），Nested 类型正确匹配
```

# 4.Repository 编程模型

## 4.1 Repository 接口层级

```
Repository<T, ID>
 └── CrudRepository<T, ID>
      └── PagingAndSortingRepository<T, ID>
           └── ElasticsearchRepository<T, ID>  ← ES 专用
```

**ElasticsearchRepository 提供的方法**：

```java
public interface ElasticsearchRepository<T, ID> extends PagingAndSortingRepository<T, ID> {
    <S extends T> S save(S entity);              // 保存/更新
    <S extends T> Iterable<S> saveAll(Iterable<S> entities);  // 批量保存

    Optional<T> findById(ID id);                  // 按 ID 查询
    boolean existsById(ID id);                    // 判断存在
    Iterable<T> findAll();                        // 查询所有
    Iterable<T> findAllById(Iterable<ID> ids);    // 批量按 ID 查询

    long count();                                 // 总数

    void deleteById(ID id);                       // 按 ID 删除
    void delete(T entity);                        // 按实体删除
    void deleteAllById(Iterable<? extends ID> ids);
    void deleteAll(Iterable<? extends T> entities);
    void deleteAll();                             // 删除所有（危险！）

    // 分页排序
    Iterable<T> findAll(Sort sort);
    Page<T> findAll(Pageable pageable);

    // 搜索（Spring Data ES 4.x+）
    <S extends T> S index(S entity, IndexCoordinates index);
    <S extends T> Iterable<S> save(Iterable<S> entities, IndexCoordinates index);
    SearchHits<T> search(Query query, Class<T> clazz, IndexCoordinates index);
    SearchPage<T> search(Query query, Class<T> clazz, Pageable pageable, IndexCoordinates index);
}
```

## 4.2 派生查询方法

```java
public interface ProductRepository extends ElasticsearchRepository<Product, String> {

    // === 精确匹配 ===
    List<Product> findByCategory(String category);

    // === 文本查询 ===
    List<Product> findByName(String name);

    List<Product> findByNameContaining(String keyword);
    // 生成：{ "match": { "name": { "query": "keyword", "operator": "and" } } }

    List<Product> findByTitleOrContent(String title, String content);

    // === 数值范围 ===
    List<Product> findByPriceBetween(BigDecimal min, BigDecimal max);

    List<Product> findByPriceGreaterThan(BigDecimal price);

    List<Product> findByViewCountLessThanEqual(Integer count);

    // === 布尔 ===
    List<Product> findByPublishedTrue();

    // === 日期范围 ===
    List<Product> findByCreateTimeAfter(LocalDateTime time);

    List<Product> findByCreateTimeBetween(LocalDateTime start, LocalDateTime end);

    // === 集合 ===
    List<Product> findByTagsIn(List<String> tags);

    List<Product> findByCategoryIn(List<String> categories);

    // === 存在性 ===
    boolean existsByName(String name);

    // === 统计 ===
    long countByCategory(String category);

    // === 删除 ===
    void deleteByCategory(String category);

    // === 排序 ===
    List<Product> findByCategoryOrderByPriceDesc(String category);

    List<Product> findByCategoryOrderByCreateTimeDescPriceAsc(String category);

    // === 限制结果 ===
    List<Product> findTop10ByCategoryOrderByPriceDesc(String category);
}
```

**派生查询生成规则**：

| 方法关键字 | 生成的 ES 查询类型 | 说明 |
|-----------|-----------------|------|
| `findBy*` | `term` / `match` | 精确匹配（Keyword）或全文匹配（Text） |
| `findBy*Containing` | `match` | 全文匹配（AND 操作） |
| `findBy*Between` | `range` | 范围查询 |
| `findBy*GreaterThan` | `range` (gt) | 大于 |
| `findBy*LessThanEqual` | `range` (lte) | 小于等于 |
| `findBy*True` | `term` (true) | 布尔查询 |
| `findBy*In` | `terms` | 多值精确匹配 |
| `deleteBy*` | 上述查询 + `delete_by_query` | 批量删除 |

# 5.高级查询

## 5.1 @Query 注解

```java
public interface ArticleRepository extends ElasticsearchRepository<Article, String> {

    // JSON 字符串查询（ES DSL JSON 的查询部分）
    @Query("{\"match\": {\"title\": {\"query\": \"?0\"}}}")
    List<Article> searchByTitle(String keyword);

    @Query("{\"bool\": {\"must\": [" +
           "  {\"match\": {\"title\": \"?0\"}}," +
           "  {\"term\": {\"category\": \"?1\"}}," +
           "  {\"range\": {\"publishTime\": {\"gte\": \"?2\"}}}" +
           "]}}")
    List<Article> search(String keyword, String category, String dateFrom);

    @Query("{\"multi_match\": {" +
           "  \"query\": \"?0\"," +
           "  \"fields\": [\"title^3\", \"content^1\", \"summary^2\"]" +
           "}}")
    List<Article> multiFieldSearch(String keyword);
}
```

## 5.2 CriteriaQuery（条件查询）

```java
@Service
public class ProductSearchService {

    private final ElasticsearchOperations operations;

    public SearchHits<Product> search(String name, BigDecimal minPrice,
                                       BigDecimal maxPrice, String category) {
        Criteria criteria = new Criteria();

        if (name != null && !name.isEmpty()) {
            criteria = criteria.and("name").contains(name);
        }
        if (category != null && !category.isEmpty()) {
            criteria = criteria.and("category").is(category);
        }
        if (minPrice != null) {
            criteria = criteria.and("price").greaterThanEqual(minPrice);
        }
        if (maxPrice != null) {
            criteria = criteria.and("price").lessThanEqual(maxPrice);
        }

        Query query = new CriteriaQuery(criteria);
        query.setPageable(PageRequest.of(0, 20));
        query.addSort(Sort.by(Sort.Order.desc("createTime")));

        return operations.search(query, Product.class);
    }
}
```

## 5.3 NativeQuery（原生查询）

```java
@Service
public class NativeQueryService {

    private final ElasticsearchOperations operations;

    // 全文搜索
    public SearchHits<Product> fullTextSearch(String keyword) {
        Query query = NativeQuery.builder()
            .withQuery(q -> q
                .match(m -> m
                    .field("name")
                    .query(keyword)
                    .operator(Operator.And)))
            .build();

        return operations.search(query, Product.class);
    }

    // 布尔组合查询
    public SearchHits<Product> boolSearch(String keyword, String category,
                                           BigDecimal minPrice, BigDecimal maxPrice) {
        Query query = NativeQuery.builder()
            .withQuery(q -> q
                .bool(b -> {
                    b.must(m -> m.match(ma -> ma.field("name").query(keyword)));
                    if (category != null) {
                        b.filter(f -> f.term(t -> t.field("category").value(category)));
                    }
                    if (minPrice != null || maxPrice != null) {
                        b.filter(f -> f.range(r -> {
                            r.field("price");
                            if (minPrice != null) r.gte(JsonData.of(minPrice));
                            if (maxPrice != null) r.lte(JsonData.of(maxPrice));
                            return r;
                        }));
                    }
                    return b;
                }))
            .withSort(s -> s.field(f -> f.field("price").order(SortOrder.Asc)))
            .withPageable(PageRequest.of(0, 20))
            .build();

        return operations.search(query, Product.class);
    }

    // multi_match 多字段搜索
    public SearchHits<Article> multiFieldSearch(String keyword) {
        Query query = NativeQuery.builder()
            .withQuery(q -> q
                .multiMatch(mm -> mm
                    .query(keyword)
                    .fields("title^3", "content", "summary^2")
                    .type(TextQueryType.BestFields)))
            .build();

        return operations.search(query, Article.class);
    }

    // term 精确查询
    public SearchHits<Product> exactMatch(String category) {
        Query query = NativeQuery.builder()
            .withQuery(q -> q.term(t -> t.field("category").value(category)))
            .build();

        return operations.search(query, Product.class);
    }

    // terms 多值查询
    public SearchHits<Product> multiCategory(List<String> categories) {
        Query query = NativeQuery.builder()
            .withQuery(q -> q.terms(t -> t.field("category")
                .terms(ts -> ts.value(categories.stream()
                    .map(FieldValue::of).toList()))))
            .build();

        return operations.search(query, Product.class);
    }

    // fuzzy 模糊查询（纠错）
    public SearchHits<Product> fuzzySearch(String keyword) {
        Query query = NativeQuery.builder()
            .withQuery(q -> q
                .fuzzy(f -> f
                    .field("name")
                    .value(keyword)
                    .fuzziness("AUTO")))
            .build();

        return operations.search(query, Product.class);
    }
}
```

## 5.4 StringQuery（JSON 字符串查询）

```java
@Service
public class StringQueryService {

    private final ElasticsearchOperations operations;

    public SearchHits<Product> searchWithJSON(String jsonQuery) {
        Query query = new StringQuery("""
            {
                "bool": {
                    "must": [
                        { "match": { "name": "%s" } }
                    ],
                    "filter": [
                        { "term": { "published": true } },
                        { "range": { "price": { "gte": 10, "lte": 1000 } } }
                    ]
                }
            }
            """.formatted("手机"));

        return operations.search(query, Product.class);
    }
}
```

# 6.聚合（Aggregation）

## 6.1 基本聚合

```java
@Service
public class AggregationService {

    private final ElasticsearchOperations operations;

    // 按分类分组统计
    public Map<String, Long> countByCategory() {
        Query query = NativeQuery.builder()
            .withAggregation("categories", a -> a
                .terms(t -> t.field("category").size(100)))
            .withMaxResults(0)  // 不返回文档，只要聚合结果
            .build();

        SearchHits<Product> searchHits = operations.search(query, Product.class);
        Aggregate aggregate = searchHits.getAggregations().get("categories");

        Map<String, Long> result = new LinkedHashMap<>();
        if (aggregate != null) {
            aggregate.getTerms().buckets().forEach(bucket ->
                result.put(bucket.key().stringValue(), bucket.docCount()));
        }
        return result;
    }

    // 统计指标：平均值、最大值、最小值、总和
    public void priceStats() {
        Query query = NativeQuery.builder()
            .withAggregation("price_stats", a -> a
                .stats(s -> s.field("price")))
            .withMaxResults(0)
            .build();

        SearchHits<Product> searchHits = operations.search(query, Product.class);
        Aggregate aggregate = searchHits.getAggregations().get("price_stats");
        if (aggregate != null) {
            StringStatsAggregate stats = aggregate.getStats();
            System.out.println("平均价: " + stats.getAvg());
            System.out.println("最高价: " + stats.getMax());
            System.out.println("最低价: " + stats.getMin());
            System.out.println("总数: " + stats.getCount());
            System.out.println("总和: " + stats.getSum());
        }
    }

    // 嵌套聚合：分类 → 平均价格
    public void categoryAvgPrice() {
        Query query = NativeQuery.builder()
            .withAggregation("by_category", a -> a
                .terms(t -> t.field("category"))
                .aggregations("avg_price", sub -> sub
                    .avg(av -> av.field("price"))))
            .withMaxResults(0)
            .build();

        SearchHits<Product> searchHits = operations.search(query, Product.class);
        Aggregate aggregate = searchHits.getAggregations().get("by_category");
        if (aggregate != null) {
            aggregate.getTerms().buckets().forEach(bucket -> {
                String category = bucket.key().stringValue();
                Double avgPrice = bucket.getAggregates().get("avg_price")
                    .getAvg().getValue();
                System.out.println(category + " → 均价: " + avgPrice);
            });
        }
    }

    // 时间直方图：按天统计
    public void dailyHistogram() {
        Query query = NativeQuery.builder()
            .withAggregation("daily", a -> a
                .dateHistogram(dh -> dh
                    .field("createTime")
                    .calendarInterval(CalendarInterval.Day)
                    .format("yyyy-MM-dd")))
            .withMaxResults(0)
            .build();

        SearchHits<Product> searchHits = operations.search(query, Product.class);
    }
}
```

## 6.2 常用聚合类型

| 聚合类型 | ES 类型 | 说明 |
|---------|--------|------|
| `terms` | 分组聚合 | 按字段值分组 |
| `stats` | 统计聚合 | count, min, max, avg, sum |
| `avg` | 平均值 | 单个字段平均值 |
| `sum` | 总和 | 单个字段总和 |
| `min` | 最小值 | 单个字段最小值 |
| `max` | 最大值 | 单个字段最大值 |
| `dateHistogram` | 时间直方图 | 按时间分组 |
| `histogram` | 数值直方图 | 按数值区间分组 |
| `range` | 范围聚合 | 按自定义区间分组 |
| `cardinality` | 基数聚合 | 去重计数（近似） |
| `nested` | 嵌套聚合 | 对 Nested 类型字段聚合 |
| `topHits` | 顶部匹配 | 每组内取 top N 文档 |

# 7.高亮（Highlight）

## 7.1 基础高亮

```java
@Service
public class HighlightService {

    private final ElasticsearchOperations operations;

    public SearchPage<Product> searchWithHighlight(String keyword, Pageable pageable) {
        Query query = NativeQuery.builder()
            .withQuery(q -> q.match(m -> m.field("name").query(keyword)))
            .withHighlight(Highlight.builder()
                .fields("name", HighlightField.builder()
                    .preTags("<em>")
                    .postTags("</em>")
                    .build())
                .fields("description", HighlightField.builder()
                    .numberOfFragments(2)
                    .fragmentSize(100)
                    .build()))
            .withPageable(pageable)
            .build();

        SearchHits<Product> searchHits = operations.search(query, Product.class);

        // 获取高亮内容
        List<SearchHit<Product>> hits = searchHits.getSearchHits();
        for (SearchHit<Product> hit : hits) {
            Product product = hit.getContent();
            // 从高亮结果中取高亮片段
            List<String> nameHighlights = hit.getHighlightField("name");
            if (nameHighlights != null && !nameHighlights.isEmpty()) {
                product.setName(nameHighlights.get(0)); // 用高亮内容替换
            }
        }

        return SearchHitSupport.searchPageFor(searchHits, pageable);
    }
}
```

## 7.2 多字段高亮

```java
public SearchHits<Article> multiFieldHighlight(String keyword) {
    Query query = NativeQuery.builder()
        .withQuery(q -> q.multiMatch(mm -> mm
            .query(keyword)
            .fields("title", "content", "summary")))
        .withHighlight(Highlight.builder()
            .fields("title", HighlightField.builder().preTags("<b>").postTags("</b>").build())
            .fields("content", HighlightField.builder()
                .preTags("<mark>").postTags("</mark>")
                .fragmentSize(200)
                .numberOfFragments(3)
                .build())
            .fields("summary", HighlightField.builder().preTags("<b>").postTags("</b>").build()))
        .withMaxResults(20)
        .build();

    return operations.search(query, Article.class);
}
```

# 8.批量操作与性能

## 8.1 批量索引

```java
@Service
public class BulkOperationService {

    private final ElasticsearchOperations operations;
    private final ProductRepository repository;

    // 方式1：使用 saveAll
    public void bulkSave(List<Product> products) {
        repository.saveAll(products);
    }

    // 方式2：使用 BulkOperations（更灵活）
    public void bulkIndex(List<Product> products) {
        BulkOperations bulkOps = operations.bulkOps(
            BulkOptions.builder().build(), IndexCoordinates.of("products"));

        for (Product product : products) {
            bulkOps.index(Collections.singletonList(
                new IndexQueryBuilder()
                    .withId(product.getId())
                    .withObject(product)
                    .build()
            ));
        }

        bulkOps.refresh();
        // 捕获部分失败
        // if (bulkOps.getErrors()) {
        //     bulkOps.getItems().forEach(item -> { ... });
        // }
    }
}
```

## 8.2 Scroll（滚动查询）

```java
public void scrollSearch() {
    Query query = NativeQuery.builder()
        .withQuery(q -> q.matchAll(ma -> ma))
        .build();

    SearchScrollHits<Product> scroll = operations.searchScrollStart(
        60000, query, Product.class, IndexCoordinates.of("products"));

    String scrollId = scroll.getScrollId();
    List<SearchHit<Product>> hits = new ArrayList<>(scroll.getSearchHits());

    while (scroll.hasSearchHits()) {
        hits.addAll(scroll.getSearchHits());
        scrollId = scroll.getScrollId();
        scroll = operations.searchScrollContinue(scrollId, 60000, Product.class);
    }

    operations.searchScrollClear(Collections.singletonList(scrollId));
}
```

**注意事项**：大数据量导出推荐使用 Pit（Point in Time）+ Search After 替代 Scroll（ES 7.10+）。

## 8.3 Search After（推荐的大数据量遍历方式）

```java
public void searchAfter() {
    Query query = NativeQuery.builder()
        .withQuery(q -> q.matchAll(ma -> ma))
        .withSort(s -> s.field(f -> f.field("_id").order(SortOrder.Asc)))
        .withMaxResults(1000)
        .build();

    SearchHits<Product> searchHits = operations.search(query, Product.class);
    List<SearchHit<Product>> hits = searchHits.getSearchHits();

    while (!hits.isEmpty()) {
        // 处理当前页
        processHits(hits);

        // 获取最后一篇文档的 sort values
        SearchHit<Product> lastHit = hits.get(hits.size() - 1);
        Object[] sortValues = lastHit.getSortValues().toArray();

        // 构建下一页查询
        query = NativeQuery.builder()
            .withQuery(q -> q.matchAll(ma -> ma))
            .withSort(s -> s.field(f -> f.field("_id").order(SortOrder.Asc)))
            .withSearchAfter(Arrays.asList(sortValues))
            .withMaxResults(1000)
            .build();

        searchHits = operations.search(query, Product.class);
        hits = searchHits.getSearchHits();
    }
}
```

# 9.索引管理

## 9.1 编程方式管理索引

```java
@Service
public class IndexManagementService {

    private final ElasticsearchOperations operations;
    private final IndexOperations indexOps;

    public IndexManagementService(ElasticsearchOperations operations) {
        this.operations = operations;
        this.indexOps = operations.indexOps(IndexCoordinates.of("products"));
    }

    // 创建索引
    public boolean createIndex() {
        if (!indexOps.exists()) {
            // 基于 @Document 注解自动生成 mapping
            indexOps.createWithMapping();
            return true;
        }
        return false;
    }

    // 创建索引 + 自定义设置
    public boolean createIndexWithSettings() {
        if (!indexOps.exists()) {
            Document settings = Document.create();
            settings.put("index.number_of_shards", 3);
            settings.put("index.number_of_replicas", 2);
            settings.put("index.refresh_interval", "5s");

            indexOps.create(settings);
            indexOps.putMapping(indexOps.createMapping(Product.class));
            return true;
        }
        return false;
    }

    // 删除索引
    public boolean deleteIndex() {
        return indexOps.delete();
    }

    // 刷新索引
    public void refreshIndex() {
        indexOps.refresh();
    }

    // 判断索引是否存在
    public boolean indexExists() {
        return indexOps.exists();
    }
}
```

## 9.2 动态索引管理

```java
@Service
public class DynamicIndexService {

    private final ElasticsearchOperations operations;

    // 按日期创建索引
    public void indexWithDate(Object document, LocalDate date) {
        String indexName = "logs-" + date.format(DateTimeFormatter.ISO_LOCAL_DATE);

        IndexOperations indexOps = operations.indexOps(IndexCoordinates.of(indexName));
        if (!indexOps.exists()) {
            indexOps.createWithMapping();
        }

        operations.save(document, IndexCoordinates.of(indexName));
    }

    // 索引别名管理
    public void manageAlias() {
        // 创建索引 logs-2024-01
        // 创建索引 logs-2024-02
        // 切换别名 logs-current → logs-2024-02
    }
}
```

# 10.响应式 Elasticsearch

```java
// 依赖
// <dependency>
//     <groupId>org.springframework.boot</groupId>
//     <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
// </dependency>
// Reactive 内置在 spring-boot-starter-data-elasticsearch 中

public interface ReactiveProductRepository
        extends ReactiveElasticsearchRepository<Product, String> {

    Flux<Product> findByNameContaining(String keyword);

    Flux<Product> findByCategory(String category);

    Flux<Product> findByPriceBetween(BigDecimal min, BigDecimal max);
}

@Service
public class ReactiveProductService {

    private final ReactiveProductRepository repository;
    private final ReactiveElasticsearchOperations reactiveOps;

    public Mono<Product> save(Product product) {
        return repository.save(product);
    }

    public Flux<Product> search(String keyword) {
        return repository.findByNameContaining(keyword);
    }

    public Flux<Product> reactiveSearch(String keyword) {
        Query query = NativeQuery.builder()
            .withQuery(q -> q.match(m -> m.field("name").query(keyword)))
            .withHighlight(Highlight.builder()
                .fields("name", HighlightField.builder().preTags("<em>").postTags("</em>").build()))
            .build();

        return reactiveOps.search(query, Product.class)
            .map(SearchHit::getContent);
    }
}
```

# 11.性能优化

## 11.1 索引设计优化

```yaml
# 索引设置最佳实践
index:
  number_of_shards: 3           # 根据数据量合理设置（单分片最大 50GB）
  number_of_replicas: 2         # 副本数（高可用至少 1）
  refresh_interval: 5s          # 搜索实时性要求不高可调到 30s
  translog:
    durability: async           # 异步写 translog（性能 vs 可靠性）
    sync_interval: 5s

  # mappings 优化
  dynamic: strict               # 禁止动态 mapping
  _source:
    enabled: true               # 查询无需原始文档可设为 false
  properties:
    content:
      type: text
      index_options: docs       # 只索引文档号，不存储位置/偏移（省空间）
      norms: false              # 不需要打分时禁用 norms
    keyword_field:
      type: keyword
      doc_values: true          # 需要排序/聚合时开启
      eager_global_ordinals: true  # 聚合加速（占用内存）
```

## 11.2 查询优化

```java
// 1. 只返回需要的字段（_source filtering）
Query query = NativeQuery.builder()
    .withQuery(q -> q.matchAll(ma -> ma))
    .withSourceFilter(new FetchSourceFilter(
        new String[]{"id", "name", "price"},  // includes
        new String[]{}))                      // excludes
    .build();

// 2. 使用 filter 而非 must（不参与评分）
NativeQuery.builder()
    .withQuery(q -> q.bool(b -> b
        .must(m -> m.match(ma -> ma.field("name").query("手机")))
        .filter(f -> f.term(t -> t.field("category").value("电子")))))
    .build();

// 3. 设置 search_type
// 不关心结果排序时使用 query_then_fetch（默认，已优化）

// 4. 使用 routing 路由查询
// 保存时指定 routing：Product{id="p1", routing="user:1"}
// 查询时指定 routing：NativeQuery.builder().withRoute("user:1")

// 5. 设置 preference
// NativeQuery.builder().withPreference("_local")  // 优先本地分片
```

## 11.3 连接池优化

```java
@Configuration
public class ESConnectionPoolConfig {

    @Bean
    public ElasticsearchClientConfigurer customizer() {
        return config -> {
            // 保持与 ClientConfiguration 的方式
        };
    }
}
```

```yaml
# Spring Boot 配置
spring:
  elasticsearch:
    connection-timeout: 3s
    socket-timeout: 60s
    # 注意：底层 HTTP 连接池由 ElasticsearchClient 自动管理
```

# 12.常见问题与面试题

## 12.1 写入后查询不到数据

```
原因：ES 默认 refresh_interval 为 1s，写入后需要等待 refresh 才能被搜索到

解决方案：
1. 使用 indexOps.refresh() 手动刷新（测试/对实时性要求极高场景）
2. 等待默认 1s（生产推荐）
3. 使用 get API 而非 search API（GET 实时，不依赖 refresh）
```

## 12.2 深度分页问题

```
ES 默认 from + size 分页限制最多 10000 条（index.max_result_window）

原因：from + size 深度分页需要在每个分片上查询 from+size 条数据，再协调
      节点汇总排序，越深性能越差

解决方案：
1. 使用 Scroll API（一次性大量数据导出）
2. 使用 Search After（实时遍历，推荐）
3. 使用 Pit（Point in Time）+ Search After（最佳实践）
4. 业务层面限制翻页深度（如只允许前 100 页）
```

## 12.3 ES 为什么近实时而非实时

```
1. 数据写入 → Lucene 内存 Buffer → 每秒 refresh → 生成新 segment → 可搜索
2. 写入后立即 get 可以获取（通过 translog），但 search 需要等到 refresh
3. 默认 refresh_interval = 1s
4. 降低 refresh_interval 可提升实时性，但增加磁盘和 CPU 开销
```

## 12.4 面试高频考点

**1. ES 的倒排索引原理？**
- 对文档内容分词，建立词项（Term）到文档 ID 的映射
- 查询时通过词项直接定位到包含该词的文档
- 配合 Term Dictionary 和 Term Index 加速查找

**2. ES 的写入流程？**
- 请求到达协调节点 → 路由到主分片 → 写入 translog + 内存 buffer
- 每秒 refresh：buffer 生成 segment，可搜索
- translog 达到阈值/flush：segment 写入磁盘，清空 translog

**3. ES 的选主流程？**
- 基于 Zen Discovery（7.x）或改进的协调算法（8.x）
- 候选 master 节点互相 ping，根据 node ID 排序选出 master
- 至少需要 (N/2)+1 个节点才能选出 master

**4. Text 和 Keyword 的区别？**
- Text：分词，全文搜索，不支持排序/聚合（除非 fielddata=true）
- Keyword：不分词，精确匹配，支持排序和聚合

**5. Nested 和 Object 的区别？**
- Object：扁平化存储，跨对象字段匹配可能导致假阳性
- Nested：独立文档存储，保证嵌套对象的查询准确性

# 13.最佳实践总结

```
1. Mapping 设计
   - 使用 strict 动态映射策略，避免意外字段
   - 明确每个字段的类型，不依赖自动映射
   - Text 类型指定分词器（如 ik_max_word）
   - 不需要索引的字段设 index: false
   - 不需要打分的 Text 字段设 norms: false

2. 索引命名
   - 按时间分索引：logs-2024.01、logs-2024.02
   - 使用索引别名：logs-current → logs-2024.02
   - 配合 ILM（索引生命周期管理）自动滚动

3. 查询优化
   - 精确匹配用 term/terms，不用 match
   - 不关心评分用 filter 替代 must
   - 深度分页用 Search After 替代 from+size
   - 大批量导出用 Scroll 或 Pit+Search After
   - 合理设置 max_result_window

4. 写入优化
   - 批量写入使用 Bulk API
   - 非实时要求降低 refresh_interval
   - 副本设为 0 提升写入速度（写完后设回）
   - 使用自动生成的 ID 比指定 ID 快（避免检查重复）

5. 集群规划
   - 单分片大小控制在 10-50GB
   - 分片数 = 数据量 / 单分片大小
   - 至少 3 个 master 节点（奇数）
   - 数据节点和协调节点分离
   - 内存 50% 给 JVM heap（最大不超过 32GB）

6. 监控
   - 集群健康（GET _cluster/health）
   - 节点统计（GET _nodes/stats）
   - 索引速率和搜索速率
   - JVM 堆使用率、GC 频率
   - 磁盘使用率
```
