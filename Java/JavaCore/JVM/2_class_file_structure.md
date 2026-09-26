
# 1. Class 文件是什么

- Class 文件是 JVM 可识别的二进制字节码文件。
- Java、Kotlin、Scala 等语言的源代码编译后都会生成 `.class` 文件。
- 每个 Class 文件对应一个类或接口的定义。

**一个简单的 Java 类：**
```java
public class Hello {
    private int count;
    
    public void sayHello() {
        System.out.println("Hello JVM");
    }
}
```

编译后生成 `Hello.class`，约几千字节的二进制文件。

# 2. Class 文件总览

Class 文件是一组以 8 字节为单位的二进制流，严格按照以下结构排列：

```
ClassFile {
    u4             magic;               // 魔数
    u2             minor_version;       // 次版本号
    u2             major_version;       // 主版本号
    u2             constant_pool_count; // 常量池大小
    cp_info        constant_pool[constant_pool_count-1]; // 常量池
    u2             access_flags;        // 访问标志
    u2             this_class;          // 本类索引
    u2             super_class;         // 父类索引
    u2             interfaces_count;    // 接口数量
    u2             interfaces[interfaces_count]; // 接口索引表
    u2             fields_count;        // 字段数量
    field_info     fields[fields_count]; // 字段表
    u2             methods_count;       // 方法数量
    method_info    methods[methods_count]; // 方法表
    u2             attributes_count;    // 属性数量
    attribute_info attributes[attributes_count]; // 属性表
}
```

- `u1, u2, u4, u8` 分别表示 1/2/4/8 字节的无符号整数。

# 3. 魔数与版本号

| 字段 | 说明 |
|------|------|
| magic (4B) | 固定值 `CA FE BA BE`（十六进制），标识这是一个 Class 文件 |
| minor_version (2B) | 次版本号 |
| major_version (2B) | 主版本号，决定哪些 JVM 版本可以执行 |

**常见版本号对照：**
| major | Java 版本 | major | Java 版本 |
|-------|-----------|-------|-----------|
| 45 | JDK 1.1 | 52 | Java 8 |
| 46 | JDK 1.2 | 53 | Java 9 |
| 47 | JDK 1.3 | 54 | Java 10 |
| 48 | JDK 1.4 | 55 | Java 11 |
| 49 | JDK 5 | 56 | Java 12 |
| 50 | JDK 6 | 57 | Java 13 |
| 51 | JDK 7 | ... | ... |
| 52 | Java 8 | 61 | Java 17 |
|  |  | 65 | Java 21 |

- 高版本 JVM 可以运行低版本 Class 文件（向后兼容），反之不行。
- 编译时指定目标版本：`javac --release 8 Hello.java`（Java 9+）或 `javac -source 1.8 -target 1.8 Hello.java`

# 4. 常量池（Constant Pool）

- 常量池是 Class 文件的资源仓库，是第一个出现的表结构数据。
- 存放两大类常量：
  - **字面量**（Literal）：如字符串、final 常量值。
  - **符号引用**（Symbolic Reference）：类/接口全限定名、字段名称和描述符、方法名称和描述符。
- 常量池中的每一项都是 `cp_info` 结构，类型标识如：

| 标志 | 常量类型 | 说明 |
|------|---------|------|
| 1 | CONSTANT_Utf8 | UTF-8 编码字符串 |
| 3 | CONSTANT_Integer | 4 字节整数 |
| 7 | CONSTANT_Class | 类或接口的符号引用 |
| 9 | CONSTANT_Fieldref | 字段的符号引用 |
| 10 | CONSTANT_Methodref | 方法的符号引用 |
| 12 | CONSTANT_NameAndType | 字段或方法的名称和描述符 |

- 常量池大小 = `constant_pool_count - 1`（常量池索引从 1 开始，0 表示"不引用任何常量"）。

示例，`Hello.java` 编译后的常量池部分：
```
#1 = Methodref    #6.#15   // java/lang/Object."<init>":()V
#2 = Fieldref     #16.#17  // java/lang/System.out:Ljava/io/PrintStream;
#3 = String       #18      // "Hello JVM"
#4 = Methodref    #19.#20  // java/io/PrintStream.println:(Ljava/lang/String;)V
#5 = Class        #21      // Hello
...
```

