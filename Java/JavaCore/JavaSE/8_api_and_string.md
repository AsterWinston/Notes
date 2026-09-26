# 1.api

```
API（Application Programming Interface，应用程序编程接口），就是 Java 已经写好的各种类和接口的统称。
使用方式：通过查阅 JDK 官方文档（javadoc），了解类和方法的使用说明。
```

# 2.字符串:字符串的内容是不会发生改变的，它的对象在创建后不能被更改

```
字符串（String）是 Java 中最常用的引用数据类型之一。
核心特性：String 对象是不可变的（immutable）。任何对字符串的修改操作都会产生新的 String 对象，原对象不受影响。
```

## 2.1String

### 2.1.1java.lang.String（java.lang包不需要手动导入）

```
String 类位于 java.lang 包下，是 Java 程序中最基础也是使用频率最高的类。
java.lang 包下的类默认导入，无需显式 import。
```

### 2.1.2创建String的两种方式

```
2.1.2.1直接赋值：String <str_name> = "content"
  这种方法会将字符串值放入字符串常量池（String Pool）中。
  当再次用相同字面量创建字符串时，会复用常量池中已有的对象。

2.1.2.2new 方式创建
  2.1.2.2.1 public String() // 创建空白字符串，不含任何内容
  2.1.2.2.2 public String(String original) // 根据传入的字符串，创建字符串对象
  2.1.2.2.3 public String(char[] chs) // 根据字符数组，创建字符串对象
  2.1.2.2.4 public String(byte[] chs) // 根据字节数组，创建字符串对象

2.1.2.3 两种方式的区别：
  直接赋值方式：字符串对象存储在字符串常量池中（JDK7之前方法区，JDK7之后堆内存）
  new 方式：每次都会在堆内存中创建新的对象
```

### 2.1.3成员函数

#### 2.1.3.1 valueOf 系列（静态方法）

```
static String valueOf(boolean b)     // 将 boolean 转为字符串
static String valueOf(char c)        // 将 char 转为字符串
static String valueOf(char[] data)   // 将字符数组转为字符串
static String valueOf(int i)         // 将 int 转为字符串
static String valueOf(long l)        // 将 long 转为字符串
static String valueOf(float f)       // 将 float 转为字符串
static String valueOf(double d)      // 将 double 转为字符串
static String valueOf(Object obj)    // 将 Object 转为字符串（调用其 toString()）
```

#### 2.1.3.2 字符串判断方法

```
boolean isEmpty()                    // 判断字符串是否为空字符串（length() == 0）
boolean isBlank()                    // JDK11+，判断字符串是否为空白（仅含空格/tab/换行等）
boolean equals(Object obj)           // 比较字符串内容是否相等（区分大小写）
boolean equalsIgnoreCase(String s)   // 比较字符串内容是否相等（忽略大小写）
boolean contains(CharSequence s)     // 判断是否包含指定字符序列
boolean startsWith(String prefix)    // 判断是否以指定前缀开头
boolean endsWith(String suffix)      // 判断是否以指定后缀结尾
boolean matches(String regex)        // 判断是否匹配正则表达式
```

#### 2.1.3.3 字符串查找方法

```
char charAt(int index)                // 返回指定索引处的字符
int indexOf(int ch)                   // 返回指定字符第一次出现的索引，未找到返回 -1
int indexOf(String str)               // 返回指定字符串第一次出现的索引
int indexOf(int ch, int fromIndex)    // 从指定位置开始查找指定字符
int indexOf(String str, int fromIndex)// 从指定位置开始查找指定字符串
int lastIndexOf(int ch)               // 返回指定字符最后一次出现的索引
int lastIndexOf(String str)           // 返回指定字符串最后一次出现的索引
```

#### 2.1.3.4 字符串截取与替换

```
String substring(int beginIndex)              // 从指定索引截取到末尾
String substring(int beginIndex, int endIndex) // 截取 [beginIndex, endIndex) 范围的子串
String replace(char oldChar, char newChar)     // 替换所有匹配的字符
String replace(CharSequence target, CharSequence replacement) // 替换所有匹配的字符串
String replaceAll(String regex, String replacement) // 使用正则替换
String replaceFirst(String regex, String replacement) // 使用正则替换第一个匹配
```

#### 2.1.3.5 字符串转换方法

