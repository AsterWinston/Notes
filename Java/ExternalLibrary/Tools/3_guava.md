# 1. 概述与简介

Guava 是 Google 开源的 Java 核心库，包含 Google 内部多年积累的最佳实践。它提供了一系列高效、设计良好的工具类，涵盖集合、缓存、函数式编程、字符串处理、并发、I/O、哈希等众多领域。

## 1.1 核心设计理念

- **简约至上**：API 设计简洁明确，单一职责
- **防御性编程**：大量使用 Preconditions 进行参数校验
- **不可变性优先**：提供丰富的不可变集合，减少副作用
- **函数式风格**：支持 Predicate、Function 等函数式接口（虽然后续版本淡化）
- **高效实现**：内部实现经过高度优化

## 1.2 Maven 依赖

```xml
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>32.1.3-jre</version>
</dependency>
```

对于 Android 项目使用 `-android` 版本：

```xml
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>32.1.3-android</version>
</dependency>
```

## 1.3 模块概览

| 模块 | 核心类 | 功能 |
|------|--------|------|
| 基础工具 | Preconditions, MoreObjects, Throwables | 参数校验、null 处理、异常 |
| 集合 | ImmutableList, Multimap, BiMap, Table, RangeSet | 增强集合 |
| 缓存 | CacheBuilder, LoadingCache, Cache | 本地缓存 |
| 字符串 | Joiner, Splitter, CharMatcher | 字符串处理 |
| 并发 | ListenableFuture, Futures, Service | 异步编程 |
| I/O | Files, CharSource, ByteSource | 简化 I/O |
| 哈希 | Hashing, HashFunction, BloomFilter | 哈希与布隆过滤器 |
| 事件 | EventBus | 事件总线 |
| 数学 | IntMath, LongMath, BigIntegerMath | 数学计算 |
| 限流 | RateLimiter | 速率限制 |

---

# 2. 核心功能与 API

## 2.1 不可变集合（Immutable Collections）

不可变集合是 Guava 的核心设计理念之一，具有线程安全、不可修改、内存友好等优势。

```java
public class ImmutableCollectionExample {

    public static void main(String[] args) {
        // ========== 创建不可变集合 ==========
        // of 方法（适用于少量元素）
        ImmutableList<String> list = ImmutableList.of("a", "b", "c");
        ImmutableSet<Integer> set = ImmutableSet.of(1, 2, 3);
        ImmutableMap<String, Integer> map = ImmutableMap.of("a", 1, "b", 2);

        // builder 模式（适用于大量元素）
        ImmutableList<String> builtList = ImmutableList.<String>builder()
                .add("a")
                .add("b", "c")
                .addAll(Arrays.asList("d", "e"))
                .build();

        // copyOf（从可变集合创建）
        List<String> mutableList = new ArrayList<>();
        mutableList.add("x");
        ImmutableList<String> immutableList = ImmutableList.copyOf(mutableList);

        // ========== 不可变集合的特点 ==========
        // 1. 任何修改操作都会抛出 UnsupportedOperationException
        try {
            immutableList.add("y");  // 抛异常！
        } catch (UnsupportedOperationException e) {
            System.out.println("不可修改");
        }

        // 2. 包含所有标准集合方法（get、contains、size 等）
        boolean contains = immutableList.contains("a");  // true
        int size = immutableList.size();                 // 5

        // 3. 支持逆向排序
        ImmutableList<String> reversed = immutableList.reverse();

        // 4. 支持子列表
        ImmutableList<String> sub = immutableList.subList(0, 2);

        // ========== 不可变集合的优势 ==========
        // 1. 线程安全：无写操作，无需同步
        // 2. 可信任 API：传入参数不会被修改
        // 3. 内存效率：复用底层数组，节省内存
        // 4. 可作为常量使用
        // 5. 可作为 Map 的 key（因为 hashCode 不变）
    }
}
```

### 不可变集合 vs Collections.unmodifiableXXX

```java
// Collections.unmodifiableList 是"视图"模式
List<String> original = new ArrayList<>();
original.add("a");
List<String> unmod = Collections.unmodifiableList(original);

original.add("b");  // 通过原集合修改，unmod 也会变化！
System.out.println(unmod.size());  // 2

// Guava 的 ImmutableList 是真正的不可变
ImmutableList<String> immutable = ImmutableList.copyOf(original);
original.add("c");   // 不影响 immutable
System.out.println(immutable.size());  // 2（真正的快照）
```

## 2.2 新集合类型

### Multimap（多值 Map）

一个键对应多个值，替代 `Map<K, Collection<V>>` 的繁琐操作。

```java
public class MultimapExample {

    public static void main(String[] args) {
        // ========== ArrayListMultimap ==========
        // 键 -> List（有序、可重复）
        ArrayListMultimap<String, String> listMultimap = ArrayListMultimap.create();
        listMultimap.put("fruit", "apple");
        listMultimap.put("fruit", "banana");
        listMultimap.put("fruit", "apple");  // 可重复
        listMultimap.put("vegetable", "carrot");

        Collection<String> fruits = listMultimap.get("fruit");
        System.out.println(fruits);  // [apple, banana, apple]

        // ========== HashMultimap ==========
        // 键 -> Set（无序、不重复）
        HashMultimap<String, String> setMultimap = HashMultimap.create();
        setMultimap.put("fruit", "apple");
        setMultimap.put("fruit", "banana");
        setMultimap.put("fruit", "apple");  // 重复键，被忽略
        System.out.println(setMultimap.get("fruit"));  // [apple, banana]

        // ========== LinkedHashMultimap ==========
        // 键 -> Set（保持插入顺序）
        LinkedHashMultimap<String, String> linkedMultimap = LinkedHashMultimap.create();

        // ========== TreeMultimap ==========
        // 键 -> Set（按自然顺序或 Comparator 排序）
        TreeMultimap<String, String> treeMultimap = TreeMultimap.create();

        // ========== 常用操作 ==========
        ArrayListMultimap<Integer, String> multimap = ArrayListMultimap.create();
        multimap.putAll(1, Arrays.asList("a", "b", "c"));
        multimap.putAll(2, Arrays.asList("d", "e"));

        // 获取所有键
        Set<Integer> keys = multimap.keySet();

        // 获取所有键值对
        Collection<Map.Entry<Integer, String>> entries = multimap.entries();

        // 获取所有值
        Collection<String> values = multimap.values();

        // 转换为 Map<K, Collection<V>>
        Map<Integer, Collection<String>> mapView = multimap.asMap();

        // 获取指定键的值集合（不会创建空集合）
        Collection<String> valueList = multimap.get(3);  // 返回空集合而不是 null

        // 包含检查
        boolean containsEntry = multimap.containsEntry(1, "a");
        boolean containsKey = multimap.containsKey(1);
        boolean containsValue = multimap.containsValue("a");

        // 删除
        multimap.remove(1, "a");      // 删除指定键值对
        multimap.removeAll(1);        // 删除键下的所有值
        multimap.values().remove("b"); // 删除所有值为 "b" 的条目
    }
}
```

