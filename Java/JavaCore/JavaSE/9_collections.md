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

集合解决的是数组**长度不可变**与**只能存固定类型**两大痛点。但集合只能存引用类型：往集合里放基本类型会发生**自动装箱**，取出时发生**拆箱**，这也埋下了 `null` 拆箱 NPE 的隐患（见第 8 节包装类）。

## 1.2 集合体系结构

`Collection` 与 `Map` 是**两棵完全独立的树**——`Map` 不继承 `Collection`，它自己就是根接口。`Collection` 又继承 `Iterable`，因此所有单列集合都能用 `for-each`；而 `Map` 不能直接遍历，必须先转成单列集合。

```text
Iterable（可迭代顶层接口，实现它才能用 for-each）
└── Collection（单列集合的根接口）
    ├── List          有序、可重复、有索引
    │   ├── ArrayList           动态数组，查快增删慢，线程不安全（默认首选）
    │   ├── LinkedList          双向链表，增删快查慢，同时实现 Deque
    │   ├── Vector              数组 + 全方法 synchronized，已过时
    │   └── CopyOnWriteArrayList  写时复制，读多写少（并发包）
    ├── Set           不可重复、无索引
    │   ├── HashSet             哈希表（数组 + 链表 + 红黑树）
    │   ├── LinkedHashSet       哈希表 + 双向链表，记住插入顺序
    │   └── TreeSet             红黑树，可排序
    └── Queue / Deque 队列 / 双端队列
        ├── ArrayDeque          循环数组实现的双端队列，推荐做栈 / 队列
        ├── PriorityQueue       二叉堆实现的优先队列
        └── LinkedList          链表实现，兼作 Queue

Map（双列集合，独立体系，不继承 Collection）
├── HashMap              哈希表，无序，线程不安全（默认首选）
├── LinkedHashMap        哈希表 + 双向链表，有序（accessOrder=true 可做 LRU）
├── TreeMap              红黑树，按键排序
├── Hashtable            JDK1.0 遗留，全表 synchronized，已淘汰
└── ConcurrentHashMap    并发安全，CAS + synchronized
```

两棵树的差异可以总结成一张表：

| 体系 | 是否继承 `Collection` | 能否直接 `for-each` | 元素唯一性判断 |
|------|----------------------|--------------------|----------------|
| `List` / `Set` / `Queue` | 是 | 能 | 元素是否唯一：`hashCode()` + `equals()` |
| `Map` | 否 | 否，需先转 `keySet()` / `entrySet()` | **键**是否唯一：`hashCode()` + `equals()`（值不参与） |

判断"元素是否唯一、键是否重复"靠 `hashCode()` + `equals()` 契约，判断"能否排序"靠 `Comparable` / `Comparator` 接口，这是后续理解 Set 与 Map 的钥匙。

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

`Iterator` 是所有单列集合统一的遍历入口，屏蔽了底层是数组还是链表的差异。

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

增强 for 底层就是**迭代器**，只是编译器做的语法糖，因此所有 `Iterable` 实现类（所有 Collection 子类）和数组都能用。遍历中给遍历变量重新赋值不影响原集合；但遍历中**增删集合**会抛 `ConcurrentModificationException`（原理见第 13 节）。

### 2.2.3 Lambda forEach（JDK8+）

```java
list.forEach(s -> System.out.println(s));   // 底层按子类不同调用不同遍历
list.forEach(System.out::println);           // 方法引用写法
```

三种遍历方式怎么选：

| 方式 | 能否在遍历中安全删除 | 能否拿到索引 | 说明 |
|------|--------------------|-------------|------|
| 迭代器 `Iterator` | **能**（用 `it.remove()`） | 否 | 最灵活，支持删除 |
| 增强 for | 否（会抛 CME） | 否 | 写法最简洁，只读遍历首选 |
| Lambda `forEach` | 否（会抛 CME） | 否 | 函数式风格，可与 Stream 衔接 |

# 3. 泛型

## 3.1 为什么需要泛型

泛型是 JDK5 引入的。它在**编译阶段**约束集合存储的数据类型并做检查，把原本运行期才暴露的 `ClassCastException` 提前到编译期，同时省去了繁琐的手动强转。

```java
// 没有泛型：什么都能存，取出来全是 Object，强转错了运行期才炸
List list = new ArrayList();
list.add("hello");
list.add(100);
String s = (String) list.get(1);   // ✗ 运行期 ClassCastException

// 有泛型：类型在编译期就锁死
List<String> list2 = new ArrayList<>();
list2.add("hello");
String s2 = list2.get(0);          // 无需强转
// list2.add(100);                 // ✗ 编译报错，不能存 Integer
```

## 3.2 泛型的定义

| 位置 | 写法 | 示例 |
|------|------|------|
| 泛型类 | `class 类名<T> {}` | `class Box<T> { private T value; }` |
| 泛型方法 | `<T> 返回值 方法名(T t) {}` | `public <T> T get(T t) { return t; }` |
| 泛型接口 | `interface 接口<T> {}` | `interface Comparator<T> {}` |

泛型类里的类型参数 `T` 在**整个类**内可用；泛型方法则把 `<T>` 声明在返回值之前，只在**该方法**内有效，即使所在类不是泛型类也能用。

