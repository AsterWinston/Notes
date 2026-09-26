# 1. 知识点
- **IoC（控制反转）**：将对象创建和依赖管理的控制权从程序代码转移给容器，通过 DI 实现对象间的解耦，降低组件耦合度，提升可测试性和可维护性。
- **DI（依赖注入）**：IoC 的具体实现方式，容器在运行时动态将依赖对象注入到目标对象中，开发者只需声明依赖关系。
- **构造器注入**：通过构造方法传入依赖，Spring 官方推荐方式，可保证依赖不可变（final），强制对象在创建时具备完整依赖。
- **Setter 注入**：通过 setter 方法注入可选依赖，适合非必需依赖项，对象可在无依赖的情况下先实例化。
- **字段注入**：通过 `@Autowired` 直接注入私有字段，代码简洁但不利于单元测试（需反射注入 mock），且隐藏依赖关系，不推荐使用。
- **@Autowired**：Spring 提供的自动装配注解，默认按类型（byType）匹配，可配合 `@Qualifier` 按名称限定。
- **@Resource**：JSR-250 标准注解，默认按名称（byName）匹配，无法找到名称时回退到按类型匹配。
- **@Primary**：当存在多个同类型 Bean 时，将标注的 Bean 设为优先注入的候选。
- **@Qualifier**：指定注入 Bean 的名称，解决 `@Autowired` 按类型匹配时的歧义问题。
- **Bean 生命周期**：实例化 → 属性注入 → Aware 接口回调 → BeanPostProcessor 前置处理 → InitializingBean → init-method → BeanPostProcessor 后置处理 → 使用 → DisposableBean → destroy-method。
- **BeanFactoryPostProcessor**：在 Bean 实例化之前修改 BeanDefinition（如 `${}` 占位符替换），`PropertySourcesPlaceholderConfigurer` 是其典型实现。
- **BeanPostProcessor**：在 Bean 初始化前后进行拦截处理，`AutowiredAnnotationBeanPostProcessor` 处理 `@Autowired` 注入是其典型实现。
- **Aware 接口**：让 Bean 获取 Spring 容器底层资源，如 `BeanFactoryAware`、`ApplicationContextAware`、`BeanNameAware`。
- **BeanFactory**：Spring 最基础的 IoC 容器，采用懒加载策略，首次获取 Bean 时才初始化，适用于资源受限场景。
- **ApplicationContext**：BeanFactory 的子接口，启动时预初始化所有单例 Bean（饿汉式），支持国际化、事件发布、资源加载等企业级功能。
- **Bean Scope（作用域）**：singleton（默认单例）、prototype（每次获取新建）、request（每个 HTTP 请求一个）、session（每个会话一个）、application（ServletContext 级别全局）、websocket（每个 WebSocket 连接一个）。
- **单例中注入多例失效问题**：单例 Bean 在构造时只注入一次 prototype 依赖，之后不再重新获取。解决方案：使用 `@Lookup` 方法注入、`ObjectFactory` 或 `Provider`。
- **循环依赖**：A 依赖 B、B 依赖 A 相互引用形成闭环。Spring 通过三级缓存机制解决单例 Setter 注入的循环依赖。
- **三级缓存**：`singletonObjects`（一级，完全初始化好的单例）、`earlySingletonObjects`（二级，提前暴露的半成品对象引用）、`singletonFactories`（三级，可生成早期引用的 ObjectFactory）。
- **循环依赖解决限制**：不能解决构造器注入的循环依赖（因为构造时必须传入完整对象，无法提前暴露）；不能解决 prototype 作用域的循环依赖。
- **AOP（面向切面编程）**：将横切关注点（日志、事务、安全）从核心业务逻辑中分离，通过切面模块化封装，提升代码复用性。
- **切面（Aspect）**：封装横切逻辑的模块，由切点和通知组成。
- **连接点（JoinPoint）**：程序执行过程中的一个点，Spring AOP 中只支持方法级别的连接点。
- **切点（Pointcut）**：匹配连接点的表达式，定义了通知在哪些方法上执行。
- **通知（Advice）**：Before（前置）、After（后置/最终）、AfterReturning（返回后）、AfterThrowing（异常后）、Around（环绕，最强大，可控制方法执行）。
- **织入（Weaving）**：将切面应用到目标对象创建代理的过程。编译时织入（AspectJ 编译器）、类加载时织入（LTW）、运行时织入（Spring AOP 默认，动态代理）。
- **代理选择规则**：目标类实现接口 → JDK 动态代理（基于接口）；没有实现接口 → CGLIB 代理（基于继承子类）。Spring Boot 中 `proxy-target-class=true` 默认使用 CGLIB。
- **`@EnableAspectJAutoProxy`**：启用 AspectJ 注解驱动的 AOP 代理，可设置 `proxyTargetClass=true` 强制使用 CGLIB。
- **事务传播行为（7 种）**：REQUIRED（默认，有则加入无则新建）、REQUIRES_NEW（挂起当前事务，新建独立事务）、SUPPORTS（有则用无则非事务）、NOT_SUPPORTED（挂起当前事务，非事务运行）、MANDATORY（必须有事务否则抛异常）、NEVER（有事务则抛异常）、NESTED（嵌套事务，可设置 Savepoint 部分回滚）。
- **事务隔离级别**：DEFAULT（使用数据库默认）、READ_UNCOMMITTED（读未提交，脏读）、READ_COMMITTED（读已提交，避免脏读）、REPEATABLE_READ（可重复读，避免不可重复读）、SERIALIZABLE（串行化，避免幻读）。
- **`@Transactional` 失效场景**：非 public 方法上使用、自调用（this 调用不走代理）、注解加在接口/抽象类（CGLIB 代理无效）、异常被 catch 吞掉、rollbackFor 不匹配（默认只回滚 RuntimeException 和 Error）、未配置事务管理器、多线程异步调用、数据库引擎不支持事务。
- **编程式事务**：通过 `TransactionTemplate` 或 `PlatformTransactionManager` 手动控制事务提交和回滚，灵活度更高但代码侵入性强。
- **事务超时与只读**：`timeout` 设置事务最大执行时间（秒），超时自动回滚；`readOnly=true` 指定只读事务，数据库可做性能优化。
- **Spring 事件机制**：基于观察者模式，`ApplicationEvent`（事件）→ `ApplicationListener`（监听器）→ `ApplicationEventPublisher`（发布者），支持 `@EventListener` 注解和 `@Async` 异步事件处理。
- **`@Component` vs `@Bean`**：`@Component` 标注在类上由 Spring 扫描自动注册；`@Bean` 标注在方法上手动声明 Bean 实例，通常用于第三方类或需要复杂初始化逻辑的 Bean。
- **BeanFactory vs FactoryBean**：BeanFactory是基础IOC容器接口，负责管理和创建Bean，支持延迟初始化；FactoryBean是特殊接口，允许自定义Bean的创建逻辑，实现该接口可返回不同类型的对象并控制Bean生命周期。BeanFactory以`&`开头获取FactoryBean本身
- **PO/VO/DAO/BO/DTO/POJO区别**：PO(Persistent Object)与DB表一一对应；VO(View Object)用于前端展示，可能组合多个PO；DAO(Data Access Object)封装数据库访问逻辑；BO(Business Object)封装业务逻辑；DTO(Data Transfer Object)用于服务间/远程调用数据传输；POJO(Plain Old Java Object)无特殊限制的普通Java对象

