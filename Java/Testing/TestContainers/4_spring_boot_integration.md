# 1. 集成的核心问题

Testcontainers 单独使用时，测试代码自己创建 `DataSource`、自己拼连接串。放进 Spring Boot 之后，问题变成：**如何把容器在运行时才知道的地址和端口注入到自动配置里**。容器端口是随机的，`application.yml` 不可能写死。

Spring Boot 3.x 提供了两条路：通用但手工的 `@DynamicPropertySource`，以及 3.1 之后引入的声明式 `@ServiceConnection`。理解差异比记住模板更重要。

# 2. @DynamicPropertySource

## 2.1 基本用法

```java
@SpringBootTest
@Testcontainers
class UserServiceTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("app")
        .withUsername("app")
        .withPassword("app_secret");
    @DynamicPropertySource
    static void datasourceProps(DynamicPropertyRegistry registry) {
        // 传入方法引用而不是值，属性在第一次读取时才求值
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}
```

注册的是 `Supplier`，不是字符串。这样能保证属性被读取时容器已经启动、随机端口已经确定。

## 2.2 为什么必须是 static

`@DynamicPropertySource` 在 Spring 上下文创建阶段处理，早于任何测试实例的创建。方法不是 static，Spring 无法在没有实例的情况下调用它。

**设计考量**：这个约束反过来暗示了容器生命周期——容器必须在 Spring 上下文创建之前存在。所以配套的 `@Container` 字段也应是 static，由 JUnit 扩展在类级别启动。实例字段此时还是 null，注册属性必然失败。

## 2.3 一次注册多个容器

```java
@DynamicPropertySource
static void registerAll(DynamicPropertyRegistry registry) {
    registry.add("spring.datasource.url", postgres::getJdbcUrl);
    registry.add("spring.datasource.username", postgres::getUsername);
    registry.add("spring.datasource.password", postgres::getPassword);
    registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    registry.add("spring.data.redis.host", redis::getHost);
    registry.add("spring.data.redis.port", () -> redis.getMappedPort(6379));
}
```

`host` 没有随机性可以直接给值，`port` 必须用 lambda 延迟求值。所有非字符串类型同样要用 `Supplier` 包装。

| 组件 | 属性键 | 来源 |
|------|--------|------|
| JDBC | `spring.datasource.url` | `getJdbcUrl()` |
| Kafka | `spring.kafka.bootstrap-servers` | `getBootstrapServers()` |
| Redis | `spring.data.redis.host` / `port` | `getHost()` / `getMappedPort(6379)` |
| RabbitMQ | `spring.rabbitmq.host` / `port` | `getHost()` / `getAmqpPort()` |

**注意**：Boot 3.x 的 Redis 前缀是 `spring.data.redis.*`，不是 2.x 的 `spring.redis.*`。键名写错不会报"未知配置"，只会静默连默认的 `localhost:6379`。

# 3. @ServiceConnection

## 3.1 依赖与启用

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
```

把它直接标在容器字段上，Spring Boot 会从容器类型推导连接信息：

```java
@SpringBootTest
@Testcontainers
class UserServiceTest {
    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16-alpine");
    @Autowired
    private UserRepository userRepository;
}
```

`PostgreSQLContainer` 与 `JdbcConnectionDetails` 有内置映射，`DataSource` 会自动配置，不再需要 `spring.datasource.*`。

## 3.2 Redis 与 Kafka

没有专用容器类的组件，Spring Boot 会按镜像名匹配连接工厂；镜像名不典型时，也可以显式指定连接名：

```java
@Container
@ServiceConnection
static GenericContainer<?> redis =
    new GenericContainer<>("redis:7.4-alpine").withExposedPorts(6379);
@Container
@ServiceConnection
static KafkaContainer kafka =
    new KafkaContainer(DockerImageName.parse("apache/kafka:3.7.0"));
