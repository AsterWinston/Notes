# 1. Tag 与测试选择

**Tag 是给测试打的可过滤标签**，用来回答"我只想跑快测/冒烟/某模块的测试"这类问题。JUnit 4 的 `@Category` 需要为每个分类定义一个标记接口、还得配 `Categories` Runner，笨重且不能表达组合条件；JUnit 5 的 `@Tag` 是一个字符串注解，可以标在类或方法上、可以叠加多个，并且原生支持**标签表达式**做与或非组合。

## 1.1 基本用法

```java
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

@Tag("slow")                    // 类级标签：类中所有测试继承
@Tag("integration")
class ReportExportTest {

    @Test
    void export_large_report() { ... }   // 带 slow + integration 两个标签

    @Tag("fast")                          // 方法可再叠加自己的标签
    @Test
    void export_empty_report() { ... }
}
```

**标签命名没有强制规范**，但团队内应统一一套词汇表，常见维度是速度（`fast`/`slow`）、层级（`unit`/`integration`/`e2e`）、环境（`docker`/`ci-only`）。标签是字符串，拼写错误不会报错只会静默不匹配，值得建一个常量类集中管理。

## 1.2 标签表达式

过滤时支持 `!`（非）、`&`（与）、`|`（或）和括号：

```bash
# Maven：跑所有 integration 但排除 slow
mvn test -Dgroups="integration & !slow"

# Gradle
gradle test -PincludeTags="fast | smoke"
```

## 1.3 构建工具过滤

| 工具 | 包含 | 排除 |
|------|------|------|
| Maven Surefire/Failsafe | `<groups>` 或 `-Dgroups=fast` | `<excludedGroups>` 或 `-DexcludedGroups=slow` |
| Gradle | `useJUnitPlatform { includeTags 'fast' }` | `excludeTags 'slow'` |
| Console Launcher | `--include-tag fast` | `--exclude-tag slow` |

```xml
<!-- Maven：CI 上默认排除慢测试，夜间任务再全量跑 -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.5</version>
    <configuration>
        <excludedGroups>slow</excludedGroups>
    </configuration>
</plugin>
```

```groovy
// build.gradle
test {
    useJUnitPlatform {
        includeTags 'fast'          // 只跑快测
        excludeTags 'docker'        // 排除需要 docker 环境的
    }
}
```

> **行业实践**：把标签过滤做成 Gradle task 或 Maven profile 而不是让开发者记命令行参数，例如注册 `fastTest`、`integrationTest` 两个 task 共享同一套 `includeTags` 配置。CI 流水线按阶段调用不同 task：PR 门禁跑 `fast`，合并后跑 `integration`，夜间跑全量。

# 2. 测试顺序控制

**JUnit 5 默认不承诺任何方法执行顺序**——它刻意做到"确定但不可预测"（deterministic but intentionally non-obvious），逼你写出相互独立的测试。但确实存在顺序即业务语义的场景：状态机流转（创建→支付→取消）、有先决条件的 API 链路测试。JUnit 5 为此提供 `@TestMethodOrder` + 可插拔的 `MethodOrderer`。

## 2.1 内置 Orderer

| Orderer | 排序依据 | 场景 |
|---------|---------|------|
| `MethodOrderer.OrderAnnotation` | `@Order(n)` 数值，小的先 | 显式指定步骤 |
| `MethodOrderer.MethodName` | 方法名字典序 | 名字带编号前缀（`step01_create`） |
| `MethodOrderer.DisplayName` | 显示名字典序 | 同上，按可读名排 |
| `MethodOrderer.Random` | 每次随机 | **检验测试间是否真的独立**，本地/CI 各跑一次结果应一致 |

```java
import org.junit.jupiter.api.*;

@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class OrderFlowTest {

    @Test @Order(1)
    void create_order() { ... }

    @Test @Order(2)
    void pay_order() { ... }

    @Test @Order(3) @DisplayName("超时自动取消")
    void cancel_order() { ... }
}
```

`@Order` 数值相同的按方法名二级排序；类级顺序则用 `@TestClassOrder` + `ClassOrderer`（同样有 `OrderAnnotation`/`ClassName`/`Random`）。

## 2.2 顺序依赖的代价

> **设计争议**：`@Order` 制造了测试间耦合——step2 失败时 step3 的失败信息毫无价值，而且单独重跑 step3 会因缺少前置状态直接失败。官方文档明确建议**仅当顺序是领域语义本身时才用**（如流程测试），日常单元测试应保持任意顺序可执行。一个务实的折中：流程测试用 `@TestMethodOrder` 但整个类共享状态放 `@TestInstance(PER_CLASS)` 的实例字段，并给类打 `@Tag("flow")` 与独立单测区分开。

# 3. 并行执行

JUnit 5 的并行是 **Platform 层原生能力**（不是 Surefire 的 fork 并行），粒度可以细到单个方法，配置全部走 `junit-platform.properties` 或系统属性。

## 3.1 开启与配置

```properties
# src/test/resources/junit-platform.properties

junit.jupiter.execution.parallel.enabled=true             # 总开关（默认 false）
junit.jupiter.execution.parallel.mode.default=concurrent  # 方法级默认模式
junit.jupiter.execution.parallel.mode.classes.default=concurrent  # 类级默认模式

# 并行度策略：dynamic（按 CPU 核数，默认）/ fixed / custom
junit.jupiter.execution.parallel.config.strategy=dynamic
junit.jupiter.execution.parallel.config.dynamic.factor=1  # 并行度 = factor × 核数
# junit.jupiter.execution.parallel.config.strategy=fixed
# junit.jupiter.execution.parallel.config.fixed.parallelism=4
```

