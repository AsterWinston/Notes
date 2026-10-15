运算符是表达式的骨架，也是 Java 语法里"看着简单、写错最多"的一块。**整除会截断小数、小于 int 的类型参与运算会先提升为 int、复合赋值隐含强制类型转换、`&` 与 `&&` 差在是否会短路**——这几条规律几乎每一个都能对应到线上事故。本篇按"分类总表 → 逐类展开 → 类型转换与溢出 → 位运算 → 优先级"的顺序，把每一类运算符的行为边界讲清楚，每个结论都配可运行示例和真实输出；浮点数的精度问题只在 2.5 点出结论，`BigDecimal` 的完整用法见第 11 篇。

# 1. 运算符概述

## 1.1 什么是运算符

**运算符**（Operator）是对一个或多个**操作数**（Operand）执行运算的符号。按需要的操作数个数，Java 把运算符分成三类：

| 类别 | 操作数个数 | 例子 |
|------|-----------|------|
| 一元运算符 | 1 | `!a`（逻辑非）、`~a`（按位取反）、`-a`（取负）、`++a`（自增） |
| 二元运算符 | 2 | `a + b`、`a && b`、`a << b` |
| 三元运算符 | 3 | `a ? b : c`（Java 中唯一的一个） |

一个表达式求值时有三个属性在起作用：**优先级**（谁先算）、**结合性**（同优先级时从左还是从右算）、**类型提升**（结果是什么类型）。绝大多数运算错误都出在后两个上，而不是"不知道 `*` 比 `+` 先算"。

## 1.2 运算符分类总表

| 类别 | 运算符 | 操作数类型 | 结果类型 | 使用频率 |
|------|--------|-----------|----------|----------|
| 算术 | `+` `-` `*` `/` `%` | 数值（`char` 也算数值） | 提升后的数值类型 | 极高 |
| 自增自减 | `++` `--` | 只能是变量 | 变量本身的类型 | 高 |
| 赋值 | `=` `+=` `-=` `*=` `/=` `%=` | 左值 = 值 | 左侧变量的类型 | 极高 |
| 关系 | `==` `!=` `>` `>=` `<` `<=` | 数值 / 引用 / `boolean` | `boolean` | 极高 |
| 逻辑 | `&&` `\|\|` `!` `^` | `boolean` | `boolean` | 高 |
| 三元 | `?:` | 条件为 `boolean` | 两个分支的统一类型 | 中 |
| 位运算 | `&` `\|` `^` `~` `<<` `>>` `>>>` | 整数（`byte`/`short`/`int`/`long`/`char`） | 提升后的整数类型 | 低（框架源码、算法题常见） |

Java 还有 `instanceof`（类型判断）、`->`（Lambda）、`::`（方法引用）等运算符，分别在第 10、12、13 篇展开，本篇只覆盖上表。

> **注意**：`&`、`|`、`^` 三个符号在 Java 里是**重载**的——操作数是 `boolean` 时做逻辑运算，操作数是整数时做位运算。`+` 同样是重载的（数值加法、字符串拼接）。这是 Java 中仅有的几个运算符重载，语言层面不允许用户自定义重载。

# 2. 算术运算符

## 2.1 除法 `/`：整数相除会截断

**两个操作数都是整数类型时，`/` 执行整除**，小数部分被直接丢弃（向零截断），既不是四舍五入，也不是向下取整。只要有一个操作数是浮点类型，就变成常规的浮点除法。

```java
public class DivisionDemo {
    public static void main(String[] args) {
        System.out.println(5 / 2);            // 2     两个 int，整除，小数被砍掉
        System.out.println(5.0 / 2);          // 2.5   double 参与，提升为 double 运算
        System.out.println(5 / 2.0);          // 2.5   同上
        System.out.println(5 / 2.0f);         // 2.5   float 参与，结果是 float
        System.out.println((double) 5 / 2);   // 2.5   先强转一个操作数，结果同样是 double
        System.out.println(-5 / 2);           // -2    向零截断，不是 -2.5 也不是 -3
    }
}
```

**这是本章最容易造成事故的一条规律**，因为它出错时悄无声息：

```java
int sum = 15;
int count = 4;

System.out.println(sum / count);                 // 3      两个 int 整除
double avgWrong = sum / count;                   // 3.0    先整除，再赋给 double —— 已经救不回来了
double avgRight = (double) sum / count;          // 3.75   先提升为 double 再除
System.out.println(avgWrong + " vs " + avgRight); // 3.0 vs 3.75

int percent = 1 / 3 * 100;                       // 0      先算 1/3 = 0，再乘 100 还是 0
System.out.println(percent + "%");               // 0%     想要 33% 必须写 1 * 100 / 3
```

> **陷阱**：`double avg = sum / count;` 里的除法**先于赋值发生**。赋值运算符的优先级最低，所以两个 `int` 的整除早就完成了，再把结果 3 提升成 `double` 得到 3.0。正确写法是**让浮点类型尽早参与运算**：`(double) sum / count` 或 `sum * 1.0 / count`。求平均值、算百分比、算比率时，这是头号错误。

## 2.2 取模 `%`：结果符号跟随被除数

`a % b` 的绝对值等于 `|a| % |b|`，**符号与左操作数（被除数）相同**。这是 Java 与 C 的"截断取余"，与 Python 的"取模"（结果符号跟随除数）不同，跨语言迁移时要注意。

```java
System.out.println(7 % 3);      // 1     绝对值 1，符号跟随 7
System.out.println(-7 % 3);     // -1    绝对值仍是 1，符号跟随 -7
System.out.println(7 % -3);     // 1     符号跟随 7
System.out.println(-7 % -3);    // -1
System.out.println(5.5 % 2);    // 1.5   浮点数也能取模（结果 = 5.5 - 2 * 2）
System.out.println(10 % 3.0);   // 1.0   有 double 参与就返回 double
```

三个典型用途：

1. **判断整除**: `a % b == 0`；
2. **判断奇偶**: `n % 2 == 0`（偶数）；注意判断奇数**不能写** `n % 2 == 1`，因为 `-3 % 2` 得 `-1`；
3. **循环下标回绕**: `arr[i % arr.length]` 可以让下标在 `i` 不断增大时自动转圈。

```java
// 取模实现循环队列的下标回绕
String[] week = {"周一", "周二", "周三"};
for (int i = 0; i < 7; i++) {
    System.out.print(week[i % week.length] + " ");   // 周一 周二 周三 周一 周二 周三 周一
}
```

> **注意**：判断奇偶最稳的写法是 `(n & 1) == 1`（见 9.3），它对负数同样正确；用 `n % 2 != 0` 也可以，但 `n % 2 == 1` 是错的。

## 2.3 减法 `-` 与乘法 `*`

`-` 既是二元减法，也是一元取负（`-a`）。一元形式与二元形式写法相同，靠**操作数个数**区分：

```java
int a = 5;
System.out.println(a - 3);      // 2     二元减法
System.out.println(-a);         // -5    一元取负
System.out.println(- -a);       // 5     两个一元运算符叠加，合法但别这么写
System.out.println(3 - -2);     // 5     3 减 -2
```

`*` 只有二元形式。**整型乘法同样会溢出**，而且溢出后不会报错：

