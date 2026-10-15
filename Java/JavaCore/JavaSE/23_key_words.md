关键字是 Java 语法里"不属于你"的那部分词汇——它们被语言预先征用，你只能按规定的位置和含义使用，不能拿来命名。本篇不再一个一个地孤立背诵，而是**按语义分组**：同一组的关键字放在一张表里对照，讲清各自的作用、写法和容易踩的注意点。开头先厘清"关键字 / 保留字 / 字面量 / 上下文关键字 / 受限标识符"这五种容易混淆的身份，后面九节把 51 个关键字全部覆盖。

# 1. 关键字、保留字、字面量与上下文关键字

## 1.1 五种"长得像关键字"的东西

Java 里被"预留"的单词其实有五种身份，约束强度依次递减。区分它们不是为了应付考试，而是为了在起名字时知道**到底能不能用**：

| 类别 | 例子 | 能否作标识符 | 数量 |
|------|------|-------------|------|
| **关键字**（keyword） | `public`、`class`、`if`、`int` | 完全不能 | 共 51 个 |
| **保留字**（reserved word） | `const`、`goto` | 完全不能，且从未被使用 | 2 个，已含在 51 内 |
| **字面量**（literal） | `true`、`false`、`null` | 不能 | 3 个 |
| **上下文关键字**（contextual keyword） | `record`、`sealed`、`permits`、`non-sealed` | 只在特定语法位置有特殊含义，其余位置可以 | 4 个 |
| **受限标识符**（restricted identifier） | `var`、`yield` | 不能作类型名，可以作变量名和方法名 | 2 个 |
| **特殊标识符** | `_` | Java 9 起不能单独使用；Java 22 起可作未命名变量 | 1 个 |

三个最容易记混的点：

1. **`true`/`false`/`null` 不是关键字**。JLS 把它们单列为 *BooleanLiteral* 和 *NullLiteral*。它们的效果和关键字一样（不能当标识符），但归类不同——所以"关键字有 51 个"这个说法里并不包含它们。
2. **`const`/`goto` 也不是"普通关键字"**，它们从未在任何 Java 程序里生效过，存在意义仅仅是"占位，防止你使用"。`const` 预留给了未来的常量，`goto` 预留给了未来的跳转。
3. **`var` 不是关键字**，它是"保留类型名"——`int var = 1;` 完全合法，`var x = 1;` 才是它的特殊用法。这条规则在 JDK 10 引入 `var` 时特意这么设计，就是为了**不破坏已有代码**（早期代码里叫 `var` 的变量太多了）。

## 1.2 分类总览

| 语义分组 | 关键字 | 本篇章节 |
|----------|--------|----------|
| 访问控制 | `public` `protected` `private` | 第 2 节 |
| 类、接口与继承 | `class` `interface` `enum` `extends` `implements` `abstract` `final` `static` `strictfp` | 第 3 节 |
| 对象与实例 | `new` `this` `super` `instanceof` `void` `null` | 第 4 节 |
| 包与导入 | `package` `import` | 第 5 节 |
| 流程控制 | `if` `else` `switch` `case` `default` `for` `while` `do` `break` `continue` `return` `yield` | 第 6 节 |
| 异常处理 | `try` `catch` `finally` `throw` `throws` `assert` | 第 7 节 |
| 并发与运行环境 | `synchronized` `volatile` `transient` `native` | 第 8 节 |
| 基本数据类型 | `byte` `short` `int` `long` `float` `double` `char` `boolean` `true` `false` | 第 9 节 |
| 未使用与特殊标识符 | `goto` `const` `_` `var` | 第 10 节 |

# 2. 访问控制

访问控制修饰符决定"谁能看见这个成员"，是封装的第一道防线。

| 关键字 | 作用 | 示例 | 注意点 |
|--------|------|------|--------|
| `public` | 表示成员变量或方法的访问权限对**所有类**开放 | `public class A {}`、`public void f() {}` | 一个源文件里**最多只能有一个 public 类**，且文件名必须与它同名 |
| `protected` | 表示成员变量或方法的访问权限限于**当前类、子类或同一包内**的类 | `protected void f() {}` | 跨包的子类只能通过"继承"访问（`this.f()`），**不能通过父类引用访问** |
| `private` | 表示成员变量或方法的访问权限**仅限于当前类内部** | `private int age;` | 构造器也可以是 `private`，这正是单例模式的实现手法；类本身不能是 `private`（内部类除外） |

四种访问级别排在一起才看得清楚：

