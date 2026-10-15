# 1. 代理模式的理解
代理模式，就是自己做不了，需要别人来代理，代替自己来完成
	最终这个行为还是要发生，只不过不是由自己来完成，而是由别人代理完成，只是对于客户其他人来说感受不到
代理模式的作用：
    1）当一个对象需要受到保护时，可以考虑使用代理模式去完成某个行为
    2）需要给某个对象的功能进行增强时，可以考虑找一个代理进行增强
    3）A 对象和 B 对象无法直接进行交互时，也可以使用代理模式来解决
代理模式中的三大角色：
    目标对象：需要被代理的对象
    代理对象：代理目标对象的对象
    目标对象和代理对象的公共接口：目标对象和代理对象之间应该具有相同的行为
		为了使用户察觉不到是由代理对象完成的，使用户感觉还是由目标对象进行完成的
使用代理模式，对于客户端程序来说，客户端无法察觉，客户端在使用代理对象的时候，就像在使用目标对象。对于客户端程序来说，使用代理对象时就像在使用目标对象一样。
代理模式是GoF23种设计模式之一。属于结构型设计模式。
代理模式的作用是：为其他对象提供一种代理以控制对这个对象的访问。在某些情况下，一个客户不想或者不能直接引用一个对象，此时可以通过一个称之为“代理”的第三者来实现间接引用。代理对象可以在客户端和目标对象之间起到中介的作用，并且可以通过代理对象去掉客户不应该看到的内容和服务或者添加客户需要的额外服务。 通过引入一个新的对象来实现对真实对象的操作或者将新的对象作为真实对象的一个替身，这种实现机制即为代理模式，通过引入代理对象来间接访问一个对象，这就是代理模式的模式动机。
代理模式的代码实现有两种形式：
    静态代理
    动态代理

# 2. 静态代理
```
项目经理提出一个新的需求：要统计所有业务接口中每一个业务方法的耗时。
解决方案一：硬编码，在每一个业务接口中的每一个业务方法中直接添加统计耗时的程序。
这种方案的缺点：
缺点一：违背OCP开闭原则。
缺点二：代码没有得到复用，相同的代码写了很多遍。
解决方案二：编写业务类的子类，让子类继承业务类，对每个业务方法进行重写。
缺点一：虽然解决了OCP开闭原则。但是这种方式会导致耦合度很高，因为采用了继承关系。
继承关系是一种耦合度非常高的关系，不建议使用。
缺点二：代码没有得到复用，相同的代码写了很多遍。
解决方案三：代理模式。
优点1：解决了OCP问题。
优点2：采用代理模式的has a，可以降低耦合度。

公共接口
package cw.study.spring.service;

/**
 * ClassName: OrderService
 * Package: cw.study.spring.service
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-28 9:32
 * @Version 1.0
 */
public interface OrderService {
    
    /**
     * 生成订单
     */
    void generate();
    
    /**
     * 修改订单
     */
    void modify();
    
    /**
     * 查看订单详情
     */
    void detail();
    
}

代理对象
代理对象和目标对象要具有相同的行为，就需要实现同样的接口，使得客户端在使用代理对象的时候，就像在使用目标对象一样
在代理对象中最终也还是要执行目标对象中的目标方法，为了能够在代理对象中执行目标对象的目标方法，我们可以将目标对象作为代理对象的属性，代理对象中要有目标对象的引用，这种关系为关联关系，耦合度比泛化关系
    关联关系：在A中，有B作为其属性，A has a B
    泛化关系：A继承B，A is a B
package cw.study.spring.service;

/**
 * ClassName: OrderServiceProxy
 * Package: cw.study.spring.service
 * Description:
 * 代理对象
 *
 * @Author tcw
 * @Create 2023-05-29 10:25
 * @Version 1.0
 */
public class OrderServiceProxy implements OrderService {
    
    // 使用公共接口，公共接口的耦合度低
    private OrderService target;
    
    // 创建代理对象的时候，给代理对象中目标对象引用赋值
    public OrderServiceProxy(OrderService target) {
        this.target = target;
    }
    
    @Override
    public void generate() {
        // 增强代码
        long begin = System.currentTimeMillis();
        // 调用目标对象的目标方法
        target.generate();
        long end = System.currentTimeMillis();
        System.out.println("执行耗时：" + (end - begin));
    }
    
    @Override
    public void modify() {
        long begin = System.currentTimeMillis();
        // 调用目标对象的目标方法
        target.modify();
        long end = System.currentTimeMillis();
        System.out.println("执行耗时：" + (end - begin));
    }
    
    @Override
    public void detail() {
        long begin = System.currentTimeMillis();
        // 调用目标对象的目标方法
        target.detail();
        long end = System.currentTimeMillis();
        System.out.println("执行耗时：" + (end - begin));
    }
}

public class Client {
    public static void main(String[] args) {
        // 创建目标对象
        OrderService target = new OrderSeriveImpl();
        // 创建代理对象
        OrderService proxy = new OrderServiceProxy(target);
        // 使用代理对象的代理方法
        // 使用代理对象就像在使用目标对象一样，
        // 都可以完成相同的功能，并且还可以进行加强
        proxy.generate();
        proxy.modify();
        proxy.detail();
    }
}

使用静态代理模式，没有修改原先写好的类，符合OCP原则，且在代理类中只是有目标对象的引用，耦合度比前两种方法更低
目前我们使用的是静态代理，这个静态代理的缺点是什么？
	类爆炸。假设系统中有1000个接口，那么每个接口都需要对应代理类，这样类会急剧膨胀。不好维护。
怎么解决类爆炸问题？
	可以使用动态代理来解决这个问题。
动态代理还是代理模式，只不过添加了字节码生成技术，可以在内存中为我们动态的生成一个class字节码，这个字节码就是代理类。
在内存中动态的生成字节码代理类的技术，叫做：动态代理。

```

