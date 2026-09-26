# 1.Math

```
Math
```

# 2.System

## 2.1public static void exit(int status)//终止当前运行的java虚拟机

```
public static void exit(int status)//终止当前运行的java虚拟机
```

## 2.2public static long currentTimeMillis()//返回当前系统的时间毫秒值形式，时间原点为1970.1.1.0.0.0

```
public static long currentTimeMillis()//返回当前系统的时间毫秒值形式，时间原点为1970.1.1.0.0.0
```

## 2.3public static void arrayCopy(\<source\>, \<start\_index\>, \<destination\>, \<start\_index\>, \<copy\_count\>)//数组拷贝，基本数据类型保持一致，索引超出范围报错，子类数组可以拷贝给父类数组

```
public static void arrayCopy(<source>, <start_index>, <destination>, <start_index>, <copy_count>)//数组拷贝，基本数据类型保持一致，索引超出范围报错，子类数组可以拷贝给父类数组
```

# 3.Runtime

## 3.1public static Runtime getRuntime()//当前系统的运行环境对象

```
public static Runtime getRuntime()//当前系统的运行环境对象
```

## 3.2public void exit(int status)

```
public void exit(int status)
```

## 3.3public int availableProcessors()//获取CPU的线程数

```
public int availableProcessors()//获取CPU的线程数
```

## 3.4public long maxMemory()//JVM能从系统中获取总内存大小（单位为Byte）

```
public long maxMemory()//JVM能从系统中获取总内存大小（单位为Byte）
```

## 3.5public long totalMomery()//JVM已经从系统中获取总内存大小

```
public long totalMomery()//JVM已经从系统中获取总内存大小
```

## 3.6public long freeMemory()//JVM剩余内存大小

```
public long freeMemory()//JVM剩余内存大小
```

## 3.7public Process exec(String command)//运行cmd命令

```
public Process exec(String command)//运行cmd命令
```

# 4.Object和Objects

## 4.1Object是Java中的顶级父类，所有的类都直接或间接地继承于Object类

```
Object是Java中的顶级父类，所有的类都直接或间接地继承于Object类
```

## 4.2Objec类中的方法可以被所有子类访问

```
Objec类中的方法可以被所有子类访问
```

## 4.3Object方法

```
4.3.1public String toString()//返回对象的字符串表现形式（<package_name>@<address>）
4.3.2public boolean equals(Object obj)//比较两个对象是否相等
4.3.3protected Object clone(int a)//对象克隆（小知识:如果一个接口里面没有抽象方法，那么表示当前接口是一个标记性接口）
```

## 4.4Objects:一个对象工具类

```
4.4.1public static boolean equals(Object a, Object b)//先做非空判断，比较两个对象，如果地址值相同，返回true，否则判断其中一个对象是否为空，如果空返回false，否则调用对象的equals方法
4.4.2public static boolean isNull(Object obj)
4.4.3public static boolean nonNull(Object obj)
```

# 5.BigInteger和BigDecimal

## 5.1BigInteger构造方法

```
5.1.1pulbic BigInteger(int num, Random rnd)//获取随机大整数，范围是0~2^num-1
5.1.2public BigInteger(String val)//获取指定的大整数
5.1.3public BigInteger(String val, int radix)//获取指定进制的大整数
```

## 5.2BigInteger静态方法

```
5.2.1public static BigInteger valueOf(long val)
```

## 5.3注意

```
5.3.1对象一旦创建BigInteger内部记录的值不能发生改变
5.3.2只要进行计算都会产生一个新的BigInteger对象
```

## 5.4BigInteger常用方法

```
5.4.1public BigInteger add(BigInteger val)
5.4.2public BigInteger subtract(BigInteger val)
5.4.3public BigInteger multiply(BigInteger val)
5.4.4public BigInteger divide(BigInteger val)
5.4.5public BigInteger[] divideAndRemainder(BigInteger val)
5.4.6public boolean equals(Object x)
5.4.7public BigInteger pow(int exponent)
5.4.8public BigInteger max/min(BigInteger val)
5.4.9pulbic int intValue(BigInteger val)//转换为int
```

## 5.5BigDecimal构造

```
5.5.1public BigDecimal(double val)
5.5.2public BigDecimal(String val)
```

## 5.6BigDecimal静态方法

```
5.6.1public static BigDecimal valueOf(double val)
5.6.2public static BigDecimal valueOf(long val)
```

## 5.7BigDecimal常用方法

