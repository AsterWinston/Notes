# 1. 准备工作
```
创建module（Maven的普通Java模块）：mybatis-002-crud
pom.xml
打包方式jar
依赖：
mybatis依赖
mysql驱动依赖
junit依赖
logback依赖
mybatis-config.xml放在类的根路径下
CarMapper.xml放在类的根路径下
logback.xml放在类的根路径下
提供com.powernode.mybatis.utils.SqlSessionUtil工具类
创建测试用例：com.powernode.mybatis.CarMapperTest
```
# 2. insert
```
分析以下SQL映射文件中SQL语句存在的问题
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<!--namespace先随便写-->
<mapper namespace="car">
    <insert id="insertCar">
        insert into t_car(car_num,brand,guide_price,produce_time,car_type) values('103', '奔驰E300L', 50.3, '2022-01-01', '燃油车')
    </insert>
</mapper>

存在的问题是：SQL语句中的值不应该写死，值应该是用户提供的。之前的JDBC代码是这样写的：
// JDBC中使用 ? 作为占位符。那么MyBatis中会使用什么作为占位符呢？
String sql = "insert into t_car(car_num,brand,guide_price,produce_time,car_type) values(?,?,?,?,?)";
// ......
// 给 ? 传值。那么MyBatis中应该怎么传值呢？
ps.setString(1,"103");
ps.setString(2,"奔驰E300L");
ps.setDouble(3,50.3);
ps.setString(4,"2022-01-01");
ps.setString(5,"燃油车");

在MyBatis中可以这样做：
在Java程序中，将数据放到Map集合中
在sql语句中使用 #{map集合的key} 来完成传值，#{} 等同于JDBC中的 ? ，#{}就是占位符
Java程序这样写：

package com.powernode.mybatis;

import com.powernode.mybatis.utils.SqlSessionUtil;
import org.apache.ibatis.session.SqlSession;
import org.junit.Test;

import java.util.HashMap;
import java.util.Map;

/**
 * 测试MyBatis的CRUD
 * @author 老杜
 * @version 1.0
 * @since 1.0
 */
public class CarMapperTest {
    @Test
    public void testInsertCar(){
        // 准备数据
        Map<String, Object> map = new HashMap<>();
        map.put("k1", "103");
        map.put("k2", "奔驰E300L");
        map.put("k3", 50.3);
        map.put("k4", "2020-10-01");
        map.put("k5", "燃油车");
        // 获取SqlSession对象
        SqlSession sqlSession = SqlSessionUtil.openSession();
        // 执行SQL语句（使用map集合给sql语句传递数据）
        int count = sqlSession.insert("insertCar", map);
        System.out.println("插入了几条记录：" + count);
    }
}

SQL语句这样写：

<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<!--namespace先随便写-->
<mapper namespace="car">
    <insert id="insertCar">
        insert into t_car(car_num,brand,guide_price,produce_time,car_type) values(#{k1},#{k2},#{k3},#{k4},#{k5})
    </insert>
</mapper>

#{} 的里面必须填写map集合的key，不能随便写。

如果#{}里写的是map集合中不存在的key会有什么问题？

<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="car">
    <insert id="insertCar">
        insert into t_car(car_num,brand,guide_price,produce_time,car_type) values(#{kk},#{k2},#{k3},#{k4},#{k5})
    </insert>
</mapper>

通过测试，看到程序并没有报错。正常执行。不过 #{kk} 的写法导致无法获取到map集合中的数据，最终导致数据库表car_num插入了NULL。
在以上sql语句中，可以看到#{k1} #{k2} #{k3} #{k4} #{k5}的可读性太差，为了增强可读性，我们可以将Java程序做如下修改：

Map<String, Object> map = new HashMap<>();
// 让key的可读性增强
map.put("carNum", "103");
map.put("brand", "奔驰E300L");
map.put("guidePrice", 50.3);
map.put("produceTime", "2020-10-01");
map.put("carType", "燃油车");

<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="car">
    <insert id="insertCar">
        insert into t_car(car_num,brand,guide_price,produce_time,car_type) values(#{carNum},#{brand},#{guidePrice},#{produceTime},#{carType})
    </insert>
</mapper>


使用Map集合可以传参，那使用pojo（简单普通的java对象）可以完成传参吗？测试一下：

第一步：定义一个pojo类Car，提供相关属性。
package com.powernode.mybatis.pojo;

/**
 * POJOs，简单普通的Java对象。封装数据用的。
 * @author 老杜
 * @version 1.0
 * @since 1.0
 */
public class Car {
    private Long id;
    private String carNum;
    private String brand;
    private Double guidePrice;
    private String produceTime;
    private String carType;

    @Override
    public String toString() {
        return "Car{" +
                "id=" + id +
                ", carNum='" + carNum + '\'' +
                ", brand='" + brand + '\'' +
                ", guidePrice=" + guidePrice +
                ", produceTime='" + produceTime + '\'' +
                ", carType='" + carType + '\'' +
                '}';
    }

    public Car() {
    }

    public Car(Long id, String carNum, String brand, Double guidePrice, String produceTime, String carType) {
        this.id = id;
        this.carNum = carNum;
        this.brand = brand;
        this.guidePrice = guidePrice;
        this.produceTime = produceTime;
        this.carType = carType;
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getCarNum() {
        return carNum;
    }

    public void setCarNum(String carNum) {
        this.carNum = carNum;
    }

    public String getBrand() {
        return brand;
    }

    public void setBrand(String brand) {
        this.brand = brand;
    }

    public Double getGuidePrice() {
        return guidePrice;
    }

    public void setGuidePrice(Double guidePrice) {
        this.guidePrice = guidePrice;
    }

    public String getProduceTime() {
        return produceTime;
    }

    public void setProduceTime(String produceTime) {
        this.produceTime = produceTime;
    }

    public String getCarType() {
        return carType;
    }

    public void setCarType(String carType) {
        this.carType = carType;
    }
}

第二步：Java程序
@Test
public void testInsertCarByPOJO(){
    // 创建POJO，封装数据
    Car car = new Car();
    car.setCarNum("103");
    car.setBrand("奔驰C200");
    car.setGuidePrice(33.23);
    car.setProduceTime("2020-10-11");
    car.setCarType("燃油车");
    // 获取SqlSession对象
    SqlSession sqlSession = SqlSessionUtil.openSession();
    // 执行SQL，传数据
    int count = sqlSession.insert("insertCarByPOJO", car);
    System.out.println("插入了几条记录" + count);
}

第三步：SQL语句
<insert id="insertCarByPOJO">
  <!--#{} 里写的是POJO的属性名-->
  insert into t_car(car_num,brand,guide_price,produce_time,car_type) values(#{carNum},#{brand},#{guidePrice},#{produceTime},#{carType})
</insert>
运行程序，查看数据库表
{} 里写的是POJO的属性名，如果写成其他的会有问题吗？
<insert id="insertCarByPOJO">
  insert into t_car(car_num,brand,guide_price,produce_time,car_type) values(#{a},#{brand},#{guidePrice},#{produceTime},#{carType})
</insert>
运行程序，出现了异常
错误信息中描述：在Car类中没有找到a属性的getter方法。
修改POJO类Car的代码，只将getCarNum()方法名修改为getA()，其他代码不变
再运行程序，查看数据库表中数据
经过测试得出结论：
如果采用map集合传参，#{} 里写的是map集合的key，如果key不存在不会报错，数据库表中会插入NULL。
如果采用POJO传参，#{} 里写的是get方法的方法名去掉get之后将剩下的单词首字母变小写（例如：getAge对应的是#{age}，getUserName对应的是#{userName}），如果这样的get方法不存在会报错。

注意：其实传参数的时候有一个属性parameterType，这个属性用来指定传参的数据类型，不过这个属性是可以省略的
<insert id="insertCar" parameterType="java.util.Map">
  insert into t_car(car_num,brand,guide_price,produce_time,car_type) values(#{carNum},#{brand},#{guidePrice},#{produceTime},#{carType})
</insert>

<insert id="insertCarByPOJO" parameterType="com.powernode.mybatis.pojo.Car">
  insert into t_car(car_num,brand,guide_price,produce_time,car_type) values(#{carNum},#{brand},#{guidePrice},#{produceTime},#{carType})
</insert>

实则就算parameterType="java.util.Map"，也可以可以使用某某bean类对象传入而成功执行，反过来parameterType="某某bean类"，也可以传入Map对象而成功执行，我真没招了，报错不好么
```

