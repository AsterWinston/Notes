# 1. 知识点
- **JVM体系结构**：类加载器子系统 → 运行时数据区（方法区/堆/栈/PC寄存器/本地方法栈）→ 执行引擎（解释器/JIT编译器/GC）→ 本地方法接口(JNI)
- **程序计数器(PC Register)**：线程私有，记录当前线程正在执行的字节码指令地址（Native方法为undefined），无OOM
- **Java虚拟机栈(VM Stack)**：线程私有，每个方法执行时创建栈帧（Stack Frame），存储局部变量表、操作数栈、动态链接、方法返回地址；深度超出-Xss限制报StackOverflowError；栈可动态扩展，扩展失败报OOM
- **本地方法栈**：线程私有，与虚拟机栈类似，但服务Native方法。HotSpot将两者合二为一
- **Java堆(Heap)**：线程共享，存放对象实例和数组，GC主要区域。HotSpot分为新生代（Eden+S0+S1默认8:1:1，即新生代90%：两个Survivor各5%+5%=10%）和老年代（占堆的2/3）
- **方法区→元空间**：线程共享，存储类型信息、常量、静态变量、JIT编译后的代码缓存。JDK7及之前叫永久代(PermGen)，JDK8+移除永久代改用元空间(Metaspace)，由直接内存实现，默认无上限受系统内存限制(-XX:MaxMetaspaceSize限制)
- **运行时常量池**：class文件常量池的运行时表示，存放编译期生成的字面量和符号引用；JDK7移到堆中，String.intern()的字符串在堆的字符串常量池中
- **直接内存**：不属于JVM内存区域，是OS本地内存。NIO的ByteBuffer.allocateDirect()分配，通过Native方法操作，避免Java堆和Native堆数据拷贝。不受-Xmx限制但受系统总内存限制，OOM也可能由直接内存不够引起
- **栈帧结构**：局部变量表（基本类型/对象引用/returnAddress，以槽Slot为单位，long/double占2槽）、操作数栈（LIFO，方法执行中的计算中间结果）、动态链接（指向运行时常量池中该方法的符号引用，支持运行时绑定）、方法返回地址（正常返回或异常返回）
- **HotSpot堆结构**：新生代(YoungGen) = 1个Eden + 2个Survivor（S0/S1或From/To），默认Eden:S0:S1 = 8:1:1；老年代(Tenured/Old Gen)；字符串常量池在JDK7+移到堆中
- **对象创建过程**：①当JVM遇到new指令 → 检查类是否加载（未加载则先类加载）→ ②从堆中分配内存（指针碰撞:堆规整时，Serial/ParNew；空闲列表:堆不规整，CMS）→ ③CAS或TLAB（本地线程分配缓冲）保证分配原子性 → ④初始化零值（保证不赋初值也能用）→ ⑤设置对象头（MarkWord+类型指针）→ ⑥执行构造器<init>方法，初始化字段
- **对象内存布局**：MarkWord(8/12字节：hashCode/GC年龄/锁状态/偏向信息)、类型指针(Klass Pointer，指向方法区的类元数据，默认-XX:+UseCompressedClassPointers压缩为4字节)、实例数据(字段，相同宽度字段尽量排在一起对齐)、对齐填充(确保对象大小是8字节的整数倍)
- **对象访问定位**：句柄池（引用→句柄池→实例数据+类型数据，GC时只需修改句柄池中指针，但多一次间接访问）；直接指针（引用→实例数据→对象头中的类型指针→类型数据，HotSpot使用，访问快一次）。HotSpot选择直接指针（访问快），通过压缩指针减少内存占用
- **判断对象存活**：引用计数法（简单但无法解决循环引用）；可达性分析算法（Reachability Analysis）：以GC Roots为起点，从根节点向下搜索，不可达的对象判定为可回收
- **GC Roots有哪些**：虚拟机栈中引用的对象、方法区静态变量引用的对象、方法区常量引用的对象、本地方法栈JNI引用的对象、Java虚拟机内部的引用（Class对象/异常对象/类加载器）、同步锁(synchronized)持有的对象、JVM内部驻留字符串对象
- **四大引用**：强引用(new，永远不回收)、软引用(SoftReference，内存不足时第二次OOM前回收，适合缓存)、弱引用(WeakReference，下次GC就回收，适合ThreadLocal/WeakHashMap)、虚引用(PhantomReference，最弱，get()总返回null，唯一作用是对象被回收时收到系统通知，用于NIO直接内存清理)
- **ReferenceQueue**：引用队列，配合软/弱/虚引用使用，当引用的对象被GC回收后，引用对象本身被放入队列，可做后置清理工作
- **GC算法**：
  - 标记-清除：标记所有可达对象，清除未标记对象。简单，但产生内存碎片，效率一般
  - 标记-整理：标记存活对象，将存活对象向一端移动，清理边界外内存。无碎片，但移动对象耗时（STW）
  - 复制算法：将内存分为两块，每次使用一块，存活对象复制到另一块，清除第一块。无碎片，但浪费一半空间；HotSpot新生代使用(一个Eden+两个Survivor，仅10%浪费)
  - 分代收集算法：新生代用复制(对象存活率低)，老年代用标记-整理或标记-清除(对象存活率高)
