# 1.集合基础

```
1.1 集合与数组的对比

                    数组 (Array)                集合 (Collection)
  长度            固定，创建后不可变               动态可变，自动扩容
  存储类型         基本类型 + 引用类型              仅引用类型（JDK5+ 自动装箱）
  泛型支持         不支持                          支持
  可存储不同类型   是（Object[]）                  不推荐（Collection<Object> 可）
  专有方法         仅 .length 属性                 丰富的增删改查方法

1.2 集合体系结构
  Collection 接口（单列集合的顶层接口）
  ├── List 接口（有序、可重复、有索引）
  │   ├── ArrayList（数组实现，查询快增删慢）
  │   ├── LinkedList（双向链表实现，增删快查询慢）
  │   └── Vector（数组实现，线程安全，已过时）
  └── Set 接口（无序、不可重复、无索引）
      ├── HashSet（哈希表，无序）
      │   └── LinkedHashSet（哈希表+链表，有序）
      └── TreeSet（红黑树，可排序）
```

# 2.ArrayList

```
ArrayList 是基于动态数组实现的 List 接口的实现类。
特点：查询快（O(1)），增删慢（O(n)），线程不安全。
默认初始容量为 10，JDK7+ 使用延迟初始化（懒加载），即 new ArrayList<>() 时分配空数组，
第一次 add 时才分配容量为 10 的数组。
```

## 2.1构造方法:左边一定要写泛型，右边可以省略

```
2.1.1 ArrayList()                           // 默认构造，JDK8+ 延迟初始化
2.1.2 ArrayList(int initialCapacity)         // 指定初始容量
2.1.3 ArrayList(Collection<? extends E> c)   // 使用已有集合构造
```

## 2.2成员方法

```
2.2.1 boolean add(E e)                      // 尾部追加元素，始终返回 true
2.2.2 void add(int index, E e)              // 在指定索引处插入元素（原有元素后移）
2.2.3 boolean remove(E e)                  // 删除第一个匹配的元素，有返回 true，无返回 false
2.2.4 E remove(int index)                   // 删除并返回指定索引的元素
2.2.5 E set(int index, E e)                // 修改指定索引下的元素，返回原来的元素
2.2.6 E get(int index)                      // 获取指定索引的元素
2.2.7 int size()                            // 返回元素个数（不是容量）
2.2.8 boolean contains(Object o)             // 判断是否包含指定元素（内部调用 equals）
2.2.9 int indexOf(Object o)                 // 返回第一次出现的索引，未找到返回 -1
2.2.10 int lastIndexOf(Object o)             // 返回最后一次出现的索引
2.2.11 void clear()                          // 清空所有元素
2.2.12 boolean isEmpty()                     // 判断是否为空
2.2.13 Object[] toArray()                    // 转换为 Object 数组
2.2.14 <T> T[] toArray(T[] a)               // 转换为指定类型的数组
2.2.15 boolean addAll(Collection<? extends E> c)     // 添加集合中所有元素
2.2.16 boolean removeAll(Collection<?> c)            // 删除与集合 c 的交集元素
2.2.17 boolean retainAll(Collection<?> c)            // 保留与集合 c 的交集元素
2.2.18 void forEach(Consumer<? super E> action)      // JDK8+，Lambda 遍历
2.2.19 boolean removeIf(Predicate<? super E> filter) // JDK8+，条件删除
2.2.20 void replaceAll(UnaryOperator<E> operator)    // JDK8+，替换所有元素
2.2.21 void sort(Comparator<? super E> c)            // JDK8+，按比较器排序
2.2.22 List<E> subList(int fromIndex, int toIndex)   // 获取子列表视图
```

## 2.3ArrayList底层原理

