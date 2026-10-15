这一篇是 Java 的地基：注释、关键字、字面量、变量、进制、数据类型、标识符与键盘录入。它们看起来零碎，却决定了后面所有语法的边界——为什么 `byte b = 128;` 编译不过、为什么 `0.1 + 0.2 != 0.3`、为什么 `nextInt()` 后面紧跟 `nextLine()` 会读到一个空字符串，答案全在这八节里。入门阶段把这一篇的每个表格都过一遍，后面写代码时会少踩很多莫名其妙的坑。

# 1. 注释

## 1.1 三种注释形式

Java 只有三种注释写法，但它们服务的对象并不相同：前两种面向"读代码的人"，第三种面向"用这个类的人"。

| 形式 | 写法 | 作用范围 | 主要用途 |
|------|------|----------|----------|
| 单行注释 | `// 注释内容` | 从 `//` 起到本行结束 | 解释某一行、某一个变量的意图 |
| 多行注释（块注释） | `/* 注释内容 */` | 两个标记之间，可跨多行 | 临时注释掉一段代码、写较长的说明 |
| 文档注释（javadoc） | `/** 注释内容 */` | 两个标记之间，可跨多行 | 生成 HTML 格式的 API 文档 |

```java
// 1. 单行注释：最常用，写在代码上方或右边
int age = 18;

/*
   2. 多行注释可以跨很多行。
   注意：它不能嵌套，遇到第一个结束标记就结束了。
*/
int score = 100;

/**
 * 3. 文档注释：javadoc 工具能把它提取成 HTML API 文档，
 *    所以它写的是"给别人用"的说明，而不是"给自己看"的笔记。
 *
 * @param a 被加数
 * @param b 加数
 * @return 两个参数之和
 * @author AsterWinston
 */
public static int add(int a, int b) {
    return a + b;
}
```

文档注释里可以写一组以 `@` 开头的**标签**，javadoc 会按标签类型排版：

| 标签 | 含义 |
|------|------|
| `@param 参数名 说明` | 描述一个方法参数 |
| `@return 说明` | 描述返回值 |
| `@throws 异常类 说明` | 描述可能抛出的异常（`@exception` 同义） |
| `@author 作者` | 作者信息 |
| `@version 版本` | 版本号 |
| `@since 版本` | 从哪个版本开始有 |
| `@deprecated 说明` | 标记已过时，常与 `@Deprecated` 注解一起用 |
| `@see 引用` | 参见某个类或方法 |
| `{@code 代码}` | 行内代码，按等宽字体渲染（注意是花括号） |
| `{@link 类名}` | 行内超链接到另一个类或方法 |

> **最佳实践**：注释要解释"**为什么这么写**"，而不是复述"这行在做什么"。`i++; // i 加一` 是噪音；`i++; // 跳过表头行，数据从第二行开始` 才是有效注释。一个反直觉的常量、一段绕开某个 bug 的逻辑、一个业务上的特殊约定，都值得留注释。

## 1.2 注释真的会被丢弃吗

答案要分三层看，不能一概而论：

1. `//` 和 `/* */` 属于**纯词法层面**的东西。词法分析之后它们就被丢弃，`.class` 里连一个字节都不会留下。
2. `/** */` 文档注释默认**也不会写进 `.class`**，它的去向是 `javadoc` 工具——`javadoc` 直接读**源码**提取注释生成 HTML，而不是从字节码里反推。所以想留文档就必须保留源码（或另发一份 `-sources.jar`）。
3. 但 `.class` 里**并非什么信息都没有**。注解、行号表、局部变量名这些元数据都会被写进字节码，它们经常被误认为"注释残留"。

| `.class` 中可能保留的信息 | 写入条件 | 观察方式 |
|--------------------------|----------|----------|
| `RuntimeVisibleAnnotations` | 注解标注了 `@Retention(RetentionPolicy.RUNTIME)` | `javap -v` |
| `RuntimeInvisibleAnnotations` | 注解是 `CLASS` 保留策略（默认） | `javap -v` |
| 行号表 `LineNumberTable` | 默认开启，可 `-g:none` 关闭 | `javap -l` |
| 局部变量表 `LocalVariableTable` | 需要 `-g` 或 `-g:vars` 编译 | `javap -l` |
| 方法参数名 `MethodParameters` | 需要 `-parameters` 编译 | `javap -v` |

用一个例子验证。下面这个类的单行注释、多行注释、文档注释在编译后全部消失，唯一留下来的是 `@Deprecated` 这个**运行时注解**：

```java
/**
 * 文档注释：会被 javadoc 提取成 HTML，但不会进 .class
 */
@Deprecated                                 // RetentionPolicy.RUNTIME，会进 .class
public class CommentDemo {
    public static void main(String[] args) {
        int x = 1;                          // 单行注释：编译后彻底消失
    }
}
```

```bash
javac -g CommentDemo.java     # -g 生成局部变量表等调试信息，便于观察
javap -v CommentDemo.class    # -v 输出常量池、注解、行号表等全部细节
```

`javap -v` 的输出里能看到类似这样的片段，而注释文字一个字节都找不到：

```text
RuntimeVisibleAnnotations:
  0: #7()
    java.lang.Deprecated
LineNumberTable:
  line 6: 0
  line 7: 4
```

