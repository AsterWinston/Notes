# 1. Java 中锁的优化历程

## 1.1 时间线总览

```text
JDK 1.0-1.2: synchronized 直接使用 OS 互斥量（重量级锁），非常慢
JDK 1.3-1.4: 初步优化（自旋思想、轻量级锁雏形）
JDK 1.5:   引入 J.U.C (ReentrantLock, AtomicXXX, ConcurrentHashMap 等)，锁家族一分为二
JDK 1.6:   锁优化重大升级 —— 偏向锁、轻量级锁、自适应自旋、锁消除、锁粗化
JDK 1.7-1.8: 持续优化，ConcurrentHashMap 改用 CAS + synchronized
JDK 13:    偏向锁默认关闭
JDK 15:    偏向锁正式废弃（JEP 374，默认禁用并标记 for removal）
JDK 18:    偏向锁相关实现与参数彻底移除
JDK 21:    虚拟线程转正（JEP 444）—— synchronized 阻塞会 pin 载体线程，锁选型逻辑再次改变（见第 15 节）
JDK 24:    JEP 491：虚拟线程在 synchronized 中不再钉住载体线程（绝大多数场景）
```

## 1.2 各年代"主要矛盾"

| 时期 | 矛盾 | 应对 |
|------|------|------|
| JDK 6 之前 | 大量遗留 synchronized 代码（Vector/Hashtable 时代）在单线程场景也要付 mutex 成本 | 偏向/轻量级：把"无竞争"从 OS 拉回用户态 |
| JDK 6 | 自旋次数难写死：固定值要么浪费 CPU 要么过早挂起 | 自适应自旋 + 编译期锁消除/粗化 |
| JDK 15+ | 偏向锁撤销要 STW、收益锐减、阻碍 VM 演进 | 直接移除（JEP 374） |
| JDK 21+ | 百万虚拟线程 × 平台线程绑定的 monitor | 先规约（少在 sync 块里阻塞），后由 JEP 491 根治 |

# 2. Mark Word 中的锁信息

## 2.1 各锁状态的位图（64 bit）

回顾 Mark Word（详见 [9_object_layout](9_object_layout.md)）：

| 锁状态 | 标志位(2bit) | 偏向锁位(1bit) | 存储内容 |
|--------|-------------|---------------|----------|
| 无锁 | 01 | 0 | hashCode（31bit），分代年龄（4bit） |
| 偏向锁 | 01 | 1 | 线程 ID（54bit），epoch（2bit），分代年龄（4bit） |
| 轻量级锁 | 00 | - | 指向栈中锁记录（Lock Record）的指针 |
| 重量级锁 | 10 | - | 指向 ObjectMonitor 的指针 |
| GC 标记 | 11 | - | 空 |

```text
64 位视角（自高到低示意）：

无锁    [  unused  |   identity hashCode 31   | age 4 | 0 | 01 ]
偏向    [  thread ID 54  | epoch 2 | u 1 | age 4 | 1 | 01 ]
轻量级  [        栈上 Lock Record 指针 62        | 00 ]
重量级  [        ObjectMonitor 指针 62           | 10 ]
GC      [        转发指针 62                     | 11 ]

判定口诀：低 2 位认状态；低 2 位同为 01 时，再看第 3 位（偏向位）区分无锁/偏向。
```

## 2.2 两个值得注意的细节

- **hashCode 与偏向的冲突**：identity hash 算出后要占 31 位，偏向位被挤掉 → 对偏向锁对象调用 `System.identityHashCode()` 会**立即撤销偏向**（降级为无锁存 hash）。这是偏向收益不稳定的原因之一。
- **age 只有 4 bit**：晋升阈值上限 15（联动 [6_heap](6_heap.md) 动态年龄规则）。

# 3. 锁状态转换

## 3.1 升级流程图

```text
                   ┌──────────┐
                   │   无锁    │
                   └────┬─────┘
             首次线程A 请求锁
                         │
                    ┌────▼─────┐
                    │  偏向锁   │  ← 记录线程A的ID，A再进无需CAS
                    └────┬─────┘
             线程B 竞争同一把锁
                         │
                    ┌────▼─────┐
      撤销偏向 ←────│  竞争     │
         │          └────┬─────┘
         撤销后            │
         升级              ▼
                    ┌──────────┐ ← 自旋获取（CAS）
                    │ 轻量级锁  │
                    └────┬─────┘
           自旋超过阈值 / 竞争激烈
                         │
                    ┌────▼─────┐
                    │ 重量级锁  │ ← ObjectMonitor, OS 互斥
                    └──────────┘
```

