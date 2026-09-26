# 1. singleton与prototype
```
Spring默认情况下是如何管理这个Bean的：
默认情况下Bean是单例的。(单例：singleton)
在Spring上下文初始化的时候实例化。
每一次调用getBean()方法的时候，都返回那个单例的对象。

当将bean的scope属性设置为prototype：
bean是多例的。
spring上下文初始化的时候，并不会初始化这些prototype的bean。
每一次调用getBean()方法的时候，实例化该bean对象。
prototype翻译为：原型。

<!--
    目前来说：scope属性有两个值
        第一个值：singleton 单例（默认情况下就是单例的。）
        第二个值：prototype 原型/多例
-->
<bean id="sb" class="com.powernode.spring6.bean.SpringBean" scope="threadScope"></bean>

<!--
    目前来说：scope属性有两个值
        第一个值：singleton 单例（默认情况下就是单例的。）
        第二个值：prototype 原型/多例

    其实scope属性有多个值：
        例如：request session
        但是request session要求项目必须是一个web应用。
				当你引入springmvc框架的时候，这两个值就可以使用了。
        request：一次请求当中一个bean
        session：一次会话中只有一个bean
-->
<bean id="sb" class="com.powernode.spring6.bean.SpringBean" scope="session"></bean>

```

# 2. Bean的作用域类型
scope属性的值一共包括8个选项：
singleton：默认的，单例。
prototype：原型。每调用一次getBean()方法则获取一个新的Bean对象。或每次注入的时候都是新对象。
request：一个请求对应一个Bean。仅限于在WEB应用中使用。
session：一个会话对应一个Bean。仅限于在WEB应用中使用。
global session：portlet应用中专用的。如果在Servlet的WEB应用中使用global session的话，和session一个效果。（portlet和servlet都是规范。servlet运行在servlet容器中，例如Tomcat。portlet运行在portlet容器中。）
application：一个应用对应一个Bean。仅限于在WEB应用中使用。
websocket：一个websocket生命周期对应一个Bean。仅限于在WEB应用中使用。
自定义scope：很少使用。

# 3. 自定义作用域Scope
```
接下来自定义一个Scope，线程级别的Scope，在同一个线程中，获取的Bean都是同一个。跨线程则是不同的对象：
第一步：自定义Scope。（实现Scope接口：import org.springframework.beans.factory.config.Scope;）
spring内置了线程范围的类：org.springframework.context.support.SimpleThreadScope，可以直接用，我们不需要额外自己实现一个Scope。
第二步：将自定义的Scope注册到Spring容器中。
第三步：使用Scope。
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <!--配置我们自定义的作用域-->
    <!-- CustomScopeConfigurer用户自定义的范围配置器 -->
		<!-- 将自定义作用域注入该配置器中，以便可以使用 -->
    <bean class="org.springframework.beans.factory.config.CustomScopeConfigurer">
        <property name="scopes">
            <map>
                <!-- key为使用自定义范围时的名 -->
                <entry key="threadScope">
                    <!--这个Scope接口的实现类使用的是Spring框架内置的。也可以自定义。-->
                    <bean class="org.springframework.context.support.SimpleThreadScope"/>
                </entry>
            </map>
        </property>
    </bean>
    
    <!-- 第三步：使用Scope。 -->
    <bean id="sb" class="com.powernode.spring6.bean.SpringBean" scope="threadScope"></bean>

</beans>

补充：实现Scope接口
package me.aster.spring.util;

import org.springframework.beans.factory.ObjectFactory;
import org.springframework.beans.factory.config.Scope;

public class MyScope implements Scope {
    @Override
    public Object get(String name, ObjectFactory<?> objectFactory) {
        return null;
    }

    @Override
    public Object remove(String name) {
        return null;
    }

    @Override
    public void registerDestructionCallback(String name, Runnable callback) {

    }

    @Override
    public Object resolveContextualObject(String key) {
        return null;
    }

    @Override
    public String getConversationId() {
        return "";
    }
}

```

# 4. 源码解析：Scope 机制与 Bean 生命周期管理

## 4.1. Scope 接口定义

```
public interface Scope {

    /**
     * ★ 核心方法：从 Scope 中获取 Bean
     * @param name          bean 的名称
     * @param objectFactory 创建 bean 的工厂（当 scope 中不存在 bean 时调用）
     * @return bean 实例
     */
    Object get(String name, ObjectFactory<?> objectFactory);

    /**
     * 从 Scope 中移除 Bean（通常用于销毁时）
     */
    @Nullable
    Object remove(String name);

    /**
     * 注册销毁回调（Bean 被从 Scope 中移除时会调用）
     */
    void registerDestructionCallback(String name, Runnable callback);

    /**
     * 解析上下文对象（如 request scope 中的当前 request 对象）
     */
    @Nullable
    Object resolveContextualObject(String key);

    /**
     * 获取当前 Scope 的会话 ID（如 session id）
     */
    @Nullable
    String getConversationId();
}
```

