`record` 是 JDK14 预览、**JDK16 正式**引入的一类特殊的类，专门用来表达"不可变数据载体"（data carrier）——也就是那些"装一组数据、别的什么都不干"的类。它把过去必须手写几十行的构造器、访问器、`equals`、`hashCode`、`toString` 压缩成一行声明，同时用编译器强制保证不可变。本篇讲清 **Record 解决什么问题、语法与编译产物、紧凑构造器的用法、有哪些限制**，最后落到它与模式匹配、密封接口组合出的现代写法。

# 1. Record 解决什么问题

## 1.1 手写数据载体类的样板代码

假设要表示一个坐标点，用传统的 JavaBean 写法需要这么多代码：

```java
import java.util.Objects;

public class Point {
    private final int x;
    private final int y;

    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }

    public int getX() { return x; }
    public int getY() { return y; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;                       // 同一引用直接相等
        if (o == null || getClass() != o.getClass()) return false;
        Point other = (Point) o;
        return x == other.x && y == other.y;
    }

    @Override
    public int hashCode() {
        return Objects.hash(x, y);
    }

    @Override
    public String toString() {
        return "Point[x=" + x + ", y=" + y + "]";
    }
}
```

接近 40 行，其中**真正携带信息的只有 `int x` 和 `int y` 两个字段名**，剩下的全是机械重复的样板代码（boilerplate）。更糟的是这份代码**每加一个字段就要手动改四五个地方**，`equals` 里漏比较一个字段是经典的隐蔽 bug——`HashMap` 里取不到值、`Set` 里去不掉重复元素，往往就是这么来的。

## 1.2 record 一行搞定

```java
public record Point(int x, int y) { }
```

**就这一行**，行为与上面 40 行完全等价（除了访问器名字，见 2.3）。

## 1.3 与 Lombok @Data 的对比

Lombok 是过去十年 Java 程序员对付样板代码的主流方案，`record` 则是语言层面的官方答案。两者不是简单的替代关系：

| 维度 | `record` | Lombok `@Data` |
|------|----------|----------------|
| 提供方式 | **语言原生**（JDK16+） | 第三方库 + 编译期注解处理器 |
| 依赖 | 无，开箱即用 | 需要引 `lombok` 依赖，IDE 还要装插件 |
| 可变性 | 字段隐式 `final`，**不可变**，没有 `setter` | 可变，自动生成 `setXxx()` |
| 访问器命名 | `x()`（**方法名就是字段名**） | `getX()` / `isX()`（符合 JavaBean 规范） |
| 继承 | 隐式 `final`，不能被继承，也不能继承别的类 | 无限制，可以继承/被继承 |
| 字段位置 | 只能写在类头部，**不能额外声明实例字段** | 任意字段随便加 |
| 能否实现接口 | 能 | 能 |
| 反射识别 | `Class#isRecord()` 可以直接判断 | 与普通类无区别 |
| 编译产物 | 由 `javac` 原生生成 | 靠注解处理器往字节码里"注入"方法 |
| IDE 兼容性 | 原生支持，跳转补全都正常 | 依赖插件，偶发"方法不存在"的误报 |
| 框架兼容性 | 新框架普遍已支持（Jackson 2.12+、MyBatis 3.5.x+） | 兼容性最好，老框架也认 |

**选择建议**：

| 场景 | 推荐 |
|------|------|
| 新项目、纯不可变数据载体 | **`record`**，语言原生、零依赖、语义清晰 |
| 需要可变对象（有 `setter`） | 只能用 Lombok 或手写 |
| 需要继承结构、需要自定义 `setter` 逻辑 | Lombok 或手写 |
| 老项目、老框架（对 record 支持差） | 继续用 Lombok，别为了"新"而引入风险 |
| 团队已有 Lombok 体系且用到 `@Builder`、`@Slf4j` | 保留 Lombok；record 可与 Lombok 共存但不叠加 `@Data` |

> **行业实践**：`record` 与 Lombok **不是二选一**。常见做法是——数据载体、DTO、返回值用 `record`，业务类继续用 Lombok 的 `@Getter`/`@Builder`/`@Slf4j`。不要给 `record` 再套 `@Data`（多余的，而且 `@Data` 想生成的 `setter`、无参构造器在 record 上根本无法实现）。

