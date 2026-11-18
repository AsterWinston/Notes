
# 1. 七个阶段

## 1.1 阶段总图

```text
加载 → 验证 → 准备 → 解析 → 初始化 → 使用 → 卸载
│        │                 │        │
│<──────连接─────────>│        │
│        │                 │        │
│<────────类加载过程────────────────>│
```

## 1.2 顺序哪些固定、哪些不固定

- **加载、验证、准备、初始化、卸载** 5 个阶段顺序确定。
- **解析**可在初始化之前或之后开始——不固定。
- "连接（Linking）"是验证+准备+解析三个子阶段的合称，规范没有规定连接内部必须串行到何种程度。

## 1.3 为什么解析可以乱序

- 解析 = 把符号引用换成直接引用。若强制在初始化前解析完一切，动态绑定就玩不转：

```java
interface Pay { void pay(); }
Pay p = loadAtRuntime();   // 实现类可能初始化之后才加载出来
p.pay();                   // invokeinterface 要等运行期才知道目标
```

- JVM 层的多态（vtable/itable 查找、`invokedynamic`）都要求"引用推迟确定"，所以规范只保证：**每次使用某符号引用前，它一定已被解析**——何时解析实现自由（HotSpot 策略见 5.3）。

# 2. 加载（Loading）

## 2.1 加载做的三件事

### 2.1.1 ① 通过全限定名获取二进制字节流

通过全限定名获取类的二进制字节流——来源不限（见 2.2）。

### 2.1.2 ② 转换为运行时数据结构

将字节流代表的静态存储结构转化为方法区的运行时数据结构（HotSpot：`Klass` 元数据 + 堆上镜像）。

### 2.1.3 ③ 生成 Class 对象

在堆中生成 `java.lang.Class` 对象，作为访问入口。

- 三件事完成后，类的"存在形式"从磁盘字节变成内存中的元数据 + Class 镜像，但**尚不可执行**（还没验证/初始化）。

## 2.2 字节流的获取方式

| 来源 | 案例 |
|------|------|
| 磁盘 | `.class` 文件、JAR/WAR 内读取（**最常见**） |
| 网络 | Applet（历史）、RMI 按需下载类、Java Web Start |
| 运行时计算生成 | 动态代理生成 `$Proxy0`、CGLIB 生成子类、Lambda 的隐藏类 |
| 数据库读取 | 把类字节码存 DB，按名捞取 |
| 动态编译 | JSP → Servlet class；IDE 热编译 |
| 加密文件 | 自定义类加载器解密后 `defineClass`（见 [4_classloader](4_classloader.md) 5.5） |

## 2.3 数组类 vs 非数组类

- 非数组类由类加载器加载；**数组类由 JVM 直接创建**，不经过任何类加载器。
- JVM 创建 `[Lcom/foo/Bar;` 时的特殊规则：

```text
① 元素类型 com.foo.Bar 已加载 → 直接造数组类，复用其类加载器
② 未加载    → 递归走加载流程先加载元素类型（此时才用类加载器）
```

- 所以 `Bar[]` 的 Class 对象名是 `[Lcom/foo/Bar;`，其类加载器与 Bar 相同；`int[]` 则是 `Class.forName("[I")` 合法但 `[int` 非法。

## 2.4 Class 对象放哪里

- 是 `java.lang.Class` 的**普通实例，存放在堆中**（HotSpot 称 klassOop→镜像 oop），并非方法区里的特殊结构。
- 方法区（元空间）里的是 `Klass` 元数据；Class 对象是它的**Java 侧句柄**，二者互相引用。
- 生命周期：Class 对象可达 → 类不被卸载（见第 8 节）。

# 3. 验证（Verification）

确保 Class 文件字节流符合 JVM 规范，不会危害 JVM 安全。**这是安全底线，不是可选项**——恶意字节码完全可以手工拼出来。

## 3.1 四个验证阶段

### 3.1.1 文件格式验证
- 检查点清单：
  - 魔数 `CAFEBABE`；主次版本在支持范围内
  - 常量池每项 tag 合法、无指向不存在的常量池项
  - 所有 u1/u2/u4 字段取值范围；文件长度与 `attribute_length` 一致
  - 不通过 → `VerifyError` / `ClassFormatError`

