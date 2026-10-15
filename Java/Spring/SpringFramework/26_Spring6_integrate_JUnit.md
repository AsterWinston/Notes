# 1. Spring 6 集成 JUnit
spring底层有一套API，这套API专门支持JUnit
# 1. Spring 对 JUnit4 的支持
```
Spring提供的方便主要是这几个注解：
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(“classpath:spring.xml”)
在单元测试类上使用这两个注解之后，在单元测试类中的属性上可以使用@Autowired。比较方便。

```
## 1.1 依赖
```
<dependencies>
  <!-- spring核心依赖 -->
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>6.0.8</version>
  </dependency>
  <!-- spring对JUnit的支持的依赖 -->
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-test</artifactId>
    <!-- 这个版本支持JUnit4也支持JUnit5 -->
    <version>6.0.8</version>
  </dependency>
  <!-- junit 4 -->
  <dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.13.2</version>
    <scope>test</scope>
  </dependency>
</dependencies>
```
## 1.2 实体类
```
package cw.study.spring.bean;

import org.springframework.beans.factory.annotation.Value;

/**
 * ClassName: User
 * Package: cw.study.spring.bean
 * Description:
 *
 * @Author tcw
 * @Create 2023-06-05 12:18
 * @Version 1.0
 */
@Component
public class User {
    @Value("张三")
    private String name;
    
    public User() {
    }
    
    public User(String name) {
        this.name = name;
    }
    
    public String getName() {
        return name;
    }
    
    public void setName(String name) {
        this.name = name;
    }
    
    @Override
    public String toString() {
        return "User{" + "name='" + name + '\'' + '}';
    }
}

```
## 1.3 配置文件spring.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
						   http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">

	<!-- 组件扫描 -->
	<context:component-scan base-package="cw.study.spring.bean"/>
	
</beans>

```
## 1.4 测试程序
```
import cw.study.spring.bean.User;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

/**
 * ClassName: TestJUnit4
 * Package: PACKAGE_NAME
 * Description:
 *
 * @Author tcw
 * @Create 2023-06-05 12:19
 * @Version 1.0
 */
@RunWith(SpringJUnit4ClassRunner.class) // JUnit4中的注解
// 使用注解的方式为测试类加载配置文件创建spring容器
@ContextConfiguration("classpath:spring.xml")
// 如果使用配置类代替配置文件，注解用这个@ContextConfiguration(classes = Spring6Config.class)
public class TestJUnit4 {

    @Autowired // 自动为属性注入
    private User user;
    
    @Test
    public void test01() {
        // 测试方法中可以直接使用spring容器中的组件，不用在创建容器获取组件
        System.out.println(user);
    }
       
}

```

# 2. Spring对Junit5的支持
## 2.1 依赖
```
引入JUnit5的依赖，Spring对JUnit支持的依赖还是：spring-test
<dependencies>
  <!-- spring核心依赖 -->
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>6.0.8</version>
  </dependency>
  <!-- spring对JUnit的支持的依赖 -->
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-test</artifactId>
    <!-- 这个版本支持JUnit4也支持JUnit5 -->
    <version>6.0.8</version>
  </dependency>
  <!--junit5依赖-->
  <dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.9.0</version>
    <scope>test</scope>
  </dependency>
</dependencies>

```
## 2.2 实体类
```
package cw.study.spring.bean;

import org.springframework.beans.factory.annotation.Value;

/**
 * ClassName: User
 * Package: cw.study.spring.bean
 * Description:
 *
 * @Author tcw
 * @Create 2023-06-05 12:18
 * @Version 1.0
 */
@Component
public class User {
    @Value("张三")
    private String name;
    
    public User() {
    }
    
    public User(String name) {
        this.name = name;
    }
    
    public String getName() {
        return name;
    }
    
    public void setName(String name) {
        this.name = name;
    }
    
    @Override
    public String toString() {
        return "User{" + "name='" + name + '\'' + '}';
    }
}

```
## 2.3 spring.xml配置文件
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
						   http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">

	<!-- 组件扫描 -->
	<context:component-scan base-package="cw.study.spring.bean"/>
	
</beans>

```
## 2.4 测试程序
```
如果使用配置类代替配置文件，注解用这个@ContextConfiguration(classes = Spring6Config.class)

import cw.study.spring.bean.User;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit.jupiter.SpringExtension;

/**
 * ClassName: TestJUnit5
 * Package: PACKAGE_NAME
 * Description:
 *
 * @Author tcw
 * @Create 2023-06-05 12:31
 * @Version 1.0
 */
@ExtendWith(SpringExtension.class) // JUnit5中的注解
// 使用注解的方式为测试类加载配置文件创建spring容器
@ContextConfiguration("classpath:spring.xml")
public class TestJUnit5 {
    
        @Autowired // 自动为属性注入
        private User user;

        @Test
        public void test01() {
            // 测试方法中可以直接使用spring容器中的组件，不用在创建容器获取组件
            System.out.println(user);
        }
    
}


```

