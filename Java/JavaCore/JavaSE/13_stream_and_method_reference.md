`Stream` 与**方法引用**是 Java 函数式编程的两块拼图：Stream 用"流水线"把"过滤 → 映射 → 收集"这类集合操作写成声明式的链条，方法引用则把现成的方法直接"塞给"函数式接口，替掉啰嗦的 Lambda。本篇在第 9 篇集合与第 12 篇 Lambda 的基础上，把 Stream 的**四大特点、惰性求值与一次性、中间/终结操作全表、`Collectors` 收集器体系、特化流、并行流与常见坑**讲透，再给出方法引用的四种形式与选择原则。集合基础、泛型、`Map`/`Set` 请回看第 9 篇，Lambda 基础语法见第 12 篇（本篇的方法引用部分更详细，第 12 篇只讲基础）。

# 1. Stream 是什么

## 1.1 从"怎么算"到"算什么"

传统集合遍历是**外部迭代**：由程序员写 `for` 循环，自己控制"怎么遍历"——下标从几开始、什么时候 `break`、结果放进哪个临时集合。Stream 是**内部迭代**：只声明"要做什么"（留下以 a 开头的、转成大写、收集起来），"怎么遍历"交给 JDK 去决定。

```java
List<String> words = List.of("apple", "banana", "avocado", "cherry");

// 外部迭代：自己写循环、自己维护临时集合
List<String> result1 = new ArrayList<>();
for (String w : words) {
    if (w.startsWith("a")) {                  // 怎么筛，由我控制
        result1.add(w.toUpperCase());
    }
}

// 内部迭代：只描述做什么，遍历方式交给 Stream
List<String> result2 = words.stream()
        .filter(w -> w.startsWith("a"))       // 留下以 a 开头的
        .map(String::toUpperCase)             // 转成大写
        .collect(Collectors.toList());        // 收集成 List

System.out.println(result2);                  // [APPLE, AVOCADO]
```

## 1.2 Stream 与集合的四大区别

| 维度 | 集合（`Collection`） | 流（`Stream`） |
|------|---------------------|----------------|
| 是否存储数据 | **存储**元素本身，是一个数据结构 | **不存储**，只描述对数据的处理过程 |
| 是否改变源对象 | 增删改会改变自身 | **不改变**源集合/数组，只产出新结果 |
| 求值时机 | 调用即执行 | **惰性求值**，遇到终结操作才执行 |
| 迭代方式 | 外部迭代，自己写 `for` | **内部迭代**，遍历由 JDK 负责 |

这四条里最容易被想歪的是"不存储、不改源"。更准确的心智模型是：**数据仍然老老实实待在原来的集合里，流只是一份"加工说明书"**——它不复制数据，也不搬动数据，只规定数据经过哪些环节、最后以什么形式产出。也正因为如此，源是 `List`、`Set` 还是数组都不影响下游写法，只要能生成流，后面的处理链条就是同一套。

## 1.3 惰性求值：没有终结操作就什么都不执行

中间操作**不会立即执行**，它只是往流水线上"挂一个环节"；只有终结操作出现时，整条流水线才会被一次性驱动起来，每个元素依次走完全部中间操作。

```text
  数据源                 中间操作（惰性，只搭管道）              终结操作（触发执行）

            ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
  words ─▶  │ stream() │─▶  │  filter  │─▶  │   map    │─▶  │ collect  │ ─▶ 结果
            └──────────┘    └──────────┘    └──────────┘    └──────────┘
              获取流           中间操作         中间操作         终结操作
                             └────── 到这里为止，一行都没执行 ──────┘
```

用几行代码就能验证这一点：只挂中间操作时，`println` 一次也不会打印。

```java
List<String> words = List.of("apple", "banana", "avocado");

// 只挂中间操作，没有终结操作：什么都不会输出
words.stream()
        .filter(w -> {
            System.out.println("filter 执行：" + w);      // 不会输出
            return w.startsWith("a");
        });

// 加上终结操作 forEach，整条流水线才开始跑
words.stream()
        .filter(w -> {
            System.out.println("filter 执行：" + w);      // 输出 apple / banana / avocado
            return w.startsWith("a");
        })
        .forEach(System.out::println);                     // 输出 apple / avocado
```

> **注意**：没有终结操作时，Stream 代码**能编译通过但一行也不执行**，这是新手最常见的"写了没效果"的坑。原因正是惰性求值——中间操作只是返回了一个描述性的新流，从未被驱动。另一个常见表现是"调试断点打不进去""日志没打印"，本质相同。

惰性还带来一个额外好处：**短路**。`limit(n)`、`findFirst()`、`anyMatch()` 这类操作不需要遍历完整个数据源，拿到够用的元素就立即停止。

```java
// 只要第一个以 "a" 开头的词，后面的元素根本不会被 filter 处理
Optional<String> first = words.stream()
        .filter(w -> w.startsWith("a"))
        .findFirst();
System.out.println(first.orElse("无"));   // apple
```

## 1.4 流的一次性

Stream 只能被消费一次。一旦执行过终结操作，这个流对象就"关闭"了，再对它做任何操作都会抛 `IllegalStateException`。

```java
Stream<String> stream = List.of("a", "b", "c").stream();
stream.forEach(System.out::println);   // 正常：a b c
stream.count();                        // ✗ 抛 IllegalStateException: stream has already been operated upon or closed
```

