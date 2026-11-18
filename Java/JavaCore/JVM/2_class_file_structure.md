
# 1. Class 文件是什么

## 1.1 一份"字节流规范"，而非 Java 语法

- Class 文件是 JVM 可识别的二进制字节码文件，其格式由 JVMS 第 4 章以**字节级伪代码**严格规定。
- Java、Kotlin、Scala 等语言的源代码编译后都会生成 `.class` 文件；**每个 Class 文件对应一个类或接口的定义**。
- 关键点：规范约束的是"字节怎么排"，不关心"源码怎么写"——所以任何语言只要能按格式吐出字节流，JVM 就照单全收。

**一个简单的 Java 类：**
```java
public class Hello {
    private int count;

    public void sayHello() {
        System.out.println("Hello JVM");
    }
}
```

编译后生成 `Hello.class`，约几百到几千字节的二进制文件。

## 1.2 伪结构的四大块

```text
┌─────────────────────────────────────────────┐
│ ① 版本信息: magic + minor + major           │ ← "我是谁、我几岁"
├─────────────────────────────────────────────┤
│ ② 常量池 constant_pool                      │ ← 全文件共享的"资源字典"
├─────────────────────────────────────────────┤
│ ③ 类骨架: access_flags + this/super/接口    │ ← "我是什么、我爹是谁"
├─────────────────────────────────────────────┤
│ ④ 成员与扩展: fields[] / methods[] /        │
│              attributes[]                   │ ← 字段表·方法表·属性表
└─────────────────────────────────────────────┘
        所有对名字/类型的引用 → 只存常量池索引
```

## 1.3 为什么是严格紧凑、无 padding 的格式

| 设计选择 | 原因 |
|----------|------|
| 字段间**没有任何分隔符/对齐 padding** | 文件要通过网络传输、内嵌进 jar、被 Applet 时代下载，字节必须寸土寸金 |
| 一切定长或自带长度前缀 | 解析器可以**顺序流式读取**，无需回溯、无需 seek |
| 引用一律用 u2 索引 | 把重复出现的字符串/类名收敛进常量池去重 |
| 大端序（Big-Endian）固定 | 消除跨 CPU 字节序歧义，Moto 网络字节序习惯 |
- 推论：Class 文件**不能**用"偏移量"硬编码解析——常量池每项长度不同，必须逐项读 tag 才知道跳几个字节。

# 2. Class 文件总览

## 2.1 ClassFile 伪代码

Class 文件是一组以 **8 bit（1 字节）为单位**的二进制流，严格按照以下结构排列：

```text
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

## 2.2 基本数据类型

- `u1, u2, u4, u8` 分别表示 1/2/4/8 字节的**无符号**整数。
- 没有 u3/s1 之类的有符号类型——有符号数值（如 `CONSTANT_Integer`）用补码存放在 u4 里，语义由使用方解释。

## 2.3 自描述与顺序固定

- 前面 10 个字节（magic+minor+major）任何解析器都能盲读；之后每一项的长度都可由"已读内容"推出——这是它能被 ASM 等工具**从任意字节流中间重建**的基础。
- 例外：属性表**不规定顺序、允许未知属性**，读到不认识的属性直接按 `attribute_length` 跳过——向后兼容的扩展点（新 JDK 特性几乎都以新属性落地，见第 8 节）。

# 3. 魔数与版本号

## 3.1 魔数（magic）

### 3.1.1 0xCAFEBABE 的由来

| 字段 | 说明 |
|------|------|
| magic (4B) | 固定值 `CA FE BA BE`（十六进制），标识这是一个 Class 文件 |

- 彩蛋名场面：Mac 启动时报"笑脸炸弹"的硬件代码正是 0xCAFEBABE；Sun 工程师顺手取来做魔数，"看着开心"即可。
- 为什么用魔数而不是扩展名 `.class`：JVM 的输入是**字节流**，来源可能是 jar/网络/数据库/内存生成（见 10.2），根本没有文件名——只有头 4 字节能证明身份。jar 里的 `.class` 后缀只是给人看的。

### 3.1.2 其他格式的魔数对照

| 格式 | 魔数 |
|------|------|
| Class | `CA FE BA BE` |
| zip/jar | `50 4B`（"PK"） |
| gzip | `1F 8B` |
| ELF | `7F 45 4C 46` |

## 3.2 版本号

### 3.2.1 major version 与 JDK 对照

- minor_version (2B)：次版本号；恒为 0 表示正式版，非 0 是历史遗留的预发布标记（如 45.3、45.65535）。
- major_version (2B)：主版本号，决定哪些 JVM 版本可以执行。

| major | Java 版本 | major | Java 版本 |
|-------|-----------|-------|-----------|
| 45 | JDK 1.1 | 55 | Java 11 |
| 46 | JDK 1.2 | 58 | Java 14 |
| 47 | JDK 1.3 | 61 | **Java 17 (LTS)** |
| 48 | JDK 1.4 | 65 | **Java 21 (LTS)** |
| 49 | JDK 5 | 68 | Java 24 |
| 50 | JDK 6 | 69 | **Java 25 (LTS)** |
| 51 | JDK 7 | 70 | Java 26 |
| 52 | Java 8 | — | …… |

- 速记：Java N（≥8）的 major = 44 + N（8→52，17→61，21→65，25→69）。
- 高版本 JVM 可以运行低版本 Class 文件（向后兼容），反之不行。
- 编译时指定目标版本：`javac --release 8 Hello.java`（Java 9+，**推荐**，同时锁 API 签名）或 `javac -source 1.8 -target 1.8 Hello.java`（只改字节码版本，API 仍是新版，可能埋雷）。

### 3.2.2 UnsupportedClassVersionError 案例分析

```text
Exception in thread "main" java.lang.UnsupportedClassVersionError:
  com/example/App has been compiled by a more recent version of the
  Java Runtime (class file version 65.0), this version of the Java
  Runtime only recognizes class file versions up to 61.0
