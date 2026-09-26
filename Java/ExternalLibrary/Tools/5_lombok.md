# 1. 概述与简介

Lombok 是一个 Java 编译器级别的代码生成库，通过注解自动生成 getter、setter、equals、hashCode、toString、构造器等方法，大幅减少样板代码（boilerplate code），提高开发效率和代码可读性。

## 1.1 核心设计理念

- **消除样板代码**：自动生成 Java Bean 中重复性高的方法
- **编译时生成**：在编译阶段修改 AST（抽象语法树），不污染运行时
- **注解驱动**：通过在代码中添加注解来控制代码生成
- **可配置性**：通过 `lombok.config` 配置文件自定义行为
- **IDE 友好**：主流 IDE 均有插件支持，提供实时预览

## 1.2 Maven 依赖

```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.30</version>
    <scope>provided</scope>  <!-- 不会打入最终包 -->
</dependency>
```

Gradle：

```groovy
compileOnly 'org.projectlombok:lombok:1.18.30'
annotationProcessor 'org.projectlombok:lombok:1.18.30'
```

## 1.3 工作原理概览

```
Java 源码（含 Lombok 注解）
        ↓
   Java Compiler (javac)
        ↓
  Lombok Annotation Processor
    (JSR 269 Pluggable Annotation Processing API)
        ↓
  AST (抽象语法树) 修改
    - 添加 getter/setter 方法节点
    - 添加构造器方法节点
    - 添加 toString/equals/hashCode 方法节点
        ↓
  生成 .class 字节码（包含生成的方法）
        ↓
  JVM 加载执行（完全不知道 Lombok 的存在）
```

---

# 2. 核心注解与用法

## 2.1 @Getter 和 @Setter

最基础的注解，自动生成 getter 和 setter 方法。

```java
import lombok.AccessLevel;
import lombok.Getter;
import lombok.Setter;

public class User {

    @Getter @Setter  // 生成 getter 和 setter
    private String name;

    @Getter @Setter(AccessLevel.PROTECTED)  // setter 访问级别为 protected
    private int age;

    @Getter(AccessLevel.NONE)  // 不生成 getter
    @Setter
    private String secret;

    @Setter(AccessLevel.PRIVATE)  // setter 为 private
    private String internalOnly;

    // 类级别：对所有非静态字段生效
}

// ========== 类级别使用 ==========
@Getter
@Setter
public class Employee {
    private String name;       // 自动生成 getter/setter
    private double salary;     // 自动生成 getter/setter
    @Getter(AccessLevel.NONE)  // 覆盖类级别配置
    private String ssn;        // 不生成 getter
}

// 生成的代码等价于：
// public String getName() { return name; }
// public void setName(String name) { this.name = name; }
// public double getSalary() { return salary; }
// public void setSalary(double salary) { this.salary = salary; }
// 布尔类型：isActive() 而非 getActive()
```

### Getter/Setter 命名规则

| 字段类型 | 字段名 | 生成的 getter | 生成的 setter |
|---------|--------|---------------|---------------|
| 非 boolean | name | getName() | setName() |
| boolean | active | isActive() | setActive() |
| boolean (Optional) | active | getActive() | setActive() |
| boolean | isActive | isActive() | setActive() |

## 2.2 @ToString

```java
import lombok.ToString;

@ToString   // 生成 toString() 方法
public class User {

    private String name;

    @ToString.Exclude  // 排除此字段
    private String password;

    @ToString.Include(name = "nickname")  // 自定义字段名
    private String nick;

    @ToString.Include(rank = 1)  // 控制输出顺序（数字越小越靠前）
    private int age;
}

// 生成的 toString 示例：
// User(name=张三, nickname=小张, age=25)
// （password 被排除）

// ========== 调用父类 toString ==========
@ToString(callSuper = true)
public class AdminUser extends User {
    private String role;
}
// AdminUser(super=User(name=张三, ...), role=admin)

// ========== 只包含指定字段 ==========
@ToString(onlyExplicitlyIncluded = true)
public class Person {
    @ToString.Include
    private String name;
    private String idCard;  // 被排除
}
```

## 2.3 @EqualsAndHashCode

