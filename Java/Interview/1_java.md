本篇是 Java 语言层面的高频面试题合集，按"主题块（一级标题）→ 详细分块（二级标题）→ 面试题（三级标题）→ 解答（正文）"组织。五个主题块：语言基础与对象模型、集合框架、并发编程、Java 8+ 新特性、IO/NIO。新增题目直接在对应二级分块下追加三级标题即可。

# 1. 语言基础与对象模型

## 1.1 对象模型与类

### 1.1.1 面向对象三大特性是什么？分别解释

**结论**：封装、继承、多态，分别解决"安全访问、代码复用、统一调用"三个问题。

- **封装**：把数据和操作绑定在一起，private 字段 + public getter/setter 隐藏内部实现、控制访问权限。好处：降低耦合、便于加入校验逻辑、内部实现可独立演化。
- **继承**：子类通过 `extends` 复用父类所有非 private 成员；**构造器不被继承**，子类构造器默认在第一行隐式调用 `super()`。Java 只允许单继承（类之间），多份"行为契约"靠实现多个接口补足。
- **多态**：父类引用指向子类对象（向上转型），同一个调用在不同对象上有不同表现。前提：继承 + 重写 + 向上转型；底层机制是 JVM 虚方法表（vtable）的动态绑定——**编译时看左边（声明类型），运行时看右边（实际对象类型）**。

> **面试追问**：字段有多态吗？——没有。字段访问永远按声明类型（左边）解析，只有被重写的方法才走动态绑定。

### 1.1.2 == 和 equals 的区别

**结论**：`==` 比较栈中的值——基本类型比值，引用类型比地址；`equals` 是 `Object` 定义的方法，默认等价于 `==`，String/Integer 等重写后比较内容。

```java
String s1 = new String("hello");
String s2 = new String("hello");
s1 == s2;        // false，两个堆对象地址不同
s1.equals(s2);   // true，内容相同

Integer a = 200, b = 200;
a == b;          // false，超出 Integer 缓存范围(-128~127)，是两个对象
Integer c = 127, d = 127;
c == d;          // true，valueOf 命中缓存，同一个对象
```

**易错点**：

- `equals` 可能为 null 调用方抛 NPE，推荐 `Objects.equals(a, b)`（null 安全）。
- 重写 equals 必须同时重写 hashCode，否则放进哈希集合会出 bug（见下题）。
- 比较日期时间不要依赖 `Date.equals`，用 `compareTo` 或新时间 API。

### 1.1.3 hashCode 与 equals 的契约关系？为什么要一起重写？

**结论**：`equals` 相等的两个对象 `hashCode` 必须相等；`hashCode` 相等不代表 `equals` 相等（哈希碰撞）。重写 equals 就必须重写 hashCode，否则哈希容器定位与判等逻辑脱节。

HashMap 判断 key 相同的源码逻辑：

```java
p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k)))
```

先用 hashCode 把候选缩小到同一个桶，再用 equals 精确确认。**只重写 equals 不重写 hashCode** 时，两个"逻辑相等"的对象落到不同桶：put 时存进两个重复元素、get 时按 hashCode 定位找不到，`HashSet` 去重直接失效。

**设计考量**——写 hashCode 的通用姿势：用参与 equals 比较的同一组字段，`Objects.hash(f1, f2)` 或 31 倍乘子（奇数素数，乘法可用 `(i << 5) - i` 位运算优化且分布均匀）。

### 1.1.4 Object 类有哪些方法？

| 方法 | 作用 | 备注 |
|------|------|------|
| `getClass()` | 返回运行时 Class 对象 | 反射入口，final 方法不可重写 |
| `hashCode()` | 哈希码 | 哈希容器定位桶 |
| `equals(Object)` | 逻辑相等 | 默认比地址，子类通常重写 |
| `toString()` | 字符串表示 | 默认 `类名@十六进制hash` |
| `clone()` | 浅拷贝 | protected，需实现 Cloneable，否则抛 CloneNotSupportedException |
| `wait()` / `wait(long)` / `wait(long, int)` | 进入监视器等待集，释放锁 | 必须在 synchronized 内调用，否则 `IllegalMonitorStateException` |
| `notify()` / `notifyAll()` | 唤醒等待集线程 | 同上；notify 随机唤醒一个，notifyAll 唤醒全部 |
| `finalize()` | GC 回收前回调 | JDK9 起 @Deprecated，时机不确定、性能差、可能导致对象复活 |

> **面试追问**：`getClass()` 为什么是 final？——防止子类篡改类型信息破坏安全机制；`wait/notify` 为什么放在 Object 而不是 Thread？——锁是所有对象都可能持有的监视器锁，等待的是"这把锁"，与线程本身无关。

### 1.1.5 Java 创建对象有哪几种方式？

1. **new 关键字**：调用构造器，最常见。
2. **反射**：`Class.forName("X").getDeclaredConstructor().newInstance()`（JDK9 后 `Class.newInstance()` 已废弃，因为它会吞掉构造器抛出的受检异常）。可指定任意构造器。
3. **clone()**：实现 Cloneable 调 `super.clone()`，**不调构造器**，逐位复制内存。
4. **反序列化**：`ObjectInputStream.readObject()`，从字节流重建，同样不调构造器（父类无 Serializable 时父类构造器会被调用）。
5. **Unsafe.allocateInstance()**：只分配内存不执行构造器，字段全是零值/null，框架和 ORM 底层使用。

**记忆线**：1、2 走构造器；3、4、5 绕过构造器——所以"构造器里做的初始化在 3/4/5 中可能失效"，这也是单例被反射/反序列化破坏的原理（枚举单例天然免疫，见 29_design_patterns.md 对应题）。

### 1.1.6 Java 是值传递还是引用传递？

**结论**：Java **只有值传递**。传对象时，传的是"引用地址的副本"，不是对象本身，更不是引用变量本身。

- 基本类型：数值副本，方法内改不动原变量。
- 引用类型：地址副本，两个引用指向同一对象——
  - 通过副本修改对象内部状态（`user.setName(...)`）→ 影响原对象；
  - 让副本指向新对象（`user = new User()`）→ 原引用不受影响。

```java
void changeName(User user) { user.setName("李四"); }   // 原对象被改
void changeRef(User user)  { user = new User("王五"); } // 原引用不变
```

真正的引用传递（C++ `&`、C# `ref`）能在方法内改变外部变量的指向，Java 做不到。"基本类型值传递、引用类型引用传递"是错误简化。

### 1.1.7 深拷贝和浅拷贝的区别？如何实现深拷贝？

- **浅拷贝**：只复制对象本身的字段值，引用类型字段复制的是地址（两边共享引用对象）。实现：实现 `Cloneable` 后调 `super.clone()`。
- **深拷贝**：整个对象图逐层复制，互不影响。

深拷贝实现方式（按常用程度）：

1. **重写 clone 逐层克隆**：每个引用字段也调用其 clone，样板代码多。
2. **JSON 序列化往返**：Jackson `objectMapper.readValue(objectMapper.writeValueAsString(obj), Xxx.class)`，简单但有类型/性能损耗，private 字段需配置。
3. **JDK 序列化反序列化**：`ByteArrayOutputStream + ObjectOutputStream` → 反向读回；要求全对象图 Serializable，性能最差。
4. **Apache Commons** `SerializationUtils.clone(obj)`。
5. **拷贝构造器 / copyOf 方法**：`new ArrayList<>(src)` 这种显式复制，语义最清楚，推荐在业务代码中手写。

> **注意**：`String`、`Integer` 等不可变类型浅拷共享无害，真正要递归处理的是 Date、集合、可变对象字段。

### 1.1.8 Java 中如何实现不可变类？

不可变类：实例创建后状态不可修改（String、Integer、LocalDate）。实现步骤：

1. 类声明 `final`，防止子类破坏不可变性（或构造器私有 + 静态工厂）。
2. 所有字段 `private final`。
3. 不提供任何 setter，字段值只在构造器确定。
4. **可变引用字段做防御性拷贝**——构造器和 getter 都拷贝，切断外部持有内部可变对象的通道。
5. 不暴露可变内部状态（集合返回 `Collections.unmodifiableList` 或直接拷贝）。

```java
public final class ImmutablePerson {
    private final String name;
    private final Date birthday;  // Date 本身可变

    public ImmutablePerson(String name, Date birthday) {
        this.name = name;
        this.birthday = new Date(birthday.getTime());  // 入参防御性拷贝
    }

    public String getName() { return name; }
    public Date getBirthday() { return new Date(birthday.getTime()); }  // 出参防御性拷贝
}
```

**收益**：天然线程安全、可作为 HashMap key（hashCode 不变）、可安全缓存与共享（字符串常量池就是靠这个成立的）。JDK14 的 `record` 是语言级不可变数据载体（见 4.4.2）。

## 1.2 接口、抽象类与多态

### 1.2.1 抽象类和接口的区别？（JDK8+）

**结论**：抽象类是"is-a"的模板复用，接口是"can-do"的行为契约。都不能实例化，都可含抽象方法。

| 维度 | 抽象类 | 接口 |
|------|--------|------|
| 关键字 | abstract class | interface |
| 继承关系 | 单继承 | 多实现 |
| 构造器 | 有（供子类 super 调用） | 无 |
| 成员变量 | 任意 | 只能 `public static final` 常量 |
| 方法 | 任意（可非抽象） | JDK8 前全抽象；JDK8 起 default/static；JDK9 起 private 方法 |
| 访问修饰 | 任意 | 抽象方法默认 public |
| 状态 | 可有实例状态 | 无实例状态 |

JDK8 引入 default 方法解决"接口演进向后兼容"（Collection 加 stream() 的例子见 4.4.1），JDK9 的 private 方法供多个 default 复用。

> **选型经验**：已知实现类共享代码和状态 → 抽象类；只定义能力、需要多实现或横向组合 → 接口。默认优先接口，需要模板方法模式时再抽抽象类。

### 1.2.2 重载(Overload)和重写(Override)的区别

| 维度 | 重载 | 重写 |
|------|------|------|
| 发生位置 | 同一类（或父子类间同名） | 父子类之间 |
| 方法签名 | 方法名相同，参数列表不同 | 完全相同 |
| 返回值 | 可不同 | 相同或协变（返回子类） |
| 异常 | 任意 | 不能抛出更宽泛的受检异常 |
| 访问权限 | 任意 | 不能比父类更严格 |
| 绑定 | 编译时静态绑定（按声明类型+实参选方法） | 运行时动态绑定（按实际对象类型） |
| static/final | 可重载 | static 不能重写（叫隐藏），final 方法不能重写 |

**口诀**："两同两小一大"——方法名参数表同、返回值和异常更小、权限更大。

> **面试追问**：能否用返回值区分重载？——不能，参数列表相同仅返回值不同编译报错；调用处没有上下文可供编译器选择。

### 1.2.3 Java 为什么不支持类的多继承？接口为什么可以？

**结论**：类多继承的致命问题是**菱形继承**——D 继承 B、C，B 和 C 都重写了 A.foo()，D 调 foo() 语义无法确定；加上多份父类构造链、字段布局冲突，实现复杂度爆炸。

接口能多实现是因为 JDK8 之前接口**没有实现、没有状态**，只是方法签名集合，不存在"冲突的实现选哪个"的问题。JDK8 引入 default 方法后，"类优先 + 必须显式重写冲突方法"的规则就是给这个口子打的补丁（见 4.4.1）。

替代方案：Java 用"**单继承 + 多实现 + JDK8 default + 组合**"模拟多继承的效果，也是合成复用原则（CRP）的体现。

### 1.2.4 JDK 动态代理和 CGLIB 动态代理的区别？

| 维度 | JDK 动态代理 | CGLIB |
|------|-------------|-------|
| 原理 | 实现接口：`Proxy.newProxyInstance` + `InvocationHandler`，运行时生成 `$Proxy0` 继承 Proxy 并实现目标接口 | 继承：`Enhancer` + `MethodInterceptor` 生成目标类的子类并重写方法（ASM 字节码） |
| 要求 | 目标必须实现接口 | 类、方法不能 final |
| 方法调用 | 反射 invoke，恒定开销 | FastClass 方法索引直接调用，执行更快 |
| 生成性能 | 生成快 | 生成慢（字节码工作量大） |
| Spring AOP | 有接口默认使用 | 无接口或 proxy-target-class=true 时使用 |

**设计考量**：Spring Boot 2.x 起 `spring.aop.proxy-target-class` 默认 true，即默认 CGLIB——避免"代理对象强转实现类之外的具体类型失败"这类注入问题。final 类只能走 JDK 代理（且必须有接口）。代理相关深入题与手写示例见 29_design_patterns.md 与 7_spring.md。

### 1.2.5 面向对象设计七大原则（SOLID+）分别是什么？

| 原则 | 核心含义 | 落地例子 |
|------|---------|---------|
| SRP 单一职责 | 一个类只有一个变化的原因 | Controller/Service/DAO 分层 |
| OCP 开闭 | 对扩展开放、对修改关闭 | 策略模式 + 工厂替代 if-else 加分支 |
| LSP 里氏替换 | 子类替换父类不破坏程序正确性 | 正方形悖论：Square extends Rectangle 就是反例 |
| ISP 接口隔离 | 不强迫依赖用不到的方法 | 把 God Interface 拆成精专接口 |
| DIP 依赖倒置 | 高层依赖抽象，不依赖细节 | Spring 面向接口注入 |
| LoD 迪米特 | 只和直接朋友通信，降低耦合 | 门面、消息只传 DTO 不传整个聚合 |
| CRP 合成复用 | 优先组合（黑盒）再继承（白盒） | 组合可运行时替换委托对象，继承是编译期静态绑定 |

SOLID 是前五个首字母缩写。这些原则共同服务于"低耦合、高内聚、易扩展、易维护"，面试中最好结合具体模式讲（策略、模板方法、装饰器对应哪条原则）。

## 1.3 类型、字符串与数字

### 1.3.1 int 和 Integer 的区别？为什么要有包装类？

| 维度 | int | Integer |
|------|-----|---------|
| 类型 | 基本类型 | 引用类型，堆上对象 |
| 默认值 | 0 | null |
| 存储 | 局部变量在栈帧 / 对象内内联 | 引用指向堆对象 |
| 内存 | 4 字节 | 16 字节（对象头 8 + int 4 + 对齐填充 4） |
| 泛型 | 不可用 | `List<Integer>` 可用 |

**为什么要有包装类**：

1. **泛型擦除要求类型参数是 Object 子类**，`List<int>` 不合法——最核心原因。
2. 集合框架存取 Object，基本类型进不了容器。
3. 需要 null 表达"无值/未设置"（数据库可空列、RPC 部分字段更新）。
4. 提供工具方法（parseInt、toHexString、compare）。

**选型建议**：局部变量、循环计数用 int（免拆箱）；PO/VO、集合元素、可空语义用 Integer。注意 `Map<Integer, xxx>` 大量 key 时装箱开销与缓存复用。

### 1.3.2 自动拆装箱的原理、缓存机制和 null 风险？

**原理**：编译器在赋值/运算处自动插入 `Integer.valueOf()`（装箱）和 `intValue()`（拆箱）。

**好处**：代码简洁（`Integer i = 10`）；泛型可用包装类；兼容接收 Object 的旧 API；可表达 null；`valueOf` 命中缓存减少对象分配。

**缓存机制**（`valueOf` 才走缓存，`new` 永远新建）：

| 类型 | 缓存范围 | 备注 |
|------|---------|------|
| Integer | -128 ~ 127 | 上限可调（见 1.3.3） |
| Long / Short | -128 ~ 127 | |
| Byte | 全部值 | |
| Character | 0 ~ 127 | |
| Boolean | true/false | |
| Float / Double | 不缓存 | |

