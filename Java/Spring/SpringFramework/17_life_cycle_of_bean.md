# 1. 什么是生命周期
Bean 的生命周期就是：对象从创建开始到最终销毁的整个过程，Bean 对象从创建到销毁中经历了哪些过程
什么时候创建Bean对象？
创建Bean对象的前后会调用什么方法？
Bean对象什么时候销毁？
Bean对象的销毁前后调用什么方法？
Spring其实就是一个管理Bean对象的工厂，它负责对象的创建，对象的销毁等。

# 2. 为什么要知道Bean的生命周期
生命周期的本质是：在哪个时间节点上调用了哪个类的哪个方法。
我们需要充分的了解在这个生命线上，都有哪些特殊的时间节点，在这些特殊节点会调用哪个类的哪个方法。
我们可能需要在某个特殊的时间点上执行一段特定的代码，只有我们知道了特殊的时间节点都在哪，在该特殊节点会调用什么方法，我们才可以确定代码写到哪，这段代码就可以放到这个节点上，当生命线走到这里的时候，自然会被调用。

# 3. Bean的生命周期-5步
```
Bean生命周期的管理，可以参考Spring的源码：AbstractAutowireCapableBeanFactory类的doCreateBean()方法。
Bean生命周期可以粗略的划分为五大步：
第一步：实例化Bean，调用无参构造方法创建对象
第二步：Bean属性赋值，调用对象的set方法，进行set注入
第三步：初始化Bean，调用Bean的init方法（该方法需要自己写，然后进行配置）
第四步：使用Bean
第五步：销毁Bean，调用Bean的Destroy方法（该方法需要自己写，然后进行配置）
进行Bean的销毁，必须关闭Spring容器，只有关闭了Spring容器，Bean才会进行销毁

package cw.spring.study.pojo;

/**
 * ClassName: User
 * Package: cw.spring.study.pojo
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-26 17:21
 * @Version 1.0
 */
public class User {
    private String name;
    
    public User() {
        System.out.println("1. User 无参构造方法执行");
    }
    
    public void setName(String name) {
        this.name = name;
        System.out.println("2. User setName 方法执行");
    }
    
    public void init() {
        System.out.println("3. User init 方法执行，进行 Bean 的初始化");
    }
    
    public void destroy() {
        System.out.println("5. User destroy 方法执行，进行 Bean 的销毁");
    }
}

<!-- 
  手动指定Bean的初始化方法和销毁方法
    init-method 属性
        配置 Bean 的初始化方法
    destroy-method
        配置 Bean 的销毁方法
-->
<bean id="user" class="cw.spring.study.pojo.User" init-method="init" destroy-method="destroy">
  <property name="name" value="张三"/>
</bean>

@org.junit.Test
public void testUser() {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    User user = applicationContext.getBean("user", User.class);
    System.out.println("4. User Bean 的使用，" + user);
    // 进行Bean的销毁，必须关闭Spring容器，只有关闭了Spring容器，Bean才会进行销毁
    // ApplicationContext 接口中无关闭容器的方法，ClassPathXmlApplicationContext 中有声明定义
    ClassPathXmlApplicationContext context = (ClassPathXmlApplicationContext) applicationContext;
    // 关闭容器
    context.close();
}

```

# 4. Bean的生命周期-7步
```
Bean生命周期七步，比Bean生命周期五步，在初始化Bean的前和后，添加了两步。
在以上的5步中，第3步是初始化Bean，在初始化前和初始化后执行的方法为“Bean后处理器”的before和after方法，通过“Bean后处理器”我们可以在初始化Bean的前后添加需要执行的代码。
要添加“Bean后处理器”需要类实现BeanPostProcessor接口，并且重写before和after方法
Bean生命周期之7步：
第一步：实例化Bean
第二步：Bean属性赋值
第三步：执行“Bean后处理器”的before方法。
第四步：初始化Bean
第五步：执行“Bean后处理器”的after方法。
第六步：使用Bean
第七步：销毁Bean

创建 Bean 后处理器
package cw.spring.study.pojo;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanPostProcessor;

/**
 * ClassName: LogBeanPostProcessor
 * Package: cw.spring.study.pojo
 * Description:
 * 日志Bean后处理器，
 * 在Bean对象的初始化前后执行相应的代码
 *
 * @Author tcw
 * @Create 2023-05-27 10:26
 * @Version 1.0
 */
public class LogBeanPostProcessor implements BeanPostProcessor {
    /**
     * 初始化Bean之前执行的代码
     *
     * @param bean 刚创建的Bean
     * @param beanName Bean的名字
     * @return
     * @throws BeansException
     */
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("日志 Bean 后处理器的 before 方法执行...");
        // return 不要修改
        return BeanPostProcessor.super.postProcessBeforeInitialization(bean, beanName);
    }
    
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("日志 Bean 后处理器的 after 方法执行...");
        // return 不要修改
        return BeanPostProcessor.super.postProcessAfterInitialization(bean, beanName);
    }
}

配置 Bean 后处理器
在 Spring 的配置文件中配置 Bean 后处理器
Bean 后处理器配置完成后，将会作用于整个配置文件中每个 Bean 的生命周期过程中，如在spring.xml文件中配置的Bean后处理器将会作用于当前配置文件中所有的Bean。
<!-- 
  配置 Bean 后处理器
  将会作用于本配置文件中所有Bean对象的生命周期中 
-->
<bean class="cw.spring.study.pojo.LogBeanPostProcessor"/>

```