## 3.2 单向不可逆？——比"不可降级"更准确的说法

- **锁只能升级，不能降级**（偏向锁可撤销退回无锁，这算"撤销"不算"降级链"）：

| 问题 | 论证 |
|------|------|
| 为什么不设计"重量级→轻量级" | 需要检测"无竞争足够久"：加计时器/计数器 → 每次解锁多付原子操作；而 monitor 本身可被回收复用（近年 JDK 的异步 deflation 已让空闲 ObjectMonitor 进缓存池复用），降级收益 < 复杂度成本 |
| 升级判定放哪 | 全在 CAS 失败路径上顺手完成，零额外元数据；降级必须引入"竞争已消失"的全局观测，safepoint 或后台线程代价高 |
| 工程含义 | 只要历史上出现过一次激烈竞争，这把锁此后一直按重量级成本走——**热点争用锁尽量在源头避免**，别指望 VM 帮你"养回去" |

# 4. 偏向锁（Biased Locking）

## 4.1 解决什么问题

- 背景：JDK 6 之前的世界里，库里到处是"防君子不防小人"的 `synchronized`（Vector/Hashtable/StringBuffer），而实际调用**几乎永远单线程**。
- 即便有轻量级锁，单线程重入也要"每次加锁 1 次 CAS + 每次解锁 1 次 CAS"；偏向锁把这条路径削成**零原子操作**。

```java
// 偏向锁受益场景
public class BiasedLocking {
    // Vector 的 add 方法是 synchronized 的
    private final Vector<String> list = new Vector<>();
    
    public void add() {
        // 大多数情况只有单线程调用 → 偏向锁极高效
        list.add("item");
    }
}
```

## 4.2 机制与撤销

### 4.2.1 原理：把"再入"降为一次普通比较

| 动作 | 实现 | 成本 |
|------|------|------|
| 首次加锁 | CAS 把**线程 ID + epoch** 写进 Mark Word，偏向位置 1 | 1 次 CAS |
| 同线程再入 | 只比较 Mark Word 里的 ThreadID == 当前线程 | 一次普通 load+cmp |
| 同线程解锁 | **什么都不做**（锁状态留在对象上） | 0 |
| 别的线程想加锁 | 必须撤销偏向（见 4.2.2） | 昂贵 → 触发设计死刑 |

### 4.2.2 撤销（Revoke）流程

竞争者出现时，偏向必须整体作废——因为 Mark Word 里那个"指向持有线程"的字段没有第二个位置可用：

```text
① 等待全局安全点（safepoint）——保证没有线程正把该对象当锁用到一半
      │
② 重新检查对象头，找到原持有线程 T：
      ├─ T 已退出同步块 / T 已死 → 对象复活为无锁（或重偏向新持有者）
      └─ T 仍活着且在临界区内   → 为 T 膨胀出 ObjectMonitor，Mark Word 指向 monitor
                                    → 对象直接变重量级锁
③ 恢复执行，竞争双方回到重量级/轻量级路径排队
```

- 撤销要 **STW 级别的安全点**，这是它"竞争场景下越用越亏"的根源。
- 注意：撤销后该对象一般**直接膨胀为重量级**，而非退回偏向起点。

### 4.2.3 批量重偏向与批量撤销（epoch 机制）

JVM 发现"这个类的偏向在 systematically 失败"时启动群体策略，靠 Mark Word 里的 **epoch（2 bit）**做群体失效标记：

| 机制 | 触发条件 | 动作 |
|------|----------|------|
| 批量重偏向（bulk rebias） | **同一类**对象的偏向撤销累计过阈值（历史上默认 20，后续版本有调整） | 把类的 epoch +1 → 该类所有已偏向对象在下次被访问时"发现 epoch 过期"→ 惰性重新偏向**当前持有线程**（暗示：可能只是两个线程交替使用，重分发给活人即可） |
| 批量撤销（bulk unlink/disable） | 撤销次数继续冲高（历史阈值约 40） | JVM 判定该类**天生有竞争**，修改类标记**禁用其偏向**，已有偏向对象全部走安全点匿名化（降级无锁/重量级） |

