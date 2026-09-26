# 1.编写java类，package声明

```
package 关键字是 Java 组织类文件和防止命名冲突的基本机制。

所有非默认包下的类文件，必须在源代码文件的第一行使用 package 关键字声明其所属的包名（例如：`package com.example.util;`）。

这个声明决定了该类在文件系统中的**目录结构**：类文件必须位于与包名对应的子目录下（例如：`com/example/util/`）。
```

# 2.javac编译

```
javac 是 Java 编译器，负责将 .java 源代码文件编译成 JVM 可执行的 .class 字节码文件。

（使用-d <dir>参数编译时，就会受到package的影响了，会自动在指定的dir下创建package指定的目录存放.class文件，如果不使用-d <dir>那默认放在.java同级目录下，且不受package指定目录影响）

-d <dir> 参数： 用于指定编译输出的根目录。当源代码中包含 package 声明时，javac 会在该目录下自动创建包名对应的目录结构，并将 .class 文件放入其中。这是标准项目编译的推荐做法。

无 -d 参数： .class 文件会直接生成在 .java 文件的同级目录下，但类文件内部仍然保留 package 声明。这通常只用于简单的测试。
```

# 3.jar打包

```
jar 是 Java 的归档工具，用于将多个 .class 文件、资源文件或目录打包成一个可移植的 .jar 文件。

（.jar包里的路径收到jar打包命令路径影响，如果使用jar cf <.jar> -C <dir>，那么只会将<dir>下的目录信息放入jar包中，但是如果使用jar cf <.jar> <file>[ <file> <file> ...]，那么file所带的所有路径都会放入jar包中，相当于是-C <dir>在dir目录下罗列出相对于父路径的所有.class文件，然后带着相对路径打包成jar）

jar cf <.jar> -C <dir>： 这是推荐的打包方式。-C <dir> 会将 <dir> 视为根目录，只将 <dir> 内部的文件和目录结构（即包结构）打包进去，确保 JAR 内部的路径是干净的（例如：从 com/example/ 开始）。

jar cf <.jar> <file>...： 这种方式会将 <file> 甚至其父目录的结构都原样带入 JAR 包，可能导致 JAR 内部路径冗余或错误，不推荐用于打包带有 package 的类。
```

# 4.import声明

```
import 关键字用于在 Java 源代码中引入其他包或 JAR 包中的类或接口，使代码可以直接使用类名而非全限定名（Fully Qualified Name）。

编写代码使用:注意jar包结构，使用jar tf <.jar>，这决定了如何使用import

当使用外部 JAR 包时，需要使用 jar tf <.jar> 命令来查看 JAR 包内部的结构，以确定正确的 package 路径，从而编写正确的 import 语句。例如，如果 JAR 内部的类路径是 `com/library/Api.class`，则 import 声明应为 `import com.library.Api;`。
```

# 5.javac带jar包编译

```
编译时，如果您的代码依赖于外部的 .jar 包或非当前目录下的 .class 文件，你需要使用 -cp (classpath) 参数告诉 javac 它们在哪里。

javac -cp <.jar>[;<.jar>;...] <.java>[ <.java> <.java>]（注意ps对带有;命令的解析异常，所以用ps时需要用""将带;连接的参数引起来，下面的java运行也一样）（不只是.jar可以用-cp指定，.class文件也可以用-cp指定.class，只不过指定的是.class路径而不是.class）（多个.java同时编译，javac会自动处理依赖关系）（javac编译会自动处理import）

-cp 参数： 用于指定编译器查找依赖类的路径。路径可以是 .jar 文件，也可以是包含 .class 文件根目录的文件夹。

Windows/PowerShell 注意事项： 在 Windows 命令行或 PowerShell 中，路径分隔符是分号 (;) ，如果路径中包含多个依赖项，最好用双引号将整个 -cp 参数引起来，以避免命令行解析错误。

自动处理： javac 编译器非常智能，如果同时编译多个 .java 文件，它会自动处理它们之间的依赖关系和 import 语句。
```

# 6.java带包运行

```
java 命令用于启动 JVM 并执行已编译的字节码文件。当运行带有 package 声明的类时，必须正确使用 -cp 参数。

java -cp <CP>[;<CP>;...];<.jar>[;<.jar>;...] <main_class_name>（注意<CP>指的是除了jar包外涉及的所有.class文件所在路径，注意是路径而非.class完整路径，<main_class_name>指的是包含main入口函数的.class文件名，不要带后缀）（java不使用-cp时默认在当前路径下搜索所有.class而不会搜索.jar，所以带jar包运行必须用-cp指定，且使用了-cp后，java不会默认在当前目录下搜索所有.class了，所以带jar包运行，不仅要指定jar包路径，还要指定所有.class路径，包括主类）这个

<main_class_name>： 必须是类的**全限定名**（Fully Qualified Name），即包含 package 路径的类名（例如：`com.example.MainApp`）。

-cp 运行时的特殊性：

1.  当使用了 -cp 参数后，JVM 将不再默认在当前目录 ( . ) 下搜索 .class 文件。
2.  因此，除了指定的外部 .jar 包路径外，你还必须显式地在 -cp 中包含包含主类（main class）以及所有其他自定义 .class 文件的根目录（通常用 . 或实际目录名表示）。
3.  <CP> 指的是包含 package 目录结构的**根目录**，而不是 .class 文件的具体路径。

例如：要运行 package 为 `com.my.app` 的主类 `Main`，且依赖 `lib.jar`，如果 `.class` 文件位于 `build` 目录下，命令可能是：`java -cp build;lib.jar com.my.app.Main`。
```