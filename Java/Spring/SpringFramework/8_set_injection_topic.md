# 1. 注入外部Bean
```
外部Bean：要进行注入的bean定义到需要被注入的Bean的外面
对于外部Bean，在property标签中使用ref属性进行注入，或者使用ref标签进行注入，其中通过ref属性进行注入是常用。
<!--声明/定义Bean-->
<bean id="orderDaoBean" class="com.powernode.spring6.dao.OrderDao"></bean>

<bean id="orderServiceBean" class="com.powernode.spring6.service.OrderService">
    <!--使用ref属性来引入外部Bean，这就是注入外部Bean-->
    <property name="orderDao" ref="orderDaoBean"/>
</bean>

```
# 2. 注入内部Bean
```
内部Bean：在bean标签中嵌套bean标签，即需要进行注入的Bean通过Bean标签声明在需要被注入的Bean中，然后直接注入到需要被注入的Bean中。
<bean id="orderServiceBean2" class="com.powernode.spring6.service.OrderService">
    <property name="orderDao">
        <!--在property标签中使用嵌套的bean标签，这就是内部Bean-->
        <bean class="com.powernode.spring6.dao.OrderDao"></bean>
    </property>
</bean>

```
# 3. 注入简单类型
```
如果给简单类型赋值，即给属性注入简单类型，使用value属性或value标签，而不是ref。
public class User {
    private String username; // String是简单类型
    private String password;
    private int age; // int是简单类型

    public void setUsername(String username) {
        this.username = username;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "User{" +
                "username='" + username + '\'' +
                ", password='" + password + '\'' +
                ", age=" + age +
                '}';
    }
}

<!--注入简单类型-->
<bean id="userBean" class="com.powernode.spring6.bean.User">
    <!--重点：如果是给简单类型赋值，就不能使用ref了。就需要使用value了。-->
    <property name="username" value="张三"/>
    <property name="password" value="123"/>
    <property name="age" value="20"/>
</bean>

通过BeanUtils类的源码分析得知，简单类型包括：
基本数据类型
基本数据类型对应的包装类
String或其他的CharSequence子类
Number子类
Date子类，java.util.Date是简单类型
Enum子类
URI
URL
Temporal子类，Temporal是Java8提供的时间和时区类型
Locale，Locale是语言类，也是简单类型。
Class
另外还包括以上简单值类型对应的数组类型。

public class BeanUtils{
    
    //.......
    
    /**
	 * Check if the given type represents a "simple" property: a simple value
	 * type or an array of simple value types.
	 * <p>See {@link #isSimpleValueType(Class)} for the definition of <em>simple
	 * value type</em>.
	 * <p>Used to determine properties to check for a "simple" dependency-check.
	 * @param type the type to check
	 * @return whether the given type represents a "simple" property
	 * @see org.springframework.beans.factory.support.RootBeanDefinition#DEPENDENCY_CHECK_SIMPLE
	 * @see org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#checkDependencies
	 * @see #isSimpleValueType(Class)
	 */
	public static boolean isSimpleProperty(Class<?> type) {
		Assert.notNull(type, "'type' must not be null");
		return isSimpleValueType(type) || (type.isArray() && isSimpleValueType(type.getComponentType()));
	}

	/**
	 * Check if the given type represents a "simple" value type: a primitive or
	 * primitive wrapper, an enum, a String or other CharSequence, a Number, a
	 * Date, a Temporal, a URI, a URL, a Locale, or a Class.
	 * <p>{@code Void} and {@code void} are not considered simple value types.
	 * @param type the type to check
	 * @return whether the given type represents a "simple" value type
	 * @see #isSimpleProperty(Class)
	 */
	public static boolean isSimpleValueType(Class<?> type) {
		return (Void.class != type && void.class != type &&
				(ClassUtils.isPrimitiveOrWrapper(type) ||
				Enum.class.isAssignableFrom(type) ||
				CharSequence.class.isAssignableFrom(type) ||
				Number.class.isAssignableFrom(type) ||
				Date.class.isAssignableFrom(type) ||
				Temporal.class.isAssignableFrom(type) ||
				URI.class == type ||
				URL.class == type ||
				Locale.class == type ||
				Class.class == type));
	}
    
    //........
}
（这是旧的Spring6.0.0M2，isSimpleValueType在BeanUtils中；Spring6.2.11中，该方法移植到了package org.springframework.util;的public abstract class ClassUtils当中，变成了
	public static boolean isSimpleValueType(Class<?> type) {
		return (!isVoidType(type) &&
				(isPrimitiveOrWrapper(type) ||
				Enum.class.isAssignableFrom(type) ||
				CharSequence.class.isAssignableFrom(type) ||
				Number.class.isAssignableFrom(type) ||
				Date.class.isAssignableFrom(type) ||
				Temporal.class.isAssignableFrom(type) ||
				ZoneId.class.isAssignableFrom(type) ||
				TimeZone.class.isAssignableFrom(type) ||
				File.class.isAssignableFrom(type) ||
				Path.class.isAssignableFrom(type) ||
				Charset.class.isAssignableFrom(type) ||
				Currency.class.isAssignableFrom(type) ||
				InetAddress.class.isAssignableFrom(type) ||
				URI.class == type ||
				URL.class == type ||
				UUID.class == type ||
				Locale.class == type ||
				Pattern.class == type ||
				Class.class == type));
	}）

测试所有的简单类型
编写一个程序，把所有的简单类型全部测试一遍：
package com.powernode.spring6.beans;

import java.net.URI;
import java.net.URL;
import java.util.Date;
import java.util.Locale;

public class A {
    private byte b;
    private short s;
    private int i;
    private long l;
    private float f;
    private double d;
    private boolean flag;
    private char c;

    private Byte b1;
    private Short s1;
    private Integer i1;
    private Long l1;
    private Float f1;
    private Double d1;
    private Boolean flag1;
    private Character c1;

    private String str;

    private Date date;

    private Season season;

    private URI uri;

    private URL url;

    private Locale locale;

    private Class clazz;
    
    // 生成setter方法
    // 生成toString方法
}

enum Season {
    SPRING, SUMMER, AUTUMN, WINTER
}

需要注意的是：
如果把Date当做简单类型的话，日期字符串格式不能随便写，格式必须符合Date的toString()方法格式。显然这就比较鸡肋了。如果我们提供一个这样的日期字符串：2010-10-11，在这里是无法赋值给Date类型的属性的。正确格式是Wed Dec 24 12:04:42 CST 2025
Temporal 是 **Java 8 日期时间 API（java.time 包）**中的一个接口，用来表示可操作的日期/时间对象的通用类型。它是 Java 时间体系的核心接口之一，定义了一些对日期时间对象通用的操作方法。
常见实现类有：
接口/类	描述
LocalDate	仅日期（年月日）
LocalTime	仅时间（时分秒纳秒）
LocalDateTime	日期+时间（不带时区）
ZonedDateTime	日期+时间+时区
OffsetDateTime	日期+时间+时区偏移
Instant	时间线上的一个瞬时点（UTC）

spring6之后，当注入的是URL，那么这个url字符串是会进行有效性检测的。如果是一个存在的url，那就没问题。如果不存在则报错。

注入简单类型经典案例
经典案例：给数据源的属性注入值：
假设我们现在要自己手写一个数据源，我们都知道所有的数据源都要实现javax.sql.DataSource接口，并且数据源中应该有连接数据库的信息，例如：driver、url、username、password 等。
数据源：能够给你提供Connection对象的，都是数据源
/**
 * 所有的数据源都要实现java规范：javax.sql.DataSource
 **/
public class MyDataSource implements DataSource { // 可以把数据源交给Spring容器来管理。

    private String driver;
    private String url;
    private String username;
    private String password;

    public void setDriver(String driver) {
        this.driver = driver;
    }

    public void setUrl(String url) {
        this.url = url;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    @Override
    public String toString() {
        return "MyDataSource{" +
                "driver='" + driver + '\'' +
                ", url='" + url + '\'' +
                ", username='" + username + '\'' +
                ", password='" + password + '\'' +
                '}';
    }

    @Override
    public Connection getConnection() throws SQLException {
        // 获取数据库连接对象的时候需要4个信息：driver url username password
        return null;
    }

    @Override
    public Connection getConnection(String username, String password) throws SQLException {
        return null;
    }

    @Override
    public PrintWriter getLogWriter() throws SQLException {
        return null;
    }

    @Override
    public void setLogWriter(PrintWriter out) throws SQLException {

    }

    @Override
    public void setLoginTimeout(int seconds) throws SQLException {

    }

    @Override
    public int getLoginTimeout() throws SQLException {
        return 0;
    }

    @Override
    public Logger getParentLogger() throws SQLFeatureNotSupportedException {
        return null;
    }

    @Override
    public <T> T unwrap(Class<T> iface) throws SQLException {
        return null;
    }

    @Override
    public boolean isWrapperFor(Class<?> iface) throws SQLException {
        return false;
    }
}

<!--让spring来管理我们的数据源-->
<bean id="myDataSource" class="com.powernode.spring6.jdbc.MyDataSource">
    <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
    <property name="url" value="jdbc:mysql://localhost:3306/spring6"/>
    <property name="username" value="root"/>
    <property name="password" value="123456"/>
</bean>

@Test
public void testDataSource(){
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring-datasource.xml");
    MyDataSource dataSource = applicationContext.getBean("dataSource", MyDataSource.class);
    System.out.println(dataSource);
}

```
# 4. 级联属性赋值
```
// 班级类
public class Clazz {
    // 班级名称
    private String name;

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "Clazz{" +
                "name='" + name + '\'' +
                '}';
    }
}

public class Student {
    private String name;

    // 学生属于哪个班级
    private Clazz clazz;

    public void setClazz(Clazz clazz) {
        this.clazz = clazz;
    }

    // 使用级联属性赋值，这个需要这个get方法。
    // 因为级联给学生所在的班级属性赋值会调用学生的getClazz()方法
    public Clazz getClazz() {
        return clazz;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "Student{" +
                "name='" + name + '\'' +
                ", clazz=" + clazz +
                '}';
    }
}

原先的注入方法：
<bean id="studentBean" class="com.powernode.spring6.bean.Student">
    <!--简单类型，使用value-->
    <property name="name" value="张三"/>
    <!--这不是简单类型，使用ref-->
    <property name="clazz" ref="clazzBean"/>
</bean>

<bean id="clazzBean" class="com.powernode.spring6.bean.Clazz">
    <property name="name" value="高三一班"/>
</bean>

级联注入：
注意：级联属性赋值，对于对其他对象引用的成员变量，一定要提供get方法，因为进行级联属性赋值时，需要先调用get方法获取对其他对象引用的成员变量，然后再调用该成员变量属性的set方法进行属性赋值
<!--使用级联属性赋值需要注意两点：
        1. 配置的顺序不能颠倒，必须如下顺序。
        2. clazz属性必须提供getter方法。
-->
<bean id="studentBean" class="com.powernode.spring6.bean.Student">
    <!--简单类型，使用value-->
    <property name="name" value="张三"/>
    <!--这不是简单类型，使用ref-->
		<!-- 因为后面要对该对象的属性赋值，所以必须先有该对象 -->
    <property name="clazz" ref="clazzBean"/>
    <!--级联属性赋值-->
    <!-- name="clazz.name" 会调用学生的getClazz()方法，
  			所以学生类必须提供getClazz()方法，然后调用set方法给班级的name赋值 
  	-->
    <property name="clazz.name" value="高三二班"/>
</bean>

<!-- 这里不进行注入 -->
<bean id="clazzBean" class="com.powernode.spring6.bean.Clazz"></bean>

```
# 5. 注入数组

