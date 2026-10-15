# 1. Hibernate 性能概述

## 1.1 性能优化的层次

Hibernate 性能优化是一个系统工程，涉及从架构设计到基础设施的多个层次。每个层次关注的重点不同，但相互影响：

| 层次 | 关注点 | 典型技术 |
|------|--------|---------|
| **设计层** | 领域模型设计、关联方向、聚合边界 | DDD 聚合设计、读写分离、CQRS |
| **映射层** | 实体映射、集合类型、继承策略、懒加载 | @DynamicUpdate、LAZY/EAGER、Set vs List |
| **查询层** | SQL 生成效率、抓取策略、DTO 投影 | JOIN FETCH、EntityGraph、DTO Projection |
| **连接层** | 连接池、事务边界、会话管理 | HikariCP、@Transactional、OSIV |
| **基础设施层** | 数据库索引、分区、JDBC 参数 | 索引优化、连接参数、网络延迟 |

优化应遵循**自上而下**的顺序：设计层的优化收益最大，基础设施层的优化收益最小（但往往最容易被关注）。

## 1.2 常见性能问题的来源

| 问题类别 | 具体表现 | 影响程度 |
|---------|---------|---------|
| N+1 查询 | 循环访问懒加载关联，产生大量额外 SQL | 极高 |
| 笛卡尔积 | 同时 JOIN FETCH 多个集合，结果集膨胀 | 极高 |
| 无投影全字段查询 | SELECT * 加载不必要的大字段（BLOB/TEXT） | 高 |
| 持久化上下文膨胀 | 大量实体加载后未被清理，内存溢出 | 高 |
| 缺少 JDBC 批处理 | 逐条 INSERT/UPDATE，网络往返开销大 | 高 |
| No-Generator 查询 | 每次 save 都先 SELECT 获取 ID | 中 |
| 快照比较开销 | 大量实体脏检查时全字段对比 | 中 |
| 查询计划缓存过小 | 频繁编译 HQL/Criteria 为 SQL | 中 |
| 连接池泄漏 / 耗尽 | 事务边界过大或 OSIV 导致连接占用 | 高 |

## 1.3 性能优化的黄金法则

> **先测量，后优化。没有测量就没有优化。**

优化之前必须回答三个问题：
1. **瓶颈在哪？**——是 CPU、内存、I/O，还是数据库？
2. **慢在哪里？**——哪条 SQL？哪个方法？哪个关联？
3. **优化效果如何？**——优化前后的对比数据是什么？

盲目优化（如把所有 EAGER 改为 LAZY）可能引入新问题（如 N+1）。每次优化必须有数据支撑。

## 1.4 性能指标

| 指标 | 定义 | 关注点 |
|------|------|--------|
| **响应时间（Response Time）** | 从请求发起到收到响应的总时间 | P50、P95、P99 |
| **吞吐量（Throughput）** | 单位时间内处理的请求数 | TPS / QPS |
| **数据库负载** | 连接数、活跃事务数、锁等待数 | 连接池使用率、慢查询数 |
| **内存占用** | 持久化上下文大小、查询计划缓存大小 | GC 频率、Full GC 耗时 |
| **SQL 执行次数** | 单个业务操作产生的 SQL 语句数 | 越低越好（理想 1~3 条） |

---

# 2. 抓取策略优化（解决 N+1 问题）

## 2.1 N+1 问题详解

### 2.1.1 什么是 N+1 问题？

N+1 问题是指：执行 1 条查询获取 N 个父实体，然后对每个父实体再执行 1 条查询加载其关联实体，总计 **1 + N** 条 SQL。这是 ORM 中最常见也最具破坏性的性能问题。

### 2.1.2 SQL 日志演示

```java
// 实体定义
@Entity
public class Author {
    @Id @GeneratedValue
    private Long id;
    private String name;

    @OneToMany(mappedBy = "author", fetch = FetchType.LAZY)
    private List<Book> books;
}

@Entity
public class Book {
    @Id @GeneratedValue
    private Long id;
    private String title;

    @ManyToOne(fetch = FetchType.LAZY)
    private Author author;
}

// 触发 N+1 的代码
List<Author> authors = session.createQuery("FROM Author", Author.class)
        .getResultList();                           // 1 条 SQL

for (Author author : authors) {
    System.out.println(author.getBooks().size());   // N 条 SQL（每个 author 一条）
}
```

产生的 SQL 日志：

```sql
-- 1 条查询获取所有 Author
SELECT * FROM author;

-- N 条查询加载每个 Author 的 books（触发了 10 个 author，就是 10 条）
SELECT * FROM book WHERE author_id = 1;
SELECT * FROM book WHERE author_id = 2;
SELECT * FROM book WHERE author_id = 3;
-- ... 生成了 N 条
```

### 2.1.3 产生原因

1. 关联字段设置了 `FetchType.LAZY`（懒加载）
2. 业务代码在循环中访问了懒加载的关联属性
3. 每次访问触发一个新的 SQL 查询

### 2.1.4 如何发现 N+1 问题

**Hibernate Statistics**

```java
// 启用统计
Statistics stats = sessionFactory.getStatistics();
stats.setStatisticsEnabled(true);

// 执行业务操作后查看
long queryCount = stats.getQueryExecutionCount();
System.out.println("SQL 执行次数: " + queryCount);

// 理想情况：对于查询 10 个 Author + 其 books，
// 合理的 SQL 数量应为 1~2 条
// 如果发现 11 条，说明存在 N+1 问题
```

**SQL 日志分析**

```yaml
# application.yml
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE  # 显示参数绑定
```

**集成测试中断言 SQL 查询次数**

```java
// 使用 datasource-proxy 或自定义 DataSource 代理
@Test
void shouldNotTriggerNPlusOneSelectOnFindAll() {
    // 使用 datasource-proxy 统计 SQL 执行次数
    List<Author> authors = authorRepository.findAll();

    // 触发关联加载
    for (Author a : authors) {
        a.getBooks().size();
    }

    // 断言：总共执行的 SELECT 数不应超过 2
    long selectCount = proxyDataSource.getQueryCount();
    assertThat(selectCount).isLessThanOrEqualTo(2);
}
```

**Hypersistence Optimizer / JPlusOne**

```xml
<!-- Hypersistence Optimizer -->
<dependency>
    <groupId>io.hypersistence</groupId>
    <artifactId>hypersistence-optimizer</artifactId>
    <version>2.6.3</version>
</dependency>
```

```java
// 在测试配置中启用
@Bean
public HypersistenceOptimizer hypersistenceOptimizer(EntityManagerFactory emf) {
    return new HypersistenceOptimizer(
        new JpaConfig(emf)
            .setHypersistenceOptimizerConfig(
                new HypersistenceOptimizerConfig()
            )
    );
}

// 运行时会自动检测 N+1 问题并在日志中告警
```

## 2.2 JOIN FETCH

### 2.2.1 JPQL/HQL 中使用 JOIN FETCH

`JOIN FETCH` 在一条 SQL 中通过 JOIN 将关联实体一起加载，从根源上解决 N+1 问题：

```java
// 单层关联：Author + Books
List<Author> authors = session.createQuery(
    "SELECT DISTINCT a FROM Author a JOIN FETCH a.books",
    Author.class
).getResultList();

// 多层关联：Author + Books + Publisher
List<Author> authors = session.createQuery(
    "SELECT DISTINCT a FROM Author a " +
    "JOIN FETCH a.books b " +
    "JOIN FETCH b.publisher",
    Author.class
).getResultList();
```

产生的 SQL：

```sql
-- JOIN FETCH 产生的 SQL（1 条解决问题）
SELECT DISTINCT a.*, b.*
FROM author a
INNER JOIN book b ON a.id = b.author_id;
```

### 2.2.2 LEFT JOIN FETCH vs JOIN FETCH

| 类型 | 行为 | 使用场景 |
|------|------|---------|
| `JOIN FETCH` | INNER JOIN，只返回有匹配关联的实体 | 确定关联一定存在时 |
| `LEFT JOIN FETCH` | LEFT OUTER JOIN，返回所有实体（含无关联的） | 关联可能为空时 |

```java
// LEFT JOIN FETCH：即使 author 没有 books 也会返回
List<Author> authors = session.createQuery(
    "SELECT DISTINCT a FROM Author a LEFT JOIN FETCH a.books",
    Author.class
).getResultList();
```

### 2.2.3 多个集合 FETCH → MultipleBagFetchException

当同时 JOIN FETCH 多个 `List`（Bag）类型的集合时，Hibernate 无法正确处理笛卡尔积：

```java
// 这将抛出 MultipleBagFetchException！
List<Author> authors = session.createQuery(
    "SELECT a FROM Author a " +
    "JOIN FETCH a.books " +       // List<Book>
    "JOIN FETCH a.articles",      // List<Article>
    Author.class
).getResultList();
// 错误：org.hibernate.loader.MultipleBagFetchException:
// cannot simultaneously fetch multiple bags
```

**解决方案一：改用 Set 替代 List**

```java
@Entity
public class Author {
    @OneToMany(mappedBy = "author")
    private Set<Book> books = new HashSet<>();   // Set 而非 List

    @OneToMany(mappedBy = "author")
    private Set<Article> articles = new HashSet<>();
}

// 使用 Set 后可以同时 JOIN FETCH 多个集合
List<Author> authors = session.createQuery(
    "SELECT DISTINCT a FROM Author a " +
    "JOIN FETCH a.books " +
    "JOIN FETCH a.articles",
    Author.class
).getResultList();
```

Hibernate 内部使用 `Bag` 类型表示 `List`（未标注 `@OrderColumn`），而 `Bag` 无法区分重复行是原始数据重复还是 JOIN 产生的重复。`Set` 天然去重，因此可以正确处理。

**解决方案二：拆分为多次查询**

```java
// 第一次查询：加载 Author + Books
List<Author> authors = session.createQuery(
    "SELECT DISTINCT a FROM Author a JOIN FETCH a.books",
    Author.class
).getResultList();

// 第二次查询：为已加载的 Author 填充 articles（利用一级缓存）
List<Author> authorsWithArticles = session.createQuery(
    "SELECT DISTINCT a FROM Author a JOIN FETCH a.articles WHERE a IN :authors",
    Author.class
).setParameter("authors", authors)
 .getResultList();
```

### 2.2.4 JOIN FETCH + 分页冲突

在 JPQL/HQL 中使用 `JOIN FETCH` 同时设置分页时，Hibernate 会在**内存中**进行分页，而非在 SQL 层面：

```java
// 危险：这会加载所有数据到内存，然后在内存中分页
List<Author> authors = session.createQuery(
    "SELECT a FROM Author a JOIN FETCH a.books",
    Author.class
).setFirstResult(0)
 .setMaxResults(20)
 .getResultList();
```

Hibernate 会输出以下警告：

```
HHH000104: firstResult/maxResults specified with collection fetch;
applying in memory!
```

**解决方案一：两阶段查询**

```java
// 第一阶段：只查 ID，使用分页
List<Long> ids = session.createQuery(
    "SELECT a.id FROM Author a ORDER BY a.id",
    Long.class
).setFirstResult(0)
 .setMaxResults(20)
 .getResultList();

// 第二阶段：用 ID 列表做 JOIN FETCH
List<Author> authors = session.createQuery(
    "SELECT DISTINCT a FROM Author a JOIN FETCH a.books WHERE a.id IN :ids",
    Author.class
).setParameter("ids", ids)
 .getResultList();
```

**解决方案二：使用 @NamedEntityGraph（见 2.3 节）**

**解决方案三：使用 Window Function（数据库层面）**

```java
// 适用于支持 ROW_NUMBER() 的数据库（PostgreSQL、MySQL 8+、Oracle）
String sql = """
    SELECT * FROM (
        SELECT a.*, ROW_NUMBER() OVER (ORDER BY a.id) AS rn
        FROM author a
        JOIN book b ON a.id = b.author_id
    ) WHERE rn BETWEEN :start AND :end
    """;
```

### 2.2.5 @NamedEntityGraph 作为 JOIN FETCH 的替代

将抓取策略声明在实体上，查询时按需选择（详见 2.3 节）：

```java
@Entity
@NamedEntityGraph(
    name = "Author.books",
    attributeNodes = @NamedAttributeNode("books")
)
public class Author { ... }

// 使用
List<Author> authors = session.createQuery("FROM Author", Author.class)
    .setHint("jakarta.persistence.fetchgraph",
             session.getEntityGraph("Author.books"))
    .setFirstResult(0)
    .setMaxResults(20)
    .getResultList();
```

