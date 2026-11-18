# 1. 方法内联（Method Inlining）

## 1.1 为什么内联是"优化之母"

- 将目标方法的代码复制到调用处，消除调用开销（栈帧创建、参数传递、返回跳转）。
- **更本质的价值：内联打开了后续优化的视野**。逃逸分析、CSE、标量替换、锁消除全都以"调用点与方法体在同一编译单元"为前提——不内联，C2 的分析范围就被调用边界切碎。

```java
// 内联前
public int calc() {
    return add(1, 2);                 // 方法调用
}
public int add(int a, int b) {
    return a + b;
}

// 内联后（等效）
public int calc() {
    return 1 + 2;                     // 内联展开 → 常量折叠为 3
}
```

```text
不内联：  [calc 能看到的优化空间]     [add 能看到的优化空间]   ← 各自为战
内联后：  [■■■■■■ 合并后的巨大优化窗口 ■■■■■■]                ← CSE/EA/折叠全解锁
```

> 经验法则：**小而热的方法最该被内联**，"为了性能把方法写长"是反模式——写小方法 + 依赖 JIT 内联才是 Java 的正确姿势。

## 1.2 内联决策：三档成本阈值模型

### 1.2.1 按调用点热度分三档

HotSpot 按**调用点热度**把候选分成三档，套用不同的大小上限（字节码字节数）：

| 档位 | 判定 | 可用阈值 | 相关参数 |
|------|------|----------|----------|
| Hot（热调用点） | 所在方法已达 C2 编译热度 | 可到 **325** 字节 | `-XX:FreqInlineSize=325` |
| 一般 | 普通调用点 | **35** 字节以内 | `-XX:MaxInlineSize=35` |
| Marginal（边缘） | 极小方法（约 10 字节内），单独不值得内联，攒够次数后合并考虑 | ~10 字节 | 由 FreqInlineSize 流程回收 |

### 1.2.2 总量与深度预算

再叠加**总量/深度限制**，防止代码爆炸：

| 参数 | 默认 | 含义 |
|------|------|------|
| `-XX:MaxInlineLevel` | JDK 8 为 9，JDK 9+ 为 **15** | 内联嵌套最大层数（A 调 B 调 C… 的链深） |
| `-XX:InlineSmallCode` | 非分层 1000；分层 C2 下 **2500** | 已编译目标方法的**机器码**小于此值才允许内联 |
| `-XX:MaxInlineSize` | 35 | 未达热度的调用点字节码上限 |
| `-XX:FreqInlineSize` | 325 | 热点调用点字节码上限 |

### 1.2.3 不可内联的常见情形

- 未加载类的调用点（首次调用时先记录，第二次再决策）；
- 虚方法且类型剖析显示多态（见 1.4）；
- `@Don'tInline` / native / 含 `jsr` 的古老字节码等硬性禁止项；
- 超过上表任一大小预算。

## 1.3 PrintInlining 输出解读

### 1.3.1 开关与输出形态

```bash
-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining
```

```text
@ 123  java.lang.String::hashCode (55 bytes)
  @ 130  java.lang.String::charAt (25 bytes)   inline (hot)
  @ 145  java.lang.String::length (11 bytes)   inline (hot)
  @ 160  com.foo.Bar::check (110 bytes)        too big
  @ 176  com.foo.Baz::run (812 bytes)          hot method too big
```

### 1.3.2 关键词与对策

| 输出关键词 | 含义 | 处理思路 |
|------------|------|----------|
| `inline (hot)` | 热点调用点，成功内联 | 健康 |
| `already compiled into a small method` | 目标已编译且够小，直接内联编译产物 | 健康 |
| `too big` | 字节码超 MaxInlineSize/FreqInlineSize | 拆小热点方法体 |
| `hot method too big` | 很热但仍超 325 上限被拒 | 重点优化对象：把热路径抽成小方法 |
| `not inlineable` | 硬性禁内联（native/synchronized 特例等） | — |
| `already compiled into a big method` | 目标编译产物超 InlineSmallCode | 调大 InlineSmallCode 或拆方法 |
| `executing <init> method` 等 | 特殊语义限制 | — |
| 缩进层级 | 内联树：越深表示嵌套层级越高（对照 MaxInlineLevel） | — |