## 5.1 数组中的元素是简单类型
```
public class QianDaYe {
    private String[] aiHaos;

    public void setAiHaos(String[] aiHaos) {
        this.aiHaos = aiHaos;
    }

    @Override
    public String toString() {
        return "QianDaYe{" +
                "aiHaos=" + Arrays.toString(aiHaos) +
                '}';
    }
}
<bean id="yuQian" class="com.powernode.spring6.bean.QianDaYe">

    <!-- 这个数组属性当中的元素类型是String，是简单类型 -->
    <property name="aiHaos">
        <array>
            <value>抽烟</value>
            <value>喝酒</value>
            <value>烫头</value>
        </array>
    </property>

</bean>
```
## 5.2 数组中的元素是非简单类型
```
public class Woman {
    private String name;

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "Woman{" +
                "name='" + name + '\'' +
                '}';
    }
}
public class QianDaYe {
    private String[] aiHaos;

    // 多个女性朋友
    private Woman[] womens;

    public void setWomens(Woman[] womens) {
        this.womens = womens;
    }

    public void setAiHaos(String[] aiHaos) {
        this.aiHaos = aiHaos;
    }

    @Override
    public String toString() {
        return "QianDaYe{" +
                "aiHaos=" + Arrays.toString(aiHaos) +
                ", womens=" + Arrays.toString(womens) +
                '}';
    }
}

<bean id="w1" class="com.powernode.spring6.bean.Woman">
    <property name="name" value="小花"/>
</bean>

<bean id="w2" class="com.powernode.spring6.bean.Woman">
    <property name="name" value="小亮"/>
</bean>

<bean id="w3" class="com.powernode.spring6.bean.Woman">
    <property name="name" value="小明"/>
</bean>

<bean id="yuQian" class="com.powernode.spring6.bean.QianDaYe">

    <!-- 这个数组属性当中的元素类型是String简单类型 -->
    <property name="aiHaos">
        <array>
            <value>抽烟</value>
            <value>喝酒</value>
            <value>烫头</value>
        </array>
    </property>

    <!-- 这个数组当中的类型就不是简单类型了-->
    <property name="womens">
        <array>
            <ref bean="w1"/>
            <ref bean="w2"/>
            <ref bean="w3"/>
        </array>
    </property>

</bean>
```
## 5.3 注入数组小结
如果数组中是简单类型，使用value标签。
如果数组中是非简单类型，使用ref标签。
# 6. 注入List集合与Set集合
```
List集合：有序可重复
注意：注入List集合的时候使用list标签，如果List集合中是简单类型使用value标签，反之使用ref标签。
Set集合：无序不可重复
要点： 注入set集合的时候使用<set>标签，如果set集合中元素是简单类型的使用value标签，反之使用ref标签。

public class Person {

    // 注入List集合
    private List<String> names;

    // 注入Set集合
    private Set<String> addrs;

    public void setNames(List<String> names) {
        this.names = names;
    }

    public void setAddrs(Set<String> addrs) {
        this.addrs = addrs;
    }

    @Override
    public String toString() {
        return "Person{" +
                "names=" + names +
                ", addrs=" + addrs +
                '}';
    }
}

<bean id="personBean" class="com.powernode.spring6.bean.Person">

    <property name="names">
        <!--list集合有序可重复-->
        <list>
            <value>张三</value>
            <value>李四</value>
            <value>王五</value>
            <value>张三</value>
            <value>张三</value>
            <value>张三</value>
            <value>张三</value>
        </list>
    </property>

    <property name="addrs">
        <!--set集合无序不可重复-->
        <set>
            <value>北京大兴区</value>
            <value>北京大兴区</value>
            <value>北京海淀区</value>
            <value>北京海淀区</value>
            <value>北京大兴区</value>
        </set>
    </property>
</bean>

```
# 7. 注入Map集合与注入Porperties
```
Map集合：
使用<map>标签
如果key是简单类型，使用 key 属性，反之使用 key-ref 属性。
如果value是简单类型，使用 value 属性，反之使用 value-ref 属性。
java.util.Properties继承java.util.Hashtable，所以Properties也是一个Map集合。
Properties使用<props>标签嵌套<prop>标签完成。
在Properties中，key和value的类型都是String类型

public class Person {

    // 注入Map集合
    // 多个电话
    private Map<Integer, String> phones;

    // 注入属性类对象
    // Properties本质上也是一个Map集合。
    // Properties的父类Hashtable，Hashtable实现了Map接口。
    // 虽然这个也是一个Map集合，但是和Map的注入方式有点像，但是不同。
    // Properties的key和value只能是String类型。
    private Properties properties;

    public void setProperties(Properties properties) {
        this.properties = properties;
    }

    public void setPhones(Map<Integer, String> phones) {
        this.phones = phones;
    }

    @Override
    public String toString() {
        return "Person{" +
                "phones=" + phones +
                ", properties=" + properties +
                '}';
    }
}

<bean id="personBean" class="com.powernode.spring6.bean.Person">
    
    <property name="properties">
        <!--注入Properties属性类对象-->
        <props>
            <prop key="driver">com.mysql.cj.jdbc.Driver</prop>
            <prop key="url">jdbc:mysql://localhost:3306/spring6</prop>
            <prop key="username">root</prop>
            <prop key="password">123456</prop>
        </props>
    </property>

    <property name="phones">
        <!--注入Map集合-->
        <map>
            <!--如果key和value不是简单类型就用这个配置。-->
            <!--<entry key-ref="" value-ref=""/>-->
            <!--如果是简单类型就是key和value-->
            <entry key="1" value="110"/>
            <entry key="2" value="120"/>
            <entry key="3" value="119"/>
        </map>
    </property>
</bean>

```
# 8. 注入null和空字符串
```
注入空字符串使用：<value/> 或者 value=""
注入null使用：<null/> 或者不为该属性赋值
public class Cat {
    private String name;
    private int age;

    public void setName(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "Cat{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}

<bean id="catBean" class="com.powernode.spring6.bean.Cat">
    <!--不给属性注入，属性的默认值就是null-->（即没有任何property的name属性是这个属性名，就会自动被赋值为null）
    <!-- 这不是注入null，这只是注入了一个"null"字符串-->
    <!--<property name="name" value="null"/>-->
    <!--这种方式是手动注入null-->
    <!--<property name="name">
        <null/>
    </property>-->

    <!--注入空字符串第一种方式-->
    <!--<property name="name" value=""/>-->
    <!--注入空字符串第二种方式-->
    <property name="name">
        <value/>（又或者是<value></value>，单标签一定可以替换为双标签，双标签有时候可以替换为单标签，当双标签不需要嵌套其他内容时可以替换为单标签）
    </property>

    <property name="age" value="3"></property>
</bean>

```
# 9. 注入的值中含有特殊符号
```
XML中有5个特殊字符，分别是：<、>、'、"、&
以上5个特殊符号在XML中会被特殊对待，会被当做XML语法的一部分进行解析，如果这些特殊符号直接出现在注入的字符串当中，会报错。
解决方案包括两种：
第一种：特殊符号使用转义字符代替。
第二种：将含有特殊符号的字符串放到： <![CDATA[]]> 当中。放在CDATA区中的数据不会被XML文件解析器解析。使用CDATA时，不能使用value属性，只能使用value标签。

五个特殊字符对应的转义字符分别是
> &gt;
< &lt;
' &apos;
" &quot;
& &amp;

public class MathBean {
    private String result;

    public void setResult(String result) {
        this.result = result;
    }

    @Override
    public String toString() {
        return "MathBean{" +
                "result='" + result + '\'' +
                '}';
    }
}

<bean id="mathBean" class="com.powernode.spring6.bean.MathBean">
	<!--第一种方案：使用实体符号代替特殊符号-->
	<!--<property name="result" value="2 &lt; 3" />-->

	<!--第二种方案：使用<![CDATA[]]>-->
	<property name="result">
		<!--只能使用value标签-->
		<value><![CDATA[2 < 3]]></value>
	</property>
</bean>

```

