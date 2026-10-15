# 1. c 命名空间注入
```
c命名空间是简化构造方法注入的。
使用c命名空间的两个前提条件：
第一：需要在xml配置文件头部添加c命名空间的配置信息：xmlns:c="http://www.springframework.org/schema/c"
第二：需要提供构造方法。
c命名空间是依靠构造方法的。
注意：不管是p命名空间还是c命名空间，注入的时候都可以注入简单类型以及非简单类型。

public class People {
    private String name;
    private int age;
    private boolean sex;

    // c命名空间是简化构造注入的。
    // c命名空间注入办法是基于构造方法的。
    public People(String name, int age, boolean sex) {
        this.name = name;
        this.age = age;
        this.sex = sex;
    }

    @Override
    public String toString() {
        return "People{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", sex=" + sex +
                '}';
    }
}

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:c="http://www.springframework.org/schema/c"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <!--
        第一步：在spring的配置文件头部添加: 
  					xmlns:c="http://www.springframework.org/schema/c"
        第二步：使用
            c:_0 下标方式（非简单类型用c:_0-ref）
            c:name 参数名方式（非简单类型用c:name-ref）
    -->
    <!--<bean id="peopleBean" class="com.powernode.spring6.bean.People" c:_0="zhangsan" c:_1="30" c:_2="true"></bean>-->
    <bean id="peopleBean" class="com.powernode.spring6.bean.People" c:name="jack" c:age="30" c:sex="true"></bean>

</beans>

除此之外Spring 6 + JDK 17 默认拿不到构造器参数名，所以使用c:name/c:name-ref方式会失效，而且XML中属性的解析顺序 ≠ 构造器顺序，从而可能导致注入失败（xml写的参数和注入时顺序对不上），此时需要加入以下内容到pom.xml中，从而在构建项目时可以使用到-parameters参数，从而项目可以获取到构造器的参数名
<project>
	...
	<build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.13.0</version>
                <configuration>
                    <parameters>true</parameters>
                    <source>17</source>
                    <target>17</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

# 2. 源码解析：c-namespace 与构造注入的底层映射

## 2.1 c-namespace 如何映射到构造方法参数

```
c-namespace 在 BeanDefinitionParserDelegate.decorateBeanDefinitionIfRequired() 中处理，
它被转换为 ConstructorArgumentValues，与 <constructor-arg> 标签等价。

解析流程：
  c:name="jack"   → constructorArgumentValues.add(0, "name", TypedStringValue("jack"))
  c:_0="zhangsan" → constructorArgumentValues.add(0, null, TypedStringValue("zhangsan"))
  c:userDao-ref="ud" → constructorArgumentValues.add(0, "userDao", RuntimeBeanReference("ud"))

源码片段（BeanDefinitionParserDelegate）：
if (isCNamespace(nodeName)) {
    String argName = nodeName.substring(nodeName.indexOf(':') + 1);
    if (nodeName.endsWith("-ref")) {
        // ★ 引用类型注入
        finalDefinition = parseCArgWithRef(argName, nodeValue, finalDefinition);
        // 内部：new ConstructorArgumentValues.ValueHolder(
        //         new RuntimeBeanReference(nodeValue), null, argName)
    } else {
        // ★ 简单类型注入
        finalDefinition = parseCArgWithValue(argName, nodeValue, finalDefinition);
        // 内部：new ConstructorArgumentValues.ValueHolder(
        //         new TypedStringValue(nodeValue), null, argName)
    }
}
```

## 2.2 JDK 17 -parameters 标志的源码原因

```
为什么 Spring 6 + JDK 17 需要 -parameters 才能使用 c:name 方式？

原因在 Constructor.getParameters() 的行为上：

// 没有 -parameters 时：
Constructor<People> ctor = People.class.getConstructor(String.class, int.class, boolean.class);
Parameter[] params = ctor.getParameters();
params[0].getName(); // → "arg0" （合成名，不是 "name"）
params[1].getName(); // → "arg1" （合成名，不是 "age"）

// 有 -parameters 时：
params[0].getName(); // → "name" （真实参数名）
params[1].getName(); // → "age"  （真实参数名）

Spring 的 ConstructorResolver.autowireConstructor() 中查找匹配构造方法时：
if (paramNames != null) {
    // ★ 有参数名 → 按名称精确匹配
    argsHolder = createArgumentArray(beanName, mbd, resolvedValues, bw, 
            paramTypes, paramNames, ...);
} else {
    // ★ 无参数名 → 按类型 + 下标匹配（需要 c:_0 方式）
    argsHolder = createArgumentArray(beanName, mbd, resolvedValues, bw, 
            paramTypes, null, ...);
}

这就是为什么没有 -parameters 时，c:name 方式失效，只能用 c:_0 方式。

Spring 6 官方推荐：
  <plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
      <parameters>true</parameters>
    </configuration>
  </plugin>
```

## 2.3 构造注入参数解析：ConstructorResolver 源码片段

```
ConstructorResolver.resolveConstructorArguments() — 解析构造参数的核心：

private Object[] resolveConstructorArguments(
        String beanName, RootBeanDefinition mbd, BeanWrapper bw,
        ConstructorArgumentValues cargs, ConstructorArgumentValues resolvedValues) {

    // 获取候选构造方法
    Constructor<?>[] candidates = determineConstructorsFromBeanPostProcessors(beanClass, beanName);

    for (Constructor<?> candidate : candidates) {
        Class<?>[] paramTypes = candidate.getParameterTypes();
        String[] paramNames = ParameterNameDiscoverer.getParameterNames(candidate);
        //                   ↑ 没有 -parameters 时返回 null

        for (int i = 0; i < paramTypes.length; i++) {
            String paramName = (paramNames != null ? paramNames[i] : null);
            // ★ 从 constructorArgumentValues 中查找匹配的值
            ValueHolder valueHolder = resolveConstructorArgument(
                    beanName, mbd, paramName, i, paramTypes[i]);
            // 查找规则：优先按 name → 再按 index → 最后按 type 自动匹配
        }
    }
}
```