```java
import lombok.EqualsAndHashCode;

@EqualsAndHashCode  // 生成 equals() 和 hashCode()
public class User {

    private Long id;
    private String name;

    @EqualsAndHashCode.Exclude  // 排除此字段
    private int age;

    @EqualsAndHashCode.Include(replaces = "name")  // 用其他字段替换
    private String nameLower;
}

// ========== 调用父类 ==========
@EqualsAndHashCode(callSuper = true)
public class AdminUser extends User {
    private String role;
}
// 比较时包括父类的字段

// ========== 警告：使用 @Data 时可能 vs 继承的陷阱 ==========
// 见 6.2 常见坑
```

## 2.4 @Data

`@Data` 是 `@Getter` + `@Setter` + `@ToString` + `@EqualsAndHashCode` + `@RequiredArgsConstructor` 的组合注解。

```java
import lombok.Data;

@Data  // 包含以下所有注解的组合
public class User {
    private Long id;
    private String name;
    private int age;
    private String email;
}

// 等价于：
// @Getter
// @Setter
// @ToString
// @EqualsAndHashCode
// @RequiredArgsConstructor
```

### 注意：@Data 的默认行为

```java
@Data
public class User {
    private final Long id;    // final 字段 -> @RequiredArgsConstructor 生成构造器参数
    private String name;
    private String email;
}

// 生成的构造器：User(Long id)
// 只有 final 字段才会被 RequiredArgsConstructor 包含
```

## 2.5 @Value（不可变类）

```java
import lombok.Value;
import lombok.With;

@Value  // 生成所有字段的 final + getter + toString + equals + hashCode + AllArgsConstructor
public class ImmutableUser {
    String name;
    int age;

    // @With 可生成 withXxx 方法（返回修改后的新实例）
    @With
    String email;
}

// @Value 等价于：
// - 所有字段 private final
// - @Getter (没有 setter)
// - @ToString
// - @EqualsAndHashCode
// - @AllArgsConstructor

// 使用示例：
ImmutableUser user = new ImmutableUser("张三", 25, "zhangsan@example.com");
String name = user.getName();  // getter
// user.setName("李四");      // 编译错误！没有 setter

ImmutableUser updated = user.withEmail("new@example.com");
// 返回一个新对象，其他字段值相同
```

## 2.6 @Builder

最强大的注解之一，实现建造者模式。

```java
import lombok.Builder;
import lombok.Singular;

@Builder  // 生成 Builder 内部类
public class Order {

    private String orderId;
    private String customerName;

    @Builder.Default  // 设置默认值
    private String status = "PENDING";

    @Singular  // 自动生成单个添加方法
    private List<String> items;

    @Builder.ObtainVia(method = "calculateTotal")  // 从方法获取
    private BigDecimal total;
}

// ========== 使用 Builder ==========
Order order = Order.builder()
        .orderId("ORD-001")
        .customerName("张三")
        .status("PAID")
        .item("Java 编程思想")    // @Singular 生成单数形式方法
        .item("深入理解 JVM")
        .item("Effective Java")
        .build();

// ========== 带有必填字段的 Builder ==========
@Builder(builderMethodName = "strictBuilder")
public class User {
    @Builder.Default
    private String name = "default";

    @Builder.ObtainVia(field = "name")
    private String displayName;
}

// ========== toBuilder 模式 ==========
@Builder(toBuilder = true)  // 生成从实例创建 Builder 的方法
public class Product {
    private String id;
    private String name;
    private BigDecimal price;
}

Product original = Product.builder()
        .id("P001")
        .name("笔记本电脑")
        .price(new BigDecimal("5999.00"))
        .build();

// 基于已有实例修改部分字段
Product modified = original.toBuilder()
        .price(new BigDecimal("5499.00"))
        .build();
// 新实例，id 和 name 相同，price 不同

// ========== 静态 Builder ==========
// 允许在静态方法上使用 @Builder 用于复杂对象构建
public class ApiResult {

    private int code;
    private String message;
    private Object data;

    @Builder(builderClassName = "SuccessBuilder")
    public static ApiResult success(Object data) {
        ApiResult result = new ApiResult();
        result.code = 200;
        result.message = "success";
        result.data = data;
        return result;
    }
}
```

## 2.7 @AllArgsConstructor / @NoArgsConstructor / @RequiredArgsConstructor

