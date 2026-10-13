集合是 Java 用来**存储和操作一组对象**的容器，分为 `Collection`（单列集合）与 `Map`（双列集合）两大体系。本篇从集合框架结构讲起，完整覆盖 `List` / `Set` / `Queue` / `Map` 四类常用实现、泛型、包装类与 `Collections` 工具类，最后落到不可变集合与迭代器的并发修改问题。每个实现类都讲清"**怎么用、怎么选、底层为什么**"；`Stream` 与方法引用属函数式进阶，见第 13 篇。

# 1. 集合框架概述

## 1.1 数组与集合的对比

| 维度 | 数组 (Array) | 集合 (Collection) |
|------|-------------|------------------|
| 长度 | 固定，创建后不可变 | 动态可变，自动扩容 |
| 存储类型 | 基本类型 + 引用类型 | 仅引用类型（JDK5+ 自动装箱） |
| 泛型支持 | 不支持 | 支持 |
| 存放不同类型 | 可以（`Object[]`） | 不推荐（`Collection<Object>` 可以） |
| 专有属性/方法 | 仅 `.length` 属性 | 丰富的增删改查方法 |

★ **关键认知**：集合解决的是数组"**长度不可变**"和"**只能存固定类型**"两大痛点。⚠ 但集合只能存对象——存基本类型时发生自动装箱，取出时发生拆箱，这也埋下了 `null` 拆箱 NPE 的隐患（见第 8 篇包装类）。

## 1.2 集合体系结构

```text
Iterable（可迭代的顶层接口）
└── Collection（单列集合的顶层接口）
    ├── List         有序、可重复、有索引
    │   ├── ArrayList          数组实现，查询快增删慢，线程不安全
    │   ├── LinkedList        双向链表，增删快查询慢，也是 Deque
    │   ├── Vector            数组实现，线程安全，已过时
    │   └── CopyOnWriteArrayList   写时复制，读多写少（并发）
    ├── Set          无序、不可重复、无索引
    │   ├── HashSet           哈希表（数组+链表+红黑树）
    │   ├── LinkedHashSet     哈希表 + 双向链表，记录插入顺序
    │   └── TreeSet           红黑树，可排序
    └── Queue/Deque  队列/双端队列
        ├── ArrayDeque        数组实现的双端队列，推荐做栈/队列
        ├── PriorityQueue     二叉堆实现的优先队列
        └── LinkedList        链表实现的 List + Deque

Map（双列集合，独立体系，不继承 Collection）
├── HashMap            哈希表，无序，线程不安全
├── LinkedHashMap      哈希表 + 双向链表，有序
├── TreeMap            红黑树，按键排序
├── Hashtable          JDK1.0 遗留，全表 synchronized，已过时
└── ConcurrentHashMap  并发安全的 Map（CAS + synchronized）
```

★ **关键认知**：`Collection` 与 `Map` 是**两棵独立的树**——Map 不继承 Collection。`Collection` 的根还继承 `Iterable`（所以单列集合都可 `for-each`）。⚠ 判断"元素是否唯一/键是否重复"靠 `hashCode()` + `equals()`，判断"能否排序"靠 `Comparable` / `Comparator`，这是后续理解 Set 与 Map 的钥匙。

# 2. Collection 接口

## 2.1 通用方法

| 方法 | 说明 |
|------|------|
| `boolean add(E e)` | 添加元素。List 系列永远返回 true；Set 系列元素重复时返回 false（插入失败） |
| `void clear()` | 清空集合 |
| `boolean remove(E e)` | 删除第一个匹配的元素 |
| `boolean contains(Object o)` | 判断是否包含元素（内部调用 `equals()`） |
| `boolean isEmpty()` | 判断是否为空 |
| `int size()` | 返回元素个数 |

## 2.2 遍历方式

### 2.2.1 迭代器 Iterator

```java
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    System.out.println(it.next());
}
```