```
String toLowerCase()                 // 转换为小写
String toUpperCase()                 // 转换为大写
String trim()                        // 去除首尾空格（半角空格，不处理全角空格和\t）
String strip()                       // JDK11+，去除首尾的空白字符（包括 Unicode 空白）
String stripLeading()                // JDK11+，去除前导空白
String stripTrailing()               // JDK11+，去除尾部空白
char[] toCharArray()                 // 将字符串转换为字符数组
byte[] getBytes()                    // 使用默认字符集将字符串编码为字节数组
byte[] getBytes(String charsetName)  // 使用指定字符集编码
byte[] getBytes(Charset charset)     // 使用指定字符集编码
```

#### 2.1.3.6 字符串分割与连接

```
String[] split(String regex)         // 根据正则表达式分割字符串
String[] split(String regex, int limit) // 根据正则分割，limit 控制分割份数
String concat(String str)            // 拼接字符串（等同于 +，但功能更受限）
static String join(CharSequence delimiter, CharSequence... elements) // JDK8+，用分隔符连接
static String join(CharSequence delimiter, Iterable<? extends CharSequence>) // JDK8+
```

#### 2.1.3.7 字符串长度与重复

```
int length()                          // 返回字符串长度
String repeat(int count)              // JDK11+，将字符串重复 count 次
int codePointCount(int begin, int end) // 返回指定范围内的 Unicode 码点数量
```

#### 2.1.3.8 正则相关（详见 11_api.md 第6-10节）

```
boolean matches(String regex)         // 判断是否匹配正则
String replaceAll(String regex, String replacement) // 正则替换全部
String replaceFirst(String regex, String replacement) // 正则替换第一个
String[] split(String regex)          // 正则分割
```

## 2.2StringBuilder:可以看作一个容器，创建之后里面的内容是可变的，为了解决String创建对象过程产生大量无用对象而造成资源浪费而存在，提高了字符串的操作效率，常用于字符串反转和拼接

### 2.2.1构造方法

```
2.2.1.1public StringBuilder()                  // 创建空容器，初始容量 16 个字符
2.2.1.2public StringBuilder(String str)        // 根据字符串创建，初始容量 = str.length() + 16
2.2.1.3public StringBuilder(int capacity)      // 指定初始容量
2.2.1.4public StringBuilder(CharSequence seq)  // 根据 CharSequence 创建
```

### 2.2.2成员方法

```
2.2.2.1public StringBuilder append(<ANY_TYPE> data) // 添加数据，返回对象本身（链式调用）
2.2.2.2public StringBuilder reverse()               // 反转容器中的内容
2.2.2.3public int length()                           // 返回字符数（不是容量）
2.2.2.4public int capacity()                         // 返回当前容量
2.2.2.5public char charAt(int index)                 // 获取指定索引的字符
2.2.2.6public StringBuilder delete(int start, int end)       // 删除 [start, end) 的字符
2.2.2.7public StringBuilder deleteCharAt(int index)           // 删除指定位置的字符
2.2.2.8public StringBuilder insert(int offset, <ANY_TYPE>)    // 在指定位置插入数据
2.2.2.9public StringBuilder replace(int start, int end, String str) // 替换指定范围字符串
2.2.2.10public void setCharAt(int index, char ch)            // 设置指定位置字符
2.2.2.11public String toString()                             // 转换为 String
2.2.2.12public void setLength(int newLength)                 // 设置字符序列长度
2.2.2.13public void ensureCapacity(int minimumCapacity)     // 确保容量至少为指定值
2.2.2.14public String substring(int start)                   // 截取子串（返回 String）
2.2.2.15public String substring(int start, int end)          // 截取子串
2.2.2.16public int indexOf(String str)                       // 查找子串位置
2.2.2.17public int lastIndexOf(String str)                   // 从后往前查找
2.2.2.18public void trimToSize()                             // 将容量缩减到实际长度
```

### 2.2.3StringBuilder扩容机制（源码分析）

```
初始容量：无参构造默认 16，带参构造 = 参数长度 + 16。

扩容时机：当 append 操作使得当前字符数超过容量时触发扩容。

扩容策略：
  JDK8+：
    新容量 = (旧容量 << 1) + 2，即 旧容量 * 2 + 2。
    如果新容量仍不足，则新容量 = 所需的最小容量。
    使用 Arrays.copyOf() 复制底层数组到新数组。
    底层基于 byte[]（JDK9+）或 char[]（JDK8-）实现。

示例流程：
  1. new StringBuilder()      → 容量 16, 长度 0
  2. append("1234567890123456") → 容量 16, 长度 16 (刚好满)
  3. append("7")              → 触发扩容: 16*2+2=34, 容量 34, 长度 17
```

## 2.3StringJoiner:常用于字符串拼接，相对于StringBuilder更方便

### 2.3.1构造方法

