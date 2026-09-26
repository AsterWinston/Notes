
# 1. 概述

## 1.1 什么是 CDI

CDI（Contexts and Dependency Injection，上下文与依赖注入）是 JavaEE 的**类型安全依赖注入**规范，定义于 JSR-299，参考实现为 **Weld**。CDI 为 JavaEE 组件（EJB、Servlet、JSF Bean、JPA 等）提供统一的依赖注入与生命周期管理。

## 1.2 Jakarta DI vs CDI

| 对比项 | Jakarta DI（JSR-330） | CDI（JSR-299） |
|--------|----------------------|----------------|
| 定位 | 最小化的依赖注入规范 | 完整的上下文与 DI 平台 |
| 核心注解 | `@Inject`、`@Named`、`@Singleton`、`@Qualifier` | `@Inject` + 作用域/限定符/事件/生产者/装饰器等 |
| 依赖 | 仅需要 `jakarta.inject` | 建立在 JSR-330 之上 |
| 能力 | 注入 | 注入 + 上下文作用域 + 事件 + 拦截器 + 装饰器 + 生产者 |
| 类比 Spring | 类似 `@Autowired` 基础版 | 类似 Spring Core 全家 |

- 实际应用中常把两者统称 CDI 使用，`@Inject` 来自 JSR-330，作用域/事件等来自 JSR-299。

## 1.3 与 Spring 的对应关系

| CDI/Jakarta DI | Spring | 说明 |
|----------------|--------|------|
| `@Inject` | `@Autowired` | 依赖注入 |
| `@Named` | `@Component` | Bean 注册 |
| `@ApplicationScoped` | `@Component`（单例） | 应用作用域 |
| `@RequestScoped` / `@SessionScoped` | `@RequestScope` / `@SessionScope` | Web 作用域 |
| `@Dependent` | 原型/每次都新建 | 默认作用域 |
| `@Produces` | `@Bean` | 生产者方法 |
| `@Observes` | `@EventListener` | 事件监听 |
| `@Interceptors` | `@Aspect` / `HandlerInterceptor` | 拦截器 |
| `@Qualifier` | `@Qualifier` | 限定符 |

# 2. CDI 依赖注入基础

## 2.1 使用 `@Inject`

```java
public class OrderService {
    @Inject
    private PaymentService paymentService;   // 字段注入

    @Inject
    public void setLogger(LoggerService logger) { }   // 方法注入

    @Inject
    public OrderService(OrderRepository repo) { }      // 构造器注入（推荐）
}
```

- `@Inject` 可注解字段、构造器、方法。
- 多个构造器时只能有一个标注 `@Inject`。
- 从 JavaEE 7 起，无参构造器不是必须的（CDI 1.1 后可用带参构造器注入）。

## 2.2 Bean 的可发现性（Bean Discovery）

CDI 默认扫描打包中的 **Bean 归档**（`beans.xml` 存在，或隐式 bean 归档）：

```xml
<!-- beans.xml（可空文件，也可配置 bean-discovery-mode） -->
<beans xmlns="http://xmlns.jcp.org/xml/ns/javaee"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
                           http://xmlns.jcp.org/xml/ns/javaee/beans_2_0.xsd"
       version="2.0" bean-discovery-mode="annotated"/>
```

| bean-discovery-mode | 说明 |
|---------------------|------|
| `all` | 发现所有类（含无注解的类） |
| `annotated` | 只发现带 CDI 注解的类（默认） |
| `none` | 禁用 CDI |

## 2.3 `@Named` 与 EL 集成

`@Named` 使 Bean 可用名称访问，并可在 EL 表达式（JSP/JSF）中使用：

```java
@Named
@RequestScoped
public class CartBean {
    public int getItemCount() { return 3; }
}
```

```xhtml
购物车数量：${cartBean.itemCount}
```

# 3. CDI 作用域（Scope）

## 3.1 内置作用域

| 作用域 | 注解 | 生命周期 |
|--------|------|----------|
| 依赖 | `@Dependent` | 与注入它的 Bean 同生共死（默认作用域） |
| 请求 | `@RequestScoped` | 一次 HTTP 请求 |
| 会话 | `@SessionScoped` | 一次 HTTP 会话 |
| 应用 | `@ApplicationScoped` | 应用生命周期（全局单例） |
| 对话 | `@ConversationScoped` | 需要时激活/结束的长时间对话 |
| 视图 | `@ViewScoped`（Jakarta Faces） | 一次视图 |

```java
@ApplicationScoped
public class CacheService { }

@SessionScoped
public class UserSession implements Serializable {
    private User currentUser;
    // getter/setter...
}
```

- 跨上下文共享的 Bean（`@SessionScoped`、`@ConversationScoped`、`@ViewScoped`）需要实现 `Serializable`。

## 3.2 自定义作用域

```java
@Qualifier
@Scope(CustomScope.class)
@Retention(RUNTIME)
public @interface MyScoped { }
```

# 4. 限定符（Qualifier）

当同一类型有多个 Bean 时，用限定符区分注入目标：

```java
@Qualifier
@Retention(RUNTIME)
@Target({FIELD, METHOD, PARAMETER, TYPE})
public @interface Premium { }
```

```java
@Premium
@ApplicationScoped
public class PremiumUserService implements UserService { }

@Default
@ApplicationScoped
public class NormalUserService implements UserService { }
```

```java
public class OrderService {
    @Inject @Premium
    private UserService userService;   // 注入 PremiumUserService
}
```

- `@Default` 是默认限定符，若只有一个实现可以省略。
- 配合 `@Named` 字符串限定在 CDI 2.0 已不推荐，优先用类型安全的限定符。

# 5. 生产者与消费（Producer / Disposer）

