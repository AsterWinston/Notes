本篇是 JavaSE 系列的开篇，讲清三件事：**Java 技术体系由哪些部分组成、开发环境怎么安装和配置、第一个程序从源码到运行经历了什么**。很多人急着跳过这一篇直接学语法，结果在后续每一篇的编译运行环节反复卡在 "`javac` 不是内部或外部命令"、"找不到或无法加载主类"、"`NoClassDefFoundError`" 这类本可以一次性解决的问题上。JDK、JRE、JVM 的层次关系中藏着**跨平台**的全部秘密，而环境变量与编码配置则决定了后面每一条 `javac`、`java`、`jar` 命令能不能跑通；第 2 篇会在此基础上讲代码的组织方式（package、import 与 jar 打包）。

# 1. Java 技术体系与版本演进

Java 从来不只是"一门语言"，而是一个由**语言规范 + 虚拟机规范 + 标准类库 + 工具链**四部分组成的完整平台。同一份 Java 源码能编译成同一份字节码，而字节码由各平台的 JVM 翻译成本地机器码——这种"语言与实现分离"的架构，是它能在企业级市场存活三十年的根本原因。

## 1.1 三大平台：SE、ME 与 EE

Java 在历史上按**目标设备的资源规模**划分出三个平台，各自解决不同场景的问题。

**Java SE（Standard Edition，标准版）** 是 Java 技术的核心和基础，包含语言基础语法、核心类库（`java.lang`、`java.util`、`java.io`、`java.net`）以及编译运行工具。它用于开发桌面程序、控制台程序和基础服务端程序。**本系列讲的 30 篇内容，全部属于 Java SE 的范畴。**

**Java ME（Micro Edition，微型版）** 是面向资源受限设备（早期手机、PDA、机顶盒）的精简版，核心技术是 CLDC/MIDP。它曾在功能机时代占据主流，但随着 Android 和 iOS 生态的崛起，Java ME 已基本退出历史舞台，今天只需要知道它存在过即可。

**Java EE（Enterprise Edition，企业版）** 是为大型分布式、多层企业级应用（Web 服务、电商平台、金融系统）设计的，在 SE 的基础上增加了 Servlet、JSP、EJB、JMS、JPA 等一整套服务端规范。2017 年 Oracle 把 Java EE 移交给 **Eclipse 基金会**，由于 Oracle 保留了 "Java" 商标，该平台更名为 **Jakarta EE**，包名也从 `javax.*` 迁移到 `jakarta.*`——这是从 Spring Boot 2 升级到 Spring Boot 3 时必须面对的最大破坏性变更之一。

| 平台 | 定位 | 代表技术 | 现状 |
|------|------|----------|------|
| Java SE | 标准版，语言与基础类库 | `java.lang`、集合、IO、多线程、网络、反射 | 一切的基础，本系列主线 |
| Java ME | 微型版，资源受限设备 | MIDP、CLDC | 已被 Android / iOS 取代 |
| Java EE | 企业版，分布式服务端 | Servlet、JSP、EJB、JMS、JPA | 2017 年移交 Eclipse，更名 Jakarta EE |

> **设计考量**：三分法的本质是按**运行时资源**做切分——手机内存以 KB 计，服务器内存以 GB 计，同一套语言无法同时迁就两端。后来 Android 用另一条路线（自研 ART 虚拟机 + 精简类库）解决了移动端问题，Java ME 失去了存在意义；而 Java EE 的规范化对象是"服务端能力"，它没有消失，只是换了名字并继续演进。

## 1.2 版本演进与 LTS 版本选择

Java 的版本历史上有两个关键转折点。第一次是 **2006 年 OpenJDK 开源**，Sun 把 Java SE 的实现以 GPL 协议开放，从此存在商业版 Oracle JDK 与开源版 OpenJDK 两条并行的实现路线；2010 年 Oracle 收购 Sun 后，OpenJDK 成为事实上所有下游 JDK 发行版（包括 Oracle JDK 自身）的公共代码库。

第二次是 **2017 年的 JDK 9**。它带来了模块化系统（JPMS），同时把发布节奏从"憋几年发一个大版本"改成**每 6 个月一个特性版本**，并约定**每 2 年指定一个 LTS（Long-Term Support，长期支持）版本**，LTS 获得多年的安全更新与商业支持。这个改变让 Java 的迭代速度显著加快，但也带来了"版本选择"这个新问题。

| 版本 | 发布时间 | 是否 LTS | 代表特性 | 现状与建议 |
|------|----------|----------|----------|-----------|
| JDK 8 | 2014-03 | 是 | Lambda、Stream、新日期时间 API（`java.time`）、`Optional`、接口默认方法 | 存量项目占比最高，但已停止免费商用更新；新项目不建议 |
| JDK 11 | 2018-09 | 是 | 标准化 HTTP Client、`String` 新方法（`isBlank`/`strip`/`repeat`）、移除 Java EE 与 CORBA 模块 | 很多中间件的最低支持版本，属于"保底"选择 |
| JDK 17 | 2021-09 | 是 | 密封类（`sealed`）、`instanceof` 模式匹配、文本块（JDK 15 转正）、强封装 JDK 内部 API | 近年企业升级的主流落点，生态兼容性最好 |
| JDK 21 | 2023-09 | 是 | **虚拟线程**、记录模式、`switch` 模式匹配、分代 ZGC、`SequencedCollection` | 新项目首选；虚拟线程对高并发 IO 服务收益明显 |
| JDK 25 | 2025-09 | 是 | 在 21 的能力上继续演进，并转正一批早期预览特性 | 下一代 LTS，升级前需评估依赖链兼容性 |

