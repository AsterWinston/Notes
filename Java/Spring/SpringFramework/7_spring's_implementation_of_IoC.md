# 1. IoC控制反转
控制反转是一种思想。
控制反转是为了降低程序耦合度，提高程序扩展力，达到OCP原则，达到DIP原则。
控制反转，反转的是什么？
将对象的创建权利交出去，交给第三方容器负责。
将对象和对象之间关系的维护权交出去，交给第三方容器负责。
控制反转思想的实现：DI（Dependency Injection，依赖注入）

# 2. 依赖注入
依赖注入实现了控制反转的思想。
Spring通过依赖注入的方式来完成Bean管理的。
Bean管理：Bean对象的创建，以及Bean对象中属性的赋值（或者叫做Bean对象之间关系的维护）。
依赖注入：
依赖指的是对象和对象之间的关联关系。
注入指的是一种数据传递行为，通过注入行为来让对象和对象产生关系。
依赖注入常见的实现方式包括两种：
第一种：set注入
第二种：构造注入

## 2.1 set注入
```
set注入，基于set方法实现的，底层会通过反射机制调用属性对应的set方法然后给属性赋值。
这种方式要求属性必须对外提供set方法。
public class UserDao {

    // 获取本类的日志记录器
    private static final Logger logger = LoggerFactory.getLogger(UserDao.class);

    public void insert(){
        // 使用一下log4j2日志框架
        logger.info("数据库正在保存用户信息。");
    }
}

public class VipDao {

    // 获取本类的日志记录器
    private static final Logger logger = LoggerFactory.getLogger(VipDao.class);

    public void insert(){
        logger.info("正在保存Vip信息！！！！");
    }
}

public class UserService {

    private UserDao userDao;
    private VipDao vipDao;

    public void setAbc(VipDao vipDao){
        this.vipDao = vipDao;
    }

    // set注入，必须提供一个set方法。
    // Spring容器会调用这个set方法，来给userDao属性赋值。
    // 我自己写一个set方法，不使用IDEA工具生成的。不符合javabean规范。
    // 至少这个方法是以set单词开始的。前三个字母不能随便写，必须是“set"
    /*public void setMySQLUserDao(UserDao xyz){
        this.userDao = xyz;
    }*/


    // 这个set方法是IDEA工具生成的，符合javabean规范。
    public void setUserDao(UserDao userDao) {
        this.userDao = userDao;
    }

    public void saveUser(){
        // 保存用户信息到数据库
        userDao.insert();
        vipDao.insert();
    }
}

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <!--配置dao-->
    <bean id="userDaoBean" class="com.powernode.spring6.dao.UserDao"/>

    <!--配置service-->
    <bean id="userServiceBean" class="com.powernode.spring6.service.UserService">
        <!-- 想让Spring调用对应的set方法，需要配置property标签 -->
        <!-- name属性值：set方法的方法名，去掉set，然后把剩下的单词首字母变小写。-->
        <!-- ref翻译为引用，英语单词：referencesref，其属性值指定的是要注入的bean的id。-->
        <!--<property name="mySQLUserDao" ref="userDaoBean"/>-->

        <!--set方法起名的时候，不要为难自己，按照规范来。所以一般情况下name位置写属性名就行了。-->
        <property name="userDao" ref="userDaoBean"/>

        <!--<property name="vipDao" ref="vipDaoBean"/>-->
				<!-- name属性值：set方法的方法名，去掉set，然后把剩下的单词首字母变小写。 -->
        <property name="abc" ref="vipDaoBean"/>

    </bean>

    <bean id="vipDaoBean" class="com.powernode.spring6.dao.VipDao"/>

</beans>

set 注入分析
set 注入的实现原理：
通过property标签获取到属性名：userDao
通过属性名推断出与该属性对应的set方法的方法名：setUserDao
通过反射机制调用setUserDao()方法给属性赋值
一般情况下，set方法的命名符合JavaBean规范，property标签的name属性的属性值就是属性名。
property标签的name属性的属性值是，setUserDao()方法名演变得到的，演变的规律是：
去掉set，然后剩下的部分首字母改为小写
setUsername() 演变为 username
setPassword() 演变为 password
setUserDao() 演变为 userDao
setUserService() 演变为 userService
property标签的ref属性的属性值是要注入的bean对象的id，通过ref属性来完成bean的装配，这是bean最简单的一种装配方式。
装配指的是：创建系统组件之间关联的动作
对于property标签来说，ref属性也可以采用标签的方式，不使用ref属性：

<bean id="userServiceBean" class="com.powernode.spring6.service.UserService">
  <property name="userDao">
    <ref bean="userDaoBean"/>
  </property>
</bean>

总结：set注入的核心实现原理：通过反射机制调用set方法来给属性赋值，让两个对象之间产生关系。
```