## 2.3 EntityGraph（JPA 2.1+）

### 2.3.1 @NamedEntityGraph 声明

`@NamedEntityGraph` 允许在实体上声明多种抓取策略，由查询时按名称选择：

```java
@Entity
@NamedEntityGraphs({
    @NamedEntityGraph(
        name = "Author.summary",
        attributeNodes = @NamedAttributeNode("books")
    ),
    @NamedEntityGraph(
        name = "Author.detail",
        attributeNodes = {
            @NamedAttributeNode("books"),
            @NamedAttributeNode("articles")
        }
    )
})
public class Author {
    @Id @GeneratedValue
    private Long id;
    private String name;

    @OneToMany(mappedBy = "author")
    private List<Book> books;

    @OneToMany(mappedBy = "author")
    private List<Article> articles;
}
```

### 2.3.2 @NamedAttributeNode 指定属性

```java
@NamedEntityGraph(
    name = "Book.withAuthorAndReviews",
    attributeNodes = {
        @NamedAttributeNode("author"),       // 单层关联
        @NamedAttributeNode("reviews")       // 集合关联
    }
)
```

### 2.3.3 @NamedSubgraph 嵌套图

当需要加载关联实体的关联实体（三层及以上）时使用：

```java
@NamedEntityGraph(
    name = "Author.withBooksAndPublisher",
    attributeNodes = @NamedAttributeNode(
        value = "books",
        subgraph = "books.publisher"
    ),
    subgraphs = @NamedSubgraph(
        name = "books.publisher",
        attributeNodes = @NamedAttributeNode("publisher")
    )
)
public class Author {
    @OneToMany(mappedBy = "author")
    private List<Book> books;
}

@Entity
public class Book {
    @ManyToOne
    private Publisher publisher;  // 嵌套加载
}
```

### 2.3.4 EntityManager.find() 使用 EntityGraph

```java
// 方式一：find() + EntityGraph
EntityGraph<?> graph = session.getEntityGraph("Author.detail");
Map<String, Object> hints = new HashMap<>();
hints.put("jakarta.persistence.fetchgraph", graph);

Author author = session.find(Author.class, 1L, hints);
// author.getBooks() 已被加载（LAZY 属性实际已 EAGER）
```

### 2.3.5 Query Hint 使用 EntityGraph

```java
// 方式二：JPQL/Criteria + EntityGraph
EntityGraph<?> graph = session.getEntityGraph("Author.summary");

List<Author> authors = session.createQuery("FROM Author a WHERE a.name LIKE :name", Author.class)
    .setParameter("name", "%张%")
    .setHint("jakarta.persistence.fetchgraph", graph)
    .getResultList();
```

### 2.3.6 编程方式动态创建 EntityGraph

```java
// 方式三：动态创建 EntityGraph（无需注解）
EntityGraph<Author> graph = session.createEntityGraph(Author.class);
graph.addAttributeNodes("books");
Subgraph<Book> bookSubgraph = graph.addSubgraph("books");
bookSubgraph.addAttributeNodes("publisher");

List<Author> authors = session.createQuery("FROM Author", Author.class)
    .setHint("jakarta.persistence.loadgraph", graph)
    .getResultList();
```

### 2.3.7 fetchgraph vs loadgraph 的区别

| 属性 | `fetchgraph` | `loadgraph` |
|------|-------------|-------------|
| **行为** | **仅加载** EntityGraph 中列出的属性，其他 `LAZY` 属性保持懒加载 | EntityGraph 中列出的属性**变为 EAGER**，其他属性按原有映射策略 |
| **效果** | 严格模式：未在 Graph 中的属性不会被加载 | 宽松模式：原有 EAGER 保持 EAGER |
| **适用场景** | 确定性抓取，精确控制 SQL | 叠加抓取：在原有基础上额外加载 |

```java
// fetchgraph：只有 EntityGraph 中声明的属性才会被 EAGER 加载
hints.put("jakarta.persistence.fetchgraph", graph);

// loadgraph：EntityGraph 中声明的属性 + 所有原本标注为 EAGER 的属性
hints.put("jakarta.persistence.loadgraph", graph);
```

### 2.3.8 EntityGraph 与 JOIN FETCH 对比

| 维度 | JOIN FETCH | EntityGraph |
|------|-----------|-------------|
| 定义位置 | 每次查询的 JPQL 中 | 实体注解或编程创建 |
| 复用性 | 低（每次查询需重复编写） | 高（定义一次，多处使用） |
| 分页支持 | 不支持（内存分页） | 支持（SQL 层面分页） |
| 动态性 | JPQL 字符串拼接 | 编程 API 动态添加节点 |
| 可读性 | JPQL 较长时难读 | 注解清晰，图形化 |
| 类型安全 | 否 | 是（编程方式） |
| 多集合 | 受 MultipleBagFetchException 限制 | 同样受限 |

**推荐策略**：简单查询用 `JOIN FETCH`，复杂/复用场景用 `EntityGraph`。

## 2.4 @BatchSize

### 2.4.1 批量懒加载原理

`@BatchSize` 优化的是**懒加载触发时**的查询效率。不改变懒加载的时机，但将 N 条单独的 SQL 合并为少量多条 IN 查询。

```java
@Entity
public class Author {
    @Id @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "author")
    @BatchSize(size = 10)
    private List<Book> books;
}
```

### 2.4.2 @BatchSize 注解位置

```java
// 1. 标注在实体类上（影响该实体的所有懒加载）
@Entity
@BatchSize(size = 20)
public class Book {
    @ManyToOne(fetch = FetchType.LAZY)
    private Author author;
}

// 2. 标注在集合字段上（影响该集合的懒加载）
@OneToMany(mappedBy = "author")
@BatchSize(size = 15)
private List<Book> books;

// 3. 两者结合使用
@Entity
@BatchSize(size = 30)
public class Author {
    @OneToMany(mappedBy = "author")
    @BatchSize(size = 10)   // 集合级别覆盖实体级别
    private List<Book> books;
}
```

### 2.4.3 全局配置

```yaml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 30  # 全局默认批量大小
```

### 2.4.4 工作方式与 SQL 示例

假设有 25 个 Author，`@BatchSize(size = 10)`，访问每个 Author 的 books：

```sql
-- 初始查询
SELECT * FROM author;  -- 25 个 Author

-- 批量懒加载（size=10），分 3 批而非 25 条
-- 第 1 批（Author ID 1~10）
SELECT * FROM book WHERE author_id IN (1, 2, 3, ..., 10);

-- 第 2 批（Author ID 11~20）
SELECT * FROM book WHERE author_id IN (11, 12, ..., 20);

-- 第 3 批（Author ID 21~25）
SELECT * FROM book WHERE author_id IN (21, 22, 23, 24, 25);
```

**效果**：
- 无优化：1 + 25 = 26 条 SQL
- @BatchSize(10)：1 + 3 = 4 条 SQL
- @BatchSize(25+)：1 + 1 = 2 条 SQL

### 2.4.5 size 设置建议

| size | 优缺点 |
|------|--------|
| 5~10 | 适用于单次访问少量父实体（<10 个）的场景 |
| 20~50 | 最常用范围，兼顾 SQL 长度与数据库解析开销 |
| 100+ | IN 子句过长，数据库解析变慢，甚至超出限制 |

**最佳实践**：全局 `default_batch_fetch_size = 30`，特殊场景通过 `@BatchSize` 覆盖。

### 2.4.6 @BatchSize 与 @Fetch(FetchMode.SUBSELECT) 对比

| 特性 | @BatchSize | @Fetch(SUBSELECT) |
|------|-----------|-------------------|
| SQL 形式 | `IN (id1, id2, ...)` | `WHERE parent_id IN (SELECT id FROM parent WHERE ...)` |
| SQL 条数 | 1+(N/size) 条 | 1+1 条（固定 2 条） |
| 可预测性 | 可预测 SQL 数量 | 固定 2 条但子查询可能很复杂 |
| 第一个惰性加载时机 | 访问第一个代理时立即触发批量加载 | 访问第一个代理时不触发，全部初始化完成后一次性批量加载 |
| 排序/DISTINCT | 不保证原查询的排序 | 复用原始查询条件 |

## 2.5 @Fetch(FetchMode.SUBSELECT)

### 2.5.1 子查询抓取原理

将原始查询作为子查询，一次性加载所有关联数据：

```java
@Entity
public class Author {
    @OneToMany(mappedBy = "author")
    @Fetch(FetchMode.SUBSELECT)
    private List<Book> books;
}
```

### 2.5.2 SQL 示例

```java
// 查询 name 包含"张"的 Author
List<Author> authors = session.createQuery(
    "FROM Author a WHERE a.name LIKE :name", Author.class)
    .setParameter("name", "%张%")
    .getResultList();

authors.get(0).getBooks().size();  // 触发 SUBSELECT 加载
```

生成的 SQL：

```sql
-- 主查询
SELECT * FROM author WHERE name LIKE '%张%';

-- 子查询方式一次性加载所有关联 books
SELECT * FROM book
WHERE author_id IN (
    SELECT id FROM author WHERE name LIKE '%张%'
);
```

### 2.5.3 适用场景

- **查询大量父实体**时（如列表页面），`@BatchSize` 仍会产生多条 SQL，`SUBSELECT` 仅 2 条
- 原始查询条件**有过滤**（WHERE 子句），子查询能复用该条件
- 关注**SQL 数量最小化**而非每批 SQL 开销

### 2.5.4 与 @BatchSize 的决策表

| 场景 | 推荐策略 |
|------|---------|
| 每次访问 1~3 个父实体 | `@BatchSize(5)` |
| 每次访问 10~50 个父实体（列表页） | `@BatchSize(30)` |
| 每次访问 50+ 父实体且原始查询有复杂过滤条件 | `@Fetch(SUBSELECT)` |
| 父实体数量不确定（如搜索结果） | `@Fetch(SUBSELECT)` 或 `JOIN FETCH` |

### 2.5.5 配置示例

```java
@Entity
public class Order {
    @Id @GeneratedValue
    private Long id;

    // 订单列表页通常展示大量订单，使用 SUBSELECT
    @OneToMany(mappedBy = "order")
    @Fetch(FetchMode.SUBSELECT)
    private List<OrderItem> items;

    // 订单通常只有 1 个支付记录，使用 BatchSize
    @OneToMany(mappedBy = "order")
    @BatchSize(size = 5)
    private List<Payment> payments;
}
```

## 2.6 各抓取策略对比与决策表

### 2.6.1 完整对比

| 策略 | SQL 条数（10 父实体） | 实现方式 | 优点 | 缺点 |
|------|---------------------|---------|------|------|
| **JOIN FETCH** | 1 条 | JPQL 关键字 | SQL 最少；1 次网络往返 | 不能与分页并存；笛卡尔积风险；结果集重复需 DISTINCT |
| **EntityGraph** | 1 条 | JPA Hint | 支持分页；声明式定义；可复用 | 多集合同样受限；定义较繁琐 |
| **@BatchSize(10)** | 1+1=2 条 | 注解 | 懒加载优化；不改变原有行为 | IN 列表过长时性能下降 |
| **@Fetch(SUBSELECT)** | 1+1=2 条 | 注解 | 复用原始查询过滤；SQL 条数固定 | 子查询可能很复杂；首次访问不初始化 |
| **EAGER** | 1 条 | 映射配置 | 简单直接 | 每次查询都加载；无法按需选择 |
| **LAZY（无优化）** | 1+10=11 条 | 映射配置 | 按需加载 | N+1 问题 |

### 2.6.2 场景选择决策表

```
场景：用户列表页，每页 20 条，显示用户名、文章数、粉丝数
决策：不需要加载文章/粉丝详情，仅需数量
      → 使用 @Formula 或 DTO 投影，避免加载完整集合，不触发 N+1

场景：用户详情页，显示用户信息 + 文章列表
决策：总是需要加载文章
      → JOIN FETCH 或 EntityGraph，1 条 SQL 搞定

场景：订单列表页，显示订单信息，hover 时才加载订单项
决策：大多数情况不需要订单项，少数情况需要
      → @BatchSize(20)，懒加载时批量查询

场景：管理后台，导出 5000 条用户及其角色数据
决策：一次性加载所有数据
      → DTO 投影 + 批量查询，或 StatelessSession
```

### 2.6.3 组合使用策略

