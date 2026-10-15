异常是 Java 处理"运行期出错"的统一机制：它把"出错了"从五花八门的返回值（`-1`、`null`、错误码）升格为一种**类型**，从而能被捕获、分类、包装和传播。本篇从 `Throwable` 体系讲到受检/非受检异常的设计取舍，再逐一讲透 `try-catch-finally`、`try-with-resources`、异常链与性能开销，最后落到一份可落地的最佳实践清单。

# 1. 异常体系

## 1.1 异常体系结构

Java 中所有能被 `throw` 的东西都继承自 `Throwable`，它下面只有两个直接子类：`Error` 和 `Exception`。

```text
                        ┌─────────────┐
                        │  Throwable  │   所有可抛出物的根
                        └──────┬──────┘
                               │
              ┌────────────────┴────────────────┐
              ▼                                 ▼
       ┌─────────────┐                   ┌─────────────┐
       │    Error    │                   │  Exception  │
       └─────────────┘                   └──────┬──────┘
      JVM 级严重错误，程序                          │
      无法处理，不该 catch                  ┌──────┴──────┐
                                            ▼             ▼
                                   ┌────────────────┐  ┌───────────────┐
                                   │RuntimeException│  │ 其它 Exception │
                                   └────────────────┘  └───────────────┘
                                     非受检（Unchecked）    受检（Checked）
                                     编译器不强制处理       编译器强制处理
```

三层的职责分得很清楚：`Error` 是**JVM 自己的问题**，`RuntimeException` 是**程序写错了**，其余 `Exception` 是**外部环境出问题了**。后面所有处理策略上的争论，本质上都是围绕这三层的划分展开的。

## 1.2 Error：程序不该处理的错误

`Error` 表示 JVM 内部错误，通常是系统级别的问题，**程序本身无法处理，也不应该被 catch**——因为一旦发生，JVM 的运行环境已经不可信了，勉强继续执行只会产生更难排查的连锁故障。

| 常见 `Error` | 触发原因 |
|--------------|----------|
| `OutOfMemoryError` | 堆内存不足，GC 也无法回收出足够空间 |
| `StackOverflowError` | 栈溢出，通常由无限递归导致 |
| `NoClassDefFoundError` | 运行时找不到类文件（编译时存在但运行时不存在） |
| `NoSuchMethodError` | 运行时找不到方法（编译时存在但运行时版本不匹配） |

## 1.3 Exception：受检与非受检

`Exception` 又按"编译器管不管"分成两支。

### 1.3.1 运行时异常（`RuntimeException` 及其子类）

**编译时不强制要求处理**，运行时才可能发生。它们几乎全都是"代码写错了"的表现，属于**非受检异常（Unchecked Exception）**。

| 常见运行时异常 | 触发原因 |
|----------------|----------|
| `NullPointerException` | 调用 `null` 对象的方法或属性 |
| `ArrayIndexOutOfBoundsException` | 数组索引越界 |
| `StringIndexOutOfBoundsException` | 字符串索引越界 |
| `IndexOutOfBoundsException` | 索引越界（`List` / 数组越界的公共父类） |
| `ArithmeticException` | 算术异常，如整数除以零 |
| `ClassCastException` | 类型转换异常 |
| `IllegalArgumentException` | 非法参数异常 |
| `NumberFormatException` | 数字格式异常（如把 `"abc"` 转成 `Integer`） |
| `ConcurrentModificationException` | 并发修改异常（迭代器遍历时修改集合） |
| `UnsupportedOperationException` | 不支持的操作（如 `Arrays.asList()` 返回的 `List` 不支持 `add`） |

### 1.3.2 编译时异常（除 `RuntimeException` 及其子类外的 `Exception`）

**在编译阶段就必须处理**，不处理则编译不通过，属于**受检异常（Checked Exception）**。

| 常见编译时异常 | 触发原因 |
|----------------|----------|
| `IOException` | IO 操作异常的父类 |
| `FileNotFoundException` | 文件未找到（`IOException` 的子类） |
| `SQLException` | 数据库操作异常 |
| `ClassNotFoundException` | 类未找到（`Class.forName` 时） |
| `InterruptedException` | 线程中断异常 |
| `ParseException` | 日期解析异常 |
| `CloneNotSupportedException` | 不支持克隆异常 |
| `MalformedURLException` | URL 格式错误异常 |

