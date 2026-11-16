数组是 Java 里第一个真正意义上的**容器**，也是理解 JVM 内存模型的最佳切入点。在此之前，我们的数据单位都是"一个变量存一个值"；从数组开始，代码第一次需要在**栈和堆之间建立联系**——数组变量本身在栈上，它指向的对象在堆上，这个"引用"的概念会贯穿后面的对象、集合、多态与线程安全。本篇从定义与初始化讲起，用内存图讲清引用与共享，再收束到 JVM 运行时数据区的完整划分。

# 1. 数组是什么

## 1.1 定义与三个特征

**数组**（Array）是一种**引用数据类型**，用于在**一块连续的内存**中存储**相同类型**的多个数据。它有三个必须同时记住的特征：

1. **类型相同**——一个数组只能存一种类型，可以是基本类型（`int[]`、`double[]`），也可以是引用类型（`String[]`、`Student[]`，甚至 `int[][]`）。
2. **长度固定**——一旦创建，长度永不可改（`arr.length` 由 JVM 保证只读）。需要"可变长度"就得用集合，见第 9 篇。
3. **连续存储 + 索引访问**——元素在内存中连续排列，通过从 `0` 开始的下标定位，访问任意元素的时间复杂度都是 **O(1)**。

```java
public class ArrayIntroDemo {
    public static void main(String[] args) {
        int[] nums = {10, 20, 30};        // 长度为 3 的 int 数组

        System.out.println(nums[0]);      // 10   下标从 0 开始
        System.out.println(nums[2]);      // 30   最后一个元素的下标是 length - 1
        System.out.println(nums.length);  // 3    length 是属性，后面没有括号
    }
}
```

如果不用数组，存 3 个分数要写 `int s1, s2, s3;`，存 100 个就得写 100 行声明。数组把"一组同类型数据"变成了一个**可以整体操作的对象**——可以整体传递、整体取长度、整体循环处理。

## 1.2 为什么访问是 O(1)：连续内存与寻址公式

下标访问能到 O(1)，靠的是一个简单的寻址公式：

```text
   元素地址 = 数组首地址 + 下标 * 单个元素占用的字节数

   arr ──▶ ┌────────┬────────┬────────┬────────┐
           │ arr[0] │ arr[1] │ arr[2] │ arr[3] │
           └────────┴────────┴────────┴────────┘
           首地址     +4       +8       +12      （int 占 4 字节）

   访问 arr[2]：不需要"从头数"，直接算 首地址 + 2 * 4 就得到地址
```

因为元素**连续**且**等长**，`arr[i]` 的地址可以一步算出来。这两点也解释了数组的两个"不方便"：

- **为什么长度不可变**：扩容意味着要在堆上重新申请一块更大的**连续**空间，再把数据搬过去。Java 把这个动作留给了集合——`ArrayList` 的扩容就是这样做的（每次扩容 1.5 倍，见第 9 篇）。
- **为什么不能插队**：在中间插入一个元素，后面所有元素都得整体后移。这也是 `ArrayList` 随机插入慢的根源。

## 1.3 与集合的关系

一句话区分：**数组定长、能存基本类型、性能最好；集合变长、只能存引用类型、功能丰富**。日常业务代码用 `ArrayList` 居多，但在性能敏感、长度确定的场景（如批量计算、缓冲区、底层框架）数组仍有不可替代的位置。两者的完整对比与选择见第 9 篇。

# 2. 数组的定义与初始化

## 2.1 两种声明格式

```java
int[] arr1;        // 格式一：TYPE[] 变量名 —— 推荐，类型与变量一目了然
int arr2[];        // 格式二：TYPE 变量名[] —— C/C++ 风格，Java 兼容但不推荐
```

两种写法**完全等价**（编译出的字节码一样），推荐格式一。差别在多变量声明时最明显：

```java
int[] a, b;        // a 和 b 都是 int 数组
int c[], d;        // c 是 int 数组，但 d 只是一个 int —— 极易看错
```

**声明只是声明**，此时 `a` 里还没有任何数组对象，它的值是 `null`。

## 2.2 静态初始化

静态初始化在创建时就给定元素的值，**数组长度由 JVM 根据元素个数自动确定**。

### 2.2.1 完整格式

```java
int[] numbers = new int[]{10, 20, 30};
String[] names = new String[]{"Alice", "Bob"};
double[] empty = new double[]{};            // 长度为 0 的合法数组
int[][] grid = new int[][]{{1, 2}, {3, 4}}; // 二维数组
```

### 2.2.2 简化格式

```java
int[] numbers = {10, 20, 30};
String[] names = {"Alice", "Bob"};
int[][] grid = {{1, 2}, {3, 4}};
```

### 2.2.3 简化格式不能先声明后赋值

这是静态初始化最容易踩的一条限制：**简化格式只能用在声明变量的同一行**。

```java
int[] arr;
// arr = {1, 2, 3};               // ✗ 编译错误：数组初始化器不能用在赋值语句中
arr = new int[]{1, 2, 3};         // ✓ 必须用完整格式

// 作为方法实参时同理：
// printArr({1, 2, 3});           // ✗ 编译错误
printArr(new int[]{1, 2, 3});     // ✓
```

**原因在语法层面**：`{1, 2, 3}` 叫做**数组初始化器**（Array Initializer），它只在**变量声明语句**的上下文里被允许——编译器需要从声明处的类型推断出元素类型和数组类型。在普通赋值语句或方法实参里，`{...}` 是**块语句**的开头（`{` 后面跟语句），编译器无法区分你要表达什么，于是直接判定为语法错误。

