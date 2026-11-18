# 1. 虚拟机栈概述

## 1.1 定位

- 每个**线程**创建时分配，**线程私有**，随线程生灭——栈内数据天然不需要并发控制。
- 每个方法调用创建一个**栈帧**（Stack Frame），方法的调用-返回 = 栈帧的入栈-出栈，LIFO。
- 大小由 `-Xss` 控制，与直接内存/堆共同瓜分进程物理内存（见 [5_runtime_data_area](5_runtime_data_area.md) 第 4 节）。

```text
线程的栈内存：

栈顶 ┌──────────────────┐ ← 当前执行的方法
     │    栈帧 (bar)     │
     ├──────────────────┤
     │    栈帧 (foo)     │
     ├──────────────────┤
     │    栈帧 (main)    │
栈底 └──────────────────┘
```

## 1.2 与本地方法栈的关系

HotSpot 把两者合并为同一个栈：Java 帧与 native 帧混排，`-Xss` 同时约束二者（详见 [5_runtime_data_area](5_runtime_data_area.md) 第 5 节）。

# 2. 栈帧的内部结构

## 2.1 五部分总图

```text
┌──────────────────────────┐
│       栈帧 (Stack Frame)  │
├──────────────────────────┤
│   局部变量表 (Local Vars) │  ← 方法参数 + 局部变量 + this
├──────────────────────────┤
│   操作数栈 (Operand Stack)│  ← 字节码运算的工作台/参数暂存
├──────────────────────────┤
│   动态链接 (Dynamic Link) │  ← 指向运行时常量池中本方法的引用
├──────────────────────────┤
│   方法返回地址 (Return)   │  ← 正常/异常出口时恢复到调用者
├──────────────────────────┤
│   附加信息                │  ← 方法版本(常量池指针)、调试信息等
└──────────────────────────┘
```

## 2.2 栈帧在内存中的布局（概念示意）

```text
高地址 ┌───────────────────────────────┐
       │ 调用者帧 (foo)                 │
       ├───────────────────────────────┤ ← 帧底：先分配
       │ 返回地址 / 帧头(方法元数据)     │
       │ 动态链接 → 本方法的常量池项     │
       │ 局部变量表 slot0..slotN        │
       │            ↑                  │
       │          ESP 操作数栈从此向低   │ ← 帧顶：新帧往低地址方向生长
       │            地址方向生长/收缩    │
低地址 └───────────────────────────────┘   新帧 bar() 压在这下面
```

> 说明：这是规范/教材视角的概念图。HotSpot 实现里局部变量区与操作数区共享一块预分配空间、方向与平台 ABI 相关，不同 CPU/不同帧类型（解释帧/编译帧）细节不同，笔记不必纠结字节级方向，抓住"每帧定长、由 max_locals 与 max_stack 决定"即可。

## 2.3 帧大小在编译期确定

```text
Class 文件 Code 属性：
  max_stack  = 2    ← 操作数栈最大深度（javac 算好）
  max_locals = 5    ← 局部变量表 slot 数（javac 算好）
帧大小 ≈ (max_locals + max_stack) × 4B + 固定开销
```

- 方法越复杂（局部变量多、表达式嵌套深）→ 帧越大 → 同样 `-Xss` 下递归越浅。
- 递归 = 同名方法的**多个帧**沿栈排下去，这就是栈溢出的全部机制（见 10.2）。

# 3. 局部变量表（Local Variables）

## 3.1 Slot 数组与索引规则

- 以**变量槽**（slot）为最小单位，32 位 JVM 每槽 4B；64 位下引用槽靠压缩指针也保持 4B（见 [9_object_layout](9_object_layout.md)）。
- 非静态方法 slot 0 存 `this`；参数依次排列。
- `long`/`double` 各占 **2 个 slot**，其余基本类型/引用各 1 个。
- 容量上限：Code 属性计数限制，slot 数最大 65535（javac 会直接报错，不必担心）。

```java
public int add(int a, int b) {  // this(0), a(1), b(2)
    int sum = a + b;             // sum(3)
    return sum;
}
```

```text
Slot 0: this     (引用)
Slot 1: a        (int)
Slot 2: b        (int)
Slot 3: sum      (int)
Slot 4: (未用)
```

## 3.2 Slot 复用

作用域结束、且后续变量需要的槽位会**覆盖复用**：

