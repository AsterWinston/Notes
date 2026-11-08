# 1. 测试套件（Suite）

随着测试类数量增长，"跑全部测试"变得又慢又粗：提交前只想跑受影响模块，发版前想跑全量回归，夜间任务想跑耗时的集成测试。JUnit 4 提供的组织手段是 **Suite**——把若干测试类聚合进一个"套件类"，套件本身也可以作为普通测试类被执行，从而支持嵌套和批量调度。它与 Category 一个管"静态聚合"、一个管"动态过滤"，是 JUnit 4 时代测试组织的两条腿。

## 1.1 基本用法

Suite 同样建立在 Runner 机制上：`@RunWith(Suite.class)` 声明这是套件，`@SuiteClasses` 列出成员测试类。套件类本身没有方法体，它只是一个"清单"：

```java
package com.example.suite;

import org.junit.runner.RunWith;
import org.junit.runners.Suite;
import org.junit.runners.Suite.SuiteClasses;

@RunWith(Suite.class)                       // 使用 Suite Runner
@SuiteClasses({                             // 成员测试类清单
    UserServiceTest.class,
    OrderServiceTest.class,
    InventoryServiceTest.class
})
public class ServiceTestSuite {
    // 套件类不需要任何方法体，仅作为聚合入口
}
```

在 IDE 中直接运行 `ServiceTestSuite`，或在 Maven 中把它当作普通测试类指定执行，三个成员类的所有 `@Test` 就会按声明顺序依次执行。

**套件可以嵌套**：`@SuiteClasses` 的成员既可以是普通测试类，也可以是另一个 Suite 类，由此拼出"模块套件 → 子系统套件 → 全量回归套件"的树状结构。另外，如果成员类自身标注了别的 Runner（如 `@RunWith(Parameterized.class)`），Suite 会尊重成员的 Runner——套件只负责聚合调度，不改变成员的执行模型。

`@FixMethodOrder` 可以配合套件约束成员类内部方法的执行顺序，但**类与类之间的顺序由 `@SuiteClasses` 的声明顺序决定**，JVM 不保证反射枚举类的顺序稳定，这也是把顺序敏感的场景写进 Suite 声明数组而不是散在别处的原因。

## 1.2 适用场景

- **回归测试集合**：把发版前必须全绿的测试固化成一个 `RegressionSuite`，CI 里一条命令跑完。
- **按模块分组执行**：每个 Maven 模块或业务域一个套件，提交阶段只跑受影响模块的套件，缩短反馈时间。
- **冒烟测试子集**：从全量测试中挑出最核心的一小撮组成 `SmokeSuite`，部署后快速验证。

## 1.3 套件模式的弊端

Suite 看起来只是"清单 + 批量执行"，但它在工程实践中的口碑很差，原因都来自**清单是手工维护的静态代码**：

- **双重维护成本**：每新增一个测试类，都要记得把它加进相关套件。忘了加，这个类就永远不会被回归套件跑到——而且没有任何机制提醒你它被漏掉了，是一种静默的覆盖率漏洞。
- **重复执行**：一个测试类可以同时属于 `ModuleASuite` 和 `RegressionSuite`，如果 CI 分别跑两个套件，它就被执行两遍，浪费时间；如果只跑大套件，模块级的快速反馈又没了。
- **与构建工具的天然机制冲突**：Maven/Gradle 本来就会自动发现 classpath 下所有测试类，套件等于在自动发现之上又叠了一层手工白名单，方向是反的。
- **无法表达交叉维度**："慢测试""需要数据库的测试""不稳定的测试"这些横切维度用套件表达，就要为每个维度 × 每个模块的组合各写一个套件类，类数量爆炸。

> **行业实践**：现代构建流程基本不用 Suite 类，而是用构建工具的自动发现 + 过滤规则（Surefire 的 `includes`、Gradle 的 tag/category 过滤器）来组织测试执行。"哪些测试属于哪个批次"由命名约定、目录结构或标签声明，而不是由一份手工清单声明——清单会腐化，约定和标签跟着代码走。

Suite 至今保留的价值主要是**在 IDE 里一键跑一组相关测试**，以及老项目遗留。新项目应优先考虑第 2 节的 Category（JUnit 4）或 JUnit 5 的 `@Tag`。

