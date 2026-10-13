`Stream` 与**方法引用**是 Java 函数式编程的两块拼图：Stream 用"流水线"把集合操作变成声明式的链条（过滤→映射→收集），方法引用则把现成的方法直接"塞给"函数式接口，替代啰嗦的 Lambda。本篇聚焦上篇（第 9 篇）集合之外的高级话题——Stream 的获取/中间/终结操作与惰性求值哲学，以及方法引用的四种形式。集合基础、泛型、Map/Set 请回看第 9 篇。

# 1. Stream 流

## 1.1 为什么用 Stream

对集合做"过滤 + 转换 + 收集"这类链式处理时，传统写法要写多个 `for` + `if` 临时变量；Stream 用一套声明式 API 一步到位，且**不存储数据、不修改源数据、惰性求值、内部迭代**。

```java
// 传统写法
List<String> result = new ArrayList<>();
for (String s : list) {
    if (s.startsWith("a")) result.add(s.toUpperCase());
}
// Stream 写法
List<String> result = list.stream()
    .filter(s -> s.startsWith("a"))
    .map(String::toUpperCase)
    .collect(Collectors.toList());
```

## 1.2 获取 Stream

| 数据源 | 方式 |
|--------|------|
| 单列集合 | `Collection.stream()`（`list.stream()`） |
| 双列集合 | 不能直接取，先转单列：`map.keySet().stream()` / `map.entrySet().stream()` / `map.values().stream()` |
| 数组 | `Arrays.stream(T[] a)`（基本类型数组也支持） |
| 多个值 | `Stream.of(1, 2, 3)`（数组需是引用类型） |

## 1.3 中间操作（返回新 Stream，惰性）

| 方法 | 说明 |
|------|------|
| `filter(Predicate)` | 过滤 |
| `limit(long n)` | 取前 n 个 |
| `skip(long n)` | 跳过前 n 个 |
| `distinct()` | 去重（依赖 hashCode + equals） |
| `map(Function)` | 转换元素类型 |
| `sorted()` / `sorted(Comparator)` | 排序 |
| `concat(Stream a, Stream b)` | 合并两个流 |

★ **关键认知**：中间操作**不立即执行**，只是搭"管道"；只有遇到**终结操作**才会真正遍历数据、一次性走完所有中间操作（**惰性求值**）。

## 1.4 终结操作（触发计算）

| 方法 | 说明 |
|------|------|
| `forEach(Consumer)` | 遍历 |
| `count()` | 统计个数 |
| `toArray()` | 收集到数组 |
| `collect(Collector)` | 收集到集合（配 `Collectors`） |

常用 `Collectors`：`toList()`、`toSet()`、`toMap(k, v)`、`groupingBy(分类)`、`joining(分隔符)`、`counting()`、`summarizingInt()`。

## 1.5 使用注意

- **一次性**：中间方法返回的是新 Stream，原 Stream 只能消费一次，建议链式编程
- **不改源数据**：Stream 处理不改变原集合/数组
- **toMap 陷阱**：key 重复会抛 `IllegalStateException`，需要 `toMap(k, v, merge函数)` 指定合并策略
- **parallelStream**：默认用 `ForkJoinPool.commonPool()`，非线程安全的数据结构慎用；不是所有场景并行都快，数据量小/有阻塞 IO 时不划算

# 2. 方法引用

方法引用是 Lambda 的**进一步简化**，把已存在的方法直接"引用"给函数式接口。

## 2.1 使用要求

1. 引用处是**函数式接口**（只有一个抽象方法）
2. 被引用方法**已存在**
3. 被引用方法的**形参和返回值**与抽象方法一致
4. 被引用方法的功能满足需求

## 2.2 四种形式

| 形式 | 写法 | 示例 |
|------|------|------|
| 引用静态方法 | `类名::静态方法` | `Math::abs` |
| 引用其他类实例方法 | `对象::方法` | `str::toUpperCase` |
| 引用本类实例方法 | `this::方法` | `this::handle` |
| 引用父类实例方法 | `super::方法` | `super::handle` |
| 引用构造方法 | `类名::new` | `User::new` |
| 引用数组构造 | `类型[]::new` | `String[]::new` |

**特定类型任意对象的实例方法**（如 `String::length`）：第一个参数是被引用方法的**接收者**，其余参数作为方法参数——它替代了形如 `s -> s.length()` 的 Lambda。

## 2.3 与 Lambda 的对比

```java
list.forEach(s -> System.out.println(s));  // Lambda
list.forEach(System.out::println);         // 方法引用，等价
```

★ **关键认知**：方法引用**能省则省，不能省别硬凑**——只有当 Lambda 体就是一个方法调用（`x -> f(x)`、`x -> x.f()`、`x -> new X(x)`）时才能转成方法引用；Lambda 里有额外逻辑时就用 Lambda。方法引用唯一优势是**更简洁、可读性更好**，语义上与等价 Lambda 完全一致。