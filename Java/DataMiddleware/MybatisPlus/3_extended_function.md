# 1.代码生成

Mybatis-Plus 提供了强大的代码生成器，可以根据数据库表自动生成 Entity、Mapper、Service、Controller 等代码。

## 1.1 引入依赖

```xml
<!-- 代码生成器（Mybatis-Plus 3.5.3+ 推荐） -->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-generator</artifactId>
    <version>3.5.9</version>
</dependency>

<!-- 模板引擎（默认使用 Velocity，也可选 Freemarker） -->
<dependency>
    <groupId>org.apache.velocity</groupId>
    <artifactId>velocity-engine-core</artifactId>
    <version>2.3</version>
</dependency>
```

## 1.2 FastAutoGenerator 快速生成

Mybatis-Plus 3.5.3+ 推荐使用 `FastAutoGenerator`，链式调用，配置直观。

```java
public class CodeGenerator {
    public static void main(String[] args) {
        // 数据库连接信息
        String url = "jdbc:mysql://localhost:3306/mybatis_plus?useUnicode=true&characterEncoding=utf-8";
        String username = "root";
        String password = "root";

        FastAutoGenerator.create(url, username, password)
            // 1. 全局配置
            .globalConfig(builder -> {
                builder.author("开发者姓名")           // 作者名，会生成到 @author 中
                       .outputDir("D:\\project\\src\\main\\java")  // 输出目录
                       .commentDate("yyyy-MM-dd")     // 注释日期格式
                       .disableOpenDir();             // 生成后不打开输出目录
            })
            // 2. 包配置
            .packageConfig(builder -> {
                builder.parent("com.example")          // 父包名
                       .moduleName("system")           // 模块名
                       .entity("entity")               // 实体类包名
                       .mapper("mapper")               // Mapper 包名
                       .service("service")             // Service 包名
                       .serviceImpl("service.impl")    // ServiceImpl 包名
                       .controller("controller")       // Controller 包名
                       .xml("mapper.xml");             // XML 文件夹名
            })
            // 3. 策略配置
            .strategyConfig(builder -> {
                builder.addInclude("t_user", "t_order", "t_product")  // 要生成的表（可正则）
                       .addTablePrefix("t_")            // 过滤表前缀，生成类名时去掉 t_

                       // Entity 策略
                       .entityBuilder()
                       .enableLombok()                  // 使用 Lombok
                       .enableTableFieldAnnotation()    // 生成 @TableField 注解
                       .enableFileOverride()            // 覆盖已有文件
                       .logicDeleteColumnName("deleted")// 逻辑删除字段名
                       .versionColumnName("version")    // 乐观锁字段名

                       // Mapper 策略
                       .mapperBuilder()
                       .enableMapperAnnotation()        // 生成 @Mapper 注解
                       .enableBaseResultMap()           // 生成 BaseResultMap
                       .enableBaseColumnList()          // 生成 BaseColumnList

                       // Service 策略
                       .serviceBuilder()
                       .formatServiceFileName("%sService")      // Service 接口名格式
                       .formatServiceImplFileName("%sServiceImpl")  // ServiceImpl 名格式

                       // Controller 策略
                       .controllerBuilder()
                       .enableRestStyle()               // 生成 @RestController
                       .enableHyphenStyle()             // URL 使用连字符风格
                       .formatFileName("%sController"); // Controller 名格式
            })
            // 4. 模板引擎配置（可选）
            .templateEngine(new VelocityTemplateEngine())
            .execute();
    }
}
```

## 1.3 自定义模板

如果默认生成的代码不满足需求，可以自定义 Velocity 模板：

```java
// 指定自定义模板路径
FastAutoGenerator.create(url, username, password)
    .templateConfig(builder -> {
        builder.entity("templates/entity.java")          // 自定义 Entity 模板
               .mapper("templates/mapper.java")          // 自定义 Mapper 模板
               .service("templates/service.java")        // 自定义 Service 模板
               .serviceImpl("templates/serviceImpl.java");// 自定义 ServiceImpl 模板
    })
    // ...
    .execute();
```