# 2. Category 分类

Category 是 JUnit 4.8 引入的机制，解决的问题正是 Suite 解决不好的那个：**按"性质"而非按"清单"选择测试**。做法是先声明若干个空的标记接口作为类别，再给测试类或方法打上 `@Category` 注解，执行时用过滤器按类别包含或排除。类别是横切维度，一个测试可以同时属于多个类别，且新增测试只要打好注解就自动进入过滤范围，没有清单要维护。

## 2.1 定义与使用

类别本身就是空接口，习惯上集中放在一个包里统一管理：

```java
package com.example.category;

public interface SlowTest { }               // 耗时测试：不适合每次提交都跑

public interface FlakyTest { }              // 不稳定测试：结果不可信，需隔离观察

public interface DatabaseTest { }           // 依赖数据库：本地无库环境应跳过
```

`@Category` 可以标注在类上（整个类生效）或方法上（仅该方法生效），值是一个类别接口的 Class，可以写多个：

```java
package com.example.service;

import com.example.category.DatabaseTest;
import com.example.category.SlowTest;
import org.junit.Test;
import org.junit.experimental.categories.Category;

@Category(DatabaseTest.class)               // 类级别：整个类都属于数据库测试
public class LargeDataTest {

    @Test
    public void should_load_million_rows() {
        // ... 耗时逻辑
    }

    @Test
    @Category(SlowTest.class)               // 方法级别：额外再标记为慢测试
    public void should_rebuild_index() {
        // ... 更耗时的逻辑
    }

    @Test
    public void should_count_rows() {
        // 只属于 DatabaseTest（继承类上的标注）
    }
}
```

标注是**叠加而不是覆盖**：`should_rebuild_index` 同时属于 `DatabaseTest` 和 `SlowTest` 两个类别。这正是 Category 相对 Suite 的核心优势——类别之间正交，一次标注可被任意多个过滤维度复用。

## 2.2 与 Suite 结合：Categories Runner

单独打注解不产生任何效果，执行时必须挂上过滤器。JUnit 4 的方式是 `Categories` Runner（它继承自 Suite Runner，所以写法上仍是套件形态）：

```java
package com.example.suite;

import com.example.category.FlakyTest;
import com.example.category.SlowTest;
import org.junit.experimental.categories.Categories;
import org.junit.experimental.categories.Categories.ExcludeCategory;
import org.junit.experimental.categories.Categories.IncludeCategory;
import org.junit.runner.RunWith;
import org.junit.runners.Suite.SuiteClasses;

@RunWith(Categories.class)                  // Categories 是 Suite 的子类
@IncludeCategory(SlowTest.class)            // 包含：慢测试
@ExcludeCategory(FlakyTest.class)           // 排除：不稳定测试
@SuiteClasses({                             // 候选范围仍要手工列清单
    LargeDataTest.class,
    ReportExportTest.class,
    BatchJobTest.class
})
public class SlowTestSuite { }
```

这个写法暴露了 Categories Runner 的尴尬：**过滤是动态的，但候选集合仍是 `@SuiteClasses` 的静态清单**——你只是把"手工挑类"变成了"手工挑类 + 自动挑类中的测试"，双重维护问题只解决了一半。

`@IncludeCategory` 与 `@ExcludeCategory` 各有默认语义细节：`IncludeCategory` 默认只匹配"恰好标注了该类别"的测试，设置 `matchAny = true` 后才对多类别测试按"任一命中"匹配；`ExcludeCategory` 默认是"任一命中即排除"（`matchAny = true`）。多类别叠加时这两个语义容易搞混，配置后务必用实际运行验证。

如果不想写 Suite 类，也可以用 `Request.filterWith` 在编程式调用中动态过滤，或者直接用下面 3.1/3.2 的构建工具配置——后者才是主流做法。

# 3. 执行过滤

真正让 Category 发挥价值的是构建工具集成：Maven Surefire 和 Gradle 都能直接按类别过滤，**无需编写任何 Suite 类**，过滤规则写在构建脚本里、可以用命令行参数覆盖，这才补上了 Categories Runner 缺的"动态候选集"。

## 3.1 Maven Surefire 过滤

