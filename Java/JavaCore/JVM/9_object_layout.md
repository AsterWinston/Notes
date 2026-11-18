# 1. 对象布局总览

## 1.1 三部分结构

HotSpot 堆中一个普通 Java 对象的内存布局分三部分：

```text
┌──────────────────────────┐
│     对象头 (Header)       │  ← Mark Word + Klass Pointer
├──────────────────────────┤
│     实例数据 (Instance    │  ← 各字段数据（含父类字段）
│     Data)                │
├──────────────────────────┤
│     对齐填充 (Padding)    │  ← 保证对象大小是 8 字节的倍数
└──────────────────────────┘
```

## 1.2 64 位典型示例（压缩指针开启）

```text
地址增长方向 ──────────────▶

┌──────────────┬───────────┬──────────────────┬─────────┐
│  Mark Word   │ Klass Ptr │   实例数据(字段)   │ Padding │
│    8 B       │   4 B     │    按规则重排       │  补齐    │
└──────────────┴───────────┴──────────────────┴─────────┘
        对象头共 12 B；一个空 Object = 12 → 补到 16 B（最小对象）
```

| 事实 | 推论 |
|------|------|
| 空 `new Object()` 也要 16B | 对象数海量的小对象服务，头部占比是真实成本（JEP 519 的动机，见第 10 节） |
| 头部 + 字段 + padding 是 8 的倍数 | 压缩指针才能"右移 3 位"省 3 bit（见第 6 节） |
| 数组比对象多一个 4B length | 数组布局单独在 7.1 节 |

## 1.3 对齐规则

- 对象起始地址必须是 8 字节倍数（`-XX:ObjectAlignmentInBytes=8`，可改 16，但会改变压缩指针覆盖范围，见 6.3）。

# 2. 对象头：Mark Word

## 2.1 64 位 Mark Word 状态表（JDK 8 ~ JDK 14 经典布局）

每种状态的 64 bit **划分方式不同**，按状态逐行展开（高位 → 低位）：

### 2.1.1 无锁态

| 锁状态 | 位段划分（高 → 低） | 低 2 bit |
|--------|---------------------|----------|
| 无锁 | unused(25) + **hashCode(31)** + unused(1) + **GC age(4)** + 偏向位(0) | **01** |

### 2.1.2 偏向锁态（JDK 15 后取消）

| 锁状态 | 位段划分（高 → 低） | 低 2 bit |
|--------|---------------------|----------|
| 偏向锁 | **线程 ID(54)** + epoch(2) + unused(1) + age(4) + 偏向位(1) | 01 |

### 2.1.3 轻量级锁态

| 锁状态 | 位段划分（高 → 低） | 低 2 bit |
|--------|---------------------|----------|
| 轻量级锁 | **指向栈中 Lock Record 的指针(62)** | 00 |

### 2.1.4 重量级锁态

| 锁状态 | 位段划分（高 → 低） | 低 2 bit |
|--------|---------------------|----------|
| 重量级锁 | **指向 Monitor（重量级锁对象）的指针(62)** | 10 |

### 2.1.5 GC 标记态

| 锁状态 | 位段划分（高 → 低） | 低 2 bit |
|--------|---------------------|----------|
| GC 标记 | 空(62) | 11 |

- 低 2 bit 锁标志：`01` 无锁/偏向、`00` 轻量级、`10` 重量级、`11` GC 标记，JVM 靠它一眼判断对象锁状态。
- 32 位 JVM 的 Mark Word 为 4B，同样的字段思路、位宽更紧凑，此处不展开。
- **版本提醒**：偏向锁 JDK 15 默认关闭并废弃（JEP 374），其后被移除——新版本不再有"偏向锁"行，54 bit 线程 ID 区被回收为备用位。

## 2.2 锁状态视角的字段复用

