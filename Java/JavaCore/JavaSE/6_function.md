方法（Method）是程序中最小的可复用执行单元，它把"完成某个功能"的若干条语句打包并命名，从此调用者只需要关心**做什么**，而不必关心**怎么做**。本篇从方法为什么存在讲起，依次拆开定义格式、栈帧层面的调用过程、**参数传递（Java 只有值传递）**这一高频面试点，再讲重载、可变参数与递归，最后收束二维数组这种"数组的数组"。其中**值传递**和**重载的判定规则**是最需要彻底弄清的两处。

# 1. 方法是什么，为什么需要它

## 1.1 定义与定位

**方法（Method）** 是完成特定功能的代码块，在很多编程语言里也叫**函数（Function）**。Java 里没有独立存在的函数——所有方法都必须写在类或接口内部，因此它是隶属于对象或类的行为，而不是一段游离的代码。

把代码封装成方法的两个直接目的：

1. **实现代码的模块化**，让一段逻辑有名字、有边界，便于组织和管理；
2. **实现代码的重用性**，同一段逻辑不必在多处重复编写。

这两条听起来朴素，但它们是整个软件工程的地基。没有方法，程序就退化成一坨从上往下执行的长流水账：想改一处逻辑，得在所有复制粘贴过的地方都改一遍，漏掉一处就是 bug。

## 1.2 方法、函数与"最小执行单元"

"最小执行单元"这个说法指的是：**方法内部的语句不再单独对外暴露，外界只能整体调用它**。一段代码一旦成为方法，它就有了三重身份——一个名字（调用入口）、一组输入（参数）、一个输出（返回值）。这三样东西合起来就是所谓的**接口（Interface）**，也是后面讲面向对象封装的雏形。

> **设计考量**：判断"要不要抽一个方法"有一个实用标准——如果一段代码你**需要写注释才能说清它在干什么**，那它就应该变成一个名字自解释的方法。方法名本身就是最好的注释。

# 2. 方法的定义格式

## 2.1 完整格式

```java
修饰符 返回值类型 方法名(参数列表) {
    方法体语句;
    return 返回值;      // 返回值类型为 void 时可以省略
}
```

一个最完整的例子：

```java
public class MethodFormat {
    /**
     * 计算两个整数的和
     *
     * @param a 第一个加数
     * @param b 第二个加数
     * @return 两数之和
     */
    public static int add(int a, int b) {
        int sum = a + b;
        return sum;          // 把结果交还给调用者
    }

    public static void main(String[] args) {
        int result = add(3, 5);          // 用同类型的变量接收返回值
        System.out.println(result);      // 8
    }
}
```

逐部分拆解：

| 组成部分 | 英文名 | 说明 |
|----------|--------|------|
| 修饰符 | Modifier | `public` / `private` / `static` / `final` 等，控制访问权限和归属，可有可无 |
| 返回值类型 | Return Type | 方法执行完毕后交还给调用者的数据类型；不返回任何值时必须写 `void` |
| 方法名 | Method Name | 见名知意，小驼峰命名（`getMaxScore`），是调用的唯一入口 |
| 参数列表 | Parameters | `(类型 形参名, 类型 形参名)`，调用时按顺序传入**实参**；没有参数时写空括号 `()` 不能省略 |
| 方法体 | Method Body | 大括号内的语句序列，方法真正干活的地方 |
| `return` | Return Statement | 交出返回值并**立即结束方法** |

## 2.2 返回值类型与 `void`

**返回类型**指定了方法执行完毕后返回的数据类型。这里有三条容易踩的规则：

- 声明了具体类型，就**必须**有 `return` 带出该类型（或其可自动转换的子类型）的值，所有分支都要覆盖到；
- 声明为 `void`，表示**不返回任何数据**，此时可以不写 `return`；
- `void` 方法里也可以写 `return;`——**不带值的 `return;`** 只用来提前结束方法（比如参数校验失败时直接返回）。

```java
public class VoidDemo {
    // void 方法：不返回数据，但可以提前 return 结束
    public static void printPositive(int n) {
        if (n <= 0) {
            System.out.println("参数非法，提前结束");
            return;                  // 不带值的 return，只是"到此为止"
        }
        System.out.println("正数：" + n);
    }

    public static void main(String[] args) {
        printPositive(-5);   // 参数非法，提前结束
        printPositive(7);    // 正数：7
    }
}
```

> **陷阱**：`void` 方法里写 `return 值;` 是编译错误——既然说好了不返回任何东西，就不能返回。`void` 只允许 `return;` 这一种形式。

## 2.3 `return` 的两条铁律

**第一，`return` 一旦执行，方法立即结束。** `return` 语句之后的任何代码都是**不可达代码（Unreachable Code）**，编译器会直接报错（唯一例外是 `finally` 块中的代码，它无论如何都会执行，见第 14 篇）。

```java
public static int bad() {
    return 1;
    System.out.println("永远不会执行");   // 编译错误：Unreachable statement
}
```

**第二，有返回值的方法必须在所有可能的执行路径上都 `return`。** 漏掉任何一条分支，编译器同样不放过：

