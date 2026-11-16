`java.lang` 是唯一一个**不需要 `import` 就能直接使用**的包——编译器自动导入它，JVM 启动时也要靠它。它是 JDK 里最底层的一层：`Object` 定义了所有类的公共契约，`Class` 是反射的入口，`String` 和八个包装类几乎出现在每一行业务代码里。本篇按八个语义分组把它整理成一份**可以随时回来查**的速查手册：每组一张表，列出类、作用、常用方法、典型用途和对应章节，并点出这个包里"最容易被忽略但最有用"的东西。

# 1. 基础类

这一组的类几乎从不直接出现在业务代码里，却是反射、泛型、注解、序列化的共同底座。`Object` 的方法契约（`equals`/`hashCode`/`toString`）决定了一个类能否安全地放进 `HashMap`；`Class` 是所有框架的入口；`Void` 和 `Package` 则是两个"平时想不起来、需要时救急"的小类。

| 类 | 一句话作用 | 最常用的方法 | 典型用途 | 详见 |
|----|-----------|-------------|----------|------|
| `Object` | **所有类的基类** | `equals()`、`hashCode()`、`toString()`、`getClass()`、`clone()`、`wait()`/`notify()`/`notifyAll()` | 重写 `equals`/`hashCode` 契约；用 `Object` 接收任意类型 | 第 7 篇 |
| `Class<T>` | 表示类的**运行时信息**，是反射的入口 | `getName()`、`getSimpleName()`、`forName()`、`getDeclaredFields()`、`getDeclaredMethods()`、`getConstructor()`、`isAssignableFrom()`、`getClassLoader()` | 反射、动态代理、框架的组件扫描 | 第 19 篇 |
| `Enum<E>` | 所有枚举类型的**隐式父类** | `name()`、`ordinal()`、`compareTo()`、`getDeclaringClass()`、`valueOf()` | 枚举公共行为的来源 | 第 20 篇 |
| `Void` | 代表 `void` 类型的**占位类** | 只有一个常量 `Void.TYPE` | 泛型里表示"无返回值"，如 `Callable<Void>`、`CompletableFuture<Void>` | —— |
| `Package` | 表示一个包的**运行时元信息** | `getName()`、`getImplementationVersion()`、`getImplementationTitle()`、`isSealed()`、`getSpecificationVersion()` | 打印启动信息、确认到底加载了哪个版本的 jar | —— |
| `System` | 系统级操作类（IO、GC 等） | 方法很多，见第 5 节 | 时间戳、数组拷贝、系统属性、退出 JVM | 第 5 节 |
| `Runtime` | JVM 运行时环境控制 | 见第 5 节 | 获取 CPU 核数、堆内存、注册关闭钩子 | 第 5 节 |

`Object` 一共只有 11 个方法，但其中的契约是 Java 里最重要的一条：

| 方法 | 能否重写 | 说明 |
|------|---------|------|
| `equals(Object)` | 可重写 | 默认实现就是 `==`（比较地址）；**重写它必须同时重写 `hashCode()`** |
| `hashCode()` | 可重写 | 默认返回由对象地址派生的值；相等的对象必须有相同的哈希值 |
| `toString()` | 可重写 | 默认输出 `类名@十六进制哈希`，调试时几乎总要重写 |
| `getClass()` | **不能**（`final`） | 返回运行时类型，是反射的起点 |
| `clone()` | 可重写 | 受保护的**浅拷贝**，必须实现 `Cloneable`，否则抛 `CloneNotSupportedException` |
| `wait()` / `wait(long)` / `wait(long,int)` | **不能**（`final`） | 释放锁并等待，只能在 `synchronized` 内调用 |
| `notify()` / `notifyAll()` | **不能**（`final`） | 唤醒在该锁上等待的线程 |
| `finalize()` | 可重写 | **已废弃**（JDK 9 起 `@Deprecated`，JDK 18 起标记为 `forRemoval`），不要再使用 |

> **注意**：`equals` 和 `hashCode` 是一对**必须同进同退**的方法。只重写 `equals` 会让对象在 `HashMap`/`HashSet` 里"存进去就找不回来"——因为两个相等的对象哈希值不同，被放进了不同的桶。IDE 和 `record` 都会自动生成成对的实现，手写时务必一起改。

# 2. 包装类

八个包装类把基本类型"对象化"，这才让 `List<Integer>` 这样的泛型和集合成为可能。这里最容易被忽略的有两件事：一是 `Integer` 的 **`-128 ~ 127` 缓存**会让 `==` 的结果变得不可预期；二是 `Integer`/`Long` 里藏着一整套**位运算工具方法**，做位图、权限掩码时比自己写循环强得多。

