# 1. 知识点
- **七大设计原则**：开闭原则(OCP:对扩展开放/对修改关闭)、里氏替换原则(LSP:子类可完全替换父类使用)、依赖倒置原则(DIP:依赖抽象而非具体实现)、单一职责原则(SRP:一个类只负责一项职责)、接口隔离原则(ISP:接口精简化/不强迫实现多余方法)、迪米特法则(LoD/最少知道:减少类之间依赖)、合成复用原则(CRP:优先组合而非继承)
- **开闭原则(OCP)**：Open for Extension, Closed for Modification。通过抽象接口扩展新功能而无需修改原有代码。Spring的AOP/扩展点、Java SPI都是OCP的体现
- **里氏替换(LSP)**：子类替换父类使用时行为不发生错误。要求：子类不改变父类方法原有含义、不抛出父类未声明的异常、前置条件不能强化/后置条件不能弱化
- **依赖倒置(DIP)**：High-level modules should not depend on low-level modules. Both should depend on abstractions. 具体依赖抽象。Spring的IoC/DI就是DIP的实践
- **单一职责(SRP)**：一个类只负责一个功能领域。好处：高内聚低耦合、可读性好、易维护。拆分粒度要适中（过细造成类爆炸）
- **接口隔离(ISP)**：使用多个精专的接口优于使用单一臃肿接口。客户端不应该被迫依赖它不需要的接口。如Runnable单个抽象方法vs大的UserService
- **迪米特法则(LoD)**：一个对象对另一个对象知道得越少越好。只和直接朋友交流，不和不认识的对象通信。反面：getA().getB().getC().doSomething()（链式调用）
- **合成复用(CRP)**：优先使用组合/聚合，其次使用继承。继承是强耦合（白盒复用，父类改变子类受影响），组合是弱耦合（黑盒复用）
- **单例模式**：保证一个类只有一个实例并提供全局访问点。实现方式：饿汉式(类加载创建/线程安全)、懒汉式DCL(双重检查+volatile)、静态内部类(Holder模式利用类加载机制延迟)、枚举(推荐，防反射/序列化)
- **DCL双重检查锁定**：synchronized前null检查(避免每次都加锁)、synchronized后null检查(避免并发情况下创建多个实例)。instance必须volatile（禁止指令重排序）
- **静态内部类(Holder)**：利用JVM类加载机制的隐式线程安全。SingletonHelper类在被调用时才加载，instance被初始化。类加载的clinit由JVM保证线程安全
- **枚举单例**：简洁、线程安全(JVM保证只会实例化一次)、防反射(Constructor.newInstance抛异常)、防序列化(反序列化返回同一实例)。最安全的单例实现
- **破坏单例的方式**：反射setAccessible破坏(除枚举外无法防御，可在构造器中抛异常防御)、序列化反序列化产生新对象(readObject创建新实例，重写readResolve返回原实例解决)
- **工厂模式**：简单工厂(一个工厂方法根据参数创建不同产品)、工厂方法(定义产品创建接口，子类决定创建哪种产品，符合OCP)、抽象工厂(创建相关产品族，不需要指定具体类)
- **建造者模式(Builder)**：链式调用设置参数，最后build生成对象。Lombok的@Builder注解自动生成。适用于参数多/可选参数多/参数校验复杂的对象创建
- **原型模式(Prototype)**：通过克隆创建对象。实现Cloneable接口并实现clone方法。Spring中Bean的作用域prototype每次返回一个新对象拷贝。浅拷贝vs深拷贝、原型管理器(缓存+克隆)
- **代理模式**：为对象提供代理控制对其访问。静态代理(编译时生成，手动编写代理类)、JDK动态代理(基于接口+InvocationHandler+Proxy.newProxyInstance)、CGLIB动态代理(基于继承+Enhancer+MethodInterceptor)
- **Java反射破坏单例**：通过Constructor.setAccessible(true)获取私有构造器，调用newInstance()创建新实例。枚举excepted(Constructor直接抛IllegalArgumentException)。防治：在构造器中判断实例是否已存在并抛异常
- **序列化破坏单例**：ObjectInputStream.readObject()调用readOrdinaryObject时会通过反射创建新对象（未执行构造器）。解决：在单例类中重写readResolve()方法返回已存在的实例
- **适配器模式(Adapter)**：将不兼容的接口转换为客户期望的另一个接口。类适配器(继承src类，实现target接口)、对象适配器(组合src对象，实现target接口，更灵活)、接口适配器(抽象类空实现接口，子类选择性覆盖)。Spring MVC中HandlerAdapter将不同类型的Controller适配给DispatcherServlet
- **装饰器模式(Decorator)**：动态给对象添加新职责。与代理区别：装饰器增强功能/添加行为，代理控制访问。Java IO流FilterInputStream/FilterOutputStream族是典型装饰器模式
- **桥接模式(Bridge)**：将抽象与实现解耦，使两者可独立变化。如JDBC中Driver(抽象)与Connection(实现)桥接，不同数据库驱动切换不影响API使用。JDBC的DriverManager就是桥接模式
- **外观模式(Facade)**：提供一个统一的高层接口，简化子系统使用。如网关聚合多个微服务接口、MyBatis的Configuration/SqlSession、SLF4J日志门面
- **组合模式(Composite)**：将对象组合成树形结构表示部分-整体层次，使客户端对单个对象和组合对象的使用一致。如文件系统(File/Folder)、Spring MVC参数解析器Composite
- **享元模式(Flyweight)**：共享细粒度对象，减少内存使用。内部状态(intrinsic)不变可共享，外部状态(extrinsic)可变由外部管理。Integer.valueOf(-128~127)缓存、数据库连接池、Spring单例Bean、String常量池
- **享元模式深入**: 将对象状态分为内在状态(Intrinsic/共享/不可变/存储在享元对象内部)和外在状态(Extrinsic/客户端管理/可变)。核心思想: 复用已有对象减少创建/共享相同内在状态的对象。典型应用: String常量池/Integer缓存-128~127/数据库连接池/线程池
- **策略模式(Strategy)**：定义一系列算法族，算法可互换，客户端选择不同的策略。如支付策略(微信/支付宝)、排序Comparator、Spring MVC参数解析器选择
- **模板方法模式(Template Method)**：定义算法骨架，具体步骤延迟到子类实现。父类定义final模板方法固定流程，子类实现抽象步骤。如AbstractList.get()、JdbcTemplate、HttpServlet.service()、RedisTemplate
- **观察者模式(Observer)**：一对多依赖关系，主题状态变化时通知所有观察者。Spring事件机制ApplicationEvent+ApplicationListener/ApplicationEventPublisher、ZooKeeper Watcher、Java9 Flow API(响应式)
- **责任链模式(Chain of Responsibility)**：多个处理者按链传递请求直到被处理或链结束。优点：请求者与处理者解耦。Servlet Filter链、Spring HandlerInterceptor、Netty ChannelPipeline、Sentinel SlotChain
- **状态模式(State)**：对象内部状态变化时改变其行为。将状态封装为独立的状态类（State接口+具体状态类），Context维护当前状态对象。如订单状态(待支付→已支付→已发货→已完成)、电梯状态、TCP连接状态
- **命令模式(Command)**：将请求封装为对象，参数化请求发送与执行。支持撤销/日志/队列。如JdbcTemplate.StatementCallback、线程池Runnable、GUI按钮Action
- **访问者模式(Visitor)**：数据结构与操作分离，在不修改数据结构的前提下定义新操作。元素类定义accept(Visitor)，Visitor定义visit每个具体元素。如FileVisitor遍历文件树、ASM字节码操作、注解处理器
- **Spring中的设计模式**：IoC容器(单例工厂+注册表)、DI依赖注入(桥梁)、AOP动态代理(Proxy/Decorator)、ApplicationContext事件(Observer)、HandlerAdapter(Adapter)、JdbcTemplate/RedisTemplate/RestTemplate(Template Method)、DispatcherServlet(前端控制器)、BeanPostProcessor(模板方法)、MVC参数解析器(Strategy/Composite)

