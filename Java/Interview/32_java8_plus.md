# 1. 知识点
- **Lambda表达式**：匿名函数，(参数) -> { 表达式 }，本质是实现函数式接口的匿名内部类的语法糖，编译后使用invokedynamic指令
- **函数式接口(@FunctionalInterface)**：仅有一个抽象方法的接口（default/static方法不算），注解可选但推荐标注。如Runnable、Callable、Comparator
- **Java内置函数式接口**：Predicate<T>（test判断，返回boolean）、Function<T,R>（apply转换，T入R出）、Consumer<T>（accept消费，void）、Supplier<T>（get提供，无参返回T）、BiFunction<T,U,R>（两个参数）、UnaryOperator<T>（一元操作，入T出T，继承Function）
- **Stream API**：对集合进行函数式操作，不存储数据、不修改源数据、惰性求值、内部迭代
- **创建Stream**：Collection.stream()/parallelStream()、Arrays.stream()、Stream.of()、Stream.iterate()/generate()、IntStream.range()
- **中间操作**：filter(过滤)、map(转换)、flatMap(扁平化)、peek(窥视/调试)、sorted(排序)、distinct(去重)、limit(截取前N)、skip(跳过前N)
- **终止操作**：forEach/forEachOrdered、collect(Collectors收集器)、reduce(归约)、count、findFirst/findAny、anyMatch/allMatch/noneMatch、min/max
- **短路操作**：findFirst、findAny、anyMatch、allMatch、limit，满足条件后不再处理后续元素
- **并行流注意事项**：不是所有操作并行都快（数据量大/Cpu密集才有效）、线程安全问题（使用线程安全的累加器/concurrent收集器）、parallelStream使用ForkJoinPool.commonPool()（慎用，不要在其中执行阻塞操作）
- **Collectors工具类**：toList/toSet/toMap、groupingBy(分组)、partitioningBy(分区，结果为Map<Boolean,List>)、joining(拼接)、counting、summarizingInt(统计)
- **groupingBy多级分组**：groupingBy(一级分类, groupingBy(二级分类))
- **toMap注意事项**：key不能重复（否则抛IllegalStateException），解决：toMap(Function, Function, mergeFunction)；value不能为null
- **Optional**：容器对象，解决NPE。创建：of(非null)/ofNullable(可为null)/empty(空)。取值：orElse(直接返回默认值，无论Optional是否为空都执行)/orElseGet(Supplier懒加载)/orElseThrow(抛异常)。链式：map/flatMap/filter。最佳实践：不用于字段、不用于方法参数、不用于集合元素
- **接口默认方法(default)**：为接口方法提供默认实现，子类可选择重写。原因：接口演进时向后兼容（如Collection新增stream()方法不影响已有实现）
- **接口静态方法**：接口中定义工具方法（类似Collections.sort，JDK8后Comparator.naturalOrder()直接在Comparator接口），不用再写伴生Util类
- **默认方法冲突三规则**：①类优先（类的方法声明的优先级高于接口默认方法）②子接口优先（更具体的接口优先）③手动覆盖（继承多个冲突接口时，o需要手动重写并使用Interface.super.method()指定调哪个）
- **方法引用**：ClassName::staticMethod（静态方法引用）、object::instanceMethod（实例方法引用）、ClassName::instanceMethod（特定类型任意对象的实例方法引用，如String::length）、ClassName::new（构造器引用）
- **新日期API(java.time)**：JDK8引入JSR-310
- **LocalDate/LocalTime/LocalDateTime**：不可变、线程安全，日期/时间/日期时间。now()、of()、parse("2024-01-01")、plusDays/minusDays、isBefore/isAfter
- **Instant**：时间戳（UTC时间点），计算机友好，System.currentTimeMillis()替代
- **ZoneId**：时区，ZoneId.of("Asia/Shanghai")；ZonedDateTime = LocalDateTime + ZoneId
- **DateTimeFormatter**：格式化和解析，线程安全替代SimpleDateFormat。ofPattern("yyyy-MM-dd HH:mm:ss")，format/parse
- **Period**：日期差异（年月日），Duration：时间差异（时分秒纳秒）
- **TemporalAdjusters**：时间调节器，如firstDayOfMonth()、lastDayOfYear()、next(DayOfWeek.MONDAY)
- **重复注解(Repeatable)**：@Repeatable注解，同一位置可重复使用同一注解（JDK8前需用包装注解），如@Scheduled
- **CompletableFuture**：见并发章节，JDK8的异步编程核心
- **JDK9模块化(Jigsaw)**：module-info.java定义模块，exports暴露包，requires依赖其他模块，opens反射开放。实现强封装，内部API不可外部访问
- **JDK10 var**：局部变量类型推断（Java是强类型，var只是语法糖，编译时推断确定类型）。限制：仅局部变量、必须初始化、不能用于方法参数/返回值/字段、Lambda不能用var除非显式声明参数类型、null赋值不行
- **JDK11新特性**：String新增isBlank()/lines()/strip()/stripLeading()/stripTrailing()/repeat(n)；HttpClient标准化(java.net.http)；Files.readString/writeString；Lambda参数支持var；Epsilon No-Op GC（只分配不回收，测试用）
- **JDK14 Records**：透明数据载体类。public record Point(int x, int y){} 自动生成构造器、getter、equals、hashCode、toString。不可变（字段final），不能继承（隐式final），可自定义构造器（紧凑型Canonical）
- **JDK14 instanceof模式匹配**：`if (obj instanceof String s && s.length()>5)` 同时判断并绑定变量，减少强制转换
- **JDK15 Sealed Classes(最终版JDK17)**：sealed class A permits B, C 限制哪些类可以继承。子类需声明为final、sealed或non-sealed（允许无限制扩展）
- **JDK14 Switch表达式(预览, JDK17 LTS)**：箭头语法->和yield返回值；`String result = switch(day) { case MONDAY -> "忙碌"; default -> { yield "休息";} };` 可替代复杂if-else
- **Text Blocks(JDK15)**：三引号`"""..."""`多行字符串，自动去除每行公共前导空格，减少转义。
- **JDK21 Virtual Threads**：Project Loom，轻量级虚拟线程，Thread.ofVirtual().start()，高并发低开销
- **JDK21 Record Patterns**：解构记录，`if (p instanceof Point(int x, int y))` 同时提取x和y
- **JDK21 Pattern Matching for switch**：switch支持模式匹配+when条件守卫
- **JDK21 Sequenced Collections**：新增SequencedCollection/SequencedSet/SequencedMap接口，统一顺序访问（addFirst/addLast/getFirst/getLast/reversed()）
- **JDK21 StringTemplate(预览)**：STR."Hello \{name}" 字符串模板
- **G1收集器**：JDK9默认，Region分区，可预测停顿；JDK12+默认的Shenandoah（Red Hat，低延迟）和ZGC（JDK15生产可用，超低延迟<1ms，着色指针+读屏障+并发）

