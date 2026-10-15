# 1. 多对一
```
多种⽅式，常⻅的包括三种：

第⼀种⽅式：⼀条SQL语句，级联属性映射。
第⼆种⽅式：⼀条SQL语句，association。
第三种⽅式：两条SQL语句，分步查询。（这种⽅式常⽤：优点⼀是可复⽤。优点⼆是⽀持懒加载。）
```
## 1.1 级联属性映射（cascade）
```
pojo类Student中添加⼀个属性：Clazz clazz; 表示学⽣关联的班级对象。

编写sql语句：
<!--多对一映射的第一种方式：一条SQL语句，级联属性映射-->
<resultMap id="studentResultMap" type="Student">
    <id property="sid" column="sid"/>
    <result property="sname" column="sname"/>
    <!--property="clazz.cid" 中填的是Student对象中Clazz属性中的cid属性-->
    <!--column="cid" 中填的是数据库查询出来的列名-->
    <result property="clazz.cid" column="cid"/>
    <result property="clazz.cname" column="cname"/>
</resultMap>

<!--Student selectById(Integer sid);-->
<select id="selectById" resultMap="studentResultMap">
    select s.sid, s.sname, c.cid, c.cname
    from t_stu s left join t_clazz c on s.cid = c.cid
    where s.sid = #{sid}
</select>

测试程序：
@Test
public void testSelectById(){
    SqlSession sqlSession = SqlSessionUtil.openSession();
    StudentMapper mapper = sqlSession.getMapper(StudentMapper.class);
    Student student = mapper.selectById(1);
    System.out.println(student);
    sqlSession.close();
}

```
## 1.2 association（resultMap配置中使用association）
```
其他位置都不需要修改，只需要修改resultMap中的配置：association即可。
<resultMap id="studentResultMapAssociation" type="Student">
    <id property="sid" column="sid"/>
    <result property="sname" column="sname"/>
    <!--
        association:翻译为关联。一个Student对象关联一个Clazz对象
            property="clazz" 中填写要映射的pojo类的属性名
            javaType="Clazz" 中填写要映射的Java类型（自动起了别名，不需要写权限的类名）
    -->
    <association property="clazz" javaType="Clazz">
		<id property="cid" column="cid"/>
		<result property="cname" column="cname"/>
    </association>
</resultMap>

<!--Student selectByIdAssociation(Integer sid);-->
<select id="selectByIdAssociation" resultMap="studentResultMapAssociation">
    select s.sid, s.sname, c.cid, c.cname
    from t_stu s left join t_clazz c on s.cid = c.cid
    where s.sid = #{sid}
</select>

测试结果和第一种方法一样
```

## 1.3 分步查询
```
其他位置不需要修改，只需要修改以及添加以下三处：
第⼀处：association中select位置填写sqlId。sqlId=namespace+id。其中column属性作为这条⼦sql语句的条件。
<!--两天SQl语句完成多对一的分步查询-->
<!--第一步：根据学生的id查询学生的所有信息，这些信息中含有班级id（cid）-->
<resultMap id="selectResultMapByStep" type="Student">
    <id property="sid" column="sid"/>
    <result property="sname" column="sname"/>
    <!--property="" 中填写Student对象中的属性名clazz-->
    <!--select="" 中填写第二步需要执行的SQl语句的id-->
    <!--column="" 中填写的是sql语句中查询出来的列名-->
    <association property="clazz"
                 select="com.ziv.mybatis.mapper.ClazzMapper.selectByIdStepTwo"
                 column="cid"/>
</resultMap>

<!--Student selectByIdStepOne(Integer sid);-->
<select id="selectByIdStepOne" resultMap="selectResultMapByStep">
    select sid,sname,cid from t_stu where sid = #{sid}
</select>

第⼆处：在ClazzMapper接⼝中添加⽅法
package com.ziv.mybatis.mapper;

import com.ziv.mybatis.pojo.Clazz;

/**
 * @author ziv
 */
public interface ClazzMapper {

    /**
     * 根据id查询班级的信息
     * @param cid
     * @return
     */
    Clazz selectByIdStepTwo(Integer cid);
}

第三处：在ClazzMapper.xml⽂件中进⾏配置
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.ziv.mybatis.mapper.ClazzMapper">

    <!--Clazz selectByIdStepTwo(Integer cid);-->
    <!--分步查询第二步：根据cid获取班级信息-->
    <select id="selectByIdStepTwo" resultType="Clazz">
        select cid,cname from t_clazz where cid = #{cid}
    </select>
</mapper>

分步优点：

第⼀个优点：代码复⽤性增强。
第⼆个优点：⽀持延迟加载。【暂时访问不到的数据可以先不查询。提⾼程序的执⾏效率。】
```

