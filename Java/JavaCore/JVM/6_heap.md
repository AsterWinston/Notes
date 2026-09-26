
# 1. 堆的内存划分

```
物理上：不连续的内存空间（逻辑上连续）

┌──────────────────────────────────────────────────┐
│                     堆 (Heap)                    │
│                                                  │
│  ┌──────────────────────────────────────────┐    │
│  │            新生代 (Young Generation)       │    │
│  │                                          │    │
│  │  ┌──────────┬───────────┬───────────┐    │    │
│  │  │   Eden   │ Survivor0 │ Survivor1 │    │    │
│  │  │  (80%)   │   (10%)   │   (10%)   │    │    │
│  │  └──────────┴───────────┴───────────┘    │    │
│  └──────────────────────────────────────────┘    │
│                                                  │
│  ┌──────────────────────────────────────────┐    │
│  │            老年代 (Old Generation)         │    │
│  │           Tenured / Old Gen               │    │
│  └──────────────────────────────────────────┘    │
└──────────────────────────────────────────────────┘

比例关系：
  Young : Old     = 1 : 2    (-XX:NewRatio=2)
  Eden : S0 : S1  = 8 : 1 : 1 (-XX:SurvivorRatio=8)
```

分代收集理论假说：
- **弱分代假说**：绝大多数对象都是朝生夕死的。
- **强分代假说**：熬过越多次 GC 的对象越难消亡。
- **跨代引用假说**：跨代引用相对于同代引用是极少数。

# 2. 新生代（Young Generation）

- 存放新创建的对象（大多数"朝生夕死"）。
- 细分为 Eden 区和两个 Survivor 区（S0 / S1，也叫 From / To）。
- 触发 Minor GC / Young GC。

## 2.1 Eden 区

- 大多数新对象在 Eden 分配内存。
- 空间不足时触发 Minor GC。
- TLAB（Thread Local Allocation Buffer）内存在 Eden 区中。

## 2.2 Survivor 区

- 两个 Survivor 区始终有一个为空（To 区）。
- Minor GC 时，存活对象从 Eden + From 区复制到 To 区。
- 对象在 Survivor 区中每熬过一次 Minor GC，年龄 +1。
- 年龄达到阈值（默认 15），晋升到老年代。

为什么需要两个 Survivor 区？— 避免内存碎片化。复制算法需要一块完整的空闲空间来存放存活对象。

# 3. 老年代（Old Generation）

- 存放生命周期较长的对象。
- 触发 Major GC / Full GC。
- Full GC 通常比 Minor GC 慢 10 倍以上（因此要避免频繁 Full GC）。

对象进入老年代的途径：
1. **年龄晋升**：Survivor 区对象年龄超过 `-XX:MaxTenuringThreshold`（默认 15）。
2. **大对象直接分配**：超过 `-XX:PretenureSizeThreshold` 的对象直接在老年代分配（仅 Serial/ParNew 有效）。
3. **Survivor 空间不足**：Survivor 区放不下的对象直接进入老年代（分配担保）。
4. **动态年龄判断**：Survivor 区中相同年龄的对象大小之和超过 Survivor 空间的一半，则年龄 >= 该年龄的对象直接晋升。

# 4. TLAB（Thread Local Allocation Buffer）

- 多线程同时分配对象时，为了避免对同一块内存区域的竞争，JVM 为每个线程在 Eden 中分配一小块专属内存。
- 线程在自己的 TLAB 中分配对象，无需加锁。
- TLAB 用完了，分配新的 TLAB。

```bash
-XX:+UseTLAB             # 默认开启
-XX:+PrintTLAB           # 打印 TLAB 使用情况
-XX:TLABSize=<size>      # 设置 TLAB 大小
-XX:-ResizeTLAB          # 禁止动态调整 TLAB 大小
```

```
┌─────────────────────────────────────────────┐
│                  Eden 区                      │
│  ┌──────┬──────┬──────┬──────┬───────────┐  │
│  │TLAB1 │TLAB2 │TLAB3 │TLAB4 │  共享区域   │  │
│  │线程1  │线程2  │线程3  │线程4  │  (竞争)   │  │
│  └──────┴──────┴──────┴──────┴───────────┘  │
└─────────────────────────────────────────────┘
```

