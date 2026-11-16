# 1. 迁移策略总览

JUnit 4 到 5 的迁移**不是一次性替换，而是可以先共存再渐进**。关键支撑是 JUnit Platform 的多引擎架构：`junit-vintage-engine` 负责在 Platform 上运行存量 JUnit 4（乃至 JUnit 3）测试，`junit-jupiter-engine` 运行新测试，两者可以在同一个项目里同时工作。因此推荐路径是：**先让构建工具切到 JUnit Platform，加上 Vintage 引擎保证存量测试照跑，之后新测试一律写 Jupiter，存量测试按模块逐步改写，最后移除 Vintage**。

## 1.1 三种策略

| 策略 | 做法 | 适用 |
|------|------|------|
| 全量迁移 | 一次性改写全部测试 | 小项目、测试量少（几十个类以内） |
| 渐进迁移（推荐） | 新增测试用 Jupiter，存量用 Vintage 执行，分批改写 | 中大型项目、持续交付不允许大爆炸式改动 |
| 混合共存 | 两套长期并存，不主动改写 | 维护期项目、改写收益低于风险 |

> **行业实践**：绝大多数生产项目选渐进迁移。落地的第一步不是改测试代码，而是改构建配置——一旦 Surefire/Gradle 切到 Platform + Vintage，CI 绿灯不变，团队就获得了"随时可以开始改写"的安全网。反过来，"等有空再一次性迁完"的计划几乎从不兑现。

## 1.2 共存期的注意事项

- **一个测试类不要混用两代注解**：`@Before` 和 `@BeforeEach` 同时出现时，Vintage 只认前者、Jupiter 只认后者，会出现"某个初始化没执行"的诡异现象。
- **依赖对齐**：Vintage 引擎与 Jupiter 版本必须一致（都来自 junit5 的 BOM），JUnit 4 用 4.12+（建议 4.13.2）。
- **断言混用无碍**：`org.junit.Assert` 与 `org.junit.jupiter.api.Assertions` 在同一个类里共存没有运行时问题，只是风格不统一，改写时顺手替换。

# 2. 关键差异映射

## 2.1 注解与生命周期对照

| JUnit 4 | JUnit 5 | 差异说明 |
|---------|---------|---------|
| `@BeforeClass` | `@BeforeAll` | 默认仍要求 static；`PER_CLASS` 下可非 static |
| `@AfterClass` | `@AfterAll` | 同上 |
| `@Before` | `@BeforeEach` | 语义相同，**名字更准确** |
| `@After` | `@AfterEach` | 语义相同 |
| `@Test`（org.junit） | `@Test`（org.junit.jupiter.api） | 包名不同；Jupiter 的不再承载 expected/timeout 属性 |
| `@Ignore` | `@Disabled` | `@Disabled` 支持 reason，且可用于类级 |
| `@Category(X.class)` | `@Tag("x")` | 标记接口 → 字符串，可叠加、可表达式过滤 |
| `@RunWith(Parameterized.class)` | `@ParameterizedTest` + 数据源注解 | 参数化从"类级 Runner"降级为"方法级注解" |
| `@Rule` / `@ClassRule` | Extension + `@RegisterExtension` | 见 2.4 |
| 无 | `@TestFactory` / `@Nested` / `@DisplayName` | Jupiter 新增能力，迁移后可逐步利用 |

**最容易漏的一条**：JUnit 5 的测试类和测试方法**不要求 public**，包级私有即可；同时 Jupiter 默认每个测试方法新建一个测试实例（`PER_METHOD`），与 JUnit 4 行为一致，但 JUnit 5 允许切到 `PER_CLASS`。

## 2.2 断言与异常对照

JUnit 4 声明式异常断言迁移到 JUnit 5 的 `assertThrows`：

```java
// JUnit 4：expected 属性——只能断类型，不能断消息，且整个方法只允许一个异常点
@Test(expected = IllegalArgumentException.class)
public void createWithNullName() {
    service.create(null);
}

// JUnit 5：assertThrows 返回异常对象，可以继续断消息、字段，且方法里可以有多处
@Test
void create_with_null_name() {
    IllegalArgumentException ex = assertThrows(
        IllegalArgumentException.class,
        () -> service.create(null)
    );
    assertEquals("name is required", ex.getMessage());   // 断得更细
}
```

其他断言差异：`assertEquals` 的 message 参数从第一位挪到**最后一位**；新增 `assertAll`（组合断言，全部执行完再汇报）、`assertTimeout` / `assertTimeoutPreemptively`（替代 `@Test(timeout=...)`）、`assertArrayEquals` 支持更多类型。`org.junit.Assert.assertThat` 被移除，替代方案是 Hamcrest 自己的 `MatcherAssert.assertThat` 或引入 AssertJ。