```java
public void test() {
    {
        int a = 1;       // Slot 1 (0是this)
    }                    // a 的 Scope 结束
    int b = 2;           // Slot 1 被复用
}
```

## 3.3 Slot 复用与对象可达性：澄清一个面试谣言

谣言版本："局部变量离开作用域后引用槽没被覆盖，对象就不能回收，所以必须 `dummy=0` 触发复用或手动置 null。"

真相分执行模式和场景：

### 3.3.1 四种执行模式对照表

| 模式 | 不覆盖 slot | 结论 |
|------|-------------|------|
| `-Xint` 纯解释 | 离开作用域但帧未结束 | **不能回收**——解释器的 GC 映射保守地把整张局部变量表都当根 |
| `-Xint` + 后续变量复用槽 | — | 能回收（旧引用被覆盖） |
| 默认 JIT（C1/C2） | 不覆盖 | **能回收**——安全点处按当前字节码位置生成 **OopMap**，结合变量活跃性（源码中 OAL，Object Allocation in Lifetime 信息）精确知道哪些槽的引用还"活着" |
| 任何模式 + 远程 debug 挂接 | 不覆盖 | 退化：调试信息要求方法结束前局部变量都可查，活跃性被延长 |

### 3.3.2 验证代码与工程结论

```java
public void method() {
    byte[] big = new byte[100 * 1024 * 1024];   // 100MB
    // big 离开作用域后：JIT 模式已可回收；解释器/debug 模式仍被当根
    System.gc();                                 // 观察分配是否发生在别处，验证回收
}
```

工程结论：生产环境（JIT）不需要为了"帮 GC"写置 null/复用槽的怪代码；这段谣言的价值在于引出 **OopMap / 安全点精确性**（详见 [11_gc_algorithm](11_gc_algorithm.md)）。

# 4. 操作数栈（Operand Stack）

## 4.1 字节码的工作台

### 4.1.1 load-compute-store 闭环

后进先出，容量 = `max_stack`（编译期确定）。所有计算 = "从局部变量表 load 进栈 → 指令弹栈运算 → 结果压栈 → store 回表"。

### 4.1.2 `(a+b)*c` 逐条演算

示例 `(a + b) * c`（实例方法，a/b/c 在 slot1/2/3，结果存 slot4）：

```text
指令              栈变化（左为栈顶？否，右为栈顶）        max_stack 峰值
iload_1     a:    [a]                                  1
iload_2     b:    [a, b]                               2
iadd              [(a+b)]                              1
iload_3     c:    [(a+b), c]                           2
imul              [(a+b)*c]                            1
istore 4          []        → 结果进 slot4
```

## 4.2 第二大作用：准备方法调用参数

```java
obj.doSomething(x, y);
// 字节码逻辑：
aload_1          // obj 压栈（接收者最先，参数依次压）
iload_2          // x 压栈
iload_3          // y 压栈
invokevirtual    // 弹出 obj, x, y 定位并调用；返回值（若有）压栈
```

返回值的交接：被调方 `xreturn` 把结果压到**栈顶** → 帧弹出后成为**调用者帧操作数栈顶** → 调用者用 `istore/astore` 收进自己的局部变量表（见第 6 节）。

## 4.3 栈式 vs 寄存器式

### 4.3.1 两种虚拟机对比表

| 维度 | 栈式（JVM） | 寄存器式（如 Lua、Dalvik） |
|------|-------------|---------------------------|
| 指令表达 | 操作数在隐式栈顶，指令短 | 指令带寄存器编号，单条信息多 |
| 指令条数 | 多（load/store 频繁） | 少 |
| 可移植性 | **强**：不假设目标机寄存器数量/名字 | 依赖对虚拟寄存器的映射 |
| JIT 友好度 | 编译时消掉冗余 load/store，栈模型不损失最终机器码质量 | — |

### 4.3.2 JVM 选栈式的理由

JVM 选栈式的核心理由：**与硬件解耦的可移植性**——"一次编译到处运行"，代价交给 JIT 在目标机上抹平。

# 5. 动态链接（Dynamic Linking）

## 5.1 机制

每个栈帧持有一个指向**运行时常量池中本方法符号引用**的指针。调用点要的不是 Class 文件里的 `CONSTANT_Methodref` 文本，而是能真正跳转的入口地址：

