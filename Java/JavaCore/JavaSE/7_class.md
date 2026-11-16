上一篇讲的方法是"行为"，但行为总得依附于某个东西。**面向对象（OOP）** 解决的正是这件事：把描述同一个事物的**数据**和**行为**绑在一起，形成一个可以独立存在、独立变化的单元——**对象**。本篇按"定义类 → 创建对象 → 使用对象"这条主线走一遍，重点讲清**对象在内存里长什么样**、**构造方法与 `this` 的工作机制**、**封装到底封装了什么**，最后落到**对象什么时候会被回收**。

# 1. 面向对象：定义类、创建对象、使用对象

## 1.1 从面向过程到面向对象

**面向对象（Object Oriented Programming, OOP）** 是 Java 的核心编程范式，它的世界观是"**万物皆对象**"——现实世界中的任何事物都可以抽象为程序中的对象。

| 维度 | 面向过程 | 面向对象 |
|------|----------|----------|
| 关注点 | "**怎么做**"，按步骤一步步实现功能 | "**谁来做**"，把数据和操作数据的代码打包 |
| 组织单位 | 函数 | 类与对象 |
| 数据与行为的关系 | 分离，数据到处传来传去 | 绑定在同一个类里 |
| 适合场景 | 简单任务、算法题、脚本 | 大型项目，便于扩展与维护 |
| 典型代表 | C 语言 | Java、C++、Python |

两种范式不是对立的。写一个"两数相加"的程序，用过程式思维一行就完了；但写一个订单系统，就必须先想清楚"订单有哪些属性、能做什么动作"——这正是面向对象擅长的规模。

## 1.2 属性与行为

把一个现实事物抽象为对象时，只需要抽取两类东西：

- **属性**：对象的**静态特征**，对应类中的**成员变量**（也叫字段，field）。例如学生的姓名、年龄、学号。
- **行为**：对象的**动态特征**，对应类中的**成员方法**。例如学生的"学习"、"自我介绍"。

面向对象的三大特性是**封装、继承、多态**：封装讲"把数据藏起来、只暴露可控的入口"（本篇第 6 节）；继承与多态讲"类与类之间的关系"，见第 10 篇。

## 1.3 三步走：定义类 → 创建对象 → 使用对象

下面这个小例子会贯穿全文，后续每一节都拿它举例：

```java
// 第一步：定义类 —— 描述"学生"这一类事物
public class Student {
    // 属性（成员变量）：描述这类事物"有什么"
    String name;
    int age;

    // 行为（成员方法）：描述这类事物"能做什么"
    public void show() {
        System.out.println("我叫 " + name + "，今年 " + age + " 岁");
    }
}
```

```java
// 第二步 + 第三步：创建对象并使用它
public class StudentTest {
    public static void main(String[] args) {
        // 创建对象：类名 变量名 = new 类名();
        Student s = new Student();

        // 使用对象：对象名.成员变量 / 对象名.成员方法()
        s.name = "张三";
        s.age = 18;
        s.show();               // 我叫 张三，今年 18 岁
    }
}
```

三件事的用词必须分清：

| 概念 | 含义 | 例子 |
|------|------|------|
| **类（Class）** | 抽象的描述，是"模板"、是"图纸" | `Student` |
| **对象（Object）** | 类的实例化结果，是具体存在的实体 | `s` 指向的那个 `Student` |
| **实例化（Instantiate）** | 用类造出对象的过程，靠 `new` 完成 | `new Student()` |

`Student` 这个类本身不能"叫张三"——它只是规定了"凡是学生都有 name 和 age、都会 show"。真正的数据在对象里。**类是设计，对象是产品**。

## 1.4 类的分类

按用途，日常写的类大致分三类：

| 类型 | 特征 | 例子 |
|------|------|------|
| **JavaBean 类** | 用来描述一类事物，**不写 `main` 方法** | `Student`、`Order`、`User` |
| **测试类** | 编写 `main` 方法的类，用来测试其他类 | `StudentTest` |
| **工具类** | 只提供静态方法、通常**私有化构造方法**防止被实例化 | `Arrays`、`Math`、自定义的 `DateUtils`（见第 10 篇 1.2.2） |

## 1.5 一个文件里能写几个类

- 一个 `.java` 文件中**可以定义多个类**，但**只能有一个类是 `public` 修饰的**；
- 且 **`public` 修饰的类名必须与文件名完全一致**（`Student.java` 里只能有一个 `public class Student`）；
- **实际开发中建议一个文件只定义一个类**，文件名与类名保持一致，便于定位。

成员变量的完整定义格式：

```java
修饰符 数据类型 变量名 = 初始化值;    // 初始化值一般无需指定，成员变量有默认值
```

| 类型 | 默认值 |
|------|--------|
| `byte` / `short` / `int` / `long` | `0` |
| `float` / `double` | `0.0` |
| `char` | `'\u0000'`（空字符） |
| `boolean` | `false` |
| 引用类型（类 / 数组 / 接口） | `null` |

# 2. 类的组成