```
2.3.1.1public StringJoiner(CharSequence delimiter)
  仅指定分隔符，例如：new StringJoiner(", ") → 拼接结果如 "a, b, c"

2.3.1.2public StringJoiner(CharSequence delimiter, CharSequence prefix, CharSequence suffix)
  指定分隔符、前缀、后缀，例如：new StringJoiner(", ", "[", "]") → 拼接结果如 "[a, b, c]"
```

### 2.3.2成员方法

```
2.3.2.1public StringJoiner add(CharSequence newElement)  // 添加元素，返回 this
2.3.2.2public StringJoiner merge(StringJoiner other)      // 合并另一个 StringJoiner
2.3.2.3public int length()                                // 返回当前字符串表示的长度
2.3.2.4public StringJoiner setEmptyValue(CharSequence emptyValue) // 设置空值时的默认返回
2.3.2.5public String toString()                           // 返回拼接结果
```

## 2.4字符串原理

### 2.4.1字符串存储的内存原理

```
2.4.1.1直接赋值会复用字符串常量池中的对象
  JDK7之前：字符串常量池位于方法区（永久代）。
  JDK7开始：字符串常量池移到了堆内存中。
  当使用 String s = "hello"; 时，JVM 会先检查字符串常量池中是否存在 "hello"：
    - 如果存在，直接返回引用。
    - 如果不存在，在常量池中创建 "hello" 对象并返回引用。

2.4.1.2new出来的String对象不会复用常量池，而是在堆区开辟空间
  String s = new String("hello"); 会创建两个对象（如果常量池中无 "hello"）：
    - 首先在常量池中创建 "hello"（如果不存在）
    - 然后在堆内存中创建一个新的 String 对象，内容指向常量池中的 "hello"

2.4.1.3intern() 方法
  调用 intern() 方法会将字符串对象放入常量池（如果已存在则返回已存在的引用）：
    String s1 = new String("hello");
    String s2 = s1.intern();  // s2 指向常量池中的 "hello"
    String s3 = "hello";      // s3 也指向常量池中的 "hello"
    System.out.println(s2 == s3);  // true
```

### 2.4.2==比较的原理

```
2.4.2.1基本数据类型：比较的是数据值
  int a = 10; int b = 10; System.out.println(a == b);  // true

2.4.2.2引用数据类型：比较的是地址值
  String s1 = new String("abc");
  String s2 = new String("abc");
  System.out.println(s1 == s2);        // false，两个不同对象
  System.out.println(s1.equals(s2));   // true，内容相同

  String s3 = "abc";
  String s4 = "abc";
  System.out.println(s3 == s4);        // true，都指向常量池同一对象
```

### 2.4.3字符串拼接的底层

```
2.4.3.1拼接的时候没有变量，都是字面量字符串
  触发字符串的优化机制，在编译时就已经是最后的结果，复用串池中的字符串。
  例如：
    String s = "a" + "b" + "c";
    编译后等同于：String s = "abc";  // 直接查找/创建常量池中的 "abc"

2.4.3.2拼接的时候有变量
  JDK8以前：
    底层使用 StringBuilder 进行拼接：
      String s = "a" + b + "c";
      编译后近似于：
        String s = new StringBuilder().append("a").append(b).append("c").toString();

  JDK8以后（实际为JDK9优化）：
    使用 invokedynamic + StringConcatFactory 进行优化，
    底层采用类似数组的方式直接拼接，减少中间对象创建。
```

### 2.4.4StringBuilder提高效率原理图:所有要拼接的内容都会往StringBuilder中存放，不会创建很多无用的内存空间

```
为什么 StringBuilder 比 String 拼接效率高？

String 拼接：
  String result = "Hello";
  result += " World";   // 创建了新的 String 对象，原 "Hello" 变为垃圾
  result += "!";        // 又创建了新的 String 对象

  每次拼接都创建新对象 → 大量临时对象 → 频繁 GC → 性能低下

StringBuilder 拼接：
  StringBuilder sb = new StringBuilder();
  sb.append("Hello");
  sb.append(" World");
  sb.append("!");
  String result = sb.toString();  // 只在最后生成一个 String

  全程在自身数组上操作 → 零临时对象 → 高性能
```

## 2.5StringBuffer（与 StringBuilder 对比）

```
StringBuffer 是 StringBuilder 的线程安全版本，所有 public 方法都用 synchronized 修饰。

            String      StringBuilder    StringBuffer
  可变性    不可变       可变             可变
  线程安全  不可变即安全  非线程安全       线程安全（synchronized）
  性能      —           高              较低（同步开销）
  适用场景  少量字符串   单线程拼接       多线程拼接

实际开发中，大多数场景使用 StringBuilder 即可（方法内的局部变量不存在多线程问题）。
```

