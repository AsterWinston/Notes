本篇讲两块内容：一是**常用算法**（排序、查找、递归）——它们是写代码的底层功力，也是面试的常客；二是 **Lambda 表达式**——JDK8 之后最重要的语法特性，是从"命令式"跨到"函数式"的入口。Lambda 与 `Stream` 的进阶用法在第 13 篇展开，本篇只讲基础并做交叉引用。

# 1. 排序算法

## 1.1 排序算法总览

先说一个贯穿全章的指标——**稳定性**：排序前后，**值相等的元素相对顺序保持不变**就是稳定排序。它在"多关键字排序"时很重要，比如先按成绩排、成绩相同的再按学号排。

| 算法 | 最好 | 平均 | 最坏 | 空间 | 稳定性 | 什么时候用 |
|------|------|------|------|------|--------|-----------|
| 冒泡排序 | O(n) | O(n²) | O(n²) | O(1) | **稳定** | 教学演示，数据量极小 |
| 选择排序 | O(n²) | O(n²) | O(n²) | O(1) | 不稳定 | 要求交换次数尽量少（最多 n-1 次） |
| 插入排序 | O(n) | O(n²) | O(n²) | O(1) | **稳定** | 小规模、基本有序的数据 |
| 快速排序 | O(n log n) | O(n log n) | O(n²) | O(log n) | 不稳定 | 通用最快，大数据量的首选 |

## 1.2 冒泡排序（Bubble Sort）

**思路**：相邻两两比较，大的往后放。每一轮都把当前未排序区里最大的元素"冒"到末尾。

**算法步骤**：

1. 比较相邻元素，前一个大于后一个就交换；
2. 每轮结束后，本轮最大的元素沉到未排序区末尾；
3. 重复 n-1 轮；若某一轮一次交换都没发生，说明已经有序，可提前结束。

```java
public static void bubbleSort(int[] arr) {
    for (int i = 0; i < arr.length - 1; i++) {          // 一共最多 n-1 轮
        boolean swapped = false;                        // 本轮是否发生过交换
        for (int j = 0; j < arr.length - 1 - i; j++) {  // 每轮把最大的"冒"到末尾
            if (arr[j] > arr[j + 1]) {                  // 前一个比后一个大，交换
                int tmp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = tmp;
                swapped = true;
            }
        }
        if (!swapped) break;                            // 一轮无交换 => 已有序，提前结束
    }
}

public static void main(String[] args) {
    int[] arr = {5, 2, 8, 3, 1};
    bubbleSort(arr);
    System.out.println(Arrays.toString(arr));   // [1, 2, 3, 5, 8]
}
```

`swapped` 标志是"提前终止"优化：最好情况下（数组已有序）只需跑一轮，退化为 O(n)。

## 1.3 选择排序（Selection Sort）

**思路**：每一轮从未排序区里选出最小（或最大）的元素，放到已排序区的末尾。

**算法步骤**：

1. 从 `i = 0` 开始，找到 `[i, n-1]` 中最小元素的下标 `minIndex`；
2. 交换 `arr[i]` 与 `arr[minIndex]`；
3. `i++`，重复直到 `i == n-1`。

```java
public static void selectionSort(int[] arr) {
    for (int i = 0; i < arr.length - 1; i++) {
        int minIndex = i;                               // 先假设 i 处是最小值
        for (int j = i + 1; j < arr.length; j++) {
            if (arr[j] < arr[minIndex]) minIndex = j;   // 找到 [i, n-1] 真正的最小值
        }
        if (minIndex != i) {                            // 交换到已排序区末尾
            int tmp = arr[i];
            arr[i] = arr[minIndex];
            arr[minIndex] = tmp;
        }
    }
}
```

选择排序**什么数据都是 O(n²)**（比较次数固定），但交换次数最多只有 n-1 次，所以适合"交换成本远高于比较成本"的场景。它是**不稳定**的：交换可能把相等元素的前后关系打乱。

## 1.4 插入排序（Insertion Sort）

**思路**：把数组看成"已排序区 + 未排序区"，每次从未排序区取一个元素，在已排序区里从后往前找位置插入（边找边腾位置）。

**算法步骤**：

1. 从索引 1 开始，把当前元素视为待插入元素；
2. 在已排序区 `[0, i-1]` 中从后向前找插入位置；
3. 比待插入元素大的元素整体后移一位；
4. 找到位置后插入。