```java
// ✗ 编译错误：缺少 return 语句
public static int badBranch(int n) {
    if (n > 0) {
        return 1;
    }
    // n <= 0 时没有返回值，编译器直接拒绝
}

// ✓ 补上兜底分支即可
public static int goodBranch(int n) {
    if (n > 0) {
        return 1;
    }
    return 0;
}
```

> **最佳实践**：有返回值的方法尽量**只有一个出口**，让流程清晰；只有"参数校验失败要提前退出"这类场景才用多个 `return`。另外，`finally` 中不要写 `return`——它会覆盖 `try` 里的返回值并吞掉异常，是极难排查的坑。

## 2.4 方法不能嵌套定义

**Java 不允许在方法内部再定义一个方法。** 类的方法只能写在类的成员位置，方法体里可以**调用**别的方法，但不能**定义**方法。

```java
// ✗ 编译错误：方法里不能再定义方法
public static void outer() {
    public static void inner() { }   // 不允许
}

// ✓ 正确：平级定义，互相调用
public static void outer() {
    inner();                          // 调用是允许的
}
public static void inner() { }
```

> **注意**：这条限制的作用是配合"先声明、后使用"的编译模型——JVM 在加载类时一次性确定所有方法的签名与位置，嵌套定义会让方法的作用域和访问规则变得异常复杂。需要"函数里套函数"的效果时，用**内部类 + 方法**或 **Lambda 表达式**代替（见第 10 篇、第 13 篇）。

## 2.5 方法的调用格式

```java
方法名(实际参数列表);
```

调用的四种典型写法：

| 场景 | 写法 | 说明 |
|------|------|------|
| 有返回值，需要接收 | `int r = add(3, 5);` | 用**同类型变量**接收，最常用 |
| 有返回值，只做副作用 | `add(3, 5);` | 合法但浪费，除非方法本身有打印等副作用 |
| 直接参与运算 | `System.out.println(add(3, 5));` | 返回值直接当表达式用，可以继续参与运算 |
| 无返回值 | `printPositive(7);` | `void` 方法只能当语句调用，不能赋值给变量 |

**调用点必须提供与形参列表匹配的实参**——个数、类型、顺序都要对得上（可变参数和自动类型转换是仅有的两个例外，见第 5、6 节）。

# 3. 方法调用的底层原理：栈帧

## 3.1 虚拟机栈与栈帧

方法不只是"写在源码里的代码"，运行时它还需要一块内存来存放**局部变量、参数、返回地址**。这块内存就是**虚拟机栈（VM Stack）** 上的**栈帧（Stack Frame）**。

三句话概括：

1. **每个线程有自己独立的虚拟机栈**，所以方法内的局部变量天然线程安全；
2. **每调用一次方法，就为它压入一个新的栈帧**，方法返回时该栈帧被弹出并销毁；
3. 栈帧遵循**先进后出（FILO）**，因此**方法执行完毕的顺序一定和调用顺序相反**。

## 3.2 `main → methodA → methodB` 的压栈出栈过程

以 `main()` 调用 `methodA()`、`methodA()` 再调用 `methodB()` 为例，栈的变化如下：

```text
  ① main 开始            ② main 调用 A          ③ A 调用 B
  ┌──────────────┐       ┌──────────────┐       ┌──────────────┐
  │  main 栈帧    │       │  A 的栈帧     │←栈顶   │  B 的栈帧     │←栈顶
  └──────────────┘       ├──────────────┤       ├──────────────┤
                         │  main 栈帧    │       │  A 的栈帧     │
                         └──────────────┘       ├──────────────┤
                                                │  main 栈帧    │
                                                └──────────────┘

  ④ B 返回                ⑤ A 返回              ⑥ main 返回
  ┌──────────────┐       ┌──────────────┐       ┌──────────────┐
  │  A 的栈帧     │←栈顶   │  main 栈帧    │←栈顶   │              │  栈空
  ├──────────────┤       └──────────────┘       └──────────────┘
  │  main 栈帧    │         栈恢复原状             整个虚拟机栈清空
  └──────────────┘
```

对应的代码与执行轨迹：

```java
public class StackFrameDemo {
    public static void main(String[] args) {
        System.out.println("① main 开始");
        methodA();
        System.out.println("⑥ main 结束");
    }

    public static void methodA() {
        System.out.println("② 进入 methodA");
        methodB();
        System.out.println("⑤ 回到 methodA");
    }

    public static void methodB() {
        System.out.println("③ 进入 methodB");
        System.out.println("④ methodB 结束");
    }
}
```

输出：

```text
① main 开始
② 进入 methodA
③ 进入 methodB
④ methodB 结束
⑤ 回到 methodA
⑥ main 结束
```

注意 `②` 和 `⑤` 的顺序：`methodB()` 调用一行卡在 `methodA` 中间，**必须等 `methodB` 整个栈帧弹完，`methodA` 才能继续往下走**。这就是同步方法调用的本质——调用者会被"挂起"，直到被调用者返回。

## 3.3 栈溢出：为什么递归必须有出口

栈的容量是有限的（HotSpot 默认约 512KB ~ 1MB，可用 `-Xss` 调整）。如果方法一直调用下去、栈帧只压不弹，就会撑爆栈空间，抛出 **`StackOverflowError`**：

