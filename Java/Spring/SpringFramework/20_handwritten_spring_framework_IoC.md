# 1. 手写 Spring IoC 框架
Spring IoC 的实现原理：工厂模式 + 解析XML + 反射机制
# 1. 环境准备
```
打包方式与依赖引入
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	
	<groupId>cw.study.spring</groupId>
	<artifactId>spring-study-011-myspring</artifactId>
	<version>1.0-SNAPSHOT</version>
	<packaging>jar</packaging>
	
	<dependencies>
		<!-- dom4j 解析XML文件 -->
		<dependency>
			<groupId>org.dom4j</groupId>
			<artifactId>dom4j</artifactId>
			<version>2.1.3</version>
		</dependency>
    	<!-- jaxen 需要使用xpath -->
		<dependency>
			<groupId>jaxen</groupId>
			<artifactId>jaxen</artifactId>
			<version>1.2.0</version>
		</dependency>
		<!-- 单元测试 -->
		<dependency>
			<groupId>junit</groupId>
			<artifactId>junit</artifactId>
			<version>4.13.2</version>
			<scope>test</scope>
		</dependency>
	</dependencies>
	
	<properties>
		<maven.compiler.source>17</maven.compiler.source>
		<maven.compiler.target>17</maven.compiler.target>
	</properties>

</project>

```

# 2. Spring框架核心接口实现ApplicationContext
```
package org.myspringframework.core;

/**
 * ClassName: ApplicationContext
 * Package: org.myspringframework.core
 * Description:
 * MySpring框架应用上下文接口
 *
 * @Author tcw
 * @Create 2023-05-31 14:22
 * @Version 1.0
 */
public interface ApplicationContext {
    
    /**
     * 通过Bean的id从容器中获取相应的Bean对象
     *      Bean的名称就是配置文件中Bean的id
     *
     * @param beanName Bean的id
     * @return Bean对象
     */
    Object getBean(String beanName);
    
}
```

