# 1. 引入子配置文件

Spring 配置文件可以有多个，并且可以在核心配置文件中使用 import 引入子配置文件。拆分的直接动因是**单一 XML 膨胀到几百上千行后，定位、review、合并冲突都会变成负担**，把关注点切开——组件扫描、数据源、事务各自成文件——每个文件回到"一屏可读完"的尺度。

例如把组件扫描单独定义到一个配置文件中：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context https://www.springframework.org/schema/context/spring-context.xsd">

    <!-- 组件扫描 -->
    <context:component-scan base-package="cw.study.spring"/>

</beans>
```

在核心配置文件中使用 import 标签引入子配置文件：

```xml
<!-- 组件扫描 -->
<!-- <context:component-scan base-package="cw.study.spring"/> -->
<!-- 将组件扫描配置单独定义到一个配置文件中 -->
<!-- 引入子配置文件 -->
<import resource="spring-component-scan.xml"/>
```

在实际开发中，配置可能会很多，service 单独配置到一个文件中，dao 单独配置到一个文件中，然后在核心配置文件中引入，养成好习惯。**关键认知：无论拆成多少个文件，容器启动后所有 BeanDefinition 都注册进同一个 BeanFactory**，拆分只是物理文件组织手段，不改变逻辑上的单一容器——名字冲突、注入关系都在同一个池子里解析。

# 2. 源码解析：<import> 标签的处理机制

`<import resource="spring-dao.xml"/>` 标签在 DefaultBeanDefinitionDocumentReader 中处理：

```java
// parseDefaultElement() 中：
if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
    importBeanDefinitionResource(ele);
}
```

importBeanDefinitionResource() 的核心逻辑：

```java
protected void importBeanDefinitionResource(Element ele) {
    // 1. 获取 resource 属性值
    String location = ele.getAttribute(RESOURCE_ATTRIBUTE);

    // 2. 解析占位符（如 ${module.name}）
    location = getReaderContext().getEnvironment().resolveRequiredPlaceholders(location);

    Set<Resource> actualResources = new LinkedHashSet<>(4);

    // 3. 判断是绝对路径还是相对路径
    boolean absoluteLocation = false;
    try {
        absoluteLocation = ResourcePatternUtils.isUrl(location)
                || ResourceUtils.toURI(location).isAbsolute();
    } catch (URISyntaxException ex) { ... }

    if (absoluteLocation) {
        // 绝对路径：直接加载
        int importCount = getReaderContext().getReader()
                .loadBeanDefinitions(location, actualResources);
    } else {
        // 相对路径：相对于当前配置文件的路径
        int importCount = getReaderContext().getReader()
                .loadBeanDefinitions(location, actualResources);
    }

    // 4. 实际的加载：递归调用 XmlBeanDefinitionReader.loadBeanDefinitions()
    //    将子配置文件中的 <bean> 全部注册到同一个 BeanFactory 中

    // 5. 触发导入事件
    Resource[] actResArray = actualResources.toArray(new Resource[0]);
    getReaderContext().fireImportProcessed(location, actResArray,
            extractSource(ele));
}
```

关键点归纳：

1. `<import>` 只是物理上将配置拆分到多个文件，最终所有 BeanDefinition 都注册到同一个 DefaultListableBeanFactory 中；
2. 不支持同名 Bean 的覆盖（除非 allowBeanDefinitionOverriding=true）；
3. 相对路径是相对于当前 XML 配置文件的位置，不是 classpath 根路径——`import resource="spring-dao.xml"` 与当前 XML 同目录查找，跨目录要写 `classpath:xxx/spring-dao.xml`；
4. 使用通配符 `<import resource="classpath*:spring-*.xml"/>` 可以导入多个文件。

> **注意**：第 3 点在 Spring Boot 里口径相反——Boot 2.1 起 `spring.main.allow-bean-definition-overriding` 默认 false，XML 引入与注解 bean 重名时启动直接报 BeanDefinitionOverrideException，这是很多老项目升级 Boot 后的第一坑；同名不是"后者覆盖前者"的隐式合并，而应视为需要消灭的配置事故。

Java Config 方式的等价操作：

```java
@Configuration
@Import({DaoConfig.class, ServiceConfig.class})
public class AppConfig { }
```

# 3. @Import 的三种形态

`@Import` 表面上是"引入另一个配置"，实际按被导入类型分三支，能力逐级增强（解析入口在 ConfigurationClassParser，见 21 篇源码篇）：

```text
@Import(X.class)
   ├── X 是 @Configuration/@Component 类   → 直接注册为 Bean，等价于把它纳入容器扫描
   ├── X 是 ImportSelector 实现            → 运行时动态返回一组类名字符串（条件选"哪些配置类"）
   └── X 是 ImportBeanDefinitionRegistrar  → 拿到 Registry 手工 registerBeanDefinition（精雕"每个 Bean"）