| 修饰符 | 本类 | 同包其他类 | 子类（不同包） | 任意类 |
|--------|------|-----------|---------------|--------|
| `private` | ✓ | ✗ | ✗ | ✗ |
| 无（默认，也叫 package-private） | ✓ | ✓ | ✗ | ✗ |
| `protected` | ✓ | ✓ | ✓ | ✗ |
| `public` | ✓ | ✓ | ✓ | ✓ |

```java
package com.example;

public class Account {
    private   double balance;      // 只有本类能碰：外部只能走 deposit / withdraw
    protected String owner;        // 同包 + 子类可见
    String    branch;              // 默认修饰符：仅同包可见（写代码时通常省略不写）
    public    String getOwner() {  // 谁都能调
        return owner;
    }

    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("金额必须为正数");
        balance += amount;         // 余额的修改被"关"在类里，这就是封装
    }
}
```

> **最佳实践**：**默认从 `private` 开始写**，只有当确实需要更大的可见范围时才逐步放宽到包级、`protected`、`public`。字段几乎总是 `private`，对外暴露行为（方法）而不是状态。反过来"字段全 public"的类等于没有封装，任何一处改动都可能破坏别的代码。

# 3. 类、接口与继承

这一组是 Java 类型系统的骨架，也是数量最多的一组。

| 关键字 | 作用 | 示例 | 注意点 |
|--------|------|------|--------|
| `class` | 用于**声明一个类** | `public class User { }` | 类名用大驼峰 |
| `interface` | 用于声明接口。接口是类的一种特殊类型，**定义了类应实现的方法但没有具体实现** | `public interface Flyable { void fly(); }` | Java 8 起可以有 `default` / `static` 方法；接口中的字段默认是 `public static final` |
| `enum` | 用于定义枚举类型，枚举是一组常量的集合 | 见下方代码 | 构造器只能是 `private`；常量列表后声明了成员就必须加分号 |
| `extends` | 用于继承类，表示当前类是另一个类的**子类** | `class Dog extends Animal { }` | Java 类只能单继承；`extends` 也可以用于泛型上界 `<? extends Number>` |
| `implements` | 用于实现接口，表示当前类实现某个接口的所有方法 | `class Bird implements Flyable { }` | 可以一次实现多个接口，用逗号分隔 |
| `abstract` | 用于声明一个抽象类，或者定义一个抽象方法（方法没有实现） | `abstract class Shape { abstract double area(); }` | **有抽象方法的类必须是抽象类**；抽象类不能 `new`；抽象方法不能是 `private`/`static`/`final` |
| `final` | 定义常量，表示不可变的值；用于方法表示该方法不能被重写；用于类表示该类不能被继承 | `final int MAX = 10;`、`final void f() {}`、`final class A {}` | `final` 修饰引用类型时锁的是"引用不可改"，**对象内容仍可改**（`final List` 依然能 `add`） |
| `static` | 表示该成员**属于类，而不是类的实例** | `static int count;`、`static void f() {}` | 静态方法里没有 `this`；静态方法不能直接访问实例成员；`static` 代码块在类加载时执行一次 |
| `strictfp` | 用于限制浮点运算的精度，保证不同平台上浮点数运算结果的一致性。用于类、接口、函数上，确保所有浮点数的计算遵循严格的标准（IEEE 754），这样无论在哪个平台上运行，浮点运算的结果都是一致的 | `strictfp class Calculator { }` | **Java 17 起所有浮点运算都已是严格语义**，这个关键字不再有实际效果（JDK 17 之后编译器会给出"过时"提示） |
| `record` | 声明一个不可变的**数据载体类**（JDK 16 正式） | `public record Point(int x, int y) { }` | 自动生成全参构造器、访问器、`equals`/`hashCode`/`toString`；**隐式 `final`，不能继承也不能被继承** |
| `sealed` | 声明一个**密封**的类或接口，限制谁能继承它 | `sealed interface Shape permits Circle, Square { }` | 子类必须出现在 `permits` 列表里，且必须是 `final`、`sealed` 或 `non-sealed` |
| `permits` | 与 `sealed` 配对，明确列出被允许继承的**直接子类** | 同上 | 如果所有子类都和父类在**同一个源文件**里，可以省略 `permits` |
| `non-sealed` | 声明一个密封类的子类**重新开放继承** | `non-sealed class Square implements Shape { }` | 它是唯一带连字符的关键字；一个密封层级里必须至少留一个 `non-sealed` 或者全部 `final` |