# 5. Bean的生命周期-10步
```
Bean生命周期十步比七步多的三步：
点位1：在“Bean后处理器”before方法之前，在Bean属性赋值之后（检查Bean是否是实现了Aware的相关接口，并设置相关依赖）
点位2：在“Bean后处理器”before方法之后，在Bean的初始化之前（检查Bean是否实现了InitializingBean接口，并调用接口方法）
点位3：使用Bean之后，或者说销毁Bean之前（检查Bean是否实现了DisposableBean接口，并调用接口方法）
添加的这三个点位的特点：都是在检查你这个Bean是否实现了某些特定的接口，如果实现了这些接口，则Spring容器会调用这个接口中的方法。
Bean生命周期-10步：
第一步：实例化Bean（构造方法）
第二步：Bean属性赋值（setter赋值）
第三步：检查Bean是否实现了Aware的相关接口，并设置相关依赖（点位1）
第四步：Bean后处理器before执行
第五步：检查是否实现了InitializingBean接口，并调用接口方法（点位2）
第六步：初始化Bean（init方法）
第七步：Bean后处理器after执行
第八步：使用Bean
第九步：检查是否实现了DisposableBean接口，并调用接口方法（点位3）
第十步：销毁Bean（destroy方法）

实现 Aware 相关接口
Aware相关的接口包括：BeanNameAware、BeanClassLoaderAware、BeanFactoryAware
当Bean实现了BeanNameAware，Spring会将Bean的名字传递给Bean。
当Bean实现了BeanClassLoaderAware，Spring会将加载该Bean的类加载器传递给Bean。
当Bean实现了BeanFactoryAware，Spring会将Bean工厂对象传递给Bean。
在“Bean后处理器”before方法之前，会检查Bean是否实现了Aware相关的接口，如果实现了接口则调用这些接口中的方法。然后调用这些方法的目的是为了给你传递一些数据，让你更加方便使用。

package cw.spring.study.pojo;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.BeanClassLoaderAware;
import org.springframework.beans.factory.BeanFactory;
import org.springframework.beans.factory.BeanFactoryAware;
import org.springframework.beans.factory.BeanNameAware;

/**
 * ClassName: User
 * Package: cw.spring.study.pojo
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-26 17:21
 * @Version 1.0
 */
public class User implements BeanNameAware, BeanClassLoaderAware, BeanFactoryAware {
    private String name;
    
    public User() {
        System.out.println("1. User 无参构造方法执行");
    }
    
    public void setName(String name) {
        this.name = name;
        System.out.println("2. User setName 方法执行");
    }
    
    public void init() {
        System.out.println("4. User init 方法执行，进行 Bean 的初始化");
    }
    
    public void destroy() {
        System.out.println("7. User destroy 方法执行，进行 Bean 的销毁");
    }
    
    @Override
    public void setBeanClassLoader(ClassLoader classLoader) {
        System.out.println("User Bean 的类加载器：" + classLoader);
    }
    
    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        System.out.println("生产 User Bean 的工厂：" + beanFactory);
    }
    
    @Override
    public void setBeanName(String name) {
        System.out.println("User Bean 的名字为：" + name);
    }
}

实现 InitializingBean 接口
在Bean初始化之前，检查Bean是否实现了InitializingBean接口，如果实现了，则调用接口中的方法。
InitializingBean的方法早于init-method的执行。
public class User implements BeanNameAware, BeanClassLoaderAware, BeanFactoryAware, InitializingBean {
	...

	@Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("InitializingBean 的 afterPropertiesSet 方法执行...");
    }
}

实现 DisposableBean 接口
在销毁Bean之前，检查Bean是否实现了DisposableBean接口，如果实现了，则调用接口中的方法。
DisposableBean的方法早于destroy-method的执行
public class User implements BeanNameAware, BeanClassLoaderAware, BeanFactoryAware,
                    InitializingBean, DisposableBean {
	...

	@Override
    public void destroy() throws Exception {
        System.out.println("DisposableBean destroy 方法执行...");
    }

    // 之前销毁Bean的方法与DisposableBean接口中的方法重名，
    // 进行销毁方法名的修改，并修改Spring.xml文件中的配置
    public void myDestroy() {
        System.out.println("7. User destroy 方法执行，进行 Bean 的销毁");
    }

	...

}

```