**null 风险与经典坑**：

```java
Integer i = null;
int j = i;              // NPE：拆箱调 i.intValue()

Integer a = 127, b = 127;
a == b;                 // true
Integer x = 100, y = 200, z = x + y;
z == 300;               // 编译错误?不，与 int 比较会拆箱，true；
Integer m = 300;
z == m;                 // false，超缓存是两个对象

List<Integer> list = ...;
int total = 0;
for (Integer v : list) { total += v; }   // 任一元素为 null 即 NPE
```

> **行业实践**：包装类拆箱 NPE 是线上高频故障，来自"数据库可空字段直接赋值 int"。MyBatis 映射字段用包装类、业务计算入口判空或 `Objects.requireNonNull` 快速暴露。

### 1.3.3 Integer 缓存范围可以调整吗？

可以调**上限**：`-XX:AutoBoxCacheMax=<size>` 或 `-Djava.lang.Integer.IntegerCache.high=500`，启动时 `IntegerCache` 静态块据此扩容；**下限固定 -128 不能调低**。本质是享元模式（Flyweight）。

```java
Integer a = 128, b = 128;  // 默认 false
// 加 -XX:AutoBoxCacheMax=1000 重启 → true
```

> **注意**：调整缓存掩盖不了 `==` 比较包装类型的错误姿势，正确做法永远是 `equals` / `Objects.equals`。`Integer.valueOf` 之外，字符串 `String.intern`、`Boolean.valueOf` 也是享元的同族例子。

### 1.3.4 String、StringBuilder、StringBuffer 的区别？为什么 String 不可变？

| 特性 | String | StringBuilder | StringBuffer |
|------|--------|---------------|--------------|
| 可变性 | 不可变 | 可变 | 可变 |
| 线程安全 | 安全（不可变） | 不安全 | 安全（方法 synchronized） |
| 性能 | 拼接产生新对象 | 快 | 中（锁开销） |
| 版本 | JDK1.0 | JDK1.5 | JDK1.0 |
| 底层 | JDK9+ byte[] + coder | byte[]（AbstractStringBuilder） | byte[] |

**String 不可变的实现**：类 final 防子类破坏；`private final byte[] value`（JDK9 前 char[]）；所有"修改"方法返回新对象。

**为什么这样设计**：

1. **字符串常量池复用**的前提——共享对象若可变则互相污染。
2. **线程安全**：任意线程共享无需同步。
3. **hashCode 缓存**：hash 只算一次（final int hash），作为 HashMap key 高效可靠。
4. **安全**：类名、路径、URL 等被 JVM/ClassLoader 使用，不可变防止被篡改。

**设计考量**：单线程大量拼接一律 StringBuilder（能预估容量更佳 `new StringBuilder(len)`）；多线程拼接罕见场景用 StringBuffer，或直接避免共享；JDK9+ 简单 `+` 已被优化（见 1.3.6），短小拼接不必刻意换 StringBuilder。

### 1.3.5 new String("abc") 创建了几个对象？intern 是什么？

**结论**：常量池此前没有 "abc" 时创建 **2 个**——类加载/首次使用时字面量 "abc" 入池一个，`new` 在堆上再一个；池里已有则只创建 1 个。

`intern()`：native 方法，确保字符串在常量池中并返回池中引用。

- JDK6：池在永久代，池中无则**复制一份**堆字符串进池，返回副本引用（堆上对象与池对象不是一个）。
- JDK7+：池移到堆，池中无则把**当前堆对象的引用**直接入池并返回。

经典判断题（JDK7+）：

```java
String s1 = new String("1") + new String("1"); // 堆中 "11"，池中无
s1.intern();        // 把 s1 的引用放进池
String s2 = "11";   // 池中已有，直接复用
System.out.println(s1 == s2);  // true
// 若去掉 intern()，则为 false
```

**记忆**：字面量编译期确定入池；运行期拼接结果在堆上、不在池中；intern 是"手动入池"。

### 1.3.6 字符串 + 拼接的底层原理？

- **全常量**：编译期折叠，`"a" + "b"` 直接变成常量池 `"ab"`。
- **变量拼接（JDK8 及以前）**：编译为 `new StringBuilder().append(a).append(b).toString()`。
- **循环拼接**：每轮迭代 new 一个 StringBuilder，产生大量临时对象——必须在循环外建一个 StringBuilder 复用。
- **JDK9+**：改为 `invokedynamic` 调用 `StringConcatFactory`，运行期选择最优策略（MethodHandle 组合、数组直拷），字节码更小、性能更好。
- `String.concat()`：新建数组一次拷贝，恰好两段拼接可用；`+=` 在 final/常量场景等价折叠。

### 1.3.7 BigDecimal 的精度问题？equals 和 compareTo 的区别？

**精度问题**：二进制无法精确表示 0.1 这类十进制小数，`new BigDecimal(0.1)` 得到 `0.1000000000000000055511151231257827...`。正确姿势：**字符串构造** `new BigDecimal("0.1")` 或 `BigDecimal.valueOf(0.1)`（内部走 Double.toString）。

**equals vs compareTo**：

- `equals` 比较 **数值 + 精度(scale)**：`new BigDecimal("1.0").equals(new BigDecimal("1.00"))` → false。
- `compareTo` 只比数值：上例 `compareTo` → 0。

> **行业实践**：金额一律 BigDecimal 存储计算，除法显式指定 `setScale(2, RoundingMode.HALF_UP)`；比较一律用 compareTo；需要"1.0 和 1.00 当同一键"时别用 equals/hashCode（或先统一 scale）。数据库用 DECIMAL 而非 FLOAT/DOUBLE。

### 1.3.8 类型转换和类型提升的规则？

- **隐式宽化**：byte→short→int→long→float→double，char→int，小类型自动变大类型无损。
- **强制窄化**：`(byte) 200` → -56（溢出回绕）、`(int) 3.9` → 3（截断）。
- **类型提升**：byte/short/char 参与算术运算一律提升为 int——`byte a=1,b=2; byte c=a+b;` 编译错误，需强转；表达式含 long/float/double 时提升到最高等级。
- **复合赋值隐含强转**：`b += 1` 等价 `b = (byte)(b + 1)` 合法，`b = b + 1` 非法。
- **整数溢出**：int 最大约 21.4 亿，超界回绕变负。防御：`Math.addExact/multiplyExact`（溢出抛异常）或提前升级 long。

### 1.3.9 switch 支持哪些类型？底层原理？

**支持**：byte、short、char、int 及对应包装类（拆箱后比较）、枚举（JDK5）、String（JDK7）。**不支持** long、float、double、boolean。

底层字节码：

- `tableswitch`：case 值密集连续 → 跳转表，O(1)。
- `lookupswitch`：case 稀疏 → 有序键值对二分查找。
- **String switch**：先按 `hashCode` 做跳转表筛选，命中后再 `equals` 二次确认（哈希碰撞兜底）。
- **枚举 switch**：编译生成 ordinal → case 的映射数组，本质按序号 int 跳转。JDK17 的 switch 模式匹配不再依赖该数组。

### 1.3.10 位运算符有哪些？常见应用场景？

七个：`&` 与、`|` 或、`^` 异或、`~` 取反、`<<` 左移、`>>` 算术右移（符号位扩展）、`>>>` 无符号右移（高位补 0）。

常见应用：

- `n & 1` 判奇偶；`n << 1` 乘 2、`n >> 1` 除 2（向零取整）。
- `a ^ b ^ b == a`：无临时变量交换（仅对整数，面试可提可读性差生产不推荐）。
- HashMap 扰动函数 `(h ^ (h >>> 16))`、`(n-1) & hash` 代替取模。
- 权限位掩码：`READ=1<<0, WRITE=1<<1`，`perm & READ` 判权限；`|` 加权限、`& ~` 去权限。
- JDK8 `Long.bitCount`、`Integer.numberOfLeadingZeros` 对应 POPCNT/LZCNT 指令。

> **注意**：移位结果至少是 int；`int` 移位数取 `mod 32`、`long` 取 `mod 64`。`>>` 对负数高位补 1，`>>>` 无符号右移可能把负数变成巨大正数。


## 1.4 关键字与语法

### 1.4.1 final、finally、finalize 的区别？

- **final**（修饰符）：修饰类不可继承（String、`Integer` 等）；修饰方法不可重写（模板方法里锁定骨架）；修饰变量值不可变——基本类型是数值不变，引用类型是**地址不变但对象内容可变**（`final List` 仍可 add）；修饰参数方法内不可重新赋值。
- **finally**（异常处理）：try-catch-finally 中无论如何都执行的块，用于释放资源（`System.exit()`、线程被 kill 等除外）。
- **finalize**（Object 方法）：GC 回收前最多调用一次，JDK9 废弃——执行时机不确定、拖慢回收、可复活对象造成幽灵引用；替代是 `Cleaner`/try-with-resources。

**易错**：`finally` 中不要 return/抛异常，会覆盖 try/catch 的返回值与异常；final 字段可在构造器内赋值一次（空白 final），配合 `private final` 实现不可变类。

### 1.4.2 static 关键字的用法和作用？

- **静态变量**：属于类、所有实例共享一份，类加载的准备阶段赋零值、初始化阶段赋程序指定值；`static final` 编译期常量在准备阶段直接定值。
- **静态方法**：无 this，只能访问静态成员；不能被重写（子类同签名方法叫**隐藏**，调用由声明类型决定）；常见用途是工具方法与工厂方法（`Integer.valueOf`、`List.of`）。
- **静态代码块**：类初始化时按书写顺序执行且只执行一次，用于复杂静态初始化。
- **静态内部类**：不持有外部类引用（对比成员内部类，见 1.8.2）。
- **静态导入**：`import static` 直接引用常量/方法（`import static org.junit.Assert.*`）。

**执行顺序**（单类）：静态变量/静态块（按书写顺序）→ 实例变量/实例块 → 构造器。多继承场景见 1.10.1。

### 1.4.3 this 和 super 的区别与用法？

| 关键字 | 含义 | 典型用法 |
|--------|------|---------|
| `this` | 当前对象引用（编译后是隐藏的第一个参数） | `this.x` 区分成员与局部变量；`this()` 调用本类重载构造器；把 this 传给回调/方法（注意引用逸出） |
| `super` | 直接父类的成员入口 | `super.x` 访问被遮蔽的父类字段；`super.method()` 调被重写的方法；`super()` 调父类构造器 |

关键规则：

- `this()` / `super()` 都必须在构造器**第一行**，二者只能出现其一；不写则编译器隐式插入无参 `super()`。
- 父类没有无参构造器时，子类必须显式 `super(参数)`，否则编译错误。
- static 方法中没有 this/super。
- `super` 不能跨代（没有 `super.super`）；`this` 与 super 访问**字段按声明类型静态解析**，方法才走动态绑定。

### 1.4.4 什么是可变长参数（Varargs）？

用 `类型... 名字` 声明，允许传 0 到 N 个同类型参数。**编译后就是数组**：`sum(int... nums)` 等价 `sum(int[] nums)`，调用处编译器自动包装数组。

```java
public static int sum(int... nums) {
    int total = 0;
    for (int n : nums) total += n;
    return total;
}
sum(1, 2, 3);  // 编译为 sum(new int[]{1,2,3})
sum();         // 空数组，不是 null
```

**规则与坑**：

- 可变参数必须是**最后一个**参数，一个方法只能有一个。
- 传 `int[]` 数组实参与逐个元素等价；但 `new int[]{1,2,3}` 传给 `Object...` 会被当成**一个元素**（数组是 Object）。
- 与重载共存时：编译器优先匹配"无需装箱/可变参数"的方法，精确匹配 > 装箱匹配 > 可变参数匹配。
- 方法内可以改数组内容，但每次调用都会新建数组，高频路径注意开销。

### 1.4.5 Java 中变量和常量有什么区别？

| 维度 | 变量 | 常量 |
|------|------|------|
| 修饰符 | 无 | final（通常配 static） |
| 值 | 运行时可变 | 初始化后不可变 |
| 命名 | 小驼峰 | 全大写 + 下划线 |
| 内存 | 运行时分配 | 编译期常量：值内联进调用方常量池，分配一次 |
| 用途 | 存储可变状态 | 固定阈值、协议码、共享不可变值 |

**注意**：引用类型常量只是地址不可变（`final List L` 内容仍可改）；`static final` 基本类型/字符串字面量是编译期常量，被**内联**，修改后要重新编译所有依赖方（这也是"对外 API 别用编译期常量暴露可配置值"的原因）；空白 final 允许构造器内一次赋值。

## 1.5 异常体系

### 1.5.1 Java 异常体系是怎样的？checked 和 unchecked 区别？

`Throwable` 为根，两大分支：

- **Error**：JVM 层面的严重错误——`OutOfMemoryError`、`StackOverflowError`、`NoClassDefFoundError`。程序不应捕获，通常也无力恢复。
- **Exception**：
  - **checked（编译时异常）**：Exception 下除 RuntimeException 的部分——IOException、SQLException、ClassNotFoundException。编译器**强制**处理（catch 或 throws），用于"外部环境可恢复故障"。
  - **unchecked（运行时异常）**：RuntimeException 子类——NPE、IndexOutOfBounds、ClassCastException、IllegalArgumentException。多为**程序 bug**，编译期不强制。

**try-catch-finally 执行细节**：finally 必定执行（除非 `System.exit`）；try 中 return 时返回值**先暂存**再执行 finally；finally 里再 return 会**覆盖** try/catch 的返回值和异常——严禁。

> **设计争议**：受检异常强迫调用方正视失败，但层层 throws 污染签名，Spring 因此把 JDBC/IO 包装成 DataAccessException、UncheckedIOException 这类非受检体系。新框架倾向"要么恢复要么崩溃"，业务代码大量使用 unchecked。

### 1.5.2 throw 和 throws 的区别？

- **throw**：方法体内的语句，抛出一个异常实例，后面的代码不可达（编译器据此做流程分析）。
- **throws**：方法签名上的声明，告知调用方"可能抛这些受检异常"，是契约不是动作。
- 重写方法时 `throws` 的受检异常范围不能比父类更宽（与"两小一大"呼应）；构造器上也可写 throws。
- 泛型异常方法 `<T extends Throwable> void m(Class<T> type) throws T` 可实现"精确重抛"，包装异常而不丢编译期信息（Lombok `@SneakyThrows` 则是绕过检查的字节码黑科技，慎用）。

### 1.5.3 try-with-resources 原理是什么？

JDK7 引入：实现了 `AutoCloseable` 的资源写在 try 括号里，**代码块结束（无论正常或异常）自动调用 close()**，且 close 顺序与声明顺序**相反**。

```java
try (BufferedReader r1 = open(a); BufferedReader r2 = open(b)) {
    ...
}
// 编译后近似：
// finally { if (r2 != null) r2.close(); if (r1 != null) r1.close(); }
```

**关键细节**：当 try 体和 close 都抛异常时，**try 体的主异常被抛出**，close 的异常通过 `getSuppressed()` 附带（suppressed 机制），不会像传统 finally-close 那样吞掉根因——这正是它优于手写 finally 的地方。JDK9 进一步允许括号里直接写" effectively final 的外部变量"。

### 1.5.4 什么是异常链？异常处理有哪些最佳实践？

**异常链**：捕获底层异常后抛新异常时，把原异常作为 cause 挂上，保留根因堆栈：

```java
try {
    queryDao();
} catch (SQLException e) {
    throw new BizException("订单查询失败", e); // e 成为 cause
}
// 外层 getCause() 可追溯；printStackTrace 会打印 "Caused by:"
```

