# 1. JUnit 5 架构概览

JUnit 5 不是一次版本升级，而是一次**架构重写**：JUnit 4 是"一个 jar 包 = 注解 + 断言 + Runner 执行器"的整体式设计，JUnit 5 则拆成三个职责分离的模块——**JUnit Platform** 负责"测试如何被发现和运行"，**JUnit Jupiter** 负责"测试如何编写"，**JUnit Vintage** 负责"老测试如何继续运行"。这个拆分是理解 JUnit 5 一切新特性（Extension、@Tag、并行执行、动态测试）的前提：新特性之所以能存在，是因为执行基础设施被抽出来做成了通用平台。

```text
┌────────────────────────────────────────────────────────┐
│         构建工具 / IDE（Maven、Gradle、IntelliJ…）        │
└──────────────────────┬─────────────────────────────────┘
                       ▼ 只对接 Launcher API
┌────────────────────────────────────────────────────────┐
│                   JUnit Platform                        │
│   Launcher · DiscoverySelector · TestPlan · Listener    │
└───────┬──────────────────┬──────────────────┬──────────┘
        ▼                  ▼                  ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────────┐
│    Jupiter   │   │   Vintage    │   │  第三方引擎        │
│    Engine    │   │    Engine    │   │  Spock/TestNG/    │
│  JUnit 5 API │   │ JUnit 3/4 API│   │  jqwik/Cucumber   │
└──────────────┘   └──────────────┘   └──────────────────┘
        ▲                  ▲
        │ 编写新测试         │ 运行存量测试
   @Test(jupiter)      @Test(junit4)
```

三个模块的定位：

- **JUnit Platform**：测试发现、执行与报告的运行时基础设施。它本身不知道怎么运行任何一种测试，只定义协议（Launcher API 与 TestEngine SPI），让工具方与引擎方各自面向协议编程。
- **JUnit Jupiter**：新的编程模型（`@Test`、`@BeforeEach`、断言、参数化）与扩展模型（Extension），以及执行它们的 `junit-jupiter-engine`。这是日常写 JUnit 5 测试真正依赖的 API。
- **JUnit Vintage**：把 JUnit 3/4 的测试类适配成 Platform 上的一个引擎，让存量测试不改一行代码就能在新平台上运行。

对应的 Maven 坐标也分得很清楚，`junit-jupiter` 是聚合依赖（api + params + engine），Platform 与 Vintage 按需引入：

```xml
<!-- 写 JUnit 5 测试：只需这一个聚合依赖 -->
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.10.2</version>              <!-- Jupiter 5.10+ 对应 Platform 1.10+ -->
    <scope>test</scope>
</dependency>

<!-- 存量 JUnit 4 测试要一起跑时才引入 -->
<dependency>
    <groupId>org.junit.vintage</groupId>
    <artifactId>junit-vintage-engine</artifactId>
    <version>5.10.2</version>
    <scope>test</scope>
</dependency>
```

## 1.1 模块化设计目标

**目标一：解决 JUnit 4 扩展能力受限的问题。** JUnit 4 的扩展点 `@RunWith` 是独占的——一个测试类只能有一个 Runner，参数化、Spring 注入、Mock 框架支持无法叠加，社区为此发明了 `SpringJUnit4ClassRunner` 包 `Parameterized` 这类脆弱组合，以及 Rule 这种受限的补充机制（见第 5 篇）。JUnit 5 把"替换整个执行器"改为"在生命周期各回调点挂载 Extension"，多个扩展可以并存组合，这是 Jupiter 编程模型层面的目标。

**目标二：支持多引擎并存。** JUnit 4 时代 Spock、TestNG 各自实现了一整套运行器，构建工具要为每种框架写一遍发现与执行逻辑。Platform 抽出 `TestEngine` SPI 后，Spock 2.x、TestNG、Cucumber、jqwik 都以引擎形式跑在同一平台上，工具方只对接一次。

