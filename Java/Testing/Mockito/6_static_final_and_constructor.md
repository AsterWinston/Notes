# 1. 静态、final 与构造器 Mock

早期的 Mockito 有一条明确边界：不能 Mock 静态方法、final 类和 final 方法。很多团队因此把"静态工具类"当成不可测试的合理借口。Mockito 5 默认启用 inline mock maker 后，这层限制被打破，`mockStatic`、`mockConstruction` 和 final 支持都可以用了。但能力越大，越需要知道什么时候不该用。

## 1.1 mock maker 与能力边界

Mockito 通过 **mock maker** 生成替身。传统实现基于动态代理或 Byte Buddy 子类，无法处理 final 和 static；inline mock maker 则在字节码层面修改目标类的方法体，因此可以拦截更多场景。

| 能力 | 传统 mock maker | inline mock maker |
|------|-----------------|-------------------|
| 普通接口/非 final 类 | 支持 | 支持 |
| final 类 | 不支持 | 支持 |
| final 方法 | 不支持 | 支持 |
| 静态方法 | 不支持 | 支持 |
| 构造器 | 不支持 | 支持 |
| 修改 `equals`/`hashCode` | 受限 | 支持 |
| 依赖 | 无 | Byte Buddy + Java agent |

Mockito 5 起，`mockito-core` 默认使用 inline mock maker，不再需要额外引入 `mockito-inline`。如果是 Mockito 4 或更早版本，需要显式添加 `mockito-inline` 依赖，或者通过 `mockito-extensions/org.mockito.plugins.MockMaker` 配置。

> **注意**：inline mock maker 需要动态加载 Java agent。JDK 21 及以上在首次加载时会打印警告，这属于正常现象。生产构建中如果禁用了 agent 加载，final/static Mock 会失效，需要检查 JVM 启动参数和构建环境。

## 1.2 mockStatic 基本用法

`mockStatic` 用于替换某个类的静态方法。它返回一个 `MockedStatic<T>`，必须在 `try-with-resources` 中使用：

```java
import org.junit.jupiter.api.Test;
import org.mockito.MockedStatic;

import java.time.Instant;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.mockito.Mockito.mockStatic;

class TimeServiceTest {

    @Test
    void should_use_fixed_clock() {
        Instant fixed = Instant.parse("2026-01-01T00:00:00Z");

        // 只在这个代码块内生效，离开 try 自动恢复真实静态方法
        try (MockedStatic<TimeUtils> mocked = mockStatic(TimeUtils.class)) {
            mocked.when(TimeUtils::now).thenReturn(fixed);

            TimeService service = new TimeService();
            assertEquals(fixed, service.currentTime());
        }
    }
}
```

`mocked.when(...)` 接收一个 `Verification`，也就是一个"调用静态方法的 lambda"。带参数的静态方法这样写：

```java
try (MockedStatic<IdGenerator> mocked = mockStatic(IdGenerator.class)) {
    // 只有参数为 "ORDER" 时才返回固定 ID
    mocked.when(() -> IdGenerator.next("ORDER")).thenReturn("ORD-1001");
    // 其他参数可以继续配置
    mocked.when(() -> IdGenerator.next("USER")).thenReturn("USR-2001");
}
```

除了打桩，`MockedStatic` 也支持验证：

```java
try (MockedStatic<IdGenerator> mocked = mockStatic(IdGenerator.class)) {
    service.createOrder();

    mocked.verify(() -> IdGenerator.next("ORDER"));
    mocked.verify(() -> IdGenerator.next("USER"), never());
    mocked.verifyNoMoreInteractions();
}
```

| 方法 | 作用 |
|------|------|
| `when(Verification)` | 对静态方法打桩 |
| `when(Verification, Answer)` | 用 Answer 动态返回 |
| `verify(Verification)` | 验证静态方法被调用一次 |
| `verify(Verification, VerificationMode)` | 按次数/模式验证 |
| `verifyNoMoreInteractions()` | 验证没有其他未验证的静态调用 |
| `reset()` | 清除当前静态 Mock 的所有桩和记录 |
| `close()` | 恢复真实静态方法，通常由 try-with-resources 调用 |

## 1.3 静态 Mock 的作用域与线程模型

`MockedStatic` 的作用域是**当前线程**。在测试线程里创建的静态 Mock，不会自动传播到线程池里的工作线程：

```java
try (MockedStatic<TimeUtils> mocked = mockStatic(TimeUtils.class)) {
    mocked.when(TimeUtils::now).thenReturn(fixed);

    // 当前线程能命中桩
    assertEquals(fixed, TimeUtils.now());

    // 子线程默认看不到这个静态 Mock，可能返回真实时间
    CompletableFuture.runAsync(() -> {
        // 这里的 TimeUtils.now() 不受上面 mocked 的影响
    }).join();
}
```