```java
// 经典溢出：30 天有多少毫秒（正确值是 2_592_000_000，超过 int 上限）
int millis = 1000 * 60 * 60 * 24 * 30;
System.out.println(millis);                 // -1702967296   静默溢出，结果完全不可用

long correct = 1000L * 60 * 60 * 24 * 30;   // 只要让第一个数带上 L，整个表达式就按 long 运算
System.out.println(correct);                // 2592000000

long tooLate = 1000 * 60 * 60 * 24 * 30;    // 先按 int 溢出，再赋给 long，已经来不及
System.out.println(tooLate);                // -1702967296
```

> **陷阱**：`long x = int 运算` 这种写法救不了溢出——类型提升发生在**赋值之前**，等到赋值时数值已经错了。要让整个表达式按 `long` 计算，必须让**参与运算的第一个操作数**就是 `long`：`1000L * 60 * ...`。这个坑在"毫秒转天""字节转 GB"这类计算里极其常见。

## 2.4 加号 `+` 的三重身份

`+` 是 Java 里含义最多的运算符，它根据操作数类型分别解释为：

| 左操作数 / 右操作数 | 行为 | 结果类型 |
|---------------------|------|----------|
| 数值 / 数值 | 数值加法 | 提升后的数值类型 |
| `char` / 数值 | 按 Unicode 码点做数值加法 | `int` |
| `String` / 任意类型 | **字符串拼接**（右边会转成字符串） | `String` |

```java
public class PlusDemo {
    public static void main(String[] args) {
        System.out.println(10 + 20);              // 30        数值加法
        System.out.println('a' + 1);              // 98        char 提升为 int，按码点相加
        System.out.println((char) ('a' + 1));     // b         再强转回 char
        System.out.println('a' + "1");            // a1        String 参与 → 拼接

        System.out.println("Hello" + 10 + 20);    // Hello1020 从左到右：10 和 20 都成了文本
        System.out.println(10 + 20 + "Hello");    // 30Hello   先算 10+20=30，再拼接
        System.out.println("Hello" + (10 + 20));  // Hello30   括号改变结合顺序

        System.out.println("" + 1 + 2);           // 12        加空串是最常见的转字符串手段
        System.out.println(1 + 2 + "");           // 3         注意 1+2 先算

        Object obj = null;
        System.out.println("值: " + obj);         // 值: null  不抛异常，String.valueOf 处理了 null
    }
}
```

**关键规律：`+` 是左结合的，从左往右逐个拼接。** 一旦表达式里出现过 `String`，它右边的所有 `+` 都变成字符串拼接，直到被括号打断。这是"为什么输出是 1020 而不是 30"的完整解释——调试日志里输出拼接错误，多半就是这里。

`"Hello" + null` 不会抛 `NullPointerException`，因为编译器把它编译成 `new StringBuilder().append("Hello").append(obj).toString()`，而 `append(Object)` 内部调用了 `String.valueOf(obj)`，该方法对 `null` 返回字符串 `"null"`。但如果调用的是 `obj.toString()`，就会直接 NPE。

## 2.5 浮点数的精度问题

```java
System.out.println(0.1 + 0.2);          // 0.30000000000000004
System.out.println(0.1 + 0.2 == 0.3);   // false
System.out.println(1.0 - 0.9);          // 0.09999999999999998
System.out.println(0.1f + 0.2f == 0.3f);// false   float 同样不精确
```

原因在 IEEE 754 标准：`float` 和 `double` 用**二进制**表示小数，而 0.1、0.2 在二进制里是无限循环小数（就像十进制写不出精确的 1/3），只能存一个近似值，参与运算后误差被放大。

**正确的处理方式有三种**：

```java
// 方式一：判断相等时用误差范围（epsilon）
double a = 0.1 + 0.2, b = 0.3;
System.out.println(Math.abs(a - b) < 1e-9);   // true  1e-9 是允许的误差

// 方式二：四舍五入到指定位数后比较
System.out.println(Math.round(a * 100) == Math.round(b * 100));  // true

// 方式三：涉及金额一律用 BigDecimal（且必须用字符串构造）
System.out.println(new BigDecimal("0.1").add(new BigDecimal("0.2")));  // 0.3
System.out.println(new BigDecimal(0.1).add(new BigDecimal(0.2)));      // 0.3000000000000000166...
```

> **设计考量**：`new BigDecimal(0.1)` 会把 `double` 的**误差原样带进来**，所以构造 `BigDecimal` 时要传字符串（`new BigDecimal("0.1")`）或整数（`BigDecimal.valueOf(10)`）。**凡是"钱、税率、坐标、比例"这类要求精确的数值，一律不要用浮点**。`BigDecimal` 的构造、精度、舍入模式与除法除不尽的处理，见第 11 篇。

## 2.6 算术运算的共同规则

所有二元算术运算符在求值前都会先做**二进制数值提升**：

1. 只要有一个操作数是 `double`，另一个就提升为 `double`，结果 `double`；
2. 否则只要有 `float`，另一个提升为 `float`，结果 `float`；
3. 否则只要有 `long`，另一个提升为 `long`，结果 `long`；
4. 否则**两边都提升为 `int`**，结果 `int`——这就是 `byte + byte` 得到 `int` 的原因（见 3.1）。

# 3. 类型转换

## 3.1 隐式转换（自动类型提升）

隐式转换发生在两种场合：**赋值时**（小范围赋给大范围）和**运算时**（按上面的提升规则统一类型）。

### 3.1.1 取值范围排序

```text
   ┌──────┐   ┌───────┐   ┌─────┐   ┌──────┐   ┌───────┐   ┌────────┐
   │ byte │──▶│ short │──▶│ int │──▶│ long │──▶│ float │──▶│ double │
   │ 1 字节│   │ 2 字节 │   │4 字节│   │8 字节 │   │4 字节  │   │8 字节   │
   └──────┘   └───────┘   └─────┘   └──────┘   └───────┘   └────────┘

   ┌──────┐
   │ char │──────────────────▶ 只能转入 int、long、float、double
   │2 字节 │                     （char 无符号，与 byte 不能隐式互转）
   └──────┘
```

注意 `long`（8 字节）到 `float`（4 字节）是**自动**的——Java 看的是**取值范围**而不是字节数：`float` 的指数位能表示到约 3.4×10^38，远超 `long` 的 9.2×10^18，代价是精度只有 24 位有效数字（见 3.2.4）。

### 3.1.2 byte、short、char 一律先提升为 int

```java
public class PromotionDemo {
    public static void main(String[] args) {
        byte b1 = 10;
        byte b2 = 20;
        // byte b3 = b1 + b2;         // ✗ 编译错误：b1 + b2 的结果已经是 int
        int b3 = b1 + b2;             // ✓ 30
        byte b4 = (byte) (b1 + b2);   // ✓ 想放回 byte 必须强转

        short s1 = 100, s2 = 200;
        // short s3 = s1 + s2;        // ✗ 同样报错
        int s3 = s1 + s2;             // ✓ 300

        char c = 'a';
        System.out.println(c + 1);            // 98    char 被提升为 int
        System.out.println((char) (c + 1));   // b     再转回 char
        // char c2 = c + 1;                  // ✗ 编译错误
    }
}
```

> **注意**：这是 Java 特有的一条规定。根本原因是 **JVM 的指令集只对 `int` 及更宽的类型提供算术指令**，没有 `baload` 之外的字节级加法，所以小于 `int` 的类型必须先提升为 `int` 才能参与运算。这条规律同时解释了两件事：为什么 `byte b = 1; b + b` 得到 `int`，以及为什么 `s += 1` 合法而 `s = s + 1` 报错（见 5.2）。