**目标三：让测试发现与执行对 IDE/构建工具统一。** JUnit 4 里 Surefire、Gradle、各家 IDE 各自通过反射和类路径扫描"猜"哪些是测试类，行为细节互有出入（比如 Surefire 与 IDEA 对抽象类、内部类的处理曾经不一致）。Platform 提供了标准的 Launcher API 与 `TestPlan` 数据结构，工具执行测试变成"调 API、收事件"，不再各自解析框架内部结构。

> **设计争议**：拆出 Platform 的代价是概念数量翻倍——初学者要先理解 Platform/Engine/Launcher/TestPlan 才能明白"我的 @Test 是怎么跑起来的"，而 JUnit 4 的心智模型只是"Runner 反射找 @Test"。社区早期的抱怨集中于此。但从事后看这笔交换是划算的：JUnit 5 十年的演进（并行执行、动态测试、@Suite 替代品、Kotlin/Scala 友好性）全部受益于平台层的存在，Spock/TestNG 迁移到 Platform 也证明了 SPI 的通用性。

# 2. Platform 运行模型

## 2.1 核心概念

Platform 的四个核心概念构成一条流水线：**Launcher 是入口，DiscoverySelector 描述"找什么"，TestPlan 是发现结果的树，TestEngine 是真正执行者，ExecutionListener 接收执行事件**。

- **TestEngine**：测试执行引擎的 SPI。引擎实现两个关键方法——`discover(DiscoveryRequest, uniqueId)` 声明自己能发现哪些测试（返回 `TestDescriptor` 树），`execute(ExecutionRequest)` 执行之。`junit-jupiter-engine`、`junit-vintage-engine` 都是它的实现，通过 `ServiceLoader` 机制（`META-INF/services/org.junit.platform.engine.TestEngine` 文件）被 Platform 加载。
- **Launcher**：面向工具方的统一启动入口。`LauncherFactory.create()` 得到实例后，先 `discover` 后 `execute`，两步分离使得工具可以在执行前展示测试树（IDE 左侧的测试列表就来自这一步）。
- **DiscoverySelectors**：选择器的工厂类，把"要跑什么"表达成结构化对象——`selectClass`、`selectMethod`、`selectPackage`、`selectClasspathRoots` 等。IDE 里右键单个方法运行，本质就是 `selectMethod("com.Foo#bar")`。
- **TestPlan**：所有引擎发现结果的聚合树，节点是 `TestIdentifier`（稳定的 UniqueId + 显示名 + 是否为容器/测试）。它是**只读快照**，工具据此渲染 UI、决定并行度。

## 2.2 典型调用流程

完整流程是"发现阶段"与"执行阶段"的两段式，两阶段之间以 TestPlan 为交接物：

```text
┌──────────┐  LauncherDiscoveryRequest（选择器 + 过滤器）
│ Launcher │────────────────────────────────────┐
└────┬─────┘                                    ▼
     │                              ┌─────────────────────────┐
     │  discover()                  │ 遍历 ServiceLoader 加载的 │
     │                              │ 每个 TestEngine.discover │
     │                              └────────────┬────────────┘
     │                                           ▼
     │                                 TestPlan（聚合描述符树）
     │                                           │
     │  execute(plan, listeners)                 │
     ▼                                           ▼
┌──────────┐   executionStarted/Skipped/   ┌─────────────┐
│ Listener │◀──Finished 事件流 ────────────│ TestEngine  │
│(IDE/构建 │                               │  逐个执行    │
│ 工具/报告)│                               └─────────────┘
└──────────┘
```

用代码表达一遍，这段 Launcher API 就是 Surefire/Gradle/IDE 内部真正做的事：