```java
public static void insertionSort(int[] arr) {
    for (int i = 1; i < arr.length; i++) {              // 从第 2 个元素开始
        int cur = arr[i];                               // 待插入的元素
        int j = i - 1;
        while (j >= 0 && arr[j] > cur) {                // 比它大的整体后移
            arr[j + 1] = arr[j];
            j--;
        }
        arr[j + 1] = cur;                               // 插入到空出来的位置
    }
}
```

插入排序在**数据基本有序**时极快（每个元素只需比较一两次，接近 O(n)），所以实际工程里的混合排序（如 TimSort）在数据量小时会退回到插入排序。

## 1.5 快速排序（Quick Sort）

**思路**：分治法。选一个**基准值 pivot**，把数组划分成"小于基准"和"大于基准"两部分，再对两部分递归排序。

**算法步骤**：

1. 选择基准值；
2. 用双指针从两端向中间扫描，把小于基准的放到左边、大于基准的放到右边（这个动作叫 **partition 划分**）；
3. 基准值最终落在两指针相遇处，此时它左边的都不大于它、右边的都不小于它；
4. 对左右两段递归执行。

```java
public static void quickSort(int[] arr, int left, int right) {
    if (left >= right) return;                          // 只剩 0 或 1 个元素，天然有序
    int pivotIndex = partition(arr, left, right);       // 一趟划分，返回基准的最终位置
    quickSort(arr, left, pivotIndex - 1);               // 递归排左段
    quickSort(arr, pivotIndex + 1, right);              // 递归排右段
}

// 挖坑法：以最左边元素为基准
private static int partition(int[] arr, int left, int right) {
    int pivot = arr[left];                              // 基准值先"挖"出来，left 处留下一个坑
    int i = left, j = right;
    while (i < j) {
        while (i < j && arr[j] >= pivot) j--;           // 右指针左移，找比基准小的
        if (i < j) arr[i++] = arr[j];                   // 填到左坑，j 处变成新坑
        while (i < j && arr[i] <= pivot) i++;           // 左指针右移，找比基准大的
        if (i < j) arr[j--] = arr[i];                   // 填到右坑，i 处变成新坑
    }
    arr[i] = pivot;                                     // 基准落位
    return i;                                           // 返回基准最终下标
}

public static void main(String[] args) {
    int[] arr = {5, 2, 8, 3, 1};
    quickSort(arr, 0, arr.length - 1);
    System.out.println(Arrays.toString(arr));   // [1, 2, 3, 5, 8]
}
```

**基准值怎么选很关键**：固定选最左（或最右）元素时，遇到**已经有序**或**全部相等**的输入，每次划分都会把区间分成"1 个和 n-1 个"，递归深度退化成 O(n)，时间复杂度恶化到 **O(n²)**（同时也是栈溢出风险）。常见改进：

| 选法 | 说明 |
|------|------|
| 随机化 | 每轮随机选一个元素当基准，避免恶意/有序输入总是最坏 |
| 三数取中 | 取左、中、右三个数的中位数当基准，简单有效、无需随机数 |
| 双轴快排 | 选两个基准把数组分成三段，JDK 用于基本类型排序（见 1.6） |

快排的平均空间复杂度是 O(log n)（递归栈深度），最坏 O(n)。

## 1.6 为什么 Java 的 Arrays.sort 这么选

Java 的 `Arrays.sort` 对**不同数组类型**采用了不同的算法，这不是随意选择：

| 排序对象 | 使用的算法 | 原因 |
|----------|-----------|------|
| 基本类型数组（`int[]`、`double[]`…） | **双轴快速排序**（Dual-Pivot Quicksort） | 基本类型没有"稳定性"一说（两个 `1` 无法区分先后），快排最快、原地、常数小，所以直接用两基准把数组切三段的高效快排 |
| 对象数组（`T[]`） | **TimSort**（归并排序的改进版） | 对象相等时可能有其他区别，用户期望**稳定排序**；TimSort 稳定，且能识别数据中已有序的片段（run），接近有序时可接近 O(n) |

同样的道理，`Collections.sort(list)` 底层就是 `list.sort()` → 对象数组的 TimSort。这个知识点常被问到"为什么不用一种排序搞定"，答案就是**稳定性需求不同 + 数据特征不同**。

# 2. 查找算法

## 2.1 二分查找的前提

二分查找（Binary Search）的**唯一前提是数据必须有序**（升序或降序）。它的思路是：每次取中间元素与目标比较，根据大小关系把搜索范围缩小一半，时间复杂度 O(log n)。无序数组上做二分，结果没有任何保证。

