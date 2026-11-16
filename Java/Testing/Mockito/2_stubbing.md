# 1. Stub 与行为预设

打桩（Stubbing）是 Mockito 使用频率最高的能力：在测试开始前告诉替身"当某个方法以某组参数被调用时，应该返回什么或抛出什么"。理解打桩，等于理解了 Mockito 的一半。另一半是验证，放到下一篇展开。

## 1.1 Stub 与 Mock 的边界

在测试理论里，Stub 和 Mock 是两种不同的测试替身。Stub 负责**提供预设响应**，让被测代码沿着某条路径继续执行；Mock 负责**记录交互**，让测试在事后验证"依赖是否被正确调用"。

| 维度 | Stub | Mock |
|------|------|------|
| 关注点 | 输入/输出 | 交互行为 |
| 典型 API | `when(...).thenReturn(...)` | `verify(...)` |
| 断言对象 | 被测方法的返回值 | 依赖的方法是否被调用 |
| 失败含义 | 业务结果不符合预期 | 协作方式不符合预期 |
| 过度使用后果 | 测试变成"喂数据" | 测试耦合实现细节 |

Mockito 用同一个代理对象同时支持这两种用法，所以代码里经常出现"既打桩又验证"的测试。**设计考量**：能通过返回值断言的结果，尽量不要再用 `verify` 重复验证；只有返回值无法表达副作用时（例如发送邮件、写入审计日志），交互验证才是必要的。

## 1.2 when/thenReturn 基本用法

最基础的形式是 `when(方法调用).thenReturn(返回值)`：

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Optional;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository repository;

    @Test
    void should_return_name() {
        // 参数精确匹配 1L 时，返回固定用户
        when(repository.findById(1L))
                .thenReturn(Optional.of(new User(1L, "Ada")));

        UserService service = new UserService(repository);

        assertEquals("Ada", service.getUser(1L).name());
    }
}
```

`when(repository.findById(1L))` 这一行并不是在"执行"仓库查询，而是在向 Mockito 登记一条规则：**如果之后有人以 `1L` 调用 `findById`，就返回我指定的值**。Mockito 通过代理拦截了这次调用，因此不会真的访问数据库。

### 1.2.1 匹配任意参数

当参数值不重要，或者需要覆盖多个输入时，可以用参数匹配器：

```java
import static org.mockito.ArgumentMatchers.anyLong;
import static org.mockito.Mockito.when;

when(repository.findById(anyLong()))
        .thenReturn(Optional.of(new User(1L, "Ada")));
```

注意：一旦某个参数使用了匹配器，**同一个方法的所有参数都必须使用匹配器**。这个规则会在第 3 篇详细展开。

## 1.3 默认返回值

Mockito 对未打桩的方法不会抛异常，而是返回该类型的默认值。这个设计让测试可以只关心真正重要的协作，但也容易掩盖遗漏的桩。

| 返回类型 | 默认值 |
|----------|--------|
| 对象引用 | `null` |
| `int` / `long` / `short` / `byte` | `0` |
| `double` / `float` | `0.0` |
| `boolean` | `false` |
| `char` | `'\u0000'` |
| `Optional<T>` | `Optional.empty()` |
| `Stream<T>` | `Stream.empty()` |
| `List` / `Set` / `Map` | 空集合 |
| `String` | `null` |

> **注意**：`Optional.empty()` 和空集合的默认值从 Mockito 2 开始提供，不要在老版本上假设它们一定存在。若项目仍在使用 Mockito 1.x，默认值会退化为 `null`。

需要改变全局默认行为时，可以在创建 Mock 时指定 Answer：

```java
import org.mockito.Mockito;
import static org.mockito.Mockito.mock;

// 让未打桩的对象方法返回"智能 null"，调用其方法时给出可读的错误
UserRepository smart = mock(
        UserRepository.class,
        Mockito.RETURNS_SMART_NULLS
);
```

| Answer 策略 | 行为 | 适用场景 |
|-------------|------|----------|
| `RETURNS_DEFAULTS` | 返回类型默认值 | 默认行为 |
| `RETURNS_SMART_NULLS` | 返回能提示问题的 SmartNull | 排查空指针来源 |
| `RETURNS_MOCKS` | 对象返回另一个 Mock | 链式调用，谨慎使用 |
| `RETURNS_DEEP_STUBS` | 自动生成深层桩 | 遗留代码，容易掩盖耦合 |
| `CALLS_REAL_METHODS` | 调用真实方法 | 部分 Mock，等价于 Spy |

**设计考量**：`RETURNS_DEEP_STUBS` 看起来很方便，例如 `when(service.getUser().getAddress().getCity())`，但它会让测试与对象图结构强绑定。一旦中间某层重构，测试全部失效。大多数情况下，应该把需要的数据直接构造出来，而不是让 Mock 自动向下展开。

## 1.4 抛出异常

`thenThrow` 用于模拟依赖失败，例如数据库连接超时、远程接口返回 500：

```java
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.mockito.Mockito.when;

