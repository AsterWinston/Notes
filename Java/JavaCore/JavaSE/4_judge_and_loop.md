程序等于数据加流程，前面几篇把变量、运算符这些"数据"讲完了，本篇讲"流程"——**怎么让代码按条件分叉、按次数重复、以及在循环中途提前退出**。分支与循环的语法非常简单，真正的难点全在边界上：悬空 else 到底挂在哪个 if 上、case 忘写 break 会穿透到哪里、带标签的 break 该放在哪一行、循环变量出了作用域还能不能看见——每一个结论都配可运行示例和真实输出。

# 1. 流程控制结构总述

程序默认是**顺序执行**的，只有三种基本结构能改变它。任何复杂的逻辑都可以由这三种结构组合而成（这是结构化程序设计的核心结论）。

| 结构 | 关键字 | 作用 | 典型场景 |
|------|--------|------|----------|
| 顺序结构 | 无 | 从上到下、从左到右依次执行 | 所有代码的默认形态 |
| 分支结构 | `if`、`switch` | 按条件选择执行哪一段 | 参数校验、状态分发、枚举路由 |
| 循环结构 | `for`、`while`、`do-while` | 重复执行某一段 | 遍历、累加、轮询、重试 |

```text
   顺序结构                   分支结构                    循环结构
   ┌──────────┐              ┌──────────┐           ┌────────────────────┐
   │  语句 1   │              │  条件 ?  │           │                    │
   └────┬─────┘              └────┬─────┘           ▼                    │
        │                         │ 真        ┌──────────┐               │
        ▼                         ▼           │  条件 ?  │── 假 ──▶ 后续  │
   ┌──────────┐              ┌──────────┐     └────┬─────┘               │
   │  语句 2   │              │  语句块   │          │ 真                  │
   └────┬─────┘              └────┬─────┘          ▼                     │
        │                         ▼           ┌──────────┐               │
        ▼                    ┌──────────┐     │  循环体   │               │
   ┌──────────┐              │ 后续代码  │     └────┬─────┘               │
   │  语句 3   │              └──────────┘          │                     │
   └──────────┘                                    └─────────────────────┘
                                                    执行完循环体后回到条件判断
```

## 1.1 顺序结构

最基础的结构，代码从上到下、从左到右依次执行，没有任何跳转。前面章节的代码几乎都是顺序结构：

```java
public class SequenceDemo {
    public static void main(String[] args) {
        int a = 10;
        int b = 20;
        int sum = a + b;          // 必须等 a、b 都赋值完成才能执行
        System.out.println(sum);  // 30
    }
}
```

顺序结构的关键是**语句之间存在依赖关系**：如果把 `int sum = a + b;` 提到 `int b = 20;` 之前，编译器会直接报"变量 b 可能尚未初始化"。所谓"代码顺序"，本质上就是**数据的依赖顺序**。

## 1.2 三种结构的组合

任意两种结构都可以互相嵌套，形成更复杂的逻辑：

```java
// 分支里嵌套循环：只处理性别为"男"的学生
if (students != null) {                 // 分支
    for (Student s : students) {        // 循环
        if (s.isMale()) {               // 分支
            System.out.println(s.getName());
        }
    }
}
```

> **设计考量**：嵌套层数超过三层就该考虑**抽取方法**了。深层嵌套让"当前处于什么条件下"变得难以追踪，这也是为什么后面介绍的 `return` 提前返回、`switch` 表达式、以及带标签跳转都要小心使用——它们都能减少嵌套，但也都能让控制流变得难读。

# 2. if 分支

## 2.1 三种形态

### 2.1.1 单分支 if

```java
int age = 20;
if (age >= 18) {                 // 条件必须是 boolean
    System.out.println("已成年");  // 条件成立才执行
}
// 已成年
```

**条件表达式必须是 `boolean`**，这是 Java 与 C 的重要差别：

```java
// if (1) { }               // ✗ 编译错误：int 不能当 boolean
// if ("true") { }          // ✗ 编译错误：String 不能当 boolean
if (1 > 0) { }              // ✓ 必须是比较运算的结果
```

### 2.1.2 双分支 if-else

```java
int score = 59;
if (score >= 60) {
    System.out.println("及格");
} else {
    System.out.println("不及格");   // 两条分支互斥，必执行且只执行其一
}
// 不及格
```

### 2.1.3 多分支 if-else if-else

```java
int score = 85;
if (score >= 90) {
    System.out.println("优秀");
} else if (score >= 80) {
    System.out.println("良好");      // 命中后，后面的条件不再判断
} else if (score >= 60) {
    System.out.println("及格");
} else {
    System.out.println("不及格");
}
// 良好
```

**核心规则：从上到下依次判断，命中一个就跳出整条 if-else 链**，后面的条件连求值都不会发生（相当于隐含了短路）。

这带来一个重要推论：**条件的顺序决定语义**。

```java
// 把顺序写反，85 分会变成"及格"
int score = 85;
if (score >= 60) {
    System.out.println("及格");       // 85 >= 60 先命中，后面的分支全被跳过
} else if (score >= 80) {
    System.out.println("良好");       // 永远不会被执行到
} else if (score >= 90) {
    System.out.println("优秀");
}
// 及格
```

上面的 `score >= 80` 与 `score >= 90` 两个分支是**不可达代码**，编译器不会报错（它不做区间推理），但逻辑已经错了。

> **注意**：写多分支 if 时，条件的**重叠与顺序**是最容易出错的地方。判断标准是：对每个分支的边界值（90、89、80、79、60、59）都心算一遍它该走哪一条。范围判断最好写成"从最严格往最宽松"或"从最宽松往最严格"的**单调**顺序，不要来回跳。

## 2.2 大括号与悬挂 else

`if` 后面如果只有**一条语句**，大括号可以省略：

