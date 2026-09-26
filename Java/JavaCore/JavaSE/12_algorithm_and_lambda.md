# 1.常见算法

## 1.1 冒泡排序（Bubble Sort）

```
基本思想：相邻元素两两比较，大的往后放。每一轮将当前最大的元素"冒"到最后。

时间复杂度：最好 O(n)，最坏 O(n²)，平均 O(n²)
空间复杂度：O(1)（原地排序）
稳定性：稳定

算法步骤：
  1. 比较相邻元素，如果前一个大于后一个则交换
  2. 每轮结束后最大的元素沉到末尾
  3. 重复 n-1 轮（或提前终止：若某轮无交换则已有序）

示例：
  int[] arr = {5, 2, 8, 3, 1};
  // 第一轮: {2, 5, 3, 1, 8}
  // 第二轮: {2, 3, 1, 5, 8}
  // 第三轮: {2, 1, 3, 5, 8}
  // 第四轮: {1, 2, 3, 5, 8}
```

## 1.2 选择排序（Selection Sort）

```
基本思想：每一轮从未排序区域选出最小（或最大）的元素，放到已排序区域的末尾。

时间复杂度：O(n²)（最好、最坏、平均都一样）
空间复杂度：O(1)
稳定性：不稳定（因为交换操作可能打乱相同元素的相对顺序）

算法步骤：
  1. 从 i=0 开始，找到 [i, n-1] 中最小元素的索引 minIndex
  2. 交换 arr[i] 与 arr[minIndex]
  3. i++，重复直到 i == n-1
```

## 1.3 插入排序（Insertion Sort）

```
基本思想：将数组分为已排序和未排序两部分，每次从未排序区取一个元素，
找到它在已排序区的合适位置并插入。

时间复杂度：最好 O(n)，最坏 O(n²)，平均 O(n²)
空间复杂度：O(1)
稳定性：稳定

算法步骤：
  1. 从索引 1 开始，将当前元素视为待插入元素
  2. 在已排序区 [0, i-1] 中从后向前找插入位置
  3. 边找边将腾位置（元素后移）
  4. 找到合适位置后插入

适用于：小规模数据、基本有序的数据。
```

## 1.4 快速排序（Quick Sort）

```
基本思想：分治法。选择一个基准值（pivot），将数组分为"小于基准"和"大于基准"两部分，
然后递归排序两部分。

时间复杂度：最好 O(n log n)，最坏 O(n²)（每次基准选到极值），平均 O(n log n)
空间复杂度：O(log n)（递归栈开销）
稳定性：不稳定

算法步骤：
  1. 选择基准值（如最左边元素）
  2. 从右往左找到第一个比基准小的，从左往右找到第一个比基准大的，交换
  3. 重复步骤 2 直到左右指针相遇
  4. 将基准值放到相遇位置
  5. 对左右两段递归执行
```

## 1.5 二分查找（Binary Search）

```
前提：数组必须是有序的（升序或降序）。

基本思想：每次取中间元素与目标比较，根据大小关系缩小搜索范围一半。

时间复杂度：O(log n)

算法步骤：
  1. 定义左右边界 left = 0, right = arr.length - 1
  2. while (left <= right)：
     a. mid = (left + right) / 2
     b. 若 arr[mid] == target → 返回 mid
     c. 若 arr[mid] < target → left = mid + 1
     d. 若 arr[mid] > target → right = mid - 1
  3. 未找到返回 -1

Arrays 工具类中的 binarySearch：
  int index = Arrays.binarySearch(arr, target);
  找到返回索引，未找到返回 -(插入点 + 1)（插入点：第一个大于 target 的位置）
```

## 1.6 递归

```
递归：方法自己调用自己。需要满足两个条件：
  1. 递归终止条件（基准条件）：防止无限递归
  2. 递归公式：将大问题化为同类小问题

经典递归示例——斐波那契数列：
  F(0) = 0, F(1) = 1, F(n) = F(n-1) + F(n-2)  (n >= 2)

注意事项：
  - 递归深度过大可能导致栈溢出（StackOverflowError）
  - 递归可造成大量重复计算，可用记忆化搜索优化
  - 能用迭代解决的问题，优先考虑迭代（效率更高）

递归与迭代对比：
               递归                      迭代
  可读性       代码简洁、表达力强          代码较长
  性能         有函数调用开销，可能重复计算  通常更高
  空间         额外栈空间                  通常 O(1)
  适用场景     树的遍历、分治算法等          简单的重复计算
```

# 2.Arrays

```
Arrays 是 java.util 包下的数组工具类，提供了一系列静态方法。

2.1 public static String toString(<array>)     // 将数组转换为字符串
    例如：Arrays.toString(new int[]{1,2,3}) → "[1, 2, 3]"

2.2 public static int binarySearch(<array>, <element>)
    二分查找：升序有序，存在返回索引，不存在返回 (-插入点 - 1)

2.3 public static int[] copyOf(<source_array>, <new_array_length>)
    复制数组，并指定新长度。若新长度大于原长度，多出部分填充默认值。
    若新长度小于原长度，截取前 newLength 个元素。

2.4 public static int[] copyOfRange(<source_array>, <start_index>, <end_index>)
    复制数组指定范围 [start, end)，左闭右开。

2.5 public static void fill(<array>, <element>)
    用指定值填充数组的每个元素。

2.6 public static void sort(<array>)
    对数组进行升序排序（底层使用优化版快速排序 Dual-Pivot QuickSort）。
    基本类型：使用双轴快排。
    引用类型：使用 TimSort（归并排序改进版，稳定排序）。

2.7 public static void sort(<array>, <sort_rule>)
    使用指定 Comparator 规则排序。

2.8 public static boolean equals(<array1>, <array2>)
    比较两个数组内容是否相等（逐元素比较）。

2.9 public static <T> List<T> asList(T... a)
    将数组转换为 List（注意：返回的 List 是固定大小的，不支持 add/remove）。

2.10 public static void parallelSort(<array>)
    JDK8+，并行排序，适用于大数据量的排序。
```