**最佳实践**：

1. catch 具体异常，别 `catch (Exception)` 一把梭；多 catch **先小后大**（子类在前），否则编译错误。
2. 业务边界做**异常翻译**：技术异常（SQLException）转统一业务异常（配错误码），对上层暴露稳定契约。
3. 不要用异常控制正常流程（性能差、语义乱）；可预期的"没找到"用 Optional/空集合表达。
4. finally 不 return 不抛新异常；资源优先 try-with-resources。
5. 日志打印异常要带堆栈（`log.error("msg", e)`），不要只打 `e.getMessage()`；抛出前不重复打印（避免同异常满屏日志）。
6. 全局兜底：`@RestControllerAdvice` / Thread.UncaughtExceptionHandler / `Thread.setUncaughtExceptionHandler`。

## 1.6 泛型

### 1.6.1 泛型的类型擦除是什么？PECS 原则是什么？

**类型擦除**：Java 泛型是编译期机制——编译时检查类型，然后把类型参数擦除替换为**上限类型**（无界 `<T>` 擦成 Object，`<T extends Number>` 擦成 Number），并在调用处插入强制转换。因此：

- `List<String>.class == List<Integer>.class`（同一个 ArrayList 的 Class 对象）；
- 不能 `new T()`、不能 `T[] arr = new T[10]`、不能 `instanceof List<String>`；
- 桥方法因此产生（见 1.6.2）；重载 `f(List<String>)` 与 `f(List<Integer>)` 冲突。

保留信息的地方：**方法签名和字段声明上的泛型信息保留在字节码的 Signature 属性中**，Jackson/Gson 反序列化"嵌套泛型"就是靠反射读 Signature 拿到的。

**PECS 原则**（Producer Extends, Consumer Super）：

- `? extends T`：上界通配符，集合是**生产者**（你从里面读 T）——可读不可写（写会破坏具体子类型约束，add 只允许 null）。`Collections.copy(dest, src)` 的 src 就是 `List<? extends T>`。
- `? super T`：下界通配符，集合是**消费者**（你往里写 T）——可写，读出来只能当 Object。`Comparator<? super T>` 让 `Comparator<Object>` 也能给 `List<String>` 排序。
- 既读又写就不要用通配符，直接用 `<T>`。

### 1.6.2 什么是桥方法（Bridge Method）？

类型擦除后，编译器为维持多态**自动生成的合成方法**。

```java
class MyCmp implements Comparator<String> {
    public int compare(String a, String b) { ... }
}
// 编译器额外生成：
public /*synthetic*/ int compare(Object a, Object b) {
    return compare((String) a, (String) b); // 强转后调用真实方法
}
```

擦除后接口方法签名是 `compare(Object, Object)`，若不生成桥方法，`MyCmp` 就"没实现接口"、也无法通过 Comparator 引用多态调用。

**影响**：`getDeclaredMethods()` 能看到这些 ACC_BRIDGE/ACC_SYNTHETIC 方法，直接反射调用可能抛 ClassCastException；协变返回类型重写（父类返回 Number、子类返回 Integer）同样靠桥方法衔接。判断依据：`method.isBridge()`。

## 1.7 反射、注解与 SPI

### 1.7.1 什么是反射？有哪些应用场景和代价？

**结论**：运行时动态获取类型信息（构造器/方法/字段/注解）并操作对象的能力，是"框架面向配置编程"的根基。

获取 Class 的四种方式：`X.class`（编译期，最安全）、`obj.getClass()`（运行时实际类型）、`Class.forName("全限定名")`（会触发类初始化，JDBC 注册驱动靠它）、类加载器 `loadClass`（只加载不初始化）。

常用 API：`getDeclaredMethods`/`getMethods`（Declared 含私有不含继承，无后缀版只含 public 含继承）、`getDeclaredField` + `setAccessible(true)` 暴力反射、`getDeclaredConstructor(...).newInstance(...)`、`isAnnotationPresent`/`getAnnotation`。

**应用场景**：JDK 动态代理、Spring IoC 实例化与注入、JDBC 驱动加载、Jackson/Gson 序列化、JUnit 执行测试方法、配置文件指定类名的插件加载、Arthas 诊断。

**代价**：调用比直接慢几十倍（权限检查、装箱、无法被 JIT 完全内联）；破坏封装与编译期类型安全；`setAccessible(true)` 在 JDK9 模块化后会被模块封装拦截。现代缓解：`MethodHandle`、`LambdaMetafactory`（Spring 5 的 `ReflectionUtils` 大量改用 MethodHandle）、缓存 `Class`/`Method` 对象。

### 1.7.2 注解的元注解有哪些？如何自定义注解？

**五大元注解**：

| 元注解 | 作用 |
|--------|------|
| @Target | 可标注位置：TYPE/FIELD/METHOD/PARAMETER/CONSTRUCTOR/… |
| @Retention | 保留策略：SOURCE（编译丢弃，Lombok 处理器用）、CLASS（进 class 不加载，默认）、RUNTIME（反射可读，Spring 注解全用这个） |
| @Documented | 进 javadoc |
| @Inherited | 子类继承父类的注解（只对类上的注解生效，方法/字段不继承） |
| @Repeatable(JDK8) | 同位置可重复标注（配容器注解） |

**自定义注解三件套**：注解定义 + 处理器（AOP/拦截器/BeanPostProcessor/编译期 Processor）+ 触发时机。

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Desensitization {
    Type value() default Type.PHONE;   // 脱敏类型
    String[] groups() default {};      // 数组是注解属性的标配形态
}
```

属性只能放：基本类型、String、Class、枚举、注解及其数组；**无成员注解是标记注解**（如 @Override）。RUNTIME 注解靠反射在运行时被消费，CLASS 注解靠字节码工具（ASM）消费，SOURCE 注解靠注解处理器（APT）在编译期生成代码——三种保留期对应三种消费方式，这是理解 Lombok、ARouter、MapStruct 的钥匙。

### 1.7.3 Java SPI 机制是什么？与 Spring Boot 自动装配的区别？

**SPI（Service Provider Interface）**：JDK 内置的"接口 + 外部实现发现"机制，面向"由第三方提供实现"的扩展场景。约定：jar 包内 `META-INF/services/<接口全限定名>` 文件，内容为实现类全限定名；`ServiceLoader.load(Driver.class)` 返回懒加载迭代器，逐个实例化。

**典型应用**：JDBC 驱动（`java.sql.Driver`，这就是为什么新版不用手动 Class.forName）、SLF4J 绑定、Dubbo 扩展点（自研增强版 SPI，按名取扩展+自适应）。

**与 Spring Boot 自动装配对比**：

| 维度 | JDK SPI | Spring Boot 自动装配 |
|------|---------|---------------------|
| 加载方式 | 一次性实例化所有实现 | 按条件装配（@ConditionalOnClass/MissingBean 等） |
| 按需取实现 | 不支持，想用哪个迭代哪个 | @Autowired 按类型/名称注入 |
| 与 IoC 集成 | 无 | 深度集成，Bean 生命周期完整 |
| 配置文件 | META-INF/services | 旧版 spring.factories → 2.7+ 新位置 `AutoConfiguration.imports` |

> **注意**：SPI 的"加载所有实现"意味着任一实现类抛异常整个加载失败，且实例由 ServiceLoader 自己 new，无法享受依赖注入——这是 Spring 另起炉灶的根本原因。

## 1.8 内部类

### 1.8.1 内部类有哪几种？分别有什么特点？

| 类型 | 定义位置 | 能否访问外部实例成员 | 持有外部引用 | 典型用途 |
|------|---------|--------------------|--------------|---------|
| 成员内部类 | 类内非 static | 能 | 是（隐式 Outer.this） | 迭代器、成对逻辑 |
| 静态内部类 | 类内 static | 只能静态成员 | 否 | Builder、HashMap.Node、Holder 单例 |
| 局部内部类 | 方法内 | 能 + final 局部变量 | 是 | 方法内临时类（几乎被 Lambda 取代） |
| 匿名内部类 | new 表达式 | 能 + effectively final 局部变量 | 是 | 一次性回调（Listener、Comparator） |

编译产物：`Outer$Inner.class`、`Outer$1.class`（匿名按序号）。成员/匿名内部类都**隐式持有外部类实例引用**——所以 Android 里非静态内部类持 Activity 是经典泄漏源，Handler 一律写成 static 内部类 + WeakReference。

### 1.8.2 匿名内部类访问局部变量为什么必须 final / effectively final？

**根因是拷贝语义 + 生命周期错位**：局部变量活在栈帧里，方法返回即销毁；匿名内部类对象却在堆上长期存活（被线程池、回调持有）。Java 的做法是**把局部变量的值拷贝一份**存进内部类的合成字段——如果允许两边分别修改，语义必然分裂。因此强制变量不可变：JDK7 前要求显式 `final`，JDK8 放宽为 effectively final（实际没再赋值即可，声明处不用写 final）。

对比：访问**成员变量**不需要 final——内部类持有 `Outer.this` 引用，读写的都是同一块堆内存，天然一致。Lambda 捕获变量同理，也是"值拷贝"而非引用共享（见 4.1.1 的 this 差异）。

### 1.8.3 成员内部类为什么能访问外部类的 private 成员？会不会造成内存泄漏？

编译期在外部类中生成 **access$xxx 静态合成方法**（JDK11 后改用 NestMembers 属性，nest 内直接访问、少生成桥方法），内部类通过它读私有字段——封装在字节码层面被"同 nest 豁免"了。

外部引用问题：成员内部类构造器实际是 `Inner(Outer o)`，持有外部实例。若内部类实例被静态集合/长任务持有，外部类整体都无法回收，构成**间接泄漏**；静态内部类无此引用。实践上：不需要外部实例就一律 static；Lambda 不产生 this 引用，比匿名内部类更安全。

## 1.9 序列化

### 1.9.1 Serializable 和 Externalizable 区别？transient 作用？serialVersionUID 呢？

- **Serializable**：纯标记接口，序列化由 JVM 默认机制完成——沿类层级写入所有**非 transient、非 static** 字段（含 private，反射直写，不走 getter/setter）。
- **Externalizable**：继承 Serializable，必须手动实现 `writeExternal()/readExternal()`，完全控制写入内容（可脱敏、可压缩）；代价是**反序列化会先调用该类的无参构造器**（跳过 Serializable 的"不执行构造"规则），字段初始化逻辑可能意外生效。
- **transient**：字段不参与默认序列化，反序列化后取类型默认值（null/0/false）。用途：密码等敏感字段、可重算的缓存字段、不可序列化的引用（Thread、Socket）。**静态字段不参与序列化**（属于类不属于对象），transient 对 static 无意义。
- **serialVersionUID**：类版本指纹。反序列化时校验，不匹配抛 `InvalidClassException`。不显式声明则由 JVM 按类结构自动生成——加个字段都可能导致旧数据全部读不出来。> **行业实践**：实现 Serializable 必手写 `private static final long serialVersionUID = 1L;`。

补充两个高频细节：序列化对对象图是**引用共享**的（同一对象多次出现，反序列化后仍是同一实例，靠 handle 表）；父类不可序列化时，父类字段由"沿继承链第一个非 Serializable 类的无参构造器"重建，值不是保存时那份。Java 原生序列化因体积/性能/安全（反序列化 RCE 漏洞链）问题，分布式场景应换 JSON/Protobuf/Kryo（见 5.4.1）。

## 1.10 类加载与初始化

### 1.10.1 类的初始化时机和执行顺序是什么？

类加载五阶段：**加载 → 验证 → 准备 → 解析 → 初始化**（加载、验证、解析统称连接）。准备阶段给 static 变量分配内存并赋**零值**（`static final` 编译期常量直接赋确定值）；初始化阶段执行 `<clinit>`（静态变量赋值 + 静态块，按书写顺序）。

**主动引用触发初始化（6 种）**：new、读写静态变量（编译期常量除外）、调用静态方法、反射 Class.forName、初始化子类先触发父类、含 main 的启动类。

**被动引用不触发（3 种）**：

```java
Sub.value          // 读父类静态字段：只初始化 Super，不初始化 Sub
Sub[] arr = new Sub[10];  // 定义数组：只初始化数组的类，不初始化 Sub
Super.CONST        // 引用 static final 编译期常量：值已内联，谁都不初始化
```

**完整初始化顺序**：

```text
父类静态变量/静态块 → 子类静态变量/静态块
→ 父类实例变量/实例块 → 父类构造器
→ 子类实例变量/实例块 → 子类构造器
```

口诀：静态先于实例、父先于子、按书写顺序、构造器最后；静态只执行一次，实例部分每次 new 都走。

`Class.forName` 执行初始化（JDBC 驱动靠 static 块自注册），`ClassLoader.loadClass` 只加载不初始化（Spring 懒加载、CGLIB 延迟增强利用此差异）。委托模型细节见 2_jvm.md。

## 1.11 工程与场景题

### 1.11.1 什么是数据脱敏？如何实现？

**结论**：对手机号、身份证、银行卡等敏感数据做变形展示/存储，使未授权者无法识别原文。常见规则：手机号 `138****5678`（留前3后4）、身份证留前6后4、银行卡只留后4、姓名留姓、邮箱 `j***@example.com`。

实现路径（按侵入性从低到高）：

1. **序列化层脱敏（最常用）**：自定义 Jackson 注解 + 序列化器，字段出参时替换——

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@JacksonAnnotationsInside
@JsonSerialize(using = DesensitizationSerializer.class)
public @interface Desensitization {
    DesensitizationType value();  // PHONE / ID_CARD / NAME...
}
// 序列化器内部按类型掩码处理，常用 Hutool StrUtil.hide(phone, 3, 7)
```

2. **AOP/拦截器统一处理**：返回 DTO 前反射扫描注解字段。
3. **数据库层**：查询函数直接脱敏（用于导数、报表）；存储层用加密代替脱敏。
4. **日志脱敏**：logback 自定义 Converter 或 MDC 过滤，防止敏感信息进日志文件。

> **注意**：脱敏发生在"展示/导出"，**存储要加密**（AES+KMS），两者别混为一谈；需要还原原文的场景（客服核身）用"部分脱敏+权限控制查看"，不要返回可逆形式。

### 1.11.2 SPU 和 SKU 的区别与联系？

- **SPU**（Standard Product Unit，标准产品单元）：描述"这是什么商品"——品牌、名称、类目、主图、详情，与库存无关。iPhone 15 是一个 SPU。
- **SKU**（Stock Keeping Unit，库存量单位）：描述"具体哪个规格"——颜色、容量、价格、库存、条码。iPhone 15 黑色 256G 是一个 SKU。

一个 SPU 对应多个 SKU（1:N）。数据库建模两张表：`spu`（通用信息）+ `sku`（规格、价格、库存，外键指向 spu_id）。电商域的高频追问："多规格属性怎么建表"（`spu_property`、`property_value` 与 SKU 规格组合的关系）见 16_mysql.md 设计题。

### 1.11.3 Objects、System.arraycopy 等常用工具方法？

- `Objects.requireNonNull(obj, "msg")`：入参校验，null 抛带消息 NPE（JDK8 起广泛用作构造器防御）。
- `Objects.equals(a, b)` / `isNull` / `nonNull`：null 安全比较，替代手写判空。
- `Objects.hash(...)` / `Objects.hashCode(obj)`：生成/计算 hashCode，null 记 0。
- `System.arraycopy(src, sPos, dest, dPos, len)`：native 数组复制（浅拷贝），目标数组要自己创建；ArrayList 扩容、Arrays.copyOf 底层都是它。
- `Arrays.copyOf(arr, newLen)`：新建目标数组 + arraycopy，返回新数组。
- `Math.addExact / subtractExact / multiplyExact`：溢出检测，溢出抛 ArithmeticException（对 int 溢出静默回绕的场景做保险）。