# 3. 动态代理
动态代理：在程序运行阶段，在内存中动态生成代理类，被称为动态代理，目的是为了减少代理类的数量。解决代码复用的问题。
在内存当中动态生成类的技术常见的包括：
    JDK动态代理技术：java.lang.reflect.Proxy，只能代理接口，即只适合有一个目标类和一个公共接口的情况
    CGLIB动态代理技术：CGLIB(Code Generation Library)是一个开源项目。是一个强大的，高性能，高质量的Code生成类库，它可以在运行期扩展Java类与实现Java接口。它既可以代理接口，又可以代理类，底层是通过继承的方式实现的。性能比JDK动态代理要好。（底层有一个小而快的字节码处理框架ASM。）
    	由于动态生成的类是在内存中生成的，可以采用继承的方式，无所谓其耦合度
    Javassist动态代理技术：Javassist是一个开源的分析、编辑和创建Java字节码的类库。是由东京工业大学的数学和计算机科学系的 Shigeru Chiba （千叶 滋）所创建的。它已加入了开放源代码JBoss 应用服务器项目，通过使用Javassist对字节码操作为JBoss实现动态"AOP"框架。
    
是实现接口还是继承形式的动态代理，对于后面使用AOP的使用还是有些许影响的（在getBean()方法中传入的Class<?>就会被这个影响）
## 3.1 JDK 动态代理技术
JDK 动态代理只能代理接口 
还是使用静态代理中的例子：一个接口和一个实现类。
### 3.1.1 公共接口
```
package cw.study.spring.service;

/**
 * ClassName: OrderService
 * Package: cw.study.spring.service
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-28 9:32
 * @Version 1.0
 */
public interface OrderService {
    
    String getName();
    
    /**
     * 生成订单
     */
    void generate();
    
    /**
     * 修改订单
     */
    void modify();
    
    /**
     * 查看订单详情
     */
    void detail();
    
}
```
### 3.1.2 目标对象
```
package cw.study.spring.service;


/**
 * ClassName: OrderSeriveImpl
 * Package: cw.study.spring.service
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-28 9:34
 * @Version 1.0
 */
public class OrderSeriveImpl implements OrderService {
    @Override
    public String getName() {
        return "张三";
    }
    
    @Override
    public void generate() {
        try {
            Thread.sleep(30);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("订单生成成功...");
    }
    
    @Override
    public void modify() {
        try {
            Thread.sleep(20);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("订单修改成功...");
    }
    
    @Override
    public void detail() {
        try {
            Thread.sleep(10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("订单查询成功...");
    }
}
```
### 3.1.3 动态生成代理类分析
```
在动态代理中代理类是可以动态生成的，这个类不需要写，我们直接写客户端程序即可
需要理解：Object proxyObj = Proxy.newProxyInstance(类加载器, 代理类要实现的接口(公共接口), 调用处理器)
	newProxyInstance：创建代理对象，调用该方法可以创建代理对象
		Proxy.newProxyInstance()：该方法的执行在内存中生成了代理类的字节码，并且通过内存中生成的代理类的字节码创建了代理对象
	Proxy.newProxyInstance() 的三个参数：
		类加载器 ClassLoader loader：内存中动态生成的类的字节码需要加载到JVM中，需要类加载器，JDK要求代理类的类加载器和目标类的类加载器要是同一个
		代理类要实现的接口 Class<?>[] interfaces：代理类要和目标类实现相同的接口，代理类需要实现的接口需要我们进行告知
		调用处理器 InvocationHandler h：JDK不可能知道我们要代理类增强目标类哪些功能，JDK不知道增强的代码，这个需要我们进行传递，我们可以通过调用处理器告诉JDK代理类的增强代码，调用处理器 InvocationHandler 是一个接口，需要我们进行实现，在需要实现的方法中编写增强代码
```
### 3.1.4 实现调用处理器接口 InvocationHandler 编写增强代码
```
    编写增强代码的调用处理器只需要编写一次即可
    实现调用处理器接口 InvocationHandler，需要实现调用处理器的invoke方法，实现该方法其实就是在实现代理类实现公共接口时需要实现公共接口的方法的方法体
    增强代码在调用处理器接口 InvocationHandler 的invoke方法中编写
    invoke方法由JDK在底层进行调用，如何调用invoke方法，JDK在底层已经写好了，当代理对象调用代理方法的时候，invoke方法会被调用

package client.cw.study.spring.improve;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

/**
 * ClassName: TimerInvocationHandler
 * Package: client.cw.study.spring.improve
 * Description:
 * 负责计时的调用处理器类
 * 在该类中编写关于计时的增强代码
 *
 * @Author tcw
 * @Create 2023-06-01 23:59
 * @Version 1.0
 */
public class TimerInvocationHandler implements InvocationHandler {
    
    // 目标对象
    private Object target;
    
    /**
     * 调用目标对象的目标方法是通过反射进行调用的，
     * 所以需要目标对象的引用
     *
     * @param target 目标对象
     */
    public TimerInvocationHandler(Object target) {
        this.target = target;
    }
    
    /**
     * 在invoke方法中写增强代码。
     * 调用代理对象的代理方法，对目标方法进行增强时要保证目标方法执行。
     * invoke方法是JDK进行调用的，JDK在调用该方法时，会将invoke方法需要的参数
     * 传递过来
     *
     * @param proxy 代理对象的引用
     * @param method 目标对象的目标方法
     * @param args 目标方法上的实参
     * @return 目标对象的目标方法的返回值
     * @throws Throwable
     */
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 调用目标对象的目标方法的前后编写增强代码
        // 目标方法执行开始时间
        long start = System.currentTimeMillis();
        // 调用目标对象的目标方法
        Object returnVal = method.invoke(target, args);
        // 目标方法执行结束时间
        long end = System.currentTimeMillis();
        // 计算输出目标方法的执行耗时
        System.out.println(end - start);
        // 返回目标对象的目标方法的返回值
        return returnVal;
    }
}
```
### 3.1.5 客户端程序
```
package client;

import client.cw.study.spring.improve.TimerInvocationHandler;
import cw.study.spring.service.OrderSeriveImpl;
import cw.study.spring.service.OrderService;
import cw.study.spring.service.OrderServiceProxy;

import java.lang.reflect.Proxy;

/**
 * ClassName: Client
 * Package: client
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-28 9:38
 * @Version 1.0
 */
public class Client {
    public static void main(String[] args) {
        // 创建目标对象
        OrderService target = new OrderSeriveImpl();
        // 创建代理对象
        // Object proxyObj = Proxy.newProxyInstance(类加载器, 代理类要实现的接口(公共接口), 调用处理器)
        OrderService orderService = (OrderService) Proxy.newProxyInstance(
                target.getClass().getClassLoader(),
                target.getClass().getInterfaces(),
                new TimerInvocationHandler(target)
        );
        // 使用代理对象的代理方法
        orderService.generate();
        orderService.modify();
        orderService.detail();
        String name = orderService.getName();
        System.out.println(name);
    }
}
```
### 3.1.6 JDK 动态代理工具类封装
```
package client.cw.study.spring.improve;

import cw.study.spring.service.OrderService;

import java.lang.reflect.Proxy;

/**
 * ClassName: ProxyUtil
 * Package: client.cw.study.spring.improve
 * Description:
 *
 * @Author tcw
 * @Create 2023-06-02 19:46
 * @Version 1.0
 */
public class ProxyUtil {
    
    private ProxyUtil() {}
    
    /**
     * 创建目标对象的代理对象
     *
     * @param target 目标对象
     * @return 代理对象
     */
    public static Object newProxyInstance(Object target) {
        return Proxy.newProxyInstance(
                target.getClass().getClassLoader(), // 获取目标对象的目标类的类加载器
                target.getClass().getInterfaces(), // 获取目标对象的目标类实现的接口
                new TimerInvocationHandler(target) // 调用处理器
        );
    }
    
}
public class Client {
    public static void main(String[] args) {
        // 创建目标对象
        OrderService target = new OrderSeriveImpl();
        // 创建代理对象
        // Object proxyObj = Proxy.newProxyInstance(类加载器, 代理类要实现的接口(公共接口), 调用处理器)
        // OrderService orderService = (OrderService) Proxy.newProxyInstance(
        //         target.getClass().getClassLoader(),
        //         target.getClass().getInterfaces(),
        //         new TimerInvocationHandler(target)
        // );
        OrderService orderService = (OrderService) ProxyUtil.newProxyInstance(target);
        // 使用代理对象的代理方法
        orderService.generate();
        orderService.modify();
        orderService.detail();
        String name = orderService.getName();
        System.out.println(name);
    }
}
```
## 3.2 CGLIB 动态代理
CGLIB 动态代理可以代理接口，也可以代理类
CGLIB 动态代理底层采用继承的方式实现。所以被代理的目标类不能使用final修饰。
### 3.2.1 CGLIB依赖
```
<dependency>
  <groupId>cglib</groupId>
  <artifactId>cglib</artifactId>
  <version>3.3.0</version>
</dependency>
```
### 3.2.2 目标类
```
package cw.study.spring.service;

/**
 * ClassName: UserService
 * Package: cw.study.spring.service
 * Description:
 * 目标类
 *
 * @Author tcw
 * @Create 2023-06-03 15:27
 * @Version 1.0
 */
public class UserService {
    
    public boolean login(String username, String password) {
        System.out.println("正在验证身份...");
        if ("admin".equals(username) && "123".equals(password)) return true;
        return false;
    }
    
    public void logout() {
        System.out.println("退出登录...");
    }
    
}

```
### 3.2.3 方法拦截器MethodInterceptor
```
相当于要执行目标方法的时候，会被拦截器拦截，在执行增强代码的过程中执行目标方法，实现对目标方法的增强
package client.cw.study.spring.improve;

import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

import java.lang.reflect.Method;

/**
 * ClassName: TimerMethodInterceptor
 * Package: client.cw.study.spring.improve
 * Description:
 *
 * @Author tcw
 * @Create 2023-06-03 15:35
 * @Version 1.0
 */
public class TimerMethodInterceptor implements MethodInterceptor {
    
    /**
     * 在该方法中编写对目标方法的增强代码
     * 
     * @param o 目标对象
     * @param method 目标方法
     * @param objects 目标方法调用时的实参
     * @param methodProxy 代理方法
     * @return 目标方法的返回值
     * @throws Throwable
     */
    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        // 调用目标前后编写增强代码
        long start = System.currentTimeMillis();
        // 调用目标对象的方法
        // 调用代理对象的父类的方法
        Object returnVal = methodProxy.invokeSuper(o, objects);
        long end = System.currentTimeMillis();
        System.out.println("耗时：" + (end - start));
        // 返回目标方法的返回值
        return returnVal;
    }
}

```
### 3.2.4 客户端
```
package client;


import client.cw.study.spring.improve.TimerMethodInterceptor;
import cw.study.spring.service.UserService;
import net.sf.cglib.proxy.Enhancer;

/**
 * ClassName: Client
 * Package: client
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-28 9:38
 * @Version 1.0
 */
public class Client {
    public static void main(String[] args) {
        // 创建字节码增强器对象，CGLIB的核心对象，用于代理类的生成
        Enhancer enhancer = new Enhancer();
        // 告诉CGLIB代理的目标类，由于CGLIB采用的是继承的方式，所以目标类为代理类的父类
        enhancer.setSuperclass(UserService.class);
        // 设置回调，等同于JDK动态代理的调用处理器
        // 在CGLIB中需要实现的接口为MethodInterceptor方法拦截器
        enhancer.setCallback(new TimerMethodInterceptor());
        // 创建代理对象
        // 会在内存中生成目标类的子类，即代理类，然后会创建代理类的对象
        UserService userService = (UserService) enhancer.create();
        // 使用代理对象的代理方法
        System.out.println(userService.login("admin", "123") ? "登录成功" : "登录失败");
        userService.logout();
    }
}

对于高版本的JDK，如果使用CGLIB，需要在启动项中添加两个启动参数：
--add-opens java.base/java.lang=ALL-UNNAMED
--add-opens java.base/sun.net.util=ALL-UNNAMED
```
### 3.2.5 CGLIB代理类名格式
```
cw.study.spring.service.UserService$$EnhancerByCGLIB$$d609db49@2d6a9952

class UserService$$EnhancerByCGLIB$$d609db49 extends UserService {} 
```

