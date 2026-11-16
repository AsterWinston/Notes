本篇覆盖设计模式的高频面试题，按"主题块 → 详细分块 → 题目 → 解答"组织。五个主题块：原则与态度、创建型模式、结构型模式、行为型模式、框架与面试综合。Spring 的 IoC/AOP 机制见 7_spring.md（AOP 与事务章节），并发可见性与 volatile 见 1_java.md（并发章节），框架内的责任链实现见 5_tomcat.md、20_sentinel.md（2.2.1）。

# 1. 原则与态度

## 1.1 SOLID 与设计原则

### 1.1.1 什么是 SOLID？每条原则各解决什么问题、怎么用一句话识别违反？

| 原则 | 一句话定义 | 违反的信号 | 落地手段 |
|------|-----------|-----------|---------|
| **SRP 单一职责** | 一个类只对**一个变化原因（一个角色/干系人）**负责 | 类名出现 "And/Manager/Util"、改一个需求要动同一个类的多处、测试要 mock 一堆无关依赖 | 按"谁会要求它变"拆；方法级内聚 |
| **OCP 开闭** | 对扩展开放、对修改关闭 | 每加一个类型就改 `switch/if-else` 链 | 抽象 + 多态 + 注册表/SPI（策略、工厂） |
| **LSP 里氏替换** | 子类必须能**无感替换**父类（行为契约不变） | 子类抛父类没有的异常、空实现/`UnsupportedOperationException`、调用方要 `instanceof` 判断 | 继承只表达"is-a + 行为契约"；用组合替代"为复用而继承" |
| **ISP 接口隔离** | 客户端不应依赖它不用的方法 | 接口几十个方法、实现类一堆空方法、测试要写桩 | 拆"角色接口"（读/写、查询/命令分开） |
| **DIP 依赖倒置** | 高层与低层都依赖**抽象**，细节依赖抽象 | `new` 具体实现、业务层 import 基础设施类 | 面向接口编程 + 构造器注入（IoC/DI） |

**四条补充原则（不属 SOLID 但常一起问）**：**LoD 迪米特**（少知道：只和直接朋友通信，`a.getB().getC().do()` 是坏味道）；**CRP 合成复用**（优先组合，继承是白盒强耦合）；**DRY**（避免重复知识，不是"避免重复代码"——两段长得像但变化原因不同的代码不该合并）；**KISS/YAGNI**（简单优先、别为想象中的需求设计）。

**易错（面试官最爱追问的边界）**：① **SRP 不是"一个类一个方法"**——是按"变化原因"分，过拆会类爆炸（1.1.2）；② **LSP 的经典反例是"正方形继承长方形"**（设置宽高会破坏父类契约）——继承看的是**行为契约**不是"现实世界的 is-a"；③ **OCP 不可能 100%**——需求变化必然要改代码，"对修改关闭"指的是**核心稳定点**的封闭，不是"永远不改"；④ **DIP 不等于"所有类都要接口"**——稳定、无替换需求的实体直接依赖具体类完全合理。

**答题收口**："SOLID 是**降低变更成本的启发式**，不是宗教——每条原则的背面都有一个代价（抽象成本、类数量、调试复杂度），成熟度体现在'知道什么时候不用'（1.1.2）。"

### 1.1.2 设计原则用过头会怎样？"模式驱动"和"演进式设计"怎么平衡？

**过度设计的五种典型病**：
1. **抽象癌**：每个类都配接口 + 抽象类 + 工厂，实际只有一个实现——**间接层不产生价值，只增加跳转成本**（"以后可能要换"是 YAGNI 的反面）；
2. **模式堆砌**：为用模式而用模式（一个 CRUD 塞进策略+命令+访问者），代码像"模式展览馆"，新人无法维护；
3. **过度分层**：Controller→Service→Manager→Helper→Util 五层转发，每层只调下一层（**贫血的转发层**）；
4. **过早通用化**：为"未来多租户/多协议/多存储"设计插件体系，结果需求没来、复杂度先到（**唯一确定的未来是需求会变，但变的方向不确定**）；
5. **牺牲可读性换"扩展性"**：事件总线替代直接调用、反射替代显式依赖——**调试时找不到调用方**是最大的生产事故源。

**平衡的四条准则（面试的成熟度答案）**：
- **Rule of Three（三次法则）**：同一种变化出现三次再抽象——第一次直接写、第二次复制、第三次提炼（避免"两次就抽象"的过度）；
- **演进式设计**：先让代码**可测试、可读**，在真实变化点出现时重构（重构的方向由"痛点"驱动，而不是由"模式目录"驱动）；
- **模式是"命名"不是"目标"**：模式的价值是**让团队用同一个词沟通设计意图**（"这里是策略模式"），而不是"必须凑齐 23 个"；
- **可逆性优先**：不确定时选**最容易改回去**的方案（简单实现 + 清晰边界 > 复杂抽象 + 错误方向）。

**收口句**："**设计模式是'事后总结的词汇'，不是'事前套用的模板'**——先写清楚、再重构出模式；面试里说'我在 X 场景用了策略模式，因为 Y 变化频繁'，比背 23 个定义有说服力得多（5.2.2）。"

## 1.2 原则的落地

### 1.2.1 合成复用 vs 继承怎么选？什么时候必须继承？

**继承的三个代价**：**白盒耦合**（父类实现细节暴露给子类）、**脆弱基类**（父类改一处，所有子类行为可能变）、**单继承限制**（Java 只能继承一个类，且编译期绑定）。

**组合的优势**：黑盒复用（只看接口）、运行期可替换（策略/委托）、职责清晰（一个类只委托一件事）；**代价**是"多写一层转发"（但现代 IDE 与内联优化让这点成本可忽略）。

