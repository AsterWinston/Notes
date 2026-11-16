反射（Reflection）是 Java 从"静态编译语言"伸向"运行时动态能力"的那只手：它让程序在**运行期**才知道要操作哪个类、调哪个方法、改哪个字段。没有反射就没有 Spring 的 IOC、没有 MyBatis 的映射、没有 JUnit 的测试方法发现，也没有代理与 AOP。本篇讲清**反射的入口 `Class` 对象怎么来、构造方法/字段/方法怎么操作、动态代理怎么把增强织进调用链**，最后用"配置文件 + 反射"手写一个迷你容器，看清 Spring 的雏形。

# 1. 反射是什么，为什么需要它

## 1.1 定义

**反射（Reflection）**是 Java 语言提供的动态特性：程序在运行时可以获取任意类的内部信息（构造方法、成员变量、成员方法、注解、修饰符），并且能够**创建对象、读写字段、调用方法**——哪怕这些成员是私有的、哪怕这个类的名字是运行时才从配置文件里读出来的。

正常情况下我们写 `new UserService().addUser("张三")`，这些信息在**编译期**就已经确定并且被写进字节码：编译器知道类是 `UserService`、方法是 `addUser`、参数是 `String`，还会顺手做类型检查。反射把这一切推迟到**运行期**，代价是丢掉编译期检查。

## 1.2 为什么需要：从"编译期决定"到"运行期决定"

反射存在的根本原因，是有些场景**在写代码的时候根本不知道要用哪个类**：

| 场景 | 编译期知道什么 | 运行期才知道什么 |
|------|---------------|-----------------|
| JDBC 连接数据库 | 只知道有 `Driver` 接口 | 具体是 MySQL 还是 PostgreSQL 的驱动类（写在配置里） |
| Spring 容器 | 只知道要管理 Bean | 要实例化哪些类、往哪些字段注入、调哪些回调（来自扫描或配置） |
| MyBatis | 只知道有 `UserMapper` 接口 | 实现类根本不存在，是靠动态代理在运行时造出来的 |
| JUnit | 只知道要跑测试 | 哪些方法带 `@Test`、怎么实例化测试类、怎么调它 |
| 序列化框架（Jackson/Fastjson） | 只知道有 `Object` | 对象的字段名与 JSON key 的对应关系 |

一句话概括：**框架是"给别人写代码的代码"，它必须在不知道业务类名的前提下操作业务类**，所以只能靠反射和动态代理。

## 1.3 代价：反射不是免费的

| 代价 | 具体表现 |
|------|----------|
| **性能开销** | 方法调用要装箱参数、做访问检查、JIT 难以内联，比直接调用慢数倍（见第 6 节） |
| **编译期检查消失** | 类名/方法名写错只在运行时报错（`ClassNotFoundException` / `NoSuchMethodException`），IDE 也帮不上忙 |
| **封装被打破** | `setAccessible(true)` 可以读写 private 成员，直接违反面向对象的封装原则 |
| **可读性下降** | 看代码看不出到底调了什么，只看到一串 `getMethod("xxx")` 字符串 |
| **安全与稳定性风险** | 框架可以改任意对象内部状态；JDK9 之后模块系统专门收紧了这种能力（见第 7 节） |

> **行业实践**：**业务代码几乎不该用反射**，它是给框架、工具、中间件用的。业务代码里如果发现自己在写 `Class.forName`，先想想能不能用接口 + 多态、`Map` 表驱动或函数式接口代替。

# 2. Class 对象：反射的入口

## 2.1 类加载与 Class 对象的关系

要理解反射，先要知道 `Class` 对象是哪来的。JVM 把 `.class` 字节码文件通过**类加载器（ClassLoader）**读进内存，经历"加载 → 链接（验证、准备、解析）→ 初始化"三个阶段；在**加载**阶段，JVM 会在**堆中创建一个 `java.lang.Class` 对象**，作为访问方法区里那份类结构信息的入口。

```text
   Hello.class（磁盘上的字节码文件）
         │  ① 加载：类加载器读取字节流，在方法区构造出类的结构信息
         ▼
   ┌───────────────────────────────────┐
   │  方法区：类的结构信息               │
   │  （字段表、方法表、常量池）          │
   └───────────────┬───────────────────┘
                   │  ② 同时在堆中创建一个 java.lang.Class 对象
                   ▼
   ┌───────────────────────────────────┐
   │  堆：Class 对象                    │
   │  同一个类 + 同一个类加载器          │
   │  = 全局唯一的一份 Class 对象        │
   └───────────────────────────────────┘
                   │
                   ▼  ③ 链接（验证 / 准备 / 解析） → ④ 初始化（执行静态代码块）
```

由此得到两个非常重要的结论：

1. **`Class` 对象是类加载的产物**，它不是"类的描述文字"，而是 JVM 里那份类型元数据的**唯一访问句柄**。拿到它，反射才有支点。
2. **同一个类在同一个类加载器下只会有一个 `Class` 对象**（这正是"三种获取方式拿到的是同一个对象"的原因，`==` 比较成立）。反过来，**同一个类被两个不同的类加载器加载，会得到两个不同的 `Class` 对象**，它们互不兼容、不能互相赋值——这正是 Tomcat 里多个 Web 应用加载同名类、热部署时 `ClassCastException` 的根源。

## 2.2 获取 Class 对象的三种方式

### 2.2.1 方式一：`Class.forName("全限定名")`

以字符串形式给出类的全限定名，返回 `Class` 对象。适用于**类名来自配置文件、字符串或注解**的场景，也是 JDBC 驱动加载的经典写法。

```java
Class<?> clazz = Class.forName("java.lang.String");
```

它会抛**受检异常** `ClassNotFoundException`，必须处理。另外它默认会**触发类的初始化**（也就是会执行静态代码块），这在加载 JDBC 驱动时正是我们想要的——驱动类的静态块会把自己注册到 `DriverManager`。

### 2.2.2 方式二：`类名.class`

直接写类字面量，**编译期就确定**，不需要实例化对象。

```java
Class<String> clazz = String.class;
```

它不会抛异常，**也不会触发类初始化**——类字面量在《Java 虚拟机规范》里属于"被动引用"，拿到的只是类加载阶段产出的那个 `Class` 对象。所有类型都有这个属性，包括基本类型、数组和 void：

```java
Class<Integer> c1 = int.class;          // 基本类型
Class<int[]>    c2 = int[].class;       // 数组类型
Class<Void>     c3 = void.class;        // void
System.out.println(int.class == Integer.TYPE);   // true
```

### 2.2.3 方式三：`对象.getClass()`

从已有实例出发，继承自 `Object`，是所有对象都能调用的方法。

```java
String s = "hello";
Class<? extends String> clazz = s.getClass();
```