# 2. 面试题

## 2.1 设计模式的七大原则及含义？
1. **开闭原则(OCP)**：对扩展开放，对修改关闭。通过抽象扩展新功能，而不是修改已有代码。如定义接口，新增功能时添加新实现类。
2. **里氏替换原则(LSP)**：子类必须能够完全替换父类而不产生错误。子类不应修改父类方法预期行为，不应抛出父类未声明的异常。
3. **依赖倒置原则(DIP)**：高层模块不应依赖低层模块，两者都应依赖其抽象。抽象不应依赖细节，细节应依赖抽象。Spring IoC就是此原则的实践。
4. **单一职责原则(SRP)**：一个类只负责一项职责（一个变化原因）。提高内聚性，当需求变化时只需要修改对应的类。
5. **接口隔离原则(ISP)**：客户端不应被迫依赖它不使用的接口。臃肿接口应拆分为多个精专接口。如UserService拆分ReadService+WriteService。
6. **迪米特法则(LoD)**：最少知道原则，一个对象应保持对其他对象最少了解。只与直接朋友通信，减少类间耦合度。
7. **合成复用原则(CRP)**：优先使用组合/聚合关系（黑盒复用），其次使用继承（白盒复用）。组合更灵活、耦合更低。

## 2.2 单例模式有几种实现方式？枚举单例为什么最安全？
**常用实现方式**：
1. **饿汉式**：static final字段直接new，类加载时创建，线程安全。缺点：若不用则浪费内存
```java
public class Singleton {
    private static final Singleton INSTANCE = new Singleton();
    private Singleton() {}
    public static Singleton getInstance() { return INSTANCE; }
}
```
2. **DCL懒汉式**：双重检查 + volatile + synchronized，懒加载
3. **静态内部类(Holder)**：内部类Holder持有实例，类被加载时才创建，线程安全
```java
public class Singleton {
    private Singleton() {}
    private static class Holder {
        static final Singleton INSTANCE = new Singleton();
    }
    public static Singleton getInstance() { return Holder.INSTANCE; }
}
```
4. **枚举**：最简洁、最安全
```java
public enum Singleton {
    INSTANCE;
    public void doSomething() {}
}
```