这个设计有好有坏。好处是测试之间不容易互相污染；坏处是被测代码一旦把静态调用放到异步线程，Mock 就会失效。遇到这种情况，应该把时间、ID 等静态依赖抽成可注入的接口（如 `Clock`、`IdGenerator`），而不是想办法让静态 Mock 跨线程传播。

> **警告**：不要在多个测试线程中同时对同一个类执行 `mockStatic`。虽然每个线程有自己的作用域，但底层字节码转换和类加载仍然共享状态，容易出现难以复现的 `MockitoException` 或行为错乱。并行测试时应避免静态 Mock，或者给每个测试隔离的类加载环境。

## 1.4 mockConstruction 基本用法

`mockConstruction` 拦截 `new` 表达式，让代码里创建的实例变成 Mock。它适合处理"被测类在方法内部直接 new 依赖"的遗留代码：

```java
import org.junit.jupiter.api.Test;
import org.mockito.MockedConstruction;

import static org.junit.jupiter.api.Assertions.assertTrue;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.Mockito.mockConstruction;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

class ReportExporterTest {

    @Test
    void should_write_report_with_mocked_writer() {
        try (MockedConstruction<PdfWriter> mocked =
                     mockConstruction(PdfWriter.class, (mock, context) -> {
                         // 每次构造时都配置这个 Mock 的默认行为
                         when(mock.write(anyString())).thenReturn(true);
                     })) {

            ReportExporter exporter = new ReportExporter();
            boolean ok = exporter.export("monthly");

            assertTrue(ok);
            // constructed() 按构造顺序返回所有被 Mock 的实例
            PdfWriter writer = mocked.constructed().get(0);
            verify(writer).write("monthly");
        }
    }
}
```

初始化器里的 `context` 提供了构造上下文：

| 方法 | 作用 |
|------|------|
| `context.arguments()` | 获取本次构造器调用的参数列表 |
| `context.count()` | 当前是该类型第几次构造，从 0 开始 |
| `context.getMock()` | 获取当前正在初始化的 Mock |

```java
try (MockedConstruction<Connection> mocked =
             mockConstruction(Connection.class, (mock, context) -> {
                 // 根据构造参数决定 Mock 行为
                 String url = (String) context.arguments().get(0);
                 when(mock.getUrl()).thenReturn(url);
                 when(mock.isOpen()).thenReturn(true);
             })) {

    // 被测代码内部 new Connection("jdbc:...") 时，返回配置好的 Mock
}
```

与静态 Mock 一样，构造器 Mock 也是**当前线程**生效，并且必须关闭。`MockedConstruction` 实现了 `AutoCloseable`，忘记关闭会导致后续测试中所有 `new` 都返回 Mock，症状是"对象字段全是 null"或"构造函数里的逻辑没执行"。

## 1.5 final 类与 final 方法

inline mock maker 下，final 类和 final 方法都可以像普通类型一样 Mock：