### 3.1.3 隐式转换总表

| 目标类型 | 可直接接收的类型 | 说明 |
|----------|-----------------|------|
| `short` | `byte` | |
| `int` | `byte`、`short`、`char` | |
| `long` | `byte`、`short`、`char`、`int` | |
| `float` | `byte`、`short`、`char`、`int`、`long` | 可能丢精度 |
| `double` | 以上全部 | |
| `char` | 无 | 只能靠强转 |

## 3.2 强制类型转换

格式：`(目标类型) 表达式`。强制转换告诉编译器"我知道可能有损失，照做"。它由**开发者主动写**，因此编译器不再做范围检查。

### 3.2.1 浮点转整数：直接截断

```java
System.out.println((int) 3.99);       // 3    砍掉小数，不是四舍五入
System.out.println((int) -3.99);      // -3   向零截断（不是 -4）
System.out.println((int) 3.49);       // 3
System.out.println((int) (3.49 + 0.5)); // 3  想四舍五入得自己加 0.5
System.out.println((int) (3.99 + 0.5)); // 4  或者直接用 Math.round(3.99)
System.out.println(Math.round(3.5));    // 4  标准四舍五入，返回 long
System.out.println((long) 1e18);      // 1000000000000000000

// 超出目标类型范围时：饱和，而不是回绕
System.out.println((int) 1e20);       // 2147483647   Integer.MAX_VALUE
System.out.println((int) -1e20);      // -2147483648  Integer.MIN_VALUE
System.out.println(Double.NaN);       // NaN
System.out.println((int) Double.NaN); // 0            NaN 转整数得 0
```

> **注意**：浮点转整数有两种"越界"表现，别搞混——**小数部分**是直接截断，**整数部分超出范围**则饱和到 `MAX_VALUE`/`MIN_VALUE`（`NaN` 变成 0）。这与下面整数之间的窄化转换（回绕）是两套完全不同的规则。

### 3.2.2 整数窄化：砍掉高位（回绕）

窄化转换保留**低 N 位**（N 是目标类型的位数），丢弃高位。

```text
   (byte) 130 的完整过程

   130 的 int 补码:   0000 0000 0000 0000 0000 0000 1000 0010
                      丢弃高 24 位  ▼
   byte 只留低 8 位:                                1000 0010
                                                     │
                            最高位是 1 → 这是负数      ▼
                                                      -126
```

```java
System.out.println((byte) 130);      // -126   130 - 256
System.out.println((byte) 300);      // 44     300 - 256（低 8 位 0010 1100 = 44）
System.out.println((byte) -130);     // 126    -130 + 256
System.out.println((short) 40000);   // -25536 40000 - 65536
System.out.println((int) 3_000_000_000L); // -1294967296  3e9 - 2^32
```

记忆方法：窄化就是"**减 2 的 N 次方，直到落进目标范围**"。`(byte)130` 减 256 得 -126；`(short)40000` 减 65536 得 -25536。

### 3.2.3 为什么 `byte b = 130;` 会报错

```java
byte b1 = 130;           // ✗ 编译错误：不兼容的类型: 从 int 转换到 byte 可能会有损失
byte b2 = (byte) 130;    // ✓ 编译通过，运行结果为 -126
byte b3 = 100;           // ✓ 合法，100 在 byte 范围内
byte b4 = 10 + 20;       // ✓ 合法，30 也在范围内
byte b5 = 127;           // ✓ 合法，正好是 byte 上限
```

原因：`130` 是一个 `int` **字面量**，它的值超出 `byte` 的范围 `[-128, 127]`。编译器在**编译期**就能算出这个赋值必然丢数据，于是直接报错——**这是编译错误，不是运行时异常**，代码根本跑不起来。而 `(byte) 130` 是显式强转，等于开发者签字确认"我知道会丢"，编译器就放行了。

编译器对**常量表达式**会做"折叠后检查"，所以下面这些也是合法的：

```java
final int MAX = 127;
byte b6 = MAX;           // ✓ 合法：常量在编译期折叠，值在范围内
byte b7 = 100 + 20 + 7;  // ✓ 合法：常量折叠得到 127

int notFinal = 100;
// byte b8 = notFinal;   // ✗ 编译错误：编译器不会为变量做范围推断
```

> **陷阱**：`byte b = 128 - 1;` 合法（折叠得 127），但 `byte b = x - 1;`（`x` 是 `int` 变量）报错。判断标准是：**这个值在编译期能不能算出来**。同一个道理，`byte b = someMethod();` 永远报错，哪怕方法实际返回 10。

### 3.2.4 精度丢失是双向的

不要以为只有"大范围转小范围"才丢数据——`int` 转 `float` 也会丢，而且**不报错**，因为编译器认为 `float` 的范围更大：

```java
int big = 16777217;                   // 2^24 + 1
float f = big;                        // 隐式转换，编译器不拦
System.out.println(f);                // 1.6777216E7       末尾的 1 没了
System.out.println((int) f == big);   // false  16777216 != 16777217
System.out.println((int) f);          // 16777216

long huge = 1234567890123456789L;
System.out.println((float) huge);     // 1.23456794E18     精度只剩 7 位左右
System.out.println((double) huge);    // 1.23456789012345677E18   double 也只有 15~16 位有效数字
```

`float` 有 24 位有效数字（尾数 23 位 + 隐含的 1），所以大于 2^24 = 16777216 的整数无法逐个表示；`double` 有 53 位，大于 2^53 之后同样会丢。**这就是为什么 ID、订单号、金额不能用浮点存储**——它们动辄超过 16 位。

## 3.3 转换相关的其它规则

| 规则 | 说明 |
|------|------|
| `boolean` 不能与任何类型互转 | `(int) true` 编译错误；C 语言里 `true` 是 1，Java 里不是 |
| `char` 与 `byte`/`short` 之间必须强转 | `char` 无符号 `[0, 65535]`，`byte` 有符号 `[-128, 127]`，范围不重叠 |
| `char` 与 `int` 之间可以隐式/显式转换 | `int i = 'a';` 合法（隐式）；`char c = (char) 97;` 必须强转 |
| 引用类型的转换 | 向上转型自动、向下转型需强转且可能抛 `ClassCastException`，见第 10 篇 |

```java
System.out.println((int) 'A');        // 65     char → int 可以显式
System.out.println((char) 65);        // A
System.out.println((char) (0x4E2D));  // 中     Unicode 码点转字符
System.out.println((int) '中');        // 20013
// System.out.println((int) true);    // ✗ 编译错误：boolean 不能转 int
```

# 4. 自增自减运算符

## 4.1 前置与后置的语义

`++` 和 `--` 都有前置、后置两种形式，它们的区别只在**表达式的值**上，对变量本身的作用是一样的：

| 表达式 | 对变量的作用 | 表达式的值 |
|--------|-------------|-----------|
| `++a`（前置自增） | `a = a + 1` | 自增**之后**的新值 |
| `a++`（后置自增） | `a = a + 1` | 自增**之前**的原值 |
| `--a`（前置自减） | `a = a - 1` | 自减**之后**的新值 |
| `a--`（后置自减） | `a = a - 1` | 自减**之前**的原值 |

