# 1. 测试事务模型

## 1.1 默认事务行为

在 Spring Test 中，标注 `@Transactional` 的测试默认会在**测试结束后回滚**。

```java
@Transactional
@SpringBootTest
class TransactionTest {}
```

## 1.2 事务监听器

`TransactionalTestExecutionListener` 负责：

- 开启事务
- 测试结束回滚
- 处理 `@Rollback` 与 `@Commit`

---

# 2. `@Rollback` 与 `@Commit`

## 2.1 强制回滚

```java
@Rollback
@Test
void rollbackTest() {}
```

## 2.2 强制提交

```java
@Commit
@Test
void commitTest() {}
```

适合验证提交后行为（如触发 DB 触发器）。

---

# 3. 传播与隔离

## 3.1 常见传播行为

| 传播级别 | 说明 |
|----------|------|
| `REQUIRED` | 默认，加入现有事务 |
| `REQUIRES_NEW` | 新建事务，挂起外部事务 |
| `NOT_SUPPORTED` | 非事务执行 |

## 3.2 测试中的意义

- 使用 `REQUIRES_NEW` 验证事务边界
- 使用 `NOT_SUPPORTED` 验证非事务查询

---

# 4. `@Sql` 脚本支持

```java
@Sql(scripts = "classpath:cleanup.sql", executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
class SqlCleanupTest {}
```

可在测试前后准备或清理数据。

---

# 5. 事务与 `@DirtiesContext`

- 事务回滚通常比 `@DirtiesContext` 成本低
- 只有在修改 Bean 状态导致污染时才使用 `@DirtiesContext`

---

# 6. 多线程与异步场景

- 事务上下文与线程绑定，异步线程默认不会继承事务
- 若需验证异步提交效果，应明确等待异步执行完成
