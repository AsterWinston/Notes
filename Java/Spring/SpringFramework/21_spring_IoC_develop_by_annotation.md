# 1. 基于注解的 IoC 开发

Spring6 倡导全注解开发
注解的存在主要是为了简化 XML 的配置。
注解其实就是一个标记

# 2. 回顾自定义注解
```
元注解：标注注解的注解
@Target注解：用来修饰注解可以出现的位置
@Target(value = {ElementType.TYPE, ElementType.FIELD})，Target标注的注解可以出现在类或字段上
@Target(value = {ElementType.TYPE})，Target标注的注解可以出现在类上
使用某个注解的时候，如果注解的属性名是value的话，value可以省略。
@Target({ElementType.TYPE})
使用某个注解的时候，如果注解的属性值是数组，并且数组中只有一个元素，大括号可以省略。
@Target(ElementType.TYPE)
@Retention：也是一个元注解，用来标注注解最终可以保留到什么时刻
如果注解要保存到class文件当中，并且可以被反射机制读取，则使用@Retention(RetentionPolicy.RUNTIME)
@Retention(RetentionPolicy.SOURCE)，表示Retention标注的注解只能保留到源文件中，字节码和运行时看不到Retention标注的注解
自定义注解与定义注解属性：

// 标注该注解可以用在类上
@Target(ElementType.TYPE)
// 标注@Component注解最终保留在class文件当中，并且可以被反射机制读取。
@Retention(RetentionPolicy.RUNTIME)
public @interface Component {

    // 定义注解的属性
    // String是属性类型
    // value是属性名
    String value();

    // 其他的属性
    // 属性类型String
    // 属性名是name
    //String name();

    // 数组属性
    // 属性类型是：String[]
    // 属性名：names
    //String[] names();

    //int[] ages();

    //int age();
}

```

# 3. 回顾自定义注解的使用
```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Component {
    String value();
}

@Component("user")
public class User {}

```

# 4. 通过反射获取注解
```
@org.junit.Test
public void testGetAnnotationByReflect() throws Exception{
    // 将类加载到JVM中
    Class<?> clazz = Class.forName("cw.study.spring.bean.User");
    // 判断类上是否有自定义的Component注解
    if (clazz.isAnnotationPresent(Component.class)) {
        // 如果有，获取该注解
        Component component = clazz.getAnnotation(Component.class);
        // 获取该类上注解的属性值
        System.out.println(component.value());
    }
}

```

# 5. 组件扫描原理
```
需求：给定一个包名，将这个包下的所有的带@Component注解的类进行实例化
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Component {
    String value();
}

@Component("user")
public class User {}

@Component("cat")
public class Cat {}

public class Person {}

@org.junit.Test
public void componentScan() {
    // 已知的包名如下，需要扫描包下所有的类，将有Component注解的类实例化
    String packageName = "cw.study.spring.bean";
    // 包名其实就是目录名，将包名转化为目录
    // . 在正则中代表任意字符，需要进行转义，而 \\ 表示 \ 所以使用 \\.
    String directoryName = packageName.replaceAll("\\.", "/");
    // 软件包是在类路径下的，所以可以使用系统加载器加载目录获取URL
    URL url = ClassLoader.getSystemClassLoader().getResource(directoryName);
    // 通过URL获取包对应的绝对路径
    String path = url.getPath();
    System.out.println("path = " + path);
    // 根据绝对路径创建目录对应的对象
    File file = new File(path);
    // 获取目录下的文件对象
    File[] files = file.listFiles();
    System.out.println(files.length);
    // 遍历每个文件对象
    Arrays.stream(files).forEach(f -> {
        try {
            // 获取文件名
            String fName = f.getName();
            // System.out.println("fName = " + fName);
            // 得到类名
            String className = packageName + "." + fName.split("\\.")[0];
            // 加载类到JVM
            Class<?> clazz = Class.forName(className);
            // 判断类上是否有Component注解
            if (clazz.isAnnotationPresent(Component.class)) {
                // 获取注解
                Component component = clazz.getAnnotation(Component.class);
                // 获取注解的属性（Bean的id）
                String id = component.value();
                System.out.println(id);
                // 实例化对象
                Object o = clazz.getDeclaredConstructor().newInstance();
                System.out.println(o);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    });
}

```

# 6. 声明Bean的注解
```
负责声明Bean的注解，常见的包括四个：
@Component：组件
@Controller：控制器
@Service：业务
@Repository：仓库（Dao）
上面四个注解中，@Controller，@Service，@Repository 都为 @Component 的别名（@AliasFor），其实这四个注解的功能都一样，用哪个都可以，但是在不同用途的 Bean 上使用不同的注解可以增强程序的可读性：
普通 bean：Component
控制器类上使用：Controller（控制层）
service类上使用：Service（业务层）
dao类上使用：Repository（持久层）
上面四个注解中，都是只有一个value属性，用来指定 bean 的 id
```
## 6.1 @Component
```
@Target({ElementType.TYPE}) // 只能使用在类上
@Retention(RetentionPolicy.RUNTIME) // 可以通过反射获取该注解
@Documented
@Indexed
public @interface Component {
    String value() default "";
}
```
## 6.2 @Controller
```
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Controller {
    // Component的别名
    @AliasFor(
        annotation = Component.class
    )
    String value() default "";
}
```
## 6.3 @Service
```
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Service {
    // Component的别名
    @AliasFor(
        annotation = Component.class
    )
    String value() default "";
}
```
## 6.4 Repository
```
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Repository {
    // Component的别名
    @AliasFor(
        annotation = Component.class
    )
    String value() default "";
}

```