```java
int age = 20;
if (age >= 18)
    System.out.println("已成年");     // 语法合法，但不推荐
```

但省略大括号会引来**悬挂 else（dangling else）**问题：**不带大括号时，`else` 绑定到最近的、还没有 else 的那个 `if`**，与缩进无关。

```java
int x = 5;
int y = 10;

if (x > 0)
    if (y > 0)
        System.out.println("x>0 且 y>0");
else                                       // 这个 else 属于内层 if（最近的），不是外层！
    System.out.println("x<=0");
// x>0 且 y>0
```

实际输出是 `x>0 且 y>0`，`else` 分支没有执行。如果按缩进理解成"else 属于外层 `if (x > 0)`"，就会得出完全错误的结论。Java 编译器**不会给出任何警告**，缩进会骗人。

```java
// 用大括号消除一切歧义
if (x > 0) {
    if (y > 0) {
        System.out.println("x>0 且 y>0");
    }
} else {
    System.out.println("x<=0");
}
```

> **最佳实践**：**永远写大括号**，即使分支里只有一条语句。理由有三：一是后续往分支里加一行代码时忘补大括号会直接改变逻辑（且不报错）；二是 `if`、`for`、`while`、`switch` 的大括号规则各不相同，统一写反而更好记；三是主流团队的静态检查（阿里规约、Sonar、Checkstyle）都把它列为强制规则。格式上按主流约定，左大括号跟在行尾，不要另起一行。

## 2.3 三个高频陷阱

### 2.3.1 if 后面误加分号

```java
int age = 20;
if (age >= 18);                     // 注意：分号是一条空语句，if 只管这条空语句
{
    System.out.println("已成年");      // 这个块无条件执行
}
// 无论 age 是多少，都会打印"已成年"
```

分号把 `if` 的"受控语句"从代码块变成了空语句，后面的 `{ }` 只是一个普通的语句块，永远会执行。**`if (...)` 后面不要出现分号**，这是新手最常犯的错。

### 2.3.2 `=` 与 `==`

```java
int a = 5;
// if (a = 5) { }                    // ✗ 编译错误：int 不能当 boolean 用
if (a == 5) { }                      // ✓

boolean flag = true;
if (flag = false) {                  // ✗ 能编译通过！把 false 赋给 flag 后判断，逻辑整个反了
    System.out.println("成立");
}
System.out.println(flag);            // false —— flag 被改掉了，还附带了副作用
```

对 `int` 的检查是 Java 相对 C 的一大保护（因为 `int` 不能当布尔值用）；但对 `boolean` 变量**没有这层保护**，`if (flag = true)` 依然合法且**恒为真**。防住的唯一办法是养成习惯：**判断布尔值不写 `==`**。

```java
if (flag) { }             // ✓ 最好
if (!flag) { }            // ✓ 取反
if (flag == true) { }     // 能编译，但啰嗦，且给了打错成 = 的机会
```

### 2.3.3 浮点数直接比较相等

```java
double d = 0.1 + 0.2;
// if (d == 0.3) { }                  // ✗ 永远不成立
if (Math.abs(d - 0.3) < 1e-9) {       // ✓ 用误差范围
    System.out.println("约等于 0.3");
}
// 约等于 0.3
```

浮点数的表示误差见第 3 篇 2.3，容器类型转换引发的精度问题见第 3 篇第 3 节。

# 3. switch 分支

`switch` 用于把**一个值**和若干**候选值**做等值匹配，比一串 `if-else if` 更清晰，也更快——JDK7 起，编译成 `tableswitch`（连续值，查表跳转）或 `lookupswitch`（离散值，二分查找）字节码指令，而不是逐个 `if` 比较。

## 3.1 基本语法与执行流程

```java
switch (要匹配的值) {
    case 值1:
        语句;
        break;
    case 值2:
        语句;
        break;
    default:
        语句;
        break;
}
```

```text
   switch (表达式)   计算一次，结果必须是小整数、字符、枚举或字符串
        │
        ▼
   ┌───────────────────────┐
   │ 从上到下逐个比对 case 的值 │
   └───────────┬───────────┘
               │
       ┌───────┴────────┐
       │ 命中           │ 没有命中任何 case
       ▼                ▼
   ┌────────────┐   ┌──────────────────┐
   │ 执行该 case │   │ 执行 default（若有）│
   └──────┬─────┘   └────────┬─────────┘
          │                  │
          ▼                  ▼
   ┌────────────────────────────────┐
   │ 遇到 break 或 switch 的 } 就结束  │
   │ 没有 break 则继续执行下一个 case  │
   │ （这就是"穿透"）                 │
   └────────────────────────────────┘
```

```java
int week = 3;
switch (week) {
    case 1:
        System.out.println("周一");
        break;
    case 3:
        System.out.println("周三");
        break;
    default:
        System.out.println("其它");
        break;
}
// 周三
```

## 3.2 匹配值类型的演进

这是 switch 最值得记住的一条"历史线"，每一次扩展都对应一个 JDK 版本：

| JDK 版本 | 新增支持的类型 | 说明 |
|----------|---------------|------|
| JDK 1.0 | `byte`、`short`、`int`、`char` | 初始支持；对应的包装类 `Byte`/`Short`/`Integer`/`Character` 也能用，靠自动拆箱 |
| JDK 5 | 枚举类型（`enum`） | 编译时把枚举常量转成它的 `ordinal()`（序号）再比较 |
| JDK 7 | `String` | 先算 `hashCode()` 定位，再用 `equals()` 确认 |
| JDK 12（预览）/ JDK 14（转正） | switch **表达式**、`->` 箭头语法、`yield` | 语言形态的升级，见 3.6 |
| JDK 21 | 模式匹配：`case String s ->`、`case null ->`、记录模式 | 见第 29 篇 |
| **一直不支持** | `long`、`float`、`double`、`boolean` | 见下面的原因分析 |

