注解（Annotation）是 JDK5 引入的一种**元数据**机制，它本身不改变程序的行为，只是往类、方法、字段上"贴标签"。真正让这些标签产生作用的是**反射**——框架读取标签、据此决定怎么处理你的代码。理解注解的关键就一句话：**注解必须靠反射读取才有意义**；只定义注解而不写读取它的代码，那些注解和注释没有区别。本篇讲清注解的定义语法、元注解、JDK 内置注解，最后用两个可运行的小框架把"自定义注解 + 反射"这条链路走通。

# 1. 注解是什么

## 1.1 注解是一种元数据

注解（`Annotation`）是一种**引用数据类型**，编译之后同样会生成 `Xxx.class` 文件。它的本质是**附加在程序元素上的元数据（metadata）**——描述"这个类是什么"、"这个方法有什么特性"的数据。

```java
// @Override 就是一个注解：它贴在方法上，但不参与任何计算
@Override
public String toString() {
    return "demo";
}
```

从 JVM 的角度看，注解是一张挂在类、字段、方法、参数上的**附加表**。当 `@Retention` 设为 `RUNTIME` 时，这张表会被写进 class 文件的属性区（`RuntimeVisibleAnnotations`），运行期可以通过反射完整读出来。

## 1.2 注解 vs 注释

初学者最容易混淆的一对概念，差别是本质性的：

| 维度 | 注释（Comment） | 注解（Annotation） |
|------|----------------|-------------------|
| 写法 | `// ...`、`/* ... */`、`/** ... */` | `@Xxx` |
| 面向对象 | **人**（读代码的人） | **程序**（编译器和反射） |
| 编译产物 | 直接被丢弃，不进 class 文件 | 可能进 class 文件，甚至运行期可读 |
| 是否影响行为 | 完全不影响 | 配合反射读取，可以决定程序行为 |
| 典型例子 | `// 这里要判空` | `@Override`、`@Deprecated`、`@RestController` |

一句话概括：**注释是写给人看的说明，注解是写给程序看的标签**。

## 1.3 注解本身不改变行为，必须靠反射读取

这是本篇最重要的一句话。请看这段代码：

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface PrintHello {
}

public class Demo {
    @PrintHello
    public void foo() {
        System.out.println("foo 执行了");
    }

    public static void main(String[] args) {
        new Demo().foo();   // 输出：foo 执行了
    }
}
```

运行结果是 `foo 执行了`——**`@PrintHello` 什么也没做**。它没有让 `foo()` 自动打印，也没有触发任何魔法。如果希望 `@PrintHello` 真的"打印 Hello"，必须有人去读它：

```java
// 只有这段"读取注解"的代码存在，@PrintHello 才有意义
for (Method m : Demo.class.getDeclaredMethods()) {
    if (m.isAnnotationPresent(PrintHello.class)) {
        System.out.println("Hello");   // 注解真正"生效"的地方
        m.invoke(new Demo());
    }
}
```

这也解释了框架的工作原理：Spring 的 `@Autowired`、`@Transactional`、JUnit 的 `@Test`、MyBatis 的 `@Select`——**它们本身都不干活，是框架里的反射代码在干活**。注解只是和框架约定好的"暗号"。

> **注意**：这条规则有两个例外，都与"反射"无关。一是**编译器内置处理的注解**（`@Override`、`@SuppressWarnings`、`@FunctionalInterface`、`@SafeVarargs`），由 `javac` 在编译期检查；二是**注解处理器**（APT，`javax.annotation.processing`）在编译期读取注解生成新代码，Lombok、MapStruct、Dagger 走的就是这条路。除此之外，所有运行期生效的注解都离不开反射，详见第 19 篇。

# 2. 定义注解的完整语法

## 2.1 @interface 声明

注解用 `@interface` 声明，注意它**不是接口的简写**，而是一个独立的关键字（编译器会把它翻译成继承 `java.lang.annotation.Annotation` 的接口）：

```java
[修饰符列表] @interface 注解类型名 {
    // 属性（也叫"元素"）声明
}
```

```java
// 一个最简单的注解：没有任何属性
public @interface MyAnnotation {
}

