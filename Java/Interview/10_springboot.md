# 1. 知识点
- **`@SpringBootApplication`**：核心组合注解，包含 `@SpringBootConfiguration`（配置类标识）、`@EnableAutoConfiguration`（自动配置）、`@ComponentScan`（组件扫描）三个注解。
- **`@EnableAutoConfiguration`**：自动配置核心，通过 `@Import(AutoConfigurationImportSelector.class)` 读取 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件（Spring Boot 3.x），旧版读取 `spring.factories` 中的 `EnableAutoConfiguration` 键，加载所有自动配置类。
- **自动配置原理**：Spring Boot 根据 classpath 中的 jar 包和配置，通过条件注解（`@Conditional` 系列）自动配置 Bean。如 classpath 存在 `DataSource` 类且没有自定义 `DataSource` Bean 时，自动配置 `HikariCP` 数据源。
- **`@ConditionalOnClass`**：当 classpath 中存在指定类时，配置类/Bean 才生效。对应 `@ConditionalOnMissingClass`。
- **`@ConditionalOnBean`**：当容器中存在指定 Bean 时，配置才生效。对应 `@ConditionalOnMissingBean`。
- **`@ConditionalOnProperty`**：根据配置文件中指定属性的值决定是否生效，支持 `havingValue` 匹配值和 `matchIfMissing` 默认值。
- **`@ConditionalOnResource`**：当 classpath 中存在指定资源文件时生效。
- **`@ConditionalOnWebApplication`**：判断当前是否为 Web 应用，可选值 SERVLET、REACTIVE。
- **`@ConditionalOnExpression`**：根据 SpEL 表达式计算结果决定是否生效，更灵活的条件判断。
- **启动流程**：`SpringApplication.run()` → 创建 `SpringApplication` 实例（推断 Web 环境类型、加载 ApplicationContextInitializer 和 ApplicationListener）→ 发布 `ApplicationStartingEvent` → 准备 `Environment` → 发布 `ApplicationEnvironmentPreparedEvent` → 创建 `ApplicationContext` → 准备上下文（执行 Initializer、加载 Bean、发布 `ApplicationPreparedEvent`）→ `refresh()` 上下文 → 启动内嵌 WebServer → 发布 `ApplicationStartedEvent` → 执行 `CommandLineRunner`/`ApplicationRunner` → 发布 `ApplicationReadyEvent`。
- **配置文件优先级（从高到低）**：命令行参数 > 环境变量 > profile 特定配置文件 `application-{profile}.yml` > `application.yml` > jar 包外部 `application.yml` > jar 包内部 `application.yml` > `@PropertySource` > 默认值。
- **`@Value` vs `@ConfigurationProperties`**：`@Value` 用于注入单个配置值（需 SpEL）；`@ConfigurationProperties` 用于批量注入一组相关配置到 POJO 类，支持宽松绑定（relaxed binding：驼峰、短横线、下划线互相转换）、JSR-303 校验、类型安全、`@NestedConfigurationProperty` 嵌套对象。
- **`@EnableConfigurationProperties`**：启用 `@ConfigurationProperties` 标注的类并将其注册为 Bean，无需在每个类上加 `@Component`。
- **`@ConfigurationPropertiesScan`**：扫描指定包路径下的 `@ConfigurationProperties` 类并自动注册，替代逐个 `@EnableConfigurationProperties`。
- **Profile 环境隔离**：通过 `spring.profiles.active=dev` 激活不同环境配置，文件命名 `application-dev.yml`、`application-prod.yml`。支持 `spring.profiles.include` 包含多个 profile。
- **日志**：Spring Boot 默认使用 Logback（SLF4J 门面），可通过 `logback-spring.xml` 配置文件，支持 `logging.level.root`、`logging.file.path`、`logging.pattern.console` 等属性。也可切换到 Log4j2。
- **嵌入式容器**：Spring Boot 内嵌 Tomcat（默认）、Undertow、Jetty 三种 Servlet 容器，无需部署到外部容器。可通过 `server.port`、`server.ssl.*` 配置端口和 SSL。排除默认 Tomcat 并添加其他容器依赖即可切换。
- **Starter 机制**：Starter 是一组依赖描述符，将特定功能所需的依赖打包在一起。命名规范：官方 `spring-boot-starter-*`，第三方 `*-spring-boot-starter`。开发者只需引入一个 Starter 即可获得完整功能栈。
- **自定义 Starter**：创建 autoconfigure 模块（含自动配置类、`AutoConfiguration.imports`、条件注解）和 starter 模块（pom 依赖引入 autoconfigure）。核心是定义 `@ConfigurationProperties` 类和利用条件注解控制配置生效。
- **Actuator 监控端点**：`/actuator/health`（健康检查）、`/actuator/beans`（所有 Bean 列表）、`/actuator/metrics`（指标数据）、`/actuator/env`（环境变量）、`/actuator/mappings`（URL 映射）、`/actuator/configprops`（@ConfigurationProperties 值）、`/actuator/loggers`（运行日志级别动态调整）。
- **自定义 Actuator 端点**：通过 `@Endpoint` 注解定义自定义端点，配合 `@ReadOperation`、`@WriteOperation` 提供 HTTP/JMX 访问。
- **自定义 HealthIndicator**：实现 `HealthIndicator` 接口，返回 `Health.up().withDetail()` 或 `Health.down()`，应用的健康状态由所有 HealthIndicator 的聚合结果决定。
- **Actuator 安全**：通过 `management.endpoints.web.exposure.include=health,info` 控制暴露的端点。`management.endpoint.*.enabled=false` 单独禁用。结合 Spring Security 可限制端点访问权限。
- **Spring Boot Admin**：独立的监控仪表盘（Server 端），Client 端引入 `spring-boot-admin-starter-client` 通过 HTTP 向 Server 注册。支持查看线程堆栈、动态修改日志级别、JVM 指标、环境变量，结合 Spring Cloud 实现服务发现。
- **外部化配置**：Spring Boot 支持从多种来源读取配置。除了 application.yml，还支持命令行参数、环境变量、Java System 属性、Servlet 配置参数等，按优先级顺序覆盖。
- **多数据源配置**：通过 `@Primary` 指定主数据源，自定义 `DataSourceProperties`、`DataSource`、`PlatformTransactionManager` 等 Bean，通过 `@Qualifier` 区分不同数据源。使用 `@ConfigurationProperties` 分别绑定每个数据源的连接参数。
- **`ApplicationRunner` vs `CommandLineRunner`**：都可在应用启动后执行一次性逻辑。`CommandLineRunner` 接收原始字符串数组参数，`ApplicationRunner` 接收 `ApplicationArguments` 对象（封装了选项和非选项参数），提供了更方便的参数解析方法。
- **懒加载**：设置 `spring.main.lazy-initialization=true` 可全局开启懒加载（替代默认的饿汉式初始化），所有 Bean 在首次使用时才初始化，减少启动时间，但可能掩盖运行时刻的配置错误。
- **配置元数据**：`spring-boot-configuration-processor` 生成 `META-INF/spring-configuration-metadata.json`，为 IDE 提供 `application.yml` 中属性的自动补全、类型校验和文档提示。
- **优雅停机**：`server.shutdown=graceful` + `spring.lifecycle.timeout-per-shutdown-phase=30s`，在关闭时等待当前请求处理完成。
- **Spring Boot DevTools**：开发工具包，提供自动重启（比冷启动快）、LiveReload（浏览器自动刷新）、禁用模板缓存、全局配置等功能。
- **Fat Jar**：Spring Boot Maven/Gradle 插件将应用及其所有依赖打包成单个独立可执行的 Jar 文件，内嵌启动类和 ClassLoader，可直接 `java -jar` 运行。
- **`@SpringBootTest`**：集成测试注解，加载完整的 Spring 应用上下文，支持设置 `webEnvironment` 模拟 Web 环境，可指定 `classes` 值加载特定配置类。