# 7. Spring注解的使用
```
第一步：加入aop的依赖
当加入spring-context依赖之后，会自动关联加入aop的依赖

第二步：在配置文件中添加context命名空间
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
						   http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">

</beans>

第三步：在配置文件中指定扫描的包
在配置文件中指定扫描的包，告诉Spring需要进行扫描的包在什么位置
Spring会扫描指定的包中的类，及其子包中的类，如果类被声明Bean的注解标记，则会创建该类的实例并将其放到Spring容器中
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
						   http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">

	<!-- 指定扫描的包，告诉Spring需要进行扫描的包在什么位置 -->
	<context:component-scan base-package="cw.study.spring"/>
	
</beans>

第四步：在Bean类上使用注解
@Component("user")
public class User {}

@Controller("userController")
public class UserController {}

测试
@org.junit.Test
public void test01() {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    Object user = applicationContext.getBean("user");
    Object userController = applicationContext.getBean("userController");
    System.out.println(user);
    System.out.println(userController);
}

注意点
如果注解的属性名是value，那么value是可以省略的。
如果把value属性彻底去掉，spring会为Bean自动取名，默认名字的规律是：Bean类名首字母小写即可。

如果要对多个包进行扫描，有两种解决方案：
    第一种：在配置文件中指定多个包，用逗号隔开。
        <context:component-scan base-package="cw.study.spring.bean, cw.study.spring.controller"/>
    第二种：指定多个包的共同父包。
        Spring会扫描指定的包中的类，及其子包中的类
        使用此方式会牺牲一些效率
        <context:component-scan base-package="cw.study.spring"/>
```

# 8. 选择性实例化Bean
```
选择性实例化 Bean，就是只选择满足某些条件的类进行 bean 的实例化，或者排除满足某些条件的类，对这些类不进行 Bean 的实例化
package cw.spring.bean;

import org.springframework.stereotype.Component;
import org.springframework.stereotype.Controller;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Service;

/**
 * ClassName: A
 * Package: cw.spring.bean
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-14 11:42
 * @Version 1.0
 */
@Component
public class A {
    public A() {
        System.out.println("A的无参数构造方法执行");
    }
}

@Controller
class B {
    public B() {
        System.out.println("B的无参数构造方法执行");
    }
}

@Service
class C {
    public C() {
        System.out.println("C的无参数构造方法执行");
    }
}

@Repository
class D {
    public D() {
        System.out.println("D的无参数构造方法执行");
    }
}

@Controller
class E {
    public E() {
        System.out.println("E的无参数构造方法执行");
    }
}

方式一：use-default-filters=“false” + context:include-filter
	use-default-filters=“false”：不使用 spring 默认的实例化规则，即所有带有声明 bean 的注解全部失效，用注解 Component、Controller、Service、Repository 标注的 bean 都不进行实例化（让所有的声明 bean 的注解失效）
		use-default-filters 的默认值为 true
	<context:include-filter type="annotation" expression="org.springframework.stereotype.Component"/>：
        表示被 Component 注解标注的 bean 进行实例化
        type 属性用于指定过滤类的条件的类型，即根据什么进行过滤，type=“annotation”，根据类的注解进行过滤
        expression 属性用于指定过滤的条件，expression=“org.springframework.stereotype.Component”，只有被Component注解标注的类才进行实例化
	由于其他三个注解只是 Component 的别名，所以包含 Component 也就包含其他三种注解
	注意：使用context:include-filter，use-default-filters的属性值必须为false，否则context:include-filter无效
<!-- 指定要进行扫描的包 -->
<context:component-scan base-package="cw.spring.bean" use-default-filters="false">
  <!-- 被 Component 注解标注的 bean 进行实例化 -->
  <!-- 由于其他三个注解只是 Component 的别名，所以包含 Component 也就包含其他三种注解 -->
  <!-- <context:include-filter type="annotation" expression="org.springframework.stereotype.Component"/> -->
  <!-- 被 Controller 注解标注的 bean 进行实例化 -->
  <context:include-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
</context:component-scan>

方式二：**use-default-filters=“true” + **context:exclude-filter
	use-default-filters=“true”：使用 spring 默认的实例化规则，即所有带有声明 bean 的注解全部生效，用注解 Component、Controller、Service、Repository 标注的 bean 都进行实例化
		use-default-filters 的默认值为 true
	<context:exclude-filter type="annotation" expression="org.springframework.stereotype.Component"/>：
		表示被 Component 注解标注的 bean 不进行实例化，排除被 Component 注解标注的 bean
	由于其他三个注解只是 Component 的别名，所以排除 Component 也就排除了其他三种注解
	
<!-- 指定要进行扫描的包 -->
<context:component-scan base-package="cw.spring.bean">
  <!-- 排除被 Component 注解标注的 bean，这些 bean 不参与实例化 -->
  <!-- 由于其他三个注解只是 Component 的别名，所以排除 Component 也就排除了其他三种注解 -->
  <!-- <context:exclude-filter type="annotation" expression="org.springframework.stereotype.Component"/> -->
  <!-- 排除被 Controller 注解标注的 bean，这些 bean 不参与实例化 -->
  <context:exclude-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
</context:component-scan>

```

