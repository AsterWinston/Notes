第 1 篇写出的 `HelloWorld.java` 只有一个文件、没有包、也没有任何依赖，`javac HelloWorld.java` 就能搞定。真实项目远不止于此：类要按**包（package）**分层组织，代码要用 `import` 引用其他包和第三方库，交付时要打成 **jar** 包，运行时还要让 JVM 知道去哪找依赖（**classpath**）。本篇把 **package → import → javac → jar → java** 这条命令行工具链一次性讲通——它正是 Maven、Gradle 和 IDE 的 Build 按钮背后真正在做的事，理解了它，遇到"找不到或无法加载主类""`NoClassDefFoundError`"这类报错才知道该往哪查。

本篇所有命令都以一个贯穿全篇的小项目为例，目录结构如下，后续每一节都在它上面操作：

```text
D:\demo
├── src
│   └── com
│       └── example
│           ├── util
│           │   └── Strings.java          →  package com.example.util;
│           └── app
│               └── MainApp.java          →  package com.example.app;
└── lib                                    →  存放外部 jar
```

# 1. 包（package）：类的组织与命名空间

`package` 是 Java 组织类文件和防止命名冲突的基本机制。它不只是"文件夹"这么简单，而是同时承担**命名空间**、**访问控制边界**和**类加载定位**三重职责。

## 1.1 为什么需要包

**第一重职责是命名空间。** 同一个类名在不同场景下可能都合理——`java.util.Date` 和 `java.sql.Date` 就被 JDK 自己同时用了。如果没有包，整个类路径下的类名必须全局唯一，几万个类放在一起根本没法维护。有了包，类的真正名字是**全限定名（Fully Qualified Name，FQN）**，即"包名 + 类名"，`java.util.Date` 和 `java.sql.Date` 是两个不同的类。

**第二重职责是访问控制。** Java 的四种访问权限中有一个叫**包级私有**（不写任何修饰符，也叫 default）：同一个包内的类可以互相访问包级私有成员，包外不行。这让"一组协作紧密的类共享内部实现、同时对外只暴露少数公开类"成为可能——包成为封装的天然边界。

**第三重职责是类加载定位。** JVM 加载类时要求**包名与目录结构严格对应**：`com.example.util.Strings` 这个类，必须能找到路径为 `com/example/util/Strings.class` 的文件，否则就报 `ClassNotFoundException`。这一条是后面 classpath 全部规则的来源。

## 1.2 package 声明与目录结构

所有非默认包下的类，**必须在源文件的第一条有效语句位置**使用 `package` 声明所属的包：

```java
// 文件位置：src/com/example/util/Strings.java
// 注意：package 声明之前只能有注释和空行，不能有任何其他语句
package com.example.util;

public class Strings {

    /** 判断字符串是否为 null 或空串，调用方不必再写两次判断 */
    public static boolean isNullOrEmpty(String s) {
        return s == null || s.isEmpty();
    }
}
```

这条声明决定了类在文件系统中的**目录结构**：类文件必须位于与包名对应的子目录下。上例中 `package com.example.util;` 对应目录 `com/example/util/`，二者不一致时 `javac` 会报错或产生一个找不到的类。规则可以总结成一句话：**包名用点分层，目录用斜杠分层，两者一一对应。**

```java
// 文件位置：src/com/example/app/MainApp.java
package com.example.app;

import com.example.util.Strings;   // 引用同项目里另一个包中的类

public class MainApp {
    public static void main(String[] args) {
        for (String arg : args) {
            // 这里能直接用 Strings，是因为上面写了 import
            System.out.println("[" + arg + "] 是否为空: " + Strings.isNullOrEmpty(arg));
        }
    }
}
```

## 1.3 包的命名规范

包名有一套业界通行的约定，违反它不会导致编译失败，但会导致冲突和协作问题：

| 规范 | 说明 | 示例 |
|------|------|------|
| **全小写** | 不用 `Com.Example`，避免与类名混淆 | `com.example.util` |
| **域名倒写** | 用组织自有域名倒序，天然保证全局唯一 | `com.alibaba.fastjson`、`org.apache.commons` |
| **按职责分层** | 末级包名体现职责而非技术堆叠 | `com.example.order.service` |
| **不用 Java 前缀** | 禁止以 `java.` 开头 | JDK 会拒绝加载 |
| **避免单个单词的顶级包** | `utils`、`common` 这类顶层包容易撞名 | 至少两级 |

`java.*` 开头是 JDK 保留的命名空间，自己定义会触发安全检查失败（JDK 9 之后由模块系统强封装）。`javax.*` 曾经是扩展包命名空间，Java EE 移交 Eclipse 后相关包名已迁移到 `jakarta.*`——升级 Spring Boot 3 时会大量遇到这个改动（见第 1 篇 1.1）。

