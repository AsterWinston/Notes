本篇覆盖 Spring 框架（IoC/AOP/事务/事件）的高频面试题，按"主题块 → 详细分块 → 题目 → 解答"组织。五个主题块：IoC 容器与 Bean 管理、作用域与循环依赖、AOP、事务、事件与其他。Spring MVC 见 8_spring_mvc.md，Spring Boot 自动配置见 10_springboot.md。

# 1. IoC 容器与 Bean 管理

## 1.1 容器总览

### 1.1.1 什么是 IoC？什么是 DI？解决了什么问题？

**结论**：IoC（控制反转）指"对象的创建与依赖装配"的控制权从程序代码转移到容器——传统写法 `new Service()` 是程序主动找依赖，IoC 是容器把依赖"喂"给程序。DI（依赖注入）是 IoC 的落地手段：容器运行时把协作对象注入目标（构造器/setter/字段）。

解决的问题：

1. **解耦**：面向接口编程，调用方不感知实现类的构造细节（符合 DIP，见 1_java.md 1.2.5）；
2. **可测试**：依赖可 mock 替换，单测无需启动全链路；
3. **统一管理**：Bean 生命周期（创建→初始化→销毁）、作用域、代理增强收口容器；
4. **配置化装配**：换实现改配置不改源码（多数据源、多策略扩展）。

> **易错**：IoC 不是 Spring 发明（控制反转是 Martin Fowler 提出的思想），Spring 是 DI 容器的最主流实现；没有 DI 的 IoC 还有"依赖查找"（JNDI、`context.getBean()` 主动拉取）这条分支。

### 1.1.2 BeanFactory 和 ApplicationContext 有什么区别？

| 维度 | BeanFactory | ApplicationContext |
|------|-------------|-------------------|
| 层级 | IoC 顶层接口（延迟加载） | BeanFactory 子接口（启动预初始化单例） |
| 加载时机 | 首次 `getBean()` 才创建 | `refresh()` 时实例化所有非懒加载单例 |
| 功能 | 基础 IoC/DI | 国际化 MessageSource、事件发布、ResourceLoader、Environment、自动注册 BeanPostProcessor |
| 场景 | 理论上的资源受限环境 | 企业应用唯一现实选择 |

饿汉式初始化的真正价值：**启动即暴露配置错误**（缺依赖、装配冲突当场失败），而不是运行到某条分支才炸——生产发布的"快速失败"哲学。

### 1.1.3 BeanFactory 和 FactoryBean 的区别？

完全不相干的两个概念，命名相近是高频考点：

- **BeanFactory**：IoC 容器本体的顶层接口，负责**管理所有 Bean**（getBean/创建/装配/生命周期）。
- **FactoryBean`<T>`**：一个**特殊的 Bean**，实现了它，容器里存的不再是它本身而是 `getObject()` 返回的产品：

```java
public interface FactoryBean<T> {
    T getObject() throws Exception;
    Class<?> getObjectType();
    default boolean isSingleton() { return true; }
}
```

`getBean("x")` 拿到产品对象；`getBean("&x")` 才拿到 FactoryBean 本体。

**动机**：创建过程复杂的对象（需要多步配置、代理生成、第三方协议连接）不想让容器按普通 Bean 流程走。典型应用：MyBatis `SqlSessionFactoryBean`（见 6_mybatis.md 3.3.1）、`ProxyFactoryBean`（AOP 代理）、Feign 客户端、JNDI `JndiObjectFactoryBean`。

### 1.1.4 PO、VO、DAO、BO、DTO、POJO 的区别？

| 术语 | 全称 | 职责 | 所在层 |
|------|------|------|--------|
| PO | Persistent Object | 与表一一对应的持久化对象（@TableName/@Entity） | DAO 层 |
| DAO | Data Access Object | 数据访问接口（Mapper/Repository 均属此类角色） | 数据层 |
| BO | Business Object | 业务逻辑组合对象（多 PO 聚合出计算结果） | Service 层 |
| DTO | Data Transfer Object | 跨服务/跨层传输（RPC 出入参、MQ 消息体） | 服务边界 |
| VO | View Object | 面向前端展示（裁剪、格式化、组合） | 接口出参 |
| POJO | Plain Old Java Object | 无框架约束的普通 Java 对象的统称 | —— |

**工程意义**：分层模型防止"一个实体类走天下"——数据库字段泄露到前端（安全/耦合）、前端参数直通 update（过度绑定）。转换成本用 MapStruct 等编译期映射器消化。

## 1.2 Bean 的定义与注册

### 1.2.1 @Component 和 @Bean 的区别？@Service/@Repository/@Controller 呢？

- **@Component**（类级）：标注类自身，组件扫描发现后注册；适合**自己写的类**。`@Service/@Repository/@Controller/@Configuration` 都是它的**派生注解**（`@Component` + 语义 + 部分附加行为：@Repository 触发异常转译 SQL→DataAccessException，@Configuration 带 CGLIB 增强，@Controller/@RestController 被 MVC 识别）。
- **@Bean**（方法级）：写在 `@Configuration` 类的方法上，方法返回值注册为 Bean，方法名即默认 beanName；适合**第三方类**（jar 里的类不能加注解）或**需要复杂构造/条件逻辑**的对象（DataSource、HttpClient、`new ObjectMapper()`）。

**追问**：@Bean 方法参数会自动注入依赖；同一类型两个 @Bean 且参数名对不上 beanName 时报 NoUniqueBeanDefinition——参数名当限定符是隐式规则。

### 1.2.2 @ComponentScan 的工作原理和过滤规则？

启动扫描流程：`ClassPathBeanDefinitionScanner.doScan()` → 按 basePackages 递归读 `.class` 资源（`PathMatchingResourcePatternResolver`）→ `@Component` 元注解链判定（元可标注：自定义注解上标 @Component 也能被识别，**Spring 支持注解的注解解析**）→ 生成 AnnotatedGenericBeanDefinition 注册进容器。