它同样不抛异常，而且既然对象已经创建出来了，这个类**肯定早就初始化过了**。

### 2.2.4 三种方式的对比

| 维度 | `Class.forName(name)` | `类名.class` | `对象.getClass()` |
|------|----------------------|-------------|------------------|
| 输入 | 类的全限定名字符串 | 编译期已知的类型 | 已有对象实例 |
| 触发**类初始化**（执行静态代码块） | **会** | **不会** | **不会**（对象已存在，必然已初始化） |
| 是否抛受检异常 | **抛 `ClassNotFoundException`** | 不抛 | 不抛 |
| 编译期类型检查 | 无（类名写错运行时才发现） | **有** | 有 |
| 典型用途 | 读配置文件、加载 JDBC 驱动、框架按名建对象 | 获取类型元信息、方法参数传类型 | 通用工具方法（如 `equals` 里比较类型） |

三条补充信息：

- 如果**不想触发静态代码块**又必须用字符串加载，可以调用重载版本：`Class.forName(name, false, loader)`，第二个参数 `initialize=false` 表示"只加载不初始化"。
- 还有第四条路：`classLoader.loadClass(name)`。它同样**不触发初始化**，且不要求类必须存在（`ClassNotFoundException` 一样会抛），在类加载器相关的框架代码里更常见。
- 三种方式（以及上面两条）拿到的都是**同一个对象**，可以直接用 `==` 比较：

```java
String s = "hello";
Class<?> c1 = Class.forName("java.lang.String");
Class<?> c2 = String.class;
Class<?> c3 = s.getClass();
System.out.println(c1 == c2);    // true
System.out.println(c2 == c3);    // true
```

> **注意**：`Class.forName` 的"会初始化"这一点经常被拿来出面试题。下面的例子能直接看到区别。

```java
// 同一个文件里的另一个类（非 public 可以放在同一 .java 中）
class Target {
    static {
        // 静态代码块只在类初始化时执行一次
        System.out.println(">>> Target 静态代码块执行了 —— 类被初始化");
    }
}

public class ClassInitDemo {
    public static void main(String[] args) throws Exception {
        System.out.println("--- 用 类名.class 获取 ---");
        Class<?> c1 = Target.class;
        System.out.println("拿到 " + c1.getSimpleName() + "，注意上面没有输出静态代码块");

        System.out.println("--- 用 Class.forName 获取 ---");
        Class<?> c2 = Class.forName("Target");
        System.out.println("拿到 " + c2.getSimpleName() + "，静态代码块被触发了");

        System.out.println("--- 用 Class.forName(..., false, ...) 获取 ---");
        Class<?> c3 = Class.forName("Target", false, ClassInitDemo.class.getClassLoader());
        System.out.println("拿到 " + c3.getSimpleName() + "，静态代码块没有再次执行");

        System.out.println("三次拿到的对象是同一个吗: " + (c1 == c2 && c2 == c3));   // true
    }
}
```

## 2.3 Class 的常用方法

| 分类 | 方法 | 说明 |
|------|------|------|
| **基本信息** | `String getName()` | 全限定类名（含包名），如 `java.lang.String` |
| | `String getSimpleName()` | 简单类名（不含包名），如 `String`；匿名类返回空串 |
| | `String getPackageName()` | 包名（JDK9+ 引入，推荐替代 `getPackage().getName()`） |
| | `Package getPackage()` | `Package` 对象，可能为 null |
| | `Class<? super T> getSuperclass()` | 父类的 `Class` 对象；接口、基本类型、void 返回 null |
| | `String getTypeName()` | 类型名，数组会带上 `[]`，泛型会保留（如 `java.util.List<java.lang.String>`） |
| | `String toGenericString()` | 带泛型和修饰符的完整声明字符串 |
| **判断类型** | `boolean isInterface()` | 是否为接口 |
| | `boolean isArray()` | 是否为数组 |
| | `boolean isPrimitive()` | 是否为基本类型 |
| | `boolean isEnum()` | 是否为枚举 |
| | `boolean isAnnotation()` | 是否为注解类型 |
| | `boolean isRecord()` | 是否为 Record（JDK16+） |
| | `boolean isAssignableFrom(Class<?>)` | 判断"能否赋值"，比 `instanceof` 更灵活（可用于两个 `Class` 对象之间） |
| | `boolean isInstance(Object)` | 等价于 `obj instanceof 本类` |
| **接口与修饰符** | `Class<?>[] getInterfaces()` | 直接实现的**接口**数组（**不含父类实现的接口**） |
| | `Type[] getGenericInterfaces()` | 带泛型信息的接口数组 |
| | `int getModifiers()` | 修饰符的位掩码，配合 `Modifier.toString(int)` 解读 |
| **创建对象** | `T getDeclaredConstructor().newInstance()` | **推荐**的创建方式，替代已废弃的 `newInstance()` |
| **获取成员** | `getConstructors()` / `getDeclaredConstructors()` | 见第 3 节 |
| | `getFields()` / `getDeclaredFields()` | 见第 4 节 |
| | `getMethods()` / `getDeclaredMethods()` | 见第 5 节 |
| **注解** | `Annotation[] getAnnotations()` | 获取所有注解（含继承的） |
| | `<A extends Annotation> A getAnnotation(Class<A>)` | 获取指定注解，没有返回 null |
| | `boolean isAnnotationPresent(Class<? extends Annotation>)` | 是否标注了某个注解 |
| **杂项** | `ClassLoader getClassLoader()` | 获取加载该类的类加载器（JDK 内置类可能返回 null，表示引导类加载器） |
| | `T cast(Object obj)` | 安全的强制转换（失败抛 `ClassCastException`） |
| | `T[] getEnumConstants()` | 获取枚举的所有常量 |

```java
import java.lang.reflect.Modifier;
import java.util.ArrayList;

public class ClassApiDemo {
    public static void main(String[] args) {
        Class<ArrayList> clazz = ArrayList.class;

        System.out.println("全限定名: " + clazz.getName());          // java.util.ArrayList
        System.out.println("简单名:   " + clazz.getSimpleName());     // ArrayList
        System.out.println("包名:     " + clazz.getPackageName());    // java.util
        System.out.println("父类:     " + clazz.getSuperclass().getName());  // java.util.AbstractList
        System.out.println("修饰符:   " + Modifier.toString(clazz.getModifiers()));  // public
        System.out.println("是接口吗: " + clazz.isInterface());       // false

        // getInterfaces 只返回"直接实现"的接口，不递归父类
        for (Class<?> itf : clazz.getInterfaces()) {
            System.out.println("直接实现的接口: " + itf.getName());
        }

        // isAssignableFrom 判断"能不能把后面那个类型赋给前面这个类型"
        System.out.println(Number.class.isAssignableFrom(Integer.class));  // true
        System.out.println(Integer.class.isAssignableFrom(Number.class));  // false
    }
}
```

