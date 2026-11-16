# 1. 数据库测试策略

数据库是集成测试里最"重"的依赖：真库慢且有副作用，假库快但可能与生产行为不一致。数据库测试的所有工程决策，本质都在回答一个问题——**这层测试想验证什么，愿意为"真实性"付多少成本**。

## 1.1 选型维度

| 方案 | 优点 | 缺点 | 适用 |
|------|------|------|------|
| 内嵌数据库（H2/HSQLDB/Derby） | 快、轻量、零运维、进程内启动 | 方言与生产库有差异（函数、锁、约束行为） | Repository 单元层验证、`@DataJpaTest` 默认 |
| 专用测试库（独立 MySQL/PG 实例） | 与生产同引擎，方言一致 | 共享实例易互相污染、维护成本高、CI 需预置 | 团队级集成测试环境 |
| 容器化数据库（Testcontainers） | 与生产完全一致、每次测试全新实例、CI 友好 | 需要 Docker、首次拉镜像慢、单容器秒级启动 | 集成测试主流选择 |

在不使用外部容器的前提下，推荐：

- 单元测试不依赖数据库（把持久化 Mock/Fake 掉）
- Repository 测试使用内嵌数据库
- 集成测试使用专用测试库或 Testcontainers

**内嵌库最大的风险是"绿灯幻觉"**：H2 的 MySQL 兼容模式能跑通大部分 SQL，但窗口函数边界、隐式类型转换、`ON DUPLICATE KEY` 细节、事务隔离默认值都可能与真 MySQL 不同——H2 上全绿的原生 SQL，到生产库可能报错或结果不同。原则：**只有纯 JPQL/Criteria 的测试可以放心用 H2；含 `nativeQuery = true` 的 SQL 必须对真库方言验证一遍**。

```text
            真实性
              ▲
              │   Testcontainers（真引擎、一次性实例）
              │
              │   专用测试库（真引擎、共享实例）
              │
              │   H2 兼容模式（像真引擎）
              │
              │   H2 原生模式 / Mock Repository
              └──────────────────────────────▶ 速度与隔离性
```

---

# 2. `@DataJpaTest`

## 2.1 特性

`@DataJpaTest` 是 JPA 层切片测试：只装配 `EntityManager`、Spring Data Repository、事务管理器等持久化相关 Bean，Service/Controller 一概不加载。默认行为三件套：

- **自动替换数据源为内嵌库**（有 H2/HSQLDB/Derby 在 classpath 时）
- **每个测试方法默认包在事务里并回滚**（`@Transactional` 语义，见第 13 篇）
- **提供 `TestEntityManager`**：`EntityManager` 的测试特化包装

```java
@DataJpaTest
class UserRepositoryTest {

    @Autowired
    private TestEntityManager em;      // persistAndFlush / find / clear 的便捷封装

    @Autowired
    private UserRepository repository; // 切片自动扫描到的 Repository

    @Test
    void should_find_by_email() {
        em.persistAndFlush(new User(null, "tom@example.com"));  // 直接落库（事务内）

        User found = repository.findByEmail("tom@example.com").orElseThrow();
        assertThat(found.getEmail()).isEqualTo("tom@example.com");
    }
}
```

`TestEntityManager` 与 `EntityManager` 的分工：前者省略了事务样板（`persistAndFlush` = persist + flush），并**故意不提供 commit**——切片事务由测试框架管理，测试代码不应该自己提交。需要完整 JPA API（如 `createQuery`）时仍注入 `EntityManager`。

其他常用属性：

```java
@DataJpaTest(
    showSql = false,                          // 关掉默认的 SQL 打印（默认 true，日志很吵）
    properties = "spring.jpa.hibernate.ddl-auto=create"  // 切片级属性覆盖
)
class QuietJpaTest { }
```

## 2.2 实体扫描

切片从 `@SpringBootConfiguration` 所在包向下扫描 `@Entity`。若实体不在默认扫描路径（多模块项目常见），显式指定：

```java
@DataJpaTest
@EntityScan("com.example.domain")        // 实体在另一个模块/包
class EntityScanTest { }

@DataJpaTest
@EnableJpaRepositories("com.example.infra.repo")  // Repository 不在默认路径时同理
class RepoScanTest { }
```

> **注意**：`@DataJpaTest` **会执行 Flyway/Liquibase 迁移**——两者的自动配置在切片的 imports 清单里。只要迁移工具在 classpath 且配置了位置，Schema 就由迁移脚本建出来，且 Boot 检测到迁移工具后会让 Hibernate 的 `ddl-auto` 靠边（不再自动建表）。想让切片退回纯 Hibernate 建表，用 `@TestPropertySource(properties = "spring.flyway.enabled=false")` 显式关闭。

---

# 3. `@AutoConfigureTestDatabase`

## 3.1 替换与不替换

