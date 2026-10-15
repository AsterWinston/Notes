# 1. AOP介绍
AOP（Aspect Oriented Programming）：面向切面编程，面向方面编程。
	切面：业务流程中，与业务逻辑不挂钩的非业务逻辑通用代码，如：事务
AOP是一种编程技术
面向切面编程：将和业务逻辑不挂钩的通用代码抽取出来，形成一个独立的组件，形成一个横向的切面，在纵向的业务逻辑代码中，如果需要用到切面时，可以将切面以交叉业务的形式切入业务流程中。可以增强代码的复用性、可维护性、使我们更专注于业务逻辑代码
AOP是OOP的补充延伸
AOP底层是使用动态代理实现的，动态代理是面向切面编程思想的实现
Spring的AOP使用的动态代理是：JDK动态代理 + CGLIB动态代理技术。
    Spring在这两种动态代理中灵活切换，
    如果是代理接口，会默认使用JDK动态代理，
    如果要代理某个类，这个类没有实现接口，就会切换使用CGLIB。
    当然，可以强制通过一些配置让Spring只使用CGLIB。
一般一个系统当中都会有一些系统服务，例如：日志、事务管理、安全等。这些系统服务被称为：交叉业务
	纵向的为业务逻辑代码，横向的为交叉业务
这些交叉业务几乎是通用的，不管你是做银行账户转账，还是删除用户数据。日志、事务管理、安全，这些都是需要做的。
如果在每一个业务处理过程当中，都掺杂这些交叉业务代码进去的话，存在两方面问题：
    第一：交叉业务代码在多个业务流程中反复出现，显然这个交叉业务代码没有得到复用。并且修改这些交叉业务代码的话，需要修改多处，维护成本高。
    第二：程序员无法专注核心业务代码的编写，在编写核心业务代码的同时还需要处理这些交叉业务。
使用AOP（动态代理）可以很轻松的解决以上问题
AOP：将与核心业务无关的代码独立的抽取出来，形成一个独立的组件，然后以横向交叉的方式应用到业务流程当中的过程被称为AOP。
AOP的优点：
    第一：代码复用性增强。
    第二：代码易维护。
    第三：使开发者更关注业务逻辑。

# 2. 七大术语
通知（Advice）：又叫增强，具体增强的代码，如具体的事务代码、日志代码…
    通知描述的是代码（增强代码）
    通知包括：
        前置通知：通知的代码只放在业务方法之前
        后置通知：通知的代码只放在业务方法之后
        环绕通知：通知的代码在业务方法之前和之后都有
        异常通知：通知代码在捕获异常 catch(){} 的代码块之中
        最终通知：通知代码在 finally{} 的代码块之中
织入（Weaving）：把通知应用到目标对象上的过程
连接点（Joinpoint）：可以织入切面的位置
    方法的执行前后、异常抛出之后、finally{} 的代码块等
    连接点描述的是位置，可以加入增强代码的位置，可以应用通知的位置
切点（Pointcut）：织入切面的那个方法
    切点本质上就是方法，织入切面的方法
    切点，要被通知增强的方法
切面（Aspect）：切点 + 通知
代理对象（Proxy）：目标对象被织入通知后产生的新对象
目标对象（Target）：被织入通知的对象


# 3. 切点表达式
```
切点是要在其前后织入通知的方法
切点表达式，可以匹配切点方法的表达式，用于定义通知往哪些方法上切入
切点表达式语法
切点表达式的语法格式：
execution([访问控制权限修饰符] 返回值类型 [全限定类名]方法名(形式参数列表) [异常])
切点表达式以execution(开始，以)结束
访问控制权限修饰符：
    public | protected | private | default
    可选项。
    没写，就是4个访问控制权限都包括。
    写public就表示只包括公开的方法。
返回值类型：
    必填项。
        表示返回值类型任意。
    如果 String，返回值类型就是 String
全限定类名：
    带包名的类名
    可选项。
    两个点“…”代表当前包以及子包下的所有类。com.. 表示com包以及子包下的所有类
    全限定类名省略时表示所有的类。
方法名：
    必填项。
    *表示所有方法。
    set*表示所有的以set开头的方法。
形式参数列表：
    必填项
    () 表示没有参数的方法
    (…) 参数类型和个数随意的方法
    (*) 只有一个参数的方法
    (*, String) 第一个参数类型随意，第二个参数是String的。
异常：
    可选项。
    省略时表示任意异常类型。

切点表达式实例
    execution(public * com.powernode.mall.service.*.delete*(..))：service包下所有的类中以delete开始的所有公共的任意返回值的方法
    execution(* com.powernode.mall..*(..))：mall包下所有的类的所有的方法
    execution(* *(..))：所有类的所有方法

```

# 4. 使用Spring的AOP
Spring对AOP的实现包括以下3种方式：
    第一种方式：Spring框架结合AspectJ框架实现的AOP，基于注解方式。
    第二种方式：Spring框架结合AspectJ框架实现的AOP，基于XML方式。
        AspectJ框架专门做AOP的框架
    第三种方式：Spring框架自己实现的AOP，基于XML配置方式。
AspectJ 框架
    Eclipse组织的一个支持AOP的框架。
    AspectJ框架是独立于Spring框架之外的一个框架，Spring框架借助于AspectJ
