# 1.MyBatis-Spring 整合概述

```
MyBatis-Spring 是将 MyBatis 无缝集成到 Spring 框架中的桥梁模块。

核心目标：
  1. 让 Spring 管理 MyBatis 的 SqlSessionFactory 和 Mapper 对象的生命周期
  2. 将 MyBatis 的事务纳入 Spring 的事务管理
  3. 简化 Mapper 的注入（不需要手动调用 getMapper）
  4. 提供线程安全的 SqlSession（SqlSessionTemplate）

版本对应关系：
  MyBatis-Spring 1.x → Spring 4.x + MyBatis 3.x
  MyBatis-Spring 2.x → Spring 5.x + MyBatis 3.5+
  MyBatis-Spring 3.x → Spring 6.x (Spring Boot 3) + MyBatis 3.5+
  mybatis-spring-boot-starter → Spring Boot 自动配置
```

# 2.核心组件

```
┌─────────────────────────────────────────────────────────────────┐
│                  MyBatis-Spring 核心组件                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SqlSessionFactoryBean                                           │
│    将 MyBatis 的 SqlSessionFactory 作为一个 Spring Bean 管理      │
│    实现了 FactoryBean<SqlSessionFactory> 和 InitializingBean     │
│                                                                 │
│  SqlSessionTemplate                                             │
│    SqlSession 接口的线程安全实现                                  │
│    内部使用动态代理 + TransactionSynchronizationManager            │
│    每次操作从 Spring 事务管理器获取或创建 SqlSession                │
│                                                                 │
│  MapperFactoryBean                                               │
│    为每个 Mapper 接口生成代理对象的工厂 Bean                        │
│    实现了 FactoryBean<T> 接口                                    │
│                                                                 │
│  MapperScannerConfigurer                                         │
│    自动扫描指定包下的 Mapper 接口                                 │
│    自动注册 MapperFactoryBean 到 Spring 容器                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

# 3.SqlSessionFactoryBean — 工厂 Bean

## 3.1 源码分析

```java
/**
 * SqlSessionFactoryBean 是 Spring 管理 MyBatis 的入口
 * 源码：org.mybatis.spring.SqlSessionFactoryBean
 *
 * 实现了两个关键接口：
 *   - FactoryBean<SqlSessionFactory>: 通过 getObject() 返回 SqlSessionFactory
 *   - InitializingBean: 在属性设置完毕后调用 afterPropertiesSet() 完成初始化
 *   - ApplicationListener: 监听 Spring 事件
 */
