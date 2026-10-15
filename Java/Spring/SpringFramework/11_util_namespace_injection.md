# 1. util 命名空间注入
```
使用util命名空间可以让配置复用。
使用util命名空间的前提是：在spring配置文件头部添加配置信息。
如下：
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:util="http://www.springframework.org/schema/util"
       xsi:schemaLocation="
       		http://www.springframework.org/schema/beans 												
       		http://www.springframework.org/schema/beans/spring-beans.xsd
            http://www.springframework.org/schema/util 												
            http://www.springframework.org/schema/util/spring-util.xsd">

</beans>

package com.powernode.spring6.beans;

import java.util.Properties;

public class MyDataSource1 {
	// Properties属性类对象，这是一个Map集合，
    // key和value都是String类型。
    private Properties properties;

    public void setProperties(Properties properties) {
        this.properties = properties;
    }

    @Override
    public String toString() {
        return "MyDataSource1{" +
                "properties=" + properties +
                '}';
    }
}

package com.powernode.spring6.beans;

import java.util.Properties;

public class MyDataSource2 {
    private Properties properties;

    public void setProperties(Properties properties) {
        this.properties = properties;
    }

    @Override
    public String toString() {
        return "MyDataSource2{" +
                "properties=" + properties +
                '}';
    }
}

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:util="http://www.springframework.org/schema/util"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
                           http://www.springframework.org/schema/util http://www.springframework.org/schema/util/spring-util.xsd">

    <!--引入util命名空间
        在spring的配置文件头部添加：
        		xmlns:util="http://www.springframework.org/schema/util"
		        http://www.springframework.org/schema/util http://www.springframework.org/schema/util/spring-util.xsd
    -->
    <util:properties id="prop">
        <prop key="driver">com.mysql.cj.jdbc.Driver</prop>
        <prop key="url">jdbc:mysql://localhost:3306/spring6</prop>
        <prop key="username">root</prop>
        <prop key="password">123</prop>
    </util:properties>

    <!--数据源1-->
    <bean id="ds1" class="com.powernode.spring6.jdbc.MyDataSource1">
        <property name="properties" ref="prop"/>
    </bean>

    <!--数据源2-->
    <bean id="ds2" class="com.powernode.spring6.jdbc.MyDataSource2">
        <property name="properties" ref="prop"/>
    </bean>

</beans>

```

# 2. 源码解析：util 命名空间的元素与解析

## 2.1 UtilNamespaceHandler 注册的全部元素

```
在 spring-beans 模块的 META-INF/spring.handlers 中：
http\://www.springframework.org/schema/util=\
  org.springframework.beans.factory.xml.UtilNamespaceHandler

UtilNamespaceHandler.init() 注册了 5 种元素解析器：

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
            new PropertiesBeanDefinitionParser());
}
```

## 2.2 各 util 元素的用途与源码

```
<util:constant> — 引用静态常量
  示例：<util:constant id="maxValue" static-field="java.lang.Integer.MAX_VALUE"/>
  源码：ConstantBeanDefinitionParser → BeanDefinition(beanClass=Integer.MAX_VALUE.getClass())
        最终 getBean("maxValue") → 2147483647

<util:property-path> — 引用嵌套 Bean 的属性
  示例：<util:property-path id="dbName" path="dataSource.url"/>
  源码：PropertyPathBeanDefinitionParser → 解析 dataSource.url → getBean("dataSource").getUrl()

<util:list> — 创建可复用的 List
  示例：<util:list id="myList" list-class="java.util.LinkedList">
          <value>a</value><ref bean="foo"/></util:list>
  源码：ListBeanDefinitionParser → BeanDefinition(beanClass=LinkedList, propertyValues=[0→a, 1→ref(foo)])

<util:set> — 创建可复用的 Set
  示例：<util:set id="mySet" set-class="java.util.TreeSet">
          <value>b</value></util:set>
  源码：SetBeanDefinitionParser → BeanDefinition(beanClass=TreeSet, ...)

<util:map> — 创建可复用的 Map
  示例：<util:map id="myMap" map-class="java.util.TreeMap">
          <entry key="k" value="v"/></util:map>
  源码：MapBeanDefinitionParser → BeanDefinition(beanClass=TreeMap, ...)

<util:properties> — 创建可复用的 Properties
  示例：<util:properties id="prop" location="jdbc.properties">
          <prop key="driver">com.mysql.cj.jdbc.Driver</prop></util:properties>
  源码：PropertiesBeanDefinitionParser → BeanDefinition(beanClass=Properties, ...)
```

## 2.3 <util:properties> 的完整解析源码

```
// PropertiesBeanDefinitionParser.doParse()
protected void doParse(Element element, BeanDefinitionBuilder builder) {
    super.doParse(element, builder);  // 解析 <prop> 子元素

    // ★ 支持 location 属性：加载外部 .properties 文件
    builder.addPropertyValue("location",
            new TypedStringValue(element.getAttribute("location")));

    // ★ 支持 properties-ref 属性：引用其他的 Properties Bean
    builder.addPropertyReference("properties",
            element.getAttribute("properties-ref"));

    // ★ 支持 local-override 属性：本地覆盖外部
    builder.addPropertyValue("localOverride",
            element.getAttribute("local-override"));
}

// 解析 <prop> 子元素的父类逻辑：
// 遍历 <prop key="x">value</prop> → Properties.setProperty("x", "value")
```

## 2.4 util 命名空间 vs 内嵌集合的对比

```
不使用 util：
  <bean id="ds1" class="MyDataSource1">
    <property name="properties">
      <props>  ← 每个使用处都要重复写 <prop>
        <prop key="driver">...</prop>
        <prop key="url">...</prop>
      </props>
    </property>
  </bean>
  <bean id="ds2" class="MyDataSource2">
    <property name="properties">
      <props>  ← 重复！
        <prop key="driver">...</prop>
        ...
      </props>
    </property>
  </bean>

使用 util：
  <util:properties id="prop">  ← ★ 定义一次，多处引用
    <prop key="driver">...</prop>
    <prop key="url">...</prop>
  </util:properties>
  <bean id="ds1" class="MyDataSource1">
    <property name="properties" ref="prop"/>  ← 引用
  </bean>
  <bean id="ds2" class="MyDataSource2">
    <property name="properties" ref="prop"/>  ← 引用
  </bean>
```