```text
无锁:     [ hash:31 | age:4 | 0 | 01 ]        哈希与年龄共存
   │ 首线程 CAS 获得偏向
   ▼
偏向:     [ threadId:54 | epoch:2 | age:4 | 1 | 01 ]   记录"属于哪个线程"
   │ 第二个线程来竞争 →  safepoint 撤销偏向
   ▼
轻量级:   [ 62-bit 指针 → 栈上 Lock Record ]  00
                     Lock Record 里存 displaced Mark Word（被顶替的原头）
   │ 竞争激烈 / 自旋失败膨胀
   ▼
重量级:   [ 62-bit 指针 → ObjectMonitor ]    10   (关联 OS mutex)
```

锁升级过程与 `displaced header` 的意义详见 [19_lock_optimization](19_lock_optimization.md)。

## 2.3 hashCode 延迟计算：一连串取舍的源头

- `new` 时 **不**填哈希；第一次调用 `Object.hashCode()`（且未重写）才计算并写进 Mark Word 的 31-bit hash 区。
- 连锁推论：
  1. 哈希一旦写入，Mark Word 前 25 bit 就不能再被挪作他用（无锁态的 unused 只剩 25+1）。
  2. 偏向锁要存 54-bit 线程 ID，与"已存哈希"的对象冲突 → 产生撤销/降级问题——偏向锁最终被移除的原因之一。
  3. 紧凑对象头想把整块头压到 8B，"哈希必须驻留头部"成为主要障碍 → 只能改成截断/重算方案（见第 10 节）。

## 2.4 GC age：4 bit，上限 15

- 年龄字段只有 4 bit → `-XX:MaxTenuringThreshold` 最大 15（CMS 系默认 6）。
- 晋升规则详见 [6_heap](6_heap.md) 2.2.4。

## 2.5 Klass Pointer（类型指针）

- 指向**元数据**：元空间中的 `InstanceKlass`，不是"类的实例"。JVM 靠它回答"这是哪个类的对象"、找方法表、做 instanceof。

```text
对象头里的 4B/8B            元空间 InstanceKlass
┌──────────┐  klass ptr  ┌────────────────────────────┐
│ klass ───┼────────────▶│ _metadata: 类名/修饰符/父类 │
└──────────┘             │ _fields: 字段表+偏移/宽度    │
                          │ _methods/_adapter/_vtable   │ ← 虚方法表（8.2 节动态分派用）
                          │ _constants: 运行时常量池     │
                          │ _method_data: Profiling     │
                          └────────────────────────────┘
```

- 64 位未压缩 = 8B；开启压缩类指针（`-XX:+UseCompressedClassPointers`，依赖压缩 oops）= 4B。
- 经典布局里它**始终独立于 Mark Word**（Mark 8B + Klass 4B = 12B 头）；JEP 374 移除偏向锁回收了 Mark 的位，才让"两者合并成 8B 头"成为可能（第 10 节）。

# 3. 实例数据（Instance Data）

## 3.1 内容范围

真正存字段的地方；**父类字段也排在这里**（子类字段之后？否——父类字段先排）。

## 3.2 字段重排规则：从大到小

### 3.2.1 宽度排序清单

HotSpot 默认按宽度排序以最小化 padding：

```text
1. double / long      (8B)
2. int / float        (4B)
3. short / char       (2B)
4. byte / boolean     (1B)
5. oop 引用            (4B 压缩 / 8B)
```

### 3.2.2 重排前后示例

```java
class Order {
    String id;    // 声明顺序：引用在最后（重排后仍如此）
    int no;       // 
    byte flag;    // 
    long ts;      // 
}
// 存储顺序（重排后）：ts(8) | no(4) | id(ref 4) | flag(1) [+3 padding]
// 若不重排：ref4+int4+byte1+long8 → byte 后 long 前浪费 7B 对齐
```

历史参数 `-XX:FieldsAllocationStyle`（0/1）控制宽字段先排还是后排，现代版本已无人需要动它。

## 3.3 继承链布局

```text
┌─ 头(12B) ─┬─ 父类字段（先）─┬─ 子类字段（后）─┬─ padding ─┐
```

父类字段永远在前 → 子类对象可以按"前缀相同"理解，向上转型时字段偏移解释简单。

# 4. 对齐填充（Padding）

## 4.1 规则与算例

### 4.1.1 对齐规则

