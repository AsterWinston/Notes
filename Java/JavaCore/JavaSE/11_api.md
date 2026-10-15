JDK 里除了 `String` 和集合，还有一大批"平时随手就用、但很少深究"的类和工具：`Math`、`System`、`Runtime`、`Object`、`Objects`、`BigInteger`、`BigDecimal`，以及**正则表达式**和**日期时间 API** 这两块自成体系的内容。本篇按**包与功能分章**把它们串起来，重点讲清每个 API **什么时候用、底层怎么实现、有哪些坑**；尤其是 `Object` 的 `equals/hashCode` 契约、`BigDecimal` 的精度陷阱、正则的贪婪与回溯、`SimpleDateFormat` 的线程不安全，这四点都是真实项目里出过事故的地方。

# 1. `Math`：数学运算

## 1.1 基本用法

`Math` 是一个 **`final` 类、构造方法私有、全部方法都是 `static`** 的工具类——它不需要也无法实例化，所有功能都通过类名直接调用。除了方法，它还提供两个常用常量：`Math.PI`、`Math.E`。

| 方法 | 说明 |
|------|------|
| `static int abs(int a)` | 绝对值（有 `int`/`long`/`float`/`double` 四个重载） |
| `static double ceil(double a)` | 向上取整，返回 **`double`** |
| `static double floor(double a)` | 向下取整，返回 **`double`** |
| `static long round(double a)` | 四舍五入，返回 **`long`**（`float` 参数返回 `int`） |
| `static int max(int a, int b)` / `static int min(int a, int b)` | 最大/最小值 |
| `static double pow(double a, double b)` | `a` 的 `b` 次方 |
| `static double sqrt(double a)` | 平方根 |
| `static double cbrt(double a)` | 立方根 |
| `static double random()` | 返回 `[0.0, 1.0)` 的伪随机 `double` |

```java
System.out.println(Math.abs(-5));               // 5
System.out.println(Math.ceil(3.1));             // 4.0
System.out.println(Math.floor(3.9));            // 3.0
System.out.println(Math.round(3.5));            // 4
System.out.println(Math.round(-1.5));           // -1
System.out.println(Math.max(3, 7));             // 7
System.out.println(Math.pow(2, 10));            // 1024.0
System.out.println(Math.sqrt(16));              // 4.0
System.out.println(Math.abs(Integer.MIN_VALUE)); // -2147483648
```

> **陷阱**：`Math.round(-1.5)` 的结果是 **`-1` 而不是 `-2`**。因为 `round` 的底层实现是 `floor(x + 0.5)`，`-1.5 + 0.5 = -1.0`，向下取整仍是 `-1`。涉及负数分账、退款金额时，这个"四舍五入"的方向和直觉不一致，务必用测试确认。

> **陷阱**：`Math.abs(Integer.MIN_VALUE)` 的结果**还是负数**（`-2147483648`）。因为 int 的最小值没有对应的正数（`2^31 - 1` 才是最大正数），取绝对值后**溢出回原值**。JDK15 起提供了 `Math.absExact(int)`，遇到这种情况直接抛 `ArithmeticException`，比静默返回错误值安全得多。

> **注意**：`ceil` 和 `floor` 返回的是 `double`，如果后续要当整数用，记得强转——`(int) Math.ceil(3.1)` 才是 `4`。

## 1.2 随机数与唯一标识

`Math.random()` 只能生成 `[0.0, 1.0)` 的 `double`，既不能指定范围，也不能固定种子（无法复现），实际项目里更常用 `java.util.Random`：

```java
// 生成 [0, 100) 的随机整数
java.util.Random random = new java.util.Random();
int n = random.nextInt(100);
System.out.println(n);                     // 例如 42

// 生成 [1, 100] 的随机整数：nextInt(bound) 的上界是开区间，所以 +1
int m = random.nextInt(100) + 1;

// 高并发场景用 ThreadLocalRandom，避免多线程争抢同一个 Random 的种子
int k = java.util.concurrent.ThreadLocalRandom.current().nextInt(10);
```

生成**全局唯一标识**（订单号、链路 traceId、临时文件名）则用 `UUID`：

```java
String id = java.util.UUID.randomUUID().toString();
System.out.println(id);     // 例如 6f3a8f2e-1c4b-4a0e-9d3c-0b2f1a7e5d10，固定 36 个字符
```

> **注意**：`UUID.randomUUID()` 基于随机数生成，理论上**不保证绝对唯一**，但概率低到可以忽略（122 位随机）。它的优点是无中心、生成快；缺点是**没有顺序性**，直接做主键会让数据库索引页频繁分裂——生产环境更常用雪花算法（Snowflake）这类"时间有序 + 机器号"的方案。

# 2. `System`：系统级操作

`System` 同样是 `final` 类 + 私有构造 + 全静态方法，代表"当前 JVM 运行所在的系统"。最常用的是它的三个标准流字段和三个核心方法。

## 2.1 标准流与退出

| 成员 | 说明 |
|------|------|
| `System.out` | 标准输出流（`PrintStream`），`System.out.println()` 就来自它 |
| `System.err` | 标准错误流，输出通常是红色，用于打印异常、警告 |
| `System.in` | 标准输入流（`InputStream`），配合 `Scanner` 使用 |
| `static void exit(int status)` | 终止当前正在运行的 JVM |

```java
System.out.println("正常输出");
System.err.println("错误输出");     // 通常显示为红色，且与 System.out 的缓冲顺序不保证一致
```

`exit(int status)` 的参数约定：**`0` 表示正常退出，非 `0` 表示异常退出**。

```java
System.out.println("开始");
System.exit(0);
System.out.println("这行永远不会执行");   // 不可达
```

> **陷阱**：`System.exit(0)` 会**立刻终止整个 JVM**，`finally` 块不会执行、`ShutdownHook` 之外的后台线程会被直接掐断、数据库连接不会优雅关闭。业务代码里几乎不应该调用它——只有命令行工具"算完就退"、或守护进程收到致命错误要快速失败时才用。在 Web 应用里调用 `System.exit` 等于把整个服务干掉。

## 2.2 时间戳：`currentTimeMillis` 与 `nanoTime`

| 方法 | 说明 |
|------|------|
| `static long currentTimeMillis()` | 返回当前时间距离**时间原点**的毫秒数 |
| `static long nanoTime()` | 返回一个高精度的纳秒计时值，**只用于测耗时**，无绝对含义 |

**时间原点（Epoch）是 `1970-01-01 00:00:00 UTC`**，这是 Unix 系统的约定，Java 沿用了下来。`currentTimeMillis` 返回的就是"现在距离那个时刻过了多少毫秒"，它是一个**绝对时刻**的表达，与时区无关——时区只在把它格式化成"年月日时分秒"时才起作用。

```java
long now = System.currentTimeMillis();
System.out.println(now);                       // 例如 1792000000000
System.out.println(now / 1000 / 60 / 60 / 24); // 约 20740，即从 1970 年至今的天数
```

`nanoTime` 则是**单调递增**的计时值，专门用来测量一段代码执行了多久：

```java
long start = System.nanoTime();
// ... 待测代码 ...
long cost = System.nanoTime() - start;
System.out.println("耗时 " + cost / 1_000_000 + " ms");
```

> **注意**：**测性能必须用 `nanoTime`，不能用 `currentTimeMillis`**。因为 `currentTimeMillis` 依赖系统墙上时钟，NTP 校时、用户手动改时间都可能让"结束时间"比"开始时间"还小，算出负数；`nanoTime` 是 JVM 从某个任意起点开始的单调计数器，不受时钟调整影响，也没有"跨时区"问题。`currentTimeMillis` 只用来表达"某个绝对时刻"。

## 2.3 数组拷贝：`arraycopy`

```java
public static void arraycopy(Object src, int srcPos,
                             Object dest, int destPos, int length)
```

`arraycopy` 是 `native` 方法，由 JVM 用底层内存复制指令实现，**比 `for` 循环逐个赋值快得多**，是 `ArrayList` 扩容、`Arrays.copyOf`、集合增删的底层基石。

| 参数 | 含义 |
|------|------|
| `src` | 源数组 |
| `srcPos` | 从源数组的哪个索引开始拷 |
| `dest` | 目标数组 |
| `destPos` | 拷贝到目标数组的哪个索引开始 |
| `length` | 拷贝多少个元素 |

```java
int[] src = {1, 2, 3, 4, 5};
int[] dest = new int[5];
System.arraycopy(src, 0, dest, 1, 3);      // 把 src 的 [0,3) 拷到 dest 的索引 1 起
System.out.println(java.util.Arrays.toString(dest));   // [0, 1, 2, 3, 0]

// 同一个数组内部拷贝也是允许的，JVM 会正确处理重叠区域
int[] arr = {1, 2, 3, 4, 5};
System.arraycopy(arr, 0, arr, 1, 4);
System.out.println(java.util.Arrays.toString(arr));    // [1, 1, 2, 3, 4]
```

三条必须记住的规则：

1. **基本类型数组要求类型完全一致**——`int[]` 只能拷给 `int[]`，`int[]` 拷给 `long[]` 会抛 `ArrayStoreException`；
2. **引用类型数组支持"子类拷给父类"**（数组协变），但反向或运行期类型不符会抛 `ArrayStoreException`；
3. **`src` 和 `dest` 可以是同一个数组**，JVM 内部会正确处理重叠。

具体到异常：

| 情况 | 抛出的异常 |
|------|-----------|
| 索引越界（`srcPos`、`destPos`、`length` 导致超出数组范围） | `IndexOutOfBoundsException` |
| `src` 或 `dest` 不是数组 | `ArrayStoreException` |
| 元素类型不兼容（基本类型不一致，或引用类型运行期不匹配） | `ArrayStoreException` |
| `src` 或 `dest` 为 `null` | `NullPointerException` |

