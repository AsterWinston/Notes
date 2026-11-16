枚举（`enum`）是 Java 5 引入的**有限离散常量集合的类型**，语法上是一颗语法糖，编译后却变成一个货真价实的 `final class`。它解决的是"一组取值固定、但用 `int` 或 `String` 表达又不安全"的问题——订单状态、会员等级、运算符号、错误码，凡是"只能取这几个值"的东西，都该是枚举。本篇先看它的字节码本质，再讲构造器与常量专有行为，最后落到 `EnumSet`/`EnumMap`、枚举单例和"用枚举替代 switch"这三个真正改变代码结构的用法。

# 1. 枚举的本质

## 1.1 从语法糖到字节码

写了一行 `public enum Color { RED, GREEN, BLUE; }`，编译器生成的其实是一个完整的类：

```java
// 编译器生成的等价形态（伪代码）
public final class Color extends java.lang.Enum<Color> {
    public static final Color RED   = new Color("RED",   0);
    public static final Color GREEN = new Color("GREEN", 1);
    public static final Color BLUE  = new Color("BLUE",  2);

    private static final Color[] $VALUES = { RED, GREEN, BLUE };   // 合成字段，values() 的数据源

    private Color(String name, int ordinal) { super(name, ordinal); }   // 隐式 private

    public static Color[] values() { return $VALUES.clone(); }          // 合成方法
    public static Color valueOf(String name) { /* 按名字查表 */ }        // 合成方法
}
```

用 `javap` 反编译就能看到（`-p` 才会显示 `private` 成员，`-c` 会显示字节码）：

```bash
javac Color.java
javap -p Color.class      # -p 显示全部成员，包括 private 的
```

```text
Compiled from "Color.java"
public final class Color extends java.lang.Enum<Color> {
  public static final Color RED;
  public static final Color GREEN;
  public static final Color BLUE;
  private static final Color[] $VALUES;
  public static Color[] values();
  public static Color valueOf(java.lang.String);
  private Color(java.lang.String, int);
  static {};
}
```

由此可以读出四件确定的事：

| 结论 | 依据 |
|------|------|
| 枚举**不能继承别的类** | 已经隐式 `extends java.lang.Enum`，Java 是单继承 |
| 枚举类是 `final` 的 | 所以不能被继承（带类体的常量是它的匿名子类，是编译器的特权） |
| 每个常量都是 **单例** | `public static final` 字段，类型就是枚举类自己 |
| 常量在**类初始化时创建** | 创建逻辑都在 `static {}` 静态初始化块里，JVM 保证只执行一次 |

第三个事实是枚举所有高级特性的根基——**每个枚举常量在整个 JVM 里只有一个实例**，所以 `Color.RED == Color.valueOf("RED")` 永远是 `true`，用 `==` 比较枚举比 `equals()` 更快也更安全。

## 1.2 枚举 vs 静态常量

在没有枚举的年代，一组固定取值是用 `static final int`（或 `String`）表达的。两者对比：

| 维度 | `static final int` 常量 | 枚举 |
|------|------------------------|------|
| 类型安全 | 弱：`void setStatus(int s)` 可以传任意 `int`，甚至 `-999` | 强：参数类型就是 `OrderStatus`，传别的类型编译不过 |
| 取值约束 | 无法限制，全靠调用方自觉 | 只能取已定义的常量 |
| 可遍历 | 需要自己额外维护一个数组 | `values()` 直接给出全量 |
| 能否带属性 | 要另建 `Map` 或平行数组，数据与常量分离 | 常量自带字段，一个常量就是一个数据对象 |
| 能否带行为 | 不能 | 能（常量专有行为，见第 3 节） |
| `switch` 支持 | 支持 `int`（`String` 从 Java 7 起） | 原生支持，`case` 里不用写类名前缀 |
| 命名空间 | 常量平铺在类里，命名靠人工加前缀 | 天然挂在类型下，`OrderStatus.PAID` 自带限定 |
| 打印可读性 | `System.out.println(status)` 输出 `20` | 输出 `PAID`，默认就是常量名 |
| 单例特性 | 无 | 天然单例，可实现最安全的单例模式 |
| 序列化安全 | 无保障 | JVM 特殊处理，反序列化不会产生新实例 |