### 3.1.2 元数据（语义）验证
- 检查点清单：
  - 除了 Object，是否有父类
  - 声明 `final` 的类不能被继承；抽象类不能被实例化（此时查方法表标志）
  - 抽象方法是否被全部实现
  - 字段/方法名是否与父类冲突且非法（如静态方法隐藏实例方法且签名不兼容）

### 3.1.3 字节码验证
- 检查点清单（最贵的一段）：
  - 跳转指令的目标必须是某条指令的起始位置，不能跳进指令中间/方法外
  - 操作数栈与局部变量表上的类型在每条指令处匹配：`iadd` 顶上两个必须是 int
  - 控制流可达性；方法结尾必须 return/athrow 到达
- 演进：JDK 6（class 50）引入 **StackMapTable** 让编译器预先声明分支处类型，校验从"数据流分析"退化为"查表验证"；**JDK 7 起旧类型推断验证器被移除**，StackMapTable 成为唯一依据（见 [2_class_file_structure](2_class_file_structure.md) 8.1）。

### 3.1.4 符号引用验证
- 发生在**解析阶段之前**（对懒解析而言是首次使用时）：
  - 引用的类是否存在且可访问
  - 字段/方法名+描述符能否找到目标
  - 访问权限（private/包级）是否允许

## 3.2 关闭验证：参数与风险

```bash
-Xverify:none        # / -noverify：跳过字节码验证（只剩格式校验）
                     # JDK 13 起官方废弃，启动时打印警告，未来版本将失效
                     # JDK 8u 后期更新起，Bootstrap 类路径的类无视该参数、始终验证
-XX:-BytecodeVerificationRemote   # 细粒度：仅关非启动路径类的字节码验证
-XX:-BytecodeVerificationLocal    # 连同启动路径一起关（危险）
```

| 诱惑 | 真相 |
|------|------|
| "启动快一点" | 只省字节码验证，收益在多数应用里 <100ms |
| "老项目加速" | 一旦混入构造的坏字节码（如越界跳转），从"抛异常"变成"踩坏内存/执行任意代码" |
- 现代替代：默认已开启 **CDS/AppCDS** 缓存已加载类，启动加速不必拿安全换（见 [20_jvm_tools](20_jvm_tools.md)）。

# 4. 准备（Preparation）

## 4.1 分配内存 + 零值

- 为**类变量（static）**分配内存并赋予**系统默认零值**；`= 123` 这类赋值属于初始化阶段。
- 注意：`static final` 的编译期常量在准备阶段直接赋予最终值（见 4.3）。

```java
public static int value = 123;         // 准备阶段 value = 0，初始化阶段才赋 123
public static final int CONST = 123;   // 编译时就确定，准备阶段 CONST = 123
```

## 4.2 各类型零值表

| 类型 | 零值 | 类型 | 零值 |
|------|------|------|------|
| int | 0 | boolean | false |
| long | 0L | char | '\u0000' |
| float | 0.0f | short | (short)0 |
| double | 0.0d | byte | (byte)0 |
| 引用类型 | null | | |

## 4.3 ConstantValue 属性的捷径

- javac 对 `static final` 基本类型/String 常量生成 `ConstantValue` 属性 → 准备阶段 JVM 读属性直接写终值，**不执行任何字节码**。
- 连带效应（面试常考）：常量在编译期被**内联进调用方的常量池**，改了 `Const.V` 重编 Const 而不重编引用方，引用方看到的仍是旧值。
- `static final int x = new Random().nextInt();` 没有 ConstantValue（非编译期常量），仍走初始化阶段。

## 4.4 类变量的存储位置：版本演进

| 时期 | 类变量在哪 |
|------|-----------|
| JDK ≤ 6 | 方法区 = **永久代**（GC 难调优） |
| JDK 7 | 静态变量数据随 Class 对象移入**堆**（JSR 292 配套改造） |
| JDK 8+ | 永久代彻底移除：类**元数据**进本地内存的元空间；类**变量**随镜像在堆，GC 可达即可回收 |

```text
JDK 8 视角:
  Metaspace(native): Klass 结构、方法字节码、运行时常量池
  Heap:              Class 镜像对象 ← static 字段数据挂在这
```
- 详见 [7_method_area](7_method_area.md)。

# 5. 解析（Resolution）

## 5.1 符号引用 vs 直接引用

| 符号引用 | 直接引用 |
|----------|----------|
| 一组符号描述目标 | 指向目标的指针、偏移量或句柄 |
| 与 VM 实现无关 | 与 VM 内存布局直接相关 |

