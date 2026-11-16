`API`（Application Programming Interface，应用程序编程接口）是 Java 官方已经写好、开发者直接调用的类与接口的统称，而 `String` 是其中使用频率最高、也最容易踩坑的一个。本篇先讲清**怎么查 API、怎么读 JDK 源码里的 API**，再围绕 `String` 的**不可变性**这条主线，把**字符串常量池、`==` 与 `equals`、拼接在编译期与运行期的差异、`StringBuilder` 的扩容**讲透，最后落到**包装类**以及它带来的**自动装箱拆箱 NPE** 和 **`Integer` 缓存陷阱**。

# 1. API 概述

## 1.1 什么是 API

**API 就是别人已经写好的、你可以直接调用的类和方法**。在 Java 里，这句话具体指 JDK 提供的成千上万个类：`String`、`ArrayList`、`HashMap`、`LocalDate` 都是 API 的一部分。学习 Java 的一大半工作，其实不是"写代码"，而是"知道有哪些现成的类可以用、它们分别解决什么问题"。

> **最佳实践**：养成"先查再写"的习惯。要实现一个功能（比如字符串去空白、集合排序、日期格式化），先花 30 秒在 JDK 文档里搜关键词，通常会发现标准库早就写好了，而且比自己手写的更健壮。自己造轮子往往在边界条件上出问题。

## 1.2 怎么查 API：javadoc

JDK 自带的文档叫 **javadoc**，是对外说明类与方法的权威来源。查阅时重点看四样东西：

| 位置 | 看什么 |
|------|--------|
| 类声明 | 是不是 `final`（能否继承）、实现了哪些接口、有没有构造方法 |
| 方法签名 | 参数类型、返回值类型、是否 `static`、是否抛受检异常 |
| 方法说明 | 这个方法是干什么的、边界条件怎么处理 |
| `Throws` 段落 | 什么情况下会抛异常，这是写健壮代码的关键信息 |

IDE 里把光标停在方法上按 `Ctrl + Q`（IntelliJ IDEA）即可看到同样的文档；想看实现就 `Ctrl + B` 跳到源码——**读 JDK 源码是提升 Java 水平最直接的途径**，本篇后面分析 `String` 不可变、`StringBuilder` 扩容时都会直接引用源码。

## 1.3 `java.lang` 包为什么不用 import

`java.lang` 包由 **JVM 自动导入**，其中的类在任何 Java 文件里都能直接使用：`String`、`Object`、`System`、`Integer`、`Math`、`Thread`、`Exception` 等都在这个包里。其他包（`java.util`、`java.io`、`java.time` 等）必须显式 `import`。

> **注意**：`java.lang` 的自动导入是**编译器行为**，不是"这些类不用加载"。它只影响书写便利，不影响性能。正因为 `String`、`Object` 全在 `java.lang` 里，它们才是 Java 中出现频率最高的两个类。

# 2. String 类

## 2.1 String 的不可变性

`String` 是 Java 中最常用的引用数据类型之一，它最核心的特性是**不可变（immutable）**：**一个 `String` 对象创建之后，它代表的那串字符就再也不会改变**。所有看起来像"修改"的方法——`substring`、`replace`、`toUpperCase`、`concat`——都不是在原对象上动手，而是**返回一个全新的 `String` 对象**，原对象原封不动。

这一点必须先建立直觉，否则后面所有的内存与性能问题都无从谈起：

```java
String s = "hello";
String upper = s.toUpperCase();   // 没有改变 s，而是返回了新对象
System.out.println(s);            // hello —— s 依然是原来的内容
System.out.println(upper);        // HELLO
System.out.println(s == upper);   // false —— 两个不同的对象
```

### 2.1.1 不可变是怎么做到的：源码层面

"不可变"不是靠约定，而是靠源码里四道锁共同保证的：

```java
// JDK8 及以前
public final class String implements java.io.Serializable, Comparable<String>, CharSequence {
    private final char[] value;   // 字符真正存在这个数组里
    private int hash;             // 缓存 hashCode，默认 0，算过一次就不再算
}

// JDK9 及以后：压缩字符串（Compact Strings）优化
public final class String implements java.io.Serializable, Comparable<String>, CharSequence {
    private final byte[] value;   // 底层改成 byte[]
    private final byte coder;     // LATIN1=0：每字符 1 字节；UTF16=1：每字符 2 字节
    private int hash;
}
```

四道锁分别是：

1. **类被 `final` 修饰**——不能被继承，杜绝子类重写方法去篡改行为；
2. **数据字段 `private final`**——`final` 保证引用不可变，`private` 保证外部碰不到这个数组；
3. **类内部不提供任何修改内容的方法**——没有 `setCharAt`、没有 `append`；
4. **不对外泄露内部数组**——构造函数里对传入的数组做 `Arrays.copyOf` 拷贝，`toCharArray()` 返回的也是**副本**而不是原数组。

> **注意**：这里有一个常见误解——"`final` 保证了不可变"。其实 `final char[] value` 只保证 **`value` 这个引用不能指向别的数组**，数组里的元素照样可以被改写。真正让它不可变的是 **`private` + 不提供修改方法 + 不泄露引用** 这套组合。换句话说，**`final` 只是必要条件，不是充分条件**。

> **注意**：JDK9 的压缩字符串不是"改语法"，而是**省内存**。以前每个字符固定占 2 字节（`char` 是 UTF-16 码元），纯英文的字符串有一半空间是浪费的。JDK9 改成：内容全是 Latin-1（即每个字符 0~255）时用 `byte[]` 每字符 1 字节存，否则用 2 字节。对使用者完全透明，`length()`、`charAt()` 的行为不变。

### 2.1.2 不可变带来的四个好处

| 好处 | 原因 |
|------|------|
| **线程安全** | 内容永远不变，多个线程同时读同一个 `String` 不可能读到"中间状态"，天然安全，无需加锁 |
| **可安全作 `HashMap` 的 key** | `hashCode` 会被缓存且永不变化，key 的哈希桶位置稳定 |
| **可复用省内存** | 正因为不可变，相同字面量可以共享同一个对象（字符串常量池，见 2.2） |
| **安全性** | 数据库连接串、文件路径、类名这类"校验之后就要用"的值，不会在校验之后被偷偷改掉 |

`hashCode` 的缓存值得单独说一句。`String` 重写了 `hashCode`，按字符逐位计算：

```java
// String.hashCode() 的简化逻辑
public int hashCode() {
    int h = hash;                    // 先读缓存的 hash 字段（默认 0）
    if (h == 0 && value.length > 0) {
        for (byte v : value) {
            h = 31 * h + (v & 0xff); // 31 是奇素数，散列效果好且能被 JIT 优化成位移
        }
        hash = h;                    // 写回缓存，下次直接返回
    }
    return h;
}
```

因为字符串不可变，这个哈希值算一次就永远有效，所以缓存是安全的——这也是 `String` 特别适合做 `HashMap` 键的重要原因。

## 2.2 字符串常量池与两种创建方式

### 2.2.1 字符串常量池（String Pool）

**字符串常量池**是一块专门存放字符串字面量的特殊内存区域。它的存在就是为了**复用**：程序里 `"abc"` 可能出现在十几个地方，如果每个都新建对象就太浪费了，所以 JVM 让它们全部指向常量池里的同一个对象。

它的位置在版本间变过：

| 版本 | 常量池位置 |
|------|-----------|
| JDK6 及以前 | 方法区（永久代，`PermGen`） |
| JDK7 及以后 | **堆内存**中 |