# 3. delete
```
需求：根据car_num进行删除。
SQL语句这样写：
<delete id="deleteByCarNum">
  delete from t_car where car_num = #{SuiBianXie}
</delete>

Java程序这样写：
@Test
public void testDeleteByCarNum(){
    // 获取SqlSession对象
    SqlSession sqlSession = SqlSessionUtil.openSession();
    // 执行SQL语句
    int count = sqlSession.delete("deleteByCarNum", "102");
    System.out.println("删除了几条记录：" + count);
}

注意：当占位符只有一个的时候，${} 里面的内容可以随便写。无需与参数名字对应
```

# 4. update
```
需求：修改id=34的Car信息，car_num为102，brand为比亚迪汉，guide_price为30.23，produce_time为2018-09-10，car_type为电车
SQL语句如下：
<update id="updateCarByPOJO">
  update t_car set 
    car_num = #{carNum}, brand = #{brand}, 
    guide_price = #{guidePrice}, produce_time = #{produceTime}, 
    car_type = #{carType} 
  where id = #{id}
</update>

Java代码如下：
    @Test
    public void testUpdateCarByPOJO(){
        // 准备数据
        Car car = new Car();
        car.setId(34L);
        car.setCarNum("102");
        car.setBrand("比亚迪汉");
        car.setGuidePrice(30.23);
        car.setProduceTime("2018-09-10");
        car.setCarType("电车");
        // 获取SqlSession对象
        SqlSession sqlSession = SqlSessionUtil.openSession();
        // 执行SQL语句
        int count = sqlSession.update("updateCarByPOJO", car);
        System.out.println("更新了几条记录：" + count);
    }
    
当然了，如果使用map传数据也是可以的。
```

