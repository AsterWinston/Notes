# 1. 知识点
- **MyBatis-Plus定位**：MyBatis的增强工具，在MyBatis基础上只做增强不做改变，为简化开发而生。核心功能：通用CRUD（BaseMapper）、条件构造器（Wrapper）、分页插件、代码生成器等
- **无侵入式增强**：MyBatis-Plus不修改MyBatis原有功能，引入后原MyBatis代码可正常运行，只是在基础之上增加了更多自动化操作。底层依然使用MyBatis的SqlSessionFactory和MapperProxy
- **BaseMapper<T>**：通用Mapper接口，泛型T为实体类。继承后自动获得以下方法：insert/deleteById/deleteByMap/delete/deleteBatchIds/updateById/update/selectById/selectBatchIds/selectByMap/selectOne/selectCount/selectList/selectPage等。**不需要写任何SQL**即可完成大部分单表操作
- **IService<T>和ServiceImpl<M,T>**：Service层的通用封装，IService定义接口（继承BaseMapper的所有方法+批量操作+链式调用），ServiceImpl提供默认实现。提供了saveBatch/updateBatchById等批量方法（底层自动拼接为多条SQL批量提交，不是foreach拼接）
- **条件构造器Wrapper**：用于构建复杂的WHERE条件，避免手写SQL拼接。抽象父类Wrapper下分：QueryWrapper（查询条件）、UpdateWrapper（更新条件）、LambdaQueryWrapper（Lambda方式防字段名拼错）、LambdaUpdateWrapper
- **QueryWrapper**：通过字符串字段名构建条件，如`new QueryWrapper<User>().eq("name", "张三").gt("age", 18)`，容易因字段名拼写错误在运行时报错
- **LambdaQueryWrapper**：通过Lambda表达式获取字段名，编译期检查，**推荐使用**。如`new LambdaQueryWrapper<User>().eq(User::getName, "张三").gt(User::getAge, 18)`，IDE可自动补全和重构
- **常用条件方法**：eq（等于）、ne（不等于）、gt（大于）、ge（大于等于）、lt（小于）、le（小于等于）、between（区间）、notBetween、like（模糊）、notLike、likeLeft/likeRight（左/右模糊）、isNull、isNotNull、in、notIn、orderByAsc/Desc、groupBy、having、or（或）、and（且，默认）、apply（直接拼接SQL片段）
- **分页插件PaginationInnerInterceptor**：实现物理分页的核心，继承InnerInterceptor。执行query前自动识别分页参数并拼接Limit SQL。配置方式：`MybatisPlusInterceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL))`
- **分页使用**：`IPage<User> page = new Page<>(pageNum, pageSize);`，调用`userMapper.selectPage(page, wrapper)`，返回的Page对象包含records（数据列表）、total（总数）、current（当前页）、pages（总页数）
- **MybatisPlusInterceptor**：MyBatis-Plus 3.4.0+的核心插件容器，替代旧版本的各种独立分页插件。通过addInnerInterceptor()可以添加多个内置拦截器：分页、多租户、乐观锁、防全表更新删除等
- **主键策略IdType**：控制实体插入时主键的生成方式。AUTO（数据库自增，数据库字段必须设置为AUTO_INCREMENT）、NONE（无策略，跟随全局设置）、INPUT（用户手动设置id值）、ASSIGN_ID（默认，雪花算法生成Long型19位ID）、ASSIGN_UUID（生成32位UUID去除横线）
- **雪花算法(Snowflake)**：Twitter开源的分布式ID生成算法，MyBatis-Plus使用改良版。结构：1bit符号位（0）+ 41bit时间戳（毫秒级，69年）+ 10bit工作机器ID（5bit数据中心+5bit机器ID，支持32*32=1024节点）+ 12bit序列号（毫秒内计数，4096个/ms）。趋势递增、不依赖数据库、全局唯一
- **主键策略配置**：全局配置`mybatis-plus.global-config.db-config.id-type=auto`，或通过@TableId注解单字段配置（优先级高于全局）：`@TableId(value = "id", type = IdType.ASSIGN_ID)`
- **自动填充**：实体字段的创建时间和更新时间自动填充，无需手动set。实现MetaObjectHandler接口，重写insertFill()和updateFill()方法。在实体字段上标注`@TableField(fill = FieldFill.INSERT)`或`@TableField(fill = FieldFill.INSERT_UPDATE)`或`@TableField(fill = FieldFill.UPDATE)`
- **自动填充获取当前用户**：通过RequestContextHolder.getRequestAttributes()获取HttpSession/Header中的用户信息，或使用ThreadLocal在登录拦截器中设置，实现fill方法中获取操作人
- **逻辑删除**：不物理删除数据，而是将删除标记字段更新为已删除状态。通过`@TableLogic`注解标记实体字段，配置：未删除值(0)/已删除值(1)。MyBatis-Plus自动将DELETE语句转换为UPDATE语句，且所有SELECT查询自动附加`WHERE deleted=0`条件
- **逻辑删除配置**：全局配置`mybatis-plus.global-config.db-config.logic-delete-field=deleted`和`logic-delete-value=1`和`logic-not-delete-value=0`；局部通过@TableLogic注解。注意：逻辑删除后的数据唯一索引可能会冲突（因为物理上数据仍存在），需用联合唯一索引或业务处理
- **乐观锁**：解决并发更新数据丢失的问题。实体字段标注`@Version`，数据库表添加version字段（int或long，初始值0或1）。更新时MP自动在WHERE中附加`version = #{version}`并将version+1，若影响行数为0说明数据已被其他事务修改，抛出异常
- **乐观锁插件配置**：`mybatisPlusInterceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor())`。更新时必须带上version字段的原始值（即先查出来再更新），否则乐观锁不生效
- **乐观锁 vs 悲观锁**：乐观锁适用于读多写少、冲突概率低的场景（如用户资料编辑）；悲观锁适用于写多、冲突概率高的场景（如库存扣减），通过SELECT...FOR UPDATE实现
- **多租户**：通过tenantLineHandler实现多租户数据隔离。实现TenantLineHandler接口（getTenantId获取当前租户ID、getTenantIdColumn指定租户字段列名、ignoreTable忽略不处理的表），注册到MybatisPlusInterceptor中。所有查询自动附加租户条件
- **多租户实现方式**：独立数据库（每个租户一个库）、共享数据库独立Schema（每个租户一个Schema）、共享数据库共享表（表中加tenant_id字段区分，MP默认方式）、混合模式
- **动态表名**：通过DynamicTableNameInnerInterceptor实现，适用场景如按月分表。实现TableNameHandler接口（根据SQL和参数动态返回真实表名），注册到MybatisPlusInterceptor中
- **防全表删除/更新**：BlockAttackInnerInterceptor（攻击阻断内部拦截器），拦截没有WHERE条件的DELETE和UPDATE语句，抛出异常。针对分表的场景也拦截不带分区键的全表扫描。通过`mybatisPlusInterceptor.addInnerInterceptor(new BlockAttackInnerInterceptor())`配置
- **动态数据源**：通过dynamic-datasource-spring-boot-starter配合`@DS`注解实现读写分离和多数据源切换。`@DS("master")`/`@DS("slave")`注解在类或方法上指定使用的数据源，支持通过Header/Session动态切换
- **读写分离实现**：配置master和slave数据源，Service方法不需要手动切换（框架注解驱动）。读操作标注@DS("slave")走从库，写操作标注@DS("master")走主库。支持一主多从、多主多从
- **代码生成器（AutoGenerator）**：MyBatis-Plus 3.5.1+新版本代码生成器。核心配置：dataSourceConfig（数据源）、packageConfig（包名配置）、strategyConfig（生成策略，包含哪些表/排除哪些表/前缀去除/命名策略）、templateEngine（模板引擎，默认Velocity，可选Freemarker/Beetl）、全局配置（作者/输出目录/日期格式等）
- **代码生成器生成内容**：实体类（Entity，含@TableName/@TableId/@TableField注解）、Mapper接口（继承BaseMapper）、Mapper XML（基本CRUD）、Service接口（继承IService）、ServiceImpl实现类（继承ServiceImpl）、Controller（可选，含RESTful接口）
- **与MyBatis的区别**：MyBatis需手写SQL（即使单表CRUD）和XML映射文件，MyBatis-Plus通过BaseMapper自动提供单表CRUD（零SQL）；MyBatis需手动处理分页（或集成PageHelper），MyBatis-Plus内置分页插件；MyBatis没有条件构造器（需手写动态SQL或Example），MyBatis-Plus有强大的Wrapper/LambdaWrapper链式条件封装
- **选型建议**：新项目推荐MyBatis-Plus（减少重复代码，提高开发效率）；已有MyBatis项目渐进式引入MP（不冲突，可共存）；复杂多表关联/HQL/复杂动态报表场景可考虑JPA/Hibernate；对SQL有极致控制需求的保留原生MyBatis
- **链式调用LambdaQueryChainWrapper**：Service层的链式查询，如`userService.lambdaQuery().eq(User::getName,"张三").gt(User::getAge,18).list()`，代码更简洁，融合了Service和Wrapper
- **批量操作saveBatch原理**：ServiceImpl的saveBatch()底层循环调用mapper.insert()（非SQL拼接foreach），默认每1000条执行一次flushStatements（JDBC批量提交），通过事务保证原子性。区别于foreach拼接（一条SQL的批量插入），saveBatch是多条SQL的批量提交
- **枚举映射TypeHandler**：MP内置枚举类型处理，在application.yml配置mybatis-plus.type-enums-package指定枚举包路径，实体中可直接使用枚举字段，数据库存int/String自动映射
- **字段类型处理器@TableField(jdbcType/typeHandler)**：解决Java类型与JDBC类型的映射问题，如JSON字段存入数据库（JacksonTypeHandler使对象自动JSON序列化反序列化）
- **全局配置**：mybatis-plus.mapper-locations（mapper.xml位置）、mybatis-plus.type-aliases-package（别名包）、mybatis-plus.configuration.map-underscore-to-camel-case（驼峰命名，默认true）、mybatis-plus.global-config.db-config.table-prefix（表名前缀）、mybatis-plus.global-config.db-config.id-type（全局主键策略）