```java
int[] a = {1, 2, 3};
int[] b = new int[3];
try {
    System.arraycopy(a, 0, b, 0, 5);       // 5 > 数组长度 3
} catch (IndexOutOfBoundsException e) {
    System.out.println("越界：" + e.getClass().getSimpleName());   // 越界：ArrayIndexOutOfBoundsException
}
```

> **注意**：`arraycopy` 报越界时抛的具体类型是 `ArrayIndexOutOfBoundsException`（它是 `IndexOutOfBoundsException` 的子类），所以 catch 父类即可。这个细节经常被写进面试题。

> **注意**：`arraycopy` 做的是**浅拷贝**。如果数组元素是对象，复制过去的是**同一个对象引用**，两个数组下标改的是同一个对象——需要深拷贝时得自己逐个 clone 或用序列化。

## 2.4 其他常用方法

| 方法 | 说明 |
|------|------|
| `static String getProperty(String key)` | 读取系统属性，如 `"os.name"`、`"java.version"`、`"user.dir"` |
| `static String getenv(String name)` | 读取环境变量，如 `"PATH"`、`"JAVA_HOME"` |
| `static String lineSeparator()` | 返回当前平台的行分隔符（Windows `\r\n`，Linux `\n`） |
| `static void gc()` | 建议 JVM 执行垃圾回收，**只是建议，不保证执行** |

```java
System.out.println(System.getProperty("java.version"));   // 17.0.9 之类
System.out.println(System.getProperty("os.name"));        // Windows 11
System.out.println(System.getenv("JAVA_HOME"));
```

> **注意**：`System.gc()` 只是**建议**，JVM 完全可以忽略它。生产代码里不要调用它——显式触发一次 Full GC 会造成长时间停顿（STW），比自然 GC 危害更大。真正要控制 GC 行为应该靠 JVM 参数和堆内存规划。

# 3. `Runtime`：运行时环境

## 3.1 单例设计

`Runtime` 代表**当前 Java 进程的运行环境**，它可以获取 CPU 核数、内存信息、执行外部命令。它有两个特点：

1. **私有构造方法**——外部无法 `new Runtime()`；
2. **通过 `Runtime.getRuntime()` 获取全局唯一的实例**。

这正是**饿汉式单例模式**的标准写法：类加载时就创建好唯一实例，对外只暴露一个静态工厂方法。

```java
public class Runtime {
    private static final Runtime currentRuntime = new Runtime();   // 类加载时创建

    private Runtime() {}                                           // 私有构造，禁止外部 new

    public static Runtime getRuntime() {
        return currentRuntime;                                     // 永远返回同一个对象
    }
}
```

> **注意**：`Runtime` 是 JDK 里最简单的单例示范——**私有构造 + 静态常量 + 静态获取方法**。它的好处是天生的线程安全（实例在类初始化阶段就建好了，由 JVM 保证可见性），缺点是"不管用不用都先创建"。这是懒汉式和饿汉式单例的核心区别，第 30 篇讲设计模式时会再展开。

## 3.2 常用方法

| 方法 | 说明 |
|------|------|
| `static Runtime getRuntime()` | 获取当前系统的运行环境对象（唯一实例） |
| `void exit(int status)` | 终止 JVM，`0` 正常、非 `0` 异常，等价于 `System.exit` |
| `int availableProcessors()` | 获取可用 CPU 逻辑核数 |
| `long maxMemory()` | JVM 最多能从系统获取的内存（字节），受 `-Xmx` 控制 |
| `long totalMemory()` | JVM 当前已经从系统申请到的内存总量（字节） |
| `long freeMemory()` | `totalMemory` 中尚未使用的部分（字节） |
| `Process exec(String command)` | 执行系统命令，返回 `Process` 对象 |

```java
Runtime rt = Runtime.getRuntime();
System.out.println(rt.availableProcessors());                 // 例如 8
System.out.println(rt.maxMemory() / 1024 / 1024 + " MB");     // 例如 4096 MB
System.out.println(rt.totalMemory() / 1024 / 1024 + " MB");   // 例如 256 MB
System.out.println(rt.freeMemory() / 1024 / 1024 + " MB");    // 例如 240 MB
```

三个内存方法的区别是理解 JVM 堆的关键：

| 方法 | 含义 | 关系 |
|------|------|------|
| `maxMemory()` | 堆的**上限**，由 `-Xmx` 决定 | 最大 |
| `totalMemory()` | 堆**已经向操作系统申请**的大小，会随 GC 动态增长 | 介于两者之间 |
| `freeMemory()` | `totalMemory` 里**尚未分配**给对象的部分 | —— |

> **注意**：**`totalMemory()` 不是常量，而是随程序运行逐渐增长**。JVM 的堆是"用多少申请多少"，启动时只申请一小块，不够了再向操作系统要，直到触及 `maxMemory`。所以"已用内存"应该算 `totalMemory() - freeMemory()`，而不是直接看 `freeMemory()`。另注意方法名是 `totalMemory` 而不是常见的拼写错误 `totalMomery`。

`availableProcessors()` 最典型的用途是**设置线程池大小**：

```java
int cores = Runtime.getRuntime().availableProcessors();
// CPU 密集型任务：核数 + 1
// IO 密集型任务：核数 × (1 + 等待时间/计算时间)，倍数常取 2~4
```

> **注意**：`availableProcessors()` 返回的是**逻辑核数**（含超线程），且在容器里可能返回宿主机核数——JDK8 早期版本在 Docker 中读不到 cgroup 限制，会误以为有几十个核，导致线程池开得过大。JDK10 之后才支持容器感知。

## 3.3 `exec` 与 `ProcessBuilder`

`exec` 用于在 Java 进程里调用外部命令：

```java
Runtime rt = Runtime.getRuntime();
Process process = rt.exec("notepad.exe");        // Windows 下打开记事本
// Process 可查看输出、等待结束、强制终止
// process.destroy();
```

> **最佳实践**：`Runtime.exec` 功能有限（不能方便地合并标准错误流、不能设置工作目录、不能改环境变量），**JDK5 之后应该优先用 `java.lang.ProcessBuilder`**。而且 `Runtime.exec` 自 JDK18 起已被标记废弃。更常见的做法是：能用 Java 库实现的功能就别调外部命令，因为 `exec` 有安全风险（命令注入）、跨平台差、还容易因输出流没读干净而**卡死**——子进程缓冲区写满后会阻塞，父进程却在 `waitFor()`，形成死锁。

# 4. `Object`：所有类的根

## 4.1 `Object` 的地位

**`Object` 是 Java 中所有类的顶级父类**——任何类都直接或间接继承它（写不写 `extends Object` 都一样，编译器会自动补上）。这意味着 **`Object` 里的方法可以被所有对象访问**，也意味着"任何对象都能被赋给 `Object` 类型的变量"，这是泛型、集合、反射、`equals` 众多机制的地基。

```java
Object o1 = "字符串";       // String 也是 Object
Object o2 = 123;            // 自动装箱成 Integer，也是 Object
Object o3 = new int[5];     // 数组也是 Object
System.out.println(o3 instanceof Object);   // true
```

## 4.2 `Object` 的 11 个方法总览

`Object` 一共有 **11 个方法**（其中 `wait` 有 3 个重载），它们构成了所有 Java 对象的通用能力：

| 方法 | 作用 |
|------|------|
| `public String toString()` | 返回对象的字符串表示，默认是 `类名@十六进制hashCode` |
| `public boolean equals(Object obj)` | 判断两个对象是否"相等"，**默认比较地址** |
| `public native int hashCode()` | 返回对象的哈希码，供 `HashMap`/`HashSet` 使用 |
| `public final native Class<?> getClass()` | 返回对象的运行时类，**`final` 不能重写** |
| `protected native Object clone()` | 浅拷贝对象，类必须实现 `Cloneable` 否则抛异常 |
| `protected void finalize()` | 对象被 GC 回收前调用，**已废弃，不要用** |
| `public final void wait()` | 让当前线程等待，直到被唤醒（必须在同步块内） |
| `public final void wait(long timeout)` | 等待指定毫秒 |
| `public final void wait(long timeout, int nanos)` | 等待指定毫秒 + 纳秒 |
| `public final void notify()` | 随机唤醒一个在该对象上等待的线程 |
| `public final void notifyAll()` | 唤醒所有在该对象上等待的线程 |

> **注意**：`wait`、`notify`、`notifyAll` 被 `Object` 而非 `Thread` 持有，是因为**锁的粒度是"对象"而不是"线程"**——等待唤醒必须绑定在某个具体的锁对象上。这三个方法都必须在 `synchronized` 块内、用锁对象调用，否则抛 `IllegalMonitorStateException`（详见第 17 篇）。

## 4.3 `toString`

默认实现是 `getClass().getName() + "@" + Integer.toHexString(hashCode())`，也就是形如 `com.example.Person@1b6d3586` 的字符串。它对调试几乎没用，所以**实体类都应该重写它**：

```java
public class Person {
    private String name;
    private int age;

    @Override
    public String toString() {
        return "Person{name='" + name + "', age=" + age + "}";
    }
}
```

```java
Person p = new Person("张三", 20);
System.out.println(p);     // Person{name='张三', age=20}——println 会自动调用 toString()
```

> **行业实践**：现代项目里 `toString` 基本都交给 **Lombok 的 `@ToString`** 或 IDE 自动生成。打印日志时直接 `log.info("订单：{}", order)` 就能输出完整内容，靠的就是 `toString`。但要注意**别把密码、身份证号等敏感字段放进 `toString`**，日志泄露是常见的安全事故。

## 4.4 `equals` 与 `hashCode` 的契约

这是 `Object` 里最重要的一条规则，也是面试与线上事故的双重高发区。

**首先看默认行为**：`Object.equals` 比较的是**地址**，即 `this == obj`；`hashCode` 默认由对象地址派生（具体实现依赖 JVM，HotSpot 默认是随机数或地址派生）。也就是说，默认情况下"两个不同的对象永远不相等"。