# 9. 负责注入的注解
```
@Component @Controller @Service @Repository 这四个注解只是用来声明 Bean 的，声明后，这些 Bean 将会被实例化。
给 Bean 的属性赋值，使用如下注解：
    @Value：org.springframework.beans.factory.annotation.Value
    @Autowired：org.springframework.beans.factory.annotation.Autowired
    @Qualifier：org.springframework.beans.factory.annotation.Qualifier
    @Resource：javax.annotation.Resource
```
## 9.1 @Value：注入配置文件中的值（字符串、表达式等）
```
用于简单类型注入
import org.springframework.beans.factory.annotation.Value;
使用的是反射机制直接注入的，并不属于set/construct注入
当属性的类型是简单类型时，可以使用@Value注解进行注入。
@Value注解用于代替<property name="" value=""/>
    
@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Value {
	String value();
}

用在属性上
	@value 可以直接写在属性上，可以不用提供对应的 set 方法（不会使用set注入，更不会使用构造注入，而是直接字段注入）
package cw.spring.bean;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

/**
 * ClassName: MyDataSource
 * Package: cw.spring.bean
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-14 12:13
 * @Version 1.0
 */
@Component
public class MyDataSource {
    @Value("com.mysql.cj.jdbc.Driver")
    private String driver;
    @Value("jdbc:mysql://localhost:3306/spring")
    private String url;
    @Value("root")
    private String username;
    @Value("123456")
    private String password;
    
    @Override
    public String toString() {
        return "MyDataSource{" + "driver='" + driver + '\'' + ", url='" + url + '\'' + ", username='" + username + '\'' + ", password='" + password + '\'' + '}';
    }
}

用在 set 方法上
    @value 也可以写在属性对应的 set 方法上，实现属性值的注入
    为了简化代码，一般不提供 set 方法，直接在属性上使用@Value 注解完成属性赋值。
package cw.spring.bean;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

/**
 * ClassName: MyDataSource
 * Package: cw.spring.bean
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-14 12:13
 * @Version 1.0
 */
@Component
public class MyDataSource {
    private String driver;
    private String url;
    private String username;
    private String password;
    
    @Value("com.mysql.cj.jdbc.Driver")
    public void setDriver(String driver) {
        this.driver = driver;
    }
    
    @Value("jdbc:mysql://localhost:3306/spring")
    public void setUrl(String url) {
        this.url = url;
    }
    
    @Value("root")
    public void setUsername(String username) {
        this.username = username;
    }
    
    @Value("123456")
    public void setPassword(String password) {
        this.password = password;
    }
    
    @Override
    public String toString() {
        return "MyDataSource{" + "driver='" + driver + '\'' + ", url='" + url + '\'' + ", username='" + username + '\'' + ", password='" + password + '\'' + '}';
    }
}

用在构造方法的形参上
	@Value 注解也可以用在构造方法的形参上
package cw.spring.bean;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

/**
 * ClassName: MyDataSource
 * Package: cw.spring.bean
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-14 12:13
 * @Version 1.0
 */
@Component
public class MyDataSource {
    private String driver;
    private String url;
    private String username;
    private String password;
    
    public MyDataSource(
            @Value("com.mysql.cj.jdbc.Driver") String driver, 
            @Value("jdbc:mysql://localhost:3306/spring") String url, 
            @Value("root") String username, 
            @Value("123123") String password
    ) {
        this.driver = driver;
        this.url = url;
        this.username = username;
        this.password = password;
    }
    
    @Override
    public String toString() {
        return "MyDataSource{" + "driver='" + driver + '\'' + ", url='" + url + '\'' + ", username='" + username + '\'' + ", password='" + password + '\'' + '}';
    }
}
```
## 9.2 @Autowired：按类型自动装配 Bean
```
@Autowired 注解用来注入非简单类型。
	Autowired：自动连线，或自动装配
    @Autowired，通过注解的方式进行自动装配
@Autowired 注解不需要 set 方法
单独使用 @Autowired 注解，默认根据类型进行装配，即默认是 byType，如果需要根据名字进行自动装配则需要配合 @Qualifier 注解
    @Autowired 注解是根据类型进行自动装配，如果只使用@Autowired 注解的话，Spring容器中不能存在两个相同类型的实例，如果要使用@Autowired 注解并且Spring容器中存在两个相同类型的实例，则需要配合 @Qualifier 注解，根据名称进行装配
    该注解可以标注在构造方法上、方法上、形参上、属性上、注解上，用法书写位置和@Value一样
    该注解有一个 required 属性，默认值是 true，表示在注入的时候要求被注入的 Bean 必须是存在的，如果不存在则报错。如果 required 属性设置为 false，表示注入的 Bean 存在或者不存在都没关系，存在的话就注入，不存在的话，也不报错。

@Target({ElementType.CONSTRUCTOR, ElementType.METHOD, ElementType.PARAMETER, ElementType.FIELD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Autowired {
    boolean required() default true;
}
@Autowired 注解使用的时候，可以不指定任何属性，直接使用
/**
 * ClassName: OrderDao
 * Package: cw.spring.dao
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-14 12:50
 * @Version 1.0
 */
public interface OrderDao {
    void insert();
}

/**
 * ClassName: OrderDaoImplForMySQL
 * Package: cw.spring.dao.impl
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-14 12:50
 * @Version 1.0
 */
@Repository
public class OrderDaoImplForMySQL implements OrderDao {
    @Override
    public void insert() {
        System.out.println("MySQL数据库正在保存订单信息...");
    }
}

import cw.spring.dao.OrderDao;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

/**
 * ClassName: OrderService
 * Package: cw.spring.service
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-14 12:49
 * @Version 1.0
 */
@Service("orderService")
public class OrderService {
    @Autowired
    private OrderDao orderDao;
    
    public void save() {
        orderDao.insert();
    }
}

当类中构造方法只有一个时，并且构造方法上的参数和需要注入的属性能够对应上，@Autowired 注解可以省略，如果有多个构造方法，@Autowired 不能省略的。
最好不要省略，程序的可读性更高
/**
 * ClassName: OrderService
 * Package: cw.spring.service
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-14 12:49
 * @Version 1.0
 */
@Service("orderService")
public class OrderService {
    // @Autowired
    // @Qualifier("orderDaoImplForOracle")
    private OrderDao orderDao;
    
    public OrderService(OrderDao orderDao) {
        this.orderDao = orderDao;
    }
    
    public void save() {
        orderDao.insert();
    }
}
```
## 9.3 @Qualifier：与 `@Autowired` 配合，按名称指定注入 Bean
```
@Autowired 注解和 @Qualifier 注解联合起来才可以根据名称进行装配，在@Qualifier注解中指定Bean名称。

@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER, ElementType.TYPE, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Qualifier {

	String value() default "";

}

/**
 * ClassName: OrderDaoImplForOracle
 * Package: cw.spring.dao.impl
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-14 12:59
 * @Version 1.0
 */
@Repository
public class OrderDaoImplForOracle implements OrderDao {
    @Override
    public void insert() {
        System.out.println("Oracle数据库正在保存订单信息...");
    }
}

/**
 * ClassName: OrderService
 * Package: cw.spring.service
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-14 12:49
 * @Version 1.0
 */
@Service("orderService")
public class OrderService {
    @Autowired
    @Qualifier("orderDaoImplForOracle")
    private OrderDao orderDao;
    
    public void save() {
        orderDao.insert();
    }
}
```
## 9.4 @Resource：JDK 提供的注解，按名称注入 Bean（也可按类型）
```
    @Resource注解可以完成非简单类型注入。
    @Resource注解是JDK扩展包中的，也就是说属于JDK的一部分，而@Autowired注解是Spring框架自己的。
    @Resource是JDK标准规范中的，更具有通用性，更推荐使用
    @Resource注解默认根据名称装配byName，未指定name时，使用属性名作为name。通过name找不到的话会自动启动通过类型byType装配。
    @Resource注解用在属性上、setter方法上、构造方法上、构造方法参数上
@Target({ElementType.TYPE, ElementType.FIELD, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Repeatable(Resources.class)
public @interface Resource {
    String name() default "";

    String lookup() default "";

    Class<?> type() default Object.class;

    AuthenticationType authenticationType() default Resource.AuthenticationType.CONTAINER;

    boolean shareable() default true;

    String mappedName() default "";

    String description() default "";

    public static enum AuthenticationType {
        CONTAINER,
        APPLICATION;

        private AuthenticationType() {
        }
    }
}

@Resource注解属于JDK扩展包，所以不在JDK当中，需要额外引入以下依赖
    引入扩展依赖
    如果是JDK8的话不需要额外引入依赖，高于JDK11或低于JDK8需要引入以下依赖。

spring6+
<dependency>
  <groupId>jakarta.annotation</groupId>
  <artifactId>jakarta.annotation-api</artifactId>
  <version>2.1.1</version>
</dependency>

spring5-（包括spring5）
<dependency>
  <groupId>javax.annotation</groupId>
  <artifactId>javax.annotation-api</artifactId>
  <version>1.3.2</version>
</dependency>

/**
 * ClassName: StudentDao
 * Package: cw.spring.dao
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-14 16:03
 * @Version 1.0
 */
public interface StudentDao {
    void deleteById();
}

/**
 * ClassName: StudentDaoImplForMySQL
 * Package: cw.spring.dao.impl
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-14 16:04
 * @Version 1.0
 */
@Repository
public class StudentDaoImplForMySQL implements StudentDao {
    @Override
    public void deleteById() {
        System.out.println("MySQL数据库正在删除学生信息...");
    }
}

/**
 * ClassName: StudentService
 * Package: cw.spring.service
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-14 16:05
 * @Version 1.0
 */
@Service
public class StudentService {
    // name属性用于指定将要被注入到该属性的Bean的名字
    @Resource(name = "studentDaoImplForMySQL")
    private StudentDao studentDao;
    
    public void delete() {
        studentDao.deleteById();
    }
}

```

