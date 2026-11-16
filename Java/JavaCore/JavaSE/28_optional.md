`Optional` 是 JDK 8 引入的容器类，它本身不解决任何 `null` 引发的崩溃，真正的作用是**把"这个值可能为空"从一句口头约定变成方法签名上看得见的类型信息**。这篇讲清它要解决的语义问题、创建与取值的全部方法、`orElse` 与 `orElseGet` 这类高频面试题的区别，以及一份明确的**反模式清单**——`Optional` 用错方向的代价，比不用它还大。

# 1. Optional 要解决的问题

## 1.1 null 带来的两宗罪

`null` 引用是 Tony Hoare 在 1965 年设计 ALGOL W 时引入的，他后来称之为自己的"**十亿美元错误**"。在 Java 里它造成两类问题：

**第一宗罪：`NullPointerException`（NPE）**。这是 Java 生产环境里最常见的一类线上异常，且它的问题不在"抛异常"本身，而在**它抛得太晚**——错误发生在调用链末端，根因却在很早以前某个返回了 `null` 的地方，排查时栈信息只能告诉你"谁挂了"，不能告诉你"谁给了 null"。

**第二宗罪：语义模糊**。"一个方法返回了 `null`，到底代表什么？"至少有五种可能：

| 返回 `null` 的可能含义 | 例子 |
|----------------------|------|
| 查无此数据 | `findUserById(999)` 没查到 |
| 查询出错了（被吞掉了异常） | 数据库连接失败，catch 后返回 `null` |
| 参数非法 | 传了 `id = -1`，直接返回 `null` |
| 还没初始化 | 延迟加载尚未触发 |
| 程序 bug | 忘了赋值 |

调用方**无法从签名上区分**这些情况，只能靠文档或读源码。`Optional` 至少把"我可能什么都没找到"这一种明确表达出来了。

## 1.2 Optional 的设计目的

`Optional<T>` 是一个**只能装 0 个或 1 个元素的容器**：要么包含一个非空值，要么是 `Optional.empty()`。它带来三个收益：

1. **语义显式化**——`Optional<User> findById(long id)` 一眼就能看出"可能查不到"；
2. **强制调用方处理空值**——拿到 `Optional` 就必须想清楚"空的时候怎么办"，编译器会替你把这条路堵住；
3. **消除样板代码**——多层 `if (x != null)` 的嵌套可以被一条链式调用替代。

> **注意**：`Optional` **不是**用来消灭 `null` 的，`get()`、`orElse(null)` 里依然有 `null`，字段、参数、集合元素里依然到处都是 `null`。它只解决一个很具体的场景：**方法返回值的"可能没有"如何表达**。把它用在别处，收益立刻变成负的。

# 2. 创建 Optional 对象

| 方法 | 传入 `null` 时 | 适用场景 |
|------|---------------|---------|
| `Optional.of(T value)` | **立即抛 `NullPointerException`** | 你**确定**值非空（比如刚从 `new` 出来的对象） |
| `Optional.ofNullable(T value)` | 返回 `Optional.empty()` | 值**可能为 null**，最常用、最安全 |
| `Optional.empty()` | —— | 明确返回"空"，相当于 `Optional.ofNullable(null)` |

```java
import java.util.Optional;

public class CreateDemo {
    public static void main(String[] args) {
        // of：值确定非空时用；传 null 会当场 NPE，属于"快速失败"
        Optional<String> a = Optional.of("Hello");
        System.out.println(a);                      // Optional[Hello]

        // ofNullable：值来自外部（数据库、参数、Map.get）时的标准写法
        Optional<String> b = Optional.ofNullable(null);
        System.out.println(b);                      // Optional.empty

        // empty：明确表达"什么都没有"
        Optional<String> c = Optional.empty();
        System.out.println(c == Optional.empty());  // true（empty() 返回同一个单例实例）

        System.out.println(Optional.ofNullable("x").equals(Optional.of("x")));  // true

        try {
            Optional.of(null);
        } catch (NullPointerException e) {
            System.out.println("Optional.of(null) 直接抛 NPE");   // 会打印这行
        }
    }
}
```