Surefire 通过 `<groups>` 与 `<excludedGroups>` 配置包含/排除的类别，值是类别接口的全限定名：

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.5</version>
    <configuration>
        <!-- 包含：只跑 SlowTest 类别；多个类别逗号分隔 -->
        <groups>com.example.category.SlowTest</groups>
        <!-- 排除：FlakyTest 类别一律不跑 -->
        <excludedGroups>com.example.category.FlakyTest</excludedGroups>
    </configuration>
</plugin>
```

更常用的是把类别绑到 Maven Profile 上，让"提交跑快测、夜间跑慢测"变成切换 Profile 的事：

```xml
<profiles>
    <profile>
        <id>fast</id>                       <!-- 默认：排除慢测试与不稳定测试 -->
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
        <build>
            <plugins>
                <plugin>
                    <artifactId>maven-surefire-plugin</artifactId>
                    <configuration>
                        <excludedGroups>com.example.category.SlowTest,com.example.category.FlakyTest</excludedGroups>
                    </configuration>
                </plugin>
            </plugins>
        </build>
    </profile>
    <profile>
        <id>nightly</id>                    <!-- 夜间：全量执行，包括慢测试 -->
        <build>
            <plugins>
                <plugin>
                    <artifactId>maven-surefire-plugin</artifactId>
                    <configuration>
                        <excludedGroups>com.example.category.FlakyTest</excludedGroups>
                    </configuration>
                </plugin>
            </plugins>
        </build>
    </profile>
</profiles>
```

```bash
mvn test                  # 走默认 fast profile，跳过慢测试
mvn test -P nightly       # 夜间任务：激活 nightly profile 跑全量
```

## 3.2 Gradle 过滤

Gradle 在 `test` 任务的 `useJUnit` 块中配置 `includeCategories` / `excludeCategories`，并同样建议把慢测试拆成独立任务：

```groovy
test {
    useJUnit {
        includeCategories 'com.example.category.SlowTest'      // 只跑慢测试
        excludeCategories 'com.example.category.FlakyTest'     // 排除不稳定测试
    }
}

// 更常见的组织方式：默认 test 排除慢测试，另建 slowTest 任务专跑慢测试
tasks.register('slowTest', Test) {
    useJUnit {
        includeCategories 'com.example.category.SlowTest'
    }
    // 与主任务区分输出目录，避免报告互相覆盖
    reports.html.outputLocation = file("$buildDir/reports/slow-tests")
}