# 2. 面试题

## 2.1 Lambda表达式本质是什么？如何编译的？
Lambda本质是实现函数式接口的匿名方法的简化语法。编译时不会生成匿名内部类的.class文件，而是使用invokedynamic指令在运行时通过LambdaMetafactory动态生成：

**编译过程**：
1. 编译器将Lambda体编译成所在类的private static方法（Lambda$0）
2. 生成invokedynamic调用点(call site)，运行时由LambdaMetafactory.metafactory()生成对应的函数式接口实现类

与匿名内部类的区别：
- 编译不生成具体的.class文件，运行时动态生成
- Lambda中的this指向外层类实例，匿名内部类中的this指向内部类实例
- Lambda访问外层变量不需要final声明（effectively final即可），匿名内部类需要

## 2.2 Stream API的中间操作和终止操作有哪些？惰性求值怎么理解？
**中间操作（返回Stream，惰性）**：filter(过滤)、map(转换元素)、flatMap(扁平化流)、peek(查看)、sorted(排序)、distinct(去重)、limit(n)截取、skip(n)跳过

**终止操作（产生结果或副作用，触发计算）**：
- 遍历：forEach/forEachOrdered
- 收集：collect(Collectors.toXxx)
- 归约：reduce(初始值, 累加器)、count()、max()/min()
- 匹配：anyMatch/allMatch/noneMatch
- 查找：findFirst()/findAny()