## 2.4 `getXxx` 与 `getDeclaredXxx` 的区别（最容易混的一点）

这是反射里记忆负担最大的一处，但口诀只有一句：**带 `Declared` 的"看得见私有、看不见继承"；不带的"只看 public、但看得见继承"。**

| 方法 | 只返回 public | 能返回 private/protected/默认 | 包含**继承**来的成员 |
|------|--------------|-----------------------------|---------------------|
| `getFields()` | ✓ | ✗ | **✓**（含父类 public 字段） |
| `getDeclaredFields()` | ✗（public 也返回） | **✓** | ✗（只有本类声明的） |
| `getMethods()` | ✓ | ✗ | **✓**（含父类方法，**包括 `Object` 的 `equals` / `hashCode` / `wait` / `notify` 等**） |
| `getDeclaredMethods()` | ✗（public 也返回） | **✓** | ✗ |
| `getConstructors()` | ✓ | ✗ | ——（**构造方法不继承**，所以两组都能看到本类的全部） |
| `getDeclaredConstructors()` | ✗（public 也返回） | **✓** | —— |

三个必须记住的推论：

1. **想拿 private 成员只能用 `getDeclaredXxx`**，而且拿到之后还要 `setAccessible(true)` 才能真的访问。
2. **`getMethods()` 会返回一大堆 `Object` 的方法**。当你用反射做"打印这个对象的所有方法"这类工具时，看到 `wait`、`notify`、`getClass` 不要惊讶，需要自己过滤 `method.getDeclaringClass() != Object.class`。
3. **`getMethods()` / `getFields()` / `getDeclaredMethods()` 返回数组的顺序是未定义的**。JVM 实现可以按任意顺序返回，不要写"取第 0 个就是 xxx"这种代码。需要固定顺序时，自己按名字排序。

> **陷阱**：`getDeclaredMethod("name", 参数类型...)` 只在本类里找。如果该方法定义在父类，必须去父类找，或者用 `getMethod`（它会沿继承链向上搜索 public 方法）。这是"反射调用父类方法拿不到 `NoSuchMethodException`"的常见原因。

# 3. 操作构造方法：Constructor

## 3.1 获取 Constructor 对象

| 方法 | 说明 |
|------|------|
| `Constructor<?>[] getConstructors()` | 返回所有 **public** 构造方法 |
| `Constructor<?>[] getDeclaredConstructors()` | 返回**全部**构造方法（含 private / protected / 默认） |
| `Constructor<T> getConstructor(Class<?>... parameterTypes)` | 按参数类型获取**指定的 public** 构造方法 |
| `Constructor<T> getDeclaredConstructor(Class<?>... parameterTypes)` | 按参数类型获取**指定的**构造方法（含 private） |

参数类型用 `Class` 的可变参数表示，**必须和声明完全一致**，否则抛 `NoSuchMethodException`：

```java
Class<Student> clazz = Student.class;

// Student(String name, int age)
Constructor<Student> c1 = clazz.getDeclaredConstructor(String.class, int.class);

// Student()：无参构造，不传参数即可
Constructor<Student> c2 = clazz.getDeclaredConstructor();

// 注意 Integer.class 与 int.class 是两个不同的类型，不能混用
// c1 = clazz.getDeclaredConstructor(Integer.class);   // ✗ NoSuchMethodException
```

## 3.2 Constructor 的常用方法

| 方法 | 说明 |
|------|------|
| `T newInstance(Object... initargs)` | 按这个构造方法**创建对象**，参数与构造方法一一对应 |
| `String getName()` | 构造方法名，永远等于类名（全限定名形式） |
| `int getParameterCount()` | 参数个数 |
| `Class<?>[] getParameterTypes()` | 参数类型数组 |
| `int getModifiers()` | 修饰符位掩码 |
| `void setAccessible(boolean flag)` | 设为 `true` 时**取消访问权限检查**，即"暴力反射" |
| `boolean isAccessible()` | 当前的 accessible 标志（注意它反映的是标志位，不是真实权限） |
| `Class<T> getDeclaringClass()` | 声明该构造方法的类 |

## 3.3 暴力反射与 `Class.newInstance()` 的废弃

**暴力反射**指调用 `setAccessible(true)` 关掉 Java 的访问权限检查，从而访问 private 构造方法、private 字段、private 方法。它的本质不是"改了权限"，而是**告诉 JVM：跳过这次安全检查**，所以它只在当前反射调用中生效，其他地方仍然访问不了。

```java
import java.lang.reflect.Constructor;

class Student {
    private String name;
    private int age;

    private Student(String name) {   // 私有构造方法
        this.name = name;
        System.out.println("私有构造被反射调用了，name = " + name);
    }

    @Override
    public String toString() {
        return "Student{name='" + name + "', age=" + age + "}";
    }
}

public class ConstructorDemo {
    public static void main(String[] args) throws Exception {
        Class<Student> clazz = Student.class;

        // 1) 拿到私有构造方法
        Constructor<Student> cons = clazz.getDeclaredConstructor(String.class);
        // 2) 暴力反射：不写这一行会抛 IllegalAccessException: class ... can not access
        cons.setAccessible(true);
        // 3) 创建对象
        Student stu = cons.newInstance("张三");

        System.out.println(stu);   // Student{name='张三', age=0}
    }
}
```

**为什么 `Class.newInstance()` 在 JDK9 被标记废弃？** 它有三个说不清的毛病，而 `clazz.getDeclaredConstructor().newInstance()` 一次全解决：

| 问题 | `Class.newInstance()`（已废弃） | `getDeclaredConstructor().newInstance()` |
|------|--------------------------------|------------------------------------------|
| 构造方法选择 | **只能调无参构造** | 可以指定任意参数列表的构造方法 |
| 访问权限 | 只能调**可访问**的构造方法，无法暴力反射私有构造 | 配合 `setAccessible(true)` 可以调私有构造 |
| 异常语义 | 构造方法里抛出的**受检异常被原样抛出**，而它在方法签名里没有声明——调用者可能被一个"没声明的异常"击中，编译器无法检查 | 构造方法内的异常被包装成 `InvocationTargetException`，调用者能明确区分"构造失败"和"反射失败" |

```java
// 旧写法（已废弃）：只能无参、只能 public、异常语义混乱
Student s = clazz.newInstance();

// 新写法（推荐）：等价于无参 + public，但异常语义清晰，还能扩展成任意构造
Student s2 = clazz.getDeclaredConstructor().newInstance();
```

> **注意**：`setAccessible(true)` 虽然方便，但它**打破了封装**。框架用它是因为"框架必须能操作业务类"，业务代码用它几乎总是设计问题的信号。JDK9 之后，模块系统还会进一步限制它的能力（见第 7 节）。