# 3.Lambda（jdk8开始）

## 3.1格式:(params)->{<func_body>}//原型为new <interface_name>(){@Override <func_override>}

```
Lambda 表达式的格式：(params) -> { <func_body> }
它是对匿名内部类的一种简化写法，尤其适用于只包含一个方法的接口（函数式接口）。

本质：Lambda 表达式是函数式接口的实例（一个对象）。
生成方式：编译器通过 invokedynamic + LambdaMetafactory 在运行时动态生成实现类，
而不是像匿名内部类那样在编译期生成 .class 文件。因此 Lambda 更轻量，类加载更少。

示例对比：
  // 匿名内部类写法
  Runnable r1 = new Runnable() {
      @Override
      public void run() {
          System.out.println("Hello");
      }
  };

  // Lambda 写法
  Runnable r2 = () -> System.out.println("Hello");
```

## 3.2注意

```
3.2.1 Lambda表达式可以用来简化匿名内部类的书写
3.2.2 Lambda表达式只能简化函数式接口的匿名内部类的写法
3.2.3 函数式接口：有且仅有一个抽象方法的接口叫函数式接口，
    接口上方可以加 @FunctionalInterface 注解（非函数式接口加这个注解编译报错）
3.2.4 Lambda 表达式中的 this 指向的是包含 Lambda 表达式的类的实例，
    而不是 Lambda 表达式生成的内部对象（这与匿名内部类不同）
```

## 3.3Lambda表达式的省略规则（总的来说：可推导，可省略）

```
3.3.1 参数类型可以省略不写（因为可以从接口方法中推导）
3.3.2 如果只有一个参数，参数类型可以省略，同时 () 也可以省略
3.3.3 如果 Lambda 表达式的方法体只有一行，大括号 {}、分号 ; 和 return 关键字可以同时省略（必须同时省略）
3.3.4 如果有多个参数，参数类型要么全写，要么全不写

示例：
  // 完整写法
  (int a, int b) -> { return a + b; }
  // 省略参数类型
  (a, b) -> { return a + b; }
  // 省略大括号、分号和 return（单行表达式）
  (a, b) -> a + b
  // 单参数省略括号
  x -> x * x
```

## 3.4 常用内置函数式接口（java.util.function 包）

```
3.4.1 Consumer<T>        消费型接口
    void accept(T t)     接收一个参数，无返回值
    常用于：遍历、打印等操作
    示例：list.forEach(s -> System.out.println(s))

3.4.2 Predicate<T>       断言型接口
    boolean test(T t)    接收一个参数，返回 boolean
    常用于：过滤、条件判断
    示例：list.removeIf(s -> s.length() > 5)

3.4.3 Function<T, R>     函数型接口
    R apply(T t)         接收 T 类型参数，返回 R 类型
    常用于：类型转换、数据映射
    示例：map(str -> str.length())

3.4.4 Supplier<T>        供给型接口
    T get()              无参数，返回一个值
    常用于：工厂模式、懒加载
    示例：Supplier<Double> random = () -> Math.random();

3.4.5 UnaryOperator<T>   一元操作（Function 的子接口）
    T apply(T t)         输入和输出类型相同
    常用于：同一类型的数据变换
    示例：list.replaceAll(s -> s.toUpperCase())

3.4.6 BinaryOperator<T>  二元操作（BiFunction 的子接口）
    T apply(T t1, T t2)  接收两个同类型参数，返回同类型
    常用于：Reducer 操作
    示例：BinaryOperator<Integer> add = (a, b) -> a + b;

3.4.7 BiFunction<T, U, R> 双参数函数
    R apply(T t, U u)    接收两个不同类型参数，返回 R
    常用于：需要两个输入的计算

3.4.8 BiConsumer<T, U>   双参数消费
    void accept(T t, U u) 接收两个参数，无返回值
    常用于：Map 遍历 map.forEach((k, v) -> ...)
```

## 3.5 Lambda 与 Stream 结合示例

```
List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "David");

// 过滤 + 映射 + 收集
List<String> result = names.stream()
    .filter(name -> name.length() > 3)     // Predicate：过滤长度大于3的
    .map(name -> name.toUpperCase())        // Function：转为大写
    .collect(Collectors.toList());          // 收集为 List

// 简化写法（方法引用）
List<String> result2 = names.stream()
    .filter(name -> name.length() > 3)
    .map(String::toUpperCase)
    .toList();  // JDK16+ 简化为 .toList()

System.out.println(result);  // [ALICE, CHARLIE, DAVID]
```

## 3.6 方法引用（:: 操作符）

```
方法引用是 Lambda 的进一步简化形式，当 Lambda 表达式只调用了一个已有方法时使用。

四种形式：
  3.6.1 静态方法引用：类名::静态方法
        (s) -> Integer.parseInt(s)  →  Integer::parseInt

  3.6.2 实例方法引用（特定对象）：对象名::实例方法
        () -> obj.getName()  →  obj::getName

  3.6.3 实例方法引用（任意对象）：类名::实例方法
        (s) -> s.toUpperCase()  →  String::toUpperCase
        （第一个参数作为调用对象，其余参数作为方法参数）

  3.6.4 构造方法引用：类名::new
        () -> new Student()    →  Student::new
        (name) -> new Student(name)  →  Student::new
```