# 6. Bean的作用域不同，管理方式不同
```
Spring 会根据Bean的作用域来选择管理方式
对于singleton作用域的Bean，Spring 能够精确地知道该Bean何时被创建，何时初始化完成，以及何时被销毁；而对于 prototype 作用域的 Bean，Spring 只负责创建，当容器创建了 Bean 的实例后，对该Bean初始化完毕，等到客户端程序获取到该Bean之后，Bean 的实例就交给客户端代码管理，Spring 容器将不再跟踪其生命周期。
Spring容器只对singleton的Bean进行完整的生命周期管理，对于prototype的Bean，Spring只进行部分的生命周期管理
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

	<!-- 配置 Bean 后处理器 -->
	<bean class="cw.spring.study.pojo.LogBeanPostProcessor"/>
	
	<bean 
			id="user" 
			class="cw.spring.study.pojo.User" 
			init-method="init" 
			destroy-method="myDestroy"
			scope="prototype"
	>
		<property name="name" value="张三"/>
	</bean>
	
</beans>

```

# 7. 将自己new的对象交给Spring管理
```
有些时候可能会遇到这样的需求，某个java对象是我们自己new的，然后我们希望这个对象被Spring容器管理
public class Student {}

@org.junit.Test
public void testRegisterBean() {
    Student student = new Student();
    System.out.println(student);
    
    // 将自己new的对象交给Spring进行管理
    // 使用DefaultListableBeanFactory对象进行Bean实例的注册
    DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
    // 将自己new的对象注册到Spring容器中
    factory.registerSingleton("student", student);
    
    // 从Spring容器中获取Bean
    System.out.println(factory.getBean("student"));
}
```

# 8. 源码解析：Bean 创建完整调用链路

## 8.1. 容器刷新入口：AbstractApplicationContext.refresh()

```
Spring 容器创建 Bean 并非在 new ClassPathXmlApplicationContext() 时立即执行，而是在容器刷新 (refresh) 过程中完成的。
refresh() 方法包含了 Spring 容器初始化的全部 13 个步骤，Bean 的创建发生在第 11 步 finishBeanFactoryInitialization()。

class AbstractApplicationContext — refresh() 方法源码（Spring 6.0.x）:

public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");

        // 1. 准备刷新：设置启动时间、激活状态等
        prepareRefresh();

        // 2. 获取 BeanFactory（对于 XmlApplicationContext 就是 DefaultListableBeanFactory）
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // 3. 准备 BeanFactory：设置类加载器、添加后处理器、注册环境 Bean 等
        prepareBeanFactory(beanFactory);

        try {
            // 4. 允许子类对 BeanFactory 进行后处理（空实现，留给子类扩展）
            postProcessBeanFactory(beanFactory);

            StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
            // 5. 调用所有 BeanFactoryPostProcessor
            //    包括 ConfigurationClassPostProcessor（处理 @Configuration、@ComponentScan）
            //    和 PropertySourcesPlaceholderConfigurer（处理 ${...} 占位符）
            invokeBeanFactoryPostProcessors(beanFactory);

            // 6. 注册所有 BeanPostProcessor（将其作为 Bean 提前实例化）
            //    注意：这里只是注册 BPP 这个 Bean，还没有执行它们的回调方法
            registerBeanPostProcessors(beanFactory);
            beanPostProcess.end();

            // 7. 初始化 MessageSource（国际化）
            initMessageSource();

            // 8. 初始化事件广播器 ApplicationEventMulticaster
            initApplicationEventMulticaster();

            // 9. 留空给子类扩展（如 Spring Boot 中启动内嵌 Web 服务器）
            onRefresh();

            // 10. 注册所有 ApplicationListener
            registerListeners();

            // ★★★ 11. 实例化所有非懒加载的单例 Bean ★★★
            //     这是 Bean 生命周期真正启动的入口
            finishBeanFactoryInitialization(beanFactory);

            // 12. 完成刷新，发布 ContextRefreshedEvent 事件
            finishRefresh();
        } catch (BeansException ex) {
            // ... 销毁已创建的 Bean
            throw ex;
        } finally {
            // 13. 重置公共缓存
            resetCommonCaches();
            contextRefresh.end();
        }
    }
}
```