# 2. 面试题

## 2.1 什么是 IoC？它解决了什么问题？
IoC（Inversion of Control，控制反转）将对象创建和依赖管理的控制权从应用程序转移到外部容器（Spring 容器）。在传统编程中，程序通过 `new` 关键字主动创建依赖对象，属于正向控制；IoC 则让容器负责创建和装配对象。它解决的问题：
- **解耦**：对象之间不再直接耦合具体实现，依赖接口而非实现。
- **易于测试**：依赖可被 mock 替换，方便单元测试。
- **便于维护扩展**：修改依赖只需调整配置或注解，无需改动源码。
- **统一管理对象生命周期**：容器负责 Bean 的创建、初始化、销毁。

## 2.2 构造器注入、Setter 注入和字段注入的区别？为什么推荐构造器注入？
| 维度 | 构造器注入 | Setter 注入 | 字段注入 |
|------|-----------|-------------|----------|
| 注入时机 | 对象实例化时 | 实例化后调用 setter | 实例化后通过反射 |
| 不可变性 | 可声明 final | 不能 final | 不能 final |
| 依赖明确性 | 显式声明所有必需依赖 | 可隐藏依赖 | 完全隐藏依赖 |
| 循环依赖 | 无法解决 | 可解决 | 可解决 |
| 单元测试 | 直接 new 传参 | 调用 setter | 需要反射或启动容器 |

