# 1. ApplicationContext 缓存机制

上下文缓存是 Spring 测试性能的第一支柱：**一个中大型 Boot 应用的容器启动动辄 5-30 秒，500 个测试类各起一次容器 CI 要以小时计；有了缓存，配置相同的测试类共享同一个容器，全套件可能只启动三五次。** 大部分"测试越来越慢"的问题，根因都不是单个测试慢，而是缓存没命中导致容器反复重建。本篇量化缓存收益、拆解 key 构成、讲清 `@DirtiesContext` 的语义与代价，并给出避免滥用的工程策略。

## 1.1 为什么需要缓存

**启动 Spring Boot 容器是一次昂贵的一次性投资，成本集中在四处**：自动配置扫描与条件评估（Boot 3.x 候选自动配置类 140+ 个，每个都要做 `@Conditional` 评估）；Bean 实例化与依赖装配（几百上千个 Bean 的构造、注入、`BeanPostProcessor` 链）；配置属性绑定（`@ConfigurationProperties` 的宽松绑定与校验）；基础设施初始化（Web 容器、数据源连接池、JPA 的 EntityManagerFactory——扫描实体建元模型单独就可能要几秒）。

缓存 `ApplicationContext` 把这笔投资从"每测试类一次"摊薄成"每配置指纹一次"。收益可以粗略量化：

| 场景 | 容器启动次数 | 套件总耗时（估算，单容器启动 10s，500 个测试类、每类平均 1s） |
|------|------------|--------------------------------------------------------|
| 无缓存（假想） | 500 | 500×(10+1)s ≈ 92 分钟 |
| 配置收敛到 3 种 | 3 | 3×10s + 500×1s ≈ 11 分钟 |
| 配置发散到 50 种 | 50 | 50×10s + 500×1s ≈ 17 分钟 |

> **注意**：缓存的另一个隐性收益是**并行友好**——JUnit 5 并行执行时，共享同一缓存容器的测试类不会重复付出启动成本；而缓存 miss 风暴（大量类同时未命中）会造成多个容器并发启动，内存峰值飙升甚至 OOM。

## 1.2 缓存 Key 的组成

**TestContext 使用 `MergedContextConfiguration`（MCC）作为缓存 Key，它是测试类全部上下文注解"压平合并"后的不可变对象，其 `equals`/`hashCode` 决定命中与否。** 参与 key 计算的关键组成：

- **配置源**：`@ContextConfiguration` 的 locations/classes、`@SpringBootTest(classes = ...)`；Boot 场景下还包括"向上查找到的 `@SpringBootConfiguration`"（同一主配置查出来是同一个类，通常不构成差异）
- **激活的 profiles**：`@ActiveProfiles`（含继承与合并结果）
- **属性源**：`@TestPropertySource` 的文件与内联 properties、`@SpringBootTest(properties = ...)`、`args`
- **`@DynamicPropertySource` 的动态值**：方法注册进来的每个属性都进 key——Testcontainers 随机端口若直接进属性，会导致每个测试类一个容器（正确解法见下）
- **上下文初始化器**：`@ContextConfiguration(initializers = ...)` 列出的 `ApplicationContextInitializer` 集合
- **ContextLoader 类型与 Web 环境**：`webEnvironment`（MOCK/RANDOM_PORT/NONE/DEFINED_PORT）——同一组配置类，MOCK 与 RANDOM_PORT 是两个不同的容器
- **父子层级**：`@ContextHierarchy` 声明的层级结构
- **Bean 覆盖**：`@MockitoBean`/`@MockitoSpyBean` 定义的替身集合（类型 + 名字 + 限定符）——**最常见的 key 发散源**：Mock 字段的任何增减组合都产生新 key

**结论：哪怕是单个属性的差异，也会导致新容器。** 判断两个测试类能否共享容器，等价于问"它们的全部上述配置逐项相等吗"。

工程上让 key 收敛的手段按优先级：

```java
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;

// 手段一：公共属性收敛到统一入口，而不是每类各写 properties
// application-test.yml（src/test/resources）承载所有共享测试配置
@SpringBootTest
@ActiveProfiles("test") // 全部容器型测试统一这一个 profile
class SomeIntegrationTest { }

// 手段二：Testcontainers 端口用 @ServiceConnection / @DynamicPropertySource
// 但要保证"同一个容器实例"被多个测试类共享，属性值才稳定
@SpringBootTest
class OrderRepoIT {
    // static + 单例容器：JVM 级共享，端口固定，动态属性值一致 → key 相同
    static final PostgreSQLContainer<?> POSTGRES = SharedContainers.POSTGRES;

    @DynamicPropertySource
    static void datasourceProps(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", POSTGRES::getJdbcUrl); // 值跨类稳定
    }
}
```