```java
public class StackOverflowDemo {
    public static void recurse(int depth) {
        if (depth % 5000 == 0) {
            System.out.println("当前递归深度：" + depth);   // 别每层都打印，否则慢得看不出来
        }
        recurse(depth + 1);        // 没有终止条件，栈帧只压不弹
    }

    public static void main(String[] args) {
        recurse(1);                // 跑到一两万层左右就崩
    }
}
```

输出（最后一行由 JVM 打印）：

```text
当前递归深度：5000
当前递归深度：10000
当前递归深度：15000
Exception in thread "main" java.lang.StackOverflowError
```

> **陷阱**：`StackOverflowError` 属于 `Error` 而不是 `Exception`，`catch (Exception e)` 抓不到它。它表示"程序已经无力回天"，正确的做法是**修掉无限递归**，而不是捕获它。

# 4. 参数传递：Java 只有值传递

这是 Java 基础里被讨论最多、也最容易讲错的一个点。先把结论放前面：**Java 中所有参数传递都是值传递（Pass by Value），没有引用传递。** 所谓"传引用类型"，传的也是**值**——只不过这个值恰好是一个地址引用。

## 4.1 传基本数据类型：复制值的副本

基本数据类型有整数类型、浮点数类型、布尔类型、字符类型。它们作为实参传入方法时，**传递的是值的一份副本**，形参和实参是两个互不相干的变量。

```java
public class PrimitivePassDemo {
    // 试图交换两个数：方法内确实换了，方法外毫无变化
    public static void swap(int a, int b) {
        int temp = a;
        a = b;
        b = temp;
        System.out.println("方法内：a = " + a + ", b = " + b);
    }

    public static void main(String[] args) {
        int x = 10;
        int y = 20;
        swap(x, y);
        System.out.println("方法外：x = " + x + ", y = " + y);
    }
}
```

输出：

```text
方法内：a = 20, b = 10
方法外：x = 10, y = 20
```

内存上发生了什么：

```text
  main 的栈帧              swap 的栈帧
  ┌──────────────┐        ┌──────────────┐
  │ x = 10       │        │ a = 10       │  ← 只是 x 的副本
  │ y = 20       │        │ b = 20       │  ← 只是 y 的副本
  └──────────────┘        └──────────────┘
         ▲                        │
         │   方法内改的是副本      │  副本随栈帧一起销毁
         └────────────────────────┘  实参 x、y 从头到尾没被碰过

  想要"交换两个 int"？只能用数组 / 包装类当容器，或直接返回新值。
  例如 int[] r = swap(x, y); 通过 r[0]、r[1] 拿结果。
```

**所以 `swap(int, int)` 这种写法在 Java 里注定失败**，这是值传递的直接推论，不是语言缺陷。

## 4.2 传引用数据类型：复制地址值

`new` 出来的都是引用数据类型。传入方法时传递的**也是值的副本**，但这个值是对象在堆内存中的**地址**。于是产生了一个关键区别：

- **顺着地址去改对象内部的字段**——改的是堆里同一份数据，方法外**能看到变化**；
- **让形参重新指向一个新对象**——只是把副本的指向改了，方法外**毫无影响**。

```java
class Student {
    String name;
    int age;

    Student(String name, int age) {
        this.name = name;
        this.age = age;
    }
}

public class ReferencePassDemo {
    // ① 顺地址改字段：能改到
    public static void changeAge(Student s) {
        s.age = 30;                   // 顺着副本里的地址找到堆中对象，改的是同一份数据
    }

    // ② 让形参指向新对象：改不到
    public static void replace(Student s) {
        s = new Student("李四", 40);   // 只是让副本换了指向，原对象不受影响
        System.out.println("方法内 s.name = " + s.name);
    }

    public static void main(String[] args) {
        Student stu = new Student("张三", 18);

        changeAge(stu);
        System.out.println("改字段后 stu.age = " + stu.age);     // 30

        replace(stu);
        System.out.println("换引用后 stu.name = " + stu.name);   // 还是张三
    }
}
```

输出：

```text
改字段后 stu.age = 30
方法内 s.name = 李四
换引用后 stu.name = 张三
```

内存视图：

```text
        栈（main）              栈（replace）              堆
  ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────────┐
  │ stu ─────────────┼──▶│ s ───────────────┼──▶│ name = "张三"         │
  └──────────────────┘   └──────────────────┘   │ age  = 30            │
        ↑ 副本也指向同一个对象 ┘                   └──────────────────────┘
                                                  ↑
        执行 s = new Student("李四", 40) 之后：
        ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
        │ stu ─────────────┼──▶│ 原来的张三对象     │   │ 新对象：李四      │
        └──────────────────┘   └──────────────────┘   └──────────────────┘
                                   ▲
        ┌──────────────────┐       │ 副本 s 改指向了新对象
        │ s ───────────────┼───────┘   （方法返回后 s 销毁，新对象成为垃圾）
        └──────────────────┘
```