### 1.11.4 Arrays.asList 有哪些坑？

1. 返回的是 **`Arrays$ArrayList`**（Arrays 的私有静态内部类），不是 `java.util.ArrayList`——只支持读取视图，**add/remove 抛 UnsupportedOperationException**。
2. **与原数组双向联动**：它直接包装原数组，`list.set(0, x)` 会改原数组，反之亦然。
3. **基本类型数组陷阱**：`Arrays.asList(new int[]{1,2,3})` 把 int[] 当成一个元素，得到 `List<int[]> size=1`；正确写法 `Arrays.asList(1, 2, 3)`（装箱）或 `IntStream.of(arr).boxed().toList()`。
4. 要可变副本就再包一层：`new ArrayList<>(Arrays.asList(...))`；JDK9+ 只要只读就用 `List.of(...)`（真不可变、禁 null，见 2.1.3）。


# 2. 集合框架

## 2.1 体系总览与通用问题

### 2.1.1 Java 集合体系结构是怎样的？

**结论**：两大根——`Collection`（单列，继承 Iterable）与 `Map`（双列，独立体系不继承 Collection）。

```text
Collection ─┬─ List：ArrayList / LinkedList / Vector(过时) / CopyOnWriteArrayList
            ├─ Set：HashSet / LinkedHashSet / TreeSet / CopyOnWriteArraySet
            └─ Queue/Deque：ArrayDeque / PriorityQueue / LinkedList / BlockingQueue 家族
Map ─┬─ HashMap / LinkedHashMap / Hashtable(过时)
     └─ TreeMap / ConcurrentSkipListMap / ConcurrentHashMap
```

- List：有序可重复；Set：不可重复（依赖 hashCode/equals 或比较器）；Queue：FIFO/优先级/阻塞语义；Map：键不可重复。
- Arrays 工具类操作数组、Collections 工具类操作集合（排序、同步包装、不可变视图）。
- fail-fast 迭代（ArrayList/HashMap）与 fail-safe 弱一致迭代（CopyOnWrite、ConcurrentHashMap）是贯穿体系的第二条线。

> **易错**：`Map` 没有 `size()` 以外的 Iterable 入口，遍历 key/value 用 entrySet() 或 JDK8 的 `forEach((k,v)->...)`；`Dictionary` 等老类一律不用。

### 2.1.2 Iterator 和 ListIterator 有什么区别？

| 维度 | Iterator | ListIterator |
|------|----------|--------------|
| 适用范围 | 所有 Collection（List/Set/Queue） | 仅 List（及 Deque 的 descendingIterator 另有渠道） |
| 遍历方向 | 单向 | 双向（hasPrevious/previous） |
| 修改能力 | 只有 remove() | add()/set()/remove() |
| 位置信息 | 无 | nextIndex()/previousIndex() |

要点：`Iterator.remove()` 删的是**上一次 next() 返回的元素**，并同步 expectedModCount，所以不会触发 fail-fast——这是"迭代中删除"的正确姿势之一；ListIterator 的 add 插入在游标之前。增强 for 循环底层就是 Iterator（数组被编译成索引循环），编译成 Iterator 后忘调 remove 仍会 fail-fast。

### 2.1.3 Collections.unmodifiableList 和 JDK9 的 List.of 有何区别？

- **unmodifiableList**：**装饰器视图**——内部 final 字段持有原集合，读方法委托、写方法抛 UnsupportedOperationException。但它是"只读壳"：原集合被别的引用修改，视图跟着变，且拿它当"防御性拷贝"是错误用法（对方仍可改原集合）。
- **List.of()/Set.of()/Map.of()（JDK9+）**：**真不可变集合**——自身独立存储、不允许 null、结构封闭；序列化返回的是内部 `Collections$UnmodifiableRandomAccessList` 之类，任何修改方法直接抛异常，性能上查找实现也更省。

**选型**：对外暴露常量/内部状态一律 `List.of`（JDK8 及以下用 `Collections.unmodifiableList(new ArrayList<>(copy))`，注意先拷贝再包视图）。

## 2.2 List 家族

### 2.2.1 ArrayList 和 LinkedList 的区别与适用场景？

| 维度 | ArrayList | LinkedList |
|------|-----------|-----------|
| 底层 | Object[] 数组 | 双向链表（Node: prev/item/next） |
| 随机访问 get(i) | O(1) | O(n)（先决定从头/尾哪端遍历） |
| 尾部 add | 均摊 O(1)（偶发扩容拷贝） | O(1) |
| 头部/中间插删 | O(n)（元素整体搬移） | 定位 O(n) + 改指针 O(1) |
| 内存 | 连续数组 + 预留容量 | 每节点额外两个指针（24+ 字节） |
| 接口 | List + RandomAccess | List + Deque |
| CPU 友好性 | 好（缓存连续） | 差（指针跳跃） |

**场景**：默认选 ArrayList。频繁**头尾**增删、要队列/栈语义时，用 **ArrayDeque**（比 LinkedList 更快、内存更省，不支持 get 索引）而不是 LinkedList——实际工程中几乎找不到"必须用 LinkedList"的场景，连 JDK 自己的 ArrayDeque 都更优。`subList` 是视图不是拷贝，`ArrayList.clone` 是浅拷贝。

### 2.2.2 ArrayList 的扩容机制是怎样的？

无参构造时 `elementData` 先指向空数组 `DEFAULTCAPACITY_EMPTY_ELEMENTDATA`（用来区分"默认构造"与"给了容量 0"）：

1. 首次 add 扩容到 `DEFAULT_CAPACITY = 10`；
2. 之后每次 `newCapacity = oldCapacity + (oldCapacity >> 1)`，即 **1.5 倍**；
3. 若 1.5 倍仍小于需求，取需求容量；超过 `MAX_ARRAY_SIZE(Integer.MAX_VALUE-8)` 走 hugeCapacity；
4. `Arrays.copyOf` 建新数组 + `System.arraycopy` 搬元素。

**设计考量**：已知数据量时 `new ArrayList<>(expectedSize)` 预分配，避免反复"扩容+拷贝"。扩容成本在 add 全部完成时是 O(n) 均摊，但单次触发扩容的 add 是 O(n)。

### 2.2.3 fail-fast 机制是什么？迭代中删除为什么会抛异常？正确做法？

集合维护 `modCount`（结构修改计数），迭代器创建时快照 `expectedModCount = modCount`；`next()` 前检查两者不等即抛 `ConcurrentModificationException`（fail-fast 意为"尽早暴露"，不是"保证检出"——单线程也可能漏报，它不是并发安全机制）。

单线程里 `for (String s : list) { list.remove(s); }` 同样触发：直接调集合的 remove 只改 modCount、不改迭代器快照。

正确做法：

1. **迭代器自身 remove()**（同步 expectedModCount）；
2. JDK8 `collection.removeIf(predicate)`（内部走迭代器逻辑）；
3. 并发场景用 **CopyOnWriteArrayList**（快照迭代，永不抛 CME）。

### 2.2.4 CopyOnWriteArrayList 的原理和适用场景？

**原理**：写时复制。数据存于 `volatile Object[] array`；读（get/size/contains）完全无锁直读当前数组；写（add/set/remove）先拿 ReentrantLock，**复制整个数组**→在副本上修改→volatile 替换引用→释放锁。

- 优点：读线程零阻塞、零锁竞争；迭代器基于创建时快照，绝不抛 CME。
- 缺点：写代价 O(n) 全量拷贝 + 写期间内存双份；读到的可能不是最新（**最终一致**，非强一致）。
- 适用：**读多写少且数据量不大**——监听器列表、黑白名单配置、字典表缓存。写频繁或数据大时内存/GC 压力会失控。
- 迭代器弱一致：快照读，看不到迭代期间的写入。

### 2.2.5 为什么 Vector 过时了？它和 CopyOnWriteArrayList 有什么区别？

Vector 是 JDK1.0 遗留：所有方法 `synchronized` 全表粗锁，复合操作（先 contains 再 add）仍需外部加锁，性能差；`Stack` 同样基于它。替代品：一般场景 `Collections.synchronizedList`（同样全锁但更可控）、高并发读多写少 `CopyOnWriteArrayList`、需要队列语义 `ConcurrentLinkedQueue/BlockingQueue`。`Hashtable` 同理被 ConcurrentHashMap 取代（对比见 2.4.1）。

> **易错**：`Collections.synchronizedList` 返回的集合做**迭代**时仍要手动 `synchronized (list) {}` 包住，否则并发修改照样 CME——包装类只保证单个方法原子。

## 2.3 HashMap 家族

### 2.3.1 HashMap 的底层结构？put 的执行流程？

**结构（JDK8）**：数组 + 链表 + 红黑树。链表长度 ≥8 **且数组长度 ≥64** 才树化（数组小则优先扩容），链表 ≤6 退化回树；默认容量 16、负载因子 0.75、阈值 = 容量 × 0.75。

**put 流程**：

1. 计算 hash：`(key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16)`（扰动，让高 16 位参与低位运算，降低碰撞）；
2. 数组为空/未初始化 → `resize()` 初始化（默认构造首次 put 才给 16）；
3. `(n - 1) & hash` 定位桶；
4. 桶空 → 直接 new Node 放入；
5. 桶非空 → 首节点 key 相同（`hash 相等 && (== 或 equals)`）则替换；是 TreeNode → `putTreeVal`；否则遍历链表，找到相同 key 替换，找不到**尾插**；插入后长度 ≥8 调 `treeifyBin` 尝试树化；
6. `modCount++`、`size++`，`size > threshold` → `resize()` 扩容为 2 倍；
7. 返回旧值（覆盖）或 null（新增）。

**get 同理**：hash 定位桶 → 桶内 equals 找 key。HashMap 允许 key/value 为 null（单个 null key 固定放 0 号桶，见 2.3.4）。

### 2.3.2 HashMap 为什么容量必须是 2 的幂？负载因子为什么默认 0.75？

**2 的幂**：

1. 定位桶用 `(n-1) & hash` 代替 `hash % n`——仅当 n=2^k 时二者等价，位运算更快；n-1 二进制全 1，与 hash 按位与让分布更均匀（若 n 非 2 幂，低位会全 0 集中某些桶）。
2. 扩容迁移高效：新位置要么是**原位置**，要么是**原位置 + oldCap**，由 `hash & oldCap` 是否为 0 决定，无需重算 hash。

**负载因子 0.75**：时间与空间的折中。太大（1.0）→ 碰撞增多、桶内链更长；太小（0.3）→ 空间浪费、扩容频繁。0.75 下桶内元素数近似**泊松分布**，链表长到 8 的概率约 0.00000006——这也是树化阈值 8 的来历。

### 2.3.3 JDK1.7 和 1.8 的 HashMap 主要区别？为什么 1.7 扩容会死循环？

| 特性 | JDK1.7 | JDK1.8 |
|------|--------|--------|
| 结构 | 数组+链表 | 数组+链表+红黑树 |
| 插入 | **头插法** | **尾插法** |
| 初始化 | 构造时就建表 | 首次 put 才 resize |
| hash 扰动 | 4 次位运算 + 5 次异或 | 一次 `h ^ (h>>>16)` |
| 扩容 | 每个元素重算 index | `hash & oldCap` 二分原位/移动 |
| 树化 | 无 | 链≥8 且表≥64 树化，≤6 还原 |

**1.7 死循环原理**：头插法在 `transfer()` 迁移链表时把节点顺序反转。两个线程并发扩容同一桶链表 `[A→B]`：线程1 执行到把 B 头插、A 的 next 还指向 B；线程2 完成同样反转后表变成 `[B→A→B 环]`；此后任何线程 get 命中该桶就 infinite loop → **CPU 100%**。1.8 尾插保持原有顺序，不再成环，但并发 put 仍会**数据覆盖、size 失真**——HashMap 从来不是线程安全的。

### 2.3.4 HashMap 允许 null 键/值吗？如何处理的？

允许。key 为 null 时 hash 直接记 0，永远放 **0 号桶**；value 可以是 null，所以"get 返回 null"不能区分"没有该 key"和"值就是 null"——判断存在性用 `containsKey`，或 JDK8 的 `getOrDefault`/`computeIfAbsent`（后者注意函数里不能再改本 map，可能死循环）。

对比记忆：Hashtable 键值都不许 null（早期设计为了避免"歧义的 null"）；ConcurrentHashMap 键值都不许 null——**并发下 containsKey 和 get 是两次检查，中间值可能变化，null 的歧义无法消除**，JDK 作者 Doug Lea 直接禁掉，逼你用显式语义。`TreeMap` 键不许 null（无法 compareTo），值可 null；`TreeSet` 元素不许 null。

### 2.3.5 什么样的对象适合做 HashMap 的 key？为什么 String/Integer 常用作 key？

**要求**：hashCode/equals 稳定且符合契约（1.1.3）；重写 equals 必重写 hashCode；可变对象做 key 后**不要再修改参与 hash 计算的字段**——放得进取不出（经典事故）。

String/Integer 天然合适：**不可变**（构造后 hashCode 不会变）、equals 语义明确、且 **String 的 hashCode 有缓存**（final int hash 字段只算一次）。JDK 不可变类（包装类、LocalDate、枚举）都是安全 key。业务对象做 key：要么不可变类，要么用 record/只读字段构建 hashCode。

> **面试追问**：为什么不用 Object 默认的 hashCode？——identity hash 两个逻辑相等对象不同值，检索必然失败；这也解释了为什么自定义 key 必须成对重写。

### 2.3.6 HashMap 遍历和批量写入有哪些性能注意点？

- **遍历用 entrySet()** 或 `forEach((k,v)->...)`；keySet() 再 get 等于每次再查一遍表，多一倍开销。
- **批量写入预设容量**：`new HashMap<>(expectedSize / 0.75f + 1)`（Guava `Maps.newHashMapWithExpectedSize`），避免反复 resize。
- **computeIfAbsent / merge / getOrDefault** 取代"get 判 null 再 put"三板斧，且语义原子（单线程内）；注意 computeIfAbsent 的 mappingFunction 里不能再次修改本 map，否则可能抛 IllegalStateException 或死循环。
- JDK8 红黑树只解决"极端碰撞下的最坏性能退化"，正常负载下几乎不会树化；**恶意 hash 碰撞攻击**（Web 场景构造同名 hashCode 的参数）在 JDK7 靠扰动缓解、JDK8 靠树化兜底。

## 2.4 ConcurrentHashMap 与线程安全集合

### 2.4.1 ConcurrentHashMap JDK1.7 和 1.8 的实现区别？

**JDK1.7 分段锁**：结构 `Segment[]`（每段继承 ReentrantLock）+ 段内 `HashEntry[]`，默认 16 段 → 并发度 16。put：hash 定位段 → tryLock（自旋 64 次）→ lock 阻塞。size()：先无锁试算 2 次，不一致再全段加锁统计。段数初始化后不可变。

**JDK1.8 CAS + synchronized**：结构 Node[] + 链表 + 红黑树（与 HashMap 同构）。

- put：`spread(hash)` → 桶空 **CAS 插入**；桶非空 **synchronized 锁桶首节点**再挂链表/树；遇到 `hash=MOVED(-1)` 的 ForwardingNode 说明在扩容，协助迁移。
- 初始化：`initTable` 用自旋 CAS 竞争 table 基址。
- 扩容：**多线程协同 transfer**，以 32 个桶为粒度（stride）分配区间，`transferIndex` CAS 领取任务，忙不过来帮助搬运。
- size：`baseCount + CounterCell[]`（**LongAdder 思想**），无锁分散计数。
- 读：Node 字段 volatile，**get 全程无锁**且能看到最新值。

