# 1. OCP开闭原则
OCP 开闭原则是软件七大开发原则当中最基本的一个原则
OCP 开闭原则：
在软件开发过程中应当对扩展开放，对修改关闭。
在软件开发过程中，通过添加额外的类对原先程序已有的功能进行扩展是没问题的，但是如果因为要进行功能扩展而修改之前运行正常的程序，这是不被允许的。
因为一旦修改之前运行正常的程序，那么项目整体就要进行重新测试，这会导致额外的开销和资源消耗，并且测试过程相当麻烦。
OCP 开闭原则是最核心的、最基本的，其他的六个开发原则都是为这个原则服务的。
是否违反 OCP 开闭原则的判断：
只要你在扩展系统功能的时候，没有修改以前写好的代码，那么你就是符合 OCP 原则的。
反之，如果在扩展系统功能的时候，你修改了之前的代码，那么这个设计是失败的，违背了OCP原则。

# 2. 依赖倒置原则（Dependency Inversion Principle，即DIP）
依赖倒置原则，就是不再显示的建立类之间的关联关系，而是面向接口编程，面向抽象编程，不要面向具体编程，从而降低程序的耦合度，提高程序的扩展力，增强代码复用性
依赖倒置原则的目的：降低程序的耦合度，提高程序的扩展力，增强代码复用性
是否符合依赖倒置原则：
上 不依赖 下，就是符合，即类的实例对象之间没有明确固定在代码中的依赖关系
上 依赖 下，就是违背。只要“下”一改动，“上”就受到牵连。
符合依赖倒置原则的编程思想：控制反转
# 3. IoC控制反转
当前程序的设计，显然既违背OCP，又违背DIP，可以采用“控制反转”这种编程思想来解决这个问题。
控制反转（Inversion of Control，缩写为IoC），是面向对象编程中的一种设计思想，可以用来降低代码之间的耦合度，符合依赖倒置原则。
由于控制反转思想出现的比较新，所以没有被纳入GoF23种设计模式范围内。
控制反转的核心思想：
将对象的创建权交出去
将对象和对象之间关系的管理权交出去
对象的创建和对象之间关系的维护都由第三方容器来负责
控制反转中的反转是两件事：
第一件事：不在程序中采用硬编码的方式来new对象了，将new对象的权利交出去
第二件事：不在程序中采用硬编码的方式来维护对象的关系了，对象之间关系的维护权交出去了
控制反转思想的实现：依赖注入
# 4. 依赖注入
控制反转的实现方式有多种，其中比较重要的叫做：依赖注入(Dependency Injection，简称DI)。
依赖注入，用于实现对象之间关系的建立
控制反转是思想。依赖注入是这种思想的具体实现。
依赖注入DI，又包括常见的两种方式：
第一种：set注入（执行set方法给属性赋值）
第二种：构造方法注入（执行构造方法给属性赋值）
依赖注入 中 “依赖”是什么意思？ “注入”是什么意思？
依赖：A对象和B对象的关系。
注入：是一种手段，通过这种手段，可以让A对象和B对象产生关系。
依赖注入：对象A和对象B之间的关系，靠注入的手段来维护。而注入包括：set注入和构造注入。
# 5. 术语小结
OCP：开闭原则（开发原则）
DIP：依赖倒置原则（开发原则）
IoC：控制反转（一种思想，一种新型的设计模式）
DI：依赖注入（控制反转思想的具体实现方式）

# 6. 源码视角：设计原则在 Spring 中的体现

## 6.1 OCP 开闭原则 — BeanPostProcessor 扩展点

```
Spring 中对扩展开放、对修改闭合的典型体现是 BeanPostProcessor 扩展点：

// 核心流程是"闭合"的 —— doCreateBean 不需要修改
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
    // ... 实例化、属性赋值等核心流程

    // ★ 但流程中的每个节点都对外开放了扩展点
    wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    //        ↑ 遍历所有 BeanPostProcessor，逐个调用

    invokeInitMethods(beanName, wrappedBean, mbd);

    wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    //        ↑ 遍历所有 BeanPostProcessor，逐个调用

    return exposedObject;
}

// 扩展只需实现 BeanPostProcessor 接口即可，无需修改任何 Spring 核心代码：
@Component
public class MyCustomProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        // 自定义逻辑：对 Bean 进行额外处理
        return bean;
    }
}

★ OCP 原则的体现：
  - 对扩展开放：实现 BeanPostProcessor 接口，自定义处理逻辑
  - 对修改闭合：doCreateBean 的核心骨架不需要任何修改
```

## 6.2 DIP 依赖倒置原则 — 面向接口编程

