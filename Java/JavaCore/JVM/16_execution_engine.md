# 1. 执行引擎概览

## 1.1 执行引擎的定位

- JVM 的执行引擎负责把 Class 文件中的**字节码指令**变成 CPU 真正执行的机器码。
- HotSpot 是"解释器 + JIT 编译器"的**混合执行引擎**，这也是 "HotSpot"（热点探测）名字的由来。

```text
字节码(bytecode) → 执行引擎 → 本地机器码 → CPU 执行
                      │
         ┌────────────┼────────────┐
         │            │            │
   解释器(Interpreter)  JIT编译器  GC
     逐条解释执行    热点代码编译
```

## 1.2 三种执行方式对比

| 维度 | 解释执行 | JIT（运行期编译） | AOT（提前编译） |
|------|----------|------------------|----------------|
| 产物生成时机 | 不生成，逐条翻译即执行 | 运行中检测到热点后编译 | 打包/安装阶段编译为机器码 |
| 启动速度 | **最快**（加载即跑） | 慢（要等编译+预热） | 快（直接跑机器码） |
| 峰值性能 | 低（无深度优化空间） | **最高**（基于运行期剖析做激进优化+可去优化） | 中（缺少运行期反馈，优化保守） |
| 内存开销 | 几乎为零 | CodeCache 驻留机器码 | 二进制文件更大 |
| 典型代表 | `-Xint` | C1 / C2（分层编译） | GraalVM Native Image；JDK 9 曾实验原生 AOT（JEP 295 的 jaotc），后已被废弃移除 |

> JIT 峰值性能能超过 AOT 的根本原因：编译决策依赖**运行期类型剖析 / 分支概率**，且假设失败可去优化重来。详见 [17_jit_optimization](17_jit_optimization.md) 第 11 节。

## 1.3 混合模式工作流（默认）

```text
启动阶段：解释器快速响应
     ↓
运行时发现热点代码（计数器超标，见第 3 节）
     ↓
JIT 后台编译线程异步编译（应用线程不等待）
     ↓
方法下一次调用（或循环回到循环头）时切入机器码
     ↓
运行期假设被打破 → 去优化回解释器，重新剖析再编译
```

```bash
# 查看当前模式
java -version
# Java HotSpot(TM) 64-Bit Server VM (mixed mode)

# 强制纯解释
java -Xint -version
# Java HotSpot(TM) 64-Bit Server VM (interpreted mode)

# 强制纯编译
java -Xcomp -version
# Java HotSpot(TM) 64-Bit Server VM (compiled mode)
```

> `-Xcomp` 启动反而经常**更慢**：所有方法（含只执行一次的）都强制编译，编译时间远超收益——这正说明"解释+JIT 分工"的必要性。

## 1.4 栈式执行器与栈帧

字节码是**栈架构**指令集：操作数放在操作数栈上，而非寄存器中。

```text
bytecode:  iconst_1        // 1 压栈
           iconst_2        // 2 压栈
           iadd            // 弹出两个、相加、结果压栈
           istore_1        // 弹栈存入局部变量表 slot1

┌──────────── 栈帧（见 8_vm_stack.md）────────────┐
│ 局部变量表 [0][1][2]…      操作数栈 …│ 2 │ 1 │ ←栈顶
└──────────────────────────────────────────────────┘
```

| 设计选择 | 优点 | 代价 |
|----------|------|------|
| 栈式字节码 | 指令短、跨平台、JIT 易映射到任意 CPU 寄存器结构 | 单条指令信息量少，纯解释执行慢（频繁压栈弹栈） |
| 寄存器式字节码（如 Dalvik） | 指令数少、解释稍快 | 指令编码复杂 |

> 栈帧结构、方法调用与 `iadd` 类指令的逐条执行细节详见 [8_vm_stack](8_vm_stack.md)。

# 2. 解释器与 JIT 的协作

## 2.1 为什么"有了 JIT 还必须留着解释器"

### 2.1.1 职责一：省掉启动编译开销

类一加载就能执行，避免"为只跑一次的方法付编译时间"——冷代码全程解释，总耗时反而更低。