# 10. 源码解析：属性注入的底层实现

## 10.1 populateBean() — 属性赋值入口

```
在 doCreateBean() 中，createBeanInstance() 完成实例化后，紧接着调用 populateBean()：

protected void populateBean(String beanName, RootBeanDefinition mbd,
        @Nullable BeanWrapper bw) {

    // 1. 获取所有 PropertyValues（来自 XML <property> 标签或注解注入）
    PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

    // 2. 自动装配 byName / byType（参见第12章源码解析）
    int resolvedAutowireMode = mbd.getResolvedAutowireMode();
    if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
        MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
        if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
            autowireByName(beanName, mbd, bw, newPvs);
        }
        if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
            autowireByType(beanName, mbd, bw, newPvs);
        }
        pvs = newPvs;
    }

    // 3. ★ 执行 InstantiationAwareBeanPostProcessor 的 postProcessProperties()
    //    这里 AutowiredAnnotationBeanPostProcessor 会处理 @Autowired/@Value
    boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
    boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);
    if (hasInstAwareBpps) {
        for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
            PropertyValues pvsToUse = bp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
            if (pvsToUse == null) {
                return; // 如果返回 null，说明不需要继续处理
            }
            pvs = pvsToUse;
        }
    }

    // 4. ★★★ 核心：将 PropertyValues 应用到 Bean 上 ★★★
    if (pvs != null) {
        applyPropertyValues(beanName, mbd, bw, pvs);
    }
}
```