## 1.4 默认包（无名包）的陷阱

没有 `package` 声明的类属于**默认包**（也叫无名包）。把类放进默认包能跑起来，但会带来一个致命的割裂：

**默认包中的类，无法被任何有包名的类 `import`。** 这不是约定，而是编译器强制的规则（JLS 明确禁止从有名字的包中导入无名包的类型）。这意味着只要项目里有一个类留在默认包中，它就成了一座孤岛：它可以引用别人，别人引用不了它。

```java
// ✗ 编译报错：无法从有包名的类中引用默认包中的类
package com.example.app;

import Strings;      // 错误：找不到符号

public class MainApp { }
```

> **注意**：IDE 通常会对默认包中的类给出警告，并且很多构建工具在打包时会直接忽略默认包。学习阶段图省事不写包名可以理解，**但任何一旦超过一个文件的项目，第一件事就是把包名补上**。

# 2. import：把类名从"全限定名"缩短

`import` 关键字用于在源代码中引入其他包或 JAR 包中的类、接口，使代码可以直接使用简单类名而非全限定名。它是**纯粹的编译期语法糖**：`.class` 文件里记录的始终是全限定名，`import` 不会出现在字节码中，也不会影响运行期行为、体积或性能。

## 2.1 全限定名与单类型导入

不写 `import` 时，唯一的写法是把类的完整路径写全：

```java
public class Demo {
    public static void main(String[] args) {
        // 不导入就必须写全限定名，冗长但完全合法
        java.util.ArrayList<String> list = new java.util.ArrayList<>();
        java.util.LinkedList<String> queue = new java.util.LinkedList<>();
    }
}
```

用单类型导入把常用类型"提"到当前文件的命名空间，代码立刻清爽：

```java
import java.util.ArrayList;    // 单类型导入：只把 ArrayList 提进来
import java.util.LinkedList;   // 想用几个就写几行

public class Demo {
    public static void main(String[] args) {
        ArrayList<String> list = new ArrayList<>();
        LinkedList<String> queue = new LinkedList<>();
    }
}
```

`import` 语句必须写在 `package` 之后、类声明之前，顺序上**包声明唯一且在最先，导入语句可以有多条**。

## 2.2 按需导入（通配符）

用 `*` 表示"这个包里用到的类型都可以直接写简单名"：

```java
import java.util.*;   // 按需导入：用到哪个才解析哪个

public class Demo {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>();   // 两个类都能直接用
        Map<String, Integer> map = new HashMap<>();
    }
}
```

这里有三个必须澄清的误区：

1. **`*` 不是"把包里的类全部加载进来"**——它是编译期按需解析，用到哪个名字才去查哪个，不会拖慢编译，也不会让 class 变大；
2. **`*` 只覆盖当前包的类，不覆盖子包**：`import java.util.*;` 不会让你用上 `java.util.concurrent.ConcurrentHashMap`；
3. **`*` 的真实代价是可读性**：看代码的人无法从文件头部一眼看出这个类到底依赖了哪些东西，重名类也更容易造成歧义。

三种导入形式的对比：

| 形式 | 写法 | 作用范围 | 推荐度 | 说明 |
|------|------|----------|--------|------|
| 全限定名 | `java.util.Date d = ...` | 单个使用点 | 局部使用 | 只在同名冲突时用，最精确 |
| 单类型导入 | `import java.util.Date;` | 整个文件 | **首选** | IDE 默认行为，依赖清单清晰 |
| 按需导入 | `import java.util.*;` | 整个文件 | 谨慎 | 写起来快，但依赖不透明、易歧义 |
| 静态导入 | `import static java.lang.Math.PI;` | 整个文件 | 特定场景 | 见 2.3 |

## 2.3 静态导入

静态导入可以把某个类的**静态成员**直接引入，调用时省去类名：

```java
import static java.lang.Math.PI;          // 导入单个静态常量
import static java.lang.Math.max;         // 导入单个静态方法
import static java.lang.Math.*;           // 按需导入该类的全部静态成员

public class StaticImportDemo {
    public static void main(String[] args) {
        // 不用写 Math.PI 和 Math.max
        System.out.println("圆周长: " + 2 * PI * 3);
        System.out.println("较大值: " + max(3, 5));
    }
}
```

测试代码里静态导入几乎是标配，因为断言方法全在静态工具类中：

```java
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;

// 调用时直接写 assertEquals(...)，不必写 Assertions.assertEquals(...)
```

> **行业实践**：静态导入要**克制**使用。`import static java.lang.Math.*` 之后，`max(a, b)` 这个方法名看不出任何来源，阅读者需要翻到文件头部才能确认。判断标准很简单：**用静态导入之后，代码的可读性是变好了还是变差了**——`assertEquals`、`isNull` 这类语义自明的断言方法通常变好，而 `abs`、`of`、`copy` 这类通用名词通常变差。