### 2.2.4 什么时候用哪种初始化

| 场景 | 用哪种 | 例子 |
|------|--------|------|
| 元素值**已知且固定**，就在声明处 | 静态初始化（简化格式） | `int[] week = {1, 2, 3, 4, 5, 6, 7};` |
| 元素值**已知但不在声明处**（方法返回值、参数传入） | 静态初始化（**完整格式**） | `return new int[]{lo, hi};` |
| 只知道**长度**，值后续计算填入 | 动态初始化 | `int[] buf = new int[1024];` |
| 需要**空数组**做兜底 | 任意 | `new int[0]` /  `new int[]{}` |

## 2.3 动态初始化

只指定长度，元素使用**默认值**：

```java
int[] arr = new int[5];                // 长度 5，五个元素都是 0
double[] scores = new double[3];       // 都是 0.0
boolean[] flags = new boolean[10];     // 都是 false
String[] names = new String[3];        // 都是 null

// 长度可以是变量，也可以是表达式
int size = 5;
double[] d = new double[size];
int[] e = new int[size * 2 + 1];       // 长度 11
```

`new` 后面括号里的长度可以是变量，但**变量必须已经有值**：

```java
int n;
// int[] bad = new int[n];             // ✗ 编译错误：变量 n 可能尚未初始化
n = 5;
int[] good = new int[n];               // ✓
```

> **陷阱**：动态初始化的长度是**运行时**才确定的，为负数会抛 `NegativeArraySizeException`（运行时异常，编译器查不出来）。这与"数组长度不可变"是两回事——长度可以在运行时决定，但一旦定下来就不能改。

```java
int n = -1;
// new int[n];                         // ✗ 运行时抛 NegativeArraySizeException: -1
```

## 2.4 元素的默认值

数组创建后，**每个元素都会被自动赋予该类型的零值**。这一点与局部变量不同——局部变量不初始化直接使用会**编译报错**，而数组元素会被 JVM 自动清零。

| 元素类型 | 默认值 | 打印出来的样子 |
|----------|--------|---------------|
| `byte` / `short` / `int` / `long` | `0` | `0` |
| `float` / `double` | `0.0` | `0.0` |
| `char` | `'\u0000'` | 一个不可见的空白字符 |
| `boolean` | `false` | `false` |
| 引用类型（`String`、对象、另一个数组） | `null` | `null` |

```java
public class DefaultValueDemo {
    public static void main(String[] args) {
        System.out.println(new int[1][0]);                  // 0
        System.out.println(new double[1][0]);               // 0.0
        System.out.println(new boolean[1][0]);              // false
        System.out.println("[" + new char[1][0] + "]");     // []    char 默认是 \u0000
        System.out.println(new String[1][0]);               // null
        System.out.println((int) new char[1][0]);           // 0
        System.out.println((int) ' ');                      // 32
    }
}
```

> **注意**：`char` 的默认值不是空格（`' '`，即 `' '`）而是 `'\u0000'`。两者打印出来都像空白，但 `(int) '\u0000'` 是 0，`(int) ' '` 是 32。用 `char[]` 拼字符串或做位运算时踩过这个坑的人不少。

# 3. 元素访问与索引

格式：`数组名[下标]`

- 下标从 **0** 开始，合法范围是 `[0, length - 1]`；
- 下标必须是整数类型（`byte`、`short`、`int`、`char`），**不能是 `long`**；
- 数组表达式既可以**读**（右值）也可以**写**（左值）：

```java
public class AccessDemo {
    public static void main(String[] args) {
        int[] arr = new int[3];

        arr[0] = 20;                    // 写：把 20 放进 0 号位置
        arr[1] = 100;
        System.out.println(arr[0]);     // 读：20
        System.out.println(arr[0] + arr[1]);   // 120   可以参与运算

        int i = 0;
        arr[i] = 200;                   // 下标可以是变量
        System.out.println(arr[0]);     // 200

        // char 也能当下标（会提升为 int）
        System.out.println(arr['a' - 97]);  // 200   'a'-97 == 0

        // 复合赋值也能作用于元素
        arr[2] += 5;                    // 等价于 arr[2] = arr[2] + 5
        System.out.println(arr[2]);     // 5
    }
}
```

```text
   int[] arr = new int[3]; 然后 arr[0]=200, arr[1]=100, arr[2]=5

   ┌─────────┬─────────┬─────────┐
   │  200    │  100    │    5    │      上面一行是元素的值
   ├─────────┴─────────┴─────────┤
   │ arr[0]  │ arr[1]  │ arr[2]  │      这一行是合法下标范围：0 ~ length-1
   └─────────┴─────────┴─────────┘
    第一个元素                       再往后一个（arr[3]）就越界了
```

# 4. 数组的内存模型

这是本篇最重要的一节。**数组变量存的不是数组本身，而是数组对象在堆中的地址**——理解了这一点，"两个变量共享同一个数组""传参能不能改到原数组""为什么打印出来是乱码"这些问题就都有了统一答案。

## 4.1 栈与堆的分工

| 区域 | 存什么 | 谁负责分配 |
|------|--------|-----------|
| **栈** | 方法调用时的栈帧：局部变量、方法参数、**对象引用（地址）** | 编译器确定大小，随方法调用返回自动进出 |
| **堆** | 所有 `new` 出来的东西：**数组对象、对象实例** | JVM 动态分配，GC 负责回收 |

## 4.2 内存图：`int[] arr = new int[3]`