## 10.2 applyPropertyValues() — 属性值解析与应用

```
这是真正将 XML <property> 或自动装配产生的 PropertyValue 设置到 Bean 上的方法：

protected void applyPropertyValues(String beanName, BeanDefinition mbd,
        BeanWrapper bw, PropertyValues pvs) {

    if (pvs.isEmpty()) {
        return;
    }

    // 获取类型转换器
    MutablePropertyValues mpvs = null;
    List<PropertyValue> original;
    if (pvs instanceof MutablePropertyValues m) {
        mpvs = m;
        // 如果已经进行过值转换（之前的缓存），直接设置
        if (mpvs.isConverted()) {
            bw.setPropertyValues(mpvs);
            return;
        }
        original = mpvs.getPropertyValueList();
    } else {
        original = Arrays.asList(pvs.getPropertyValues());
    }

    TypeConverter converter = getCustomTypeConverter();
    if (converter == null) {
        converter = bw; // BeanWrapper 本身就是 TypeConverter
    }

    // ★ 创建 BeanDefinitionValueResolver：负责解析 TypedStringValue → 实际值
    //    例如 "jdbc:mysql://..." → String, "20" → int 20
    BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(
            this, beanName, mbd, converter);

    // ★ 创建深拷贝列表
    List<PropertyValue> deepCopy = new ArrayList<>(original.size());
    boolean resolveNecessary = false;

    for (PropertyValue pv : original) {
        // ★ 如果值已经被解析过，直接使用
        if (pv.isConverted()) {
            deepCopy.add(pv);
        } else {
            String propertyName = pv.getName();
            Object originalValue = pv.getValue();
            // ★ 判断这个值是否需要解析
            if (originalValue == AUTOWIRE_RESOLVED_VALUE) {
                // 自动装配的结果，已在 autowire 阶段解析好
                Object autowiredValue = bw.resolveDependency(...);
                deepCopy.add(new PropertyValue(propertyName, autowiredValue));
            } else if (originalValue instanceof TypedStringValue typedStringValue) {
                // ★ XML <property value="xxx"/> 生成的就是 TypedStringValue
                Object resolvedValue = evaluateBeanDefinitionString(
                        typedStringValue.getValue(), mbd);
                deepCopy.add(new PropertyValue(propertyName, resolvedValue));
            } else if (originalValue instanceof BeanDefinitionHolder bdh) {
                // ★ 内嵌 <bean>
                Object resolvedValue = resolveInnerBean(beanName,
                        bdh.getBeanDefinition(), mbd, propertyName);
                deepCopy.add(new PropertyValue(propertyName, resolvedValue));
            } else if (originalValue instanceof RuntimeBeanReference ref) {
                // ★ XML <property ref="beanId"/> 生成 RuntimeBeanReference
                Object resolvedValue = resolveBeanReference(beanName, ref);
                deepCopy.add(new PropertyValue(propertyName, resolvedValue));
            } else if (originalValue instanceof ManagedArray || originalValue instanceof ManagedList
                    || originalValue instanceof ManagedSet || originalValue instanceof ManagedMap
                    || originalValue instanceof ManagedProperties) {
                // ★ <array>、<list>、<set>、<map>、<props> 集合类型
                Object resolvedValue = resolveManagedCollection(beanName, originalValue, mbd);
                deepCopy.add(new PropertyValue(propertyName, resolvedValue));
            } else {
                // 其他已解析的值（如注解注入的结果）
                deepCopy.add(pv);
            }
        }
    }

    // 标记为已转换
    if (mpvs != null && !resolveNecessary) {
        mpvs.setConverted();
    }

    // ★★★ 通过 BeanWrapper 将属性值设置到 Bean 上 ★★★
    try {
        bw.setPropertyValues(new MutablePropertyValues(deepCopy));
    } catch (BeansException ex) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                "Error setting property values", ex);
    }
}
```