### BiMap（双向 Map）

键和值都是唯一的，可互相查找。

```java
public class BiMapExample {

    public static void main(String[] args) {
        // ========== HashBiMap ==========
        HashBiMap<String, Integer> biMap = HashBiMap.create();
        biMap.put("Alice", 1001);
        biMap.put("Bob", 1002);
        biMap.put("Charlie", 1003);

        // 正向查找：键 -> 值
        Integer id = biMap.get("Alice");     // 1001

        // 反向查找：值 -> 键
        BiMap<Integer, String> inverse = biMap.inverse();
        String name = inverse.get(1002);     // "Bob"

        // 重要：BiMap 的值必须唯一
        try {
            biMap.put("David", 1001);  // 抛 IllegalArgumentException!
        } catch (IllegalArgumentException e) {
            System.out.println("值 1001 已存在");
        }

        // 强制 put（会删除旧条目）
        biMap.forcePut("David", 1001);  // 删除 Alice -> 1001
    }
}
```

### Table（双键 Map）

使用两个键（行和列）定位一个值，相当于 `Map<R, Map<C, V>>` 的简化。

```java
public class TableExample {

    public static void main(String[] args) {
        // ========== HashBasedTable ==========
        // 行 -> 列 -> 值
        Table<String, String, Integer> table = HashBasedTable.create();
        table.put("张三", "语文", 90);
        table.put("张三", "数学", 85);
        table.put("李四", "语文", 88);
        table.put("李四", "数学", 92);

        // 获取单个值
        Integer score = table.get("张三", "数学");  // 85

        // 获取整行
        Map<String, Integer> zhangsanScores = table.row("张三");
        System.out.println(zhangsanScores);  // {语文=90, 数学=85}

        // 获取整列
        Map<String, Integer> mathScores = table.column("数学");
        System.out.println(mathScores);  // {张三=85, 李四=92}

        // 获取所有行
        Set<String> rows = table.rowKeySet();    // [张三, 李四]

        // 获取所有列
        Set<String> cols = table.columnKeySet(); // [语文, 数学]

        // 是否包含
        boolean hasEntry = table.contains("张三", "语文");
        boolean hasRow = table.containsRow("张三");
        boolean hasCol = table.containsColumn("数学");

        // ========== TreeBasedTable ==========
        // 行和列都排序
        TreeBasedTable<String, String, Integer> sortedTable = TreeBasedTable.create();

        // ========== ArrayTable ==========
        // 固定行列的 Table，性能更好
        ArrayTable<String, String, Integer> arrayTable = ArrayTable.create(table);

        // ========== ImmutableTable ==========
        // 不可变 Table
    }
}
```

### RangeSet（区间集合）

用于管理不重叠的区间集合。

```java
public class RangeSetExample {

    public static void main(String[] args) {
        // ========== TreeRangeSet ==========
        RangeSet<Integer> rangeSet = TreeRangeSet.create();

        rangeSet.add(Range.closed(1, 10));     // [1, 10]
        rangeSet.add(Range.closed(11, 20));    // [1, 20]（自动合并相邻区间）
        rangeSet.add(Range.closed(30, 40));    // [1, 20] ∪ [30, 40]

        // 包含检查
        boolean contains = rangeSet.contains(15);   // true
        boolean contains2 = rangeSet.contains(25);  // false

        // 区间补集
        RangeSet<Integer> complement = rangeSet.complement();

        // 包含当前值的区间
        Range<Integer> range = rangeSet.rangeContaining(15);  // [1, 20]

        // 遍历所有区间
        for (Range<Integer> r : rangeSet.asRanges()) {
            System.out.println(r);  // [1, 20]  [30, 40]
        }

        // ========== RangeMap ==========
        RangeMap<Integer, String> rangeMap = TreeRangeMap.create();
        rangeMap.put(Range.closed(1, 10), "低优先级");
        rangeMap.put(Range.closed(11, 20), "中优先级");
        rangeMap.put(Range.closed(21, 100), "高优先级");

        String priority = rangeMap.get(15);  // "中优先级"
    }
}
```

### ClassToInstanceMap

类型到实例的 Map，类型安全。

```java
ClassToInstanceMap<Object> classMap = MutableClassToInstanceMap.create();
classMap.putInstance(String.class, "hello");
classMap.putInstance(Integer.class, 123);

String str = classMap.getInstance(String.class);   // 无需强制转换
Integer i = classMap.getInstance(Integer.class);
```

## 2.3 字符串工具

### Joiner（连接器）

```java
public class JoinerExample {

    public static void main(String[] args) {
        // ========== 基本使用 ==========
        Joiner joiner = Joiner.on(", ").skipNulls();
        String result = joiner.join("a", null, "b", "c");
        System.out.println(result);  // "a, b, c"

        // 使用默认值替代 null
        Joiner joinerWithDefault = Joiner.on(", ").useForNull("(null)");
        String result2 = joinerWithDefault.join("a", null, "b");
        System.out.println(result2);  // "a, (null), b"

        // 连接数组
        String[] arr = {"x", "y", "z"};
        String joined = Joiner.on("-").join(arr);  // "x-y-z"

        // 连接 Iterable
        List<Integer> numbers = Arrays.asList(1, 2, 3);
        String numStr = Joiner.on(":").join(numbers);  // "1:2:3"

        // 连接 Map（键值对）
        Map<String, Integer> map = new HashMap<>();
        map.put("a", 1);
        map.put("b", 2);
        String mapStr = Joiner.on("&").withKeyValueSeparator("=").join(map);
        // "a=1&b=2"
    }
}
```

### Splitter（分割器）

