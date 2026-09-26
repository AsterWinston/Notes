# 1. JUnit 4 生命周期概览

JUnit 4 的生命周期围绕测试类与测试方法展开，默认每个测试方法都会创建新的测试类实例。

## 1.1 生命周期顺序

1. `@BeforeClass`
2. `@Before`
3. `@Test`
4. `@After`
5. `@AfterClass`

**注意**：`@BeforeClass/@AfterClass` 必须是 `static` 方法。

# 2. 常用注解详解

## 2.1 `@Test`

- 标记测试方法。
- 可设置 `expected` 与 `timeout`（不推荐作为异常断言的首选方式）。

```java
@Test(expected = IllegalArgumentException.class, timeout = 100)
public void should_throw_exception() { }
```

## 2.2 `@Before` / `@After`

用于每个测试方法的前置与后置处理，例如数据初始化、资源释放。

## 2.3 `@BeforeClass` / `@AfterClass`

用于类级别的初始化和清理，例如启动嵌入式数据库或共享资源。

## 2.4 `@Ignore`

- 跳过测试方法或测试类。
- 建议给出原因，避免“永久忽略”。

```java
@Ignore("等待外部依赖稳定")
@Test
public void flaky_test() { }
```

# 3. 测试执行顺序

## 3.1 默认顺序

JUnit 4 不保证方法执行顺序，依赖顺序会导致测试不稳定。

## 3.2 `@FixMethodOrder`

可通过方法名顺序控制执行：

```java
@FixMethodOrder(MethodSorters.NAME_ASCENDING)
public class OrderedTest { }
```

**建议**：只在无法避免的情况下使用，不作为常规策略。

# 4. 典型示例

```java
public class UserServiceTest {

    private UserService service;

    @BeforeClass
    public static void beforeClass() {
        // 类级别资源初始化
    }

    @Before
    public void setUp() {
        service = new UserService();
    }

    @Test
    public void create_user_should_success() {
        User user = service.create("alice");
        assertNotNull(user.getId());
    }

    @After
    public void tearDown() {
        // 每次测试后的清理
    }

    @AfterClass
    public static void afterClass() {
        // 类级别资源释放
    }
}
```

# 5. 常见问题与建议

## 5.1 静态资源的副作用

`@BeforeClass` 创建的资源需要保证线程安全和可复用，避免污染后续测试。

## 5.2 与依赖注入框架的协作

JUnit 4 本身不提供 DI，通常通过 Spring Test 或自定义 Runner 实现。
