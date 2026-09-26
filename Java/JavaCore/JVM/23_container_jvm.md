
# 1. 容器中的 JVM 挑战

```
传统部署：                 容器化部署：
┌──────────────┐          ┌──────────────────┐
│   物理机/VM  │          │     Pod          │
│  ┌────────┐  │          │ ┌──────────────┐ │
│  │  JVM   │  │          │ │     JVM      │ │
│  │ 8个CPU │  │          │ │ 看到所有CPU  │ │
│  │ 16G内存│  │          │ │ 看到所有内存 │ │
│  └────────┘  │          │ └──────────────┘ │
│              │          │  limits:         │
│              │          │    cpu: 2        │
│              │          │    memory: 4Gi   │
└──────────────┘          └──────────────────┘

问题：JVM 默认读取宿主机的 CPU 和内存，
     不知道容器的 limits！
```

# 2. Java 版本的容器感知演进

| 版本 | 容器支持情况 |
|------|-------------|
| ≤ JDK 8u131 | 不支持容器限制，读取宿主机资源 |
| JDK 8u131+ | 实验性支持（需显式开启） |
| JDK 8u191+ | `-XX:+UseContainerSupport` 默认开启 |
| JDK 10+ | `UseContainerSupport` 默认开启 |
| JDK 15+ | 全面支持 cgroup v2 |

## 2.1 关键参数

```bash
# JDK 8u191+ 默认开启
-XX:+UseContainerSupport      # 启用容器支持
-XX:ActiveProcessorCount=2    # 手动指定可用 CPU 数（覆盖自动检测）
-XX:InitialRAMPercentage=75   # 初始堆占容器内存的百分比
-XX:MaxRAMPercentage=75       # 最大堆占容器内存的百分比
-XX:MinRAMPercentage=50       # 小内存时堆占比
```

## 2.2 容器内存分配详解

```
容器内存 = 4Gi

JVM 内存构成：
├── 堆 (Xmx/Xms)         ← -XX:MaxRAMPercentage=75 → 约 3Gi
├── Metaspace             ← -XX:MaxMetaspaceSize → 建议设为 256m
├── 线程栈 (N × Xss)     ← 比如 100线程 × 1MB = 100MB
├── 直接内存              ← -XX:MaxDirectMemorySize
├── 代码缓存              ← -XX:ReservedCodeCacheSize
├── GC 开销               ← 各类 GC 数据结构
├── Native JVM            ← JVM 自身的 C++ 对象
└── 其他 (NIO Buffer 等)

总内存占用 > 堆大小  ← 容器 OOMKilled 的原因！
```

**推荐的内存分配：**
```
容器内存 = 4Gi
  ├── 堆 (75%)        ≈ 3Gi
  ├── Metaspace        = 256m
  ├── 线程栈           ≈ 100MB (根据线程数调整)
  ├── 直接内存         = 100MB
  ├── 其他             ≈ 200MB
  └── 安全余量         ≈ 400MB  ← 重要！
```

# 3. 常见容器化 JVM 问题

## 3.1 问题 1：OOMKilled

- 容器内存超限被 Kubernetes kill。
- **原因**：堆只占了一部分，但 JVM 整体内存（含 native）超过容器限制。

```bash
# 错误配置（只有 4Gi 的容器）
-Xmx3g                       # 堆 3G
# 元空间默认无限制            # 可能到 1G
# 线程栈 200 线程 × 1MB      # 200MB
# 总内存 > 4Gi → OOMKilled

# 正确配置
-Xms2304m -Xmx2304m           # 堆 ~2.25G (< 75%)
-XX:MaxMetaspaceSize=256m
-XX:MaxDirectMemorySize=128m
-XX:ReservedCodeCacheSize=128m
-Xss256k                       # 减小栈，允许更多线程
```

## 3.2 问题 2：CPU 限流（Throttling）

- 容器分配 2 核，JVM 看到宿主机 32 核 → 创建 16 个 GC 线程 → CPU 竞争 / 限流。

```bash
# 解决方案
-XX:ActiveProcessorCount=2     # 告诉 JVM 只有 2 核
# 或自动检测
-XX:+UseContainerSupport       # 自动读取 cgroup 限制
```

## 3.3 问题 3：GC 线程过多