```java
package com.example.launcher;

import org.junit.platform.launcher.Launcher;
import org.junit.platform.launcher.LauncherDiscoveryRequest;
import org.junit.platform.launcher.TestIdentifier;
import org.junit.platform.launcher.TestPlan;
import org.junit.platform.launcher.core.LauncherDiscoveryRequestBuilder;
import org.junit.platform.launcher.core.LauncherFactory;
import org.junit.platform.launcher.listeners.SummaryGeneratingListener;
import org.junit.platform.launcher.listeners.TestExecutionSummary;
import org.junit.platform.engine.discovery.DiscoverySelectors;

import java.io.PrintWriter;

import static org.junit.platform.engine.discovery.DiscoverySelectors.selectPackage;

public class RunTestsManually {

    public static void main(String[] args) {
        // 1. 构造发现请求：按包选择 + 按标签过滤
        LauncherDiscoveryRequest request = LauncherDiscoveryRequestBuilder.request()
                .selectors(selectPackage("com.example.service"))   // 要发现的范围
                .filters(org.junit.platform.launcher.TagFilter.includeTags("fast"))
                .build();

        // 2. 创建 Launcher，先发现、得到只读测试计划
        Launcher launcher = LauncherFactory.create();
        TestPlan plan = launcher.discover(request);
        System.out.println("发现测试数: " + plan.countTestIdentifiers(TestIdentifier::isTest));

        // 3. 注册监听器后执行，事件流实时回调
        SummaryGeneratingListener listener = new SummaryGeneratingListener();
        launcher.execute(plan, listener);

        // 4. 汇总结果：失败明细、计数，可直接打印或接入自定义报告
        TestExecutionSummary summary = listener.getSummary();
        summary.printTo(new PrintWriter(System.out));
        summary.printFailuresTo(new PrintWriter(System.out));
    }
}
```

**发现与执行分离是 Platform 设计里最值得注意的一点**：TestPlan 在发现阶段就完全确定，执行阶段只按树遍历，不会再"边跑边发现"。这让工具可以在执行前做任意分析（预估并行分片、生成执行计划缓存、按历史耗时排序），也是 JUnit 5 并行执行能安全实现的基础。

## 2.3 Launcher API 与构建工具的集成原理

Maven Surefire 与 Gradle 对 JUnit 5 的支持，本质是**各自内置了一个 Provider/适配器来调用 Launcher API**：

- Surefire 检测到 classpath 上有 `junit-platform-engine` 时启用 `surefire-junit-platform` provider，把 Maven 的 includes/excludes/groups 翻译成 DiscoveryRequest 的选择器与过滤器，注册监听器收集结果生成 XML/TXT 报告。这就是为什么 Surefire 2.22.0 之前完全无法运行 JUnit 5——provider 是后来才加的。
- Gradle 的 `useJUnitPlatform()` 同理：不开这个开关，Gradle 默认走 JUnit 4 路径，Jupiter 测试会被静默跳过（一个非常经典的"测试没跑但不报错"事故源）。
- IDE 通过 Launcher 先 discover 渲染测试树，用户点谁就把对应 UniqueId 作为选择器 execute。

```groovy
// build.gradle：启用 Platform 路径，并按 Tag 过滤
test {
    useJUnitPlatform {                       // 必须显式声明，否则 Jupiter 测试不执行
        includeTags 'fast & !flaky'          // Tag 表达式：支持 & | ! 与括号
    }
}
```

> **注意**：`useJUnitPlatform()` 与 JUnit 4 的 `useJUnit()` 是互斥路径。混合项目（部分 Jupiter、部分 Vintage）只要声明 `useJUnitPlatform()` 并让 `junit-vintage-engine` 在 classpath 上，两种测试都能被发现；反过来只配 `useJUnit()` 则 Jupiter 测试全部漏跑。排查"JUnit 5 测试数量为 0"的问题，第一反应就是检查这个开关。

# 3. Jupiter 编程模型

## 3.1 核心特性

Jupiter 是使用者直接接触的部分，相对 JUnit 4 的编程模型变化可以归成四类：

