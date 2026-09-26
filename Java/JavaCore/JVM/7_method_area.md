
# 1. 方法区的演进

```
JDK 6及之前：永久代 (PermGen) — 在 JVM 堆中
JDK 7：       永久代 — 但字符串常量池、静态变量移到堆中
JDK 8+：      元空间 (Metaspace) — 使用本地内存，移除永久代
```

移除永久代的原因：
- 永久代有固定大小上限 `-XX:MaxPermSize`，容易 OOM。
- 永久代调优困难，不同应用类元数据大小差异大。
- 永久代 GC 复杂（与堆混合回收）。
- JRockit 没有永久代概念，HotSpot 与 JRockit 合并后统一为元空间。

```
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
                          │ └──────────────┘     │
                          └──────────────────────┘
```

# 2. 元空间的内存分配

- 元空间由多个 **Metachunk** 组成，多个 Chunk 属于同一个 **Metablock**。
- 类加载器持有的类元数据从对应的 Metachunk 中分配。
- 类加载器被回收时，它持有的 Chunk 归还给元空间。
- 元空间大小可以自动扩展，受物理内存限制。

```bash
# 元空间参数
-XX:MetaspaceSize=128m        # 元空间初始大小(也是触发GC的阈值)
-XX:MaxMetaspaceSize=256m     # 元空间最大大小(默认无限制)
-XX:MinMetaspaceFreeRatio=40  # GC 后最小空闲比例(低于则扩容)
-XX:MaxMetaspaceFreeRatio=70  # GC 后最大空闲比例(高于则缩容)
```

# 3. 元空间 GC

- 元空间 GC 发生在 Full GC 时或元空间达到 `MetaspaceSize` 阈值时。
- GC 回收的是无用的类对应的元数据空间。
- 一个类被判定为"无用类"的条件：
  1. 该类所有实例已回收
  2. 加载该类的 ClassLoader 已回收
  3. 该类的 Class 对象未被引用

**元空间 OOM 的常见原因：**
```java
// CGLIB 动态代理无限生成类 → Metaspace OOM
while (true) {
    Enhancer enhancer = new Enhancer();
    enhancer.setSuperclass(User.class);
    enhancer.setUseCache(false);       // 不使用缓存
    enhancer.setCallback((MethodInterceptor) (obj, method, args, proxy) -> {
        return proxy.invokeSuper(obj, args);
    });
    enhancer.create();
}
// Exception: java.lang.OutOfMemoryError: Metaspace
```

**查看元空间使用：**
```bash
jstat -gc <pid>
# 查看 MU (元空间使用量)、MC (元空间容量)
```

# 4. 运行时常量池

- 类加载后，Class 文件的常量池被放入方法区的运行时常量池中。
- **符号引用 → 直接引用** 的解析也在这里发生。
- 动态加入：`String.intern()` 等。

**Class 文件常量池 vs 运行时常量池：**

| 对比 | Class 文件常量池 | 运行时常量池 |
|------|-----------------|-------------|
| 存储位置 | Class 文件中 | 方法区（元空间） |
| 存在形态 | 静态的二进制数据 | JVM 运行时动态使用 |
| 动态性 | 编译时确定 | 运行时可通过 intern() 等方法添加 |
| 符号引用 | 存储符号引用 | 可解析为直接引用 |

# 5. 代码缓存（Code Cache）

- JIT 编译器将热点代码编译为本地机器码后，存储在代码缓存中。
- 代码缓存也在本地内存中，与元空间分开。
- 当代码缓存满了，JVM 会关闭 JIT 编译（危险！性能急剧下降）。

```bash
-XX:InitialCodeCacheSize=128m    # 初始大小
-XX:ReservedCodeCacheSize=256m   # 保留大小（最大）
-XX:+UseCodeCacheFlushing        # 允许刷出较少使用的编译代码（默认开启）
```

代码缓存满时的表现：
```
Java HotSpot(TM) 64-Bit Server VM warning:
CodeCache is full. Compiler has been disabled.
```

# 6. 静态变量的存储位置演变

```java
public class Example {
    public static int COUNT = 10;
    public static String NAME = "hello";
}
```

- JDK 6：`COUNT` 和 `NAME` 的引用在永久代的类信息附近。
- JDK 7+：静态变量随 Class 对象存储（Class 对象在堆中），所以静态变量实际在堆中。
- `NAME` 指向的字符串 "hello" 在**字符串常量池**（堆中）。

# 7. 实战：元空间溢出

```java
// -XX:MaxMetaspaceSize=10m -XX:MetaspaceSize=10m
public class MetaspaceOOM {
    public static void main(String[] args) {
        for (int i = 0; ; i++) {
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(OOMObject.class);
            enhancer.setUseCache(false);
            enhancer.setCallback((MethodInterceptor) (obj, method, args1, proxy) ->
                    proxy.invokeSuper(obj, args1));
            enhancer.create();
            System.out.println("Created: " + i);
        }
    }
    
    static class OOMObject {}
}
// Caused by: java.lang.OutOfMemoryError: Metaspace
```

# 8. 元空间调优建议

| 场景 | 建议 |
|------|------|
| 一般应用 | 使用默认值即可 |
| 大量使用动态代理/反射/CGLIB | 适当增大 `MaxMetaspaceSize` |
| 微服务/容器 | 必须设置 `MaxMetaspaceSize`（默认无限制可能撑爆容器内存） |
| 启动慢 | 减小 `MetaspaceSize`，减少初次 GC 阈值 |
| 应用稳定 | `MetaspaceSize` 和 `MaxMetaspaceSize` 设相同值 |

```bash
# 企业常见配置示例
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=256m
-XX:ReservedCodeCacheSize=128m
```

# 9. 方法区相关 JVM 参数汇总

```bash
# JDK 7 永久代
-XX:PermSize=256m
-XX:MaxPermSize=256m

# JDK 8+ 元空间
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=256m

# 代码缓存
-XX:ReservedCodeCacheSize=128m
-XX:+UseCodeCacheFlushing

# 类卸载
-XX:+TraceClassLoading      # 追踪类加载
-XX:+TraceClassUnloading    # 追踪类卸载
-XX:+CMSClassUnloadingEnabled  # CMS 时允许卸载类（Java 8 CMS）
```