```text
   栈（每个线程私有）                    堆（所有线程共享，GC 管理）
   ┌──────────────────────────┐        ┌───────────────────────────────────┐
   │ main 方法的栈帧             │        │   数组对象（new int[3]）            │
   │                          │        │                                   │
   │  ┌────────────────────┐  │        │   ┌────────┬────────┬────────┐  │
   │  │  arr  =  0x00A1F3C4│─────────▶│   │   0    │   0    │   0    │  │
   │  │                     │  │        │   └────────┴────────┴────────┘  │
   │  └────────────────────┘  │        │    arr[0]   arr[1]   arr[2]      │
   │                          │        │    首地址 0x00A1F3C4              │
   └──────────────────────────┘        └───────────────────────────────────┘

   1) new int[3] 在堆中开辟 3 * 4 = 12 字节的连续空间，元素全部清零
   2) 这块空间的首地址（假设 0x00A1F3C4）被赋给栈上的变量 arr
   3) 之后所有对 arr[i] 的读写，都是"取 arr 里的地址，按偏移找元素"
```

由这张图可以推出三个结论：

1. **`arr` 本身只占 4 或 8 个字节**（一个地址的大小），不管它指向的数组有 3 个元素还是 300 万个；
2. **`arr.length` 存在堆上的数组对象里**（对象头之后），所以它是"数组对象"的属性，而不是变量的属性；
3. **数组是引用类型**，`arr` 可以被赋值为 `null`，此时它不指向任何对象，访问元素会抛 `NullPointerException`。

```java
int[] arr = new int[3];
System.out.println(arr);            // [I@1b6d3586
System.out.println(arr.getClass()); // class [I
System.out.println(arr.getClass().getName());  // [I
```

`[I@1b6d3586` 里每个字符都有含义：

| 片段 | 含义 |
|------|------|
| `[` | 表示这是一个数组（有多少个 `[` 就是多少维） |
| `I` | 元素类型是 `int`（`L` 开头加 `;` 表示引用类型，如 `[Ljava.lang.String;`） |
| `@` 后面 | 对象的 `hashCode()` 的十六进制（**不是**内存地址，但可以当作身份标识） |

所以 `[I` 是 `int[]`，`[[I` 是 `int[][]`，`[Ljava.lang.String;` 是 `String[]`。理解了这套编码，看 JVM 日志和反射报错时会轻松很多。

## 4.3 数组共享（引用复制）

`int[] arr2 = arr1;` 复制的是**地址**，不是元素——两个变量指向堆上**同一个数组对象**。

```java
public class ShareDemo {
    public static void main(String[] args) {
        int[] arr1 = {1, 2, 3};
        int[] arr2 = arr1;              // 复制的是地址，没有任何元素被复制
        arr2[0] = 99;

        System.out.println(arr1[0]);    // 99   arr1 看到的也变了
        System.out.println(arr1 == arr2);// true 比较的是地址，是同一个对象
        System.out.println(arr1.equals(arr2)); // true 数组没有重写 equals，等价于 ==
    }
}
```

```text
   int[] arr1 = {1, 2, 3};
   int[] arr2 = arr1;
   arr2[0] = 99;

   栈                             堆
   ┌───────────────┐           ┌──────────────────────────────┐
   │ arr1 = 0x100  │─────┬────▶│  ┌────────┬────────┬──────┐  │
   └───────────────┘     │     │  │   99   │   2    │  3   │  │
   ┌───────────────┐     │     │  └────────┴────────┴──────┘  │
   │ arr2 = 0x100  │─────┘     │   arr[0] 被改成了 99          │
   └───────────────┘           └──────────────────────────────┘
   （两个引用指向同一块内存，这就是"数组共享"的全部含义）
```

**想要真正独立的两份数据，必须显式复制**：

```java
int[] src = {1, 2, 3};

int[] a = src.clone();                          // 方式一：clone()
int[] b = Arrays.copyOf(src, src.length);       // 方式二：Arrays.copyOf
int[] c = new int[src.length];
System.arraycopy(src, 0, c, 0, src.length);     // 方式三：System.arraycopy（最快）

c[0] = 99;
System.out.println(src[0]);                     // 1 改的是副本，原数组不受影响
System.out.println(a[0]);                       // 1
System.out.println(Arrays.toString(b));         // [1, 2, 3]
```

三种方式都是**浅拷贝**。对基本类型数组，浅拷贝就是完整的值复制；但对**对象数组**或**二维数组**，复制的是元素的引用，两个数组的同一个位置仍指向同一个对象：

```java
int[][] grid = {{1, 2}, {3, 4}};
int[][] copy = grid.clone();        // 只复制了外层数组，内层还是同一个对象
copy[0][0] = 99;
System.out.println(grid[0][0]);     // 99  原数组被改到了

// 想彻底独立，必须逐层复制
int[][] deep = new int[grid.length][];
for (int i = 0; i < grid.length; i++) {
    deep[i] = grid[i].clone();      // 每一行都单独复制
}
deep[0][0] = -1;
System.out.println(grid[0][0]);     // 99  这次没被影响
System.out.println(deep[0][0]);     // -1
```

深拷贝与浅拷贝的系统讨论在第 10 篇。

## 4.4 数组作为方法参数

Java **只有值传递**：基本类型传的是值的副本，引用类型传的是**地址的副本**。所以：

```java
public class PassDemo {
    static void modify(int[] a) {
        a[0] = 99;                      // ✓ 通过地址改了堆上的内容，调用方能看到
        a = new int[]{7, 7, 7};         // ✗ 只是让形参 a 指向了新对象，与调用方无关
        a[0] = 1000;
    }

    public static void main(String[] args) {
        int[] arr = {1, 2, 3};
        modify(arr);
        System.out.println(arr[0]);     // 99 —— 是 99，不是 1000
    }
}
```