推荐构造器注入的原因：保证依赖不可变（final）；确保对象在构造完成后即处于可用状态；依赖关系明确可见，不会遗漏；测试友好，无需依赖 Spring 容器。

## 2.3 Spring Bean 的完整生命周期是怎样的？
1. **实例化**：容器通过反射调用构造方法创建 Bean 实例。
2. **属性填充**：根据 BeanDefinition 注入属性值（包括其他 Bean 的引用）。
3. **Aware 接口回调**：如果 Bean 实现了 `BeanNameAware`、`BeanFactoryAware`、`ApplicationContextAware` 等接口，依次调用对应方法。
4. **BeanPostProcessor 前置处理**：遍历所有 BeanPostProcessor，执行 `postProcessBeforeInitialization()`。
5. **InitializingBean 回调**：如果 Bean 实现了 `InitializingBean` 接口，调用 `afterPropertiesSet()`。
6. **init-method**：执行 `@PostConstruct` 注解的方法或 XML 配置的 `init-method`。
7. **BeanPostProcessor 后置处理**：执行 `postProcessAfterInitialization()`，AOP 代理在此阶段生成。
8. **Bean 就绪**：放入 `singletonObjects` 一级缓存，可供使用。
9. **DisposableBean 回调**：容器关闭时，如果实现了 `DisposableBean`，调用 `destroy()`。
10. **destroy-method**：执行 `@PreDestroy` 注解的方法或 XML 配置的 `destroy-method`。

## 2.4 BeanFactory 和 ApplicationContext 有什么区别？
| 维度 | BeanFactory | ApplicationContext |
|------|-------------|-------------------|
| 层级关系 | 顶层接口 | 继承自 BeanFactory |
| 实例化时机 | 懒加载，首次 getBean 时才创建 | 预初始化，启动时创建所有单例 Bean |
| 企业级功能 | 仅基础 IoC/DI | 国际化 MessageSource、事件发布 Event、资源加载 ResourceLoader |
| 自动注册 | 需手动注册 BeanPostProcessor | 自动识别并注册容器内的 BeanPostProcessor |
| 适用场景 | 资源受限（如移动设备、Applet） | 企业应用开发（绝大多数场景） |

实际开发中几乎全部使用 ApplicationContext，其饿汉式初始化能在启动时暴露配置错误，避免运行时才发现问题。

## 2.5 @Autowired 的注入流程是怎样的？
1. Spring 容器启动时注册 `AutowiredAnnotationBeanPostProcessor`，这是一个 BeanPostProcessor。
2. Bean 在属性填充阶段，`AutowiredAnnotationBeanPostProcessor` 扫描带 `@Autowired` 注解的字段或方法。
3. 通过 `DefaultListableBeanFactory` 查找匹配类型的候选 Bean。
4. 如果只有一个候选，直接注入；如果有多个且其中一个标注了 `@Primary`，则注入该 Bean。
5. 如果有多个且没有 `@Primary`，则尝试按 `@Qualifier` 指定名称匹配。
6. 如果仍未确定，将字段名/方法参数名作为 Bean 名称进行匹配。
7. 找不到或存在多个无法确定的 Bean 时抛出 `NoUniqueBeanDefinitionException`。
8. 可通过设置 `required=false` 使注入变为可选。

## 2.6 Spring 如何解决循环依赖？有什么限制？
Spring 通过**三级缓存**解决单例 Bean 的 Setter 注入循环依赖：

- **一级缓存 `singletonObjects`**：存放完全初始化完成的单例 Bean。
- **二级缓存 `earlySingletonObjects`**：存放提前暴露的半成品 Bean（已完成实例化但未完成属性填充）。
- **三级缓存 `singletonFactories`**：存放能生成 Bean 早期引用的 ObjectFactory。

**流程**（以 A↔B 为例）：
1. 创建 A，将 A 的 ObjectFactory 放入三级缓存，开始填充 A 的属性，发现依赖 B。
2. 创建 B，将 B 的 ObjectFactory 放入三级缓存，填充 B 的属性时发现依赖 A。
3. 从三级缓存取出 A 的 ObjectFactory 生成 A 的早期引用，放入二级缓存，注入给 B。
4. B 完成初始化，放入一级缓存，注入给 A。
5. A 完成初始化，放入一级缓存。

