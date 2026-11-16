# 1. IDE 运行与调试

测试写出来只是第一步，**能不能在 IDE 里快速运行、精准调试，直接决定了写测试的体验和迭代速度**。主流 IDE（IntelliJ IDEA、Eclipse、VS Code）都通过 JUnit Platform 的 Launcher API 支持 JUnit 5，运行入口基本一致，但底层接线方式各有差异，理解这些差异能避免"IDE 里能跑、命令行跑不了"这类经典问题。

## 1.1 三大 IDE 的运行方式

**IntelliJ IDEA** 对 JUnit 5 的支持最完整。IDEA 通过 `junit-platform-launcher` 发现并执行测试，编辑器左侧会出现绿色运行图标：点方法旁的图标运行单个测试方法，点类旁的图标运行整个测试类，右键包名可以运行整个包或模块的测试。运行配置（Run Configuration）支持按标签过滤、指定 VM 参数、设置环境变量，且支持 Gradle/Maven 委托执行。

```text
IDEA 中的三个运行粒度：

  方法级 → 点 @Test 方法左侧图标     （最快的反馈回路）
  类级   → 点测试类左侧图标          （验证一个类的全部行为）
  包级   → 右键包 → Run Tests       （提交前的局部回归）
```

**Eclipse** 需要在 Build Path 中引入 JUnit 5 库（Eclipse 2019-03 之后内置支持，右键 → Run As → JUnit Test 即可）。Maven 项目只要 `pom.xml` 里有 `junit-jupiter` 依赖，Eclipse 会自动识别；老项目常见问题是 Build Path 里残留 JUnit 4 库导致 Runner 混用。

**VS Code** 依赖 "Test Runner for Java" 扩展（Extension Pack for Java 自带）。它同样通过 JUnit Platform Launcher 执行测试，测试树显示在侧边栏的 Testing 面板里，支持单方法运行、调试和查看失败详情。轻量但对 Gradle 多模块项目的索引偶尔需要手动触发（Java: Clean Java Language Server Workspace）。

> **行业实践**：日常开发中 90% 的测试运行发生在"方法级"。把光标放在测试方法里按快捷键（IDEA 默认 `Ctrl+Shift+F10`，VS Code 的 CodeLens "Run Test"）是最短反馈路径，跑全量测试应该留给构建工具。

## 1.2 IDE 与构建工具的委托关系

IDE 直接跑测试和构建工具跑测试**用的是两套发现机制**：IDE 用自己的 Launcher 按类路径扫描，Maven/Gradle 则通过 Surefire/Test 插件。两者可能出现行为不一致，典型场景是：

- **类路径差异**：IDE 用的是编译输出目录，构建工具用的是打包依赖解析结果，`test` scope 依赖缺失时只有构建工具会报错。
- **配置不生效**：`junit-platform.properties` 里的并行配置、标签过滤表达式，IDE 的运行配置可能没有读取。
- **注解处理器**：Lombok、MapStruct 生成的代码，IDE 增量编译和构建工具全量编译的时序不同。

IDEA 提供了 **Delegate IDE build/run actions to Gradle/Maven** 选项（Settings → Build Tools），打开后 IDE 的构建和测试运行完全走构建工具，保证与 CI 行为一致，代价是启动更慢。

> **注意**：排查"IDE 通过、CI 失败"的问题时，第一步就是在本地用构建工具跑一遍（`mvn test` / `gradle test`），确认不是 IDE 环境的假绿。

## 1.3 调试技巧

调试测试和普通调试没有本质区别，但有几个针对测试的实用技巧。**断点打在断言行之前**，观察被测方法的实际返回值与期望值的差异；对循环中的失败，用 IDEA 的条件断点（右键断点 → Condition）只在特定迭代暂停。

对**不可重复的偶发失败**（Flaky），断点往往抓不到现场，此时应该：

- 用 `@RepeatedTest(100)` 放大出现概率，配合日志输出每次迭代的输入；
- 用 `@Timeout` 暴露隐藏的慢路径；
- 检查测试间共享状态（static 字段、外部资源），单独跑通过、批量跑失败多半是测试间污染。

