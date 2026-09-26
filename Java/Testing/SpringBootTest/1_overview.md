# 1. 定义与目标

## 1.1 Spring Boot Test 是什么

Spring Boot Test 是 Spring Boot 为测试提供的一整套能力集合，核心目标是**让测试在“接近真实运行环境”的前提下依然保持可控、可读、可维护**。它不是单一框架，而是对以下组件的整合与增强：

- Spring Test（TestContext Framework、MockMvc 等）
- JUnit 5（JUnit Platform + Jupiter）
- Mockito、AssertJ、JSONassert、Hamcrest 等
- Spring Boot 自动配置与测试切片

## 1.2 设计目标与价值

### 1.2.1 快速可靠

- 通过 TestContext 缓存 ApplicationContext，减少重复启动成本
- 提供测试切片（slice）只加载必要 Bean，缩短启动时间
- 提供注解式配置，降低测试初始化复杂度

### 1.2.2 接近真实

- `@SpringBootTest` 可以启动完整应用上下文
- `WebEnvironment.RANDOM_PORT` 启动嵌入式容器，模拟真实 HTTP 请求
- 与 Spring Boot 自动配置一致，避免“测试能过、生产挂”

### 1.2.3 可控可替换

- `@MockBean`、`@SpyBean` 可替换真实 Bean
- `@TestConfiguration` 可为测试提供专用配置
- `@DynamicPropertySource` 动态替换运行时配置

## 1.3 与 Spring Test / JUnit 的关系

- Spring Test 提供底层机制：TestContext、测试生命周期、依赖注入
- JUnit 5 提供测试引擎与扩展点：`@Test`、`@ExtendWith`
- Spring Boot Test 在两者之上提供自动配置、切片测试与便捷注解

---

# 2. 测试类型与覆盖面

## 2.1 测试金字塔在 Boot 场景的映射

| 层级 | 目标 | Spring Boot 典型手段 |
|------|------|---------------------|
| 单元测试 | 逻辑验证、快速反馈 | JUnit + Mockito（无 Spring） |
| 切片测试 | 局部组件验证 | `@WebMvcTest` / `@DataJpaTest` / `@JsonTest` |
| 集成测试 | 多组件协作 | `@SpringBootTest` + `@MockBean` |
| 端到端测试 | 真实链路验证 | `@SpringBootTest(web=RANDOM_PORT)` + `TestRestTemplate` |

## 2.2 选择测试类型的核心准则

- **速度优先**：能不启 Spring 就不启
- **边界清晰**：只验证“我关心的那一层”
- **失败可定位**：失败时尽量指向单一问题
- **复用真实配置**：避免测试和生产配置分叉过大

## 2.3 典型组合

- 控制器层：`@WebMvcTest` + MockMvc + `@MockBean` service
- 仓储层：`@DataJpaTest` + 内嵌数据库
- 系统回归：`@SpringBootTest` + 随机端口 + `TestRestTemplate`

---

# 3. 版本与生态关系

## 3.1 Spring Boot 3.x 与 2.x 差异

| 维度 | Boot 2.x | Boot 3.x |
|------|----------|----------|
| 基线 JDK | 8/11 | 17+ |
| Java EE | javax.* | jakarta.* |
| Spring Framework | 5.x | 6.x |
| JUnit | 5 已默认 | 5 深度集成 |

**测试影响点**：

- 引入 Jakarta 包名迁移，测试代码需更新 import
- Spring Test 内部 API 与监听器链有升级

## 3.2 核心依赖

### 3.2.1 Maven

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

### 3.2.2 Gradle

```gradle
dependencies {
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

---

# 4. 核心术语

## 4.1 ApplicationContext 与 TestContext

- **ApplicationContext**：Spring 容器的运行时模型
- **TestContext**：测试运行时上下文，负责加载与缓存 ApplicationContext

## 4.2 测试切片（Slice）

只加载某一层所需的 Bean，减少启动时间并提升测试隔离性。

## 4.3 Test Double（替身）

| 类型 | 说明 | Boot 典型用法 |
|------|------|--------------|
| Mock | 模拟行为与交互 | `@MockBean` |
| Spy | 代理真实对象 | `@SpyBean` |
| Stub | 固定返回值 | Mockito `when` |
| Fake | 简化实现 | `@TestConfiguration` 注入 |

---

# 5. 最小示例

## 5.1 业务代码

```java
@Service
public class GreetingService {
    public String greet(String name) {
        return "hello, " + name;
    }
}
```

## 5.2 测试代码

```java
@SpringBootTest
class GreetingServiceTest {

    @Autowired
    private GreetingService greetingService;

    @Test
    void greet_shouldReturnHello() {
        assertThat(greetingService.greet("spring")).isEqualTo("hello, spring");
    }
}
```

## 5.3 执行方式

```text
mvn -q -DskipTests=false test
```

```text
gradle test
```