两个动作的区别是理解"传引用"的关键：

| 动作 | 效果 | 调用方可见 |
|------|------|-----------|
| 修改 `a[i]`（改堆上的内容） | 通过副本地址找到同一个对象并改它 | **是** |
| 给 `a` 重新赋值（改地址本身） | 只改了栈上的那个副本 | 否 |

同理，交换两个数组的方法**无法生效**：

```java
static void swapWrong(int[] x, int[] y) {
    int[] tmp = x; x = y; y = tmp;      // ✗ 换的是形参的地址副本，外面没变
}

static void swapRight(int[] x, int[] y) {
    for (int i = 0; i < x.length; i++) { // ✓ 真正交换内容，调用方可见
        int tmp = x[i]; x[i] = y[i]; y[i] = tmp;
    }
}
```

> **陷阱**：形如 `swap(int[] a, int[] b)` 的方法**永远不可能实现数组变量的交换**，这是 Java 语言层面的限制（没有 `ref`/`out` 参数）。要想"交换"只能交换内容，或者把两个数组包进一个对象再交换对象字段。

# 5. 数组的遍历

## 5.1 普通 for 循环

```java
int[] arr = {10, 20, 30};
for (int i = 0; i < arr.length; i++) {
    System.out.println("arr[" + i + "] = " + arr[i]);
}
// arr[0] = 10
// arr[1] = 20
// arr[2] = 30
```

循环条件用 `i < arr.length`（**不是** `<=`），这是最常见的越界来源。也可以反向遍历：

```java
for (int i = arr.length - 1; i >= 0; i--) {
    System.out.print(arr[i] + " ");     // 30 20 10
}
System.out.println();
```

## 5.2 增强 for 循环（foreach）

```java
int[] arr = {10, 20, 30};
for (int x : arr) {
    System.out.println(x);              // 10 20 30
}
```

它的内部实现就是普通 for + 下标（对数组）或迭代器（对集合），好处是**不用管下标，也就写不出越界**。

## 5.3 对比与选择

| 维度 | 普通 for | 增强 for |
|------|----------|----------|
| 是否需要下标 | 需要（计算、回填、比较相邻元素） | 不需要 |
| 能否反向遍历 | 能 | 不能 |
| 能否跳步（`i += 2`） | 能 | 不能 |
| 越界风险 | 有（条件写错） | 无 |
| 遍历中修改元素 | 能（`arr[i] = x`） | **改不到**（改的是副本） |
| 遍历中删除元素 | 一般可行（配合下标与 `i--`） | 集合会抛异常，数组无所谓 |
| 可读性 | 一般 | 好 |

```java
int[] scores = {88, 95, 72};

// 需要下标：找最大值所在的索引
int maxIdx = 0;
for (int i = 1; i < scores.length; i++) {
    if (scores[i] > scores[maxIdx]) maxIdx = i;
}
System.out.println("最高分 " + scores[maxIdx] + "，在第 " + maxIdx + " 位");  // 最高分 95，在第 1 位

// 只读：求和
int sum = 0;
for (int s : scores) sum += s;
System.out.println(sum);    // 255
```

> **最佳实践**：**默认用增强 for，需要下标时才用普通 for**。判断标准是问自己两个问题——"我需要知道它是第几个吗？""我需要修改元素吗？"两个都为否，就用增强 for。反之，凡是"回填数组""比较相邻元素""从后往前"这类需求，一律用普通 for。

# 6. 二维数组

二维数组本质是"**数组的数组**"：外层数组的每个元素，都指向一个内层数组。

```java
// 静态初始化
int[][] grid = {{1, 2, 3}, {4, 5}, {6}};

// 动态初始化：3 行 4 列
int[][] matrix = new int[3][4];        // 12 个元素，全是 0
int[][] multi = new int[2][];          // 只指定行数，内层还是 null
multi[0] = new int[]{1, 2};            // 逐行分配
multi[1] = new int[]{3, 4, 5, 6};      // 每行长度可以不同
```

访问用 `grid[行][列]`，行列下标都从 0 开始：

```java
int[][] grid = {{1, 2, 3}, {4, 5}};
System.out.println(grid[0][0]);         // 1
System.out.println(grid[1][1]);         // 5
System.out.println(grid.length);        // 2   行数
System.out.println(grid[0].length);     // 3   第 0 行的列数
System.out.println(grid[1].length);     // 2   第 1 行的列数（不一样！）
```

遍历要两层循环：

```java
// 普通 for
for (int i = 0; i < grid.length; i++) {            // grid.length 是行数
    for (int j = 0; j < grid[i].length; j++) {     // grid[i].length 是这一行的列数
        System.out.print(grid[i][j] + " ");
    }
    System.out.println();
}
// 1 2 3
// 4 5

// 增强 for：外层拿到的是"一行"，类型是 int[]
for (int[] row : grid) {
    for (int x : row) {
        System.out.print(x + " ");
    }
    System.out.println();
}
```

二维数组的内存结构（注意内层数组**不是连续**排在后面的）：

```text
   栈                  堆
   ┌───────────┐     ┌─────────────────────────────┐
   │ grid=0x200│────▶│  外层数组（长度 2）            │
   └───────────┘     │  ┌──────────┬──────────┐   │
                     │  │  0x300   │  0x400   │   │
                     │  └────┬─────┴────┬─────┘   │
                     └───────┬──────────┬─────────┘
                             │          │
                             ▼          ▼
                     ┌────────────┐ ┌─────────────┐
                     │  1  2  3   │ │  4  5       │      每行长度可以不同
                     └────────────┘ └─────────────┘
                        行 0            行 1
```