```java
import lombok.AllArgsConstructor;
import lombok.NoArgsConstructor;
import lombok.RequiredArgsConstructor;

@NoArgsConstructor        // 生成无参构造器
@AllArgsConstructor      // 生成全参构造器（所有字段）
@RequiredArgsConstructor // 生成包含 final 和 @NonNull 字段的构造器
public class User {

    private Long id;         // 被 AllArgsConstructor 包含
    private String name;     // 被 AllArgsConstructor 包含

    @NonNull
    private String email;    // 被 RequiredArgsConstructor 包含（并且会判空）

    private final String type = "NORMAL";  // 被 RequiredArgsConstructor 包含
}

// ========== 访问级别控制 ==========
@NoArgsConstructor(access = AccessLevel.PRIVATE)  // 私有构造器
public class Config {
    // 用于工具类或单例模式
}

// ========== 静态工厂方法 ==========
@NoArgsConstructor(staticName = "createEmpty")
@AllArgsConstructor(staticName = "of")
public class Order {
    private String id;
    private String name;
    private BigDecimal amount;
}

// 使用：
Order empty = Order.createEmpty();
Order order = Order.of("ORD-001", "测试订单", new BigDecimal("100"));
```

## 2.8 @Slf4j 及日志变种

```java
import lombok.extern.slf4j.Slf4j;
import lombok.extern.log4j.Log4j2;
import lombok.extern.java.Log;

@Slf4j  // SLF4J 日志（最常用）
public class UserService {

    public void createUser(User user) {
        log.info("创建用户: {}", user.getName());  // 直接使用 log 变量
        try {
            // 业务逻辑
        } catch (Exception e) {
            log.error("创建用户失败", e);
        }
    }
}

// 日志框架变种：
// @Log         - java.util.logging
// @Log4j       - Apache Log4j
// @Log4j2      - Apache Log4j 2
// @Slf4j       - SLF4J
// @XSlf4j      - SLF4J + 扩展
// @CommonsLog  - Apache Commons Logging
// @Flogger     - Google Flogger
// @CustomLog   - 自定义日志框架

// 生成的等价代码：
// private static final org.slf4j.Logger log =
//     org.slf4j.LoggerFactory.getLogger(UserService.class);
```

## 2.9 @NonNull

```java
import lombok.NonNull;

public class UserService {

    // 方法参数判空
    public void createUser(@NonNull User user) {
        // user != null，否则抛出 NullPointerException
        // 并在异常信息中包含参数名 "user"
        saveUser(user);
    }

    // 字段级别（配合构造器）
    @NonNull
    private String name;

    // 生成的全参构造器会包含判空：
    // public User(@NonNull String name) {
    //     if (name == null) throw new NullPointerException("name");
    //     this.name = name;
    // }
}
```

## 2.10 @With

```java
import lombok.With;

@With  // 为所有字段生成 withXxx 方法
public class ImmutableConfig {
    private final String key;
    private final String value;
    private final boolean enabled;
}

// 使用：
ImmutableConfig config = new ImmutableConfig("timeout", "5000", true);
ImmutableConfig updated = config.withValue("3000");
// 返回新对象，key 和 enabled 保持不变，value 更新为 "3000"

// 等价于手动实现：
// public ImmutableConfig withValue(String value) {
//     return new ImmutableConfig(this.key, value, this.enabled);
// }
```

## 2.11 @Singular（在 @Builder 中使用）

```java
@Builder
public class ShoppingCart {

    @Singular  // 生成 item() 单数添加方法
    private List<String> items;

    @Singular("addTag")  // 自定义方法名
    private Set<String> tags;

    @Singular("addEntry")  // 用于 Map
    private Map<String, Integer> quantities;
}

// 生成的 Builder 方法：
// ShoppingCartBuilder item(String item)           // 添加单个
// ShoppingCartBuilder items(Collection<String>)   // 添加多个
// ShoppingCartBuilder clearItems()                // 清空
// 类似地：addTag(), tags(), clearTags()
// 对于 Map：addEntry(String key, Integer value), entries(Map), clearEntries()

ShoppingCart cart = ShoppingCart.builder()
        .item("苹果")
        .item("香蕉")
        .addTag("水果")
        .addTag("生鲜")
        .addEntry("苹果", 5)
        .addEntry("香蕉", 3)
        .build();
```

## 2.12 @Accessors

控制 getter/setter 的生成风格（需要单独引入 experimental 包或使用标准版）。

```java
import lombok.experimental.Accessors;

@Accessors(chain = true)  // setter 返回 this，支持链式调用
public class User {

    private String name;
    private int age;

    // 生成的 setter：
    // public User setName(String name) { this.name = name; return this; }
    // public User setAge(int age) { this.age = age; return this; }
}

// 链式调用：
User user = new User()
        .setName("张三")
        .setAge(25);

// fluent = true：去掉 set 前缀
@Accessors(fluent = true)
public class FluentUser {
    private String name;

    // 生成的：public FluentUser name(String name) { ... }
    // 同时 getter 也会变成：public String name() { ... }
}

FluentUser fu = new FluentUser()
        .name("张三");
String name = fu.name();  // 不是 getName！
```