## 5.2 四类解析目标

- **类或接口解析**：CONSTANT_Class → 找到/加载目标类，指向其元数据。
- **字段解析**：从本类往上沿父类链找 name+descriptor 匹配的字段 → vtable 外直接偏移。
- **方法解析（类方法）**：沿继承链找签名匹配的方法 → 得到入口地址/方法块偏移。
- **接口方法解析**：不绑定具体实现，只确认接口内存在该方法；调用时靠 itable 查找定位。

## 5.3 急切解析 vs 懒解析（HotSpot 的选择）

| 引用类型 | HotSpot 时机 |
|----------|--------------|
| 类/接口引用 | **偏急切**：验证阶段就要比对目标类型（StackMapTable 逼迫），事实上提前 |
| 字段/方法引用 | **懒解析**：字节码首次执行到时才解析，结果缓存在运行时常量池 |
| `invokedynamic` | 更晚：首次执行触发**引导方法**返回 CallSite，之后缓存复用 |

- 懒解析的代价：首次执行多一次查表；换来的是"类可以永远不被用到就不加载"（减小启动负担，配合动态绑定的语义）。

# 6. 初始化（Initialization）

## 6.1 主动引用：触发初始化的场景

执行类构造器 `<clinit>()` 方法（见第 7 节）。JVMS 列举的主动引用：

| # | 场景 | 备注 |
|---|------|------|
| 1 | `new` 实例化 / 读写类静态字段 / 调用类静态方法 | getstatic/putstatic/invokestatic 触发 |
| 2 | 反射 `Class.forName("X")` | 默认触发；`forName(name, false, loader)` 可只加载不初始化 |
| 3 | 初始化子类时，先初始化父类 | 递归到顶 |
| 4 | 启动类（main 所在类） | VM 启动流程保证 |
| 5 | `MethodHandle`/动态语言（JSR 292）解析到静态字段/方法 | 同 1 |
| 6 | 接口声明了 default 方法，其实现类初始化时 | 接口本身也需先初始化（JDK 8 起） |
| 7 | 接口静态字段被使用（非常量） | 只初始化声明字段的接口 |

## 6.2 被动引用：看起来该触发却不触发

### 6.2.1 五个反直觉代码例

```java
// 1. 访问父类静态字段，不会初始化子类
class Parent { static int x = 1; }
class Child extends Parent { static int y = 2; }
int v = Child.x;  // 只初始化 Parent（字节码里 owner 就是 Parent）

// 2. 通过数组定义来引用类
Parent[] arr = new Parent[10];  // 创建的是 Parent[] 类，不触发 Parent 初始化

// 3. 访问编译期常量
class Const { static final int V = 1; }
int v = Const.V;  // 编译期常量传播，V 已内联进调用方，不初始化 Const

// 4. 实现类初始化 ≠ 接口初始化
interface I { int N = 100; }            // 常量接口
class Impl implements I { }
new Impl();                              // 不会触发 I 的 <clinit>（接口若无 default/非常量字段甚至可能永不初始化）

// 5. 子类静态方法内部访问父类字段
class P { static int a = 1; static int fa(){ return a; } }
class C extends P { static int b = 2; }
C 初始化会先触发 P 初始化；但只调 P.fa() 触发的是 P 而非 C
```

### 6.2.2 共同判据

- 判据永远是：**这条字节码的 owner 是谁，就初始化谁**——"通过子类引用"只是源码表象（`Child.x` 编译后 owner = `Parent`）。

## 6.3 初始化顺序总规则

```text
父类静态变量/静态块（按文本顺序）
  → 子类静态变量/静态块（按文本顺序）
    → 父类实例初始化块/实例字段（按文本顺序） → 父类构造器
      → 子类实例初始化块/实例字段（按文本顺序） → 子类构造器
```

- 实例初始化块与字段初始化被编译进 `<init>`，插在 `super(...)` **之后**、本类构造器体**之前**，保持源码文本顺序。

# 7. `<clinit>()` 类构造器

## 7.1 字节码示例

```java
public class Example {
    static int a = 1;
    static int b;

    static {
        b = a * 2;
        System.out.println("clinit executed");
    }
}
```

编译后生成 `<clinit>` 等效于：
```text
static {};
  descriptor: ()V
  Code:
     0: iconst_1
     1: putstatic   #2   // a = 1
     4: getstatic   #2
     7: iconst_2
     8: imul
     9: putstatic   #3   // b = a * 2
    12: getstatic   #4   // System.out
    15: ldc         #5   // "clinit executed"
    17: invokevirtual #6
    20: return
```