## 2.2 构造注入
```
核心原理：通过调用构造方法来给属性赋值。
与set注入相比，构造注入是在创建对象的同时进行注入，进行属性的赋值，而set注入是在对象创建之后。

public class CustomerService {

    private UserDao userDao;
    private VipDao vipDao;

    public CustomerService(UserDao userDao, VipDao vipDao) {
        this.userDao = userDao;
        this.vipDao = vipDao;
    }

    public void save(){
        userDao.insert();
        vipDao.insert();
    }

}

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="xxxx" class="com.powernode.spring6.dao.UserDao"/>

    <bean id="yyyy" class="com.powernode.spring6.dao.VipDao"/>

    <bean id="csBean3" class="com.powernode.spring6.service.CustomerService">
        <!--不指定下标，也不指定参数名，让spring自己做类型匹配。-->
        <!--这种方式实际上是根据类型进行注入的。spring会自动根据类型来判断把ref注入给相应的参数。-->
        <constructor-arg ref="yyyy"/>
        <constructor-arg ref="xxxx"/>
    </bean>

    <bean id="csBean2" class="com.powernode.spring6.service.CustomerService">
        <!--根据构造方法参数的名字进行注入。-->
				<!-- name属性值为要进行注入的构造函数的参数 -->
        <constructor-arg name="vipDao" ref="yyyy"/>
        <constructor-arg name="userDao" ref="xxxx"/>
    </bean>

    <bean id="csBean" class="com.powernode.spring6.service.CustomerService">
        <!--构造注入-->
        <!--
            index属性指定参数下标，第一个参数是0，第二个参数是1，第三个参数是2，以此类推。
            ref属性用来指定注入的bean的id
        -->
        <!--指定构造方法的第一个参数，下标是0-->
        <constructor-arg index="0" ref="xxxx"/>
        <!--指定构造方法的第二个参数，下标是1-->
        <constructor-arg index="1" ref="yyyy"/>
    </bean>

</beans>

通过构造方法进行注入有三种形式：
可以通过下标
可以通过参数名
也可以不指定下标和参数名，可以类型自动推断。
Spring在装配方面还是比较健壮的
```

# 3. 源码解析：XML 配置如何变为 BeanDefinition

## 3.1 从 XML 文件到 BeanDefinition 的完整链路