```java
public class Box<T> {              // 泛型类
    private T value;
    public void set(T value) { this.value = value; }
    public T get() { return value; }

    // 泛型方法：自己声明 T，与类的 T 相互独立
    public static <T> T firstOf(List<T> list) {
        return list.isEmpty() ? null : list.get(0);
    }
}
```

## 3.3 通配符与上下界

| 通配符 | 含义 | 用途 |
|--------|------|------|
| `<?>` | 任意类型（无限定） | 只读遍历 |
| `<? extends T>` | T 及其子类（上界） | **读**数据（能拿 T，不能写） |
| `<? super T>` | T 及其父类（下界） | **写**数据（能写 T，读出来是 Object） |

**PECS 原则**：**Producer Extends，Consumer Super**——需要从集合里"产"出值时用 `? extends`，需要往集合里"消"费（写入）时用 `? super`。

```java
// Producer：只读，用 extends —— 可以当作 Number 安全取出
static double sum(List<? extends Number> nums) {
    double s = 0;
    for (Number n : nums) s += n.doubleValue();
    return s;                       // 可以读；但不能 nums.add(...)，因为不知道具体子类型
}

// Consumer：只写，用 super —— 可以安全写入 Integer
static void fill(List<? super Integer> list) {
    for (int i = 1; i <= 3; i++) list.add(i);   // 可以写
    // Integer x = list.get(0);   // ✗ 读出来只能当 Object
}
```

> **注意**：`<? extends T>` 不能写入（除了写 `null`），因为编译器无法确定集合的真实元素类型是 `T` 的哪个子类；`<? super T>` 能写入 `T`，但读出来只能赋给 `Object`。简单记：**能读的不能写，能写的只能读成 Object**。

## 3.4 类型擦除及其后果

Java 的泛型是**伪泛型**：编译时检查完类型，生成的字节码会把泛型信息**擦除**，替换为上界类型（无上界则 `Object`），需要处再插入强转。所以 `List<String>` 与 `List<Integer>` 在运行期是**同一个类**。擦除是为了兼容 JDK5 之前的老代码（那时没有泛型），代价是一系列限制：

| 后果 | 说明 |
|------|------|
| 无法用 `instanceof` 判断泛型 | `x instanceof List<String>` 编译报错，只能写 `x instanceof List` |
| 不能创建泛型数组 | `new T[]`、`new List<String>[]` 都不行，只能 `(List<String>[]) new List[...]` 并加 `@SuppressWarnings` |
| 不能 `new T()` | 运行期拿不到 `T` 的 Class，只能靠反射或传入 `Supplier<T>` 工厂 |
| 静态成员不能用类泛型参数 | `static T value;` 编译报错（静态成员属于类，其类型参数在类级别） |
| 基本类型不能作泛型实参 | 必须用包装类：`List<int>` ✗，应写 `List<Integer>` |
| 重载可能冲突 | `void f(List<String>)` 与 `void f(List<Integer>)` 报"same erasure"，因为擦除后签名相同 |

> **注意**：正因为"编译期检查、运行期擦除"，泛型只在编译期提供类型安全，**运行期仍然可能因强转失败抛 `ClassCastException`**（比如把一个 `List<Integer>` 通过裸类型 `List` 塞进字符串再取出）。编译器会在关键位置自动插入 `checkcast` 指令来兜底。

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

基于**动态数组**（`Object[] elementData`），查询快 O(1)，增删慢 O(n)，线程不安全。**绝大多数场景下它都是 List 的首选**。

### 4.1.1 构造方法

```java
new ArrayList<>();                       // 默认构造，JDK8+ 懒加载
new ArrayList<>(20);                     // 指定初始容量
new ArrayList<>(existingList);           // 用已有集合构造
new ArrayList<>(Arrays.asList(1, 2, 3)); // 快速构造（注意 Arrays.asList 固定长度的坑，见第 12 节）
```

JDK7 的 `new ArrayList<>()` 会立刻创建长度 10 的数组；JDK8 起改为**懒加载**——先赋一个空数组，直到第一次 `add` 才真正分配容量 10（`DEFAULT_CAPACITY`），避免了"创建却不用"的内存浪费。此外，菱形语法 `<>` 允许右边省略类型，但**左边泛型必须写**（否则退化成裸类型，失去编译期检查）。

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

- **数据结构**：`transient Object[] elementData`，用数组承载所有元素
- **默认容量**：`DEFAULT_CAPACITY = 10`（JDK8 首次 `add` 时才分配）
- **扩容时机**：`add` 时数组已满（`size == elementData.length`）
- **扩容策略**：新容量 = 旧容量 + (旧容量 >> 1)，即 **1.5 倍**
- **扩容实现**：`Arrays.copyOf(elementData, newCapacity)`，底层是 `System.arraycopy` 的整块内存复制

容量增长序列为 **10 → 15 → 22 → 33 → 49 → …**（每次都向下取整）。

```text
   扩容前（容量 10，已满）                扩容后（容量 15，新数组）
   ┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐
   │ 1│ 2│ 3│ 4│ 5│ 6│ 7│ 8│ 9│10│
   └──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘
                   │  Arrays.copyOf(elementData, 15)：整块复制
                   ▼
   ┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐
   │ 1│ 2│ 3│ 4│ 5│ 6│ 7│ 8│ 9│10│  │  │  │  │  │
   └──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘
     ↑ 老数组被丢弃、等待 GC；elementData 引用指向新数组

   若一次性 addAll 超过现有容量，则直接扩到足够大，不会按 1.5 倍反复扩
   （新容量 = max(所需容量, 旧容量 × 1.5)）
```

