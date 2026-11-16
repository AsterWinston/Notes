面向对象的三大特性是**封装、继承、多态**，而 Java 用来支撑它们的语法工具，基本都集中在 `static`、`extends`、`implements`、`abstract`、权限修饰符和内部类这几样东西上。本篇把这套"类和类之间怎么组织"的规则一次讲透：先讲不属于任何对象的 `static`，再讲继承与多态这对搭档，最后落到抽象类、接口与内部类的选型。学完之后真正要建立的判断力只有两条——**这个成员属于类还是属于对象**，**这个引用编译时是什么类型、运行时是什么对象**；把这两问想清楚，绝大多数 Java 语法题都会变成送分题。

# 1. static 关键字

## 1.1 静态成员变量

`static` 的字面意思是"静态的"，在 Java 里的准确定义是**属于类而不属于对象**。不加 `static` 的成员变量叫**实例变量**，每 `new` 一个对象就在堆里存一份；加了 `static` 的变量叫**静态变量**（也叫类变量），整个类只有一份，被所有对象共享。

两者的差别可以用一张表彻底说清：

| 维度 | 实例变量（无 `static`） | 静态变量（有 `static`） |
|------|----------------------|----------------------|
| 归属 | 属于对象 | 属于类 |
| 内存份数 | 每个对象一份 | 全类共享一份 |
| 存放位置 | 堆中的对象内部 | JDK7 起随 `Class` 对象存放（HotSpot 在堆中）|
| 创建时机 | `new` 对象时 | **类加载时**，早于任何对象的创建 |
| 销毁时机 | 对象被 GC 回收 | 类被卸载（基本等同于程序结束）|
| 推荐访问方式 | `对象名.变量名` | `类名.变量名` |

教材里常说"静态变量放在方法区"，这只描述了 JDK7 之前 HotSpot 的情况。JDK7 起 HotSpot 把静态字段随 `java.lang.Class` 对象一起挪进了堆，JDK8 之后方法区本身也改由**元空间**实现。所以更准确的说法是：**静态变量跟着 `Class` 对象走**。不过对写代码的人来说，需要记住的只有一句——**静态成员的生命周期和类一致，而不是和对象一致**。

```java
public class Student {
    // 静态变量：全班共用一个计数，它属于 Student 这个类
    public static int totalCount = 0;

    // 实例变量：每个学生一份，它属于某个具体的学生对象
    public String name;

    public Student(String name) {
        this.name = name;
        totalCount++;          // 编译器会把它当成 Student.totalCount++
    }

    public static void main(String[] args) {
        Student s1 = new Student("张三");
        Student s2 = new Student("李四");

        System.out.println(s1.name + "、" + s2.name);   // 张三、李四：实例变量各不相同
        System.out.println(Student.totalCount);         // 2：类名访问，推荐写法
        System.out.println(s1.totalCount);              // 2：用对象也能访问，但不推荐
    }
}
```

用对象访问静态成员只是"语法上放行"，语义上完全是误导。`javac` 在编译 `s1.totalCount` 时会直接把它解析成 `Student.totalCount`，生成一条 `getstatic` 指令，连"这个对象是谁"都不关心。

> **注意**：正因为编译后是 `getstatic`，用一个值为 `null` 的引用去读静态变量**不会抛 `NullPointerException`**。这恰好证明了"用对象访问静态成员"只是一种语法糖，跟那个对象毫无关系。反过来说，用对象访问静态成员也成了一种"假象"——读者会误以为它和对象状态相关，所以阿里规约一类的规范都要求**静态成员一律用类名访问**。

判断一个成员该不该加 `static`，只需要问一句："**这个值和具体某个对象有关吗**？"如果所有对象看到的值都一样，或者这个值在没有任何对象存在时就必须能用，那它就该是静态的。典型的静态变量用途是：全类共享的计数器、全局配置、缓存表、以及配合 `final` 定义的常量。

## 1.2 静态成员方法

`static` 修饰的方法叫**静态方法**，它同样属于类，通过 `类名.方法名()` 调用。静态方法最经典的应用场景是**工具类**——只提供功能、不需要保存任何状态，也不希望被 `new` 出来。

```java
public class ArrayUtil {              // 类名见名知意：一看就知道是数组工具
    private ArrayUtil() { }           // 私有化构造方法：禁止外部 new，这个类不是用来造对象的

    public static int max(int[] arr) {  // 方法全部静态：直接用类名调用
        if (arr == null || arr.length == 0) {
            throw new IllegalArgumentException("数组不能为空");
        }
        int max = arr[0];
        for (int v : arr) {
            if (v > max) max = v;
        }
        return max;
    }
}

// 调用：不需要创建对象
int result = ArrayUtil.max(new int[]{3, 1, 4, 1, 5});
System.out.println(result);   // 5
```

工具类的三个约定俗成的特征可以总结成一句话：

- **类名见名知意**，通常以 `Util`、`Utils`、`Helper`、`s` 结尾；
- **私有化构造方法**，让 `new ArrayUtil()` 直接编译不过，从源头杜绝"用对象调用静态方法"这种写法；
- **方法全部 `static`**，调用方永远写 `类名.方法(...)`。

JDK 自己就是这套模板的模范生：`java.lang.Math`、`java.util.Arrays`、`java.util.Collections`、`java.util.Objects` 全是私有构造 + 全静态方法。静态方法还有几个附带的语法限制，一并记住：**不能用 `this`**（原因见下一节）、**不能被重写**（只能被子类隐藏）、**不能声明为 `abstract`**、在接口里从 JDK8 起可以定义但只能通过接口名调用（见 6.5 节）。

## 1.3 静态方法为什么不能访问实例成员

这是 Java 语法里最容易被死记硬背、又最该讲清楚的一条规则。核心原因只有一个：**静态方法没有 `this`**。

`this` 的含义是"**当前对象的引用**"。那么"当前对象"是怎么来的？是当你写 `cat.eat()` 时，JVM 把 `cat` 这个对象作为隐藏参数传进 `eat()` 方法，方法内部才拿得到 `this`。而静态方法的调用方式是 `类名.方法()`——调用的时候**可能一个对象都不存在**：

```java
System.out.println(Student.totalCount);   // 直接触发类加载，全程没有一个 Student 对象
```

既然调用静态方法时并不需要一个对象，JVM 就没有东西可以塞给 `this`。没有 `this`，也就无从知道"要访问的是哪个对象的 `name`"，编译器只能在编译期就禁止静态方法直接访问实例成员。

完整的访问规则如下：

| 调用位置 | 访问静态变量/静态方法 | 访问实例变量/实例方法 | 能否使用 `this` |
|---------|-------------------|-------------------|---------------|
| 静态方法 | 可以，直接写名字 | **不可以**（除非自己 `new` 出对象再通过对象访问）| 不可以 |
| 实例方法 | 可以，直接写名字 | 可以，直接写名字 | 可以 |
| 静态代码块 | 可以 | 不可以（此时对象还不存在）| 不可以 |
| 构造方法 | 可以 | 可以 | 可以（并且隐含 `super()`）|

```java
public class AccessDemo {
    private int instanceNum = 10;                 // 实例变量
    private static int staticNum = 20;            // 静态变量

    public static void staticMethod() {
        System.out.println(staticNum);            // ✓ 静态方法访问静态变量
        // System.out.println(instanceNum);       // ✗ 编译错误：无法从静态上下文中引用非静态变量
        AccessDemo demo = new AccessDemo();       // 变通办法：自己造一个对象
        System.out.println(demo.instanceNum);     // ✓ 通过对象访问就合法了
    }

    public void instanceMethod() {
        System.out.println(staticNum);            // ✓ 实例方法访问静态变量
        System.out.println(instanceNum);          // ✓ 实例方法访问实例变量
        System.out.println(this.instanceNum);     // ✓ 实例方法里 this 是存在的
    }
}
```

顺便记住一个反向的事实：**实例方法可以无障碍地访问静态成员**。因为实例方法执行时必然已经有一个对象存在，`this` 有了，类也早就加载好了，两类成员都在手边。

> **设计考量**：这条限制看起来是"禁止"，其实是在**帮你防错**。假如允许静态方法访问实例成员，那么 `Student.printName()` 里写 `System.out.println(name)` 时，`name` 到底是谁的名字？编译器必须临时造一个对象，或者在运行时抛异常。Java 选择了最简单也最安全的做法：**编译期直接报错**，逼你把"这个方法该不该是静态的"想清楚。

## 1.4 静态成员的加载时机

要讲清静态成员"什么时候存在"，得先知道类加载的过程。JVM 加载一个类会经历**加载、验证、准备、解析、初始化**几个阶段，其中和静态变量直接相关的是后两个：

| 阶段 | 静态变量身上发生了什么 |
|------|--------------------|
| 准备 | 为静态变量分配内存，并赋**类型默认值**（`int` → `0`，引用 → `null`，`boolean` → `false`）。此时源码里写的 `= 10` **还没有生效** |
| 初始化 | 执行编译器自动生成的 `<clinit>()` 方法，按**源码书写顺序**执行静态变量的显式赋值和静态代码块，此刻 `10` 才真正写进去 |

"准备阶段赋默认值"这一点能解释一个经典现象：如果静态代码块写在静态变量声明之前，它在块里读到的会是默认值而不是初始值。

```java
public class InitOrder {
    static {
        // 这里读到的 a 是准备阶段给的默认值 0，不是下面写的 10
        System.out.println("静态代码块中 a = " + a);
    }

    static int a = 10;

    public static void main(String[] args) {
        System.out.println("main 中 a = " + a);
    }
}
// 输出：
// 静态代码块中 a = 0
// main 中 a = 10
```

类的**初始化**（也就是静态成员真正就位的那一刻）由以下几种情况触发，称为"主动引用"：

1. 创建类的实例（`new`）；
2. 访问类的静态变量，或调用类的静态方法；
3. 通过反射（`Class.forName`）访问类；
4. 初始化一个子类时，会**先初始化它的父类**；
5. 作为程序入口的类（含 `main` 方法的那个类），JVM 启动时主动初始化。

关键结论是：**静态成员的初始化早于任何对象的创建**。你在 `main` 里第一次碰到某个类时，这个类的静态代码块已经跑完了。

还需要区分**主动引用**和**被动引用**：通过子类名去访问父类的静态变量，只会初始化**父类**，不会初始化子类；访问被 `final` 修饰的**编译期常量**，甚至根本不会触发类的初始化，因为编译时它已经被替换成了字面量（见 4.3 节的常量折叠）。这类细节经常出现在面试题里，考的就是"你是否真的理解类初始化发生在什么时候"。

## 1.5 静态代码块

静态代码块的格式是 `static { ... }`，它写在类的成员位置，**随类的加载而执行，并且整个 JVM 生命周期只执行一次**。它最常见的用途是"类加载时初始化静态资源"——读取配置、注册驱动、填充静态缓存。

```java
public class Config {
    // 静态常量容器：所有代码共享同一份，
    public static final Map<String, String> SETTINGS = new HashMap<>();

    static {                                   // 类加载时执行，全 JVM 只跑一次
        System.out.println("Config 类被初始化");
        SETTINGS.put("timeout", "3000");
        SETTINGS.put("retry", "3");
    }

    private Config() { }                       // 工具类，禁止实例化
}
```

它有几个必须记住的性质：只能访问静态成员（因为执行时对象还不存在）、不能写 `return`、不能抛出受检异常（编译器会把它编译进 `<clinit>()`，受检异常无处可抛，必须当场 `try-catch`）。静态代码块的完整对比放在第 4.7 节，和构造代码块、局部代码块一起看更清楚。

静态代码块还有一个常被忽视的副作用：**它抛出的异常会被 JVM 包装成 `ExceptionInInitializerError`**，并且这个类之后再也无法被正常使用（JVM 会把它标记为错误状态，再访问只会得到 `NoClassDefFoundError`）。所以静态代码块里做的事情要尽量简单可靠，避免强依赖外部资源——"数据库连不上就让整个类无法加载"是不少线上事故的起点。

## 1.6 静态导入

如果一个类里要反复调用某个类的静态成员，可以用**静态导入**省掉类名前缀：

```java
import static java.lang.Math.*;            // 通配：导入 Math 的所有静态成员
import static java.lang.Math.max;          // 精确：只导入 max 这一个方法

public class StaticImportDemo {
    public static void main(String[] args) {
        System.out.println(max(3, 5));     // 5，不用再写 Math.max
        System.out.println(abs(-7));       // 7
    }
}
```

静态导入确实能让测试代码更好读——`assertEquals(3, add(1, 2))` 比 `Assert.assertEquals(...)` 清爽得多。但它也是一把双刃剑：读者看到 `max(3, 5)` 时，无法一眼判断这个方法是从哪来的。

> **陷阱**：不要在生产代码里用 `import static xxx.*`。方法多起来之后，"这个 `copy()` 到底是哪个类的"会变成一场考古。静态导入的合理边界是——**只对语义极强、没有歧义的断言类方法使用**，比如 JUnit 的 `Assertions.*`、AssertJ 的 `assertThat`。

## 1.7 为什么 main 方法是 static 的

`public static void main(String[] args)` 这一行里的四个成分，每一个都不是随便加的：

| 成分 | 为什么必须有 |
|------|-----------|
| `public` | JVM 是外部调用者，权限必须开放到最大，否则 JVM 访问不到 |
| `static` | JVM 调用 `main` 时**还没有任何对象**，只能按"类名.方法"的方式调用，没有 `static` 就必须先构造对象——而构造对象又需要知道用哪个构造方法、传什么参数，JVM 无从下手 |
| `void` | 返回值是给调用方的，JVM 不需要 `main` 的返回值，返回值也就没有意义 |
| `String[] args` | 接收命令行参数，是 `main` 与外界交互的唯一入口 |