> **注意**：这个改动很关键——JDK7 把常量池搬到堆里，就是为了让不再被引用的字符串能被 **GC 正常回收**。永久代几乎不参与 GC，大量动态生成的字符串（`intern` 的、拼接出来的）会把它撑爆。这是"内存永久代 → 元空间"演进的一部分。

### 2.2.2 两种创建方式

```java
// 方式一：直接赋值——对象进常量池
String s1 = "hello";
String s2 = "hello";
System.out.println(s1 == s2);   // true，两个变量指向常量池里同一个对象

// 方式二：new——每次都在堆上新建
String s3 = new String("hello");
String s4 = new String("hello");
System.out.println(s3 == s4);        // false，堆上两个不同的对象
System.out.println(s3 == s1);        // false，一个堆对象，一个常量池对象
System.out.println(s3.equals(s1));   // true，内容相同
```

直接赋值时，JVM 的执行顺序是：**先去常量池里找有没有 `"hello"`**——有就直接返回它的引用；没有就在池里创建 `"hello"` 再返回引用。

`new` 方式则不同，它一定会走 `String` 的构造方法，在**堆**上开辟新空间。`new` 的四个常用构造方法：

| 构造方法 | 作用 | 典型场景 |
|----------|------|----------|
| `public String()` | 创建空字符串 `""` | 极少用，`""` 更直观 |
| `public String(String original)` | 根据已有字符串创建新对象 | 需要一份独立副本时 |
| `public String(char[] chs)` | 根据字符数组创建 | 把 `char[]` 转成字符串 |
| `public String(byte[] chs)` | 根据字节数组创建 | 网络、文件读到的 `byte[]` 解码成字符串 |

```java
char[] chars = {'J', 'a', 'v', 'a'};
String s = new String(chars);
System.out.println(s);              // Java

byte[] bytes = {104, 101, 108, 108, 111};   // 字符 'h','e','l','l','o' 的 ASCII 码
System.out.println(new String(bytes));      // hello —— 按默认字符集解码
System.out.println(new String(bytes, java.nio.charset.StandardCharsets.UTF_8)); // hello
```

> **陷阱**：`new String(byte[])` 用的是**平台默认字符集**。同一段字节在 Windows（GBK）和 Linux（UTF-8）上可能解出完全不同的字符串。处理文本 IO、网络报文时应显式指定 `StandardCharsets.UTF_8`，这也是第 16 篇 IO 会反复强调的一点。

### 2.2.3 `new String("abc")` 到底创建了几个对象

这是高频面试题，答案分两种情况：

| 常量池中是否已有 `"abc"` | 创建的对象数 | 说明 |
|--------------------------|--------------|------|
| **没有** | **2 个** | 先在常量池创建 `"abc"`，再在堆上 `new` 一个副本 |
| **已有** | **1 个** | 常量池复用，只在堆上 `new` 一个 |

```java
// 假设程序运行到此为止还没有出现过 "abc"
String s = new String("abc");   // 创建 2 个对象：常量池 1 个 + 堆 1 个
```

> **注意**：不要把 `String s = "abc"` 和 `String s = new String("abc")` 的"对象数"记混。前者只创建 **0 或 1 个**（常量池没有才创建，有就复用），后者至少要有 **1 个堆对象**，可能额外再补一个常量池对象。JDK8 的 `new String(String)` 源码就是 `this.value = original.value; this.hash = original.hash;`——**直接共享了底层数组**，因为反正不可变，共享是安全的。

### 2.2.4 `intern()`：手动入池

`intern()` 的作用是**把字符串对象放进常量池，并返回池中那个对象的引用**：

```java
String s1 = new String("hello");   // 堆上的对象
String s2 = s1.intern();           // 返回常量池里 "hello" 的引用
String s3 = "hello";               // 也指向常量池的 "hello"
System.out.println(s2 == s3);      // true
System.out.println(s1 == s2);      // false，s1 是堆对象
```

> **行业实践**：`intern()` 在**大量重复字符串**的场景能省内存（比如解析日志时百万个重复的字段名），但它会占用常量池、且 JDK6 时代有性能和内存问题。**日常业务代码里几乎不需要主动调用 `intern()`**，了解它能解释清楚 `==` 的行为即可。

## 2.3 判断与查找类方法

### 2.3.1 判断类方法

判断方法都返回 `boolean`，用于"校验/分支"，是所有业务逻辑里出现频率最高的一类。

| 方法 | 说明 |
|------|------|
| `boolean isEmpty()` | 是否为空字符串，等价于 `length() == 0` |
| `boolean isBlank()` | JDK11+，是否为空白（只含空格、`\t`、`\n` 等） |
| `boolean equals(Object obj)` | 内容是否相等，**区分大小写** |
| `boolean equalsIgnoreCase(String s)` | 内容是否相等，**忽略大小写** |
| `boolean contains(CharSequence s)` | 是否包含指定子串 |
| `boolean startsWith(String prefix)` | 是否以指定前缀开头 |
| `boolean endsWith(String suffix)` | 是否以指定后缀结尾 |
| `boolean matches(String regex)` | 是否匹配正则表达式（**整串匹配**） |

```java
String s = "  ";
System.out.println(s.isEmpty());   // false，长度为 2
System.out.println(s.isBlank());   // true，全是空白（JDK11+）

String name = "Java";
System.out.println(name.contains("av"));       // true
System.out.println(name.startsWith("Ja"));     // true
System.out.println(name.endsWith("va"));       // true
```

> **陷阱**：`isEmpty()` 只判断长度是否为 0，**不判断是否只有空格**。做表单校验时，用户输入 `"   "`（几个空格）用 `isEmpty()` 会放过去，应该用 `isBlank()`。而 `isBlank()` 需要 JDK11+，老项目里要用 `s.trim().isEmpty()` 替代。

> **陷阱**：调用 `equals` 时，**常量写前面更安全**：写 `"ok".equals(status)` 而不是 `status.equals("ok")`。前者即使 `status` 为 `null` 也只返回 `false`，后者会直接抛 `NullPointerException`。

### 2.3.2 查找与索引类方法

查找方法返回字符或索引，常用于"定位、截取、判断存在性"。**所有 `indexOf` 系列找不到时统一返回 `-1`**，这一点必须记住。

| 方法 | 说明 |
|------|------|
| `char charAt(int index)` | 返回指定索引处的字符，索引越界抛 `StringIndexOutOfBoundsException` |
| `int indexOf(int ch)` | 指定字符第一次出现的索引，没有返回 `-1` |
| `int indexOf(int ch, int fromIndex)` | 从 `fromIndex` 起往后找 |
| `int indexOf(String str)` | 指定子串第一次出现的索引 |
| `int indexOf(String str, int fromIndex)` | 从指定位置起往后找 |
| `int lastIndexOf(int ch)` | 指定字符最后一次出现的索引 |
| `int lastIndexOf(String str)` | 指定子串最后一次出现的索引 |

```java
String s = "Hello World";
System.out.println(s.charAt(0));          // H
System.out.println(s.indexOf('o'));       // 4  —— 第一次出现
System.out.println(s.lastIndexOf('o'));   // 7  —— 最后一次出现
System.out.println(s.indexOf("xyz"));     // -1 —— 没找到

// 循环找出所有 'l' 的位置，这是 indexOf(fromIndex) 的典型用法
int pos = s.indexOf('l');
while (pos != -1) {
    System.out.print(pos + " ");          // 2 3 9
    pos = s.indexOf('l', pos + 1);
}
```