## 2.1 类的完整结构

```java
修饰符 class 类名 {
    成员变量;      // 属性：描述"有什么"
    成员方法;      // 行为：描述"能做什么"
    构造方法;      // 创建对象时执行的初始化代码
    代码块;        // 一段在特定时机自动执行的语句，见第 10 篇 4.4
    内部类;        // 写在类里的类，见第 10 篇第 7 节
}
```

```text
  ┌─────────────────────────── 类 ───────────────────────────┐
  │  成员变量     name、age              ← 数据（属性）        │
  │  成员方法     show()                 ← 行为                │
  │  构造方法     Student(...)           ← 创建时初始化         │
  │  代码块       { ... }  static { ... } ← 按时机自动执行      │
  │  内部类       class Inner { ... }     ← 更细粒度的组织      │
  └──────────────────────────────────────────────────────────┘
```

这五样是类的全部成员。前两样决定"这个类是什么"，后三样是 Java 提供的组织与初始化工具。

## 2.2 成员变量

写在**类中、方法外**的变量叫**成员变量**。它的完整格式是 `修饰符 数据类型 变量名 = 初始化值;`。

两点需要注意：

1. **可以不写初始化值**，JVM 会给默认值（见 1.5 的表）——这与局部变量"必须手动赋值才能使用"截然不同；
2. 成员变量在整个类的方法里都可以直接访问，不需要传参。

## 2.3 成员方法

即第 6 篇讲的方法，只是不带 `static` 修饰时它属于**对象**，调用前必须 `new` 出对象。`static` 修饰的方法属于类，详见第 10 篇第 1 节。

## 2.4 构造方法

创建对象时自动执行，用来给成员变量赋初值。它是本篇的重点，见第 4 节。

## 2.5 代码块

代码块是写在成员位置上的一对 `{}`，按执行时机分三种：局部代码块（在方法里，用来限定变量作用域）、构造代码块（每次 `new` 都执行一次，早于构造方法）、静态代码块（类加载时执行一次，只执行一次）。完整规则见第 10 篇 4.4。

## 2.6 内部类

写在类里面的类叫**内部类**，用于表达"只服务于外部类"的辅助类型，也是匿名内部类（Lambda 的语法基础）的载体。分类与用法见第 10 篇第 7 节。

## 2.7 匿名对象

**匿名对象**是没有变量接收的对象，只能使用一次：

```java
public class AnonymousDemo {
    public static void main(String[] args) {
        // 具名对象：可以反复使用
        Student s = new Student();
        s.name = "张三";
        s.show();
        s.show();             // 同一个对象再用一次

        // 匿名对象：创建出来立刻用掉，没有引用指向它，用完即成垃圾
        new Student().show();  // 只能调这一次
        // new Student().name = "李四";   // 语法上把匿名对象当左值不成立，通常只用于调用方法

        // 典型用途：只传一次参
        printStudent(new Student());
    }

    public static void printStudent(Student stu) {
        System.out.println("收到一个学生对象：" + stu);
    }
}
```

> **注意**：匿名对象每次 `new` 都是**不同的对象**，`new Student().show(); new Student().show();` 是让两个互不相干的对象各表演了一次，不是同一个对象重复使用。它适合"用完即弃"的场景（一次性传参、一次性调用），如果后面还要用，就必须用变量接住。

# 3. 对象的创建与内存布局

## 3.1 创建对象的五个步骤

执行 `Student s = new Student()` 时，JVM 依次做了五件事：

```text
  ① 类加载：首次用到 Student 时，把 Student.class 加载进方法区，生成类信息
        │
        ▼
  ② 栈中声明引用变量 s（此时还没有指向任何对象）
        │
        ▼
  ③ 堆中开辟对象空间，成员变量先被赋上默认值（name = null，age = 0）
        │
        ▼
  ④ 执行构造方法，用传入的参数覆盖默认值，完成初始化
        │
        ▼
  ⑤ 把堆中对象的首地址赋给栈上的引用变量 s
```

注意第 ③ 步的顺序：**先赋默认值，再执行构造方法**。所以即便构造方法里不碰某个成员变量，它也不会是"未初始化"状态，而是默认值。

## 3.2 内存图

```text
      栈（Stack）                堆（Heap）
  ┌────────────────┐      ┌──────────────────────────┐
  │  main 栈帧      │      │                          │
  │  ┌──────────┐  │      │    Student 对象           │
  │  │ s = 0x1A │──┼─────▶│  ┌────────────────────┐  │
  │  └──────────┘  │      │  │ name = "张三"       │  │
  └────────────────┘      │  │ age  = 18          │  │
                          │  └────────────────────┘  │
   方法区（Method Area）    └──────────────────────────┘
  ┌────────────────────┐
  │ Student 类信息      │   ← 所有 Student 对象共用一份
  │  成员变量表          │     对象头里的 Klass Pointer 指向它
  │  方法字节码          │
  │  常量池              │
  └────────────────────┘
```

三者分工：