## 1.4 虚方法内联（Devirtualization）

### 1.4.1 三态判定表

- JIT 遇到虚方法调用时，需要先确定**实际接收者类型**才能内联，依据是 Level 3 攒下的类型剖析（`MethodData` 的 receiver type profile）。
- 剖析结果按形态处理：

| 调用点形态 | 类型数 | JIT 策略 |
|------------|--------|----------|
| 单态 Monomorphic | 1 | **直接内联** + 插入类型检查 guard（CHA 假设"该类无其他实现被加载"） |
| 二态 Bimorphic | 2 | 两条内联分支 + 类型判断 |
| 多态 Megamorphic | ≥3 | 放弃内联，退化为 vtable/itable 虚调用（或内联缓存查表） |

### 1.4.2 带 guard 的内联与 CHA 去虚化

```java
// 多态调用
interface Animal { void sound(); }
class Dog implements Animal { public void sound() { ... } }
class Cat implements Animal { public void sound() { ... } }

public void makeSound(Animal a) {
    a.sound();  // 虚方法调用
}

// JIT profiling 发现历史 100% 是 Dog → 编译为：
public void makeSound(Animal a) {
    if (a.getClass() != Dog.class) trap();   // guard，失败去优化
    // Dog.sound() 已内联在此
}
```

- guard 检查的开销是一条 `cmp+je`，换来整个调用点消失——这就是"带假设优化 + 去优化兜底"的典型样板（见第 11 节）。
- **CHA（类层次分析）**：若 JVMCI/全局发现某虚方法只有一个实现，即使剖析数据不足也可去虚化内联，代价是后续加载新实现类时**全体相关代码去优化**，因此策略保守。

## 1.5 JVMCI 与可插拔编译器（一句话）

- JVMCI 让 Graal 这样的 Java 实现编译器接管内联等决策（JDK 10 实验，JEP 317），**JDK 21 已移除**（JEP 410，详见 [16_execution_engine](16_execution_engine.md) 4.5），HotSpot 语境下内联策略即 C1/C2 内置策略。

# 2. 逃逸分析（Escape Analysis）

> 分配层面的完整机制（栈上分配/标量替换/部分逃逸）详见 [15_memory_allocation](15_memory_allocation.md)。本节强调它在**编译优化链**中的位置。

## 2.1 三种逃逸程度

JIT 判断一个对象是否"逃逸"：

- **不逃逸**：对象只在一个方法内使用 → 栈上分配 + 标量替换 + 同步消除
- **方法逃逸**：对象作为参数传递给其他方法
- **线程逃逸**：对象被其他线程访问

```java
// 不逃逸 → 可优化
public void localOnly() {
    Object o = new Object();  // 线程内方法内，完全不逃逸
}

// 方法逃逸 → 不可栈上分配，但可消除锁
public void methodEscape(StringBuilder sb) {
    sb.append("hello");       // sb 从外部传入，有方法逃逸
}

// 线程逃逸 → 完全不可优化
public void threadEscape(Object o) {
    globalField = o;          // 赋值给静态变量，线程逃逸
}
```

## 2.2 与内联的联动关系

```text
new Foo().bar() 写在不同方法里
        │ 没有内联：EA 在调用点边界失明，对象"看起来"逃逸
        ▼
内联把 bar() 拉进同一编译单元 → EA 视野扩大 → 分配消失
```

- **逃逸分析使"跨方法创建的对象"也能被优化掉**——优化效果对代码组织方式敏感，这正是内联排第一的原因。
- **标量替换后"分配消失"**：对象字段拆成寄存器/栈槽局部量，不仅省一次 GC 压力，还把"堆内存屏障/写后同步"从依赖图中移除，给后续 CSE、锁消除腾出证明空间。

