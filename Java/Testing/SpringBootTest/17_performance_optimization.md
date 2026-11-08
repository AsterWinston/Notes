# 1. 性能瓶颈来源

Spring 测试套件变慢几乎从不是"某个测试写得慢"，而是**结构性问题**：容器起了太多次、每次起得太重、测试粒度选错。优化前先用一句话定位瓶颈类型——是"上下文数量多"还是"单个上下文重"还是"测试本身慢"，三者的解法完全不同。

## 1.1 上下文启动成本

一次 `@SpringBootTest` 全量启动的耗时构成：

- **自动配置类扫描与条件评估**：Boot 3.x 约 140+ 个 AutoConfiguration 候选，每个都要跑 `@Conditional` 判定
- **Bean 实例化与依赖注入**：组件扫描出的每个 Bean 的创建、代理织入（AOP/事务）、后置处理
- **数据源初始化**：连接池预热、Flyway 迁移重放、Hibernate Schema 校验/建表、元数据加载
- Web 环境下还有嵌入式容器（Tomcat/Netty）启动

量级感受：一个中型业务应用全量上下文 5-20 秒，切片 1-3 秒，纯 Jupiter + Mockito 测试 <10ms。**上下文启动成本只有在缓存未命中时才支付**，所以第 2 节的一切手段都围绕"少建容器、建轻容器"。

## 1.2 测试粒度过大

将所有测试都写成 `@SpringBootTest` 是最常见的性能陷阱。判断一个测试该用什么粒度，只看它**真正需要容器里的什么**：

```text
 粒度决策树
┌──────────────────────────────────────────────┐
│ 测试类里有 @Autowired 吗？                      │
│   ├─ 没有 → 纯 Jupiter + Mockito（毫秒级）      │
│   └─ 有 → 需要什么 Bean？                       │
│        ├─ 只要 Web 层 → @WebMvcTest             │
│        ├─ 只要 JPA 层 → @DataJpaTest            │
│        ├─ 只要 JSON/RestClient → @JsonTest 等    │
│        └─ 跨层协同/完整链路 → @SpringBootTest     │
└──────────────────────────────────────────────┘
```

一个健康的套件形态是金字塔：**底部大量纯单测（不进容器），中部切片测试（轻容器、高缓存命中），顶部少量全量集成测试**。全量测试的数量应该以"核心业务流各一条冒烟"为限。

---

# 2. 降低启动成本的方法

## 2.1 优先使用测试切片

- `@WebMvcTest` / `@DataJpaTest` 等只加载本层 Bean，自动配置也被限制到相关子集
- 切片的上下文**在同配置测试类之间天然共享**（同一 Controller 集合、同一属性）
- 切片外依赖用 `@MockitoBean` 补齐——但注意 Mock 组合参与缓存 Key，**Mock 字段集合尽量在测试类间保持一致**

## 2.2 限定配置类

全量测试不必加载整个应用：`classes` 显式指定最小配置集，跳过组件扫描：

```java
@SpringBootTest(classes = {
        OrderService.class,           // 被测服务
        OrderRepository.class,        // 它的依赖
        DataSourceConfig.class        // 基础设施配置
})
@ActiveProfiles("test")
class OrderServiceIT {
    // 没有组件扫描、没有自动配置全家桶——但也要自己补齐所需 Bean
}
```

这是"迷你容器"路线：比切片灵活、比全量轻。代价是配置类清单要手工维护，漏一个 Bean 就启动失败。

## 2.3 排除自动配置

上下文里用不到的重型自动配置（数据源、安全、Kafka、Actuator）可以排除：

```java
// 方式一：注解级（仅影响当前测试）
@SpringBootTest(properties =
    "spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration")
class NoDbTest { }

// 方式二：测试 profile 级（影响整个测试环境，推荐收敛到这里）
```

```yaml
# src/test/resources/application-test.yml
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.kafka.KafkaAutoConfiguration
      - org.springframework.boot.autoconfigure.mail.MailSenderAutoConfiguration
```

