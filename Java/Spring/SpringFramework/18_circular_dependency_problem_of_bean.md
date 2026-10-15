# 1. 什么是Bean的循环依赖
```
Bean的循环依赖，就是A对象中有B属性，B对象中有A属性。即我依赖你，你也依赖我。
比如：丈夫类Husband，妻子类Wife，Husband中有Wife的引用，Wife中有Husband的引用。

package cw.spring.bean;

/**
 * ClassName: Wife
 * Package: cw.spring.bean
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-12 16:35
 * @Version 1.0
 */
public class Wife {
    private String name;
    private Husband husband;
    
    public void setName(String name) {
        this.name = name;
    }
    
    public String getName() {
        return name;
    }
    
    public void setHusband(Husband husband) {
        this.husband = husband;
    }
    
    @Override
    public String toString() {
        return "Wife{" + "name='" + name + '\'' + ", husband=" + husband.getName() + '}';
    }
}

package cw.spring.bean;

/**
 * ClassName: Husband
 * Package: cw.spring.bean
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-12 16:36
 * @Version 1.0
 */
public class Husband {
    private String name;
    private Wife wife;
    
    public void setName(String name) {
        this.name = name;
    }
    
    public String getName() {
        return name;
    }
    
    public void setWife(Wife wife) {
        this.wife = wife;
    }
    
    @Override
    public String toString() {
        return "Husband{" + "name='" + name + '\'' + ", wife=" + wife.getName() + '}';
    }
}

```

# 2. 单例模式+set注入
## 2.1 单例模式 + set 注入情况下的循环依赖
```
在单例模式 + set 注入的情况下，循环依赖是没有问题的，Spring 可以解决这个问题。
<bean id="husband" class="cw.spring.bean.Husband" scope="singleton">
  <property name="name" value="张三"/>
  <property name="wife" ref="wife"/>
</bean>
<bean id="wife" class="cw.spring.bean.Wife" scope="singleton">
  <property name="name" value="李四"/>
  <property name="husband" ref="husband"/>
</bean>

@org.junit.Test
public void test01() {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    Husband husband = applicationContext.getBean("husband", Husband.class);
    Wife wife = applicationContext.getBean("wife", Wife.class);
    System.out.println(husband);
    System.out.println(wife);
}
```
## 2.2 分析单例模式 + set 注入情况下的循环依赖
在 set + singleton 模式下 Spring 对 Bean 的管理主要分为两个阶段：
第一个阶段：在 Spring 容器加载的时候，实例化 Bean，只要其中任意一个 Bean 实例化之后，马上进行“曝光”，不等属性赋值就曝光
提前曝光，不等属性赋值就可以使用，因为单例下就只会创建该类型的对象一个，早曝光和晚曝光一样，都是这个对象
第二个阶段：Bean“曝光”之后，再进行属性的赋值，调用set方法。
解决单例模式 + set 注入情况下的循环依赖的核心解决方案是：实例化对象和对象的属性赋值分为两个阶段来完成的。
注意：只有在 scope 是 singleton 的情况下，Bean 才会采取提前“曝光”的措施。prototype 下不会进行提前曝光，如果多例模式下多个对象进行提前曝光，则Spring在使用对象的时候，不知道要使用哪个对象

# 3. 多例模式+set注入
```
<bean id="husband" class="cw.spring.bean.Husband" scope="prototype">
  <property name="name" value="张三"/>
  <property name="wife" ref="wife"/>
</bean>
<bean id="wife" class="cw.spring.bean.Wife" scope="prototype">
  <property name="name" value="李四"/>
  <property name="husband" ref="husband"/>
</bean>

当循环依赖的两个 bean 的 scope 都是 prototype 的时候，在 prototype + setter 模式下的循环依赖，存在问题，会出现异常 BeanCurrentlyInCreationException（当前的 Bean 正处于创建中异常）。
因为都是多例模式，所以每次需要对象的时候都会new新的对象出来，A对象中需要B对象，会new一个新的B对象，B对象中需要A对象，会new一个新的A对象，新的A对象中需要B对象，会再new一个新的B对象…最终导致对象永远创建不完
如果其中任意一个是 singleton 的，就不会出现异常（因为单例模式下的 bean 实例化完成就会进行“曝光”，“曝光”后就可以停止死循环依赖）
singleton 在解析配置文件时就创建对象，当需要 prototype 时可以马上创建一个 prototype 对象赋值给singleton，而 prototype 对象所需的 singleton 只有唯一的一个，从而可以让依赖停止下来，所以没有问题，prototype 对象和 singleton 对象的创建时机反过来也是类似。

```