锁粒度从"段"细化到"桶"，并发度不再受 Segment 数限制。

### 2.4.2 ConcurrentHashMap 的 get 和 size 是怎么做到无锁的？

- **get 无锁**：table 引用 volatile、Node 的 val/next volatile、首次读 volatile 保证后续读可见（安全发布思想）。链表/树读到中间状态最多"短暂旧值"，结构一致性由 volatile 写序保证。
- **size 无锁**：不维护单一天然原子计数。更新时 CAS 改 `baseCount`；CAS 失败（竞争高）就改各自线程绑定的 `CounterCell`，`size()` 汇总 base + sum(cells)。这与 LongAdder 的"分散热点"完全同构——**精确值只在统计瞬间，先聚合再展示**。
- `mappingCount()` 是更推荐的版本（返回 long，避免 size 溢出语义）。

> **注意**：CHM 无锁 ≠ 组合操作原子。`if (!map.containsKey(k)) map.put(k,v)` 依然竞态，要用 `putIfAbsent`/`computeIfAbsent`（JDK9 修复了 compute 系列递归修改的坑，允许在函数里安全操作同一 map）。

### 2.4.3 线程安全的集合如何选型？

| 需求 | 选择 | 说明 |
|------|------|------|
| 高并发 Map | ConcurrentHashMap | CAS+锁桶头，读无锁 |
| 高并发有序 Map | ConcurrentSkipListMap | 跳表，范围操作并发友好 |
| 读多写少 List | CopyOnWriteArrayList | 快照读，写 O(n) 复制 |
| 低并发简单场景 | Collections.synchronizedXxx | 全方法粗锁，迭代仍需手动同步 |
| 队列/生产者消费者 | BlockingQueue 家族 | 阻塞语义自带 |
| 遗留 Hashtable/Vector/Stack | 不用 | JDK1.0 粗锁遗产 |

> **行业实践**：并发容器解决的是**单个方法**的线程安全，**复合业务操作**（读-改-写、跨键一致性）仍要业务层锁或 compute 系列原子方法；这是"用了 ConcurrentHashMap 还出并发 bug"的最常见原因。

## 2.5 Map/Set 其他成员

### 2.5.1 LinkedHashMap 如何实现 LRU 缓存？

LinkedHashMap 继承 HashMap，Entry 额外维护 before/after 指针，把所有节点串成**双向链表**记录顺序：

- `accessOrder=false`（默认）：插入顺序。
- `accessOrder=true`：**访问顺序**——每次 get/put 命中把节点移到链尾，链头即"最久未使用"。

LRU 实现 = 访问顺序 + 超限逐出：

```java
public class LRUCache<K,V> extends LinkedHashMap<K,V> {
    private final int capacity;
    public LRUCache(int capacity) {
        super(capacity, 0.75f, true);          // 开启访问顺序
        this.capacity = capacity;
    }
    @Override
    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return size() > capacity;               // put 后回调，true 则删链头
    }
}
```

机制：HashMap.putVal 插入后回调 `afterNodeInsertion(true)`，其中检查 removeEldestEntry 决定是否 `removeNode` 最老元素。手写 LRU（HashMap+双向链表）是同一思想的面试变体。

### 2.5.2 HashSet 如何保证元素唯一？

底层就是 HashMap：`add(e)` 转调 `map.put(e, PRESENT)`——value 是统一的 `static final Object PRESENT` 占位常量。判重复用 HashMap key 逻辑：**hash 定位桶 + equals 确认**，已存在则 put 返回旧 value（add 返回 false）。

推论：元素必须遵守 hashCode/equals 契约（1.1.3）；`LinkedHashSet` 用 LinkedHashMap 构造维护插入顺序；`TreeSet` 用 TreeMap，唯一性靠 **compareTo 返回 0**（不是 equals！浮点 compareTo 的 0.0/-0.0 陷阱在此体现）。

### 2.5.3 TreeMap 如何实现排序？红黑树有哪些特性？

**排序两选一**（优先级 Comparator > Comparable）：key 实现 Comparable 自然排序，或构造时传入 `TreeMap(Comparator)`；比较结果为 0 视为同一个 key（见 2.5.2 的 -0.0 陷阱，`put(new Double(-0.0))` 会被 `0.0` 顶掉）。

**红黑树五条性质**：节点非红即黑；根为黑；NIL 叶为黑；红节点的子必为黑（无连续红）；任一节点到其所有叶的路径黑节点数相同（黑高一致）。由此推出**最长路径 ≤ 2×最短路径**——近似平衡，查/插/删 O(log n)，插入最多 3 次旋转、删除最多旋转+染色组合，工程上比 AVL（严格平衡、旋转更多）综合更优。

TreeMap 独有能力：`floorKey/ceilingKey/lowerKey/higherKey`（范围检索）、`firstKey/lastKey`、`subMap` 视图——这也是它比"排序 List"更适合有序字典场景的原因；并发替代是 ConcurrentSkipListMap（跳表实现同样的有序语义）。

## 2.6 队列家族

### 2.6.1 PriorityQueue 的底层原理？

底层 `Object[]` 实现的**完全二叉堆**（默认小顶堆）：父节点 i 的左右子为 `2i+1`、`2i+2`。元素需 Comparable 或构造传 Comparator。

- **入队 offer**：追加到数组末尾 → `siftUp` 与父比较上浮，O(log n)。
- **出队 poll**：堆顶（最小）拿走，**末元素放顶** → `siftDown` 与更小的子交换下沉，O(log n)。
- `peek()` O(1)；不支持 null、不保证出队稳定排序全序（相等元素不约定先后）；**非线程安全**，并发版 PriorityBlockingQueue（内部单锁，无界）；延时取用 DelayQueue（内部就是 PriorityQueue + ReentrantLock + Condition，元素实现 Delayed）。
- 建堆 `heapify` 是 O(n) 不是 O(n log n)。

### 2.6.2 BlockingQueue 有哪些实现？说说生产者-消费者模式

常用实现对比：

| 队列 | 边界 | 底层 | 锁 | 特点 |
|------|------|------|----|------|
| ArrayBlockingQueue | 有界（必填） | 数组 | 单 ReentrantLock + notFull/notEmpty 两个 Condition | FIFO，缓存行友好 |
| LinkedBlockingQueue | 可选有界（默认 Integer.MAX_VALUE） | 链表 | **putLock/takeLock 双锁** | 吞吐更高；"默认无界"是隐患 |
| SynchronousQueue | 容量 0 | 交接（TransferQueue/Stack） | CAS | put 必须等到 take，线程池 Cached 用它 |
| PriorityBlockingQueue | 无界 | 二叉堆 | 单锁 | 按优先级出队 |
| DelayQueue | 无界 | 堆 + Delayed | 单锁 + Condition | 到期才能取 |

**生产者-消费者**：生产者 `put(e)`（满则 await(notFull)）、消费者 `take()`（空则 await(notEmpty)），入队/出队后 signal 对侧条件队列——一把锁两个 Condition 即完成"满了等待、空了等待"的协调。工程上队列**必须有界**，无界 LinkedBlockingQueue 只是把 OOM 从线程数挪到堆内存（线程池场景同理，见 3.5）。


# 3. 并发编程

## 3.1 线程基础

### 3.1.1 创建线程有哪几种方式？start() 和直接调 run() 的区别？

1. 继承 `Thread` 重写 run；2. 实现 `Runnable` 交给 Thread（推荐，避免单继承限制、任务与执行解耦）；3. 实现 `Callable<V>` + `FutureTask`（有返回值、可抛受检异常）；4. 线程池提交（生产标准做法）；5. JDK21 虚拟线程 `Thread.ofVirtual()`（见 3.6.4）。

**start vs run**：`start()` 请求 JVM 经操作系统创建新线程，由新线程回调 run()；直接调 `run()` 只是当前线程的一次普通方法调用，没有任何并发。start 只能调一次，重复调抛 `IllegalThreadStateException`。

### 3.1.2 线程的生命周期和状态转换？

`Thread.State` 六种：

| 状态 | 含义 | 进入条件 |
|------|------|---------|
| NEW | 已创建未启动 | new Thread() |
| RUNNABLE | 就绪 + 正在运行（OS 的 ready/running 合并） | start() 后；等待 IO 也在 RUNNABLE（native 层阻塞） |
| BLOCKED | 等待 synchronized 监视器锁 | 进同步块抢锁失败 |
| WAITING | 无限期等待被唤醒 | `wait()`、`join()`、`LockSupport.park()` |
| TIMED_WAITING | 限时等待 | `sleep(t)`、`wait(t)`、`join(t)`、`parkNanos/till` |
| TERMINATED | 结束 | run 完成或未捕获异常退出 |

```text
NEW →start()→ RUNNABLE ⇄ BLOCKED(抢锁失败/获锁)
                    ⇄ WAITING(wait/park/join → notify/unpark/join完成)
                    ⇄ TIMED_WAITING(sleep/wait(t)/parkNanos → 超时或唤醒)
                    → run 结束 → TERMINATED
```

> **易错**：`sleep` 不释放锁，WAITING/BLOCKED 的区分看"在等什么"——等锁是 BLOCKED，等条件是 WAITING；WAITING 线程不会被中断唤醒（park 除外，park 返回也可能是虚假唤醒，要循环判断条件）。

### 3.1.3 wait 和 sleep 的区别？

| 维度 | wait()（Object 方法） | sleep()（Thread 静态方法） |
|------|----------------------|---------------------------|
| 锁 | **释放监视器锁** | 不释放锁 |
| 使用前提 | 必须在 synchronized 内，否则 IllegalMonitorStateException | 任意位置 |
| 唤醒 | notify/notifyAll、中断、超时 | 超时自动醒、中断（抛 InterruptedException） |
| 归属 | 对象监视器的等待集 | 当前线程自身 |

**为什么 wait 是 Object 的方法**：等待/唤醒围绕"任意对象的监视器锁"进行，锁不是 Thread 的属性；synchronized 可以锁任何对象，wait 必须跟着锁的宿主走。配合记忆：Condition 的 await/signal 是同一角色的 API 层复刻（见 3.3.3）。

### 3.1.4 线程间通信/协作的方式有哪些？

1. `wait()/notifyAll()`——synchronized + 对象监视器；
2. `Condition.await()/signal()`——ReentrantLock，可多条件队列（如 notFull/notEmpty 分离）；
3. `LockSupport.park()/unpark(thread)`——更底层，可不配对时序（先 unpark 后 park 也能立刻返回）、支持精准唤醒；wait/notify 本质也是它的封装；
4. JUC 同步器：CountDownLatch / CyclicBarrier / Semaphore（见 3.6.1）；
5. `BlockingQueue.put/take`——生产消费解耦，首选的数据协作方式；
6. volatile 轮询（只适合状态标志位，不适合精确的"等待-通知"）；
7. `join()`/Future.get——等待结果。

**选型**：数据传递用队列，时序等待用同步器，自定义条件用 Lock+Condition；裸 wait/notify 只在手写并发组件（线程池、连接池）时出现。

### 3.1.5 什么是死锁？如何排查和避免？

**结论**：两个以上线程互相持有对方所需资源、又互不退让，永久阻塞。

**四个必要条件**（破坏任意一个即可预防）：互斥、占有且等待、不可剥夺、循环等待。

**排查**：

1. `jps -l` 找进程 PID；
2. `jstack <PID>`（或 `jcmd <PID> Thread.print`）——输出末尾直接给出 `Found one Java-level deadlock`，列出互相等待的锁与栈帧；
3. `jconsole/VisualVM` 的线程页"检测死锁"按钮；
4. 线上常用 Arthas `thread -b` 一步定位 blocking 线程对。

**避免**：固定加锁顺序（所有线程同序拿锁）；用 `tryLock(timeout)` 拿不到就放弃重试；缩小锁粒度与嵌套层数；能用无锁结构（ConcurrentHashMap、原子类）就不手写锁。数据库死锁另有一套排查（`SHOW ENGINE INNODB STATUS`，见 16_mysql）。

## 3.2 JMM、volatile 与 CAS

### 3.2.1 什么是 Java 内存模型（JMM）？happens-before 是什么？

**JMM**：Java 内存模型不是"堆栈划分"，而是**多线程可见性的抽象规范**——规定每个线程有工作内存（缓存/寄存器的抽象），共享变量在主内存；线程对变量的读写在工作内存进行，何时同步到主内存由 JMM 规则约束。编译器与 CPU 还会**指令重排序**（as-if-serial 只保证单线程语义，不保证多线程）。

**happens-before** 八条核心规则（A happens-before B ⇒ A 的结果对 B 可见，且顺序不被重排跨过）：

1. 程序顺序规则（单线程内）；
2. **监视器锁定**：unlock 先于后续对该锁的 lock；
3. **volatile**：写先于后续读；
4. 传递性；
5. start()：主线程 start 前的所有写对线程可见；
6. join()：线程结束前的所有写对 join 返回后的代码可见；
7. 线程中断：interrupt() 先于检测到中断；
8. 对象构造：构造完成先于 finalize。

**底层保障**：x86 上 volatile 写用 lock 前缀指令（写合并缓冲刷新 + MESI 使其它核缓存行失效），配合 StoreStore/StoreLoad 等屏障禁排序。三大特性：**原子性、可见性、有序性**是贯穿并发题的骨架，synchronized/volatile/final/happens-before 分别给出不同强度的保障。

### 3.2.2 volatile 的作用？为什么 DCL 单例必须加 volatile？

**保证**：可见性（写立即刷主存、读强制拉最新）+ 有序性（内存屏障禁重排）。**不保证原子性**——`i++` 是读-改-写三步复合，volatile 照样出错。

DCL 的问题在 `instance = new Singleton()` 一步：字节码是"分配内存 → 执行构造初始化 → 引用赋值"三步，JIT 可能重排为 ①③②：

```java
public class Singleton {
    private static volatile Singleton instance;   // 没有 volatile 就出洞
    public static Singleton getInstance() {
        if (instance == null) {                   // ① 第一次检查(无锁)
            synchronized (Singleton.class) {
                if (instance == null) {           // ② 第二次检查
                    instance = new Singleton();   // 可能被重排
                }
            }
        }
        return instance;
    }
}
```

线程 A 走到"引用已赋值但未初始化"，线程 B 在①处读到非 null 直接用→拿到半成品对象。volatile 禁止③先于②重排（StoreStore 屏障），同时保证引用写入对其他线程立即可见。

### 3.2.3 CAS 的原理？ABA 问题及解决方案？

**CAS**（Compare And Swap）：CPU 原子指令（x86 `CMPXCHG`）——比较内存值 V 与预期值 A，相等则更新为 B 并返回 true，否则不动返回 false，由调用方自旋重试。Java 入口：JDK8 `sun.misc.Unsafe.compareAndSwapInt/Object/Long`，JDK9+ 推荐 `VarHandle`；上层的 AtomicInteger、AQS 的 state、ConcurrentHashMap 的桶插入都建立在它上面。

**三个缺陷**：

1. **ABA**：值被改成 B 又改回 A，CAS 无法感知中间历史。链表头被别的线程摘除又复用同一节点时后果严重。解决：**版本号**——`AtomicStampedReference`（引用+int stamp，每次更新 stamp++）；只需知道"被动过没有"用 `AtomicMarkableReference`（boolean mark）。
2. 自旋开销：竞争激烈时空转耗 CPU，可用有限自旋+退避。
3. 只能保证**单变量**原子：多字段复合更新要用锁、`AtomicReference` 整体替换不可变对象，或 LongAdder 这类分段结构。