**重写 `equals` 时，必须同时重写 `hashCode`**，因为二者之间有严格契约：

1. 同一个对象多次调用 `hashCode`，只要 `equals` 用到的字段没变，返回值必须一致；
2. **`equals` 相等的两个对象，`hashCode` 必须相等**；
3. `equals` 不等的两个对象，`hashCode` 尽量不同（不强制，但哈希表性能依赖这一点）。

违反第 2 条的后果非常具体：**对象存进 `HashMap`/`HashSet` 后就再也取不出来了**。

```java
// 反例：只重写 equals，不重写 hashCode
class User {
    String name;

    User(String name) { this.name = name; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User)) return false;
        return name.equals(((User) o).name);
    }
    // 没有重写 hashCode！
}

java.util.Set<User> set = new java.util.HashSet<>();
set.add(new User("张三"));

System.out.println(set.contains(new User("张三")));   // false！明明"相等"却找不到
System.out.println(set.size());                       // 1，但 add 第二个还会再加进去

java.util.Map<User, String> map = new java.util.HashMap<>();
map.put(new User("张三"), "VIP");
System.out.println(map.get(new User("张三")));         // null！取不出来
```

原因在于 `HashMap` 的查找分两步：**先用 `hashCode` 定位到哪个桶，再用 `equals` 在桶内比较**。两个"相等"的对象 `hashCode` 不同，第一步就被分到了不同的桶里，后面的 `equals` 根本没机会执行。

正确的做法是两个方法一起重写：

```java
class User {
    private String name;
    private int age;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;                              // 同一引用直接相等
        if (o == null || getClass() != o.getClass()) return false; // 类型不同不相等
        User other = (User) o;
        return age == other.age && java.util.Objects.equals(name, other.name);
    }

    @Override
    public int hashCode() {
        // Objects.hash 根据所有参与 equals 的字段生成哈希，顺序一致即可
        return java.util.Objects.hash(name, age);
    }
}
```

```java
java.util.Set<User> set = new java.util.HashSet<>();
set.add(new User("张三", 20));
System.out.println(set.contains(new User("张三", 20)));   // true，这次找得到了
```

> **陷阱**：**只要重写了 `equals`，就必须重写 `hashCode`**，这条规则没有例外。反过来，只重写 `hashCode` 不重写 `equals` 也有问题：两个 `equals` 不相等的对象哈希值相同，会全部挤在一个桶里，`HashMap` 退化成链表，查询从 `O(1)` 掉到 `O(n)`。两者必须成对出现。

> **最佳实践**：用 `java.util.Objects.hash(...)` 生成哈希、用 `Objects.equals(...)` 做字段比较，它们都天然处理了 `null`。实体类用 Lombok 的 `@EqualsAndHashCode` 更省事——它会自动挑非静态、非瞬态的字段参与计算。**参与 `equals` 和 `hashCode` 的字段必须一致**，否则又会违反契约。

> **注意**：用于 `HashMap` 键的对象**必须是不可变的**（或至少 `hashCode` 依赖的字段不可变）。如果对象存进 `HashMap` 后修改了参与 `hashCode` 的字段，哈希值变了，同样会"取不出来"。这正是 `String`、`Integer` 适合做键，而可变实体类不适合的原因。

## 4.5 `clone` 与标记接口

`clone()` 用于复制对象，但它是个**浅拷贝**（只复制字段值，对象字段仍指向同一个对象），而且用起来很别扭：**类必须实现 `Cloneable` 接口，否则调用时抛 `CloneNotSupportedException`**。

```java
public class Sheep implements Cloneable {     // 必须实现这个接口
    private String name;

    @Override
    public Sheep clone() throws CloneNotSupportedException {
        return (Sheep) super.clone();          // 调用 Object 的 native 实现
    }
}
```

**`Cloneable` 是一个"标记性接口"**——它里面没有任何抽象方法，唯一的作用就是"打个标记"，告诉 JVM"这个类允许被 clone"。没有这个标记，`Object.clone()` 会直接抛异常。JDK 里类似的标记接口还有 `Serializable`、`RandomAccess`。

> **行业实践**：**新代码不要用 `clone()`**。它有几个硬伤：浅拷贝语义容易误用、绕过构造方法、和 `final` 字段冲突、异常处理别扭。要复制对象，推荐**拷贝构造方法**（`new Person(other)`）、**静态工厂**（`Person.copyOf(other)`），或者用序列化/JSON 做深拷贝。需要"分离的副本"时，明确写出你的意图比依赖 `clone` 的隐式语义更安全。

> **注意**：`finalize()` 曾经用于"对象回收前做清理"，但它的调用时机完全不可控、会拖慢 GC、还可能让对象"复活"，已经在 JDK9 被标记废弃、JDK18 标记为"未来删除"。**清理资源请用 `try-with-resources` 或显式 `close()`**。

# 5. `Objects`：空安全工具类

`Objects`（注意有 `s`）是 JDK7 引入的工具类，专门解决"`null` 判断写起来啰嗦又容易漏"的问题。`Object` 是根类，`Objects` 是工具类，两者没有继承关系。

## 5.1 `Objects.equals`：空安全比较

```java
public static boolean equals(Object a, Object b)
```

它的逻辑是：**两个都是 `null` 返回 `true`，只有一个 `null` 返回 `false`，都不为 `null` 时再调用 `a.equals(b)`**。这样就彻底避免了 `a.equals(b)` 里 `a` 为 `null` 时的 NPE。

```java
System.out.println(Objects.equals(null, null));     // true
System.out.println(Objects.equals(null, "a"));      // false
System.out.println(Objects.equals("a", "a"));       // true

String a = null, b = "x";
System.out.println(a.equals(b));                    // ✗ NPE
System.out.println(Objects.equals(a, b));           // ✓ false，安全
```

## 5.2 `requireNonNull`：参数校验

```java
public static <T> T requireNonNull(T obj)
public static <T> T requireNonNull(T obj, String message)
```

`obj` 为 `null` 时抛 `NullPointerException`（可带自定义信息），否则**原样返回 `obj`**。这个"原样返回"的设计让它能直接嵌在赋值语句里，是参数校验最简洁的写法：

```java
public void register(String username, Integer age) {
    this.username = Objects.requireNonNull(username, "用户名不能为空");
    this.age = Objects.requireNonNull(age, "年龄不能为空");
}
```

> **最佳实践**：**在方法入口处快速失败**。让错误尽早、带着清晰信息暴露在调用边界，而不是等到三层调用之后才在某个 `intValue()` 上炸出一个不带上下文的 NPE。这是"防御式编程"最廉价也最有效的形式。

## 5.3 其他常用方法

| 方法 | 说明 |
|------|------|
| `static boolean isNull(Object obj)` | 是否为 `null`，比 `obj == null` 更像"业务语义"（可选） |
| `static boolean nonNull(Object obj)` | 是否非 `null`，常用于 Stream 的 `filter(Objects::nonNull)` |
| `static String toString(Object o)` | 空安全 `toString`，`null` 得到 `"null"` |
| `static String toString(Object o, String nullDefault)` | `null` 时返回指定的默认值 |
| `static int hash(Object... values)` | 根据多个字段生成哈希值，**重写 `hashCode` 时用** |
| `static int compare(T a, T b, Comparator c)` | 空安全比较，`null` 排在前面 |

```java
System.out.println(Objects.toString(null));              // null
System.out.println(Objects.toString(null, "默认值"));     // 默认值

// filter(Objects::nonNull) 是过滤集合中 null 元素的惯用写法
java.util.List<String> list = new java.util.ArrayList<>(java.util.Arrays.asList("a", null, "b"));
list.removeIf(Objects::isNull);                          // 等价于 filter
System.out.println(list);                                // [a, b]
```

> **注意**：`Objects.hash(...)` 底层是 `Arrays.hashCode(new Object[]{...})`，每次调用都会**创建一个数组**，对性能极度敏感的路径（比如每秒百万次调用的 `hashCode`）可以手写 `31 * result + field` 的形式。但对绝大多数业务对象，`Objects.hash` 的可读性和正确性都更值得。

# 6. `BigInteger` 与 `BigDecimal`

基本类型的数值范围有限：`long` 最大约 `9.2 × 10^18`，`double` 有 15 位左右有效数字且无法精确表示十进制小数。**涉及超大整数或金额计算时，必须换成 `BigInteger` 和 `BigDecimal`。**

## 6.1 `BigInteger`：超出 `long` 的大整数

`BigInteger` 用于表示**任意精度的整数**，内部用一个 `int[]` 数组按二进制分段存储数值。

| 构造方法 | 说明 |
|----------|------|
| `BigInteger(String val)` | 按十进制字符串创建（最常用） |
| `BigInteger(String val, int radix)` | 按指定进制创建 |
| `BigInteger(int numBits, Random rnd)` | 生成 `0 ~ 2^numBits - 1` 的随机大整数 |
| `static BigInteger valueOf(long val)` | 从 `long` 创建，是**推荐**的轻量创建方式 |

| 常用方法 | 说明 |
|----------|------|
| `BigInteger add(BigInteger val)` | 加法 |
| `BigInteger subtract(BigInteger val)` | 减法 |
| `BigInteger multiply(BigInteger val)` | 乘法 |
| `BigInteger divide(BigInteger val)` | 除法（整数除法，向下取整） |
| `BigInteger[] divideAndRemainder(BigInteger val)` | 返回 `[商, 余数]` 数组 |
| `BigInteger pow(int exponent)` | 幂运算 |
| `BigInteger mod(BigInteger m)` | 取模 |
| `BigInteger gcd(BigInteger val)` | 最大公约数 |
| `BigInteger max(BigInteger val)` / `min` | 最大/最小值 |
| `boolean equals(Object x)` | 判断两个 `BigInteger` 的数值是否相等 |
| `int compareTo(BigInteger val)` | 比较大小 |
| `int intValue()` / `longValue()` | 转为基本类型 |