> **注意**：把"修改对象字段成功"当成"Java 是引用传递"的证据，是最常见的误解。真正的引用传递（如 C++ 的 `T&`）意味着**形参和实参是同一个变量**，形参重新赋值实参也会跟着变——这在 Java 里永远做不到。

## 4.3 为什么说 Java 只有值传递

判断是值传递还是引用传递，标准只有一条：**形参的重新赋值，会不会影响实参？**

| 对比项 | 传基本类型 | 传引用类型 |
|--------|-----------|-----------|
| 实际传递的东西 | 变量中的**数值副本** | 变量中的**地址副本** |
| 方法内改形参本身 | 不影响实参 | 不影响实参 |
| 方法内顺着引用改内容 | —— | **影响实参所指向的对象** |
| 传递方式 | 值传递 | 值传递（值是地址） |
| 类比 | 复印一份文件 | 复印一把钥匙，两把钥匙开同一把锁 |

一句话总结：**Java 传递的永远是"变量里装的那个东西"的副本**，基本类型装的是数值，引用类型装的是地址，仅此而已。

> **最佳实践**：不要试图通过参数"带回"结果。要返回多个值，优先**返回对象**或 `return` 一个封装好的结果类；其次是返回数组；传入可变对象并在方法内修改它**语义不清晰**，调用者很难预判副作用，是代码评审里的减分项。

## 4.4 三个高频陷阱

**陷阱一：`String` 是不可变的。** 传 `String` 进去"拼接"，方法外看不到变化：

```java
public static void appendWorld(String s) {
    s = s + "World";        // String 不可变，这里生成的是新对象，副本改指向而已
}

public static void main(String[] args) {
    String text = "Hello";
    appendWorld(text);
    System.out.println(text);   // 仍然是 Hello，不是 HelloWorld
}
```

**陷阱二：数组是引用类型，内容可以被改掉。**

```java
public static void reset(int[] arr) {
    arr[0] = 0;             // 改内容 → 调用者能看到
}

public static void main(String[] args) {
    int[] data = {1, 2, 3};
    reset(data);
    System.out.println(data[0]);   // 0，被改掉了
}
```

**陷阱三：包装类的"赋值"同样是换引用。** `Integer`、`Double` 等包装类是不可变对象，方法里 `i++` 只是让形参指向了新的装箱对象，调用者的变量纹丝不动。

> **设计考量**：`String` 和包装类设计成不可变，正是为了让"值传递"的语义**可预测**——只要看到参数是这两类，就能断定方法不可能悄悄改掉调用者的数据。这是把不可变性当作隔离手段的典范。

# 5. 方法重载

## 5.1 什么是重载

**重载（Overloading）** 指在**同一个类中**定义多个**名称相同、参数列表不同**的方法。它让"同一个动作、不同的输入形态"共用同一个名字，调用者不必去记 `addInt`、`addDouble`、`addThree` 这一堆变体名。

判断是否构成重载，**只看方法名和参数列表**。参数列表不同包括三种情形：

| 判定依据 | 示例 | 是否构成重载 |
|----------|------|-------------|
| **参数类型**不同 | `add(int, int)` 与 `add(double, double)` | 是 |
| **参数个数**不同 | `sum(int, int)` 与 `sum(int, int, int)` | 是 |
| **参数顺序**不同 | `func(int, String)` 与 `func(String, int)` | 是，但**不建议** |

```java
public class OverloadDemo {
    public static int add(int a, int b) {
        return a + b;
    }

    public static double add(double a, double b) {   // 类型不同
        return a + b;
    }

    public static int add(int a, int b, int c) {     // 个数不同
        return a + b + c;
    }

    public static void main(String[] args) {
        System.out.println(add(1, 2));           // 3      → 匹配 add(int, int)
        System.out.println(add(1.5, 2.5));       // 4.0    → 匹配 add(double, double)
        System.out.println(add(1, 2, 3));        // 6      → 匹配 add(int, int, int)
    }
}
```

**参数顺序不同的重载**虽然编译器允许，但会让调用者一眼看不出选的是哪个，可读性极差。实践中应改用不同的方法名或把参数类型设计得更有区分度。

## 5.2 什么不算重载

**方法的返回类型、权限修饰符（`public` / `private` 等）和 `static` 关键字都不作为重载的判断依据。** 换一句话说，只改这些修饰而不动参数列表，等于**重复定义**，编译器会直接报错。

```java
// ✗ 编译错误：'add(int, int)' is already defined —— 只是返回值不同，不算重载
public static int add(int a, int b)  { return a + b; }
public static long add(int a, int b) { return a + b; }

// ✗ 编译错误：只是权限修饰符不同，不算重载
public static int sub(int a, int b)  { return a - b; }
private static int sub(int a, int b) { return a - b; }

// ✗ 编译错误：只是 static 有无不同，不算重载（且 static 与非 static 本就无法共存）
public static int mul(int a, int b) { return a * b; }
public int mul(int a, int b)        { return a * b; }
```

> **陷阱**：为什么返回值不能作为判定依据？因为**方法调用在只看得到"方法名 + 实参"的语境下就必须唯一确定要调用哪个方法**。`add(1, 2);` 这种当语句用的调用根本不关心返回值，编译器无法靠返回值来区分。同理，只有参数列表（即方法的**签名**，`方法名 + 参数类型序列`）是唯一的判别依据。