### 3.2.4 AtomicLong 和 LongAdder 的区别？

高并发计数场景：**AtomicLong** 所有线程 CAS 同一个 value 字段，竞争激烈时大量 CAS 失败自旋，吞吐随线程数恶化。**LongAdder**（JDK8）把热点拆开：`base` + `Cell[]`，无竞争直接改 base；有竞争线程映射到各自 Cell 分别累加（Cell 内仍是 CAS，缓存行填充 @Contended 防伪共享），`sum()` 时汇总——**写分散、读聚合**。

代价：sum 不是瞬时精确值（弱一致）；占用更多内存。选型：高并发计数用 LongAdder；需要 compareAndSet/单调序号仍用 AtomicLong（ConcurrentHashMap 的 size 维护就是 LongAdder 思想，见 2.4.2）。

## 3.3 synchronized 与锁

### 3.3.1 synchronized 的锁升级过程？（含偏向锁/轻量级/重量级）

JDK6 后基于对象头 MarkWord 低位标记（11 GC、01 无锁/偏向、00 轻量级、10 重量级），**只升不降**：

1. **无锁**：新建未加锁。
2. **偏向锁**：第一个线程 CAS 把线程 ID 写入 MarkWord；此后同线程重入只查 ID，**零 CAS**。出现第二个线程竞争 → 到 safepoint 撤销偏向 → 升级。JDK15 起默认禁用、18 移除（现代应用共享访问多，偏向收益不抵复杂度）。
3. **轻量级锁**：线程在栈帧建 LockRecord（存 MarkWord 副本），CAS 把对象头指向它；失败说明有竞争 → **自适应自旋**等待（自旋次数由 JIT 根据历史成功率动态决定）。
4. **重量级锁**：自旋超时或等待者增多 → MarkWord 指向 ObjectMonitor（EntryList 阻塞队列 / WaitSet 条件队列），线程挂起进内核态，靠 OS 调度唤醒，上下文切换成本高。

**触发重量级的常见情况**：竞争加剧、调用 wait（需要 WaitSet）、锁膨胀。

**JIT 配套优化**：**锁消除**（逃逸分析确定锁对象不外泄则直接删锁）、**锁粗化**（连续加解锁合并成大块）。synchronized 修饰静态方法锁 Class 对象、修饰实例方法锁 this、修饰代码块锁指定对象——粒度可控是优化第一步。

### 3.3.2 synchronized 和 ReentrantLock 的区别？如何选择？

| 维度 | synchronized | ReentrantLock |
|------|-------------|---------------|
| 层级 | JVM 关键字（Monitor） | JDK API（AQS） |
| 释放 | 自动（异常也释放） | 必须 finally unlock() |
| 可中断 | 不可 | lockInterruptibly() |
| 超时尝试 | 无 | tryLock(timeout) |
| 公平性 | 非公平 | 可选公平/非公平 |
| 条件队列 | 单 wait/notify | **多个 Condition** |
| 锁状态查询 | 不可 | isLocked 等 |
| 性能 | JDK6 锁升级后接近 | 相当（高级功能换来手动开销） |

**选择**：默认 synchronized——简洁、不会忘解锁、JVM 优化空间大；需要**可中断/超时/公平/多条件**（生产者消费者的 notFull/notEmpty 分离、读写锁降级场景）才上 ReentrantLock。注意：ReentrantLock 加锁失败会因忘记 unlock 造成永久死锁；synchronized 则可能在异常路径长时间持锁，两者都不是"无脑加锁"的理由。

### 3.3.3 ReentrantLock 的实现原理？公平锁与非公平锁的区别？

基于 **AQS**（state=重入次数，exclusiveOwnerThread=持有者），子类 FairSync / NonfairSync。

- **非公平 lock**：先 `CAS(state 0→1)` **插队抢一次**，失败才进 `acquire(1)` 排队；tryAcquire 里 state==0 再 CAS，state>0 且持有者是当前线程则 **state+1**（重入）。
- **公平 lock**：tryAcquire 先 `hasQueuedPredecessors()`——队列里有人排队就不抢，老实入队。公平 = 消除饥饿但吞吐降低（每次切换队头都强制线程上下文切换，浪费唤醒窗口）。
- **解锁**：tryRelease state-1，减到 0 清空持有者并 `unpark` 队头。
- **Condition**：`newCondition()` 返回 AQS 内部 ConditionObject，**一个 Lock 可建多个条件队列**；`await()` = fullyRelease + 入条件队列 + park；`signal()` = 把条件队列首节点**转移到同步队列**（不是直接拿锁），等它再竞争。这解释了为什么 signal 后要"重新抢锁"。

### 3.3.4 AQS 的原理是什么？

AQS（AbstractQueuedSynchronizer）是 JUC 同步器框架：**volatile int state + CLH 变体的 FIFO 双向队列**（Node 存线程、waitStatus：SIGNAL/CONDITION/CANCELLED/PROPAGATE、独占/共享模式）。

- **独占模式**（ReentrantLock）：`acquire → tryAcquire 失败 → addWaiter 入队 → acquireQueued 自旋：前驱是 head 才尝试拿锁，否则 park 等 unpark/超时`；释放 `release → tryRelease → 唤醒 head.next`。取消节点通过 setSuccessor 修正链。
- **共享模式**（Semaphore/CountDownLatch）：`tryAcquireShared` 返回剩余许可；成功后 `setHeadAndPropagate` 级联唤醒后续共享节点（允许多线程同时通过）。
- 子类只需实现 tryAcquire/tryRelease（或共享版）+ isHeldExclusively，队列、park/唤醒全部复用——这是"模板方法模式在并发包的落地"。

Semaphore 用 state=剩余许可，CountDownLatch 用 state=计数（只减不增不可复用），ReentrantReadWriteLock 用 **state 高 16 位读计数 + 低 16 位写计数**。

### 3.3.5 ReentrantReadWriteLock 有什么问题？StampedLock 好在哪？

读写锁：state 高位读/低位写，**读读共享、读写/写写互斥**；写锁可**降级**为读锁（先持写再取读再放写，保证过渡不断档），**读锁不能升级写锁**（多个读线程同时升级互等→死锁）。

经典问题**写线程饥饿**：读锁源源不断，写锁永远排队 → 升级写锁的降级方案也失效。

**StampedLock（JDK8）**：引入**乐观读** `tryOptimisticRead()`——不加锁拿一个 stamp 直接读，读完 `validate(stamp)` 检查期间无写者提交，失败才升级为悲观读。读吞吐最高；但**不可重入、不支持 Condition**、API 更繁琐易错（stamp 校验忘写就出静默 bug）。工程折中：读极多、可校验一致性时考虑，一般场景读写锁甚至粗粒度 synchronized 已够。

## 3.4 ThreadLocal

### 3.4.1 ThreadLocal 的原理和内存泄漏问题？

**原理**：数据不存 ThreadLocal 对象里——每个 `Thread` 内部有 `ThreadLocalMap threadLocals`，**以 ThreadLocal 实例为 key、线程的变量副本为 value**。get/set 先拿当前线程，再操作其 map。map 是开放地址法线性探测的哈希表（初始 16，阈值 2/3），key 的哈希用 ThreadLocal 自身的 `hashCode`（AtomicInteger 递增的魔数散列）。

**泄漏链**：Entry 的 **key 是弱引用**（继承 WeakReference），**value 是强引用**。外部强引用 ThreadLocal 消失后，GC 回收 key → Entry 变成 `null → value`，仍被 `Thread → map → Entry → value` 引用无法回收；线程池线程不死，泄漏持续累积。

**治理**：

1. 唯一可靠做法：**finally 里 `remove()`**（get/set/remove 顺带的 expungeStaleEntry 清理不可依赖）；
2. ThreadLocal 声明为 `private static final`（类级生命周期，key 不消失）；
3. 框架层典型应用：SimpleDateFormat 隔离、Spring TransactionSynchronizationManager 事务上下文、MDC 日志链路——都有配套 filter/interceptor 收尾 remove。

### 3.4.2 InheritableThreadLocal 和 TransmittableThreadLocal 的区别？

- **InheritableThreadLocal**：仅在**创建子线程时**（`Thread.init()`）把父线程的表**浅拷贝**给子线程——线程池复用早已存在的线程时完全无效，拷贝的也是引用（可变对象两边共享，改一值两变）。
- **TransmittableThreadLocal（阿里 TTL）**：解决"**提交任务时**"的上下文传递——包装 Runnable/Callable，在**提交**时捕获当前上下文快照、**执行**时回放进工作线程、执行完恢复。配 `TtlExecutors.getTtlExecutorService(pool)` 接入线程池。

场景：全链路 traceId、登录态在多层线程池间透传。虚拟线程场景则考虑 JDK21 的 Scoped Values（见 4.4.7）。

## 3.5 线程池

### 3.5.1 线程池的 7 个核心参数和执行流程？

```java
new ThreadPoolExecutor(
    corePoolSize,      // 核心线程数（常驻，默认不回收）
    maximumPoolSize,   // 最大线程数
    keepAliveTime,     // 非核心线程空闲存活时长（allowCoreThreadTimeOut=true 也适用核心）
    unit,
    workQueue,         // 阻塞队列：ArrayBQ/LinkedBQ/SynchronousQueue/PriorityBQ
    threadFactory,     // 线程工厂：命名/daemon/异常处理器
    handler);          // 拒绝策略
```

**提交流程**：线程数 < core → 建核心线程；≥ core → 入队；队满 且 < max → 建非核心线程；≥ max 且队满 → 拒绝策略。注意顺序是"**先队再非核心**"（与直觉相反的常问点），且核心线程可预启动 `prestartAllCoreThreads()`。

**为什么禁用 Executors 快捷工厂**：`newFixedThreadPool/newSingleThreadExecutor` 用无界 LinkedBlockingQueue → 任务无限堆积 **OOM**；`newCachedThreadPool` max=Integer.MAX_VALUE → 线程爆炸 **OOM/上下文切换雪崩**。生产必须手动 ThreadPoolExecutor，队列必须有界。

### 3.5.2 4 种拒绝策略分别是什么？如何自定义？

1. **AbortPolicy**（默认）：抛 `RejectedExecutionException`，调用方感知失败——适合"宁失败不积压"。
2. **CallerRunsPolicy**：提交线程自己执行——天然的**反压降速**，主线程干活期间无法继续提交；但会阻塞提交线程，用在 Tomcat/网关线程上会拖慢入口。
3. **DiscardPolicy**：静默丢弃新任务——只适合可丢数据（日志采样）。
4. **DiscardOldestPolicy**：丢队头最老任务，重试提交当前——适合"新数据更值钱"（行情、心跳）。

自定义实现 `RejectedExecutionHandler`：常见组合是**落库/MQ 兜底 + 告警 + 限次重试**，把"被拒"变成可观测事件而不是丢任务。

### 3.5.3 线程池大小怎么设置？

**起点公式**：CPU 密集型 `N_cpu + 1`（留一给页缺失等）；IO 密集型近似 `N_cpu × (1 + 等待时间/计算时间)`，或经验值 2N。这只是**起跑线**：

- 区分快慢任务拆分多个池（隔离原则：一个慢接口拖死共享池是典型事故；舱壁模式/不同业务不同线程池）。
- 用**压测 + 观测**调参：监控队列堆积、拒绝次数、任务排队时间（队列积压说明核心不够，队列长而不加 max 只延后爆点）。
- 动态调参：暴露 `setCorePoolSize/setMaximumPoolSize`（ThreadPoolExecutor 支持运行时改）接配置中心，大促前调大。
- 别忘任务本身优化（合并提交、减少同步块）——线程数不是万能药。

### 3.5.4 线程池如何关闭？shutdown 和 shutdownNow 的区别？线程为什么能复用？

- **shutdown()**：状态置 SHUTDOWN，不再收新任务，**已入队任务继续跑完**，worker 耗尽自然退出。
- **shutdownNow()**：状态 STOP，中断所有 worker、清空队列并返回未执行任务列表。
- 标准姿势：shutdown → `awaitTermination(timeout)` → 超时再 shutdownNow，配合拒绝策略兜底。
- **复用原理**：worker 循环 `runWorker → getTask()`，`workQueue.poll(keepAliveTime)` **阻塞等任务**，拿到就执行——线程不退出，避免反复创建销毁；空闲超时的非核心线程 getTask 返回 null 自然退出。所以"线程池复用线程"本质是**线程阻塞在队列上**，队列才是心脏。

## 3.6 并发工具与异步编排

### 3.6.1 CountDownLatch、CyclicBarrier、Semaphore 的区别？

| 特性 | CountDownLatch | CyclicBarrier | Semaphore |
|------|---------------|---------------|-----------|
| 计数 | 只减，一次性 | 到达数递增，可 reset **循环** | 许可增减 |
| 作用 | "等 N 件事做完" | "凑齐 N 人一起走" | "最多 N 人同时进" |
| 触发 | countDown()/await() | await()（最后一个到达触发可选回调） | acquire()/release() |
| 原理 | AQS 共享，state=计数 | ReentrantLock + Condition | AQS 共享，state=许可 |
| 场景 | 主线程等多个并行 RPC | 分段计时起点、多线程汇合 | 限流、连接池 |

- Latch 计数归零后所有 await 立即放行，**不可复用**；
- Barrier 的 `isBroken()`：有线程超时/中断则整道屏障破坏，其余线程抛 BrokenBarrierException——设计语义是"要么一起、要么都不"；
- Semaphore 注意 **公平/非公平构造**、`acquire(n)` 批量、以及"忘记 release 导致许可泄漏"——放 finally。

### 3.6.2 CompletableFuture 的常用方法和异步编排场景？

JDK8 异步编排核心，实现 Future 与 CompletionStage：

- **创建**：`supplyAsync(supplier[, executor])` 有返回、`runAsync` 无返回。**强烈建议显式传自定义线程池**，默认 ForkJoinPool.commonPool 全 JVM 共享，被阻塞任务占满会拖累 parallelStream 等所有使用者。
- **串联**：`thenApply`（转换）、`thenAccept`（消费）、`thenRun`（不关心结果）、`thenCompose`（扁平化接下一个异步任务，等价 flatMap）、`thenCombine`（两个 future 结果合并）。XxxAsync 后缀=换线程执行，无后缀=完成线程或调用线程执行。
- **汇聚**：`allOf(...)` 全部完成（返回 Void，需逐个 join 取值）、`anyOf(...)` 任一完成。
- **异常**：`exceptionally`（兜底值）、`handle`（成败都走、可转换）、`whenComplete`（旁观不改结果）。异常在链条里以 CompletionException 包装传播，没人消费就静默——**每个链尾必须收尾处理异常**。
- **超时**：JDK9 `orTimeout/completeOnTimeout`。

典型场景：商品详情页——价格/库存/评价三路 RPC `thenCombine` 并行聚合；查询→富化→落库→发通知链式编排。

### 3.6.3 ForkJoinPool 和工作窃取是什么？

JDK7 的**分治任务框架**：`RecursiveTask<V>`（有返回）/`RecursiveAction` 重写 `compute()`，fork 子任务、join 汇总。

**工作窃取（work-stealing）**：每个工作线程维护自己的**双端队列**，执行时从**头部 LIFO** 拿自己的任务（缓存友好）；空闲线程从别人队列**尾部 FIFO** 偷——尾端竞争小、减少冲突。适合"可拆分、CPU 密集"的任务：大数组求和、归并排序；`Arrays.parallelSort`、parallelStream 底层就是它（commonPool）。