```java
BigInteger a = new BigInteger("123456789012345678901234567890");
BigInteger b = new BigInteger("98765432109876543210");

System.out.println(a.add(b));                  // 123456789111111111111111111100
System.out.println(a.multiply(b));             // 很大的一串数字
System.out.println(a.pow(2));                  // 60 位左右的数
System.out.println(a.compareTo(b));            // 1，a 更大

BigInteger[] dr = a.divideAndRemainder(b);
System.out.println("商 = " + dr[0] + "，余数 = " + dr[1]);
```

> **注意**：`BigInteger` 和 `BigDecimal` 一样是**不可变类**。`a.add(b)` **不会改变 `a`**，而是返回一个全新的对象；如果你不接收返回值，运算就白做了。这和 `String` 的 `toUpperCase` 是同一个道理。

> **注意**：`add`、`multiply` 这些方法名是**动词**，不能用 `+`、`*` 运算符——Java 不支持运算符重载。这是从其他语言（或从 `int` 习惯）转过来时最容易犯的错：写 `a + b` 会直接编译不过。

## 6.2 `BigDecimal` 的精度陷阱

`BigDecimal` 用于表示**任意精度的十进制小数**，是**金额计算的标准答案**。它的第一个大坑就在构造方法上：

| 构造方法 | 说明 |
|----------|------|
| `BigDecimal(double val)` | **不推荐**，会把 `double` 的二进制误差原样带进来 |
| `BigDecimal(String val)` | **推荐**，按字面量精确构造 |
| `BigDecimal(int val)` / `BigDecimal(long val)` | 精确构造，可用 |
| `static BigDecimal valueOf(double val)` | 内部用 `Double.toString` 转字符串再构造，结果正确 |

```java
System.out.println(new BigDecimal(0.1));
// 0.1000000000000000055511151231257827021181583404541015625

System.out.println(new BigDecimal("0.1"));       // 0.1
System.out.println(BigDecimal.valueOf(0.1));     // 0.1
```

**为什么会这样**：`double` 用二进制的"符号位 + 指数 + 尾数"表示数值，而 `0.1` 在二进制里是**无限循环小数**，只能截断成最接近的近似值。`new BigDecimal(double)` 会忠实地把这个近似值（一长串数字）完整保留下来。用 `String` 构造时，`BigDecimal` 按十进制字面量解析，完全精确。

> **陷阱**：**`new BigDecimal(0.1)` 是金额计算里最经典的事故来源**。它不会报错、编译也通过，但差值会在累加、对账时逐渐放大。**构造 `BigDecimal` 时永远传字符串**——`new BigDecimal("19.99")` 或 `BigDecimal.valueOf(19.99)`（后者内部先 `Double.toString`，对常见的有限小数能还原出正确结果）。

> **注意**：`double` 自身的精度问题更隐蔽。`System.out.println(0.1 + 0.2)` 输出的是 `0.30000000000000004` 而不是 `0.3`，所以**任何金额、利率、税费计算都不该用 `float`/`double`**。

## 6.3 运算与舍入

| 方法 | 说明 |
|------|------|
| `BigDecimal add(BigDecimal val)` | 加法 |
| `BigDecimal subtract(BigDecimal val)` | 减法 |
| `BigDecimal multiply(BigDecimal val)` | 乘法 |
| `BigDecimal divide(BigDecimal val)` | 除法，**除不尽会抛异常**，见下 |
| `BigDecimal divide(BigDecimal val, int scale, RoundingMode mode)` | 除法并指定精度和舍入模式（**推荐**） |
| `BigDecimal setScale(int scale, RoundingMode mode)` | 设置小数位数并舍入 |
| `BigDecimal stripTrailingZeros()` | 去掉末尾多余的 0 |
| `String toPlainString()` | 用普通十进制输出，不用科学计数法 |
| `BigDecimal abs()` / `negate()` / `pow(int)` | 绝对值 / 取反 / 幂 |
| `int compareTo(BigDecimal val)` | 比较数值大小 |

**除法必须指定精度**，否则除不尽时直接抛异常：

```java
BigDecimal a = new BigDecimal("1");
BigDecimal b = new BigDecimal("3");

// a.divide(b);
// ArithmeticException: Non-terminating decimal expansion; no exact representable decimal result.

System.out.println(a.divide(b, 2, RoundingMode.HALF_UP));    // 0.33
System.out.println(a.divide(b, 10, RoundingMode.HALF_UP));   // 0.3333333333
```

原因和 `1 / 3` 在十进制下是无限循环小数一样：`BigDecimal` 要求结果是**精确的**，`1 ÷ 3` 无法精确表示，所以它拒绝给出一个"近似答案"——除非你明确告诉它"保留几位、怎么舍入"。

`RoundingMode` 常用取值：

| 模式 | 含义 |
|------|------|
| `UP` | 远离零方向舍入（`1.1 → 2`，`-1.1 → -2`） |
| `DOWN` | 向零方向舍入（截断，`1.9 → 1`） |
| `CEILING` | 向正无穷方向（`1.1 → 2`，`-1.1 → -1`） |
| `FLOOR` | 向负无穷方向（`1.9 → 1`，`-1.1 → -2`） |
| `HALF_UP` | 四舍五入（**最常用**） |
| `HALF_DOWN` | 五舍六入 |
| `HALF_EVEN` | 银行家舍入（四舍六入五成双），**金融标准** |
| `UNNECESSARY` | 要求绝对精确，有任何舍入就抛 `ArithmeticException` |

```java
BigDecimal price = new BigDecimal("19.99");
BigDecimal count = new BigDecimal("3");
System.out.println(price.multiply(count));                                   // 59.97
System.out.println(price.divide(new BigDecimal("3"), 2, RoundingMode.HALF_UP)); // 6.66

BigDecimal v = new BigDecimal("1.2300");
System.out.println(v.stripTrailingZeros());      // 1.23
System.out.println(new BigDecimal("1E+3").toPlainString());   // 1000，避免科学计数法
```

> **行业实践**：金融、税务、对账场景**推荐用 `HALF_EVEN`**（银行家舍入）而不是 `HALF_UP`，因为它在大量数据上能避免"舍入总是偏向一方"造成的系统性偏差。但如果业务规则、合同明确写的是"四舍五入"，就按合同来——**舍入模式必须由业务确认，不能由程序员随手选**。

## 6.4 `equals` 与 `compareTo` 的区别

这是 `BigDecimal` 的第二个大坑：

```java
BigDecimal x = new BigDecimal("0.1");
BigDecimal y = new BigDecimal("0.10");

System.out.println(x.equals(y));        // false —— 值和精度（scale）都要相等
System.out.println(x.compareTo(y));     // 0 —— 只比较数值，认为相等
```

`BigDecimal` 的 `equals` **同时比较数值和 `scale`**（小数位数），所以 `0.1` 和 `0.10` 被判为不相等；`compareTo` **只比较数值大小**，返回 `0`。

> **陷阱**：把 `BigDecimal` 用作 `HashMap` 的键、或在集合里做去重/`contains` 时，`0.1` 和 `0.10` 会被当成两个不同的元素。**判断"金额是否相等"应该用 `compareTo(...) == 0`，不要用 `equals`**。这一点和直觉相反，务必记住。

## 6.5 三种数值类型的对比与选择

| 维度 | `double` | `BigInteger` | `BigDecimal` |
|------|----------|--------------|--------------|
| 精度 | 15 位有效数字，有二进制误差 | **任意精度整数，精确** | **任意精度十进制，精确** |
| 表示小数 | 可以但不精确 | 不可以（整数） | **可以且精确** |
| 范围 | 有上限（约 `1.8 × 10^308`） | 无上限（受内存限制） | 无上限 |
| 性能 | **最快**（CPU 原生指令） | 慢 | 慢 |
| 是否为对象 | 基本类型 | 是（不可变） | 是（不可变） |
| 适用场景 | 科学计算、图形、统计 | 超大整数（密码学、大数运算） | **金额、利率、税费** |

> **最佳实践**：**选型的判断标准是"能不能容忍误差"**。涉及钱的场景一律 `BigDecimal` 且用 `String` 构造；纯数学计算、物理仿真用 `double`；只有确确实实会超过 `long` 范围（比如加密算法的密钥运算、大数阶乘）才用 `BigInteger`。`BigDecimal` 用来存整数也是可以的，但业务上没必要。

# 7. 正则表达式基础

**正则表达式（Regular Expression）** 是一套用来描述"字符串模式"的迷你语言，用于校验、查找、替换、分割文本。它独立于 Java 存在，Java 通过 `java.util.regex` 包和 `String` 的几个方法提供支持。