## 2.13 @UtilityClass

```java
import lombok.experimental.UtilityClass;

@UtilityClass  // 工具类注解
public class StringUtils {

    // 自动：
    // 1. 所有字段和方法的 static 关键字（字段变为 final）
    // 2. 私有构造器
    // 3. 防止继承（final class）

    public boolean isEmpty(String str) {
        return str == null || str.isEmpty();
    }
}

// 生成的等价代码：
// public final class StringUtils {
//     private StringUtils() { throw new UnsupportedOperationException(); }
//     public static boolean isEmpty(String str) { ... }
// }
```

---

# 3. 高级特性

## 3.1 @Builder 的深入用法

### 构建器继承

```java
import lombok.Builder;
import lombok.experimental.SuperBuilder;

@SuperBuilder  // 支持继承的 Builder
public class Animal {
    private String name;
    private int age;
}

@SuperBuilder
public class Dog extends Animal {
    private String breed;
}

// 使用：
Dog dog = Dog.builder()
        .name("旺财")
        .age(3)
        .breed("金毛")
        .build();
```

### 自定义 Builder 方法

```java
@Builder
public class Pagination {

    private int page;
    private int size;
    private String sort;

    // 添加自定义验证
    public static class PaginationBuilder {
        public Pagination build() {
            if (page < 1) page = 1;
            if (size < 1 || size > 100) size = 20;
            return new Pagination(page, size, sort);
        }
    }
}
```

## 3.2 @Builder 与 @Data 的组合使用

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UserDTO {
    private Long id;
    private String name;
    private String email;
    private Integer age;
}

// @Data 生成 getter/setter/toString/equals/hashCode/requiredArgs
// @Builder 生成 Builder
// @NoArgsConstructor 生成无参构造（@Data 不包含无参）
// @AllArgsConstructor 生成全参构造（Builder 需要）
```

## 3.3 @FieldNameConstants

```java
import lombok.experimental.FieldNameConstants;

@FieldNameConstants  // 生成包含字段名常量的内部类
public class User {
    private String name;
    private int age;
}

// 使用：
String fieldName = User.Fields.name;  // "name"
String fieldAge = User.Fields.age;    // "age"

// 适用于反射、MyBatis、QueryDSL 等场景
```

## 3.4 @Builder.Default 的高级用法

```java
@Builder
public class Order {

    private String id;

    @Builder.Default
    private List<OrderItem> items = new ArrayList<>();  // 可变集合

    @Builder.Default
    private OrderStatus status = OrderStatus.PENDING;

    // 复杂默认值
    @Builder.Default
    private BigDecimal total = BigDecimal.ZERO;

    // 使用 Supplier 生成默认值
    @Builder.Default
    private String timestamp = LocalDateTime.now().toString();
    // 注意：每次调用 build() 时重新计算
}
```

## 3.5 组合注解（自己定义）

```java
// 自定义组合注解
@Getter
@Setter
@ToString
@EqualsAndHashCode
@AllArgsConstructor
@NoArgsConstructor
public @interface DataWithAllArgs {
    // 通过组合多个 Lombok 注解创建自己的"宏注解"
}

// 使用自定义组合注解
@DataWithAllArgs
public class Product {
    private Long id;
    private String name;
    private BigDecimal price;
}
```

## 3.6 lombok.config 配置

在项目根目录下创建 `lombok.config` 文件：

```properties
# 配置文件格式

# 配置 @Data 不生成 setter
lombok.data.flagUsage=warning

# 配置 @AllArgsConstructor 的访问级别
lombok.allArgsConstructor.accessLevel=PROTECTED

# 检查 Experimental API 的使用
lombok.experimental.flagUsage=error

# 配置日志注解的字段名（默认 log）
lombok.log.fieldName=logger

# 配置 @ToString 是否包含字段名
lombok.toString.doNotUseGetters=false

# 配置 @EqualsAndHashCode 是否使用 getter
lombok.equalsAndHashCode.doNotUseGetters=true

# 全局禁止生成某个方法
lombok.anyConstructor.addConstructorProperties=false

# 配置 @Getter 的懒加载
lombok.getter.lazy.default=false

