# 1. Bean 的实例化方式
Spring为Bean提供了多种实例化方式，包括如下4种方式：
第一种：通过构造方法实例化
第二种：通过简单工厂模式实例化
第三种：通过factory-bean实例化（工厂方法模式实例化）
第四种：通过FactoryBean接口实例化
Spring中为Bean对象的创建准备了多种方案，目的是为了能够更加灵活


# 1. 通过构造方法实例化
```
我们之前一直使用的就是这种方式。
在spring配置文件中直接配置类全路径，即在Spring配置文件中配置bean，Spring会自动调用该类的无参数构造方法来实例化Bean

/**
 * ClassName: SpringBean
 * Package: cw.sspring.study.bean
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-25 18:10
 * @Version 1.0
 */
public class SpringBean {
}

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

	<bean id="springBean" class="cw.spring.study.bean.SpringBean"/>
	
</beans>

@org.junit.Test
public void test1() {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    System.out.println(applicationContext.getBean("springBean", SpringBean.class));
}

```

# 2. 通过简单工厂模式实例化
```
抽象产品类
/**
 * ClassName: Fruit
 * Package: cw.spring.study.bean
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-25 18:20
 * @Version 1.0
 */
public abstract class Fruit {}

具体产品类
/**
 * ClassName: Apple
 * Package: cw.spring.study.bean
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-25 18:20
 * @Version 1.0
 */
public class Apple extends Fruit{}

工厂类
/**
 * ClassName: FruitFactory
 * Package: cw.spring.study.bean
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-25 18:21
 * @Version 1.0
 */
public class FruitFactory {

    // 创建水果对象的方法
    public static Fruit getFruit() {
        // 最终的Apple类对象是通过调用该工厂类的该方法new的对象
        return new Apple();
    }
    
}

配置文件配置工厂类生产 bean
在Spring配置文件中告诉Spring框架，调用哪个类的哪个方法获取 Bean
<!--
  factory-method属性
    指定的是工厂类当中的静态方法，也就是告诉Spring框架，调用这个方法可以获取Bean
  创建出来的为产品对象，不为工厂对象，工厂类中的方法为静态方法，通过类名直接调用创建对象
  通过class属性指定的类调用该类的静态工厂方法，进行对象的创建，
	最终创建的对象为产品对象，id为apple
-->
<bean id="apple" class="cw.spring.study.bean.FruitFactory" factory-method="getFruit"/>

测试
@org.junit.Test
public void test1() {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    System.out.println(applicationContext.getBean("apple", Fruit.class));
}

```