# 10. 全注解开发
```
全注解开发就是不再使用 spring 配置文件，写一个配置类来代替配置文件。

@Configuration 注解标注配置类（org.springframework.context.annotation.Configuration）
    配置类使用 @Configuration 注解进行标注
@Configuration
public class Spring6Config {
}

@ComponentScan 注解配置扫描的包
	通过 @ComponentScan 注解配置要扫描的包
	
@Configuration
@ComponentScan({"cw.spring.dao", "cw.spring.service"})
public class Spring6Config {
}

全注解开发下获取 Spring 容器
	获取 spring 容器不再 new ClassPathXmlApplicationContext() 对象了，而是 new AnnotationConfigApplicationContext()
@org.junit.Test
public void test03() {
    // 获取Spring容器对象时，需要传配置类为参数
    ApplicationContext applicationContext = new AnnotationConfigApplicationContext(Spring6Config.class);
    StudentService studentService = applicationContext.getBean("studentService", StudentService.class);
    studentService.delete();
}

```

# 11. 源码解析：注解驱动 IoC 容器全流程

## 11.1 AnnotationConfigApplicationContext 的构造链路

```
当我们执行 new AnnotationConfigApplicationContext(Spring6Config.class) 时，
内部发生了什么？

public class AnnotationConfigApplicationContext extends GenericApplicationContext
        implements AnnotationConfigRegistry {

    private final AnnotatedBeanDefinitionReader reader;
    private final ClassPathBeanDefinitionScanner scanner;

    // ★ 最常用的构造器
    public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
        // 步骤 1: 初始化 AnnotatedBeanDefinitionReader 和 ClassPathBeanDefinitionScanner
        this();

        // 步骤 2: 将传入的配置类注册为 BeanDefinition
        register(componentClasses);

        // 步骤 3: ★ 调用 AbstractApplicationContext.refresh() ★
        //        这是整个 Spring 容器初始化流程的入口
        //        内部包含了 13 个子步骤，其中最关键的是：
        //        - invokeBeanFactoryPostProcessors(): 调用 ConfigurationClassPostProcessor
        //        - registerBeanPostProcessors(): 注册 BeanPostProcessor
        //        - finishBeanFactoryInitialization(): 实例化所有单例 Bean
        refresh();
    }

    // 默认构造器
    public AnnotationConfigApplicationContext() {
        StartupStep createAnnotatedBeanDefReader =
                this.getApplicationStartup().start("spring.context.annotated-bean-reader.create");

        // ★ 创建 AnnotatedBeanDefinitionReader：
        //   负责解析 @Configuration 类中的 @Bean、@Import、@ComponentScan 等注解
        //   将解析结果注册为 BeanDefinition
        this.reader = new AnnotatedBeanDefinitionReader(this);

        createAnnotatedBeanDefReader.end();

        // ★ 创建 ClassPathBeanDefinitionScanner：
        //   负责扫描指定包路径下的所有类
        //   将带有 @Component/@Service/@Repository/@Controller 的类注册为 BeanDefinition
        this.scanner = new ClassPathBeanDefinitionScanner(this);

        // 注意：这里不执行扫描，扫描在 refresh() → ConfigurationClassPostProcessor 中触发
    }
}
```

### 11.2 AnnotatedBeanDefinitionReader 注册配置类

```
public void register(Class<?>... componentClasses) {
    for (Class<?> componentClass : componentClasses) {
        registerBean(componentClass);
    }
}

public void registerBean(Class<?> beanClass) {
    doRegisterBean(beanClass, null, null, null, null);
}

private <T> void doRegisterBean(
        Class<T> beanClass, @Nullable String name,
        @Nullable Class<? extends Annotation>[] qualifiers,
        @Nullable Supplier<T> supplier,
        @Nullable BeanDefinitionCustomizer[] customizers) {

    // ★ 1. 封装为 AnnotatedGenericBeanDefinition
    //    内部会解析 @Scope、@Lazy、@DependsOn、@Primary、@Description、@Role 等注解
    AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(beanClass);
    if (!this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
        abd.setInstanceSupplier(supplier);
        // 2. 解析 scope
        ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
        abd.setScope(scopeMetadata.getScopeName());
        // 3. 生成 beanName（如果未指定，默认为类名首字母小写）
        String beanName = (name != null ? name :
                this.beanNameGenerator.generateBeanName(abd, this.registry));
        // 4. 解析 @Lazy、@Primary、@DependsOn、@Role、@Description 等注解
        AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
        // 5. 处理 @Qualifier
        if (qualifiers != null) { /* ... */ }
        // 6. 用户自定义
        if (customizers != null) { /* ... */ }
        // ★ 7. 注册到 DefaultListableBeanFactory 的 beanDefinitionMap
        BeanDefinitionReaderUtils.registerBeanDefinition(
                new BeanDefinitionHolder(abd, beanName), this.registry);
    }
}
```

