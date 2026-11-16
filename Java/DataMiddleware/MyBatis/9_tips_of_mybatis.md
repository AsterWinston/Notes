# 1. #{}和${}

**#{} 是占位符，${} 是字符串拼接**。#{} 底层是 PreparedStatement 实现：每个 #{} 在解析阶段被替换成 JDBC 占位符 `?`，参数以绑定变量传给数据库，**可以防止 SQL 注入，是绝大多数场景的默认选择**。${} 则先拼接 SQL 文本、再编译执行，底层是 Statement 实现，变量值直接成为语句的一部分，**存在 SQL 注入现象**，只有在需要进行 SQL 语句关键字拼接的情况下才会用到。

**必须使用 ${} 的时机**：拼接排序关键字（向 SQL 注入 asc/desc 完成升降序——写成 `order by #{order}` 时参数被当作字符串常量，等价于按常量排序，排序失效）；拼接表名（前端传参决定查询 t_log20221201 等按天分表）；拼接列名。

> **注意**：${} 拼进去的值一旦来自外部输入就是注入点。线上必须对 order 这类参数做**白名单校验**（只允许 "asc"/"desc"），对表名做前缀正则校验，绝不能把用户原始输入直接透传给 ${}。

模糊查询有三种解决方法：

```sql
-- 错误示范：'%${name}%' 直接把值拼进 SQL 文本，存在注入风险
name like '%${name}%'

-- 正确做法：concat 函数 + #{} 占位符，参数走绑定变量，防注入且跨库可用
name like concat('%', #{name}, '%')

-- 可用：双引号使 "%"#{name}"%" 整体作为一个字符串参数交给 #{}
name like "%"#{name}"%"
```

# 2. typeAliases

**typeAliases 的作用是给 Java 类型起别名，缩短 XML 中 resultType / resultMap type 的书写**。别名在 mybatis-config.xml 的 `<typeAliases>` 标签中指定，位置有严格约束：写在 settings 之后、environments 之前——XMLConfigBuilder 按 DTD 规定的顺序解析子标签，顺序颠倒直接抛异常。

三条规则：**别名不区分大小写**；namespace 不能使用别名，只能写接口全限定名；alias 省略时默认就是类简名。更省事的是 `<package>` 批量扫描，包下所有类自动以类简名注册进 TypeAliasRegistry。坑在于不同子包的**同名类**会撞同一别名，后注册覆盖先注册，冲突时显式 alias 区分或退回全限定名。

```xml
<!-- 在mybatis-config.xml中指定（settings之后，environments之前） -->
<!--起别名-->
<typeAliases>
    <!--
         type:指定给哪个类型起别名
         alias:指定别名
         别名不区分大小写
         namespace不能使用别名
         alias属性可以省略，默认就是类的简名（Car）
    -->
    <typeAlias type="com.ziv.mybatis.pojo.Car" alias="car"/>
    <typeAlias type="com.ziv.mybatis.pojo.Car"/>
    <!-- 使用package将对应包下的所有类全部自动起别名，默认就是类的简名 -->
    <package name="com.ziv.mybatis.pojo"/>
</typeAliases>

<!-- 在CarMapper.xml标签resultType属性中使用 -->
<select id="selectAll" resultType="car">
    select id,
    car_num      as carNum,
    brand,
    guide_price  as guidePrice,
    produce_time as produceTime,
    car_type     as carType
    from t_car
</select>
```

> **行业实践**：与 Spring 集成或使用 starter 时一般不再手写这段 XML，一行 `mybatis.type-aliases-package` 即可完成同样的包扫描。

# 3. mappers

`<mappers>` 负责把映射文件或映射接口注册进 Configuration，**没注册的语句 MyBatis 根本不认识**，运行时表现为"找不到语句"的绑定异常。resource/url 面向 XML 为中心的传统工程；class 与 package 面向 Mapper 接口：注册接口时会顺带解析同目录同名 XML，同一 id 两边重复定义在启动期报重复映射。

