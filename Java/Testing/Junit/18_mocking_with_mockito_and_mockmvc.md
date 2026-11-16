# 1. Mockito 与 JUnit 5 的接线

Mockito 解决的是"**隔离**"问题：被测单元的正确性不应该取决于数据库是否在线、下游接口是否可用。它通过字节码生成（默认 ByteBuddy 子类化，5.x 对 final 类可切换 inline mock maker）造出"行为可编程的假对象"，让你专注验证被测对象自身的逻辑。**Mockito 与 JUnit 5 的集成点只有一个：`MockitoExtension`**，它负责在测试生命周期里初始化注解、注入 Mock、并在每个测试后校验 stubbing 的健康度。

## 1.1 MockitoExtension 与核心注解

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.Captor;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.anyLong;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class) // 等价于 JUnit 4 的 MockitoJUnitRunner
class UserServiceTest {

    @Mock
    private UserRepository userRepository;  // 生成 Mock，所有方法默认返回 null/0/空集合

    @Mock
    private MailClient mailClient;

    @InjectMocks
    private UserService userService;        // 真实对象，@Mock 按类型+名字注入其构造器/字段

    @Captor
    private ArgumentCaptor<MailMessage> mailCaptor; // 捕获实际传入参数的"录像机"

    @Test
    void should_send_welcome_mail_when_user_registered() {
        // Arrange：编程假对象的行为
        when(userRepository.save(any(User.class)))
                .thenAnswer(inv -> inv.getArgument(0));
        when(userRepository.findById(anyLong()))
                .thenReturn(Optional.of(new User(1L, "tom", "tom@ex.com")));

        // Act：调用被测对象
        userService.register(new User(1L, "tom", "tom@ex.com"));

        // Assert：验证交互 + 捕获参数细节
        verify(mailClient).send(mailCaptor.capture());
        assertThat(mailCaptor.getValue().getTo()).isEqualTo("tom@ex.com");
        assertThat(mailCaptor.getValue().getSubject()).contains("欢迎");
    }
}
```

**`@InjectMocks` 的注入策略**按顺序尝试：构造器注入（挑参数最多的构造器）→ setter 注入 → 字段注入。两个 `@Mock` 类型相同（如两个 `DataSource`）时按**字段名**匹配，这也是 Mock 字段名应与被测类中依赖名保持一致的原因。

> **注意**：`@InjectMocks` 创建的是**真实对象**，它不会递归 Mock 被测对象内部 new 出来的依赖。如果 `UserService` 内部 `new HttpClient()` 直连外部，Mockito 无能为力——这正是"可测性倒逼依赖注入设计"的含义。

## 1.2 不用 Extension 的替代方式

`MockitoExtension` 不是唯一入口。不想让测试类被注解绑定时可用 `MockitoAnnotations.openMocks(this)`（注意返回的 `AutoCloseable` 必须关闭，通常放 `@AfterEach`），或者干脆不用注解、直接 `mock()` 工厂方法：

```java
class PlainMockitoTest {

    private UserRepository repo = mock(UserRepository.class); // 工厂方法，无需任何 Extension
    private UserService service = new UserService(repo, mock(MailClient.class));

    @Test
    void should_return_user_name() {
        when(repo.findById(1L)).thenReturn(Optional.of(new User(1L, "tom", "tom@ex.com")));
        assertThat(service.userName(1L)).isEqualTo("tom");
    }
}
```

工厂方法风格在依赖只有一两个时更直白；注解风格在依赖多、需要 `@Captor`/`@Spy` 时更整洁。团队内统一一种即可。

# 2. Stubbing 与 Verify

## 2.1 行为编程：when/thenReturn 家族

Stubbing（打桩）就是"当假对象被这样调用时，返回那个结果"。核心 API 与适用场景：

```java
// 正常返回
when(repo.findById(1L)).thenReturn(Optional.of(user));

// 连续调用返回不同值（模拟状态变化）
when(counter.next()).thenReturn(1, 2, 3);   // 第一次 1，第二次 2，之后一直 3

// 抛异常（测试错误分支的标准手段）
when(repo.findById(99L)).thenThrow(new EntityNotFoundException("user 99"));

// 动态计算返回值（thenAnswer，能拿到调用现场的全部信息）
when(repo.save(any(User.class))).thenAnswer(inv -> {
    User u = inv.getArgument(0);
    u.setId(idSeq.incrementAndGet());      // 模拟数据库生成主键
    return u;
});