# 3. 源码解析：Spring TestContext 框架机制

## 3.1 SpringJUnit4ClassRunner — JUnit4 集成核心

```
@RunWith(SpringJUnit4ClassRunner.class) 替换了 JUnit 默认的 Runner：

public class SpringJUnit4ClassRunner extends BlockJUnit4ClassRunner {

    private final TestContextManager testContextManager;

    public SpringJUnit4ClassRunner(Class<?> clazz) throws InitializationError {
        super(clazz);
        if (clazz == null) {
            throw new IllegalArgumentException("Class must not be null");
        }
        // ★ 创建 TestContextManager
        this.testContextManager = createTestContextManager(clazz);
    }

    // 重写 createTest() → Spring 不通过无参构造创建测试实例
    // 而是从 ApplicationContext 中获取（如果需要的话）
    @Override
    protected Object createTest() throws Exception {
        Object testInstance = super.createTest();         // 无参构造
        // ★ 注入依赖（@Autowired 等）
        getTestContextManager().prepareTestInstance(testInstance);
        return testInstance;
    }
}

TestContextManager.prepareTestInstance():
  → 遍历所有 TestExecutionListener
  → 调用 listener.prepareTestInstance(context)
  → DependencyInjectionTestExecutionListener:
      利用 AutowiredAnnotationBeanPostProcessor 处理 @Autowired 注入
```

## 3.2 SpringExtension — JUnit5 集成核心

```
@ExtendWith(SpringExtension.class) 是 JUnit5 的扩展机制：

public class SpringExtension implements BeforeAllCallback, AfterAllCallback,
        TestInstancePostProcessor, BeforeEachCallback, AfterEachCallback,
        BeforeTestExecutionCallback, AfterTestExecutionCallback, ParameterResolver {

    @Override
    public void postProcessTestInstance(Object testInstance, ExtensionContext context)
            throws Exception {
        // ★ 获取 TestContextManager 并准备测试实例
        getTestContextManager(context).prepareTestInstance(testInstance);
    }

    private TestContextManager getTestContextManager(ExtensionContext context) {
        // 从 Store 中获取或创建 TestContextManager
        Store store = getStore(context);
        return store.getOrComputeIfAbsent(..., TestContextManager.class);
    }
}
```

## 3.3 TestContextManager — 测试上下文管理器

```
TestContextManager 是整个 Spring Test 框架的核心调度器：

public class TestContextManager {

    private final TestContext testContext;
    private final List<TestExecutionListener> testExecutionListeners;

    public TestContextManager(Class<?> testClass) {
        // ★ 解析 @ContextConfiguration、@ActiveProfiles、@TestPropertySource 等
        this.testContext = new TestContext(testClass);

        // ★ 注册默认的 TestExecutionListener 列表
        this.testExecutionListeners = new ArrayList<>(Arrays.asList(
            new ServletTestExecutionListener(),       // Servlet API mock
            new DirtiesContextBeforeModesTestExecutionListener(),
            new ApplicationEventsTestExecutionListener(),
            new DependencyInjectionTestExecutionListener(),  // ★ @Autowired 注入
            new DirtiesContextTestExecutionListener(),
            new TransactionalTestExecutionListener(), // ★ @Transactional 测试回滚
            new SqlScriptsTestExecutionListener(),    // @Sql 脚本执行
            new EventPublishingTestExecutionListener()
        ));
    }

    public void prepareTestInstance(Object testInstance) {
        for (TestExecutionListener listener : getTestExecutionListeners()) {
            listener.prepareTestInstance(this.testContext);
        }
    }

    // 在 JUnit5 中调用：
    public void beforeTestMethod(Object testInstance, Method testMethod) {
        for (TestExecutionListener listener : getTestExecutionListeners()) {
            listener.beforeTestMethod(this.testContext);
        }
    }
}
```

## 3.4 @ContextConfiguration → ApplicationContext 的创建

```
@ContextConfiguration("classpath:spring.xml")
    │
    ▼
TestContext 创建时解析该注解：
    │
    ├── 读取 locations = ["classpath:spring.xml"]（或 classes = {Spring6Config.class}）
    │
    └── 调用 ContextLoader.loadContext(mergedConfig)

加载策略（由 @ContextConfiguration(loader=...) 指定，默认 DelegatingSmartContextLoader）：
    1. 如果只有 locations（XML 路径）→ GenericXmlContextLoader → ClassPathXmlApplicationContext
    2. 如果只有 classes（配置类）→ AnnotationConfigContextLoader → AnnotationConfigApplicationContext
    3. 两者都有 → 两者都加载，合并

★ 关键优化：ApplicationContext 被缓存！
   同一个测试类层级的多个测试方法共享同一个 ApplicationContext，
   避免重复创建容器。
   缓存键 = mergedConfig（locations + classes + activeProfiles + ...）
```

## 3.5 DependencyInjectionTestExecutionListener — @Autowired 注入