# 3.字符串比较（String对象的方法，如果使用的是==比较，那么比较地址，只适用于比较两个直接用""赋值的字符串）

## 3.1equals

```
boolean equals(Object anObject)
  比较两个字符串的内容是否相等（区分大小写）。
  参数类型为 Object，可以传入任意类型对象。
  如果传入的不是 String 类型，返回 false。

示例：
  String s1 = "Hello";
  String s2 = "Hello";
  String s3 = new String("Hello");
  System.out.println(s1.equals(s2));  // true
  System.out.println(s1.equals(s3));  // true

源码逻辑摘要：
  1. 先判断是否同一个引用（this == anObject），是则直接返回 true
  2. 判断参数是否为 String 类型，不是则返回 false
  3. 比较长度，长度不同返回 false
  4. 逐字符比较数组内容，全部相同则返回 true
```

## 3.2equalsIgnoreCase

```
boolean equalsIgnoreCase(String anotherString)
  比较两个字符串的内容是否相等，忽略大小写。

示例：
  String s1 = "Hello";
  System.out.println(s1.equalsIgnoreCase("HELLO"));   // true
  System.out.println(s1.equalsIgnoreCase("hello"));    // true
  System.out.println(s1.equalsIgnoreCase("HeLLo"));    // true
```

## 3.3compareTo

```
int compareTo(String anotherString)
  按字典顺序比较两个字符串，基于 Unicode 码点值。
  返回值：
    0    → 两个字符串相等
    < 0  → 当前字符串小于参数字符串
    > 0  → 当前字符串大于参数字符串

int compareToIgnoreCase(String str)
  忽略大小写的字典顺序比较。
```

# 4.字符串操作

## 4.1public char charAt(int index)

```
返回字符串中指定索引位置的字符，索引从 0 开始。
如果索引超出范围 [0, length()-1]，抛出 StringIndexOutOfBoundsException。

示例：
  String s = "Hello";
  System.out.println(s.charAt(0));  // H
  System.out.println(s.charAt(4));  // o
```

## 4.2public int length()

```
返回字符串中字符的个数（长度）。
与数组的 length 属性不同，这是 String 类的方法。

示例：
  String s = "Hello World";
  System.out.println(s.length());  // 11（包含空格）
```

## 4.3String substring(int beginIndex, int endIndex)//左开右闭，将String中下标从beginIndex到endIndex的字符串截取出来

```
String substring(int beginIndex)
  截取从 beginIndex 开始到结尾的子串。

String substring(int beginIndex, int endIndex)
  截取从 beginIndex 开始到 endIndex 结束（不含 endIndex 位置的字符）的子串。

示例：
  String s = "HelloWorld";
  System.out.println(s.substring(5));     // "World"
  System.out.println(s.substring(0, 5));  // "Hello"
```

## 4.4String replace(String target,String replacement)//替换String中所有的target为replacement

```
String replace(char oldChar, char newChar)
  将所有 oldChar 替换为 newChar。

String replace(CharSequence target, CharSequence replacement)
  将所有匹配的 target 子串替换为 replacement。

示例：
  String s = "Hello World";
  System.out.println(s.replace('o', 'x'));        // "Hellx Wxrld"
  System.out.println(s.replace("World", "Java"));  // "Hello Java"
```

## 4.5其他常用操作

```
String toLowerCase()           // 全部转为小写
String toUpperCase()           // 全部转为大写
String trim()                  // 去除首尾空格
String strip()                 // JDK11+，去除首尾空白（更强）
String[] split(String regex)   // 按正则分割
boolean contains(CharSequence) // 是否包含子串
int indexOf(String str)        // 查找子串首次出现位置
String repeat(int count)       // JDK11+，重复拼接
String concat(String str)      // 拼接
static String join(String delimiter, String... elements) // JDK8+，分隔符连接
```

# 5.字符串与基本类型的转换

```
5.1 基本类型 → 字符串
  String.valueOf(基本类型值)
  或者直接使用 + ""（利用字符串连接符自动转换）
  
  示例：
    int num = 100;
    String s1 = String.valueOf(num);  // "100"
    String s2 = num + "";              // "100"

5.2 字符串 → 基本类型
  使用对应的包装类 parse 方法：
    int i = Integer.parseInt("100");
    double d = Double.parseDouble("3.14");
    boolean b = Boolean.parseBoolean("true");
    long l = Long.parseLong("100");

  如果字符串格式不正确，抛 NumberFormatException。
```