# 2. 面试题

## 2.1 MyBatis-Plus和MyBatis有什么区别？如何选型？
| 维度 | MyBatis | MyBatis-Plus |
|------|---------|--------------|
| 单表CRUD | 需手写SQL或通用Mapper | BaseMapper自动化，零SQL |
| 条件构造 | 手写动态SQL或Example | Wrapper/LambdaWrapper链式编程 |
| 分页 | 手写LIMIT或集成PageHelper | 内置PaginationInnerInterceptor |
| 主键策略 | 手动处理 | ASSIGN_ID(雪花)/AUTO/UUID自动 |
| 逻辑删除 | 手动UPDATE+查询添加条件 | @TableLogic自动转换 |
| 乐观锁 | 手动WHERE version+SET version=version+1 | @Version插件自动处理 |
| 自动填充 | 手动set值 | MetaObjectHandler+@TableField自动填充 |
| 代码生成 | 需集成MBG | 内置AutoGenerator |
| 多租户 | 手动处理 | tenantLineHandler自动SQL注入 |

**选型建议**：
- 新项目：优先使用MyBatis-Plus，开发效率高，减少大量重复代码
- 已有MyBatis项目：可渐进迁移，两者完全兼容（MP只是增强工具）
- 单表操作多的项目：MP优势最大
- 复杂多表关联：两者都可胜任，MP不改变MyBatis任何功能，复杂查询仍可手写SQL