手段三是 **Mock 组合收敛**：把常见的替身集合定义成共享基类（如 `abstract class WithPaymentMockTest`，内含统一的 `@MockitoBean PaymentGateway` 字段），需要相同 Mock 的测试类都继承它——字段定义一致，key 才一致。

> **行业实践**：`@ServiceConnection`（Boot 3.1+）配合 Testcontainers 的 `static` 单例容器是目前最省心的做法——连接属性由 Boot 自动注册且值稳定，不需要每个测试类手写 `@DynamicPropertySource`，天然避免了"注册方法不同导致 key 差异"的坑。

## 1.3 缓存大小与淘汰

**`ContextCache` 默认上限 32 个上下文，采用 LRU 淘汰：超过上限时最久未使用的容器被 `close()` 销毁，之后再用它的测试类会触发重建。**

- 上限通过系统属性 `spring.test.context.cache.maxSize` 调整；设为 0 或负数表示不限制（危险，见下）
- 每个缓存的容器都持有真实资源：数据源连接池、线程池、（RANDOM_PORT 时）一个监听端口。**32 个全量 Boot 容器可能就是十几 GB 的堆占用**——上限的存在首先是内存保护，不是性能考虑
- 缓存是 JVM 级的，surefire fork 出的每个 JVM 各有一份；fork 之间不共享
- 观察缓存行为：日志级别调 `org.springframework.test.context.cache=DEBUG`，Spring 会输出缓存统计（hit/miss 计数）；调 `maxSize` 前先看清 miss 模式

**大量不同配置的测试类会导致缓存频繁失效**：50 个不同 key、上限 32，意味着至少 18 次淘汰 + 重建，且重建顺序取决于执行顺序——套件耗时变得不可预测。这种情况下调大 maxSize 只是把问题从"时间"转移到"内存"，正解仍是收敛配置种类。

---

# 2. `@DirtiesContext` 的语义

## 2.1 作用与代价

**`@DirtiesContext` 表示当前测试会污染容器状态，声明框架在执行前/后把上下文标记为"脏"——脏上下文从缓存移除并 `close()`，后续需要它的测试类重新构建。** 它不是"清理数据"，而是"销毁重建整个容器"，代价 = 一次完整的容器启动 + 被淘汰容器持有的资源释放。

什么才算"污染容器"？判断标准是：**修改了单例 Bean 的内部状态且无法在测试内还原**。典型场景：往单例的内存缓存/注册表里塞了数据且该 Bean 没有清理接口；通过反射或 setter 改了配置类 Bean 的字段值；测试触发了单例 Bean 的状态机迁移（如熔断器打开），影响后续测试观察到的行为。

反过来，**这些不是脏化，不需要 `@DirtiesContext`**：数据库里的测试数据（事务回滚或 `@Sql` 清理即可）、Mock 的桩配置（框架每个测试方法后自动重置）、`RequestContextHolder` 之类的线程绑定状态（监听器自动清理）。"数据脏了"和"容器脏了"是两回事——前者清数据，后者才重建容器，混淆这两者是 `@DirtiesContext` 被滥用的主要原因。

## 2.2 常见使用方式

**类级与方法级两个维度，各有 BEFORE/AFTER 系列模式：**

```java
import org.junit.jupiter.api.Test;
import org.springframework.test.annotation.DirtiesContext;
import org.springframework.test.annotation.DirtiesContext.ClassMode;
import org.springframework.test.annotation.DirtiesContext.MethodMode;

// 类级：整个测试类执行完后销毁上下文（最常用的形态）
@DirtiesContext(classMode = ClassMode.AFTER_CLASS)
class DirtyContextTest {

    // 方法级：单个方法执行前销毁（保证该方法拿到全新容器）
    @DirtiesContext(methodMode = MethodMode.BEFORE_METHOD)
    @Test
    void needs_pristine_context() {
        // 该测试直接操作了单例的内部状态，无法用还原逻辑清理
    }
}
```

| 级别 | 模式 | 说明 |
|------|------|------|
| classMode | `BEFORE_CLASS` | 类执行前销毁并重建（本类拿到干净容器） |
| classMode | `AFTER_CLASS` | 类执行后销毁（保护后续测试类，类级默认值） |
| classMode | `BEFORE_EACH_TEST_METHOD` | 每个方法前销毁重建 |
| classMode | `AFTER_EACH_TEST_METHOD` | 每个方法后销毁重建 |
| methodMode | `BEFORE_METHOD` | 该方法执行前销毁 |
| methodMode | `AFTER_METHOD` | 该方法执行后销毁（方法级默认值） |