自定义模板示例 `templates/entity.java.vm`（基于 Velocity 语法）：
```velocity
package ${package.Entity};

import com.baomidou.mybatisplus.annotation.*;
import lombok.Data;
import java.time.LocalDateTime;

@Data
@TableName("${table.name}")
public class ${entity} {

    @TableId(type = IdType.ASSIGN_ID)
    private Long id;

#foreach($field in ${table.fields})
#if(!${field.keyFlag})
    /**
     * ${field.comment}
     */
    @TableField("${field.name}")
    private ${field.propertyType} ${field.propertyName};

#end
#end
}
```

## 1.4 交互式生成（旧版 API）

如果需要通过控制台交互选择表，可使用旧版 `AutoGenerator` API：

```java
public class InteractiveCodeGenerator {
    public static void main(String[] args) {
        AutoGenerator generator = new AutoGenerator();

        // 数据源配置
        DataSourceConfig dataSourceConfig = new DataSourceConfig
            .Builder("jdbc:mysql://localhost:3306/mybatis_plus", "root", "root")
            .build();

        // 全局配置
        GlobalConfig globalConfig = new GlobalConfig.Builder()
            .author("开发者")
            .outputDir("D:\\project\\src\\main\\java")
            .build();

        // 包配置
        PackageConfig packageConfig = new PackageConfig.Builder()
            .parent("com.example")
            .build();

        // 策略配置
        StrategyConfig strategyConfig = new StrategyConfig.Builder()
            .addInclude("t_user")
            .build();

        generator.setDataSource(dataSourceConfig);
        generator.setGlobalConfig(globalConfig);
        generator.setPackageInfo(packageConfig);
        generator.setStrategy(strategyConfig);

        generator.execute();
    }
}
```

---

# 2.静态工具（Db 类）

Mybatis-Plus 3.5.1+ 提供了 `Db` 静态工具类，无需注入 Mapper 即可在任何位置操作数据库。

> **与 Wrappers 的区别：** `Db` 执行数据库操作并返回结果，`Wrappers` 仅构建条件不执行。

```java
public class DbUtilsExample {

    // ===== 查询操作 =====
    public void queryExamples() {
        // 查询所有
        List<User> allUsers = Db.list(User.class);

        // 按 ID 查询
        User user = Db.getById(1L, User.class);

        // Lambda 条件查询
        List<User> users = Db.lambdaQuery(User.class)
                .eq(User::getStatus, 1)
                .gt(User::getAge, 18)
                .orderByDesc(User::getCreateTime)
                .list();

        // 查询单个
        User one = Db.lambdaQuery(User.class)
                .eq(User::getUsername, "张三")
                .one();

        // 只查指定字段
        List<User> users2 = Db.lambdaQuery(User.class)
                .select(User::getId, User::getUsername)
                .ge(User::getAge, 18)
                .list();

        // 查询数量
        long count = Db.lambdaQuery(User.class)
                .eq(User::getStatus, 1)
                .count();

        // 查询 Map 列表
        List<Map<String, Object>> maps = Db.listMaps(
                Wrappers.<User>lambdaQuery().eq(User::getStatus, 1)
        );

        // 分页查询
        Page<User> page = new Page<>(1, 10);
        Page<User> result = Db.page(page, Wrappers.<User>lambdaQuery().eq(User::getStatus, 1));
    }

    // ===== 新增操作 =====
    public void insertExamples() {
        // 单条插入
        User user = new User();
        user.setUsername("王五");
        Db.save(user);

        // 批量插入
        List<User> userList = new ArrayList<>();
        for (int i = 0; i < 100; i++) {
            User u = new User();
            u.setUsername("用户" + i);
            userList.add(u);
        }
        Db.saveBatch(userList, 50);   // 每批 50 条
    }

    // ===== 更新操作 =====
    public void updateExamples() {
        // 按 ID 更新
        User user = Db.getById(1L, User.class);
        user.setUsername("新名字");
        Db.updateById(user);

        // 条件更新
        boolean updated = Db.lambdaUpdate(User.class)
                .set(User::getStatus, 0)
                .eq(User::getId, 1L)
                .update();
    }

    // ===== 删除操作 =====
    public void deleteExamples() {
        // 按 ID 删除
        Db.removeById(1L, User.class);

        // 按条件删除
        boolean deleted = Db.remove(
                Wrappers.<User>lambdaQuery().eq(User::getStatus, -1)
        );
    }
}
```