// 带属性的注解
public @interface MyAnnotation2 {
    String value();          // 无默认值，使用时必须赋值
    int age() default 0;     // 有默认值，使用时可以省略
    Class<?> clazz() default Object.class;
}
```

> **陷阱**：属性的声明形式长得像方法——`String value();`，**后面必须带一对空括号**，这是语法的硬性要求。写 `String value;` 会编译失败。

## 2.2 属性的类型限制

注解属性的类型**只能是下面这些**，其他一律不允许：

| 允许的类型 | 写法示例 | 说明 |
|-----------|----------|------|
| 八种基本类型 | `int age() default 0;`<br>`boolean enabled() default true;` | 不支持 `Integer`、`Boolean` 这类包装类 |
| `String` | `String name();` | 最常用 |
| `Class` 或 `Class<?>` | `Class<?> clazz() default Object.class;` | 泛型只能写 `Class<?>` 或具体化，不能写 `Class<T>` |
| 枚举类型 | `Level level() default Level.LOW;` | 枚举必须与注解一起可见 |
| 注解类型 | `Retry retry() default @Retry(times = 3);` | 注解可以嵌套注解 |
| 上述类型的**数组** | `String[] likes();`<br>`int[] nums() default {};` | 数组元素类型仍受同样限制 |

```java
// 反例：以下全部编译失败
public @interface Bad {
    // Integer max();          // ✗ 不能是包装类
    // Object data();          // ✗ 不能是 Object
    // List<String> list();    // ✗ 不能是集合
    // String value() { }      // ✗ 不能有方法体
}
```

> **注意**：属性**不能为 `null`**。`String name() default null;` 是编译错误。需要用"没有值"这个语义时，约定用一个空字符串 `""`（Spring 里到处都是 `String value() default "";`），或者用 `@Nullable` 之类的额外注解表达。

## 2.3 属性的默认值与使用规则

四条必须记住的赋值规则：

```java
@Retention(RetentionPolicy.RUNTIME)
public @interface Config {
    String value();                                  // 无默认值，必须显式赋值
    int age() default 18;                            // 有默认值，可以不写
    String[] likes() default {};                     // 数组类型用大括号赋值
    Class<?> clazz() default Object.class;
}
```

| 规则 | 说明 | 示例 |
|------|------|------|
| 没有默认值的属性**必须赋值** | 漏写会编译报错 | `@Config("abc")` ✓<br>`@Config` ✗ |
| 有 `default` 的属性可以不赋值 | 想覆盖默认值时才写 | `@Config(value = "abc")` ✓<br>`@Config(value = "abc", age = 20)` ✓ |
| **属性名为 `value` 且只需要给它赋值时，`value =` 可以省略** | 这是 Spring 里到处写 `@RequestMapping("/user")` 的原因 | `@Config("abc")` 等价于 `@Config(value = "abc")` |
| **数组类型用大括号赋值** | 只有一个元素时也不能省括号（这一点与 Java 数组初始化不同） | `@Config(value = "x", likes = {"a", "b"})`<br>单个元素也必须写 `likes = {"a"}` |

> **陷阱**：`value =` 的省略规则只看"**这次赋值是不是只给 `value` 一个属性赋值**"，与注解定义了几个属性无关。所以 `@Config("abc")` 合法（`age`、`likes` 用默认值），而 `@Config("abc", age = 20)` **非法**——一旦要给多个属性赋值，每一个都必须写成 `属性名 = 值` 的形式，`value` 也不例外。最典型的错误是 `@RequestMapping("/user", method = RequestMethod.GET)`，第二个参数一出现，前面的路径就必须补写成 `value = "/user"`。

## 2.4 一个完整的定义示例

```java
package com.example.anno;

import java.lang.annotation.*;

/**
 * 演示注解属性的各种类型：基本类型、String、Class、枚举、注解、数组。
 */
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
@Documented
public @interface ApiEndpoint {

    /** 路径，无默认值，必须赋值。属性名为 value，可以省略 "value =" */
    String value();

    /** HTTP 方法，枚举类型 */
    HttpMethod method() default HttpMethod.GET;

    /** 是否需要登录 */
    boolean authRequired() default true;

    /** 处理该请求的类，Class 类型 */
    Class<?> handler() default Object.class;

    /** 允许的角色，数组类型 */
    String[] roles() default {};

    /** 超时配置，注解类型 */
    Timeout timeout() default @Timeout;

    /** HTTP 方法枚举，定义在注解内部或外部都可以 */
    enum HttpMethod { GET, POST, PUT, DELETE }

    /** 嵌套注解 */
    @Retention(RetentionPolicy.RUNTIME)
    @interface Timeout {
        int millis() default 3000;
    }
}
```

使用它：

```java
@ApiEndpoint(value = "/user/list", method = ApiEndpoint.HttpMethod.GET, roles = {"admin", "user"})
public class UserController {