## 2.3 Categories 到 Tag

```java
// JUnit 4：先定义标记接口，Runner 配置繁琐
public interface SlowTests {}

@RunWith(Categories.class)
@Categories.IncludeCategory(SlowTests.class)
@Suite.SuiteClasses({ATest.class, BTest.class})
public class SlowSuite {}

// JUnit 5：字符串标签直接用，过滤交给构建工具
@Tag("slow")
@Test
void heavy_test() { ... }
```

迁移时把每个 Category 接口映射成一个 tag 字符串，构建命令从 `-Dgroups=SlowTests`（Surefire Categories 配置）换成 `-Dgroups="slow"`。Tag 还支持组合表达式（`"slow & !flaky"`），这是 Categories 做不到的。

## 2.4 Runner / Rule 到 Extension

`@RunWith` 换成 `@ExtendWith`，主流框架都提供了现成映射：

| JUnit 4 | JUnit 5 替代 |
|---------|-------------|
| `SpringRunner` | `SpringExtension`（`@SpringBootTest` 已内置，直接删注解） |
| `MockitoJUnitRunner` | `MockitoExtension`（`mockito-junit-jupiter` 依赖） |
| `Parameterized` | `@ParameterizedTest` |
| `SpringClassRule`/`SpringMethodRule` | `SpringExtension` |
| 自定义 Runner | 拆解为一组 Extension 回调接口重写 |

Rule 的常见迁移路径：

```java
// JUnit 4：TemporaryFolder Rule
@Rule
public TemporaryFolder folder = new TemporaryFolder();

// JUnit 5：内建 @TempDir，无需自定义扩展
@TempDir
Path tempDir;
```

`ExpectedException` Rule → `assertThrows`；`TestWatcher` Rule → 实现 `TestWatcher` 接口的 Extension；`ExternalResource` Rule → `BeforeEachCallback`/`AfterEachCallback` 组合；第三方 Rule（如 Testcontainers 的 `@Container` 时代）→ 优先找官方 Jupiter 扩展（`testcontainers` 的 `@Testcontainers` + `@Container` 注解直接支持 Jupiter）。**没有现成扩展的自定义 Rule，是迁移工作量的主要来源**，需要用 12 篇讲的 Extension 模型重写。

# 3. 构建配置

## 3.1 Maven

Surefire 2.22.0+ 才原生支持 JUnit Platform，建议直接上 3.x：

```xml
<dependencies>
    <!-- Jupiter：新测试 -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.10.2</version>
        <scope>test</scope>
    </dependency>
    <!-- Vintage：存量 JUnit 4 测试的过渡引擎，迁完删除 -->
    <dependency>
        <groupId>org.junit.vintage</groupId>
        <artifactId>junit-vintage-engine</artifactId>
        <version>5.10.2</version>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>3.2.5</version>   <!-- 自动探测 Platform，无需手动配 provider -->
        </plugin>
    </plugins>
</build>
```

多模块项目用 `junit-bom` 统一版本，避免 Jupiter/Vintage/Params 版本漂移：

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.junit</groupId>
            <artifactId>junit-bom</artifactId>
            <version>5.10.2</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

## 3.2 Gradle

```groovy
dependencies {
    testImplementation platform('org.junit:junit-bom:5.10.2')   // 对齐版本
    testImplementation 'org.junit.jupiter:junit-jupiter'
    testRuntimeOnly 'org.junit.vintage:junit-vintage-engine'    // 过渡期引擎
}

test {
    useJUnitPlatform()      // 没有这行，JUnit 5 测试一个都不会跑
}
```

**`useJUnitPlatform()` 是 Gradle 迁移最常漏的一步**：不加它时 Gradle 仍用 JUnit 4 执行器，Jupiter 测试被静默跳过（0 个测试执行，构建还是绿的），比报错更危险。

# 4. 自动化迁移工具

手工逐类改写效率低，两个主流工具可以完成 80% 的机械替换：

## 4.1 OpenRewrite

```bash
# 一次性执行 JUnit 4 → 5 配方：注解、断言、生命周期、import 全自动替换
mvn -U org.openrewrite.maven:rewrite-maven-plugin:run \
    -Drewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-migrate-java:RELEASE \
    -Drewrite.activeRecipes=org.openrewrite.java.migrate.junit.JUnit4to5Migration
```

Gradle 用 `org.openrewrite.rewrite` 插件挂同一配方。配方会处理 `@Before`→`@BeforeEach`、`Assert`→`Assertions`、message 参数位置调整、`@Ignore`→`@Disabled` 等，并保留原格式风格。**跑完必须全量执行一遍测试并人工抽查 diff**——工具对 `expected`/`timeout` 属性和自定义 Rule 的转换不保证完美。