```
Spring 中所有的核心组件都通过接口定义契约，具体实现可以灵活切换：

// 依赖接口（抽象），不依赖具体实现
@Service
public class OrderService {
    @Autowired
    private OrderDao orderDao;  // ← 依赖接口 OrderDao，不依赖 OrderDaoImplForMySQL
}

// 运行时切换实现，OrderService 代码无需任何修改：
// 方案A：@Qualifier("mysqlDao") → 走 MySQL 实现
// 方案B：@Qualifier("oracleDao") → 走 Oracle 实现

// Spring 内部也是同样设计：
// BeanFactory → DefaultListableBeanFactory
// ApplicationContext → ClassPathXmlApplicationContext / AnnotationConfigApplicationContext
// InstantiationStrategy → SimpleInstantiationStrategy / CglibSubclassingInstantiationStrategy
// AopProxyFactory → DefaultAopProxyFactory（内部选择 JDK 或 CGLIB）

★ DIP 原则的体现：
  - 上层模块（OrderService）依赖接口（OrderDao），不依赖具体类
  - 具体实现类（OrderDaoImplForMySQL）也依赖接口（OrderDao）
  - 这就是"上下都依赖抽象"，完全符合 DIP
```

## 6.3 IoC 控制反转 — 容器管理对象

```
传统方式（控制正转）：
  OrderService service = new OrderService();        // 自己 new
  service.setOrderDao(new OrderDaoImplForMySQL());   // 自己组装依赖

IoC 方式（控制反转）：
  // 对象的创建权和依赖关系的管理权都交给了 Spring 容器
  // 我们只管"声明"需要什么，Spring 负责"提供"

  @Service
  public class OrderService {
      @Autowired
      private OrderDao orderDao;  // ★ 只需声明依赖，创建和注入由容器完成
  }

Spring 源码中的 IoC 体现：
// DefaultListableBeanFactory 存储了所有 Bean 的创建规则和实例
private final Map<String, BeanDefinition> beanDefinitionMap;  // 创建规则
// + 继承自 DefaultSingletonBeanRegistry 的三级缓存：      // 实例存储
// private final Map<String, Object> singletonObjects;       // 一级缓存

// getBean("orderService") 时：
// 1. 查 BeanDefinition → 知道要创建 OrderService，且需要注入 OrderDao
// 2. 递归查 OrderDao 的 BeanDefinition → 找到 OrderDaoImplForMySQL
// 3. 创建 OrderDaoImplForMySQL → 注入到 OrderService
// 4. 返回完整的 OrderService 实例
```

## 6.4 DI 依赖注入 — 两种注入方式的源码实现

```
// Setter 注入（BeanWrapperImpl.setPropertyValue）：
// 通过 JavaBean 的 setter 方法注入
PropertyHandler ph = beanWrapper.getLocalPropertyHandler("orderDao");
Method writeMethod = pd.getWriteMethod();  // setOrderDao() 方法
writeMethod.invoke(bean, value);           // ★ 反射调用 setter

// 构造注入（ConstructorResolver.autowireConstructor）：
// 通过构造方法参数注入
Constructor<?> ctor = chosenCtors[0];       // 选中的构造方法
Object[] args = resolveConstructorArguments(beanName, mbd, ctor);
// args[0] = 从容器获取的 OrderDao 实例
BeanUtils.instantiateClass(ctor, args);     // ★ 带参构造方法创建对象

// 字段直接注入（AutowiredFieldElement.inject）：
// 通过反射直接设置字段值，不需要 setter
Field field = (Field) this.member;
ReflectionUtils.makeAccessible(field);
field.set(bean, value);                     // ★ 直接设置字段
```

## 6.5 设计原则关系图

```
    OCP（开闭原则）
    ┌──────────────────────────────────────┐
    │  对扩展开放，对修改闭合                 │
    │  Spring: BeanPostProcessor 扩展点     │
    └──────────────┬───────────────────────┘
                   │ 是目标
                   ▼
    DIP（依赖倒置原则）
    ┌──────────────────────────────────────┐
    │  面向接口编程，不依赖具体实现           │
    │  Spring: BeanFactory 接口 + 多种实现   │
    └──────────────┬───────────────────────┘
                   │ 指导思想
                   ▼
    IoC（控制反转）
    ┌──────────────────────────────────────┐
    │  将对象创建权和关系维护权交给容器        │
    │  Spring: ApplicationContext 管理 Bean  │
    └──────────────┬───────────────────────┘
                   │ 具体实现
                   ▼
    DI（依赖注入）
    ┌──────────────────────────────────────┐
    │  Setter/构造/字段 三种注入方式          │
    │  Spring: populateBean + BeanWrapper   │
    └──────────────────────────────────────┘
```