其中"`static` 的原因"最值得展开。启动流程是这样的：`java ArgsDemo hello world` 执行时，JVM 先启动一个虚拟机，找到 `ArgsDemo` 这个类、把它加载并初始化，然后**通过类名直接定位并调用 `main` 方法**——整个过程没有创建过任何 `ArgsDemo` 对象。如果 `main` 不是静态的，JVM 就得先知道"该 `new` 哪个对象、走哪个构造方法"，这显然不可能。

```java
public class ArgsDemo {
    public static void main(String[] args) {
        // 命令行：java ArgsDemo hello world
        System.out.println("参数个数：" + args.length);
        for (int i = 0; i < args.length; i++) {
            System.out.println("args[" + i + "] = " + args[i]);
        }
    }
}
```

```text
参数个数：2
args[0] = hello
args[1] = world
```

> **注意**：`main` 的签名**必须严格写成 `public static void main(String[] args)`**。把 `static` 去掉、把参数类型换成 `String` 或 `Object`、把返回类型改成 `int`，编译器都不会报错——因为你只是定义了一个"名字叫 main 的普通方法"。但 JVM 找不到合法的入口，运行时会报 `错误: 在类 xxx 中找不到 main 方法, 请将 main 方法定义为: public static void main(String[] args)`。

## 1.8 静态成员的典型误用

`static` 用起来很方便，也因此是最容易被滥用的关键字。三种误用值得单独提醒。

**第一种是把静态变量当成"全局变量"**。Java 没有全局变量，于是很多人用 `public static` 字段来充当。这在单线程的小程序里看不出问题，但一旦涉及多线程，静态变量就成了**共享可变状态**——所有线程读写同一份内存，是并发 bug 的高发地：

```java
public class Counter {
    public static int count = 0;      // 危险：所有线程共享同一份内存

    public static void increase() {
        count++;                      // 读 → 加一 → 写回，三步之间可能被其他线程打断
    }
}
```

修法有两条：改成成员变量（每个对象一份，天然隔离），或者保留静态但改用 `AtomicInteger`、对访问加锁。这部分原理在第 17 篇有完整展开。

**第二种是让静态集合无限增长**。静态变量的生命周期和类一致，基本等于"程序不结束就不回收"，所以往静态集合里塞数据等于把数据永久留在内存里：

```java
public class Cache {
    // 这个 Map 会一直活着，塞进去的对象永远不会被回收
    public static final Map<String, Object> CACHE = new HashMap<>();
}
```

需要缓存时应该用 `WeakHashMap`、`Caffeine` 这类会按策略淘汰的实现，而不是裸的 `HashMap`。

**第三种是让工具类带上状态**。工具类的价值在于"输入相同、输出必然相同"，一旦它有了静态可变状态，行为就变得不可预测，单元测试也会因为执行顺序不同而时好时坏。工具类的正确姿势是**只有静态常量，没有静态可变变量**。

> **设计考量**：把 `static` 的适用范围收缩到三种情况，基本就不会出错——**常量**（`static final`）、**无状态的工具方法**、**类级别的共享只读数据**。凡是"会被修改的共享数据"，都应该放进对象里，或者交给专门的并发容器管理。

# 2. 继承

## 2.1 继承的概念与 extends

**继承**描述的是现实世界里"**是一种**"（is-a）的关系：猫**是一种**动物，所以 `Cat` 可以继承 `Animal`。继承让子类自动拥有父类的非私有成员，从而做到**代码复用**；同时子类可以在父类的基础上**新增**成员或**改写**行为，做到**功能扩展**。

术语对照：被继承的类叫 **父类**（基类 / 超类 / superclass），继承的类叫 **子类**（派生类 / subclass）。语法只多了一个 `extends` 关键字：

```java
修饰符 class 子类名 extends 父类名 { }
```

判断两个类之间该不该用继承，有一个很实用的检验方法：**能用"B 是一种 A"读通，才适合用继承**。`Cat 是一种 Animal` 成立，`Cat extends Animal` 就合理；但 `OrderService 是一种 OrderDao` 明显读不通，这种"一个类需要用另一个类的能力"的关系应该用**组合**——在 `OrderService` 里持有一个 `OrderDao` 类型的字段，而不是去继承它。继承是**最强的耦合**（父类的任何改动都会波及所有子类），用错了代价很大。

## 2.2 继承了什么、没继承什么

"子类继承父类"这句话太笼统，真正要分清的是**每种成员各自的待遇**：

| 父类成员 | 是否被继承 | 子类能否直接使用 | 说明 |
|---------|-----------|----------------|------|
| `public` 成员变量 | 是 | 能 | 完全可用 |
| `protected` 成员 | 是 | 能（同包，或子类）| 跨包时只能通过子类对象访问 |
| 缺省（包私有）成员 | 是 | 仅限同包 | 一旦跨包就不可见 |
| `private` 成员 | **否** | 否 | 但对象内存里确实存在，只能通过父类的 `public` 方法间接使用 |
| 构造方法 | **否** | 不能用 `子类名()` 调用 | 但可以在子类构造方法里用 `super(...)` 调用 |
| 静态成员 | 随类传递 | 能（推荐用 `父类名.成员`）| 属于"隐藏"，**不是**"重写" |

其中 `private` 这一行最需要解释。说它"不能被继承"，指的是**子类代码里写不出 `this.age` 这样的访问**；但它"确实在子类对象里"，指的是**内存层面父类的私有字段仍然被分配了空间**。

```text
 一个 Cat 对象在堆中的内存分布（HotSpot，开启指针压缩）

 ┌──────────────────────────────┐
 │ 对象头：Mark Word（哈希/锁）   │
 ├──────────────────────────────┤
 │ 对象头：类型指针 → Cat.class  │
 ├──────────────────────────────┤
 │ Animal 的 private 字段 age    │  ← 子类"看不到"，但内存里确实有
 ├──────────────────────────────┤
 │ Animal 的 protected 字段 name │
 ├──────────────────────────────┤
 │ Cat 自己的字段 color          │  ← 子类新增的部分排在父类字段之后
 └──────────────────────────────┘
```

这段结论可以用一段代码直接验证：子类不去碰那个私有字段，却能从父类的方法里把它读出来。

```java
class Animal {
    private int age = 3;                 // 子类无法直接访问

    public int getAge() { return age; }   // 但父类自己的方法能访问
}

class Cat extends Animal {
    // 在这里写 this.age 会编译报错：age 在 Animal 中是 private 的
}

public class PrivateFieldDemo {
    public static void main(String[] args) {
        Cat cat = new Cat();
        System.out.println(cat.getAge());   // 3
    }
}
```

输出是 `3`，这说明 `age` 确实躺在 `cat` 这个对象里。因为 `getAge()` 定义在父类中，方法体里的 `this` 指向的是一个**运行时类型为 `Cat`** 的对象；如果 `Cat` 实例中没有 `age` 这块内存，这个 `3` 就无处可取。所以"内存上有、语法上看不到"才是准确的表述。

为什么要把 `private` 设计成"继承不到"？因为继承的语义是"**子类可以拿去用**"，而 `private` 的语义是"**只有我自己能用**"——两者天然冲突。更重要的是，如果子类能随意读写父类的私有字段，父类就再也无法安全地修改自己的内部实现，封装就失效了。

> **陷阱**：`private` 方法**不参与重写**。子类里写一个名字和参数都相同的方法，那只是"另起炉灶的新方法"，不是重写。当父类引用指向子类对象时，调用的依然是父类那个版本——因为编译器在编译期就把 `private` 方法的调用绑定死了（生成 `invokespecial` 指令，不走虚方法表）。同理，`static` 方法和 `final` 方法也都不能重写，`static` 只能叫**隐藏**。

## 2.3 单继承与多层继承

Java 的继承有三条硬规则：**一个类只能有一个直接父类**（单继承）、**可以有多层继承**（`A` extends `B` extends `C`）、**每个类都直接或间接继承 `Object`**（`Object` 是所有类的根，哪怕你不写 `extends`，编译器也会补上）。

```text
      ┌───────────────────────────────┐
      │  Object（java.lang.Object）    │   所有类的最终父类
      └───────────────┬───────────────┘
                      │ extends
      ┌───────────────▼───────────────┐
      │  Animal                       │   直接父类
      │  - name                       │
      │  + eat()                      │
      └───────────────┬───────────────┘
                      │ extends
      ┌───────────────▼───────────────┐
      │  Cat                          │
      │  + catchMouse()               │
      └───────────────┬───────────────┘
                      │ extends
      ┌───────────────▼───────────────┐
      │  PersianCat                   │   对 Animal、Object 而言都是间接子类
      │  + faceRound()                │
      └───────────────────────────────┘
```

多层继承带来的直接好处是**继承链上的方法都能用**：`PersianCat` 对象可以调用 `faceRound()`、`catchMouse()`、`eat()` 乃至 `Object` 的 `toString()`。但也要注意一个细节——**`super` 只能访问直接父类**，`PersianCat` 里写 `super.name` 找的是 `Cat`，如果 `name` 定义在 `Animal` 里，`super.name` 依然能访问到（沿链向上查找），但 `PersianCat` 想跳过 `Cat` 去找 `Animal` 的**同名**成员是不可能的。

既然所有类最终都继承 `Object`，那么每个对象天生就带着 `Object` 的方法，其中最常用的三个是：`toString()`（默认返回 `类名@十六进制哈希`，`println` 一个对象时会自动调用它）、`equals(Object)`（默认比较地址，重写后可以按内容比较）、`hashCode()`（决定对象在哈希表中的位置，配合 `equals` 使用）。这三者有一条铁律——**重写了 `equals` 就必须重写 `hashCode`**，否则把对象塞进 `HashSet` 或当作 `HashMap` 的键时，会出现"两个明明相等的对象却各占一格"的错误。

Java 不支持多继承，是为了避开 **C++ 的菱形继承问题**：如果 `B` 和 `C` 都继承 `A`，而 `D` 同时继承 `B` 和 `C`，那么 `D` 对象里到底有几份 `A` 的字段？调用 `D.eat()` 时该用 `B` 继承来的那份还是 `C` 的那份？

> **设计考量**：Java 的解法是**把多继承拆成两半**——"**类的实现单继承，接口的契约多继承**"。字段和方法体这类"实现"因为可能冲突，只允许单继承；而"具备某种能力"这类"契约"因为只是方法签名，允许多实现（见第 6 节）。这条设计贯穿了整个 Java 类库：`List` 是接口可以多实现，`AbstractList` 是类只能单继承。

## 2.4 成员变量的访问特点：就近原则

当父子类出现同名变量时，访问遵循**就近原则**：局部变量 > 本类成员变量 > 父类成员变量。想要打破就近原则，就得用 `this.` 和 `super.` 显式指定。

```java
class Father {
    int num = 10;
}

class Son extends Father {
    int num = 20;

    public void show() {
        int num = 30;                  // 局部变量
        System.out.println(num);        // 30 ← 最近的局部变量
        System.out.println(this.num);   // 20 ← 本类的成员变量
        System.out.println(super.num);  // 10 ← 父类的成员变量
    }
}

// new Son().show(); 的三个输出依次是：30、20、10
```

换个角度看，就近原则其实是"作用域由近及远"这条通用规则在继承体系上的自然延伸：局部变量在大括号里，成员变量在对象里，父类成员在父类那部分内存里——谁离使用点最近，就用谁。

如果层级再多一层，**间接父类的重名变量就彻底无法直接访问了**——`super` 只能向上一层，没有 `super.super` 这种语法。这时候唯一的办法是让父类提供一个 `getter` 方法。

成员变量还有一个极为重要的性质：**它不参与多态**。编译器遇到 `a.num` 时，直接按照 `a` 的**编译时类型**去类里定位字段，生成一条"固定偏移量"的访问指令；运行时**不会**因为对象实际是子类就去读子类的那个字段。

> **注意**：变量没有动态绑定可言，这是"**成员变量编译看左边、运行也看左边**"的根本原因。方法之所以能做到"运行看右边"，是因为 JVM 会查虚方法表；而字段访问走的是 `getfield` 加一个写死在字节码里的偏移量，没有任何查表的机会。这条结论和第 3.4 节完全对应。

## 2.5 成员方法的重写与 @Override

**重写（Override）**指子类定义了一个和父类**方法名、参数列表完全一致**的非静态方法，用自己的实现覆盖掉父类在虚方法表中的那一项。

重写的规则可以用一句口诀概括：**两同两小一大**。

| 口诀 | 具体规则 | 说明 |
|------|---------|------|
| 两同 | 方法名相同 | 不同就不是重写 |
| 两同 | 参数列表相同（类型、个数、顺序全一致）| 参数不同就是**重载**，编译能过但多态会失效 |
| 两小 | 返回值类型**小于等于**父类 | 只能是父类返回值的子类（协变返回，如 `Number` → `Integer`）|
| 两小 | 抛出的受检异常**小于等于**父类 | 不能抛更多、更宽的受检异常 |
| 一大 | 访问权限**大于等于**父类 | `public` > `protected` > 缺省 > `private` |

```java
class Animal {
    protected Number getAge() throws IOException {
        return 3;
    }
}

class Cat extends Animal {
    @Override
    public Integer getAge() {          // 权限：protected → public，变大 ✓
        return 3;                      // 返回值：Number → Integer，变小 ✓
    }                                  // 异常：抛出了更少的异常，变小 ✓
}
```

重写的**前提是父类方法必须是"虚方法"**。哪些方法不属于虚方法？**`final`、`private`、`static` 修饰的方法**，以及**构造方法**。构造方法之所以不能重写，原因很朴素：方法名必须与本类类名相同，子类的构造方法名字必然和父类不同，压根没有"同名"的可能。

`@Override` 注解的作用是**编译期校验**：

```java
class Animal2 {
    public void eat(String food) {
        System.out.println("动物吃" + food);
    }
}

class Cat2 extends Animal2 {
    // 意图是重写，但形参写成了 Object，实际是重载
    // @Override                       // ← 一旦加上这行，编译立刻报错，帮你发现笔误
    public void eat(Object food) {
        System.out.println("猫吃" + food);
    }
}

public class OverrideCheckDemo {
    public static void main(String[] args) {
        Animal2 a = new Cat2();
        a.eat("鱼");        // 输出：动物吃鱼 —— 因为 Cat2 根本没有重写 eat(String)
    }
}
```