```
DependencyInjectionTestExecutionListener 负责将 Spring Bean 注入到测试实例：

public class DependencyInjectionTestExecutionListener
        extends AbstractTestExecutionListener {

    @Override
    public void prepareTestInstance(TestContext testContext) throws Exception {
        if (shouldInjectDependencies(testContext)) {
            // ★ 核心：将测试实例注入到 ApplicationContext 中
            injectDependencies(testContext);
        }
    }

    protected void injectDependencies(TestContext testContext) throws Exception {
        Object bean = testContext.getTestInstance();
        Class<?> clazz = testContext.getTestClass();
        AutowireCapableBeanFactory beanFactory =
                testContext.getApplicationContext().getAutowireCapableBeanFactory();

        // ★★ 通过 AutowireCapableBeanFactory 处理 @Autowired/@Value 注入 ★★
        beanFactory.autowireBeanProperties(bean, AutowireCapableBeanFactory.AUTOWIRE_NO, false);
        // autowireBeanProperties → 调用 AutowiredAnnotationBeanPostProcessor
        // → 扫描 @Autowired/@Value 注解 → 从容器解析依赖 → 反射注入

        // 处理 BeanFactoryAware、ApplicationContextAware 等 Aware 接口
        beanFactory.initializeBean(bean, clazz.getName() + AUTOWIRED_SUFFIX);
    }
}
```

## 3.6 TransactionalTestExecutionListener — 测试事务自动回滚

```
@Transactional 在测试类上的行为和业务代码中不同：
  ★ 测试中的 @Transactional 默认会在测试完成后自动回滚！

public class TransactionalTestExecutionListener extends AbstractTestExecutionListener {

    @Override
    public void beforeTestMethod(TestContext testContext) throws Exception {
        // ★ 如果测试方法上有 @Transactional，在方法执行前开启事务
        TransactionContext txContext = testContext.getTransactionContext();
        if (txContext != null) {
            txContext.startTransaction();
        }
    }

    @Override
    public void afterTestMethod(TestContext testContext) throws Exception {
        // ★ 方法执行完成后，标记事务回滚（默认行为）
        TransactionContext txContext = testContext.getTransactionContext();
        if (txContext != null) {
            // 默认 rollback=true，可以通过 @Rollback(false) 或 @Commit 覆盖
            txContext.endTransaction();
        }
    }
}

★ 这个特性使测试不会污染数据库：
   testInsert() {
       userService.insert(new User("test"));  // 插入成功
   }
   // ★ 方法执行完后自动回滚，数据库中不会留下测试数据！
```

## 3.7 JUnit4 vs JUnit5 的 Spring 集成差异

```
┌──────────────────┬──────────────────────────┬──────────────────────────┐
│ 特性             │ JUnit4                   │ JUnit5                   │
├──────────────────┼──────────────────────────┼──────────────────────────┤
│ Runner/Extension  │ @RunWith(SpringJUnit4    │ @ExtendWith(Spring       │
│                  │ ClassRunner.class)        │ Extension.class)         │
├──────────────────┼──────────────────────────┼──────────────────────────┤
│ 测试实例创建       │ Runner.createTest()      │ TestInstancePostProcessor│
│                  │                          │ .postProcessTestInstance()│
├──────────────────┼──────────────────────────┼──────────────────────────┤
│ @Autowired 注入   │ DependencyInjection      │ 相同                      │
│                  │ TestExecutionListener     │                           │
├──────────────────┼──────────────────────────┼──────────────────────────┤
│ 参数解析          │ 不支持                    │ ParameterResolver        │
│                  │                          │ 支持 @Autowired 参数注入   │
├──────────────────┼──────────────────────────┼──────────────────────────┤
│ 并发测试          │ 有限支持                  │ @Execution(CONCURRENT)    │
│                  │                          │ 更好的并发支持             │
└──────────────────┴──────────────────────────┴──────────────────────────┘

JUnit5 ParameterResolver 示例：
@Test
public void test01(@Autowired User user) {
    // JUnit5 支持直接在测试方法参数中注入 Spring Bean
    System.out.println(user);
}
```

## 3.8 设计思想总结

1. 监听器模式（TestExecutionListener）：
   Spring Test 框架采用了监听器/观察者模式，每个 TestExecutionListener
   负责一个关注的测试生命周期阶段。新增功能只需实现新的 Listener，符合 OCP 原则。

2. ApplicationContext 缓存：
   测试方法之间共享 ApplicationContext，避免了重复初始化容器的开销。
   这是 Spring 测试框架性能优化的关键设计。

3. 事务自动回滚：
   测试中的 @Transactional 默认回滚是 Spring Test 最有价值的特性之一，
   保证了测试数据的隔离性，使测试可以反复运行而不污染数据库。

4. 与 Spring IoC 的深度集成：
   测试实例本质上被当作一个特殊的 Bean，通过 AutowireCapableBeanFactory
   获得与其他 Bean 完全一致的依赖注入支持。