| 区域 | 存放什么 | 生命周期 | 线程关系 |
|------|----------|----------|----------|
| **栈（Stack）** | 局部变量、方法参数、**对象引用** | 随方法调用而生，随方法结束而亡 | 每个线程独立 |
| **堆（Heap）** | 所有 `new` 出来的对象和数组 | 随对象创建而存活，**由 GC 回收** | 所有线程共享 |
| **方法区（Method Area）** | 类信息、静态变量、常量池、JIT 编译后的代码 | 随类加载而存在，随类卸载而消失 | 所有线程共享 |

方法区在 JDK8 以前叫"永久代（PermGen）"，位于堆外；**JDK8 以后改为"元空间（Metaspace）"，使用本地内存**，不再受 `-XX:MaxPermSize` 限制。

> **设计考量**：为什么对象放堆、引用放栈？因为对象的**生命周期不确定**——它可能被传出去、被缓存、活到程序结束；而引用的生命周期严格跟随方法调用，放在栈上可以随方法结束**自动清理**，零成本。把"长短不一"的东西分开放，是现代语言内存管理的基本思路。

## 3.3 对象在堆里的三段组成

一个对象在堆内存中并非只有成员变量，它由三部分组成：

| 部分 | 内容 | 说明 |
|------|------|------|
| **对象头** | Mark Word | 哈希码、GC 分代年龄、锁状态标志等运行时数据（锁的实现就藏在这里，见第 30 篇） |
| | Klass Pointer | 指向方法区中类元数据的指针，告诉 JVM"我是哪个类的实例" |
| **实例数据** | 各个成员变量的值 | 真正存储业务数据的地方，父类字段通常排在前面 |
| **对齐填充** | 补齐字节 | JVM 要求对象起始地址是 **8 字节的整数倍**，不够的位置用空白填充 |

> **注意**：一个空类 `new` 出来的对象也**不是 0 字节**——对象头本身就要占十几个字节。所以"用对象封装一个 `int`"在内存上是不划算的，这正是 Java 提供基本类型而不是全用包装类的理由之一。

## 3.4 多个引用指向同一对象 / 两个对象互不影响

**情形一：多个引用指向同一个对象。**

```java
Student s1 = new Student();
Student s2 = s1;               // 赋值的是地址，s1 和 s2 指向堆中同一个对象
s1.name = "张三";
System.out.println(s2.name);   // 张三 —— 通过任何一个引用都能看到同一份数据
```

```text
  ┌──────────────┐
  │ s1 = 0x1A ───┼──┐
  └──────────────┘  │      ┌────────────────────┐
                    ├─────▶│ name = "张三"       │
  ┌──────────────┐  │      │ age  = 0           │
  │ s2 = 0x1A ───┼──┘      └────────────────────┘
  └──────────────┘
     两个引用，一个对象
```

**情形二：两个独立对象互不影响。**

```java
Student a = new Student();
Student b = new Student();     // 各自 new，堆里是两块独立空间
a.name = "张三";
b.name = "李四";
System.out.println(a.name);    // 张三
System.out.println(b.name);    // 李四
```

```text
     栈                        堆
  ┌──────────┐          ┌──────────────────┐
  │ a = 0x1A │─────────▶│ name = "张三"     │
  ├──────────┤          └──────────────────┘
  │ b = 0x2B │─────────▶┌──────────────────┐
  └──────────┘          │ name = "李四"     │
                        └──────────────────┘
```

> **陷阱**：`Student s2 = s1;` 是**引用赋值**，不是复制对象。想复制出一个独立对象，必须重新 `new`，或者实现 `Cloneable` / 写拷贝构造方法（浅拷贝与深拷贝的区别见第 11 篇）。

# 4. 构造方法

## 4.1 格式

```java
修饰符 类名(参数列表) {
    方法体;
}
```

```java
public class Student {
    String name;
    int age;

    // 全参构造：参数名与成员变量同名时，靠 this 区分（见第 5 节）
    public Student(String name, int age) {
        this.name = name;
        this.age = age;
    }
}
```

## 4.2 六个特点

1. **方法名必须与类名完全相同**（连大小写都要一致）；
2. **没有返回值类型，连 `void` 都不能写**——写了 `void` 就变成普通方法了；
3. **没有具体的返回值**，不能由 `return` 带回结果数据（可以写裸 `return;` 提前结束）；
4. **如果没有定义任何构造方法，系统会赠送一个默认的无参构造方法**；
5. **一旦自己定义了构造方法（哪怕只有一个有参的），系统就不再提供默认无参构造**。
6. **构造方法不能被继承**——子类不能继承父类的构造方法，只能通过 `super(...)` 调用（见第 10 篇 2.3.1）。

```java
public class BadConstructor {
    // ✗ 编译错误：构造方法不能写返回值类型
    public void BadConstructor() { }

    // ✓ 这才是构造方法
    public BadConstructor() { }
}
```

> **陷阱**：上面那个 `public void BadConstructor()` 是个非常隐蔽的 bug——它**语法完全合法**，只是不再被当作构造方法，而成了一个普通方法。后果是"类里明明写了一个看起来是无参构造的东西，`new BadConstructor()` 却说找不到构造方法"。