属性要点：`basePackages`（字符串，可占位符 `${}`）、`basePackageClasses`（类型安全防重构改名）、`includeFilters/excludeFilters`（FilterType：ANNOTATION/ASSIGNABLE_TYPE/REGEX/CUSTOM/ASPECTJ）、`lazyInit`、`nameGenerator`（bean 命名策略）。

**易错**：不写 basePackages 时默认从**启动类所在包**扫（Boot 约定"包结构即边界"的由来）；重复扫描同类型两次——后者覆盖前者，beanName 相同会静默换实现。

### 1.2.3 @Import 和 @Enable 系列注解的原理？

`@Import` 可导入**四种东西**（Spring Boot 自动装配的基石）：

1. **普通 @Configuration/组件类**：直接注册；
2. **ImportSelector 实现**：`String[] selectImports(...)` 返回类名数组动态决定导入谁——`@EnableAutoConfiguration` 背后 `AutoConfigurationImportSelector` 读候选清单并按 `@Conditional` 过滤（见 10_springboot.md）；
3. **ImportBeanDefinitionRegistrar 实现**：`registerBeanDefinitions()` 里用 `BeanDefinitionRegistry` **编程式注册**——`@MapperScan` 的 MapperScannerRegistrar 即此（见 6_mybatis.md 3.3.2）；
4. **DeferredImportSelector**：等所有配置类解析完再处理（自动配置专用，保证用户 Bean 优先）。

**@EnableXxx 的本质**：几乎都 = `@Import` 某个 Selector/Registrar/配置类——`@EnableTransactionManagement` 导入 ProxyTransactionManagementConfiguration（事务三件套 Bean）、`@EnableAsync`、`@EnableAspectJAutoProxy`、`@EnableScheduling` 同理。**自定义中间件开关**就照抄这个套路：注解 + Registrar。

### 1.2.4 什么是 BeanDefinition？和 Bean 实例什么关系？

**BeanDefinition 是"Bean 的图纸"**：扫描/解析阶段（XML、注解、手动 API）都不创建实例，先产出 BeanDefinition 存进容器注册表，实例化阶段按图纸执行。内容：目标类名、scope、lazy-init、构造参数、属性值、init/destroy 方法、工厂方法与工厂 Bean、是否 primary/abstract 等。

实现类谱系：`ScannedGenericBeanDefinition`（扫描产物）→ `AnnotatedGenericBeanDefinition` → `ConfigurationClassBeanDefinition`（@Bean 方法）→ `RootBeanDefinition`（解析后统一形态，getBean 实际消费的对象）。

**工程价值**：`registry.registerBeanDefinition("ds", bd)` 可**运行时动态注册 Bean**——多租户动态数据源、按配置批量注册 Feign/Mapper、starter 的批量装配都靠它。

## 1.3 依赖注入

### 1.3.1 构造器注入、Setter 注入、字段注入的区别？为什么官方推荐构造器？

| 维度 | 构造器注入 | Setter 注入 | 字段注入（@Autowired 标字段） |
|------|-----------|-------------|------------------------------|
| 时机 | 实例化即注入 | 实例化后调 setter | 实例化后反射直写 |
| final 字段 | 可以 | 不行 | 不行 |
| 对象完整性 | 构造完成即完整可用 | 可先建后补（可选依赖友好） | 同 setter |
| 循环依赖 | 无法靠三级缓存解 | 可 | 可 |
| 单测 | `new Service(mockA, mockB)` 直接测 | 调 setter | 反射或启动容器 |
| 依赖显式性 | 构造签名即依赖清单（坏味道：参数超 5 个该拆类） | 显式 | **隐藏** |

推荐构造器的三条理由：**不可变**（final）、**完整性**（杜绝半初始化对象）、**显式**（依赖即契约）。可选依赖用 `Optional<Xxx>`/`@Nullable` setter。**字段注入的真实问题**：脱离容器无法实例化、IDE 看不出依赖关系、子类扩展容易漏注入。

### 1.3.2 @Autowired 的注入流程？

装配引擎：`AutowiredAnnotationBeanPostProcessor`（BPP）在 populateBean 阶段工作。

1. `postProcessMergedBeanDefinition` 阶段先扫描构造器/@Autowired 字段与方法，缓存为 InjectionMetadata；
2. `InstantiationAwareBeanPostProcessor.postProcessProperties()` 执行注入：字段反射 set、配置方法由容器调用（入参即被注入项）；
3. 单个依赖解析在 `DefaultListableBeanFactory.doResolveDependency()`：
   - 类型收集候选 → `@Qualifier` 过滤（显式指定优先）；
   - 多个候选 → 看 `@Primary` → 仍多个 → 按**注入点变量名 == beanName** 匹配；
   - 0 个：`required=true` 抛 NoSuchBeanDefinitionException；
   - 集合注入（`List<Xxx>`/`Map<String,Xxx>`）：按类型收集全部实现注入，不抛异常；
4. 支持 Java 8 `@Nullable`、`Optional<Xxx>` 表达"可缺省"；自引用（A 注入 A）被排除。

### 1.3.3 @Resource 和 @Autowired 的区别？

| 维度 | @Autowired | @Resource |
|------|-----------|-----------|
| 来源 | Spring 专属 | JSR-250 标准（jakarta.annotation） |
| 默认装配 | **按类型** byType | **按名称** byName（name 属性/字段名），找不到名称再按类型 |
| 限定 | 配 @Qualifier | 用 name 属性 |
| 构造器注入 | 支持 | **不支持**（只用于字段/setter） |
| 可选性 | required=false | 无 |