```java
public final class TaxCalculator {

    public final BigDecimal calculate(BigDecimal amount) {
        // 真实税率计算
        return amount.multiply(new BigDecimal("0.13"));
    }
}
```

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.math.BigDecimal;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class InvoiceServiceTest {

    @Mock
    private TaxCalculator taxCalculator; // final 类也能 Mock

    @Test
    void should_use_mocked_tax() {
        when(taxCalculator.calculate(new BigDecimal("100")))
                .thenReturn(new BigDecimal("13"));

        InvoiceService service = new InvoiceService(taxCalculator);

        assertEquals(new BigDecimal("113"), service.total(new BigDecimal("100")));
    }
}
```

虽然技术上可行，但**不要因为能 Mock final 类就到处 Mock final 类**。`String`、`BigDecimal`、`LocalDate` 这些值对象应该直接使用真实实例；对它们打桩会让测试脱离真实语义，而且 `BigDecimal.equals` 对精度敏感，很容易出现"参数看起来一样但不相等"的问题。

## 1.6 资源关闭与嵌套作用域

`MockedStatic` 和 `MockedConstruction` 都是需要关闭的资源。推荐用 `try-with-resources`，并保持作用域尽可能小：

```java
@Test
void should_scope_static_mock_tightly() {
    // 只把需要静态 Mock 的那一行包起来，而不是整个测试方法
    try (MockedStatic<TimeUtils> mocked = mockStatic(TimeUtils.class)) {
        mocked.when(TimeUtils::now).thenReturn(fixed);
        service.run();
    }

    // 这里已经恢复真实静态方法
    assertEquals(realNow.getYear(), TimeUtils.now().getYear());
}
```

嵌套使用多个静态 Mock 时，关闭顺序与打开顺序相反：

```java
try (MockedStatic<TimeUtils> time = mockStatic(TimeUtils.class);
     MockedStatic<IdGenerator> ids = mockStatic(IdGenerator.class)) {

    time.when(TimeUtils::now).thenReturn(fixed);
    ids.when(() -> IdGenerator.next("ORDER")).thenReturn("ORD-1");

    service.createOrder();
}
// 先关闭 ids，再关闭 time，try-with-resources 自动保证顺序
```

如果确实需要在 `@BeforeEach` 中打开静态 Mock，必须在 `@AfterEach` 中调用 `close()`；忘记这一步会污染后续测试。更好的做法仍然是缩小作用域，只在单个测试方法内部打开。

## 1.7 适用边界与替代方案

静态 Mock 和构造器 Mock 是"遗留代码的止痛药"，不是日常设计的首选。它们会让测试与类的加载、线程、资源生命周期绑定，调试成本明显高于普通 Mock。

| 场景 | 推荐方案 | 不推荐 |
|------|----------|--------|
| 时间获取 | 注入 `Clock` | `mockStatic(LocalDateTime.class)` |
| ID 生成 | 注入 `IdGenerator` | `mockStatic(UUID.class)` |
| 第三方静态工具 | 包一层适配器接口 | 直接在业务代码里静态调用 |
| 对象内部 new 依赖 | 构造器注入工厂 | `mockConstruction` 到处用 |
| 旧系统无法改结构 | 临时使用静态/构造 Mock | 作为长期架构 |
| 需要验证协议细节 | WireMock / Testcontainers | Mock 第三方 SDK |

**设计考量**：判断是否该用静态 Mock，可以问一个问题：这个静态依赖是否代表一个**外部边界**？如果是，优先包成接口并注入；如果只是遗留代码暂时无法改，静态 Mock 可以作为过渡。把 `System.currentTimeMillis()` 换成 `Clock`，把 `UUID.randomUUID()` 换成 `IdGenerator`，长期收益远大于在每个测试里写 `mockStatic`。

> **行业实践**：Google、Netflix 等团队的测试规范里，静态 Mock 通常被列为"最后手段"。它适合验证"静态工具方法被正确调用"的少量场景，不适合作为常规测试策略。测试代码里出现大量 `mockStatic`，往往说明生产代码的可测试性需要重构。

## 1.8 常见错误与排错

| 现象 | 原因 | 修正 |
|------|------|------|
| `mockStatic` 编译不通过 | Mockito 版本过低，或没有 inline mock maker | 升级到 Mockito 5 |
| 静态 Mock 在子线程失效 | `MockedStatic` 是线程局部的 | 把依赖抽成接口，或让异步任务在测试线程执行 |
| 后续测试行为异常 | 忘记关闭 `MockedStatic` | 用 try-with-resources |
| `new` 出来的对象全是 null | `MockedConstruction` 未关闭，污染后续测试 | 立即关闭，检查测试隔离 |
| 构造器逻辑没有执行 | 构造器 Mock 默认不调用真实构造器 | 需要真实初始化时改用工厂注入或部分 Mock |
| 静态初始化副作用仍然发生 | 类的静态初始化在加载时执行 | 不要用 `mockStatic` 规避静态初始化 |
| JDK 类 Mock 失败 | inline 无法可靠转换引导类加载器中的类 | 改用 `Clock` 等抽象，不要 Mock JDK 核心类 |
| 并行测试偶发失败 | 静态 Mock 与线程/类加载交互 | 禁用并行，或隔离测试进程 |

# 2. 练习与自检

1. 用 `mockStatic` 把 `TimeUtils.now()` 固定为 `2026-01-01T00:00:00Z`，并保证离开代码块后恢复真实方法。
2. 用 `mockConstruction` 拦截 `new PdfWriter(path)`，让 `write` 返回 `true` 并验证调用参数。
3. 解释 `mockStatic` 为什么在子线程失效，并给出一个不依赖静态 Mock 的改造方案。

# 3. 小结

`mockStatic`、`mockConstruction` 和 final 支持让 Mockito 能覆盖更多遗留代码，但它们不是"更高级的 Mock"，而是"更靠近底层的补丁"。使用时牢牢记住三点：作用域限于当前线程、必须用 try-with-resources 关闭、优先用依赖注入替代对静态状态的拦截。

| 需求 | 推荐 API | 风险等级 |
|------|----------|----------|
| 替换静态工具方法 | `MockedStatic` | 中高 |
| 拦截内部 `new` | `MockedConstruction` | 高 |
| Mock final 类/方法 | Mockito 5 inline | 低到中 |
| 替换时间 | 注入 `Clock` | 低 |
| 替换 ID/随机数 | 注入生成器接口 | 低 |
| 测试遗留系统 | 局部临时使用 | 中 |

下一篇 `7_spring_integration.md` 把视角从纯单元测试抬到 Spring 测试切片，讨论 `@MockitoBean`、`@WebMvcTest`、`MockMvc` 以及 Spring Boot 3 带来的变化。