## 5.3 Java 没有默认参数，只能靠重载

C++、Python、Kotlin 都支持**默认参数**（`int add(int a, int b = 0)`），Swift 还支持**命名参数**，调用时可以跳过某些可选参数。**Java 两种都不支持**——这是 Java 语言设计上为了保持"方法签名唯一、解析简单"而做的取舍。

于是 Java 里表达"某个参数可选"只能靠**重载 + 转发**：

```java
public class LogUtils {
    // 全参版本：真正干活的方法只有一个
    public static void log(String level, String message, boolean withTime) {
        String prefix = withTime ? "[" + System.currentTimeMillis() + "] " : "";
        System.out.println(prefix + level + " " + message);
    }

    // 省略 withTime 的重载：委托给全参版本，避免逻辑重复
    public static void log(String level, String message) {
        log(level, message, false);
    }

    // 再省略 level 的重载
    public static void log(String message) {
        log("INFO", message, false);
    }

    public static void main(String[] args) {
        log("启动完成");
        log("WARN", "磁盘空间不足");
        log("ERROR", "写入失败", true);
    }
}
```

输出（时间戳随运行时刻变化）：

```text
INFO 启动完成
WARN 磁盘空间不足
[1730000000000] ERROR 写入失败
```

这种写法的要点是：**让所有重载最终都委托给"参数最全"的那一个**，业务逻辑只写一遍，其余重载只做参数补默认值。这样新增重载不会引入重复代码。

> **注意**：重载数量一多就会爆炸（3 个可选参数 = 8 个重载）。当可选参数变多时，正确的做法是改用**建造者模式**，或用**第 28 篇的 `Optional`** 来表达"这个参数可能没有值"，让"可空"成为类型系统的一部分，而不是靠记住有哪几个重载。

## 5.4 重载的匹配优先级

当多个重载都能勉强接受某个实参时，编译器按固定优先级挑选**最匹配**的那个：

```text
  精确匹配（类型完全一致）
        │ 不成立
        ▼
  基本类型的宽化转换（byte → short → int → long → float → double）
        │ 不成立
        ▼
  装箱 / 拆箱（int ⇄ Integer）
        │ 不成立
        ▼
  可变参数（int... 视为 int[]）
```

```java
public class OverloadResolution {
    public static void print(int i)     { System.out.println("匹配 int"); }
    public static void print(long l)    { System.out.println("匹配 long"); }
    public static void print(Integer i) { System.out.println("匹配 Integer"); }
    public static void print(int... a)  { System.out.println("匹配 可变参数"); }

    public static void main(String[] args) {
        print(10);              // 精确匹配 → int
        print(10L);             // 精确匹配 → long
        print((Integer) 10);    // 精确匹配 → Integer
        short s = 10;
        print(s);               // short 无精确匹配，宽化优于装箱 → int
        print();                // 只能落到可变参数 → 可变参数
    }
}
```

输出：

```text
匹配 int
匹配 long
匹配 Integer
匹配 int
匹配 可变参数
```

> **陷阱**：`print(10)` 在同时存在 `print(int)` 和 `print(Integer)` 时**一定**选 `print(int)`，因为宽化/精确匹配的优先级高于装箱。但反过来，如果只存在 `print(long)` 和 `print(Integer)`，传 `int` 会选 `print(long)` 而不是 `print(Integer)`——这类规则记不清就容易写出"为什么调错了方法"的困惑代码。**实操建议是避免设计这种需要靠优先级才能分辨的重载。**

# 6. 可变参数

## 6.1 语法与本质

**可变参数（Variable Arguments）** 用 `...` 表示，允许方法接受**零个或多个**指定类型的参数。定义格式：

```text
修饰符 返回值类型 方法名(其他参数, 类型... 变量名) {
    方法体
}
```

例如 `public static int sum(int... numbers)`。**它的本质是数组**——编译之后，`int... numbers` 与 `int[] numbers` 在字节码层面完全等价，编译器只是额外做了一步"把实参打包成数组"的工作。

```java
import java.util.Arrays;

public class VarArgsDemo {
    // 可变参数必须是参数列表的最后一个
    public static int sum(String label, int... numbers) {
        int total = 0;
        for (int n : numbers) {        // 方法内部 numbers 就是一个 int[]，数组特性全能用
            total += n;
        }
        System.out.println(label + "：个数 = " + numbers.length + "，内容 = "
                + Arrays.toString(numbers) + "，和 = " + total);
        return total;
    }

    public static void main(String[] args) {
        sum("不传");                       // numbers 是长度为 0 的数组，不是 null
        sum("传三个", 1, 2, 3);            // 编译器自动打包成 int[]{1, 2, 3}
        sum("传数组", new int[]{4, 5});    // 直接传一个数组也合法

        int[] arr = {7, 8, 9};
        sum("传数组变量", arr);            // 等价于把 arr 整个交进去，不再是"一个元素"
    }
}
```

输出：