# 2. 面试题

## 2.1 Spring Boot 自动配置原理是什么？
Spring Boot 自动配置的核心是 `@EnableAutoConfiguration` 注解：

1. `@EnableAutoConfiguration` 通过 `@Import(AutoConfigurationImportSelector.class)` 导入自动配置选择器。
2. `AutoConfigurationImportSelector` 在运行时调用 `SpringFactoriesLoader`，读取 classpath 下 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件（Spring Boot 3.x），获取所有候选自动配置类的全限定名。
3. 加载这些自动配置类，但这些类并不会无条件全部生效，每个类上都有**条件注解**（`@ConditionalOnClass`、`@ConditionalOnBean`、`@ConditionalOnProperty` 等）控制的生效条件。
4. 只有满足条件的配置类才会创建对应的 Bean。例如 `DataSourceAutoConfiguration` 在 classpath 有 `DataSource` 类且容器中没有自定义 `DataSource` Bean 时，才自动配置 HikariCP 数据源。
5. 用户可以在 `@SpringBootApplication` 中通过 `exclude`/`excludeName` 属性排除不需生效的自动配置类。

## 2.2 @SpringBootApplication 由哪三个注解组成？各自作用？
```java
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan
public @interface SpringBootApplication { ... }
```

- **`@SpringBootConfiguration`**：本质就是 `@Configuration`，表示当前类是 Spring Boot 的配置类，允许在其中定义 `@Bean`。
- **`@EnableAutoConfiguration`**：核心注解，开启自动配置机制，根据 classpath 和已有 Bean 自动配置缺失的组件。
- **`@ComponentScan`**：组件扫描，默认扫描当前配置类所在包及其子包中带 `@Component`、`@Service`、`@Repository`、`@Controller` 等注解的类，注册为 Bean。