# 3. 通过factory-bean实例化（工厂方法模式实例化）
```
抽象产品类
/**
 * ClassName: Fruit
 * Package: cw.spring.study.bean
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-25 18:20
 * @Version 1.0
 */
public abstract class Fruit {}

具体产品类
/**
 * ClassName: Apple
 * Package: cw.spring.study.bean
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-25 18:20
 * @Version 1.0
 */
public class Apple extends Fruit{}

抽象工厂类
/**
 * ClassName: FruitFactory
 * Package: cw.spring.study.bean
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-25 18:21
 * @Version 1.0
 */
public abstract class FruitFactory {
    
    /**
     * 水果工厂生产水果的方法
     * 
     * @return 水果
     */
    public abstract Fruit getFruit();
    
}

具体工厂类
/**
 * ClassName: AppleFactory
 * Package: cw.spring.study.bean
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-25 18:37
 * @Version 1.0
 */
public class AppleFactory extends FruitFactory{

    // 实例方法
    @Override
    public Fruit getFruit() {
        // 生产苹果
        return new Apple();
    }
}

配置文件配置工厂类生产 bean
告诉Spring框架，调用哪个对象的哪个方法来获取Bean
<!-- 
  工厂方法模式，通过 factory-bean属性 + factory-method属性来共同完成 
  由于工厂类用于创建对象的方法为实例方法，所以需要先创建工厂对象，工厂对象也需要被spring管理
  AppleFactory 实际上是一个工厂bean，用于创建对象的bean
-->
<bean id="appleFactory" class="cw.spring.study.bean.AppleFactory"/>
<!--
  factory-bean属性
    告诉Spring调用哪个工厂对象，进行bean的创建
  factory-method属性
    告诉Spring调用工厂对象的哪个实例方法创建bean
  该配置用于指定用哪个工厂对象的哪个方法创建对象
-->
<bean id="apple" factory-bean="appleFactory" factory-method="getFruit"/>

测试
@org.junit.Test
public void test1() {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    System.out.println(applicationContext.getBean("apple", Fruit.class));
}

```
# 4. 通过FactoryBean接口实例化
```
以上的第三种方式中，factory-bean（生产bean的工厂bean）是我们自定义的，factory-method（生产bean的工厂方法）也是我们自己定义的。
在Spring中，当我们编写的类直接实现FactoryBean接口之后，用于创建bean的factory-bean就不需要我们指定了，工厂bean的factory-method也不需要我们指定了。factory-bean会自动指向实现FactoryBean接口的类，factory-method会自动指向该类中的getObject()方法。
因为我们在配置文件中指定的用于创建bean的类在实现FactoryBean接口之后，Spring就知道这是一个工厂类，并且知道工厂方法是哪个
这种方式实际上就是第三种方式的简化
FactoryBean在Spring中是一个接口，被称为“工厂Bean”。
“工厂Bean”是一种特殊的Bean，所有的“工厂Bean”都是用来协助Spring框架来创建其他Bean对象的。

产品类
/**
 * ClassName: Person
 * Package: cw.spring.study.bean
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-25 18:56
 * @Version 1.0
 */
public class Person {}

工厂类
import org.springframework.beans.factory.FactoryBean;

/**
 * ClassName: PersonFactoryBean
 * Package: cw.spring.study.bean
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-25 18:57
 * @Version 1.0
 */
public class PersonFactoryBean implements FactoryBean<Person> {
    
    // 创建bean的工厂方法
    @Override
    public Person getObject() throws Exception {
        return new Person();
    }
    
    @Override
    public Class<?> getObjectType() {
        return null;
    }
}

public interface FactoryBean<T> {
    String OBJECT_TYPE_ATTRIBUTE = "factoryBeanObjectType";

    @Nullable
    T getObject() throws Exception;

    @Nullable
    Class<?> getObjectType();

    // 该方法接口进行了默认实现，工厂bean创建的对象是否为单例的
    // 默认情况下，创建的对象是单例的
    // 如果要让工厂bean创建的对象为多例，则重写该方法，返回false即可
    default boolean isSingleton() {
        return true;
    }
}

配置文件配置工厂类生产 bean
<!--
  这种方式实际上就是第三种方式的简化。
  由于你编写的类实现了FactoryBean接口，所以这个类是一个特殊的类，
  不需要你再手动指定：factory-bean、factory-method
  实现了FactoryBean接口，Spring知道这是一个工厂类，并且知道用于创建bean的工厂方法是哪个
  
  通过一个特殊的Bean：工厂Bean，来返回一个普通的Bean Person对象。
  通过FactoryBean这个工厂Bean主要是想对普通Bean进行加工处理。
-->
<bean id="person" class="cw.spring.study.bean.PersonFactoryBean"/>

测试
@org.junit.Test
public void test1() {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    System.out.println(applicationContext.getBean("person", Person.class));
}

```

# 5. 工厂方法实例化小结
相比于通过构造方法进行bean的实例化，采用工厂方法进行bean的实例，我们可以在bean对象创建出来到将创建出来的对象交给Spring容器进行管理这一段时间内，我们可以对bean对象进行加工处理。

# 6. BeanFactory和FactoryBean的区别
## 6.1 BeanFactory
BeanFactory Spring 容器的顶级父接口
Spring IoC容器的顶级对象，BeanFactory被翻译为“Bean工厂”，在Spring的IoC容器中，“Bean工厂”负责创建Bean对象，即BeanFactory是一个创建bean对象的工厂。
BeanFactory是工厂。
## 6.2 FactoryBean
FactoryBean：它是一个Bean，是一个能够辅助Spring实例化其它Bean对象的一个Bean。
在Spring中，Bean可以分为两类：
第一类：普通Bean
第二类：工厂Bean，工厂Bean也是一种Bean，只不过这种Bean比较特殊，它可以辅助Spring实例化其它Bean对象，即工厂Bean是一种用于创建Bean对象的Bean。
我们自己写的，用于实例化Bean对象的工厂Bean就是FactoryBean
工厂Bean是一种特殊的Bean，用于获取其他Bean的一种特殊Bean