这段代码的输出是 `动物吃鱼` 而不是 `猫吃鱼`，正是漏写 `@Override` 会带来的真实事故。`@Override` **本身不影响多态**（不写也能重写成功），它的价值全在于"**让编译器替你守住规则**"。

重写能够生效，靠的是 JVM 的**虚方法表（vtable）**。类加载的链接阶段，JVM 会为每个类建一张方法表：子类先**复制**父类的表，再把被重写的方法表项**替换**成自己的实现，最后追加自己新增的方法。

```text
        ┌──────────────────────────────────┐
        │ Animal 的虚方法表                 │
        │  eat()    ─▶ Animal.eat          │
        │  sleep()  ─▶ Animal.sleep        │
        └────────────────┬─────────────────┘
                         │ 加载 Cat 时复制并覆写表项
        ┌────────────────▼─────────────────┐
        │ Cat 的虚方法表                    │
        │  eat()    ─▶ Cat.eat      覆写    │
        │  sleep()  ─▶ Animal.sleep  继承   │
        │  catchMouse() ─▶ Cat.catchMouse  │
        └──────────────────────────────────┘
```

调用 `a.eat()` 时，字节码里的 `invokevirtual` 指令会拿"对象头的类型指针 → 类 → 虚方法表 → 表项"去查，查到谁就调谁。这就是"运行看右边"的硬件级实现。

> **设计考量**：虚方法表是典型的**空间换时间**——每个类多花一张表，换来每次调用 O(1) 的定位速度。而 `final`、`private`、`static` 方法不进表，JVM 可以用 `invokespecial`、`invokestatic` 直接绑定，甚至内联展开，所以它们的调用比虚方法略快。这就是"能加 `final` 就加 `final`"能带来一点点性能收益的底层原因——当然，这个收益小到绝大多数场景可以忽略，加 `final` 的主要动机仍然是**表达设计意图**。

规则里的"返回值变小"指的就是**协变返回类型**：子类重写时，可以把返回值类型改成父类返回值的**子类**。这是 JDK5 才放开的能力，它带来的好处是调用方不必再做多余的强制转换。

```java
class Animal {
    public Animal create() { return new Animal(); }
}

class Cat extends Animal {
    @Override
    public Cat create() {              // 返回值从 Animal 收窄成子类 Cat，合法
        return new Cat();
    }
}

public class CovariantDemo {
    public static void main(String[] args) {
        Cat c = new Cat().create();    // 不需要强转，直接拿到 Cat 类型
        System.out.println(c.getClass().getSimpleName());   // Cat
    }
}
```

如果子类不这样收窄，`new Cat().create()` 的返回类型就是 `Animal`，调用方想用 `Cat` 特有的方法就必须再写一次 `(Cat)`。所以很多库在设计 API 时会有意让 `create()`、`clone()`、`builder()` 这类方法使用协变返回，把便利留给调用者。

## 2.6 重写与重载的对比

初学者最容易混淆的一对概念。它们的区别可以浓缩成一句话：**重载是"同一个类里方法名复用"，重写是"子类改写父类的行为"**。

| 维度 | 重写（Override） | 重载（Overload） |
|------|-----------------|-----------------|
| 发生位置 | 子类与父类之间 | 同一个类中（子类也可新增重载）|
| 方法名 | 必须相同 | 必须相同 |
| 参数列表 | 必须相同 | **必须不同**（类型 / 个数 / 顺序至少一个不同）|
| 返回值类型 | 只能变小或相同 | 无任何要求 |
| 访问权限 | 只能变大或相同 | 无任何要求 |
| 绑定时机 | **运行期**（动态绑定，查虚方法表）| **编译期**（静态绑定，按实参静态类型选）|
| 与多态的关系 | 是多态能够生效的前提 | 与多态无关 |
| 常见现象 | `Animal a = new Cat(); a.eat()` 输出猫的行为 | 同一个对象因静态类型不同而调用不同版本 |

重载是编译期决定的，这一点可以用代码直接证明——**同一个对象，只因为变量的编译时类型不同，调用的重载版本就不同**：

```java
class Animal { }
class Cat extends Animal { }

class Printer {
    public void print(Animal a) { System.out.println("打印动物"); }
    public void print(Cat c)    { System.out.println("打印猫"); }
}

public class OverloadDemo {
    public static void main(String[] args) {
        Cat cat = new Cat();
        Animal ani = cat;              // 同一个对象，两个不同的静态类型

        Printer p = new Printer();
        p.print(cat);                  // 打印猫    ← 实参的静态类型是 Cat
        p.print(ani);                  // 打印动物  ← 同一个对象，静态类型是 Animal
    }
}
```

两次调用传入的是**同一个对象**，输出却不同，因为 `javac` 在编译期就已经把调用的方法写死在字节码里了。这也解释了一个常见的困惑："我明明重载了，为什么多态没生效？"——因为**多态只作用于重写**。

## 2.7 this 与 super 的完整用法

`this` 和 `super` 是访问类成员的两种"定向指针"，但它们的本质完全不同：**`this` 是一个引用变量，指向当前对象；`super` 是父类存储空间的标识，它不是对象**。

| 用法 | `this` | `super` |
|------|--------|---------|
| 访问成员变量 | `this.变量`：本类的（含继承来的）| `super.变量`：直接父类的 |
| 调用成员方法 | `this.方法()`：先本类再父类 | `super.方法()`：跳过本类，直接去父类找 |
| 调用构造方法 | `this(参数)`：调用本类其他构造方法 | `super(参数)`：调用父类的构造方法 |
| 本质 | 一个引用变量，指向当前对象 | 关键字，代表父类存储空间，**不是对象** |
| 能否单独使用 | `return this;` 合法（链式调用）| `return super;` 编译报错 |
| 位置要求 | `this(...)` 必须是构造方法第一行 | `super(...)` 必须是构造方法第一行 |

```java
class Father2 {
    int num = 10;

    public void show() {
        System.out.println("父类 show");
    }
}

class Son2 extends Father2 {
    int num = 20;

    @Override
    public void show() {
        super.show();                     // 先执行父类的版本
        System.out.println("子类 show");
        System.out.println(this.num);     // 20
        System.out.println(super.num);    // 10
    }
}
```

关于**构造方法互相调用**，有三条必须记牢的规则：

1. **`this(...)` 和 `super(...)` 不能同时出现在一个构造方法里**——因为它们都被要求写在第一行，一个方法只有一个第一行；
2. **一旦用了 `this(...)`，就不会再自动补 `super()`**。因为 `this(...)` 最终会链到本类的某个构造方法，而那个构造方法里必然（显式或隐式）调用 `super()`，父类初始化依然会发生，只是延后了一步；
3. **`super(...)` 只能调用直接父类的构造方法**，且**它不是"创建父类对象"**。很多初学者以为 `super()` 相当于 `new Father2()`，其实创建出来的只有一个 `Son2` 对象，`super(...)` 只是借用父类的构造方法**初始化这个对象里属于父类的那部分内存**。

```java
class Son3 extends Father2 {
    public Son3() {
        this(5);                              // 调用本类的另一个构造方法
        System.out.println("无参构造结束");
    }

    public Son3(int n) {
        super();                              // 这里才真正调父类构造（也可以不写，编译器会补）
        this.num = n;
        System.out.println("有参构造结束，num = " + n);
    }
}

// new Son3(); 的输出顺序：
// 有参构造结束，num = 5
// 无参构造结束
```

## 2.8 构造方法的继承与调用

构造方法**不能被继承**——因为方法名必须与类名相同，`Animal` 的构造方法叫 `Animal`，`Cat` 里根本写不出同名的构造方法。但构造方法**可以被调用**，通道就是 `super(...)`。

子类构造方法的第一行有一条**隐式的 `super()`**，这是 `javac` 自动补上的：

```java
public class Cat extends Animal {
    public Cat() {
        super();          // 编译器自动补的，你写不写都会执行
        // 子类自己的初始化...
    }
}
```

由此引出一条非常常见的编译错误：**父类没有无参构造方法时，子类必须显式 `super(...)`**。

```java
class Animal3 {
    private String name;

    // 一旦手动写了任意构造方法，编译器就不再生成默认无参构造
    public Animal3(String name) {
        this.name = name;
    }

    public String getName() { return name; }
}

class Cat4 extends Animal3 {
    public Cat4(String name) {
        super(name);          // 必须写在第一行，否则编译报错
    }
}

public class SuperDemo {
    public static void main(String[] args) {
        Cat4 cat = new Cat4("小白");
        System.out.println(cat.getName());   // 小白 —— 名字是在父类构造里赋上的
    }
}
```

如果把 `super(name)` 删掉或挪到第二行，编译器立刻报 `constructor Animal3 in class Animal3 cannot be applied to given types`（找不到可用的父类无参构造）或 `call to super must be first statement in constructor`（`super` 必须是第一条语句）。

为什么 `super(...)` 一定要放在第一行？因为**对象的内存是先分配父类那部分、再分配子类那部分的**，初始化也必须按这个顺序来：先让父类把属于它的状态准备好，子类的初始化代码才有可能依赖父类的状态。如果允许先写子类的代码再调 `super()`，就会出现"子类已经开始用 `this.name` 了，而 `name` 还在默认值 `null`"的荒谬局面。

> **陷阱**：只要你在父类写了任何一个构造方法，编译器就**不再**为你生成默认的无参构造。这是初学阶段最高频的编译错误来源。最省事的做法是**给父类补一个无参构造**；如果业务上确实要求必须传参，那就在每个子类构造方法的第一行老老实实写 `super(...)`。

## 2.9 构造方法的执行顺序

这是本篇最值得用代码跑一遍的知识点。完整顺序是：

```text
 父类静态代码块
      ↓
 子类静态代码块
      ↓
 父类构造代码块 / 父类实例变量显式初始化（按书写顺序）
      ↓
 父类构造方法
      ↓
 子类构造代码块 / 子类实例变量显式初始化（按书写顺序）
      ↓
 子类构造方法
```

注意最后一步是"子类构造方法**体**"，因为 `super(...)` 那一行已经在第 4 步完成了父类构造。下面这段代码把这个顺序完整打印出来：

```java
class Father5 {
    static { System.out.println("1 父类静态代码块"); }

    { System.out.println("3 父类构造代码块"); }

    private int a = printA();          // 实例变量显式初始化，与构造代码块按书写顺序执行

    private int printA() {
        System.out.println("4 父类实例变量显式初始化");
        return 1;
    }

    public Father5() {
        System.out.println("5 父类构造方法");
    }
}

class Son5 extends Father5 {
    static { System.out.println("2 子类静态代码块"); }

    { System.out.println("6 子类构造代码块"); }

    public Son5() {
        // 这里隐藏了一句 super()，所以父类的实例初始化先跑完
        System.out.println("7 子类构造方法");
    }
}

public class InitOrderDemo {
    public static void main(String[] args) {
        System.out.println("=== 第一次 new ===");
        new Son5();
        System.out.println("=== 第二次 new ===");
        new Son5();
    }
}
```

```text
=== 第一次 new ===
1 父类静态代码块
2 子类静态代码块
3 父类构造代码块
4 父类实例变量显式初始化
5 父类构造方法
6 子类构造代码块
7 子类构造方法
=== 第二次 new ===
3 父类构造代码块
4 父类实例变量显式初始化
5 父类构造方法
6 子类构造代码块
7 子类构造方法
```

从这个输出能抽出四条规律：

1. **静态的部分只和"类加载"有关**：`1`、`2` 只在第一次出现，且父类先于子类。第二次 `new` 时它们不再执行，因为类早就加载好了；
2. **实例的部分每次 `new` 都跑一遍**：`3` 到 `7` 重复出现；
3. **同一个类内部，构造代码块和实例变量显式初始化按书写顺序执行**，且都排在构造方法体之前。上面把构造代码块写在字段前面，所以 `3` 在 `4` 之前；
4. **父类的构造方法体一定在子类构造方法体之前执行完**，因为子类构造的第一行隐含着 `super()`——控制流会先跳到父类，跑完再跳回来。

这个顺序里藏着一个很容易踩的坑：**在父类构造方法里调用了会被子类重写的方法**。

```java
class Father6 {
    public Father6() {
        print();          // 危险：这里会调用到子类重写后的版本
    }

    public void print() {
        System.out.println("父类 print");
    }
}

class Son6 extends Father6 {
    private int value = 100;

    public Son6() { }

    @Override
    public void print() {
        // 期望 value 是 100，实际是 0 —— 因为子类的字段初始化还没执行
        System.out.println("子类 print，此时 value = " + value);
    }
}

// new Son6(); 的实际输出：子类 print，此时 value = 0
```

原因还是那个顺序：父类构造方法执行时，子类的构造代码块和字段初始化**还没轮到**，`value` 尚处于准备阶段的默认值 `0`。JDK 内部不少类（早期 `HashMap` 的实现讨论）都因为这一点被反复拿出来当反面教材。

> **设计考量**：这条坑总结成编码规范就是"**构造方法里只做简单的字段赋值，不要调用可以被重写的方法**"。如果确实需要，就把这类方法声明成 `private` 或 `final`，主动切断被重写的可能。

# 3. 多态

## 3.1 多态的概念与三个前提

**多态**指同一个方法调用，在运行时根据对象的实际类型表现出不同的行为。它是面向对象里最"值钱"的特性——正是因为多态，调用方才能只依赖抽象的父类或接口，完全不关心背后是哪个具体实现。

多态要成立需要三个前提：

| 前提 | 含义 | 不满足会怎样 |
|------|------|------------|
| 有继承或实现关系 | 父类与子类、接口与实现类 | 两个毫无关系的类之间无法互相赋值 |
| 有父类引用指向子类对象 | 也就是**向上转型** | 子类引用可以自动转父类引用，反之需要强转 |
| 有方法重写 | 父类声明了该方法，子类改写了它的实现 | 编译仍能通过，但调用结果永远与父类一致，等于"多态失效" |