**为什么不支持 `long`**：`tableswitch`/`lookupswitch` 指令的 case 值和索引都是 `int` 类型，`long` 放不下。

**为什么不支持 `float`/`double`**：浮点数无法精确比较（见第 3 篇），用它做**等值**分支本身就是错误的设计。真要按浮点分档，应该用 `if-else` 加区间判断。

**为什么不支持 `boolean`**：只有两个取值，`if-else` 已经足够，加进来只会让语言更复杂。

```java
// char 与 String 的示例
char grade = 'B';
switch (grade) {
    case 'A':
        System.out.println("优秀");
        break;
    case 'B':
        System.out.println("良好");
        break;
    default:
        System.out.println("其它");
        break;
}
// 良好

String day = "MON";
switch (day) {                          // JDK7+ 支持 String
    case "MON":
    case "TUE":                         // 多个 case 共用一段代码（穿透的正当用法）
        System.out.println("工作日");
        break;
    case "SAT":
    case "SUN":
        System.out.println("休息日");
        break;
    default:
        System.out.println("非法输入");
        break;
}
// 工作日
```

`String` 的 switch 底层是**先算 `hashCode()` 得到一个 `int` 再走 switch，然后在每个 case 里用 `equals()` 二次确认**。之所以要二次确认，是因为不同字符串的 `hashCode()` 可能相同（哈希冲突），只比哈希不足以判定相等。所以 `switch(String)` 比 `switch(int)` 要慢一些，且**值为 `null` 时会直接抛 `NullPointerException`**（`null.hashCode()` 行不通）。

## 3.3 case 的规则

**规则一：case 后面只能是常量表达式或字面量，不能是变量。**

```java
int a = 1;
int x = 1;
// switch (x) { case a: ... }         // ✗ 编译错误：case 表达式必须是常量表达式

final int B = 2;                      // final 修饰的编译期常量可以用
switch (x) { case B: System.out.println("B"); break; }

static final int C = 3;               // static final 常量也行
```

**规则二：各个 case 的值必须互不相同**，重复会直接编译报错：

```java
// switch (x) { case 1: ...; case 1: ...; }   // ✗ 编译错误：case 标签重复
```

**规则三：JDK 12+ 支持多值合并**，用逗号连接：

```java
int month = 4;
switch (month) {
    case 1, 3, 5, 7, 8, 10, 12 -> System.out.println("31 天");   // 比写 7 个空 case 清爽得多
    case 4, 6, 9, 11           -> System.out.println("30 天");
    case 2                     -> System.out.println("28 或 29 天");
    default                    -> System.out.println("非法月份");
}
// 30 天
```

## 3.4 case 穿透（fall-through）

一个 case 的代码块结尾没有 `break`，程序就会**继续执行下一个 case 的代码**，直到遇到 `break` 或整个 switch 结束。这是 switch 最著名的特性，也是头号 bug 来源。

### 3.4.1 有意利用穿透：多个值共用一段代码

```java
int month = 4;
int days;
switch (month) {
    case 1: case 3: case 5: case 7: case 8: case 10: case 12:
        days = 31;                    // 这批 case 依次落进同一段代码
        break;                        // 到这里才停
    case 4: case 6: case 9: case 11:
        days = 30;
        break;
    case 2:
        days = 28;
        break;
    default:
        throw new IllegalArgumentException("非法月份: " + month);
}
System.out.println(days);   // 30
```

这是穿透的**唯一正当用法**，而且要配合缩进与注释，让人一眼看出是故意的。

### 3.4.2 无意的穿透：忘写 break

```java
int level = 1;
String desc;
switch (level) {
    case 1:
        desc = "青铜";
        // 忘了 break
    case 2:
        desc = "白银";
        // 忘了 break
    case 3:
        desc = "黄金";
        break;
    default:
        desc = "未知";
}
System.out.println(desc);   // 黄金 —— 不是预期的"青铜"
```

`level = 1` 时会依次执行 case 1、case 2、case 3 的代码，`desc` 被反复覆盖，最终留下"黄金"。**既不会报错，也不会有警告**（早期 `javac` 的 `-Xlint:fallthrough` 可以提示，但默认关闭）。

再看一个更隐蔽的例子，穿透导致的**副作用重复执行**：

```java
int n = 1;
switch (n) {
    case 1:
        System.out.println("第一段");
        // 忘了 break
    case 2:
        System.out.println("第二段");
        break;
}
// 第一段
// 第二段      两段都执行了
```

> **陷阱**：判断一个 switch 是否写对了，就数一数——**每个 case 代码块的末尾是不是都有 `break`（或 `return`/`throw`）**。如果一段代码里同时存在"有意穿透"和"无意穿透"，后来者根本无法分辨哪些是故意的。**引入 JDK14 的箭头语法可以从语法层面彻底消灭这个问题**（见 3.6）。

## 3.5 default 的用法

`default` 分支在**所有 case 都不匹配**时执行。关于它有三条要注意的：

**一、可以省略，但省略后若没有 case 命中，switch 什么都不做**，程序静默往下走——这往往是 bug 的源头（比如给枚举新增了一个值，但忘了加 case）。

```java
int status = 99;
switch (status) {
    case 1:
        System.out.println("待支付");
        break;
    case 2:
        System.out.println("已支付");
        break;
    // 没有 default，status = 99 时什么也不打印
}
System.out.println("继续执行");      // 继续执行
```

**二、位置不一定要放在最后**。语法上放在开头或中间都合法，执行顺序仍然是"先匹配所有 case，都不中才来 default"。但按惯例放在最后，可读性最好。

```java
int x = 99;
switch (x) {
    default:                                    // 放在开头也合法
        System.out.println("未知");
        break;                                  // 需要 break，否则会穿透到 case 1
    case 1:
        System.out.println("一");
        break;
}
// 未知
```