- **Serial收集器**：单线程，新生代复制算法，STW。Client模式默认。简单高效（无线程交互开销）。(新生代Serial+老年代Serial Old)
- **ParNew**：Serial的多线程版，新生代复制+STW，配合CMS（ParNew+CMS组合）。(新生代ParNew+老年代CMS)
- **Parallel Scavenge**：新生代多线程，复制算法，侧重吞吐量(运行时间/(运行时间+GC时间))，适合后台运算无交互场景。(新生代Parallel Scavenge+老年代Parallel Old)
- **Parallel Old**：Parallel Scavenge的老年代版，多线程标记-整理算法
- **CMS(Concurrent Mark Sweep)**：目标最短回收停顿时间（低延迟，如Web交互应用）。老年代标记-清除：初始标记(STW)→并发标记→重新标记(STW)→并发清除。并发收集、低停顿。缺点：CPU敏感（占用CPU）、浮动垃圾（并发标记时新产生的垃圾）、Concurrent Mode Failure（老年代空间不足时退化Serial Old单线程）、碎片（标记-清除固有）
- **G1(Garbage First)**：JDK9+默认，面向服务端，将堆分为多个等大Region(1~32MB，约2048个)，取消了新生代和老年代的物理隔离（逻辑上仍有分代），优先回收垃圾最多的Region(Garbage First)，停顿时间可预测(-XX:MaxGCPauseMillis)。Remembered Set(RSet)记录跨Region引用，SATB(Snapshot-At-The-Beginning)保证并发标记准确性。整体标记-整理+局部复制
- **G1 vs CMS对比**：
  - G1整体标记-整理(局部复制)，CMS标记-清除→G1无碎片
  - G1可预测停顿，CMS不可控
  - G1适合大堆(6GB+)，CMS适合中小堆
  - G1的Each Region单独回收=对停顿更可控
  - G1写屏障更复杂，CPU占用稍高
  - G1的每个Region都有RememberedSet，占堆5%-20%额外内存
- **ZGC**：JDK11实验，JDK15生产，超低延迟<1ms。着色指针(Colored Pointer)+读屏障(Load Barrier)+并发处理。动态Region（分小/中/大）。与堆大小无关的停顿时间。JDK21支持分代ZGC
- **Shenandoah**：类似ZGC，由Red Hat开发，JDK12+。使用Brooks Pointer（转发指针）而非着色指针实现并发整理，停顿时间也与堆大小无关
- **三色标记法**：黑(Black)自身及其引用的对象都已处理、灰(Grey)自身已处理但引用的对象未处理完、白(White)未处理（结束回收）。并发标记问题：漏标(本来是白色对象被已扫描过的黑色对象的新引用指向，同时其原灰色引用断开) → CMS用增量更新Incremental Update(黑色→白色时，重新标记黑色)，G1用原始快照SATB(灰色→白色引用断开时，记录下白色)。
- **类加载7阶段**：加载(Loading)→验证(Verification)→准备(Preparation)→解析(Resolution)→初始化(Initialization)→使用(Using)→卸载(Unloading)
- **加载阶段**：①通过类的全限定名获取二进制字节流 ②将字节流静态结构转为方法区运行时数据结构 ③在内存中生成Class对象作为访问入口
- **验证阶段（文件格式验证）**：CAFEBABE魔数、主次版本号检查、常量池常量类型检查、元数据验证（是否有父类、final类未被继承）、字节码验证（跳转指令不越界、类型转换合法）、符号引用验证
- **准备阶段**：为静态变量(static)在方法区分配内存并赋默认初始值（注意只是分配内存赋零值，赋程序指定值在初始化阶段）
- **解析阶段**：将常量池中的符号引用替换为直接引用（类/字段/方法/接口方法解析）
- **初始化阶段**：执行类的<clinit>()方法（由编译器自动收集类中静态变量的赋值动作和static代码块合并产生）。<clinit>()保证父类先执行，接口不一定先执行父接口。同一ClassLoader下，<clinit>()只会执行一次（线程安全/加锁）
- **双亲委派模型**：Bootstrap ClassLoader(启动类加载器，加载jre/lib) → Extension ClassLoader(扩展类加载器，加载jre/lib/ext，JDK9+被Platform ClassLoader替代) → Application ClassLoader(系统类加载器，加载classpath) → Custom ClassLoader(自定义)
- **双亲委派工作原理**：loadClass()检查是否已加载 → 未加载则先委托父加载器加载 → 父加载器无法加载时才自己尝试加载。优点：避免类重复加载、防止核心API被篡改（如自定义java.lang.String无法加载）
- **双亲委派打破场景**：
  - Tomcat：每个WebApp用独立WebappClassLoader，隔离不同应用的同名类
  - SPI/JDBC：ServiceLoader使用线程上下文类加载器（Thread.setContextClassLoader）打破双亲委派，加载SPI实现类
  - OSGI：网状委托模型，更复杂的模块化类加载
  - 热部署/热替换：自定义ClassLoader反复加载同一个class
- **Class文件结构**：魔数CAFEBABE(4B)、主次版本号(4B)、常量池(变长)、访问标志(public/abstract/final等,2B)、类索引(2B)父索引(2B)接口索引集(变长)、字段表集合、方法表集合、属性表集合
- **JVM常用参数**：
  - 堆：-Xms初始大小 -Xmx最大大小 -Xmn新生代大小
  - 元空间：-XX:MetaspaceSize初始 -XX:MaxMetaspaceSize最大
  - 栈：-Xss每个线程栈大小（默认1M）
  - GC日志：-Xlog:gc*（JDK9+）；-XX:+PrintGCDetails -XX:+PrintGCDateStamps（JDK8）
  - GC选择：-XX:+UseG1GC / -XX:+UseZGC
  - HeapDump：-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=
  - G1暂停目标：-XX:MaxGCPauseMillis=200