> **注意**："注释会被丢弃"这个说法**只对 `//` 和 `/* */` 成立**。反编译工具看到的 `@Deprecated`、参数名、行号，都是注解和调试信息，不是注释。这也是为什么"靠反编译恢复源码注释"永远做不到——那部分信息在编译期就已经不存在了。

# 2. 关键字与保留字

## 2.1 关键字

**关键字（keyword）** 是 Java 语言预先征用的、有固定含义的单词，**字母全部小写**，不能用来给类、变量、方法命名。JLS 3.9 一共列出 50 个（其中 `const` 和 `goto` 从未被使用），Java 9 起又加入 `_`，合计 51 个：

| 语义类别 | 关键字 | 个数 |
|----------|--------|------|
| 访问控制 | `public` `protected` `private` | 3 |
| 类与接口 | `class` `interface` `enum` `extends` `implements` `abstract` `final` `static` `strictfp` | 9 |
| 对象与实例 | `new` `this` `super` `instanceof` `void` | 5 |
| 包与导入 | `package` `import` | 2 |
| 流程控制 | `if` `else` `switch` `case` `default` `for` `while` `do` `break` `continue` `return` | 11 |
| 异常处理 | `try` `catch` `finally` `throw` `throws` `assert` | 6 |
| 并发与运行环境 | `synchronized` `volatile` `transient` `native` | 4 |
| 基本数据类型 | `byte` `short` `int` `long` `float` `double` `char` `boolean` | 8 |
| 保留未用 | `goto` `const` | 2 |

每个关键字的含义、示例与注意点，第 23 篇有完整的分组速查表，这里只需建立"关键字是按语义分组的"这个印象。

## 2.2 保留字

`const` 和 `goto` 是**保留字（reserved word）**：它们被 Java 保留下来但从未启用，唯一的作用就是"禁止你把它们用作标识符"。

| 保留字 | 预留的用途 | 为什么 Java 不用它 |
|--------|-----------|-------------------|
| `const` | 常量声明 | Java 用 `final`（字段）和 `static final`（类常量）代替 |
| `goto` | 无条件跳转 | 结构化编程认为无条件跳转破坏可读性；Java 只保留了带标签的 `break`/`continue` |

它们和关键字的**效果完全一样**——写成 `int const = 1;` 直接编译错误——差别只在"从未投入使用"这一点上。所以严格说，`const` 和 `goto` 是"关键字里被冻结的两个"。

## 2.3 true、false、null 不是关键字

`true`、`false`、`null` 经常被归到关键字里一起背，但 JLS 把它们单列为**字面量（literal）**：`true`/`false` 叫 *BooleanLiteral*，`null` 叫 *NullLiteral*。

| 名称 | 是否关键字 | 能否作标识符 | 类型 |
|------|-----------|-------------|------|
| `public` | 是 | 否 | 无类型，只是语法标记 |
| `null` | **否**，是字面量 | 否 | **没有类型** |
| `true` / `false` | **否**，是字面量 | 否 | `boolean` |

实际编码中真正的区别只有一个：`null` 是**唯一没有类型的字面量**，所以它只能赋给引用类型——`int i = null;` 编译不过，`Integer i = null;` 才行。

> **注意**：`var`、`yield`、`record`、`sealed`、`permits` 这些既不是关键字也不是字面量，而是**上下文关键字（contextual keyword）**：只在特定位置有特殊含义，放到别处照样能当普通标识符用。它们的完整区别见第 23 篇第 1 节。

# 3. 字面量

## 3.1 字面量的作用

**字面量的作用，就是指明数据在程序中的书写格式。**

**字面量（literal）** 是在源码里直接写出来的、值本身的数据。它是编译器和程序员之间的一个约定：看到 `10` 就认定是 `int`，看到 `10L` 才认定是 `long`，看到 `"10"` 才认定是 `String`。字面量没有名字、没有变量名可寻址，它在编译期就被写进字节码的常量池里。

## 3.2 字面量的种类

| 种类 | 写法示例 | 推断出的类型 | 备注 |
|------|----------|-------------|------|
| 整数类型 | `10`、`100` | `int` | 超出 `int` 范围要用 `L` 后缀转 `long` |
| 小数类型 | `3.14`、`0.5` | `double` | 加 `F` 后缀才是 `float` |
| 字符串类型 | `"hello"` | `String` | 双引号；可以用 `+` 连接两个字符串字面量 |
| 字符类型 | `'A'`、`'1'` | `char` | 单引号；只能包含单个字符 |
| 布尔类型 | `true`、`false` | `boolean` | 只有这两个值 |
| 空类型 | `null` | 无类型 | 只能赋值给引用数据类型 |

```java
public class LiteralDemo {
    public static void main(String[] args) {
        int     i = 100;                  // 整数类型字面量，默认就是 int
        long    l = 100L;                 // 长整型要加 L，且建议大写，避免与数字 1 混淆
        double  d = 3.14;                 // 小数类型字面量，默认就是 double
        float   f = 3.14F;                // 单精度要加 F，不加会报"不兼容的类型：double 无法转换为 float"
        String  s = "hello" + "world";    // 字符串字面量，用 + 连接后是 "helloworld"
        char    c = 'A';                  // 字符字面量，单引号内只能有一个字符
        boolean b = true;                 // 布尔字面量
        String  n = null;                 // 空字面量，只能赋给引用类型

        System.out.println(s + c + i + l + d + f + b + n);
        // 输出：helloworldA1001003.143.14truenull
        // 注意 null 参与字符串拼接时会变成字符串 "null"，这也是它最容易被忽略的行为
    }
}
```