```java
@Entity
public class User {
    @OneToMany(mappedBy = "user")
    @BatchSize(size = 20)           // 默认懒加载优化
    private List<Order> orders;

    @OneToMany(mappedBy = "user")
    @BatchSize(size = 10)
    private List<Address> addresses;
}

// 默认查询：依赖 @BatchSize 优化懒加载
List<User> users = userRepository.findAll();

// 特定查询：使用 EntityGraph 覆盖
@Entity
@NamedEntityGraph(
    name = "User.withOrders",
    attributeNodes = @NamedAttributeNode("orders")
)
public class User { ... }

// 需要时指定 EntityGraph
List<User> users = entityManager.createQuery("FROM User", User.class)
    .setHint("jakarta.persistence.fetchgraph",
             entityManager.getEntityGraph("User.withOrders"))
    .getResultList();
```

**原则**：@BatchSize 作为全局兜底，EntityGraph/JOIN FETCH 作为特定查询的精确抓取。

---

# 3. 查询优化

## 3.1 DTO 投影

### 3.1.1 概述

实体查询加载完整实体对象（纳入持久化上下文管理），而 DTO 投影只加载所需字段，性能优势显著：

| 特性 | 实体查询 | DTO 投影 |
|------|---------|---------|
| 持久化上下文管理 | 是（一级缓存、脏检查） | 否 |
| 脏检查开销 | 有（快照对比） | 无 |
| 内存占用 | 高（完整对象 + 代理 + 快照） | 低（仅所需字段） |
| 可控字段 | 否（默认 SELECT *） | 是（仅 SELECT 所需列） |
| 后续更新 | 支持 | 不支持（只读） |

### 3.1.2 SELECT NEW 构造器表达式

```java
// 定义 DTO
public class AuthorBookDTO {
    private String authorName;
    private String bookTitle;
    private Long bookCount;

    // 必须提供与 JPQL 中参数顺序和类型匹配的构造器
    public AuthorBookDTO(String authorName, String bookTitle, Long bookCount) {
        this.authorName = authorName;
        this.bookTitle = bookTitle;
        this.bookCount = bookCount;
    }
}

// JPQL 查询
List<AuthorBookDTO> dtos = session.createQuery(
    "SELECT NEW com.example.dto.AuthorBookDTO(a.name, b.title, COUNT(b)) " +
    "FROM Author a JOIN a.books b " +
    "GROUP BY a.name, b.title",
    AuthorBookDTO.class
).getResultList();
```

产生的 SQL 仅包含需要的列：

```sql
SELECT a.name, b.title, COUNT(b.id)
FROM author a INNER JOIN book b ON a.id = b.author_id
GROUP BY a.name, b.title;
```

### 3.1.3 Hibernate 6+ TupleTransformer

Hibernate 6 增强了 `TupleTransformer` 支持，可以将 `Object[]` 直接转换为 DTO：

```java
// Hibernate 6+ 的 Transformers
List<AuthorBookDTO> dtos = session.createQuery(
    "SELECT a.name AS authorName, b.title AS bookTitle, COUNT(b) AS bookCount " +
    "FROM Author a JOIN a.books b GROUP BY a.name, b.title",
    AuthorBookDTO.class  // Hibernate 6 直接支持
).getResultList();

// 或使用 TupleTransformer（旧方式）
List<AuthorBookDTO> dtos = session.createQuery(
    "SELECT a.name, b.title, COUNT(b) FROM Author a JOIN a.books b GROUP BY a.name, b.title"
).setTupleTransformer((tuple, aliases) -> new AuthorBookDTO(
    (String) tuple[0],
    (String) tuple[1],
    (Long) tuple[2]
)).getResultList();
```

### 3.1.4 Tuple 查询

```java
// 使用 jakarta.persistence.Tuple 获取类型安全的结果
List<Tuple> tuples = session.createQuery(
    "SELECT a.name AS authorName, COUNT(b) AS bookCount " +
    "FROM Author a LEFT JOIN a.books b GROUP BY a.id, a.name",
    Tuple.class
).getResultList();

for (Tuple tuple : tuples) {
    String name = tuple.get("authorName", String.class);
    Long count = tuple.get("bookCount", Long.class);
}
```

### 3.1.5 Object[] 投影

```java
// 最简单方式：返回 Object 数组
List<Object[]> results = session.createQuery(
    "SELECT a.name, COUNT(b) FROM Author a LEFT JOIN a.books b GROUP BY a.id, a.name"
).getResultList();

for (Object[] row : results) {
    String name = (String) row[0];
    Long count = (Long) row[1];
}
```

### 3.1.6 DTO 投影的性能优势

```java
// 场景：查询 10000 条 Author 的 name + bookCount

// 方式 1：实体查询 + 懒加载（极慢）
// - SELECT * FROM author;                     → 10000 个字段全加载
// - 10000 次 lazy 加载 books                  → 10001 条 SQL
// - 10000 个实体纳入持久化上下文，内存巨大

// 方式 2：JOIN FETCH + DISTINCT（中等）
// - SELECT DISTINCT a.*, b.* FROM author a
//   LEFT JOIN book b ...                      → 1 条 SQL，但结果集很大
// - 10000 个 Author 实体 + 各自的 books 集合  → 持久化上下文压力
// - DISTINCT 需要数据库额外排序去重

// 方式 3：DTO 投影（最优）
// - SELECT a.name, COUNT(b.id)                → 只查 2 列
//   FROM author a LEFT JOIN book b ...
//   GROUP BY a.id, a.name                     → 聚合在数据库完成
// - 对象不纳入持久化上下文                     → 无脏检查、无快照
List<AuthorStatDTO> dtos = session.createQuery(
    "SELECT NEW com.example.dto.AuthorStatDTO(a.name, COUNT(b)) " +
    "FROM Author a LEFT JOIN a.books b GROUP BY a.id, a.name",
    AuthorStatDTO.class
).getResultList();
```

## 3.2 只读查询优化

### 3.2.1 @QueryHint(name = "org.hibernate.readOnly", value = "true")

```java
// Spring Data JPA
@QueryHints(@QueryHint(name = "org.hibernate.readOnly", value = "true"))
@Query("SELECT a FROM Author a")
List<Author> findAllReadOnly();

// 原生 Hibernate
List<Author> authors = session.createQuery("FROM Author", Author.class)
    .setHint("org.hibernate.readOnly", true)
    .getResultList();
```

### 3.2.2 Session.setDefaultReadOnly(true)

```java
Session session = sessionFactory.openSession();
session.setDefaultReadOnly(true);

// 此后所有查询加载的实体都是只读的
List<Author> authors = session.createQuery("FROM Author", Author.class)
    .getResultList();

// 只读实体的修改不会被自动持久化
Author author = authors.get(0);
author.setName("New Name");
session.flush();  // 不会产生 UPDATE 语句
```

### 3.2.3 @Immutable 实体

```java
@Entity
@Immutable  // 标记实体不可变
public class Country {
    @Id
    private String code;
    private String name;
}

// 或仅在映射层级设置不可变
@org.hibernate.annotations.Immutable
```

`@Immutable` 的效果：
1. 加载后不保存快照（无脏检查开销）
2. 任何修改都不会产生 UPDATE
3. `session.flush()` 时忽略此实体
4. 适用于字典表、历史记录表等一旦插入就不修改的数据

### 3.2.4 只读优化效果总结

| 机制 | 快照保存 | 脏检查 | UPDATE 生成 | 适用场景 |
|------|---------|--------|------------|---------|
| `readOnly = "true"` | 否 | 否 | 否（当前 Session） | 单次只读查询 |
| `session.setDefaultReadOnly(true)` | 否 | 否 | 否（整个 Session） | 整个 Session 都是只读 |
| `@Immutable` | 否 | 否 | 否（任何 Session） | 永久只读实体 |

**效果量化**：对于 1000 个实体的查询，设置只读可以：
- 减少 50% 的 EntityEntry 内存（不保存快照）
- 跳过 flush 时的脏检查，显著减少 CPU 消耗

## 3.3 分页优化

### 3.3.1 传统分页（OFFSET/LIMIT）的性能问题

```java
// 传统分页
List<Author> authors = session.createQuery("FROM Author ORDER BY id", Author.class)
    .setFirstResult(100000)
    .setMaxResults(20)
    .getResultList();
```

产生的 SQL：

```sql
SELECT * FROM author ORDER BY id LIMIT 20 OFFSET 100000;
```

**问题**：数据库必须扫描并跳过前 100000 行才能返回结果。OFFSET 越大，性能越差。

### 3.3.2 游标分页（Keyset Pagination / Seek Method）

```java
// 游标分页：基于上一页最后一条记录的排序字段
Long lastId = previousPage.get(previousPage.size() - 1).getId();

List<Author> authors = session.createQuery(
    "FROM Author WHERE id > :lastId ORDER BY id",
    Author.class
).setParameter("lastId", lastId)
 .setMaxResults(20)
 .getResultList();
```

```sql
SELECT * FROM author WHERE id > 100000 ORDER BY id LIMIT 20;
```

**优势**：
- 始终使用索引查找，O(log N) 复杂度
- 不受数据总量影响
- 适合无限滚动（infinite scroll）

**限制**：
- 只能"下一页"，不能直接跳页
- 排序字段必须唯一（否则可能漏数据）
- 多字段排序时，WHERE 条件较复杂

### 3.3.3 游标分页的完整实现

```java
// 封装游标分页工具
public class PageCursor {
    private Long lastId;
    private String lastSortValue;
    private int size;

    // 组件
}

public <T> List<T> findNextPage(
    Session session, String hql, String sortField,
    Object lastSortValue, int size) {

    return session.createQuery(
        hql + " WHERE " + sortField + " > :cursor ORDER BY " + sortField,
        (Class<T>) ...
    ).setParameter("cursor", lastSortValue)
     .setMaxResults(size)
     .getResultList();
}

// 多字段游标分页
List<Order> orders = session.createQuery(
    "FROM Order o WHERE (o.createdAt, o.id) > (:lastDate, :lastId) " +
    "ORDER BY o.createdAt, o.id",
    Order.class
).setParameter("lastDate", lastDate)
 .setParameter("lastId", lastId)
 .setMaxResults(20)
 .getResultList();
```

### 3.3.4 大数据量分页统计优化

传统 COUNT 查询在大数据量下也很慢：

```sql
-- 传统 COUNT：全表扫描
SELECT COUNT(*) FROM author WHERE status = 'ACTIVE';  -- 可能几秒
```

优化方案：

```java
// 方案 1：使用数据库 EXPLAIN 估算行数（PostgreSQL）
// EXPLAIN SELECT * FROM author WHERE status = 'ACTIVE';
// 解析 EXPLAIN 输出的预估行数

// 方案 2：维护统计表
@Entity
public class AuthorStats {
    @Id
    private Long id;
    private Long activeCount;
    // 通过定时任务或触发器维护
}

// 方案 3：减少精确统计需求
// - 前端不显示总页数，只显示"下一页"
// - 或只统计最近 N 天的数据

// 方案 4：使用缓存
@Cacheable("authorCounts")
public Long countActiveAuthors() {
    return session.createQuery(
        "SELECT COUNT(*) FROM Author WHERE status = 'ACTIVE'", Long.class
    ).uniqueResult();
}
```

## 3.4 查询计划缓存

### 3.4.1 概念

Hibernate 在执行 HQL/JPQL/Criteria 查询时，需要将查询字符串编译为 SQL（查询计划）。这个编译过程有开销，因此 Hibernate 缓存编译结果。

```java
// 同一 HQL 不同参数值，共享同一个查询计划
session.createQuery("FROM Author WHERE name = :name")
    .setParameter("name", "张三").getResultList();  // 编译一次

session.createQuery("FROM Author WHERE name = :name")
    .setParameter("name", "李四").getResultList();  // 命中缓存
```

### 3.4.2 查询计划缓存大小配置

```yaml
spring:
  jpa:
    properties:
      hibernate:
        query:
          plan_cache_max_size: 2048        # 默认 2048，根据应用查询数量调整
          plan_parameter_metadata_max_size: 128  # 参数元数据缓存大小
```

### 3.4.3 参数化查询的重要性

```java
// 好：参数化查询，共享查询计划
String hql = "FROM Author WHERE name = :name";

// 差：拼接参数值，每个值都是不同的查询计划
String hql = "FROM Author WHERE name = '" + name + "'";  // 每次一个新计划！

// 差：IN 子句直接拼接
List<Long> ids = ...;
String hql = "FROM Author WHERE id IN (" +
    ids.stream().map(String::valueOf).collect(Collectors.joining(",")) + ")";
// ids 的元素数量不同 = 不同的查询计划
```

### 3.4.4 IN 子句参数填充优化

```yaml
spring:
  jpa:
    properties:
      hibernate:
        query:
          in_clause_parameter_padding: true  # Hibernate 5.2.18+
```

