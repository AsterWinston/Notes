# 1. 方法区概述与演进

## 1.1 规范中的逻辑区域 vs HotSpot 实现

| 视角 | 说法 |
|------|------|
| JVM 规范 | "方法区"是一个**所有线程共享的逻辑区域**，存类元数据、常量池等；是否 GC、放哪里，规范不管 |
| HotSpot 实现 | JDK 7 及以前：用堆内一块固定区域实现（**永久代 PermGen**）；JDK 8+：用本地内存实现（**元空间 Metaspace**），JIT 代码再单独放 **Code Cache** |

推论：讨论"方法区 OOM / GC"时，必须先声明讨论的是规范概念还是 HotSpot 某个版本的实现。

## 1.2 PermGen → Metaspace 时间线

### 1.2.1 版本时间线

```text
JDK 6 及之前   PermGen 全量模式：类元数据 + 常量池 + 静态变量 + 字符串常量池 都在永久代
JDK 7          大搬家（PermGen 本身还在）：
                 ├─ 字符串常量池、静态变量 → 堆
                 └─ 类元数据留在 PermGen（为 JDK 8 彻底移除铺路）
JDK 8          JEP 122 移除 PermGen → Metaspace（本地内存）
JDK 9          G1 成默认；类元数据布局随 CompactObjectHeaders 等后续继续瘦身
JDK 25         紧凑对象头转正（JEP 519），类元数据在头部只占 32bit 索引
```

### 1.2.2 内存结构对照图

```text
JDK 7                     JDK 8+
┌──────────────┐         ┌──────────────────────┐
│   JVM 堆     │         │      JVM 堆          │
│ ┌──────────┐ │         │ ┌──────────────┐     │
│ │ 永久代    │ │         │ │ 字符串常量池  │     │
│ │ 类元数据  │ │         │ │ 静态变量     │     │
│ │ 常量池    │ │         │ │ (在Class对象中)│    │
│ │ 静态变量  │ │         │ │ 对象实例     │     │
│ └──────────┘ │         │ └──────────────┘     │
│ ┌──────────┐ │         └──────────────────────┘
│ │ 对象实例  │ │
│ │ 字符串池  │ │         ┌──────────────────────┐
│ └──────────┘ │         │   本地内存 (Native)   │
└──────────────┘         │ ┌──────────────┐     │
                          │ │ 元空间        │     │
                          │ │ 类元数据     │     │
                          │ └──────────────┘     │
                          │ ┌──────────────┐     │
                          │ │ JIT编译缓存   │     │
                          │ │ (Code Cache) │     │
                          │ └──────────────┘     │
                          └──────────────────────┘
```

## 1.3 为什么移除永久代

| 原因 | 展开 |
|------|------|
| 大小固定难调 | `MaxPermSize` 与"应用类元数据总量"没有可推导的关系：调小了 PermGen OOM，调大了浪费堆地址空间 |
| 回收时机差 | 元数据基本只在 **Full GC** 时回收 → 类生成快一点就要付一次全堆停顿的代价 |
| 与并发 GC 冲突 | 永久代的类卸载逻辑和 CMS 等并发收集器耦合复杂，bug 多、实现难 |
| 合并路线图 | JRockit 从来没有永久代概念；HotSpot 与 JRockit 合并时统一改为本地内存实现 |
| 语言多样性 | lambda / 动态语言 / 字节码生成框架让"运行时类数量"从相对固定变成高度动态，本地内存按需扩展更合适 |

类元数据本身的结构（InstanceKlass）与加载流程详见 [3_class_lifecycle](3_class_lifecycle.md)。

# 2. 元空间内存管理

## 2.1 基于本地内存 mmap

- 元空间不占用 `-Xmx`，向 OS **预留（reserve）大片虚拟地址空间**，按实际使用 **提交（commit）** 物理内存——预留便宜、提交才花钱。
- 因此"元空间默认无上限"指的是提交量受物理内存/容器配额约束，而非真的有数值上限。