# 2. 基本语法与编译产物

## 2.1 语法

```java
// 最简形式：头部声明组件（components），类体可以为空
public record Point(int x, int y) { }

// 带泛型
public record Pair<K, V>(K key, V value) { }

// 实现接口
public record User(String name, int age) implements Comparable<User> {
    @Override
    public int compareTo(User other) {
        return Integer.compare(this.age, other.age);
    }
}

// 加自定义方法
public record Range(int lo, int hi) {
    /** 区间长度，自定义的实例方法 */
    public int length() {
        return hi - lo;
    }
}
```

## 2.2 编译器自动生成了什么

`public record Point(int x, int y) { }` 编译后等价于：

```java
public final class Point extends java.lang.Record {
    // ① 每个组件对应一个 private final 字段
    private final int x;
    private final int y;

    // ② 规范构造器（canonical constructor）：参数顺序与头部声明一致
    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }

    // ③ 访问器：方法名与组件名同名，没有 get 前缀
    public int x() { return x; }
    public int y() { return y; }

    // ④ equals：逐字段比较，且要求对方是同一个 record 类型
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || o.getClass() != this.getClass()) return false;
        Point other = (Point) o;
        return x == other.x && y == other.y;
    }

    // ⑤ hashCode：由所有组件一起算出
    @Override
    public int hashCode() {
        return java.util.Objects.hash(x, y);
    }

    // ⑥ toString：格式固定为 类名[字段1=值1, 字段2=值2]
    @Override
    public String toString() {
        return "Point[x=" + x + ", y=" + y + "]";
    }
}
```

几个必须记住的细节：

| 产物 | 说明 |
|------|------|
| 类本身是 `final` | 不能被继承 |
| 父类是 `java.lang.Record` | record **隐式继承**了它，所以不能再 `extends` 别的类 |
| 字段是 `private final` | 不可变，创建后无法修改 |
| 所有组件都是"一等公民" | `equals`/`hashCode`/`toString` 全部基于**全部**组件，不会漏字段 |
| `equals` 的类型判定 | 用 `getClass()` 而非 `instanceof`，所以 `Point` 与另一个同结构的 record 不相等 |

```java
// 直接验证编译产物的行为
public class RecordDemo {
    public static void main(String[] args) {
        Point p1 = new Point(1, 2);
        Point p2 = new Point(1, 2);

        System.out.println(p1);                       // Point[x=1, y=2]
        System.out.println(p1.equals(p2));            // true：逐字段比较
        System.out.println(p1.hashCode() == p2.hashCode());  // true
        System.out.println(p1.x());                   // 1：借访问器取字段值
        System.out.println(Point.class.isRecord());   // true：反射可以直接识别
        System.out.println(Point.class.getSuperclass());  // class java.lang.Record
    }
}
```

```text
Point[x=1, y=2]
true
true
1
true
class java.lang.Record
```

## 2.3 访问器叫 x() 而不是 getX()

这是从传统 JavaBean 转过来最容易踩的一脚：

```java
Point p = new Point(1, 2);
int a = p.x();        // ✓ 正确：方法名就是字段名
// int b = p.getX();  // ✗ 编译错误：没有这个方法
```

**为什么不叫 `getX()`？** 这是 Java 语言设计者的明确选择——`record` 表达的是"透明数据载体"，组件名本身就是语义，加 `get` 前缀既冗余又不符合"数据即值"的直觉。而且 `getX()` 这套命名来自 JavaBeans 规范，它是为可变 Bean 设计的，record 天然可变性为零。

> **陷阱**：**大量老框架依赖 `getX()` 命名约定**。把 record 交给它们用时会失败：
> - **Jackson**：2.12 之前不识别 record 的访问器，序列化出来是空对象；2.12+ 才原生支持 record。
> - **MyBatis**：默认靠 JavaBean 约定映射，且 record 没有无参构造器，需要配置构造器映射（3.5.x 起支持自动构造器映射）。
> - **JSP / 老式 EL 表达式 `${user.name}`**：底层走 `getName()`，在 record 上取不到值。
> - **手写的工具类**：任何用 `Introspector` 或 `PropertyDescriptor` 的地方都认不出 record。
>
> 迁移到 record 前，先确认上下游框架的版本支持。