# 3. Spring框架核心接口实现类ClassPathXmlApplicationContext
```
package org.myspringframework.core;

import org.dom4j.*;
import org.dom4j.io.SAXReader;

import java.io.InputStream;
import java.lang.reflect.Method;
import java.util.HashMap;
import java.util.List;
import java.util.Map;


/**
 * ClassName: ClassPathXmlApplicationContext
 * Package: org.myspringframework.core
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-31 14:25
 * @Version 1.0
 */
public class ClassPathXmlApplicationContext implements ApplicationContext{
    
    // 用于存放Bean对象的map集合，使用Bean的名称可以获取相应的Bean对象
    private Map<String, Object> singletonObjects = new HashMap<>();
    
    /**
     * 根据Spring配置文件创建Spring容器的构造方法
     * 该方法一执行就会对配置文件进行解析创建Bean对象
     * 使用ClassPathXmlApplicationContext，配置文件应该放在类路径下
     *
     * @param configLocation Spring配置文件的路径
     */
    public ClassPathXmlApplicationContext(String configLocation) {
        try {
            // 解析XML核心配置文件，实例化Bean，将Bean存放到map集合中
            // 获取dom4j解析xml文件的核心对象，读取xml文件中的内容
            SAXReader reader = new SAXReader();
            // 获取xml的输入流，通过类加载器以流的方式获取资源
            InputStream inputStream = ClassLoader.getSystemClassLoader().getResourceAsStream(configLocation);
            // 读取xml文件中的内容，获取xml对应的文档对象
            Document document = reader.read(inputStream);
            // 获取xml文件中所有的bean标签
            List<Node> beansNode = document.selectNodes("//bean");
            // 遍历所有的bean标签，创建相应的对象放到容器中
            beansNode.forEach(bean->{
                try {
                    // System.out.println(bean);
                    // org.dom4j.tree.DefaultElement@74ad1f1f
                    // bean实际上是Element类型，向下转型，因为Element中方法更丰富
                    Element beanElement = (Element) bean;
                    // 获取Bean id
                    String id = beanElement.attributeValue("id");
                    // 获取bean的类型
                    String className = beanElement.attributeValue("class");
                    // System.out.println(id + ": " + className);
                    // 利用反射机制进行对象的创建，并放到map集合中，进行提前曝光
                    Object beanInstance = createBeanObjectByReflect(className);
                    // 将创建的对象放入map集合中进行曝光
                    singletonObjects.put(id, beanInstance);
                    // System.out.println(singletonObjects);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            });
            // 再次遍历所有的bean标签，为创建的所有bean对象进行属性的赋值
            beansNode.forEach(bean->{
                try {
                    // 进行强转
                    Element beanElement= (Element) bean;
                    // 获取bean的id
                    String id = beanElement.attributeValue("id");
                    // // 判断bean是否已经被创建了
                    // if (!singletonObjects.containsKey(id)) {
                    //     // 如果没有创建bean对象，则进行bean对象的创建
                    //     Object beanInstance = createBeanObjectByReflect(className);
                    // }
                    // 获取bean的类名
                    String className = beanElement.attributeValue("class");
                    // 加载类
                    Class<?> clazz = Class.forName(className);
                    // 获取bean标签中的所有子标签属性标签
                    List<Element> properties = beanElement.elements("property");
                    // 遍历bean标签中的所有子标签属性标签，为bean的属性进行赋值
                    properties.forEach(property->{
                        try {
                            // 获取property标签的name属性值，获取要为哪个属性赋值
                            String fieldName = property.attributeValue("name");
                            // 获取property标签的value属性值，获取要为属性赋的值(标签中如果无value，返回null)
                            String fieldValue = property.attributeValue("value");
                            // 获取property标签的ref属性值，获取要为属性赋的值(标签中如果无ref，返回null)
                            String ref = property.attributeValue("ref");
                            // System.out.println(fieldName + " = " + fieldValue);
                            // 获取set方法的方法名
                            String setMethodName = "set" + fieldName.toUpperCase().charAt(0) + fieldName.substring(1);
                            // 获取set方法对应的属性的类型
                            Class<?> fieldType = clazz.getDeclaredField(fieldName).getType();
                            // 获取set方法
                            Method setMethod = clazz.getDeclaredMethod(setMethodName, fieldType);
                            // 如果属性的值是通过value进行赋值
                            if (fieldValue != null) {
                                // 值为简单类型
                                // 调用id对应的bean对象的set方法(set方法无返回值)
                                // 从标签的属性中获取的值都为字符串类型，但是set方法需要的参数有具体的数据类型，需要进行处理
                                // 获取set方法参数的简单类型名
                                String fieldTypeSimpleName = fieldType.getSimpleName();
                                String propertyValue = fieldValue; // 简单类型的属性值
                                // 进行数据类型匹配，对参数进行类型转换
                                Object propertyVal = null;
                                switch (fieldTypeSimpleName) {
                                    case "byte": case "Byte":
                                        propertyVal = Byte.valueOf(propertyValue);
                                        break;
                                    case "short": case "Short":
                                        propertyVal = Short.valueOf(propertyValue);
                                        break;
                                    case "int": case "Integer":
                                        propertyVal = Integer.valueOf(propertyValue);
                                        break;
                                    case "long": case "Long":
                                        propertyVal = Long.valueOf(propertyValue);
                                        break;
                                    case "float": case "Float":
                                        propertyVal = Float.valueOf(propertyValue);
                                        break;
                                    case "double": case "Double":
                                        propertyVal = Double.valueOf(propertyValue);
                                        break;
                                    case "boolean": case "Boolean":
                                        propertyVal = Boolean.valueOf(propertyValue);
                                        break;
                                    case "char": case "Character":
                                        propertyVal = propertyValue.charAt(0);
                                        break;
                                    case "String":
                                        propertyVal = propertyValue;
                                        break;
                                }
                                // 调用id对应的bean对象的set方法(set方法无返回值)
                                setMethod.invoke(singletonObjects.get(id), propertyVal);
                            }
                            // 如果属性的值是通过ref进行赋值
                            if (ref != null) {
                                // 值为非简单类型
                                // 调用id对应的bean对象的set方法(set方法无返回值)
                                setMethod.invoke(singletonObjects.get(id), singletonObjects.get(ref));
                            }
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    });
                } catch (Exception e) {
                    e.printStackTrace();
                }
            });
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    
    /**
     * 通过反射机制进行相应类的对象的创建
     *
     * @param className 类的全限定类名
     * @return 类的对象
     * @throws Exception 异常
     */
    private Object createBeanObjectByReflect(String className) throws Exception {
        // 加载类
        Class<?> clazz = Class.forName(className);
        // 获取类的无参构造器创建对象
        return clazz.getDeclaredConstructor().newInstance();
    }
    
    /**
     * 根据bean的id获取bean对象
     *
     * @param beanName Bean的id
     * @return bean对象
     */
    @Override
    public Object getBean(String beanName) {
        return singletonObjects.get(beanName);
    }
}

```

# 4. MySpring框架测试
```
@org.junit.Test
public void test01() {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("myspring.xml");
    UserService userService = (UserService) applicationContext.getBean("userService");
    userService.save();
    User user = (User) applicationContext.getBean("user");
    System.out.println(user);
}

```

# 5. 源码对比：手写 MySpring vs 真实 Spring IoC

## 5.1 架构对比