枚举的典型写法：

```java
enum Direction {
    NORTH(0), EAST(90), SOUTH(180), WEST(270);   // 常量后跟构造器参数

    private int degree;

    Direction(int degree) {          // 构造器隐式为 private
        this.degree = degree;
    }

    public int getDegree() {
        return degree;
    }
}
```

`record` 与 `sealed` 的写法：

```java
// record：一行等价于手写一个带全参构造器、getter、equals/hashCode/toString 的 final 类
public record Point(int x, int y) { }

// sealed：把"谁能继承我"写进类型定义本身
public sealed interface Shape permits Circle, Rectangle, Triangle { }

final       class Circle    implements Shape { }   // final：到此为止，不能再继承
non-sealed  class Rectangle implements Shape { }   // non-sealed：重新开放，谁都能继承
sealed      class Triangle  implements Shape permits IsoscelesTriangle { }   // 继续密封
```

`extends` 和 `implements` 的区别：

| 维度 | `extends` | `implements` |
|------|-----------|--------------|
| 面向对象关系 | 继承父类（"**是一个**"，is-a） | 实现接口（"**能做什么**"，can-do） |
| 数量限制 | 类只能有一个父类；接口可以 `extends` 多个接口 | 可以一次实现多个接口 |
| 必须实现的方法 | 只继承已有的具体方法，抽象方法必须实现 | 接口里的抽象方法都必须实现 |
| 能否访问父类实现 | 能，通过 `super` 复用父类代码 | 只能复用 `default` 方法 |

`abstract` 与 `interface` 的选择：

| 维度 | 抽象类 `abstract class` | 接口 `interface` |
|------|------------------------|-----------------|
| 能否有构造器 | 有（供子类 `super()` 调用） | 没有 |
| 能否有实例字段 | 有 | 没有（字段都是 `public static final`） |
| 方法实现 | 可以有具体方法 | Java 8 起可以有 `default` / `static` 方法 |
| 继承数量 | 单继承 | 多实现 |
| 设计意图 | 复用**共性的代码**（模板方法模式） | 约定**能力契约**（策略、回调） |
| 何时选 | 多个类共享状态和实现 | 只想约定"必须有哪些方法" |

> **注意**：`final` 修饰引用类型时，**锁住的是引用本身，不是对象内容**。`final List<String> list = new ArrayList<>();` 之后不能再 `list = ...`，但 `list.add("x")` 完全合法。想要真正的不可变，得用 `List.of(...)` 这类不可变集合，或者在 `record` 里只放不可变字段。

# 4. 对象与实例

这一组关键字处理"对象从哪来、指向谁、属于哪一层"。

| 关键字 | 作用 | 示例 | 注意点 |
|--------|------|------|--------|
| `new` | 用于创建对象或数组 | `new ArrayList<>()`、`new int[10]` | 每次都开辟新对象；字符串字面量 `"abc"` 走常量池，`new String("abc")` 会额外造一个对象 |
| `this` | 指代**当前对象的引用** | `this.name = name;` | 静态方法里没有 `this`；`this(...)` 可调用本类其他构造器，但**必须写在构造器第一行** |
| `super` | 用于访问父类的成员变量、方法或构造方法 | `super.hello();` | `super(...)` 调用父类构造器，也必须在第一行，因此**与 `this(...)` 不能同时出现** |
| `instanceof` | 用于判断一个对象是否是某个类或接口的实例 | `obj instanceof String` | 左边为 `null` 时**恒返回 `false`，不抛异常**；Java 16 起支持模式匹配 `obj instanceof String s` |
| `void` | 用于方法声明，表示该方法**没有返回值** | `public void run() { }` | `void` 是关键字，但 `Void` 是一个包装类（`Void.class` 常用于泛型和反射） |
| `null` | 用于表示**空引用** | `String s = null;` | 它是**字面量不是关键字**；只能赋给引用类型；`null instanceof X` 恒为 `false` |

`this` 和 `super` 的完整对照：

```java
class Parent {
    protected String tag = "父类的 tag";

    Parent()           { System.out.println("父类无参构造器"); }
    Parent(String tag) { System.out.println("父类有参构造器：" + tag); }

    void hello()       { System.out.println("父类的方法"); }
}

class Child extends Parent {
    private String tag = "子类的 tag";

    Child() {
        super("子类传进去的");     // 1. 调用父类构造器，必须是第一行
        super.hello();            // 2. 调用父类被覆盖的方法
        System.out.println(super.tag);   // 3. 访问父类的字段：父类的 tag
        System.out.println(this.tag);    // 4. 访问本类的字段：子类的 tag
    }

    @Override
    void hello() { System.out.println("子类覆盖后的方法"); }
}

// new Child() 的输出：
// 父类有参构造器：子类传进去的
// 父类的方法
// 父类的 tag
// 子类的 tag
```