**惰性求值**：中间操作不会立即执行，只是在构建一个处理管道。只有当终止操作被调用时，整个管道才开始执行。这意味着可以一次性遍历处理所有操作，提升效率。如：`list.stream().filter().map().collect()` 只在collect时才开始遍历数据。

## 2.3 并行流parallelStream的使用注意事项？
1. **线程安全问题**：并行流非线程安全的数据结构会出错（如ArrayList.add()并发丢失）。应使用线程安全集合或collect(Collectors.toList())
2. **不是所有场景都快**：数据量小、元素复杂、有阻塞IO时不适用。使用前先测试
3. **自定义ForkJoinPool**：parallelStream默认使用ForkJoinPool.commonPool()（CPU核数-1个线程），可通过`new ForkJoinPool(n).submit(() -> list.parallelStream()...)`使用自定义线程池
4. **避免在并行流中执行阻塞操作**：会耗尽commonPool线程影响其他使用commonPool的代码
5. **依赖顺序的操作不适合**：如limit、findFirst在并行流中可能有性能开销
6. **使用concurrent收集器**：groupingByConcurrent、toConcurrentMap

## 2.4 Optional的使用及其最佳实践？
Optional是一个容器对象，解决null可能引发的NPE。

**创建**：
- Optional.of(value)：value非null，null抛NPE
- Optional.ofNullable(value)：value可为null
- Optional.empty()：空Optional

**取值**：
- orElse(T other)：不管Optional是否为空都会执行构造other（可能有副作用）
- orElseGet(Supplier)：只在Optional为空时才执行Supplier（推荐，延迟求值）
- orElseThrow(Supplier)：直接抛异常

**链式调用**：
- map()：转换内部值
- flatMap()：扁平化（返回的是Optional<Optional<T>>变Optional<T>）
- filter()：条件过滤，不符合返回Optional.empty()

**最佳实践（不建议的使用方式）**：
- 不要将Optional用作类的字段（不是为序列化设计的）
- 不要用作方法参数（增加调用者复杂度，可用方法重载替代）
- 不要将集合包装成Optional（直接用空集合表示无数据）
- 不要用在getter中直接返回Optional（建议内部使用）

## 2.5 接口默认方法为什么引入？冲突怎么解决？
**引入原因**：JDK8在Collection接口中新增stream()方法，如果作为抽象方法，所有ArrayList/HashSet等实现类都要修改编译，向后兼容性灾难。因此引入default方法提供默认实现，让已有接口演进的同时不影响已有实现。

**冲突解决三规则**：
1. **类优先**：如果父类中声明了相同方法（不管是否抽象），接口default方法被忽略
2. **子接口优先**：若一个类实现了多个接口，子接口中重写的default方法优先于父接口的default方法
3. **覆盖调用**：若两个不相关的接口有同名同参的默认方法，未手动重写则编译报错。解决：在实现类中重写该方法，并用`接口名.super.方法名()`调用指定接口的默认方法

## 2.6 新日期API相比旧API有哪些优势？
| 旧API（Date/Calendar） | 新API（java.time） |
|------|------|
| 可变，线程不安全（SimpleDateFormat） | 不可变，线程安全 |
| Date的year从1900开始，month从0开始 | 语义清晰，年份直读 |
| API混乱（Date既表示日期又表示时间） | 职责分离：LocalDate/LocalTime/LocalDateTime |
| 时区处理困难 | ZoneId/ZonedDateTime优雅处理时区 |
| Calendar类复杂难用 | TemporalAdjuster提供便捷日期计算 |
| 格式化串解析线程不安全 | DateTimeFormatter线程安全 |

```java
LocalDate now = LocalDate.now();
LocalDate nextMonday = now.with(TemporalAdjusters.next(DayOfWeek.MONDAY));
DateTimeFormatter fmt = DateTimeFormatter.ofPattern("yyyy-MM-dd");
String formatted = now.format(fmt);
```

## 2.7 Record类是什么？与普通类有什么不同？
Record是JDK14引入的不可变数据载体（透明数据类）：
```java
public record Person(String name, int age) {}
```
自动生成：
- 全参构造器（Canonical Constructor）
- 每个组件的访问器方法（name()、age()，不是getName()）
- equals()、hashCode()、toString()
- 字段隐式为private final