> **行业实践**：**选版本只选 LTS**。非 LTS 版本只有 6 个月的支持窗口，下个版本一发布就停止更新，绝不应用于生产。落地时通常是"**新项目直接用当前最新 LTS，老项目从 8 升到 17 或 21**"，而不是逐个版本爬楼梯——中间版本没有升级价值。

JDK 21 引入的**虚拟线程**（Virtual Threads）、**记录模式**、**`switch` 模式匹配**、**分代 ZGC** 等新特性，在第 29 篇《新特性》中有逐条展开，本篇只需要知道"版本号背后对应着不同的语言能力"这一层即可。

# 2. JDK、JRE 与 JVM 的关系

初学者最容易混淆的三个缩写，其实是一条清晰的包含链：**JDK ⊃ JRE ⊃ JVM**。分清楚它们的边界，才能回答"服务器上到底该装什么""为什么我只装了 JDK 却没有 `javac` 命令"这类问题。

## 2.1 JVM：执行字节码的虚拟机

**JVM（Java Virtual Machine，Java 虚拟机）** 是 Java 程序的运行载体，负责**加载、验证并执行字节码**。它是一份规范，不是一份具体软件——任何人只要实现了《Java 虚拟机规范》定义的指令集、类文件格式和运行时数据区，都可以叫 JVM。HotSpot（Oracle/OpenJDK 默认实现）、OpenJ9（IBM 系）、GraalVM 都是不同的 JVM 实现。

JVM 屏蔽了操作系统的差异：字节码里没有"Windows 系统调用"或"Linux 系统调用"，只有"把两个整数相加""调用某个方法"这类抽象指令，由各平台的 JVM 负责翻译成对应的本地机器码。**这就是跨平台的全部秘密。**

## 2.2 JRE：运行 Java 程序的最小集合

**JRE（Java Runtime Environment，Java 运行时环境）** 是提供给"只是要运行 Java 程序"的用户的环境，包含三部分：

- **JVM**：JRE 的核心，负责加载、验证、执行字节码；
- **核心类库**：程序里 `import` 的那些标准 API 的实现，包含运行程序所需的标准类库子集；
- **运行工具**：主要是 `java` 启动器（launcher），也就是我们敲的那条 `java HelloWorld` 命令。

JRE 里**没有** `javac`。这就是为什么只装 JRE 的机器能跑 `.class` 和 `.jar`，却编译不了 `.java`。

## 2.3 JDK：开发者的完整工具包

**JDK（Java Development Kit，Java 开发工具包）** 是给开发者用的，**包含 JRE 的全部内容，再加上一批开发工具**：

- `javac`：编译器，把 `.java` 编译成 `.class`；
- `javadoc`：从文档注释生成 HTML API 文档；
- `javap`：反汇编器，能把 `.class` 还原成可读的字节码指令，是理解"编译后到底变成了什么"的最佳工具；
- `jdb`：命令行调试器；
- `jar`：打包工具，把多个 `.class` 和资源文件压成一个 `.jar`；
- `jshell`：JDK 9 引入的交互式 REPL，适合验证一小段语法；
- `jps`、`jstack`、`jmap`：JDK 自带的诊断工具，生产环境排查问题基本靠它们。

## 2.4 三者的包含关系

```text
┌─────────────────────────────────────────────────────────────┐
│ JDK（开发工具包）                                            │
│   ├── 开发工具：javac、javadoc、javap、jdb、jar、jshell      │
│   └── JRE（运行时环境）                                      │
│        ├── 运行工具：java 启动器（launcher）                  │
│        ├── 核心类库：java.lang、java.util、java.io ...        │
│        └── JVM（虚拟机）                                     │
│             ├── 类加载子系统                                  │
│             ├── 字节码执行引擎（解释器 + JIT 即时编译器）      │
│             └── 运行时数据区（堆、栈、方法区、程序计数器）      │
└─────────────────────────────────────────────────────────────┘
```

顺着这张图可以推出两条实用结论：**装了 JDK 就一定能运行**（因为包含 JRE）；**只装 JRE 则不能编译**（因为没有 `javac`）。因此现代 Java 环境安装时**一律装 JDK**，不必纠结。

## 2.5 为什么 JDK 9 之后没有独立的 JRE 目录了

用 JDK 8 的人会记得安装目录下有个 `jre` 子目录，而 JDK 9 之后它消失了，只剩 `bin`、`lib`、`conf`、`jmods` 等。原因还是**模块化**：在 JDK 8 中，`jre` 目录是"核心类库的另一份拷贝"，`tools.jar`（开发工具的实现）和 `rt.jar`（核心类库的实现）互相割裂，同一份代码存在两处。