### 8.2. finishBeanFactoryInitialization() → preInstantiateSingletons()

```
finishBeanFactoryInitialization() 的核心逻辑是调用 beanFactory.preInstantiateSingletons()：

@Override
public void preInstantiateSingletons() throws BeansException {
    // 获取所有 BeanDefinition 的名称
    List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

    for (String beanName : beanNames) {
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
        // 只实例化非抽象、非懒加载的 singleton Bean
        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            if (isFactoryBean(beanName)) {
                // FactoryBean 需要用 & 前缀获取工厂本身
                Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
                if (bean instanceof SmartFactoryBean<?> smartFactoryBean && smartFactoryBean.isEagerInit()) {
                    getBean(beanName); // 提前初始化 FactoryBean 产出的 Bean
                }
            } else {
                // ★ 核心调用：获取 Bean，触发完整的创建流程
                getBean(beanName);
            }
        }
    }

    // 所有单例 Bean 初始化完成后，触发 SmartInitializingSingleton 回调
    for (String beanName : beanNames) {
        Object singletonInstance = getSingleton(beanName);
        if (singletonInstance instanceof SmartInitializingSingleton sis) {
            sis.afterSingletonsInstantiated();
        }
    }
}
```

## 8.2. Bean 获取核心：AbstractBeanFactory.doGetBean()

```
getBean(beanName) 经过层层调用最终进入 doGetBean()，这是 Bean 获取/创建的总调度方法。
位于：org.springframework.beans.factory.support.AbstractBeanFactory

protected <T> T doGetBean(
        String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
        throws BeansException {

    // 1. 转换 beanName（去掉 & 前缀、处理别名）
    String beanName = transformedBeanName(name);

    // 2. ★★★ 尝试从三级缓存中获取 Bean ★★★
    //    调用 DefaultSingletonBeanRegistry.getSingleton(beanName, true)
    //    这里会依次查：一级缓存 → 二级缓存 → 三级缓存
    Object sharedInstance = getSingleton(beanName);
    Object beanInstance;

    if (sharedInstance != null && args == null) {
        // 缓存命中，说明这个 Bean 已经创建过了（可能是完整 Bean 也可能是早期引用）
        beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    } else {
        // 缓存未命中，需要创建

        // 3. 检查原型模式下的循环依赖
        if (isPrototypeCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }

        // 4. 检查父 BeanFactory
        BeanFactory parentBeanFactory = getParentBeanFactory();
        if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
            // 当前容器没有此 Bean 的定义，委托给父容器
            String nameToLookup = originalBeanName(name);
            if (parentBeanFactory instanceof AbstractBeanFactory abf) {
                return abf.doGetBean(nameToLookup, requiredType, args, typeCheckOnly);
            } else if (args != null) {
                return (T) parentBeanFactory.getBean(nameToLookup, args);
            } else if (requiredType != null) {
                return parentBeanFactory.getBean(nameToLookup, requiredType);
            } else {
                return (T) parentBeanFactory.getBean(nameToLookup);
            }
        }

        // 5. 标记该 Bean 正在创建中（用于检测循环依赖）
        if (!typeCheckOnly) {
            markBeanAsCreated(beanName);
        }

        StartupStep beanCreation = this.applicationStartup.start("spring.beans.instantiate")
                .tag("bean", name);
        try {
            if (requiredType != null) {
                beanCreation.tag("beanType", requiredType::toString);
            }
            // 6. 合并 BeanDefinition（合并父 Bean 属性、处理继承关系）
            RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
            checkMergedBeanDefinition(mbd, beanName, args);

            // 7. 确保 depends-on 依赖的 Bean 先初始化
            String[] dependsOn = mbd.getDependsOn();
            if (dependsOn != null) {
                for (String dep : dependsOn) {
                    registerDependentBean(dep, beanName);
                    getBean(dep);  // 递归创建依赖的 Bean
                }
            }

            // 8. 根据 scope 创建 Bean
            if (mbd.isSingleton()) {
                // ★ singleton：调用 DefaultSingletonBeanRegistry.getSingleton()
                //   内部先查询缓存，未命中则调用 createBean() 并放入缓存
                sharedInstance = getSingleton(beanName, () -> {
                    try {
                        return createBean(beanName, mbd, args);
                    } catch (BeansException ex) {
                        destroySingleton(beanName);
                        throw ex;
                    }
                });
                beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
            } else if (mbd.isPrototype()) {
                // prototype：每次调用都创建新实例，不缓存
                Object prototypeInstance = null;
                try {
                    beforePrototypeCreation(beanName);
                    prototypeInstance = createBean(beanName, mbd, args);
                } finally {
                    afterPrototypeCreation(beanName);
                }
                beanInstance = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
            } else {
                // 其他 scope（request, session 等）
                String scopeName = mbd.getScope();
                Scope scope = this.scopes.get(scopeName);
                if (scope == null) {
                    throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                }
                try {
                    Object scopedInstance = scope.get(beanName, () -> {
                        beforePrototypeCreation(beanName);
                        try {
                            return createBean(beanName, mbd, args);
                        } finally {
                            afterPrototypeCreation(beanName);
                        }
                    });
                    beanInstance = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                } catch (IllegalStateException ex) {
                    throw new ScopeNotActiveException(beanName, scopeName, ex);
                }
            }
        } catch (BeansException ex) {
            beanCreation.tag("exception", ex.getClass().toString());
            beanCreation.tag("message", String.valueOf(ex.getMessage()));
            cleanupAfterBeanCreationFailure(beanName);
            throw ex;
        } finally {
            beanCreation.end();
        }
    }

    // 9. 类型检查和转换
    return adaptBeanInstance(name, beanInstance, requiredType);
}
```