- **性能监控工具**：jps(查看Java进程PID)、jstat(监视JVM统计信息: -gc -gcutil)、jstack(线程堆栈，排查死锁/CPU高)、jmap(堆dump: -dump:format=b,file=xxx.hprof)、jhat(分析HeapDump，已过时)、jvisualvm(图形化: 堆/线程/CPU)、JMC飞行记录、Arthas阿里(诊断: thread/dashboard/watch/trace)
- **Full GC频繁排查**：jstat -gcutil观察伊甸/老年代/元空间使用率 → 如果老年代或元空间持续增长 → jmap -histo查看对象分布(哪些类实例过多) → MAT分析HeapDump找大对象/泄漏 → GC日志分析频率和时长
- **CPU 100%排查**：top找到高CPU进程PID → top -Hp PID看线程 → 或ps -mp PID -o THREAD,tid,time → 找出高CPU线程TID → printf "%x\n" TID转十六进制 → jstack PID | grep -A 30 十六进制 → 定位到代码行
- **OOM排查**：开启-XX:+HeapDumpOnOutOfMemoryError → MAT/JProfiler/VisualVM分析dump文件 → 查看支配树(Dominator Tree)找到大对象 → 跟踪引用链找到泄漏根源 → 结合GC日志判断是否正常业务增长
- **JIT编译器**：C1 Client Compiler（编译快但优化少，启动快，C1编译阈值低）、C2 Server Compiler（编译慢但优化激进，运行快，C2编译阈值高）、分层编译(Tiered Compilation)：C1和C2协作，C1做简单优化快速编译，C2在后台做深度优化。JDK8默认开启分层编译。-Xint纯解释、-Xcomp纯编译、-Xmixed混合（默认）
- **用户态和内核态**: 用户态进程权限受限，无法直接访问硬件/系统资源；内核态拥有完全访问权限。程序需要特权操作(磁盘I/O、网络)时通过**系统调用**触发用户态→内核态切换。synchronized偏向锁/轻量级锁在用户态操作避免切换，重量级锁涉及内核态切换
- **Arthas常用命令**: `dashboard` 实时监控面板(CPU/内存/线程/GC); `thread -n 3` 查看CPU占用Top3线程; `thread -b` 检测死锁; `trace ClassName methodName` 追踪方法调用路径和耗时; `jad` 反编译; `watch` 监控方法入参/返回值; `heapdump` 导出堆dump

# 2. 面试题

## 2.1 JVM内存区域有哪些？各自作用？
**线程私有区域**：
- 程序计数器：当前线程执行的字节码行号指示器，分支/循环/跳转/异常处理/线程恢复都依赖它。唯一无OOM的区域
- Java虚拟机栈：每个方法执行创建栈帧，存储局部变量表、操作数栈、动态链接、方法出口。StackOverflowError(-Xss太小或深度递归)或OOM(动态扩展失败)
- 本地方法栈：类似虚拟机栈，但服务于native方法。HotSpot合二为一

**线程共享区域**：
- 堆：存放对象实例和数组，GC核心区域。新生代(Eden+S0+S1)+老年代。Xmx/Xms控制大小
- 方法区(JDK8元空间)：存储类信息、常量、静态变量、JIT编译后的代码。JDK7及以前是永久代(PermGen, 在堆中，-XX:MaxPermSize)，JDK8+移除改用元空间(本地内存，-XX:MaxMetaspaceSize)。字符串常量池在JDK7移到堆中
- 运行时常量池：class文件常量池的运行时表示。动态性：运行时也可添加（String.intern()）
- 直接内存：NIO的DirectByteBuffer，不受Xmx限制但受物理内存限制

## 2.2 对象创建过程是怎样的？（从new到对象分配）
1. **类加载检查**：遇到new指令，检查类是否加载，未加载则执行类加载
2. **分配内存**：根据堆是否规整选择分配方式：
   - 指针碰撞：内存规整（Serial/ParNew），空闲内存和已用内存间有指针分隔
   - 空闲列表：内存不规整（CMS），JVM维护空闲块列表
3. **并发分配处理**：CAS+失败重试 或 TLAB（Thread Local Allocation Buffer，每个线程预先分配一小块堆内存）
4. **初始化零值**：将分配到的内存空间初始化为零值（不包括对象头），保证不赋值即可使用
5. **设置对象头**：设置MarkWord（hashCode、GC分代年龄、锁状态标志等）和类型指针（指向方法区类元数据）
6. **执行init方法**：调用构造器<init>()方法，按程序员设定赋值

## 2.3 对象的堆内存布局是怎样的？
一个对象在堆内存中分为三部分：
1. **MarkWord(标记字段/对象头)**：32位JVM占4字节，64位占8字节。存储：hashCode(25bit)、GC分代年龄(4bit, 最高15)、偏向锁标志(1bit)、锁标志(2bit)、偏向线程ID(23bit=54-25-2-2)
2. **类型指针(Klass Pointer)**：指向方法区中类元数据。32位4字节，64位默认压缩为4字节(-XX:+UseCompressedClassPointers)
3. **实例数据**：父类字段在前，子类字段在后；相同宽度的字段尽量分配在一起（long/double、int/float、short/char、byte/boolean、引用）；子类较窄的变量可能插入父类间隙
4. **对齐填充**：HotSpot要求对象起始地址必须是8字节的整数倍，内存模型类似C结构体对齐规则。不够时填充补齐

## 2.4 判断对象是否存活的方法有哪些？GC Roots有哪些？
**引用计数法**：每个对象维护一个引用计数器，被引用时+1，引用失效时-1，为0时回收。简单但无法解决循环引用（A引用B，B引用A，但外部无引用）。Java未采用。

**可达性分析算法**：从一组GC Roots出发，通过引用链可达的对象是存活的，不可达的是可回收的。

**GC Roots包括**：
- 虚拟机栈（栈帧中的局部变量表）引用的对象（当前正在执行的方法中的参数/局部变量）
- 方法区中静态变量引用的对象
- 方法区中常量引用的对象
- 本地方法栈中JNI引用的对象
- JVM内部引用的对象（Class对象、类加载器、异常对象）
- 被同步锁(synchronized)持有的对象
- 反映Java虚拟机内部情况的JMXBean、JVMTI中注册的回调等

## 2.5 强引用、软引用、弱引用、虚引用分别是什么？
| 引用类型 | 回收时机 | 用途 | 获取对象 |
|----------|----------|------|----------|
| 强引用(Strong) | 永不回收(除非没有GC Roots) | 普通对象引用 new | 直接 |
| 软引用(SoftReference) | 内存不足时，第二次OOM前回收 | 内存敏感的缓存 | softRef.get() |
| 弱引用(WeakReference) | 下一次GC必定回收 | WeakHashMap, ThreadLocal | weakRef.get() |
| 虚引用(PhantomReference) | 随时可能(仅通知) | NIO直接内存清理 Cleaner | 总返回null |

