Spring还可以完成自动化的注入，自动化注入又被称为自动装配。
它可以根据名字进行自动装配，也可以根据类型进行自动装配。
# 1. 根据名称自动装配
```
如果根据名称装配(byName)，底层会调用set方法进行注入。
例如：setAge() 对应的名字是age，setPassword()对应的名字是password，setEmail()对应的名字是email。

public class OrderDao {

    private static final Logger logger = LoggerFactory.getLogger(OrderDao.class);

    public void insert(){
        logger.info("订单正在生成....");
    }
}

public class OrderService {

    private OrderDao orderDao;

    // 通过set方法给属性赋值。
    public void setOrderDao(OrderDao orderDao) {
        this.orderDao = orderDao;
    }

    /**
     * 生成订单的业务方法。。。
     */
    public void generate(){
        orderDao.insert();
    }
}

    <!--根据名字进行自动装配-->
    <!--注意：自动装配也是基于set方式实现的。-->
    <bean id="orderService" class="com.powernode.spring6.service.OrderService" autowire="byName"></bean>

    <!--id一般也叫作bean的名称。-->
    <!--根据名字进行自动装配的时候，被注入的对象的bean的id不能随便写，
            set方法的方法名去掉set，剩下单词首字母小写。-->
    <bean id="orderDao" class="com.powernode.spring6.dao.OrderDao"/>

    OrderService Bean中需要添加autowire=“byName”，表示通过名称自动进行装配。
    OrderService 类中有一个OrderDao属性，而OrderDao属性的名字是orderDao，对应的set方法是setOrderDao()，正好和OrderDao Bean的id是一样的，可以根据名称自动装配。

```
# 2. 根据类型自动装配
```
无论是byName还是byType，在装配的时候都是基于set方法的，所以set方法是必须要提供的，提供构造方法是不行的
如果byType，根据类型装配时，如果配置文件中有两个类型一样的bean会报错
当byType进行自动装配的时候，配置文件中某种类型的Bean必须是唯一的，不能出现多个。
如果存在多个类型一样的Bean，则Spring会不知道使用那个Bean进行注入

public class CustomerService {

    private UserDao userDao;
    private VipDao vipDao;

    public void setUserDao(UserDao userDao) {
        this.userDao = userDao;
    }

    public void setVipDao(VipDao vipDao) {
        this.vipDao = vipDao;
    }

    public void save(){
        userDao.insert();
        vipDao.insert();
    }

}

    <!--根据类型进行自动装配-->
    <!--自动装配是基于set方法的-->
    <!--根据类型进行自动装配的时候，在有效的配置文件当中，某种类型的实例只能有一个。-->
    <bean class="com.powernode.spring6.dao.VipDao"></bean>
    <bean id="x" class="com.powernode.spring6.dao.UserDao"></bean>
    <!--如果byType，根据类型装配时，如果配置文件中有两个类型一样的bean会报错-->
    <!--<bean id="y" class="com.powernode.spring6.dao.UserDao"></bean>-->
    <bean id="cs" class="com.powernode.spring6.service.CustomerService" autowire="byType"></bean>

```

# 3. 源码解析：XML 自动装配的底层实现

## 3.1 XML autowire 属性如何驱动自动装配

```
XML 中的 autowire="byName" 或 autowire="byType" 最终会被解析并存储在
BeanDefinition 的 autowireMode 属性中。

解析发生在 BeanDefinitionParserDelegate.parseBeanDefinitionAttributes()：

if (ele.hasAttribute(AUTOWIRE_ATTRIBUTE)) {
    String autowire = ele.getAttribute(AUTOWIRE_ATTRIBUTE);  // "byName" / "byType" / "constructor"
    bd.setAutowireMode(getAutowireMode(autowire));
}

getAutowireMode() 将字符串映射为 int 常量：
    "no"          → 0 (AUTOWIRE_NO)
    "byName"      → 1 (AUTOWIRE_BY_NAME)
    "byType"      → 2 (AUTOWIRE_BY_TYPE)
    "constructor" → 3 (AUTOWIRE_CONSTRUCTOR)
    "default"     → -1 → 去 <beans default-autowire="..."> 查找默认值
```

## 3.2 populateBean() 中的自动装配触发

```
在 doCreateBean() → populateBean() 中，会自动装配逻辑被触发：

protected void populateBean(String beanName, RootBeanDefinition mbd,
        @Nullable BeanWrapper bw) {

    // ... 其他属性注入逻辑

    // ★ 获取自动装配模式（从 XML autowire 属性而来）
    int resolvedAutowireMode = mbd.getResolvedAutowireMode();

    if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
        // 创建 PropertyValues 对象来存放自动装配的结果
        MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

        // ★ 自动装配 byName
        if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
            autowireByName(beanName, mbd, bw, newPvs);
        }

        // ★ 自动装配 byType
        if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
            autowireByType(beanName, mbd, bw, newPvs);
        }

        // 将自动装配的结果合并到 PropertyValues
        pvs = newPvs;
    }

    // ... 之后继续执行常规的 PropertyValues 注入
}
```

## 3.3 autowireByName() — 按名称自动装配源码