执行机制上，`DirtiesContextBeforeModesTestExecutionListener` 处理 BEFORE 系列，`DirtiesContextTestExecutionListener` 处理 AFTER 系列（第 4 篇监听器表）。

**BEFORE 与 AFTER 的选择有讲究**：`AFTER_*` 是"我脏了，别连累别人"——脏测试类自己仍复用旧容器，成本由后续重建者承担；`BEFORE_*` 是"我要干净的起点"——成本由声明者自己承担。污染者用 AFTER，对前置状态敏感的昂贵测试用 BEFORE。**`*_EACH_TEST_METHOD` 是最重的模式**：容器启动次数 = 测试方法数，只应出现在"每个方法都必然破坏容器"的极端场景。

> **设计争议**：有观点认为 `@DirtiesContext` 应该彻底禁用（部分团队在 CI 里 grep 它并 fail build），理由是"需要重建容器说明测试设计失败——有状态单例本就该被替换成替身"。另一派认为在遗留系统里它是务实的止血带。折中立场：**新代码禁止，存量代码保留但登记在案**，把每个 `@DirtiesContext` 视为一张技术债工单。

---

# 3. Context Hierarchy 与缓存

## 3.1 父子上下文

**`@ContextHierarchy` 声明多层上下文，父容器承载公共 Bean、子容器承载测试专属配置——父子各自独立缓存，多个子上下文可以共享同一个父。**

```java
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.ContextHierarchy;

@ContextHierarchy({
    @ContextConfiguration(classes = InfraConfig.class),   // 父：数据源、消息基础设施
    @ContextConfiguration(classes = BillingConfig.class)  // 子：被测业务模块
})
class BillingServiceTest {
}
```

```text
ParentContext（InfraConfig，独立缓存，可被多个子配置共享）
  ├── ChildContext-A（BillingConfig）  ← BillingServiceTest
  └── ChildContext-B（OrderConfig）    ← OrderServiceTest
```

缓存语义：MCC 里记录父级的 MCC，**父上下文命中缓存则直接复用，只有子级重建**——N 个业务模块的测试各自建轻量子容器，重的基础设施容器只建一次。子容器还能"看见"父容器的 Bean（标准 Spring 父子语义），Bean 覆盖也遵循"子优先"。

## 3.2 适用场景

- **多个测试模块共享公共基础设施**：数据源、缓存、消息连接这类"启动最贵"的部分收敛到父级
- **需要隔离业务模块与基础模块**：子上下文只含被测模块的 Bean，比全量容器小得多、启动快得多
- **同一测试类内验证跨层交互**：模拟"网关上下文 + 业务上下文"这类多容器架构

> **注意**：Boot 的 `@SpringBootTest` 全量容器场景很少用 hierarchy——自动配置本身就是一体的，拆父子反而破坏装配假设。hierarchy 主要服务于纯 Spring（`@ContextConfiguration`）测试与多模块 XML/JavaConfig 分层架构；Boot 项目达到同样目的的更常见手段是"共享 Testcontainers 单例 + 收敛 profile"。

---

# 4. 避免过度 DirtiesContext 的策略

## 4.1 使用事务回滚

**数据层污染的正解是事务回滚，不是容器重建：`@Transactional` 测试默认在每个方法后回滚，数据从不落库，自然谈不上脏。** `@DataJpaTest`、`@JdbcTest` 已隐含 `@Transactional`；`@SpringBootTest` 需要显式加。需要真实提交时用 `@Commit` 或 `@Rollback(false)` 并自行清理。事务细节与"测试里 save 的数据外部连接看不到"的经典困惑见第 13 篇。

## 4.2 使用测试切片

**切片容器更小，污染的爆炸半径也更小**：`@WebMvcTest` 的容器里根本没有数据源与业务单例，能被测试破坏的状态天然有限；即使真需要重建，重建一个切片容器的成本也远低于全量容器。"把测试降级到能覆盖断言的最小切片"同时是性能策略与稳定性策略。

## 4.3 使用替身与 TestConfiguration

**对有状态 Bean，替换比清理更可靠**——测试中不修改真实 Bean 的状态，而是让容器里从一开始就是无状态或可重置的替身：

```java
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.test.context.bean.override.mockito.MockitoBean;

@SpringBootTest
class ExternalFlowTest {

    @MockitoBean // Boot 3.4+；旧版本用 @MockBean。替身每方法后自动重置，无状态残留
    private ExternalClient externalClient;

    @TestConfiguration(proxyBeanMethods = false)
    static class FakeCounterConfig {
        @Bean
        public Counter counter() {
            return new ResettableCounter(); // Fake 实现带 reset()，@AfterEach 调用
        }
    }
}
```