```java
public class SplitterExample {

    public static void main(String[] args) {
        // ========== 基本使用 ==========
        // 按字符分割
        Iterable<String> parts = Splitter.on(',')
                .trimResults()
                .omitEmptyStrings()
                .split(" a ,, b ,c ");
        // ["a", "b", "c"]

        // 按正则分割
        Iterable<String> byRegex = Splitter.onPattern("\\d+")
                .split("a1b2c3d");
        // ["a", "b", "c", "d"]

        // 固定长度分割
        Iterable<String> fixed = Splitter.fixedLength(3)
                .split("abcdefghi");
        // ["abc", "def", "ghi"]

        // 限制分割次数
        Iterable<String> limited = Splitter.on(",")
                .limit(3)
                .split("a,b,c,d,e");
        // ["a", "b", "c,d,e"]

        // ========== 分割为 List ==========
        List<String> list = Splitter.on(",")
                .trimResults()
                .splitToList(" a , b , c ");
        // ["a", "b", "c"]

        // ========== 分割 Map ==========
        Map<String, String> map = Splitter.on("&")
                .withKeyValueSeparator("=")
                .split("name=张三&age=25&city=北京");
        // {name=张三, age=25, city=北京}
    }
}
```

### CharMatcher（字符匹配器）

```java
public class CharMatcherExample {

    public static void main(String[] args) {
        // ========== 预定义匹配器 ==========
        CharMatcher.digit();         // 匹配数字
        CharMatcher.javaLetter();    // 匹配 Java 字母
        CharMatcher.whitespace();    // 匹配空白
        CharMatcher.javaIsoControl(); // 匹配控制字符
        CharMatcher.none();          // 不匹配任何字符
        CharMatcher.any();           // 匹配所有字符

        // ========== 常用操作 ==========
        // 移除匹配的字符
        String noDigits = CharMatcher.digit().removeFrom("abc123def456");
        System.out.println(noDigits);  // "abcdef"

        // 保留匹配的字符
        String onlyDigits = CharMatcher.digit().retainFrom("abc123def456");
        System.out.println(onlyDigits);  // "123456"

        // 修剪（类似 trim，可定制）
        String trimmed = CharMatcher.whitespace().trimFrom("  abc  ");
        System.out.println(trimmed);  // "abc"

        // 替换匹配的字符
        String replaced = CharMatcher.digit().replaceFrom("abc123", "*");
        System.out.println(replaced);  // "abc***"

        // 折叠匹配的字符（替换为单一字符）
        String collapsed = CharMatcher.whitespace().collapseFrom("a   b   c", ' ');
        System.out.println(collapsed);  // "a b c"

        // 匹配全部或任一
        boolean allMatch = CharMatcher.digit().matchesAllOf("12345");   // true
        boolean anyMatch = CharMatcher.digit().matchesAnyOf("abc123");  // true
        boolean noMatch = CharMatcher.digit().matchesNoneOf("abc");     // true

        // ========== 自定义匹配器 ==========
        CharMatcher custom = CharMatcher.anyOf("aeiou");  // 匹配元音字母
        CharMatcher range = CharMatcher.inRange('a', 'z'); // 匹配 a-z

        // 组合匹配器
        CharMatcher combined = CharMatcher.digit()
                .or(CharMatcher.inRange('a', 'f'));  // 匹配数字或 a-f
    }
}
```

## 2.4 Preconditions（前置条件）

```java
public class PreconditionsExample {

    public void processUser(User user, int age, String name, List<String> tags) {
        // 检查非 null
        Preconditions.checkNotNull(user, "user 不能为 null");
        Preconditions.checkNotNull(name, "name 不能为 null");

        // 检查条件
        Preconditions.checkArgument(age > 0, "age 必须大于 0，当前值: %s", age);

        // 检查状态
        Preconditions.checkState(tags != null && !tags.isEmpty(), "tags 不能为空");

        // 检查元素索引
        Preconditions.checkElementIndex(5, tags.size(), "index 越界");

        // 检查位置索引
        Preconditions.checkPositionIndex(3, tags.size());

        // 检查位置索引范围
        Preconditions.checkPositionIndexes(1, 4, tags.size());

        // 验证通过后执行业务逻辑
    }

    // 对比传统写法
    public void traditionalWay(User user) {
        if (user == null) {
            throw new IllegalArgumentException("user 不能为 null");
        }
        if (user.getName() == null || user.getName().isEmpty()) {
            throw new IllegalArgumentException("name 不能为空");
        }
    }

    // Guava 方式
    public void guavaWay(User user) {
        Preconditions.checkNotNull(user);
        Preconditions.checkArgument(user.getName() != null && !user.getName().isEmpty());
    }
}
```

## 2.5 Optional（Guava 版本）

注意：Guava 的 Optional 在 Java 8+ 中已被 `java.util.Optional` 替代，但在某些场景仍有优势。

```java
public class OptionalExample {

    public static void main(String[] args) {
        // ========== 创建 ==========
        Optional<Integer> present = Optional.of(5);        // 值不能为 null
        Optional<Integer> absent = Optional.absent();       // 空 Optional
        Optional<Integer> nullable = Optional.fromNullable(null);  // 允许 null

        // ========== 判断 ==========
        boolean isPresent = present.isPresent();   // true
        boolean isAbsent = absent.isPresent();     // false

        // ========== 获取值 ==========
        int value1 = present.get();                // 5
        int value2 = present.or(0);                // 5
        int value3 = absent.or(0);                 // 0
        int value4 = absent.orNull();              // null
        int value5 = absent.or(new Supplier<Integer>() {
            @Override
            public Integer get() {
                return computeDefault();
            }
        });

        // ========== 转换 ==========
        Optional<String> transformed = present.transform(input -> "Number: " + input);
        System.out.println(transformed.get());  // "Number: 5"

        // ========== 集合中的 Optional ==========
        Iterable<Integer> values = Optional.presentInstances(
            Arrays.asList(
                Optional.of(1),
                Optional.absent(),
                Optional.of(2)
            )
        );
        // 只包含 [1, 2]
    }

    private static int computeDefault() {
        return -1;
    }
}
```

## 2.6 排序与比较

### ComparisonChain

```java
public class Person implements Comparable<Person> {
    private String lastName;
    private String firstName;
    private int age;

    @Override
    public int compareTo(Person other) {
        // 链式比较，优雅处理 null
        return ComparisonChain.start()
                .compare(this.lastName, other.lastName, Ordering.natural().nullsLast())
                .compare(this.firstName, other.firstName, Ordering.natural().nullsLast())
                .compare(this.age, other.age)
                .result();
    }
}
```

