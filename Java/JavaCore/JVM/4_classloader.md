
# 1. 类加载器的作用

## 1.1 职责边界：只负责"加载"

- JVM 的类加载动作由类加载器完成，但**加载只是 7 阶段的第一步**：验证、准备、解析、初始化不归它管（见 [3_class_lifecycle](3_class_lifecycle.md)）。
- 一句话：类加载器 = "字节流搬运工 + defineClass 触发者"。

## 1.2 类身份的两个维度

- 运行时的"一个类"由 **(全限定类名, 加载它的 ClassLoader)** 共同确定。
- 类加载器同时划定了**命名空间**：同一加载器内类名唯一；不同加载器之间互不干涉。

## 1.3 演示：同名类不是同一个类

```java
URL[] urls = { new URL("file:/app/classes/") };
ClassLoader loader1 = new URLClassLoader(urls);
ClassLoader loader2 = new URLClassLoader(urls);   // 同一路径、不同实例

Class<?> c1 = loader1.loadClass("com.example.User");
Class<?> c2 = loader2.loadClass("com.example.User");

System.out.println(c1 == c2);                     // false
System.out.println(c1.equals(c2));                // false（Class.equals 比较 名字+loader）
Object u = c1.getDeclaredConstructor().newInstance();
System.out.println(c2.isInstance(u));             // false —— instanceof 同样判false
User cast = (User) u;                             // 若当前类的 User 来自 loader2 → ClassCastException
```

```text
        类名字符串相同                但
   ┌────────────────────┐      ┌──────────────────────────┐
   │ com.example.User   │      │ loader1≠loader2 → 两个独立的 Class、两份静态变量、互不兼容的类型
   └────────────────────┘      └──────────────────────────┘
```

# 2. 类加载器层次与来源

## 2.1 分类总览

JVM 将类加载器划分为两种：
- **启动类加载器**（Bootstrap ClassLoader），C++ 实现，JVM 的一部分。
- **其他类加载器**（Java 实现，继承 `java.lang.ClassLoader`）。

## 2.2 三层结构（Bootstrap / Platform(Extension) / App）

三层类加载器（Java 8 及之前）：
```text
Bootstrap ClassLoader       ← 加载 <JAVA_HOME>/lib 下的 rt.jar 等
        ↑
Extension ClassLoader       ← 加载 <JAVA_HOME>/lib/ext 目录
        ↑
Application ClassLoader     ← 加载用户 classpath 下的类
```

Java 9+ 变化（模块化）：
```text
Bootstrap ClassLoader       ← 加载 java.base 等核心模块
        ↑
Platform ClassLoader        ← 取代 Extension，加载 java.se 等平台模块
        ↑
Application ClassLoader     ← 加载 classpath / module-path
```

| 加载器 | JDK 8 | JDK 9+ | 加载内容 |
|--------|-------|--------|----------|
| Bootstrap | C++，无 Java 对象（parent 链上是 null） | 同左；实现入口变为 `jrt:/` 文件系统访问 jmod | `java.base` 核心类（Object/String/System…） |
| Extension → Platform | `sun.misc.Launcher$ExtClassLoader` | 更名 `jdk.internal.loader.ClassLoaders$PlatformClassLoader` | `java.se` 等平台模块（`-Djava.ext.dirs` 目录消失） |
| Application | `sun.misc.Launcher$AppClassLoader` | `jdk.internal.loader.ClassLoaders$AppClassLoader` | classpath / module-path 用户类 |
| 自定义 | 继承 ClassLoader | 同左；JPMS 下还有 Layer 机制 | 按需 |

## 2.3 各加载器职责细节

### 2.3.1 Bootstrap：C++ 实现，加载 jmods/模块核心
- 由 `libjvm` 在 `JNI_CreateJavaVM` 时创建；识别 `-Xbootclasspath`（JDK 9 后改名 `--boot-class-path` 且不再支持用户追加类）。
- 它不继承 `java.lang.ClassLoader`——所以 `getParent()` 返回 null 是"没有 Java 父对象"，不是"没有父亲"。

### 2.3.2 获取与验证（保留原示例 + JDK 9+ 类名变化）

```java
// Application ClassLoader 获取
ClassLoader appClassLoader = ClassLoader.getSystemClassLoader();

// Extension / Platform ClassLoader
ClassLoader extClassLoader = appClassLoader.getParent();

// Bootstrap ClassLoader (C++ 实现，返回 null)
ClassLoader bootstrapClassLoader = extClassLoader.getParent(); // null
```