**原理**：Hibernate 将 IN 子句的参数数量填充到 2 的幂次，减少不同的查询计划数量：

```sql
-- in_clause_parameter_padding = false
WHERE id IN (?, ?)          -- 计划 1
WHERE id IN (?, ?, ?)       -- 计划 2
WHERE id IN (?, ?, ?, ?)    -- 计划 3
WHERE id IN (?, ?, ?, ?, ?, ?, ?, ?)  -- 计划 4

-- in_clause_parameter_padding = true（填充到 2 的幂）
WHERE id IN (?, ?)                  -- 2 个参数
WHERE id IN (?, ?, ?, ?)            -- 填充到 4
WHERE id IN (?, ?, ?, ?)            -- 不填充，已经是 4
WHERE id IN (?, ?, ?, ?, ?, ?, ?, ?)-- 填充到 8
```

**效果**：IN 子句的查询计划从 O(N) 降低到 O(log N)。

### 3.4.5 查询计划缓存内存占用

每个查询计划占用内存约 **1~2 KB**。假设 2048 个缓存条目：
- 内存占用：约 2~4 MB
- 如果查询量大且未参数化（字符串拼接），缓存会迅速膨胀并频繁淘汰
- 监控缓存命中率：

```java
Statistics stats = sessionFactory.getStatistics();
stats.setStatisticsEnabled(true);

long cacheHitCount = stats.getQueryPlanCacheHitCount();
long cacheMissCount = stats.getQueryPlanCacheMissCount();
double hitRatio = (double) cacheHitCount / (cacheHitCount + cacheMissCount);
System.out.printf("查询计划缓存命中率: %.2f%%%n", hitRatio * 100);
```

## 3.5 统计信息

### 3.5.1 Hibernate Statistics 启用

```yaml
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true       # 启用统计
```

```java
// 编程方式启用
Statistics stats = sessionFactory.getStatistics();
stats.setStatisticsEnabled(true);
```

### 3.5.2 关键指标

```java
Statistics stats = sessionFactory.getStatistics();

// 查询相关
long queryCount = stats.getQueryExecutionCount();          // HQL/JPQL/Criteria 执行次数
long queryCacheHit = stats.getQueryCacheHitCount();        // 查询缓存命中
long queryCacheMiss = stats.getQueryCacheMissCount();      // 查询缓存未命中
long queryCachePut = stats.getQueryCachePutCount();        // 查询缓存写入

// 实体相关
long entityLoadCount = stats.getEntityLoadCount();         // 实体加载总次数
long entityFetchCount = stats.getEntityFetchCount();       // 通过关联加载的次数
long entityInsertCount = stats.getEntityInsertCount();     // 插入次数
long entityUpdateCount = stats.getEntityUpdateCount();     // 更新次数
long entityDeleteCount = stats.getEntityDeleteCount();     // 删除次数

// 集合相关
long collectionLoadCount = stats.getCollectionLoadCount(); // 集合加载次数
long collectionFetchCount = stats.getCollectionFetchCount(); // 集合抓取次数
long collectionUpdateCount = stats.getCollectionUpdateCount(); // 集合更新次数

// 缓存相关
double secondLevelCacheHitRatio = stats.getSecondLevelCacheHitCount()
    / (double) (stats.getSecondLevelCacheHitCount() + stats.getSecondLevelCacheMissCount());

// 事务相关
long transactionCount = stats.getTransactionCount();       // 事务数
long successfulTransactionCount = stats.getSuccessfulTransactionCount();
long optimisticFailureCount = stats.getOptimisticFailureCount();

// 连接相关
long connectCount = stats.getConnectCount();               // 获取连接次数
long sessionOpenCount = stats.getSessionOpenCount();
long sessionCloseCount = stats.getSessionCloseCount();

// 日志输出
stats.logSummary();  // 输出统计摘要到日志
```

### 3.5.3 监控方式

**JMX 导出**：

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jmx:
          enabled: true
        generate_statistics: true
```

然后通过 JConsole 或 VisualVM 连接查看 MBean `Hibernate > Statistics`。

**Spring Boot Actuator + Micrometer**（Hibernate 5.4+）：

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-core</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: metrics,hibernate
  metrics:
    enable:
      hibernate: true
```

---

# 4. 批量操作优化

## 4.1 JDBC 批处理

### 4.1.1 核心配置

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 30                  # 每批最多 30 条 SQL
        order_inserts: true               # 排序 INSERT 以合并批处理
        order_updates: true               # 排序 UPDATE 以合并批处理
        batch_versioned_data: true        # 版本化实体也参与批处理
```

### 4.1.2 批处理原理

```java
// 无批处理：每次 persist 立即执行 INSERT
for (int i = 0; i < 1000; i++) {
    session.persist(new Book("Title " + i));
}
// 实际执行：1000 次网络往返，1000 次数据库 SQL 解析

// 有批处理：累积到 batch_size 后再批量发送
for (int i = 0; i < 1000; i++) {
    session.persist(new Book("Title " + i));
}
session.flush();  // 1000 / 30 ≈ 34 次网络往返
```

`order_inserts = true` 的作用：

```sql
-- order_inserts = false（不排序，随机批量）
INSERT INTO book VALUES (?, ?);
INSERT INTO author VALUES (?, ?);
INSERT INTO book VALUES (?, ?);   -- 无法与前一条 book INSERT 合并
INSERT INTO author VALUES (?, ?); -- 无法与前一条 author INSERT 合并
-- 实际无法合并，等同于逐条执行

-- order_inserts = true（先按表名排序，再批量）
INSERT INTO author VALUES (?, ?); -- 合并到同一批
INSERT INTO author VALUES (?, ?); -- ✓ 合并
INSERT INTO book VALUES (?, ?);   -- 合并到同一批
INSERT INTO book VALUES (?, ?);   -- ✓ 合并
```

### 4.1.3 IDENTITY 主键的限制

```java
// 问题：IDENTITY 主键生成策略无法批处理
@Entity
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)  // 无法批处理！
    private Long id;
}
```

**原因**：IDENTITY 策略要求每条 INSERT 执行后立即获取生成的 ID（通过在 JDBC 层执行 `Statement.getGeneratedKeys()`），无法延迟到批量 flush 时获取。Hibernate 会**关闭** IDENTITY 实体的 JDBC 批处理。

**解决方案**：使用 SEQUENCE 或 TABLE 策略：

```java
@Entity
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "product_seq")
    @SequenceGenerator(name = "product_seq", sequenceName = "product_seq", allocationSize = 50)
    private Long id;
}
```

`allocationSize = 50` 表示每次从数据库获取 50 个 ID 缓存在内存中，显著减少数据库交互。

### 4.1.4 批处理性能对比

```
环境：本地 MySQL，插入 10000 条记录
┌─────────────────────┬──────────┬──────────┬──────────┐
│ 策略                 │ 耗时      │ SQL 批数  │ 网络往返  │
├─────────────────────┼──────────┼──────────┼──────────┤
│ 无批处理             │ ~8500ms  │ 10000    │ 10000    │
│ batch_size=30        │ ~1200ms  │ 334      │ 334      │
│ batch_size=100       │ ~850ms   │ 100      │ 100      │
│ order_inserts=true   │ ~600ms   │ 100      │ 100      │
│ + rewriteBatchedInserts│ ~400ms │ 100       │ 100     │
└─────────────────────┴──────────┴──────────┴──────────┘
```

## 4.2 批量持久化

### 4.2.1 循环 persist + 定期 flush + clear 模式

```java
@Transactional
public void batchImport(List<BookDTO> bookDTOs) {
    Session session = entityManager.unwrap(Session.class);
    int batchSize = 50;

    for (int i = 0; i < bookDTOs.size(); i++) {
        Book book = convertToBook(bookDTOs.get(i));
        session.persist(book);

        // 每 batchSize 条就 flush + clear，防止持久化上下文膨胀
        if (i > 0 && i % batchSize == 0) {
            session.flush();   // 同步到数据库
            session.clear();   // 清空一级缓存，释放内存
        }
    }

    // 处理最后一批
    session.flush();
    session.clear();
}
```

### 4.2.2 关键点解释

| 操作 | 作用 |
|------|------|
| `session.flush()` | 将持久化上下文中积累的 INSERT/UPDATE 按 batch_size 批量发送到数据库 |
| `session.clear()` | 清空一级缓存，释放内存。**不清理会导致 OutOfMemoryError** |
| `@Transactional` | 确保整个导入在同一个事务中，享受 JDBC 批处理优化 |
| `batchSize=50` | 根据内存和业务设定。太小 flush 频繁，太大内存压力大 |

### 4.2.3 内存释放原理

```
无 flush+clear：
  内存中的持久化上下文
  [Book1, Book2, ..., Book50000]  → 占用数百 MB，可能 OOM

有 flush+clear：
  每 50 条：
  1. flush → [SQL 已发送]
  2. clear → [上下文清空，GC 可回收]
  → 内存稳定在 ~50 个实体的水平
```

## 4.3 StatelessSession

### 4.3.1 概念与特点

`StatelessSession` 是 Hibernate 提供的**无持久化上下文**的轻量级 Session，专为大批量操作设计：

| 特性 | Session | StatelessSession |
|------|---------|-----------------|
| 持久化上下文（一级缓存） | 有 | **无** |
| 脏检查（自动更新） | 有 | **无** |
| 级联操作 | 支持 | **不支持** |
| 懒加载 | 支持 | **不支持** |
| 拦截器 / 事件监听器 | 支持 | **不支持** |
| 二级缓存交互 | 读写 | **仅写入（不读取）** |
| 集合处理 | 托管 | **需手动处理** |
| 速度 | 基准 | **更快**（省略大量 ORM 开销） |

### 4.3.2 适用场景

- **大批量数据导入**（ETL、数据迁移）
- **批量导出**（导出数百万行到文件）
- **纯 JDBC 替代**（仍享受映射配置的便利）

### 4.3.3 完整代码示例

```java
public void bulkImportUsingStatelessSession(List<Author> authors, List<Book> books) {
    // 1. 打开 StatelessSession（不支持声明式事务，需手动管理）
    StatelessSession statelessSession = sessionFactory.openStatelessSession();
    Transaction tx = statelessSession.beginTransaction();

    try {
        // 2. 批量插入 Author
        for (Author author : authors) {
            statelessSession.insert(author);
        }

        // 3. 批量插入 Book（需手动设置外键关联）
        for (Book book : books) {
            // statelessSession 无级联操作，必须显式设置关联
            // book.setAuthor() 已经在转换时设置好
            statelessSession.insert(book);
        }

        tx.commit();
    } catch (Exception e) {
        tx.rollback();
        throw e;
    } finally {
        statelessSession.close();
    }
}

// 使用 JDBC 批处理 + StatelessSession
public void bulkImportWithBatch(List<Author> authors) {
    StatelessSession statelessSession = sessionFactory.openStatelessSession();
    Transaction tx = statelessSession.beginTransaction();

    try {
        for (int i = 0; i < authors.size(); i++) {
            statelessSession.insert(authors.get(i));

            if (i > 0 && i % 50 == 0) {
                // StatelessSession 无脏检查，但 insert 也受 batch_size 控制
                // 不调用 flush 也可以（StatelessSession 每次操作立即执行）
            }
        }
        tx.commit();
    } catch (Exception e) {
        tx.rollback();
        throw e;
    } finally {
        statelessSession.close();
    }
}
```

### 4.3.4 使用限制

```java
// 限制 1：insert() 后不返回生成的 ID
Author author = new Author("张三");
statelessSession.insert(author);
author.getId();  // null！StatelessSession 不管理实体状态

// 限制 2：不支持懒加载
Book book = statelessSession.get(Book.class, 1L);
book.getAuthor();  // LazyInitializationException（无 Session）

// 限制 3：不支持级联
Author author = new Author("张三");
author.getBooks().add(new Book("Java"));
statelessSession.insert(author);
// Book 不会被级联插入！必须手动 insert(book)

// 限制 4：无脏检查
statelessSession.update(author);
author.setName("李四");
// 修改不会自动同步！必须再次 update(author)

// 限制 5：不透明的集合代理
// 需要手动管理双向关联
```

## 4.4 HQL 批量操作

### 4.4.1 HQL UPDATE / DELETE

```java
// 批量更新
int updatedCount = session.createMutationQuery(
    "UPDATE Book b SET b.price = b.price * 1.1 WHERE b.publishYear < :year"
).setParameter("year", 2020)
 .executeUpdate();