```text
不传：个数 = 0，内容 = []，和 = 0
传三个：个数 = 3，内容 = [1, 2, 3]，和 = 6
传数组：个数 = 2，内容 = [4, 5]，和 = 9
传数组变量：个数 = 3，内容 = [7, 8, 9]，和 = 24
```

## 6.2 四条必须记住的规则

**规则一：可变参数必须是参数列表中的最后一个。** 这是为了消除编译器的解析歧义——如果写成 `sum(int... nums, String label)`，编译器无法判断实参从哪里开始属于可变部分。

**规则二：一个方法只能有一个可变参数。** 同理，两个可变参数会让"边界在哪"彻底无解。

**规则三：不传参数时，变量是长度为 0 的数组，而不是 `null`。** 这一点让处理更安全——直接 `for` 遍历不会 NPE。

**规则四：方法内部把它当数组用。** 可以 `length`、可以下标访问、可以 foreach，也可以整体传给另一个接收数组的方法。

```java
// ✓ 合法
public static void ok(String label, int... nums) { }

// ✗ 编译错误：可变参数不是最后一个
public static void bad1(int... nums, String label) { }

// ✗ 编译错误：一个方法只能有一个可变参数
public static void bad2(int... a, String... b) { }
```

## 6.3 与 `int[]` 的区别

| 维度 | `int... nums` | `int[] nums` |
|------|--------------|-------------|
| 调用写法 | `sum(1, 2, 3)` 或 `sum(arr)` | 必须 `sum(arr)` |
| 零参调用 | 可以 `sum()`，内部是空数组 | 不可以，`sum()` 编译错误 |
| 字节码层面 | 就是一个数组参数 | 就是一个数组参数 |
| 编译期提示 | 编译器会标注"可变参数方法" | 无特殊提示 |
| 适用场景 | 参数个数天然不定（日志、求和、`printf`） | 参数本来就是一批数据 |

`int...` **只影响调用端的写法**，方法体代码可以一字不改地在两者之间切换。

## 6.4 `null` 陷阱与重载歧义

**陷阱一：显式传 `null` 会 NPE。**

```java
sum("危险", (int[]) null);   // 编译通过，但方法内 numbers.length 立刻抛 NullPointerException
```

因为此时 `numbers` 就是 `null`，与"不传参数得到空数组"是两码事。防御性写法有两种：

```java
public static int sumSafe(String label, int... numbers) {
    if (numbers == null) {          // ① 显式判空
        numbers = new int[0];
    }
    // ② 或者更简洁：numbers = Objects.requireNonNullElse(numbers, new int[0]);
    int total = 0;
    for (int n : numbers) total += n;
    return total;
}
```

**陷阱二：与固定参数的版本同时存在时，固定参数优先。**

```java
public static void show(int a, int b)  { System.out.println("固定参数版本"); }
public static void show(int... a)      { System.out.println("可变参数版本"); }

// show(1, 2)  → 输出"固定参数版本"，因为精确匹配优先于可变参数
// show(1)     → 输出"可变参数版本"，固定参数版本接不住一个参数
```

**陷阱三：`main(String[] args)` 也可以写成 `main(String... args)`**，两者完全等价。JVM 只按名字 `main` 和参数类型 `String[]` 来找入口，可变参数形式编译后同样是 `String[]`，所以合法。

> **最佳实践**：可变参数适合"参数个数是补充性、非核心"的场景。一旦发现方法有 5 个以上业务参数还带可变参数，说明这个方法职责太重，应该拆开或改用参数对象。

# 7. 递归

## 7.1 递归的三要素

**递归（Recursion）** 是方法自己调用自己。第 12 篇从算法角度提过它，这里补齐执行层面的理解。

写对递归必须同时具备三样东西：

| 要素 | 含义 | 缺失后果 |
|------|------|----------|
| **终止条件（基准条件）** | 一个不再递归、直接返回的出口 | 无限递归 → `StackOverflowError` |
| **递推公式** | 把原问题拆成规模更小的同类子问题 | 无法靠近出口，同样栈溢出 |
| **规模递减** | 每次递归参数必须朝终止条件方向变化 | 形式上有出口但永远到不了 |

递归的底层其实就是第 3 节讲的栈帧：每一层递归都是一个新栈帧压在栈顶，直到命中出口才开始逐层弹出并返回结果。这解释了为什么递归天然**吃栈空间**，也解释了为什么"递"和"归"是两个独立的阶段。

## 7.2 示例：阶乘

```java
public class FactorialDemo {
    /**
     * 计算 n 的阶乘：n! = n × (n-1) × ... × 1
     * 递推公式：f(n) = n * f(n-1)；终止条件：f(0) = f(1) = 1
     */
    public static long factorial(int n) {
        if (n < 0) {
            throw new IllegalArgumentException("n 不能为负数：" + n);
        }
        if (n <= 1) {                    // ① 终止条件
            return 1;
        }
        return n * factorial(n - 1);     // ② 递推公式 + 规模递减
    }

    public static void main(String[] args) {
        for (int i = 0; i <= 5; i++) {
            System.out.println(i + "! = " + factorial(i));
        }
    }
}
```

输出：

```text
0! = 1
1! = 1
2! = 2
3! = 6
4! = 24
5! = 120
```