```
2.3.1 数据结构
  ArrayList 底层基于 Object[] elementData 数组存储元素。
  JDK7：new ArrayList<>() 时直接创建长度为 10 的数组。
  JDK8：new ArrayList<>() 时创建空数组 {}，第一次 add() 时才分配容量。

2.3.2 扩容机制
  触发条件：当 add 时数组已满（size == elementData.length）。
  扩容策略：
    新容量 = 旧容量 + (旧容量 >> 1)  即 旧容量的 1.5 倍
    示例：10 → 15 → 22 → 33 → ...
  扩容实现：
    使用 Arrays.copyOf(elementData, newCapacity) 复制到新数组。
    扩容操作的代价较高，涉及数组复制。
  
2.3.3 添加元素流程
  1. 确保容量足够（容量不足则扩容）
  2. 将新元素放入 elementData[size]
  3. size++

2.3.4 插入元素流程
  1. 确保容量足够
  2. 从插入位置开始，所有元素向后移动一位
  3. 将新元素放入空出的位置
  4. size++

2.3.5 删除元素流程
  1. 获取被删除位置的元素（用于返回）
  2. 从删除位置开始，所有后续元素向前移动一位
  3. 将最后一个位置置 null（避免内存泄漏）
  4. size--
```

## 2.4遍历方式

```
2.4.1 普通 for 循环
  for (int i = 0; i < list.size(); i++) {
      System.out.println(list.get(i));
  }

2.4.2 增强 for 循环（foreach）
  for (String s : list) {
      System.out.println(s);
  }
  内部本质是迭代器。遍历过程中不能修改集合（会抛 ConcurrentModificationException）。

2.4.3 迭代器 Iterator
  Iterator<String> it = list.iterator();
  while (it.hasNext()) {
      System.out.println(it.next());
  }
  遍历中可以使用 it.remove() 删除元素（不能用 list.remove()）。

2.4.4 Lambda 表达式（JDK8+）
  list.forEach(s -> System.out.println(s));
  或方法引用：list.forEach(System.out::println);

2.4.5 ListIterator（双向迭代器）
  ListIterator<String> lit = list.listIterator();
  while (lit.hasNext()) {
      System.out.println(lit.next());
  }
  // 支持向前遍历：while (lit.hasPrevious()) {...}
```

## 2.5注意事项

```
2.5.1 线程安全问题
  ArrayList 不是线程安全的。多线程环境下：
    - 使用 Collections.synchronizedList(new ArrayList<>()) 包装
    - 或使用 CopyOnWriteArrayList（读写分离，写时复制）

2.5.2 删除陷阱
  普通 for 循环中删除元素需要注意索引变化：
    错误示例：
    for (int i = 0; i < list.size(); i++) {
        if (条件) list.remove(i); // 删除后后续元素前移，i++ 会跳过一个元素
    }
    正确做法：
    for (int i = list.size() - 1; i >= 0; i--) {
        if (条件) list.remove(i);
    }
    或使用迭代器的 remove() 方法。

2.5.3 subList 注意事项
  subList 返回的是视图（View），对 subList 的修改会反映到原 List。
  原 List 的删除/插入等结构性修改会导致子列表抛出 ConcurrentModificationException。
```

# 3.LinkedList

```
LinkedList 是基于双向链表实现。查询慢（O(n)），增删快（O(1)）。

3.1 特有方法
  3.1.1 public void addFirst(E e)       // 在开头插入元素
  3.1.2 public void addLast(E e)        // 在结尾追加元素（同 add）
  3.1.3 public E getFirst()             // 获取第一个元素
  3.1.4 public E getLast()              // 获取最后一个元素
  3.1.5 public E removeFirst()          // 删除并返回第一个元素
  3.1.6 public E removeLast()           // 删除并返回最后一个元素
  3.1.7 public E pop()                  // 模拟栈，弹出第一个元素（等价 removeFirst）
  3.1.8 public void push(E e)           // 模拟栈，压入元素（等价 addFirst）

3.2 底层原理
  LinkedList 基于双向链表，内部类 Node 结构：
    private static class Node<E> {
        E item;
        Node<E> next;   // 后继节点
        Node<E> prev;   // 前驱节点
    }
  维护两个指针：first（头节点）和 last（尾节点）。

3.3 ArrayList vs LinkedList 使用场景
  ArrayList 适合：频繁随机访问（get/set）、尾部追加
  LinkedList 适合：频繁头尾增删、不需要随机访问的队列/栈场景
```

# 4.包装类