字符字面量和字符串字面量长得像，但**参与运算时的行为完全不同**——`char` 会先提升为 `int`（Unicode 码点），而 `String` 只会拼接：

```java
System.out.println('1' + 0);    // 49：字符 '1' 的 Unicode 码点是 49，做的是加法
System.out.println("1" + 0);    // 10：字符串直接拼接
```

## 3.3 数值字面量的进制前缀与下划线

整数类型的字面量支持四种进制写法，Java 7 起还允许用下划线 `_` 做视觉分组：

| 前缀 | 进制 | 示例 | 十进制值 |
|------|------|------|----------|
| `0b` 或 `0B` | 二进制（Binary） | `0b1010` | 10 |
| `0` | 八进制（Octal） | `012` | 10 |
| 无前缀 | 十进制（Decimal） | `10` | 10 |
| `0x` 或 `0X` | 十六进制（Hexadecimal） | `0xF`、`0xa3` | 15、163 |
| 下划线（可与任意进制组合） | —— | `1_000_000`、`0b1010_1010` | 1000000、170 |

```java
public class RadixLiteral {
    public static void main(String[] args) {
        int  dec = 10;             // 十进制
        int  bin = 0b1010;         // 二进制，前缀 0b
        int  oct = 012;            // 八进制，前缀是数字 0，不是字母 O
        int  hex = 0xF;            // 十六进制，前缀 0x，字母大小写都行
        long big = 1_000_000L;     // 下划线只为人眼分组，编译器直接忽略

        System.out.println(dec + " " + bin + " " + oct + " " + hex + " " + big);
        // 输出：10 10 10 15 1000000
        // 前三个写法不同，值相同，都是十进制 10
    }
}
```

下划线的规则不多，但错一处就编译失败——它**只能出现在数字之间**：

```java
int a = 1_000_000;      // ✓ 合法
int b = 0b1010_1010;    // ✓ 合法，二进制里也能分组
// int c = _1000;       // ✗ 不能放在开头
// int d = 1000_;       // ✗ 不能放在结尾
// int e = 0x_FF;       // ✗ 不能紧跟在进制前缀之后
// int f = 1_.0;        // ✗ 不能紧挨小数点
```

> **陷阱**：八进制以 `0` 开头这个规则极易出事。`int x = 08;` 会直接编译失败，因为八进制里没有 8 这个数码；更隐蔽的是 `int code = 010;` 你以为写的是 10，实际值是 **8**。看到前导零就要立刻警觉，日常写数字不要用八进制。

# 4. 进制与进制转换

## 4.1 四种进制的基本盘

| 进制 | 基数 | 数码 | Java 前缀 | 典型场景 |
|------|------|------|-----------|----------|
| 二进制 | 2 | `0` `1` | `0b` | 位运算、位掩码、权限标记 |
| 八进制 | 8 | `0`~`7` | `0` | 早期 Unix 权限码（`0644`） |
| 十进制 | 10 | `0`~`9` | 无 | 日常业务数值 |
| 十六进制 | 16 | `0`~`9`、`a`~`f` | `0x` | 内存地址、颜色值、字节码、哈希值 |

## 4.2 进制之间的对应关系

进制转换的秘诀其实只有一句：**八进制是 3 位二进制一组，十六进制是 4 位二进制一组**（因为 2³ = 8、2⁴ = 16），所以二进制与它俩之间的互转完全可以查表。

| 十进制 | 二进制 | 八进制 | 十六进制 |
|--------|--------|--------|----------|
| 0 | 0 | 0 | 0 |
| 1 | 1 | 1 | 1 |
| 2 | 10 | 2 | 2 |
| 3 | 11 | 3 | 3 |
| 4 | 100 | 4 | 4 |
| 5 | 101 | 5 | 5 |
| 6 | 110 | 6 | 6 |
| 7 | 111 | 7 | 7 |
| 8 | 1000 | 10 | 8 |
| 9 | 1001 | 11 | 9 |
| 10 | 1010 | 12 | A |
| 11 | 1011 | 13 | B |
| 12 | 1100 | 14 | C |
| 13 | 1101 | 15 | D |
| 14 | 1110 | 16 | E |
| 15 | 1111 | 17 | F |
| 16 | 10000 | 20 | 10 |

拿 `0b1010_1010` 练一遍：按 4 位一组切开是 `1010` `1010`，查表得 `A` `A`，所以等于 `0xAA`；按 3 位一组从右往左切是 `10` `101` `010`，查表得 `2` `5` `2`，所以等于 `0252`。三者换成十进制都是 `128 + 32 + 8 + 2 = 170`。

Java 的 `Integer` 和 `Long` 内置了转换方法，不用自己手算：

```java
public class RadixConvert {
    public static void main(String[] args) {
        // 十进制数字 → 其他进制的字符串
        System.out.println(Integer.toBinaryString(170));   // 10101010
        System.out.println(Integer.toOctalString(170));    // 252
        System.out.println(Integer.toHexString(170));      // aa
        System.out.println(Integer.toString(170, 2));      // 10101010，支持 2~36 任意进制

        // 其他进制的字符串 → 十进制数字
        System.out.println(Integer.parseInt("10101010", 2));   // 170
        System.out.println(Integer.parseInt("252", 8));        // 170
        System.out.println(Integer.parseInt("aa", 16));        // 170

        // decode 能自动识别前缀，但只认 0x / # / 0 三种，不认 0b
        System.out.println(Integer.decode("0xF"));    // 15
        System.out.println(Integer.decode("#F"));     // 15（# 等价于 0x）
        System.out.println(Integer.decode("012"));    // 10（八进制）
        // Integer.decode("0b1010");                  // ✗ 抛 NumberFormatException
    }
}
```