## 2.2 Chunk 分配机制：三层结构

### 2.2.1 VirtualSpaceSpace / Chunk / MetaBlock 三层

元空间不是"一整块连续内存"，而是三级切分：

```text
Metaspace
└── VirtualSpaceSpace          ← 大块虚拟地址空间（reserve，逻辑上"仓库"）
    └── Chunk                  ← commit 的固定大小块（几十~几百 KB 量级），元数据真正住在这里
        └── MetaBlock          ← Chunk 内部的分配单元（类元数据、方法数据各取所需）

MemPool（记账单位，与物理结构正交）
├── 每个 ClassLoader 一个私有 MemPool → 只用"非共享 Chunk"
└── 一个共享 MemPool（WKA，Well-Known Attributes）→ java.lang.Object 等
    多加载器重复加载的基础类元数据放这里，跨加载器共享、省内存
```

### 2.2.2 工程推论

关键推论：

| 特性 | 后果 |
|------|------|
| Chunk 不要求连续 | 元空间"碎片"不靠整理解决，靠 **ClassLoader 粒度归还** |
| 非共享 Chunk 与 ClassLoader 绑定 | ClassLoader 被回收 → 它私有的整块 Chunk 直接还给元空间，无需逐对象标记 |
| 类元数据按大小路由 | 小类走小块、大类走大块/多块，避免一个 InstanceKlass 跨块拼接失败 |

## 2.3 MetaspaceSize 是"首次 GC 高水位线"，不是上限！

### 2.3.1 参数语义

```bash
-XX:MetaspaceSize=128m        # ⚠ 语义=触发 GC 的使用量阈值，不是"初始大小"也不是上限
-XX:MaxMetaspaceSize=256m     # 真正的容量上限，默认无限制（=物理内存）
-XX:MinMetaspaceFreeRatio=40  # GC 后空闲比例低于 40% → 把高水位线抬高（扩容方向）
-XX:MaxMetaspaceFreeRatio=70  # GC 后空闲比例高于 70% → 把高水位线压低（缩容方向）
```

### 2.3.2 水位线移动机理

```text
使用量
 │          ▲触发GC        ▲GC后水位线按空闲比例上移
 │   ▂▄▆█   │  ▂▄▆        │  ▂▄▆▇   ← 每次 GC 后 MetaspaceSize(水位线) 自适应
 │   █      │  █          │  █
 ├─ ─ ─ ─ ─┼─ ─ ─ ─ ─ ─ ─┼─ ─ ─ ─ ─ → 水位线移动
 0     MetaspaceSize(初始阈值)
 └──────────────────────────────────────▶ 时间：越过 MaxMetaspaceSize 才 OOM
```

### 2.3.3 启动期陷阱与参数迁移注意

- 默认 `MetaspaceSize` 很小（几十 MB 以内），大应用启动期会**连环触发以元空间为借口的 GC**——这是"启动慢/启动期 Full GC 多"的经典原因（见 8.1）。
- 旧文常说的 `PermSize` 是"初始大小"，而 `MetaspaceSize` **没有**对应语义，别做参数名平移。

# 3. 元空间 GC

## 3.1 元空间 GC 的触发与类卸载

### 3.1.1 触发链条图

```text
元空间内分配失败
   │
   ├─ committed < MaxMetaspaceSize ──▶ 再 mmap 一个 VirtualSpaceSpace/Chunk，继续跑（成本最低）
   │
   └─ 使用量达到高水位线(MetaspaceSize)
          │
          ▼
       触发 GC（多数收集器表现为 Full GC；G1/ZGC 可并发承担）
          │
          ├─ GC 中执行类卸载（条件见 3.1.2）→ 水位线按 FreeRatio 调整 → 继续运行
          │
          └─ 卸载后仍无空间且触到 MaxMetaspaceSize
                 ▼
              OutOfMemoryError: Metaspace   （或 Compressed class space）
```