> **陷阱**：把流存成变量、或把流当参数在方法之间来回传递，很容易"不小心用了两次"。正确做法是**链式编程一口气写完**；确实需要复用，就重新从数据源调一次 `stream()`——集合本身可以被反复 `stream()`，只有**流对象**不能复用。

# 2. 获取 Stream

| 数据源 | 方式 | 说明 |
|--------|------|------|
| 单列集合 | `Collection.stream()` | `list.stream()`、`set.stream()`，日常最常用 |
| 双列集合 | 先转单列再取流 | `map.keySet().stream()`（所有键）、`map.values().stream()`（所有值）、`map.entrySet().stream()`（所有键值对） |
| 数组 | `Arrays.stream(T[] a)` | 基本类型数组也支持，返回特化流如 `IntStream` |
| 零散值 | `Stream.of(1, 2, 3)` | 底层就是 `Arrays.stream`；传入**对象数组**时不会自动展开 |
| 无限流 | `Stream.generate(Supplier)` / `Stream.iterate(seed, f)` | 必须配 `limit()` 截断，否则永不结束 |

```java
int[] nums = {1, 2, 3, 4, 5};
IntStream intStream = Arrays.stream(nums);              // 基本类型数组 → IntStream（不装箱）
System.out.println(intStream.sum());                    // 15

// 无限流：generate 每次调用 supplier；iterate 在前一个值的基础上迭代
Stream<Integer> even = Stream.iterate(0, n -> n + 2).limit(5);
even.forEach(System.out::print);                        // 02468
```

> **注意**：`Map` 本身不是 `Collection`，没有 `stream()` 方法，必须经由 `keySet()` / `values()` / `entrySet()` 中转。这与第 9 篇讲的"`Map` 与 `Collection` 是两套体系"完全一致。

# 3. 中间操作

## 3.1 中间操作总览

所有中间操作**都返回一个新的 `Stream`**，返回值能继续 `.` 下去，所以叫"中间"。它们全都不碰数据，只往管道上加环节。

| 方法 | 是否有状态 | 作用 |
|------|-----------|------|
| `filter(Predicate)` | 无状态 | 保留满足条件的元素 |
| `map(Function)` | 无状态 | 一个元素 → 另一个元素（一对一转换） |
| `mapToInt` / `mapToLong` / `mapToDouble` | 无状态 | 转成特化流，避免装箱 |
| `flatMap(Function)` | 无状态 | 一个元素 → 一个流，再把所有流"拍平"（一对多） |
| `distinct()` | 有状态 | 去重，依赖 `hashCode()` + `equals()` |
| `sorted()` / `sorted(Comparator)` | 有状态 | 自然序 / 自定义比较器排序 |
| `limit(long n)` | 短路 | 只要前 n 个，后面的不再处理 |
| `skip(long n)` | 有状态 | 跳过前 n 个 |
| `peek(Consumer)` | 无状态 | 只做观察（常用于调试打印），不改变元素 |
| `Stream.concat(a, b)` | 静态方法 | 合并两个流，注意它是**静态方法**，写法是 `Stream.concat(a, b)` 而非 `a.concat(b)` |

**有状态**指该操作必须先看到全部（或部分）元素才能出结果，比如 `sorted()` 要等所有元素到齐才能排；**无状态**指每个元素各自独立处理即可。这个区分在并行流下很关键——有状态操作往往需要额外的缓冲和合并，代价更高（见第 7 节）。

## 3.2 map 与 flatMap

`map` 是一对一：`["a", "bb"]` 可以映射成长度 `[1, 2]`。`flatMap` 是一对多再拍平：把每个元素**换成一个流**，最后把所有流的内容连成一条。

```java
List<String> lines = List.of("hello world", "java stream");

// map：每个元素 → 一个 String[]，结果里保留的是数组（嵌套结构）
lines.stream()
        .map(s -> s.split(" "))
        .forEach(arr -> System.out.println(Arrays.toString(arr)));   // [hello, world] / [java, stream]

// flatMap：每个元素 → 一个 Stream，再拍平成单一的元素序列
List<String> words = lines.stream()
        .flatMap(s -> Arrays.stream(s.split(" ")))   // String[] → Stream<String>
        .collect(Collectors.toList());
System.out.println(words);                           // [hello, world, java, stream]
```

> **最佳实践**：只要出现"把集合的集合变成一个大集合"，就用 `flatMap`；只有需要保留嵌套结构时才用 `map`。`flatMap` 是嵌套 `for` 循环的声明式替代，写法上通常能消掉两层循环。

## 3.3 特化流与 mapToInt

`mapToInt` 把 `Stream<T>` 转成 `IntStream`，从而拿到 `sum()`、`average()`、`max()` 这些数值专属方法，同时**避免每个元素装箱成 `Integer`**。下面这个 `Student` 类在各节示例中反复使用，先给出定义。

```java
// 示例实体类：后续各节的示例都用它
class Student {
    private final String name;
    private final String clazz;
    private final int score;

    Student(String name, int score) { this(name, "一班", score); }

    Student(String name, String clazz, int score) {
        this.name = name;
        this.clazz = clazz;
        this.score = score;
    }

    String getName()  { return name; }     // 对应方法引用 Student::getName
    String getClazz() { return clazz; }
    int getScore()    { return score; }

    @Override
    public String toString() { return name + "(" + score + ")"; }
}
```