> **陷阱**：`Integer.decode()` **不支持 `0b` 二进制前缀**。它把开头的 `0` 认成八进制前缀，于是 `"0b1010"` 被当成"八进制字符串 `b1010`"解析，直接抛 `NumberFormatException: For input string: "b1010" under radix 8`。要解析二进制字符串只能用 `Integer.parseInt(s, 2)`。

> **注意**：`Integer.parseInt()` 碰到不合法字符会抛 `NumberFormatException`（它是 `IllegalArgumentException` 的子类），属于运行时异常，**不需要也不应该用 try-catch 包住一切**——用户输入这种"外部数据"才需要防御性处理。

## 4.3 为什么 0.1 不能用二进制精确表示

整数在二进制里可以精确表示，小数则不一定。把十进制小数 `0.1` 转成二进制，做法是"**不断乘 2，取整数部分**"：

```text
0.1 × 2 = 0.2   → 取 0，余 0.2
0.2 × 2 = 0.4   → 取 0，余 0.4
0.4 × 2 = 0.8   → 取 0，余 0.8
0.8 × 2 = 1.6   → 取 1，余 0.6
0.6 × 2 = 1.2   → 取 1，余 0.2   ← 又回到了 0.2，开始循环
0.2 × 2 = 0.4   → 取 0，余 0.4
...              → 0.0001100110011... 无限循环
```

余数永远回不到 0，于是得到一个**无限循环小数**。而 `double` 只有 64 位，其中尾数只有 52 位，装不下无限长的二进制串，只能**截断**。因此计算机里存的 `0.1` 其实是一个近似值，写成十进制大约是 `0.1000000000000000055511151231257827`。

```java
public class FloatPrecision {
    public static void main(String[] args) {
        System.out.println(0.1 + 0.2);          // 0.30000000000000004，不是 0.3
        System.out.println(0.1 + 0.2 == 0.3);   // false
        System.out.println(1.0f - 0.9f);        // 0.100000024，float 误差更明显

        // 需要精确计算时必须用 BigDecimal，且要用字符串构造器
        java.math.BigDecimal a = new java.math.BigDecimal("0.1");
        java.math.BigDecimal b = new java.math.BigDecimal("0.2");
        System.out.println(a.add(b));           // 0.3
        // new BigDecimal(0.1) 会把 double 的误差一起带进来，输出 0.1000000000000000055511151231257827021181583404541015625
    }
}
```

精度对照：

| 类型 | 位数 | 有效十进制位数 | 能否精确表示 0.1 |
|------|------|----------------|------------------|
| `float` | 32 | 约 7 位 | 否 |
| `double` | 64 | 约 15~16 位 | 否（只是误差更小） |
| `BigDecimal` | 任意 | 任意（构造时指定） | 能，用字符串构造即可 |

> **陷阱**：用 `==` 比较两个浮点数**几乎总是错的**。判断相等要用 `Math.abs(a - b) < 1e-9`（容差比较），涉及金额、利率这类不能有误差的场景则一律用 `BigDecimal`，并显式指定舍入模式（`RoundingMode.HALF_UP`）。

# 5. 变量

## 5.1 变量的三要素与定义格式

变量是内存中一块**有名字、有类型、有值**的存储空间，三要素缺一不可：

- **数据类型**：决定这块空间占多少字节、能存什么样的数据；
- **变量名**：访问这块空间的入口；
- **值**：当前存进去的内容。

定义格式就是原文那一行：

```text
数据类型 变量名 = 数据值;
```

完整的生命周期是"先声明，再赋值，最后使用"三步，声明与赋值也可以合并：

```java
public class VariableDeclare {
    public static void main(String[] args) {
        int age;                  // 1. 声明：申请一块 4 字节空间并起名 age，此时还不能用
        age = 18;                 // 2. 赋值：把 18 写进去
        System.out.println(age);  // 3. 使用：读出 18

        int score = 100;          // 声明与赋值合并，这是最常用的写法
        score = score + 1;        // 重新赋值会覆盖旧值，原来的 100 不再保留
        System.out.println(score); // 101
    }
}
```

## 5.2 使用变量的五条注意事项

| 规则 | 说明 | 反例 |
|------|------|------|
| 只能存一个值 | 变量是"一个盒子"，新值会覆盖旧值 | `int a = 1; a = 2;` 之后 `1` 就丢了 |
| 变量名不允许重复定义 | 同一作用域内不能有两个同名变量 | `int a = 1; int a = 2;` 编译错误 |
| 一条语句可以定义多个变量 | 语法合法，但不推荐，可读性差 | `int a = 1, b = 2, c = 3;` |
| 使用前一定要赋值（初始化） | **局部变量**必须显式初始化；**成员变量**有默认值 | `int a; System.out.println(a);` 编译错误 |
| 有作用域限制 | 通常为其所在的一对花括号 `{}` 之内 | 出了花括号就"找不到符号" |