| 包装类 | 对应基本类型 | 最常用的方法 | 典型用途 |
|--------|-------------|-------------|----------|
| `Boolean` | `boolean` | `valueOf()`、`parseBoolean()`、`booleanValue()`、常量 `TRUE`/`FALSE` | 布尔值解析与传递 |
| `Byte` | `byte` | `valueOf()`、`parseByte()`、`byteValue()`、`toUnsignedInt()`、`compareUnsigned()` | 字节与无符号数转换 |
| `Character` | `char` | `isDigit()`、`isLetter()`、`isWhitespace()`、`isUpperCase()`、`toUpperCase()`、`getNumericValue()`、`compare()` | 字符判断、大小写转换 |
| `Short` | `short` | `valueOf()`、`parseShort()`、`shortValue()`、`reverseBytes()` | 短整型处理 |
| `Integer` | `int` | `valueOf()`、`parseInt()`、`intValue()`、`compare()`、`toBinaryString()`、`toHexString()`、`bitCount()`、`numberOfLeadingZeros()`、`compareUnsigned()`、`MAX_VALUE`/`MIN_VALUE` | **最常用的包装类** |
| `Long` | `long` | `valueOf()`、`parseLong()`、`longValue()`、`toUnsignedString()`、`compareUnsigned()`、`MAX_VALUE`/`MIN_VALUE` | 时间戳、分布式 ID |
| `Float` | `float` | `valueOf()`、`parseFloat()`、`isNaN()`、`isInfinite()`、`compare()`、`MAX_VALUE`/`MIN_VALUE` | 单精度数值 |
| `Double` | `double` | `valueOf()`、`parseDouble()`、`isNaN()`、`isInfinite()`、`compare()`、`MIN_NORMAL`、`MAX_VALUE` | 双精度数值 |

```java
public class WrapperDemo {
    public static void main(String[] args) {
        // 自动装箱 / 拆箱：编译器自动插入 valueOf() 和 xxxValue()
        Integer a = 100;             // 装箱：Integer.valueOf(100)
        int     b = a;               // 拆箱：a.intValue()

        // 陷阱一：Integer 缓存 -128 ~ 127，超出这个范围就是两个不同对象
        Integer x = 127, y = 127;
        Integer m = 128, n = 128;
        System.out.println(x == y);            // true（命中缓存，同一个对象）
        System.out.println(m == n);            // false（超出缓存，各自 new 了一个）
        System.out.println(m.equals(n));       // true（比较数值只能靠 equals）

        // 陷阱二：包装类型参与运算会先拆箱，null 会直接抛 NullPointerException
        Integer nullable = null;
        // int sum = nullable + 1;             // NullPointerException

        // 常用静态方法
        System.out.println(Integer.parseInt("42"));         // 42
        System.out.println(Integer.toBinaryString(170));    // 10101010
        System.out.println(Integer.toHexString(170));       // aa
        System.out.println(Integer.bitCount(170));          // 4（二进制里 1 的个数）
        System.out.println(Integer.MAX_VALUE);              // 2147483647
        System.out.println(Double.isNaN(0.0 / 0.0));        // true
        System.out.println(Integer.compareUnsigned(-1, 1)); // 1（把 -1 当成无符号数则更大）
    }
}
```

> **最佳实践**：涉及数值比较时**永远用 `equals()` 或 `compareTo()`，不要用 `==`**。`==` 在 `-128 ~ 127` 内"看起来能用"，一旦数值变大就突然失效，这类 bug 极难定位。另外，把包装类型用在小循环里会产生大量装箱对象，性能敏感的场景优先用基本类型。

# 3. 字符串处理类

三个类的分工很清晰：`String` 不可变、`StringBuilder` 可变但不安全、`StringBuffer` 可变且线程安全。最容易被忽略的是**编译器已经做了优化**——单次拼接 `"a" + "b" + "c"` 在编译期就折叠成 `"abc"`，运行期零开销；只有在循环或条件分支里反复拼接时，`StringBuilder` 才有意义。

| 类 | 一句话作用 | 最常用的方法 | 典型用途 |
|----|-----------|-------------|----------|
| `String` | **不可变**字符串 | `length()`、`charAt()`、`substring()`、`indexOf()`、`split()`、`join()`、`replace()`、`strip()`、`equals()`、`equalsIgnoreCase()`、`format()`、`valueOf()`、`toCharArray()`、`isBlank()` | 几乎所有文本处理；不可变性让它能安全共享、能当 `Map` 的 key |
| `StringBuilder` | 可变字符串，**非线程安全** | `append()`、`insert()`、`deleteCharAt()`、`replace()`、`reverse()`、`setLength()`、`toString()` | 循环里拼字符串，单线程首选 |
| `StringBuffer` | 可变字符串，**线程安全**（方法带 `synchronized`） | 与 `StringBuilder` 完全一致 | 多线程共享同一个缓冲区（实际极罕见） |