```text
计数对象：Class 的 _bias_epoch + 撤销计数
思路：不逐个追踪"哪个对象该不该偏向"，而是对整个类"一刀切"——
     因为同类的锁行为高度一致（都来自同一 synchronized 方法的 this 锁）。
```

## 4.3 废弃与移除（JEP 374）

- **JDK 13 起默认关闭**偏向锁，**JDK 15 正式废弃并默认禁用（JEP 374）**，后续版本已把实现整体移除（JDK 18 起参数 `-XX:±UseBiasedLocking` 出现即告警并被忽略）。
- 官方动机（按重要性）：

| 原因 | 说明 |
|------|------|
| 收益锐减 | 现代代码转向 J.U.C 并发容器与不可变对象，"单线程 synchronized 海"变少 |
| 撤销/重偏向开销 | 竞争场景下安全点操作抖动 GC 与延迟（微服务对尾延迟敏感） |
| 复杂度与坑 | identityHashCode 冲突、批量策略阈值、与各模块耦合的边角 bug |
| 挡路 | 为 HotSpot 后续锁重构（如 JEP 491 虚拟线程与 monitor 的新关系、值类型）腾挪空间 |

```bash
# 历史参数（现均无效，读老配置时认识即可）
-XX:+UseBiasedLocking            # JDK 13 前默认开启，13 起默认关，15 废弃
-XX:BiasedLockingStartupDelay=0  # 取消启动后约 4s 的偏向宽限期
```

# 5. 轻量级锁（Lightweight Locking）

## 5.1 加锁、重入与解锁

### 5.1.1 加锁：栈上 Lock Record + CAS

- 线程在进入同步块前，在当前**栈帧**中创建 **Lock Record**：

```text
线程栈                   堆中的对象
┌────────────┐         ┌──────────────────┐
│ 栈帧        │         │  对象头            │
│ ┌────────┐ │    CAS  │ ┌──────────────┐  │
│ │Lock    │ │←───────│ │ Mark Word    │  │
│ │Record  │ │         │ │ (指向LR)      │  │
│ └────────┘ │         │ ├──────────────┤  │
└────────────┘         │ │ Klass Pointer│  │
                       │ └──────────────┘  │
                       └──────────────────┘
```

流程：

```text
① 复制对象 Mark Word → Lock Record 的 displaced mark word 字段（备份，解锁时写回）
② CAS：把对象 Mark Word 从旧值改为 "指向该 Lock Record 的指针 + 00"
③ CAS 成功 → 拿到轻量级锁（全程用户态，无内核参与）
④ CAS 失败 → 进入 5.1.2 分支判断
```

### 5.1.2 重入与解锁

| 情形 | 处理 |
|------|------|
| CAS 失败，但对象 Mark Word 已指向**本线程栈内**的 Lock Record | 是重入：再压一条 **displaced = NULL** 的 Lock Record 充当计数（不碰对象头） |
| CAS 失败，Mark Word 指向**他人**的 Lock Record | 真竞争 → 自旋（见第 7 节）或膨胀重量级 |
| 解锁（displaced 非 NULL 的首条记录） | CAS 把 displaced mark word 写回对象头；**写回前检查对象头是否已被改动**——被改说明解锁期间有线程竞争过 → 走重量级解锁流程 |
| 解锁（displaced = NULL 的重入记录） | 弹掉计数记录即可，什么都不用写 |

```text
"检查 displaced 是否被改动"是关键设计：
   竞争者抢锁失败时有权把 Mark Word 置成 10（膨胀），
   持有者解锁时据此发现"这锁已经不属于轻量级世界了"，交还给 ObjectMonitor 处理。
```

### 5.1.3 适用与局限

- 适用：**两个线程交替执行同步块**（重叠极少、临界区极短）——CAS 一遍就过，成本一次原子指令。
- 局限：
  - 自旋等待烧 CPU（膨胀条件即为此设）；
  - 深递归加锁在栈上叠 Lock Record，占栈空间；
  - ABA 类问题在实现层由"指针指向本栈 + 版本号语义"规避，无需用户操心。
