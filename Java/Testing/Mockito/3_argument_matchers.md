# 1. 参数匹配器

真实业务里，依赖方法的参数往往不是固定值：ID 由数据库生成、时间戳每次不同、对象经过多层转换。如果只能按精确值打桩和验证，测试会变得又长又脆。**参数匹配器（Argument Matcher）**就是 Mockito 提供的"按规则匹配参数"机制，让测试可以表达"任意用户 ID""字符串以 `ORD-` 开头"这类条件。

## 1.1 为什么需要参数匹配器

先看一个没有匹配器的测试：

```java
when(orderRepository.save(new Order(null, "BOOK-1", 2)))
        .thenReturn(new Order(100L, "BOOK-1", 2));
```

这段代码假设 `Order` 正确实现了 `equals`，并且被测代码传入的对象与测试里构造的对象完全相等。一旦服务在保存前给订单加了创建时间、优惠金额或默认状态，测试就会失败——即使业务行为完全正确。匹配器可以把断言从"对象完全相等"放宽到"我关心的字段符合条件"：

```java
when(orderRepository.save(argThat(order ->
        order != null && "BOOK-1".equals(order.sku()) && order.quantity() == 2
))).thenReturn(new Order(100L, "BOOK-1", 2));
```

**设计考量**：匹配器解决的是"参数值不固定"的问题，不是"懒得写精确断言"的问题。匹配条件越宽，测试越不敏感；条件越窄，测试越接近精确值。好的测试通常在"能表达业务规则"和"不过度绑定实现"之间取平衡。

## 1.2 常用匹配器速查

Mockito 的匹配器集中在 `org.mockito.ArgumentMatchers` 中。常用方法如下：

| 匹配器 | 匹配范围 | 是否匹配 `null` |
|--------|----------|-----------------|
| `eq(value)` | 使用 `equals` 比较 | 取决于值 |
| `same(value)` | 引用相同 | 取决于值 |
| `any()` | 任意对象，包括 `null` | 是 |
| `any(Class<T>)` | 指定类型的非空对象 | 否 |
| `anyString()` | 任意非空字符串 | 否 |
| `anyInt()` / `anyLong()` | 任意基本类型值 | 不适用 |
| `isNull()` | 仅 `null` | 是 |
| `notNull()` | 非 `null` | 否 |
| `nullable(Class<T>)` | 指定类型或 `null` | 是 |
| `contains("x")` | 字符串包含子串 | 否 |
| `startsWith("x")` | 字符串以指定内容开头 | 否 |
| `endsWith("x")` | 字符串以指定内容结尾 | 否 |
| `matches(regex)` | 字符串匹配正则 | 否 |
| `isA(Class<T>)` | 类型匹配，支持子类 | 否 |
| `anyList()` / `anyMap()` | 任意非空集合 | 否 |
| `argThat(matcher)` | 自定义谓词 | 由谓词决定 |
| `aryEq(array)` | 数组内容逐项相等 | 否 |

`any()` 和 `any(Class)` 的差异是最容易踩坑的地方。`any()` 会匹配 `null`，而 `any(Class)`、`anyString()` 都不会。也就是说，下面两条桩的语义完全不同：

```java
// 任何参数，包括 null，都返回空用户
when(repository.findByName(any())).thenReturn(null);

// 只匹配非空 String；传 null 时不会命中这条桩
when(repository.findByName(anyString())).thenReturn(null);
```

> **注意**：`any(Class)` 的"非空"限制经常导致测试中出现莫名其妙的 `PotentialStubbingProblem`。如果被测代码可能传入 `null`，要么显式用 `isNull()` 单独打桩，要么用 `nullable(String.class)` 覆盖两种情况。

## 1.3 匹配器混用规则

Mockito 有一条硬性规则：**一旦某个参数使用了匹配器，同一个方法调用的所有参数都必须使用匹配器**。不能再混入原始值。

```java
// 错误示范：第一个参数是匹配器，第二个是原始值
when(repository.findByStatusAndName(any(Status.class), "Ada"))
        .thenReturn(List.of(ada));
```