> **最佳实践**：判断标准很简单——**如果某个字段的取值只能是几个固定的值，那它就应该是一个枚举**。状态、类型、级别、渠道、币种、错误码全都属于这一类。用 `int` 或 `String` 表达它们，等于把这层约束从类型系统里删掉，把编译期错误推迟成运行期错误。

# 2. 枚举的定义与使用

## 2.1 最简形式

```java
public enum Color {
    RED, GREEN, BLUE;   // 默认调用无参构造器
}
```

常量之间用逗号分隔。最后一个常量后面的分号**可写可不写**，但如果后面还要声明字段或方法，就**必须写**。

```java
public class EnumBasic {
    public static void main(String[] args) {
        Color c = Color.RED;
        System.out.println(c);                      // RED（toString() 默认返回常量名）
        System.out.println(c.name());               // RED
        System.out.println(c.ordinal());            // 0（按声明顺序编号，从 0 开始）
        System.out.println(c == Color.RED);         // true：枚举常量是单例，可以用 ==
        System.out.println(c.equals(Color.RED));    // true：equals() 内部就是 ==

        for (Color color : Color.values()) {        // 遍历全部常量
            System.out.println(color + " -> " + color.ordinal());
        }
        // 输出：
        // RED -> 0
        // GREEN -> 1
        // BLUE -> 2
    }
}
```

## 2.2 构造器、字段与方法

给常量加参数，就能让每个常量携带自己的数据。语法是"**常量名后跟一个括号，里面是构造器参数**"：

```java
public enum Planet {
    MERCURY(3.303e+23, 2.4397e6),
    VENUS  (4.869e+24, 6.0518e6); // 常量列表后必须加分号（因为后面还有成员声明）

    private final double mass;    // 质量，单位 kg
    private final double radius;  // 半径，单位 m

    // 构造器隐式为 private，不能用 public / protected 修饰
    Planet(double mass, double radius) {
        this.mass = mass;
        this.radius = radius;
    }

    public double mass()   { return mass; }
    public double radius() { return radius; }

    public static final double G = 6.67300E-11;   // 枚举也可以有静态常量

    public double surfaceGravity() {              // 枚举也可以有实例方法
        return G * mass / (radius * radius);
    }
}

// 使用：常量既可以当"枚举值"用，也可以直接当"带数据的小对象"用
class PlanetTest {
    public static void main(String[] args) {
        // 每个常量在类初始化时就已经通过构造器把数据装好了
        System.out.printf("%.4f%n", Planet.MERCURY.surfaceGravity());   // 3.7030
        System.out.printf("%.4f%n", Planet.VENUS.surfaceGravity());     // 8.8714
        System.out.println(Planet.VENUS.mass());                        // 4.869E24
        System.out.println(Planet.MERCURY.ordinal());                   // 0：顺带还能当枚举值遍历
    }
}
```

要点汇总：

| 要点 | 说明 |
|------|------|
| 构造器**隐式为 private** | 不能用 `public` / `protected` 修饰；显式写 `private` 合法但多余 |
| 常量即实例 | 每个常量实际上是通过调用对应构造器创建的实例，**只在类初始化时创建一次** |
| 常量列表后**必须加分号** | 只要声明了字段或方法，那个 `;` 就不能省 |
| 参数与构造器一一对应 | `MERCURY(3.303e+23, 2.4397e6)` 本质就是 `new Planet("MERCURY", 0, 3.303e+23, 2.4397e6)` 的语法糖，前两个参数由编译器自动补上 |
| 不能继承别的类 | 已隐式继承 `java.lang.Enum`；**但可以实现接口** |
| 常量名有作用域 | 常量名就像类的静态字段，跨枚举重名不冲突（`Color.RED` 与 `Status.RED` 互不干扰） |

> **陷阱**：**枚举构造器里不能引用本枚举的静态字段**（除了那些常量本身）。因为静态字段的初始化排在常量创建之后，编译器会直接报"非法引用静态字段"。下面这段编译不过：

```java
public enum Status {
    OK(1),
    FAIL(CODE);          // ✗ 编译错误：不能在构造器调用中引用静态字段 CODE

    static final int CODE = 500;
    Status(int code) { }
}
```