> **注意**：`Optional.of(null)` 是纯粹的 bug 写法——你既然能写出 `null` 字面量，就说明这个位置本来就不该有值，直接用 `Optional.empty()` 更明确。`of(null)` 唯一的价值是"**断言这里绝不为空**"，作为一道快速失败的校验。

# 3. 判断值是否存在

## 3.1 isPresent / isEmpty

| 方法 | 版本 | 说明 |
|------|------|------|
| `boolean isPresent()` | JDK 8 | 是否有值 |
| `boolean isEmpty()` | **JDK 11+** | 是否为空，`isPresent()` 的反义词，可读性更好 |

```java
Optional<String> opt = Optional.ofNullable("hello");

if (opt.isPresent()) {          // 老写法：本质还是 null 判断，只换了个壳
    System.out.println(opt.get());
}

if (opt.isEmpty()) {            // JDK 11+，语义更清晰
    System.out.println("没有值");
}
```

> **陷阱**：`if (opt.isPresent()) { opt.get(); }` 这种写法等于**绕了一圈又回到 `if (x != null)`**，`Optional` 的价值完全没体现，反而多了一次装箱开销。`isPresent()` 的正确用途只有两个：**作为布尔结果参与复杂判断**，或者**在 `isPresent() && isEmpty()` 这类需要显式分支的边界场景**。其余情况一律优先用第 3.2 节的 `ifPresent` 系列。

## 3.2 ifPresent / ifPresentOrElse

| 方法 | 版本 | 说明 |
|------|------|------|
| `void ifPresent(Consumer<? super T> action)` | JDK 8 | 有值就执行 `action`，没值什么都不做 |
| `void ifPresentOrElse(Consumer<? super T> action, Runnable emptyAction)` | **JDK 9+** | 有值执行前者，没值执行后者 |

```java
Optional<String> opt = Optional.ofNullable("hello");

opt.ifPresent(v -> System.out.println("值: " + v));      // 值: hello
opt.ifPresent(System.out::println);                      // 方法引用写法：hello

// JDK 9+：两个分支一次写完，"空"分支不再被忽略
opt.ifPresentOrElse(
        v -> System.out.println("处理: " + v),           // 处理: hello
        () -> System.out.println("值为空，走降级逻辑")
);

Optional<String> empty = Optional.empty();
empty.ifPresentOrElse(
        v -> System.out.println("处理: " + v),
        () -> System.out.println("值为空，走降级逻辑")      // 会打印这行
);
```

`ifPresentOrElse` 的意义在于**它让"空"分支也必须被写出来**。用 `ifPresent` 时很容易忘了处理空的情况，而 `ifPresentOrElse` 的方法签名强迫你同时给出两条路。

# 4. 获取值

## 4.1 get()：不推荐直接用

```java
T get()   // 有值返回值；为空抛 NoSuchElementException
```

`get()` 是 `Optional` 里最"危险"的方法——它把 `NoSuchElementException` 换了个名字，**你依然要先用 `isPresent()` 判断才能安全调用**，那还不如直接用 `null` 判断。JDK 10 起它在 Javadoc 里被明确标注为"不鼓励使用"。

## 4.2 orElse 与 orElseGet：高频面试题

这是 `Optional` 里最容易踩的一个坑，区别只有一句话：**`orElse` 的参数无论如何都会被求值，`orElseGet` 的参数只在为空时才求值**。

| 方法 | 签名 | 求值时机 |
|------|------|---------|
| `orElse` | `T orElse(T other)` | **立即求值**——参数是值，方法调用之前就算好了 |
| `orElseGet` | `T orElseGet(Supplier<? extends T> supplier)` | **惰性求值**——只有为空时才调用 `supplier.get()` |
| `orElseThrow` | `T orElseThrow()` / `T orElseThrow(Supplier<? extends X>)` | 为空时抛异常 |

```java
import java.util.Optional;

public class OrElseVsOrElseGet {
    public static void main(String[] args) {
        Optional<String> hasValue = Optional.of("真实值");

        // ✗ orElse：参数在进入方法前就已经算好了，哪怕有值也白算一次
        String r1 = hasValue.orElse(createDefault());
        System.out.println("orElse 结果: " + r1);

        // ✓ orElseGet：有值时 supplier 根本不会被调用
        String r2 = hasValue.orElseGet(() -> createDefault());
        System.out.println("orElseGet 结果: " + r2);
    }

    // 模拟一个"有开销 / 有副作用"的默认值构造
    private static String createDefault() {
        System.out.println(">>> createDefault() 被调用了");
        return "默认值";
    }
}
```