    @ApiEndpoint("/user/detail")                     // value = 可省略
    public String detail() { return "detail"; }

    @ApiEndpoint(value = "/user/save", timeout = @ApiEndpoint.Timeout(millis = 5000))
    public String save() { return "saved"; }
}
```

# 3. 元注解

**元注解（meta-annotation）是"用来修饰注解的注解"**——它们决定自定义注解本身能在哪用、活到什么时候、是否可重复。JDK 一共提供六个：`@Target`、`@Retention`、`@Documented`、`@Inherited`、`@Repeatable`、`@Native`（`@Native` 只用于修饰字段，表示该常量可能被本地代码引用，几乎用不到）。

## 3.1 @Target：注解能贴在哪儿

`@Target` 用来标注"被标注的注解可以出现在哪些位置"。它的值是 `ElementType` 枚举数组。

| 取值 | 可以标注的位置 | 典型用途 |
|------|---------------|----------|
| `TYPE` | 类、接口、枚举、注解类型、record | 类级别的注解，如 `@Component` |
| `FIELD` | 字段（成员变量）、枚举常量 | 依赖注入、ORM 映射，如 `@Autowired`、`@Column` |
| `METHOD` | 方法 | 最常用，如 `@Test`、`@RequestMapping` |
| `PARAMETER` | 方法参数 | 参数校验，如 `@RequestParam`、`@NotNull` |
| `CONSTRUCTOR` | 构造方法 | 如 Spring 的 `@Autowired` 构造器注入 |
| `LOCAL_VARIABLE` | 局部变量 | `@SuppressWarnings` 可以贴在局部变量上 |
| `ANNOTATION_TYPE` | 注解类型（即元注解） | 自定义元注解时使用 |
| `PACKAGE` | 包声明（写在 `package-info.java` 里） | 包级别的约定 |
| `TYPE_PARAMETER` | 类型参数（泛型 `<T>`） | JDK8 新增，如 `class Box<@NonNull T>` |
| `TYPE_USE` | 任何**用到类型**的地方 | JDK8 新增，最灵活，可以做类型检查工具（Checker Framework） |
| `MODULE` | 模块声明（`module-info.java`） | JDK9 新增 |
| `RECORD_COMPONENT` | record 的组件（头部参数） | JDK16 新增 |

```java
// 多重取值用大括号
@Target({ElementType.METHOD, ElementType.FIELD, ElementType.PARAMETER})
public @interface NotNull {
}

// 单个取值可以省略大括号
@Target(ElementType.CONSTRUCTOR)
public @interface Inject {
}
```

> **注意**：**不写 `@Target` 时，这个注解可以贴在几乎所有位置**（除 `TYPE_PARAMETER`、`TYPE_USE`、`MODULE` 等 JDK8 之后新增的位置外，哪些能用取决于 JDK 版本）。虽然"不写"更省事，但**推荐显式写出**——它既是文档，也能让误用变成编译错误。比如 `@MyTest` 明确写了 `@Target(ElementType.METHOD)`，别人把它贴到类上就会立刻报错。

## 3.2 @Retention：注解活到什么时候

`@Retention` 用来标注"被标注的注解最终保存在哪里"，由 `RetentionPolicy` 枚举控制，**这是三个元注解里最需要理解底层的一个**。

| 取值 | 保存在哪 | 编译期可见 | 运行期反射可见 | 典型例子 |
|------|----------|-----------|---------------|----------|
| `SOURCE` | 仅保存在 `.java` 源文件中，**编译后丢弃** | 是 | 否 | `@Override`、`@SuppressWarnings`、Lombok 的 `@Data` |
| `CLASS` | 保存在 `.class` 文件里（`RuntimeInvisibleAnnotations` 属性），**但 JVM 运行时不加载** | 是 | 否 | 这是**不写 `@Retention` 时的默认值**，如 `@NonNull` |
| `RUNTIME` | 保存在 `.class` 文件里（`RuntimeVisibleAnnotations` 属性），**运行期可被反射读取** | 是 | **是** | `@Deprecated`、Spring 的 `@Component`、JUnit 的 `@Test` |

```java
@Retention(RetentionPolicy.SOURCE)   // 编译完就没了，只能用于编译期检查
public @interface Override { }

