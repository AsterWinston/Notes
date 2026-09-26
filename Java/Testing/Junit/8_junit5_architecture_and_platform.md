# 1. JUnit 5 架构概览

JUnit 5 由三个模块构成：

- **JUnit Platform**：测试发现、执行与运行时基础设施。
- **JUnit Jupiter**：新的编程与扩展模型（JUnit 5 API）。
- **JUnit Vintage**：兼容 JUnit 3/4 的引擎。

## 1.1 模块化设计目标

- 解决 JUnit 4 的扩展能力受限问题。
- 支持多引擎并存（JUnit、Spock、TestNG）。
- 让测试发现与执行对 IDE/构建工具统一。

# 2. Platform 运行模型

## 2.1 核心概念

- **TestEngine**：测试执行引擎。
- **Launcher**：统一启动入口。
- **DiscoverySelectors**：选择测试类/方法。
- **TestPlan**：测试计划树。

## 2.2 典型调用流程

```
Launcher -> DiscoveryRequest -> TestPlan
  -> EngineExecutionListener -> Execution
```

# 3. Jupiter 编程模型

## 3.1 核心特性

- 注解驱动生命周期
- 参数化测试与动态测试
- 可组合扩展模型（Extension）
- 可并行执行与资源锁

# 4. Vintage 兼容层

## 4.1 作用

- 让 JUnit 4 测试在 JUnit 5 平台上运行。
- 为迁移提供渐进式路径。

## 4.2 限制

- 无法直接使用 Jupiter 的扩展能力。
- JUnit 4 的 Runner/Rule 仍保持原模型。

# 5. 架构优势总结

- **统一入口**：构建工具与 IDE 只需对接 Platform。
- **高扩展性**：多引擎共存，避免单一实现锁定。
- **高可组合性**：Extension 替代 Runner/Rule，组合更灵活。