# 4. 操作成员变量：Field

## 4.1 获取 Field 对象

| 方法 | 说明 |
|------|------|
| `Field[] getFields()` | 所有 **public** 字段，**含继承**的 |
| `Field[] getDeclaredFields()` | 本类声明的**全部**字段（含 private），**不含继承**的 |
| `Field getField(String name)` | 按名字取指定的 public 字段（含继承） |
| `Field getDeclaredField(String name)` | 按名字取本类声明的字段（含 private） |

## 4.2 Field 的常用方法

| 方法 | 说明 |
|------|------|
| `void set(Object obj, Object value)` | 把 `obj` 对象的这个字段设为 `value`；**静态字段 `obj` 传 `null`** |
| `Object get(Object obj)` | 取出 `obj` 对象的这个字段值；**静态字段 `obj` 传 `null`** |
| `String getName()` | 字段名 |
| `Class<?> getType()` | 字段类型 |
| `int getModifiers()` | 修饰符位掩码 |
| `void setAccessible(boolean flag)` | 取消访问检查，用于访问 private 字段 |
| `Object get(Object)` 的原始类型版本 | `getInt` / `getLong` / `getDouble` / `getBoolean` 等，免去装箱 |

```java
import java.lang.reflect.Field;

class Student {
    private String name;        // 私有实例字段
    public int age;             // public 实例字段
    public static String school = "某大学";   // 静态字段
}

public class FieldDemo {
    public static void main(String[] args) throws Exception {
        Class<Student> clazz = Student.class;
        Student stu = new Student();

        // 1) 实例字段：private 需要暴力反射
        Field nameField = clazz.getDeclaredField("name");
        nameField.setAccessible(true);          // 不写会抛 IllegalAccessException
        nameField.set(stu, "李四");
        System.out.println("name = " + nameField.get(stu));   // 李四

        // 2) public 实例字段：不需要 setAccessible
        Field ageField = clazz.getField("age");
        ageField.set(stu, 25);
        System.out.println("age = " + ageField.get(stu));     // 25

        // 3) 静态字段：第一个参数必须传 null
        Field schoolField = clazz.getField("school");
        schoolField.set(null, "另一所大学");                    // 第一个参数是 null，不是 stu
        System.out.println("school = " + schoolField.get(null));

        // 4) 遍历本类所有字段，打印名字与类型
        for (Field f : clazz.getDeclaredFields()) {
            System.out.println("字段: " + f.getName() + "，类型: " + f.getType().getSimpleName());
        }
    }
}
```

> **陷阱**：**反射赋值的类型必须能转换到字段类型**。给 `int` 字段传 `Integer` 会自动拆箱、给 `Integer` 字段传 `int` 会自动装箱，这些没问题；但把 `String` 塞给 `int` 字段会抛 `IllegalArgumentException`。做通用的属性拷贝（如把 Map 转成对象）时，类型转换必须自己处理。

> **注意**：对 **`static final` 字段**，`setAccessible(true)` 之后 `set` 通常仍然失败（JDK9+ 会抛 `IllegalAccessException: Can not set static final ... field`），因为 final 的语义被写死在 JVM 里。想改只能靠 `Unsafe`，而那已经属于"反模式"，不要在生产代码里用。

# 5. 操作成员方法：Method

## 5.1 获取 Method 对象

| 方法 | 说明 |
|------|------|
| `Method[] getMethods()` | 所有 **public** 方法，**含继承**（包括 `Object` 的那一堆方法） |
| `Method[] getDeclaredMethods()` | 本类声明的**全部**方法（含 private），**不含继承** |
| `Method getMethod(String name, Class<?>... parameterTypes)` | 按名字 + 参数类型取 public 方法（会沿继承链找） |
| `Method getDeclaredMethod(String name, Class<?>... parameterTypes)` | 按名字 + 参数类型取本类声明的方法（含 private） |

参数类型同样必须精确匹配。**无参方法不要传 `null`**，直接省略即可：

```java
Method m1 = clazz.getMethod("getAge");                        // 无参：什么都不传
Method m2 = clazz.getMethod("setAge", int.class);             // 一个 int 参数
// Method m3 = clazz.getMethod("setAge", null);               // ✗ 有歧义，不要这么写
```

## 5.2 Method 的常用方法

| 方法 | 说明 |
|------|------|
| `Object invoke(Object obj, Object... args)` | 调用方法。`obj` 是目标对象（**静态方法传 `null`**），`args` 是实参；返回方法返回值（**void 方法返回 `null`**） |
| `String getName()` | 方法名 |
| `Class<?> getReturnType()` | 返回值类型 |
| `Class<?>[] getParameterTypes()` | 参数类型数组 |
| `int getParameterCount()` | 参数个数 |
| `int getModifiers()` | 修饰符位掩码 |
| `void setAccessible(boolean flag)` | 取消访问检查，用于调用 private 方法 |
| `Class<?> getDeclaringClass()` | 声明该方法的类（判断是否继承自 `Object` 时有用） |
| `boolean isVarArgs()` | 是否是可变参数方法 |

```java
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

class Student {
    private int age;

    public void setAge(int age) { this.age = age; }
    public int getAge() { return age; }

    private String secret() { return "私有方法的结果"; }

    public static String staticHello(String name) { return "你好，" + name; }

    /** 会抛异常的方法：用于演示 InvocationTargetException */
    public void boom() { throw new IllegalStateException("业务逻辑自己抛的异常"); }
}

public class MethodDemo {
    public static void main(String[] args) throws Exception {
        Class<Student> clazz = Student.class;
        Student stu = new Student();

        // 1) 调用实例方法（参数是 int，会自动装箱成 Integer 放进来）
        Method setAge = clazz.getMethod("setAge", int.class);
        setAge.invoke(stu, 25);

        Method getAge = clazz.getMethod("getAge");
        Object result = getAge.invoke(stu);
        System.out.println("age = " + result);      // 25（基本类型被装箱成 Integer）

        // 2) 调用私有方法：getDeclaredMethod + 暴力反射
        Method secret = clazz.getDeclaredMethod("secret");
        secret.setAccessible(true);
        System.out.println(secret.invoke(stu));     // 私有方法的结果

        // 3) 调用静态方法：目标对象传 null
        Method staticHello = clazz.getMethod("staticHello", String.class);
        System.out.println(staticHello.invoke(null, "张三"));

        // 4) 目标方法自己抛异常时，会被包装成 InvocationTargetException
        Method boom = clazz.getMethod("boom");
        try {
            boom.invoke(stu);
        } catch (InvocationTargetException e) {
            // 真实异常在 getCause() 里，直接看 e 只会看到"反射调用失败"这个壳
            Throwable real = e.getCause();
            System.out.println("目标方法抛出了: " + real.getClass().getSimpleName() + " - " + real.getMessage());
        }
    }
}
```