// 批量删除
int deletedCount = session.createMutationQuery(
    "DELETE FROM Book b WHERE b.publishYear < :year"
).setParameter("year", 2010)
 .executeUpdate();
```

### 4.4.2 绕过 PersistenceContext 的问题

HQL 批量操作**直接作用于数据库**，不经过持久化上下文：

```java
// 问题演示
Book book = session.find(Book.class, 1L);  // 加载到一级缓存
session.createMutationQuery(
    "UPDATE Book SET price = 99 WHERE id = 1"
).executeUpdate();                          // 直接 UPDATE 数据库

// 此时 book 的 price 在一级缓存中仍然是旧值！
System.out.println(book.getPrice());        // 旧值（缓存未更新）

// 解决方案：批量操作后清理缓存
session.clear();

// 或者：在批量操作前先 flush + clear
session.flush();
session.clear();
session.createMutationQuery("UPDATE ...").executeUpdate();
```

### 4.4.3 批量操作 + Session.clear() 的完整模式

```java
@Transactional
public int batchUpdateBookPrices(int year, double multiplier) {
    Session session = entityManager.unwrap(Session.class);

    // 1. flush 现有缓存（确保之前的修改不丢失）
    session.flush();
    session.clear();

    // 2. 执行批量操作
    int count = session.createMutationQuery(
        "UPDATE Book b SET b.price = b.price * :multiplier " +
        "WHERE b.publishYear < :year"
    ).setParameter("multiplier", multiplier)
     .setParameter("year", year)
     .executeUpdate();

    // 3. 清理缓存（确保后续查询获取最新数据）
    session.clear();

    return count;
}
```

### 4.4.4 MutationQuery（Hibernate 6+）

Hibernate 6 将 INSERT/UPDATE/DELETE 查询统一为 `MutationQuery`：

```java
// Hibernate 6 新增
MutationQuery mutationQuery = session.createMutationQuery(
    "UPDATE Book b SET b.status = :newStatus WHERE b.publishYear < :year"
);
mutationQuery.setParameter("newStatus", Status.OUTDATED);
mutationQuery.setParameter("year", 2010);
mutationQuery.setTimeout(30);           // 超时 30 秒
// mutationQuery.setHint("org.hibernate.comment", "批量更新陈旧书籍");

int updated = mutationQuery.executeUpdate();

// Spring Data JPA 6+ 中使用 @Modifying
@Modifying(flushAutomatically = true, clearAutomatically = true)
@Query("UPDATE Book b SET b.price = b.price * :multiplier WHERE b.publishYear < :year")
int bulkUpdatePrices(@Param("multiplier") double multiplier, @Param("year") int year);
```

---

# 5. 连接与事务优化

## 5.1 连接池配置

### 5.1.1 HikariCP 推荐配置

HikariCP 是目前 Java 生态中性能最优的连接池，Spring Boot 2.x+ 默认使用 HikariCP：

```yaml
spring:
  datasource:
    hikari:
      # 连接池大小
      minimum-idle: 5                     # 最小空闲连接数
      maximum-pool-size: 20               # 最大连接数
      # 连接超时
      connection-timeout: 30000           # 等待连接的最大时间（ms），默认 30s
      idle-timeout: 600000                # 空闲连接最大存活时间（ms），默认 10min
      max-lifetime: 1800000               # 连接最大存活时间（ms），默认 30min
      # 连接验证
      connection-test-query: SELECT 1     # 连接有效性验证 SQL（MySQL）
      validation-timeout: 5000            # 验证超时时间（ms）
      # 泄漏检测
      leak-detection-threshold: 60000     # 连接泄漏检测阈值（ms），0 表示禁用
      # 连接初始化
      initialization-fail-timeout: 1      # 连接池初始化失败快速失败
```

### 5.1.2 连接池大小设置原则

公式：

```
连接池大小 = ((CPU 核心数 * 2) + 有效磁盘数)
```

但实际还需考虑：

| 因素 | 方向 | 说明 |
|------|------|------|
| 慢 SQL | 减小 | 连接占用时间越长，需要更多连接应对排队 |
| 事务边界短 | 减小 | 快速获取释放，降低所需连接数 |
| 高并发 | 增大 | 更多并发请求需要更多连接 |
| 数据库连接数限制 | 上限 | 不能超过数据库 `max_connections` |
| 微服务实例数 | 减小 | 多个实例分摊，单实例不需太大 |

**经验值**：
- 简单 CRUD 应用：5~10
- 中等复杂度应用：10~20
- 高并发 Web 应用：20~30
- 超过 50 需要审视：是否存在连接泄漏、事务边界过大

### 5.1.3 连接超时与泄漏检测

```yaml
spring:
  datasource:
    hikari:
      connection-timeout: 30000           # 获取连接超时
      leak-detection-threshold: 60000     # 连接持有超过 60s 视为泄漏
      # 日志输出
      pool-name: HikariPool-Business
```

泄漏检测日志示例：

```
[HikariPool-Business] Connection leak detection triggered
for conn45995719 on thread http-nio-8080-exec-3 during 65021ms,
stack trace follows...
```

### 5.1.4 配置示例（完整）

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @ConfigurationProperties("spring.datasource.hikari")
    public HikariConfig hikariConfig() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
        config.setUsername("user");
        config.setPassword("password");
        config.setMaximumPoolSize(20);
        config.setMinimumIdle(5);
        config.setConnectionTimeout(30000);
        config.setIdleTimeout(600000);
        config.setMaxLifetime(1800000);
        config.setLeakDetectionThreshold(60000);
        config.setPoolName("BusinessPool");

        // MySQL 优化属性
        config.addDataSourceProperty("cachePrepStmts", "true");
        config.addDataSourceProperty("prepStmtCacheSize", "250");
        config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
        config.addDataSourceProperty("useServerPrepStmts", "true");

        return config;
    }

    @Bean
    public DataSource dataSource(HikariConfig config) {
        return new HikariDataSource(config);
    }
}
```

## 5.2 事务优化

### 5.2.1 事务边界尽量短

```java
// 差的实践：事务覆盖整个 HTTP 请求
@Transactional  // 事务在 Controller 层开启，连接从请求开始持有到结束
@PostMapping("/orders")
public Order createOrder(@RequestBody OrderRequest request) {
    validateRequest(request);            // 不涉及数据库，占用连接
    Order order = orderService.createOrder(request);
    sendNotification(order);             // 不涉及数据库，占用连接
    saveAuditLog(order);                 // 审计日志，可异步
    return order;
}
// 连接释放：整个方法执行完毕 → 连接占用时间长

// 好的实践：事务仅在 Service 层，边界最小化
@PostMapping("/orders")
public Order createOrder(@RequestBody OrderRequest request) {
    validateRequest(request);             // 无事务
    Order order = orderService.createOrder(request);  // 事务仅在这里
    sendNotification(order);             // 无事务，可异步
    return order;
}

// Service 层
@Service
public class OrderService {
    @Transactional  // 事务仅在数据库操作时
    public Order createOrder(OrderRequest request) {
        Order order = orderRepository.save(request.toOrder());
        inventoryService.deductStock(request.getItems());
        return order;
    }
}
```

### 5.2.2 读写分离：只读事务

```java
// Spring 声明式只读事务
@Transactional(readOnly = true)
public List<Order> findOrdersByUser(Long userId) {
    return orderRepository.findByUserId(userId);
}

// Hibernate 只读事务效果：
// 1. FlushMode 自动设为 MANUAL（不自动 flush）
// 2. 跳过脏检查（readOnly 实体不保存快照）
// 3. 不对实体生成 UPDATE 语句
// 4. JDBC Connection 可设置为只读模式
```

**配置**：

```java
// Spring Data JPA + 读写分离数据源
@Bean
@Primary
public DataSource routingDataSource(
    @Qualifier("masterDataSource") DataSource master,
    @Qualifier("slaveDataSource") DataSource slave) {

    AbstractRoutingDataSource routingDataSource =
        new AbstractRoutingDataSource() {
            @Override
            protected Object determineCurrentLookupKey() {
                return TransactionSynchronizationManager
                    .isCurrentTransactionReadOnly() ? "SLAVE" : "MASTER";
            }
        };

    Map<Object, Object> targetDataSources = new HashMap<>();
    targetDataSources.put("MASTER", master);
    targetDataSources.put("SLAVE", slave);

    routingDataSource.setTargetDataSources(targetDataSources);
    routingDataSource.setDefaultTargetDataSource(master);
    return routingDataSource;
}
```

### 5.2.3 FlushMode 设置

```java
// 方式 1：通过 Session 设置
Session session = entityManager.unwrap(Session.class);
session.setHibernateFlushMode(FlushMode.MANUAL);

// 方式 2：只读事务自动设置（Spring）
@Transactional(readOnly = true)
public List<Order> queryOrders() {
    // FlushMode 自动设为 MANUAL
}

// 方式 3：写事务显式设置
@Transactional
public void bulkInsert(List<Order> orders) {
    Session session = entityManager.unwrap(Session.class);
    session.setHibernateFlushMode(FlushMode.COMMIT);
    // 只在事务提交时 flush，中间不自动 flush
}
```

### 5.2.4 事务超时设置

```java
// 方式 1：@Transactional 超时（秒）
@Transactional(timeout = 30)  // 30 秒超时
public void longRunningOperation() { ... }

// 方式 2：全局配置
spring:
  transaction:
    default-timeout: 30     # 默认事务超时 30 秒

// 方式 3：通过 Hibernate 连接超时
spring:
  jpa:
    properties:
      jakarta:
        persistence:
          query:
            timeout: 10000  # 查询超时 10 秒（毫秒）
```

## 5.3 Open Session in View 的取舍

### 5.3.1 OSIV 开启的行为

```yaml
spring:
  jpa:
    open-in-view: true   # 默认开启
```

```
请求流程：
1. 请求到达 DispatcherServlet
2. Session 打开 ← 获得数据库连接
3. → Controller → Service（事务内，使用连接）
4. → Controller（事务外，Session 仍存在）
5. → View 渲染（可触发懒加载）
6. Session 关闭 → 释放数据库连接

连接占用时间 = 请求总时间
```

### 5.3.2 OSIV 关闭的行为

```yaml
spring:
  jpa:
    open-in-view: false
```

```
请求流程：
1. 请求到达
2. → Controller → Service（事务内）→ Session 关闭 → 释放数据库连接
3. → Controller → View 渲染（无法触发懒加载 → LazyInitializationException）
```

### 5.3.3 懒加载异常与解决方案

OSIV 关闭后的常见异常：

```
org.hibernate.LazyInitializationException:
could not initialize proxy [com.example.entity.Author#1] - no Session
```

**解决方案 1：在 Service 层通过 DTO 传输所需数据**

```java
// Service 层返回 DTO（事务内完成数据组装）
@Service
public class OrderService {
    @Transactional(readOnly = true)
    public OrderDTO getOrderDTO(Long orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        // 在事务内触发所有懒加载
        return new OrderDTO(
            order.getId(),
            order.getCustomer().getName(),     // 触发 customer 加载
            order.getItems().stream()
                .map(i -> new ItemDTO(i.getProductName(), i.getQuantity()))
                .toList()                       // 触发 items 加载
        );
    }
}
```

**解决方案 2：使用 EntityGraph/JOIN FETCH**

```java
// 在 Repository 层使用 EntityGraph 预先加载
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    @EntityGraph(attributePaths = {"customer", "items"})
    Optional<Order> findById(Long id);
}
```

**解决方案 3：显式初始化代理**

```java
@Transactional
public Order getOrderDetail(Long orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    Hibernate.initialize(order.getItems());      // 显式初始化
    Hibernate.initialize(order.getCustomer());   // 显式初始化
    return order;
}
```

### 5.3.4 现代最佳实践

```
推荐：关闭 OSIV
原因：
1. 连接释放更快，连接池利用率更高
2. 避免事务边界模糊导致的意外数据库操作
3. 强制开发者思考数据加载策略
4. 适应微服务架构（无 View 层概念）

配套措施：
1. 所有 @ManyToOne 设为 LAZY
2. 查询使用 DTO 投影或 EntityGraph
3. Service 层返回 DTO 而非实体
4. 不再把实体传递到 Controller 层
```

---

# 6. 映射优化

## 6.1 懒加载策略

### 6.1.1 @ManyToOne 必须设为 LAZY

JPA 规范中 `@ManyToOne` 和 `@OneToOne` **默认是 EAGER**！这是最常见的性能陷阱。