> **注意**：`charAt` 的参数是**索引**而不是"第几个字符"，索引从 `0` 开始，合法范围是 `[0, length()-1]`。字符串长度为 `n` 时最后一个字符的索引是 `n-1`，写成 `charAt(length())` 必然越界。

## 2.4 截取、替换与转换类方法

### 2.4.1 截取与替换

`substring` 的区间是**左闭右开**的 `[beginIndex, endIndex)`——包含起点、不包含终点。这个约定和数组、`List.subList`、`StringBuilder.delete` 完全一致，记住一次到处适用。

| 方法 | 说明 |
|------|------|
| `String substring(int beginIndex)` | 从 `beginIndex` 截到末尾 |
| `String substring(int beginIndex, int endIndex)` | 截取 `[beginIndex, endIndex)` |
| `String replace(char oldChar, char newChar)` | 替换**所有**匹配的字符 |
| `String replace(CharSequence target, CharSequence replacement)` | 替换**所有**匹配的子串 |
| `String replaceAll(String regex, String replacement)` | 按**正则**替换全部 |
| `String replaceFirst(String regex, String replacement)` | 按**正则**替换第一个 |

```java
String s = "HelloWorld";
System.out.println(s.substring(5));        // World
System.out.println(s.substring(0, 5));     // Hello

String s2 = "Hello World";
System.out.println(s2.replace('o', 'x'));       // Hellx Wxrld
System.out.println(s2.replace("World", "Java")); // Hello Java
```

> **注意**：`substring` 返回的是**新字符串**，原串不变。JDK7 之前 `substring` 会共享底层 `char[]`（导致一个短子串拖住一个大数组无法回收），JDK7 起改成**复制**出新数组，这个历史坑已经不存在了，理解即可。

> **陷阱**：`replace` 和 `replaceAll` 名字很像，但语义完全不同——`replaceAll` 的第一个参数是**正则表达式**。`"a.b".replaceAll(".", "-")` 的结果不是 `"a-b"` 而是 `"----"`，因为 `.` 在正则里表示"任意字符"。要替换字面量就老老实实用 `replace`。

### 2.4.2 大小写与空白处理

| 方法 | 说明 |
|------|------|
| `String toLowerCase()` | 全部转小写 |
| `String toUpperCase()` | 全部转大写 |
| `String trim()` | 去除首尾空白，只认 `<= ' '` 的字符（半角空格、`\t`、`\n` 等） |
| `String strip()` | JDK11+，去除首尾空白，**支持 Unicode 空白**（如全角空格） |
| `String stripLeading()` | JDK11+，只去前导空白 |
| `String stripTrailing()` | JDK11+，只去尾部空白 |

```java
String s = "　Java　";   // 　 是全角空格
System.out.println("[" + s.trim() + "]" + s.trim().length());   // [　Java　]8，trim 去不掉全角空格
System.out.println("[" + s.strip() + "]" + s.strip().length());  // [Java]4，strip 可以
```

> **陷阱**：处理用户输入时，中文输入法很容易打出全角空格 `　`，`trim()` 对它无效。如果需要 JDK8 兼容又想去掉全角空白，可以用 `s.replaceAll("^[\\s\\u3000]+|[\\s\\u3000]+$", "")` 或直接升级到 JDK11 用 `strip()`。

### 2.4.3 大小写转换的区域陷阱

`toLowerCase()` / `toUpperCase()` 使用**默认区域（Locale）**。最经典的坑是土耳其语：在土耳其语环境下 `"I".toLowerCase()` 得到的是 `"ı"`（无点的 i）而不是 `"i"`，导致 `equals` 判断、配置项匹配全部失效。

```java
// 涉及协议名、HTTP 头、文件扩展名等"与语言无关"的转换，显式指定 Locale.ROOT
String s = "IMAGE.PNG";
System.out.println(s.toLowerCase(java.util.Locale.ROOT));   // image.png
```

> **行业实践**：凡是"大小写转换结果会被比较"的场景（文件后缀、协议头、枚举名），都传 `Locale.ROOT`；只有面向用户的展示文本才用默认 Locale。

### 2.4.4 与其他类型的转换

| 方法 | 说明 |
|------|------|
| `char[] toCharArray()` | 转成字符数组（返回副本，改它不影响原串） |
| `byte[] getBytes()` | 用**平台默认字符集**编码为字节数组 |
| `byte[] getBytes(Charset charset)` | 用指定字符集编码，**推荐** |
| `String repeat(int count)` | JDK11+，重复 `count` 次 |
| `String concat(String str)` | 拼接，等价于 `+` 但只能接字符串、不能接其他类型 |
| `static String join(CharSequence delimiter, CharSequence... elements)` | JDK8+，用分隔符连接多个元素 |
| `static String join(CharSequence delimiter, Iterable<? extends CharSequence> elements)` | JDK8+，连接集合 |

```java
System.out.println("ab".repeat(3));                     // ababab
System.out.println(String.join("-", "a", "b", "c"));    // a-b-c
System.out.println(String.join(",", java.util.List.of("x", "y"))); // x,y

byte[] data = "中文".getBytes(java.nio.charset.StandardCharsets.UTF_8);
System.out.println(data.length);                        // 6，UTF-8 下一个汉字占 3 字节
```

> **注意**：`concat` 只在拼接**字符串**时能用，`"a".concat(1)` 编译不过；而 `"a" + 1` 可以（会调用 `String.valueOf`）。所以日常拼接用 `+` 更顺手，`concat` 用得很少。

## 2.5 分割、连接与遍历

### 2.5.1 `split`：按正则分割

| 方法 | 说明 |
|------|------|
| `String[] split(String regex)` | 按正则分割（**丢弃**末尾的空字符串） |
| `String[] split(String regex, int limit)` | `limit > 0` 最多分 `limit` 份；`limit == 0` 丢弃尾部空串；`limit < 0` 保留所有空串 |

```java
String s = "a,b,,c,,";
System.out.println(java.util.Arrays.toString(s.split(",")));      // [a, b, , c] —— 尾部空串被丢
System.out.println(java.util.Arrays.toString(s.split(",", -1)));  // [a, b, , c, , ] —— 全部保留
System.out.println(java.util.Arrays.toString(s.split(",", 3)));   // [a, b, ,c,,] —— 最多 3 份
```

> **陷阱**：`split` 的参数是**正则**，不是一个普通字符。用 `"."` 分割时得到的是空数组——因为 `.` 在正则里是"任意字符"。正确写法是 `s.split("\\.")` 或 `s.split("[.]")`；同理，`|`、`+`、`*`、`?`、`(`、`)`、`[`、`]`、`{`、`}`、`^`、`$` 这些**正则元字符**做分隔符时都要转义。需要按纯字面量分割时可以用 `Pattern.quote(".")` 自动转义。

> **最佳实践**：分割字符串时，如果关心"末尾的空字段"（比如解析 CSV、按固定列拆分日志），一定要显式传 `limit = -1`。默认行为丢掉尾部空串，是很多解析 bug 的来源。

### 2.5.2 遍历字符串的四种方式

| 方式 | 写法 | 适用场景 |
|------|------|----------|
| 普通 `for` + `charAt` | `for (int i = 0; i < s.length(); i++) s.charAt(i)` | 需要**索引**时 |
| `toCharArray` + 增强 for | `for (char c : s.toCharArray())` | 只关心字符、不关心位置，可读性最好 |
| Stream | `s.chars().forEach(...)` | 函数式风格，`chars()` 返回的是 `int` 码点 |
| `codePoints` | `s.codePoints().forEach(...)` | 需要正确处理 emoji 等**增补字符**时 |