```

## 3.3 匹配规则与对比

匹配大体分三类：按容器类（`PostgreSQLContainer`）、按镜像名（`redis:*`、`mongo:*`）、按显式名称（`@ServiceConnection("redis")`）。

| 维度 | `@DynamicPropertySource` | `@ServiceConnection` |
|------|--------------------------|----------------------|
| 引入版本 | Spring 5.2+ | Spring Boot 3.1+ |
| 写法 | 手写属性键 | 注解声明 |
| 适用容器 | 任意 | 有连接工厂的组件 |
| 灵活性 | 高，可注入任意属性 | 中，围绕标准连接信息 |
| 出错概率 | 键名、端口容易写错 | 较低 |

> **行业实践**：能用 `@ServiceConnection` 就用它，属性键少、升级时跟随 Boot 约定。只有需要注册非标准属性时才回到 `@DynamicPropertySource`。

# 4. @Testcontainers 的生命周期

## 4.1 注解与字段规则

`@Testcontainers` 是 JUnit 5 扩展，扫描被 `@Container` 标注的字段并负责启动停止。字段必须是 `Container` 或 `Startable` 类型，且不要用 `private`。

| 声明方式 | 启动时机 | 适用 |
|----------|---------|------|
| `static` | 类中所有测试之前启动一次 | 大多数集成测试 |
| 非 static | 每个测试方法前启动、后停止 | 强隔离、极少数场景 |

实例字段隔离最好，但每个方法都要启动容器。绝大多数项目应选 static 加数据清理，把隔离成本从"启动容器"降到"执行一条 TRUNCATE"。

## 4.2 无 Docker 时跳过

```java
@Testcontainers(disabledWithoutDocker = true)
class DockerOptionalTest {
    // 无 Docker 时整个类被跳过，而不是失败
}
```

**注意**：跳过不等于通过。CI 里如果所有容器测试都被跳过，流水线显示绿色，但集成覆盖为零。应显式检查 Docker 可用性，而不是依赖这个开关。

# 5. 上下文复用

## 5.1 TestContext 缓存键

Spring Test 会缓存 `ApplicationContext`，缓存键由配置类、profile、属性源、上下文定制器、`@DynamicPropertySource` 方法等共同决定。**键相同才复用**。这解释了一个常见困惑：两个类的容器配置完全一样，上下文却启动了两次——因为它们的动态属性方法是两个不同的方法。

## 5.2 用基类共享配置

把容器与属性注册放进抽象基类，子类继承，缓存键就一致了：

```java
@SpringBootTest
@Testcontainers
public abstract class AbstractIntegrationTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("app")
        .withUsername("app")
        .withPassword("app_secret")
        .withInitScript("db/schema.sql");
    @DynamicPropertySource
    static void datasourceProps(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}

class UserRepositoryTest extends AbstractIntegrationTest {
    // 与 OrderRepositoryTest 共享上下文和容器
}
```

共享上下文意味着共享容器、共享数据，子类之间必须约定清理策略，否则测试顺序会影响结果。

## 5.3 单例容器模式

容器也可以在静态块里启动，整个 JVM 只启动一次：

```java
public final class PostgresSingleton {
    public static final PostgreSQLContainer<?> INSTANCE =
        new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("app")
            .withUsername("app")
            .withPassword("app_secret");
    static {
        INSTANCE.start();   // 类加载时启动，不主动 stop
    }
}
```

容器不主动停止，由 Ryuk 在 JVM 退出时兜底清理。适合多个模块共享一套数据库的大型项目，但容器状态在整个 JVM 内累积，清理必须严格。另需注意 `@DirtiesContext`：它会让上下文在测试后关闭并重建，static 容器虽不重启，连接池与 Flyway 仍会重新初始化，能靠清理数据解决的问题不要用它。

# 6. 测试切片中的使用

## 6.1 @DataJpaTest

`@DataJpaTest` 默认把 `DataSource` 替换成内嵌数据库，要连真实容器必须关掉替换：

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class UserRepositorySliceTest {
    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");
    @Autowired
    private UserRepository userRepository;
}
```

切片只加载 JPA 相关 Bean，启动比 `@SpringBootTest` 快，适合只验证仓储层。