```java
public class IncDecDemo {
    public static void main(String[] args) {
        int a = 5;
        int b = ++a;                       // 先自增，再用新值
        System.out.println("a=" + a + ", b=" + b);   // a=6, b=6

        int c = 5;
        int d = c++;                       // 先用原值，再自增
        System.out.println("c=" + c + ", d=" + d);   // c=6, d=5

        int e = 5;
        int f = --e;
        System.out.println("e=" + e + ", f=" + f);   // e=4, f=4

        int g = 5;
        int h = g--;
        System.out.println("g=" + g + ", h=" + h);   // g=4, h=5
    }
}
```

## 4.2 单独成句时两者没有区别

自增自减的操作数必须是**变量**（可修改的左值），不能是字面量或表达式：

```java
int i = 1;
i++;              // ✓
++i;              // ✓ 与 i++ 在单独成句时完全等价
// 5++;           // ✗ 编译错误：意外的类型
// (i + 1)++;     // ✗ 编译错误
```

`i++;` 和 `++i;` 单独成句时**编译出的字节码完全相同**（都是 `iinc` 指令），所以那句广为流传的"前置自增比后置自增快"在 Java 里**不成立**——那是 C++ 的说法（C++ 里后置版本要为对象保存一份副本）。Java 里选哪个只取决于可读性。

## 4.3 经典陷阱

### 4.3.1 `i = i++` 之后 i 不变

```java
int i = 5;
i = i++;
System.out.println(i);   // 5 —— 不是 6
```

拆解成三步等价代码：

```java
int tmp = i;      // 后置自增先"返回"原值 5，可以理解为存在临时变量里
i = i + 1;        // 此时 i 已经是 6
i = tmp;          // 赋值语句把临时变量的 5 写回 i，把 6 覆盖掉
System.out.println(i);   // 5
```

同理 `i = ++i;` 的结果是 6（前置版本返回的是新值 6，赋回去还是 6），`i = i--;` 得到原值。这类写法**只在面试题里出现**，实际代码中毫无意义。

### 4.3.2 `a++ + ++a`

```java
int a = 5;
int r = a++ + ++a;
System.out.println("a=" + a + ", r=" + r);   // a=7, r=12
```

求值顺序严格**从左到右**：`a++` 返回 5（同时 `a` 变成 6），`++a` 先把 `a` 变成 7 再返回 7，所以 `5 + 7 = 12`，最终 `a = 7`。

### 4.3.3 溢出：byte 自增会回绕

```java
byte b = 127;
b++;                        // 等价于 b = (byte)(b + 1)
System.out.println(b);      // -128

// 但下面这行是编译错误：
// b = b + 1;               // ✗ b + 1 是 int，不能赋给 byte
```

`b++` 是合法的而 `b = b + 1` 不合法，是因为自增运算符**自带隐式强转**——它被定义为"对变量本身做加一并存回变量"，不经过"表达式提升结果类型"这一步。这与复合赋值的规则一致（见 5.2）。

### 4.3.4 循环中的下标混用

```java
int[] arr = {10, 20, 30};
int i = 0;
System.out.println(arr[i++]);   // 10   取的是 i 的原值 0，之后 i 变成 1
System.out.println(arr[++i]);   // 30   i 先变成 2，再取值 → arr[2]
System.out.println(i);          // 2
```

> **最佳实践**：**不要在同一个表达式里对同一个变量既自增又使用**。`i = i++`、`f(i++, i++)`、`arr[i++] = i` 这类写法虽然结果可预测（Java 的求值顺序有明确规定），但可读性极差，等于给自己和同事埋雷。**拆成两行代码**，一行只做一件事。真正适合自增的场景只有两种：循环的步进语句（`for (int i = 0; ...; i++)`）和"取出当前值后前进"的迭代器式写法。

# 5. 赋值运算符

## 5.1 基本赋值 `=`

赋值运算符把右侧的值写入左侧变量，**要求类型兼容**：

```java
int a = 10;
a = 20;                       // ✓ 同类型
long l = 10;                  // ✓ int 字面量可以隐式提升为 long
double d = 10;                // ✓ 同理
// int x = 10L;               // ✗ 编译错误：long 不能隐式窄化为 int
// a = 3.14;                  // ✗ 编译错误：double 不能赋给 int
a = (int) 3.14;               // ✓ 强转后可以
char c = 65;                  // ✓ 字面量在 char 范围内
```

赋值表达式**本身也有值**（就是被赋的那个值），所以可以连写。这也是赋值运算符**右结合**的原因：

```java
int x, y, z;
x = y = z = 5;                // 等价于 x = (y = (z = 5))
System.out.println(x + "," + y + "," + z);   // 5,5,5
```

## 5.2 复合赋值运算符与隐式强转

复合赋值共 11 个：`+=` `-=` `*=` `/=` `%=`，以及位运算版本 `&=` `|=` `^=` `<<=` `>>=` `>>>=`。

**核心特性：复合赋值隐含了一次强制类型转换。** Java 语言规范定义的等价关系是：

```text
   E1 op= E2   完全等价于   E1 = (E1 的类型) ((E1) op (E2))
```

也就是说，`s += 1` 不是 `s = s + 1`，而是 `s = (short)(s + 1)`。这一条解释了本节所有现象：

```java
short s = 1;
// s = s + 1;      // ✗ 编译错误：s + 1 是 int
s += 1;            // ✓ 等价于 s = (short)(s + 1)
System.out.println(s);   // 2

int i = 5;
i *= 2.5;                 // 等价于 i = (int)(i * 2.5)
System.out.println(i);    // 12 —— 12.5 被截断，静默丢精度

byte b = 10;
b += 250;                 // 等价于 b = (byte)(10 + 250) = (byte)260
System.out.println(b);    // 4    260 - 256

short m = 32767;
m += 1;                   // 等价于 m = (short)(32767 + 1)
System.out.println(m);    // -32768   回绕

int n = 1;
n /= 2;
System.out.println(n);    // 0    整除
```

> **陷阱**：复合赋值会让编译器"闭嘴"，把本该报错的窄化赋值变成**静默的溢出或截断**。`i *= 2.5`、`b += 250`、`m += 1` 三行都能编译通过，但结果全在意料之外。判断标准很简单：**只要 `op=` 右侧的类型与左侧不同，就应该多想一秒**。需要精确控制时写出完整形式 `i = (int)(i * 2.5)`，让意图显式暴露。

### 5.2.1 `+=` 用在字符串上

`String` 是不可变类，`s += "x"` 每次都创建一个新对象（详见第 8 篇）：

```java
String s = "a";
s += 1;                  // 等价于 s = s + 1，结果是 "a1"
System.out.println(s);   // a1

s += 2 + 3;              // 右侧先算 2+3=5（算术优先于拼接），再拼接
System.out.println(s);   // a15

s += 2 + "" + 3;         // 一旦出现字符串，后面全变成拼接
System.out.println(s);   // a1523
```

> **注意**：在循环里用 `+=` 拼字符串是**性能陷阱**——每次循环都新建一个 `String` 对象，复杂度从 O(n) 退化到 O(n^2)。循环内拼接应该用 `StringBuilder`（第 8 篇）。

### 5.2.2 效率上的说明

`i += 1`、`i++`、`i = i + 1` 对 `int` 而言**编译结果一致**，不存在性能差异，现代 JIT 也不会因为写法不同而生成不同的机器码。选哪种只看可读性。