## 2.2 手写二分与边界陷阱

```java
public static int binarySearch(int[] arr, int target) {
    int left = 0, right = arr.length - 1;           // 闭区间 [left, right]
    while (left <= right) {
        int mid = left + (right - left) / 2;        // 防溢出的中点写法
        if (arr[mid] == target) return mid;         // 命中
        if (arr[mid] < target) left = mid + 1;      // 目标在右半区
        else right = mid - 1;                       // 目标在左半区
    }
    return -1;                                      // 循环结束仍未找到
}
```

两个最容易踩的边界陷阱：

- **`mid` 的溢出**：`mid = (left + right) / 2` 在数组很大时，`left + right` 可能超过 `int` 的最大值变成负数，导致数组越界。正确写法是 `left + (right - left) / 2` 或 `(left + right) >>> 1`。这曾是 JDK 里的真实缺陷（`Arrays.binarySearch` 的经典 bug），JDK6 才修复。
- **区间开闭要一致**：`while (left <= right)` 配 `right = mid - 1`（闭区间 `[left, right]`），或 `while (left < right)` 配 `right = mid`（半开区间 `[left, right)`）。两套写法不能混，否则会死循环或漏查。

## 2.3 Arrays.binarySearch 的返回值语义

`Arrays.binarySearch(arr, key)` 的返回值有讲究：

- **找到**：返回某个等于 `key` 的下标（有重复元素时返回哪一个不确定）；
- **未找到**：返回 **`-(插入点) - 1`**。插入点是"第一个大于 `key` 的元素下标"，若所有元素都小于 `key` 则为数组长度。用 `-(返回值) - 1` 就能反推出插入点。

```java
int[] a = {1, 3, 5, 7, 9};
System.out.println(Arrays.binarySearch(a, 5));   //  2      找到，下标 2
System.out.println(Arrays.binarySearch(a, 4));   // -3      未找到，插入点 2 => -(2)-1
System.out.println(Arrays.binarySearch(a, 0));   // -1      插入点 0 => -(0)-1
System.out.println(Arrays.binarySearch(a, 10));  // -6      插入点 5 => -(5)-1
```

> **注意**：用 `Arrays.binarySearch` 或 `Collections.binarySearch` 前，数组/列表**必须已按同一比较规则排好序**；对对象数组排序用的 `Comparator`，查找时也要用同一个，否则结果不可预期。

# 3. 递归

## 3.1 递归的三要素

递归是"方法自己调用自己"。写出正确的递归，必须同时明确三件事：

1. **终止条件（base case）**：什么时候不再递归、直接返回，防止无限递归；
2. **递归公式**：如何把原问题拆成规模更小的**同类子问题**（如 `f(n) = n * f(n-1)`）；
3. **收敛方向**：每一步分解都必须让问题规模**朝终止条件靠近**，否则永远到不了 base case。

## 3.2 递归调用栈（栈帧图）

每进入一层递归，JVM 就会在虚拟机栈上压入一个**栈帧**，保存该层的参数、局部变量和返回地址；方法返回时再弹栈。以 `factorial(3)` 为例：

```text
        ┌──────────────────────────┐  ← 栈顶（最先返回）
        │ factorial(0)  return 1   │
        ├──────────────────────────┤
        │ factorial(1)  return 1*1 │
        ├──────────────────────────┤
        │ factorial(2)  return 2*1 │  下层结果回来后才能继续算
        ├──────────────────────────┤
        │ factorial(3)  return 3*2 │
        ├──────────────────────────┤
        │ main()                   │  ← 栈底
        └──────────────────────────┘

   压栈顺序：main → factorial(3) → factorial(2) → factorial(1) → factorial(0)
   弹栈顺序：factorial(0) → factorial(1) → factorial(2) → factorial(3) → main
```

这也解释了递归为什么耗内存：**递归深度 = 同时存在的栈帧数**，深度越大占用的栈空间越多。

## 3.3 经典示例

**阶乘**：

```java
public static long factorial(int n) {
    if (n <= 1) return 1;              // 终止条件
    return n * factorial(n - 1);       // 递归公式
}
// factorial(5) = 120
// 注意：long 在 n > 20 时就会溢出，需要 BigInteger
```

**斐波那契数列**（`F(0)=0, F(1)=1, F(n)=F(n-1)+F(n-2)`）：