# 4. 源码视角：代理模式在 Spring AOP 中的实现

## 4.1 GoF 代理模式 ↔ Spring AOP 代理的映射

```
GoF 代理模式的三大角色在 Spring AOP 中的对应：

  GoF 代理模式                        Spring AOP
  ┌─────────────┐                  ┌──────────────────────┐
  │ Subject     │ ← 公共接口        │ 接口（如 UserService）│
  │ (抽象主题)   │                  │ 或被代理类本身        │
  └──────┬──────┘                  └──────────────────────┘
         │
  ┌──────┴──────┐
  │             │
  ▼             ▼
┌──────────┐ ┌──────────────┐
│RealSubject│ │ Proxy        │
│(真实主题) │ │ (代理)        │
└──────────┘ └──────────────┘
     ▲              ▲
     │              │
┌────────────┐ ┌──────────────────────────┐
│ 目标对象    │ │ JdkDynamicAopProxy        │
│ (Target)   │ │ 或 CglibAopProxy          │
│ @Service   │ │ + 拦截器链 (Advice Chain) │
└────────────┘ └──────────────────────────┘
```

## 4.2 静态代理 vs 动态代理 在 Spring 中的运用

```
手写静态代理：
  public class OrderServiceProxy implements OrderService {
      private OrderService target;  // 持有目标对象
      public void createOrder() {
          System.out.println("记录日志...");  // 增强
          target.createOrder();               // 代理调用目标
          System.out.println("提交事务...");  // 增强
      }
  }
  缺点：每个接口都需要写一个 Proxy 类 → 类爆炸

Spring JDK 动态代理（解决类爆炸）：
  OrderService proxy = (OrderService) Proxy.newProxyInstance(
      classLoader, new Class[]{OrderService.class},
      new InvocationHandler() {
          public Object invoke(Object proxy, Method method, Object[] args) {
              // ★ 同一个 InvocationHandler 处理所有方法
              // 通过反射 + Advice 列表实现通用增强
              return method.invoke(target, args);
          }
      });
  
  Spring 的 JdkDynamicAopProxy 就是这个 InvocationHandler 的实例化：
  class JdkDynamicAopProxy implements AopProxy, InvocationHandler {
      @Override
      public Object invoke(Object proxy, Method method, Object[] args) {
          // 获取匹配的拦截器链
          List<Object> chain = advised.getInterceptorsAndDynamicInterceptionAdvice(
                  method, targetClass);
          if (chain.isEmpty()) {
              return AopUtils.invokeJoinpointUsingReflection(target, method, args);
          }
          // ★ 创建 MethodInvocation 驱动拦截器链
          return new ReflectiveMethodInvocation(
                  proxy, target, method, args, targetClass, chain).proceed();
      }
  }
```

