# 1. 引入子配置文件
```
spring配置文件有多个，并且可以在spring的核心配置文件中使用import进行引入子配置文件，我们可以将组件扫描单独定义到一个配置文件中
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context https://www.springframework.org/schema/context/spring-context.xsd">
	
	<!-- 组件扫描 -->
	<context:component-scan base-package="cw.study.spring"/>

</beans>

在核心配置文件中使用import标签引入子配置文件
<!-- 组件扫描 -->
<!-- <context:component-scan base-package="cw.study.spring"/> -->
<!-- 将组件扫描配置单独定义到一个配置文件中 -->
<!-- 引入子配置文件 -->
<import resource="spring-component-scan.xml"/>

在实际开发中，配置可能会很多，service单独配置到一个文件中，dao单独配置到一个文件中，然后在核心配置文件中引入，养成好习惯。
```

# 2. 源码解析：<import> 标签的处理机制

```
<import resource="spring-dao.xml"/> 标签在 DefaultBeanDefinitionDocumentReader 中处理：

// parseDefaultElement() 中：
if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
    importBeanDefinitionResource(ele);
}

// importBeanDefinitionResource() 核心逻辑：
protected void importBeanDefinitionResource(Element ele) {
    // 1. ★ 获取 resource 属性值
    String location = ele.getAttribute(RESOURCE_ATTRIBUTE);

    // 2. ★ 解析占位符（如 ${module.name}）
    location = getReaderContext().getEnvironment().resolveRequiredPlaceholders(location);

    Set<Resource> actualResources = new LinkedHashSet<>(4);

    // 3. 判断是绝对路径还是相对路径
    boolean absoluteLocation = false;
    try {
        absoluteLocation = ResourcePatternUtils.isUrl(location)
                || ResourceUtils.toURI(location).isAbsolute();
    } catch (URISyntaxException ex) { ... }

    if (absoluteLocation) {
        // ★ 绝对路径：直接加载
        int importCount = getReaderContext().getReader()
                .loadBeanDefinitions(location, actualResources);
    } else {
        // ★ 相对路径：相对于当前配置文件的路径
        int importCount = getReaderContext().getReader()
                .loadBeanDefinitions(location, actualResources);
    }

    // 4. ★ 实际的加载：递归调用 XmlBeanDefinitionReader.loadBeanDefinitions()
    //    将子配置文件中的 <bean> 全部注册到同一个 BeanFactory 中

    // 5. 触发导入事件
    Resource[] actResArray = actualResources.toArray(new Resource[0]);
    getReaderContext().fireImportProcessed(location, actResArray,
            extractSource(ele));
}

★ 关键点：
1. <import> 只是物理上将配置拆分到多个文件
2. 最终所有 BeanDefinition 都注册到同一个 DefaultListableBeanFactory 中
3. 不支持同名 Bean 的覆盖（除非设置 allowBeanDefinitionOverriding=true）
4. 相对路径是相对于当前 XML 配置文件的位置，不是 classpath 根路径
5. 如果使用通配符，如 <import resource="classpath*:spring-*.xml"/> 可以导入多个文件

Java Config 方式的等价操作：
@Configuration
@Import({DaoConfig.class, ServiceConfig.class})
public class AppConfig { }
```
