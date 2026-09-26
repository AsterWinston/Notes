# 1. jdk安装

```
Java Development Kit (JDK) 是 Java 语言的核心组件，包含了开发、编译和运行 Java 程序所需的一切工具。

下载： 访问 Oracle 官网或 OpenJDK 官网，根据操作系统（Windows, macOS, Linux）选择对应版本进行下载。
安装： 运行安装程序（如 .exe 或 .dmg），推荐将 JDK 安装在一个路径不含中文或空格的目录下（例如：C:\Java\jdk-17）。
注意： 安装过程中通常会提示安装 JRE，你可以选择安装。
```

# 2. 环境变量

```
正确配置环境变量是确保 Java 命令能在任何目录下运行的关键步骤。主要需要配置以下三个变量：

JAVA_HOME：
    目的： 指向 JDK 的安装根目录。许多 Java 相关的工具（如 Tomcat、Maven）会依赖此变量来查找 Java 环境。
    值： C:\Java\jdk-17 (根据实际安装路径填写)。
PATH：
    目的： 允许系统在任何目录下找到 JDK 的可执行文件（如 javac.exe 和 java.exe）。
    配置： 在现有的 PATH 变量后追加 %JAVA_HOME%\bin。
CLASSPATH (可选，现代开发中已不常用)：
    目的： 指定 JVM 搜索 .class 文件和库文件的路径。
    配置： 通常设置为 .;%JAVA_HOME%\lib\dt.jar;%JAVA_HOME%\lib\tools.jar，其中 . 代表当前目录。
```

# 3. notepad++

```
Notepad++ 是一个轻量级的开源文本编辑器，常被用于编写基础的 Java 源代码（.java 文件）。

特点：
    启动速度快。
    支持代码高亮显示，能够识别 Java 语法。
    适合初学者快速编写和修改代码。
注意： 虽然适合编写代码，但大型项目开发通常需要使用功能更强大的集成开发环境 (IDE)，如 IntelliJ IDEA 或 Eclipse。
```

# 4. java分类

```
Java 技术主要被划分为三个核心平台，以应对不同的应用场景：
```

## 4.1 javase (Java Standard Edition)

```
定位： Java 标准版，是 Java 技术的核心和基础。
用途： 用于开发桌面应用程序、控制台程序、以及 Applet (历史遗留)。包含了 Java 语言的基础语法、核心类库（如 java.lang, java.util）和工具。
```

## 4.2 javame (Java Micro Edition)

```
定位： Java 微型版，是针对资源受限的设备（如早期的手机、PDA、机顶盒等）设计的。
现状： 随着 Android 和 iOS 的兴起，Java ME 平台已基本被淘汰。
```

## 4.3 javaee (Java Enterprise Edition)

```
定位： Java 企业版，是为开发大型的、分布式的、多层的企业级应用（如 Web 服务、大型电商平台）而设计的。
核心技术： Servlet、JSP、EJB、JMS、JPA 等。
现状： 2017 年后，Java EE 被移交给 Eclipse 基金会，并更名为 Jakarta EE。
```

# 5. java特点

```
Java 语言因其独特的设计哲学，拥有以下几个显著优势：
```

## 5.1 面向对象

```
Java 是一种纯粹的面向对象语言。它支持面向对象的三大基本特性：封装 (Encapsulation)、继承 (Inheritance) 和 多态 (Polymorphism)。
这使得 Java 程序结构清晰，易于扩展和维护。
```

## 5.2 安全性

```
无指针： Java 不支持直接访问内存的指针，从根本上杜绝了内存越界和非法访问的问题。
字节码校验器： JVM 在执行前会检查字节码，确保其符合 Java 规范，防止恶意代码破坏系统。
安全沙箱： 限制了程序对系统资源的访问权限。
```

## 5.3 多线程

```
Java 语言内置了对多线程的支持，提供了一套完整的线程同步和管理机制（如 Thread 类、synchronized 关键字）。
这使得 Java 在开发高并发、高性能的服务器应用时具有天然的优势。
```

## 5.4 简单

```
相比于 C++，Java 移除了复杂的特性，如运算符重载、多重继承（通过接口实现），尤其是移除了手动内存管理（指针）。
垃圾回收 (GC)： JVM 自动管理内存的分配和回收，大大减轻了开发者的负担。
```

## 5.5 开源

```
OpenJDK (Open Java Development Kit) 是 Java SE 的官方开源实现，被广泛采纳和使用。
这确保了 Java 技术的透明性、社区的活力以及持续的创新。
```

## 5.6 跨平台

```
一次编写，到处运行 (Write Once, Run Anywhere, WORA)： Java 程序编译后生成的是平台无关的字节码（.class 文件）。
只要目标系统安装了对应的 Java 虚拟机 (JVM)，这段字节码就可以在该系统上运行，无需重新编译。
```

# 6. jre（java runtime environment）和jdk（java development kit）

```
JDK 和 JRE 是 Java 生态中两个核心但又相互关联的概念。
```

## 6.1 jdk

```
定义： Java 开发工具包，是提供给 Java 开发者使用的工具包。
功能： 包含 JRE 的所有功能，以及额外的开发工具。
```

### 6.1.1 jvm虚拟机

```
包含： JVM 是 JDK 的一部分，用于执行 Java 字节码。
```

### 6.1.2 核心类库

```
包含： 包含了所有 Java 标准 API，供开发者在编写程序时调用。
```

### 6.1.3 开发工具

```
包含： 诸如 javac（编译器）、javadoc（文档生成器）、jdb（调试工具）等，用于代码的创建、编译和调试。
```

## 6.2 jre

```
定义： Java 运行时环境，是提供给 Java 用户使用的运行环境。
功能： 包含运行 Java 应用程序所需的一切组件，但不包含开发工具。
```

### 6.2.1 jvm

```
包含： JVM 是 JRE 的核心，负责加载、验证和执行 Java 字节码。
```

### 6.2.2 核心类库

```
包含： 仅包含运行 Java 程序所需的标准类库的子集。
```

### 6.2.3 运行工具

```
包含： 主要包含 java 启动器（launcher），用于执行 .class 文件。
```

# 7. helloworld.java

```
这是一个经典的 Java 程序示例，用于演示 Java 程序的编译和运行过程。

假设你已经创建了以下文件：

Helloworld.java
public class Helloworld {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}
```

## 7.1 javac helloworld.java

```
动作： 编译 (Compile) 源代码。
命令： javac Helloworld.java
结果： 如果没有语法错误，javac 编译器会将 Helloworld.java 文件转换为机器可执行的 字节码文件：Helloworld.class。
```

## 7.2 java helloworld//不需要加.class

```
动作： 运行 (Execute) 字节码。
命令： java Helloworld
结果： JVM 启动，加载并执行 Helloworld.class 文件中的字节码，最终在控制台输出 Hello, World!。注意：运行时不需要带 .class 后缀。
```