`instanceof` 的用法，尤其是 Java 16 之后的模式匹配：

```java
public class InstanceofDemo {
    public static void main(String[] args) {
        Object o = "hello";

        // 旧写法：判断与强转分开，中间还要再声明一次变量
        if (o instanceof String) {
            String s = (String) o;
            System.out.println(s.length());     // 5
        }

        // Java 16+ 模式匹配：判断成功即完成绑定，s 作用域自动限定在 if 块内
        if (o instanceof String s) {
            System.out.println(s.length());     // 5
        }

        System.out.println(o instanceof Object);          // true
        System.out.println(null instanceof Object);       // false：null 恒为 false，不抛异常
    }
}
```

> **陷阱**：`this(...)` 和 `super(...)` **都必须在构造器的第一行**，所以同一个构造器里只能出现其中之一。如果既想复用本类另一个构造器、又要传参给父类，只能让被调用的那个构造器去 `super(...)`。另外，`super.field` 只能访问父类的**字段**，不能像 C++ 那样写 `super.super.method()`——Java 没有跳过一层的语法。

# 5. 包与导入

包（package）解决的是**类的命名冲突与访问边界**，导入（import）只是让代码少写几个前缀。

| 关键字 | 作用 | 示例 | 注意点 |
|--------|------|------|--------|
| `package` | 用于定义一个类、接口或枚举的**包名** | `package com.example.demo;` | 必须是源文件的**第一条非注释语句**；包名全小写、域名倒写 |
| `import` | 用于导入其他包中的类或接口 | `import java.util.List;` | 必须写在 `package` 之后、类定义之前；`java.lang` 包**自动导入**，不用写 |

三种导入写法：

```java
package com.example.demo;              // 包声明：必须是第一句

import java.util.List;                 // 1. 导入单个类
import java.util.*;                    // 2. 通配符导入：只导入本包，不含子包
import static java.lang.Math.PI;       // 3. 静态导入：之后可以直接写 PI 而不是 Math.PI

public class ImportDemo {
    public static void main(String[] args) {
        List<String> list = new java.util.ArrayList<>();   // 不导入时的全限定名写法
        System.out.println(PI);        // 静态导入的效果：3.141592653589793
    }
}
```

> **注意**：`import java.util.*;` **不会**导入 `java.util.concurrent` 这类子包，每个包都要单独导入。另外两个容易忽略的点：如果两个包里有**同名类**（比如 `java.util.Date` 和 `java.sql.Date`），通配符导入会导致编译错误，此时必须至少有一个用全限定名；`import` 只是**编译期的语法糖**，不影响运行，也不会增加 `.class` 的体积。

# 6. 流程控制

分支与循环共 11 个关键字，加上 Java 14 引入的 `yield`。

| 关键字 | 作用 | 示例 | 注意点 |
|--------|------|------|--------|
| `if` | 用于判断条件是否成立，若成立执行指定的代码块 | `if (x > 0) { }` | 条件必须是 `boolean`，**不能写 `if (x = 1)`**（那也不会通过编译） |
| `else` | 与 `if` 语句结合，表示 `if` 条件不成立时执行的代码块 | `if (a) { } else { }` | 建议总是加花括号，避免悬空 `else` 问题 |
| `switch` | 用于选择多个分支之一，通常与 `case` 一起使用 | `switch (day) { }` | 支持 `byte`/`short`/`char`/`int`/枚举/`String`（Java 7+）；Java 14 起有箭头形式的 switch 表达式 |
| `case` | 用于 `switch` 语句中，表示不同的条件分支 | `case 1 -> ...` | 传统冒号形式会**贯穿（fall-through）**，要记得 `break`；箭头形式不会 |
| `default` | 用于 `switch` 语句中的默认分支，若没有匹配的 `case`，则执行 `default` | `default -> ...` | 还有第二个含义：接口里的**默认方法** `default void f() { }` |
| `for` | 用于定义 `for` 循环 | `for (int i = 0; i < n; i++) { }` | 增强 `for` 遍历集合时**不能删元素**，否则抛 `ConcurrentModificationException` |
| `while` | 用于定义一个 `while` 循环，只有条件满足时才执行循环体 | `while (n < 3) { }` | 先判断再执行，可能一次都不执行 |
| `do` | 用于 `do-while` 循环，**先执行循环体，再判断条件** | `do { } while (n < 3);` | **至少执行一次**；结尾那个分号不能省 |
| `break` | 退出循环或 `switch` 语句块 | `if (i == 3) break;` | 带标签 `break outer;` 可以跳出多层循环 |
| `continue` | 跳过当前循环的剩余部分，继续下一次循环 | `if (i == 1) continue;` | 同样支持标签 `continue outer;`，用于跳过多层循环的本次迭代 |
| `return` | 用于从方法中返回值或控制流 | `return sum;`、`return;` | `return` 在 `finally` 块里会**覆盖掉 try 里的返回值**，不要这么写 |
| `yield` | 在 **switch 表达式的代码块分支**里交出一个值 | `default -> { yield 42; }` | 它是受限标识符不是关键字；只用于 switch 表达式，不用于普通循环 |