## 10.3 BeanWrapperImpl.setPropertyValues() — 反射设置属性

```
BeanWrapperImpl 是 Spring 中 BeanWrapper 接口的默认实现，
负责通过反射将属性值设置到 Bean 上。

public class BeanWrapperImpl extends AbstractNestablePropertyAccessor
        implements BeanWrapper {

    @Override
    public void setPropertyValues(PropertyValues pvs) throws BeansException {
        setPropertyValues(pvs, false, false);
    }

    @Override
    public void setPropertyValues(PropertyValues pvs, boolean ignoreUnknown,
            boolean ignoreInvalid) throws BeansException {

        List<PropertyAccessException> propertyAccessExceptions = null;
        List<PropertyValue> propertyValues = (pvs instanceof MutablePropertyValues mpvs
                ? mpvs.getPropertyValueList() : Arrays.asList(pvs.getPropertyValues()));

        for (PropertyValue pv : propertyValues) {
            try {
                // ★ 核心：对每个 PropertyValue 调用 setPropertyValue
                setPropertyValue(pv);
            } catch (NotWritablePropertyException ex) {
                if (!ignoreUnknown) {
                    throw ex;
                }
            } catch (NullValueInNestedPathException ex) {
                if (!ignoreInvalid) {
                    throw ex;
                }
            } catch (PropertyAccessException ex) {
                if (propertyAccessExceptions == null) {
                    propertyAccessExceptions = new ArrayList<>();
                }
                propertyAccessExceptions.add(ex);
            }
        }

        if (propertyAccessExceptions != null) {
            PropertyAccessException[] paeArray =
                    propertyAccessExceptions.toArray(new PropertyAccessException[0]);
            throw new PropertyBatchUpdateException(paeArray);
        }
    }
}
```

