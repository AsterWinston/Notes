# 1. 序列化与序列（Sequence）

## 1.1 内置主键策略对比

| 主键策略 | 枚举值 | 数据类型 | 说明 |
|---------|--------|---------|------|
| 雪花算法 | `ASSIGN_ID` | Long | 默认策略，分布式唯一，趋势递增 |
| UUID | `ASSIGN_UUID` | String | 32位无中划线 UUID |
| 自增 | `AUTO` | Long/Integer | 依赖数据库自增 |
| 手动 | `INPUT` | 任意 | 程序手动设置 |

## 1.2 自定义 ID 生成器

```java
@Component
public class CustomIdGenerator implements IdentifierGenerator {

    @Override
    public Number nextId(Object entity) {
        // 根据实体类名分配不同业务前缀
        String bizKey = entity.getClass().getSimpleName();
        // 可使用 Redis 自增、号段模式、或者自定义雪花算法
        return IdWorker.getId(entity);
    }

    @Override
    public String nextUUID(Object entity) {
        return IdWorker.get32UUID();
    }
}
```

---

# 2. ActiveRecord 模式

Mybatis-Plus 支持 ActiveRecord 模式，实体类直接继承 `Model<T>` 即可拥有 CRUD 方法。

```java
@Data
@EqualsAndHashCode(callSuper = false)
@TableName("t_user")
public class User extends Model<User> {
    private Long id;
    private String username;
    private Integer age;
    private String email;
}
```

```java
// 使用 ActiveRecord 模式
User user = new User();
user.setUsername("张三");
user.setAge(25);
user.insert();                     // 直接保存

User dbUser = new User();
dbUser.setId(1L);
User result = dbUser.selectById(); // 直接查询

User updateUser = new User();
updateUser.setId(1L);
updateUser.setUsername("李四");
updateUser.updateById();           // 直接更新

User delUser = new User();
delUser.setId(1L);
delUser.deleteById();              // 直接删除

// 批量操作
user.insertOrUpdate();             // 有ID则更新，无则插入

// 链式调用（MP 3.4.0+）
List<User> users = new User()
        .selectList(new LambdaQueryWrapper<User>()
                .gt(User::getAge, 18));
```

> **不推荐 ActiveRecord 模式的原因：** 实体与数据库操作耦合，违反单一职责原则。项目中仍建议使用 Mapper + Service 的分层架构。

---

# 3. 自定义通用方法（SQL 注入器）

当 Mybatis-Plus 内置的通用方法不满足需求时，可以通过 SQL 注入器自定义通用方法。

## 3.1 定义 SQL 方法

```java
// 定义自定义方法（如：全量删除）
public class DeleteAll extends AbstractMethod {

    @Override
    public MappedStatement injectMappedStatement(Class<?> mapperClass,
                                                  Class<?> modelClass,
                                                  TableInfo tableInfo) {
        String sql = "DELETE FROM " + tableInfo.getTableName();
        SqlSource sqlSource = languageDriver.createSqlSource(configuration, sql, modelClass);
        return this.addDeleteMappedStatement(mapperClass, "deleteAll", sqlSource);
    }
}
```

## 3.2 注册自定义方法

```java
public class MySqlInjector extends DefaultSqlInjector {

    @Override
    public List<AbstractMethod> getMethodList(Configuration configuration,
                                               Class<?> mapperClass,
                                               TableInfo tableInfo) {
        List<AbstractMethod> methodList = super.getMethodList(configuration, mapperClass, tableInfo);
        methodList.add(new DeleteAll());  // 注册自定义方法
        return methodList;
    }
}
```

## 3.3 配置自定义注入器

```java
@Configuration
public class MybatisPlusConfig {

    @Bean
    public MySqlInjector mySqlInjector() {
        return new MySqlInjector();
    }
}
```

## 3.4 自定义 BaseMapper

```java
// 扩展的通用 Mapper 接口
public interface MyBaseMapper<T> extends BaseMapper<T> {

    // 自定义通用方法：清空表数据
    int deleteAll();
}

// 实际项目的 Mapper 继承自定义的 MyBaseMapper
@Mapper
public interface UserMapper extends MyBaseMapper<User> {
}
```

---

# 4. 动态表名插件

当需要根据时间、租户等动态切换表名时使用（如按月分表：`t_order_202401`、`t_order_202402`）。