```java
public class VariableRule {
    static int field;              // 成员变量：不赋值也有默认值 0

    public static void main(String[] args) {
        // 1. 只能存一个值，后赋的值覆盖前一个
        int a = 1;
        a = 2;
        System.out.println(a);             // 2

        // 2. 同一作用域不能重复定义——下面这行会报"已在方法 main 中定义了变量 a"
        // int a = 3;

        // 3. 一条语句定义多个变量，合法但不推荐
        int x = 1, y = 2, z = 3;
        System.out.println(x + y + z);     // 6

        // 4. 成员变量有默认值，局部变量必须手动初始化
        System.out.println(field);         // 0
        // int local;
        // System.out.println(local);      // 编译错误：变量 local 可能尚未初始化
    }
}
```

各种数据类型的成员变量默认值：

| 类型 | 默认值 |
|------|--------|
| `byte` / `short` / `int` / `long` | `0`（`long` 是 `0L`） |
| `float` / `double` | `0.0f` / `0.0d` |
| `char` | `'\u0000'`（即数字 0，不是字符 `'0'`） |
| `boolean` | `false` |
| 所有引用类型 | `null` |

> **注意**：为什么局部变量不给默认值？因为局部变量的生命周期极短、作用域极小，编译器**能够**通过数据流分析确认它是否在被赋值前使用；而成员变量的赋值时机分散在多个方法里，编译器无法保证，只能统一给默认值。这个设计差异也解释了为什么"局部变量未初始化"是编译期错误，而"成员变量读到 0"是个静默的错误。

## 5.3 变量的作用域

**作用域（scope）** 指的是变量能被访问的代码范围。Java 采用**块级作用域**：变量的作用域就是它所在的那一对 `{}`，从声明位置开始一直到这对花括号结束。

```java
public class ScopeDemo {
    public static void main(String[] args) {
        int outer = 1;                            // 作用域：整个 main 方法体
        {
            int inner = 2;                        // 作用域：仅这个内层花括号
            System.out.println(outer + inner);    // 3：内层可以访问外层变量
            // int outer = 9;                     // ✗ 编译错误：不能在嵌套块中重复声明同名局部变量
        }
        // System.out.println(inner);             // ✗ 编译错误：找不到符号——inner 出了花括号就失效
        System.out.println(outer);                // 1
    }
}
```

关于作用域有三点值得记住：

1. **内层能看见外层，外层看不见内层**——变量的可见性是"由内向外单向"的；
2. **Java 不允许内层遮蔽外层的局部变量**（C++ 允许），所以 `int outer = 9;` 在内层块里会报错，这避免了同名变量带来的困惑；
3. **循环变量属于循环体**——`for (int i = 0; ...)` 里的 `i` 出了循环就不可见，所以两个并列的 `for` 循环可以用同一个名字 `i`，互不冲突。

> **最佳实践**：**把变量的作用域压到最小**。声明得离使用点越近越好，用一个 `{}` 包住不再需要的变量。这样变量的存活时间短、被误改的可能小、垃圾回收也更容易判定它已不可达。

# 6. 数据类型

## 6.1 八种基本数据类型

Java 的基本数据类型**只有八种**，字节数固定、不依赖操作系统（这也是 Java "一次编写，到处运行" 的基础之一）。

| 类型 | 字节数 | 位数 | 取值范围 | 默认值 | 包装类 | 字面量后缀 |
|------|--------|------|----------|--------|--------|-----------|
| `byte` | 1 | 8 | -128 ~ 127 | `0` | `Byte` | 无 |
| `short` | 2 | 16 | -32768 ~ 32767 | `0` | `Short` | 无 |
| `int` | 4 | 32 | -2147483648 ~ 2147483647 | `0` | `Integer` | 无（默认） |
| `long` | 8 | 64 | -9223372036854775808 ~ 9223372036854775807 | `0L` | `Long` | `L`（建议大写） |
| `float` | 4 | 32 | ±3.4E-38 ~ ±3.4E+38，约 7 位有效数字 | `0.0f` | `Float` | `F` |
| `double` | 8 | 64 | ±4.9E-324 ~ ±1.7E+308，约 15~16 位有效数字 | `0.0d` | `Double` | `D`（默认） |
| `char` | 2 | 16 | 0 ~ 65535（**无符号**，对应 Unicode 字符） | `'\u0000'` | `Character` | 无 |
| `boolean` | 1 | —— | `true` / `false` | `false` | `Boolean` | 无 |

几点补充说明：

- `char` **只能赋值正数**（0 ~ 65535），因为它本质是一个**无符号的 16 位整数**，对应一个 Unicode 码点；没有负数，也就没有"负的字符"。
- `boolean` 在 JVM 规范里**没有明确规定字节数**，HotSpot 中单独存字段时按 1 字节处理，但在数组中每个元素固定占 1 字节。它不是 0/1 的整数，Java 也**不允许** `boolean` 与任何数值类型互转。
- `long` 和 `float` 的后缀建议**用大写**：`100l` 和 `1001` 肉眼几乎分不出，`3.14f` 和 `3.14F` 倒无所谓，但统一大写是好习惯。