```

**形态一：导入配置类/普通组件**。最简单的一支，把没被扫描包覆盖的类拉进容器：

```java
@Configuration
@Import({AppConfig.class, DaoConfig.class})
public class MainConfig { }
```

被导入的 @Configuration 类同样经过增强（CGLIB 代理保证 @Bean 互调单例），即使没加 @Configuration 注解也会被注册。

**形态二：ImportSelector——按条件挑选配置类**。返回的是类名数组，选择逻辑在 `selectImports()` 里动态执行，还能感知 Environment 与 Metadata。Spring 事务注解的开关就是它：`@EnableTransactionManagement` 通过 `@Import(TransactionManagementConfigurationSelector.class)` 按 AdviceMode 决定导入 PROXY 模式还是 ASPECTJ 模式的两组配置类（见 25 篇）；Boot 自动装配的 AutoConfigurationImportSelector 更是这一形态的极致——读 `AutoConfiguration.imports` 文件、跑条件过滤后返回几十个配置类名。手写示例：

```java
public class DevProdSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata metadata) {
        // 依据环境返回不同配置类的全限定名
        return "prod".equals(System.getProperty("env"))
                ? new String[]{"com.study.ProdDsConfig"}
                : new String[]{"com.study.DevDsConfig"};
    }
}

@Configuration
@Import(DevProdSelector.class)
public class MainConfig { }
```

**形态三：ImportBeanDefinitionRegistrar——手工注册 BeanDefinition**。直接操作 `BeanDefinitionRegistry`，适合"生成一串动态 Bean"的场景：@EnableAspectJAutoProxy 的 AspectJAutoProxyRegistrar 用它补注册自动代理创建器（见 24 篇），MyBatis 的 @MapperScan 底层 MapperScannerRegistrar 同款套路：

```java
public class MyBeanRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata,
                                        BeanDefinitionRegistry registry) {
        RootBeanDefinition bd = new RootBeanDefinition(RocketLauncher.class);
        bd.getPropertyValues().add("level", 5);   // 属性可以在代码里拼装
        registry.registerBeanDefinition("rocketLauncher", bd);
    }
}
```

**三形态对照**：

| 形态 | 输入 | 输出 | 典型用途 |
|------|------|------|----------|
| 配置类 | 编译期确定 | 该类及 @Bean 方法 | 跨包引入、手工补充组件 |
| ImportSelector | AnnotationMetadata + Environment | 类名字符串数组 | @Enable* 按模式选择一组配置 |
| ImportBeanDefinitionRegistrar | Registry 直接操作 | 任意 BeanDefinition | 动态生成 Bean（代理处理器、Mapper 代理） |

> **行业实践**：写自定义 @Enable* 注解时按粒度选形态——要引入"一批写好的配置类"用 Selector，要"按属性动态造 Bean"用 Registrar，纯静态补一个类直接 @Import 配置类。@Enable 生态（Feign、MyBatis、Boot Starter）大多落在后两支上，看懂这两支等于看懂注解驱动框架的骨架。

# 4. @PropertySource 加载外部属性

`@PropertySource` 是 XML `<context:property-placeholder>` 的注解等价物，把一个 properties 文件挂进 Environment 的 PropertySources 链：

```java
@Configuration
@PropertySource("classpath:jdbc.properties")
public class JdbcConfig {

    // 依赖 PropertySourcesPlaceholderConfigurer 才会替换 ${}
    @Value("${jdbc.url}")
    private String url;

    @Bean
    public MyDataSource ds() { ... }   // 取值逻辑同 13 篇
}
```

两个必知的坑。其一，**纯注解环境下 @PropertySource + @Value("${...}") 需要容器里有 PropertySourcesPlaceholderConfigurer**：原生 Spring（非 Boot）不会自动注册，要显式补一个 static @Bean（非 static 会导致解析晚于注入时机、占位符失败）。其二，**属性查找优先级链与 XML 完全相同**（系统属性 > 环境变量 > 属性文件，13 篇全表），`username` 这类与系统环境变量撞名的 key 照样被覆盖，加前缀 `jdbc.username` 的纪律不因换写法而放松。

> **注意**：@PropertySource 只支持 .properties，**原生不支持 yml**——yml 需要 Boot 的 YamlPropertySourceLoader 或手写 Factory 指定；XML 的 location 属性同样如此。看到"@PropertySource 加载不了 application.yml"的提问，多数是没分清 Spring Framework 与 Spring Boot 的能力边界。

# 5. profiles 与 @Profile：一套配置的多环境形态

profile 解决"同一份配置、不同环境取不同 Bean/属性"的问题，dev/test/prod 数据源、日志级别、Mock 依赖都靠它切换。XML 时代用 `<beans profile="dev">…</beans>` 段落包裹；注解时代（Spring 6.x）在配置类或 @Bean 方法上标注：

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @Profile("dev")
    public DataSource devDataSource() {
        return new EmbeddedDatabaseBuilder().build();   // 内存库
    }

    @Bean
    @Profile("prod")
    public DataSource prodDataSource() {
        return new DruidDataSource();                   // 真实连接池
    }
}
```