更根本的方案是**设计侧消除可变单例状态**：把计数、缓存、注册表这类状态移出 Bean 字段（交给 Redis/Caffeine 等外部化缓存，测试里可整体清空），容器就没有可脏之处。

## 4.4 决策树

```text
测试之间互相影响了？
   │
   ├─ 影响的是数据库数据 ────────▶ @Transactional 回滚 / @Sql 清理
   ├─ 影响的是 Mock 桩 ──────────▶ 什么都不用做（框架自动重置）
   ├─ 影响的是单例 Bean 内部状态
   │     ├─ 能加 reset 接口/换 Fake ──▶ @TestConfiguration 替换
   │     ├─ 能整体 Mock ─────────────▶ @MockitoBean 替换
   │     └─ 都不行 ──────────────────▶ @DirtiesContext（最后手段）
   └─ 影响的是 JVM 静态状态 ──────▶ 修代码，容器重建救不了 static
```

---

# 5. 性能优化建议

## 5.1 最小化上下文

- **`@SpringBootTest(classes = MinimalTestConfig.class)`**：显式指定最小配置集，绕过全量自动配置查找——适合"只需要三五个 Bean"的场景
- **移除不必要的 `@ComponentScan`**：宽泛的包扫描会把无关 Bean 全拉进来；用 `@Import` 精确引入
- **排除重量级自动配置**：`properties = "spring.autoconfigure.exclude=..."` 排除定时任务、消息监听、actuator 等与断言无关的装配（注意：排除会改变 MCC，别每个类排一套）

## 5.2 合理使用 Profile

- **`@ActiveProfiles("test")` 全工程统一一个测试 profile**：既收敛缓存 key，又把测试专用配置（内存库地址、关邮件、短超时）集中管理
- **限定数据源与外部依赖**：test profile 下数据源指向 Testcontainers/内嵌库、外部 URL 指向 WireMock，避免误连真实环境
- **不要为每个测试类开专属 profile**：profile 进缓存 key，一个类一个 profile 等于一个类一个容器

## 5.3 延迟初始化

**`spring.main.lazy-initialization=true` 让容器启动时只注册 Bean 定义、不实例化，未用到的 Bean 永不初始化**——对"每个测试类只用容器一小角"的套件提速明显：

```yaml
# application-test.yml：只在测试 profile 下开启延迟初始化
spring:
  main:
    lazy-initialization: true
```

代价两条：**首次访问变慢**（初始化成本从启动期摊到使用期，测单个方法时感知为"第一次断言前卡一下"）；**装配错误延迟暴露**——Bean 依赖缺失、循环依赖这类问题在生产是启动即炸，在 lazy 模式下要等到第一次用到才炸，"上下文能启动"这个全量测试的核心价值被削弱。折中：Boot 的 `LazyInitializationExcludeFilter` 可排除关键 Bean；或补一个 `contextLoads()` 测试显式触发全量装配验证。

> **行业实践**：性能优化的动手顺序应该是——①先开 `org.springframework.test.context.cache=DEBUG` 看缓存命中统计，确认是"miss 太多"还是"单容器太慢"；②miss 多 → 收敛配置（1.2 的手段）；③单容器慢 → 切片降级 + lazy-initialization + 排除自动配置；④还慢 → 考虑并行执行与增量测试。直接调 `maxSize` 是最常被做错的一步。

---

# 6. 小结

| 主题 | 关键结论 |
|------|---------|
| 缓存收益 | 500 测试类无缓存 ≈ 92 分钟，收敛到 3 种配置 ≈ 11 分钟；缓存是 Spring 测试性能第一支柱 |
| 缓存 key | MCC = 配置类 + profiles + properties/args + 动态属性 + 初始化器 + webEnvironment + 层级 + Mock 替身集合，逐项相等才命中 |
| key 发散源 | 每类各写 properties、`@DynamicPropertySource` 值不稳定、`@MockitoBean` 组合随意 |
| 缓存上限 | 默认 32、LRU 淘汰，`spring.test.context.cache.maxSize` 可调；上限首先是内存保护 |
| @DirtiesContext | 销毁重建整个容器，只用于"单例内部状态不可还原"；数据脏用回滚/清理 |
| classMode/methodMode | AFTER 系列由污染者声明、BEFORE 系列由敏感者声明；EACH_TEST_METHOD 系列最重 |
| Context Hierarchy | 父容器独立缓存可共享，适合公共基础设施 + 多业务模块；Boot 全量场景少用 |
| 优化顺序 | 先看缓存统计定位 miss/慢，再收敛配置、切片降级、lazy-initialization，最后才 maxSize 与并行 |

下一篇深入 `@SpringBootTest` 注解本身：webEnvironment 四种模式的机制差异、args/properties/classes 属性与自动配置装载过程。
