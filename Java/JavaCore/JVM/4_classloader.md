
# 1. 类加载器分类

JVM 将类加载器划分为两种：
- **启动类加载器**（Bootstrap ClassLoader），C++ 实现，JVM 的一部分。
- **其他类加载器**（Java 实现，继承 `java.lang.ClassLoader`）。

三层类加载器（Java 8 及之前）：
```
Bootstrap ClassLoader       ← 加载 <JAVA_HOME>/lib 下的 rt.jar 等
        ↑
Extension ClassLoader       ← 加载 <JAVA_HOME>/lib/ext 目录
        ↑
Application ClassLoader     ← 加载用户 classpath 下的类
```

Java 9+ 变化（模块化）：
```
Bootstrap ClassLoader       ← 加载 java.base 等核心模块
        ↑
Platform ClassLoader        ← 取代 Extension，加载 java.se 等平台模块
        ↑
Application ClassLoader     ← 加载 classpath / module-path
```

# 2. 核心源码

```java
// Application ClassLoader 获取
ClassLoader appClassLoader = ClassLoader.getSystemClassLoader();

// Extension / Platform ClassLoader
ClassLoader extClassLoader = appClassLoader.getParent();

// Bootstrap ClassLoader (C++ 实现，返回 null)
ClassLoader bootstrapClassLoader = extClassLoader.getParent(); // null

System.out.println(appClassLoader);     // sun.misc.Launcher$AppClassLoader
System.out.println(extClassLoader);     // sun.misc.Launcher$ExtClassLoader
System.out.println(bootstrapClassLoader); // null
```

# 3. 双亲委派模型（Parents Delegation Model）

**核心思想：** 类加载器收到类加载请求时，先委托父类加载器尝试加载，只有在父类加载器无法找到该类时，才尝试自己去加载。

```
请求加载 com.example.User
        │
        ▼
Application ClassLoader
  → "委托父加载器"
        │
        ▼
Extension/Platform ClassLoader
  → "委托父加载器"
        │
        ▼
Bootstrap ClassLoader
  → 搜索核心库
  → 未找到，返回 null
        │
        ▼
Extension/Platform ClassLoader
  → 搜索 ext / 平台模块
  → 未找到，返回 null
        │
        ▼
Application ClassLoader
  → 搜索 classpath
  → 找到，加载并返回
```

**`ClassLoader.loadClass()` 关键源码：**
```java
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    synchronized (getClassLoadingLock(name)) {
        // 1. 检查是否已加载
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            try {
                // 2. 委托父加载器
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    // 父类加载器为 null → 调用 Bootstrap ClassLoader
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // 父类加载不了，继续
            }
            if (c == null) {
                // 3. 自己加载
                c = findClass(name);
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```

# 4. 为什么需要双亲委派

核心目的：**保证 Java 核心类库的类型安全**。

- 防止用户自定义类覆盖 JDK 核心类（如自定义 `java.lang.Object`）。
- 避免同一个类被不同的类加载器加载而产生多个 `Class` 对象，导致类型转换异常。
- JVM 中对类的唯一性判定：**同一个类文件 + 同一个类加载器** = 同一个类。

```java
// 自定义 java.lang.String 将不会被加载
// 因为 Bootstrap ClassLoader 已经加载了 rt.jar 中的 String
package java.lang;
public class String {
    public static void main(String[] args) {
        // 报错：类 java.lang.String 中没有 main 方法
    }
}
```

# 5. 打破双亲委派

## 5.1 SPI 机制（线程上下文类加载器）

JDBC 是典型案例：
- `DriverManager` 由 Bootstrap ClassLoader 加载（rt.jar 中）。
- MySQL 驱动 `com.mysql.cj.jdbc.Driver` 在 classpath 下。
- Bootstrap ClassLoader 无法加载 classpath 下的类。
- 解决方案：通过 `Thread.currentThread().getContextClassLoader()` 获取应用类加载器。

```java
// JDBC 4.0+ 使用 SPI 机制，DriverManager 静态块中调用 ServiceLoader
// ServiceLoader 使用线程上下文类加载器加载实现类
ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
```

## 5.2 Tomcat 类加载器