# 6. 关系运算符

关系运算符用于比较大小或判断相等，**结果永远是 `boolean`**（不像 C 那样返回 0/1）。

| 运算符 | 含义 | 可比较的类型 |
|--------|------|-------------|
| `==` | 等于 | 任意类型（数值、`char`、`boolean`、引用） |
| `!=` | 不等于 | 同上 |
| `>` | 大于 | 只能是比较类类型（数值 + `char`），`boolean` 不行 |
| `>=` | 大于等于 | 同上 |
| `<` | 小于 | 同上 |
| `<=` | 小于等于 | 同上 |

```java
System.out.println(5 > 3);           // true
System.out.println(5 >= 5);          // true
System.out.println(5 != 3);          // true
System.out.println('a' < 'b');       // true   按 Unicode 码点比较：97 < 98
System.out.println('a' == 97);       // true   char 提升为 int 后比较
System.out.println(1 == 1.0);        // true   int 提升为 double 后比较
// System.out.println(true > false); // ✗ 编译错误：boolean 不能比大小
// System.out.println("a" < "b");    // ✗ 编译错误：引用类型不能比大小
```

## 6.1 两条必须记住的规则

**规则一：关系运算符的优先级低于算术运算符。**

```java
System.out.println(1 + 2 > 2);       // true   先算 1+2=3，再比 3 > 2
System.out.println(1 > 2 + 3);       // false  先算 2+3=5，再比 1 > 5
System.out.println(1 + 1 == 2);      // true   算术 → 相等比较
```

**规则二：`==` 和 `=` 是两个不同的运算符。**

```java
int a = 5;
// if (a = 5) { }        // ✗ 编译错误：int 不能当 boolean 用（Java 在这一点上保护了你）
if (a == 5) { }          // ✓

boolean flag = true;
if (flag = false) { }    // ✗ 能编译！把 false 赋给 flag 后判断，逻辑整个反了
if (flag == false) { }   // ✓ 但更推荐下面这种
if (!flag) { }           // ✓ 最好的写法
```

对 `int` 的检查是 Java 相对 C 的一大改进；但对 `boolean` 变量**没有这层保护**，`if (flag = true)` 依然合法（且恒为真），是真实的坑。

## 6.2 引用类型的 `==` 比较的是地址

```java
String s1 = new String("hi");
String s2 = new String("hi");
System.out.println(s1 == s2);         // false  两个不同的对象，地址不同
System.out.println(s1.equals(s2));    // true   内容相同
System.out.println(s1.equals(null));  // false  equals 对参数为 null 是安全的
// System.out.println(null.equals(s1)); // ✗ NPE：调用方为 null
```

比较**内容**用 `equals()`，比较**是不是同一个对象**用 `==`。字符串常量池、"`String s = "hi"` 和 `new String("hi")` 有什么区别"这些问题在第 8 篇展开。对象默认继承的 `equals()` 就是 `==`，需要重写才能比较内容（第 7、10 篇）。

# 7. 逻辑运算符

逻辑运算符的**操作数必须是 `boolean`**（`&`、`|`、`^` 用于整数时是位运算，见第 9 节）。

| 运算符 | 名称 | 是否短路 | 规则 |
|--------|------|----------|------|
| `&` | 逻辑与（非短路） | 否 | 两边都求值，全为 `true` 才 `true` |
| `&&` | 短路与 | **是** | 左边为 `false` 则右边不执行 |
| `\|` | 逻辑或（非短路） | 否 | 两边都求值，有一个 `true` 就 `true` |
| `\|\|` | 短路或 | **是** | 左边为 `true` 则右边不执行 |
| `^` | 逻辑异或 | 否 | 两边**不同**才 `true` |
| `!` | 逻辑非 | —— | 一元运算符，取反 |

## 7.1 `&&` 与 `&` 的区别：短路

**短路（short-circuit）**指左侧结果已经能决定整个表达式的值时，右侧**不再求值**。

```java
public class AndDemo {
    public static void main(String[] args) {
        int i = 0;

        // ✓ 短路与：左侧 false，右侧的 10 / i 根本不执行
        boolean r1 = (i != 0) && (10 / i > 1);
        System.out.println(r1);                    // false，没有除零异常

        // ✗ 非短路与：右侧照样执行，直接除零
        // boolean r2 = (i != 0) & (10 / i > 1);   // ArithmeticException: / by zero

        // 用副作用观察右侧到底有没有执行
        int a = 0;
        boolean r3 = (a++ > 0) && (a++ > 0);       // 左侧 false，右侧跳过
        System.out.println("a=" + a + ", r3=" + r3);   // a=1, r3=false

        int b = 0;
        boolean r4 = (b++ > 0) & (b++ > 0);        // 右侧仍然执行
        System.out.println("b=" + b + ", r4=" + r4);   // b=2, r4=false
    }
}
```

`a=1` 和 `b=2` 的差别就是短路的直接证据。

真实代码里 `&&` 有 90% 的用途是**空值保护**和**边界保护**：

```java
// ✓ 推荐：先判空再调用，逐级短路
if (user != null && user.getName() != null && user.getName().startsWith("张")) {
    System.out.println("姓张的用户");
}

// ✗ 写成 & 就是自找麻烦：user 为 null 时右侧照跑，直接 NPE
// if (user != null & user.getName().startsWith("张")) { ... }
```

## 7.2 `||` 与 `|` 的区别

```java
String s = null;

// ✓ 短路或：左侧已经是 true，右侧完全不执行，不会 NPE
boolean r1 = (s == null) || s.isEmpty();
System.out.println(r1);                    // true

int i = 1;
// ✓ 短路或：左侧 true，右侧的除零被跳过
System.out.println((i > 0) || (10 / 0 > 1));   // true
// ✗ 非短路或：右边照跑 → ArithmeticException
// System.out.println((i > 0) | (10 / 0 > 1));
```

> **注意**：短路的顺序很关键。`(s == null) || s.isEmpty()` 不会 NPE，但把两个条件换一下写成 `s.isEmpty() || s == null`，就会**先 NPE**。写复合条件时，"能判断安全性的条件"必须放在前面。

## 7.3 `^` 逻辑异或与 `!` 逻辑非

`^` 在 `boolean` 上表示"两个操作数**不同**才为 `true`"，相同为 `false`：

```java
System.out.println(true ^ true);     // false
System.out.println(true ^ false);    // true
System.out.println(false ^ true);    // true
System.out.println(false ^ false);   // false
```

在业务上，`^` 表达的是"**恰好满足一个**"的语义：

```java
boolean hasCoupon = true;
boolean hasDiscount = true;

if (hasCoupon || hasDiscount) { }    // 允许两个都成立
if (hasCoupon ^ hasDiscount) { }     // 要求恰好只有一个成立
```

`!` 是一元取反，优先级很高：

```java
System.out.println(!true);              // false
System.out.println(!(5 > 3));           // false
System.out.println(!!true);             // true   合法但别这么写
// 注意不要和 != 混：!= 是二元比较，! 是一元取反
```

## 7.4 什么时候必须用非短路的 `&` 和 `|`

真实场景只有两类：