扩容是"**建新数组 + 全量复制**"的高成本操作，频繁扩容很伤性能，因此**已知大小时务必用 `new ArrayList<>(n)` 预估容量**。另外删除元素后，`ArrayList` 会把 `size` 边界外的引用置为 `null`（`elementData[--size] = null`），否则这个槽位仍然强引用着对象，导致它无法被 GC 回收而内存泄漏。

各类操作的时间复杂度：

| 操作 | 复杂度 | 原因 |
|------|--------|------|
| 按下标访问 `get(i)` / `set(i)` | O(1) | 数组随机访问 |
| 尾部追加 `add(e)` | 均摊 O(1) | 偶尔触发扩容，均摊下来是常数 |
| 中间插入 / 删除 | O(n) | 需要移动后续所有元素 |
| 按值查找 `contains` / `indexOf` | O(n) | 需要逐个 `equals` 比较 |

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
// ✓ 或：用迭代器的 remove() / 用 removeIf()
```

**subList 陷阱**：`subList` 返回的是**视图**而非新集合，修改它会反映到原 List；若此时对原 List 做了结构性修改（增删），子列表后续操作会抛 `ConcurrentModificationException`。

## 4.2 LinkedList

基于**双向链表**，查询慢 O(n)，增删快 O(1)（前提是已经定位到节点）。每个节点是一个 `Node` 对象：

```java
private static class Node<E> {
    E item;
    Node<E> next;   // 后继
    Node<E> prev;   // 前驱
}
```

底层维护 `first`（头）与 `last`（尾）两个指针：

```text
      first                                        last
        │                                            │
        ▼                                            ▼
   ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐
   │   A    │◀─▶│   B    │◀─▶│   C    │◀─▶│   D    │
   └────────┘   └────────┘   └────────┘   └────────┘
   首节点的 prev 与尾节点的 next 都指向 null

   查第 i 个元素：只能从头（或从尾，看 i 靠哪边近）逐个 next/prev 走，所以是 O(n)
   在已知节点前后插入/删除：只改几个指针，O(1)
```

**为什么查询慢**：链表在内存中**不是连续的**，没有下标寻址能力，想访问第 i 个元素只能从 `first`（或 `last`）一路走过 i 个节点，无法像数组那样一步跳过去。所以"增删快 O(1)"严格来说是指**已知位置后的指针改写**；如果按索引增删，定位本身仍要 O(n)。

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
| 内存 | 连续、尾部有容量冗余 | 每个节点额外存前后指针（开销更大） |
| 实现接口 | List | List + Deque |
| 缓存友好度 | 高（内存连续） | 低（节点分散） |

**选择建议**：**读多、按索引访问选 ArrayList（默认首选）**；只在确实需要频繁头尾增删、当队列/栈用时考虑 `ArrayDeque`（性能比 LinkedList 更好，见第 6 节）。实践中 LinkedList 因为缓存不友好、每节点额外开销大，反而很少是正确选择。

## 4.3 Vector（了解）

ArrayList 的线程安全版，所有方法都加 `synchronized`，锁粒度大、性能差，**已过时不推荐**。现代做法是用 `Collections.synchronizedList` 或 `CopyOnWriteArrayList` 替代。

## 4.4 CopyOnWriteArrayList（了解）

**写时复制**：写操作（add/set/remove）先复制一份底层数组，在新数组上改完，再把内部引用指向新数组；读操作完全无锁。适合**读多写少**场景（配置、监听器列表）。它的迭代器基于**快照**，遍历期间看不到后续的修改，属于**弱一致性**，因此不会抛 `ConcurrentModificationException`。

# 5. Set 接口

无序、不可重复、无索引（由 `Iterable` 根保证可 for-each）。没有独有索引方法，主要重写 `Collection` 的方法。

## 5.1 HashSet

底层是一个 `HashMap`——`HashSet` 用一个固定的 `PRESENT` 对象作为所有键的"值"，只把元素本身当**键**存进去。所以"元素不可重复"等价于"HashMap 的键不可重复"。元素按哈希值散列存放，因此**无序**。

### 5.1.1 去重原理：hashCode 与 equals 契约

`add(e)` 的执行流程是：

1. 计算 `e` 的 `hashCode()`，经扰动后对数组长度取模，定位到某个"桶"（数组下标）；
2. 该桶为空 → 直接放入；
3. 该桶非空 → 逐个与桶内元素用 `equals()` 比较；全部不相等则加入（挂到链表 / 红黑树上），有相等则视为重复、不再插入。

所以"元素是否已存在"= **先用 `hashCode` 快速筛选，再用 `equals` 精确确认**。这带来三条必须遵守的契约：

- 两个对象 `equals` 相等 → `hashCode` **必须**相等；
- `hashCode` 相等 → `equals` 不一定相等（这是**哈希冲突**，正常现象）；
- 默认 `hashCode` 由对象地址派生，所以**自定义对象必须同时重写 `hashCode()` 与 `equals()`**，否则"属性值相同"的两个对象会被当成不同元素。

> **陷阱**：只重写 `equals` 不重写 `hashCode`，两个"相等"的对象会算出不同的哈希、落进不同的桶，去重失效；只重写 `hashCode` 不重写 `equals`，它们会落进同一个桶但 `equals` 返回 false，去重同样失效。**两者必须成对重写**。

### 5.1.2 哈希表底层结构

JDK8 之前是"数组 + 链表"；JDK8 起是"**数组 + 链表 + 红黑树**"：

```text
   table：Node[] 数组，长度恒为 2 的幂（默认 16）
   下标 = (数组长度 - 1) & 扰动后的 hash

   index
     ┌───┐
   0 │   │──▶ null
     ├───┤
   1 │ ●─┼──▶ [ k1 , v1 ] ─▶ [ k2 , v2 ] ─▶ null   ← 哈希冲突：同一个桶用链表串起来
     ├───┤
   2 │   │──▶ null
     ├───┤
   3 │ ●─┼──▶ [ k3 , v3 ] ─▶ null
     ├───┤
   4 │ ●─┼──▶ ┌───────────┐
     │   │    │ 红黑树的根  │                       ← 链表长度 ≥ 8 且表长 ≥ 64 时树化
     ├───┤    └────┬──────┘
   5 │   │──▶ null │
     ├───┤     ┌───┴───┐
   … │   │     ▼       ▼
     └───┘   [k]      [k]   （红黑树左右子节点，查询从 O(n) 降为 O(log n)）