# 5. 访问标志（Access Flags）

- 类或接口的修饰符标记：`public`、`final`、`abstract`、`interface`、`annotation`、`enum` 等。

| 标志值 | 含义 | 标志值 | 含义 |
|--------|------|--------|------|
| 0x0001 | ACC_PUBLIC | 0x0200 | ACC_INTERFACE |
| 0x0010 | ACC_FINAL | 0x0400 | ACC_ABSTRACT |
| 0x0020 | ACC_SUPER | 0x2000 | ACC_ANNOTATION |
| 0x1000 | ACC_SYNTHETIC | 0x4000 | ACC_ENUM |

- `public class Hello` → `ACC_PUBLIC | ACC_SUPER` = `0x0021`

# 6. 字段表（Fields）

```java
private int count;
static String name;
```

编译后对应两个 `field_info` 结构：
```
field_info {
    u2 access_flags;      // ACC_PRIVATE(0x0002)
    u2 name_index;        // 指向常量池 "count"
    u2 descriptor_index;  // 指向常量池 "I" (int的表示)
    u2 attributes_count;
    attribute_info attributes[];
}
```

**字段描述符对应表：**
| 描述符 | 类型 | 描述符 | 类型 |
|--------|------|--------|------|
| B | byte | J | long |
| C | char | F | float |
| D | double | S | short |
| I | int | Z | boolean |
| [ | 数组，如 `[I` = int[] | L<classname>; | 对象，如 `Ljava/lang/String;` |

# 7. 方法表（Methods）

- 方法体代码编译后存储在属性表中的 `Code` 属性内。
- 方法描述符格式：`(参数描述符)返回值描述符`

```java
public int add(int a, int b) { return a + b; }
```
描述符为：`(II)I` —— 两个 int 参数，返回 int。

```
method_info {
    u2 access_flags;       // ACC_PUBLIC
    u2 name_index;         // 指向 "add"
    u2 descriptor_index;   // 指向 "(II)I"
    u2 attributes_count;
    attribute_info attributes[];
}
```

# 8. 属性表（Attributes）

- 属性表是 Class 文件中扩展性最强的部分，可以出现在 Class 文件、字段表、方法表、Code 属性中。

**常用属性：**
| 属性名 | 位置 | 说明 |
|--------|------|------|
| Code | 方法表 | 方法的字节码指令 |
| LineNumberTable | Code | 字节码与源码行号对应（调试信息） |
| LocalVariableTable | Code | 局部变量表（调试信息） |
| SourceFile | Class文件 | 源文件名 |
| ConstantValue | 字段表 | final 常量值 |
| Signature | 类/字段/方法 | 泛型签名信息 |
| Deprecated | 类/字段/方法 | 过时声明 |
| StackMapTable | Code | JVM 字节码校验用 |
| BootstrapMethods | Class文件 | invokedynamic 引导方法 |

# 9. 实战：javap 反编译

```bash
# 查看所有信息
javap -verbose Hello.class

# 查看常量池
javap -c -p -v Hello.class
```

**输出关键部分解读：**
```
Classfile /path/to/Hello.class
  Last modified 2024-01-01; size 318 bytes
  MD5 checksum ...
  Compiled from "Hello.java"
public class Hello
  minor version: 0
  major version: 61   ← Java 17
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref    #6.#15   // java/lang/Object."<init>":()V
   ...
{
  public void sayHello();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2    // Field java/lang/System.out
         3: ldc           #3    // String Hello JVM
         5: invokevirtual #4    // Method PrintStream.println
         8: return
      LineNumberTable:
        line 4: 0
        line 5: 8
}
```

# 10. Class 文件的限制

- 每个类或接口最多 65535 个字段、65535 个方法。
- 方法字节码最长 65535 字节。
- 常量池最多约 65535 条。
- 这些限制由 Class 文件格式中的 `u2` 类型导致。