**限制**：无法解决构造器注入的循环依赖（构造时对象未创建完毕，无法提前暴露）；无法解决 prototype 作用域的循环依赖（不缓存）。

## 2.7 Spring AOP 的代理机制是怎样的？JDK 动态代理和 CGLIB 有何区别？
| 维度 | JDK 动态代理 | CGLIB 代理 |
|------|-------------|------------|
| 实现方式 | 基于接口，通过反射生成实现相同接口的代理类 | 基于继承，通过 ASM 字节码框架生成目标类的子类 |
| 代理对象 | 实现了目标接口的 `Proxy` 类 | 目标类的一个子类 |
| 性能 | 创建开销小，执行时反射调用稍慢 | 创建开销大（生成字节码），执行时直接方法调用快 |
| 限制 | 目标类必须实现至少一个接口 | 不能代理 final 类和方法，不能代理 private 方法 |
| Spring 选择策略 | 目标类实现接口且未设置 proxy-target-class=true | 目标类无接口或 proxy-target-class=true |

Spring Boot 2.x 起 `spring.aop.proxy-target-class=true` 默认使用 CGLIB。

## 2.8 事务传播行为有哪些？分别适用什么场景？
| 传播行为 | 含义 | 适用场景 |
|---------|------|---------|
| REQUIRED | 当前有事务则加入，无则新建 | 大多数业务场景，确保方法在事务中执行 |
| REQUIRES_NEW | 挂起当前事务，新建独立事务执行 | 日志记录、发消息等不应受主事务影响的操作 |
| SUPPORTS | 有事务则用，无则非事务执行 | 查询操作，不强制需要事务 |
| NOT_SUPPORTED | 挂起事务，非事务执行 | 不需要事务的耗时操作，避免占用事务资源 |
| MANDATORY | 必须在事务中，否则抛异常 | 严格要求在已有事务中执行的操作 |
| NEVER | 有事务则抛异常 | 严格要求非事务执行的场景 |
| NESTED | 嵌套事务，通过 Savepoint 可部分回滚 | 需要部分回滚的复杂业务（批量操作中部分失败不回滚全部） |

## 2.9 @Transactional 在哪些情况下会失效？
1. **非 public 方法**：Spring AOP 代理只能拦截 public 方法，private/protected/default 修饰符下事务不生效。
2. **自调用（this 调用）**：在类内部通过 `this.method()` 调用不走代理，事务切面不拦截。需通过 `AopContext.currentProxy()` 或注入自身代理对象调用。
3. **`@Transactional` 加在接口上**：使用 CGLIB 代理时（`proxy-target-class=true`），接口上的事务注解不被继承。
4. **异常被 catch 吞掉**：方法内捕获异常但未重新抛出，Spring 感知不到异常不会回滚。
5. **rollbackFor 不匹配**：默认只回滚 `RuntimeException` 和 `Error`，checked exception 不回滚。需显式配置 `@Transactional(rollbackFor = Exception.class)`。
6. **数据库引擎不支持事务**：如 MyISAM 引擎不支持事务，注解无效。
7. **未配置事务管理器**：没有 `DataSourceTransactionManager` 或 `JpaTransactionManager` 等。
8. **多线程调用**：事务通过 ThreadLocal 绑定到当前线程，新线程无法获取原事务。
9. **Spring MVC 父子容器问题**：如果 Service 被 DispatcherServlet 子容器扫描，事务管理器配置在根容器，可能导致事务代理失效。

## 2.10 Spring 事件机制是如何工作的？如何使用异步事件？
Spring 事件机制基于**观察者模式**，核心组件：
- **ApplicationEvent**：事件对象，继承自 `java.util.EventObject`。
- **ApplicationEventPublisher**：事件发布器，`ApplicationContext` 实现了此接口，调用 `publishEvent()` 发布事件。
- **ApplicationListener**：事件监听器，实现接口或使用 `@EventListener` 注解。

**使用方式**：
```java
// 定义事件
public class OrderEvent extends ApplicationEvent { ... }

// 发布事件
@Autowired ApplicationEventPublisher publisher;
publisher.publishEvent(new OrderEvent(this, order));

// 监听事件
@EventListener
public void handleOrderEvent(OrderEvent event) { ... }
```

