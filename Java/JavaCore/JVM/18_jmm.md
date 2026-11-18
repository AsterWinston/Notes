# 1. JMM 是什么

## 1.1 一句话定位：抽象规范，不是物理内存分区

- JMM（Java Memory Model，JSR-133 / JLS §17.4）定义了多线程程序中**变量访问的规则**：什么情况下一个线程的写对另一个线程可见、哪些重排序被允许。
- 它是 **编译器、运行时、硬件三方共同遵守的契约**：

| 缔约方 | JMM 对它说什么 |
|--------|----------------|
| javac / JIT 编译器 | 你只能做这些重排，遇到 volatile/锁必须插入屏障 |
| 运行时（HotSpot） | 在目标平台上用这些指令组合实现屏障语义 |
| 硬件内存模型（x86/ARM） | 只要你的乱序不违反下面这张许可表，随便乱 |

- **常见误解澄清**：JMM 的"主内存/工作内存"**不是**堆/栈、也**不是**L1/L2 缓存的物理划分（运行时数据区见 [5_runtime_data_area](5_runtime_data_area.md)）。

## 1.2 为什么需要 JMM：性能与可预期语义的折中

硬件层面无处不在"乱序"：

```text
             处理器核0                                处理器核1
  store a=1 → [Store Buffer①] → L1$ → ┐        ┌ → L1$ ← [Store Buffer②] ← store b=1
                                       ▼        ▼
                                     主内存（写到达顺序 ≠ 发射顺序）
```

- CPU：写缓冲（store buffer）让 store 先"记账"后落地 → 别的核暂时看不到；
- 编译器：寄存器缓存、循环外提（JIT 甚至能把循环里的读**提出循环**，见第 9 节）；
- 若一切按"顺序一致性"执行 → 上述优化全部作废 → 现代 CPU/编译器不存在。

```java
public class VisibilityProblem {
    private static boolean flag = false;
    
    public static void main(String[] args) throws InterruptedException {
        new Thread(() -> {
            while (!flag) {
                // 线程可能永远看不到 flag 被改为 true
            }
            System.out.println("done");
        }).start();
        
        Thread.sleep(1000);
        flag = true;  // 主线程修改了 flag
    }
}
```

- 没有 JMM 的保证，线程间变量修改的可见性不能保证。
- JMM 的策略：**给程序员最小但足够的承诺**（happens-before），把其余自由留给硬件——程序员用 volatile/锁/final "申请"强承诺。

## 1.3 JMM 管什么、不管什么

| 数据 | 是否 JMM 管辖 | 原因 |
|------|---------------|------|
| 实例字段、静态字段、数组元素 | ✅ | 线程间共享 |
| 方法内局部变量（基本类型） | ❌ | 线程私有，天然无竞争 |
| 局部变量**引用的对象** | ⚠️ 引用本身私有，但对象在堆上共享 | 对象一逃逸（被别的线程摸到）立刻入管辖 |

# 2. 主内存与工作内存

## 2.1 抽象模型

```text
┌────────────────────────────────────────────────┐
│                      JVM                        │
│  ┌─────────────────┐    ┌─────────────────┐    │
│  │    线程 1        │    │    线程 2        │    │
│  │  ┌───────────┐  │    │  ┌───────────┐  │    │
│  │  │ 工作内存   │  │    │  │ 工作内存   │  │    │
│  │  │ (CPU缓存)  │  │    │  │ (CPU缓存)  │  │    │
│  │  └─────┬─────┘  │    │  └─────┬─────┘  │    │
│  └────────┼────────┘    └────────┼────────┘    │
│           │       交互             │            │
│           ▼                     ▼            │
│  ┌──────────────────────────────────────┐      │
│  │            主内存 (Main Memory)       │      │
│  │               (共享变量)             │      │
│  └──────────────────────────────────────┘      │
└────────────────────────────────────────────────┘
```

- **主内存**：所有线程共享，存放实例字段、静态字段、数组元素。
- **工作内存**：线程私有，是主内存中变量的**副本抽象**。

## 2.2 交互动作：八操作模型