```xml
<!--
    mapper的属性：
        resource：从类路径中加载
        url：从指定的全限定资源路径中加载
        class：使用映射器接口的完全限定类名。需将XxxMapper.xml放到与接口同目录下，
               且接口名与XML名一致。指定class后，mybatis会到对应classpath目录下找同名XML
        package：将包内的映射器接口实现全部注册为映射器，同样要求XML与同名接口同目录，
                 比逐个<mapper class=""/>更高效
-->
<mapper resource=""/>
<mapper class=""/>
<mapper url=""/>
<package name="com.ziv.mybatis.mapper"/>
```

# 4. idea配置文件模板

每次新建配置文件都要复制几十行 DTD 声明头，容易拼错。利用 IDEA 的 **File → Settings → Editor → File and Code Templates** 固化模板，并可用内置变量 `${PACKAGE_NAME}`、`${NAME}` 自动拼出 namespace——**namespace 是 Mapper XML 与接口绑定的桥梁**，接口全限定名必须与它完全一致，写错就是绑定失败。Configuration 模板：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <properties resource=""/>
    <typeAliases>
        <package name=""/>
    </typeAliases>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="${jdbc.driver}"/>
                <property name="url" value="${jdbc.url}"/>
                <property name="username" value="${jdbc.username}"/>
                <property name="password" value="${jdbc.password}"/>
            </dataSource>
        </environment>
    </environments>
    <mappers>
        <package name=""/>
    </mappers>
</configuration>
```

Mapper 模板：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="">

</mapper>
```

# 5. 插入数据时获取自动生成的主键

业务上常见"插入后马上要用这条记录的 id"的场景（插完订单接着插明细）。自增主键由数据库生成，MyBatis 用 **useGeneratedKeys + keyProperty** 解决：底层等价于 `prepareStatement(sql, RETURN_GENERATED_KEYS)`，执行后取回自增值，再按 keyProperty **反射写回参数对象**。

```xml
<!--
    useGeneratedKeys="true" 使用自动生成的主键值
    keyProperty="id" 指定主键值赋值给对象的哪个属性。
    这个表示将自动生成的主键值赋值给Car的id属性
-->
<insert id="insertCarUseGeneratedKey" useGeneratedKeys="true" keyProperty="id">
    insert into t_car(id, car_num, brand, guide_price, produce_time, car_type)
    values (null, #{carNum}, #{brand}, #{guidePrice}, #{produceTime}, #{carType})
</insert>
```

```java
@Test
public void testInsertCarUseGeneratedKey() {
    SqlSession sqlSession = SqlSessionUtil.openSession();
    CarMapper mapper = sqlSession.getMapper(CarMapper.class);
    Car car = new Car(null, "9000", "Kaimeirui", 35.0, "2000-10-10", "燃油车");
    mapper.insertCarUseGeneratedKey(car);
    sqlSession.commit();
    sqlSession.close();
    System.out.println(car);
}
```

输出 car 之后发现 id 被赋上了自动生成的值。

**设计考量**：

- **MySQL 首选 useGeneratedKeys**：依赖驱动的 getGeneratedKeys 支持。
- **Oracle（12c 之前）没有自增列**，要用 `<selectKey order="BEFORE">select seq_car.nextval from dual</selectKey>` 先取序列再拼进 insert。
- **批量插入回填主键要谨慎**：依赖驱动与语句形态，跨库建议插入后按业务唯一键二次查询。

# 6. 多参数必须用 @Param

方法只有一个参数时，XML 里取值很宽容；一旦**多个参数，MyBatis 会把它们装进 ParamMap**，默认 key 是 arg0/arg1 与 param1/param2 这种可读性差、极易写错位的命名。

```java
// 错误示范：不加 @Param，XML 里只能写 #{arg0}、#{param1}，可读性差且容易错位
int deleteByBrandAndType(String brand, String carType);

// 正确做法：@Param 指定参数名，XML 里直接用 #{brand}、#{carType}
int deleteByBrandAndType(@Param("brand") String brand,
                         @Param("carType") String carType);
```