public class SqlSessionFactoryBean implements FactoryBean<SqlSessionFactory>,
        InitializingBean, ApplicationListener<ApplicationEvent> {

    // Spring 配置属性
    private Resource configLocation;            // mybatis-config.xml 的位置
    private Configuration configuration;        // 直接注入 Configuration
    private Resource[] mapperLocations;         // XxxMapper.xml 文件位置
    private DataSource dataSource;              // 数据源
    private TransactionFactory transactionFactory; // 事务工厂

    // 以下是 mybatis-config.xml 中可选配置的等价属性
    private Properties configurationProperties; // 属性
    private Interceptor[] plugins;              // 插件
    private TypeHandler[] typeHandlers;         // 类型处理器
    private TypeAlias[] typeAliases;            // 类型别名
    private DatabaseIdProvider databaseIdProvider; // 数据库厂商标识
    private Class<?>[] typeAliasesClasses;      // 别名扫描
    private LanguageDriver[] scriptingLanguageDrivers; // 语言驱动
    private Cache cache;                        // 二级缓存引用
    private ObjectFactory objectFactory;        // 对象工厂
    private ObjectWrapperFactory objectWrapperFactory; // 对象包装工厂

    private SqlSessionFactory sqlSessionFactory; // 最终构建的 SqlSessionFactory

    /**
     * 在 Spring 容器初始化该 Bean 时，
     * 在所有属性设置完成后被调用
     */
    @Override
    public void afterPropertiesSet() throws Exception {
        // ... 参数校验 ...
        this.sqlSessionFactory = buildSqlSessionFactory();
    }

    /**
     * 核心：构建 SqlSessionFactory
     *
     * 逻辑：
     *   1. 如果指定了 configLocation → 用 XMLConfigBuilder 解析
     *      （配置中的 dataSource/plugins 等会被 Spring 配置覆盖）
     *   2. 如果没指定 configLocation → 直接 new Configuration()
     *   3. 设置各种配置属性到 Configuration
     *   4. new SqlSessionFactoryBuilder().build(configuration)
     *
     * 关键点：无论哪种方式，都会将 dataSource 注入到 Configuration 中。
     *        所以只需要在 Spring 中配置 dataSource 即可。
     */
    protected SqlSessionFactory buildSqlSessionFactory() throws Exception {
        final Configuration targetConfiguration;

        // 方式1：通过 XML 配置文件
        if (this.configuration != null) {
            targetConfiguration = this.configuration;
        } else if (this.configLocation != null) {
            // ★ 用 XMLConfigBuilder 解析 mybatis-config.xml ★
            // 注意：XML 中的 environments/dataSource 配置会被 Spring 的 dataSource 覆盖
            XMLConfigBuilder xmlConfigBuilder = new XMLConfigBuilder(
                this.configLocation.getInputStream(), null, this.configurationProperties);
            targetConfiguration = xmlConfigBuilder.getConfiguration();
        } else {
            // 方式2：纯 Java 配置（常见于 Spring Boot）
            targetConfiguration = new Configuration();
            targetConfiguration.setVariables(this.configurationProperties);
        }

        // ★ 设置各种可选配置 ★
        if (this.objectFactory != null) {
            targetConfiguration.setObjectFactory(this.objectFactory);
        }
        if (this.objectWrapperFactory != null) {
            targetConfiguration.setObjectWrapperFactory(this.objectWrapperFactory);
        }

        // ★ 设置 DataSource ★
        if (this.dataSource != null) {
            targetConfiguration.setEnvironment(new Environment("SqlSessionFactoryBean",
                this.transactionFactory == null
                    ? new SpringManagedTransactionFactory()
                    : this.transactionFactory,
                this.dataSource));
        }

        // ★ 设置插件 ★
        if (!isEmpty(this.plugins)) {
            for (Interceptor plugin : this.plugins) {
                targetConfiguration.addInterceptor(plugin);
            }
        }

        // ★ 解析 Mapper 文件 ★
        if (!isEmpty(this.mapperLocations)) {
            for (Resource mapperLocation : this.mapperLocations) {
                XMLMapperBuilder xmlMapperBuilder = new XMLMapperBuilder(
                    mapperLocation.getInputStream(),
                    targetConfiguration,
                    mapperLocation.toString(),
                    targetConfiguration.getSqlFragments());
                xmlMapperBuilder.parse();
            }
        }

        // ★ 构建 SqlSessionFactory ★
        return this.sqlSessionFactoryBuilder.build(targetConfiguration);
    }

    /**
     * FactoryBean 的 getObject() 方法
     * Spring 调用此方法获取实际要注册的 Bean
     */
    @Override
    public SqlSessionFactory getObject() throws Exception {
        if (this.sqlSessionFactory == null) {
            afterPropertiesSet();
        }
        return this.sqlSessionFactory;
    }

    @Override
    public Class<? extends SqlSessionFactory> getObjectType() {
        return this.sqlSessionFactory == null ? SqlSessionFactory.class : this.sqlSessionFactory.getClass();
    }

    @Override
    public boolean isSingleton() {
        return true;  // SqlSessionFactory 是单例
    }
}
```

## 3.2 XML 配置方式

```xml
<!-- Spring applicationContext.xml -->
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans.xsd">

    <!-- 1. 数据源 -->
    <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
        <property name="driverClassName" value="com.mysql.cj.jdbc.Driver"/>
        <property name="url" value="jdbc:mysql://localhost:3306/test"/>
        <property name="username" value="root"/>
        <property name="password" value="123456"/>
    </bean>

    <!-- 2. SqlSessionFactoryBean -->
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <!-- 数据源（必填，会覆盖 XML 中的配置） -->
        <property name="dataSource" ref="dataSource"/>
        <!-- MyBatis 核心配置文件（可选） -->
        <property name="configLocation" value="classpath:mybatis-config.xml"/>
        <!-- Mapper.xml 映射文件位置（可选，也可以直接在 mybatis-config.xml 中配置） -->
        <property name="mapperLocations" value="classpath:mapper/*.xml"/>
        <!-- 插件 -->
        <property name="plugins">
            <array>
                <bean class="com.github.pagehelper.PageInterceptor"/>
            </array>
        </property>
        <!-- 属性 -->
        <property name="configurationProperties">
            <props>
                <prop key="mapUnderscoreToCamelCase">true</prop>
            </props>
        </property>
    </bean>

    <!-- 3. Mapper 接口扫描（方式一：逐个配置） -->
    <bean id="carMapper" class="org.mybatis.spring.mapper.MapperFactoryBean">
        <property name="mapperInterface" value="com.test.mapper.CarMapper"/>
        <property name="sqlSessionFactory" ref="sqlSessionFactory"/>
    </bean>

    <!-- 3. Mapper 接口扫描（方式二：批量扫描，推荐） -->
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="basePackage" value="com.test.mapper"/>
        <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"/>
    </bean>