JDK 9 之后，整个 JDK 被拆成约 70 个模块（`java.base`、`java.sql`、`java.desktop` ...），`java` 启动器会根据程序实际用到的模块**自动组装**运行时镜像。因此不再需要一份静态的 `jre` 目录——真想要一个精简运行时，可以用 `jlink` 按需生成，产物往往只有完整 JRE 的几分之一，这对容器镜像瘦身非常有用。

> **注意**：JDK 8 时代配置 `CLASSPATH` 常写的 `%JAVA_HOME%\lib\dt.jar` 和 `tools.jar` 在 JDK 9 之后**已经不存在了**，照抄老教程会得到一个指向不存在文件的 classpath，这也是"现代开发不推荐手动设置 `CLASSPATH`"的原因之一。

## 2.6 该装哪一个

| 你的目标 | 需要安装 | 说明 |
|----------|----------|------|
| 学习或开发 Java | JDK | 必须有 `javac`、`jar`、调试工具 |
| 只运行别人给的程序 | JRE 即可 | 但装 JDK 更省事，且不占多少空间 |
| 服务器部署 | JDK 或 `jlink` 生成的自包含运行时 | 容器镜像建议用 `jlink`/`jpackage` 瘦身 |
| 想看字节码、调 GC | JDK | `javap`、`jstat`、`jmap` 都在 JDK 里 |

# 3. 安装 JDK 与配置环境变量

安装本身是点几下"下一步"的事，真正容易出错的是**环境变量**。这一节给出可直接照做的配置步骤，以及每一步"为什么需要它"的解释。

## 3.1 下载与安装

**下载来源**上，Oracle JDK 与 OpenJDK 各发行版都可选。Oracle JDK 从 JDK 17 起采用 NFTC 许可，商用需要付费订阅；而 OpenJDK 的各家发行版在许可上更宽松，功能上又几乎完全等价（因为它们都基于同一份 OpenJDK 源码构建）。

| 发行版 | 维护方 | 特点 |
|--------|--------|------|
| Oracle JDK | Oracle | 官方实现，商用收费，包含少量商业特性 |
| Eclipse Temurin | Eclipse Adoptium | 社区最常用的免费构建，兼容性验证完善 |
| Amazon Corretto | Amazon | 免费，AWS 环境适配好，长期提供安全补丁 |
| Azul Zulu | Azul | 版本覆盖最全，含 Zulu Prime 低延迟 GC |
| Microsoft Build of OpenJDK | 微软 | 与 Windows / Azure 集成好 |
| Dragonwell | 阿里巴巴 | 针对大规模服务端场景优化，国内镜像快 |

**安装建议**：把 JDK 装在一个**路径不含中文和空格**的目录下，例如 `C:\Java\jdk-17`。这不是形式主义——很多构建脚本和工具链在处理路径时不做转义，`C:\Program Files\...` 里的空格会把参数切断，`C:\用户\张三\...` 里的中文则可能因为编码不一致变成乱码，最终表现为各种莫名其妙的"文件找不到"。

安装程序在旧版本（如 JDK 8）中会额外询问是否安装 **公共 JRE**，可以照装；JDK 9 之后不再有这个提示，因为已经没有独立 JRE 的概念（见 2.5）。

## 3.2 三个环境变量

环境变量的作用是**让命令在任何目录下都能找到 JDK，并让其他工具能定位到 JDK 的安装位置**。历史上需要配三个，今天只需要两个。

| 变量 | 作用 | 是否必配 | 典型值 |
|------|------|----------|--------|
| `JAVA_HOME` | 指向 JDK 安装根目录，供 Tomcat、Maven、Gradle 等工具查找 Java 环境 | **必配** | `C:\Java\jdk-17` |
| `PATH` | 让系统在任何目录下都能找到 `javac.exe`、`java.exe` 等可执行文件 | **必配** | 追加 `%JAVA_HOME%\bin` |
| `CLASSPATH` | 指定 JVM 搜索 `.class` 与类库的路径 | **不建议配置** | 见下方说明 |

关于 `CLASSPATH`，老教程常写：

```text
.;%JAVA_HOME%\lib\dt.jar;%JAVA_HOME%\lib\tools.jar
```

其中 `.` 代表当前目录。**但今天不应该这样配**，理由有三条：

1. `dt.jar` 与 `tools.jar` 在 JDK 9 之后已被移除，配了就是指向不存在的路径；
2. 不设置 `CLASSPATH` 时，JVM 的默认值就是当前目录 `.`，已经能满足大部分需求；
3. 一旦手动设置 `CLASSPATH`，就会**覆盖默认值**——如果漏写那个 `.`，编译和运行时会连自己写的类都找不到，报出令人困惑的 `ClassNotFoundException`。

> **注意**：`JAVA_HOME` 的值只能是**目录本身**，不能带 `\bin`、不能带结尾的分号、一般也不要加引号。写成 `C:\Java\jdk-17\bin` 或 `"C:\Java\jdk-17"` 会让 Maven、Tomcat 在拼接路径时得到错误结果，而 `java -version` 却可能因为 `PATH` 单独配对了而正常——这类"一部分命令正常、一部分诡异报错"的现象，多半个 `JAVA_HOME` 的锅。