## 1.4 受检与非受检的本质区别与设计取舍

两者的差别**不只是"编译期要不要写 try"，而是"这个错误该由谁负责"**：

| 维度 | 受检异常（`Checked`） | 非受检异常（`Unchecked`） |
|------|----------------------|--------------------------|
| 编译器是否强制处理 | **强制**，不处理编译不过 | 不强制，可以不写 `try` 也不写 `throws` |
| 典型代表 | `IOException`、`SQLException` | `NullPointerException`、`IllegalArgumentException` |
| 错误的来源 | **外部环境**：文件被删、网络断了、数据库挂了 | **程序本身**：空指针、越界、参数非法 |
| 调用者能否合理恢复 | 通常**能**——重试、换路径、提示用户 | 通常**不能**——只能改代码、修 bug |
| 设计意图 | 把"必须考虑失败的场景"写进方法签名 | 让 API 签名保持干净，不被无意义的 `throws` 污染 |

**为什么 `RuntimeException` 不用声明？** 因为强制声明这类异常**没有收益**：`NullPointerException` 出现在哪一行是运行期才知道的事，让每个方法都写 `throws NullPointerException` 只会制造噪音，而调用者也不可能针对它写有意义的恢复逻辑（正确做法是修掉产生 `null` 的代码）。反过来，`IOException` 是方法的**正常业务语义的一部分**——"读这个文件"天然就可能"文件不存在"，所以它必须出现在签名里，逼调用者表态。

```java
// 受检异常：签名里明明白白告诉你"这个方法会失败"
static String readFile(String path) throws IOException { /* ... */ }

// 非受检异常：签名干净，但错了就是错了
static int parseInt(String s) { return Integer.parseInt(s); }   // 可能抛 NumberFormatException，无需声明
```

> **注意**：这套设计在业界一直有争议。反对者认为受检异常强迫上层写大量"我其实处理不了"的 `try-catch` 或层层 `throws`，反而让代码变脏；Spring、JPA 等框架因此把大量底层受检异常统一转换成了非受检异常（如 `DataAccessException`）。**实践中的折中方案**是：**能被调用者合理恢复的外部故障用受检异常，其余一律用非受检异常**，并在业务层用自定义的非受检异常 + 异常链统一包装。

# 2. 异常的处理方式

## 2.1 JVM 默认的处理方式

如果异常一路抛出、没有任何 `catch` 接住，就交由 JVM 的默认异常处理器处理，它会做两件事：

1. 把异常的名称、原因以及出现位置等信息输出到控制台（即 `e.printStackTrace()` 的效果）；
2. **终止程序**，抛出点之后的代码不会再执行。

完整的查找流程如下：

1. 程序出现异常，JVM 创建对应的异常对象；
2. 沿调用栈**逐层向上查找**合适的 `catch` 块；
3. 若当前方法里找不到，就把异常抛给调用者；
4. 若一直抛到 `main` 方法仍找不到，交给 JVM 的默认异常处理器；
5. 打印异常堆栈信息到 `System.err`，终止程序。

```java
public static void main(String[] args) {
    System.out.println("开始");
    System.out.println(10 / 0);        // 抛 ArithmeticException: / by zero
    System.out.println("这行不会执行");  // 永远看不到
}
// 输出：
// 开始
// Exception in thread "main" java.lang.ArithmeticException: / by zero
//     at Demo.main(Demo.java:3)
```

> **注意**：异常信息打印到的是 `System.err` 而不是 `System.out`，两者是**不同的流**。在有重定向或异步日志的环境里，这会导致异常信息出现在输出文件的不同位置，排查时别以为是丢了日志。

## 2.2 自己处理：try-catch

### 2.2.1 基本格式

```java
try {
    // 可能出现异常的代码
} catch (异常类型 变量名) {
    // 异常处理逻辑
}
// 可选的多个 catch 块
```

### 2.2.2 catch 的匹配规则

捕获取胜的原则是"**就近匹配，从上往下**"，由此派生三条规则：