| 动作 | 方向 | 说明 |
|------|------|------|
| read | 主内存 | 读出变量值 |
| load | → 工作内存 | 写入线程副本 |
| use | 工作内存 | 引擎执行时取用 |
| assign | → 工作内存 | 引擎算出的值赋给副本 |
| store | → 主内存 | 副本回写 |
| write | 主内存 | 落到共享变量 |
| lock / unlock | — | **JSR-133 起已从规范移出**，由 monitor 语义（monitorenter/exit）取代 |

规范真正的约束是：**不是每条动作都必须原子/即时发生**（否则等于禁优化），但 volatile 读写、加锁解锁必须**同步完成**整个 read-load-use / assign-store-write 链。

## 2.3 面试澄清（必考辨析）

```text
❌ 错误说法："工作内存就是 CPU 缓存的划分"
✅ 正确说法：工作内存是对 寄存器 / L1/L2 缓存 / store buffer 等
   "线程私有的值存放处" 的统称抽象；它和物理缓存层级只是相似，不是一一对应
```

- 同步的抽象定义：**交换快照的约定**——不加同步时，各线程可以永远拿着自己的旧快照；`volatile/lock` 就是"此刻作废旧快照、去主内存换新的"的信号。

# 3. 重排序

## 3.1 三种来源

| 类型 | 说明 |
|------|------|
| 编译器重排序 | JIT 编译器在不改变单线程语义下调整指令顺序 |
| 指令级并行重排序 | CPU 指令流水线/乱序执行优化（如果数据不依赖） |
| 内存系统重排序 | CPU 缓存/写缓冲导致写入主内存的顺序可能与执行顺序不同 |

```text
源代码 → 编译器优化重排序 → 指令级并行重排序 → 内存系统重排序 → 最终执行的指令序列
```

## 3.2 as-if-serial：单线程的保底承诺

- 规则：**不管怎么重排，单线程程序的执行结果不能改变**。
- 实现手段：存在**数据依赖**的指令不许重排：

```text
a = 1;        ┐ 写后读(WAR)/写后写(WWW) 若有依赖 → 保序
b = a + 1;    ┘ 编译器和 CPU 都认这条账

x = 1;        ┐ 无依赖 → 允许互换（单线程无人观察，结果等价）
y = 2;        ┘
```

- **关键推论**：as-if-serial 只保护"自己看自己"。两个线程各看各的，交叉起来就可以出现"各自觉得顺序没错、合起来矛盾"的现象——这正是下面 IATSE 的成因。

## 3.3 经典案例：IATSE（独立地址间的存储加载乱序）

```java
// 初值全为 0： int x=0, y=0, a=0, b=0;
// 线程1              // 线程2
a = 1;                b = 1;
x = b;                y = a;
```

**结果 `{a=1, b=1, x=0, y=0}` 在 x86 上都能出现**——x86 允许 StoreLoad 乱序：

```text
时间 →
线程1:  store a=1 ──▶ [写缓冲，暂未可见]     load b → 0（对方 store 还没落地）
线程2:  store b=1 ──▶ [写缓冲，暂未可见]     load a → 0
                         └────── 之后缓冲区才刷入缓存 ──────┘
```

| 观察视角 | 看到的顺序 |
|----------|------------|
| 线程1 自己 | 先写 a 后读 b，as-if-serial 满足 |
| 线程2 自己 | 先写 b 后读 a，也满足 |
| 全局 | 两个"程序顺序"不可能同时成立 → 无同步时不存在全局顺序 |

- 若两个写/读之间放一个 `volatile` 写或锁，StoreLoad 屏障清空写缓冲，该结果即被禁止。

## 3.4 重排序的经典受害者：DCL（未加 volatile）

```java
// 经典的"双重检查锁定"问题（JDK 5 之前）
public class Singleton {
    private static Singleton instance;
    
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                    // new Singleton() 分解为：
                    // 1. 分配内存
                    // 2. 调用构造函数
                    // 3. 将引用赋值给 instance
                    // 重排序后可能 1→3→2，其他线程拿到未构造完的对象
                }
            }
        }
        return instance;
    }
}
```

- 注意：这里的重排发生在 `putstatic instance`（引用赋值）与 `<init>` 字段写之间，锁保护了"写方互斥"，但**没保护的恰是锁外那个 `if (instance == null)` 的读**。修复见 4.3。

## 3.5 Dekker 算法演示：为什么"写+读"互探不够