# 3. 构造器

record 支持三种构造器写法，各有明确用途。

## 3.1 规范构造器（canonical constructor）

参数列表与 record 头部**完全一致**的构造器。编译器默认生成一个；如果需要自定义逻辑（校验、拷贝、规范化），可以显式覆盖：

```java
public record Point(int x, int y) {
    /** 显式覆盖规范构造器：参数必须与头部完全一致，且必须给所有字段赋值 */
    public Point(int x, int y) {
        if (x < 0 || y < 0) {
            throw new IllegalArgumentException("坐标不能为负：" + x + "," + y);
        }
        this.x = x;
        this.y = y;
    }
}
```

> **注意**：显式写规范构造器时，**必须给每一个字段赋值**，否则编译报错。这是编译器在帮你保证"不存在半初始化的 record 对象"。

## 3.2 紧凑构造器（compact constructor）

只想做校验/规范化、不想重复写参数列表和赋值语句时，用紧凑构造器——**省略参数列表**：

```java
public record Range(int lo, int hi) {
    /** 紧凑构造器：不写参数列表，也不写 this.lo = lo，编译器在末尾自动补齐赋值 */
    public Range {
        if (lo > hi) {
            throw new IllegalArgumentException("lo 不能大于 hi：" + lo + " > " + hi);
        }
        // 可以对参数重新赋值，末尾的隐式赋值会写入"规范化之后"的值
        lo = Math.max(lo, 0);
    }
}
```

```java
System.out.println(new Range(3, 7));    // Range[lo=3, hi=7]
System.out.println(new Range(-5, 7));   // Range[lo=0, hi=7]  ← 规范化生效
new Range(7, 3);                        // 抛 IllegalArgumentException: lo 不能大于 hi：7 > 3
```

紧凑构造器的三条规则：

| 规则 | 说明 |
|------|------|
| **不能写参数列表** | 写了 `()` 就变成普通构造器，不再是紧凑形式 |
| **不能写 `this.x = x`** | 编译器自动在构造器末尾补上字段赋值；手写反而编译错误 |
| **不能改变"每个参数赋给同名字段"这条规则** | 可以给参数重新赋值做规范化，但不能把 `lo` 赋给 `hi` 字段；参数与字段的对应关系是固定的 |

> **最佳实践**：紧凑构造器是 record 最优雅的一个特性，**它是为"让非法对象无法被创建"而生的**。把校验放进紧凑构造器后，只要对象存在，就一定能保证不变量（invariant）成立——这比"写完 setter 再靠外部校验"可靠得多，也让"值对象"在领域驱动设计中真正站稳了脚。

## 3.3 其他构造器必须委托给规范构造器

除了规范构造器，其他构造器都是"附带的"（非规范构造器），**第一行必须 `this(...)` 委托给规范构造器**，不能直接给字段赋值：

```java
public record Point(int x, int y) {
    /** 无参构造：委托给规范构造器 */
    public Point() {
        this(0, 0);
    }

    /** 单参构造：委托给规范构造器 */
    public Point(int v) {
        this(v, v);
    }

    /** 从一个字符串解析 */
    public Point(String s) {
        this(Integer.parseInt(s.split(",")[0]), Integer.parseInt(s.split(",")[1]));
    }
}
```

```java
System.out.println(new Point());          // Point[x=0, y=0]
System.out.println(new Point(5));         // Point[x=5, y=5]
System.out.println(new Point("1,2"));     // Point[x=1, y=2]
```

> **陷阱**：record 的所谓"没有无参构造器"只对**默认情况**成立。很多框架（尤其是老的 ORM、序列化库）要求无参构造器，可以**手动补一个委托给规范构造器的无参构造器**来满足它们——但这样做通常意味着该 record 的 `equals` 会把两个"空对象"判为相等，语义上要交代清楚。

# 4. 限制与能做的事

## 4.1 不能做的事