**迁移提醒**：JDK11 移除 `javax.annotation` 包后 Spring 对 @Resource 的解析由 `CommonAnnotationBeanPostProcessor` 提供、依赖 jakarta 注解坐标——Boot3 用 `jakarta.annotation.Resource`。选型：纯 Spring 项目 @Autowired + @Qualifier；在意可移植性用 @Resource。

### 1.3.4 @Primary 和 @Qualifier 的协同规则？

多同类型 Bean 时的裁决顺序：

1. 按注入点类型收集候选；
2. 有 `@Qualifier`（或其组合注解）→ **按限定符精确匹配，@Primary 被忽略**；
3. 无限定符 → 有且仅有一个 @Primary → 注入它；
4. 否则按参数名/字段名匹配 beanName；
5. 全部失败 → NoUniqueBeanDefinitionException。

记忆：**@Qualifier > @Primary > 名称匹配**——Qualifier 表达"调用方意图"，永远压过"提供方默认值"。@Primary 还能配 @Qualifier 组合：为"无标注的通用消费者"保留默认实现。

## 1.4 Bean 生命周期与容器扩展

### 1.4.1 Spring Bean 的完整生命周期？

**主线四大步**：实例化 → 属性填充 → 初始化 → 销毁（源码对应 `AbstractAutowireCapableBeanFactory`）：

1. **实例化**：`createBeanInstance()`——工厂方法/构造器/Supplier 反射建对象；
2. **属性填充**：`populateBean()`——InstantiationAwareBeanPostProcessor 执行 @Autowired/@Resource 注入；
3. **Aware 回调**：`invokeAwareMethods()` 只处理三件套（BeanNameAware → BeanFactoryAware → BeanClassLoaderAware）；ApplicationContext/Env/Resource 等其他 Aware 由 `ApplicationContextAwareProcessor`（一个 BPP）在前置处理阶段回调；
4. **初始化**：`initializeBean()`：
   - BPP `postProcessBeforeInitialization()`（@PostConstruct 在此被 InitDestroyAnnotationBeanPostProcessor 触发）；
   - `InitializingBean.afterPropertiesSet()`；
   - 自定义 init-method（@Bean(initMethod)）；
   - BPP `postProcessAfterInitialization()`（**AOP 代理在此生成**）；
5. **就绪**：单例存入一级缓存 singletonObjects，供使用；
6. **销毁**：容器关闭 → @PreDestroy → DisposableBean.destroy() → destroy-method。

> **面试追问**："三级缓存提前暴露"发生在哪步之间？——实例化后、属性填充前（addSingletonFactory 在 populateBean 之前）；BPP 生成代理在初始化末尾（见 2.2.2 与 ObjectFactory 的取舍）。

### 1.4.2 @PostConstruct、afterPropertiesSet()、init-method 的执行顺序？

**顺序固定**：`@PostConstruct` → `InitializingBean.afterPropertiesSet()` → 自定义 `init-method`。

原理：三者都在 initializeBean 阶段，@PostConstruct 由"前置 BPP"执行（在接口回调之前），afterPropertiesSet 与 init-method 在 `invokeInitMethods()` 内一先一后。

选型：推荐 **@PostConstruct**（JSR-250 标准、贴近方法级、不耦合 Spring 接口）；afterPropertiesSet 是 Spring 专有侵入方案；init-method 留给"无法改源码的第三方类"。**销毁对称**：@PreDestroy → DisposableBean.destroy → destroy-method。

### 1.4.3 Aware 接口的作用和常用有哪些？

让 Bean **反向感知容器设施**的回调族（属性填充之后、初始化回调之前执行）：

| 接口 | 获得 | 典型用途 |
|------|------|---------|
| BeanNameAware | beanName | 动态注册/多实例区分自身 |
| BeanFactoryAware | BeanFactory | 主动 getBean（少用） |
| ApplicationContextAware | ApplicationContext | 全局上下文、事件发布、资源读取 |
| ApplicationEventPublisherAware | 事件发布器 | 发布领域事件 |
| MessageSourceAware | 国际化 | i18n |
| ResourceLoaderAware | 资源加载 | 读 classpath 文件 |
| EnvironmentAware | 配置环境 | 读 profile/property |
| EmbeddedValueResolverAware | StringValueResolver | 手动解析 ${} 占位符 |

**注意**：Aware 拿到的是"裸设施"，属于框架侵入式设计——新代码优先 `@Autowired ApplicationContext`？其实同样侵入；更推荐**构造器注入目标设施**（如直接注入 ApplicationEventPublisher），Aware 主要留给框架基础设施与静态持有场景（见 1.4.5）。

### 1.4.4 BeanFactoryPostProcessor 和 BeanPostProcessor 的区别？

| 维度 | BeanFactoryPostProcessor | BeanPostProcessor |
|------|--------------------------|-------------------|
| 作用对象 | **BeanDefinition（图纸）** | **Bean 实例（成品）** |
| 时机 | 所有 BeanDefinition 注册完、任何实例化**之前** | 每个 Bean 初始化的前后 |
| 能改什么 | 属性值替换、改 scope/lazy、注册新 BD、占位符解析 | 包装代理、注入处理、校验、增强 |
| 典型实现 | `ConfigurationClassPostProcessor`（解析 @Configuration/@ComponentScan/@Import，属其子接口 BDRPP）、`PropertySourcesPlaceholderConfigurer`（${} 替换）、`MapperScannerConfigurer`（注册所有 Mapper BD） | `AutowiredAnnotationBeanPostProcessor`（@Autowired）、`CommonAnnotationBeanPostProcessor`（@Resource）、`AnnotationAwareAspectJAutoProxyCreator`（AOP 代理）、`ApplicationListenerDetector` |

扩展版：`BeanDefinitionRegistryPostProcessor` 先于普通 BFPP 执行，**可注册更多 BeanDefinition**——"注册器"与"修改器"的分层让框架的框架（Boot starter）有稳定接入点。