**"必须/适合继承"的四种场景**：
1. **真正的 is-a + 行为契约稳定**：`ArrayList` 扩展、异常体系（`IOException` 家族）、`InputStream` 家族——**父类的契约就是子类的契约**；
2. **模板方法**：父类固定流程、子类填步骤（`HttpServlet.service`、`AbstractList`）——**继承是设计的一部分**（4.1.3）；
3. **框架扩展点明确要求**：Spring 的 `AbstractController`（历史）、JUnit 的 `TestCase`（历史）——框架让你继承，就继承；
4. **性能/对象布局敏感**（极少见）：避免虚调用/额外对象（现代 JVM 下基本不是理由）。

**判断口诀**："**'是不是'用继承，'有没有/能不能'用组合**"——`Dog is an Animal` 可以继承；`Car has an Engine`、`OrderService uses a PaymentStrategy` 用组合。**"为了复用代码而继承"是坏味道**（比如 `OrderService extends BaseService` 只为拿几个工具方法——应改成委托或工具类）。

**易错**：用继承做"配置"（子类只为改一个常量/行为——用构造参数/策略）；继承 + 重写把父类方法改成完全不同的语义（LSP 违反）；接口的 `default` 方法滥用（Java 8 后接口演化方便了，但**默认方法不是"多继承实现"**，状态仍不能放）。

### 1.2.2 迪米特法则与接口隔离在接口/微服务设计中怎么体现？

**LoD 的现代解读**：不是"不能链式调用"（`stream().filter().map()` 是流畅接口，没问题），而是**不要暴露/依赖对方的内部结构**——`order.getUser().getAddress().getCity()` 让调用方耦合了三级对象图，任何一层结构变化都炸；正解是**让对象自己回答**（`order.getUserCity()`）或**用 DTO 裁剪**（只给调用方需要的字段）。

**在微服务/API 设计中的投影**：
- **接口隔离 → 服务契约拆分**：一个"万能用户服务"暴露 50 个方法，所有调用方都依赖它——按**消费方角色**拆（`UserQueryApi`/`UserCommandApi`，或 BFF 层裁剪），调用方只依赖自己需要的契约（这也是"接口即契约"在 8_spring_mvc.md 的延伸）；
- **LoD → 别把内部模型直接透出**：Entity 直接当 API 返回值=数据库结构变成公共契约（改表就破坏调用方）——DTO 转换是"少知道"的工程化（14_canal.md 3.3.2 的"下游先兼容"纪律同源）；
- **ISP 在消息/事件设计**：一个"大而全"的领域事件（包含所有字段）逼所有消费者解析全部内容——按消费者需要拆事件/用投影（CQRS 的读模型思路，34_distributed_theory.md）。

**易错**：把 LoD 做成"每个类都加一堆 getter 包装"（贫血的转发，反而更耦合）；ISP 拆得太碎（一个方法一个接口，调用方要注入五个接口）；微服务里"共享 DTO jar"（看似复用，实则把版本耦合死——契约版本化更稳）。

# 2. 创建型模式

## 2.1 单例

### 2.1.1 单例有几种实现？DCL 为什么必须加 volatile？

**五种实现对照（按"懒加载/线程安全/防破坏"三维打分）**：

| 实现 | 懒加载 | 线程安全 | 备注 |
|------|--------|---------|------|
| **饿汉式** | ❌（类加载即创建） | ✅（类初始化由 JVM 保证） | 简单；不用也占内存（可忽略） |
| **DCL 双重检查** | ✅ | ✅（**必须 volatile**） | 延迟加载 + 性能折中，写法最容易错 |
| **静态内部类（Holder）** | ✅（首次调用才加载内部类） | ✅（类初始化锁） | **推荐**：无锁、懒加载、代码干净 |
| **枚举** | ❌（类加载即创建） | ✅ | **最安全**（防反射/序列化），2.1.2 展开 |
| 懒汉 + 全方法 synchronized | ✅ | ✅ | 每次调用都加锁，性能差（历史写法） |

**DCL 的标准写法与 volatile 的必要性**：