# 4. 单例模式+构造注入
```
单例模式 + 构造注入的方式下产生的循环依赖是无法解决的

package cw.spring.pojo;

/**
 * ClassName: Husband
 * Package: cw.spring.pojo
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-12 17:16
 * @Version 1.0
 */
public class Husband {
    private String name;
    private Wife wife;
    
    public Husband(String name, Wife wife) {
        this.name = name;
        this.wife = wife;
    }
    
    public String getName() {
        return name;
    }
    
    @Override
    public String toString() {
        return "Husband{" + "name='" + name + '\'' + ", wife=" + wife.getName() + '}';
    }
}

package cw.spring.pojo;

import cw.spring.bean.Husband;

/**
 * ClassName: Wife
 * Package: cw.spring.pojo
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-12 17:16
 * @Version 1.0
 */
public class Wife {
    private String name;
    private Husband husband;
    
    public Wife(String name, Husband husband) {
        this.name = name;
        this.husband = husband;
    }
    
    public String getName() {
        return name;
    }
    
    @Override
    public String toString() {
        return "Wife{" + "name='" + name + '\'' + ", husband=" + husband.getName() + '}';
    }
}

<!-- 默认情况下，Spring创建对象都是单例模式的 -->
<bean id="wife" class="cw.spring.pojo.Wife">
  <constructor-arg name="name" value="张三"/>
  <constructor-arg name="husband" ref="husband"/>
</bean>
<bean id="husband" class="cw.spring.pojo.Husband">
  <constructor-arg name="name" value="李四"/>
  <constructor-arg name="wife" ref="wife"/>
</bean>

@org.junit.Test
public void test01() {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    Husband husband = applicationContext.getBean("husband", Husband.class);
    Wife wife = applicationContext.getBean("wife", Wife.class);
    System.out.println(husband);
    System.out.println(wife);
}

Requested bean is currently in creation: Is there an unresolvable circular reference?
请求的bean当前正在创建中：是否存在无法解决的循环引用？
基于构造注入的方式下产生的循环依赖是无法解决的，使用构造注入无法创建完对象马上进行曝光，因为创建对象需要另一个对象作为属性，无法完成对象的创建(类似死锁)
当创建A对象的时候，需要B对象，会马上进行B对象的创建，但是创建B对象又需要A对象，可是这个时候A对象还在创建中，导致B对象也无法完成创建，最终两个对象的创建都无法进行，使得创建对象和依赖注入失败
基于构造注入的方式下产生的循环依赖是无法解决的，是由于构造方法注入会导致实例化对象的过程和对象属性赋值的过程没有分离开，必须在一起完成导致的。

```