```text
栈帧 ────→ 运行时常量池
              ├── #5 = Methodref  com.example.Foo.add:(II)I
              │                        ↓ 解析
              │                   直接引用 = 方法入口 / vtable 槽
              └── ...
```

## 5.2 解析时机：静态解析 vs 动态连接

| 阶段 | 处理内容 | 对应指令 |
|------|----------|----------|
| 类加载解析期（可延后到首次使用） | 编译期即可确定版本、不受重写影响的引用：static / private / 构造器 / final 语义方法、字面量 | invokestatic / invokespecial |
| 运行期首次执行 | 依赖实际类型的引用：虚方法、接口方法 | invokevirtual / invokeinterface |
| 运行期引导方法决定 | 完全动态：调用点与目标的绑定推迟到 BSM 执行 | invokedynamic（见第 9 节） |

链接阶段本身发生在类加载的链接环节，时序详见 [3_class_lifecycle](3_class_lifecycle.md)；Class 文件里符号引用的存储格式见 [2_class_file_structure](2_class_file_structure.md)。

# 6. 方法返回地址与方法出口

## 6.1 两种出口

| 出口 | 触发 | 动作 |
|------|------|------|
| 正常出口 | `return / ireturn / lreturn / freturn / dreturn / areturn` | 返回值（若有）压到栈顶 → 帧弹出 → 值落到**调用者操作数栈顶** → 恢复调用者局部变量表与操作数栈状态 |
| 异常出口 | 抛出的异常在本帧找不到处理器 | **操作数栈清空**，局部变量表保留 → 沿调用链向上逐帧查找异常表 → 找到则把异常对象压入该帧栈顶跳 handler；整链找不到 → 线程栈全部弹出 |

## 6.2 异常表查找

异常处理逻辑不是指令，而是 Code 属性的 `exception_table`：`from/to`（受保护字节码区间）、`target`（handler 入口）、`type`（异常类索引或空=finally 兜底）。运行时拿当前 PC 在表里线性匹配 → 详见 [2_class_file_structure](2_class_file_structure.md)。

## 6.3 PC 的恢复

```text
main() 调用 foo():
  caller: 0: invokestatic foo   ← 返回地址(下一条 0x??)随 foo 的帧一起保存
  foo 帧内: ... 执行 ...
  foo: return                   ← 帧弹出，main 的 PC 恢复为"invokestatic 的下一条"
```

PC 只负责"回到哪一行"；"回到什么状态"由被弹出的帧归还（返回值、局部变量、操作数栈都在各自帧里管理）。

# 7. 方法调用五指令

## 7.1 五指令语义与解析时机

### 7.1.1 五指令对照表

| 指令 | 用于什么方法 | 分派时机 | 备注 |
|------|--------------|----------|------|
| `invokestatic` | 静态方法、接口静态方法 | 编译期定死（静态解析） | 无接收者压栈 |
| `invokespecial` | 构造器 `<init>`、**private** 方法、`super.xxx` 父类调用 | 编译期定死 | 名字里的 special = "不参与动态分派的实例方法" |
| `invokevirtual` | 普通实例方法（虚方法，含 final：final 只是 JIT 不再查表） | 运行期按实际类型 | 接收者最先压栈 |
| `invokeinterface` | 接口方法 | 运行期 | 不知道目标在 vtable 哪个槽 → 查 itable，开销更高（JIT 有内联缓存补救） |
| `invokedynamic` | 动态调用点：lambda、动态语言、字符串拼接(JEP 280) | 运行期由引导方法绑定 | 见第 9 节 |

### 7.1.2 静态解析 vs 动态分派对照示例

静态解析 vs 动态分派的最小对照（写在 Father 类内部，private 访问才合法）：

```java
class Father {
    static void staticMethod() {}
    private void privateHelper() {}
    void virtualMethod() {}

    void demo(Father obj) {            // obj 实际类型是 Child
        staticMethod();                // invokestatic  → 目标编译期写死
        obj.privateHelper();           // invokespecial → 同类内访问 private，编译期写死
        obj.virtualMethod();           // invokevirtual → 运行期看实际类型 → Child.virtualMethod()
    }
}
class Child extends Father {
    @Override void virtualMethod() { }
}
```

# 8. 分派：静态（重载）与动态（重写）

## 8.1 静态分派 = 重载，编译期决定

### 8.1.1 重载示例