## 4.1 依赖
```
<!--spring context依赖-->
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-context</artifactId>
  <version>6.0.8</version>
</dependency>
<!--spring aop依赖-->
<!-- 
	不需要自己手动引入spring-aop依赖，
	引入spring-context依赖会自动关联引入spring-aop依赖
-->
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-aop</artifactId>
  <version>6.0.8</version>
</dependency>
<!--spring aspects依赖-->
<!-- AspectJ -->
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-aspects</artifactId>
  <version>6.0.8</version>
</dependency>
```
## 4.2 Spring 配置文件中添加 context 命名空间和 aop 命名空间
```
    基于注解的方式需要组件扫描，需要 context 命名空间
    需要使用aop，需要aop命名空间
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
						   http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
						   http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">

  <!-- 组件扫描 -->
  <context:component-scan base-package="cw.study.spring"/>

</beans>
```
## 4.3 目标类
```
package cw.study.spring.service;

/**
 * ClassName: UserService
 * Package: cw.study.spring.service
 * Description:
 *
 * @Author tcw
 * @Create 2023-06-03 22:37
 * @Version 1.0
 */
@Service
public class UserService {
    
    public void login() {
        System.out.println("登录中...");
    }
    
}
```
## 4.4 切面（切面 = 通知 + 切点）
```
package cw.study.spring.aspect;

import org.aspectj.lang.annotation.Before;
import org.springframework.context.annotation.Bean;
import org.springframework.stereotype.Component;

/**
 * ClassName: LogAspect
 * Package: cw.study.spring.aspect
 * Description:
 * 记录日志的切面
 * 切面 = 通知 + 切点
 * 通知就是增强代码，通知以方法的形式出现
 *
 * @Author tcw
 * @Create 2023-06-03 22:41
 * @Version 1.0
 */
@Component // 切面也是组件
@Aspect // 切面类需要改注解进行标注
public class LogAspect {
    
    /**
     * @Before 注解标注的方法为前置通知
     * 注解中需要写切点表达式，因为 切面 = 通知 + 切点
     * 所以注解的属性value的属性值为切点表达式，告知要织入的切点
     * cw.study.spring.service包下的UserService类中的所有方法
     * 方法名可以随意定义，要见名知意即可
     */
    @Before(value = "execution(* cw.study.spring.service.UserService.*(..))")
    public void beforeAdvice() {
        System.out.println("前置通知...");
    }
    
}
```
## 4.5 开启 AspectJ 自动代理
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
						   http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
						   http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">

	<!-- 组件扫描 -->
	<context:component-scan base-package="cw.study.spring"/>
	
	<!-- 开启AspectJ自动代理 -->
  <!-- 
  	spring 在扫描类时，会检查类上是否有 Aspect 注解
  	如果有，则给这个类生成代理对象
  -->
  <!-- 
    proxy-target-class="true" 表示强制使用CGLIB动态代理
    proxy-target-class="false" 默认值，表示接口使用JDK动态代理，反之使用CGLIB动态代理
  -->
	<aop:aspectj-autoproxy proxy-target-class="true"/>
	
</beans>
```
## 4.6 Before 前置通知测试
```
前置通知：@Before 目标方法执行之前的通知

@org.junit.Test
public void testBeforeAdvice() {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    UserService userService = applicationContext.getBean("userService", UserService.class);
    userService.login();
}
```
## 4.7 AfterReturning 后置通知
```
后置通知：@AfterReturning 目标方法执行之后的通知

@AfterReturning("execution(* cw.spring.service.UserService.*(..))")
public void afterReturningAdvice() {
    System.out.println("后置通知...");
}
```
## 4.8 Around 环绕通知
```
环绕通知：@Around 目标方法之前添加通知，同时目标方法执行之后添加通知。
环绕通知是范围最大的通知，前环绕在前置通知之前，后环绕在所有通知之后
/**
 * 环绕通知
 * 
 * @param joinPoint 连接点
 */
@Around("execution(* cw.spring.service.UserService.*(..))")
public void aroundAdvice(ProceedingJoinPoint joinPoint) throws Throwable {
    // 前环绕
    System.out.println("前环绕...");
    // 调用执行目标
    joinPoint.proceed();
    // 后环绕
    System.out.println("后环绕...");
}
```
## 4.9 After 最终通知
```
	最终通知：@After 放在finally语句块中的通知
// 最终通知
@After("execution(* cw.spring.service.UserService.*(..))")
public void afterAdvice() {
    System.out.println("最终通知...");
}
```
## 4.10 AfterThrowing 异常通知
```
	异常通知：@AfterThrowing 发生异常之后执行的通知
// 异常通知
@AfterThrowing("execution(* cw.spring.service.UserService.*(..))")
public void afterThrowingAdvice() {
    System.out.println("异常通知...");
}
public class UserService {
    
    public void login() {
        System.out.println("登录中...");
        throw new RuntimeException("运行时异常...");
    }
    
}

如果环绕通知中调将用目标方法时的异常在环绕通知代码中进行捕获，则后环绕通知会执行，相当于在环绕通知中将目标方法抛出的异常进行了捕获，异常被捕获程序可以继续向下执行
/**
 * 环绕通知
 *
 * @param joinPoint 连接点
 */
@Around("execution(* cw.spring.service.UserService.*(..))")
public void aroundAdvice(ProceedingJoinPoint joinPoint) {
    // 前环绕
    System.out.println("前环绕...");
    // 调用执行目标
    try {
        joinPoint.proceed();
    } catch (Throwable e) {
        e.printStackTrace();
    }
    // 后环绕
    System.out.println("后环绕...");
}
```
## 4.11 Order 切面顺序
```
使用 @Order 注解可以指定切面的执行顺序
    数字越小，优先级越高
package cw.spring.service;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

/**
 * ClassName: SecurityAspect
 * Package: cw.spring.service
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-17 9:32
 * @Version 1.0
 */
@Component
@Aspect
@Order(1)
public class SecurityAspect {
    // 前置通知
    @Before("execution(* cw.spring.service.UserService.*(..))")
    public void beforeAdvice() {
        System.out.println("优先级 1 前置通知...");
    }
    
    // 后置通知
    @AfterReturning("execution(* cw.spring.service.UserService.*(..))")
    public void afterReturningAdvice() {
        System.out.println("优先级 1 后置通知...");
    }
    
    /**
     * 环绕通知
     *
     * @param joinPoint 连接点
     */
    @Around("execution(* cw.spring.service.UserService.*(..))")
    public void aroundAdvice(ProceedingJoinPoint joinPoint) {
        // 前环绕
        System.out.println("优先级 1 前环绕...");
        // 调用执行目标
        try {
            joinPoint.proceed();
        } catch (Throwable e) {
            e.printStackTrace();
        }
        // 后环绕
        System.out.println("优先级 1 后环绕...");
    }
    
    // 最终通知
    @After("execution(* cw.spring.service.UserService.*(..))")
    public void afterAdvice() {
        System.out.println("优先级 1 最终通知...");
    }
    
    // 异常通知
    @AfterThrowing("execution(* cw.spring.service.UserService.*(..))")
    public void afterThrowingAdvice() {
        System.out.println("优先级 1 异常通知...");
    }
}
package cw.spring.service;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

/*
 * ClassName: LogAspect
 * Package: cw.spring.service
 * Description:
 * 切面 = 通知 + 切点
 *
 * @Author tcw
 * @Create 2023-05-16 16:03
 * @Version 1.0
 */
@Component
@Aspect // 标注为切面类
@Order(0)
public class LogAspect {
    
    // 前置通知
    @Before("execution(* cw.spring.service.UserService.*(..))")
    public void beforeAdvice() {
        System.out.println("优先级 0 前置通知...");
    }
    
    // 后置通知
    @AfterReturning("execution(* cw.spring.service.UserService.*(..))")
    public void afterReturningAdvice() {
        System.out.println("优先级 0 后置通知...");
    }
    
    /**
     * 环绕通知
     *
     * @param joinPoint 连接点
     */
    @Around("execution(* cw.spring.service.UserService.*(..))")
    public void aroundAdvice(ProceedingJoinPoint joinPoint) {
        // 前环绕
        System.out.println("优先级 0 前环绕...");
        // 调用执行目标
        try {
            joinPoint.proceed();
        } catch (Throwable e) {
            e.printStackTrace();
        }
        // 后环绕
        System.out.println("优先级 0 后环绕...");
    }
    