```java
List<Student> students = List.of(
        new Student("张三", 88), new Student("李四", 92), new Student("王五", 75));

int total = students.stream()
        .mapToInt(Student::getScore)     // Stream<Student> → IntStream（全程不装箱）
        .sum();
System.out.println("总分：" + total);      // 总分：255

IntSummaryStatistics stat = students.stream().mapToInt(Student::getScore).summaryStatistics();
System.out.println(stat.getMax() + " / " + stat.getMin() + " / " + stat.getAverage());
// 92 / 75 / 85.0
```

## 3.4 排序、去重、截断与调试

```java
List<String> data = List.of("banana", "apple", "cherry", "apple", "date");

data.stream()
        .distinct()                       // 去重：apple 只留一个
        .sorted()                         // 自然序（此处是字母序）
        .skip(1)                          // 跳过第一个
        .limit(2)                         // 只取两个
        .forEach(System.out::println);    // banana / cherry
```

`peek` 专门用来"看一眼流水线中间长什么样"，它不改变元素，是排查 Stream 问题最顺手的手段（等价于给管道装了个观察窗）。

```java
List.of(1, 2, 3, 4, 5).stream()
        .peek(n -> System.out.println("原始：" + n))
        .filter(n -> n % 2 == 0)
        .peek(n -> System.out.println("过滤后：" + n))
        .map(n -> n * 10)
        .forEach(n -> System.out.println("映射后：" + n));
// 原始：1  原始：2  过滤后：2  映射后：20  原始：3  原始：4  过滤后：4  映射后：40  原始：5
```

> **注意**：从输出的交错顺序能看出 Stream 是"**每个元素走完整条链路**"而不是"所有元素先一起过第一道、再一起过第二道"。这与写 `for` 循环的直觉不同，正是惰性求值的直接体现。

# 4. 终结操作

## 4.1 终结操作总览

终结操作返回**非流的结果**（`void`、基本类型、`Optional`、集合等）。一旦调用，流就被消费掉，不能再用。

| 方法 | 返回类型 | 作用 |
|------|----------|------|
| `forEach(Consumer)` | `void` | 遍历每个元素 |
| `count()` | `long` | 统计元素个数 |
| `max(Comparator)` / `min(Comparator)` | `Optional<T>` | 最大值 / 最小值 |
| `collect(Collector)` | 任意 | 收集成集合、`Map`、字符串等（最灵活） |
| `reduce(...)` | `Optional<T>` / `T` | 把元素逐个"折叠"成一个值 |
| `toArray()` | `Object[]` / `T[]` | 收集成数组，`toArray(T[]::new)` 可指定类型 |
| `anyMatch(Predicate)` | `boolean` | 是否**存在**满足条件的元素（短路） |
| `allMatch(Predicate)` | `boolean` | 是否**全部**满足（短路） |
| `noneMatch(Predicate)` | `boolean` | 是否**没有**满足条件的元素（短路） |
| `findFirst()` | `Optional<T>` | 找第一个，有序流下结果稳定 |
| `findAny()` | `Optional<T>` | 找任意一个，并行流下更快但结果不确定 |

## 4.2 遍历与统计：forEach / count / max / min

```java
List<Integer> nums = List.of(3, 1, 4, 1, 5, 9, 2, 6);

nums.forEach(System.out::print);                                        // 31415926
System.out.println(nums.stream().count());                              // 8
System.out.println(nums.stream().max(Integer::compareTo).orElse(-1));   // 9
System.out.println(nums.stream().min(Integer::compareTo).orElse(-1));   // 1
```

> **注意**：`max` / `min` 返回的是 `Optional` 而不是元素本身，因为**空流时根本没有最值可言**。用 `Optional` 作为返回类型是强制调用者正视这种可能性，而不是拿到一个 `null` 再踩空指针。`Optional` 的取值方式（`orElse` / `orElseGet` / `ifPresent` / `map` / `flatMap`）见第 28 篇，本篇只需记住"为什么返回 `Optional`"。

## 4.3 匹配与查找：anyMatch / allMatch / noneMatch / findFirst / findAny

```java
List<Integer> nums = List.of(3, 1, 4, 1, 5, 9, 2, 6);

System.out.println(nums.stream().anyMatch(n -> n > 8));    // true（存在 9）
System.out.println(nums.stream().allMatch(n -> n > 0));    // true（全为正数）
System.out.println(nums.stream().noneMatch(n -> n > 10));  // true（没有大于 10 的）
System.out.println(nums.stream().filter(n -> n % 2 == 0).findFirst().orElse(-1));  // 4
```

`anyMatch` / `noneMatch` / `findFirst` 都**短路**：一旦能确定答案就立即停止遍历，不像 `filter` 那样必须把数据走完。

> **注意**：`findAny()` 与 `findFirst()` 的差异只在并行流下才明显。`findFirst()` 必须保证"第一个"的语义，并行时要按原顺序合并各段结果，代价更高；`findAny()` 只要返回任意一个满足条件的元素即可，并行时更快。**串行流下两者结果通常相同，但不要依赖这一点写业务逻辑。**

## 4.4 reduce：把流"折叠"成一个值

`reduce` 有三个重载：