```bash
# GC 线程数默认 = CPU 核数 × 5/8（Parallel GC）
# 或 ≈ CPU 核数 × 1/4（G1 并发线程）

# 容器 2 核时：
# ParallelGCThreads ≈ 2
# ConcGCThreads ≈ 1

# 可手动指定：
-XX:ParallelGCThreads=2
-XX:ConcGCThreads=1
```

## 3.4 问题 4：/proc/cpuinfo 和 /proc/meminfo

- JDK 8u131 之前，JVM 读 `/proc/cpuinfo` 和 `/proc/meminfo`，获得宿主机数据。
- 新版本 JDK 会读取 cgroup 配置（`/sys/fs/cgroup/`）。

# 4. 容器化最佳实践

## 4.1 推荐配置模板

```bash
# 容器 2CPU / 4Gi 内存
-XX:MaxRAMPercentage=75              # 堆占 75%
-Xss256k                             # 栈 256K
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=256m
-XX:MaxDirectMemorySize=128m
-XX:ReservedCodeCacheSize=128m
-XX:+UseContainerSupport             # 容器感知
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:+HeapDumpOnOutOfMemoryError
-XX:+ExitOnOutOfMemoryError          # k8s 自动重启
```

## 4.2 Dockerfile 建议

```dockerfile
FROM eclipse-temurin:21-jre

# 使用 exec 形式（接收信号）
ENTRYPOINT ["java", \
    "-XX:MaxRAMPercentage=75", \
    "-XX:+UseContainerSupport", \
    "-XX:+UseG1GC", \
    "-XX:+ExitOnOutOfMemoryError", \
    "-jar", "/app.jar"]
```

## 4.3 Kubernetes 配置

```yaml
resources:
  requests:
    memory: "4Gi"
    cpu: "2"
  limits:
    memory: "4Gi"      # request = limit 更稳定
    cpu: "2"

env:
  - name: JAVA_OPTS
    value: >
      -XX:MaxRAMPercentage=75
      -XX:+UseG1GC
      -XX:ActiveProcessorCount=2
```

# 5. JVM 性能里程碑（各版本重要更新）

| 版本 | JVM 关键更新 |
|------|-------------|
| JDK 5 | 泛型、枚举、自动装箱、J.U.C |
| JDK 6 | 锁优化（偏向锁、轻量级锁、自旋锁、锁消除）、CMS 增强 |
| JDK 7 | G1 引入、`invokedynamic`、逃逸分析改进、永久代中字符串常量池移到堆 |
| JDK 8 | 永久代移除→元空间、Lambda、分层编译默认开启、G1 正式版 |
| JDK 9 | G1 为默认 GC、模块化（Jigsaw）、字符串压缩（Compact Strings）、统一日志 |
| JDK 10 | 并行 Full GC (G1)、容器感知增强 |
| JDK 11 | ZGC 实验性、CMS 废弃、JFR 开源 |
| JDK 12 | Shenandoah 实验性、G1 可中断 Mixed GC |
| JDK 13 | 动态 CDS 归档（AppCDS） |
| JDK 14 | CMS 移除、ZGC 支持 macOS/Windows、JFR 事件流 |
| JDK 15 | ZGC 正式版、Shenandoah 正式版、偏向锁默认关闭 |
| JDK 17 LTS | 密封类、G1 NUMA 感知、ZGC 持续优化 |
| JDK 19-20 | 虚拟线程（Virtual Threads）预览 |
| JDK 21 LTS | 虚拟线程正式版、分代 ZGC（实验性）、字符串模板 |

# 6. JVM 高频面试考点

## 6.1 必问（基础）
1. **JVM 内存模型**：堆、栈、方法区/元空间、程序计数器。
2. **类加载过程**：加载→验证→准备→解析→初始化，双亲委派模型。
3. **对象创建过程**：分配内存→零值→对象头→init，对象内存布局。
4. **GC 判断方法**：引用计数法 vs 可达性分析，GC Roots。
5. **GC 算法**：标记-清除、标记-复制、标记-整理，三色标记。
6. **GC 收集器**：Serial、Parallel、CMS、G1 各自特点和适用场景。