```java
public class FlowDemo {
    public static void main(String[] args) {
        int score = 85;

        // if / else if / else
        if (score >= 90) {
            System.out.println("优秀");
        } else if (score >= 60) {
            System.out.println("及格");          // 输出这一行
        } else {
            System.out.println("不及格");
        }

        // switch 表达式（Java 14+）：箭头形式不会贯穿，也不需要 break
        String level = switch (score / 10) {
            case 10, 9 -> "A";
            case 8    -> "B";                    // 命中这里
            default   -> "C";
        };
        System.out.println(level);               // B

        // 代码块形式的分支要用 yield 交出值
        int code = switch (score / 10) {
            case 10, 9 -> 1;
            default -> {
                int base = 10;
                yield base + 3;                  // yield 把 13 作为这个分支的值返回
            }
        };
        System.out.println(code);                // 13

        // 三种循环：for 已知次数；while 可能 0 次；do-while 至少 1 次
        for (int i = 0; i < 3; i++) {
            System.out.print(i + " ");           // 0 1 2
        }
        int n = 0;
        while (n < 3) n++;
        System.out.println(n);                   // 3
        do {
            n--;
        } while (n > 3);                         // 条件一开始就不成立，但循环体已经执行过一次
        System.out.println(n);                   // 2

        // break 与 continue 的区别
        for (int i = 0; i < 5; i++) {
            if (i == 3) break;                   // 到 3 直接结束整个循环
            if (i == 1) continue;                // 1 跳过，不打印
            System.out.print(i + " ");           // 0 2
        }
    }
}
```

> **陷阱**：传统 `switch` 的冒号形式**默认会贯穿**——`case 1: doA(); case 2: doB();` 在命中 `case 1` 时会**把 `doB()` 也执行一遍**。这是 Java 里最经典的 bug 之一。新代码一律用箭头形式 `case 1 -> doA();`，它天然不贯穿，也不需要写 `break`。

# 7. 异常处理

异常处理的关键字只有六个，但 `try`/`catch`/`finally`/`throw`/`throws` 这五个名字长得太像，是初学者最容易混的地方。

| 关键字 | 作用 | 示例 | 注意点 |
|--------|------|------|--------|
| `try` | 用于定义异常处理的开始部分，捕获可能抛出的异常 | `try { ... } catch (...) { }` | 从 Java 7 起支持 try-with-resources：`try (Scanner sc = ...) { }` 自动关闭资源 |
| `catch` | 用于异常处理，表示捕获特定类型的异常 | `catch (IOException e) { }` | 多个 `catch` 必须**从子类到父类**排列；Java 7 起支持多异常捕获，省掉重复的 `catch` 块 |
| `finally` | 在异常处理过程中，`finally` 中的代码总会执行，无论是否捕获到异常 | `finally { unlock(); }` | 只有 `System.exit()` 或 JVM 崩溃才不执行；**不要在 `finally` 里 `return`**，会吞掉异常 |
| `throw` | 用于**手动抛出异常** | `throw new IllegalArgumentException("参数非法");` | 后面跟一个异常**对象**；抛出受检异常需要方法声明 `throws` |
| `throws` | 用于**声明当前方法可能抛出的异常类型** | `void read() throws IOException { }` | 写在方法签名上，声明的是**异常类型**；调用方必须处理或继续上抛 |
| `assert` | 用于调试，检查某个条件是否为 `true`，若为 `false`，抛出 `AssertionError`。一般在开发中使用，生产环境中可禁用。格式如 `assert <condition>[:<expr>]`，其中 `expr` 可以是任意类型 | `assert age >= 0 : "年龄为负：" + age;` | **默认关闭**，必须用 `java -ea` 启用；绝不能用它校验外部输入或业务规则 |