**枚举为什么最安全**：
- 反射安全：通过Constructor.newInstance()尝试创建枚举实例时，Constructor内部会检查若为枚举类型则直接抛IllegalArgumentException("Cannot reflectively create enum objects")
- 序列化安全：枚举的readObject方法通过ObjectInputStream读取名称，然后Enum.valueOf(name)获取同一个实例，确保序列化/反序列化前后是同一对象
- 线程安全：枚举类的初始化由JVM在类加载阶段完成，clinit方法是线程安全的

## 2.3 如何破坏单例模式？如何防御？
**破坏方式**：
1. **反射**：通过Constructor.setAccessible(true)获取私有构造器，调用newInstance()
2. **序列化/反序列化**：ObjectInputStream.readObject()通过反射创建新实例
3. **克隆**：实现Cloneable，若有clone方法（单例通常不应实现Cloneable）

**防御**：
1. 反射防御（非枚举）：在私有构造器中判断实例是否已存在，若已存在抛异常; volatile + AtomicReference标志位
2. 序列化防御：重写readResolve()方法返回已存在的实例
```java
private Object readResolve() { return getInstance(); }
```
3. 枚举天然防御：Constructor源码中对枚举类型直接禁止newInstance；序列化原生支持单例

## 2.4 工厂模式和抽象工厂模式的区别？
| 特性 | 简单工厂 | 工厂方法 | 抽象工厂 |
|------|----------|----------|----------|
| 定义 | 一个工厂类根据参数创建不同产品 | 定义创建产品接口，子类决定创建哪种 | 创建相关/相互依赖的一组产品 |
| 产品 | 一个产品等级 | 一个产品等级 | 多个产品等级（产品族） |
| 扩展 | 修改工厂类(违反OCP) | 添加新工厂子类(符合OCP) | 添加新产品族(符合OCP) |
| 复杂度 | 简单 | 中等 | 复杂 |

