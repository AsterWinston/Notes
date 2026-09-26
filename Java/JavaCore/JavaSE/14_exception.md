# 1.异常分类

## 1.1Error

```
Error：JVM 内部错误，通常是系统级别问题，程序本身无法处理，不应该被 catch。
常见 Error：
  OutOfMemoryError   // 堆内存不足，GC 也无法回收足够空间
  StackOverflowError // 栈溢出，通常由无限递归导致
  NoClassDefFoundError // 运行时找不到类文件（编译时存在但运行时不存在）
  NoSuchMethodError  // 运行时找不到方法（编译时存在但运行时版本不匹配）
```

## 1.2Exception

### 1.2.1运行时异常（RuntimeException本身和其子类）

```
运行时异常（RuntimeException / Unchecked Exception）：
  编译时不强制要求处理，运行时可能发生。

常见运行时异常：
  NullPointerException           // 调用 null 对象的方法/属性
  ArrayIndexOutOfBoundsException // 数组索引越界
  StringIndexOutOfBoundsException// 字符串索引越界
  ArithmeticException            // 算术异常，如除以零
  ClassCastException             // 类型转换异常
  IllegalArgumentException       // 非法参数异常
  NumberFormatException          // 数字格式异常（如 "abc" 转 Integer）
  ConcurrentModificationException // 并发修改异常（迭代器遍历时修改集合）
  IndexOutOfBoundsException      // 索引越界（List/Array 的父异常）
  UnsupportedOperationException  // 不支持的操作（如 Arrays.asList() 返回的 List 不支持 add）
```

### 1.2.2编译时异常（除了RuntimeException和其子类外的）

```
编译时异常（Checked Exception）：
  在编译阶段就必须处理的异常，否则编译不通过。

常见编译时异常：
  IOException                    // IO 操作异常（父类）
  FileNotFoundException          // 文件未找到（IOException 子类）
  SQLException                   // 数据库操作异常
  ClassNotFoundException         // 类未找到（Class.forName 时）
  InterruptedException           // 线程中断异常
  ParseException                 // 日期解析异常
  CloneNotSupportedException     // 不支持克隆异常
  MalformedURLException          // URL 格式错误异常
```

# 2.异常的处理方式

## 2.1JVM默认的处理方式

```
2.1.1 把异常的名称、异常的原因以及异常出现的位置等信息输出在了控制台（通过 e.printStackTrace()）
2.1.2 程序停止执行，下面的代码不会再执行了

JVM 默认处理流程：
  1. 程序出现异常，JVM 创建对应的异常对象
  2. 在调用栈中逐层向上查找合适的 catch 块
  3. 若当前方法中找不到，则抛给调用者
  4. 若到 main 方法仍找不到 → 交由 JVM 的默认异常处理器
  5. 打印异常堆栈信息到 System.err，终止程序
```

## 2.2自己处理

### 2.2.1格式

```java
try {
    sentences; // 可能出现异常的代码
} catch (exception_type var_name) {
    sentences; // 异常处理逻辑
} [catch(...){...}...] // 可选的多个 catch 块
```

### 2.2.2注意

```
2.2.2.1 catch多种异常，父类异常放子类异常下面（异常从上往下匹配，如果子类在下，则永远不会匹配到，造成编译错误）
2.2.2.2 一个catch如果想捕获多个异常，多个异常参数用|连接（仅仅在jdk7之后）
      例如：catch (IOException | SQLException e) { ... }
      注意：| 连接的异常不能有父子关系
2.2.2.3 如果抛出异常但是没有被catch语句捕获，那么异常会被JVM以默认方式处理
```

### 2.2.3try-catch 执行流程详解

```
try-catch 执行流程：
  1. try 块中的代码逐行执行
  2. 如果某行抛出异常 → 跳转到匹配的 catch 块
     → catch 块执行完毕后继续执行 try-catch 后面的代码
  3. 如果 try 中没有异常 → 跳过所有 catch 块 → 继续执行后面的代码
  4. try 块中异常位置之后的代码不会再执行（因为直接跳转了）

try-catch-finally 执行流程：
  1. try 正常 → finally → 继续执行后面的代码
  2. try 异常 → catch 匹配并执行 → finally → 继续执行后面的代码
  3. try 异常，但未匹配到 catch → finally → 异常传给上层调用者
  4. 即使 try/catch 中有 return，finally 也会在 return 之前执行
```

## 2.3抛出异常

### 2.3.1throws

```
throws:
  2.3.1.1 写在方法定义处，表示声明一个异常，告诉调用者，使用本方法可能会有哪些异常
        public void readFile(String path) throws IOException {
            // ...
        }
  2.3.1.2 编译时异常必须要写（或 try-catch），运行时异常可以不写
  2.3.1.3 子类重写父类方法时，不能声明比父类更多的异常（不能抛出父类方法未声明的编译时异常）
```

### 2.3.2throw

```
throw：写在方法体内，用于手动抛出一个异常对象。
  格式：throw new 异常类型("异常信息");
  一旦执行 throw，方法立即终止，不会执行后面的代码。
  
示例：
  public void checkAge(int age) {
      if (age < 0 || age > 150) {
          throw new IllegalArgumentException("年龄不合法: " + age);
      }
  }
```