```java
// 朴素递归：存在大量重复计算，时间复杂度 O(2^n)
public static long fib(int n) {
    if (n < 2) return n;               // 终止条件
    return fib(n - 1) + fib(n - 2);    // 递归公式
}

// 记忆化搜索：用数组缓存已算过的值，降到 O(n)
public static long fib(int n, long[] memo) {
    if (n < 2) return n;
    if (memo[n] != 0) return memo[n];              // 命中缓存，直接返回
    memo[n] = fib(n - 1, memo) + fib(n - 2, memo);
    return memo[n];
}
```

朴素斐波那契递归是"重复计算"的典型反面教材：算 `fib(5)` 会反复计算 `fib(3)`、`fib(2)`，把指数级的时间浪费掉；记忆化（把算过的结果存起来复用）是通用的优化手段。

**汉诺塔**（把 n 个盘子从 `from` 经 `by` 移到 `to`）：

```java
public static void hanoi(int n, char from, char by, char to) {
    if (n == 1) {                                   // 终止条件：只剩一个盘子，直接搬
        System.out.println(from + " -> " + to);
        return;
    }
    hanoi(n - 1, from, to, by);                     // 先把上面 n-1 个挪到中转柱
    System.out.println(from + " -> " + to);         // 把最大的盘子搬到目标柱
    hanoi(n - 1, by, from, to);                     // 再把 n-1 个从中转柱挪到目标柱
}

public static void main(String[] args) {
    hanoi(3, 'A', 'B', 'C');
    // 输出（共 2^3 - 1 = 7 步）：
    // A -> C
    // A -> B
    // C -> B
    // A -> C
    // B -> A
    // B -> C
    // A -> C
}
```

## 3.4 递归 vs 迭代、尾递归与栈溢出

**栈溢出**：每层递归都压栈，深度过大（JVM 默认栈大小有限）会抛 `StackOverflowError`。

```java
public static int sum(int n) {         // 递归求 1+2+...+n
    if (n == 1) return 1;
    return n + sum(n - 1);             // n 过大 => StackOverflowError
}

public static int sumIter(int n) {     // 迭代版：无栈溢出风险
    int s = 0;
    for (int i = 1; i <= n; i++) s += i;
    return s;
}
```

**尾递归**指递归调用是整个方法的**最后一步操作**（返回值直接就是递归调用的结果），理论上编译器可以把栈帧复用、优化成循环。但 **JVM 并不支持尾递归优化**，所以 Java 里的尾递归写法仍然会逐层压栈，照样可能栈溢出。

递归与迭代的取舍：

| 维度 | 递归 | 迭代 |
|------|------|------|
| 可读性 | 代码简洁、表达力强 | 代码较长 |
| 性能 | 有函数调用开销，可能重复计算 | 通常更高 |
| 空间 | 额外栈空间 O(深度) | 通常 O(1) |
| 风险 | 深度过大 → `StackOverflowError` | 无 |
| 适用场景 | 树/图的遍历、分治、汉诺塔 | 简单重复计算、深度很大的场景 |

**选择建议**：能用迭代解决的问题优先用迭代；当问题天然是递归结构（树遍历、分治、回溯）且深度可控时，递归更清晰。

# 4. Arrays 工具类

`Arrays` 是 `java.util` 包下的数组工具类，方法全是静态的。

| 方法 | 说明 |
|------|------|
| `toString(数组)` | 把数组转成可读字符串，如 `Arrays.toString(new int[]{1,2,3})` → `"[1, 2, 3]"` |
| `deepToString(数组)` | 打印多维数组（普通 `toString` 对多维数组只打印地址） |
| `binarySearch(数组, 元素)` | 二分查找。**要求升序有序**；找到返回索引，未找到返回 `-(插入点) - 1` |
| `copyOf(源数组, 新长度)` | 复制并指定新长度：变长则补默认值，变短则截断 |
| `copyOfRange(源数组, from, to)` | 复制 `[from, to)` 范围，左闭右开 |
| `fill(数组, 元素)` | 用指定值填充每个元素 |
| `sort(数组)` | 升序排序。**基本类型用双轴快排；引用类型用 TimSort**（稳定） |
| `sort(数组, Comparator)` | 按指定比较器排序（仅引用类型数组支持） |
| `equals(数组1, 数组2)` | 逐元素比较内容是否相等（`==` 比的是地址） |
| `asList(T... a)` | 数组转 `List`。**返回的是固定长度集合，不支持 add/remove**，见第 9 篇 12.1 |
| `stream(数组)` | 把数组转成 `Stream`，便于链式处理（见第 13 篇） |
| `parallelSort(数组)` | JDK8+ 并行排序，适合大数据量 |
| `setAll(数组, 函数)` | 用函数按索引生成并填充元素（JDK8+） |