```java
public class Singleton {
    private static volatile Singleton instance;      // volatile 不能省
    private Singleton() {}
    public static Singleton getInstance() {
        if (instance == null) {                       // 第一次检查：避免每次都加锁
            synchronized (Singleton.class) {
                if (instance == null) {               // 第二次检查：防并发重复创建
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

**`volatile` 防的是"指令重排序"**：`new Singleton()` 在字节码/CPU 层可拆成三步——① 分配内存、② 调用构造器初始化、③ 把引用赋给 `instance`；**② 和 ③ 可能被重排**（先赋值后初始化）。没有 `volatile` 时：线程 A 执行到 ③（`instance != null` 但对象还没初始化完），线程 B 在第一次检查看到非空、直接返回——**拿到一个"半成品对象"**（字段还是默认值）。`volatile` 的写屏障保证"初始化完成之前的写"对读线程可见（happens-before），并禁止 ②③ 重排（1_java.md 的内存模型章节）。

**为什么推荐 Holder**：`getInstance()` 触发 `Holder` 类初始化，JVM 的**类初始化锁**保证只执行一次 `INSTANCE = new Singleton()`，且**天然延迟**（不调用就不加载）——没有显式锁、没有 volatile 心智负担；**它的边界**是"无法防御反射/序列化"（仍需 2.1.2 的手段）。

**易错**：volatile 加在方法上（`public static synchronized` 与 volatile 是两回事）；双检只写一层（并发下创建多个实例）；认为"Spring 单例 = JVM 单例"——**Spring 的 singleton 是"每个容器一个 Bean"**（多个 ApplicationContext 就有多个实例），且**线程安全要自己保证**（无状态 Bean 天然安全，有状态要加锁/用 ThreadLocal）。

### 2.1.2 单例怎么被破坏？怎么防？枚举为什么最安全？

**四种破坏方式**：

| 方式 | 原理 | 枚举是否免疫 |
|------|------|-------------|
| **反射** | `Constructor.setAccessible(true)` 调私有构造器 `newInstance()` | ✅（`Constructor.newInstance` 对枚举直接抛 `IllegalArgumentException`） |
| **序列化/反序列化** | `ObjectInputStream.readObject()` 会**绕过构造器**创建新实例 | ✅（枚举按 name 反查 `Enum.valueOf`，返回同一实例） |
| **克隆** | 实现 `Cloneable` 并暴露 `clone()` | ✅（枚举不可克隆） |
| **多个类加载器** | 同一个类被不同 ClassLoader 加载 = 两个 Class = 两个"单例" | ❌（枚举也挡不住——但这是容器/热部署场景，不是常规攻击） |

**非枚举的防御**：
1. **反射**：构造器里判重（`if (instance != null) throw new IllegalStateException()`）——注意**要在静态初始化完成后才可靠**，且要防"先反射创建再正常调用"的顺序问题（用标志位更稳）；
2. **序列化**：实现 `readResolve()` 返回既有实例：
   ```java
   private Object readResolve() { return getInstance(); }   // 反序列化时用返回值替换新对象
   ```
3. **克隆**：单例类**不要实现 Cloneable**（或 clone 里抛异常）。

**枚举为什么"最安全"（三层原因答全）**：
1. **JVM 保证实例唯一**：枚举常量在类初始化时创建，JLS 规定每个枚举常量只实例化一次；
2. **反射被堵死**：JDK 的 `Constructor.newInstance` 对 `ENUM` 类型有显式检查，直接抛异常——**不是靠"约定"，是语言级**；
3. **序列化被接管**：枚举的序列化只写 `name`，反序列化用 `valueOf` 返回**同一个常量**——不会创建新对象（普通类要靠 `readResolve` 手动补）。

**枚举单例的代价（成熟度）**：不能懒加载（类加载即创建，但单例本来就不该"很重且可能不用"）；不能继承其他类（枚举已继承 `Enum`）；代码可读性上"一个枚举常量当单例"对不熟悉的人是隐晦的。**工程结论**：**"需要绝对防破坏 → 枚举；普通业务单例 → Holder；Spring 环境 → 交给容器（并保证无状态）"**。

## 2.2 工厂与构建

### 2.2.1 简单工厂、工厂方法、抽象工厂的区别？什么时候还需要工厂？

**三者的演进逻辑（从"一个 switch"到"产品族"）**：

| 模式 | 结构 | 扩展新品 | 扩展新产品族 | 典型场景 |
|------|------|---------|-------------|---------|
| **简单工厂**（非 GoF） | 一个工厂类 + `switch(type)` 创建 | **改工厂类**（违反 OCP） | — | 类型少、稳定、内部工具 |
| **工厂方法** | `Factory` 接口 + 每个产品一个具体工厂 | **加工厂类**（符合 OCP） | — | 一个维度扩展（不同支付渠道/不同解析器） |
| **抽象工厂** | 工厂接口创建**一组相关产品**（产品族） | 加产品族（加一个具体工厂） | **加新产品种类要改接口**（较难） | 多维度/成套产品（不同数据库的 Connection/Statement 家族、不同 UI 主题的按钮/输入框） |

**判据**："**一个维度变化 → 工厂方法；多个相关维度成套变化 → 抽象工厂；类型极少且封闭 → 简单工厂甚至直接 new**"。

**现代工程里"还需要工厂吗"（这题的加分点）**：
- **DI 容器（Spring）已经承担了大部分"创建"职责**——`@Bean`/构造器注入让"对象怎么来"从业务代码里消失，**普通业务不需要手写工厂**；
- 工厂仍有价值的场景：**运行期按参数选择实现**（支付渠道、文件解析器、策略实现）——本质是"**策略 + 注册表**"：`Map<String, Strategy>` 注入 + `get(type)`；
- **`FactoryBean`** 是 Spring 的"复杂对象工厂"（创建过程本身是逻辑：代理、SqlSessionFactory 等），与业务工厂不是一回事（7_spring.md）；
- **工厂方法在框架里的影子**：`BeanFactory.getBean`、`Executors.newXxx`、`Calendar.getInstance`、`NumberFormat.getInstance`。

**易错**：简单工厂里 `switch` 每加一个类型都要改（把"创建"和"选择"混在一起——选择逻辑可以留在调用方/注册表）；抽象工厂加"新产品种类"要改所有工厂（**产品族稳定、产品种类变化**才适合）；用工厂掩盖"依赖注入该做的事"（`new` 出来的对象无法被容器管理/测试替换）。

### 2.2.2 Builder、原型、对象池各解决什么？Lombok `@Builder` 有哪些坑？

**Builder（建造者）**：解决**"参数多、可选参数多、需要校验/分步构建、想要不可变对象"**——替代" telescoping constructor（重叠构造器）"与"JavaBean setter（可变、可能半初始化）"：

```java
Order order = Order.builder()
        .userId(42L).items(items).coupon(coupon)   // 链式
        .build();                                   // 在此统一校验/冻结