切片测试默认**把真实数据源替换为内嵌库**，防止测试误连开发/生产数据库。需要对着真库（或 Testcontainers）跑时，关掉替换：

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
// 不替换 → 使用 application-test.yml / @DynamicPropertySource 提供的真实数据源
class RealDatabaseRepositoryIT { }
```

`Replace` 的三个取值：

| 取值 | 行为 |
|------|------|
| `ANY`（默认） | 无条件替换为内嵌库 |
| `EMBEDDED` | 仅当原数据源指向内嵌库时才替换（几乎等于 ANY） |
| `NONE` | 不替换，用环境里配置的真实数据源 |

与 Testcontainers 组合的标准形态：

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = Replace.NONE)   // 用容器里的真 PostgreSQL
@Testcontainers
class OrderRepositoryIT {

    @Container
    @ServiceConnection                                // Boot 3.1+ 自动注册 url/账号
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16");

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void should_support_native_jsonb_query() {
        orderRepository.save(new Order(null, "{\"channel\":\"app\"}"));
        // 原生 SQL 用到 jsonb 操作符——H2 上测不出来的部分
        assertThat(orderRepository.findByChannelNative("app")).hasSize(1);
    }
}
```

> **行业实践**：容器复用加速。每个测试类起一个 Postgres 容器约 2-5 秒，套件大了受不了。Testcontainers 的 singleton 模式（容器声明为静态字段、不调用 `stop`、跨类共享）或 `withReuse(true)` 配合 `~/.testcontainers.properties` 的 `testcontainers.reuse.enable=true`，可以把整个套件压到只启动一次容器。代价是容器内数据跨类残留，需要配合每类清表或事务回滚。

## 3.2 全量测试中的数据源

`@SpringBootTest` 不做任何替换——数据源就是 Environment 里配置的那个。所以全量集成测试连什么库完全由 `application-test.yml` + `@DynamicPropertySource` 决定，这也是为什么测试 profile 里**必须**显式写死测试库地址，绝不依赖"本地默认配置"。

---

# 4. 数据库迁移与初始化

## 4.1 Flyway / Liquibase

迁移工具是测试 Schema 的**首选来源**：测试跑的就是生产同款迁移脚本，Schema 漂移（"测试库少个索引/字段"）从根上消失。

- 测试容器每次全新 → 每次全量重放迁移 → 顺带验证了迁移脚本本身可执行
- 迁移脚本含方言 SQL 时，测试库必须是真方言（回到第 3 节：`Replace.NONE` + Testcontainers）
- 想加快重放，Flyway 可配 `spring.flyway.clean-disabled=false` + 测试专用 clean，但更常见的是接受几秒开销

**版本化迁移在测试中的典型问题**是"开发分支合并后迁移版本号冲突"（两个分支各写了 V12）。Flyway 会报 checksum/版本冲突，测试环境的处理与生产一致——重命名迁移文件，而不是在测试里 `flyway.repair()` 掩盖。

## 4.2 DDL 策略

`spring.jpa.hibernate.ddl-auto` 在测试语境下的取值权衡：

| 取值 | 测试场景 | 说明 |
|------|----------|------|
| `create-drop` | 内嵌库、无迁移工具 | 每次启动重建 Schema，干净但脱离生产 |
| `validate` | 有 Flyway/Liquibase | **推荐**：Schema 由迁移建，Hibernate 只校验实体映射与表结构一致 |
| `update` | 都不推荐 | 生产禁用；测试里它会掩盖实体与迁移脚本的不一致 |
| `none` | 特殊场景 | 完全交给外部 |

生产跑 `validate` 的项目，测试也应该跑 `validate`——这样"实体改了字段但忘写迁移脚本"的错误会在测试阶段就炸出来，而不是上线时。

## 4.3 `@Sql` 脚本初始化

迁移解决 Schema，`@Sql` 解决**每个测试的数据前置**：

```java
@DataJpaTest
@Sql(scripts = "/sql/seed-users.sql")                    // 方法前执行（默认 BEFORE_TEST_METHOD）
class UserQueryTest {

    @Test
    void should_count_active_users() {
        assertThat(userRepository.countByActiveTrue()).isEqualTo(3);  // 种子数据决定的期望值
    }

    @Test
    @Sql(scripts = "/sql/cleanup.sql",
         executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)  // 方法后清理
    @Transactional(propagation = Propagation.NOT_SUPPORTED)      // 提交模式才需要清理
    void should_update_user_committing() { }
}
```

`@Sql` 关键属性：

- `executionPhase`：`BEFORE_TEST_METHOD`（默认）/ `AFTER_TEST_METHOD`
- `statements`：内联 SQL，小数据量不必建文件
- `config`：指定脚本的事务传播与错误处理（`SqlConfig`）
- 脚本默认**加入测试事务**，随事务回滚——所以在回滚型测试里不需要清理脚本；只有 `@Commit`/无事务测试才需要 AFTER 清理

初始化手段的分工：Schema 交给迁移或 `schema.sql`（`spring.sql.init.schema-locations`），少量固定种子交给 `data.sql` 或 `@Sql`，**每个测试独有的数据用代码构造**（`repository.save(...)` 或 `TestDataBuilder`），可读性最好、重构最安全。

---

# 5. Repository 测试实践

## 5.1 Flush 与断言