```text
>>> createDefault() 被调用了
orElse 结果: 真实值
orElseGet 结果: 真实值
```

输出说明了一切：`Optional` 明明有值，`createDefault()` **还是被执行了一次**，它的返回值随即被丢弃。

| 场景 | `orElse` | `orElseGet` |
|------|----------|-------------|
| 默认值是**常量 / 字面量** | ✓ `orElse("未知")`，开销为零 | 可用但没必要，多一层 Lambda |
| 默认值要**查库、构造对象、发请求** | ✗ **每次调用都会白跑一次** | ✓ **唯一正确选择** |
| 默认值构造成本高但调用极少为空 | ✗ 每次都付出代价 | ✓ 只在少数情况付出代价 |

> **陷阱**：如果 `createDefault()` 带有**副作用**（写日志、改计数器、分配资源），用 `orElse` 会导致**副作用被莫名其妙地多执行一次**，而且异常栈里看不出原因。这类 bug 极难定位，所以**凡是默认值需要计算，一律用 `orElseGet`**，把 `orElse` 留给常量和字面量。

## 4.3 orElseThrow：参数校验的标准姿势

```java
// JDK 10+：无参版本，为空时抛 NoSuchElementException
T orElseThrow()

// JDK 8 起：自定义异常
<X extends Throwable> T orElseThrow(Supplier<? extends X> exceptionSupplier) throws X
```

```java
// 写法一（JDK 10+）：只需要"失败了就抛默认异常"
String value = Optional.ofNullable(map.get(key)).orElseThrow();

// 写法二：指定有业务含义的异常与消息
public User getUser(long id) {
    return userRepository.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("用户不存在: id=" + id));
}
```

> **最佳实践**：`orElseThrow` 是 `Optional` 在**参数校验 / 数据兜底**里最有价值的用法——"找不到就抛业务异常"这个语义被压缩成一行，且**抛的异常类型由你决定**，比 `get()` 抛出的含义贫瘠的 `NoSuchElementException` 好得多。注意 JDK 10 的无参版本只是 JDK 8 那个版本的语法糖：`orElseThrow()` ≡ `orElseThrow(NoSuchElementException::new)`。

## 4.4 or：链式降级（JDK 9+）

```java
Optional<T> or(Supplier<? extends Optional<? extends T>> supplier)
```

`or` 是"多个数据源依次尝试"的写法——值存在就返回自身，不存在才去问下一个：

```java
// 先查缓存，缓存没有查本地，本地没有查远程
Optional<Config> config = cache.get(key)
        .or(() -> localStore.get(key))
        .or(() -> remote.fetch(key));

System.out.println(config.orElseThrow(() -> new IllegalStateException("配置缺失: " + key)));
```

注意它和 `orElseGet` 的区别：**`orElseGet` 的 supplier 返回的是值 `T`，而 `or` 的 supplier 返回的是另一个 `Optional<T>`**——所以 `or` 可以无限嵌套地往下接，不会破坏链式结构。

# 5. 值转换与过滤

## 5.1 map：普通转换

```java
<U> Optional<U> map(Function<? super T, ? extends U> mapper)
```

值存在就应用 `mapper` 转成新值并包成新的 `Optional`；值不存在就直接返回 `Optional.empty()`——**关键点在于 `mapper` 根本不会被调用**，所以 `map` 里的 `s -> s.length()` 不用担心 NPE。

```java
Optional<String> name = Optional.of("Hello");
Optional<Integer> len = name.map(String::length);
System.out.println(len);                            // Optional[5]

Optional<String> empty = Optional.empty();
System.out.println(empty.map(String::length));      // Optional.empty（length() 没被调用）
```

## 5.2 flatMap：避免嵌套 Optional

`flatMap` 与 `map` 的差别是**传入函数的返回类型**：`map` 的 `mapper` 返回普通值 `U`，`flatMap` 的 `mapper` 返回 `Optional<U>`。

**为什么需要它？** 假设链式调用里每一步的方法本身就返回 `Optional`：

