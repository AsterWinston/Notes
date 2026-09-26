# 1. 创建项目

# 2. 创建第一个Spring模块

# 3. 引入依赖
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	
	<groupId>cw.spring.study</groupId>
	<artifactId>spring-study-001</artifactId>
	<version>1.0-SNAPSHOT</version>
	<!-- 导包方式为 jar -->
	<packaging>jar</packaging>
	
	<properties>
		<!-- JDK 使用 17 -->
		<maven.compiler.source>17</maven.compiler.source>
		<maven.compiler.target>17</maven.compiler.target>
	</properties>
	
	<!-- 引入项目所需要的依赖 -->
	<dependencies>
		<!-- spring 基础依赖 -->
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-context</artifactId>
			<version>6.0.8</version>
		</dependency>
		<!-- 单元测试依赖 -->
		<dependency>
			<groupId>junit</groupId>
			<artifactId>junit</artifactId>
			<version>4.13.2</version>
			<scope>test</scope>
		</dependency>
	</dependencies>

</project>

```

# 4. 定义普通Java类
```
/**
 * ClassName: User
 * Package: cw.spring.study.pojo
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-25 17:20
 * @Version 1.0
 */
public class User {
    private String name;
    private Integer age;
    
    public User() {
    }
    
    public User(String name, Integer age) {
        this.name = name;
        this.age = age;
    }
    
    public String getName() {
        return name;
    }
    
    public void setName(String name) {
        this.name = name;
    }
    
    public Integer getAge() {
        return age;
    }
    
    public void setAge(Integer age) {
        this.age = age;
    }
    
    @Override
    public String toString() {
        return "User{" + "name='" + name + '\'' + ", age=" + age + '}';
    }
}

```

# 5. 编写Spring配置文件
```
Spring 配置文件放在类根路径下，即 resource 目录下
该文件放在类的根路径下方便后期的移植
放在resources目录下，相当于放在类的根路径下

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
	<!-- spring的配置文件 -->
	<!-- 在配置文件中配置bean，spring才能帮助我们管理这个对象 -->
	<!--
		id: bean的唯一标识，对象的唯一标识
		class: 类的全路径，用来指定要创建的java对象的类名
	 -->
	<bean id="userBean" class="cw.spring.study.pojo.User" />
</beans>

```

# 6. 编写测试类
```
public class Test {
    @org.junit.Test
    public void tset1() {
        // 第一步：获取Spring容器对象。
        // ApplicationContext 翻译为：应用上下文。其实就是Spring容器。
        // ApplicationContext 是一个接口。
        // ApplicationContext 接口下有很多实现类。其中有一个实现类叫做：ClassPathXmlApplicationContext
        // ClassPathXmlApplicationContext 专门用于从类路径当中加载spring配置文件的一个Spring上下文对象。
        // 这行代码只要执行，就相当于启动了Spring容器，同时会解析spring.xml（Spring配置文件名）文件，
        // 并且实例化在Spring配置文件中配置的所有的bean对象，创建完成对象后会将其放到spring容器当中。
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    
        // 第二步：根据bean的id从Spring容器中获取这个对象。（id Bean的唯一标识）
        Object userBean = applicationContext.getBean("userBean");
        System.out.println(userBean);
    }
}

```

# 7. 源码解析：第一行代码发生了什么

```
new ClassPathXmlApplicationContext("spring.xml") 这行代码到底做了什么？

ClassPathXmlApplicationContext 继承链：
  ClassPathXmlApplicationContext
    → AbstractXmlApplicationContext
      → AbstractRefreshableConfigApplicationContext
        → AbstractRefreshableApplicationContext
          → AbstractApplicationContext
            → DefaultResourceLoader
              → ...

简化版源码追踪：

// 1. 构造器入口
public ClassPathXmlApplicationContext(String... configLocations) throws BeansException {
    this(configLocations, true, null); // refresh=true
}

// 2. 核心构造器
public ClassPathXmlApplicationContext(
        String[] configLocations, boolean refresh, ApplicationContext parent) {
    super(parent);                           // 初始化父容器相关属性
    setConfigLocations(configLocations);     // 保存配置路径 [ "spring.xml" ]
    if (refresh) {
        refresh();  // ★ 触发容器初始化！
    }
}

// 3. refresh() 是 AbstractApplicationContext 的核心方法
//    包含 13 个子步骤（详见第17章源码解析）
//    其中关键步骤：
//    - obtainFreshBeanFactory() → 创建 DefaultListableBeanFactory + 加载 XML
//    - finishBeanFactoryInitialization() → 实例化所有单例 Bean

// 4. getBean("userBean") 执行时：
//    → Bean 已在 refresh 阶段创建完成
//    → 直接从 singletonObjects 缓存返回
//    → 不会再次创建实例
```