先记住两条使用规范：**在 Java 字符串里写正则，反斜杠要写两遍**（`\d` 要写成 `"\\d"`），因为 Java 编译器会先处理一次转义；**正则的元字符（`.`、`*`、`+`、`?`、`()`、`[]`、`{}`、`^`、`$`、`|`）在匹配字面量时要转义**，用 `\` 或放进 `[]` 里。

## 7.1 字符类：只匹配一个字符

用 `[]` 框起来，表示"这个位置上可以是里面任意一个字符"。

| 写法 | 含义 |
|------|------|
| `[abc]` | 只能是 `a`、`b` 或 `c` 中的一个 |
| `[^abc]` | 除了 `a`、`b`、`c` 之外的任何字符 |
| `[a-zA-Z]` | `a` 到 `z`，或 `A` 到 `Z` |
| `[a-d[m-p]]` | `a` 到 `d`，或 `m` 到 `p`（**并集**） |
| `[a-z&&[def]]` | `a-z` 与 `def` 的**交集**，即 `d`、`e`、`f` |
| `[a-z&&[^bc]]` | `a-z` 与非 `bc` 的交集，即 `[ad-z]` |
| `[a-z&&[^m-p]]` | `a-z` 与"除了 `m` 到 `p`"的交集，即 `[a-lq-z]` |

> **注意**：`^` 在 `[]` **里面**表示取反，在 `[]` **外面**表示"行首"。同一个符号两种含义，看位置来区分，这是初学正则最容易混淆的一点。

## 7.2 预定义字符

| 写法 | 含义 |
|------|------|
| `.` | 任意字符（默认**不匹配换行符**） |
| `\d` | 一个数字，即 `[0-9]` |
| `\D` | 一个非数字，即 `[^0-9]` |
| `\s` | 一个空白字符，即 `[\t\n\x0B\f\r]` |
| `\S` | 一个非空白字符 |
| `\w` | `[a-zA-Z_0-9]`（字母、数字、下划线，**不含中文**） |
| `\W` | `[^\w]`，即非单词字符 |

> **陷阱**：`\w` 只包含英文字母、数字、下划线，**不匹配中文**。要匹配"任意非空白字符"应该用 `\S`，要匹配"汉字"用 `[一-龥]`。很多人写用户名校验时用 `^\w+$`，结果中文用户名一律不通过。

> **注意**：`.` 默认不匹配换行，要让 `.` 也能匹配换行需要开启 `DOTALL` 模式（`Pattern.DOTALL`，或在正则里加 `(?s)`）。

## 7.3 数量词

数量词修饰前面的一个字符或一个分组，控制它出现多少次。

| 写法 | 含义 |
|------|------|
| `X?` | 0 次或 1 次 |
| `X*` | 0 次或多次 |
| `X+` | 1 次或多次 |
| `X{n}` | 正好 `n` 次 |
| `X{n,}` | 至少 `n` 次 |
| `X{n,m}` | 至少 `n` 次、不超过 `m` 次 |

```java
// 校验：11 位手机号（1 开头，第二位 3-9，后面 9 位数字）
String phone = "13812345678";
System.out.println(phone.matches("1[3-9]\\d{9}"));     // true

// 校验：6~16 位密码，只能含字母、数字、下划线
System.out.println("abc_123".matches("[a-zA-Z0-9_]{6,16}"));   // true
```

## 7.4 特殊符号

| 写法 | 含义 |
|------|------|
| `[]` | 字符类，里面的内容匹配一次 |
| `()` | 分组 |
| `^` | 在 `[]` 外表示行首；在 `[]` 内表示取反 |
| `$` | 行尾 |
| `&&` | 交集（只能用在 `[]` 内） |
| `\|` | 或（半角竖线），如 `cat\|dog` 匹配 `cat` 或 `dog` |
| `(?i)` | 忽略后面内容的大小写，可用括号绑定指定片段 |
| `(?=regex)` | 正向先行断言：**后面**要跟着 `regex`，但 `regex` 不计入结果（"获取前面部分"） |
| `(?!regex)` | 负向先行断言：**后面**不能是 `regex` |
| `(?:regex)` | 非捕获分组：只分组、不占用组号 |
| `X+?`、`X*?` | 非贪婪（勉强）模式，见 7.5 |

```java
// (?i) 忽略大小写，可用括号绑定范围
System.out.println("AbC".matches("(?i)abc"));            // true
System.out.println("abC".matches("a(?i)bc"));            // true，只忽略后面部分
System.out.println("aBC".matches("(?i)a(?-i)bc"));       // false

// 正则里的"或"用 |，写在括号里可以限定作用范围
System.out.println("cat".matches("cat|dog"));            // true
System.out.println("dog".matches("(cat|dog)"));          // true
```

> **注意**：在正则表达式里"或"是 `|`（半角竖线），而在 `[]` 字符类里写 `|` 只是普通字符。`[abc|def]` 匹配的是 `a`、`b`、`c`、`|`、`d`、`e`、`f` 中任意一个，**不是"abc 或 def"**——后者要写 `(abc|def)`。

## 7.5 贪婪、勉强与独占

数量词默认是**贪婪（Greedy）**的：**先尽可能多地匹配，匹配失败再一个字符一个字符地"吐"回来（回溯）**。这个默认行为在 `<.+>` 这类模式上会产生反直觉的结果。

| 类型 | 写法 | 行为 |
|------|------|------|
| **贪婪** Greedy | `X*`、`X+`、`X{n,m}` | 尽可能多匹配，失败再回溯 |
| **勉强/懒惰** Reluctant | `X*?`、`X+?`、`X{n,m}?` | 尽可能少匹配，不够再回溯着多吃 |
| **独占** Possessive | `X*+`、`X++`、`X{n,m}+` | 尽可能多匹配，**绝不回溯** |

```java
String html = "<a><b>";
System.out.println(html.replaceAll("<.+>", "X"));    // X   —— 贪婪：一口气吃到最后一个 >
System.out.println(html.replaceAll("<.+?>", "X"));   // XX  —— 勉强：遇到第一个 > 就停
```

> **注意**：初学正则处理 HTML、JSON 这类嵌套结构时，`"<.+>"` 会从第一个 `<` 一直吃到最后一个 `>`，把中间内容全吞掉。要"匹配一对标签"得用懒惰模式 `"<.+?>"`。当然，**用正则解析 HTML/XML 本身就是错误做法**，遇到嵌套结构应该用专门的解析器。

> **注意**：贪婪模式靠**回溯**实现"先多吃再吐"，极端情况下（比如 `a*` 匹配一长串 `a` 加一个不匹配的字符）回溯次数会爆炸，形成 **ReDoS 拒绝服务攻击**。**独占模式 `X*+` 不留回溯，能避免这个问题**，代价是它可能错过一些本可匹配的情况。对来源不可控的输入跑正则时（尤其是复杂的嵌套量词），要警惕回溯性能。

## 7.6 一个完整的校验示例

```java
// 校验邮箱：用户名@域名.后缀
String email = "zhangsan@example.com";
String regex = "^[a-zA-Z0-9_]+@[a-zA-Z0-9]+(\\.[a-zA-Z]{2,})+$";
System.out.println(email.matches(regex));     // true
System.out.println("bad@@x".matches(regex));  // false
```

# 8. `Pattern` 与 `Matcher`

## 8.1 为什么用 `Pattern`

`String.matches`、`String.replaceAll`、`String.split` 用起来方便，但它们**每次调用都要重新编译一遍正则表达式**——把正则字符串解析成内部的状态机。在循环里反复调用，编译开销会累积得很可观。

`Pattern` 解决的就是这个问题：**一次编译，多次复用**。`Pattern.compile(regex)` 得到的是一个**不可变、线程安全**的对象，可以放心地当静态常量缓存。

## 8.2 核心用法

```java
Pattern p = Pattern.compile("\\d+");        // 预编译，只做一次
Matcher m = p.matcher("abc123def456");      // 针对具体输入创建匹配器

while (m.find()) {                          // find()：查找下一个匹配
    System.out.println(m.group() + " 位置 [" + m.start() + ", " + m.end() + ")");
}
// 123 位置 [3, 6)
// 456 位置 [9, 12)
```

`Pattern` 与 `Matcher` 的常用方法：

| 类 | 方法 | 说明 |
|----|------|------|
| `Pattern` | `static Pattern compile(String regex)` | 编译正则，返回可复用对象 |
| `Pattern` | `static Pattern compile(String regex, int flags)` | 带标志编译，如 `CASE_INSENSITIVE`、`DOTALL` |
| `Pattern` | `Matcher matcher(CharSequence input)` | 为指定输入创建匹配器（`String` 实现了 `CharSequence`） |
| `Pattern` | `static boolean matches(String regex, CharSequence input)` | 一次性匹配，等价于 `compile(regex).matcher(input).matches()` |
| `Matcher` | `boolean matches()` | 尝试把**整个字符串**与正则匹配（等价于 `String.matches`） |
| `Matcher` | `boolean find()` | 查找**下一个**匹配子串，找到返回 `true` |
| `Matcher` | `boolean find(int start)` | 从指定位置开始查找 |
| `Matcher` | `String group()` | 返回本次匹配到的子串，**只能在 `find()` 返回 `true` 后调用** |
| `Matcher` | `String group(int group)` | 返回第 `group` 组的内容，**`0` 号是正则整体** |
| `Matcher` | `int start()` / `int end()` | 本次匹配的起始/结束索引（左闭右开） |
| `Matcher` | `int groupCount()` | 分组数量 |
| `Matcher` | `String replaceAll(String replacement)` | 按本次匹配替换全部 |
| `Matcher` | `String replaceFirst(String replacement)` | 只替换第一个 |
| `Matcher` | `Matcher reset(CharSequence input)` | 重置匹配器，换一个输入再用 |

> **陷阱**：`matches()` 和 `find()` 的语义完全不同，**这是初学者最容易用错的点**。`matches()` 要求**整个字符串完全匹配**正则（相当于正则自带 `^...$`）；`find()` 只要求**包含**一个能匹配的子串。`"abc123".matches("\\d+")` 是 `false`（因为开头是字母），但 `find()` 能找出 `123`。

> **最佳实践**：**同一个正则在多处使用时，把它编译成 `static final Pattern` 常量**。`Pattern` 线程安全，天然适合做常量。这就是 `String.matches`（每次重新编译）和预编译 `Pattern` 的核心性能差别。

```java
public class Validators {
    // 预编译成常量，全类共用，只编译一次
    private static final Pattern PHONE = Pattern.compile("1[3-9]\\d{9}");
    private static final Pattern EMAIL =
            Pattern.compile("^[a-zA-Z0-9_]+@[a-zA-Z0-9]+(\\.[a-zA-Z]{2,})+$");

    public static boolean isPhone(String s) {
        return s != null && PHONE.matcher(s).matches();
    }
}
```

# 9. 正则表达式在 `String` 方法中的使用

`String` 有四个方法直接接受正则，日常用得最多，但它们是"一次性"的（每次都会重新编译正则）：

| 方法 | 说明 |
|------|------|
| `boolean matches(String regex)` | 判断整个字符串是否满足正则规则 |
| `String replaceAll(String regex, String replacement)` | 按正则替换**全部**匹配项 |
| `String replaceFirst(String regex, String replacement)` | 按正则替换**第一个**匹配项 |
| `String[] split(String regex)` | 按正则切割字符串 |

```java
System.out.println("13812345678".matches("1[3-9]\\d{9}"));       // true