@Retention(RetentionPolicy.RUNTIME)  // 必须在 .class 里留存，反射才能读到
public @interface Autowired { }
```

> **陷阱**：**自定义注解几乎都要写 `@Retention(RetentionPolicy.RUNTIME)`**。因为常见的忘记写法是干脆不写 `@Retention`，此时默认是 `CLASS`——注解确实进了 class 文件，但 `method.getAnnotation(Xxx.class)` 永远返回 `null`，程序不报任何错，只是逻辑静默失效。这是注解开发中最隐蔽的一类 bug。

> **行业实践**：只有两种情况下才用 `SOURCE`——一是给编译器看的检查注解（`@Override`），二是**编译期代码生成**（Lombok、MapStruct、Dagger 的 APT）。`CLASS` 的典型用户是字节码分析工具（ASM、Byte Buddy 写的 agent 在运行期直接读 class 文件的属性区）。

## 3.3 @Documented

被 `@Documented` 标注的注解，会出现在 `javadoc` 生成的 API 文档里。如果不加，该注解在文档中不可见。

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface ApiEndpoint { }
```

`@Deprecated` 就带 `@Documented`，所以 javadoc 里会显示"已过时"的标记。这个元注解对程序行为没有任何影响，纯粹是文档层面的。

## 3.4 @Inherited

`@Inherited` 表示该注解**可以被子类继承**。注意它有两个重要限制：

1. **只对类继承有效，对接口实现无效**——实现类不会继承接口上的注解；
2. **只对类级别的注解有效**，方法、字段上的注解永远不会被"继承"（子类重写方法不会带上父类方法的注解）。

```java
@Inherited
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface InheritedAnno { }

@InheritedAnno
public class Parent { }

public class Child extends Parent { }

// 反射验证
System.out.println(Child.class.isAnnotationPresent(InheritedAnno.class));  // true
```

> **注意**：Spring 的 `@Transactional` 就没有用 `@Inherited`，但它通过自己的 `AnnotationUtils.findAnnotation()` 实现了对父类/接口注解的查找。**框架自己实现查找逻辑比依赖 `@Inherited` 更灵活**，所以实际项目里 `@Inherited` 用得并不多。

## 3.5 @Repeatable：可重复注解

JDK8 之前，同一个位置不能贴两个相同的注解。JDK8 引入 `@Repeatable` 解决了这个问题，代价是**必须再定义一个"容器注解"**：

```java
import java.lang.annotation.*;

/** 容器注解：属性名必须是 value，类型是被容纳注解的数组 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Schedules {
    Schedule[] value();
}

/** 可重复注解：通过 @Repeatable 指向容器 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Repeatable(Schedules.class)
public @interface Schedule {
    String dayOfWeek();
    int hour() default 9;
}
```

使用与读取：

```java
// 写法一：直接重复贴多次（JDK8 之后允许的语法）
@Schedule(dayOfWeek = "周一")
@Schedule(dayOfWeek = "周三", hour = 14)
public class WeeklyReport {
}

// 写法二（等价的老写法）：显式使用容器注解
@Schedules({
    @Schedule(dayOfWeek = "周一"),
    @Schedule(dayOfWeek = "周三", hour = 14)
})
public class WeeklyReport2 {
}
```

```java
// 反射有两种读法，结果不同：
Schedule[] arr = WeeklyReport.class.getAnnotationsByType(Schedule.class);   // 推荐：自动"拆包"
System.out.println(arr.length);                                             // 2

Schedules container = WeeklyReport.class.getAnnotation(Schedules.class);     // 老写法：手动拆包
System.out.println(container.value().length);                                // 2
```

> **注意**：`@Repeatable` 只是**编译期语法糖**——`javac` 会把多个 `@Schedule` 自动包装进 `@Schedules`。所以 `getAnnotation(Schedule.class)`（不带 `ByType`）永远返回 `null`，必须用 `getAnnotationsByType()` 或先取容器再取 `value()`。这个细节是"注解明明写了却读不到"的常见原因。

# 4. JDK 内置注解

## 4.1 @Override

`@Override` 是一个**编译检查注解**，`@Retention` 为 `SOURCE`，编译后就不存在了。它没有任何运行时行为，价值全在编译期的保护：

```java
public class Parent {
    public void eat() { System.out.println("Parent eat"); }
}

public class Child extends Parent {
    @Override
    public void eat() { System.out.println("Child eat"); }      // ✓ 确实重写了

    // @Override
    // public void eat(String food) { }                          // ✗ 编译报错：父类没有这个方法

    // 不加 @Override 的灾难场景：
    public void eat(String food) { }    // 本意是重写，实际写成了重载！父类方法照旧被调用
}
```