```java
// 假设有个方法返回 Optional<Address>
Optional<Address> getAddress() { ... }

// ✗ 用 map：结果是 Optional<Optional<Address>> —— 嵌套了！
Optional<Optional<Address>> bad = Optional.of(user).map(User::getAddress);

// ✓ 用 flatMap：自动"拍平"，结果就是 Optional<Address>
Optional<Address> good = Optional.of(user).flatMap(User::getAddress);
```

对照示例：

```java
import java.util.Optional;

public class MapVsFlatMap {
    public static void main(String[] args) {
        // map：把 String 转成它的长度（普通值）
        System.out.println(Optional.of("hello").map(String::length));      // Optional[5]

        // flatMap：mapper 自己返回 Optional，必须用 flatMap 才不会嵌套
        System.out.println(Optional.of("123").flatMap(MapVsFlatMap::parseInt));   // Optional[123]
        System.out.println(Optional.of("abc").flatMap(MapVsFlatMap::parseInt));   // Optional.empty
    }

    // 一个"解析失败就返回空"的方法，返回类型天然就是 Optional
    private static Optional<Integer> parseInt(String s) {
        try {
            return Optional.of(Integer.parseInt(s));
        } catch (NumberFormatException e) {
            return Optional.empty();
        }
    }
}
```

判断用哪个的方法：**看 `mapper` 的返回类型**——返回 `Optional` 就用 `flatMap`，返回普通值就用 `map`。链式调用中混用非常常见：

```java
// map 与 flatMap 交替：getAddress / getManager 返回 Optional，getName 返回普通值
public Optional<String> getManagerName(Employee emp) {
    return Optional.ofNullable(emp)
            .flatMap(Employee::getDepartment)   // 返回 Optional<Department>
            .flatMap(Department::getManager)    // 返回 Optional<Employee>
            .map(Employee::getName);            // 返回普通 String
}
```

## 5.3 filter：条件过滤

```java
Optional<T> filter(Predicate<? super T> predicate)
```

有值且满足条件返回自身，否则返回 `Optional.empty()`。**它不会抛异常，只是把不满足条件的值"变成空"**：

```java
Optional<String> opt = Optional.of("hello");

System.out.println(opt.filter(s -> s.length() > 3));    // Optional[hello]
System.out.println(opt.filter(s -> s.length() > 10));   // Optional.empty

// 多个 filter 串联 = 逻辑与
Optional<String> pwd = Optional.ofNullable("Abc12345");
boolean valid = pwd.filter(p -> p.length() >= 8)
                   .filter(p -> p.matches(".*[A-Z].*"))
                   .filter(p -> p.matches(".*\\d.*"))
                   .isPresent();
System.out.println("密码合法: " + valid);               // 密码合法: true
```

## 5.4 stream：Optional 转流（JDK 9+）

```java
Stream<T> stream()   // 有值 → 单元素 Stream；无值 → 空 Stream
```

这个方法的杀手级用途是**批量筛选**：把一批 `Optional` 里"有值的"抽出来，一次遍历搞定。

```java
import java.util.List;
import java.util.Optional;
import java.util.stream.Collectors;

public class StreamDemo {
    public static void main(String[] args) {
        List<Optional<String>> opts = List.of(
                Optional.of("a"), Optional.empty(), Optional.of("b"), Optional.empty());

        // 传统做法：先 filter(isPresent) 再 map(get)，两步还带一次 get()
        List<String> old = opts.stream()
                .filter(Optional::isPresent)
                .map(Optional::get)                 // 这里 get() 因为前面过滤过才安全
                .collect(Collectors.toList());

        // JDK 9+：stream() 一步到位，空的那个直接不产生元素
        List<String> now = opts.stream()
                .flatMap(Optional::stream)          // 拍平成 Stream<String>
                .collect(Collectors.toList());

        System.out.println(old);   // [a, b]
        System.out.println(now);   // [a, b]
    }
}
```

# 6. Optional 在 Stream 中的典型用法

`Stream` 里有几个终结操作**返回类型就是 `Optional`**——因为"流可能为空"是常态：