```java
// 危险：默认 EAGER
@Entity
public class Book {
    @ManyToOne  // 默认 FetchType.EAGER！每次查询 Book 都 JOIN Author
    private Author author;
}

// 正确：显式设为 LAZY
@Entity
public class Book {
    @ManyToOne(fetch = FetchType.LAZY)  // 必须显式声明
    private Author author;
}
```

**全局配置建议**（Hibernate 特有的 hint）：

```yaml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 30   # 配合 LAZY 使用
```

注意：JPA/Hibernate 没有"全局默认 LAZY"的配置项，**必须逐个**设置 `fetch = FetchType.LAZY`。

### 6.1.2 @OneToOne 懒加载的局限性

```java
@Entity
public class User {
    @Id @GeneratedValue
    private Long id;

    @OneToOne(mappedBy = "user", fetch = FetchType.LAZY)
    private UserProfile profile;
}

@Entity
public class UserProfile {
    @Id
    private Long id;

    @OneToOne(fetch = FetchType.LAZY)
    @MapsId
    private User user;
}
```

**问题**：Hibernate 无法确定 `UserProfile` 是否存在，因此即使是 `LAZY`，查询 `User` 时可能仍会 LEFT JOIN `UserProfile`。

**原因**：Hibernate 用 `null` 区分"未加载"和"没有关联数据"。对于 `@OneToOne`（外键在对方表），Hibernate 不加载则无法判断关联是否存在。

**解决方案**：字节码增强（`@LazyToOne(NO_PROXY)`）+ 构建时增强。

### 6.1.3 @Basic(fetch = LAZY)

```java
@Entity
public class Article {
    @Id @GeneratedValue
    private Long id;

    @Lob
    @Basic(fetch = FetchType.LAZY)  // 懒加载大字段
    private String content;          // 长文本，不在列表查询中加载
}
```

**重要**：`@Basic(fetch = LAZY)` **需要字节码增强**才能生效。无增强时等同于 EAGER。

启用方式：

```xml
<!-- 构建时增强 -->
<plugin>
    <groupId>org.hibernate.orm.tooling</groupId>
    <artifactId>hibernate-enhance-maven-plugin</artifactId>
    <version>${hibernate.version}</version>
    <executions>
        <execution>
            <goals><goal>enhance</goal></goals>
            <configuration>
                <enableLazyInitialization>true</enableLazyInitialization>
            </configuration>
        </execution>
    </executions>
</plugin>
```

### 6.1.4 不能使用 LAZY 的场景

```java
// 1. final class → 无法创建代理
public final class CannotBeProxied { ... }  // 必须是非 final

// 2. final method → 代理无法拦截
@Entity
public class Entity {
    public final String getName() { ... }  // 无法懒加载
}

// 3. 空构造器为 private → 代理无法实例化
@Entity
public class Entity {
    private Entity() {}  // 需要 protected 或 public 无参构造器
}

// 4. primitive type → 无法用 null 表示未加载
@Entity
public class Entity {
    @Basic(fetch = LAZY)
    private int count;  // int 无法为 null，需要 Integer
}
```

## 6.2 @DynamicUpdate / @DynamicInsert

### 6.2.1 概念与配置

```java
@Entity
@DynamicUpdate      // 生成 UPDATE 时只包含实际变化的列
@DynamicInsert      // 生成 INSERT 时只包含非 null 的列
public class Product {
    @Id @GeneratedValue
    private Long id;

    private String name;
    private String description;
    @Column(length = 2000)
    private String specifications;
    private BigDecimal price;
    private Integer stock;
    private String category;
    private String brand;
    // ... 很多列
}
```

### 6.2.2 生成的 SQL 差异

```sql
-- 无 @DynamicUpdate：更新所有列
UPDATE product SET
    name = ?,
    description = ?,
    specifications = ?,
    price = ?,
    stock = ?,
    category = ?,
    brand = ?,
    updated_at = ?
WHERE id = ?;                          -- 9 个 SET 参数

-- 有 @DynamicUpdate：只更新变化的列
UPDATE product SET
    price = ?,
    stock = ?,
    updated_at = ?
WHERE id = ?;                          -- 仅 3 个 SET 参数
```

### 6.2.3 优缺点分析

| 优点 | 缺点 |
|------|------|
| SQL 体积小，网络传输少 | 每次生成的 SQL 不同，无法重用 PreparedStatement |
| 减少数据库写入（未变列不更新） | 增加查询计划缓存条目（每种组合一个计划） |
| 减少索引更新开销 | 更新语句编译次数增加 |
| 减少 binlog/redolog 大小 | 内存占用可能反增（计划缓存膨胀） |

### 6.2.4 适用场景

```
适合 @DynamicUpdate：
✓ 宽表（20+ 列）且大部分更新只修改少数列
✓ 列包含大字段（BLOB/TEXT）
✓ 数据库带宽是瓶颈

不适合 @DynamicUpdate：
✗ 窄表（<5 列）
✗ 大多数更新修改大部分列
✗ 写频繁的应用（查询计划缓存压力大）
```

## 6.3 集合选择

### 6.3.1 Set vs List（Bag）深度对比

```java
// 方式 1：Set
@OneToMany(mappedBy = "author")
private Set<Book> books = new HashSet<>();
// 优点：支持多个集合 JOIN FETCH、元素唯一
// 缺点：需要正确实现 equals/hashCode

// 方式 2：List + @OrderColumn
@OneToMany(mappedBy = "author")
@OrderColumn(name = "book_order")
private List<Book> books = new ArrayList<>();
// 优点：维护集合元素顺序
// 缺点：增删元素时需要更新索引列，产生大量 UPDATE

// 方式 3：List（Bag，无 @OrderColumn）
@OneToMany(mappedBy = "author")
private List<Book> books = new ArrayList<>();
// 优点：代码简单
// 缺点：不支持多个集合同时 FETCH、重加载顺序不确定
```

### 6.3.2 List + @OrderColumn 的性能陷阱

```java
// 假设已有 [A, B, C, D, E] 五个元素

// 在索引 1 插入新元素 X
books.add(1, newBook);
// 生成 SQL：
// INSERT INTO book ... (book_order = 1)
// UPDATE book SET book_order = 2 WHERE id = 'B'
// UPDATE book SET book_order = 3 WHERE id = 'C'
// UPDATE book SET book_order = 4 WHERE id = 'D'
// UPDATE book SET book_order = 5 WHERE id = 'E'
// → 1 次 INSERT + 4 次 UPDATE！
```

### 6.3.3 选择决策表

| 场景 | 推荐集合 | 原因 |
|------|---------|------|
| 元素不重复，顺序不重要 | `Set` | 支持多 FETCH、元素自动去重 |
| 顺序重要，元素可能重复 | `List` + `@OrderColumn` | 维护顺序 |
| 顺序不重要，元素可能重复，仅单向遍历 | `List`（Bag） | 简单 |
| 顺序重要，元素数量小（<50） | `List` + `@OrderColumn` | 索引更新开销可接受 |
| 顺序重要，元素数量大 | `List` + `@OrderBy`（基于字段排序） | 不维护索引列，基于自然顺序 |

### 6.3.4 @OrderBy 替代 @OrderColumn

```java
// 基于实体字段排序，不维护索引列
@OneToMany(mappedBy = "author")
@OrderBy("createdAt DESC, title ASC")
private List<Book> books = new ArrayList<>();
```

生成的 SQL（自动添加 ORDER BY）：

```sql
SELECT * FROM book WHERE author_id = ? ORDER BY created_at DESC, title ASC;
```

使用 `@OrderBy` 的优势：
- 不需要额外的索引列
- 增删元素不产生额外的 UPDATE
- 顺序由数据库查询时决定，而非存储时

---

# 7. 字节码增强

## 7.1 字节码增强的作用

Hibernate 字节码增强是在编译或类加载阶段修改实体类的字节码，以支持更高级的优化：

| 增强功能 | 默认行为 | 增强后 |
|---------|---------|--------|
| **懒加载属性**（@Basic(fetch=LAZY)） | 需显式编程处理 | 自动代理，透明懒加载 |
| **脏检查优化**（Dirty Tracking） | 全字段快照比较 | 属性级跟踪，仅比较变化字段 |
| **双向关联同步** | 需手动维护 | 自动同步双向关联 |
| **@LazyToOne(NO_PROXY)** | 使用代理（可能触发额外 SQL） | 无代理懒加载，更高效 |

## 7.2 增强方式

### 7.2.1 构建时增强（Maven）

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.hibernate.orm.tooling</groupId>
            <artifactId>hibernate-enhance-maven-plugin</artifactId>
            <version>${hibernate.version}</version>
            <executions>
                <execution>
                    <configuration>
                        <!-- 启用懒加载属性增强 -->
                        <enableLazyInitialization>true</enableLazyInitialization>
                        <!-- 启用脏检查优化（属性级别跟踪） -->
                        <enableDirtyTracking>true</enableDirtyTracking>
                        <!-- 启用自动关联管理 -->
                        <enableAssociationManagement>true</enableAssociationManagement>
                        <!-- 扩展持久化上下文（允许跨 Session 访问已加载实体） -->
                        <!-- <enableExtendedEnhancement>true</enableExtendedEnhancement> -->
                    </configuration>
                    <goals>
                        <goal>enhance</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

### 7.2.2 Gradle 配置

```groovy
buildscript {
    dependencies {
        classpath "org.hibernate:hibernate-gradle-plugin:${hibernateVersion}"
    }
}

apply plugin: 'org.hibernate.orm'

hibernate {
    enhance {
        enableLazyInitialization = true
        enableDirtyTracking = true
        enableAssociationManagement = true
    }
}
```

### 7.2.3 运行时增强（Java Agent）

```bash
java -javaagent:path/to/hibernate-enhancer-${version}.jar \
     -jar myapp.jar
```

运行时增强适用于不能修改构建流程的场景，但会增加启动时间。

### 7.2.4 增强效果验证

```java
// 验证脏检查增强
// 增强后的实体类会实现 ManagedEntity 接口
public class VerificationTest {

    @Test
    void verifyEnhancementApplied() {
        // 增强后，实体类会实现特定接口
        assertThat(Author.class)
            .isAssignableTo(org.hibernate.engine.spi.ManagedEntity.class);

        // 或检查增强后的方法存在（通过反射）
        Method[] methods = Author.class.getDeclaredMethods();
        boolean has$$_hibernate_getFieldValue = Arrays.stream(methods)
            .anyMatch(m -> m.getName().startsWith("$$_hibernate_"));
        assertThat(has$$_hibernate_getFieldValue).isTrue();
    }
}
```

### 7.2.5 配置项

```yaml
spring:
  jpa:
    properties:
      hibernate:
        enhancer:
          enableLazyInitialization: true       # 启用懒加载增强
          enableDirtyTracking: true            # 启用脏检查增强
          enableAssociationManagement: true    # 启用关联管理增强
```

## 7.3 脏检查优化原理

### 7.3.1 默认方式：全字段快照比较

```
1. 从数据库加载实体时 → 保存所有字段的快照（Object[]）
2. flush 时：
   for each managed entity:
       for each field:
           if snapshot[field] != current[field]:
               → 标记为 DIRTY
       if dirty:
           → 生成 UPDATE 语句
```

**缺点**：
- 每个实体保存两份字段数据（当前值 + 快照）
- flush 时需要遍历所有实体的所有字段
- 对于宽实体（很多列）和大量实体（10000+），开销很大

### 7.3.2 增强方式：属性级脏跟踪

```
1. 增强后，每个 setter 自动记录变化的字段：
   public void setName(String name) {
       $$_hibernate_trackChange("name");   // 字节码增强插入的代码
       this.name = name;
   }

2. flush 时：
   for each managed entity:
       if entity.hasDirtyProperties():    // O(1) 检查
           → 仅更新变化的字段
```

**效果**：
- 不需要保存完整快照（内存减半）
- flush 时无需遍历所有字段
- 配合 `@DynamicUpdate` 效果最佳

### 7.3.3 @OptimisticLocking 配合优化

```java
@Entity
@DynamicUpdate
@OptimisticLocking(type = OptimisticLockType.DIRTY)
// 乐观锁版本检查仅比较变化字段，而非所有字段
public class Product {
    @Id @GeneratedValue
    private Long id;

    @Version
    private Long version;

    private String name;
    private BigDecimal price;
    // ...
}
```

```sql
-- OptimisticLockType.ALL（默认）：WHERE 子句包含所有字段
UPDATE product SET price = ?
WHERE id = ? AND version = ? AND name = ? AND price = ? ...;

-- OptimisticLockType.DIRTY：WHERE 子句仅包含变化字段 + version
UPDATE product SET price = ?
WHERE id = ? AND version = ? AND price = ?;
-- 更短、更快、更少锁冲突
```