```java
public class StaticDispatch {
    public void say(Human h)  { System.out.println("human"); }
    public void say(Man m)    { System.out.println("man"); }
    public void say(Woman w)  { System.out.println("woman"); }

    public static void main(String[] args) {
        Human man = new Man();     // 静态类型 Human，实际类型 Man
        Human woman = new Woman();

        StaticDispatch sd = new StaticDispatch();
        sd.say(man);       // "human" ← 只看静态类型
        sd.say(woman);     // "human"
        sd.say((Man) man); // "man"   ← 强转改变静态类型
    }
}
```

### 8.1.2 javap 证明选择结果已烙进字节码

`javap -c` 证明选择结果已烙进字节码——描述符里带着编译期挑中的参数类型：

```text
13: invokevirtual #22  // Method say:(LHuman;)V   ← say(man) 编成 (LHuman;)V
20: invokevirtual #23  // Method say:(LMan;)V     ← say((Man)man) 是另一个重载
```

编译器按"静态类型 + 参数个数/类型 + 继承最近匹配"选版本；自动装箱、varargs、`null` 实参引发的重载歧义都发生在这一层（JLS 重载决议），与 JVM 无关。

## 8.2 动态分派 = 重写，运行期查虚方法表

### 8.2.1 重写触发动态分派的示例

```java
Human obj = new Man();
obj.say();  // "man say" ← invokevirtual 解析时看操作数栈顶对象的实际类型
```

### 8.2.2 vtable 槽位与运行期解析四步

```text
Human 的 vtable             Man 的 vtable
┌────────────────┐          ┌────────────────┐
│ [0] toString    │          │ [0] toString    │
│ [1] equals      │          │ [1] equals      │
│ [2] say →Human  │  重写→   │ [2] say →Man    │   ← 同槽位，不同入口
└────────────────┘          └────────────────┘

invokevirtual 运行时解析：
  1. 栈顶引用 → 对象 → Klass Pointer → 实际类型 C（见 [9_object_layout](9_object_layout.md)）
  2. C 的 vtable 中按【方法名+描述符】找匹配槽
  3. 找不到 → 沿父类继续找（子类未重写时槽里存的就是父类方法入口）
  4. 全链找不到 → AbstractMethodError
```

## 8.3 invokeinterface 为什么慢

虚方法表槽位在继承链中位置稳定，`invokevirtual` 一条偏移即可取入口；接口方法在不同实现类里的位置**不确定** → `itable` 需搜索匹配 → 更慢。JIT 的对策：类型谱（monomorphic/multimorphic）+ 内联缓存，绝大多数调用点只有一个/少数实现，直接去虚化内联（见 [17_jit_optimization](17_jit_optimization.md)）。

# 9. invokedynamic

## 9.1 为什么存在

JVM 字节码原以为 Java 专用：静态类型 + 编译期绑定写进了指令设计。JSR 292（JDK 7）加入 `invokedynamic`，把"**调用点的目标怎么找**"从语言层下放为一个可插拔的运行期问题——Groovy/Ruby/JS 引擎、以及后来 Java 自己的 lambda 与字符串拼接都走这条路。

## 9.2 三件套：BootstrapMethod + CallSite + 目标方法

### 9.2.1 触发 indy 的 Java 代码

```java
List<String> list = Arrays.asList("a", "b");
list.forEach(s -> System.out.println(s));
```

### 9.2.2 BSM → CallSite 绑定时序

```text
字节码: invokedynamic #N:apply:()Ljava/util/function/Consumer;   （只执行一次）
   │ #N → BootstrapMethods 表第 0 项
   ▼
BootstrapMethod = LambdaMetafactory.metafactory(caller, "accept", ...,
       samMethodType=(Object)V, implMethod=(MethodHandle) 合成方法 lambda$main$0, ...)
   ▼
执行 BSM → 生成并缓存 CallSite（内含目标方法句柄/隐藏的 lambda 类实例）
   │ 之后所有次执行
   ▼
直接走同一个 CallSite —— BSM 永不第二次运行
```

### 9.2.3 lambda 落地细节与查看手段

- `javap -v` 里 lambda 调用点只显示为 `invokedynamic` + `BootstrapMethods` 属性；**lambda 不再产生 `$1.class` 内部类文件**。
- lambda 方法体编译为同类中的合成静态方法，命名形如 `lambda$main$0`。
- 早期实现每次为接口生成一个普通类；JDK 9+ 转向 `MethodHandle`+类隐藏技术，**JDK 15（JEP 371）隐藏类**后 lambda 类不进公共类加载视图、可直接卸载。
- 想看生成的 lambda 类：`-Djdk.internal.lambda.dumpProxyClasses=<dir>`。