**三、即使放在最后，也建议保留 `break`**，这样后续追加 case 时不会意外穿透。

```java
// 推荐的 default 写法：对非法输入显式失败，而不是静默放过
switch (status) {
    case 1: return "待支付";
    case 2: return "已支付";
    default:
        throw new IllegalStateException("未知的订单状态: " + status);
}
```

> **设计考量**：`default` 的处理方式其实反映了你对"非法输入"的策略——**抛异常**（快速失败，适合内部调用，能把问题暴露在测试阶段）、**返回兜底值**（适合对外接口，保证不崩）、**记日志后继续**（适合可容忍的场景）。唯独**不要让 default 留空**：留空等于声明"这种情况不存在"，而它往往真的存在，只是你还没遇到。

## 3.6 JDK 14+ 的箭头语法与 switch 表达式

JDK 12 引入预览、JDK 14 正式转正的 switch 表达式带来两个重大变化。

**变化一：`->` 代替 `:`，天然不穿透。**

```java
int level = 1;
String desc = switch (level) {          // switch 可以作为表达式，直接赋值
    case 1 -> "青铜";
    case 2 -> "白银";
    case 3 -> "黄金";
    default -> "未知";
};
System.out.println(desc);               // 青铜
```

`->` 后面**只能跟一条语句或一个表达式**，执行完这个 case 就自动结束，**不会穿透**——这是它相对 `:` 最大的价值。

**变化二：`{ }` + `yield` 支持多语句并返回值。**

`->` 后面要写多行，必须用大括号包裹成代码块；此时如果整个 switch 还要作为**表达式**返回值，就必须用 `yield` 显式交出结果：

```java
int month = 12;
int quarter = switch (month) {
    case 1, 2, 3    -> 1;               // 单表达式，直接返回
    case 4, 5, 6    -> 2;
    case 7, 8, 9    -> 3;
    case 10, 11, 12 -> 4;
    default -> {
        // 多行代码块，必须用 yield 返回结果
        System.out.println("非法月份: " + month);
        yield -1;                       // yield 只结束 switch 表达式，不是结束方法
    }
};
System.out.println(quarter);            // 4
```

两种写法的完整对比：

| 维度 | 传统 `:` 写法 | `->` 箭头写法（JDK14+） |
|------|--------------|------------------------|
| 穿透 | 会穿透，必须手写 `break` | 不会穿透 |
| 作为表达式赋值 | 不支持（要用临时变量中转） | 支持，`case X -> 值` 直接返回 |
| 多值合并 | 只能靠 `case 1: case 2:` 堆叠 | `case 1, 2 ->` 一行写完 |
| 多语句 + 返回值 | —— | `{ ...; yield 值; }` |
| 穷尽性检查 | 无 | 用于表达式时，编译器**强制要求覆盖所有取值** |
| 书写量 | 长 | 短，且不容易漏 break |

**穷尽性检查**是箭头写法带来的额外收益，在枚举上尤其有用：

```java
enum Season { SPRING, SUMMER, AUTUMN, WINTER }

Season s = Season.SUMMER;
String tip = switch (s) {
    case SPRING -> "春天";
    case SUMMER -> "夏天";
    case AUTUMN -> "秋天";
    case WINTER -> "冬天";
    // 四个枚举常量全覆盖，可以不给 default 也能编译
};
System.out.println(tip);   // 夏天
```

**日后往枚举里加一个 `MONSOON`，这段代码会直接编译报错**，提醒你补上分支——这是 `:` 写法做不到的（传统写法只会静默走到 `default` 或什么都不做）。

> **注意**：`yield` **不是** `return`。`yield` 只是结束当前 switch 表达式并把值交给外层；`return` 会直接结束整个方法。在 switch 表达式的 `->` 代码块里用 `return` 会导致编译错误（"在 switch 表达式内部使用 return 试图跳出方法"）。switch 的模式匹配（`case String s ->`、`case null ->`、记录模式）是第 29 篇的重点内容。

## 3.7 switch 与 if-else 的选择

| 场景 | 推荐 | 原因 |
|------|------|------|
| 对**一个值**做**等值**多路分发 | `switch`（表达式更佳） | 可读性好，字节码是查表/二分，效率高 |
| **范围**判断（`> 60`、`< 0`） | `if-else` | switch 只能等值匹配 |
| 条件涉及**多个变量**的组合 | `if-else` | switch 只匹配一个表达式 |
| 分支里需要多行逻辑 | 新代码用 `switch` 表达式 + `yield` | 从语法上避免忘写 break |
| 枚举 / 状态机路由 | `switch` 表达式 | 编译器帮忙做穷尽性检查 |
| 只判断"真/假" | `if` | 用 switch 是杀鸡用牛刀 |

# 4. 循环结构

## 4.1 for 循环

### 4.1.1 标准格式与执行顺序

```java
for (初始化语句; 条件判断语句; 条件控制语句) {
    循环体;
}
```

执行顺序是：初始化（**只执行一次**）→ 条件判断 → （成立）循环体 → 条件控制 → 条件判断 → ...

```text
   ┌──────────────────────────┐
   │ 初始化语句（只执行一次）      │
   └────────────┬─────────────┘
                ▼
   ┌──────────────────────────┐  假
   │      条件判断语句          │──────────▶ 循环结束，执行后续代码
   └────────────┬─────────────┘
                │ 真
                ▼
   ┌──────────────────────────┐
   │        循环体             │
   └────────────┬─────────────┘
                ▼
   ┌──────────────────────────┐
   │      条件控制语句          │
   └────────────┬─────────────┘
                └────────▶ 回到条件判断
```

```java
// 求 1~100 的和
int sum = 0;
for (int i = 1; i <= 100; i++) {
    sum += i;
}
System.out.println(sum);    // 5050
```