## 3.3 Windows 上的配置步骤

图形界面方式（推荐，便于检查）：

1. 打开"此电脑"→ 右键"属性"→"高级系统设置"→"环境变量"；
2. 在"系统变量"中点击"新建"，变量名 `JAVA_HOME`，变量值 `C:\Java\jdk-17`；
3. 找到已有的 `Path`，编辑 → 新建一行 → 填入 `%JAVA_HOME%\bin` → 把它**上移到最顶部**；
4. 不要新建 `CLASSPATH`；
5. **关闭并重新打开**终端（环境变量只在进程启动时读取一次）。

命令行方式（适合批量部署或脚本化）：

```text
REM 设置 JAVA_HOME，/M 表示写入系统变量而非当前用户变量
setx JAVA_HOME "C:\Java\jdk-17" /M

REM 把 %JAVA_HOME%\bin 追加到 PATH 末尾；%PATH% 会被展开成当前 PATH 的实际内容
setx PATH "%PATH%;%JAVA_HOME%\bin" /M
```

> **注意**：`setx` 把 `%PATH%` 展开成实际内容后写回，如果原来的 `PATH` 超过 1024 个字符会被截断（`setx` 的历史限制），可能损坏系统 PATH。生产机更推荐用 PowerShell 的 `[Environment]::SetEnvironmentVariable` 或直接在图形界面里改。

## 3.4 Linux 与 macOS 上的配置

```bash
# 编辑当前用户的 shell 配置（bash 用 ~/.bashrc，zsh 用 ~/.zshrc）
export JAVA_HOME=/usr/lib/jvm/temurin-17-jdk
# 用 $JAVA_HOME/bin 拼出实际路径，避免两处配置不一致
export PATH="$JAVA_HOME/bin:$PATH"
```

```bash
# 让配置立即生效，不必重开终端
source ~/.bashrc

# Debian/Ubuntu 上还可以用系统级的 alternatives 机制统一管理多个 JDK
sudo update-alternatives --install /usr/bin/java java /usr/lib/jvm/temurin-17-jdk/bin/java 100
sudo update-alternatives --config java      # 交互式切换当前使用的 java
```

## 3.5 多版本共存与切换

一台机器上同时存在 JDK 8 和 JDK 17 很常见（老项目跑 8，新项目跑 17）。此时**唯一决定"用哪个版本"的就是 `PATH` 中 `%JAVA_HOME%\bin` 的先后顺序**——系统找到第一个就停。

工具层面的解决办法有三种：Windows 用 PowerShell 临时会话覆盖变量；Linux/macOS 用 `update-alternatives` 或 `jenv`；工程层面则在 Maven/Gradle 里指定编译目标版本，让构建工具自己选 JDK。第三种最可靠，因为版本需求本质上是**项目的属性**而不是机器的属性。

```text
REM cmd：只在当前窗口临时生效，关掉窗口即失效，适合临时验证老项目
set JAVA_HOME=C:\Java\jdk-8
set PATH=%JAVA_HOME%\bin;%PATH%
java -version
```

## 3.6 验证安装与常见故障

配置完成后，重新打开终端执行以下命令：

```bash
java -version      # 查看运行时版本
javac -version     # 查看编译器版本，两者必须一致
echo $JAVA_HOME    # Linux/macOS 下检查 JAVA_HOME
```

```text
REM Windows 下对应使用 %JAVA_HOME%，并可用 where 查出命令实际来自哪里
echo %JAVA_HOME%
where java
where javac
```

`java -version` 的正常输出形如：

```text
java version "21.0.2" 2024-01-16 LTS
Java(TM) SE Runtime Environment (build 21.0.2+13-LTS-58)
Java HotSpot(TM) 64-Bit Server VM (build 21.0.2+13-LTS-58, mixed mode, sharing)
```

三行分别说明：**语言版本与是否为 LTS**、**运行时环境版本**、**虚拟机实现与运行模式**（`mixed mode` 表示解释执行与 JIT 编译混合，`sharing` 表示启用了 CDS 类数据共享，能加快启动）。注意 `java -version` 的输出流到的是**标准错误**而不是标准输出，所以 `java -version | grep xxx` 在某些环境下抓不到内容，需要写成 `java -version 2>&1 | ...`。

常见故障与处理：

| 现象 | 原因 | 解决 |
|------|------|------|
| `'java' 不是内部或外部命令` | `PATH` 未配置或终端未重启 | 检查 `%JAVA_HOME%\bin` 是否在 `Path` 中，重开终端 |
| `'javac' 不是内部或外部命令`（但 `java` 正常） | 只装了 JRE，或 `Path` 指向了 JRE | 改指 JDK 的 `bin` |
| `java` 与 `javac` 版本不一致 | `PATH` 里有多份 JDK，顺序错乱 | 用 `where java` / `where javac` 逐个排查 |
| `UnsupportedClassVersionError` | 用高版本编译、用低版本运行 | 统一版本，或编译时加 `--release` 指定目标版本 |
| `错误: 编码 GBK 的不可映射字符` | 源文件是 UTF-8，`javac` 却按系统 GBK 解码 | 编译时显式加 `-encoding UTF-8`（见第 2 篇） |
| `Error: A JNI error has occurred` | 类文件损坏或 JDK 与依赖版本不匹配 | 清理重新编译 |