# 在生成的 getter/setter 上添加 @Override
lombok.getter.noIsPrefix=false

# 在 setter 上生成返回 this（类似于 @Accessors(chain=true)）
lombok.accessors.chain=true

# 配置日志注解的字段名
lombok.log.fieldName=log
```

### 配置文件查找顺序

```
1. lombok.config（当前目录）
2. lombok.config（父目录，递归向上）
3. ~/.lombok.config（用户目录）
```

---

# 4. 底层原理

## 4.1 注解处理机制（JSR 269）

Lombok 使用 Java 的"可插拔注解处理 API"（JSR 269）在编译时修改源代码。

```
编译流程：
1. javac 解析源码生成 AST
2. Lombok Annotation Processor 拦截处理
3. Lombok 修改 AST 添加方法节点
4. javac 根据修改后的 AST 生成字节码
```

### 关键源码类

| 类名 | 作用 |
|------|------|
| `lombok.core.LombokProcessor` | JSR 269 处理器入口 |
| `lombok.javac.JavacTransformer` | javac AST 转换 |
| `lombok.eclipse.EclipseTransformer` | Eclipse AST 转换 |
| `lombok.javac.handlers.HandleGetter` | 处理 @Getter |
| `lombok.javac.handlers.HandleSetter` | 处理 @Setter |
| `lombok.javac.handlers.HandleData` | 处理 @Data |
| `lombok.javac.handlers.HandleBuilder` | 处理 @Builder |

## 4.2 AST 修改流程（以 @Getter 为例）

```java
// Lombok 内部简化流程
public class HandleGetter {

    public void handle(Annotation annotation, JCTree.JCClassDecl classNode) {
        // 1. 遍历类中的所有字段
        for (JCTree.JCVariableDecl field : classNode.getFields()) {
            // 2. 检查是否有 @Getter 注解
            if (hasGetterAnnotation(field)) {
                // 3. 构造 getter 方法 AST 节点
                JCTree.JCMethodDecl getterMethod = createGetterMethod(field);

                // 4. 将方法节点添加到类中
                classNode.defs = classNode.defs.append(getterMethod);
            }
        }
    }

    private JCTree.JCMethodDecl createGetterMethod(JCTree.JCVariableDecl field) {
        // 构造类似的方法 AST：
        // public String getName() {
        //     return this.name;
        // }
        // 返回对应的方法 AST 节点
    }
}
```

## 4.3 Lombok 为什么不是运行时代理

- **编译时代码生成**：在 `.class` 文件中已包含生成的方法
- **零运行时依赖**：不需要反射或代理
- **IDE 友好**：IDE 通过 Lombok 插件在编辑器中显示生成的方法
- **性能无损失**：生成的方法和手写的一样高效

## 4.4 Lombok 的局限性

```java
// 1. 必须安装 IDE 插件
// 没有插件的 IDE 无法识别 Lombok 注解

// 2. 调试问题
// 调试时不会看到生成的方法源码

// 3. Javadoc 问题
// 生成的 getter/setter 没有文档注释

// 4. 继承问题
// @EqualsAndHashCode(callSuper=true) 容易引起问题

// 5. 与某些框架的兼容性
// - MyBatis（需要无参构造器）
// - JPA（需要无参构造器和 getter/setter）
```

---

# 5. 性能与注意事项

## 5.1 性能影响

```java
// 运行时性能：无影响
// Lombok 在编译时生成代码，运行期和手写代码完全一样

// 编译性能：略有影响
// - 增加了编译时间（注解处理开销）
// - 对于大型项目，建议使用 incremental compilation

// 内存影响：无
// - 生成的代码大小通常小于手写代码
```

## 5.2 与 IDE 配合

### IntelliJ IDEA

```java
// 需要安装 Lombok 插件：
// File -> Settings -> Plugins -> 搜索 "Lombok"
// 启用 Annotation Processing：
// File -> Settings -> Build -> Compiler -> Annotation Processors
// -> Enable annotation processing

// 常见 IDEA 设置：
// 1. 安装 Lombok 插件
// 2. 开启 Annotation Processing
// 3. 重启 IDE
```

### Eclipse / VS Code

```java
// Eclipse：运行 lombok.jar 安装
// java -jar lombok.jar

// VS Code：安装 "Lombok Annotations Support" 插件
```

## 5.3 编译构建工具集成

```java
// Maven（确保 annotation processor 正确配置）
// 见 1.2 Maven 依赖