运行时会抛出 `InvalidUseOfMatchersException`，提示 "When using matchers, all arguments have to be provided by matchers"。正确写法是用 `eq` 包装原始值：

```java
// 正确做法：所有参数都交给匹配器
when(repository.findByStatusAndName(any(Status.class), eq("Ada")))
        .thenReturn(List.of(ada));
```

同样的规则适用于 `verify`：

```java
// 错误：verify 里混用了 any() 和原始值
verify(repository).findByStatusAndName(any(Status.class), "Ada");

// 正确：统一用匹配器
verify(repository).findByStatusAndName(any(Status.class), eq("Ada"));
```

这条规则的底层原因是 Mockito 用调用栈中的参数匹配器数量来推断参数位置。混用时参数数量对不上，Mockito 无法确定每个匹配器对应哪个参数，于是直接报错。

## 1.4 argThat 与自定义匹配器

当内置匹配器无法表达复杂条件时，可以使用 `argThat` 传入 lambda 或 `ArgumentMatcher` 实现：

```java
import static org.mockito.ArgumentMatchers.argThat;
import static org.mockito.Mockito.when;

when(repository.save(argThat(order ->
        order != null
                && order.total().compareTo(BigDecimal.valueOf(100)) > 0
                && "VIP".equals(order.channel())
))).thenReturn(savedOrder);
```

对于需要复用、需要可读描述或逻辑较长的匹配器，建议抽成独立类：

```java
import org.mockito.ArgumentMatcher;

public class AdultUserMatcher implements ArgumentMatcher<User> {

    @Override
    public boolean matches(User user) {
        // 空值必须显式处理，否则可能抛出 NullPointerException
        return user != null && user.age() >= 18;
    }

    @Override
    public String toString() {
        // 失败信息里会展示这段描述，帮助定位问题
        return "年龄大于等于 18 岁的用户";
    }
}
```

使用方式：

```java
when(userRepository.save(argThat(new AdultUserMatcher())))
        .thenReturn(savedUser);

verify(userRepository).save(argThat(new AdultUserMatcher()));
```

`ArgumentMatcher` 的 `matches` 方法接收的是实际参数，**可能为 `null`**。如果谓词直接调用 `user.age()` 而不判空，遇到空参数就会抛 `NullPointerException`，而不是返回 `false`。这是自定义匹配器最常见的错误。

### 1.4.1 匹配器中的异常处理

```java
// 错误示范：没有判空，null 参数直接 NPE
return user.age() >= 18;

// 正确做法：先判空，再执行业务判断
return user != null && user.age() >= 18;
```

如果确实希望匹配 `null`，不要依赖 `argThat` 的隐式行为，明确使用 `isNull()` 或 `nullable()` 会更清晰。

## 1.5 泛型、基本类型与空值

Java 的泛型擦除让匹配器在链式调用中经常出现类型推断问题。典型表现是编译器提示 `Object` 无法转换为目标类型，或者需要显式类型见证（type witness）。

### 1.5.1 显式指定泛型

```java
// 编译失败或得到不合适的推断时，可以在方法名前显式指定类型
when(repository.findById(ArgumentMatchers.<Long>any()))
        .thenReturn(Optional.empty());

// 集合参数同理
when(batchService.process(ArgumentMatchers.<List<Order>>any()))
        .thenReturn(1);
```

### 1.5.2 基本类型要用专用匹配器

```java
// 错误示范：any(Class) 对基本类型不直观，且可能引入装箱问题
when(counter.increment(any(Integer.class))).thenReturn(1);

// 正确做法：基本类型使用 anyInt、anyLong 等
when(counter.increment(anyInt())).thenReturn(1);
```

`anyInt()` 等匹配器返回的是基本类型默认值 `0`，不会参与装箱；在参数为 `int` 的方法中可以直接使用。反过来，`any(Integer.class)` 返回的是 `null`，传给 `int` 参数会在运行时拆箱失败。