    // 最终通知
    @After("execution(* cw.spring.service.UserService.*(..))")
    public void afterAdvice() {
        System.out.println("优先级 0 最终通知...");
    }
    
    // 异常通知
    @AfterThrowing("execution(* cw.spring.service.UserService.*(..))")
    public void afterThrowingAdvice() {
        System.out.println("优先级 0 异常通知...");
    }
    
}
数字越小，优先级越高
后置通知，优先级越高越后执行
前置通知，优先级越高越先执行
环绕通知，优先级越高越外层执行
在切点执行之前执行的通知和在切点执行之后执行的通知，同一优先级执行完，再执行下一级优先级
```
## 4.12 通用切点表达式
```
    使用通用切点表达式可以简化切点表达式，使切点表达式的代码得到复用
    在其他切面类中使用："包名.LogAspect.通用切点表达式对应方法的方法名()"
@Component
@Aspect // 标注为切面类
@Order(0)
public class LogAspect {
    
    // 通用切点表达式
    // 该通用切点表达式要跨类使用，在其他切面类中使用，前面写上包名类名即可
    // 如当前通用切点表达式在LogAspect类中，
    // 
    // 该方法只是一个标记，方法名任意，方法体不用写代码
    @Pointcut("execution(* cw.spring.service.UserService.*(..))")
    public void GeneralPointcutExpression() {}
    
    // 前置通知
    @Before("GeneralPointcutExpression()")
    public void beforeAdvice() {
        System.out.println("优先级 0 前置通知...");
    }
    
    // 后置通知
    @AfterReturning("GeneralPointcutExpression()")
    public void afterReturningAdvice() {
        System.out.println("优先级 0 后置通知...");
    }
    
    /**
     * 环绕通知
     *
     * @param joinPoint 连接点
     */
    @Around("GeneralPointcutExpression()")
    public void aroundAdvice(ProceedingJoinPoint joinPoint) {
        // 前环绕
        System.out.println("优先级 0 前环绕...");
        // 调用执行目标
        try {
            joinPoint.proceed();
        } catch (Throwable e) {
            e.printStackTrace();
        }
        // 后环绕
        System.out.println("优先级 0 后环绕...");
    }
    
    // 最终通知
    @After("GeneralPointcutExpression()")
    public void afterAdvice() {
        System.out.println("优先级 0 最终通知...");
    }
    
    // 异常通知
    @AfterThrowing("GeneralPointcutExpression()")
    public void afterThrowingAdvice() {
        System.out.println("优先级 0 异常通知...");
    }
    
}
```

## 4.13 连接点
```
环绕通知，Spring在调用通知方法时，会传入ProceedingJionPoint对象（连接点对象）
    除了环绕通知以外的其他通知，Spring在调用时，会传入JoinPoint对象（连接点对象）
    通过JoinPoint对象（连接点对象），我们可以获取目标方法的相关信息
对于JoinPoint对象（前置通知（@Before）、后置通知（@After）、异常通知（@AfterThrowing））
| 方法               | 说明                  |
| ---------------- | ------------------- |
| `getArgs()`      | 获取方法参数数组            |
| `getTarget()`    | 获取目标对象（被代理对象）       |
| `getThis()`      | 获取代理对象              |
| `getSignature()` | 获取方法签名信息，如方法名、返回类型等 |
对于ProceedingJoinPoint（环绕通知（@Around））
| 方法                       | 说明              |
| ------------------------ | --------------- |
| `proceed()`              | 执行目标方法，返回结果     |
| `proceed(Object[] args)` | 执行目标方法，并可传入新的参数 |

// 前置通知
@Before("GeneralPointcutExpression()")
public void beforeAdvice(JoinPoint joinPoint) {
    System.out.println("优先级 0 前置通知...");
    // 获取目标方法的相关信息
    // 获取目标方法的签名，即 public void login() 方法的修饰符列表、方法名等
	// 方法的签名：访问权限修饰符开始到方法名
    Signature signature = joinPoint.getSignature();
    // 通过方法的签名可以获取方法的具体信息
    System.out.println("方法的方法名：" + signature.getName());
}

```
## 4.14 执行链
```
JdkDynamicAopProxy / CglibAopProxy
        ↓
ReflectiveMethodInvocation.proceed()
        ↓
┌─ ExposeInvocationInterceptor（可选）
│
├─ AspectJAroundAdvice.invoke()        → @Around（前）
│      ↓
│   MethodInvocation.proceed()
│      ↓
│   ┌─ MethodBeforeAdviceInterceptor   → @Before
│   │      ↓
│   │   MethodInvocation.proceed()
│   │      ↓
│   │   ┌─ 目标方法 invoke()
│   │   │
│   │   └─ AfterReturningAdviceInterceptor → @AfterReturning
│   │
│   └─ AfterAdviceInterceptor          → @After
│
└─ AspectJAroundAdvice.invoke()        → @Around（后）

```

# 5. 全注解式开发
```
@Configuration // 代替spring.xml
@ComponentScan({"cw.spring.service"}) // 指定组件扫描的包
// 开启aspectj自动代理
// proxyTargetClass = true 使用CGLIB自动代理
// proxyTargetClass = false 默认值，表示接口使用JDK动态代理，反之使用CGLIB动态代理
@EnableAspectJAutoProxy(proxyTargetClass = true) 
public class Spring6Config {
}

@org.junit.Test
public void test01() {
    ApplicationContext applicationContext = new AnnotationConfigApplicationContext(Spring6Config.class);
    UserService userService = applicationContext.getBean("userService", UserService.class);
    userService.login();
}

```

# 6. 基于XML配置方式的AOP
## 6.1 依赖
```
<dependencies>
	<!-- spring 核心依赖 -->
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>6.0.8</version>
  </dependency>
	<!-- spring AspectJ 依赖 -->
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aspects</artifactId>
    <version>6.0.8</version>
  </dependency>
  <dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.13.2</version>
    <scope>test</scope>
  </dependency>
</dependencies>

```
## 6.2 目标类
```
public class UserService {
    
    public void logout() {
        System.out.println("退出...");
    }
    
}

```
## 6.3 切面类
```
public class LogAspect {
    
    /**
     * 环绕通知
     * 
     * @param joinPoint 连接点
     */
    public void aroundAdvice(ProceedingJoinPoint joinPoint) throws Throwable {
        // 前环绕
        long start = System.currentTimeMillis();
        // 执行目标方法
        joinPoint.proceed();
        // 后环绕
        long end = System.currentTimeMillis();
        System.out.println("耗时：" + (end - start));
    }
    
}