</beans>
```

## 3.3 Spring Boot 配置方式

```yaml
# application.yml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/test
    username: root
    password: 123456
    type: com.alibaba.druid.pool.DruidDataSource

# MyBatis 配置
mybatis:
  # 核心配置文件位置（两种方式二选一）
  config-location: classpath:mybatis-config.xml

  # 或者直接在这里配置（和 config-location 不要同时用）
  # configuration:
  #   map-underscore-to-camel-case: true
  #   cache-enabled: true
  #   lazy-loading-enabled: true

  # Mapper.xml 文件位置
  mapper-locations: classpath:mapper/*.xml

  # 类型别名包（多个用逗号分隔）
  type-aliases-package: com.test.pojo

  # TypeHandler 扫描包
  type-handlers-package: com.test.handler

  # 属性配置
  configuration-properties:
    some-key: some-value
```

# 4.SqlSessionTemplate — 线程安全的 SqlSession

```java
/**
 * SqlSessionTemplate 是 SqlSession 的线程安全实现
 * 源码：org.mybatis.spring.SqlSessionTemplate
 *
 * 为什么需要 SqlSessionTemplate？
 *   - MyBatis 原生的 DefaultSqlSession 是线程不安全的
 *   - 在 Spring 中，Service 层的 Bean 是单例的，多线程共享
 *   - 如果直接注入 DefaultSqlSession 会出现线程安全问题
 *   - SqlSessionTemplate 通过动态代理实现了线程安全
 *
 * 核心机制：
 *   1. 通过 SqlSessionFactory 为每个线程/事务创建独立的 SqlSession
 *   2. 创建的 SqlSession 绑定到 Spring 的 TransactionSynchronizationManager
 *   3. 同一事务内的多次操作共享同一个 SqlSession
 *   4. 事务提交时统一 commit
 *   5. 事务回滚时统一 rollback
 *   6. 事务结束后统一 close
 */
public class SqlSessionTemplate implements SqlSession, DisposableBean {

    private final SqlSessionFactory sqlSessionFactory;
    private final ExecutorType executorType;

    // ★ 核心：SqlSession 的代理对象 ★
    // 通过代理拦截所有方法调用，在方法内部获取"当前事务的 SqlSession"
    private final SqlSession sqlSessionProxy;

