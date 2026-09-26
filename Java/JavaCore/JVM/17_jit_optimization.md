
# 1. 方法内联（Method Inlining）

- **最重要的编译优化，没有之一**。
- 将目标方法的代码复制到调用处。
- 目的：消除方法调用开销（栈帧创建、参数传递、返回），同时为后续优化创造条件。

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
    return 1 + 2;                     // 内联展开
}
```

**内联判断条件：**
- `-XX:MaxInlineSize=35`：方法字节码 < 35 字节（内联）
- `-XX:FreqInlineSize=325`：热点方法字节码 < 325 字节（内联）
- `-XX:MaxInlineLevel=9`：最大内联嵌套层数
- 方法不可内联的情况：未加载的类、虚方法（非 final/非 private）、超过大小等。

**查看内联决策：**
```bash
-XX:+PrintInlining
# 输出每个方法的内联决策：
# @ 123  java.lang.String::hashCode (55 bytes)
#   @ 130  java.lang.String::charAt (25 bytes)   inline (hot)
#   @ 145  java.lang.String::length (11 bytes)   inline (hot)
#   @ 160  com.foo.Bar::check (110 bytes)        too big
```

# 2. 逃逸分析（Escape Analysis）

详见第 15 节。这里补充编译层面的影响。

JIT 编译器判断一个对象是否"逃逸"：
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

# 3. 锁消除（Lock Elimination）

JIT 通过逃逸分析发现同步块被对象独占，则消除锁。

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

```bash
# 查看锁消除
-XX:+PrintEliminateLocks
```

# 4. 锁粗化（Lock Coarsening）

连续的加锁/解锁合并为一次大锁。

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

# 5. 公共子表达式消除（Common Subexpression Elimination）

```java
// 优化前
int a = (b + c) * d + (b + c) * e;
// 编译后
int tmp = b + c;       // 公共子表达式只计算一次
int a = tmp * d + tmp * e;
```

# 6. 数组边界检查消除（Array Bounds Check Elimination）

Java 每次数组访问都会检查索引是否越界，JIT 可以消除明显不会越界的检查。

```java
// 优化前
int[] arr = new int[10];
for (int i = 0; i < arr.length; i++) {
    arr[i] = i;       // 每次都有边界检查
}

// JIT 优化后：循环内移除边界检查（因为 i < arr.length 保证了不越界）
for (int i = 0; i < arr.length; i++) {
    arr[i] = i;       // 无边界检查
}
```

# 7. 循环优化

## 7.1 循环展开（Loop Unrolling）

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

## 7.2 循环表达式外提（Loop Invariant Code Motion）

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

## 7.3 循环判断外提（Loop Unswitching）

```java
// 优化前
for (int i = 0; i < 1000; i++) {
    if (flag) {
        doA(i);
    } else {
        doB(i);
    }
}

// 优化后
if (flag) {
    for (int i = 0; i < 1000; i++) { doA(i); }
} else {
    for (int i = 0; i < 1000; i++) { doB(i); }
}
```

# 8. 虚方法内联（Virtual Method Inlining）

- JIT 遇到虚方法调用时，需要确定实际调用哪个方法才能内联。
- 使用**类型预测**：基于 profiling 信息猜测最可能的接收者类型。

```java
// 多态调用
interface Animal { void sound(); }
class Dog implements Animal { public void sound() { ... } }
class Cat implements Animal { public void sound() { ... } }

public void makeSound(Animal a) {
    a.sound();  // 虚方法调用
}

// JIT 通过 profiling 发现 99% 的调用是 Dog 类型
// 内联为：
public void makeSound(Animal a) {
    if (a instanceof Dog) {
        // Dog.sound() 的内联代码
    } else {
        a.sound(); // 罕见路径，走虚方法调用
    }
}
```

**单形调用（Monomorphic）：** 绝大多数只调用一种类型 → 内联
**双形调用（Bimorphic）：** 两种类型 → 两条内联分支
**过多形态（Megamorphic）：** 三种及以上 → 退化为虚方法调用

# 9. 空值检查消除（Null Check Elimination）

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

# 10. 窥孔优化（Peephole Optimization）

- 在生成的机器码上做局部"窥孔"级别的优化。
- 消除冗余的加载/存储指令、合并代数运算等。

# 11. 分支预测优化

- JIT 根据 profiling 收集的分支概率信息，优化 CPU 的分支预测。
- 常见分支（高概率路径）放在"不跳转"的路径上（CPU 流水线更友好）。

```java
// 如果 profiling 显示 error 极少发生
if (error) {
    handleError();  // 罕见路径 → 放在跳转指令处
}
// 正常路径 → 顺序执行，不跳转，CPU 流水线友好
processNormal();
```

# 12. 激进优化与去优化

```java
public void process(Object obj) {
    if (obj instanceof String) {
        String s = (String) obj;
        // JIT 可能猜测 obj 总是 String，永远走这条路径
        // 优化后等效于直接操作 obj 为 String
    }
}
```

- 如果后来有不 String 的对象传入 → JIT 需要**去优化**（Deoptimization），回退到解释执行。
- 去优化后，JIT 会重新 profiling，生成新的、更准确的编译版本。

# 13. 实战：观察 JIT 编译效果

## 13.1 JMH 微基准测试

```java
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Thread)
public class JITBenchmark {
    private int a = 10;
    private int b = 20;
    
    @Benchmark
    public int testInline() {
        return add(a, b);      // 会被内联
    }
    
    private int add(int x, int y) {
        return x + y;
    }
    
    @Benchmark
    public int testNoInline() {
        return addComplex(a, b); // 可能不会内联（更复杂）
    }
    
    private int addComplex(int x, int y) {
        // 大量代码...超过内联阈值
        return x + y;
    }
}
```

## 13.2 查看 JIT 活动

```bash
# 打印被 JIT 编译的方法
java -XX:+PrintCompilation MyApp

# 打印内联决策
java -XX:+PrintInlining MyApp

# 查看编译后代码（汇编级别，需 HSDis 插件）
java -XX:+PrintAssembly MyApp
```

# 14. 编译优化参数速查

```bash
# 内联相关
-XX:MaxInlineSize=35             # 普通方法内联阈值（字节码大小）
-XX:FreqInlineSize=325           # 热点方法内联阈值
-XX:MaxInlineLevel=9             # 最大内联深度
-XX:InlineSmallCode=2000         # 已经被编译的方法大小 < N 字节可内联

# 逃逸分析与分配
-XX:+DoEscapeAnalysis            # 逃逸分析
-XX:+EliminateAllocations        # 标量替换
-XX:+EliminateLocks              # 锁消除

# 循环优化
-XX:+UseLoopPredicate            # 循环断言（边界检查优化）
-XX:+UseCountedLoops             # 可计数循环优化
-XX:LoopUnrollLimit=60           # 循环展开的节点数限制

# 编译日志
-XX:+PrintCompilation            # 编译日志
-XX:+PrintInlining               # 内联日志
-XX:+UnlockDiagnosticVMOptions   # 解锁诊断选项
-XX:+LogCompilation              # 记录编译日志到文件
-XX:+PrintOptoAssembly           # C2 汇编输出（需 debug JDK）
```