# 7. FactoryBean练习-注入自定义Date
```
java.util.Date在Spring中被当做简单类型，简单类型在注入的时候可以直接使用value属性或value标签来完成。
但是对于Date类型来说，采用value属性或value标签赋值的时候，对日期字符串的格式要求非常严格，必须是这种格式的：Mon Oct 10 14:30:26 CST 2022。其他格式是不会被识别的。
/**
 * ClassName: Student
 * Package: cw.spring.study.bean
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-26 9:04
 * @Version 1.0
 */
public class Student {
    private Date birth;
    
    public void setBirth(Date birth) {
        this.birth = birth;
    }
    
    @Override
    public String toString() {
        return "Student{" + "birth=" + birth + '}';
    }
}

```
## 7.1 以简单类型的方式注入 Date
```
<bean id="student" class="cw.spring.study.bean.Student">
  <property name="birth" value="Mon Oct 10 14:30:26 CST 2022"/>
</bean>

@org.junit.Test
public void test2() {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    System.out.println(applicationContext.getBean("student", Student.class));
}
对于Date来说，我们也可以采用非简单类型的形式，但是如下这种方式创建出来的Date对象，默认是系统当前时间，也只能是系统当前时间
<!-- 采用这种形式创建出来的Date对象，默认是系统当前时间，也只能是系统当前时间 -->
<bean id="studentBirth" class="java.util.Date"/>

```

## 7.2 以非简单类型的方式通过 FactoryBean 接口实例化
```
采用工厂方法模式以非简单类型的方式通过 FactoryBean 接口实例化，我们可以在工厂方法中创建Date对象，同时还可以对Date对象进行加工，从而创建我们需要格式的日期对象
/**
 * ClassName: DateFactoryBean
 * Package: cw.spring.study.bean
 * Description:
 * DateFactoryBean 这是一个工厂Bean，
 * 用于协助Spring创建Date对象
 *
 * @Author tcw
 * @Create 2023-05-26 9:13
 * @Version 1.0
 */
public class DateFactoryBean implements FactoryBean<Date> {
    
    private String dateStr; // 日期字符串（简单类型）
    
    public void setDateStr(String dateStr) {
        this.dateStr = dateStr;
    }
    
    @Override
    public Date getObject() throws Exception {
        SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
        return dateFormat.parse(dateStr);
    }
    
    @Override
    public Class<?> getObjectType() {
        return null;
    }
}

<!-- 利用工厂Bean DateFactoryBean 创建指定日期的Date对象 -->
<bean id="date" class="cw.spring.study.bean.DateFactoryBean">
  <property name="dateStr" value="2023-05-22"/>
</bean>

<bean id="student" class="cw.spring.study.bean.Student">
  <property name="birth" ref="date"/>
</bean>
```

# 8. 源码解析：Bean 实例化策略与 FactoryBean 机制

## 8.1 createBeanInstance() — Bean 实例化的决策树

```
AbstractAutowireCapableBeanFactory.createBeanInstance() 是实例化的核心入口：

protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd,
        @Nullable Object[] args) {

    // 1. ★ 解析 Bean 的类型
    Class<?> beanClass = resolveBeanClass(mbd, beanName);

    if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers())
            && !mbd.isNonPublicAccessAllowed()) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                "Bean class isn't public, and non-public access not allowed: "
                + beanClass.getName());
    }

    // 2. ★ Supplier 回调实例化（最高优先级）
    //    通过 registerBean(..., Supplier<T>) 或 BeanDefinition 的 instanceSupplier
    Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
    if (instanceSupplier != null) {
        return obtainFromSupplier(instanceSupplier, beanName);
    }

    // 3. ★ 工厂方法实例化（factory-method）
    //    - 静态工厂方法：class 指定 factoryMethod
    //    - 实例工厂方法：factory-bean + factory-method
    if (mbd.getFactoryMethodName() != null) {
        return instantiateUsingFactoryMethod(beanName, mbd, args);
    }

    // 4. 缓存：如果之前解析过构造方法或工厂方法，直接复用
    boolean resolved = false;
    boolean autowireNecessary = false;
    if (args == null) {
        synchronized (mbd.constructorArgumentLock) {
            if (mbd.resolvedConstructorOrFactoryMethod != null) {
                resolved = true;
                autowireNecessary = mbd.constructorArgumentsResolved;
            }
        }
    }

    if (resolved) {
        // 5. ★ 之前已解析过构造方法 → 直接走构造注入
        if (autowireNecessary) {
            return autowireConstructor(beanName, mbd, null, null);
        } else {
            // ★ 无参构造 → 走默认实例化策略
            return instantiateBean(beanName, mbd);
        }
    }

    // 6. ★ 首次创建：检查是否有 @Autowired 构造方法
    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);

    // 7. ★ 根据检查结果选择实例化方式
    if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR
            || mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
        // ★ 构造注入（带参构造）
        return autowireConstructor(beanName, mbd, ctors, args);
    }

    // 8. ★ 默认：无参构造方法实例化
    return instantiateBean(beanName, mbd);
}
```