```java
public class DataTypeDemo {
    public static void main(String[] args) {
        byte    b     = 127;              // byte 的上限，写成 128 直接编译失败
        short   s     = 32767;
        int     i     = 100;
        long    l     = 100L;             // 加 L 才是 long
        float   f     = 3.14F;            // 加 F 才是 float
        double  d     = 3.14;
        char    c     = 'A';              // 本质是 0~65535 的无符号整数
        boolean flag  = false;

        System.out.println(Integer.MAX_VALUE);   // 2147483647
        System.out.println(Integer.MIN_VALUE);   // -2147483648
        System.out.println(Long.MAX_VALUE);      // 9223372036854775807
        System.out.println((int) c);             // 65：char 可以当整数用，'A' 的码点是 65
        System.out.println((char) 66);           // B：反过来也能转

        // 溢出演示：int 超出范围不会报错，而是静默环绕
        int max = Integer.MAX_VALUE;
        System.out.println(max + 1);             // -2147483648
    }
}
```

> **陷阱**：**整数溢出是静默的**。`Integer.MAX_VALUE + 1` 不会抛异常，而是变成一个负数，这种 bug 在业务里极难发现（比如累加金额、计算分页偏移）。涉及大数时用 `Math.addExact()`，它会在溢出时抛 `ArithmeticException`；或者直接换 `long` / `BigInteger`。

## 6.2 自动类型提升与强制类型转换

当**取值范围小**的类型和**取值范围大**的类型一起运算时，Java 会自动把小类型提升上去，规则是一条链：

```text
byte → short → int → long → float → double
         char → ↗
```

两级提升要分清楚：

1. **一元数值提升**：`byte`、`short`、`char` 在参与任何算术运算前，**先被提升为 `int`**，然后再参与运算。
2. **二元数值提升**：两个操作数按 "double > float > long > int" 的顺序取较宽的那个。

| 运算 | 结果类型 | 说明 |
|------|----------|------|
| `byte + byte` | `int` | 两个 `byte` 先各自提升为 `int`，再相加 |
| `short + char` | `int` | `char` 也先转 `int`（`'A'` → `65`） |
| `int + long` | `long` | 向宽的一方看齐 |
| `long + float` | `float` | `long` 转 `float` 是自动的，但**可能丢精度** |
| `float + double` | `double` | 精度最高的一方胜出 |

```java
public class PromotionDemo {
    public static void main(String[] args) {
        byte b1 = 10, b2 = 20;
        // byte b3 = b1 + b2;               // ✗ 编译错误：b1 + b2 的结果类型已经是 int
        int  b3 = b1 + b2;                  // ✓ 用 int 接
        byte b4 = (byte) (b1 + b2);         // ✓ 想要 byte 就必须强制转换

        char c  = 'A';
        int  ci = c + 1;                    // char 参与运算先转 int：65 + 1
        System.out.println(ci);             // 66
        System.out.println((char) ci);      // B：再强转回 char

        long  big   = 1L << 62;             // 2^62
        float rough = big;                  // long → float 自动提升，但 float 只有 24 位有效尾数
        System.out.println(big);            // 4611686018427387904
        System.out.println((long) rough);   // 4611686018427387904（2^62 恰好能精确表示，换个数就不一定）

        int  i        = 300;
        byte overflow = (byte) i;           // 强制转换：直接截断，只保留低 8 位
        System.out.println(overflow);       // 44，因为 300 的低 8 位是 0x2C
    }
}
```

有一个容易被忽略的细节：**常量在编译期就能确定值**，所以字面量赋值不受上面的规则限制：

```java
byte b1 = 10;         // ✓ 字面量 10 在 byte 范围内，编译器允许
byte b2 = 10 + 20;    // ✓ 编译期常量折叠成 30，仍在 byte 范围内
byte b3 = 100 + 100;  // ✗ 编译错误：常量折叠结果是 200，超出 byte 范围
short s = 'A';        // ✓ char 字面量 65 在 short 范围内
```

`byte` 的复合赋值运算符还带**隐式强转**，这是它和 `b = b + 1` 的关键区别：

```java
byte b = 10;
b += 1;             // ✓ 等价于 b = (byte) (b + 1)，复合赋值自带隐式转换
// b = b + 1;       // ✗ 这句编译不过，因为 b + 1 是 int
```

两种转换的对比：

| 维度 | 自动类型提升（隐式） | 强制类型转换（显式） |
|------|--------------------|--------------------|
| 方向 | 小范围 → 大范围 | 大范围 → 小范围 |
| 语法 | 直接写，无需标记 | 目标类型必须写在括号里：`(byte) i` |
| 风险 | `long` → `float`/`double` 可能**丢精度** | 可能**溢出、截断、符号反转** |
| `boolean` | 不参与任何数值转换 | 同样不参与 |

## 6.3 引用数据类型与 null

与八种基本类型相对的，是**引用数据类型**：类、接口、数组、枚举、注解、记录类都属于此列。它和基本类型最大的区别是——变量里存的**不是数据本身，而是对象在堆中的地址**。

| 维度 | 基本数据类型 | 引用数据类型 |
|------|-------------|-------------|
| 变量中存的内容 | 值本身 | 对象的地址（引用） |
| 种类 | 固定 8 种 | 无限多（所有类、接口、数组……） |
| 成员变量默认值 | `0` / `false` / `'\u0000'` | `null` |
| 局部变量 | 必须显式初始化 | 必须显式初始化 |
| 内存位置 | 变量在栈上，值也在栈上 | 变量（引用）在栈上，对象在堆上 |
| 是否涉及 GC | 否 | 是，堆上的对象由垃圾回收器管理 |