**异步事件**：默认事件是同步的，会阻塞发布者线程。开启异步只需在监听方法上加 `@Async` 注解，并在配置类上标注 `@EnableAsync`。注意异步事件的事务独立于发布者的事务。

## 2.11 什么是 Spring 的三级缓存？它如何解决循环依赖？
Spring 的三级缓存是解决单例 Bean 循环依赖的核心机制：

```
singletonObjects          (一级) —> 完全初始化的 Bean
earlySingletonObjects     (二级) —> 提前暴露的半成品 Bean
singletonFactories        (三级) —> ObjectFactory，可生成 Bean 的代理/引用
```

**解决流程**（A↔B）：
1. `createBeanInstance(A)` → A 实例化完成
2. `addSingletonFactory(A)` → A 的 ObjectFactory 放入三级缓存
3. `populateBean(A)` → 属性填充时发现依赖 B
4. `getBean(B)` → B 不存在，开始创建 B
5. `createBeanInstance(B)` → B 实例化完成
6. `addSingletonFactory(B)` → B 的 ObjectFactory 放入三级缓存
7. `populateBean(B)` → 发现依赖 A，`getBean(A)` 从三级缓存获取 A 的早期引用
8. A 的早期引用从三级缓存移到二级缓存，注入到 B
9. B 完成初始化 → 存入一级缓存，删除二三级缓存
10. B 注入到 A，A 完成初始化 → 存入一级缓存

**三级缓存不只是解决循环依赖**：第三级缓存的 ObjectFactory 还能为 AOP 代理提供服务，确保注入的是代理对象而非原始对象。

## 2.12 @Resource 和 @Autowired 的区别？
| 维度 | @Autowired | @Resource |
|------|-----------|-----------|
| 来源 | Spring 框架 | JSR-250（JDK 标准） |
| 默认装配方式 | 按类型（byType） | 按名称（byName） |
| 名称匹配 | 需要配合 @Qualifier | 可通过 name 属性指定 |
| 必需性控制 | required 属性 | 无此属性 |
| 支持构造器 | 支持（构造器注入） | 不支持 |
| 支持字段/Setter | 支持 | 支持 |

建议使用 `@Autowired` + `@Qualifier` 组合，语义更清晰。

## 2.13 prototype 作用域的实际使用场景？有什么注意事项？
**场景**：
- 有状态的 Bean（如用户购物车、请求上下文对象）。
- 每次执行需要新建对象的场景（如批量任务中的处理器实例）。

**注意事项**：
- Spring 不管理 prototype Bean 的完整生命周期，只负责创建和初始注入，销毁方法需要调用方手动处理。
- **单例 Bean 中注入 prototype 会失效**：单例 Bean 只创建一次，其依赖的 prototype 在构造时只注入一次，之后不会重新获取。解决方式：`@Lookup` 方法注入或注入 `ObjectFactory`/`Provider`。
- prototype Bean 不参与循环依赖的解决。

## 2.14 Spring 中如何自定义事件监听？
1. **自定义事件**：继承 `ApplicationEvent`。
2. **发布事件**：注入 `ApplicationEventPublisher`，调用 `publishEvent()`。
3. **监听事件**：
   - 实现 `ApplicationListener<CustomEvent>` 接口。
   - 或使用 `@EventListener` 注解在方法上。
4. **异步监听**：在 `@EventListener` 方法上加 `@Async`（需 `@EnableAsync` 开启）。
5. **事务绑定**：使用 `@TransactionalEventListener` 在事务的特定阶段（如 `AFTER_COMMIT`）处理事件。

## 2.15 Bean 实例化过程中 Aware 接口的作用是什么？
Aware 接口让 Bean 在初始化过程中获取 Spring 容器底层组件的引用，属于**回调机制**：
- `BeanNameAware`：获取 Bean 在容器中的名称。
- `BeanFactoryAware`：获取 BeanFactory 容器实例。
- `ApplicationContextAware`：获取 ApplicationContext 实例。
- `ApplicationEventPublisherAware`：获取事件发布器。
- `ResourceLoaderAware`：获取资源加载器。
- `EnvironmentAware`：获取 Environment 环境配置。

这些接口在 Bean 实例化后、属性注入后、初始化前被调用，使 Bean 获得与容器交互的能力。