```java
public class StringDemo {
    public static void main(String[] args) {
        String s = "hello";

        // String 不可变：所有"修改"都返回新对象，原对象不受影响
        String upper = s.toUpperCase();
        System.out.println(s);                        // hello
        System.out.println(upper);                    // HELLO

        // StringBuilder：循环拼接才是它的战场
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < 3; i++) {
            sb.append(i).append('-');                 // 链式调用，返回 this
        }
        System.out.println(sb.toString());             // 0-1-2-
        System.out.println(sb.reverse());              // -2-1-0

        // 但单次拼接不需要 StringBuilder，编译器已经折叠好了
        String full = "a" + "b" + "c";                // 编译期直接变成 "abc"，运行期零开销

        // 几个容易用错的方法
        System.out.println("a,b,,c".split(",").length);   // 4：末尾的空串也会算进去
        System.out.println("  x  ".strip());              // x：JDK 11+，按 Unicode 空白判断
        System.out.println("   ".isBlank());              // true：JDK 11+，全是空白也算 blank
        System.out.println("  x  ".strip().isEmpty());    // false
    }
}
```

> **陷阱**：`"   ".isEmpty()` 返回 `false`，`"   ".isBlank()` 才是 `true`——校验"用户是不是没填"要用 `isBlank()`。另一个高频坑是**字符串比较**：`==` 比的是地址（常量池让短字符串有时恰好相等），判断内容一律用 `equals()`。

# 4. 线程和并发

`java.lang` 里的并发工具其实很少：`Thread`、`Runnable`、`ThreadGroup`、`ThreadLocal` 加上两个异常类，真正的武器库在 `java.util.concurrent`。这一节里最容易被忽略、也最容易出事的是 **`ThreadLocal` 的清理**：它在线程池场景下不会自动回收，用完不 `remove()` 就会造成内存泄漏和请求间的数据串台。

| 类 / 接口 | 一句话作用 | 最常用的方法 | 典型用途 |
|-----------|-----------|-------------|----------|
| `Thread` | 线程类 | `start()`、`run()`、`join()`、`sleep()`、`interrupt()`、`isInterrupted()`、`setDaemon()`、`setName()`/`getName()`、`currentThread()`、`getState()`、`setPriority()` | 创建与控制线程 |
| `Runnable` | 线程任务接口（函数式接口） | `run()` | 把任务与线程解耦，提交给线程池或 `Thread` |
| `ThreadGroup` | 一组线程的管理容器 | `activeCount()`、`enumerate()`、`interrupt()`、`getName()`、`getParent()` | 批量管理线程；**已过时**，现代做法用线程池 |
| `ThreadLocal<T>` | **线程局部变量**，每个线程一份独立副本 | `get()`、`set()`、`remove()`、`withInitial()` | 请求上下文传递（用户身份、链路 ID）、复用非线程安全对象 |
| `InheritableThreadLocal<T>` | 可被**子线程继承**的 `ThreadLocal` | 与 `ThreadLocal` 一致 | 把上下文从父线程传给新建的子线程 |
| `InterruptedException` | 线程在阻塞时被中断抛出的**受检异常** | —— | `sleep`/`wait`/`join` 被中断时抛出 |
| `IllegalThreadStateException` | 线程状态不合法时抛出 | —— | 例如对已启动的线程再调 `start()`，属于编码错误 |

```java
public class ThreadLocalDemo {
    // withInitial 提供初始值；每个线程拿到的都是自己的副本
    private static final ThreadLocal<String> CONTEXT = ThreadLocal.withInitial(() -> "默认值");

    public static void main(String[] args) throws InterruptedException {
        CONTEXT.set("主线程的值");
        System.out.println(CONTEXT.get());            // 主线程的值

        Thread t = new Thread(() -> {
            System.out.println(CONTEXT.get());        // 默认值：子线程拿到的是自己的副本
            CONTEXT.set("子线程的值");
            System.out.println(CONTEXT.get());        // 子线程的值
            CONTEXT.remove();                         // 关键：用完必须清理，否则线程池里会残留
        });
        t.start();
        t.join();

        System.out.println(CONTEXT.get());            // 主线程的值：没有被子线程污染
        CONTEXT.remove();                             // 主线程同样要清理
    }
}
```

> **最佳实践**：`ThreadLocal` 一律遵守"**谁 set 谁 remove**"，并且把 `remove()` 放在 `finally` 里。在线程池里，一个线程会被成百上千个任务复用，上一步遗留的值会被下一步任务读到——这不仅是内存泄漏，更是**数据串台**的安全隐患。

# 5. 系统与运行控制

这一节是整个 `java.lang` 里"实用性最高、最容易被当成魔法"的部分。`System` 的六个方法几乎每个项目都在用；`Runtime` 和 `ProcessBuilder` 是 Java 与外部程序打交道的入口。此处最容易被忽略的是 **`System.arraycopy()`**——它被 JVM 内联成内存块拷贝，比手写循环快一个数量级，`ArrayList` 的扩容、`Arrays.copyOf` 底层走的都是它。