```java
// 期望：两线程互斥进入临界区（同一时刻至多一个在打印）
Thread1: flag1 = true;  if (!flag2) 进临界区;
Thread2: flag2 = true;  if (!flag1) 进临界区;
```

- 朴素实现下两个 `flag` 的 store 都滞留在各自写缓冲 → 双方都读到对方 `flag = false` → **同时进临界区**。
- 修复：flag 声明 `volatile` —— store 后立即插 StoreLoad 屏障（刷缓冲），load 前插 LoadLoad/LoadStore（不许提前用旧值），互探恢复正确。

# 4. volatile

## 4.1 语义与屏障实现

### 4.1.1 两个语义

1. **可见性**：一个线程对 volatile 变量的修改，其他线程**立即可见**（写必刷出、读必刷新）。
2. **禁止重排序**：volatile 变量的读写与其**上下邻近**的读写之间不允许被穿透重排（靠屏障实现，且通过 HB 传递性间接钉住更多操作对——见第 5 节）。

```java
// volatile 保证可见性
volatile boolean flag = false;

// 线程1
flag = true;  // 写入后立即刷回主内存

// 线程2
if (flag) {   // 每次读取都从主内存获取最新值
    // ...
}
```

### 4.1.2 屏障插入模板

JMM 在编译期按固定模板插屏障（"烘焙面包图"）：

```text
── 普通读写 ──
[StoreStore 屏障]     ← 保证之前的普通写不被排到 volatile 写之后
── volatile 写 ──
[StoreLoad 屏障]      ← 防止 volatile 写与后续 volatile 读重排（最贵）

── volatile 读 ──
[LoadLoad 屏障]       ← 后续普通读不许提到它之前
[LoadStore 屏障]      ← 后续普通写不许提到它之前
── 普通读写 ──
```

### 4.1.3 平台落地：x86 几乎白嫖，ARM 真掏钱

| 抽象屏障 | x86（TSO，见第 8 节） | HotSpot 实际生成 |
|----------|----------------------|------------------|
| LoadLoad / LoadStore / StoreStore | 硬件天然保证 | **空操作，不生成指令** |
| volatile 写（= 全屏障效果） | 需要刷写缓冲 | `lock addl $0x0,(%rsp)` 或 `xchg`——lock 前缀锁缓存行，等价 `mfence` |
| volatile 读 | TSO 下 load 本就不会越过前面的 store | 普通 `mov`，**零开销** |
| ARM/aarch64 | 弱序，四道屏障都可能真插 | `dmb ish` / `ldar`/`stlr`（acquire/release 加载存储指令） |

```text
Java:  volatileFlag = true;
x86:   movl   $0x1, 0x123456(%rip)   // 写变量
       lock addl $0x0, (%rsp)        // ← 隐形全屏障，几~几十 ns
```

## 4.2 volatile 不保证原子性

### 4.2.1 count++ 反例

```java
volatile int count = 0;

// 多线程执行 count++ — 依然有并发问题！
// count++ 是三个操作：读-改-写
Thread t1 = new Thread(() -> { for (int i = 0; i < 10000; i++) count++; });
Thread t2 = new Thread(() -> { for (int i = 0; i < 10000; i++) count++; });
// 结果不一定是 20000
```

### 4.2.2 字节码交错时间线与正解

字节码三步与交错时间线：

```text
getstatic count   // 读（volatile 语义：读到最新值）
iconst_1 / iadd   // 改
putstatic count   // 写（volatile 语义：立即可见）

线程A: getstatic=5          线程B:      getstatic=5
线程A:            putstatic 6
线程B: putstatic 6          ← B 的写基于过期读数，A 的结果被覆盖
```

- **volatile 管"值的可见与不乱序"，不管"读改写不可分割"**。计数场景的正解：`AtomicInteger`（CAS）或加锁（对比见 [19_lock_optimization](19_lock_optimization.md) 第 12 节）。

## 4.3 正确写法：volatile DCL 与字节码分析

### 4.3.1 正确代码

```java
public class Singleton {
    private static volatile Singleton instance;
    
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

### 4.3.2 字节码级时序分析

```text
instance = new Singleton() 的三条字节码：
  0 new           #2  Singleton        // ① 分配内存（此时可被别的线程引用到地址）
  3 dup
  4 invokespecial #3  <init>           // ② 执行构造，填充字段
  7 putstatic     #4  instance         // ③ 引用逃逸给全 JVM