| 表达式 | 含义 | 类型 |
|--------|------|------|
| `grid` | 整个二维数组的引用 | `int[][]` |
| `grid.length` | **行数**（外层数组的长度） | `int` |
| `grid[i]` | 第 i 行（一个一维数组） | `int[]` |
| `grid[i].length` | 第 i 行的**列数** | `int` |
| `grid[i][j]` | 第 i 行第 j 列的元素 | `int` |

> **陷阱**：**二维数组每行的长度可以不同**（这叫"锯齿数组"）。`new int[3][4]` 只是"3 行且每行 4 列"的简写，不能假设所有行一样长。用 `grid[0].length` 去驱动所有行的遍历，遇到长度不同的行就会越界。**遍历第 i 行时一定要用 `grid[i].length`。**

> **陷阱**：`new int[3][]` 创建后，内层三行都是 `null`。此时访问 `grid[0][0]` 抛的是 **`NullPointerException`**（因为 `grid[0]` 是 `null`），而不是 `ArrayIndexOutOfBoundsException`——异常类型不同，排查方向也不同。只 `new` 了外层而忘了初始化内层，是二维数组最常见的错误。

# 7. 常见异常与陷阱

## 7.1 数组未初始化

只声明不赋值时数组变量是 `null`，访问元素直接 NPE：

```java
int[] arr;
// arr[0] = 1;                     // ✗ 编译错误：变量 arr 可能尚未初始化

int[] arr2 = null;
// arr2[0] = 1;                    // ✗ 运行时 NullPointerException

int[] arr3 = new int[3];
arr3[0] = 1;                       // ✓
```

两种情况要分清：

| 情况 | 报错时机 | 报错类型 |
|------|----------|----------|
| 局部变量**从未赋值**就使用 | **编译期** | 编译错误："变量可能尚未初始化" |
| 变量赋值为 `null` 后使用 | **运行时** | `NullPointerException` |

前者编译器帮你挡住了，后者只能靠测试和空值校验：

```java
// 正确姿势：先判 null，再判长度（顺序不能反）
if (arr != null && arr.length > 0) {
    System.out.println(arr[0]);
}
```

## 7.2 数组越界

访问 `[0, length - 1]` 之外的索引，抛 `ArrayIndexOutOfBoundsException`（运行时异常）：

```java
int[] arr = {1, 2, 3};
System.out.println(arr[3]);
// 异常信息：Exception in thread "main" java.lang.ArrayIndexOutOfBoundsException:
//          Index 3 out of bounds for length 3
```

异常信息里明确给出了"**访问的索引**"和"**数组长度**"这两个数字，看到它们就能立刻定位问题。三个高频越界写法：

```java
// (1) 条件用了 <=
for (int i = 0; i <= arr.length; i++) { }        // ✗ i 会取到 length

// (2) 下标计算多算了 1
System.out.println(arr[arr.length - 0]);          // ✗ 等价于 arr[length]

// (3) 先取值后自减
int i = arr.length;
while (i > 0) { arr[i--] = 0; }                   // ✗ 第一次就是 arr[length]
while (i > 0) { arr[--i] = 0; }                   // ✓ 先减再用，正确
```

## 7.3 `length` 是属性不是方法

```java
int[] arr = new int[3];
System.out.println(arr.length);      // 3     属性，没有括号
// System.out.println(arr.length()); // ✗ 编译错误：找不到符号 length()
```

对比一下其他"取长度"的写法，别记混：

| 类型 | 取长度 | 本质 |
|------|--------|------|
| 数组 | `arr.length` | **属性**（字段） |
| `String` | `str.length()` | **方法** |
| `List` / `Set` / `Map` | `list.size()` | **方法** |
| `Collection` | `c.size()` | 方法 |
| `File` | `file.length()` | 方法 |

**为什么数组的 `length` 是属性？** 因为数组是 JVM **内建**的类型，不是普通的类。它的 `length` 由 JVM 直接支持（就在对象头之后的位置），不走方法调用，也就没有任何调用开销。

## 7.4 长度不可修改

```java
int[] arr = new int[3];
// arr.length = 5;                   // ✗ 编译错误：无法为最终变量 length 分配值
```

`length` 是 `final` 的。需要"变大"时只能新建一个更大的数组再把数据复制过去——`Arrays.copyOf` 就是干这个的，而这也正是 `ArrayList` 扩容的底层动作：

```java
int[] old = {1, 2, 3};
int[] bigger = Arrays.copyOf(old, 6);
System.out.println(Arrays.toString(bigger));   // [1, 2, 3, 0, 0, 0]
// old 本身没有任何变化，只是多了一个新数组
System.out.println(Arrays.toString(old));      // [1, 2, 3]
```

## 7.5 其它易错点

```java
// (1) "空数组"和 null 是两回事
int[] empty = new int[0];
System.out.println(empty.length);    // 0      合法，只是个没有元素的数组
int[] nil = null;
// System.out.println(nil.length);   // ✗ NullPointerException
// 空数组可以安全遍历（一次都不进循环），null 不能

// (2) 下标不能是 long
int[] a = new int[3];
long l = 1;
// System.out.println(a[l]);         // ✗ 编译错误：从 long 转换到 int 可能会有损失
System.out.println(a[(int) l]);      // ✓ 需要显式强转

// (3) 忘记给数组元素赋初值就用
double[] scores = new double[3];
double total = 0;
for (int i = 0; i < scores.length; i++) {
    total += scores[i];              // 逻辑上行得通，但全是 0，平均值也是 0
}
System.out.println(total / scores.length);   // 0.0 —— 业务上大概率是错的

// ④ 数组长度为 0 时做除法
int[] data = new int[0];
// System.out.println(10 / data.length);   // ✗ ArithmeticException: / by zero
```