    public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
        this(sqlSessionFactory, sqlSessionFactory.getConfiguration().getDefaultExecutorType());
    }

    public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType) {
        this.sqlSessionFactory = sqlSessionFactory;
        this.executorType = executorType;

        // ★ 创建代理对象 ★
        // 每次调用 SqlSession 的方法时，代理会获取当前事务的 SqlSession 来执行
        this.sqlSessionProxy = (SqlSession) Proxy.newProxyInstance(
            SqlSessionFactory.class.getClassLoader(),
            new Class[] { SqlSession.class },
            new SqlSessionInterceptor()
        );
    }

    /**
     * 内部代理类，负责获取当前事务的 SqlSession
     */
    private class SqlSessionInterceptor implements InvocationHandler {
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            // ★ 获取当前 Spring 事务管理的 SqlSession ★
            SqlSession sqlSession = getSqlSession(
                SqlSessionTemplate.this.sqlSessionFactory,
                SqlSessionTemplate.this.executorType,
                SqlSessionTemplate.this.exceptionTranslator
            );

            try {
                // 委托给实际的 SqlSession 执行
                Object result = method.invoke(sqlSession, args);

                // 如果不是 Spring 管理的事务 → 手动提交
                if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
                    sqlSession.commit(true);
                }
                return result;
            } catch (Throwable t) {
                // 异常处理：将 MyBatis 的 PersistenceException 转换为 Spring 的 DataAccessException
                Throwable unwrapped = unwrapThrowable(t);
                throw unwrapped;
            } finally {
                // 如果不是 Spring 管理的事务 → 手动关闭
                if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
                    sqlSession.close();
                }
            }
        }
    }

    /**
     * 获取当前事务的 SqlSession
     *
     * 逻辑：
     *   1. 从 TransactionSynchronizationManager 获取当前事务的 SqlSessionHolder
     *   2. 如果已存在 → 直接返回（同一事务复用）
     *   3. 如果不存在 → 创建新的 SqlSession → 注册到 TransactionSynchronizationManager
     */
    public static SqlSession getSqlSession(SqlSessionFactory sessionFactory,
                                           ExecutorType executorType,
                                           PersistenceExceptionTranslator exceptionTranslator) {
        // 1. 尝试从事务同步管理器中获取
        SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager
            .getResource(sessionFactory);

        SqlSession session = sessionHolder(executorType, holder);
        if (session != null) {
            return session;
        }

        // 2. 不存在 → 创建新的 SqlSession
        session = sessionFactory.openSession(executorType);

        // 3. 如果当前有 Spring 事务 → 注册到事务同步管理器
        if (TransactionSynchronizationManager.isSynchronizationActive()) {
            // 绑定到当前事务
            registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);
            // 注册事务同步回调（在事务提交/回滚时执行）
            TransactionSynchronizationManager.registerSynchronization(
                new SqlSessionSynchronization(holder, sessionFactory));
            holder.setSynchronizedWithTransaction(true);
            holder.requested();
        }

        return session;
    }

    // ==================== 所有方法都委托给 sqlSessionProxy ====================

    @Override
    public <T> T selectOne(String statement) {
        return this.sqlSessionProxy.selectOne(statement);
    }

    @Override
    public <T> T selectOne(String statement, Object parameter) {
        return this.sqlSessionProxy.selectOne(statement, parameter);
    }

    @Override
    public int insert(String statement) {
        return this.sqlSessionProxy.insert(statement);
    }

    @Override
    public <T> T getMapper(Class<T> type) {
        return getConfiguration().getMapper(type, this);
    }

    // ... 其他方法都类似地委托给 sqlSessionProxy
}
```

# 5.MapperFactoryBean — Mapper 接口的工厂 Bean

```java
/**
 * MapperFactoryBean 负责为每个 Mapper 接口创建代理对象
 * 源码：org.mybatis.spring.mapper.MapperFactoryBean
 *
 * 实现了 FactoryBean<T>，其中 T 就是 Mapper 接口类型
 *
 * 核心逻辑：
 *   getObject() 返回的是 sqlSessionTemplate.getMapper(mapperInterface) 的结果
 *   这个结果是一个 MapperProxy 代理对象
 */
public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T> {

    private Class<T> mapperInterface;

    public MapperFactoryBean(Class<T> mapperInterface) {
        this.mapperInterface = mapperInterface;
    }

    /**
     * 创建 Mapper 代理对象
     * 最终调用 SqlSession.getMapper(mapperInterface)
     */
    @Override
    public T getObject() throws Exception {
        // getSqlSession() 返回 SqlSessionTemplate（线程安全）
        return getSqlSession().getMapper(this.mapperInterface);
    }

    @Override
    public Class<T> getObjectType() {
        return this.mapperInterface;
    }

    @Override
    public boolean isSingleton() {
        return true;  // 返回的是代理对象，可以共享
        // 注意：虽然返回的是单例代理，但代理内部会根据当前事务获取不同的 SqlSession
    }
}
```

# 6.MapperScannerConfigurer — 自动扫描 Mapper

```java
/**
 * MapperScannerConfigurer 自动扫描 Mapper 接口并注册为 Spring Bean
 * 源码：org.mybatis.spring.mapper.MapperScannerConfigurer
 *
 * 核心逻辑：
 *   1. 根据 basePackage 扫描类路径
 *   2. 找到所有接口
 *   3. 为每个接口创建 MapperFactoryBean 的 BeanDefinition
 *   4. 注册到 Spring 容器
 *
 * 在 Spring Boot 中，@MapperScan 注解等价于 MapperScannerConfigurer
 */