### 4.1.2 三个部分都可以省略

**条件判断语句省略时默认为 `true`**，也就是死循环：

```java
// 1. 省略条件判断：死循环，必须靠内部 break/return 退出
for (int i = 0; ; i++) {
    if (i >= 3) break;
    System.out.print(i + " ");       // 0 1 2
}
System.out.println();

// 2. 省略全部三个部分：最简洁的死循环写法
for (;;) {                           // 等价于 while (true)
    System.out.println("只执行一次");
    break;
}

// 3. 省略初始化与控制：循环变量提到外面
int i = 0;
for (; i < 3; ) {
    System.out.print(i + " ");       // 0 1 2
    i++;
}
System.out.println();

// 4. 逗号表达式：初始化和控制部分可以有多个语句
for (int a = 0, b = 10; a < b; a++, b--) {
    System.out.println("a=" + a + ", b=" + b);
}
// a=0, b=10
// a=1, b=9
// a=2, b=8
// a=3, b=7
// a=4, b=6
```

### 4.1.3 循环变量的作用域

```java
for (int i = 0; i < 3; i++) {
    System.out.println(i);
}
// System.out.println(i);   // ✗ 编译错误：找不到符号 i
```

各位置的作用域规则：

| 声明位置 | 作用域 |
|----------|--------|
| 初始化语句里声明的变量（`int i`） | 整个 for 结构内部（含括号与循环体） |
| 条件判断里的变量 | 同上 |
| **循环体内声明的变量** | **仅限当前这一轮循环体** |

最后一条最容易踩：

```java
// 循环体内声明的变量，每一轮都会重新创建
for (int i = 0; i < 3; i++) {
    int count = 0;              // 每一轮都是新的 count = 0
    count++;
    System.out.println(count);  // 1 1 1 —— 不是 1 2 3
}

// 想在循环之间保留状态，变量必须声明在循环外
int count = 0;
for (int i = 0; i < 3; i++) {
    count++;
}
System.out.println(count);      // 3
```

> **陷阱**：把累加器、计数器写在循环体内部，是新手最常见的错误之一。判断标准很简单——**这个变量需要在循环结束后还有意义吗？需要就声明在循环外。**"统计有多少个偶数""累加总分"这两类需求都是这个模式。

### 4.1.4 增强 for 循环（for-each）

格式：`for (元素类型 变量名 : 数组或集合) { ... }`

适用于**数组**和所有实现了 `java.lang.Iterable` 接口的对象（`List`、`Set`、`Queue`、`Map` 的 `entrySet()` 等）：

```java
int[] arr = {10, 20, 30};
for (int x : arr) {
    System.out.print(x + " ");      // 10 20 30
}
System.out.println();

List<String> list = List.of("A", "B", "C");
for (String s : list) {
    System.out.print(s + " ");      // A B C
}
System.out.println();
```

增强 for 的**四条限制**：

| 限制 | 说明 |
|------|------|
| 拿不到索引 | 需要下标（"第几个"、回填数组、比较相邻元素）时必须用普通 for |
| 不能反向或跳步遍历 | 只能从头到尾逐个走 |
| 不能修改集合结构 | 遍历中 `add`/`remove` 会抛 `ConcurrentModificationException`（见 6.3） |
| 遍历中给变量赋值无效 | 改 `x` 只改副本，数组/集合本身不变 |

最后一条要展开说：

```java
int[] arr = {1, 2, 3};

for (int x : arr) {
    x = x * 2;                     // 只改了局部变量 x，数组没有任何变化
}
System.out.println(Arrays.toString(arr));   // [1, 2, 3]

for (int i = 0; i < arr.length; i++) {
    arr[i] = arr[i] * 2;           // 按下标改，才是真的改数组
}
System.out.println(Arrays.toString(arr));   // [2, 4, 6]
```

`int` 是基本类型，增强 for 拿到的就是值的**拷贝**。即使元素是对象，改**字段**能生效（因为改的是堆上的对象），但给变量重新赋值仍然只影响副本（参见第 5 篇的参数传递）。

## 4.2 while 循环

```java
while (条件判断语句) {
    循环体;
}
```

**先判断，后执行**：条件一开始就不成立时，循环体**一次都不执行**。适用于**循环次数不确定、由条件驱动**的场景。

```java
int i = 0;
while (i < 3) {
    System.out.print(i + " ");     // 0 1 2
    i++;
}
System.out.println();

// 条件一开始就是 false
int j = 10;
while (j < 3) {
    System.out.println("永远不会执行");
}
System.out.println("循环结束");     // 循环结束
```

`while` 与 `for` 在**能力上完全等价**，任何 `for` 都能改写成 `while`，反之亦然。区别只在语义表达：

```java
// for 版本：循环范围一目了然
for (int k = 0; k < 3; k++) { }

// while 版本：初始化、条件、自增分散在三处，容易漏掉最后一步
int k = 0;
while (k < 3) { k++; }
```

`while` 的经典场景是**条件驱动的循环**：

```java
// 轮询重试，最多 3 次，成功即停
int retry = 0;
boolean success = false;
while (!success && retry < 3) {
    success = tryConnect();        // 假设这是可能失败的操作
    retry++;
    System.out.println("第 " + retry + " 次尝试, 结果: " + success);
}
```

## 4.3 do-while 循环

```java
do {
    循环体;
} while (条件判断语句);        // 结尾的分号不能漏
```

**先执行一次，再判断**：循环体**至少执行一次**。适用场景：菜单至少显示一次、至少校验一轮、"先做一次再决定要不要继续"。

```java
int i = 10;
do {
    System.out.println("至少执行一次, i=" + i);   // 会打印
    i++;
} while (i < 3);
// 至少执行一次, i=10

int j = 0;
do {
    System.out.print(j + " ");     // 0 1 2
    j++;
} while (j < 3);
System.out.println();
```

