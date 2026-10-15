```
我们都知道编写数据源的时候是需要连接数据库的信息的，例如：driver url username password等信息。这些信息可以单独写到一个属性配置文件中吗，这样用户修改起来会更加的方便

以下是jdbc.properties
jdbc.driverClass=com.mysql.cj.jdbc.Driver
jdbc.url=jdbc:mysql://localhost:3306/spring6
jdbc.username=root
jdbc.password=123

以下是spring.xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
                           http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">

    <!--
        引入外部的properties文件
        第一步：引入context命名空间。
          xmlns:context="http://www.springframework.org/schema/context"
          http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
        第二步：使用标签context:property-placeholder的location属性来指定属性配置文件的路径。
                    location默认从类的根路径下开始加载资源。
    -->
    <context:property-placeholder location="jdbc.properties"/>

    <!--配置数据源-->
    <bean id="ds" class="com.powernode.spring6.jdbc.MyDataSource">
        <!--怎么取值呢？第三步：${key}-->
        <property name="driver" value="${jdbc.driverClass}"></property>
        <property name="url" value="${jdbc.url}"></property>
        <!--加前缀是由于spring加载变量是优先从系统变量中进行加载，username系统变量已经存在-->
        <property name="username" value="${jdbc.username}"></property>
        <property name="password" value="${jdbc.password}"></property>
    </bean>

</beans>

```

# 1. 源码解析：属性占位符解析机制

## 1.1 PropertySourcesPlaceholderConfigurer 的实现机制

```
<context:property-placeholder location="jdbc.properties"/>

这个 XML 标签背后注册的是 PropertySourcesPlaceholderConfigurer，
它实现了 BeanFactoryPostProcessor 接口。

NamespaceHandler 处理流程：
ContextNamespaceHandler → PropertyPlaceholderBeanDefinitionParser
  → 注册 PropertySourcesPlaceholderConfigurer 的 BeanDefinition

PropertySourcesPlaceholderConfigurer 的类继承结构：

BeanFactoryPostProcessor（接口）
  └── PropertiesLoaderSupport
       └── PropertyResourceConfigurer
            ├── PropertyOverrideConfigurer（覆盖配置）
            │   用于强制覆盖 Bean 的属性值
            │
            └── PropertySourcesPlaceholderConfigurer ★（占位符替换）
                实现了 BeanFactoryPostProcessor 和 EnvironmentAware
                继承自 PlaceholderConfigurerSupport
```

## 1.2 postProcessBeanFactory() — 占位符处理入口

```
PropertySourcesPlaceholderConfigurer.postProcessBeanFactory()：

public class PropertySourcesPlaceholderConfigurer
        extends PlaceholderConfigurerSupport implements EnvironmentAware {

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory)
            throws BeansException {

        if (this.propertySources == null) {
            // 从 Environment 中获取 PropertySources
            this.propertySources = new MutablePropertySources();
            // 优先查找 Spring Environment 中的属性
            if (this.environment != null) {
                this.propertySources.addLast(
                    new PropertySource<Environment>("environmentProperties",
                            this.environment) {
                        @Override
                        @Nullable
                        public String getProperty(String key) {
                            return this.source.getProperty(key);
                        }
                    }
                );
            }
            // 从加载的资源文件（jdbc.properties）中创建 PropertySource
            PropertySource<?> localPropertySource =
                    loadProperties(new DefaultResourceLoader());
            if (localPropertySource != null) {
                this.propertySources.addLast(localPropertySource);
            }
        }

        // ★ 关键：创建一个 StringValueResolver 来处理占位符
        //   将其注册为嵌入值解析器（embedded value resolver）
        //   后续所有包含 ${...} 的地方都会通过这个 Resolver 来解析
        processProperties(beanFactory, // 创建 StringValueResolver
                (ConfigurablePropertyResolver) this.appliedPropertySources::getProperty);

        // ★ 实际处理 BeanDefinition 中的占位符
        doProcessProperties(beanFactory, valueResolver);
    }
}
```

## 1.3 占位符解析的两条路径

```
Spring 中有两条占位符解析路径：

路径 1: XML 配置文件中的 ${...} 占位符 — BeanDefinition 级别
─────────────────────────────────────────────────────
在 XML 中写 <property name="url" value="${jdbc.url}"/> 时，
${jdbc.url} 被解析为 TypedStringValue 存储在 BeanDefinition 中。

在 doProcessProperties() 中，Spring 遍历所有 BeanDefinition，
对其中的每个 PropertyValue 和 ConstructorArgumentValue 进行占位符替换：

protected void doProcessProperties(
        ConfigurableListableBeanFactory beanFactoryToProcess,
        StringValueResolver valueResolver) {

    // ★ 创建 BeanDefinitionVisitor
    BeanDefinitionVisitor visitor = new BeanDefinitionVisitor(valueResolver);

    // ★ 遍历所有 BeanDefinition
    String[] beanNames = beanFactoryToProcess.getBeanDefinitionNames();
    for (String curName : beanNames) {
        BeanDefinition bd = beanFactoryToProcess.getBeanDefinition(curName);
        try {
            visitor.visitBeanDefinition(bd);
            // visitBeanDefinition 内部会：
            // 1. 遍历所有 PropertyValue
            // 2. 对每个 TypedStringValue 调用 valueResolver.resolveStringValue()
            //    将 ${jdbc.url} 替换为 jdbc:mysql://localhost:3306/spring6
            // 3. 对 ConstructorArgumentValues 也做同样处理
            // 4. 对 factoryBeanName、factoryMethodName、initMethodName 等做替换
        } catch (Exception ex) {
            throw new BeanDefinitionStoreException(bd.getResourceDescription(),
                    curName, ex.getMessage(), ex);
        }
    }

    // ★ 注册全局的 embeddedValueResolver
    //    后续 @Value("${...}") 注解也会通过这个 Resolver 来解析
    beanFactoryToProcess.addEmbeddedValueResolver(valueResolver);
}


路径 2: @Value("${...}") 注解中的占位符 — 字段级别
─────────────────────────────────────────────────────
在 AutowiredAnnotationBeanPostProcessor 处理 @Value 注解时：

1. @Value("${jdbc.url}") 被解析到 InjectionMetadata
2. 在注入时，调用 beanFactory.resolveDependency() 
3. 内部调用 StringValueResolver.resolveStringValue("${jdbc.url}")
4. 从 PropertySources 中查找 jdbc.url 的值
5. 返回 "jdbc:mysql://localhost:3306/spring6"
6. 通过 TypeConverter 转换类型
7. 反射设置到字段/setter 上
```