public class MapperScannerConfigurer implements BeanDefinitionRegistryPostProcessor, InitializingBean, ApplicationContextAware {

    private String basePackage;                        // 扫描的基础包
    private String sqlSessionFactoryBeanName;          // SqlSessionFactory Bean 名称
    private String sqlSessionTemplateBeanName;         // SqlSessionTemplate Bean 名称
    private Class<? extends Annotation> annotationClass; // 只扫描有指定注解的接口（如 @Mapper）
    private Class<?> markerInterface;                  // 只扫描指定父接口

    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
        // 1. 创建 ClassPathMapperScanner
        ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
        scanner.setSqlSessionFactoryBeanName(this.sqlSessionFactoryBeanName);
        scanner.setSqlSessionTemplateBeanName(this.sqlSessionTemplateBeanName);
        scanner.setAnnotationClass(this.annotationClass);
        scanner.setMarkerInterface(this.markerInterface);

        // 2. 扫描指定包
        scanner.scan(StringUtils.tokenizeToStringArray(this.basePackage,
            ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
    }
}

/**
 * ClassPathMapperScanner — 实际执行扫描的类
 * 继承自 Spring 的 ClassPathBeanDefinitionScanner
 */
class ClassPathMapperScanner extends ClassPathBeanDefinitionScanner {

    @Override
    protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
        Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

        if (beanDefinitions.isEmpty()) {
            logger.warn("No MyBatis mapper was found in '" + Arrays.toString(basePackages) + "' package.");
        } else {
            // 对每个扫描到的 Mapper 接口进行后处理
            processBeanDefinitions(beanDefinitions);
        }

        return beanDefinitions;
    }

    private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
        for (BeanDefinitionHolder holder : beanDefinitions) {
            GenericBeanDefinition definition = (GenericBeanDefinition) holder.getBeanDefinition();

            // ★ 将 Bean 的 class 改为 MapperFactoryBean ★
            // 这样 Spring 实例化时实际创建的是 MapperFactoryBean
            definition.getConstructorArgumentValues().addGenericArgumentValue(
                definition.getBeanClassName());
            definition.setBeanClass(MapperFactoryBean.class);

            // 注入 SqlSessionFactory
            definition.getPropertyValues().add("sqlSessionFactory",
                new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
        }
    }
}
```

# 7.Spring Boot 中使用 @MapperScan

```java
/**
 * @MapperScan 是 MyBatis-Spring 提供的注解
 * 等价于 MapperScannerConfigurer 的 XML 配置
 *
 * 通常放在 @SpringBootApplication 启动类上
 */

@SpringBootApplication
@MapperScan("com.test.mapper")  // ★ 扫描 Mapper 接口
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

// ======= Mapper 接口 =======
// 不需要添加任何额外注解（如果已经在 @MapperScan 的包范围内）

package com.test.mapper;

public interface CarMapper {
    @Select("select * from t_car where id = #{id}")
    Car selectById(Long id);
}

// ======= Service 层使用 =======
// 可以直接 @Autowired 注入 Mapper 接口

package com.test.service;

@Service
public class CarService {

    @Autowired  // ★ Spring 自动注入 Mapper 代理对象 ★
    private CarMapper carMapper;

    @Transactional  // ★ Spring 事务管理 ★
    public void transfer(String fromAct, String toAct, Double money) {
        // 同一事务中多次调用 carMapper 方法
        // 底层共享同一个 SqlSession
        carMapper.updateBalance(fromAct, -money);
        carMapper.updateBalance(toAct, +money);
    }
}
```

# 8.Spring 事务管理整合原理

```java
/**
 * SpringManagedTransaction — Spring 事务管理的桥接
 * 源码：org.mybatis.spring.transaction.SpringManagedTransaction
 *
 * MyBatis 原生的两个 Transaction 实现：
 *   - JdbcTransaction：自己管理 commit/rollback/close
 *   - ManagedTransaction：交给容器管理
 *
 * SpringManagedTransaction 是第三种实现：
 *   - 获取 Connection 时：从 Spring 的 DataSourceUtils 获取
 *     （会返回当前线程绑定的 Connection，保证同一事务同一连接）
 *   - commit()：不执行（由 Spring 事务管理器统一提交）
 *   - rollback()：不执行（由 Spring 事务管理器统一回滚）
 *   - close()：不执行（由 Spring 事务管理器统一关闭）
 *   - 但会实际关闭非 Spring 管理的 Connection
 */