---

# 8. 数据库层面优化

## 8.1 索引策略

### 8.1.1 Hibernate 自动索引

Hibernate 默认为主键列自动创建索引。外键列需要手动创建索引：

```java
@Entity
@Table(name = "orders", indexes = {
    @Index(name = "idx_orders_user_id", columnList = "user_id"),
    @Index(name = "idx_orders_status_created", columnList = "status, created_at"),
    @Index(name = "idx_orders_order_no", columnList = "order_no", unique = true)
})
public class Order {
    @Id @GeneratedValue
    private Long id;  // Hibernate 自动创建索引

    @Column(name = "user_id")
    private Long userId;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    @Column(name = "order_no")
    private String orderNo;
}
```

### 8.1.2 外键列必须创建索引

```java
// 差：外键列没有索引
@ManyToOne
@JoinColumn(name = "author_id")  // 无索引
private Author author;
// → SELECT * FROM book WHERE author_id = ? → 全表扫描

// 好：显式创建外键索引
@ManyToOne
@JoinColumn(name = "author_id",
    foreignKey = @ForeignKey(name = "fk_book_author"))
@Index(name = "idx_book_author_id")  // 显式索引
private Author author;

// 或通过 @Table 级别声明
@Entity
@Table(name = "book", indexes = {
    @Index(name = "idx_book_author_id", columnList = "author_id"),
    @Index(name = "idx_book_isbn", columnList = "isbn", unique = true)
})
public class Book { ... }
```

### 8.1.3 复合索引

```java
@Entity
@Table(name = "orders", indexes = {
    // 复合索引：列顺序很重要！
    @Index(name = "idx_orders_user_status_created",
           columnList = "user_id, status, created_at"),
    // 覆盖索引（包含查询所需的全部列）
    @Index(name = "idx_orders_cover",
           columnList = "status, created_at, user_id, total_amount")
})
public class Order { ... }
```

```sql
-- 使用复合索引前 N 列的查询可以命中索引
-- idx_orders_user_status_created (user_id, status, created_at)

-- ✓ 命中索引（使用前 1 列）
SELECT * FROM orders WHERE user_id = 1;

-- ✓ 命中索引（使用前 2 列）
SELECT * FROM orders WHERE user_id = 1 AND status = 'PENDING';

-- ✓ 命中索引（使用全部 3 列）
SELECT * FROM orders WHERE user_id = 1 AND status = 'PENDING'
ORDER BY created_at;

-- ✗ 不命中（跳过了 user_id，不满足最左前缀原则）
SELECT * FROM orders WHERE status = 'PENDING';
```

### 8.1.4 索引对查询性能的影响

| 场景 | 无索引 | 有索引 |
|------|--------|--------|
| WHERE 等值条件 | 全表扫描 O(N) | B+Tree 查找 O(log N) |
| ORDER BY | FileSort | 索引直接有序 |
| JOIN 外键列 | 全表扫描 × N | 索引查找 × N |
| GROUP BY | 临时表 | 索引直接分组 |
| 覆盖索引 | 回表查询 | 仅索引扫描 |

## 8.2 DDL 优化

### 8.2.1 使用 Flyway/Liquibase 管理 Schema

**生产环境严禁使用 `hbm2ddl.auto`**：

```yaml
# 开发环境
spring:
  jpa:
    hibernate:
      ddl-auto: validate  # 或 update（开发用）

# 生产环境
spring:
  jpa:
    hibernate:
      ddl-auto: validate  # 仅验证，不修改
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
```

Flyway 迁移脚本示例：

```sql
-- V1__create_author_table.sql
CREATE TABLE author (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- V2__add_index_on_book_author_id.sql
CREATE INDEX idx_book_author_id ON book(author_id);

-- V3__add_partition_on_orders.sql
ALTER TABLE orders
PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

### 8.2.2 分区表

```java
@Entity
@Table(name = "orders")
public class Order {
    @Id
    private Long id;

    @Column(name = "created_at")
    private LocalDateTime createdAt;
}
```

配合数据库层面的分区：

```sql
-- MySQL 范围分区（按年）
ALTER TABLE orders
PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025)
);
```

Hibernate 查询时，数据库自动进行分区裁剪（Partition Pruning）：

```java
// 此查询只扫描 p2024 分区，不需要全表扫描
List<Order> orders = session.createQuery(
    "FROM Order WHERE createdAt BETWEEN :start AND :end", Order.class)
    .setParameter("start", LocalDateTime.of(2024, 1, 1, 0, 0))
    .setParameter("end", LocalDateTime.of(2024, 12, 31, 23, 59))
    .getResultList();
```

### 8.2.3 物化视图

```java
// 定义一个只读实体映射到物化视图
@Entity
@Immutable
@Table(name = "mv_author_stats")
public class AuthorStatsView {
    @Id
    private Long authorId;

    private String authorName;
    private Long bookCount;
    private Long totalReaders;
}
```

数据库层面创建物化视图：

```sql
-- PostgreSQL
CREATE MATERIALIZED VIEW mv_author_stats AS
SELECT
    a.id AS author_id,
    a.name AS author_name,
    COUNT(b.id) AS book_count,
    SUM(b.reader_count) AS total_readers
FROM author a
LEFT JOIN book b ON a.id = b.author_id
GROUP BY a.id, a.name;

-- 创建索引以加速查询
CREATE UNIQUE INDEX idx_mv_author_stats_id ON mv_author_stats(author_id);

-- 定期刷新
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_author_stats;
```

## 8.3 数据库连接参数优化

### 8.3.1 MySQL 连接参数

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb?
      useSSL=true&
      requireSSL=true&
      serverTimezone=Asia/Shanghai&
      characterEncoding=utf8mb4&
      useUnicode=true&
      socketTimeout=30000&
      connectTimeout=10000&
      autoReconnect=true&
      cachePrepStmts=true&
      prepStmtCacheSize=250&
      prepStmtCacheSqlLimit=2048&
      useServerPrepStmts=true&
      rewriteBatchedStatements=true&
      useCompression=false
```

| 参数 | 说明 | 建议值 |
|------|------|--------|
| `socketTimeout` | Socket 读取超时 | 30000 ms |
| `connectTimeout` | 建立连接超时 | 10000 ms |
| `useSSL` | 使用 SSL | true（生产） |
| `characterEncoding` | 字符编码 | utf8mb4 |
| `cachePrepStmts` | 缓存 PreparedStatement | true |
| `prepStmtCacheSize` | PS 缓存大小 | 250 |
| `rewriteBatchedStatements` | 将多条 INSERT 重写为单条多 VALUES 语句 | true |
| `useServerPrepStmts` | 使用服务端预编译 | true（减少 SQL 注入风险） |

### 8.3.2 PostgreSQL 连接参数

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb?
      currentSchema=public&
      ApplicationName=myapp&
      socketTimeout=30&
      connectTimeout=10&
      tcpKeepAlive=true&
      prepareThreshold=5&
      reWriteBatchedInserts=true
```

### 8.3.3 rewriteBatchedStatements 的重要作用

```sql
-- rewriteBatchedStatements = false（逐条 INSERT）
INSERT INTO book (title, author_id) VALUES ('Book 1', 1);
INSERT INTO book (title, author_id) VALUES ('Book 2', 1);
INSERT INTO book (title, author_id) VALUES ('Book 3', 1);

-- rewriteBatchedStatements = true（合并为单条多 VALUES 语句）
INSERT INTO book (title, author_id) VALUES
('Book 1', 1),
('Book 2', 1),
('Book 3', 1);
```

单条多 VALUES INSERT 比多次独立 INSERT 快 **3~5 倍**。

---

# 9. 性能测试与监控

## 9.1 性能测试方法

### 9.1.1 JMH（Java Microbenchmark Harness）

```java
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
@State(Scope.Thread)
@Fork(1)
@Warmup(iterations = 3, time = 1)
@Measurement(iterations = 5, time = 1)
public class FetchStrategyBenchmark {

    private EntityManagerFactory emf;
    private EntityManager em;

    @Setup
    public void setup() {
        emf = Persistence.createEntityManagerFactory("test-pu");
        em = emf.createEntityManager();

        // 准备测试数据：100 个 Author，每个 10 个 Book
        em.getTransaction().begin();
        for (int i = 0; i < 100; i++) {
            Author author = new Author("Author " + i);
            for (int j = 0; j < 10; j++) {
                author.getBooks().add(new Book("Book " + i + "-" + j, author));
            }
            em.persist(author);
        }
        em.getTransaction().commit();
        em.clear();
    }

    @Benchmark
    public List<Author> lazyWithNPlusOne() {
        List<Author> authors = em.createQuery("FROM Author", Author.class)
            .getResultList();
        for (Author a : authors) {
            a.getBooks().size();  // 触发 N+1
        }
        return authors;
    }

    @Benchmark
    public List<Author> joinFetch() {
        return em.createQuery(
            "SELECT DISTINCT a FROM Author a JOIN FETCH a.books", Author.class
        ).getResultList();
    }

    @Benchmark
    public List<Author> batchSize() {
        List<Author> authors = em.createQuery("FROM Author", Author.class)
            .getResultList();
        for (Author a : authors) {
            a.getBooks().size();  // @BatchSize 优化后的懒加载
        }
        return authors;
    }

    @TearDown
    public void tearDown() {
        em.close();
        emf.close();
    }
}
```

### 9.1.2 集成测试中的 SQL 断言

```java
@SpringBootTest
@AutoConfigureTestDatabase(replace = Replace.NONE)
class AuthorRepositoryPerformanceTest {

    @Autowired
    private DataSource dataSource;

    private ProxyDataSource proxyDataSource;
    private JdbcTemplate jdbcTemplate;

    @BeforeEach
    void setUp() {
        proxyDataSource = ProxyDataSourceBuilder.create(dataSource)
            .countQuery()    // 统计查询次数
            .build();
    }

    @Test
    void shouldNotHaveNPlusOneProblem() {
        proxyDataSource.reset();

        List<Author> authors = authorService.findAllWithBooks();

        // 断言：不应超过 2 次 SELECT
        long selectCount = proxyDataSource.getQueryCount();
        assertThat(selectCount)
            .as("作者列表查询不应产生 N+1 问题")
            .isLessThanOrEqualTo(2);
    }

    @Test
    void batchInsertShouldUseBatching() {
        proxyDataSource.reset();

        authorService.batchCreate(100);

        // 断言：100 次 INSERT 的批次数不应超过 5
        long batchCount = proxyDataSource.getExecuteBatchCount();
        assertThat(batchCount)
            .as("批量插入应使用 JDBC 批处理")
            .isLessThanOrEqualTo(5);
    }
}
```

### 9.1.3 datasource-proxy 配置

```java
@Configuration
public class DataSourceProxyConfig {

    @Bean
    @Primary
    public DataSource dataSource(DataSource originalDataSource) {
        return ProxyDataSourceBuilder.create(originalDataSource)
            .name("ProxyDS")
            .countQuery()
            .logQueryToSysOut()             // 输出 SQL 到控制台
            .logSlowQueryToSysOut(50, MILLISECONDS)  // 慢查询阈值
            .multiline()                     // 多行格式化
            .build();
    }
}
```

### 9.1.4 p6spy 工具

```xml
<dependency>
    <groupId>p6spy</groupId>
    <artifactId>p6spy</artifactId>
    <version>3.9.1</version>
</dependency>
```

```properties
# spy.properties
driverlist=com.mysql.cj.jdbc.Driver
dateformat=yyyy-MM-dd HH:mm:ss.SSS
appender=com.p6spy.engine.spy.appender.Slf4JLogger
logMessageFormat=com.p6spy.engine.spy.appender.MultiLineFormat
excludecategories=info,debug,result,resultset,commit,rollback
outagedetection=true
outagedetectioninterval=2
```

```java
@Configuration
public class P6SpyConfig {
    @Bean
    public DataSource dataSource() {
        return new P6DataSource(originalDataSource());
    }
}
```

## 9.2 生产环境监控

### 9.2.1 Hibernate Statistics + JMX

```yaml
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true
        jmx:
          enabled: true
          session_factory_name: "persistence:app=HibernateStats"
