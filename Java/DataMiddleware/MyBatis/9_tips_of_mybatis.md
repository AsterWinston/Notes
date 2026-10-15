# 1. #{}和${}
```
#{}：先编译sql语句，再给占位符传值，底层是PreparedStatement实现。可以防止sql注入，比较常用。
${}：先进行sql语句拼接，然后再编译sql语句，底层是Statement实现。存在sql注入现象。只有在需要进行sql语句关键字拼接的情况下才会用到。

使用${}的时机：

当需要进行sql语句关键字拼接的时候。必须使用${}

例如：通过向sql语句中注入asc或desc关键字，来完成数据的升序或降序排列。

拼接表名

例如：在前端传参数决定查询哪个表，t_log20221201,t_log20221202等等。

模糊查询

有三种解决方法

name like '%${name}%'
name like concat('%',#{name},'%')
name like "%"#{name}"%"

```

# 2. typeAliases
```
在mybatis-config.xml中指定（settings之后，environments之前）
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
    <!--使用package将对应包下的所有类全部自动起别名，默认就是类的简名-->
    <package name="com.ziv.mybatis.pojo"/>
</typeAliases>

在CarMapper.xml标签resultType属性中使用
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

# 3. mappers
```
<!--
    mapper的三个属性：
        resource：从类路径中加载
        url：从指定的全限定资源路径中加载
        class：使用映射器接口实现类的完全限定类名，这种方式，要将XxxMapper.xml这个文件放到与接口同目录下，接口名字和XxxMapper.xml中的XxxMapper一致，也就是说，当指定<mapper class="me.aster.mybatis.mapper.AccountMapper">后，mybatis会到classPath:me/aster/mybatis/mapper这个目录下找AccountMapper.xml文件
        package：将包内的映射器接口实现全部注册为映射器
-->
<mapper resource=""/>
<mapper class=""/>
<mapper url=""/>
<!--
	package：将包内的映射器接口实现全部注册为映射器
	使用此方法需要将SqlMapper.xml文件和同名接口放在同一个目录下，比<mapper class=""/>更加高效了
-->
<package name="com.ziv.mybatis.mapper"/>

```

# 4. idea配置文件模板
```
File->Settings->Editor->File and Code Templates

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


<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="">

</mapper>

```

# 5. 插入数据时获取自动生成的主键
```
<!--
    useGeneratedKeys="true" 使用自动生成的主键值
    keyProperty="id" 指定主键值赋值给对象的哪个属性。这个表示将自动生成的主键值赋值给Car的id属性
-->
<insert id="insertCarUseGeneratedKey" useGeneratedKeys="true" keyProperty="id">
    insert into t_car(id, car_num, brand, guide_price, produce_time, car_type)
    values (null, #{carNum}, #{brand}, #{guidePrice}, #{produceTime}, #{carType})
</insert>

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

输出car之后发现id被赋上了自动生成的值
```