```
5.7.1public BigDecimal add(BigDecimal val)
5.7.1public BigDecimal substract(BigDecimal val)
5.7.3public BgiDecimal multiply(BigDecimal val)
5.7.4public BigDecimal divide(BigDecimal val)//除不尽不可以使用该方法
5.7.5public BigDecimal divide(BigDecimal val, 精确位数(int), 舍入模式(RoundingMode))
```

# 6.正则表达式（stringobj.matchs(String regex)）

## 6.1字符类（只匹配一个字符）

```
6.1.1[abc]//只能是a,b或c
6.1.2[^abc]//除了a,b,c之外的任何字符
6.1.3[a-zA-Z]//a到zA-Z
6.1.4[a-d[m-p]]//a到d，或m到p
6.1.5[a-z&&[def]]//a-z和def的交集即def
6.1.6[a-z&&[^bc]]//a-z和非bc的交集，即[ad-z]
6.1.7[a-z&&[^m-p]]//a到z和除了m到p的交集，即[a-lq-z]
```

## 6.2预定义字符

```
6.2.1.//任意字符，包括中文
6.2.2\d//一个数字即[0-9]
6.2.3\D//非数字即[^0-9]
6.2.4\s//一个空白字符即[\t\n\x0B\f\r]
6.2.5\S//非空白字符即[^\s]
6.2.6\w//[a-zA-Z_0-9]
6.2.7\W//[^\w]
```

## 6.3数量词

```
6.3.1X?//一次或零次
6.3.2X*//零次或多次
6.3.3X+//一次或多次
6.3.4X{n}//正好n次
6.3.5X{n,}//至少n次
6.3.6X{n, m}至少n次但是不超过m次
```

## 6.4特殊符号

```
6.4.1[]//里面的内容出现一次
6.4.2()//分组
6.4.3^//取反
6.4.4&&//交集
6.4.5|//
6.4.6(?i)//忽略后面字符的大小写（可以用括号绑定特定字符段指定忽略部分字符串的大小写）
6.4.7(?=<regex>)//获取前面部分
6.4.8(?:<regex>)//获取所有
6.4.9(?!<regex>)//获取不是指定内容的前面部分
6.4.10?//在带有+或者*的正则表达式后加上，变为非贪婪爬取
```

# 7.Pattern和Matcher

## 7.1Pattern

```
7.1.1public static Pattern compile(String regex)
7.1.2public Matcher matcher(CharSequence input)//String实现了CharSequence接口
```

## 7.2Matcher

```
7.2.1public boolean find()
7.2.2public String group()//获取匹配的子字符串，只能在find()返回true后使用
7.2.3public boolean matches()//尝试将整个字符串与正则表达式进行匹配
7.2.4public String group(int group)//获取特定分组（即正则表达式中用括号括起来的部分）的匹配，其中0号代表正则表达式本身
```

# 8.正则表达式在字符串方法中的使用

## 8.1public boolean matches(String regex)//判断字符串是否满足正则表达式的规则

```
public boolean matches(String regex)//判断字符串是否满足正则表达式的规则
```

## 8.2public String replaceAll(String regex, String newStr)//按照正则表达式的规则进行替换

```
public String replaceAll(String regex, String newStr)//按照正则表达式的规则进行替换
```

## 8.3public String[] split(String regex)//按照正则表达式的规则切割字符串

```
public String[] split(String regex)//按照正则表达式的规则切割字符串
```

# 9.捕获分组

## 9.1分组原则:只看左括号，第几个左括号就是第几组的开始（第0组是正则表达式本身）

```
分组原则:只看左括号，第几个左括号就是第几组的开始（第0组是正则表达式本身）
```

## 9.2分组的使用

```
9.2.1内部:\n（代表复用第n组的内容）
9.2.2外部:$n（如在String的replaceAll方法，有regex和replacement两个参数，第二个参数就可以通过$n使用第一个参数的第n组内容）
```

# 10.非捕获分组:分组之后不需要再用本组数据，仅仅是把数据括起来，不占用组号

## 10.1(?:)

```
(?:)
```

## 10.2(?=)

```
(?=)
```

## 10.3(?\!)

```
(?!)
```

# 11.jdk7以前时间相关类

## 11.1Date

### 11.1.1构造

```
11.1.1.1public Date()//根据系统当前时间构造
11.1.1.2public Date(long time)//自定义时间
```

### 11.1.2Date方法

```
11.1.2.1public long getTime()
11.1.2.2public setTime(long time)
```

## 11.2SimpleDateFormat

### 11.2.1构造

```
11.2.1.1public SimpleDateFormat()//构造一个SimpleDateFormat，使用默认格式
11.2.1.2public SimpleDateFormat(String pattern)//构造一个SimpleDateFormat，使用指定格式
```

### 11.2.2方法