## 2.16 Spring 的 AOP 通知类型有哪些？Around 通知和其他通知的区别？
| 通知类型 | 注解 | 执行时机 |
|---------|------|---------|
| Before | @Before | 目标方法执行前 |
| After | @After | 目标方法执行后（无论是否异常，类似 finally） |
| AfterReturning | @AfterReturning | 目标方法正常返回后 |
| AfterThrowing | @AfterThrowing | 目标方法抛出异常后 |
| Around | @Around | 包裹目标方法，可控制执行前后逻辑 |

Around 通知是最强大的，因为它可以：
- 决定是否执行目标方法（`proceedingJoinPoint.proceed()`）。
- 修改目标方法的参数。
- 修改目标方法的返回值。
- 在方法执行前后加入自定义逻辑。

但也要谨慎使用，忘记调用 `proceed()` 会导致目标方法无法执行。

## 2.17 Spring 事务隔离级别各有什么含义？
| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 说明 |
|---------|------|----------|------|------|
| READ_UNCOMMITTED | 是 | 是 | 是 | 可读到未提交的数据，最低级别 |
| READ_COMMITTED | 否 | 是 | 是 | 只读已提交数据（Oracle 默认） |
| REPEATABLE_READ | 否 | 否 | 是 | 同一事务中多次读取结果一致（MySQL InnoDB 默认） |
| SERIALIZABLE | 否 | 否 | 否 | 完全串行化执行，最高级别 |

大多数应用使用 READ_COMMITTED 或 REPEATABLE_READ，在并发性能和数据一致性之间取得平衡。

## 2.18 @ComponentScan 和 @ComponentScans 的作用？
`@ComponentScan` 用于指定 Spring 扫描 Bean 的包路径。Spring 会扫描指定包及其子包下带 `@Component`、`@Service`、`@Repository`、`@Controller` 等注解的类并注册为 Bean。

常用属性：
- `basePackages`：指定扫描的包名数组。
- `basePackageClasses`：以类所在包为基准扫描。
- `includeFilters`：包含过滤，指定需要扫描的类型。
- `excludeFilters`：排除过滤，排除不需要扫描的类型。

```java
@ComponentScan(basePackages = "com.example", 
               excludeFilters = @ComponentScan.Filter(type = FilterType.ANNOTATION, 
                                                       classes = Controller.class))
```

## 2.19 什么是 Spring 的 Aware 回调？有哪些常用的 Aware 接口？
Aware 是 Spring 提供的一种回调机制，让 Bean 感知并获取到 Spring 容器的内部组件。Spring 在 Bean 初始化过程中检测 Bean 是否实现了特定的 Aware 接口，如果实现了就调用对应的 setter 方法注入相应组件。

常用 Aware 接口：
- `BeanNameAware` → 获取当前 Bean 的 id/name
- `BeanFactoryAware` → 获取 BeanFactory 引用
- `ApplicationContextAware` → 获取 ApplicationContext
- `MessageSourceAware` → 获取国际化消息源
- `ApplicationEventPublisherAware` → 获取事件发布器
- `ResourceLoaderAware` → 获取资源加载器
- `EnvironmentAware` → 获取环境变量配置
- `EmbeddedValueResolverAware` → 获取 StringValueResolver 解析占位符

## 2.20 Spring 中 @Primary 和 @Qualifier 如何协同工作？
当存在多个同类型的 Bean 时：
- `@Primary` 声明在类/方法上，标记该 Bean 为**默认首选项**，当不指定名称时优先注入。
- `@Qualifier` 声明在注入点上，**显式指定**要注入的 Bean 名称，优先级高于 `@Primary`。

两者的配合策略：
1. Spring 先查找类型匹配的候选 Bean。
2. 如果只有一个，直接注入。
3. 如果有多个，先检查是否存在 `@Primary` 标注的 Bean（有则注入）。
4. 如果注入点有 `@Qualifier`，直接按名称匹配（忽略 `@Primary`）。
5. 如果找不到且 `required=false`，跳过注入；否则抛异常。

简单记忆：`@Qualifier` 优先级 > `@Primary`，因为它更精确。

## 2.21 BeanFactory和FactoryBean的区别是什么？
| 维度 | BeanFactory | FactoryBean |
|------|-------------|-------------|
| 本质 | Spring IoC 容器的基础接口 | 一个特殊的 Bean 创建工厂接口 |
| 职责 | 管理和创建所有 Bean 实例 | 自定义特定 Bean 的创建逻辑 |
| 返回对象 | 返回 Bean 本身 | 返回任意类型对象（由 getObject() 决定） |
| 获取方式 | 直接 getBean("beanName") | getBean("&beanName") 获取 FactoryBean 本身，getBean("beanName") 获取其创建的对象 |
| 生命周期 | 不参与 | 可控制 Bean 的创建、初始化、销毁 |