排除的判据是 `-Ddebug` 条件报告里"匹配了但测试根本用不到"的重型配置。**注意别排除了被测对象依赖的配置**——排除 Mail 自动配置后又测邮件服务，会得到一个"Bean 缺失"的假故障。

> **设计争议**：`spring.main.lazy-initialization=true` 与排除自动配置是同一目标（少干活）的两种手段，但懒加载有隐性代价：**装配错误从启动期推迟到首次访问**。测试里某个 Bean 依赖缺失，急切初始化会在上下文启动时立刻炸（错误定位清晰），懒加载则可能在测试中途甚至根本不炸（该 Bean 恰好没被用到）——"测试全绿但应用起不来"的事故源。行业实践是：**测试环境可以开懒加载提速，但要保留至少一个不开懒加载的全量冒烟测试**兜住装配正确性。

## 2.4 延迟初始化

```yaml
spring:
  main:
    lazy-initialization: true    # 测试 profile 下开启：未被用到的 Bean 不初始化
```

效果与风险见上面的设计争议。逐 Bean 精细控制可用 `@Lazy`，但对测试提速意义不大，全局开关 + 冒烟兜底是更实用的组合。

## 2.5 上下文缓存最大化

**最贵的容器是"本可以不建"的容器。** 缓存 Key（`MergedContextConfiguration`）由配置类、profiles、内联属性、Mock 集合等共同决定，任何一项不同就是新容器。收敛手段：

| 发散源 | 收敛做法 |
|--------|----------|
| 各类各写 `properties` | 公共属性下沉 `application-test.yml`，个性属性才留注解 |
| `@MockitoBean` 组合五花八门 | 固定几种 Mock 套餐（做成自定义组合注解） |
| `@ActiveProfiles` 随意组合 | 统一 "test"，特殊需求用属性覆盖而非新 profile |
| `@Import` 零散 | 共享测试配置抽成一个 `IntegrationTestConfig` |
| `@DirtiesContext` 成片出现 | 修掉污染源，别销毁容器 |

验证收敛效果：`logging.level.org.springframework.test.context.cache=DEBUG`，看"Retrieved cached ApplicationContext"与"Storing new ApplicationContext"的比例。一个 500 测试类的工程，上下文从 50 个收敛到 5 个，CI 时间差一个数量级。

---

# 3. 并行测试

## 3.1 JUnit 5 并行配置

```properties
# junit-platform.properties（src/test/resources）
junit.jupiter.execution.parallel.enabled=true
junit.jupiter.execution.parallel.mode.default=same_thread      # 方法级默认串行
junit.jupiter.execution.parallel.mode.classes.default=concurrent  # 类级并行
junit.jupiter.execution.parallel.config.strategy=dynamic       # 按 CPU 核数
```

**类间并行、方法串行**是 Spring 项目最稳的起点：同一测试类的方法往往共享容器状态与测试数据，方法级并行容易自相残杀；类间只要上下文可共享、无独占资源，并行收益已经很大。

```java
@Isolated                          // 独占执行：该类跑时其他测试全部暂停
class GlobalStateTest { }

@ResourceLock("database")          // 资源锁：与其他标记同锁的测试互斥
class SchemaMigrationTest { }
```

## 3.2 与上下文缓存的交互

并行执行与 Spring 上下文缓存是**兼容但微妙**的组合：

- 上下文加载有锁保护：两个线程同时首次需要同一配置时，只有一个执行加载，另一个等待后复用——不会重复建容器；
- **不同配置的容器并行存在**：内存占用 = 并行度 × 平均上下文大小，500MB 一个的容器开 8 路并行就是 4GB，CI 容器内存上限要先算这笔账；
- 上下文缓存有 LRU 上限（默认 32，`spring.test.context.cache.maxSize` 可调）：并行 + 配置发散时可能触发淘汰，被淘汰的容器下次访问重建，反而更慢；
- `@DirtiesContext` 在并行下的销毁-重建窗口内，其他线程可能正在用这个上下文——**并行套件里应彻底禁用 `@DirtiesContext`**。

