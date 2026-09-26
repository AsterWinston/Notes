# 1. 入门程序分析
```
在spring的配置文件中，每个bean的id是不能重名，因为id是每个bean的唯一表示，用于通过容器对象获取相应的bean

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
	<bean id="userBean" class="cw.spring.study.pojo.User" />
	<bean id="userBean" class="cw.spring.study.pojo.User" />
</beans>

org.springframework.beans.factory.parsing.BeanDefinitionParsingException: 
	Configuration problem: Bean name 'userBean' is already used in this <beans> element
Offending resource: class path resource [spring.xml]

spring是通过反射机制调用类的无参数构造方法来创建对象的，所以要想让spring给你创建对象，必须保证无参数构造方法是存在的。

public class User {
    private String name;
    private Integer age;
    
    public User() {
        System.out.println("User 无参构造方法执行...");
    }
    ...
}

spring把创建好的对象存储到一个Map集合当中

bean对象最终存储在spring容器中，在spring源码底层就是一个map集合，存储bean的map在DefaultListableBeanFactory类中

Spring容器加载到Bean类时 , 会把这个类的描述信息, 以包名加类名的方式存到beanDefinitionMap 中,
Map<String,BeanDefinition> , 其中
String是Key , 默认是类名首字母小写，bean的id
BeanDefinition , 存的是类的定义(描述信息) , 我们通常叫BeanDefinition接口为 : bean的定义对象。

public class DefaultListableBeanFactory 
	extends AbstractAutowireCapableBeanFactory 
	implements ConfigurableListableBeanFactory, BeanDefinitionRegistry, Serializable {
    	...
        private final Map<String, BeanDefinition> beanDefinitionMap;
        ...
}

spring配置文件名字是我们负责提供的，spring配置文件的名字是随意的
因为在Spring程序中获取Spring配置文件创建Spring容器中，Spring配置文件是由我们进行指定的，所以spring配置文件的名字可以是任意的
spring的配置文件可以有多个，在ClassPathXmlApplicationContext构造方法的参数上传递文件路径即可。且配置文件的文件名、文件路径都是任意的

//ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring6.xml");
//ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring6.xml", "beans.xml");
ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring6.xml", "beans.xml", "xml/beans.xml");

// ClassPathXmlApplicationContext的有参构造
public ClassPathXmlApplicationContext(String... configLocations) throws BeansException {
    this(configLocations, true, (ApplicationContext)null);
}

在spring配置文件中配置的bean可以是任意类，只要这个类不是抽象的，并且提供了无参数构造方法。
getBean()方法调用时，id不存在的时候，会出现异常

getBean()方法返回的类型是Object，如果访问子类的特有属性和方法时，还需要向下转型，有其它办法可以解决这个问题吗？
//Object nowTime = applicationContext.getBean("nowTime");
//Date nowTime = (Date) applicationContext.getBean("nowTime");
// 不想强制类型转换，可以使用以下代码（通过第二个参数来指定返回的bean的类型。）
Date nowTime = applicationContext.getBean("nowTime", Date.class);

ClassPathXmlApplicationContext是从类路径中加载配置文件，如果没有在类路径当中，又应该如何加载配置文件呢？
没有在类路径中的话，需要使用FileSystemXmlApplicationContext类进行加载配置文件。
从指定的文件路径加载配置文件
这种方式较少用。一般都是将配置文件放到类路径当中，这样可移植性更强。

使用示例：
ApplicationContext applicationContext2 = new FileSystemXmlApplicationContext("d:/spring6.xml");

ApplicationContext的超级父接口BeanFactory。

BeanFactory是Spring容器的超级接口。
ApplicationContext是BeanFactory的子接口。

//ApplicationContext接口的超级父接口是：BeanFactory
//BeanFactory翻译为Bean工厂，就是能够生产Bean对象的一个工厂对象
//BeanFactory是IoC容器的顶级接口。
//Spring的IoC容器底层实际上使用了：工厂模式。
//Spring底层的IoC是怎么实现的？XML解析 + 工厂模式 + 反射机制

//ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring6.xml");
BeanFactory beanFactory = new ClassPathXmlApplicationContext("spring6.xml");
User user = applicationContext.getBean("userBean", User.class);
System.out.println(user);

Spring底层的IoC容器是通过：XML解析+工厂模式+反射机制实现的。
spring不是在调用getBean()方法的时候创建对象，执行new ClassPathXmlApplicationContext("spring6.xml");加载配置文件的时候，就会实例化对象。
```

