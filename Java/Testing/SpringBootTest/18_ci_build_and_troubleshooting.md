# 1. CI 测试分层与命名

CI 里跑 Spring 测试的核心矛盾是**反馈速度与覆盖面**：PR 流水线要几分钟内给出结论，全量集成测试却要十几分钟。解法是分层——不同阶段跑不同粒度的测试，靠命名约定和构建插件把它们分开。

## 1.1 命名约定

- 单元测试：`*Test`——纯 Jupiter 或切片测试，不依赖外部服务
- 集成测试：`*IT`（Integration Test）——需要数据库容器、真实端口、外部依赖

命名不只是风格问题：Maven 的 Surefire/Failsafe **靠文件名模式识别**该跑哪些测试，Gradle 的自定义任务同样按 pattern 过滤。约定一旦被破坏（集成测试叫了 `XxxTest`），PR 流水线就会意外拉起 Testcontainers，反馈时间瞬间劣化。

## 1.2 Maven Surefire / Failsafe

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.5</version>
    <configuration>
        <includes>
            <include>**/*Test.java</include>      <!-- 只跑单元测试 -->
        </includes>
    </configuration>
</plugin>
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-failsafe-plugin</artifactId>
    <version>3.2.5</version>
    <configuration>
        <includes>
            <include>**/*IT.java</include>        <!-- 集成测试 -->
        </includes>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>integration-test</goal>     <!-- 绑定 verify 阶段前 -->
                <goal>verify</goal>               <!-- 检查结果并让构建失败 -->
            </goals>
        </execution>
    </executions>
</plugin>
```

Surefire 执行单元测试（`test` 阶段），Failsafe 执行集成测试（`integration-test` + `verify` 阶段）。两者的关键区别在失败语义：**Surefire 测试失败立即终止构建；Failsafe 的 `integration-test` 阶段失败不会立刻终止**，而是等到 `verify` goal 才让构建失败——这个设计是为了保证 `post-integration-test` 阶段有机会停掉预启动的应用/容器（`pre-integration-test` 启动、`post-integration-test` 关闭，Failsafe 夹在中间）。只配了 `integration-test` 忘配 `verify` 的话，**集成测试失败构建照样绿**——经典 CI 事故。

## 1.3 Gradle 测试任务

```kotlin
tasks.test {                                    // 内置 test 任务：单元测试
    useJUnitPlatform()
    filter { excludeTestsMatching("*IT") }
}

val integrationTest by tasks.registering(Test::class) {   // 自定义集成测试任务
    useJUnitPlatform()
    filter { includeTestsMatching("*IT") }
    shouldRunAfter(tasks.test)                  // 顺序：单测先跑
}

tasks.check { dependsOn(integrationTest) }      // check 聚合，CI 调 ./gradlew check
```

可自定义 `integrationTest` 任务执行集成测试，报告输出到独立目录（`reports.junitXml.outputLocation`），CI 上分别归档。

## 1.4 分层流水线

```text
┌───────────────┐    ┌────────────────┐    ┌─────────────────┐
│  PR 流水线      │    │  main 合并后     │    │  夜间/发布前      │
│               │    │                │    │                 │
│  编译 + 静态检查 │    │  全量单元 + 切片  │    │  完整 IT 套件     │
│  单元测试(*Test)│───▶│  集成测试(*IT)   │───▶│  多版本库矩阵      │
│  <5 分钟       │    │  <15 分钟       │    │  性能基线对比      │
└───────────────┘    └────────────────┘    └─────────────────┘
     反馈优先              覆盖优先              真实性优先
```

> **行业实践**：PR 流水线失败必须"值得阻塞合并"。如果某类测试 flaky 到每周误伤一次，就把它降级到 main 后阶段并限期修复——**被开发者习惯性无视的红灯比没有红灯更糟**。分层的本质是让每一层都对得起它占用的等待时间。

---

# 2. 标签与过滤

## 2.1 JUnit 标签

命名约定解决"两类测试"，`@Tag` 解决更细的维度（慢、需要 Docker、需要外网）：

```java
@Tag("slow")                       // 类级
@Tag("docker")
@SpringBootTest(webEnvironment = RANDOM_PORT)
class FullStackIT { }