### Ordering（Guava 排序器）

```java
public class OrderingExample {

    public static void main(String[] args) {
        List<String> list = Arrays.asList("banana", "apple", "cherry", "date");

        // 自然排序
        Ordering<String> natural = Ordering.natural();
        List<String> sorted = natural.sortedCopy(list);  // [apple, banana, cherry, date]

        // 反转排序
        List<String> reversed = natural.reverse().sortedCopy(list);

        // null 处理
        List<String> withNull = Arrays.asList("b", null, "a", null);
        List<String> nullsFirst = natural.nullsFirst().sortedCopy(withNull);
        List<String> nullsLast = natural.nullsLast().sortedCopy(withNull);

        // 自定义比较器
        Ordering<String> byLength = Ordering.natural()
                .onResultOf(String::length)
                .compound(natural);  // 先按长度，同长度按字典序

        // 取最大/最小
        String max = natural.max("a", "b", "c");       // "c"
        String min = natural.min("a", "b", "c");       // "a"
        List<String> top2 = natural.greatestOf(list, 2); // ["cherry", "date"]

        // 判断是否有序
        boolean isOrdered = natural.isOrdered(list);     // false
        boolean isStrictlyOrdered = natural.isStrictlyOrdered(list);

        // 二分查找
        int index = natural.binarySearch(sorted, "cherry");
    }
}
```

---

# 3. 高级特性

## 3.1 缓存（CacheBuilder）

Guava 的缓存是功能最强大的本地缓存实现之一。

```java
public class CacheExample {

    public static void main(String[] args) throws Exception {
        // ========== LoadingCache（自动加载） ==========
        LoadingCache<String, User> userCache = CacheBuilder.newBuilder()
                // 缓存配置
                .maximumSize(1000)                        // 最大条目数
                .expireAfterWrite(10, TimeUnit.MINUTES)   // 写入后过期
                .expireAfterAccess(5, TimeUnit.MINUTES)   // 访问后过期
                .refreshAfterWrite(1, TimeUnit.MINUTES)   // 写入后自动刷新
                .recordStats()                            // 开启统计
                .removalListener(new RemovalListener<String, User>() {
                    @Override
                    public void onRemoval(RemovalNotification<String, User> notification) {
                        System.out.println("缓存移除: " + notification.getKey() +
                                ", 原因: " + notification.getCause());
                    }
                })
                // 构建 LoadingCache
                .build(new CacheLoader<String, User>() {
                    @Override
                    public User load(String key) throws Exception {
                        System.out.println("从数据库加载: " + key);
                        return findUserFromDatabase(key);
                    }

                    // 批量加载（可选优化）
                    @Override
                    public Map<String, User> loadAll(Iterable<? extends String> keys)
                            throws Exception {
                        return findAllFromDatabase(keys);
                    }
                });

        // ========== 使用缓存 ==========
        // 1. 从缓存获取（如果不存在则调用 load）
        User user1 = userCache.get("user_123");

        // 2. 获取时指定默认值
        User user2 = userCache.get("user_456", () -> new User("默认用户"));

        // 3. 批量获取
        ImmutableMap<String, User> batch = userCache.getAll(
            Arrays.asList("user_111", "user_222"));

        // 4. 手动写入/刷新
        userCache.put("user_333", new User("手动添加"));
        userCache.refresh("user_111");  // 异步刷新

        // ========== 缓存统计 ==========
        CacheStats stats = userCache.stats();
        double hitRate = stats.hitRate();            // 命中率
        double loadExceptionRate = stats.loadExceptionRate();  // 加载异常率
        long averageLoadPenalty = stats.averageLoadPenalty();  // 平均加载耗时(ns)

        // ========== 清除缓存 ==========
        userCache.invalidate("user_123");        // 清除单个
        userCache.invalidateAll(keys);           // 批量清除
        userCache.invalidateAll();               // 清除全部

        // ========== Cache（手动控制的缓存） ==========
        Cache<String, User> manualCache = CacheBuilder.newBuilder()
                .maximumSize(500)
                .expireAfterWrite(30, TimeUnit.MINUTES)
                .build();

        // 手动操作
        manualCache.put("key", new User("value"));
        User value = manualCache.getIfPresent("key");
        manualCache.invalidate("key");

        // ========== 权重缓存（按权重而非数量限制） ==========
        Cache<String, String> weightCache = CacheBuilder.newBuilder()
                .maximumWeight(10000)
                .weigher((String key, String value) -> value.length())
                .build();
    }

    private static User findUserFromDatabase(String key) {
        // 模拟数据库查询
        return new User("User_" + key);
    }

    private static Map<String, User> findAllFromDatabase(Iterable<? extends String> keys) {
        Map<String, User> result = new HashMap<>();
        for (String key : keys) {
            result.put(key, new User("User_" + key));
        }
        return result;
    }

    static class User {
        private String name;
        User(String name) { this.name = name; }
        public String getName() { return name; }
    }
}
```

### 缓存配置详解

| 配置项 | 说明 | 适用场景 |
|--------|------|----------|
| `maximumSize` | 最大条目数 | 控制内存占用 |
| `maximumWeight` | 最大权重 | 按自定义大小控制 |
| `expireAfterWrite` | 写入后过期 | 数据变化不频繁 |
| `expireAfterAccess` | 访问后过期 | 热点数据缓存 |
| `refreshAfterWrite` | 写入后自动刷新 | 数据可延迟过期 |
| `weakKeys` | 弱引用键 | 允许 GC 回收键 |
| `weakValues` | 弱引用值 | 允许 GC 回收值 |
| `softValues` | 软引用值 | JVM 内存不足时回收 |
| `recordStats` | 开启统计 | 监控缓存效率 |
| `concurrencyLevel` | 并发级别 | 高并发访问优化 |
| `initialCapacity` | 初始容量 | 减少扩容开销 |

## 3.2 EventBus（事件总线）

Guava 的 EventBus 提供了解耦的发布-订阅模式实现。