| 方法 | 说明 |
|------|------|
| `boolean hasNext()` | 判断是否还有元素 |
| `E next()` | 获取当前元素，并把指针后移一位 |
| `void remove()` | 删除 `next()` 返回的最后一个元素 |

### 2.2.2 增强 for（for-each）

```java
for (String s : list) {
    System.out.println(s);
}
```

★ **关键认知**：增强 for 底层就是**迭代器**，只是语法糖，因此所有 `Iterable` 实现类（所有 Collection 子类）和数组都能用。⚠ 遍历中修改遍历变量不影响原集合；遍历中增删集合会抛 `ConcurrentModificationException`（见第 13 篇）。

### 2.2.3 Lambda forEach（JDK8+）

```java
list.forEach(s -> System.out.println(s));   // 底层按子类不同调用不同遍历
list.forEach(System.out::println);           // 方法引用写法
```

# 3. 泛型

## 3.1 为什么需要泛型

JDK5 引入。在**编译阶段**约束集合存储的数据类型并做检查，把运行期的 `ClassCastException` 提前到编译期暴露，避免手动强转。

```java
List<String> list = new ArrayList<>();
list.add("hello");
String s = list.get(0);   // 无需强转
// list.add(1);           // ✗ 编译报错，不能存 Integer
```

## 3.2 泛型的定义

| 位置 | 写法 | 示例 |
|------|------|------|
| 泛型类 | `class 类名<T> {}` | `class Box<T> {}` |
| 泛型方法 | `<T> 返回值 方法名(T t) {}` | `public <T> T get(T t)` |
| 泛型接口 | `interface 接口<T> {}` | `interface Comparator<T> {}` |

## 3.3 通配符与上下界

| 通配符 | 含义 | 用途 |
|--------|------|------|
| `<?>` | 任意类型（无限定） | 只读遍历 |
| `<? extends T>` | T 及其子类（上界） | **读**数据（能拿 T，不能写） |
| `<? super T>` | T 及其父类（下界） | **写**数据（能写 T，读出来是 Object） |

★ **关键认知**（PECS 法则）：**Producer Extends，Consumer Super**——需要从集合"产"出值时用 `? extends`，需要往集合"消"费（写入）时用 `? super`。

## 3.4 类型擦除（简述）

Java 的泛型是**伪泛型**：编译时检查完类型，生成的字节码会**擦除**泛型信息，替换为上界类型（无上界则 `Object`），需要处插入强转。所以 `List<String>` 与 `List<Integer>` 运行期是同一个类，无法用 `instanceof` 判断泛型类型（完整原理见 JVM 笔记）。

# 4. List 接口

有序、可重复、有索引。相对 Collection 独有的方法：

| 方法 | 说明 |
|------|------|
| `void add(int index, E e)` | 在指定索引插入，原有元素后移 |
| `E remove(int index)` | 删除并返回指定索引元素 |
| `E set(int index, E e)` | 修改指定索引元素，返回旧元素 |
| `E get(int index)` | 获取指定索引元素 |
| `ListIterator<E> listIterator()` | 获取双向列表迭代器 |

`ListIterator` 支持 `hasPrevious()/previous()` 反向遍历、`add()/set()/remove()` 修改、`nextIndex()/previousIndex()` 取索引。

## 4.1 ArrayList

基于**动态数组**（`Object[] elementData`），查询快 O(1)，增删慢 O(n)，线程不安全。

### 4.1.1 构造方法

```java
new ArrayList<>();                       // 默认构造，JDK8+ 延迟初始化
new ArrayList<>(20);                     // 指定初始容量
new ArrayList<>(existingList);           // 用已有集合构造
new ArrayList<>(Arrays.asList(1, 2, 3)); // 快速构造（详见 9.2）
```

★ **关键认知**：JDK7 `new ArrayList<>()` 直接建长度 10 的数组；JDK8 起改为**懒加载**——先赋空数组，第一次 `add` 时才分配容量 10。⚠ 左边泛型必须写，右边可省略（菱形语法 `<>`）。