- 静态块写在赋值语句前后不影响"按文本顺序合并"这一铁律。

## 7.2 特性清单

- 无参数无返回值（描述符 `()V`），JVM 内部调用，程序无法直接 invoke。
- 父类 `<clinit>` 先于子类；**接口的 `<clinit>` 独立**——只有用到其非常量静态字段或 default 方法链路才触发，且实现类初始化不强制触发无 default 的接口初始化。
- 类没有静态变量赋值也没有静态块 → **不生成** `<clinit>`（javap 看不到 `static {};`）。
- 每个类只会成功执行一次；抛异常则该类进入"初始化失败"终态。

## 7.3 线程安全：JVM 锁 Class 对象

- 规范强制 `<clinit>` 对**每个类恰好执行一次且线程安全**；HotSpot 实现 = 对该类的 Class 镜像对象加初始化锁（历史文档称 "class initialization lock"，等价于持有初始化权限）。
- 因此静态块单例无需 double-check：

```java
public class Singleton {
    private static final Singleton I = new Singleton(); // clinit 线程安全 → 天然只 new 一次
    public static Singleton get() { return I; }
}
```

## 7.4 死锁案例：两个 `<clinit>` 互触发

### 7.4.1 最小复现代码

```java
class A { static { Thread.sleep(0); B.touch(); } static void touch() {} }
class B { static { A.touch(); } static void touch() {} }
// 线程1: new A() … 线程2: new B()  几乎同时发生
```

### 7.4.2 时序与观测

```text
时间线:
T1: 获得 A 的初始化锁 ──── clinit 里调 B.touch() ──▶ 等 B 的锁…
T2:                     获得 B 的初始化锁 ── clinit 里调 A.touch() ──▶ 等 A 的锁…
                                    ★ 环形等待，双方都永远起不来
```

- 关键反直觉点：`synchronized` 保护不了初始化锁的环——这是 **JVM 级死锁**，业务代码里互相触发的 static 块就是导火索。
- 观测：jstack 会报 `Found one Java-level deadlock`，栈帧显示 `waiting to lock <0x…> (a java.lang.Class for demo.A)`；HSDB/类转储里能看到 `Initializing class 'demo.A' locked by thread '<T1>'` 字样。

## 7.5 耗时 clinit 的连锁阻塞

- 同线程二次进入不阻塞（锁可重入），**其他线程**访问该类会排队等 clinit 完成：

```text
jstack:
"pool-1-thread-3" ... WAITING for monitor entry
   - waiting to lock <0x00000007156081a8> (a java.lang.Class for com.foo.HttpClient)
```

- 典型病灶：clinit 里做网络请求/DNS/读大资源/加载本地库。修复方向见 10.2。

# 8. 类的卸载

## 8.1 卸载的三个苛刻条件

- 该类**所有实例**均已被回收；
- 加载它的 **ClassLoader 已被回收**；
- 类的 `Class` 对象**没有任何地方引用**（不能通过反射句柄等可达）。
- 且只有 Full GC / 对应分代回收触及该区域时才执行。三条同时满足的概率极低 → "永久代几乎不回收"的观感由此而来。

## 8.2 谁能被卸载

- 由启动类加载器、平台/扩展类加载器、应用类加载器加载的类，其加载器伴随 VM 全程 → **整个 JVM 生命周期中几乎不会被卸载**。
- **只有自定义类加载器加载的类才可能被卸载**（丢掉引用即可回收）。

## 8.3 实际应用：热部署

- Tomcat 重新加载应用 = 丢弃整个 WebappClassLoader 建新实例 → 旧应用全部类可达性断裂 → 卸载（原理链见 [4_classloader](4_classloader.md) 5.4）。

## 8.4 观测参数（版本差异）

```bash
# JDK 8（develop 参数，部分构建需 -XX:+UnlockDiagnosticVMOptions）
java -XX:+TraceClassLoading -XX:+TraceClassUnloading MyApp

# JDK 9+ 统一日志（旧 Trace 参数已移除）
java -Xlog:class+load=info MyApp        # [info][class,load] Hello source: file:/...
java -Xlog:class+unload=info MyApp      # [info][class,unload] unloading class ...
java -Xlog:class+load,class+unload=info:file=cl.log MyApp
```