### 2.1.2 职责二：给 JIT 当剖析数据的采集器

解释执行的每一轮都在累积方法调用计数、回边计数；分层编译下 Level 0~3 还要在 `MethodData` 里记录分支概率、类型剖面——没有这些反馈，C2 的激进优化无从谈起（见 [17_jit_optimization](17_jit_optimization.md)）。

### 2.1.3 职责三：JIT 停摆与去优化的兜底落点

- CodeCache 满 / 编译队列积压 → 热点方法继续留在解释器（见 9.1）；
- 激进假设失败触发去优化 → 代码**退回解释器**从"虚拟帧"恢复执行，重新收集剖析（见 9.3）。

```text
             ┌──────────────────────────┐
   字节码 ──▶│ 解释器：立即执行 + 攒剖析 │──剖析数据──▶ JIT 编译决策
             └───────────▲──────────────┘                    │
                         │ 去优化回退 / 编译未完成兜底          ▼
                         └───────────────────────── CodeCache 机器码
```

## 2.2 模板解释器（Template Interpreter）

HotSpot 默认的解释器不是"逐条 switch-case 翻译"的朴素实现，而是**模板解释器**：

- 启动时为**每条字节码指令**在内存里生成一小段 x86/aarch64 汇编模板（源码中 `templateTable_x86.cpp`），字节码 ↔ 模板一一对应。
- 执行 = 按 PC 取字节码 → 跳到对应模板 → 模板直接压栈/弹栈/调用运行时。
- 单条指令的时间复杂度 ≈ **该模板的机器指令条数**（简单指令几条、复杂指令几十条），而朴素 C++ 解释器是 switch 分发 + 函数调用，常数大得多。

## 2.3 为什么 HotSpot 曾需要两种解释器

| 对比项 | 字节码解释器（Bytecode Interpreter） | 模板解释器（Template Interpreter） |
|--------|-------------------------------------|-----------------------------------|
| 实现方式 | 纯 C++，一个 switch-case 大循环 | 每指令一段生成的汇编模板 |
| 开发维护成本 | 低（与平台无关，写一遍） | 高（每个 CPU 架构一套模板） |
| 执行速度 | 慢 | 快数倍 |
| 用途 | 移植新平台初期的占位实现、诊断兜底 | 生产默认 |

- 新平台移植 HotSpot 时先带字节码解释器保证"能跑"，再逐步补汇编模板。
- 可用诊断参数强制切换（product build 中受限）：`-XX:-UseTemplateTableInterpreter`。

# 3. 热点探测

HotSpot 采用**基于计数器的热点探测**，没有使用采样法。

## 3.1 为什么不用采样法

| 对比 | 计数器探测 | 采样探测（如 JVMPI Sampling / async-profiler） |
|------|-----------|-----------------------------------------------|
| 原理 | 精确统计调用/回边次数 | 定时器周期抓栈，统计出现频率 |
| 精度 | 方法级精确（哪怕调用极少也能发现） | 依赖频率，低频热点易漏 |
| 开销 | 计数器自增，但**每次调用都要付** | 采样间隙零开销，但需安全点式抓栈 |
| 实现复杂度 | 高（要维护半衰期等热度模型） | 相对低 |
| 结论 | HotSpot 选择它：JIT 编译决策需要"确定性的热度证据" | 采样更适合诊断工具而非编译触发 |

## 3.2 两个计数器的构成

### 3.2.1 两类计数器的分工

| 计数器 | 统计什么 | 存放位置 | 超标后果 |
|--------|----------|----------|----------|
| 方法调用计数器 | 方法被**普通调用**的次数 | 方法关联的 `MethodCounters` | 提交普通 JIT 编译 |
| 回边计数器 | 方法中**backward 分支**（循环回跳）执行次数 | 各循环对应的 `MethodData` 条目 | 触发 **OSR 栈上替换编译**（见第 7 节） |

### 3.2.2 存放位置与信号流