| 限制 | 原因 | 报错信息 |
|------|------|----------|
| **不能 `extends` 其他类** | record 已隐式继承 `java.lang.Record`，Java 是单继承 | `class cannot extend any class` |
| **不能声明实例字段** | 组件的完整列表必须在头部一次性声明，保证 `equals`/`hashCode` 覆盖所有状态 | `instance field is not allowed in record` |
| **隐式 `final`，不能被继承** | 保证不可变性和 `equals` 语义 | `cannot inherit from final class` |
| **不能声明 `abstract`** | 隐式 `final` 与 `abstract` 冲突 | `record class cannot be abstract` |
| **组件字段隐式 `final`，没有 setter** | 不可变性的核心 | `cannot assign a value to final variable` |
| **不能声明 `native` 方法** | 与编译器自动生成的方法冲突，且破坏语义一致性 | 编译报错 |

```java
// ✗ 以下全部编译失败
// public record Bad(int x) extends Object { }        // 不能继承类
// public record Bad2(int x) { private int y; }       // 不能有实例字段
// public abstract record Bad3(int x) { }             // 不能是 abstract
public class Sub extends Point { }                    // ✗ 不能被继承（假设 Point 是 record）
```

> **陷阱**：**JPA 实体不能用 record**。`@Entity` 要求：(1) 有**无参构造器**供 Hibernate 反射实例化；(2) 字段**可变**，供脏检查（dirty checking）和代理生成；(3) 类**不能是 final**，Hibernate 需要生成子类代理做懒加载。record 三条全都不满足。硬要对 record 加 `@Entity`，会在启动或运行期报 `No default constructor` / `Cannot extend final class` 之类的错误。

## 4.2 可以有的事

record 的限制只针对"状态"，行为可以随便加：

```java
public record Point(int x, int y) {
    // ✓ 静态字段：属于类而非实例，不影响 equals/hashCode
    public static final Point ORIGIN = new Point(0, 0);
    private static final int MAX = 1000;

    // ✓ 静态初始化块
    static {
        System.out.println("Point 类初始化");
    }

    // ✓ 静态方法
    public static Point of(int x, int y) {
        return new Point(x, y);
    }

    // ✓ 实例方法
    public int sum() {
        return x + y;
    }

    // ✓ 重写 equals / hashCode / toString（一般不需要，除非有特殊语义）
    @Override
    public String toString() {
        return "(" + x + ", " + y + ")";
    }

    // ✓ 实现接口（甚至能实现 sealed interface，见 7.2）
}
```

| 允许的成员 | 说明 |
|-----------|------|
| 静态字段 | 必须 `static`，可以是 `final` 也可以不是 |
| 静态方法 | 常用作工厂方法，如 `Point.of(...)` |
| 实例方法 | 任意方法，只是不能修改组件 |
| 静态/实例初始化块 | 实例初始化块里不能给组件赋值 |
| 其他构造器 | 必须委托给规范构造器 |
| 重写 `equals` / `hashCode` / `toString` | 允许，但会失去"自动覆盖全部组件"的安全保证 |
| 实现接口 | 可以 `implements`，不能 `extends` |
| 嵌套 record | record 可以定义在类或另一个 record 内部 |

# 5. 序列化

record 默认**不实现** `Serializable`，需要显式声明：

```java
import java.io.*;

public record Range(int lo, int hi) implements Serializable {
    /** 紧凑构造器里的校验，反序列化时同样会执行 */
    public Range {
        if (lo > hi) {
            throw new IllegalArgumentException("lo 不能大于 hi：" + lo + " > " + hi);
        }
    }
}
```

```java
public class SerializeDemo {
    public static void main(String[] args) throws Exception {
        Range original = new Range(1, 10);

        // 序列化
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        try (ObjectOutputStream oos = new ObjectOutputStream(bos)) {
            oos.writeObject(original);
        }

        // 反序列化
        try (ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(bos.toByteArray()))) {
            Range restored = (Range) ois.readObject();
            System.out.println(restored);                  // Range[lo=1, hi=10]
            System.out.println(restored.equals(original)); // true
        }
    }
}
```

```text
Range[lo=1, hi=10]
true
```

**record 序列化最值得注意的一点**：它**不走普通 Java 对象那套"绕过构造器"的私有协议**。普通 `Serializable` 类反序列化时是 JVM 直接往内存里填字段，**构造器完全不执行**（这也是反序列化漏洞的根源之一）；而 record 的反序列化**会调用规范构造器**，把流里的每个组件当作构造参数传进去。