**易错**：容器**早期**注册的 BPP 会被"过早初始化"，它依赖的 Bean 还没走完整周期——BPP 实现里慎注入其他 Bean。

### 1.4.5 静态工具类想拿 Spring Bean，有哪些做法？

工具类（`SpringUtil.getBean()` 式）本质是**把容器引用交给静态字段**，常见三种：

```java
@Component
public class SpringUtils implements ApplicationContextAware {
    private static ApplicationContext ctx;
    @Override
    public void setApplicationContext(ApplicationContext c) { ctx = c; }
    public static <T> T getBean(Class<T> t) { return ctx.getBean(t); }
}
```

1. `ApplicationContextAware`（如上，最普遍）；
2. `@PostConstruct` 给静态字段赋值注入的 ApplicationContext；
3. 构造器注入 + 赋静态变量（`@Autowired public SpringUtils(ApplicationContext c){ SpringUtils.ctx = c; }`）。

**取舍警告**：静态持有让单测无法 mock、依赖彻底隐藏、还有"Bean 未就绪就调用"的时序坑——仅限旧代码改造（遗留静态调用）或框架基建（如 Hutool/Spring 自身工具）；业务代码坚持**构造器注入**。Hutool `SpringUtil`、Spring `ApplicationContextAwareProcessor` 都是同一模式的正反教材。


# 2. 作用域、循环依赖与容器机制

## 2.1 作用域

### 2.1.1 Bean 有哪些作用域（Scope）？

| Scope | 生命周期 | 说明 |
|-------|---------|------|
| singleton（默认） | 容器级，一个实例 | 启动时创建（非 lazy），全部共享，**必须无状态才安全** |
| prototype | 每次获取新建 | `getBean()`/注入点每次触发新实例；**容器不管其销毁** |
| request | 每个 HTTP 请求 | Web 环境，请求属性级 |
| session | 每个会话 | Web 环境 |
| application | ServletContext 级 | 整个应用一份（类似 singleton 但绑 Servlet 上下文） |
| websocket | 每个 WS 连接 | 见 36_websocket.md |

配置：`@Scope(SCOPE_PROTOTYPE)` 或 XML scope 属性。

**易错**：singleton Bean 持有可变成员 = 并发事故温床（Spring 不保护你）；prototype 的"销毁回调不执行"是高频陷阱——容器创建完就脱手，关连接/清缓存要调用方自己做。

### 2.1.2 单例 Bean 里注入 prototype Bean 会怎样？怎么解决？

**问题**：单例只创建一次，注入发生在创建时——prototype 依赖也只会被塞进那**一份**，此后每次使用拿到的都是同一个"prototype"（实际退化成单例）。

四种解法：

1. **@Lookup 方法注入**：容器重写该方法，每次调用 `context.getBean(prototype.class)`：

```java
@Component
public class OrderService {
    @Lookup
    protected PrototypalHandler handler() { return null; } // 空实现即可，被子类化替换
    public void process() { handler().doSomething(); }
}
```

2. **注入 ObjectFactory/Provider**：`ObjectFactory<PrototypalHandler>` 或 `Provider`（javax.inject），每次 `getObject()/get()` 触发新建；
3. **直接注入 ApplicationContext** 手动 getBean（侵入重）；
4. **`@Scope(proxyMode = TARGET_CLASS)`**：给 prototype 注入**作用域代理**（注入的是代理壳，方法调用时才解析目标实例）——对使用方透明，推荐。

> 同理适用于 request scope 注入 singleton：作用域代理是官方兜底方案（CGLIB/JDK 代理按 exposeProxy）。

## 2.2 循环依赖

### 2.2.1 Spring 如何用三级缓存解决循环依赖？完整流程？

**三级缓存定义**（DefaultSingletonBeanRegistry）：

```text
singletonObjects（一级）       : 完全初始化的成品
earlySingletonObjects（二级）  : 提前暴露的半成品（实例化完成、属性未填满）
singletonFactories（三级）     : ObjectFactory，延迟决定"暴露原始对象还是代理"
```

**A↔B setter 循环依赖流程**：

1. `getBean(A)` → 创建 A：实例化 → `addSingletonFactory(A)`（lambda：`() -> getEarlyBeanReference(A)` 入三级）→ populateBean 发现要注入 B → `getBean(B)`；
2. `getBean(B)` 同理会把 B 的工厂入三级，再 populate 发现要 A → `getBean(A)`：
3. 查一级无 → 查二级无 → **查三级命中工厂** → 执行工厂（若 A 需 AOP 则此处生成代理，否则返回原始对象）→ 结果入**二级**、删三级 → 把这个"早期 A"注入 B；
4. B 初始化完成 → 入一级；回到 A 继续 populate（拿到成品 B）→ A 初始化完成 → 入一级，二级删。

关键：**注入给 B 的 A 引用与最终 A 是同一对象**（A 未代理时），或同一代理（A 需代理时提前生成，后续初始化照常对同一代理做填充？——实际实现是 early 暴露引用即最终引用，A 初始化结束后不再替换）。

### 2.2.2 为什么需要第三级缓存？直接用二级缓存（存早期对象）不行吗？

可以"解出循环"，但破坏 AOP 语义。三级缓存的本质是**把"要不要提前创建代理"的决定延迟到真正发生循环依赖时**：

- 若二级缓存方案直接放原始对象：
  1. 循环依赖时注入的是**原始 A**，A 初始化后又生成代理成品——**同一容器两个身份**，B 拿到旧引用，AOP 失效；
  2. 或对**所有** Bean 实例化后立刻生成代理——没发生循环依赖的 Bean 也被迫提前代理，绕过 BPP 后置扩展点，行为退化（Spring 2.x 曾靠二级缓存 + 提前代理解决）。