## 4.3 执行时机

- **创建对象的时候由虚拟机自动调用，不能手动调用**（`s.Student()` 是编译错误）；
- **每创建一次对象，就会调用一次构造方法**——10 个对象就执行 10 次。

```java
public class Student {
    String name;

    public Student() {
        System.out.println("构造方法执行了");
    }

    public static void main(String[] args) {
        Student a = new Student();   // 构造方法执行了
        Student b = new Student();   // 构造方法执行了
        Student c = new Student();   // 构造方法执行了
    }
}
```

输出：

```text
构造方法执行了
构造方法执行了
构造方法执行了
```

> **注意**：构造方法**不是**"创建对象的方法"，而是"**对象创建完成后的初始化入口**"。对象空间在构造方法执行之前就已经在堆里分配好了（见 3.1 的第 ③ 步）。

## 4.4 默认无参构造的由来与经典坑

**为什么会有默认无参构造？** 因为 `new Student()` 这个调用需要一个无参构造方法才能成立。如果类里一个构造方法都没写，编译器会**自动插入**一个空的 `public Student() {}`，让最朴素的 `new` 能跑通。

**经典坑**：只要你写了任何一个构造方法，这个"赠品"就消失了。

```java
class Person {
    String name;

    // 只写了有参构造 —— 编译器不再赠送无参构造
    public Person(String name) {
        this.name = name;
    }
}

public class Demo {
    public static void main(String[] args) {
        Person p1 = new Person("张三");   // ✓ 正常
        Person p2 = new Person();         // ✗ 编译错误：找不到符号 构造器 Person()
    }
}
```

报错信息是 `constructor Person in class Person cannot be applied to given types; required: String; found: no arguments`——很多框架（尤其是靠反射创建对象的 Spring、Jackson、MyBatis）**都要求存在无参构造**，缺了它会在运行时报错，排查起来很费劲。

> **最佳实践**：**无论是否用到，都手动写上无参构造**。一旦定义了有参构造，立刻补一个 `public Xxx() {}`，避免"加了参数就崩"这种低级故障。这也是第 7 节标准 JavaBean 要求"至少两个构造方法"的原因。

## 4.5 构造方法重载与 `this(...)` 链

构造方法**支持重载**：可以定义多个参数列表不同的构造方法。

```java
public class Student {
    private String name;
    private int age;

    // 无参构造：委托给全参构造，避免重复写赋值逻辑
    public Student() {
        this("无名氏", 0);        // this(...) 必须是构造方法的第一条语句
    }

    // 单参构造：同样委托过去
    public Student(String name) {
        this(name, 0);
    }

    // 全参构造：真正干活的只有这一个
    public Student(String name, int age) {
        this.name = name;
        this.age = age;
    }
}
```

这种写法叫**构造方法链（Constructor Chaining）**，好处是**初始化的默认值只定义一次**，修改时不会漏掉某个分支。

```java
public class ConstructorChainDemo {
    public static void main(String[] args) {
        System.out.println(new Student().getName() + " / " + new Student().getAge());
        System.out.println(new Student("张三").getName() + " / " + new Student("张三").getAge());
        System.out.println(new Student("李四", 20).getName() + " / " + new Student("李四", 20).getAge());
    }
}
```

输出：

```text
无名氏 / 0
张三 / 0
李四 / 20
```

两条硬规则：

1. `this(...)` / `super(...)` **必须是构造方法中的第一条语句**，前面不能有任何代码；
2. **`this(...)` 和 `super(...)` 不能同时出现**——一个构造方法只能有一个出口指向别的构造方法。

> **陷阱**：构造方法之间不能**循环调用**（`Student()` 调 `Student(String)`、后者又调回前者），编译器直接报 `recursive constructor invocation`。另外 `this(...)` 前如果先写了 `int x = 1;` 之类，同样编译不过。

## 4.6 创建对象时的完整执行顺序

```java
Student s = new Student("张三", 18);
```

```text
  ① 类加载（仅首次）
     父类静态代码块 → 子类静态代码块 → 静态成员变量初始化
          │
          ▼
  ② 堆中开辟空间，成员变量赋默认值（name = null，age = 0）
          │
          ▼
  ③ 父类构造代码块 → 父类构造方法
          │
          ▼
  ④ 子类构造代码块 → 子类构造方法
     （若构造方法里有 this(...)，先跳到被调用的那个构造方法执行）
          │
          ▼
  ⑤ 把对象地址赋给栈上的引用变量 s
```

**一句话记忆**：**静态先行（父先子后）→ 默认值 → 代码块 → 构造方法**。涉及继承的部分详见第 10 篇 2.7。

# 5. `this` 关键字

## 5.1 本质：当前对象的引用

`this` 代表**当前对象的引用**——即"调用这个方法的那个对象"。它的存在是为了让方法知道自己是在为谁服务。