**注解驱动生命周期**：`@BeforeAll`/`@BeforeEach`/`@Test`/`@AfterEach`/`@AfterAll` 五段式与 JUnit 4 对应物一一映射（`BeforeClass`→`BeforeAll`、`Before`→`BeforeEach`），但测试类和方法**不再要求 public**，包级私有即可；异常处理也更宽松——`@BeforeEach` 抛异常不再导致整个类中止（详见第 9 篇）。注意 `org.junit.Test`（JUnit 4）与 `org.junit.jupiter.api.Test`（Jupiter）是两个不同的注解，import 错了测试会被静默忽略，这是迁移期最高频的事故。

**参数化与动态测试**：`@ParameterizedTest` + 多种数据源注解解决了 JUnit 4 Parameterized Runner 的独占与类型安全问题（见第 6、11 篇）；`@TestFactory` 允许在运行时生成测试节点，突破了"测试必须在编译期以注解形式存在"的限制。

**可组合扩展模型**：`Extension` 接口族（`BeforeAllCallback`、`BeforeEachCallback`、`ParameterResolver`、`TestExecutionExceptionHandler` 等十余个回调点）取代 Runner/Rule。多个扩展通过 `@ExtendWith` 并列声明、按序生效，Mockito 的 `MockitoExtension`、Spring 的 `SpringExtension`、超时的 `TimeoutExtension` 可以任意组合——这正是 JUnit 4 架构做不到的。

**可并行执行与资源锁**：Platform 层提供并行执行配置，Jupiter 用 `@ResourceLock`、`@Isolated` 声明测试间的互斥关系。并行能力放在平台层而非框架层，意味着任何引擎的测试都能纳入统一的并行调度。

> **行业实践**：Jupiter API 与 Engine 的依赖要区分场景——被测模块只依赖 `junit-jupiter-api`（编译期），运行时由构建工具统一提供 `junit-jupiter-engine`。直接引聚合包 `junit-jupiter` 对单模块项目最省事，多模块大仓则常把 engine 收敛到根构建脚本，避免版本漂移。

# 4. Vintage 兼容层

## 4.1 作用

`junit-vintage-engine` 的使命是**让 JUnit 3/4 测试原封不动地跑在 Platform 上**：它实现 TestEngine SPI，discover 阶段用 JUnit 4 自己的 `AllDefaultPossibilitiesBuilder` 逻辑扫描测试类（包括 JUnit 3 风格以 `test` 开头的用例），把 JUnit 4 的 `Description` 树翻译成 Platform 的 `TestDescriptor`；execute 阶段用 JUnit 4 的 `Runner` 体系真正运行，再把 `RunListener` 事件翻译回 Platform 的监听器事件。

这为迁移提供了**渐进式路径**：加上 Vintage 依赖后，新旧测试在同一次构建里被发现、执行、汇总进同一份报告，团队可以按模块甚至按文件逐个把 JUnit 4 测试改写成 Jupiter，而不需要"某个周末一次性切完"的大爆炸迁移。迁移期间唯一的构建配置要求是工具走 Platform 路径（Gradle `useJUnitPlatform()`、Surefire 2.22+）。

```xml
<!-- 混合期依赖组合：Jupiter 写新测试，Vintage 跑存量 JUnit 4 测试 -->
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.10.2</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.junit.vintage</groupId>
    <artifactId>junit-vintage-engine</artifactId>
    <version>5.10.2</version>              <!-- 版本号与 Jupiter 保持一致 -->
    <scope>test</scope>
</dependency>
```

## 4.2 限制

Vintage 的定位是"运行"而不是"增强"，边界必须清楚：