| 重载 | 签名 | 返回 | 说明 |
|------|------|------|------|
| 一参 | `reduce(BinaryOperator<T>)` | `Optional<T>` | 没有初始值，空流返回 `Optional.empty()` |
| 二参 | `reduce(T identity, BinaryOperator<T>)` | `T` | 有初始值，空流返回 `identity` |
| 三参 | `reduce(U identity, BiFunction<U,T,U>, BinaryOperator<U>)` | `U` | 类型可不同，**第三个参数只在并行流下用于合并各段结果** |

```java
List<Integer> nums = List.of(1, 2, 3, 4, 5);

// 一参：从第一个元素开始两两合并
Optional<Integer> sum1 = nums.stream().reduce((a, b) -> a + b);
System.out.println(sum1.orElse(0));                       // 15

// 二参：以 100 为初始值
int sum2 = nums.stream().reduce(100, Integer::sum);
System.out.println(sum2);                                 // 115

// 三参：初始值是 int，元素是 Integer，最后并行合并
int len = nums.stream().reduce(
        0,                                                // identity：int
        (acc, n) -> acc + String.valueOf(n).length(),     // 累加器：int + int
        Integer::sum);                                    // 并行下的合并器
System.out.println(len);                                  // 5（数字 1~5 各占一位）
```

> **注意**：`reduce` 的累加函数必须满足**结合律**——`(a ⊕ b) ⊕ c == a ⊕ (b ⊕ c)`，最好还满足可交换性。因为并行流会把数据切成多段、各段独立归约后再合并，不同分段方式会得到不同的运算顺序。减法、字符串顺序拼接、`(a, b) -> a` 这类不对称操作在并行流下会得到与串行**不同甚至错误**的结果。

```java
// ✗ 危险：减法不满足结合律，并行下结果不可预期
System.out.println(
        List.of(1, 2, 3, 4, 5, 6, 7, 8).stream().parallel().reduce((a, b) -> a - b).get());
// 串行得 -34；并行可能是 -34，也可能是别的值，取决于切分方式
```

## 4.5 collect 与需求选择

`collect` 是把流"倒回"集合的标准出口，具体行为由传入的 `Collector` 决定（详见第 5 节）。选择顺序可以记成一句话：**能用内置 `Collectors` 表达的，就不要手写 `reduce`**——内置收集器既好读，又已经保证了并行安全。

```java
List<String> list = words.stream().collect(Collectors.toList());     // 收成 List
Set<String> set   = words.stream().collect(Collectors.toSet());      // 收成 Set（去重）
String joined     = words.stream().collect(Collectors.joining(", ")); // 收成字符串
Map<String, List<Student>> byClass =
        students.stream().collect(Collectors.groupingBy(Student::getClazz));  // 收成 Map
```

# 5. Collectors 详解

`Collectors` 是与 `collect` 配套的工具类，方法分两类：**收集器**直接返回 `Collector`（如 `toList`）；**下游收集器**用于嵌套组合，作为 `groupingBy` / `partitioningBy` 的第二个参数。

## 5.1 收集成集合：toList / toSet / toCollection

| 收集器 | 结果 | 特点 |
|--------|------|------|
| `toList()` | `List` | 保留流的顺序，允许重复 |
| `toSet()` | `Set` | 自动去重，**不保证顺序** |
| `toCollection(Supplier)` | 任意 `Collection` | 需要 `TreeSet`（有序）、`ArrayList`（可改）等具体实现时用 |

```java
List<String> list  = words.stream().collect(Collectors.toList());
Set<String> set    = words.stream().collect(Collectors.toSet());
TreeSet<String> ts = words.stream().collect(Collectors.toCollection(TreeSet::new));  // 有序去重
```

> **注意**：`Collectors.toList()` 返回的 `List` **不承诺可修改**（规范只说"是 `List`"）。要往结果里继续增删，用 `Collectors.toCollection(ArrayList::new)`；JDK16 起也可以直接写 `stream.toList()`，它返回的是**不可变**列表，和 `Collectors.toList()` 不是一回事。

## 5.2 toMap 与 key 重复

| 重载 | 用途 |
|------|------|
| `toMap(k, v)` | 最常用，**key 重复时抛 `IllegalStateException`** |
| `toMap(k, v, mergeFn)` | 指定 key 冲突时的合并策略 |
| `toMap(k, v, mergeFn, mapSupplier)` | 再指定 `Map` 实现，如 `LinkedHashMap::new` 可保序 |

```java
List<Student> students = List.of(
        new Student("张三", 88), new Student("李四", 92));

Map<String, Integer> nameToScore = students.stream()
        .collect(Collectors.toMap(Student::getName, Student::getScore));
System.out.println(nameToScore);      // {张三=88, 李四=92}
```

**key 重复会直接抛异常**，这是最常见的 `toMap` 坑：

```java
List<String> words = List.of("a", "b", "a");

words.stream().collect(Collectors.toMap(w -> w, String::length));
// ✗ 抛 IllegalStateException: Duplicate key a (attempted merging values 1 and 1)

// ✓ 解法一：指定合并策略，冲突时保留后出现的值
Map<String, Integer> m1 = words.stream()
        .collect(Collectors.toMap(w -> w, String::length, (oldVal, newVal) -> newVal));
System.out.println(m1);   // {a=1, b=1}

// ✓ 解法二：改用分组，把重复 key 收成一个列表
Map<String, List<String>> m2 = words.stream().collect(Collectors.groupingBy(w -> w));
System.out.println(m2);   // {a=[a, a], b=[b]}
```