## 2.4 为什么 java.lang 不用导入

`java.lang` 是**唯一一个被编译器隐式按需导入的包**，写不写 `import java.lang.*;` 效果一样。`String`、`System`、`Object`、`Math`、`Integer`、`Thread`、`Exception` 这些使用频率最高的类都在其中——正因为它们太常用，语言规范干脆把导入固化下来，避免每个文件都写一堆重复的 import。

需要注意两点边界：**隐式导入只针对 `java.lang` 本身，不含它的子包**，所以用 `java.lang.reflect.Method` 时仍然要自己导入；另外，如果你自定义了一个与 `java.lang` 中同名的类（比如自己写一个 `String`），编译器会优先使用同一包中的那个类，这是另一个需要警惕的坑。

## 2.5 同名类冲突怎么办

当两个 `import` 把同名类带进来时，编译器无法判断你要哪个，直接报错：

```java
import java.util.Date;
import java.sql.Date;     // ✗ 编译错误：Date 的导入与另一个导入冲突

public class Conflict {
    Date d;               // 编译器不知道这是哪一个 Date
}
```

解决办法是**只导入一个，另一个用全限定名**：

```java
import java.util.Date;

public class Conflict {
    Date utilDate;                    // 默认用导入的那个
    java.sql.Date sqlDate;            // 需要另一个时写全限定名
}
```

`java.util.Date` 与 `java.sql.Date` 的冲突几乎是每个 JDBC 开发者都会遇到的第一个编译错误。IDE 的 "Optimize Imports" 功能做的就是"删掉多余的导入、把冲突项改成全限定名"。

# 3. javac：编译命令详解

`javac` 是 Java 编译器，负责把 `.java` 源代码编译成 JVM 可执行的 `.class` 字节码。这一节讲清最常用的几个参数——尤其是 `-d`，它决定了包结构会不会在输出目录里被正确还原。

## 3.1 基本用法与 -d 参数

最简单的用法是把源文件直接交给 `javac`：

```bash
javac MainApp.java
```

`.class` 文件默认会生成在 **`.java` 文件同级目录**下，**不会**根据 `package` 声明创建目录结构。但 `.class` 文件**内部仍然保留着 package 声明**——这就产生了一个需要小心的错位：

- 如果源文件本身就放在 `src/com/example/app/MainApp.java` 这种"目录与包名一致"的位置，`.class` 落在同目录，恰好也是正确的包结构，运行时能用；
- 如果源文件是"平铺"存放的（`src/MainApp.java` 里写着 `package com.example.app;`），`.class` 就会落在 `src/MainApp.class`，而它的包名却声称自己是 `com.example.app`。JVM 按 `com/example/app/MainApp.class` 去找，永远找不到。

因此**标准做法是使用 `-d <dir>` 指定编译输出的根目录**：

```bash
javac -encoding UTF-8 -d out src/com/example/util/Strings.java src/com/example/app/MainApp.java
```

`-d` 告诉 `javac`：把 `out` 当作**输出根目录**，然后**根据每个类的 `package` 声明自动创建对应的目录结构**并放入 `.class`。编译完成后得到：

```text
out
└── com
    └── example
        ├── app
        │   └── MainApp.class
        └── util
            └── Strings.class
```

这套目录正好就是运行期 classpath 需要的形状——`-d` 的价值就在于此。**只要源码里有 `package` 声明，就应该用 `-d`**，不用的场景仅限于编译一个没有包名的临时测试类。

## 3.2 常用参数

| 参数 | 作用 | 示例 | 说明 |
|------|------|------|------|
| `-d <dir>` | 指定输出根目录，按包名建目录 | `-d out` | 有 package 时必用 |
| `-cp` / `-classpath` | 指定依赖查找路径 | `-cp "lib/a.jar;lib/b.jar"` | 见第 5 节 |
| `-encoding <enc>` | 指定源文件字符编码 | `-encoding UTF-8` | 源文件是 UTF-8 时必须加，否则中文会静默乱码或报"不可映射字符" |
| `--release <N>` | 按指定版本的 API 与字节码编译 | `--release 17` | 比 `-source`/`-target` 更严格，会校验 API 是否属于该版本 |
| `-source` / `-target` | 源码版本 / 目标字节码版本 | `-source 17 -target 17` | 老项目常见，新项目优先 `--release` |
| `-Xlint:all` | 打开全部编译警告 | `-Xlint:all` | 能看到未检查的泛型转换、`try` 资源未关闭等隐患 |
| `-Werror` | 把警告当成错误 | `-Werror` | CI 中常用，防止警告越积越多 |
| `-g` | 生成全部调试信息 | `-g` | 行号、局部变量表，调试必需；`-g:none` 则全部不生成 |
| `-parameters` | 保留方法参数名到字节码 | `-parameters` | Spring、MyBatis 等框架按参数名绑定值时必须加 |
| `-verbose` | 打印编译过程细节 | `-verbose` | 排查"到底加载了哪个类" |
| `-implicit:none` | 不隐式编译被依赖的源文件 | `-implicit:none` | 只想编译指定的类时用 |

