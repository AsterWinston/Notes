# 1.MyBatis-Plus 概述

```
MyBatis-Plus（简称 MP）是一个 MyBatis 的增强工具，
在 MyBatis 的基础上只做增强不做改变，为简化开发、提高效率而生。

官方愿景：成为 MyBatis 最好的搭档，就像魂斗罗中的 1P 和 2P。

关键特征：
  - 无侵入：只做增强不做改变，引入它不会对现有工程产生影响
  - 损耗小：启动即会自动注入基本 CRUD，性能基本无损耗
  - 强大的 CRUD 操作：内置通用 Mapper、通用 Service
  - 支持 Lambda 形式调用：通过 Lambda 表达式编写条件，无需担心字段写错
  - 支持多种数据库：MySQL、Oracle、SQL Server、PostgreSQL 等
  - 内置分页插件、逻辑删除、乐观锁等功能
```

# 2.MyBatis-Plus 与 MyBatis 的关系

```
┌─────────────────────────────────────────────────────────┐
│                    MyBatis-Plus                          │
│  ┌───────────────────────────────────────────────────┐  │
│  │  增强层（MP 特有）                                    │  │
│  │  • BaseMapper<T> — 通用 CRUD                       │  │
│  │  • IService<T> — 通用 Service                       │  │
│  │  • LambdaQueryWrapper — Lambda 条件构造器            │  │
│  │  • 分页插件（PaginationInnerInterceptor）           │  │
│  │  • 乐观锁（OptimisticLockerInnerInterceptor）        │  │
│  │  • 逻辑删除（@TableLogic）                           │  │
│  │  • 自动填充（@TableField(fill=...)）                 │  │
│  │  • 代码生成器                                       │  │
│  │  • 多数据源（dynamic-datasource）                    │  │
│  └───────────────────────────────────────────────────┘  │
│                         │                               │
│  ┌───────────────────────────────────────────────────┐  │
│  │  MyBatis 核心层（完全复用）                            │  │
│  │  • MapperProxy / MapperMethod                       │  │
│  │  • Executor / StatementHandler                      │  │
│  │  • SqlSession / Configuration                       │  │
│  │  • 一级缓存 / 二级缓存                                │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

# 3.快速开始

## 3.1 依赖引入

```xml
<!-- Spring Boot Starter -->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.5.5</version>
</dependency>

<!-- MySQL 驱动 -->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.33</version>
</dependency>
```

## 3.2 配置

```yaml
# application.yml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/test
    username: root
    password: 123456

# MP 配置
mybatis-plus:
  configuration:
    map-underscore-to-camel-case: true    # 驼峰映射
    log-impl: org.apache.ibatis.logging.slf4j.Slf4jImpl
  global-config:
    db-config:
      id-type: auto                       # 主键策略：自增
      logic-delete-field: deleted         # 逻辑删除字段
      logic-delete-value: 1               # 已删除值
      logic-not-delete-value: 0           # 未删除值
  mapper-locations: classpath:mapper/*.xml
```

## 3.3 实体类

```java
@Data
@TableName("t_car")  // 指定表名
public class Car {
    @TableId(type = IdType.AUTO)  // 主键自增
    private Long id;

    @TableField("car_num")
    private String carNum;

    private String brand;

    @TableField("guide_price")
    private Double guidePrice;

    @TableField("produce_time")
    private Date produceTime;

    @TableField("car_type")
    private String carType;

    @TableLogic  // 逻辑删除
    private Integer deleted;

    @TableField(fill = FieldFill.INSERT)  // 插入时自动填充
    private Date createTime;

    @TableField(fill = FieldFill.INSERT_UPDATE)  // 插入和更新时自动填充
    private Date updateTime;
}
```

## 3.4 Mapper 接口

```java
/**
 * 继承 BaseMapper<T> 即可拥有通用 CRUD 方法
 *
 * BaseMapper 提供的方法（无需写任何 SQL）：
 *   insert(T entity)                     — 插入一条记录
 *   deleteById(Serializable id)          — 根据 ID 删除
 *   deleteByMap(Map<String, Object>)     — 根据条件删除
 *   delete(Wrapper<T>)                   — 根据条件构造器删除
 *   deleteBatchIds(Collection<?>)        — 批量删除
 *   updateById(T entity)                 — 根据 ID 更新
 *   update(T entity, Wrapper<T>)         — 根据条件更新
 *   selectById(Serializable id)          — 根据 ID 查询
 *   selectBatchIds(Collection<?>)        — 批量 ID 查询
 *   selectByMap(Map<String, Object>)     — 根据 Map 条件查询
 *   selectOne(Wrapper<T>)                — 查询一条
 *   selectCount(Wrapper<T>)              — 查询总数
 *   selectList(Wrapper<T>)               — 查询列表
 *   selectMaps(Wrapper<T>)               — 查询返回 Map
 *   selectObjs(Wrapper<T>)               — 查询返回第一个字段值
 *   selectPage(Page<T>, Wrapper<T>)      — 分页查询
 */
