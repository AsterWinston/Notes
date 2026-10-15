# 1. p 命名空间注入
```
目的：简化配置。
p命名空间实际上是对set注入的简化。
使用p命名空间注入的前提条件包括两个：
第一：在XML头部信息中添加p命名空间的配置信息：xmlns:p="http://www.springframework.org/schema/p"
第二：p命名空间注入是基于setter方法的，所以需要对应的属性提供setter方法。

public class Dog {
    // 简单类型
    private String name;
    private int age;
    // 非简单类型
    private Date birth;

    // p命名空间注入底层还是set注入，只不过p命名空间注入可以让spring配置变的更加简单。
    public void setName(String name) {
        this.name = name;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public void setBirth(Date birth) {
        this.birth = birth;
    }

    @Override
    public String toString() {
        return "Dog{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", birth=" + birth +
                '}';
    }
}

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:p="http://www.springframework.org/schema/p"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <!--
        第一步：在spring的配置文件头部添加p命名空间。
  					xmlns:p="http://www.springframework.org/schema/p"
        第二步：使用 p:属性名 = "属性值"
    -->
    <bean id="dogBean" class="com.powernode.spring6.bean.Dog" p:name="小花" p:age="3" p:birth-ref="birthBean"/>

    <!--这里获取的是当前系统时间。-->
    <bean id="birthBean" class="java.util.Date"/>

</beans>

```

# 2. 源码解析：Spring XML 命名空间机制

## 2.1 NamespaceHandler — 自定义命名空间的背后

```
<p:name="小花" />、<c:_0="张三" />、<util:properties>、<context:component-scan> 等
自定义命名空间的标签如何被 Spring 识别和处理？

答案：NamespaceHandler 机制。

Spring 在 META-INF/spring.handlers 文件中注册每个命名空间的处理器：

# spring-context 模块的 spring.handlers 文件内容：
http\://www.springframework.org/schema/context=org.springframework.context.config.ContextNamespaceHandler
http\://www.springframework.org/schema/jee=org.springframework.ejb.config.JeeNamespaceHandler
http\://www.springframework.org/schema/lang=org.springframework.scripting.config.LangNamespaceHandler
http\://www.springframework.org/schema/task=org.springframework.scheduling.config.TaskNamespaceHandler
http\://www.springframework.org/schema/cache=org.springframework.cache.config.CacheNamespaceHandler

# spring-aop 模块的 spring.handlers 文件：
http\://www.springframework.org/schema/aop=org.springframework.aop.config.AopNamespaceHandler

# spring-tx 模块的 spring.handlers 文件：
http\://www.springframework.org/schema/tx=org.springframework.transaction.config.TxNamespaceHandler

# p/c/util 命名空间没有对应的 spring.handlers！
# 因为它们是通过 BeanDefinitionParserDelegate 直接处理的

XML 解析时，DefaultBeanDefinitionDocumentReader 中：
if (delegate.isDefaultNamespace(ele)) {
    parseDefaultElement(ele, delegate);  // <bean>, <import>, <alias>, <beans>
} else {
    delegate.parseCustomElement(ele);    // ★ 自定义命名空间 → NamespaceHandler
}
```

## 2.2 ContextNamespaceHandler — 处理 <context:> 标签

```
public class ContextNamespaceHandler extends NamespaceHandlerSupport {

    @Override
    public void init() {
        // 为每个 <context:> 子标签注册对应的 BeanDefinitionParser
        registerBeanDefinitionParser("property-placeholder",
                new PropertyPlaceholderBeanDefinitionParser());
                // <context:property-placeholder/> → PropertySourcesPlaceholderConfigurer

        registerBeanDefinitionParser("property-override",
                new PropertyOverrideBeanDefinitionParser());

        registerBeanDefinitionParser("annotation-config",
                new AnnotationConfigBeanDefinitionParser());
                // <context:annotation-config/> → AutowiredAnnotationBeanPostProcessor 等

        registerBeanDefinitionParser("component-scan",
                new ComponentScanBeanDefinitionParser());
                // <context:component-scan/> → ClassPathBeanDefinitionScanner

        registerBeanDefinitionParser("load-time-weaver",
                new LoadTimeWeaverBeanDefinitionParser());

        registerBeanDefinitionParser("spring-configured",
                new SpringConfiguredBeanDefinitionParser());

        registerBeanDefinitionParser("mbean-export",
                new MBeanExportBeanDefinitionParser());

        registerBeanDefinitionParser("mbean-server",
                new MBeanServerBeanDefinitionParser());
    }
}

// NamespaceHandlerSupport.parse() 的流程：
public BeanDefinition parse(Element element, ParserContext parserContext) {
    // 1. 获取标签对应的 BeanDefinitionParser
    BeanDefinitionParser parser = findParserForElement(element, parserContext);
    // 2. 调用 parser.parse() 返回 BeanDefinition
    return (parser != null ? parser.parse(element, parserContext) : null);
}
```

## 2.3 p/c 命名空间的处理机制