@Test
void should_wrap_repository_failure() {
    when(repository.findById(1L))
            .thenThrow(new DataAccessResourceFailureException("数据库不可用"));

    UserService service = new UserService(repository);

    assertThrows(DataAccessResourceFailureException.class,
            () -> service.getUser(1L));
}
```

如果抛出的是受检异常，测试方法需要声明 `throws` 或捕获它，否则异常会一路传播出去；`thenThrow` 本身不限制异常类型，但要保证测试能处理。对于 `void` 方法，不能使用 `when(...)`，要改用 `doThrow(...).when(mock).method()`，否则代码无法通过编译或会在运行时触发真实调用。

```java
// void 方法抛异常：必须使用 doThrow
doThrow(new IllegalStateException("队列已关闭"))
        .when(messageQueue)
        .send(anyString());

// 非 void 方法也可以使用 doThrow，但通常 thenThrow 更直观
doThrow(new IOException("磁盘已满"))
        .when(fileWriter)
        .write(anyString());
```

## 1.5 连续返回与状态变化

有些方法会被调用多次，每次需要返回不同结果，例如第一次失败、第二次成功，或者分页查询逐页返回。`thenReturn` 支持传入多个值：

```java
// 第一次返回 1，第二次返回 2，第三次返回 3
when(counter.next()).thenReturn(1, 2, 3);

// 等价的链式写法
when(counter.next())
        .thenReturn(1)
        .thenReturn(2)
        .thenReturn(3);

// 混合返回值与异常：第一次抛异常，第二次正常返回
when(remoteClient.fetch("key"))
        .thenThrow(new IllegalStateException("首次超时"))
        .thenReturn("cached-value");
```

当预设值用完之后，Mockito 会**重复使用最后一个值**。例如 `thenReturn(1, 2)` 第三次调用会继续返回 `2`，而不是回到 `1`。这一点在模拟重试逻辑时非常有用。

## 1.6 使用 Answer 动态计算返回值

固定返回值无法表达"根据入参计算"或"每次调用产生不同结果"的场景。`thenAnswer` 允许你在调用发生时执行一段逻辑：

```java
import org.mockito.invocation.InvocationOnMock;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.when;

when(repository.save(any(User.class))).thenAnswer(invocation -> {
    User input = invocation.getArgument(0);
    // 模拟数据库生成自增主键
    return new User(100L, input.name());
});
```

`InvocationOnMock` 提供几个常用方法：

| 方法 | 作用 |
|------|------|
| `getArgument(int index)` | 按位置取参数，返回泛型推断类型 |
| `getArgument(int index, Class<T> type)` | 按位置取参数并指定类型 |
| `getArguments()` | 取全部参数数组 |
| `getMethod()` | 获取被调用的 `Method` 对象 |
| `callRealMethod()` | 调用真实方法（Spy 场景） |

Answer 也可以抽成独立类，便于复用：

```java
public class GenerateIdAnswer implements Answer<User> {

    private final AtomicLong sequence = new AtomicLong(1);

    @Override
    public User answer(InvocationOnMock invocation) {
        User input = invocation.getArgument(0);
        // 每次保存都生成递增 ID，模拟数据库序列
        return new User(sequence.getAndIncrement(), input.name());
    }
}
```

> **行业实践**：Answer 里不要写复杂业务逻辑。测试中的 Answer 越复杂，测试本身越难维护。如果一段 Answer 超过十行，通常说明被测代码的依赖边界需要重新设计，或者应该改用 Fake 实现。

## 1.7 doReturn、doThrow 与 doAnswer

`when(...)` 形式在语法上会先执行一次 mock 方法。对普通 Mock 来说这没有副作用，但对 Spy 或真实对象来说，这一次执行会**真的调用真实方法**。为了避免这种副作用，Mockito 提供了 `do...when` 语法：

```java
import static org.mockito.Mockito.doReturn;
import static org.mockito.Mockito.doThrow;
import static org.mockito.Mockito.doNothing;

// Spy 场景：避免调用真实方法
doReturn("mocked").when(spy).expensiveQuery();