```java
// ========== 定义事件 ==========
class OrderCreatedEvent {
    private final Long orderId;
    private final String customerName;

    public OrderCreatedEvent(Long orderId, String customerName) {
        this.orderId = orderId;
        this.customerName = customerName;
    }

    public Long getOrderId() { return orderId; }
    public String getCustomerName() { return customerName; }
}

class OrderPaidEvent {
    private final Long orderId;
    private final BigDecimal amount;

    public OrderPaidEvent(Long orderId, BigDecimal amount) {
        this.orderId = orderId;
        this.amount = amount;
    }

    public Long getOrderId() { return orderId; }
    public BigDecimal getAmount() { return amount; }
}

// ========== 事件监听器 ==========
class EmailNotificationListener {

    @Subscribe
    public void handleOrderCreated(OrderCreatedEvent event) {
        System.out.println("发送邮件: 订单 " + event.getOrderId()
                + " 已创建，客户: " + event.getCustomerName());
    }

    @Subscribe
    public void handleOrderPaid(OrderPaidEvent event) {
        System.out.println("发送邮件: 订单 " + event.getOrderId()
                + " 已支付，金额: " + event.getAmount());
    }
}

class SmsNotificationListener {

    @Subscribe
    public void handleOrderCreated(OrderCreatedEvent event) {
        System.out.println("发送短信: 订单 " + event.getOrderId() + " 已创建");
    }
}

class LoggingListener {

    @Subscribe
    public void logAllEvents(Object event) {
        System.out.println("日志: " + event.getClass().getSimpleName()
                + " = " + event);
    }
}

// ========== 使用 EventBus ==========
public class EventBusExample {

    public static void main(String[] args) {
        // 创建 EventBus（同步）
        EventBus eventBus = new EventBus("orderBus");

        // 创建 AsyncEventBus（异步）
        ExecutorService executor = Executors.newFixedThreadPool(4);
        AsyncEventBus asyncEventBus = new AsyncEventBus("asyncOrderBus", executor);

        // 注册监听器
        eventBus.register(new EmailNotificationListener());
        eventBus.register(new SmsNotificationListener());
        eventBus.register(new LoggingListener());

        // 发布事件
        eventBus.post(new OrderCreatedEvent(1001L, "张三"));
        eventBus.post(new OrderPaidEvent(1001L, new BigDecimal("99.99")));

        // 注销监听器
        // eventBus.unregister(listener);

        // 注意：
        // 1. 默认是同步执行（订阅者按注册顺序执行）
        // 2. 异常不会影响其他订阅者（DeadEvent 机制）
        // 3. 支持事件继承（父类事件会被子类订阅者处理）
        // 4. 不支持事件优先级

        // DeadEvent：没有订阅者的事件
        class DeadEventListener {
            @Subscribe
            public void handleDeadEvent(DeadEvent deadEvent) {
                System.out.println("未处理事件: " + deadEvent.getEvent());
            }
        }
        eventBus.register(new DeadEventListener());
        eventBus.post("someStringEvent");  // 没有订阅者匹配，转为 DeadEvent
    }
}
```

## 3.3 RateLimiter（限流器）

用于控制访问速率，防止系统过载。

```java
public class RateLimiterExample {

    public static void main(String[] args) throws Exception {
        // ========== 基本使用 ==========
        // 每秒允许 5 个请求
        RateLimiter limiter = RateLimiter.create(5.0);

        for (int i = 0; i < 20; i++) {
            // 获取许可（阻塞直到获取到）
            double waitTime = limiter.acquire();
            System.out.println("请求 " + i + " 等待了 " + waitTime + " 秒");
            processRequest();
        }

        // ========== tryAcquire（非阻塞） ==========
        RateLimiter nonBlocking = RateLimiter.create(10.0);

        for (int i = 0; i < 100; i++) {
            // 尝试获取，不阻塞
            if (nonBlocking.tryAcquire()) {
                // 获取成功，执行请求
                processRequest();
            } else {
                // 获取失败，拒绝或排队
                System.out.println("请求被限流");
            }
        }

        // ========== 带超时的 tryAcquire ==========
        if (nonBlocking.tryAcquire(100, TimeUnit.MILLISECONDS)) {
            processRequest();
        }

        // ========== 突发流量处理 ==========
        // 创建支持突发流量的限流器
        RateLimiter burstLimiter = RateLimiter.create(5.0);
        // 预热期（平滑突发）
        RateLimiter warmupLimiter = RateLimiter.create(5.0, 2, TimeUnit.SECONDS);

        // ========== 实际应用：API 限流 ==========
    }

    // API 限流示例
    static class ApiRateLimiter {
        // 按用户限流
        private final LoadingCache<String, RateLimiter> userLimiters = CacheBuilder.newBuilder()
                .maximumSize(10000)
                .expireAfterWrite(1, TimeUnit.HOURS)
                .build(new CacheLoader<String, RateLimiter>() {
                    @Override
                    public RateLimiter load(String userId) {
                        return RateLimiter.create(100.0);  // 每用户每秒 100 次
                    }
                });

        public boolean tryAcquire(String userId) {
            return userLimiters.getUnchecked(userId).tryAcquire();
        }
    }

    private static void processRequest() {
        // 处理请求
    }
}
```

### RateLimiter 原理

RateLimiter 基于令牌桶算法（Token Bucket）实现：
- 以固定速率向桶中添加令牌
- 每个请求消耗一个令牌
- 桶有最大容量（突发能力）
- 请求到达时如果有令牌则立即通过
- 无令牌时等待或拒绝

Guava 提供了两种实现：
- **SmoothBursty**：平滑突发，允许一定程度的突发流量
- **SmoothWarmingUp**：平滑预热，启动时速率逐渐升高到目标值

## 3.4 BloomFilter（布隆过滤器）

用于判断一个元素是否在集合中，有误判率但不会漏判。

```java
public class BloomFilterExample {

    public static void main(String[] args) {
        // ========== 创建布隆过滤器 ==========
        // 预计插入 1000 个元素，误判率 1%
        BloomFilter<String> bloomFilter = BloomFilter.create(
                Funnels.stringFunnel(Charset.forName("UTF-8")),
                1000,
                0.01);

        // ========== 插入元素 ==========
        bloomFilter.put("张三");
        bloomFilter.put("李四");
        bloomFilter.put("王五");

        // ========== 判断是否存在 ==========
        boolean mightContain = bloomFilter.mightContain("张三");   // true（极大概率）
        boolean mightContain2 = bloomFilter.mightContain("赵六");  // false（确定不存在）
        boolean mightContain3 = bloomFilter.mightContain("王五");  // true

        // ========== 实际应用场景 ==========
        // 1. 缓存穿透防护
        boolean userExists = bloomFilter.mightContain("user_9999");
        if (!userExists) {
            // 直接返回空，无需查询数据库
            return;
        }
        // 存在则查询（仍有误判可能，但不会漏掉）

        // 2. 防止重复爬取
        BloomFilter<String> crawledUrls = BloomFilter.create(
                Funnels.stringFunnel(Charset.forName("UTF-8")),
                1_000_000,
                0.001);

        // 3. 推荐系统去重
        // 4. 垃圾邮件过滤
    }
}
```