```

| 解读 | 结论 |
|------|------|
| 65.0 | 这个 jar 用 **JDK 21** 编译 |
| up to 61.0 | 运行它的是 **JDK 17** JVM |
| 修复选项 | 升级运行时 / `javac --release 17` 重编 / 检查 CI 是否用错 JDK |
- 排查命令：`javap -v X.class | findstr version`，或直接批量扫 jar：`unzip -p lib.jar META-INF/... ` + `xxd -l 8 foo.class`（前 8 字节即 magic+version）。

# 4. 常量池（Constant Pool）

> 本章是 Class 文件的心脏，篇幅最长；加载后它进入方法区成为"运行时常量池"（见 [7_method_area](7_method_area.md)）。

## 4.1 两类常量

- 常量池是 Class 文件的资源仓库，是版本号之后**第一个**出现的表结构。
- 存放两大类内容：
  - **字面量**（Literal）：字符串 `"Hello JVM"`、`final` 编译期常量值、数字。
  - **符号引用**（Symbolic Reference）：类/接口全限定名、字段名+描述符、方法名+描述符——**编译期不知道目标内存地址，只能以"名字"引用**。

## 4.2 结构特点

### 4.2.1 索引规则

- `constant_pool_count = 实际项数 + 1`：索引从 **1** 开始，**0 表示"不引用任何常量"**（如 `catch_type=0` 代表 finally）。
- **long / double 各占两项**：每项本身 8 字节，规范规定其后续一项被"吞掉"不可用——所以第 5 项若是 long，第 6 项索引直接跳过。
- 每项以 1 字节 `tag` 开头，长度由 tag 决定 → 解析必须顺序逐项推进。

### 4.2.2 入口类型总表（tag 值）

| tag | 常量类型 | 结构要点 | 引入版本 |
|-----|---------|----------|----------|
| 1 | CONSTANT_Utf8 | length + bytes（MUTF-8 编码） | 1.0 |
| 3 | CONSTANT_Integer | u4（补码） | 1.0 |
| 4 | CONSTANT_Float | u4（IEEE754） | 1.0 |
| 5 | CONSTANT_Long | u8，**占 2 项** | 1.0 |
| 6 | CONSTANT_Double | u8，**占 2 项** | 1.0 |
| 7 | CONSTANT_Class | name_index → Utf8（全限定名，斜杠分隔） | 1.0 |
| 8 | CONSTANT_String | string_index → Utf8（运行期解析为 intern 后的 String 引用） | 1.0 |
| 9 | CONSTANT_Fieldref | class_index + name_and_type_index | 1.0 |
| 10 | CONSTANT_Methodref | class_index + name_and_type_index | 1.0 |
| 11 | CONSTANT_InterfaceMethodref | 同上，目标为接口 | 1.0 |
| 12 | CONSTANT_NameAndType | name_index + descriptor_index | 1.0 |
| 15 | CONSTANT_MethodHandle | kind(1~9, 对应 invoke*/new/field) + reference_index | 7 |
| 16 | CONSTANT_MethodType | descriptor_index | 7 |
| 17 | CONSTANT_Dynamic | bootstrap_method_attr_index + name_and_type_index | **11** |
| 18 | CONSTANT_InvokeDynamic | bootstrap_method_attr_index + name_and_type_index | 7 |
| 19 | CONSTANT_Module | name_index | 9 |
| 20 | CONSTANT_Package | name_index | 9 |

## 4.3 常量池是 javac 与 JVM 的"字典"

所有表（字段/方法/属性）里从不直接写名字，只写索引；一条 `invokevirtual` 背后的解析链：

```text
字节码: invokevirtual #4
            │
  #4 = Methodref          #19.#20     // ① 指向 类#19 的 名#20
  #19 = Class             #21         // ② 类名索引
  #21 = Utf8               java/io/PrintStream
  #20 = NameAndType        #22:#23    // ③ 名字+描述符
  #22 = Utf8               println
  #23 = Utf8               (Ljava/lang/String;)V
