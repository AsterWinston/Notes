# 1. TestContext Framework 概览

## 1.1 核心定位

TestContext Framework 是 Spring Test 的基础设施，负责把**测试生命周期**与**ApplicationContext 生命周期**连接起来。它解决了以下问题：

- 测试类如何加载 Spring 容器
- 测试方法执行前后如何做依赖注入、事务管理、回滚、SQL 脚本执行
- ApplicationContext 如何缓存以减少启动成本

## 1.2 关键角色

| 组件 | 作用 |
|------|------|
| `TestContextManager` | 驱动测试生命周期与监听器链 |
| `TestContext` | 测试执行时上下文信息载体 |
| `ContextLoader` | 负责创建 ApplicationContext |
| `TestExecutionListener` | 插件式生命周期扩展 |

---

# 2. 启动与加载流程

## 2.1 ContextBootstrapper

`ContextBootstrapper` 负责解析测试类上的元信息，构建 `MergedContextConfiguration`，并选择合适的 `ContextLoader`。

在 Spring Boot 中，默认使用 `SpringBootTestContextBootstrapper`。

## 2.2 Spring Boot 的 ContextLoader

`SpringBootContextLoader` 会：

1. 构建 `SpringApplication`
2. 应用环境与属性
3. 创建 `ApplicationContext`
4. 执行自动配置

**关键价值**：确保测试与生产启动路径一致。

---

# 3. TestExecutionListener 机制

## 3.1 默认监听器（常见）

| 监听器 | 作用 |
|--------|------|
| `DependencyInjectionTestExecutionListener` | 为测试类注入依赖 |
| `DirtiesContextTestExecutionListener` | 处理 `@DirtiesContext` |
| `TransactionalTestExecutionListener` | 事务管理与回滚 |
| `SqlScriptsTestExecutionListener` | 执行 `@Sql` 脚本 |
| `ServletTestExecutionListener` | Servlet 相关上下文准备 |

## 3.2 自定义监听器

```java
@TestExecutionListeners(
    value = CustomListener.class,
    mergeMode = TestExecutionListeners.MergeMode.MERGE_WITH_DEFAULTS
)
class CustomListenerTest {}
```

使用场景：需要在测试前后设置线程上下文、审计信息或外部依赖。

---

# 4. 依赖注入与测试实例

## 4.1 测试类注入

Spring 可以像业务类一样向测试类注入 Bean：

```java
@SpringBootTest
class UserServiceTest {

    @Autowired
    private UserService userService;
}
```

## 4.2 测试专用配置

```java
@TestConfiguration
static class TestConfig {
    @Bean
    public Clock fixedClock() {
        return Clock.fixed(Instant.parse("2024-01-01T00:00:00Z"), ZoneOffset.UTC);
    }
}
```

---

# 5. 源码视角理解

## 5.1 关键对象关系

```text
TestContextManager
  └── TestContext
        └── MergedContextConfiguration
             └── ContextLoader
                  └── ApplicationContext
```

## 5.2 设计动机

- 通过 TestContext 把“测试元数据”与“上下文实例”解耦
- 通过监听器链提供扩展点，避免测试框架与业务框架强耦合
