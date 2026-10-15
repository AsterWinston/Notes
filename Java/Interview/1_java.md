# 1. 知识点
- **面向对象三大特性**：封装（隐藏内部实现，通过public方法访问）、继承（子类复用父类代码，单继承）、多态（父类引用指向子类对象，运行时绑定）
- **封装**：使用private修饰字段，提供public getter/setter控制访问权限，保护数据完整性
- **继承**：extends关键字，子类拥有父类所有非private成员，构造方法不继承，子类构造默认调用super()
- **多态**：编译时看左边（父类），运行时看右边（子类），前提是继承+重写+向上转型
- **抽象类vs接口**：抽象类有构造器、可以有非抽象方法、单继承；接口JDK8后支持default/static方法，JDK9支持private方法，多实现
- **JDK8+接口变化**：引入default方法（解决接口演进问题）、static方法（工具方法），接口和抽象类边界模糊
- **JDK9+接口变化**：支持private方法，供default方法内部复用
- **重载(Overload)**：同一类中方法名相同、参数列表不同（类型/个数/顺序），返回值可不同，与异常无关，编译时多态
- **重写(Override)**：子类重写父类方法，方法签名一致，返回值协变（子类型），访问权限不能更严，异常不能更宽，运行时多态
- **== vs equals**：==比较栈中值（基本类型比值，引用类型比地址），equals默认同==，String/Integer等重写为比较内容
- **hashCode与equals契约**：equals相等的对象hashCode必相等；hashCode相等equals不一定相等（哈希碰撞）；重写equals必须重写hashCode
- **为什么重写equals必须重写hashCode**：HashMap/HashSet等集合先用hashCode定位桶，再用equals判断，不重写会导致逻辑相等的对象被存入不同桶
- **深拷贝**：完全复制对象及其引用的对象，两者完全独立，修改不互相影响；实现方式：重写clone逐层克隆、序列化反序列化、手动set
- **浅拷贝**：只复制对象本身的值字段，引用类型字段复制的是地址引用，修改引用对象会相互影响；实现方式：实现Cloneable接口调用super.clone()
- **Throwable**：Java异常体系根类，两个直接子类Error和Exception
- **Error**：JVM无法处理的严重错误（OOM、StackOverflow），不应捕获处理
- **Exception**：程序可处理的异常，分checked（编译时检查，必须try-catch或throws，如IOException）和unchecked（RuntimeException，如NPE）
- **try-catch-finally执行顺序**：try执行→catch匹配到则执行catch→finally一定执行（除非System.exit(0)）；finally中的return会覆盖try/catch的return；try中return值先在finally前暂存
- **try-with-resources**：JDK7引入，实现AutoCloseable接口的资源会在try块结束后自动调用close()，无论是否异常，语法更简洁
- **泛型**：类型参数化，编译期类型安全检查，运行期类型擦除
- **类型擦除**：泛型信息编译后擦除，替换为上限类型（无界→Object），在需要时插入强制类型转换；因此List<String>和List<Integer>运行时Class对象相同
- **PECS原则**：Producer extends（读取用? extends T，只能读不能写）、Consumer super（写入用? super T，只能写读为Object）
- **上界通配符(? extends T)**：T及其子类，常用于读取数据
- **下界通配符(? super T)**：T及其父类，常用于写入数据
- **反射机制**：运行时获取类信息并操作对象的能力，核心类Class/Method/Field/Constructor
- **获取Class对象方式**：类名.class、对象.getClass()、Class.forName("全限定名")、类加载器loadClass
- **反射常用API**：getDeclaredMethods/getMethods、getDeclaredFields/getFields、newInstance、setAccessible暴力反射访问私有成员
- **反射性能影响**：比直接调用慢几十倍，因为涉及权限检查、类型检查、装箱拆箱等，可使用setAccessible(true)减少部分检查
- **注解**：元数据标记，JDK5引入，可被编译器/VM/运行时读取
- **元注解**：@Target（作用目标TYPE/METHOD/FIELD等）、@Retention（保留策略SOURCE/CLASS/RUNTIME）、@Documented、@Inherited、@Repeatable(JDK8)
- **自定义注解**：@interface定义，通过反射的isAnnotationPresent/getAnnotation获取运行时注解信息
- **注解处理器**：javax.annotation.processing，编译期扫描注解生成代码（如Lombok），以AbstractProcessor为基类，通过SPI注册
- **SPI机制**：Service Provider Interface，在META-INF/services/下以接口全限定名命名的文件，内容为实现类全限定名；ServiceLoader.load()加载
- **JDBC驱动加载**：DriverManager通过SPI加载META-INF/services/java.sql.Driver中的驱动实现
- **SPI vs Spring Boot自动装配**：SPI是JDK原生，简单但不支持条件装配/IoC；Spring Boot通过@AutoConfiguration+spring.factories实现条件化自动装配
- **序列化**：Serializable标记接口，将对象转为字节流存储或传输；Externalizable是其子接口，需要手动实现writeExternal/readExternal
- **transient**：修饰字段不参与默认序列化，反序列后值为默认值（null/0）
- **static字段**：不参与序列化，因为属于类而非对象
- **serialVersionUID**：版本控制，反序列化时验证类版本是否兼容，不指定则编译器自动生成（类变化会变），建议显式定义
- **String不可变性**：final类、char数组final且private（JDK9+改为byte数组）、所有方法返回新String对象；好处：字符串常量池、线程安全、Hash缓存
- **字符串常量池**：JDK7后移到堆中，字面量创建的字符串自动入池，new String("hello")创建两个对象（常量池+堆）；intern()尝试将字符串放入池中并返回引用
- **StringBuilder**：可变，非线程安全，效率高，单线程使用
- **StringBuffer**：可变，线程安全（synchronized方法），效率较低，多线程使用
- **BigDecimal**：高精度计算，使用字符串构造器避免精度丢失（new BigDecimal("0.1")而非new BigDecimal(0.1)）
- **BigDecimal equals vs compareTo**：equals比较值和精度（scale），1.0和1.00不等；compareTo仅比较数值大小
- **成员内部类**：定义在类内部，可直接访问外部类成员，外部类需通过内部类对象访问
- **静态内部类**：static修饰，不依赖外部类实例，只能访问外部类静态成员，常用作Helper
- **局部内部类**：定义在方法/代码块中，作用域仅限于方法内
- **匿名内部类**：没有类名，直接new接口或父类并实现方法，常用于回调；JDK8后可被Lambda替代（函数式接口）
- **final**：修饰类不可继承，修饰方法不可重写，修饰变量不可修改（基本类型值不变，引用类型地址不变但对象内容可变）
- **finally**：异常处理中一定执行的代码块
- **finalize**：Object方法，GC前调用一次，JDK9已废弃，因执行时机不确定、性能差
- **static关键字**：修饰变量（类变量，所有实例共享）、方法（类方法，无this）、代码块（类加载执行一次）、内部类、导入（import static静态导入方法/常量）
- **Comparable**：内部比较器，类实现compareTo方法，自然排序，排序逻辑耦合在类内部
- **Comparator**：外部比较器，单独定义比较规则，排序逻辑与类解耦，支持多种排序策略；可配合Lambda使用
- **动态代理**：运行时生成代理类，在方法调用前后插入逻辑
- **JDK动态代理**：基于接口的Proxy.newProxyInstance + InvocationHandler，生成的代理类继承Proxy实现指定接口，使用反射invoke
- **CGLIB动态代理**：基于继承，通过Enhancer+MethodInterceptor生成目标类的子类，使用ASM字节码，final类/方法无法代理
- **Spring AOP选择**：目标类有接口默认用JDK代理，无接口用CGLIB；proxy-target-class=true强制CGLIB
- **自动拆装箱**：基本类型与包装类自动转换，编译器插入intValue/valueOf调用；缓存Integer(-128~127)、Long(-128~127)、Character(0~127)、Boolean(true/false)
- **自动装箱null安全问题**：Integer i = null; int j = i; 自动拆箱调用i.intValue()抛NPE
- **增强for循环原理**：编译后使用Iterator实现，被遍历对象必须实现Iterable接口（数组除外，编译为普通for循环）
- **枚举(Enum)**：Java的enum继承java.lang.Enum，不可继承，可实现接口；values()返回所有值、valueOf(String)按名称获取、ordinal()返回序号
- **枚举单例**：天然防反射（Constructor.newInstance抛异常）、防序列化（反序列化返回同一个实例）、线程安全（类加载时初始化），最安全的单例实现
- **数据脱敏**: 保护敏感数据的技术，通过修改数据显示/存储形式使未授权人员无法识别。常用于手机号(138****1234)、身份证、邮箱等。实现方式: 自定义Jackson序列化器 + Hutool的`StrUtil.hide`方法 + `@Desensitization`注解标记字段
- **SPU和SKU**: SPU (Standard Product Unit) 关注商品本身信息；SKU (Stock Keeping Unit) 侧重于具体规格和库存管理。一个SPU包含多个SKU (如同一款式衣服的不同颜色/尺码组合)
- **装箱的好处**: (1) 简化代码 `Integer i = 10`;(2) 泛型支持 `List<Integer>`;(3) 兼容旧代码调用需要包装类型参数的方法;(4) 可处理null值;(5) 自动装箱减少手动new的开销
- **可变长参数(Varargs)**: 方法接受不定数量参数，用`...`声明，方法内部作为数组处理，必须是参数列表最后一个参数
- **变量和常量的区别**: 变量值运行时可变/驼峰命名/运行时分配内存；常量final修饰/值不可变/全大写下划线命名/编译期确定值分配一次
- **Integer缓存机制**: Integer内部缓存 -128 ~ 127 的常用整数对象 (可通过`java.lang.Integer.IntegerCache.high`调整上限)。`Integer.valueOf()`在缓存范围内直接返回缓存对象，避免重复创建，本质是**享元模式**
- **类加载5阶段**：加载（读.class字节流生成Class对象）→ 验证（文件格式/元数据/字节码校验）→ 准备（为static变量分配内存并赋默认值0/null，static final常量直接赋确定值）→ 解析（符号引用替换为直接引用）→ 初始化（执行<clinit>，为static变量赋程序员指定的值）
- **类初始化时机（主动引用6种）**：new实例、读写static字段（final编译期常量除外）、调用static方法、反射Class.forName、初始化子类先触发父类初始化、含main的启动类
- **类不会初始化（被动引用3种）**：子类引用父类静态字段（只初始化父类）、通过数组定义引用类（`A[] a = new A[10]`不触发A初始化）、引用编译期常量（static final值可编译期确定，被内联进常量池直接使用）
- **Class.forName vs loadClass**：Class.forName执行初始化（加载+连接+初始化，JDBC驱动靠static块注册驱动）；ClassLoader.loadClass只加载不初始化（Spring懒加载Bean、AOP延迟代理用此特性）
- **初始化顺序（有继承）**：父类静态变量/静态块 → 子类静态变量/静态块 → 父类实例变量/实例块 → 父类构造器 → 子类实例变量/实例块 → 子类构造器；静态只执行一次，实例每次new都按此顺序执行
- **this关键字**：引用当前对象；this.x区分成员字段与局部变量、this()调用本类重载构造器（必须首行且只能调用一次）；static方法中不能使用this
- **super关键字**：super.x访问父类成员、super.method()调用父类方法、super()调用父类构造器（子类构造默认隐式调用无参super()，必须首行）；super不能跨代只能访问直接父类
- **switch支持类型**：byte/short/char/int及其包装类、枚举enum(JDK5)、String(JDK7)；不支持long/float/double/boolean；编译分为tableswitch（case密集连续，跳转表O(1)）和lookupswitch（case稀疏，二分查找）
- **类型提升(Type Promotion)**：byte/short/char参与算术运算自动提升为int（byte+byte结果也是int）；表达式中出现long/float/double则结果提升为最高类型
- **隐式/显式转换**：宽化自动（byte→short→int→long→float→double，char→int）；窄化需强转（可能丢精度/溢出）；复合赋值+=、-=隐含强转（`byte b=10; b+=1`合法，`b=b+1`编译错）
- **整数溢出**：int最大约21亿，超范围回绕为负数；用Math.addExact/Math.multiplyExact（溢出抛ArithmeticException）或升级long/BigDecimal
- **位运算**：&按位与、|按位或、^异或、~取反、<<左移(乘2^n)、>>右移(除2^n，符号位填充)、>>>无符号右移(高位补0)；应用：n&1判奇偶、n<<1乘2、a^b^b==a交换两数、HashMap扰动(h^(h>>>16))、权限位掩码
- **短路运算**：&&/||短路（左条件已能确定结果则不执行右边），&/|不短路（两边都执行）；&\|同时用于按位与/或
- **字符串拼接底层**：+拼接编译期常量直接折叠为常量池字符串；变量拼接编译为StringBuilder.append；循环中每轮new StringBuilder有性能问题（建议循环外复用）；JDK9改用invokedynamic+StringConcatFactory动态拼装
- **intern()版本差异**：native方法；JDK6将字符串复制到永久代（返回副本引用）；JDK7+常量池在堆中，intern先查池，无则把当前对象的引用直接放入池并返回该引用（不复制）
- **Arrays.asList的坑**：返回的ArrayList是Arrays内部类（定长），底层仍是原数组，add/remove抛UnsupportedOperationException，修改会同步影响原数组；原始类型数组`int[]`会被当作单个元素
- **Objects工具类**：requireNonNull(obj,msg)判空抛NPE、equals(null安全比较)、isNull/nonNull、hashCode(自动处理null为0)、toString
- **System.arraycopy vs Arrays.copyOf**：System.arraycopy是native方法，需目标数组已存在，只能数组间复制；Arrays.copyOf内部调arraycopy，自动创建目标数组并返回
- **异常链**：catch后抛新异常时把原异常作为cause传入（new XxxException(msg, e)或initCause(e)），保留根因；靠Throwable.getCause()追溯
- **桥方法(Bridge Method)**：类型擦除后编译器为保持多态生成的合成方法。实现Comparator<String>会生成compare(Object,Object)桥方法，先强转再调compare(String,String)；反射getMethods能看到且调用可能抛ClassCastException
- **匿名内部类访问局部变量**：JDK7前要求final，JDK8+改为effectively final（声明后不再修改即可）；原因：内部类拷贝了局部变量副本，若变量可变会导致方法返回后内外不一致，故强制不可变
- **集合体系**：Collection（Iterable根）分List/Set/Queue，Map是独立体系不继承Collection；核心实现：ArrayList、LinkedList、HashSet、TreeSet、HashMap、TreeMap、ConcurrentHashMap
- **ArrayList vs LinkedList**：ArrayList=Object[]数组，随机访问O(1)、尾插O(1)均摊、扩容1.5倍；LinkedList=双向链表，头尾操作O(1)、随机访问O(n)、内存开销大
- **HashMap**：JDK8数组+链表+红黑树（链表≥8且数组≥64才树化）；hash=(h=hashCode())^(h>>>16)，(n-1)&hash定位；容量2的幂、负载因子0.75、扩容2倍；JDK7头插法扩容死链 vs JDK8尾插法仍有数据覆盖问题
- **ConcurrentHashMap**：JDK7分段锁Segment(继承ReentrantLock) vs JDK8 CAS+synchronized锁桶头节点；get无锁，put锁桶，多线程协同扩容
- **fail-fast**：modCount与expectedModCount不一致抛ConcurrentModificationException；迭代中删除用迭代器remove()，多线程用CopyOnWriteArrayList
- **ArrayList线程安全替代**：Vector(过时)、Collections.synchronizedList、CopyOnWriteArrayList(写时复制，读多写少)
- **线程6状态**：NEW / RUNNABLE / BLOCKED / WAITING / TIMED_WAITING / TERMINATED
- **锁升级**：无锁 → 偏向锁 → 轻量级锁(自旋) → 重量级锁(阻塞)，基于对象头MarkWord的低2位标记
- **volatile**：可见性 + 禁止指令重排，不保证原子性；DCL单例的instance必须volatile
- **CAS/ABA**：CAS是CPU原子指令(compareAndSwap)；ABA问题用AtomicStampedReference(版本号)解决
- **AQS**：volatile state + CLH队列，ReentrantLock/CountDownLatch/Semaphore等JUC同步器的基础
- **ReentrantLock vs synchronized**：ReentrantLock可中断/超时/公平/多Condition，需手动释放；synchronized自动释放、JVM锁升级优化好
- **线程池7参数+4拒绝策略**：core/max/keepAlive/queue/factory/handler；AbortPolicy抛异常(默认)/CallerRuns调用者执行/Discard丢弃/DiscardOldest丢最老重试
- **ThreadLocal**：每线程一个ThreadLocalMap；Entry的key是弱引用，ThreadLocal被GC后value强引用残留导致泄漏，用完必须remove()
- **BIO/NIO/AIO**：同步阻塞(一连接一线程) / 同步非阻塞(Selector多路复用) / 异步非阻塞(OS回调)
- **NIO三组件**：Channel(双向通道) / Buffer(数据容器) / Selector(多路复用器)
- **零拷贝**：mmap文件映射、FileChannel.transferTo(sendfile)、Netty的DirectByteBuf/FileRegion/CompositeByteBuf
- **Reactor线程模型**：主从Reactor多线程(Netty)，BossGroup接受连接、WorkerGroup处理读写