- 三级缓存存 `ObjectFactory`：**只有 getBean 二次进入**（即真发生循环）才执行工厂→ `getEarlyBeanReference()` 让 `SmartInstantiationAwareBeanPostProcessor`（如 AbstractAutoProxyCreator）按需生成代理；无人来要早期引用，就完全不提前代理。

**一句话**：三级缓存解决的不是循环依赖本身，而是"循环依赖 × AOP 代理"的组合——这是它区别于教科书二级方案的真正理由。

### 2.2.3 哪些循环依赖解决不了？为什么？

1. **构造器注入循环**：A 的 `createBeanInstance` 阶段就要拿到 B，而 B 要等实例化完成后才能暴露早期引用——A 还没实例化，工厂都没地方放。Spring 抛 `BeanCurrentlyCreationException: circular reference`。解法：改 setter、`@Lazy` 注入代理占位、重构拆类（循环依赖通常是设计异味）；
2. **prototype 循环依赖**：作用域机制根本不缓存 prototype——A(原型)→B(原型)→A 永远在"新建下一层"，直接栈式失败（BeanCreationException）；
3. **`@Async` 方法的循环依赖**：Async 代理靠 `AnnotationAsyncExecutionInterceptor` 后置生成，且自调用/代理时序复杂时"早期暴露的对象 ≠ 最终代理对象"，Boot 2.6+ 默认 `spring.main.allow-circular-references=false` **直接禁止循环依赖**（启动报错），这是官方态度——三级缓存是历史包袱兜底，不是鼓励信号。

> **补充**：`allowCircularReferences=true` 可回开；`@Lazy` 在构造参数上加（生成延迟解析代理）是构造器循环的官方逃生门。

## 2.3 容器内部机制

### 2.3.1 getBean() 从调用到返回经历了什么？

以 singleton 为例（`AbstractBeanFactory.doGetBean`）：

1. `transformedBeanName`（去掉 `&`/别名）→ 查 `aliasMap`；
2. **先查缓存**：`getSingleton(name)` 依次一级 → 二级（allowEarlyReference 时三级工厂也会执行）——命中直接返回（循环依赖靠这里）；
3. 未命中：`mergedBeanDefinition`（父定义合并）→ 标记 `currentlyCreated`（**检测循环依赖的标志位**）；
4. 按作用域分支：singleton → `getSingleton(name, singletonFactory)` → `createBean()`；prototype → 每次走 `createBean`（不入缓存）；
5. `createBean`：`resolveBeforeInstantiation`（BPP 有机会**短路**返回代理，如 @Async 场景之外的自定义 InstantiationAware）→ `_createBean`：实例化（AutowiredConstructorResolver 挑构造器）→ populateBean → initializeBean；
6. 完成：`addSingleton` 入一级缓存、清二三级与 currentlyCreated → 返回；异常则回滚标记。

**追问点**：prototype 的 `getObjectProvider`/ObjectFactory 复用此路径；`@Bean` 方法返回的 DisposableBean 依赖 `registerDisposableBeanIfNecessary`。

### 2.3.2 Spring 用到了哪些设计模式？

| 模式 | 体现 |
|------|------|
| 工厂方法 | BeanFactory、FactoryBean、`@Bean` 方法 |
| 单例 | DefaultSingletonBeanRegistry 缓存、容器本身 |
| 代理 | AOP（JDK/CGLIB）、@Lookup、作用域代理 |
| 模板方法 | `JdbcTemplate`（回调留给你）、`refresh()` 流程骨架、`AbstractAutowireCapableBeanFactory.doCreateBean` |
| 观察者 | 事件机制（ApplicationEvent/Listener） |
| 责任链 | `BeanPostProcessor` 链、MVC 拦截器链、`ExceptionHandlerExceptionResolver` 链 |
| 适配器 | `HandlerAdapter`（MVC 适配不同 Controller 形态）、AdvisorAdapter（AOP 通知适配 MethodInterceptor） |
| 策略 | 实例化策略（Supplier/工厂/构造器三选一）、`@Conditional` 装配策略、资源加载策略 |
| 组合 | `CompositeCacheManager`、`DelegatingApplicationListener` |
| 装饰器 | `BeanDefinitionDecorator`、事务的 TransactionAwareDataSourceProxy（弱例） |

**答题姿势**：挑 3~4 个能讲"在哪里+解决什么"的——模板方法（refresh）+ 观察者（事件）+ 代理（AOP）+ 责任链（BPP）是最能体现读源码深度的组合。

### 2.3.3 什么是父子容器？Spring MVC 里怎么体现？

`ApplicationContext` 支持 `setParent()`：**子容器可向上找父容器的 Bean，父看不到子**（getBean 逐级上溯；BeanDefinition 注册表各自独立）。

经典 Web 架构：

- **Root 容器（父）**：`ContextLoaderListener` 启动，管 Service/DAO/DataSource/事务；
- **Servlet 容器（子）**：`DispatcherServlet` 启动，管 Controller/HandlerMapping/ViewResolver。

典型事故：

1. **重复扫描**：子容器也扫了 @Service → Service 在父子各一份，事务配置在父、注入在子 → "事务不生效"；正解：父子各自限定 includeFilters/excludeFilters；
2. Controller 注入父容器 Bean 正常，父容器 Bean 想反注入子容器 Controller 不可能。

**Spring Boot**：嵌入式容器 + 单一 ApplicationContext，父子容器问题结构性消失（见 10_springboot.md）。

# 3. AOP

## 3.1 基础与实现

### 3.1.1 什么是 AOP？核心概念有哪些？

**结论**：AOP 把**横切关注点**（日志、事务、权限、监控）从业务主流程中剥离成可复用的"切面"——本质是 OCP 的开闭原则落地：加横切逻辑不改业务代码。