**`System` 的完整方法表**：

| 方法 | 说明 |
|------|------|
| `static long currentTimeMillis()` | 当前时间的毫秒时间戳（自 1970-01-01 UTC 起）；适合生成时间戳，**不适合测耗时**（受系统时钟调整影响） |
| `static long nanoTime()` | 纳秒级**单调**时钟，只能做差值；**测耗时、写基准测试必须用它** |
| `static void arraycopy(Object src, int srcPos, Object dest, int destPos, int length)` | 数组拷贝，JVM 内联优化；支持自己拷给自己（数组内平移） |
| `static String getProperty(String key)` | 读系统属性：`java.version`、`java.home`、`os.name`、`user.dir`、`file.separator`、`line.separator` |
| `static String getProperty(String key, String def)` | 带默认值的版本 |
| `static String setProperty(String key, String value)` | 设置系统属性（启动参数写 `-Dkey=value` 效果相同） |
| `static String getenv(String name)` | 读**环境变量**（来自操作系统，与系统属性不是一回事） |
| `static Map<String,String> getenv()` | 读全部环境变量 |
| `static void exit(int status)` | 终止 JVM，`0` 表示正常退出；当前线程的 `finally` **不会**执行，但注册的 shutdown hook 会执行 |
| `static void gc()` | 建议 JVM 回收垃圾，**只是建议，不保证执行**；生产代码不要调用 |
| `static void runFinalization()` | 建议运行待终结对象的方法（已废弃） |
| `static void setIn(InputStream)` / `setOut(PrintStream)` / `setErr(PrintStream)` | 重定向标准流，测试时常用它捕获 `System.out` |
| `static Console console()` | 获取控制台对象，用于读取密码（不回显）；在 IDE 里可能返回 `null` |
| `static int identityHashCode(Object x)` | 返回对象的"身份哈希"，即使 `hashCode()` 被重写也返回默认实现的值 |

**`Runtime` 的常用方法**：

| 方法 | 说明 |
|------|------|
| `static Runtime getRuntime()` | 获取唯一的 `Runtime` 实例（饿汉单例） |
| `int availableProcessors()` | 可用 CPU 逻辑核数，用来估算线程池大小 |
| `long totalMemory()` / `freeMemory()` / `maxMemory()` | 堆的总量 / 空闲量 / 上限，用于内存诊断 |
| `Process exec(String command)` | 执行外部命令（简易版，已被 `ProcessBuilder` 取代） |
| `void addShutdownHook(Thread hook)` | 注册 JVM 关闭钩子，用于优雅停机（Spring 的优雅关闭就依赖它） |
| `void halt(int status)` | **强制**退出，**连 shutdown hook 都不执行**，慎用 |
| `void gc()` / `void runFinalization()` | 与 `System` 的两个"建议"方法等价 |

**进程相关**：

| 类 | 一句话作用 | 常用方法 | 典型用途 |
|----|-----------|----------|----------|
| `Process` | 表示一个**外部进程** | `getInputStream()`（读子进程的输出）、`getErrorStream()`、`getOutputStream()`（给子进程写输入）、`waitFor()`、`exitValue()`、`isAlive()`、`destroy()`、`pid()` | 读取外部命令的输出、控制子进程 |
| `ProcessBuilder` | 创建和配置外部进程的**推荐方式** | `new ProcessBuilder(String...)`、`directory(File)`、`redirectErrorStream(true)`、`inheritIO()`、`redirectOutput()`、`start()` | 调用 git、ffmpeg、python 等外部程序 |

```java
import java.io.IOException;

public class SystemDemo {
    public static void main(String[] args) throws IOException, InterruptedException {
        // 1. 测耗时的正确姿势：nanoTime 做差，而不是 currentTimeMillis
        long start = System.nanoTime();
        long sum = 0;
        for (int i = 0; i < 1_000_000; i++) sum += i;
        System.out.println("耗时 " + (System.nanoTime() - start) + " ns，结果 " + sum);
        // 输出形如：耗时 1234500 ns，结果 499999500000

        // 2. arraycopy：JVM 内联的内存拷贝，Arrays.copyOf / ArrayList 扩容都用它
        int[] src  = {1, 2, 3, 4, 5};
        int[] dest = new int[5];
        System.arraycopy(src, 1, dest, 0, 3);         // 从 src[1] 开始拷 3 个到 dest[0]
        System.out.println(java.util.Arrays.toString(dest));   // [2, 3, 4, 0, 0]

        // 3. 系统属性与环境变量
        System.out.println(System.getProperty("java.version"));      // 17.0.9（视具体 JDK 而定）
        System.out.println(System.getProperty("os.name"));           // Windows 11
        System.out.println(System.getProperty("user.dir"));          // 当前工作目录
        System.out.println(System.getProperty("no.such.key", "默认值"));   // 默认值
        System.out.println(System.getenv("PATH") != null);           // true

        // 4. ProcessBuilder：调用外部程序并读取它的输出
        ProcessBuilder pb = new ProcessBuilder("java", "-version");
        pb.redirectErrorStream(true);                 // java -version 走的是 stderr，必须合并，否则读不到
        Process process = pb.start();
        try (var reader = new java.io.BufferedReader(
                new java.io.InputStreamReader(process.getInputStream()))) {
            reader.lines().forEach(System.out::println);
        }
        int exitCode = process.waitFor();             // 等待子进程结束
        System.out.println("退出码：" + exitCode);       // 退出码：0

        // 5. 关闭钩子：JVM 退出时执行，用于优雅停机（释放连接、刷盘、注销服务）
        Runtime.getRuntime().addShutdownHook(new Thread(() ->
                System.out.println("JVM 即将退出，做最后的清理")));

        // 6. 看 CPU 核数与堆内存
        System.out.println(Runtime.getRuntime().availableProcessors());   // 例如 16
    }
}
```