Tomcat 打破双亲委派的原因：
- 一个 Tomcat 部署多个 Web 应用，各应用类库需要隔离。
- Web 应用间的类不应该互相可见。
- Web 应用共享库可以被共享。

Tomcat 的类加载体系：
```
Bootstrap ClassLoader
    ↑
System ClassLoader
    ↑
Common ClassLoader          ← Tomcat 通用类
    ↑        ↑
    │        └── Shared ClassLoader  ← 应用共享类（Tomcat 8+ 已移除）
    │
Webapp ClassLoader 1    Webapp ClassLoader 2  (各应用独立)
```

Webapp ClassLoader 加载顺序（缺省）：
1. 先在 Web 应用的 `/WEB-INF/classes`、`/WEB-INF/lib` 中查找
2. 让父 ClassLoader 加载
3. 自己在 `/WEB-INF` 中查找

这实际上"颠倒了"双亲委派顺序——优先从本地加载，避免应用类污染。

## 5.3 OSGi（模块化框架）

OSGi 的类加载器是网状结构而非树状，每个模块（Bundle）有自己的类加载器，模块间可以通过声明来依赖和暴露类，完全脱离了双亲委派模型。

## 5.4 热部署 / 热替换

```java
// 自定义类加载器实现热加载（每次用新类加载器加载新版本）
public class HotSwapClassLoader extends ClassLoader {
    private String classPath;
    
    public HotSwapClassLoader(String classPath) {
        this.classPath = classPath;
    }
    
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        try {
            byte[] bytes = loadClassBytes(name);
            return defineClass(name, bytes, 0, bytes.length);
        } catch (IOException e) {
            throw new ClassNotFoundException(name);
        }
    }
    
    private byte[] loadClassBytes(String name) throws IOException {
        String path = classPath + name.replace('.', '/') + ".class";
        try (FileInputStream fis = new FileInputStream(path);
             ByteArrayOutputStream baos = new ByteArrayOutputStream()) {
            byte[] buf = new byte[4096];
            int len;
            while ((len = fis.read(buf)) != -1) {
                baos.write(buf, 0, len);
            }
            return baos.toByteArray();
        }
    }
}
```

## 5.5 自定义 ClassLoader 实现场景

```java
public class DecryptClassLoader extends ClassLoader {
    private String classDir;
    
    public DecryptClassLoader(String classDir) {
        this.classDir = classDir;
    }
    
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        String path = classDir + name.replace('.', '/') + ".class";
        try {
            byte[] encrypted = Files.readAllBytes(Paths.get(path));
            byte[] decrypted = decrypt(encrypted);  // 解密
            return defineClass(name, decrypted, 0, decrypted.length);
        } catch (IOException e) {
            throw new ClassNotFoundException(name);
        }
    }
    
    private byte[] decrypt(byte[] data) {
        // 解密逻辑
        return data;
    }
}
```

# 6. 破坏双亲委派的通用方式

重写 `loadClass()` 而不是只重写 `findClass()`：
```java
@Override
protected Class<?> loadClass(String name, boolean resolve) 
        throws ClassNotFoundException {
    // 自己的类自己加载，不走双亲委派
    if (name.startsWith("com.myapp")) {
        return findClass(name);
    }
    // 其他类走双亲委派
    return super.loadClass(name, resolve);
}
```

# 7. 类的唯一性

```java
// 不同类加载器加载同一个类文件 → 两个不同的 Class 对象
ClassLoader loader1 = new MyClassLoader("path1");
ClassLoader loader2 = new MyClassLoader("path2");
Class<?> c1 = loader1.loadClass("com.example.User");
Class<?> c2 = loader2.loadClass("com.example.User");
System.out.println(c1 == c2);  // false

// instanceof 也受影响
Object obj = c1.newInstance();
// obj instanceof com.example.User  →  取决于当前类加载器
```

# 8. 类加载实战调试

```bash
# 打印类加载信息
java -XX:+TraceClassLoading MyApp

# 打印类卸载信息
java -XX:+TraceClassUnloading MyApp
```

示例输出：
```
[0.123s][info][class,load] Hello source: file:/path/to/classes/
[0.124s][info][class,load] java.lang.Object source: jrt:/java.base
[0.125s][info][class,load] java.lang.String source: jrt:/java.base
```