HotSpot 要求对象大小是 `ObjectAlignmentInBytes`（默认 8）的倍数。完整算一遍：

### 4.1.2 class A 完整算例

```java
class A { int a; byte b; }
// 对象头(压缩下 Mark8+Klass4) = 12B
// 实例数据 = int(4) + byte(1) = 5B
// 12 + 5 = 17B → 补 7B → 24B（8 的倍数）
```

### 4.1.3 四对象快算表

| 对象 | 头 | 数据 | 合计 | 补后 |
|------|----|------|------|------|
| `new Object()` | 12 | 0 | 12 | **16** |
| `A{int,byte}` | 12 | 5 | 17 | **24** |
| `B{int,int}` | 12 | 8 | 20 | **24** |
| `C{int,int,int}` | 12 | 12 | 24 | **24**（刚好不用补） |

## 4.2 @Contended 防伪共享

对齐 8B 是给"对象大小"的，不是给"CPU 缓存行"的——同一缓存行（64B）里相邻字段被两个核频繁写，会互相 invalidate（伪共享）：

### 4.2.1 伪共享原理图

```text
无 @Contended：一个缓存行(64B)装下 4~8 个 Cell，线程A写Cell0 → 整行失效 → 线程B的Cell1遭殃
┌───────────────────────── 64B 缓存行 ─────────────────────────┐
│ Cell0.value │ Cell1.value │ Cell2.value │ Cell3.value │ ...  │
└───────────────────────────────────────────────────────────────┘

加 @Contended：每个 Cell 独占（至少）64B，前后垫到独立缓存行
┌──64B──┐ ┌──64B──┐ ┌──64B──┐ ┌──64B──┐
│ Cell0  │ │ Cell1  │ │ Cell2  │ │ Cell3  │   ← padding 计入对象大小，费内存换吞吐
└────────┘ └────────┘ └────────┘ └────────┘
```

### 4.2.2 JDK 内部用法与版本可用性

```java
// java.util.concurrent.Striped / LongAdder 的 Cell 正是如此：
@jdk.internal.vm.annotation.Contended
public static class Cell implements Serializable {
    volatile long value;
}
```

| 版本 | 可用性 |
|------|--------|
| JDK 8 | 业务代码使用需 `-XX:-RestrictContended`，注解在 `sun.misc.Contended` |
| JDK 9+ | 模块内自动生效；填充宽度 `-XX:ContendedPaddingWidth`（默认 128B） |

原理链见 [18_jmm](18_jmm.md)（内存屏障与缓存一致性）。

# 5. 实战：JOL 分析对象布局

## 5.1 用法

```java
// 依赖：org.openjdk.jol:jol-core（取当前版本即可）
import org.openjdk.jol.info.ClassLayout;
import org.openjdk.jol.info.GraphLayout;

public class JOLDemo {
    public static void main(String[] args) {
        Object o = new Object();
        System.out.println(ClassLayout.parseInstance(o).toPrintable());
        System.out.println(GraphLayout.parseInstance(args).toFootprint()); // 整棵对象图
    }
}
// Mark Word 十六进制列可以直接对照 2.1.x 逐状态划分读
```

## 5.2 输出逐行解读

### 5.2.1 典型输出

```text
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION               VALUE
       0     4        (object header)           01 00 00 00 (00000001 00000000 ...)
       4     4        (object header)           00 00 00 00 (00000000 00000000 ...)
       8     4        (object header)           e5 01 00 f8 (11111001 10111110 ...)
      12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

### 5.2.2 逐 OFFSET 解读

- OFFSET 0-7：Mark Word 两小节（8B）。最低两位是锁标志——`(00000001 ...)` 最低 2 bit = `01` → 无锁；synchronized 后重打可见 `00`/`10` 结尾。
- OFFSET 8-11：Klass Pointer（4B，压缩类指针开启）。
- OFFSET 12-15：对齐损失。
- `internal` = 对象**内部**字段间的 padding；`external` = 为 8 字节对齐**对象末尾**补的字节。

### 5.2.3 关闭压缩指针后的布局对比

对比关闭压缩指针：

```text
# -XX:-UseCompressedOops
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION               VALUE
       0     4        (object header)           ...
       4     4        (object header)           ...
       8     4        (object header)           ...
      12     4        (object header)           ...
      16     8        (loss due to alignment)