## 3.5 Hashing（哈希）

Guava 提供了多种哈希算法的高效实现。

```java
public class HashingExample {

    public static void main(String[] args) {
        // ========== 常用哈希函数 ==========
        HashFunction md5 = Hashing.md5();
        HashFunction sha256 = Hashing.sha256();
        HashFunction sha512 = Hashing.sha512();
        HashFunction crc32 = Hashing.crc32();
        HashFunction murmur3_32 = Hashing.murmur3_32();   // 高性能非加密哈希
        HashFunction murmur3_128 = Hashing.murmur3_128();
        HashFunction sipHash24 = Hashing.sipHash24();     // 防哈希碰撞攻击

        // ========== 哈希计算 ==========
        String input = "hello world";

        // 方式 1：hashString
        HashCode hashCode = sha256.hashString(input, Charset.forName("UTF-8"));
        String hex = hashCode.toString();         // 十六进制字符串
        long asLong = hashCode.asLong();          // long 值
        int asInt = hashCode.asInt();             // int 值
        byte[] bytes = hashCode.asBytes();        // 字节数组

        // 方式 2：hashBytes
        HashCode fromBytes = sha256.hashBytes(input.getBytes());

        // ========== 一致性哈希 ==========
        // 用于分布式缓存/负载均衡
        ConsistentHash<String, String> consistentHash = Hashing
                .consistentHash(Hashing.murmur3_128(), 10);  // 10 个节点
        // 注意：API 已变更，新版使用：
        // int bucket = Hashing.consistentHash(hashCode, 10);

        // ========== 组合哈希 ==========
        HashCode combined = Hashing.combineOrdered(
            Arrays.asList(md5.hashString("a", Charset.forName("UTF-8")),
                         md5.hashString("b", Charset.forName("UTF-8")))
        );

        // ========== 布隆过滤器的哈希分解 ==========
        int numHashFunctions = Hashing.consistentHash(hashCode, 10);
    }
}
```

## 3.6 Throwables（异常处理）

```java
public class ThrowablesExample {

    public static void main(String[] args) {
        try {
            dangerousOperation();
        } catch (Exception e) {
            // ========== 传播异常 ==========
            // 将检查型异常转为非检查型异常抛出
            Throwables.throwIfUnchecked(e);

            // 只传播指定类型的异常
            try {
                Throwables.propagateIfPossible(e, IOException.class);
            } catch (IOException ioEx) {
                System.out.println("处理 IO 异常");
            }

            // ========== 获取异常链 ==========
            List<Throwable> causalChain = Throwables.getCausalChain(e);
            Throwable rootCause = Throwables.getRootCause(e);

            // ========== 获取堆栈信息 ==========
            String stackTraceStr = Throwables.getStackTraceAsString(e);
        }
    }

    private static void dangerousOperation() throws IOException {
        throw new IOException("连接失败",
                new RuntimeException("底层错误",
                        new IllegalArgumentException("参数错误")));
    }
}
```

---

# 4. 底层原理

## 4.1 不可变集合的实现原理

Guava 的不可变集合在创建时一次性分配好底层数组，所有"修改"操作直接抛 `UnsupportedOperationException`。

```java
// 简化原理
public abstract class ImmutableCollection<E> extends AbstractCollection<E> {
    // 不实现 add、remove 等方法
    // 内部数组一旦初始化就不允许修改

    // 使用 Array 或类似结构直接存储
    private final E[] elements;

    // 读操作（get、size 等）直接委托给数组
    // 不提供写操作，或者写操作抛异常
}
```

## 4.2 Cache 的实现原理

Guava 的 `LocalCache` 是分段锁的实现（类似 ConcurrentHashMap）：

- 将缓存分为多个 Segment（默认 4 个，最大 16 个）
- 每个 Segment 有自己的锁和队列
- 支持引用类型（强引用、软引用、弱引用）
- 通过链式队列维护访问/写入顺序（用于过期策略）
- 自动清理通过 `cleanUp()` 线程触发（惰性删除）

```java
// 简化结构
public class LocalCache<K, V> {
    final Segment<K, V>[] segments;  // 分段数组

    static class Segment<K, V> {
        volatile AtomicReferenceArray<ReferenceEntry<K, V>> table;  // 哈希表
        final Queue<ReferenceEntry<K, V>> writeQueue;   // 写入顺序队列
        final Queue<ReferenceEntry<K, V>> accessQueue;  // 访问顺序队列

        // 过期检查：发生在 get/put 时
        // 惰性删除：不主动扫描，而是在操作时顺带清理
    }
}
```

## 4.3 EventBus 的实现原理

```java
// EventBus 核心使用注解 + 反射
public class EventBus {
    // 使用 Multimap 缓存事件类型 -> 订阅者方法的映射
    private final ConcurrentMap<Class<?>, CopyOnWriteArraySet<Subscriber>> subscribers;

    // 通过反射查找 @Subscribe 注解的方法
    // 在 register 时扫描并缓存
    // 在 post 时根据事件类型查找对应的订阅者并调用
}
```

## 4.4 RateLimiter 的实现原理

RateLimiter 基于令牌桶算法：

```java
// 简化实现
abstract class RateLimiter {
    // storedPermits: 当前存储的令牌数
    // maxPermits: 最大令牌数（桶容量）
    // stableIntervalMicros: 两个令牌之间的间隔（纳秒）

    double reserveEarliestAvailable(int requiredPermits, long nowMicros) {
        // 1. 重新计算当前令牌数（根据时间流逝补充）
        resync(nowMicros);

        // 2. 返回值：需要等待的时间
        long waitTime = 0;
        if (storedPermits < requiredPermits) {
            waitTime = (long) ((requiredPermits - storedPermits) * stableIntervalMicros);
        }

        // 3. 消耗令牌
        storedPermits -= requiredPermits;
        return waitTime;
    }
}
```