```java
String s = "Java";
for (int i = 0; i < s.length(); i++) {
    System.out.print(s.charAt(i) + " ");     // J a v a
}
for (char c : s.toCharArray()) {
    System.out.print(c + " ");               // J a v a
}
```

> **陷阱**：`String` 的 `length()` 返回的是 **UTF-16 码元（`char`）的个数**，不是"人类眼里的字符数"。一个 emoji（如 `"😀"`）在 UTF-16 里占 **2 个 `char`**，所以 `"😀".length()` 是 `2`，`charAt(0)` 拿到的是半个代理对。需要按真正的字符（码点）处理时，用 `codePointCount(0, s.length())` 统计个数、用 `codePoints()` 遍历。

## 2.6 `valueOf` 与字符串、基本类型的互转

### 2.6.1 基本类型 → 字符串

`String.valueOf` 是静态方法族，把各种类型安全地转成字符串；它和 `+ ""` 的效果一样，但更清晰、也不产生额外的拼接语义。`valueOf(Object)` 内部会调用对象的 `toString()`，`null` 会被安全地转成 `"null"` 而不是抛异常。

| 方法 | 说明 |
|------|------|
| `static String valueOf(boolean b)` | boolean → 字符串 |
| `static String valueOf(char c)` | char → 字符串 |
| `static String valueOf(char[] data)` | 字符数组 → 字符串 |
| `static String valueOf(int i)` | int → 字符串 |
| `static String valueOf(long l)` | long → 字符串 |
| `static String valueOf(float f)` | float → 字符串 |
| `static String valueOf(double d)` | double → 字符串 |
| `static String valueOf(Object obj)` | 任意对象 → 字符串（内部调用 `toString`，`null` 得 `"null"`） |

```java
int num = 100;
System.out.println(String.valueOf(num));       // 100
System.out.println(String.valueOf(3.14));      // 3.14
System.out.println(num + "");                  // 100，效果相同但不推荐
System.out.println(String.valueOf((Object) null)); // null —— 不会 NPE
```

### 2.6.2 字符串 → 基本类型

反过来靠各包装类的 `parseXxx` 静态方法。**格式不对就抛 `NumberFormatException`**（它是 `RuntimeException` 的子类，属于非受检异常，编译器不强制处理）。

| 方法 | 说明 |
|------|------|
| `Integer.parseInt(String s)` | 字符串 → int |
| `Long.parseLong(String s)` | 字符串 → long |
| `Double.parseDouble(String s)` | 字符串 → double |
| `Boolean.parseBoolean(String s)` | 字符串 → boolean（只有忽略大小写等于 `"true"` 才是 `true`） |
| `Integer.parseInt(String s, int radix)` | 按指定进制解析 |

```java
int i = Integer.parseInt("100");            // 100
double d = Double.parseDouble("3.14");      // 3.14
boolean b = Boolean.parseBoolean("TRUE");   // true
int hex = Integer.parseInt("FF", 16);       // 255，按 16 进制解析

try {
    Integer.parseInt("abc");                // 抛 NumberFormatException
} catch (NumberFormatException e) {
    System.out.println("不是合法数字：" + e.getMessage());  // For input string: "abc"
}
```

> **陷阱**：`Boolean.parseBoolean("anything")` 只有在忽略大小写等于 `"true"` 时返回 `true`，**其他一切输入都返回 `false`，不抛异常**。这意味着 `"yes"`、`"1"`、`"T"` 都会被静默当成 `false`，配置解析时容易出错，需要自己做映射。

> **注意**：`parseXxx` 返回的是**基本类型**，`valueOf` 返回的是**包装类对象**。`Integer.valueOf("100")` 等价于 `Integer.parseInt` 之后装箱，会走 `-128~127` 的缓存（见第 6 节）。选择依据是"要基本类型还是对象"。

## 2.7 字符串拼接的底层原理

拼接是 `String` 最容易写出性能问题的操作。理解它的底层，前提是记住：**因为 `String` 不可变，每次拼接都只能产生新对象**。但"产生多少对象"取决于拼接方式。

### 2.7.1 全是字面量：编译期常量折叠

如果参与拼接的**全部是字面量或 `final` 基本类型常量**，编译器会在**编译期**就把它们算成一个常量：

```java
String s = "a" + "b" + "c";
// 编译后完全等价于：
String s = "abc";
```

所以 `"a" + "b" == "ab"` 的结果是 `true`——它根本就是常量池里的 `"ab"`。**运行期零开销**。

> **注意**：`final` 修饰的基本类型/字符串变量参与拼接，也算编译期常量；而 `final` 修饰的**对象引用**不算。这个细节在面试里经常被追问。

### 2.7.2 有变量参与：编译期插入 `StringBuilder`

只要拼接中出现**变量**，就无法在编译期确定结果，编译器会生成拼接代码。不同 JDK 版本策略不同：

| 版本 | 底层实现 |
|------|----------|
| JDK8 及以前 | 编译成 `new StringBuilder().append(...).append(...).toString()` |
| JDK9 及以后 | 使用 `invokedynamic` + `StringConcatFactory`，按"近似数组"的方式直接拼，减少中间对象 |

```java
String b = "b";
String s = "a" + b + "c";
// JDK8 编译后近似：
String s = new StringBuilder().append("a").append(b).append("c").toString();
```

### 2.7.3 循环里拼接：为什么必须手动用 `StringBuilder`

这是拼接性能最重要的结论。看这段代码：

```java
// 反例：循环里用 + 拼接
String result = "";
for (int i = 0; i < 10000; i++) {
    result += i;      // 每次循环都 new 一个 StringBuilder，toString() 出一个新 String
}
```

`result += i` 等价于 `result = new StringBuilder().append(result).append(i).toString()`。也就是说，**每迭代一次就创建一个 `StringBuilder`、一个 `String`，并把前面已经攒下的所有内容再复制一遍**。循环 `n` 次的总复制量是 `O(n²)`，`n = 10000` 时会造成大量临时对象和频繁 GC，性能断崖式下跌。

```java
// 正例：循环外用同一个 StringBuilder
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10000; i++) {
    sb.append(i);     // 全程操作同一个底层数组，只在最后 toString() 一次
}
String result = sb.toString();
```

两者的复杂度对比：

| 写法 | 时间复杂度 | 临时对象 |
|------|-----------|----------|
| 循环内 `+=` | **O(n²)** | 约 2n 个 |
| 循环外 `StringBuilder` | **O(n)**（均摊） | 1 个 `StringBuilder` + 1 个 `String` |

> **最佳实践**：判断标准很简单——**拼接发生在循环里、或者拼接次数事先不确定时，一律用 `StringBuilder`**。单行里拼接三五个变量（如拼 SQL、拼日志）直接用 `+` 即可，编译器已经优化得足够好，强行用 `StringBuilder` 反而降低可读性。

> **注意**：JDK9 之后 `+` 的底层改成了 `invokedynamic`，单次拼接更快了，但这**不改变循环场景的结论**——循环里每次迭代仍然是一个独立的拼接表达式，仍然会反复创建对象。JDK9 的优化是"同样一次拼接少建几个对象"，不是"把 n 次拼接合并成一次"。

# 3. 字符串比较

## 3.1 `==` 与 `equals`

这是 Java 里最经典、也最必须一次记住的规则：

