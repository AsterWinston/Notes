# 1. 简单工厂模式
BeanFactory的getBean()方法，通过唯一标识来获取Bean对象。是典型的简单工厂模式（静态工厂模式）；
简单工厂模式不为23中设计模式之一

# 2. 工厂方法模式
FactoryBean是典型的工厂方法模式。在配置文件中通过factory-method属性来指定工厂方法，该方法是一个实例方法。

# 3. 单例模式
Spring用的是双重判断加锁的单例模式。
请看下面代码，我们之前讲解Bean的循环依赖的时候见过
# 4. 代理模式
Spring的AOP就是使用了动态代理实现的。
# 5. 装饰器模式
JavaSE中的IO流是非常典型的装饰器模式。BufferedReader就是装饰者，InputStream就是被装饰着
装饰者可以对被装饰着进行功能增强
Spring 中配置 DataSource 的时候，这些dataSource可能是各种不同类型的，比如不同的数据库：Oracle、SQL Server、MySQL等，也可能是不同的数据源：比如apache 提供的org.apache.commons.dbcp.BasicDataSource、spring提供的org.springframework.jndi.JndiObjectFactoryBean等。
这时，能否在尽可能少修改原有类代码下的情况下，做到动态切换不同的数据源？此时就可以用到装饰者模式。
Spring根据每次请求的不同，将dataSource属性设置成不同的数据源，以到达切换数据源的目的。外层进行了包装装饰
Spring中类名中带有：Decorator和Wrapper单词的类，都是装饰器模式。
# 6. 观察者模式
定义对象间的一对多的关系，当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并自动更新。
Spring中观察者模式一般用在listener的实现。
Spring中的事件编程模型就是观察者模式的实现。在Spring中定义了一个ApplicationListener（应用监听器）接口，用来监听Application的事件，Application其实就是ApplicationContext，ApplicationContext内置了几个事件，其中比较容易理解的是：ContextRefreshedEvent（上下文刷新事件）、ContextStartedEvent（上下文开启事件）、ContextStoppedEvent（上下文停止事件）、ContextClosedEvent（上下文关闭事件）
被观察的对象ApplicationContext执行相应的事件，观察者ApplicationListener中相应的方法就会执行
# 7. 策略模式
策略模式的重要体现，面向接口编程，实现类进行切换，使用者感受不到
策略模式是行为性模式，调用不同的方法，适应行为的变化 ，强调父类的调用子类的特性，运行时为父类类型，切换具体的实现子类对象，使用者感受不到。
getHandler是HandlerMapping接口中的唯一方法，用于根据请求找到匹配的处理器。
比如我们自己写了AccountDao接口，然后这个接口下有不同的实现类：AccountDaoForMySQL，AccountDaoForOracle。对于service来说不需要关心底层具体的实现，只需要面向AccountDao接口调用，底层可以灵活切换实现，这就是策略模式。
# 8. 模板方法模式
Spring中的JdbcTemplate类就是一个模板类。它就是一个模板方法设计模式的体现。
在模板类的模板方法execute中编写核心算法，具体的实现步骤在子类中完成。
HttpServlet中的service方法中已经编写请求的核心处理流程，相应类型的请求调用相应的处理请求的方法，该处理请求的方法的具体实现延迟到用户继承HttpServlet重写方法时来实现，处理调用时调用父类HttpServlet中service方法，请求的具体处理调用子类的方法

# 9. 源码实例：各设计模式在 Spring 中的代码体现

## 9.1 简单工厂模式 — BeanFactory.getBean()

```
// BeanFactory 根据 beanName 返回不同的 Bean 实例
BeanFactory factory = new ClassPathXmlApplicationContext("spring.xml");
Object user = factory.getBean("user");    // 返回 User 实例
Object order = factory.getBean("order");  // 返回 Order 实例

// DefaultListableBeanFactory 的核心存储：
private final Map<String, BeanDefinition> beanDefinitionMap =
        new ConcurrentHashMap<>(256);
// 通过 String key 获取 BeanDefinition → createBean() → 返回具体产品
```

## 9.2 工厂方法模式 — FactoryBean

```
// FactoryBean 接口定义了工厂方法 getObject()
public interface FactoryBean<T> {
    T getObject() throws Exception;       // 工厂方法：创建产品
    Class<?> getObjectType();             // 返回产品类型
    default boolean isSingleton() { return true; }
}

// Spring 内部大量使用 FactoryBean：
// - SqlSessionFactoryBean → 创建 SqlSessionFactory
// - ProxyFactoryBean → 创建 AOP 代理
// - JndiObjectFactoryBean → JNDI 查找

// AbstractBeanFactory 中的处理：
if (beanInstance instanceof FactoryBean<?> factoryBean) {
    object = getObjectFromFactoryBean(factoryBean, beanName, !synthetic);
    // 内部调用 factoryBean.getObject()
}
```

## 9.3 单例模式 — 双重检查锁定

