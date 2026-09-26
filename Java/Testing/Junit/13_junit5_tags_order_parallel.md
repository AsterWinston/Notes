# 1. Tag 与测试选择

JUnit 5 使用 `@Tag` 替代 JUnit 4 的 Category。

## 1.1 基本用法

```java
@Tag("slow")
@Test
void slow_test() { }
```

## 1.2 构建工具过滤

- Maven Surefire/Failsafe 的 `includeTags` / `excludeTags`
- Gradle `useJUnitPlatform { includeTags "slow" }`

# 2. 测试顺序控制

## 2.1 `@TestMethodOrder`

```java
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class OrderedTest {
    @Order(1)
    @Test
    void first() { }
}
```

**建议**：仅在顺序确实是业务要求时使用，避免依赖顺序。

# 3. 并行执行

JUnit 5 支持并行测试执行，但需关注共享资源竞争。

## 3.1 配置方式

```properties
junit.jupiter.execution.parallel.enabled=true
junit.jupiter.execution.parallel.mode.default=concurrent
junit.jupiter.execution.parallel.config.strategy=fixed
junit.jupiter.execution.parallel.config.fixed.parallelism=4
```

## 3.2 资源锁

```java
@ResourceLock(value = "db")
@Test
void access_db() { }
```

# 4. 并发风险与策略

- 避免共享可变静态状态
- 使用隔离的测试数据或事务回滚
- 对外部依赖使用资源锁或串行策略