- 与 JDK 21 虚拟线程的互动：轻量级/偏向都以 **platform thread 身份**记录持有者，虚拟线程语义下的持有关系见第 15 节。

# 6. 重量级锁（Heavyweight Locking）

## 6.1 膨胀路径与 ObjectMonitor

### 6.1.1 依赖 OS：mutex / futex

- 轻量级 CAS 反复失败（自旋超时、或竞争者太多）→ 膨胀：分配/关联 **ObjectMonitor**，Mark Word 置 `10` 指向它。
- 未获锁线程**挂起**（BLOCKED），让出 CPU；唤醒走内核调度：

```text
用户态 CAS 排队（try） ──失败──▶ futex_wait 陷入内核 ──解锁方 futex_wake──▶ 重新调度
                     ▲ 省掉系统调用时便宜                  ▲ 每次穿越内核 ≈ µs 级 + 上下文切换
```

- Linux 实现链：`pthread_mutex` 底层 futex（有竞争才睡）；Windows 用 critical section 同理。**"重量级"的贵不在互斥本身，而在线程睡眠/唤醒的内核往返。**

### 6.1.2 ObjectMonitor 核心字段

```text
对象关联的 ObjectMonitor（每个膨胀对象一个）:
┌───────────────────────────────┐
│ _owner      → 当前持锁线程      │
│ _recursions → 重入计数          │
│ _EntryList  → 阻塞等锁的线程队列 │
│ _WaitSet    → wait() 挂起的线程  │
│ _cxq        → 竞争者交接入队栈   │
└───────────────────────────────┘
```

### 6.1.3 wait/notify 与监视器的舞蹈

```text
持锁线程 wait():
   释放 monitor（owner 清空）→ 自己进 _WaitSet → 挂起
                                    │
其他线程 notify()/notifyAll():       ▼
   把 _WaitSet 中的等待者移入 _EntryList（只是"可以开始抢锁"，不是"得到锁"）
                                    │
   等待者从 EntryList 醒来参与竞争 ──┴─▶ 抢到才从 wait() 返回（所以必须 while 循环复检条件）
```

## 6.2 公平性与吞吐取舍

| 特性 | 说明 |
|------|------|
| 非公平 | 新来的线程可以"插队"（barging）直接抢 owner 刚腾出的锁——省一次唤醒，吞吐优先 |
| 惊群缓解 | notifyAll 后并非全部立刻争用，EntryList/等待队列分批处理 |
| 无公平选项 | `synchronized` 没有"公平锁"开关；要公平只能 ReentrantLock(true)（第 11 节） |

## 6.3 近年的持续改进（模糊但真实）

- JDK 15 前后重写了 ObjectMonitor 的等待队列（cxq/EntryList 交互、异步 deflation 回收闲置 monitor），争用吞吐与内存占用明显改善；
- 此后各版本仍在队列策略、NUMA/容器核数自适应上迭代——**"重量级锁很慢"是 JDK 6 时代的记忆，今天的重量级锁在激烈争用下已经是相当工程化的实现**，但用户态/内核态切换的固有成本不会消失。

# 7. 自旋锁与自适应自旋

## 7.1 自旋锁

- 轻量级锁 CAS 失败后，线程不立即阻塞，而是执行空循环（自旋）。
- 如果锁很快释放，自旋避免了线程挂起/唤醒的开销。
- 如果自旋太久，浪费 CPU。

```bash
-XX:PreBlockSpin=10            # 固定自旋次数（JDK 6 及之前的思路，参数早已废弃）
-XX:+UseSpinning               # （历史参数，现已无意义，JDK 7 起移除）
```

## 7.2 自适应自旋（JDK 6+）

- 自旋时间不再固定，由 **profiling 反馈**动态决定——对象头/Klass 上记录历史成功率：

| 历史信号 | 当前策略 |
|----------|----------|
| 该锁最近一次自旋成功过 | 允许更久自旋（甚至默认这次也会成功） |
| 最近自旋很少成功 / 持有者还在被阻塞 | 直接挂起，不陪跑 |
| 持有线程已不在运行（被换下 CPU） | 自旋无意义，立即 park |