> **注意**：`getProperty()` 读的是**系统属性**（JVM 层面的键值对，可用 `-Dkey=value` 注入），`getenv()` 读的是**环境变量**（操作系统层面的，`PATH`、`JAVA_HOME` 等）。两者来源完全不同，排查配置不生效时先分清用的是哪一个。另外 `System.exit()` 会执行 shutdown hook 但不执行当前线程的 `finally`，而 `Runtime.halt()` 连 hook 都不执行——**优雅停机用 `exit()`，绝不要用 `halt()`**。

# 6. 错误与异常类

`Throwable` 是所有可抛出对象的唯一根，往下分成两支：`Error` 表示 JVM 层面的严重问题（通常不该捕获），`Exception` 表示程序可以处理的问题。这一节最容易被忽略的是**受检与非受检的分界线**——`RuntimeException` 及其子类是**非受检**异常，编译器不强制处理；其余 `Exception` 都是**受检**异常，必须 `try-catch` 或 `throws` 声明。

| 类 | 继承自 | 是否受检 | 说明 |
|----|--------|----------|------|
| `Throwable` | `Object` | —— | 所有可抛出对象的根。常用方法：`getMessage()`、`printStackTrace()`、`getCause()`、`getStackTrace()`、`addSuppressed()` |
| `Error` | `Throwable` | 非受检 | 严重错误，不建议捕获 |
| `Exception` | `Throwable` | —— | 所有可捕获异常的父类 |
| `RuntimeException` | `Exception` | **非受检** | 运行时异常父类（可不强制捕获） |

**`Error` 家族**——这些通常代表"环境坏了"，捕获后往往也无能为力：

| 错误 | 触发场景 |
|------|----------|
| `OutOfMemoryError` | 堆内存耗尽；分为 `Java heap space`、`Metaspace`、`GC overhead limit exceeded` 等多种 |
| `StackOverflowError` | 栈溢出，通常是递归太深或无限递归 |
| `NoClassDefFoundError` | 编译期存在、运行期找不到该类（依赖缺失或静态初始化失败） |
| `NoSuchMethodError` | 运行期找不到方法——**依赖版本冲突的经典症状** |
| `ExceptionInInitializerError` | 静态初始化块里抛了异常 |
| `AssertionError` | `assert` 断言失败时抛出 |

**`RuntimeException` 家族**——这些才是日常真正要处理的：

| 异常 | 触发场景 |
|------|----------|
| `NullPointerException` | 对 `null` 调用方法或访问字段；JDK 14+ 的 helpful NPE 会明确指出是哪个变量为 `null` |
| `IndexOutOfBoundsException` | 下标越界（父类）；`ArrayList.get()` 越界抛的就是它 |
| `ArrayIndexOutOfBoundsException` | 数组下标越界，如长度为 3 的数组访问 `arr[10]` |
| `StringIndexOutOfBoundsException` | 字符串下标越界，如 `"abc".charAt(5)` |
| `IllegalArgumentException` | 参数不合法；自己写参数校验时抛这个 |
| `NumberFormatException` | 字符串转数字失败（`IllegalArgumentException` 的子类） |
| `IllegalStateException` | 对象状态不满足调用前提，如重复消费同一个 `Stream` |
| `ArithmeticException` | 算术错误，如整数 `1 / 0`（浮点除零不会抛异常，得到 `Infinity`） |
| `ClassCastException` | 强制类型转换失败，如把 `String` 强转成 `Integer` |
| `UnsupportedOperationException` | 不支持该操作，如对 `List.of()` 的结果调 `add()` |
| `ConcurrentModificationException` | 迭代过程中修改集合（fail-fast 机制） |
| `NegativeArraySizeException` | 数组长度为负数 |

**常用的受检异常**：`InterruptedException`、`IOException`、`FileNotFoundException`、`ClassNotFoundException`、`CloneNotSupportedException`、`TimeoutException`、`ParseException`、`SQLException`（在 `java.sql`）。