## 6.2 @WebMvcTest 不需要数据库

`@WebMvcTest` 只加载 Web 层，Service 用 `@MockBean` 替换。这种测试不需要数据库容器，也不应该启动一个——那只是白白增加启动时间。容器只在真正需要它的切片里启动。

# 7. 完整示例

```java
@SpringBootTest
@Testcontainers
class AccountServiceIntegrationTest {
    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withInitScript("db/schema.sql");
    @Container
    @ServiceConnection
    static KafkaContainer kafka =
        new KafkaContainer(DockerImageName.parse("apache/kafka:3.7.0"));
    @Autowired
    private AccountService accountService;
    @Autowired
    private JdbcTemplate jdbcTemplate;
    @BeforeEach
    void clean() {
        jdbcTemplate.execute("TRUNCATE TABLE account RESTART IDENTITY CASCADE");
    }

    @Test
    void should_transfer() {
        accountService.create("A", new BigDecimal("100.00"));
        accountService.create("B", new BigDecimal("0.00"));
        accountService.transfer("A", "B", new BigDecimal("30.00"));
        BigDecimal balanceA = jdbcTemplate.queryForObject(
            "SELECT balance FROM account WHERE owner = 'A'", BigDecimal.class);
        assertThat(balanceA).isEqualByComparingTo("70.00");
    }
}
```

两个 `@ServiceConnection` 分别配置 `DataSource` 和 Kafka 客户端，测试代码里没有一行连接串。

# 8. 常见错误

| 现象 | 根因 | 处理 |
|------|------|------|
| "must be static" | 方法或容器字段不是 static | 两者都改为 static |
| 仍然连 `localhost:5432` | 属性键拼错或被其他配置覆盖 | 检查 `spring.datasource.url` |
| 上下文反复重建 | 每个类各写一份动态属性方法 | 抽基类或单例容器 |
| `@ServiceConnection` 不生效 | 缺依赖或 Boot 版本低于 3.1 | 补 `spring-boot-testcontainers` |
| `@DataJpaTest` 还是连 H2 | 未关闭自动替换 | 加 `@AutoConfigureTestDatabase(replace = NONE)` |
| 测试之间数据串味 | 共享上下文且无清理 | 每方法清理或改用独立容器 |
| 容器字段为 null | 实例字段，上下文先于测试实例创建 | 改为 static |

# 9. 练习与自检

**练习一**：分别用 `@DynamicPropertySource` 和 `@ServiceConnection` 连接同一个 PostgreSQL 容器，对比两种写法的行数与出错概率。

**练习二**：写两个测试类继承同一个抽象基类，在日志里确认 Spring 上下文只启动一次；再让其中一个类自己声明容器，观察上下文启动次数变化。

**练习三**：把 `@DataJpaTest` 连接到 Testcontainers 的 PostgreSQL，验证自增主键和 `ON CONFLICT` 行为，确认它不再使用 H2。

**自检清单**：

- 能解释 `@DynamicPropertySource` 为什么必须 static、为什么用 `Supplier`；
- 知道 `@ServiceConnection` 的适用版本与匹配方式；
- 能说清上下文缓存键由哪些因素决定；
- 能判断一个测试类该用基类共享还是独立容器。

# 10. 小结

| 主题 | 核心结论 |
|------|---------|
| DynamicPropertySource | 通用、灵活，静态方法 + 方法引用 |
| ServiceConnection | Boot 3.1+ 声明式注入，属性零手写 |
| Testcontainers | static 字段类级启动，实例字段方法级启动 |
| 上下文复用 | 缓存键一致才复用，基类或单例是常用手段 |
| 切片测试 | `@DataJpaTest` 需关闭内嵌库替换，`@WebMvcTest` 不需要容器 |
| 清理 | 共享上下文必须配严格的数据清理 |

容器能启动、能注入，接下来就是"启动几次、什么时候停"。下一篇深入生命周期与复用，讲 static 容器的边界、reuse 模式、并行测试和资源清理。