软引用和弱引用通常配合ReferenceQueue使用。对象被回收后，其软/弱/虚引用对象会被加入关联的ReferenceQueue，应用程序可以在队列中拿到引用对象做后置清理。

弱引用在ThreadLocal中的应用：ThreadLocalMap的Entry key是WeakReference<ThreadLocal>，使得ThreadLocal外部无强引用时可以正常GC。

## 2.6 GC算法有哪些？分代收集是什么意思？
**标记-清除（Mark-Sweep）**：标记所有可达对象 → 清除未标记对象。优点：简单。缺点：碎片（无法分配大对象+频繁GC）、效率一般（两次遍历）

**标记-复制（Copying）**：内存分两半，用一半，存活对象复制到另一半，清空当前区域。优点：无碎片、效率高。缺点：浪费50%空间（HotSpot新生代E:S0:S1=8:1:1仅浪费10%，因为新生代对象98%朝生夕灭）

**标记-整理（Mark-Compact）**：标记存活对象 → 向一端移动 → 清理边界外。优点：无碎片、无需额外空间。缺点：移动对象增加STW时间

**分代收集**：根据对象存活周期将堆分为新生代和老年代，对不同区域采用最适合的算法——新生代存活率低用复制，老年代存活率高用标记-清除或标记-整理。

## 2.7 CMS收集器的工作原理和缺点？
CMS(Concurrent Mark Sweep)老年代收集器，获取最短回收停顿时间为目标。

**工作流程（4步）**：
1. **初始标记(STW)**：标记GC Roots直接关联的对象（仅一层），停顿短
2. **并发标记(Concurrent)**：从GC Roots出发遍历对象图（与用户线程并发，最耗时）
3. **重新标记(STW)**：修正并发标记期间因用户线程运行而变动的标记（使用增量更新，重新标记灰色→黑色变化的黑色节点），停顿比初始标记长但远短于并发标记
4. **并发清除(Concurrent)**：清除不可达对象（与用户线程并发）

**缺点**：
1. CPU敏感：并发阶段占用CPU线程（默认(re_cpu+3)/4个回收线程）
2. 浮动垃圾(Floating Garbage)：并发标记/清除阶段用户线程继续产生新垃圾（标记已完成），只能下次GC回收
3. Concurrent Mode Failure：并发清理时老年代不够放晋升的对象，CMS并发失败退化到Serial Old单线程收集（停顿时间长）
4. 碎片问题：标记-清除固有缺点，大量碎片可能导致提前触发Full GC，使用-XX:+UseCMSCompactAtFullCollection强制整理（STW）

## 2.8 G1收集器的核心特点和原理？
G1(Garbage First)是面向服务端应用的收集器，JDK9+默认。

**核心特点**：
1. **Region分区**：堆分为2000+个等大Region(1~32MB)，取消新生代/老年代物理隔离，逻辑上仍有Eden/Survivor/Old/Humongous(大对象，Region一半以上为巨对象，单独存储)
2. **可预测停顿**：维护每个Region的回收价值（垃圾大小+回收耗时），用户设置-XX:MaxGCPauseMillis(默认200ms)，G1在该时间内尽量回收最有价值的Region
3. **Remembered Set(RSet)**：每个Region都记录其他Region对本Region内对象的引用（解决跨Region GC Roots问题），占5~20%堆空间
4. **SATB(Snapshot-At-The-Beginning)**：在并发标记开始时建立一个对象图快照，标记是"逻辑上在该时刻存活"的对象，解决漏标（通过写屏障将引用变化前的旧值记录到SATB队列）

**工作流程**：
- 年轻GC(Eden满触发)：Eden复制到Survivor，晋升到Old
- 混合GC(标记后)：年轻GC+回收部分老年代Region（回收价值高的Old Region）
- Full GC：类似Serial Old，G1避免出现Full GC

## 2.9 类加载过程有哪几个阶段？双亲委派机制是什么？
**7个阶段**：
1. **加载**：通过全限定名获取二进制字节流 → 转为方法区运行时数据结构 → 生成Class对象
2. **验证**：文件格式(CAFEBABE)、元数据、字节码、符号引用验证
3. **准备**：为static变量在方法区分配内存并赋零值（final static基本类型/字符串常量直接赋常量值）
4. **解析**：符号引用→直接引用
5. **初始化**：执行<clinit>()，static变量赋值+static代码块
6. 使用
7. 卸载

**双亲委派**：类加载器收到加载请求后，先委托父加载器加载（不自己加载），父加载器在缓存中找不到且加载范围不包含该类时才交给子加载器加载。层级：Bootstrap → Extension(Platform)→ Application → Custom。

**优点**：避免类重复加载；保护核心API不被篡改（自定义的java.lang.String无法被加载，因为优先由Bootstrap加载，加载完返回的是核心库的String）

## 2.10 什么情况下会打破双亲委派？SPI是怎么打破的？
**打破场景**：
1. **SPI机制**：ServiceLoader加载META-INF/services/下的实现类。核心接口（如JDBC的Driver）在Bootstrap ClassLoader，但其实现类（如MySQL驱动）在classpath。Bootstrap无法加载classpath下的类 → 使用**线程上下文类加载器(Thread.setContextClassLoader)**，默认是AppClassLoader，通过它加载实现类
2. **Tomcat类加载**：每个WebApp有独立的WebappClassLoader，打破双亲委派实现应用隔离
3. **热部署/模块化**：OSGI/Spring等自定义类加载器，每个模块独立ClassLoader

**JDBC SPI打破双亲委派具体流程**：
1. DriverManager（在rt.jar，Bootstrap加载）初始化时通过ServiceLoader.load(Driver.class)查找驱动
2. ServiceLoader内部取Thread.currentThread().getContextClassLoader()（线程上下文类加载器，默认AppClassLoader）
3. 用AppClassLoader加载META-INF/services/java.sql.Driver文件中列出的实现类
4. 加载到MySQL/PostgreSQL的Driver实现，注册到DriverManager