Instance size: 24 bytes
```

关闭后：Mark Word 8B + Klass 8B + 填充 8B = 24B。命令行版：`java -jar jol-cli.jar internals java.lang.Object` 可直接透视 JDK 内部类。紧凑对象头开启后（10.3），以上布局整体前移 4B。

# 6. 压缩指针（Compressed Oops）

## 6.1 动机：64 位的"指针膨胀"税

| 项目 | 不压缩 | 压缩 |
|------|--------|------|
| 引用字段 / klass 指针 | 8B | 4B |
| 同堆能放的对象数 | - | +（缓存密度↑，GC 扫描的引用字节数↓） |
| CPU 成本 | 直接用 | 每次解引用 `<<3 + base`（通常折进寻址模式，近似免费） |

## 6.2 为什么恰好 32G：推导

对象按 8 字节对齐 → 地址低 3 位恒为 0 → 存储时把低 3 位砍掉，右移 3 位保存；32 位无符号数最大表示 `2^32 - 1` 个"8B 槽"：

### 6.2.1 可寻址范围推导

```text
可寻址范围 = 2^32 × 8B = 2^35 B = 32GB   ← 阈值由此而来，不是玄学
```

### 6.2.2 编解码公式

```text
编码:  oop32  =  (addr - base) >> shift
解码:  addr   =  base + (oop32 << shift)      shift 由对齐决定(8B→3)

        64-bit 真实地址:  | base | oop32 (31bit 有效) | 000(丢失的低3位) |
```

## 6.3 堆大小与压缩模式

### 6.3.1 三档模式表与 32G 悬崖

| 堆大小 | 模式 | 说明 |
|--------|------|------|
| < 4G（且低地址） | unscaled / 零基移位均可 | shift 甚至可为 0 |
| 4G ~ 32G | **零基 + shift 3**（主流默认） | 需要堆能映射到低 32G 地址段，否则自动降级 |
| ≥ 32G | 压缩关闭 | 所有引用回到 8B → "32G 悬崖"：31G 堆往往比 33G 堆能装更多对象 |

### 6.3.2 参数与验证命令

```bash
-XX:+UseCompressedOops              # 64 位 + 堆≤32G 时默认开
-XX:+UseCompressedClassPointers     # klass 4B（依赖上一项）
-XX:ObjectAlignmentInBytes=16       # 改 16 可把阈值推到 64G，但全堆 padding 变多
# 验证：jcmd <pid> VM.flags | grep -i compressed   或看 -Xlog:gc+init 输出
```

## 6.4 压缩类指针与类区

- Klass 的 32 bit 是**压缩类空间内的索引**：元空间划出专用 `-XX:CompressedClassSpaceSize`（默认 1G）只放 InstanceKlass。
- 类太多/太大 → `OutOfMemoryError: Compressed class space`（与 `Metaspace` 报错分家，见 [7_method_area](7_method_area.md)）。

# 7. 数组与字符串的特殊布局

## 7.1 数组：头里多一个 length

普通对象靠元数据就知道自身大小；数组长度运行期才知道 → 头后插 4B `length`，元素紧随、连续存放。

```text
int[10]：
┌────────────┬───────────┬──────────┬───────────────────────┐
│ Mark Word  │ Klass Ptr │ length 4B│ 10 × int (40B)         │
│    8B      │   4B      │          │                        │
└────────────┴───────────┴──────────┴───────────────────────┘
  12 + 4 + 40 = 56 B   → 已是 8 的倍数，无需 padding ✓

