# 1. 知识点
- **JUnit5组成**: JUnit Platform(启动测试框架的基础平台)、JUnit Jupiter(新的编程模型和扩展模型)、JUnit Vintage(兼容JUnit3/4的测试引擎)
- **JUnit Platform**: 提供TestEngine API，是JVM上启动测试框架的基础。IDE和构建工具(Maven/Gradle)通过Launcher发现和执行测试
- **TestEngine**: 测试引擎接口，JUnit Jupiter和JUnit Vintage都实现了此接口，Platform可同时运行多个引擎
- **@Test**: 标记测试方法，使用JUnit Jupiter的org.junit.jupiter.api.Test
- **@BeforeAll**: 静态方法，在所有测试方法前执行一次(需static)。用于初始化重量级资源(数据库连接等)
- **@AfterAll**: 静态方法，在所有测试方法后执行一次(需static)。用于清理重量级资源
- **@BeforeEach**: 实例方法，每个测试方法前执行。用于测试前的公共初始化(如创建测试对象)
- **@AfterEach**: 实例方法，每个测试方法后执行。用于清理(如重置Mock)
- **@DisplayName**: 为测试类或测试方法指定可读性更强的显示名称，支持emoji和空格
- **@Nested**: 嵌套测试类，组织层级测试结构，内部类可访问外部类的成员(非静态内部类)
- **@Tag**: 给测试方法/类打标签，支持过滤执行。如@Tag("slow")，mvn test -Dgroups=slow只执行slow标签的测试
- **@Disabled**: 禁用测试方法/类，等效于JUnit4的@Ignore。可加原因说明
- **@ParameterizedTest**: 参数化测试，比普通@Test支持参数注入。配合参数源注解使用
- **@ValueSource**: 提供简单类型参数(字符串/整数/布尔等)。@ValueSource(ints = {1, 2, 3})
- **@CsvSource**: 提供CSV格式参数，支持多列。@CsvSource({"apple, 1", "banana, 2"})
- **@MethodSource**: 引用静态方法提供参数(返回Stream/Iterable)。方法名默认与测试方法名相同
- **@EnumSource**: 提供枚举类型的所有值作为参数。@EnumSource(value = TimeUnit.class, names = {"DAYS", "HOURS"})
- **assertEquals(expected, actual)**: 断言相等。数组比较用assertArrayEquals
- **assertTrue/assertFalse**: 断言布尔条件
- **assertNull/assertNotNull**: 断言空/非空
- **assertThrows**: 断言执行代码抛出指定异常。assertThrows(IllegalArgumentException.class, () -> method(null))
- **assertAll**: 分组断言，所有断言都会执行(不短路)，一次性报告所有失败
- **assertTimeout**: 断言代码在指定时间内完成
- **Mockito概述**: Java单元测试Mock框架，模拟依赖对象的行为，隔离测试目标
- **@Mock**: 创建Mock对象(不调用真实方法)。@Mock private UserRepository userRepository;
- **@InjectMocks**: 创建被测试对象，并自动将@Mock/@Spy标记的依赖注入到被测试对象中(通过构造器/setter/字段)
- **@Spy**: 创建部分Mock对象，默认调用真实方法，可针对特定方法模拟(stub)
- **@Captor**: 创建ArgumentCaptor，用于捕获传递给Mock方法的参数进行断言
- **when().thenReturn()**: 为Mock对象的方法调用指定返回值
- **when().thenThrow()**: 为Mock对象的方法调用指定抛出异常
- **doThrow()**: 用于void方法抛异常，doThrow(new RuntimeException()).when(mock).voidMethod()
- **doNothing()**: void方法不执行任何操作(默认行为)
- **verify()**: 验证Mock对象的方法是否被调用及调用次数
- **verify调用次数**: times(n)精确n次、never()0次、atLeastOnce()至少1次、atLeast(n)至少n次、atMost(n)最多n次
- **参数匹配器(Matchers)**: any()任意值、anyInt()、anyString()、any(Class.class)、eq(value)精准匹配、argThat(predicate)自定义匹配
- **ArgumentCaptor**: 捕获传递给Mock方法的参数用于后续断言。ArgumentCaptor.forClass(User.class)创建，verify后captor.getValue()获取
- **spy部分Mock**: @Spy标注的对象默认调用真实方法，可通过doReturn(value).when(spy).method()部分mock
- **@SpringBootTest**: Spring Boot集成测试注解，启动完整的Spring容器。webEnvironment=RANDOM_PORT随机端口启动Web容器
- **@MockBean**: Spring Boot集成测试中，创建Mock对象并替换Spring容器中的Bean
- **@SpyBean**: Spring Boot集成测试中，创建Spy对象并替换Spring容器中的Bean
- **TestRestTemplate**: Spring Boot集成测试中用于HTTP请求的RestTemplate变体，自动指向测试Server
- **Testcontainers**: Java集成测试库，通过Docker运行临时依赖(MySQL/Redis/Kafka等)，测试结束自动销毁容器
- **Testcontainers使用**: @Testcontainers + @Container static GenericContainer<?> redis = new GenericContainer<>("redis:7") + 动态配置spring.datasource.url
- **代码覆盖率JaCoCo**: Java Code Coverage库，统计行覆盖率、分支覆盖率、方法覆盖率。Maven插件jacoco-maven-plugin
- **JaCoCo Maven配置**: pom.xml配置jacoco插件，mvn test后mvn jacoco:report生成HTML报告，可设置覆盖率阈值
- **TDD测试驱动开发**: Red-Green-Refactor循环。先写失败的测试(Red) → 写最简代码让测试通过(Green) → 重构(Refactor)。迫使先思考接口设计
- **测试覆盖率理想值**: 核心业务逻辑70-80%，接口关键路径100%。不追求100%(性价比低，POJO/配置类不适合单元测试)