## 2.11 Full GC频繁如何排查？
1. **jstat -gcutil PID 1000** 持续观察各区域使用率、GC次数和耗时
   - 重点关注：老年代使用率是否反复飙升又下降（正常GC）还是持续上升（泄漏/内存不足）
   - 元空间使用率是否接近MaxMetaspaceSize上限
2. **jmap -histo PID** 查看类实例分布，找出数量异常多的类
3. **jmap -dump:format=b,file=heap.hprof PID** 导出堆dump
4. **MAT/Eclipse Memory Analyzer分析dump**：
   - 打开Dominator Tree（支配树），看哪些对象占用比例最大
   - Leak Suspects Report自动分析可疑泄漏点
   - 查看Histogram找出实例数异常的类
   - 用OQL查询分析大对象的引用链
5. **分析GC日志**：观察每种GC的频率、耗时、回收量 → 判断是老年代空间不够、元空间不够、还是有大对象频繁分配
6. **代码层面**：检查集合持续add不remove、连接未释放、缓存无限增长、ThreadLocal未清理等常见泄漏点

## 2.12 CPU 100%如何排查？
1. **top命令**：找到CPU使用率高的Java进程PID
2. **top -Hp PID**：查看该进程下各线程CPU使用率，找到异常高的线程PID（TID）
3. **printf "%x\n" TID**：将十进制TID转十六进制nid
4. **jstack PID > jstack.log**：导出线程堆栈
5. **grep 十六进制nid jstack.log -A 30**：定位到具体线程
6. **分析代码**：
   - 死循环：while(true)无wait、无sleep
   - GC频繁：jstat -gcutil查看Full GC（CMS并发标记时CPU高）
   - 死锁或大量线程竞争：jstack搜"BLOCKED"/"deadlock"
7. **Arthas辅助**：`dashboard`查看实时面板、`thread -n 3`查看CPU最高的3个线程、`thread -b`检测死锁

## 2.13 常见的JVM内存参数有什么区别？
| 参数 | 作用 | 示例 |
|------|------|------|
| -Xms | 初始堆大小 | -Xms2g |
| -Xmx | 最大堆大小 | -Xmx4g（一般Xms=Xmx避免扩容） |
| -Xmn | 新生代大小 | -Xmn1g |
| -Xss | 线程栈大小 | -Xss512k（默认1m） |
| -XX:MetaspaceSize | 元空间初始 | 默认约21M |
| -XX:MaxMetaspaceSize | 元空间最大 | -XX:MaxMetaspaceSize=256m |
| -XX:NewRatio | 老/新比例 | -XX:NewRatio=2（Old:Young=2:1） |
| -XX:SurvivorRatio | Eden/Survivor比例 | -XX:SurvivorRatio=8（Eden:S=8:1） |
| -XX:MaxDirectMemorySize | 直接内存最大 | -XX:MaxDirectMemorySize=512m（默认==Xmx） |
| -XX:+HeapDumpOnOutOfMemoryError | OOM时dump | 必加 |
| -XX:HeapDumpPath | dump文件路径 | -XX:HeapDumpPath=/tmp/ |
| -Xlog:gc* | GC日志(JDK11+) | -Xlog:gc*:file=gc.log |
| -XX:MaxGCPauseMillis | G1暂停目标 | -XX:MaxGCPauseMillis=200 |

建议生产标配：`-Xms与Xmx相同 -XX:+HeapDumpOnOutOfMemoryError -XX:+UseG1GC -Xlog:gc*:file=gc.log`

## 2.14 什么是三色标记法？CMS和G1如何解决漏标？
**三色标记法**是并发标记的核心算法，将对象分为三种颜色：
- **白色**：尚未被标记器访问。标记结束后白色对象=不可达=可回收
- **灰色**：对象自身被访问（标记），但其引用对象尚未全部访问
- **黑色**：对象自身及其全部引用都已访问

**并发标记的漏标问题**：满足以下两个条件会导致原本可达的对象被漏标（Condition: ①黑色对象新增了指向白色对象的引用 ②灰色对象到该白色对象的引用被删除）

**CMS解决（增量更新Incremental Update）**：当黑色对象新增指向白色对象的引用时，写屏障将黑色对象变为灰色，在重新标记阶段重新扫描灰色对象。破坏了条件①

**G1解决（原始快照SATB）**：并发标记开始前建立对象逻辑快照。写屏障将灰色对象到白色对象的引用断开前，记录旧值。重新标记阶段重新扫描这些旧引用（认为它们仍是可达的）。破坏了条件②

SATB可能造成少量浮动垃圾（某些SATB中记录的对象本来已死但仍被标记为存活），但保证了不漏标。

## 2.15 JIT编译器C1和C2有什么区别？分层编译是什么？
| 特性 | C1(Client Compiler) | C2(Server Compiler) |
|------|------|------|
| 目标 | 快速编译 | 深度优化、快速执行 |
| 编译时间 | 短 | 长 |
| 优化策略 | 简单优化（内联/去虚拟化等基础） | 激进优化（逃逸分析/标量替换/锁消除/代码展开） |
| 启动时间 | 快 | 慢 |
| 峰值性能 | 较低 | 高 |
| 适用模式 | Client | Server（默认） |

**分层编译（Tiered Compilation, JDK8默认开启）**：
- Level 0：解释执行（无编译）、性能计数
- Level 1：C1简单编译（无profiling）
- Level 2：C1简单编译+limited profiling
- Level 3：C1全profiling编译
- Level 4：C2编译（最高优化级别）

工作流程：先解释执行 → 热度达到阈值 → C1编译(Level 1-3) → 收集运行时数据(profiling) → 热度极高 → C2深度优化编译(Level 4)。如果C2反优化（激进优化的假设不再成立），降级回解释执行。

-Xint纯解释执行（最慢）；-Xcomp纯编译执行（启动也全部编译，启动慢）；-Xmixed混合模式（默认，分层编译）。