## 8.3. Bean 创建三大步：AbstractAutowireCapableBeanFactory.doCreateBean()

```
这是 Bean 创建最核心的方法，位于：
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory

protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
        throws BeanCreationException {

    // ============ 第一步：实例化 ============
    // 通过构造方法或工厂方法创建 Bean 的原始实例（wrapper 中包装的就是原始对象）
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        // 先尝试从 factoryBeanInstanceCache 中取（提前缓存的半成品）
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
        // ★ 调用 createBeanInstance() 创建实例
        //    内部会选择合适的实例化策略：构造方法注入、工厂方法、简单反射等
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    Object bean = instanceWrapper.getWrappedInstance();
    Class<?> beanType = instanceWrapper.getWrappedClass();
    if (beanType != NullBean.class) {
        mbd.resolvedTargetType = beanType;
    }

    // 允许 MergedBeanDefinitionPostProcessor 修改合并后的 BeanDefinition
    // （如 AutowiredAnnotationBeanPostProcessor 在此处解析 @Autowired 元数据）
    synchronized (mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            try {
                applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            } catch (Throwable ex) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Post-processing of merged bean definition failed", ex);
            }
            mbd.markAsPostProcessed();
        }
    }

    // ★★★ 早期曝光：如果是 singleton 且允许循环引用，将 ObjectFactory 放入三级缓存 ★★★
    boolean earlySingletonExposure = (mbd.isSingleton()
            && this.allowCircularReferences
            && isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        // 将一个 ObjectFactory（lambda 表达式）放入三级缓存 singletonFactories
        // 这个 lambda 内部调用 getEarlyBeanReference() 遍历 SmartInstantiationAwareBeanPostProcessor
        // 如果 Bean 被 AOP 代理了，这里会返回代理对象！
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }

    // 保存原始对象（后续 populateBean 会操作这个引用）
    Object exposedObject = bean;

    try {
        // ============ 第二步：属性赋值 ============
        // ★ 调用 populateBean() 注入属性
        populateBean(beanName, mbd, instanceWrapper);

        // ============ 第三步：初始化 ============
        // ★ 调用 initializeBean() 执行 Aware、BeanPostProcessor、InitMethod
        exposedObject = initializeBean(beanName, exposedObject, mbd);
    } catch (Throwable ex) {
        if (ex instanceof BeanCreationException bce && beanName.equals(bce.getBeanName())) {
            throw bce;
        }
        throw new BeanCreationException(mbd.getResourceDescription(), beanName, ex.getMessage(), ex);
    }

    // ★★ 循环依赖校验：如果提前暴露了早期引用，但初始化后 bean 变了（如被代理替换）
    //    需要确保最终给到其他 bean 的是同一个引用
    if (earlySingletonExposure) {
        Object earlySingletonReference = getSingleton(beanName, false);
        if (earlySingletonReference != null) {
            if (exposedObject == bean) {
                // 初始化后的对象和原始对象相同，说明没有被替换
                // 将早期引用赋给暴露对象（因为早期引用可能已经是代理了）
                exposedObject = earlySingletonReference;
            } else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
                String[] dependentBeans = getDependentBeans(beanName);
                Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
                for (String dependentBean : dependentBeans) {
                    if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                        actualDependentBeans.add(dependentBean);
                    }
                }
                // 抛异常：暴露的早期引用和最终 Bean 不一致，存在注入风险
                if (!actualDependentBeans.isEmpty()) {
                    throw new BeanCurrentlyInCreationException(beanName,
                            "Bean with name '" + beanName + "' has been injected into other beans [" +
                            StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                            "] in its raw version as part of a circular reference, but has eventually been " +
                            "wrapped. This means that said other beans do not use the final version of the " +
                            "bean. This is often the result of over-eager type matching - consider using " +
                            "'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");
                }
            }
        }
    }

    // 注册 Bean 的销毁方法
    try {
        registerDisposableBeanIfNecessary(beanName, bean, mbd);
    } catch (BeanDefinitionValidationException ex) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                "Invalid destruction signature", ex);
    }

    return exposedObject;
}
```