## 11.2 ConfigurationClassPostProcessor — 注解配置处理的核心

```
ConfigurationClassPostProcessor 是整个注解驱动的核心处理器！
它实现了 BeanDefinitionRegistryPostProcessor，优先级为 Ordered.HIGHEST_PRECEDENCE。

在 refresh() → invokeBeanFactoryPostProcessors() 阶段被调用：

public class ConfigurationClassPostProcessor
        implements BeanDefinitionRegistryPostProcessor, PriorityOrdered, ResourceLoaderAware,
        ApplicationStartupAware, BeanClassLoaderAware, EnvironmentAware {

    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
        int registryId = System.identityHashCode(registry);
        // ... 防止重复处理
        processConfigBeanDefinitions(registry);
    }

    public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
        List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
        String[] candidateNames = registry.getBeanDefinitionNames();

        // ★ 步骤 1: 收集所有被 @Configuration 标注的类（配置类）
        for (String beanName : candidateNames) {
            BeanDefinition beanDef = registry.getBeanDefinition(beanName);
            if (beanDef.getAttribute(
                    ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE) != null) {
                // 已处理过的跳过
            } else if (ConfigurationClassUtils.checkConfigurationClassCandidate(
                    beanDef, this.metadataReaderFactory)) {
                // ★ checkConfigurationClassCandidate 会检查是否有 @Configuration 注解
                // 如果有，则设置 CONFIGURATION_CLASS_ATTRIBUTE 为 "full" 或 "lite"
                configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
            }
        }

        if (configCandidates.isEmpty()) {
            return;
        }

        // 按 @Order 排序配置类
        configCandidates.sort((bd1, bd2) -> {
            int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
            int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
            return Integer.compare(i1, i2);
        });

        // ★ 步骤 2: 创建 ConfigurationClassParser（配置类解析器）
        ConfigurationClassParser parser = new ConfigurationClassParser(
                this.metadataReaderFactory, this.problemReporter, this.environment,
                this.resourceLoader, this.componentScanBeanNameGenerator, registry);

        Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
        Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());

        do {
            // ★ 步骤 3: 解析所有配置类
            //   解析 @ComponentScan、@Import、@Bean、@ImportResource 等注解
            //   新发现的配置类也会被加入 candidates
            StartupStep processConfig = this.applicationStartup.start(
                    "spring.context.config-classes.parse");
            parser.parse(candidates);
            parser.validate();

            // ★ 步骤 4: 读取解析结果，加载 BeanDefinition
            Set<ConfigurationClass> configClasses = new LinkedHashSet<>(
                    parser.getConfigurationClasses());
            configClasses.removeAll(alreadyParsed);

            // 创建 ConfigurationClassBeanDefinitionReader 来处理 @Bean 方法
            if (this.reader == null) {
                this.reader = new ConfigurationClassBeanDefinitionReader(
                        registry, this.sourceExtractor, this.resourceLoader,
                        this.environment, this.importBeanNameGenerator,
                        parser.getImportRegistry());
            }

            // ★ 步骤 5: 注册 @Bean 方法、Import 进来的类等
            this.reader.loadBeanDefinitions(configClasses);
            alreadyParsed.addAll(configClasses);

            processConfig.tag("classCount", () -> String.valueOf(configClasses.size())).end();

            candidates.clear();
            // 如果解析过程中新注册了 BeanDefinition，继续处理
            if (registry.getBeanDefinitionCount() > candidateNames.length) {
                // ... 继续循环
            }
        } while (!candidates.isEmpty());

        // ★ 步骤 6: 为 @Configuration full 模式的类创建 CGLIB 代理
        //   确保 @Bean 方法调用能保证单例（即使在同一类内部调用）
        // ...
    }
}
```

## 11.3 ConfigurationClassParser.parse() — 注解解析过程

```
ConfigurationClassParser 负责解析配置类上的关键注解：

protected final SourceClass doProcessConfigurationClass(
        ConfigurationClass configClass, SourceClass sourceClass, Predicate<String> filter)
        throws IOException {

    // ★ 1. 处理 @Component 和 @ComponentScan（递归扫描）
    //    如果是内部类（被 @Configuration 标注的普通 Bean），也可以扫描
    if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
        // 递归处理内部类
        processMemberClasses(configClass, sourceClass, filter);
    }

    // ★ 2. 处理 @PropertySource
    for (AnnotationAttributes propertySource :
            AnnotationConfigUtils.attributesForRepeatable(
                    sourceClass.getMetadata(), PropertySources.class,
                    org.springframework.context.annotation.PropertySource.class)) {
        if (this.propertySourceRegistry != null) {
            this.propertySourceRegistry.processPropertySource(propertySource);
        }
    }

    // ★★★ 3. ★ 处理 @ComponentScan ★★★
    Set<AnnotationAttributes> componentScans =
            AnnotationConfigUtils.attributesForRepeatable(
                    sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
    if (!componentScans.isEmpty()
            && !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(),
                    ConfigurationPhase.REGISTER_BEAN)) {
        for (AnnotationAttributes componentScan : componentScans) {
            // ★ 调用 ClassPathBeanDefinitionScanner.doScan()
            //   扫描 basePackage 下的所有 .class 文件
            //   过滤出有 @Component 注解及其派生注解的类
            //   将它们注册为 BeanDefinition
            Set<BeanDefinitionHolder> scannedBeanDefinitions =
                    this.componentScanParser.parse(componentScan,
                            sourceClass.getMetadata().getClassName());

            // 扫描到的类中也可能有 @Configuration 注解，递归处理
            for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
                BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
                if (bdCand == null) {
                    bdCand = holder.getBeanDefinition();
                }
                if (ConfigurationClassUtils.checkConfigurationClassCandidate(
                        bdCand, this.metadataReaderFactory)) {
                    parse(bdCand.getBeanClassName(), holder.getBeanName());
                }
            }
        }
    }

    // ★ 4. 处理 @Import
    //    @Import 可以导入：
    //    - 普通类 → 注册为 BeanDefinition
    //    - ImportSelector → 调用 selectImports()
    //    - ImportBeanDefinitionRegistrar → 调用 registerBeanDefinitions()
    processImports(configClass, sourceClass, getImports(sourceClass), filter, true);

    // ★ 5. 处理 @ImportResource（引入 XML 配置文件）
    AnnotationAttributes importResource =
            AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
    if (importResource != null) {
        String[] resources = importResource.getStringArray("locations");
        Class<? extends BeanDefinitionReader> readerClass =
                importResource.getClass("reader");
        for (String resource : resources) {
            configClass.addImportedResource(resource, readerClass);
        }
    }

    // ★ 6. 处理 @Bean 方法
    Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
    for (MethodMetadata methodMetadata : beanMethods) {
        // 将 @Bean 方法封装为 BeanMethod 对象，稍后由 reader 注册
        configClass.addBeanMethod(
                new BeanMethod(methodMetadata, configClass));
    }

    // 7. 处理接口默认方法上的 @Bean
    processInterfaces(configClass, sourceClass);

    // 8. 如果有父类，递归处理
    if (sourceClass.getMetadata().hasSuperClass()) {
        String superclass = sourceClass.getMetadata().getSuperClassName();
        if (superclass != null && !superclass.startsWith("java")
                && !this.knownSuperclasses.containsKey(superclass)) {
            this.knownSuperclasses.put(superclass, configClass);
            return sourceClass.getSuperClass();
        }
    }

    return null;  // 处理完毕
}
```