/**
 * Spring 与 MyBatis 事务整合的完整流程：
 *
 * 1. Service 层标注 @Transactional
 *    → Spring 开启事务，绑定 Connection 到当前线程
 *
 * 2. 调用 carMapper.selectById(1)
 *    → SqlSessionTemplate.sqlSessionProxy.selectOne()
 *    → getSqlSession() → 从 TransactionSynchronizationManager 获取 SqlSession
 *    → 如果不存在 → 创建新的 SqlSession，使用 SpringManagedTransaction
 *    → SpringManagedTransaction 通过 DataSourceUtils.getConnection()
 *      获取当前事务的 Connection（同一连接）
 *
 * 3. 调用 carMapper.updateBalance(act, money)
 *    → 复用同一个 SqlSession（从 TransactionSynchronizationManager 获取）
 *    → 复用同一个 Connection
 *
 * 4. Service 方法正常返回
 *    → Spring 事务管理器执行 commit
 *    → TransactionSynchronizationManager 中的回调执行
 *    → SqlSession.commit() → Connection.commit()
 *    → SqlSession.close()
 *
 * 5. Service 方法抛出异常
 *    → Spring 事务管理器执行 rollback
 *    → TransactionSynchronizationManager 中的回调执行
 *    → Connection.rollback()
 *    → SqlSession.close()
 */
```

# 9.整合后的完整数据流

```
  @Service
  public class TransferService {
      @Autowired private AccountMapper mapper;

      @Transactional
      public void transfer(...) {
          Account from = mapper.selectByActno(fromAct);  // (1)
          Account to   = mapper.selectByActno(toAct);    // (2)
          // 更新操作
          mapper.update(from);                            // (3)
          mapper.update(to);                              // (4)
      }
  }

  ┌─────────────────────────────────────────────────────────────────┐
  │                        执行流程分析                               │
  ├─────────────────────────────────────────────────────────────────┤
  │                                                                 │
  │  (1) mapper.selectByActno(fromAct)                              │
  │      → SqlSessionTemplate.selectOne()                           │
  │      → sqlSessionProxy.selectOne()  [jdk proxy]                 │
  │      → SqlSessionInterceptor.invoke()                            │
  │        → getSqlSession()                                         │
  │          → 检查 TransactionSynchronizationManager                │
  │          → 发现有事务 → 创建 SqlSessionHolder                     │
  │          → 注册到 ThreadLocal                                    │
  │          → 创建 SqlSession (executor = CachingExecutor           │
  │                              wrapped with SpringManagedTransaction)│
  │        → sqlSession.selectOne() [new DefaultSqlSession]         │
  │        → ... (正常的 MyBatis 执行流程)                            │
  │                                                                 │
  │  (2) mapper.selectByActno(toAct)                                │
  │      → ... 同上 ...                                             │
  │      → getSqlSession()                                          │
  │        → TransactionSynchronizationManager 中发现已有            │
  │        → ★ 复用同一个 SqlSession ★                               │
  │        → ★ 复用同一个 Connection ★                               │
  │                                                                 │
  │  (3)(4) 更新操作同理，复用同一个 SqlSession 和 Connection            │
  │                                                                 │
  └─────────────────────────────────────────────────────────────────┘
```

# 10.常见问题与解决

```
问题 1：Mapper 接口注入失败（NoSuchBeanDefinitionException）

  原因：@MapperScan 没有扫描到 Mapper 接口所在包
  解决：
    - 检查 @MapperScan 的 basePackages 是否正确
    - 或者直接在 Mapper 接口上添加 @Mapper 注解（Spring Boot）
    - 推荐用 @MapperScan("com.xxx.xxx.mapper") 批量扫描

问题 2：事务不生效

  原因：
    - Service 方法没有被 Spring AOP 代理
    - 不是通过 Spring 容器获取的 Bean
    - @Transactional 标注在 private 方法上
    - 同类内部调用（this.method()）不会走代理
    - 异常被捕获并吞掉了
  解决：
    - 确保通过 @Autowired 注入的 Bean 调用
    - @Transactional 放在 public 方法上
    - 不要吞掉异常，或者在 catch 中手动回滚