用 `@Produces` 生成无法直接构造的 Bean（如第三方对象、带参对象），用 `@Disposes` 释放资源：

```java
@ApplicationScoped
public class ConnectionProducer {

    @Produces
    @ApplicationScoped
    public Connection createConnection() {
        // 手工创建并返回
        return DriverManager.getConnection(URL, USER, PASS);
    }

    public void closeConnection(@Disposes Connection conn) {
        // Bean 销毁时自动调用
        conn.close();
    }
}
```

- 生产者的作用域通过额外注解控制（上例 `@ApplicationScoped` 表示生产的 Connection 为应用级单例）。

## 5.1 生产者字段

```java
@Produces @Named("maxUsers") int maxUsers = 1000;
```

# 6. 事件（Events）

CDI 事件实现组件间的**松耦合通信**：

```java
// 事件类型
public class OrderCompleted {
    private final long orderId;
    public OrderCompleted(long orderId) { this.orderId = orderId; }
    public long getOrderId() { return orderId; }
}
```

```java
// 触发事件
public class OrderService {
    @Inject
    private Event<OrderCompleted> orderEvent;

    public void checkout() {
        // ... 业务逻辑
        orderEvent.fire(new OrderCompleted(1001L));
    }
}
```

```java
// 监听事件
public class NotificationService {
    public void onOrderCompleted(@Observes OrderCompleted event) {
        System.out.println("订单完成：" + event.getOrderId());
    }
}
```

## 6.1 事务观察者

```java
public void onOrder(@Observes(during = TransactionPhase.AFTER_SUCCESS) OrderCompleted e) {
    // 仅在事务成功提交后执行，适合发短信/邮件等
}
```

## 6.2 异步事件（CDI 2.0+）

```java
event.fireAsync(new OrderCompleted(1001L));
// 观察者方法返回 CompletionStage 或使用 @Async
```

# 7. 拦截器（Interceptor）

## 7.1 定义拦截器

```java
@Interceptor
@Loggable
public class LoggingInterceptor {
    @AroundInvoke
    public Object log(InvocationContext ctx) throws Exception {
        long start = System.currentTimeMillis();
        try {
            return ctx.proceed();
        } finally {
            System.out.println(ctx.getMethod() + " 耗时 " +
                    (System.currentTimeMillis() - start) + "ms");
        }
    }
}
```

## 7.2 使用拦截器

```java
@Loggable
@ApplicationScoped
public class ReportService {
    public void generate() { /* ... */ }
}
```

- 需在 `beans.xml` 中启用拦截器，或使用 `@Priority`：

```xml
<interceptors>
    <class>com.example.LoggingInterceptor</class>
</interceptors>
```

> 详细见 `6_interceptors_managed_beans.md`。

# 8. 装饰器（Decorator）

装饰器用于**增强**某个类型的所有注入点（与拦截器按方法不同，装饰器针对 Bean 整体）：

```java
@Decorator
public abstract class AccountServiceDecorator implements AccountService {
    @Inject @Delegate
    AccountService delegate;

    @Override
    public void withdraw(double amount) {
        // 前置增强
        delegate.withdraw(amount);
        // 后置增强
    }
}
```

```xml
<decorators>
    <class>com.example.AccountServiceDecorator</class>
</decorators>
```

# 9. 作用域与代理

## 9.1 客户端代理（Client Proxy）

- 注入的不是 Bean 实例本身，而是**客户端代理**。
- 调用方法时代理根据当前上下文解析真正的实例，从而保证作用域正确。

```
调用方 ──► 客户端代理 ──► 真正的 Bean 实例
           │
           └─ 根据作用域解析（请求/会话/应用上下文）
```

- `@Dependent` 作用域无代理，直接注入实例。
- 因此作用域为 `@RequestScoped` 的 Bean 注入到 `@ApplicationScoped` 单例中是安全的（通过代理）。

## 9.2 为什么需要代理

- 避免作用域泄漏：长生命周期 Bean 注入短生命周期 Bean 时，每次调用都重新解析。
- 使 EL 访问、跨层注入保持一致语义。

# 10. CDI 上下文（Context）

| 上下文 | 负责的组件 |
|--------|-----------|
| RequestContext | `@RequestScoped` Bean |
| SessionContext | `@SessionScoped` Bean |
| ApplicationContext | `@ApplicationScoped` Bean |
| ConversationContext | `@ConversationScoped` Bean |
| DependentContext | `@Dependent` Bean |

- 上下文负责 Bean 的创建、保存、销毁。
- 例如请求开始时创建请求上下文，请求结束时销毁其中的 Bean。

# 11. 实战：CDI 最小示例

## 11.1 依赖

```xml
<dependency>
    <groupId>jakarta.enterprise</groupId>
    <artifactId>jakarta.enterprise.cdi-api</artifactId>
    <version>4.0.1</version>
    <scope>provided</scope>
</dependency>
<dependency>
    <groupId>org.jboss.weld.se</groupId>
    <artifactId>weld-se-core</artifactId>
    <version>5.1.2.Final</version>
</dependency>
```

## 11.2 代码

```java
@ApplicationScoped
public class GreetingService {
    public String greet(String name) {
        return "Hello, " + name;
    }
}

public class Main {
    public static void main(String[] args) {
        try (WeldContainer container = new Weld().initialize()) {
            GreetingService service = container.select(GreetingService.class).get();
            System.out.println(service.greet("CDI"));
        }
    }
}
```

# 12. CDI 与 EJB 的关系

- EJB 会话 Bean 也是 CDI Bean，可以直接注入。
- CDI 解决"组件如何组装"，EJB 解决"企业能力：事务/安全/集群/消息"。
- 实践中二者常组合：EJB 提供声明式事务，CDI 负责装配。