```text
       ┌──────────── method ────────────┐
       │  调用计数器 invocation counter   │ ← "这个方法被叫了多少次"
       │  MDO(方法数据对象)               │
       │    ├─ bci 12: 回边计数 + 类型剖面 │ ← "这个循环体转了多少圈"
       │    └─ bci 20: 分支 taken/not     │
       └─────────────────────────────────┘
```

- **OSR 进入的方法调用不增加方法调用计数器**：栈上替换是"循环热"不是"方法热"的证据，两类信号分开累积。
- 两者可以**联合判定**：单次调用里循环转了 50 万次、方法本身只被调 3 次，回边计数器足以促成编译。

## 3.3 阈值与公式

### 3.3.1 经典数字（非分层编译）

```text
方法调用计数阈值 CompileThreshold：Client ≈ 1500，Server ≈ 10000
回边计数阈值   BackEdgeThreshold： 默认约 100000
权重修正      ClientModeEffect（默认 4）：参与两个计数器实际生效阈值的折算
```

### 3.3.2 分层编译下由 Tier3\*/Tier4\* 阈值接管

分层编译下（JDK 8+ 默认）经典数字不再直接生效，改由 `Tier3*` / `Tier4*` 两组阈值接管（下面为常见默认值，随版本有微调，可用 `-XX:+PrintFlagsFinal` 核对）：

| 参数 | 默认 | 含义 |
|------|------|------|
| `-XX:Tier3InvocationThreshold` | 200 | 升到 Level 3（C1 全量剖析）的调用数 |
| `-XX:Tier3CompileThreshold` | 2000 | 调用+回边联合计数超此值也升 L3 |
| `-XX:Tier3BackEdgeThreshold` | 60000 | L3 的 OSR 回边阈值 |
| `-XX:Tier4InvocationThreshold` | 5000 | 升到 Level 4（C2）的调用数 |
| `-XX:Tier4CompileThreshold` | 15000 | L4 联合计数阈值 |
| `-XX:Tier4BackEdgeThreshold` | 40000 | L4 的 OSR 回边阈值 |

```text
直觉理解：
  调用几百次 → 值得让 C1 带着完整剖析编译它（Level 3，产出数据）
  调用几千次 → 数据攒够了，值得 C2 花几十毫秒深度优化（Level 4）
  所以"方法调用约 1 万/回边约 10 万"的老数字在分层模式下并不直接生效。
```

```bash
-XX:CompileThreshold=10000    # 非分层 Server 的方法调用阈值（分层时被 Tier4* 取代）
-XX:BackEdgeThreshold=100000  # 非分层回边阈值
```

## 3.4 热度衰减（半衰期）

方法调用计数器统计的不是"绝对次数"而是**一段时间内的相对热度**：

- `-XX:+UseCounterDecay`（默认开启）：周期性给调用计数"打折"。
- `-XX:CounterHalfLifeTime=30`：半衰周期约 30 秒，超期未再热的方法计数衰减一半。
- 目的：防止启动期被批量调用、之后常年冷寂的方法（如初始化逻辑）被误判为热点浪费编译资源。

```text
计数
  │\
  │ \____          每过 30s 未达标 → 减半
  │      \____
  └──────────────▶ 时间
```

# 4. C1 与 C2 编译器

## 4.1 两个编译器的分工

| 编译器 | 全称 | 编译速度 | 优化强度 | 剖析数据使用 | 适用 |
|--------|------|----------|----------|--------------|------|
| C1 | Client Compiler | 快（µs~ms 级） | 简单 | 只记调用/回边计数等少量数据 | 启动阶段、低配机器 |
| C2 | Server Compiler | 慢（ms~百ms 级） | 激进 | 依赖**类型剖面、分支概率**做假设 | 长期运行的服务端 |

## 4.2 C1：用编译换解释时间

- 编译流水线：字节码 → HIR（高级 IR，SSA）→ LIR（低级 IR，贴近机器）→ 线性扫描寄存器分配 → 机器码。
- 典型优化：轻量内联、常量传播、冗余消除；**不做**逃逸分析等重型分析。
- 定位是"比解释器快、比 C2 出码快"的中间档。