与 `while` 的唯一区别就是**首轮是否无条件执行**：

```java
int a = 100;
while (a < 3)  { System.out.println("while 分支"); }        // 一次都不打印
do { System.out.println("do-while 分支"); } while (a < 3);   // 打印一次
// do-while 分支
```

## 4.4 三种循环的对比与选择

| 维度 | `for` | `while` | `do-while` |
|------|-------|---------|------------|
| 最少执行次数 | 0 | 0 | **1** |
| 判断时机 | 每轮循环体**之前** | 每轮循环体**之前** | 每轮循环体**之后** |
| 循环变量作用域 | 可限制在循环内（推荐） | 通常在循环外声明 | 同 `while` |
| 是否需要 `break` 退出 | 视情况 | 视情况 | 视情况 |
| 典型场景 | 已知次数、需要下标、遍历 | 条件驱动、次数未知、轮询 | 至少执行一次 |
| 常见错误 | 条件写成 `<=` 导致越界 | 忘记更新条件变量 → 死循环 | **漏写结尾的分号** |

**"什么时候用哪个"的实用判断**：

1. 有**明确的次数或范围**（遍历数组、跑 n 次）→ `for`；
2. 需要**下标参与计算**、需要反向或跳步遍历 → 普通 `for`；
3. 只是**遍历全部元素**、不需要下标 → 增强 `for`；
4. **次数未知、由条件决定**（读到末尾、队列非空、重试直到成功）→ `while`；
5. 业务上**必须先做一次**（先展示菜单、先发一次请求再判断）→ `do-while`。

> **设计考量**：`do-while` 在实际业务代码里很少见，因为它把"至少执行一次"这个前提**隐含**在语法里，读者很容易忽略"这段代码一定会跑一次"这一点。如果跳出条件比较复杂，宁可写成 `while (true) { ...; if (结束条件) break; }`，把退出路径放在显眼的位置。**可读性优先于语法精简。**

# 5. 跳转语句

## 5.1 break

`break` 用于**立即终止**当前所在的**最近一层**循环或 `switch` 语句。

```java
// 在数组中找目标值，找到就停，不再继续扫描
int[] arr = {3, 7, 1, 9};
int target = 9;
int index = -1;
for (int i = 0; i < arr.length; i++) {
    if (arr[i] == target) {
        index = i;
        break;                     // 跳出整个 for，不再比较后面的元素
    }
}
System.out.println(index);         // 3
```

**关键点：`break` 只跳出它所在的那一层。** 嵌套循环里，内层的 `break` 不影响外层：

```java
for (int i = 0; i < 3; i++) {
    for (int j = 0; j < 3; j++) {
        if (j == 1) break;          // 只跳出内层 j 循环
        System.out.println("i=" + i + ", j=" + j);
    }
}
// i=0, j=0
// i=1, j=0
// i=2, j=0
```

外层循环照常跑完三轮，每轮只打印了 `j=0` 那一次。

## 5.2 continue

`continue` 用于**跳过本轮循环体中 `continue` 之后的代码**，直接进入下一轮（对 `for` 而言是先执行条件控制语句再判断条件；对 `while` 而言是回到条件判断）。

```java
// 打印 1~10 中的偶数，跳过奇数
for (int i = 1; i <= 10; i++) {
    if (i % 2 != 0) {
        continue;                   // 奇数直接进入下一轮
    }
    System.out.print(i + " ");      // 2 4 6 8 10
}
System.out.println();
```

三者的作用范围对比：

| 语句 | 作用范围 | 效果 |
|------|----------|------|
| `break` | 当前这一层循环 / `switch` | 整个循环结束，跳到循环之后的代码 |
| `continue` | 当前这一轮 | 本轮剩余代码跳过，继续下一轮 |
| `return` | **整个方法** | 方法立即结束，后面所有代码（含外层循环）都不执行 |

## 5.3 return

`return` 结束**当前方法**。写在循环里，效果是"跳出所有层循环并直接返回"：

```java
public static int findFirstEven(int[] arr) {
    for (int x : arr) {
        System.out.println("检查 " + x);
        if (x % 2 == 0) {
            return x;               // 直接返回，后面的循环与代码全部终止
        }
    }
    return -1;                      // 遍历完都没找到
}

// 调用 findFirstEven(new int[]{1, 3, 4, 6}) 的输出：
// 检查 1
// 检查 3
// 检查 4
// 返回 4
```

**把多层循环封装成方法、用 `return` 提前返回**，通常比带标签的 break 更清晰，这也是后面推荐的替代方案。

## 5.4 带标签的 break 与 continue

Java **保留了 `goto` 关键字但禁止使用**，作为补偿提供了**标签加 `break`/`continue`** 的受控跳转。

### 5.4.1 标签能放在哪

标签是一个标识符加冒号，只能放在**循环语句或者代码块之前**：

```java
outer:                              // ✓ 合法：标签在 for 之前
for (int i = 0; i < 3; i++) { }

block: {                            // ✓ 合法：标签在普通代码块之前
    System.out.println("可以配合 break 提前跳出这个块");
}

// 下面都非法：
// int x = 0; label:                // ✗ 标签不能悬空放在普通语句后面
// label: int x = 0;                // ✗ 标签后面必须是循环或代码块
```

### 5.4.2 带标签的 break

从内层直接跳出到**任意外层**，而不是只跳一层。

```java
public class LabeledBreakDemo {
    public static void main(String[] args) {
        outerLoop:
        for (int i = 0; i < 3; i++) {
            for (int j = 0; j < 3; j++) {
                if (i * j == 4) {
                    System.out.println("命中 i=" + i + ", j=" + j);
                    break outerLoop;      // 跳到 outerLoop 标识的整个循环之外
                }
                System.out.println("i=" + i + ", j=" + j);
            }
        }
        System.out.println("循环结束");
    }
}
```