解决办法是把那个值写成构造器参数、或者放进一个独立的工具类/常量类里。

# 3. 常量专有行为与枚举的多态

如果某个常量的行为和其他常量不一样，可以在常量后面直接跟一个 `{}` 类体，为它单独实现方法。类里再声明一个 `abstract` 方法，编译器就会强制**每个常量都必须给出实现**：

```java
public enum Operation {
    PLUS {                        // 常量后带 { } 类体，为它单独实现 apply
        @Override
        double apply(double x, double y) { return x + y; }
    },
    MINUS {
        @Override
        double apply(double x, double y) { return x - y; }
    },
    TIMES {
        @Override
        double apply(double x, double y) { return x * y; }
    },
    DIVIDE {
        @Override
        double apply(double x, double y) { return x / y; }
    };

    abstract double apply(double x, double y);   // 抽象方法，强制每个常量实现
}

class OperationTest {
    public static void main(String[] args) {
        double x = 8, y = 2;
        for (Operation op : Operation.values()) {
            // %s 打印枚举会调用 toString()，默认就是常量名
            System.out.printf("%.1f %s %.1f = %.1f%n", x, op, y, op.apply(x, y));
        }
        // 输出：
        // 8.0 PLUS 2.0 = 10.0
        // 8.0 MINUS 2.0 = 6.0
        // 8.0 TIMES 2.0 = 16.0
        // 8.0 DIVIDE 2.0 = 4.0
    }
}
```

**每个带类体的常量会被编译为该枚举的匿名子类实例**，这一点用 `javap` 或看 `out/` 目录里的 `.class` 文件能直接验证：

| 写法 | 编译产物 | 含义 |
|------|----------|------|
| `RED, GREEN` | `Color` 自身的实例 | 普通常量 |
| `PLUS { ... }` | `Operation$1`（`Operation` 的匿名子类）的实例 | 常量专有行为 |

所以第 1.1 节说枚举类是 `final` 的、不能被继承，但**编译器自己可以给常量生成子类**——这正是它实现"每个常量有不同行为"的手法，等价于手写一个抽象类加四个匿名子类，只是省掉了所有样板代码。

> **最佳实践**：常量专有行为让枚举天然具备**多态**能力，这是它区别于 `static final int` 的最强特性——`switch`/`if` 分支被"下推"到各自的常量体里，新增一个常量时**编译器会强制你实现抽象方法，不可能漏掉分支**。这正是下一节"用枚举替代 switch"的核心动机。

# 4. 常用 API

## 4.1 五个必会方法

| 方法 | 来源 | 说明 | 示例 |
|------|------|------|------|
| `static E[] values()` | 编译器生成 | 返回包含全部常量的数组，顺序与声明一致 | `Color[] cs = Color.values();` |
| `static E valueOf(String name)` | 编译器生成 | 按名字查找常量，找不到抛 `IllegalArgumentException` | `Color.valueOf("RED")` |
| `String name()` | `Enum`（`final`） | 返回常量名，即声明时的标识符 | `Color.RED.name()` → `"RED"` |
| `int ordinal()` | `Enum` | 返回**声明顺序**编号，从 0 开始 | `Color.BLUE.ordinal()` → `2` |
| `int compareTo(E o)` | `Enum` | 按 `ordinal()` 比较，返回差值 | `Color.RED.compareTo(Color.BLUE)` → `-2` |

另有几个不太常用但值得知道的：`toString()` 可以覆写（`name()` 是 `final` 的，覆写不了）；`getDeclaringClass()` 返回常量所属的枚举类（对带类体的常量，它返回外层枚举而不是匿名子类）；`Enum.valueOf(Class, String)` 是 `Class` 版本的反查入口。