第三条需要精确理解：严格来说，**只要父类声明了同名同参的方法，`父类引用.方法()` 就能编译通过**；重写的意义在于**运行时执行的是子类版本，产生了"不同对象不同行为"的效果**。所以重写不是"语法前提"，而是"多态有意义的必要条件"。

三条前提里，前两条是语法层面的硬约束（不满足根本编译不过），第三条是**语义层面的软约束**。如果只有前两条而没有重写，代码照样能跑，只是 `a.eat()` 永远执行父类的那个版本——多态就成了"形式上有、效果上没有"。

## 3.2 多态的格式

```java
父类类型 对象名 = new 子类类型();
```

```java
class Animal {
    public void eat() { System.out.println("动物吃东西"); }
}

class Cat extends Animal {
    @Override
    public void eat() { System.out.println("猫吃鱼"); }
}

class Dog extends Animal {
    @Override
    public void eat() { System.out.println("狗啃骨头"); }
}

public class PolyFormatDemo {
    public static void main(String[] args) {
        Animal a1 = new Cat();     // 向上转型：把猫当成动物看
        Animal a2 = new Dog();     // 同一句代码的形式，装不同的对象

        a1.eat();                  // 猫吃鱼
        a2.eat();                  // 狗啃骨头
    }
}
```

除了直接赋值，多态还有两种更常见的用法——**父类作方法参数**和**父类作方法返回值**：

```java
public static void feed(Animal a) {   // 形参是父类
    a.eat();
}

public static Animal create(int type) {   // 返回值是父类
    return type == 1 ? new Cat() : new Dog();
}
```

## 3.3 编译看左边、运行看右边

多态的核心原理只有八个字：**编译看左边，运行看右边**。它解决的是一个矛盾——**编译器只知道变量的类型，JVM 只知道对象的类型**。

```text
        Animal a = new Cat();
        ──────      ─────────
      编译时类型      运行时类型
         Animal          Cat
            │              │
            ▼              ▼
      javac 只允许调用     invokevirtual 指令去
      Animal 中声明过的    Cat 的虚方法表里查真正的
      方法，否则编译报错   实现，查不到就沿继承链向上找
```

把这两侧的职责分清楚，所有多态的"怪现象"都能解释：

- **编译期**：`javac` 看的是变量 `a` 的声明类型 `Animal`。所以 `a.catchMouse()` 会直接报错——`Animal` 类里没有这个方法，编译器根本不知道运行时会是什么；
- **运行期**：字节码执行到 `a.eat()` 时，JVM 拿到的是**堆上那个对象的真实类型** `Cat`，于是去 `Cat` 的虚方法表里找到 `Cat.eat` 并执行。

那么编译器为什么不干脆"按右边来"，一开始就允许调用子类的方法？因为**编译一个类的时候，它的子类可能还没被写出来**，而且未来会被写出来多少个、叫什么名字都是未知的。编译器无法枚举"所有可能被赋给这个变量的类型"，也就无法知道该检查哪些方法。唯一的出路是拿"**声明类型**"作为上界做类型检查，把具体类型的解析完全推迟到运行期——这是所有静态类型语言支持动态分派的通用做法。

```java
class Animal {
    public String name = "动物";

    public void eat() { System.out.println("动物吃东西"); }
    public void sleep() { System.out.println("动物睡觉"); }
}

class Cat extends Animal {
    public String name = "猫";

    @Override
    public void eat() { System.out.println("猫吃鱼"); }

    public void catchMouse() { System.out.println("猫抓老鼠"); }
}

public class PolyRuleDemo {
    public static void main(String[] args) {
        Animal a = new Cat();              // 编译时 Animal，运行时 Cat

        System.out.println(a.name);        // 动物  ← 变量看左边
        a.eat();                           // 猫吃鱼 ← 方法看右边（子类重写了）
        a.sleep();                         // 动物睡觉 ← 子类没重写，向上找到父类的版本
        // a.catchMouse();                 // ✗ 编译错误：找不到符号 catchMouse
        //                                  //   因为 Animal 类里根本没有这个方法
    }
}
```

这段运行的输出依次是 `动物`、`猫吃鱼`、`动物睡觉`——三行代码用同一个变量 `a`，却分别体现了变量看左边、方法看右边、以及"没重写就向上找"三条不同的规则。

## 3.4 成员变量与成员方法的访问规则差异

多态下访问不同成员，规则并不统一，这是最容易记混的地方：

| 访问的内容 | 编译期检查 | 运行期执行 | 一句话记忆 |
|-----------|-----------|-----------|-----------|
| 成员变量 | 看左边（父类有没有该变量）| **看左边**（访问父类的那个）| 变量不参与多态 |
| 成员方法 | 看左边（父类有没有该方法）| **看右边**（子类重写后的版本）| 方法查虚方法表 |
| 静态方法 | 看左边 | **看左边**（静态方法不参与动态绑定）| 静态方法被"隐藏"而非重写 |
| 静态变量 | 看左边 | 看左边 | 同成员变量 |

成员变量为什么"看左边"？因为字段访问在字节码里是 `getfield` 加上一个**写死的偏移量**，编译期就定好了读对象内存的哪个位置，运行时没有查表的机会。而方法调用是 `invokevirtual`，会走虚方法表，才有"看右边"的可能。

静态方法为什么也"看左边"？因为静态方法属于**类**而不是对象，编译期就能确定调用哪个类的版本（生成 `invokestatic` 指令），JVM 不需要、也不能根据对象的实际类型去查。

```java
class Shape {
    public static void describe() { System.out.println("这是形状"); }
    public void draw() { System.out.println("画形状"); }
}

class Circle extends Shape {
    // 这不是重写，只是"隐藏"了父类的同名静态方法
    public static void describe() { System.out.println("这是圆"); }

    @Override
    public void draw() { System.out.println("画圆"); }
}

public class StaticPolyDemo {
    public static void main(String[] args) {
        Shape s = new Circle();
        s.describe();      // 这是形状 ← 静态方法编译运行都看左边
        s.draw();          // 画圆     ← 实例方法运行看右边
    }
}
```

## 3.5 多态的优势

多态带来的好处可以归纳成三点：

**第一是解耦**。调用方写的是 `Animal a`，它不需要知道、也不需要导入 `Cat`、`Dog`、`Bird` 这些具体类。将来 `Animal` 的实现换成别的子类，调用方一行都不用改。

**第二是可扩展**。新增一个 `Bird` 子类时，`feed(Animal a)` 这个方法不需要任何修改就自动支持它——这就是常说的"**对扩展开放，对修改关闭**"。

**第三是可以把父类当参数和返回值用**，一行代码处理任意多个子类对象。

```java
public class Zoo {
    public static void feed(Animal a) {    // 形参是父类，能接收任意子类对象
        a.eat();
    }

    public static void main(String[] args) {
        Animal[] animals = { new Cat(), new Dog(), new Cat() };
        for (Animal a : animals) {
            feed(a);       // 同一行代码，输出三种不同行为
        }
    }
}
```

```text
猫吃鱼
狗啃骨头
猫吃鱼
```

在真实工程里，多态几乎无处不在：`List<Animal> list = new ArrayList<>()` 用的是集合接口多态；JDBC 的 `Connection conn = DriverManager.getConnection(...)` 拿到的是一个我们从未见过的具体实现类，却能用统一接口操作 MySQL、Oracle、PostgreSQL；Spring 的依赖注入本质上就是"声明接口类型，运行时注入实现类对象"。

> **行业实践**：可以说，**面向接口编程的所有好处都来自多态**。团队编码规范里"依赖抽象不依赖具体"这一条，落地手段就是——方法的参数和返回值尽量声明成接口或抽象类，具体实现类只在 `new` 的那一刻出现。

## 3.6 多态的弊端与向下转型

多态的弊端只有一条，但它很关键：**不能调用子类特有的方法**。因为编译看左边，而父类里根本没有声明这些方法。

```java
Animal a = new Cat();
// a.catchMouse();          // ✗ 编译错误：Animal 中不存在 catchMouse
```

有人会问：既然多态下连子类特有的方法都调不了，那它的意义何在？答案是——**多态的目的从来不是"能调用所有方法"，而是"能统一处理一批对象"**。批量处理用多态，个别处理才需要转型。如果一段代码必须靠强转才能干活，往往说明这个职责不该由"面向父类的通用逻辑"承担，而应该把这段差异上移成父类的一个抽象方法，交给子类自己实现（见 3.7 节末尾的例子）。

解决办法是**向下转型**（也叫强制类型转换），把父类引用还原成子类类型：

```java
Cat c = (Cat) a;            // 向下转型：把"动物"还原成"猫"
c.catchMouse();             // ✓ 现在可以调用子类特有的方法了
```

转型的语法要写成括号放在变量前：`子类类型 变量名 = (子类类型) 父类引用;`。有一点必须明确：**转型改变的只是"编译器看待这个引用的方式"，并没有创建任何新对象**——`a` 和 `c` 指向的仍然是堆上同一个 `Cat` 实例。所以向下转型是一个**零开销**的操作，不需要担心性能。

## 3.7 instanceof 与 ClassCastException

向下转型有一个前提：**这个引用指向的对象，真的得是目标类型**。否则运行时会抛 `ClassCastException`。

```java
import java.util.Date;

public class CastFailDemo {
    public static void main(String[] args) {
        Animal a = new Dog();      // 实际是一只狗
        Cat c = (Cat) a;           // 编译能过：Dog 和 Cat 有共同父类，编译器认为"有转的可能"
        // 运行到上一行就崩了：
        // Exception in thread "main" java.lang.ClassCastException:
        //     class Dog cannot be cast to class Cat
    }
}
```

为什么编译期不拦住它？因为强制转换是程序员向编译器做的一个**承诺**："我知道它其实是 `Cat`，你放行"。编译器只检查**两个类型之间是否存在继承关系**（存在就说明"理论上有转成功的可能"），至于这个对象实际上是什么，编译期无从得知——它可能是从数据库、从网络、从方法返回值传过来的。所以这个检查只能推迟到运行期，由 JVM 在 `checkcast` 指令处验证。

正确的做法是**转型前先用 `instanceof` 判断**：

```java
public class InstanceofDemo {
    public static void main(String[] args) {
        Animal[] animals = { new Cat(), new Dog() };

        for (Animal a : animals) {
            if (a instanceof Cat) {         // 先判断，再转型
                Cat c = (Cat) a;
                c.catchMouse();             // 猫抓老鼠
            } else if (a instanceof Dog) {
                System.out.println("这是一只狗，跳过");
            }
        }
    }
}
```

JDK14 引入了**模式匹配的 `instanceof`**，把"判断 + 强转 + 声明变量"三步合成一步：

```java
for (Animal a : animals) {
    if (a instanceof Cat c) {      // 判断成功后，直接把 a 转成 Cat 类型的变量 c
        c.catchMouse();            // c 的作用域被限制在 if 为真的分支内，出了分支就不能用
    }
}
```

模式匹配的好处是**消灭了样板代码**，也顺手消灭了"判断用 `Cat`、强转却写成 `Dog`"这类笔误。完整的语法细节（作用域规则、与 `&&` 结合、`!(x instanceof T)` 的反向写法）见第 29 篇。

不过要提醒的是，**代码里大量出现 `instanceof` 本身就是一个坏味道**：它说明父类没有把差异抽象成方法，调用方只好自己按类型分支。更符合面向对象思路的做法是把差异上移，让多态自己去分发。

```java
// 坏味道：调用方替每个子类写分支，新增子类还得回来改这里
if (a instanceof Cat c) {
    c.catchMouse();
} else if (a instanceof Dog d) {
    d.watchDoor();
}

// 更好的设计：把"我能做什么"上移到父类，让每个子类自己实现
abstract class Animal {
    public abstract void doWork();     // 猫抓老鼠、狗看门，各写各的
}

// 调用方只剩一行，新增子类完全不用改
for (Animal a : animals) {
    a.doWork();
}
```

> **行业实践**：判断标准很简单——**类型分支是"天然的、封闭的"（如解析固定几种报文格式），可以用 `instanceof`；类型分支是"会不断新增的"（每加一个子类就要回来加一条 `else if`），就必须改成多态**。前者是数据驱动的解析，后者是设计缺陷。

> **注意**：`instanceof` 对 `null` 永远返回 `false`——`null instanceof Cat` 是 `false` 而不是报错。所以它天然就能防住空指针，不需要额外写 `a != null`。另外，如果两个类型的编译期类型之间**根本不存在继承关系**（例如 `String s = (String) new Integer(1);`），编译器会直接报错 `不可转换的类型`，这种情况根本到不了运行期。

# 4. 包、final、权限修饰符、代码块

## 4.1 包

**包（package）**本质就是操作系统的文件夹，在语言层面提供三个作用：

1. **解决类名冲突**——两个人都写了 `Date` 类，只要包名不同就相安无事；
2. **便于管理**——按功能分目录，找代码有迹可循；
3. **配合权限修饰符控制访问范围**——缺省权限和 `protected` 都离不开"包"的概念。

包名的命名规范是**公司域名反写 + 项目名 + 模块名**，全部小写、多级之间用 `.` 分隔：

```java
package com.itheima.order.service;      // 公司域名 itheima.com 反写 + 项目 + 模块
```

几个约定俗成的讲究：包名全部小写（不要用 `com.itheima.MyPackage` 这类大小写混排）；不要以 `java.` 开头（这是 JDK 保留的，写了会被类加载器拒绝）；`package` 语句必须是文件里第一条有效语句（注释之外），且一个源文件只能有一个。

在同一个包下出现**同名类**时，就必须使用**全类名**（包名 + 类名）来区分：

```java
public class DateDemo {
    public static void main(String[] args) {
        java.util.Date d1 = new java.util.Date();          // 通过全类名指定是哪一个 Date
        java.sql.Date d2 = new java.sql.Date(System.currentTimeMillis());
        System.out.println(d1.getClass().getName());       // java.util.Date
        System.out.println(d2.getClass().getName());       // java.sql.Date
    }
}
```