@Mapper
public interface CarMapper extends BaseMapper<Car> {
    // 复杂查询仍然可以自定义
    @Select("select * from t_car where brand like concat('%',#{brand},'%')")
    List<Car> selectByBrand(String brand);
}
```

# 4.条件构造器（Wrapper）

```java
/**
 * MP 提供了强大的条件构造器，支持链式调用和 Lambda 表达式
 *
 * Wrapper 体系：
 *   Wrapper (抽象类)
 *     ├── AbstractWrapper
 *     │     ├── QueryWrapper     — 字符串形式 ("car_num")
 *     │     ├── UpdateWrapper    — 更新条件
 *     │     └── AbstractLambdaWrapper
 *     │           ├── LambdaQueryWrapper  — Lambda 形式 (Car::getCarNum)
 *     │           └── LambdaUpdateWrapper — Lambda 更新
 *     └── AbstractChainWrapper (链式调用增强)
 */

// ====== QueryWrapper 示例（字符串形式）======
QueryWrapper<Car> wrapper = new QueryWrapper<>();
wrapper.eq("brand", "丰田")
       .gt("guide_price", 20)
       .like("car_num", "10")
       .between("produce_time", "2020-01-01", "2023-01-01")
       .orderByDesc("guide_price");
List<Car> cars = carMapper.selectList(wrapper);

// ====== LambdaQueryWrapper 示例（推荐，类型安全）======
LambdaQueryWrapper<Car> lambdaQuery = new LambdaQueryWrapper<>();
lambdaQuery.eq(Car::getBrand, "丰田")
           .gt(Car::getGuidePrice, 20)
           .like(Car::getCarNum, "10")
           .orderByDesc(Car::getGuidePrice);
List<Car> cars = carMapper.selectList(lambdaQuery);

// ====== 条件构造器常用方法 ======
/*
  eq("column", val)          → column = val
  ne("column", val)          → column <> val
  gt("column", val)          → column > val
  ge("column", val)          → column >= val
  lt("column", val)          → column < val
  le("column", val)          → column <= val
  between("column", v1, v2)  → column between v1 and v2
  notBetween(...)            → column not between v1 and v2
  like("column", val)        → column like '%val%'
  notLike(...)               → column not like '%val%'
  likeLeft("column", val)    → column like '%val'
  likeRight("column", val)   → column like 'val%'
  isNull("column")           → column is null
  isNotNull("column")        → column is not null
  in("column", coll)         → column in (v1,v2,...)
  notIn("column", coll)      → column not in (v1,v2,...)
  groupBy("column")          → group by column
  orderByAsc("column")       → order by column asc
  orderByDesc("column")      → order by column desc
  having("sql")              → having sql
  or()                       → 拼接 OR
  and(Consumer)              → 嵌套 AND
  apply("sql")               → 拼接自定义 SQL
  last("sql")                → 无视优化规则直接拼接在最后
  exists("sql")              → 拼接 EXISTS 子查询
  notExists("sql")           → 拼接 NOT EXISTS 子查询
  nested(Consumer)           → 正常嵌套（不带 AND 或 OR）
*/
```

# 5.分页插件

```java
/**
 * MP 内置了比 PageHelper 更简洁的分页方式
 */

// 1. 配置分页插件
@Configuration
public class MybatisPlusConfig {
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        // 添加分页插件
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        // 添加乐观锁插件
        interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());
        return interceptor;
    }
}