// Gradle（需要 annotationProcessor 配置）
// 见 1.2 Gradle 配置

// 注意：如果 Lombok 生成的代码编译失败
// 1. 检查 annotation processor 是否正确配置
// 2. 检查 Lombok 和 Java 版本兼容性
// 3. 尝试 clean build
```

---

# 6. 最佳实践与常见坑

## 6.1 最佳实践

### 1. 使用 @Data + @Builder 的模式

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Entity {
    private Long id;
    private String name;
}
```

### 2. @Builder.Default 为集合设置默认值

```java
@Builder
public class Order {
    @Builder.Default
    private List<String> items = new ArrayList<>();
}
```

### 3. 使用 @Slf4j 统一日志

```java
@Slf4j
@RestController
public class UserController {
    // 直接使用 log 变量
}
```

### 4. 不可变对象使用 @Value + @With

```java
@Value
@With
public class ImmutableConfig {
    String host;
    int port;
    String username;
}
```

### 5. @Accessors(chain = true) 配合 Builder

```java
@Setter
@Accessors(chain = true)
public class QueryParams {
    private String keyword;
    private int page;
    private int size;
}

// 链式调用
QueryParams params = new QueryParams()
        .setKeyword("Java")
        .setPage(1)
        .setSize(10);
```

### 6. 工具类使用 @UtilityClass

```java
@UtilityClass
public class DateUtils {
    public String format(Date date) {
        return new SimpleDateFormat("yyyy-MM-dd").format(date);
    }
}
```

### 7. 合理使用 @EqualsAndHashCode

```java
// JPA 实体建议 @EqualsAndHashCode(onlyExplicitlyIncluded = true)
// 避免懒加载代理导致的问题
@Entity
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
public class User {
    @Id
    @EqualsAndHashCode.Include
    private Long id;

    private String name;
}
```

## 6.2 常见坑

### 坑 1：@EqualsAndHashCode 与继承

```java
// 父类和子类混合使用时可能违反 equals 的对称性
@Data
public class Animal {
    private String name;
}

@Data
@EqualsAndHashCode(callSuper = true)
public class Dog extends Animal {
    private String breed;
}

// Animal a = new Animal();
// a.setName("旺财");
// Dog d = new Dog();
// d.setName("旺财");
// d.setBreed("金毛");
// a.equals(d)  // true? 还是 false?
// d.equals(a)  // 抛异常？还是 false?

// 解决方案：组合优先于继承，或避免在这种场景使用 @EqualsAndHashCode
```

### 坑 2：@Data 和 @Builder 同时使用时无参构造器缺失

```java
@Data
@Builder
public class User {
    private String name;
    private int age;
}
// 如果使用了 Jackson 或 MyBatis 需要无参构造器时：
// 需要额外添加 @NoArgsConstructor 和 @AllArgsConstructor

// 正确做法：
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class User {
    private String name;
    private int age;
}
```

### 坑 3：@Builder 中的默认值丢失

```java
@Builder
public class Order {
    private String status = "PENDING";  // 注意：这个默认值不会被 Builder 使用！
    private List<String> items = new ArrayList<>();  // 这个也不会！
}

// 正确做法：使用 @Builder.Default
@Builder
public class Order {
    @Builder.Default
    private String status = "PENDING";

    @Builder.Default
    private List<String> items = new ArrayList<>();
}

// 错误使用：
Order order = Order.builder().build();
System.out.println(order.status);  // null（不是 "PENDING"！）
```

### 坑 4：@Value 与 JPA/Hibernate

```java
// @Value 生成 final 类和 final 字段
// JPA/Hibernate 需要无参构造器和非 final 的 setter
// 不要对 JPA 实体使用 @Value！

// 正确做法：手写或使用 @Data
@Entity
@Data
public class JpaEntity {
    @Id private Long id;
    private String name;
}
```

### 坑 5：@Slf4j 字段名冲突

```java
@Slf4j
public class LogService {
    // 如果类中已有 log 字段，会编译冲突
    // private Object log;  // 冲突！

    // 解决方案：在 lombok.config 中修改字段名
    // lombok.log.fieldName=logger
}
```

### 坑 6：Lombok 与 MapStruct / RecordBuilder 等冲突