```
## 6.4 spring.xml配置
```
命名空间依然需要
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
						   http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
						   http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">

	<!-- 组件纳入spring容器 -->
	<bean id="userService" class="cw.study.spring.service.UserService"/>
	<!-- 切面也是组件，纳入spring容器 -->
	<bean id="logAspect" class="cw.study.spring.aspect.LogAspect"/>
	
	<!-- aop 配置 -->
	<aop:config>
		<!-- 切点表达式 cw.study.spring.service包下的所有类的所有方法 -->
		<aop:pointcut id="myPointcut" expression="execution(* cw.study.spring.service..*(..))"/>
		<!-- 切面 = 通知 + 切点 -->
		<!-- ref 使用的切面类 -->
		<aop:aspect ref="logAspect">
			<!-- 
				不同类型的通知需要使用相应的标签，
				method：通知的方法
				pointcut-ref：切点
			-->
			<aop:around method="aroundAdvice" pointcut-ref="myPointcut"/>
		</aop:aspect>
	</aop:config>
	
</beans>
```

## 6.5 测试
```
@org.junit.Test
public void test() {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    UserService userService = applicationContext.getBean("userService", UserService.class);
    userService.logout();
}

```


# 7. AOP的实际案例：事务处理
```
项目中的事务控制是在所难免的。在一个业务流程当中，可能需要多条DML语句共同完成，为了保证数据的安全，这多条DML语句要么同时成功，要么同时失败。这就需要添加事务控制的代码。
在业务类中的每一个业务方法都是需要控制事务的，而控制事务的代码又是固定的格式，都是：
try{
    // 开启事务
    startTransaction();

    // 执行核心业务逻辑
    //......

    // 提交事务
    commitTransaction();
}catch(Exception e){
    // 回滚事务
    rollbackTransaction();
}
这个控制事务的代码就是和业务逻辑没有关系的“交叉业务”。这些交叉业务的代码没有得到复用，在每个业务类的业务方法中，都要开启事务、提交事务以及回滚事务的代码，代码复用低，并且如果这些交叉业务代码需要修改，那必然需要修改多处，难维护，怎么解决？可以采用AOP思想解决。
可以把以上控制事务的代码作为环绕通知，切入到目标类的方法当中。
```
## 7.1 依赖
```
<dependencies>
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>6.0.8</version>
  </dependency>
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aspects</artifactId>
    <version>6.0.8</version>
  </dependency>
  <dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.13.2</version>
    <scope>test</scope>
  </dependency>
</dependencies>
```
## 7.2 目标类（业务类）
```
package cw.study.spring.service;

/**
 * ClassName: AccountService
 * Package: cw.study.spring.service
 * Description:
 * 银行账户业务类
 *
 * @Author tcw
 * @Create 2023-06-04 20:02
 * @Version 1.0
 */
@Service
public class AccountService {
    
    public void transfer() {
        System.out.println("转账中...");
        throw new RuntimeException("运行时异常...");
    }
    
    public void withdraw() {
        System.out.println("取款中...");
    }
    
}
/**
 * ClassName: OrderService
 * Package: cw.study.spring.service
 * Description:
 * 订单业务类
 *
 * @Author tcw
 * @Create 2023-06-04 20:04
 * @Version 1.0
 */
@Service
public class OrderService {
    
    public void generate() {
        System.out.println("订单生成中...");
    }
    
    public void cancel() {
        System.out.println("订单取消中...");
    }
    
}
```
## 7.3 切面类
```
package cw.study.spring.aspect;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;

/**
 * ClassName: TransactionAspect
 * Package: cw.study.spring.aspect
 * Description:
 * 事务切面类
 *
 * @Author tcw
 * @Create 2023-06-04 20:06
 * @Version 1.0
 */
@Component
@Aspect
public class TransactionAspect {
    
    // cw.study.spring.service 包下的所有类的所有方法
    @Around("execution(* cw.study.spring.service..*(..))")
    public void aroundAdvice(ProceedingJoinPoint joinPoint) {
        try {
            // 前环绕，开启事务
            System.out.println("开启事务...");
            // 执行目标
            joinPoint.proceed();
            // 后环绕
            // 完成事务，提交事务
            System.out.println("提交事务...");
        } catch (Throwable e) {
            // 发生异常，事务回滚
            System.out.println("回滚事务");
        }
    }
    
}
```
## 7.4 spring.xml配置
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
						   http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
						   http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">

	<!-- 组件扫描 -->
	<context:component-scan base-package="cw.study.spring"/>
	
	<!-- 开启AOP自动代理 -->
	<aop:aspectj-autoproxy/>
	
</beans>
```
## 7.5 测试代码
```
@org.junit.Test
public void test() {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    AccountService accountService = applicationContext.getBean("accountService", AccountService.class);
    OrderService orderService = applicationContext.getBean("orderService", OrderService.class);
    accountService.transfer();
    accountService.withdraw();
    orderService.generate();
    orderService.cancel();
}

```

# 8. AOP的实际案例：安全日志
需求是这样的：项目开发结束了，已经上线了。运行正常。客户提出了新的需求：凡是在系统中进行修改操作的，删除操作的，新增操作的，都要把这个人记录下来。因为这几个操作是属于危险行为。
## 8.1 目标类
```
package cw.study.spring.service;

import org.springframework.stereotype.Service;

/**
 * ClassName: UserService
 * Package: cw.study.spring.service
 * Description:
 *
 * @Author tcw
 * @Create 2023-06-03 22:37
 * @Version 1.0
 */
@Service
public class UserService {
    
    public void insertUser() {
        System.out.println("新增用户信息...");
    }
    
    public void updateUser() {
        System.out.println("修改用户信息...");
    }
    
}

```
## 8.2 切面类
```
package cw.study.spring.aspect;


import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.stereotype.Component;

import java.text.SimpleDateFormat;
import java.util.Date;

/**
 * ClassName: LogAspect
 * Package: cw.study.spring.aspect
 * Description:
 *
 * @Author tcw
 * @Create 2023-06-03 22:41
 * @Version 1.0
 */
@Component
@Aspect
public class LogAspect {
    
    // 定义通用切点表达式
    // cw.study.spring.service包下的所有insert开头的方法
    @Pointcut("execution(* cw.study.spring.service..insert*(..))")
    public void insertPointcut() {}
    // cw.study.spring.service包下的所有update开头的方法
    @Pointcut("execution(* cw.study.spring.service..update*(..))")
    public void updatePointcut() {}
    // cw.study.spring.service包下的所有delete开头的方法
    @Pointcut("execution(* cw.study.spring.service..delete*(..))")
    public void deletePointcut() {}
    // cw.study.spring.service包下的所有modify开头的方法
    @Pointcut("execution(* cw.study.spring.service..modify*(..))")
    public void modifyPointcut() {}
    
    // 前置通知
    // 切点表达式 insertPointcut() 或者 updatePointcut() 或者 deletePointcut() 或者 modifyPointcut()
    @Before("insertPointcut() || updatePointcut() || deletePointcut() || modifyPointcut()")
    public void beforeAdvice(JoinPoint joinPoint) {
        // 获取系统时间
        SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss SSS");
        String date = simpleDateFormat.format(new Date());
        // joinPoint 包含了执行方法的相关信息
        // getDeclaringTypeName 获取执行方法的类名
        // getName 获取执行方法的方法名
        System.out.println(date + " 张三 : " + joinPoint.getSignature().getDeclaringTypeName() + "." +
                joinPoint.getSignature().getName());
        
    }
    
}

```
## 8.3 spring.xml配置
```
<!-- 组件扫描 -->
<context:component-scan base-package="cw.study.spring"/>

<!-- 开启AOP自动代理 -->
<aop:aspectj-autoproxy/>

```
## 8.4 测试
```
@org.junit.Test
public void test() {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    UserService userService = applicationContext.getBean("userService", UserService.class);
    userService.insertUser();
    userService.updateUser();
}
```