| 比较对象 | `==` 比较什么 | `equals` 比较什么 |
|----------|---------------|-------------------|
| 基本数据类型 | **数据值** | ——（基本类型没有方法） |
| 引用数据类型 | **地址值**（是不是同一个对象） | 默认也是地址；但 `String` 重写后比较**内容** |

```java
int a = 10, b = 10;
System.out.println(a == b);              // true，比较值

String s1 = new String("abc");
String s2 = new String("abc");
System.out.println(s1 == s2);            // false，两个不同的堆对象
System.out.println(s1.equals(s2));       // true，内容相同

String s3 = "abc";
String s4 = "abc";
System.out.println(s3 == s4);            // true，都指向常量池的同一个对象
System.out.println(s3 == s1);            // false，常量池对象 vs 堆对象
```

`equals` 的源码逻辑（`String` 重写版）分为四步：

```java
public boolean equals(Object anObject) {
    if (this == anObject) return true;               // 1. 同一个引用，直接 true
    if (!(anObject instanceof String)) return false; // 2. 不是 String，返回 false
    String another = (String) anObject;
    if (value.length != another.value.length) return false;   // 3. 长度不同直接 false
    // 4. 逐字符比较底层数组（JDK8 是 char[]，JDK9+ 是 byte[] 且先比 coder）
    return Arrays.equals(value, another.value);
}
```

> **最佳实践**：**比较字符串内容永远用 `equals`，不要用 `==`**。`==` 只在一种情况下"看起来能用"——两边都是同一个字面量常量池对象（`s3 == s4` 的那种），但只要中间经过了 `new`、`substring`、拼接、从 IO/网络读入，结果立刻变成 `false`。依赖 `==` 判断字符串内容，是初学者最常见的一类 bug。

> **行业实践**：把常量放左边可以顺带避免 NPE——`"yes".equals(input)` 即使在 `input` 为 `null` 时也安全返回 `false`（详见 2.3.1 的陷阱提示）。现代写法更推荐 `Objects.equals(a, b)`（见第 11 篇），它对两个参数都做空判断。

## 3.2 `equalsIgnoreCase`

```java
boolean equalsIgnoreCase(String anotherString)
```

不区分大小写地比较内容。典型用途是**协议、命令、枚举式的字符串比较**：

```java
String s = "Hello";
System.out.println(s.equalsIgnoreCase("HELLO"));   // true
System.out.println(s.equalsIgnoreCase("hello"));   // true
System.out.println(s.equalsIgnoreCase("World"));   // false
```

> **注意**：`equalsIgnoreCase` 内部同样受 Locale 影响，只是它做了两两字符的 `toUpperCase`/`toLowerCase` 比较，多数场景没问题。真正对 Locale 敏感的场景（土耳其语 `i`）还是要显式用 `Locale.ROOT` 先统一大小写。

## 3.3 `compareTo`

```java
int compareTo(String anotherString)          // 区分大小写
int compareToIgnoreCase(String str)          // 忽略大小写
```

按**字典顺序**（底层是 Unicode 码点值）比较，返回 `int`。它的返回值**不是简单的 `-1 / 0 / 1`**，而是：

| 返回值 | 含义 |
|--------|------|
| `0` | 两个字符串内容相等 |
| `< 0` | 当前字符串在参数字符串**前面** |
| `> 0` | 当前字符串在参数字符串**后面** |

源码逻辑是：从头逐字符比较，返回第一个不同字符的**码点差值**；如果比较到某个字符串结束都相同，就返回**长度差值**。

```java
System.out.println("apple".compareTo("banana"));   // -1  ('a' - 'b')
System.out.println("banana".compareTo("apple"));   // 1   ('b' - 'a')
System.out.println("ab".compareTo("abc"));         // -1  (长度差 2-3)
System.out.println("abc".compareTo("abc"));        // 0

// 实用场景：a 小 => 升序；b 小 => 降序
java.util.List<String> list = new java.util.ArrayList<>(java.util.List.of("b", "c", "a"));
list.sort((x, y) -> x.compareTo(y));
System.out.println(list);                          // [a, b, c]
```

> **注意**：`compareTo` 是 `Comparable` 接口的方法，`String` 实现了它，所以字符串可以直接用 `Collections.sort` / `Stream.sorted()` 排序。返回值是差值而非 `-1/0/1` 这一点，在写自定义 `Comparator` 时要注意——**不要写 `return a.compareTo(b) > 0 ? 1 : -1`**，直接在比较器里返回差值更简洁（但要防止整数溢出，字符串比较不会溢出，数字比较才要小心）。

# 4. StringBuilder

## 4.1 为什么需要 StringBuilder

`StringBuilder` 本质上是一个**可变字符序列容器**。它的存在理由只有一句话：**`String` 不可变，所以拼接会不断产生新对象；`StringBuilder` 用一个可变的底层数组，把"反复拼接"变成对同一个数组的追加**。

```java
// 用 String：产生 4 个对象（"Hello"、"Hello World"、"Hello World!"，以及中间被丢弃的）
String s = "Hello";
s += " World";
s += "!";

// 用 StringBuilder：只产生 1 个 StringBuilder + 最后 1 个 String
StringBuilder sb = new StringBuilder();
sb.append("Hello");
sb.append(" World");
sb.append("!");
String result = sb.toString();
```

**底层原理**：`StringBuilder` 内部维护一个 `byte[]`（JDK9+，与 `String` 一致）或 `char[]`（JDK8-）作为字符缓冲区，外加一个 `count` 记录当前已用长度。`append` 就是把新内容写进数组的 `count` 位置，然后 `count` 前移——**没有任何对象创建**，除非容量不够触发扩容。所以它的价值在于"**变**"：可变代表可以原地追加。

`StringBuilder` 的典型用法是**字符串拼接**和**字符串反转**（`sb.reverse()` 一行搞定）。

## 4.2 构造方法与初始容量

| 构造方法 | 说明 |
|----------|------|
| `public StringBuilder()` | 空容器，**初始容量 16 个字符** |
| `public StringBuilder(String str)` | 初始容量 = `str.length() + 16`，并用 `str` 填充 |
| `public StringBuilder(int capacity)` | 指定初始容量，**预知长度时用**，避免扩容 |
| `public StringBuilder(CharSequence seq)` | 用任意 `CharSequence` 初始化（`String`、`StringBuffer` 都是） |

```java
System.out.println(new StringBuilder().capacity());        // 16
System.out.println(new StringBuilder("abc").capacity());   // 19 = 3 + 16
System.out.println(new StringBuilder("abc").length());     // 3
System.out.println(new StringBuilder(1024).capacity());    // 1024
```

> **最佳实践**：**已知最终长度时，构造时就传容量**。比如要拼接 1000 个元素，每个平均 10 个字符，就 `new StringBuilder(10000)`——一次分配到位，全程零扩容。这是拼接性能优化的最后一块拼图：复用同一个 `StringBuilder` 解决了对象数量问题，预分配容量解决了数组复制问题。

> **注意**：`length()` 是**已用的字符数**，`capacity()` 是**数组的容量**，两者不是一回事。`new StringBuilder()` 的 `length()` 是 `0`、`capacity()` 是 `16`。写代码时关心的一般是 `length()`。

## 4.3 常用方法