```java
// 多个注解处理器可能对相同字段生成冲突代码
// 解决方案：
// 1. 确保注解处理器版本兼容
// 2. 使用最新的 Lombok 版本
// 3. 必要时手动实现某些方法

// Maven 中配置注解处理器顺序：
// <plugin>
//     <groupId>org.apache.maven.plugins</groupId>
//     <artifactId>maven-compiler-plugin</artifactId>
//     <configuration>
//         <annotationProcessorPaths>
//             <path>
//                 <groupId>org.projectlombok</groupId>
//                 <artifactId>lombok</artifactId>
//             </path>
//             <path>
//                 <groupId>org.mapstruct</groupId>
//                 <artifactId>mapstruct-processor</artifactId>
//             </path>
//         </annotationProcessorPaths>
//     </configuration>
// </plugin>
```

### 坑 7：@AllArgsConstructor 导致 Jackson 反序列化失败

```java
@AllArgsConstructor  // 生成全参构造器
@NoArgsConstructor   // 需要显式添加无参构造器
public class User {
    private String name;
}
// Jackson 默认需要无参构造器 + setter
// 如果没有 @NoArgsConstructor，Jackson 反序列化会失败
```

### 坑 8：链式 setter + 继承的碰撞

```java
@Setter
@Accessors(chain = true)
public class Base {
    private int id;
}

@Setter
@Accessors(chain = true)
public class Derived extends Base {
    private String name;
}

// 问题：Derived.setId() 返回的是 Base 类型，不是 Derived 类型
// 无法链式：new Derived().setId(1).setName("test");
// 因为 setId(1) 返回 Base，没有 setName 方法

// 解决方案：手写返回 Derived 类型的 setId
```

---

# 7. 与其他技术对比

## 7.1 Lombok vs Java Records（Java 14+）

| 对比项 | Lombok @Data | Java Records |
|--------|-------------|-------------|
| 引入版本 | 2010 (1.0) | Java 14 (preview) / Java 16 (stable) |
| 可变性 | 默认可变（@Data）或不可变（@Value） | 不可变 |
| 定义方式 | 注解驱动 | `record Point(int x, int y) {}` |
| 继承 | 可以 | 不能 |
| 额外方法 | 可以添加 | 只能添加实例方法 |
| Builder 模式 | @Builder 生成 | 需要手写 |
| JPA 实体 | 支持 | 不支持 |
| 第三方库 | 需要 IDE 插件 | 纯语言特性 |
| 学习成本 | 需要了解注解 | 零学习成本 |

```java
// Java Record
public record UserRecord(String name, int age, String email) {
    // 自动生成：构造器、getter、equals、hashCode、toString
    // 字段默认 private final

    // 可以添加 Compact Constructor 做验证
    public UserRecord {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("name 不能为空");
        }
    }

    // 可以添加实例方法
    public String displayName() {
        return name + " (" + age + ")";
    }
}

// 对比 Lombok
@Data
@AllArgsConstructor
public class UserLombok {
    private String name;
    private int age;
    private String email;
}
```

## 7.2 Lombok vs Kotlin 数据类

| 对比项 | Lombok | Kotlin Data Class |
|--------|--------|-------------------|
| 语言 | Java | Kotlin |
| 语法 | `data class User(val name: String, val age: Int)` |
| 集成度 | 第三方库 | 语言内置 |
| 互操作性 | Java | 与 Java 完全互通 |

```kotlin
// Kotlin Data Class
data class User(val name: String, val age: Int, val email: String)

// 自动生成：
// - getter/setter（val 只有 getter，var 都有）
// - equals()/hashCode()
// - toString()
// - copy()
// - component1()/component2()/component3()（解构声明）

// 使用：
// val user = User("张三", 25, "zhangsan@example.com")
// val (name, age, email) = user  // 解构
// val updated = user.copy(age = 26)  // 类似 @With
```

## 7.3 Lombok vs Immutables

| 对比项 | Lombok | Immutables |
|--------|--------|------------|
| 定位 | 通用样板代码消除 | 专门生成不可变对象 |
| 不可变对象 | @Value | @Value.Immutable |
| Builder | @Builder | 自动生成 |
| 性能 | 手写级（编译时生成） | 手写级（编译时生成） |
| 学习成本 | 低 | 中 |
| 流行度 | 极高 | 较低 |

## 7.4 选型建议

- **Java 14+ 新项目**：DTO 类使用 Record，实体类使用 Lombok
- **Java 8-11 项目**：Lombok 是最佳选择
- **JPA/Hibernate 项目**：Lombok 需要小心配置（见常见坑）
- **新项目（无历史包袱）**：可以考虑 Kotlin Data Class
- **高度不可变需求**：Immutables 有更好的不可变语义