| Stream 方法 | 返回类型 | 说明 |
|------------|---------|------|
| `findFirst()` / `findAny()` | `Optional<T>` | 可能一个元素都没有 |
| `max(Comparator)` / `min(Comparator)` | `Optional<T>` | 空流没有最大 / 最小值 |
| `reduce(BinaryOperator)` | `Optional<T>` | 空流没有归约结果 |
| `average()`（`IntStream` 等） | `OptionalDouble` | 空流求不出平均值 |

```java
import java.util.*;
import java.util.stream.*;

public class StreamOptionalDemo {
    public static void main(String[] args) {
        List<User> users = List.of(new User("Alice", 30), new User("Bob", 17), new User("Carol", 25));

        // 找年龄最大的用户：max 返回 Optional，用 map 转成名字再给默认值
        String oldestName = users.stream()
                .max(Comparator.comparingInt(User::age))
                .map(User::name)                    // Optional<User> → Optional<String>
                .orElse("无用户");
        System.out.println("年龄最大: " + oldestName);    // 年龄最大: Alice

        // 空流上的 reduce 返回 Optional.empty，链式给出兜底
        Optional<Integer> sum = Stream.<Integer>empty().reduce(Integer::sum);
        System.out.println(sum.orElse(0));               // 0

        // 先按条件过滤，再取第一个 —— findFirst 天然带"可能没有"的语义
        users.stream()
             .filter(u -> u.age() > 100)
             .findFirst()
             .ifPresentOrElse(
                     u -> System.out.println("找到: " + u.name()),
                     () -> System.out.println("没有符合条件的用户"));   // 会打印这行
    }

    record User(String name, int age) {}
}
```

> **最佳实践**：`Stream` 返回 `Optional` 时，**不要**写 `.orElse(null)` 再回到 null 世界，也不要写 `.get()`；正确姿势是**继续用 `map` 往下转换，最后用一个 `orElse` / `orElseThrow` / `ifPresentOrElse` 收尾**，让整条链保持"空值被显式处理"的状态直到终点。

# 7. 使用示例

## 7.1 多层嵌套取值：传统 vs Optional

```java
// 传统写法：三层 if 嵌套，且每一层都可能漏判
public String getCity(User user) {
    if (user != null) {
        Address addr = user.getAddress();
        if (addr != null) {
            City city = addr.getCity();
            if (city != null) {
                return city.getName();
            }
        }
    }
    return "未知";
}

// Optional 写法：链式表达"逐层取值，任一层为空就返回默认值"
public String getCity(User user) {
    return Optional.ofNullable(user)
            .map(User::getAddress)      // getAddress 返回普通值，用 map
            .map(Address::getCity)      // getCity 也返回普通值
            .map(City::getName)
            .orElse("未知");
}
```

注意这里**用的是 `map` 而不是 `flatMap`**——因为 `getAddress()`、`getCity()` 返回的是普通对象而非 `Optional`。如果它们本身返回 `Optional`，就必须换成 `flatMap`，见 5.2。

## 7.2 四个典型场景

```java
// 场景一：仓储层查询"可能不存在"的数据 —— Optional 最正统的用法
public Optional<User> findById(long id) {
    return Optional.ofNullable(database.get(id));   // 用 ofNullable，null 转成 empty
}

// 调用方：直接链式处理，不接触 null
findById(1)
        .map(User::getName)
        .ifPresent(name -> System.out.println("用户名: " + name));

// 场景二：查配置，取不到用默认值
public String getConfig(String key) {
    return Optional.ofNullable(configMap.get(key)).orElse("defaultConfig");
}

// 场景三：一串校验条件，靠 filter 串起来
Optional<String> password = Optional.ofNullable(inputPassword);
boolean valid = password.filter(p -> p.length() >= 8)
                        .filter(p -> p.matches(".*[A-Z].*"))
                        .isPresent();

// 场景四：扁平化链式调用（每层都可能为空，且都返回 Optional）
public Optional<String> getManagerEmail(Employee emp) {
    return Optional.ofNullable(emp)
            .flatMap(Employee::getDepartment)
            .flatMap(Department::getManager)
            .map(Employee::getEmail);
}
```

## 7.3 实际案例：orElse 的坑导致缓存被击穿