byte[10]：12 + 4 + 10 = 26 → 补 6 → 32 B
long[10]：12 + 4 + 80 = 96 B（元素起点 offset16，天然 8 对齐 ✓）
String[10]：12 + 4 + 10×4 = 56 B（压缩指针下引用 4B）
```

## 7.2 JDK 9 紧凑字符串（JEP 254）

`String` 从 `char[]` 改为 `byte[] + coder`，Latin-1 文本一个字符只用 1 字节：

### 7.2.1 String 新布局与 coder

```java
// JDK 9+ String 的实例数据（重排后）：
int hash;          // 4B
byte[] value;      // 引用 4B
byte coder;        // 1B：0=LATIN1, 1=UTF16（UTF16 按大小端序存）
// 12头 + 9 = 21 → String 壳对象 24B
```

### 7.2.2 三组内存对比

```text
"abc"（全 ASCII）：  String(24B) + byte[3](12+4+3→24B)  共 48B
JDK 8：            String(24B) + char[3](12+4+6→24B)   共 48B   ← 短串持平
100 字符 ASCII：    byte[100]=116→120B  vs  char[100]=212→216B  ← 长串省近一半 ✓
纯中文：coder=UTF16，byte[] 2n，与旧版持平（不再更差）
```

- 关闭：`-XX:-CompactStrings`（几乎没人需要）。
- 代价：所有 String 内部操作多了 coder 分支；`hashCode` 公式对 UTF16/LATIN1 分别实现。

# 8. 对象的访问定位

规范只规定"栈里的引用指向对象"，没规定怎么指。HotSpot 选**直接指针**。

## 8.1 两种访问方式对比

### 8.1.1 方式一：句柄访问（间接指针）

```text
栈 (reference) → ┌──────────────┐
                 │   句柄池      │
                 │ ┌──────────┐ │    ┌──────────┐
                 │ │ 对象指针  │─┼───▶│ 实例数据  │
                 │ │ 类型指针  │─┼─┐  └──────────┘
                 │ └──────────┘ │ │  ┌──────────┐
                 └──────────────┘ └─▶│ 类型数据  │
                                     └──────────┘
```

| 优点 | 缺点 |
|------|------|
| GC 移动对象只改句柄，reference 不变 → 栈/其他引用零成本 | 每次访问多一跳（句柄），缓存友好性差 |

### 8.1.2 方式二：直接指针（HotSpot）

```text
栈 (reference) ──▶ ┌──────────────────────────┐
                  │     对象实例               │
                  │ ┌──────────────────────┐  │
                  │ │  Klass Pointer       │──┼──▶ 元空间中的类型数据
                  │ ├──────────────────────┤  │
                  │ │  实例数据             │  │
                  │ └──────────────────────┘  │
                  └──────────────────────────┘
```

| 优点 | 缺点 |
|------|------|
| 一跳到位，访问频繁的场景显著更快 | GC 移动对象 = 改所有引用本身 → 必须靠**屏障**保证正确性 |

## 8.2 直接指针 + 屏障的组合

- 复制/整理型 GC 搬对象 → 更新引用，跨代写引用要记录 → **写屏障/卡表**（见 [11_gc_algorithm](11_gc_algorithm.md)）。
- ZGC 更进一步：**着色指针 + load barrier**，移动对象时读引用现场纠正（见 [13_low_latency_gc](13_low_latency_gc.md)）。
- 结论：直接指针的"移动难"已被屏障工程化解决，句柄方案只在教学里活着。

# 9. 对象创建流程

## 9.1 五步流程

```text
1. 类加载检查     new 指令的符号引用能否解析？未加载 → 先走加载/链接/初始化（见 3）
2. 分配内存       连续(复制/整理型) → 指针碰撞；不连续(清除型 CMS) → 空闲列表
3. 并发安全       共享 Eden CAS+重试；主流路径是线程私有 TLAB（见 6/15）
4. 初始化零值     除对象头外的字段全部置 0/null/false（"int 默认 0"的实现层来源）
5. 设置对象头     Mark Word（无锁态 01）、Klass Pointer；哈希/age 此时**不填**
   ────────────▶ 之后才轮到 <init>()：开发者字段赋值真正生效
```

## 9.2 javap 看 `new Object()` 指令序列

### 9.2.1 指令序列

```text
 0: new           #2    // class java/lang/Object  → 走完上面 5 步，栈中压入未初始化引用
 3: dup                                            → 复制引用：一份给构造函数，一份留给调用方
 4: invokespecial #3    // Method "<init>":()V     → 弹出引用+参数执行构造（引用被消耗）
 7: astore_1                                       → 把留下的引用存入局部变量 slot1