## 9.3 与旧式匿名内部类对比

| 维度 | 匿名内部类（JDK 7 前风格） | invokedynamic（JDK 8+ lambda） |
|------|---------------------------|-------------------------------|
| 产物 | 每个 lambda 位置一个 `$1` 类，永久进类加载器 | 首次调用点由 BSM 生成 CallSite，类可隐藏/可卸载 |
| 实例化 | 每次 `new` | **不捕获变量 → 单例复用**；捕获 → 工厂带上参数 |
| 初始化成本 | 反射式构造器 + 向下转型 | 直接方法句柄，无转型 |
| JIT 视角 | 独立类边界，内联跨类受限 | 合成方法在同一类，更易内联/标量替换 |

## 9.4 顺带一提：JEP 280（JDK 9）

`a + b` 编译成 `invokedynamic`（`StringConcatFactory`），运行期自适应选择拼接策略——字符串拼接因此不再是 StringBuilder 固定展开，常量池也瘦身。

# 10. 栈内存控制

## 10.1 -Xss 的权衡：单帧大小 × 线程数

```bash
-Xss256k      # 换更多可创建线程数（配合虚拟线程出现前的高并发场景）
-Xss1m        # 常见默认（Linux/Windows x64 默认在 512K~1M 区间，平台/版本相关）
```

| 调小 -Xss | 调大 -Xss |
|-----------|-----------|
| 同内存下线程容量↑ | 深递归/复杂表达式更安全 |
| 风险：帧大 → 深调用 SOE（JSON 深树解析、正则回溯常见） | 线程栈总量↑ → `unable to create new native thread` 风险（见 [5_runtime_data_area](5_runtime_data_area.md) 4.3、[22_troubleshooting](22_troubleshooting.md)） |

## 10.2 递归过深演示（修正原示例的编译错误）

### 10.2.1 演示代码

```java
// -Xss256k
public class StackOverflow {
    private int count = 0;
    public void recursive() {
        count++;
        recursive();
    }
    public static void main(String[] args) {
        StackOverflow so = new StackOverflow();
        try {
            so.recursive();
        } catch (StackOverflowError e) {
            System.out.println("Stack depth: " + so.count); // 256k 下常见数千量级
        }
    }
}
```

### 10.2.2 深度数量级表与经验公式

| -Xss | 简单递归（帧小）可容纳深度 |
|------|---------------------------|
| 128k | 千级 |
| 256k | 数千 |
| 1m   | 1~3 万 |

经验式：`深度 ≈ Xss / ((max_locals+max_stack)×4B + 固定开销)`——局部变量多的递归方法，深度会断崖式下降。

## 10.3 线程数造成的栈总量 OOM

每线程一份栈：`线程数 × Xss ≤ 物理内存 - 堆 - 元空间 - CodeCache`。线程数失控时减小 `-Xss` 或限制线程数（线程池化）；容器环境还要注意 cgroup 限制（见 [23_container_jvm](23_container_jvm.md)）。

## 10.4 JDK 21 虚拟线程：栈模型的松动

JEP 444 虚拟线程不独占 OS 线程栈：栈数据切成 **StackChunk 存在堆里**，挂载/卸载时随任务搬上搬下，大小按需伸缩——`-Xss` 对虚拟线程不再逐线程兑现物理内存，海量虚拟线程可行。但虚拟线程仍有**栈深度上限**，超限照样 `StackOverflowError`。

## 10.5 栈 dump 阅读入口

```bash
jstack <pid>            # 每线程一段：帧链自顶向下 = 当前调用路径
```

```text
"main" #1 prio=5 os_hnd=...
   at com.foo.Deep.recurse(Deep.java:12)     ← 帧：类.方法(源文件:行号)
   at com.foo.Deep.recurse(Deep.java:12)     ← 同名连续重复 = 递归
   at com.foo.Deep.main(Deep.java:20)
```

连续重复帧找递归点、`(Native method)` 看 JNI 层、锁等待看 `- locked / - waiting on` ——工具全景见 [20_jvm_tools](20_jvm_tools.md)。