## 3.3 构建工具层的并行

JUnit 并行之外，构建工具还有第二层并行——**模块间并行**（多模块项目各模块的测试任务并行跑）：

```xml
<!-- Maven：Surefire forkCount 并行（进程级，隔离性好于线程级） -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <forkCount>1C</forkCount>          <!-- 每核一个 JVM fork -->
        <reuseForks>true</reuseForks>      <!-- fork 复用，摊薄 JVM 启动成本 -->
    </configuration>
</plugin>
```

```kotlin
// Gradle：maxParallelForks 进程级并行
tasks.test {
    useJUnitPlatform()
    maxParallelForks = (Runtime.getRuntime().availableProcessors() / 2).coerceAtLeast(1)
}
```

**进程级并行（fork）与线程级并行（JUnit parallel）不要同时激进**：4 个 fork × 每 fork 8 线程 = 32 路并发打同一个数据库，连接池和 Testcontainers 先撑不住。常见搭配是 fork 并行 + fork 内串行，或单 fork + JUnit 类间并行。

## 3.4 风险点

- **共享静态状态**：静态字段、单例里的可变状态、`SecurityContextHolder` 的线程模式；
- **共享外部资源**（端口、文件、数据库）：固定端口必炸（用 `RANDOM_PORT`）；共享库上的 `TRUNCATE` 型清理会删掉别人正在用的数据（并行提交型测试要么独占库，要么保持串行）；
- **顺序假设**：并行下"上一个测试留下的数据"不再存在。

> **行业实践**：并行化改造的正确顺序是——先把上下文收敛到个位数、把提交型测试隔离干净，**再**开并行；反过来先开并行会收获一片 flaky。改造后连续跑 10 次全绿才算稳。

---

# 4. 稳定性与可重复性

性能优化的前提是测试结果可信，可重复性规则在并行语境下加倍重要。

## 4.1 避免依赖测试顺序

- JUnit 5 默认**不保证方法顺序**（MethodOrderer 默认确定性但非声明顺序）；
- 如需顺序，应显式使用 `@TestMethodOrder(OrderAnnotation.class)` + `@Order`——但更好的动作是**消除顺序依赖本身**：每个测试自建数据、自清理，谁先跑都绿；
- 类级顺序依赖（A 类跑完 B 类才能跑）在并行下直接非法，重构是唯一出路。

## 4.2 清理策略

- 数据库使用事务回滚或 `@Sql` 清理（第 13 篇）；提交型测试在并行下需要资源锁或独占容器；
- 文件系统使用 `@TempDir`（JUnit 5 每测试独立临时目录，自动删除）；
- 静态/单例状态在 `@AfterEach` 还原，或改造成实例级状态。

## 4.3 时间与时区

依赖系统时间的测试（月末结算、过期判断）在特定日期必挂，属于"可重复性"范畴的慢病——注入 `Clock` 解决（第 15 篇 2.3），CI 里另加 `user.timezone` 显式固定，避免构建机时区漂移。

---

# 5. 测试启动耗时分析

优化前先测量，Spring 生态的测量工具从粗到细：

## 5.1 条件评估报告 `-Ddebug`

```bash
mvn test -Dtest=OrderServiceIT -Ddebug
# 输出 ConditionEvaluationReport：每个自动配置 matched/negative matches 及原因
```

用途不是看耗时，而是**找到"根本不需要却加载了"的重型配置**，作为 2.3 排除清单的依据。

## 5.2 上下文缓存统计

```properties
logging.level.org.springframework.test.context.cache=DEBUG
```

```text
# 日志里的关键行
Retrieved cached ApplicationContext for [MergedContextConfiguration...]  ← 命中
Storing new ApplicationContext (...) in cache                            ← 新建（贵！）
Spring test ApplicationContext cache statistics:
  [ContextCache@... size = 7, maxSize = 32]                              ← 套件结束时的总结
```

`size` 就是本套件实际建过的容器数——**这是性能优化的第一 KPI**。每次大重构后看一眼这个数字有没有回弹。

