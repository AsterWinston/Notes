
# 1. 执行引擎概述

- JVM 的执行引擎负责执行字节码指令。
- 主要方式：解释执行 + 即时编译（JIT）+ 垃圾回收。

```
字节码(bytecode) → 执行引擎 → 本地机器码 → CPU 执行
                      │
         ┌────────────┼────────────┐
         │            │            │
    解释器(Interpreter)  JIT编译器  GC
      逐条解释执行    热点代码编译
```

# 2. 解释器与编译器的协作

## 2.1 解释器（Interpreter）

- 逐条将字节码翻译为机器码并执行。
- **优点**：启动快，无需编译等待。
- **缺点**：运行效率低（重复翻译相同的字节码）。

## 2.2 JIT 编译器（Just-In-Time Compiler）

- 将热点代码（频繁执行的方法/循环）编译为本地机器码，缓存后直接执行。
- **优点**：运行效率高（编译后接近 C/C++ 速度）。
- **缺点**：编译耗时，启动阶段可能比解释更慢。

## 2.3 混合模式（默认）

```
启动阶段：解释器快速响应
     ↓
运行时发现热点代码
     ↓
JIT 后台编译为本地机器码
     ↓
后续调用直接执行机器码（快）
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

# 3. 热点探测

HotSpot 使用**基于计数器的热点探测**。

## 3.1 方法调用计数器

- 统计方法被调用的次数。
- 超过阈值 → 提交 JIT 编译。
- 有半衰期（热度衰减）：在 GC 时计数器减半，防止冷方法因为历史调用多而被误编译。

```bash
-XX:CompileThreshold=10000    # 方法调用计数器阈值（默认：Client 1500, Server 10000）
-XX:CounterHalfLifeTime=30    # 半衰周期（秒）
-XX:+UseCounterDecay          # 开启热度衰减（默认）
```

## 3.2 回边计数器

- 统计循环体执行的次数。
- 超过阈值 → 触发 **OSR（On-Stack Replacement，栈上替换）**。
- OSR：不等待方法返回，直接在循环中途将解释执行的代码替换为编译后的机器码。

# 4. C1 和 C2 编译器

HotSpot 包含两个 JIT 编译器：

| 编译器 | 全称 | 特点 | 适用场景 |
|--------|------|------|----------|
| C1 (Client) | Client Compiler | 编译快，优化简单，启动快 | 桌面应用、客户端 |
| C2 (Server) | Server Compiler | 编译慢，深度优化，运行高效 | 长期运行的服务端 |

## 4.1 C1 编译（Client Compiler）

- 关注**启动时间**和**局部优化**。
- 编译速度快，但优化程度浅。
- 典型优化：方法内联、冗余消除、基本复制传播等。

```bash
-XX:TieredStopAtLevel=1
```

C1 的编译层次（Tier 1-3）：
```
Level 0: 纯解释执行
Level 1: C1 编译，无 profiling（全速）
Level 2: C1 编译，简单 profiling（方法调用次数、回边次数）
Level 3: C1 编译，完整 profiling（分支概率、类型信息）
Level 4: C2 编译（最深度优化）
```

## 4.2 C2 编译（Server Compiler）

- 关注**运行时性能**。
- 编译速度慢，但生成的代码质量极高。
- 使用大量高级优化技术（逃逸分析、内联、循环展开、虚方法内联等）。

# 5. 分层编译（Tiered Compilation）

- JDK 7+ 引入，JDK 8 默认开启。
- 结合了 C1 的快速启动和 C2 的深度优化。

```
分层编译的工作流：
    
方法被多次调用：
  Level 0 (解释) → Level 3 (C1 + 完整 profiling) → Level 4 (C2 深度优化)
  
不太热的方法：
  Level 0 (解释) → Level 1 (C1 无 profiling)
  
循环热点：
  Level 0 → Level 2 → Level 4