```java
public class ReferenceDemo {
    public static void main(String[] args) {
        int[] arr = {1, 2, 3};          // 数组是引用类型
        String s = "hello";             // String 也是引用类型

        int[] arr2 = arr;               // 复制的是"地址"，不是数组内容
        arr2[0] = 99;
        System.out.println(arr[0]);     // 99：两个变量指向堆上的同一个数组

        String t = null;                // 引用类型可以指向"没有任何对象"
        System.out.println(t == null);  // true
        // System.out.println(t.length());   // NullPointerException：对 null 调用方法
    }
}
```

> **陷阱**：`null` 是**没有类型**的，所以它不能赋给任何基本类型（`int i = null;` 编译失败），只能给引用类型。另外两个常被忽略的点：`null instanceof X` 的结果**恒为 `false`**（不会抛异常）；把 `null` 传进重载方法时，编译器会挑"最具体"的那个重载，有时会挑中你不想调的那个，这在 `println(null)` 之类的地方会引发歧义错误。

# 7. 标识符

## 7.1 硬性规则

**标识符（identifier）** 就是类、方法、变量、包等元素的名字。硬性规则只有四条，违反了就是编译错误：

| 规则 | 说明 | 合法示例 | 非法示例 |
|------|------|----------|----------|
| 由数字、字母、下划线 `_`、美元符 `$` 组成 | 字母包含 Unicode 字母，中文也算 | `age`、`_count`、`$price`、`年龄` | `my-age`、`user name` |
| 不能以数字开头 | 第一个字符必须是字母、`_` 或 `$` | `age2`、`_2x` | `2age` |
| 不能是关键字 | 51 个关键字 + `true`/`false`/`null` 都不可用 | `clazz`、`klass` | `class`、`null` |
| 区分大小写 | `age` 和 `Age` 是两个不同的变量 | `name` 与 `Name` 可并存 | —— |

Java 9 起还有一个额外的约束：**单独一个下划线 `_` 不能再作为标识符**（`int _ = 1;` 会编译错误），因为它被预留作"未命名变量"；`_count`、`a_b` 这类含下划线的名字不受影响。

```java
public class IdentifierDemo {
    public static void main(String[] args) {
        int age   = 18;    // ✓ 字母开头
        int _count = 1;    // ✓ 下划线开头
        int $price = 2;    // ✓ 美元符开头（框架内部生成的类名常用，业务代码不推荐）
        int age2  = 3;     // ✓ 数字可以出现在中间或结尾
        int 年龄   = 4;    // ✓ 中文也是 Unicode 字母，语法合法，但强烈不推荐
        System.out.println(age + _count + $price + age2 + 年龄);   // 28

        // int 2age = 5;        // ✗ 不能以数字开头
        // int class = 6;       // ✗ 不能是关键字
        // int my-age = 7;      // ✗ 不能包含连字符（会被当成减法）
        // int _ = 8;           // ✗ Java 9 起单独的下划线不再是合法标识符
    }
}
```

> **注意**：`$` 虽然在语法上合法，但**业务代码里不要用**。编译器在内层类、Lambda、`switch` 表达式等场景会自动生成 `$` 开头的名字（如 `Outer$1`），自己再用 `$` 容易撞名，也会让反编译和调试变得混乱。

## 7.2 软性规范

硬性规则保证"能编译"，软性规范保证"人能看懂"。业界通用的是**驼峰命名法**：

| 场景 | 规范 | 示例 |
|------|------|------|
| 类名、接口名、枚举名 | **大驼峰**（PascalCase）：所有单词首字母大写 | `ClassName`、`MyFirstClass`、`UserService` |
| 方法名、变量名 | **小驼峰**（camelCase）：第一个单词全小写，其余单词首字母大写 | `variableName`、`runMethod`、`findById` |
| 常量（`static final`） | 全大写 + 下划线分隔 | `MAX_VALUE`、`DEFAULT_TIMEOUT` |
| 包名 | 全小写，域名倒写 | `com.example.demo` |
| 抽象类 | 以 `Abstract` / `Base` 开头 | `AbstractList`、`BaseController` |
| 异常类 | 以 `Exception` 结尾 | `OrderNotFoundException` |
| 布尔变量或方法 | 用 `is` / `has` / `can` 开头 | `isDeleted`、`hasNext`、`canWrite` |
| 测试类 | 以 `Test` 结尾 | `UserServiceTest` |
| 泛型类型参数 | 单个大写字母 | `T`（类型）、`E`（元素）、`K`/`V`（键值） |

> **最佳实践**：命名要**见名知意**，长度以"不用看注释就能懂"为准。`int a1, a2;` 一周后自己都看不懂，`int retryCount, maxRetries;` 则一读即明。缩写要谨慎——`id`、`url`、`api` 这类业界通用缩写可以直接用，`usr`、`mgr`、`cnt` 这种自创缩写只会增加阅读成本。

# 8. 键盘录入

## 8.1 Scanner 的三步走

`Scanner` 是 `java.util` 包下最常用的输入类，固定三步：

| 步骤 | 代码 | 说明 |
|------|------|------|
| 1. 导包 | `import java.util.Scanner;` | 必须写在类定义的上方 |
| 2. 创建对象 | `Scanner sc = new Scanner(System.in);` | `System.in` 代表标准输入流（键盘） |
| 3. 接收数据 | `sc.nextInt()`、`sc.nextLine()` 等 | 不同方法读不同类型的值 |