## 4.2 import 规则

使用其他包中的类需要**导包**，规则可以总结成一张表：

| 场景 | 是否需要导包 | 说明 |
|------|------------|------|
| 使用**同一个包**下的类 | 不需要 | 同包天然可见 |
| 使用 `java.lang` 包下的类 | 不需要 | `String`、`System`、`Math`、`Object` 等会被自动导入 |
| 使用其他包的类 | 需要 `import` | 如 `java.util.ArrayList` |
| 同时使用两个包中的**同名类** | 都不导，改用**全类名** | 或者只导一个，另一个写全类名 |
| 使用静态成员 | 可用 `import static` | 见 1.6 节 |

```java
package com.itheima.demo;                 // 声明包：必须最先写

import java.util.ArrayList;               // 单类型导入（推荐）
import java.util.List;                    // 需要几个就导几个

public class ImportDemo { }
```

如果同包中有一个类，和导入的另一个包中的类重名，**同包优先**——`import` 永远抢不过"同一个包里的类"。

> **行业实践**：尽量不用 `import java.util.*;` 这种通配符导入。它虽然省几行，但让读者无法一眼看出这个文件到底依赖了什么，而且将来两个包新增同名类时会莫名其妙地编译失败。IDEA 的"优化导入"功能默认就会把通配符展开成单类型导入，团队规范通常也直接禁止通配符导入。

还有一个容易被忽略的事实：**`import` 只影响编译期，不影响运行期**。编译生成的 `class` 文件中，所有类型都被写成了全类名，所以类加载过程和 `import` 怎么写没有任何关系。这也解释了一个现象——两个类互相 `import` 不会造成"循环依赖"错误，真正会带来麻烦的是它们互相持有对方的实例并参与构造。

## 4.3 final 关键字

`final` 表示"最终的、不可改变的"，它修饰不同对象时效果差别很大：

| 修饰对象 | 效果 | 典型用途 |
|---------|------|---------|
| 类 | 不能被继承 | `String`、`Integer` 等不可变类；防止子类破坏语义 |
| 方法 | 不能被重写 | 模板方法中不希望被改写的固定步骤 |
| 变量（基本类型）| 值只能赋值一次 | 常量 |
| 变量（引用类型）| **地址**只能赋值一次，对象内容**可变** | 见 4.4 节 |
| 成员变量 | 必须在声明时、构造代码块中或构造方法中完成赋值 | 常量、不可变字段 |

```java
public final class MathConstants {                  // 类不能被继承
    public static final double PI = 3.1415926;      // 常量：命名全大写，多词用下划线分隔
    public static final int MAX_RETRY_COUNT = 3;

    public final void check() { }                   // 方法不能被重写
}
```

**常量的命名规范**：单个单词全部大写（`PI`、`MAX`）；多个单词全部大写、用下划线分隔（`MAX_RETRY_COUNT`、`DEFAULT_TIMEOUT`）。这么写的目的就是让读者一眼区分"常量"和"变量"。

`final` 修饰成员变量时，**必须保证在对象构造完成之前被赋值一次**，赋值位置只有三处：声明时直接赋值、构造代码块中赋值、构造方法中赋值。三处都不写就会编译报错。

> **注意**：`static final` 修饰的**编译期常量**（值在编译时就能确定）会触发**常量折叠**——`javac` 会把用到它的地方直接替换成字面量。这意味着修改这个常量的值后，**所有引用了它的类都必须重新编译**，否则那些类里还是旧值。这是用 IDE 增量编译时偶尔遇到"改了常量却不生效"的原因。想避开这一点，就别用编译期常量，改用 `static final` 修饰一个运行时才能确定的值（比如 `UUID.randomUUID()` 的结果）。

## 4.4 final 修饰引用类型的关键区分

这是 `final` 最容易误解的地方，必须单独讲：**`final` 锁住的是变量里存的那个地址，而不是堆上的那个对象**。

```java
public class FinalReferenceDemo {
    public static void main(String[] args) {
        final int[] arr = {1, 2, 3};
        arr[0] = 100;                   // ✓ 合法：改的是对象内部的数据，地址没变
        System.out.println(arr[0]);      // 100
        // arr = new int[]{4, 5};       // ✗ 编译错误：无法为 final 变量赋值

        final List<String> list = new ArrayList<>();
        list.add("hello");               // ✓ 合法：列表内容可以变
        list.add("world");
        System.out.println(list);        // [hello, world]
        // list = new ArrayList<>();     // ✗ 编译错误：不能指向别的对象

        final StringBuilder sb = new StringBuilder("a");
        sb.append("b");                  // ✓ 合法：内容可以变
        System.out.println(sb);          // ab
    }
}
```

一句话记忆：**`final` 让"这个变量"不可变，不让"这个对象"不可变**。它实现的是"引用不可变"，不是"深度不可变"。

如果真的需要一个连内容都不能改的集合，得用别的手段：

```java
List<String> immutable1 = List.of("a", "b", "c");            // 不可变列表，add 会抛异常
List<String> immutable2 = Collections.unmodifiableList(list); // 只读视图：原 list 变了它也变
List<String> immutable3 = List.copyOf(list);                  // 真正复制一份，与原列表解耦
```

> **陷阱**：`final StringBuilder sb = new StringBuilder(); sb.append("x");` 是**完全合法**的，很多初学者会误以为它编译不过。反过来说，把 `final` 当成"线程安全"的保证也是错的——`final List` 里的元素照样可能被多个线程同时修改。

顺带说一句不可变对象的价值：`String` 之所以被设计成 `final` 且不可变，是因为它要当 `HashMap` 的键（哈希值不能在存入后变化）、要被字符串常量池共享（内容不变才敢共享）、要被用于类加载与网络连接等安全敏感场景（中途被改会造成安全隐患）。`final` 阻止了子类改写这些行为，不可变性又保证了状态不会被悄悄篡改——**这就是"不可变对象天生线程安全"这句话的由来**。

## 4.5 权限修饰符

权限修饰符用来**控制一个成员能被访问的范围**，可以修饰成员变量、成员方法、构造方法、内部类（**不能**修饰局部变量）。

| 修饰符 | 本类 | 同包其他类 | 不同包的子类 | 不同包无关类 |
|--------|------|-----------|------------|------------|
| `private` | ✓ | ✗ | ✗ | ✗ |
| 缺省（不写）| ✓ | ✓ | ✗ | ✗ |
| `protected` | ✓ | ✓ | ✓ | ✗ |
| `public` | ✓ | ✓ | ✓ | ✓ |

从范围看是层层扩大的：`private` ⊂ 缺省 ⊂ `protected` ⊂ `public`。实际使用时的选择依据很简单：

- **成员变量一律私有**（`private`），需要对外暴露就提供 `getter` / `setter`——这就是封装；
- **只想给同包用的工具方法**，用缺省或 `protected`；
- **想让所有子类能重写或复用**的方法，用 `protected`；
- **对外提供的 API、构造方法**，用 `public`。

`protected` 有一个非常隐蔽的细节：**不同包的子类中，只能通过"子类自己的实例"访问 `protected` 成员，不能通过父类实例访问**。

```java
// 文件一：package com.a;
package com.a;

public class Parent {
    protected int value = 1;
}

// 文件二：package com.b;
package com.b;

import com.a.Parent;

class Child extends Parent {
    void test() {
        System.out.println(this.value);        // ✓ 通过子类自己的实例访问
        Parent p = new Parent();
        // System.out.println(p.value);        // ✗ 编译错误：不能通过父类实例访问 protected 成员
    }
}
```

之所以这样设计，是因为 `protected` 的本意是"**让子类继承并复用**"，而不是"让子类窥探别的父类对象"。

## 4.6 类的权限修饰符

类本身**只能使用 `public` 和缺省**两种修饰符，不能是 `private` 或 `protected`——因为"包"是一级访问单位，而一个顶层类要么对所有包开放，要么只对本包开放：

| 修饰符 | 可见范围 |
|--------|---------|
| `public` | 任何包中的任何类都可以访问 |
| 缺省（不写）| 只能被**同一个包**中的类访问 |

同时还有一条编译规则：**一个 `.java` 文件里最多只能有一个 `public` 类，且它的名字必须和文件名完全一致**；文件里其他类都只能是缺省权限。这一条解释了为什么"一个文件写多个类"时，只有和文件名同名的那个能对外暴露。

实践上还有一条"最小可见性"原则：**对外暴露的 API 类才声明成 `public`，纯内部的辅助类保持缺省**。把内部实现类藏起来，调用方就没办法依赖到不该依赖的细节，将来重构这些类时也就不会"牵一发而动全身"。

> **行业实践**：Java 的标准做法是**一个文件一个 `public` 类**，文件名与类名一致。虽然语法上允许在同文件里塞若干缺省类，但这会让代码难以检索——`Find Class` 只能按文件名找。

## 4.7 三种代码块

Java 里有三种用大括号括起来、但执行时机完全不同的代码块：

| 类型 | 写法 | 位置 | 执行时机 | 执行次数 |
|------|------|------|---------|---------|
| 局部代码块 | `{ ... }` | 方法体内部 | 代码执行流到达时 | 每次到达都执行 |
| 构造代码块（实例初始化块）| `{ ... }` | 类中的成员位置 | 每次 `new` 对象时，**在构造方法体之前** | 每创建一个对象执行一次 |
| 静态代码块 | `static { ... }` | 类中的成员位置 | **类加载初始化时** | 整个 JVM 生命周期只执行一次 |

**局部代码块**的作用是缩小变量的作用域，让变量尽早失效、尽早被回收。现在有了更好的工具（把逻辑抽成独立方法），它已经很少用了：

```java
public void method() {
    {
        int temp = 100;         // temp 的作用域只有这个大括号
        System.out.println(temp);
    }
    // System.out.println(temp);   // ✗ 编译错误：找不到符号 temp
}
```

**构造代码块**用来抽取多个构造方法中重复的初始化代码：

```java
public class User {
    private String name;
    private String country;

    {                                    // 构造代码块：每个构造方法执行前都会先跑这里
        country = "中国";
        System.out.println("构造代码块执行");
    }

    public User() {
        System.out.println("无参构造执行");
    }

    public User(String name) {
        this.name = name;
        System.out.println("有参构造执行，name = " + name);
    }

    public static void main(String[] args) {
        new User();
        new User("张三");
    }
}
```

```text
构造代码块执行
无参构造执行
构造代码块执行
有参构造执行，name = 张三
```

不过构造代码块现在已经**不推荐使用**，因为同样的目的可以用构造方法互相调用（`this(...)`）实现，而且 `this(...)` 的意图更清晰、控制更灵活——构造代码块是"无条件执行"，而 `this(...)` 可以按需选择要链到哪个构造方法。

**静态代码块**的完整格式与用途见第 1.5 节，这里重点验证它的执行时机——它一定**早于 `main` 方法体**：

```java
public class StaticBlockDemo {
    static {
        System.out.println("静态代码块执行");
    }

    public static void main(String[] args) {
        System.out.println("main 开始");
        new StaticBlockDemo();
        new StaticBlockDemo();
    }
}
```

```text
静态代码块执行
main 开始
```

`main` 输出在静态代码块之后，原因很直接：**`main` 本身也是静态方法**，JVM 必须先完成类的初始化（也就是跑完静态代码块），然后才能调用 `main`。后面两次 `new` 没有再打印，证明了静态代码块**只执行一次**。

> **注意**：静态代码块里**只能访问静态成员**，因为执行时对象还不存在。此外它还有两条语法限制：不能写 `return`；不能抛出受检异常（编译器会把它塞进 `<clinit>()` 方法，受检异常在那里无处可抛，必须当场 `try-catch`）。如果想在静态代码块的 somewhere 失败时中止程序，只能抛非受检异常，或者干脆调用 `System.exit()`。

三种代码块的执行时机可以串成一条时间线：**类加载 → 静态代码块 → （每次 `new` 时）构造代码块 → 构造方法**。记住这条线，2.9 节那张跨父子类的执行顺序表其实就是它在继承链上的展开。

# 5. 抽象类

## 5.1 抽象方法

**抽象方法**是"只有声明、没有方法体"的方法，格式是在返回值类型前加 `abstract`，并以分号结尾：

```java
修饰符 abstract 返回值类型 方法名(参数列表);
```

```java
public abstract void eat();       // 注意：末尾是分号，没有大括号
```

抽象方法的语法很"轻"——一个 `abstract` 关键字加一个分号，但它表达的约束很重：**任何一个具体子类都必须给它一个实现**。子类只要漏掉一个，编译器就直接报错，除非这个子类自己也声明成抽象类。

抽象方法的出现解决了一个真实的设计困境：父类知道"**所有子类都必须能做这件事**"，但说不出"**具体怎么做**"。比如 `Animal` 明确知道每只动物都要吃东西，但"吃"这件事对猫是吃鱼、对狗是啃骨头，父类既写不出一个合理的默认实现，又必须强制子类提供实现。抽象方法就是为这种场景设计的语法：**只规定"必须有"，不规定"怎么做"**。

## 5.2 抽象类的定义

**如果一个类中存在抽象方法，那么这个类必须声明为抽象类**——因为抽象方法没有方法体，如果这个类还能被 `new`，那么调用到这个方法时就无代码可执行了。

```java
修饰符 abstract class 类名 { }
```

反过来说，**抽象类中不一定有抽象方法**。这一条经常被忽略：抽象类的第二种用途是"**我不想让这个类被实例化，但它本身的功能是完整的**"。例如 JDK 里的 `AbstractList`、`AbstractCollection` 都是这样——它们提供骨架实现，同时用抽象类这个身份阻止你直接 `new`。

## 5.3 抽象类的注意事项