@Test
@Tag("flaky-quarantine")           // 隔离区标签：疑似不稳定，暂不阻塞
void timingSensitiveTest() { }
```

## 2.2 构建工具过滤

```bash
# Maven
mvn test -Dgroups=slow                       # 只跑带标签的
mvn test -DexcludedGroups=slow,docker        # 排除（Surefire 3.x 用 excludedGroups）
mvn test -Dtest='Order*Test'                 # 按类名过滤
```

```kotlin
// Gradle：JUnit Platform 标签表达式
tasks.test {
    useJUnitPlatform {
        includeTags("fast & !docker")        // 支持 &、|、! 组合表达式
    }
}
```

```bash
gradle test --tests "*OrderServiceTest"      # 按类/方法过滤
```

隔离区（quarantine）模式的用法：flaky 测试打上 `flaky-quarantine` 标签从主流水线排除，但**夜间流水线仍然跑它并统计通过率**——隔离是暂存不是流放，两周内修不好就删。

---

# 3. Flaky Test 处理策略

Flaky（时红时绿）测试的破坏力大于稳定失败的测试：它训练团队忽略红灯。处理纪律：

- **首选修复根因**（时序、并发、外部依赖）：Spring 测试的四大 flaky 源——异步等待不足（用 Awaitility 替代 sleep）、测试间共享状态（上下文缓存里的脏单例、未清理的表）、时间依赖（月末/闰年挂掉，注入 Clock）、端口/资源竞争（并行下固定端口）；
- **不建议长期依赖重试机制掩盖问题**：Surefire 的 `rerunFailingTestsCount`、Gradle 的 test-retry 插件可以作为止血带，但每次重试成功都要记录进 flaky 报表，重试次数超过阈值（比如一个测试月内被重试 3 次）就强制进隔离区；
- **重跑与隔离的边界**：本地复现用 `mvn test -Dtest=FlakyTest -Dsurefire.rerunFailingTestsCount=0` 循环跑到红为止，配合固定随机种子（`junit.jupiter.execution.order.random.seed`）复现顺序相关的 flaky。

```xml
<!-- Surefire 重试配置（止血用，配报表） -->
<configuration>
    <rerunFailingTestsCount>2</rerunFailingTestsCount>