1. **父类异常必须写在子类异常下面**。匹配是从上往下进行的，如果把父类写在前面，子类分支**永远不会被匹配到**，编译器会直接报错；
2. **一个 `catch` 想捕获多种异常，用 `|` 连接**（JDK7 起支持），例如 `catch (IOException | SQLException e) { ... }`；用 `|` 连接的异常**不能有父子关系**（否则前者已经涵盖后者，没有意义）；
3. **如果异常被抛出却没有被任何 `catch` 捕获**，它会继续向上传播，最终由 JVM 以默认方式处理（即 2.1 的两步）。

```java
try {
    // ...
} catch (FileNotFoundException e) {     // 子类在前：先精确匹配
    System.out.println("文件不存在：" + e.getMessage());
} catch (IOException e) {               // 父类在后：兜住其余 IO 问题
    System.out.println("IO 出错：" + e.getMessage());
}
// 反过来写（IOException 在前）会编译报错：已被捕获的异常
```

### 2.2.3 try-catch 的执行流程

不带 `finally` 时：

1. `try` 块中的代码逐行执行；
2. 如果某行抛出异常，立即跳转到匹配的 `catch` 块，`catch` 执行完继续执行 `try-catch` **后面**的代码；
3. 如果 `try` 中没有异常，跳过所有 `catch` 块，直接执行后面的代码；
4. **`try` 块中异常位置之后的代码不会执行**（因为已经跳转到 `catch` 了）。

带 `finally` 时：

| 情况 | 执行顺序 |
|------|----------|
| `try` 正常结束 | `try` → `finally` → 后面的代码 |
| `try` 抛异常且被捕获 | `try`（执行到出错处）→ `catch` → `finally` → 后面的代码 |
| `try` 抛异常但没匹配到 `catch` | `try` → `finally` → 异常继续抛给上层调用者 |
| `try` 或 `catch` 中有 `return` | `finally` 仍然会在 `return` 之前执行（详见第 5 节） |

> **最佳实践**：`try` 块里**只放真正可能抛异常的代码**，不要图省事把几十行全塞进去。`try` 块越大，越说不清到底是哪一句出的错，也越容易顺手捕获到不该捕获的异常。

## 2.3 抛出异常

### 2.3.1 throws：声明异常

`throws` 写在**方法定义处**，表示"本方法可能会抛出这些异常"，用来提醒调用者。

```java
public void readFile(String path) throws IOException {
    // ...
}
```

三条要点：

1. **编译时异常必须写 `throws`（或者就地 `try-catch`）**，否则编译不通过；运行时异常可以不写；
2. **子类重写父类方法时，不能声明比父类更宽泛的异常**——不能抛出父类方法未声明的编译时异常，也不能抛出父类异常父类的异常。这是为了保证"用父类型调用子类对象"时的编译期契约不被破坏；
3. 可以一次声明多个，用逗号分隔，如 `throws IOException, SQLException`。

### 2.3.2 throw：手动抛出异常

`throw` 写在**方法体内**，用于手动抛出一个异常对象。格式为 `throw new 异常类型("异常信息");`。一旦执行 `throw`，**方法立即终止**，其后的代码不会执行。

```java
public void checkAge(int age) {
    if (age < 0 || age > 150) {
        throw new IllegalArgumentException("年龄不合法: " + age);
    }
    System.out.println("年龄校验通过: " + age);
}
```

> **注意**：`throws`（有 s，复数）是**声明**，出现在方法签名上；`throw`（无 s）是**动作**，出现在方法体里。两者一个是"可能抛"，一个是"现在就抛"。

# 3. Throwable 的成员方法

| 方法 | 返回值 | 说明 |
|------|--------|------|
| `getMessage()` | `String` | 返回详细消息字符串，即构造时传入的 `message` |
| `toString()` | `String` | 返回简短描述，格式为 **类名 + ": " + `getMessage()`** |
| `printStackTrace()` | `void` | 把异常的错误信息输出到 `System.err`，包含异常类型、原因和完整堆栈跟踪；这正是 JVM 默认异常处理器的处理方式 |
| `getCause()` | `Throwable` | 返回引发当前异常的原始异常（异常链，见第 7 节） |
| `getStackTrace()` | `StackTraceElement[]` | 返回堆栈跟踪数组，**第一个元素是异常抛出的位置** |
| `getSuppressed()` | `Throwable[]` | 返回被"抑制"的异常数组（`try-with-resources` 场景，见 6.3） |