| 场景 | XML 中可用的 key |
|------|------------------|
| 单参数、无 @Param | 参数本身（基本类型任意名可取；POJO 用属性名） |
| 多参数、无 @Param | arg0、arg1… 与 param1、param2… 并存 |
| 使用 @Param("xxx") | xxx 与 param1、param2… 并存（argN 被替换掉） |

**@Param 替换的是 ParamMap 中 arg0、arg1 这一组 key，param1、param2 依然可用**——这是源码 ParamNameResolver 的产物（详见 10_mybatis_param_process.md）。工程上建议不论参数个数一律加 @Param：它是明确的编程契约，不依赖 `-parameters` 编译选项。

# 7. resultMap 复用：extends 与 sql 片段

多张表共享大量公共字段（id、create_time、update_time、deleted）时，每个 resultMap 从头写一遍既啰嗦又难维护。**resultMap 支持 extends 属性继承另一个 resultMap 的映射**，公共列声明一次，子 resultMap 只写增量；列清单复用交给 `<sql>` + `<include>`（见 12_dynamic_sql.md）。

```xml
<sql id="baseColumns">
    id, create_time, update_time, deleted
</sql>

<resultMap id="baseResultMap" type="BaseEntity">
    <id property="id" column="id"/>
    <result property="createTime" column="create_time"/>
    <result property="updateTime" column="update_time"/>
    <result property="deleted" column="deleted"/>
</resultMap>

<!-- extends 继承父 resultMap，只补充自己的列 -->
<resultMap id="carResultMap" type="Car" extends="baseResultMap">
    <result property="carNum" column="car_num"/>
    <result property="brand" column="brand"/>
</resultMap>

<select id="selectAll" resultMap="carResultMap">
    select
    <include refid="baseColumns"/>,
    car_num, brand, guide_price, produce_time, car_type
    from t_car
</select>
```

> **注意**：extends 的继承在解析阶段由 ResultMapResolver 递归处理，**父 resultMap 必须先被解析到**（同文件写在前面，跨 namespace 用全限定 refid），否则抛 "ResultMap extends not found"。`<include>` 也可跨文件复用片段，还能用 `<property>` 传参。

# 8. foreach 大批量插入要分片

foreach 拼接多值 insert（`insert into t_car values (...),(...),(...)`）把 N 行拼成**一条** SQL，一次网络往返完成插入，性能远好于逐行执行。但 SQL 文本超过 MySQL 的 `max_allowed_packet`（5.7 默认 4MB，8.0 默认 64MB）会被直接拒绝。

```java
// 正确做法：按固定片大小切分，循环分批调用（经验值：500~1000 行一片）
public int insertBatchInShards(List<Car> cars) {
    int shardSize = 500;
    int affected = 0;
    for (int i = 0; i < cars.size(); i += shardSize) {
        List<Car> shard = cars.subList(i, Math.min(i + shardSize, cars.size()));
        affected += carMapper.insertBatch(shard);
    }
    return affected;
}
```**设计考量**：

- **foreach 多值拼接**：适合几千行以内的批量导入，片大小按行宽估算，含大字段的宽表要调小。
- **ExecutorType.BATCH + 逐行 insert**：走 JDBC batch，MySQL 必须连接串加 `rewriteBatchedStatements=true` 才会合并成多值语句。
- **百万级导入**：绕开 MyBatis，用 LOAD DATA INFILE 或数据库厂商批量通道。

# 9. 懒加载配置与它的坑

**懒加载的本质是 association / collection 分步查询时，主对象先加载，级联对象推迟到第一次调用 getter 时才执行子 SQL**。开启后返回的是代理对象，只有 getXxx() 获取级联对象时才触发相应 SQL（机制详见 13_advanced_mapping_and_lazy_loading.md）。

