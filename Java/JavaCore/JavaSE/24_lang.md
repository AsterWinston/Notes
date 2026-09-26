# 1. 基础类

```
Object	所有类的基类
Class	表示类的运行时信息
Enum	枚举类型的基类
System	系统级操作类（IO、GC等）
Runtime	JVM 运行时环境控制
Void	代表 void 类型
```

# 2. 包装类

```
boolean	Boolean
byte	Byte
char	Character
short	Short
int	Integer
long	Long
float	Float
double	Double
```

# 3. 字符串处理类

```
String	不可变字符串
StringBuilder	可变字符串（非线程安全）
StringBuffer	可变字符串（线程安全）
```

# 4. 线程和并发

```
Thread	线程类
ThreadGroup	一组线程管理
Runnable	线程任务接口
ThreadLocal	线程局部变量
InterruptedException	线程中断异常
IllegalThreadStateException	线程状态异常
```

# 5. 系统与运行控制

```
System	输入输出、环境变量、时间戳、GC
Runtime	控制 JVM，如运行外部程序
Process	表示外部进程
```

# 6. 错误与异常类

```
Exception	所有可捕获异常的父类
RuntimeException	运行时异常父类（可不强制捕获）
Error	严重错误，不建议捕获
```

# 7. 数学和实用类

```
Math	常用数学函数类（如 sin, cos, pow, sqrt）
StrictMath	精度更严格的 Math 替代品
Number	所有数值包装类的父类（抽象类）
```

# 8. 其他接口类

```
Comparable<T>	排序比较接口（如 String, Integer 实现它）
CharSequence	字符序列接口（String, StringBuilder 实现它）
AutoCloseable	可自动关闭资源的接口（用于 try-with-resources）
Cloneable	支持对象克隆的标记接口
Iterable<T>	可迭代对象（for-each 循环）
Appendable	可追加字符序列（如 StringBuilder）
```