# 1. if标签
```
<!--
    if标签中test属性的必须的，值为boolean类型，如果为true则把标签内容拼接到sql语句中，反之不拼接
    test中参数的使用：
        1.当使用了@Param注解时，则使用@Param注解指定的参数名
        2.当没有使用@Param注解时，则使用param1 param2 param3 arg0 arg1 arg2
        3.当使用了pojo类，则使用pojo类的属性名
    在mybatis的动态sql中，不能使用&&，只能使用and
-->
<!--为了解决参数为空的问题，在where后面加一个恒成立的语句（1=1），这样就需要第一个if前加一个and，来保证sql语句的正确-->
<select id="selectByMultiCondition" resultType="car">
    select * from t_car where 1=1
    <if test="brand != null and brand != ''">
        and brand like "%"#{brand}"%"
    </if>
    <if test="guidePrice != null and guidePrice != ''">
        and guide_price > #{guidePrice}
    </if>
    <if test="carType != null and carType != ''">
        and car_type =#{carType}
    </if>
</select>

test使用的参数必须要可以从传入的参数中获取，哪怕是null或者''（也就是至少要声明了），否则抛出异常org.apache.ibatis.binding.BindingException，就如brand, guidePrice, catType这三个，必须可以从接口传入的参数中获取到
```
# 2. where标签
```
where标签的作用：让where子句更加动态智能。

所有条件都为空时，where标签保证不会生成where子句。
自动去除某些条件前面多余的and或or。

<select id="selectByMultiConditionWithWhere" resultType="car">
    select * from t_car
    <where>
        <if test="brand != null and brand != ''">
            and brand like "%"#{brand}"%"
        </if>
        <if test="guidePrice != null and guidePrice != ''">
            and guide_price > #{guidePrice}
        </if>
        <if test="carType != null and carType != ''">
            and car_type =#{carType}
        </if>
    </where>
</select>

```
# 3. trim标签
```
trim标签的属性：

prefix：在trim标签中的语句前添加内容
suffix：在trim标签中的语句后添加内容
prefixOverrides：前缀覆盖掉（去掉）
suffixOverrides：后缀覆盖掉（去掉）

<!--
    prefix="where" 是在trim标签所有内容的前面添加 where
    suffixOverrides="and|or" 是把trim标签中内容的后缀and或or去掉
-->
<select id="selectByMultiConditionWithTrim" resultType="car">
    select * from t_car
    <trim prefix="where" suffixOverrides="and|or">
        <if test="brand != null and brand != ''">
            brand like "%"#{brand}"%" and
        </if>
        <if test="guidePrice != null and guidePrice != ''">
            guide_price > #{guidePrice} and
        </if>
        <if test="carType != null and carType != ''">
            car_type =#{carType}
        </if>
    </trim>
</select>

```

# 4. set标签
```
<update id="updateBySet">
update t_car
<set>
    <if test="carNum !=null and carNum != ''">car_num=#{carNum},</if>
    <if test="brand !=null and brand != ''">brand=#{brand},</if>
    <if test="guidePrice !=null and guidePrice != ''">guide_price=#{guidePrice},</if>
    <if test="produceTime !=null and produceTime != ''">produce_time=#{produceTime},</if>
    <if test="carType !=null and carType != ''">car_type=#{carType},</if>
</set>
where id = #{id};
</update>
```
# 5. choose when otherwise
```
<choose>
    <when></when>
    <when></when>
    <when></when>
    <otherwise></otherwise>
</choose>

就等同于

if () {
} else if () {
} else if () {
} else if () {
} else {
}

有且仅有一个分支会被选择！！！！

<select id="selectByChoose" resultType="Car">
    select * from t_car
    <where>
        <choose>
            <when test="brand != null and brand != ''">
                brand like "%"#{brand}"%"
            </when>
            <when test="guidePrice != null and guidePrice != ''">
                guide_price > #{guidePrice}
            </when>
            <when test="carType != null and carType != ''">
                car_type = #{carType}
            </when>
        </choose>
    </where>
</select>

```
# 6. foreach标签
```
循环数组或集合，动态生成sql，比如这样的SQL：

delete from t_car where id in(1,2,3);
delete from t_car where id = 1 or id = 2 or id = 3;

insert into t_car values
    (null,'1001','凯美瑞',35.0,'2010-10-11','燃油车'),
    (null,'1002','比亚迪唐',31.0,'2020-11-11','新能源'),
    (null,'1003','比亚迪宋',32.0,'2020-10-11','新能源')
```
## 6.1 批量删除
```
定义接口

int deleteByIds(@Param("ids")Long[] ids);

sql语句

<!--
    foreach标签的属性：
        collection:指定数组或者集合，默认值为array或arg0.也可以使用@Param注解自行指定
        item:代表数组或集合中的元素
        separator:循环之间的分隔符
        open:在循坏开始之前拼接的内容
        close:在循坏结束之后拼接的内容
-->

<delete id="deleteByIds">
    delete from t_car where id in
        <foreach collection="ids" item="id" separator="," open="(" close=")">
            #{id}
        </foreach>
</delete>
```
## 6.2 批量插入
```
定义接口

int insertBatch(@Param("cars") List<Car> cars);

sql语句

<insert id="insertBatch">
    insert into t_car values
    <foreach collection="cars" item="car" separator=",">
        (null,#{car.carNum},#{car.brand},#{car.guidePrice},#{car.produceTime},#{car.carType})
    </foreach>
</insert>
```

# 7. sql标签与include标签

```
sql标签用来声明sql片段
include标签用来将声明的sql片段包含到某个sql语句当中
作用：代码复用。易维护。

<sql id="carColumnName">
    id,
    car_num      as carNum,
    brand,
    guide_price  as guidePrice,
    produce_time as produceTime,
    car_type     as carType
</sql>

<select id="selectCarById" resultType="com.ziv.mybatis.pojo.Car">
    select 
    <include refid="carColumnName"/>
    from t_car
    where id = #{id}
</select>

```