```java
public class EnumApiDemo {
    public static void main(String[] args) {
        System.out.println(Color.values().length);               // 3
        System.out.println(Color.valueOf("GREEN"));              // GREEN
        System.out.println(Color.GREEN.name());                  // GREEN
        System.out.println(Color.GREEN.ordinal());               // 1
        System.out.println(Color.RED.compareTo(Color.BLUE));     // -2（0 - 2）

        // valueOf 找不到常量时抛 IllegalArgumentException
        try {
            Color.valueOf("PINK");
        } catch (IllegalArgumentException e) {
            System.out.println("不存在这个常量：" + e.getMessage());
            // 不存在这个常量：No enum constant Color.PINK
        }

        System.out.println(Color.RED == Color.valueOf("RED"));   // true：单例，== 可靠且更快

        // values() 每次返回的都是新数组，改它不会影响枚举本身
        System.out.println(Color.values() == Color.values());    // false
        Color[] first = Color.values();
        first[0] = null;                                         // 只改了副本
        System.out.println(Color.RED);                           // RED：枚举安然无恙
    }
}
```

> **注意**：`values()` 每次调用都会**新建并克隆一个数组**（内部是 `$VALUES.clone()`），返回的数组因此是安全的。但在热路径里反复调用会产生大量临时对象——常量数量固定时应该缓存：`private static final Color[] ALL = Color.values();`。

## 4.2 ordinal() 的陷阱

`ordinal()` 是**声明顺序**的编号，看起来像一个天然的 id，所以特别容易被误用。它绝不能作为业务上的标识：

| 错误用法 | 后果 |
|----------|------|
| 存进数据库当状态码 | 在中间插一个新常量，后面所有常量的 `ordinal()` 全部平移，历史数据全部对不上 |
| 当业务判断依据 | `if (status.ordinal() > 1)` 里的 `1` 是什么？不可读，且随常量增删而失效 |
| 用来排序 | 顺序由声明位置决定，挪一行声明就改变排序结果，不是显式表达的意图 |
| 用于网络协议或序列化 | 与数据库同理：跨版本兼容必然出问题 |

正确做法是**显式定义一个 code 字段**，把它当作对外的持久化标识：

```java
public enum OrderStatus {
    CREATED(10, "已创建"),
    PAID(20, "已付款"),
    SHIPPED(30, "已发货"),
    FINISHED(40, "已完成");

    private final int code;        // 显式业务码，持久化用这个
    private final String desc;

    OrderStatus(int code, String desc) {
        this.code = code;
        this.desc = desc;
    }

    public int getCode()    { return code; }
    public String getDesc() { return desc; }

    // 通过业务码反查枚举：常量数量很小，线性遍历完全够用
    public static OrderStatus ofCode(int code) {
        for (OrderStatus s : values()) {
            if (s.code == code) return s;
        }
        throw new IllegalArgumentException("未知的订单状态码：" + code);
    }
}
```

```java
// 使用
System.out.println(OrderStatus.PAID.getCode());          // 20：入库时存这个
System.out.println(OrderStatus.ofCode(20).getDesc());    // 已付款：查回来时用这个
System.out.println(OrderStatus.PAID.ordinal());          // 1：这个数永远不要存
```

如果反查在热路径上，就把遍历换成静态 `Map`，类初始化时建好索引：

```java
private static final Map<Integer, OrderStatus> CODE_INDEX = new HashMap<>();

static {
    for (OrderStatus s : values()) {
        CODE_INDEX.put(s.code, s);
    }
}
```

> **陷阱**：`ordinal()` 的正当用途**只有一个**——给 JDK 自己用（`EnumSet` 的位向量、`EnumMap` 的数组下标、编译器生成的 `switch` 映射表，都靠它定位）。业务代码里凡是看到 `ordinal()`，几乎都应该改成显式的 code 字段。

# 5. 枚举与 switch

`switch` 原生支持枚举，而且 `case` 里**只写常量名、不写类名前缀**（写了反而报错）：

```java
public class EnumSwitch {
    public static void main(String[] args) {
        OrderStatus status = OrderStatus.PAID;

        // Java 14+ 的 switch 表达式：覆盖全部常量时不需要 default
        String tip = switch (status) {
            case CREATED  -> "等待付款";
            case PAID     -> "等待发货";
            case SHIPPED  -> "等待收货";
            case FINISHED -> "交易完成";
        };
        System.out.println(tip);   // 等待发货
    }
}
```

用枚举配 `switch` 有三点好处和一个注意：