```

**Lombok `@Builder` 的五个坑**（生产高频）：① **默认值丢失**——字段初始值在 builder 里不生效，要 `@Builder.Default`；② **必填校验缺失**——builder 不强制字段，漏传就是 null，配 `@NonNull`/构造器校验；③ **继承**——父类字段不自动进子类 builder，要 `@SuperBuilder`；④ **Jackson 反序列化**——只有 builder 没有无参构造/Setter 时反序列化失败，要 `@Jacksonized` 或补注解；⑤ **builder 可变/非线程安全**——复用一个 builder 实例会串数据。

**原型（Prototype）**：通过**克隆**创建对象，适合"创建成本高（查库/计算）/结构复杂/需要保留快照"；Java 的实现坑：`Cloneable` 是**标记接口**（没有 `clone` 方法）、`Object.clone()` 是 protected + 抛 `CloneNotSupportedException`、**默认浅拷贝**（引用字段共享）；深拷贝要么手写、要么序列化/JSON（成本高、类型丢失）、要么**拷贝构造器**（推荐，语义清晰）。**Spring 的 prototype 作用域**是"每次 getBean 新实例"，与 GoF 原型不是同一层概念（容器创建 vs 对象克隆）。

**对象池（不是 GoF，但常考）**：复用**创建昂贵/有状态且需独占**的资源——连接池（HikariCP）、线程池（ThreadPoolExecutor）、Netty ByteBuf 池；核心参数：**最大容量/最小空闲/借出超时/空闲回收/健康检查/泄漏检测**；**不要池化"创建便宜"的对象**（GC 比池化更高效，且池化引入泄漏与并发风险）。**与享元的分界**：**池化对象是"有状态、借出独占、归还复用"；享元是"无状态/不可变、所有人共享同一份"**（3.2.2）。

# 3. 结构型模式

## 3.1 代理与装饰

### 3.1.1 静态代理、JDK 动态代理、CGLIB 的原理与选型？Spring AOP 用的是哪种？

**三者对比**：

| 方式 | 原理 | 约束 | 性能 |
|------|------|------|------|
| **静态代理** | 手写代理类，持有目标引用，方法里前后增强 | 每个接口/类都要手写；目标方法增减要改代理 | 最高（无反射） |
| **JDK 动态代理** | `Proxy.newProxyInstance(loader, interfaces, handler)` 运行期生成 `$ProxyN`（**实现目标接口**），方法调用进 `InvocationHandler.invoke`（反射） | **只能代理接口**（目标必须有接口） | 反射调用有开销（现代 JVM 已优化） |
| **CGLIB** | 用 ASM 生成**目标类的子类**，方法调用进 `MethodInterceptor.intercept`（可 `MethodProxy` 直接调，比反射快） | **不能代理 final 类/final 方法/private 方法**；构造器/初始化要小心 | 生成类较慢（可缓存），调用较快 |

**Spring AOP 的选择（版本差异要答准）**：**Spring Framework 默认**——目标有接口用 JDK 代理，没有接口用 CGLIB；**Spring Boot 2.x 起默认 `proxy-target-class=true`（统一 CGLIB）**——原因是"面向接口编程在业务代码里并不普遍 + CGLIB 能代理类更省心"；可用 `spring.aop.proxy-target-class=false` 切回"有接口用 JDK"。**Spring 5.2+** 对 CGLIB 做了内置（不再依赖外部 cglib 包）。

**三个必考坑（7_spring.md 的同源）**：
1. **同类自调用不经过代理**——`this.method()` 绕开代理对象，`@Transactional/@Cacheable/@Async` 全部失效（"事务不生效"的第一大原因）；解法：拆类、注入自身、`AopContext.currentProxy()`（需 `exposeProxy=true`）；
2. **final/private 方法不被 CGLIB 代理**（子类无法重写）；**JDK 代理只能调接口方法**（实现类特有方法不可见）；
3. **代理对象的 `equals/hashCode/toString`** 与目标对象语义可能不一致（缓存 key/日志排查时注意）。

**追问"为什么 JDK 代理必须基于接口"**：生成的 `$Proxy` 已经 `extends Proxy`，Java 单继承——**只能再 implements 接口**来"长得像目标"；CGLIB 用继承，所以能代理类、但不能碰 final。

### 3.1.2 代理、装饰器、适配器、外观怎么区分？

**四种模式的"意图"对照（结构相似，动机不同）**：

| 模式 | 意图（一句话） | 接口关系 | 典型例子 |
|------|--------------|---------|---------|
| **代理 Proxy** | **控制访问**（权限/延迟/远程/日志/事务） | 与目标**同接口**，调用方"以为在调目标" | Spring AOP、RPC Stub、`@Transactional`、图片懒加载 |
| **装饰器 Decorator** | **动态叠加功能**（可多层组合） | 与目标**同接口**，持有目标并增强 | `BufferedReader(new FileReader(...))`、`Collections.unmodifiableList`、HTTP 请求包装 |
| **适配器 Adapter** | **接口转换**（让不兼容的双方合作） | 实现目标接口，**持有/继承被适配者** | `InputStreamReader`（字节→字符）、`HandlerAdapter`（Controller→统一调用）、SLF4J 绑定 |
| **外观 Facade** | **简化子系统**（一个入口包住多个组件） | **新接口**，内部调多个子系统 | SLF4J 门面、`JdbcTemplate`（简化 JDBC 样板）、启动脚本/服务门面 |

**两组最容易混的辨析**：
- **代理 vs 装饰器**：结构几乎一样（同接口 + 持有目标），**区别在"目的"与"是否可组合"**——代理通常**一个**、由框架生成、控制访问；装饰器**可以套多层**、由调用方组合、每层加一个职责（IO 流是教科书：`BufferedInputStream(new GZIPInputStream(new FileInputStream(...)))`）；
- **外观 vs 适配器**：外观是"**简化**"（子系统本来就可用，只是复杂），适配器是"**转换**"（接口本来不兼容）；外观不改变接口语义，适配器改变调用形态。

**答题句**："**看意图不看结构**——同接口 + 控制访问 = 代理；同接口 + 叠加增强 = 装饰；换接口 = 适配；包住多个 = 外观。"

## 3.2 组合与共享

### 3.2.1 桥接模式解决什么问题？JDBC 和日志门面怎么体现？

**要解决的问题：多维度变化导致的类爆炸**——"形状（圆/方）× 颜色（红/蓝）"用继承要 4 个类，加一个维度就乘一倍；桥接把**抽象维度**与**实现维度**分开，用**组合**连接：`Shape` 持有 `Color`，两个维度各自扩展（M+N 而不是 M×N）。

```java
interface Color { String fill(); }
class Red implements Color { ... }
abstract class Shape { protected Color color; Shape(Color c){this.color=c;} abstract void draw(); }
class Circle extends Shape { void draw(){ /* 用 color.fill() */ } }
```

**JDBC 的桥接（经典答法）**：**抽象维度** = `java.sql` 的 `Connection/Statement/ResultSet` 接口（应用只面向它编程）；**实现维度** = 各数据库厂商的 Driver 实现；`DriverManager`（以及 `DataSource`）是"桥"——**换数据库只换驱动，应用代码不变**。严格说 JDBC 是"桥接 + 抽象工厂/工厂方法"的混合（驱动创建连接家族），答题点出"**两个维度独立变化**"即可。

**日志门面的准确说法（易混点）**：**SLF4J 是"外观/门面"（Facade）**——给应用一个统一 API，屏蔽 Logback/Log4j2 等实现；**"jul-to-slf4j / log4j-over-slf4j" 这类桥接包是"适配器"**（把旧 API 的调用转到 SLF4J）；**SLF4J 与绑定实现之间**的关系可类比桥接（API 与实现分离），但面试更稳的说法是"**门面 + 适配器**"——别把"桥接"当成万能标签。

**易错**：把桥接与策略混（策略是"选算法"，桥接是"两个维度独立扩展"）；桥接滥用（只有一个维度变化时就是过度设计）；JDBC 说成"工厂模式"（只说对一半，核心是抽象与实现分离）。

### 3.2.2 组合、享元、外观的应用与边界？

**组合（Composite）**：把对象组织成**树**，让"叶子"与"容器"实现同一接口，客户端**一致对待**——文件系统（文件/目录）、UI 组件树、表达式树、组织架构；Spring 的 `CompositeCacheManager`/`CompositePropertySource`（多个同类型组件聚合成一个）。**边界**：递归深度（栈溢出）、循环引用、容器与叶子语义差异过大时（强塞同一接口会违反 LSP）。

**享元（Flyweight）**：把对象状态拆成**内在状态**（不变、可共享）与**外在状态**（随场景变、由客户端传入），**共享内在状态**省内存——`Integer.valueOf(-128~127)` 缓存、`String` 常量池、`Boolean.TRUE/FALSE`、字符/字形对象。**Java 的坑**：**自动装箱的 `==`**——`Integer a=127,b=127; a==b` 为 true（缓存），`128` 为 false（新对象），比较必须 `equals`；缓存范围可用 `-XX:AutoBoxCacheMax` 调（上限受 `Integer` 缓存数组实现约束）。

**享元 vs 对象池（高频混淆）**：**享元对象不可变、可同时被所有调用方共享**（`Integer` 的 100 就是 100）；**池化对象有状态、借出期间独占**（一个数据库连接同一时刻只给一个线程）——"连接池/线程池是享元"是**常见错误**，它们是**对象池**（2.2.2）。

**外观（Facade）**：给复杂子系统一个**简单入口**（SLF4J、`JdbcTemplate` 对 JDBC 样板的封装、启动引导类）；**边界**：外观容易长成"上帝类"（什么方法都往里塞），应只做**编排**、不承载业务规则；子系统仍可被直接使用（外观不是"禁止访问"）。

**收口**："**组合管'结构'，享元管'内存'，外观管'入口'**——三者都在'把复杂藏起来'，但藏的东西不同：树的形状、对象的数量、子系统的细节。"

# 4. 行为型模式

## 4.1 解耦与流程

### 4.1.1 观察者模式与发布订阅？Spring 事件机制的同步/异步与坑？

**观察者 vs 发布订阅**：观察者是**对象直接持有观察者列表**并通知（Subject 知道 Observer）；发布订阅引入**事件通道/总线**（发布者与订阅者互不知道，靠事件类型解耦）——Spring 的 `ApplicationEventPublisher` 更接近后者（容器充当通道）。

**Spring 事件的四件套**：
```java
// 1) 事件：Spring 4.2+ 可以是任意对象（不必继承 ApplicationEvent）
public record OrderPaidEvent(Long orderId, BigDecimal amount) {}