```java
import org.junit.jupiter.api.RepeatedTest;
import org.junit.jupiter.api.RepetitionInfo;

class IdGeneratorTest {

    @RepeatedTest(50) // 重复 50 次，放大偶发冲突的概率
    void should_never_generate_duplicate_id(RepetitionInfo info) {
        long id = IdGenerator.next();
        System.out.printf("第 %d 次迭代生成 id=%d%n",
                info.getCurrentRepetition(), id);
        assertNotEquals(0L, id); // 简单校验，真实去重逻辑见对应测试
    }
}
```

# 2. CLI 执行：junit-platform-console-standalone

脱离 IDE 和构建工具运行测试，靠的是 **JUnit Platform Console Standalone**——一个把 Platform、Jupiter 引擎、Launcher 和 ConsoleLauncher 全部打进一个可执行 JAR 的独立发行包。它的典型用途是：验证一个只包含 class 文件的产物、在没有 Maven/Gradle 的环境（如某些容器镜像、教学环境）里跑测试、以及在脚本里做精细化的测试筛选。

## 2.1 获取与基本运行

从 Maven Central 下载 standalone JAR（版本号与 JUnit 5.x 对应，如 5.10.2 对应 1.10.2）：

```bash
# 下载（也可直接从浏览器获取）
wget https://repo1.maven.org/maven2/org/junit/platform/junit-platform-console-standalone/1.10.2/junit-platform-console-standalone-1.10.2.jar

# 扫描类路径下所有测试并执行
java -jar junit-platform-console-standalone-1.10.2.jar \
  --class-path target/classes:target/test-classes \
  --scan-class-path
```

执行结束后控制台输出树状摘要（Tree Details），包含每个容器/测试的成功失败状态与耗时，进程退出码非 0 表示有失败，可直接用于脚本判断。

## 2.2 常用参数速查

| 参数 | 作用 |
|------|------|
| `-cp, --class-path` | 指定被测类与测试类所在的类路径（含依赖） |
| `--scan-class-path` | 扫描类路径上的全部测试 |
| `-c, --select-class` | 只运行指定测试类，如 `com.example.OrderServiceTest` |
| `-m, --select-method` | 只运行指定方法，如 `com.example.OrderServiceTest#should_reject_empty_order` |
| `-p, --select-package` | 运行指定包下的全部测试 |
| `--include-tag` / `--exclude-tag` | 按标签过滤（5.10 中名为 `--include-tagname`/`--exclude-tagname`，5.11 起改为现名，旧名仍兼容） |
| `-n, --include-classname` | 按类名正则过滤，默认 `^(Test.*\|.+[.$]Test.*\|.*Tests?)$` |
| `--details` | 输出详细程度：`none` / `summary` / `flat` / `tree`（默认） |
| `--reports-dir` | 输出 XML 报告的目录 |
| `--config-file` | 指定 `junit-platform.properties` 的位置 |
| `--fail-if-no-tests` | 没有发现任何测试时以失败退出，防止 CI 静默跳过 |
| `--disable-ansi-colors` | 关闭彩色输出（CI 日志必开，否则日志里全是转义符） |

```bash
# CI 中典型的精细化运行：只跑订单模块的 fast 标签测试，输出 XML 报告
java -jar junit-platform-console-standalone-1.10.2.jar \
  --class-path "build/classes/java/main:build/classes/java/test:$(cat cp.txt)" \
  --scan-class-path \
  --include-tag fast \
  --exclude-tag slow \
  --reports-dir build/test-results \
  --details flat \
  --disable-ansi-colors \
  --fail-if-no-tests
```

> **行业实践**：生产项目的日常测试执行仍以 Maven Surefire / Gradle Test 插件为主（见第 15 篇），standalone 更多用于**无构建工具环境的验证**和**教学演示**。它的价值在于让你看清 JUnit Platform 的本质——测试执行就是"Launcher 拿着选择器（Selector）和过滤器（Filter）去类路径上发现并执行测试"。

# 3. 报告与输出

## 3.1 JUnit XML 报告格式