## 8.4. 初始化方法 initializeBean() 完整源码

```
initializeBean() 是生命周期中初始化阶段的总入口，位于 AbstractAutowireCapableBeanFactory：

protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {

    // ===== 点位A：调用 Aware 方法 =====
    // BeanNameAware.setBeanName()
    // BeanClassLoaderAware.setBeanClassLoader()
    // BeanFactoryAware.setBeanFactory()
    invokeAwareMethods(beanName, bean);

    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {

        // ===== 点位B：执行所有 BeanPostProcessor 的 postProcessBeforeInitialization() =====
        // 这包括 CommonAnnotationBeanPostProcessor（处理 @PostConstruct）
        // ApplicationContextAwareProcessor（处理 EnvironmentAware、ApplicationContextAware 等）
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }

    try {
        // ===== 点位C：调用 init 方法 =====
        // 先执行 InitializingBean.afterPropertiesSet()
        // 再执行 init-method 指定的自定义初始化方法
        invokeInitMethods(beanName, wrappedBean, mbd);
    } catch (Throwable ex) {
        throw new BeanCreationException(
                (mbd != null ? mbd.getResourceDescription() : null), beanName, ex.getMessage(), ex);
    }

    if (mbd == null || !mbd.isSynthetic()) {

        // ===== 点位D：执行所有 BeanPostProcessor 的 postProcessAfterInitialization() =====
        // ★ 这是 AOP 代理创建的真正入口！
        // AnnotationAwareAspectJAutoProxyCreator（实现了 BeanPostProcessor）在这里
        // 判断 Bean 是否需要创建代理，如果需要则返回代理对象
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }

    return wrappedBean;
}
```

### 8.5. invokeAwareMethods() 源码

```
private void invokeAwareMethods(String beanName, Object bean) {
    if (bean instanceof Aware) {
        if (bean instanceof BeanNameAware beanNameAware) {
            beanNameAware.setBeanName(beanName);
        }
        if (bean instanceof BeanClassLoaderAware beanClassLoaderAware) {
            ClassLoader bcl = getBeanClassLoader();
            if (bcl != null) {
                beanClassLoaderAware.setBeanClassLoader(bcl);
            }
        }
        if (bean instanceof BeanFactoryAware beanFactoryAware) {
            beanFactoryAware.setBeanFactory(AbstractAutowireCapableBeanFactory.this);
        }
    }
}
这里只处理了三个 Aware：
• BeanNameAware
• BeanClassLoaderAware
• BeanFactoryAware

那 ApplicationContextAware 等其他 Aware 在哪里处理？
答案是：ApplicationContextAwareProcessor（一个 BeanPostProcessor）
它在 applyBeanPostProcessorsBeforeInitialization() 阶段被调用，通过 instanceof 检查并注入。
```

### 8.6. invokeInitMethods() 源码

