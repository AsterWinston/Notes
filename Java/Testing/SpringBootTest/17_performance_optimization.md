# 1. 性能瓶颈来源

## 1.1 上下文启动成本

- 自动配置类扫描
- Bean 实例化与依赖注入
- 数据源初始化

## 1.2 测试粒度过大

将所有测试都写成 `@SpringBootTest` 是最常见的性能陷阱。

---

# 2. 降低启动成本的方法

## 2.1 优先使用测试切片

- `@WebMvcTest` / `@DataJpaTest`
- 只加载必要 Bean

## 2.2 限定配置类

```java
@SpringBootTest(classes = MinimalConfig.class)
```

## 2.3 排除自动配置

```java
@SpringBootTest
@EnableAutoConfiguration(exclude = DataSourceAutoConfiguration.class)
class NoDbTest {}
```

## 2.4 延迟初始化

```yaml
spring:
  main:
    lazy-initialization: true
```

---

# 3. 并行测试

## 3.1 JUnit 5 并行配置

```properties
junit.jupiter.execution.parallel.enabled=true
junit.jupiter.execution.parallel.mode.default=concurrent
```

## 3.2 风险点

- 共享静态状态
- 共享外部资源（端口、文件、数据库）

---

# 4. 稳定性与可重复性

## 4.1 避免依赖测试顺序

- 默认不保证顺序
- 如需顺序，应显式使用 `@TestMethodOrder`

## 4.2 清理策略

- 数据库使用事务回滚或 `@Sql` 清理
- 文件系统使用 `@TempDir`

---

# 5. 实践建议

- 将慢测试标记为 `@Tag("slow")`
- 在 CI 中单独执行慢测试
- 保持测试数量与反馈时间平衡