## 1.4 PropertySources 的属性查找优先级

```
PropertySources 是一个有序的属性源链，查找时按优先级从高到低依次搜索：

查找顺序（优先级从高到低）：
1. 系统属性（System.getProperties()）
2. 系统环境变量（System.getenv()）
3. servletContextInitParams（Servlet 上下文初始化参数）
4. servletConfigInitParams（Servlet 配置初始化参数）
5. JNDI 属性
6. jdbc.properties 等自定义属性文件
7. application.properties / application.yml（Spring Boot）
8. 以上都没找到 → 抛 IllegalArgumentException

★ 这就是为什么 username 属性会被系统环境变量覆盖的原因！
  系统环境变量中通常有 USERNAME（Windows）或 USER（Linux），
  如果你在 jdbc.properties 中定义了 username=root，
  会被优先级更高的系统环境变量所覆盖！

解决方法：
1. 使用有区分度的 key：jdbc.username 替代 username
2. 在占位符中设置默认值：${username:root}
3. 设置 ignoreUnresolvablePlaceholders = true 来忽略无法解析的占位符
4. 自定义 PropertySourcesPlaceholderConfigurer 调整查找优先级

示例：
<context:property-placeholder location="jdbc.properties"
    local-override="true"     ← 本地属性优先于系统属性
    ignore-unresolvable="true"/> ← 找不到时保留原样
```

## 1.5 占位符嵌套与默认值语法

```
Spring 占位符支持以下高级语法：

1. 默认值：${key:defaultValue}
   如果 jdbc.url 没定义，使用 default
   例如：${jdbc.url:jdbc:mysql://localhost:3306/mydb}

2. 嵌套占位符：${${env}.url}
   先从 env 获取环境名（如 dev），再拼接为 dev.url
   例如：${spring.profiles.active:dev} → 得到 "dev"
         然后拼接得到 ${dev.jdbc.url}

3. 多级嵌套：${${${env}}}
   但不推荐过度嵌套，可读性差

源码中的解析：
class PropertyPlaceholderHelper {

    protected String parseStringValue(
            String value, PlaceholderResolver placeholderResolver,
            @Nullable Set<String> visitedPlaceholders) {

        // 递归解析 ${...} 占位符
        int startIndex = value.indexOf(this.placeholderPrefix);  // 搜索 "${"
        while (startIndex != -1) {
            int endIndex = findPlaceholderEndIndex(value, startIndex + this.placeholderPrefix.length());
            String placeholder = value.substring(startIndex + this.placeholderPrefix.length(), endIndex);

            // 支持默认值
            String defaultValue = null;
            int separatorIndex = placeholder.indexOf(this.valueSeparator);  // 搜索 ":"
            if (separatorIndex != -1) {
                defaultValue = placeholder.substring(separatorIndex + this.valueSeparator.length());
                placeholder = placeholder.substring(0, separatorIndex);  // 去掉默认值部分
            }

            // ★ 递归解析（支持嵌套占位符）
            String propVal = placeholderResolver.resolvePlaceholder(placeholder);
            if (propVal == null) {
                propVal = defaultValue;  // 使用默认值
            }
            if (propVal != null) {
                // ★ 递归处理 propVal 中可能还存在的占位符
                propVal = parseStringValue(propVal, placeholderResolver, visitedPlaceholders);
                // 替换当前占位符
                value = value.substring(0, startIndex) + propVal + value.substring(endIndex + 1);
            }
            // 继续扫描下一个 ${...}
            startIndex = value.indexOf(this.placeholderPrefix, startIndex + propVal.length());
        }
        return value;
    }
}
```

## 1.6 设计思想总结

```
1. BeanFactoryPostProcessor 时机：
   PropertySourcesPlaceholderConfigurer 在 Bean 创建之前就完成了占位符替换，
   这意味着 Bean 被实例化时，拿到的已经是替换后的属性值。
   这个时机发生在 refresh() → invokeBeanFactoryPostProcessors() 阶段，
   早于 Bean 的实例化。

2. 分层属性源（PropertySources）：
   Spring 的 PropertySources 采用"有序链"模式，而不是单一的 Properties 对象，
   这使得不同来源的属性可以有优先级，类似操作系统的 PATH 环境变量机制。

3. 嵌入式值解析器（EmbeddedValueResolver）：
   StringValueResolver 作为全局的功能性组件，统一了 XML 占位符和 @Value 注解
   的解析逻辑。这体现了 DRY（Don't Repeat Yourself）原则。

4. 与 Spring Boot 的关系：
   Spring Boot 在此基础上扩展了 application.properties/yml 的支持、
   多环境配置（application-{profile}.properties）、配置中心集成等，
   但其底层依赖的仍然是 PropertySourcesPlaceholderConfigurer。
```