## 2.16 什么是用户态和内核态？synchronized锁升级与它们的关系？
**用户态**：CPU的一种运行级别，进程运行在用户态时只能访问受限的资源，无法直接访问硬件设备、内存管理等系统资源，不能执行特权指令。

**内核态**：CPU的最高运行级别，拥有对所有硬件和系统资源的完全访问权限，可以执行所有指令。

**切换方式**：当用户程序需要执行特权操作（如文件I/O、网络通信、线程调度、内存分配），必须通过**系统调用（syscall）**触发用户态→内核态的切换。切换过程涉及CPU上下文保存/恢复，开销较大（通常在几百到几千CPU周期）。

**synchronized锁升级与用户态/内核态的关系**：
- **偏向锁/轻量级锁**：完全在用户态通过CAS自旋操作完成，不涉及系统调用，无需切换到内核态，开销低。适合锁竞争不激烈的场景
- **重量级锁**：当锁竞争激烈（CAS多次失败）时，锁膨胀为重量级锁，涉及pthread_mutex等操作系统互斥量，线程被挂起/阻塞需要系统调用，触发用户态↔内核态切换，开销较大

锁升级的本质是：从用户态操作逐步升级到内核态操作，在"短期少量竞争"和"长期激烈竞争"之间做权衡。偏向锁→轻量级锁→重量级锁的升级不可逆（在GC safepoint时JVM可能偏向锁批量撤销）。

## 2.17 Arthas常用命令有哪些？各自作用是什么？
| 命令 | 作用 |
|------|------|
| `dashboard` | 实时显示当前系统的运行面板（线程、内存、GC、运行时信息），相当于可视化top |
| `thread` | 查看所有线程信息，`thread -n 3` 显示CPU占用Top3的线程，`thread -b` 检测死锁 |
| `trace ClassName methodName` | 追踪方法内部调用路径及各步骤耗时，输出类似调用树，可加 `-n` 限制次数 |
| `jad ClassName` | 在线反编译class文件，查看运行的代码实际是什么样的（排查部署版本错误） |
| `watch ClassName methodName` | 观察函数的入参、返回值、异常信息，支持条件过滤和输出表达式 `'{params, returnObj}'` |
| `heapdump` | 导出当前堆dump（类似jmap -dump），`heapdump --live /tmp/dump.hprof` 只dump存活对象 |
| `monitor` | 监控方法调用次数、成功/失败率、平均耗时等聚合统计 |
| `stack` | 输出当前方法的被调用路径（调用栈），结合 `'params[0]>0'` 等条件过滤 |
| `tt` (TimeTunnel) | 方法执行数据时空隧道，记录方法每次调用的入参/返回值/异常/耗时，支持回放 |
| `ognl` | 执行OGNL表达式，查看/修改静态变量值、调用Spring Bean方法等动态操作 |
| `redefine` | 热替换class文件（不重启JVM替换代码），慎用（JDK限制不能新增/删除字段方法） |
| `reset` | 重置增强类（关闭所有Arthas增强），恢复原状 |

典型使用流程：`dashboard`查看整体 → `thread`找问题线程 → `trace`追踪慢方法 → `jad`确认代码版本 → `watch`观察参数/返回值 → `reset`清理。

## 2.18 JVM故障诊断工具有哪些？各自用途？
**JDK自带命令行工具**：
| 工具 | 用途 |
|------|------|
| `jps` | 列出JVM进程ID（类似ps -ef \| grep java） |
| `jstat` | 监视JVM运行时统计信息（类加载、GC、JIT编译等），常用于`jstat -gcutil PID 1000`持续观察GC |
| `jinfo` | 查看和调整JVM参数（可运行时修改flag） |
| `jmap` | 生成堆dump（-dump）、查看堆概要（-heap）、查看类实例统计（-histo） |
| `jstack` | 生成线程堆栈，排查死锁、线程hang住、CPU高 |
| `jhat` | 分析堆dump（已过时，被MAT取代） |
| `jcmd` | JDK7+多功能命令，集jmap/jstack等功能于一身 |

**可视化工具**：
| 工具 | 用途 |
|------|------|
| **JVisualVM** | JDK自带GUI工具，插件丰富（Visual GC可视化GC过程），监控CPU/内存/线程/类 |
| **JMC(Java Mission Control)** | JDK7/8商业特性（JDK11开源），配合JFR(JFR Flight Recorder)做深度性能分析，开销极低（<1%） |
| **MAT** | Eclipse Memory Analyzer Tool，分析堆dump的标准工具，支持Dominator Tree/Leak Suspects/OQL查询 |
| **JProfiler** | 商业性能分析工具，CPU/内存/线程/数据库调用全方位分析，动态插桩 |
| **YourKit** | 商业分析工具，功能类似JProfiler |

**第三方/阿里系工具**：
| 工具 | 用途 |
|------|------|
| **Arthas** | 阿里开源的Java在线诊断工具，线上问题排查利器（无需重启、无需修改代码） |
| **BTrace** | 安全的动态跟踪工具，通过字节码注入实现无侵入监控，Arthas底层部分依赖其思路 |
| **Async-Profiler** | Linux下低开销的CPU/内存采样分析器，生成火焰图 |

**选择建议**：线上实时诊断 → Arthas；堆dump分析 → MAT；GC监控 → jstat+GC日志；性能分析 → JMC/JFR(生产)+JProfiler(开发)；CPU/线程问题 → Arthas+jstack。

## 2.19 为什么建议将元空间初始大小和最大大小设置为一样？
**建议原因**：避免元空间扩容带来的开销和风险。

1. **避免Full GC触发**：元空间扩容时，如果剩余空间不足会触发Full GC来回收无用的类和类加载器。如果每次扩容都触发一次GC，会导致频繁的STW停顿。预先分配好固定大小可避免扩容触发的GC。

2. **避免内存碎片**：元空间使用本地内存（非堆），动态扩容/缩容可能导致本地内存碎片化，影响系统整体性能。固定大小在JVM启动时一次性分配连续内存块。