1. **编译期穷举检查**。`switch` 表达式如果覆盖了全部常量，就不需要 `default`；等以后新增一个常量，这里会**直接编译报错**，强迫你处理新分支——这是枚举 + switch 相比 `int` + switch 最大的优势。
2. **可读性**。`case PAID` 比 `case 20` 清楚得多。
3. **类型安全**。`switch` 的表达式类型必须是枚举，传错类型编译不过。

注意：传统的 `switch` **语句**（在分支里执行动作而不是返回值）仍然建议保留 `default`，并在其中抛 `IllegalStateException`，作为"不可能到达"的兜底——语句形式的穷举检查不如表达式形式严格。

> **注意**：枚举的 `switch` 底层其实**仍然是按 `ordinal()` 分发**的。旧版 `javac` 会生成一个 `$SwitchMap$...` 静态数组把 `ordinal()` 映射成连续整数，新版（JDK 21+）改用 `invokedynamic` 的 `enumSwitch` 引导方法。这层映射完全由编译器生成，所以**声明顺序变化不会影响业务代码的正确性**——但这也再次说明：`ordinal()` 是给编译器用的，不是给你用的。

# 6. EnumSet 与 EnumMap

JDK 专门为枚举提供了两个集合实现。它们比 `HashSet`/`HashMap` 高效得多，原因很直接：**元素的全体和下标在编译期就已知**，不需要哈希。

| 维度 | `EnumSet` | 普通 `HashSet` |
|------|-----------|----------------|
| 底层结构 | **位向量**：≤64 个常量时用一个 `long`（`RegularEnumSet`），超过则用 `long[]`（`JumboEnumSet`） | 哈希表 + 链表/红黑树 |
| 单个元素占用 | 1 个 bit | 一个 `Node` 对象（几十字节） |
| 遍历顺序 | **按 `ordinal()` 顺序**，天然有序 | 无序 |
| 判存在 | 一次位与运算 | `hashCode()` + 比较 |
| 是否允许 `null` | 不允许，插入 `null` 抛 `NullPointerException` | 允许 |

| 维度 | `EnumMap` | 普通 `HashMap` |
|------|-----------|----------------|
| 底层结构 | `Object[]`，**直接用 `ordinal()` 当下标** | 哈希表 |
| 查找开销 | 一次数组随机访问，无哈希计算、无冲突 | `hashCode()` + 冲突处理 |
| 遍历顺序 | **按 key 的 `ordinal()` 顺序** | 无序 |
| `null` 键 | 不允许（枚举本身不能为 `null`） | 允许 |

```java
public enum Day { MON, TUE, WED, THU, FRI, SAT, SUN }

public class EnumCollectionDemo {
    public static void main(String[] args) {
        // EnumSet：适合表示"某几个枚举值"的集合
        EnumSet<Day> workdays = EnumSet.range(Day.MON, Day.FRI);   // 取区间：周一到周五
        EnumSet<Day> weekend  = EnumSet.of(Day.SAT, Day.SUN);
        System.out.println(workdays);                              // [MON, TUE, WED, THU, FRI]
        System.out.println(workdays.contains(Day.WED));            // true

        // 其他常用工厂方法
        EnumSet<Day> all      = EnumSet.allOf(Day.class);           // 全部常量
        EnumSet<Day> none     = EnumSet.noneOf(Day.class);          // 空集，之后可 add
        EnumSet<Day> workOnly = EnumSet.complementOf(weekend);      // 补集：非周末

        // EnumMap：以枚举为 key 的映射，底层就是一个按 ordinal 索引的数组
        EnumMap<Day, Integer> hours = new EnumMap<>(Day.class);     // 构造时必须传 Class 对象
        hours.put(Day.MON, 8);
        hours.put(Day.SAT, 0);
        for (var entry : hours.entrySet()) {
            System.out.println(entry.getKey() + " 工作 " + entry.getValue() + " 小时");
        }
        // 输出（按声明顺序，不是插入顺序）：
        // MON 工作 8 小时
        // SAT 工作 0 小时
    }
}
```

`EnumSet` 是一个**抽象类**，实际用哪个实现由 `allOf`/`of`/`range` 等工厂方法在运行时决定，不能直接 `new`；`EnumMap` 的构造器必须接收 `Class` 对象，就是为了在运行时拿到枚举类型、创建对应长度的数组。