```
protected void invokeInitMethods(String beanName, Object bean, @Nullable RootBeanDefinition mbd)
        throws Throwable {

    boolean isInitializingBean = (bean instanceof InitializingBean);
    if (isInitializingBean && (mbd == null || !mbd.hasExplicitlyDestroyedInitMethod("afterPropertiesSet"))) {
        // 1. 先调用 InitializingBean.afterPropertiesSet()
        ((InitializingBean) bean).afterPropertiesSet();
    }

    if (mbd != null && bean.getClass() != NullBean.class) {
        String initMethodName = mbd.getInitMethodName();
        if (StringUtils.hasLength(initMethodName) && !(isInitializingBean
                && "afterPropertiesSet".equals(initMethodName))
                && !mbd.hasExplicitlyDestroyedInitMethod(initMethodName)) {
            // 2. 再通过反射调用 init-method 指定的方法
            invokeCustomInitMethod(beanName, bean, mbd);
        }
    }
}
```

## 8.5. 完整的 Bean 生命周期流程图（15 步）

```
┌──────────────────────────────────────────────────────────────┐
│                    Bean 生命周期 - 15 步                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Step 1:  实例化 — createBeanInstance()                       │
│           反射调用构造方法 / 工厂方法创建对象                   │
│           ↓                                                  │
│  Step 2:  属性赋值前 — applyMergedBeanDefinitionPostProcessors │
│           如 AutowiredAnnotationBeanPostProcessor             │
│           解析 @Autowired、@Value 的注入元数据                 │
│           ↓                                                  │
│  Step 3:  早期曝光 — addSingletonFactory()                    │
│           将 ObjectFactory 放入三级缓存（解决循环依赖）         │
│           ↓                                                  │
│  Step 4:  属性赋值 — populateBean()                           │
│           注入属性值（包括 @Autowired、setter 注入等）          │
│           ↓                                                  │
│  Step 5:  BeanNameAware.setBeanName()                         │
│           ↓                                                  │
│  Step 6:  BeanClassLoaderAware.setBeanClassLoader()           │
│           ↓                                                  │
│  Step 7:  BeanFactoryAware.setBeanFactory()                   │
│           ↓                                                  │
│  Step 8:  其他 Aware — ApplicationContextAwareProcessor        │
│           EnvironmentAware → EmbeddedValueResolverAware →     │
│           ResourceLoaderAware → ApplicationEventPublisherAware│
│           → MessageSourceAware → ApplicationContextAware       │
│           ↓                                                  │
│  Step 9:  BeanPostProcessor.postProcessBeforeInitialization() │
│           ★ CommonAnnotationBeanPostProcessor 在此处理        │
│           @PostConstruct 注解                                 │
│           ↓                                                  │
│  Step 10: InitializingBean.afterPropertiesSet()               │
│           ↓                                                  │
│  Step 11: 自定义 init-method（反射调用）                       │
│           ↓                                                  │
│  Step 12: BeanPostProcessor.postProcessAfterInitialization()  │
│           ★ AOP 代理创建入口（AbstractAutoProxyCreator）       │
│           ★ 如果需要代理，返回代理对象替代原始对象              │
│           ↓                                                  │
│  Step 13: SmartInitializingSingleton.afterSingletonsInstantiated│
│           所有单例 Bean 初始化完成后回调（Spring 6.1+）        │
│           ↓                                                  │
│  Step 14: Bean 就绪，可以被使用了                              │
│           ↓                                                  │
│  Step 15: 容器关闭时销毁                                      │
│           DisposableBean.destroy() → 自定义 destroy-method     │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

## 8.6. Spring 6 中内置的核心 BeanPostProcessor

```
Spring 6 内置了大量 BeanPostProcessor，各有不同的触发时机和职责。
按照执行阶段分类如下：