| 方法 | 说明 |
|------|------|
| `StringBuilder append(任意类型)` | 追加数据，**返回 `this`**，可链式调用 |
| `StringBuilder reverse()` | 反转内容，返回 `this` |
| `int length()` | 返回字符数（不是容量） |
| `int capacity()` | 返回当前容量 |
| `char charAt(int index)` | 取指定索引的字符 |
| `StringBuilder delete(int start, int end)` | 删除 `[start, end)` 区间 |
| `StringBuilder deleteCharAt(int index)` | 删除指定位置的字符 |
| `StringBuilder insert(int offset, 任意类型)` | 在 `offset` 处插入数据 |
| `StringBuilder replace(int start, int end, String str)` | 用 `str` 替换 `[start, end)` |
| `void setCharAt(int index, char ch)` | 修改指定位置字符（**无返回值**） |
| `String toString()` | 转为 `String`（返回副本） |
| `void setLength(int newLength)` | 设置长度，变短就截断，变长补 `\0` |
| `void ensureCapacity(int minimumCapacity)` | 确保容量至少为指定值（手动预扩容） |
| `String substring(int start)` | 截取子串，返回 `String`（**返回新对象，不是视图**） |
| `String substring(int start, int end)` | 截取 `[start, end)`，返回 `String` |
| `int indexOf(String str)` | 查找子串位置，没有返回 `-1` |
| `int lastIndexOf(String str)` | 从后往前查找 |
| `void trimToSize()` | 把容量缩到实际长度，释放多余空间 |

```java
StringBuilder sb = new StringBuilder();
sb.append("Hello")
  .append(' ')
  .append("World")
  .append('!');                      // 链式调用，因为 append 返回 this
System.out.println(sb);              // Hello World!（println 会调 toString）

sb.reverse();
System.out.println(sb);              // !dlroW olleH

sb.reverse().delete(5, 6);           // 删除 [5,6) 即那个空格
System.out.println(sb);              // HelloWorld!

sb.insert(5, "-");
System.out.println(sb);              // Hello-World!
System.out.println(sb.charAt(0));    // H
```

> **注意**：`append` 之所以能链式调用，是因为它返回 `this`（自身）。而 `setCharAt`、`setLength` 返回 `void`，**不能**参与链式调用。写链式代码时如果编译不过，先看看中间某个方法的返回类型。

> **陷阱**：`StringBuilder` 的 `substring` 返回的是 `String`（**新对象、副本**），不是原缓冲区的视图。`String` 的 `substring` 是 JDK7 后也改成复制。真正的"视图"语义只在 `CharBuffer` 之类的 NIO 类里才有，别把它们的语义混起来。

## 4.4 扩容机制

扩容是 `StringBuilder` 唯一的性能暗坑，理解它才能写出零扩容的代码。逻辑在父类 `AbstractStringBuilder` 里。

**触发时机**：`append` / `insert` 时，如果**已用长度 + 新增长度 > 当前容量**，就触发扩容。

**扩容策略**：

```text
1. 新容量 = (旧容量 << 1) + 2，即 旧容量 * 2 + 2
2. 如果新容量仍不够用，就直接取"所需的最小容量"
3. 如果容量溢出/负数，取 Integer.MAX_VALUE - 8（MAX_ARRAY_SIZE）
4. 用 Arrays.copyOf() 把老数组的内容复制到新数组，底层数组引用替换
```

**完整示例流程**：

```text
1. new StringBuilder()                → 容量 16，长度 0
2. append("1234567890123456")         → 容量 16，长度 16（刚好填满，不扩容）
3. append("7")                        → 超出，触发扩容：16 * 2 + 2 = 34
                                        容量 34，长度 17
4. 之后每超过一次容量，容量都按 2n+2 增长：34 → 70 → 142 ...
```

```java
StringBuilder sb = new StringBuilder();
System.out.println(sb.capacity());     // 16
sb.append("1234567890123456");         // 16 个字符
System.out.println(sb.capacity());     // 16，刚好放满
sb.append("7");
System.out.println(sb.capacity());     // 34 = 16 * 2 + 2
System.out.println(sb.length());       // 17
```

> **注意**：`2n + 2` 的增长率让 `StringBuilder` 的**均摊时间复杂度是 O(1)**——虽然偶尔要复制整个数组，但每次扩容容量翻倍，平均到每次 `append` 上的代价是常数级。这也解释了为什么"预知长度就预分配"是纯收益：不预分配也不会慢到 O(n²)，只是会多做几次数组复制。

> **陷阱**：`StringBuilder` **不是线程安全的**，`append` 和扩容在多线程下会出现数据错乱甚至数组越界。需要多线程共享时要么用 `StringBuffer`，要么在外面加锁，要么每个线程各建一个（**最推荐**）。

## 4.5 `String`、`StringBuilder`、`StringBuffer` 对比

| 维度 | `String` | `StringBuilder` | `StringBuffer` |
|------|----------|-----------------|----------------|
| 可变性 | **不可变** | 可变 | 可变 |
| 线程安全 | 不可变即安全 | **非线程安全** | 线程安全（方法上 `synchronized`） |
| 性能 | 拼接产生大量对象，最低 | **最高** | 略低（有同步开销） |
| 出现版本 | JDK1.0 | JDK5 | JDK1.0 |
| 适用场景 | 字符串常量、少量拼接 | **单线程拼接（绝大多数场景）** | 多线程共享同一个缓冲区 |

```java
// 需要线程安全时的现代写法：不要用 StringBuffer，直接用局部 StringBuilder
public String buildLine(String[] fields) {
    StringBuilder sb = new StringBuilder();   // 方法内局部变量，天然线程安全
    for (String f : fields) {
        sb.append(f).append(',');
    }
    return sb.toString();
}
```

> **最佳实践**：**默认选 `StringBuilder`**。"多线程拼接"这个场景在真实代码里几乎不存在——`StringBuilder` 只要作为**方法内的局部变量**，每个线程各有一份，就不存在共享问题。`StringBuffer` 只在"极少数必须共享同一个可变缓冲区"的遗留代码里才会出现，新代码不应主动使用。需要拼接时也可以优先考虑 `StringJoiner`（见第 5 节）或 `String.join`。

# 5. StringJoiner

## 5.1 为什么需要 StringJoiner

用 `StringBuilder` 拼接"带分隔符"的列表时，有个很烦的细节：**最后一个元素后面不该有分隔符**，于是要写 `if (first)` 之类的判断，或者拼完再 `substring` 掉尾巴。`StringJoiner` 就是 JDK8 为此提供的封装，它自动处理分隔符、前缀、后缀。

| 构造方法 | 说明 |
|----------|------|
| `public StringJoiner(CharSequence delimiter)` | 只指定分隔符 |
| `public StringJoiner(CharSequence delimiter, CharSequence prefix, CharSequence suffix)` | 同时指定分隔符、前缀、后缀 |

```java
// 只带分隔符
StringJoiner sj = new StringJoiner(", ");
sj.add("a").add("b").add("c");
System.out.println(sj);                    // a, b, c

// 带前缀后缀
StringJoiner sj2 = new StringJoiner(", ", "[", "]");
sj2.add("a").add("b").add("c");
System.out.println(sj2);                   // [a, b, c]

// 一个元素都没有时，toString 返回 "[]"（前后缀都在，但没有中间内容）
System.out.println(new StringJoiner(", ", "[", "]"));   // []
```

## 5.2 常用方法

| 方法 | 说明 |
|------|------|
| `StringJoiner add(CharSequence newElement)` | 添加元素，返回 `this`，可链式 |
| `StringJoiner merge(StringJoiner other)` | 把另一个 `StringJoiner` 的内容并入当前对象 |
| `int length()` | 返回当前字符串表示的长度 |
| `StringJoiner setEmptyValue(CharSequence emptyValue)` | 设置**没有任何元素**时的返回值 |
| `String toString()` | 返回拼接结果 |