```bash
-XX:TieredStopAtLevel=1    # 只用 C1（快速启动场景，如 CLI 工具、lambda 编译入口）
```

## 4.3 C2：用编译时间换峰值性能

- 以 **Sea-of-Nodes** 图 IR 做全局数据流/控制流分析。
- 独有能力：激进的类型推断内联、**逃逸分析 + 标量替换**、循环展开/判定外提、范围检查消除、锁消除/粗化——全部详见 [17_jit_optimization](17_jit_optimization.md)。
- 假设失败时通过 uncommon trap **去优化**回退（见 9.3），这是它敢做激进优化的底气。

## 4.4 编译线程模型（概念）

- C1/C2 各自运行在后台 **JavaCompilerThread** 上，与应用线程并行；编译产物写入 CodeCache。
- 应用线程只负责"发现热点 → 投递编译请求到队列"，不等待编译完成。
- 线程数量见第 6 节。

## 4.5 Graal：实验性第三方 JIT（已移除）

- **JDK 10 引入 JVMCI + Graal 实验性编译器（JEP 317）**：通过 `-XX:+UseGraalJIT -XX:+UnlockExperimentalVMOptions` 用 Java 写的 Graal 替代 C2，目的是让优化器可被 Java 开发者扩展。
- **JDK 21 移除 Graal 作为实验性 JIT（JEP 410）**：官方维护成本过高，能力迁移到 GraalVM 发行版继续发展；JVMCI 接口本身保留（仍支撑 GraalVM 与部分 AOT 路线）。
- 日常 HotSpot 调优语境下，**C1 + C2 分层编译仍是唯一默认答案**。

# 5. 分层编译（Tiered Compilation）

## 5.1 为什么需要分层

单一路线各有致命短板：

```text
只解释：     峰值性能差
只 C2：      启动慢（每方法几十 ms 编译）+ 前期没有剖析数据，只能保守编译
只 C1：      优化太浅
分层 = 让方法按热度逐档升级，启动性能与峰值性能兼得
```

- JDK 7 作为实验特性引入（`-XX:+TieredCompilation`），**JDK 8 起默认开启并成为唯一模式**。

## 5.2 分层结构与升级路径

### 5.2.1 五个编译层

| Level | 执行形态 | 剖析数据 | 出码质量 | 典型去向 |
|-------|----------|----------|----------|----------|
| 0 | 解释器执行 | 只统计调用/回边计数 | — | 冷方法永远停在 L0 |
| 1 | C1 编译 | **不收集**剖析 | 低 | 小而不热的方法终态 |
| 2 | C1 编译 | 简单剖析（调用次数、回边次数，内联受限） | 低 | OSR 过渡档 |
| 3 | C1 编译 | **完整剖析**（分支概率、类型 profile） | 中 | 攒数据喂给 C2 |
| 4 | C2 编译 | 消费 L3 攒好的剖析数据 | **最高** | 长期热点终态 |

### 5.2.2 升级路径

```text
方法被多次调用（普通路线）：
  Level 0 (解释) → Level 3 (C1 + 完整 profiling) → Level 4 (C2 深度优化)

不太热的小方法：
  Level 0 → Level 1 (C1 无剖析，够用就行)

循环热点（方法调用少但循环疯狂）：
  Level 0 → Level 2 (简单剖析) → Level 4 (OSR 编译)

降级：任何层假设失败 → 去优化 → 回到 Level 0 重新攒数据
```

> 分层下各档切换的具体阈值见 3.3 的 `Tier3* / Tier4*` 表。

```bash
-XX:+TieredCompilation         # JDK 8+ 默认开启（现为唯一编译模式）
-XX:TieredStopAtLevel=1        # 封顶 C1：牺牲峰值换启动（JRebel/开发模式常用）
-XX:TieredStopAtLevel=4        # 默认值：允许到 C2
```

### 5.2.3 TieredStopAtLevel 的工程场景

