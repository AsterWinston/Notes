# 1. Spring Boot Test 体系

Spring Boot 提供 `spring-boot-starter-test`，内置 JUnit 5、Mockito、AssertJ、Spring Test 等依赖。

## 1.1 依赖引入

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-test</artifactId>
  <scope>test</scope>
</dependency>
```

# 2. 核心测试注解

## 2.1 `@SpringBootTest`

加载完整 Spring 容器，适合端到端或集成测试。

```java
@SpringBootTest
class AppTest { }
```

## 2.2 Slice 测试

- `@WebMvcTest`：只加载 MVC 层
- `@DataJpaTest`：只加载 JPA 相关
- `@RedisTest`：只加载 Redis 相关

# 3. Context 缓存与性能

## 3.1 缓存机制

Spring Test 默认缓存 ApplicationContext，用于复用减少启动成本。

## 3.2 性能优化建议

- 使用 Slice 测试减少启动组件
- 避免在测试中修改 `@DirtiesContext`

# 4. 测试配置与 Profile

```java
@SpringBootTest
@ActiveProfiles("test")
class UserServiceTest { }
```

# 5. 常见陷阱

## 5.1 过度使用 `@SpringBootTest`

会导致测试执行过慢，应优先使用 Slice 或 Mock。

## 5.2 MockBean 滥用

`@MockBean` 会替换容器中的 Bean，使用前需明确测试边界。