### 3.1.2 类卸载三条件（缺一不可）

1. 该类的所有**实例**已被回收；
2. 加载该类的 **ClassLoader** 已被回收；
3. 该类的 **Class 对象**没有任何强引用（没被反射持有等）。

三条件的判定发生在 GC 可达性分析中，详见 [3_class_lifecycle](3_class_lifecycle.md)。

### 3.1.3 各收集器对类卸载的支持

| 收集器 | 类卸载 | 说明 |
|--------|--------|------|
| CMS（JDK 9 废弃 / JDK 14 移除） | 默认**不**在并发周期卸载 | 需 `-XX:+CMSClassUnloadingEnabled`；否则类元数据几乎只能等 Full GC |
| Parallel / Serial | Full GC 时卸载 | 传统行为 |
| G1 | 默认支持，随 GC 周期进行 | 无需额外参数 |
| ZGC / Shenandoah | 支持，且可并发卸载 | 低延迟场景类元数据回收不再付全停代价 |

```bash
# JDK 8 + CMS 老应用最常被漏配的一条：
-XX:+CMSClassUnloadingEnabled
```

# 4. 运行时常量池

## 4.1 与 Class 文件常量池的关系

```text
class 文件                    类加载后
┌────────────────┐  加载     ┌────────────────────┐
│ constant_pool   │ ───────▶ │   运行时常量池       │
│ #1 = Class      │          │ (元空间，随类存在)    │
│ #2 = Utf8 "Foo" │          │ 结构不变，含义升级：   │
│ #5 = Methodref  │          │ 符号引用→直接引用     │
└────────────────┘           └────────────────────┘
```

| 对比 | Class 文件常量池 | 运行时常量池 |
|------|-----------------|-------------|
| 存储位置 | Class 文件中 | 方法区（元空间） |
| 存在形态 | 静态的二进制数据 | JVM 运行时动态使用 |
| 动态性 | 编译时确定 | 运行时可通过 intern() 等方法添加 |
| 符号引用 | 存储符号引用 | 可解析为直接引用 |

Class 文件侧的结构细节见 [2_class_file_structure](2_class_file_structure.md)。

## 4.2 符号引用 → 直接引用的时机

- **静态解析（链接阶段）**：invokestatic / invokespecial 目标、private/final 方法、字面量常量——在类加载解析期就能绑定（前提目标类已可确定）。
- **动态解析（首次使用）**：invokevirtual / invokeinterface 指向的符号引用，要到第一次执行该指令时按实际类型解析——这是多态的入口。
- 两条路径的完整时机表与栈帧视角详见 [8_vm_stack](8_vm_stack.md) 第 5 节。

## 4.3 动态性演示

```java
// ① intern：运行期把字符串加进 StringTable（堆）
String s = new StringBuilder("hel").append("lo").toString().intern();

// ② 反射：方法名可以来自配置/字符串拼接，编译期常量池里根本没有
Method m = obj.getClass().getMethod(System.getProperty("entry"));

// ③ JDK 9+ 字符串拼接（JEP 280）编译成 invokedynamic，
//    拼接策略运行期由 StringConcatFactory 决定，常量池只留 BootstrapMethod 引用
String x = a + b;
```

## 4.4 常量池存放内容清单

| 内容 | 举例 | 解析后变成 |
|------|------|-----------|
| 字面量 | `int 42`、`"abc"`、`1.0f` | 不变（或直接内联） |
| 类/接口符号引用 | `CONSTANT_Class → com.Foo` | 指向 InstanceKlass 的直接引用 |
| 字段符号引用 | `CONSTANT_Fieldref Foo.bar:I` | 字段表偏移/地址 |
| 方法符号引用 | `CONSTANT_Methodref Foo.add:(II)I` | 方法入口地址 / vtable 槽 |
| 方法句柄/动态调用点 | `CONSTANT_InvokeDynamic`（lambda） | CallSite 目标（见 [8_vm_stack](8_vm_stack.md)） |
| 方法签名/名字 | Utf8 条目 | 供反射、异常栈、链接匹配 |