```java
public class ExceptionFamilyDemo {
    public static void main(String[] args) {
        // 非受检异常：编译器不强制处理，但运行期照样会抛
        try { int x = 1 / 0; }
        catch (ArithmeticException e) { System.out.println("/ by zero"); }

        try { Object o = "s"; Integer i = (Integer) o; }
        catch (ClassCastException e) { System.out.println("类型转换失败"); }

        try { Integer.parseInt("abc"); }
        catch (NumberFormatException e) {
            System.out.println("数字格式错误：" + e.getMessage());   // 数字格式错误：For input string: "abc"
        }

        try { java.util.List.of(1, 2).add(3); }
        catch (UnsupportedOperationException e) { System.out.println("不可变集合不支持 add"); }

        try {
            java.util.List<String> list =
                    new java.util.ArrayList<>(java.util.List.of("a", "b", "c"));
            for (String s : list) list.remove(s);      // 迭代中直接修改，触发 fail-fast
        } catch (java.util.ConcurrentModificationException e) {
            System.out.println("迭代中不能直接修改集合，应改用 iterator.remove()");
        }

        // 受检异常：必须 try-catch 或 throws 声明，否则编译不过
        try { Class.forName("com.example.NotExist"); }
        catch (ClassNotFoundException e) { System.out.println("类找不到：" + e.getMessage()); }
    }
}
```

> **陷阱**：`ConcurrentModificationException` 的触发条件比想象中微妙。对 `["a", "b"]` 这样的**两个元素**的列表执行 `for (String s : list) list.remove(s);`，居然**不会抛异常**——删掉第一个元素后 `ArrayList` 迭代器的 `cursor` 恰好等于新的 `size`，`hasNext()` 直接返回 `false`，循环"正常"结束了，剩下的 `[b]` 被静默留下。元素变成三个才会抛。所以要删元素，正确做法始终是显式用 `iterator.remove()`，或者用 `removeIf()`。

> **最佳实践**：什么时候自定义**受检**异常、什么时候用**非受检**？判断标准是"**调用方能不能合理地从错误中恢复**"：能恢复（文件不存在可以换个路径重试）就用受检异常，强制调用方处理；属于**编程错误**（参数是 `null`、状态不对）就用 `IllegalArgumentException` / `IllegalStateException` 这类非受检异常，让它快速失败、在测试期暴露出来。

# 7. 数学和实用类

`Math` 提供静态数学函数，`StrictMath` 是它的"逐位可复现"版本。这里最容易出错的不是不会用，而是**几个取整方法的名字太像**：`round` 是四舍五入且返回 `long`，`rint` 是银行家舍入且返回 `double`，`ceil` 向上、`floor` 向下。另外 `Math.abs(Integer.MIN_VALUE)` 的结果仍然是负数，这是补码表示的一个经典陷阱。

| 类 | 一句话作用 | 最常用的方法 | 典型用途 |
|----|-----------|-------------|----------|
| `Math` | 常用数学函数类（如 `sin`、`cos`、`pow`、`sqrt`） | `abs()`、`max()`、`min()`、`pow()`、`sqrt()`、`cbrt()`、`ceil()`、`floor()`、`round()`、`rint()`、`random()`、`toRadians()`、`hypot()`、`floorDiv()`、`floorMod()`、`addExact()`、`multiplyExact()`；常量 `PI`、`E` | 通用数值计算 |
| `StrictMath` | **精度更严格**的 `Math` 替代品 | 方法名与 `Math` 完全一致 | 需要**跨平台逐位可复现**的结果（`Math` 在某些平台允许用更快的近似实现） |
| `Number` | 所有数值包装类的父类（**抽象类**） | `intValue()`、`longValue()`、`floatValue()`、`doubleValue()`、`byteValue()`、`shortValue()` | 写一个"能吃下任意数值类型"的方法 |

需要**任意精度**时，要用的是 `java.math` 包的 `BigDecimal`（精确小数、金额计算）和 `BigInteger`（超大整数），它们不在 `java.lang` 里，但它们的共同父类 `Number` 在这里。

几个极易混淆的取整方法：

| 表达式 | 结果 | 说明 |
|--------|------|------|
| `Math.round(2.5)` | `3` | 四舍五入到最近整数，**返回 `long`**；对负数相当于 `floor(x + 0.5)`，所以 `Math.round(-2.5)` 是 `-2` |
| `Math.rint(2.5)` | `2.0` | 返回 `double`，**银行家舍入**（`.5` 向偶数取整），`rint(3.5)` 得 `4.0` |
| `Math.ceil(2.1)` | `3.0` | 向上取整（向正无穷） |
| `Math.floor(2.9)` | `2.0` | 向下取整（向负无穷） |
| `Math.abs(Integer.MIN_VALUE)` | `-2147483648` | **仍然为负**！补码没有对应的正数，这是经典陷阱 |
| `Math.floorMod(-1, 3)` | `2` | 结果符号**跟随除数**；`-1 % 3` 得 `-1`，做环形下标时应该用 `floorMod` |
| `Math.addExact(a, b)` | 溢出时抛 `ArithmeticException` | 替代会静默环绕的 `a + b` |