**`this` 的本质**：每个**非静态方法**中都隐含着一个 `this` 引用。编译之后，`this` 会作为该方法的**第一个参数隐式传入**，所以在 JVM 看来，实例方法其实长这样：

```text
  Java 源码：                      编译后的等价形式：
  void setName(String name) {     void setName(Student this, String name) {
      this.name = name;               this.name = name;
  }                                }
```

这也顺带解释了第 6 篇值传递那条规则：**对象方法调用时，那个"隐藏的第一个参数"传的也是地址的副本**。

## 5.2 用法一：区分成员变量和局部变量

当**形参名与成员变量名相同**时，方法内直接写 `name` 用的是**形参**（就近原则），要访问成员变量必须加 `this.`：

```java
public class Student {
    private String name;
    private int age;

    // 形参名刻意与成员变量同名，这是最规范的写法，靠 this 区分
    public void setName(String name) {
        this.name = name;      // 左边是成员变量，右边是形参
    }

    public String getName() {
        return name;           // 没有同名局部变量时不加 this 也指向成员变量
    }

    public void setAge(int age) {
        this.age = age;
    }

    public int getAge() {
        return age;
    }

    public static void main(String[] args) {
        Student s = new Student();
        s.setName("张三");
        s.setAge(18);
        System.out.println(s.getName() + " / " + s.getAge());   // 张三 / 18
    }
}
```

> **最佳实践**：**所有成员变量都显式加 `this.`**（或至少在 setter 里加），即使当前没有同名局部变量。这样将来方法里新增同名局部变量时不会莫名改变行为，代码意图也更清楚。

## 5.3 用法二：`this(...)` 调用本类其他构造方法

见 4.5。要点是**只能出现在构造方法的第一行**，用于构造方法之间的复用。

## 5.4 用法三：返回当前对象，实现链式调用

方法把 `this` 返回出去，调用者就能接连点下去，这就是**链式调用（Fluent API）**：

```java
public class Student {
    private String name;
    private int age;

    // 每个 setter 都返回 this，调用方就能一路点下去
    public Student setName(String name) {
        this.name = name;
        return this;
    }

    public Student setAge(int age) {
        this.age = age;
        return this;
    }

    @Override
    public String toString() {
        return "Student{name='" + name + "', age=" + age + "}";
    }

    public static void main(String[] args) {
        // 链式调用：每一步返回的都是同一个对象
        Student s = new Student()
                .setName("张三")
                .setAge(18);
        System.out.println(s);   // Student{name='张三', age=18}
    }
}
```

`StringBuilder`、`Stream`、`Optional` 以及各种建造者（Builder）都用了这套设计。返回 `this` 的代价是**放弃了返回值**——这个方法只能用来设置属性，不能再返回别的东西。

> **设计考量**：链式调用的可读性收益来自"调用顺序 = 阅读顺序"，适合参数多的构建过程。但它也掩盖了"每一步都在修改同一个对象"这一事实，在并发或需要复用中间状态的场景下要慎用。

## 5.5 用法四：作为参数传递、在内部类中引用外部类实例

**传递当前对象**：把 `this` 交给其他方法，等于"把我自己交出去"。

```java
public class Student {
    private String name;

    public void register(School school) {
        school.enroll(this);        // 把自己的引用交给学校
    }
}
```

**在内部类中引用外部类实例**：内部类里的 `this` 指的是内部类对象，要拿外部类对象得写 `OuterClass.this`。详见第 10 篇 7.2。

## 5.6 注意事项

| 规则 | 说明 |
|------|------|
| 静态方法中不能用 `this` | `static` 方法属于类，不属于任何对象，没有"当前对象"可指 |
| `this(...)` 必须是第一条语句 | 前面不能有任何代码，且与 `super(...)` 互斥 |
| 构造方法之间不能循环调用 | 会编译报错 `recursive constructor invocation` |
| `this` 不是变量，不能赋值 | `this = new Student();` 是编译错误 |

# 6. 封装

## 6.1 为什么要封装

**封装（Encapsulation）** 指隐藏对象的属性和实现细节，仅对外提供公共的访问方式。它的思想是：**一个对象代表什么，就得封装对应的数据，并提供数据对应的行为**。

先看看不封装会怎样：

```java
// ✗ 不封装：字段全部 public，任何地方都能塞进非法数据
class StudentBad {
    public String name;
    public int age;
}

public class BadDemo {
    public static void main(String[] args) {
        StudentBad s = new StudentBad();
        s.age = -1;              // 语法完全合法，业务上荒谬
        s.name = null;           // 也没人拦着
        System.out.println("年龄是 " + s.age);   // 年龄是 -1
    }
}
```

问题有三层：

1. **数据和行为脱节**——`age` 是"学生的年龄"，但对 `age` 的任何操作都散落在调用方，`Student` 类本身完全不知情；
2. **无法校验**——直接赋值绕过了所有规则，非法数据长驱直入；
3. **无法演进**——将来想把 `age` 改名为 `birthYear`，所有 `.age` 的地方都得改，改动面不可控。

## 6.2 合理隐藏 + 合理暴露