## 2.2 BaseMapper提供了哪些常用方法？
BaseMapper<T> 提供的核心方法：
- **插入**：insert(T entity)
- **删除**：deleteById(Serializable id)、deleteByMap(Map<String,Object>)、delete(Wrapper<T>)、deleteBatchIds(Collection<?> ids)
- **更新**：updateById(T entity)、update(T entity, Wrapper<T>)
- **查询单条**：selectById(Serializable id)、selectOne(Wrapper<T>)、selectBatchIds(Collection<?> ids)
- **查询列表**：selectList(Wrapper<T>)、selectByMap(Map<String,Object>)
- **查询数量**：selectCount(Wrapper<T>)
- **分页查询**：selectPage(IPage<T> page, Wrapper<T>)、selectMapsPage(IPage<T> page, Wrapper<T>)
- **其他**：selectMaps(Wrapper<T>)（返回Map列表）、selectObjs(Wrapper<T>)（返回第一个字段值的列表）、exists(Wrapper<T>)

所有方法都不需要写SQL，MyBatis-Plus根据实体类注解和Wrapper条件自动生成SQL。

## 2.3 MyBatis-Plus的主键策略有哪些？雪花算法的原理是什么？
**主键策略(IdType)**：
- **AUTO(0)**：数据库ID自增。数据库字段必须设置为AUTO_INCREMENT。受数据库单机限制，不适合分库分表
- **NONE(1)**：无策略，跟随全局设置（mybatis-plus.global-config.db-config.id-type配置）
- **INPUT(2)**：用户手动设置ID值后插入。如果用户未设置，insert时ID为null（可能报错）
- **ASSIGN_ID(3)**：默认策略，MP 3.3.0+的默认值。使用雪花算法生成Long类型ID（19位数字），分布式唯一、趋势递增
- **ASSIGN_UUID(4)**：生成32位UUID字符串（去除了横线）