`factorial(5)` 的执行过程：

```text
  递（压栈）                                    归（出栈求值）
  factorial(5) = 5 * factorial(4)                factorial(1) → 1
  factorial(4) = 4 * factorial(3)                factorial(2) → 2 * 1   = 2
  factorial(3) = 3 * factorial(2)                factorial(3) → 3 * 2   = 6
  factorial(2) = 2 * factorial(1)                factorial(4) → 4 * 6   = 24
  factorial(1) → 命中终止条件，返回 1             factorial(5) → 5 * 24  = 120
```

## 7.3 示例：斐波那契与"重复计算"问题

```java
public class FibonacciDemo {
    // 朴素递归：代码最短，但同一子问题被反复计算，时间复杂度 O(2^n)
    public static long fibSlow(int n) {
        if (n <= 1) return n;                    // 终止条件：f(0)=0, f(1)=1
        return fibSlow(n - 1) + fibSlow(n - 2);  // 递推公式
    }

    private static long[] cache;                 // 缓存已经算过的结果

    // 记忆化递归：把时间降到 O(n)，代价是 O(n) 的额外空间
    public static long fibMemo(int n) {
        if (n <= 1) return n;
        if (cache[n] != 0) {                     // 查缓存，算过就直接返回
            return cache[n];
        }
        cache[n] = fibMemo(n - 1) + fibMemo(n - 2);
        return cache[n];
    }

    public static void main(String[] args) {
        cache = new long[100];

        long start = System.nanoTime();
        System.out.println("朴素递归 fib(40) = " + fibSlow(40));
        System.out.println("朴素递归耗时：" + (System.nanoTime() - start) / 1_000_000 + " ms");

        start = System.nanoTime();
        System.out.println("记忆化   fib(40) = " + fibMemo(40));
        System.out.println("记忆化耗时：" + (System.nanoTime() - start) / 1_000_000 + " ms");
    }
}
```

输出（耗时随机器不同，量级差异是重点）：

```text
朴素递归 fib(40) = 102334155
朴素递归耗时：468 ms
记忆化   fib(40) = 102334155
记忆化耗时：0 ms
```

**为什么朴素递归这么慢？** 计算 `fib(5)` 需要 `fib(4)` 和 `fib(3)`，而 `fib(4)` 里又要算一次 `fib(3)`——同一棵子树被反复展开。调用次数随 `n` 指数增长，`n = 50` 就已经是天文数字。**记忆化搜索（Memoization）** 用一个数组把算过的结果存下来，每个子问题只算一次，时间立刻降到线性。

## 7.4 递归 vs 迭代

| 维度 | 递归 | 迭代（循环） |
|------|------|-------------|
| 代码可读性 | 树、图、分治等结构上**更接近数学定义** | 线性问题更直观 |
| 空间开销 | 每层一个栈帧，**O(n) 栈空间** | 通常 O(1) |
| 时间开销 | 有方法调用开销，朴素写法可能指数级 | 一般更优 |
| 溢出风险 | 深度过大会 `StackOverflowError` | 不会因深度溢出 |
| 典型场景 | 树的遍历、快排、归并、回溯、分治、目录扫描 | 数组遍历、累加、线性查找 |

> **最佳实践**：能用迭代就别用递归；只有当**问题的结构本身就是递归的**（树、图、分治）时，递归带来的可读性收益才值得那点栈开销。写递归时务必确认三要素齐全，必要时把递归改成显式的栈 + 循环，或加记忆化去掉重复子问题。

# 8. 二维数组

## 8.1 本质：数组的数组

**二维数组本质上是元素为一维数组的数组**，逻辑上可以看作矩阵或表格。理解这句话的三个推论：

1. 它的每个"元素"本身是一个**一维数组对象**，所以二维数组在堆里是"一个外层数组 + 若干内层数组"的结构，**不是一块连续的内存**；
2. 因为每行都是独立创建的对象，所以**各行的长度可以不同**（不规则数组）；
3. `arr.length` 是**行数**，`arr[i].length` 是**第 i 行的列数**——两个 `length` 是两次不同的数组访问。

数组的基础语法（定义、索引、遍历、越界、`Arrays` 工具类）见第 5 篇，这里只讲二维特有的部分。

## 8.2 三种定义与初始化方式

**完整格式（静态初始化）**——定义时就把每个元素填好：

```java
int[][] matrix = new int[][]{{1, 2, 3}, {4, 5, 6}};   // 完整写法
int[][] simple = {{1, 2, 3}, {4, 5, 6}};              // 省略 new int[][]，只能用于定义处
```

**不初始化格式（动态初始化）**——只给长度，元素是默认值：

```java
int[][] regular = new int[3][4];    // 3 行 4 列的规则矩阵，元素全为 0
int[][] jagged  = new int[3][];     // 只确定 3 行，每行的列数后续单独 new
```

`new int[3][4]` 创建的是**规则的矩阵**（3 行 4 列）；`new int[3][]` 只确定了行数，内层数组需要后续逐个创建，因此可以做成**不规则**的。

**只定义格式**——只声明引用变量，未分配内存：

