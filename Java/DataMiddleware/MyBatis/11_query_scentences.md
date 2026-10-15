# 1. 返回Bean
```
当查询的结果，有对应的实体类，并且查询结果只有⼀条时，可以直接用实体类接收，也可以用List集合接收。
```
# 2. 返回List
```
当查询的记录条数是多条的时候，必须使⽤集合接收。如果使⽤单个实体类接收会出现异常。
```
# 3. 返回Map
```
当返回的数据，没有合适的实体类对应的话，可以采⽤Map集合接收。字段名做key，字段值做value。查询如果可以保证只有⼀条数据，则返回⼀个Map集合即可。
```

# 4. 返回List
```
查询结果条数⼤于等于1条数据，则可以返回⼀个存储Map集合的List集合。
```
# 5. 返回Map<String,Map>（@MapKey）
```
拿Car的id做key，以后取出对应的Map集合时更⽅便。

/**
  * 获取所有的Car，返回⼀个Map集合。 
  * Map集合的key是Car的id。 
  * Map集合的value是对应Car。 
  * 需要在接口上添加@MapKey注解，指定id作为map的key
  * @return
  */
@MapKey("id")
Map<Long,Map<String,Object>> selectAllRetMap();

    <select id="selectAllRetMap" resultType="map">
        select id,
               car_num      carNum,
               brand,
               guide_price  guidePrice,
               produce_time produceTime,
               car_type     carType
        from t_car
    </select>

其中MapKey属于
import org.apache.ibatis.annotations.MapKey;
```
# 6. resultMap结果映射
```
查询结果的列名和java对象的属性名对应不上怎么办？

第⼀种⽅式：as 给列起别名
第⼆种⽅式：使⽤resultMap进⾏结果映射
第三种⽅式：是否开启驼峰命名⾃动映射（配置settings）
```
## 6.1 使用as起别名（XxxMapper.xml的sql语句中使用as起别名）
```
略
```
## 6.2 使⽤resultMap进⾏结果映射（mybatis-config.xml配置标签resultMap）
```
<!--
    resultMap:
		id:这个结果映射的标识，作为select标签的resultMap属性的值。
		type:结果集要映射的类。可以使用别名。
		property：POJO类的属性名
		column：数据库表的列名
-->
<resultMap id="carResultMap" type="${bean全限定类名}">
    <!--对象的唯一标识，如果有主键，建议使用id标签，官方解释是：为了提高mybatis的性能，建议写上，虽然说也可以用普通的result标签进行映射-->
    <id property="id" column="id"/>
    <result property="carNum" column="car_num"/>
    <!--当属性名和数据库列名一致时，可以省略，但是建议都写上-->
    <!--javaType用来指定属性类型。jdbcType用来指定列类型，一般可以省略-->
    <result property="brand" column="brand" javaType="string" jdbcType="VARCHAR"/>
    <result property="guidePrice" column="guide_price"/>
    <result property="produceTime" column="produce_time"/>
    <result property="carType" column="car_type"/>
</resultMap>

<!--resultMap属性的值必须和resultMap标签中id属性值一致。-->
<select id="selectAllByResultMap" resultMap="carResultMap">
    select * from t_car
</select>

注意：resultMap和resultType不要同时出现，resultMap优先级大于resultType，
```
## 6.3 驼峰命名自动映射（mybatis-config.xml中配置setttings标签）
```
是否开启驼峰命名⾃动映射
使⽤这种⽅式的前提是：属性名遵循Java的命名规范，数据库表的列名遵循SQL的命名规范。
Java命名规范：⾸字⺟⼩写，后⾯每个单词⾸字⺟⼤写，遵循驼峰命名⽅式。
SQL命名规范：全部⼩写，单词之间采⽤下划线分割。

在mybatis-config.xml⽂件中进⾏配置：

<settings>
    <setting name="mapUnderscoreToCamelCase" value="true"/>
</settings>

//underscore：下划线，强调

settings放在properties之后、typeAlias之前
```

# 7. 返回总记录条数

```
<select id="selectAll" resultType="long">
    select count(*) from t_car
</select>

@Test
public void testSelectTotal() {
    CarMapper carMapper = SqlSessionUtil.openSession().getMapper(CarMapper.class);
    Long total = carMapper.selectTotal();
    System.out.println(total);
}
```