### 1.5.3 空值匹配

```java
// 只匹配 null
when(repository.findByEmail(isNull())).thenReturn(Optional.empty());

// 匹配 String 或 null
when(repository.findByEmail(nullable(String.class)))
        .thenReturn(Optional.empty());

// 匹配非 null 的任意对象
when(repository.findByEmail(notNull())).thenReturn(Optional.empty());
```

`eq(null)` 在语法上可行，但语义不如 `isNull()` 明确，也容易和"参数忘记传"混淆。代码审查时看到 `eq(null)` 通常建议改成 `isNull()`。

## 1.6 数组、引用与反射相等

数组的 `equals` 是引用比较，因此 `eq(new int[]{1, 2})` 永远匹配不到内容相同的另一个数组。正确方式是 `aryEq`：

```java
// 错误示范：数组的 equals 比较的是引用
verify(repository).saveBytes(eq(new byte[]{1, 2, 3}));

// 正确做法：按内容逐项比较
verify(repository).saveBytes(aryEq(new byte[]{1, 2, 3}));
```

如果关心的是"同一个对象实例"，用 `same`：

```java
verify(listener).onEvent(same(event));
```

如果需要按字段反射比较两个对象，可以用 `refEq`，但它对嵌套对象、集合和自定义 `equals` 的处理有限，生产测试中不如 `argThat` 可控：

```java
// 按字段反射比较，忽略指定的字段
verify(repository).save(refEq(expected, "createdAt", "updatedAt"));
```

## 1.7 常见误用与排错

| 误用 | 后果 | 建议 |
|------|------|------|
| `any()` 与原始值混用 | `InvalidUseOfMatchersException` | 所有参数统一用匹配器 |
| 用 `anyString()` 匹配可能为 `null` 的参数 | 桩不命中，返回默认值 | 改用 `nullable(String.class)` |
| `argThat` 中不判空 | 参数为 `null` 时 NPE | 谓词第一行判空 |
| 匹配器写在 `when`/`verify` 之外 | 匹配器状态泄漏到下一次调用 | 只在 `when`、`verify` 参数位置使用 |
| 用 `eq` 比较数组 | 永远不相等 | 改用 `aryEq` |
| 在 `argThat` 里做副作用操作 | 匹配次数不确定，副作用重复执行 | 匹配器保持纯函数 |
| 匹配条件过宽 | 测试无法发现参数错误 | 用 `argThat` 精确表达关键字段 |

还有一个隐蔽问题：`argThat` 的谓词可能在一次调用中被执行多次（例如参数不匹配时的诊断重试），因此**谓词里不能有 `count++`、写日志、修改外部集合等副作用**。匹配器应该是纯函数，只依赖入参。

# 2. 练习与自检

1. 下面的调用为什么运行时报错：`when(repository.find(any(), "BOOK-1"))`？请改正。
2. 写一个 `argThat` 匹配器，要求订单金额大于 100 且币种为 `CNY`，并处理 `null` 参数。
3. 说明 `any()`、`any(String.class)` 和 `nullable(String.class)` 在空值语义上的差异。

# 3. 小结

参数匹配器让测试从"值必须完全相等"升级为"值符合业务规则"。使用时记住三条主线：`eq` 负责精确值，`any`/`nullable` 负责空值与类型范围，`argThat` 负责复杂谓词；同时严格遵守"全匹配器或全原始值"的混用规则。

| 场景 | 推荐写法 |
|------|----------|
| 精确匹配某个值 | `eq(value)` |
| 任意非空对象 | `any()` 或 `any(Class)` |
| 可能为空的字符串 | `nullable(String.class)` |
| 只匹配 null | `isNull()` |
| 复杂业务条件 | `argThat(predicate)` |
| 数组内容比较 | `aryEq(array)` |
| 基本类型 | `anyInt()` / `anyLong()` 等 |

下一篇 `4_verification.md` 进入行为验证，讨论 `verify`、调用次数、执行顺序、超时验证，以及为什么"验证得越多，测试反而越脆弱"。