# 8. 数组的常见操作

## 8.1 求最值

```java
public class MaxMinDemo {
    public static void main(String[] args) {
        int[] arr = {12, 45, 7, 89, 33};

        int max = arr[0];                        // 从第一个元素出发
        for (int x : arr) {
            if (x > max) max = x;
        }
        System.out.println("最大值: " + max);     // 最大值: 89

        int min = arr[0];
        for (int x : arr) {
            if (x < min) min = x;
        }
        System.out.println("最小值: " + min);     // 最小值: 7
    }
}
```

> **陷阱**：把初值写成 `0` 在"全是负数"的数组上会给出错误答案——`{-5, -3}` 会返回 0 而不是 -3。**最保险的做法是拿数组的第一个元素做初值**（前提是数组非空，用之前先判 `length > 0`）。同理求最小值时不能把初值设成 `Integer.MAX_VALUE` 以外的"业务哨兵值"，那会污染结果。

## 8.2 求和与平均值

```java
int[] arr = {1, 2, 3, 4};
int sum = 0;
for (int x : arr) sum += x;

System.out.println(sum);                            // 10
System.out.println(sum / arr.length);               // 2    两个 int 相除被截断
System.out.println((double) sum / arr.length);      // 2.5  先强转，再除
System.out.println(sum * 1.0 / arr.length);         // 2.5  等价写法
```

**求平均值时整除陷阱是第一号错误**，原因见第 3 篇 2.1。

## 8.3 数组反转

```java
// 双指针法：一头一尾交换，向中间收缩
int[] arr = {1, 2, 3, 4, 5};
for (int i = 0, j = arr.length - 1; i < j; i++, j--) {
    int tmp = arr[i];
    arr[i] = arr[j];
    arr[j] = tmp;
}
System.out.println(Arrays.toString(arr));    // [5, 4, 3, 2, 1]
```

条件是 `i < j` 而不是 `i <= j`：当长度为奇数时，中间的 `i == j` 是自己和自己交换，没有意义，直接停就好。另外这个例子也说明了第 4 节 `4.4` 的结论——**必须交换元素内容，交换变量是没用的**。

## 8.4 查找

```java
// 线性查找：不要求有序，逐个比较，O(n)
int[] arr = {1, 2, 3, 4};
int target = 3;
int index = -1;
for (int i = 0; i < arr.length; i++) {
    if (arr[i] == target) { index = i; break; }   // 找到就跳出
}
System.out.println(index);                              // 2

// 二分查找：要求数组已排序，O(log n)
System.out.println(Arrays.binarySearch(arr, 3));        // 2
System.out.println(Arrays.binarySearch(arr, 5));        // -5
System.out.println(Arrays.binarySearch(arr, 0));        // -1
```

二分查找的返回值有讲究：**未找到时返回 `-(插入点) - 1`**，负数不只是"没找到"的信号，它还能告诉你"如果插入，应该放在哪个下标"：

| 查找值 | 返回 | 含义 |
|--------|------|------|
| `3`（存在） | `2` | 在下标 2 |
| `0`（不存在） | `-1` | 插入点是 0，`-(0)-1 = -1` |
| `5`（不存在） | `-5` | 插入点是 4，`-(4)-1 = -5` |
| `2`（存在） | `1` | 在 |

排序与查找算法的完整实现（冒泡、选择、插入、快排、二分）见第 12 篇。

## 8.5 排序

```java
int[] arr = {5, 2, 9, 1};
Arrays.sort(arr);                             // 默认升序
System.out.println(Arrays.toString(arr));     // [1, 2, 5, 9]

// 只排一段区间 [from, to)
int[] arr2 = {5, 2, 9, 1, 7};
Arrays.sort(arr2, 1, 4);
System.out.println(Arrays.toString(arr2));    // [5, 1, 2, 9, 7]

// 字符串数组按字典序
String[] names = {"Bob", "Alice", "Cindy"};
Arrays.sort(names);
System.out.println(Arrays.toString(names));   // [Alice, Bob, Cindy]

// 自定义排序规则（需要包装类型或 Comparator）
Integer[] nums = {3, 1, 2};
Arrays.sort(nums, (x, y) -> y - x);           // 降序
System.out.println(Arrays.toString(nums));    // [3, 2, 1]
```

`Arrays.sort` 对**基本类型**用双轴快速排序（DualPivotQuicksort，不稳定），对**对象数组**用 TimSort（稳定）。排序算法的细节见第 12 篇。

## 8.6 复制

| 方式 | 写法 | 特点 |
|------|------|------|
| `Arrays.copyOf` | `Arrays.copyOf(src, newLen)` | 长度可加长（补默认值）或截短，最常用 |
| `Arrays.copyOfRange` | `Arrays.copyOfRange(src, from, to)` | 复制区间 `[from, to)` |
| `System.arraycopy` | `System.arraycopy(src, sIdx, dst, dIdx, len)` | **性能最好**，底层是内存拷贝，支持区间与自重叠复制 |
| `arr.clone()` | `int[] b = arr.clone()` | 简洁，长度不变 |
| 手动循环 | `for (int i...) b[i] = a[i];` | 最慢，只在需要自定义逻辑时用 |