## 4.2. Spring 内置的 Scope 实现

```
Singleton Scope — 单例（DefaultSingletonBeanRegistry 内部管理，不实现 Scope 接口）
  生命周期：容器启动时创建，容器关闭时销毁
  存储位置：DefaultSingletonBeanRegistry.singletonObjects (ConcurrentHashMap)

Prototype Scope — 原型（不实现 Scope 接口，直接在 doGetBean 中特殊处理）
  生命周期：每次 getBean() 都创建新实例
  Spring 不管理其完整生命周期，只负责创建并初始化，之后交给调用者

Request Scope — 请求级别（RequestScope 实现）
  生命周期：一次 HTTP 请求
  存储位置：RequestAttributes（通过 RequestContextHolder 获取）
  底层实际存储在 HttpServletRequest.setAttribute()

Session Scope — 会话级别（SessionScope 实现）
  生命周期：一次 HTTP 会话
  存储位置：HttpSession.setAttribute()

Application Scope — 应用级别（ServletContextScope 实现）
  生命周期：web 应用生命周期
  存储位置：ServletContext.setAttribute()

WebSocket Scope — WebSocket 级别（WebSocketScope 实现）
  生命周期：一个 WebSocket 连接
  存储位置：WebSocketSession.getAttributes()

SimpleThreadScope — 线程级别（Spring 内置的自定义 scope 示例）
  生命周期：同一个线程内
  存储位置：ThreadLocal<Map<String, Object>>
```

## 4.3. doGetBean() 中的 Scope 处理分支

```
AbstractBeanFactory.doGetBean() 中的 Scope 分支：

if (mbd.isSingleton()) {
    // ========== Singleton 路径 ==========
    sharedInstance = getSingleton(beanName, () -> {
        try {
            return createBean(beanName, mbd, args);
        } catch (BeansException ex) {
            destroySingleton(beanName);
            throw ex;
        }
    });
    beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
}
else if (mbd.isPrototype()) {
    // ========== Prototype 路径 ==========
    Object prototypeInstance = null;
    try {
        beforePrototypeCreation(beanName);  // 标记正在创建中
        prototypeInstance = createBean(beanName, mbd, args);
    } finally {
        afterPrototypeCreation(beanName);  // 移除创建中标记
    }
    beanInstance = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
}
else {
    // ========== 其他 Scope 路径（request / session / custom）==========
    String scopeName = mbd.getScope();
    Scope scope = this.scopes.get(scopeName);  // ★ 从 scopes Map 取 Scope 实例
    if (scope == null) {
        throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
    }
    try {
        // ★ 调用 Scope.get(name, objectFactory)
        //   如果是 request scope：
        //     RequestScope.get() → RequestContextHolder.getRequestAttributes()
        //       → request.getAttribute(name) → 有就返回
        //       → 没有就调用 objectFactory.getObject() 创建
        //       → 创建后 put 到 request.setAttribute()
        Object scopedInstance = scope.get(beanName, () -> {
            beforePrototypeCreation(beanName);
            try {
                return createBean(beanName, mbd, args);
            } finally {
                afterPrototypeCreation(beanName);
            }
        });
        beanInstance = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
    } catch (IllegalStateException ex) {
        throw new ScopeNotActiveException(beanName, scopeName, ex);
    }
}
```

## 4.4. SimpleThreadScope 源码

```
Spring 内置的 SimpleThreadScope 是自定义 Scope 的参考实现：

public class SimpleThreadScope implements Scope {

    private final ThreadLocal<Map<String, Object>> threadScope =
            new NamedThreadLocal<Map<String, Object>>("SimpleThreadScope") {
                @Override
                protected Map<String, Object> initialValue() {
                    return new HashMap<>();
                }
            };

    @Override
    public Object get(String name, ObjectFactory<?> objectFactory) {
        // ★ 从当前线程的 ThreadLocal Map 中取 Bean
        Map<String, Object> scope = this.threadScope.get();
        Object scopedObject = scope.get(name);
        if (scopedObject == null) {
            // ★ 如果当前线程还没有这个 Bean，调用 ObjectFactory 创建
            scopedObject = objectFactory.getObject();
            scope.put(name, scopedObject);
        }
        return scopedObject;
    }

    @Override
    @Nullable
    public Object remove(String name) {
        Map<String, Object> scope = this.threadScope.get();
        return scope.remove(name);
    }

    @Override
    public void registerDestructionCallback(String name, Runnable callback) {
        logger.warn("SimpleThreadScope does not support destruction callbacks. "
                + "Consider using RequestScope in a web environment.");
    }

    @Override
    @Nullable
    public Object resolveContextualObject(String key) {
        return null;
    }

    @Override
    @Nullable
    public String getConversationId() {
        return Thread.currentThread().getName();  // 线程名作为会话 ID
    }
}
```