# 5. Lambda 表达式（JDK8 起）

## 5.1 从匿名内部类到 Lambda

Lambda 是对**匿名内部类**的简化写法，尤其适用于只包含一个方法的接口（**函数式接口**）。

```java
// 匿名内部类写法
Runnable r1 = new Runnable() {
    @Override
    public void run() {
        System.out.println("Hello");
    }
};

// Lambda 写法：语义完全一致，但简洁得多
Runnable r2 = () -> System.out.println("Hello");
```

**本质**：Lambda 表达式是**函数式接口的一个实例（对象）**。它的生成方式是编译器通过 `invokedynamic` + `LambdaMetafactory` 在**运行时动态生成**实现类，而不是像匿名内部类那样在编译期生成一个 `.class` 文件，因此 Lambda 更轻量、类加载更少。

## 5.2 函数式接口与 @FunctionalInterface

**函数式接口**：有且仅有一个抽象方法的接口。接口上方可以加 `@FunctionalInterface` 注解，用来在编译期强制校验（非函数式接口加这个注解会编译报错）。

Lambda 只能简化**函数式接口**的匿名内部类写法——如果一个接口有多个抽象方法，就没办法用 Lambda 表示。

> **注意**：Lambda 中的 `this` 指向的是**包含该 Lambda 的类的实例**，而不是 Lambda 自身（因为它没有独立的 `this`）。这与匿名内部类不同——匿名内部类的 `this` 指向那个匿名对象自己。

## 5.3 Lambda 的延迟执行

Lambda 的一个重要特性是**延迟执行**：Lambda 体只有在函数式接口的抽象方法被真正调用时才执行。利用这一点可以把"可能耗时的计算"推迟到"确实需要时"才做，从而避免无谓开销。

```java
// 只有日志级别达标时才拼接字符串，不达标时 Lambda 体根本不执行
public static void log(int level, Supplier<String> message) {
    if (level == 1) {                     // 先判断条件
        System.out.println(message.get()); // 到这一步才真正执行 lambda 体、拼字符串
    }
}

log(2, () -> "一条很贵的日志：" + heavyComputation());  // level != 1，heavyComputation 不会被调用
```

如果把参数写成普通字符串 `String message`，那么无论级别是否达标，`heavyComputation()` 都会先被算出来——延迟执行的价值就在这里。

## 5.4 Lambda 的省略规则

总的来说就是"**可推导即可省略**"：

1. 参数类型可以省略（能从接口方法的签名推导出来）；
2. 只有一个参数时，参数类型和 `()` 都可以省略；
3. 方法体只有一行时，`{}`、`;` 和 `return` 可以**同时**省略（必须同时省）；
4. 有多个参数时，参数类型要么都写、要么都不写，不能只写一部分。

```java
// 完整写法
(int a, int b) -> { return a + b; }
// 省略参数类型
(a, b) -> { return a + b; }
// 省略大括号、分号和 return（单行表达式）
(a, b) -> a + b
// 单参数省略括号
x -> x * x
```

## 5.5 java.util.function 四大核心接口

`java.util.function` 包提供了一批内置函数式接口，记住四个核心的即可覆盖绝大多数场景：

| 接口 | 抽象方法 | 类型 | 典型用途 |
|------|---------|------|----------|
| `Consumer<T>` | `void accept(T t)` | 消费型：有进无出 | 遍历、打印：`list.forEach(...)` |
| `Supplier<T>` | `T get()` | 供给型：无进有出 | 工厂、懒加载：`() -> Math.random()` |
| `Function<T,R>` | `R apply(T t)` | 函数型：有进有出 | 映射、转换：`map(str -> str.length())` |
| `Predicate<T>` | `boolean test(T t)` | 断言型：有进，出布尔 | 过滤、判断：`list.removeIf(...)` |

常用变体：

| 变体 | 抽象方法 | 说明 |
|------|---------|------|
| `UnaryOperator<T>` | `T apply(T t)` | `Function` 的子接口，输入输出同类型，如 `list.replaceAll(s -> s.toUpperCase())` |
| `BinaryOperator<T>` | `T apply(T t1, T t2)` | `BiFunction` 的子接口，两个同类型入参、返回同类型，常用于求和 `(a, b) -> a + b` |
| `BiFunction<T,U,R>` | `R apply(T t, U u)` | 两个不同类型入参，返回 R |
| `BiConsumer<T,U>` | `void accept(T t, U u)` | 两个入参、无返回，如 Map 遍历 `map.forEach((k, v) -> ...)` |
| `IntConsumer` / `IntFunction` / `IntPredicate` / `IntSupplier`… | 基本类型特化 | 避免装箱拆箱开销，如 `IntPredicate` 的 `test(int)` |