| 结论 | 原因 |
|------|------|
| 抽象类**不能实例化** | 抽象方法没有方法体，`new` 出来的对象在语义和运行时都不完整 |
| 抽象类**可以有构造方法** | 供子类通过 `super(...)` 调用，用来初始化父类那部分成员 |
| 抽象类可以有普通成员变量、普通方法、静态成员、常量 | 它终究是一个类，只是"不完整"而已 |
| 子类要么**重写全部抽象方法**，要么自己**也是抽象类** | 只要还剩一个没实现，子类就同样"不完整" |
| 抽象类**不能有** `static` 的抽象方法 | 静态方法属于类、不参与动态绑定，谈不上"交给子类实现" |

```java
abstract class Animal {
    private String name;

    public Animal(String name) {             // 抽象类可以有构造方法
        this.name = name;
        System.out.println("Animal 构造方法执行，name = " + name);
    }

    public abstract void eat();              // 抽象方法：交给子类实现

    public void sleep() {                    // 普通方法：所有子类直接复用
        System.out.println(name + " 在睡觉");
    }

    public String getName() { return name; }
}

class Cat extends Animal {
    public Cat(String name) {
        super(name);                          // 子类通过 super 调用抽象类的构造方法
    }

    @Override
    public void eat() {                       // 重写全部抽象方法
        System.out.println(getName() + " 吃鱼");
    }
}

public class AbstractDemo {
    public static void main(String[] args) {
        // Animal a = new Animal("x");         // ✗ 编译错误：Animal 是抽象的，无法实例化

        Animal a = new Cat("小白");            // ✓ 多态：父类引用指向子类对象
        a.eat();                              // 小白 吃鱼
        a.sleep();                            // 小白 在睡觉
    }
}
```

输出里 `小白 吃鱼` 证明了两件事：抽象类的构造方法确实被调用了（`name` 有值），并且 `eat()` 执行的是子类的实现。同时注释掉的那一行也证明了**抽象类不能 `new`**——但它的构造方法依然有用，因为子类构造的第一行永远要 `super(...)`。

从 JVM 角度看，"抽象类不能实例化"不只是 `javac` 的一句约定，而是**在字节码层面被禁止**的：`new` 一个对象对应字节码里的 `new` 指令，JVM 会去常量池解析出这个类的构造方法引用；抽象类的构造方法在 `Class` 文件里带着 `ACC_ABSTRACT` 标志，校验阶段就会被拒绝。所以即使绕过编译器也 `new` 不出来——用反射调 `newInstance()` 会收到 `InstantiationException`，手写字节码触发 `new` 指令则直接抛 `InstantiationError`。

> **设计考量**：抽象类的构造方法是一个容易被误解的设计。它不是"用来创建抽象类对象"的（那被禁止了），而是**父类给自己留的一个初始化入口**——子类 `new` 出来的对象内存里包含父类那部分字段，这些字段只能由父类的构造方法来初始化。这也解释了为什么抽象类的构造方法通常声明成 `protected`：对外没有意义，对子类才开放。

## 5.4 abstract 不能与哪些修饰符共存

`abstract` 表达的语义是"**必须被继承 / 必须被重写**"，所以任何否定这个语义的修饰符都与它冲突：

| 修饰符 | 能否与 `abstract` 共存 | 冲突原因 |
|--------|---------------------|---------|
| `final` | ✗ | `final` 类不能被子类继承，而抽象类**必须**被继承；`final` 方法不能被重写，而抽象方法**必须**被重写 |
| `private` | ✗ | `private` 成员子类根本看不见，抽象方法若私有就永远不可能有实现 |
| `static` | ✗ | 静态方法属于类、不参与动态绑定，"由子类实现"这个语义无从落地 |
| `native` | ✗ | 两者语义直接矛盾：一个说"没有方法体"，一个说"方法体在别的地方" |
| `synchronized` | ✗ | 加锁是方法体的行为，没有方法体就无从加锁 |
| `strictfp` | ✗ | 浮点严格模式也是方法体的属性 |

反过来，`abstract` 可以和 `public`、`protected`、缺省权限共存，因为这些修饰符都不否定"被重写"这个前提。

```java
// 以下写法全部编译报错
// public abstract final class A { }
// private abstract void m();
// public static abstract void m();
```

> **陷阱**：最常见的组合错误是 `abstract` + `final`。初学时常有人想"这个抽象类我不想让别人改"，于是加上 `final`，结果编译器直接拒绝。正确的表达方式是——**如果不想被继承，就不要声明成抽象类**；抽象类这个身份本身就意味着"等着被继承"。

## 5.5 抽象类的设计意义

抽象类最大的价值在于承载**模板方法模式**：父类把**固定不变的流程**写死，把**会变化的部分**声明成抽象方法交给子类。

```java
abstract class Game {
    // 模板方法：整体流程固定，用 final 防止子类打乱顺序
    public final void play() {
        init();
        start();
        end();
    }

    protected abstract void init();            // 变化点一
    protected abstract void start();           // 变化点二

    protected void end() {                     // 钩子方法：有默认实现，子类可选择重写
        System.out.println("游戏结束");
    }
}

class BasketBall extends Game {
    @Override protected void init() { System.out.println("篮球：热身"); }
    @Override protected void start() { System.out.println("篮球：比赛中"); }
}

// new BasketBall().play(); 依次输出：篮球：热身 / 篮球：比赛中 / 游戏结束
```

JDK 与各大框架里到处是这种模式：`InputStream` 把 `read()` 声明成抽象方法，由子类决定数据从哪来，而 `read(byte[])`、`skip()` 等通用流程由抽象类自己实现；`HttpServlet` 的 `service()` 负责分发 HTTP 请求，把 `doGet()`、`doPost()` 留给开发者；JUnit 的 `TestCase`、Spring 的 `AbstractApplicationContext` 也是同一个套路。

模板方法的价值在于**把"变化"收敛到一个点上**。如果没有抽象类，`BasketBall`、`FootBall`、`Swimming` 各自都要把 `play()` 的流程抄一遍；将来流程变一步（比如开始前要先"加载资源"），就得回头改 N 个类。有了模板方法，只需要改父类那一处，所有子类自动跟上——这正是开闭原则里"**对修改关闭、对扩展开放**"的落地方式。

反过来也要注意：模板方法把控制权从子类手里收走了（子类只能填变化点，不能改流程）。如果某个子类确实需要一套完全不同的流程，硬塞进这个模板只会越来越别扭——这时候就该考虑它是不是另一个抽象类了。

> **设计考量**：抽象类的本质是"**用继承表达"是一种"关系，并向上抽取共性**"。它和接口最大的区别在于**抽象类可以持有状态（实例字段）、可以提供部分实现、可以定义构造逻辑**，而接口只能定义契约。当多个类不仅有共同的行为，还有共同的数据结构时，抽象类往往是更自然的选择。

# 6. 接口

## 6.1 接口的定义与实现

**接口（interface）**是一种**完全抽象的行为规范**：它只描述"能做什么"，不关心"怎么做的"，也不持有任何状态。定义用 `interface` 关键字，实现用 `implements`：

```java
public interface 接口名 { }

public class 实现类名 implements 接口名 { }
```

```java
interface Swimmable {
    void swim();                 // 等价于 public abstract void swim();
}

interface Flyable {
    void fly();
}

// 一个类可以同时实现多个接口 —— 这就是"接口多继承"
class Duck implements Swimmable, Flyable {
    @Override public void swim() { System.out.println("鸭子游泳"); }
    @Override public void fly()  { System.out.println("鸭子飞"); }
}
```

接口和类一样**不能被实例化**，只能被实现。它的子类叫**实现类**，实现类要么重写接口中**全部**抽象方法，要么自己也声明为抽象类。此外，接口还支持"**继承一个类的同时实现多个接口**"：

```java
class RubberDuck extends Bird implements Swimmable {
    @Override public void swim() { System.out.println("橡皮鸭漂着"); }
}
```

这里用"**实现**（`implements`）"而不是"继承（`extends`）"，用词是刻意的：**继承意味着"得到父类的东西"，实现意味着"承诺遵守接口的约定"**。接口不会给你任何现成的实现，它只是列出一份"必须提供哪些方法"的清单，具体怎么写完全由实现类决定。同样地，实现类对象也是"能赋值给接口类型"的（因为约定被遵守了），这正是接口多态的基础。

## 6.2 接口中成员的特点

接口的成员规则和普通类差别很大，值得逐条对比：

| 成员 | 默认修饰符 | 说明 |
|------|-----------|------|
| 成员变量 | `public static final` | **只能是常量**，必须显式初始化；不能有实例变量 |
| 构造方法 | —— | **没有构造方法**，因为接口不需要被实例化，也没有实例字段要初始化 |
| 成员方法（JDK7 及以前）| `public abstract` | 只能是抽象方法 |
| 默认方法（JDK8+）| `public default` | 有方法体，实现类可以不重写 |
| 静态方法（JDK8+）| `public static` | 有方法体，**只能通过接口名调用** |
| 私有方法（JDK9+）| `private` / `private static` | 有方法体，只能在接口内部被其他方法调用 |

```java
public interface MyInterface {
    // 以下三种写法编译后完全等价，编译器一律补成 public static final
    int A = 1;
    public int B = 2;
    public static final int C = 3;
    // int D;                        // ✗ 编译错误：接口中的变量必须初始化

    void show();                     // 等价于 public abstract void show();

    default void hello() {           // JDK8 默认方法
        System.out.println("hello");
    }

    static void util() {             // JDK8 静态方法
        System.out.println("util");
    }
}
```

接口没有构造方法这一点很重要——它从语法上就杜绝了"接口被实例化"的可能，也意味着**接口不能持有实例状态**。这正是接口与抽象类最本质的分界：**接口只能描述行为，抽象类还能描述状态**。

再往深一层看，接口不能有实例字段和构造方法，等于把"**接口只能描述行为、不能描述状态**"这条设计原则落到了语法层面——它不是"Java 没实现"，而是"故意不让做"。想清楚这一点，抽象类与接口的分工也就呼之欲出了：**要保存状态、要复用实现细节时用抽象类；只描述"能做什么"时用接口**（取舍见 6.7 节）。

> **注意**：接口中的变量默认是 `public static final`，所以 `int A = 1;` 这种写法**看起来像实例变量，实际是常量**。这也是为什么接口里不能定义 `private int count;`——接口没有实例，这些字段无处安放。多实现时需要注意：如果两个接口定义了同名常量而类型不同，同时使用会造成歧义，编译器会报错。

## 6.3 类与类、类与接口、接口与接口

三种关系放在一起对比，区别一目了然：

| 关系 | 关键字 | 数量限制 |
|------|--------|---------|
| 类与类 | `extends` | **单继承**，但可以多层继承 |
| 类与接口 | `implements` | 可以单实现、**多实现**；还可以"继承一个类的同时实现多个接口" |
| 接口与接口 | `extends` | 可以单继承，也可以**多继承** |

```java
interface A { void a(); }
interface B { void b(); }

interface C extends A, B { }       // 接口与接口：多继承，C 同时具备 a 和 b 的约束

class Impl implements C {
    @Override public void a() { }
    @Override public void b() { }
    // 注意：C 从 A、B 各继承了一个方法，但两个接口若定义了同名同参的方法，只需重写一次
}
```

接口之所以敢于允许多继承，根本原因是**接口里没有状态**——继承到的只是一份"方法签名清单"：签名相同就合并成一条，签名不同就各自保留，天然不存在 C++ 菱形继承中"同一个字段存几份、同一个方法体听谁的"这类冲突。JDK8 引入 `default` 方法后，这个前提被打破了一半，Java 的补救办法就是 6.4 节那条规则：**默认方法冲突时，必须由实现类显式重写**。

如果多个接口中出现**同名同参的抽象方法**，实现类**只需重写一次**即可（因为它们的签名完全相同，一个实现就同时满足两个约束）。但如果是**同名同参的默认方法**，情况就不同了（见 6.4 节）。

## 6.4 默认方法 default（JDK8）

默认方法是为了解决**接口升级的兼容问题**而引入的。设想一个已经被上百个类实现的接口，现在要给它加一个新方法：如果加成抽象方法，**所有实现类会立刻编译失败**——这是一种毁灭性的破坏性变更。

`default` 方法给新加的方法提供一份默认实现，老实现类一行都不用改就继续可用：

```java
interface PayService {
    void pay(double amount);

    // 新增需求：所有支付方式都要打日志。
    // 如果写成 void log(String msg);  所有实现类立刻编译报错
    // 写成 default 则老实现类无感升级
    default void log(String msg) {
        System.out.println("[支付日志] " + msg);
    }
}
```

三条使用规则：

1. **默认方法不是抽象方法**，不强制重写；一旦重写，**重写时要去掉 `default` 关键字**；
2. `public` 可以省略，但 **`default` 不能省略**（省了就变成抽象方法，必须分号结尾）；
3. **多个接口存在同名同参的默认方法时，实现类必须重写**，否则编译器无法决定用哪一个。

```java
interface A2 { default void show() { System.out.println("A2 的默认实现"); } }
interface B2 { default void show() { System.out.println("B2 的默认实现"); } }

class C2 implements A2, B2 {
    @Override
    public void show() {              // 必须重写，否则编译报错
        A2.super.show();              // 想沿用某个接口的实现，用 接口名.super.方法() 显式指定
        B2.super.show();
        System.out.println("C2 自己的补充逻辑");
    }
}

// new C2().show(); 输出：
// A2 的默认实现
// B2 的默认实现
// C2 自己的补充逻辑
```

这里还有一个**优先级规则**：如果一个类继承的父类中有一个具体方法，同时它实现的接口里有一个同名同参的默认方法，那么**父类的方法优先**——类的方法永远胜过接口的默认方法。这条规则保证了"给类库新增默认方法不会悄悄改变已有子类的行为"。

## 6.5 静态方法 static（JDK8）

JDK8 起接口也可以定义静态方法，但有一条硬规则：**只能通过接口名调用，不能通过实现类名或对象名调用**。