---

# 3.逻辑删除

逻辑删除指的是在数据库中不真正删除记录，而是通过一个字段标记记录是否被删除。查询时自动过滤已删除记录。

## 3.1 配置方式

**方式一：全局配置（推荐）**

```yaml
mybatis-plus:
  global-config:
    db-config:
      logic-delete-field: deleted      # 逻辑删除字段名（实体类中的字段名）
      logic-delete-value: 1            # 已删除的值（默认 1）
      logic-not-delete-value: 0        # 未删除的值（默认 0）
```

**方式二：注解配置**

如果只想对特定实体开启逻辑删除：

```java
@Data
@TableName("t_user")
public class User {
    private Long id;
    private String username;

    @TableLogic
    private Integer deleted;  // 0 未删除，1 已删除
}
```

> 如果全局配置了 `logic-delete-field`，则无需在每个字段上加 `@TableLogic`。

## 3.2 效果演示

配置逻辑删除后，MP 会自动改写 SQL：

```java
// 执行删除操作
userMapper.deleteById(1L);
// 实际执行的 SQL：UPDATE t_user SET deleted = 1 WHERE id = 1 AND deleted = 0

// 条件删除
LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
wrapper.eq(User::getStatus, -1);
userMapper.delete(wrapper);
// 实际执行的 SQL：UPDATE t_user SET deleted = 1 WHERE status = -1 AND deleted = 0

// 执行查询操作
List<User> users = userMapper.selectList(null);
// 实际执行的 SQL：SELECT id, username, age FROM t_user WHERE deleted = 0

// 执行更新操作
userMapper.updateById(user);
// 实际执行的 SQL：UPDATE t_user SET ... WHERE id = 1 AND deleted = 0
```

## 3.3 注意事项

- 逻辑删除后，**所有 CRUD 操作都会自动忽略"已删除"的记录**
- 逻辑删除字段**建议在数据库中设置默认值为 0**（未删除）
- 若需要查询已删除的记录，需**自定义 SQL**，因为 MP 自动拼接的条件在自定义 SQL 中不生效
- MySQL 中逻辑删除字段不能加唯一索引（删过的数据无法再次添加相同值）
- 逻辑删除字段需要设置 `@TableField(select = false)` 已在全局配置中自动处理

---

# 4.枚举处理器

Mybatis-Plus 提供了枚举类型与数据库值之间的自动转换功能，避免手动进行枚举与 Integer/String 的转换。

## 4.1 引入枚举处理器

Mybatis-Plus 3.5.8+ 枚举处理器移至独立模块，需单独引入：

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-jsqlparser-3.5.9</artifactId>
    <version>3.5.9</version>
</dependency>
```

## 4.2 定义枚举类

使用 `@EnumValue` 标记存储到数据库的值：

```java
// 性别枚举
public enum GenderEnum {
    MALE(1, "男"),
    FEMALE(2, "女"),
    UNKNOWN(0, "未知");

    @EnumValue          // 标记存储到数据库的值
    private final int value;

    @JsonValue          // Jackson 序列化时输出的值（如果是 Web 项目）
    private final String desc;

    GenderEnum(int value, String desc) {
        this.value = value;
        this.desc = desc;
    }

    // 可选：根据数据库值反查枚举
    public static GenderEnum of(int value) {
        for (GenderEnum e : GenderEnum.values()) {
            if (e.value == value) {
                return e;
            }
        }
        return UNKNOWN;
    }
}
```

```java
// 状态枚举（使用 String 值示例）
public enum StatusEnum {
    ACTIVE("A", "启用"),
    INACTIVE("I", "停用"),
    DELETED("D", "已删除");

    @EnumValue
    private final String code;
    private final String desc;

    StatusEnum(String code, String desc) {
        this.code = code;
        this.desc = desc;
    }
}
```

## 4.3 配置枚举包路径

```yaml
mybatis-plus:
  type-enums-package: com.example.enums    # 多个包用逗号分隔