// void 方法不能 when().thenReturn()，用 doXxx 系列
doNothing().when(mailClient).send(any());
doThrow(new SmtpException()).when(mailClient).send(argThat(m -> m.getTo() == null));
doAnswer(inv -> { log.add(inv.getArgument(0)); return null; }).when(audit).record(any());
```

**参数匹配器（ArgumentMatchers）的规则只有一条容易踩坑：一个调用里要么全用字面量，要么全用匹配器**。`when(repo.findByIdAndName(1L, anyString()))` 会直接抛 `InvalidUseOfMatchersException`，必须写成 `eq(1L)`：

```java
when(repo.findByIdAndName(eq(1L), anyString())).thenReturn(Optional.of(user));
```

常用匹配器：`any()`（任意，含 null）、`anyString()/anyLong()/anyList()`（任意非 null 对应类型）、`eq(x)`（等值）、`argThat(predicate)`（自定义谓词）、`isNull()`。

## 2.2 交互验证：verify 家族

Stubbing 决定"假对象怎么表现"，verify 回答"被测对象有没有按约定使用假对象"：

```java
verify(repo).save(user);                    // 恰好一次（默认 times(1)）
verify(repo, times(3)).refreshCache();      // 恰好三次
verify(mailClient, never()).send(any());    // 从未调用——验证"不该发生的事没发生"
verify(repo, atLeastOnce()).findById(anyLong());
verifyNoMoreInteractions(repo);             // 除已 verify 外无其他交互（慎用，见 5.1）
```

> **行业实践**：`verify` 的正当用途是**验证有副作用且无返回值的交互**（发邮件、写审计、发消息），以及**验证不该发生的调用**（`never()`）。对"有返回值且结果可断言"的调用，优先断言业务结果而不是 verify 调用次数——后者把实现细节焊死在测试里，重构必碎。

## 2.3 BDD 风格

Mockito 内置 BDD 语义的别名 API，`given/willReturn` 读起来更像行为描述，与 AssertJ 的 `then(assertThat)` 组合后 Arrange-Act-Assert 三段一目了然：

```java
import static org.mockito.BDDMockito.given;
import static org.mockito.BDDMockito.then;
import static org.mockito.ArgumentMatchers.any;

@Test
void should_persist_user_when_register() {
    given(userRepository.save(any(User.class))).willReturn(savedUser); // Arrange

    userService.register(newUser);                                     // Act

    then(userRepository).should().save(newUser);                       // Assert（交互）
    then(mailClient).should(never()).send(any());                      // Assert（不发生）
}
```

功能与 `when/verify` 完全等价，选择纯属团队口味。

## 2.4 Strict Stubs：严格模式

`MockitoExtension` 默认开启 **STRICT_STUBS**，它在每个测试结束后做两项检查：

- **UnnecessaryStubbingException**：打了桩但整个测试从未走到——要么是测试路径写错了，要么是桩是复制粘贴来的死代码；
- **PotentialStubbingProblem**：桩定义时用了 `findById(1L)`，实际调用却是 `findById(2L)`——参数对不上导致 Mock 返回默认值 null，被测代码在 null 上继续跑，最终以一个莫名其妙的 NPE 失败。严格模式把失败点提前到"桩未命中"的现场，报错信息直接给出"定义的桩 vs 实际的调用"对比。

这是 Mockito 3 之后测试体验改善最大的一项特性。个别场景（参数化测试共享桩）确实需要放宽时，局部降级：

```java
@ExtendWith(MockitoExtension.class)
@MockitoSettings(strictness = Strictness.LENIENT) // 整类放宽（慎用）
class LegacyParamTest { }

// 或对单个桩放宽
lenient().when(repo.findById(anyLong())).thenReturn(Optional.of(user));
```

> **设计争议**：`lenient()` 该不该出现在代码库里？它本质是关掉严格模式的一盏灯。多数团队的立场是：**新代码禁止**，严格模式报出的 UnnecessaryStubbing 99% 是真实的测试坏味道（多余的 Arrange、错误的分支假设），应该修测试而不是关检查；存量代码迁移期允许临时使用并挂 TODO。

# 3. MockMvc 与 Web 层测试

MockMvc 在**不启动真实 Servlet 容器**的前提下模拟 HTTP 请求穿过 Spring MVC 的完整处理链（DispatcherServlet → HandlerMapping → 拦截器 → Controller → 参数解析/序列化 → 异常处理器）。它验证的是"URL、方法、参数绑定、状态码、响应体"这条 Web 契约，比直接调用 Controller 方法真实得多，又比启动端口的端到端测试便宜得多。

## 3.1 独立搭建 vs @WebMvcTest

MockMvc 有两种搭建方式，对应两种测试定位：

**独立搭建（standaloneSetup）**——纯 Mockito 世界，不启动任何 Spring 容器，Controller 手工 new、依赖手工 Mock：

```java
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;