# 5. 编译器生成代码：Code Cache

## 5.1 定位

JIT（C1/C2）编译出的**机器码**存放在本地内存的 Code Cache——规范上可归入"方法区的一部分"，实现上与 Metaspace **各自独立的 mmap 区域**，互不挤占。执行引擎细节见 [16_execution_engine](16_execution_engine.md)。

```bash
-XX:InitialCodeCacheSize=...        # 初始提交量（默认很小，按需扩）
-XX:ReservedCodeCacheSize=240m      # 预留上限，分层编译开启时默认 240M
-XX:+UseCodeCacheFlushing           # 满了先"刷走"冷方法（默认开启）
```

## 5.2 JDK 9+ 三段式布局

### 5.2.1 三段布局图

```text
CodeCache (Reserved, 默认 240M)
┌───────────────┬──────────────────────────┬─────────────────────────┐
│ non-nmethods  │ profiled_nmethods        │ non-profiled_nmethods   │
│ intrinsic/    │ C1 带 Profiling 代码      │ C2 完全优化代码          │
│ adapter 桩     │ (tier1/2/3) 默认 122M    │ (c2 compiled) 默认 111M  │
│ 默认 5M        │                          │                         │
└───────────────┴──────────────────────────┴─────────────────────────┘
```

### 5.2.2 分段参数与独立水位

- 三段各自独立水位：任何一段写满都会把**对应层级的编译**卡住（如 profiled 段满 → 新代码只能停在解释执行，无法去 C1）。
- 参数：`-XX:NonNMethodCodeHeapSize` / `-XX:ProfiledCodeHeapSize` / `-XX:NonProfiledCodeHeapSize` 可微调分段。

## 5.3 写满的后果与自救

### 5.3.1 告警信号

```text
Java HotSpot(TM) 64-Bit Server VM warning:
CodeCache is full. Compiler has been disabled.
Try increasing the -XX:ReservedCodeCacheSize to allow more code to be compiled.
```

### 5.3.2 后果链与自救机制

| 后果链 | 说明 |
|--------|------|
| JIT 停摆 | 新热点代码不再编译，退化解释执行 → CPU 使用率、RT 显著上升（**不报 OOM**，容易漏监控） |
| UseCodeCacheFlushing | 默认开启：GC 时识别"很久没执行"的编译代码，刷掉腾空间；刷完热点再回来 → 性能震荡但保命 |
| 排查入口 | 日志中该 warning；`jcmd <pid> Compiler.codelist` 看已编译方法量；NMT `Code` 类别看真实占用（见 [20_jvm_tools](20_jvm_tools.md)） |

> 经典场景：大量反射/动态代理生成小方法把 C2 段撑满。治本=减少生成量或调大 ReservedCodeCacheSize；不要关 flushing。

# 6. 静态变量存储位置演变

```java
public class Example {
    public static int COUNT = 10;
    public static String NAME = "hello";
}
```

| 版本 | COUNT / NAME 本尊在哪 | 指向的对象在哪 |
|------|----------------------|----------------|
| JDK 6 | 永久代（Class 元数据旁边） | 也在永久代 |
| JDK 7+ | **堆**：作为 `Class` 对象（`java.lang.Class` 实例）内部的镜像字段 | NAME → "hello"（字面量进字符串常量池，也在堆） |

## 6.1 与 Class 对象生命周期绑定意味着什么

```text
静态变量 ── 属于 ──▶ Class 对象(堆) ── 属于 ──▶ 它的 ClassLoader
   │                                            │
   └─ 只要类活着，静态变量引用的对象永远可达       └─ ClassLoader 回收 ⇐ 它加载的所有类都不被引用
```

由此推出经典的 **ClassLoader 泄漏 → 静态变量泄漏** 链条：