### 4.1.2 成员方法（常用）

| 方法 | 说明 |
|------|------|
| `add(E e)` / `add(int i, E e)` | 尾部追加 / 指定位置插入 |
| `remove(Object o)` / `remove(int i)` | 按值删第一个匹配 / 按索引删并返回 |
| `set(int i, E e)` / `get(int i)` | 改 / 查 |
| `size()` / `isEmpty()` | 元素个数 / 是否为空 |
| `contains(Object o)` / `indexOf` / `lastIndexOf` | 判断 / 首次索引 / 末次索引 |
| `clear()` | 清空 |
| `toArray()` / `toArray(T[] a)` | 转 Object[] / 指定类型数组 |
| `addAll(c)` / `removeAll(c)` / `retainAll(c)` | 并集添加 / 删交集 / 保留交集 |
| `forEach(Consumer)` / `removeIf(Predicate)` | JDK8+ 条件遍历 / 条件删除 |
| `replaceAll(UnaryOperator)` / `sort(Comparator)` | JDK8+ 全量替换 / 按比较器排序 |
| `subList(int from, int to)` | 子列表**视图** |

### 4.1.3 底层原理与扩容机制

- **数据结构**：`Object[] elementData`
- **扩容时机**：`add` 时数组已满（`size == elementData.length`）
- **扩容策略**：新容量 = 旧容量 + (旧容量 >> 1)，即 **1.5 倍**（10 → 15 → 22 → 33…）
- **扩容实现**：`Arrays.copyOf(elementData, newCapacity)` 复制到新数组，代价高

★ **关键认知**：扩容是"**建新数组 + 全量复制**"的成本，所以频繁扩容很伤性能。⚠ 已知大小时务必用 `new ArrayList<>(n)` 预估容量。删除元素时，ArrayList 会把最后一个元素位置置 `null`，避免对象无法被 GC 回收（内存泄漏）。

### 4.1.4 注意事项

**线程安全**：ArrayList 非线程安全，多线程可用 `Collections.synchronizedList(new ArrayList<>())` 或 `CopyOnWriteArrayList`。

**删除陷阱**（普通 for 循环删元素会跳过元素）：

```java
// ✗ 错误：删除后元素前移，i++ 会跳过一个元素
for (int i = 0; i < list.size(); i++) {
    if (条件) list.remove(i);
}
// ✓ 正确：倒序遍历
for (int i = list.size() - 1; i >= 0; i--) {
    if (条件) list.remove(i);
}
// ✓ 或：用迭代器 remove() / 用 removeIf()
```

**subList 陷阱**：`subList` 返回的是**视图**，修改它会反映到原 List；若此时对原 List 做结构性修改，子列表会抛 `ConcurrentModificationException`。

## 4.2 LinkedList

基于**双向链表**，查询慢 O(n)，增删快 O(1)。内部 `Node` 结构：

```java
private static class Node<E> {
    E item;
    Node<E> next;   // 后继
    Node<E> prev;   // 前驱
}
```

维护 `first`（头）与 `last`（尾）两个指针。

### 4.2.1 特有方法

| 方法 | 说明 |
|------|------|
| `addFirst(e)` / `addLast(e)` | 头部插入 / 尾部追加（同 `add`） |
| `getFirst()` / `getLast()` | 取头 / 取尾 |
| `removeFirst()` / `removeLast()` | 删并返回头 / 尾 |
| `push(e)` / `pop()` | 模拟栈：压入（=addFirst）/ 弹出（=removeFirst） |
| `offer(e)` / `poll()` | 模拟队列：入队（=addLast）/ 出队（=removeFirst） |

### 4.2.2 ArrayList vs LinkedList