> **注意**：`-encoding` 是最容易忽略又最容易造成困惑的参数。Windows 中文环境的默认编码是 GBK，而现代编辑器保存源码默认是 UTF-8，两者不一致时会出现**两种症状**：一种是直接报 `错误: 编码 GBK 的不可映射字符`，报错行往往指向中文注释而非真正的错误位置；另一种更隐蔽——`javac` 不报任何错，但字符串字面量已被错误地重新解码，**中文字符串静默变成乱码**，直到运行时才发现输出不对。另外要注意 **JDK 18 起 `javac` 的默认源文件编码已改为 UTF-8**（JEP 400），而 JDK 8~17 仍沿用系统默认编码。**凡是有中文的源文件，一律显式加上 `-encoding UTF-8`**，跨平台脚本里更是必须加——显式写出编码这件事本身，就消除了对"编译机用什么默认编码"这一前提的依赖。

## 3.3 多文件编译与依赖自动处理

`javac` 的一个重要特性是**自动处理多个源文件之间的依赖关系和 `import`**：不需要你按依赖顺序排列文件，只要都在命令行里交给它即可。

```bash
# 一次性编译所有源文件，顺序任意，javac 自己解析依赖
javac -encoding UTF-8 -d out \
  src/com/example/app/MainApp.java \
  src/com/example/util/Strings.java
```

`javac` 在解析 `MainApp` 时遇到 `com.example.util.Strings`，会先到 classpath 里找 `.class`，找不到就**沿着 sourcepath 找同名的 `.java` 源文件**并顺带编译它——所以有时只编译主类也能成功。但这种隐式编译依赖搜索路径配置，在大型项目里行为不易预测（比如误编译到旧版本的源码）。**稳妥做法是显式列出所有源文件，或者干脆交给构建工具。**

```bash
# 用通配符批量编译（注意：cmd 与 PowerShell 不展开 ** 通配符，需要用下面这种形式）
find src -name "*.java" > sources.txt     # Linux/macOS：生成源文件清单
javac -encoding UTF-8 -d out @sources.txt # @ 前缀表示从文件读取参数
```

```text
REM Windows cmd：用 for /r 递归收集所有 .java 写入清单
for /r src %f in (*.java) do @echo %f >> sources.txt
javac -encoding UTF-8 -d out @sources.txt
```

`@文件` 这个写法值得记住：它可以让参数从文件读取，避免命令行长度超限。

# 4. jar：打包与可执行 jar

`jar` 是 Java 的归档工具，把多个 `.class`、资源文件或目录打包成一个可移植的 `.jar` 文件。它是 Maven 中央仓库里的一切依赖的形态，也是交付一个 Java 程序的基本单位。

## 4.1 jar 包的结构

**jar 本质就是一个 ZIP 文件**，只多了一条约定：根目录下要有 `META-INF/MANIFEST.MF`。内部目录结构就是包结构。

```bash
# 把上一节编译产物打包（-C out . 的用法见 4.3）
jar cf lib/util.jar -C out com/example/util

# 查看 jar 内部结构 —— 这是写 import 之前最该做的一件事
jar tf lib/util.jar
```

```text
META-INF/
META-INF/MANIFEST.MF
com/
com/example/
com/example/util/
com/example/util/Strings.class
```

**`jar tf` 是确定 import 写法的最可靠手段。** 引入外部 jar 时，光看文件名是猜不出包名的；必须用 `jar tf` 看到类在 jar 内的实际路径，才能反推出 import 语句——如果 jar 内部路径是 `com/library/Api.class`，那么正确的写法就是：

```java
import com.library.Api;   // 由 jar tf 看到的 com/library/Api.class 反推而来
```

## 4.2 jar 命令常用参数

`jar` 命令的第一个参数是**必选的操作字母**，后续字母是修饰选项，其中 `f` 几乎总要出现：

```text
jar {c|t|x|u} [vfMm0eC] [jar文件] [文件...]
```