</configuration>
```

---

# 4. 常见问题排查

## 4.1 Context 加载失败

`Failed to load ApplicationContext` 是信息量最低的错误——真正的根因在 caused-by 链的最底层，**永远先看最后一个 `Caused by`**：

| 现象 | 常见原因 | 解决 |
|------|----------|------|
| `Failed to load ApplicationContext` | Bean 缺失/创建失败 | 看最底层 Caused by；`@MockitoBean` 补缺失依赖或补配置 |
| `Port already in use` | 固定端口冲突（本地残留进程/并行） | 使用 `RANDOM_PORT` |
| `NoSuchBeanDefinitionException` | 切片测试缺少 Bean | `@Import` 或 `@MockitoBean` |
| `BeanDefinitionOverrideException` | 测试 Bean 与主 Bean 同名 | 测试 Bean 改名 + `@Primary`（第 11 篇） |
| `UnsatisfiedDependencyException` | 注入点类型歧义/缺失 | 看 Bean 名定位注入点，补 `@Qualifier` 或配置 |
| `Unable to find a @SpringBootConfiguration` | 测试类不在主类包下 | `@SpringBootTest(classes = Application.class)` |

**Bean 循环依赖**单独说：Boot 2.6+ 默认**禁止**循环引用，启动直接报 `BeanCurrentlyInCreationException`。测试里撞上的处理顺序：① 生产代码解开循环（构造器注入改setter/抽第三个 Bean）——这是正解；② 确认不是测试配置把两个不该相见的 Bean 装进了同一容器；③ 万不得已 `spring.main.allow-circular-references=true`（欠债标记，别合进 main）。

## 4.2 数据库相关错误

| 现象 | 原因 | 解决 |
|------|------|------|
| 表不存在 | 未执行迁移 | 打开 Flyway/Liquibase；检查脚本位置配置 |
| 数据污染 | 回滚失效/提交型测试未清理 | 检查事务配置（第 13 篇）；补 AFTER 清理 |
| `Connection refused` | Testcontainers 没起来/Docker 不可用 | CI runner 装 Docker；检查 `DOCKER_HOST` |
| 约束异常但本地能过 | H2 与生产方言差异 | 该测试切真库容器（第 12 篇） |
| 死锁/超时（并行时） | 连接池太小或 TRUNCATE 竞争 | 调池大小；提交型测试加资源锁 |

## 4.3 Profile 不生效

- 未设置 `@ActiveProfiles`；
- 配置文件名称错误（`application_test.yml` 下划线、放错 resources 目录）；
- `src/test/resources/application.yml` 遮蔽了 main 的同名文件（第 11 篇 1.1 的注意点）。

## 4.4 读上下文缓存统计与 ApplicationEvents

性能型故障（套件越跑越慢）的排查入口在第 17 篇已给：`org.springframework.test.context.cache=DEBUG` 看容器数量。这里补一个**运行时诊断**手段——`ApplicationEvents` 不止能记录业务事件，框架自身的 `ContextRefreshedEvent`、`DirtiesContextEvent` 也在里面：

```java
@SpringBootTest
@RecordApplicationEvents
class ContextDiagnosticsTest {

    @Autowired
    private ApplicationEvents events;

    @Test
    void dump_framework_events() {
        events.stream()
              .map(e -> e.getClass().getSimpleName())
              .forEach(System.out::println);   // 观察容器刷新/脏化事件，定位缓存失效点
    }
}
```

配合启动日志里的 `Started Application in X seconds` 出现次数，可以数出一次 CI 构建到底起了几个容器。

## 4.5 排错流程图

```text
 测试失败
    │
    ├─ 编译期失败？───▶ 修代码，与测试框架无关
    │
    ├─ 单个测试失败、本地可复现？
    │      │
    │      ├─ 是 Context 加载失败？
    │      │     ├─ 看最底层 Caused by
    │      │     ├─ Bean 缺失 → @MockitoBean / @Import / 补配置
    │      │     ├─ 端口占用 → RANDOM_PORT
    │      │     └─ 循环依赖 → 解开生产代码循环
    │      │
    │      ├─ 是断言失败？
    │      │     ├─ 数据看不到 → 事务回滚/异步可见性（第 13 篇）
    │      │     ├─ 值不稳定 → 时间/顺序依赖 → Clock / 自建数据
    │      │     └─ SQL 方言 → H2 换真库容器（第 12 篇）
    │      │
    │      └─ 是超时/挂死？
    │            ├─ 裸 future.get() / sleep → Awaitility + 超时
    │            └─ Testcontainers 拉镜像慢 → 预热镜像缓存
    │
    └─ 仅 CI 失败、本地绿？
           ├─ 顺序/并行差异 → 固定种子复现，查共享状态
           ├─ 资源差异（内存/Docker）→ 查 runner 配置
           ├─ 时间/时区差异 → 显式 user.timezone
           └─ 依赖版本漂移（SNAPSHOT/缓存）→ 锁版本、清缓存重跑