```

### 9.2.2 为什么需要 dup

`dup` 的存在是栈式字节码的直接证据：`invokespecial` 会消耗接收者引用，若不先复制，对象地址就丢了。

## 9.3 逃逸分析可能截胡

分配前先问 JIT：对象是否被外部方法/线程引用？不逃逸 → 栈上分配甚至**标量替换**（拆成寄存器/局部变量，压根没有堆对象）。`-XX:+DoEscapeAnalysis` 早就是默认；观测见 [15_memory_allocation](15_memory_allocation.md)、[17_jit_optimization](17_jit_optimization.md)。

# 10. 紧凑对象头（Compact Object Headers）

## 10.1 时间线与开关

```text
JEP 450  JDK 21 实验     → 后续版本持续孵化打磨
JEP 519  JDK 25 转正并默认开启
```

```bash
# JDK 21~24 手动开启（实验版会打 warning）：
-XX:+UnlockExperimentalVMOptions -XX:+UseCompactObjectHeaders   # JDK 21 时代
-XX:+UseCompactObjectHeaders
# JDK 25 默认开启；回退：-XX:-UseCompactObjectHeaders
```

## 10.2 布局：96 bit → 64 bit

### 10.2.1 新旧头部对比图

```text
经典头(12B)： ┌─ Mark Word 64b ─┬─ Klass 32b ─┐ + 4B padding（凑 16B 空对象）
紧凑头(8B)：  ┌─ Mark 32b ─┬─ Klass Word 32b ─┐   → 空对象 16B → 8B，不再需要 padding
```

### 10.2.2 新 Mark Word 与 Klass Word 内部

- 新 **Mark Word** 挤进了旧头的全部分量：锁状态位、GC age、**identity hash 的截断值**（哈希太大时改为可重算/借用侧表，语义上限变化）、轻量级锁不再放 62-bit 指针——**displaced Mark Word 挪进线程栈上的专用 displaced 栈，头里只存 32b 索引**（JEP 450 的核心腾挪）。
- 新 **Klass Word**：不再是裸指针，而是"元数据 ID + 压缩索引"方案——按讨论口径约 16 bit 元数据 ID、其余位作压缩索引（精确位域以对应 JEP 图为准）。
- 前提依赖：压缩类指针必须开（索引语义建立在类区可压缩上）；这也解释了 JEP 374 先清掉偏向锁占位。

## 10.3 收益

| 收益 | 量化口径 |
|------|----------|
| 小对象直接瘦身 | 头部 12B→8B；叠加 padding 消除，典型小对象实例省 4~8B，JEP 基准里小对象密集类合计省 16~56B 量级 |
| 同堆装更多对象 | 小对象为主的服务实测堆占用降 **10%~20%** |
| 缓存友好 | 对象图更紧凑 → 同样大小的缓存行装更多有效数据 → GC 扫描与遍历时 locality 提升 |

## 10.4 代价与限制（谨慎项）

- identity hashCode 截断存储 → 冲突概率、依赖哈希分布的容器行为在超大应用上需重估（JEP 给出重算策略）。
- Klass Word 索引位宽 → **可加载类总数有硬上限**（数十万量级；依赖树超大的单体应用要留意版本说明的确切上限）。
- 与老工具的兼容：JOL 输出的头偏移与宽度改变；直接按 96-bit 头假设的 unsafe 布局代码/序列化黑魔法要重测。

## 10.5 一图总结版本演进

```text
JDK 8~14   Mark64+Klass32/64 (偏向锁仍在)          16B 起步
JDK 15     偏向锁默认关闭 (JEP 374) → Mark 位腾出
JDK 21     紧凑对象头实验 (JEP 450)：头 → 8B
JDK 25     紧凑对象头转正默认 (JEP 519)
```

对堆密度、TLAB 大小、GC 扫描量的连锁影响，回看 [6_heap](6_heap.md) 9.4。