```bash
-XX:+DoEscapeAnalysis        # C2 默认开（C1 不做）
-XX:+EliminateAllocations    # 标量替换（依赖 EA）
```

# 3. 锁消除（Lock Elimination）

JIT 通过逃逸分析发现同步块被对象**线程独占**，则整锁删除（synchronized 连轻量级锁的 CAS 都不做）。

```java
// 同步消除前
public String build() {
    StringBuffer sb = new StringBuffer();  // 局部变量，不会逃逸
    sb.append("a");  // synchronized
    sb.append("b");  // synchronized
    return sb.toString();
}

// 同步消除后，等效于使用非同步的 StringBuilder
public String build() {
    StringBuilder sb = new StringBuilder();
    sb.append("a");
    sb.append("b");
    return sb.toString();
}
```

- 判定链条：`EA 证明 sb 不逃逸出本线程` → `锁不可能被他人竞争` → `monitorenter/exit 成对删除`。
- 一旦对象被赋给字段/静态变量/作为参数外传，消除立即失效——**别把正确性寄托在锁消除上**（逃逸判断随版本/内联结果变化）。
- 锁状态机与消除的上下游关系详见 [19_lock_optimization](19_lock_optimization.md) 第 9 节。

```bash
-XX:+EliminateLocks          # 默认开（诊断日志需 -XX:+PrintEliminateLocks，product build 常无输出）
```

# 4. 锁粗化（Lock Coarsening）

连续的"加锁-解锁-加锁-解锁"若中间**没有线程可能切换到有意义争用**，合并为一次加锁。

```java
// 锁粗化前
for (int i = 0; i < 1000; i++) {
    synchronized (lock) {     // 每次循环都加锁/解锁，1000次
        counter++;
    }
}

// 锁粗化后（等效）
synchronized (lock) {         // 整个循环只加锁一次
    for (int i = 0; i < 1000; i++) {
        counter++;
    }
}
```

```text
优化前 IR:  enter → exit → enter → exit → enter → exit   ← 每次 enter 都有 CAS
优化后 IR:  enter ────────────────────────────── exit    ← 一次 CAS
```

- 动机：反复加解锁让轻量级锁的 CAS/栈上 Lock Record 读写白白循环，且可能诱发假性膨胀为重量级锁。
- 粗化范围有限：JIT 只在**相邻/近距离**的锁对上操作，不会把整段不相干代码圈进去——想要大临界区请自己在源码层加。
- 相关机制（偏向/轻量级的成本模型）见 [19_lock_optimization](19_lock_optimization.md) 第 8 节。

# 5. 公共子表达式消除（CSE）

同一表达式在两个值必经的控制流上重复出现且期间无写入，则只算一次。

```java
// 优化前
int a = (b + c) * d + (b + c) * e;
// 编译后
int tmp = b + c;       // 公共子表达式只计算一次
int a = tmp * d + tmp * e;
```

字节码视角的伪对比：

```text
优化前：                      优化后：
  iload b                       iload b
  iload c                       iload c
  iadd                          iadd
  iload d                       istore tmp   ← 结果留存
  imul                          iload d
  iload b                       imul
  iload c                       iload tmp
  iadd   ← 重复计算！             iload e
  iload e                       imul
  imul                          iadd
  iadd
```

- C2 用 **GVN（全局值编号）**在 Sea-of-Nodes 图上识别等价值，跨基本块合并。
- **依赖控制流分析**：若两次出现之间可能写入 `b`/`c`（含方法调用副作用），表达式不"公共"，必须保留两份——这也是内联影响 CSE 效果的又一例。
- 经典面试陷阱：

```java
int x = 0, y = 0;
synchronized (this) { x = 1; }
synchronized (this) { y = x + x; }
// 有人以为两次锁内读 x 可 CSE 合并——错，中间虽有"无写"，
// 但其他线程可能在两个同步块之间修改 x；JIT 只能在能证明独占时合并。
```

# 6. 数组边界检查消除（Range Check Elimination）