import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@ExtendWith(MockitoExtension.class)
class UserControllerStandaloneTest {

    @Mock
    private UserService userService;

    private MockMvc mockMvc;

    @BeforeEach
    void setUp() {
        // 手工组装 Controller，不经过 Spring 容器
        mockMvc = MockMvcBuilders
                .standaloneSetup(new UserController(userService))
                .build(); // 需要拦截器/异常处理器时用 .addInterceptors()/.setControllerAdvice() 追加
    }

    @Test
    void should_return_404_when_user_absent() throws Exception {
        when(userService.findById(99L)).thenThrow(new EntityNotFoundException());

        mockMvc.perform(get("/users/99"))
               .andExpect(status().isNotFound());
    }
}
```

**容器搭建（@WebMvcTest）**——启动 Web 切片容器，自动装配真实的拦截器、`@ControllerAdvice`、Jackson 配置、安全过滤器链（见第 17 篇）。

| 维度 | standaloneSetup | @WebMvcTest |
|------|-----------------|-------------|
| Spring 容器 | 不启动 | 启动 Web 切片 |
| 启动耗时 | 毫秒级 | 秒级（受上下文缓存摊薄） |
| 拦截器/异常处理器 | 手工注册，容易漏 | 自动装配，与生产一致 |
| 参数校验（@Valid） | 默认不生效，需手工配置 Validator | 生效 |
| 安全过滤器链 | 无 | `@WithMockUser` 等全套可用 |
| 定位 | Controller 自身的快速单测 | Web 层契约的集成测试 |

**选择标准**：只测 Controller 内的分支逻辑（状态码映射、参数校验分支）用 standalone 图快；要验证"整条 Web 链路的装配"（拦截器顺序、全局异常处理、JSON 序列化配置）必须用 `@WebMvcTest`——standalone 下这些组件根本不在场，测了也是假绿。

## 3.2 典型用法与断言

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;   // 用生产同款序列化器构造请求体，避免手写 JSON 串

    @MockitoBean
    private OrderService orderService;

    @Test
    void should_create_order_and_return_201() throws Exception {
        CreateOrderRequest req = new CreateOrderRequest(List.of(new Item("sku-1", 2)));
        when(orderService.create(any())).thenReturn(new Order(100L, "CREATED"));

        mockMvc.perform(post("/orders")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(req)))
               .andExpect(status().isCreated())
               .andExpect(header().string("Location", containsString("/orders/100")))
               .andExpect(jsonPath("$.id").value(100))
               .andExpect(jsonPath("$.status").value("CREATED"));
    }
}
```

请求构造常用件：`get/post/put/delete`、`param("page","0")`（查询参数）、`header(...)`、`content(...)`（请求体）、`sessionAttrs(...)`；认证场景用 spring-security-test 的 `@WithMockUser` / `SecurityMockMvcRequestPostProcessors.user(...)`。断言常用件：`status()`、`header()`、`content().string(...)`、`jsonPath("$.a.b")`、`jsonPath(...).isArray()`，以及整体对比 `content().json(expectedJson, lenient)`。

## 3.3 andDo(print()) 调试

MockMvc 断言失败时的默认报错往往只有"expected 200 but was 500"，看不到响应体和异常栈。`andDo(print())` 把完整请求/响应（含 DispatcherServlet 抛出的异常）打到标准输出，是 Web 层测试的第一调试手段：

```java
mockMvc.perform(get("/orders/1"))
       .andDo(print())                       // 全量打印：请求头、响应体、处理链、异常
       .andExpect(status().isOk());
```

生产化做法是注册全局 printer，只在失败时打印（`MockMvcBuilders...alwaysDo(print())` 太吵），或用 SLF4J 版的 `MockMvcResultHandlers.log()`。`print()` 输出里最有价值的是 **Resolved Exception** 一行——500 背后的真实异常类型直接可见。