JDK 8 输出：
```text
sun.misc.Launcher$AppClassLoader@1db4d3a1
sun.misc.Launcher$ExtClassLoader@5e8c92f4
null
```
JDK 21 的 jshell 验证：
```java
// jshell
ClassLoader c = ClassLoader.getSystemClassLoader();
while (c != null) {
    System.out.println(c.getClass().getName());
    c = c.getParent();
}
// jdk.internal.loader.ClassLoaders$AppClassLoader@…
// jdk.internal.loader.ClassLoaders$PlatformClassLoader@…
// （循环结束，Bootstrap 在 Java 侧不可见 → null）
```

## 2.4 自定义类加载器入口：findClass

- 官方约定：**重写 `findClass` 而不是 `loadClass`**（想破坏委派才动 loadClass，见第 5/6 节）。
- 核心动作：拿到 byte[] → `defineClass(name, b, off, len)` → 得到 Class 对象。

# 3. 双亲委派模型（Parents Delegation Model）

## 3.1 核心思想

**核心思想：** 类加载器收到类加载请求时，先委托父类加载器尝试加载，只有在父类加载器无法找到该类时，才尝试自己去加载。

## 3.2 "双亲"是 parent 引用，不是继承两个爹

- 英文原文 *Parents Delegation Model* 的 parent 指 **组合字段** `ClassLoader.parent`（构造时传入），与 `extends` 无关。
- "委派"是**责任链模式**：每层加载器持有上一层的引用，请求沿引用上抛。

## 3.3 委派流程图

### 3.3.1 上溯与回传路径

```text
请求加载 com.example.User
        │  (先查自己已加载缓存 findLoadedClass)
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
  → 未找到，向下回传
        │
        ▼
Application ClassLoader
  → 搜索 classpath（findClass）
  → 找到，defineClass 并返回
```

### 3.3.2 图中三步与代码分支的对应

| 图中步骤 | loadClass 代码分支（源码见 3.4） |
|----------|----------------------------------|
| 先查缓存 | `findLoadedClass(name)` 命中 → 直接返回 |
| 逐级委托父加载器 | `parent.loadClass(name, false)`；parent 为 null → `findBootstrapClassOrNull(name)` |
| 上溯全失败才自己加载 | 父辈抛 `ClassNotFoundException` 被吞 → `c == null` → `findClass(name)` |

## 3.4 `ClassLoader.loadClass()` 关键源码

```java
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    synchronized (getClassLoadingLock(name)) {   // JDK 7u40+ 支持按类名并行加锁
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

## 3.5 覆写约定：findClass vs loadClass

| 覆写点 | 效果 | 使用场景 |
|--------|------|----------|
| `findClass` | 保持委派，只在"父辈都找不到"时被调用 | 99% 的自定义加载器（加密类、隔离类） |
| `loadClass` | 改变/绕过委派顺序 | Tomcat、热部署等"破坏双亲委派"场景 |
| `getClassLoadingLock` | 把"按加载器串行"细化为"按类名并行" | 高并发启动加速 |

# 4. 为什么需要双亲委派

## 4.1 核心目的：保证 Java 核心类库的类型安全

- 防止用户自定义类覆盖 JDK 核心类（如自定义 `java.lang.String` / `java.time` 相关类）。
- 没有委派时，classpath 上恶意放一个 `java.sql.DriverManager` 冒名顶替就能劫持数据库连接；有委派时这类请求**永远先到 Bootstrap**，核心版本一旦加载即命中缓存，伪造类根本没机会。

```java
// 自定义 java.lang.String 将不会被加载
// 因为 Bootstrap ClassLoader 已经加载了核心库中的 String
package java.lang;
public class String {
    public static void main(String[] args) {
        // 报错：类 java.lang.String 中没有 main 方法
        // （main 方法找不到的是真 String 的签名字符串，伪造版根本没被使用）
    }
}
// JDK 9+ 更狠：编译器直接拒绝为 java.* 包生成类
```

## 4.2 防重复加载

- 避免同一个类被不同的类加载器加载而产生多个 `Class` 对象，导致类型转换异常。
- 委派链保证：核心类只可能有一份（Bootstrap 命名空间）。

## 4.3 体系化协作

- 分层=分工：每层只负责自己可信的来源域（核心库/平台库/应用库），"谁离信任根越近，谁优先说话"。
- JVM 中对类的唯一性判定：**同一个类文件 + 同一个类加载器** = 同一个类。

# 5. 打破双亲委派

## 5.1 SPI 机制（线程上下文类加载器）

### 5.1.1 问题背景：接口在核心库、实现在外部 jar

JDBC 是典型案例：
- `DriverManager` 由 Bootstrap ClassLoader 加载（核心库中）。
- MySQL 驱动 `com.mysql.cj.jdbc.Driver` 在 classpath 下。
- Bootstrap ClassLoader **向下**无法加载 classpath 下的类——委派链只能上溯不能下探，这是模型的方向性缺陷。
- 解决方案：通过 `Thread.currentThread().getContextClassLoader()` 获取应用类加载器，"借"它向下加载。

### 5.1.2 反向委派时序图

```text
        ┌────────────┐  load(DriverManager)  ┌────────────┐ load(核心类之外的实现) ┌───────────┐
        │ Bootstrap  │──────────────────────▶│   线程TCCL  │◀──────────────────────│ App CL    │
        └────────────┘                       └────────────┘   （反向委派！）        └───────────┘