**雪花算法原理**（Twitter Snowflake）：
```
0 | 0000000000 0000000000 0000000000 0000000000 0 | 00000 | 00000 | 000000000000
1bit符号    41bit时间戳(毫秒)                             5bit数据中心  5bit机器ID  12bit序列号
```
- **1bit符号位**：固定为0，表示正数
- **41bit时间戳**：存储从起始时间点(2010-11-04 01:42:54)以来的毫秒数，可支持约69年
- **10bit工作机器ID**：5bit数据中心ID + 5bit机器ID，最多支持1024个节点
- **12bit序列号**：同一毫秒内最多生成4096个ID（0~4095），超出则等待下一毫秒

优势：趋势递增（数据库索引友好，B+Tree插入效率高）、分布式唯一（不依赖数据库/Redis）、高性能（本地算法，无网络开销）、无第三方依赖

## 2.4 MyBatis-Plus的逻辑删除如何实现？有哪些注意事项？
**实现原理**：
1. 实体类字段添加`@TableLogic`注解，配置value="0"（未删除）和delval="1"（已删除）
2. 数据库表添加deleted字段（int类型，默认值0）
3. MyBatis-Plus在运行时自动修改SQL行为：
   - **DELETE语句转UPDATE**：`DELETE FROM user WHERE id=1` → `UPDATE user SET deleted=1 WHERE id=1`
   - **SELECT自动加过滤**：`SELECT * FROM user WHERE age>18` → `SELECT * FROM user WHERE age>18 AND deleted=0`
   - **UPDATE自动加过滤**：`UPDATE user SET name='xx' WHERE id=1` → `UPDATE user SET name='xx' WHERE id=1 AND deleted=0`

**注意事项**：
1. **唯一索引冲突**：逻辑删除后物理数据仍存在，如果表有唯一索引（如user_name唯一），被删除的user_name仍然占用，重新插入同名会报冲突。解决方案：联合唯一索引包含deleted（如(user_name, deleted)联合唯一）或改为程序层面校验
2. **查询性能**：所有查询自动加deleted条件，如果deleted字段没有索引，全表查询可能受影响。deleted字段需单独建索引或联合索引
3. **关联查询**：手动关联查询时需自行在SQL中加上deleted条件，MyBatis-Plus不会自动处理关联查询中的逻辑删除过滤
4. **不能再物理删除**：配置了@TableLogic后无法使用物理DELETE（需要物理删除的场景需要额外处理）

## 2.5 MyBatis-Plus的乐观锁如何实现？底层SQL是怎么生成的？
**实现步骤**：
1. 数据库表添加version字段（int类型，默认0或1）
2. 实体类字段添加`@Version`注解：`@Version private Integer version;`
3. 注册乐观锁插件：`mybatisPlusInterceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor())`
4. 更新操作前必须先查询获取当前version值，设置到实体中再更新

**底层SQL生成**：
```java
// 1. 先查询
User user = userMapper.selectById(1L); // 假设version=2

// 2. 修改属性
user.setName("新名字");

// 3. 执行更新，MP自动生成SQL：
// UPDATE user SET name='新名字', version=3 WHERE id=1 AND version=2
userMapper.updateById(user);
```

关键：WHERE条件中增加了`AND version=2`，如果version已不是2（说明被其他线程修改了），影响行数为0，MyBatis-Plus自动抛出OptimisticLockerException（可全局异常处理）。