```java
@Configuration
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

        // 动态表名插件
        DynamicTableNameInnerInterceptor dynamicTableNameInterceptor =
                new DynamicTableNameInnerInterceptor();

        // 设置表名处理器
        dynamicTableNameInterceptor.setTableNameHandler((sql, tableName) -> {
            // 按月分表：t_order → t_order_202401
            if ("t_order".equals(tableName)) {
                // 从 ThreadLocal 中获取当前月份
                String month = TableNameContextHolder.getMonth();
                return tableName + "_" + month;
            }
            return tableName;  // 其他表名不变
        });

        interceptor.addInnerInterceptor(dynamicTableNameInterceptor);
        return interceptor;
    }
}

// ThreadLocal 工具
public class TableNameContextHolder {
    private static final ThreadLocal<String> MONTH_HOLDER = new ThreadLocal<>();

    public static void setMonth(String month) { MONTH_HOLDER.set(month); }
    public static String getMonth() { return MONTH_HOLDER.get(); }
    public static void clear() { MONTH_HOLDER.remove(); }
}
```

---

# 5. 数据安全保护（字段加密）

## 5.1 配置密钥

```yaml
mybatis-plus:
  type-handlers-package: com.example.handler
```

## 5.2 自定义加解密处理器

```java
// AES 加解密 TypeHandler
public class AESEncryptHandler extends BaseTypeHandler<String> {

    private static final String AES_KEY = "1234567890123456"; // 16位密钥（不可硬编码）

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i,
                                     String parameter, JdbcType jdbcType) throws Exception {
        // 写入数据库前加密
        ps.setString(i, AESUtils.encrypt(parameter, AES_KEY));
    }

    @Override
    public String getNullableResult(ResultSet rs, String columnName) throws Exception {
        // 读取数据库后解密
        String value = rs.getString(columnName);
        return value == null ? null : AESUtils.decrypt(value, AES_KEY);
    }

    @Override
    public String getNullableResult(ResultSet rs, int columnIndex) throws Exception {
        String value = rs.getString(columnIndex);
        return value == null ? null : AESUtils.decrypt(value, AES_KEY);
    }

    @Override
    public String getNullableResult(CallableStatement cs, int columnIndex) throws Exception {
        String value = cs.getString(columnIndex);
        return value == null ? null : AESUtils.decrypt(value, AES_KEY);
    }
}
```

## 5.3 实体类使用

```java
@Data
@TableName(value = "t_user", autoResultMap = true)
public class User {
    private Long id;

    @TableField(typeHandler = AESEncryptHandler.class)
    private String phone;      // 手机号加密存储

    @TableField(typeHandler = AESEncryptHandler.class)
    private String idCard;     // 身份证号加密存储
}
```

---

# 6. Mybatis-Plus 与 Mybatis 对比

| 方面 | Mybatis | Mybatis-Plus |
|------|---------|-------------|
| 单表 CRUD | 需手写 SQL | 继承 BaseMapper 零 SQL |
| 条件构造 | 手写 `<if>` 标签 | QueryWrapper 动态拼接 |
| 分页 | 手写 LIMIT + COUNT | 自动分页插件 |
| 主键生成 | 手动设置或 useGeneratedKeys | 雪花算法自动生成 |
| 逻辑删除 | 手写 update 标记 | 注解/全局配置自动处理 |
| 乐观锁 | 手写 where version=? | @Version 注解自动处理 |
| 自动填充 | 手写 set 逻辑 | MetaObjectHandler 自动填充 |
| 代码生成 | 无 | FastAutoGenerator 一键生成 |
| 多租户 | 每条 SQL 手动加 tenant_id | 插件自动注入 |
| SQL 日志 | 需要第三方或 log4j | 内置 stdout 日志 |
| **结论** | 更灵活，完全掌控 SQL | 更高效，大幅减少模板代码 |

> **最佳实践：** 单表操作交给 Mybatis-Plus，复杂关联/统计查询手写 SQL。两者可无缝混用。

---

# 7. 常见问题与最佳实践

## 7.1 字段更新策略

```yaml
mybatis-plus:
  global-config:
    db-config:
      # NOT_NULL：字段非 NULL 时才更新（默认）
      # NOT_EMPTY：字段非 NULL 且非空字符串时更新
      # IGNORED：忽略判断，总是更新
      # NEVER：永远不更新
      update-strategy: not_null
      insert-strategy: not_null
```

## 7.2 Lambda 表达式不能序列化问题

Mybatis-Plus 的 `LambdaQueryWrapper` 使用了 JDK 序列化获取字段名。某些 JDK 版本会报 `SerializedLambda` 相关错误。

```java
// 解决方式：引入 Lombok 并确保实体类上有 @Data
@Data
public class User implements Serializable {   // 实现 Serializable
    private static final long serialVersionUID = 1L;
    // ...
}
```