# 2. 面试题

## 2.1 面向对象三大特性是什么？分别解释
封装：将数据和操作数据的方法绑定在一起，通过访问权限控制隐藏内部实现细节，只暴露必要的公开接口。好处是降低耦合、提高安全性和可维护性。

继承：子类通过extends关键字获得父类的属性和方法，实现代码复用。Java单继承，但可通过实现多个接口达到多继承效果。子类构造器默认调用super()，子类可以重写父类方法。

多态：同一个行为具有多个不同表现形式。编译时类型和运行时类型不一致，父类引用指向子类对象，调用方法时执行子类重写的方法（动态绑定）。实现前提：继承+重写+向上转型。

## 2.2 抽象类和接口的区别？（JDK8+）
共同点：都不能实例化，都可以包含抽象方法。

区别：
| 特性 | 抽象类 | 接口 |
|------|--------|------|
| 关键字 | abstract class | interface |
| 继承/实现 | 单继承 | 多实现 |
| 构造器 | 有 | 无 |
| 成员变量 | 任意类型 | 默认public static final |
| 方法 | 可有非抽象方法 | JDK8前只能抽象，JDK8支持default/static，JDK9支持private |
| 访问修饰符 | 任意 | 默认public |
| 设计理念 | is-a，提取共性 | has-a/can-do，定义行为契约 |