```java
public class ExceptionDemo {
    // throws：声明本方法可能抛出 ArithmeticException，交给调用方处理
    static int divide(int a, int b) throws ArithmeticException {
        if (b == 0) {
            throw new ArithmeticException("除数不能为 0");   // throw：手动抛出异常对象
        }
        return a / b;
    }

    public static void main(String[] args) {
        try {                                    // try：圈出可能出现异常的代码
            System.out.println(divide(10, 0));
        } catch (ArithmeticException e) {        // catch：捕获指定类型的异常
            System.out.println("捕获到：" + e.getMessage());   // 捕获到：除数不能为 0
        } finally {                              // finally：无论是否发生异常都会执行
            System.out.println("清理工作");                    // 清理工作
        }

        // 断言：默认被 JVM 忽略，只有加 -ea 才生效
        int age = -1;
        assert age >= 0 : "年龄不能为负数，当前值 " + age;   // -ea 下抛 AssertionError
        System.out.println("这行在 -ea 模式下根本不会执行");
    }
}
```

这段代码在两种启动方式下结果完全不同：

```bash
java ExceptionDemo        # 输出：捕获到：除数不能为 0 / 清理工作 / 这行在 -ea 模式下根本不会执行
java -ea ExceptionDemo    # 前两行同上，随后抛 AssertionError: 年龄不能为负数，当前值 -1
```

`throw` 与 `throws` 的区别：

| 维度 | `throw` | `throws` |
|------|---------|----------|
| 位置 | 方法体**内部** | 方法签名**上** |
| 后面跟什么 | 一个异常**对象** | 一个或多个异常**类型** |
| 数量 | 一次只能抛一个 | 可以声明多个，逗号分隔 |
| 作用 | 执行抛出动作 | 声明可能抛出的异常，提示调用方 |

> **最佳实践**：`finally` 的用途是**释放资源**（关流、解锁），不是放业务逻辑。**绝对不要在 `finally` 里写 `return`**——它会覆盖 `try` 块里的返回值和抛出的异常，把错误静默吞掉，这是最难排查的一类 bug。需要自动关闭资源时，优先用 try-with-resources，它比手写 `finally` 更短也更不容易漏。

# 8. 并发与运行环境

| 关键字 | 作用 | 示例 | 注意点 |
|--------|------|------|--------|
| `synchronized` | 用于实现**线程同步**，确保同一时刻只有一个线程能执行代码块 | `synchronized (lock) { }` | 锁对象必须**唯一**且不可变；静态方法的锁是 `类.class`，实例方法的锁是 `this` |
| `volatile` | 用于定义一个变量，表明该变量可能会被多个线程同时访问，因此禁止对该变量进行优化。它确保**每次读取该变量时都会从主内存中获取最新值，而不是从 CPU 缓存中获取** | `private volatile boolean running = true;` | 只保证**可见性和有序性**，**不保证原子性**——`volatile int` 上的 `i++` 依然不安全；且约等于半个 `synchronized` 的开销 |
| `transient` | 表示成员变量**不会被序列化** | `private transient String password;` | 反序列化后该字段变成默认值（`null` / `0` / `false`）；只对 `Serializable` 生效，对 `static` 字段无意义（静态字段本来就不序列化） |
| `native` | 用于声明**本地方法**，这些方法由 Java 以外的语言（如 C/C++）编写 | `public native int hashcode();` | 只有方法声明没有方法体（结尾是分号）；实现由 JNI 或 JNA 提供，`Object.hashCode()`、`System.currentTimeMillis()` 都是 `native` 方法 |

```java
public class VolatileCounter {
    private volatile int count = 0;     // volatile 只保证读到的是最新值

    // count++ 不是原子操作（读取→加一→写回），volatile 挡不住并发覆盖
    public void increment() {
        count++;
    }

    // 想保证原子性，有三种选择：
    // 1) public synchronized void increment() { count++; }
    // 2) private final AtomicInteger count = new AtomicInteger(); count.incrementAndGet();
    // 3) 用 LongAdder（高并发下吞吐更好）
}
```

`volatile` 的正确用途是**状态标志位**，而不是计数器：