# 9. 源码解析：AOP 代理创建与执行链

## 9.1 @EnableAspectJAutoProxy 的事件链路

```
当我们在配置类上添加 @EnableAspectJAutoProxy 时，发生了什么？

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {
    boolean proxyTargetClass() default false;  // 默认 false：有接口用 JDK，无接口用 CGLIB
    boolean exposeProxy() default false;       // 默认 false：是否暴露代理到 AopContext
}

★ @Import(AspectJAutoProxyRegistrar.class) 是关键！

AspectJAutoProxyRegistrar 是一个 ImportBeanDefinitionRegistrar 实现类，
它在 Spring 容器启动的 ConfigurationClassPostProcessor 处理阶段被调用：

class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(
            AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

        // ★ 注册 AnnotationAwareAspectJAutoProxyCreator（一个 BPP）
        AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);

        AnnotationAttributes enableAspectJAutoProxy =
                AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
        if (enableAspectJAutoProxy != null) {
            if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
                // 强制使用 CGLIB 代理
                AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
            }
            if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
                // 将代理对象暴露到 AopContext（用于同一类内方法互调）
                AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
            }
        }
    }
}

// AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary() 做的事情：
public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(
        BeanDefinitionRegistry registry, @Nullable Object source) {
    return registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class,
            registry, source);
    // ★ 向容器注册 AnnotationAwareAspectJAutoProxyCreator 的 BeanDefinition
    // Bean 名称为：org.springframework.aop.config.internalAutoProxyCreator
}
```

## 9.2 AnnotationAwareAspectJAutoProxyCreator 的类继承结构

```
AopProxy (Interface)                                 BeanPostProcessor (Interface)
  │                                                    │
  └─── AopProxyFactory (Interface)                      │
                                                        │
InstantiationAwareBeanPostProcessor (Interface)          │
  │                                                      │
SmartInstantiationAwareBeanPostProcessor (Interface)     │
  │                                                      │
AbstractAutoProxyCreator                                 │
  │  - postProcessBeforeInstantiation()                  │
  │  - postProcessAfterInitialization() ★ 核心           │
  │  - getEarlyBeanReference() ★ 用于循环依赖             │
  │  - wrapIfNecessary() — 判断是否需要代理               │
  │  - createProxy() — 创建代理对象                       │
  │                                                      │
  └─── AbstractAdvisorAutoProxyCreator                   │
       │  - getAdvicesAndAdvisorsForBean()               │
       │  - findEligibleAdvisors()                       │
       │                                                 │
       └─── AnnotationAwareAspectJAutoProxyCreator       │
            ★ 最终实现类                                  │
            - 扫描 @Aspect 注解的 Bean                    │
            - 解析其中的 @Before/@Around/@After 等通知    │
            - 转化为 Advisor 列表                         │
```

### 9.2.1 关键接口说明

```
SmartInstantiationAwareBeanPostProcessor 定义了三个与实例化相关的方法：

1. getEarlyBeanReference(Object bean, String beanName)
   → 在循环依赖场景下，提前获取 Bean 的引用（可能返回代理对象）
   → 确保循环依赖 + AOP 场景下的一致性

2. postProcessBeforeInstantiation(Class<?> beanClass, String beanName)
   → 在 Bean 实例化之前调用
   → 如果返回非 null，Spring 将使用返回值作为 Bean（跳过后续实例化流程）
   → AbstractAutoProxyCreator 在此处处理自定义 TargetSource 场景

3. postProcessAfterInitialization(Object bean, String beanName)
   → 在 Bean 初始化完成后调用
   → ★ 这是普通 AOP 代理创建的主入口
```

## 9.3 AOP 代理创建流程：wrapIfNecessary() 源码

```
AbstractAutoProxyCreator.postProcessAfterInitialization() — AOP 代理创建的主入口：

@Override
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
    if (bean != null) {
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        // 检查该 Bean 之前是否已经被处理过（避免重复代理）
        if (this.earlyProxyReferences.remove(cacheKey) != bean) {
            // ★ 核心方法：如果该 Bean 需要 AOP，则创建代理
            return wrapIfNecessary(bean, beanName, cacheKey);
        }
    }
    return bean;
}

protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    // 1. 如果该 Bean 已经被 TargetSource 方式处理过（如 <aop:scoped-proxy/>），跳过
    if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
        return bean;
    }

    // 2. 如果该 Bean 被标记为"不应被代理"（如 AopInfrastructureBean），跳过
    if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
        return bean;
    }

    // 3. 如果该 Bean 本身是基础设施（如 Advisor、Advice、AopInfrastructureBean），跳过
    //    或者该 Bean 应该在织入时被跳过（shouldSkip 方法）
    if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }

    // ★★★ 4. 核心：获取与当前 Bean 匹配的所有 Advisor（通知 + 切点的组合） ★★★
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    if (specificInterceptors != DO_NOT_PROXY) {  // DO_NOT_PROXY 是一个特殊的 null 标记
        // 标记该 Bean 已被代理
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        // ★ 创建代理对象
        Object proxy = createProxy(bean.getClass(), beanName, specificInterceptors,
                new SingletonTargetSource(bean));
        // 存储代理的类型
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    // 没有匹配的 Advisor，标记为不需要代理
    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}
```

### 9.3.1 AnnotationAwareAspectJAutoProxyCreator.getAdvicesAndAdvisorsForBean()