// 把字符串中所有数字替换成 #
System.out.println("a1b22c333".replaceAll("\\d+", "#"));          // a#b#c#

// 只替换第一个数字
System.out.println("a1b22c333".replaceFirst("\\d+", "#"));        // a#b22c333

// 按任意空白字符（一个或多个）切割
System.out.println(java.util.Arrays.toString("a  b\tc".split("\\s+")));   // [a, b, c]

// 按逗号或分号切割（正则的"或"）
System.out.println(java.util.Arrays.toString("a,b;c".split("[,;]")));     // [a, b, c]
```

> **注意**：`replaceAll` 的第二个参数（替换内容）里，`$` 和 `\` 有特殊含义——`$n` 表示引用第 `n` 组，`\` 用来转义。所以**替换内容本身要包含 `$` 或 `\` 时必须写成 `\\$` 或 `\\\\`**。这个坑在拼接带 `$` 的金额符号时经常出现。反过来，如果只是想替换字面量而不是正则，用 `replace` 而不是 `replaceAll`。

# 10. 捕获分组与非捕获分组

## 10.1 捕获分组

**用 `()` 包起来的部分就是一组**。分组规则很简单：**从左到右数左括号，第几个左括号就是第几组**，而 **`0` 号组永远代表正则表达式整体**。

```java
// (\\d{4})-(\\d{2})-(\\d{2})   →  第 1 组是年，第 2 组是月，第 3 组是日
Pattern p = Pattern.compile("(\\d{4})-(\\d{2})-(\\d{2})");
Matcher m = p.matcher("2026-10-15");
if (m.matches()) {
    System.out.println(m.group(0));   // 2026-10-15，整体
    System.out.println(m.group(1));   // 2026
    System.out.println(m.group(2));   // 10
    System.out.println(m.group(3));   // 15
    System.out.println(m.groupCount()); // 3
}
```

## 10.2 分组的两种引用方式

| 使用位置 | 写法 | 说明 |
|----------|------|------|
| **正则内部** | `\n` | 复用第 `n` 组匹配到的内容（**反向引用**） |
| **正则外部** | `$n` | 引用第 `n` 组的内容，如 `replaceAll` 的替换字符串 |

反向引用最典型的用法是**匹配连续重复的字符**：

```java
// (.)\\1 表示"某个字符，后面跟着和它完全相同的一个字符"
System.out.println("aabbccdd".replaceAll("(.)\\1", "$1"));   // abcd，把连续重复的合成一个

// (.)\\1+ 表示同一个字符连续出现两次以上
Pattern p = Pattern.compile("(.)\\1+");
Matcher m = p.matcher("aaabbbcccd");
while (m.find()) {
    System.out.println(m.group() + " 起始索引 " + m.start());
}
// aaa 起始索引 0
// bbb 起始索引 3
// ccc 起始索引 6
```

> **陷阱**：**`\n` 在 Java 字符串里要写成 `"\\n"`**——这不是"换行符"，而是"第 n 组的反向引用"。反过来 `"\n"` 才是换行符。`"(.)\\1"` 和 `"(.)\1"`（错误写法）的区别，就是"引用了第 1 组"和"引用了换行符"的区别，编译期不报错但结果全错。**在正则里 `\1` 是反向引用、`\\1` 才是 Java 源码里的正确写法**，这个双重转义是正则 + Java 组合最常见的一类 bug。

## 10.3 三种分组语法对比

| 写法 | 名称 | 是否占用组号 | 作用 |
|------|------|--------------|------|
| `(regex)` | 捕获分组 | **占用** | 分组、可被 `\n` 或 `$n` 引用 |
| `(?:regex)` | 非捕获分组 | **不占用** | 只想把内容括起来（配合数量词、或"或"），后面不再引用 |
| `(?=regex)` | 正向先行断言 | **不占用** | 后面必须跟着 `regex`，但 `regex` 不出现在结果里 |
| `(?!regex)` | 负向先行断言 | **不占用** | 后面不能是 `regex` |

**非捕获分组的价值**：如果只是想把一段内容"括起来"配合数量词使用，但又不需要引用它，用 `(?:)` 可以让组号不被占用，避免后续 `$n` 的编号被打乱。

```java
// 用捕获分组：第 1 组是 "ab"，组号被占用
System.out.println("ab".repeat(3).matches("(ab)+"));     // true，但占用了 1 号组

// 用非捕获分组：不占用组号
System.out.println("ababab".matches("(?:ab)+"));         // true，组号还是空的

// 对比：groupCount() 的差别
System.out.println(Pattern.compile("(ab)+").matcher("abab").groupCount());    // 1
System.out.println(Pattern.compile("(?:ab)+").matcher("abab").groupCount());  // 0
```

**先行断言的典型用途**是"既要满足条件、又不想让条件进入结果"：

```java
// 只匹配"后面跟着 - 的数字"：断言 (?=-) 要求后面是 -，但不消耗它
Pattern p = Pattern.compile("\\d+(?=-)");
Matcher m = p.matcher("10-20-30");
while (m.find()) System.out.print(m.group() + " ");   // 10 20（30 后面没有 -，不匹配）

// 负向先行断言：匹配后面不是逗号的数字
Pattern p2 = Pattern.compile("\\d+(?![,])");
Matcher m2 = p2.matcher("1,2,3");
while (m2.find()) System.out.print(m2.group() + " "); // 3（1、2 后面都是逗号）
```

> **注意**："先行断言"（lookahead）只看**后面**，`(?=)` 不消耗字符、不占组号。Java 还支持"后行断言"`(?<=regex)` 和 `(?<!regex)`（看前面），但篇幅所限不展开。记住核心区别即可：**`(?=)` 是"后面要是"，`(?!)` 是"后面不能是"**。

# 11. JDK7 及以前的时间类

## 11.1 `Date`

`java.util.Date` 表示一个**精确到毫秒的时刻**，本质是对 `long` 时间戳的一层包装。

| 构造方法 | 说明 |
|----------|------|
| `Date()` | 根据系统当前时间构造 |
| `Date(long time)` | 根据指定的毫秒时间戳构造（原点仍是 1970-01-01 UTC） |

| 方法 | 说明 |
|------|------|
| `long getTime()` | 获取该日期对应的毫秒时间戳 |
| `void setTime(long time)` | 设置毫秒时间戳 |

```java
Date now = new Date();
System.out.println(now.getTime());            // 例如 1792000000000

Date epoch = new Date(0L);                    // 时间原点
System.out.println(epoch);                    // Thu Jan 01 08:00:00 CST 1970（东八区）
```

> **注意**：`new Date(0L)` 打印出来是 `08:00:00` 而不是 `00:00:00`，因为 `Date.toString()` 会按**默认时区**输出。原点本身是 UTC 的 00:00，到了东八区就是 08:00。这说明 **`Date` 存的是"时刻"，时区只在显示时才有意义**。

> **注意**：`Date` 里绝大部分方法（`getYear`、`getMonth`、`getDay`、`getHours` 等）在 JDK1.1 就已废弃，别再用了——它们既有"月份从 0 开始"的坑，也把日期和日历逻辑混在一起。要取字段请用 `Calendar`，或者直接上 JDK8 的时间 API。

## 11.2 `SimpleDateFormat`

`SimpleDateFormat` 负责**日期与字符串之间的互相转换**：格式化（`Date` → `String`）和解析（`String` → `Date`）。

| 构造方法 | 说明 |
|----------|------|
| `SimpleDateFormat()` | 使用默认格式 |
| `SimpleDateFormat(String pattern)` | 使用指定格式 |

| 方法 | 说明 |
|------|------|
| `final String format(Date date)` | 格式化：日期对象 → 字符串 |
| `Date parse(String source)` | 解析：字符串 → 日期对象，**失败抛 `ParseException`（受检异常）** |

常用的模式字母：

| 字母 | 含义 | 示例 |
|------|------|------|
| `yyyy` | 年 | `2026` |
| `MM` | 月（**`MM` 补零，`M` 不补**） | `10` / `10` |
| `dd` | 日 | `15` |
| `HH` | 24 小时制小时 | `22` |
| `mm` | 分 | `30` |
| `ss` | 秒 | `05` |
| `SSS` | 毫秒 | `123` |
| `E` | 星期 | `Thu` |

```java
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
String text = sdf.format(new Date());
System.out.println(text);                       // 2026-10-15 22:30:05

try {
    Date date = sdf.parse("2026-10-15 22:30:05");
    System.out.println(date.getTime());         // 对应的毫秒时间戳
} catch (java.text.ParseException e) {
    System.out.println("日期格式不对：" + e.getMessage());
}
```

### 11.2.1 线程不安全（重点）

**`SimpleDateFormat` 不是线程安全的**。它内部用一个共享的 `Calendar` 实例字段来保存"解析/格式化过程中"的中间状态，多线程同时调用 `format` 或 `parse` 会互相覆盖这个中间状态，导致：

- 输出结果错乱（A 线程拿到 B 线程的日期）；
- 抛 `NumberFormatException`；
- 解析出完全错误的日期。

```java
// 反例：把 SimpleDateFormat 作为静态共享字段
private static final SimpleDateFormat SDF = new SimpleDateFormat("yyyy-MM-dd");

// 10 个线程同时调用 SDF.format(...) → 结果可能错乱甚至抛异常
```

三种正确做法：

```java
// 方案一：每次用都 new 一个（简单，但有对象创建开销）
String s1 = new SimpleDateFormat("yyyy-MM-dd").format(new Date());

// 方案二：用 ThreadLocal 每个线程一份（经典做法）
private static final ThreadLocal<SimpleDateFormat> SDF =
        ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
