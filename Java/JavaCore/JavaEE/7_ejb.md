
# 1. 概述

## 1.1 什么是 EJB

EJB（Enterprise JavaBeans）是 JavaEE 的**服务器端业务组件规范**，由容器（应用服务器）统一管理业务对象的事务、安全、并发、集群、持久化等横切能力，让开发者专注于业务逻辑。

## 1.2 EJB 解决的痛点

```
没有 EJB 前（手写）：
  业务方法 + 手动开启事务 + 手动鉴权 + 手动管理并发 + 手动连接池...

有了 EJB 后（声明式）：
  加一个 @Stateless 注解，容器自动提供上述横切能力
```

## 1.3 发展历程

| 版本 | JavaEE | 关键变化 |
|------|--------|----------|
| EJB 1.0 | J2EE 1.2 | 引入，但需要 Home 接口、部署描述符，极其繁琐 |
| EJB 2.x | J2EE 1.3/1.4 | 复杂巅峰：Local/Remote Home、EJB-QL |
| EJB 3.0 | JavaEE 5 | **注解革命**：`@Stateless`、`@Stateful`、消除 Home 接口 |
| EJB 3.1 | JavaEE 6 | 单例 Bean `@Singleton`、嵌入式 API、无接口视图 |
| EJB 3.2 | JavaEE 7 | 简化 API，完善异步/定时 |
| Jakarta EJB | Jakarta EE 9+ | 包名迁移，与 CDI 融合 |

## 1.4 EJB 的现状

- EJB 在 JavaEE 生态中**地位很高但使用率下降**：多数能力已被 Spring 体系（Spring 事务、Spring Security、Spring Data）取代。
- 大型传统金融/电信系统仍存在大量 EJB 存量代码。
- 理解 EJB 有助于理解声明式事务、容器管理这些核心思想。

# 2. EJB 的类型

| 类型 | 注解 | 特点 |
|------|------|------|
| **无状态会话 Bean**（Stateless） | `@Stateless` | 无实例状态，方法间不保留数据，可池化复用 |
| **有状态会话 Bean**（Stateful） | `@Stateful` | 为单个客户端保存状态，生命周期与客户端会话绑定 |
| **单例 Bean**（Singleton） | `@Singleton` | 应用内唯一实例，全局共享状态 |
| **消息驱动 Bean**（MDB） | `@MessageDriven` | 异步消费 JMS 消息 |

## 2.1 类型选择

```
需要无状态服务、高并发（如查询、计算）  → @Stateless
需要客户端会话状态（如购物车）          → @Stateful
需要全局共享状态（如计数器、配置缓存）  → @Singleton
需要异步消费消息（如订单确认通知）      → @MessageDriven
```

# 3. 无状态会话 Bean

## 3.1 无接口视图（推荐）

```java
@Stateless
public class CalculatorService {
    public int add(int a, int b) {
        return a + b;
    }
}
```

- EJB 3.1+ 支持无接口视图：直接注入具体类，无需定义接口。
- 无状态 Bean 的实例可被池化，多个客户端共享同一实例池，**方法内不能依赖实例字段**。

## 3.2 接口视图

```java
@Local
public interface OrderService {
    void placeOrder(Order order);
}

@Stateless
public class OrderServiceBean implements OrderService {
    @Override
    public void placeOrder(Order order) { /* ... */ }
}
```

| 视图 | 注解 | 说明 |
|------|------|------|
| 本地接口 | `@Local` | 同一 JVM 内调用（同一应用） |
| 远程接口 | `@Remote` | 跨 JVM / 跨机器调用（基于 RMI） |
| 无接口 | 无 | EJB 3.1+，最简用法 |

## 3.3 注入 EJB

```java
@Stateless
public class OrderFacade {
    @EJB
    private OrderService orderService;   // 注入本地 EJB

    // 也可以通过 JNDI 查找
}
```

# 4. 有状态会话 Bean

```java
@Stateful
public class ShoppingCart {
    private List<Item> items = new ArrayList<>();

    public void addItem(Item item) {
        items.add(item);
    }

    public double getTotal() {
        return items.stream().mapToDouble(Item::getPrice).sum();
    }
}
```

- 每个客户端获得**独立实例**，通过 `@SessionScoped` CDI 或序列化保存引用。
- 状态保存在服务器内存，客户端与实例一一对应。
- 可使用 `@Remove` 标记移除方法：

```java
@Remove
public void checkout() {
    // 业务完成后移除该 Stateful Bean
}
```

## 4.1 有状态 Bean 的钝化

- 长时间未使用、内存不足时，有状态 Bean 可能被**钝化（序列化到磁盘）**。
- 字段需要实现 `Serializable`。
- 可配置超时，超时后被容器移除。

# 5. 单例 Bean

```java
@Singleton
@Startup          // 应用启动时立即初始化
@ConcurrencyManagement(ConcurrencyManagementType.CONTAINER)
public class AppConfig {
    private Map<String, String> config = new HashMap<>();

    @Lock(LockType.READ)
    public String get(String key) {
        return config.get(key);
    }

    @Lock(LockType.WRITE)
    public void put(String key, String value) {
        config.put(key, value);
    }
}
```

## 5.1 并发管理

| 模式 | 说明 |
|------|------|
| `@Lock(READ)` | 读锁，多个客户端可并发读 |
| `@Lock(WRITE)` | 写锁，互斥 |
| `@ConcurrencyManagement(BEAN)` | Bean 自己管理同步（加 `synchronized` 等） |

- 容器并发管理（默认）：`READ` 可共享，`WRITE` 互斥。
- `@Startup` + `@PostConstruct` 适合启动时加载配置/预热缓存。