## 4.3 CGLIB 代理在 Spring 中的源码实现

```
代理类 UserService$$EnhancerByCGLIB$$d609db49 extends UserService

Spring 的 ObjenesisCglibAopProxy 创建 CGLIB 代理的核心：

class CglibAopProxy implements AopProxy {
    @Override
    public Object getProxy(@Nullable ClassLoader classLoader) {
        Enhancer enhancer = createEnhancer();
        enhancer.setSuperclass(this.advised.getTargetClass());  // ★ 继承目标类
        enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
        enhancer.setCallbackFilter(new CallbackFilter() {
            @Override
            public int accept(Method method) {
                // 根据方法是否匹配切点，返回不同的回调
                return 0; // 0 = AOP 拦截器, 1 = 直接调用目标
            }
        });
        enhancer.setCallbacks(new Callback[]{
            new DynamicAdvisedInterceptor(this.advised),  // ★ AOP 拦截器
            new StaticUnadvisedInterceptor(target)         //   无增强方法
        });
        return enhancer.create();  // 创建子类实例
    }
}
```

## 4.4 代理模式总结

1. GoF 代理模式的核心意图：控制访问 + 增强功能
   在 Spring AOP 中表现为：事务控制、日志记录、安全校验等横切关注点

2. 静态代理 → JDK 动态代理 → CGLIB 动态代理 是一步步解决"类爆炸"问题的演进：
   静态代理：手动为每个类写代理 → O(n) 个代理类
   JDK 动态代理：运行时生成接口代理 → 1 个 InvocationHandler 处理所有接口
   CGLIB 动态代理：运行时生成子类代理 → 1 个 MethodInterceptor 处理所有类

3. Spring 的选择策略（DefaultAopProxyFactory）：
   - 目标类实现了接口 → 优先 JDK 动态代理（轻量，Java 内置）
   - 目标类没有实现接口 → 使用 CGLIB 动态代理（通过继承）
   - proxyTargetClass=true → 强制使用 CGLIB（如 @EnableAspectJAutoProxy(proxyTargetClass=true)）