String s2 = SDF.get().format(new Date());

// 方案三（推荐）：直接用 JDK8 的 DateTimeFormatter，它本身就是线程安全的
private static final java.time.format.DateTimeFormatter FMT =
        java.time.format.DateTimeFormatter.ofPattern("yyyy-MM-dd");
String s3 = FMT.format(java.time.LocalDate.now());
```

> **行业实践**：**新项目一律用 `java.time.format.DateTimeFormatter`，不要再碰 `SimpleDateFormat`**。两者功能完全等价，但 `DateTimeFormatter` 是不可变、线程安全的，可以安全地作为静态常量共享，不用 `ThreadLocal` 也不用每次 `new`。只有维护老代码时才需要处理 `SimpleDateFormat` 的线程安全问题。

## 11.3 `Calendar`

`Calendar` 是一个**日历对象**，可以在 `Date` 的基础上按"年、月、日、时、分、秒"这些**字段**来读改时间。它是一个**抽象类，不能直接 `new`**，要通过静态工厂获取实例。

| 方法 | 说明 |
|------|------|
| `static Calendar getInstance()` | 获取当前时间对应的日历对象 |
| `final Date getTime()` | 获取该日历对应的 `Date` 对象 |
| `final void setTime(Date date)` | 用 `Date` 设置日历 |
| `long getTimeInMillis()` | 获取毫秒时间戳 |
| `void setTimeInMillis(long millis)` | 用毫秒时间戳设置日历 |
| `int get(int field)` | 取某个字段的值（年、月、日等） |
| `void set(int field, int value)` | 修改某个字段 |
| `void add(int field, int amount)` | 给某个字段增减指定值（会自动进位） |

```java
Calendar c = Calendar.getInstance();
System.out.println(c.get(Calendar.YEAR));            // 2026
System.out.println(c.get(Calendar.MONTH));           // 9  ← 注意！10 月的值是 9
System.out.println(c.get(Calendar.DAY_OF_MONTH));    // 15
System.out.println(c.get(Calendar.DAY_OF_WEEK));     // 5（周日=1，周一=2，...）

c.add(Calendar.DAY_OF_MONTH, 7);                     // 加 7 天
System.out.println(c.getTime());                     // 一周后的日期
```

> **陷阱**：**`Calendar` 的月份是从 `0` 开始的**（`0` 是 1 月，`11` 是 12 月）。所以 `get(Calendar.MONTH)` 拿到 10 月时返回 `9`，而 `set(Calendar.MONTH, 10)` 设的是 11 月。这是 `Calendar` 最著名的坑，几乎所有用过它的人都踩过。字段常量 `Calendar.JANUARY` 的值就是 `0`，用它代替魔数能减少犯错。

> **注意**：`Calendar` 的星期是 `SUNDAY = 1`、`MONDAY = 2`……`SATURDAY = 7`，**不是从周一开始的**。用它算"本周第几天"时一定要减一或做映射，否则整体偏移一天。

> **注意**：`Calendar` 和 `Date` 一样是**可变、非线程安全**的，而且 API 设计啰嗦（到处是 `int field` 常量）。JDK8 的时间 API 正是为了取代它而设计的。

## 11.4 老时间 API 的三大缺陷

`Date`、`SimpleDateFormat`、`Calendar` 这一套在 JDK8 被整体替换，原因是三个设计问题：

| 缺陷 | 具体表现 |
|------|----------|
| **线程不安全** | `SimpleDateFormat`、`Calendar` 内部持有可变状态，多线程共享会出错 |
| **可变、易被误改** | `Calendar.set` 直接改自身，方法传参后可能被意外修改，且到处是副作用 |
| **设计混乱、易错** | 月份从 0 开始、年份从 1900 开始、`Date` 既是时刻又是日期、时区处理含糊 |

# 12. JDK8 新增时间 API

JDK8 在 `java.time` 包下重做了一套时间 API（参考 Joda-Time），核心目标是**不可变、线程安全、职责清晰**。

## 12.1 设计思想

| 特性 | 说明 |
|------|------|
| **不可变** | 所有类都是 `final` + 不可变，任何"修改"都返回**新对象**，原对象不变 |
| **线程安全** | 不可变带来了天然线程安全，可以直接作为静态常量共享 |
| **职责清晰** | 按"时刻/日期/时间/时区"拆成不同类，不再一个 `Date` 包打天下 |
| **方法链式** | `with...`、`plus...`、`minus...` 返回新对象，可链式调用 |

这套 API 主要分四块：

| 类别 | 类 | 表示什么 |
|------|-----|----------|
| 日期时间 | `LocalDate` | 年月日 |
| | `LocalTime` | 时分秒 |
| | `LocalDateTime` | 年月日时分秒（**不带时区**） |
| 时刻 | `Instant` | 时间戳（UTC 瞬间） |
| 时区 | `ZoneId`、`ZoneOffset`、`ZonedDateTime` | 时区与带时区的时间 |
| 间隔与格式化 | `Period`、`Duration`、`ChronoUnit`、`DateTimeFormatter` | 差值计算与格式化 |

## 12.2 `LocalDate` / `LocalTime` / `LocalDateTime`

三个最常用的类，**都不带时区信息**，用于表示"人看到的日期时间"。

| 类 | 含义 | 示例 |
|----|------|------|
| `LocalDate` | 年月日 | `2026-10-15` |
| `LocalTime` | 时分秒 | `22:30:05` |
| `LocalDateTime` | 年月日时分秒 | `2026-10-15T22:30:05` |

```java
LocalDate today = LocalDate.now();
System.out.println(today);                              // 2026-10-15

LocalDate birthday = LocalDate.of(2000, 1, 1);
System.out.println(birthday);                           // 2000-01-01
System.out.println(birthday.getDayOfWeek());            // SATURDAY
System.out.println(birthday.isLeapYear());              // true（2000 是闰年）

LocalDate nextWeek = today.plusDays(7);                 // 返回新对象
LocalDate changed = today.withYear(2030);
System.out.println(today);                              // 2026-10-15 ← 原对象没变
System.out.println(nextWeek);                           // 2026-10-22
System.out.println(changed);                            // 2030-10-15
```

公共方法分四类：

| 前缀 | 含义 | 示例 |
|------|------|------|
| `now` | 获取当前 | `LocalDateTime.now()` |
| `of` | 按数值创建 | `LocalDateTime.of(2026, 10, 15, 22, 30)` |
| `with...` | 修改某个字段，返回新对象 | `withYear(2030)` |
| `plus...` / `minus...` | 增加 / 减少，返回新对象 | `plusDays(7)`、`minusHours(2)` |
| `get...` | 获取字段 | `getYear()`、`getDayOfWeek()` |
| `is...` | 判断 | `isLeapYear()`、`isBefore(other)` |

> **注意**：`LocalDate` 这些类**没有时区**，所以它们**不能表示一个绝对的"时刻"**。`LocalDateTime.now()` 拿到的只是"当前时区下墙上时钟的读数"，换一个时区跑程序，同一句代码在同一物理时刻会得到不同结果。要表示"全世界的同一瞬间"，用 `Instant` 或 `ZonedDateTime`。

## 12.3 `Instant`：时间戳

`Instant` 表示**时间轴上的一个瞬间**，内部就是"距离 1970-01-01T00:00:00Z 的秒数 + 纳秒数"，是 Java 对时间戳的正式表达。

| 方法 | 说明 |
|------|------|
| `static Instant now()` | 当前时刻（UTC） |
| `static Instant ofEpochMilli(long milli)` | 按毫秒时间戳创建 |
| `static Instant ofEpochSecond(long second)` | 按秒时间戳创建 |
| `long toEpochMilli()` | 转成毫秒时间戳 |
| `long getEpochSecond()` | 转成秒时间戳 |
| `ZonedDateTime atZone(ZoneId zone)` | 指定时区，得到带时区的时间 |
| `Instant plus...` / `minus...` | 增减时间，返回新对象 |
| `boolean isBefore(Instant other)` / `isAfter` | 判断先后 |
| `static Instant of...(long epochMilli)` | 按秒/毫秒/纳秒创建的各种重载 |

```java
Instant now = Instant.now();
System.out.println(now);                            // 2026-10-15T14:30:05.123Z（UTC，带 Z 后缀）
System.out.println(now.toEpochMilli());             // 毫秒时间戳

Instant epoch = Instant.ofEpochMilli(0L);
System.out.println(epoch);                          // 1970-01-01T00:00:00Z

System.out.println(now.isAfter(epoch));             // true
```

> **注意**：`Instant` 打印出来永远带 `Z`（UTC），而且**无法直接格式化成年月日**——它没有"年月日"的概念，只有"离原点多久"。要显示成本地时间，必须先 `atZone(ZoneId.systemDefault())` 转成 `ZonedDateTime`。

## 12.4 `ZoneId` 与 `ZonedDateTime`

**`ZoneId` 表示时区**，比如 `Asia/Shanghai`、`UTC`、`America/New_York`。

| 方法 | 说明 |
|------|------|
| `static Set<String> getAvailableZoneIds()` | 获取 Java 支持的所有时区 ID |
| `static ZoneId systemDefault()` | 获取系统默认时区 |
| `static ZoneId of(String zoneId)` | 根据时区 ID 获取时区对象 |

```java
System.out.println(ZoneId.systemDefault());             // Asia/Shanghai
System.out.println(ZoneId.getAvailableZoneIds().size()); // 600 左右