不加 volatile：②③ 可交换（③ 写进缓存即可，② 还在后面）→ 锁外读侧拿到半成品
加 volatile 后：
  - putstatic 被升级为 volatile putstatic
  - JIT 在 ② 与 ③ 之间插 StoreStore 屏障 → 构造字段写先于引用发布可见
  - 读侧 getstatic 后插 LoadLoad/LoadStore → 拿到引用后再读字段，必然命中已发布的初始化值
```

## 4.4 适用场景表

| 场景 | 模式 | 例子 |
|------|------|------|
| 状态标志位 | 一写多读，读到 true 时相关数据已 HB 可见 | `volatile boolean shutdown` |
| 一次性安全发布 | holder 模式：静态变量只赋值一次，之后只读 | `volatile Resource r`（先写完 r 内部字段再赋 r） |
| DCL 单例 | 引用的发布与初始化顺序约束 | 4.3 |
| 独立读优先、写极少 | 读侧免锁降低争用 | 配置开关、缓存指针 |
| ❌ 计数/累加 | 复合 RMW 不原子 | 用 AtomicInteger / LongAdder |
| ❌ 多写者协调 | 可见≠互斥 | 用锁或 CAS 循环 |

```java
// 经典"读多写少的低开销读写"折中（可见性归 volatile，原子性归锁）
volatile int count = 0;
public int getCount() { return count; }              // 读：无锁
public synchronized void increment() { count++; }    // 写：锁保证读改写原子
```

# 5. happens-before 原则

## 5.1 定义与八条规则

### 5.1.1 定义

happens-before 是 JMM 判断数据是否存在竞争、是否线程安全的**主要依据**。

若操作 A happens-before 操作 B，则 A 的结果对 B 可见，且逻辑上 A 在 B "之前"（注意：指**可见性偏序**，不是物理时间先后）。

### 5.1.2 八条规则

| # | 规则 | 说明 | 示例 |
|------|------|------|------|
| 1 | 程序次序规则 | 同一线程中前面的操作 hb 后面的 | `A=1; B=A;` |
| 2 | 监视器锁规则 | 同对象 unlock hb 后续 lock | synchronized 进出 |
| 3 | volatile 规则 | volatile 写 hb 后续对该变量的读 | DCL |
| 4 | 线程启动规则 | `Thread.start()` hb 该线程内所有操作 | start 前赋值的字段新线程可见 |
| 5 | 线程终止规则 | 线程内所有操作 hb 其他线程 `join()` 成功返回 | 也含 `isAlive()==false` 检测 |
| 6 | 中断规则 | `interrupt()` hb 被中断线程检测到中断（`interrupted()`） | 取消信号 |
| 7 | 对象终结规则 | 构造函数结束 hb `finalize()` 开始 | 遗留语义（finalize 已废弃） |
| 8 | 传递性 | A hb B, B hb C → A hb C | 串联上面七条的胶水 |

> 周志明《深入理解 Java 虚拟机》常精简为"4 大规则"：程序顺序/锁/volatile/传递性；start/join/中断/finalize 是 JLS 17.4.5 里与线程 API 绑定的补充边。

### 5.1.3 传递性链示例

```java
// happens-before 实例
int a = 0;
volatile boolean flag = false;

// 线程 1
a = 1;          // 1
flag = true;    // 2

// 线程 2
if (flag) {     // 3  (volatile 读)
    int b = a;  // 4  (由于 2 hb 3，且 1 hb 2，所以 1 hb 4，b 一定 = 1)
}
```

- 若 3 没读到 true（走 else），则 2 hb 3 这条边不成立，b 的值无从保证——HB 边只在**事件真实发生**时构成。

## 5.2 两个高频误区

| 误区 | 纠正 |
|------|------|
| "hb = 时间上先发生" | hb 是**可见性承诺**；JLS 明言：若两操作间既无 hb 边也推不出先后，JVM 可任意重排，哪怕墙钟时间一前一后 |
| "有 hb 就原子" | hb 只管可见与序，不合并 RMW；`count++` 全程满足 hb 照样丢更新（4.2） |

## 5.3 用 HB 图证明 DCL 安全性

```text
构造器内写字段 obj.x ──程序顺序──▶ ② volatile putstatic instance
                                        │ volatile 规则（发布）
                                        ▼