# 5. 对象晋升流程

```
  new Object()
       │
       ▼
  ┌─────────┐
  │尝试栈上分配│──成功──▶ 栈上对象（GC无需回收）
  └────┬────┘
       │失败
       ▼
  ┌─────────┐
  │尝试TLAB分配│──成功──▶ Eden 区分配完成
  └────┬────┘
       │失败(TLAB满)
       ▼
  ┌─────────────┐
  │Eden 共享区域CAS│──失败──▶ 新TLAB或降级到CAS分配
  │   分配      │
  └────┬────────┘
       │分配完成
       ▼
  ┌──────────┐
  │Eden 区中  │ ──Minor GC──▶ Survivor 区
  └──────────┘                  │
                         年龄 >= 15 或 动态年龄
                                │
                                ▼
                           老年代 (Old Gen)
```

# 6. 字符串常量池（String Pool）

- Java 7 之前：在永久代中。
- Java 7+ 之后：移到堆中，可以被 GC 回收。

```java
String s1 = "abc";                    // 字面量 → 直接入池
String s2 = new String("abc");        // 创建新对象在堆中（不在池中）
String s3 = s2.intern();              // intern()：若池中有则返回池中引用，否则入池
System.out.println(s1 == s2);         // false
System.out.println(s1 == s3);         // true
```

示例：`intern()` 在不同场景下的行为：
```java
String s1 = new StringBuilder("hel").append("lo").toString();
System.out.println(s1.intern() == s1);  // JDK 7+ : true (s1的引用入池)

String s2 = new StringBuilder("ja").append("va").toString();
System.out.println(s2.intern() == s2);  // false ("java" 早已被 JVM 加载入池)
```

**字符串常量池参数：**
```bash
# 字符串去重（G1）
-XX:+UseStringDeduplication
# 设置字符串常量池桶数（默认60013）
-XX:StringTableSize=100000
```

高并发场景下字符串常量池查询可能成为性能瓶颈：
```bash
# 查看字符串常量池统计
jmap -heap <pid>
```

# 7. 堆参数详解

```bash
# 基础配置
-Xms4g                     # 初始堆 4G
-Xmx4g                     # 最大堆 4G（建议 Xms = Xmx，避免堆扩容收缩）
-Xmn2g                     # 新生代 2G

# 分代比例
-XX:NewRatio=2             # 老年代:新生代 = 2:1（堆 6G 时新生代 2G）
-XX:SurvivorRatio=8        # Eden:Survivor = 8:1 (Eden 80%, S0 10%, S1 10%)

# 晋升阈值
-XX:MaxTenuringThreshold=15   # 默认 15（CMS下为6）
-XX:TargetSurvivorRatio=90    # Survivor 区占用达到 90% 时动态调整晋升年龄

# 大对象阈值
-XX:PretenureSizeThreshold=3145728  # 超过 3MB 的对象直接进入老年代

# 对象年龄记录（PrintTenuringDistribution 输出用）
-XX:+PrintTenuringDistribution
```

# 8. 堆溢出演示

```java
// -Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError
public class HeapOOM {
    public static void main(String[] args) {
        List<byte[]> list = new ArrayList<>();
        while (true) {
            list.add(new byte[1024 * 1024]); // 每次分配1MB
        }
    }
}
// Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
// Dumping heap to java_pid<pid>.hprof ...
```

# 9. JDK 不同版本堆变化

| 版本 | 变化 |
|------|------|
| JDK 7及之前 | 永久代（PermGen），堆 + 永久代 |
| JDK 8 | 移除永久代，引入元空间（Metaspace），字符串常量池移入堆 |
| JDK 9 | 默认 GC 改为 G1（堆的分代物理上不再连续划分） |
| JDK 11 | 废弃 CMS；引入 ZGC（实验性） |
| JDK 15 | ZGC 转正 |
| JDK 21 | 分代 ZGC（实验性），减少停顿 |