## 7.3 大字段（BLOB/TEXT）查询优化

```java
// 选择性查询大字段
@TableField(select = false)   // 默认查询时不返回
private String content;       // 文章内容等大文本

// 需要大字段时单独查
@Select("SELECT content FROM t_article WHERE id = #{id}")
String selectContentById(Long id);
```

## 7.4 批量操作性能优化

```yaml
# MySQL 批量插入优化
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mybatis_plus?rewriteBatchedStatements=true&useServerPrepStmts=false
    # rewriteBatchedStatements=true：将多条 INSERT 合并成一条批量 INSERT
    # useServerPrepStmts=false：批量执行时关闭服务端预处理

mybatis-plus:
  global-config:
    db-config:
      # 批量插入时每批数量（默认 1000）
      batch-size: 500
```

## 7.5 Wrapper 查询指定字段不推荐使用

```java
// 不推荐：select 指定字段，返回的实体中只有这些字段有值，其余为 null（容易误解）
wrapper.select("id", "username");

// 推荐：使用专门的 VO/DTO
@Data
public class UserVO {
    private Long id;
    private String username;
}
// Mapper 中写自定义 SQL 返回 VO
```

## 7.6 非法 SQL 注入检查器

```java
@Configuration
public class MybatisPlusConfig {
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

        // 防止恶意的全表更新与删除（开发/测试环境建议启用）
        interceptor.addInnerInterceptor(new BlockAttackInnerInterceptor());

        // 检查 WHERE 条件是否含有非法注入字符
        interceptor.addInnerInterceptor(new IllegalSQLInnerInterceptor());

        return interceptor;
    }
}
```

## 7.7 SQL 性能规范插件（开发环境）

```java
// 检查是否缺少 WHERE 条件、是否存在全表扫描等
interceptor.addInnerInterceptor(new IllegalSQLInnerInterceptor());
```

## 7.8 常见错误排查

| 错误 | 原因 | 解决 |
|------|------|------|
| `Invalid bound statement (not found)` | XML 路径与 `mapper-locations` 不匹配 | 检查配置和 XML 路径 |
| 逻辑删除查询仍能查到 | 未配置 `@TableLogic` 或全局配置 | 确认注解/配置生效 |
| `@TableField(exist=false)` 字段报错 | 可能写在 XML 中使用 | `exist=false` 字段不出现在 XML |
| 分页不生效 | 未配置 `MybatisPlusInterceptor` | 配置分页插件 |
| 乐观锁更新 0 行 | Version 不匹配或被覆盖 | 检查并发逻辑 |
| 雪花算法 ID 重复 | 机器标识配置错误 | 配置 worker-id 和 data-center-id |
| `selectPage` 报错 | `PaginationInnerInterceptor` 未注册 | 确认插件配置 |

---

# 8. Mybatis-Plus 配置大全

```yaml
mybatis-plus:
  # Mapper XML 文件路径
  mapper-locations: classpath*:/mapper/**/*.xml
  # 实体类包路径
  type-aliases-package: com.example.entity
  # 枚举类包路径
  type-enums-package: com.example.enums
  # 类型处理器扫描
  type-handlers-package: com.example.handler
  # 配置扫描（不推荐使用，有注入风险）
  check-config-location: false
  # 执行器类型：SIMPLE(默认), REUSE, BATCH
  executor-type: simple

  configuration:
    # 下划线转驼峰（默认开启）
    map-underscore-to-camel-case: true
    # 二级缓存（默认关闭）
    cache-enabled: false
    # 日志实现
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
    # 当结果集为空时，返回 null 还是空集合
    return-instance-for-empty-row: false
    # 调用 setter 时，空值也设置（默认 false）
    call-setters-on-nulls: false
    # SQL 方言
    database-id: mysql
    # 默认语句超时时间（秒）
    default-statement-timeout: 30
    # 是否允许在嵌套语句中使用分页（RowBounds）
    safe-rowbounds-enabled: false

  global-config:
    # 是否控制台输出 Mybatis-Plus Banner
    banner: true
    db-config:
      # 主键类型
      id-type: assign_id
      # 表名前缀
      table-prefix: t_
      # 表名下划线风格（默认 true）
      table-underline: true
      # 逻辑删除字段名
      logic-delete-field: deleted
      # 逻辑删除值
      logic-delete-value: 1
      # 逻辑未删除值
      logic-not-delete-value: 0
      # 字段更新策略
      update-strategy: not_null
      # 字段插入策略
      insert-strategy: not_null
      # WHERE 策略
      where-strategy: not_null
```