```java
Consumer<String> printer = s -> System.out.println(s);          // 消费
Supplier<Double> random = () -> Math.random();                  // 供给
Function<String, Integer> length = String::length;             // 映射
Predicate<String> isLong = s -> s.length() > 5;                // 断言

printer.accept("hello");            // hello
System.out.println(random.get());  // 例如 0.7231...
System.out.println(length.apply("hello"));   // 5
System.out.println(isLong.test("hello"));    // false
```

## 5.6 Lambda 与 Stream 结合

Lambda 是 `Stream` 的"燃料"——`filter` 要 `Predicate`，`map` 要 `Function`，`forEach` 要 `Consumer`，`collect` 的 `Collectors` 也大量依赖函数式接口：

```java
List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "David");

List<String> result = names.stream()
    .filter(name -> name.length() > 3)      // Predicate：过滤长度大于 3 的
    .map(name -> name.toUpperCase())        // Function：转为大写
    .collect(Collectors.toList());          // 收集为 List

System.out.println(result);   // [ALICE, CHARLIE, DAVID]
```

> **注意**：`Stream` 的完整用法（获取 / 中间操作 / 终结操作、惰性求值、`Collectors` 常用方法）在第 13 篇展开，本篇只需建立"Lambda 是 Stream 的参数来源"这一层联系。

## 5.7 方法引用（`::` 操作符）

当 Lambda 体**只调用了一个已存在的方法**时，可以进一步简化为方法引用。四种形式：

| 形式 | 写法 | 示例 |
|------|------|------|
| 静态方法引用 | `类名::静态方法` | `s -> Integer.parseInt(s)` → `Integer::parseInt` |
| 实例方法引用（特定对象） | `对象名::实例方法` | `() -> obj.getName()` → `obj::getName` |
| 实例方法引用（任意对象） | `类名::实例方法` | `s -> s.toUpperCase()` → `String::toUpperCase` |
| 构造方法引用 | `类名::new` | `name -> new Student(name)` → `Student::new` |

其中第三种（"任意对象的实例方法"）最特殊：`String::toUpperCase` 里**第一个参数是方法的接收者**，其余参数才是方法参数。

> **注意**：方法引用的完整规则与更多形式（`this::`、`super::`、`类型[]::new` 等）见第 13 篇，本篇只讲基础形式，避免两篇重复。

# 6. 小结

| 主题 | 一句话要点 |
|------|-----------|
| 冒泡排序 | 相邻比较、大的后移；加 `swapped` 标志可提前终止；稳定，O(n²) |
| 选择排序 | 每轮选最小值放到已排序区末尾；交换次数最少；不稳定，恒为 O(n²) |
| 插入排序 | 从后往前找位置插入；基本有序时接近 O(n)；稳定 |
| 快速排序 | 分治 + partition；**基准选择**决定是否退化成 O(n²)，可用随机化/三数取中；平均 O(n log n)，不稳定 |
| Arrays.sort 的选择 | 基本类型用**双轴快排**，对象数组用 **TimSort**（稳定）——稳定性需求不同 |
| 二分查找 | **前提必须有序**；`mid = left + (right-left)/2` 防溢出；`Arrays.binarySearch` 未找到返回 `-(插入点)-1` |
| 递归 | 三要素：终止条件、递归公式、向终止条件收敛；每层压一个栈帧；JVM **不做尾递归优化**，深递归会 `StackOverflowError` |
| Arrays 工具类 | `toString` / `sort` / `binarySearch` / `copyOf` / `equals` / `asList`（**固定长度**）/ `stream` |
| Lambda | 是**函数式接口**的实例，用 `invokedynamic` 运行时生成；有**延迟执行**特性；省略规则"可推导即可省" |
| 函数式接口 | 四大核心：`Consumer`（消费）、`Supplier`（供给）、`Function`（映射）、`Predicate`（断言），另有 `UnaryOperator` / `BinaryOperator` 等变体 |
| 方法引用 | `::` 是 Lambda 的进一步简化，四种形式；详见第 13 篇 |

下一篇（第 13 篇）专门讲 `Stream` 流的完整用法与方法引用的所有细节；集合框架的基础知识回看第 9 篇。
