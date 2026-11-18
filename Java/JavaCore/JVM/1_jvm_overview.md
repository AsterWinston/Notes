
# 1. 什么是 JVM

## 1.1 JVM 的定位

- JVM（Java Virtual Machine）是一个**虚拟计算机**：自定义了指令集（字节码）、寄存器（操作数栈/局部变量表）、内存模型（运行时数据区）与执行流程。
- 它是"一次编译，到处运行"（Write Once, Run Anywhere）的基石——`.class` 文件运行在 JVM 之上，屏蔽了底层操作系统与 CPU 指令集差异。
- HotSpot VM 是 Oracle JDK / OpenJDK 的默认 JVM 实现，也是最常用的；细节见后续各章。

| 属性 | 说明 |
|------|------|
| 本质 | 运行在真实 OS 之上的"软件实现的计算机" |
| 输入 | 符合 JVMS 规范的字节码（Class 文件） |
| 输出 | 依赖平台的机器码（解释执行 + JIT 编译） |
| 职责 | 类加载、内存分配与回收、线程调度桥接、安全管理 |

## 1.2 "一次编译，到处运行"的原理

### 1.2.1 字节码：平台无关的中间层

编译被拆成两段，JVM 是中间那层"翻译契约"：

```text
                Java 源码 (Hello.java)
                        │  javac（一次编译，产物统一）
                        ▼
                字节码 (Hello.class)      ←— 平台无关，格式由 JVMS 规定
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
   Windows JVM      Linux JVM       macOS JVM     ←— "到处运行"靠各家实现
   (x64 机器码)     (x64/arm64)     (arm64 机器码)
```

- javac 的产物（字节码）在所有平台上**完全相同**；差异全部下沉到 JVM 实现层。
- 真正"到处运行"的是字节码，不是 `.java` 源码，也不是最终机器码。

### 1.2.2 JVM 替你屏蔽了什么

| 差异维度 | 源码层面看到的 | JVM 层面实际发生的 |
|----------|----------------|--------------------|
| 指令集 | 同一份 `iadd` 字节码 | 解释器翻译 / JIT 生成 x86 `add` 或 arm64 `addw` |
| 内存管理 | `new` 之后不用管 | GC 自动回收（见 [10_gc_basics](10_gc_basics.md)） |
| 线程 | `java.lang.Thread` | 1:1 映射到 OS 线程；JDK 21 起还有虚拟线程（JEP 444） |
| 字节序/对齐 | 无感 | 类文件固定大端，堆布局由实现决定（见 [9_object_layout](9_object_layout.md)） |

# 2. JDK / JRE / JVM 的关系

## 2.1 包含关系

