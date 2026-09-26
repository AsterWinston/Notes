# 1. Extension 模型概览

JUnit 5 的 Extension 用于替代 JUnit 4 的 Runner/Rule，提供可组合、可复用的扩展能力。

## 1.1 主要接口

- `BeforeAllCallback` / `AfterAllCallback`
- `BeforeEachCallback` / `AfterEachCallback`
- `ParameterResolver`
- `TestExecutionExceptionHandler`
- `ExecutionCondition`

# 2. 扩展注册方式

## 2.1 `@ExtendWith`

```java
@ExtendWith(TimingExtension.class)
class UserServiceTest { }
```

## 2.2 自动注册

- `@RegisterExtension`
- `junit.jupiter.extensions.autodetection.enabled=true`

# 3. ExtensionContext

ExtensionContext 是扩展的核心上下文，提供：

- 当前测试/容器信息
- Store（类似 Map，用于跨回调共享数据）
- 测试唯一标识

## 3.1 Store 示例

```java
public void beforeEach(ExtensionContext context) {
    context.getStore(ExtensionContext.Namespace.GLOBAL)
        .put("start", System.nanoTime());
}
```

# 4. 自定义扩展示例

```java
public class TimingExtension implements BeforeEachCallback, AfterEachCallback {
    @Override
    public void beforeEach(ExtensionContext context) {
        context.getStore(ExtensionContext.Namespace.GLOBAL)
            .put("start", System.nanoTime());
    }

    @Override
    public void afterEach(ExtensionContext context) {
        long start = context.getStore(ExtensionContext.Namespace.GLOBAL)
            .get("start", long.class);
        long cost = System.nanoTime() - start;
        System.out.println(context.getDisplayName() + " cost=" + cost);
    }
}
```

# 5. 扩展顺序与组合

- 通过 `@Order` 控制扩展执行顺序。
- 多个扩展可叠加，不再受 Runner 只能单一生效限制。