问题 3：同一事务中无法获取自增主键

  原因：BatchExecutor 模式下需要 flushStatements 才能获取
  解决：使用默认的 SimpleExecutor，或者在批量后手动 flush

问题 4：一级缓存数据不一致

  原因：在 Spring 事务内，同一个 SqlSession 的一级缓存可能返回旧数据
  解决：
    - 设置 localCacheScope=STATEMENT（关闭一级缓存）
    - 或手动 sqlSession.clearCache()

问题 5：mapperLocations 配置了但不生效

  原因：
    - 路径配置错误
    - mapperLocations 和 mybatis-config.xml 中的 mappers 冲突
  解决：建议在 Spring Boot 中只用 mybatis.mapper-locations 配置
```

# 11.MyBatis-Spring Boot Starter 自动配置原理

```java
/**
 * MyBatisAutoConfiguration — Spring Boot 自动配置类
 * 源码：org.mybatis.spring.boot.autoconfigure.MybatisAutoConfiguration
 *
 * 这个类通过 Spring Boot 的自动配置机制，
 * 根据 application.yml 中的配置自动创建 SqlSessionFactory 和 SqlSessionTemplate
 */
@Configuration
@ConditionalOnClass({SqlSessionFactory.class, SqlSessionFactoryBean.class})
@ConditionalOnSingleCandidate(DataSource.class)
@EnableConfigurationProperties(MybatisProperties.class)
@AutoConfigureAfter({DataSourceAutoConfiguration.class, MybatisLanguageDriverAutoConfiguration.class})
public class MybatisAutoConfiguration implements InitializingBean {

    private final MybatisProperties properties;
    private final Interceptor[] interceptors;
    private final TypeHandler[] typeHandlers;
    private final LanguageDriver[] languageDrivers;
    private final ResourceLoader resourceLoader;
    private final DatabaseIdProvider databaseIdProvider;
    private final List<ConfigurationCustomizer> configurationCustomizers;

    // ★ 自动创建 SqlSessionFactory Bean ★
    @Bean
    @ConditionalOnMissingBean
    public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
        SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
        factory.setDataSource(dataSource);

        // 设置各种配置
        factory.setMapperLocations(
            this.properties.resolveMapperLocations());  // 从 mybatis.mapper-locations 读取
        factory.setTypeAliasesPackage(
            this.properties.getTypeAliasesPackage());   // 从 mybatis.type-aliases-package 读取
        factory.setTypeHandlersPackage(
            this.properties.getTypeHandlersPackage());
        factory.setPlugins(this.interceptors);

        // 应用 ConfigurationCustomizer（用户自定义配置）
        applyConfiguration(factory);

        return factory.getObject();
    }

    // ★ 自动创建 SqlSessionTemplate Bean ★
    @Bean
    @ConditionalOnMissingBean
    public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
        ExecutorType executorType = this.properties.getExecutorType();
        if (executorType != null) {
            return new SqlSessionTemplate(sqlSessionFactory, executorType);
        }
        return new SqlSessionTemplate(sqlSessionFactory);
    }
}
```

# 12.手动配置 vs Spring Boot 自动配置

```
┌──────────────────────┬─────────────────────────┬───────────────────────────────┐
│      配置方式          │        手动（XML）        │     Spring Boot（自动配置）     │
├──────────────────────┼─────────────────────────┼───────────────────────────────┤
│  SqlSessionFactory    │ SqlSessionFactoryBean    │ 自动创建 + mybatis.* 配置      │
│  SqlSessionTemplate   │ 需要手动配置              │ 自动创建                       │
│  Mapper 扫描          │ MapperScannerConfigurer  │ @MapperScan 或 @Mapper       │
│  事务管理              │ DataSourceTransactionManager │ 自动配置                   │
│  数据源               │ 需要手动配置              │ 自动配置（spring.datasource.*）│
│  插件注册             │ <bean> + <property>      │ @Bean Interceptor 自动注册    │
│  配置复杂度            │ 较高                     │ 较低                          │
│  灵活性               │ 高                       │ 高（可通过 Java Config 调整）   │
│  推荐场景             │ 老项目迁移                │ 新项目（推荐）                  │
└──────────────────────┴─────────────────────────┴───────────────────────────────┘
```