## 4.5. Scope 注册机制

```
<bean class="org.springframework.beans.factory.config.CustomScopeConfigurer">
    <property name="scopes">
        <map>
            <entry key="threadScope">
                <bean class="org.springframework.context.support.SimpleThreadScope"/>
            </entry>
        </map>
    </property>
</bean>

CustomScopeConfigurer 实现了 BeanFactoryPostProcessor，在 refresh() 阶段执行：

public class CustomScopeConfigurer implements BeanFactoryPostProcessor {

    private Map<String, Object> scopes;

    public void setScopes(Map<String, Object> scopes) {
        this.scopes = scopes;
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory)
            throws BeansException {
        if (this.scopes != null) {
            this.scopes.forEach((scopeKey, scopeValue) -> {
                if (scopeValue instanceof Scope scope) {
                    // ★ 将自定义 Scope 注册到 BeanFactory 的 scopes Map 中
                    beanFactory.registerScope(scopeKey, scope);
                    // 内部调用：this.scopes.put(scopeKey, scope)
                } else if (scopeValue instanceof Class<?> scopeClass) {
                    // 如果是 Class，先实例化再注册
                    beanFactory.registerScope(scopeKey,
                            (Scope) BeanUtils.instantiateClass(scopeClass));
                }
            });
        }
    }
}

结合 doGetBean() 的调用：
  String scopeName = mbd.getScope();  // "threadScope"
  Scope scope = this.scopes.get(scopeName);  // ★ 从注册的 scopes Map 中取出
  Object scopedInstance = scope.get(beanName, objectFactory);
```

## 4.6. Bean 作用域选择策略与生命周期对比

```
┌─────────────┬──────────────┬──────────────┬──────────────┬────────────────────┐
│ 作用域      │ 创建时机      │ 存储位置      │ 销毁时机      │ Spring 管理生命周期 │
├─────────────┼──────────────┼──────────────┼──────────────┼────────────────────┤
│ singleton   │ 容器启动时    │ ConcurrentHM │ 容器关闭时    │ 完整管理 ★★★       │
│             │ (默认)       │              │              │                    │
├─────────────┼──────────────┼──────────────┼──────────────┼────────────────────┤
│ prototype   │ 每次getBean()│ 无存储        │ 不管理        │ 仅管理创建+初始化    │
│             │             │              │              │ 不管销毁 ☆          │
├─────────────┼──────────────┼──────────────┼──────────────┼────────────────────┤
│ request     │ 每次HTTP请求  │ Request      │ 请求结束时    │ 完整管理 ★★★       │
│             │ 首次getBean()│ .setAttribute│              │                    │
├─────────────┼──────────────┼──────────────┼──────────────┼────────────────────┤
│ session     │ 每次会话首次  │ Session      │ 会话过期/销毁│ 完整管理 ★★★       │
│             │ getBean()   │ .setAttribute│              │                    │
├─────────────┼──────────────┼──────────────┼──────────────┼────────────────────┤
│ threadScope │ 每线程首次    │ ThreadLocal  │ 线程结束时    │ 简单管理 ★★        │
│ (自定义)    │ getBean()   │ Map          │              │ 无销毁回调          │
└─────────────┴──────────────┴──────────────┴──────────────┴────────────────────┘
```

## 4.7. 设计思想总结

1. 策略模式（Scope 接口）：
   不同的 Scope 实现对应不同的 Bean 生命周期管理策略。
   Scope 接口的方法签名反映了一个通用的缓存模式：
   - get(name, factory): 先从缓存取，没有则工厂创建 + 缓存
   - remove(name): 从缓存移除
   - registerDestructionCallback(name, callback): 注册销毁回调

2. ObjectFactory 的延迟创建（Lazy Creation）：
   Scope.get() 接受 ObjectFactory 参数，由 Scope 实现决定何时创建 Bean。
   这实现了"按需创建"的语义——只有当 Scope 中确实没有这个 Bean 时才创建。

3. Scope 与 Web 容器集成：
   RequestContextHolder 是 Spring 与 Servlet 容器之间的桥梁：
   - 通过 RequestContextListener 在请求开始/结束时设置/清空 RequestAttributes
   - 通过 ThreadLocal 存储当前请求的 RequestAttributes
   - RequestScope 通过这个机制获取当前 Request 对象

4. 扩展性：
   Spring 的 Scope 机制是开放的——任何人都可以实现 Scope 接口并注册。
   例如：自定义"一天内单例"的 DailyScope、"十分钟内单例"的 TimedScope 等。