```

**各层说明：**
```
0: 解释器执行
1: C1 不带 profiling（简单方法快速编译）
2: C1 带简单 profiling（方法计数 + 回边计数）
3: C1 带完整 profiling（收集分支概率、类型信息等，供 C2 使用）
4: C2 深度优化编译
```

```bash
-XX:+TieredCompilation         # 开启分层编译（JDK 8+ 默认）
-XX:TieredStopAtLevel=4        # 最高到达 Level 4
# 不同编译层次线程数
-XX:CICompilerCount=3          # 编译器线程总数（C1 + C2）
```

# 6. OSR（On-Stack Replacement）

- 栈上替换：如果一个方法中有一个热点循环，不等方法返回，直接在**循环入口处**将解释执行替换为编译执行。

```java
public void heavyLoop() {
    // 前几次解释执行
    for (int i = 0; i < 1_000_000; i++) {
        // 循环太热，触发 OSR
        // JIT 编译后 从这里开始执行机器码
        compute(i);
    }
    // OSR 只在循环期间有效，下次调用方法仍然解释执行
}
```

- OSR 编译的代码不如 C2 普通编译（因为没有方法入口的 prologue 优化）。
- 如果方法本身也很热（方法调用计数器超阈值），会再有一次普通 C2 编译，质量更高。

# 7. 编译线程

- JIT 编译由后台线程异步完成。
- 初始编译线程数：`CPU 核心数 / 2`（分层编译下）。

```bash
# 查看编译线程数
java -XX:+PrintFlagsFinal -version | grep CICompilerCount

# 手动设置
-XX:CICompilerCount=2
```

- 编译线程是守护线程，当编译任务过多时会排队。
- 如果编译线程繁忙，方法会继续解释执行，直到编译完成。

# 8. 编译日志

```bash
# 打印被编译的方法
-XX:+PrintCompilation

# 输出格式：
# 时间戳  编译ID  属性  方法名  大小
#   123    45     n    java.lang.String::hashCode (55 bytes)
#   456    46     s    com.example.MyClass::compute (123 bytes)

# 属性说明：
# n = native 方法
# s = synchronized 方法
# ! = 有异常处理器
# % = OSR 编译
# 空白 = 普通编译
```

**详细编译日志（Java 9+）：**
```bash
-Xlog:compilation*=info:file=compile.log
```

# 9. 编译相关常见问题

## 9.1 代码缓存满

```
Java HotSpot(TM) 64-Bit Server VM warning: 
CodeCache is full. Compiler has been disabled.
```

**后果**：所有代码回退到解释执行，性能急剧下降。

**解决**：
```bash
-XX:ReservedCodeCacheSize=256m   # 增大代码缓存
-XX:+UseCodeCacheFlushing        # 允许刷出冷的编译代码（默认开启）
```

## 9.2 编译阈值调优

```bash
# 降低阈值 → 更早 JIT 编译 → 启动预热更快，但编译开销增加
-XX:CompileThreshold=1000       # 默认 Client: 1500, Server: 10000

# 增加阈值 → 更晚 JIT 编译 → 启动开销小，但热点达到峰值时间更长
-XX:CompileThreshold=20000
```

## 9.3 去优化（Deoptimization）

- JIT 基于假设做激进优化，当假设不成立时需要"去优化"——回退到解释执行。
- 常见触发条件：
  - 类层次结构变化（新加载了子类/实现类）
  - profiling 信息不足以支持激进的类型预测
  - 去优化过多会影响性能

```bash
# 打印去优化日志
-XX:+PrintDeoptimizationDetails
```

# 10. 即时编译器实战参数

```bash
# 基础编译参数
-XX:+TieredCompilation            # 分层编译
-XX:CICompilerCount=3             # 编译线程数
-XX:ReservedCodeCacheSize=256m    # 代码缓存

# 编译阈值
-XX:CompileThreshold=10000        # 方法调用阈值
-XX:BackEdgeThreshold=100000      # 回边计数器阈值 (OSR)

# 编译优化控制
-XX:+DoEscapeAnalysis             # 逃逸分析
-XX:+EliminateAllocations         # 标量替换
-XX:+EliminateLocks               # 同步消除
-XX:+AggressiveOpts               # 激进优化（已废弃）

# 调试
-XX:+PrintCompilation             # 打印编译日志
-XX:+PrintInlining                # 打印内联决策
-XX:+UnlockDiagnosticVMOptions    # 解锁诊断参数
-XX:+LogCompilation               # 输出编译日志到文件
```