- **没有用户参数可调**（PreBlockSpin 时代结束）；编译期能确定的场景，锁策略由 JIT 直接选路。
- 容器/低核数环境注意：自旋 = 与业务线程抢 CPU，核数紧张时表现更差（联动 [23_container_jvm](23_container_jvm.md)）。

# 8. 锁粗化（Lock Coarsening）

- JIT 编译时优化，将频繁的加锁/解锁合并——与相邻循环内反复 enter/exit 造成的"假竞争"作斗争。

```java
// 优化前
for (int i = 0; i < 10000; i++) {
    synchronized (lock) {
        counter++;
    }
}

// JIT 粗化后
synchronized (lock) {
    for (int i = 0; i < 10000; i++) {
        counter++;
    }
}
```

- 判定要点：连续的 lock/unlock 对之间**没有别的线程介入的合理窗口**（如方法调用/逃逸点）；
- 上限：粗化范围有限，不会跨大段代码"能圈多大圈多大"——临界区语义敏感（圈进来一个新调用就可能改变可见性），编译器只做保守合并。
- 编译层完整说明见 [17_jit_optimization](17_jit_optimization.md) 第 4 节。

# 9. 锁消除（Lock Elimination）

- 基于逃逸分析，消除**不可能被别的线程看到**的锁。

```java
// StringBuffer → 逃逸分析发现 sb 只在线程内使用 → 消除 synchronized
public String build() {
    StringBuffer sb = new StringBuffer();
    sb.append("a");
    sb.append("b");
    return sb.toString();
}
// 优化后等效于 StringBuilder
```

```text
证据链： EA 证明 sb 不逃逸出本线程
      → 不存在第二个线程能拿到同一个 monitor
      → monitorenter/monitorexit 恒为"无竞争路径"
      → 整对删除（连轻量级 CAS 都不留）
```

| 前提失效场景 | 结果 |
|--------------|------|
| sb 被赋给字段/参数外传（且内联边界外无法证明） | 消除失败，锁真实存在 |
| JDK 版本/分层状态不同（C1 不做 EA） | 同一代码在预热前后行为不同——**正确性绝不能依赖锁消除** |

- 参数：`-XX:+EliminateLocks`（默认开）；机制详见 [15_memory_allocation](15_memory_allocation.md) 与 [17_jit_optimization](17_jit_optimization.md) 第 2、3 节。

# 10. synchronized 的底层实现

## 10.1 同步方法的字节码证据

```java
public synchronized void doWork() {
    // ...
}

// 等效于
public void doWork() {
    synchronized (this) {
        // ...
    }
}
```

```bash
javap -v -p Demo.class
```

```text
// 同步方法：flags 里出现
  flags: (0x0002) ACC_SYNCHRONIZED   ← JVM 见到该标志自动按"锁 this / 锁 Class"执行整套 monitor 逻辑

// 同步代码块：显式指令 + 完整异常表
  0: aload_1
  1: monitorenter             // 尝试获取锁
  ...code...
  n: monitorexit              // 正常路径退出
  ...
  m: aload_1
     monitorexit              // 异常处理器里再退出（保证异常也解锁）
     athrow
Exception table:
  from 1  to n-1  target (异常处)  any     ← "any" 覆盖一切异常 → synchronized 天生异常安全
```

| 层面 | 要点 |
|------|------|
| monitorenter | 走第 3~5 节的加锁决策树（偏向→轻量→重量） |
| monitorexit ×2 | 正常 + 异常路径各一份，由编译器自动织入（finally 语义由字节码结构背书） |
| 同步方法 | 无 monitorenter 指令，flag 隐式处理；异常同样自动解锁 |
| 锁对象 | 实例方法 `this`；静态方法 `Class` 对象；代码块任意引用类型对象（不可为 null、不可为基本类型装箱陷阱如 Integer 缓存值——经典事故源） |

## 10.2 monitorenter 运行时决策（C++ 伪代码，JDK ≤14 完整形态）

```cpp
void monitorenter(oop obj) {
    if (obj->isBiasedLocked()) {
        // 偏向锁路径：检查线程ID，匹配则直接获取
        if (obj->biasedThread() == currentThread) return;
        // 否则撤销偏向锁
        revokeBias(obj);
    }
    
    // 轻量级锁路径：CAS Mark Word → Lock Record
    if (CAS(obj->markWord, DisplacedMarkWord, pointerToLockRecord)) {
        return;  // 成功获取轻量级锁
    }
    
    // 重量级锁路径：膨胀为 ObjectMonitor
    ObjectMonitor* monitor = inflate(obj);
    monitor->enter();  // 可能阻塞
}
```