```

JDK7 用纯链表，最坏情况下所有元素撞到同一个桶，退化成 O(n) 查询，甚至被构造恶意数据变成哈希碰撞攻击。JDK8 引入红黑树，把最坏查询降到 O(log n)。

### 5.1.3 扰动函数

定位下标靠 `(n - 1) & hash`，其中 `hash` 不是直接用 `hashCode()`，而是先过一遍**扰动函数**：

```java
static final int hash(Object key) {
    int h;
    // 高 16 位异或低 16 位：让高位也参与寻址，降低冲突概率
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

因为数组长度 `n` 是 2 的幂，`n - 1` 的低位全是 1，`(n-1) & hash` 等价于对 `n` 取模（比 `%` 更快），但它**只用到了 hash 的低位**。如果很多对象的 `hashCode` 高位不同、低位相同，就会全部撞在一起。扰动函数把高 16 位异或到低 16 位，让高位信息也参与定位，使分布更均匀。

### 5.1.4 负载因子、扩容与树化

| 常量 | 值 | 含义 |
|------|-----|------|
| `DEFAULT_INITIAL_CAPACITY` | 16 | 默认数组长度 |
| `DEFAULT_LOAD_FACTOR` | 0.75 | 默认负载因子 |
| `TREEIFY_THRESHOLD` | 8 | 链表长度达到 8 时尝试树化 |
| `UNTREEIFY_THRESHOLD` | 6 | 树节点数降到 6 时退化回链表 |
| `MIN_TREEIFY_CAPACITY` | 64 | 树化的前提是数组长度至少 64 |

**负载因子与扩容**：阈值 `threshold = capacity × loadFactor`。当元素个数超过阈值时触发 `resize()`，容量翻倍（×2），并把所有元素重新散列到新数组。例如默认容量 16、阈值 12，放入第 13 个元素时扩容到 32、阈值变 24。

为什么是 **0.75**？这是**空间与时间的折中**：太小（如 0.5）则数组很快被填满、空间浪费严重；太大（如 1.0）则冲突概率陡增、链表变长、查询变慢。0.75 是经验上的均衡点——官方注释指出，在随机哈希下，一个桶里链表长度达到 8 的概率约为千万分之六，几乎不可能，所以红黑树极少真正派上用场。

为什么容量必须是 **2 的幂**？这样 `(n-1) & hash` 才能等价于取模，且扩容时每个元素的新位置只有两种可能——**要么留在原下标，要么原下标 + 旧容量**（因为只需看 hash 中新增的那一位是 0 还是 1），迁移效率极高。

**树化阈值 8 与退化阈值 6 为什么不等**？中间留一段缓冲区，避免元素数在 7 和 8 之间来回抖动时反复地"树化 → 退化 → 再树化"，浪费性能。

### 5.1.5 JDK8 把"头插"改成"尾插"解决了什么问题

JDK7 的 HashMap 在 `resize()` 迁移链表时用**头插法**（`e.next = newTable[i]; newTable[i] = e;`），新节点被插到桶的链表头部。单线程下没问题，但在**并发扩容**时，两个线程同时迁移同一条链表，可能把节点的 `next` 指针改成一个**环**。此后任何 `get` 走到这个桶都会在环里死循环、CPU 飙到 100%。

JDK8 改成**尾插法**：迁移时按 `(e.hash & oldCap) == 0` 把一条链表拆成 `lo`、`hi` 两条，保持节点的**原相对顺序**，从根本上杜绝了环形链表的形成（顺带也修正了 JDK7 头插导致扩容后链表顺序反转的问题）。

> **注意**：这**不代表 JDK8 的 HashMap 就线程安全了**——并发 put 仍然可能丢失数据、`size` 计数不准。要并发必须用 `ConcurrentHashMap`（见 7.7）。

## 5.2 LinkedHashSet

`HashSet` 的子类，内部直接基于 `LinkedHashMap`。它在哈希表的每个节点上**额外挂了两个指针**（`before` / `after`），把所有节点串成一条双向链表，从而记住元素的**插入顺序**。因此它的迭代顺序就是插入顺序，性能略低于 `HashSet`、空间开销略大。

## 5.3 TreeSet

底层是**红黑树**（一种自平衡的二叉查找树），因此元素**可排序**，增删查均为 O(log n)。红黑树通过旋转和变色自动维持"近似平衡"，保证树高始终是 O(log n)，不会像普通二叉查找树那样在有序输入下退化成链表。

排序规则有两种：

| 规则 | 做法 | 优先级 |
|------|------|--------|
| 自然排序 | 元素实现 `Comparable` 接口的 `compareTo` | 默认 |
| 比较器排序 | 构造时传入 `Comparator` 比较器 | 传入则以它为准 |

`compareTo` / `compare` 返回 **< 0** 认为小、**= 0** 认为已存在、**> 0** 认为大。注意 TreeSet 的**去重依据是"比较结果是否为 0"，而不是 `equals()`**——只要 `compareTo` 返回 0，就认为重复。

> **陷阱**：TreeSet 存自定义对象时，要么实现 `Comparable`，要么构造时传 `Comparator`，否则运行时抛 `ClassCastException`。此外 TreeSet **不允许存 `null`**（无法比较，插第一个 null 可以，插入之后会 NPE / 抛异常）。

三种 Set 实现的选择：

| 实现 | 底层 | 顺序 | 增删查性能 | 允许 `null` | 去重依据 |
|------|------|------|-----------|------------|----------|
| `HashSet` | 哈希表 | 无序 | O(1) | 允许一个 `null` | `hashCode` + `equals` |
| `LinkedHashSet` | 哈希表 + 双向链表 | 插入顺序 | 略慢于 HashSet | 允许 | `hashCode` + `equals` |
| `TreeSet` | 红黑树 | 排序顺序 | O(log n) | 不允许 | `compareTo` / `compare` 返回 0 |

**选择建议**：去重为主、不在乎顺序用 `HashSet`；要保留插入顺序用 `LinkedHashSet`；要按键排序或需要范围查询（`first()`、`floor()`、`subSet()`）用 `TreeSet`。

# 6. Queue / Deque

队列（先进先出 FIFO）与双端队列（两端都可进出）。`LinkedList` 由于实现了 `Deque`，也可当队列/栈用，但**首选 `ArrayDeque`**（基于数组，无 Node 对象开销、缓存更友好、性能更好，Java 官方推荐用它替代古老的 `Stack`）。

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

数组实现的**循环双端队列**：

```text
   head                tail
     │                  │
     ▼                  ▼
   ┌───┬───┬───┬───┬───┬───┬───┬───┐
   │   │ B │ C │ D │   │   │   │   │   数组首尾相连成环，head/tail 到达末尾就绕回下标 0
   └───┴───┴───┴───┴───┴───┴───┴───┘
     ↑ index 0                  ↑ index 7（下一个位置可能是 0）
```

默认初始容量 16，每次扩容为 2 倍，存储元素**不允许 `null`**（`null` 被用作"标记空位"的哨兵）。做栈（`push/pop`）和队列（`offer/poll`）都比 `LinkedList` / `Stack` 高效。

## 6.3 PriorityQueue

**优先队列**，底层是 `Object[]` 实现的**二叉堆**（默认小顶堆），每次出队取"最小（优先级最高）"的元素。元素必须实现 `Comparable` 或构造时传入 `Comparator`。插入时 `siftUp` 上浮、删除时 `siftDown` 下沉，复杂度均为 O(log n)。非线程安全（并发场景用 `PriorityBlockingQueue`）。

# 7. Map 接口

## 7.1 特点与常见 API

| 特点 | 说明 |
|------|------|
| 双列 | 一次存一对键值，键值合称 Entry 对象 |
| 键唯一 | 键不能重复，值可重复 |
| 对应关系 | 一个键只能找到自己对应的值 |

| 方法 | 说明 |
|------|------|
| `put(K k, V v)` | 添加，键已存在则**覆盖**并返回旧值（键不存在返回 null） |
| `remove(Object k)` | 按键删除键值对，返回旧值 |
| `clear()` | 清空 |
| `containsKey(k)` / `containsValue(v)` | 是否含键 / 值 |
| `isEmpty()` / `size()` | 空 / 键值对个数 |
| `getOrDefault(k, d)` | 取值，键不存在返回默认值 `d` |
| `putIfAbsent(k, v)` | 仅当键不存在时才放入 |
| `computeIfAbsent(k, fn)` | 键不存在时用函数算出值再放入（常用于构建多值 Map） |
| `merge(k, v, fn)` | 键存在则用函数合并，不存在则直接放入 |

```java
// computeIfAbsent 经典用法：把 List 当 Map 的值，省去"先判空再 put"的样板代码
Map<String, List<String>> groups = new HashMap<>();
for (String word : words) {
    groups.computeIfAbsent(word.substring(0, 1), k -> new ArrayList<>()).add(word);
}
```

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

`entrySet()` 只遍历一次、直接拿到键值对，比 `keySet()` + `get()` 少一次查表，是**性能最优的遍历方式**。

## 7.3 HashMap

底层**哈希表**（数组 + 链表 + 红黑树），与 `HashSet` 完全同构（`HashSet` 就是一个只用键的 `HashMap`），**无序、不重复、无索引**，这些特性都由**键**决定。哈希表的完整原理（`hashCode`/`equals` 契约、扰动函数、树化阈值 8 与退化阈值 6、负载因子 0.75、扩容 resize、头插改尾插）见 **5.1**，此处不再重复。HashMap 特有的要点：

- **键唯一**：依赖键的 `hashCode()` 和 `equals()`，键存自定义对象需重写这两个方法（**值不需要**）
- **容量为 2 的幂、默认负载因子 0.75、超阈值扩容 2 倍**
- **允许一个 `null` 键、多个 `null` 值**（`null` 键的 hash 按 0 处理，固定落在下标 0 的桶）
- **线程不安全**：并发 put 会数据覆盖、`size` 不准（并发用 `ConcurrentHashMap`）

## 7.4 LinkedHashMap

`HashMap` 的子类，在每个 Entry 上多挂一条**双向链表**，因此**有序**。它有两个开关：

- 默认 `accessOrder = false`：链表维护**插入顺序**；
- 设 `accessOrder = true`：链表维护**访问顺序**——每次 `get`/`put` 命中某个键，就把该节点移动到链表尾部。

配合重写 `removeEldestEntry`（返回值 = 是否删除最老的节点），就能用几行代码实现一个 **LRU 缓存**：

```java
// 一个固定容量 3 的 LRU 缓存：超过容量就淘汰最久未被访问的键
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    public LRUCache(int capacity) {
        super(capacity, 0.75f, true);   // 第三个参数 accessOrder = true
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;       // 超过容量 => 删除链表头部（最久未访问）
    }
}

// 使用
LRUCache<String, Integer> cache = new LRUCache<>(3);
cache.put("a", 1);
cache.put("b", 2);
cache.put("c", 3);
cache.get("a");          // 访问 a，a 被移到链表尾部（变为"最新"）
cache.put("d", 4);       // 容量超限，淘汰最久未访问的 b
System.out.println(cache.keySet());   // [c, a, d]
```

## 7.5 TreeMap

底层**红黑树**，**按键排序**（与 `TreeSet` 同构）。默认按**键的自然顺序**排序（键实现 `Comparable`），或构造时传入 `Comparator`。因为是有序的，它还支持范围查询：`firstKey()`、`lastKey()`、`floorKey()`、`ceilingKey()`、`subMap()`、`headMap()`、`tailMap()`。

> **注意**：TreeMap 的"键唯一"同样是按 `compareTo` / `compare` 返回 0 判断，而不是 `equals()`——这一点和 TreeSet 一致。

## 7.6 Hashtable（了解）

JDK1.0 遗留类，被淘汰的原因很明确：

- **锁粒度太大**：几乎所有方法都加 `synchronized`，锁的是整个表，并发性能极差；
- **不允许 `null` 键/值**（而 HashMap 允许），使用上更拘束；
- **API 命名不一致**：`elements()`、`keys()` 等老方法与新集合体系风格脱节；
- 已被完全取代：**非并发场景用 `HashMap`，并发场景用 `ConcurrentHashMap`**。

## 7.7 ConcurrentHashMap（了解）

并发安全 Map，是并发场景的首选。它的实现经历了两次演进：

| 版本 | 加锁方式 | 特点 |
|------|----------|------|
| JDK7 | **分段锁 Segment** | 把表分成默认 16 个 `Segment`（继承 `ReentrantLock`），每段独立加锁，不同段可并行写；并发度 = 段数 |
| JDK8+ | **CAS + synchronized 锁单桶** | 取消 Segment，锁粒度细化到"桶的头节点"；`get` 完全无锁，`put` 只锁一个桶，并发度大幅提升 |

JDK8 的几个关键设计：`get` 不加锁（靠 `volatile` 读保证可见性）；`put` 时若目标桶为空则用 **CAS** 直接放，不为空才 `synchronized` 锁住桶头；扩容支持**多线程协同迁移**（`helpTransfer`）；`size` 借鉴 `LongAdder` 的分段计数思想，避免全局竞争。它同样**不允许 `null` 键/值**（并发下无法区分"不存在"和"值为 null"）。

> **注意**：锁的底层原理（对象头、锁升级、AQS、CAS）集中在第 30 篇展开，本篇只讲演进脉络。

# 8. 包装类

## 8.1 为什么需要包装类

- 集合只能存对象，不能存基本类型；
- 提供丰富的转换/判断工具方法（如 `Integer.parseInt`、`Character.isDigit`）；
- 支持用 `null` 表达"值不存在"。

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

> **陷阱**：`Integer i = null; int n = i;` 拆箱时会调用 `i.intValue()`，对 `null` 调用成员方法直接抛 `NullPointerException`。这是集合里取出元素后参与运算时最常见的 NPE 来源。

## 8.3 缓存机制（享元模式）

`valueOf()` 对常用值做缓存，`new` 不使用缓存：

```java
Integer a = 127, b = 127;
System.out.println(a == b);   // true（缓存中是同一个对象）
Integer c = 128, d = 128;
System.out.println(c == d);   // false（超出缓存范围，各自新建对象）
```

| 包装类 | 缓存范围 |
|--------|---------|
| Byte | -128 ~ 127（全部） |
| Short / Long / Integer | -128 ~ 127 |
| Character | 0 ~ 127（ASCII） |
| Boolean | true / false 均静态常量 |
| Float / Double | 无缓存 |

Integer 上限可用 `-Djava.lang.Integer.IntegerCache.high=500` 调高（仅能调高不能调低）。

> **注意**：包装类比较**值**一律用 `equals()`，绝不要用 `==`——`==` 比较的是引用，只有在缓存范围内才"碰巧"相等。这也是为什么用 `Integer` 做锁对象有隐患（缓存会导致意料之外的锁共享，见第 17 篇）。

## 8.4 常用方法

```java
Integer.parseInt("100");            // "100" → int 100
Integer.valueOf(100);               // 装箱（含缓存）
Integer.toString(100);              // "100"
Integer.toBinaryString(6);          // "110"
Character.isDigit('5');             // true
Boolean.parseBoolean("true");       // 仅 "true"（忽略大小写）为 true
```

> **陷阱**：`parseInt` / `valueOf` 传入非数字字符串会抛 `NumberFormatException`；`Boolean.parseBoolean` 对任何非 "true" 的字符串（包括 "TRUE" 之外的乱码）都返回 `false`，不会报错。

## 8.5 包装类与字符串转换

```java
// 包装类 → String
Integer.toString(100);   // "100"
String.valueOf(100);     // "100"
// String → 包装类
Integer.valueOf("100");  // 返回 Integer 对象
Integer.parseInt("100"); // 返回 int 基本类型
```

# 9. Collections 工具类

`Collections` 是操作集合的工具类，方法全是静态的。

| 方法 | 说明 |
|------|------|
| `addAll(Collection c, T... e)` | 批量添加（内部用可变参数） |
| `shuffle(List)` | 打乱顺序（洗牌） |
| `sort(List)` / `sort(List, Comparator)` | 自然排序 / 指定比较器排序 |
| `binarySearch(List, key)` | 二分查找（要求列表**已排序**） |
| `max(Collection)` / `min(Collection)` | 最大 / 最小 |
| `swap(List, i, j)` | 交换指定索引处的元素 |
| `copy(dest, src)` | 复制（要求 dest 的 size 不小于 src） |
| `fill(List, obj)` | 用指定对象替换所有元素 |
| `reverse(List)` | 反转列表顺序 |
| `replaceAll(List, old, new)` | 把所有 old 替换为 new |
| `frequency(Collection, obj)` | 统计某元素出现次数 |
| `synchronizedList(List)` | 包装成线程安全的 List |
| `unmodifiableList(List)` | 包装成只读 List（不可变视图） |
| `emptyList()` / `singletonList(obj)` | 返回空的 / 单元素的不可变列表 |

```java
List<Integer> list = new ArrayList<>(Arrays.asList(3, 1, 4, 1, 5));
Collections.sort(list);                 // 自然升序
System.out.println(list);               // [1, 1, 3, 4, 5]
Collections.shuffle(list);              // 随机打乱
System.out.println(list);               // 例如 [4, 1, 5, 3, 1]
System.out.println(Collections.max(list));   // 5
```

# 10. 增强 for 循环

```java
for (元素类型 变量名 : 数组或 Iterable 实现类) {
    // 操作变量名
}
```

它是 JDK5 引入的语法糖，**底层就是迭代器**（遍历数组时则编译成普通下标 for）。两点要注意：

- 遍历中给遍历变量重新赋值，不影响原集合/数组的元素；
- 遍历中**增删集合**会抛 `ConcurrentModificationException`（见第 13 节），因为改的是集合、绕过了迭代器。

# 11. 可变参数（JDK5+）

```java
public static int sum(int... nums) {
    int total = 0;
    for (int n : nums) total += n;   // 方法内部把 nums 当作数组处理
    return total;
}

sum();            // 0
sum(1, 2, 3);     // 6
sum(new int[]{1, 2, 3});   // 6，也可以直接传数组
```

- 本质：编译后就是一个**数组**参数；
- 规则：形参列表中**最多一个**可变参数，且必须**放在最后**；
- 它还是 `Arrays.asList(...)` 这类 API 的实现基础（注意基本类型数组会被当成"一个元素"，也是一处经典坑）。

# 12. 不可变集合

## 12.1 Arrays.asList 的坑（固定长度集合）

`Arrays.asList(...)` 返回的**不是 `ArrayList`**，而是 `Arrays` 内部的一个私有类，其底层直接引用传入的数组，**长度固定**：

```java
List<String> list = Arrays.asList("a", "b", "c");
list.set(0, "x");            // ✓ 允许，会修改背后的数组
list.add("d");               // ✗ UnsupportedOperationException
list.remove("a");            // ✗ UnsupportedOperationException
```

> **陷阱**：因为它是原数组的"视图"，`set` 会**双向影响原数组**；而对它做增删会抛 `UnsupportedOperationException`（不是 `ConcurrentModificationException`）。要真正可增删，必须再包一层：`new ArrayList<>(Arrays.asList(...))`。

## 12.2 List.of / Set.of / Map.of（JDK9+）

JDK9 起用静态工厂创建**真正不可变**的集合：

```java
List.of(1, 2, 3);
Set.of("a", "b");
Map.of("k1", 1, "k2", 2);         // 最多 20 个参数（10 个键值对）
Map.ofEntries(Map.entry("a", 1)); // 键值对较多时用 ofEntries
Map.copyOf(map);                  // JDK10，创建指定 Map 的不可变副本
```

这些集合**不允许 `null` 元素**（传入 null 直接抛 NPE），且任何修改操作都会抛 `UnsupportedOperationException`。

几种"不可变"方式对比：

| 方式 | 是否真不可变 | `set` 能否改 | 允许 `null` | 与原数据关系 |
|------|-------------|-------------|-------------|-------------|
| `Arrays.asList(...)` | 否（仅长度固定） | **能**（会改到原数组） | 允许 | 共享同一数组，双向影响 |
| `List.of(...)` | 是 | 不能 | **不允许** | 无，全新集合 |
| `Collections.unmodifiableList(list)` | 只读视图 | 不能 | 取决于原列表 | 是视图，原列表变它也变 |

**选择建议**：需要一个不可变的常量集合就用 `List.of(...)`（JDK9+）；只想把已有集合"只读暴露"出去用 `Collections.unmodifiableList(...)`；用 `Arrays.asList(...)` 时务必记住它长度固定、且与数组共享数据。

# 13. 迭代器与并发修改异常（fail-fast）

集合内部维护一个 `modCount`（修改计数器），每次结构性修改（增删元素）都会 `modCount++`。迭代器在创建时把当前值记到 `expectedModCount`，此后每次 `next()` / `remove()` 都会校验二者是否一致，不一致就抛 `ConcurrentModificationException`。以 `ArrayList` 的迭代器为例：

```java
// ArrayList.Itr 的简化版核心逻辑
private class Itr implements Iterator<E> {
    int cursor;                          // 下一个要返回的元素下标
    int lastRet = -1;                    // 上一个返回的元素下标
    int expectedModCount = modCount;     // 创建迭代器时"快照"当前修改次数

    public E next() {
        checkForComodification();        // 每次取元素前先校验
        // ... 返回 elementData[cursor] 并 cursor++
    }

    final void checkForComodification() {
        if (modCount != expectedModCount) // 集合被外部改动过 => 快速失败
            throw new ConcurrentModificationException();
    }

    public void remove() {
        ArrayList.this.remove(lastRet);  // 真正删除
        expectedModCount = modCount;     // 关键：把记录同步过来，避免误报
    }
}
```

由此得出两条结论：

- **用迭代器自身的 `remove()`** → 它同步更新了 `expectedModCount`，安全；
- **用集合的方法增删**（`list.add/remove`）→ 只改了 `modCount`，迭代器下次 `next()` 检测到不一致 → 抛 `ConcurrentModificationException`。

三种"边遍历边删"的写法：

```java
// ✗ 抛 ConcurrentModificationException：增强 for 底层是迭代器，list.remove 绕过迭代器
for (String s : list) {
    if (s.equals("b")) list.remove(s);
}
// ✓ 用迭代器自己的 remove()
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().equals("b")) it.remove();
}
// ✓ JDK8 最简洁：removeIf
list.removeIf(s -> s.equals("b"));
```

`ConcurrentModificationException` 属于集合的 **fail-fast（快速失败）**机制，目的是"一旦发现迭代过程中集合被外部改动就立刻报错"，而不是悄悄给出错误结果。要特别注意：**它不能 100% 检测多线程并发修改**——它只是尽力而为的检测，真并发下仍可能出现漏报。多线程安全遍历应使用 `CopyOnWriteArrayList`（快照读、不抛异常）或 `ConcurrentHashMap`（弱一致迭代器），这类**不抛异常**的机制称为 **fail-safe**。

# 14. 小结

| 主题 | 一句话要点 |
|------|-----------|
| 框架结构 | `Collection` 与 `Map` 是**两棵独立的树**；`Collection` 继承 `Iterable`（可 for-each），`Map` 不能直接遍历 |
| 泛型 | 编译期类型安全、运行期擦除；`? extends` 读、`? super` 写（**PECS**）；擦除导致不能 `instanceof` / 不能 `new T[]` |
| ArrayList | 动态数组，默认容量 10、**1.5 倍扩容**（`Arrays.copyOf`），按索引查 O(1)、中间增删 O(n)，**默认首选** |
| LinkedList | 双向链表，按索引查 O(n)、已知位置增删 O(1)，兼作 Deque；当队列/栈优先用 `ArrayDeque` |
| HashSet / HashMap | 哈希表 = 数组 + 链表 + 红黑树；靠 `hashCode` + `equals` 契约；**树化 8 / 退化 6**；负载因子 0.75、容量 2 的幂、翻倍扩容；JDK8 头插改尾插避免并发成环 |
| TreeSet / TreeMap | 红黑树，O(log n)；`Comparable`（自然排序）或 `Comparator`（定制排序）；去重按比较结果 = 0 |
| LinkedHashSet / LinkedHashMap | 哈希表 + 双向链表维持顺序；`accessOrder=true` + 重写 `removeEldestEntry` 可实现 **LRU** |
| 并发 Map | `Hashtable` 已淘汰（全表锁）；`ConcurrentHashMap` 从 JDK7 分段锁演进到 JDK8 的 CAS + synchronized 锁单桶 |
| 迭代器 | 遍历中增删集合会抛 `ConcurrentModificationException`（`modCount` 与 `expectedModCount` 不一致）；用 `it.remove()` 或 `removeIf` |
| 工具类与不可变 | `Collections` 提供 sort / shuffle / unmodifiable 等静态方法；`Arrays.asList` 是**固定长度**视图（`set` 会改原数组），`List.of` 才是真不可变 |

下一篇（第 10 篇）进入面向对象进阶，看接口、多态与内部类如何组织复杂系统；`Stream` 与方法引用的进阶用法见第 13 篇，锁的底层原理见第 30 篇。