```

"仅 CI 失败"里最阴险的是**缓存中毒**：CI 复用了旧的 Maven/Gradle 依赖缓存或 Testcontainers 镜像，行为与干净环境不同。排查时先跑一次 `--no-cache`/清缓存构建确认。

---

# 5. Docker 化测试环境

## 5.1 CI Runner 上的 Testcontainers

Testcontainers 需要 Docker 环境，CI 上三种供给方式：

| 方式 | 说明 | 适用 |
|------|------|------|
| Docker-in-Docker（dind） | runner 容器里再起 dockerd | GitLab CI 常见；特权模式有安全面 |
| Docker socket 挂载 | 挂宿主机 `/var/run/docker.sock` | 简单快；容器可见宿主资源 |
| 云 CI 原生 Docker | GitHub Actions 的 ubuntu runner 自带 Docker | 零配置，主流选择 |

GitHub Actions 示例：

```yaml
jobs:
  integration-test:
    runs-on: ubuntu-latest              # 自带 Docker daemon
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 21
          cache: maven                   # 依赖缓存
      - name: Run integration tests
        run: mvn verify -B               # Failsafe 阶段跑 *IT，Testcontainers 自动起容器
```

加速要点：**镜像预拉取**（CI 缓存层里 `docker pull postgres:16`，或 Testcontainers 的 `pull.pause.timeout` 配置）、**容器复用**（`withReuse(true)` + runner 持久化 `~/.testcontainers.properties`；注意 ephemeral runner 上 reuse 无效，改用 singleton 容器模式）、镜像用 slim/alpine 变体减小拉取量。

## 5.2 无 Docker 环境的降级

某些企业 CI 不给 Docker。降级路线：数据库退回 H2（接受方言风险，原生 SQL 测试打 `@Tag("docker")` 跳过）、或连共享的常驻测试库（数据隔离压力转给清表策略）。**不要为了绕开 Docker 把集成测试全 Mock 掉**——那等于删掉了这层测试。

---

# 6. 面试与高频要点

## 6.1 高频问题

1. `@SpringBootTest` 与切片测试的区别
2. TestContext 缓存机制与 `@DirtiesContext`
3. `@MockBean`（`@MockitoBean`）与 `@Mock` 的差异
4. 为什么测试默认回滚
5. MockMvc 与 TestRestTemplate 选型
6. Surefire 与 Failsafe 的分工、为什么 Failsafe 要配 `verify` goal
7. CI 里 Testcontainers 怎么加速

## 6.2 回答思路

- **说明目的与边界**：先讲这个机制解决什么问题（缓存解决容器贵、Failsafe 解决"失败也要清理"）；
- **解释底层机制**：缓存 Key 是 `MergedContextConfiguration`；Mock 替换发生在 BeanFactoryPostProcessor；回滚由 `TransactionalTestExecutionListener` 执行；
- **给出场景与取舍**：什么时候用、什么时候别用、代价是什么——这一层最能区分"背过"与"用过"。

# 7. 小结

| 主题 | 关键结论 |
|------|---------|
| 分层 | `*Test` PR 跑、`*IT` 合并后/夜间跑；Surefire/Failsafe 按名字识别 |
| Failsafe | 必须配 `verify` goal，否则集成测试失败构建仍绿 |
| 标签 | `@Tag` 管慢/docker/隔离区维度；隔离区夜间仍统计通过率 |
| Flaky | 修根因优先，重试只做止血带并强制进报表 |
| 排错 | Context 失败看最底层 Caused by；仅 CI 失败先怀疑顺序/资源/缓存 |
| 缓存统计 | `test.context.cache=DEBUG` + `ApplicationEvents` 数容器数量 |
| Docker | GitHub Actions 原生 Docker；镜像预拉取 + singleton 容器加速 |
| 循环依赖 | Boot 2.6+ 默认禁止，解生产代码的环，别开 allow 开关合进 main |

至此 SpringBootTest 系列完结：从体系概览（第 1 篇）到 CI 落地（本篇），中间覆盖了 TestContext 框架、切片、MockMvc、Mock 替身、配置属性、数据库、事务、安全、异步、可观测性与性能——一套完整的 Spring Boot 测试工程知识地图。