# 源码解析：ClassPathXmlApplicationContext 初始化链路

## 1. 构造器调用链

```
new ClassPathXmlApplicationContext("spring.xml")
    │
    ├── 调用 ClassPathXmlApplicationContext(String... configLocations)
    │     configLocations = [ "spring.xml" ]
    │
    ├── 调用 ClassPathXmlApplicationContext(String[], boolean, ApplicationContext)
    │     参数: configLocations, refresh=true, parent=null
    │
    ├── super(parent) → 初始化父容器 AbstractApplicationContext
    │     - 初始化 this.beanFactoryPostProcessors = new ArrayList<>()
    │     - 初始化 this.applicationListeners = new LinkedHashSet<>()
    │     - 初始化 this.startupShutdownMonitor = new Object()
    │
    ├── setConfigLocations(configLocations) → 存储配置文件路径
    │     this.configLocations = [ "spring.xml" ]
    │
    └── if (refresh) { ★ refresh(); } ← 触发容器刷新
```

## 2. refresh() → obtainFreshBeanFactory()

```
refresh() 是 AbstractApplicationContext 的核心方法，包含 13 个子步骤。
obtainFreshBeanFactory() 是第 2 步，负责创建 BeanFactory 并加载 Bean 定义：

protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    refreshBeanFactory();  // ★ 核心：创建 + 加载
    return getBeanFactory();
}

// AbstractRefreshableApplicationContext.refreshBeanFactory()
protected final void refreshBeanFactory() throws BeansException {
    if (hasBeanFactory()) {
        destroyBeans();       // 销毁旧 Bean
        closeBeanFactory();   // 关闭旧 BeanFactory
    }
    try {
        // ★ 创建 DefaultListableBeanFactory（核心 IoC 容器）
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        beanFactory.setSerializationId(getId());

        // 定制 BeanFactory（如 allowBeanDefinitionOverriding、allowCircularReferences）
        customizeBeanFactory(beanFactory);

        // ★★★ 加载 Bean 定义 ★★★
        // 这里会解析 spring.xml，将 <bean> 标签注册为 BeanDefinition
        loadBeanDefinitions(beanFactory);

        this.beanFactory = beanFactory;
    } catch (IOException ex) {
        throw new ApplicationContextException("I/O error parsing XML", ex);
    }
}

// AbstractXmlApplicationContext.loadBeanDefinitions()
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws IOException {
    // 创建 XmlBeanDefinitionReader（XML 解析器）
    XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

    // 设置环境、资源加载器等
    beanDefinitionReader.setEnvironment(this.getEnvironment());
    beanDefinitionReader.setResourceLoader(this);
    beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

    // 初始化 BeanDefinitionReader（留空，子类可扩展）
    initBeanDefinitionReader(beanDefinitionReader);

    // ★ 加载 spring.xml 配置文件中的所有 BeanDefinition
    loadBeanDefinitions(beanDefinitionReader);
}
```

## 3. getBean() — 如何从容器中获取 Bean

```
applicationContext.getBean("user", User.class)
    │
    └── AbstractApplicationContext.getBean(String, Class<T>)
         └── getBeanFactory().getBean(name, requiredType)
              └── AbstractBeanFactory.getBean(String, Class<T>)
                   └── doGetBean(name, requiredType, null, false)
                        │
                        ├── transformedBeanName("user") → "user"（处理 & 前缀和别名）
                        ├── getSingleton("user") → 首次返回 null（Bean 尚未创建）
                        │     │
                        │     └── 查一级缓存 singletonObjects.get("user") → null
                        │         （因为 Bean 是在 refresh → finishBeanFactoryInitialization
                        │           → preInstantiateSingletons 中预创建的）
                        │
                        └── 如果缓存命中，直接返回
                            如果缓存未命中（prototype 等其他情况），调用 createBean()

注意：对于 singleton + 非 lazy-init 的 Bean，在 getBean() 时已经创建完成，
所以 getSingleton() 直接从一级缓存中返回，不会再走 createBean()。
```

