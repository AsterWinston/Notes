# 1. Mock、Spy 与依赖注入

前面的打桩和验证都建立在"已经有一个替身对象"这个前提上。本篇回答替身从哪里来：`@Mock` 创建一个完全虚构的对象，`@Spy` 包装一个真实对象，`@InjectMocks` 负责把替身塞进被测类。三者组合起来，就是 Mockito 最常见的测试骨架。

## 1.1 两条测试替身路线

Mockito 的替身大致分两条路线：

| 维度 | `@Mock` | `@Spy` |
|------|---------|--------|
| 本质 | 动态生成的代理对象 | 真实对象的包装 |
| 未打桩方法 | 返回默认值 | 执行真实逻辑 |
| 构造器 | 不调用被测类型构造器 | 可能调用真实构造器 |
| 适用场景 | 隔离外部依赖 | 部分模拟遗留代码 |
| 风险 | 过度隔离，测试失真 | 真实调用带来副作用 |
| 推荐度 | 默认选择 | 谨慎选择 |

`@InjectMocks` 不是替身，它只是"注入器"。它把当前测试类里的 `@Mock`、`@Spy` 对象按规则注入到被测对象中，本身不创建替身，也不改变替身的类型。

```text
┌─────────────────────────────────────────────┐
│              UserServiceTest                 │
│                                             │
│  @Mock UserRepository ──┐                   │
│  @Spy  PasswordEncoder ─┼──▶ @InjectMocks   │
│  @Mock AuditClient ─────┘      UserService  │
└─────────────────────────────────────────────┘
```

## 1.2 @Mock 的创建与配置

`@Mock` 是最常用的注解。它在测试实例创建后、`@BeforeEach` 之前生成一个实现目标类型的代理。对接口来说，它生成接口的实现；对类来说，它生成子类（inline mock maker 下也可以处理 final 类）。

```java
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;

    @Mock
    private PaymentGateway paymentGateway;

    @Mock
    private InventoryClient inventoryClient;
}
```

`@Mock` 还支持 `answer = ...` 调整默认行为，例如 `RETURNS_SMART_NULLS` 或 `RETURNS_DEEP_STUBS`。后者容易让测试与对象图强耦合，不要默认开启。

**设计考量**：`@Mock` 应该优先用于**接口或端口**，而不是具体实现类。依赖接口意味着测试只关心契约，不关心实现；如果被测类直接依赖了某个具体类，通常说明依赖倒置没有做好，先考虑重构，而不是用 Mockito 硬 Mock。

## 1.3 @Spy 的两种创建方式

`@Spy` 包装一个真实对象。未打桩的方法执行真实逻辑，打桩的方法返回预设值。它有两种常见初始化方式：

### 1.3.1 字段声明时提供实例

```java
import org.mockito.Spy;

@Spy
private PriceCalculator priceCalculator = new PriceCalculator();

@Spy
private PasswordEncoder passwordEncoder = new BCryptPasswordEncoder();
```

这种方式最直观：Mockito 不会重新调用构造器，而是在你提供的实例基础上生成 spy 代理。

### 1.3.2 只声明类型，由 Mockito 构造

```java
@Spy
private PriceCalculator priceCalculator;
```

如果字段没有初始化，Mockito 会尝试调用目标类的**无参构造器**创建实例。如果目标类没有无参构造器，测试启动时会直接失败。因此对需要依赖的类，推荐显式传入实例，或者干脆使用 `@Mock`。

> **注意**：`Mockito.spy(new Foo())` 会复制传入对象的状态，但不会再次执行构造器。也就是说，构造器里的副作用只发生一次；而 `@Spy` 无初始化字段时，构造器副作用会在 Mockito 创建实例时发生。Spy 的真实调用风险从构造阶段就开始了。

## 1.4 @InjectMocks 的注入机制

`@InjectMocks` 会尝试把测试类里的 Mock/Spy 注入被测对象。Mockito 的注入顺序大致是：

```text
┌──────────────────────┐
│ 1. 构造器注入          │  选择参数最多的构造器，按类型匹配
└──────────┬───────────┘
           ▼
┌──────────────────────┐
│ 2. Setter 注入        │  按类型匹配，再按字段名匹配
└──────────┬───────────┘
           ▼
┌──────────────────────┐
│ 3. 字段注入            │  反射写入非 final 字段
└──────────────────────┘
```

### 1.4.1 构造器注入（推荐）

```java
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class CheckoutServiceTest {

    @Mock
    private OrderRepository orderRepository;

    @Mock
    private PaymentGateway paymentGateway;

    @Mock
    private InventoryService inventoryService;

    @InjectMocks
    private CheckoutService checkoutService; // 通过构造器注入上面三个 Mock
}
```