3. **避免OOM**：元空间默认无上限（受系统内存限制），在某些场景（如大量动态代理类、Groovy脚本、JSP编译、反射生成类）下可能会无限增长，耗尽系统内存导致整个系统OOM。设置固定上限可以"快速失败"，不至于拖垮整个服务器。

4. **性能稳定**：预分配避免了运行时动态向OS申请/释放内存的开销，元空间性能更稳定可预测。

**配置示例**：`-XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=256m`，两者设为相同值。这个策略与堆的`-Xms = -Xmx`设置原因类似——消除运行时扩容/缩容的开销和不确定性。

## 2.20 类加载的触发时机是什么？主动引用和被动引用有什么区别？
JVM 规范规定有且仅有以下 6 种**主动引用**场景会立即触发类的初始化：

1. **new 关键字创建实例**、读取/设置类的静态字段（final static 常量除外）、调用静态方法。
2. **反射调用**（`Class.forName()`，`Constructor/Method/Field.invoke()`）。
3. **初始化子类时，先触发父类初始化**。
4. **启动时指定主类**（main 方法所在的类）。
5. **MethodHandle 的 REF_getStatic/REF_putStatic/REF_invokeStatic 解析**（JDK7+）。
6. **接口中定义了 default 方法时**，实现该接口的类被初始化前，接口也会被初始化。

**被动引用**（不会触发初始化，常被面试官挖坑）：
```java
// 1. 引用父类的静态字段 → 只初始化父类，子类不会初始化
class Parent { static int value = 1; }
class Child extends Parent { static { System.out.println("Child init"); } }
int v = Child.value;  // 只输出 "Parent init"，子类不会被初始化

// 2. 通过数组引用类 → 不触发初始化
Parent[] arr = new Parent[10];  // 不会触发 Parent 类初始化

// 3. 引用类的编译时常量 → 不触发初始化
class ConstClass { static final String CONST = "hello"; }
String s = ConstClass.CONST;  // 编译时常量已在编译期传播，不会触发初始化
```
关键点：编译时常量（static final 且值在编译期确定）在编译阶段就存入调用类的常量池，运行时不需要访问定义它的类，因此不会触发初始化。

## 2.21 TLAB 是什么？如何优化对象分配？
TLAB（Thread Local Allocation Buffer，线程本地分配缓冲）是 JVM 在新生代 Eden 区为每个线程预先分配的一块独占内存区域，用于对象快速分配。

**为什么需要 TLAB**：
- 对象分配是所有线程共用的操作。多个线程同时 new 对象时，如果不加锁会冲突（同一块内存被多个对象占用）。
- 最简单的方法是 CAS + 失败重试，但在高并发下 CAS 竞争激烈会导致性能下降。
- TLAB 让每个线程先在自己的专有区域内分配，写满后再通过 CAS 申请新的 TLAB，大大减少 CAS 冲突。

**TLAB 工作流程**：
1. 每个线程向 Eden 区申请一块 TLAB（通过 CAS）。
2. 线程在自己的 TLAB 内用**指针碰撞**方式分配对象（无需任何同步），`top += objSize`。
3. TLAB 剩余空间放不下新对象 → 根据对象大小：
   - 大对象：直接走堆分配（CAS），不走 TLAB
   - 普通对象：申请新的 TLAB（CAS），老的 TLAB 未用满的空间留给其他对象分配

**相关参数**：
| 参数 | 默认值 | 作用 |
|------|--------|------|
| `-XX:+UseTLAB` | 开启 | 启用 TLAB（JDK8+ 默认开启） |
| `-XX:TLABSize` | 自动计算 | TLAB 初始大小 |
| `-XX:+TLABStats` | — | 打印 TLAB 统计信息 |
| `-XX:+PrintTLAB` | — | 打印 TLAB 分配详情 |

**性能影响**：TLAB 将对象分配的同步开销从"每次 new 都 CAS"降低为"每个 TLAB 耗尽才 CAS"，在大量对象分配的 OLTP 业务中效果显著。

## 2.22 逃逸分析是什么？栈上分配、标量替换和同步消除如何优化？
逃逸分析（Escape Analysis）是 JIT 编译器的一项高级优化技术：分析对象动态作用域，判断对象是否"逃逸"出方法或线程。

**逃逸判断**：
- 对象在方法内创建、不被外部引用、不被 return → 未逃逸 → 可栈上分配或标量替换
- 对象被 return 或被传递给外部变量 → 逃逸 → 必须堆分配

**三个核心优化**：
1. **栈上分配（Stack Allocation）**：未逃逸的对象直接在栈帧中分配，方法结束自动销毁，**不需要 GC 介入**。大幅降低 GC 压力。
2. **标量替换（Scalar Replacement）**：将对象的成员变量拆分为独立的基本类型局部变量。`Point p = new Point(1,2);` 优化为 `int x = 1; int y = 2;`，对象本身不再存在。
3. **同步消除（Lock Elision）**：如果逃逸分析确定对象只被一个线程访问（线程本地对象），则消除对该对象的同步操作。`synchronized(new Object()){}` 中的锁被消除。

**参数**：`-XX:+DoEscapeAnalysis`（JDK8+ 默认开启）。

**局限性**：逃逸分析需要消耗 CPU 进行数据流分析，JIT 会综合判断是否值得做。简单方法内的小对象最容易触发栈上分配和标量替换。HotSpot 目前不支持对象数组的逃逸分析。

## 2.23 ZGC 的工作原理和核心特点是什么？
ZGC（Z Garbage Collector）是 Oracle 从 JDK11 引入的实验性超低延迟收集器，JDK15 生产可用，JDK21 支持分代模式。

**核心特点**：
| 特性 | ZGC | G1 |
|------|-----|----|
| 停顿目标 | <1ms（亚毫秒级） | 默认 200ms |
| 最大堆 | 16TB | 几十 GB |
| 并发阶段 | 标记、整理、重映射全部并发 | 标记并发，部分整理需要 STW |
| 分代支持 | JDK21 支持分代 ZGC | 天然分代 |
| 核心技术 | 着色指针 + 读屏障 | Remembered Set + SATB |