// 2) 发布：注入 ApplicationEventPublisher
publisher.publishEvent(new OrderPaidEvent(orderId, amount));

// 3) 监听：注解或实现接口
@EventListener
public void onPaid(OrderPaidEvent e) { ... }

// 4) 异步/事务阶段
@Async @EventListener                      // 需 @EnableAsync + 线程池
@TransactionalEventListener(phase = AFTER_COMMIT)   // 事务提交后才执行
public void afterCommit(OrderPaidEvent e) { ... }
```

**默认行为与四个坑（这题的含金量）**：
1. **默认同步**：监听器在**发布者线程**里顺序执行（`SimpleApplicationEventMulticaster`）——**任何一个监听器抛异常/变慢，都会影响发布方**（甚至回滚主事务）；监听器里做远程调用=把主链路 RT 拉长（用 `@Async` 或 MQ 解耦）；
2. **`@Async` 的边界**：异常默认被吞（要配 `AsyncUncaughtExceptionHandler`）、线程池打满会丢/阻塞、事务上下文不传播（异步线程看不到未提交数据）；
3. **事务阶段**：`@TransactionalEventListener` 默认 `AFTER_COMMIT` 之外还有 `BEFORE_COMMIT/AFTER_ROLLBACK/AFTER_COMPLETION`——**发消息/发邮件/调第三方等"不可回滚的副作用"必须 `AFTER_COMMIT`**，否则"事务回滚了消息已发"（34_distributed_theory.md 的本地消息表/事务消息正是这个问题的重型解）；
4. **事件不是可靠消息**：进程崩溃/异步队列满=事件丢——**要可靠投递请上 MQ**（13_rocketmq.md），Spring 事件只做"进程内解耦"（7_spring.md 的事件章节同款结论）。

**顺序与递归**：监听器可用 `@Order` 排序；事件里再发事件要防**循环触发/事件风暴**（加"已处理"标记或改同步调用）；监听器数量多时考虑按类型拆分/异步化。

**易错**：把 Spring 事件当"分布式事件总线"（只在单 JVM 内）；在监听器里修改发布方还在用的对象（共享可变状态）；`@Async` 与 `@Transactional` 同时标在监听器上（异步线程无事务上下文，`@Transactional` 开新事务但看不到原事务数据）；事件类用可变对象（异步执行时已被后续修改）。

### 4.1.2 责任链模式在框架里怎么实现？写一条链要注意什么？

**结构**：`Handler` 接口（`handle(ctx)` + 可选的 `next`），每个处理器决定"处理/放行/短路"——**请求者与处理者解耦、链可动态编排**。

**四个框架实例（答出机制而非名字）**：
- **Servlet Filter**：`FilterChain.doFilter()` 递归调用下一个 Filter，`web.xml`/注解决定顺序（5_tomcat.md 的请求链路）；
- **Spring `HandlerInterceptor`**：`preHandle`（返回 false 短路）→ Controller → `postHandle` → `afterCompletion`（**异常也会走**，用于清理 ThreadLocal）；顺序由 `@Order`/注册顺序决定（8_spring_mvc.md）；
- **Netty `ChannelPipeline`**：inbound 从 head→tail、outbound 从 tail→head，**双向链**（`ChannelHandlerContext.fireChannelRead` 传播）；可动态 add/remove handler；
- **Sentinel `SlotChain`**：NodeSelector→ClusterBuilder→Statistic→Authority→System→Flow→Degrade，**顺序即语义**（20_sentinel.md 2.2.1）；
- **网关 `GatewayFilter`**：路由级 + 全局过滤器按 order 合并（19_spring_cloud_gateway.md 2.2.1）。

**写链的六条纪律**：① **顺序显式化**（`@Order`/order 字段，别依赖 Bean 注册顺序的偶然性）；② **短路要明确**（返回 false/直接写响应后不再传递）；③ **异常与清理**（`afterCompletion`/`finally` 里释放 ThreadLocal、连接、MDC）；④ **不要在链里做重活**（链上每个请求都执行，链路越长延迟越明显）；⑤ **上下文传递**（用 Context 对象而非 ThreadLocal 满天飞，便于异步/测试）；⑥ **动态增删的并发安全**（Netty pipeline 的 add/remove 要在 EventLoop 线程或加锁）。

**易错**：链中某处理器"忘了调用 next"（请求静默丢失）；异常被吞导致 afterCompletion 不执行（资源泄漏）；拦截器里抛异常但没统一处理（返回容器默认错误页）；把"业务分支"写成巨型链（链适合横切，不适合业务流程编排——那是状态机/工作流的领域，4.1.3）。

### 4.1.3 模板方法、策略、状态怎么区分？各自适合什么？

| 维度 | 模板方法 | 策略 | 状态 |
|------|---------|------|------|
| 复用方式 | **继承**（父类定骨架，子类填步骤） | **组合**（Context 持有 Strategy） | **组合**（Context 持有 State） |
| 谁决定"用哪个" | 父类固定流程（好莱坞原则：别找我们，我们会找你） | **客户端/工厂**选择算法 | **对象内部**随状态迁移自动切换 |
| 变化点 | 算法步骤的实现 | 算法族可互换 | 对象在不同状态下的行为不同 |
| 典型 | `JdbcTemplate`、`AbstractApplicationContext.refresh()`、`HttpServlet.service()`、`RedisTemplate` | 支付渠道、折扣计算、`Comparator`、`HandlerMapping` | 订单状态机、TCP 连接状态、电梯、工作流 |
| 关键区别 | 流程**不变**、步骤可变 | 算法**可替换**、调用方选 | 行为随**内部状态**变、状态间有转移规则 |

**判据三问**：① 流程骨架固定、只有个别步骤不同 → **模板方法**；② 同一动作有多种可互换实现、由外部选 → **策略**；③ 对象行为随自身状态变化、且有明确的转移条件 → **状态**。

**策略 vs 状态的经典混淆**：两者类图几乎一样（Context + 接口 + 多实现），**区别在"切换权"与"转移语义"**——策略是"我选一个算法用完就完"（策略之间无关系）；状态是"我处于某状态，做完动作后**自动**进入下一状态"（状态对象知道下一步该是谁）。**订单状态**用状态模式/状态机（避免 `switch(status)` 散落各处），**支付渠道**用策略。

**模板方法的现代替代**：**回调/函数式接口**（`JdbcTemplate.query(sql, rowMapper)`）比"继承抽象类"更轻——Spring 大量用"模板 + 回调"而非纯继承；面试答"模板方法的骨架 + 回调实现"即贴近现代框架。

## 4.2 行为扩展

### 4.2.1 策略模式怎么落地才不像"为了模式而模式"？

**从 if-else 到策略的演进（真实重构路径）**：

```java
// 坏味道：每加一个渠道就改这里，且越来越长
if ("ALIPAY".equals(type)) { ... }
else if ("WECHAT".equals(type)) { ... }
else if ("UNIONPAY".equals(type)) { ... }