```java
StringJoiner left = new StringJoiner("-");
left.add("a").add("b");

StringJoiner right = new StringJoiner("-");
right.add("c").add("d");

left.merge(right);
System.out.println(left);                  // a-b-c-d

// setEmptyValue：一个元素都没添加时，toString 返回自定义内容而不是默认的 "[]"
StringJoiner empty = new StringJoiner(",", "[", "]");
empty.setEmptyValue("(空)");
System.out.println(empty);                 // (空)
empty.add("a");
System.out.println(empty);                 // [a]
```

> **注意**：`StringJoiner` 本质上是对 `StringBuilder` 的**语法糖封装**，底层同样用 `StringBuilder`，性能没有额外损失。`String.join(delimiter, elements)` 内部也是用 `StringJoiner` 实现的，所以"静态方法一次连接一个集合"用 `String.join` 更简洁，"逐个添加、还要前缀后缀"才用 `StringJoiner`。

> **行业实践**：拼接 SQL 的 `IN (...)` 列表、JSON 数组、HTTP 查询串这类"逗号分隔"的场景，`StringJoiner` 比手写 `StringBuilder` + 判断最后一位清爽得多，也更不容易漏掉"末尾多一个逗号"的 bug。

# 6. 包装类

## 6.1 为什么需要包装类

Java 是"面向对象"的语言，但有 8 个**基本数据类型**（`byte`、`short`、`int`、`long`、`float`、`double`、`char`、`boolean`）不是对象，不能调用方法，也不能放进只接受对象的容器——集合、泛型都只认引用类型。**包装类**就是为这 8 个基本类型各配一个"对象外壳"：

| 基本类型 | 包装类 | 是否有缓存 |
|----------|--------|-----------|
| `byte` | `Byte` | 有（`-128 ~ 127`，即全部取值） |
| `short` | `Short` | 有（`-128 ~ 127`，可通过参数扩展） |
| `int` | `Integer` | 有（`-128 ~ 127`，可通过参数扩展） |
| `long` | `Long` | 有（`-128 ~ 127`，可通过参数扩展） |
| `char` | `Character` | 有（`0 ~ 127`） |
| `boolean` | `Boolean` | 有（`TRUE` / `FALSE` 两个常量） |
| `float` | `Float` | **无** |
| `double` | `Double` | **无** |

除了 `Character` 和 `Boolean`，其余 6 个包装类都继承自 `Number`（所以都有 `intValue()`、`doubleValue()` 等拆箱方法），并且都实现了 `Comparable`、`Serializable`。**包装类都是 `final` 的，不能被继承**。

## 6.2 Integer 的创建方式

`Integer` 是最常用的包装类，它有"构造方法"和"静态工厂"两套创建方式，**两者的语义差别很大**：

| 创建方式 | 说明 |
|----------|------|
| `public Integer(int value)` | 根据 int 创建对象（JDK9 起标记废弃） |
| `public Integer(String s)` | 根据字符串创建对象，格式不对抛 `NumberFormatException`（JDK9 起废弃） |
| `static Integer valueOf(int i)` | 静态工厂，**会复用 `-128~127` 的缓存对象** |
| `static Integer valueOf(String s)` | 字符串 → `Integer` |
| `static Integer valueOf(String s, int radix)` | 按指定进制解析字符串 |

```java
Integer byNew = new Integer(100);        // 每次都在堆上新建一个对象（不推荐）
Integer byFactory = Integer.valueOf(100); // 优先复用缓存
System.out.println(byNew == byFactory);  // false：一个是新对象，一个是缓存里的对象

Integer byAssign = 100;                  // 自动装箱，等价于 Integer.valueOf(100)
System.out.println(byFactory == byAssign); // true：都指向缓存里的同一个对象
```

> **陷阱**：**`new Integer(...)` 是必须避免的写法**，它会绕开缓存、每次都在堆上创建新对象，既浪费内存又制造 `==` 陷阱。JDK9 起这两个构造方法已被标记为废弃（`@Deprecated(since = "9")`），JDK16 起标记为"未来删除"。**统一用 `Integer.valueOf()` 或直接赋值（自动装箱）**。

> **注意**：`Integer.valueOf(String)` 内部就是先 `parseInt` 再 `valueOf(int)`，所以它也会走缓存。`parseInt` 只返回 `int`，`valueOf` 返回对象——需要对象用 `valueOf`，需要基本类型用 `parseInt`。

## 6.3 自动装箱与拆箱

JDK5 引入了**自动装箱（autoboxing）**和**自动拆箱（unboxing）**，让基本类型和包装类能互相直接赋值、直接参与运算：

| 转换 | 术语 | 编译器实际做的事 |
|------|------|------------------|
| `int` → `Integer` | 自动装箱 | `Integer.valueOf(i)` |
| `Integer` → `int` | 自动拆箱 | `i.intValue()` |

```java
Integer i = 10;             // 自动装箱：编译成 Integer.valueOf(10)，走缓存
int j = i;                  // 自动拆箱：编译成 i.intValue()
Integer sum = i + j;        // 先拆箱相加，再把结果装箱
System.out.println(sum);    // 20
```

方便是真方便，但**自动拆箱埋着 NPE**：

```java
Integer i = null;
int j = i;                  // 编译后是 i.intValue() → NullPointerException
```

**最容易踩的三种 NPE 场景**：

```java
// 场景一：Map 取值后直接拆箱
java.util.Map<String, Integer> map = new java.util.HashMap<>();
int count = map.get("不存在的key");   // NPE！get 返回 null，拆箱时炸

// 正确做法：先判空，或用 getOrDefault
int count2 = map.getOrDefault("不存在的key", 0);   // 0

// 场景二：包装类与基本类型混合的三目运算
Integer a = null;
boolean flag = false;
// int r = flag ? a : 0;   // NPE：三目两侧一个是 Integer、一个是 int，
                           //      整体类型被提升为 int，于是 a 被拆箱

// 场景三：包装类参与算术和比较
Integer x = null;
// System.out.println(x > 0);   // NPE：> 运算符会触发拆箱
// System.out.println(x + 1);   // NPE
```

> **陷阱**：**自动拆箱 NPE 的特点是"编译期完全看不出来"**。代码里写的是 `int j = i;`、`x > 0`、`map.get(k) + 1`，看起来都是普通操作，但只要有一步拆箱且对象是 `null`，运行时就炸。凡是从 `Map`、实体类、方法返回值拿到包装类的场合，用之前都要有"它可能是 `null`"的警觉。

> **最佳实践**：**实体类字段、方法返回值用包装类（能表达 `null`），局部变量和运算用基本类型**。看到 `int` 和 `Integer` 出现在同一个表达式里（尤其是三目、`==`、`+`），就要停下来想一下会不会拆箱。JDK8 的 `OptionalInt` 等类型能进一步帮上忙，但核心还是"用之前判空"。

## 6.4 缓存机制与 `==` 陷阱

`Integer` 在类加载时会预先创建 `-128 ~ 127` 这 256 个对象的缓存数组（内部类 `IntegerCache`），`valueOf(int)` 落在这个范围内就直接返回缓存对象。**超出范围就 new 一个新对象。**

```java
Integer a = 127;
Integer b = 127;
System.out.println(a == b);        // true  —— 都在缓存里，是同一个对象
System.out.println(a.equals(b));   // true

Integer c = 128;
Integer d = 128;
System.out.println(c == d);        // false —— 超出缓存，两个不同的对象
System.out.println(c.equals(d));   // true  —— 值相等
```

