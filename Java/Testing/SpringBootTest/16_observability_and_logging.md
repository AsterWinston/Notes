# 1. 日志与输出捕获

## 1.1 OutputCaptureExtension

Spring Boot 提供输出捕获扩展，用于断言日志或控制台输出：

```java
@ExtendWith(OutputCaptureExtension.class)
class LogTest {
    @Test
    void logShouldContain(CapturedOutput output) {
        assertThat(output.getOut()).contains("started");
    }
}
```

## 1.2 日志配置覆盖

在 `src/test/resources` 中配置 `logback-test.xml` 或 `log4j2-test.xml`。

```xml
<logger name="org.springframework" level="INFO"/>
```

---

# 2. 诊断与排错工具

## 2.1 提升上下文日志

```properties
logging.level.org.springframework.test.context=DEBUG
logging.level.org.springframework.boot.test=DEBUG
```

## 2.2 观察 Bean 加载

```properties
logging.level.org.springframework.beans=DEBUG
```

---

# 3. Actuator 在测试中的使用

## 3.1 启用端点

```properties
management.endpoints.web.exposure.include=health,info,metrics
```

## 3.2 通过 MockMvc 验证

```java
mockMvc.perform(get("/actuator/health"))
    .andExpect(status().isOk());
```

---

# 4. 观测与指标

- 在测试中可验证关键指标是否被记录
- 注意指标采集异步化带来的时序问题

---

# 5. 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| 日志缺失 | 日志级别过高 | 调低 `logging.level` |
| 输出断言不稳定 | 异步日志 | 强制同步或等待 |