# 9. 实战：观察类加载顺序

## 9.1 静态序列题

```java
class Parent {
    public static int A = 1;
    static { System.out.println("Parent static block, A=" + A); }
}

class Child extends Parent {
    public static int B = A;
    static { System.out.println("Child static block, B=" + B); }

    public static void main(String[] args) {
        System.out.println("main start");
        new Child();
    }
}
```

输出：
```text
Parent static block, A=1
Child static block, B=1
main start
```

- 执行 `Child.main` 前必须先初始化 Child → 递归先初始化 Parent。
- `<clinit>()` 只执行一次；`new Child()` 不再触发静态块，只走 `<init>`。

## 9.2 综合题：实例初始化块 + 字段 + 构造器

### 9.2.1 题目代码

```java
class P {
    static { System.out.println("1 父静态块"); }
    { System.out.println("3 父实例块"); }
    P() { System.out.println("4 父构造器"); }
}
public class C extends P {
    static { System.out.println("2 子静态块"); }
    { System.out.println("5 子实例块A"); }
    int v = print("6 子实例字段赋值");
    { System.out.println("7 子实例块B"); }
    C() { System.out.println("8 子构造器"); }
    static String print(String s) { System.out.println(s); return s; }
    public static void main(String[] args) { new C(); }
}
```

### 9.2.2 输出与逐步解析

```text
1 父静态块          ← 执行 C.main 需先初始化 C → 先跑 P.<clinit>，再跑 C.<clinit>
2 子静态块          ← C 没有静态字段，静态块单独存在；有字段时按文本顺序合并进 <clinit>
3 父实例块          ← new C() → C.<init>: 先 super() → P.<init>: 实例块→P 构造器
4 父构造器
5 子实例块A         ← 回到 C.<init>：按文本顺序 块A → 字段v → 块B
6 子实例字段赋值
7 子实例块B
8 子构造器          ← 最后才是构造器自身方法体
```

- 两个易错点：① 实例块与字段的先后**只看文本顺序**，不看"块/字段"身份；② 这一切都发生在 `super()` 之后、本类构造器体之前（`this(...)` 委托构造则规则另算，见下）。
- 扩展：显式 `super(...)`/`this(...)` 必须在 `<init>` 最前；javac 拒绝两者同时出现。

# 10. 初始化中的线程安全问题

## 10.1 静态块单例（保留示例）

```java
public class Singleton {
    private static Singleton instance;

    static {
        // JVM 保证 clinit 的线程安全，这里不会产生多实例
        instance = new Singleton();
    }

    public static Singleton getInstance() {
        return instance;
    }
}
```

- JVM 通过初始化锁保证 `<clinit>` 的线程安全；调用方拿到的 instance 是"完全构造好"的（final 字段冻结语义 + 监视器释放，双重保险，见 [18_jmm](18_jmm.md) Safe Publication）。

## 10.2 真实场景：枚举单例 vs 静态块单例

| 维度 | 静态块/Holder 单例 | 枚举单例 |
|------|--------------------|----------|
| 线程安全来源 | clinit 初始化锁 | 同样是 clinit（`static final INSTANCE = new …`）|
| 序列化 | 需自行处理 readResolve | 天然防反射/反序列化多实例 |
| 风险 | clinit 里做耗时/有环引用 → 7.4/7.5 的死锁与长阻塞 | 构造器同样在 clinit 执行，别在里面联网 |
- 实践守则：
  - clinit 里**只做赋值，不做 IO/加锁/回调外部类**；
  - 可能失败或耗时的初始化改为显式 `initialize()` 方法（失败可重试，不锁死其他线程）；
  - 两类的 clinit 存在互相引用时，保证只有一个入口方向触发（破环）。

## 10.3 排障入口

- 类初始化相关异常速查：

| 异常 | 含义 | 动作 |
|------|------|------|
| `ExceptionInInitializerError` | `<clinit>` 抛异常（附 cause） | 看 cause，修静态初始化本身 |
| `NoClassDefFoundError: Could not initialize class X` | **再次**访问已初始化失败的 X | 往前翻日志找第一次的 ExceptionInInitializerError |
| 线程全卡 `waiting to lock ... (a java.lang.Class for X)` | clinit 慢/死锁 | jstack 找持有初始化锁的线程 |
- 系统排查流程见 [22_troubleshooting](22_troubleshooting.md)。