JDK8引入default方法后接口可以添加方法默认实现，使得接口演进更方便，但接口依然没有构造器和状态。

## 2.3 重载(Overload)和重写(Override)的区别

| 维度 | 重载 | 重写 |
|------|------|------|
| 发生位置 | 同一类中 | 父子类之间 |
| 方法签名 | 方法名相同，参数列表不同 | 完全相同 |
| 返回值 | 可以不同 | 相同或协变 |
| 异常 | 可以任意 | 不能抛出更宽泛的检查异常 |
| 访问权限 | 可以不同 | 不能比父类更严格 |
| 多态类型 | 编译时多态（静态绑定） | 运行时多态（动态绑定） |
| static | 可重载 | 不能重写（但可隐藏） |
| final | 可重载 | 不可重写 |

重写规则记忆："两同两小一大"——方法名和参数列表相同、返回值类型和抛出异常更小、访问权限更大。

## 2.4 == 和 equals 的区别
- ==：比较栈内存中的值。对于基本类型比较值的大小，对于引用类型比较引用地址（是否指向同一对象）
- equals()：Object中默认使用==比较地址，子类（String、Integer等）重写后比较对象内容是否相等

示例：
```java
String s1 = new String("hello");
String s2 = new String("hello");
s1 == s2;        // false，不同对象
s1.equals(s2);   // true，内容相同

Integer a = 200;
Integer b = 200;
a == b;          // false，超出缓存范围(-128~127)
```

## 2.5 hashCode与equals的契约关系？为什么要一起重写？
契约：equals相等的两个对象，hashCode必须相等；hashCode相等的对象，equals不一定相等（哈希碰撞）。

原因：HashMap/HashSet等哈希容器先通过hashCode定位数据桶位置，再通过equals确认元素是否相同。如果只重写equals不重写hashCode，两个逻辑相等的对象hashCode不同，会被分配到不同桶，导致put时存入重复"相同"元素，get时找不到元素。

HashMap判断key是否相等的逻辑：`p.hash == hash && (k == key || key.equals(k))`，先用hashCode快速筛选，再用equals精确判断。

## 2.6 深拷贝和浅拷贝的区别？如何实现深拷贝？
- 浅拷贝：复制对象本身，引用类型字段复制的是引用地址（指向同一个对象）。实现：实现Cloneable接口直接调用super.clone()
- 深拷贝：复制整个对象图，包括所有引用对象，修改互不影响。实现方式：
  1. 重写clone方法，逐层手动克隆引用类型字段
  2. 使用序列化反序列化：`ByteArrayOutputStream + ObjectOutputStream` → `ByteArrayInputStream + ObjectInputStream`
  3. 使用JSON序列化（如Jackson/Gson）
  4. Apache Commons的SerializationUtils.clone()
  5. 构造函数复制（new对象时传入原始对象逐个赋值）

## 2.7 Java异常体系是怎样的？checked和unchecked区别？
Throwable是根类，分为：
- Error：JVM错误（OOM、StackOverflow），不可恢复，不应catch
- Exception：程序异常
  - Checked异常（编译时异常）：IOException、SQLException、ClassNotFoundException，编译期强制处理（try-catch或throws），否则编译不通过
  - Unchecked异常（运行时异常）：RuntimeException及其子类（NPE、IndexOutOfBounds、ArithmeticException），编译期不强制处理，程序逻辑错误

try-catch-finally中，finally一定会执行（除非System.exit(0)）。try中有return时，先把返回值暂存，再执行finally，最后返回暂存值。但如果finally也有return，会覆盖try的return值。

## 2.8 try-with-resources原理是什么？
JDK7引入，实现了AutoCloseable接口的资源对象，在try代码块执行完毕后自动调用close()方法释放资源，无论是否发生异常。

```java
try (BufferedReader br = new BufferedReader(new FileReader("file"))) {
    return br.readLine();
}
// 编译后等价于在finally中判断资源!=null后调用close()
```

注意：try-with-resources本质上就是编译后生成finally块调用close，但当try块和close方法都抛异常时，try块的异常会被抛出，close的异常会被压制（suppressed），通过getSuppressed获取。

## 2.9 泛型的类型擦除是什么？PECS原则是什么？
**类型擦除**：Java泛型通过编译时类型擦除实现。编译后泛型信息被擦除，泛型类型参数被替换为上限类型（无上限则替换为Object），并在需要的地方插入强制类型转换。因此`List<String>`和`List<Integer>`在运行时是同一个Class对象，无法通过`instanceof`检查泛型类型。

**PECS原则**（Producer Extends, Consumer Super）：
- `? extends T`（上界通配符）：生产者，只能从中读取T类型对象，不能写入（因为不知道具体子类型）。如List<? extends Number>可以get但不能add
- `? super T`（下界通配符）：消费者，可以向其中写入T类型对象，读取只能得到Object。如List<? super Integer>可以add但不能get到具体类型

