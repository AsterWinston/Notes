# 1. 测试配置覆盖体系

## 1.1 配置优先级要点

Spring Boot 的配置来源有明确优先级，测试中常见覆盖方式：

1. `@SpringBootTest(properties = ...)`
2. `@TestPropertySource`
3. `@DynamicPropertySource`
4. `application-test.yml`（配合 `@ActiveProfiles`）

优先级靠前者会覆盖靠后的配置。

---

# 2. `@ActiveProfiles`

## 2.1 作用

激活指定 Profile，从而加载 `application-{profile}.yml`。

```java
@ActiveProfiles("test")
@SpringBootTest
class ProfileTest {}
```

## 2.2 常见约定

- `application-test.yml`：测试默认配置
- `application-local.yml`：开发本地配置

---

# 3. `@TestPropertySource`

用于在测试类级别覆盖配置。

```java
@TestPropertySource(properties = "app.featureX=true")
class PropertyOverrideTest {}
```

也可以指定外部配置文件：

```java
@TestPropertySource(locations = "classpath:custom-test.yml")
```

---

# 4. `@DynamicPropertySource`

适合运行时动态属性，例如随机端口、动态生成的连接信息。

```java
@DynamicPropertySource
static void register(DynamicPropertyRegistry registry) {
    registry.add("app.token", () -> UUID.randomUUID().toString());
}
```

注意：动态属性会参与 Context 缓存 Key 计算。

---

# 5. `@TestConfiguration`

用于提供测试专用 Bean：

```java
@TestConfiguration
static class TestConfig {
    @Bean
    public Clock fixedClock() {
        return Clock.systemUTC();
    }
}
```

适合替换外部依赖或固定时间。

---

# 6. `@Import` 与组合

```java
@SpringBootTest
@Import(TestConfig.class)
class ImportTest {}
```

当测试类不在启动类同包或子包时，可显式导入配置类。

---

# 7. 常见问题

## 7.1 配置未生效

- Profile 未激活
- 属性优先级被覆盖
- Context 缓存导致旧配置复用

## 7.2 配置泄漏

多个测试类共享同一 Context 时，属性覆盖需一致，否则会触发新容器。