```java
try {
    int[] arr = new int[2];
    System.out.println(arr[5]);
} catch (ArrayIndexOutOfBoundsException e) {
    System.out.println(e.getMessage());      // Index 5 out of bounds for length 2
    System.out.println(e.toString());        // java.lang.ArrayIndexOutOfBoundsException: Index 5 out of bounds for length 2
    System.out.println(e.getStackTrace()[0]);// Demo.main(Demo.java:3)，第一个元素就是抛出点
    // e.printStackTrace();                  // 完整堆栈，输出到 System.err
}
```

# 4. 自定义异常

JDK 内置的异常表达不了业务语义时，就定义自己的异常类型。标准做法是五步：

1. 定义异常类，**类名以 `Exception` 或 `Error` 结尾**（这是全 Java 世界的命名约定）；
2. 确定继承关系——**继承 `Exception` 表示受检异常，继承 `RuntimeException` 表示运行时异常**；业务异常通常选后者；
3. 提供**空参构造**；
4. 提供**带参构造**，通常包含 `String message` 参数并调用 `super(message)`；
5. 可选：提供一个带 `Throwable cause` 的构造方法，用于传递异常链。

```java
// 自定义编译时异常
public class UserNotFoundException extends Exception {
    public UserNotFoundException() {}

    public UserNotFoundException(String message) {
        super(message);
    }

    public UserNotFoundException(String message, Throwable cause) {
        super(message, cause);      // 保留原始异常，形成异常链
    }
}

// 自定义运行时异常（业务中最常见）
public class BusinessException extends RuntimeException {
    private final String code;      // 业务错误码，便于上游做分支处理

    public BusinessException(String code, String message) {
        super(message);
        this.code = code;
    }

    public BusinessException(String code, String message, Throwable cause) {
        super(message, cause);
        this.code = code;
    }

    public String getCode() { return code; }
}
```

> **最佳实践**：自定义异常不要只加一个 `message` 就完事——**带上业务错误码**才能让调用方在不解析字符串的前提下做分支判断（比如前端根据 `code` 决定是提示重试还是跳登录页）。另外，不要为了省事让所有业务异常都继承同一个根，适当细分（`UserNotFoundException` / `InsufficientBalanceException`）才能被精确 `catch`。

# 5. finally 详解

## 5.1 finally 的执行时机

`finally` 中的代码**一定会执行**（除非 JVM 提前退出，见 5.3）。它的典型用途是**释放资源**：关闭流、关闭数据库连接、释放锁。因为无论 `try` 里是正常结束、抛异常、还是 `return`，收尾逻辑都不会被漏掉。

```java
FileInputStream fis = null;
try {
    fis = new FileInputStream("a.txt");
    // 使用 fis
} catch (IOException e) {
    e.printStackTrace();
} finally {
    if (fis != null) {
        try { fis.close(); } catch (IOException e) { e.printStackTrace(); }
    }
}
```

> **注意**：上面这种"手动关资源"的写法在 JDK7 之后已经被 `try-with-resources` 取代（见第 6 节）——后者代码更短，而且**即使 `close()` 本身失败也不会吞掉主异常**。

## 5.2 finally 与 return 的执行顺序

这是最高频的考点，也是实际写代码时最容易踩的坑。结论先说：**`finally` 一定在 `return` 真正返回之前执行**，但 `return` 的**值**早在进入 `finally` 之前就已经被确定了。

```java
public class FinallyReturnDemo {

    // 情况一：finally 里没有 return —— try 的返回值已被"暂存"，finally 改局部变量无效
    static int noReturnInFinally() {
        int x = 1;
        try {
            return x;              // ① 先把 x 的值 1 存到临时槽位
        } finally {
            x = 100;               // ② finally 改的是局部变量 x，改不动已暂存的返回值
            System.out.println("finally 执行了，x = " + x);   // 输出 x = 100
        }
    }

    // 情况二：finally 里有 return —— 直接覆盖 try 的返回值
    static int returnInFinally() {
        try {
            return 1;
        } finally {
            return 2;              // 覆盖 return 1
        }
    }

    public static void main(String[] args) {
        System.out.println("结果：" + noReturnInFinally());   // finally 执行了，x = 100 → 结果：1
        System.out.println("结果：" + returnInFinally());     // 结果：2
    }
}
```