```
该方法位于 AbstractAdvisorAutoProxyCreator，但 findCandidateAdvisors() 和
findAdvisorsThatCanApply() 由 AnnotationAwareAspectJAutoProxyCreator 实现：

@Override
protected List<Advisor> findCandidateAdvisors() {
    // ★ 步骤 1: 获取所有通过 XML/<aop:advisor> 注册的 Advisor
    List<Advisor> advisors = super.findCandidateAdvisors();

    // ★ 步骤 2: 扫描所有 @Aspect 注解的 Bean，解析其中的 @Before/@Around 等注解
    //    将每个通知方法包装为一个 InstantiationModelAwarePointcutAdvisorImpl
    if (this.aspectJAdvisorsBuilder != null) {
        advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
    }
    return advisors;
}

// 对于每个候选 Advisor，调用 AopUtils.canApply(advisor, targetClass) 判断是否匹配：
// → 切点表达式的 execution 是否匹配目标类的某个方法
// → getIntroductionFilter() 是否有引介增强
```

### 9.3.2 createProxy() 源码

```
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
        @Nullable Object[] specificInterceptors, TargetSource targetSource) {

    if (this.beanFactory instanceof ConfigurableListableBeanFactory clbf) {
        AutoProxyUtils.exposeTargetClass(clbf, beanName, beanClass);
    }

    // ★ 创建 ProxyFactory
    ProxyFactory proxyFactory = new ProxyFactory();
    proxyFactory.copyFrom(this);

    // 设置是否强制使用 CGLIB
    if (proxyFactory.isProxyTargetClass()) {
        if (ProxyTargetClassAware.class.isAssignableFrom(beanClass)) {
            // ...
        }
    }

    // 决定使用 JDK 还是 CGLIB
    if (!proxyFactory.isProxyTargetClass()) {
        if (shouldProxyTargetClass(beanClass, beanName)) {
            proxyFactory.setProxyTargetClass(true);
        } else {
            // ★ 这里实际调用 AopProxyFactory.createAopProxy() 做决策
            evaluateProxyInterfaces(beanClass, proxyFactory);
        }
    }

    // 构建 Advisor 数组（优先处理 specificInterceptors）
    Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
    proxyFactory.addAdvisors(advisors);
    proxyFactory.setTargetSource(targetSource);
    customizeProxyFactory(proxyFactory);

    proxyFactory.setFrozen(this.freezeProxy);
    if (advisorsPreFiltered()) {
        proxyFactory.setPreFiltered(true);
    }

    // 使用原始 ClassLoader
    ClassLoader classLoader = getProxyClassLoader();
    if (classLoader instanceof SmartClassLoader smartClassLoader && smartClassLoader != classLoader) {
        classLoader = smartClassLoader.getOriginalClassLoader();
    }

    // ★ 真正创建 AOP 代理
    return proxyFactory.getProxy(classLoader);
}
```

## 9.4 JDK vs CGLIB 代理决策：DefaultAopProxyFactory

```
DefaultAopProxyFactory.createAopProxy() 是决定使用 JDK 动态代理还是 CGLIB 的最终决策点：

public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {

    @Override
    public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
        // config.isProxyTargetClass() 为 true 表示强制 CGLIB
        // config.isOptimize() 为 true 表示启用 CGLIB 优化策略
        // config.isInterfaceProxied() 检查是否有至少一个接口
        if (!NativeDetector.inNativeImage() &&
                (config.isOptimize() || config.isProxyTargetClass()
                        || hasNoUserSuppliedProxyInterfaces(config))) {

            Class<?> targetClass = config.getTargetClass();
            if (targetClass == null) {
                throw new AopConfigException("TargetSource cannot determine target class: " +
                        "Either an interface or a target is required for proxy creation.");
            }

            // ★ 如果目标类本身就是接口，或者目标类是 JDK 动态代理（Proxy 类），则仍使用 JDK
            if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)
                    || ClassUtils.isLambdaClass(targetClass)) {
                return new JdkDynamicAopProxy(config);
            }

            // ★ 使用 CGLIB 代理
            return new ObjenesisCglibAopProxy(config);
        } else {
            // ★ 使用 JDK 动态代理
            return new JdkDynamicAopProxy(config);
        }
    }

    // 判断是否有用户提供的代理接口
    private boolean hasNoUserSuppliedProxyInterfaces(AdvisedSupport config) {
        Class<?>[] ifcs = config.getProxiedInterfaces();
        return (ifcs.length == 0 || (ifcs.length == 1 && SpringProxy.class.isAssignableFrom(ifcs[0])));
    }
}
```

### 9.4.1 代理选择决策表

```
                                     ┌─────────────────────────────┐
                                     │ 目标类是否实现了至少一个非    │
                                     │ SpringProxy 的接口？         │
                                     └──────────┬──────────────────┘
                                                │
                              ┌─────────────────┼─────────────────────┐
                              │ YES             │                     │ NO
                              ▼                 │                     ▼
                  ┌──────────────────────┐      │         ┌──────────────────────┐
                  │ proxyTargetClass     │      │         │ 直接使用 CGLIB 代理    │
                  │ = true ？            │      │         └──────────────────────┘
                  └──────┬───────────────┘      │
                         │                      │
               ┌─────────┼──────────┐           │
               │ YES     │          │ NO        │
               ▼         │          ▼           │
           CGLIB代理      │      JDK代理         │
                          │  (基于接口)          │
                          │                      │
                           ──────────────────────┘
```

## 9.5 通知执行链：ReflectiveMethodInvocation.proceed() 核心源码

```
这是整个 AOP 通知调用链的核心！它使用递归 + 责任链模式实现拦截器链的执行。

public class ReflectiveMethodInvocation implements ProxyMethodInvocation, Cloneable {

    // 拦截器列表（包含了所有匹配的 Advice 转化而来的 MethodInterceptor）
    protected final List<?> interceptorsAndDynamicMethodMatchers;

    // ★ 当前拦截器在列表中的索引（初始为 -1）
    private int currentInterceptorIndex = -1;

    @Override
    @Nullable
    public Object proceed() throws Throwable {
        // ★★ arrive at the joinpoint ★★
        // 当 currentInterceptorIndex 达到拦截器列表的最后一个位置时
        // 说明所有拦截器都已执行完毕，调用目标方法
        if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
            return invokeJoinpoint();
        }

        // ★ 获取当前要执行的拦截器（currentInterceptorIndex 自增后获取）
        Object interceptorOrInterceptionAdvice =
                this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);

        // 处理动态切入点匹配的 InterceptorAndDynamicMethodMatcher 类型
        if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher dm) {
            // 在运行时动态评估切入点是否匹配
            if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
                // 匹配成功：调用拦截器的 invoke(this)，this 就是当前方法调用本身
                // ★ 关键设计：把 this 传给 invoke()，形成递归链
                return dm.interceptor.invoke(this);
            } else {
                // 不匹配：跳过此拦截器，递归调用 proceed() 继续下一个
                return proceed();
            }
        } else {
            // ★★★ 核心：调用拦截器的 invoke()，传入 this ★★★
            // 每个拦截器的内部会：
            //   1. 执行自己的逻辑（如 Before 通知的增强代码）
            //   2. 调用 invocation.proceed() 继续链上的下一个拦截器
            //   3. 执行自己的后续逻辑（如 AfterReturning 通知的增强代码）
            //
            // 这个设计是"递归责任链"模式 —— 每个拦截器都通过 proceed() 来驱动下一个
            return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
        }
    }

    // 调用目标方法本身
    @Nullable
    protected Object invokeJoinpoint() throws Throwable {
        return AopUtils.invokeJoinpointUsingReflection(this.target, this.method, this.arguments);
    }
}
```