| 维度 | 普通 `Serializable` 类 | `record` |
|------|----------------------|----------|
| 反序列化是否调用构造器 | **不调用** | **调用规范构造器** |
| 校验逻辑是否生效 | 不生效（数据可以被"走私"进来） | **生效**，非法数据直接导致反序列化失败 |
| 数据完整性 | 可能拿到不符合不变量的对象 | 不变量始终成立 |
| 序列化格式 | 记录所有字段 | 只记录组件（与字段一一对应） |
| 能否自定义 | 可用 `readObject`/`writeObject` 完全接管 | **禁止**自定义 `readObject`/`writeObject` |

> **注意**：正因为反序列化走规范构造器，如果紧凑构造器里抛异常，反序列化会失败并抛出 `InvalidObjectException`（包装了你抛出的异常）。这是**好事**——攻击者无法绕过校验构造出非法状态的对象。这也是为什么安全建议在新代码里优先用 record 承载需要序列化的数据。

# 6. 使用场景

## 6.1 适合的场景

| 场景 | 示例 | 收益 |
|------|------|------|
| **DTO / VO** | `record UserDTO(String name, int age) {}` | 少写 40 行，改字段不会漏改 `equals` |
| **方法返回多个值** | `record Range(int min, int max) {}` | 免去 `int[]` 或 `Map<String, Object>` 这种弱类型返回 |
| **配置参数载体** | `record ServerConfig(String host, int port) {}` | 配置存在即合法，天然不可变可共享 |
| **值对象（DDD 的 Value Object）** | `record Money(long cents, String currency) {}` | 用紧凑构造器守住不变量 |
| **`Map` 的复合 key** | `record Coordinate(int x, int y) {}` | `equals`/`hashCode` 自动正确，避免手写漏字段 |
| **模式匹配的载体** | 见 7.2 的 ADT | record 可以被解构 |
| **事件对象** | `record OrderCreatedEvent(long id, Instant at) {}` | 事件天然不可变，跨线程传递安全 |
| **临时计算结果的缓存键** | `record CacheKey(String ns, String id) {}` | 生成的 `hashCode` 稳定可靠 |

```java
// 典型用法：一次返回两个值，避免包装出只出现一次的小 POJO
public record Range(int min, int max) { }

public static Range minMax(int[] arr) {
    int min = Integer.MAX_VALUE, max = Integer.MIN_VALUE;
    for (int v : arr) {
        min = Math.min(min, v);
        max = Math.max(max, v);
    }
    return new Range(min, max);
}

public static void main(String[] args) {
    Range r = minMax(new int[]{3, 1, 4, 1, 5, 9, 2, 6});
    System.out.println("最小值 " + r.min() + "，最大值 " + r.max());   // 最小值 1，最大值 9
}
```

配合 `var`（JDK10，见第 29 篇）一起用会更简洁——类型由编译器从右边推断，右边是 `record` 时不必再重复写一遍类名：

```java
// record 作为返回值时，var 让"接收结果"这一步也变短
var range = minMax(new int[]{3, 1, 4, 1, 5});
System.out.println(range.min() + " ~ " + range.max());   // 1 ~ 5

// 构造 record 时同样适用
var point = new Point(1, 2);
System.out.println(point);                               // Point[x=1, y=2]
```

## 6.2 不适合的场景

| 场景 | 原因 | 替代方案 |
|------|------|----------|
| **需要可变状态** | 没有 `setter`，字段 `final` | 普通类或 Lombok `@Data` |
| **需要继承结构** | record 隐式 `final`，也不能继承别人 | 抽象类 + 普通子类 |
| **JPA 实体（`@Entity`）** | 需要无参构造器 + 可变字段 + 非 `final`，三条全不满足 | 普通类 + Lombok |
| **需要携带大量自定义行为** | 类体会变成"什么都往外写"的杂货铺，失去简洁性 | 普通类 |
| **需要懒加载 / 代理** | 字段 `final`，无法被代理替换 | 普通类 |
| **框架强依赖 JavaBean 约定** | 访问器叫 `x()` 不是 `getX()` | 升级框架版本或改用普通类 |

