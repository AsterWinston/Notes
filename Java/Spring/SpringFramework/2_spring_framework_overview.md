# 1. Spring简介
Spring是一个开源框架
Spring是一个轻量级的控制反转(IoC)和面向切面(AOP)的容器框架。
Spring实现了控制反转思想，Spring框架可以帮你维护对象和对象之间的关系
Spring实现了面向切面编程
由于我们可以将创建对象交给Spring负责，所以Spring也是一个存放对象的容器
Spring为简化开发而生，让程序员只需关注核心业务的实现，尽可能的不再关注非业务逻辑代码（事务控制，安全日志等）。
Spring是一个实现了IoC思想的容器。
# 2. Spring 8大模块
Spring5版本之后是8个模块。在Spring5中新增了WebFlux模块。

Spring AOP：基于动态代理实现轻量级面向切面编程，将日志、事务等横切逻辑与业务代码解耦，支持注解 / XML 配置。
Spring Web MVC：基于 Spring Web 构建的同步阻塞式MVC 框架，通过 DispatcherServlet 实现请求分发，适配传统 Web 开发。
Spring Webflux：基于响应式编程模型的异步非阻塞式Web 框架，支持 Reactor/ RxJava，适配高并发非阻塞场景。
Spring Web：提供 Web 环境下的基础支撑（如 WebApplicationContext、请求上下文、文件上传），是所有 Web 模块的基础。
Spring DAO：统一数据访问层的异常体系，封装数据访问的通用逻辑，为 JDBC/ORM 等模块提供基础数据访问抽象。
Spring ORM：作为适配层，实现 Spring 与 Hibernate/MyBatis/JPA 等 ORM 框架的无缝集成，统一 ORM 事务与异常管理。
Spring Context：基于 Core 扩展，提供企业级 IoC 上下文能力（国际化、事件驱动、资源加载、环境适配等），是 IoC 容器的完整版。
Spring Core：定义 Spring IoC 容器的底层核心实现，提供依赖注入（DI）的基础能力，是所有模块的根依赖。

# 3. Spring的特点
轻量
完整的Spring框架可以在一个大小只有1MB多的JAR文件里发布，并且Spring所需的处理开销也是微不足道的。
Spring是非侵入式的：spring框架的运行不需要依赖其他框架
控制反转
Spring实现了控制反转思想
面向切面
Spring提供了面向切面编程的丰富支持
允许开发者只需专注于业务逻辑的开发，与核心业务无关的代码可以以切面的方式加入核心业务代码的执行过程中，将核心业务的执行流程看成是纵向执行的，与核心业务无关的代码（如事务、日志等）可以以横向切面的方式加入核心业务的执行过程中
容器
Spring 可以包含并管理应用对象的配置和生命周期，即Spring可以负责对象的创建到对象的销毁这整个生命周期过程中对象的维护和管理，Spring就好比一个存放和管理对象的容器
框架
Spring可以将简单的组件配置、组合成为复杂的应用。
# 4. Spring对JDK的要求
Spring6要求JDK最低版本是Java17

# 5. 源码视角：Spring 模块与核心类的对应关系

