# 1. 测试配置覆盖体系

测试环境几乎从不使用生产配置：数据库要换成内嵌或容器实例、邮件要关掉、超时要缩短、特性开关要能按测试类拨动。Spring Boot 的解法不是"另一套配置系统"，而是**复用生产的 Environment 抽象，再叠加测试专属的覆盖入口**。搞清楚这些入口的优先级顺序，就能回答两个高频问题："我在 yml 里改了为什么没生效？"和"这个属性到底是谁给的？"

## 1.1 配置优先级要点

Spring Boot 的 PropertySource 是有严格顺序的栈，测试中常见覆盖方式全部位于栈顶附近：

1. `@SpringBootTest(properties = ...)` / `@TestPropertySource(properties = ...)`——测试内联属性，**在测试上下文里优先级最高**
2. `@DynamicPropertySource`——运行期动态注册，紧随内联属性
3. `@TestPropertySource(locations = ...)` 指定的测试属性文件
4. `application-test.yml`（配合 `@ActiveProfiles("test")`）
5. 常规 `application.yml` 与系统属性、环境变量

优先级靠前者会覆盖靠后的配置。完整的生产级顺序（Spring Boot 3.x，从高到低）值得背一遍骨架：

| 优先级 | PropertySource | 说明 |
|--------|----------------|------|
| 1（最高） | `@TestPropertySource` 内联属性 | 仅测试上下文存在 |
| 2 | `@DynamicPropertySource` 动态属性 | 容器启动前注册 |
| 3 | 命令行参数 `--key=value` | `@SpringBootTest(args=...)` 传入 |
| 4 | `SPRING_APPLICATION_JSON` | 环境变量里的 JSON 配置 |
| 5 | Servlet 参数、JNDI | Web 场景 |
| 6 | `System.getProperties()` | `-Dkey=value` |
| 7 | OS 环境变量 | `KEY_VALUE` 宽松绑定 |
| 8 | jar 外的 `application-{profile}.yml` | 部署目录配置 |
| 9 | jar 内的 `application-{profile}.yml` | **测试 profile 文件在这一档** |
| 10 | jar 内 `application.yml` | 基线配置 |
| 11 | `@PropertySource` 注解引入 | 通常最低的可控入口 |
| 12 | `SpringApplication.setDefaultProperties` | 兜底默认值 |

```text
 测试上下文 Environment（简化视图）
┌────────────────────────────────────────────┐
│ Inlined Test Properties  ← @TestPropertySource(properties)
│ Dynamic Properties       ← @DynamicPropertySource
│ Command Line Args        ← @SpringBootTest(args)
│ System Properties / Env
│ application-test.yml     ← @ActiveProfiles("test")
│ application.yml          ← 基线
└────────────────────────────────────────────┘
        查找方向：自上而下，先命中先赢
```

> **注意**：`src/test/resources/application.yml` 会**整个遮蔽** `src/main/resources/application.yml`——同名文件在测试 classpath 上排在前面，main 里的那份根本不会被加载（不是合并！）。想让测试配置叠加而非替换，用不同文件名（`application-test.yml`）或 `@TestPropertySource(locations = "classpath:custom.yml")`。

## 1.2 用 Environment 反查属性来源

调试"这个值哪来的"不用猜，测试里直接问 Environment：

```java
@SpringBootTest
class PropertyOriginTest {

    @Autowired
    private Environment env;

    @Test
    void should_resolve_property_from_expected_source() {
        // 打印所有 PropertySource 的名字与顺序
        ConfigurableEnvironment ce = (ConfigurableEnvironment) env;
        ce.getPropertySources().forEach(ps -> System.out.println(ps.getName()));

        // 断言最终生效值
        assertThat(env.getProperty("app.retry.max")).isEqualTo("3");
    }
}
```

Actuator 的 `/actuator/env` 端点同样能在 `RANDOM_PORT` 集成测试里通过 HTTP 查看，排配置问题时比翻代码快。

---

# 2. `@ActiveProfiles`

## 2.1 作用

激活指定 Profile，从而加载 `application-{profile}.yml`，并影响所有 `@Profile` 条件 Bean 的装配：

```java
@ActiveProfiles("test")
@SpringBootTest
class ProfileTest {
    // 容器加载 application.yml + application-test.yml（后者覆盖前者）
    // 且只装配 @Profile("test") 或无 @Profile 限制的 Bean
}
```