| 场景 | 常用配置 | 原因 |
|------|----------|------|
| IDE/热部署（JRebel 等） | `=1` | 类随时可能改，C2 编译成果会作废；C1 出码快损失小 |
| CLI / Serverless 短进程 | `=1` 或配合 CDS | 进程活不到 C2 回本，编译时间纯属浪费 |
| 长驻服务 | `=4`（默认） | 值得为峰值性能付预热成本 |

# 6. 编译线程

## 6.1 线程数量与默认值

- JIT 编译由后台线程**异步**完成，编译线程数受 `CICompilerCount` 控制，默认与 CPU 核数相关。

| 模式 | 默认编译线程 |
|------|--------------|
| 非分层 Client | 1 |
| 非分层 Server | 通常 4（核数不足时缩减） |
| 分层编译 | 总数按 C1 : C2 ≈ 2:1 划分（如 3 线程 = 2×C1 + 1×C2），随核数自适应 |

```bash
# 查看实际生效值
java -XX:+PrintFlagsFinal -version | grep CICompilerCount

# 手动设置（低配容器里可压缩编译线程给应用让 CPU）
-XX:CICompilerCount=2
```

## 6.2 排队、背压与编译风暴

- 编译线程是守护线程，任务多时进入 **编译队列**排队；队列积压时方法继续留在当前层执行（解释/C1 顶住），不会阻塞应用线程。
- 极端情况（微基准里疯狂生成匿名类）会出现"编译风暴"，CPU 被 JavaThread 抢走——`jstack` 看到 `C2 CompilerThread0` 高 CPU 即此景（排查见 [22_troubleshooting](22_troubleshooting.md)）。

# 7. OSR：栈上替换

## 7.1 动机："方法还没调用完，循环先热了"

普通 JIT 在**方法被调用时**才有机会切入编译代码；而一个只被调用一次、内部却有几亿次循环的方法（批处理、`main` 里的压测循环），等它返回再编译毫无意义。

```java
public void heavyLoop() {
    // 前几次解释执行
    for (int i = 0; i < 1_000_000_000L; i++) {
        // 回边计数器超标 → 触发 OSR 编译
        compute(i);
    }
    // OSR 只在本次调用期间生效
}
```

## 7.2 替换过程

```text
① 解释执行循环，回边计数器超标
      │
② 提交 OSR 编译请求（参数 = 方法 + 循环 bci）
      │   JIT 生成特殊入口的编译代码：
      │   普通编译入口从调用栈拿参数；OSR 入口从【当前解释帧】取局部变量
      │   → 编译产物签名不同（"on-stack 替换版"方法）
③ 编译完成，MethodData 标记 OSR 可用
      │
④ 解释器下一次执行到该循环头 → 跳转到机器码
      │   当前帧的局部变量/操作数栈按映射搬进新帧布局（"替换"发生在这条帧上）
      ▼
⑤ 循环体以机器码速度跑完，方法正常返回
```

## 7.3 OSR 代码的质量折扣

- OSR 编译**缺少方法入口信息**（参数从半路帧里捞），寄存器分配与 prologue 优化受限，质量通常低于普通 C2 编译。
- 若方法本身也达到调用阈值，后续会再来一次**普通 C2 编译**，下次调用直接用高质量版本。
- 观察：`PrintCompilation` 日志中带 `%` 标记的即 OSR 编译（见 8.1）。

# 8. 编译日志

## 8.1 PrintCompilation 逐列解读

```bash
-XX:+PrintCompilation
```

```text
时间戳  层  编译序号  标记      方法                        字节码大小
  1234   4    567    %n  3     java.lang.String::hashCode    (55 bytes)
   890   3    123     s  1     com.example.MyClass::put      (123 bytes)
```

| 列 | 含义 |
|----|------|
| 时间戳 | JVM 启动以来的毫秒数 |
| 层（tier） | 3 = C1 完整剖析出码，4 = C2；非分层不显示 |
| 编译序号 | 本次 JVM 内编译任务的递增 ID |
| `n` | native 方法包装器（为 JNI 调用生成 stub） |
| `s` | synchronized 方法 |
| `!` | 字节码中存在异常处理器（编译器需保守处理） |
| `%` | **OSR 编译**（回边触发） |
| 空白 | 普通编译 |
| `made not entrant` / `made zombie`（单独成行） | 旧代码作废：前者=新代码接管入口跳转；后者=无人再引用，可被清扫 |