## 2.3 Spring Boot 启动的完整流程是怎样的？
1. **构造 `SpringApplication` 实例**：
   - 推断 Web 应用类型（SERVLET、REACTIVE、NONE）—— 通过 classpath 中是否存在特定类判断。
   - 从 `spring.factories` 加载并初始化 `ApplicationContextInitializer`。
   - 从 `spring.factories` 加载并初始化 `ApplicationListener`。
   - 推断主类（main 方法所在类）。
2. **执行 `run()` 方法**：
   - 创建 `StopWatch` 计时器，开始计时。
   - 创建 `BootstrapContext`。
   - 通过 `SpringApplicationRunListeners` 发布 `ApplicationStartingEvent`。
   - 准备 `Environment`（根据当前激活的 profile 加载配置）。
   - 发布 `ApplicationEnvironmentPreparedEvent`。
   - 打印 Banner。
   - 根据 Web 类型创建对应的 `ApplicationContext`（Servlet → `AnnotationConfigServletWebServerApplicationContext`）。
   - 准备 `ApplicationContext`：设置 Environment、执行 Initializer、加载 Bean 定义、发布 `ApplicationContextInitializedEvent`。
   - 刷新 `ApplicationContext`（`refresh()` 方法，完成 Bean 的创建和依赖注入）。
   - 发布 `ApplicationStartedEvent`。
   - 调用 `ApplicationRunner` 和 `CommandLineRunner`。
   - 发布 `ApplicationReadyEvent`，应用启动完成。

## 2.4 Spring Boot Starter 是什么？如何自定义一个 Starter？
**Starter** 是一站式依赖描述符，将某个功能所需的依赖（jar 包、自动配置）打包在一起，开发者只需引入一个 Starter 即可获得完整功能。

**自定义 Starter 步骤**：
1. **创建 autoconfigure 模块**：包含自动配置类、`@ConfigurationProperties` 属性绑定类、业务逻辑类、元数据文件 `AutoConfiguration.imports`。
2. **创建 starter 模块**：pom 中引入 autoconfigure 模块，作为聚合依赖的入口（通常只有 pom 文件）。
3. **配置类示例**：
```java
@Configuration
@EnableConfigurationProperties(MyProperties.class)
@ConditionalOnClass(MyService.class)
public class MyAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public MyService myService(MyProperties properties) {
        return new MyService(properties);
    }
}
```
4. 在 `META-INF/spring/AutoConfiguration.imports` 中添加自动配置类的全限定名。

**命名规范**：官方 starter 为 `spring-boot-starter-*`，第三方为 `*-spring-boot-starter`。