三个高频坑：

| 坑 | 现象 | 说明 |
|----|------|------|
| **受检异常被包装** | 明明业务代码只抛 `IllegalStateException`，反射处却要 catch `InvocationTargetException` | `invoke` 把目标方法抛出的任何异常都包一层，**原始异常必须用 `e.getCause()` 取** |
| **可变参数被"拍平"** | 反射调用 `foo(String...)` 时参数被当成多个实参 | `invoke(obj, args)` 的 `args` 本身就是 `Object[]`，传数组时会被自动展开。想让它作为"一个数组参数"传入，必须强转：`invoke(obj, (Object) new String[]{"a", "b"})` |
| **依赖 `getMethods()` 的顺序** | 换 JVM 版本后逻辑错乱 | 返回顺序**未定义**，需要顺序就自己排序 |

```java
// 可变参数的经典陷阱：两种写法结果完全不同
class VarArgs {
    public void printAll(String... items) {
        System.out.println("收到了 " + items.length + " 个参数");
    }
}

Method printAll = VarArgs.class.getMethod("printAll", String[].class);
// ✓ 作为"一个 String[] 参数"传入：输出 "收到了 2 个参数"
printAll.invoke(new VarArgs(), (Object) new String[]{"a", "b"});
// ✗ 被当成两个独立实参传进去，参数个数不匹配时会抛 IllegalArgumentException
// printAll.invoke(new VarArgs(), new String[]{"a", "b"});
```

> **注意**：`invoke` 的实参个数不匹配、类型不匹配时抛的都是 `IllegalArgumentException`（注意跟 `InvocationTargetException` 区分：前者是"你传错了"，后者是"目标方法自己炸了"）。排查反射问题时，先把这两个异常分清楚，能省一半时间。

# 6. 反射的性能开销与优化

## 6.1 开销从哪来

反射比直接调用慢，原因有四个，都要能说出来：

1. **访问权限检查**——每次调用都要走一遍 `Reflection` 的权限校验；打开 `setAccessible` 可以省掉；
2. **参数需要装箱**——不管什么类型的方法，参数都得塞进 `Object[]`，基本类型要装箱成对象；
3. **调用链更长**——正常的 `invokevirtual` 是一条字节码指令，反射要经过 `Method.invoke` → `MethodAccessor` → 本地代码，JIT 难以内联；
4. **元数据查找本身有成本**——`getDeclaredMethod` 每次都要遍历方法表并**返回一个 `Method` 副本对象**，放在循环里调用是灾难。

## 6.2 五条优化手段

| 手段 | 做法 | 收益 |
|------|------|------|
| **开 `setAccessible(true)`** | 在拿到 `Method` / `Field` / `Constructor` 后立即调用一次 | 跳过访问检查，明显的加速，也是"顺手"的收益 |
| **缓存反射对象** | 用 `static final` 或 `ConcurrentHashMap<Class, Method>` 缓存 `Method` / `Field` / `Constructor` | 避免重复查找与副本创建，**收益最大的一条** |
| **用 `MethodHandle`（JDK7+）** | `MethodHandles.lookup().findVirtual(...)` | 现代 JVM 上更易被 JIT 内联，性能接近直接调用 |
| **用 `LambdaMetafactory`** | 把反射调用点转成函数式接口的实现 | 理论上最快，但写法复杂，一般框架内部才用 |
| **能不用就不用** | 用接口 + 多态、`Map<String, Function>`、`Supplier` 替代 | 零开销，且保留编译期检查 |

```java
import java.lang.invoke.MethodHandle;
import java.lang.invoke.MethodHandles;
import java.lang.invoke.MethodType;

class Greeter {
    public String greet(String name) { return "你好，" + name; }
}

public class MethodHandleDemo {
    public static void main(String[] args) throws Throwable {
        Greeter greeter = new Greeter();

        // MethodHandles.lookup() 拿到的 Lookup 对象带有"调用者权限"，
        // 因此它能访问的范围和当前类直接写代码能访问的范围一致
        MethodHandles.Lookup lookup = MethodHandles.lookup();
        MethodHandle mh = lookup.findVirtual(Greeter.class, "greet",
                MethodType.methodType(String.class, String.class));   // 返回值类型 + 参数类型

        String r = (String) mh.invoke(greeter, "张三");               // 像函数指针一样调用
        System.out.println(r);                                       // 你好，张三

        // 还可以绑定接收者，得到一个"已绑定对象"的句柄，之后不用每次传对象
        MethodHandle bound = mh.bindTo(greeter);
        System.out.println(bound.invoke("李四"));
    }
}
```

> **行业实践**：**绝不要用 `System.currentTimeMillis()` 手写反射性能测试**。JIT 的预热、内联、逃逸分析会让微基准结果完全失真。要测性能请用 **JMH**（OpenJDK 官方微基准框架），并且只在**真实热点路径**上做优化——框架启动时用反射建对象的成本，相对于整个请求链路可以忽略。所谓"反射慢"，在绝大多数业务场景里是个伪命题。

下面这张表只是给出**数量级上的直觉**，具体倍率随 JVM 版本、JIT 状态、方法复杂度剧烈变化，不要当结论用：

| 调用方式 | 相对耗时（量级参考） |
|----------|---------------------|
| 直接调用 `obj.method()` | 1× |
| `MethodHandle.invoke` | 约 1.5 ~ 3× |
| 反射 + 已缓存 `Method` + `setAccessible(true)` | 约 2 ~ 5× |
| 反射 + 已缓存 `Method`（不开 `setAccessible`） | 约 5 ~ 20× |
| 每次循环里都 `getDeclaredMethod` | **几十倍以上**，因为有元数据查找开销 |

# 7. JDK9 模块化对反射的限制

JDK9 引入 **JPMS（Java Platform Module System，模块系统）**后，`setAccessible(true)` 的能力被收紧了。它的新语义是：

> **注意**：`setAccessible(true)` 的新语义是——**成员所属的包只要对调用方所在模块是 `open` 的，就能访问；否则抛 `InaccessibleObjectException`**。

也就是说，"暴力反射"不再是"想打哪儿就打哪儿"，而是必须在**模块边界上被显式授权**。

| 情况 | 结果 |
|------|------|
| 同一个模块内访问自己类的私有成员 | ✓ 允许 |
| 目标包被 `opens` 到了你的模块 | ✓ 允许（如 `opens com.foo.internal to spring.core;`） |
| 目标是 JDK 内部类，且包未被导出/开放 | ✗ `InaccessibleObjectException` |
| 代码在**未命名模块**（classpath 上的普通 jar）访问 JDK 内部包 | ✗ 默认失败，需要加启动参数 |

