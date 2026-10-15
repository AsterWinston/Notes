# 1. javassist的使用

## 1.1 介绍
```
Javassist（Java Programming Assistant）是一个用于处理 Java 字节码的开源库。它允许开发者在运行时（Run-time）动态地创建、检查和修改 Java 类，而无需了解底层的字节码规范。
```
## 1.2 定义
```
定义: Javassist 是一个高性能的 Java 字节码工程库（Bytecode Engineering Library）。

目标: 简化字节码操作的复杂性。与 ASM 或 ByteBuddy 等库不同，Javassist 允许开发者使用高层级的 Java 源代码形式来定义或修改类结构，而不是直接操作底层的 JVM 指令。
```
## 1.3 核心能力
```
动态创建: 在内存中从零开始定义新的类。

运行时修改: 在 JVM 加载类之前或之后，修改现有类的方法体、字段和构造器。

方法织入: 在现有方法的开始、结束或环绕（around）添加代码片段（AOP 基础）。
```
## 1.4 关键概念与类
```
ClassPool	类的容器，负责查找、读取和存储 CtClass 实例。它类似于 JVM 的 ClassLoader。
CtClass	"Compile-time Class" 的缩写。它是 Javassist 中类的抽象表示，提供了修改类结构的方法。
CtMethod	"Compile-time Method"。用于表示和操作类中的方法。
CtField	"Compile-time Field"。用于表示和操作类中的字段。
```

## 1.5 常见用途 (Use Cases)
```
AOP (面向切面编程): Spring 框架等在使用 Javassist 或类似的字节码技术，在运行时为 Service 等对象生成代理类，以实现事务、日志记录等横切关注点。

Mocking 框架: 某些测试框架（如 PowerMock）使用 Javassist 来修改类的私有方法或 final 方法，以实现更强大的 Mocking 能力。

ORM/持久化框架: 有些 ORM 框架利用字节码技术动态地为实体类添加延迟加载（Lazy Loading）功能。

热部署/Hotswap: 允许在不重启应用的情况下替换和更新已加载的类定义。
```
# 2. 使用javassist生成DaoImpl类

```
package com.ziv.bank.utils;

import org.apache.ibatis.javassist.CannotCompileException;
import org.apache.ibatis.javassist.ClassPool;
import org.apache.ibatis.javassist.CtClass;
import org.apache.ibatis.javassist.CtMethod;
import org.apache.ibatis.mapping.SqlCommandType;
import org.apache.ibatis.session.SqlSession;

import java.lang.reflect.Method;
import java.util.Arrays;

/**
 * 工具类：可以动态的生成DAO的实现类。（或者说可以动态生成DAO的代理类）
 */
public class GenerateDaoProxy {

    public static Object generate(SqlSession sqlSession, Class daoInterface) {
        //类池
        ClassPool pool = ClassPool.getDefault();
        //制造类(com.ziv.bank.dao.AccountDao --> com.ziv.bank.dao.AccountDaoProxy)
        CtClass ctClass = pool.makeClass(daoInterface.getName() + "Proxy");
        //制造接口
        CtClass ctInterface = pool.makeInterface(daoInterface.getName());
        //实现接口
        ctClass.addInterface(ctInterface);
        //实现接口中的所有方法
        Method[] methods = daoInterface.getDeclaredMethods();
        Arrays.stream(methods).forEach(method -> {
            //method是接口中的抽象方法
            //实现method这个抽象方法
            // Account selectByActno(String actno);
            // public Account selectByActno(String actno){代码;}
            StringBuilder methodCode = new StringBuilder();
            methodCode.append("public ");
            methodCode.append(method.getReturnType().getName());
            methodCode.append(" ");
            methodCode.append(method.getName());
            methodCode.append("(");
            Class<?>[] parameterTypes = method.getParameterTypes();
            for (int i = 0; i < parameterTypes.length; i++) {
                Class<?> parameterType = parameterTypes[i];
                methodCode.append(parameterType.getName());
                methodCode.append(" ");
                methodCode.append("arg" + i);
                if (i != parameterTypes.length - 1) {
                    methodCode.append(",");
                }
            }
            methodCode.append(")");
            methodCode.append("{");
            //
            methodCode.append("org.apache.ibatis.session.SqlSession sqlSession = com.ziv.bank.utils.SqlSessionUtil.openSession();");
            String sqlId = daoInterface.getName() + "." + method.getName();
            SqlCommandType commandType = sqlSession.getConfiguration().getMappedStatement(sqlId).getSqlCommandType();
            if (commandType == SqlCommandType.DELETE) {

            }
            if (commandType == SqlCommandType.INSERT) {

            }
            if (commandType == SqlCommandType.UPDATE) {
                methodCode.append("return sqlSession.update(\"" + sqlId + "\", arg0);");
            }
            if (commandType == SqlCommandType.SELECT) {
                methodCode.append("return (" + method.getReturnType().getName() + ")sqlSession.selectOne(\"" + sqlId + "\", arg0);");
            }
            methodCode.append("}");
            try {
                CtMethod ctMethod = CtMethod.make(String.valueOf(methodCode), ctClass);
                ctClass.addMethod(ctMethod);
            } catch (CannotCompileException e) {
                e.printStackTrace();
            }
        });
        //创建对象
        Object obj = null;
        try {
            Class<?> clazz = ctClass.toClass();
            obj = clazz.newInstance();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return obj;
    }
}

# 3.MyBatis 内部对 Javassist 的使用

```
MyBatis 内部大量使用了 Javassist，主要用于以下场景：
  1. 延迟加载代理对象生成
  2. 对象工厂（创建结果对象）
  3. Reflector（反射工具优化）
  4. SqlRunner（SQL 执行工具）
