# 1. 定义（是什么）

```
record 是 Java 一种特殊的类，用来简化不可变数据载体类（data carrier class）的定义。

语法糖，编译器会自动为你生成：
private final 字段（对应构造参数）
全参构造器
getter（方法名就是字段名）
equals()、hashCode()、toString()

可以把它理解为 Java 官方版的“数据类”，类似于：
Kotlin 的 data class
C# 的 record
Python 的 dataclass
```

# 2. 基本语法

```
public record Point(int x, int y) {}

等价于手写：

public final class Point extends java.lang.Record {
    private final int x;
    private final int y;

    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }
    public int x() { return x; }
    public int y() { return y; }

    @Override
    public boolean equals(Object o) { ... }

    @Override
    public int hashCode() { ... }

    @Override
    public String toString() { return "Point[x=" + x + ", y=" + y + "]"; }
}
```

# 3. 特性

```
自动生成常用方法
避免样板代码（boilerplate）。

不可变
字段默认是 private final
没有 setter
对象一旦创建，内容就不可修改。

继承限制
隐式 final，不能被继承。
默认继承 java.lang.Record。

支持泛型
public record Pair<K, V>(K key, V value) {}

可以实现接口
public record User(String name, int age) implements Comparable<User> {
    public int compareTo(User other) { return this.age - other.age; }
}
```

# 4. 构造器形式

```
紧凑构造器（Compact Constructor）
可以省略参数声明，直接用字段名，适合做校验：

public record User(String name, int age) {
    public User {
        if (age < 0) throw new IllegalArgumentException("age < 0");
    }
}

标准构造器（全参）
编译器自动生成，但可以覆盖。
```

# 5. 限制（和普通类的不同点）

```
不能显式继承其它类（除了 Record）。

字段必须在头部定义，不能再额外定义可变实例字段（但允许定义 static 字段）。

没有无参构造器，必须初始化所有字段。

不能声明 abstract 方法。
```

# 6. 使用场景

```
数据传输对象（DTO / VO）

配置参数载体

值对象（Value Object，领域驱动设计中的概念）

函数返回多值（避免写一堆小的 POJO）

例子：

public record Range(int min, int max) {}

代替传统：

public class Range {
    private final int min;
    private final int max;
    // 构造器 + getter + toString + equals + hashCode ...
}
```

# 7. 结合新语法

```
switch 模式匹配（JDK 21 完善）

record Point(int x, int y) {}
static void print(Object obj) {
    switch (obj) {
        case Point(int x, int y) -> System.out.println("Point: " + x + "," + y);
        default -> System.out.println("Other");
    }
}

var + record：简洁返回数据。

record Result(boolean success, String msg) {}
```

# 8. 适合 vs 不适合

```
适合：

纯粹的数据封装类（不可变，逻辑简单）。

DTO、VO、配置、消息、日志记录对象。

不适合：

需要可变状态的实体类。

需要复杂继承结构的类。

需要手写大量自定义方法的业务核心类。
```

# 9. 总结

```
record 是 简化不可变数据类的语法糖。

自动生成 getter、构造器、equals、hashCode、toString。

不可变、不可继承、可实现接口。

特别适合 DTO / 值对象 / 返回值容器。
```