字节码层面看得很清楚：`return x` 会先把 `x` 的值**复制到栈顶**，然后才转到 `finally` 块执行；`finally` 结束后再把这个**早已复制好的值**返回。所以 `finally` 里改 `x` 对返回值毫无影响——被改的只是局部变量本身。

> **陷阱**：如果 `try` 中 `return` 的是一个**引用类型**，`finally` 里修改这个对象的**内容**是生效的（因为它们指向同一个对象），但让引用指向新对象则无效。这比 `int` 的情况更隐蔽：
> ```java
> static StringBuilder bad() {
>     StringBuilder sb = new StringBuilder("a");
>     try {
>         return sb;
>     } finally {
>         sb.append("b");      // ✓ 生效：返回的是同一个对象，内容变成了 "ab"
>         sb = new StringBuilder("c");   // ✗ 无效：只改了局部引用的指向
>     }
> }
> ```

## 5.3 finally 什么情况下不执行

`finally` 并非绝对会执行。以下几种情况它会被跳过：

| 情况 | 说明 |
|------|------|
| `System.exit(int)` | 直接请求 JVM 退出，`finally` 不会执行 |
| JVM 崩溃或进程被强制杀死 | 如 `kill -9`、段错误，收尾逻辑没有机会运行 |
| 断电、机器宕机 | 属于硬件层面的中断，任何代码都救不回来 |
| `finally` 之前的代码死循环或永久阻塞 | 根本走不到 `finally` |
| 守护线程中的 `finally` | 所有非守护线程结束时，守护线程会被 JVM 直接终止，`finally` 可能来不及执行 |

```java
try {
    System.out.println("try 中");
    System.exit(0);              // JVM 立刻退出
} finally {
    System.out.println("这行不会打印");   // 被跳过了
}
```

> **注意**：正因为存在这些例外，`finally` 只能保证"**正常情况下一定会执行**"。对于**必须完成的关键业务**（如数据库事务提交、消息投递），不能只依赖 `finally`——要有补偿机制或事务保障，不能把正确性押在收尾代码上。

## 5.4 不要在 finally 里写 return

`finally` 里写 `return`（或 `throw`）会带来两个严重的后果：

1. **吞掉异常**：如果 `try` 中抛了异常，而 `finally` 里 `return`，那么异常**永远不会被抛出**，调用方完全不知道出了问题——这是最难排查的一类 bug；
2. **覆盖返回值**：如 5.2 所示，`try` 里的 `return` 值会被悄悄替换掉。

```java
// ✗ 灾难性写法：异常被静默吞掉
static int evil() {
    try {
        throw new RuntimeException("出错了！");
    } finally {
        return 0;      // 异常消失了，调用方拿到 0，还以为一切正常
    }
}
```

**最佳实践：永远不要在 `finally` 中使用 `return` 或 `throw`。** `finally` 只应该做"收尾"，不应该参与"决定返回什么"。

## 5.5 final / finally / finalize 的区别

这三个词长得像，含义却毫无关系：

| 关键字 | 类别 | 作用 |
|--------|------|------|
| `final` | **修饰符** | 表示不可变：类不可继承、方法不可重写、变量只能赋值一次 |
| `finally` | **异常处理的代码块** | 确保其中的代码一定执行（见 5.1） |
| `finalize` | **`Object` 的方法** | GC 回收对象前调用，**已废弃**，JDK9 起不推荐使用 |

# 6. try-with-resources（自动关闭资源）

`try-with-resources` 是 JDK7 引入的语法糖：只要资源对象实现了 `AutoCloseable` 接口，就可以在 `try` 的括号里声明，**代码块执行完毕或抛出异常时会自动调用它的 `close()`**，再也不用写 `finally` 里那一长串关闭代码。

## 6.1 JDK7 的格式

```java
try (FileInputStream fis = new FileInputStream("a.txt");
     FileOutputStream fos = new FileOutputStream("b.txt")) {
    // 使用 fis 和 fos
    // 无论是否发生异常，fis 和 fos 都会被自动关闭
} catch (IOException e) {
    e.printStackTrace();
}
```

## 6.2 JDK9 的增强格式

JDK9 起，可以把资源的声明写在 `try` 外面，`try` 只负责关闭（资源变量必须是 `final` 或事实上的 `final`）：