```
// DefaultSingletonBeanRegistry.getSingleton()：
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // ★ 第一次检查（不加锁）— 一级缓存
    Object singletonObject = this.singletonObjects.get(beanName);

    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        // ★ 同步块（加锁）
        synchronized (this.singletonObjects) {
            // ★ 第二次检查 — 双重检查锁定
            singletonObject = this.singletonObjects.get(beanName);
            if (singletonObject == null) {
                singletonObject = this.earlySingletonObjects.get(beanName);
                if (singletonObject == null) {
                    ObjectFactory<?> singletonFactory =
                            this.singletonFactories.get(beanName);
                    if (singletonFactory != null) {
                        singletonObject = singletonFactory.getObject();
                        this.earlySingletonObjects.put(beanName, singletonObject);
                        this.singletonFactories.remove(beanName);
                    }
                }
            }
        }
    }
    return singletonObject;
}
```

## 9.4 代理模式 — JdkDynamicAopProxy

```
// JDK 动态代理的核心：InvocationHandler.invoke()
final class JdkDynamicAopProxy implements AopProxy, InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 1. 获取拦截器链
        List<Object> chain = this.advised
                .getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
        if (chain.isEmpty()) {
            // 2. 无拦截器 → 直接调用目标方法
            return AopUtils.invokeJoinpointUsingReflection(target, method, args);
        }
        // 3. ★ 创建 MethodInvocation 驱动责任链
        MethodInvocation invocation = new ReflectiveMethodInvocation(
                proxy, target, method, args, targetClass, chain);
        return invocation.proceed();
    }
}
```

## 9.5 装饰器模式 — TransactionAwareDataSourceProxy

```
// 类名带 Decorator/Wrapper 的都是装饰器模式
public class TransactionAwareDataSourceProxy extends DelegatingDataSource {
    @Override
    public Connection getConnection() throws SQLException {
        // ★ 不从原始 DataSource 获取连接
        //    而是从 TransactionSynchronizationManager 获取事务连接
        return getTransactionAwareConnectionProxy(obtainTargetDataSource());
    }
    protected Connection getTransactionAwareConnectionProxy(DataSource ds) {
        Connection con = DataSourceUtils.doGetConnection(ds);
        // ★ 返回代理连接：close() 不真正关闭，而是归还给事务管理器
        return getCloseSuppressingConnectionProxy(con);
    }
}
// 被装饰者：原始 DataSource → 装饰者：TransactionAwareDataSourceProxy
// 增强：事务感知 + 智能关闭 → 对外接口不变
```

## 9.6 观察者模式 — ApplicationEvent 机制

```
// 1. 自定义事件
public class OrderCreatedEvent extends ApplicationEvent {
    private final Long orderId;
    public OrderCreatedEvent(Object source, Long orderId) {
        super(source);
        this.orderId = orderId;
    }
}

// 2. 监听器（也可用 @EventListener 注解简化）
@Component
public class OrderEventListener
        implements ApplicationListener<OrderCreatedEvent> {
    @Override
    public void onApplicationEvent(OrderCreatedEvent event) {
        System.out.println("订单创建，ID=" + event.getOrderId());
    }
}

// 3. 源码中的事件广播：
// ApplicationEventMulticaster.multicastEvent()
// → 遍历所有 ApplicationListener → invokeListener(listener, event)

// 4. 发布事件：
// applicationContext.publishEvent(new OrderCreatedEvent(this, 123L));

// 内置事件：
// ContextRefreshedEvent — 容器刷新完成
// ContextClosedEvent — 容器关闭
```

## 9.7 策略模式 — 接口切换

```
// 数据访问层策略
public interface AccountDao { void insert(); }
@Repository("mysqlDao")
public class AccountDaoForMySQL implements AccountDao { ... }
@Repository("oracleDao")
public class AccountDaoForOracle implements AccountDao { ... }

@Service
public class AccountService {
    @Autowired @Qualifier("mysqlDao")  // 切换实现即可切换策略
    private AccountDao accountDao;
}

// Spring 内部策略应用：
// 1. InstantiationStrategy: Simple vs CGLIB
// 2. AopProxyFactory: JdkDynamicAopProxy vs ObjenesisCglibAopProxy
// 3. Propagation: 7 种事务传播行为
```

## 9.8 模板方法模式 — JdbcTemplate.execute()

```
// 模板方法：定义"获取连接→创建语句→执行→释放资源"的骨架
public <T> T execute(PreparedStatementCreator psc,
        PreparedStatementCallback<T> action) {
    // ★ 骨架（不可变）
    Connection con = DataSourceUtils.getConnection(obtainDataSource());
    PreparedStatement ps = null;
    try {
        ps = psc.createPreparedStatement(con);
        applyStatementSettings(ps);
        T result = action.doInPreparedStatement(ps); // ★ 钩子：具体逻辑
        handleWarnings(ps);
        return result;
    } catch (SQLException ex) {
        throw translateException(...);
    } finally {
        JdbcUtils.closeStatement(ps);
        DataSourceUtils.releaseConnection(con, ...);
    }
}
```