```
protected void autowireByName(
        String beanName, AbstractBeanDefinition mbd, BeanWrapper bw,
        MutablePropertyValues pvs) {

    // ★ 步骤 1: 获取所有"需要自动装配的"属性名
    //    通过 BeanWrapper 分析目标类的 PropertyDescriptor
    //    返回所有具有 setter 方法的属性名数组
    //    例如 OrderService 中有 setOrderDao()，返回 ["orderDao"]
    String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);

    // ★ 步骤 2: 遍历每个属性名
    for (String propertyName : propertyNames) {
        // 检查容器中是否有 id/name 等于 propertyName 的 Bean
        if (containsBean(propertyName)) {
            // ★ 从容器中获取该 Bean
            Object bean = getBean(propertyName);

            // ★ 将属性名和对应的 Bean 封装为 PropertyValue
            //    例如：propertyName="orderDao", value=OrderDao实例
            pvs.add(propertyName, bean);

            // ★ 注册依赖关系（用于确定销毁顺序）
            registerDependentBean(propertyName, beanName);
        } else {
            // beanName 在容器中找不到匹配的 Bean，跳过
        }
    }
}

// unsatisfiedNonSimpleProperties() 的实现：
// 1. 遍历 BeanWrapper 中所有的 PropertyDescriptor
// 2. 过滤出"可写"的属性（有 setter 方法）
// 3. 过滤出"非简单类型"的属性（排除 String、int 等基本类型）
// 4. 过滤出"尚未被显式配置"的属性（即 XML 中 <property> 标签已配置的跳过）
// 5. 返回最终需要自动装配的属性名数组
```

## 3.4 autowireByType() — 按类型自动装配源码

```
protected void autowireByType(
        String beanName, AbstractBeanDefinition mbd, BeanWrapper bw,
        MutablePropertyValues pvs) {

    // 获取类型转换器
    TypeConverter converter = getCustomTypeConverter();
    if (converter == null) {
        converter = bw;
    }

    // ★ 步骤 1: 获取所有需要自动装配的属性名（同上）
    String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);

    for (String propertyName : propertyNames) {
        // 根据属性名获取属性的目标类型
        PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
        Class<?> propertyType = pd.getPropertyType();

        if (Object.class != propertyType) {
            // ★ 步骤 2: 根据类型从容器中查找匹配的 Bean
            //    调用 DependencyDescriptor.resolveDependency()
            MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
            // 不允许 eager init（防止意外的循环依赖）
            boolean eager = !(bw.getWrappedInstance() instanceof PriorityOrdered);
            DependencyDescriptor desc = new AutowireByTypeDependencyDescriptor(
                    methodParam, eager);

            // ★ 核心方法：按类型解析依赖
            //   内部会遍历所有匹配类型的 Bean：
            //   - 如果只有一个匹配 → 直接返回
            //   - 如果有多个匹配 → 根据 @Primary 或 @Priority 或名称匹配来决定
            //   - 如果找不到 → 根据 required 标记决定是否抛异常
            Object autowiredArgument = resolveDependency(desc, beanName,
                    autowiredBeanNames, converter);

            if (autowiredArgument != null) {
                pvs.add(propertyName, autowiredArgument);
            }

            // 注册依赖关系
            for (String autowiredBeanName : autowiredBeanNames) {
                registerDependentBean(autowiredBeanName, beanName);
            }
            autowiredBeanNames.clear();
        }
    }
}
```

## 3.5 自动装配与显式注入的执行顺序

```
在 populateBean() 中，执行顺序是：

1. hasInstantiationAwareBeanPostProcessors?
   → autowireByType → resolveDependency() 遍历候选 Bean
     ↓
   → autowireByName → getBean() 直接获取
     ↓
   → 生成新的 MutablePropertyValues(newPvs)
     ↓

2. ★ 应用 BeanPostProcessor 的 postProcessProperties()
   （如 AutowiredAnnotationBeanPostProcessor 处理 @Autowired/@Value）

3. ★ 应用 PropertyValues（来自 XML 显式配置的 <property>）
   applyPropertyValues(beanName, mbd, bw, pvs)
     ↓
   这里 pvs 中：
   - 前面自动装配的结果在先
   - XML 显式配置的 <property> 在后
   - ★★★ 后设置的会覆盖先设置的 ★★★
   即：<property> 标签的显式配置优先级高于 autowire 自动装配！
```

## 3.6 设计思想总结

1. 约定优于配置（Convention over Configuration）：
   autowire="byName" 体现了这个思想——只要 Bean 的 id 和 setter 方法名匹配，
   就自动完成注入，减少显式配置。但这也带来了隐式依赖的风险（重命名导致失效）。

2. 显式覆盖约定：
   自动装配的结果可以被 <property> 显式配置覆盖，体现了"显式优先于隐式"的原则。

3. 自动装配的演进：
   XML autowire → @Autowired(byType) → @Resource(byName) → 构造方法自动推断
   Spring 的自动装配机制从 XML 的"少见"到注解的"常见"，
   反映了业界从"显式配置"到"约定 + 注解"的范式转变。