# 5. Spring解决循环依赖的机理
```
Spring 可以解决 set + singleton 模式下循环依赖：
根本的原因在于：这种方式可以做到将“实例化Bean”和“给Bean属性赋值”这两个动作分开去完成。
实例化Bean的时候：调用无参数构造方法来完成。此时可以先不给属性赋值，可以提前将该Bean对象“曝光”给外界。
给Bean属性赋值的时候：调用setter方法来完成。
两个步骤是完全可以分离开去完成的，并且这两步不要求在同一个时间点上完成。 也就是说，Bean都是单例的，我们可以先把所有的单例Bean实例化出来，放到一个集合当中（我们可以称之为缓存），所有的单例Bean全部实例化完成之后，以后我们再慢慢的调用setter方法给属性赋值。这样就解决了循环依赖的问题。

Spring 容器启动后，会执行 AbstractAutowireCapableBeanFactory 中的 doCreateBean 方法进行 Bean 的创建
这里对Bean进行缓存，缓存的是创建Bean的工厂对象，从而对Bean进行曝光，使得需要使用该Bean的对象可以获取该Bean

这里对Bean进行缓存，缓存的是创建Bean的工厂对象，从而对Bean进行曝光，使得需要使用该Bean的对象可以获取该Bean
addSingletonFactory 为 DefaultSingletonBeanRegistry(默认单例Bean注册) 类中的方法
	在 DefaultSingletonBeanRegistry 类中有三个Map集合，即缓存Bean的三级缓存
        private final Map<String, Object> singletonObjects：一级缓存
        private final Map<String, Object> earlySingletonObjects：二级缓存
        private final Map<String, ObjectFactory<?>> singletonFactories：三级缓存
	Map集合的key存储的都是bean的name（bean id）：
        一级缓存存储的是：单例Bean对象。完整的单例Bean对象，也就是说这个缓存中的Bean对象的属性都已经赋值了。是一个完整的Bean对象。
        二级缓存存储的是：早期的单例Bean对象。这个缓存中的单例Bean对象的属性没有赋值。只是一个早期的实例对象。
        三级缓存存储的是：单例工厂对象。这个里面存储了大量的“工厂对象”，每一个单例Bean对象都会对应一个单例工厂对象。这个集合中存储的是，创建该单例对象时对应的那个单例工厂对象。

DefaultSingletonBeanRegistry 类中有 getSingleton 方法，用于获取单例 Bean 对象

从源码中可以看到，spring会先从一级缓存中获取Bean，如果获取不到，则从二级缓存中获取Bean，如果二级缓存还是获取不到，则从三级缓存中获取之前曝光的ObjectFactory对象，通过ObjectFactory对象获取Bean实例，这样就解决了循环依赖的问题。
缓存 Bean 对象之后，会在 AbstractAutowireCapableBeanFactory 中的 doCreateBean 方法中对 Bean 对象进行赋值
单例Bean实例对象的创建和赋值过程分开，提前进行Bean曝光，解决循环依赖问题

Spring只能解决setter方法注入的单例bean之间的循环依赖。ClassA依赖ClassB，ClassB又依赖ClassA，形成依赖闭环。Spring在创建ClassA对象后，不需要等给属性赋值，直接将其曝光到bean缓存当中。在解析ClassA的属性时，又发现依赖于ClassB，再次去获取ClassB，当解析ClassB的属性时，又发现需要ClassA的属性，但此时的ClassA已经被提前曝光加入了正在创建的bean的缓存中，则无需创建新的的ClassA的实例，直接从缓存中获取即可。从而解决循环依赖问题。
————————————————
版权声明：本文为CSDN博主「小炜1128」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/m0_53022813/article/details/128813298
```

# 6. 源码解析：三级缓存机制深入

## 6.1 三级缓存的数据结构

```
Spring 的三级缓存定义在 DefaultSingletonBeanRegistry 中：

public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry {

    // ========== 一级缓存：singletonObjects ==========
    // 存放"完全初始化完成"的单例 Bean（已经完成了属性赋值 + 初始化）
    // key: beanName, value: 完整的单例 Bean 对象
    // 这就是我们平时 getBean() 最优先查找的地方
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

    // ========== 二级缓存：earlySingletonObjects ==========
    // 存放"早期"的单例 Bean（已经实例化但还未完成属性赋值和初始化）
    // key: beanName, value: 早期的 Bean 实例
    // 注意：这里的对象可能已经经过了 SmartInstantiationAwareBeanPostProcessor
    //       的 getEarlyBeanReference() 处理（用于 AOP 代理的早期暴露）
    private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);

    // ========== 三级缓存：singletonFactories ==========
    // 存放"创建 Bean 的工厂"— ObjectFactory（函数式接口）
    // key: beanName, value: 一个 ObjectFactory，调用其 getObject() 就可以获得早期 Bean
    // ObjectFactory 的存在是为了延迟执行 getEarlyBeanReference()
    // 因为如果不需要处理循环依赖，就不需要提前创建代理对象
    private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

    // ========== 正在创建中的 Bean 名称集合 ==========
    // 用于检测循环依赖：当 getSingleton 发现 beanName 在这个 Set 中，说明正在创建
    private final Set<String> singletonsCurrentlyInCreation =
            Collections.newSetFromMap(new ConcurrentHashMap<>(16));

    // ========== 已注册的单例 Bean 名称集合 ==========
    // 存放所有已经注册的单例 bean 的名称（无论是否已完成创建）
    private final Set<String> registeredSingletons = new LinkedHashSet<>(256);
}
```

### 6.2 为什么需要三级缓存而不是两级？

关键问题：如果只有一级和二级缓存，直接存早期 Bean 对象即可，为什么还要加一层 ObjectFactory？

回答：因为 AOP + 循环依赖 的组合场景。

场景：A 和 B 循环依赖，且 A 需要 AOP 代理

如果 A 的早期引用直接就是一个原始对象（未代理），那么：
  B 注入的是原始 A（没有被 AOP 增强）
  最终 A 被代理后，AOP 增强对 B 中持有的 A 引用无效 → 不一致！