| 参数 | 含义 | 作用 |
|------|------|------|
| `c` | create | 创建新 jar |
| `t` | list | 列出 jar 内容（等价于 `-tf`） |
| `x` | extract | 解压 jar |
| `u` | update | 更新已有 jar 中的文件 |
| `f` | file | 指定 jar 文件名，**必须紧跟操作字母**（如 `cf`） |
| `v` | verbose | 打印处理过程 |
| `C` | change dir | 先切换到指定目录再处理后续文件 |
| `e` | entrypoint | 指定主类，写入 Manifest 的 `Main-Class` |
| `m` | manifest | 使用指定的自定义 Manifest 文件 |
| `M` | no manifest | 不生成默认的 `MANIFEST.MF` |
| `i` | index | 为 jar 生成索引，加快类查找 |
| `0` | no compression | 不压缩存包，体积变大但启动略快 |

四组最常用的命令：

```bash
# 创建（推荐用 -C，见 4.3）
jar cf lib/util.jar -C out .

# 创建并打印过程，适合确认打进包里的路径对不对
jar cvf lib/util.jar -C out .

# 列出内容
jar tf lib/util.jar

# 解压到当前目录（调试"包内结构到底是什么样"时很有用）
jar xf lib/util.jar
```

## 4.3 -C 与直接列文件的区别

这是打包时**最容易出错的地方**：jar 内部保留的路径，取决于你在命令行里怎么描述文件。

**写法一（推荐）：`jar cf <.jar> -C <dir> .`**

`-C <dir>` 表示"先切换到 `<dir>` 目录，再把后面的文件参数相对于这个目录来解释"。写成 `-C out .` 就是"切到 `out`，把 `out` 下的所有内容按其**相对于 `out`** 的路径打进包"。这样 jar 内部的路径从包名开始，干净且正确：

```text
  方式一：jar cf lib/util.jar -C out .

    out/                                  lib/util.jar
     └── com/example/util/                 └── com/example/util/
          └── Strings.class                      └── Strings.class   （路径干净）
```

**写法二（不推荐）：`jar cf <.jar> <file>...`**

直接把文件路径写上去时，这些路径会**按相对于当前目录的完整形式原样带进 jar**。等价地说：它和"用 `-C <当前目录> 罗列出这些文件"是一样的效果——多出来的父目录前缀会被一并打包：

```text
  方式二：jar cf lib/util.jar out/com/example/util/Strings.class

                                        lib/util.jar
                                         └── out/com/example/util/
                                              └── Strings.class   （多了 out/ 前缀）
```

打包出来的 jar 里类的实际路径变成了 `out/com/example/util/Strings.class`，而类的全限定名仍然是 `com.example.util.Strings`——**路径与包名对不上，JVM 永远找不到它**，运行时表现为 `NoClassDefFoundError` 或"找不到或无法加载主类"。用 `jar tf` 一看就能发现多了一层前缀。

> **最佳实践**：**打包带有 package 的类，一律用 `jar cf <包名> -C <输出根目录> .`**。这个写法的另一个好处是输出目录天然就是 `javac -d` 的产物目录，整条链路无缝衔接：`javac -d out ...` → `jar cf x.jar -C out .`。

## 4.4 Manifest 与 Main-Class：打出可执行 jar

`META-INF/MANIFEST.MF` 是 jar 的元信息文件，最常用的属性是**指定主类**。一个手写的 Manifest 长这样：

```text
Manifest-Version: 1.0
Main-Class: com.example.app.MainApp
Class-Path: lib/commons-lang3-3.14.0.jar
```

各行含义：

| 属性 | 作用 | 注意 |
|------|------|------|
| `Manifest-Version` | Manifest 文件格式版本 | 固定写 `1.0` |
| `Main-Class` | 主类的**全限定名** | 不带 `.class` 后缀；决定 `java -jar` 的入口 |
| `Class-Path` | 依赖的 jar 列表 | 多个用**空格**分隔，路径是**相对于本 jar 所在目录**的相对路径，不能用绝对路径分发 |

生成带 `Main-Class` 的 jar 有两种办法。第一种是直接用 `-e` 参数一步到位：

```bash
# c=创建 f=指定文件名 e=指定入口主类，再配合 -C 把 out 的内容打进去
jar cfe app.jar com.example.app.MainApp -C out .
```

第二种是准备一个 Manifest 文件再用 `-m`：

```text
REM 把下面内容存为 manifest.txt，注意文件末尾必须留一个空行
Manifest-Version: 1.0
Main-Class: com.example.app.MainApp
```

```bash
jar cfm app.jar manifest.txt -C out .
```

> **注意**：`MANIFEST.MF` 的最后一行**必须以换行符结尾**。如果最后一行是 `Main-Class: com.example.app.MainApp` 且文件到此为止没有换行，很多工具在解析时**会丢掉最后一行属性**，结果就是 `java -jar` 报 `no main manifest attribute`。这是手工写 Manifest 最经典的坑，用 `-e` 参数则可以完全避开。

## 4.5 运行可执行 jar

```bash
java -jar app.jar hello ""
```

```text
[hello] 是否为空: false
[] 是否为空: true
```

关于 `-jar` 有三条必须记住的规则：