## 8.2 InstantiationStrategy — 实例化策略接口

```
Spring 提供了两个 InstantiationStrategy 实现，负责实际的反射调用：

public interface InstantiationStrategy {
    Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner)
            throws BeansException;
    Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner,
            Constructor<?> ctor, Object... args) throws BeansException;
    Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner,
            @Nullable Object factoryBean, Method factoryMethod, Object... args)
            throws BeansException;
}

两个实现类：

1. SimpleInstantiationStrategy（简单反射策略）
   - 使用 Java 反射（Constructor.newInstance()、Method.invoke()）
   - 适用于普通 Bean 的实例化
   - 不支持构造方法注入的"方法覆盖"（lookup-method）

2. CglibSubclassingInstantiationStrategy（CGLIB 子类化策略）
   - 继承自 SimpleInstantiationStrategy
   - 额外支持"方法注入"（lookup-method 和 replace-method）
   - 使用 CGLIB Enhancer 创建运行时子类
```

### 8.3 SimpleInstantiationStrategy.instantiate() 源码

```
public Object instantiate(RootBeanDefinition bd, @Nullable String beanName,
        BeanFactory owner, Constructor<?> ctor, Object... args) {

    // ... 其他逻辑

    // ★ 核心：使用反射调用构造方法
    Constructor<?> constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
    if (constructorToUse == null) {
        constructorToUse = ctor;
    }

    // 经典的反射调用
    return BeanUtils.instantiateClass(constructorToUse, args);
}

// BeanUtils.instantiateClass():
public static <T> T instantiateClass(Constructor<T> ctor, Object... args)
        throws BeanInstantiationException {
    try {
        ReflectionUtils.makeAccessible(ctor);
        // 处理 Kotlin 的可空参数
        return (KotlinDetector.isKotlinReflectPresent() &&
                KotlinDetector.isKotlinType(ctor.getDeclaringClass()) ?
                KotlinDelegate.instantiateClass(ctor, args) : ctor.newInstance(args));
    } catch (...) { ... }
}
```

### 8.4 CglibSubclassingInstantiationStrategy.instantiate() 源码

```
public Object instantiate(@Nullable Constructor<?> ctor, Object... args) {

    Class<?> subclass = createEnhancedSubclass(this.beanDefinition);
    Object instance;

    if (ctor == null) {
        // 无参构造
        instance = BeanUtils.instantiateClass(subclass);
    } else {
        try {
            // ★ 通过 CGLIB 子类的特定构造方法实例化
            Constructor<?> enhancedSubclassConstructor =
                    subclass.getConstructor(ctor.getParameterTypes());
            instance = enhancedSubclassConstructor.newInstance(args);
        } catch (Exception ex) { ... }
    }

    // ★ 为 CGLIB 子类设置方法拦截回调
    //    用于 lookup-method 和 replace-method
    Factory factory = (Factory) instance;
    factory.setCallbacks(new Callback[] {
            NoOp.INSTANCE,
            new LookupOverrideMethodInterceptor(this.beanDefinition, this.owner),
            new ReplaceOverrideMethodInterceptor(this.beanDefinition, this.owner)
    });
    return instance;
}
```

## 8.3 FactoryBean 机制源码：getObjectForBeanInstance()