`@ActiveProfiles` 还支持继承与合并策略：

```java
@ActiveProfiles(profiles = "test", inheritProfiles = false) // 不继承父类 profile
class IsolatedProfileTest extends BaseIntegrationTest { }

@ActiveProfiles({"test", "no-mail"})                        // 多 profile，后者文件覆盖前者
class MultiProfileTest { }
```

## 2.2 常见约定

- `application-test.yml`：测试默认配置——内嵌库连接、关闭邮件/短信真实发送、缩短超时、关掉调度
- `application-local.yml`：开发本地配置——连本地或开发环境数据库
- Profile 之间**只做差异覆盖**，公共项留在 `application.yml`，否则三个文件各改一处、谁也说不清最终值

一份典型的测试 profile 文件：

```yaml
# src/test/resources/application-test.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1   # 内嵌库，进程内共享
    driver-class-name: org.h2.Driver
  mail:
    host: localhost                              # 指向不存在的服务也没关系，
    port: 3025                                   # 测试里 MailSender 已被替换为 Fake
  jpa:
    hibernate:
      ddl-auto: create-drop                      # 测试库不保留 Schema
app:
  billing:
    timeout: 500ms                               # 超时缩短，失败测试快死快超生
  scheduler:
    enabled: false                               # 关掉定时任务，避免后台干扰
```

> **行业实践**：测试 profile 应该让**所有外部副作用默认关闭**（邮件、短信、Webhook、调度），需要验证副作用的测试再单独打开并用 Fake/Testcontainers 承接。"跑一次单测给真客户发了封邮件"是真实发生过的事故。

---

# 3. `@TestPropertySource`

## 3.1 内联属性覆盖

用于在测试类级别覆盖配置，优先级压过一切配置文件：

```java
@TestPropertySource(properties = {
        "app.featureX=true",              // 只为这个测试类开启特性 X
        "app.retry.max=1"                 // 覆盖 yml 里的重试次数
})
@SpringBootTest
class PropertyOverrideTest { }
```

`@SpringBootTest(properties = "...")` 是它的简写等价物，两者可叠加（合并进同一份内联属性）。需要**删除**某个已定义的属性，用空值语法：`properties = "app.featureX="`。

## 3.2 外部属性文件

也可以指定测试专属的属性文件，适合多个测试类共享一大组配置：

```java
@TestPropertySource(locations = "classpath:custom-test.properties")
class SharedConfigTest { }
```

`locations` 支持 `classpath:`、`file:` 前缀；文件格式除 `.properties` 外，Boot 环境也接受 `.yml`（由 `YamlPropertySourceLoader` 处理）。

> **注意**：内联 `properties` 与 `locations` 同时存在时，**内联优先**。另外 `@TestPropertySource` 不继承 `@SpringBootTest` 的组件扫描语义——它只影响 Environment，不会让配置类多加载或少加载。

## 3.3 与上下文缓存的相互作用

`@TestPropertySource` 的每个不同组合都会产生一个新的 `MergedContextConfiguration`，即新的上下文缓存 Key：

```text
TestA: @SpringBootTest                          → ctx-1（裸配置）
TestB: @SpringBootTest(properties="x=1")        → ctx-2（多一条属性 → 新容器）
TestC: @SpringBootTest(properties="x=1")        → ctx-2（与 B 完全一致 → 命中缓存）
TestD: @SpringBootTest(properties="x=2")        → ctx-3（值不同也是新 Key）
```

这就是配置收敛纪律的来源：**类级属性覆盖是"每类一次容器启动"的价格标签**。少量关键测试值得付，几十上百个测试类各写各的 `properties`，CI 就会为每种组合各起一个容器。公共项下沉到 `application-test.yml`，个性项才留在注解上。

---

# 4. `@DynamicPropertySource`

## 4.1 适用场景

有些属性在**编写测试时不可能知道**：Testcontainers 起的 MySQL 随机映射了宿主机端口、WireMock 随机选端口、临时目录路径……这些值要等测试 JVM 运行起来才产生。`@DynamicPropertySource`（Spring Framework 5.2.5+）允许在容器刷新前把"取值函数"注册进 Environment。完整可跑的 Testcontainers 组合：

完整可跑的 Testcontainers 组合：