---

# 5. 性能优化

## 5.1 集合选择

```java
// 根据场景选择合适的集合类型

// 小而固定的数据 -> Immutable 集合
ImmutableList.of("a", "b", "c");  // 复用底层数组

// 需要快速查找的 Map -> ImmutableMap（单次构建）
// 需要在迭代时删除的 Map -> ConcurrentHashMap（非 Guava）

// 多值 Map -> Multimap（避免 Map<K, List<V>> 的样板代码）
Multimap<String, String> multimap = ArrayListMultimap.create();

// 双重索引 -> Table（比 Map<R, Map<C, V>> 更高效）
Table<String, String, Integer> table = HashBasedTable.create();

// 区间查询频繁 -> RangeSet
```

## 5.2 缓存优化

```java
// 1. 合理设置最大容量
CacheBuilder.newBuilder()
    .maximumSize(1000)  // 根据内存和对象大小设置
    .build();

// 2. 使用权重代替数量
CacheBuilder.newBuilder()
    .maximumWeight(1024 * 1024)  // 1MB
    .weigher((k, v) -> v.toString().length())
    .build();

// 3. 合理设置并发级别
CacheBuilder.newBuilder()
    .concurrencyLevel(Runtime.getRuntime().availableProcessors())
    .build();

// 4. 使用 refreshAfterWrite 代替 expireAfterWrite
//    减少热点数据同时过期的"缓存雪崩"
LoadingCache<String, Data> cache = CacheBuilder.newBuilder()
    .refreshAfterWrite(5, TimeUnit.MINUTES)  // 过期前刷新
    .expireAfterWrite(10, TimeUnit.MINUTES)  // 最终过期时间
    .build(new CacheLoader<String, Data>() {
        @Override
        public Data load(String key) {
            return loadData(key);
        }
    });
```

## 5.3 字符串处理优化

```java
// 1. 缓存 Joiner/Splitter/CharMatcher 实例（不可变，线程安全）
public static final Joiner COMMA_JOINER = Joiner.on(",").skipNulls();
public static final Splitter COMMA_SPLITTER = Splitter.on(",").trimResults();

// 2. 使用 CharMatcher 替代正则表达式
// 慢：str.replaceAll("\\d", "")
// 快：CharMatcher.digit().removeFrom(str)

// 3. 使用 Preconditions 替代 if/throw
// 编译时优化：非热点路径的方法调用可被 JIT 内联
```

## 5.4 BloomFilter 内存计算

```java
// BloomFilter 内存计算：
// n = 预计元素数, p = 误判率
// m = -n * ln(p) / (ln(2))^2  （位数）
// k = m/n * ln(2)            （哈希函数个数）
// 例如：n=1000, p=0.01 => m≈9585 bit≈1.2KB, k≈7

BloomFilter<String> filter = BloomFilter.create(
    Funnels.stringFunnel(Charset.forName("UTF-8")),
    1_000_000,     // 100万个元素
    0.001          // 0.1% 误判率
);
```

---

# 6. 最佳实践与常见坑

## 6.1 最佳实践

### 1. 优先使用不可变集合

```java
// 推荐：方法返回不可变集合
public ImmutableList<String> getNames() {
    return ImmutableList.copyOf(internalList);
}

// 推荐：常量定义
public static final ImmutableMap<String, String> ERROR_CODES = ImmutableMap.of(
    "400", "Bad Request",
    "401", "Unauthorized",
    "404", "Not Found"
);
```

### 2. 缓存配置模板

```java
LoadingCache<String, Object> cache = CacheBuilder.newBuilder()
    .maximumSize(1000)
    .expireAfterWrite(1, TimeUnit.HOURS)
    .recordStats()
    .build(new CacheLoader<String, Object>() {
        @Override
        public Object load(String key) {
            return loadFromDatabase(key);
        }
    });
```

### 3. Preconditions 统一参数校验

```java
public void createUser(String name, Integer age, String email) {
    Preconditions.checkNotNull(name, "name");
    Preconditions.checkArgument(name.length() >= 2, "name 至少2个字符", name);
    Preconditions.checkNotNull(age, "age");
    Preconditions.checkArgument(age >= 0 && age <= 150, "age 不合法", age);
    Preconditions.checkNotNull(email, "email");
    // ...
}
```

### 4. Splitter/Joiner 作为常量

```java
public class CsvConstants {
    public static final Joiner CSV_JOINER = Joiner.on(",").skipNulls();
    public static final Splitter CSV_SPLITTER = Splitter.on(",")
            .trimResults().omitEmptyStrings();
}
```

### 5. EventBus 使用规范

```java
// 为不同领域创建独立 EventBus
EventBus orderEventBus = new EventBus("order");
EventBus paymentEventBus = new EventBus("payment");

// 避免在 @Subscribe 方法中执行耗时操作
@Subscribe
public void handleEvent(OrderEvent event) {
    // 快速处理或提交到线程池
    executor.submit(() -> process(event));
}
```

## 6.2 常见坑

### 坑 1：ImmutableList 包含 null

```java
// ImmutableList 不允许 null 元素！
try {
    ImmutableList.of("a", null, "b");  // NullPointerException
} catch (NullPointerException e) {
    // 抛出异常
}

// 如果需要 null，使用 ArrayList
List<String> list = new ArrayList<>();
list.add("a");
list.add(null);
```

### 坑 2：Multimap.get() 返回的是活视图

```java
ArrayListMultimap<String, String> multimap = ArrayListMultimap.create();
multimap.put("key", "a");

Collection<String> values = multimap.get("key");
values.add("b");  // 会影响 multimap！

// multimap 现在包含 {key=[a, b]}
```

### 坑 3：Cache 的 value 不能为 null

```java
LoadingCache<String, String> cache = CacheBuilder.newBuilder()
    .build(new CacheLoader<String, String>() {
        @Override
        public String load(String key) {
            return null;  // InvalidCacheLoadException!
        }
    });

try {
    cache.get("key");
} catch (InvalidCacheLoadException e) {
    // CacheLoader 返回 null 时抛出此异常
}

// 解决方案：使用 Optional 包装
Cache<String, Optional<String>> cache2 = CacheBuilder.newBuilder()
    .build(new CacheLoader<String, Optional<String>>() {
        @Override
        public Optional<String> load(String key) {
            return Optional.fromNullable(loadFromDatabase(key));
        }
    });
```

