# 1. 为什么数据库是第一个落地点

数据库访问层是集成测试收益最高的地方。这一层的代码几乎不做逻辑判断，它的价值在于"生成的 SQL 能否被真实数据库执行、事务边界是否符合预期、迁移脚本能否从零建库"。用 Mock 验证 `JdbcTemplate` 被调用过，等于什么都没验证；用 H2 验证 PostgreSQL 方言，则是用近似值证明精确值。

## 1.1 真实数据库能暴露的问题

- 方言差异：`ON CONFLICT`、`RETURNING`、`JSONB`、窗口函数、`ILIKE`；
- 类型映射：`timestamptz`、`numeric` 精度、`json` 与 `jsonb` 的读写；
- 约束行为：唯一键冲突的异常类型、外键级联、`CHECK` 约束；
- 事务与锁：隔离级别、行锁等待、死锁检测；
- 迁移脚本：Flyway/Liquibase 从空库执行是否成功；
- 字符集与排序：中文、emoji、大小写敏感排序。

# 2. PostgreSQL 容器

## 2.1 最小示例

```java
@Testcontainers
class PostgresSmokeTest {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16-alpine");

    @Test
    void should_expose_connection_info() {
        // 容器已启动，直接读取连接信息
        System.out.println(postgres.getJdbcUrl());
        System.out.println(postgres.getUsername());
    }
}
```

默认库名、用户名、密码都是 `test`，端口 `5432`。默认镜像标签会随模块版本变化，**生产项目应显式指定镜像版本**，避免升级后数据库大版本悄悄改变。

## 2.2 常用配置

```java
@Container
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
    .withDatabaseName("orders")          // 初始化数据库名
    .withUsername("app")                 // 业务账号
    .withPassword("app_secret")          // 业务密码
    .withInitScript("db/schema.sql")     // 启动后执行初始化脚本
    .withCommand("postgres", "-c", "max_connections=200");
```

| 方法 | 作用 | 注意 |
|------|------|------|
| `withDatabaseName` | 指定库名 | 影响 `getJdbcUrl()` |
| `withUsername` / `withPassword` | 指定账号 | 不要与生产账号混用 |
| `withInitScript` | 执行 classpath 下的 SQL | 单文件，失败会导致启动失败 |
| `withCopyFileToContainer` | 复制文件进容器 | 适合多个初始化脚本 |
| `withStartupTimeout` | 调整启动超时 | 慢机器或大镜像时有用 |

## 2.3 初始化脚本

### 2.3.1 用 withInitScript

```java
@Container
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
    .withDatabaseName("orders")
    .withUsername("app")
    .withPassword("app_secret")
    .withInitScript("db/schema.sql");
```

```sql
-- src/test/resources/db/schema.sql
CREATE TABLE customer (
    id         BIGSERIAL PRIMARY KEY,
    name       VARCHAR(64) NOT NULL,
    email      VARCHAR(128) NOT NULL UNIQUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE orders (
    id          BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL REFERENCES customer (id),
    amount      NUMERIC(12, 2) NOT NULL CHECK (amount > 0),
    status      VARCHAR(16) NOT NULL DEFAULT 'CREATED'
);

INSERT INTO customer (name, email) VALUES ('张三', 'zhangsan@example.com');
```

**注意**：`withInitScript` 只接受一个脚本，执行时机固定在容器启动之后。脚本里不要包含 `CREATE DATABASE`，它会与镜像入口脚本冲突。

### 2.3.2 挂载多个脚本

多脚本场景把文件挂到 `/docker-entrypoint-initdb.d/`，镜像会在**数据目录为空**时按文件名字典序执行 `.sql`、`.sql.gz`、`.sh`：

```java
.withCopyFileToContainer(
    MountableFile.forClasspathResource("db/01_schema.sql"),
    "/docker-entrypoint-initdb.d/01_schema.sql")
.withCopyFileToContainer(
    MountableFile.forClasspathResource("db/02_data.sql"),
    "/docker-entrypoint-initdb.d/02_data.sql");
```

所以 `01_`、`02_` 这样的前缀比 `schema`、`data` 更可靠。

> **行业实践**：测试库的 schema 最好由与生产相同的迁移工具生成，而不是另写一份手写脚本。两份脚本会逐渐分叉，最终测试通过但生产迁移失败。

# 3. MySQL 容器

## 3.1 最小示例

```java
@Container
static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.4")
    .withDatabaseName("orders")
    .withUsername("app")
    .withPassword("app_secret")
    .withCommand("--character-set-server=utf8mb4",
                 "--collation-server=utf8mb4_0900_ai_ci");
```