1. **需要右侧的副作用**——两个方法都必须被调用，无论左边结果如何：
```java
// 两个校验都要跑（都要记录日志/都要清理资源），不能用 &&
boolean ok = validateFormat(input) & validateRange(input);
```
这种写法**通常说明代码该拆开**，写成两行更清楚。

2. **位运算**——这才是 `&`、`|`、`^` 的主场（第 9 节）。它们的优先级也和其他逻辑运算符不同：`&` 高于 `^`，`^` 高于 `|`。

> **最佳实践**：布尔逻辑一律用 `&&` 和 `||`。只有明确需要"两边都执行"时才用 `&`、`|`，并且**加注释说明原因**——否则下一个读代码的人会以为是打错了字，顺手改成 `&&`，把逻辑改坏。

# 8. 三元运算符

Java 中唯一的**三元运算符**，是 `if-else` 的**表达式**形式。

格式：`条件表达式 ? 值1 : 值2`，条件为 `true` 时取值1，否则取值2。

```java
int a = 10, b = 20;
int max = a > b ? a : b;
System.out.println(max);                    // 20

String tip = a > b ? "a 大" : "b 大";
System.out.println(tip);                    // b 大

// 嵌套：a、b、c 中取最大
int c = 30;
int maxOfThree = a > b ? (a > c ? a : c) : (b > c ? b : c);
System.out.println(maxOfThree);             // 30
```

## 8.1 三条语法规则

1. **条件表达式必须是 `boolean`**，不能像 C 那样用非 0 值代表真：
```java
// int r = 1 ? 10 : 20;   // ✗ 编译错误：int 不能当 boolean
int r = (1 > 0) ? 10 : 20;  // ✓
```
2. **两个分支的类型会被统一**，编译器做二进制数值提升：
```java
Object o = true ? 1 : 2.0;                                  // 结果是 Double，不是 Integer
System.out.println(o);                                      // 1.0  注意是 1.0
System.out.println(((Object) (true ? 1 : 2.0)).getClass()); // class java.lang.Double
```
3. **可以嵌套，但嵌套两层以上就该改成 `if-else` 或 `switch`**——可读性会断崖式下降。

## 8.2 自动拆箱的经典 NPE

这是三元运算符最容易踩的坑，也是 JDK8 引入 `Optional` 时被大量吐槽的问题：

```java
Map<String, Integer> map = new HashMap<>();
Integer v = map.get("notExist");        // null

// ✗ 看起来"已经判空了"，其实会抛 NPE
// int n = v == null ? 0 : v;

// ✓ 正确写法一：显式把 0 写成包装类型
int n1 = (v != null) ? v : Integer.valueOf(0);
System.out.println(n1);                 // 0

// ✓ 正确写法二：用 Map 自带方法（推荐）
int n2 = map.getOrDefault("notExist", 0);
System.out.println(n2);                 // 0
```

**为什么 `v == null ? 0 : v` 会 NPE？** 三元表达式要求两个分支类型统一：`0` 是 `int`，`v` 是 `Integer`，编译器为了让两者同类型，会对 `v` 做**自动拆箱**（`v.intValue()`）。而这个拆箱动作发生在**类型统一阶段**，早于条件判断——所以即使 `v` 为 `null` 时走的是 `0` 这一支，`v` 的拆箱也已经执行过了，结果是 `NullPointerException`。

> **陷阱**：`条件 ? 基本类型 : 包装类型` 这种混搭会触发**提前拆箱**。凡是条件里出现包装类型，两个分支就都写成包装类型，或者干脆用 `if-else`。这个坑在 Stream 的 `reduce`、`Collectors` 的返回值处理里反复出现。

## 8.3 三元与 if-else 的选择

| 场景 | 推荐 | 理由 |
|------|------|------|
| 结果要**赋给变量**或**参与运算** | 三元 | 它本身是表达式，可以直接嵌进语句 |
| 分支里要做**多件事**（打印、调用、赋值多个变量） | `if-else` | 三元的分支只能是单个表达式 |
| **嵌套超过两层** | `if-else` / `switch` | 三元嵌套几乎无法阅读 |
| 只是给一个返回值做兜底 | 三元或 `Optional` | 注意 8.2 的拆箱陷阱 |

# 9. 位运算

位运算直接操作整数的**二进制补码位**，只能用于 `byte`、`short`、`int`、`long`、`char`（`boolean` 不行）。这是本篇最底层的一节，也是**框架源码与算法题里出现频率最高**的一部分。

## 9.1 四种基础位运算

| 运算符 | 名称 | 规则 | 记忆 |
|--------|------|------|------|
| `&` | 按位与 | 两位都是 1 才得 1 | 有 0 则 0，可用来"取位/清零" |
| `\|` | 按位或 | 有一位是 1 就得 1 | 有 1 则 1，可用来"置位" |
| `^` | 按位异或 | 两位不同得 1 | 不同为 1，可用来"翻转位" |
| `~` | 按位取反 | 0 变 1，1 变 0 | 一元运算符 |

以 6（`0000 0110`）与 3（`0000 0011`）为例：

```text
   6         0000 0000 0000 0000 0000 0000 0000 0110
   3         0000 0000 0000 0000 0000 0000 0000 0011

   6 & 3     0000 0000 0000 0000 0000 0000 0000 0010   = 2
   6 | 3     0000 0000 0000 0000 0000 0000 0000 0111   = 7
   6 ^ 3     0000 0000 0000 0000 0000 0000 0000 0101   = 5
   ~6        1111 1111 1111 1111 1111 1111 1111 1001   = -7
```

```java
System.out.println(6 & 3);     // 2
System.out.println(6 | 3);     // 7
System.out.println(6 ^ 3);     // 5
System.out.println(~6);        // -7    规律：~a == -(a + 1)
System.out.println(~10);       // -11
System.out.println(~0);        // -1    全 1 的补码就是 -1
System.out.println(~-1);       // 0
```

`~a == -(a + 1)` 这条恒等式值得记住，它是补码设计的直接结果：取反再加 1 等于取负（`~a + 1 == -a`），把 1 移到左边就得到 `~a == -(a+1)`。

## 9.2 三种移位运算

| 运算符 | 名称 | 规则 | 补位 |
|--------|------|------|------|
| `<<` | 左移 | 高位丢弃，低位补 0 | 固定补 0 |
| `>>` | 有符号右移 / 算术右移 | 低位丢弃，高位补**符号位** | 正数补 0，负数补 1 |
| `>>>` | 无符号右移 / 逻辑右移 | 低位丢弃，高位补 0 | **一律补 0** |

`byte`、`short`、`char` 在移位前会先提升为 `int`，所以 `byte b = 1; b << 1` 的结果是 `int` 类型的 2。

```java
public class ShiftDemo {
    public static void main(String[] args) {
        System.out.println(3 << 2);      // 12    3 * 2^2
        System.out.println(-3 << 2);     // -12   负数左移同样按位来，结果仍是负数
        System.out.println(16 >> 2);     // 4     16 / 2^2
        System.out.println(-8 >> 1);     // -4    高位补 1，结果仍是负数
        System.out.println(-7 >> 1);     // -4    向下取整（而 -7 / 2 == -3）
        System.out.println(-8 >>> 1);    // 2147483644   高位补 0，直接变成正数
        System.out.println(-1 >>> 1);    // 2147483647   等于 Integer.MAX_VALUE
        System.out.println(1L << 40);    // 1099511627776  long 可以移到更高位
    }
}
```