```java
int[] src = {1, 2, 3};

int[] longer = Arrays.copyOf(src, 5);
System.out.println(Arrays.toString(longer));    // [1, 2, 3, 0, 0] 多出来的补默认值

int[] shorter = Arrays.copyOf(src, 2);
System.out.println(Arrays.toString(shorter));   // [1, 2]

int[] range = Arrays.copyOfRange(src, 0, 2);
System.out.println(Arrays.toString(range));     // [1, 2] 不含下标 2

int[] part = new int[3];
System.arraycopy(src, 0, part, 0, 3);
System.out.println(Arrays.toString(part));      // [1, 2, 3]

System.arraycopy(src, 1, part, 0, 2);           // 从下标 1 复制 2 个到 part 的下标 0
System.out.println(Arrays.toString(part));      // [2, 3, 3]

// 自重叠复制也能正确处理（内部按方向拷贝，不会互相覆盖）
int[] dup = {1, 2, 3, 4, 5};
System.arraycopy(dup, 0, dup, 1, 4);
System.out.println(Arrays.toString(dup));       // [1, 1, 2, 3, 4]
```

## 8.7 打印数组

```java
int[] arr = {1, 2, 3};
System.out.println(arr);                        // [I@1b6d3586  打印的是"地址"
System.out.println(Arrays.toString(arr));       // [1, 2, 3]    这才是想要的内容

int[][] grid = {{1, 2}, {3, 4}};
System.out.println(Arrays.toString(grid));      // [[I@1b6d3586, [I@7852e922]
System.out.println(Arrays.deepToString(grid));  // [[1, 2], [3, 4]]
```

**直接 `println` 一个数组永远得不到想要的结果**，必须用 `Arrays.toString`（一维）或 `Arrays.deepToString`（多维）。

## 8.8 常用工具方法速查

| 方法 | 作用 |
|------|------|
| `Arrays.toString(arr)` | 一维数组转字符串（打印/日志必备） |
| `Arrays.deepToString(arr)` | 多维数组转字符串 |
| `Arrays.sort(arr)` / `sort(arr, from, to)` | 升序排序，可指定区间 |
| `Arrays.sort(arr, comparator)` | 自定义规则排序（需包装类型） |
| `Arrays.binarySearch(arr, key)` | 二分查找（要求已排序） |
| `Arrays.copyOf(arr, newLen)` | 复制并调整长度 |
| `Arrays.copyOfRange(arr, from, to)` | 复制区间 `[from, to)` |
| `Arrays.fill(arr, val)` | 用同一个值填满整个数组 |
| `Arrays.fill(arr, from, to, val)` | 填充指定区间 |
| `Arrays.equals(a, b)` | 比较两个数组**内容**是否相同（`==` 比的是地址） |
| `Arrays.compare(a, b)` | 字典序比较（JDK9+） |
| `Arrays.asList(...)` | 转成 `List`（**定长**，见第 9 篇） |
| `Arrays.stream(arr)` | 转成流，进入 Stream 世界（见第 13 篇） |
| `System.arraycopy(...)` | 最高效的数组拷贝 |

```java
// Arrays.equals 与 == 的区别
int[] x = {1, 2, 3};
int[] y = {1, 2, 3};
System.out.println(x == y);                  // false  不是同一个对象
System.out.println(Arrays.equals(x, y));     // true   内容相同
System.out.println(x.equals(y));             // false  数组没有重写 equals，等价于 ==

// Arrays.fill 的两种用法
int[] f = new int[5];
Arrays.fill(f, 7);
System.out.println(Arrays.toString(f));      // [7, 7, 7, 7, 7]
Arrays.fill(f, 1, 3, 0);                     // 只填下标 1~2
System.out.println(Arrays.toString(f));      // [7, 0, 0, 7, 7]
```

> **注意**：`Arrays.asList(...)` 返回的是**定长**列表（内部是 `Arrays$ArrayList`，不是 `java.util.ArrayList`），调用 `add`/`remove` 会抛 `UnsupportedOperationException`。需要可变列表时要包一层 `new ArrayList<>(Arrays.asList(...))`。这个坑的详细说明在第 9 篇。

# 9. JVM 运行时数据区

Java 程序运行时，JVM 把内存划分成若干区域，各自的用途、生命周期、线程可见性都不同。这是理解后面所有"内存"问题的基础——**对象在哪里、GC 回收谁、为什么会 OOM、为什么局部变量天然线程安全**，答案都在这张表里。

| 区域 | 存什么 | 是否线程共享 | 生命周期 | 常见异常 |
|------|--------|-------------|-----------|----------|
| **堆（Heap）** | `new` 出来的一切：**数组**、对象实例；JDK7 起字符串常量池也移入这里 | **共享** | JVM 启动到退出，由 GC 管理 | `OutOfMemoryError: Java heap space` |
| **方法区（Method Area）** | 类元信息、静态变量、运行时常量池、JIT 编译后的代码 | **共享** | JVM 启动到退出 | `OutOfMemoryError: Metaspace` |
| **虚拟机栈（VM Stack）** | 每个方法一个**栈帧**：局部变量表（含**对象引用**）、操作数栈、动态链接、返回地址 | **私有**（每个线程一个） | 随方法调用进入/返回而进出，线程结束销毁 | `StackOverflowError`（递归太深） |
| **本地方法栈（Native Method Stack）** | JVM 调用 native 方法（C/C++ 实现）时的栈帧 | **私有** | 同上 | `StackOverflowError` |
| **程序计数器（PC Register）** | 当前线程正在执行的字节码行号 | **私有** | 随线程 | **唯一不会抛 OOM 的区域** |