```
当我们执行 new ClassPathXmlApplicationContext("spring.xml") 时，
XML 配置文件如何被解析为 Spring 内部的 BeanDefinition？

完整链路：

ClassPathXmlApplicationContext 构造器
    │
    ├─ setConfigLocations("spring.xml") — 保存配置文件路径
    │
    └─ refresh() — 容器刷新
         │
         └─ obtainFreshBeanFactory()
              │
              └─ refreshBeanFactory()
                   │
                   └─ loadBeanDefinitions(beanFactory)
                        │
                        ├─ 创建 XmlBeanDefinitionReader
                        └─ reader.loadBeanDefinitions(configLocations)
                             │
                             ├─ ResourceLoader 将路径解析为 Resource 对象
                             │   (如 ClassPathResource("spring.xml"))
                             │
                             ├─ Resource 被包装为 EncodedResource（处理编码）
                             │
                             └─ XmlBeanDefinitionReader.loadBeanDefinitions(Resource)
                                  │
                                  ├─ InputStream inputStream = resource.getInputStream()
                                  ├─ InputSource inputSource = new InputSource(inputStream)
                                  └─ ★ doLoadBeanDefinitions(inputSource, resource)
                                       │
                                       ├─ 获取 XML Document 对象
                                       │   Document doc = doLoadDocument(inputSource, resource)
                                       │
                                       └─ ★ registerBeanDefinitions(doc, resource)
                                            │
                                            └─ DefaultBeanDefinitionDocumentReader
                                                 .registerBeanDefinitions(doc, readerContext)
```

## 3.2 XmlBeanDefinitionReader — XML 读取器

```
XmlBeanDefinitionReader 是 Spring 专门用来从 XML 文件读取 Bean 定义的 Reader：

public class XmlBeanDefinitionReader extends AbstractBeanDefinitionReader {

    // ★ 核心入口：加载 XML 文件中的所有 BeanDefinition
    public int loadBeanDefinitions(EncodedResource encodedResource)
            throws BeanDefinitionStoreException {

        try (InputStream inputStream = encodedResource.getResource().getInputStream()) {
            // 创建 SAX InputSource
            InputSource inputSource = new InputSource(inputStream);
            if (encodedResource.getEncoding() != null) {
                inputSource.setEncoding(encodedResource.getEncoding());
            }
            // ★ 执行加载
            return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
        } catch (IOException ex) {
            // ...
        }
    }

    protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
            throws BeanDefinitionStoreException {

        try {
            // ★ 步骤 1: XML → Document（使用 DOM 解析器）
            Document doc = doLoadDocument(inputSource, resource);
            // ★ 步骤 2: Document → BeanDefinition → 注册到 BeanFactory
            int count = registerBeanDefinitions(doc, resource);
            return count;
        } catch (...) { ... }
    }

    protected Document doLoadDocument(InputSource inputSource, Resource resource)
            throws Exception {
        // 使用 DocumentLoader 将 XML 解析为 Document 对象
        // 默认使用 DefaultDocumentLoader，底层是 JAXP 的 DocumentBuilder
        return this.documentLoader.loadDocument(
                inputSource, getEntityResolver(), this.errorHandler,
                getValidationModeForResource(resource), isNamespaceAware());
    }
}
```

## 3.3 DefaultBeanDefinitionDocumentReader — Document → BeanDefinition

```
DefaultBeanDefinitionDocumentReader 负责遍历 XML Document 中的每个元素：

public class DefaultBeanDefinitionDocumentReader
        implements BeanDefinitionDocumentReader {

    protected void doRegisterBeanDefinitions(Element root) {
        // 创建代理 reader（用于装饰模式的其他处理）
        BeanDefinitionParserDelegate parent = this.delegate;
        this.delegate = createDelegate(getReaderContext(), root, parent);

        // 预处理器：检查 <beans> 的 default-* 属性
        // default-lazy-init, default-autowire, default-init-method 等
        preProcessXml(root);

        // ★ 核心：解析 <beans> 下的所有子元素
        parseBeanDefinitions(root, this.delegate);

        // 后处理器
        postProcessXml(root);
        this.delegate = parent;
    }

    protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {

        // 检查 root 是否使用默认命名空间 http://www.springframework.org/schema/beans
        if (delegate.isDefaultNamespace(root)) {
            NodeList nl = root.getChildNodes();
            for (int i = 0; i < nl.getLength(); i++) {
                Node node = nl.item(i);
                if (node instanceof Element ele) {
                    if (delegate.isDefaultNamespace(ele)) {
                        // ★ 默认命名空间的元素（如 <bean>、<import>、<alias>）
                        parseDefaultElement(ele, delegate);
                    } else {
                        // ★ 自定义命名空间（如 <context:component-scan>、<aop:config>）
                        //    通过 NamespaceHandler 来处理
                        delegate.parseCustomElement(ele);
                    }
                }
            }
        } else {
            delegate.parseCustomElement(root);
        }
    }

    private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
        if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
            // <import resource="..."/>
            importBeanDefinitionResource(ele);
        } else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
            // <alias name="..." alias="..."/>
            processAliasRegistration(ele);
        } else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
            // ★★★ <bean> — 核心处理 ★★★
            processBeanDefinition(ele, delegate);
        } else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
            // <beans> 嵌套 — 递归处理
            doRegisterBeanDefinitions(ele);
        }
    }
}
```