被测类应当写成构造器注入：

```java
public class CheckoutService {

    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;
    private final InventoryService inventoryService;

    public CheckoutService(OrderRepository orderRepository,
                           PaymentGateway paymentGateway,
                           InventoryService inventoryService) {
        this.orderRepository = orderRepository;
        this.paymentGateway = paymentGateway;
        this.inventoryService = inventoryService;
    }

    public void checkout(Order order) {
        inventoryService.reserve(order.sku(), order.quantity());
        paymentGateway.charge(order.total());
        orderRepository.save(order);
    }
}
```

这是最稳妥的组合。构造器注入让依赖关系显式，`@InjectMocks` 只需按参数类型找到对应的 Mock。如果构造器参数类型有重复（例如两个 `String` 或两个同类型仓库），Mockito 会按字段名做二次匹配，但可读性会下降，最好避免。

### 1.4.2 字段注入（兼容旧代码）

```java
public class LegacyUserService {
    private UserRepository userRepository;

    public void setUserRepository(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public User load(Long id) {
        return userRepository.findById(id).orElseThrow();
    }
}
```

`@InjectMocks` 会先尝试 setter，再尝试直接写字段。字段注入看起来省事，但它让依赖关系隐藏在类内部，生产代码也更容易出现"对象创建到一半、依赖还是 null"的问题。测试能跑通不代表设计健康。

### 1.4.3 注入失败是静默的

`@InjectMocks` 最危险的地方在于：**注入失败通常不会报错**。如果被测类没有匹配的构造器、setter 或字段，Mockito 可能留下一个 `null` 字段，然后在测试执行时抛出 `NullPointerException`，错误信息却指向业务代码。

需要特别注意的是，Mockito 不会修改 `final` 字段。遇到构造器参数不匹配、字段又是 `final` 的情况，应该改用构造器注入或手动 `new`，而不是想办法用反射绕过。

> **行业实践**：新项目统一使用构造器注入，测试里甚至可以不用 `@InjectMocks`，直接 `new UserService(repository, gateway)`。显式构造更容易阅读、调试和重构，也避免了注入失败时的静默 `null`。

## 1.5 部分 Mock 与 doReturn

Spy 的典型用途是"只替换类中的一两个方法，其余逻辑保持真实"。例如一个报表服务里，`generate()` 依赖耗时的 `loadData()`，测试只想验证汇总逻辑：

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Spy;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.List;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.mockito.Mockito.doReturn;

@ExtendWith(MockitoExtension.class)
class ReportServiceTest {

    @Spy
    private ReportService reportService = new ReportService();

    @Test
    void should_summarize_loaded_rows() {
        // 只替换耗时的数据加载，汇总逻辑仍然执行真实代码
        doReturn(List.of(10, 20, 30)).when(reportService).loadData();

        Report report = reportService.generate();

        assertEquals(60, report.total());
    }
}
```

这里必须使用 `doReturn(...).when(spy).loadData()`。如果写成 `when(spy.loadData()).thenReturn(...)`，`loadData()` 会先被真实执行一次，可能访问数据库、读文件或发起 HTTP 请求，测试还没开始就已经产生了副作用。

| 写法 | 是否触发真实方法 | 推荐场景 |
|------|------------------|----------|
| `when(spy.method()).thenReturn(x)` | 是 | 不推荐用于 Spy |
| `doReturn(x).when(spy).method()` | 否 | Spy 打桩的标准写法 |
| `doThrow(e).when(spy).voidMethod()` | 否 | Spy 的 void 方法 |
| `doCallRealMethod().when(spy).method()` | 是（验证时） | 显式恢复真实调用 |

## 1.6 Spy 的真实调用风险

Spy 的直觉是"只改一个方法，其他都真实"，但它带来的风险比 `@Mock` 高得多：

| 风险 | 说明 | 应对 |
|------|------|------|
| 构造器副作用 | `@Spy` 无初始化字段会执行构造器 | 显式传入安全实例，或改用 Mock |
| 真实 I/O | 未打桩的方法可能查库、发请求 | 只对纯逻辑类使用 Spy |
| 状态污染 | 真实方法修改内部集合/缓存 | 每个测试新建 Spy，不复用 |
| 时间与随机 | 真实方法依赖 `now()`、`random()` | 把时间/随机源抽成依赖再 Mock |
| 静态状态 | 真实方法读写静态字段 | 优先重构，不要用 Spy 掩盖 |
| final 方法 | 默认 mock maker 无法拦截 | Mockito 5 inline 可处理，但仍应谨慎 |

```java
// 危险：未打桩的 send() 会真的发邮件
@Spy
private EmailSender emailSender = new SmtpEmailSender();