# 4. 常见组合模式

## 4.1 Service 层纯单测：Mockito 独立作战

绝大多数 Service 逻辑测试不需要 Spring：`@ExtendWith(MockitoExtension.class)` + `@Mock` 依赖 + `@InjectMocks` 被测类。毫秒级执行，是测试金字塔底座的主力形态。

## 4.2 Controller 切片：@WebMvcTest + @MockitoBean

Web 契约测试的标准三件套（见 3.2）。边界约定：**Controller 以下全部 Mock**，本切片只验证 HTTP 语义。

## 4.3 集成测试中 Mock 外部边界：@SpringBootTest + @MockitoBean

全量容器测试里，数据库等内部依赖用真实的（或 Testcontainers），**第三方 HTTP 服务用 Mock 替换**；调用外部的 RestTemplate/RestClient 也可用 `MockRestServiceServer` 拦截在 HTTP 报文层，比 Mock Java 对象更真实（能验证序列化与 URL）。

```text
测试形态光谱（隔离度递减，真实度递增）：

  纯 Mockito      @WebMvcTest       @SpringBootTest      @SpringBootTest
  Service 单测  →  Web 切片      →   + Mock 外部边界   →   + Testcontainers
  毫秒级           秒级(可缓存)       秒级                   十秒级
  验证逻辑分支      验证 HTTP 契约     验证装配与协作          验证真实中间件行为
```

# 5. 边界与误区：何时用真实现、何时 Mock

## 5.1 过度 Mock

**过度 Mock 的本质是把测试从"验证行为"退化成"验证实现步骤的复读机"**。症状：测试代码逐行 mock 了被测方法的每个内部调用，然后 verify 这些调用逐一发生——被测方法重构（换一个内部调用顺序、抽一个私有方法）测试就红，但业务行为根本没变。这种测试不防 bug，只防重构。判断口径：

- **该 Mock 的**：进程边界之外的东西（数据库、HTTP、消息队列、时钟、随机数）、有副作用且不可逆的东西（发邮件、扣款）。
- **不该 Mock 的**：被测体系内部的协作者。值对象、纯函数工具类、简单的领域服务——直接 new 真实现，测试更快也更稳。
- **灰色地带**：内部但有状态/重的组件（缓存、本地队列）。倾向真实现，除非初始化代价破坏了单测的速度契约。

同理，`verifyNoMoreInteractions` 把"没有发生任何其他调用"也焊进契约，几乎必然阻碍重构，除非在防重复扣款这类"多一次调用就是事故"的场景，否则不要使用。

## 5.2 Spy：半真半假的双刃剑

`spy()`/`@Spy` 包装真实对象，未打桩的方法走真实现。它的合理用途是**对难以注入的遗留代码做部分替换**（如 spy 一个真实 Service，只把其中访问外网的 `fetch()` 打桩）。危险在于：spy 上的 `when(spy.method()).thenReturn(...)` 写法会**先真实执行一次 method()**（副作用已经发生！），必须用 `doReturn(...).when(spy).method()`。新项目里如果频繁需要 spy，通常说明对象职责过重、该拆了。

## 5.3 只验调用、不验结果

另一种失衡是通篇 `verify(mock).xxx(...)` 却没有一条对返回值的断言。调用发生了不等于行为正确——**先断言业务结果，再按需验证关键副作用**。一个健康的测试通常两类断言都有，且业务断言在前。

# 6. 小结

| 主题 | 关键结论 |
|------|---------|
| 接线 | `MockitoExtension` 负责注解初始化与 strict stubs；也可 `mock()` 工厂方法零注解使用 |
| Stubbing | `when/thenReturn`、`thenAnswer`；void 用 `doXxx`；匹配器要么全用要么全不用 |
| Verify | 用于副作用与"不发生"断言；有返回值可断言时优先断言结果 |
| Strict Stubs | 默认严格，抓 UnnecessaryStubbing 与参数不匹配；`lenient()` 仅限存量迁移 |
| MockMvc | standaloneSetup 快但缺装配；`@WebMvcTest` 全链路真实；调试用 `andDo(print())` |
| Mock 边界 | Mock 进程外与副作用，真实现进程内；警惕"复读机测试" |

下一篇进入 Testcontainers，把"进程外的依赖"从 Mock 换成真实的容器。