**核心技术——着色指针（Colored Pointer）**：ZGC 在 64 位指针中嵌入元数据（Finalizable/Remapped/Mark1/Mark0 共 4 个状态位），指针本身携带 GC 状态信息。这要求 Linux 的 `/proc/sys/vm/max_map_count` 足够大来承载大量内存映射。

**核心技术——读屏障（Load Barrier）**：每次从堆读取对象引用时，加载屏障检查指针的颜色位。如果指向尚未整理的对象，读屏障更新引用到新位置（"自愈"）。读屏障开销极小（通常 <1%），换来的是并发整理的零 STW。

**Region 类型**：小 Region（2MB）、中 Region（32MB）、大 Region（N×2MB）。弱指针清理（N×2MB大页清理原则），类似 G1 的 Region 概念但更灵活。

**使用方式**：`-XX:+UseZGC`（JDK17+ 直接可用，JDK21+ 推荐启用 `-XX:+ZGenerational` 分代模式）。

## 2.24 如何选择垃圾收集器？（JDK8/11/17/21 各选什么）
| JDK版本 | 默认收集器 | 推荐选择 |
|---------|-----------|---------|
| JDK8 | Parallel Scavenge + Parallel Old（吞吐优先） | CMS（低延迟 Web）/ Parallel（批处理） |
| JDK11 | G1（JDK9+ 默认） | G1（通用）/ ZGC（极低延迟，实验性） |
| JDK17 | G1 | G1（通用）/ ZGC（生产可用，低延迟首选）/ Shenandoah |
| JDK21 | G1 | G1（通用，经典）/ ZGC 分代模式（大堆低延迟首选）/ 虚拟线程 + G1 |

**选型决策矩阵**：
| 场景 | 推荐收集器 | 原因 |
|------|-----------|------|
| 吞吐优先（批处理/后台任务） | Parallel | 多线程并行，最大化吞吐，GC 后台耗时低 |
| 低延迟（Web 应用/API 服务） | G1 | 停顿可控（-XX:MaxGCPauseMillis），平衡吞吐和延迟 |
| 超大堆（32GB+，低延迟） | ZGC（分代模式 JDK21+） | 亚毫秒级停顿，与堆大小脱钩 |
| 内存紧张（<4GB） | Serial | 单线程无切换开销，内存最小时占比最低 |
| 响应时间硬实时 | ZGC / Shenandoah | <1ms 最大停顿目标 |

**G1 调优核心参数**：`-XX:MaxGCPauseMillis=100`（缩小停顿目标）+ `-XX:G1HeapRegionSize=4m`（大堆增大 Region）+ `-XX:InitiatingHeapOccupancyPercent=45`（提前开始并发标记，给混合 GC 更多时间）。

## 2.25 SafePoint 是什么？什么时候会进入 SafePoint？
SafePoint（安全点）是代码中某些特定位置，当 JVM 需要执行 Stop-The-World（STW）操作时，所有用户线程必须到达一个 SafePoint 才能挂起。

**为什么需要 SafePoint**：JVM 不能随时暂停线程——如果线程正在执行指令的一半，寄存器状态不完整，GC 无法准确找到该线程栈帧中的对象引用。SafePoint 保证了线程在已知的、可安全暂停的位置。

**常见进入 SafePoint 的场景**：
- GC 开始前的 STW（最重要的场景）
- 偏向锁撤销/批量重偏向
- 线程 dump（jstack/Arthas）
- 类重定义（热部署/JVMTI retransform）
- 取消偏向锁批量撤销
- 代码反优化（Deoptimization）

**SafePoint 位置**：
- 方法调用后（return 指令）
- 循环跳转指令后（loop back edge，间隔可选）
- 异常跳转位置
- 解释器和编译代码中的特定安全点校验指令

**SafePoint 问题排查**：如果 GC 日志显示 `ref-proc` 或 `sync` 阶段耗时异常长，可能是某些线程长时间未到达 SafePoint（如无循环的长时间计算、大循环缺失 SafePoint 检测、C2 编译的代码"密集执行区"缺少 SafePoint）。使用 `-XX:+PrintSafepointStatistics -XX:PrintSafepointStatisticsCount=1` 打印 SafePoint 统计。

## 2.26 堆内存 -Xms 和 -Xmx 为什么要设置成一样？
**核心原因：避免堆扩容/缩容的开销和 GC 不确定性。**

1. **避免堆扩容触发 Full GC**：当堆内存不足扩容时，JVM 可能触发一次 Full GC 来回收空间。如果每次扩容都伴随 Full GC，会导致不可预期的长时间停顿。生产环境中停顿时长是必须可预测的。

2. **避免内存抖动**：堆的扩缩容涉及向 OS 申请/释放连续内存块，动态扩缩容导致 JVM 内存占用曲线反复抖动，影响机器整体的内存管理。

3. **消除分配决策不确定性**：堆大小变化会影响 GC 决策逻辑（新生代/老年代比例、晋升阈值等），固定堆大小使 GC 行为更加稳定和可预测。

4. **减少与元空间的交互风险**：堆扩缩容可能同时伴随元空间和直接内存的调整，三者交织加剧问题排查难度。

**例外场景**：在容器环境（Docker/K8s）中，如果使用 `-XX:+UseContainerSupport`（JDK8u191+/JDK10+ 默认），堆大小由容器内存限制自动计算。此时显式设置 `-Xms = -Xmx` 可以避免容器环境的"臃肿堆"问题——容器感知的计算结果随时间变化可能导致堆不断扩缩。

**注意**：`-Xms = -Xmx` 加上 `-XX:+AlwaysPreTouch` 在启动时立即触摸所有堆页（物理分配 + 零填充），虽然启动变慢但运行时零 page fault 开销，对低延迟应用至关重要。