**核心价值：防止"想重写却写成了重载"。** 少写一个参数、把参数类型写错、方法名拼错（`equals` 写成 `equal`），这些错误不加 `@Override` 时编译器一声不吭，只在运行期表现为"我的方法怎么没被调用"。加上 `@Override`，编译器立刻报错。

> **最佳实践**：**只要是在重写方法，就无条件加上 `@Override`**。现代 IDE 都能自动补全，成本为零。接口实现类和抽象方法实现同样适用——JDK6 之后 `@Override` 也允许用在实现接口方法上。

## 4.2 @Deprecated

标记某个程序元素"已过时，不推荐使用"。`@Retention` 为 `RUNTIME`，所以运行期也读得到（框架可以据此打警告日志）。

```java
@Deprecated
public void oldMethod() { }

// 带属性的形式（JDK9 之后）
@Deprecated(since = "1.5", forRemoval = true)
public void toBeRemoved() { }
```

| 属性 | 类型 | 默认值 | 含义 |
|------|------|--------|------|
| `since` | `String` | `""` | 从哪个版本开始过时，写版本号字符串 |
| `forRemoval` | `boolean` | `false` | 是否**计划在未来版本删除**；设为 `true` 时，调用处会出现更严厉的警告 |

```bash
# 编译时开启 lint，让"过时用法"变成可见的警告（-Xlint:deprecation）
javac -Xlint:deprecation -Xlint:removal Demo.java
```

```text
Demo.java:8: 警告: [deprecation] Demo 中的 oldMethod() 已过时
Demo.java:9: 警告: [removal] Demo 中的 toBeRemoved() 已过时且标记为待移除
```

> **行业实践**：`@Deprecated` 是**对外承诺的"即将废弃"信号**，不要随手加。加了之后应当在 javadoc 里用 `@deprecated` 标签写清"替代方案是什么、从哪个版本开始、计划何时删除"。只加注解不给替代方案，等于给使用者挖了个坑。

## 4.3 @SuppressWarnings

抑制编译器警告。`@Retention` 为 `SOURCE`，只在编译期起作用。

```java
// 最经典的场景：泛型数组创建，不抑制就是一堆 unchecked 警告
@SuppressWarnings("unchecked")
public <T> T[] toArray(List<T> list) {
    return (T[]) list.toArray();
}
```

常见取值：

| 取值 | 抑制的警告 |
|------|-----------|
| `"unchecked"` | 未经检查的类型转换（泛型相关），最常用 |
| `"rawtypes"` | 使用原始类型（如 `List` 而不是 `List<String>`） |
| `"deprecation"` | 调用了 `@Deprecated` 的元素 |
| `"removal"` | 调用了 `forRemoval = true` 的元素 |
| `"unused"` | 未使用的变量、方法、类 |
| `"serial"` | 可序列化类缺少 `serialVersionUID` |
| `"all"` | 抑制全部警告，**慎用** |
| `"fallthrough"` | `switch` 语句缺少 `break` 造成的穿透 |

```java
// 多个警告一起抑制
@SuppressWarnings({"unchecked", "rawtypes", "deprecation"})
public void legacy() { }
```

> **最佳实践**：`@SuppressWarnings` 的作用范围**越小越好**——能放在局部变量上就不要放在方法上，能放在方法上就不要放在类上，永远不要用 `@SuppressWarnings("all")` 把整类的警告关掉。抑制警告前先想清楚"这个警告为什么存在"，抑制的理由应该写一句注释在旁边。

## 4.4 @FunctionalInterface

标记一个接口是**函数式接口**（有且仅有一个抽象方法），`@Retention` 为 `RUNTIME`。它的作用是编译期检查：

```java
@FunctionalInterface
public interface Calculator {
    int calc(int a, int b);
}

// ✗ 编译错误：多个抽象方法，不再是函数式接口
// @FunctionalInterface
// interface Bad {
//     int a();
//     int b();
// }
```

JDK 中大量接口都带这个注解：`Runnable`、`Comparator`、`Callable`、`Function`、`Predicate`、`Consumer`、`Supplier` 等。它的价值在于**一旦有人不小心往里加了第二个抽象方法，编译立刻失败**，从而保护了所有用 lambda 的地方。详见第 12 篇。

## 4.5 @SafeVarargs

抑制"泛型可变参数"造成的堆污染警告。泛型可变参数 `T...` 在底层是 `Object[]`，类型不安全，编译器会给出警告；当你能**保证不会写入非法元素**时，用 `@SafeVarargs` 声明"我知道风险，这里是安全的"。