使用场景：MyBatis 的 SqlSessionFactoryBean、Spring 的 ProxyFactoryBean 都是 FactoryBean 的典型应用，用于创建复杂对象并托管给容器。

## 2.22 PO、VO、DAO、BO、DTO、POJO 有什么区别和使用场景？
| 术语 | 全称 | 含义 | 使用场景 |
|------|------|------|---------|
| PO | Persistent Object | 持久化对象，与数据库表字段一一对应 | ORM 映射，如 @Entity 标注的实体类 |
| VO | View Object | 视图对象，面向前端展示的数据结构 | 接口返回给前端、页面渲染 |
| DAO | Data Access Object | 数据访问对象，封装对数据库的 CRUD 操作 | 数据访问层，如 Mapper/Repository |
| BO | Business Object | 业务对象，封装核心业务逻辑 | Service 层，组合多个 PO 处理业务规则 |
| DTO | Data Transfer Object | 数据传输对象，用于跨服务或跨层传输数据 | RPC 调用、Controller 接收参数、远程接口返回值 |
| POJO | Plain Old Java Object | 普通 Java 对象，无特殊限制和依赖 | 最简单的 Java 对象，不继承任何框架类 |

核心区分：PO 直连数据库，VO 面向页面，DTO 面向传输，BO 面向业务逻辑，DAO 面向数据访问，POJO 是一个统称，所有上述对象本质上都是 POJO。

## 2.23 Spring 事务的底层实现原理是什么？（AOP + TransactionInterceptor）
Spring 事务基于 **AOP 代理 + TransactionInterceptor** 实现：

1. **开启事务**：通过 `@EnableTransactionManagement` 注解或 `<tx:annotation-driven/>` 注册一个 `BeanFactoryTransactionAttributeSourceAdvisor`，它是一个 Advisor，用于匹配被 `@Transactional` 标记的方法。

2. **代理创建**：Spring 容器在 Bean 初始化后（`BeanPostProcessor` 的 `postProcessAfterInitialization` 阶段）判断 Bean 是否被 `@Transactional` 标记，若是则为 Bean 创建代理对象。

3. **方法拦截**：当调用 `@Transactional` 方法时，进入 `TransactionInterceptor.invoke()`：
   - 调用 `TransactionAttributeSource.getTransactionAttribute()` 获取方法上的事务属性（传播行为、隔离级别、超时、只读、rollbackFor 等）。
   - 调用 `PlatformTransactionManager.getTransaction()` —— 创建新事务或加入已有事务（根据传播行为决定）。
   - 执行目标方法（`invocation.proceed()`）。
   - 成功则 `commitTransactionAfterReturning()`；失败则 `completeTransactionAfterThrowing()` —— 根据 rollbackFor 判断是否回滚 → `rollback()` 或 `commit()`。

4. **事务管理器**：`PlatformTransactionManager` 是最上层抽象，具体实现有 `DataSourceTransactionManager`（JDBC 单数据源）、`JtaTransactionManager`（分布式事务）。Spring Boot 中常用的是 `DataSourceTransactionManager`。

**AOP 代理的选择**：`@Transactional` 底层与 Spring AOP 一致——有接口默认 JDK 动态代理，无接口用 CGLIB（`proxy-target-class=true` 可强制 CGLIB）。这就是为什么"同类方法自调用时事务失效"——自调用不走代理。

## 2.24 @Configuration 和 @Component 的区别？CGLIB 增强做了什么？
| 维度 | @Configuration（Full 模式） | @Component（Lite 模式） |
|------|---------------------------|------------------------|
| CGLIB 代理 | 容器为其生成 CGLIB 子类代理 | 不使用 CGLIB 代理 |
| 内部 @Bean 方法调用 | 每次调用都返回容器中的**同一个单例 Bean** | 每次调用都是**普通 Java 方法调用**，返回新对象 |
| proxyBeanMethods | 默认 `true`（启用代理） | 不支持（默认为 false） |