test {
    useJUnit {
        excludeCategories 'com.example.category.SlowTest'
        excludeCategories 'com.example.category.FlakyTest'
    }
}
```

```bash
./gradlew test        # 快速反馈：不含慢测试
./gradlew slowTest    # 单独执行慢测试批次
```

**没有 `includeCategories` 时默认全包含，`excludeCategories` 无条件生效**。两者同时配置时先包含后排除。Gradle 6.8+ 对 JUnit 4 类别过滤还支持 `includeCategories`/`excludeCategories` 接收多个参数，一次排除多个类别不必写多个任务。

> **注意**：Surefire 与 Gradle 的类别过滤都依赖测试运行在对应的 JUnit 平台上。JUnit 4 项目里 Surefire 自动识别 Category；但如果项目混用 JUnit 4 和 JUnit 5（Vintage 引擎共存），`@Category` 只对 JUnit 4 测试生效，JUnit 5 测试要用 `@Tag` + `<groups>`（Surefire 对 Tag 同样走 groups 配置），两套标签体系并存期间要分别检查。

# 4. 常见问题

## 4.1 Category 只在 JUnit 4 生效

`@Category` 是 JUnit 4 独有的机制，**JUnit 5 的对应物是 `@Tag`**：同样是横切标签，但 `@Tag` 用字符串而非标记接口（`@Tag("slow")`），不需要预先定义类别接口，过滤通过 JUnit Platform 的 Tag 表达式（`groups`/`excludedGroups` 或 Gradle 的 `useJUnitPlatform { includeTags 'slow' }`）完成。迁移到 JUnit 5 时 `@Category` 注解会被直接忽略——不报错、不过滤，静默失效，这是混合迁移期最隐蔽的坑。

两者的对照与迁移策略：

| 维度 | JUnit 4 `@Category` | JUnit 5 `@Tag` |
|------|--------------------|----------------|
| 标签形态 | 标记接口（编译期类型） | 字符串（无编译期检查） |
| 定义成本 | 每个类别一个接口文件 | 零，直接写 |
| 组合逻辑 | Include/Exclude 两维 | 完整 Tag 表达式：`slow & !flaky` |
| 过滤入口 | Categories Runner 或构建工具 | Launcher/构建工具原生支持 |
| 迁移方式 | — | 注解替换 + 构建配置从类别接口名改为标签字符串 |

迁移时把类别接口名直接映射成同名字符串标签（`SlowTest.class` → `@Tag("slow")`），构建脚本里的 `groups` 值从全限定类名改成标签名即可；标记接口文件在全部测试迁移完成后删除。

## 4.2 Suite 与 Runner 冲突

Suite 与 Categories 都建立在 `@RunWith` 上，因此继承了 Runner 机制的独占性：**一个测试类只能有一个 Runner**。冲突出现在两个方向：

一是成员类方向——被 Suite 聚合的成员类自身可以用别的 Runner（Suite 会尊重它），这没问题；二是套件类方向——`@RunWith(Categories.class)` 已经占用套件类的 Runner 槽位，无法再叠加其他 Runner。实践中更常见的困境是：想让某个"既是参数化测试、又需要按类别过滤"的类工作，`@RunWith(Parameterized.class)` 和 Categories 过滤无法在同一个类上共存，只能把过滤放到构建工具层面做。

```text
┌─────────────────────────────────────────────┐
│             JUnit 4 组织测试的两条路          │
│                                             │
│  静态聚合                    动态过滤        │
│  ┌─────────┐               ┌──────────┐    │
│  │  Suite  │──继承──────────▶│Categories│    │
│  └────┬────┘               └────┬─────┘    │
│       │                         │          │
│       ▼                         ▼          │
│  @SuiteClasses            @Category 注解    │
│  手工清单，需双重维护       类别横切，可叠加   │
│       │                         │          │
│       └────────┬────────────────┘          │
│                ▼                            │
│     都依赖 @RunWith（独占槽位）              │
│     都要求手工圈定候选类范围                  │
│                │                            │
│                ▼                            │
│   JUnit 5 的替代：构建工具自动发现 + @Tag     │
│   （候选集自动、标签动态、可表达式组合）       │
└─────────────────────────────────────────────┘
```

## 4.3 Category 不生效的排查顺序

配置了过滤却发现"该跑的没跑、不该跑的还在跑"，按以下顺序排查：

1. **类别接口是否同一份**：`groups` 里的全限定名与实际标注的接口必须是同一个类，多模块项目中同名接口被复制两份时过滤会静默失效。
2. **注解是否在正确层级**：`@Category` 标在方法上只对方法生效；想整类过滤必须标在类上。
3. **Runner 是否被替换**：Surefire 对 Category 的过滤依赖标准 JUnit 4 执行路径，自定义 Runner 可能绕过过滤逻辑。
4. **是否混入了 JUnit 5 测试**：Vintage 引擎下 JUnit 5 的 `@Tag` 测试不认 `@Category` 配置，反之亦然。

# 5. 小结

| 要点 | 结论 |
|------|------|
| Suite | `@RunWith(Suite.class)` + `@SuiteClasses` 静态清单聚合，可嵌套，尊重成员自己的 Runner |
| Suite 的弊端 | 手工清单双重维护、静默漏测、重复执行、无法表达横切维度，现代流程已基本弃用 |
| Category | 空标记接口 + `@Category` 注解，横切、可叠加、类/方法两级生效 |
| Categories Runner | 继承 Suite，过滤动态但候选集仍是静态清单，且 matchAny 语义易混 |
| Maven 过滤 | Surefire `<groups>`/`<excludedGroups>` 配 Profile，实现快测/夜测分批 |
| Gradle 过滤 | `useJUnit { includeCategories/excludeCategories }`，慢测试建议拆独立任务 |
| 迁移 | `@Category` 仅 JUnit 4 生效，JUnit 5 用字符串 `@Tag` + 标签表达式，混用期两套体系分别配置 |
| 选型建议 | 新代码不写 Suite 类，用构建工具自动发现 + 标签过滤组织测试批次 |