```java
@SafeVarargs                       // 有这个注解，调用处就不会再有警告
public static <T> List<T> of(T... elements) {
    List<T> list = new ArrayList<>();
    Collections.addAll(list, elements);   // 只读不写，不会污染数组
    return list;
}
```

使用限制：`@SafeVarargs` 只能用在**静态方法、final 实例方法、构造方法或私有实例方法（JDK9+）**上——因为这些方法**不可被重写**，编译器才能确信注解的承诺不会被破坏。

```java
// ✗ 编译错误：非 final 的实例方法可能被子类重写，无法保证安全
// @SafeVarargs
// public <T> void bad(T... args) { }
```

## 4.6 JDK 内置注解总表

| 注解 | `@Retention` | 处理时机 | 作用 |
|------|-------------|----------|------|
| `@Override` | `SOURCE` | 编译期 | 检查是否真的重写了父类/接口方法 |
| `@Deprecated` | `RUNTIME` | 编译期警告 + 运行期可读 | 标记过时元素，`since` / `forRemoval` 属性可细粒度描述 |
| `@SuppressWarnings` | `SOURCE` | 编译期 | 抑制指定的编译警告 |
| `@FunctionalInterface` | `RUNTIME` | 编译期 | 检查接口只有一个抽象方法 |
| `@SafeVarargs` | `RUNTIME` | 编译期 | 声明泛型可变参数不会造成堆污染 |
| `@Native` | `SOURCE` | 编译期 | 标记常量字段可能被本地代码引用（几乎不用） |
| `@Retention` / `@Target` / `@Documented` / `@Inherited` / `@Repeatable` | `RUNTIME` | 元注解 | 用来修饰其他注解 |

# 5. 实战一：用自定义注解写一个简陋的测试框架

这是把注解讲通的最好方式——JUnit 的核心原理不过如此。

## 5.1 定义注解

```java
package com.example.mytest;

import java.lang.annotation.*;

/**
 * 模仿 JUnit 的 @Test。属性 value 用来给用例起名字，
 * 因为有默认值，可以不写。
 */
@Retention(RetentionPolicy.RUNTIME)     // 必须 RUNTIME，否则反射读不到
@Target(ElementType.METHOD)             // 只能贴在方法上
@Documented
public @interface MyTest {
    String value() default "";
}
```

## 5.2 反射扫描并执行

```java
package com.example.mytest;

import java.lang.reflect.Method;

/**
 * 极简测试运行器：扫描被测类里所有带 @MyTest 的方法并依次调用。
 * 这就是 JUnit、TestNG 的工作原理骨架。
 */
public class MyTestRunner {

    public static void run(Class<?> testClass) throws Exception {
        int passed = 0, failed = 0;

        // 反射创建被测类的实例（框架调用方法的"靶子"）
        Object instance = testClass.getDeclaredConstructor().newInstance();

        for (Method method : testClass.getDeclaredMethods()) {
            // 只处理贴了 @MyTest 的方法，其他方法一律跳过
            if (!method.isAnnotationPresent(MyTest.class)) {
                continue;
            }
            method.setAccessible(true);                  // 私有方法也能调用，模拟框架的行为

            MyTest anno = method.getAnnotation(MyTest.class);
            String name = anno.value().isEmpty() ? method.getName() : anno.value();

            try {
                method.invoke(instance);                 // 调用被测方法
                System.out.println("通过 ✓ " + name);
                passed++;
            } catch (Exception e) {
                // 反射调用抛出的异常会被包一层 InvocationTargetException
                Throwable cause = e.getCause() == null ? e : e.getCause();
                System.out.println("失败 ✗ " + name + "：" + cause.getMessage());
                failed++;
            }
        }

        System.out.println("\n共 " + (passed + failed) + " 个用例，通过 " + passed + " 个，失败 " + failed + " 个");
    }

    public static void main(String[] args) throws Exception {
        run(DemoTest.class);
    }
}
```

## 5.3 被测类

```java
package com.example.mytest;

public class DemoTest {

    @MyTest("加法应该算对")
    public void testAdd() {
        if (1 + 1 != 2) {
            throw new AssertionError("加法算错了");
        }
    }

    @MyTest("这个用例会故意失败")
    public void testFail() {
        throw new AssertionError("我故意抛出的失败");
    }

    @MyTest                                  // 不写 value，用默认值，名字回退到方法名
    public void testNothing() {
        // 什么都不做，算通过
    }

    // 没有 @MyTest，运行器不会碰它
    public void notATest() {
        System.out.println("这一行永远不该被打印");
    }
}
```