```java
interface StringComparator {
    static int compareByLength(String a, String b) {
        return a.length() - b.length();
    }
}

class MyComparator implements StringComparator { }

public class StaticMethodDemo {
    public static void main(String[] args) {
        System.out.println(StringComparator.compareByLength("abc", "a"));   // 2

        // MyComparator.compareByLength("abc", "a");   // ✗ 编译错误：找不到符号
        // new MyComparator().compareByLength(...);    // ✗ 编译错误：找不到符号
    }
}
```

之所以这样设计，是因为**接口的静态方法不参与继承**——它不属于实现类。可以设想如果允许继承：某个类同时实现 `A`、`B` 两个接口，两者都定义了 `static void util()`，那么 `MyClass.util()` 到底该调到哪个？禁止继承直接消灭了这个二义性。

> **设计考量**：接口的静态方法适合放"**和这个接口契约强相关的工具方法**"。例如 `Comparator` 接口在 JDK8 之后新增了 `Comparator.comparing(...)`、`Comparator.naturalOrder()` 等静态工厂方法。放在接口里的好处是"名字直接说明归属"——`Comparator.comparing` 一眼就知道是给比较器用的，比单独建一个 `ComparatorUtils` 类更紧凑。

## 6.6 私有方法 private（JDK9）

JDK9 之前，接口里如果有多个 `default` 方法需要共用一段代码，只能把这段代码抽成一个 `public default` 方法——但那样它就成了接口对外 API 的一部分，实现类可以调用甚至重写，**污染了接口契约**。JDK9 引入私有方法解决了这个问题：

```java
public interface Logger {
    default void info(String msg) { print("INFO", msg); }
    default void error(String msg) { print("ERROR", msg); }

    // 两个 default 方法共用的逻辑，抽成 private —— 对外不可见
    private void print(String level, String msg) {
        System.out.println("[" + level + "] " + msg);
    }
}
```

两种私有方法各司其职：**普通私有方法**给 `default` 方法用，**静态私有方法**给接口的 `static` 方法用。它们都**不能是抽象的**，也**不能被实现类调用或重写**——它们只是接口内部的实现细节。

## 6.7 抽象类与接口的对比与选择

这是面向对象设计里最经典的一道选择题。两者都能声明抽象方法、都不能被实例化、都能实现多态，差别主要在这几处：

| 维度 | 抽象类 | 接口 |
|------|--------|------|
| 声明关键字 | `abstract class` | `interface` |
| 与子类的关系 | `extends`，**单继承** | `implements`，**可多实现** |
| 成员变量 | 任意类型、任意权限，可以是实例变量 | 只能 `public static final` 常量 |
| 构造方法 | 有，供子类 `super` 调用 | **没有** |
| 方法类型 | 抽象方法 + 普通方法 | 抽象方法 + `default` / `static` / `private` 方法 |
| 能否持有状态 | **能**（实例字段） | 不能 |
| 设计定位 | 表达 **is-a**："是一类东西" | 表达 **can-do**："具备某种能力" |
| 复用能力 | 能复用状态与部分实现 | 只能复用行为契约（默认方法） |
| 演进成本 | 加普通方法不影响子类 | JDK8 前加抽象方法会破坏所有实现类；JDK8 后可用 `default` 缓解 |
| 典型用例 | `InputStream`、`AbstractList`、`HttpServlet` | `Comparable`、`Runnable`、`List`、`Map` |

**选择建议**可以按下面的顺序判断：

1. **需要复用状态或构造逻辑**（有实例字段、有初始化流程）→ 选**抽象类**；
2. **只是想定义一组行为契约**，而且这个类可能还需要继承别的父类 → 选**接口**；
3. **两者都要** → 让抽象类实现接口，具体类再继承这个抽象类，形成"**接口定契约 + 抽象类给骨架**"的结构；
4. **拿不准时优先接口**——接口没有单继承限制，将来扩展的余地更大。

再落到几个常见的具体场景上，选型就更明确了：

| 场景 | 选什么 | 理由 |
|------|-------|------|
| 一组类共享字段和初始化逻辑（各种图形都有 `name`、都要算面积）| 抽象类 | 需要复用状态 |
| 给类库定义"可以被外部实现"的扩展点 | 接口 | 实现方可能已经继承了别的类，用接口才不被单继承挡住 |
| 让多个互不相关的类具备同一种能力（可比较、可序列化、可关闭）| 接口 | "能力"天然是多继承的 |
| 一个功能既有固定流程又有可变步骤 | 抽象类（模板方法）| 需要把流程写死在父类里，只留变化点 |
| 既想定契约又想让实现方少写样板代码 | 接口 + 抽象骨架类 | 两者兼得 |

判断时最容易犯的错是"**因为想复用代码而选了抽象类**"，结果把一个本来很独立的概念强行塞进继承链。继承表达的是"**是一种**"，如果两个类之间说不出"B 是一种 A"这句话，那就该用接口或者组合，而不是继承。

> **行业实践**：现代 Java 类库的通行结构是"**接口定义契约 + 抽象类提供骨架实现**"。集合框架就是最好的范例：`List` 是接口，`AbstractList` 是骨架实现；`Collection` 是接口，`AbstractCollection` 是骨架。调用方声明类型时用接口（多态、易 mock、易替换），实现方继承抽象类以省去大量样板代码。自己写工具库时照抄这个结构，几乎不会错。

## 6.8 接口的应用：规则抽象与接口多态

接口的本质是**对行为的抽象，是一套规则**。它把"调用方需要的操作"和"实现方的具体做法"彻底分开，从而获得和多态一样的好处：**调用方只依赖接口，不依赖实现**。

```java
interface Usb {
    void start();
    void stop();
}

class Mouse implements Usb {
    @Override public void start() { System.out.println("鼠标启动"); }
    @Override public void stop()  { System.out.println("鼠标停止"); }
}

class Keyboard implements Usb {
    @Override public void start() { System.out.println("键盘启动"); }
    @Override public void stop()  { System.out.println("键盘停止"); }
}

public class UsbTest {
    // 形参是接口类型，可以接收该接口所有实现类的对象 —— 这就是接口多态
    public static void useDevice(Usb usb) {
        usb.start();
        usb.stop();
    }

    public static void main(String[] args) {
        useDevice(new Mouse());       // 鼠标启动 / 鼠标停止
        useDevice(new Keyboard());    // 键盘启动 / 键盘停止
    }
}
```

`useDevice` 不需要知道传入的到底是鼠标还是键盘，也不需要导入这两个类。将来新增一个 `Camera implements Usb`，这个方法一行都不用改就自动支持——这正是接口多态带来的可扩展性。

## 6.9 适配器设计模式

当接口里的抽象方法太多，而某个实现类**只用得上其中一两个**时，被迫实现全部方法就非常痛苦。**适配器模式**用一个中间类解决这个问题：中间类实现接口并给出**全部空实现**，使用者只需要继承这个中间类、重写自己关心的方法。

```java
interface Listener {
    void onClick();
    void onLongClick();
    void onDoubleClick();
    void onScroll();
    void onTouch();
}

// 适配器：实现接口的全部方法，但方法体一律留空
// 用 abstract 修饰，防止别人直接 new 一个"什么都没做"的适配器
abstract class ListenerAdapter implements Listener {
    @Override public void onClick()     { }
    @Override public void onLongClick() { }
    @Override public void onDoubleClick() { }
    @Override public void onScroll()    { }
    @Override public void onTouch()     { }
}

// 使用者只关心点击事件，其余四个方法一个字都不用写
class MyListener extends ListenerAdapter {
    @Override
    public void onClick() {
        System.out.println("只处理点击事件");
    }
}
```

两个设计要点值得展开：

**第一，为什么适配器类要用 `abstract` 修饰？** 因为它的所有方法都是空实现，`new ListenertAdapter()` 出来的对象**什么也不会做**，在业务上毫无意义。用 `abstract` 把它标记成"半成品"，强制使用者必须继承并重写——这样"只用其中一部分方法"的语义才能落到实处。

**第二，它为什么叫"适配器"？** 在整个结构中，`ListenerAdapter` 扮演的是"接口与使用者之间的转换层"：它把"必须实现 5 个方法"这个硬约束，适配成"只重写你关心的那几个"。这和电源适配器把一种接口转换成另一种接口的思路是一致的。

顺带澄清一个命名上的混淆：设计模式里的"适配器模式"（Adapter）本意是**把一个类的接口转换成客户期望的另一种接口**，让原本不兼容的两个类能协同工作（就像电源转接头）。本节讲的"接口适配器"是它在 Java 接口场景下的一个特例用法——用一份空实现来**抹平接口中不需要的方法**，目的是让实现者少写代码。

> **行业实践**：JDK8 有了 `default` 方法之后，**新设计的接口可以直接给方法加默认空实现**，从源头消灭"必须实现全部方法"的问题，适配器模式在新接口上已经不常用了。但它仍然是理解 JDK 自带 API 的钥匙——`java.awt.event.MouseAdapter`、`KeyAdapter`、`WindowAdapter` 全是这个套路；同时它也是"接口定契约 + 类给骨架实现"这一设计思路最朴素的形态。

# 7. 内部类

## 7.1 内部类的概念与访问特点

**内部类**是定义在另一个类内部的类。它存在的意义有两点：**组织上**把只服务于外部类的辅助类型藏进外部类，避免污染包空间；**逻辑上**让类定义紧贴使用位置，读起来更连贯。JDK 自己就大量使用内部类——`HashMap.Node`、`ArrayList.Itr`、`Thread.State` 都是。

内部类的访问特点有两条，它们是相反方向的：

- **内部类可以直接访问外部类的成员，包括私有的**；
- **外部类要访问内部类的成员，必须先创建内部类对象**，再用对象调用。

第一条之所以成立，是因为内部类编译后会隐式持有一个指向外部类对象的引用；第二条则是因为内部类的成员属于内部类的实例，外部类没有任何捷径可以绕过"创建对象"这一步。

不过也要清醒地看待内部类的代价：**每个内部类都会生成独立的 `.class` 文件**（成员内部类是 `Outer$Inner.class`，匿名内部类是 `Outer$1.class`），成员内部类还会额外持有一个外部类引用，增加内存开销。所以它只适合"确实只服务于外部类"的场景，不要为了让代码"看起来紧凑"而把本来独立的类硬塞进去——**判断标准是"这个类离开外部类还有没有意义"**。

## 7.2 成员内部类

写在**成员位置**（和成员变量、成员方法同级）的内部类叫成员内部类。它可以被 `private`、缺省、`protected`、`public`、`static`、`final`、`abstract` 等修饰符修饰，完全和普通成员一样。还有一条版本相关的规则：**JDK16 之前成员内部类不能定义静态成员，JDK16 起才允许**。

创建成员内部类对象有两种方式——一种是**在外部类里提供方法**对外暴露（推荐，能配合 `private` 使用），一种是**用 `new` 直接构造**：

```java
public class Outer {
    private int outerNum = 10;
    public static String tag = "OUTER";

    public class Inner {                     // 成员内部类
        private int innerNum = 20;
        private int outerNum = 30;           // 故意与外部类同名

        public void show() {
            System.out.println(innerNum);            // 20：内部类自己的
            System.out.println(outerNum);            // 30：就近原则，内部类的
            System.out.println(Outer.this.outerNum); // 10：外部类的那一个
            System.out.println(tag);                 // OUTER：可以直接访问外部类静态成员
        }
    }

    public Inner getInner() {                // 方式一：对外提供内部类对象
        return new Inner();
    }
}
```

```java
// 方式一：通过外部类的实例方法拿到内部类对象
Outer.Inner in1 = new Outer().getInner();

// 方式二之一：分两步
Outer outer = new Outer();
Outer.Inner in2 = outer.new Inner();

// 方式二之二：一步到位
Outer.Inner in3 = new Outer().new Inner();

in3.show();   // 20 / 30 / 10 / OUTER
```

注意方式二的语法：`外部类对象.new 内部类()`。这里的 `new` 是**由"外部类对象"来调用的**，因为成员内部类对象必须依附于一个外部类对象存在。

当内部类和外部类出现**同名成员**时，就用 `外部类名.this.成员名` 来访问外部类的那一份。`Outer.this` 是一个**隐含的、指向外部类对象的指针**——每个内部类对象被创建时，编译器都会给它加一个这样的引用。它在内存里的样子是这样的：

```text
 ┌────────────────────────┐          ┌────────────────────────────┐
 │  Outer 对象             │◀─────────│  Outer$Inner 对象           │
 │  outerNum = 10          │  隐含引用 │  innerNum = 20             │
 │  tag = "OUTER"（静态）  │  this$0  │  outerNum = 30             │
 │                         │          │  this$0 → 指向外部类对象    │
 └────────────────────────┘          └────────────────────────────┘
```

正是这个 `this$0` 让内部类能畅通无阻地访问外部类的私有成员。但反过来，它也带来一个隐患：**只要内部类对象还活着，外部类对象就无法被 GC 回收**——这是 Android 里 `Handler` 造成内存泄漏的经典原因，也是选型时优先考虑静态内部类的主要理由。

> **陷阱**：这个 `this$0` 造成过一次至今仍被反复引用的经典事故——Android 里在 `Activity` 中用匿名内部类创建 `Handler` 并发送延迟消息：消息队列持有 `Handler`，`Handler` 的 `this$0` 又持有 `Activity`，于是 `Activity` 明明已经销毁，却因为这条引用链始终无法被 GC 回收，造成内存泄漏。修法正是把 `Handler` 改成**静态内部类**，再用 `WeakReference` 持有 `Activity`。这条经验在 Android 之外同样成立：**只要一个长生命周期的对象持有了内部类实例，就等于间接持有了整个外部类对象**。

## 7.3 静态内部类

用 `static` 修饰的内部类叫**静态内部类**，它属于**外部类本身**而不是外部类的某个对象，因此**不持有外部类对象的引用**。这一条差别带来了三个直接结果：

1. 静态内部类**只能直接访问外部类的静态成员**；想访问实例成员，必须先自己创建外部类对象；
2. 创建它**不需要先有外部类对象**；
3. 它可以有自己的静态方法和静态变量（没有 JDK16 的限制）。