- JDK 15+ 去掉第一段（偏向路径），`revokeBias` 分支不复存在；这就是"移除偏向锁后 synchronized 更快了吗"的答案来源：**无竞争微基准略慢一点（首次也要 CAS 了），竞争场景普遍持平或更好**。

# 11. ReentrantLock vs synchronized

| 对比 | synchronized | ReentrantLock |
|------|-------------|---------------|
| 实现 | JVM 层面（C++，Mark Word/ObjectMonitor） | JDK 层面（Java，AQS + CAS + LockSupport.park） |
| 锁释放 | 自动（出了作用域，含异常） | 必须手动 unlock（finally） |
| 可中断 | 不可中断（等锁时不响应 interrupt） | 可中断（`lockInterruptibly()`） |
| 超时放弃 | 无 | `tryLock(timeout)` |
| 公平锁 | 非公平 | 可选公平/非公平 |
| 条件变量 | wait/notify 一个条件 | 多个 `Condition`（分组唤醒，生产者-消费者精确通知） |
| 锁绑定条件 | 一个对象一个条件 | 一个锁多个条件 |
| 性能 | JDK 6+ 优化后接近 | 低争用下略优，差距很小 |

```java
// synchronized
synchronized (lock) {
    // 业务代码
}

// ReentrantLock
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    // 业务代码
} finally {
    lock.unlock();  // 必须手动释放
}
```

## 11.1 内部机制一句话（AQS）

- ReentrantLock 继承 AbstractQueuedSynchronizer：`volatile int state` 记重入次数，CAS 改 state 抢锁，失败进 CLH 双向队列 park；`Condition` 各持一个单向条件队列。
- 它的"有序性"依赖 volatile 写读 + CAS（`LOCK CMPXCHG`），正对应 [18_jmm](18_jmm.md) 的 hb 边与第 12 节。

## 11.2 选择准则

| 需求 | 选择 |
|------|------|
| 简单互斥、方法级保护 | synchronized（免费拿异常安全 + JIT 锁优化全家桶） |
| 需要 tryLock/超时/可中断/公平/多 Condition | ReentrantLock |
| 虚拟线程高并发 I/O（JDK 21~23） | 优先 ReentrantLock（避免 pinning，见第 15 节） |
| 读写分离 | ReadWriteLock / StampedLock（超出本篇，属 J.U.C） |

# 12. CAS（Compare And Swap）

- 无锁算法的核心、AtomicXXX 等原子类的实现基础。
- CPU 提供的原子指令（x86：`LOCK CMPXCHG`）。

```java
// AtomicInteger 的 incrementAndGet 底层实现
public final int incrementAndGet() {
    return U.getAndAddInt(this, VALUE, 1) + 1;
}

// Unsafe.getAndAddInt:
public final int getAndAddInt(Object o, long offset, int delta) {
    int v;
    do {
        v = getIntVolatile(o, offset);
    } while (!compareAndSwapInt(o, offset, v, v + delta));
    return v;
}
```

## 12.1 LOCK 前缀到底做了什么

```text
lock cmpxchg：
  ① 锁住目标地址所在 cache line（MESI 独占态，其他核的副本失效）
  ② 原子完成 "比较寄存器 == 内存值？→ 是则写新值并置 ZF，否则仅置 ZF"
  ③ 隐含全屏障语义（等价 mfence 效果）→ 它同时也是 JMM 的 volatile 写实现手段之一
```

## 12.2 CAS 的三大问题

| 问题 | 说明 | 解决 |
|------|------|------|
| ABA 问题 | 值从 A→B→A，CAS 认为没变 | AtomicStampedReference（加版本号）/ AtomicMarkableReference |
| 循环开销大 | 高并发下 CAS 失败 → 自旋 → 所有核在同一缓存行上互相打脸 | 分段（LongAdder）、退避（PAUSE）、@Contended |
| 只能保证一个变量的原子操作 | 多个变量需要原子更新 | AtomicReference 包不可变对象 / 加锁 |