# 5. select

## 5.1 select（查一个，返回结果是一个）
```
<select id="selectById" resultType="com.ziv.mybatis.pojo.Car">
    select id,car_num as carNum, brand, guide_price as guidePrice,
    produce_time as produceTime, car_type as carType
    from t_car 
    where id = #{id}
</select>
@Test
public void testSelectById(){
    SqlSession sqlSession = SqlSessionUtil.openSession();
    Car car = sqlSession.selectOne("selectById", 12);
    System.out.println("car = " + car);
    sqlSession.commit();
    sqlSession.close();
}

mybatis执行了select语句之后，会返回一个ResultSet对象，然后从中取出数据，再封装成Java对象。所以需要通过select标签中的 resultType属性指明需要封装成什么对象。

注意：select语句中查询的列别名要和pojo类中的属性名一致，本质就是和setXXX方法中的XXX首字母小写一致就好
```
## 5.2 select（查所有）
```
<select id="selectAll" resultType="com.ziv.mybatis.pojo.Car">
    select id,car_num as carNum, brand, guide_price as guidePrice,
    produce_time as produceTime, car_type as carType
    from t_car
</select>

@Test
public void testSelectAll(){
    SqlSession sqlSession = SqlSessionUtil.openSession();
    List<Object> carList = sqlSession.selectList("selectAll");
    System.out.println("carList = " + carList);
    sqlSession.commit();
    sqlSession.close();
}

```
# 6. 关于SQL Mapper 的namespace
```
在SQL Mapper配置文件中标签的namespace属性可以翻译为命名空间，这个命名空间主要是为了防止sqlId冲突的。

当有两个不同的Mapper.xml文件中的sql语句id相同的时候，就需要使用命名空间加id的形式。

@Test
public void testNamespace(){
    // 获取SqlSession对象
    SqlSession sqlSession = SqlSessionUtil.openSession();
    // 执行SQL语句，使用namespace.id的形式
    List<Object> cars = sqlSession.selectList("car.selectCarAll");
    // 输出结果
    cars.forEach(car -> System.out.println(car));
}

```
# 7. CRUD 操作的源码调用链