```java
public class Outer2 {
    private int num = 10;                 // 实例成员
    private static int staticNum = 20;    // 静态成员

    static class Inner {
        public void show() {
            System.out.println(staticNum);            // ✓ 可以直接访问外部类静态成员
            // System.out.println(num);               // ✗ 编译错误：访问不到外部类的实例成员
            System.out.println(new Outer2().num);     // ✓ 自己造一个外部类对象就行了
        }

        public static void staticShow() {             // 静态内部类可以有静态方法
            System.out.println("静态内部类的静态方法");
        }
    }
}
```

```java
// 创建对象：不需要先 new Outer2()
Outer2.Inner inner = new Outer2.Inner();
inner.show();                 // 20 / 10

// 调用静态方法：直接用类名
Outer2.Inner.staticShow();    // 静态内部类的静态方法
```

对比一下创建语法就能看出本质区别：成员内部类是 `new Outer().new Inner()`，静态内部类是 `new Outer.Inner()`——**前者要依托对象，后者只需要类**。

静态内部类还有一个非常经典的应用——**静态内部类实现的单例模式**，它同时做到了"线程安全"和"懒加载"：

```java
public class Singleton {
    private Singleton() { }                     // 私有构造：外部无法 new

    private static class Holder {                // 静态内部类：不持有外部类引用
        private static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return Holder.INSTANCE;                  // 首次调用时才触发 Holder 的类加载
    }
}
```

这段代码的巧妙之处在于：`Holder` 只有在第一次被访问时才会被加载（对应 1.4 节"访问静态成员触发类初始化"），而 JVM 保证类的初始化是**线程安全且只执行一次**的。于是既不需要加锁，也不存在饿汉式单例"程序一启动就创建对象"的浪费。

> **行业实践**：**能用静态内部类就用静态内部类**。因为它不持有外部类引用，既没有内存泄漏风险，也更省内存。Java 里两个最典型的用途是：**静态内部类实现的单例模式**（利用"类加载时初始化且只初始化一次"的线程安全特性），以及 **Builder 模式**（如 `Map.Entry`、`Person.Builder`）。JDK 甚至专门提供 `java.util.Map.Entry` 这样的静态内部接口。

## 7.4 局部内部类

定义在**方法或代码块内部**的内部类叫局部内部类，它的地位类似于方法里的局部变量。三个特点：**作用域仅限方法内部**，外界无法直接使用，必须在方法内部创建对象使用；它**可以直接访问外部类的成员**，也可以访问**方法内的局部变量**（该局部变量必须是 `final` 或等效 final）；它**不能有静态成员**。

```java
public class Outer3 {
    private int num = 10;

    public void method() {
        int localNum = 100;              // 等效 final：赋值后再也没改过
        final String msg = "hi";

        class Local {                    // 局部内部类
            void show() {
                System.out.println(num);        // 10：访问外部类成员
                System.out.println(localNum);   // 100：访问方法内的局部变量
                System.out.println(msg);        // hi
            }
        }

        new Local().show();              // 只能在方法内部创建并使用
    }
}
```

局部内部类在实际工程中很少见——绝大多数场景下，**匿名内部类或 Lambda 都能更简洁地完成同样的工作**。它的主要价值是理解"捕获局部变量"这一机制，而这正是下一节的内容。

还有两个语法细节：局部内部类**不能被 `public`、`private`、`protected`、`static` 修饰**（它连"类的成员"都算不上，只是方法里的一个局部声明），但可以被 `final` 和 `abstract` 修饰；同理，它里面**不能定义静态成员**——因为局部内部类的初始化时机取决于方法的执行，JVM 无法为它的静态成员确定一个"类加载"时点。

## 7.5 匿名内部类

**匿名内部类**是没有名字的内部类，它把"定义类"和"创建对象"合成了一步：

```java
new 类名或接口名() {
    // 重写的方法
};                                    // 注意末尾的分号不能丢
```

理解匿名内部类最准确的一句话是：**它本质上是一个"实现了某个接口"或"继承了某个类"的、没有名字的子类对象**。

```java
public class AnonDemo {
    public static void main(String[] args) {
        // new 接口名() { 重写方法 } —— 花括号里的就是"子类"的类体
        Runnable r = new Runnable() {
            @Override
            public void run() {
                System.out.println("匿名内部类实现的任务");
            }
        };
        new Thread(r).start();

        // 需要捕获局部变量时，该变量必须"等效 final"（只赋值一次）
        int base = 10;
        Runnable r2 = new Runnable() {
            @Override
            public void run() {
                System.out.println("base = " + base);
            }
        };
        r2.run();                      // base = 10

        // base = 20;                  // ✗ 一旦解开注释，上面的匿名内部类立刻编译报错
    }
}
```

它有四个必须记住的特点：

| 特点 | 说明 |
|------|------|
| 本质 | 一个实现接口或继承类的**匿名子类对象**，同时完成了"定义类"和"创建对象" |
| 只能使用一次 | 没有类名，就没办法再 `new` 第二次；想复用只能把对象存进变量 |
| 编译产物 | 每个匿名内部类生成一个 `<外部类>$1.class`、`<外部类>$2.class`，按出现顺序编号 |
| 捕获局部变量的限制 | 局部变量必须是 `final` 或**等效 final** |

"只能使用一次"这一点常被误解成"这个对象只能被调用一次"，其实它的准确含义是——**这种写法只能创建一个对象**。因为类没有名字，你没办法像普通类那样 `new` 出第二个来；想再要一个同类对象，只能把那段 `new 接口() { ... }` 的整个代码块再抄一遍。但**创建好的对象本身可以被反复使用**：

```java
Runnable r = new Runnable() {
    @Override
    public void run() { System.out.println("run"); }
};

r.run();      // ✓ 同一个对象可以被多次调用
r.run();
```

**为什么捕获的局部变量必须等效 final？** 这是最需要讲清楚的一点。局部变量存放在**栈**上，方法执行结束就随之销毁；而匿名内部类对象存放在**堆**上，生命周期可能比方法长得多。为了让内部类对象在方法结束后还能读到那个变量，`javac` 采取的办法是**值复制**——把局部变量的值复制一份，作为匿名内部类的成员变量保存下来。

既然保存的是"复制出来的那一刻的值"，如果原变量之后还能被修改，就会出现"外部变量已经变了、内部副本还是旧值"的诡异局面——程序员会以为内部类看到的是最新值，实际不是。Java 的选择是干脆**禁止这种修改**：只要你在匿名内部类里用到了某个局部变量，这个变量就必须"一次赋值、永不修改"。JDK8 之前必须显式写 `final`，JDK8 之后放宽为**等效 final**（effectively final），也就是"虽然没写 `final`，但事实上从没被重新赋值"。

> **陷阱**：匿名内部类最常被忽略的两点，一是**末尾的分号**（`}` 后面漏一个 `;` 是最常见的语法错误），二是**在匿名内部类里访问外部实例方法时 `this` 的含义**——`this` 指的是**匿名内部类自己的对象**，不是外部类对象。要访问外部类的成员，必须写 `外部类名.this.成员`。

## 7.6 四种内部类对比

把四种内部类放在一张表里，选型时就不容易搞混：

| 维度 | 成员内部类 | 静态内部类 | 局部内部类 | 匿名内部类 |
|------|-----------|-----------|-----------|-----------|
| 定义位置 | 类的成员位置 | 类的成员位置 + `static` | 方法或代码块内部 | 表达式位置（`new` 处）|
| 能否有静态成员 | JDK16 前不行，JDK16 起可以 | 可以 | 不可以 | 不可以 |
| 是否持有外部类对象引用 | **持有**（`Outer.this`）| 不持有 | 持有（在实例方法中）| 持有（在实例方法中）|
| 直接访问外部类实例成员 | 可以 | 不可以（需先 `new` 外部类）| 可以 | 可以 |
| 创建方式 | `new Outer().new Inner()` | `new Outer.Inner()` | 只能在方法内 `new` | 定义即创建 |
| 编译后文件名 | `Outer$Inner.class` | `Outer$Inner.class` | `Outer$1Local.class` | `Outer$1.class` |
| 典型用途 | 只服务于外部类的成员 | 单例、Builder、`Map.Entry` | 方法内的临时辅助类 | 回调、事件监听、线程任务 |

选择建议按下面的顺序考虑：

1. **优先静态内部类**——不隐式持有外部类引用，没有内存泄漏风险，还省一次对象引用的开销；
2. **需要读写外部类的实例成员**时，用成员内部类；
3. **只需要用一次的回调**，用匿名内部类；如果那个接口是函数式接口（只有一个抽象方法），**优先写成 Lambda**；
4. **局部内部类几乎不用**，它的场景基本都被匿名内部类和 Lambda 覆盖了。

## 7.7 从匿名内部类到 Lambda

JDK8 引入 Lambda 之前，**函数式接口**（只有一个抽象方法的接口，如 `Runnable`、`Comparator`、`Callable`）的唯一实现方式就是匿名内部类。Lambda 出现之后，这类代码可以压缩到一行。

```java
// 匿名内部类写法
Runnable r1 = new Runnable() {
    @Override
    public void run() {
        System.out.println("run");
    }
};

// Lambda 写法：省掉"接口名 + 方法名 + 参数类型 + 大括号"，只留参数和方法体
Runnable r2 = () -> System.out.println("run");

// 带参数的例子
Comparator<String> c1 = new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
        return a.length() - b.length();
    }
};
Comparator<String> c2 = (a, b) -> a.length() - b.length();
```

Lambda 之所以能这么写，是因为编译器可以根据**左侧的目标类型**（`Runnable`、`Comparator<String>`）推断出"要实现的是哪个接口、哪个方法"。这条机制也决定了 Lambda 的一个限制：**它不能脱离目标类型单独存在**——`var x = () -> {};` 是编译不过的，因为编译器不知道要往哪个接口里套。

也正因为如此，Lambda 虽然写法上更像"表达式"，但它背后的类型系统依然是**接口**：`Runnable r = () -> {};` 里的 `r` 类型仍然是 `Runnable`，你依然可以对它调用 `r.run()`。理解这一点，就不会把 Lambda 误解成某种与接口无关的"函数类型"语法糖了。它也解释了另一个常见疑问——**为什么变量能捕获的范围和匿名内部类一样**：因为两者编译后做的事本质相同，都是把捕获的值复制成内部字段。

> **注意**：Lambda 只能用于**函数式接口**。如果一个接口有多个抽象方法（哪怕加了 `default` 实现了其他方法也无所谓，关键是抽象方法只有一个），就仍然只能用匿名内部类。Lambda 的完整语法（参数省略规则、方法引用 `::`、变量捕获范围）见第 12 篇。

# 8. 小结

| 主题 | 一句话要点 |
|------|-----------|
| `static` | 属于类而不属于对象；静态方法**没有 `this`**，所以只能访问静态成员；静态成员随类加载初始化，早于任何对象；`main` 必须是 `static`，因为 JVM 调用它时还没有对象 |
| 继承 | 单继承 + 多层继承，所有类最终继承 `Object`；构造方法与 `private` 成员不能被继承（但私有字段确实在对象内存里）；子类构造第一行隐含 `super()` |
| 重写 | 口诀"**两同两小一大**"；只有虚方法（非 `final` / `private` / `static`）才能被重写；`@Override` 是编译期校验，不写也能重写但极容易写错；重写靠**虚方法表**生效 |
| 重写 vs 重载 | 重写发生在父子类之间、运行期绑定、与多态有关；重载发生在同一个类中、编译期绑定、与多态无关 |
| 构造顺序 | 父类静态代码块 → 子类静态代码块 → 父类构造代码块/字段初始化 → 父类构造方法 → 子类构造代码块/字段初始化 → 子类构造方法；静态的只跑一次 |
| 多态 | 前提是"有继承/实现 + 父类引用指向子类对象 + 有方法重写"；**编译看左边、运行看右边**；变量、静态方法都看左边，只有实例方法看右边 |
| 转型 | 向上转型自动、向下转型要强转；强转前先 `instanceof` 判断；JDK14 可用模式匹配 `if (a instanceof Cat c)`；有继承关系时编译器拦不住，越界就是运行期 `ClassCastException` |
| 权限修饰符 | `private` ⊂ 缺省 ⊂ `protected` ⊂ `public`，按"本类 / 同包 / 不同包子类 / 不同包无关类"四档记忆；**类的权限只有 `public` 和缺省** |
| `final` | 修饰类不能被继承、修饰方法不能被重写、修饰变量只能赋一次值；修饰**引用类型时锁的是地址，对象内容仍然可变** |
| 代码块 | 局部代码块（缩小作用域）、构造代码块（每次 `new` 都跑、在构造方法体前）、静态代码块（类加载时跑一次，早于 `main`）|
| 抽象类 | 有抽象方法就必须是抽象类；不能实例化但**可以有构造方法**；子类要么重写全部抽象方法、要么自己也是抽象类；`abstract` 不能与 `final`/`private`/`static` 共存 |
| 接口 | 变量默认 `public static final`；**没有构造方法**；JDK7 前方法只能是 `public abstract`，JDK8 加 `default` 与 `static`，JDK9 加 `private`；接口可以多继承接口，类可以多实现接口 |
| 抽象类 vs 接口 | 抽象类表达 **is-a**、能持有状态、单继承；接口表达 **can-do**、不能持有状态、可多实现；实务上"接口定契约 + 抽象类给骨架" |
| 内部类 | 成员内部类持有外部引用（`Outer.this`）、静态内部类不持有（**优先选它**）、局部内部类作用域限于方法、匿名内部类本质是"一次性的子类对象"且捕获的局部变量必须等效 final |

下一篇（第 11 篇）进入 Java 的常用 API 世界，看 `Math`、`System`、`Runtime`、`Object`、`Objects`、`BigInteger`、`BigDecimal` 这些开箱即用的工具怎么用、什么时候用；Lambda 与方法引用见第 12 篇，`instanceof` 模式匹配等新语法集中在第 29 篇。