### 12.2.1 ABA 问题的示例与解法

```java
// ABA 问题示例
// 线程1: CAS(A→C)  先取到 A
// 线程2: CAS(A→B) → CAS(B→A)  (A→B→A)
// 线程1: CAS 成功！但它不知道经历过 A→B→A   ← 链表节点复用场景最致命

// 解决方法：AtomicStampedReference
AtomicStampedReference<Integer> ref = new AtomicStampedReference<>(100, 0);
int stamp = ref.getStamp();
ref.compareAndSet(100, 200, stamp, stamp + 1);  // 同时比较值和版本号
```

### 12.2.2 高争用下"循环耗时"：LongAdder 的分段思路

```text
AtomicLong:  16 线程 × 同一缓存行 CAS → 90%+ 时间在失败重试（缓存行乒乓）

LongAdder:   base ──┬── Cell[0] @Contended   ← 每线程按 hash 落一个 Cell
                   ├── Cell[1] @Contended      竞争只发生在"撞同一格"时
                   └── ...（格子随争用扩容到 ≈ CPU 数）
   sum() = base + Σ Cell（最终一致，读时不保证瞬时精确）
```

- `@Contended` 填充防空隙共享（false sharing）的布局细节见 [9_object_layout](9_object_layout.md)。
- 取舍：要"瞬时精确读"仍用 AtomicLong/加锁；要"写极多读极少"用 LongAdder。

### 12.2.3 单变量原子 ≠ 复合操作原子

```java
// 经典反例：x = y + 1 —— 即使对 x 的写入用 CAS，整体依然不原子
int v = y.get();            // ① 读 y 后，别的线程改了 y
x.compareAndSet(old, v + 1);// ② CAS 成功 ≠ 结果正确（基于过期快照）

// 跨两个变量的"原子更新"正解：
// a) 锁；b) 打包成不可变 Pair 用 AtomicReference 换整体；c) 单变量化设计
```

# 13. 实战：验证锁升级与锁争用

## 13.1 JOL 观察锁状态（JDK ≤ 14 语境）

```java
// -XX:+UseBiasedLocking -XX:BiasedLockingStartupDelay=0
// 使用 JOL 观察锁状态变化（JDK 15+ 偏向已移除，只会看到 无锁→轻量→重量）
import org.openjdk.jol.info.ClassLayout;

public class LockUpgrade {
    public static void main(String[] args) throws InterruptedException {
        Object obj = new Object();
        
        System.out.println("===== 无锁状态 =====");
        System.out.println(ClassLayout.parseInstance(obj).toPrintable());
        
        // 偏向锁
        synchronized (obj) {
            System.out.println("===== 偏向锁 =====");
            System.out.println(ClassLayout.parseInstance(obj).toPrintable());
        }
        
        // 轻量级锁：两线程交替
        Thread t2 = new Thread(() -> {
            synchronized (obj) {
                System.out.println("===== 轻量级锁 =====");
                System.out.println(ClassLayout.parseInstance(obj).toPrintable());
            }
        });
        t2.start();
        t2.join();
        
        // 重量级锁：竞争
        synchronized (obj) {
            new Thread(() -> {
                synchronized (obj) {
                    // 竞争
                }
            }).start();
            Thread.sleep(100); // 确保发生竞争
            
            System.out.println("===== 重量级锁 =====");
            System.out.println(ClassLayout.parseInstance(obj).toPrintable());
        }
    }
}
```

```text
JOL 输出尾部速读：
  ... 0 01   → 无锁        ... 1 01 → 偏向
  ... 00     → 轻量级      ... 10   → 重量级
注意：new Object() 后直接打印可能先显示 hash 相关状态；
     调用了 identityHashCode 的对象不可能再偏向（见 2.2）。
```

## 13.2 生产可用的争用观测手段

| 工具 | 用法 | 看到什么 |
|------|------|----------|
| **JFR** | `jcmd <pid> JFR.start settings=profile`；事件 `jdk.JavaMonitorEnter`（含阻塞时长、争用对象、方法栈） | 哪个方法在 monitor 上等了多久（分析用 JMC，见 [21_visual_tools](21_visual_tools.md)） |
| **jstack** | `jstack <pid> \| grep -c BLOCKED`；看 `waiting to lock <0x...>` 与被 `held by thread` 指向的栈 | 当下谁堵谁、锁地址聚集度 |
| **async-profiler** | `asprof -e lock -d 60 <pid>`（lock 事件基于对象监视器采样） | 争用热点方法火焰图 |
| 死锁检测 | `jstack` 尾部自带 `Found one Java-level deadlock`；`jcmd <pid> Thread.print` | 循环等待链 |