```
Spring 8 大模块与核心类的源码位置对应：

Spring Core（spring-core）
  ├── org.springframework.util.*
  │   └── ClassUtils, StringUtils, ReflectionUtils  ← 工具类基础
  ├── org.springframework.core.*
  │   └── ResolvableType, AnnotationUtils, Ordered    ← 类型和注解抽象
  └── org.springframework.cglib / org.springframework.objenesis
      └── 内嵌的 CGLIB/Objenesis（用于动态代理）

Spring Beans（spring-beans）
  ├── org.springframework.beans.factory.*
  │   ├── BeanFactory (接口)                          ← IoC 容器顶级接口
  │   ├── DefaultListableBeanFactory                  ← ★ 核心实现（存放 beanDefinitionMap）
  │   ├── AbstractAutowireCapableBeanFactory          ← createBean/doCreateBean
  │   └── BeanWrapper / BeanWrapperImpl              ← 属性注入的核心
  ├── org.springframework.beans.factory.config.*
  │   ├── BeanPostProcessor (接口)                    ← ★ Bean 后处理器
  │   ├── BeanFactoryPostProcessor (接口)            ← ★ BeanFactory 后处理器
  │   └── BeanDefinition                              ← ★ Bean 定义对象
  └── org.springframework.beans.factory.support.*
      └── DefaultSingletonBeanRegistry                ← 三级缓存 + 单例管理

Spring Context（spring-context）
  ├── org.springframework.context.*
  │   ├── ApplicationContext (接口)                    ← 应用上下文
  │   ├── AbstractApplicationContext                  ← refresh() 的 13 步
  │   ├── AnnotationConfigApplicationContext          ← 注解驱动容器
  │   └── ClassPathXmlApplicationContext              ← XML 驱动容器
  ├── org.springframework.context.annotation.*
  │   ├── ConfigurationClassPostProcessor             ← @Configuration 处理
  │   └── ConfigurationClassParser                    ← @ComponentScan/@Bean 解析
  └── org.springframework.context.support.*
      └── PropertySourcesPlaceholderConfigurer        ← ${...} 占位符处理

Spring AOP（spring-aop）
  ├── org.springframework.aop.framework.*
  │   ├── JdkDynamicAopProxy                          ← JDK 动态代理
  │   ├── CglibAopProxy (ObjenesisCglibAopProxy)      ← CGLIB 代理
  │   ├── ProxyFactory                                ← 代理工厂
  │   └── ReflectiveMethodInvocation                  ← ★ 拦截器链
  └── org.springframework.aop.aspectj.*
      └── AnnotationAwareAspectJAutoProxyCreator       ← ★ @Aspect 处理
      └── AspectJAroundAdvice / AspectJMethodBeforeAdvice 等

Spring JDBC（spring-jdbc）
  ├── org.springframework.jdbc.core.*
  │   └── JdbcTemplate                                ← JDBC 模板
  └── org.springframework.jdbc.datasource.*
      └── DataSourceUtils                             ← 事务感知连接管理
      └── DataSourceTransactionManager                ← ★ JDBC 事务管理器

Spring TX（spring-tx）
  └── org.springframework.transaction.*
      ├── TransactionInterceptor                      ← ★ @Transactional 拦截器
      ├── TransactionSynchronizationManager          ← ★ ThreadLocal 事务管理
      └── AbstractPlatformTransactionManager          ← 事务管理器模板

Spring Web MVC（spring-webmvc）
  ├── org.springframework.web.servlet.*
  │   └── DispatcherServlet                           ← ★ 前端控制器
  └── org.springframework.web.servlet.mvc.method.annotation.*
      └── RequestMappingHandlerAdapter                ← 请求映射处理

Spring WebFlux（spring-webflux）
  └── org.springframework.web.reactive.*
      └── DispatcherHandler                            ← 响应式前端控制器
```

# 6.设计思想总结

```
1. 分层架构：
   Core → Beans → Context → AOP → 上层模块（JDBC/TX/MVC）
   下层不依赖上层，每层专注于特定领域，体现了分层解耦原则。

2. 接口 + 实现分离：
   - BeanFactory 接口（API 规范）→ DefaultListableBeanFactory（默认实现）
   - ApplicationContext 接口 → ClassPathXmlApplicationContext / AnnotationConfigApplicationContext
   这种设计让用户可以在不修改核心代码的情况下，替换实现。

3. 扩展点体系：
   - BeanPostProcessor：干预 Bean 的创建过程
   - BeanFactoryPostProcessor：干预 BeanDefinition 的注册过程
   - ApplicationListener：监听容器事件
   这些都是典型的"对扩展开放，对修改闭合"（OCP）实践。

4. 命名模式：
   Spring 中的类名遵循一致的命名规范：
   - Abstract*：抽象实现（如 AbstractBeanFactory）
   - Default*：默认实现（如 DefaultListableBeanFactory）
   - Simple*：简单实现（如 SimpleInstantiationStrategy）
   - *Support：支持类，提供辅助功能
   - *Utils：工具类
```