封装的两条设计原则：

- **合理隐藏**：用 `private` 把属性声明为私有，禁止外部直接修改；
- **合理暴露**：通过 `public` 的 `getter` / `setter` 方法提供**可控的**访问接口。

```java
public class Student {
    private String name;      // 隐藏：外界看不见也摸不着
    private int age;

    public String getName() {          // 暴露：只读入口
        return name;
    }

    public void setName(String name) {
        if (name != null && !name.trim().isEmpty()) {   // 校验：非法数据进不来
            this.name = name;
        }
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        if (age >= 0 && age <= 150) {                    // 校验：范围外的直接拒绝
            this.age = age;
        }
    }
}
```

## 6.3 对照：`setAge(-1)` 到底会发生什么

```java
public class EncapsulationContrast {
    public static void main(String[] args) {
        // 版本一：public 字段，非法数据照单全收
        StudentBad bad = new StudentBad();
        bad.setAge(-1);                       // 假设也有个不校验的 setAge
        System.out.println("不封装：age = " + bad.age);    // 不封装：age = -1

        // 版本二：private + 校验，非法数据被挡在门外
        Student good = new Student();
        good.setAge(-1);
        System.out.println("封装后：age = " + good.getAge());   // 封装后：age = 0（保持默认值）

        good.setAge(18);
        System.out.println("合法值：age = " + good.getAge());   // 合法值：age = 18
    }
}
```

输出：

```text
不封装：age = -1
封装后：age = 0
合法值：age = 18
```

**这就是封装的价值的直观体现**：对象始终处于**合法状态**，非法输入被挡在边界上，而不是污染进去之后在某个遥远的业务逻辑里爆炸。

## 6.4 封装的四项好处

| 好处 | 说明 |
|------|------|
| **提高安全性** | `setter` 中可以加入任意校验逻辑，保证对象始终合法 |
| **提高复用性** | 内部实现改变时，只要对外接口不变，调用者无需修改代码 |
| **高内聚低耦合** | 数据和操作数据的方法绑定在一起，降低模块间的依赖 |
| **便于演进** | 比如把 `age` 改成由 `birthday` 计算而来，外部只调 `getAge()`，无需感知 |

## 6.5 权限修饰符

`private` 表示"只能在本类中访问"，`public` 表示"任何地方都能访问"。完整的四档权限（`private` / 默认 / `protected` / `public`）及其对子类、同包、跨包的影响，见第 10 篇 4.3。

# 7. 标准 JavaBean

## 7.1 规范

**JavaBean** 是一种 Java 语言写成的可重用组件。所谓"标准 JavaBean"（也常称为 **POJO 规范**），是一套被框架广泛依赖的书写约定：

| 编号 | 规范 |
|------|------|
| 1 | **类名见名知意**，能直接看出它描述什么事物 |
| 2 | **成员变量全部用 `private` 修饰** |
| 3 | 提供**至少两个构造方法**：无参构造 + 带全部参数的构造方法 |
| 4 | 提供每个成员变量对应的 **`set...()` 和 `get...()`** 方法 |
| 5 | 如果还有其他业务行为方法，也需要写上 |
| 6 | 建议实现 **`Serializable`** 接口（涉及序列化时） |

为什么框架如此依赖这套约定？因为 Spring 注入、Jackson 序列化、MyBatis 结果映射等，都是**靠反射按 `getXxx` / `setXxx` 命名规则**去读写属性的。字段是 `private` 无所谓——反射可以越过权限；但没有无参构造、或者没有标准的 getter/setter，框架就失灵了。

## 7.2 完整示例

```java
import java.io.Serializable;

public class Student implements Serializable {
    private static final long serialVersionUID = 1L;   // 序列化版本号，避免反序列化失败

    private String name;
    private int age;

    // 规范 3：无参构造
    public Student() {
    }

    // 规范 3：全参构造
    public Student(String name, int age) {
        this.name = name;
        this.age = age;
    }

    // 规范 4：每个成员变量一对 getter/setter
    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    // 规范 5：业务行为方法，通常也要重写 toString 方便调试
    @Override
    public String toString() {
        return "Student{name='" + name + "', age=" + age + "}";
    }

    public static void main(String[] args) {
        Student s = new Student("张三", 18);      // 全参构造
        System.out.println(s);                    // Student{name='张三', age=18}

        Student t = new Student();                // 无参构造
        t.setName("李四");
        t.setAge(20);
        System.out.println(t);                    // Student{name='李四', age=20}
    }
}
```

## 7.3 快速生成

手写 getter/setter 枯燥且易错，实践中有三种加速手段：

| 方式 | 说明 |
|------|------|
| **IDEA 快捷键 `alt + insert`** | 生成 `Getter and Setter`、`Constructor`、`toString`，最常用 |
| **ptg 插件** | 从 JSON、SQL 建表语句等多种输入快速生成 JavaBean |
| **Lombok 库** | 用 `@Data`、`@Getter`、`@Setter`、`@AllArgsConstructor`、`@NoArgsConstructor` 等注解在**编译期**自动生成模板代码 |

