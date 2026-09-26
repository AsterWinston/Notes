# 1. ApplicationContext 缓存机制

## 1.1 为什么需要缓存

启动 Spring Boot 容器成本高：

- 自动配置扫描与 Bean 实例化
- 配置属性绑定与条件判断
- Web 容器或数据源初始化

缓存 ApplicationContext 可以显著减少测试时间。

## 1.2 缓存 Key 的组成

TestContext 使用 `MergedContextConfiguration` 作为缓存 Key，关键组成包括：

- 测试类与 `@SpringBootTest` 配置
- `@ActiveProfiles` 与环境属性
- `@ContextConfiguration` / `@Import` / `@TestConfiguration`
- `@DynamicPropertySource` 的动态值

**结论**：哪怕是属性差异，也会导致新容器。

## 1.3 缓存大小与淘汰

- Spring 的 `ContextCache` 有默认大小限制
- 超过阈值会触发 LRU 淘汰
- 大量不同配置的测试类会导致缓存频繁失效

---

# 2. `@DirtiesContext` 的语义

## 2.1 作用与代价

`@DirtiesContext` 表示当前测试会污染容器状态，需要在执行后**销毁并重建**。代价是缓存失效与重启。

## 2.2 常见使用方式

```java
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_CLASS)
class DirtyContextTest {}
```

| 模式 | 说明 |
|------|------|
| `AFTER_METHOD` | 每个测试方法后销毁 |
| `AFTER_CLASS` | 测试类结束后销毁 |
| `BEFORE_METHOD` | 每个方法前销毁 |

---

# 3. Context Hierarchy 与缓存

## 3.1 父子上下文

在复杂系统中可构建父子上下文以复用公共 Bean。

```text
ParentContext (公共配置)
  └── ChildContext (测试专属)
```

## 3.2 适用场景

- 多个测试共享公共基础设施
- 需要隔离业务模块与基础模块

---

# 4. 避免过度 DirtiesContext 的策略

## 4.1 使用事务回滚

- 默认使用 `@Transactional` 在测试完成后回滚
- 避免因数据污染而重建容器

## 4.2 使用测试切片

通过切片减少容器复杂度，降低污染概率。

## 4.3 使用替身与 TestConfiguration

避免在测试中修改真实 Bean 的状态：

```java
@MockBean
private ExternalClient externalClient;
```

---

# 5. 性能优化建议

## 5.1 最小化上下文

- `@SpringBootTest(classes = MinimalConfig.class)`
- 移除不必要的 `@ComponentScan`

## 5.2 合理使用 Profile

- `@ActiveProfiles("test")`
- 限定数据源与外部依赖

## 5.3 延迟初始化

```yaml
spring:
  main:
    lazy-initialization: true
```

适合提高测试启动速度，但可能延迟暴露问题。