```
FactoryBean 的处理在 getBean() 流程中有一个专门的步骤：getObjectForBeanInstance()

AbstractBeanFactory.getObjectForBeanInstance():

protected Object getObjectForBeanInstance(
        Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

    // 1. ★ 如果 name 以 & 开头，用户想要获取 FactoryBean 本身
    //    直接返回 FactoryBean 对象，不走 getObject()
    if (BeanFactoryUtils.isFactoryDereference(name)) {
        if (beanInstance instanceof NullBean) {
            return beanInstance;
        }
        if (!(beanInstance instanceof FactoryBean)) {
            throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
        }
        // 标记这个 BeanDefinition 是 FactoryBean 类型
        if (mbd != null) {
            mbd.isFactoryBean = true;
        }
        return beanInstance;  // 返回 FactoryBean 本身
    }

    // 2. 如果不是 FactoryBean 实例，直接返回
    if (!(beanInstance instanceof FactoryBean<?> factoryBean)) {
        return beanInstance;
    }

    Object object = null;
    if (mbd != null) {
        mbd.isFactoryBean = true;
    } else {
        // ★ 从缓存中查找 FactoryBean 创建的对象
        object = getCachedObjectForFactoryBean(beanName);
    }

    if (object == null) {
        // 从 FactoryBean 中获取对象
        if (mbd == null && containsBeanDefinition(beanName)) {
            mbd = getMergedLocalBeanDefinition(beanName);
        }
        boolean synthetic = (mbd != null && mbd.isSynthetic());
        // ★ 调用 getObjectFromFactoryBean()
        object = getObjectFromFactoryBean(factoryBean, beanName, !synthetic);
    }
    return object;
}
```

### 8.4 FactoryBeanRegistrySupport.getObjectFromFactoryBean()

```
// 位于 FactoryBeanRegistrySupport（AbstractBeanFactory 的父类）
protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName,
        boolean shouldPostProcess) {

    // 1. ★ 处理单例：第二次请求从缓存返回（保证 FactoryBean 的单例语义）
    if (factory.isSingleton() && containsSingleton(beanName)) {
        synchronized (getSingletonMutex()) {
            // 从 factoryBeanObjectCache 中取（这是一个 ConcurrentHashMap）
            Object object = this.factoryBeanObjectCache.get(beanName);
            if (object == null) {
                // ★ 调用 FactoryBean.getObject()
                object = doGetObjectFromFactoryBean(factory, beanName);
                Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
                if (alreadyThere != null) {
                    object = alreadyThere;
                } else {
                    // 后处理（如属性赋值、初始化后 AOP 代理等）
                    if (shouldPostProcess) {
                        if (isSingletonCurrentlyInCreation(beanName)) {
                            return object;  // 循环依赖中，不做后处理
                        }
                        beforeSingletonCreation(beanName);
                        try {
                            // ★ 执行 BeanPostProcessor 链
                            object = postProcessObjectFromFactoryBean(object, beanName);
                        } catch (Throwable ex) {
                            throw new BeanCreationException(beanName,
                                    "Post-processing of FactoryBean's singleton object failed", ex);
                        } finally {
                            afterSingletonCreation(beanName);
                        }
                    }
                    // 放入缓存
                    if (containsSingleton(beanName)) {
                        this.factoryBeanObjectCache.put(beanName, object);
                    }
                }
            }
            return object;
        }
    } else {
        // 2. ★ 处理原型：每次调用都会重新执行 getObject()
        Object object = doGetObjectFromFactoryBean(factory, beanName);
        if (shouldPostProcess) {
            try {
                object = postProcessObjectFromFactoryBean(object, beanName);
            } catch (Throwable ex) {
                throw new BeanCreationException(beanName,
                        "Post-processing of FactoryBean's object failed", ex);
            }
        }
        return object;
    }
}

protected Object doGetObjectFromFactoryBean(FactoryBean<?> factory, String beanName)
        throws BeanCreationException {
    Object object;
    try {
        if (System.getSecurityManager() != null) {
            AccessControlContext acc = getAccessControlContext();
            try {
                object = AccessController.doPrivileged(
                        (PrivilegedExceptionAction<Object>) factory::getObject, acc);
            } catch (PrivilegedActionException pae) {
                throw pae.getException();
            }
        } else {
            // ★ 真正调用 FactoryBean.getObject() !
            object = factory.getObject();
        }
    } catch (FactoryBeanNotInitializedException ex) {
        throw new BeanCurrentlyInCreationException(beanName, ex.toString());
    } catch (Throwable ex) {
        throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex);
    }

    // 如果 getObject() 返回 null，返回一个 NullBean 占位
    if (object == null) {
        if (isSingletonCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(
                    beanName, "FactoryBean which is currently in creation returned null from getObject");
        }
        object = new NullBean();
    }
    return object;
}
```