## 5.3 启动步骤耗时（startup 端点与 BufferingApplicationStartup）

Boot 2.4+ 提供 `ApplicationStartup` 追踪启动每一步的耗时：

```java
// 测试专用启动器：记录启动步骤到缓冲区
SpringApplication app = new SpringApplication(Application.class);
app.setApplicationStartup(new BufferingApplicationStartup(4096));
ConfigurableApplicationContext ctx = app.run("--spring.profiles.active=test");

// 导出最慢的步骤
StartupTimeline timeline = ((BufferingApplicationStartup) app.getApplicationStartup()).getBufferedTimeline();
timeline.getEvents().stream()
        .sorted(Comparator.comparing(e -> e.getDuration().toMillis(), Comparator.reverseOrder()))
        .limit(10)
        .forEach(e -> System.out.printf("%6d ms  %s%n", e.getDuration().toMillis(), e.getStartupStep().getName()));
```

生产/CI 环境也可以开 `startup` 端点（`management.endpoints.web.exposure.include=startup`）拉同样的数据。典型发现：Bean 实例化大头在连接池预热、Flyway、Jackson 模块注册——然后针对性地懒加载或排除。

## 5.4 套件级测量

- **Maven**：`-Dsurefire.reportFormat=plain` 看每类耗时；或 Surefire XML 报告聚合分析；
- **Gradle**：`gradle test --scan`（Develocity）给出每测试耗时火焰图；本地用 `buildscan` 或测试报告 HTML；
- 简单粗暴：`@EnabledIfSystemProperty` 或直接看 CI 的分阶段耗时。找出 top 10 慢测试类，逐个问"能不能降级成切片/纯单测"。

---

# 6. 实践建议

- **将慢测试标记为 `@Tag("slow")`**：Testcontainers 集成测试、端到端冒烟都打上标签；

```java
@Tag("slow")                 // 类级标签，构建工具按标签过滤
@SpringBootTest(webEnvironment = RANDOM_PORT)
class FullStackIT { }
```

```bash
mvn test -Dgroups='!slow'            # 日常开发：排除慢测试
mvn verify -Dgroups=slow             # 提交前/夜间：只跑慢测试
```

```kotlin
// Gradle 按标签分任务
tasks.test { useJUnitPlatform { excludeTags("slow") } }
tasks.register<Test>("slowTest") { useJUnitPlatform { includeTags("slow") } }
```

- **在 CI 中单独执行慢测试**：PR 流水线只跑快套件（目标 <5 分钟），main 合并后或定时任务跑全量（第 18 篇的分层策略）；
- **保持测试数量与反馈时间平衡**：本地全量反馈超过 10 分钟，开发者就会开始跳过测试——比没有测试更糟。反馈时间是测试架构的硬指标；
- **性能预算进 CI**：给"上下文数量""快套件耗时"设阈值，超了让流水线黄/红，防止优化成果被后来的提交悄悄吃掉。

# 7. 小结

| 手段 | 层级 | 收益 | 风险/代价 |
|------|------|------|-----------|
| 粒度降级（全量→切片→纯单测） | 单测试 | 最大（10-1000 倍） | 覆盖面收窄，需分层互补 |
| 上下文配置收敛 | 套件 | 大（容器数 → 个位数） | 需要纪律与组合注解 |
| 排除自动配置 / 迷你配置类 | 单上下文 | 中 | 手工维护清单，可能漏 Bean |
| 懒加载 | 单上下文 | 中 | 装配错误延迟暴露，留冒烟兜底 |
| JUnit 类间并行 | 套件 | 中-大 | 共享状态/资源需先治理 |
| fork 并行（Surefire/Gradle） | 构建 | 中 | 内存与外部资源翻倍压力 |
| `@DirtiesContext` 治理 | 套件 | 中 | —— 它是纯成本，应清零 |
| 测量（cache DEBUG / startup / -Ddebug） | 诊断 | 前置条件 | 无 |

下一篇进入 CI 构建与排错：分层流水线、Surefire/Failsafe、容器化测试环境与上下文加载失败的排查流程。