```text
热部署(webapp reload) 每次新建 ClassLoader
   │
   ├─ 旧 ClassLoader 想死，但它加载的类还有实例活着
   │     （最常见：单例/ThreadLocal/线程持有旧类实例，或某新类直接引用旧静态字段）
   ├─ 实例活着 → Class 活着 → ClassLoader 活着
   └─ 结果：堆里堆满旧版本实例，元空间里 Chunk 无法归还
         最终 → java.lang.OutOfMemoryError: Metaspace（报错在元空间，根因在引用链）
```

工具上先用 `jstat -class` 确认 Loaded 只增不减，再用 MAT 追 ClassLoader 引用链（见 [21_visual_tools](21_visual_tools.md)）。

# 7. 实战：元空间溢出

## 7.1 保留演示：CGLIB 无限生成类

```java
// -XX:MaxMetaspaceSize=10m -XX:MetaspaceSize=10m
public class MetaspaceOOM {
    public static void main(String[] args) {
        for (int i = 0; ; i++) {
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(OOMObject.class);
            enhancer.setUseCache(false);           // 关键：禁用类缓存
            enhancer.setCallback((MethodInterceptor) (obj, method, args1, proxy) ->
                    proxy.invokeSuper(obj, args1));
            enhancer.create();
            System.out.println("Created: " + i);
        }
    }
    static class OOMObject {}
}
// java.lang.OutOfMemoryError: Metaspace
```

## 7.2 真实场景来源表

| 场景 | 生成机制 | 泄漏特征 | 修法 |
|------|----------|----------|------|
| CGLIB/Spring AOP 滥用 | 每次增强生成新子类 | Loaded 类数随调用量线性涨 | 复用代理缓存（别 setUseCache(false)） |
| JDK 动态代理 × new ClassLoader | Proxy 类按 (loader, 接口组) 缓存 | 每次换 loader 就绕过缓存 | 固定 loader / 缓存代理对象 |
| Groovy 脚本 | 每次 evaluate **编译一个新类**，脚本含时间戳/UUID 等动态内容时缓存全失效 | 类数随脚本执行数涨 | 保证脚本文本稳定；定期 `groovyClassLoader.clearCache()` |
| fastjson ASM | 反序列化器用 ASM 生成类；每个新 `ParserConfig`/Feature 组合一套 | 频繁 new ParserConfig 的服务类数异常 | 全局复用 ParserConfig；`ParserConfig.setAsmEnable(false)` 可退化为反射实现（有性能代价） |
| 热部署/OSGi | 每次 reload 一个 ClassLoader | 见 6.1 链条 | 修静态持有者；reload 后确认旧 loader 可达性 |

## 7.3 观测手段（按可靠性排序）

### 7.3.1 类数量与用量水位

```bash
# ① 最可靠：类数量趋势 —— Loaded 只涨不跌 = 元数据在堆积
jstat -class <pid> 1000 10
#   Loaded  Unloaded  ClassSpace(KB)

# ② 元空间用量水位（MU/MC 与 jstat -gcutil 第 7/8 列）
jstat -gc <pid> 1000 | awk '{print $7, $8}'
```

### 7.3.2 类加载日志与诊断注意

```bash
# ③ 看每一个类加载自何处（量大，适合复现环境）
-verbose:class                     # JDK 8
-Xlog:class+load=info              # JDK 9+

# ④ 诊断类命令 GC.class_stats 需诊断开关且输出格式随版本变化，
#    生产慎用；版本可用性不确定时优先用 ①②③。
jcmd <pid> VM.flags | grep -i metaspace   # 确认当前生效阈值
```

# 8. 元空间调优建议

## 8.1 原则与典型配置

### 8.1.1 五条调优原则