## 2.10 什么是反射？有哪些应用场景？
反射：在运行时动态获取类的信息（构造器、方法、字段、注解等）并操作对象的能力。

获取Class对象三种方式：`Class.forName()`、`类名.class`、`对象.getClass()`。

应用场景：
- JDK动态代理（Proxy + InvocationHandler）
- Spring IoC（实例化Bean、依赖注入、执行初始化方法）
- Spring AOP代理
- JDBC加载驱动：Class.forName("com.mysql.cj.jdbc.Driver")
- 框架配置文件的类名加载
- 注解处理器和序列化框架（Jackson/Gson）

缺点：性能开销（比直接调用慢数十倍）、编译期类型安全丢失、代码可读性降低。

## 2.11 注解的元注解有哪些？如何自定义注解？
元注解：
- @Target：指定注解可以应用的元素类型（TYPE类、FIELD字段、METHOD方法、PARAMETER参数等）
- @Retention：指定注解保留策略（SOURCE编译时丢弃/Lombok、CLASS在class文件中但运行时不加载/默认、RUNTIME运行时可通过反射获取）
- @Documented：注解会被javadoc工具记录
- @Inherited：子类可以继承父类的该注解
- @Repeatable(JDK8)：同一个位置可重复使用该注解

自定义注解：使用`@interface`定义，通过反射的`isAnnotationPresent()`和`getAnnotation()`获取运行时注解信息进行逻辑处理。

## 2.12 Java SPI机制是什么？与Spring Boot自动装配的区别？
SPI（Service Provider Interface）：JDK内置的服务发现机制。在`META-INF/services/`目录下以接口全限定名创建文件，文件内容为实现类的全限定名。通过`ServiceLoader.load(接口.class)`加载所有实现类。

典型应用：JDBC驱动加载（`java.sql.Driver`）、SLF4J日志门面、Dubbo扩展点加载。

与Spring Boot自动装配的区别：
- SPI：纯JDK机制，简单，按需加载所有实现，使用迭代器模式
- Spring Boot自动装配：基于spring.factories或@AutoConfiguration + @Conditional条件化装配，支持IoC、条件注解（@ConditionalOnClass/@ConditionalOnMissingBean），更灵活强大
- SPI是底层标准，Spring的自动装配可以借助SPI但结合了更多Spring特性

## 2.13 Serializable和Externalizable区别？transient作用？
Serializable：标记接口，序列化机制全自动，默认序列化所有非transient、非static字段。
Externalizable：继承Serializable，需要手动实现`writeExternal()`和`readExternal()`，对序列化过程有完全控制。
transient：修饰的字段不参与序列化，反序列化后值为该类型的默认值（对象null、int 0等）。
static字段不参与序列化（属于类，不属于对象）。
serialVersionUID：显式定义可保证反序列化时类版本一致性；不定义则由JVM根据类结构自动生成，类改动后可能导致反序列化失败。

## 2.14 String、StringBuilder、StringBuffer的区别？
| 特性 | String | StringBuilder | StringBuffer |
|------|--------|---------------|--------------|
| 可变性 | 不可变 | 可变 | 可变 |
| 线程安全 | 安全（不可变） | 不安全 | 安全（synchronized） |
| 性能 | 拼接产生新对象 | 快 | 较慢 |
| 适用场景 | 少量字符串操作 | 单线程大量拼接 | 多线程大量拼接 |
| JDK版本 | JDK1.0 | JDK1.5 | JDK1.0 |
| 存储 | JDK9+ byte[] | byte[] | byte[] |

String不可变原因：final类、final private char[]（JDK9+ byte[]）、所有方法返回新String对象。好处：安全、高效（字符串常量池复用）、hashCode缓存。

## 2.15 BigDecimal精度问题？equals和compareTo区别？
精度问题：浮点数在计算机中无法精确表示十进制小数（如0.1在二进制中是无限循环），因此`new BigDecimal(0.1)`会得到一个近似值。必须使用字符串构造器`new BigDecimal("0.1")`或`BigDecimal.valueOf(0.1)`。

equals vs compareTo：
- equals：比较数值 和 精度（scale），`new BigDecimal("1.0").equals(new BigDecimal("1.00"))`返回false
- compareTo：仅比较数值大小，`new BigDecimal("1.0").compareTo(new BigDecimal("1.00"))`返回0（相等）
建议：数值比较用compareTo，逻辑判断使用equals需注意精度一致。

## 2.16 内部类有哪几种？分别有什么特点？
- 成员内部类：非static修饰，可直接访问外部类所有成员，通过`外部类.this`引用外部类实例，外部类需创建内部类对象才可访问内部类成员。
- 静态内部类：static修饰，不依赖外部类实例，只能访问外部类静态成员，常用于工具Helper类或Builder模式。
- 局部内部类：定义在方法或代码块中，作用域仅限于该方法，可访问外部类成员和方法的final局部变量（JDK8+effectively final即可）。
- 匿名内部类：无名称，直接new父类/接口实现，常用于一次性回调。JDK8+当接口为函数式接口时，可用Lambda表达式替代。

## 2.17 final、finally、finalize的区别？
final：修饰符。修饰类→不可继承（如String）；修饰方法→不可重写；修饰变量→基本类型值不可变，引用类型地址不可变但对象内容可变；修饰参数→方法内不可修改。
finally：异常处理关键字，finally块中的代码无论是否发生异常都会执行（System.exit(0)除外）。
finalize：Object类的方法，protected void finalize()，对象被GC回收前调用一次。JDK9标记为@Deprecated(since="9")，因为执行时机不保证、性能开销大、可能导致对象复活。

## 2.18 static关键字的用法和作用？
- 静态变量（类变量）：属于类，被所有实例共享，类加载时初始化，仅一份。
- 静态方法：属于类，可通过类名调用，没有this引用，只能访问静态成员。
- 静态代码块：类加载时执行一次，用于初始化静态变量（比构造器更早）。
- 静态内部类：不依赖外部类实例，声明和使用更独立。
- 静态导入：`import static`可导入其他类的静态方法和常量，直接使用无需类名前缀。
执行顺序（单类）：静态代码块 > 匿名代码块 > 构造器。有继承时：父类静态 > 子类静态 > 父类匿名代码块 > 父类构造器 > 子类匿名代码块 > 子类构造器。

## 2.19 JDK动态代理和CGLIB动态代理的区别？
| 维度 | JDK动态代理 | CGLIB代理 |
|------|-------------|-----------|
| 机制 | 实现接口 | 生成子类 |
| 代理对象 | Proxy.newProxyInstance + InvocationHandler | Enhancer + MethodInterceptor |
| 目标 | 必须实现接口 | 非final类即可 |
| 方法调用 | 反射 invoke() | FastClass 索引直接调用 |
| 性能 | 生成快，执行稍慢 | 生成慢，执行快 |
| 局限性 | 只能代理接口方法 | 不能代理final类/方法 |
| Spring AOP | 默认（有接口时） | 无接口或proxy-target-class=true时 |

Spring AOP选择逻辑：如果目标实现了接口，默认使用JDK动态代理；如果没有实现接口，或者设置了`proxy-target-class=true`，使用CGLIB。当目标类是final时，只能使用JDK动态代理（需有接口）。

## 2.20 自动拆装箱的缓存机制和null安全风险？
缓存机制：包装类的valueOf()方法对常用值做了缓存：
- Integer：-128 ~ 127（可通过-XX:AutoBoxCacheMax调整上限）
- Long：-128 ~ 127
- Short：-128 ~ 127
- Byte：全部（-128 ~ 127）
- Character：0 ~ 127（ASCII范围）
- Boolean：true和false都缓存
- Float/Double：不缓存