JDBC URL 通常还要追加参数：`mysql.getJdbcUrl() + "?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC&characterEncoding=utf8"`。MySQL 8 的认证插件与字符编码都需要显式声明，但这是测试便利配置，不要照抄到生产。默认端口 `3306`，默认库名、账号同样是 `test`。

## 3.2 与 PostgreSQL 的差异

| 维度 | PostgreSQLContainer | MySQLContainer |
|------|---------------------|----------------|
| 默认端口 | 5432 | 3306 |
| 自增主键 | `BIGSERIAL` / `GENERATED` | `AUTO_INCREMENT` |
| 标识符大小写 | 未加引号折叠为小写 | 取决于 `lower_case_table_names` |
| 时区 | 建议 `TIMESTAMPTZ` | 注意 `serverTimezone` |
| 字符集 | 默认 UTF-8 | 需显式 `utf8mb4` |
| 初始化目录 | `/docker-entrypoint-initdb.d/` | 同名目录，行为一致 |

## 3.3 字符集与时区

MySQL 的 `utf8` 实际是三字节的 `utf8mb3`，存 emoji 会失败，必须用 `utf8mb4`。时区问题则常在断言时间字段时暴露：容器默认 UTC，应用可能配置了 `Asia/Shanghai`。用 `SHOW VARIABLES LIKE 'character_set_server'` 与 `SHOW timezone` 确认服务端设置。

> **注意**：要么统一用 UTC 存储、展示层转换，要么在容器里设置 `TZ` 并保证 JDBC 连接串一致。两边不一致时，测试会稳定差 8 小时。

# 4. 连接信息与数据隔离

## 4.1 JDBC URL 从哪里来

容器端口会映射到宿主机的**随机端口**，测试里不要写死 `localhost:5432`。`getHost()` 返回宿主机地址，`getMappedPort(5432)` 返回随机端口，`getJdbcUrl()` 已把两者与库名拼好，`getUsername()` 与 `getPassword()` 返回容器账号。

## 4.2 同一容器内的清理

`@Container` 声明为 static 时，容器在类级别启动一次、停止一次，类内测试共享数据。最常用的是每个方法前清空表：

```java
@BeforeEach
void cleanUp() {
    jdbcTemplate.execute("TRUNCATE TABLE orders, customer RESTART IDENTITY CASCADE");
}
```

`RESTART IDENTITY` 重置自增序列，`CASCADE` 处理外键依赖。也可以用 `@Sql` 声明式执行清理脚本，或把 `@Container` 放在实例字段上换取方法级隔离。后者隔离最彻底，但每个方法都要启动容器，只适合极少数破坏性测试。

> **取舍**：static 还是实例字段，本质是"启动时间"与"隔离强度"的权衡。绝大多数项目应选 static 加清理脚本，把隔离交给 SQL 而不是容器。

## 4.3 事务回滚的边界

Spring Test 的 `@Transactional` 会在测试结束后回滚，但它只对**测试线程内、同一个 DataSource** 的操作有效。被测代码使用 `REQUIRES_NEW`、异步线程或消息消费时，回滚覆盖不到；DDL 在多数数据库里会隐式提交，也不会回滚。

**设计考量**：`@Transactional` 适合"只读验证 + 单线程写入"的仓储测试。一旦涉及事务传播、异步或消息，就应改用显式清理，否则会出现难以复现的脏数据。

# 5. 完整测试示例

## 5.1 被测代码

```java
@Repository
public class OrderRepository {
    private final JdbcTemplate jdbcTemplate;

    public OrderRepository(JdbcTemplate jdbcTemplate) { this.jdbcTemplate = jdbcTemplate; }

    public long insert(long customerId, BigDecimal amount) {
        // RETURNING 是 PostgreSQL 特性，H2 兼容模式下行为不同
        return jdbcTemplate.queryForObject(
            "INSERT INTO orders (customer_id, amount) VALUES (?, ?) RETURNING id",
            Long.class, customerId, amount);
    }
}
```

## 5.2 测试代码