DriverManager.<clinit> ─▶ ServiceLoader 按 META-INF/services/java.sql.Driver ─▶ 加载并注册各实现
```

### 5.1.3 ServiceLoader 用法代码

```java
// JDBC 4.0+ 使用 SPI 机制，DriverManager 静态块中调用 ServiceLoader
// ServiceLoader 使用线程上下文类加载器加载实现类
ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
for (Driver d : loadedDrivers) {           // 遍历即实例化并注册
    System.out.println(d.getClass() + " acceptURL: "
        + d.acceptsURL("jdbc:mysql://localhost/test"));
}
// 约定文件: jar 内 META-INF/services/java.sql.Driver，内容是实现类全限定名
```

- 同一套路：JNDI、JAXP、`SPI` 全家（`java.util.ServiceLoader` 的加载器默认取 TCCL）。

## 5.2 Tomcat 类加载器

### 5.2.1 为什么容器必须偏离双亲委派

Tomcat 打破双亲委派的原因：
- 一个 Tomcat 部署多个 Web 应用，各应用类库需要隔离。
- Web 应用间的类不应该互相可见。
- Web 应用共享库可以被共享；且 JSP 修改后要能**单独换掉一个类的加载器**热替换。

### 5.2.2 结构图与各级加载器

Tomcat 的类加载体系：
```text
Bootstrap ClassLoader
    ↑
System ClassLoader          ← Tomcat 自身的启动 jar
    ↑
Common ClassLoader          ← $CATALINA_HOME/lib，容器+应用都可见
    ↑        ↑
    │        └── Shared ClassLoader  ← 应用共享类（Tomcat 8+ 已移除）
    │
 Server ClassLoader         ← 只对 Tomcat 内部可见（8.0+）
    ↑        ↑
Webapp ClassLoader 1    Webapp ClassLoader 2  (各应用独立，互为兄弟不可见)
    └ JSP ClassLoader（每个 JSP 一个，改 JSP 即换 loader 热替换）
```

Webapp ClassLoader 加载顺序（缺省 delegate=false）：
1. 先在 Web 应用的 `/WEB-INF/classes`、`/WEB-INF/lib` 中查找
2. 让父 ClassLoader 加载
3. 自己在 `/WEB-INF` 中查找

这实际上"颠倒了"双亲委派顺序——优先从本地加载，避免应用类污染。

### 5.2.3 delegate 模式与版本演变

- Servlet 规范同时要求 **JavaSE 核心类仍必须走父委派**（由专门的 javaseClassLoader 兜底），所以它是"局部倒置"而非全盘推翻。
- 配置 `delegate=true` 可切回父优先；Tomcat 7+ 默认实现是 `ParallelWebappClassLoader`（按类名加锁并行加载）。
- 演进注脚：JDK 9+ 模块化与 Tomcat 10（jakarta 命名空间）时代，shared loader 早已删除、结构进一步简化为 Common→Server→Webapp 主干。

## 5.3 OSGi 与模块化

- OSGi 的类加载器是**网状结构**而非树状：每个 Bundle 有自己的类加载器，模块间通过 `Import-Package/Export-Package` 声明依赖与暴露，委派方向按声明动态决定，完全脱离双亲委派模型。
- JDK 9 的 JPMS 某种意义上是"官方收编"了这一需求：模块声明依赖（`requires`/`exports`），类查找先按模块解析；但底层仍复用加载器分层，属于"补强"而非"破坏"。

## 5.4 热部署 / 热替换

### 5.4.1 HotSwapClassLoader 骨架（只写 findClass，不动 loadClass）

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

### 5.4.2 换类 = 换加载器

使用骨架——"换类 = 换加载器"：

```java
for (;;) {
    HotSwapClassLoader loader = new HotSwapClassLoader("out/"); // 新实例=新命名空间
    Object job = loader.loadClass("com.example.Job")
                        .getDeclaredConstructor().newInstance();
    job.getClass().getMethod("run").invoke(job);
    // 丢弃 loader 与全部实例引用 → 旧类失去可达性 → 可被卸载（见 [3_class_lifecycle](3_class_lifecycle.md) 第8节）
}
```

- 约束：只有**非启动路径**的类能被这样玩；旧类若被静态字段/线程悄悄引用，会演变成类加载器泄漏（见 8.4）。
- 另一路线是 JVM 自带的 HotSwap（`-agentlib:jdwp`、Instrumentation `redefineClasses`），但只允许改方法体，不允许加字段改结构。

## 5.5 自定义 ClassLoader 实现场景（加密字节流）

### 5.5.1 覆写 findClass 骨架

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
            byte[] decrypted = decrypt(encrypted);  // 解密后就是标准 Class 文件字节流
            return defineClass(name, decrypted, 0, decrypted.length);
        } catch (IOException e) {
            throw new ClassNotFoundException(name);
        }
    }

    private byte[] decrypt(byte[] data) {
        // 解密逻辑（异或/AES 均可）
        return data;
    }
}
```