> 完整排障路径（BLOCKED 风暴、CPU 100% 的锁分析）见 [22_troubleshooting](22_troubleshooting.md)。

# 14. 锁相关 JVM 参数

```bash
# 偏向锁（历史）
-XX:+UseBiasedLocking            # JDK 13 起默认关，15 废弃（JEP 374），18 起参数被移除——见到警告勿慌
-XX:BiasedLockingStartupDelay=0  # 启动不延迟偏向宽限期（历史）

# 自旋（历史）
-XX:+UseSpinning                 # JDK 7 起移除
-XX:PreBlockSpin=10              # 早已废弃，改由自适应自旋全自动管理

# 重量级锁直通（测试用）
-XX:+UseHeavyMonitors            # 禁用偏向/轻量级，synchronized 直接膨胀重量级——仅用于对照实验

# JIT 锁优化（默认开，一般别动）
-XX:+EliminateLocks              # 锁消除 + 锁粗化总开关（C2）

# 虚拟线程时代新增（系统属性，非 -XX）
-Djdk.tracePinnedThreads=full    # JDK 21~23：打印 synchronized 钉住载体线程的栈（JEP 491 后需求大减）
```

# 15. 虚拟线程与 synchronized（JDK 21+）

## 15.1 机制与演进

### 15.1.1 pinning：synchronized 把载体线程一起拖下水

- 前提：虚拟线程（JEP 444）由少量**载体线程（carrier，即平台线程）**调度执行；虚拟线程在 `LockSupport.park`/j.u.c 锁上让出时，载体线程可以去跑别的虚拟线程——这是百万并发的本钱。
- 问题：JDK 21 中 **monitor 的持有者记录在对象头/ObjectMonitor 里，而它们只认平台线程**。虚拟线程若**在 synchronized 块内阻塞**（等 I/O、等锁、sleep），它占用的载体线程无法卸载 → **pinned**：

```text
JDK 21 争用场景：
虚拟线程 vt-1: synchronized(m){ socket.read()... }   → 挂起，但 monitor owner = carrier#3
载体线程 carrier#3: 被钉死 → 本可服务成千上万 vt 的线程在陪一个网络 RTT 睡觉
后果:  载体线程池(默认≈CPU核数) 迅速耗尽 → 吞吐雪崩，退化成"比平台线程还差"
```

- 注意触发条件是"**临界区内发生可让出事件**"，纯计算的短 synchronized 没问题。

### 15.1.2 官方态度与演进

| 版本 | 状态 |
|------|------|
| JDK 21~23 | 官方规约：**高争用/阻塞场景改用 ReentrantLock**——AQS 的 park 路径对虚拟线程友好（unpark 即让出载体） |
| JDK 24 | **JEP 491：Synchronize Virtual Threads without Pinning** —— monitor 记录虚拟线程身份，sync 块内阻塞不再钉载体（个别边角场景如 JNI 仍会 pin） |

### 15.1.3 实操建议与观测

```bash
# JDK 21 排查 pinning：
-Djdk.tracePinnedThreads=full        # 启动参数；钉住时打印整条虚拟线程栈
# 或 JFR 事件：jdk.VirtualThreadPinned（阈值默认 20ms）
jcmd <pid> JFR.start settings=profile
```

| 准则 | 理由 |
|------|------|
| 库代码（你不知道调用方是否虚拟线程）优先非监视器同步 | ReentrantLock/信号量对 Loom 始终友好 |
| synchronized 临界区保持"短、不阻塞" | 任何版本都受益：锁持有时间 ≈ 争用面积 |
| 应用升 JDK 24+ 后可放宽回 synchronized | JEP 491 解决绝大部分 pinning |
| 结构化并发/`StructuredTaskScope` 控制扇出 | 见并发库笔记方向；排障联动 [22_troubleshooting](22_troubleshooting.md) |
