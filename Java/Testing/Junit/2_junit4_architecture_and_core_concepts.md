# 1. JUnit 4 架构概览

JUnit 4 的核心思想是**通过 Runner 驱动测试类执行**。测试类被 Runner 转换为一组可执行的 Statement，并在生命周期钩子之间执行。

## 1.1 关键角色

- **Runner**：执行测试的引擎，默认是 `BlockJUnit4ClassRunner`。
- **ParentRunner**：Runner 的通用模板，负责子节点遍历和执行。
- **Statement**：可执行的测试节点（前置、后置、测试方法）。
- **Description**：测试描述信息（用于报告和筛选）。
- **RunNotifier**：事件发布器（开始、结束、失败等）。

## 1.2 典型执行流程

```
Request -> Runner -> ParentRunner.run()
  -> classBlock() -> withBeforeClasses
  -> withAfterClasses -> childrenInvoker
  -> runChild() -> methodBlock
  -> withBefores -> withAfters -> invokeMethod
```

**要点**：JUnit 4 的生命周期是通过 Statement 的嵌套组合实现的。

## 1.3 测试发现与过滤

- 使用反射扫描测试类的 `@Test` 方法。
- 通过 `Request` 与 `Filter` 支持选择性执行。
- Suite 与 Category 通过额外 Runner 增强发现能力。

# 2. 核心概念与模型

## 2.1 测试类（Test Class）

- 公共构造函数、无参构造。
- 测试方法为 `public void`，且标注 `@Test`。

## 2.2 Fixture 与生命周期

- **Fixture**：测试执行所需的环境和数据。
- `@Before/@After` 负责每个测试方法的前后置。
- `@BeforeClass/@AfterClass` 负责测试类级别。

## 2.3 断言与假设

- **断言**：验证结果是否符合预期（`Assert`）。
- **假设**：在条件不满足时跳过测试（`Assume`）。

# 3. 扩展模型

## 3.1 Runner 扩展

通过 `@RunWith` 切换 Runner，如：

```java
@RunWith(Parameterized.class)
public class ParameterizedTest { }
```

## 3.2 Rule 扩展

`Rule` 基于装饰器模式插入额外逻辑，常见用途：

- 临时文件与资源管理
- 超时控制
- 期望异常
- 外部资源启动与销毁

# 4. 架构优缺点

## 4.1 优点

- 扩展点清晰，Runner/Rule 能覆盖多数需求。
- 与 Java 生态工具兼容性好。

## 4.2 局限

- Runner 只能单一生效，组合困难。
- Rule 体系与 Runner 体系分离，模型不统一。
- 对并发/参数化/动态测试支持有限。

# 5. 与 JUnit 5 的差异简述

JUnit 5 用 Platform/Jupiter/Vintage 的**模块化架构**取代单一 Runner 模式，扩展能力与可组合性大幅提升。