Java 每次数组访问都要检查索引越界（`if (i < 0 || i >= arr.length) throw`），热点循环里这是实打实的分支开销。

## 6.1 消除原理与写法策略

### 6.1.1 归纳变量证明链

```text
for (int i = 0; i < arr.length; i++) sum += arr[i];

C2 证明链：
  ① 循环不变式：0 ≤ i < arr.length（由初始化+递增+终止条件推出）
  ② 访问点检查条件 i < arr.length ∧ i ≥ 0 恒真
  ③ 删除循环内全部 RangeCheck 节点（Loop Predicate：把证明放到循环外做一次）
```

### 6.1.2 写法有效性对比

| 写法 | 能否消除 | 说明 |
|------|----------|------|
| `for (int i = 0; i < arr.length; i++) arr[i]` | ✅ | 标准模式，归纳变量直接可证 |
| `for (int i = 0, n = arr.length; i < n; i++)` | ⚠️ 多数可 | 把 length 存局部变量**可能切断**"n 恒等于当前数组长度"的证明（数组引用若可能中途变化），别迷信 |
| `for (int i = 0; i < n && i < arr.length; i++)` | ✅ | 历史技巧：`&&` 复合条件让编译器把 length 检查并入（原 `-XX:+UseLoopPredicate` 思路，JDK 9+ 已并入常规范围检查消除，无需设置） |
| 增强 for / `Arrays.setAll` 等 | ✅ | 模式匹配友好 |
| `if (i >= 0 && i < arr.length) arr[i]` 后同块多次访问 | ✅ | **Range Check 合并**：首个显式检查建立证明，后续隐式检查共享之 |
| `arr[i]`，`i` 来自外部输入无约束 | ❌ | 证不出界 |

### 6.1.3 最佳实践

让索引与 `arr.length` 保持**同一表达式来源**的简单循环；批量访问时先做一次显式范围检查再连续使用（"range-check 模式"）；越界代价高的热路径考虑 `Math.clamp`/提前分支。

## 6.2 观察与微基准陷阱

```bash
# C2 IR 层面（需 debug/fastdebug JDK 构建）
-XX:+UnlockDiagnosticVMOptions -XX:+PrintOptoAssembly
# 输出中搜索：无 "RangeCheck" 节点行附着于 load/store 即已消除

# 常规 JDK：LogCompilation XML 里查 <loop_predicates>/<arrayload> 决策
-XX:+LogCompilation -XX:LogFile=hotspot.log    # 配 JITWatch 可视化
```

- 微基准陷阱：JMH 里"填满数组再读"若无 `blackhole` 消费结果，整个循环可能被**死代码消除**，测出 0 ns/op 的假速度（JMH 骨架见第 12.1 节）。

# 7. 循环优化

## 7.1 循环展开（Loop Unrolling）

### 7.1.1 展开前后对比

```java
// 展开前
for (int i = 0; i < 100; i++) {
    sum += a[i];
}

// 展开后（等效，减少分支预测和循环控制开销）
for (int i = 0; i < 100; i += 4) {
    sum += a[i];
    sum += a[i+1];
    sum += a[i+2];
    sum += a[i+3];
}
```

### 7.1.2 收益模型与展开策略

- 收益模型：分支/自增指令数 ÷ 展开因子；同时给指令级并行（ILP）让出多条独立加法链。
- 策略：**小循环（次数少且固定）完全展开**成直线代码；大循环按因子展开并**剥离余数**（peeling）处理 `n % factor ≠ 0`。
- 参数：`-XX:LoopUnrollLimit`（默认 60，循环体节点数上限，超了不展开）；展开过度会撑爆 CodeCache 并破坏 I-Cache 局部性——不是越大越好。

## 7.2 循环不变代码外提（LICM）

```java
// 优化前
for (int i = 0; i < n; i++) {
    double val = Math.sqrt(2.0) * i; // sqrt(2.0) 每次都计算
}

// 优化后（等效）
double sqrt2 = Math.sqrt(2.0);   // 提到循环外，只计算一次
for (int i = 0; i < n; i++) {
    double val = sqrt2 * i;
}
```