## 2.5 @ConfigurationProperties 和 @Value 的区别？
| 维度 | @ConfigurationProperties | @Value |
|------|-------------------------|--------|
| 绑定方式 | 批量绑定，按 prefix 反射注入同名属性 | 逐个注入，使用 SpEL 表达式 `${}` |
| 宽松绑定 | 支持（驼峰/下划线/短横线自动转换） | 不支持 |
| JSR-303 校验 | 支持 `@Validated` 校验 | 不直接支持 |
| 嵌套对象 | 支持 `@NestedConfigurationProperty` | 不支持（需手动逐字段注入） |
| 默认值 | 通过字段初始值或配置文件 | 通过 `:` 分隔符：`${key:default}` |
| 适用场景 | 一组相关配置，类型安全绑定到 POJO | 单个简单值，少量配置 |

## 2.6 Spring Boot 配置文件加载优先级是怎样的？
从高到低：
1. **命令行参数**（`--server.port=8081`）
2. **操作系统环境变量**
3. **JNDI 属性**（`java:comp/env`）
4. **`application-{profile}.yml`**（jar 外部）
5. **`application-{profile}.yml`**（jar 内部）
6. **`application.yml`**（jar 外部）
7. **`application.yml`**（jar 内部）
8. **`@PropertySource`** 标注的属性源
9. **SpringApplication.setDefaultProperties()** 设置的默认值

**profile 特定配置优先级高于无 profile 配置**。同一位置的 `application.yml` 和 `application.properties` 同时存在时，`.properties` 优先级更高。

## 2.7 Spring Boot 如何处理多环境配置？
通过 **Profile** 机制实现：
1. 创建环境特定的配置文件：`application-dev.yml`、`application-test.yml`、`application-prod.yml`。
2. 在 `application.yml` 中激活：
```yaml
spring:
  profiles:
    active: dev
```
3. 也可通过命令行 `--spring.profiles.active=prod` 或环境变量 `SPRING_PROFILES_ACTIVE=prod` 指定。
4. **Profile 包含**：使用 `spring.profiles.include` 可组合多个 profile。
5. **代码中判断**：通过 `Environment` 的 `acceptsProfiles()` 方法判断当前激活的 profile。

## 2.8 Spring Boot Actuator 提供了哪些端点？如何自定义健康检查？
常用端点：
| 端点 | 用途 |
|------|------|
| `/health` | 应用健康状态 |
| `/info` | 应用基本信息 |
| `/beans` | 容器中所有 Bean |
| `/metrics` | 内存、CPU、GC、HTTP 请求等指标 |
| `/env` | 环境变量和配置属性 |
| `/mappings` | URL 映射关系 |
| `/configprops` | @ConfigurationProperties Bean |
| `/loggers` | 日志级别查看和动态修改 |
| `/threaddump` | 线程堆栈快照 |
| `/heapdump` | JVM 堆转储文件 |

**自定义健康检查**：
```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        // 模拟检查数据库连通性
        try {
            boolean isConnected = checkDatabase();
            if (isConnected) {
                return Health.up().withDetail("database", "MySQL is up").build();
            }
            return Health.down().withDetail("database", "Connection failed").build();
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}
```

## 2.9 Spring Boot 如何切换嵌入式容器？
默认使用 Tomcat，切换方式：

**切换到 Undertow**：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

容器筛选优先级：Undertow > Jetty > Tomcat。通过 `WebServerApplicationContext` 检测 classpath 中存在哪个容器实现类来决定。

## 2.10 如何配置多个数据源？
```java
@Configuration
public class DataSourceConfig {
    
    @Primary
    @Bean("primaryDS")
    @ConfigurationProperties("spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean("secondaryDS")
    @ConfigurationProperties("spring.datasource.secondary")
    public DataSource secondaryDataSource() {
        return DataSourceBuilder.create().build();
    }
}
```

在 yaml 中分别配置：
```yaml
spring:
  datasource:
    primary:
      url: jdbc:mysql://localhost:3306/db1
      username: root
      password: 123456
    secondary:
      url: jdbc:mysql://localhost:3306/db2
      username: root
      password: 123456
```

注意事项：需指定一个 `@Primary`；JPA 或 MyBatis 需分别配置 `EntityManagerFactory`/`SqlSessionFactory` 和事务管理器；通过 `@Qualifier` 注入指定的数据源 Bean。