> **陷阱**：`toMap` 的 value 若来自一个**可能返回 `null`** 的方法，即使写了合并函数也会抛 `NullPointerException`——`Collectors.toMap` 底层走的是 `Map.merge`，而 `merge` 不接受 `null` 值。要么先 `.filter(Objects::nonNull)` 过滤掉，要么改用手写循环收集。

## 5.3 groupingBy 分组

`groupingBy` 按分类函数把元素分组，返回 `Map<分类, List<元素>>`，是最实用的收集器。

```java
List<Student> students = List.of(
        new Student("张三", "一班", 88),
        new Student("李四", "二班", 92),
        new Student("王五", "一班", 75));

// 一级分组：按班级分组
Map<String, List<Student>> byClass = students.stream()
        .collect(Collectors.groupingBy(Student::getClazz));
System.out.println(byClass.keySet());      // [一班, 二班]

// 二级分组：先按班级，再按"是否优秀"分组
Map<String, Map<String, List<Student>>> twoLevel = students.stream()
        .collect(Collectors.groupingBy(Student::getClazz,
                 Collectors.groupingBy(s -> s.getScore() >= 80 ? "优秀" : "普通")));
System.out.println(twoLevel);
// {一班={优秀=[张三(88)], 普通=[王五(75)]}, 二班={优秀=[李四(92)]}}

// 分组 + 下游收集器：只统计每个班的人数
Map<String, Long> countByClass = students.stream()
        .collect(Collectors.groupingBy(Student::getClazz, Collectors.counting()));
System.out.println(countByClass);          // {一班=2, 二班=1}
```

`groupingBy` 的第二个参数就是**下游收集器**：它决定"分完组之后，每组的元素再用什么收集器收一次"。正是这个参数撑起了分组统计、分组求平均、分组映射等各种组合。

## 5.4 partitioningBy

`partitioningBy` 是 `groupingBy` 的特例：按布尔条件只分成**两组**（`true` 组与 `false` 组）。它返回的 `Map` 的 key 一定是 `Boolean`，而且**两组一定都存在**——某一组没有元素时对应的值是空列表，而不是 `null`。

```java
Map<Boolean, List<Student>> passed = students.stream()
        .collect(Collectors.partitioningBy(s -> s.getScore() >= 80));
System.out.println(passed.get(true).size());    // 2
System.out.println(passed.get(false).size());   // 1
```

> **最佳实践**：只有"非此即彼"的双分类才用 `partitioningBy`，好处是取 `get(false)` 永远不会拿到 `null`；分类多于两个，或者分类键不是布尔值，就用 `groupingBy`。

## 5.5 joining

`joining` 把元素**拼成一个字符串**，只适用于 `Stream<String>`（其他类型要先 `map` 成字符串）。

| 重载 | 效果 |
|------|------|
| `joining()` | 元素直接首尾相接 |
| `joining(分隔符)` | 元素之间插入分隔符 |
| `joining(分隔符, 前缀, 后缀)` | 元素之间加分隔符，整体再加前后缀 |

```java
List<String> words = List.of("apple", "banana", "cherry");

System.out.println(words.stream().collect(Collectors.joining()));                // applebananacherry
System.out.println(words.stream().collect(Collectors.joining("-")));             // apple-banana-cherry
System.out.println(words.stream().collect(Collectors.joining(", ", "[", "]")));  // [apple, banana, cherry]
```

## 5.6 数值聚合：counting / summingInt / averagingInt / summarizingInt

这几个收集器用于"分组之后每组算个数 / 求和 / 求平均 / 综合统计"，最常作为 `groupingBy` 的下游收集器出现。

| 收集器 | 返回 | 作用 |
|--------|------|------|
| `counting()` | `Long` | 元素个数 |
| `summingInt(mapper)` | `Integer` | 求和（另有 `summingLong` / `summingDouble`） |
| `averagingInt(mapper)` | `Double` | 平均值 |
| `summarizingInt(mapper)` | `IntSummaryStatistics` | 一次拿到个数、和、最值、平均值 |

```java
// 每个班级的平均分
Map<String, Double> avgByClass = students.stream()
        .collect(Collectors.groupingBy(Student::getClazz,
                 Collectors.averagingInt(Student::getScore)));
System.out.println(avgByClass);     // {一班=81.5, 二班=92.0}

// 综合统计：个数、总和、最值、平均一次拿全
IntSummaryStatistics stat = students.stream()
        .collect(Collectors.summarizingInt(Student::getScore));
System.out.println(stat.getCount() + " / " + stat.getSum() + " / " + stat.getMax());
// 3 / 255 / 92
```

> **注意**：`summarizingInt` 比"循环里手动累加"更清晰，也比连续调 `count()`、`max()`、`average()` 更省——**它只遍历一次流**。需要多个统计量时应优先用它，而不是把同一个流算好几遍（何况流只能用一次，需要多个统计量时本来就只能靠它或 `summaryStatistics()`）。

## 5.7 下游收集器：mapping / reducing

下游收集器的作用是"在分组结果上再做一次转换或归约"，从而避免产出 `Map<班级, List<完整对象>>` 这种带着一大堆无用字段的结果。