```
p/c 命名空间没有对应的 NamespaceHandler，而是由 BeanDefinitionParserDelegate 直接处理。

p-namespace 格式：p:xxx="value" 或 p:xxx-ref="beanId"
c-namespace 格式：c:_0="value" 或 c:xxx="value" 或 c:xxx-ref="beanId"

在 BeanDefinitionParserDelegate.decorateBeanDefinitionIfRequired() 中：

public BeanDefinitionHolder decorateBeanDefinitionIfRequired(
        Element ele, BeanDefinitionHolder definitionHolder) {

    BeanDefinitionHolder finalDefinition = definitionHolder;

    // 遍历元素的所有属性
    NamedNodeMap attributes = ele.getAttributes();
    for (int i = 0; i < attributes.getLength(); i++) {
        Node node = attributes.item(i);
        String nodeName = node.getNodeName();
        String nodeValue = node.getNodeValue();

        // ★ 检查是否是 p/c 命名空间
        if (isPNamespace(nodeName)) {
            // p:name="小花" → 解析为属性 name = "小花"
            String propertyName = nodeName.substring(nodeName.indexOf(':') + 1);
            if (nodeName.endsWith("-ref")) {
                // p:birth-ref="birthBean" → 属性 birth = RuntimeBeanReference("birthBean")
                String refName = nodeValue;
                definitionHolder.getBeanDefinition().getPropertyValues()
                        .add(propertyName.substring(0, propertyName.length() - 4),
                                new RuntimeBeanReference(refName));
            } else {
                // p:name="小花" → 属性 name = TypedStringValue("小花")
                definitionHolder.getBeanDefinition().getPropertyValues()
                        .add(propertyName, new TypedStringValue(nodeValue));
            }
        }

        if (isCNamespace(nodeName)) {
            // c:_0="张三" → 构造器参数 index=0, value="张三"
            // c:name="张三" → 构造器参数 name="name", value="张三"
            String argName = nodeName.substring(nodeName.indexOf(':') + 1);
            if (nodeName.endsWith("-ref")) {
                // c:userDao-ref="userDaoBean"
                finalDefinition = parseCArgWithRef(argName, nodeValue, finalDefinition);
            } else {
                // c:name="张三"
                finalDefinition = parseCArgWithValue(argName, nodeValue, finalDefinition);
            }
        }
    }
    return finalDefinition;
}

★ 核心要点：
  p-namespace 本质上是 <property> 标签的语法糖
  c-namespace 本质上是 <constructor-arg> 标签的语法糖
  二者在 XML 解析时被转换为相同的内部表示（TypedStringValue / RuntimeBeanReference）
```

## 2.4 util 命名空间的处理机制

```
util-namespace 有对应的 NamespaceHandler：

public class UtilNamespaceHandler extends NamespaceHandlerSupport {

    @Override
    public void init() {
        registerBeanDefinitionParser("constant",
                new ConstantBeanDefinitionParser());
        registerBeanDefinitionParser("property-path",
                new PropertyPathBeanDefinitionParser());
        registerBeanDefinitionParser("list",
                new ListBeanDefinitionParser());
        registerBeanDefinitionParser("set",
                new SetBeanDefinitionParser());
        registerBeanDefinitionParser("map",
                new MapBeanDefinitionParser());
        registerBeanDefinitionParser("properties",
                new PropertiesBeanDefinitionParser());  // <util:properties>
    }
}

<util:properties> 被 PropertiesBeanDefinitionParser 解析时：

public class PropertiesBeanDefinitionParser extends AbstractSingleBeanDefinitionParser {

    @Override
    protected Class<?> getBeanClass(Element element) {
        return Properties.class;
    }

    @Override
    protected void doParse(Element element, BeanDefinitionBuilder builder) {
        // 调用父类解析 <prop> 子元素
        super.doParse(element, builder);
        // 设置属性 location（如果指定了 properties 文件路径）
        builder.addPropertyValue("location",
                new TypedStringValue(element.getAttribute("location")));
    }
}

★ 解析后的 BeanDefinition：
  <util:properties id="prop">...</util:properties>
  → beanClass = java.util.Properties
  → propertyValues = { driver → "com.mysql.cj.jdbc.Driver", ... }
  → 注册到 beanDefinitionMap["prop"]
```

## 2.5 设计思想总结

1. 策略模式 + 注册表模式（NamespaceHandler）：
   每个 XML 命名空间对应一个 NamespaceHandler，每个标签对应一个 BeanDefinitionParser。
   这种设计使得新增自定义命名空间只需实现接口并注册，完全符合 OCP 原则。

2. p/c 命名空间的"零配置"处理：
   p/c 命名空间不需要 NamespaceHandler，因为它们本质上是标准 Bean 定义属性的别名。
   这种直接处理的方式减少了架构层次，体现了"简单优先"的原则。

3. spring.handlers 的 SPI 机制：
   类似 Java 的 ServiceLoader，Spring 通过 META-INF/spring.handlers 文件
   在类路径下发现自定义命名空间处理器。第三方库可以自带命名空间（如 MyBatis-Spring 的 <mybatis:scan>）。

4. p/c 命名空间的设计洞察：
   将 XML 属性直接转换为 Bean 属性，是一种"隐式映射"（Convention over Configuration），
   减少了 XML 的冗长，但牺牲了标签嵌套带来的灵活性（如集合类型的注入）。