概念对照表：

| 术语 | 含义 | 类比 |
|------|------|------|
| Aspect 切面 | 横切逻辑的模块（类） | "日志这件事"本身 |
| JoinPoint 连接点 | 可被拦截的执行点；Spring AOP **仅方法级** | 候选插桩位置 |
| Pointcut 切点 | 匹配连接点的表达式（execution/@annotation/within） | 选择器 |
| Advice 通知 | 切点命中后做什么（五种，见 3.2.1） | 动作 |
| Weaving 织入 | 把切面应用到目标形成代理的过程 | 时机 |
| Introduction | 给目标额外实现接口（少用） | —— |

**与 OOP 关系**：OOP 纵向分层复用，AOP 横向切面复用；继承是编译期静态复用，AOP 是运行期动态增强。

### 3.1.2 JDK 动态代理和 CGLIB 的区别？Spring 如何选择？

| 维度 | JDK 动态代理 | CGLIB |
|------|-------------|-------|
| 原理 | `Proxy.newProxyInstance` 生成实现同一接口组的 $Proxy 类，InvocationHandler 转发 | ASM 生成目标类**子类**重写方法，MethodInterceptor 转发 |
| 前提 | 目标必须实现接口 | 类/方法非 final；无参/可访问构造器（Boot 3/CGLIB 4 支持 Objenesis 绕构造） |
| 性能 | 生成快；调用走反射接口分发 | 生成慢（字节码）；方法调用经 FastClass 索引更快 |
| final | 天然只能代理接口方法 | final 类/方法/private 无法拦截 |

**选择规则**：目标有接口默认 JDK；无接口 CGLIB。**Spring Boot 2.x 起 `spring.aop.proxy-target-class=true` 默认全 CGLIB**——防"注入具体类失败/接口变更引发代理类型漂移"的工程性一刀切。`@EnableAspectJAutoProxy(proxyTargetClass=true)` 同义。

**追问**：代理对象的类型？——JDK 代理 `instanceof 接口` true、强转实现类 `ClassCastException`；CGLIB 代理是子类，强转实现类可行但**自调用不过代理**、注解不继承 final 方法。深入对比见 1_java.md 1.2.4。

### 3.1.3 织入有哪三种时机？为什么 Spring AOP 用运行时织入？

1. **编译期**（AspectJ ajc 编译器）：直接产出带切面的字节码——连接点最全（字段、构造器），代价是专用编译链；
2. **类加载期 LTW**：javaagent/自定义 ClassLoader 改写字节码（AspectJ LTW、SkyWalking 探针同族）；
3. **运行期（Spring AOP）**：Bean 初始化完成后动态代理，不碰字节码加载链。

Spring AOP 选运行期的原因：**零编译依赖、零启动侵入**（加 jar 即生效），且容器本来就掌握全部 Bean 生命周期——够用场景（public 方法拦截）覆盖 95% 业务；代价：只支持方法执行连接点、只对**容器内 Bean** 生效（new 出来的对象不会被织入——AOP 失效的隐蔽根因）。

## 3.2 通知与执行细节

### 3.2.1 AOP 通知类型有哪些？@Around 强在哪？

| 通知 | 注解 | 时机 |
|------|------|------|
| Before | @Before | 方法执行前 |
| AfterReturning | @AfterReturning | 正常返回后（可接 returning 值） |
| AfterThrowing | @AfterThrowing | 异常抛出后（可接 exception） |
| After | @After | 结束无论成败（近似 finally） |
| Around | @Around | 包裹整个调用，`proceed()` 前后自主控制 |

@Around 独有能力：决定是否/何时 proceed（**缓存、限流、重试**的基座）、改参数、改返回值、吞/抛异常、测量耗时。代价：忘 proceed 目标不执行；异常处理语义要自己写对（原样 rethrow 否则下游 AfterThrowing 收不到）。

**执行顺序（单层切面）**：Around 前段 → Before → 目标 → After → Around 后段/AfterReturning(AfterThrowing)——"环绕内嵌普通通知"。

### 3.2.2 多个切面的执行顺序如何控制？

`@Order(n)`（或实现 Ordered）标在**切面类**上，数值小优先级高。代理方法调用按洋葱模型嵌套：

```text
Order(1).around前 → Order(2).around前 → before → 目标 → after → Order(2).around后 → Order(1).around后
```

工程约定：**外层放监控/日志**（要统计全量）、**内层放业务切面**；事务由 `@EnableTransactionManagement(order=)`（默认 LOWEST_PRECEDENCE-100? 实际 LOWEST）控制——想让自定义切面在事务**外**执行，order 必须比事务 Advisor 小。同优先级顺序不保证（跨版本可能变），强依赖顺序时应合并切面。


# 4. 事务

## 4.1 事务机制

### 4.1.1 Spring 事务的底层实现原理？

**结论**：声明式事务 = **AOP 代理 + TransactionInterceptor + PlatformTransactionManager** 三件套。

链路拆解（`@EnableTransactionManagement` 注册的 InfrastructureAdvisorAutoProxyCreator 只认 Advisor 类切面）：

1. 容器识别 `@Transactional`（类或方法，方法级优先）→ 为 Bean 创建代理（有接口默认 JDK，Boot 默认 CGLIB，见 3.1.2）；
2. 调用进入 `TransactionInterceptor.invoke()`：
   - `TransactionAspectSupport` 经 `TransactionAttributeSource` 解析事务属性（传播、隔离、timeout、readOnly、rollbackFor/noRollbackFor）；
   - `invokeInTransaction()` → `transactionManager.getTransaction(attr)`——按传播行为**新建/加入/挂起**事务，连接绑定到 `TransactionSynchronizationManager`（**ThreadLocal**，这就是"事务的根"）；
   - `proceed()` 执行目标方法；
   - 正常返回 → `commitTransactionAfterReturning()`；异常 → `completeTransactionAfterThrowing()` 按 rollbackFor 判定 commit/rollback（UnexpectedRollbackException 场景：内层 REQUIRES_NEW 已标 rollbackOnly，外层 commit 时爆炸）；