```java
// mapping：分组后只保留姓名 → Map<班级, List<姓名>>
Map<String, List<String>> namesByClass = students.stream()
        .collect(Collectors.groupingBy(Student::getClazz,
                 Collectors.mapping(Student::getName, Collectors.toList())));
System.out.println(namesByClass);   // {一班=[张三, 王五], 二班=[李四]}

// reducing：分组后每组再归约一次（这里取每班最高分）
Map<String, Optional<Integer>> maxByClass = students.stream()
        .collect(Collectors.groupingBy(Student::getClazz,
                 Collectors.mapping(Student::getScore,
                         Collectors.maxBy(Integer::compareTo))));
System.out.println(maxByClass);     // {一班=Optional[88], 二班=Optional[92]}
```

# 6. 特化流：IntStream / LongStream / DoubleStream

`Stream<Integer>` 的每个元素都是一个 `Integer` **对象**，做算术时既要装箱又要拆箱，元素量大时这笔开销不容忽视。`IntStream` / `LongStream` / `DoubleStream` 存储的是基本类型，天生避开了这层包装，是 Stream 在性能敏感场景下的用法。

| 特化流 | 元素类型 | 额外能力 |
|--------|----------|----------|
| `IntStream` | `int` | `range`、`rangeClosed`、`sum`、`average`、`boxed` |
| `LongStream` | `long` | 同上 |
| `DoubleStream` | `double` | `sum`、`average`，**没有 `range`** |

| 方法 | 说明 |
|------|------|
| `IntStream.range(a, b)` | 生成 `[a, b)` 的整数流，**左闭右开** |
| `IntStream.rangeClosed(a, b)` | 生成 `[a, b]` 的整数流，**两端都含** |
| `mapToObj(mapper)` | 特化流 → `Stream<对象>` |
| `boxed()` | 特化流 → `Stream<包装类>` |
| `sum()` / `average()` / `max()` / `min()` | 数值专属聚合，`average()` 返回 `OptionalDouble` |

```java
System.out.println(IntStream.range(1, 10).sum());         // 45（1~9，不含 10）
System.out.println(IntStream.rangeClosed(1, 10).sum());   // 55（1~10，含 10）

// 特化流与普通流互转
List<Integer> boxed = IntStream.rangeClosed(1, 5).boxed().collect(Collectors.toList());
System.out.println(boxed);                                // [1, 2, 3, 4, 5]
```

> **最佳实践**：凡是"对一堆数字求和、求平均、取最值"，都优先用 `mapToInt(...)` 转成特化流再聚合——既躲开了装箱，又能直接用 `sum()` / `average()` 这些方法，代码往往还更短。

# 7. 并行流

## 7.1 并行流的底层

`parallelStream()` 或 `stream().parallel()` 把流标记为并行，JDK 随即用 **`ForkJoinPool.commonPool()`** 把数据切分、分派给多个工作线程处理，最后合并结果。这个公共池的**默认线程数是 `CPU 核数 - 1`，并且被整个 JVM 里所有并行流共享**。

```java
long count = LongStream.rangeClosed(1, 10_000_000)
        .parallel()                        // 标记为并行流
        .filter(n -> n % 2 == 0)
        .count();
System.out.println(count);                 // 5000000

// 数组并行排序：内部同样是 ForkJoin 的归并策略
int[] arr = {5, 2, 9, 1, 7};
Arrays.parallelSort(arr);
System.out.println(Arrays.toString(arr));  // [1, 2, 5, 7, 9]
```

## 7.2 什么时候不该用并行流

并行流不是"加了就快"的开关，下面几类场景**不适合**用：

| 场景 | 为什么不该用 |
|------|-------------|
| 数据量小 | 切分与线程调度本身有开销，常常超过计算收益；元素量在万级以下基本不必考虑 |
| 有阻塞 IO | 工作线程都卡在等 IO 上，`commonPool` 被占满会拖累全 JVM 其它并行任务 |
| 有状态的操作 | `sorted`、`distinct` 需要全局信息，并行下缓冲与合并代价高 |
| 有顺序要求 | 并行下 `forEach` 的输出顺序不确定，应改 `forEachOrdered`，或干脆串行 |
| 用非线程安全的收集器 | 多线程往同一个 `ArrayList` / 普通 `HashMap` 里塞数据会丢数据甚至报错 |

```java
// ✗ 危险：并行流 + 非线程安全的外部集合
List<Integer> bad = new ArrayList<>();
IntStream.rangeClosed(1, 1000).parallel().forEach(bad::add);   // 结果数量可能少于 1000，甚至抛异常

// ✓ 正确：用 collect 把合并工作交给 Stream 自己
List<Integer> good = IntStream.rangeClosed(1, 1000).parallel()
        .boxed().collect(Collectors.toList());
System.out.println(good.size());                               // 1000
```

> **注意**：`Collectors.toMap` 底层依赖 `HashMap` 合并，并行流下大结果集的合并代价高、也更容易出问题。并行收集优先用 `toList` / `toSet` / `groupingBy` 这些 JDK 已保证并行安全的收集器。一句话结论：**并行流是"锦上添花"而不是"万能加速"**，用之前先问自己三件事——数据量大不大、操作纯不纯（无副作用）、结果要不要顺序。

# 8. Stream 常见坑