- 前提：表达式在循环所有迭代中值不变**且无副作用**（有副作用的外提改变语义，CSE/别名分析负责证明）。
- 与内联联动：循环体内调用的小方法先内联，其中的不变部分才可能被识别出来。

## 7.3 循环判断外提（Loop Unswitching）

```java
// 优化前
for (int i = 0; i < 1000; i++) {
    if (flag) {          // 循环内条件恒定
        doA(i);
    } else {
        doB(i);
    }
}

// 优化后：生成两个"无分支"循环版本
if (flag) {
    for (int i = 0; i < 1000; i++) { doA(i); }
} else {
    for (int i = 0; i < 1000; i++) { doB(i); }
}
```

- 意义：循环内每次迭代省一个分支，且各版本内部可继续被向量化/SIMD（分支是向量化的头号敌人）。

## 7.4 计数循环与安全点（了解）

- C2 认定满足"单向、固定步长、可证终止"的循环为 **Counted Loop**，剥离中间安全点轮询以降低开销；
- 极端大次数循环可能延迟 STW，`-XX:+UseCountedLoopSafepoints` 可强制在循环中段插桩（历史默认关闭，代价是吞吐略降）——GC 停顿异常且卡在长循环时可与 [14_gc_tuning](14_gc_tuning.md) 联动排查。

# 8. 空值检查消除（Null Check Elimination）

```java
// 优化前：每次访问都检查 null
obj.field1 = 1;
obj.field2 = 2;
obj.field3 = 3;

// JIT 优化后：只检查一次
if (obj == null) throw NPE;  // 一次检查
obj.field1 = 1;              // 后续不再检查
obj.field2 = 2;
obj.field3 = 3;
```

- 机制本质是 **NCE = 隐式检查 + CSE**：
  - 首次解引用处保留检查（x86 上常靠访问 0 页触发 SIGSEGV 的**隐式空检查**，零指令开销，故障处理器翻译为 NPE）；
  - 控制流上必经该点后，"obj ≠ null"成为可证明的事实，后续检查作为公共子表达式删除。
- **类型加载检查同理消除**：对象头的 klass 指针非空即蕴含"类已完成加载"，解释器/编译器插入的类加载检查在 JIT 代码里常被这类蕴含关系干掉（关联 [7_method_area](7_method_area.md) 的类初始化检查）。
- 反例提醒：`if (o != null) { ... }` 之外的路径（如另一线程置空字段后再读）不共享证明——**字段可被他人改，局部变量才稳**。

# 9. 窥孔优化（Peephole Optimization）

- 发生位置：**IR → 机器码之后、写出 CodeCache 之前**的指令级局部改写（也内嵌在指令选择器里）。
- 只看窗口内几条指令，不关心全局语义：

| 模式 | 改写 |
|------|------|
| `mov r1, r2` 后紧跟同寄存器搬移 | 消除冗余 mov |
| `add r, 0` / `mul r, 1` | 删除恒等运算 |
| `cmp` + `je` + 跳转目标重合 | 合并为一次条件跳转（branch fusion） |
| `lea (base,index,scale)` | 用一条寻址指令替代 shl+add 组合 |
| 连续 `push/pop` 同寄存器 | 精简栈操作 |

- 寄存器分配（C1 线性扫描 / C2 图着色）决定"值留在哪"，窥孔负责收尾擦除分配产生的搬运噪声；配合第 6 节看：**边界检查、空检查删除发生在 IR 层，窥孔发生在汇编层**，两层共同决定最终指令密度。

# 10. 分支预测与代码布局

- 现代 CPU 靠分支预测器维持流水线；JIT 能做的是**把代码摆成预测器友好的样子**：

```java
// profiling 显示 error 极少发生
if (error) {
    handleError();  // 罕见路径 → 冷块挪到方法末尾，用跳转进入
}
// 正常路径 → 顺序 fallthrough，不跳转，流水线友好
processNormal();
```