```java
public class VolatileFlag {
    private volatile boolean running = true;   // 一个线程写，其他线程读

    public void stop()  { running = false; }   // 写：立刻对其他线程可见

    public void work() {
        while (running) {                      // 读：每次从主内存取，不会读到缓存里的旧值
            // 干活
        }
        System.out.println("收到停止信号，退出循环");
    }
}
```

> **陷阱**：`volatile` 和 `synchronized` 完全不是一回事。`synchronized` 同时保证**原子性、可见性、有序性**；`volatile` 只保证**可见性和有序性**。`volatile int count` 上的 `count++` 在多线程下照样会算错——这是"背过 volatile 定义但写错代码"的典型场景。真正需要计数就用 `AtomicInteger`。

# 9. 基本数据类型

八个基本类型的关键字全部在这里。它们的字节数在 Java 里是**固定的**，不随操作系统变化。

| 关键字 | 作用 | 字节数 / 位数 | 取值范围 |
|--------|------|--------------|----------|
| `byte` | 定义一个 8 位的整数变量 | 1 字节 / 8 位 | -128 ~ 127 |
| `short` | 定义一个 16 位的短整数变量 | 2 字节 / 16 位 | -32768 ~ 32767 |
| `int` | 定义一个 32 位的整数变量 | 4 字节 / 32 位 | -2147483648 ~ 2147483647 |
| `long` | 定义一个 64 位的长整数变量 | 8 字节 / 64 位 | -9223372036854775808 ~ 9223372036854775807 |
| `float` | 定义一个 32 位的**单精度**浮点数变量 | 4 字节 / 32 位 | 约 ±3.4E-38 ~ ±3.4E+38，7 位有效数字 |
| `double` | 定义一个 64 位的**双精度**浮点数变量 | 8 字节 / 64 位 | 约 ±4.9E-324 ~ ±1.7E+308，15~16 位有效数字 |
| `char` | 定义一个字符类型的变量，占 16 位，用于表示单个字符 | 2 字节 / 16 位 | 0 ~ 65535（**无符号**，对应 Unicode 码点） |
| `boolean` | 定义一个布尔类型的变量，可以取 `true` 或 `false` | 1 字节（JVM 未明确规定） | `true` / `false` |

字面量补充（它们**不是关键字**，但通常和上表一起记）：

| 单词 | 实际身份 | 说明 |
|------|----------|------|
| `true` / `false` | 布尔字面量 | 只有这两个取值；`boolean` 不与任何数值类型互转 |
| `null` | 空字面量 | 唯一没有类型的字面量，只能赋给引用类型 |

```java
public class BasicTypeDemo {
    public static void main(String[] args) {
        byte    b = 127;                // 上限，写成 128 直接编译失败
        short   s = 32767;
        int     i = 100;
        long    l = 100L;               // long 字面量必须加 L（大写，别用 l，与数字 1 太像）
        float   f = 3.14F;              // float 字面量必须加 F
        double  d = 3.14;
        char    c = 'A';                // 本质是 0~65535 的无符号整数
        boolean flag = true;

        System.out.println((int) c);            // 65：char 参与运算会先提升为 int
        System.out.println((char) (c + 1));     // B：再强转回 char
        System.out.println(Integer.MAX_VALUE);  // 2147483647
        System.out.println(Byte.MIN_VALUE);     // -128
    }
}
```

> **注意**：**所有数值类型都是有符号的**（除 `char`），Java 没有"无符号整数类型"这一说。需要处理无符号数时用 JDK 8 引入的 API：`Integer.parseUnsignedInt()`、`Long.toUnsignedString()`、`Byte.toUnsignedInt()` 等。另外 `char` 是唯一无符号的基本类型，这也解释了为什么 `char c = -1;` 会直接编译失败。

# 10. 未使用的保留字与特殊标识符

| 单词 | 身份 | 说明 |
|------|------|------|
| `goto` | **保留关键字**，Java 中未使用 | 预留给了未来的跳转语句；Java 只保留了带标签的 `break` / `continue` 作为受限替代 |
| `const` | **保留关键字**，Java 中并未使用，通常表示常量 | 预留给了未来的常量声明；Java 用 `final`（实例常量）和 `static final`（类常量）代替 |
| `_` | 特殊标识符 | Java 9 起**单独的下划线不能再作标识符**；Java 22 起可作为**未命名变量**占位符（JEP 456 正式定稿） |
| `var` | **保留类型名**，不是关键字 | JDK 10 引入的局部变量类型推断；`int var = 1;` 合法，但 `class var {}` 不合法 |
| `yield` | **受限标识符** | 用于 switch 表达式代码块中交出值；可以作变量名和方法名，但不能作类型名 |
| `record` / `sealed` / `permits` / `non-sealed` | **上下文关键字** | 只在自己的语法位置有特殊含义，放在别处仍可作普通标识符（不过强烈不建议这么用） |