`>>` 与 `>>>` 的差别（这是本节的核心）：

```text
   -8 的 int 补码： 1111 1111 1111 1111 1111 1111 1111 1000

   -8 >> 1   （高位补符号位 1）    1111 1111 1111 1111 1111 1111 1111 1100  = -4
   -8 >>> 1  （高位一律补 0）      0111 1111 1111 1111 1111 1111 1111 1100  = 2147483644

   两行结果的差别只在最高位：>> 补的是符号位 1，>>> 补的是 0。
   符号位变成 0 之后，负数就"变成"了正数。
```

`>>` 与除法 `/` 的区别也必须记住：**`>>` 是向下取整，`/` 是向零取整**。

| 表达式 | 结果 | 规则 |
|--------|------|------|
| `7 / 2` | `3` | 向零 |
| `-7 / 2` | `-3` | 向零（-3.5 → -3） |
| `7 >> 1` | `3` | 向下 |
| `-7 >> 1` | `-4` | 向下（-3.5 → -4） |

## 9.3 移位位数会先取模

这是移位运算里最反直觉的一条：`int` 的移位只用移位数**最低 5 位**（等价于对 32 取模），`long` 用最低 6 位（对 64 取模）。

```java
System.out.println(1 << 32);     // 1    32 % 32 == 0，等于没移
System.out.println(1 << 33);     // 2    33 % 32 == 1，相当于左移 1 位
System.out.println(1 << 64);     // 1    64 % 32 == 0
System.out.println(1L << 64);    // 1    long 用移位数 % 64
System.out.println(1L << 65);    // 2
```

> **陷阱**：`hash << 32`、`1 << 64` 这类写法**不会得到 0**，而是得到原值，因为移位距离被静默取模了。这是"写了位运算但没生效"最常见的原因——你以为移出去了，其实一动没动。想要"移出去就变 0"，得用 `long` 再截断，或者干脆不用位运算。

## 9.4 位运算的真实用途

### 9.4.1 权限位掩码

一个用户同时有读、写、执行三种权限。用三个 `boolean` 字段需要三列存储，用一个 `int` 的三个位只需要一列，而且可以整存整取、一次传输：

```java
public class PermissionDemo {
    // 每一位代表一种权限，1 << n 只在第 n 位上产生一个 1
    public static final int READ    = 1 << 0;   // 0001 = 1
    public static final int WRITE   = 1 << 1;   // 0010 = 2
    public static final int EXECUTE = 1 << 2;   // 0100 = 4

    public static void main(String[] args) {
        int perm = 0;

        perm |= READ;                                 // 置位：授予读权限    0001
        perm |= WRITE;                                // 置位：追加写权限    0011
        System.out.println(perm);                     // 3

        System.out.println((perm & READ) != 0);       // true    检查读权限
        System.out.println((perm & EXECUTE) != 0);    // false   检查执行权限

        perm &= ~WRITE;                               // 清零：~WRITE = ...1111 1101
        System.out.println(perm);                     // 1

        perm ^= READ;                                 // 翻转：有则去掉、无则加上
        System.out.println(perm);                     // 0

        perm |= READ | WRITE | EXECUTE;               // 一次授予全部权限
        System.out.println(perm);                     // 7
        System.out.println(Integer.toBinaryString(perm)); // 111
    }
}
```

这套"置位用 `|=`、检查用 `&`、清零用 `&= ~`、翻转用 `^=`"的模式，在 JDK 源码里到处可见——`java.lang.reflect.Modifier`（类的 public/static/final 修饰符）、`java.nio.file.attribute.PosixFilePermission`、`java.util.regex.Pattern` 的 flag、Netty 的 `ChannelOption` 都是这么做的。

### 9.4.2 `>>>` 处理 hash：HashMap 的扰动函数

`HashMap` 用 `hash & (capacity - 1)` 定位桶。因为 `capacity` 是 2 的幂，`capacity - 1` 的低位全是 1、高位全是 0，**高位信息完全参与不进运算**。如果直接用 `hashCode()`，高 16 位就被浪费了。JDK8 的解决办法是把高 16 位异或到低 16 位：

```java
static final int hash(Object key) {
    int h;
    // h >>> 16 是无符号右移：把高 16 位移到低位，再与低 16 位异或
    // 这样高位信息被"混合"进低位，桶的分布更均匀，冲突更少
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

这里必须用 `>>>` 而不是 `>>`：`hashCode()` 可能是负数，用 `>>` 会把符号位一起带过来，混出来的结果还是被符号污染。HashMap 的完整实现（扩容、树化、并发问题）见第 9 篇。

### 9.4.3 移位代替乘除 2 的幂

对**非负整数或精确整除**的情况，`x << n` 等价于 `x * 2^n`，`x >> n` 等价于 `x / 2^n`：

```java
int x = 100;
System.out.println(x << 3);     // 800   100 * 8
System.out.println(x << 1);     // 200   100 * 2
System.out.println(x >> 2);     // 25    100 / 4
System.out.println(x * 1024 == (x << 10));   // true
```

> **陷阱**：**用移位代替除法只在确定为非负时才安全**，因为 `>>` 向下取整、`/` 向零取整，`-7 / 2 == -3` 而 `-7 >> 1 == -4`。更实际的一点是：**现代 JIT 编译器会自动把"除以 2 的幂"优化成移位指令**，手写 `>>` 在 Java 里**没有任何性能收益，只有可读性损失**。这条只适合用来读懂别人（或 JDK）的代码，不适合写进业务代码。

### 9.4.4 其它常见套路

| 目的 | 写法 | 原理 |
|------|------|------|
| 判断奇偶 | `(n & 1) == 1` | 只看最低位，对负数也正确 |
| 判断是否为 2 的幂 | `n > 0 && (n & (n - 1)) == 0` | 2 的幂只有一个 1，减 1 后按位与必为 0 |
| 取最低位的 1 | `n & -n` | 补码性质，树状数组（Fenwick Tree）的核心 |
| 清掉最低位的 1 | `n & (n - 1)` | 可用来数二进制里 1 的个数 |
| 大小写互换 | `c ^ 32` | ASCII 中大小写只差第 6 位（32） |
| 取绝对值 | `(n ^ (n >> 31)) - (n >> 31)` | 用符号位生成掩码；实际直接用 `Math.abs` |
| 交换两数 | `a ^= b; b ^= a; a ^= b;` | 不需要临时变量；实际别这么写 |

```java
public class BitTrickDemo {
    public static void main(String[] args) {
        // 用 n & (n-1) 数二进制中 1 的个数
        int n = 0b1011_0110;              // JDK7+ 支持二进制字面量，值为 182
        int original = n;
        int count = 0;
        while (n != 0) {
            n &= (n - 1);                 // 每轮消掉最低位的那个 1
            count++;
        }
        System.out.println(original + " 中有 " + count + " 个 1");   // 182 中有 5 个 1

        // JDK 已内置了这些方法，业务代码优先用它们
        System.out.println(Integer.bitCount(0b1011_0110));      // 5
        System.out.println(Integer.highestOneBit(182));         // 128   最高位的 1 的值
        System.out.println(Integer.lowestOneBit(182));          // 2     最低位的 1 的值
        System.out.println(Integer.numberOfLeadingZeros(1));    // 31    前导零个数
        System.out.println(Integer.toBinaryString(182));        // 10110110

        // 大小写转换（只对字母有效）
        char c = 'A';
        System.out.println((char) (c ^ 32));                    // a
    }
}
```

> **最佳实践**：`Integer.bitCount`、`Integer.highestOneBit` 这类 JDK 内置方法**优先使用**——它们内部还可能用到 CPU 的 `popcnt` 指令，比自己写的循环快得多。手写位运算技巧只在面试题和性能极限场景下才有意义。

### 9.4.5 位运算的优先级陷阱

`&`、`|`、`^` 的优先级**低于** `==`、`!=`，这一点和 C 语言一样反直觉：

```java
int flags = 5;                          // 0101
System.out.println(flags & 4);          // 4
System.out.println((flags & 4) == 4);   // true   ✓ 加括号