- **JUnit 4 测试无法使用 Jupiter 的任何新能力**：`@ExtendWith`、`@Nested`、`assertAll`、`@DisplayName`、Tag 表达式对 Vintage 测试统统无效。想给存量测试打 `@Tag`，必须先把它改写成 Jupiter。
- **JUnit 4 的 Runner/Rule 保持原模型**：Vintage 引擎把执行完全委托给 JUnit 4 的 Runner 体系，所以 Runner 独占问题原样存在——一个类仍不能同时用 `Parameterized` 和 Spring Runner。
- **JUnit 3 支持有版本门槛**：JUnit 5.9+ 的 Vintage 引擎要求 JUnit 4.12 以上；跑更老的 JUnit 3.8 需要显式引入 `junit:junit:3.8.1` 且部分特性（如 Assume）不可用。
- **@Category 与 @Tag 两套体系并存**：Surefire/Gradle 的类别过滤对 Vintage 测试查 `@Category`，对 Jupiter 测试查 `@Tag`，混合期过滤规则要两边都配（见第 7 篇）。

> **行业实践**：不少团队在迁移完成后会**主动移除 vintage-engine 并加一条 ArchUnit/构建检查**，防止有人无意间 import `org.junit.Test` 写回 JUnit 4 风格——Vintage 在 classpath 上时这类测试照样能跑，问题会被长期掩盖，直到某天需要删 Vintage 时才发现存量又涨了。

# 5. 与 JUnit 4 Runner 架构的对比

把两代架构并排看，能更清楚 Platform 拆分换来了什么：

| 维度 | JUnit 4 | JUnit 5 |
|------|---------|---------|
| 核心抽象 | Runner（一个类一个执行器） | TestEngine（平台级）+ Extension（回调点级） |
| 扩展方式 | `@RunWith` 整体替换执行模型 | `@ExtendWith` 多扩展并存组合 |
| 扩展组合 | 不可叠加（独占槽位），催生 Rules 作补充 | Extension 任意叠加，Rule 由 Extension 替代 |
| 工具集成 | 各工具自行反射扫描、解析框架内部结构 | 统一 Launcher API + TestPlan + 事件流 |
| 多框架支持 | 每种框架一套独立的构建工具适配 | Spock/TestNG/Cucumber 都以引擎跑在同一 Platform |
| 测试发现 | 与执行耦合，边扫边跑 | 发现/执行两阶段分离，TestPlan 只读快照 |
| 兼容旧测试 | — | Vintage 引擎，存量测试零改动运行 |
| 可见性要求 | 类与方法必须 public | 包级私有即可 |

架构演进的主线可以概括为一句话：**JUnit 4 把"如何运行测试"和"如何编写测试"耦合在一个 jar 里，扩展只能靠替换执行器；JUnit 5 把前者下沉为 Platform 协议、后者上浮为 Jupiter API，扩展变成协议内的插件**。JUnit 4 的 Rule 机制其实已经意识到了 Runner 独占的问题并试图缓解，但 Rule 只能包裹方法执行这一个点；Extension 的十余个回调点（参数解析、异常处理、条件求值、模板拦截……）才是完整的答案。

# 6. 小结

| 模块/概念 | 职责 | 关键点 |
|-----------|------|--------|
| JUnit Platform | 测试发现与执行的基础设施 | 只定协议不写测试逻辑，Tool 与 Engine 两侧面向协议编程 |
| JUnit Jupiter | JUnit 5 编程与扩展模型 | api（编译期）与 engine（运行期）分离，日常写测试用它 |
| JUnit Vintage | JUnit 3/4 兼容引擎 | 存量测试零改动运行，但享受不到 Jupiter 任何新特性 |
| TestEngine SPI | 引擎接入点 | ServiceLoader 加载；Jupiter/Vintage/Spock/TestNG 均为实现 |
| Launcher API | 工具方入口 | discover 与 execute 两阶段分离，TestPlan 为只读快照 |
| DiscoverySelectors | "跑什么"的结构化描述 | IDE 右键单方法运行 = selectMethod |
| 工具集成原理 | Surefire provider / Gradle useJUnitPlatform | Gradle 忘开开关则 Jupiter 测试静默不跑 |
| 对比 JUnit 4 | Runner 独占 → Extension 组合 | 扩展从"替换执行器"变为"生命周期回调点挂载插件" |

下一篇进入 Jupiter 编程模型内部，展开生命周期注解与测试实例模型（第 9 篇）。