## 5.4 运行结果

```text
通过 ✓ 加法应该算对
失败 ✗ 这个用例会故意失败：我故意抛出的失败
通过 ✓ testNothing

共 3 个用例，通过 2 个，失败 1 个
```

> **注意**：`getDeclaredMethods()` 返回的顺序**不保证与源码顺序一致**（JVM 规范未做要求），所以上面的输出顺序在不同 JDK 上可能不同。真实的测试框架会用 `@Order` 之类的注解显式排序，或者干脆不保证顺序。

> **行业实践**：真实的 JUnit 比这复杂得多——`@BeforeEach` / `@AfterEach` 的生命周期钩子、`@ParameterizedTest` 的参数化、超时控制、嵌套测试类、异常断言，都是在这套"扫描 + 反射调用"的骨架上不断加注解实现的。**读完这段代码，再去看 JUnit 的源码，会发现骨架完全一样**。

# 6. 实战二：用注解实现字段校验

第二个例子更贴近实际业务——参数校验是注解在 Web 开发中最常见的用途之一（Hibernate Validator / `@Valid` 就是它的工业级版本）。

## 6.1 定义校验注解

```java
package com.example.validate;

import java.lang.annotation.*;

/** 非空校验 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface NotNull {
    String message() default "字段不能为 null";
}
```

```java
package com.example.validate;

import java.lang.annotation.*;

/** 数值范围校验 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface Range {
    int min() default Integer.MIN_VALUE;
    int max() default Integer.MAX_VALUE;
    String message() default "数值超出范围";
}
```

## 6.2 校验器实现

```java
package com.example.validate;

import java.lang.reflect.Field;
import java.util.ArrayList;
import java.util.List;

/**
 * 极简字段校验器：遍历所有字段，读取其上的校验注解并逐条检查。
 */
public class Validator {

    public static List<String> validate(Object obj) throws IllegalAccessException {
        List<String> errors = new ArrayList<>();

        for (Field field : obj.getClass().getDeclaredFields()) {
            field.setAccessible(true);
            Object value = field.get(obj);        // 反射读取字段值

            // 规则一：@NotNull
            if (field.isAnnotationPresent(NotNull.class) && value == null) {
                errors.add(field.getName() + "：" + field.getAnnotation(NotNull.class).message());
            }

            // 规则二：@Range（只对 Integer 类型字段生效）
            Range range = field.getAnnotation(Range.class);
            if (range != null && value instanceof Integer num) {   // 模式匹配，见第 29 篇
                if (num < range.min() || num > range.max()) {
                    errors.add(field.getName() + "：" + range.message()
                            + "（应在 " + range.min() + " ~ " + range.max() + " 之间，实际是 " + num + "）");
                }
            }
        }
        return errors;
    }
}
```

## 6.3 被校验的类与运行结果

```java
package com.example.validate;

public class UserForm {

    @NotNull(message = "用户名不能为空")
    private String name;

    @Range(min = 0, max = 150, message = "年龄不合理")
    private Integer age;

    public UserForm(String name, Integer age) {
        this.name = name;
        this.age = age;
    }
}
```

```java
public class Main {
    public static void main(String[] args) throws Exception {
        // 非法数据：name 为 null，age 超出范围
        System.out.println(Validator.validate(new UserForm(null, 200)));

        // 合法数据：返回空列表，没有错误
        System.out.println(Validator.validate(new UserForm("张三", 30)));
    }
}
```

```text
[用户名不能为空, age：年龄不合理（应在 0 ~ 150 之间，实际是 200）]
[]
```

> **最佳实践**：两个例子都遵循同一个套路，值得抽象成一条方法论：**注解负责"声明规则"，反射负责"执行规则"**，两者通过 `getAnnotation` / `isAnnotationPresent` 接口对接。写自定义注解时先想清楚两件事——"这个注解会被谁读"、"读不到会怎样"，能省掉后面一大半调试时间。

# 7. 框架为什么重度依赖注解

理解了"注解 + 反射"这条链路，框架的很多行为就变得透明了。

## 7.1 Spring

Spring 的核心容器（`ApplicationContext`）启动时会扫描指定包下所有类：