null安全风险：Integer i = null; int j = i; 自动拆箱调用i.intValue()时会抛NullPointerException。集合接收Integer但取出赋值给int也可能触发NPE。防御：使用前判空或使用Optional。

## 2.21 什么是数据脱敏？如何实现？
数据脱敏是指对敏感数据（手机号、身份证号、邮箱、银行卡号等）进行变形处理，使未授权人员无法识别原始数据的技术。

实现方式：
1. 自定义Jackson序列化器：实现`JsonSerializer`，在`serialize`方法中对字段值进行处理
2. 使用Hutool工具类：`StrUtil.hide(phone, 3, 7)`将手机号中间4位替换为`****`
3. 定义`@Desensitization`注解：标记需要脱敏的字段，指定脱敏类型（手机号/身份证/邮箱等）
4. 搭配AOP或拦截器：在数据返回前端前统一处理，也可在数据库查询层使用SQL函数脱敏

常见脱敏规则：
- 手机号：留前3后4，中间用`****`替代
- 身份证：保留前6后4位
- 银行卡：保留后4位
- 姓名：保留首字符，其余用`*`替代
- 邮箱：保留首字符和@及后半部分，如`j***@example.com`

## 2.22 SPU和SKU的区别与联系？
**SPU（Standard Product Unit，标准产品单元）**：描述商品的宏观属性，与库存无关。如iPhone 15是一个SPU，关注商品名称、品牌、描述、图片等通用信息。

**SKU（Stock Keeping Unit，库存量单位）**：描述商品的具体规格属性，与库存直接关联。如iPhone 15 黑色 128GB是一个SKU，关注颜色、容量、价格、库存量、条形码等具体信息。

**联系**：一个SPU对应多个SKU，SPU是SKU的父级概念。例如：
- SPU：某品牌T恤 → SKU：白色S码、白色M码、黑色S码、黑色M码
- SPU：iPhone 15 → SKU：黑色128G、黑色256G、白色128G、白色256G

数据库建模通常有两张表：`spu`表（商品通用信息）+ `sku`表（规格属性+库存+价格，外键关联spu）。

## 2.23 装箱有哪些好处？
1. **简化代码**：自动装箱让代码更简洁 `Integer i = 10` 而非 `Integer i = new Integer(10)`
2. **泛型支持**：Java泛型不支持基本类型，必须使用包装类。如`List<Integer>`合法，`List<int>`不合法
3. **兼容旧代码**：调用需要`Object`类型参数的方法时，装箱使得基本类型可以直接传入
4. **可处理null值**：包装类可为null，表示"无值"状态，在某些场景（如数据库可空字段映射）非常有用
5. **自动装箱性能**：编译器自动调用`valueOf()`，相比手动`new Integer()`可复用缓存对象（-128~127），减少内存分配

## 2.24 什么是可变长参数？
可变长参数（Varargs）允许方法接受不定数量的同类型参数，用`类型... 变量名`声明。编译后可变参数会被转换为数组，因此方法内部将其作为数组处理。

规则：可变参数必须是方法参数列表的**最后一个参数**，一个方法最多只能有一个可变参数。

```java
// 声明
public static int sum(int... nums) {
    int total = 0;
    for (int n : nums) { total += n; }
    return total;
}
// 调用
sum(1, 2, 3);       // 6
sum(10, 20);         // 30
sum();               // 0（空数组）
```

当存在重载时：`method(int... nums)` 和 `method(int a, int... nums)` 都合法，调用`method(1)`会优先匹配更精确的`method(int a)`（如果有），否则匹配可变参数版本。同时有`method(int... nums)`和`method(Integer... nums)`时调用`method(1, 2)`会编译报错（歧义）。

## 2.25 Java中变量和常量有什么区别？
| 维度 | 变量 | 常量 |
|------|------|------|
| 修饰符 | 无 | final |
| 值可变性 | 运行时可变 | 不可变（初始化后不可修改） |
| 命名规范 | 驼峰命名（myVariable） | 全大写+下划线（MY_CONSTANT） |
| 内存分配 | 运行时分配 | 编译期确定，分配一次 |
| 作用 | 存储可变数据 | 存储不变值，提高可读性 |
| 示例 | `int count = 10;` | `final int MAX_COUNT = 100;` |

注意事项：
- 基本类型常量：值不可变
- 引用类型常量：引用地址不可变，但对象内容仍可变（除非对象本身不可变如String）
- 空白final：声明时未赋值的final变量，可在构造器中赋值一次
- static final 常量在准备阶段就赋确定值（基本类型/字符串），比普通final更早

## 2.26 Integer的缓存机制是什么？范围可以调整吗？
Integer内部使用缓存池（IntegerCache）缓存了 -128 到 127 的所有Integer对象，本质是**享元模式（Flyweight Pattern）**。调用`Integer.valueOf(int)`时，如果参数在缓存范围内，直接返回缓存中的对象，否则创建新对象。

```java
Integer a = 127, b = 127;
a == b;  // true（缓存对象相同）

Integer c = 128, d = 128;
c == d;  // false（超出缓存范围，分别新建对象）

Integer e = new Integer(127), f = new Integer(127);
e == f;  // false（new 不走 valueOf，直接新建）
```

**范围可以调整**：通过JVM参数 `-XX:AutoBoxCacheMax=<size>` 或设置系统属性 `-Djava.lang.Integer.IntegerCache.high=500` 可以调整缓存上限。但**只能调高不能调低**（最低-128不变），且下限必须是-128。

类似缓存机制也适用于：Long(-128~127)、Short(-128~127)、Byte(全部)、Character(0~127)、Boolean(true/false)。Float和Double不缓存。

## 2.27 Java 是值传递还是引用传递？
Java **只有值传递**，不存在引用传递。核心区别在于传递的是"值的副本"还是"变量地址本身"：

**对于基本类型**：传递的是数值的副本，方法内修改不影响原变量。

**对于引用类型**：传递的是**对象引用地址的副本**。方法接收的是一个指向相同对象的新引用，所以：
- 通过这个引用**修改对象的内部状态**（如 `obj.setXxx()`）→ 会影响原对象
- 在方法内**更改引用本身指向**（如 `obj = new Object()`）→ 不影响原引用

```java
public static void main(String[] args) {
    User u = new User("张三");
    changeName(u);      // u.name 变为 "李四"（修改了对象内部状态）
    changeRef(u);       // u 仍然指向原张三 User（u 没有变成 new User）
}
void changeName(User user) { user.setName("李四"); }
void changeRef(User user) { user = new User("王五"); }
```

**为什么不是引用传递**：真正的引用传递（如 C++ 的 `&` 引用、C# 的 `ref`）可以在方法内改变外部变量的引用指向。Java 传递的是引用副本，无法修改原引用变量本身。网传"基本类型值传递、引用类型引用传递"是错误的简化。

## 2.28 面向对象设计七大原则（SOLID）分别是什么？
| 原则 | 英文全称 | 核心含义 |
|------|---------|---------|
| SRP 单一职责 | Single Responsibility Principle | 一个类只负责一项职责。变化原因只有一个，提高内聚性 |
| OCP 开闭原则 | Open-Closed Principle | 对扩展开放，对修改关闭。通过抽象扩展功能，不修改已有代码 |
| LSP 里氏替换 | Liskov Substitution Principle | 子类必须能完全替换父类，不破坏程序正确性。子类不应重写父类非抽象方法的预期行为 |
| ISP 接口隔离 | Interface Segregation Principle | 客户端不应被迫依赖它不使用的接口。臃肿接口拆分为多个精专接口 |
| DIP 依赖倒置 | Dependency Inversion Principle | 高层模块不应依赖低层模块，两者都应依赖抽象。抽象不应依赖细节，细节应依赖抽象 |
| LoD 迪米特法则 | Law of Demeter | 最少知道原则。一个对象应保持对其他对象最少的了解，只与直接朋友通信 |
| CRP 合成复用 | Composite Reuse Principle | 优先使用组合/聚合关系（黑盒复用），其次使用继承（白盒复用） |