1. **`-jar` 读取的是 Manifest 里的 `Main-Class`**，不能在命令行再指定主类——写成 `java -jar app.jar com.example.app.MainApp` 会把这个类名当成程序参数传进去；
2. **使用 `-jar` 时，`-cp` 参数与 `CLASSPATH` 环境变量都会被忽略**。jar 之外的所有依赖只能通过 Manifest 的 `Class-Path` 属性声明，或者干脆不打成可执行 jar；
3. **一旦 `Main-Class` 指向的类不存在或包名写错，报的是 `Could not find or load main class`**，且不会告诉你具体缺了哪个类。

反过来，如果不想用 `-jar`，就用 `-cp` 显式指定主类，这条路径更灵活（可以自由组合多个 jar 与目录）：

```bash
# Windows 用分号分隔
java -cp "app.jar;lib/util.jar" com.example.app.MainApp hello
```

# 5. classpath：编译期与运行期的依赖查找

classpath（类路径）是 JVM 与 `javac` **查找 `.class` 文件和 jar 包的路径列表**。它是一切"找不到类"报错的根源，也是编译期与运行期行为差异最大的一个参数。

## 5.1 classpath 的构成与默认值

一条 classpath 由若干**类路径项（classpath entry）**组成，每一项可以是：

- 一个**目录**：该目录下必须有完整的包结构（如 `out` 目录里放着 `com/example/app/MainApp.class`）；
- 一个 **jar 文件**：如 `lib/util.jar`；
- 一个 **zip 文件**（历史遗留，极少用）；
- 通配符 `*`：匹配某目录下的所有 jar（见 5.4）。

**默认值只有当前目录 `.`**——也就是说，不带任何 `-cp` 时，JVM 只在当前目录下找 `.class`，**不会自动去任何 jar 里找**。这正是"带 jar 包运行必须写 `-cp`"的原因。

## 5.2 编译期的 -cp

编译时如果代码依赖外部 `.jar` 或不在当前目录下的 `.class`，必须用 `-cp`（或等价的 `-classpath`）告诉 `javac` 它们在哪：

```bash
# 先做一个"外部库" jar 来模拟第三方依赖
javac -encoding UTF-8 -d out-lib src/com/example/util/Strings.java
jar cf lib/util.jar -C out-lib .

# 编译 MainApp 时依赖这个 jar，必须用 -cp 指出来
javac -encoding UTF-8 -cp "lib/util.jar" -d out src/com/example/app/MainApp.java
```

**`-cp` 指向的既可以是 jar，也可以是包含 `.class` 的普通目录**，此时要给的是**根目录**而不是 `.class` 文件的完整路径：

```bash
# ✓ 正确：给包含包结构的根目录
javac -cp "out-lib" -d out src/com/example/app/MainApp.java

# ✗ 错误：给到了 .class 文件本身
javac -cp "out-lib/com/example/util/Strings.class" -d out src/com/example/app/MainApp.java
```

## 5.3 运行期的 -cp：为什么"顺手写一个"往往是错的

运行期的 `-cp` 与编译期有一个**关键差异**：

> **注意**：**一旦使用了 `-cp` 参数，JVM 就不再默认搜索当前目录了。** `-cp` 是**覆盖**默认值，而不是在默认值上追加。因此带 jar 运行时，既要指定 jar 路径，又必须**显式把主类所在的根目录写进去**。

```bash
# ✗ 只指定 jar，主类所在的 out 目录被排除在外，找不到主类
java -cp "lib/util.jar" com.example.app.MainApp
# 报错：错误: 找不到或无法加载主类 com.example.app.MainApp

# ✓ 显式带上 out 目录（Windows 用分号分隔）
java -cp "out;lib/util.jar" com.example.app.MainApp hello ""

# ✓ Linux / macOS 分隔符是冒号
java -cp "out:lib/util.jar" com.example.app.MainApp hello ""

# ✓ 当前目录语义：. 表示"当前目录"，等价于把当前目录放进 classpath
java -cp ".;lib/util.jar" com.example.app.MainApp hello ""
```

另外两条规则同样重要：

- **`<main_class_name>` 必须是全限定名**，即包含 package 路径的类名，如 `com.example.app.MainApp`，并且**不带 `.class` 后缀**；
- **`<CP>` 指的是包含包目录结构的根目录**，而不是 `.class` 文件的具体路径。`java -cp out com.example.app.MainApp` 里，`out` 是根，JVM 会自动去 `out/com/example/app/MainApp.class` 找。

一个完整可复现的例子：假设主类 `com.example.app.MainApp` 的 `.class` 在 `build` 目录下，同时依赖 `lib/util.jar`：

```bash
java -cp "build;lib/util.jar" com.example.app.MainApp
```

## 5.4 查找顺序与通配符