### 10.3.1 BeanWrapperImpl.setPropertyValue() 单属性设置

```
@SuppressWarnings("unchecked")
@Override
public void setPropertyValue(PropertyValue pv) throws BeansException {
    AbstractNestablePropertyAccessor nestedPa;
    try {
        nestedPa = getPropertyAccessorForPropertyPath(pv.getName());
        // 1. 获取嵌套路径（如 clazz.name → 先获取 clazz，再在其上设置 name）
        String nestedProperty = nestedPa.getFinalPath(nestedPa.getNestedPropertyPrefix());
        
        // 2. ★ 获取属性的 PropertyHandler
        //    PropertyHandler 封装了 JavaBean 的 getter/setter 方法
        PropertyHandler ph = nestedPa.getLocalPropertyHandler(nestedProperty);
        if (ph == null) {
            throw new NotWritablePropertyException(bean.getClass(), nestedProperty);
        }

        // 3. 类型转换：将值转换为目标属性类型
        Object valueToApply = pv.getValue();
        if (pv.isConverted()) {
            // 值已经转换过了（如自动装配的结果）
        } else {
            // ★ 使用 TypeConverter 进行类型转换
            //    如 "20" → Integer 20, "张三" → String "张三"
            valueToApply = convertForProperty(nestedPa, ph, pv);
        }

        // 4. ★★★ 反射调用 setter 方法 ★★★
        ph.setValue(valueToApply);
        
    } catch (InvalidPropertyException ex) {
        throw new NotWritablePropertyException(
                getRootClass(), this.nestedPath + pv.getName(),
                "Nested property in path '" + pv.getName() + "' does not exist", ex);
    }
}

// PropertyHandler.setValue() 核心实现：
@Override
public void setValue(@Nullable Object value) throws Exception {
    // 获取 setter 方法的 Method 对象
    Method writeMethod = this.pd.getWriteMethod();
    if (writeMethod != null) {
        // ★ 反射调用 setter
        ReflectionUtils.makeAccessible(writeMethod);
        writeMethod.invoke(getWrappedInstance(), value);
    }
}
```