```java
// ✗ 错误写法：cache.get(key) 明明命中，refreshCache() 还是每次都被调用
//   如果 refreshCache 是"清空缓存并回源"，这行代码会把刚命中的缓存立刻打掉
public Value getValue(String key) {
    return cache.get(key).orElse(refreshCache(key));
}

// ✓ 正确写法：只有真的没命中才回源
public Value getValue(String key) {
    return cache.get(key).orElseGet(() -> refreshCache(key));
}
```

这类 bug 的特点正是**结果看起来是对的，只是性能莫名其妙地差**（或多了一次写操作），日志里毫无线索。这也是为什么 `orElse` / `orElseGet` 会成为面试高频题。

# 8. 反模式清单

`Optional` 的设计者（Brian Goetz）在 Stack Overflow 上给过一条权威回答：**`Optional` 的首要用途是作为方法的返回类型**。除此之外的每一种用法都需要有极强理由。

| 反模式 | 为什么错 |
|--------|---------|
| **用作类的字段** | `Optional` **没有实现 `Serializable`**，序列化直接失败；且每个字段多包一层对象，堆占用成倍增加；`null` 字段本来就是合法的，没必要包 |
| **用作方法参数** | 调用方仍然可以传 `null` 进来，你反而要写成 `param == null ? Optional.empty() : param`，凭空多一层；参数校验用 `Objects.requireNonNull` 更直接 |
| **用 `Optional` 包集合** | `Optional<List<User>>` 是重复表达——**空集合本身就完美表达了"没有"**，返回 `Collections.emptyList()` 即可 |
| **`isPresent()` + `get()`** | 绕了一圈又回到 `if (x != null) x.xxx()`，只是多了两次方法调用和一次对象创建，**等于没摆脱 null 判断** |
| **不检查就 `get()`** | `get()` 抛的 `NoSuchElementException` 比 NPE 只差一个异常类型，语义还不如 NPE 清楚；应改 `orElseThrow(Supplier)` 指定业务异常 |
| **`Optional.of(null)`** | 立即抛 NPE，毫无意义；想要空值用 `Optional.empty()` |
| **在集合 / 数组里放 `Optional`** | 元素级别多一层装箱，内存与 GC 开销显著上升；用 `Optional.stream()` 拍平或直接过滤更合适 |
| **getter 返回 `Optional` 又在别处 `get()`** | 一半代码遵守约定、一半代码破坏约定，比全部不用 `Optional` 更糟糕 |
| **`Optional` 里再套 `Optional`** | 说明该用 `flatMap` 却用了 `map`，见 5.2 |

```java
// ✗ 反模式集中演示
public class BadUsage {
    private Optional<String> name;                    // ✗ 用作字段

    public void process(Optional<String> param) {     // ✗ 用作参数
        if (param.isPresent()) {
            String v = param.get();                   // ✗ isPresent + get
        }
        Optional<String> name = getName();
        System.out.println(name.get());               // ✗ 不检查就 get
    }

    public Optional<List<String>> getTags() {         // ✗ 用 Optional 包集合
        return Optional.of(Collections.emptyList());
    }
}

// ✓ 正确写法
public class GoodUsage {
    private String name;                              // 字段就用普通类型，允许 null

    public Optional<String> findName() {              // ✓ 只用作方法返回值
        return Optional.ofNullable(name);
    }

    public List<String> getTags() {                   // ✓ 空集合表达"没有"
        return Collections.emptyList();
    }

    public void process(String param) {               // ✓ 参数用 requireNonNull 校验
        String v = Objects.requireNonNull(param, "param 不能为 null");
    }
}
```

> **行业实践**：团队里推行 `Optional` 务必**只推"方法返回值"这一条**，其余用法一律禁止。实践中最常见的失败模式是"仓储层返回 `Optional`，Service 层 `get()` 出来继续传 `null`"——这不但没解决问题，还多了一层包装。要么整条链路都用，要么干脆别用。

# 9. 性能开销与基本类型 Optional

## 9.1 性能开销

`Optional` 是一个普通对象（内部只有一个 `final T value` 字段），所以每次调用返回 `Optional` 都是**一次对象分配**：