// 2. 使用分页
Page<Car> page = new Page<>(2, 10);  // 第2页，每页10条
Page<Car> result = carMapper.selectPage(page,
    new LambdaQueryWrapper<Car>().gt(Car::getGuidePrice, 20));

System.out.println("当前页: " + result.getCurrent());
System.out.println("每页大小: " + result.getSize());
System.out.println("总记录数: " + result.getTotal());
System.out.println("总页数: " + result.getPages());
System.out.println("数据: " + result.getRecords());

// 3. 自定义 SQL 分页（写 XML 时第一个参数必须是 Page）
// Mapper 接口
IPage<Car> selectCustomPage(Page<Car> page, @Param("brand") String brand);

// Mapper.xml
// <select id="selectCustomPage" resultType="Car">
//     select * from t_car where brand = #{brand}
// </select>
```

# 6.逻辑删除

```java
/**
 * 逻辑删除：不是真正从数据库中删除，而是标记为已删除
 *
 * 配置后，所有 deleteById/delete/deleteByMap 等方法都会变为 UPDATE 语句
 * 所有 select/selectList/selectPage 等方法都会自动添加 deleted = 0 条件
 */

// 1. 全局配置
// mybatis-plus.global-config.db-config.logic-delete-field=deleted
// mybatis-plus.global-config.db-config.logic-delete-value=1
// mybatis-plus.global-config.db-config.logic-not-delete-value=0

// 2. 实体类配置
@Data
public class Car {
    @TableLogic
    private Integer deleted;  // 0=正常, 1=已删除
}

// 3. 效果演示
carMapper.deleteById(1L);
// 执行的不是: delete from t_car where id = 1
// 而是: update t_car set deleted = 1 where id = 1 and deleted = 0

carMapper.selectById(1L);
// 执行的是: select * from t_car where id = 1 and deleted = 0
// 如果数据已被逻辑删除，这里查不到
```

# 7.乐观锁

```java
/**
 * 乐观锁：在并发场景下保证数据一致性
 *
 * 原理：更新时带上版本号条件，版本号不匹配则更新失败
 */

// 1. 配置插件（见上文分页插件配置中的 OptimisticLockerInnerInterceptor）

// 2. 实体类
@Data
public class Car {
    @Version  // 乐观锁版本字段
    private Integer version;
}

// 3. 使用
// 先查询
Car car = carMapper.selectById(1L);  // version = 1

// 修改
car.setBrand("新品牌");

// 更新
carMapper.updateById(car);
// 执行: update t_car set brand='新品牌', version=2 where id=1 and version=1
// 如果此期间有其他线程也更新了，version 已经变成 2，则此更新失败（影响行数为 0）
```

# 8.自动填充

```java
/**
 * 自动填充：在插入或更新时自动设置某些字段的值
 *
 * 常见场景：createTime、updateTime、createBy、updateBy
 */

// 1. 实体类
@Data
public class Car {
    @TableField(fill = FieldFill.INSERT)
    private Date createTime;

    @TableField(fill = FieldFill.INSERT_UPDATE)
    private Date updateTime;
}

// 2. 自动填充处理器
@Component
public class MyMetaObjectHandler implements MetaObjectHandler {

    @Override
    public void insertFill(MetaObject metaObject) {
        this.strictInsertFill(metaObject, "createTime", Date.class, new Date());
        this.strictInsertFill(metaObject, "updateTime", Date.class, new Date());
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        this.strictUpdateFill(metaObject, "updateTime", Date.class, new Date());
    }
}

// 3. 效果：插入或更新时，createTime 和 updateTime 字段自动填充，无需手动设置
```

# 9.Service 层增强

```java
/**
 * MP 还提供了通用的 Service 层封装 — IService 和 ServiceImpl
 */

// ====== Service 接口 ======
public interface CarService extends IService<Car> {
    // 继承 IService 获得通用方法
    // 自定义业务方法
    List<Car> findExpensiveCars(Double minPrice);
}

// ====== Service 实现 ======
@Service
public class CarServiceImpl extends ServiceImpl<CarMapper, Car> implements CarService {