```java
FileInputStream fis = new FileInputStream("a.txt");
FileOutputStream fos = new FileOutputStream("b.txt");
try (fis; fos) {          // 这里只书写变量名，不重新声明
    // 使用 fis 和 fos
} catch (IOException e) {
    e.printStackTrace();
}
```

## 6.3 资源关闭顺序与抑制异常

**关闭顺序是"先创建的后关闭"**——可以理解为 `close()` 的调用顺序与资源的声明顺序**相反**，类似于栈的先进后出。这一点很关键：`fos` 先关、`fis` 后关，如果两者有依赖（比如写数据依赖读数据），顺序反了就会出错。

更值得讲透的是**抑制异常（Suppressed Exception）**机制。如果 `try` 块里的业务代码抛了异常，紧接着资源 `close()` 时**又抛了一个异常**，此时 Java **不会丢掉任何一个**：先抛出的那个作为**主异常**向外传播，后抛出的被挂到主异常上，通过 `getSuppressed()` 读取。

```java
public class SuppressDemo {
    static class Resource1 implements AutoCloseable {
        @Override
        public void close() { throw new RuntimeException("Resource1 关闭失败"); }
    }
    static class Resource2 implements AutoCloseable {
        @Override
        public void close() { throw new RuntimeException("Resource2 关闭失败"); }
    }

    public static void main(String[] args) {
        try (Resource1 r1 = new Resource1();     // 先声明、先构造
             Resource2 r2 = new Resource2()) {   // 后声明、后构造
            System.out.println("业务代码执行完毕");
        } catch (Exception e) {
            System.out.println("主异常：" + e.getMessage());       // Resource2 关闭失败
            for (Throwable t : e.getSuppressed()) {
                System.out.println("被抑制：" + t.getMessage());   // Resource1 关闭失败
            }
        }
    }
}
```

要点归纳：

1. 关闭顺序是**后创建的先关闭**，所以 `r2` 先 `close()`，它抛的异常成了**主异常**；
2. `r1` 后 `close()`，它抛的异常被挂为**抑制异常**，不会消失；
3. `printStackTrace()` 会把抑制异常一并打印出来，格式是 `Suppressed: ...`；
4. JDK7 也提供了 `Throwable.addSuppressed(Throwable)`，手写 `finally` 时可以用它达到同样效果（不过在 JDK7 之后没必要再手写了）。
5. 如果 `try` 块里**没有**业务异常，那么 `close()` 抛出的异常就直接是主异常。

> **最佳实践**：**凡是实现了 `AutoCloseable` 的资源，一律用 `try-with-resources`**——不要手写 `finally` 关闭。手写版本不仅要判空、要嵌套 `try`，还很容易在 `close()` 出错时把主异常挤掉，而 `try-with-resources` 由编译器保证这些细节。

## 6.4 AutoCloseable 与 Closeable

| 接口 | 包 | `close()` 的签名 | 说明 |
|------|-----|------------------|------|
| `AutoCloseable` | `java.lang` | `void close() throws Exception` | JDK7 引入，能配合 `try-with-resources` 的最小接口 |
| `Closeable` | `java.io` | `void close() throws IOException` | 继承自 `AutoCloseable`，**缩小了异常范围**、且要求 `close()` 可重复调用无副作用 |

大多数 IO 资源类（`InputStream` / `OutputStream` / `Reader` / `Writer` / `Connection` 等）实现的都是 `Closeable`。抛出的异常范围更窄，写业务代码时更省心。

# 7. 异常链

**异常链**用于把一个异常包装成另一个异常，同时**保留原始异常信息**。典型场景是：底层模块抛出的异常（如 `IOException`）传到上层时，上层用语义更清晰的异常（如 `ServiceException`）重新抛出，并且把原异常带上——这样既不暴露实现细节，又不丢失根因。

两种构造方式：

```java
// 方式一：通过构造方法传入 cause（推荐）
try {
    // 底层可能抛 IOException 的代码
} catch (IOException e) {
    throw new ServiceException("服务调用失败", e);   // e 作为 cause
}

// 方式二：通过 initCause 方法（只在无法通过构造方法传入时才用）
ServiceException se = new ServiceException("服务调用失败");
se.initCause(new IOException("网络错误"));
throw se;
```