```java
public class MathDemo {
    public static void main(String[] args) {
        System.out.println(Math.abs(-3));                 // 3
        System.out.println(Math.max(3, 5));               // 5
        System.out.println(Math.pow(2, 10));              // 1024.0
        System.out.println(Math.sqrt(16));                // 4.0
        System.out.println(Math.ceil(2.1));               // 3.0：向上取整
        System.out.println(Math.floor(2.9));              // 2.0：向下取整
        System.out.println(Math.round(2.5));              // 3：四舍五入，返回 long
        System.out.println(Math.rint(2.5));               // 2.0：银行家舍入，.5 取偶
        System.out.println(Math.floorMod(-1, 3));         // 2：结果符号跟随除数
        System.out.println(Math.abs(Integer.MIN_VALUE));  // -2147483648：溢出，仍为负！

        // addExact 把静默溢出变成异常
        try {
            Math.addExact(Integer.MAX_VALUE, 1);
        } catch (ArithmeticException e) {
            System.out.println("加法溢出：" + e.getMessage());   // 加法溢出：integer overflow
        }

        // Number：所有数值包装类的共同父类，写通用方法时非常有用
        System.out.println(toDouble(3));        // 3.0
        System.out.println(toDouble(3.14f));    // 3.140000104904175（float 转 double 的精度痕迹）
        System.out.println(toDouble(100L));     // 100.0
    }

    // 一个方法吃下所有数值类型：Integer、Long、Float、Double、BigDecimal 都能传
    static double toDouble(Number n) {
        return n.doubleValue();
    }
}
```

> **注意**：`Math.random()` 返回 `[0.0, 1.0)` 的 `double`，**不适合用于安全场景**（如生成 token、验证码），它基于可预测的线性同余算法。安全用途必须用 `java.security.SecureRandom`。另外金额计算不要用 `double`，误差会在累加中被放大，一律用 `BigDecimal`。

# 8. 其他接口类

这一节是 `java.lang` 的接口清单，它们定义了 Java 里最多人实现、也最容易被写错的几套"公共协议"：`Comparable` 决定自然顺序、`Comparator` 决定外部顺序、`Iterable` 让 `for-each` 可用、`AutoCloseable` 让 try-with-resources 可用、`CharSequence` 是字符串家族的共同抽象。最容易被忽略的是 **`Runnable` 与 `Callable` 的区别只有"能不能返回值"**——线程池的 `submit()` 两个都收，但只有 `Callable` 能拿到结果。

| 接口 | 一句话作用 | 核心方法 | 典型用途 | 详见 |
|------|-----------|----------|----------|------|
| `Comparable<T>` | 排序比较接口（如 `String`、`Integer` 实现它），定义**自然顺序** | `compareTo(T o)` | 让对象自身可排序；`Collections.sort()`、`TreeSet`、`Arrays.sort()` | 第 9 篇 |
| `Comparator<T>` | **外部比较器**（实际在 `java.util`），定义任意顺序 | `compare(T a, T b)`、`comparing()`、`thenComparing()`、`reversed()`、`nullsFirst()` | 同一个类需要多种排序方式；不能改源码的类 | 第 9 篇 |
| `CharSequence` | **字符序列**接口（`String`、`StringBuilder` 实现它） | `length()`、`charAt()`、`subSequence()`、`chars()`、`isEmpty()` | 参数写成它就能同时接受 `String`/`StringBuilder`/`StringBuffer` | —— |
| `AutoCloseable` | 可**自动关闭**资源的接口（用于 try-with-resources） | `close()` | 任何"用完必须释放"的资源：流、连接、锁 | 第 14 篇 |
| `Cloneable` | 支持对象克隆的**标记接口** | 无方法（纯标记） | 配合 `Object.clone()`；不实现就抛 `CloneNotSupportedException` | —— |
| `Iterable<T>` | **可迭代**对象（`for-each` 循环的前提） | `iterator()`、`forEach()`、`spliterator()` | 让自己写的容器支持增强 `for` | 第 9 篇 |
| `Runnable` | 表示一个**无返回值、不能抛受检异常**的任务 | `run()` | 线程任务、线程池任务 | 第 17 篇 |
| `Appendable` | 可**追加字符序列**（如 `StringBuilder`） | `append(CharSequence)`、`append(char)` | 让方法接受任意"可追加"的目标 | —— |
| `Readable` | 可**读取字符序列**的接口 | `read(CharBuffer)` | 与 `Appendable` 方向相反，用于 `Scanner` 这类输入源 | —— |

`Comparable` 与 `Comparator` 的选择：