## 6.2 必问（进阶）
7. **volatile 原理**：可见性、禁止重排序、内存屏障、happens-before。
8. **synchronized 优化**：偏向锁→轻量级锁→重量级锁，锁消除/粗化。
9. **CAS 与 ABA**：CAS 原理、ABA 问题及解决（版本号）。
10. **DCL 为什么加 volatile**：防止指令重排序导致的半初始化对象。
11. **OOM 排查**：各区域 OOM 的原因和排查方法。
12. **内存泄漏排查**：MAT 分析、ThreadLocal 泄漏、常见泄漏模式。

## 6.3 加分项（高级）
13. **ZGC/Shenandoah**：染色指针、读屏障、Brooks 前向指针。
14. **JIT 编译**：逃逸分析、方法内联、标量替换、分层编译。
15. **Arthas 实战**：dashboard、thread、trace、watch、profiler。
16. **容器化 JVM**：容器感知、OOMKilled、MaxRAMPercentage。
17. **线上故障排查**：CPU 飙高、OOM、死锁、GC 频繁的完整排查链路。
18. **happens-before 规则**：8 条规则分别背诵。
19. **Class 文件结构**：魔数、版本号、常量池、access flag。
20. **对象大小计算**：Mark Word + Klass Pointer + 实例数据 + padding，压缩指针。

## 6.4 面试模拟

**Q: CMS 和 G1 区别？**
- CMS 是老年代收集器，G1 是全堆收集器。
- CMS 基于标记-清除，有碎片；G1 基于标记-复制+整理，无碎片。
- CMS 停顿目标 ~200ms，G1 可配置 `-XX:MaxGCPauseMillis`。
- G1 有 RSet 和 CSet，可预测停顿。
- CMS 已在 JDK 9 废弃，JDK 14 移除。

**Q: 什么时候触发 Full GC？**
- `System.gc()`
- 老年代空间不足
- 元空间空间不足
- CMS/G1 并发模式失败
- Minor GC 时分配担保失败

**Q: 如何排查 CPU 飙高？**
1. `top -c` → 找到高 CPU 进程 PID
2. `top -Hp <pid>` → 找到高 CPU 线程 TID
3. 转为十六进制 → `jstack <pid>` 定位线程栈
4. 或用 Arthas：`thread -n 5` / `profiler start`

**Q: 类加载器有哪些？双亲委派为什么被打破？**
- Bootstrap / Platform / Application
- 打破场景：SPI（JDBC 的 ServiceLoader）、Tomcat（多应用隔离）、OSGi（网状依赖）
- 通过重写 `loadClass()` 或使用线程上下文类加载器

# 7. JVM 学习路线图

```
第一阶段：JVM 基础
├── JVM 架构和运行时数据区
├── 类加载机制（双亲委派）
├── 对象内存布局
└── 对象创建过程

第二阶段：GC 深入
├── 可达性分析
├── GC 算法（标记-清除/复制/整理、三色标记）
├── 经典收集器（Serial/Parallel/CMS/G1）
├── 低延迟收集器（ZGC/Shenandoah）
└── GC 调优实战

第三阶段：字节码与执行
├── Class 文件结构
├── 字节码指令
├── 执行引擎（解释器/JIT）
└── JIT 编译优化

第四阶段：并发与内存模型
├── JMM 与 happens-before
├── volatile/synchronized 底层
├── 锁升级与优化
└── CAS 原理

第五阶段：故障排查（企业实战）
├── 命令行工具（jps/jstat/jmap/jstack/jcmd）
├── 可视化工具（VisualVM/JMC/Arthas/MAT）
├── OOM/CPU/死锁/GC 频率排查
└── 容器化 JVM 调优
```

# 8. 推荐资源

| 资源 | 说明 |
|------|------|
| 《深入理解 Java 虚拟机》（周志明） | JVM 圣经，必读 |
| 《Java 虚拟机规范》 | 官方规范 |
| 《实战 Java 虚拟机》 | 偏实战 |
| GC 日志分析 | https://gceasy.io/ |
| 美团技术博客 JVM 系列 | 企业实战案例丰富 |
| Oracle JVM 调优指南 | 官方文档 |
| perfma 参数调优 | 懒人调参平台 |
| JOL (Java Object Layout) | 对象内存分析工具 |