构建工具和 standalone 都能产出事实标准——**JUnit XML 报告**，几乎所有 CI 系统（Jenkins、GitLab CI、GitHub Actions、TeamCity）都能解析它并在界面上渲染失败详情。格式核心是两层结构：`<testsuite>` 汇总统计，`<testcase>` 逐条记录：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<testsuite name="com.example.OrderServiceTest" tests="3" failures="1"
           errors="0" skipped="0" time="0.128">
  <testcase name="should_reject_empty_order" classname="com.example.OrderServiceTest"
            time="0.031"/>
  <testcase name="should_calculate_total_price" classname="com.example.OrderServiceTest"
            time="0.045">
    <failure message="expected: &lt;100&gt; but was: &lt;90&gt;"
             type="org.opentest4j.AssertionFailedError">
      org.opentest4j.AssertionFailedError: expected: &lt;100&gt; but was: &lt;90&gt;
        at com.example.OrderServiceTest.should_calculate_total_price(OrderServiceTest.java:28)
      <!-- 完整堆栈 -->
    </failure>
  </testcase>
  <testcase name="should_apply_coupon" classname="com.example.OrderServiceTest"
            time="0.001">
    <skipped message="@Disabled: 优惠券服务未就绪"/>
  </testcase>
</testsuite>
```

各工具的产出位置：

| 工具 | XML 报告位置 | 说明 |
|------|-------------|------|
| Maven Surefire | `target/surefire-reports/TEST-*.xml` | 每个测试类一个文件 |
| Gradle Test | `build/test-results/test/TEST-*.xml` | 另附 HTML 报告 `build/reports/tests/test/index.html` |
| ConsoleLauncher | `--reports-dir` 指定目录下的 `TEST-junit-jupiter.xml` | 单文件汇总 |
| Failsafe（集成测试） | `target/failsafe-reports/TEST-*.xml` | `*IT.java` 命名 |

**注意区分 `failure` 与 `error`**：`failure` 是断言失败（测试逻辑判定的"不符合预期"），`error` 是意外异常（如 NPE、连接超时）。CI 分析时 error 往往指向环境问题而非代码缺陷，两者的根因排查路径完全不同。

## 3.2 CI 系统的解析与展示

CI 解析 XML 后通常提供三种能力：**趋势图**（失败率随构建次数的变化）、**失败下钻**（点击构建直接看到失败的 testcase 与堆栈，不用翻全量日志）、**变更归因**（把失败关联到本次提交的 diff）。以 GitHub Actions 为例：

```yaml
name: test
on: [push]
jobs:
  unit-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17
      - name: Run tests
        run: mvn -B test   # Surefire 生成 target/surefire-reports/*.xml
      - name: Publish test report
        if: always()       # 测试失败时也要发布报告，否则看不到失败详情
        uses: mikepenz/action-junit-report@v4
        with:
          report_paths: target/surefire-reports/TEST-*.xml
```

`if: always()` 是容易被忽略的关键配置——测试失败的构建恰恰是最需要报告的时候。

## 3.3 失败信息的可读性

失败信息是测试留给未来的自己的"案发现场照片"，**断言失败时必须能一眼看出"哪个业务场景、期望什么、实际什么"**。原生 `assertEquals(expected, actual)` 的默认消息只有值对比，缺乏上下文，应该养成带 message 的习惯，或者改用 AssertJ 的流式断言（`as()` 描述 + 更智能的 diff）：

```java
// 不推荐：失败时只能看到 expected: <100> but was: <90>，不知道是哪个字段
assertEquals(100, order.getTotal());

// 推荐一：JUnit 5 原生断言 + 上下文消息（消息用 Supplier 惰性构造，避免成功路径的拼接开销）
assertEquals(100, order.getTotal(),
        () -> "订单 " + order.getId() + "（3 件商品 + 满 100 减 10 券）总价应为 100");

// 推荐二：AssertJ，失败消息自动包含对象结构 diff
assertThat(order.getTotal())
        .as("订单总价：3 件商品单价 30/40/40，应用满 100 减 10 券")
        .isEqualTo(100);