```

通过 JMX 监控的关键指标：

| MBean 属性 | 含义 | 告警阈值 |
|-----------|------|---------|
| `QueryExecutionCount` | 查询总次数 | 与预期对比，突增意味着 N+1 |
| `SessionOpenCount - SessionCloseCount` | 活跃 Session 数 | > 0 意味着 Session 泄漏 |
| `SecondLevelCacheHitCount / (Hit+Miss)` | 缓存命中率 | < 80% |
| `CollectionFetchCount / CollectionLoadCount` | 集合抓取比 | > 20% 需检查 |
| `ConnectCount` | 获取连接次数 | 与 Session 数对比 |
| `OptimisticFailureCount` | 乐观锁冲突次数 | > 0 需关注 |

### 9.2.2 Spring Boot Actuator + Micrometer

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  metrics:
    enable:
      hibernate: true
    distribution:
      percentiles-histogram:
        "[hibernate.query.execution.time]": true
        "[hibernate.entity.fetch.time]": true
  export:
    prometheus:
      enabled: true
```

Grafana + Prometheus 监控面板关键指标：

```
# Hibernate 查询耗时分布
hibernate_query_execution_time_seconds_bucket

# Hibernate Session 数量
hibernate_session_metrics_session_count

# 实体加载次数
hibernate_statistics_entity_loads_total

# 查询缓存命中率
rate(hibernate_query_cache_hit_total[5m])
/ (rate(hibernate_query_cache_hit_total[5m]) + rate(hibernate_query_cache_miss_total[5m]))
```

### 9.2.3 Hibernate 内置 Micrometer 集成（Hibernate 5.4+）

```java
// Hibernate 6 原生支持 Micrometer
@Configuration
public class HibernateMetricsConfig {

    @Bean
    public MeterRegistry meterRegistry() {
        return new SimpleMeterRegistry();
    }

    @Bean
    public HibernateMetrics hibernateMetrics(
            EntityManagerFactory entityManagerFactory,
            MeterRegistry meterRegistry) {
        SessionFactory sessionFactory =
            entityManagerFactory.unwrap(SessionFactory.class);

        return new HibernateMetrics(sessionFactory, "hibernate", List.of("app"));
    }
}
```

### 9.2.4 慢查询检测（Hibernate 6+）

```yaml
spring:
  jpa:
    properties:
      hibernate:
        session:
          events:
            log:
              LOG_QUERIES_SLOWER_THAN_MS: 100  # 超过 100ms 的查询记录日志
```

日志输出：

```
HHH90005001: Execution time for query "FROM Author a JOIN FETCH a.books"
exceeded the threshold of 100ms by 347ms.
```

自定义慢查询处理器：

```java
public class SlowQueryHandler implements SessionEventListener {

    private static final long SLOW_THRESHOLD_MS = 200;

    @Override
    public void jdbcExecuteStatementEnd(SessionEventListenerManager eventManager) {
        long executionTime = eventManager.getExecutionTime();
        if (executionTime > SLOW_THRESHOLD_MS) {
            log.warn("慢查询检测 [{}ms] SQL: {}",
                executionTime, eventManager.getQueryString());
        }
    }
}

// 注册
SessionFactory factory = ...;
factory.getSessionFactoryOptions().getSessionEventListeners()
    .add(new SlowQueryHandler());
```

---

# 10. 常见性能问题与解决方案速查表

| 问题 | 症状 | 原因 | 解决方案 | 参考章节 |
|------|------|------|---------|---------|
| **N+1 查询** | 日志中出现大量重复 SQL | LAZY + 循环访问关联 | JOIN FETCH / @BatchSize / EntityGraph | 2.1~2.6 |
| **MultipleBagFetchException** | 启动异常：cannot simultaneously fetch multiple bags | 同时 JOIN FETCH 多个 List | 改 List 为 Set / 拆分查询 | 2.2 |
| **笛卡尔积爆炸** | 结果集异常大 / 内存飙升 | 多对多 JOIN FETCH 结果集膨胀 | 分开查询 / 使用 Set / 减少同时 FETCH 的集合数 | 2.2 |
| **大量数据加载导致 OOM** | OutOfMemoryError | 未分页 + 未 clear | 分页 / DTO 投影 / flush+clear / StatelessSession | 4.2, 4.3 |
| **批量 INSERT 慢** | 导入 10000 条数据耗时数分钟 | 未配置 JDBC 批处理 | batch_size + order_inserts + rewriteBatchedInserts | 4.1 |
| **IDENTITY 批处理失效** | 即使配置了 batch_size，INSERT 仍逐条执行 | IDENTITY 主键策略无法批处理 | 改用 SEQUENCE 策略 | 4.1 |
| **LazyInitializationException** | 事务/OSIV 关闭后访问懒加载属性 | Session 已关闭 | EntityGraph / DTO / Hibernate.initialize() | 5.3, 3.1 |
| **连接池耗尽** | 等待连接超时 | 事务边界过大 / OSIV / 连接池过小 | 缩小事务边界 / 关闭 OSIV / 增加连接池 | 5.1~5.3 |
| **查询计划缓存膨胀** | 内存持续增长 | IN 子句参数数量多变 / 拼接字符串查询 | 参数化查询 / in_clause_parameter_padding | 3.4 |
| **脏检查慢** | flush 耗时异常 | 大量实体 + 宽表 | 字节码增强 / @DynamicUpdate / @Immutable / 仅查 DTO | 7.3, 6.2 |
| **二级缓存命中率低** | 命中率 < 50% | 实体更新频繁 / 缓存空间不足 | 增大缓存 / 检查更新模式 / 改用查询缓存 | 10 |
| **查询缓存频繁失效** | 命中率极低 | 底层表更新频繁 | 移除查询缓存 / 仅在频繁查询、极少修改的表使用 | 10 |
| **自增 ID 耗尽连接** | 获取不到新连接 | IDENTITY 策略每次 INSERT 后立即获取连接 | 改用 SEQUENCE + allocationSize | 4.1 |
| **List @OrderColumn 索引更新风暴** | 一次 add 产生多条 UPDATE | 中间位置插入触发后续元素索引重排 | 使用 @OrderBy 替代 / 用 Set | 6.3 |
| **JOIN FETCH + 分页警告** | HHH000104 警告 | 内存分页 | 两阶段查询 / EntityGraph | 2.2 |
| **持久化上下文膨胀** | 内存持续增长直至 OOM | 未定期 clear | 批量操作后 flush+clear | 4.2 |
| **慢查询问题** | 某些查询耗时几秒 | 缺少索引 / 全表扫描 | 创建索引 / EXPLAIN 分析 / 游标分页 | 8.1, 3.3 |
| **@ManyToOne EAGER 拖慢所有查询** | 每次查 Book 都 JOIN Author | 忘记设置 LAZY | 所有 @ManyToOne 设为 LAZY | 6.1 |
| **@OneToOne LAZY 不生效** | 查 User 总是 JOIN UserProfile | 无法确定关联是否存在 | 字节码增强 / @LazyToOne(NO_PROXY) | 6.1 |
| **数据库连接泄漏** | 连接数持续增长不回收 | 未正确关闭 EntityManager / Session | try-with-resources / 连接泄漏检测 | 5.1 |

---

# 11. 调优清单（Checklist）

## 11.1 设计层（5 项）

- [ ] **1.** 聚合边界是否合理？是否避免了过大的聚合根？（DDD 聚合设计）
- [ ] **2.** 是否需要 CQRS 分离读写模型？（写用实体，读用 DTO/视图）
- [ ] **3.** 高频读取、低频写入的数据是否考虑使用缓存（Redis/Caffeine）？
- [ ] **4.** 是否设计了合适的分库分表策略？（水平/垂直拆分）
- [ ] **5.** 是否评估了数据库读写分离？（主库写、从库读）

## 11.2 映射层（9 项）

- [ ] **6.** 所有 `@ManyToOne` 是否显式设置了 `fetch = FetchType.LAZY`？（默认 EAGER！）
- [ ] **7.** 所有 `@OneToMany` 的 `mappedBy` 是否正确配置？（避免多余的外键关联表）
- [ ] **8.** 集合属性类型选择是否正确？（Set vs List vs 有 @OrderColumn 的 List）
- [ ] **9.** `@OneToOne` 的懒加载是否正常生效？是否需要 `@LazyToOne(NO_PROXY)`？
- [ ] **10.** 大字段（@Lob）是否设置了 `@Basic(fetch = LAZY)`？且已启用字节码增强？
- [ ] **11.** 宽表是否使用了 `@DynamicUpdate` / `@DynamicInsert`？
- [ ] **12.** 实体是否正确实现了 `equals()` 和 `hashCode()`？（基于业务键）
- [ ] **13.** 是否使用了 `@Immutable` 标注只读实体（字典表、历史表）？
- [ ] **14.** IDENTITY 主键策略的业务表是否考虑改为 SEQUENCE 策略以支持批处理？

## 11.3 查询层（7 项）

- [ ] **15.** 是否使用 `JOIN FETCH` 或 `EntityGraph` 解决 N+1 问题？
- [ ] **16.** 是否配置了全局 `hibernate.default_batch_fetch_size`？（建议 30）
- [ ] **17.** 列表查询是否使用了 DTO 投影？（而非加载完整实体）
- [ ] **18.** 只读查询是否设置了 `@QueryHint(readOnly = true)` 或 `@Transactional(readOnly = true)`？
- [ ] **19.** 分页查询是否考虑了游标分页（Keyset Pagination）替代深分页 OFFSET？
- [ ] **20.** 是否使用参数化查询？IN 子句是否启用了 `in_clause_parameter_padding`？
- [ ] **21.** 查询计划缓存大小是否足够？（`hibernate.query.plan_cache_max_size`，默认 2048）

## 11.4 连接与事务层（6 项）

- [ ] **22.** 连接池配置是否合理？（HikariCP 推荐 maximum-pool-size ≤ 核心数 × 2 + 磁盘数）
- [ ] **23.** 是否启用了连接泄漏检测？（`leak-detection-threshold`）
- [ ] **24.** 事务边界是否最小化？（事务不应覆盖 HTTP 请求全过程）
- [ ] **25.** 是否关闭了 OSIV？（`spring.jpa.open-in-view = false`）
- [ ] **26.** 读事务是否标注了 `@Transactional(readOnly = true)`？（配合读写分离）
- [ ] **27.** 事务超时是否合理设置？（防止长时间持有锁）

## 11.5 JDBC 层（5 项）

- [ ] **28.** 是否配置了 `hibernate.jdbc.batch_size`？（建议 20~50）
- [ ] **29.** 是否启用了 `hibernate.order_inserts` 和 `hibernate.order_updates`？
- [ ] **30.** MySQL 是否启用了 `rewriteBatchedStatements`？
- [ ] **31.** 是否启用了 `hibernate.batch_versioned_data`？
- [ ] **32.** 连接 URL 参数是否优化？（socketTimeout、connectTimeout、cachePrepStmts）

## 11.6 数据库层（4 项）

- [ ] **33.** 所有外键列是否都有索引？（尤其是 `@ManyToOne` 的 `@JoinColumn`）
- [ ] **34.** 是否使用 Flyway/Liquibase 管理 DDL？（生产环境禁用 `hbm2ddl.auto`）
- [ ] **35.** 大数据量表是否考虑分区？（如按时间分区）
- [ ] **36.** 高频聚合查询是否考虑物化视图？

## 11.7 测试与监控（4 项）

- [ ] **37.** 集成测试中是否断言了 SQL 执行次数？（检测 N+1）
- [ ] **38.** 是否启用了 Hibernate Statistics？（`generate_statistics = true`）
- [ ] **39.** 是否配置了慢查询日志？（`LOG_QUERIES_SLOWER_THAN_MS`）
- [ ] **40.** 是否集成 Micrometer + Prometheus + Grafana 进行可视化监控？

## 11.8 字节码增强（3 项）

- [ ] **41.** 是否需要字节码增强？（脏检查优化、@Basic LAZY、@LazyToOne）
- [ ] **42.** 增强方式是否合适？（构建时增强 > 运行时增强）
- [ ] **43.** 增强后是否验证了效果？（实体类实现 ManagedEntity 接口）

---

> **调优原则总结**：
> 1. **先测量，后优化**——用 Hibernate Statistics / SQL 日志 / datasource-proxy 定位瓶颈
> 2. **先设计，后编码**——合理的聚合边界和关联方向是性能的根基
> 3. **先索引，后代码**——数据库索引是最低成本的性能提升手段
> 4. **先查询优化，后缓存**——查询优化是治本，缓存是治标
> 5. **先 DTO 投影，后 JOIN FETCH**——不需要管理的实体不要加载
> 6. **先批量，后逐个**——批量操作（batch_size + flush+clear）是大数据量的基本操作
> 7. **定期 Review**——业务变化后，旧的优化策略可能成为新的瓶颈