所有 CRUD 操作（insert/delete/update/selectOne/selectList）
在 DefaultSqlSession 中都最终委托给 Executor 执行。

## 7.1 insert/update/delete 调用链

```java
/**
 * DefaultSqlSession 中 insert/update/delete 都调用 update()
 *
 * 核心调用链：
 *   sqlSession.insert/update/delete("sqlId", param)
 *     → DefaultSqlSession.update()
 *       → executor.update(ms, wrapCollection(parameter))
 *         → BaseExecutor.update()
 *           → clearLocalCache()          ← ★ 清空一级缓存
 *           → doUpdate(ms, parameter)    ← 子类实现（模板方法）
 *             → SimpleExecutor.doUpdate()
 *               → Configuration.newStatementHandler()
 *               → prepareStatement(handler)
 *                 → getConnection()
 *                 → handler.prepare()    ← 创建 PreparedStatement
 *                 → handler.parameterize() ← 设置参数
 *               → handler.update(stmt)   ← 执行 executeUpdate()
 *               → closeStatement(stmt)
 *             → 返回影响行数
 */
```

## 7.2 selectOne/selectList 调用链

```java
/**
 * selectOne() 内部调用 selectList()，然后校验返回结果数量
 *
 * 核心调用链：
 *   sqlSession.selectOne("sqlId", param)
 *     → selectList("sqlId", param)
 *       → executor.query(ms, parameter, RowBounds.DEFAULT, ...)
 *         → CachingExecutor.query()          ← 二级缓存（如果开启）
 *           → delegate.query()               ← 委托给 BaseExecutor
 *             → BaseExecutor.query()
 *               → 创建 CacheKey
 *               → 查询一级缓存 → 命中则返回
 *               → queryFromDatabase()          ← 一级缓存未命中
 *                 → doQuery()                ← 模板方法，子类实现
 *                   → SimpleExecutor.doQuery()
 *                     → 创建 StatementHandler
 *                     → prepareStatement()
 *                     → handler.query(stmt, resultHandler)
 *                       → ps.execute()
 *                       → resultSetHandler.handleResultSets()
 *                 → 结果放入一级缓存
 *     → 返回结果列表
 */
```

## 7.3 参数封装 wrapCollection()

```java
/**
 * DefaultSqlSession.wrapCollection() — 参数预处理
 *
 * 如果参数是 Collection 或 Array，则包装为 Map
 * 方便在 SQL 中通过 collection / list / array 名称引用
 */
private Object wrapCollection(final Object object) {
    if (object instanceof Collection) {
        StrictMap<Object> map = new StrictMap<>();
        map.put("collection", object);
        if (object instanceof List) {
            map.put("list", object);
        }
        return map;
    } else if (object != null && object.getClass().isArray()) {
        StrictMap<Object> map = new StrictMap<>();
        map.put("array", object);
        return map;
    }
    return object;
}

// 这就是为什么在 foreach 中 collection 属性可以写 list 或 array
//   <foreach collection="list" item="item">
//   <foreach collection="ids" item="id">  ← 如果用 @Param("ids") 指定了名称
```