Lombok 版本的等价写法：

```java
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data                   // 等价于 @Getter + @Setter + @ToString + @EqualsAndHashCode
@NoArgsConstructor      // 自动生成无参构造
@AllArgsConstructor     // 自动生成全参构造
public class Student {
    private String name;
    private int age;
}
```

> **注意**：Lombok 是**编译期**生效的注解处理器，需要 IDE 安装插件，团队里必须统一（否则别人拉下代码会报"找不到 `getName()`"）。它对单元测试覆盖率、断点调试也有一定影响，引入前需评估。

## 7.4 现代替代：Record 与 Optional

Java 14 起提供了 **`record`**，把"不可变的纯数据载体"压缩成一行：

```java
public record Student(String name, int age) { }
```

编译器自动生成全参构造、访问器（`name()`、`age()` 而非 `getName()`）、`equals`、`hashCode`、`toString`。代价是**字段不可变、不能继承**，因此适合 DTO / 值对象，不适合需要 setter 的传统实体。详见第 26 篇。

可空参数则用 `Optional<T>` 表达，让"可能没有值"变成类型的一部分，而不是靠 `null` 约定。详见第 28 篇。

**选择建议**：

| 场景 | 推荐写法 |
|------|----------|
| 需要被 Spring / MyBatis / Jackson 操作的传统实体 | 标准 JavaBean（或 Lombok `@Data`） |
| 不可变的值对象 / DTO / 方法多返回值 | `record` |
| 可能缺失的单个返回值 | `Optional` |

# 8. 成员变量与局部变量

这是初学时极易混淆的一组概念。局部变量指的是**定义在方法内、方法参数、构造方法参数、代码块内**的变量。

| 对比维度 | 成员变量（实例变量） | 局部变量 |
|----------|---------------------|----------|
| **定义位置** | 类中、方法外 | 方法内 / 方法参数 / 构造方法参数 / 代码块内 |
| **内存位置** | 堆内存（对象的一部分） | 栈内存（所在方法的栈帧里） |
| **默认值** | **有**：`0` / `0.0` / `'\u0000'` / `false` / `null` | **无**，必须手动赋值后才能使用，否则编译错误 |
| **生命周期** | 随对象的创建而存在，随对象的销毁而消失 | 随方法的调用而存在，随方法执行完毕而消失 |
| **作用域** | 整个类（配合 `this` 或对象名访问） | 从声明处到所在 `{}` 结束 |
| **可用修饰符** | `private` / `public` / `protected` / `static` / `final` / `transient` 等 | **只能加 `final`** |
| **线程安全** | 共享对象上的成员变量**不安全** | 各线程独立栈帧，**天然安全**（见第 17 篇） |

```java
public class VariableScopeDemo {
    int member = 100;          // 成员变量：有默认值，可加各种修饰符

    public void test() {
        int local;             // 局部变量：没有默认值
        // System.out.println(local);   // ✗ 编译错误：变量 local 可能尚未初始化
        local = 10;
        System.out.println("local = " + local + ", member = " + member);   // local = 10, member = 100
    }

    public static void main(String[] args) {
        new VariableScopeDemo().test();
    }
}
```

**就近原则**：当局部变量与成员变量同名时，方法内**优先使用局部变量**；要访问成员变量必须用 `this.变量名` 区分：

```java
public class ShadowDemo {
    int num = 1;                     // 成员变量

    public void show() {
        int num = 2;                 // 局部变量，遮蔽了成员变量
        System.out.println(num);       // 2（就近原则，用的是局部变量）
        System.out.println(this.num);  // 1（用 this 才能拿到成员变量）
    }

    public static void main(String[] args) {
        new ShadowDemo().show();
    }
}
```

**`static` 修饰的成员变量（静态变量 / 类变量）** 是第三种情况，需要单独记住：

- 存储在**方法区**（JDK8+ 实际在堆中的元空间对应位置），**不属于任何对象**；
- 随**类的加载**而加载，被所有对象**共享**——任何一个对象改了它，其他对象看到的都是新值；
- 可以用 **`类名.变量名`** 直接访问（推荐），也能通过对象名访问（不推荐，容易误以为它是实例变量）。

详见第 10 篇第 1 节。

# 9. 内存回收与垃圾回收（GC）

## 9.1 为什么 Java 不用手动释放内存

C / C++ 需要程序员自己 `free` / `delete`，漏掉就是内存泄漏，重复释放就是崩溃。Java 把这件事交给了 **GC（Garbage Collector，垃圾回收器）**——一个由 JVM 自动运行的后台线程，负责找出并回收不再被引用的对象。

好处是**大幅降低内存管理的负担**；代价是**回收时机不可预测**、极端场景下（比如大对象分配过快）会有停顿。GC 的具体算法（分代收集、标记-清除、复制、G1/ZGC 等）在 JVM 板块展开，这里只讲**判断标准**和**开发侧的注意事项**。

## 9.2 怎么判断一个对象"该被回收"