典型的失败场景，就是把老代码升级到 JDK 9+ 时最常撞见的报错：

```text
Exception in thread "main" java.lang.reflect.InaccessibleObjectException:
  Unable to make field private final byte[] java.lang.String.value accessible:
  module java.base does not "opens java.lang" to unnamed module @1b6d3586
```

解决方案是在启动参数里显式"打开"这个包：

```bash
# 允许所有未命名模块（也就是 classpath 上的代码）反射 java.lang 包
java --add-opens java.base/java.lang=ALL-UNNAMED -jar app.jar
```

`--add-opens` 正是 Spring、Hibernate、Jackson 等在 JDK 17 上运行时经常需要加的参数；也是很多项目从 JDK 8 升到 17 时"第一脚就踩空"的地方。

> **注意**：模块化的限制并**不影响**普通反射的应用方法——你对自己的业务类做反射、对 classpath 上同一个模块内的类做反射，都不需要任何额外配置。只有当你试图去改 `java.lang.String` 的内部数组、`sun.misc.Unsafe` 这类 JDK 内部实现时，才会撞墙。这本身是一种保护。

# 8. 动态代理

## 8.1 什么是代理，静态代理有什么问题

**代理模式**的核心是：为某个对象提供一个"替身"，外部对目标的调用都先经过替身，替身可以在调用前后插入额外逻辑（日志、事务、权限、缓存），而**目标对象的代码完全不用改**。

最朴素的实现是**静态代理**：手写一个类实现同样的接口，持有目标对象，在每个方法里加增强：

```java
public interface UserService {
    void addUser(String name);
    String getUser(int id);
}

public class UserServiceImpl implements UserService {
    @Override public void addUser(String name) { System.out.println("添加用户: " + name); }
    @Override public String getUser(int id) { return "User" + id; }
}

// 静态代理：为 UserService 手写一个代理类
public class UserServiceProxy implements UserService {
    private final UserService target;
    public UserServiceProxy(UserService target) { this.target = target; }

    @Override
    public void addUser(String name) {
        System.out.println("[日志] addUser 开始");     // 重复的样板代码
        target.addUser(name);
        System.out.println("[日志] addUser 结束");     // 每个方法都要复制一遍
    }

    @Override
    public String getUser(int id) {
        System.out.println("[日志] getUser 开始");     // 换个方法，同样的代码再抄一遍
        String r = target.getUser(id);
        System.out.println("[日志] getUser 结束");
        return r;
    }
}
```

问题很明确：**接口一多、方法一多，代理类就爆炸，而且增强逻辑到处复制**。如果 50 个接口各 10 个方法，就要手写 500 遍同样的日志代码。**动态代理**就是让 JVM 在运行时**自动生成**这个代理类，从而只写一遍增强逻辑。

## 8.2 JDK 动态代理的原理：为什么只能代理接口

JDK 提供的动态代理位于 `java.lang.reflect.Proxy`，它的工作方式是：调用 `Proxy.newProxyInstance(...)` 时，**在内存里动态生成一个类的字节码**（类名通常形如 `com.sun.proxy.$Proxy0` 或 `jdk.proxy1.$Proxy0`），这个类有两个关键特征：

1. 它 **`implements` 你传入的所有接口**——接口方法因此都有了；
2. 它 **`extends java.lang.reflect.Proxy`**——所有方法的实现都被转发给 `InvocationHandler.invoke()`。

```text
   调用方 main
       │
       ▼
   ┌──────────────────────────────────────────────┐
   │  代理对象 $Proxy0                             │
   │  - 由 Proxy.newProxyInstance 在运行期生成      │
   │  - implements 目标接口 UserService            │
   │  - extends java.lang.reflect.Proxy            │
   │  - 每个接口方法都只做一件事：转发给 handler     │
   └────────────────────┬─────────────────────────┘
                        │  invoke(proxy, method, args)
                        ▼
   ┌──────────────────────────────────────────────┐
   │  InvocationHandler 实现类                     │
   │  - 前置增强：日志 / 开启事务 / 权限校验         │
   │  - method.invoke(target, args) —— 调真正的方法 │
   │  - 后置增强：提交事务 / 统计耗时 / 缓存结果     │
   └────────────────────┬─────────────────────────┘
                        │
                        ▼
   ┌──────────────────────────────────────────────┐
   │  目标对象 UserServiceImpl                     │
   │  - 纯业务逻辑，完全不知道自己被代理了           │
   └──────────────────────────────────────────────┘
```

**为什么 JDK 动态代理只能代理接口？** 答案就藏在上面第一句里：生成的代理类**已经 `extends Proxy` 了**。Java 是单继承，一个类不能同时继承 `Proxy` 和 `UserServiceImpl`，所以"通过继承目标类来复写方法"这条路走不通，只能靠"实现接口"来获得方法。因此：

- 目标对象**必须实现至少一个接口**，否则 `newProxyInstance` 直接抛 `IllegalArgumentException: ... is not an interface`；
- 代理对象**实现了目标接口，但不是目标类的实例**，所以 `proxy instanceof UserServiceImpl` 是 `false`，强转成实现类会抛 `ClassCastException`。

> **陷阱**：这是使用 JDK 动态代理最常见的错误——`UserServiceImpl proxy = (UserServiceImpl) Proxy.newProxyInstance(...)` 会直接 `ClassCastException`。**必须用接口类型接收**：`UserService proxy = (UserService) ...`。同理，Spring 里用 JDK 代理的 Bean 也必须按接口注入。

## 8.3 `Proxy.newProxyInstance` 三个参数

```java
public static Object newProxyInstance(
    ClassLoader loader,       // ① 类加载器
    Class<?>[] interfaces,    // ② 目标实现的所有接口
    InvocationHandler h       // ③ 调用处理器
)
```

| 参数 | 含义 | 怎么传 |
|------|------|--------|
| `loader` | 用于加载动态生成的代理类 | 通常用 `target.getClass().getClassLoader()`；也可以用自己的类加载器 |
| `interfaces` | 代理类要实现的接口数组 | `target.getClass().getInterfaces()`；**必须全是接口**，含具体类就抛 `IllegalArgumentException` |
| `h` | 所有方法调用的统一入口 | 自己实现 `InvocationHandler`，在其中调用 `method.invoke(target, args)` |