### 5.5.2 解密字节流的注意点

- 关键：魔数校验发生在解密**之后**——所以加密工具必须对完整字节流可逆（见 [2_class_file_structure](2_class_file_structure.md) 10.2）。
- 派生类注意：签名 jar 解密后 `Signature` 失效 → CodeSource/证书链检查会抛异常，需要白名单或重签名策略；仅"防逆向"级别的 XOR 加密挡不住 agent 直接 dump 内存。

# 6. 破坏双亲委派的通用方式

## 6.1 重写 loadClass（改变委派顺序）

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

## 6.2 线程上下文类加载器（TCCL，反向委派）

```java
ClassLoader tccl = Thread.currentThread().getContextClassLoader();
Class<?> impl = Class.forName("com.example.DriverImpl", true, tccl);
```

- 原理：加载器对象是普通引用，核心库代码"借用"上层（数量意义上更下层）加载器，即绕过委派方向。

## 6.3 线程池下的 TCCL 陷阱

```text
线程 A(webapp请求) 提交任务 ──▶ 线程池 worker 早就创建完毕
                                 worker 的 contextClassLoader = 创建那一刻的捕获值
```

| 注意点 | 说明 |
|--------|------|
| 继承时机 | 线程的 TCCL 在**创建时**从父线程复制；池内老线程不会随请求变化自动更新 |
| 应用重部署 | worker 持有已废弃 WebappClassLoader → 旧类全被 pin 住， metaspace 泄漏 |
| 对策 | 提交任务时显式 `t.setContextClassLoader(reqClassLoader)`，或用装饰器统一处理；池随应用一起销毁 |

# 7. 类的唯一性与命名空间

## 7.1 命名空间定义

- 一个类加载器实例 + 其委派得到的所有父辈 = 该加载器的**导入命名空间**；它自己定义的类构成**本地命名空间**。
- 不同加载器实例 = 不同命名空间：1.3 的 `c1 != c2` 即由此而来；两份类的静态变量也各自独立（热部署/多租户的基础设施属性）。

## 7.2 并行加载的隔离

- 双亲委派是"名字唯一"的粗粒度模型；JDK 9 模块系统在其上再叠一层：**类的运行时标识 = (Module, Class)**，`--module-path` + Layer（`ModuleLayer.boot().defineModule(...)`）可在同一 JVM 内装载同名类的多个版本而互不冲突。
- JPMS 没有取代加载器，而是给加载器"记账"：每个 ClassLoader 关联一组 Module。

## 7.3 Tomcat 应用间类隔离图

```text
              Common CL (lib/*)
             ┌──────┴───────┐
      WebappCL-A          WebappCL-B
   guava 20.0(私有)     guava 31.0(私有)
        │                    │
   A 引不到 B 的类        B 引不到 A 的类
   —— 同 FQCN 不同 loader，天然共存
```

# 8. 实战排查

## 8.1 打印 ClassLoader 树

