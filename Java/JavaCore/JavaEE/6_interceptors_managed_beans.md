
# 1. 概述

本笔记覆盖三个紧密相关的 JavaEE 规范：

| 规范 | 作用 |
|------|------|
| **Jakarta Interceptors** | 面向方法调用/生命周期事件的声明式拦截 |
| **Jakarta Managed Beans** | 托管 Bean 生命周期回调规范 |
| **Jakarta Annotations（公共注解）** | 跨规范共享的通用注解（`@Resource`、`@PostConstruct`、`@PreDestroy` 等） |

# 2. Jakarta Interceptors

## 2.1 什么是拦截器

拦截器（Interceptor）在**目标方法调用前后**插入横切逻辑，是 AOP 思想的声明式实现，可应用于 EJB、CDI Bean 等。

```
调用方
  │
  ▼
拦截器链（多个 @AroundInvoke）
  │
  ▼
目标业务方法
  │
  ▼
返回 / 异常
```

## 2.2 与 Spring AOP 的对照

| Jakarta Interceptors | Spring AOP |
|----------------------|-----------|
| `@AroundInvoke` | `@Around` |
| `@AroundConstruct` | 构造织入 |
| `@PostConstruct` / `@PreDestroy` | `@PostConstruct` / `@PreDestroy` |
| `@AroundTimeout` | EJB 定时器拦截 |
| 绑定方式：`@InterceptorBinding` | 绑定方式：`@Around` 切面 + Pointcut |

## 2.3 定义拦截器

```java
@Interceptor
@Loggable
public class LoggingInterceptor {

    @AroundInvoke
    public Object log(InvocationContext ctx) throws Exception {
        System.out.println("before: " + ctx.getMethod());
        try {
            return ctx.proceed();
        } finally {
            System.out.println("after: " + ctx.getMethod());
        }
    }
}
```

## 2.4 拦截器绑定注解（Interceptor Binding）

拦截器通过**绑定注解**与被拦截的目标关联：

```java
@InterceptorBinding
@Target({TYPE, METHOD})
@Retention(RUNTIME)
public @interface Loggable { }
```

- 绑定注解是拦截器与业务类的"中间人"。
- 一个拦截器可以绑定多个绑定注解，一个业务类可以应用多个拦截器。

## 2.5 使用拦截器

```java
@Loggable
@Stateless
public class OrderEJB {
    public void createOrder(Order order) { /* ... */ }
}
```

- 应用在类上：拦截该类所有业务方法。
- 应用在方法上：只拦截该方法。

## 2.6 启用拦截器

**方式一：`beans.xml`（CDI 环境）**

```xml
<beans ...>
    <interceptors>
        <class>com.example.LoggingInterceptor</class>
    </interceptors>
</beans>
```

**方式二：`@Priority`（CDI 2.0+，自动启用）**

```java
@Interceptor
@Priority(Interceptor.Priority.APPLICATION)
@Loggable
public class LoggingInterceptor { }
```

**方式三：EJB 的 `ejb-jar.xml`**

```xml
<assembly-descriptor>
    <interceptor-binding>
        <ejb-name>OrderEJB</ejb-name>
        <interceptor-class>com.example.LoggingInterceptor</interceptor-class>
    </interceptor-binding>
</assembly-descriptor>
```

## 2.7 拦截器执行顺序

- `@Priority` 值越小，优先级越高，先执行。
- `beans.xml` 中声明的拦截器按声明顺序执行。
- 拦截器链遵循"洋葱模型"：进入依次调用，返回逆向。

## 2.8 其他拦截点

| 拦截点 | 方法 | 说明 |
|--------|------|------|
| 业务方法 | `@AroundInvoke` | 最常用，拦截方法调用 |
| 构造 | `@AroundConstruct` | 拦截 Bean 实例化（CDI 2.0+） |
| 定时器 | `@AroundTimeout` | 拦截 EJB 定时器执行 |
| 生命周期 | `@PostConstruct` / `@PreDestroy` | 见 3.2 |

# 3. Jakarta Managed Beans

## 3.1 什么是托管 Bean

托管 Bean（Managed Bean）是 JavaEE 中**由容器管理生命周期**的普通 POJO，规范定义统一的生命周期回调约定：

```
容器创建实例
    │
    ▼
@PostConstruct 方法调用（初始化）
    │
    ▼
业务使用
    │
    ▼
@PreDestroy 方法调用（清理）
    │
    ▼
容器销毁实例
```

## 3.2 生命周期回调注解