所以需要三级缓存中的 ObjectFactory：
  1. 将 A 的原始对象放入 ObjectFactory（延迟执行，此时先不决定是否要代理）
  2. B 需要 A 时，从三级缓存拿到 ObjectFactory，调用 getObject()
  3. getObject() 内部调用 getEarlyBeanReference() → 遍历所有
     SmartInstantiationAwareBeanPostProcessor
  4. AnnotationAwareAspectJAutoProxyCreator（实现了该接口）检查 A 是否需要代理
  5. 如果需要代理，返回代理对象；不需要就返回原始对象
  6. 将结果放入二级缓存，移除三级缓存

这样保证了：B 中注入的 A 和最终暴露的 A 是同一个代理对象！

## 6.2 从缓存获取 Bean：getSingleton() 源码逐行解读

```
DefaultSingletonBeanRegistry.getSingleton(String beanName, boolean allowEarlyReference)：

/**
 * @param beanName         要获取的 bean 名称
 * @param allowEarlyReference 是否允许从二级和三级缓存获取早期引用
 *                            true：完整流程（解决循环依赖时使用）
 *                            false：只查一级缓存（确保取到的是完全初始化好的 Bean）
 */
@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // ★ 第一步：查一级缓存 singletonObjects
    //   如果这里能取到，说明 Bean 已经完全初始化好了，直接返回
    Object singletonObject = this.singletonObjects.get(beanName);

    // ★ 如果一级缓存没有，并且这个 Bean 正在创建中（在 singletonsCurrentlyInCreation 中）
    //   说明发生了循环依赖 —— A 需要 B，B 又回头需要 A
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        // ★ 第二步：查二级缓存 earlySingletonObjects
        //   二级缓存存放的是"实例化后但未完全初始化"的早期 Bean
        singletonObject = this.earlySingletonObjects.get(beanName);

        if (singletonObject == null && allowEarlyReference) {
            // 锁住一级缓存，开始从三级缓存中获取
            synchronized (this.singletonObjects) {
                // 双重检查（可能其他线程已经完成创建并放入了一级缓存）
                singletonObject = this.singletonObjects.get(beanName);
                if (singletonObject == null) {
                    singletonObject = this.earlySingletonObjects.get(beanName);
                    if (singletonObject == null) {
                        // ★★★ 第三步：查三级缓存 singletonFactories ★★★
                        //   取出 ObjectFactory（它是一个 lambda，包含 getEarlyBeanReference 调用）
                        ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                        if (singletonFactory != null) {
                            // 调用 ObjectFactory.getObject() 获取早期 Bean
                            // 内部执行：getEarlyBeanReference(beanName, mbd, bean)
                            // → 遍历 SmartInstantiationAwareBeanPostProcessor
                            // → 如果该 Bean 有 AOP 切面，返回代理对象
                            singletonObject = singletonFactory.getObject();

                            // ★ 将早期 Bean 放入二级缓存
                            //   这样后续再有循环依赖需要这个 Bean 时，直接从二级缓存拿
                            this.earlySingletonObjects.put(beanName, singletonObject);

                            // ★ 移除三级缓存
                            //   工厂已经用过了，不再需要（避免重复调用 getObject()）
                            this.singletonFactories.remove(beanName);
                        }
                    }
                }
            }
        }
    }
    return singletonObject;
}
```

### 6.3 流程图：getSingleton 的三级缓存查询

```
                  getSingleton(beanName, true)
                          │
                          ▼
              ┌──────────────────────┐
              │  1. 查一级缓存        │
              │  singletonObjects    │
              └──────┬───────────────┘
                     │
          ┌──────────┼──────────┐
          │ 命中      │          │ 未命中
          ▼           │          ▼
      直接返回         │   isSingletonCurrentlyInCreation(beanName)?
      完整Bean         │          │
                      │   ┌──────┴──────┐
                      │   │ 否           │ 是
                      │   ▼              ▼
                      │ 返回null    ┌──────────────┐
                      │             │ 2. 查二级缓存  │
                      │             │ earlySingleton│
                      │             │ Objects       │
                      │             └──┬────────────┘
                      │                │
                      │       ┌────────┼────────┐
                      │       │ 命中    │        │ 未命中
                      │       ▼         │        ▼
                      │    直接返回      │  ┌─────────────────┐
                      │   早期Bean       │  │ 3. 查三级缓存     │
                      │                 │  │ singletonFactories│
                      │                 │  └──────┬────────────┘
                      │                 │         │
                      │                 │  ┌──────┼──────┐
                      │                 │  │ 命中  │      │ 未命中
                      │                 │  ▼       │      ▼
                      │                 │ factory  │   返回null
                      │                 │ .getObj()│   (最终异常)
                      │                 │    │     │
                      │                 │    ▼     │
                      │                 │ 得到早期Bean│
                      │                 │    │     │
                      │                 │    ▼     │
                      │                 │ 放入二级缓存│
                      │                 │ 移除三级缓存│
                      │                 │    │     │
                      │                 │    ▼     │
                      │                 │ 返回早期  │
                      │                 │ Bean     │
                      └─────────────────┴──────────┘
```