获取原始异常用 `getCause()`；`printStackTrace()` 会打印成 `Caused by: ...` 的形式，把整条链路都展示出来：

```text
com.example.ServiceException: 服务调用失败
    at com.example.OrderService.query(OrderService.java:20)
    ...
Caused by: java.io.IOException: 网络错误
    at com.example.HttpClient.get(HttpClient.java:35)
    ...
```

> **陷阱**：最常见的错误是"**转换时把 cause 丢了**"——`throw new ServiceException("服务调用失败")` 没有传 `e`，堆栈到这一层就断了，真正的根因（超时？连接被拒？证书过期？）再也无从查起。**包装异常时必须传 cause**，这是异常链存在的全部意义。

> **最佳实践**：**不要吞异常**。"吞异常"指捕获之后既不处理也不记录也不重新抛出，例如 `catch (Exception e) { }`。它让程序"带着错误继续跑"，等到别处再暴露时已经离现场很远了。如果不打算处理，就在同一个 `catch` 里**要么记日志、要么重新抛出**，二选一。

# 8. 异常处理的性能开销

异常并非"免费的 if"，它的开销集中在两个地方：

1. **创建异常对象时的 `fillInStackTrace()`**。`Throwable` 的构造方法会调用它，**遍历当前线程的整个调用栈并逐帧记录**，栈越深越慢。相比创建普通对象，构造一个带完整堆栈的异常可能慢上一到两个数量级；
2. **异常抛出后的栈展开**。JIT 对正常路径的优化在异常路径上大多失效，本地代码需要回退到解释执行，实际成本往往比 `fillInStackTrace()` 更高。

由此得出两条实践结论：

- **绝对不要用异常控制流程**。比如用 `try { return list.get(i); } catch (IndexOutOfBoundsException e) { return null; }` 来判断越界——正确写法是先判 `i < list.size()`。异常只应表达"**异常情况**"，不是分支语句；
- 频繁发生的**已知**异常也会有可观开销。JDK 自身对此做了优化：热点路径上的隐式异常（如 NPE、越界）在反复触发后，JIT 会用 `-XX:-OmitStackTraceInFastThrow` 对应的快速抛出机制**丢弃堆栈**，此时你会看到一个**没有堆栈信息**的 `NullPointerException`，这不是 bug，而是性能优化的结果（排查时需要加上 `-XX:-OmitStackTraceInFastThrow` 关闭该优化）。

```java
// 想省掉堆栈开销，可以让自定义异常不记录堆栈（很少需要，属于极端优化）
public class NoStackTraceException extends RuntimeException {
    public NoStackTraceException(String message) {
        // protected Throwable(String message, Throwable cause, boolean enableSuppression, boolean writableStackTrace)
        super(message, null, false, false);   // writableStackTrace = false → 不调用 fillInStackTrace()
    }
}
```

> **注意**：上面这个技巧只在"异常极高频、且日志里已经有足够上下文"时才值得用。**不要为了省性能而牺牲堆栈**——没有堆栈的异常在排查线上问题时几乎等于没抛。绝大多数业务异常的频率都很低，正常写法就够了。

# 9. 异常处理最佳实践

| 实践 | 说明 |
|------|------|
| **不要用异常控制流程** | 异常只表达异常情况；分支判断用 `if`，别用 `catch` 兜 |
| **捕获具体异常，而非 `Exception`** | `catch (Exception e)` 会把运行时异常一并吞进来，掩盖真正的 bug；只捕获你能处理的特定异常 |
| **不要吞异常** | `catch (Exception e) { }` 是最坏写法；至少记日志或重新抛出 |
| **不要重复打印** | 已经 `log.error(...)` 了就别再 `throw` 同一个异常，或用 `printStackTrace()` 打印第二遍，否则日志里同一异常出现多次 |
| **包装异常必须传 cause** | `throw new ServiceException(msg, e)`，否则堆栈断链（见第 7 节） |
| **尽量用标准异常** | `IllegalArgumentException`、`IllegalStateException`、`UnsupportedOperationException`、`IndexOutOfBoundsException` 已被 JDK 广泛使用，自定义代替不了它们的语义 |
| **优先 `try-with-resources`** | 替代手写 `finally` 关资源，还能保住抑制异常（见 6.3） |
| **不要让 `finally` 影响返回值** | 不要在 `finally` 里写 `return` / `throw`（见 5.4） |
| **异常信息要有意义** | `throw new IllegalArgumentException("年龄必须在 0-150 之间，当前值: " + age)` 远好于 `throw new IllegalArgumentException("参数错误")` |
| **受检/非受检的选择** | 调用者能合理处理的用受检异常；通常是编程错误（空指针、越界）的用非受检异常（见 1.4） |
| **返回值的空值处理用 `Optional`** | 用"返回 `null` 表示没有结果"容易被忽略；改为返回 `Optional` 能强制调用方处理，见第 28 篇 |
| **不要用异常做返回值兼容** | 如需表示"查不到"，用 `Optional` / 空集合 / 业务错误码，而不是抛一个 `NotFoundException` 让调用方天天 `try-catch` |