// 策略 + 注册表（Spring 注入 Map：key=Bean 名）
public interface PayChannel { String code(); PayResult pay(PayCmd cmd); }

@Component("ALIPAY")
class AlipayChannel implements PayChannel { ... }

@Service
class PayService {
    private final Map<String, PayChannel> channels;   // 构造器注入，Spring 自动按 Bean 名聚合
    PayService(List<PayChannel> list) {
        this.channels = list.stream().collect(toMap(PayChannel::code, identity()));
    }
    PayResult pay(String type, PayCmd cmd) {
        PayChannel c = channels.get(type);
        if (c == null) throw new BizException("不支持的支付渠道: " + type);
        return c.pay(cmd);
    }
}
```

**五个落地要点**：
1. **策略必须无状态**（单例 Bean 被并发调用——有状态就加参数/上下文对象，别放成员变量）；
2. **选择逻辑仍要存在**（注册表 key、枚举校验、默认策略）——策略消灭的是"算法分支"，不是"选择分支"；
3. **小策略用枚举/函数式**（`Map<Type, Function<Cmd, Result>>`）比"一个类一个文件"更轻；
4. **注册冲突要暴露**（`toMap` 遇到重复 key 直接抛异常，启动期发现而不是运行期覆盖）；
5. **事务/幂等边界**在策略外层统一（别让每个策略自己开事务/做幂等，5.1.1 的幂等键在服务层）。

**策略 + 工厂的关系**：工厂负责"创建/查找策略"，策略负责"执行算法"——**注册表本身就是轻量工厂**；不必为了"模式齐全"再套一层 `StrategyFactory`。

**易错**：用 `@Qualifier` 字符串到处选 Bean（key 散落，改成注册表集中）；策略类里依赖 `ApplicationContext` 反查 Bean（服务定位器反模式，难测试）；策略过多（几十个渠道/规则）→ 考虑规则引擎/配置化（但不是所有场景都值得，1.1.2 的过度设计）。

### 4.2.2 命令、访问者、迭代器、备忘录各解决什么？什么时候不该用？

**命令（Command）**：把"请求"封装成对象——支持**参数化调用、排队、记录日志、撤销**；`Runnable`/`Callable` 是最朴素的命令；Spring 的 `JdbcTemplate` 回调更像"模板 + 命令"；业务里"下单命令/退款命令"配合 CQRS 的写模型（34_distributed_theory.md）。**不该用**：只是"调个方法"却包三层（命令对象爆炸）；没有撤销/排队/审计需求时，直接调用更清晰。

**访问者（Visitor）**：**数据结构稳定、操作经常增加**时，把"操作"从"元素类"里拿出来（双分派：`element.accept(visitor)` → `visitor.visit(concreteElement)`）；JDK 的 `FileVisitor`、ASM 的 `ClassVisitor`、`javax.lang.model` 注解处理都是它。**不该用**：元素类型经常增加（每加一种元素要改所有 Visitor，违反 OCP 的方向反了）；**现代替代**：Java 17+ 的 **sealed interface + switch 模式匹配**让"封闭类型集 + 穷尽分支"更直接（编译器检查完整性），很多场景不必再手写 Visitor。

**迭代器（Iterator）**：把"遍历"与"集合内部结构"解耦；Java 的 `Iterator`/增强 for；**fail-fast**（`ArrayList` 的 `modCount` 检查，遍历中修改抛 `ConcurrentModificationException`）vs **fail-safe**（`CopyOnWriteArrayList` 的弱一致快照）；**内部迭代**（`Stream` 的 `forEach/map`）把控制权交给库，便于并行/惰性求值。**不该用**：自己写集合时忘记实现/维护迭代器语义（并发修改、`remove`）；用迭代器做业务过滤（用 Stream/查询）。

**备忘录（Memento）**：在不破坏封装的前提下**保存/恢复对象状态**（撤销、快照、事务回滚的简化版）；Caretaker 管理快照列表；代价是**内存**（大对象/高频快照）与**深拷贝正确性**。**不该用**：状态巨大且变化频繁（用事件溯源/数据库版本/命令日志替代）；把内部字段全部暴露给外部（破坏封装）。

**其余行为模式一句话**：中介者（Mediator，聊天室/消息总线，降低网状依赖）；解释器（Interpreter，规则/表达式引擎，手写少、多用现成引擎）；空对象（Null Object，消除 null 判断，现代用 `Optional`）。

# 5. 框架与面试综合

## 5.1 框架中的模式

### 5.1.1 Spring 用了哪些设计模式？IoC/AOP 各对应什么？

| 模式 | Spring 中的落点 | 机制要点 |
|------|----------------|---------|
| **单例** | Bean 默认 `singleton` 作用域 | **每容器一个**（不是 JVM 全局），线程安全自负 |
| **工厂** | `BeanFactory`/`ApplicationContext`、`FactoryBean`、`@Bean` | 容器=对象工厂 + 注册表；`FactoryBean` 用于复杂创建（代理、SqlSessionFactory） |
| **建造者** | `BeanDefinitionBuilder`、`UriComponentsBuilder` | 分步构建复杂对象 |
| **原型** | `prototype` 作用域 | 每次 `getBean` 新实例 |
| **代理** | **AOP**（JDK/CGLIB）、`@Transactional`、`@Async`、`@Cacheable` | 运行期生成代理，横切逻辑（3.1.1） |
| **装饰器** | `HttpServletRequestWrapper`、`ServerHttpRequestDecorator`、`TransactionAwareCacheDecorator` | 同接口包装增强 |
| **适配器** | `HandlerAdapter`（多种 Controller 统一调用）、`AdvisorAdapter` | 接口转换（8_spring_mvc.md） |
| **模板方法** | `JdbcTemplate`/`RedisTemplate`/`RestTemplate`/`TransactionTemplate`、`AbstractApplicationContext.refresh()` | 骨架 + 回调/抽象步骤（4.1.3） |
| **策略** | `Resource`、`HandlerMapping`、`ViewResolver`、`PlatformTransactionManager` | 同接口多实现，运行期选择 |
| **观察者** | `ApplicationEventPublisher` + `@EventListener` | 容器内事件广播（4.1.1） |
| **责任链** | Filter、`HandlerInterceptor`、AOP Advisor 排序、异常处理器链 | 顺序执行/短路（4.1.2） |
| **桥接** | JDBC `DataSource`/驱动、日志门面 | 抽象与实现分离（3.2.1） |
| **组合** | `CompositeCacheManager`、`CompositePropertySource` | 树形聚合 |

**IoC/AOP 的"模式本质"**：**IoC/DI 是 DIP 的容器化实现**（控制反转=对象创建权交给容器，依赖注入=容器把依赖"推"进来）——它本身不是 GoF 23 之一，但**让工厂/单例/策略的很多手写代码消失**；**AOP 是代理模式 + 责任链**（多个 Advisor 组成拦截器链）的框架化。

**答题收口**："Spring 不是'用了 23 个模式'，而是**用少数几个模式（工厂/单例/代理/模板/策略/观察者）搭了一个容器**——理解容器机制比背模式清单重要（7_spring.md 的 IoC/AOP 主线）。"

### 5.1.2 设计模式在 JDK、MyBatis、Netty 里各有哪些影子？

**JDK（背 6 个即可）**：`Runtime`（单例）、`Executors`/`Calendar.getInstance`（工厂）、`InputStream` 家族（装饰器）、`Proxy`（动态代理）、`Collections.unmodifiableList`（装饰器）、`Comparator`（策略）、`Runnable`/`Callable`（命令）、`Iterator`（迭代器）、`StringBuilder`（建造者）、`try-with-resources`（模板方法式的资源管理）。

**MyBatis（框架题高频）**：`SqlSessionFactoryBuilder`（建造者）→ `SqlSessionFactory`（工厂）；`MapperProxy`（**JDK 动态代理**，Mapper 接口无实现类的秘密）；`BaseExecutor`（**模板方法**，一级缓存/事务骨架固定，query/update 由子类实现）；`TypeHandler`（**策略**）；`Interceptor` 链（**责任链 + 代理**，插件机制）；`PerpetualCache` 外套 `LruCache`/`LoggingCache`（**装饰器**）；动态 SQL 的 `SqlNode` 树（**组合/解释器**）。

**Netty**：`Bootstrap/ServerBootstrap`（建造者）、`ChannelPipeline`（**责任链**，inbound/outbound 双向）、`ChannelHandlerAdapter`（**适配器**，空实现便于选择方法）、`ByteBufAllocator`（工厂 + 池化）、`EventLoop`（Reactor 模式，不在 GoF 23 内但必须知道）。

**为什么面试爱问"框架里的模式"**：它证明你**读过源码/理解框架扩展点**——比如知道 `HandlerAdapter` 才能理解"为什么加一种 Controller 类型不用改 DispatcherServlet"（OCP），知道 `MapperProxy` 才能解释"MyBatis 接口为什么不用实现类"。

## 5.2 面试综合

### 5.2.1 手写单例/责任链/策略时，怎么写才像生产代码？

**单例（Holder 版，生产推荐）**：
```java
public final class ConfigHolder {                 // final：防继承破坏
    private ConfigHolder() {}                     // 私有构造
    private static class Holder {
        static final ConfigHolder INSTANCE = new ConfigHolder();
    }
    public static ConfigHolder getInstance() { return Holder.INSTANCE; }
}
```
加分点：`final`、私有构造、无状态/线程安全说明、序列化场景补 `readResolve`（2.1.2）。

**责任链（接口 + 有序列表，避免递归过深）**：
```java
public interface OrderHandler {
    /** 返回 true 继续，false 短路 */
    boolean handle(OrderContext ctx);
}