| 原则 | 理由 |
|------|------|
| **MaxMetaspaceSize 必设** | 默认无限 → 类泄漏吃光本地内存时，OS 杀进程（容器里是 OOMKilled）而不是 JVM 报 Metaspace OOM，损失更大 |
| **MetaspaceSize 调大到稳态水位附近** | 默认值太小 → 启动期连环"以元空间为借口的 GC"；建议取压测稳态 MU 的 1.2~1.5 倍 |
| 稳定应用两者可同值 | 避免水位线自适应来回抖动 |
| 监控 MU/MC 比例 | `jstat -gcutil <pid> 1000` 的 MU 列持续爬升不回落 = 类泄漏前兆（见 [20_jvm_tools](20_jvm_tools.md)） |
| 容器内必须设上限并计入内存预算 | 元空间不受 -Xmx 管辖，容器内存预算 = 堆 + 元空间 + Code Cache + 线程栈×N + 直接内存 + 杂项（详见 [23_container_jvm](23_container_jvm.md)） |

### 8.1.2 企业配置示例

```bash
# 企业常见配置（16G 容器，堆 8G）：
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=256m
-XX:ReservedCodeCacheSize=128m
```

## 8.2 修正一个常见误区

早期笔记常写"启动慢 → **减小** MetaspaceSize"：方向恰好相反。启动慢若是元空间触发 GC 导致，应**增大** MetaspaceSize（抬高首次 GC 水位），让启动期的类加载不再被 GC 打断。

# 9. 相关 JVM 参数总表

## 9.1 JDK 7 永久代 vs JDK 8+ 元空间对照

| JDK 7 (PermGen) | JDK 8+ (Metaspace) | 语义是否平移 |
|-----------------|--------------------|--------------|
| `-XX:PermSize=256m`（初始大小） | **无对应**：`-XX:MetaspaceSize` 是 GC 水位线不是初始值 | ✗ 语义变了 |
| `-XX:MaxPermSize=256m`（上限） | `-XX:MaxMetaspaceSize=256m`（上限，默认无限） | ✓ 迁移必改 |
| 字符串常量池参数随堆 | StringTable 在堆，无独立大小参数（intern 行为见 [6_heap](6_heap.md)） | ✓ |
| `-XX:-UseSplitSuperClass` 等 PermGen 细节 | 全部作废 | ✗ |
| GC 日志 `[PS PermGen: ...]` | `[Metaspace: used XXk]` / `jstat` MU/MC | 观测方式变 |

## 9.2 完整参数清单

```bash
# JDK 7 永久代（历史遗留，仅存档）
-XX:PermSize=256m
-XX:MaxPermSize=256m

# JDK 8+ 元空间
-XX:MetaspaceSize=256m          # 首次 GC 高水位线（不是初始大小！）
-XX:MaxMetaspaceSize=256m       # 上限，默认无限
-XX:MinMetaspaceFreeRatio=40    # GC 后空闲 <40% → 抬水位扩容
-XX:MaxMetaspaceFreeRatio=70    # GC 后空闲 >70% → 压水位缩容
-XX:CompressedClassSpaceSize=1g # 压缩类指针专用类区上限（默认 1G）
-XX:InitialBootClassLoaderMetaspaceSize=4m  # boot loader 元空间初始提交

# 代码缓存
-XX:InitialCodeCacheSize=...
-XX:ReservedCodeCacheSize=240m  # 分层编译默认 240M
-XX:+UseCodeCacheFlushing       # 默认开启
-XX:NonNMethodCodeHeapSize=5m          # JDK9+ 三段（以下两段合计≈233M）
-XX:ProfiledCodeHeapSize=122m
-XX:NonProfiledCodeHeapSize=111m

# 类加载/卸载观测
-verbose:class                  # JDK8 常用；JDK9+ 用 -Xlog:class+load
-XX:+TraceClassLoading          # JDK 8 参数；JDK 9 起由 -Xlog:class 统一日志取代
-XX:+TraceClassUnloading        # JDK8 已 deprecated，JDK9+ 用 -Xlog:class+unload
-XX:+CMSClassUnloadingEnabled   # 仅 CMS 需要；CMS 已于 JDK14 移除，JDK9+ 无效
```