```

- javac 侧：编译时"写字典"，去重（同名字符串只存一份 Utf8）。
- JVM 侧：链接时"查字典"，把符号引用换成直接引用（见 [3_class_lifecycle](3_class_lifecycle.md) 解析阶段）。

## 4.4 描述符解读规则

对象/方法类型全部压缩成一个字符串，规则背一张表就够：

| 源码类型 | 描述符 | 说明 |
|----------|--------|------|
| `int`/`long`/… 8 种基本类型 | `I` `J` `B` `C` `D` `F` `S` `Z` | 助记：大写首字母；boolean→Z |
| `String` | `Ljava/lang/String;` | **L+斜杠全限定名+分号**，缺一不可 |
| `int[]` | `[I` | 一维一个 `[` |
| `String[][]` | `[[Ljava/lang/String;` | 多维连续多个 `[` |
| `main` 的参数数组 | `[Ljava/lang/String;` | 对象数组同样 `L…;` |

方法描述符 = `(参数列表)返回值`，void 返回写 `V`：

```text
(Ljava/lang/String;)V          void println(String)
(II)I                          int add(int, int)
([Ljava/lang/String;)V         void main(String[])
(I[Ljava/lang/Object;)String   String f(int, Object[])
()V                            void run()   ← <init> 与 <clinit> 均为 ()V
```

- 泛型不体现在描述符里（类型擦除），签名信息另存 `Signature` 属性。
- 内部类名形如 `Outer$Inner`；合成方法名如 `access$100`（编译器生成的私有成员访问桥）。

## 4.5 Hello.class 常量池节选

```text
#1 = Methodref    #6.#15   // java/lang/Object."<init>":()V
#2 = Fieldref     #16.#17  // java/lang/System.out:Ljava/io/PrintStream;
#3 = String       #18      // "Hello JVM"
#4 = Methodref    #19.#20  // java/io/PrintStream.println:(Ljava/lang/String;)V
#5 = Class        #21      // Hello
...
```

# 5. 类索引与访问标志

## 5.1 this_class / super_class / interfaces

- `this_class`、`super_class`、`interfaces[]` 全部是 CONSTANT_Class 的**索引**。
- Java 单继承：`super_class` 只有一个。唯一特例是 `java/lang/Object`，它的 `super_class = 0`（不引用任何常量）。
- 接口列表只是"声明"，真正的方法查找规则（invokeinterface 找实现）在链接与运行期完成。

## 5.2 类级 access_flags

类或接口的修饰符标记：`public`、`final`、`abstract`、`interface`、`annotation`、`enum` 等。

| 标志值 | 含义 | 标志值 | 含义 |
|--------|------|--------|------|
| 0x0001 | ACC_PUBLIC | 0x1000 | ACC_SYNTHETIC（编译器生成，源码无） |
| 0x0010 | ACC_FINAL | 0x2000 | ACC_ANNOTATION |
| 0x0020 | ACC_SUPER | 0x4000 | ACC_ENUM |
| 0x0200 | ACC_INTERFACE | 0x8000 | ACC_MODULE（JDK 9，module-info） |
| 0x0400 | ACC_ABSTRACT | | |

- `public class Hello` → `ACC_PUBLIC | ACC_SUPER` = `0x0021`。
- 注意：`static`/`synchronized` 等类级无意义；字段/方法表复用另一套位值（见 6/7 节）。

## 5.3 ACC_SUPER 的历史包袱

- 早期虚拟机 `invokespecial` 会沿父类链找同名方法（可能调到祖父类），语义有坑；设置 ACC_SUPER 后严格按"直接父类"解析。
- 现代 javac **无条件置位** ACC_SUPER，规范也将其列为强制——读到没有它的老 Class（102 时代的编译器产物）才需要警觉。

# 6. 字段表（Fields）

## 6.1 表结构：count → info[]

- `fields_count`（u2）声明数量，后跟等长 `field_info[]`；**继承来的字段不在本表**（每个类文件只描述自己声明的字段）。

```java
private int count;
static String name;
```

编译后对应两个 `field_info` 结构：

```text
field_info {
    u2 access_flags;      // ACC_PRIVATE(0x0002)
    u2 name_index;        // 指向常量池 "count"
    u2 descriptor_index;  // 指向常量池 "I" (int的表示)
    u2 attributes_count;
    attribute_info attributes[];
}
```

## 6.2 字段级 access_flags

| 标志 | 值 | 标志 | 值 |
|------|----|------|----|
| ACC_PUBLIC | 0x0001 | ACC_FINAL | 0x0010 |
| ACC_PRIVATE | 0x0002 | ACC_VOLATILE | 0x0040 |
| ACC_PROTECTED | 0x0004 | ACC_TRANSIENT | 0x0080 |
| ACC_STATIC | 0x0008 | ACC_SYNTHETIC/ENUM | 0x1000/0x4000 |

- 编译器生成的合成字段（如枚举的 `$VALUES`、内部类引用的 `this$0`）都带 ACC_SYNTHETIC，`javap -p` 才能看到。

## 6.3 字段描述符对应表

| 描述符 | 类型 | 描述符 | 类型 |
|--------|------|--------|------|
| B | byte | J | long |
| C | char | F | float |
| D | double | S | short |
| I | int | Z | boolean |
| [ | 数组，如 `[I` = int[] | L`<classname>`; | 对象，如 `Ljava/lang/String;` |

# 7. 方法表（Methods）

## 7.1 method_info 结构

- 方法表与字段表同构：`flags + name_index + descriptor_index + attributes[]`。
- **方法体代码编译后不在这里，而在属性表中的 `Code` 属性内**；native/abstract 方法没有 Code 属性（只有 Native/抽象标志）。

```text
method_info {
    u2 access_flags;       // ACC_PUBLIC
    u2 name_index;         // 指向 "add"
    u2 descriptor_index;   // 指向 "(II)I"
    u2 attributes_count;
    attribute_info attributes[];
}
```

## 7.2 方法描述符

格式：`(参数描述符)返回值描述符`

```java
public int add(int a, int b) { return a + b; }
```
描述符为：`(II)I` —— 两个 int 参数，返回 int。

- 重载方法描述符不同 → 常量池条目不同 → **JVM 层不存在"重载歧义"**，歧义只属于 javac。
- `<init>` / `<clinit>` 是编译器生成的特殊方法名，反射拿不到，但字节码里就是普通方法表条目。

## 7.3 方法级特有标志

| 标志 | 值 | 说明 |
|------|----|------|
| ACC_SYNCHRONIZED | 0x0020 | 方法级锁（与 Code 内显式 monitorenter 等效，粗粒度） |
| ACC_BRIDGE | 0x0040 | 桥方法：泛型协变/擦除适配时编译器生成 |
| ACC_VARARGS | 0x0080 | 可变参数 T... |
| ACC_NATIVE | 0x0100 | 本地方法，无 Code 属性，走 JNI |
| ACC_ABSTRACT / ACC_STRICT | 0x0400 / 0x0800 | strictfp（基本已无意义） |

# 8. 属性表（Attributes）

## 8.1 属性分布四层级总表

### 8.1.1 属性为什么是唯一扩展点

- 属性表是 Class 文件中扩展性最强的部分，可以出现在 **Class 文件、字段表、方法表、Code 属性**四个层级。
- 新语言特性几乎都靠"新增属性"落地（sealed → PermittedSubclasses、record → Record），老 JVM 读到不认识的属性按 `attribute_length` 整体跳过（见 2.3），文件格式因此多年不用大改。

### 8.1.2 四层级一览

| 层级 | 属性名 | 作用 | 关键版本 |
|------|--------|------|----------|
| Class | SourceFile | 源文件名（异常堆栈用） | 1.0 |
| Class | InnerClasses | 内部类关系表 | 1.1 |
| Class | EnclosingMethod | 匿名类的外围方法 | 1.1 |
| Class | NestHost / NestMembers | 嵌套类访问控制（替代合成 access$） | **11** |
| Class | PermittedSubclasses | sealed 密封类白名单 | **17** |
| Class | Record | 记录类组件表 | **16** |
| Class | Module | module-info 内容 | 9 |
| Class | BootstrapMethods | invokedynamic 的引导方法表 | 7 |
| 字段 | ConstantValue | 编译期常量值直存 | 1.0 |
| 方法 | Exceptions | throws 声明列表 | 1.0 |
| 方法 | Code | **字节码本体** | 1.0 |
| 方法 | MethodParameters | 保留参数名 | 8 |
| Code 内 | StackMapTable | 类型检查证据（校验器用） | 6 |
| Code 内 | LineNumberTable | 字节码↔源码行号 | 1.0 |
| Code 内 | LocalVariableTable / TypeTable | 局部变量名/类型（调试） | 1.0 |
| 三层通用 | Signature | 泛型签名（擦除后找回类型） | 5 |
| 三层通用 | Deprecated | @Deprecated 标记 | 1.0 |

## 8.2 Code 属性逐字段

### 8.2.1 逐字段结构

```text
attribute_info {                       // 方法表中的 Code
  u2 attribute_name_index;             // "Code"
  u4 attribute_length;
  u2 max_stack;                        // 操作数栈深度上限（字节码级"寄存器数"）
  u2 max_locals;                       // 局部变量表容量（含 this/参数；long/double 占2槽）
  u4 code_length;
  u1 code[code_length];                // ★ 字节码
  u2 exception_table_length;
  {   u2 start_pc;                     // 监控区间[起, 止)
      u2 end_pc;
      u2 handler_pc;                   // 跳转到的 catch 块起始
      u2 catch_type;                   // 异常类索引；0 = 匹配一切(finally 语义)
  } exception_table[exception_table_length];
  u2 attributes_count;                 // Code 自己的属性（行号表等）
  attribute_info attributes[attributes_count];
}
```

### 8.2.2 try-catch 与 finally 的实现细节

- try-catch 在字节码里**不是指令**，只是这张四元组表：异常抛出时由 JVM 查表跳 handler——所以"没人 catch 也不影响正常路径性能"。
- JSR 205/旧式 finally 用 `jsr/ret`；JDK 6+ javac 改为**复制 finally 块**到每条出口，异常表膨胀换执行简洁。
- 运行时含义：`max_stack`/`max_locals` 决定栈帧分配大小（见 [8_vm_stack](8_vm_stack.md)）。

## 8.3 javac -g 与调试信息取舍

```bash
javac -g:none Hello.java     # 去掉 LocalVariableTable/LineNumberTable
                             # 效果: 体积更小；代价: 无法看变量名、堆栈无行号
```
- 生产常见折中：保留行号（`-g:lines,source`），方便异常堆栈定位。

## 8.4 BootstrapMethods 与 invokedynamic

```text
字节码: invokedynamic #30, 0
            │
  #30 = InvokeDynamic  0:#31       // 指向 BootstrapMethods[0] 与 NameAndType
  BootstrapMethods 表:
    [0] bootstrap_method_ref = MethodHandle → StringConcatFactory.makeConcatWithConstants
        num_bootstrap_arguments = 1 ...
```

- `invokedynamic` 把"调哪个方法"的决定权**推迟到首次执行**，交给引导方法返回的 CallSite；lambda、字符串 `+` 拼接（JDK 9 起）、动态语言调用全走这条路。
- 详见 [8_vm_stack](8_vm_stack.md) 第 9 节；运行期语义见 [16_execution_engine](16_execution_engine.md)。

# 9. 实战：javap 反编译

## 9.1 常用参数

| 参数 | 作用 |
|------|------|
| `-c` | 反汇编：输出方法内字节码指令 |
| `-p` | 显示 private/protected 成员（默认只 public） |
| `-s` | 打印内部类型签名（描述符） |
| `-v` / `-verbose` | 完整输出：魔数、版本、常量池、全部属性 |
| `-l` | 行号表与局部变量表 |
| `java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly` | 更进一步看 JIT 生成的机器码（需 hsdis） |

```bash
javap -verbose Hello.class          # 查看所有信息
javap -c -p -v Hello.class          # 查看常量池+私有成员+字节码
```

## 9.2 输出关键部分解读

```text
Classfile /path/to/Hello.class
  Last modified 2024-01-01; size 318 bytes
  MD5 checksum ...
  Compiled from "Hello.java"
public class Hello
  minor version: 0
  major version: 61   ← Java 17（用 JDK 21 编译则此处为 65）
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

- `stack=2`：max_stack；`locals=1, args_size=1`：实例方法的 slot0 是 `this`。

## 9.3 逐行解读一个含运算的方法

### 9.3.1 源码与字节码清单

```java
public static int calc() {
    int a = 100;
    int b = 200;
    return a + b;
}
```

```text
0: bipush  100     // 单字节操作数(−128~127)压栈
2: istore_0        // 弹栈 → 局部变量 slot0 (a)
3: sipush  200     // 双字节操作数(−32768~32767)压栈
6: istore_1        // slot1 (b)
7: iload_0         // a 压栈
8: iload_1         // b 压栈
9: iadd            // 弹出两个、和压栈（int 加）
10: istore_2       // 结果存 slot2（JDK 不优化这步，靠 JIT 消掉）
11: iload_2
12: ireturn        // 弹栈作为返回值
```

### 9.3.2 操作数栈随指令的演化（顶在右侧）

```text
bipush 100   [100]
istore_0     []            slots: a=100
sipush 200   [200]
istore_1     []            slots: a=100 b=200
iload_0      [100]
iload_1      [100 200]
iadd         [300]
istore_2     []            slots: a=100 b=200 c=300
iload_2      [300]
ireturn      (方法结束，返回300)
```

## 9.4 压栈指令速记（int 常量怎么进栈最快）

| 指令 | 范围 | 字节数 |
|------|------|--------|
| `iconst_m1`~`iconst_5` | −1~5 | 1 |
| `bipush` | −128~127 | 2 |
| `sipush` | −32768~32767 | 3 |
| `ldc` | 其余 int/float/String 等（走常量池） | 2（另有 `ldc_w`/`ldc2_w`） |
- javac 按"最小指令优先"挑选；`long` 只有 `lconst_0/1`，其余一律 `ldc2_w`。

# 10. Class 文件的加载入口与限制

## 10.1 字节流 → 校验的入口流程

```text
任意来源的 byte[]
   │ ① 魔数核对 (CAFEBABE)
   │ ② major/minor 版本 ≤ 当前 VM 支持上限
   ▼
类加载器 defineClass() → 进入"加载→验证→…"生命周期
   （验证四阶段详见 [3_class_lifecycle](3_class_lifecycle.md) 第 3 节）
```

## 10.2 "Class 文件不一定来自 .class 文件"

### 10.2.1 各来源的典型案例

规范原文：*"represent a class or interface, but are not necessarily limited to being generated by the Java programming language compiler"*。字节流获取方式：

| 来源 | 工具/场景 | 典型用户 |
|------|-----------|----------|
| 磁盘 `.class`/jar | 常规运行 | 所有应用 |
| 编译期多语言 | Kotlin/Scala/Groovy → 合规字节码 | 混合栈 |
| 运行期生成 | **ASM**（字节码框架）、**Javassist**（源码式 API）、**ByteBuddy**（现代封装） | 框架代码织入 |
| 动态代理 | JDK Proxy（$Proxy0 由 ByteWriter 现场生成）/ **CGLIB**（继承目标类生成子类，Spring AOP 旧默认） | AOP、Mock（Mockito） |
| 加密/自定义存储 | 解密后 `defineClass`；数据库读 blob | 类加密、热部署（见 [4_classloader](4_classloader.md)） |
| 网络下发 | Applet（历史）、RMI 按需下载 stub | 遗留系统 |
| 运行期"编译" | JSP → Servlet 类 | Tomcat |
### 10.2.2 安全推论

- 推论：安全扫描不能只盯 classpath 上的 jar——内存里随时可能"长出"新类。

## 10.3 Class 文件的限制

| 限制 | 根源 | 突破思路 |
|------|------|----------|
| 常量池最多约 65535 项 | count 为 u2 | 类拆小；避免海量字符串字面量 |
| 每类最多 65535 个字段 / 方法 | fields/methods count 为 u2 | Android 主 dex 65536 方法数上限同源于此（u2 索引 + 1 保留）→ multidex/R8 裁剪 |
| 单方法字节码 ≤ 65535 字节 | code_length 名义 u4，规范钉死 ≤65535 | 巨型 switch/内联展开失败会报 "code too large" → 拆方法 |
- 这些限制由 Class 文件格式中的 `u2` 类型导致；写代码生成器（ASM）时尤其要当回事。
- 巨型自动生成的类（protobuf/gson TypeAdapter、R.java）是典型触限者。