## 2.6 MyBatis-Plus的分页插件如何配置和使用？
**配置**（MyBatis-Plus 3.4.0+）：
```java
@Configuration
public class MybatisPlusConfig {
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        PaginationInnerInterceptor paginationInterceptor = new PaginationInnerInterceptor(DbType.MYSQL);
        // 单页最大限制500条
        paginationInterceptor.setMaxLimit(500L);
        // 溢出总页数后是否处理（默认不处理，设为true则回到首页）
        paginationInterceptor.setOverflow(true);
        interceptor.addInnerInterceptor(paginationInterceptor);
        return interceptor;
    }
}
```

**使用**：
```java
// 分页查询
Page<User> page = new Page<>(1, 10);  // 第1页，每页10条
IPage<User> result = userMapper.selectPage(page,
    new LambdaQueryWrapper<User>().gt(User::getAge, 18));

// result.getRecords() 数据列表
// result.getTotal()   总记录数
// result.getCurrent() 当前页码
// result.getPages()   总页数
```

分页插件原理：拦截Executor.query()方法，从参数中提取Page对象，修改SQL添加方言分页（如MySQL的LIMIT offset,size），执行分页查询后再执行COUNT查询获取总数，最后将结果设置回Page对象。

## 2.7 QueryWrapper和LambdaQueryWrapper有什么区别？为什么推荐LambdaQueryWrapper？
| 对比 | QueryWrapper | LambdaQueryWrapper |
|------|-------------|-------------------|
| 字段名方式 | 字符串："name" | Lambda：User::getName |
| 编译期检查 | 无（字符串拼错运行时报错） | 有（编译期报错） |
| IDE支持 | 无代码提示 | 自动补全、重构支持 |
| 可读性 | 一般 | 高（类型安全） |
| 性能 | 相同 | 相同 |
| 使用场景 | 动态字段名（如配置表） | 绝大多数场景（推荐） |

```java
// QueryWrapper - 字符串字段名
queryWrapper.eq("name", "张三").gt("age", 18);
// 若字段名写错（如"nmae"），运行时才会暴露错误

// LambdaQueryWrapper - Lambda表达式
lambdaQuery.eq(User::getName, "张三").gt(User::getAge, 18);
// 字段名写错编译期就会报错，IDE可自动重构
```

推荐LambdaQueryWrapper是因为**类型安全+IDE友好**，减少低级错误。MySQL字段名与Java属性名不同时采用`@TableField("column_name")`映射，Lambda会自动解析正确的列名。

## 2.8 MyBatis-Plus的自动填充功能如何实现？如何获取当前操作人？
**实现步骤**：
1. 实体类字段添加`@TableField`注解指定填充时机：
```java
@TableField(fill = FieldFill.INSERT)
private Date createTime;
@TableField(fill = FieldFill.INSERT_UPDATE)
private Date updateTime;
@TableField(fill = FieldFill.INSERT)
private String createBy;
@TableField(fill = FieldFill.INSERT_UPDATE)
private String updateBy;
```

2. 实现MetaObjectHandler：
```java
@Component
public class MyMetaObjectHandler implements MetaObjectHandler {
    @Override
    public void insertFill(MetaObject metaObject) {
        this.strictInsertFill(metaObject, "createTime", Date.class, new Date());
        this.strictInsertFill(metaObject, "updateTime", Date.class, new Date());
        String userId = getCurrentUserId();
        this.strictInsertFill(metaObject, "createBy", String.class, userId);
        this.strictInsertFill(metaObject, "updateBy", String.class, userId);
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        this.strictUpdateFill(metaObject, "updateTime", Date.class, new Date());
        this.strictUpdateFill(metaObject, "updateBy", String.class, getCurrentUserId());
    }

    private String getCurrentUserId() {
        // 从SecurityContext/Session/ThreadLocal获取当前登录用户
        return UserContextHolder.getCurrentUserId();
    }
}
```

获取当前用户的方式：
- 通过RequestContextHolder获取HttpServletRequest，从Session/Header中提取
- 使用ThreadLocal（在登录拦截器中设置，退出时清理）
- 使用Spring Security的SecurityContextHolder

## 2.9 MyBatis-Plus的多租户是如何实现的？
通过**tenantLineHandler**在多租户插件中实现SQL自动注入租户条件：