```text
   ┌───────────────────────────────────────────────────────────────┐
   │                      JVM 运行时数据区                           │
   ├───────────────────────────────┬───────────────────────────────┤
   │       线程共享（所有线程可见）    │      线程私有（每个线程一份）     │
   │                               │                               │
   │  ┌─────────────────────────┐  │  ┌──────────────────────────┐ │
   │  │  堆（Heap）              │  │  │  虚拟机栈                 │ │
   │  │  - 数组、对象实例         │  │  │  ┌────────────────────┐  │ │
   │  │  - GC 的主要战场          │  │  │  │ main 方法栈帧       │  │ │
   │  │  - TLAB、逃逸分析的结果    │  │  │  │ arr = 堆中的地址     │  │ │
   │  └─────────────────────────┘  │  │  └────────────────────┘  │ │
   │                               │  ├──────────────────────────┤ │
   │  ┌─────────────────────────┐  │  │  本地方法栈（native）      │ │
   │  │  方法区（元空间 Metaspace）│  │  ├──────────────────────────┤ │
   │  │  - 类信息、静态变量        │  │  │  程序计数器（PC）          │ │
   │  │  - 常量池、字符串常量池     │  │  └──────────────────────────┘ │
   │  └─────────────────────────┘  │                               │
   └───────────────────────────────┴───────────────────────────────┘

   栈与堆之间的唯一连接就是"引用"：栈上只存地址，不存数据本身，
   arr 里的那个地址指向堆中的数组对象。
```

四条必须补充的说明：

**一、方法区是规范中的概念，实现方式随版本变化。** JDK8 之前用"永久代"（PermGen）实现，受 `-XX:MaxPermSize` 限制；JDK8 起换成**元空间（Metaspace）**，改用手持的**本地内存**，因此不再受堆大小限制，但也可能把整台机器的内存吃光，需要设 `-XX:MaxMetaspaceSize`。

**二、寄存器属于 CPU 硬件，不属于 JVM 运行时数据区。** Java 代码先编译成字节码，由 JVM 解释执行或被 JIT 编译成本地机器码，机器码最终当然要用到 CPU 寄存器——但它不在 JVM 的内存划分里，Java 代码也无法直接操纵。面试里把"PC 寄存器（程序计数器）"和"CPU 寄存器"混为一谈是常见错误，前者是 JVM 的一块内存，后者是芯片里的物理存储单元。

**三、栈与堆的分工决定了哪些数据是共享的。** 每个线程有自己的虚拟机栈，方法里的 `int i`、数组引用 `arr` 都在自己的栈里，**天然不被其他线程看到**；但引用指向的**堆上对象**是所有线程共享的。这就是"线程安全问题"的全部来源——第 17 篇会反复用到这个结论。

**四、局部变量必须初始化，但数组元素会被自动清零。** 因为局部变量在栈帧里只是一块空间，JVM 不会主动擦除；而 `new` 出来的对象在堆上，JVM 保证分配时清零（这是安全要求，否则会读到上一个对象残留的数据）。

# 10. 小结

| 主题 | 一句话要点 |
|------|-----------|
| 是什么 | 引用类型；**类型相同、长度固定、内存连续**；`arr[i]` 靠"首地址 + i × 元素大小"做到 O(1) |
| 声明 | 推荐 `int[] a`；`int a[]` 兼容但不推荐；声明后变量是 `null`，还没指向任何数组 |
| 静态初始化 | `new int[]{1,2,3}`（完整）与 `{1,2,3}`（简化）；**简化格式不能先声明后赋值，也不能当方法实参** |
| 动态初始化 | `new int[5]` 只给长度；长度可以是变量，但**负数运行时会抛 `NegativeArraySizeException`** |
| 默认值 | 数值 `0`、`boolean` `false`、**`char` 是 `'\u0000'`（不是空格）**、引用 `null` |
| 内存模型 | 栈上存**地址**，堆上存**数组对象**；`arr` 本身只占 4/8 字节，与数组长度无关 |
| 打印 | `println(arr)` 得到 `[I@1b6d3586`；要看内容必须 `Arrays.toString` / `deepToString` |
| 数组共享 | `arr2 = arr1` 复制的是地址；改一个另一个也变；要独立必须 `clone`/`copyOf`/`arraycopy`（**浅拷贝**） |
| 传参 | Java 只有值传递，传的是**地址的副本**：改元素生效，重新赋值整个数组不生效 |
| 遍历 | 需要下标、要改元素、要反向 → 普通 for；只读遍历 → 增强 for（**改不到元素**） |
| 二维数组 | "数组的数组"，**每行长度可不同**；遍历第 i 行必须用 `grid[i].length` |
| 越界 | `ArrayIndexOutOfBoundsException`，异常信息含"索引"与"长度"两个数字；只 `new` 外层时是 **NPE** 不是越界 |
| `length` | **属性不是方法**（对比 `String.length()`、`List.size()`）；`final` 不可改；变长只能新建 + 复制 |
| 常用工具 | `Arrays.toString` / `sort` / `copyOf` / `fill` / `binarySearch`（未找到返回 `-(插入点)-1`）、`System.arraycopy` |
| JVM 内存 | 堆与方法区**线程共享**，虚拟机栈、本地方法栈、程序计数器**线程私有**；寄存器是 CPU 硬件，不在其中 |

下一篇（第 6 篇）把"一段逻辑"封装起来，讲方法的定义、重载、可变参数与递归；集合与数组的完整取舍在第 9 篇，排序查找算法的实现细节在第 12 篇，栈与堆在多线程下的共享问题在第 17 篇。