3. `PlatformTransactionManager` 是顶层抽象：`DataSourceTransactionManager`（JDBC/MyBatis）、`JpaTransactionManager`、`JtaTransactionManager`（分布式，见 34_distributed_theory.md）。

**本质推论**：事务生效的前提是**走代理**——这直接解释自调用失效（4.3.1）。

### 4.1.2 声明式事务和编程式事务怎么选？

- **声明式** `@Transactional`：零侵入、覆盖面广，但粒度是"方法级"——长事务里混进 RPC 调用没法细控，动态条件难表达。
- **编程式**：

```java
@Service
public class PayService {
    private final TransactionTemplate tx;   // 自动注入，默认 REQUIRED
    public void confirm(Order o) {
        saveDraft(o);                       // 事务外
        tx.executeWithoutResult(st -> {     // 事务内：精确圈范围
            deductStock(o);
            markPaid(o);
        });
        notifyMq(o);                        // 事务外：提交后再发（防"消息发了事务回滚"）
    }
}
```

`TransactionCallback<T>` 有返回值版、`execute()` 内异常照常回滚；需要更底层可手操 `PlatformTransactionManager.getTransaction/commit/rollback`（自定义传播/挂起逻辑）。

**选型**：默认声明式；**把远程调用、慢 IO 赶出事务边界**的场景（连接池占用时间 = 事务时长）用编程式显式圈小。

## 4.2 传播行为与隔离级别

### 4.2.1 事务传播行为有哪 7 种？分别适用什么场景？

| 传播行为 | 语义 | 典型场景 |
|---------|------|---------|
| **REQUIRED**（默认） | 有则加入，无则新建 | 绝大多数业务入口 |
| **REQUIRES_NEW** | 挂起当前，新建独立事务（**新连接**） | 审计日志、流水记录——主业务失败也要留痕 |
| **NESTED** | 有则嵌套（Savepoint），无则新建 | 批量导入：单条失败回滚到该条存点，不影响整批 |
| SUPPORTS | 有则用，无则非事务 | 辅助查询 |
| NOT_SUPPORTED | 挂起当前，非事务执行 | 耗时且不参与事务的操作（外部接口探活），减少连接占用 |
| MANDATORY | 必须在事务中，否则抛异常 | 强制要求被事务包裹的内部方法 |
| NEVER | 有事务抛异常 | 明确不可重入的底层方法 |

**易错**：加入（REQUIRED）= 共用同一物理事务，**回滚一体**；挂起 = 外层事务暂停，内层独立提交后外层恢复——被"挂起"的事务此刻**不会**提交或回滚，只是不再生效。

### 4.2.2 REQUIRES_NEW 和 NESTED 的区别？

| 维度 | REQUIRES_NEW | NESTED |
|------|--------------|--------|
| 物理实现 | 挂起外层 + **新开连接新事务** | **同一事务内的 Savepoint** |
| 内层回滚 | 外层不受影响（外层可选择补救） | 回滚到 savepoint，外层可继续 |
| 外层回滚 | 已提交的内层**回滚不回来**（独立提交） | 连带回滚（同事务） |
| 提交时机 | 内层方法返回即提交 | 随外层最终提交 |
| 风险 | 双重占用连接：外层持有 1 条 + 内层再取 1 条，池被外层占满时内层等连接 → **死锁式挂起**（事故高发） | 依赖 JDBC savepoint 支持；rollback-only 语义更绕 |

**选择**：日志/记账"必须留下"→ REQUIRES_NEW；批处理"坏一条不影响整批"→ NESTED。用 REQUIRES_NEW 时控制并发 ≤ 池大小/2 是经验保命法则。

### 4.2.3 事务隔离级别有哪些？Spring 怎么设置？

DEFAULT（用数据库默认）/ READ_UNCOMMITTED / READ_COMMITTED / REPEATABLE_READ / SERIALIZABLE，对应关系与脏读、不可重复读、幻读的防御矩阵见 16_mysql.md（InnoDB 默认 RR + MVCC 已基本消除幻读；Oracle 默认 RC）。

```java
@Transactional(isolation = Isolation.READ_COMMITTED, timeout = 3, readOnly = true)
```

`@Transactional` 属性全家福：**propagation / isolation / timeout（超时回滚）/ readOnly（连接只读+Hibernate flush 跳过，走从库路由依据）/ rollbackFor / noRollbackFor**。隔离级别是"每次事务执行时 SET TRANSACTION ISOLATION LEVEL"（Spring 负责恢复原级别）。

> **行业实践**：互联网 MySQL 栈常见组合 = RC + 业务层幂等/锁（Binlog ROW 格式），避开 RR 的间隙锁死锁高发；改隔离级别要全链路评估，不是单点开关。

## 4.3 失效场景与修复

### 4.3.1 @Transactional 失效的常见场景？

本质只有一条：**事务靠代理，不经过代理就不生效**。具体形态：