两个维度正交：**`mode.default` 控制方法间是否并行，`mode.classes.default` 控制类间是否并行**，各有 `same_thread` 和 `concurrent` 两个取值。四种组合里最常用的是"类间并行、方法串行"（`mode.default=same_thread` + `mode.classes.default=concurrent`）——不同类的测试互不干扰概率高，同类方法常共享 fixture 则串行更安全。也可以在代码里按类/方法覆盖：

```java
@Execution(ExecutionMode.CONCURRENT)      // 此类的方法并行执行
class IndependentCalcTest { ... }

@Execution(ExecutionMode.SAME_THREAD)     // 个别类强制串行
class LegacyFileTest { ... }
```

## 3.2 资源锁

并行后共享资源（数据库、文件、端口）用 `@ResourceLock` 声明互斥，**本质是告诉调度器"持有同一资源 key 的测试不能同时跑"**：

```java
import org.junit.jupiter.api.parallel.ResourceLock;
import org.junit.jupiter.api.parallel.ResourceAccessMode;

@ParameterizedTest
@ValueSource(strings = {"users", "orders"})
void truncate_table(String table) { ... }

@ResourceLock(value = "DATABASE", mode = ResourceAccessMode.READ_WRITE)
@Test
void migrate_schema() { ... }        // 与所有持 DATABASE 锁的测试互斥

@ResourceLock(value = "DATABASE", mode = ResourceAccessMode.READ)
@Test
void query_only() { ... }            // READ 锁之间可以并发
```

访问模式沿用读写锁语义：`READ` 共享、`READ_WRITE` 独占。Platform 还预置了 `Resources.SYSTEM_PROPERTIES`、`SYSTEM_OUT`、`LOCALE` 等常量，改系统属性/输出流的测试应声明对应锁。

## 3.3 并行的坑

**共享可变静态状态是并行失败的头号来源。** 典型的坑与对策：

- **静态字段缓存**：`private static List<X> cache` 在方法间被并发读写。对策：状态放实例字段，配合默认的 `PER_METHOD` 生命周期每个测试一个新实例。
- **`@TestInstance(PER_CLASS)`**：实例字段被该类所有测试共享，方法并行时字段就是共享可变状态，**PER_CLASS + concurrent 是危险组合**，用了 PER_CLASS 的类建议标 `@Execution(SAME_THREAD)`。
- **时间敏感断言**：并行导致 CPU 争抢，`assertTimeout(Duration.ofMillis(50))` 这类紧阈值会随机超时。对策：阈值放宽一个数量级，或性能断言单独隔离跑。
- **端口/文件冲突**：两个测试都绑 8080 或写 `target/tmp.txt`。对策：随机端口（`new ServerSocket(0)`）、`@TempDir`（天然每测试隔离）。
- **数据库测试数据互相污染**：对策：每测试独立事务回滚（Spring `@Transactional`）、独立 schema、或 `@ResourceLock` 串行化写库测试。

> **注意**：Spring 的 `@SpringBootTest` 测试开并行要格外小心——上下文缓存（`spring.test.context.cache.maxSize` 默认 32）在并发下可能反复重建上下文，`@MockBean` 会污染缓存 key，并行收益常常抵不过上下文重建的开销。经验做法是：**纯单元测试开并行，Spring 集成测试保持串行或仅类级并行**。

## 3.4 原生并行与 Surefire 并行的取舍

| 维度 | JUnit Platform 原生并行 | Surefire forkCount/parallel | Gradle maxParallelForks |
|------|------------------------|----------------------------|------------------------|
| 粒度 | 可到单个方法 | JVM 内线程或独立 JVM | 独立 JVM 进程 |
| 隔离性 | 共享堆内存，靠 ResourceLock | fork 模式下内存完全隔离 | 进程级隔离 |
| 开销 | 最低（线程级） | fork 每进程一次 JVM 启动 | 同左 |
| 静态状态冲突 | 会冲突 | fork 隔离后不冲突 | 不冲突 |
| 适用 | 大量轻量单测提速 | 依赖重、需要内存隔离 | 同 Surefire fork |

两者可以叠加：Surefire fork 出 N 个 JVM，每个 JVM 内再开 Platform 并行。**测试数量大且互相独立时用原生线程并行收益最高；测试带重型依赖（本地缓存、静态单例改不动）时用进程 fork 隔离更省心**。

# 4. 小结

| 主题 | 要点 |
|------|------|
| Tag | 类/方法可叠加，构建工具 `groups`/`excludeTags` 过滤，支持 `& \| ! ()` 表达式 |
| 顺序 | 默认无序；`@TestMethodOrder` + 四种 Orderer；仅在顺序即领域语义时使用 |
| 并行开关 | `junit.jupiter.execution.parallel.enabled=true`，方法/类两个维度独立配置 |
| 并行度 | dynamic（factor × 核数，默认）/ fixed / custom |
| 资源互斥 | `@ResourceLock` + READ/READ_WRITE 读写语义 |
| 最大的坑 | 共享静态状态、PER_CLASS + concurrent、时间阈值、端口文件冲突 |
| 与构建工具并行 | 原生=线程级低开销，Surefire/Gradle fork=进程级强隔离，可叠加 |