## 11.4 ClassPathBeanDefinitionScanner — 组件扫描源码

```
ComponentScanAnnotationParser 调用 ClassPathBeanDefinitionScanner.doScan()：

protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
    Assert.notEmpty(basePackages, "At least one base package must be specified");
    Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
    for (String basePackage : basePackages) {
        // ★ 1. 扫描指定包，查找所有候选组件（带有 @Component 注解的类）
        Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
        for (BeanDefinition candidate : candidates) {
            // 2. 解析 @Scope
            ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
            candidate.setScope(scopeMetadata.getScopeName());
            // 3. 生成 beanName
            String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
            // 4. 处理 @Lazy、@Primary、@DependsOn 等
            if (candidate instanceof AbstractBeanDefinition abd) {
                postProcessBeanDefinition(abd, beanName);
            }
            if (candidate instanceof AnnotatedBeanDefinition abd) {
                AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
            }
            // 5. 检查是否与已注册的 BeanDefinition 冲突
            if (checkCandidate(beanName, candidate)) {
                BeanDefinitionHolder definitionHolder =
                        new BeanDefinitionHolder(candidate, beanName);
                // 6. 注册到容器
                registerBeanDefinition(definitionHolder, this.registry);
                beanDefinitions.add(definitionHolder);
            }
        }
    }
    return beanDefinitions;
}

// findCandidateComponents() 的核心实现：
// 通过 ClassPathScanningCandidateComponentProvider.scanCandidateComponents()
// 1. 将包路径转换为文件系统路径（如 com.example → com/example/）
// 2. 递归扫描该路径下所有 .class 文件
// 3. 对每个 .class 文件，使用 ASM（MetadataReader）读取类元数据（不加载类！）
// 4. 检查类上是否有 @Component 注解（通过 includeFilter）
// 5. 检查类是否被 @Conditional 条件过滤掉
// 6. 通过 excludeFilter 排除不符合条件的类
// 7. 返回 ScannedGenericBeanDefinition 集合
```

## 11.5 AutowiredAnnotationBeanPostProcessor — @Autowired 注入源码

```
AutowiredAnnotationBeanPostProcessor 是注解驱动 DI 的核心 BeanPostProcessor：

public class AutowiredAnnotationBeanPostProcessor
        implements SmartInstantiationAwareBeanPostProcessor, MergedBeanDefinitionPostProcessor,
        PriorityOrdered, BeanFactoryAware {

    // ★ 第一阶段：postProcessMergedBeanDefinition()
    //   在 doCreateBean() → applyMergedBeanDefinitionPostProcessors() 中调用
    //   解析类中所有需要注入的字段和方法
    @Override
    public void postProcessMergedBeanDefinition(
            RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {

        // ★ 查找所有 @Autowired、@Value、@Inject 标注的字段和方法
        InjectionMetadata metadata = findAutowiringMetadata(beanName, beanType, null);
        metadata.checkConfigMembers(beanDefinition);
    }

    // 查找注入元数据
    private InjectionMetadata findAutowiringMetadata(
            String beanName, Class<?> clazz, @Nullable PropertyValues pvs) {

        // 缓存键
        String cacheKey = (StringUtils.hasLength(beanName) ? beanName : clazz.getName());
        InjectionMetadata metadata = this.injectionMetadataCache.get(cacheKey);

        if (InjectionMetadata.needsRefresh(metadata, clazz)) {
            synchronized (this.injectionMetadataCache) {
                metadata = this.injectionMetadataCache.get(cacheKey);
                if (InjectionMetadata.needsRefresh(metadata, clazz)) {
                    metadata = buildAutowiringMetadata(clazz);
                    this.injectionMetadataCache.put(cacheKey, metadata);
                }
            }
        }
        return metadata;
    }

    // ★ 构建注入元数据：扫描类中的所有字段和方法
    private InjectionMetadata buildAutowiringMetadata(Class<?> clazz) {
        List<InjectionMetadata.InjectedElement> elements = new ArrayList<>();
        Class<?> targetClass = clazz;

        do {
            final List<InjectionMetadata.InjectedElement> currElements = new ArrayList<>();

            // ★ 扫描所有字段（包括 private 字段）
            ReflectionUtils.doWithLocalFields(targetClass, field -> {
                // 检查字段上是否有 @Autowired、@Value、@Inject 注解
                MergedAnnotation<?> ann = findAutowiredAnnotation(field);
                if (ann != null) {
                    // 静态字段不支持注入
                    if (Modifier.isStatic(field.getModifiers())) {
                        return;
                    }
                    boolean required = determineRequiredStatus(ann);
                    currElements.add(new AutowiredFieldElement(field, required));
                }
            });

            // ★ 扫描所有方法（包括 private 方法）
            ReflectionUtils.doWithLocalMethods(targetClass, method -> {
                Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
                if (!BridgeMethodResolver.isVisibilityBridgeMethodPair(method, bridgedMethod)) {
                    return;
                }
                MergedAnnotation<?> ann = findAutowiredAnnotation(bridgedMethod);
                if (ann != null && method.equals(ClassUtils.getMostSpecificMethod(method, clazz))) {
                    if (Modifier.isStatic(method.getModifiers())) {
                        return;
                    }
                    if (method.getParameterCount() == 0) {
                        // 不是 setter 方法
                    }
                    boolean required = determineRequiredStatus(ann);
                    // 查找参数的 @Value 注解
                    PropertyDescriptor pd = BeanUtils.findPropertyForMethod(
                            bridgedMethod, clazz);
                    currElements.add(new AutowiredMethodElement(method, required, pd));
                }
            });

            elements.addAll(0, currElements);
            targetClass = targetClass.getSuperclass(); // 处理父类
        }
        while (targetClass != null && targetClass != Object.class);

        return InjectionMetadata.forElements(elements, clazz);
    }

    // ★ 第二阶段：postProcessProperties()（实际注入）
    //   在 doCreateBean() → populateBean() 中调用
    @Override
    public PropertyValues postProcessProperties(
            PropertyValues pvs, Object bean, String beanName) {

        // 获取之前缓存的注入元数据
        InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
        try {
            // ★★★ 执行注入 ★★★
            metadata.inject(bean, beanName, pvs);
        } catch (BeanCreationException ex) {
            throw ex;
        } catch (Throwable ex) {
            throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
        }
        return pvs;
    }
}
```