```java
@Testcontainers
class OrderRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("orders")
        .withUsername("app")
        .withPassword("app_secret")
        .withInitScript("db/schema.sql");

    static JdbcTemplate jdbcTemplate;

    @BeforeAll
    static void setUp() {
        DriverManagerDataSource dataSource = new DriverManagerDataSource();
        dataSource.setUrl(postgres.getJdbcUrl());
        dataSource.setUsername(postgres.getUsername());
        dataSource.setPassword(postgres.getPassword());
        jdbcTemplate = new JdbcTemplate(dataSource);
    }

    @BeforeEach
    void clean() {
        jdbcTemplate.execute("TRUNCATE TABLE orders RESTART IDENTITY CASCADE");
    }

    @Test
    void should_return_generated_id() {
        long id = new OrderRepository(jdbcTemplate).insert(1L, new BigDecimal("99.90"));

        assertThat(id).isPositive();
        Integer count = jdbcTemplate.queryForObject(
            "SELECT count(*) FROM orders WHERE id = ?", Integer.class, id);
        assertThat(count).isEqualTo(1);
    }
}
```

**注意**：清理只针对 `orders`，`schema.sql` 预置的 `customer.id = 1` 会保留，正好作为外键依赖。若把 `customer` 也一起清空，就需要在清理脚本里重新插入基础数据。

## 5.3 用 Flyway 初始化

```java
@BeforeAll
static void migrate() {
    Flyway.configure()
        .dataSource(postgres.getJdbcUrl(), postgres.getUsername(), postgres.getPassword())
        .locations("classpath:db/migration")
        .load()
        .migrate();
}
```

让 Flyway 在空库上执行同一套迁移，**顺带验证了迁移脚本本身**，比手写 schema 更接近生产。

# 6. JDBC URL 模式

Testcontainers 支持在 JDBC URL 里直接声明容器，需要 `org.testcontainers:jdbc` 在测试类路径上：

```properties
# src/test/resources/application-test.properties
spring.datasource.url=jdbc:tc:postgresql:16:///orders
```

| 优点 | 限制 |
|------|------|
| 配置极简，只改一行 URL | 初始化脚本能力有限 |
| 对 Spring Boot 零侵入 | 生命周期由驱动管理，调试不直观 |
| 适合快速试点 | 难以与其他容器共享网络 |

URL 里可以带 `TC_INITSCRIPT`、`TC_IMAGE_TAG` 等参数，但语义不如显式容器清晰。**行业实践**：把 `jdbc:tc` 当作快速验证手段，正式落地改用显式容器加动态属性或 `@ServiceConnection`。

# 7. 常见错误

| 现象 | 根因 | 处理 |
|------|------|------|
| `Connection refused` | 容器未就绪就建连接 | 用模块自带等待策略，自定义容器加 `Wait` |
| 初始化脚本没执行 | 路径不在 classpath | 确认 `src/test/resources` 下有该文件 |
| 表已存在导致启动失败 | 初始化目录被重复执行 | 每次新建容器，不要挂持久卷 |
| 中文乱码 | MySQL 字符集不是 `utf8mb4` | 启动参数显式设置字符集 |
| 时间断言差 8 小时 | 容器 UTC 与应用时区不一致 | 统一 UTC 或设置 `TZ` |
| 自增 ID 在测试间重复 | 只 `DELETE` 未重置序列 | 用 `TRUNCATE ... RESTART IDENTITY` |

# 8. 练习与自检

**练习一**：分别用 PostgreSQL 和 MySQL 容器执行同一条带 `RETURNING` 的插入语句，记录哪个支持、哪个报错，并改为各自的等价写法。

**练习二**：写两个测试类共享同一个镜像，验证"A 类插入的数据在 B 类不可见"，理解 static 容器的作用域。

**练习三**：用 Flyway 在空容器上执行两条迁移，故意制造一个与现有数据冲突的迁移，观察失败发生在哪个阶段。

**自检清单**：

- 能说出 `getJdbcUrl()` 为什么不能写死 `localhost:5432`；
- 知道 `withInitScript` 与挂载初始化目录的区别；
- 能解释 `@Transactional` 回滚在什么情况下失效；
- 能根据启动时间与隔离需求选择 static 或实例容器。

# 9. 小结

| 主题 | 核心结论 |
|------|---------|
| PostgreSQL | 用 `PostgreSQLContainer`，显式指定镜像版本 |
| MySQL | 注意 `utf8mb4`、时区与认证参数 |
| 初始化 | 单脚本用 `withInitScript`，多脚本挂初始化目录 |
| 连接 | 一律使用容器返回的随机端口与账号 |
| 隔离 | static 容器 + 方法前清理最实用，事务回滚有边界 |
| 迁移 | 让 Flyway/Liquibase 在测试里跑真实脚本 |

数据库打通之后，下一个高频依赖是消息队列与缓存。下一篇进入 Kafka 与 Redis，重点讲等待就绪、生产者消费者测试，以及容器端口与网络的处理方式。