## 4.2 IDE 与 Maven 插件辅助

IntelliJ IDEA 提供 "Migrate to JUnit 5" 的 intention 与批量 inspection，逐类改写时比 OpenRewrite 更可控；`junit5-migration` 相关的 Gradle 插件生态较小，主流还是 OpenRewrite + IDE 两件套。

# 5. 分阶段迁移方案

**迁移按阶段推进，每个阶段都保持 CI 绿色**，任何一步出问题都可以停在原地而不阻塞业务：

```text
┌────────────────────────────────────────────────────┐
│ 阶段 1：基础设施                                     │
│   引入 junit-bom + jupiter + vintage               │
│   Surefire 升 3.x / Gradle 加 useJUnitPlatform()   │
│   验收：存量 JUnit 4 测试全部照常通过                │
├────────────────────────────────────────────────────┤
│ 阶段 2：增量切换                                     │
│   团队约定：新增测试一律 Jupiter                     │
│   高频修改的测试类顺手改写（童子军规则）              │
├────────────────────────────────────────────────────┤
│ 阶段 3：批量改写                                     │
│   OpenRewrite 跑 JUnit4to5Migration 配方            │
│   按模块分批提交，每批全量回归                        │
│   自定义 Rule/Runner 手工重写为 Extension            │
├────────────────────────────────────────────────────┤
│ 阶段 4：收尾                                        │
│   删除 junit-vintage-engine 与 junit:junit 依赖     │
│   CI 加依赖检查，防止 JUnit 4 回流                   │
└────────────────────────────────────────────────────┘
```

判断存量覆盖是否清零：Vintage 在场时，报告里 `test-engine-junit4` 的执行数为 0 即可进入阶段 4；或直接用 `gradle test --tests` / Surefire 报告统计仍带 `org.junit.Test` import 的类。

# 6. 迁移过程中的常见问题

## 6.1 测试执行不到（静默跳过）

最高频的问题，排查顺序：Gradle 是否加了 `useJUnitPlatform()`；Maven Surefire 版本是否 ≥ 2.22.0；`junit-jupiter` 是否在 test classpath（scope 写错成 provided 也会静默不跑）；类/方法是否误留 JUnit 4 的 `@Test` import 而项目已删 Vintage——此时该类两个引擎都不认领，**0 个测试执行且构建成功**，只有看报告里的用例数才能发现。

## 6.2 扩展行为不一致

- **Rule 与 Extension 的执行时机不同**：Rule 的 `apply` 环绕的是"整个 statement"，而 `BeforeEachCallback` 在 `@BeforeEach` 之前、`BeforeTestExecutionCallback` 在其之后，改写时要对号入座，否则初始化顺序变化会暴露隐藏 bug。
- **Runner 的组合能力变化**：JUnit 4 一个类一个 Runner，迁移后如果原 Runner 同时干了"注入 + 条件 + 环绕"三件事，应拆成多个 Extension 分别注册。
- **`@BeforeAll` 的 static 要求**：从 `PER_CLASS` 之外的类迁移时保持 static 即可；想利用 `PER_CLASS` 去掉 static，属于增强而非迁移，分开提交。

## 6.3 IDE 运行差异

IDEA/Eclipse 需要较新版本才完整支持 Jupiter（IDEA 2017.3+、Eclipse Oxygen.1a+ 且装 JUnit 5 插件）。旧 IDE 上单测右键运行走 JUnit 4 执行器，会出现"命令行通过、IDE 里找不到测试"。团队升级 IDE 应作为阶段 1 的一部分，而不是等问题出现再补。

# 7. 小结

| 主题 | 要点 |
|------|------|
| 总策略 | Platform + Vintage 先共存，新测试写 Jupiter，存量分批改，最后删 Vintage |
| 注解 | `@Before`→`@BeforeEach`、`@BeforeClass`→`@BeforeAll`、`@Ignore`→`@Disabled`、`@Category`→`@Tag` |
| 断言 | `expected` 属性→`assertThrows`；message 参数移到最后；新增 `assertAll`/`assertTimeout` |
| Runner/Rule | `@RunWith`→`@ExtendWith`；`TemporaryFolder`→`@TempDir`；自定义 Rule 重写为 Extension |
| 构建 | Surefire 3.x 自动探测；Gradle 必须 `useJUnitPlatform()`；BOM 对齐版本 |
| 工具 | OpenRewrite `JUnit4to5Migration` 配方完成机械替换，跑完全量回归 + 抽查 diff |
| 静默跳过 | 双 import 混用、缺 Vintage、缺 useJUnitPlatform 都会让测试"消失"且不报错 |