## 10.4 TypedStringValue 的值解析流程

```
XML 中的 <property name="age" value="20"/> 会产生一个 TypedStringValue("20")：

在 applyPropertyValues() → evaluateBeanDefinitionString() 解析：

1. TypedStringValue 内部存储了原始字符串 "20"
2. 如果有 targetTypeName（如 "int"），会通过 ClassUtils.forName() 加载类型
3. ★ 调用 TypeConverter.convertIfNecessary("20", int.class)
   → TypeConverterDelegate.convertIfNecessary()
   → 判断目标类型 → int 是基本类型
   → 使用 Integer.valueOf("20") 转换
   → 返回 Integer(20)
4. 如果 XML 还使用了 SpEL（如 value="#{...}"），会走 SpEL 解析路径

值解析支持的类型转换器链：
  String → int/long/float/double/boolean/char
  String → Integer/Long/Float/Double/Boolean/Character
  String → Enum（通过 Enum.valueOf()）
  String → Date（要求特殊格式 "Mon Oct 10 14:30:26 CST 2022"）
  String → Temporal（Java 8 时间类型）
  String → URI / URL（Spring 6 中 URL 会验证有效性）
```

## 10.5 RuntimeBeanReference 解析源码

```
XML 中的 <property name="orderDao" ref="orderDaoBean"/>
产生一个 RuntimeBeanReference("orderDaoBean")：

protected Object resolveBeanReference(String beanName, RuntimeBeanReference ref) {
    try {
        String refName = ref.getBeanName();  // "orderDaoBean"
        // ★ 调用 BeanFactory.getBean(refName) 获取引用对象
        Object bean = this.beanFactory.getBean(refName);
        // 注册依赖关系
        registerDependentBean(refName, beanName);
        return bean;
    } catch (BeansException ex) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                "Cannot resolve reference to bean '" + ref.getBeanName() + "'", ex);
    }
}
```