| 注解 | 触发时机 | 用途 |
|------|----------|------|
| `@PostConstruct` | 依赖注入完成后、Bean 可用之前 | 初始化资源、加载配置 |
| `@PreDestroy` | Bean 被销毁前 | 释放连接、关闭线程池 |

```java
public class ReportGenerator {

    private ExecutorService pool;

    @PostConstruct
    public void init() {
        pool = Executors.newFixedThreadPool(4);
        System.out.println("初始化线程池");
    }

    @PreDestroy
    public void cleanup() {
        pool.shutdown();
        System.out.println("关闭线程池");
    }
}
```

## 3.3 使用注意事项

- 一个类中 `@PostConstruct` / `@PreDestroy` 各只能有一个方法。
- 方法不能有参数，可以是任意访问级别，不能抛受检异常。
- 回调执行顺序：父类 → 子类；接口回调先于父类。
- 这两个注解在 EJB、CDI、Servlet、JSF 中通用。

## 3.4 `@PostConstruct` vs 构造器

| 对比 | 构造器 | `@PostConstruct` |
|------|--------|-----------------|
| 依赖注入状态 | 尚未注入完成 | 注入已完成 |
| 覆盖方式 | 无法被容器增强 | 容器可调用 |
| 用途 | 基础赋值 | 需要依赖注入后的初始化 |
| 可以抛异常 | 可以 | 只能抛运行时异常 |

# 4. Jakarta Annotations（公共注解）

## 4.1 常用注解清单

| 注解 | 适用组件 | 作用 |
|------|----------|------|
| `@Resource` | EJB/Servlet/JSF 等 | 资源注入（数据源、JMS 连接工厂、环境条目） |
| `@Resources` | 同上 | 多个 `@Resource` 的容器 |
| `@PostConstruct` | 所有托管组件 | 初始化回调 |
| `@PreDestroy` | 所有托管组件 | 销毁回调 |
| `@DeclareRoles` | Servlet/EJB | 声明安全角色 |
| `@RolesAllowed` | Servlet/EJB | 允许访问的角色（授权） |
| `@PermitAll` | Servlet/EJB | 允许所有角色访问 |
| `@DenyAll` | Servlet/EJB | 拒绝所有访问 |
| `@RunAs` | Servlet/EJB | 以指定角色运行 |
| `@PersistenceContext` | EJB/Servlet | 注入 EntityManager |
| `@PersistenceUnit` | EJB/Servlet | 注入 EntityManagerFactory |
| `@EJB` | 任意容器组件 | 注入 EJB |
| `@WebServiceRef` | 任意容器组件 | 注入 Web 服务客户端 |
| `@DataSource` | 任意容器组件 | 注入数据源（Jakarta EE 8+） |

## 4.2 `@Resource` 资源注入

```java
public class JndiClient {
    @Resource(lookup = "java:comp/env/jdbc/MyDS")
    private DataSource dataSource;

    @Resource
    private SessionContext ctx;   // EJB 上下文
}
```

- `@Resource` 注入到 JNDI 命名空间中的资源。
- 相比手动 `lookup`，注入更简洁、类型安全。

## 4.3 安全注解示例

```java
@Stateless
@DeclareRoles({"admin", "user"})
public class AdminEJB {

    @RolesAllowed("admin")
    public void deleteUser(long id) { /* 仅管理员可调用 */ }

    @PermitAll
    public User getUser(long id) { /* 所有已认证用户可调用 */ }
}
```

# 5. 三者综合示例

```java
@Stateless
@Loggable                     // 拦截器绑定
@RolesAllowed("admin")        // 公共注解-安全
public class OrderEJB {

    @Resource(lookup = "java:comp/env/jdbc/MyDS")
    private DataSource ds;

    @PostConstruct
    public void init() {
        System.out.println("OrderEJB 初始化");
    }

    @PreDestroy
    public void cleanup() {
        System.out.println("OrderEJB 销毁");
    }

    public void createOrder(Order o) {
        // 被 LoggingInterceptor 拦截
    }
}
```

# 6. 与 Spring 对应关系速查

| Jakarta 概念 | Spring 等价 |
|--------------|-------------|
| `@AroundInvoke` | `@Around` 切面通知 |
| `@Interceptor` + `@InterceptorBinding` | `@Aspect` + 切点表达式 |
| `@PostConstruct` / `@PreDestroy` | 同注解（`jakarta.annotation` 或 `javax.annotation`） |
| `@Resource` | `@Resource`（Spring 兼容） |
| `@RolesAllowed` | `@PreAuthorize` |
| Managed Bean 生命周期 | Spring Bean 生命周期回调 |