| 信息来源 | 用途 |
|----------|------|
| Level 3 的分支剖析（taken/not-taken 计数） | 热出边设为 fallthrough，冷出边重排到函数尾部（并常转 uncommon trap） |
| 无剖析数据时的静态启发式 | **向后跳（循环回边）倾向 taken、向前跳倾向 not-taken**；异常处理路径视为冷 |
| `ProfileInterpreter` 收集的数据流经 `MethodData` 传入 C2 | 决定 unswitching、guard 布局、循环展开取舍 |

- 业务侧启示：把**预期常见分支写在前面**、少用"否定条件嵌套"，能帮助剖析数据快速收敛到单态形态（关联 1.4 的类型剖析）。

# 11. 激进优化与去优化的互动

## 11.1 JIT 敢激进的前提：假设 + 退路

| 激进假设 | 失败信号 | 退路 |
|----------|----------|------|
| CHA：类层次封闭，虚方法只有一个实现 | 之后加载了新的子类/实现类 | 全局/局部 deopt，重编译或降级 |
| 类型剖面单态：调用点只见过 Dog | 传入 Cat | guard 命中 → uncommon trap |
| 分支 never-taken：error 路径 0 次 | 第一次走进 error | 该路径直接编译为 trap 桩 |
| 数组必然非空/界内 | 极端输入 | 隐式异常或显式 trap |

```java
public void process(Object obj) {
    if (obj instanceof String) {
        String s = (String) obj;
        // JIT 曾假设 obj 总是 String，直接内联 String 路径
    }
}
```

```text
运行期时间线：
  C2 编译(假设单态 String) → 高速执行 → 首次传入 Integer
      → guard 失败，uncommon trap（一条 ud2/int3 类指令）
      → 帧被"反物化"回解释器视角（虚拟帧重建局部变量）
      → MDO 记录新类型（多态）→ 计数回升
      → 再次编译：改为双态内联或放弃内联（可能比上一版慢，但不再反复横跳）
```

- **为什么 JIT 可能比 AOT 快**：AOT 编译期看不到真实类型分布/分支概率，只能生成保守的通用分支链；JIT 用运行期反馈"赌"最可能路径，赌错也只损失一次去优化成本。

## 11.2 去优化风暴的排查

```bash
-Xlog+deopt                     # JDK 9+：打印每次 deopt 的 reason 与位置
-XX:+PrintDeoptimizationEvents  # 旧式开关
```

```text
典型日志：
  <deopt reason="unloaded or not initialized" action="reinterpret" ...>
  Uncommon trap: trap_request=0xffffff4d  reason=class_check
```

| 定位套路 | 动作 |
|----------|------|
| 同一 `method+bci` 反复 deopt/recompile | 检查该处是否类型抖动（多态混合输入）、是否被 `-XX:RecompileLimit` 卡死后终身解释 |
| `reason=range_check` 频繁 | 循环写法破坏了边界证明（回看 6.1.2） |
| `reason=unstable_if` | 分支概率反复翻转，通常可容忍；量大则审视热路径 if 设计 |
| LogCompilation + JITWatch 看 deopt 曲线 | 关联到具体编译事件（工具流程见第 12 节） |

# 12. 实战观察

## 12.1 JMH 最小完整示例

### 12.1.1 依赖与基准骨架

Maven 依赖：

```xml
<dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-core</artifactId>
    <version>1.37</version>
</dependency>
<dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-generator-annprocess</artifactId>
    <version>1.37</version>
    <scope>provided</scope>
</dependency>
```

```java
@BenchmarkMode(Mode.AverageTime)          // avgt：单次操作平均耗时
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Thread)
@Warmup(iterations = 5, time = 1)         // 预热必须足够长，让分层编译爬到 Level 4
@Measurement(iterations = 5, time = 1)
@Fork(1)
public class JITBenchmark {
    private int a = 10;
    private int b = 20;

    @Benchmark
    public int testInline() {
        return add(a, b);      // 会被内联（甚至常量折叠）
    }
    private int add(int x, int y) { return x + y; }

    @Benchmark
    public int testNoInline() {
        return addComplex(a, b);   // 可能不会内联（更复杂）
    }
    private int addComplex(int x, int y) {
        // 大量代码...超过内联阈值
        return x + y;
    }
}
```