**限制**：
- 不能继承其他类（隐式继承Record类）
- 不能被其他类继承（隐式final）
- 不能声明除组件外的实例字段
- 可以定义紧凑型构造器（compact constructor）：`public Person { if (age < 0) throw ...; }`（省略参数列表和字段赋值）
- 可以实现接口
- 可以有静态字段和方法
- 适合DTO、配置、返回值等纯数据的场景

## 2.8 Sealed Classes密封类是什么？使用场景？
JDK15预览/17正式。通过permits限制可以扩展该类的子类列表：
```java
sealed interface Shape permits Circle, Rectangle, Triangle {}
final class Circle implements Shape {}
non-sealed class Rectangle implements Shape {}  // 允许无限制扩展
record Triangle(double a, double b) implements Shape {} // record隐式final
```

子类必须满足：声明为final、sealed或non-sealed。

**使用场景**：
- 配合switch模式匹配进行完备性检查（编译器知道所有子类，switch覆盖所有case则不需要default）
- 状态机中限制状态数量
- API设计中严格控制继承层次
- 与Record结合构建代数数据类型(ADT)

## 2.9 JDK11有哪些实用的新特性对比JDK8？
1. **String新增方法**：
   - isBlank()：判断是否为空或仅包含空白字符
   - lines()：按行分割返回Stream<String>
   - strip()/stripLeading()/stripTrailing()：去除Unicode空白（trim()只去ASCII空白<=0x20）
   - repeat(n)：重复字符串N次
2. **HttpClient标准化**（java.net.http）：HTTP/2和WebSocket支持、同步/异步编程模型、Builder模式
3. **集合的toArray增强**：`list.toArray(String[]::new)` 替代`list.toArray(new String[0])`
4. **Files新方法**：readString(Path)、writeString(Path, CharSequence)，一行读/写文件
5. **Lambda参数支持var**：`(var x, var y) -> x + y`，可以使用var注解（@Nullable var x）
6. **Epsilon垃圾收集器**：只分配不回收的GC（测试/性能基准用）
7. **Flight Recorder**：低开销的JVM诊断数据采集

## 2.10 JDK21有哪些重要的新特性？
1. **Virtual Threads（正式GA）**：Project Loom，轻量级虚拟线程  
2. **Sequenced Collections**：新增SequencedCollection、SequencedSet、SequencedMap接口，统一支持reversed()、addFirst/addLast/getFirst/getLast  
3. **Record Patterns**：`if (p instanceof Point(int x, int y))` 直接解构record组件  
4. **Pattern Matching for switch**：switch支持模式匹配和条件守卫  
5. **String Templates（预览）**：STR."Hello \{name}，you are \{age} years old"  
6. **Scoped Values（孵化）**：替代ThreadLocal用于虚拟线程场景，更轻量  
7. **Structured Concurrency（预览）**：结构化并发，将子任务作为父任务的一部分  
8. **Unnamed Classes and Instance Main Methods（预览）**：简化初学者Java入门

## 2.11 G1和ZGC的区别和适用场景？
| 特性 | G1 | ZGC |
|------|-----|-----|
| 停顿时间目标 | 可预测的（默认200ms） | 亚毫秒（<1ms，甚至<0.1ms） |
| 暂停时间 | 随堆增大而微弱增加 | 与堆大小无关，始终超低 |
| 最大堆 | TB级 | 16TB |
| 核心机制 | Region分区 + Remembered Set + SATB | 着色指针(Colored Pointer) + 读屏障(Load Barrier) |
| 分代 | 有分代（年轻代/老年代） | 不分代（JDK21支持分代ZGC） |
| 标记阶段 | 并发标记 | 并发标记 |
| 整理阶段 | 混合GC，按需整理Region | 并发整理（转移阶段与应用程序并发） |
| 默认 | JDK9+ | 可选，JDK17+推荐低延迟场景 |
| 适用场景 | 通用服务器，追求吞吐量 | 低延迟关键业务（交易、游戏、金融） |

G1将堆划分为多个Region（1~32MB），优先回收垃圾最多的Region（Garbage First），通过预测模型控制停顿时间。ZGC使用着色指针（在64位指针中嵌入元数据）+ 读屏障自愈（访问到转移中的对象时自动修正）实现并发转移，停顿<1ms。