| 坑 | 现象 | 正确做法 |
|----|------|----------|
| 流被用了两次 | 抛 `IllegalStateException` | 链式写完，或重新 `stream()` |
| 没有终结操作 | 代码不执行，且无编译错误 | 补上 `collect` / `forEach` 等终结操作 |
| 在流里修改外部集合 | 结果错乱、抛 `ConcurrentModificationException` | 用 `collect` 产出新集合，不改源数据 |
| `forEach` 里堆副作用 | 代码难读，并行下不安全 | 用 `map` + `collect` 表达"产出什么" |
| `toMap` key 重复 | 抛 `IllegalStateException` | 传合并函数，或改用 `groupingBy` |
| `toMap` 的 value 为 `null` | 抛 `NullPointerException` | 先 `filter(Objects::nonNull)` |
| 滥用并行流 | 比串行还慢，甚至结果出错 | 对照 7.2 的判断清单 |

```java
// ✗ 反例：副作用式写法——在 forEach 里往外部集合塞数据
List<String> result = new ArrayList<>();
words.stream().filter(w -> w.length() > 5).forEach(result::add);

// ✓ 正例：用 collect 直接描述"要产出什么"
List<String> result2 = words.stream()
        .filter(w -> w.length() > 5)
        .collect(Collectors.toList());
```

> **陷阱**：流里的 Lambda 应当**只做转换、不产生副作用**——不改外部变量、不写文件、不调远程接口。串行时副作用也许看不出问题，一旦改成并行流，顺序错乱与并发问题会立刻暴露，而且极难排查。

# 9. 方法引用

方法引用是 Lambda 的**进一步简化**：当 Lambda 体只是"原样调用一个已存在的方法"时，可以直接把这个方法"引用"过来，省掉参数列表。第 12 篇介绍了它的基本形式，本篇展开使用条件、四种形式与选择原则。

## 9.1 使用前提

四条必须同时满足：

1. 使用处必须是**函数式接口**（有且仅有一个抽象方法，标了 `@FunctionalInterface` 的更明确）；
2. 被引用的方法**已经存在**（JDK 自带的或自己写的都行）；
3. 被引用方法的**形参个数与类型、返回值类型**都要和抽象方法的签名匹配；
4. 被引用方法**能完成需求**（语义对得上，而不只是签名凑巧对得上）。

> **注意**：判定标准其实只有一条——**这个 Lambda 能不能原封不动地"换成方法名"**。能换就是方法引用，不能换就别硬凑。

## 9.2 四种形式

| 形式 | 写法 | 等价 Lambda | 示例 |
|------|------|-------------|------|
| 引用静态方法 | `类名::静态方法` | `x -> 类名.静态方法(x)` | `Math::abs`、`Integer::parseInt` |
| 引用特定对象的实例方法 | `对象::实例方法` | `() -> 对象.实例方法()` | `System.out::println`、`str::toUpperCase` |
| 引用特定类型的任意对象的实例方法 | `类名::实例方法` | `x -> x.实例方法()` | `String::length`、`Student::getName` |
| 引用构造方法 | `类名::new` | `x -> new 类名(x)` | `Student::new`、`ArrayList::new` |

第一种最直观——形参列表和返回值完全一致，直接把方法名搬过来就行。

```java
// Lambda：把字符串原样交给 Integer.parseInt
Function<String, Integer> f1 = s -> Integer.parseInt(s);
// 方法引用：省掉参数列表
Function<String, Integer> f2 = Integer::parseInt;
System.out.println(f2.apply("123"));   // 123
```

第二种是"**对象已经确定**，每次调用它的同一个方法"。

```java
String prefix = "hello";
Supplier<String> s1 = () -> prefix.toUpperCase();   // 对象 prefix 是固定的
Supplier<String> s2 = prefix::toUpperCase;          // 直接引用这个对象的实例方法
System.out.println(s2.get());                       // HELLO
```

第三种（**特定类型的任意对象**）最需要理解：`类名::实例方法` 里没有指定具体对象，所以**函数式接口的第一个参数充当调用者**，剩下的参数才是被调方法的实参。

```java
// Lambda：参数 s 既是"被调用的对象"，又提供了实参
Function<String, Integer> g1 = s -> s.length();
Function<String, Integer> g2 = String::length;      // 第一个参数 s 当接收者
System.out.println(g2.apply("java"));               // 4

// 两个参数时更明显：第一个是接收者，第二个才是实参
BiFunction<String, String, Boolean> h1 = (s, t) -> s.startsWith(t);
BiFunction<String, String, Boolean> h2 = String::startsWith;   // 等价于 s.startsWith(t)
System.out.println(h2.apply("stream", "str"));      // true
```

> **注意**：`System.out::println` 属于第二种（对象是 `System.out` 这个 `PrintStream` 实例），而不是第三种。判断标准很简单：`::` 左边**能当作一个变量或表达式**就是第二种，**只能是类型名**就是第三或第四种。

## 9.3 this:: 与 super::

当被引用的方法是**本类或父类的实例方法**时，可以显式写成 `this::方法名` 与 `super::方法名`。它们是第二种形式的特例，只不过对象分别是 `this` 和 `super`。

```java
class Printer {
    public void printAll(String s) { System.out.println("[父类]" + s); }
}

class SubPrinter extends Printer {
    @Override
    public void printAll(String s) { System.out.println("[子类]" + s); }

    public void demo() {
        Consumer<String> c1 = this::printAll;    // 调子类重写后的方法
        Consumer<String> c2 = super::printAll;   // 显式调父类方法
        c1.accept("A");   // [子类]A
        c2.accept("B");   // [父类]B
    }
}
```