| 维度 | ArrayList | LinkedList |
|------|-----------|-----------|
| 底层 | 动态数组 | 双向链表 |
| 随机访问 | O(1) | O(n) |
| 头/中间增删 | O(n)（元素移动） | O(1)（改指针） |
| 内存 | 连续、尾部有容量冗余 | 每节点额外存前后指针 |
| 实现接口 | List | List + Deque |

★ **关键认知**：**读多、按索引访问选 ArrayList（默认首选）；频繁头尾增删、当队列/栈用选 LinkedList 或 ArrayDeque**（`ArrayDeque` 性能更佳，见第 6 篇）。

## 4.3 Vector（了解）

ArrayList 的线程安全版，所有方法加 `synchronized`，性能差，**已过时不推荐**，用 `Collections.synchronizedList` 或 `CopyOnWriteArrayList` 替代。

## 4.4 CopyOnWriteArrayList（了解）

**写时复制**：写操作（add/set/remove）先复制底层数组，在新数组上改完再替换引用；读操作完全无锁。适合**读多写少**（配置、监听器列表），迭代器是快照，弱一致性。

# 5. Set 接口

无序、不可重复、无索引（由 `Iterable` 根保证可 for-each）。没有独有索引方法，主要重写 `Collection` 方法。

## 5.1 HashSet

底层是**哈希表**（JDK8 前数组+链表，JDK8 起数组+链表+红黑树）。元素按哈希值散列存放，故**无序**。

**去重原理**：`add` 时先算 `hashCode` 定位桶，再调 `equals` 判断。因此自定义对象要保证去重正确，必须**同时重写 `hashCode()` 与 `equals()`**：

- 两个对象 `equals` 相等 → `hashCode` 必须相等
- `hashCode` 相等 → `equals` 不一定相等（哈希冲突）
- 默认 `hashCode` 按地址计算，不重写则"属性值相同"的两个对象也会被当成不同元素

⚠ **HashSet 判断元素是否已存在的逻辑是 `hashCode` 快速筛选 + `equals` 精确确认**，只重写其一都会导致去重失效。

## 5.2 LinkedHashSet

在 HashSet 哈希表基础上，额外维护一条**双向链表**记录元素插入顺序，故**有序（按插入顺序）**。

## 5.3 TreeSet

底层是**红黑树**，**可排序**。两种排序规则：

| 规则 | 做法 | 优先级 |
|------|------|--------|
| 自然排序 | 元素实现 `Comparable` 接口的 `compareTo` | 默认 |
| 比较器排序 | 构造时传入 `Comparator` 比较器 | 传入则以它为准 |

`compareTo` 返回 <0 认为小、=0 认为已存在、>0 认为大。⚠ TreeSet 存自定义对象要么实现 `Comparable`，要么传 `Comparator`，否则运行时报 `ClassCastException`。

# 6. Queue / Deque

队列（先进先出 FIFO）与双端队列（两端都可进出）。`LinkedList` 由于实现了 Deque，也可当队列/栈用，但**首选 `ArrayDeque`**（基于数组，无 Node 对象开销、性能更好，Java 官方推荐替代 `Stack`）。

## 6.1 Queue 接口方法

| 方法 | 队列空/满时的行为 | 说明 |
|------|-----------------|------|
| `add(e)` | 抛异常 | 入队 |
| `offer(e)` | 返回 false | 入队（推荐） |
| `remove()` | 抛异常 | 出队 |
| `poll()` | 返回 null | 出队（推荐） |
| `element()` | 抛异常 | 看队头 |
| `peek()` | 返回 null | 看队头 |

`Deque` 额外提供 `addFirst/addLast/offerFirst/offerLast/removeFirst/pollFirst` 等双端方法，以及 `push/pop` 栈操作。

## 6.2 ArrayDeque

数组实现的**循环双端队列**。默认初始容量 16，每次扩容为 2 倍，存储元素不允许 `null`。做栈（`push/pop`）和队列（`offer/poll`）都比 `LinkedList`/`Stack` 高效。

## 6.3 PriorityQueue