JPA 有持久化上下文（一级缓存），`save` 之后 SQL 可能还没发出去，直接断言会测到缓存而非数据库：

```java
User saved = repo.save(user);
em.flush();                       // 把挂起的 SQL 真正发给数据库（触发约束校验）
em.clear();                       // 清空一级缓存，下次查询走数据库
User loaded = repo.findById(saved.getId()).orElseThrow();
assertThat(loaded.getName()).isEqualTo("tom");   // 这才是"数据库里"的值
```

通过 `flush/clear` 避免一级缓存影响断言。**`flush` 还有一个常被忽略的价值：唯一约束、非空约束等数据库层校验发生在 flush 时**——想验证"重复邮箱会抛 `DataIntegrityViolationException`"，不 flush 就永远看不到异常。`@DataJpaTest` 里用 `TestEntityManager.persistAndFlush` 一步到位。

## 5.2 分页与排序

分页与排序是 Repository 测试的重点验证对象——参数拼错、排序字段与业务不一致，编译器完全查不出来：

```java
@Test
void should_page_by_created_desc() {
    IntStream.range(0, 25).forEach(i ->
            repo.save(new Order(null, "O-" + i, Instant.now().plusSeconds(i))));
    em.flush();

    Page<Order> page = repo.findAll(
            PageRequest.of(1, 10, Sort.by(DESC, "createdAt")));  // 第 2 页、每页 10 条

    assertThat(page.getContent()).hasSize(10);
    assertThat(page.getTotalElements()).isEqualTo(25);
    assertThat(page.getTotalPages()).isEqualTo(3);
    assertThat(page.getContent().get(0).getCode()).isEqualTo("O-14"); // 降序第 11 名
}
```

派生查询（`findByXxxAndYyy`）的方法名解析错误在启动期就会报 `PropertyReferenceException`，所以**只要上下文能启动，派生查询的字段名就是对的**；需要测的是查询语义（大小写、null 处理、关联抓取）与自定义 `@Query` 的正确性。

## 5.3 Repository 测试与全链路测试的分工

| 维度 | `@DataJpaTest`（切片） | `@SpringBootTest`（全链路） |
|------|------------------------|------------------------------|
| 验证对象 | 单个 Repository 的查询语义 | 事务、缓存、服务编排、真实 SQL 的协同 |
| 启动成本 | 秒级，只装配 JPA 层 | 全量容器 |
| 数据库 | 默认 H2，可切真库 | 由测试配置决定 |
| 数量配比 | 多（每个复杂查询一个） | 少（关键业务流各一条冒烟） |

> **行业实践**：分层配比大致是"每个含 `@Query`/派生查询的 Repository 一个切片测试类 + 每条核心业务流一个全链路测试"。把查询正确性验证全塞进全链路测试，会让"改一个查询"要跑整条业务流才能发现错误；反过来全链路测试不可省——事务边界、二级缓存、多 Repository 协作只有它能覆盖。

---

# 6. 性能与稳定性

- **避免在测试中依赖真实网络数据库**：共享开发库做测试是 flaky 的温床——别人删了数据、改了表，你的测试就红。网络抖动还会把断言失败和连接失败混在一起。
- **保持数据清理策略一致**（回滚或清表）：同一套件里一半测试靠事务回滚、一半靠 `DELETE` 清表，出问题时没人说得清库里的数据是谁留下的。切片测试统一用默认回滚，提交型测试统一用 `@Sql` AFTER 清理。
- **测试间不要共享可变数据状态**：种子数据要么每测试自建，要么设计成只读；"测试 A 插入的数据被测试 B 断言依赖"是顺序耦合，并行化时必炸。
- **约束自增主键的坑**：H2 与 MySQL 的自增起点、序列缓存行为不同，断言里不要硬编码 `id = 1`，用 `saved.getId()` 或按业务字段查回。
- **连接池尺寸**：测试里把 HikariCP 缩到 2-4 个连接（`spring.datasource.hikari.maximum-pool-size`），并行测试时避免打爆容器数据库的连接上限。

# 7. 小结

| 主题 | 关键结论 |
|------|---------|
| 选型 | 纯 JPQL 用 H2 切片即可；含原生 SQL 必须对真方言（Testcontainers + `Replace.NONE`） |
| `@DataJpaTest` | 只装配 JPA 层、默认内嵌库、默认事务回滚、提供 `TestEntityManager` |
| `@AutoConfigureTestDatabase` | `Replace.NONE` 关闭内嵌库替换，是接真库/容器的开关 |
| Schema 来源 | 首选 Flyway/Liquibase 重放生产迁移；`ddl-auto=validate` 校验实体映射 |
| 数据前置 | 固定种子用 `@Sql`，测试专属数据用代码构造；回滚型测试无需清理脚本 |
| 断言纪律 | `flush` 触发真实 SQL 与约束校验，`clear` 排除一级缓存干扰 |
| 分工 | 查询语义在切片测，事务与编排在少量全链路测 |

下一篇进入事务与回滚：解释"测试为什么默认回滚、回滚让哪些行为测不到、什么时候必须提交"。