# 4. 编辑器与 IDE 的选择

初学阶段最容易产生的误区是"必须用 IDEA 才能写 Java"。事实恰好相反：**在学习 `javac`、`java`、classpath 这些概念时，用一个纯文本编辑器手工敲命令，理解成本最低、收益最高**——因为 IDE 会把这些步骤全部藏起来，出问题时你无从下手。

**Notepad++** 是 Windows 上最常用的轻量级开源编辑器，特点是启动速度快、支持 Java 语法高亮、适合初学者快速编写和修改单个 `.java` 文件。它用于编写基础 Java 源码完全够用，但它不具备代码补全、断点调试、依赖管理和重构能力，**大型项目开发必须使用真正的 IDE**。

| 工具 | 定位 | 优点 | 缺点 | 适合阶段 |
|------|------|------|------|----------|
| Notepad++ | 轻量文本编辑器 | 秒开、零配置、强迫你手敲命令 | 无补全、无调试、无重构 | 入门理解编译链路 |
| VS Code（+ Java 扩展包） | 轻量编辑器 + 插件 | 启动快、跨语言、插件生态好 | 大型 Java 项目的索引与调试不如 IDEA | 中小项目、多语言场景 |
| IntelliJ IDEA | 专业 IDE | 智能补全与重构最强、调试与 Spring 支持最好 | 吃内存、社区版功能有取舍 | 主力开发 |
| Eclipse | 老牌 IDE | 免费开源、插件体系成熟、老项目多 | 交互体验相对陈旧 | 维护 Eclipse 系老项目 |

> **最佳实践**：入门阶段先用 Notepad++ 手写一次 `javac` 和 `java`，把全文所有命令跑通一遍；之后再切换到 IDEA。IDE 里的 "Build" 按钮本质上就是在替你执行 `javac -cp ... -d out`，理解这层映射，遇到 `NoClassDefFoundError` 时才不会束手无策。此外，用 Notepad++ 保存 `.java` 文件时要选择 **UTF-8 无 BOM** 编码——带 BOM 的文件开头会多出不可见的 `\ufeff` 字符，`javac` 会在第 1 行直接失败，且报错位置看上去毫无问题，极具迷惑性：文案随 JDK 版本而异，可能是 `需要 class、interface、enum 或 record`，也可能是 `非法字符: '\ufeff'`。

# 5. 第一个程序与编译运行全过程

环境就绪后，用最小可运行的程序把整条链路走通。这个例子看似简单，却包含了 Java 全部核心机制的缩影：**类、包、访问修饰符、静态方法、标准输出，以及"编译"与"运行"两个完全分离的阶段**。

## 5.1 编写源码

新建文件 `HelloWorld.java`（注意**文件名大小写必须与 `public` 类的类名完全一致**，这是 Java 的硬性规定）：

```java
// HelloWorld.java
// 一个 public 类必须独占一个同名文件：文件名 HelloWorld.java 对应类 HelloWorld
public class HelloWorld {

    // main 方法是 JVM 唯一认得的入口，签名一个字符都不能改
    // public：JVM 从外部调用，必须公开
    // static：类加载后即可调用，不需要先 new 一个对象
    // void：  没有返回值
    // String[] args：接收命令行参数，如 java HelloWorld a b，则 args = {"a", "b"}
    public static void main(String[] args) {
        System.out.println("Hello, World!");   // println 输出后自动换行，print 不换行
    }
}
```

如果文件名写成 `Helloworld.java`（小写 w）而类名是 `HelloWorld`，`javac` 会直接报错：

```text
错误: 类 HelloWorld 是公共的, 应在名为 HelloWorld.java 的文件中声明
```

> **注意**：`public` 类的名字必须与文件名一致，但**非 public 类可以放在任意文件里**（一个 `.java` 文件可以有多个类，其中最多一个 public 类）。这条规则的目的是让编译器能按类名直接定位源文件，IDE 和构建工具都依赖这一点。

## 5.2 编译：javac

在源码所在目录打开终端，执行：

```bash
javac HelloWorld.java
```

`javac` 做的是**把 Java 源码翻译成字节码**：先做词法、语法分析，再做类型检查（这就是语法错误的报错时机），全部通过后生成字节码文件 `HelloWorld.class`。它**不会执行任何程序逻辑**——`System.out.println` 在这时候只是被翻译成一条 `invokevirtual` 指令，不会真的打印。

```text
REM 编译成功后目录下会多出字节码文件
dir /b
REM 输出：HelloWorld.class  HelloWorld.java
```

如果代码有语法错误，`javac` 会给出**行号 + 错误描述**，且**不生成** `.class`：