### 坑 4：EventBus 异常处理

```java
// EventBus 默认不会抛出异常！
// 如果订阅者抛出异常，默认只打印日志
eventBus.register(new Object() {
    @Subscribe
    public void handler(String event) {
        throw new RuntimeException("出错");  // 被 EventBus 吞掉
    }
});

// 解决方案：使用 SubscriberExceptionHandler
EventBus safeBus = new EventBus((exception, context) -> {
    System.err.println("订阅者执行异常: " + exception.getMessage()
        + ", 事件: " + context.getEvent()
        + ", 订阅者: " + context.getSubscriber());
});
```

### 坑 5：RateLimiter 不支持分布式

```java
// RateLimiter 是单机限流
// 分布式环境需使用 Redis + Lua 或 Sentinel 等方案
```

### 坑 6：ComparisonChain 不能比较 long/double 正确性

```java
// ComparisonChain 的 compare 方法使用的是 Long.compare / Double.compare
// 对于 float/double 的 NaN 处理与常规不同
// 建议使用 Double.compare / Float.compare 的显式比较
```

### 坑 7：Guava 版本冲突

```java
// Guava 不同大版本 API 不兼容！
// 如 Ordering 在 32.x 中标记为 @Beta 或已移除
// 建议：统一项目中的 Guava 版本，避免多个版本冲突
```

---

# 7. 与其他技术对比

## 7.1 Guava vs Apache Commons

| 对比项 | Guava | Apache Commons |
|--------|-------|---------------|
| 集合增强 | 极强（Multimap, BiMap, Table 等） | 较弱 |
| 不可变集合 | 完善 | 无（用 Collections.unmodifiable） |
| 缓存 | 强大（CacheBuilder） | 无 |
| 字符串工具 | Joiner/Splitter/CharMatcher | StringUtils |
| 限流 | RateLimiter | 无 |
| 事件总线 | EventBus | 无 |
| 布隆过滤器 | BloomFilter | 无 |
| 数学计算 | IntMath/LongMath | 无 |
| 反射 | 有限（MoreObjects） | 丰富（BeanUtils, ReflectUtils） |
| 依赖 | 仅 Guava 自身 | 模块间依赖 |

## 7.2 Guava vs Hutool

| 对比项 | Guava | Hutool |
|--------|-------|--------|
| JSON | 无 | 内置 JSONUtil |
| HTTP | 无 | 内置 HttpUtil |
| 加密 | Hashing（仅哈希） | SecureUtil（全面） |
| 文件 IO | 有限 | 极其丰富 |
| 集合增强 | 极强 | CollUtil/MapUtil |
| 函数式编程 | 强 | 弱 |
| 不可变集合 | 核心特性 | 无 |
| 缓存 | 极强 | 简单 |
| 国际化 | 好 | 偏中文 |

## 7.3 选型建议

- **大型企业级项目**：Guava（集合、缓存、并发） + Commons（其他工具）
- **快速开发项目**：Hutool
- **需要强不可变集合**：Guava
- **需要本地缓存**：Guava Cache（比 Caffeine 稍弱，但功能完善）
- **Android 项目**：Android 官方逐渐迁移到 Kotlin 标准库
- **混合使用**：Guava 和 Hutool 可以兼容使用，但注意功能重合部分不要混用

---

# 8. 面试要点

## 8.1 基础问题

**Q1: ImmutableList 和 Collections.unmodifiableList 有什么区别？**

A: `Collections.unmodifiableList` 是视图包装，原始集合变化会影响它；`ImmutableList` 是真正的不可变集合，创建后内容完全不可变。此外，`ImmutableList` 有更低的内存占用和更高的性能（JIT 优化友好）。

**Q2: Guava 的 Multimap 有什么作用和实现类型？**

A: Multimap 是一个键对应多个值的 Map，避免了 `Map<K, Collection<V>>` 的繁琐操作。实现类型包括 `ArrayListMultimap`（值有序可重复）、`HashMultimap`（值无序不重复）、`LinkedHashMultimap`（值保持插入顺序）、`TreeMultimap`（值排序）。

**Q3: Guava Cache 的过期策略有哪些？**

A: `expireAfterWrite`（写入后过期）、`expireAfterAccess`（最后访问后过期）、`refreshAfterWrite`（写入后自动刷新，不阻塞其他请求）。

**Q4: RateLimiter 的令牌桶算法如何工作？**

A: 以固定速率向桶中添加令牌，桶有最大容量。请求到来时消耗一个令牌，有令牌则通过，无令牌则阻塞等待或直接拒绝。

**Q5: ComparisonChain 的作用？**

A: 简化 `Comparable.compareTo()` 方法的实现，按顺序比较多个字段，遇到第一个非 0 结果时提前返回。

## 8.2 进阶问题

**Q6: Guava Cache 如何处理并发加载？**

A: 通过 `ConcurrentHashMap` 分段锁实现。同一键的并发加载只有一个线程执行 `load()` 方法，其他线程等待结果。通过 `Future` 机制实现。

**Q7: EventBus 的 DeadEvent 机制是什么？**

A: 当发布的事件没有订阅者时，EventBus 会将事件包装为 `DeadEvent` 重新发布。通过注册 `DeadEvent` 的订阅者可以监控未处理事件。

**Q8: Guava 的 `Futures` 和 `ListenableFuture` 与 Java 的 `CompletableFuture` 关系？**

A: `ListenableFuture` 是 Guava 对 Future 的增强（添加回调注册），Java 8 的 `CompletableFuture` 是更完整的实现。`Futures` 工具类提供了转换、组合等操作。建议新代码直接使用 `CompletableFuture`。

**Q9: BloomFilter 的原理和误判率？**

A: 使用位数组和多个哈希函数。插入时对元素做 k 次哈希，将对应位置 1；查询时检查 k 个位置是否全为 1。可能误判（不在集合中判为在），但不会漏判。误判率由位数组大小和哈希函数数量决定。

**Q10: Ordering 在 Guava 32+ 版本中的状态？**

A: `Ordering` 已经在较新版本中标记为 `@Beta` 或计划移除。推荐使用 Java 8 的 `Comparator` 接口（尤其是 `Comparator.comparing()`、`thenComparing()` 等）。