```java
@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
    interceptor.addInnerInterceptor(new TenantLineInnerInterceptor(new TenantLineHandler() {
        @Override
        public Expression getTenantId() {
            // 从当前上下文获取租户ID（如ThreadLocal、Session）
            return new LongValue(TenantContextHolder.getCurrentTenantId());
        }

        @Override
        public String getTenantIdColumn() {
            return "tenant_id";  // 表中区分租户的字段名
        }

        @Override
        public boolean ignoreTable(String tableName) {
            // 某些公共表不需要租户隔离（如字典表、配置表）
            return "sys_dict".equalsIgnoreCase(tableName)
                || "sys_config".equalsIgnoreCase(tableName);
        }
    }));
    return interceptor;
}
```

**效果**：所有SELECT/INSERT/UPDATE/DELETE SQL自动附加租户条件，如`SELECT * FROM user WHERE age>18` → `SELECT * FROM user WHERE age>18 AND tenant_id=1001`。租户ID从当前请求上下文动态获取，实现数据隔离。

## 2.10 MyBatis-Plus的代码生成器如何配置使用？
**MyBatis-Plus 3.5.3+版本**：
```java
FastAutoGenerator.create("jdbc:mysql://localhost:3306/db", "root", "password")
    .globalConfig(builder -> builder
        .author("开发者")           // 作者名
        .outputDir("D://output")    // 输出路径
        .dateType(DateType.TIME_PACK)  // 时间类型策略（LocalDateTime）
        .enableSpringdoc()         // 启用Springdoc注解
    )
    .packageConfig(builder -> builder
        .parent("com.example")     // 父包名
        .entity("entity")          // 实体类包名
        .mapper("mapper")          // mapper接口包名
        .service("service")        // service接口包名
        .serviceImpl("service.impl")
        .controller("controller")
        .xml("mapper.xml")
    )
    .strategyConfig(builder -> builder
        .addInclude("user", "order", "product")  // 要生成的表
        .addTablePrefix("t_", "tb_")             // 过滤表前缀
        .entityBuilder()
            .enableLombok()               // 使用Lombok
            .enableTableFieldAnnotation() // 生成@TableField注解
            .logicDeleteColumnName("deleted")  // 逻辑删除字段
            .versionColumnName("version")      // 乐观锁字段
        .mapperBuilder()
            .enableBaseResultMap()
            .enableBaseColumnList()
        .serviceBuilder()
            .formatServiceFileName("%sService")
            .formatServiceImplFileName("%sServiceImpl")
        .controllerBuilder()
            .enableRestStyle()           // REST风格
    )
    .templateEngine(new FreemarkerTemplateEngine())  // 模板引擎
    .execute();
```

生成内容：Entity（含注解）、Mapper（继承BaseMapper）、Service/ServiceImpl、Controller（RESTful）、Mapper XML。

## 2.11 MyBatis-Plus的批量操作saveBatch与foreach拼接SQL有什么区别？
**saveBatch/updateBatchById**（ServiceImpl提供）：
- 底层循环调用mapper.insert()/updateById()，使用**JDBC的addBatch() + executeBatch()**批量提交
- 本质是**多条独立的SQL语句批量提交**（不是一条SQL插入多条数据）
- 默认每1000条flush一次Statement
- 优点：不会超出数据库的max_allowed_packet限制、每条数据独立处理（支持主键生成策略等）
- 缺点：比一条SQL插入多行数据稍慢

**foreach拼接（手写XML中的foreach标签）**：
- 将多条数据拼接为一条SQL：`INSERT INTO user VALUES (),(),()...`
- 本质是**一条SQL插入多行数据**
- 优点：性能最高（一次SQL解析+执行）
- 缺点：受数据库max_allowed_packet限制（默认4MB），数据量大需要分批；SQL过长可能超时

选择建议：少量数据（几百以内）用foreach；大量数据用saveBatch分批；中等数量两者均可。

## 2.12 如何防止MyBatis-Plus中误操作导致的全表删除或更新？
配置**BlockAttackInnerInterceptor**（攻击阻断拦截器）：
```java
@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
    interceptor.addInnerInterceptor(new BlockAttackInnerInterceptor());
    return interceptor;
}
```

**拦截规则**：
- DELETE语句如果没有WHERE条件：`DELETE FROM user` → 抛异常
- UPDATE语句如果没有WHERE条件：`UPDATE user SET name='x'` → 抛异常
- 对于WHERE条件中有人为拼接的全量条件（如`WHERE 1=1`），拦截器无法识别，需代码Review辅助