> **注意**：`this::printAll` 在实例方法里可以简写成 `printAll`（编译器自动补上 `this.`），但 `super::printAll` 的 `super` **不能省略**——省掉就变成了引用本类的 `printAll`，语义完全不同。

## 9.4 构造方法与数组构造引用

`类名::new` 引用构造方法，编译器会依据**上下文期望的函数式接口签名**去挑选匹配的那个构造器。`数组类型[]::new` 则是数组构造引用，最常用于"按指定长度创建数组"或把流收集成数组。

```java
// 示例类：三个构造器，用来演示"同一个 ::new 按上下文挑不同构造器"
class Point {
    private final int x, y;
    Point()             { this(0, 0); }                 // 无参构造
    Point(int x)        { this(x, 0); }                 // 一参构造
    Point(int x, int y) { this.x = x; this.y = y; }     // 两参构造
    @Override public String toString() { return "(" + x + ", " + y + ")"; }
}

// 无参构造 → Supplier
Supplier<Point> s1 = Point::new;                        // 等价于 () -> new Point()
System.out.println(s1.get());                           // (0, 0)

// 一参构造 → Function
Function<Integer, Point> s2 = Point::new;                // 等价于 x -> new Point(x)
System.out.println(s2.apply(5));                        // (5, 0)

// 两参构造 → BiFunction
BiFunction<Integer, Integer, Point> s3 = Point::new;     // 等价于 (x, y) -> new Point(x, y)
System.out.println(s3.apply(3, 4));                     // (3, 4)

// 数组构造引用：按长度创建数组
Function<Integer, int[]> arrMaker = int[]::new;
int[] arr = arrMaker.apply(5);
System.out.println(arr.length);                                // 5

// 典型用途：把流收集成指定类型的数组
String[] array = Stream.of("a", "b", "c").toArray(String[]::new);
System.out.println(Arrays.toString(array));                    // [a, b, c]
```

## 9.5 方法引用与 Lambda 的选择原则

```java
list.forEach(s -> System.out.println(s));   // Lambda
list.forEach(System.out::println);          // 方法引用，完全等价
```

**可以转成方法引用的三种 Lambda 形态**：

| Lambda 形态 | 可转成的形式 |
|-------------|--------------|
| `x -> f(x)` | `类名::f`（静态方法） |
| `x -> x.f()` | `类名::f`（任意对象的实例方法） |
| `x -> new X(x)` | `X::new`（构造方法引用） |

**选择原则**：

- **能省则省，不能省别硬凑**。只有 Lambda 体"就是一个方法调用"时才转方法引用；Lambda 里还有判断、拼接、算术等额外逻辑，就保留 Lambda。
- 方法引用的唯一优势是**更简洁、更能表达"就是复用已有方法"的意图**；语义上与等价 Lambda **完全一致**，不存在性能差异。
- 两个例外要留意：引用**重载方法**时，具体匹配哪个重载由编译器按上下文推断，可读性可能反而下降；方法名本身语义不明（如 `x -> x.get()`）时，保留 Lambda 更清楚。

> **最佳实践**：优先写方法引用，因为它更短、更少出错；但当方法引用让读者必须回头翻签名才能看懂时，**果断退回 Lambda**。可读性优先于"少打几个字符"。

# 10. 小结

| 主题 | 一句话要点 |
|------|-----------|
| Stream 是什么 | **不存储数据、不改变源对象、惰性求值、内部迭代**；数据仍在原集合里，流只是一份加工说明书 |
| 惰性求值 | 中间操作只搭管道，**没有终结操作就一行也不执行**；`limit` / `findFirst` / `anyMatch` 还会短路 |
| 流的一次性 | 一个流只能消费一次，再用抛 `IllegalStateException`，链式写法最稳 |
| 中间操作 | `filter` / `map` / `mapToInt` / `flatMap` / `distinct` / `sorted` / `limit` / `skip` / `peek` / `concat` |
| 终结操作 | `forEach` / `count` / `max` / `min` / `collect` / `reduce` / `toArray` / `anyMatch` / `allMatch` / `noneMatch` / `findFirst` / `findAny` |
| `Collectors` | `toList` / `toSet` / `toMap`（key 重复抛异常，传合并函数或改 `groupingBy`）/ `groupingBy`（可二级嵌套）/ `partitioningBy` / `joining` / `counting` / `summingInt` / `averagingInt` / `summarizingInt` / `mapping` / `reducing` |
| `reduce` | 三种重载；累加函数**必须满足结合律**，否则并行流结果与串行不一致 |
| 特化流 | `IntStream` / `LongStream` / `DoubleStream` **避免装箱**；`range` 左闭右开、`rangeClosed` 两端都含，`boxed()` 转回 |
| 并行流 | 底层是 `ForkJoinPool.commonPool()`；数据量小、有阻塞 IO、有状态或顺序要求、用非线程安全收集器时**不要用** |
| 方法引用 | 前提是函数式接口 + 方法已存在 + 签名匹配；四种形式 `类名::静态方法` / `对象::实例方法` / `类名::实例方法` / `类名::new`；能转则转，看不懂就回退 Lambda |

`Optional` 的完整用法（`orElse` / `orElseGet` / `ifPresent` / `map` / `flatMap`）见第 28 篇；Lambda 与函数式接口的基础语法见第 12 篇；集合框架本身（`List` / `Set` / `Map` 的选型与底层结构）见第 9 篇。
