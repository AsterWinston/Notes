# 1. CI 测试分层与命名

## 1.1 命名约定

- 单元测试：`*Test`
- 集成测试：`*IT`

## 1.2 Maven Surefire / Failsafe

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.5</version>
</plugin>
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-failsafe-plugin</artifactId>
    <version>3.2.5</version>
</plugin>
```

Surefire 执行单元测试，Failsafe 执行集成测试（`*IT`）。

## 1.3 Gradle 测试任务

```gradle
tasks.test {
    useJUnitPlatform()
}
```

可自定义 `integrationTest` 任务执行集成测试。

---

# 2. 标签与过滤

## 2.1 JUnit 标签

```java
@Tag("slow")
@Test
void slowTest() {}
```

## 2.2 构建工具过滤

- Maven: `-Dgroups=slow` 或 `-Dtest=*Test`
- Gradle: `--tests "*IT"`

---

# 3. Flaky Test 处理策略

- 首选修复根因（时序、并发、外部依赖）
- 不建议长期依赖重试机制掩盖问题

---

# 4. 常见问题排查

## 4.1 Context 加载失败

| 现象 | 常见原因 | 解决 |
|------|----------|------|
| `Failed to load ApplicationContext` | Bean 缺失 | 使用 `@MockBean` 或补全配置 |
| `Port already in use` | 固定端口冲突 | 使用 `RANDOM_PORT` |
| `NoSuchBeanDefinitionException` | 切片测试缺少 Bean | `@Import` 或 `@MockBean` |

## 4.2 数据库相关错误

| 现象 | 原因 | 解决 |
|------|------|------|
| 表不存在 | 未执行迁移 | 打开 Flyway/Liquibase |
| 数据污染 | 回滚失效 | 检查事务配置 |

## 4.3 Profile 不生效

- 未设置 `@ActiveProfiles`
- 配置文件名称错误

---

# 5. 面试与高频要点

## 5.1 高频问题

1. `@SpringBootTest` 与切片测试的区别
2. TestContext 缓存机制与 `@DirtiesContext`
3. `@MockBean` 与 `@Mock` 的差异
4. 为什么测试默认回滚
5. MockMvc 与 TestRestTemplate 选型

## 5.2 回答思路

- 说明目的与边界
- 解释底层机制
- 给出场景与取舍
