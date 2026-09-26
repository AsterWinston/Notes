# 1. 注解 Annotation 基础概念

```
注解Annotation是一种引用数据类型，编译之后也是生成Xxx.class
```

# 2. 定义注解的语法

```
[修饰符列表] @interface 注解类型名{}
```

# 3. 注解可用位置

```
类、属性
```

# 4. 内置 Override 注解

```
编译检查注解
```

# 5. 元注解

## 5.1 Target

```
用来标注被标注的注解可以出现的位置
```

## 5.2 Retention

```
用来标注被标注的注解最终保存在哪里
5.2.1 @Retention(RetentionPolicy.SOURCE): 表示该注解只被保存在Java源文件中
5.2.2 @Retention(RetentionPolicy.CLASS): 表示该注解被保存在Class文件中
5.2.3 @Retention(RetentionPolicy.RUNTIME): 表示该注解被保存在Class文件中，并且可以被反射机制读取
```

# 6. 内置 Deprecated 注解

```
表示这个类过时了
```

# 7. 注解的属性

```
7.1 有属性值不能为空
7.2 有默认值可以不指定
7.3 属性名为value且只有一个的时候value=可以省略不写，只写值就好
7.4 数组类型的属性用大括号赋值
```