### 12.1.2 输出解读

```text
Benchmark                 Mode  Cnt    Score    Error  Units
JITBenchmark.testInline   avgt    5    0.312 ±  0.021  ns/op
JITBenchmark.testNoInline avgt    5    1.847 ±  0.102  ns/op
# Score=平均值 Error=99.9%置信区间
# Error 过大：预热不足/环境噪声，加大 Warmup 或 Fork
```

- JMH 防"测了个寂寞"：返回值必须被消费（防 DCE，见 6.2）、`@Fork` 起独立 JVM 防编译缓存污染——这些正是理解 JIT 行为后才能明白的设计。

## 12.2 编译活动观察工作流

### 12.2.1 实时观察：谁被编译、为何没内联

```bash
# ① 谁被编译了、哪一层：轻量实时
java -XX:+PrintCompilation MyApp

# ② 内联决策：为什么没内联
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining MyApp
```

### 12.2.2 LogCompilation + JITWatch 与 JFR

```bash
# ③ 全量 XML 日志 → JITWatch（AdoptOpenJDK 出品）可视化：
#    编译时间线、内联树、deopt 事件、剖析值一览
java -XX:+UnlockDiagnosticVMOptions -XX:+LogCompilation \
     -XX:LogFile=hotspot.log MyApp
# 注意需 -XX:+TraceClassLoading 同场时才能完整解析；JITWatch 导入 hotspot.log + classpath 即可

# ④ JFR 也可采编译统计（低开销，适合生产常开）
java -XX:StartFlightRecording=settings=profile MyApp
```

## 12.3 看真实机器码：hsdis + PrintAssembly

```bash
# 前置：下载与 JDK 匹配的 hsdis 动态库放入 <jdk>/bin（Server JRE 不附带）
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly \
     -XX:CompileCommand=print,com.foo.Bar::hotMethod MyApp > out.asm
```

- 输出为 Intel/AT&T 语法汇编 + 注释（源码行、deopt 桩、stub 标记）；
- 用它验证：6.2 的 RangeCheck 是否真的没了、1.4 的 guard 是否只有两条指令、8 的隐式空检查是否利用 0 页。
- 只打印指定方法（`CompileCommand=print,类::方法`），全量打印输出可达数百 MB。

# 13. 优化参数速查

```bash
# 内联相关
-XX:MaxInlineSize=35             # 普通方法内联阈值（字节码大小）
-XX:FreqInlineSize=325           # 热点方法内联阈值
-XX:MaxInlineLevel=15            # 最大内联深度（JDK 8 默认 9，JDK 9+ 默认 15）
-XX:InlineSmallCode=2500         # 已编译方法机器码 < N 可再内联（非分层默认 1000）

# 逃逸分析与分配
-XX:+DoEscapeAnalysis            # 逃逸分析
-XX:+EliminateAllocations        # 标量替换
-XX:+EliminateLocks              # 锁消除（兼锁粗化总开关）

# 循环与边界检查
# -XX:+UseLoopPredicate          # 已移除：能力并入 C2 范围检查消除
-XX:+UseCountedLoops             # 可计数循环优化
-XX:LoopUnrollLimit=60           # 循环展开的节点数限制

# 编译日志
-XX:+PrintCompilation            # 编译日志
-XX:+PrintInlining               # 内联日志
-XX:+UnlockDiagnosticVMOptions   # 解锁诊断参数（上述多数的前置）
-XX:+LogCompilation              # XML 编译日志（jitwatch 输入）
-XX:+PrintOptoAssembly           # C2 IR/装配伪输出（需 fastdebug 构建）
-XX:+PrintAssembly               # 真机器码（需 hsdis）
-Xlog+deopt                      # JDK 9+ 去优化事件日志
```