常用的读值方法：

| 方法 | 读取内容 | 结束条件 |
|------|----------|----------|
| `nextInt()` | 一个 `int` | 遇到空白字符（空格、Tab、换行） |
| `nextLong()` / `nextDouble()` / `nextFloat()` | 对应的数值类型 | 同上 |
| `nextBoolean()` | 一个 `boolean` | 同上 |
| `next()` | 一个"单词"（不含空白） | 遇到空白字符 |
| `nextLine()` | **一整行**（可以包含空格） | 遇到换行符，且**会读走换行符** |
| `hasNextInt()` | 下一个是不是整数 | 返回 `boolean`，用于校验输入 |

```java
import java.util.Scanner;      // 1. 导包：必须写在类定义的上方

public class ScannerDemo {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);      // 2. 创建对象，绑定标准输入

        System.out.print("请输入姓名：");
        String name = sc.nextLine();              // 读一整行，可以带空格

        System.out.print("请输入年龄：");
        int age = sc.nextInt();                   // 读一个整数

        System.out.print("请输入身高（米）：");
        double height = sc.nextDouble();          // 读一个小数

        System.out.println("姓名：" + name + "，年龄：" + age + "，身高：" + height);
        sc.close();                               // 3. 用完关闭，释放底层资源
    }
}
```

## 8.2 nextInt() 后接 nextLine() 的经典坑

这是入门阶段踩得最多、也最莫名其妙的一个坑：**`nextInt()` 只读走数字，把回车符留在缓冲区里**，紧接着的 `nextLine()` 遇到这个残留的换行符就立刻返回一个**空字符串**，用户根本没机会输入。

```java
Scanner sc = new Scanner(System.in);

int age = sc.nextInt();          // 用户输入 "18" 并回车：读走 "18"，"\n" 留在缓冲区
String name = sc.nextLine();     // 立刻读到缓冲区里那个 "\n"，返回空字符串
System.out.println("[" + name + "]");   // 输出：[]   —— 看起来"跳过"了这次输入
```

根因在于这两类方法对**分隔符**的处理不同：

| 方法 | 是否消费掉结尾的换行符 |
|------|----------------------|
| `nextInt()` / `next()` 等 | **不消费**，换行符留在缓冲区 |
| `nextLine()` | **消费**，读到换行符并把它一起读走 |

两种修法：

```java
// 方案一（最常用）：nextInt() 之后再补一次 nextLine()，把残留的换行符吃掉
int age = sc.nextInt();
sc.nextLine();                   // 专门用来"吃掉"换行符，返回值直接丢弃
String name = sc.nextLine();     // 这次才真正等待用户输入

// 方案二（最稳妥）：统一用 nextLine() 读，再手动解析类型
String ageText = sc.nextLine();
int age2 = Integer.parseInt(ageText.trim());   // trim 去掉首尾空白，避免解析失败
String name2 = sc.nextLine();
```

| 读法 | 优点 | 缺点 |
|------|------|------|
| 全部用 `nextLine()` + 手动解析 | 完全不存在缓冲区残留问题，`next()` 与 `nextLine()` 的语义差被抹平 | 要自己 `parseInt`，格式不对会抛 `NumberFormatException` |
| 数值方法 + 补 `nextLine()` | 代码短，类型自动转换 | 必须记得补那行，漏写就是上面那个坑 |

> **最佳实践**：**一个 `Scanner` 只用一种读法**。如果程序里既要读数字又要读字符串，就统一用 `nextLine()` 读进来自己解析——代码长一点，但永远不会出现"少读了一个换行"这种诡异 bug。

# 9. 小结

| 主题 | 一句话要点 |
|------|-----------|
| 注释 | 三种形式：`//`、`/* */`、`/** */`；前两种编译后彻底消失，文档注释由 `javadoc` 从**源码**提取，`.class` 里留着的是注解和调试信息 |
| 关键字 | 51 个，全小写；`const`、`goto` 是保留未用的；`true`/`false`/`null` 是**字面量**不是关键字；`var`/`yield`/`record` 是上下文关键字 |
| 字面量 | 六类：整数、小数、字符串、字符、布尔、空；`0b`/`0`/`0x` 是进制前缀，`_` 只能出现在数字之间 |
| 进制 | 三/四位二进制一组就是八/十六进制；`0.1` 的二进制是无限循环因而无法精确表示，比较浮点数不能用 `==` |
| 变量 | 三要素 = 类型 + 名字 + 值；局部变量必须先初始化，成员变量有默认值；作用域就是它所在的那对花括号，且**不能遮蔽外层同名局部变量** |
| 数据类型 | 八种基本类型字节数与范围要背下来；`byte`/`short`/`char` 参与运算先升 `int`；整数溢出是静默的；引用类型存地址，默认值是 `null` |
| 标识符 | 硬性规则四条 + 大驼峰/小驼峰/常量全大写的软性规范；Java 9 起单独的 `_` 不再合法 |
| 键盘录入 | `Scanner` 三步走；`nextInt()` 不消费换行符，后面跟 `nextLine()` 会读到空串，统一读法即可根治 |

下一篇（第 23 篇）把这 51 个关键字按语义分组逐个讲清，补上本篇只列了类别、没展开的那张表。