## 6.3 加入缓存：addSingletonFactory() 和 addSingleton()

```
// =========== addSingletonFactory: 将 Bean 的 ObjectFactory 放入三级缓存 ===========
// 调用时机：doCreateBean() 中，实例化完成后、属性赋值之前
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(singletonFactory, "Singleton factory must not be null");
    synchronized (this.singletonObjects) {
        // 只有一级缓存中还没有这个 Bean 时，才放入三级缓存
        if (!this.singletonObjects.containsKey(beanName)) {
            // ★ 放入三级缓存
            this.singletonFactories.put(beanName, singletonFactory);
            // ★ 从二级缓存中移除（以防之前就存在）
            this.earlySingletonObjects.remove(beanName);
            // ★ 记录这个 beanName 已经注册了
            this.registeredSingletons.add(beanName);
        }
    }
}

// =========== addSingleton: 将完全初始化好的 Bean 放入一级缓存 ===========
// 调用时机：Bean 完全创建完成后
protected void addSingleton(String beanName, Object singletonObject) {
    synchronized (this.singletonObjects) {
        // ★ 放入一级缓存
        this.singletonObjects.put(beanName, singletonObject);
        // ★ 从三级缓存中移除（已经不需要工厂了）
        this.singletonFactories.remove(beanName);
        // ★ 从二级缓存中移除（已经不需要早期引用了）
        this.earlySingletonObjects.remove(beanName);
        // ★ 从"正在创建"集合中移除
        this.singletonsCurrentlyInCreation.remove(beanName);
        // ★ 标记为已注册
        this.registeredSingletons.add(beanName);
    }
}
```

## 6.4 完整流程时序：Husband 和 Wife 循环依赖如何解决

```
假设先创建 Husband，Husband 构造器注入 name，setter 注入 Wife：

=== Step 1: getBean("husband") ===
doGetBean("husband") 
  → getSingleton("husband", true) → 三个缓存都没有 → 返回 null
  → 需要创建 Husband

=== Step 2: createBean → doCreateBean("husband") ===
  → createBeanInstance() → 反射调用 new Husband() ✅
  → addSingletonFactory("husband", () → getEarlyBeanReference())
     ★ Husband 被放入三级缓存（此时只有一个空 name、wife=null 的 Husband 对象）

=== Step 3: populateBean("husband") ===
  对 <property name="name" value="张三"/> → 直接注入 "张三" ✅
  对 <property name="wife" ref="wife"/> → 需要 wife Bean！
     → getBean("wife")

=== Step 4: getBean("wife") ===
doGetBean("wife")
  → getSingleton("wife", true) → 三个缓存都没有 → 返回 null
  → 需要创建 Wife

=== Step 5: createBean → doCreateBean("wife") ===
  → createBeanInstance() → 反射调用 new Wife() ✅
  → addSingletonFactory("wife", () → getEarlyBeanReference())
     ★ Wife 被放入三级缓存

=== Step 6: populateBean("wife") ===
  对 <property name="name" value="李四"/> → 直接注入 "李四" ✅
  对 <property name="husband" ref="husband"/> → 需要 husband Bean！
     → getBean("husband")

=== Step 7: getBean("husband") 【第二次】 ===
doGetBean("husband")
  → getSingleton("husband", true)
     ★ 一级缓存：没有（Husband 还没完全初始化）
     ★ 但 isSingletonCurrentlyInCreation("husband") = true！（Husband 在 Step 2 被标记）
     ★ 二级缓存：没有
     ★ 三级缓存：有！— 拿到 Step 2 放入的 ObjectFactory
       → factory.getObject() → getEarlyBeanReference()
       → 返回 Husband 的早期引用

  → Wife 得到 Husband 的早期引用，setHusband() 注入 ✅

=== Step 8: Wife 初始化完成 ===
  → Wife.initializeBean() → after/init/before ✅
  → addSingleton("wife", wifeObject)
     ★ Wife 进入一级缓存
     ★ Wife 从二级/三级缓存移除

=== Step 9: 回到 Husband 的 populateBean ===
  → getBean("wife") 返回完整的 Wife 对象 ✅
  → Husband.setWife(wife) ✅

=== Step 10: Husband 初始化完成 ===
  → Husband.initializeBean() → after/init/before ✅
  → addSingleton("husband", husbandObject)
     ★ Husband 进入一级缓存

=== 最终结果 ===
一级缓存：
  husband → 完整 Husband{name='张三', wife=Wife{name='李四', husband=Husband{...}}}
  wife    → 完整 Wife{name='李四', husband=Husband{name='张三', wife=Wife{...}}}
```