### 9.5.1 各通知类型对应的 MethodInterceptor 实现

```
五种通知类型的注解分别由不同的 MethodInterceptor（也称 Advice）实现：

@Before          → MethodBeforeAdviceInterceptor
    invoke() 流程：
    ├── 1. 执行前置通知的逻辑（this.advice.before(method, args, target)）
    └── 2. 调用 mi.proceed() 驱动下一个拦截器

@AfterReturning  → AfterReturningAdviceInterceptor
    invoke() 流程：
    ├── 1. 调用 mi.proceed() 先驱动下一个拦截器（最终到目标方法）
    └── 2. 目标方法返回后，执行后置通知逻辑（this.advice.afterReturning(retVal, method, args, target)）

@AfterThrowing   → AspectJAfterThrowingAdvice
    invoke() 流程：
    ├── 1. try { mi.proceed(); }
    └── 2. catch (匹配的异常) { 执行异常通知逻辑 }

@After           → AspectJAfterAdvice
    invoke() 流程：
    ├── 1. try { mi.proceed(); }
    ├── 2. catch (Exception ex) { throw ex; }
    └── 3. finally { 执行最终通知逻辑（无论如何都会执行） }

@Around          → AspectJAroundAdvice
    invoke() 流程：
    ├── 1. 通过反射调用 @Around 标注的方法
    │      传入 ProceedingJoinPoint（包装了当前 ReflectiveMethodInvocation）
    └── 2. ProceedingJoinPoint.proceed() → 继续调用 mi.proceed()

ExposeInvocationInterceptor：固定排在第一个的拦截器
    作用：将当前 MethodInvocation 暴露到 ThreadLocal
    使得其他 Advised 可以通过 AopContext.currentProxy() 获取当前代理
```

## 9.6 执行链完整调用时序图

```
假设有一个方法被 @Around + @Before + @AfterReturning + @After 同时增强：

ReflectiveMethodInvocation.proceed()  —— currentInterceptorIndex = -1
    │
    ▼ currentInterceptorIndex = 0
MethodBeforeAdviceInterceptor.invoke(this)
    │  ├── advice.before()  ← 执行 @Before 通知
    │  └── mi.proceed()     ← 递归调用 proceed()
    │       │ currentInterceptorIndex = 1
    │       ▼
    │   AspectJAroundAdvice.invoke(this)
    │       │  ├── 反射调用 @Around 方法 → 前环绕逻辑
    │       │  ├── joinPoint.proceed()  ← ProceedingJoinPoint 继续驱动链
    │       │  │       │ currentInterceptorIndex = 2
    │       │  │       ▼
    │       │  │   (假设没有更多拦截器，到达目标方法)
    │       │  │       │
    │       │  │       ▼
    │       │  │   target.login()  ← ★ 执行目标方法 ★
    │       │  │       │
    │       │  │       │ currentInterceptorIndex = 2 (最后一个)
    │       │  │       ▼
    │       │  │   invokeJoinpoint() 返回
    │       │  │       │
    │       │  └── @Around 方法 → 后环绕逻辑
    │       │
    │       │ 继续返回
    │       ▼
    │   AfterReturningAdviceInterceptor.invoke(this)
    │       │  ├── mi.proceed() 已返回（上面已经执行完目标方法）
    │       │  └── advice.afterReturning(retVal)  ← 执行 @AfterReturning
    │       │
    │       │ 继续返回
    │       ▼
    │   AspectJAfterAdvice.invoke(this)
    │       │  └── finally { advice.after() }  ← 执行 @After（无论异常）
    │       │
    │       │ 继续返回
    │       ▼
    返回到第一个调用点 — 最终结果返回给外部调用者

关键理解：
  ★ @Around 的前逻辑在所有 @Before 之前执行
  ★ @Around 的后逻辑在所有 @AfterReturning/@After 之后执行
  ★ @Around 的前后逻辑像一个"鸡蛋壳"，包裹了中间的所有通知和实际方法
```

## 9.7 JdkDynamicAopProxy.invoke() 源码

```
JDK 动态代理的核心：InvocationHandler.invoke()

final class JdkDynamicAopProxy implements AopProxy, InvocationHandler, Serializable {

    @Override
    @Nullable
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Object oldProxy = null;
        boolean setProxyContext = false;

        TargetSource targetSource = this.advised.targetSource;
        Object target = null;

        try {
            // 处理 equals、hashCode 等基础方法
            if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
                return equals(args[0]);
            }
            if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
                return hashCode();
            }

            // 处理 DecoratingProxy、Advised 接口的方法
            Object retVal;

            if (this.advised.exposeProxy) {
                oldProxy = AopContext.setCurrentProxy(proxy);
                setProxyContext = true;
            }

            // 从 TargetSource 获取目标对象
            target = targetSource.getTarget();
            Class<?> targetClass = (target != null ? target.getClass() : null);

            // ★★★ 获取当前方法的"拦截器链" ★★★
            // 将 Advisor 列表中的 Advice 转换为 MethodInterceptor 列表
            List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(
                    method, targetClass);

            if (chain.isEmpty()) {
                // 没有拦截器，直接反射调用目标方法
                Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
                retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
            } else {
                // ★ 创建 ReflectiveMethodInvocation 并驱动拦截器链
                MethodInvocation invocation = new ReflectiveMethodInvocation(
                        proxy, target, method, args, targetClass, chain);
                retVal = invocation.proceed();  // ← 进入责任链
            }

            // 处理返回值为 this 的情况（返回代理对象而不是目标对象）
            Class<?> returnType = method.getReturnType();
            if (retVal != null && retVal == target &&
                    returnType != Object.class && returnType.isInstance(proxy) &&
                    !RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
                retVal = proxy;
            }

            return retVal;
        } finally {
            if (target != null && !targetSource.isStatic()) {
                targetSource.releaseTarget(target);
            }
            if (setProxyContext) {
                AopContext.setCurrentProxy(oldProxy);
            }
        }
    }
}
```

### 9.7.1 getInterceptorsAndDynamicInterceptionAdvice() — Advisor 与 MethodInterceptor 的转换