> **最佳实践**：只要元素或 key 是枚举类型，就**优先用 `EnumSet` / `EnumMap`**，几乎没有例外——它们更省内存、更快，而且**遍历顺序是确定的**（按声明顺序）。最后这一点在生成报表、写测试断言时价值极大：输出可复现，不用为"顺序不稳定"写额外的排序代码。

# 7. 枚举实现单例

《Effective Java》里有一句很著名的话：**单元素的枚举类型已经成为实现 Singleton 的最佳方法**。原因是枚举天然把三件麻烦事一起解决了：

| 威胁 | 双重检查锁（DCL） | 饿汉式 | **枚举单例** |
|------|------------------|--------|-------------|
| 多线程并发创建 | 需要 `volatile` + 双重检查，少写一个判断就出错 | 类加载时创建，安全 | **JVM 保证枚举常量在类初始化时只创建一次，天然线程安全** |
| 反射攻击 | `setAccessible(true)` 后调私有构造器，就能再造一个实例 | 同样可被攻破 | **`Constructor.newInstance()` 检测到目标是枚举，直接抛 `IllegalArgumentException`** |
| 反序列化破坏 | 必须手写 `readResolve()` 才能保住单例 | 同样需要 | **JVM 反序列化枚举时按"类名 + 常量名"查找已有实例，从不新建对象** |
| 代码量 | 十几行，且容易写错 | 少 | 一个常量 |

完整例子：

```java
public enum Singleton {
    INSTANCE;                       // 唯一的实例，名字约定用 INSTANCE

    // 单例自己的状态。注意：枚举只保证"只有一个对象"，不保证状态访问的线程安全
    private int counter;

    public void doSomething() {
        counter++;
        System.out.println("Singleton 被调用了 " + counter + " 次");
    }

    public int getCounter() {
        return counter;
    }
}

class SingletonTest {
    public static void main(String[] args) {
        Singleton.INSTANCE.doSomething();       // Singleton 被调用了 1 次
        Singleton.INSTANCE.doSomething();       // Singleton 被调用了 2 次

        // 无论从哪里取，拿到的都是同一个对象
        Singleton a = Singleton.INSTANCE;
        Singleton b = Singleton.INSTANCE;
        System.out.println(a == b);             // true

        // 反射也造不出第二个实例
        try {
            var ctor = Singleton.class.getDeclaredConstructors()[0];
            ctor.setAccessible(true);           // 就算放开访问检查也没用
            ctor.newInstance();
        } catch (Exception e) {
            System.out.println("反射创建失败：" + e.getMessage());
            // 反射创建失败：Cannot reflectively create enum objects
        }
    }
}
```

> **注意**：枚举单例保证的是"**只有一个实例**"，**不保证实例内部状态的线程安全**。上面那个 `counter++` 依然是非原子操作，多线程下照样会算错，需要加锁或换成 `AtomicInteger`。"枚举单例天生线程安全"这句话说的是**创建过程**安全，不是**状态访问**安全，这一点极容易被混淆。

# 8. 实战：用枚举替代 switch

## 8.1 策略模式：会员折扣

把"每种情况的算法"下推到常量体里，就是枚举版的策略模式。

```java
public enum MemberLevel {
    NORMAL("普通会员") {
        @Override
        public double discount(double price) { return price; }           // 不打折
    },
    SILVER("白银会员") {
        @Override
        public double discount(double price) { return price * 0.95; }    // 95 折
    },
    GOLD("黄金会员") {
        @Override
        public double discount(double price) { return price * 0.90; }    // 9 折
    },
    DIAMOND("钻石会员") {
        @Override
        public double discount(double price) { return price * 0.80; }    // 8 折
    };

    private final String label;

    MemberLevel(String label) { this.label = label; }
    public String getLabel()  { return label; }

    // 抽象方法：编译器强制每个常量给出自己的算法，新增等级时不可能漏
    public abstract double discount(double price);
}

class MemberLevelTest {
    public static void main(String[] args) {
        double price = 1000;
        for (MemberLevel level : MemberLevel.values()) {
            System.out.printf("%s 应付 %.2f 元%n", level.getLabel(), level.discount(price));
        }
        // 输出：
        // 普通会员 应付 1000.00 元
        // 白银会员 应付 950.00 元
        // 黄金会员 应付 900.00 元
        // 钻石会员 应付 800.00 元
    }
}
```