| 注解 | 框架读到之后做什么 |
|------|-------------------|
| `@Component` / `@Service` / `@Repository` / `@Controller` | 反射创建实例，放进容器（IoC） |
| `@Autowired` | 反射找到字段/构造器，从容器里取出匹配的 Bean 注入进去 |
| `@Value("${...}")` | 读取配置后反射写入字段 |
| `@RequestMapping` / `@GetMapping` | 解析路径，注册到 `HandlerMapping` 映射表，请求来了靠它路由 |
| `@Transactional` | 创建代理对象，在方法前后织入事务开启/提交/回滚 |
| `@Configuration` + `@Bean` | 识别配置类，反射调用 `@Bean` 方法并把返回值注册为 Bean |

## 7.2 MyBatis

| 注解 | 作用 |
|------|------|
| `@Select` / `@Insert` / `@Update` / `@Delete` | 方法上的 SQL 字符串被读取后，生成 MappedStatement 绑定到该方法 |
| `@Param` | 指定参数名，供 `${}` 和 `#{}` 占位符引用 |
| `@Results` / `@Result` | 描述字段与列的映射关系，替代 XML 的 `<resultMap>` |
| `@Mapper` | 标记接口，MyBatis 为其生成动态代理实现 |

## 7.3 JUnit

| 注解 | 作用 |
|------|------|
| `@Test` | 标记一个方法是用例 |
| `@BeforeEach` / `@AfterEach` | 每个用例前后的钩子 |
| `@BeforeAll` / `@AfterAll` | 全类只执行一次的钩子 |
| `@Disabled` | 跳过用例 |
| `@DisplayName` | 给用例起个可读的名字（正是 5.1 节那个 `String value()` 的工业级版本） |

三者共同点非常清楚：**注解声明"这是什么"，框架的反射代码执行"该怎么做"**。这也解释了为什么这些框架都要求注解是 `RUNTIME` 的——它们的读取发生在运行期，用的是反射。

> **注意**：注解本身**不能替代抽象**。滥用注解会出现"程序员在注解里写逻辑"的反模式，比如用一堆 `@WhenCondition(...)` 表达业务分支——这类逻辑用注解表达既无法调试也无法单测。判断标准是：**注解应该描述"声明式的配置"，而不是"命令式的流程"**。

# 8. 小结

| 主题 | 一句话要点 |
|------|-----------|
| 注解是什么 | 一种元数据，是引用数据类型，编译后生成 `Xxx.class`；**本身不改变程序行为，必须靠反射读取才有意义** |
| 注解 vs 注释 | 注释给人看、编译即丢弃；注解给程序看、可进 class 文件、可被反射读到 |
| 定义语法 | `[修饰符] @interface 名字 { ... }`；属性声明要带空括号，如 `String value();` |
| 属性类型 | 只能是**基本类型、`String`、`Class`、枚举、注解、以及它们的数组**；属性不能为 `null` |
| 属性赋值 | 无默认值必须赋值；有 `default` 可省略；名为 `value` 且只给它赋值时可省 `value =`；数组用 `{}` |
| `@Target` | 决定注解能贴在哪儿；取值有 `TYPE`/`FIELD`/`METHOD`/`PARAMETER`/`CONSTRUCTOR`/`LOCAL_VARIABLE`/`ANNOTATION_TYPE`/`PACKAGE`/`TYPE_PARAMETER`/`TYPE_USE`/`MODULE`/`RECORD_COMPONENT`；不写则几乎所有位置都能用 |
| `@Retention` | `SOURCE`（如 `@Override`，编译后丢弃）/ `CLASS`（**默认**，进 class 但运行期不可见）/ `RUNTIME`（反射可见，**自定义注解几乎都用它**） |
| 其他元注解 | `@Documented` 进 javadoc、`@Inherited` 只对类继承有效、`@Repeatable` 需配一个容器注解，读取用 `getAnnotationsByType` |
| 内置注解 | `@Override` 防"想重写写成重载"；`@Deprecated` 支持 `since`/`forRemoval`；`@SuppressWarnings` 取值范围越小越好；`@FunctionalInterface`、`@SafeVarargs` 都是编译期保护 |
| 框架为什么用注解 | 注解声明"是什么"，框架的反射代码执行"怎么做"；Spring / MyBatis / JUnit 全部遵循这个套路 |

注解是"声明式编程"在 Java 里的载体，它的另一半是反射——想彻底看懂 `@Autowired` 怎么把 Bean 塞进字段，就要回头看第 19 篇的 `Field#set` 与 `Method#invoke`。下一篇（第 26 篇）讲 `record`，看 Java 如何用语言层面的不可变数据载体，把过去必须手写的样板代码一次消灭。