```text
HelloWorld.java:5: 错误: 需要';'
        System.out.println("Hello, World!")
                                          ^
1 个错误
```

`.class` 文件可以用 JDK 自带的 `javap` 反汇编查看内容，这是理解"编译到底做了什么"的绝佳手段（见 5.6）。

## 5.3 运行：java

```bash
java HelloWorld
```

```text
Hello, World!
```

这里要注意两点：**类名后面不能加 `.class` 后缀**；大小写必须与类名严格一致。原因是 `java` 命令的参数不是"文件路径"，而是**类的全限定名**（Fully Qualified Name），JVM 会自己到 classpath 里按这个名称去找对应的 `.class` 文件——所以它要求的是"类叫什么"，而不是"文件在哪"（详见第 2 篇）。

常见的第一条运行错误是：

```text
错误: 找不到或无法加载主类 HelloWorld
原因: java.lang.ClassNotFoundException: HelloWorld
```

绝大多数情况是因为**执行 `java` 命令时所在的目录不对**，或者源码里写了 `package` 声明。前者切到 `.class` 所在目录即可，后者需要按第 2 篇的规则使用全限定名。

> **注意**：`main` 方法的签名必须严格是 `public static void main(String[] args)`。写成 `public void main(String[] args)`（漏了 `static`）、`public static int main(...)`（返回类型不是 `void`）、`static void main()`（参数不对）**都能通过编译**，但 `java` 命令会拒绝执行。现代 JDK 会针对不同错法给出精确提示——漏 `static` 报 "main 方法不是类 X 中的 static"、返回类型不对报 "main 方法必须返回类 X 中的空类型值"、参数或访问修饰符不合规则报 "在类 X 中找不到 main 方法"。这些提示与下面要讲的 "找不到或无法加载主类" 是**两类完全不同的错误**：前者是入口方法签名不合规，后者是类名或 classpath 有问题，排查时不要混为一谈。

## 5.4 从源码到输出的完整链路

把整个过程串起来看：

```text
  HelloWorld.java（源码，UTF-8 文本，给人看）
        │
        │  javac HelloWorld.java（编译期：语法分析、类型检查、生成字节码）
        ▼
  HelloWorld.class（字节码，平台无关，文件头魔数 CAFEBABE）
        │
        │  java HelloWorld（运行期：启动 JVM，按类名到 classpath 查找）
        ▼
  ┌──────────────────────────────────────────────────────────┐
  │ 类加载子系统：加载 → 验证 → 准备 → 解析 → 初始化           │
  │  （验证阶段会做字节码校验，确保指令合法、不越界访问内存）    │
  └──────────────────────────────────────────────────────────┘
        │
        ▼
  字节码执行引擎：解释器逐条执行 + JIT 把热点代码编译成本地机器码
        │
        ▼
  控制台输出 Hello, World!
```

这条链路上有三个**必须分清的时间点**：

1. **编译期**（`javac`）：只做静态检查与翻译，不加载类、不执行代码。类之间的引用在这一步只被记录成符号引用。
2. **类加载期**（JVM 启动时）：把 `.class` 读进内存，做验证、分配静态变量空间、把符号引用替换为直接引用。此时**静态代码块执行，静态变量完成初始化**。
3. **运行期**：真正执行字节码。`new` 对象、调用方法、读写字段都发生在这一步。

> **设计考量**：把"编译"和"运行"彻底拆开，正是 Java 能跨平台的前提——编译产物是抽象的字节码而不是某个 CPU 的机器码，平台相关的工作被推迟到了运行期的 JVM 里。代价是**首次运行需要 JIT 预热**，所以 Java 服务启动后的一段"慢热期"是正常的；GraalVM Native Image 等技术正是为了消除这个代价而出现的，代价则是牺牲了部分动态特性。

## 5.5 跨平台到底跨的是什么

"Write Once, Run Anywhere（一次编写，到处运行）"常被误读为"Java 程序在任何平台上都能直接跑"。准确的表述是：**同一份 `.class` 字节码在任何装有对应 JVM 的平台上都能运行，不需要重新编译**。

```text
                        HelloWorld.class（唯一一份，平台无关）
                                  │
        ┌───────────────┬─────────┴─────────┬───────────────┐
        ▼               ▼                   ▼               ▼
   Windows JVM     Linux JVM          macOS JVM       其他平台 JVM
        │               │                   │               │
        ▼               ▼                   ▼               ▼
   Windows 机器码   Linux 机器码       macOS 机器码     对应平台机器码
```

代价也很明显：**多了一层虚拟机的抽象，就意味着多了一层翻译开销**，同时每个平台都必须有厂商愿意维护 JVM。这就是为什么 Java 从未真正占领移动端和浏览器端（Android 自成一套，Applet 已被彻底淘汰）——JVM 本身是平台相关的，跨平台的能力建立在"有人为这个平台实现了 JVM"之上。

## 5.6 用 javap 看看字节码长什么样

```bash
javap -c HelloWorld
```