```
4.1 为什么需要包装类
  - 集合不能存储基本数据类型，需要包装类
  - 包装类提供了丰富的转换方法和工具方法
  - 包装类支持 null 值，表达"不存在"的语义

4.2 基本类型 ↔ 包装类对照表

  基本类型    包装类       父类
  byte       Byte        Number
  short      Short       Number
  int        Integer     Number
  long       Long        Number
  float      Float       Number
  double     Double      Number
  char       Character   Object
  boolean    Boolean     Object

4.3 装箱与拆箱

  装箱（Boxing）：基本类型 → 包装类
    Integer i = Integer.valueOf(100);  // 手动装箱
    Integer i = 100;                   // 自动装箱（JDK5+）

  拆箱（Unboxing）：包装类 → 基本类型
    int n = i.intValue();  // 手动拆箱
    int n = i;             // 自动拆箱（JDK5+）

4.4 包装类缓存机制（享元模式）

  Integer 缓存：默认缓存 -128 ~ 127 范围的 Integer 对象。
  通过 valueOf() 获取时会优先从缓存中返回，new Integer() 不会使用缓存。

  示例：
    Integer a = 127;  // 自动装箱 → Integer.valueOf(127)
    Integer b = 127;
    System.out.println(a == b);   // true（缓存中同一对象）

    Integer c = 128;
    Integer d = 128;
    System.out.println(c == d);   // false（超出缓存范围，各自新建对象）

  Integer 缓存可通过 JVM 参数调整上限：
    -Djava.lang.Integer.IntegerCache.high=500

  其他包装类的缓存：
    Byte：-128 ~ 127（全部缓存）
    Short：-128 ~ 127
    Long：-128 ~ 127
    Character：0 ~ 127（ASCII 范围）
    Boolean：true 和 false 均静态常量
    Float/Double：无缓存

4.5 包装类的常用方法

  Integer 常用方法：
    parseInt(String s)          // String → int
    parseInt(String s, int radix) // 按进制 String → int
    valueOf(int i)              // int → Integer（含缓存）
    valueOf(String s)           // String → Integer
    toBinaryString(int i)       // 转二进制字符串
    toOctalString(int i)        // 转八进制字符串
    toHexString(int i)          // 转十六进制字符串
    MAX_VALUE / MIN_VALUE       // 常量：最大值 / 最小值
    compare(int x, int y)       // JDK7+，比较两个 int 值

  Character 常用方法：
    isDigit(char ch)            // 是否数字
    isLetter(char ch)           // 是否字母
    isUpperCase(char ch)        // 是否大写
    isLowerCase(char ch)        // 是否小写
    isWhitespace(char ch)       // 是否空白字符
    toUpperCase(char ch)        // 转大写
    toLowerCase(char ch)        // 转小写

  Boolean 常用方法：
    parseBoolean(String s)      // String → boolean（仅 "true"（忽略大小写）返回 true）
    logicalAnd(boolean a, boolean b)  // 逻辑与
    logicalOr(boolean a, boolean b)   // 逻辑或
    logicalXor(boolean a, boolean b)  // 逻辑异或

4.6 包装类与字符串的转换

  包装类 → String：
    Integer.toString(100)          // "100"
    Integer.toHexString(255)       // "ff"
    String.valueOf(100)            // "100"

  String → 包装类：
    Integer.valueOf("100")         // Integer{100}
    Integer.parseInt("100")        // 100（返回 int 基本类型）
    new Integer("100")             // Integer{100}（已过时，JDK9+ @Deprecated）

  注意：如果字符串格式不正确，抛出 NumberFormatException。
```

# 5.增强for循环（foreach）

```
5.1 格式
  for (元素类型 变量名 : 数组或Iterable实现类) {
      // 变量名代表当前元素
  }

5.2 特点
  - JDK5 引入，简化遍历语法
  - 底层本质是迭代器（Iterator）
  - 遍历过程中不能修改集合（除非使用迭代器的 remove）
  - 修改遍历变量不影响原集合/数组

5.3 适用范围
  - 所有实现了 Iterable 接口的类（所有 Collection 子类）
  - 数组类型

示例：
  for (int num : list) {
      System.out.println(num);
  }
```