```bash
# 更常见的简写形态（JDK 8）：
#  3   45       java.lang.String::hashCode (55 bytes)
#  4   46     % java.lang.Thread::run ...  ← 4 层 + OSR
```

## 8.2 详细编译日志（JDK 9+ 统一日志）

```bash
-Xlog:compilation*=info:file=compile.log

# 诊断级：完整剖析/内联决策的 XML 日志，供 JITWatch 分析（见 17 第 12 节）
-XX:+UnlockDiagnosticVMOptions -XX:+LogCompilation -XX:LogFile=hotspot.log
```

# 9. 编译策略与限制

## 9.1 CodeCache：编译产物的家

- JIT 代码放不进堆，驻留 **CodeCache**（属于方法区/运行时代码区范畴，见 [7_method_area](7_method_area.md)）。
- 默认预留 `-XX:ReservedCodeCacheSize=240m`（分层编译开启时）。
- JDK 9+ 默认**分段 CodeCache**：non-nmethods / profiled（L1-3） / non-profiled（L2-4）三段独立扩缩，防止解释器 stub 挤占 C2 空间。

```text
Java HotSpot(TM) 64-Bit Server VM warning:
CodeCache is full. Compiler has been disabled.
```

**后果**：JIT 直接停摆，新热点只能解释执行，性能雪崩（老应用典型故障）。

**对策**：

```bash
-XX:ReservedCodeCacheSize=512m     # 增大（注意分段后单段大小按比例）
-XX:+UseCodeCacheFlushing          # 允许刷出冷编译代码腾位（默认开）
# 监控：jcmd <pid> VM.info / -Xlog:codecache，或 JFR 事件（见 22_troubleshooting.md）
```

## 9.2 哪些方法编译器会拒绝/放弃

| 情形 | 机制 |
|------|------|
| 超大方法（数千行编译单元） | C2 内部节点数超限会 **bailout 放弃编译**（与 `-XX:MaxNodeLimit`、`NodeLimitFudgeFactor` 联动），方法退回低层执行 |
| 明确不想编 | `-XX:CompileCommand=exclude,org.Foo::bar` 禁编；`compileonly` 白名单 |
| 太热但太大 | 字节码超 `FreqInlineSize` 只是不可被**内联**，自身仍可编译（见 17 第 1 节） |
| 只跑一次 | 达不到阈值，终身解释——这正是"别给一次性任务 -Xcomp"的理由 |

> 实践启示：**把巨型方法拆小**不仅是为了可读性，也是在给内联和 C2 优化创造资格。

## 9.3 去优化（Deoptimization）

### 9.3.1 假设、guard 与 uncommon trap 流程

JIT 的激进优化建立在**运行期假设**上，假设被打破即回退：

```text
假设示例：类型剖面显示某调用点 100% 是 String
   ↓ C2 直接内联 String 分支 + 插入 guard
突然传入 Integer
   ↓ guard 命中 → uncommon trap
丢弃当前帧的"物化"过程：
   编译帧 → 重建解释帧（虚拟帧）→ 回到 Level 0 从字节码继续
   ↓
MethodData 更新为多态 → 重新计数 → 下次编译采用新策略（内联缓存/放弃内联）
```

常见触发原因：

- 类型假设失败：单态 → 多态（新加载了实现类）。
- CHA 假设失败：以为无子类的方法出现新加载子类（见 17 第 11 节）。
- 首次执行的分支从未被剖析覆盖（uncommon trap 兜底路径，如隐式空检查失败）。

### 9.3.2 观察与去优化风暴排查

```bash
-XX:+UnlockDiagnosticVMOptions
-XX:+PrintDeoptimizationEvents     # JDK 9+：-Xlog+deopt，输出每次去优化的 reason
-XX:+PrintDeoptimizationDetails    # 更详细（旧版）
```

- 偶发去优化正常；**去优化风暴**（同一方法反复 deopt/recompile）会显著掉性能，排查流程见 [17_jit_optimization](17_jit_optimization.md) 第 11.2 节。