## 3.4 processBeanDefinition() — 解析 <bean> 元素

```
protected void processBeanDefinition(
        Element ele, BeanDefinitionParserDelegate delegate) {

    // ★ 步骤 1: 将 XML 元素解析为 BeanDefinition
    //   这里会处理 <bean> 的所有属性（id, class, scope, lazy-init 等）
    //   以及子元素（<property>、<constructor-arg>、<lookup-method> 等）
    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);

    if (bdHolder != null) {
        // ★ 步骤 2: 装饰 BeanDefinition（如果有自定义属性）
        bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);

        try {
            // ★ 步骤 3: 注册到 DefaultListableBeanFactory 的 beanDefinitionMap
            BeanDefinitionReaderUtils.registerBeanDefinition(
                    bdHolder, getReaderContext().getRegistry());
        } catch (BeanDefinitionStoreException ex) {
            getReaderContext().error("Failed to register bean definition...", ele, ex);
        }

        // ★ 步骤 4: 发送注册事件（给监听器）
        getReaderContext().fireComponentRegistered(
                new BeanComponentDefinition(bdHolder));
    }
}
```

## 3.5 BeanDefinitionParserDelegate.parseBeanDefinitionElement() — 元素解析细节

```
// 这是将 XML <bean> 元素变成 BeanDefinition 的核心方法：

@Nullable
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, BeanDefinition containingBean) {

    // 1. ★ 解析 id 和 name 属性
    String id = ele.getAttribute(ID_ATTRIBUTE);  // "userDaoBean"
    String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);  // 别名，用逗号/分号分隔

    List<String> aliases = new ArrayList<>();
    if (StringUtils.hasLength(nameAttr)) {
        String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr,
                MULTI_VALUE_ATTRIBUTE_DELIMITERS);  // 逗号、分号、空格
        aliases.addAll(Arrays.asList(nameArr));
    }

    String beanName = id;
    if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
        beanName = aliases.remove(0);  // 没有 id 就用第一个别名
    }

    // 2. ★ 解析 class 和 parent 属性
    String className = null;
    if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
        className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
    }
    String parent = null;
    if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
        parent = ele.getAttribute(PARENT_ATTRIBUTE);
    }

    // 3. ★ 创建 AbstractBeanDefinition 子类（默认是 GenericBeanDefinition）
    AbstractBeanDefinition bd = createBeanDefinition(className, parent,
            classLoader, beanClassResolver);

    // 4. ★ 解析 <bean> 的各种属性
    parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
    //   内部解析：
    //   - scope / singleton (deprecated)
    //   - abstract
    //   - lazy-init
    //   - autowire (byName / byType / constructor / default)
    //   - autowire-candidate
    //   - primary
    //   - depends-on
    //   - init-method
    //   - destroy-method
    //   - factory-method
    //   - factory-bean

    // 5. ★ 解析 <bean> 的子元素
    parseBeanDefinitionChildren(ele, beanName, bd);
    //   内部处理：
    //   - <constructor-arg> → ConstructorArgumentValues
    //     - index, name, type, value, ref
    //   - <property> → MutablePropertyValues
    //     - name, value, ref
    //   - <meta> → BeanMetadataAttributeAccessor
    //   - <lookup-method> → MethodOverrides
    //   - <replaced-method> → MethodOverrides
    //   - <qualifier> → 限定符

    // 6. ★ 兜底：如果还是没有 beanName，自动生成
    if (!StringUtils.hasText(beanName)) {
        beanName = this.readerContext.generateBeanName(bd);
    }

    // 返回 BeanDefinitionHolder（beanName + aliases + BeanDefinition）
    return new BeanDefinitionHolder(bd, beanName, aliases);
}
```