```
AdvisedSupport.getInterceptorsAndDynamicInterceptionAdvice():

public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method,
        @Nullable Class<?> targetClass) {
    MethodCacheKey cacheKey = new MethodCacheKey(method);
    List<Object> cached = this.methodCache.get(cacheKey);
    if (cached == null) {
        cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
                this, method, targetClass);
        this.methodCache.put(cacheKey, cached);
    }
    return cached;
}

DefaultAdvisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice():

public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
        Advised config, Method method, @Nullable Class<?> targetClass) {

    AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
    Advisor[] advisors = config.getAdvisors();
    List<Object> interceptorList = new ArrayList<>(advisors.length);

    for (Advisor advisor : advisors) {
        if (advisor instanceof PointcutAdvisor pointcutAdvisor) {
            // 判断切入点是否匹配当前方法
            if (pointcutAdvisor.getPointcut().getMethodMatcher().matches(method, targetClass)) {
                // 将 Advisor 中的 Advice 转换为 MethodInterceptor
                // 通过 AdvisorAdapter 注册表完成转换：
                //   MethodBeforeAdvice → MethodBeforeAdviceAdapter → MethodBeforeAdviceInterceptor
                //   AfterReturningAdvice → AfterReturningAdviceAdapter → AfterReturningAdviceInterceptor
                //   ThrowsAdvice → ThrowsAdviceAdapter → ThrowsAdviceInterceptor
                MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
                if (pointcutAdvisor.getPointcut().getMethodMatcher().isRuntime()) {
                    // 动态切入点：需要运行时再次判断
                    for (MethodInterceptor interceptor : interceptors) {
                        interceptorList.add(new InterceptorAndDynamicMethodMatcher(
                                interceptor, pointcutAdvisor.getPointcut().getMethodMatcher()));
                    }
                } else {
                    interceptorList.addAll(Arrays.asList(interceptors));
                }
            }
        } else if (advisor instanceof IntroductionAdvisor ia) {
            // 处理引介增强
            if (ia.getClassFilter().matches(targetClass)) {
                MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
                interceptorList.addAll(Arrays.asList(interceptors));
            }
        } else {
            // 无条件添加到拦截器链
            MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
            interceptorList.addAll(Arrays.asList(interceptors));
        }
    }
    return interceptorList;
}
```

## 9.8 CglibAopProxy 的代理创建与执行

```
CGLIB 代理使用 Enhancer 创建子类来代理目标对象：

class CglibAopProxy implements AopProxy, Serializable {

    @Override
    public Object getProxy(@Nullable ClassLoader classLoader) {
        // ... 构建 Enhancer
        Enhancer enhancer = createEnhancer();
        enhancer.setSuperclass(proxySuperClass);  // ★ 设置父类（目标类）
        enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
        enhancer.setCallbackFilter(proxyCallbackFilter);
        enhancer.setCallbackTypes(getCallbackTypes());
        // 通过 Objenesis 或反射创建代理实例
        return createProxyClassAndInstance(enhancer, callbacks);
    }

    // CGLIB 代理的回调类：DynamicAdvisedInterceptor（内部类）
    // 类似 JDK 的 InvocationHandler，在方法调用被拦截时触发
    private static class DynamicAdvisedInterceptor implements MethodInterceptor, Serializable {
        @Override
        @Nullable
        public Object intercept(Object proxy, Method method, Object[] args,
                MethodProxy methodProxy) throws Throwable {
            // ★ 内部逻辑与 JdkDynamicAopProxy.invoke() 几乎一致
            //   → 获取拦截器链
            //   → 创建 CglibMethodInvocation（类似 ReflectiveMethodInvocation）
            //   → 调用 invocation.proceed()
            //   CglibMethodInvocation 使用 methodProxy.invoke() 调用目标方法
            //   （CGLIB 的 methodProxy.invoke() 比反射调用快 3~5 倍）
        }
    }
}
```

## 9.9 @Aspect 切面类的通知转换过程

```
当一个 @Aspect 注解的 Bean 被扫描到后，每个通知方法都会被解析为一个 Advisor：

@Aspect
@Component
public class LogAspect {

    @Before("execution(* com.example.service.*.*(..))")
    public void beforeAdvice(JoinPoint joinPoint) { ... }
}
                        │
                        ▼
Spring 的 BeanFactoryAspectJAdvisorsBuilder.buildAspectJAdvisors()
                        │
                        ▼
ReflectiveAspectJAdvisorFactory.getAdvisors()
                        │
                        ▼
                                                                为每个 Advice 创建
InstantiationModelAwarePointcutAdvisorImpl
                        │
                        ▼
内部持有三个关键组件：
  ┌─ AspectJExpressionPointcut  ← 切点表达式（判断方法是否匹配）
  │     execution(* com.example.service.*.*(..))
  │
  ├─ AspectJMethodBeforeAdvice  ← 通知（实现了 MethodInterceptor）
  │    实现了 before() 方法
  │    通过 MethodInterceptorAdapter 模式桥接
  │
  └─ SimpleInstantiationModelAwarePointcutAdvisor  ← Advisor（切面 = 切点 + 通知）
```

## 9.10 设计思想总结

1. 双重代理策略模式（DefaultAopProxyFactory）：
   Spring 根据目标类是否实现接口，在 JDK 动态代理和 CGLIB 之间自动切换。
   调用方不需要关心使用了哪种代理技术。

2. 递归责任链模式（ReflectiveMethodInvocation.proceed()）：
   每一个 MethodInterceptor 在执行自己的增强后调用 mi.proceed() 来驱动链上的下一个。
   这是典型的 "Interceptor" 模式 —— 不同于传统的"链条逐个调用"，
   而是每个拦截器回环式地嵌套控制前后逻辑。

3. 适配器模式（AdvisorAdapter）：
   不同通知类型（MethodBeforeAdvice、AfterReturningAdvice、ThrowsAdvice）
   通过对应的 Adapter 统一转换为 MethodInterceptor，保证责任链的一致性。

4. 开放扩展原则：
   - 自定义 Advisor / Advice / Pointcut：可以通过实现对应接口扩展
   - 自定义 AopProxyFactory：可以替换代理创建策略
   - 自定义 MethodInterceptor：可以在责任链上插入任意逻辑
   - 所有这一切通过 BeanPostProcessor 机制实现，与 Spring IoC 无缝集成

5. @AspectJ 与 Spring AOP 的关系：
   Spring AOP 借助了 AspectJ 的切点表达式语法（通过 aspectjweaver 包），
   但运行时织入采用的是 Spring 自己的代理机制（不是 AspectJ 的编译/类加载时织入）。
   这意味着 Spring AOP 功能受限于方法级别的代理拦截，不支持字段拦截和构造器拦截。