**补充措施**：
- SQL审计（生产环境开启MySQL审计日志或中间件SQL审计）
- 数据库权限限制（应用程序账号限制DDL/DROP权限）
- Code Review确保UPDATE/DELETE都有合理WHERE条件
- 定期备份+延迟从库（如MySQL延时复制）

## 2.13 MyBatis-Plus的动态数据源@DS注解如何实现读写分离？
**引入依赖**：`dynamic-datasource-spring-boot-starter`

**配置多数据源**（application.yml）：
```yaml
spring:
  datasource:
    dynamic:
      primary: master  # 默认数据源
      strict: false
      datasource:
        master:
          url: jdbc:mysql://ip1:3306/db
          username: root
          password: xxx
        slave1:
          url: jdbc:mysql://ip2:3306/db
          username: root
          password: xxx
        slave2:
          url: jdbc:mysql://ip3:3306/db
          username: root
          password: xxx
```

**使用@DS注解切换**：
```java
@Service
public class UserService {
    @DS("slave")  // 读操作走从库
    public List<User> listUsers() {
        return userMapper.selectList(null);
    }

    @DS("master")  // 写操作走主库（也可不写，因primary=master）
    public void saveUser(User user) {
        userMapper.insert(user);
    }
}
```

@DS可加在类上（所有方法默认走该数据源）或方法上（优先级高于类）。底层基于AOP在方法执行前根据@DS值从DynamicRoutingDataSource中选择对应实际DataSource。

## 2.14 MyBatis-Plus插件拦截器链的执行顺序是怎样的？多个拦截器如何协调？
MybatisPlusInterceptor是一个复合拦截器，内部维护一个`List<InnerInterceptor>`列表。所有内部拦截器按添加顺序依次执行SQL拦截逻辑：

- **beforeQuery/beforeUpdate/beforePrepare**等：按添加顺序执行（前置处理，如分页生成COUNT和LIMIT SQL、多租户添加条件）
- **willDoQuery/willDoUpdate**等：按添加顺序逆序执行（后置处理）

**典型配置顺序**（推荐）：
```java
interceptor.addInnerInterceptor(new PaginationInnerInterceptor());     // 1. 分页（必须先改SQL）
interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor()); // 2. 乐观锁
interceptor.addInnerInterceptor(new TenantLineInnerInterceptor(...));   // 3. 多租户
interceptor.addInnerInterceptor(new BlockAttackInnerInterceptor());     // 4. 阻断攻击（最后校验）
```

不同InnerInterceptor负责不同功能，互不干扰：分页负责修改SQL添加LIMIT；多租户负责添加租户条件；乐观锁负责WHERE条件和SET版本号+1；阻断攻击负责校验WHERE是否为空。

## 2.15 MyBatis-Plus的枚举类型如何映射？
**方式一：@EnumValue注解**（推荐，MP 3.5.3+）：
```java
public enum GenderEnum {
    MALE(1, "男"),
    FEMALE(2, "女");

    @EnumValue  // 标识数据库存的值
    private final int code;
    private final String desc;
}
```
实体类中直接使用`GenderEnum gender`，数据库字段类型为int。插入/查询时自动转换。

**方式二：配置type-enums-package**（批量扫描）：
```yaml
mybatis-plus:
  type-enums-package: com.example.enums
```
该包下所有枚举类自动注册类型处理器，无需每个枚举都加配置。

**方式三：自定义TypeHandler**：实现`BaseTypeHandler<T>`接口，在`@TableField(typeHandler = GenderTypeHandler.class)`中指定。

## 2.16 MyBatis-Plus 更新时 null 字段的处理策略（FieldStrategy）
MP 默认在 UPDATE 语句中**忽略 null 值字段**（`FieldStrategy.NOT_NULL`），这是为了防止误将 null 值写入数据库覆盖原有数据。但有时确实需要将某个字段更新为 null。