```

同时**不要把关键上下文吞掉**：捕获异常后再断言时保留原始异常链；测试涉及外部输入（文件、接口响应）时，失败路径上把输入内容 dump 到日志。

## 3.4 重跑失败测试

大型测试套件中"只重跑失败的那几个"是高频需求，各工具的支持方式不同：

- **Maven Surefire**：`-Dsurefire.rerunFailingTestsCount=2` 自动重跑失败测试最多 2 次，全部重跑通过则标记为 *flaky* 而非失败——适合治理中的过渡手段，但会掩盖 Flaky 问题，不宜长期依赖。
- **Gradle**：内置 `--tests` 过滤按模式重跑：`gradle test --tests "com.example.OrderServiceTest.should_*"`；官方的 **Test Retry 插件**（`org.gradle.test-retry`）提供与 Surefire 类似的重试与 flaky 标记能力，还能区分"本地重试"与"CI 重试"。
- **IDEA**：运行结果树里右键失败节点 → Rerun Failed Tests，最快的局部重跑方式。
- **JUnit Platform 本身**不提供"重跑失败"语义，这需要外层工具记录上一次结果再做选择——这是"平台负责发现与执行、工具负责策略"分层设计的又一体现。

> **设计争议**：自动重试失败测试应该开吗？支持方认为 CI 机器抖动、网络闪断不可避免，重试能减少无效红灯；反对方认为**重试是把 Flaky Test 从"必须治理的问题"降级为"可以忽略的噪音"**，长期看腐蚀测试套件的信号价值。折中的行业实践是：开启重试但把 flaky 结果显式上报（Test Retry 插件和 Surefire 都会标记），定期清零 flaky 存量。

# 4. 代码覆盖率

## 4.1 JaCoCo 集成

覆盖率工具的主流选择是 **JaCoCo**（Cobertura 已基本停止维护）。JaCoCo 通过 Java Agent 在字节码层面插桩，Surefire 配置 `argLine` 挂上 agent 即可在测试运行时收集数据：

```xml
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <version>0.8.11</version>
  <executions>
    <execution>
      <id>prepare-agent</id>
      <goals><goal>prepare-agent</goal></goals> <!-- 注入 argLine，挂载插桩 agent -->
    </execution>
    <execution>
      <id>report</id>
      <phase>test</phase>
      <goals><goal>report</goal></goals>        <!-- 生成 target/site/jacoco/ 下的 HTML/XML 报告 -->
    </execution>
    <execution>
      <id>check</id>
      <goals><goal>check</goal></goals>
      <configuration>
        <rules>
          <rule>
            <element>BUNDLE</element>
            <limits>
              <limit>
                <counter>LINE</counter>
                <value>COVEREDRATIO</value>
                <minimum>0.60</minimum>          <!-- 行覆盖率低于 60% 时构建失败 -->
              </limit>
            </limits>
          </rule>
        </rules>
      </configuration>
    </execution>
  </executions>
</plugin>
```

Gradle 侧更简单，`jacoco` 插件是官方内置的：`plugins { id 'jacoco' }`，`gradle test jacocoTestReport` 产出 `build/reports/jacoco/test/html/index.html`。

## 4.2 覆盖率指标的正确解读

覆盖率有多个维度：**行覆盖（LINE）**、**分支覆盖（BRANCH）**、**方法覆盖（METHOD）**、**指令覆盖（INSTRUCTION）**。分支覆盖比行覆盖严格——一行 `if (a && b)` 行覆盖 100% 时分支覆盖可能只有 50%。

**覆盖率是"必要不充分"指标**：100% 覆盖率完全可能来自一堆没有断言的空跑测试。正确用法是：

- 作为**趋势指标**看变化方向，而不是作为绝对 KPI；
- 作为**风险信号**定位未覆盖的高风险代码（金额计算、状态机、边界条件），而不是追求数字达标；
- 增量覆盖率（只统计本次 diff 的覆盖）比全量覆盖率更能反映当下的工程质量，SonarQube 等平台的"Coverage on New Code"就是这个思路。

> **行业实践**：多数团队把门禁设在增量覆盖率（如新代码 ≥ 80%）而非全量覆盖率上——存量代码的历史欠账不应该阻塞今天的提交，但新代码必须达标。

# 5. 小结

| 主题 | 关键结论 |
|------|---------|
| IDE 运行 | 方法级运行是最短反馈回路；"IDE 绿、CI 红"先用构建工具本地复跑定位 |
| Console Standalone | 无构建工具环境的执行手段；`--select-*` 选择器 + `--include-tag` 过滤器是核心参数 |
| JUnit XML | 事实标准报告格式；区分 failure（断言失败）与 error（意外异常）；CI 发布报告要 `if: always()` |
| 失败可读性 | 断言带业务上下文消息，或用 AssertJ 的 `as()`；不吞异常链 |
| 重跑失败 | Surefire `rerunFailingTestsCount`、Gradle Test Retry、IDEA Rerun Failed；重试要显式上报 flaky |
| 覆盖率 | JaCoCo 是主流；看趋势与增量，不把它当唯一 KPI |

下一篇进入 Spring Boot 测试集成，看 JUnit 5 如何通过 `SpringExtension` 与 Spring 容器协作。