# 6. 消息驱动 Bean（MDB）

```java
@MessageDriven(
    activationConfig = {
        @ActivationConfigProperty(propertyName = "destinationType",
                propertyValue = "jakarta.jms.Queue"),
        @ActivationConfigProperty(propertyName = "destinationLookup",
                propertyValue = "java:app/jms/orderQueue")
    }
)
public class OrderMessageListener implements MessageListener {

    @Override
    public void onMessage(Message message) {
        try {
            if (message instanceof ObjectMessage om) {
                Order order = (Order) om.getObject();
                // 异步处理订单
                System.out.println("处理订单：" + order.getId());
            }
        } catch (JMSException e) {
            e.printStackTrace();
        }
    }
}
```

- MDB 是**异步消息消费者**，容器负责监听队列/主题并把消息交给 Bean。
- 容器自动管理事务：消息处理与业务操作可以在同一事务中。

# 7. EJB 生命周期

## 7.1 无状态 Bean 生命周期

```
不存在
  │ 容器创建（池中）
  ▼
方法就绪（Pooled）  ← 池中复用，无状态
  │
  ▼
销毁
```

回调方法：`@PostConstruct`、`@PreDestroy`。

## 7.2 有状态 Bean 生命周期

```
不存在
  │ @PostConstruct
  ▼
就绪（Ready）
  │ 长时间未用 → @PrePassivate 钝化
  ▼
钝化（Passivated）
  │ @PostActivate 活化
  ▼
就绪
  │ @PreDestroy / @Remove
  ▼
不存在
```

## 7.3 单例 Bean 生命周期

```
不存在 → @PostConstruct → 就绪 → @PreDestroy → 不存在
```

# 8. EJB 定时器（Timer Service）

```java
@Stateless
public class TimerBean {

    @Schedule(hour = "2", minute = "0", persistent = false)
    public void dailyCleanup() {
        System.out.println("每日凌晨2点执行清理任务");
    }

    @Timeout
    public void onTimeout(Timer timer) {
        System.out.println("定时器触发：" + timer.getInfo());
    }
}
```

- `@Schedule`：声明式定时任务（cron 风格）。
- `@Timeout`：程序化创建定时器的回调。
- 定时器默认**持久化**，服务器重启后仍会恢复。

# 9. EJB 异步调用

```java
@Stateless
public class MailService {

    @Asynchronous
    public Future<Boolean> sendWelcomeMail(String email) {
        // 长时间任务，异步执行
        return new AsyncResult<>(true);
    }
}
```

- `@Asynchronous` 让方法在容器线程池中异步执行。
- 返回值可以是 `void`（Fire-and-forget）或 `Future`（可后续获取结果）。

# 10. EJB 与事务

## 10.1 容器管理事务（CMT）

```java
@Stateless
public class TransferService {

    @Resource
    private SessionContext ctx;

    @TransactionAttribute(TransactionAttributeType.REQUIRED)
    public void transfer(Long from, Long to, double amount) {
        // 默认 REQUIRED：没有事务则新建，有则加入
        // 方法正常返回 → 容器提交；抛运行时异常 → 容器回滚
    }
}
```

- 默认事务属性就是 `REQUIRED`。
- 运行时异常自动回滚，业务异常可通过 `SessionContext.setRollbackOnly()` 手动回滚。

## 10.2 Bean 管理事务（BMT）

```java
@Stateless
@TransactionManagement(TransactionManagementType.BEAN)
public class ManualTxService {

    @Resource
    private UserTransaction utx;

    public void doWork() throws Exception {
        utx.begin();
        try {
            // 手动控制
            utx.commit();
        } catch (Exception e) {
            utx.rollback();
            throw e;
        }
    }
}
```

> 事务细节见 `12_jta_transactions.md`。

# 11. EJB 与 CDI 的融合

- 自 JavaEE 7（CDI 1.1）起，EJB 会话 Bean **本身也是 CDI Bean**：

```java
@Stateless
public class OrderService {
    @Inject
    private PaymentService paymentService;   // CDI 注入
}
```

- 可以在 JSF 托管 Bean、Servlet、其他 EJB 中注入 EJB。

# 12. EJB 的经典缺陷与经验

| 问题 | 说明 |
|------|------|
| 历史包袱重 | EJB 2.x 时代 API 极其繁琐，给人留下坏印象 |
| 分布式滥用 | 过度使用 `@Remote` 导致性能差（每个调用都走 RMI） |
| 单例并发陷阱 | `@Singleton` + 容器锁在高并发下易成为瓶颈 |
| 与 Spring 重叠 | Spring 已提供等效的声明式事务、AOP、注入 |
| 无状态 Bean 池陷阱 | 依赖实例字段存状态是典型 bug |

## 12.1 企业选型建议

- 新项目优先 Spring Boot，**不直接使用 EJB**。
- 存量 EJB 系统维护时需理解：CMT 事务边界、无状态 Bean 无状态约束、JNDI 查找方式。
- EJB 的核心思想（声明式事务、容器管理）仍值得学习，与 Spring 的 AOP 事务一脉相承。

# 13. 与 Spring 对照

| EJB 概念 | Spring 等价 |
|----------|-------------|
| `@Stateless` | `@Service` / `@Component`（无状态单例） |
| `@Stateful` | `@Scope("session")` / `@SessionScope` |
| `@Singleton` | `@Component` 默认单例 |
| `@MessageDriven` | `@JmsListener`（Spring JMS） |
| `@TransactionAttribute` | `@Transactional` |
| `@Asynchronous` | `@Async` |
| `@Schedule` | `@Scheduled` |
| `@EJB` 注入 | `@Autowired` |
| `SessionContext` | `TransactionAspectSupport` 等 |