③ 另一线程 getstatic instance 读到非空 ──程序顺序──▶ ④ 读 obj.x

链条：① hb ② hb ③ hb ④ ⇒ ① hb ④：看到引用 ⇒ 必然看到完整初始化的字段 ✅
（若 instance 不加 volatile：② hb ③ 这条边不存在，④ 可能读到默认值 0/null）
```

# 6. final 域的内存语义

## 6.1 final 域的重排序规则

1. **写**：构造函数内对 final 域的写入，与随后把**对象引用赋值给引用变量**，这两个操作**不能重排序**（构造 return 前插入 StoreStore 屏障"冻结"）。
2. **读**：初次读一个包含 final 域的对象**引用**，与随后第一次读该 final 域，这两个操作**不能重排序**。

```java
// final 保证线程安全的发布
public class SafePublication {
    private final int x;
    private int y;
    
    public SafePublication(int x) { 
        this.x = x;   // final 写
        // StoreStore 屏障：保证 x 的写不被重排序到构造函数之外
        this.y = 1;
    }
    
    // 多线程安全：读取安全发布的对象时，final 域一定被正确初始化
}
```

- 对比记忆：普通字段 `y` 只有**HB 链完整**才安全；`x` 只要发布动作本身"让引用可见"（哪怕通过 racy 的普通静态字段），final 冻结语义也保证 x 正确。

## 6.2 final 域是引用类型

```java
public class RefFinal {
    private final int[] array;
    
    public RefFinal() {
        array = new int[10];   // final 写
        array[0] = 1;          // 数组元素的写在构造函数 return 前完成
    }
}
```

- 构造函数内对 final 引用指向的对象的**成员域**的修改，在构造函数 return 后才能被其他线程看见。
- **边界**：保护的是"构造完成瞬间的状态"。构造后再改数组元素（`array[1]=2`）没有任何 final 保护，退化为普通共享写。

## 6.3 不可变对象安全发布模式

```java
public final class Config {
    private final Map<String, String> rules;   // final 字段

    public Config() {
        rules = new HashMap<>();
        rules.put("timeout", "3s");            // 构造内写完
        // ✔ 构造期 this 从未逸出（没调 listener.register(this) 之类）
    }
    public String get(String k) { return rules.get(k); }  // 只读方法
}
```

三要件缺一不可：

| 要件 | 破坏后果 |
|------|----------|
| 所有公开状态 final | 非 final 字段靠发布时的 hb 边，易失守 |
| 构造函数内 `this` 不逸出 | 别的线程拿到"半成品"，final 冻结未生效 |
| 类不做任何后续修改 | 一改就不是不可变，回到普通共享数据 |

- JDK 不可变类（`String`、`LocalDateTime`、`concurrent` 里的 Node key）都按此模板写。

# 7. 内存屏障（Memory Barrier）

## 7.1 屏障语义与成本

### 7.1.1 四类屏障定义矩阵

| 屏障类型 | 指令示例 | 说明 |
|----------|----------|------|
| LoadLoad | Load1; LoadLoad; Load2 | Load1 的读取一定先于 Load2 |
| StoreStore | Store1; StoreStore; Store2 | Store1 的写入对其他处理器先于 Store2 可见 |
| LoadStore | Load1; LoadStore; Store2 | Load1 的读取一定先于 Store2 的写入 |
| StoreLoad | Store1; StoreLoad; Load2 | Store1 的写入对所有处理器可见后，才能执行 Load2 |

- **StoreLoad 是最重的屏障**（同时具有其他三个屏障的效果，俗称全屏障），开销也最大。

### 7.1.2 平台实现与成本

| 平台 | 前三种屏障 | StoreLoad |
|------|------------|-----------|
| x86/x64 | 硬件天然满足 → 空操作 | `mfence` / `lock` 前缀指令 / `xchg`；只在 volatile 写、lock/unlock 出口处插入 |
| ARM/aarch64 | `dmb ishld` / `dmb ish` 或直接用 `ldar/stlr` | `dmb ish` |
| RISC-V | `fence rw,rw` 等组合 | `fence rww,w` 类全屏障 |

## 7.2 HotSpot 屏障插入策略表（JSR-133 cookbook）

| 语义点 | 插入位置 |
|--------|----------|
| volatile 写之前 | StoreStore |
| volatile 写之后 | **StoreLoad** |
| volatile 读之后 | LoadLoad + LoadStore |
| 监视器加锁之后 | LoadLoad + LoadStore（x86 不生成指令） |
| 监视器解锁之前 | StoreStore + **StoreLoad** |

```text
可见性来源总结：解锁线程刷出所有写 + 加锁线程作废缓存重新加载
→ "同一把锁"本身就构成一条完整的数据搬运通道（hb 规则 2 的物理基础）
```

# 8. JMM 与硬件内存模型

## 8.1 x86：强内存序 TSO（Total Store Order）

```text
TSO 允许的唯一乱序：StoreLoad（写缓冲导致）
  Store→Store ✅ 保序   Load→Load ✅ 保序   Load→Store ✅ 保序
  即：JMM 四道屏障里只有 StoreLoad 在 x86 上真花钱