@Component
public class OrderPipeline {
    private final List<OrderHandler> handlers;    // Spring 按 @Order 注入
    public OrderPipeline(List<OrderHandler> handlers) { this.handlers = handlers; }
    public void execute(OrderContext ctx) {
        for (OrderHandler h : handlers) {
            if (!h.handle(ctx)) return;           // 显式短路
        }
    }
}
```
加分点：显式顺序（`@Order`）、短路语义、上下文对象、异常统一处理、单测可逐段验证。

**策略（注册表 + 无状态）**：见 4.2.1 的 `Map<String, PayChannel>`——加分点在**重复 key 检测、默认策略、未知类型的明确异常、策略无状态**。

**"生产代码"的四个标志**（面试官真正看的）：**边界处理**（null/空集合/未知类型）、**线程安全**（单例/策略无状态）、**可测试**（依赖注入、无隐藏全局状态）、**可观测**（日志/异常带上下文）——而不只是"能跑"。

**易错**：单例里持有可变状态（并发 bug）；责任链用递归调用（链长时栈溢出，用循环）；策略用 `static` 工具类（无法注入依赖/测试）；手写代码里 `System.out`/吞异常（生产事故源）。

### 5.2.2 "你项目里用过什么设计模式？"怎么答才有说服力？

**错误答法**："用过单例、工厂、策略、观察者……"（像背目录）——面试官下一句"具体解决什么问题/不用会怎样"就哑火。

**正确结构：痛点 → 选择 → 权衡 → 结果（STAR 变体）**，示例：

> **背景**：支付渠道从 2 个涨到 8 个，`PayService` 里一个 300 行的 `if-else`，每次加渠道要改核心类、回归全量支付用例。
> **动作**：抽出 `PayChannel` 接口（`code()/pay()`），每个渠道一个无状态 Bean；用 Spring 注入 `List<PayChannel>` 聚合成 `Map<code, channel>`；未知渠道统一抛业务异常；渠道配置（开关/限额）走配置中心。
> **权衡**：没有上抽象工厂/规则引擎——渠道维度单一、变化可预期，策略 + 注册表足够；代价是"选择逻辑"仍在服务层（但已收敛到一处）。
> **结果**：新增渠道=新增一个类 + 配置，核心服务零修改；补了参数化测试覆盖各渠道；线上切换用开关灰度。

**追问预案（提前想好）**：为什么不继续 if-else？（可维护性/开闭/测试隔离，且分支还在增长）；为什么不用枚举 switch？（行为复杂、依赖多、单测难）；策略怎么选？（注册表 key，来自请求/配置，未知即失败）；事务和幂等放哪？（服务层统一，策略只做算法，4.2.1）。

**加分姿态**：主动讲**一次过度设计的教训**（"我们曾为'未来多存储'抽象了四层，结果只接了一种，后来重构删掉"）——显示你有**判断力**而非只会套模式（1.1.2）；再补一句"**模式是沟通词汇，不是 KPI**"，收尾即成熟。