```text
┌─────────────────────────────────┐
│              JDK                │
│  ┌───────────────────────────┐  │
│  │           JRE             │  │
│  │  ┌─────────────────────┐  │  │
│  │  │        JVM          │  │  │
│  │  ├─────────────────────┤  │  │
│  │  │    核心类库          │  │  │
│  │  └─────────────────────┘  │  │
│  ├───────────────────────────┤  │
│  │  开发工具（javac, jar,   │  │
│  │  jshell, javadoc...）     │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

| 概念 | 说明 |
|------|------|
| JDK | Java Development Kit，包含 JRE + 开发工具 |
| JRE | Java Runtime Environment，包含 JVM + 核心类库 |
| JVM | Java Virtual Machine，负责加载并执行字节码 |

## 2.2 JDK 9 模块化后 JRE 的形态变化

JEP 261（模块化系统 JPMS）之后，"JRE 目录"这个概念被彻底改写：

| 时期 | 运行时形态 | 核心库载体 |
|------|-----------|-----------|
| JDK 8 及以前 | 固定一份完整 JRE（`jre/lib/rt.jar` 约 60+MB） | rt.jar / tools.jar 等大 jar |
| JDK 9+ | 类库存放于 `jmods/*.jmod`，镜像可裁剪 | 模块（java.base、java.sql…） |
| JDK 11+ | **Oracle 不再发布独立 JRE**，只有 JDK；运行时镜像用 jlink 自造 | jmod + 自定义镜像 |

## 2.3 jlink：自定义运行时镜像

```bash
# 1. 先算出应用真正依赖哪些模块
jdeps --multi-release 21 --print-module-deps app.jar
# 输出: java.base,java.logging,java.sql

# 2. 裁剪出最小运行时（几十 MB → 常见 30~40MB）
jlink --add-modules java.base,java.logging,java.sql \
      --output my-runtime \
      --strip-debug --no-man-pages --no-header-files \
      --compress=zip-6          # JDK 21 起用 zip-N 写法，旧版为 --compress=2

# 3. 直接跑，目标机可完全不装 Java
./my-runtime/bin/java -m app/com.example.Main
./my-runtime/bin/java --list-modules   # 验证镜像内容
```

- 价值：容器镜像瘦身（配合见 [23_container_jvm](23_container_jvm.md)）、启动扫描的模块变少。

## 2.4 jdeps：依赖分析

```bash
jdeps -verbose:class Hello.class        # 逐类依赖
jdeps -R --print-module-deps app.jar    # 递归扫描，输出模块清单
jdeps --jdk-internals app.jar           # 揪出对 JDK 内部 API（sun.*）的引用，迁移排雷
```

# 3. JVM 的架构

## 3.1 总体架构图

```text
源文件(.java) → javac → 字节码(.class) → JVM
                                              │
                        ┌─────────────────────┤
                        │                     │
                   类加载子系统            运行时数据区
                        │                     │
                        ▼                     ▼
                  ┌───────────┐       ┌───────────────┐
                  │ 加载/连接 │       │ 方法区｜堆    │
                  │ /初始化   │       │ 虚拟机栈｜PC  │
                  └───────────┘       │ 本地方法栈    │
                                      └───────┬───────┘
                                              │
                                        执行引擎
                                        ▼
                                  ┌─────────────┐
                                  │ 解释器｜JIT  │
                                  │ GC｜本地接口 │
                                  └─────────────┘
```

核心三大组件：**类加载子系统**（入口）、**运行时数据区**（内存 + GC）、**执行引擎**（执行 + 本地调用）。

## 3.2 类加载子系统

- 职责：根据**全限定类名**找到字节流，完成加载 → 验证 → 准备 → 解析 → 初始化（7 阶段见 [3_class_lifecycle](3_class_lifecycle.md)）。
- 实现：双亲委派与类加载器分层见 [4_classloader](4_classloader.md)。
- 输入侧的格式契约：Class 文件结构见 [2_class_file_structure](2_class_file_structure.md)。

## 3.3 运行时数据区（管理侧）

### 3.3.1 五大区域速览

| 区域 | 线程共享 | 存什么 | 溢出异常 |
|------|---------|--------|----------|
| 堆 | 共享 | 实例对象、数组 | `OutOfMemoryError: Java heap space` |
| 方法区（JDK 8+ 元空间） | 共享 | 类元数据、运行时常量池 | `Metaspace` OOM |
| 虚拟机栈 | 私有 | 栈帧（局部变量/操作数栈） | `StackOverflowError` / OOM |
| 程序计数器 | 私有 | 当前字节码行号 | 唯一不 OOM 的区域 |
| 本地方法栈 | 私有 | native 方法调用 | 同虚拟机栈 |

### 3.3.2 GC 归入运行时数据区的"回收侧"

- 图中常把 GC 画在执行引擎里，但 GC 操作的对象是**堆与方法区**，逻辑上应归入运行时数据区的生命周期管理。
- 分配（TLAB/指针碰撞）与回收（标记-整理/复制）是一体两面：详见 [6_heap](6_heap.md)、[11_gc_algorithm](11_gc_algorithm.md)。

## 3.4 执行引擎（执行侧）

### 3.4.1 解释器 + JIT

字节码没有对应真实指令，必须"翻译"：解释器逐条翻译求快启动，JIT 把热点编译成机器码求快执行（第 6 节详述，深入见 [16_execution_engine](16_execution_engine.md)）。

### 3.4.2 本地方法接口 JNI

被调方法若标记 `ACC_NATIVE`，执行引擎不执行字节码，而是经 JNI 直接调用 C/C++ 函数：

```text
Java 栈帧                     │                Native 栈帧
  native double sin(double)   │   libjava.so / 用户 .so
        │ 调用                │
        ▼                     ▼
  执行引擎 ── 发现无 Code 属性，只有 Native 属性
        └────▶ 通过 JNI 函数表(env) 调用本地实现 ────▶ OS 符号
```

- 典型入口：`System.nanoTime()`、`Thread.start0()`、Socket 读写。
- 边界开销：Java→native 帧切换无法被 JIT 内联，是热点方法性能杀手之一。

# 4. 主流 JVM 实现

## 4.1 总览

| JVM | 特点 | 使用场景 |
|-----|------|----------|
| HotSpot VM | Sun/Oracle 官方，JDK 默认 | 企业开发主力 |
| OpenJ9 | IBM 开源，启动快、内存占用低 | 云原生、微服务 |
| GraalVM | Oracle 开源，多语言运行时、AOT 编译 | 高性能、多语言互操作 |
| Zing VM | Azul 商业版，C4 算法实现亚毫秒 GC | 金融交易、低延迟系统 |
| Dalvik/ART | Google Android 虚拟机 | Android 平台 |

## 4.2 HotSpot VM

- 名字来自"热点编译"（第 5 节）。GC 框架与编译器高度可插拔（G1/ZGC/Shenandoah，见 [12_gc_collectors](12_gc_collectors.md)）。
- 源码分层（JDK 9+ 并入 openjdk 主仓库 `src/hotspot/`）：

```text
src/hotspot/
├── share/        平台无关核心：classfile/(类加载·常量池) memory/(堆) gc/
│                 interpreter/(模板解释器) compiler/c1,compiler/c2 oops/(对象模型) runtime/
├── os/           OS 相关：linux/ windows/ macos/ pthread 封装、mmap 适配
├── cpu/          架构相关：x86/ aarch64/ ... 机器码生成、寄存器定义
└── os_cpu/       OS×CPU 组合的粘合层：stubRoutines、native 方法转发
```

- 一个字节码的执行路径：`share/bytecodes → share/interpreter(模板) → share/compiler(C2 生成 IR) → cpu/x86 发射机器码`。

## 4.3 OpenJ9

- 源自 IBM J9，Eclipse 基金会开源，与 Eclipse Open JDK 发行版搭配。
- **共享类缓存**（Shared Classes）是其省内存的关键：多个 JVM 进程通过 mmap 共享同一份缓存中的类元数据、AOT 编译代码与不可变堆对象。

```bash
# 第一个进程建立缓存，后续进程直接映射复用
java -Xshareclasses:name=appcache -Xscmx256m -jar app.jar
# 效果：类元数据不重复 → 容器多副本场景常驻内存可降 30%~60%
```

- 启动快（默认 AOT 缓存编译好的框架代码）+ 内存小 → 云原生常见选项。

## 4.4 GraalVM

- 两条路线：**JIT 编译器 Graal**（曾是 JDK 内实验产物，JEP 410 于 JDK 21 从 OpenJDK 移除 `jaotc`/Graal JIT，改由独立 GraalVM 发行版维护）；**native-image 的 AOT**。
- native-image 在构建期做**封闭世界假设**（closed-world analysis）：可达对象图必须静态可确定，直接链出机器码可执行文件。

| 维度 | 传统 JVM（JIT） | native-image（AOT） |
|------|-----------------|---------------------|
| 启动时间 | 百 ms 级 | ms 级 |
| 内存占用 | JVM + profile 开销 | 通常低数倍 |
| 峰值吞吐 | 高（运行期 profile 驱动优化） | 较低（无运行期再优化） |
| 反射/代理 | 自由 | 需写可达性元数据配置（reflect-config 等） |
| 构建 | 秒级 | 分钟级、吃内存 |

- 生态适配：Spring Boot 3 AOT / Quarkus 都是围绕封闭世界假设做提前"去反射化"。

## 4.5 Azul Zing

- 商业 JVM，招牌是 **C4**（Continuously Concurrent Compacting Collector）：并发整理 + 读屏障，停顿与堆大小基本解耦，支撑 TB 级堆。
- 开源界的同方向后继：Shenandoah、ZGC（见 [13_low_latency_gc](13_low_latency_gc.md)）。

## 4.6 Android：Dalvik / ART

| 维度 | Dalvik（≤4.4） | ART（≥5.0） |
|------|---------------|-------------|
| 输入 | dex（多类合一，非 Class 文件） | dex |
| 架构 | 寄存器虚拟机 | 寄存器虚拟机 |
| 编译 | 运行期 JIT | **安装期 AOT + 运行期 JIT 分级编译**（热点方法再 JIT 优化，配合 profile 收集） |
- ART 的分级思路与 HotSpot 混合模式殊途同归：启动与峰值性能折中。

# 5. 热点探测与 HotSpot 名称由来

## 5.1 "Hot Spot" 即"热点"

- JVM 并不对所有代码做即时编译，而是先解释执行，然后探测出"热点代码"（频繁执行的方法或循环），交给 JIT 编译器优化编译为本地机器码。
- 编译产物缓存于 **Code Cache**（`-XX:ReservedCodeCacheSize`，默认 240MB），满了会触发清扫（NMethod 逐出）。

## 5.2 两种热点探测策略

- **基于采样**（Sample Based）：定期检查各线程栈顶，统计出现频率最高的方法。
- **基于计数器**（Counter Based）：为每个方法/循环维护计数器，超阈值触发 JIT。
- HotSpot 使用**基于计数器**策略（名字里的 HotSpot 就来自这里）；OpenJ9 则用采样。

### 5.2.1 为什么 HotSpot 不用采样法

| 缺陷 | 说明 |
|------|------|
| 精度粗 | 只能到"方法级"，无法精确到"哪条循环回边热" |
| 短方法被低估 | 调用极频繁的小方法（getter/工具方法）在采样窗口里常常抓不到栈顶 |
| 窗口盲区 | 应用刚启动时还没积累样本，错过首批热点 |
- 代价：计数器要占内存、每次调用/跳转都有计数指令开销——JVM 认为这笔交换值得。

### 5.2.2 计数器法：方法调用计数器 + 回边计数器

```text
方法调用计数器                     回边计数器
  每次 invoke 类指令 +1              字节码里遇到"向后跳转"(goto/if 回到更早 pc) +1
  衡量方法被调用的次数               衡量单个循环体的迭代次数
        │                                  │
        └── 超阈值 → 整个方法 JIT 编译       └── 超阈值 → OSR 编译（栈上替换：
                                              不等方法返回，循环中途换机器码）
```

- OSR 细节（为什么循环也要单独编译、OSR 代码为何略慢）见 [16_execution_engine](16_execution_engine.md)。
- 调用计数器超过阈值但编译队列太挤时会**衰减**（计数减半清零重来），多次衰减仍排不上号的方法会被放弃编译。

## 5.3 阈值与分层编译系数

JDK 8+ 默认开启分层编译（Tiered Compilation），"触发线"不止一条，且各层生效系数不同：

```text
执行档位:  Level0 解释 → Level1/2/3 C1(带 profiling) → Level4 C2(全优化)
```

| 参数（JDK 21 默认，随版本略有浮动） | 典型值 | 含义 |
|------|--------|------|
| `-XX:Tier3InvocationThreshold` | 200 | 进入 C1+完整 profiling 的调用数 |
| `-XX:Tier3CompileThreshold` | 2000 | 进入 C1 编译的调用数 |
| `-XX:Tier3BackEdgeThreshold` | 6000 | 进入 C1 的回边数 |
| `-XX:Tier4InvocationThreshold` | 5000 | 升级到 C2 的调用数 |
| `-XX:Tier4CompileThreshold` | 15000 | 升级到 C2 的编译计数 |
| `-XX:Tier4BackEdgeThreshold` | 40000 | 升级到 C2 的回边数 |

- **生效系数**：基础阈值会乘一个随系统负载（`LoadAverage`）联动的系数——负载越高，有效阈值越低，让热点尽早进编译队列、缩短解释执行时间；同时编译线程数也会受限，二者折中。
- 统一缩放：`-XX:CompileThresholdScaling=2.0` 让所有触发线整体×2（JDK 9+，客户端偏吞吐/省内存时常用）。
- 观测：`-XX:+UnlockDiagnosticVMOptions -XX:+PrintCompilation`；JDK 11+ 还有 `-XX:+PrintTieredEvents`。

# 6. 编译与解释并存

## 6.1 工作模式图

```text
.java  →  javac  →  .class  →  JVM
                                ├── 解释器（逐行解释字节码，快速启动）
                                └── JIT编译器（热点代码编译为机器码，高效运行）
```

## 6.2 解释器的作用

- **快速启动**：编译是秒级后台活，解释器让 `main` 第一行就能跑。
- **省内存**：不是每段"只执行几次"的代码都值得占用 Code Cache。
- **反优化（deoptimization）的落点**：C2 基于投机假设（类唯一子类、分支恒定）优化，假设失效时丢弃机器码、**退回解释器**继续正确执行——解释器是正确性兜底（细节见 [17_jit_optimization](17_jit_optimization.md)）。

## 6.3 JIT 编译器的作用

- 把热点编译为**本机指令 + 深度优化**：方法内联、逃逸分析、锁消除、循环展开……这些解释器永远做不到（见 [15_memory_allocation](15_memory_allocation.md)、[17_jit_optimization](17_jit_optimization.md)）。
- 后台编译线程异步进行，不阻塞应用线程。

## 6.4 混合模式工作流时序

```text
t0 ──────────────────────────────────────────────────────▶ t
main 开始:  [解释执行，收集计数器/类型profile]
                 │ 达 Tier3 线
                 ▼
            [C1 编译 + 记录 profile]（后台线程生成，期间仍解释执行）
                 │ 数据够热够全
                 ▼
            [C2 全优化编译] ──▶ 直接执行机器码
                 │
                 └─ 假设失效(加载了新子类等) → 反优化 → 回到解释执行重新积累
```

- 所以同一段代码在生命周期里**身份会变**：解释 → C1 → C2 → （可能打回）解释。

## 6.5 执行模式查看与强制

```bash
java -version
# 输出示例：
# Java HotSpot(TM) 64-Bit Server VM (build 25.xxx, mixed mode)   ← 默认混合模式
```

```bash
java -Xint MyClass        # 纯解释：启动即稳定、无编译抖动，性能极差（基准测试/调试用）
java -Xcomp MyClass       # 纯编译：方法首次调用前必须编译完；启动极慢，且部分代码
                          #（如编译队列满、OSR 场景）实际仍走解释，名不副实
java -Xmixed MyClass      # 混合模式（默认）
```

```bash
# 查看关键开关的终值（JDK 21）
java -XX:+PrintFlagsFinal -version | findstr /i "TieredCompilation TieredStopAtLevel"
#   TieredCompilation = true                 ← 分层编译默认开启
#   TieredStopAtLevel = 4                    ← 3 表示只用 C1（启动快，如 GraalVM 工具链）
```

# 7. JVM 规范与实现分离

## 7.1 规范 vs 实现

- **JVM 规范**（Specification）：定义 JVM 必须遵守的行为标准，但不规定具体实现方式。
- **JVM 实现**（Implementation）：如 HotSpot、OpenJ9，各自采用不同方式实现规范。
- 这也是任何遵循 JVM 规范的语言（Kotlin、Scala、Groovy、Clojure）都能跑在 JVM 上的原因。

| 规范规定 | 实现自由发挥 |
|----------|-------------|
| 类文件格式 | GC 算法 |
| 字节码指令集 | JIT 编译器策略 |
| 运行时数据区结构 | 对象内存布局细节 |
| 类加载/链接/初始化流程 | 指令优化方式 |
| 虚拟机启动/退出行为 | 线程调度方式 |

## 7.2 规范文档索引（JSR / JVMS）

| 主题 | 出处 | 备注 |
|------|------|------|
| 类文件格式 | JVMS（The Java Virtual Machine Specification）第 4 章 | 对应本仓库 [2_class_file_structure](2_class_file_structure.md) |
| 字节码指令集 | JVMS 第 6 章（逐指令定义操作数栈效果） | javap 读不懂时回查这里 |
| 类加载与链接 | JVMS 第 5 章 | 对应 [3_class_lifecycle](3_class_lifecycle.md) |
| Java 内存模型与并发 | JLS 第 17 章；其演进源自 **JSR-133**（修复旧 memory model 缺陷，volatile/final 语义重定义） | 对应 [18_jmm](18_jmm.md) |
| 规范版本 | 跟随 Java SE 版本发布，读 SE 21 版即可 | 网上旧版教程常停留在 SE 7/8 表述 |

## 7.3 多语言生态：字节码是唯一硬契约

任何能产出合规 Class 文件的语言都是"JVM 语言"：

```kotlin
// Main.kt
fun main() { println("hi") }
```

```text
javap -c MainKt.class（节选）：
public final class MainKt {
  public static void main(java.lang.String[]);   ← Kotlin 顶层函数被编译成静态方法
     0: getstatic  java/lang/System.out
     3: ldc        "hi"
     5: invokevirtual println
     8: return
}
```

- Kotlin/Scala/Groovy/Clojure 的"语言特性"（协程、模式匹配）要么编译成普通字节码组合，要么靠 `invokedynamic` 挂运行时库（见 [8_vm_stack](8_vm_stack.md) 第 9 节）。
- 推论：**性能问题是字节码级的问题**——换语言不自动解决 GC/JIT 行为，排障思路通用。

# 8. JVM 启动与执行全过程串讲

## 8.1 从 `java Hello` 到 main 执行：一张图

```text
$ java -Xmx512m Hello
      │
      ▼ java 启动器（OS 原生可执行程序, libjli）
JLI_Launch()
      │ 1. 解析命令行/JDK_JAVA_OPTIONS/java.cfg，拼出 JVM 参数
      │ 2. 按顺序找 libjvm.so / jvm.dll，动态加载进启动器进程
      ▼
JNI_CreateJavaVM()                       ← C++ 侧，"虚拟计算机"上电
      │ 3. 初始化运行时数据区（堆/方法区/线程）、选定 GC、解释器就绪
      │ 4. 创建 Java 主线程（main 就跑在这个线程上）
      ▼
启动类加载器体系就绪 → 加载 Hello（Bootstrap→Platform→App 双亲委派）
      │ 5. 类加载 7 阶段：加载→验证→准备→解析→初始化（<clinit>）
      ▼
反射定位并调用 public static void main(String[])
      │ 6. 压栈帧 → 模板解释器逐条执行字节码
      ▼
计数器达标 → 提交后台编译队列 → C1/C2 生成机器码 → 切换到机器码执行
      │ 7. 运行期持续：GC 回收、JIT 优化/反优化、类加载继续发生
      ▼
main 返回 → 非守护线程清空 → 执行 shutdown hook → VM 退出
```

## 8.2 各阶段要点速查

| 阶段 | 关键事实 | 常见误区 |
|------|----------|----------|
| 启动器解析 | 先应用环境变量再应用命令行参数 | 以为 `-Xmx` 写错位置也能生效 |
| CreateJavaVM | GC/编译器参数在此刻定型，**启动后不可换收集器** | 运行中 `jcmd` 改 GC 参数幻想 |
| 主类加载 | `ClassNotFoundException` 在这一步抛出 | 误报成启动脚本问题 |
| main 执行 | 前几百 ms 是解释执行，**JIT 预热（warm-up）导致压测前期数据失真** | 压测不设预热直接取均值 |
| 退出 | `System.exit` / 非守护线程耗尽 / kill -9 之外都会走 shutdown hook | 容器 SIGKILL 丢钩子（见 [23_container_jvm](23_container_jvm.md)） |

## 8.3 本章 → 后续章节导航

```text
①字节码怎么写 ──▶ [2_class_file_structure]  ③类生命周期 ──▶ [3_class_lifecycle]
②类谁来加载   ──▶ [4_classloader]           ④内存怎么分 ──▶ [5_runtime_data_area]
                                              ├ [6_heap] [7_method_area] [8_vm_stack] [9_object_layout]
⑤字节码怎么执行 ▶ [16_execution_engine] ──优化──▶ [17_jit_optimization]
⑥对象怎么回收 ──▶ [10~14 gc 系列]           ⑦并发语义 ──▶ [18_jmm] [19_lock_optimization]
⑧出了问题怎么办 ▶ [20_jvm_tools] [21_visual_tools] [22_troubleshooting]
```