@Test
void should_render_template() {
    String body = emailSender.render("welcome", user); // 只测渲染
    // 如果这里不小心调用了 emailSender.send(...)，就真的发出去了
}
```

**设计考量**：Spy 最适合"遗留代码里的一个类，大部分方法是纯计算，只有少量方法有副作用"。如果一个类需要 Spy 才能测试，往往说明职责过重或依赖没有抽离。与其用 Spy 绕过，不如把外部交互抽成接口，用 `@Mock` 替换。

### 1.6.1 Spy 与 Spring AOP 代理

在 Spring 环境中，`@SpyBean` 或 `@MockitoSpyBean` 可能拿到 AOP 代理而不是目标对象，直接验证代理类型有时会失配。可以用 `AopTestUtils.getTargetObject(spyBean)` 取出目标对象，但这也提示测试层次过高；单元测试里直接 `new` 一个对象再 Spy 通常更简单。

## 1.7 构造器注入、字段注入与 Setter 注入对比

| 维度 | 构造器注入 | Setter 注入 | 字段注入 |
|------|------------|-------------|----------|
| 依赖可见性 | 高，编译期强制 | 中 | 低 |
| 不可变性 | 支持 final | 不支持 | 不支持 |
| 测试友好度 | 高，可直接 new | 中 | 低，依赖反射 |
| `@InjectMocks` 支持 | 支持 | 支持 | 支持（非 final） |
| 循环依赖 | 启动时暴露 | 可缓解 | 可隐藏 |
| 推荐度 | 首选 | 可选 | 避免 |

## 1.8 常见错误与排错

| 现象 | 原因 | 修正 |
|------|------|------|
| `@InjectMocks` 后字段为 `null` | 没有被测类可用的构造器/setter/字段 | 改用构造器注入，或手动 new |
| `Cannot instantiate @InjectMocks field` | 被测类没有无参构造器且无法匹配参数 | 提供可注入的构造器，或手动创建对象 |
| `@Spy` 启动时报无参构造器缺失 | 字段未初始化，Mockito 尝试默认构造 | 显式 `new` 一个实例 |
| Spy 打桩时执行了真实逻辑 | 用了 `when(spy.method())` | 改用 `doReturn` |
| Mock 注入到了错误的字段 | 同类型存在多个候选 | 用构造器参数顺序明确依赖，或改用 `@Qualifier` 风格的命名 |
| 测试之间状态串联 | Spy 实例被静态/共享字段复用 | 每个测试新建 Spy |
| Spring 中 `@Mock` 不生效 | 用了 JUnit 的 `@Mock` 而不是 Spring 的 `@MockitoBean` | 根据测试层次选择正确的注解 |
| final 字段无法注入 | Mockito 不修改 final 字段 | 构造器注入或重构 |

# 2. 练习与自检

1. `@InjectMocks` 注入失败为什么通常不报错？写出两种排查手段。
2. 用 `@Spy` 替换 `ReportService.loadData()`，保留 `generate()` 的真实汇总逻辑，并验证只调用了一次 `loadData`。
3. 比较构造器注入与字段注入在可测试性、不可变性和注入失败表现上的差异。

# 3. 小结

`@Mock` 负责隔离，`@Spy` 负责部分真实，`@InjectMocks` 负责把两者装进被测对象。最稳妥的组合是"接口依赖 + 构造器注入 + `@Mock`"；Spy 只在确实需要保留大部分真实逻辑时使用，并且永远用 `doReturn` 打桩。注入失败是静默的，测试里出现莫名其妙的 `NullPointerException` 时，先检查依赖到底有没有被注入。

| 需求 | 推荐做法 |
|------|----------|
| 隔离外部依赖 | `@Mock` + 构造器注入 |
| 部分模拟遗留类 | `@Spy` + `doReturn` |
| 自动装配依赖 | `@InjectMocks`，但优先显式 new |
| 避免构造器副作用 | 显式传入安全实例 |
| Spring 容器中的替换 | `@MockitoBean` / `@MockitoSpyBean` |
| 排查注入问题 | 打印字段、断点构造器、改用显式构造 |

下一篇 `6_static_final_and_constructor.md` 进入 Mockito 的高级能力：静态方法、构造器和 final 类/方法的 Mock。这些能力很强，但也最容易把测试带向"过度控制"的歧路。