## 8.4 FactoryBean 单例/原型 与 Bean 的单例/原型 的关系

```
两个概念容易混淆，但作用层级不同：

┌─────────────────────────────────────────────────────┐
│        FactoryBean 本身的 scope                       │
├─────────────────────────────────────────────────────┤
│  FactoryBean 本身是什么 scope？                      │
│  默认是 singleton（由 Spring 管理）                  │
│                                                     │
│  <bean id="personFactory"                           │
│        class="cw.spring.study.bean.PersonFactoryBean"│
│        scope="singleton"/>                           │
│                                                     │
│  Spring 容器中只存在一个 PersonFactoryBean 实例       │
│  （FactoryBean 本身是单例）                           │
└─────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│        getObject() 返回对象的 scope                  │
├─────────────────────────────────────────────────────┤
│  由 FactoryBean.isSingleton() 返回 true/false 决定  │
│                                                     │
│  default boolean isSingleton() { return true; }     │
│                                                     │
│  如果 isSingleton() 返回 true：                       │
│    → getObject() 只执行一次，结果被缓存               │
│    → 所有 getBean("person") 返回同一个对象            │
│                                                     │
│  如果重写 isSingleton() 返回 false：                  │
│    → 每次 getBean("person") 都调用 getObject()       │
│    → 每次返回新对象                                   │
│                                                     │
│  注意：这和 FactoryBean 本身是否单例是独立的！       │
└─────────────────────────────────────────────────────┘
```

## 8.5 autowireConstructor() — 构造注入决策源码

```
protected BeanWrapper autowireConstructor(
        String beanName, RootBeanDefinition mbd,
        @Nullable Constructor<?>[] chosenCtors, @Nullable Object[] explicitArgs) {

    // 创建 ConstructorResolver
    ConstructorResolver constructorResolver = new ConstructorResolver();
    // ... 设置各种参数

    return constructorResolver.autowireConstructor(beanName, mbd, chosenCtors, explicitArgs);
}

ConstructorResolver.autowireConstructor() 的主要逻辑：

1. 确定候选构造方法：
   - chosenCtors != null → 使用传入的构造方法（来自 @Autowired）
   - 否则 → 获取所有 public 构造方法
   - 如果只有一个且参数列表为空 → 直接使用无参构造

2. 构造方法排序：
   - 按参数个数降序排（优先匹配参数多的构造器）
   - public 优先于非 public

3. 依次尝试每个候选构造方法：
   - 解析每个参数的类型
   - 从容器中获取匹配类型的 Bean
   - 通过 TypeConverter 进行类型转换
   - 如果所有参数都能找到匹配值 → 选中该构造方法

4. 缓存解析结果：
   - resolvedConstructorOrFactoryMethod ← 选中的构造方法
   - constructorArgumentsResolved ← 参数解析结果
```

## 8.6 设计思想总结

```
1. 策略模式（InstantiationStrategy）：
   实例化的"如何创建"通过策略接口与"何时创建"（createBeanInstance）解耦。
   可以在 Simple 和 CGLIB 两个策略之间切换，甚至可以自己实现新的策略。

2. FactoryBean 的"间接创建"模式：
   FactoryBean 在 Spring 中应用广泛（如 SqlSessionFactoryBean、ProxyFactoryBean）。
   它允许创建逻辑复杂（如需要解析、组装、后处理）的 Bean 通过编程方式控制创建过程，
   而不是简单的反射 new。

3. Supplier 的注册式创建（Spring 5+）：
   通过 registerBean("name", MyBean.class, MyBean::new) 可以完全跳过反射，
   直接使用 Supplier/构造函数引用。这是 Spring 从"基于反射"向"基于 Lambda"演进的体现。

4. 构造注入的优先级：
   autowireConstructor → factoryMethod → instantiateBean
   体现了 Spring 的"显式声明优先，默认行为兜底"的设计原则。
```