| 维度 | `Comparable` | `Comparator` |
|------|--------------|--------------|
| 定义位置 | 类**自身**实现 | 外部独立对象 |
| 排序方式数量 | 只能有一种"自然顺序" | 可以有任意多种 |
| 是否需要改源码 | 需要 | 不需要 |
| 常用写法 | `class Person implements Comparable<Person>` | `Comparator.comparing(Person::getAge)` |
| 何时选 | 类有一个**天然**的排序依据（`String` 按字典序） | 需要多种排法，或类不可修改 |

```java
// Comparable：类自己定义"自然顺序"
class Person implements Comparable<Person> {
    private final String name;
    private final int age;

    Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() { return name; }
    public int getAge()    { return age; }

    @Override
    public int compareTo(Person o) {
        return Integer.compare(this.age, o.age);   // 自然顺序：按年龄升序
    }

    @Override
    public String toString() { return name + "(" + age + ")"; }
}

class InterfaceDemo {
    public static void main(String[] args) {
        java.util.List<Person> people = new java.util.ArrayList<>(java.util.List.of(
                new Person("张三", 30), new Person("李四", 25), new Person("王五", 35)));

        java.util.Collections.sort(people);                       // 用 Comparable 的自然顺序
        System.out.println(people);                               // [李四(25), 张三(30), 王五(35)]

        people.sort(java.util.Comparator.comparing(Person::getName));   // 换一种排法：按名字
        System.out.println(people);                               // [张三(30), 李四(25), 王五(35)]

        people.sort(java.util.Comparator.comparingInt(Person::getAge).reversed());  // 再换：年龄倒序
        System.out.println(people);                               // [王五(35), 张三(30), 李四(25)]

        // Iterable + AutoCloseable 的组合：一个既能 for-each 又能自动关闭的容器
        try (Resource r = new Resource()) {
            for (String item : r) {                               // for-each 依赖 Iterable
                System.out.print(item + " ");                     // a b c
            }
            System.out.println();
        }                                                         // 资源已关闭
    }
}

// 同时实现 Iterable（可遍历）与 AutoCloseable（可自动关闭）
class Resource implements Iterable<String>, AutoCloseable {
    @Override
    public java.util.Iterator<String> iterator() {
        return java.util.List.of("a", "b", "c").iterator();
    }

    @Override
    public void close() {
        System.out.println("资源已关闭");   // try-with-resources 会在块结束时自动调用
    }
}
```

> **最佳实践**：自定义容器类时，**实现 `Iterable` 而不是暴露内部数组**——调用方就能用 `for-each` 遍历，且内部结构可以随时更换。管理任何有生命周期的东西（文件、连接、锁、临时目录）时，**实现 `AutoCloseable`**，把清理逻辑放进 `close()`，调用方就能用 try-with-resources 保证不泄漏。

# 9. 小结

| 分组 | 最该记住的三件事 |
|------|-----------------|
| 基础类 | `Object` 的 `equals`/`hashCode` 必须成对重写；`Class` 是反射入口；`Package` 能查出 jar 版本 |
| 包装类 | `Integer` 缓存 `-128~127`，比较数值**一律用 `equals`**；拆箱遇 `null` 抛 NPE；`Integer` 里有一整套位运算工具 |
| 字符串 | `String` 不可变，`StringBuilder` 只在**循环拼接**时才必要；`isEmpty` 与 `isBlank` 不是一回事 |
| 线程并发 | `ThreadLocal` **谁 set 谁 remove**，且放进 `finally`；其余并发工具都在 `java.util.concurrent` |
| 系统控制 | 测耗时用 `nanoTime`；数组拷贝用 `arraycopy`；`getProperty` 读系统属性、`getenv` 读环境变量；`ProcessBuilder` 调外部程序要合并 stderr |
| 异常体系 | `Throwable` → `Error` / `Exception`；`RuntimeException` 及其子类是非受检；`NoSuchMethodError` 是依赖冲突的信号 |
| 数学实用 | `round` 回 `long`、`rint` 行银行家舍入；`Math.abs(Integer.MIN_VALUE)` **仍为负**；金额用 `BigDecimal` 不用 `double` |
| 常用接口 | `Comparable` 是自然顺序、`Comparator` 是外部顺序；`Iterable` 管遍历、`AutoCloseable` 管释放；`Runnable` 无返回值、`Callable` 有 |

`java.lang` 是"默认在用、却很少系统学过"的包，把它过一遍的收益是：以后遇到 `NoSuchMethodError`、`ThreadLocal` 泄漏、`Integer` 比较失效这类问题时，能立刻想到去哪里查。

下一篇（第 25 篇）进入注解（Annotation），看 `@Override`、`@Deprecated` 之外的注解如何通过元注解定义、又如何被反射读取——那正是 `java.lang.reflect` 与 `java.lang.annotation` 两个包配合的舞台。