**优先队列**，底层是 `Object[]` 实现的**二叉堆**（默认小顶堆），每次出队取"最小/优先级最高"的元素。元素必须实现 `Comparable` 或构造传入 `Comparator`。插入 `siftUp` 上浮、删除 `siftDown` 下沉，O(log n)。非线程安全（并发场景用 `PriorityBlockingQueue`）。

# 7. Map 接口

## 7.1 特点与常见 API

| 特点 | 说明 |
|------|------|
| 双列 | 一次存一对键值，键值合称 Entry 对象 |
| 键唯一 | 键不能重复，值可重复 |
| 对应关系 | 一个键只能找到自己对应的值 |

| 方法 | 说明 |
|------|------|
| `put(K k, V v)` | 添加，键已存在则**覆盖**（另有 `putIfAbsent` 不覆盖） |
| `remove(Object k)` | 按键删除键值对 |
| `clear()` | 清空 |
| `containsKey(k)` / `containsValue(v)` | 是否含键 / 值 |
| `isEmpty()` / `size()` | 空 / 键值对个数 |

## 7.2 遍历方式

```java
// 1. 键找值：keySet() + get()
for (String k : map.keySet()) {
    System.out.println(k + "=" + map.get(k));
}
// 2. 键值对：entrySet() + getKey()/getValue()
for (Map.Entry<String, Integer> e : map.entrySet()) {
    System.out.println(e.getKey() + "=" + e.getValue());
}
// 3. Lambda：forEach(BiConsumer)
map.forEach((k, v) -> System.out.println(k + "=" + v));
```

## 7.3 HashMap

底层**哈希表**（数组+链表+红黑树），与 HashSet 同构，**无序、不重复、无索引**（特性都由键决定）。

- **键唯一**：依赖键的 `hashCode()` 和 `equals()`，键存自定义对象需重写这两个方法（值不需要）
- **容量为 2 的幂、默认负载因子 0.75、扩容 2 倍**
- **线程不安全**：并发 put 会数据覆盖、size 不准（并发用 `ConcurrentHashMap`）

## 7.4 LinkedHashMap

HashMap 基础上每个键值对多一条**双向链表**记录插入顺序，故**有序**。设定 `accessOrder=true` 并重写 `removeEldestEntry` 可实现 **LRU 缓存**。

## 7.5 TreeMap

底层**红黑树**，**按键排序**（与 TreeSet 同构）。默认按键自然排序（键实现 `Comparable`），或构造传入 `Comparator`。

## 7.6 Hashtable（了解）

JDK1.0 遗留，全表 `synchronized`，不允许 `null` 键/值，性能差，**已过时不推荐**。

## 7.7 ConcurrentHashMap（了解）

并发安全 Map。JDK7 用分段锁 Segment；JDK8 起改用 **CAS + synchronized（锁桶头节点）**，get 无锁、put 锁单桶、支持多线程协同扩容。高并发场景 Map 首选（深度见并发笔记）。

# 8. 包装类

## 8.1 为什么需要包装类

- 集合只能存对象，不能存基本类型
- 提供丰富的转换/工具方法
- 支持 `null` 表达"不存在"

| 基本类型 | 包装类 | 父类 |
|---------|--------|------|
| byte | Byte | Number |
| short | Short | Number |
| int | Integer | Number |
| long | Long | Number |
| float | Float | Number |
| double | Double | Number |
| char | Character | Object |
| boolean | Boolean | Object |

## 8.2 装箱与拆箱

```java
// 装箱：基本类型 → 包装类（JDK5+ 自动）
Integer i = Integer.valueOf(100);  // 手动
Integer i = 100;                   // 自动
// 拆箱：包装类 → 基本类型
int n = i.intValue();              // 手动
int n = i;                         // 自动
```

⚠ **拆箱 NPE**：`Integer i = null; int n = i;` 拆箱调 `i.intValue()` 抛 NPE。

## 8.3 缓存机制（享元模式）