### 11.6 @Autowired 注入的实际执行：AutowiredFieldElement.inject()

```
// AutowiredAnnotationBeanPostProcessor 内部类 AutowiredFieldElement
private class AutowiredFieldElement extends InjectionMetadata.InjectedElement {

    @Override
    protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs)
            throws Throwable {

        Field field = (Field) this.member;
        Object value;

        // 如果字段被缓存过（通常是在原型 Bean 的第二次创建时）
        if (this.cached) {
            try {
                // ★ 从缓存的解析结果中获取依赖
                value = resolvedCachedArgument(beanName, this.cachedFieldValue);
            } catch (NoSuchBeanDefinitionException ex) {
                throw new UnsatisfiedDependencyException(
                        getBeanDescription(field), beanName, field, ex);
            }
        } else {
            // ★ 构建 DependencyDescriptor（描述需要什么类型的 Bean）
            DependencyDescriptor desc = new DependencyDescriptor(field, this.required);
            desc.setContainingClass(bean.getClass());
            Set<String> autowiredBeanNames = new LinkedHashSet<>(1);
            // 类型转换器
            TypeConverter typeConverter = beanFactory.getTypeConverter();

            try {
                // ★★★ 核心：通过 BeanFactory 解析依赖 ★★★
                //   这里会调用 DefaultListableBeanFactory.resolveDependency()
                //   根据类型、@Qualifier 注解、泛型信息等找到合适的 Bean
                value = beanFactory.resolveDependency(desc, beanName,
                        autowiredBeanNames, typeConverter);
            } catch (BeansException ex) {
                throw new UnsatisfiedDependencyException(null, beanName,
                        new InjectionPoint(field), ex);
            }

            // 缓存解析结果（优化下次创建）
            synchronized (this) {
                if (!this.cached) {
                    Object cachedFieldValue = null;
                    if (value != null || this.required) {
                        cachedFieldValue = desc;
                        // 注册依赖关系（用于销毁顺序）
                        registerDependentBeans(beanName, autowiredBeanNames);
                        if (autowiredBeanNames.size() == 1) {
                            String autowiredBeanName = autowiredBeanNames.iterator().next();
                            if (beanFactory.containsBean(autowiredBeanName)
                                    && beanFactory.isTypeMatch(autowiredBeanName, field.getType())) {
                                cachedFieldValue = new ShortcutDependencyDescriptor(
                                        desc, autowiredBeanName, field.getType());
                            }
                        }
                    }
                    this.cachedFieldValue = cachedFieldValue;
                    this.cached = true;
                }
            }
        }

        // ★ 通过反射设置字段值
        if (value != null) {
            ReflectionUtils.makeAccessible(field);
            field.set(bean, value);
        }
    }
}
```

## 11.6 @Configuration 的 CGLIB 代理机制

```
问题：为什么 @Configuration 类需要 CGLIB 代理？

@Configuration
public class AppConfig {
    @Bean
    public ServiceA serviceA() {
        return new ServiceA(serviceB());  // ← 调用 serviceB() 会创建新实例吗？
    }

    @Bean
    public ServiceB serviceB() {
        return new ServiceB();
    }
}

如果没有 CGLIB 代理：
  serviceA() 中调用 serviceB() 会直接执行 Java 方法，每次调用都 new 一个新 ServiceB
  → serviceB 被创建了 2 次（一次是 @Bean 注册，一次是 serviceA 中调用）→ 违反单例！

有了 CGLIB 代理后：
  Spring 为 @Configuration 类创建 CGLIB 子类
  所有 @Bean 方法的调用都会被拦截：
    如果容器中已有该 Bean → 直接从容器返回（保证单例）
    如果容器中没有 → 调用原始方法创建，并将结果存入容器

实现：ConfigurationClassEnhancer.BeanMethodInterceptor

private static class BeanMethodInterceptor implements MethodInterceptor, ConditionEvaluator {

    @Override
    @Nullable
    public Object intercept(Object enhancedConfigInstance, Method beanMethod,
            Object[] beanMethodArgs, MethodProxy cglibMethodProxy) throws Throwable {

        // 1. 根据方法信息，检查容器中是否已有该 Bean
        String beanName = (String) ...; // 从 beanMethod 的名称推断
        Scope scope = (Scope) ...;

        if (isCurrentlyInvokedFactoryMethod(beanMethod)) {
            // 如果当前正在被 Spring 调用（而不是用户代码通过 this.xxx() 调用）
            // 则执行原始方法
            return cglibMethodProxy.invokeSuper(enhancedConfigInstance, beanMethodArgs);
        }

        // ★ 用户代码中通过 this.serviceB() 调用时：
        //   从 BeanFactory 中获取已经注册的 Bean（保证单例）
        return resolveBeanReference(beanMethod, beanMethodArgs,
                configBeanFactory, beanName);
    }

    private Object resolveBeanReference(Method beanMethod, Object[] beanMethodArgs,
            ConfigurableBeanFactory beanFactory, String beanName) {

        // ★ 调用 BeanFactory.getBean(beanName) 从容器中获取
        //   如果容器中还没有，会触发该 Bean 的创建（确保只创建一次）
        return beanFactory.getBean(beanName);
    }
}

判定逻辑：
"full" 模式的 @Configuration 类 → 创建 CGLIB 代理 → @Bean 方法拦截
"lite" 模式的 @Configuration 类（没有 @Configuration，只有 @Component + @Bean）
  → 不创建 CGLIB 代理 → 同一类内调用 @Bean 方法 = 直接调用 Java 方法 → 可能出现多例
```

