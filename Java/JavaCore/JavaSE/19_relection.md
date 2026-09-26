# 1.反射:反射允许对成员变量、成员方法和构造方法的信息进行编程访问

```
反射（Reflection）是 Java 语言的动态特性，允许程序在运行时获取任意类的内部信息，
并操作其属性、方法和构造方法。

应用场景：
  - 框架开发（Spring、MyBatis 等大量使用反射）
  - 动态代理
  - 注解处理（APT）
  - IDE 自动补全和代码提示
  - 单元测试框架（JUnit）

优缺点：
  优点：灵活、动态、可扩展
  缺点：性能开销、安全限制（可能绕过访问控制）、代码可读性降低
```

## 1.1获取Class对象（Class是个类）

```java
1.1.1 Class.forName("<full_class_name>")
    适用于已知类全限定名，常用于加载驱动、读取配置文件等。
    需要处理 ClassNotFoundException。
    示例：Class<?> clazz = Class.forName("java.lang.String");

1.1.2 <class_name>.class
    适用于已知类名，无需实例化。
    编译时就确定，不会抛异常。
    每个类型（包括基本类型）都有一个 class 属性。
    示例：Class<String> clazz = String.class;

1.1.3 <class_obj>.getClass()
    适用于已有对象实例。
    示例：String s = "hello"; Class<?> clazz = s.getClass();

1.1.4 三种方式获取的 Class 对象是同一个（单例）
    String s = "hello";
    Class<?> c1 = Class.forName("java.lang.String");
    Class<?> c2 = String.class;
    Class<?> c3 = s.getClass();
    System.out.println(c1 == c2);  // true
    System.out.println(c2 == c3);  // true
```

## 1.2Class 对象的常用方法

```
1.2.1 基本信息方法
  String getName()                // 获取全限定类名（含包名）
  String getSimpleName()          // 获取简单类名（不含包名）
  String getPackageName()         // JDK9+，获取包名
  Package getPackage()            // 获取 Package 对象
  Class<? super T> getSuperclass() // 获取父类 Class 对象

1.2.2 判断方法
  boolean isInterface()           // 是否为接口
  boolean isArray()               // 是否为数组
  boolean isPrimitive()           // 是否为基本数据类型
  boolean isEnum()                // 是否为枚举
  boolean isAnnotation()          // 是否为注解类型
  boolean isRecord()              // JDK16+，是否为 Record

1.2.3 获取接口/父类
  Class<?>[] getInterfaces()      // 获取实现的接口
  Type[] getGenericInterfaces()   // 获取带泛型的接口

1.2.4 获取修饰符
  int getModifiers()              // 获取修饰符编码
  String toGenericString()        // 获取带泛型的类声明字符串

1.2.5 创建对象
  T newInstance()                 // JDK9 已过时，使用 clazz.getDeclaredConstructor().newInstance()
                                   // 调用无参构造，要求无参构造可访问

1.2.6 获取注解
  Annotation[] getAnnotations()   // 获取所有注解
  <A extends Annotation> A getAnnotation(Class<A> annotationClass)  // 获取指定注解

1.2.7 获取类加载器
  ClassLoader getClassLoader()    // 获取类加载器
```

## 1.3获取内容（Class对象的方法）

### 1.3.1构造方法Constructor

#### 1.3.1.1获取Constructor对象

```java
Constructor<?>[] getConstructors()    
    // 获取所有 public 构造方法（含继承的？构造方法不继承，此处包含本类的）

Constructor<?>[] getDeclaredConstructors()    
    // 获取所有声明的构造方法（包括 private、protected、default）

Constructor<T> getConstructor(Class<?>... parameterTypes)    
    // 根据参数类型获取指定的 public 构造方法
    
Constructor<T> getDeclaredConstructor(Class<?>... parameterTypes)    
    // 根据参数类型获取指定的声明构造方法（包括 private）

示例：
  Class<Student> clazz = Student.class;
  Constructor<Student> cons = clazz.getDeclaredConstructor(String.class, int.class);
```

#### 1.3.1.2Constructor中的方法

```java
T newInstance(Object... initargs)
    // 根据指定的构造方法创建对象，参数对应构造方法的参数
    // 私有构造方法需先调用 cons.setAccessible(true)

String getName()                    // 获取构造方法名（等于类名）
int getParameterCount()             // 获取参数个数
Class<?>[] getParameterTypes()      // 获取参数类型数组
int getModifiers()                  // 获取修饰符
public void setAccessible(boolean flag)
    // 设置为 true 表示取消访问检查，可以访问 private 构造方法
    // 这也称为"暴力反射"

示例：
  // 通过反射调用 private 构造创建对象
  Constructor<Student> cons = clazz.getDeclaredConstructor(String.class);
  cons.setAccessible(true);  // 取消权限检查
  Student stu = cons.newInstance("张三");
```

### 1.3.2字段（成员变量）Field

#### 1.3.2.1获取Field对象

```java
Field[] getFields()
    // 获取所有 public 字段（包括继承的 public 字段）

Field[] getDeclaredFields()
    // 获取所有声明的字段（不包括继承的字段，包括 private）

Field getField(String name)
    // 根据名称获取指定的 public 字段

Field getDeclaredField(String name)
    // 根据名称获取指定的声明字段（包括 private）
```

#### 1.3.2.2Field中的方法