实际输出（`i*j == 4` 在 `i=2, j=2` 时成立）：

```text
i=0, j=0
i=0, j=1
i=0, j=2
i=1, j=0
i=1, j=1
i=1, j=2
i=2, j=0
i=2, j=1
命中 i=2, j=2
循环结束
```

如果这里写的是不带标签的 `break`，只会跳出内层 `j` 循环，外层会继续跑到 `i=3`，输出会多出 `i=3` 那一轮的内容。

### 5.4.3 带标签的 continue

`continue 标签;` 会**跳过当前内层循环的剩余部分，直接进入标签所指外层循环的下一轮**。注意它的语义是"让外层进下一轮"，而不是"跳出循环"——这正是它与带标签 `break` 的分水岭。

```java
outer:
for (int i = 1; i <= 3; i++) {
    for (int j = 1; j <= 3; j++) {
        if (i == j) {
            System.out.println("i == j，跳过 i=" + i + " 这一轮的剩余部分");
            continue outer;            // 剩下的 j 都不跑了，直接开始 i + 1
        }
        System.out.println("i=" + i + ", j=" + j);
    }
}
```

实际输出：

```text
i == j，跳过 i=1 这一轮的剩余部分
i=2, j=1
i == j，跳过 i=2 这一轮的剩余部分
i=3, j=1
i=3, j=2
i == j，跳过 i=3 这一轮的剩余部分
```

逐轮看：`i=1` 时 `j=1` 就命中，立刻跳到 `i=2`；`i=2` 时 `j=1` 打印，`j=2` 命中后跳到 `i=3`（`j=3` 没跑）；`i=3` 时 `j=1`、`j=2` 打印，`j=3` 命中后跳 `i=4`，循环结束。

**两者对比**：

| 语句 | 跳到哪 | 外层循环是否继续 |
|------|--------|-----------------|
| `break 标签;` | 标签标识的循环**之外** | 不继续 |
| `continue 标签;` | 标签标识的循环的**下一轮** | 继续 |

### 5.4.4 标签的取舍

带标签跳转能让多层循环的跳出变得直接，但**可读性差**——调试时"跳到哪了"需要来回找标签，而标签往往和循环体隔着几十行。

> **最佳实践**：现代代码里更推荐两种替代方案。**方案一：把内层逻辑抽成方法，用 `return` 提前返回**（最清晰，也是 IDE 的重构建议）：

```java
// 方案一：抽方法 + return
private static boolean findPair(int limit) {
    for (int i = 0; i < limit; i++) {
        for (int j = 0; j < limit; j++) {
            if (i * j == 4) return true;    // 一次 return 跳出所有层
        }
    }
    return false;
}
```

**方案二：用布尔标志位控制外层条件**：

```java
// 方案二：标志位参与外层循环条件
boolean found = false;
for (int i = 0; i < 3 && !found; i++) {      // 条件里带上标志
    for (int j = 0; j < 3; j++) {
        if (i * j == 4) {
            found = true;
            break;                           // 只跳内层，外层靠条件自然停止
        }
    }
}
System.out.println(found);                   // true
```

只有在**性能敏感的热点代码**（避免方法调用开销）或**逻辑确实很局部**（两三层的小循环，标签和循环在同一屏内）时，标签才是更优选择。

# 6. 常见陷阱

## 6.1 死循环

| 写法 | 是否死循环 | 原因 |
|------|-----------|------|
| `for (;;) { }` | 是 | 条件省略默认 `true` |
| `while (true) { }` | 是 | 最常见的死循环写法 |
| `for (int i = 0; i < 10; i--)` | 是 | 条件控制写反了方向 |
| `while (i < 10) { }`（忘了 `i++`） | 是 | 循环变量永不更新 |
| `for (double d = 0; d != 1.0; d += 0.1)` | 是 | 浮点累加无法精确等于 1.0 |
| `while (i != 10) { i += 2; }`（`i` 从 1 开始） | 是 | 跳过了 10，永远不相等 |

```java
// 浮点做循环条件：经典的"看起来会停其实停不下来"
for (double d = 0.0; d != 1.0; d += 0.1) {
    System.out.println(d);
    // 会一直打印下去
}
```

> **陷阱**：**永远不要用浮点数做循环的终止条件**。`0.1` 在二进制里是无限循环小数，累加 10 次得到的是 `0.9999999999999999` 而不是 `1.0`，所以 `d != 1.0` 永远成立。要循环固定次数就用 `int` 计数器；要比较浮点就用 `Math.abs(d - 1.0) < 1e-9` 这样的误差范围（第 3 篇 2.5）。

**死循环不一定都是 bug**——服务器主循环、事件循环、`while (true)` 配内部 `break` 的选择菜单都是有意为之。判断标准是**内部是否存在可靠的退出路径**，以及这个退出路径在什么条件下才可能不成立。

## 6.2 循环中的变量作用域与闭包

Java 的局部变量没有 C 语言那样的静态存储——循环体每次进入，里面的局部变量都是**重新创建并初始化**的。这一点在闭包场景下尤其重要：

```java
// JDK8 之前：匿名内部类捕获的局部变量必须显式声明 final，写法很啰嗦
// JDK8+：effectively final —— 循环变量每轮都是"新的一份"，可以安全捕获
List<Runnable> tasks = new ArrayList<>();
for (int i = 0; i < 3; i++) {
    int snapshot = i;                  // 每轮一个新变量，值在当轮固定
    tasks.add(() -> System.out.println(snapshot));
}
tasks.forEach(Runnable::run);          // 0 1 2
```

关键在循环体内声明的 `snapshot`：**每一轮都是不同的变量**（Java 语言规范明确规定），所以每个 Lambda 捕获到的值互不影响。如果直接捕获 `i`，会因为 `i++` 破坏了 effectively final 而**编译报错**——这反而是一种保护，避免了 C# 早期版本那种"三个闭包都输出 3"的经典坑。闭包的完整讨论见第 12 篇。