```java
// ✗ 踩坑示例：想用 record 做 JPA 实体，启动就会失败
// @Entity
// @Table(name = "t_user")
// public record UserEntity(Long id, String name) { }
//
// 失败原因：
//   1. Hibernate 需要无参构造器反射实例化 → 没有
//   2. Hibernate 脏检查需要能修改字段 → 字段是 final
//   3. Hibernate 懒加载代理需要继承该类 → 类是 final
```

> **最佳实践**：JPA 场景的推荐做法是**实体用普通类，DTO 用 record**。查询结果直接投影成 record 是完全可以的——Hibernate 6 支持 `select new com.x.UserDTO(u.id, u.name)` 这类构造表达式，MyBatis 也可以用构造器映射，既拿到了 record 的简洁，又避开了实体对可变更性的硬要求。

# 7. 与现代语法结合

record 真正的威力，在于它和模式匹配、密封接口组合出的表达力——这三者合起来，Java 才算有了像样的**代数数据类型（ADT）** 支持。

## 7.1 record + instanceof 模式匹配解构

JDK16 的 `instanceof` 模式匹配 + JDK21 的 record 模式匹配，可以直接把 record 拆开取值：

```java
public record Point(int x, int y) { }
public record Line(Point start, Point end) { }

public class Deconstruct {
    /** 嵌套解构：一次性把 Line 里的两个 Point 的四个坐标全部取出来 */
    public static void print(Object o) {
        if (o instanceof Line(Point(var x1, var y1), Point(var x2, var y2))) {
            System.out.printf("线段从 (%d,%d) 到 (%d,%d)%n", x1, y1, x2, y2);
        } else {
            System.out.println("不是线段");
        }
    }

    public static void main(String[] args) {
        print(new Line(new Point(0, 0), new Point(3, 4)));   // 线段从 (0,0) 到 (3,4)
        print("hello");                                       // 不是线段
    }
}
```

过去要写 `((Line) o).start().x()` 这种一层层往下钻的代码，现在结构形状一写出来，变量就自动绑好了。

## 7.2 record + sealed interface 构建 ADT

**密封接口（sealed interface）** 限定"实现类只能有这几个"，record 提供"每种情形一个不可变载体"，两者一组合就是标准的代数数据类型：

```java
// 密封接口：穷举所有可能的形状，编译器知道只有这三种
public sealed interface Shape permits Circle, Rectangle, Triangle { }

public record Circle(double radius) implements Shape { }
public record Rectangle(double width, double height) implements Shape { }
public record Triangle(double base, double height) implements Shape { }
```

```java
import java.util.List;

public class AreaCalculator {
    /** 因为 Shape 是密封的，switch 已经穷举了所有情形，这里不需要 default 分支 */
    public static double area(Shape s) {
        return switch (s) {
            case Circle c    -> Math.PI * c.radius() * c.radius();
            case Rectangle r -> r.width() * r.height();
            case Triangle t  -> 0.5 * t.base() * t.height();
        };
    }

    /** 用 record 模式匹配直接解构，连访问器都不用调 */
    public static String describe(Shape s) {
        return switch (s) {
            case Circle(double r)                -> "半径 " + r + " 的圆";
            case Rectangle(double w, double h)   -> w + "×" + h + " 的矩形";
            case Triangle(double b, double h)    -> "底 " + b + " 高 " + h + " 的三角形";
        };
    }

    public static void main(String[] args) {
        List<Shape> shapes = List.of(new Circle(1), new Rectangle(2, 3), new Triangle(4, 5));
        for (Shape s : shapes) {
            System.out.printf("%-20s 面积 = %.2f%n", describe(s), area(s));
        }
    }
}
```

```text
半径 1.0 的圆          面积 = 3.14
2.0×3.0 的矩形         面积 = 6.00
底 4.0 高 5.0 的三角形  面积 = 10.00
```

这套写法的价值：

