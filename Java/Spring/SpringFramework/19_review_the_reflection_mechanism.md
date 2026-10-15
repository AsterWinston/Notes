# 1. 调用方法的四要素
调用方法的四要素
第一要素：使用哪个对象进行方法的调用
第二要素：调用对象的哪个方法
第三要素：调用方法的时候，需要传递什么参数
第四要素：方法执行结束之后的返回结果
总结：调用哪个对象的哪个方法，需要传递什么参数，该方法返回什么值
即使是使用反射机制进行方法的调用，也同样需要具备着四要素


# 2. 反射机制调用方法
```
/**
 * ClassName: SomeService
 * Package: PACKAGE_NAME
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-27 10:46
 * @Version 1.0
 */
public class SomeService {
    public void doSome() {
        System.out.println(" public void doSome() 执行");
    }
    
    public String doSome(String s) {
        System.out.println(" public void doSome(String s) 执行");
        return s;
    }
    
    public String doSome(String s, Integer i) {
        System.out.println(" public void doSome(String s, Integer i) 执行");
        return s + " " + i;
    }
}

import java.lang.reflect.Method;

/**
 * ClassName: Main
 * Package: PACKAGE_NAME
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-27 10:45
 * @Version 1.0
 */
public class Main {
    public static void main(String[] args) throws Exception {
        // 加载类，获取类对应的Class对象
        Class<?> someServiceClass = Class.forName("SomeService");
        
        // 创建对象
        // 调用类的无参构造方法创建对象
        SomeService someService = (SomeService) someServiceClass.getDeclaredConstructor().newInstance();
    
        // 获取方法 public String doSome(String s, Integer i)
        // 参数一：获取方法的方法名
        // 参数二：可变参数，获取方法的形参列表类型
        Method doSomeMethod = someServiceClass.getDeclaredMethod("doSome", String.class, Integer.class);
        
        // 调用方法（四要素：调用哪个对象的哪个方法，需要传递什么参数，该方法返回什么值）
        // 参数一：调用哪个对象的该方法  someService
        // 参数二：可变参数，调用方法需要传递的形参  "Hello World", 111
        // invoke 的返回值就是调用方法的返回值  Object res
        Object res = doSomeMethod.invoke(someService, "Hello World", 111);
        System.out.println(res);
    }
}

```

# 3. Spring DI（Dependency I）核心实现
```
需求：假设你现在已知以下信息：
1.有这样一个类，类名叫做：包名.User
2.这个类符合javabean规范。属性私有化，对外提供公开的setter和getter方法。
3.你还知道这个类当中有一个属性，属性的名字叫做 age
4.并且你还知道age属性的类型是int类型。
请使用反射机制调用set方法，给User对象的age属性赋值。
package cw.study.spring.pojo;

/**
 * ClassName: User
 * Package: cw.study.spring.pojo
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-28 9:59
 * @Version 1.0
 */
public class User {
    private String name;
    private int age;
    
    public User() {
    }
    
    public User(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    @Override
    public String toString() {
        return "User{" + "name='" + name + '\'' + ", age=" + age + '}';
    }
    
    public String getName() {
        return name;
    }
    
    public void setName(String name) {
        this.name = name;
    }
    
    public int getAge() {
        return age;
    }
    
    public void setAge(int age) {
        this.age = age;
    }
}

import cw.study.spring.pojo.User;

import java.lang.reflect.Field;
import java.lang.reflect.Method;

/**
 * ClassName: Main
 * Package: PACKAGE_NAME
 * Description:
 *
 * @Author tcw
 * @Create 2023-05-27 10:45
 * @Version 1.0
 */
public class Main {
    public static void main(String[] args) throws Exception {
        // 1.有这样一个类，类名叫做：包名.User
        String className = "cw.study.spring.pojo.User";
        // 2.这个类符合javabean规范。属性私有化，对外提供公开的setter和getter方法。
        // 3.你还知道这个类当中有一个属性，属性的名字叫做 age
        String fieldName = "age";
        // 4.并且你还知道age属性的类型是int类型。
        // 请使用反射机制调用set方法，给User对象的age属性赋值。
        
        // 加载类
        Class<?> userClass = Class.forName(className);
        
        // 创建对象
        User user = (User) userClass.getDeclaredConstructor().newInstance();
        
        // 获取属性
        Field field = userClass.getDeclaredField(fieldName);
        // 获取属性的类型
        Class<?> fieldType = field.getType();
    
        // 获取方法名
        String methodName = "set" + fieldName.toUpperCase().charAt(0) + fieldName.substring(1);
        // 获取方法
        Method method = userClass.getDeclaredMethod(methodName, fieldType);
        
        // 调用方法
        method.invoke(user, 22);
    
        System.out.println(user);
    }
}

```