## 3.6 <property> 标签如何解析为属性值

```
BeanDefinitionParserDelegate.parsePropertyElement():

public void parsePropertyElement(Element ele, BeanDefinition bd) {
    // 1. ★ 获取属性名
    String propertyName = ele.getAttribute(NAME_ATTRIBUTE);
    if (!StringUtils.hasLength(propertyName)) {
        error("Tag 'property' must have a 'name' attribute", ele);
        return;
    }

    // 2. ★ 解析属性值（递归处理）
    //    - 简单值：value="xxx" → new TypedStringValue("xxx")
    //    - Bean引用：ref="beanId" → new RuntimeBeanReference("beanId")
    //    - 内嵌Bean：<bean>...</bean> → 递归解析为 BeanDefinition
    //    - List/Set/Map/Props → 解析为相应的集合类型
    //    - null → new TypedStringValue(null)
    Object val = parsePropertyValue(ele, bd, propertyName);

    // 3. ★ 封装为 PropertyValue 并添加到 BeanDefinition
    PropertyValue pv = new PropertyValue(propertyName, val);
    parseMetaElements(ele, pv);  // 解析 <meta> 子元素
    pv.setSource(extractSource(ele));
    bd.getPropertyValues().addPropertyValue(pv);
}

// PropertyValue 在 populateBean() 阶段被 BeanWrapperImpl 使用：
// 1. 调用 BeanWrapperImpl.setPropertyValue(PropertyValue)
// 2. 内部通过 TypeConverter 转换值类型
// 3. 找到目标 setter 方法 → 反射调用 setXxx(value)
```

## 3.7 BeanDefinition 接口的核心属性

```
BeanDefinition 是 Spring 中描述 Bean 的元数据接口：

public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {

    // ===== 作用域 =====
    String SCOPE_SINGLETON = "singleton";     // 单例（Spring 中的常量值）
    String SCOPE_PROTOTYPE = "prototype";    // 原型
    void setScope(@Nullable String scope);
    String getScope();

    // ===== 懒加载 =====
    // true: 只有在第一次 getBean() 时才创建
    // false: 在容器启动时（refresh）就被创建
    void setLazyInit(boolean lazyInit);
    boolean isLazyInit();

    // ===== 依赖关系 =====
    // dependsOn: 确保依赖的 Bean 先于当前 Bean 被创建
    void setDependsOn(@Nullable String... dependsOn);
    String[] getDependsOn();

    // ===== 自动装配模式 =====
    int AUTOWIRE_NO = 0;            // 不自动装配
    int AUTOWIRE_BY_NAME = 1;       // 按名称自动装配（setter）
    int AUTOWIRE_BY_TYPE = 2;       // 按类型自动装配（setter）
    int AUTOWIRE_CONSTRUCTOR = 3;   // 构造方法自动装配
    void setAutowireMode(int autowireMode);

    // ===== 候选资格 =====
    // false: 该 Bean 不会被其他 Bean 的 @Autowired(byType) 注入
    void setAutowireCandidate(boolean autowireCandidate);

    // ===== 主候选 =====
    // true: 当多个同类型 Bean 存在时，该 Bean 被优先选择
    void setPrimary(boolean primary);

    // ===== 初始化/销毁方法 =====
    void setInitMethodName(@Nullable String initMethodName);
    void setDestroyMethodName(@Nullable String destroyMethodName);

    // ===== 工厂方法 =====
    void setFactoryBeanName(@Nullable String factoryBeanName);
    void setFactoryMethodName(@Nullable String factoryMethodName);

    // ===== 构造方法参数 =====
    ConstructorArgumentValues getConstructorArgumentValues();

    // ===== 属性值 =====
    MutablePropertyValues getPropertyValues();

    // ===== 其他 =====
    void setParentName(@Nullable String parentName);  // 父 BeanDefinition 的名称
    boolean isSingleton();
    boolean isPrototype();
    boolean isAbstract();           // 抽象 Bean（作为模板）
    String getBeanClassName();
    String getResourceDescription(); // 来源描述（如 spring.xml 的路径）
}
```