```java
@SpringBootTest
@Testcontainers                                        // JUnit 扩展接管容器生命周期
class OrderRepositoryIT {

    @Container                                         // 静态字段：容器先于 Spring 启动
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.4");

    @DynamicPropertySource
    static void registerMysqlProperties(DynamicPropertyRegistry registry) {
        // 注册的是 Supplier，取值延迟到属性真正被读取时——此时容器已 start
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
    }

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void should_persist_order() {
        orderRepository.save(new Order(null, "A-1"));
        assertThat(orderRepository.count()).isEqualTo(1);
    }
}
```

动态注册随机 token、固定测试数据同样可行：

```java
@DynamicPropertySource
static void register(DynamicPropertyRegistry registry) {
    registry.add("app.token", () -> UUID.randomUUID().toString());  // 每次容器构建生成
}
```

**规则与限制**：方法必须是 `static`（要在测试实例化之前执行）；每个 `@DynamicPropertySource` 方法的值参与上下文缓存 Key 的方式特殊——Spring 用"动态属性源的存在 + 名称"而非最终值参与指纹，因此**值不同但注册方法相同的测试类仍可能共享上下文，共享的是第一次注册时算出的值**。多个测试类各自起容器又想共享上下文时，应把容器和注册方法提取到公共父类。

## 4.2 Boot 3.1+ 的 `@ServiceConnection`

Boot 3.1 起，对 Testcontainers 的主流支持已经不需要手写 `@DynamicPropertySource`：

```java
@SpringBootTest
@Testcontainers
class OrderRepositoryIT {

    @Container
    @ServiceConnection                    // 自动把容器的 url/用户名/密码注册进 Environment
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.4");
}
```

`@ServiceConnection` 按容器类型（MySQL、Postgres、Redis、Kafka、MongoDB……）内置了属性映射，还支持 `connectionName` 区分同类型多容器。手写 `@DynamicPropertySource` 退居两种场景：不被内置支持的容器类型，以及需要注册**业务自定义属性**（如 token、路径）时。

> **设计争议**：`@ServiceConnection` 便利但"魔法"——属性映射规则藏在框架里，排错时看不到注册了什么。团队里至少应有一个人清楚它等价于哪些 `registry.add` 调用；对需要审计配置来源的场景，显式 `@DynamicPropertySource` 反而更直白。

---

# 5. `@TestConfiguration`

## 5.1 提供测试专用 Bean

`@TestConfiguration` 是 `@Configuration` 的特化，语义是"这份配置只属于测试"：

```java
@SpringBootTest
class OrderServiceTest {

    @TestConfiguration                        // 静态内部类：自动被当前测试类拾取
    static class TestConfig {
        @Bean
        Clock fixedClock() {
            return Clock.fixed(Instant.parse("2026-01-01T00:00:00Z"), ZoneId.of("UTC"));
        }

        @Bean
        MailSender fakeMailSender() {         // 顶掉真实邮件 Bean
            return new RecordingMailSender(); // 只记录不发送的 Fake
        }
    }

    @Autowired
    private OrderService orderService;        // 注入的依赖里 Clock/MailSender 已是测试版
}
```

## 5.2 与主配置的合并规则

`@TestConfiguration` 的行为差异全在"怎么被合并进容器"：

| 声明位置 | 是否自动注册 | 说明 |
|----------|--------------|------|
| 测试类的静态内部类 | **自动** | Spring Test 隐式把内部配置类追加到上下文配置 |
| 独立的顶层类 | 不自动 | 需 `@Import(TestConfig.class)` 或组件扫描覆盖 |
| 被主应用 `@ComponentScan` 扫到 | 不会 | `@TestConfiguration` 自带排除逻辑，主扫描会跳过它 |

合并的先后与覆盖语义：

- 测试配置在主配置**之后**处理；
- 同名 Bean 默认**不允许覆盖**（Boot 2.1+ `spring.main.allow-bean-definition-overriding=false`），测试配置里定义与主配置**同名**的 Bean 会直接启动失败——解法要么开覆盖开关，要么让测试 Bean 用不同名字 + `@Primary`：

```java
@TestConfiguration
static class MailConfig {
    @Bean
    @Primary                              // 类型相同、名字不同，@Primary 决定注入谁
    MailSender recordingMailSender() {
        return new RecordingMailSender();
    }
}
```

- 顶层 `@TestConfiguration` 类如果想被多个测试共享，放在 `src/test/java` 下用 `@Import` 显式引入，比依赖扫描更可控。