## 6.3 循环中修改集合

```java
List<String> list = new ArrayList<>(List.of("A", "B", "C"));

// ✗ 遍历中用集合自己的方法增删，会抛 ConcurrentModificationException
// for (String s : list) {
//     if ("B".equals(s)) list.remove(s);
// }

// ✓ 写法一：用迭代器自己的 remove
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if ("B".equals(it.next())) {
        it.remove();                   // 迭代器同步了修改计数，安全
    }
}
System.out.println(list);              // [A, C]

// ✓ 写法二：JDK8+ 的 removeIf，最简洁
list.removeIf("C"::equals);
System.out.println(list);              // [A]
```

增强 for 内部用的就是迭代器，遍历中直接调用集合的增删方法会导致迭代器的 `modCount` 校验失败。原因与三种遍历方式的选择见第 9 篇。

## 6.4 switch 里的变量作用域

传统 `:` 写法的 switch，**所有 case 共享同一个作用域**——在一个 case 里声明的变量，在另一个 case 里也能看见：

```java
int n = 2;
switch (n) {
    case 1:
        int x = 10;                  // 作用域是整个 switch 块，不是这个 case
        System.out.println(x);
        break;
    case 2:
        // int x = 20;               // ✗ 编译错误：变量 x 已在作用域中定义过
        // System.out.println(x);    // ✗ 编译错误：变量 x 可能尚未初始化
        break;
}
```

解决办法是**给每个 case 加一对大括号**限定作用域：

```java
switch (n) {
    case 1: {
        int x = 10;                  // ✓ 这一对大括号创造了独立作用域
        System.out.println("case 1: " + x);
        break;
    }
    case 2: {
        int x = 20;                  // ✓ 与上面的 x 不冲突
        System.out.println("case 2: " + x);
        break;
    }
}
// case 2: 20
```

这也是为什么 `->` 箭头写法的多行分支**必须**写大括号——它顺手把作用域问题一起解决了。

## 6.5 其它易错点

```java
// (1) for 的条件用错了边界
int[] arr = {1, 2, 3};
// for (int i = 0; i <= arr.length; i++)   // ✗ i 会取到 3，越界
for (int i = 0; i < arr.length; i++) { }    // ✓

// (2) while 的循环变量在循环体内才初始化，导致条件永远成立
int count;
// while (count < 5) { count = 0; count++; }  // ✗ count 未初始化，编译错误
// 即便写了 int count = 0; 放在循环体内，count 每轮清零，也是死循环

// (3) continue 之前忘了更新条件变量，造成死循环
int k = 0;
while (k < 5) {
    if (k == 2) {
        continue;                              // ✗ k 永远是 2，无限循环
    }
    k++;
}
// 正确写法：在 continue 之前先自增
int m = 0;
while (m < 5) {
    m++;
    if (m == 3) continue;                      // ✓ 条件变量已经更新
    System.out.print(m + " ");                 // 1 2 4 5
}
```

> **陷阱**：`while` 循环里用 `continue` 时，**必须在 `continue` 之前完成条件变量的更新**，否则循环条件永远不变。`for` 循环没有这个问题，因为 `continue` 会照常执行条件控制语句（`i++`）——这是 `for` 相对 `while` 的一个隐性优势，也是"计数循环优先用 `for`"的原因之一。

# 7. 小结

| 主题 | 一句话要点 |
|------|-----------|
| 三种结构 | 顺序、分支、循环；复杂逻辑都是它们的组合，嵌套超三层就该抽方法 |
| if | 条件必须是 `boolean`；多分支**从上到下命中即止**，条件顺序决定语义 |
| 大括号 | **永远写**，即使只有一条语句；省略会引入悬挂 else（绑定最近的 if，缩进骗人） |
| if 的坑 | `if (x);` 后面的代码块无条件执行；boolean 变量写 `if (flag = true)` 能编译且恒真 |
| switch 类型 | `byte`/`short`/`int`/`char` → JDK5 枚举 → JDK7 `String`；**永不支持** `long`、浮点、`boolean` |
| case | 只能是常量表达式，值不能重复；JDK12+ 用 `case 1, 2 ->` 合并多值 |
| case 穿透 | 不写 `break` 就会继续执行下一个 case；多值共用一个代码块是唯一正当用法，其余都是 bug |
| default | 可省略但不建议；位置任意但放最后最清晰；**别留空**，非法输入要显式失败 |
| switch 表达式 | JDK14+ 用 `->`（不穿透）+ `yield`（多语句返回值）；作为表达式时有穷尽性检查，加枚举值会编译报错 |
| 三种循环 | `for` 已知次数、`while` 条件驱动、`do-while` 至少一次；计数循环优先 `for`（`continue` 会执行 `i++`） |
| 增强 for | 遍历数组与 `Iterable`；**拿不到下标、改不到元素、不能边遍历边增删** |
| 跳转语句 | `break` 跳一层、`continue` 跳本轮、`return` 结束方法；`while` 里 `continue` 前必须先更新条件变量 |
| 带标签跳转 | `break 标签` 跳出到标签之外、`continue 标签` 跳到标签的下一轮；**优先用抽方法 + `return` 代替** |
| 死循环 | 别用浮点做循环条件；`for (;;)` 与 `while (true)` 等价；有意死循环必须有明确退出路径 |
| 作用域 | 循环体内声明的变量每轮重置（累加器要放循环外）；switch 的 `:` 写法所有 case 共享作用域，加 `{ }` 隔离 |

下一篇（第 5 篇）讲数组——当同一套处理要作用于"一组数据"而不是"一个数据"时，就需要一个能装下多个元素的容器，而那也第一次把栈和堆的关系摆到了台面上。