工厂方法：ProductA接口 → ConcreteProductA1、ConcreteProductA2实现; FactoryA接口 → ConcreteFactoryA1、ConcreteFactoryA2分别生产A1和A2。

抽象工厂：同时生产ProductA(CPU)和ProductB(GPU)，IntelFactory生产IntelCPU+IntelGPU，AmdFactory生产AmdCPU+AmdGPU。两个产品形成一个产品族。

## 2.5 动态代理模式有哪些？Spring AOP用哪种？
**JDK动态代理**：
- 基于接口，通过反射机制生成代理类实现目标接口
- Proxy.newProxyInstance(ClassLoader, interfaces[], InvocationHandler)
- 代理类extends Proxy implements interfaces
- 只能代理接口中的方法

**CGLIB动态代理**：
- 基于继承，生成目标类的子类
- Enhancer.create() + MethodInterceptor
- 基于ASM字节码技术，通过FastClass直接调用（比反射快）
- 不能代理final类/方法

**Spring AOP选择**：
- 默认：目标有接口用JDK动态代理，无接口用CGLIB
- proxy-target-class=true：统一用CGLIB
- Spring Boot 2.0+ AOP默认使用CGLIB（不管有没有接口）

## 2.6 代理模式和装饰器模式的区别？
表面相似：两者都是持有目标对象引用，通过包装增强功能。

**本质区别**：
- **代理模式**：控制对目标对象的访问（权限校验、延迟加载、远程代理、AOP）。关注访问控制，代理和目标对象通常事先确定
- **装饰器模式**：动态给对象增加新功能（添加行为）。关注功能增强，装饰器可以在运行时组合多个

实例：IO流中BufferedReader是装饰器（添加缓冲功能），而RPC远程调用的Stub是代理（控制网络通信）。Spring AOP的@Transactional代理属于代理模式。

类比：代理是你让秘书帮你挡电话（控制访问），装饰器是给手机加保护壳（添加功能）。

## 2.7 适配器模式有哪些形式？在Spring MVC中有何应用？
三种形式：
1. **类适配器**：Adapter继承Adaptee并实现Target接口。使用继承，不够灵活，不能同时适配多个Adaptee
2. **对象适配器**：Adapter持有Adaptee引用并实现Target接口。使用组合，更灵活，推荐
3. **接口适配器(缺省适配器)**：抽象类空实现接口的全部方法，子类选择性重写需要的方法。如WindowAdapter为空实现WindowListener接口

**Spring MVC中的应用**：
DispatcherServlet接收到请求后，通过HandlerMapping找到Handler（Controller），但DispatcherServlet不知道Handler的具体类型（可能是注解的@Controller、HttpRequestHandler、Servlet等），于是通过HandlerAdapter进行适配调用。每个HandlerAdapter支持特定类型的Handler，如RequestMappingHandlerAdapter适配@RequestMapping注解的Controller。

## 2.8 观察者模式是什么？Spring中如何使用？
观察者模式定义了一对多依赖关系，当主题(Subject/Observable)状态发生变化时，所有依赖的观察者(Observer)都会被通知并自动更新。

Java原生实现：java.util.Observable(被观察者基类)和java.util.Observer(观察者接口)。因Observable是类而非接口等原因，不太实用。

**Spring中的实现**：
1. 定义事件类继承ApplicationEvent
2. 定义监听器实现ApplicationListener<EventType>或使用@EventListener注解
3. 通过ApplicationEventPublisher.publishEvent(event)发布事件
4. Spring容器自动将匹配的监听器注册到事件

Spring事件机制特点：默认同步（广播事件阻塞等待所有监听器处理完），可以通过@Async+@EnableAsync实现异步处理。底层使用SimpleApplicationEventMulticaster管理监听器注册和广播。