## 2.11 Spring Boot 中如何实现优雅停机？
```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

优雅停机的行为：
1. 收到关闭请求（如 SIGTERM 或 `/actuator/shutdown`）后，不再接收新请求。
2. 等待正在处理的请求完成（超时时间到则强制关闭）。
3. 销毁 Spring 容器中的 Bean（`@PreDestroy`、`DisposableBean.destroy()`）。
4. 关闭嵌入式 Web 容器。

## 2.12 Spring Boot 如何实现热部署？
通过 `spring-boot-devtools` 实现：
1. 引入依赖：`spring-boot-devtools`（只在开发期生效）。
2. DevTools 会监听 classpath 下的文件变化，自动触发**应用重启**（不是 JVM 重启，而是释放并重新加载旧的 ClassLoader，比冷启动快）。
3. 静态资源和模板变化不触发重启，直接刷新（配合 LiveReload 浏览器自动刷新）。
4. 通过 `spring.devtools.restart.exclude` 排除不需要监听的文件。

IDEA 中需开启：Settings → Build → Compiler → "Build project automatically"；Ctrl+Shift+Alt+/ → Registry → "compiler.automake.allow.when.app.running"。

## 2.13 如何自定义 Spring Boot 的 Banner？
在 `resources` 目录下放一个 `banner.txt` 文件，内容自定义。可使用的占位符：
- `${application.version}` — 版本号
- `${application.formatted-version}` — 格式化的版本号
- `${spring-boot.version}` — Spring Boot 版本
- `${application.title}` — 应用标题
- `${AnsiColor.BRIGHT_RED}` — 颜色控制（需 `spring.output.ansi.enabled=ALWAYS`）

也可通过 `banner.image` 属性设置图片 banner。

## 2.14 Spring Boot 中如何编写和执行测试？
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class UserControllerTest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @MockBean  // 替换容器中的真实 Bean 为 Mock 对象
    private UserService userService;
    
    @Test
    void testGetUser() {
        when(userService.getById(1L)).thenReturn(new User(1L, "张三"));
        
        ResponseEntity<User> response = restTemplate.getForEntity("/users/1", User.class);
        assertEquals(HttpStatus.OK, response.getStatusCode());
        assertEquals("张三", response.getBody().getName());
    }
}
```

- `@SpringBootTest` 启动完整上下文，适合集成测试。
- `@WebMvcTest` 只加载 MVC 层（Controller），需要 Mock Service。
- `@DataJpaTest` 只加载 JPA 层，使用内嵌数据库。
- `@MockBean` 替换 Spring 容器中的 Bean 为 Mockito Mock。
- `@SpyBean` 使用真实 Bean 但可通过 Mockito 进行部分行为打桩。

## 2.15 ApplicationRunner 和 CommandLineRunner 的区别和使用场景？
两者都在 Spring 容器启动完成、`ApplicationReadyEvent` 发布之前执行，用于应用启动后的初始化工作。

| 维度 | CommandLineRunner | ApplicationRunner |
|------|------------------|-------------------|
| 参数类型 | `String... args`（原始字符串数组） | `ApplicationArguments`（封装对象） |
| 参数解析 | 需手动解析 `--key=value` | 提供 `getOptionNames()`、`getOptionValues()` 等方法 |
| 使用复杂度 | 简单参数直接用 | 参数较多时更方便 |

```java
@Order(1)  // 排序
@Component
public class MyRunner implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) {
        // args.getOptionValues("name") → 获取 --name=xxx 的值
        // args.getNonOptionArgs() → 获取非选项参数
    }
}
```

## 2.16 Spring Boot 如何实现全局异常处理？
```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(BusinessException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public Result handleBusiness(BusinessException e) {
        return Result.error(e.getCode(), e.getMessage());
    }
    
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public Result handleException(Exception e) {
        return Result.error(500, "服务器内部错误");
    }
}
```

`@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`。可以通过 `@ExceptionHandler` 处理不同类型的异常，按照异常继承关系就近匹配。

如果需要返回错误页面（非 JSON），使用 `@ControllerAdvice` 并返回页面视图名。

## 2.17 Spring Boot 的启动方式有哪些？
1. **IDE 直接运行 main 方法**（开发常用）。
2. **`java -jar app.jar`**（生产常用，独立运行 Fat Jar）。
3. **`mvn spring-boot:run`**（Maven 插件运行）。
4. **`gradle bootRun`**（Gradle 插件运行）。
5. **部署到外部 Tomcat**：pom 中将打包方式改为 `war`，排除嵌入式 Tomcat，主类继承 `SpringBootServletInitializer`。Spring Boot 2.x 后推荐使用内嵌容器。
6. **使用 Systemd/Linux 服务**。