| 开销来源 | 说明 |
|----------|------|
| 对象分配 | 每次返回值都 `new` 一个 `Optional`（`Optional.empty()` 是单例，不分配） |
| 装箱拆箱 | `Optional<Integer>`、`Optional<Long>` 等会额外产生包装对象，`Integer` 缓存只覆盖 `-128~127` |
| 逃逸分析 | JIT 在对象不逃逸时能栈上分配甚至消除分配，但**不适合依赖它**——热点代码里 `Optional` 大量使用仍会增加 GC 压力 |

结论是：**`Optional` 适合用在方法边界（返回值），不适合用在循环体内的热路径**。在一个循环一百万次的代码里创建一百万个 `Optional`，性能损失是可测量的。

## 9.2 基本类型 Optional

为避免装箱开销，JDK 8 同时提供了三个基本类型版本：

| 类 | 内部字段 | 取值方法 | 额外的专用方法 |
|----|---------|---------|---------------|
| `OptionalInt` | `int value` | `getAsInt()` | `orElse(int)`、`orElseGet(IntSupplier)`、`ifPresent(IntConsumer)` |
| `OptionalLong` | `long value` | `getAsLong()` | 同上 |
| `OptionalDouble` | `double value` | `getAsDouble()` | 同上 |

```java
import java.util.OptionalInt;
import java.util.stream.IntStream;

OptionalInt opt = OptionalInt.of(42);
System.out.println(opt.getAsInt());              // 42
System.out.println(opt.orElse(0));               // 42
opt.ifPresent(v -> System.out.println(v));       // 42
System.out.println(OptionalInt.empty().orElse(0));  // 0

// 与 Stream 结合：IntStream.max() 返回的就是 OptionalInt，避免 int → Integer 装箱
int max = IntStream.of(3, 1, 4, 1, 5).max().orElse(-1);
System.out.println("最大值: " + max);            // 最大值: 5
```

**为什么不推荐主动使用？** 有三个原因：

1. **没有 `map` / `flatMap` / `filter` / `or`**——链式能力几乎为零，写起来比 `Optional<Integer>` 更笨重；
2. **不能和泛型 API 互通**——`OptionalInt` 不是 `Optional<Integer>`，传给任何接收 `Optional<T>` 的方法都要手动转换；
3. **它出现的位置是 `Stream` 的返回值**（`IntStream.max()`、`average()` 等），这是 JVM 已经替你决定好的地方，**你只需要会解析它就行**，不需要在业务代码里主动创建。

> **注意**：业界对这两个类的评价是负面的——Brian Goetz 也承认"如果重来一次，可能不会加上它们"。**实践建议：业务代码只用 `Optional<T>`，遇到 `OptionalInt` 这类 API 返回值时用 `orElse` 就地转换成基本类型，不要把它继续往上层传。**

# 10. 小结

| 主题 | 一句话要点 |
|------|-----------|
| 解决什么 | 把"返回值可能为空"从口头约定变成类型信息；**不消灭 `null`，只表达语义** |
| 创建 | `of` 传 null 抛 NPE（确定非空时用）；`ofNullable` 最常用；`empty` 明确返回空 |
| 取值 | `get()` 不推荐；**`orElse` 参数必求值，`orElseGet` 惰性求值**；校验场景用 `orElseThrow(Supplier)` |
| 转换 | `map` 用于普通转换，`flatMap` 用于 mapper 返回 `Optional` 的场景（避免 `Optional<Optional<T>>`）；`filter` 把不满足条件的值变成空 |
| JDK 9+ | `ifPresentOrElse` 双分支、`stream()` 转流做批量过滤、`or` 做多数据源链式降级、`isEmpty`（JDK 11） |
| 与 Stream | `findFirst` / `max` / `min` / `reduce` 返回 `Optional`，用 `map` 继续转换、`orElse*` 收尾 |
| 反模式 | 不做字段、不做参数、不包集合、不写 `isPresent() + get()`、不在集合里放 `Optional` |
| 性能 | 每次返回都是一次对象分配，适合方法边界不适合循环热路径；`OptionalInt` 系列不推荐主动使用 |

下一篇（第 29 篇）进入 Java 8 之后的新特性，看 `var`、Switch 表达式、文本块、`Record`、密封类、模式匹配与**虚拟线程**如何一步步改变 Java 的写法；`Stream` 与 Lambda 的基础见第 13 篇，`Record` 的详细展开见第 26 篇。