`InvocationHandler` 接口只有一个方法：

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable;
```

它的三个参数分别是：

| 参数 | 是什么 | 使用注意 |
|------|--------|----------|
| `proxy` | **代理对象本身**（不是目标对象） | 极少直接使用；在 `invoke` 里用 `proxy` 调方法会导致**无限递归**（因为又走回 `invoke`） |
| `method` | 被调用的**接口方法**对象 | 用它 `getName()` 判断方法名、用它 `invoke(target, args)` 转发到目标 |
| `args` | 实参数组 | 无参方法时是 `null`，转发给 `invoke` 时**直接传即可**（`invoke` 接受 null） |

## 8.4 完整可运行示例：日志 + 耗时 + 事务

下面这个示例把"日志、耗时统计、模拟事务"三种增强都织进一个 `InvocationHandler` 里，是理解 Spring AOP 最好的模型。

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;

// ---------- 1. 业务接口 ----------
interface UserService {
    void addUser(String name);
    String getUser(int id);
    void deleteUser(int id);
}

// ---------- 2. 目标实现类（纯业务，不含任何增强代码） ----------
class UserServiceImpl implements UserService {
    private final Map<Integer, String> db = new HashMap<>();
    private int nextId = 1;

    @Override
    public void addUser(String name) {
        db.put(nextId++, name);
        System.out.println("      [目标] 插入用户: " + name);
    }

    @Override
    public String getUser(int id) {
        System.out.println("      [目标] 查询用户 id=" + id);
        String name = db.get(id);
        if (name == null) {
            // 故意抛异常，用来演示代理如何处理目标方法的异常
            throw new IllegalArgumentException("用户不存在: " + id);
        }
        return name;
    }

    @Override
    public void deleteUser(int id) {
        System.out.println("      [目标] 删除用户 id=" + id);
        db.remove(id);
    }
}

// ---------- 3. 调用处理器：所有增强逻辑集中在这里，只写一遍 ----------
class LoggingTransactionHandler implements InvocationHandler {
    private final Object target;    // 真实的目标对象

    public LoggingTransactionHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        String name = method.getName();
        String argText = args == null ? "无参" : java.util.Arrays.toString(args);

        System.out.println("   [前置] → 调用 " + name + "(" + argText + ")");
        long start = System.nanoTime();
        boolean success = false;
        try {
            // 开启事务（这里用打印模拟）
            System.out.println("   [事务] BEGIN");

            Object result = method.invoke(target, args);   // 真正调用目标方法
            success = true;
            return result;
        } catch (java.lang.reflect.InvocationTargetException e) {
            // 必须拆开包装，把原始异常抛出去，否则调用方拿到的异常语义就变了
            throw e.getCause();
        } finally {
            long cost = (System.nanoTime() - start) / 1_000_000;
            System.out.println("   [事务] " + (success ? "COMMIT" : "ROLLBACK"));
            System.out.println("   [后置] 方法 " + name + " 结束，耗时 " + cost + " ms");
        }
    }
}

// ---------- 4. 一个通用的代理工厂 ----------
class ProxyFactory {
    @SuppressWarnings("unchecked")
    public static <T> T createProxy(T target) {
        return (T) Proxy.newProxyInstance(
                target.getClass().getClassLoader(),   // ① 类加载器
                target.getClass().getInterfaces(),    // ② 目标实现的接口（必须全是接口）
                new LoggingTransactionHandler(target) // ③ 调用处理器
        );
    }
}

// ---------- 5. 使用 ----------
public class DynamicProxyDemo {
    public static void main(String[] args) {
        // 注意：必须用【接口类型】接收，强转成 UserServiceImpl 会 ClassCastException
        UserService target = new UserServiceImpl();
        UserService proxy = ProxyFactory.createProxy(target);

        System.out.println("代理对象的实际类型: " + proxy.getClass().getName());
        System.out.println("是 Proxy 的子类吗: " + (proxy instanceof Proxy));
        System.out.println("是 UserServiceImpl 的实例吗: " + (proxy instanceof UserServiceImpl));   // false

        proxy.addUser("张三");
        System.out.println("查询结果: " + proxy.getUser(1));          // 正常返回

        try {
            proxy.getUser(999);                                       // 目标方法会抛异常
        } catch (IllegalArgumentException e) {
            System.out.println("捕获到原始异常: " + e.getMessage());    // 异常类型没有被改变
        }
    }
}
```

运行这段代码可以观察到三件事：**所有方法都自动带上了日志和耗时**；**事务在异常时 ROLLBACK、正常时 COMMIT**；**目标类 `UserServiceImpl` 一行增强代码都没写**。这就是 AOP 的本质。

## 8.5 JDK 动态代理 vs CGLIB vs Spring AOP

CGLIB（Code Generation Library）走的是另一条路：它用字节码生成库（早期是 ASM）**动态生成目标类的子类**，在子类里复写每个方法，方法体里回调 `MethodInterceptor`。

| 维度 | JDK 动态代理 | CGLIB |
|------|-------------|-------|
| 实现原理 | 生成**实现接口**的代理类（`implements` 目标接口 + `extends Proxy`） | 生成目标类的**子类**并复写方法 |
| 前提条件 | 目标**必须实现接口** | 目标类**不能是 final**，被代理方法**不能是 final / private / static** |
| 依赖 | JDK 内置，无需第三方 | 第三方库（Spring 已内置 `spring-core` 里的 repackaged 版本） |
| 代理对象类型 | 与目标**实现同一接口**，不是目标类的实例 | 是目标类的**子类**，可以强转成目标类 |
| 创建速度 | 快（逻辑简单） | 慢（要生成并加载新类的字节码） |
| 调用速度 | JDK8 之后与 CGLIB 差距很小 | 早期版本调用更快 |
| 私有方法 | 不涉及（接口方法都是 public） | 无法代理（子类访问不到） |

**Spring AOP 的选择规则**（必须记住）：

| 条件 | 使用的代理 | 说明 |
|------|-----------|------|
| 目标类**实现了接口**，且未强制指定 CGLIB | **JDK 动态代理** | Spring 的默认行为 |
| 目标类**没有实现任何接口** | **CGLIB** | 只能用 CGLIB，别无选择 |
| 配置了 `proxy-target-class=true`（或 Spring Boot 2.x 及其后的默认值） | **CGLIB** | Spring Boot 默认已改为强制 CGLIB |

> **陷阱**：Spring 里有两个和代理相关的经典坑，都和"代理对象不是原对象"有关。
> 一是**按实现类注入会失败**：用 JDK 代理时 `@Autowired UserServiceImpl service;` 会报"找不到匹配的 Bean"，必须按接口 `UserService` 注入（或改用 CGLIB）。
> 二是**同类内部方法互调不走代理**：`this.otherMethod()` 用的是原始对象而非代理对象，所以 `@Transactional` / `@Cacheable` 直接失效——这是自调用（self-invocation）问题，也是"事务没生效"最常见的原因。解决办法是把方法拆到另一个 Bean，或从容器里取代理对象调用。

# 9. 反射 + 配置文件：Spring 的雏形

反射最经典的综合应用是**"读配置 → 按名建对象 → 按名调方法"**。这套模式正是所有 IOC 容器的起点：**框架不编译业务类，却能在运行时把业务类装配起来。**