# 2. 多对一延迟加载
```
要想⽀持延迟加载，⾮常简单，只需要在association标签中添加fetchType="lazy"即可。
修改StudentMapper.xml⽂件：
<resultMap id="selectResultMapByStep" type="Student">
    <id property="sid" column="sid"/>
    <result property="sname" column="sname"/>
    <!--fetchType="lazy" 开启懒加载-->
    <association property="clazz"
                 select="com.ziv.mybatis.mapper.ClazzMapper.selectByIdStepTwo"
                 column="cid" 
                 fetchType="lazy"/>
</resultMap>

测试结果，只执行了一条sql语句
@Test
public void testSelectByIdStepOne(){
    SqlSession sqlSession = SqlSessionUtil.openSession();
    StudentMapper mapper = sqlSession.getMapper(StudentMapper.class);
    Student student = mapper.selectByIdStepOne(1);
    //System.out.println(student);
    //只需要看学生的名字
    System.out.println(student.getSname());
    sqlSession.close();
}

在association标签中设置fetchType=“lazy”，只在当前association关联的sql语句中生效，如果需要添加全局设置，则需要在mybatis-config.xml中配置一下：
<settings>
    <setting name="lazyLoadingEnabled" value="true"/>
</settings>
如果开启了全局懒加载配置，但是在某个association中不希望使用懒加载，则在association中设置属性：fetchType=“eager”

启动 MyBatis 的懒加载（lazy loading）后返回的对象是一个代理对象（Proxy）。这时候，只有当使用getXxx()方法获取该对象属性里的级联对象时，才会执行相应的SQL，这就是反射机制的功劳了
```
# 3. 一对多
```
⼀对多的实现，通常是在⼀的⼀⽅中有List集合属性。
在Clazz类中添加List stus; 属性。

⼀对多的实现通常包括两种实现⽅式：

第⼀种⽅式：collection
第⼆种⽅式：分步查询
```

## 3.1 collection
```
<resultMap id="clazzResultMap" type="Clazz">
    <id property="cid" column="cid"/>
    <result property="cname" column="cname"/>
    <!--一对多。collection就是集合的意思-->
    <!--property属性指定clazz对象中的属性-->
    <!--ofType属性用来指定集合当中的元素类型-->
    <collection property="stus" ofType="Student">
        <id property="sid" column="sid"/>
        <result property="sname" column="sname"/>
    </collection>
</resultMap>

<!--Clazz selectByCollection(Integer cid);-->
<select id="selectByCollection" resultMap="clazzResultMap">
    select c.cid, c.cname, s.sid, s.sname
    from t_clazz c left join t_stu s on c.cid = s.cid
    where c.cid = #{cid}
</select>

```
## 3.2 分步查询
```
与多对一的分步查询类似
<resultMap id="clazzResultMapByStep" type="Clazz">
    <id property="cid" column="cid"/>
    <result property="cname" column="cname"/>
    <!--property属性对应对象想要赋值的属性-->
    <!--select属性对应第二步想要执行的sql语句id-->
    <!--column属性对象想要传给select属性的值，对应下面sql查出来的列名-->
    <collection property="stus"
                select="com.ziv.mybatis.mapper.StudentMapper.selectByCidStepTwo" 
                column="cid"/>
</resultMap>

<!--Clazz selectByStepOne(Integer cid);-->
<select id="selectByStepOne" resultMap="clazzResultMapByStep">
    select cid,cname from t_clazz where cid = #{cid}
</select>

```
# 4. 一对多延迟加载
```
与多对一相同
```