注意：**join 会阻塞当前 worker**，任务里有 IO 阻塞会耗尽池；异常不显式抛需主动 get。

### 3.6.4 Java 21 虚拟线程是什么？和平台线程的区别？

虚拟线程（Project Loom，JEP 444）：**JVM 调度的轻量线程**，M:N 映射到少量平台线程（载体线程）上；阻塞时（IO/sleep/LockSupport.park）自动**卸载**出载体线程，让载体继续跑别的虚拟线程。

```java
Thread.ofVirtual().start(task);
Thread.startVirtualThread(task);
try (var exec = Executors.newVirtualThreadPerTaskExecutor()) { ... }
```

对比：平台线程 1:1 包装 OS 线程，栈默认约 1MB、创建昂贵——所以传统用"线程池复用"；虚拟线程创建成本≈建对象（栈在堆上按需增长），**不要池化，一任务一线程**，百万级并发靠它把"同步写法、异步吞吐"统一起来，替代 WebFlux 的大部分动机。

**坑**：`synchronized` 块内阻塞会 **pinning** 固定载体线程（JDK21 时代问题，后续版本已大幅改进，迁移期仍建议长临界区换 ReentrantLock）；ThreadLocal 在海量虚拟线程下内存可观（用 Scoped Values）；CPU 密集型任务无收益（并行度还是核数）。


# 4. Java 8+ 新特性

## 4.1 Lambda 与函数式接口

### 4.1.1 Lambda 表达式的本质是什么？如何编译的？

**结论**：Lambda 不是匿名内部类的语法糖——编译期**不生成 `Outer$1.class`**，而是把 Lambda 体抽成所在类的私有静态/实例方法，调用点用 `invokedynamic` 指令，运行期由 `LambdaMetafactory` 动态生成实现类（早期 JDK 用 ASM，JDK 后续版本逐步改用 `直接方法句柄`，连类都不用生成）。

与匿名内部类的关键差异：

| 维度 | 匿名内部类 | Lambda |
|------|-----------|--------|
| 编译产物 | 独立 .class | invokedynamic + 合成方法 |
| this | 指向内部类实例 | **指向外层类实例**（没有新作用域） |
| 变量遮蔽 | 可以重新声明同名变量 | 不行（同一作用域语义） |
| 实例 | 每次 new 都是新对象 | 无捕获的 Lambda 可被 JVM 缓存复用单例 |

### 4.1.2 常见的函数式接口和方法引用形式？

四大核心函数式接口（java.util.function）：

| 接口 | 抽象方法 | 语义 |
|------|---------|------|
| `Function<T,R>` | `R apply(T)` | 转换 |
| `Consumer<T>` | `void accept(T)` | 消费 |
| `Supplier<T>` | `T get()` | 供给 |
| `Predicate<T>` | `boolean test(T)` | 判断 |

扩展族：`BiFunction<T,U,R>`、`UnaryOperator<T>`（入出同型，Function 特例）、`BinaryOperator<T>`（Comparator 是它带约定的子接口）、`Supplier` 的Boolean/Int/Long 变体避免装箱。`@FunctionalInterface` 标注唯一抽象方法的接口（default/static 不计），编译器强制约束。

**方法引用四种形态**：`类::静态方法`（Integer::parseInt）、`对象::实例方法`（list::contains）、`类::实例方法`（String::compareToIgnoreCase，第一个实参成为调用者）、`类::new`（构造器引用，数组构造 `int[]::new`）。方法引用能替代 Lambda 时可读性更好，但带参数适配时（`(a,b) -> b.compareTo(a)`）显式 Lambda 更清楚。

### 4.1.3 Lambda 捕获变量的规则？为什么不能修改外部局部变量？

捕获的局部变量必须 **effectively final**（与匿名内部类同源，见 1.8.2：值拷贝语义）。捕获的是**值的副本**（基本类型）或**引用的副本**（对象引用——引用不能换，但对象内容可变，`AtomicInteger`/数组/可变对象正是绕开限制的惯用技巧，代价是可见性需自己保证）。实例字段、静态字段、参数、外部类 this 都可自由访问——它们是引用同一份堆内存，没有拷贝。

## 4.2 Stream

### 4.2.1 Stream 的中间/终止操作有哪些？惰性求值怎么理解？

**中间操作**（返回 Stream，可继续链式）：filter、map、flatMap、peek、sorted、distinct、limit、skip。

**终止操作**（触发计算，管道消费后不可复用）：forEach、collect、reduce、count、min/max、findFirst/findAny、anyMatch/allMatch/noneMatch。

**惰性求值**：中间操作只搭建管道不遍历，**遇到终止操作才一次遍历按需拉动全部中间操作**（pull 模型：collect 每拉一个元素顺序穿过 filter→map）。好处：①多次中间操作合并为单次遍历；②**短路**——limit/findFirst/anyMatch/allMatch/noneMatch/findAny 满足条件即停止，无限流（Stream.iterate/generate）只可能靠短路终止。

> **易错**：Stream 不能重复消费（终止后再调抛 IllegalStateException）；peek 是调试用途，不要当副作用通道；forEach 破坏函数式语义，能用 collect/reduce 就不用 forEach。

### 4.2.2 并行流 parallelStream 的注意事项？

1. **默认走 ForkJoinPool.commonPool()**（并行度 CPU 核数-1），全 JVM 共享——任务里含 IO 阻塞会拖垮别人的 parallelStream；用 `forkJoinPool.submit(() -> stream.parallel()...)` 包裹指定自己的池。
2. **数据量小不划算**：拆分/合并/线程调度有固定成本，经验上万级元素以上才可能正收益，先测再用。
3. **顺序与稳定性**：并行下 forEach/findFirst 语义改变（要保序用 forEachOrdered；findFirst 依赖有序源才有确定结果）；limit/skip 在并行下开销大。
4. **线程安全**：源结构不可并发修改（ArrayList.add 会乱）、累加器用 collect 而非外部共享变量；分组收集器换 `groupingByConcurrent`。
5. **上下文丢失**：commonPool 线程不是你提交的线程，ThreadLocal（登录态、traceId）拿不到——与 3.6.2 同一坑。

### 4.2.3 Collectors 常用收集器与 groupingBy/toMap 注意点？

- 收集：`toList/toSet/toMap`、`joining(",")`、`counting()`、`summarizingInt/Long/Doubles`、`reducing`、`collectingAndThen`（收集后二次转换）、`groupingBy/classifier` 分组、`partitioningBy` 二分（Map`<Boolean,List>`）。
- **groupingBy 多级分组**：`groupingBy(type, groupingBy(status, counting()))`，内层可套 any downstream（toList、mapping 提取字段）。
- **toMap 三个坑**：key 冲突默认抛 `IllegalStateException` → **永远传第三个 merge 函数** `(a,b)->a` 或取新值；**value 为 null 抛 NPE**（HashMap.merge 的限制，null 值场景用 forEach 手动 put）；要可变 Map 用 toMap，要并发用 toConcurrentMap。
- `collect` 与 `Collections.unmodifiableList` 组合返回只读结果是服务层常用防御。

## 4.3 Optional 与新时间 API

### 4.3.1 Optional 的正确用法和最佳实践？

容器对象，把"可能没有值"编码进类型。**创建**：`of`（null 抛异常）、`ofNullable`、`empty`。**取值**：`orElse(default)`——**无论有没有值都会计算默认值表达式**（可能白白执行），`orElseGet(supplier)` 懒求值（优先）、`orElseThrow(supplier)` 缺失即抛、`ifPresent/ifPresentOrElse`。**链式**：map/flatMap/filter。

**最佳实践（反模式清单）**：

- 不用于**字段**（不可序列化、语义怪；DTO 用包装类型 + 注解）。
- 不用于**方法入参**（调用方被迫包 Optional，用重载/默认值替代）。
- 不把集合/数组包进去（空集合本身就是"无"）。
- 不写 `opt.isPresent() ? opt.get() : d` 这种"Optional 版三元"——直接 `orElse`。
- 适合的位置：**查找类方法的返回值**（findUserById → Optional`<User>`），强迫调用方处理"不存在"。

### 4.3.2 java.time 新日期 API 相比 Date/Calendar 有哪些优势？

| 旧 API | java.time（JSR-310） |
|--------|---------------------|
| Date 可变、SimpleDateFormat **线程不安全** | 全部不可变，DateTimeFormatter 线程安全 |
| year 从 1900 起、month 从 0 起 | 语义直读（Month.JANUARY 枚举可选） |
| Date 混杂日期与时间 | LocalDate/LocalTime/LocalDateTime/Instant/ZonedDateTime 职责分明 |
| 时区靠 GMT 字符串，混乱 | ZoneId/ZonedDateTime 显式时区 |
| Calendar 笨重 | TemporalAdjusters 日期调节器 |
| Duration/Interval 缺失 | Period（日期差）/Duration（时间差） |

```java
LocalDate now = LocalDate.now();
LocalDate nextMonday = now.with(TemporalAdjusters.next(DayOfWeek.MONDAY)); // 下周一
DateTimeFormatter fmt = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
String s = LocalDateTime.now().format(fmt);
```

**实践要点**：存库/传参用 Instant（epoch 毫秒/秒 + 时区无关）；展示层才 LocalDateTime；`Date ↔ LocalDateTime` 桥接 `toInstant().atZone(ZoneId.systemDefault())`。Spring 的 Jackson 配 `JavaTimeModule` 序列化为 ISO-8601。

## 4.4 语言特性演进

### 4.4.1 接口为什么引入 default 方法？冲突如何解决？

**动因**：JDK8 给 Collection 加 stream()——若声明为抽象方法，所有实现类（含海量第三方类）都要改代码重编译，接口演进被冻结。default 让"接口加方法"向后兼容（同时配 static 方法让工具方法回归接口本身，如 `Comparator.naturalOrder()`，伴生 Util 类可以退役）和 JDK9 的 private default 辅助方法。

**冲突裁决三规则**：

1. **类优先**：类/父类中的同签名方法永远赢，接口 default 被忽略。
2. **子接口优先**：两接口有继承关系时，更具体（子接口）的实现生效。
3. **必须重写**：两个无亲缘接口提供同名 default，实现类不重写编译报错；重写体内用 `A.super.method()` 显式委托。

### 4.4.2 Record 是什么？和常规 POJO/JavaBean 有何不同？

JDK16 正式的**透明数据载体**：

```java
public record Point(int x, int y) {
    public Point {                       // 紧凑型构造器：校验/规范化
        if (x < 0 || y < 0) throw new IllegalArgumentException();
    }
}
```

自动生成：`private final` 字段、全参构造器、访问器 `x()`/`y()`（**不是 getX**）、`equals/hashCode/toString` 基于全部组件。

限制与特性：隐式继承 `java.lang.Record` → **不能 extends 别的类**、本身隐式 final 不可被继承；**可以 implements 接口**；可有静态字段/方法；不能声明组件之外的实例字段；默认浅比较，含数组组件需自己重写 equals。

**适用**：DTO、方法多返回值、Map 复合 key、领域值对象。相比 Lombok @Value：record 是语言级不可变契约，天然线程安全、模式匹配可解构（见 4.4.7）。

### 4.4.3 Sealed Classes 密封类是什么？使用场景？

JDK17 正式：用 `permits` 声明**允许的子类型集合**：

```java
public sealed interface Shape permits Circle, Rectangle, Triangle {}
final  class Circle implements Shape {}
sealed class Triangle implements Shape permits Isoceles, Scalene {}
non-sealed class Rectangle implements Shape {}   // 放开继续扩展
```

直接子类必须标 final / sealed / non-sealed 之一。**价值**：把"继承体系开放无界"变成"类型集合封闭可枚举"——① 配合 switch 模式匹配做**穷尽性检查**（编译器知道全部子类型，缺 case 直接编译错，替代散落的 default 抛异常）；② API 设计控制扩展点；③ 与 record 组合成**代数数据类型（ADT）**：`sealed interface Expr permits Lit, Add`。

### 4.4.4 var、switch 表达式、文本块分别解决什么问题？