| 特性 | 说明 |
|------|------|
| **穷举检查** | 新增一种 `Shape` 时，所有 `switch` 都会编译报错，逼你把每个分支补全——不会漏处理 |
| **不需要 `default`** | 密封接口让编译器确信分支已穷举，少一层"永远不会走到"的兜底 |
| **模式即结构** | `case Circle(double r)` 直接把字段解构进变量，比 `c.radius()` 更直接 |
| **不可变 + 可解构** | record 保证状态不被篡改，模式匹配保证取用方式安全 |

> **行业实践**：这套 "`sealed interface` + `record` + `switch` 模式匹配" 是 Java 里表达**状态机、AST 节点、消息协议、领域事件**的现代范式。它替代了过去的"枚举 + 一堆 `instanceof`"和"访问者模式"，可读性和安全性都高一个档次。JDK21 起全部正式可用，新项目值得直接采用。

## 7.3 record 作为 switch 模式匹配的目标

record 也可以直接成为 `switch` 的主体（JDK21 起）：

```java
public record Order(String id, double amount, boolean paid) { }

public static String handle(Order o) {
    return switch (o) {
        // 守卫条件：在模式后加 when 子句，进一步筛选
        case Order(String id, double amount, boolean paid) when amount > 10000 && !paid ->
                "大额未支付订单，需要人工介入：" + id;
        case Order o2 when !o2.paid() ->
                "普通未支付订单";
        case Order o3 ->
                "已完成：" + o3.id();
    };
}

public static void main(String[] args) {
    System.out.println(handle(new Order("A001", 20000, false)));   // 大额未支付订单，需要人工介入：A001
    System.out.println(handle(new Order("A002", 100, false)));     // 普通未支付订单
    System.out.println(handle(new Order("A003", 100, true)));      // 已完成：A003
}
```

`when` 子句（guard）是模式匹配的重要补充：**模式负责"结构匹配"，`when` 负责"条件匹配"**，两者合起来才够表达业务规则。（`when` 关键字在 JDK21 预览、JDK24 起可用。）

> **注意**：`record` 模式匹配、`switch` 模式匹配、`when` 守卫子句都属于 **JDK21 才正式可用**的特性，部分语法（如 `when`）在更晚的版本才稳定。用之前先确认项目的 JDK 版本与编译参数（预览特性需要 `--enable-preview`）。这些特性的完整梳理见第 29 篇。

# 8. 小结

| 主题 | 一句话要点 |
|------|-----------|
| 解决什么问题 | 消灭不可变数据载体的样板代码；过去 40 行的 `Point`，现在一行 `record Point(int x, int y) {}` |
| 编译产物 | `final class`，隐式继承 `java.lang.Record`；自动生成 `private final` 字段、访问器、规范构造器、`equals`/`hashCode`/`toString` |
| 访问器命名 | 叫 `x()` **不是** `getX()`；老框架依赖 JavaBean 约定时会取不到值，注意版本 |
| 与 Lombok | record 是**语言原生 + 不可变 + 无 setter**；Lombok 是**编译期改字节码 + 可变 + 无限制**；不可变数据用 record，需可变或需继承用 Lombok，二者可共存 |
| 构造器 | **紧凑构造器** `public Range { ... }` 只做校验与规范化，不能写 `this.x = x`；其他构造器第一行必须 `this(...)` |
| 限制 | 不能 `extends`、不能有实例字段、隐式 `final` 不能 `abstract`、字段不可变；但**可以有静态字段/静态方法/实例方法/实现接口** |
| 序列化 | 需显式 `implements Serializable`；反序列化**会走规范构造器**，所以校验生效、非法数据无法被注入 |
| 适合 | DTO/VO、方法返回多值、`Map` 复合 key、值对象、事件对象、模式匹配的载体 |
| 不适合 | 需要可变状态、需要继承、**JPA 实体**（要无参构造 + 可变字段 + 非 final，三条全不满足） |
| 现代组合 | `record` + `sealed interface` + `switch` 模式匹配 = Java 的代数数据类型，穷举检查、可解构、无需 `default` |

`record` 是 Java 从"为可变对象设计"向"默认不可变"转向的标志性一步，它和密封接口、模式匹配一起构成了现代 Java 的骨架。下一篇（第 27 篇）转入 `NIO`，看 Java 如何处理非阻塞 IO 与内存映射文件；这些新特性的完整版本脉络可以对照第 29 篇一起看。