```text
Compiled from "HelloWorld.java"
public class HelloWorld {
  public HelloWorld();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: getstatic     #7                  // Field java/lang/System.out:Ljava/io/PrintStream;
       3: ldc           #13                 // String Hello, World!
       5: invokevirtual #15                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
       8: return
}
```

这份输出信息量很大：**构造函数是编译器自动加的**（哪怕你没写，也会有默认无参构造）；`main` 里那行 `System.out.println` 被翻译成了"取静态字段 `System.out` → 加载常量 → 调用虚方法"三条指令。看懂这段对于后面理解多态（`invokevirtual` 的动态绑定）、类加载顺序、方法调用开销都很有帮助。

> **最佳实践**：以后遇到"这行代码编译后到底是什么"的疑问，`javap -c -p 类名` 是最快的验证手段。`-p` 会额外显示私有成员，`-v` 则输出常量池、栈映射表等更底层的信息。

# 6. Java 语言的核心特点

Java 的每一条设计特点都不是凭空出现的，它们大多是对 C++ 在实际工程中暴露出的问题的**针对性修补**。理解了"它在防备什么"，才能理解后面那些语法规定的由来。

## 6.1 面向对象

Java 是**纯粹的面向对象语言**（除基本类型外一切皆对象），支持封装（Encapsulation）、继承（Inheritance）和多态（Polymorphism）三大特性。封装让内部实现可以随时替换而不影响调用方，继承让公共逻辑能够复用，多态让调用方只依赖抽象而不依赖具体实现——这三点共同构成了"程序结构清晰、易于扩展和维护"的基础。

Java 在这三点上各做了严格约束：**不允许全局变量和全局函数**（一切代码必须在类里）；**不允许类的多重继承**（避免 C++ 的菱形继承歧义），只保留接口的多实现；**大多数类默认是虚方法**（即可被重写），只有 `final`、`private`、`static` 方法不可重写。这些约束牺牲了一部分灵活性，换来的是"任何一段 Java 代码都能被工具精确分析"的确定性。

## 6.2 简单

"简单"是相对 C++ 而言的，具体体现在三个"移除"和一个"自动化"上：

- **移除指针**：Java 里没有指针运算，也就没有 C++ 中 `p++` 越界、野指针、悬空指针这一整类内存安全问题的来源。引用本质上是指针，但被语言严格约束，不能做算术运算。
- **移除运算符重载**：C++ 中 `a + b` 可能是任意行为，Java 规定运算符的含义固定（唯一的例外是 `+` 对字符串的重载），任何人读代码都能立刻知道 `+` 在做什么。
- **移除多重继承**：用接口的"多实现 + 默认方法"来替代，规避菱形继承带来的字段与方法二义性。
- **垃圾回收（GC）自动化**：不再手工 `malloc/free`，由 JVM 的 GC 自动回收不再可达的对象，从根本上消除了内存泄漏与重复释放中的一大类问题。

| C++ 概念 | Java 的处理 | 为什么这样设计 |
|----------|-------------|----------------|
| 指针与指针运算 | 只有受约束的引用 | 消除越界与野指针，提高内存安全性 |
| 手动 `new`/`delete` | GC 自动回收 | 免除手工内存管理的负担与错误 |
| 运算符重载 | 不支持（`+` 拼接字符串除外） | 保证运算符语义唯一，代码可读 |
| 多重继承 | 单继承类 + 多实现接口 | 规避菱形继承歧义 |
| 头文件与预处理 | 无，靠 `import` + 包 | 消灭宏带来的不可分析性 |
| 模板（Template） | 泛型是**类型擦除**实现 | 保证与旧版本字节码的兼容性 |

> **设计考量**：这些"移除"让 Java 失去了 C++ 的极致控制力（比如无法做零开销的泛型特化、无法精细控制内存布局），但换来的是**更低的团队协作成本**——语言去掉的表达能力，最终都变成了编译器、IDE 和静态分析工具能够检查的确定规则。

## 6.3 安全

Java 的安全性是**分层**的，从编译到运行层层设防：

- **无指针**：从语言层面杜绝内存越界和非法访问，这是最根本的一层；
- **字节码校验器**：JVM 在**类加载的验证阶段**检查字节码是否合法——操作数栈深度是否匹配、跳转目标是否在方法内部、类型转换是否合规、是否越权访问私有成员。校验不通过直接抛 `VerifyError`，恶意构造的 `.class` 无法执行；
- **安全沙箱**：通过 `SecurityManager` 限制程序对文件系统、网络、系统属性的访问权限，历史上是 Applet 能与浏览器共存的基石；
- **强封装**：JDK 9 之后默认封闭了 JDK 内部 API（`sun.*` 等），并通过模块系统限制包之间的可见性，把"能做什么"的边界写进了模块描述文件。

> **注意**：`SecurityManager` 在 JDK 17 中被标记为废弃，JDK 21 起可以通过参数禁用。今天 JDK 层面的沙箱机制已经基本退出历史舞台，取而代之的是**进程级/容器级隔离**（容器、命名空间、cgroups）。但字节码校验器依然是每次类加载都要过的硬关卡，从未失效。

## 6.4 多线程