classpath 的查找规则是**从左到右依次查找，命中即停**。这一点带来了一个隐蔽的坑：**同类名的类遮蔽**。

如果 `lib/a-1.0.jar` 和 `lib/a-2.0.jar` 里有同名的 `com.example.Foo`，那么写 `-cp "lib/a-1.0.jar;lib/a-2.0.jar"` 时永远用的是 1.0 版本，2.0 版本形同不存在。这类问题在依赖冲突中很常见，表现往往是"某个方法明明存在却报 `NoSuchMethodError`"。

`-cp "lib/*"` 这种通配符写法可以一次性引入目录下所有 jar，但有两条限制：

1. **通配符只匹配 jar 文件，不匹配 `.class` 目录，也不会递归子目录**。想同时包含目录就必须写 `-cp "out;lib/*"`；
2. **通配符展开后的 jar 顺序不确定**（取决于文件系统的枚举顺序），因此当存在同名类时，结果不可预测。需要确定顺序时，**必须逐个把 jar 列出来**。

排查"类到底是从哪加载的"最有效的手段是：

```bash
# 打印每个类的加载来源，输出形如 [Loaded com.example.util.Strings from file:/D:/demo/lib/util.jar]
java -verbose:class -cp "out;lib/util.jar" com.example.app.MainApp
```

## 5.5 分隔符：Windows 与 Unix 的差异

| 环境 | 类路径分隔符 | 说明 |
|------|--------------|------|
| Windows（cmd / PowerShell） | `;` | `-cp "out;lib/util.jar"` |
| Linux / macOS | `:` | `-cp "out:lib/util.jar"` |
| 路径本身的层级分隔符 | Windows `\`，Unix `/` | jar 内部一律用 `/` |

> **注意**：在 PowerShell 中分号 `;` 是语句分隔符，因此 `-cp` 的参数**必须用双引号整体包起来**，否则 PowerShell 会把 `;` 后面的内容当成新命令执行。cmd 中虽然不会误解析成命令，但带空格的路径同样需要引号。**跨平台脚本里，把 `-cp` 的值整体加引号是唯一稳妥的写法。**

## 5.6 常见报错对照表

| 报错 | 发生时机 | 根因 | 排查方向 |
|------|----------|------|----------|
| `ClassNotFoundException` | 运行期 | 代码**主动按名字**去加载类（`Class.forName`、`ClassLoader.loadClass`），但按名字没找到 | 检查全限定名是否写对、classpath 是否包含该 jar |
| `NoClassDefFoundError` | 运行期 | JVM **隐式加载**（`new`、调用静态方法、父类/接口解析）时找不到类；或该类**上次静态初始化失败** | 检查依赖是否打进包、包名与 jar 内路径是否一致、静态代码块是否抛异常 |
| `NoSuchMethodError` | 运行期 | 类找到了，但方法签名不存在 | **依赖版本冲突**，用 `-verbose:class` 看实际加载了哪个 jar |
| `UnsupportedClassVersionError` | 运行期 | 编译用的 JDK 版本高于运行时的 JDK | 统一版本，或编译时加 `--release` |
| 找不到或无法加载主类 | 运行期 | 主类名不是全限定名、`.class` 不在 classpath 内、jar 内路径被 `out/` 前缀污染 | 见 5.3 与 4.3 |
| `错误: 编码 GBK 的不可映射字符` | 编译期 | 源文件是 UTF-8，`javac` 按系统默认编码解码（也可能是**静默乱码**，不报错） | 加 `-encoding UTF-8` |

**`ClassNotFoundException` 与 `NoClassDefFoundError` 的区别**是高频考点，值得单独说清楚：

- **加载方式不同**：前者由代码**显式**触发（`Class.forName("com.example.Foo")`），后者由 JVM **隐式**触发（写 `new Foo()` 时）；
- **类型不同**：前者是**受检异常** `Exception`，编译器强制你 `try-catch` 或声明；后者是 **`Error`**，属于不可恢复的严重错误，代码里几乎无法优雅处理；
- **成因更广**：`NoClassDefFoundError` 有一种极易被忽略的情形——类**第一次加载时静态代码块抛了异常**，JVM 会把该类标记为"初始化失败"，之后任何再次使用它的代码都会收到 `NoClassDefFoundError: Could not initialize class com.example.Foo`。此时类其实"找得到"，只是永远初始化不了，**真正的错误藏在第一次加载时打印的 `ExceptionInInitializerError` 里**。

```java
// 会触发 ClassNotFoundException：显式按名字加载
Class<?> clazz = Class.forName("com.example.NotExist");   // 必须 try-catch