```
11.2.2.1public final String format(Date date)//格式化，将日期对象转换为字符串
11.2.2.2public Date parse(String source)//解析，将字符串转换为日期对象
```

## 11.3Calendar（月份为0-11）

### 11.3.1概述

```
11.3.1.1Calendar代表了系统当前时间的日历对象，可以单独修改，获取时间中的年，月，日
11.3.1.2Calendar是一个抽象类，不能直接创建对象（无构造）
```

### 11.3.2创建对象的方法:public static Calendar getInstance()//获取对象

```
public static Calendar getInstance()//获取对象
```

### 11.3.3常用方法

```
11.3.3.1public final Date getTime()//获取日期对象
11.3.3.2public final setTime(Date date)//给日历设置日期对象
11.3.3.3public long getTimeInMillis()//获取毫秒值
11.3.3.4public void setTimeInMillis(long millis)//给日历设置时间毫秒值
11.3.3.5public int get(int field)//取日历中的某个字段信息
11.3.3.6public void set(int field, int value)//修改日历中某个字段信息
11.3.3.7public void add(int field, int amount)//为某个字段增加/减少指定的值
```

# 12.jdk8新增时间相关类

## 12.1ZoneId//时区

```
12.1.1static Set<String> getAvailableZoneIds()//获取Java中支持的所有时区
12.1.2static ZoneId systemDefault()//获取系统默认时区对象
12.1.3static ZoneId of(String zoneId)//获取指定的时区对象
```

## 12.2Instant//时间戳

```
12.2.1static Instant now()//获取当前时间的Instant对象（标准时间）
12.2.2static Instant of...(long epochMilli)//根据秒/毫秒/纳秒获取Instant对象
12.2.3ZoneDateTime atZone(ZoneId zone)//指定时区，返回ZoneDateTime对象
12.2.4boolean is...(Instant otherInstant)//判断系列的方法
12.2.5Instant minus...(long millisToSubtract)//减少时间系列的方法
12.2.6Instant plus...(long millisToSubtract)//增加时间系列的方法
```

## 12.3ZoneDateTime//带时区的时间

```
12.3.1static ZonedDateTime now()//获取当前时间的ZoneDateTime对象
12.3.2static ZonedDateTime of...(...)//获取指定时间的ZoneDateTime对象
12.3.3ZonedDateTime with...(...)//修改时间系列的方法
12.3.4ZonedDateTime minus...(...)//减少时间系列的方法
12.3.5ZonedDateTime plus...(...)//增加时间系列的方法
```

## 12.4DateTimeFormatter

```
12.4.1static DateTimeFormatter ofPattern(<format>)//获取格式对象
12.4.2String format(<timeobj>)//按照指定方式格式化
```

## 12.5LocalDate//年月日

```
LocalDate//年月日
```

## 12.6LocalTime//时分秒

```
LocalTime//时分秒
```

## 12.7LocalDateTime//年月日和时分秒

```
LocalDateTime//年月日和时分秒
```

## 12.8Duration//用于计算两个“时间”间隔（秒、纳秒）

```
Duration//用于计算两个“时间”间隔（秒、纳秒）
```

## 12.9Period//用于计算两个“日期”间隔（年、月、日）

```
Period//用于计算两个“日期”间隔（年、月、日）
```

## 12.10ChronoUnit//用于计算两个“日期”间隔

```
ChronoUnit//用于计算两个“日期”间隔
```

# 13.包装类

## 13.1Integer

### 13.1.1构造

```
13.1.1.1public Integer(int value)//根据传递的整数创建一个Integer对象
13.1.1.2public Integer(String s)//根据传递的字符串创建一个Integer对象
13.1.1.3public static Integer valueOf(int i)//根据传递的整数创建一个Integer对象
13.1.1.4public static Integer valueOf(String s)//根据传递的字符串创建一个Integer对象
13.1.1.5public static Integer valueOf(String s, int radix)//根据传递的字符串和进制创建一个Integer对象
```

### 13.1.2方法

```
13.1.2.1public static String toBinaryString(int i)
13.1.2.2public static String toOctalString(int i)
13.1.2.3public static String toHexString(int i)
13.1.2.4public static int parseInt(String s)//将字符串类型的整数转换为int类型的整数
```

### 13.1.3特性

```
13.1.3.1自动完成int和Integer的转换，即可以将int直接赋值给Integer对象，也可以将Integer对象直接赋值给int，更可以用Integer对象之间运算
13.1.3.2public static Integer valueOf()会使用在类内部预先创建好的值为-128~127的Integer数组，这样在使用该方法时可以达到复用的效果，这是public Integer()不会的
```