**五种策略**：
| 策略 | 含义 | WHERE 条件行为 | INSERT 行为 |
|------|------|-------------|------------|
| NOT_NULL（默认） | 非 null 才操作 | UPDATE 仅非 null 字段 | INSERT 仅非 null 字段 |
| NOT_EMPTY | 非 null 且非空才操作（如 String 非空串） | 同上 | 同上 |
| IGNORED | 忽略校验，任何值都操作 | null 字段也会添加 WHERE | null 字段也会插入 |
| NEVER | 不操作该字段 | 不参与 WHERE | 不参与 INSERT |
| DEFAULT | 紧跟全局策略 | — | — |

**设置策略**：
```java
@TableField(updateStrategy = FieldStrategy.IGNORED)  // 该字段允许更新为 null
private String remark;

@TableField(whereStrategy = FieldStrategy.NOT_EMPTY)  // WHERE 条件中忽略空串
private String name;
```

或全局配置：`mybatis-plus.global-config.db-config.update-strategy=ignored`。

**常见问题**：
- 更新为 null 失败：字段使用 `NOT_NULL` 策略，需要改为 `IGNORED` 或 `NOT_EMPTY` 中的合适策略。
- WHERE 条件中 null 字段不生效：使用 `IGNORED` 策略，MP 会自动将 null 值放入 WHERE 条件中。
- 查询条件：`QueryWrapper.eq("name", null)` → 默认会被忽略不生成 WHERE 条件（防止全表查询），可用 `QueryWrapper.isNull("name")` 显式指定。

## 2.17 MyBatis-Plus 中如何实现复杂多表关联分页？
MP 的自动分页只适用于**单表查询**（通过 `PaginationInnerInterceptor` 自动在 SQL 后追加 LIMIT）。对于多表 JOIN 查询，需要手写 XML 中的关联 SQL 配合分页。

**实现方式**：
```java
// Mapper 接口
@Select("SELECT u.*, o.order_no FROM t_user u LEFT JOIN t_order o ON u.id = o.user_id WHERE u.age > #{age}")
IPage<UserVO> selectUserWithOrders(Page<UserVO> page, @Param("age") Integer age);
```

```java
// 调用
Page<UserVO> page = new Page<>(1, 20);
IPage<UserVO> result = userMapper.selectUserWithOrders(page, 18);
// result.getRecords() → 结果列表
// result.getTotal() → 总记录数
```

**注意事项**：
1. **必须传入 Page 参数**，且必须是第一个参数，MP 才能自动分页。
2. **COUNT 查询**：MP 会自动基于原 SQL 包装 `SELECT COUNT(*) FROM (原SQL) AS TOTAL`。如果原 SQL 中有 `ORDER BY`，MP 会自动移除排序以提高 COUNT 效率（`optimizeCountSql=true` 默认开启）。
3. **多表 JOIN 分页性能**：当数据量大时，`SELECT COUNT(*) FROM (多表JOIN) AS TOTAL` 会很慢。可以手动优化：
   ```java
   // 关闭自动 COUNT，传入已知总数
   page.setSearchCount(false);
   page.setTotal(knownTotal);
   ```
4. **一对多结果集的正确映射**：如果 LEFT JOIN 产生一对多结果，每行数据中主表部分重复，需要在 ResultMap XML 中正确配置 `<id>` 标签和 `<collection>` 映射，MyBatis 会自动对主键去重组装。
5. **子查询分页**：如果多表 JOIN 太复杂，可以先对主表分页查出 ID 列表，再 IN 查询关联数据组装结果。

## 2.18 MyBatis-Plus 的 ActiveRecord 模式
ActiveRecord 模式让实体类直接继承 `Model<T>` 获得 CRUD 能力，无需注入 Mapper/Service：

```java
@Data
@TableName("t_user")
public class User extends Model<User> {
    private Long id;
    private String name;
}

// 直接使用
User user = new User();
user.setName("zhangsan");
user.insert();           // 新增
user.selectById(1L);     // 查
user.setName("lisi");
user.updateById();       // 改
user.deleteById();       // 删
```

**优缺点**：
- 优点：代码更简洁，无需注入 Mapper（适合简单 CRUD 场景）。
- 缺点：业务逻辑与数据库操作耦合在一个类中，违反单一职责原则，不利于单元测试（不可 Mock）。

**选择建议**：简单 CRUD 的快速原型开发适用；正式的微服务项目不推荐——使用 Mapper + Service 层更符合分层架构。实际上自 MyBatis Plus 3.0 起官方也不推荐 ActiveRecord，建议使用常规的 Mapper 注入方式。