## 2.18 Logger 框架在 Spring Boot 中如何工作的？
Spring Boot 默认采用 **SLF4J（门面）+ Logback（实现）** 的组合：
- 通过 `spring-boot-starter-logging` 引入。
- 配置文件：`logback-spring.xml` 或 `logback-spring.groovy`（推荐用 `-spring` 后缀，可使用 `<springProfile>` 标签区分环境）。
- 也可切换到 **Log4j2**：排除 `spring-boot-starter-logging`，引入 `spring-boot-starter-log4j2`，配置文件为 `log4j2-spring.xml`。

日志级别从低到高：TRACE < DEBUG < INFO < WARN < ERROR < FATAL < OFF。`logging.level.root=INFO` 设置全局，`logging.level.com.example=DEBUG` 设置特定包。

## 2.19 Spring Boot Admin 的作用和原理是什么？
Spring Boot Admin（SBA）是一个用于管理和监控 Spring Boot 应用的社区项目，采用 **Server-Client** 架构：

- **SBA Server**：独立运行的 Web 应用，提供监控仪表盘，展示所有注册 Client 的运行状态。
- **SBA Client**：应用引入 `spring-boot-admin-starter-client`，启动后通过 HTTP 向 Server 注册（周期性心跳），暴露 Actuator 端点供 Server 采集数据。

**功能**：查看应用的 Health、Metrics、Loggers、线程堆栈、环境变量、缓存信息，动态修改日志级别，JVM 内存/GC 指标图表，上下线通知（邮件/钉钉/企业微信），与 Spring Cloud 集成支持服务发现。

## 2.20 @Conditional 系列注解有哪些？各自的判断条件是什么？
| 注解 | 判断条件 |
|------|---------|
| `@ConditionalOnClass` | classpath 中存在指定类 |
| `@ConditionalOnMissingClass` | classpath 中不存在指定类 |
| `@ConditionalOnBean` | 容器中存在指定 Bean |
| `@ConditionalOnMissingBean` | 容器中不存在指定 Bean |
| `@ConditionalOnProperty` | 配置文件中某个属性存在且有期望值 |
| `@ConditionalOnResource` | classpath 中存在指定资源文件 |
| `@ConditionalOnWebApplication` | 当前应用是 Web 应用（Servlet 或 Reactive） |
| `@ConditionalOnNotWebApplication` | 当前应用非 Web 应用 |
| `@ConditionalOnExpression` | SpEL 表达式计算结果为 true |
| `@ConditionalOnJava` | JDK 版本满足条件 |
| `@ConditionalOnSingleCandidate` | 容器中指定类型的 Bean 只有一个或存在 `@Primary` |
| `@ConditionalOnCloudPlatform` | 运行在指定云平台（如 Kubernetes） |

条件注解是自动配置能够"按需生效"的核心机制，也是自定义 Starter 的关键工具。

## 2.21 spring.factories 和 AutoConfiguration.imports 机制有什么区别？
这是 Spring Boot 2.x 到 3.x 的一个重要变化：

| 维度 | spring.factories（2.x） | AutoConfiguration.imports（3.x） |
|------|------------------------|-------------------------------|
| 文件位置 | `META-INF/spring.factories` | `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` |
| 格式 | Key-Value 键值对（`org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.xxx.AutoConfig`） | 每行一个全限定类名（简洁的列表格式） |
| 加载机制 | `SpringFactoriesLoader` 一次性加载所有 spring.factories 中的所有配置 | 只加载该文件中的自动配置类，速度更快 |
| 引入版本 | Spring Boot 1.0 | Spring Boot 3.0（Spring Framework 6） |

**3.x 的变化原因**：`spring.factories` 本来设计为通用 SPI 文件（加载各种类型的工厂），但被过度用于自动配置注册。3.x 将自动配置注册独立出来，语义更清晰，加载更快，启动时间约减少 10%~15%。