**CGLIB 增强的核心作用**——保证 `@Bean` 方法在内部被调用时，始终返回容器中的单例 Bean，而不是创建新实例：

```java
@Configuration
public class AppConfig {
    @Bean
    public A a() { return new A(); }
    
    @Bean
    public B b() {
        // CGLIB 增强后：a() 从容器中获取已存在的 a Bean（而非调用 new A()）
        return new B(a());
    }
}
```

CGLIB 代理拦截 `a()` 的调用，判断容器中已存在 `a` Bean → 直接返回已有实例，确保 `B` 中注入的 `a` 和容器中的 `a` 是同一个对象。

**何时关闭代理（`@Configuration(proxyBeanMethods = false)`）**：
- 配置类中 `@Bean` 方法之间没有相互依赖（如大量独立的 `@Bean` 定义）
- 追求更快的启动速度（无需创建 CGLIB 代理类）
- Spring Boot 中大部分自动配置类都使用 `proxyBeanMethods = false`

@Component + @Bean 是 Lite 模式，无代理，内部调用 `a()` 会创建新对象，可能导致"同一个 Bean 被多次创建"的问题。

## 2.25 Spring 父子容器（ApplicationContext 层次）是什么？在 Spring MVC 中如何体现？
Spring 支持容器层次结构：一个 ApplicationContext 可以设置父容器（`setParent()`）。

**核心行为**：子容器可以访问父容器的 Bean，但父容器不能访问子容器的 Bean。子容器通过 `getParent().getBean()` 向父容器查 Bean（如果自身找不到的话）。

**在 Spring MVC 中的体现**（经典 Web 应用结构）：
- **父容器（Root WebApplicationContext）**：由 `ContextLoaderListener` 创建，通常包含 Service、Repository、DataSource 等**通用 Bean**。
- **子容器（Servlet WebApplicationContext）**：由 `DispatcherServlet` 创建，包含 Controller、HandlerMapping、ViewResolver 等 **Web 层 Bean**。

**父子容器带来的典型问题**：
1. **事务失效**：Controller 中注入的 Service 从父容器获得（有事务代理），但如果 Controller 在子容器而配置了父容器级别的 AOP，可能「重复扫描」导致 Bean 既在父容器又在子容器，事务代理被覆盖。
2. **Spring Boot 的做法**：Spring Boot 由于使用嵌入式容器，只有一个 ApplicationContext（无父子容器分层），避免了上述问题。

## 2.26 @PostConstruct、InitializingBean.afterPropertiesSet() 和 init-method 的执行顺序？
三者的执行顺序（同一个 Bean 内）：
1. **@PostConstruct**（JSR-250，最先执行）
2. **InitializingBean.afterPropertiesSet()**（Spring 专有接口）
3. **init-method**（XML 或 `@Bean(initMethod="xxx")` 指定，最后执行）

**实现原理**：Spring 在 `AbstractAutowireCapableBeanFactory.initializeBean()` 中依次调用：
- `BeanPostProcessor.postProcessBeforeInitialization()` 遍历 → 其中 `InitDestroyAnnotationBeanPostProcessor` 处理 `@PostConstruct` 注解
- `invokeInitMethods()`：
  - 先调用 `((InitializingBean) bean).afterPropertiesSet()`
  - 再调用自定义的 `init-method`

**选择建议**：推荐 `@PostConstruct`（JSR 标准，不依赖 Spring 接口，与框架解耦）。

## 2.27 @Transactional 的 rollbackFor 有什么用？哪些异常会回滚？
**默认回滚规则**：Spring 事务默认只对 **RuntimeException（运行时异常）和 Error** 进行回滚，对 **checked exception（受检异常）不回滚**。

**原因**：受检异常在 Java 异常模型中代表"可以恢复的异常"，Spring 认为不应该回滚事务（但其实际意义取决于业务语义）。大多数业务异常都应该回滚，所以生产环境中建议显式指定 `rollbackFor`。

**配置**：
```java
@Transactional(rollbackFor = Exception.class)      // 所有异常都回滚（推荐）
@Transactional(rollbackFor = RuntimeException.class) // 仅运行时异常回滚（与默认相同）
@Transactional(noRollbackFor = IllegalArgumentException.class) // IllegalArgumentException 不回滚
```

**最佳实践**：显式指定 `rollbackFor = Exception.class`。避免碰到受检异常不回滚导致数据不一致的问题，这是常见的生产事故原因之一。