## 2.9 责任链模式是什么？在哪些框架中有应用？
责任链将多个处理者连成一条链，请求在链上传递直到被处理或到达链尾。

**结构**：Handler接口定义handle(request)方法并持有nextHandler引用（或通过数组维护），具体处理器实现Handler接口，链中按顺序调用nextHandler.handle(request)传递请求。

**框架应用**：
- **Servlet Filter**：doFilter(ServletRequest, ServletResponse, FilterChain)，filterChain.doFilter传到下一个Filter
- **Spring HandlerInterceptor**：preHandle -> Controller -> postHandle -> afterCompletion，按注册顺序执行
- **Netty ChannelPipeline**：inbound从Head到Tail传播，outbound从Tail到Head传播
- **Sentinel**：SlotChain（ProcessorSlot），规则校验链
- **网关**：GatewayFilter全局/局部过滤器链

优点：请求者与处理者解耦、可以动态增减处理器、灵活调整处理顺序。

## 2.10 Spring中用了哪些设计模式？各举例说明
| 设计模式 | Spring中体现 | 说明 |
|----------|-------------|------|
| 单例模式 | Bean默认单例作用域 | Spring IoC容器中的Bean默认是单例 |
| 工厂模式 | BeanFactory/ApplicationContext | 通过getBean()创建和管理Bean |
| 模板方法 | JdbcTemplate/RedisTemplate/RestTemplate | 定义操作流程骨架，回调处理具体步骤 |
| 代理模式 | AOP动态代理 | 为Bean生成代理对象实现横切关注点 |
| 观察者模式 | ApplicationEvent/ApplicationListener | 容器内事件发布订阅机制 |
| 适配器模式 | HandlerAdapter | 适配不同类型的Handler(Controller) |
| 策略模式 | Resource实现类(ClassPathResource/UrlResource) | 同一接口不同实现 |
| 装饰器模式 | BeanDefinitionDecorator | 动态添加Bean定义属性 |
| 责任链模式 | HandlerInterceptor链 | 拦截器按顺序处理请求 |
| 外观模式 | JdbcUtils/TransactionSynchronizationUtils | 统一简化复杂子系统操作 |

IoC容器本身综合了多种模式：工厂模式创建Bean、单例注册表管理实例、策略模式选择实现、依赖注入组合对象。Spring AOP是动态代理和装饰器模式结合的典型应用。

## 2.11 模板方法模式在框架中的应用？
模板方法模式：父类定义算法骨架(final方法固定流程)，将可变步骤抽象化延迟到子类实现。

**Spring中的应用**：
- **JdbcTemplate**：execute/query等方法定义获取连接→创建Statement→执行SQL→处理ResultSet→释放连接的骨架，回调RowMapper、StatementCallback等处理具体业务
- **AbstractApplicationContext.refresh()**：定义了容器刷新12步骨架（obtainFreshBeanFactory→prepareBeanFactory→postProcessBeanFactory→invokeBeanFactoryPostProcessors→registerBeanPostProcessors→...），子类按需扩展
- **TransactionTemplate.execute()**：开启事务→执行业务逻辑→提交/回滚→清理资源

**其他框架**：
- **HttpServlet.service()**：doGet/doPost等由子类实现的标准模板
- **MyBatis BaseExecutor**：一级缓存逻辑固定，具体query/update由子类实现

## 2.12 享元模式的原理及Java中的应用？
享元模式核心思想是共享细粒度对象以节约内存。将对象状态分为：
- **内部状态(Intrinsic)**：不随环境变化，可共享，存储在享元对象内部
- **外部状态(Extrinsic)**：随环境变化，不可共享，由客户端维护

**使用流程**：客户端通过享元工厂获取享元对象（工厂内部维护享元池），传入外部状态调用方法。

**Java中的应用**：
- Integer.valueOf(-128~127)使用IntegerCache缓存
- String常量池（字面量字符串被共享）
- 数据库连接池（连接对象复用）
- Spring单例Bean（容器中默认单例的对象复用）
- 线程池（线程复用）
- Boolean.TRUE/Boolean.FALSE缓存