- **var（JDK10）**：局部变量类型**推断**——类型在编译期就确定，仍是强类型，只是不写全名。限制：必须有初始化；不能用于字段/方法参数/返回类型；Lambda 与 null 字面量不能做推断源。收益在 `var entry = map.entrySet().iterator()` 这类冗长泛型场景；"可读性优先"，链式返回值上慎用。
- **switch 表达式（JDK14 正式）**：箭头 `case A ->` 不穿透、可返回值（多行块用 `yield`）；配合枚举/密封类型**穷尽后免 default**，替代一大坨 if-else。
- **文本块（JDK15 正式）**：`""" ... """` 多行字符串保留换行与缩进（自动去掉公共前导空白），写 JSON/SQL/HTML 不再 `\n` 拼接；结尾 `\` 控制折行。

### 4.4.5 JDK9 模块化（JPMS）是什么？

`module-info.java` 声明模块：`requires`（依赖）、`exports`（哪些包对外可见）、`opens`（哪些包允许**深度反射**，Spring/Jackson 场景必须 opens）、`transitive`。目标：**强封装**内部 API（`sun.misc.Unsafe` 等受限）、精确依赖（替代 classpath 大杂烩的"JAR 地狱"）、支持裁剪运行时镜像（jlink）。

现实：业务应用多用"unnamed module 兼容模式"（classpath 照旧），模块化主要收益在 JDK 自身瘦身与库作者显式声明 API 边界；迁移到 module-path 常踩"反射被模块拦截"的坑（`--add-opens` 临时放行）。

### 4.4.6 JDK11 相比 JDK8 有哪些实用变化？

1. **String 新方法**：`isBlank()`（空/全空白）、`lines()`（按行 Stream）、`strip()/stripLeading/Trailing()`（按 Unicode 空白，trim 只认 ≤0x20）、`repeat(n)`。
2. **HttpClient 标准化**（java.net.http）：HTTP/1.1+2、同步/异步、Builder 风格——告别 HttpURLConnection/HttpClients 选型纠结。
3. **文件**：`Files.readString/writeString` 一行读写。
4. **集合**：`List.of/Set.of/Map.of` 不可变集合（配合 2.1.3 的选型）。
5. **Lambda 内 var**：`(var x, var y) -> ...`，主要为了加注解。
6. **Flight Recorder 开源**：低开销生产诊断（配合 JFR 事件分析）。
7. **Epsilon GC**（只分配不回收）、ZGC/Shenandoah 实验性引入。
8. 其他：`String.strip`、`Collectors.toUnmodifiableList`(10)、局部优化与字符编码升级（UTF-8 语言字符串）。

### 4.4.7 JDK21 有哪些重要新特性？

1. **虚拟线程 GA**（见 3.6.4）。
2. **Sequenced Collections**：新接口 `SequencedCollection/SequencedSet/SequencedMap` 统一"有确定顺序的集合"的首尾操作——`addFirst/addLast/getFirst/getLast/removeFirst/reversed()`；List、Deque、LinkedHashSet、LinkedHashMap 等归位（长期痛点：取 List 最后一个元素要 `get(size()-1)`）。
3. **Record Patterns**：`if (obj instanceof Point(int x, int y))` 解构取值；可嵌套。
4. **switch 模式匹配**：case 接类型模式 + `when` 守卫，配合 sealed 穷尽——instanceof 判类型链的终结者。
5. **String Templates**：`STR."Hello \{name}"`——JDK21/22 两次预览后官方中止了该形态（转向模板处理器方向），未进入 GA，生产不要用。
6. **Structured Concurrency 预览**：子任务作为父任务整体——任一失败自动取消兄弟任务，消灭"孤儿 Future"。
7. **Scoped Values**：不可变、按作用域共享的线程局部值，目标是替代 ThreadLocal 在虚拟线程下的场景（JDK21 孵化起步，仍在预览迭代中）。
8. **分代 ZGC（JDK21+ 实验，23 转正）**；`Thread.onSpinWait`、`synchronized pinning` 在后续版本持续改进。

### 4.4.8 G1 和 ZGC 的区别？（速览，深入见 2_jvm.md）

| 维度 | G1 | ZGC |
|------|----|-----|
| 停顿目标 | 可控（默认 200ms 目标） | 亚毫秒（<1ms，与堆大小几乎无关） |
| 核心机制 | Region 分区 + Remembered Set + SATB，优先回收垃圾多的 Region | **着色指针**（元数据藏进指针位）+ **读屏障**（访问时自愈） |
| 整理 | Mixed GC 逐 Region  evacuate | 并发标记+并发转移（用户线程同时搬） |
| 分代 | 有年轻/老年 Region | JDK21 起分代 ZGC |
| 吞吐 | 高 | 略低（屏障开销） |
| 适用 | 通用堆（几 G 到几十 G），默认推荐 | 超大堆 + 延迟敏感（交易、广告、游戏） |

> GC 体系（CMS、三色标记、卡表、调优参数）属 JVM 主题，见 2_jvm.md。

# 5. IO/NIO

## 5.1 IO 模型

### 5.1.1 BIO、NIO、AIO 的区别？

| 维度 | BIO | NIO | AIO |
|------|-----|-----|------|
| 模型 | 同步阻塞 | 同步非阻塞（多路复用） | 异步非阻塞 |
| 线程模型 | 1 连接 1 线程 | 1 线程 N 连接（Selector） | OS 完成回调/Future |
| 读写触发 | 阻塞等数据就绪 | 轮询就绪事件再处理 | 内核通知"数据已备好" |
| 编程复杂度 | 低 | 高 | 中 |
| 适用 | 连接少且固定（传统 Tomcat HTTP/1.1） | 连接多且短、高并发（主流） | 连接多且长、OS 支持好（Linux 上不成熟） |

一句话类比：BIO 是"服务员在厨房门口干等这道菜"，NIO 是"服务员轮询每桌谁点单了"，AIO 是"菜做好后厨师端到你桌上"。核心差别在**第二阶段（数据从内核缓冲区拷到用户空间）**：NIO 仍要自己调 read（同步），AIO 连拷贝都由内核完成后回调（异步）。Java 世界事实标准是 NIO + Netty；AIO（Windows IOCP 成熟、Linux epoll 上语义打折）用得少。

**五大 UNIX IO 模型**（面试进阶）：阻塞、非阻塞、IO 多路复用、信号驱动、异步——前三种都算"同步 IO"。select/poll/epoll：select 位图 1024 上限+每次全量拷贝遍历 O(n)；poll 无上限但仍 O(n)；epoll 事件回调 O(1)+ 就绪链表，JDK 在 Linux 用 epoll、macOS 用 kqueue。NIO 的 Selector 只支持**水平触发（LT）**——没读完的事件下次 select 还会通知，边缘触发（ET）必须一次读到 EAGAIN，Netty 的 epoll native transport 则支持 ET。

### 5.1.2 流（Stream）和通道（Channel）的区别？

- 流**单向**（InputStream/OutputStream 各走各），Channel **双向全双工**；
- 流面向字节直接读写、阻塞模型；Channel 必须搭配 Buffer，可设非阻塞并注册到 Selector；
- 流的抽象适合"管道式逐段处理"（包装装饰器：缓冲、字符集、对象），Channel+Buffer 适合"高性能随机/多路复用 IO"。
- FileChannel 额外能力：`transferTo/transferFrom`（sendfile 零拷贝，见 5.3.6）、文件锁、`map()` 内存映射。
- Channel 的 read 返回 -1 表示 EOF；close 时若仍有未写出数据要注意半关闭语义。

## 5.2 NIO 核心组件

### 5.2.1 NIO 的三大核心组件及协作流程？

- **Channel**：双向数据通道。FileChannel、SocketChannel（TCP 客户端）、ServerSocketChannel（TCP 服务端）、DatagramChannel（UDP）。只有 Socket 系可配非阻塞以注册 Selector。
- **Buffer**：数据容器（HeapByteBuffer/DirectByteBuffer），四个指针 `capacity/limit/position/mark`——写模式 limit=容量、position 推进；flip 后转读（limit=旧 position，position=0）。
- **Selector**：多路复用器。Channel `register(selector, OP_ACCEPT|OP_READ...)` 后，单线程 `select()` 阻塞等待，返回**就绪 SelectionKey 集合**（OP_ACCEPT/CONNECT/READ/WRITE），遍历 key → `key.channel()` → 处理 → 取消 key 防止重复。

```text
Channel ──注册──▶ Selector ◀─select()─ 单线程
   ▲                 │ 就绪 keys
   └──read/write───Buffer ◀──┘
```

这就是 Reactor 的雏形：一个线程服务上万连接，代价是编程复杂度与空轮询 bug 风险（见 5.2.3）。

### 5.2.2 Buffer 的 flip/clear/rewind/compact 区别？

以容量 10、已写 5 字节（position=5, limit=10）为例：

| 方法 | 效果 | 用途 |
|------|------|------|
| `flip()` | limit=position(5)，position=0 | **写完转读** |
| `rewind()` | position=0，limit 不变 | 重读当前可读数据 |
| `clear()` | limit=capacity，position=0 | **转写**；数据并未清除，只是"可写窗口"重开，旧数据会被覆盖 |
| `compact()` | 未读区间 [position,limit) 搬到头部，position=未读长度，limit=capacity | 读了一半继续写（协议解析常用） |

> **易错**：`clear` 后 limit 恢复容量，若之前只写了 5 字节又没读干净，可能读到残留的旧数据——这是"NIO 读到乱码"的经典成因。

### 5.2.3 什么是 NIO 的 Selector 空轮询 bug？Netty 如何规避？

Linux 下 epoll 的 `select()` 本应阻塞，但 JDK 有 bug（JDK-6403933）会在无事件时**立即返回 0**——应用层死循环 select，CPU 飙 100%。

Netty 对策：统计 `selectNow()` 连续返回 0 的次数，超过阈值（默认 512）就**重建 Selector** 并把所有 Channel 迁移过去；同时提供 `epoll` native transport（原生 C 库绕过 JDK NIO 实现，且支持 ET）。写原生 NIO 代码时要自查该问题（JDK 已修大部分场景，但"新 Selector 唤醒不及时"类问题仍偶发）。

### 5.2.4 直接内存（DirectBuffer）和堆内存 Buffer 的区别？

- `allocate()`：堆内存，受 GC 管理；IO 读写时 JDK 先把数据拷到**临时的本地直接内存**再交给内核——一次多余拷贝。
- `allocateDirect()`：堆外（Unsafe 分配），**不被 GC 移动、不占堆**，内核可 DMA 直达；创建销毁成本高（系统调用 + Cleaner 回收）。
- **选型**：长期存活、大块、走 socket/file IO 的缓冲用 direct（Netty 默认 PooledDirect）；小、短暂、不直接进 IO 系统的用 heap。
- 风险：DirectByteBuffer 本体在堆上、真正内存在堆外，**OutOfMemoryError: Direct buffer memory**（`-XX:MaxDirectMemorySize`）与 Netty 池化泄漏（`-Dio.netty.leakDetection.level`）要监控；堆内 GC 停顿也影响不到它——这是 Netty"减少 GC 压力"卖点的来源之一。

## 5.3 Reactor 与 Netty

### 5.3.1 Reactor 线程模型是什么？Netty 用的哪种？

Reactor 模式 = **一个（或多个）EventLoop 线程用 Selector 轮询就绪事件，分发给 Handler** 的事件驱动架构，三种变体：

1. **单 Reactor 单线程**：一个线程干完 accept+read+decode+process+send。Redis 6 前如此（内存操作快）；缺点：吃不到多核、一个慢 handler 卡死全局。
2. **单 Reactor 多线程**：Reactor 线程管 accept/read/send，业务丢给 Worker 线程池。业务重了 OK，但**读写仍在单 Reactor**，高并发下它成为瓶颈。
3. **主从 Reactor 多线程**：**MainReactor（Boss）只管 accept**，把连接注册给 **SubReactor（Worker，通常 2×CPU 个）做 read/encode/send**，业务再可转 Executor。Netty 的标准形态，也是 Kafka、Nginx 思路。

Netty 的 BossGroup/WorkerGroup 即 Main/Sub，每个 NioEventLoop = 一个线程 + 一个 Selector + 一组 Channel。

### 5.3.2 Netty 的核心组件和一条消息的处理流程？

**核心组件**：`Bootstrap/ServerBootstrap`（装配启动）、`EventLoopGroup→NioEventLoop`（线程+Selector+任务队列）、`Channel`（连接抽象）、`ChannelPipeline`（责任链）、`ChannelHandler/ChannelHandlerContext`（业务节点与传播入口）、`ChannelFuture/Promise`（异步结果）、`ByteBuf`（引用计数缓冲区，池化）。

**流程**：bind → Boss EventLoop 收到 OP_ACCEPT → 新 SocketChannel 注册到某个 Worker EventLoop（轮询取，此后该 Channel 的所有事件都由这一个线程处理）→ OP_READ 就绪 → ByteBuf 进 Pipeline → 入站链依次解码（LengthFieldBasedFrameDecoder→自定义 Decoder）→ 业务 Handler → `ctx.writeAndFlush(resp)` 走**出站链反向**编码 → Socket 发出。

**关键约束**：一个 Channel 固定绑一个 EventLoop 线程（**无锁串行化**），业务 Handler 里不要做阻塞操作——耗时逻辑 `executor.submit` 转业务线程池，否则拖死同 EventLoop 上的所有连接。

### 5.3.3 ChannelPipeline 的责任链模式是怎样的？

Pipeline 是 Handler 的双向链表：Head ←→ H1 ←→ H2 ←→ Tail。Handler 分**入站**（channelRegistered/channelActive/channelRead/exceptionCaught，从 Head→Tail 传播）与**出站**（bind/write/flush/close，从 Tail→Head 传播），一个 Handler 可兼做两类。

传播控制：`ctx.fireChannelRead()` 只触发**下一个**入站节点；`ctx.write()` 从**当前位置向前**出站；`ctx.channel().write()` 从 Tail 开始走完整链——漏 fire 即事件丢失（"数据被吞"的排错要点）。`@Sharable` 才能跨 Channel 复用，否则每 Channel 新建实例。Head 负责最终 flush 到 socket，Tail 兜底释放未消费消息。

### 5.3.4 TCP 粘包/拆包的原因和 Netty 的解决方案？

**根因**：TCP 是**面向字节流的协议、没有消息边界**；叠加：发送端 Nagle 合并小包（粘包）、单包超过 MSS/发送缓冲（拆包）、接收端不及时读取导致缓冲区堆积（粘包）。

**解法 = 应用层自定边界**，Netty 内置解码器：

| 解码器 | 边界方式 |
|--------|---------|
| LineBasedFrameDecoder | `\n` / `\r\n` |
| DelimiterBasedFrameDecoder | 自定义分隔符 |
| FixedLengthFrameDecoder | 定长 |
| **LengthFieldBasedFrameDecoder** | **消息头长度字段（最常用）**：lengthFieldOffset/lengthFieldLength/lengthAdjustment/initialBytesToStrip |

典型组合：`LengthFieldBasedFrameDecoder`（拆帧）+ `ProtobufDecoder`（反序列化）；发送端配 LengthFieldPrepender。UDP 无此问题（数据报有界），但不可靠。

### 5.3.5 什么是零拷贝？Java 和 Netty 分别怎么实现？

零拷贝 = **减少内核态/用户态之间的往返复制与上下文切换**，不是 0 次拷贝。

**传统读文件发网络**（4 次拷贝 4 次切换）：磁盘→DMA→内核页缓存→CPU 拷贝到用户缓冲→CPU 拷贝到 socket 缓冲→DMA 网卡。

- **sendfile（transferTo）**：磁盘→内核页缓存→**内核 socket 缓冲**（CPU 不经用户态，现代网卡支持 gather DMA 后仅 2 次拷贝 2 次切换）。`FileChannel.transferTo/transferFrom` 即它。
- **mmap**：文件映射进用户虚拟内存，读写即操作页缓存（省一次拷，仍有页缓存→socket 一次拷贝）。`FileChannel.map()` 返回 MappedByteBuffer——Kafka 段文件、Lucene 都用它。

**Netty 的四张牌**：DirectByteBuf（堆外，用户态与内核间不再中转堆内存）、FileRegion（包 transferTo 传文件）、CompositeByteBuf（**逻辑合并**多 Buffer，省 memcpy）、slice/duplicate（视图切分同样零复制）+ 池化复用。

> **面试追问**：Kafka 为什么快？——顺序写页缓存 + mmap + sendfile + 批量压缩 + 零消费端（page cache 直达 socket），全链零拷贝组合拳。

### 5.3.6 Netty 为什么高性能？

1. **IO 模型**：主从 Reactor + epoll，少量线程扛海量连接；
2. **线程模型**：每 Channel 固定绑定 EventLoop，**无锁串行化**，Pipeline 处理无竞争；
3. **内存**：池化 ByteBuf（jemalloc 思想）+ 堆外 Direct 内存 + 零拷贝设计（见 5.3.5），GC 压力小；
4. **协议**：内置粘包解帧、HTTP/2、WebSocket 等编解码；序列化可插 Protobuf；
5. **TCP 参数**：默认 TCP_NODELAY（关 Nagle）、SO_BACKLOG、SO_KEEPALIVE、可选 native epoll transport；
6. **健壮性**：规避 JDK 空轮询 bug（5.2.3）、完善的泄漏检测与优雅关闭；
7. **生态**：与 Dubbo/Spring WebFlux/RocketMQ/RPC 框架互操作成熟。

## 5.4 序列化方案

### 5.4.1 Java 原生序列化有什么问题？有哪些替代方案？

**问题**：①**跨语言不通**（只有 JVM 认）；②**码流大**——携带类全名、serialVersionUID、字段元数据；③**慢**——反射遍历+流式写入；④版本兼容脆（serialVersionUID 管理麻烦，见 1.9.1）；⑤**安全风险**——readObject 反序列化链是 RCE 重灾区（Commons-Collections  gadgets）。

**替代方案对比**：

| 方案 | 跨语言 | 体积 | 速度 | 特点 |
|------|--------|------|------|------|
| Protobuf | ✅（IDL） | 极小（varint/TLV） | 极高（预编译无反射） | gRPC/Dubbo 推荐；需 .proto 与生成代码 |
| JSON（Jackson 等） | ✅ | 大（文本） | 中 | 可读性好、调试友好，Web 事实标准 |
| Kryo | ❌ JVM | 小 | 高 | 需注册类、线程不安全（配池/ThreadLocal） |
| Hessian | ✅ | 中小 | 中高 | 二进制，Dubbo 早期默认 |
| FST | ❌ | 小 | 高 | 兼容 Java 序列化 API 的加速替代 |
| Avro | ✅（schema） | 极小 | 高 | 大数据生态（Kafka/Hive） |

**选型**：跨语言 RPC 用 Protobuf；对外 API 用 JSON；纯 JVM 内部缓存/临时对象传输 Kryo；遗留 Dubbo 可 Hessian2。无论哪种，**协议字段演进规则**（只增不删、默认值兼容）比选库更重要。