### 11.7 Full vs Lite 模式的判定

```
ConfigurationClassUtils.checkConfigurationClassCandidate() 中：

public static boolean checkConfigurationClassCandidate(
        BeanDefinition beanDef, MetadataReaderFactory metadataReaderFactory) {

    // ...

    if (isFullConfigurationCandidate(metadata)) {
        // ★ Full 模式：@Configuration 标注的类
        beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_FULL);
    } else if (isLiteConfigurationCandidate(metadata)) {
        // ★ Lite 模式：@Component + @Bean 等（没有 @Configuration）
        beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_LITE);
    } else {
        return false;
    }
    return true;
}

// Full 模式判定条件：类上有 @Configuration 注解
public static boolean isFullConfigurationCandidate(AnnotationMetadata metadata) {
    return metadata.isAnnotated(Configuration.class.getName());
}

// Lite 模式判定条件（满足任一即可）：
// - @Component + @Bean
// - @ComponentScan
// - @Import
// - @ImportResource
// - 方法上有 @Bean（即使类没有 @Configuration）
```

## 11.7 注解驱动容器启动完整流程图

```
new AnnotationConfigApplicationContext(Spring6Config.class)
    │
    ├── 1. this() — 创建 AnnotatedBeanDefinitionReader + ClassPathBeanDefinitionScanner
    │
    ├── 2. register(Spring6Config.class)
    │        └→ AnnotatedBeanDefinitionReader.doRegisterBean()
    │             └→ 将 Spring6Config 封装为 AnnotatedGenericBeanDefinition
    │                 注册到 DefaultListableBeanFactory.beanDefinitionMap
    │
    └── 3. refresh() — 容器刷新（13 步）
          │
          ├── 3.1 prepareRefresh()
          │
          ├── 3.2 obtainFreshBeanFactory() — 获取 BeanFactory（GenericApplicationContext 内建）
          │
          ├── 3.5 ★ invokeBeanFactoryPostProcessors()
          │      │
          │      ├── 调用 ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry()
          │      │     │
          │      │     ├── 找到所有 @Configuration 类
          │      │     ├── 创建 ConfigurationClassParser
          │      │     ├── parser.parse()
          │      │     │    ├── 解析 @ComponentScan → ClassPathBeanDefinitionScanner.doScan()
          │      │     │    │    └→ 扫描包路径 → 注册 @Component 等 Bean
          │      │     │    ├── 解析 @Import → 处理 ImportSelector / ImportBeanDefinitionRegistrar
          │      │     │    ├── 解析 @Bean → 收集 @Bean 方法
          │      │     │    └── 解析 @PropertySource、@ImportResource 等
          │      │     │
          │      │     └── ConfigurationClassBeanDefinitionReader.loadBeanDefinitions()
          │      │          └→ 将 @Bean 方法注册为 BeanDefinition
          │      │
          │      └── 调用其他 BeanFactoryPostProcessor（如 PropertySourcesPlaceholderConfigurer）
          │
          ├── 3.6 ★ registerBeanPostProcessors()
          │      └→ 实例化并注册所有 BeanPostProcessor
          │           - AutowiredAnnotationBeanPostProcessor ★
          │           - CommonAnnotationBeanPostProcessor
          │           - AnnotationAwareAspectJAutoProxyCreator
          │           - 等等...
          │
          ├── 3.11 ★ finishBeanFactoryInitialization()
          │      └→ preInstantiateSingletons()
          │           └→ 为每个单例 Bean 调用 getBean()
          │                └→ doCreateBean()
          │                     ├── createBeanInstance() — 构造方法实例化
          │                     ├── applyMergedBeanDefinitionPostProcessors()
          │                     │    └→ AutowiredAnnotationBeanPostProcessor
          │                     │         解析 @Autowired/@Value 注入点
          │                     ├── populateBean()
          │                     │    └→ AutowiredAnnotationBeanPostProcessor
          │                     │         postProcessProperties() → 实际注入
          │                     └── initializeBean()
          │                          ├── @PostConstruct (CommonAnnotationBeanPostProcessor)
          │                          ├── InitializingBean.afterPropertiesSet()
          │                          ├── init-method
          │                          └── AOP 代理创建 (AnnotationAwareAspectJAutoProxyCreator)
          │                                   └→ 如有匹配的切面，创建代理对象
          │
          └── 3.12 finishRefresh() — 发布 ContextRefreshedEvent

容器启动完成，所有单例 Bean 就绪！
```

## 11.8 设计思想总结

```
1. BeanFactoryPostProcessor 扩展点：
   ConfigurationClassPostProcessor 是 Spring 最核心的 BFPP，它利用了 BFPP 可以修改
   BeanDefinition 的特性，在 Bean 创建之前完成：
   - @ComponentScan → 批量注册 BeanDefinition
   - @Bean → 将方法注册为 BeanDefinition
   - @Import → 引入外部配置

2. BeanPostProcessor 扩展点：
   AutowiredAnnotationBeanPostProcessor 利用了 BPP 的三个回调时机：
   - postProcessMergedBeanDefinition: 提前解析注入元数据（缓存）
   - postProcessProperties: 执行实际注入
   - 支持 @Autowired、@Value、@Inject 三种注解

3. CGLIB 代理 vs Java 动态代理：
   @Configuration Full 模式使用 CGLIB 代理来确保 @Bean 方法的单例语义
   这与 AOP 代理使用相同的底层技术（CGLIB/ObjenesisCglibAopProxy）

4. 责任分离：
   注解"声明"（@Component、@Autowired）与注解"处理"（各种 Processor）完全分离
   这种设计使得：
   - 新增一个注解不需要修改解析器
   - 每个 Processor 专注于单一的注解处理职责
```
