# 1.概述

```
enum 是 Java 语言从 Java 5 引入的有限离散常量集合的类型（语法糖）。
本质上每个 enum 常量是该 enum 类型的单例对象实例（类的静态 final 实例）。
enum 在 JVM 层面继承自 java.lang.Enum：public enum Color extends java.lang.Enum<Color>（编译器自动处理）。
```

# 2.示例

```java
public enum Color {
    RED, GREEN, BLUE; // 默认调用无参构造器
}
```

# 3.构造器、字段和方法（实例化细节）

```java
public enum Planet {
    MERCURY(3.303e+23, 2.4397e6),
    VENUS  (4.869e+24, 6.0518e6); // 常量列表后必须加分号

    private final double mass;
    private final double radius;

    // 构造器隐式为 private，不能用 public / protected
    Planet(double mass, double radius) {
        this.mass = mass;
        this.radius = radius;
    }

    public double mass() { return mass; }
    public double radius() { return radius; }
}
```

## 要点

```
* 构造器 隐式为 private（不能用 public / protected）。
* 每个常量实际上是通过调用对应构造器创建的实例（只在类初始化时创建一次）。
* 如果声明字段或方法，常量列表后必须加分号（;）。
```

# 4.常量专有行为（Constant-specific class body）

```
可以为某个常量定义专属实现，实现策略/多态：
```

```java
public enum Operation {
    PLUS { 
        @Override
        double apply(double x,double y){
            return x+y;
        } 
    },
    MINUS { 
        @Override
        double apply(double x,double y){
            return x-y;
        } 
    };

    abstract double apply(double x,double y);
}
// 每个带类体的常量会被编译为该 enum 的匿名子类实例。
```

# 5.常用 API（必须熟悉的方法）

```
values()：返回 enum 常量数组（编译器自动生成）。
for(Color c : Color.values()) { ... }

注意：每次调用 values() 都会返回新数组（clone()），可以安全用于迭代，但频繁调用会产生新对象。
```