| 算法 | 原理 | Java 是否采用 | 缺陷 |
|------|------|--------------|------|
| **引用计数法** | 每个对象维护一个引用计数器，计数为 0 即可回收 | **否** | 无法解决**循环引用**（A 引用 B、B 引用 A，两者计数都不为 0，却都访问不到） |
| **可达性分析算法** | 从一批根对象（GC Roots）出发沿着引用链搜索，**搜索不到的对象**即为可回收 | **是** | 需要暂停用户线程做快照（Stop-The-World） |

**对象被回收的必要条件是没有引用指向它（或从 GC Roots 不可达）。** 反过来说，只要还有一条从 GC Roots 出发的引用链能走到某个对象，它就不能被回收——这就是为什么"集合里越加越多、静态 Map 当缓存用"会导致**内存泄漏**：对象明明已经没用了，但引用链还在。

## 9.3 GC Roots 包括哪些

| GC Root | 说明 |
|---------|------|
| **虚拟机栈中引用的对象** | 各线程方法栈帧里局部变量表引用的对象 |
| **方法区中静态变量引用的对象** | `static` 字段指向的对象，最容易造成"长命"引用 |
| **方法区中常量引用的对象** | 常量池中的引用 |
| **本地方法栈中 JNI 引用的对象** | native 方法持有的引用 |
| **被同步锁（`synchronized`）持有的对象** | 正在被当作锁的对象 |
| **JVM 内部的引用** | 如基本类型的 Class 对象、系统类加载器 |

## 9.4 手动触发 GC

```java
System.gc();                        // 建议 JVM 执行垃圾回收，但不保证立即执行
Runtime.getRuntime().gc();          // 效果完全等价，底层是同一个调用
```

> **陷阱**：`System.gc()` **只是"建议"而不是"命令"**。它默认等价于 `System.runFinalization()` + 一次 Full GC 的提示，JVM 完全可以忽略（通过 `-XX:+DisableExplicitGC` 就彻底禁用它）。真实场景中调用它通常只是为了让 log 里看到回收结果，**绝不能把它当成"释放内存"的手段**，否则会在线上引发毫无必要的 Full GC 停顿。

## 9.5 `finalize()` 已废弃

```java
@Override
protected void finalize() throws Throwable {
    super.finalize();
}
```

对象被回收前，JVM 曾会调用它的 `finalize()` 方法，用于做资源释放。但这个方法**从 JDK9 起被标记为 `@Deprecated`，JDK18 起彻底移除**，原因很清楚：

1. **执行时机完全不确定**——可能很久之后才执行，也可能永远不执行（GC 不保证一定触发 finalize）；
2. **会拖慢 GC**——被 finalize 的对象至少要多活一轮回收周期；
3. **可能在 finalize 中"复活"自己**（把 `this` 赋给外部引用），让回收逻辑变得极其诡异。

**正确的资源释放方式是 try-with-resources / 显式 `close()`**，不要依赖 GC 去关文件、关连接、释放锁。详见第 14、15 篇。

# 10. 小结

| 主题 | 一句话要点 |
|------|-----------|
| 三步走 | 定义类（模板）→ `new` 创建对象（产品）→ 用 `对象名.成员` 使用；类是设计，对象是数据 |
| 类的组成 | 成员变量 + 成员方法 + 构造方法 + 代码块 + 内部类；代码块与内部类详见第 10 篇 |
| 内存布局 | 引用在**栈**、对象在**堆**、类信息在**方法区**；对象本身由对象头、实例数据、对齐填充组成 |
| 引用复制 | `Student s2 = s1;` 复制的是地址，两个引用改同一个对象；要独立对象必须重新 `new` |
| 构造方法 | 方法名同类名、无返回值类型、不能被继承；**一旦写了有参构造，默认无参构造就消失**，务必手写补上 |
| `this` | 本质是隐含的第一个参数；区分同名变量、`this(...)` 调用本类构造（必须第一行）、返回 `this` 做链式调用；静态方法中不可用 |
| 封装 | `private` 字段 + `public` getter/setter；价值在于**校验、隐藏实现、便于演进**，让对象始终合法 |
| 标准 JavaBean | 无参构造 + 全参构造 + 私有字段 + 标准 getter/setter + 建议 `Serializable`；可用 Lombok `@Data`、`record`（第 26 篇）替代 |
| 成员变量 vs 局部变量 | 位置（类中/方法内）、内存（堆/栈）、默认值（有/无）、生命周期（对象/方法）、修饰符（任意/仅 `final`）五处不同 |
| GC | 采用**可达性分析**（从 GC Roots 出发找得到才存活）；`System.gc()` 只是建议；`finalize()` 已废弃，资源释放用 try-with-resources |

下一篇（第 8 篇）从对象转向**字符串与常用 API**，看 `String` 的不可变性、常量池与 `StringBuilder` 的取舍；继承、多态、接口、内部类等面向对象进阶内容集中在第 10 篇，JVM 内存结构与 GC 算法的完整原理见 JVM 板块。