1. **自调用**：同类内 `this.methodB()` 直接目标对象调用，绕过代理（methodA 有事务、methodB 的注解被忽略）；
2. **非 public 方法**：Spring AOP 默认只拦截 public（protected 不报错但无效，静默）；
3. **方法被 final/static 修饰**：CGLIB 无法重写，同样静默；
4. **Bean 不归容器管**：自己 `new` 的对象（3.1.3）；
5. **异常被吞**：catch 后不上抛，代理看不到异常→正常 commit；
6. **rollbackFor 不匹配**：默认只回滚 RuntimeException/Error，**checked 异常照常提交**——业务里主动 throw 的受检异常最容易被漏；
7. **传播行为抵消**：propagation=NOT_SUPPORTED/SUPPORTS 在无外层事务时本就非事务；
8. **多线程**：连接绑 ThreadLocal，子线程拿不到外层连接（跨线程"事务"是各自独立的）；
9. **事务管理器没配对/多数据源没指定 `transactionManager` ref**；
10. **引擎不支持**：MySQL MyISAM 无事务；
11. 注解加在**接口**上 + CGLIB 代理：接口注解不被继承（JDK 代理可读到，Boot 默认 CGLIB 则失效）——注解放实现类或父接口声明+实现类标注都算规范做法的争议根源，直接标实现类方法最稳。

### 4.3.2 自调用导致事务/异步失效，有哪些修复方式？

1. **拆类**：把被调方法挪到另一个 Bean（治本，依赖关系也清晰）；
2. **注入自身代理**：`@Lazy @Autowired private MyService self;` 然后 `self.methodB()`（@Lazy 防"自引用未初始化"）；
3. **`AopContext.currentProxy()`**：需 `@EnableAspectJAutoProxy(exposeProxy = true)`，强转调用（对全部 AOP 生效，略丑）；
4. **AspectJ 编译期/加载期织入**：彻底不依赖代理，`@Transactional` 加在 private 也生效——重武器，一般不上；
5. **编程式事务**（4.1.2）：自调段手写 tx 边界。

同族问题：**@Async 自调用同样失效**；@Cacheable 自调用同样失效——修复手段完全复用。

# 5. 事件与配置增强

## 5.1 事件机制

### 5.1.1 Spring 事件机制如何工作？同步/异步怎么用？

**观察者模式三件套**：`ApplicationEvent`（事件，JDK8 起可直接发布任意 POJO）→ `ApplicationEventPublisher.publishEvent()`（ApplicationContext 实现）→ `ApplicationListener<E>` 或 `@EventListener` 方法（可配 `@Order`）。

执行细节：

1. **默认同步**：监听器在**发布者线程**顺序执行——慢监听器拖住主流程，且异常会传播回发布者（回滚发布者事务）；
2. **异步**：监听方法加 `@Async` + 配置 `@EnableAsync`（提供 executor）——线程池隔离，异常不再回传；
3. 泛型事件解析、`@EventListener(condition = "#event.amount > 1000")` SpEL 过滤；
4. `ApplicationReadyEvent` 等容器生命周期事件常被用来做启动后动作。

**边界**：事件是**单进程内**解耦工具，不持久化、不跨 JVM——跨服务解耦请用 MQ（见 13_rocketmq.md、31_kafka.md）；进程内异步要接受"最终一致 + 失败补偿"的语义（本地事件表方案见 34_distributed_theory.md）。

### 5.1.2 @TransactionalEventListener 解决什么问题？

痛点：订单事务**还没提交**就发布事件，监听器（发短信/推库存）读到"未提交数据"或事务回滚了通知已发出。

`@TransactionalEventListener(phase = AFTER_COMMIT)` 把监听器注册为**事务同步回调**，绑定到 `TransactionSynchronization`，在指定阶段执行：AFTER_COMMIT（默认）/ AFTER_ROLLBACK / AFTER_COMPLETION / BEFORE_COMMIT。

```java
@Transactional
public void confirm(Order o) {
    repo.save(o);
    publisher.publishEvent(new OrderConfirmedEvent(o)); // 只是登记，此刻不发
}

@TransactionalEventListener(phase = AFTER_COMMIT)
@Async                                    // 常配异步：提交后旁路通知
public void onConfirmed(OrderConfirmedEvent e) { push(e); }
```

**坑**：AFTER_COMMIT 里再写库——**默认仍在原事务连接**（fallbackExecution=false 时直接不执行），需要新事务加 `REQUIRES_NEW`；监听器异常不会回滚已提交的主事务。

## 5.2 配置与占位符

### 5.2.1 @Configuration 的 CGLIB 增强做了什么？proxyBeanMethods 怎么理解？

Full 模式（@Configuration 默认）为配置类生成 **CGLIB 子类代理**，拦截 @Bean 方法间调用：

```java
@Configuration
public class AppConfig {
    @Bean public A a() { return new A(); }
    @Bean public B b() { return new B(a()); }   // a() 被拦截 → 返回容器单例，不是 new A()
}
```

保证"@Bean 方法互调不产生新实例、容器内单例语义不破"。

`@Configuration(proxyBeanMethods = false)`（Lite 思路）：跳过 CGLIB——@Bean 方法**不接受互调**（要用参数注入 `b(A a)`），换来**启动更快**；Boot 自动配置类清一色这么写，也是官方推荐新写法。

`@Bean` 与 @Component 都能标注；@Component + @Bean = Lite 模式（无代理，互调会 new 新对象）。

### 5.2.2 ${} 占位符的解析原理？属性来源有哪些？

**PropertySourcesPlaceholderConfigurer**（BeanFactoryPostProcessor）在实例化之前，把所有 BeanDefinition 属性值里的 `${key}` 用 `PropertyResolver` 链替换。

Environment 的 PropertySource 优先级（高→低，后者兜底）：命令行参数 → `SPRING_APPLICATION_JSON` → ServletConfig/Context 参数 → **JVM 系统属性** → 环境变量 → `application-{profile}.yml` → `application.yml` → defaultProperties。

配套细节：`@Value("${server.port:8080}")` 注入时解析（走 EmbeddedValueResolver/StringValueResolver，支持 SpEL `#{...}`）；`@PropertySource` 手动挂文件；**bootstrap 阶段**的占位符（Boot）由 ConfigDataEnvironment 处理——同名 key "高优先级覆盖" 是配置漂移排查第一检查项（见 10_springboot.md）。
