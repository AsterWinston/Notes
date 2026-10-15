```
MyBatis的逆向工程
所谓的逆向工程是：根据数据库表逆向生成Java的pojo类，SqlMapper.xml文件，以及Mapper接口类等。
要完成这个工作，需要借助别人写好的逆向工程插件。

思考：使用这个插件的话，需要给这个插件配置哪些信息？

pojo类名、包名以及生成位置。
SqlMapper.xml文件名以及生成位置。
Mapper接口名以及生成位置。
连接数据库的信息。
指定哪些表参与逆向工程。
…
```
# 1. 逆向工程配置与生产

## 1.1 第一步：基础环境配置
```
打包方式jar
```
## 1.2 第二步：pom中添加逆向工程依赖
```
<!--配置-->
<!--定制构建过程-->
<build>
    <!--可配置多个插件-->
    <plugins>
        <!--其中的一个插件：mybatis逆向工程插件-->
        <plugin>
            <!--插件的GAV坐标-->
            <groupId>org.mybatis.generator</groupId>
            <artifactId>mybatis-generator-maven-plugin</artifactId>
            <version>1.4.1</version>
            <!--允许覆盖-->
            <configuration>
                <overwrite>true</overwrite>
            </configuration>
            <!--插件的依赖-->
            <dependencies>
                <!--mysql驱动依赖-->
                <dependency>
                    <groupId>mysql</groupId>
                    <artifactId>mysql-connector-java</artifactId>
                    <version>8.0.30</version>
                </dependency>
            </dependencies>
        </plugin>
    </plugins>
</build>
```

## 1.3 第三步：配置generatorConfig.xml
```
该文件名必须叫做：generatorConfig.xml
该文件必须放在类的根路径下。

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
        PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">

<generatorConfiguration>
    <!--
      targetRuntime有两个值：
          MyBatis3Simple：生成的是基础版，只有基本的增删改查。
          MyBatis3：生成的是增强版，除了基本的增删改查之外还有复杂的增删改查。
    -->
    <context id="DB2Tables" targetRuntime="MyBatis3">
        <!--防止生成重复代码-->
        <plugin type="org.mybatis.generator.plugins.UnmergeableXmlMappersPlugin"/>
        <commentGenerator>
            <!--是否去掉生成日期-->
            <property name="suppressDate" value="true"/>
            <!--是否去除注释-->
            <property name="suppressAllComments" value="true"/>
        </commentGenerator>
        <!--连接数据库信息-->
        <jdbcConnection driverClass="com.mysql.cj.jdbc.Driver"
                        connectionURL="jdbc:mysql://localhost:3306/ziv_mybatis"
                        userId="root"
                        password="root">
        </jdbcConnection>
        <!-- 生成pojo包名和位置 -->
        <javaModelGenerator targetPackage="com.ziv.mybatis.pojo" targetProject="src/main/java">
            <!--是否开启子包-->
            <property name="enableSubPackages" value="true"/>
            <!--是否去除字段名的前后空白-->
            <property name="trimStrings" value="true"/>
        </javaModelGenerator>
        <!-- 生成SQL映射文件的包名和位置 -->
        <sqlMapGenerator targetPackage="com.ziv.mybatis.mapper" targetProject="src/main/resources">
            <!--是否开启子包-->
            <property name="enableSubPackages" value="true"/>
        </sqlMapGenerator>
        <!-- 生成Mapper接口的包名和位置 -->
        <javaClientGenerator
                type="xmlMapper"
                targetPackage="com.ziv.mybatis.mapper"
                targetProject="src/main/java">
            <property name="enableSubPackages" value="true"/>
        </javaClientGenerator>
        <!-- 表名和对应的实体类名-->
        <table tableName="t_car" domainObjectName="Car"/>
    </context>
</generatorConfiguration>
```
## 1.4 第四步：运行插件
```
<context id="DB2Tables" targetRuntime="">配置不同，区别很大
```

# 2. QBC查询风格
```
QBC全称：query by criteria
```

# 3. 测试逆向工程生成的内容
```
这是对<context id="DB2Tables" targetRuntime="MyBatis3Simple">生成的内容的测试
public class BookMapperTest {
    @Test
    public void testSelectByPrimaryKey(){
        SqlSession sqlSession = SqlSessionUtil.openSqlSession();
        BookMapper bookMapper = sqlSession.getMapper(BookMapper.class);
        Book book = bookMapper.selectByPrimaryKey(1L);
        System.out.println(book);
        SqlSessionUtil.close(sqlSession);
    }
}

这是对<context id="DB2Tables" targetRuntime="MyBatis3">生成内容的测试
public class StudentMapperTest {
    @Test
    public void testSelect(){
        SqlSession sqlSession = SqlSessionUtil.openSqlSession();
        StudentExample studentExample = new StudentExample();
        StudentExample.Criteria criteria = studentExample.createCriteria();
        criteria.andAgeBetween(1, 100).andClazzIdBetween(1L, 100L);
        StudentMapper studentMapper = sqlSession.getMapper(StudentMapper.class);
        List<Student> list = studentMapper.selectByExample(studentExample);
        System.out.println(list.toString());
        SqlSessionUtil.close(sqlSession);
    }
}

```