# 2. 面试题

## 2.1 JUnit5和JUnit4有什么区别？
| 维度 | JUnit4 | JUnit5 |
|------|--------|--------|
| 包名 | org.junit | org.junit.jupiter |
| 初始化 | @BeforeClass / @Before | @BeforeAll(static) / @BeforeEach |
| 清理 | @AfterClass / @After | @AfterAll(static) / @AfterEach |
| 忽略 | @Ignore | @Disabled |
| 参数化测试 | @RunWith(Parameterized.class) | @ParameterizedTest + @ValueSource等 |
| 嵌套测试 | 不支持 | @Nested |
| 标签 | @Category | @Tag |
| 扩展机制 | Runner/Rule | Extension API(更强大) |
| Java版本 | Java 5+ | Java 8+ |
| 架构 | 单体 | 模块化(Platform/Jupiter/Vintage) |

## 2.2 @Mock和@InjectMocks有什么区别？
@Mock：创建Mock对象，其所有方法默认返回null(或空集合)/什么都不做。Mock只模拟依赖，不代表被测试对象。

@InjectMocks：创建被测试类的实例，并将所有@Mock(和@Spy)标注的Mock对象自动注入到该实例中。注入方式：构造器注入优先、Setter注入、字段注入。

```java
@Mock
private UserRepository userRepository;
@InjectMocks
private UserService userService; // userRepository自动注入
```

## 2.3 Mockito中when().thenReturn()和doReturn().when()有什么区别？
when(mock.method()).thenReturn(value)是类型安全的形式，会实际调用mock.method()一次(返回null)。对void方法不能用(编译错误)。对spy对象的方法会真实调用(副作用)，应改用doReturn。

doReturn(value).when(mock).method()不调用实际方法，直接设置返回值。适用于：1)Spy对象(避免真实调用) 2)void方法 3)抛异常doThrow().when()。

推荐：对Mock对象用when/thenReturn，对Spy对象用doReturn/when。

## 2.4 @Spy和@Mock有什么区别？
@Mock：完全模拟对象，所有方法默认不执行实际逻辑。返回类型默认返回null/空集合/0。适合完全隔离测试。

@Spy：部分模拟对象，默认调用真实方法，只有显式stub的方法才模拟。适合只想mock部分方法的场景。

```java
@Spy
private List<String> spyList = new ArrayList<>();
// spyList.add("one") 真实执行
// when(spyList.size()).thenReturn(100) 模拟返回值
```

## 2.5 ArgumentCaptor有什么作用？如何使用？
ArgumentCaptor用于捕获传递给Mock方法的参数，对参数进行细粒度断言。适用场景：方法内部构造对象作为参数传递(无法在外部直接断言)。