SOLID 是前五个的首字母缩写。这些原则不是孤立的，它们共同服务于"低耦合、高内聚、易扩展、易维护"的目标。

## 2.29 Object 类有哪些方法？
| 方法 | 作用 |
|------|------|
| `getClass()` | 返回对象的运行时类（Class 对象），反射入口 |
| `hashCode()` | 返回对象的哈希码，HashMap/HashSet 中用于定位桶 |
| `equals(Object)` | 判断两个对象是否"逻辑相等"，默认比较地址（==），子类通常重写 |
| `clone()` | 创建并返回对象的浅拷贝副本，需实现 Cloneable 接口 |
| `toString()` | 返回对象的字符串表示，默认 `类名@哈希码`，建议重写 |
| `notify()` | 随机唤醒一个在此对象监视器上等待的线程 |
| `notifyAll()` | 唤醒所有在此对象监视器上等待的线程 |
| `wait()` / `wait(long)` / `wait(long, int)` | 让当前线程进入此对象监视器的等待集，释放锁 |
| `finalize()` | 对象被 GC 回收前调用一次，JDK9 已废弃 |

其中 `wait`/`notify`/`notifyAll` 必须在 `synchronized` 块内调用，否则抛 `IllegalMonitorStateException`。`equals`/`hashCode` 必须按契约一起重写。

## 2.30 Java 中如何实现不可变类？
不可变类：实例创建后其状态不可修改。典型例子是 `String`、`Integer`、`LocalDate`。

**实现步骤**：
1. 将类声明为 `final`，防止被继承
2. 所有字段 `private final`，防止外部访问和修改
3. 不提供任何 setter 方法
4. 如果字段是可变引用类型（如 `Date`、集合），**在构造器和 getter 中进行防御性拷贝**——防止外部修改内部状态
5. 确保方法不会被重写（类 final 或方法 final）

```java
public final class ImmutablePerson {
    private final String name;
    private final Date birthday;  // Date 是可变的

    public ImmutablePerson(String name, Date birthday) {
        this.name = name;
        this.birthday = new Date(birthday.getTime());  // 防御性拷贝
    }

    public String getName() { return name; }
    public Date getBirthday() { return new Date(birthday.getTime()); }  // 防御性拷贝

    // 无 setter
}
```

**不可变类的优势**：线程安全（天然同步）、适合做 HashMap 的 key（hashCode 不变）、简单可靠、支持缓存和字符串常量池等。

## 2.31 接口中的 default 方法冲突问题如何解决？
JDK8 引入 default 方法后，一个类可能实现两个包含同名 default 方法的接口，产生冲突。

**三种冲突场景及解决**：
1. **类 vs 接口冲突**：类的方法优先级最高。类或父类中声明的方法优先级高于接口 default 方法。
2. **接口 vs 接口冲突（两个接口无继承关系）**：子类**必须显式重写**冲突方法，否则编译失败。可以用 `接口名.super.方法名()` 选择其中一个实现：
   ```java
   public class C implements A, B {
       @Override
       public void hello() {
           A.super.hello();  // 显式选择接口 A 的 default 实现
       }
   }
   ```
3. **子接口 vs 父接口**：子接口的 default 方法覆盖父接口同名方法（正常的覆盖规则，类似类继承的方法覆盖）。

**核心规则**：类优先 > 子接口 > 父接口。JDK8 增加 default 方法是"优先使用接口继承，避免抽象类"，但多接口的 default 方法冲突带来的"菱形继承"问题需要程序员显式解决（这是 JDK8 之前多实现接口不会有这个问题的根源）。

## 2.32 Java 创建对象有哪几种方式？
1. **new 关键字**：最常见，调用构造器。`User u = new User();`
2. **Class.forName().newInstance() / Class.newInstance()**：通过反射调用无参构造器。JDK9+ `newInstance()` 标记为过时，推荐 `getDeclaredConstructor().newInstance()`。
3. **Constructor.newInstance()**：反射获取指定构造器后调用。可以调用有参构造器。`Constructor<User> c = User.class.getDeclaredConstructor(String.class); User u = c.newInstance("张三");`
4. **clone()**：实现 `Cloneable` 接口，通过 `super.clone()` 创建对象。不调用构造器，直接复制内存中的对象数据。
5. **反序列化**：从字节流/文件/网络重建对象。`ObjectInputStream.readObject()`，同样不调用构造器。
6. **Unsafe.allocateInstance()**：最底层方式，直接分配内存不调用构造器，不初始化字段（零值/null），常用于框架和 ORM。

其中 1-3 调用构造器，4-6 不经过构造器。

## 2.33 int 和 Integer 的区别？为什么要有包装类？
| 维度 | int | Integer |
|------|-----|---------|
| 类型 | 基本数据类型 | 引用类型（包装类） |
| 默认值 | 0 | null |
| 存储位置 | 栈（局部变量）/ 对象内 | 堆（通过引用指向） |
| 内存占用 | 4 字节 | 16 字节（8 字节对象头 + 4 字节 int + 4 字节对齐填充） |
| 泛型支持 | ❌ 不可用于泛型 | ✅ `List<Integer>` |
| null | 不存在，无法表示"无值" | 可以，表示"无值"或"未设置" |

**为什么要有包装类**：
1. **泛型不支持基本类型**：`List<int>` 非法，必须 `List<Integer>`。这是最核心的原因——Java 泛型的类型擦除要求泛型类型参数必须是 Object 子类。
2. **提供工具方法**：`Integer.parseInt()`、`Integer.toHexString()`、`Integer.compare()` 等。
3. **可表示 null**：在某些场景下需要 null 表示"数据不存在"（如数据库可空字段映射、RPC 调用返回值）。
4. **集合框架要求**：Java 集合类存储的是 Object 类型，基本类型不能直接放入。

**选择建议**：局部变量/循环计数用 `int`（性能好、无需拆箱）；PO/VO 字段、集合元素、泛型参数用 `Integer`；需要区分"0 值"和"未设置"的场景用 `Integer`。

## 2.34 类的初始化时机和执行顺序是什么？
**触发类初始化（主动引用）的 6 种情况**：
1. `new` 实例化对象
2. 读取或设置类的 `static` 字段（`static final` 编译期常量除外）
3. 调用类的 `static` 方法
4. 反射调用该类，如 `Class.forName("xxx")`
5. 初始化子类时会先触发父类初始化
6. 含 `main` 方法的启动类（虚拟机入口）

**不会触发初始化（被动引用）的 3 种情况**：
- 子类引用父类的静态字段：`Sub.MAX` 只初始化父类，不初始化 `Sub`
- 定义数组：`A[] a = new A[10]` 不触发 `A` 初始化
- 引用编译期常量：`static final int X = 100` 被内联进常量池，直接使用，不触发初始化

**初始化顺序（有继承）**：
```
父类静态代码块/静态变量 → 子类静态代码块/静态变量
→ 父类实例代码块/实例变量 → 父类构造器
→ 子类实例代码块/实例变量 → 子类构造器
```
记忆口诀：静态先于实例，父类先于子类，变量/代码块按书写顺序，构造器最后。

**Class.forName vs loadClass**：`Class.forName` 会执行类的初始化（静态块执行），典型应用是 JDBC 驱动 `Class.forName("com.mysql.cj.jdbc.Driver")`（静态块里调用 `DriverManager.registerDriver` 注册驱动）。`ClassLoader.loadClass` 只加载不初始化，Spring 懒加载 Bean、CGLIB 延迟代理常利用此特性。