**兼容性**：Spring Boot 3.x 同时支持两种方式，但 `spring.factories` 中定义的自动配置会打印警告日志，建议迁移。

## 2.22 Spring Boot 如何排除特定的自动配置？
1. **@SpringBootApplication 注解中排除**：
   ```java
   @SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
   ```
2. **@EnableAutoConfiguration 注解中排除**（与方式 1 等效）：
   ```java
   @SpringBootApplication
   @EnableAutoConfiguration(exclude = DataSourceAutoConfiguration.class)
   ```
3. **配置文件排除**（推荐，不改代码）：
   ```yaml
   spring:
     autoconfigure:
       exclude:
         - org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
         - org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration
   ```

**排除的典型场景**：
- 不需要数据库但依赖中引入了 JDBC Starter → 排除 DataSource 自动配置
- 使用自己的配置替代默认的 Redis/JPA/Security 配置
- 不需要的 Starter 自动配置（通过 `@ConditionalOnClass` 触发但不想用）

## 2.23 Spring Boot 3.x 有哪些重大变化？
| 变化 | 说明 |
|------|------|
| **Jakarta EE 替代 Java EE** | `javax.*` → `jakarta.*`（如 `javax.servlet` → `jakarta.servlet`），这是最大的 breaking change |
| **GraalVM Native Image 支持** | 通过 `spring-boot-maven-plugin` 编译为原生可执行文件（AOT 编译），启动时间从秒级降至毫秒级，内存占用大幅降低 |
| **AOT（Ahead-of-Time）编译** | 在构建期执行部分 Spring 初始化逻辑（如 Bean 定义注册），生成静态代码，运行时不需要反射/动态代理 |
| **@AutoConfiguration 替代 @Configuration** | 自动配置类应使用专用注解（语义区分 + 优化 AOT 处理） |
| **spring.factories → .imports** | 自动配置注册方式变更 |
| **ProblemDetail（RFC 7807）** | 内置标准化错误响应格式 |
| **HttpStatusCode 接口** | HttpStatus 从枚举改为接口（允许自定义状态码） |
| **RestClient 替代 RestTemplate** | 全新的同步 HTTP 客户端，支持 Fluent API |
| **虚拟线程支持** | `spring.threads.virtual.enabled=true` 开启 Java 21 虚拟线程 |
| **Observability（Micrometer）** | 统一 metrics/tracing/logging，替代 Sleuth |
| **SSL 配置简化** | `spring.ssl.bundle.*` 统一管理证书 |

**迁移重点**：Jakarta EE 命名空间变更是最核心的代码改造；`spring.factories` → `.imports` 只是配置变更；团队应优先关注 Jakarta EE 和 AOT 能力。

## 2.24 @PostConstruct、InitializingBean、CommandLineRunner 的执行顺序？
在 Spring Boot 中，以下是**服务启动后的初始化机制及其执行顺序**：

| 序号 | 回调/机制 | 执行时机 | 说明 |
|------|----------|---------|------|
| 1 | 构造器 | Bean 实例化时 | 执行依赖注入前 |
| 2 | 依赖注入（@Autowired） | 构造器/Setter/字段注入 | Bean 属性设置完成 |
| 3 | `@PostConstruct` | 依赖注入完成后 | JSR-250 注解，最推荐 |
| 4 | `InitializingBean.afterPropertiesSet()` | 所有属性设置完成后 | Spring 专有接口 |
| 5 | `init-method`（XML/@Bean(initMethod=)） | 自定义初始化方法 | 最后执行 |
| 6 | `ApplicationRunner.run() / CommandLineRunner.run()` | 容器完全启动后（所有 Bean 初始化后，Web Server 启动后） | 适合做启动后一次性初始化任务 |
| 7 | `@EventListener(ApplicationReadyEvent.class)` | 同 6，通过事件监听器实现 | 更灵活的解耦方式 |

**关键区别**：`@PostConstruct`、`InitializingBean`、`init-method` 是**单个 Bean 级别的初始化**；`ApplicationRunner`/`CommandLineRunner` 是**整个应用启动完成后的回调**。如果需要用到其他 Bean 的初始化结果（如从数据库加载数据到缓存），应该用 `ApplicationRunner`；如果是当前 Bean 自身资源的初始化（如初始化连接池），用 `@PostConstruct`。