> **行业实践**：`@TestConfiguration` + `@Primary` 与 `@MockitoBean` 是两条替换路线：前者提供**有行为的替身**（Fake、固定 Clock、录制器），适合多个测试类复用同一套测试基础设施；后者提供**空壳 Mock**，适合单类内的快速打桩。项目里最好统一约定——比如 `src/test/java/.../config/` 下集中放共享 `@TestConfiguration`，避免散落在各测试类里重复定义。

---

# 6. `@Import` 与组合

`@Import` 把任意配置类追加进当前测试的上下文：

```java
@SpringBootTest
@Import(TestConfig.class)                 // 引入共享测试配置
class ImportTest { }
```

三类高频用法：

1. **测试类不在启动类同包或子包**时，`@SpringBootTest` 找不到 `@SpringBootConfiguration`，显式导入：`@SpringBootTest(classes = Application.class)` 或 `@ContextConfiguration(classes = ...)`；
2. **切片测试补充缺失 Bean**：`@WebMvcTest` 只扫 Web 组件，测试需要的 Jackson 定制模块、安全配置片段用 `@Import` 带进来；
3. **组合测试基础设施**：把 Testcontainers 基类、共享 Fake 配置做成注解元数据：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@SpringBootTest
@ActiveProfiles("test")
@Import(IntegrationTestConfig.class)
@TestPropertySource(properties = "app.env=ci")
public @interface StandardIntegrationTest { }   // 项目统一的集成测试"套餐"注解

@StandardIntegrationTest                         // 使用处一行搞定，且上下文 Key 一致
class OrderFlowIT { }
```

自定义组合注解有个隐藏红利：所有用它的测试类**上下文配置指纹完全一致**，共享同一个缓存容器。

---

# 7. 常见问题

## 7.1 配置未生效

按命中概率排序的排查清单：

| 现象 | 原因 | 排查/解决 |
|------|------|-----------|
| yml 改了没用 | Profile 未激活 | 确认 `@ActiveProfiles("test")`，或文件确实叫 `application-test.yml` |
| yml 改了没用 | 被更高优先级源覆盖 | 用 1.2 的 Environment 反查，看 PropertySource 顺序 |
| 值来自"上一个测试" | 上下文缓存复用了旧容器 | 属性组合不同本应新起容器；检查是否被 `@DynamicPropertySource` 指纹规则合并 |
| `@Value` 是 null | 测试类没进容器 | 纯 Jupiter 测试里 `@Value` 静默失效（不报错！） |
| test 的 application.yml 全丢了 | 同名遮蔽 | `src/test/resources/application.yml` 会整个替换 main 的同名文件 |
| Bean 覆盖失败启动报错 | 默认禁止 BeanDefinition 覆盖 | 测试 Bean 改名 + `@Primary`，或显式开 `allow-bean-definition-overriding` |

## 7.2 配置泄漏

多个测试类共享同一 Context 时，属性覆盖需一致，否则会触发新容器——但反过来也有"泄漏"陷阱：**共享容器意味着共享容器里的可变状态**。一个测试通过 `Environment` 改不了已绑定的 `@ConfigurationProperties` 对象（绑定发生在启动期），但如果某测试直接修改了单例 Bean 的内部字段，后续复用该容器的测试都会看到脏数据。防线：测试里不写单例 Bean 状态；确需修改用 `@DirtiesContext` 声明（代价是重建容器）；更好的设计是把可变状态放进每方法重建的组件。

# 8. 小结

| 覆盖手段 | 粒度 | 时机 | 上下文缓存影响 |
|----------|------|------|----------------|
| `application-test.yml` + `@ActiveProfiles` | 全局测试基线 | 容器启动加载 | Profile 参与指纹 |
| `@TestPropertySource` / `properties` | 单个测试类 | 容器启动前注入 Environment | 每种组合一个新容器 |
| `@DynamicPropertySource` | 单个测试类（运行期值） | 刷新前注册 Supplier | 注册方法参与指纹，值不参与 |
| `@ServiceConnection`（Boot 3.1+） | 容器连接属性 | 同上，自动映射 | 同上 |
| `@TestConfiguration` + `@Primary` | Bean 级替换 | 配置合并阶段 | 配置类参与指纹 |

一句话原则：**基线进 yml，个性进注解，动态进 `@DynamicPropertySource`，替身进 `@TestConfiguration`**——并且时刻记得每种组合都在为上下文缓存"付费"。