```java
public static void printLoaderTree(Class<?> clazz) {
    ClassLoader cl = clazz.getClassLoader();
    int depth = 0;
    while (cl != null) {
        System.out.println("  ".repeat(depth) + cl.getClass().getName()
                           + " @" + Integer.toHexString(cl.hashCode()));
        if (cl instanceof java.net.URLClassLoader u) {
            for (var url : u.getURLs()) System.out.println("  ".repeat(depth+1) + url);
        }
        cl = cl.getParent();
        depth++;
    }
    System.out.println("  ".repeat(depth) + "(Bootstrap)");
}
```

## 8.2 观测命令

```bash
# JDK 8：旧式 Trace 参数
java -XX:+TraceClassLoading MyApp
java -XX:+TraceClassUnloading MyApp

# JDK 9+：统一日志（当前基线 JDK 21 用这套）
java -Xlog:class+load=info MyApp
java -Xlog:class+load,class+unload,cds=info MyApp

# 运行中进程：类与加载器统计（较新 JDK 提供，命令名随版本略有差异）
jcmd <pid> VM.classloaders          # 列出存活加载器及各类数量
jcmd <pid> VM.classloader_stats     # 加载器内存/统计视图（部分版本）
jcmd <pid> GC.class_histogram | grep -i ClassLoader   # 可疑加载器实例数（泄漏信号）
```

示例输出：
```text
[0.123s][info][class,load] Hello source: file:/path/to/classes/
[0.124s][info][class,load] java.lang.Object source: jrt:/java.base
[0.125s][info][class,load] java.lang.String source: jrt:/java.base
```

## 8.3 ClassNotFoundException vs NoClassDefFoundError

### 8.3.1 成因对比表

| 维度 | ClassNotFoundException | NoClassDefFoundError |
|------|----------------------|----------------------|
| 本质 | 受检异常：显式加载请求失败 | Error：编译期存在、运行期找不到/不可用 |
| 触发点 | `Class.forName` / `loadClass` / ServiceLoader 找不到类 | 首次使用类（new/静态访问/校验）时解析失败 |
| 典型成因 | 名字写错、可选依赖没带、SPI 配置文件类名对不上 | jar 没打进包、`provided` 范围误用、依赖冲突后版本被仲裁掉 |
| 特殊变体 | — | `Could not initialize class X`：该类**此前 clinit 已失败**，真凶在更早日志里（ExceptionInInitializerError） |
| 快速定位 | 栈顶即现场 | **别只看第一次**：往前翻首次异常；`-Xlog:class+load` 看该类到底有没有被加载 |
| 配套排查 | `mvn dependency:tree` / `gradle dependencies` 查版本仲裁；`unzip -l xx.jar` 验证字节码在场 | 同左 + 见 [22_troubleshooting](22_troubleshooting.md) |

### 8.3.2 典型场景清单

| 异常 | 高频现场 |
|------|----------|
| CNFE | 框架反射装配：类名拼错 / 可选依赖未引入；`META-INF/services/` 里写的实现类不在 classpath；shade 重定位改了包名 |
| NCDFE | `provided` 依赖上线后容器没有对应实现；依赖仲裁把含类的 jar 整个降级掉（常伴生 `NoSuchMethodError`/`NoClassDefFoundError: xxx`）；该类 `<clinit>` 早前已失败，第二次访问直接 NCDFE |

## 8.4 类加载器泄漏（卸载失败的根因链）

```text
Metaspace OOM / 堆越换越大 的常见链:

泄漏的非守护线程/ThreadLocal/注册中心(DriverManager!、shutdownHook、MBean)
        │ 强引用
        ▼
  旧 WebappClassLoader ←─ 本该随应用重部署被丢弃
        │ 引用
        ▼
  它加载的全部 Class + 静态变量
        ▼
  三个卸载条件永远无法同时成立（见 [3_class_lifecycle](3_class_lifecycle.md) 8.1）
```

- 处置套路：重部署后用 `jcmd VM.classloaders` 数加载器实例是否只 +1；MAT 支配树从 ClassLoader 入手下钻；线程池与 JDBC Driver 注册是两大惯犯。

## 8.5 设计自定义加载器检查单

| 检查项 | 理由 |
|--------|------|
| 是否真的需要破坏委派？优先只写 findClass | 保持类型系统一致性（第 4 章收益不丢） |
| 核心库类（java.* / javax.*）是否仍走父委派 | 避免 ClassCastException 风暴 |
| 是否处理了并行加载锁（getClassLoadingLock） | JDK 7u40+ 的加速与死锁规避 |
| 新加载器有没有被旧引用 pin 住 | 决定热替换能否真正释放内存（8.4） |
| 加密/签名类：defineClass 时是否带 CodeSource/ProtectionDomain | 安全域丢失导致权限异常 |