这段代码是理解包装类 `==` 的关键。**同样的 `128`，两个变量用 `==` 比较却是 `false`**，因为 `Integer` 的 `==` 比较的是地址，而 `128` 不在缓存范围内，每次装箱都新建对象。

缓存的边界可以调整，但默认值就是 `-128 ~ 127`：

```java
// 通过 JVM 参数把 Integer 缓存上界调大（仅 Integer，且仅在启动时生效）
// -XX:AutoBoxCacheMax=1000
```

> **陷阱**：**包装类的 `==` 比较是纯粹的地址比较**，缓存的存在让它"有时候对、有时候错"，极其危险。`Integer`、`Long`、`Short`、`Byte` 都有这个缓存行为，`Character` 缓存 `0~127`，`Boolean` 缓存两个常量——**全都有"小值相等、大值不等"的诡异表现**。而 `Float`、`Double` **没有缓存**，所以 `Double d1 = 1.0; Double d2 = 1.0; d1 == d2` 恒为 `false`。

> **最佳实践**：**比较包装类的值，一律用 `equals`，或者先转成基本类型再比**。如果确实想用 `==` 省一点 `equals` 的开销，必须保证数值永远落在缓存范围内，并且要写注释说明——这种优化在业务代码里基本不值得。

| 比较方式 | 结果 |
|----------|------|
| `a == b` | 比较**地址**，缓存范围内为 `true`，范围外为 `false`，**不要用** |
| `a.equals(b)` | 比较**值**，推荐 |
| `a.intValue() == b.intValue()` | 先拆箱再比较值，也正确（但要注意 NPE） |

## 6.5 Integer 常用方法

| 方法 | 说明 |
|------|------|
| `static int parseInt(String s)` | 字符串 → int（最常用的解析方法） |
| `static int parseInt(String s, int radix)` | 按指定进制解析 |
| `static String toBinaryString(int i)` | 转二进制字符串 |
| `static String toOctalString(int i)` | 转八进制字符串 |
| `static String toHexString(int i)` | 转十六进制字符串 |
| `static int max(int a, int b)` / `min(int a, int b)` | 求最大/最小值 |
| `static int compare(int x, int y)` | 比较大小，返回负数/0/正数 |
| `static int sum(int a, int b)` | 求和（JDK8+） |
| `Integer.MIN_VALUE` / `Integer.MAX_VALUE` | `-2^31` 与 `2^31 - 1` 常量 |
| `int intValue()` / `long longValue()` / `double doubleValue()` | 拆箱为各种基本类型 |

```java
System.out.println(Integer.parseInt("100"));        // 100
System.out.println(Integer.toBinaryString(10));     // 1010
System.out.println(Integer.toHexString(255));       // ff
System.out.println(Integer.parseInt("FF", 16));     // 255
System.out.println(Integer.toBinaryString(-1));     // 11111111111111111111111111111111（补码）
System.out.println(Integer.MAX_VALUE);              // 2147483647
System.out.println(Integer.MIN_VALUE);              // -2147483648
```

> **注意**：`toBinaryString(-1)` 拿到的是 32 个 `1`，因为整数在计算机里用**补码**表示。负数转进制字符串时不要期待"一个减号加二进制"，`Integer.toBinaryString` 输出的是该数的补码形式。

> **注意**：`Integer.MAX_VALUE + 1` 会溢出成 `Integer.MIN_VALUE`，这是所有整数类型共有的陷阱。涉及**金额、大数运算**时不要用 `int`/`long`/`double` 硬扛——金额用 `BigDecimal`（`double` 无法精确表示 `0.1`，见第 11 篇），超大整数用 `BigInteger`（也在第 11 篇）。生成唯一标识、事务号时常用 `java.util.UUID.randomUUID().toString()`，同样属于第 11 篇的内容。

## 6.6 其他包装类

其余 7 个包装类的用法与 `Integer` 高度一致，都提供 `parseXxx`、`valueOf`、`xxxValue`：

| 包装类 | 常用静态方法 | 提醒 |
|--------|--------------|------|
| `Long` | `parseLong` / `valueOf` / `toBinaryString` | 缓存 `-128~127`；`long` 字面量要加 `L` |
| `Double` | `parseDouble` / `valueOf` / `isNaN` | **无缓存**，`==` 恒为 false；有 `NaN`、`Infinity` 特例 |
| `Float` | `parseFloat` / `valueOf` | **无缓存**；精度只有 7 位左右 |
| `Character` | `isDigit` / `isLetter` / `isWhitespace` / `toUpperCase` | 缓存 `0~127`；判断字符类型很好用 |
| `Boolean` | `parseBoolean` / `valueOf` | 只有 `"true"`（忽略大小写）为 `true`，其余均为 `false` |
| `Byte` / `Short` | `parseByte` / `parseShort` | 缓存全部取值（byte）或 `-128~127`（short） |

```java
System.out.println(Character.isDigit('7'));        // true
System.out.println(Character.isLetter('中'));      // true
System.out.println(Character.isWhitespace(' '));   // true
System.out.println(Character.toUpperCase('a'));    // A

System.out.println(Boolean.parseBoolean("yes"));   // false —— 只有 "true" 才是 true

Double d1 = 1.0, d2 = 1.0;
System.out.println(d1 == d2);        // false —— Double 无缓存
System.out.println(d1.equals(d2));   // true
```

> **注意**：`Double` 有两个特殊值 `NaN`（非数字）和 `Infinity`（无穷大）。`NaN == NaN` 永远是 `false`，判断要用 `Double.isNaN(x)`；`0.0 / 0` 得到 `NaN`，`1.0 / 0` 得到 `Infinity`。做除法前记得判断分母。

# 7. 小结

| 主题 | 一句话要点 |
|------|-----------|
| API | javadoc 是权威来源，重点看方法签名与 `Throws`；`java.lang` 包自动导入 |
| `String` 不可变 | `final` 类 + `private final` 字段 + 无修改方法 + 不泄露内部数组；好处是线程安全、可缓存 `hashCode`、可复用、安全 |
| 常量池与创建 | 字面量进常量池可复用，`new` 每次在堆上新建；`new String("abc")` 可能创建 2 个对象 |
| 拼接底层 | 全字面量在编译期折叠；含变量时 JDK8 用 `StringBuilder`、JDK9 用 `invokedynamic`；**循环内拼接必须手动用 `StringBuilder`** |
| 字符串比较 | `==` 比地址，`equals` 比内容；**比较内容永远用 `equals`**，常量放左边防 NPE |
| `StringBuilder` | 可变容器，默认容量 16，扩容策略 `2n + 2`；非线程安全，**已知长度就预分配容量** |
| 三兄弟对比 | 单线程拼接用 `StringBuilder`，多线程共享才考虑 `StringBuffer`，动辄拼接可读性优先选 `StringJoiner` / `String.join` |
| 包装类 | 8 种基本类型各有包装类，`Number` 子类 + `final`；创建统一用 `valueOf` / 自动装箱 |
| 两大陷阱 | **自动拆箱 NPE**（`Integer i = null; int j = i;`）与 **`Integer` 缓存 `==` 陷阱**（`-128~127` 内相等、范围外不等）；值比较一律用 `equals` |

下一篇（第 9 篇）进入集合框架，看 `ArrayList`、`HashMap` 如何组织数据；届时你会再次遇到本篇的两个主角——`String` 作为 `HashMap` 键依赖它的不可变性与 `hashCode` 缓存，包装类则通过自动装箱让集合能够存放基本类型的数据。