# 3.Throwable的成员方法

```
3.1 public String getMessage()
    返回此 throwable 的详细消息字符串（即构造时传入的 message）。
    
3.2 public String toString()
    返回此可抛出的简短描述：类名 + ": " + getMessage()。
    
3.3 public void printStackTrace()
    把异常的错误信息输出在控制台（System.err），
    包括异常类型、原因和完整的堆栈跟踪信息。
    这是 JVM 默认异常处理器的处理方式。

3.4 public Throwable getCause()
    返回引发当前异常的原始异常（异常链）。

3.5 public StackTraceElement[] getStackTrace()
    返回堆栈跟踪信息数组，数组的第一个元素是异常抛出位置。
```

# 4.自定义异常

```
4.1 定义异常类（类名应以 Exception 或 Error 结尾）
4.2 写继承关系（通常继承 Exception 表示编译时异常，继承 RuntimeException 表示运行时异常）
4.3 提供空参构造
4.4 提供带参构造（通常包含 String message 参数，调用 super(message)）
4.5 可选：提供一个带有 Throwable cause 的构造方法（用于异常链）

完整示例：

// 自定义编译时异常
public class UserNotFoundException extends Exception {
    public UserNotFoundException() {}

    public UserNotFoundException(String message) {
        super(message);
    }

    public UserNotFoundException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

# 5.finally 详解

```
finally 的作用：
  finally 中的代码一定会执行（除非 JVM 提前退出如 System.exit(0)）。
  通常用于释放资源（关闭流、关闭连接等）。

5.1 finally 中的 return 陷阱：
  try 中有 return 语句时，finally 仍然会在 return 之前执行。
  但是如果 finally 中也有 return，会覆盖 try 中的 return 值！

  示例（陷阱）：
    public static int test() {
        try {
            return 1;
        } finally {
            return 2;  // 最终返回 2！try 中的 return 1 被覆盖
        }
    }
    // 调用 test() 返回 2

  最佳实践：永远不要在 finally 中使用 return。

5.2 final / finally / finalize 的区别：
  final：修饰符，表示不可变（类不可继承、方法不可重写、变量不可修改）
  finally：异常处理中，确保代码一定执行
  finalize：Object 的方法，GC 回收对象前调用（已废弃，JDK9 起不推荐）
```

# 6.try-with-resources（自动关闭资源）

```
JDK7 引入的语法糖。资源对象实现了 AutoCloseable 接口的，可以在 try 中声明，
执行完毕或异常时会自动调用 close() 方法。

6.1 JDK7 格式：
  try (FileInputStream fis = new FileInputStream("a.txt");
       FileOutputStream fos = new FileOutputStream("b.txt")) {
      // 使用 fis 和 fos
      // 无论是否异常，fis 和 fos 都会被自动关闭
  } catch (IOException e) {
      e.printStackTrace();
  }
  // 关闭顺序：先打开的后关闭（fos 先关，fis 后关）

6.2 JDK9 增强格式：
  可以把资源的声明写在 try 外面，try 只负责关闭：
  FileInputStream fis = new FileInputStream("a.txt");
  FileOutputStream fos = new FileOutputStream("b.txt");
  try (fis; fos) {
      // 使用 fis 和 fos
  } catch (IOException e) {
      e.printStackTrace();
  }

6.3 AutoCloseable vs Closeable：
  AutoCloseable（java.lang）：close() throws Exception
  Closeable（java.io，继承自 AutoCloseable）：close() throws IOException
  大多数 IO 资源类都实现了 Closeable。
```

# 7.异常链（Exception Chaining）

```
异常链用于将一个异常包装成另一个异常，同时保留原始异常信息。
常用于：底层模块抛出的异常在传递给上层时转换成语义更清晰的异常。

构造方式：
  通过构造函数传入 cause：
    try {
        // ...
    } catch (IOException e) {
        throw new ServiceException("服务调用失败", e); // e 作为 cause
    }

  通过 initCause 方法（通常不需要手动调用）：
    ServiceException se = new ServiceException("服务调用失败");
    se.initCause(new IOException("网络错误"));

获取原始异常：
  Throwable cause = e.getCause();
```

# 8.异常最佳实践

```
8.1 不要忽略异常
    错误做法：
      try { ... } catch (Exception e) { }  // 空 catch，吞掉异常
    正确做法：至少打印日志

8.2 不要捕获太宽泛的异常
    错误做法：catch (Exception e) — 把运行时异常也捕获了
    正确做法：只捕获你能够处理的特定异常

8.3 尽量使用 try-with-resources 替代手动关闭

8.4 自定义异常应传递有意义的信息
    throw new IllegalArgumentException("年龄必须在0-150之间，当前值: " + age);

8.5 运行时异常 vs 编译时异常的选择
    如果调用者可以合理地处理异常 → 编译时异常
    如果通常是编程错误导致的（如空指针、越界） → 运行时异常
```