Java 在语言层面就内置了多线程支持：`Thread` 类、`synchronized` 关键字、`volatile` 关键字、`java.util.concurrent` 并发包，构成了一套完整的并发工具集。这与 C++ 直到 C++11 才把线程模型写进标准形成了鲜明对比。

Java 的并发设计哲学是"**把内存模型写进规范**"：JMM（Java Memory Model）明确定义了多线程下读写的可见性规则，以及 `volatile`、`synchronized`、`final` 的语义边界。这样一来，"某段并发代码在 x86 上跑得好好的、换到 ARM 就出问题"这类硬件相关的诡异现象，在规范层面被排除掉了。

正因为线程是一等公民，Java 才能在服务端高并发领域长期占优。而 JDK 21 引入的**虚拟线程**更是把并发模型从"线程 = 操作系统线程"推进到"线程 = JVM 调度的轻量任务"，让"一个请求一个线程"这种最自然的编程模型重新变得可行——相关内容见第 17 篇与第 29 篇。

## 6.5 跨平台

**WORA（Write Once, Run Anywhere）** 的实现机制在 5.5 已经展开：源码编译成平台无关的字节码，各平台由对应的 JVM 负责解释或编译执行。这里补充一句成本视角：跨平台的收益是**免去多平台编译与分发的负担**，代价是**启动慢、内存占用高、与本地系统交互需要 JNI**。这也是为什么现代 Java 生态一边用容器化交付（把"平台相关"的部分固化进镜像），一边用 GraalVM、Project Panama 等方案往"更贴近本机"的方向回摆。

## 6.6 开源

**OpenJDK** 是 Java SE 的官方开源参考实现，采用 GPLv2 + Classpath Exception 协议。这一协议安排很关键：GPLv2 保证源码必须公开，而 Classpath Exception 允许你把自己的程序链接到 OpenJDK 类库上而**不必开源自己的代码**——正是这条例外，让商业软件可以放心地构建在开源 JDK 之上。

开源带来的实际好处有三点：**任何人可以审计实现细节**（安全和性能问题无处隐藏）；**多家厂商可以基于同一份代码构建各自的发行版**（见 3.1 的表格）；**规范的演进有 JCP 流程和 JEP 提案公开讨论**，社区可以参与。今天除了 Oracle JDK 的部分商业特性外，几乎所有 JDK 发行版都源自 OpenJDK。

## 6.7 特点汇总

| 特点 | 具体表现 | 带来的收益 | 付出的代价 |
|------|----------|-----------|-----------|
| 面向对象 | 封装、继承、多态；类单继承、接口多实现 | 结构清晰、易扩展、易协作 | 代码量偏大、不够灵活 |
| 简单 | 无指针、无运算符重载、无多重继承、GC 自动回收 | 上手成本低、不易写出内存错误 | 失去底层控制力、GC 有停顿 |
| 安全 | 无指针、字节码校验、模块强封装 | 恶意字节码难以执行、边界清晰 | 反射受限、部分库需加 `--add-opens` |
| 多线程 | `Thread`、`synchronized`、JUC 并发包、虚拟线程 | 天然适合服务端高并发 | 并发问题调试难度高 |
| 跨平台 | 字节码 + 各平台 JVM | 一次编译到处运行 | 启动慢、内存开销大、依赖 JVM |
| 开源 | OpenJDK，多发行版 | 透明、生态繁荣、可选供应商多 | 版本与发行版选择成本高 |

# 7. 小结

| 主题 | 一句话要点 |
|------|-----------|
| 三大平台 | SE 是基础（本系列主线），ME 已淘汰，EE 2017 年移交 Eclipse 后更名 Jakarta EE |
| 版本选择 | 只选 LTS：8 / 11 / 17 / 21 / 25；新项目用最新 LTS，老项目跨版本直接升级 |
| 三者关系 | **JDK ⊃ JRE ⊃ JVM**；JDK 多了 `javac` 等开发工具，JRE 只负责运行，现代一律装 JDK |
| JDK 9 的变化 | 模块化让独立 `jre` 目录消失，`dt.jar`/`tools.jar` 被移除，`jlink` 可按需生成精简运行时 |
| 环境变量 | `JAVA_HOME` 指向 JDK 根目录（不带 `bin`），`PATH` 加入 `%JAVA_HOME%\bin`，**不要设置 `CLASSPATH`** |
| 编译运行 | `javac HelloWorld.java` 生成平台无关的 `.class`，`java HelloWorld` 按 **全限定名** 启动 JVM，不带 `.class` 后缀 |
| 跨平台 | 跨的是**字节码**，各平台各有一份 JVM 负责翻译成本地机器码；JVM 本身是平台相关的 |
| 语言特点 | 面向对象、简单、安全、多线程、跨平台、开源，每条都是对 C++ 实际问题的一次针对性取舍 |

下一篇（第 2 篇）从 `HelloWorld` 的单文件模式走向工程化组织：讲清 **package 的目录约定、import 的三种写法、`javac`/`java`/`jar` 的完整参数，以及 classpath 的查找规则**，把 `NoClassDefFoundError` 与 `ClassNotFoundException` 这类报错一次性讲透。