## 9.1 配置文件

```properties
# app.properties —— 类名、方法名、参数全部写在配置里
service.class=com.example.service.UserServiceImpl
service.method=addUser
service.arg=测试用户
```

## 9.2 完整可运行示例

```java
import java.io.InputStream;
import java.io.Reader;
import java.lang.reflect.Method;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.Properties;

public class MiniContainerDemo {
    public static void main(String[] args) throws Exception {
        Path configFile = Paths.get("app.properties");

        // 编码坑：Properties.load(InputStream) 按 ISO-8859-1 解码，
        //   配置文件里有中文时必须用 load(Reader) 并指定 UTF-8
        Properties props = new Properties();
        try (Reader reader = Files.newBufferedReader(configFile, StandardCharsets.UTF_8)) {
            props.load(reader);
        }

        String className  = props.getProperty("service.class");
        String methodName = props.getProperty("service.method");
        String arg        = props.getProperty("service.arg");
        System.out.println("配置读取完毕: " + className + "#" + methodName);

        // ① 用类名加载类（字符串来自配置，编译期完全不知道是什么类）
        Class<?> clazz = Class.forName(className);

        // ② 反射创建实例：用 getDeclaredConstructor().newInstance() 替代已废弃的 newInstance()
        Object bean = clazz.getDeclaredConstructor().newInstance();
        System.out.println("已创建实例: " + bean.getClass().getName());

        // ③ 反射找到并调用方法（参数类型是 String）
        Method method = clazz.getMethod(methodName, String.class);
        Object ret = method.invoke(bean, arg);
        System.out.println("调用完成，返回值: " + ret);
    }
}
```

配套的业务类（放在 `com.example.service` 包下）：

```java
package com.example.service;

public class UserServiceImpl {
    public String addUser(String name) {
        System.out.println("      [业务] 新增用户: " + name);
        return "OK";
    }
}
```

## 9.3 这就是 Spring 的雏形

把上面这段代码扩展一下，就得到了一个最小的 IOC 容器：

| 步骤 | 迷你容器做的事 | Spring 额外做的事 |
|------|---------------|------------------|
| 1. 读配置 | 从 `app.properties` 读类名 | 扫描 `@Component` / 解析 `@Bean` / 读 `application.yml` |
| 2. 建对象 | `Class.forName` + `newInstance` | 处理构造器注入、单例缓存、循环依赖（三级缓存）、生命周期回调 |
| 3. 注入依赖 | 无 | 通过 `Field.set` / 构造器参数把依赖塞进去 |
| 4. 调用 | 按名 `getMethod(...).invoke(...)` | 由 MVC 框架按 URL 分发，用反射调 Controller 方法 |
| 5. 加增强 | 无 | 用 JDK 动态代理 / CGLIB 把 `@Transactional` 等切面织进调用链 |

**同一个反射内核，在各大框架里的用法**：

| 框架 | 用反射做什么 |
|------|-------------|
| **Spring IOC** | 读配置/注解 → `Class.forName` → 实例化 → `Field`/`Constructor` 依赖注入 → 存入 Bean 容器 |
| **Spring AOP** | 用 `Proxy.newProxyInstance` 或 CGLIB 生成代理对象，把切面逻辑织入 |
| **Spring MVC** | 反射调用 Controller 方法，把 HTTP 参数通过 `Method.invoke` 传进去 |
| **MyBatis** | Mapper 只有接口没有实现类，靠 JDK 动态代理在运行时造出实现；查询结果用 `Field.set` 映射到实体 |
| **JUnit** | 反射遍历测试类的方法，找到带 `@Test` 的，`setAccessible(true)` 后调用 |
| **Jackson / Fastjson** | 反射读写字段，完成对象与 JSON 的互转 |

> **行业实践**：自己手写框架时，反射是绕不过去的工具；但**写业务代码时反射几乎总能用更好的方式替代**。判断标准很简单：如果那个类名是你自己**编译期就写死在代码里**的，那就用 `new`；只有"类名来自外部输入（配置、注解、数据库）"时才需要反射。

# 10. 小结

| 主题 | 一句话要点 |
|------|-----------|
| 反射是什么 | 运行期获取类信息并操作成员的能力，是"框架操作业务类"的唯一手段；代价是性能、编译期检查与封装 |
| Class 对象 | 类加载的产物；**同一个类 + 同一个类加载器 = 唯一一份**，所以可用 `==` 比较；跨类加载器会得到两个不同的对象 |
| 获取 Class | `Class.forName`（**触发静态代码块**、抛 `ClassNotFoundException`）、`类名.class`（不触发、不抛）、`对象.getClass()`（不触发、不抛） |
| `Declared` | **带 `Declared` 看得见私有、看不见继承；不带只看 public、但看得见继承**。`getMethods()` 还包含 `Object` 的一堆方法，且顺序未定义 |
| Constructor | `getDeclaredConstructor(参数类型...)` 获取，`setAccessible(true)` 暴力反射破封装，`newInstance(args)` 创建；`Class.newInstance()` 因"只能无参 + 只能 public + 异常语义混乱"在 JDK9 废弃 |
| Field | `get`/`set` 读写，**静态字段第一个参数传 `null`**；`static final` 字段即使暴力反射也改不了 |
| Method | `invoke(obj, args)`，**静态方法传 `null`**；目标方法的异常被包装成 `InvocationTargetException`，**必须用 `getCause()` 取原始异常**；可变参数传数组要强转 `(Object)` 避免被展开 |
| 性能优化 | 开 `setAccessible(true)` + **缓存 `Class`/`Method` 对象**收益最大；`MethodHandle` 更快；基准测试必须用 **JMH** |
| 模块化限制 | JDK9 起 `setAccessible` 需要目标包被 `opens`，否则抛 `InaccessibleObjectException`；解决办法是 `--add-opens` |
| 动态代理 | 运行时生成 `implements 接口 + extends Proxy` 的代理类，所有调用转发到 `InvocationHandler.invoke`；**只能代理接口**（因为已经继承了 `Proxy`），必须用接口类型接收 |
| 代理选型 | 有接口 → JDK 动态代理；无接口 → CGLIB（类与方法不能 final）；Spring Boot 2.x 起默认强制 CGLIB；注意"按实现类注入失败"和"自调用不走代理"两个坑 |
| 与框架的关系 | 读配置 → `Class.forName` → `newInstance` → 注入 → 代理增强，这一条链路就是 Spring IOC + AOP 的内核 |

下一篇（第 20 篇）讲**枚举**——一种"编译器帮你把常量、类型安全和单例都做好"的特殊类，它和反射配合时还有几个特有的限制（无法反射创建枚举实例）值得留意；AOP 在 Spring 里怎么落地，见 Spring 板块关于 IOC 与代理的章节。