```java
int[][] onlyDeclared;        // 只声明了引用，直接使用会 NPE 或编译错误
```

> **注意**：`TYPE[] arr` 和 `TYPE arr[]` 两种写法 Java 都兼容，但 **`int[][] arr` 这种"方括号跟类型"的写法更规范**，因为方括号修饰的是类型而不是变量名——`int[] a, b;` 里 `a` 和 `b` 都是数组，而 `int a[], b;` 里 `b` 只是普通 `int`，容易看错。

## 8.3 元素访问与遍历

访问格式是 `array_name[row_index][col_index]`，两次下标各访问一层数组：

```java
import java.util.Arrays;

public class TwoDArrayDemo {
    public static void main(String[] args) {
        // 静态初始化：每个内部一维数组的长度可以不同
        int[][] scores = {
                {90, 85, 78},
                {88, 92},
                {60}
        };

        System.out.println("行数            = " + scores.length);        // 3
        System.out.println("第 0 行的列数   = " + scores[0].length);     // 3
        System.out.println("第 2 行的列数   = " + scores[2].length);     // 1
        System.out.println("scores[1][1]    = " + scores[1][1]);        // 92

        // 动态初始化 + 逐行创建：拼出一个直角三角形
        int[][] triangle = new int[4][];
        for (int i = 0; i < triangle.length; i++) {
            triangle[i] = new int[i + 1];        // 第 i 行有 i + 1 列
        }
        System.out.println("不规则数组 = " + Arrays.deepToString(triangle));
        // deepToString 才能打印出内层内容；直接 Arrays.toString 只会打印一长串地址

        // 遍历：外层 foreach 拿到的每个元素本身就是一维数组
        for (int[] row : scores) {
            for (int value : row) {
                System.out.print(value + " ");
            }
            System.out.println();
        }
    }
}
```

输出：

```text
行数            = 3
第 0 行的列数   = 3
第 2 行的列数   = 1
scores[1][1]    = 92
不规则数组 = [[0], [0, 0], [0, 0, 0], [0, 0, 0, 0]]
90 85 78 
88 92 
60 
```

> **陷阱**：打印二维数组必须用 `Arrays.deepToString()`，用 `Arrays.toString()` 只会得到 `[[I@1b6d3586, [I@4554617c]` 这类地址串——因为二维数组的元素是数组对象，`toString()` 只作用于最外层。

## 8.4 二维数组的内存图

```text
        栈                      堆
  ┌──────────────┐    ┌─────────────────────────────┐
  │ scores ──────┼───▶│ 外层数组（长度 3）            │
  └──────────────┘    │ [0] ─────┐                  │
                      │ [1] ───┐ │                  │
                      │ [2] ─┐ │ │                  │
                      └──────┼─┼─┼──────────────────┘
                             │ │ │
        ┌────────────────────┘ │ └───────────────┐
        ▼                      ▼                 ▼
  ┌───────────────┐   ┌───────────────┐   ┌───────────┐
  │ {90, 85, 78}  │   │ {88, 92}      │   │ {60}      │
  └───────────────┘   └───────────────┘   └───────────┘
      内层数组 0           内层数组 1         内层数组 2
      长度 3                长度 2             长度 1
```

外层数组里存的只是**三个地址**，真正的数据分布在三块独立的堆内存里。这正是"每行长度可以不同"的原因——每行都是独立 `new` 出来的。

# 9. 小结

| 主题 | 一句话要点 |
|------|-----------|
| 方法定位 | 最小的可复用执行单元，目的是模块化与重用；Java 里方法必须写在类中，且**不能嵌套定义** |
| 定义格式 | `修饰符 返回值类型 方法名(参数列表) { ... }`；`void` 不返回数据，可有裸 `return;`；`return` 之后不能再写代码 |
| 栈帧 | 每次调用压入一个栈帧、返回时弹出销毁；调用链越深栈越满，无出口的递归会 `StackOverflowError` |
| 参数传递 | **Java 只有值传递**：基本类型传数值副本，引用类型传地址副本。改字段能影响外部，改引用指向不能 |
| 重载 | 判定依据**只有方法名和参数列表**；返回值、修饰符、`static` 都不算。匹配优先级：精确 → 宽化 → 装箱 → 可变参数 |
| 默认参数 | Java 不支持默认参数和命名参数，用**重载委托给全参版本**模拟；可选参数多时改用建造者或第 28 篇的 `Optional` |
| 可变参数 | `int...` 底层就是 `int[]`，必须放最后、只能有一个；零参时是空数组而非 `null`，显式传 `null` 会 NPE |
| 递归 | 三要素：终止条件、递推公式、规模递减；注意重复计算（用记忆化）与栈溢出风险，能用迭代优先用迭代 |
| 二维数组 | 本质是**数组的数组**，每行长度可不同；`arr.length` 是行数，`arr[i].length` 是列数，打印用 `Arrays.deepToString` |

下一篇（第 7 篇）从"方法"走向"对象"，看类如何把**成员变量（数据）**和**成员方法（行为）**打包在一起，以及对象在堆内存中到底长什么样；数组本身的语法细节见第 5 篇，递归在算法层面的应用见第 12 篇。