```java
void set(Object obj, Object value)
    // 设置 obj 对象的该 Field 值为 value
    // 静态字段 obj 可以传 null

Object get(Object obj)
    // 获取 obj 对象的该 Field 值
    // 静态字段 obj 可以传 null

String getName()                    // 获取字段名
Class<?> getType()                  // 获取字段类型
int getModifiers()                  // 获取修饰符
public void setAccessible(boolean flag)
    // 设置为 true 表示取消访问检查，可以访问 private 字段

示例：
  Class<Student> clazz = Student.class;
  Student stu = new Student();

  Field nameField = clazz.getDeclaredField("name");
  nameField.setAccessible(true);      // 私有字段需要暴力反射
  nameField.set(stu, "李四");
  
  Field nameField2 = clazz.getDeclaredField("name");
  String name = (String) nameField2.get(stu);  // "李四"
```

### 1.3.3成员方法Method

#### 1.3.3.1获取Method对象

```java
Method[] getMethods()
    // 获取所有 public 成员方法（包括继承的方法，包括 Object 中的方法）

Method[] getDeclaredMethods()
    // 获取所有声明的成员方法（不包括继承的方法，包括 private）

Method getMethod(String name, Class<?>... parameterTypes)
    // 根据名称和参数类型获取指定的 public 成员方法

Method getDeclaredMethod(String name, Class<?>... parameterTypes)
    // 根据名称和参数类型获取指定的声明成员方法（包括 private）
```

#### 1.3.3.2Method中的方法

```java
Object invoke(Object obj, Object... args)
    // 参数一：用 obj 对象调用该方法（静态方法可传 null）
    // 参数二：调用方法的参数
    // 返回值：方法的返回值（void 方法返回 null）

String getName()                    // 获取方法名
Class<?> getReturnType()            // 获取返回值类型
Class<?>[] getParameterTypes()      // 获取参数类型数组
int getParameterCount()             // 获取参数个数
int getModifiers()                  // 获取修饰符
public void setAccessible(boolean flag)
    // 设置为 true 表示取消访问检查

示例：
  Class<Student> clazz = Student.class;
  Student stu = new Student();

  Method setAge = clazz.getDeclaredMethod("setAge", int.class);
  setAge.invoke(stu, 25);

  Method getAge = clazz.getDeclaredMethod("getAge");
  int age = (int) getAge.invoke(stu);  // 25

  Method staticMethod = clazz.getDeclaredMethod("staticMethod");
  staticMethod.invoke(null);  // 静态方法传 null
```

# 2.动态代理

```
动态代理：在程序运行时，通过反射机制动态地创建代理类及其对象，
通常用于在不修改目标对象代码的情况下，增加额外的功能（如日志、事务、权限校验等）。

Java 动态代理两种方式：
  2.1 JDK 动态代理（基于接口）：java.lang.reflect.Proxy
  2.2 CGLIB 动态代理（基于继承，第三方库）
```

## 2.1JDK 动态代理（基于接口）

```
核心类：java.lang.reflect.Proxy
核心接口：java.lang.reflect.InvocationHandler

步骤：
  1. 定义目标接口和实现类
  2. 编写 InvocationHandler 实现，在 invoke() 方法中添加增强逻辑
  3. 使用 Proxy.newProxyInstance() 创建代理对象

完整示例：

  // 1. 目标接口
  public interface UserService {
      void addUser(String name);
      String getUser(int id);
  }

  // 2. 目标实现类
  public class UserServiceImpl implements UserService {
      @Override
      public void addUser(String name) {
          System.out.println("添加用户: " + name);
      }
      @Override
      public String getUser(int id) {
          return "User" + id;
      }
  }

  // 3. InvocationHandler
  public class LoggingHandler implements InvocationHandler {
      private final Object target;  // 目标对象

      public LoggingHandler(Object target) {
          this.target = target;
      }

      @Override
      public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
          // 前置增强
          System.out.println("[日志] 开始调用方法: " + method.getName());
          long start = System.currentTimeMillis();

          // 调用目标方法
          Object result = method.invoke(target, args);

          // 后置增强
          long end = System.currentTimeMillis();
          System.out.println("[日志] 方法执行完毕，耗时: " + (end - start) + "ms");

          return result;
      }
  }

  // 4. 使用代理
  UserService target = new UserServiceImpl();
  UserService proxy = (UserService) Proxy.newProxyInstance(
      target.getClass().getClassLoader(),    // 类加载器
      target.getClass().getInterfaces(),     // 目标接口数组
      new LoggingHandler(target)             // InvocationHandler
  );

  proxy.addUser("张三");  // 自动带日志输出
  String user = proxy.getUser(1);
```

## 2.2Proxy.newProxyInstance 详解

```
public static Object newProxyInstance(
    ClassLoader loader,     // 类加载器：用于加载动态生成的代理类
    Class<?>[] interfaces,  // 目标实现的所有接口（代理对象将实现这些接口）
    InvocationHandler h     // 调用处理器：所有方法调用都会被转发到此
)

代理对象特点：
  - 代理对象与目标对象实现了相同的接口
  - 所有对代理对象的方法调用都会由 InvocationHandler.invoke() 处理
  - proxy 参数是代理对象本身，method 是被调用的方法，args 是方法参数
```

# 3.反射与配置文件结合示例

```
// 配置文件 app.properties:
// className=com.example.service.UserServiceImpl
// methodName=addUser

// 读取配置并通过反射调用
Properties props = new Properties();
props.load(new FileReader("app.properties"));

String className = props.getProperty("className");
String methodName = props.getProperty("methodName");

Class<?> clazz = Class.forName(className);
Constructor<?> cons = clazz.getDeclaredConstructor();
Object obj = cons.newInstance();

Method method = clazz.getDeclaredMethod(methodName, String.class);
method.invoke(obj, "测试用户");
```