```java
// ✗ 反面教材：宽泛捕获 + 吞异常 + 重复打印
try {
    process();
} catch (Exception e) {        // 太宽泛，连 NPE 都吞了
    e.printStackTrace();       // 只打印、不处理、不抛出
}

// ✓ 正面写法：精确捕获、有意义的包装、单一出口
try {
    process();
} catch (IOException e) {
    log.error("处理订单 {} 失败，原因：{}", orderId, e.getMessage(), e);   // 记录一次，带上下文
    throw new BusinessException("ORDER_PROCESS_FAILED", "订单处理失败", e);  // 包装成业务异常向上抛
}
```

> **最佳实践**：在所有规则里，**"不要吞异常"和"包装时传 cause"这两条最重要**——线上问题排查的时间，九成耗在"异常在某一层消失了"或"堆栈断在半路"上。其余规则都是锦上添花，这两条是底线。

# 10. 小结

| 主题 | 一句话要点 |
|------|-----------|
| 异常体系 | `Throwable` → `Error`（JVM 级，不该 `catch`）/ `Exception` → `RuntimeException`（非受检）与其它（受检） |
| 受检 vs 非受检 | 本质区别是"**谁该负责**"：受检是外部环境故障、调用者能恢复；非受检是程序 bug。强制声明 `RuntimeException` 只会污染签名，没有收益 |
| 处理方式 | JVM 默认 = 打印堆栈 + 终止程序；自己处理用 `try-catch`（父类异常写子类下面，多异常用 `|` 连接）；对外声明用 `throws`，就地抛出用 `throw` |
| `Throwable` 方法 | `getMessage()`（消息）/ `toString()`（类名 + 消息）/ `printStackTrace()`（完整堆栈）/ `getCause()`（异常链）/ `getSuppressed()`（抑制异常） |
| 自定义异常 | 类名以 `Exception` 结尾；继承 `Exception` 是受检、继承 `RuntimeException` 是非受检；提供带 `message` 与 `cause` 的构造；**带上业务错误码** |
| `finally` 与 `return` | `finally` 一定在 `return` 之前执行，但 `try` 的返回值**已提前确定**，`finally` 里改局部变量无效；`finally` 里的 `return` 会**覆盖返回值并吞掉异常**，永远别写 |
| `finally` 不执行 | `System.exit()`、JVM 崩溃 / 进程被强杀 / 断电、`finally` 之前死循环、守护线程被终止 |
| `try-with-resources` | 实现 `AutoCloseable` 即可自动关闭；**关闭顺序与声明顺序相反**；`close()` 的异常被挂为**抑制异常**，可用 `getSuppressed()` 取出；`Closeable` 是 `AutoCloseable` 的 IO 版 |
| 异常链 | 用 `new XxxException(msg, cause)` 保留根因，`initCause` 是备选；打印成 `Caused by:`；**丢了 cause 等于丢了根因** |
| 性能开销 | 主要在 `fillInStackTrace()` 和栈展开，比普通对象慢一到两个数量级；因此**不要用异常控制流程** |
| 最佳实践底线 | **不吞异常、包装必传 cause、捕获具体异常、不在 `finally` 里 `return`、不用异常控制流程**；返回值空值处理见第 28 篇 `Optional` |

下一批文件（第 15 篇 `File`、第 16 篇 IO 流）会大量用到本篇的 `try-with-resources` 与 `IOException` 处理；`Optional` 如何替代"返回 `null`"见第 28 篇。