// 下面这行是编译错误，因为先算 4 == 4 得到 boolean，再与 int 做 &
// System.out.println(flags & 4 == 4);

// 同理，判断位是否设置的常见错误写法：
// if (flags & MASK == MASK)             // ✗ 编译错误或逻辑错误
if ((flags & 4) == 4) { }                // ✓ 永远给位运算加括号
```

> **最佳实践**：**位运算和比较混用时永远加括号**。绝大多数人记不住 `&` 和 `==` 谁先算，加括号是唯一可靠的沟通方式。

# 10. 运算符优先级

## 10.1 完整优先级表

从高到低（同级按"结合性"方向求值）：

| 级别 | 运算符 | 结合性 |
|------|--------|--------|
| 1（最高） | `()` `[]` `.` `::`（方法引用） | 从左到右 |
| 2 | `!` `~` `++` `--` `+` `-` `(类型)`（一元与强制类型转换） | 从右到左 |
| 3 | `*` `/` `%` | 从左到右 |
| 4 | `+` `-` | 从左到右 |
| 5 | `<<` `>>` `>>>` | 从左到右 |
| 6 | `<` `>` `<=` `>=` `instanceof` | 从左到右 |
| 7 | `==` `!=` | 从左到右 |
| 8 | `&`（按位与） | 从左到右 |
| 9 | `^`（按位异或） | 从左到右 |
| 10 | `\|`（按位或） | 从左到右 |
| 11 | `&&` | 从左到右 |
| 12 | `\|\|` | 从左到右 |
| 13 | `?:` | 从右到左 |
| 14（最低） | `=` `+=` `-=` `*=` `/=` `%=` `&=` `^=` `\|=` `<<=` `>>=` `>>>=` | 从右到左 |

几条容易记错的：

- **后置自增 `a++` 属于第 1 级**（与括号同级），**前置自增 `++a` 属于第 2 级**。所以 `a++ + 1` 是 `(a++) + 1`。
- **`&&` 高于 `||`**，很多教程把它们合并成"同一级"，那是不准确的说法。`true || false && false` 的结果是 `true`（先算 `false && false`）。
- **位运算符 `&`、`^`、`|` 的优先级比 `==` 低**。这跟 `&&`、`||` 高于 `==` 的规律正好相反，是这一整张表里最容易踩的地方。
- **赋值运算符最低**，所以 `double d = a / b` 里的除法一定先算完（见 2.1 的事故）。

## 10.2 常见易错组合

```java
public class PriorityDemo {
    public static void main(String[] args) {
        int a = 1, b = 2, c = 3;

        System.out.println(a + b * c);          // 7     乘高于加
        System.out.println((a + b) * c);        // 9
        System.out.println(1 + 2 + "3");        // 33    加号左结合：先算 1+2，再拼接
        System.out.println("1" + 2 + 3);        // 123   一旦是字符串，后面全是拼接

        System.out.println(true || false && false);   // true   && 高于 ||
        System.out.println((true || false) && false); // false  括号改变结果

        System.out.println(~2 + 1);             // -2     ~ 是一元运算符，高于 +
        System.out.println(!(1 > 2) == true);   // true   ! 先作用于括号结果

        int x = true ? 1 : 2;                   // 等价于 int x = (true ? 1 : 2)
        System.out.println(x);                  // 1
        int y = true ? 1 : 2 + 3;               // 2+3 先算，然后才轮到 ?:
        System.out.println(y);                  // 1

        // 复合赋值右结合，可以链式展开
        int p = 1, q = 2;
        p += q += 3;                            // 等价于 q = q + 3; p = p + q
        System.out.println("p=" + p + ", q=" + q);   // p=6, q=5
    }
}
```

## 10.3 记不住怎么办

> **最佳实践**：只需要记住三条主线——**一元运算符最高、赋值运算符最低、圆括号可以打破一切**；其余场景**一律主动加括号**。括号不产生任何运行时开销（编译期就确定了求值顺序），却能省下读代码的人十秒钟，也省下自己三个月后的十分钟调试。位运算、三元嵌套、字符串拼接这三类表达式**必须**加括号。

# 11. 小结

| 主题 | 一句话要点 |
|------|-----------|
| 整除 | 两个整数相除直接截断小数（向零），`5 / 2 == 2`；`double avg = sum / count` 是最经典的静默事故 |
| 取模 | 结果符号跟随**被除数**: `-7 % 3 == -1`；判断奇偶别写 `n % 2 == 1` |
| 加号 | 数值加法、字符串拼接、`char` 按码点相加三合一；`"Hello" + 10 + 20` 得 `Hello1020` |
| 溢出 | 整型溢出会**静默回绕**；`long x = 1000 * 60 * 60 * 24 * 30` 救不了，必须写 `1000L * ...` |
| 浮点精度 | `0.1 + 0.2 != 0.3`；金额用 `BigDecimal(String)`，见第 11 篇 |
| 隐式转换 | 小范围 → 大范围自动提升；`byte`/`short`/`char` 参与运算**一律先提升为 `int`** |
| 强制转换 | 整数窄化是**砍高位回绕**（`(byte) 130 == -126`）；浮点转整数是**截断**（`(int) 3.99 == 3`），越界则饱和 |
| `byte b = 130` | 字面量超范围，**编译期**就报错；`(byte) 130` 才合法（编译器认为你知情） |
| 自增自减 | 前置取新值、后置取旧值；`i = i++` 之后 `i` 不变；单独成句时两者无性能差别 |
| 复合赋值 | **隐含强制类型转换**: `s += 1` 等价于 `s = (short)(s + 1)`，会静默溢出 |
| 关系运算 | 结果恒为 `boolean`；优先级低于算术；引用类型 `==` 比的是地址 |
| 逻辑运算 | 一律用短路的 `&&`、`\|\|`；`&`、`\|`、`^` 用在整数上是位运算 |
| 三元 | 条件必须 `boolean`；两分支类型会统一，**包装类型与基本类型混搭会提前拆箱导致 NPE** |
| 位运算 | `<<`/`>>` 只在非负数上能替代乘除 2 的幂（`-7/2 != -7>>1`）；`>>>` 用于补 0 与 hash 扰动；**移位位数会被取模**（`1 << 32 == 1`） |
| 优先级 | 一元最高、赋值最低、`&&` 高于 `\|\|`，而位运算低于比较；**拿不准就加括号** |

下一篇（第 4 篇）用这些运算符搭出程序的骨架——分支与循环；`BigDecimal` 的完整 API 在第 11 篇，HashMap 的 hash 扰动与扩容在第 9 篇，字符串拼接的性能问题在第 8 篇。