## 2.35 this 和 super 的区别与用法？
| 关键字 | 含义 | 典型用法 |
|--------|------|---------|
| `this` | 引用当前对象 | `this.name` 区分成员变量与局部变量；`this()` 调用本类重载构造器 |
| `super` | 引用父类 | `super.name` 访问父类成员；`super.method()` 调用父类方法；`super()` 调用父类构造器 |

关键规则：
- `this()` / `super()` 必须写在构造器**第一行**，二者只能其一（不能同时出现）
- 子类构造器若不显式写 `super()`，编译器默认在第一行插入无参 `super()`
- `this` 不能用于 `static` 方法（没有当前对象概念）
- `super` 不能跨代：只能访问直接父类，不存在 `super.super`

## 2.36 switch 支持哪些类型？底层原理？
**支持**：`byte`、`short`、`char`、`int` 及其包装类（Byte/Short/Character/Integer）、枚举（JDK5）、`String`（JDK7）。

**不支持**：`long`、`float`、`double`、`boolean`。

**底层编译**：
- `tableswitch`：case 值连续密集时使用，跳转表 O(1) 定位
- `lookupswitch`：case 值稀疏时使用，二分查找
- `String` 的 switch 先比较 `hashCode`（因为有哈希碰撞，命中后再调用 `equals` 二次确认）
- 枚举的 switch 基于枚举 `ordinal` 序号生成映射

## 2.37 类型转换和类型提升的规则？
**类型提升（隐式）**：`byte`/`short`/`char` 参与算术运算自动提升为 `int`：
```java
byte a = 1, b = 2;
byte c = a + b;           // 编译错误，a+b 结果是 int
byte c2 = (byte)(a + b);  // 需强转
```
表达式中只要出现 `long`/`float`/`double`，结果就提升为最高等级的类型。

**复合赋值隐含强转**：`byte c = 1; c += 2;` 合法，因为 `c += 2` 等价于 `c = (byte)(c + 2)`，编译器自动插入强转。

**widening（自动）与 narrowing（强转）**：
- 自动：byte→short→int→long→float→double，char→int
- 强转：可能溢出（如 `(byte) 200 == -56`）、截断小数（`(int) 3.9 == 3`）

## 2.38 位运算符有哪些？常见应用场景？
七个位运算符：`&`(与)、`|`(或)、`^`(异或)、`~`(取反)、`<<`(左移)、`>>`(右移带符号)、`>>>`(无符号右移)。

常见应用：
- `n & 1` 判断奇偶
- `n << 1` 乘以 2，`n >> 1` 除以 2（性能优化写法）
- `a ^ b ^ b == a` 不借助临时变量交换两个数
- HashMap 扰动函数：`(h = key.hashCode()) ^ (h >>> 16)`，让高位参与低位运算
- 权限位掩码：`READ=1<<0, WRITE=1<<1, EXECUTE=1<<2`，`perm & READ` 判断是否有权限
- `>>>` 无符号右移：正数右移高位补 0

注意：移位运算结果至少为 `int`；对 `int` 移位时位数会 `mod 32`。

## 2.39 new String("abc") 创建了几个对象？intern 是什么？
`new String("abc")` 创建 **2 个对象**（前提是常量池中还没有 "abc"）：
1. 字面量 "abc" 放入字符串常量池（JDK7+ 在堆中）
2. `new` 出来的堆上的 String 对象

如果之前常量池已有 "abc"，则只创建 1 个（new 的对象）。

`intern()`：native 方法，把字符串放入常量池并返回池中引用。
- JDK6：常量池在永久代，intern 会**复制**一份到池中（返回副本引用，与堆对象不是同一个）
- JDK7+：常量池移到堆，intern 先查池，无则直接把**当前对象的引用**放入池并返回该引用（不再复制）

经典面试题（JDK7+）：
```java
String s1 = new String("1") + new String("1"); // 堆中"11"，不在常量池
s1.intern();          // 把 s1 引用放入常量池
String s2 = "11";     // 常量池已有，返回 s1 的引用
System.out.println(s1 == s2); // true
```

## 2.40 字符串 + 拼接的底层原理？
- **编译期常量折叠**：纯粹的字面量/常量拼接，编译器直接算出结果并入常量池，如 `"a"+"b"` 编译后就是 `"ab"`
- **变量拼接**：`str += "x"` 编译为 `str = new StringBuilder().append(str).append("x").toString()`，每次循环都会 new 一个 StringBuilder
- **循环拼接**：应使用 StringBuilder（容量可预估的更好），避免反复创建对象；StringBuffer 用于多线程
- **JDK9+**：`+` 拼接改用 `invokedynamic` 调用 `StringConcatFactory` 动态选择拼接策略，减少生成的字节码、提升性能
- `String.concat()`：内部用数组复制，少量拼接可用

## 2.41 Arrays.asList 有哪些坑？
1. **不是 ArrayList**：返回的是 `Arrays` 内部类 `ArrayList`，并非 `java.util.ArrayList`
2. **定长不可变结构**：`add`/`remove` 会抛 `UnsupportedOperationException`（没有重写这些方法）
3. **与原数组联动**：底层就是原数组，修改返回的 List 会同步影响原数组（反之亦然）
4. **原始类型陷阱**：`Arrays.asList(new int[]{1,2,3})` 把 `int[]` 当成单个元素，得到 `List<int[]>`；需要 `Arrays.asList(1,2,3)`（自动装箱）

需要真正可变、可增删的列表用 `new ArrayList<>(Arrays.asList(...))`。

## 2.42 Objects、System.arraycopy 等常用工具方法？
- `Objects.requireNonNull(obj, "msg")`：非空校验，空则抛带自定义消息的 NPE（参数校验）
- `Objects.equals(a, b)`：null 安全比较，内部先 `a == b`，再 `a.equals(b)`
- `Objects.isNull(obj)` / `Objects.nonNull(obj)`：可读性更好的判空
- `System.arraycopy(src, srcPos, dest, destPos, len)`：native 数组复制（浅拷贝），目标数组需提前创建
- `Arrays.copyOf(arr, newLen)`：内部调 arraycopy，自动创建目标数组并返回
- `Math.addExact(a, b)` / `Math.multiplyExact(a, b)`：检测整数溢出，溢出抛 ArithmeticException
- `Objects.hashCode(obj)`：自动处理 null（返回 0），配合重写 hashCode 使用

## 2.43 什么是异常链？异常处理有哪些最佳实践？
**异常链**：catch 捕获异常后抛出新的业务异常时，把原始异常作为 cause 传入，保留根因：
```java
try {
    queryDao();
} catch (SQLException e) {
    throw new BizException("查询失败", e); // e 作为 cause，不丢失堆栈
}
```
好处：外层通过 `getCause()` 追溯真正的根因。

**最佳实践**：
- 捕获具体的异常，不要 `catch (Exception)` 一把梭
- 子类异常在父类异常之前 catch（先小后大，否则编译报错）
- `finally` 中不要 `return` 或抛异常（会覆盖原返回值和原异常）
- 不要用异常控制正常业务流程（性能差、可读性差）
- 业务边界做异常翻译：底层技术异常转为统一业务异常再抛出
- 释放资源优先使用 try-with-resources

## 2.44 什么是桥方法（Bridge Method）？
类型擦除后，编译器为保持多态而自动生成的合成方法。

例如：
```java
class MyCmp implements Comparator<String> {
    public int compare(String a, String b) { ... }
}
```
类型擦除后 `Comparator` 接口的抽象方法变成 `compare(Object, Object)`。如果 MyCmp 没有这个方法，会被判定未实现接口而报错。因此编译器自动生成桥方法：
```java
public int compare(Object a, Object b) {
    return compare((String) a, (String) b); // 强转后调用实际方法
}
```

影响：反射 `getMethods()` 时能看到这些参数含 Object 的"多余"方法，直接调用可能抛 `ClassCastException`。这是理解泛型擦除机制的重要切入点。