```
                          手写 MySpring                    真实 Spring 6
┌──────────────┐  ┌────────────────────────┐  ┌────────────────────────────┐
│ 配置加载      │  │ 自定义 dom4j 解析 XML   │  │ XmlBeanDefinitionReader      │
│              │  │ + XPath 查找元素        │  │ + DefaultDocumentLoader (JAXP)│
├──────────────┤  ├────────────────────────┤  ├────────────────────────────┤
│ 中间模型      │  │ ❌ 无（直接创建对象）    │  │ ★ BeanDefinition（元数据层）   │
│              │  │                       │  │ + BeanDefinitionRegistry      │
├──────────────┤  ├────────────────────────┤  ├────────────────────────────┤
│ Bean 存储     │  │ Map<String,Object>     │  │ 三级缓存：                    │
│              │  │                       │  │ singletonObjects (L1)         │
│              │  │                       │  │ earlySingletonObjects (L2)    │
│              │  │                       │  │ singletonFactories (L3)      │
├──────────────┤  ├────────────────────────┤  ├────────────────────────────┤
│ 依赖注入      │  │ 两阶段：               │  │ doCreateBean 三步：           │
│              │  │ 1) 反射创建所有对象     │  │ 1) createBeanInstance()      │
│              │  │ 2) 遍历注入属性        │  │ 2) populateBean()            │
│              │  │                       │  │ 3) initializeBean()          │
├──────────────┤  ├────────────────────────┤  ├────────────────────────────┤
│ 类型转换      │  │ ❌ 无                  │  │ ★ TypeConverter 体系           │
│              │  │                       │  │ PropertyEditor/Converter      │
├──────────────┤  ├────────────────────────┤  ├────────────────────────────┤
│ 后处理器      │  │ ❌ 无                  │  │ ★ BeanPostProcessor 链         │
│              │  │                       │  │ + BeanFactoryPostProcessor    │
├──────────────┤  ├────────────────────────┤  ├────────────────────────────┤
│ 作用域        │  │ ❌ 仅 singleton        │  │ singleton / prototype /       │
│              │  │                       │  │ request / session / 自定义    │
├──────────────┤  ├────────────────────────┤  ├────────────────────────────┤
│ 循环依赖      │  │ ❌ 不支持             │  │ ★ 三级缓存解决 singleton+setter │
├──────────────┤  ├────────────────────────┤  ├────────────────────────────┤
│ 生命周期管理  │  │ ❌ 无                  │  │ Aware → BPP → Init → BPP →   │
│              │  │                       │  │ Ready → Destroy               │
└──────────────┘  └────────────────────────┘  └────────────────────────────┘
```

## 5.2 核心代码对比

```
手写 MySpring 的"两阶段"创建：
// 第一阶段：反射创建所有对象
for (Element beanElement : beanElements) {
    String id = beanElement.attributeValue("id");
    String clazz = beanElement.attributeValue("class");
    Class<?> beanClass = Class.forName(clazz);
    Object bean = beanClass.getDeclaredConstructor().newInstance();
    singletonObjects.put(id, bean);  // 存入 Map
}
// 第二阶段：遍历注入属性
for (Element beanElement : beanElements) {
    String id = beanElement.attributeValue("id");
    Object bean = singletonObjects.get(id);
    for (Element propElement : beanElement.elements("property")) {
        String name = propElement.attributeValue("name");
        String ref = propElement.attributeValue("ref");
        String value = propElement.attributeValue("value");
        // 通过反射调用 setter
        if (ref != null) {
            Object refBean = singletonObjects.get(ref);
            Method setter = bean.getClass().getMethod("set" + capitalize(name), ...);
            setter.invoke(bean, refBean);
        }
    }
}

Spring 真实的 doCreateBean()：
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, Object[] args) {
    // 1. 创建实例
    BeanWrapper instanceWrapper = createBeanInstance(beanName, mbd, args);
    // 2. ★ 早期曝光（支持循环依赖）
    addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    // 3. 属性注入
    populateBean(beanName, mbd, instanceWrapper);
    // 4. ★ 初始化（Aware → BPP → init → BPP → AOP代理）
    Object exposedObject = initializeBean(beanName, exposedObject, mbd);
    // 5. ★ 注册销毁方法
    registerDisposableBeanIfNecessary(beanName, bean, mbd);
}
```

## 5.3 设计差异的本质

手写 MySpring 抓住的是 Spring IoC 最核心的三要素：
  工厂模式 + XML解析 + 反射机制

真实 Spring 在此基础上额外解决了：
  1. BeanDefinition 中间层：解耦配置来源和对象创建
     XML / 注解 / Java Config 三种配置方式共用同一套 Bean 创建引擎
     
  2. 扩展点体系：BeanPostProcessor / BeanFactoryPostProcessor
     使 Spring 从"硬编码框架"变为"可扩展平台"
     
  3. 生命周期管理：Aware → BPP → Init → BPP → AOP → Ready
     确保每一步都可以被用户自定义代码介入
     
  4. 循环依赖：三级缓存 + 早期曝光
     解决了单例 setter 注入的循环依赖这一经典难题
     
  5. 作用域：Scope 接口 + 注册机制
     让 Bean 的生命周期从"单例/非单例"扩展到任意维度
     
  ★ 手写 MySpring 是理解了 Spring 的骨架，
    而完整的 Spring 源码则是充满血肉和神经的有机体。