注意：享元模式与单例模式的本质区别——享元可以有多个不同内部状态的实例（如Integer不同数值），单例只有一个实例。

## 2.13 策略模式和状态模式的区别？
两者类结构非常相似（都有一个Context和多个策略/状态类），但意图完全不同：
- **策略模式**：由客户端主动选择策略，策略之间无关联，不自动切换。如支付策略——用户手动选微信/支付宝
- **状态模式**：由状态对象自身决定转移到哪个状态，状态间有转移规则，自动切换。如订单状态——支付后自动变为已支付状态

类比：策略模式是你选择走哪条路（不同路口选不同路线），状态模式是交通灯自动变色（红灯→绿灯→黄灯→红灯）。

**策略模式实现对比**：
- 策略：Context持有一个Strategy引用，客户端setStrategy()主动设置
- 状态：Context持有一个State引用，每个State内部通过Context.setState()自动转换到下一状态

## 2.14 桥接模式是什么？JDBC如何体现？
桥接模式将抽象部分与实现部分分离，使它们可以独立变化。用组合代替继承来连接两个维度的变化，避免类爆炸。

**JDBC中的桥接模式**：Driver(驱动)是实现维度，Connection/Statement/ResultSet是抽象维度。应用程序只需要面向JDBC API编程，底层数据库切换只换Driver实现，API层完全不变。

Connection conn = DriverManager.getConnection(url) — 返回的是不同数据库的Connection实现，但使用方式完全相同。这就是桥接模式：API层不变（桥的一端），不同数据库实现独立变化（桥的另一端），通过DriverManager桥接器连接。

## 2.15 组合模式和享元模式的对比？
**组合模式**：处理树形/整体-部分结构，使客户端一致对待单个对象和组合对象。核心是让叶子节点和容器节点实现同一接口。

**享元模式**：解决大量细粒度对象的创建开销，通过共享节约内存。核心是区分内外状态，在享元池中复用。

两者联系：有时组合模式中的叶子节点很多时，可使用享元模式共享（但需叶子节点无差异的状态才适合）。如文本编辑器中每个字符是一个对象，用享元模式（FlyweightCharacter）避免为每个重复字符创建独立对象，用组合模式（Document → Paragraph → Character）组织文档结构。

## 2.16 什么是享元设计模式？在Java中有哪些应用？
享元模式（Flyweight Pattern）通过共享细粒度对象来减少内存占用和提高性能。核心设计：

**两种状态**：
- **内在状态（Intrinsic State）**：不随环境变化，可共享，存储在享元对象内部。如字符的字形数据（无论在哪出现，字形不变）。
- **外在状态（Extrinsic State）**：随环境变化，不可共享，由客户端维护。如字符的位置、颜色、字体大小。

**核心角色**：
- **Flyweight（享元接口）**：定义接收外在状态的方法
- **ConcreteFlyweight（具体享元）**：存储内在状态，实现享元接口
- **FlyweightFactory（享元工厂）**：维护享元池，提供获取享元对象的方法
- **Client（客户端）**：维护外在状态，通过工厂获取享元对象

**Java中的典型应用**：
| 应用 | 说明 |
|------|------|
| **String常量池** | 相同字面量字符串被JVM共享，存储在字符串常量池 |
| **Integer缓存** | `Integer.valueOf()` 缓存 -128~127 范围的Integer对象 |
| **数据库连接池** | 连接对象复用（HikariCP/Druid），避免频繁创建销毁连接 |
| **线程池** | 线程复用（ThreadPoolExecutor），减少线程创建开销 |
| **Boolean.TRUE/FALSE** | Boolean类缓存了TRUE和FALSE两个静态实例 |
| **Spring单例Bean** | IoC容器中单例作用域的Bean被共享复用 |

**与单例模式的区别**：享元模式可以有多个不同内在状态的实例（如Integer不同数值对应不同享元对象），单例模式只有一个实例。