```java
public class SpecialIdentifier {
    public static void main(String[] args) {
        // var：局部变量类型推断，编译器根据右侧表达式推断类型
        var list = new java.util.ArrayList<String>();   // 推断为 ArrayList<String>
        var name = "hello";                             // 推断为 String
        var num  = 100;                                 // 推断为 int
        System.out.println(name + num);                 // hello100

        // var 的限制：只能用于局部变量，必须有初始值，不能初始化为 null
        // var v;                 // ✗ 编译错误：需要初始值
        // var v = null;          // ✗ 编译错误：无法推断类型
        // 字段、方法参数、方法返回值都不能用 var

        // int var = 1;           // ✓ 合法！var 不是关键字，只是"保留类型名"
        // class var { }          // ✗ 编译错误：var 不能作为类型名

        // Java 22 起，_ 作为未命名变量：明确表示"这个值我不关心"
        try {
            throw new RuntimeException("演示用");
        } catch (Exception _) {          // 不关心异常对象，连变量名都省了
            System.out.println("捕获但不用这个异常对象");
        }
    }
}
```

> **注意**：`var` 是"**语法糖而非动态类型**"。`var x = 1;` 中的 `x` 依然是 `int`，只是不用手写类型；它是**编译期**推断，运行时毫无影响。用 `var` 的原则是"**右侧已经能一眼看出类型时**才用"——`var list = new ArrayList<String>()` 很清楚，`var result = service.process()` 反而降低了可读性，因为读者不知道 `result` 是什么。

# 11. 小结

| 分组 | 关键字 | 一句话要点 |
|------|--------|-----------|
| 身份辨析 | —— | 关键字 51 个；`const`/`goto` 是**保留未用**的；`true`/`false`/`null` 是**字面量**；`var`/`yield` 是**受限标识符**；`record`/`sealed`/`permits`/`non-sealed` 是**上下文关键字** |
| 访问控制 | `public` `protected` `private` | 四级可见性从宽到窄，**默认从 `private` 起写**；跨包子类不能用父类引用访问 `protected` 成员 |
| 类与继承 | `class` `interface` `enum` `extends` `implements` `abstract` `final` `static` `strictfp` `record` `sealed` `permits` `non-sealed` | `extends` 是"是一个"，`implements` 是"能做什么"；`final` 锁引用不锁内容；`strictfp` 在 Java 17 后已无实际作用 |
| 对象与实例 | `new` `this` `super` `instanceof` `void` `null` | `this(...)` 与 `super(...)` 都必须在构造器第一行，**二者只能择一**；`instanceof` 支持模式匹配 |
| 包与导入 | `package` `import` | `package` 是第一条语句；`java.lang` 自动导入；`import` 只导入本包不含子包 |
| 流程控制 | `if` `else` `switch` `case` `default` `for` `while` `do` `break` `continue` `return` `yield` | 冒号形式的 `switch` **会贯穿**；`do-while` 至少执行一次；`yield` 只用于 switch 表达式的代码块分支 |
| 异常处理 | `try` `catch` `finally` `throw` `throws` `assert` | `throw` 抛对象、`throws` 声明类型；**不要在 `finally` 里 `return`**；`assert` 默认关闭需 `-ea` |
| 并发与环境 | `synchronized` `volatile` `transient` `native` | `volatile` 只保证可见性**不保证原子性**；`transient` 字段不参与序列化；`native` 方法由 C/C++ 实现 |
| 基本类型 | `byte` `short` `int` `long` `float` `double` `char` `boolean` | 字节数固定、全部有符号（`char` 除外）；`long` 加 `L`、`float` 加 `F` |
| 特殊标识符 | `goto` `const` `_` `var` | `goto`/`const` 永不可用；`_` 自 Java 9 起被禁用、Java 22 起可作未命名变量；`var` 是编译期的类型推断，不是动态类型 |

下一篇（第 24 篇）换一个视角，把 `java.lang` 这个"每个 Java 程序都默认在用、却很少有人系统整理过"的包做一份速查手册。