# 4. 源码视角：Spring 如何使用反射

## 4.1 BeanUtils.instantiateClass() — 构造方法反射

```
Spring 中实例化 Bean 的核心反射工具：

// BeanUtils.instantiateClass() 的简化源码
public static <T> T instantiateClass(Constructor<T> ctor, Object... args)
        throws BeanInstantiationException {
    try {
        ReflectionUtils.makeAccessible(ctor);    // 绕过访问控制
        return ctor.newInstance(args);           // ★ 反射调用构造方法
    } catch (InstantiationException ex) {
        throw new BeanInstantiationException(ctor, "Is it an abstract class?", ex);
    } catch (IllegalAccessException ex) {
        throw new BeanInstantiationException(ctor, "Is the constructor accessible?", ex);
    } catch (IllegalArgumentException ex) {
        throw new BeanInstantiationException(ctor, "Illegal arguments", ex);
    } catch (InvocationTargetException ex) {
        throw new BeanInstantiationException(ctor, "Constructor threw exception",
                ex.getTargetException());
    }
}

// 调用示例（在 AbstractAutowireCapableBeanFactory 中）：
Constructor<?> constructor = beanClass.getDeclaredConstructor();
Object bean = BeanUtils.instantiateClass(constructor);
// 等价于：Object bean = constructor.newInstance();
```

## 4.2 BeanWrapperImpl — setter 方法反射

```
Spring 中属性注入的核心反射：

// BeanWrapperImpl.setPropertyValue() 的核心：
PropertyHandler ph = getLocalPropertyHandler("name");
// PropertyHandler 封装了 PropertyDescriptor

Method writeMethod = ph.getWriteMethod();  // getWriteMethod() = setName()
ReflectionUtils.makeAccessible(writeMethod);
writeMethod.invoke(bean, "张三");  // ★ 反射调用 setter
// 等价于：bean.setName("张三");
```

## 4.3 ReflectionUtils — Spring 对 Java 反射的封装

```
Spring 在 java.lang.reflect 基础上封装了 ReflectionUtils：

public abstract class ReflectionUtils {

    // ★ 使方法可访问（绕过 private/protected 限制）
    public static void makeAccessible(Method method) {
        if (!Modifier.isPublic(method.getModifiers()) ||
                !Modifier.isPublic(method.getDeclaringClass().getModifiers())) {
            method.setAccessible(true);
        }
    }

    // ★ 处理反射调用的异常转换
    public static Object invokeMethod(Method method, Object target, Object... args) {
        try {
            return method.invoke(target, args);
        } catch (Exception ex) {
            handleReflectionException(ex);
            throw new IllegalStateException("Should never get here");
        }
    }

    // ★ 带回调的字段遍历
    public static void doWithFields(Class<?> clazz, FieldCallback fc) {
        // 遍历当前类及所有父类的字段
        for (Class<?> targetClass = clazz; targetClass != null; targetClass = targetClass.getSuperclass()) {
            Field[] fields = targetClass.getDeclaredFields();
            for (Field field : fields) {
                fc.doWith(field);  // 回调处理每个字段
            }
        }
    }
}
```

## 4.4 反射在 Spring 中的实际应用场景

```
1. Bean 实例化：
   Constructor.newInstance() → 创建 Bean 对象
   位置：AbstractAutowireCapableBeanFactory.createBeanInstance()
         → SimpleInstantiationStrategy.instantiate()
         → BeanUtils.instantiateClass(ctor)

2. 属性注入：
   Method.invoke() → 调用 setter 方法赋值
   位置：BeanWrapperImpl.setPropertyValue()
         → PropertyHandler.setValue()
         → writeMethod.invoke(bean, value)

3. @Autowired 字段注入：
   Field.set() → 直接设置字段值
   位置：AutowiredFieldElement.inject()
         → field.set(bean, value)

4. Bean 初始化：
   Method.invoke() → 调用 init-method
   位置：AbstractAutowireCapableBeanFactory.invokeCustomInitMethod()

5. @PostConstruct 处理：
   Method.invoke() → 调用 @PostConstruct 标注的方法
   位置：CommonAnnotationBeanPostProcessor
         → InitDestroyAnnotationBeanPostProcessor

6. @Transactional AOP 拦截：
   Method.invoke() → 调用 AOP 拦截器链
   位置：ReflectiveMethodInvocation.proceed()
         → AopUtils.invokeJoinpointUsingReflection()
```
