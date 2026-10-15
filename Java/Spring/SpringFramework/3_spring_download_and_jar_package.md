# 1. Spring下载
```
可以直接用Maven下载，根据坐标在Maven窗口中即可找到Spring源程序

Spring 官网：https://spring.io/
Spring Framework：https://spring.io/projects/spring-framework

```

# 2. Spring官方文档

# 3. Spring的jar文件
```
以核心程序的 jar 包为例：
spring-core-5.3.9.jar：字节码（这个是支撑程序运行的jar包）
spring-core-5.3.9-javadoc.jar：代码中的注释
spring-core-5.3.9-sources.jar：源码
```

# 4. Spring所有jar包及其描述汇总
JAR文件	描述
**	这个jar 文件包含在应用中使用Spring 的AOP 特性时所需的类
spring-aspects-5.3.9.jar	提供对AspectJ的支持，以便可以方便的将面向切面的功能集成进IDE中
spring-beans-5.3.9.jar	这个jar 文件是所有应用都要用到的，它包含访问配置文件、创建和管理bean 以及进行Inversion ofControl / Dependency Injection（IoC/DI）操作相关的所有类。如果应用只需基本的IoC/DI 支持，引入spring-core.jar 及spring-beans.jar 文件就可以了。
spring-context-5.3.9.jar	这个jar 文件为Spring 核心提供了大量扩展。可以找到使用Spring ApplicationContext特性时所需的全部类，JDNI 所需的全部类，instrumentation组件以及校验Validation 方面的相关类。
spring-context-indexer-5.3.9.jar	虽然类路径扫描非常快，但是Spring内部存在大量的类，添加此依赖，可以通过在编译时创建候选对象的静态列表来提高大型应用程序的启动性能。
spring-context-support-5.3.9.jar	用来提供Spring上下文的一些扩展模块,例如实现邮件服务、视图解析、缓存、定时任务调度等
spring-core-5.3.9.jar	Spring 框架基本的核心工具类。Spring 其它组件要都要使用到这个包里的类，是其它组件的基本核心，当然你也可以在自己的应用系统中使用这些工具类。
spring-expression-5.3.9.jar	Spring表达式语言。
spring-instrument-5.3.9.jar	Spring3.0对服务器的代理接口。
spring-jcl-5.3.9.jar	Spring的日志模块。JCL，全称为"Jakarta Commons Logging"，也可称为"Apache Commons Logging"。
spring-jdbc-5.3.9.jar	Spring对JDBC的支持。
spring-jms-5.3.9.jar	这个jar包提供了对JMS 1.0.2/1.1的支持类。JMS是Java消息服务。属于JavaEE规范之一。
spring-messaging-5.3.9.jar	为集成messaging api和消息协议提供支持
spring-orm-5.3.9.jar	Spring集成ORM框架的支持，比如集成hibernate，mybatis等。
spring-oxm-5.3.9.jar	为主流O/X Mapping组件提供了统一层抽象和封装，OXM是Object Xml Mapping。对象和XML之间的相互转换。
spring-r2dbc-5.3.9.jar	Reactive Relational Database Connectivity (关系型数据库的响应式连接) 的缩写。这个jar文件是Spring对r2dbc的支持。
spring-test-5.3.9.jar	对Junit等测试框架的简单封装。
spring-tx-5.3.9.jar	为JDBC、Hibernate、JDO、JPA、Beans等提供的一致的声明式和编程式事务管理支持。
spring-web-5.3.9.jar	Spring集成MVC框架的支持，比如集成Struts等。
spring-webflux-5.3.9.jar	WebFlux是 Spring5 添加的新模块，用于 web 的开发，功能和 SpringMVC 类似的，Webflux 使用当前一种比较流程响应式编程出现的框架。
spring-webmvc-5.3.9.jar	SpringMVC框架的类库
spring-websocket-5.3.9.jar	Spring集成WebSocket框架时使用

# 5. 源码视角：Spring 6 模块依赖图与包结构

## 5.1 Maven 依赖传递关系

```
spring-context (最常用的入门依赖)
  ├── spring-aop          ← AOP 支持（JDK/CGLIB 动态代理）
  ├── spring-beans        ← IoC 容器核心（BeanFactory、BeanDefinition）
  ├── spring-core         ← 基础工具类（ClassUtils、ReflectionUtils）
  │   └── spring-jcl      ← 日志桥接（JCL → SLF4J）
  ├── spring-expression   ← SpEL 表达式语言
  └── (可选) spring-web    ← Web 上下文基础

spring-jdbc
  ├── spring-beans
  ├── spring-core
  └── spring-tx           ← 事务抽象（PlatformTransactionManager）

spring-webmvc
  ├── spring-aop
  ├── spring-beans
  ├── spring-context
  ├── spring-core
  ├── spring-expression
  └── spring-web

spring-boot-starter-web 的传递依赖精简版：
  spring-boot-starter-web
    ├── spring-boot-starter
    │   ├── spring-boot
    │   ├── spring-boot-autoconfigure
    │   ├── spring-boot-starter-logging
    │   ├── jakarta.annotation-api
    │   ├── spring-core
    │   └── snakeyaml (YAML 解析)
    ├── spring-boot-starter-json (Jackson)
    ├── spring-boot-starter-tomcat (内嵌 Tomcat)
    ├── spring-webmvc
    └── spring-web
```

## 5.2 常见 jar 内部的包结构举例

```
spring-beans-6.0.x.jar
  ├── org/springframework/beans/factory/
  │   ├── BeanFactory.class                     ← IoC 容器顶级接口
  │   ├── config/
  │   │   ├── BeanPostProcessor.class            ← Bean 后处理器
  │   │   ├── BeanFactoryPostProcessor.class     ← BeanFactory 后处理器
  │   │   └── BeanDefinition.class               ← Bean 定义
  │   └── support/
  │       ├── DefaultListableBeanFactory.class    ← ★ 核心容器实现
  │       ├── AbstractAutowireCapableBeanFactory.class
  │       └── DefaultSingletonBeanRegistry.class  ← 三级缓存
  └── org/springframework/beans/
      ├── BeanWrapper.class                       ← 属性注入
      └── PropertyValues.class

spring-context-6.0.x.jar
  ├── org/springframework/context/
  │   ├── ApplicationContext.class
  │   ├── annotation/
  │   │   ├── ConfigurationClassPostProcessor.class  ← @Configuration 处理
  │   │   └── AutowiredAnnotationBeanPostProcessor.class
  │   └── support/
  │       └── ClassPathXmlApplicationContext.class
  └── org/springframework/stereotype/
      ├── Component.class
      ├── Service.class
      └── Repository.class

spring-aop-6.0.x.jar
  ├── org/springframework/aop/framework/
  │   ├── JdkDynamicAopProxy.class
  │   ├── CglibAopProxy.class
  │   ├── ProxyFactory.class
  │   └── ReflectiveMethodInvocation.class
  └── org/springframework/aop/aspectj/
      └── annotation/
          └── AnnotationAwareAspectJAutoProxyCreator.class

spring-tx-6.0.x.jar
  └── org/springframework/transaction/
      ├── TransactionInterceptor.class
      ├── support/
      │   └── TransactionSynchronizationManager.class
      └── annotation/
          └── Transactional.class

spring-jdbc-6.0.x.jar
  ├── org/springframework/jdbc/core/
  │   └── JdbcTemplate.class
  └── org/springframework/jdbc/datasource/
      ├── DataSourceUtils.class
      └── DataSourceTransactionManager.class
```