## 6.5 为什么构造注入不能解决循环依赖

```
构造注入无法解决的根本原因在于：实例化和属性赋值无法分离。

Spring doCreateBean() 的流程是：
  1. createBeanInstance() — 创建实例
     ★ 如果是构造注入，这里就需要传入参数！
     ★ 参数包括另一个 Bean 的引用（如 new Husband(name, wife)）
     ★ 但另一个 Bean 的创建也需要当前 Bean 的引用
     ★ 形成死锁——谁也无法完成 step 1
  2. addSingletonFactory() — 曝光
     ★ 构造注入时，step 1 都完成不了，根本走不到 step 2
     ★ 没有曝光就没有早期引用，循环依赖无法解决

setter 注入之所以能解决：
  1. createBeanInstance() — 无参构造，轻松完成 ✅
  2. addSingletonFactory() — 立即曝光 ✅
  3. populateBean() — 需要另一个 Bean 时，另一个 Bean 也在 step 2 阶段
     ★ 其他 Bean 可以从三级缓存拿到当前 Bean 的早期引用 ✅

源码证据：
AbstractAutowireCapableBeanFactory.createBeanInstance() 中有专门处理构造注入的逻辑：

Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
// 如果 ctors 不为空（说明存在 @Autowired 构造器）或者 mbd 的 autowireMode 是构造注入
if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
        mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
    // 走构造注入分支：instantiateBean() → 调用带参构造方法
    // ★ 这会触发 resolveConstructorArguments() → resolveAutowiredArgument()
    //   → resolveDependency() → getBean() → 又要创建另一个 Bean
    return autowireConstructor(beanName, mbd, ctors, args);
}

// 否则走默认无参构造
return instantiateBean(beanName, mbd);
```

## 6.6 getEarlyBeanReference() — AOP 代理的提前曝光

```
在 addSingletonFactory() 中传入的 ObjectFactory 实际上调用了 getEarlyBeanReference()：

protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
    Object exposedObject = bean;
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (SmartInstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().smartInstantiationAware) {
            // ★ 遍历所有 SmartInstantiationAwareBeanPostProcessor
            //   AnnotationAwareAspectJAutoProxyCreator 就是其中之一
            exposedObject = bp.getEarlyBeanReference(exposedObject, beanName);
        }
    }
    return exposedObject;
}

AnnotationAwareAspectJAutoProxyCreator.getEarlyBeanReference() 的核心逻辑：
  → 如果这个 Bean 需要 AOP 代理，返回代理对象
  → 如果不需要代理，返回原始对象

这就是为什么即使有循环依赖，AOP 代理也不会失效：
  B 通过 getEarlyBeanReference 拿到的 A 已经是代理对象了
  后续 A 完成创建后，exposedObject 最终还是指向同一个代理对象
```

## 6.7 设计思想总结

1. 三级缓存的设计模式：类似 CPU 的多级缓存
   一级(L1)：最快，完整 Bean
   二级(L2)：早期 Bean，已从三级升级
   三级(L3)：ObjectFactory，延迟创建，用完即升到二级

2. 为什么用 ObjectFactory 而不是直接存早期对象？
   → 延迟执行原则（Lazy Evaluation）
   → 如果对象不需要被循环依赖，就不会执行 getEarlyBeanReference()
   → 避免了为所有 Bean 都创建不必要的早期代理

3. 扩展点：SmartInstantiationAwareBeanPostProcessor.getEarlyBeanReference()
   → AOP 通过这个钩子保证了循环依赖场景下代理的一致性

4. Spring 对构造注入循环依赖不处理的哲学：
   → 构造注入的设计意图是"创建时就必须给全所有依赖"
   → 这本身就暗示了不应该存在构造注入的循环依赖
   → 如果存在，那是设计问题，应该重构
