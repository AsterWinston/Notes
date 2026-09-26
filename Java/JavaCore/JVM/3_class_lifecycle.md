
# 1. 七个阶段

```
加载 → 验证 → 准备 → 解析 → 初始化 → 使用 → 卸载
│        │                 │        │
│<──────连接─────────>│        │        │
│        │                 │        │
│<────────类加载过程────────────────>│
```

- 加载、验证、准备、初始化、卸载这 5 个阶段顺序是确定的。
- 解析阶段可以在初始化之后开始（支持动态绑定）。
- 使用阶段就是对象创建、方法调用等。

# 2. 加载（Loading）

JVM 完成三件事：
1. 通过全限定名获取类的二进制字节流。
2. 将字节流代表的静态存储结构转化为方法区的运行时数据结构。
3. 在堆中生成 `java.lang.Class` 对象，作为访问入口。

获取字节流的常见方式：
- 从 ZIP/JAR 包读取（最常见）
- 从网络获取（Applet）
- 运行时计算生成（动态代理、CGLIB）
- 从数据库读取
- 动态编译（JSP → Servlet）

**数组类 vs 非数组类：**
- 非数组类由类加载器加载。
- 数组类由 JVM 直接在内存中动态构建（不经过类加载器）。

# 3. 验证（Verification）

确保 Class 文件字节流符合 JVM 规范，不会危害 JVM 安全。

验证的 4 个阶段：
| 阶段 | 说明 |
|------|------|
| 文件格式验证 | 魔数、版本号、常量池类型等 |
| 元数据验证 | 是否有父类、是否正确继承、抽象方法实现等 |
| 字节码验证 | 操作数栈类型匹配、跳转指令合法等 |
| 符号引用验证 | 引用的类/方法/字段是否存在、访问权限检查 |

**生产环境参数：**
```bash
# 关闭大部分验证（加快启动，已充分测试后使用）
-Xverify:none
```

- Java 7 之后 `-Xverify:none` 已移除，但可缩短验证阶段。

# 4. 准备（Preparation）

- 为类变量（static）分配内存并赋予**零值**。
- 这些内存在方法区（Java 8 后在元空间）中分配。
- 注意：`static final` 的常量在编译期赋值，在准备阶段直接赋予最终值。

```java
public static int value = 123;  // 准备阶段 value = 0，初始化阶段才赋 123
public static final int CONST = 123;  // 编译时就确定，准备阶段 CONST = 123
```

各类型零值：
| 类型 | 零值 | 类型 | 零值 |
|------|------|------|------|
| int | 0 | boolean | false |
| long | 0L | char | '\u0000' |
| float | 0.0f | short | (short)0 |
| double | 0.0d | byte | (byte)0 |
| 引用类型 | null |  |  |

# 5. 解析（Resolution）

将常量池中的符号引用替换为直接引用。

| 符号引用 | 直接引用 |
|----------|----------|
| 一组符号描述目标 | 指向目标的指针、偏移量或句柄 |
| 与 VM 实现无关 | 与 VM 内存布局直接相关 |

解析目标：
- 类或接口解析
- 字段解析
- 方法解析（类方法 + 接口方法）

**解析发生时机：** 可以在类加载后就解析（静态解析），也可以在符号引用被首次使用时解析（惰性解析，HotSpot 默认）。

# 6. 初始化（Initialization）

- 执行类构造器 `<clinit>()` 方法。
- `<clinit>` 由编译器收集静态变量赋值、静态代码块合并产生。
- 类初始化的时机（主动引用）：
  1. `new` 实例化对象
  2. 访问类的静态变量 / 静态方法
  3. 反射调用
  4. 初始化子类时先初始化父类
  5. 启动类（main 方法所在类）
  6. 动态语言支持（`MethodHandle`）
  7. 接口的 default 方法

**被动引用（不会触发初始化）：**
```java
// 1. 访问父类静态字段，不会初始化子类
class Parent { static int x = 1; }
class Child extends Parent { static int y = 2; }
int v = Child.x;  // 只初始化 Parent

// 2. 通过数组定义来引用类
Parent[] arr = new Parent[10];  // 不触发 Parent 初始化

// 3. 访问编译期常量
class Const { static final int V = 1; }
int v = Const.V;  // 编译期常量传播，不初始化 Const
```

# 7. `<clinit>()` 类构造器

```java
public class Example {
    static int a = 1;
    static int b;
    
    static {
        b = a * 2;
        System.out.println("clinit executed");
    }
}
```

编译后生成 `<clinit>` 等效于：
```
static {};
  descriptor: ()V
  Code:
     0: iconst_1
     1: putstatic   #2   // a = 1
     4: getstatic   #2
     7: iconst_2
     8: imul
     9: putstatic   #3   // b = a * 2
    12: getstatic   #4   // System.out
    15: ldc         #5   // "clinit executed"
    17: invokevirtual #6
    20: return
```

关键特性：
- `<clinit>()` 由 JVM 保证线程安全（加锁执行）。
- 父类 `<clinit>` 先于子类执行。
- 接口的 `<clinit>` 只有在使用到其字段或 default 方法时才执行。
- 如果类没有静态变量和静态块，则不生成 `<clinit>`。

# 8. 类的卸载

- 类卸载条件严格：该类的 `Class` 对象被 GC 回收，且该类所有的实例对象都已经回收，且加载该类的类加载器被回收。
- 由启动类加载器、扩展类加载器、系统类加载器加载的类在整个 JVM 生命周期中几乎不会被卸载。
- **只有自定义类加载器加载的类才可能被卸载**。
- 实际应用：热部署（如 Tomcat 重新加载应用）通过替换类加载器实现类卸载。

```java
// 验证类卸载需要结合 GC 日志
// -XX:+TraceClassLoading -XX:+TraceClassUnloading
```

# 9. 实战：观察类加载顺序

```java
class Parent {
    public static int A = 1;
    static { System.out.println("Parent static block, A=" + A); }
}

class Child extends Parent {
    public static int B = A;
    static { System.out.println("Child static block, B=" + B); }
    
    public static void main(String[] args) {
        System.out.println("main start");
        new Child();
    }
}
```

输出：
```
Parent static block, A=1
Child static block, B=1
main start
```

- `Child.main()` 触发 Child 的初始化，但父类先初始化。
- `<clinit>()` 只执行一次。
- `new Child()` 不再触发静态块（已初始化过）。

# 10. 初始化中的线程安全问题

```java
public class Singleton {
    private static Singleton instance;
    
    static {
        // JVM 保证 clinit 的线程安全，这里不会产生多实例
        instance = new Singleton();
    }
    
    public static Singleton getInstance() {
        return instance;
    }
}
```

- JVM 通过锁机制保证 `<clinit>` 方法的线程安全。
- 如果 `<clinit>` 中存在耗时操作，会导致该类初始化阻塞其他线程。
- 注意避免 `<clinit>` 中的循环依赖（可能死锁）。