ZoneId tokyo = ZoneId.of("Asia/Tokyo");
System.out.println(tokyo);                              // Asia/Tokyo
```

**`ZonedDateTime` 是"带时区的时间"**，它同时包含"时刻"和"时区"，是跨时区业务的标准表达。

| 方法 | 说明 |
|------|------|
| `static ZonedDateTime now()` | 当前时间（默认时区） |
| `static ZonedDateTime now(ZoneId zone)` | 指定时区的当前时间 |
| `static ZonedDateTime of(...)` | 按数值创建 |
| `ZonedDateTime with...( ... )` | 修改时间，返回新对象 |
| `ZonedDateTime plus...` / `minus...` | 增减时间 |
| `LocalDateTime toLocalDateTime()` | 去掉时区，转成本地日期时间 |
| `Instant toInstant()` | 转成时间戳 |

```java
ZonedDateTime shanghai = ZonedDateTime.now(ZoneId.of("Asia/Shanghai"));
ZonedDateTime tokyo = ZonedDateTime.now(ZoneId.of("Asia/Tokyo"));
System.out.println(shanghai);                    // 2026-10-15T22:30:05.123+08:00[Asia/Shanghai]
System.out.println(tokyo);                       // 2026-10-15T23:30:05.123+09:00[Asia/Tokyo]
System.out.println(shanghai.toInstant().equals(tokyo.toInstant()));   // true，同一时刻
```

上面这个例子最能说明问题：**同一物理时刻，在上海显示 22:30、在东京显示 23:30，但它们的 `Instant`（时间戳）完全相同**。这就是"时刻"与"本地时间"的区别。

## 12.5 `DateTimeFormatter`：格式化与解析

`DateTimeFormatter` 是 JDK8 的格式化器，**不可变、线程安全**，可以放心做静态常量。

| 方法 | 说明 |
|------|------|
| `static DateTimeFormatter ofPattern(String pattern)` | 按模式获取格式化器 |
| `String format(TemporalAccessor temporal)` | 格式化：时间对象 → 字符串 |
| `TemporalAccessor parse(CharSequence text)` | 解析为通用时间对象（一般用具体类的 `parse`） |

```java
DateTimeFormatter fmt = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
String text = fmt.format(LocalDateTime.now());
System.out.println(text);                                   // 2026-10-15 22:30:05

// 解析：每个时间类都有自己的静态 parse，可直接拿到具体类型
LocalDate date = LocalDate.parse("2026-10-15");             // 默认 ISO 格式 yyyy-MM-dd
LocalDate date2 = LocalDate.parse("2026/10/15",
        DateTimeFormatter.ofPattern("yyyy/MM/dd"));
System.out.println(date2);                                  // 2026-10-15

// 格式化 LocalDateTime、LocalDate、ZonedDateTime 都可以
System.out.println(LocalDate.now().format(DateTimeFormatter.ofPattern("yyyy年MM月dd日")));
// 2026年10月15日
```

> **最佳实践**：`DateTimeFormatter.ofPattern` 的模式字母和 `SimpleDateFormat` 基本一致（`yyyy`、`MM`、`dd`、`HH`、`mm`、`ss`），迁移成本很低。**新代码统一用它，把 `DateTimeFormatter` 声明成 `static final` 常量共享**——这是 `SimpleDateFormat` 时代需要 `ThreadLocal` 才能做到的事，现在一行就够了。

## 12.6 `Period`、`Duration` 与 `ChronoUnit`

三个类都用来算"两个时间之间差多久"，区别在于**算的是哪一类量**：

| 类 | 计算对象 | 单位 | 适用 |
|----|----------|------|------|
| `Period` | **日期**之间的间隔 | 年、月、日 | `LocalDate` 之间 |
| `Duration` | **时间**之间的间隔 | 时、分、秒、纳秒 | `LocalTime`、`Instant` 之间 |
| `ChronoUnit` | 通用单位差 | 任意（天、小时、分钟……） | 任何时间类型 |

```java
LocalDate d1 = LocalDate.of(2020, 1, 1);
LocalDate d2 = LocalDate.of(2026, 10, 15);

Period period = Period.between(d1, d2);
System.out.println(period);                                          // P6Y9M14D
System.out.println(period.getYears() + " 年 "
        + period.getMonths() + " 月 " + period.getDays() + " 天");     // 6 年 9 月 14 天

LocalTime t1 = LocalTime.of(9, 0);
LocalTime t2 = LocalTime.of(17, 30);
Duration duration = Duration.between(t1, t2);
System.out.println(duration.toMinutes());                             // 510
System.out.println(duration.toHours());                               // 8

// ChronoUnit 直接算"总共多少天/多少小时"，比 Period 更直观
long days = ChronoUnit.DAYS.between(d1, d2);
System.out.println(days);                                             // 2479
long hours = ChronoUnit.HOURS.between(t1, t2);
System.out.println(hours);                                            // 8
```

> **陷阱**：`Period.between` 返回的是"**6 年 9 个月 14 天**"这种**分量式**结果，**不是总天数**。想知道总共差多少天必须用 `ChronoUnit.DAYS.between(...)`。面试里"两个日期差几天"如果答了 `Period.getDays()` 就掉坑里了——它只返回"去掉整年整月之后剩下的天数"。

> **注意**：`Duration.between` 也能用在 `Instant` 上（比如测一段代码的耗时），`toMillis()`、`toNanos()` 都可取。但 `Period` 只能用于日期，`Duration` 不能用于 `LocalDate`——两者分工是设计好的。

## 12.7 与 `Date` 的互转

老代码里到处是 `java.util.Date`，新代码用 `java.time`，两者之间的桥梁是 **`Instant`**。

```java
// Date → Instant → LocalDateTime
Date date = new Date();
Instant instant = date.toInstant();
LocalDateTime ldt = LocalDateTime.ofInstant(instant, ZoneId.systemDefault());
System.out.println(ldt);                            // 2026-10-15T22:30:05.123

// LocalDateTime → Instant → Date
LocalDateTime now = LocalDateTime.now();
Instant back = now.atZone(ZoneId.systemDefault()).toInstant();
Date backDate = Date.from(back);
System.out.println(backDate);                       // Thu Oct 15 22:30:05 CST 2026

// 只关心时间戳时，最简写法
Date d = Date.from(Instant.now());
long millis = new Date().toInstant().toEpochMilli();
```

> **注意**：`LocalDateTime` 和 `Date` 互转时**必须显式指定时区**（一般用 `ZoneId.systemDefault()`），因为 `LocalDateTime` 没有时区信息，而 `Date` 内部是绝对时间戳。如果两边用了不同时区，转换结果会整体偏移几个小时——这类 bug 在跨时区部署时非常隐蔽。

## 12.8 新旧 API 对比

| 维度 | JDK7 及以前（`Date`/`Calendar`/`SimpleDateFormat`） | JDK8+（`java.time`） |
|------|--------------------------------------------------|---------------------|
| 可变性 | 可变，方法会改自身 | **不可变**，修改返回新对象 |
| 线程安全 | **不安全**（`SimpleDateFormat` 尤其） | **安全**，可直接做常量 |
| 日期与时间 | 混在一起，`Date` 既是时刻又是本地时间 | **职责分明的多个类** |
| 月份起始 | `Calendar.MONTH` 从 `0` 开始 | 从 `1` 开始，符合直觉 |
| 时区处理 | 含糊，依赖默认时区，方法上常有 `TimeZone` 参数 | `ZoneId` 独立建模，显式传参 |
| 格式化器 | `SimpleDateFormat`（非线程安全） | `DateTimeFormatter`（线程安全） |
| 差值计算 | 手动算毫秒再除，易错 | `Period` / `Duration` / `ChronoUnit` |

> **行业实践**：**新项目全部用 `java.time`**。老项目升级时用"新代码新 API、老接口边界处互转"的策略即可，不必一次性重构。数据库映射方面，JDBC 4.2 起原生支持 `LocalDate`/`LocalDateTime`，MyBatis、Hibernate 等框架也都已支持，可以直接存取。

# 13. 小结

| 主题 | 一句话要点 |
|------|-----------|
| `Math` | `final` + 全静态；`round(-1.5)` 是 `-1`，`abs(Integer.MIN_VALUE)` 仍是负数 |
| `System` | `exit` 会杀掉整个 JVM；测耗时用 `nanoTime` 不用 `currentTimeMillis`；`arraycopy` 是 native 浅拷贝，基本类型要求同型 |
| `Runtime` | 私有构造 + `getRuntime()` 的饿汉式单例；`maxMemory`/`totalMemory`/`freeMemory` 是三件事 |
| `Object` | 11 个方法；**重写 `equals` 必须重写 `hashCode`**，否则对象存进 `HashMap`/`HashSet` 就取不出来 |
| `Objects` | `equals` 空安全比较、`requireNonNull` 参数校验、`hash` 生成哈希；与 `Object` 无继承关系 |
| `BigInteger`/`BigDecimal` | 都是不可变的，运算返回新对象；**`BigDecimal` 必须用 `String` 构造**，`divide` 必须给精度和舍入模式；`equals` 比数值 + 精度，`compareTo` 只比数值 |
| 正则基础 | 字符类、预定义字符、数量词、`(?i)` 等特殊符号；`.` 默认不匹配换行，`\w` 不含中文 |
| 贪婪与回溯 | 默认贪婪，`X+?` 是勉强，`X++` 是独占（不回溯）；复杂正则要警惕 ReDoS |
| `Pattern`/`Matcher` | 预编译成 `static final Pattern` 复用，线程安全；`matches` 要求整串，`find` 只要求包含 |
| 分组 | 数左括号定组号（`0` 号是整体）；正则内用 `\n`、替换串里用 `$n`；`(?:)` 不占组号 |
| 老时间 API | `Calendar` 月份从 `0` 开始；`SimpleDateFormat` **线程不安全** |
| 新时间 API | 不可变、线程安全；`LocalDate`/`LocalTime`/`LocalDateTime` 无时区，`Instant` 是时间戳，`ZonedDateTime` 带时区；`Period` 算日期、`Duration` 算时间 |

下一篇（第 12 篇）进入算法与 Lambda 表达式：你会看到本篇幅讲到的 `Math`、`BigDecimal` 如何参与常见算法题，也会用上 `Objects`、比较器这些工具类；而第 13 篇的 Stream 与函数式接口，则会把这套"现成工具 + 声明式组合"的风格推到极致。