## 4. DefaultListableBeanFactory 的存储结构

```
public class DefaultListableBeanFactory extends AbstractAutowireCapableBeanFactory
        implements ConfigurableListableBeanFactory, BeanDefinitionRegistry, Serializable {

    // ★ Bean 定义的存储：beanName → BeanDefinition
    //   在 loadBeanDefinitions() 阶段填充
    private final Map<String, BeanDefinition> beanDefinitionMap =
            new ConcurrentHashMap<>(256);

    // ★ beanName 列表（保持注册顺序）
    private volatile List<String> beanDefinitionNames =
            new ArrayList<>(256);

    // ★ 单例 Bean 的存储：beanName → 完整 Bean 实例
    //   继承自 DefaultSingletonBeanRegistry
    // private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

    // ★ 别名 → beanName 的映射
    //   继承自 SimpleAliasRegistry
    // private final Map<String, String> aliasMap = new ConcurrentHashMap<>(16);
}
```

## 5. BeanDefinition 是什么

```
<bean id="user" class="cw.spring.study.pojo.User"> 
    <property name="name" value="张三"/> 
</bean>

在 XML 加载完成后，这个 <bean> 元素被解析为一个 BeanDefinition：

BeanDefinition 对象包含：
{
    beanClassName:   "cw.spring.study.pojo.User"   ← class 属性
    scope:           "singleton"                    ← 默认
    lazyInit:        false                          ← 默认
    dependsOn:       null                           ← depends-on 属性
    autowireMode:    0 (AUTOWIRE_NO)               ← autowire 属性
    initMethodName:  null                           ← init-method 属性
    destroyMethodName: null                         ← destroy-method 属性
    propertyValues: {                                ← <property> 子元素
        "name" → new TypedStringValue("张三")
    }
    constructorArgumentValues: {}                    ← <constructor-arg> 子元素
}
```

## 6. 初始化时序总结

```
new ClassPathXmlApplicationContext("spring.xml")
    │
    ├── 1. 调用父类构造器，初始化上下文环境
    │
    ├── 2. setConfigLocations(["spring.xml"]) — 保存配置路径
    │
    ├── 3. refresh() 触发 13 步容器刷新
    │      │
    │      ├── 3.2 obtainFreshBeanFactory()
    │      │      ├── 创建 DefaultListableBeanFactory
    │      │      ├── 创建 XmlBeanDefinitionReader
    │      │      └── ★ 加载 spring.xml → 注册所有 BeanDefinition
    │      │           └── beanDefinitionMap.put("user", beanDef)
    │      │
    │      ├── 3.5 invokeBeanFactoryPostProcessors()
    │      │      └── 执行 BeanFactoryPostProcessor（修改 BeanDefinition）
    │      │
    │      ├── 3.6 registerBeanPostProcessors()
    │      │      └── 注册 BeanPostProcessor
    │      │
    │      ├── 3.11 ★ finishBeanFactoryInitialization()
    │      │      └── preInstantiateSingletons()
    │      │           └── 遍历 beanDefinitionNames
    │      │                └── getBean("user")
    │      │                     ├── 查缓存 → 没有
    │      │                     ├── createBean("user", beanDef)
    │      │                     │    ├── createBeanInstance() → new User() ★
    │      │                     │    ├── populateBean() → setXxx() ★
    │      │                     │    └── initializeBean() → init
    │      │                     └── addSingleton("user", userObj)
    │      │                          └── singletonObjects.put("user", userObj) ★
    │      │
    │      └── 3.12 finishRefresh() — 发布 ContextRefreshedEvent
    │
    └── 容器就绪！

applicationContext.getBean("user")
    └── ★ 直接从 singletonObjects 缓存返回，无需再次创建
```