```

## 3.1 延迟加载代理对象生成

```java
/**
 * MyBatis 使用 Javassist 生成延迟加载的代理对象
 *
 * 源码位置：org.apache.ibatis.executor.loader.javassist.JavassistProxyFactory
 *
 * 原理：
 *   当 fetchType = LAZY 时，MyBatis 不会立即查询关联对象，
 *   而是创建一个代理对象。当第一次调用 getter 方法时，
 *   代理对象拦截调用，触发真正的 SQL 查询。
 *
 * 例如：
 *   Student { sid=1, sname="张三", clazz=null(代理) }
 *
 *   调用 student.getClazz() 时：
 *     → 代理拦截 getter
 *     → 执行 "select cid,cname from t_clazz where cid = ?"
 *     → 将查询结果设置到 clazz 属性
 *     → 返回 Clazz 对象
 *   之后再次调用 getClazz() 就直接返回结果，不再查询
 */
public class JavassistProxyFactory implements ProxyFactory {

    // 创建延迟加载代理
    @Override
    public Object createProxy(Object target, ResultLoaderMap lazyLoader,
            Configuration configuration, ObjectFactory objectFactory,
            List<Class<?>> constructorArgTypes,
            List<Object> constructorArgs) {

        // 使用 Javassist 创建代理类
        return EnhancedResultObjectProxyImpl
            .createProxy(target, lazyLoader, configuration, objectFactory,
                        constructorArgTypes, constructorArgs);
    }
}
```

## 3.2 代理对象的工作原理

```java
/**
 * 延迟加载代理对象的内部结构（简化版）
 *
 * 代理对象拦截了所有 getter 方法调用，
 * 在返回属性值前检查是否需要触发延迟加载。
 */

// 代理类大致等价于：
public class StudentProxy extends Student {
    private ResultLoaderMap lazyLoader;  // 延迟加载的配置
    private boolean clazzLoaded = false; // clazz 属性是否已加载

    @Override
    public Clazz getClazz() {
        if (!clazzLoaded) {
            // ★ 第一次调用 → 触发 SQL 查询 ★
            String sqlId = lazyLoader.getLoader("clazz").getMappedStatement();
            Clazz clazz = sqlSession.selectOne(sqlId, this.getCid());
            this.setClazz(clazz);
            clazzLoaded = true;
        }
        return super.getClazz();  // 直接返回已加载的值
    }

    @Override
    public String getSname() {
        return super.getSname();  // sname 直接返回，不触发延迟加载
    }
}
```

## 3.3 CGLIB vs Javassist

```java
/**
 * MyBatis 支持两种代理工厂，在 settings 中配置：
 *   <setting name="proxyFactory" value="JAVASSIST"/>  ← 默认，推荐
 *   <setting name="proxyFactory" value="CGLIB"/>
 *
 * 对比：
 *   Javassist：
 *     - 使用源码级别的 API，可读性好
 *     - 性能较好（尤其是代理创建速度）
 *     - MyBatis 内置，无需额外依赖
 *
 *   CGLIB：
 *     - 基于 ASM，直接操作字节码
 *     - 代理执行速度快（但创建慢）
 *     - 需要额外引入 cglib 依赖
 *     - Spring 中广泛使用
 */

// Configuration 中获取代理工厂的方式：
public ProxyFactory getProxyFactory() {
    if (proxyFactory == null) {
        setProxyFactory(new JavassistProxyFactory());  // 默认
    }
    return proxyFactory;
}
```

## 3.4 手写 GenerateDaoProxy vs MyBatis MapperProxy

```
我们手写的 GenerateDaoProxy 和 MyBatis 的 MapperProxy 解决的问题是相同的：
自动生成 DAO 接口的实现类，避免手写冗余代码。

实现差异：
  GenerateDaoProxy:
    - 使用 Javassist 在运行时生成 .class 文件
    - 需要手动编码实现每个方法的逻辑
    - 重量级（生成完整的类）

  MyBatis MapperProxy:
    - 使用 JDK 动态代理（java.lang.reflect.Proxy）
    - 不需要生成 .class 文件，在内存中生成代理对象
    - 轻量级（只需要一个 InvocationHandler）
    - 通用性强（一套代码处理所有 Mapper 接口）

MyBatis 选择 JDK 动态代理而不是 Javassist 的原因：
  1. JDK 动态代理是 JDK 内置的，零依赖
  2. Mapper 接口没有复杂的继承关系，JDK 代理足够
  3. 性能更好（不需要解析和编译 Java 源码）
  4. MyBatis 只把 Javassist 用于需要修改对象字节码的场景（延迟加载代理）

  延迟加载为什么用 Javassist 而不用 JDK 代理？
  → JDK 代理只能代理接口，而延迟加载需要代理具体的 POJO 类
  → Javassist 可以创建类的子类（和 CGLIB 一样），从而拦截 getter 方法
```

```