两种实现方式的对比：

| 维度 | `switch` + 常量 | 枚举 + 常量专有行为 |
|------|----------------|-------------------|
| 新增一种策略 | 要找到所有 `switch` 逐处补 `case`，容易漏 | 只加一个常量，编译器强制实现抽象方法 |
| 算法归属 | 散落在各个方法里，与数据分离 | 与常量放在一起，内聚 |
| 类型安全 | 靠 `String` 或 `int`，传错不报错 | 参数类型就是枚举，编译期拦截 |
| 可测试性 | 要构造各种输入才能覆盖分支 | 每个常量可单独调用测试 |
| 是否需要兜底分支 | 通常要写 `default`，且是"不可达"的噪音 | 抽象方法天然穷举，无需兜底 |
| 适合的复杂度 | 分支逻辑简单、只在少数几处判断 | 分支带数据、带行为、分散在多处 |

## 8.2 状态机：订单状态流转

把"每个状态允许的下一步"绑定在常量上，状态流转规则就集中在一个文件里：

```java
public enum OrderState {
    CREATED {
        @Override
        public OrderState next() { return PAID; }
    },
    PAID {
        @Override
        public OrderState next() { return SHIPPED; }
    },
    SHIPPED {
        @Override
        public OrderState next() { return FINISHED; }
    },
    FINISHED {
        @Override
        public OrderState next() { return this; }   // 终态：停在原地
    };

    public abstract OrderState next();

    public boolean isFinal() { return next() == this; }
}

class OrderStateTest {
    public static void main(String[] args) {
        OrderState state = OrderState.CREATED;
        while (!state.isFinal()) {
            System.out.println("当前状态：" + state + " -> 流转到：" + state.next());
            state = state.next();
        }
        System.out.println("最终状态：" + state);
        // 输出：
        // 当前状态：CREATED -> 流转到：PAID
        // 当前状态：PAID -> 流转到：SHIPPED
        // 当前状态：SHIPPED -> 流转到：FINISHED
        // 最终状态：FINISHED
    }
}
```

> **最佳实践**：把"数据"和"对数据的操作"放在一起，是枚举相对 `switch` 的核心优势。如果某段分支逻辑只在一个方法里出现一次，用 `switch` 反而更直观；一旦同一组判断散落在三四个方法里，就该把它收进枚举——**判断标准是"重复次数"和"是否带数据"，不是"哪个更高级"**。

# 9. 小结

| 主题 | 一句话要点 |
|------|-----------|
| 本质 | `enum` 编译后是 `final class X extends Enum<X>`；每个常量是 `public static final` 的实例，在 `static {}` 中创建一次 |
| 对比静态常量 | 枚举赢在**类型安全、可遍历、可带属性、原生 `switch`、天然单例**；取值固定就该用枚举 |
| 构造器与字段 | 构造器**只能是 private**；常量后的参数列表对应构造器参数；声明成员后常量列表必须加分号 |
| 常量专有行为 | 常量带 `{}` 类体 = 匿名子类实例，让枚举具备多态；抽象方法强制补全实现，新增常量不会漏分支 |
| 常用 API | `values()`/`valueOf()`/`name()`/`ordinal()`/`compareTo()`；`values()` 每次返回新数组，可缓存 |
| `ordinal()` | **只给 JDK 内部用**；持久化、判等、排序一律用显式的 code 字段 |
| `EnumSet`/`EnumMap` | 位向量与按 `ordinal` 索引的数组，更快更省内存，遍历有序，凡枚举当元素/key 就用它俩 |
| 枚举单例 | 一个 `INSTANCE` 常量搞定，防并发、防反射、防反序列化；但**不保证实例状态的线程安全** |
| 替代 `switch` | 把分支下推到常量体，从"数据与算法分离"变成"数据自带算法" |
| 局限 | 不能继承其他类（已继承 `Enum`），但可实现接口；常量集合固定，无法在运行时动态增减 |

下一篇（第 21 篇）进入 JNI，看 `native` 关键字如何让 Java 调用 C/C++ 写的本地代码，以及 `javac -h` 生成头文件的完整流程。