```xml
<settings>
    <!-- 全局开启懒加载：所有association/collection默认lazy -->
    <setting name="lazyLoadingEnabled" value="true"/>
    <!-- 关闭"激进式懒加载"，避免一次触发全部懒属性 -->
    <setting name="aggressiveLazyLoading" value="false"/>
</settings>
```

```xml
<!-- 局部控制：fetchType优先级高于全局配置 -->
<association property="clazz"
             select="com.ziv.mybatis.mapper.ClazzMapper.selectByIdStepTwo"
             column="cid"
             fetchType="lazy"/>   <!-- 全局开启后可用 fetchType="eager" 单独关闭 -->
```

| 配置组合 | 实际效果 |
|----------|----------|
| lazyLoadingEnabled=false + fetchType="lazy" | 仅此 association 懒加载（**推荐，影响面最小**） |
| lazyLoadingEnabled=true + fetchType="eager" | 全局懒加载，仅此条立即加载 |
| aggressiveLazyLoading=true | 触发条件激进，toString/调试器求值可能一次加载完全部懒属性，懒加载形同虚设 |

**懒加载只能在 SqlSession 存活期内触发**：纯 MyBatis 下 session 关闭后访问懒属性拿不到数据；与 Spring 集成时 session 由事务控制，**事务边界之外访问未加载的级联属性同样踩空**。

# 10. XML 转义：&lt; 与 CDATA

XML 对 `<`、`>` 等字符有保留含义，而 SQL 里最常用的恰恰是大小于比较。**`<` 必须写成 `&lt;`，`>` 建议写成 `&gt;`**，直接写比较号会让 SAX 解析器在启动阶段就抛异常。

```xml
<!-- 正确做法 1：转义符 -->
<select id="selectByPrice" resultType="Car">
    select * from t_car where guide_price &lt;= #{price} and guide_price &gt; 0
</select>

<!-- 正确做法 2：CDATA块，内部内容原样保留，最直观 -->
<select id="selectByPrice2" resultType="Car">
    <![CDATA[
        select * from t_car where guide_price <= #{price} and guide_price > 0
    ]]>
</select>
```

**动态 SQL 的 test 属性里是 OGNL 表达式而不是 SQL 文本**，写比较时同样要用 `&lt;` 转义（如 `test="age &lt; 30"`）。URL 拼接参数中的 `&` 要写成 `&amp;`；OGNL 中不等于统一用 `!=`，避免 `<>` 与标签语法纠缠。

# 11. typeHandler 相关坑

**typeHandler 负责 Java 类型与 JDBC 类型之间的双向转换**（setParameter 写入、getResult 读出），坑集中在两类场景。其一是 **null 值写入**：传 null 时 MyBatis 默认按 `jdbcType=OTHER` 处理，MySQL 驱动容忍，**Oracle 驱动直接抛"无效的列类型"异常**。解法：全局设置 `<setting name="jdbcTypeForNull" value="NULL"/>`，或参数上标注 `#{email,jdbcType=VARCHAR}`。其二是 **自定义 typeHandler 注册范围过宽**：`<typeHandlers>` 全局注册的处理器会拦截该类型组合的所有转换；应优先局部声明——`<result ... typeHandler="..."/>` 或 `#{profile, typeHandler=...}`（完整机制见 23_type_handler.md）。

# 12. 小结

本篇围绕使用层面的高频技巧收束：**#{} 防注入是默认，${} 仅限关键字/表名拼接且必须白名单校验**；typeAliases 与 mappers 遵守 DTD 标签顺序；主键回填 MySQL 用 useGeneratedKeys、Oracle 用 selectKey；多参数一律 @Param；resultMap 用 extends、列清单用 sql/include 复用；foreach 批量分片防 max_allowed_packet；懒加载优先 fetchType 局部控制；比较符用 `&lt;` 或 CDATA，Oracle 注意 jdbcTypeForNull。共同底色是一句话：**让拼接止于白名单、让映射收敛到声明、让全局配置只开必要的最小范围**。
