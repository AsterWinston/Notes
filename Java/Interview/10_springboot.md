本篇覆盖 Spring Boot 的高频面试题，按"主题块 → 详细分块 → 题目 → 解答"组织。五个主题块：自动配置机制、启动流程与钩子、配置体系、嵌入式容器与 Web 特性、可观测与测试。容器之上的框架本体见 7_spring.md、8_spring_mvc.md，MyBatis/MP 整合见 6_mybatis.md、9_mybatis_plus.md。

# 1. 自动配置机制

## 1.1 装配注解与条件引擎

### 1.1.1 @SpringBootApplication 由哪三个注解组成？各自作用？

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@SpringBootConfiguration          // 本质 @Configuration：本类是配置类，可写 @Bean
@EnableAutoConfiguration          // 自动配置引擎开关（见 1.1.2）
@ComponentScan(excludeFilters = { // 默认排除 @Configuration 自身重复注册
        @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
        @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication { ... }
```

- **@SpringBootConfiguration**：标注"这是 Boot 的主配置类"，配合 @ConditionalOn... 的元数据识别；
- **@EnableAutoConfiguration**：`@Import(AutoConfigurationImportSelector.class)`——自动配置入口；
- **@ComponentScan**：**默认从启动类所在包开始**扫描——"启动类放根包"这条 Boot 铁律的出处；扫描范围错误是"@Component 明明有却注入不到"的第一排查项。

### 1.1.2 Spring Boot 自动配置原理是什么？

**结论**：一句链路——**imports 清单 → 条件过滤 → 配置类注册 → @Bean 装配**。

1. `@EnableAutoConfiguration` 经 `@Import` 引入 **AutoConfigurationImportSelector**（DeferredImportSelector，在所有用户配置类解析完**之后**才处理——保证"用户定义优先"的顺序根基，见 7_spring.md 1.2.3）；
2. `getAutoConfigurationEntry()` 加载候选清单：Boot 3.x 读各 jar 的 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`（2.x 为 spring.factories，见 1.1.4）——spring-boot-autoconfigure jar 里就 100+ 个 `XxxAutoConfiguration`；
3. **去重（AutoConfigurationImportFilter：OnClassCondition/OnBeanCondition/OnWebApplicationCondition 做批量预判，比逐个 @Conditional 解析快）→ 排除（exclude 属性 + 清单内互斥）→ 条件注解全量校验**，只留下"classpath 有对应类、有对应 Bean/缺对应 Bean、配置开关打开"的少数生效；
4. 生效的自动配置类按 @Bean 注册组件：例如 classpath 有 DataSource + 连接池类、容器没有用户自建 DataSource → `DataSourceAutoConfiguration` 产出 Hikari，`@ConditionalOnMissingBean` 保证"你定义、它让路"。

**排错实操**（加分项）：`--debug`/`logging.level...AutoConfigurationReportLoggingListener` 启动日志打印 **CONDITIONS EVALUATION REPORT**（positive/negative matches），"为什么我的 Bean 没生效/为什么它注入了"都从这里读起；`@ConditionalOnProperty(prefix, name, havingValue, matchIfMissing)` 是开关型组件的标配。

### 1.1.3 @Conditional 系列注解有哪些？各自的判断条件？

| 注解 | 条件 |
|------|------|
| @ConditionalOnClass / OnMissingClass | classpath 存在/不存在某类（**常写在配置类上防 NoClassDefFoundError**） |
| @ConditionalOnBean / OnMissingBean | 容器存在/不存在某 Bean（**判定依赖注册顺序，建议只用于自动配置类之间**） |
| @ConditionalOnProperty | 属性值匹配（havingValue；matchIfMissing 控制"没配算不算开"） |
| @ConditionalOnResource | 存在指定资源文件 |
| @ConditionalOnWebApplication / NotWebApplication | Web 类型（SERVLET/REACTIVE）匹配 |
| @ConditionalOnExpression | SpEL 为真（兜底武器，可读性差） |
| @ConditionalOnJava | JDK 版本 |
| @ConditionalOnSingleCandidate | 该类型唯一或存在 @Primary |
| @ConditionalOnCloudPlatform | 云平台（Kubernetes 探测等） |

**易错**：`@ConditionalOnBean` 在**用户配置类**上使用时，BeanDefinition 注册顺序不受控导致"时灵时不灵"——官方文档明确它只适合自动配置场景；用户侧"按缺省让路"应靠 @Primary/@Qualifier 或自己显式定义覆盖。

### 1.1.4 spring.factories 和 AutoConfiguration.imports 有什么区别？

| 维度 | spring.factories（1.x~2.x） | AutoConfiguration.imports（3.x） |
|------|------------------------------|----------------------------------|
| 位置 | `META-INF/spring.factories` | `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` |
| 格式 | key=逗号分隔类名（Properties 语法） | **每行一个全限定类名**（列表语义） |
| 定位 | 通用 SPI：Initializer/Listener/FailureAnalyzer…全塞一个文件 | 自动配置注册**专用**文件 |
| 加载 | SpringFactoriesLoader 全文件解析再筛 key | 只读这一个文件，**启动加载量显著下降**（官方口径 ~10-15%） |

3.x 兼容读 spring.factories 的 EnableAutoConfiguration 键但打 deprecation 警告——自研 starter 迁移：`@EnableAutoConfiguration` 标注改为 **`@AutoConfiguration`**（自带 before/after 顺序属性，替代"靠隐式加载顺序"的旧约定）；Initializer/Listener 等其他 SPI 仍由 spring.factories 承载。

### 1.1.5 如何排除不需要的自动配置？

1. 注解排除：`@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)`；按类名排除（依赖里还没有该类时用）`excludeName = "org.springframework...DataSourceAutoConfiguration"`；
2. 配置排除（**推荐**，改配置不改代码、可按环境差异化）：

```yaml
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

3. 釜底抽薪：**不引依赖自然不触发**——自动配置以 @ConditionalOnClass 为门，删 starter 比排除更干净（排除只适合"传递依赖带进来但不想生效"）。

典型场景：引了 web 却不想开 DispatcherServlet 默认映射、starter 传递依赖带来的 Redis/Security 自动配置、测试类里排除 MailSender 等外联自动配置。

## 1.2 Starter 体系

### 1.2.1 Starter 是什么？版本怎么管理？

**Starter = 依赖描述符 + 自动配置**：一个 pom 把"某能力需要的全部 jar"聚合，配合 autoconfigure 模块的条件装配，实现"引一个坐标、得一套可用默认配置"。

命名规范：官方 `spring-boot-starter-{name}`（如 spring-boot-starter-web）；第三方 **`{name}-spring-boot-starter`**（如 mybatis-plus-boot-starter）——方向词不同是刻意区分，评审外部 starter 先认准命名。

**版本管理**：`spring-boot-starter-parent`（或 `dependencyManagement` 导入 spring-boot-dependencies BOM）统一钉住全家 jar 版本——**子依赖不写 version**，升级只动 parent 一处；自己引了 BOM 外的版本要警惕"两套版本打架"（NoClassDefFound/方法签名不匹配高发源，见 3_maven.md）。

### 1.2.2 如何自定义一个 Starter？

**两模块结构（官方推荐拆分，也可单模块）**：

1. `xxx-spring-boot-autoconfigure`：核心代码——

```java
@AutoConfiguration                              // 3.x；2.x 用 @Configuration
@EnableConfigurationProperties(MyProperties.class)
@ConditionalOnClass(MyService.class)
public class MyAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean                   // 用户自定义优先，框架让路
    public MyService myService(MyProperties props) {
        return new MyService(props.getAddress(), props.getTimeout());
    }
}

@ConfigurationProperties(prefix = "my.client")  // 配合 processor 生成 IDE 提示
public class MyProperties {
    private String address;
    private Duration timeout = Duration.ofSeconds(3);  // 宽松绑定：3s/100ms/PT3S
}
```

2. 注册清单：`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 写一行全限定类名；
3. `xxx-spring-boot-starter`：pom 仅聚合依赖（autoconfigure + 核心库 + optional 标记的增强依赖）；
4. 体验完善：`spring-boot-configuration-processor`（optional）生成 `spring-configuration-metadata.json` → application.yml 里**属性自动补全 + 类型校验**；条件里用 `@ConditionalOnProperty(prefix="my.client", name="enabled", matchIfMissing=true)` 提供开关。

**验收标准**：使用方只引 starter 坐标 + 几行 yml 即可用，不写任何 @Bean 配置——做不到就说明"默认值"设计得不够好。

# 2. 启动流程与生命周期钩子

## 2.1 启动主线与事件

### 2.1.1 SpringApplication.run() 的完整流程？

**构造阶段（new SpringApplication(Main.class)）**：

1. **推断 WebApplicationType**： classpath 有 `jakarta.servlet.Servlet` + WebMvc 相关类 → SERVLET；有 WebFlux（ReactiveWebServerFactory）→ REACTIVE；都没有 → NONE（见 4.1.1）；
2. 加载 **ApplicationContextInitializer** 与早期 **ApplicationListener**（SPI：3.x 用 `META-INF/spring/....ApplicationContextInitializer` 文件，2.x 走 spring.factories）；
3. 推断主源类（首个非匿名 main stack 元素）。

**run() 阶段按序**：

1. `ApplicationStartingEvent` → BootstrapContext；
2. **准备 Environment**：命令行（`SpringApplication.run(args)` 的 `--key=value`）→ profile 文件 → 系统属性 → 环境变量逐层叠加（见 3.1.1）→ `ApplicationEnvironmentPreparedEvent`；
3. 打印 Banner（可定制，见 4.2.2）；
4. 创建上下文：`createApplicationContext()`——**有容器的 Servlet 应用创建 `AnnotationConfigServletWebServerApplicationContext`**（上下文持有"WebServer 工厂"而不是直接建 Tomcat）；
5. `prepareContext`：挂 Environment、**执行所有 Initializer**、`registerShutdownHook`、加载主配置类 BeanDefinition → `ApplicationContextInitializedEvent`/`ApplicationPreparedEvent`；
6. **`refreshContext()`**——IoC 全家桶（BeanFactoryPostProcessor→BeanPostProcessor→实例化单例→**内嵌 WebServer 在 ServletWebServerApplicationContext.onRefresh() 里创建并启动**，见 7_spring.md 1.1.2）→ `ContextRefreshedEvent` → `WebServerInitializedEvent`；
7. `afterRefresh`： runners 执行——**ApplicationRunner/CommandLineRunner 按 @Order 排序依次 run**（见 2.1.3）；
8. `ApplicationStartedEvent` →（就绪后）`ApplicationReadyEvent`；异常则 `ApplicationFailedEvent` + FailureAnalyzers 友好报告。

### 2.1.2 启动过程有哪些事件？分别能挂什么逻辑？

事件时间线（全部经 SpringApplicationRunListener 广播，监听方式：`@EventListener` / spring.factories 注册的 listener）：

```text
ApplicationStartingEvent            → 最早，容器未建：日志系统预热
ApplicationEnvironmentPreparedEvent → Environment 就绪：动态改 profile/属性源（配置中心客户端在这拉远程配置）
ApplicationContextInitializedEvent  → 上下文已建未加载定义：框架级初始化
ApplicationPreparedEvent            → 定义将加载：Banner/日志收尾
ContextRefreshedEvent（框架级）      → 所有单例就绪：可安全 getBean 的起点
WebServerInitializedEvent           → 端口就绪：注册到注册中心（Nacos 等）前能拿到真实端口
ApplicationStartedEvent             → runners 前
ApplicationReadyEvent               → 应用可对外服务：预热缓存、消费开关放行、健康上报
ApplicationFailedEvent              → 启动失败：告警
```

**易混**：① Nacos/Eureka 注册时机在 **WebServerInitialized/ReadinessChangeEvent**，不是 Bean 就绪；② `AvailabilityChangeEvent(Liveness/Readiness)` 是 Boot 2.3+ 探针体系（配合 K8s readiness，见 32_kubernetes.md）；③ 想"数据预热完再放流量"应挂 ApplicationReadyEvent 之后的自定义 gate，而不是 @PostConstruct 里阻塞 refresh。

### 2.1.3 ApplicationRunner 和 CommandLineRunner 的区别？启动初始化回调的执行顺序？

**Runner 二选一**（容器启动完成后、Ready 之前执行，异常会终止启动）：

| 维度 | CommandLineRunner | ApplicationRunner |
|------|-------------------|-------------------|
| 参数 | 原始 `String... args` | `ApplicationArguments`：`getOptionValues("--name")` 解析好的键值、`getNonOptionArgs()` |
| 选择 | 简单场景 | 参数多/要解析 `--key=value` |
| 排序 | 实现 `@Order`/Ordered，**全局混排** | 同左 |

**全链路初始化顺序表**（高频追问，串起 7_spring.md 1.4.2）：

```text
构造器 → 依赖注入 → @PostConstruct → InitializingBean.afterPropertiesSet
→ init-method → （其他 Bean 循环）→ 容器 refresh 完成 → WebServer 启动
→ ApplicationRunner/CommandLineRunner → ApplicationStartedEvent
→ @EventListener(ApplicationReadyEvent) → 开始处理外部流量
```

**语义分工**：Bean 级初始化（建线程池、注册 shutdown hook、自校验）用 @PostConstruct；**依赖"全部 Bean + 真实端口"的一次性动作**（缓存预热、MQ 启动消费、定时任务调度器注册）用 Runner/ReadyEvent——写错阶段是"预热读到半初始化系统/端口未绑定就注册"两类事故的根源。

## 2.2 打包与运行形态

### 2.2.1 Fat Jar 的结构和启动原理？

**结构**（`java -jar` 可执行 jar）：

```text
META-INF/MANIFEST.MF        → Main-Class: org.springframework.boot.loader.launch.JarLauncher
                               Start-Class: com.example.MyApplication
org/springframework/boot/loader/  → 引导器与自定义 ClassLoader
BOOT-INF/classes/           → 你自己的字节码与资源
BOOT-INF/lib/*.jar          → 全部依赖 jar（嵌套 jar，不解压）
```

**原理**：JVM 只认 `Class-Path` 展平目录——Boot 用 **LaunchedClassLoader 突破"jar 内嵌套 jar"限制**（URL `jar:file:/app.jar!/BOOT-INF/lib/xxx.jar!/`，双亲委派的定制应用，见 2_jvm.md 3.2.1）：JarLauncher 引导 → 建 LaunchedClassLoader → 反射调用 Start-Class 的 main → 走标准 SpringApplication 启动。

**运维相关**：`java -jar` 可用 `-Dloader.path`（PropertiesLauncher）外挂 jar；Docker 场景开 **layers**（`spring-boot.build-image`/layertools extract）——依赖层与应用层分离，改一行代码不重传 200MB 依赖（见 23_docker.md）。

### 2.2.2 Spring Boot 应用有哪些启动/部署方式？

1. IDE 跑 main（开发）；
2. `java -jar app.jar`（**生产主流**，Fat Jar 自包含）；
3. `mvn spring-boot:run` / `gradle bootRun`（插件，本质也是 main）；
4. **War 部署外部容器**：打包 war + 排除内置 Tomcat（provided）+ 主类继承 `SpringBootServletInitializer`（容器经 Servlet 3.0 SPI `SpringServletContainerInitializer` 回调初始化，见 5_tomcat.md）——历史包袱场景才用，Boot 官方推荐内嵌；
5. 云原生形态：**Buildpacks 镜像**（`spring-boot:build-image` 直出容器镜像）/ GraalVM Native Image（见 3.2.3）/ K8s 部署（见 32_kubernetes.md）；
6. 常驻服务化：systemd/WinSW 包装 java -jar。

### 2.2.3 全局懒加载 spring.main.lazy-initialization 能加速启动吗？代价是什么？

开启后**所有非基础设施 Bean 首次使用才创建**：启动时间与大内存应用冷启动确实缩短，但——

1. **违背 Boot"启动即暴露装配错误"哲学**：缺 Bean/配置错推迟到运行期第一条请求才炸（7_spring.md 1.1.2 饿汉式价值）；
2. 首个请求承担初始化延迟（P99 劣化），预热失效；
3. 个别依赖 `ApplicationRunner` 时序的框架组件对 lazy 支持不佳（官方明确"不支持所有框架 Bean"）。

**正确姿势**：生产不全局开；只对**确定低频的重型 Bean** 单独 `@Lazy`；启动慢优先排查"连接远程资源的 @PostConstruct/健康检查探活阻塞"，而不是全局懒加载掩盖问题。

# 3. 配置体系

## 3.1 属性与绑定

### 3.1.1 外部化配置的来源和优先级？

**从高到低**（高覆盖低，Boot 3.x ConfigData/ConfigTree 语义）：

1. **命令行参数** `--server.port=8081`（`SpringApplication.run(args)` 传入的永远赢）；
2. OS 环境变量 `SERVER_PORT=8081`（K8s ConfigMap/Secret 注入主流通道）；
3. java 系统属性 `-Dserver.port`、JNDI（`java:comp/env`）；
4. **profile 专用配置**（同位置 `application-{profile}` 压过 `application`；jar 外目录 > jar 内）；
5. `application.yml/properties`（ConfigData 位置序：`classpath:/`、`classpath:/config/`、`file:./`、`file:./config/`、`file:./config/*/`——`config/` 目录天然高于根目录）；
6. `@PropertySource` 注解属性源；
7. `SPRING_APPLICATION_JSON`、SpringApplication defaultProperties。

**细节**：同位置 `.properties` 优先于 `.yml`；**宽松绑定**让 `server.port`/`SERVER_PORT`/`Server.Port` 视为同 key（System 环境源大写场景）；"配置没生效"排查顺序：优先级来源 → profile 是否激活 → key 拼写 → 绑定目标是不是 @ConfigurationProperties。

### 3.1.2 多环境 Profile 如何组织？

```yaml
# application.yml
spring:
  profiles:
    active: prod          # 可被 --spring.profiles.active / SPRING_PROFILES_ACTIVE 覆盖
    group:                # 3.x 组合：激活 prod 同时带 monitor 配置
      prod: [prod-db, prod-mq, monitor]
```

1. 文件维度：`application-{profile}.yml`；也可**多文档块**单文件内分：`---` 分隔 + `spring.config.activate.on-profile: dev`；
2. 激活途径：配置文件 / 命令行 / 环境变量 / `spring-boot.run.profiles`（maven 插件）——**容器化用环境变量激活**，镜像不带环境差异；
3. 代码判断：`Environment.acceptsProfiles(Profiles.of("prod"))`；`@Profile("dev")` 控制 Bean 注册（测试类常用 @Profile("test") 挂 mock）；
4. `spring.profiles.include`（Boot2.4 起只能在**非 profile 文档**使用/`group` 替代多数场景——旧文档 include 语义已调整）。

### 3.1.3 @ConfigurationProperties 和 @Value 的区别？

| 维度 | @ConfigurationProperties | @Value |
|------|------------------------|--------|
| 粒度 | **按 prefix 批量绑定**一组属性 | 单个 key |
| 宽松绑定 | 支持（kebab/camel/ENV 大写互认） | 不支持（写死 `server.port` 精确匹配） |
| SpEL | 不支持 | 支持 `#{...}` |
| 校验 | `@Validated` + JSR-380（@NotBlank/@DurationUnit 等） | 无 |
| 嵌套对象/集合/Map | 天然支持（List/嵌套 POJO） | 数组用 `{a,b}`，嵌套无解 |
| 类型转换 | 全套 Converter（Duration/DataSize：`3s`/`10MB`） | 基础转换 |
| IDE 提示 | 有 processor 元数据 | 无 |

注册方式三选一：`@EnableConfigurationProperties(X.class)`（自动配置类标配）、`@ConfigurationPropertiesScan`（启动类扫描包）、直接 `@Component`。**选型**：成组配置一律 @ConfigurationProperties；@Value 只留给零散单值与 SpEL 场景。

### 3.1.4 ${} 占位符、Environment 与配置元数据是什么关系？

- 解析器：**PropertySourcesPlaceholderConfigurer**（BeanFactoryPostProcessor）替换 BeanDefinition 里的 `${key:default}`；@Value 注入走 EmbeddedValueResolver，运行时解析（见 7_spring.md 5.2.2）；
- 来源抽象：**Environment = PropertySources（有序列表）+ Profiles**——新增配置源即向其 push 一个 PropertySource（Nacos/配置中心客户端都这么接入，见 17_nacos.md）；
- 类型化：`Binder.get(environment).bind("my.client", MyProperties.class)` 可**手动绑定**（动态配置刷新、多实例批量绑定 @ConfigurationProperties 的底层）；
- IDE 体验：`spring-boot-configuration-processor` 编译期生成 `spring-configuration-metadata.json`——自动补全与文档提示的来源；starter 必带（见 1.2.2）。

## 3.2 专题配置

### 3.2.1 如何配置多数据源？

**手动多套 Bean（隔离清晰）**：

```java
@Configuration
public class DataSourceConfig {
    @Primary @Bean("primaryDS")
    @ConfigurationProperties("spring.datasource.primary")
    public DataSource primary() { return DataSourceBuilder.create().build(); }

    @Bean("secondaryDS")
    @ConfigurationProperties("spring.datasource.secondary")
    public DataSource secondary() { return DataSourceBuilder.create().build(); }
}
```

要点：① `@Primary` 必给（自动配置与 @Autowired 兜底都要它）；② 每库配套**自己的** `PlatformTransactionManager`（@Transactional 用 `transactionManager` 指定，见 7_spring.md 4.3.1 第 9 条）；③ MyBatis/JPA 各自挂 SqlSessionFactory/EntityManagerFactory（@MapperScan 的 sqlSessionFactoryRef 区分包，见 6_mybatis.md 3.3.2、9_mybatis_plus.md 1.2.2）；④ 两套库共用一份 application.yml 前缀隔离。

**动态路由（读写分离/一键切换）**：引 `dynamic-datasource-spring-boot-starter` + `@DS`——与手动多 Bean 的本质差别是"**单 DataSource 门面 + ThreadLocal 路由**"，代价是事务与 @DS 的时序坑（见 9_mybatis_plus.md 4.3.4）；强一致跨库事务另需 Seata（见 21_seata.md）。

### 3.2.2 日志体系在 Boot 里怎么工作？

**默认**：SLF4J 门面 + **Logback** 实现（spring-boot-starter-logging 随 web starter 进入）。

1. **零配置起步**：console pattern 带时间/级别/PID/线程/logger；`logging.level.com.example=DEBUG` 调级；`logging.file.name|path` 落文件（logback-spring.xml 存在时 yml 部分属性让位）；
2. **为什么用 logback-spring.xml 而非 logback.xml**：`-spring` 后缀由 Boot 增强解析——可用 `<springProfile name="prod">` 分环境、`<springProperty source="...">` 读 Environment；普通 logback.xml 走 Logback 原生加载，早于 Environment 就绪，这些标签失效（高频事故）；
3. **切换 Log4j2**：排除 spring-boot-starter-logging + 引 spring-boot-starter-log4j2（配置 `log4j2-spring.xml`）——高吞吐异步日志场景；注意 jul/slf4j 桥接循环（log4j-to-slf4j 与 jul-to-slf4j 的引入关系，别同时引 log4j-slf4j-impl 和 log4j-to-slf4j）；
4. **Boot 3.4+** 引入 `logging.structured.format`（JSON 结构化日志原生化）——ELK/Loki 采集场景值得开（见 24_elastic_search.md）。

### 3.2.3 Spring Boot 3.x 有哪些重大变化？

| 变化 | 说明 |
|------|------|
| **Jakarta EE 9 命名空间** | `javax.* → jakarta.*`（servlet/validation/annotation）——最大 breaking change，老 jar 混入必炸 |
| Java 17 基线 | record/sealed/switch 模式匹配可用；虚拟线程另见下 |
| **AOT + GraalVM Native Image** | 构建期执行装配生成静态 BeanDefinition/反射元数据；毫秒级启动、内存减半——Serverless/CLI 场景，代价是构建复杂、动态代理受限 |
| `@AutoConfiguration` | 自动配置类专用注解（before/after 显式顺序）+ imports 文件替代 spring.factories（见 1.1.4） |
| ProblemDetail | RFC 7807 标准错误体（见 8_spring_mvc.md） |
| RestClient | 同步 HTTP 门面替代 RestTemplate 地位（WebClient 管响应式） |
| **虚拟线程** | Java 21 + `spring.threads.virtual.enabled=true`：Tomcat/WebMvc 请求线程改虚拟线程——IO 密集吞吐大提升，CPU 密集无感 |
| Observability | Micrometer Observation 统一 metrics+tracing（Sleuth 退位，改 micrometer-tracing），Actuator + OTel 导出 |
| `spring.config.import` | ConfigData 取代 bootstrap.yml（见 17_nacos.md）；`spring.profiles.group` |
| SSL bundle | `spring.ssl.bundle.*` 统一证书配置 |

**迁移清单**：javax→jakarta 全量扫（IDE 替换不够，核对第三方 jar 版本）→ spring.factories 迁 imports → 依赖坐标对齐 Boot3 BOM → 回归测试"事务/序列化/反射元数据"重灾区；Native Image 评估构建链后再上。

# 4. 嵌入式容器与 Web 特性

## 4.1 容器与生命周期

### 4.1.1 嵌入式容器是如何实现的？如何切换 Undertow/Jetty？

**实现机制**：Boot 不再把应用丢给外部 Tomcat，而是**在 refresh() 的 onRefresh() 阶段由 `ServletWebServerFactory` 创建 WebServer 并 start**。工厂 Bean 来自 `ServletWebServerFactoryAutoConfiguration`（见 1.1.2）内嵌的三个配置类 EmbeddedTomcat/EmbeddedJetty/EmbeddedUndertow——各自 `@ConditionalOnClass` 探测容器关键类 + `@ConditionalOnMissingBean(ServletWebServerFactory, search = CURRENT)` 让位，外层用固定顺序的 `@Import`（Tomcat→Jetty→Undertow）保证先后。**多种容器 jar 同时在 classpath 时不会报错、也不存在"按优先级仲裁"——先被导入的静默胜出，实际就是 Tomcat**：经典事故是引入 starter-undertow 忘了 exclude starter-tomcat，Undertow 工厂被 missing-bean 挡掉，应用照样跑在 Tomcat 上；若用户手工注册了多个工厂 Bean，`ServletWebServerApplicationContext.getWebServerFactory()` 则直接抛 `Unable to start ServletWebServerApplicationContext due to multiple ServletWebServerFactory beans` 启动失败。

容器定制三通道：yml `server.*`（端口/SSL/线程池 server.tomcat.threads.max）→ `WebServerFactoryCustomizer<TomcatServletWebServerFactory>`（连接器等原生对象）→ 注册 Filter/Servlet/Listener Bean（Boot 自动挂到容器，@WebFilter 配 @ServletComponentScan，见 8_spring_mvc.md 3.1.1）。

**切换**（排除 + 引入，唯一正解——别指望隐式探测）：

```xml
<dependency>
  <artifactId>spring-boot-starter-web</artifactId>
  <exclusions><exclusion>
    <artifactId>spring-boot-starter-tomcat</artifactId>
  </exclusion></exclusions>
</dependency>
<dependency><artifactId>spring-boot-starter-undertow</artifactId></dependency>
```

**验收方法**：看启动日志——`Tomcat started on port(s)` / `Undertow started on address`。忘 exclude 时应用照常启动不报错，容器仍是 Tomcat（机制见上），这是最隐蔽的一类"切换事故"。

**选型认知**：Tomcat 生态最全默认稳；Undertow 轻量高并发（阻塞 IO 下的线程模型优势）、WebSocket 场景常配；Jetty 长连接/HTTP2 老牌。差异主要在连接数-内存曲线与个别 API（maxThreads vs workerThreads）；切换必测：JSP 不支持 Undertow（无 Jasper）、会话复制、访问日志格式。

### 4.1.2 优雅停机如何配置？K8s 里怎么配合？

```yaml
server:
  shutdown: graceful                          # 停止接收新请求，等在途请求完成
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s           # 每阶段等待上限，超时强制关闭
```

**行为链**：SIGTERM/`ContextClosedEvent` → WebServer 停止 accept → 等待在途请求排空（≤30s）→ 容器关闭执行销毁回调（@PreDestroy/DisposableBean，见 7_spring.md 1.4.1）→ 线程池/MQ 消费者/定时任务各自 stop（自定义 Executor 记得设 `waitForTasksToCompleteOnShutdown`）。

**K8s 完整闭环**（只答 shutdown=graceful 不算满分）：① Pod 删除先发 SIGTERM，`terminationGracePeriodSeconds` ≥ 30s；② **endpoints 摘除与 SIGTERM 存在竞态**——加 `preStop: sleep 5~10s` 让流量先停；③ 就绪探针翻 false 摘流（readiness 联动 graceful shutdown，见 32_kubernetes.md）；④ 长任务/大文件上传调大 timeout-per-shutdown-phase；⑤ 兜底：请求级幂等重试（客户端在连接中断时重发）。

## 4.2 Web 周边特性

### 4.2.1 Boot 的错误处理体系？全局异常处理和 /error 什么关系？

**分层**：

1. **业务异常**：`@RestControllerAdvice + @ExceptionHandler` 就近匹配（原理与三种校验异常见 8_spring_mvc.md 3.2.1、4.1.1）；
2. **MVC 标准异常**（400/404/405）：ResponseEntityExceptionHandler 或放行给默认链；
3. **没人处理的**：异常到容器 → 错误页机制 → Boot 注册 **`/error`（BasicErrorController）**——浏览器 Accept HTML 返回白页/`error/*.html` 模板，JSON 客户端返回 `{timestamp,status,error,path}`；
4. Boot 3 可切 **ProblemDetail**（`spring.mvc.problem-details.enabled=true`，RFC 7807 标准体，见 8_spring_mvc.md 3.2.2）。

**实践**：生产环境 `server.error.include-message/stacktrace=never`（默认即收敛，别在 dev 配置惯性带上线）；`ErrorAttributes` 可定制错误体字段；**静态错误页**放 `resources/error/404.html、error.html` 即自动生效；网关/微服务链路里"谁产生错误谁负责标准体"，避免白页与 JSON 混排。

### 4.2.2 如何自定义 Banner？

`resources/banner.txt`（ASCII art）或 `banner.image`（banner.gif/png，转 ANSI 字符打印）。占位符：`${application.version}`、`${spring-boot.version}`、`${AnsiColor.BRIGHT_RED}`（需 `spring.output.ansi.enabled=ALWAYS`）。关闭：`spring.main.banner-mode=off`（日志文件环境建议关，Banner 会污染第一行日志）。实现 `Banner` 接口可完全自定义输出逻辑——彩蛋型需求的标准做法。

### 4.2.3 DevTools 热部署原理？和热重载有什么区别？

**双 ClassLoader 设计**：

- **base restart classloader**：第三方依赖 jar——不变化，重启时**不重新加载**；
- **restart classloader**：你的 classes/resources——文件变更触发**只重建容器**（restart），冷启动的"JVM 起来 + 依赖类加载"成本全省 → 秒级重启。

配套：LiveReload 服务器（浏览器插件自动刷新）、模板缓存默认关闭、remote 模式（旧）。IDEA 需开 Build project automatically + 运行时自动编译（新版 Registry 项已随版本变化，按 IDE 文档）；`spring.devtools.restart.exclude`/`trigger-file` 精控监听面。

**边界**：DevTools 是**重启**不是热替换——JVM 内对象全丢；真正的"改方法体不重启"是 HotSwap（调试器有限支持）/JRebel（付费字节码级）/Kotlin 编译通道；生产永不引入（scope=runtime + `spring.devtools.restart.enabled=false` 双重保险）。JRebel/Prod 之外，**容器化开发更多用 Skaffest/Tilt + mount 覆盖 BOOT-INF/classes** 达成同类体验（见 23_docker.md、32_kubernetes.md）。

# 5. 监控、管理与测试

## 5.1 Actuator 与监控

### 5.1.1 Actuator 有哪些端点？安全上怎么管？

| 端点 | 用途 |
|------|------|
| /health | 健康聚合（liveness/readiness 分组见 32_kubernetes.md） |
| /info | 应用信息（build/git 属性注入） |
| /metrics | Micrometer 指标（JVM/HTTP/自定义 Counter） |
| /env、/configprops | 属性源与 @ConfigurationProperties 现值（**脱敏 keys/password/secret/token**） |
| /loggers | 查看与 **POST 动态改日志级别** |
| /beans、/mappings、/conditions | 容器诊断（装配顺序、URL 表、**自动配置命中报告**——1.1.2 的 web 版） |
| /threaddump、/heapdump | 现场快照 |
| /shutdown、/schedulable tasks、/prometheus | 按需开启（prometheus 依赖 micrometer-registry-prometheus） |

**暴露规则**：默认 web 只开 health+info；`management.endpoints.web.exposure.include=health,info,prometheus,loggers` 显式白名单；`management.endpoint.shutdown.enabled=true` 才可用 /shutdown。**安全底线**：管理端口分离 `management.server.port=9091`（只对内网/探针开放）+ Spring Security 对 `/actuator/**` 鉴权（见 11_spring_security.md）——/env /heapdump 泄露 = 凭证与内存数据裸奔。

### 5.1.2 如何自定义健康检查和端点？

**HealthIndicator**：

```java
@Component("mqHealth")                      // beanName 即 health 分组名
public class MqHealthIndicator implements HealthIndicator {
    @Override public Health health() {
        try {
            broker.ping();
            return Health.up().withDetail("latencyMs", cost).build();
        } catch (Exception e) { return Health.down(e).build(); }
    }
}
```

/health 聚合**全部** indicator（一个 DOWN 整体 DOWN，可用 `management.endpoint.health.status.order/h-group` 调权重与分组——非关键依赖降级为"不影响整体 UP"是上线前必调项：`management.health.redis.enabled=false` 类开关同理）。

**自定义端点**：

```java
@Component @Endpoint(id = "cacheStats")
public class CacheEndpoint {
    @ReadOperation  public Map<String,Object> stats() {...}
    @WriteOperation public String flush(@Selector String region) {...}  // POST /actuator/cacheStats/{region}
}
```

另有 `@WebEndpoint`（精确 HTTP 映射）/`@RestEndpoint`（自定义 URL）两档；健康分组用 `@EndpointGroup` 配置（`management.endpoint.health.group.liveness.include=livenessState`）。

### 5.1.3 Spring Boot Admin 是什么原理？

**Server-Client 架构**：Server 是独立 Spring Boot 应用（`@EnableAdminServer`），**拿 Client 的 Actuator 数据渲染 UI**——Client 引 `spring-boot-admin-starter-client` 配 `spring.boot.admin.client.url`，注册方式：HTTP 自注册（周期心跳 + 实例元数据）或 **服务发现代理**（Eureka/Nacos/K8s 场景不再单独注册，SBA Server 直接订阅注册中心，见 17_nacos.md、19_spring_cloud_gateway.md）。

**能力**：实例上下线告警（邮件/钉钉/企微 webhook）、在线改日志级别、JVM/HTTP 指标图表、threaddump/heapdump 一键抓取、Env/Mappings/Beans 查看、连接 SQL（可选）——本质是 **Actuator 的集中可视化 + 操作台**，与 Prometheus/Grafana（指标时序）、Loki/ELK（日志）互补：SBA 偏"运维即时操作"，指标监控偏"趋势与告警"。安全注意：SBA Server 能操作 Client，**部署在管控网并鉴权**。

## 5.2 测试体系

### 5.2.1 Boot 的测试注解体系怎么写？

**金字塔**（全量上下文最贵，优先切片）：

```java
@SpringBootTest(webEnvironment = RANDOM_PORT)   // 全量上下文 + 真随机端口
class OrderIntegrationTest {
    @Autowired TestRestTemplate rest;            // 无鉴权版 RestTemplate
    @MockBean PaymentClient client;              // 容器内替换真实 Bean 为 Mockito mock
    @MockitoBean ...                             // Boot 3.4+ 新名（@MockBean 已弃用）
    @DynamicPropertySource static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", mysqlContainer::getJdbcUrl);  // Testcontainers 注入
    }
    @Test void create() { ... }
}
```

| 注解 | 加载范围 | 场景 |
|------|---------|------|
| @SpringBootTest | 全部 | 端到端集成 |
| @WebMvcTest | Controller+MVC 链 | MockMvc 测路由/校验/序列化（mock Service） |
| @JsonTest | Jackson 组件 | DTO 契约 |
| @DataJpaTest/@MybatisPlusTest | JPA/MP 切片+内嵌库 | 仓储层（@AutoConfigureTestDatabase 替换/Testcontainers 真库） |
| @RestClientTest | RestClient/RestTemplate | 外部调用封装 |

**工具链**：MockMvc（`@AutoConfigureMockMvc`）打 MVC 层不占端口；TestRestTemplate/WebTestClient 走真 HTTP；**@MockBean 会改容器结构——滥用导致上下文缓存失效、套件变慢**（同 mock 配置尽量复用上下文）；@SpyBean 部分打桩；Testcontainers（MySQL/Redis/Kafka 容器化真依赖）是 Boot 时代集成测试主流（见 16_mysql.md、27_redis.md）。纯业务逻辑用 JUnit5+Mockito 裸测，不起 Spring（见 java-junit 最佳实践）。