## 10.6 嵌套属性设置源码

```
级联赋值如 <property name="clazz.name" value="高三二班"/>：

AbstractNestablePropertyAccessor.getPropertyAccessorForPropertyPath("clazz.name"):

1. 解析 "clazz.name"：识别 . 分隔符
2. 第一段 "clazz"：
   → 调用 getLocalPropertyHandler("clazz")
   → 如果 clazz 已有值，拿到 Clazz 对象
   → 如果 clazz 为 null，自动创建 Clazz 对象（通过 getter 的返回类型）
3. 第二段 "name"：
   → 在 Clazz 对象上调用 getLocalPropertyHandler("name")
   → 最终调用 Clazz.setName("高三二班")

关键源码：
protected AbstractNestablePropertyAccessor getPropertyAccessorForPropertyPath(
        String propertyPath) {
    int pos = PropertyAccessorUtils.getFirstNestedPropertySeparatorIndex(propertyPath);
    if (pos > -1) {
        String nestedProperty = propertyPath.substring(0, pos);       // "clazz"
        String nestedPath = propertyPath.substring(pos + 1);          // "name"
        // ★ 递归获取嵌套对象
        Object nestedValue = getPropertyValue(nestedProperty); // 调用 Clazz.getClazz()
        if (nestedValue == null) {
            // 如果嵌套对象为 null，自动创建（通过 setDefaultValue）
            setDefaultValue(nestedProperty, ...);
            nestedValue = getPropertyValue(nestedProperty);
        }
        // 对嵌套对象递归处理
        return BeanWrapperImpl.forNestedProperty(nestedValue, nestedPath);
    } else {
        return this; // 没有 . 了，就是直接属性
    }
}
```

## 10.7 集合属性注入源码

```
XML 中的 <list>/<set>/<map>/<array>/<props> 标签被解析为对应的 Managed 类型：

- <array>   → ManagedArray     (继承 ArrayList，标记为数组)
- <list>    → ManagedList      (继承 ArrayList)
- <set>     → ManagedSet       (继承 LinkedHashSet)
- <map>     → ManagedMap       (继承 LinkedHashMap)
- <props>   → ManagedProperties(继承 Properties)

在 applyPropertyValues() 中：
→ resolveManagedCollection() → 递归解析集合中的每个元素
   - <value> → TypedStringValue → 转换为目标类型
   - <ref>   → RuntimeBeanReference → getBean() 获取引用
   - <null/> → null

for (Object element : managedCollection) {
    if (element instanceof TypedStringValue) {
        resolved.add(convertIfNecessary(element.getValue(), elementType));
    } else if (element instanceof RuntimeBeanReference) {
        resolved.add(resolveBeanReference(element.getBeanName()));
    }
}
// 最终通过反射调用 setter 方法注入集合对象
```

## 10.8 设计思想总结

1. BeanWrapper / PropertyAccessor 抽象层：
   Spring 没有直接使用反射设置属性，而是通过 BeanWrapper 包装，
   带来了类型转换、嵌套路径、事件通知等额外能力。

2. 值解析的两阶段设计：
   第一阶段：XML 解析 → TypedStringValue/RuntimeBeanReference（BeanDefinition 阶段）
   第二阶段：populateBean → BeanDefinitionValueResolver → 类型转换（实际注入阶段）
   这种延迟解析允许 BeanFactoryPostProcessor 在中间修改属性值。

3. TypeConverter 类型转换链：
   Spring 内置了大量的 PropertyEditor 和 Converter，支持 String ↔ 任意类型。
   类型转换发生在注入时，而不是解析时，这为类型转换扩展留下了空间。

4. 嵌套属性设置：
   级联赋值的"."语法是 BeanWrapper 的核心特性之一，
   它通过递归获取嵌套对象实现了 deep property access。