## 2.45 匿名内部类访问局部变量为什么要是 final / effectively final？
根本原因是 **Java 的值传递 + 生命周期错位**：
- 匿名内部类对象可能比方法更晚存活（如传给线程池、回调）
- 它对局部变量的访问，实际是**拷贝了一份变量的值**到内部类的成员字段
- 如果允许修改局部变量，方法返回后局部变量已随栈帧销毁，而内部类持有的副本与之不一致，语义混乱

因此 Java 强制要求被内部类访问的局部变量不可变：JDK7 前必须显式 `final`，JDK8+ 改为 effectively final（声明后不再被重新赋值即可）。

与之相反，访问成员变量不需要 final，因为内部类持有外部类对象的引用，通过 `Outer.this.xxx` 访问的是同一份内存。

## 2.46 Java 集合体系结构是怎样的？（速览，完整版见 30_java_collections.md）
两大根接口：
- **Collection**（单列集合，根还继承 Iterable）：
  - **List** 有序可重复：ArrayList（数组）、LinkedList（双向链表）、Vector（过时）、CopyOnWriteArrayList（并发）
  - **Set** 无序不可重复：HashSet、LinkedHashSet、TreeSet、CopyOnWriteArraySet
  - **Queue**：LinkedList、ArrayDeque、PriorityQueue、BlockingQueue 系列（ArrayBlockingQueue/LinkedBlockingQueue/SynchronousQueue/DelayQueue）
- **Map**（双列集合，独立体系，不继承 Collection）：HashMap、LinkedHashMap、TreeMap、Hashtable（过时）、ConcurrentHashMap

## 2.47 ArrayList 和 LinkedList 的区别与适用场景？（速览）
| 维度 | ArrayList | LinkedList |
|------|-----------|-----------|
| 底层 | Object[] 数组 | 双向链表（Node 含 prev/item/next） |
| 随机访问 | O(1) | O(n) |
| 头/中间增删 | O(n)（元素移动） | O(1)（修改指针，已定位节点） |
| 内存 | 连续，尾部有扩容余量 | 每个节点额外存前后指针 |
| 实现接口 | List | List + Deque |

场景：读多写少、按索引访问用 ArrayList（默认首选）；频繁头尾增删、当队列/栈用 LinkedList 或 ArrayDeque（ArrayDeque 不支持随机访问但性能更好）。

## 2.48 HashMap 底层原理？为什么线程不安全？（速览）
JDK8 结构：数组 + 链表 + 红黑树。put 流程：`hashCode` 经高 16 位异或扰动 `(h = key.hashCode()) ^ (h >>> 16)` → `(n-1) & hash` 定位桶 → 空桶直接插入 / 非空遍历链表或红黑树逐 Key 比较（先 `==` 再 `equals`）→ 相同替换 value、不同尾插 → 链表长度 ≥8 且数组 ≥64 树化 → size 超阈值扩容（2 倍）。

设计要点：容量为 2 的幂（`&` 替代取模、扩容迁移只需 `hash & oldCap`）；负载因子 0.75（时间空间折中）。

线程不安全表现：并发 put 数据覆盖、size 不准确、JDK7 头插法扩容形成死链（CPU 100%）。解决：ConcurrentHashMap。

## 2.49 线程安全的集合如何选型？（速览）
| 集合 | 原理 | 适用场景 |
|------|------|---------|
| ConcurrentHashMap | JDK8 CAS + synchronized 锁桶头，粒度细 | 高并发 Map 首选 |
| CopyOnWriteArrayList / CopyOnWriteSet | 写时复制，读无锁 | 读多写少（配置、监听器） |
| Collections.synchronizedList / Map | 全方法加 synchronized 包装 | 低并发、简单场景 |
| BlockingQueue 系列 | 锁 + Condition 阻塞等待 | 生产者-消费者 |
| Hashtable / Vector | JDK1.0 遗留全表 synchronized | 不推荐 |

## 2.50 synchronized 和 ReentrantLock 的区别？（速览，完整版见 31_java_concurrency.md）
| 维度 | synchronized | ReentrantLock |
|------|-------------|--------------|
| 实现 | JVM 内置关键字（Monitor） | JDK API（基于 AQS） |
| 释放 | 自动释放（异常也释放） | 手动 unlock()（finally 中） |
| 可中断 | 不可中断 | lockInterruptibly() 可中断 |
| 超时 | 不支持 | tryLock(time, unit) 可超时 |
| 公平锁 | 非公平 | 支持公平/非公平 |
| 条件变量 | 单条件 wait/notify | 多个 Condition |
| 锁优化 | JDK6 后锁升级优化好 | — |

JDK6 后性能接近。简单场景优先 synchronized；需要可中断、超时、公平、多条件时用 ReentrantLock。

## 2.51 volatile 和 CAS 的作用？（速览）
**volatile**：保证可见性（写刷主存、读取主存，MESI 协议）+ 禁止指令重排（内存屏障）；**不保证原子性**（`i++` 读-改-写仍会出错）。DCL 单例中 `instance` 必须 volatile，防止 `new` 的重排序（分配内存→赋引用先于初始化）导致拿到未初始化对象。

**CAS（Compare And Swap）**：CPU 级原子指令，预期值匹配才更新。JUC 大量使用（AtomicInteger、AQS、ConcurrentHashMap）。缺点：ABA 问题（用 AtomicStampedReference 加版本号解决）、自旋空转开销。

## 2.52 线程池 7 个核心参数和执行流程？（速览）
7 参数：`corePoolSize`、`maximumPoolSize`、`keepAliveTime`、`unit`、`workQueue`、`threadFactory`、`handler`（拒绝策略）。

执行流程：核心线程未满 → 建核心线程；核心满 → 任务入队；队满 → 建非核心线程（未到 max）；到达 max 且队满 → 拒绝策略：
1. AbortPolicy：抛异常（默认）
2. CallerRunsPolicy：调用者线程执行
3. DiscardPolicy：直接丢弃
4. DiscardOldestPolicy：丢弃最老任务重试

不推荐 Executors：Fixed/Single 使用无界队列可能 OOM，Cached 无限建线程可能 OOM，应手动用 ThreadPoolExecutor 指定参数。

## 2.53 ThreadLocal 的原理和内存泄漏？（速览）
原理：每个 `Thread` 内部有一个 `ThreadLocalMap`，以 ThreadLocal 对象为 key 存值，实现线程间变量隔离。

内存泄漏：Map 的 Entry 对 key（ThreadLocal）是**弱引用**。ThreadLocal 失去外部强引用被 GC 回收后，value 是强引用仍被 `Thread → ThreadLocalMap → Entry → value` 链引用，线程池线程长期存活导致 value 无法回收。

解决：使用完必须调用 `remove()`。

## 2.54 BIO、NIO、AIO 的区别？（速览，完整版见 33_java_io_nio.md）
| 模型 | IO 模型 | 线程模型 | 特点 |
|------|---------|---------|------|
| BIO | 同步阻塞 | 一连接一线程 | 简单，连接多时资源耗尽 |
| NIO | 同步非阻塞 | 一线程管 N 连接（Selector） | 高并发，编程复杂 |
| AIO | 异步非阻塞 | OS 完成回调 | 高并发，Java 生态较少用 |

实际业务常用 NIO（Netty 封装的 Reactor 主从模型）。

## 2.55 NIO 的三大核心组件？（速览）
- **Channel（通道）**：双向数据传输，FileChannel / SocketChannel / ServerSocketChannel / DatagramChannel
- **Buffer（缓冲区）**：数据容器，核心属性 capacity/limit/position/mark，`flip()` 切换读写模式
- **Selector（选择器）**：多路复用器，`select()` 阻塞等待就绪事件（OP_ACCEPT / OP_READ / OP_WRITE / OP_CONNECT）

协作流程：Channel 注册到 Selector → `select()` → 遍历 SelectionKey → `Channel.read(Buffer)` → 处理数据 → 继续轮询。