```

## 8.2 ARM / RISC-V：弱内存序

- 同地址读间可能乱（LoadLoad 要真屏障）、写可合并重排、推测加载……
- 依赖 acquire/release 指令对（`ldar/stlr`）与 `dmb` 才能拼出 JMM 承诺。

## 8.3 结论

| 事实 | 工程含义 |
|------|----------|
| 同一份无同步代码在 x86 测一万遍都对，到 ARM（Apple Silicon/Graviton 安卓）翻车 | **x86 测试通过 ≠ 无并发 bug** |
| JMM 在每个平台补齐到同一语义下限 | 程序员只需面向 hb 编程，屏障差异被 JVM 屏蔽——这正是 JMM 存在的意义 |

## 8.4 MESI 一句话预告

- 缓存**一致性**协议（MESI）解决"各核缓存副本最终同步"的问题，与内存**顺序**（store buffer 导致的乱序）是两个独立维度；MESI 让 volatile 的"可见"代价变小，但**不**提供顺序保证——细节超出 JVM 范畴，属体系结构领域。

# 9. 实战：验证可见性问题

## 9.1 复现代码

```java
public class VisibilityDemo {
    // 不加 volatile
    private static boolean flag = false;
    // private static volatile boolean flag = false;  // 加上可见性
    
    public static void main(String[] args) throws InterruptedException {
        new Thread(() -> {
            int i = 0;
            while (!flag) {
                i++;
                // 如果加入 System.out.println，可能因为 sout 内部的
                // synchronized 导致工作内存被刷新，flag 突然可见
            }
            System.out.println("Thread stopped, i=" + i);
        }).start();
        
        Thread.sleep(1000);
        flag = true;
        System.out.println("Flag set to true");
    }
}
// 不加 volatile，子线程可能永不停止（C2 编译后 flag 被优化到寄存器）
// 加上 volatile 后，子线程能立即感知 flag 的变化
```

## 9.2 死循环的真正成因（比"CPU 缓存"更准确的说法）

```text
热点循环被 C2 编译后：
  原字节码: 每次迭代 getstatic flag → test → ifeq
  编译结果: 循环开始前把 flag 读进寄存器/直接判定 !flag 恒真
           → 循环体成为"无内存依赖"的计数循环（LICM 不变量外提，见 17 第 7 节）
           → 之后再也不碰那个内存地址 → 主线程写多少次都看不到