---

# 8. 面试要点

## 8.1 基础问题

**Q1: Lombok 的工作原理是什么？**

A: Lombok 使用 JSR 269（可插拔注解处理 API）在编译时修改 AST（抽象语法树）。它在 javac 编译阶段拦截注解，在 AST 中添加对应的方法节点（如 getter/setter/toString），然后由 javac 继续编译生成包含这些方法的字节码。

**Q2: @Data 注解包含了哪些注解？**

A: @Data = @Getter + @Setter + @ToString + @EqualsAndHashCode + @RequiredArgsConstructor。

**Q3: @Value 和 @Data 的区别是什么？**

A: @Value 生成不可变类（所有字段被 `final` 修饰，没有 setter），而 @Data 生成可变类（有 setter）。@Value 还生成全参构造器（@AllArgsConstructor），而 @Data 生成必需参数构造器（@RequiredArgsConstructor）。

**Q4: @Builder 怎么设置默认值？**

A: 使用 `@Builder.Default` 注解字段。直接赋值的默认值不会被 Builder 使用，需要通过 `@Builder.Default` 显式指定。

**Q5: 为什么使用 @Data 的实体有时需要额外加 @NoArgsConstructor？**

A: @Data 只生成 @RequiredArgsConstructor（final 字段的构造器）。许多框架（Jackson、MyBatis、JPA）需要无参构造器来反射创建对象，因此需要额外添加 `@NoArgsConstructor`。同时，@Builder 通常也需要 @AllArgsConstructor 配合。

## 8.2 进阶问题

**Q6: Lombok 怎么处理 equals 和 hashCode 的继承问题？**

A: 默认情况子类不调用父类的 equals/hashCode。可以通过 `@EqualsAndHashCode(callSuper=true)` 让子类包含父类的字段。但要注意违反 equals 的对称性原则，建议组合优先于继承。

**Q7: @Accessors(chain=true) 有什么优缺点？**

A: 优点：支持链式调用，代码更简洁。缺点：与继承结合时返回类型问题，且某些框架可能不兼容（如 BeanUtils.copyProperties 可能因 setter 返回非 void 而出问题）。

**Q8: 在 Spring Boot 项目中如何配置 Lombok？**

A: 添加 Maven/Gradle 依赖（scope=provided），IDE 安装 Lombok 插件，开启 Annotation Processing。Spring Boot 对 Lombok 没有特殊要求，按标准方式使用即可。

**Q9: Lombok 和 MapStruct、MyBatis Generator 等注解处理器冲突怎么办？**

A: 在 Maven compiler 插件中通过 `<annotationProcessorPaths>` 明确指定处理器顺序。Lombok 应放在首位，因为其他处理器可能需要看到 Lombok 生成的方法。

**Q10: Java Record 能否完全替代 Lombok？**

A: 不能完全替代。Record 适合 DTO/VO 等不可变数据传输场景，但：不支持 JPA 实体（需要无参构造器和可变字段）、不支持继承、没有 Builder、没有日志注解等。Record 和 Lombok 是互补关系，不是替代关系。

## 8.3 实战题

**Q11: 你在项目中发现 @Builder 构建的对象字段值为 null（明明有默认值），是什么原因？**

A: 因为 Java 中直接赋值的字段默认值只在调用 new 构造器时生效。@Builder 使用全参构造器或 builder 模式设置值，不会读取字段的默认值。需要在字段上添加 `@Builder.Default` 注解。

**Q12: 为什么 Lombok 需要 IDE 插件，而其他注解（如 @Override）不需要？**

A: Lombok 的工作方式比较特殊——它在编译时修改 AST，IDE 需要在编辑阶段就知道生成的方法以供代码导航、自动补全和编译检查。而 @Override 只是编译检查注解，不影响 IDE 的代码模型。

**Q13: Lombok 的 @Slf4j 和直接声明 Logger 有什么区别？**

A: 本质无区别。`@Slf4j` 只是编译时代替你写 `private static final Logger log = LoggerFactory.getLogger(ClassName.class)`，生成的字节码完全一样。

**Q14: @Builder 和 @Accessors(chain=true) 有什么区别？**

A: @Builder 生成建造者模式（中间 Builder 对象 + build() 方法），可以参数化构建过程（制造不可变对象时很常用）。@Accessors(chain=true) 是直接修改 setter 返回 this 支持链式调用，不创建中间对象。前者适合复杂对象、必填参数校验；后者适合简单链式赋值。