## 3.8 BeanDefinition 从 XML 解析到最终注册的完整流程图

```
spring.xml
    │
    ▼
┌─────────────────────────────────────────────────┐
│ XmlBeanDefinitionReader                        │
│   loadBeanDefinitions(Resource)                │
│     │                                            │
│     ├─ doLoadDocument()                          │
│     │   └→ Document (DOM 树)                     │
│     │                                            │
│     └─ registerBeanDefinitions(Document)         │
│         │                                        │
│         ▼                                        │
│ DefaultBeanDefinitionDocumentReader            │
│   doRegisterBeanDefinitions(Element root)      │
│     │                                            │
│     └─ parseBeanDefinitions(root)               │
│         │                                        │
│         ├── <import>  → importBeanDefinitionResource│
│         ├── <alias>   → 注册别名                 │
│         ├── <beans>   → 递归处理（嵌套配置）     │
│         │                                        │
│         └── <bean>    → processBeanDefinition()  │
│              │                                   │
│              ├─ delegate.parseBeanDefinitionElement()│
│              │    │                               │
│              │    ├─ 解析 id, name, class, parent │
│              │    ├─ 解析 scope, lazy-init, depends-on│
│              │    ├─ 解析 autowire, primary       │
│              │    ├─ 解析 init-method, destroy-method│
│              │    ├─ 解析 <constructor-arg>        │
│              │    │    └→ index, name, type, value, ref│
│              │    ├─ 解析 <property>               │
│              │    │    └→ name, value, ref, list, map│
│              │    └─ 创建 GenericBeanDefinition    │
│              │                                     │
│              ├─ decorateBeanDefinitionIfRequired() │
│              │   (处理自定义 XML 属性)             │
│              │                                     │
│              └─ BeanDefinitionReaderUtils          │
│                   .registerBeanDefinition()        │
│                   │                                │
│                   ▼                                │
│              DefaultListableBeanFactory            │
│                .beanDefinitionMap.put(beanName, bd)│
└─────────────────────────────────────────────────┘
```

## 3.9 设计思想总结

1. Document Object Model (DOM) 解析：
   Spring 使用 DOM（而非 SAX）来解析 XML 配置文件。
   原因：DOM 允许对节点随机访问，支持嵌套配置的递归解析。

2. 委托模式（BeanDefinitionParserDelegate）：
   解析逻辑被委托给专门的 Delegate 类，将"遍历 XML 树"和"解析单个元素"分离。

3. 策略模式（NamespaceHandler）：
   自定义命名空间（如 <context:>、<aop:>、<tx:>）的处理通过 NamespaceHandler 机制实现：
   - 每个命名空间对应一个 NamespaceHandler 实现
   - NamespaceHandler 内部为每个标签注册对应的 BeanDefinitionParser
   - 新增自定义命名空间只需实现新的 Handler 即可，完全符合 OCP 原则

4. BeanDefinition 的"中间层"设计：
   XML 解析的结果不是直接创建 Bean，而是生成 BeanDefinition。
   这个中间层使得：
   - 配置来源不限于 XML（也可以是注解、Java Config、Properties）
   - BeanDefinition 可以被 BeanFactoryPostProcessor 修改（在 Bean 创建前）
   - 支持配置继承（parent 属性）和抽象 Bean 定义