```

- `-Xint` 解释执行时每次重新 getstatic，往往**复现不了**死循环——"解释器可见、编译器不可见"恰说明这是**编译优化**主导、硬件乱序次要的问题。
- 打印 `System.out.println` 能"治好"它：synchronized 块进出口的屏障强制重读。

## 9.3 验证工具

| 工具 | 用法 |
|------|------|
| `-XX:+PrintCompilation` / `-XX:+PrintAssembly`(hsdis) | 亲眼看到 flag 读取被提出循环（汇编里循环体无该地址 load；操作见 [17_jit_optimization](17_jit_optimization.md) 第 12 节） |
| **jcstress**（`org.openjdk.jcstress`） | OpenJDK 官方并发压测框架，穷举指令交错结果并归类 `观察到/禁止`，验证内存模型问题的事实标准 |
| ThreadSanitizer / helgrind | 偏 native 侧的补充手段 |

# 10. VarHandle：现代低层内存访问

## 10.1 定位（JEP 193，JDK 9）

- `java.lang.invoke.VarHandle`：JDK 9 随模块系统引入，**替代 `sun.misc.Unsafe` 的字段级原子/有序访问入口**（目标：Unsafe 内存操作方法逐步废弃）。
- 一个 VarHandle ≈ "绑定到某个字段/数组元素/字节缓冲位置"的访问器句柄，创建成本高、复用成本低（`static final` 持有）。

## 10.2 访问模式与用法示例

### 10.2.1 四种内存序访问模式

| 模式族 | 示例方法 | 保证 | 类比 C++ |
|--------|----------|------|----------|
| Plain | `get` / `set` | 单字段原子性（long/double 也保证），**无**可见性/序保证 | relaxed 以下（普通 load/store） |
| Opaque | `getOpaque` / `setOpaque` | 编译器不瞎重排，硬件级仍可能弱 | atomic relaxed |
| Acquire-Release | `getAcquire` / `setRelease` | 读后不许提前 / 写后不许滞后的单向屏障 | acquire / release |
| Volatile | `getVolatile` / `setVolatile` | 完整 volatile 语义（含 StoreLoad） | seq_cst |

另有 `compareAndSet` / `getAndAdd` / `getAndUpdate` 等**原子读改写**模式（volatile 级别）。

### 10.2.2 示例：手写一个 acquire-release 发布

```java
class Publisher {
    private static final VarHandle DATA;
    static {
        try {
            DATA = MethodHandles.lookup()
                    .findStaticVarHandle(Publisher.class, "data", Object.class);
        } catch (ReflectiveOperationException e) { throw new ExceptionInInitializerError(e); }
    }

    private static Object data;        // 不声明 volatile，由 VarHandle 管语义

    static void publish(String[] rows) {
        // 先准备内容（普通写）……
        DATA.setRelease(rows);         // release 发布：前面的写对配对的 acquire 读全部可见
    }

    static String[] poll() {
        String[] rows = (String[]) DATA.getAcquire();  // acquire 获取
        if (rows != null) { /* 此处读 rows 元素安全 */ }
        return rows;
    }
}
```

```text
release/acquire 配对 = 单向 happens-before 边：
  setRelease 之前的所有写 ──hb──▶ getAcquire 之后的所有读
比 volatile 便宜的地方：弱序平台上可省全屏障；比 volatile 危险的地方：配对错一个字母就退回无保证。
```

## 10.3 与 Atomic 家族的关系

| 组件 | 底层实现（JDK 9+） |
|------|--------------------|
| `AtomicInteger.compareAndSet` | VarHandle 的 `compareAndSet` |
| `AtomicReferenceArray` | VarHandle（数组元素句柄） |
| `LongAdder` | 内部 `Striped64.Cell.value` 用 VarHandle CAS（+ `@Contended` 防伪共享，见 [9_object_layout](9_object_layout.md)） |
| AQS（ReentrantLock 的 state） | JDK 9+ 通过 Unsafe→VarHandle 迁移路线访问 state（见 [19_lock_optimization](19_lock_optimization.md) 第 12 节） |

- 记忆锚点：**volatile/synchronized 是语言级"全自动"承诺；VarHandle 是库级"按需取用不同强度"的手动挡**。

# 11. 速查：JMM 与各层工具对应

| 工具 | JMM 基础 | 典型场景 |
|------|----------|----------|
| volatile | 可见性 + 禁止重排序 | 状态标记、DCL 单例 |
| synchronized | 原子性 + 可见性 + 禁止重排序（hb：unlock→lock） | 互斥访问 |
| Lock (J.U.C) | 基于 volatile 状态 + CAS | 更灵活的锁 |
| AtomicXXX | CAS（底层 `LOCK CMPXCHG`）/ VarHandle | 无锁计数、状态更新 |
| VarHandle | plain/opaque/acquire-release/volatile 四档 | 库作者级精细内存序 |
| final | 构造函数冻结 + 安全发布 | 不可变对象 |
| Thread.start/join | HB 边 4/5 | 一次性交接任务数据 |