// void 方法：doThrow 是首选写法
doThrow(new RuntimeException("失败")).when(mock).voidMethod();

// 让 void 方法什么都不做（默认行为，显式写出便于阅读）
doNothing().when(mock).voidMethod();

// 调用真实方法，常用于部分 Mock
doCallRealMethod().when(spy).realMethod();
```

两种写法的对比如下：

| 场景 | `when(...).then...` | `do...when(...)` |
|------|---------------------|------------------|
| 普通 Mock 返回值 | 推荐 | 可用 |
| 普通 Mock 抛异常 | 推荐 | 可用 |
| `void` 方法 | 不可用 | 必须使用 |
| Spy 打桩 | 会触发真实调用，危险 | 推荐 |
| 多次连续返回 | 支持 | 支持度较弱，建议用 `when` |
| 可读性 | 接近自然语言 | 稍微绕，但语义清晰 |

## 1.8 桩的覆盖顺序与宽松模式

当同一个方法被多次打桩时，**后声明的桩会覆盖先声明的桩**。如果参数范围有重叠，也是最后匹配的那条生效：

```java
when(repository.findById(anyLong())).thenReturn(Optional.empty());
when(repository.findById(1L)).thenReturn(Optional.of(ada));

// 1L 命中第二条，返回 ada；其他 ID 命中第一条，返回 empty
```

严格模式下，如果一个桩从未被调用，`MockitoExtension` 会在测试结束后抛出 `UnnecessaryStubbingException`。临时放宽某一条桩的检查可以这样做：

```java
import static org.mockito.Mockito.lenient;

// 只对这条桩放宽，适合在 @BeforeEach 中声明、仅部分测试使用的桩
lenient().when(repository.findById(1L)).thenReturn(Optional.of(ada));
```

> **注意**：`lenient()` 是诊断工具，不是常规写法。大量使用会掩盖"测试走了错误分支"的问题，最终让测试失去保护作用。

## 1.9 常见错误与排错

| 现象 | 原因 | 修正 |
|------|------|------|
| `when` 里调用了真实方法 | 对 Spy 使用了 `when(spy.method())` | 改用 `doReturn(...).when(spy).method()` |
| `UnnecessaryStubbingException` | 桩声明了但没走到 | 把桩移入对应测试，或检查分支条件 |
| `PotentialStubbingProblem` | 实际参数与桩参数不完全一致 | 用 `eq`/`any` 匹配器，或校正参数 |
| `when(voidMethod())` 编译失败 | `void` 没有返回值，不能放进 `when` | 改用 `doThrow` / `doNothing` |
| 桩不生效 | 参数使用了不同的匹配器或不同的对象相等性 | 打印实际参数，检查 `equals` 实现 |
| 测试之间互相影响 | 共用 Mock 或在 `@BeforeEach` 里做了全局桩 | 每个测试独立创建，避免共享状态 |

还有一个容易忽略的问题：**不要给 `toString()`、`equals()`、`hashCode()` 打桩**。这些方法被 Mockito 自身、集合框架和调试器大量调用，打桩后可能出现难以解释的行为。如果对象需要参与集合去重，应该使用真实值对象或 `ArgumentCaptor` 检查。

# 2. 练习与自检

1. 用一条链式调用模拟"第一次抛 `TimeoutException`、第二次返回缓存值、之后一直返回缓存值"。
2. 解释 `when(spy.method())` 与 `doReturn(...).when(spy).method()` 的执行差异，并说明各自适用场景。
3. 未打桩的 `Optional` 返回方法默认返回什么？这个默认值可能掩盖哪类缺陷？

# 3. 小结

打桩的本质是"控制依赖的响应"，而不是"让测试通过"。最常用的组合是 `when(...).thenReturn(...)` 处理返回值、`thenThrow` 处理异常、`thenAnswer` 处理动态逻辑；遇到 `void` 方法或 Spy 时切换到 `do...when` 语法。

| 需求 | 推荐 API |
|------|----------|
| 固定返回值 | `when(...).thenReturn(...)` |
| 抛出异常 | `when(...).thenThrow(...)` |
| 连续多个结果 | `thenReturn(a, b, c)` |
| 根据入参计算 | `thenAnswer(...)` |
| `void` 方法 | `doThrow(...).when(mock).method()` |
| Spy 打桩 | `doReturn(...).when(spy).method()` |

下一篇 `3_argument_matchers.md` 会专门讲参数匹配器，解决"参数不完全相等时怎么打桩、怎么验证"的问题。