// 会触发 NoClassDefFoundError：隐式加载，找不到就抛 Error
com.example.NotExist obj = new com.example.NotExist();     // 编译期就报错了，见 5.3
```

> **最佳实践**：现代项目**不应该手写 classpath**。Maven、Gradle 会根据依赖声明自动计算出一长串 jar 路径，IDE 再把它们传给 `javac` 和 `java`。想让交付物自带依赖，就用 Maven Shade / Spring Boot 打成 **fat jar**（把所有依赖解包后合并进一个 jar）；或者用容器镜像把依赖固化进镜像层。手写 `-cp` 的价值在于**理解原理与排查问题**，而不是日常开发手段。

# 6. 模块化（JPMS）简介

JDK 9 引入的 **JPMS（Java Platform Module System，Java 平台模块系统）** 试图在 classpath 之上再加一层更强的组织与约束机制。它在源码根目录放一个 `module-info.java` 来描述模块：

```java
// 文件位置：src/module-info.java
module com.example.app {
    requires java.sql;                    // 依赖 java.sql 模块
    requires transitive com.example.util; // 传递依赖：用到本模块的类型时也可见

    exports com.example.app.api;          // 只有这个包对外可见，其余全部强封装

    // 允许指定框架对本模块的包做反射访问（Jackson、Spring 等需要）
    opens com.example.app.model to com.fasterxml.jackson.databind;

    uses com.example.spi.Plugin;          // 声明使用某个服务接口
}
```

它与 classpath 的核心差异在于：

| 维度 | classpath | 模块路径（module path） |
|------|-----------|------------------------|
| 配置可靠性 | 缺依赖到运行时才报错 | **启动时**就能发现缺失或循环依赖 |
| 封装强度 | 所有 `public` 类都可见 | 只有 `exports` 的包可见，其余强封装 |
| 运行时裁剪 | 只能带完整 JRE | 可用 `jlink` 裁剪出只含所需模块的运行时 |
| 使用命令 | `-cp` | `--module-path`、`--module`、`--add-opens` |

> **行业实践**：模块化推出多年，**主流业务项目仍然以 classpath + 构建工具为主**，很多 Maven 项目根本不写 `module-info.java`。原因是生态迁移成本高——大量第三方库依赖反射访问 JDK 内部 API，一上模块路径就要加一堆 `--add-opens`。但它带来的两个副产品已经在被广泛使用：一是**强封装**让 `sun.*` 内部 API 不能再随便调用（JDK 17 起默认拒绝），二是**`jlink`/`jpackage`** 能把运行时裁剪到几十 MB，是容器镜像瘦身和桌面应用打包的利器。想深入时，方向是 `module-info.java` 语法、`jlink` 自定义运行时、`jpackage` 生成安装包，以及 `--add-opens`/`--add-exports` 的迁移实践。

# 7. 小结

| 主题 | 一句话要点 |
|------|-----------|
| package | 三重职责：命名空间、包级私有访问边界、类加载定位；`package` 声明必须在文件第一条有效语句，且与目录结构严格对应 |
| 命名规范 | 全小写、域名倒写、分层反映职责；不能用 `java.` 开头；**永远不要用默认包** |
| import | 只是编译期语法糖，字节码里始终是全限定名；首选单类型导入，慎用按需导入与静态导入 |
| java.lang | 唯一被编译器隐式按需导入的包；其子包（如 `java.lang.reflect`）仍需显式导入 |
| javac | 有 `package` 就必须用 `-d <dir>`，它会按包名自动建目录；有中文源码必须加 `-encoding UTF-8` |
| jar | 本质是 zip + `META-INF/MANIFEST.MF`；打包**一律用 `jar cf x.jar -C <输出根目录> .`**，直接列文件会带进多余路径前缀 |
| 可执行 jar | Manifest 写 `Main-Class`（全限定名，文件末尾要留换行）；用 `jar cfe` 可一步生成；**`java -jar` 时 `-cp` 与 `CLASSPATH` 全部被忽略** |
| classpath | 编译期与运行期都要 `-cp`；**一旦写了 `-cp`，默认的当前目录就失效**，必须把主类所在根目录显式加进去 |
| 查找规则 | 从左到右命中即停 → 会产生类遮蔽；`lib/*` 只匹配 jar、不递归、顺序不定 |
| 报错区分 | `ClassNotFoundException` 是显式加载的受检异常；`NoClassDefFoundError` 是隐式加载的 `Error`，也可能是**静态初始化失败**导致的 |
| 模块化 | JPMS 提供可靠的配置、强封装与 `jlink` 裁剪；业务项目主流仍是 classpath，但强封装与 `jlink` 已在广泛使用 |

下一篇（第 3 篇）开始进入 Java 语法正题，从**算术运算符**讲起——有了本篇的工具链，后面每个例子都可以直接 `javac -d out` 编译、`java -cp out` 运行，亲手验证一遍比只看结论有效得多。