┌─────────────────────────────────────────────────────────────┐
│      postProcessBeforeInitialization 阶段                    │
├─────────────────────────────────────────────────────────────┤
│ ApplicationContextAwareProcessor                             │
│   处理EnvironmentAware/ResourceLoaderAware/ApplicationContext│
│   Aware 等7个 Aware 接口的注入                               │
│                                                             │
│ CommonAnnotationBeanPostProcessor                            │
│   ★ 处理 @PostConstruct 注解（JSR-250）                     │
│   ★ 处理 @Resource 注解（按名称注入）                         │
│   ★ 处理 @PreDestroy 注解                                    │
│                                                             │
│ InitDestroyAnnotationBeanPostProcessor                       │
│   内部类，处理 @PostConstruct 和 @PreDestroy                  │
│   如果单独引入 spring-context 可能使用这个                    │
│                                                             │
│ ConfigurationClassPostProcessor（实际是 BFPP 而非 BPP）      │
│   处理 @Configuration、@ComponentScan、@Bean 等              │
│   严格来说它是 BeanFactoryPostProcessor                     │
│   但它也影响 @Bean 方法的生命周期                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│      postProcessAfterInitialization 阶段                     │
├─────────────────────────────────────────────────────────────┤
│ AutowiredAnnotationBeanPostProcessor                         │
│   处理 @Autowired 和 @Value 注解的注入                        │
│   ★ 这是自动装配的核心后处理器                                │
│                                                             │
│ AnnotationAwareAspectJAutoProxyCreator                       │
│   ★★★ AOP 代理创建的核心 ★★★                               │
│   继承自 AbstractAutoProxyCreator                            │
│   实现了 SmartInstantiationAwareBeanPostProcessor            │
│   在 afterInitialization 阶段判断是否需要为 Bean 创建代理     │
│                                                             │
│ AsyncAnnotationBeanPostProcessor                             │
│   处理 @Async 注解，为方法创建异步代理                        │
│                                                             │
│ ScheduledAnnotationBeanPostProcessor                         │
│   处理 @Scheduled 注解                                       │
│                                                             │
│ PersistenceExceptionTranslationPostProcessor                 │
│   为 @Repository Bean 添加持久层异常转换                      │
│   将 JPA/Hibernate 的特定异常转换为 Spring DataAccessException│
│                                                             │
│ MethodValidationPostProcessor                                │
│   处理 @Validated 注解，开启方法级别的参数校验                 │
│   实际上是 BeanPostProcessor，但校验发生在方法调用时          │
                                                              │
│ EventListenerMethodProcessor                                 │
│   处理 @EventListener 注解，注册事件监听器                    │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│      MergedBeanDefinitionPostProcessor（MBD 后处理）         │
├─────────────────────────────────────────────────────────────┤
│ 在属性赋值之前执行（不是 BeanPostProcessor 的子接口，        │
│ 但也在 doCreateBean 中被调用）                                │
│                                                             │
│ AutowiredAnnotationBeanPostProcessor                         │
│   ★ 在 postProcessMergedBeanDefinition() 中                  │
│   解析 @Autowired、@Value 注解的注入点元数据                  │
│   并将元数据缓存到 InjectionMetadata 中                      │
│                                                             │
│ CommonAnnotationBeanPostProcessor                            │
│   解析 @Resource 注解的注入点元数据                           │
│                                                             │
│ ScheduledAnnotationBeanPostProcessor                         │
│   解析 @Scheduled 注解                                       │
└─────────────────────────────────────────────────────────────┘
```

## 8.7. Bean 销毁流程源码

```
容器关闭时调用 AbstractApplicationContext.close() → doClose() → destroyBeans()：

// DefaultListableBeanFactory.destroySingletons()
public void destroySingletons() {
    super.destroySingletons();
    // 获取所有一次性（disposable）Bean
    String[] disposableBeanNames = StringUtils.toStringArray(this.disposableBeans.keySet());
    for (int i = disposableBeanNames.length - 1; i >= 0; i--) {
        destroySingleton(disposableBeanNames[i]);
    }
    // ... 清理
}

// DefaultSingletonBeanRegistry.destroySingleton() → DisposableBeanAdapter.destroy()
public void destroy() {
    // Step 1: 执行 @PreDestroy 注解的方法
    //         （由 CommonAnnotationBeanPostProcessor 注册）
    
    // Step 2: 执行 DisposableBean.destroy() 接口方法
    if (this.invokeDisposableBean) {
        ((DisposableBean) this.bean).destroy();
    }
    
    // Step 3: 执行 destroy-method 指定的方法
    if (this.destroyMethod != null) {
        invokeCustomDestroyMethod(this.destroyMethod);
    }
}
销毁顺序： @PreDestroy → DisposableBean.destroy() → destroy-method
```

## 8.8. 设计思想总结

1. 模板方法模式：
   doCreateBean() 是模板方法，其中的 createBeanInstance()、populateBean()、
   initializeBean() 都是可以被子类重写的步骤，体现了"骨架固定，细节可变"的设计思想。

2. 策略模式：
   实例化策略（InstantiationStrategy）可以在 SimpleInstantiationStrategy（反射）
   和 CglibSubclassingInstantiationStrategy（CGLIB 子类化）之间切换。

3. 责任链模式：
   BeanPostProcessor 形成的是一条处理链，每个 BPP 都可以在前后两个时间点
   对 Bean 进行操作。AOP 代理创建就是在这个链上完成的。

4. 扩展点设计：
   Spring 的生命周期是"开放-闭合"原则的典型体现——
   对扩展开放（各种 Aware、BPP、InitializingBean、init-method），
   对修改闭合（核心流程 doCreateBean 不会因扩展而改变）。