    @Override
    public List<Car> findExpensiveCars(Double minPrice) {
        // 使用 baseMapper（Service 中可以直接访问 Mapper）
        return lambdaQuery()
                .gt(Car::getGuidePrice, minPrice)
                .list();
    }
}

// ====== Controller 使用 ======
@RestController
@RequestMapping("/car")
public class CarController {
    @Autowired
    private CarService carService;

    @GetMapping("/{id}")
    public Car getById(@PathVariable Long id) {
        return carService.getById(id);  // IService 提供的方法
    }

    @PostMapping
    public boolean save(@RequestBody Car car) {
        return carService.save(car);    // IService 提供的方法
    }
}
```

# 10.代码生成器

```java
/**
 * MP 提供了强大的代码生成器（3.5.1+ 推荐使用 FastAutoGenerator）
 */

public class CodeGenerator {
    public static void main(String[] args) {
        FastAutoGenerator.create(
                "jdbc:mysql://localhost:3306/test", "root", "123456")
            // 全局配置
            .globalConfig(builder -> {
                builder.author("ziv")           // 作者
                       .outputDir("D:/generated"); // 输出目录
            })
            // 包配置
            .packageConfig(builder -> {
                builder.parent("com.test")
                       .entity("pojo")
                       .mapper("mapper")
                       .service("service")
                       .serviceImpl("service.impl")
                       .controller("controller");
            })
            // 策略配置
            .strategyConfig(builder -> {
                builder.addInclude("t_car")     // 要生成的表
                       .addTablePrefix("t_")    // 表前缀过滤
                       .entityBuilder()
                       .enableLombok()          // 启用 Lombok
                       .enableTableFieldAnnotation()
                       .controllerBuilder()
                       .enableRestStyle();      // RESTful 风格
            })
            .execute();
    }
}
```

# 11.MP 与 MyBatis 的源码衔接点

```
MP 如何做到"只增强不做改变"？

1. Mapper 代理增强
   - MP 的 MybatisMapperProxy 继承自 MyBatis 的 MapperProxy
   - 在 invoke() 中增加了对 BaseMapper 方法的处理

2. SQL 注入器
   - MP 的 DefaultSqlInjector 在启动时注入通用 CRUD 方法
   - 为每个 BaseMapper 的方法创建对应的 MappedStatement
   - 注册到 Configuration 的 mappedStatements 中

3. 拦截器增强
   - PaginationInnerInterceptor（分页）
   - OptimisticLockerInnerInterceptor（乐观锁）
   - BlockAttackInnerInterceptor（防止全表更新/删除）
   - IllegalSQLInnerInterceptor（SQL 性能规范）
   - 这些拦截器本质上都是 MyBatis 的 Plugin

4. MybatisPlusAutoConfiguration
   - Spring Boot 自动配置类
   - 自动创建 MybatisSqlSessionFactoryBean
   - 替换标准的 MyBatis 自动配置

MP 与 MyBatis 的关系总结：
   - MP 没有修改 MyBatis 一行代码
   - MP 完全基于 MyBatis 的扩展点（Plugin、TypeHandler、MapperProxy 等）
   - MP 的 Mapper 可以直接使用 MyBatis 原生的 XML 和注解
   - MP 是一个"扩展包"而非"替代品"
```

# 12.什么时候应该使用 MyBatis-Plus

```
  适合使用 MP 的场景：
    1. 快速开发：不想为每个表写重复的 CRUD SQL
    2. 单表操作多：项目中大量是单表查询
    3. 分页需求：不想集成 PageHelper
    4. 逻辑删除/乐观锁：不想手动处理
    5. 条件查询多：Wrapper 比手写动态 SQL 更简洁

  不太适合MP的场景：
    1. 遗留系统改造：已有大量复杂 XML 映射
    2. 多表关联查询多：MP 在这方面没有优势
    3. 高度定制化：需要的不是通用 CRUD 而是精细的 SQL 优化
    4. 团队不熟悉：学习成本（虽然很低）

  最佳实践：
    1. MP + MyBatis 原生混合使用
    2. 简单 CRUD 用 MP 的 BaseMapper
    3. 复杂查询仍然用 XML 或注解
    4. 两者可以完美共存
```