# 10. 实战观察

## 10.1 查看当前编译配置

```bash
java -XX:+PrintFlagsFinal -version | grep -Ei 'tiered|CICompilerCount|CodeCache|CompileThreshold'

# 关注输出：
#   TieredCompilation                  = true
#   ReservedCodeCacheSize              = 251658240 (240M)
#   CICompilerCount                    = 4（随核数变化）
#   Tier4InvocationThreshold           = 5000
```

## 10.2 启动期分层案例：亲眼看到 Level 0→3→4

```bash
java -XX:+TieredCompilation -XX:+PrintCompilation -XshowSettings:vm Demo 2>&1 | head -40
```

```text
典型输出时间线：
  120   1       java.util.HashMap::hash (20 bytes)        ← L1/L3 快速出码
  310   3  12   java.lang.String::hashCode (55 bytes)     ← C1 带剖析
 4200   4  87   java.lang.StringLatin1::hashCode (36 bytes) ← 攒够数据后 C2 接管
```

对照实验：

```bash
java -Xint  Demo   # 全程解释，启动最快、稳态最慢
java -Xcomp Demo   # 全程编译，启动期 CPU 尖刺、总时长常常更差
```

## 10.3 CDS / AppCDS：与 JIT 互补的启动优化

JIT 解决"跑得慢"，CDS 解决"起得慢"，两者常配套使用。

### 10.3.1 版本演进

| 版本 | 事件 |
|------|------|
| JDK 5 | 启动类路径归档 CDS |
| JDK 10 | 支持应用类 dump（JEP 310） |
| JDK 11 | AppCDS 转正（JEP 341） |
| JDK 13 | **CDS 默认开启**（JEP 350） |
| JDK 19 | 动态归档，退出时自动生成（JEP 476） |

### 10.3.2 生成方式与启动收益

```bash
# 手动生成/使用归档
java -Xshare:dump                              # 系统级 CDS 归档
java -XX:DumpLoadedClassList=classes.lst -jar app.jar
java -Xshare:dump -XX:SharedClassListFile=classes.lst -XX:SharedArchiveFile=app.jsa -jar app.jar
```

- 原理：类解析/字节码验证/常量池结果预存为 mmap 友好格式，启动时**跳过解析、减少页错误 IO**，类更早可执行 → 更早开始攒剖析，间接缩短 JIT 预热窗口。
- Spring Boot 3.x 的 `spring-aot` CDS 支持即基于 JEP 483 优先加载策略（JDK 24+），进一步压启动。

## 10.4 执行引擎参数速查

```bash
# 模式与分层
-XX:+TieredCompilation            # 分层编译（JDK 8+ 默认开，现为唯一模式）
-XX:TieredStopAtLevel=1           # 封顶 C1：快速启动
-XX:CICompilerCount=3             # 编译线程总数
-Xint / -Xcomp / -Xmixed          # 强制模式（仅实验用）

# 热点阈值（非分层语义为主，分层看 Tier3*/Tier4*）
-XX:CompileThreshold=10000        # 方法调用阈值
-XX:BackEdgeThreshold=100000      # 回边阈值（OSR）
-XX:+UseCounterDecay              # 热度衰减（默认开）
-XX:CounterHalfLifeTime=30        # 半衰秒数

# CodeCache
-XX:ReservedCodeCacheSize=256m
-XX:+UseCodeCacheFlushing

# C2 优化开关（默认全开，仅排障时动）
-XX:+DoEscapeAnalysis             # 逃逸分析
-XX:+EliminateAllocations         # 标量替换
-XX:+EliminateLocks               # 锁消除
# -XX:+AggressiveOpts             # 已废弃移除：曾用于试验性开关打包，勿再使用

# 观察
-XX:+PrintCompilation             # 编译日志
-XX:+PrintInlining                # 内联决策
-XX:+UnlockDiagnosticVMOptions -XX:+LogCompilation   # XML 编译日志
-XX:+PrintDeoptimizationEvents    # 去优化事件
```