```java
@Mock private EmailService emailService;
@Captor private ArgumentCaptor<Email> emailCaptor;

@Test
void testSendEmail() {
    userService.register(new User("jack", "jack@example.com"));
    verify(emailService).send(emailCaptor.capture());
    Email sent = emailCaptor.getValue();
    assertEquals("jack@example.com", sent.getTo());
}
```

## 2.6 Spring Boot集成测试中@MockBean和@Mock有什么区别？
@Mock是Mockito原生注解，需要配合MockitoAnnotations.openMocks(this)或@ExtendWith(MockitoExtension.class)使用。只初始化Mock对象，Spring容器无感知，不会替换Spring容器中的Bean。

@MockBean是Spring Boot Test提供的注解，会替换Spring容器中同类型的Bean为Mock对象。若容器中有多个相同类型Bean，需配合@Qualifier或按name指定。

```java
@SpringBootTest
class UserServiceIntegrationTest {
    @MockBean  // 替换Spring容器中的UserRepository
    private UserRepository userRepository;
    @Autowired
    private UserService userService;  // 使用的是被替换的MockBean
}
```

## 2.7 什么是Testcontainers？如何在Spring Boot集成测试中使用？
Testcontainers是一个Java库，在Docker容器中运行真实的中间件(MySQL、Redis、Kafka等)，测试结束自动销毁。比H2内存数据库更接近生产环境。

```java
@SpringBootTest
@Testcontainers
class OrderServiceTest {
    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
            .withDatabaseName("testdb");
    
    @DynamicPropertySource
    static void setProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
    }
}
```

## 2.8 JaCoCo代码覆盖率如何集成到Maven项目？
```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.10</version>
    <executions>
        <execution>
            <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals><goal>report</goal></goals>
        </execution>
        <execution>
            <id>check</id>
            <goals><goal>check</goal></goals>
            <configuration>
                <rules>
                    <rule>
                        <element>CLASS</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.70</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```
执行mvn test自动生成覆盖率报告在target/site/jacoco/index.html。低于70%行覆盖率构建失败。

## 2.9 JUnit5参数化测试有哪些方式？各有什么适用场景？
1) @ValueSource：单个简单参数，如ints/strings/booleans。适合单参数简单测试
2) @CsvSource：多列CSV参数。适合多参数组合测试(输入+期望输出)
3) @MethodSource：方法提供Stream/Iterable参数。适合复杂参数构建(如对象列表)
4) @EnumSource：枚举所有值。适合枚举类型测试
5) @ArgumentsSource：自定义参数提供器。适合需要复用参数源的场景
6) @NullSource/@EmptySource/@NullAndEmptySource：空值参数，配合@ValueSource使用

```java
@ParameterizedTest
@CsvSource({"1, 2, 3", "4, 5, 9"})
void testAdd(int a, int b, int expected) {
    assertEquals(expected, calculator.add(a, b));
}
```

## 2.10 单元测试中如何Mock静态方法？
Mockito 3.4+支持Mock静态方法(需要mockito-inline依赖)：

```java
@Test
void testStaticMethod() {
    try (MockedStatic<SomeUtil> mocked = mockStatic(SomeUtil.class)) {
        mocked.when(() -> SomeUtil.getCurrentTime()).thenReturn(1000L);
        // 执行业务代码
        mocked.verify(() -> SomeUtil.getCurrentTime(), times(1));
    }
    // try-with-resources自动关闭，静态Mock恢复
}
```
注意：静态Mock会影响全局，需用try-with-resources限制范围；并行测试需关注线程安全；尽量重构为实例方法避免静态Mock(依赖注入更好测试)。

## 2.11 什么是TDD？请描述TDD的实践流程
TDD(Test-Driven Development)测试驱动开发，核心循环：Red → Green → Refactor。

Red阶段：先写一个失败的单元测试(测试无法通过编译或断言失败)，定义期望的行为接口。
Green阶段：写最少的代码让测试通过(不追求完美代码，只要测试绿了就行)。
Refactor阶段：在测试保护下重构代码(优化命名、提取方法、去重)，确保测试仍然通过。

优点：1)强制思考接口设计 2)测试覆盖率高 3)代码更简洁(只写必要的) 4)重构有信心。缺点：前期投入大(适合核心领域逻辑，不适用UI/配置类)。在实际项目中，可对Service层核心逻辑做TDD。