```

## 4.4 实体类使用枚举

```java
@Data
@TableName("t_user")
public class User {
    private Long id;
    private String username;
    private GenderEnum gender;     // 直接使用枚举类型
    private StatusEnum status;     // 直接使用枚举类型
}
```

使用效果：
```java
// 插入
User user = new User();
user.setUsername("张三");
user.setGender(GenderEnum.MALE);    // MP 自动将枚举转为 1 存入数据库
user.setStatus(StatusEnum.ACTIVE);  // MP 自动将枚举转为 "A" 存入数据库
userMapper.insert(user);

// 查询
User dbUser = userMapper.selectById(1L);
System.out.println(dbUser.getGender());   // GenderEnum.MALE  （自动转为枚举）
System.out.println(dbUser.getStatus());   // StatusEnum.ACTIVE（自动转为枚举）
```

## 4.5 自定义枚举类型处理器（高级）

如果默认枚举处理器不满足需求，可以自定义：

```java
// 实现 IEnum 接口的方式（传统方式，不推荐，推荐 @EnumValue）
public enum AgeEnum implements IEnum<Integer> {
    ONE(1, "一岁"),
    TWO(2, "两岁");

    private final int value;
    private final String desc;

    AgeEnum(int value, String desc) {
        this.value = value;
        this.desc = desc;
    }

    @Override
    public Integer getValue() {
        return this.value;
    }
}
```

---

# 5.JSON 处理器

当数据库字段类型为 JSON（MySQL 5.7+ 支持）或 text/varchar 存储 JSON 字符串时，MP 提供了 JSON 类型处理器，自动进行 Java 对象与 JSON 字符串之间的转换。

## 5.1 配置 JSON 类型处理器

创建一个 Jackson 类型处理器（Mybatis-Plus 内置了 `JacksonTypeHandler`）：

```java
// 无需自己写，Mybatis-Plus 提供了内置的 JacksonTypeHandler
// 如果需要自定义序列化规则，继承它即可
```

## 5.2 在实体类中使用

**示例一：实体字段是对象类型**

```java
@Data
@TableName(value = "t_user", autoResultMap = true)  // 必须开启 autoResultMap
public class User {
    private Long id;
    private String username;

    // 将 Java 对象序列化为 JSON 存入数据库，读取时反序列化回 Java 对象
    @TableField(typeHandler = JacksonTypeHandler.class)
    private Address address;     // 自定义 Address 类

    @TableField(typeHandler = JacksonTypeHandler.class)
    private List<String> hobbies;  // 数组/集合类型
}

@Data
public class Address {
    private String province;
    private String city;
    private String street;
}
```

**示例二：实体字段是 Map 类型**

```java
@Data
@TableName(value = "t_product", autoResultMap = true)
public class Product {
    private Long id;
    private String name;

    @TableField(typeHandler = JacksonTypeHandler.class)
    private Map<String, Object> attributes;  // 动态属性
}
```

## 5.3 使用示例

```java
// 插入
User user = new User();
user.setUsername("张三");
Address addr = new Address();
addr.setProvince("广东省");
addr.setCity("深圳市");
user.setAddress(addr);

List<String> hobbies = Arrays.asList("读书", "游泳", "编程");
user.setHobbies(hobbies);
userMapper.insert(user);
// 数据库 address 字段存储：{"province":"广东省","city":"深圳市"}
// 数据库 hobbies 字段存储：["读书","游泳","编程"]

// 查询
User dbUser = userMapper.selectById(user.getId());
Address address = dbUser.getAddress();     // 自动反序列化为 Address 对象
List<String> hs = dbUser.getHobbies();     // 自动反序列化为 List<String>
System.out.println(address.getCity());     // 深圳市
```

## 5.4 使用 Fastjson 作为 JSON 处理器

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>2.0.53</version>
</dependency>
```

```java
// 实体类中使用 Fastjson 的类型处理器
@TableField(typeHandler = FastjsonTypeHandler.class)
private Address address;
```

> **注意：**
> - 必须开启 `autoResultMap = true`，否则 typeHandler 不生效
> - JSON 处理器需要引入对应的 JSON 库（Jackson 默认由 Spring Boot 引入）
> - MySQL 数据库字段类型建议使用 `json` 类型（MySQL 5.7+），否则可用 `text` 或 `varchar`