激活方式：`setActiveProfiles("prod")`（Environment API，测试常用）、JVM 参数 `-Dspring.profiles.active=prod`、或属性文件指定。**匹配规则**：@Profile("!dev") 表示非 dev 生效；同名 Bean 分放不同 profile 时只有激活组的会被注册，绕开了第 2 节的重名覆盖问题——这才是多环境配置的正确打开方式，而不是复制两份完整 XML 靠 import 切换。组合姿势通常是：**主配置负责 import/扫描 + 各环境子配置文件以 profile 包裹**，激活哪个环境就装配哪一撮。

# 6. 什么时候该拆：拆分边界

拆分不是目的，**降低变更耦合**才是。实践中的边界判据：

**该拆的信号**：

- 单文件超过约 300 行，或一个文件里混着数据源、MQ、缓存、安全四种不相干关注点；
- 不同文件由不同角色维护（DBA 改连接池、架构组改拦截器），合并冲突频繁；
- 存在多环境差异配置（按 profile 拆，或按 dev/prod 文件拆）；
- 模块复用——把组件扫描/通用基础设施抽成一个文件被多个应用 import。

**不该拆的信号**：

- 为拆而拆：三五个 Bean 也切出七八个文件，import 链比配置本身难读；
- 有强先后依赖的配置被拆散（BeanFactoryPostProcessor 类配置和它的业务上下文分属两文件，调试时来回跳）；
- Java Config 时代用 @Import 制造与 XML 等价的碎片——注解配置的可读性来自就近原则，一个领域一个配置类即可，不必复刻"主文件 import 子文件"的仪式。

> **设计争议**：XML 多文件 + import 与 Java Config 的 @Import 谁更"大型项目友好"？XML 阵营强调配置与代码分离、非开发角色可维护、diff 干净；注解阵营强调重构安全、IDE 可跳转、类型检查。现实走向是**Boot 一统后 XML 退出业务工程**，大型项目的"配置组织"问题转化为"Starter/自动配置 + application-{profile}.yml + 配置中心"三层，@Import 的 Selector/Registrar 两支反而成了框架作者的标配工具。

# 7. XML import 与 @Import 对照

| 维度 | XML `<import>` | Java Config `@Import` |
|------|----------------|---------------------|
| 物理单元 | 另一个 XML 文件 | 配置类 / Selector / Registrar |
| 路径语义 | 相对当前 XML；支持 classpath*: 通配 | 类引用，编译期确定；Selector 可运行期动态 |
| 占位符 | resource 属性支持 ${} | 不适用（无路径） |
| 条件装载 | profile 包裹 / 外部脚本拼接引入列表 | @Profile、Selector 动态返回 |
| 同名 Bean | 默认不允许覆盖 | 同左（Boot 默认关覆盖） |
| 最终归属 | 同一个 BeanFactory | 同一个 BeanFactory |

两张"地图"的目的地一致：**把分散的定义汇入一个容器**。理解这一点，XML 到注解的迁移就不是语法翻译，而是同一机制的两种书写。

# 8. 大型项目配置管理实践

把本篇机制收敛成五条纪律：

1. **入口唯一**：无论 XML 的 applicationContext.xml 还是 Boot 的 @SpringBootApplication，保持"一个可解释的装配入口"，import/@Import 链深度控制在两层以内；
2. **按领域拆分，不按行数拆分**：datasource/mq/cache/security 各归其位，一个领域一个配置类（或一个 XML），拆的边界就是团队的职责边界；
3. **环境差异只走 profile**：代码和主配置里不出现 if(env)，dev/prod 差异收口在 @Profile 段落与 application-{profile} 文件；
4. **属性集中、key 加前缀**：外部化配置统一进 Environment（@PropertySource/配置中心），避开系统属性优先级陷阱，敏感项放配置中心；
5. **框架化复用交给 @Enable***：跨项目共享的"一撮 Bean"做成注解 + Selector/Registrar，而不是复制粘贴 XML 片段——这是 @Import 三形态最大的工程红利。

# 9. 小结

- **XML 的 `<import>` 与注解的 `@Import` 是同一件事的两种写法**：物理拆分、逻辑单容器；相对路径、同名不覆盖、通配符导入、最终汇入同一 BeanFactory 的语义完全对仗。
- **`<import>` 由 DefaultBeanDefinitionDocumentReader 处理**，递归调用 loadBeanDefinitions 并触发 importProcessed 事件；Boot 默认关闭 bean 定义覆盖，重名即报错。
- **@Import 三形态能力递增**：配置类=静态引入；ImportSelector=按元数据/环境挑选一组配置（@Enable* 与 Boot 自动装配的骨架）；Registrar=手工注册任意 BeanDefinition（AOP 代理创建器、Mapper 扫描）。
- **@PropertySource 只是把文件挂进 Environment**，占位符解析仍依赖 PropertySourcesPlaceholderConfigurer 与系统属性优先链；不支持 yml 是 Framework 与 Boot 的边界。
- **拆分按领域与角色，不按行数**；环境差异交给 profile；跨项目复用做成 @Enable*——大型项目的配置管理，最终拼的是边界清晰度。