`valueOf()` 对常用值做缓存，`new` 不使用缓存：

```java
Integer a = 127, b = 127;
a == b;   // true（缓存中同一对象）
Integer c = 128, d = 128;
c == d;   // false（超出缓存，各自新建）
```

| 包装类 | 缓存范围 |
|--------|---------|
| Byte | -128 ~ 127（全部） |
| Short / Long / Integer | -128 ~ 127 |
| Character | 0 ~ 127（ASCII） |
| Boolean | true / false 均静态常量 |
| Float / Double | 无缓存 |

Integer 上限可用 `-Djava.lang.Integer.IntegerCache.high=500` 调高（仅调高不能调低）。

## 8.4 常用方法

```java
Integer.parseInt("100");            // "100" → 100
Integer.valueOf(100);               // 装箱（含缓存）
Integer.toString(100);              // "100"
Integer.toBinaryString(6);          // "110"
Character.isDigit('5');             // true
Boolean.parseBoolean("true");       // 仅 "true"（忽略大小写）为 true
```

⚠ `parseInt`/`valueOf` 遇非数字字符串抛 `NumberFormatException`。

## 8.5 包装类与字符串转换

```java
// 包装类 → String
Integer.toString(100);   // "100"
String.valueOf(100);     // "100"
// String → 包装类
Integer.valueOf("100");  // Integer
Integer.parseInt("100"); // int
```

# 9. Collections 工具类

| 方法 | 说明 |
|------|------|
| `addAll(Collection c, T... e)` | 批量添加（内部用可变参数） |
| `shuffle(List)` | 打乱顺序 |
| `sort(List)` / `sort(List, Comparator)` | 自然排序 / 指定比较器排序 |
| `binarySearch(List, key)` | 二分查找（要求列表有序） |
| `max(Collection)` / `min(Collection)` | 最大 / 最小 |
| `swap(List, i, j)` | 交换指定索引 |
| `copy(dest, src)` | 复制 |
| `fill(List, obj)` | 用指定对象替换所有元素 |
| `synchronizedList/reverse` | 线程安全包装 / 反转 |

# 10. 增强 for 循环

```java
for (元素类型 变量名 : 数组或 Iterable 实现类) {
    // 操作变量名
}
```

★ **关键认知**：**底层就是迭代器**。⚠ 遍历中修改变量不影响原集合；遍历中增删集合会抛异常（见第 13 篇）。

# 11. 可变参数（JDK5+）

```java
public static int sum(int... nums) { /* 内部按数组处理 */ }
```

- 本质：**一个数组**
- 规则：形参列表中**最多一个**，且必须**放最后**

# 12. 不可变集合

JDK9+ 用静态工厂创建真正的不可变集合：

```java
List.of(1, 2, 3);
Set.of("a", "b");
Map.of("k1", 1, "k2", 2);        // 最多 20 个参数（10 个键值对）
Map.ofEntries(Map.entry("a", 1)); // 键值对多时用 ofEntries
Map.copyOf(map);                  // JDK10，创建指定 Map 的不可变副本
```

⚠ 这些集合**不允许 `null` 元素**，且任何修改操作都会抛 `UnsupportedOperationException`。

# 13. 迭代器与并发修改异常（fail-fast）

集合内部维护 `modCount`（修改计数），迭代器创建时记录 `expectedModCount`，每次 `next()`/`remove()` 都会校验二者是否相等：

- **用迭代器自身的 `remove()`** → 同步更新 `expectedModCount`，安全
- **用集合的方法增删**（`list.add/remove`）→ 只改 `modCount`，迭代时检测到不一致 → 抛 `ConcurrentModificationException`

★ **关键认知**（fail-fast）：这是集合的"快速失败"机制，只在**单线程迭代时误用集合方法增删**时暴露，不能 100% 检测多线程并发修改。多线程安全遍历用 `CopyOnWriteArrayList`（快照读，不抛异常）。