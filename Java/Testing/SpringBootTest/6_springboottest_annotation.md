# 1. `@SpringBootTest` 的定位

## 1.1 适用场景

- 需要验证多个组件协作的集成测试
- 需要验证配置是否正确加载
- 需要真实容器或 Web 环境

## 1.2 与其他注解的关系

| 注解 | 定位 | 是否启动完整上下文 |
|------|------|------------------|
| `@SpringBootTest` | 集成测试 | 是 |
| `@WebMvcTest` | Web 切片 | 否 |
| `@DataJpaTest` | JPA 切片 | 否 |

---

# 2. WebEnvironment 选项

## 2.1 `MOCK`

默认值，启动 Servlet 环境但不真正打开端口，配合 MockMvc。

## 2.2 `RANDOM_PORT`

启动嵌入式容器并随机端口，适合端到端测试。

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class HttpE2eTest {}
```

## 2.3 `DEFINED_PORT`

使用固定端口，适合对外部系统有强依赖的测试，但容易端口冲突。

## 2.4 `NONE`

不启动 Web 环境，仅加载上下文。

---

# 3. 常用属性

## 3.1 classes

```java
@SpringBootTest(classes = MinimalTestConfig.class)
```

限制启动的配置类，减少上下文规模。

## 3.2 properties

```java
@SpringBootTest(properties = {"app.featureX=true"})
```

适合为测试覆盖关键配置。

## 3.3 args

```java
@SpringBootTest(args = "--app.mode=test")
```

模拟命令行参数注入。

---

# 4. 启动流程（源码视角）

## 4.1 启动链路

```text
@SpringBootTest
  └── SpringBootTestContextBootstrapper
        └── SpringBootContextLoader
              └── SpringApplication.run
                    └── ApplicationContext
```

## 4.2 设计意图

- 复用生产启动路径，减少环境差异
- 将测试配置融入 SpringApplication 生命周期

---

# 5. 常见组合

## 5.1 与 `@AutoConfigureMockMvc`

```java
@SpringBootTest
@AutoConfigureMockMvc
class MockMvcIntegrationTest {}
```

## 5.2 与 `@ActiveProfiles`

```java
@SpringBootTest
@ActiveProfiles("test")
class ProfileTest {}
```

## 5.3 与 `@MockBean`

```java
@SpringBootTest
class PaymentTest {
    @MockBean
    private PaymentGateway gateway;
}
```

---

# 6. Boot 2.x 与 3.x 差异点

- Boot 3.x 基线为 JDK 17，测试代码需升级
- Jakarta 包名迁移影响 `javax.servlet.*` 相关测试
- Spring Test 6.x 对部分监听器顺序与行为做了调整

---

# 7. 常见问题

## 7.1 启动过慢

- 优先改用测试切片
- 精简 `classes` 与自动配置

## 7.2 Bean 注入失败

- 检查测试类是否位于启动类同包或子包
- 使用 `@Import` 或 `@ComponentScan` 明确引入
