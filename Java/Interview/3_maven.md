# 1. 知识点
- **Maven核心功能**：项目构建（编译、测试、打包、部署）、依赖管理（自动下载传递依赖）、版本管理（统一版本控制）、生命周期管理（标准化构建流程）
- **构建工具演进**：Make（C/C++，手动写规则）→ Ant（Java，过程式，无生命周期）→ Maven（约定优于配置，声明式）→ Gradle（增量构建，DSL灵活，性能更优）
- **约定优于配置**：Maven预设项目结构（src/main/java、src/main/resources、src/test/java、src/test/resources），减少配置，统一团队规范
- **Maven坐标(GAV)**：groupId（组织/公司标识，如com.alibaba）、artifactId（模块名，如fastjson）、version（版本号，如1.2.83）、packaging（打包方式，默认jar，可选war/pom/ear）
- **version版本号规范**：主版本.次版本.修订版本-里程碑版本（如2.3.1-RELEASE），SNAPSHOT表示快照版（开发中，每次构建都会检查更新），RELEASE表示正式版
- **本地仓库**：`~/.m2/repository`，存放本地缓存的所有jar包和插件，Maven首先从本地仓库查找依赖
- **中央仓库**：Maven官方维护的公共仓库（https://repo1.maven.org/maven2），默认从中央仓库下载未缓存的依赖
- **远程私服(Nexus/Artifactory)**：企业内网私有仓库，缓存中央仓库jar减少外网流量，托管公司内部私有jar，提高下载速度，增强安全性
- **仓库查找顺序**：本地仓库 → 私服（settings.xml配置）→ 中央仓库（若私服未找到且未配置mirrorOf拦截）
- **mirror镜像配置**：settings.xml中配置mirror，mirrorOf指定拦截哪些仓库（`*`拦截所有，`central`仅拦截中央仓库），所有请求被重定向到镜像地址，常配合私服使用
- **依赖传递**：A依赖B，B依赖C，则A自动拥有C（传递依赖）。传递条件：scope为compile或runtime时传递，provided/test不传递
- **依赖调解原则**：最短路径优先（A→B→C 1.0 vs A→C 2.0，选C 2.0）；路径相同时最先声明优先（谁在pom.xml中先声明用谁的版本）
- **排除传递依赖(exclusions)**：`<exclusions><exclusion>`排除不需要的传递依赖，解决jar包冲突（如排除旧版本spring-core），不影响直接依赖
- **dependencyManagement统一版本**：在父POM中声明依赖版本但不实际引入，子模块引用时不需写版本号（继承父POM的版本），实现版本统一管控
- **可选依赖(optional)**：`<optional>true</optional>`表示不传递，如数据库驱动各项目自己引入，不由框架传递
- **scope作用域**：compile（默认，编译/测试/运行都可用，会传递）、test（仅测试可用，不传递，如junit）、runtime（编译不需要、运行需要，如JDBC驱动）、provided（编译/测试需要、运行由容器提供，如servlet-api，不传递）
- **import作用域**：在dependencyManagement中使用`<type>pom</type><scope>import</scope>`导入BOM（Bill of Materials），如spring-boot-dependencies统一管理所有Spring依赖版本
- **system作用域（已废弃）**：`<scope>system</scope>`配合`<systemPath>`指定本地jar路径，不推荐使用，classpath不可移植
- **Maven生命周期(lifecycle)**：三大内置生命周期，每个生命周期包含多个阶段(phase)，阶段按顺序执行
- **clean生命周期**：pre-clean → clean（删除target目录）→ post-clean
- **default生命周期**（核心构建）：validate（验证项目结构）→ compile（编译源代码）→ test（运行单元测试）→ package（打包jar/war）→ verify（集成测试验证）→ install（安装到本地仓库）→ deploy（部署到远程仓库/私服）
- **site生命周期**：pre-site → site（生成站点文档）→ post-site → site-deploy（部署站点）
- **执行mvn命令**：`mvn clean install`表示先执行clean生命周期到clean阶段，再执行default生命周期到install阶段，每个阶段会先执行之前的所有阶段
- **Maven插件(plugin)**：每个阶段绑定了插件目标(goal)，插件是Maven功能的核心执行单元，如compile阶段绑定maven-compiler-plugin的compile目标
- **maven-compiler-plugin**：编译插件，配置`<source>`和`<target>`指定JDK版本（建议配置release或同时配置encoding=UTF-8），解决"不再支持源选项5"警告
- **spring-boot-maven-plugin**：打出可执行Fat Jar（嵌套Jar），MANIFEST.MF指定Main-Class为JarLauncher，BOOT-INF/lib存放依赖jar，BOOT-INF/classes存放应用类
- **maven-surefire-plugin**：单元测试执行插件，默认绑定test阶段，可配置includes/excludes筛选测试类，配置forkCount并行执行测试
- **maven-failsafe-plugin**：集成测试插件，命名约定以IT结尾（如*IT.java），绑定verify阶段，测试失败不会使构建立即失败
- **maven-resources-plugin**：资源文件处理插件，复制资源文件到输出目录，支持filtering（变量替换，`${}`占位符通过profile/filter文件替换值）
- **docker-maven-plugin**：构建和推送Docker镜像，如spotify的docker-maven-plugin或fabric8的docker-maven-plugin
- **聚合(modules)**：父POM声明`<modules><module>子模块名</module></modules>`，执行父POM的mvn命令时对所有子模块执行相同操作，统一构建多模块项目
- **继承(parent)**：子模块声明`<parent>`引用父POM，继承groupId/version/properties/dependencies/dependencyManagement/pluginManagement等，减少重复配置
- **聚合vs继承区别**：聚合用于统一构建（父知道有哪些子模块），继承用于统一配置（子知道父POM在哪）；聚合关系在父POM中声明，继承关系在子POM中声明；两者可以同时使用
- **父POM packaging必须为pom**：既用于聚合又用于继承的父模块packaging必须为pom，不产出jar
- **dependencyManagement只声明不引入**：子模块需要该依赖时必须在自己的dependencies中引用（不写版本），否则不会实际引入
- **pluginManagement只声明不执行**：类似dependencyManagement，声明插件版本和配置但不实际执行，子模块声明该插件时才继承版本配置
- **properties统一版本号**：`<properties>`集中定义版本号，`${}`引用，修改一处全局生效，是管理版本的最常用方式
- **Maven Profile多环境**：通过`<profiles><profile>`定义不同环境配置（开发/测试/生产），通过`<activation>`（默认激活/jdk版本/操作系统/属性值）或命令行`-P`参数激活
- **spring.profiles.active与Maven profile配合**：Maven profile通过resource filtering将`${}`替换为环境值写入application.properties，或使用application-{profile}.yml与spring.profiles.active配合
- **Maven settings.xml**：全局配置（conf/settings.xml）和用户配置（~/.m2/settings.xml），配置仓库认证信息（server username/password）、镜像（mirror）、本地仓库路径（localRepository）、profile等
- **仓库认证安全**：不要将密码硬编码在pom.xml中，应在settings.xml中使用`<server><id>`配合加密存储，企业使用Nexus的token认证
- **SNAPSHOT版本检查策略**：默认每天更新一次，可通过`<updatePolicy>`改为always/never/interval:X（分钟），开发阶段频繁变化使用SNAPSHOT
- **依赖冲突检测**：`mvn dependency:tree`查看依赖树；`mvn dependency:analyze`分析未使用的声明依赖和未声明的使用依赖；IDEA的Maven Helper插件可视化冲突
- **BOM(Bill of Materials)**：将一组稳定兼容的依赖版本集合定义在POM中，通过import scope引入，如Spring Boot BOM、Spring Cloud BOM，统一版本避免冲突
- **Gradle**：新一代构建工具，基于Groovy/Kotlin DSL，构建脚本是代码而非XML，支持增量构建（仅构建变更部分），build cache和daemon进程大幅提升性能
- **Gradle vs Maven性能**：Gradle利用增量构建（跟踪input/output，变化才重新执行）、构建缓存（复用之前任务输出）、守护进程（长期运行JVM避免冷启动），大型项目构建速度快2-10倍
- **Gradle依赖管理**：`implementation`（编译和运行可用，不传递给消费者）、`api`（编译和运行可用，传递给消费者）、`compileOnly`（同provided）、`runtimeOnly`（同runtime）、`testImplementation`（同test）
- **Gradle Wrapper**：`gradlew`/`gradlew.bat`脚本+`gradle-wrapper.properties`指定Gradle版本，团队成员无需安装Gradle，首次使用自动下载，确保版本一致性
- **Maven Wrapper(mvnw)**：类似Gradle Wrapper，`mvnw`脚本+`.mvn/wrapper/maven-wrapper.properties`指定Maven版本，确保CI和开发环境使用相同Maven版本
- **Maven的插件版本锁定**：在父POM的pluginManagement中锁定所有插件版本，避免不同环境Maven默认插件版本不一致导致的行为差异
- **Maven私服上传(deploy)**：在pom.xml中配置`<distributionManagement><repository>`指定私服地址，settings.xml中配置server认证，`mvn deploy`执行上传
- **Maven私服下载(settings.xml + mirror)**：配置mirror将所有仓库拦截到私服，私服自动代理远程仓库，本地所有依赖下载都走私服
- **多模块项目依赖顺序问题**：通过`<dependency>`依赖同项目其它模块时，Maven自动确定构建顺序；配合`mvn install`将模块安装到本地仓库再供下游使用

# 2. 面试题

## 2.1 Maven的生命周期是什么？有哪些内置生命周期？
Maven有三个内置生命周期：
- **clean**：清理项目，包含pre-clean、clean（删除target目录）、post-clean三个阶段
- **default**：核心构建流程，包含validate → compile → test → package → verify → install → deploy等23个阶段
- **site**：生成项目站点文档，包含pre-site、site、post-site、site-deploy

每个生命周期由多个阶段（phase）组成，阶段按顺序执行。执行命令时，该阶段以及之前的所有阶段都会被顺序执行。例如`mvn install`会依次执行validate → compile → test → package → verify → install。每个阶段绑定了插件目标（goal），实际构建工作由插件完成。

## 2.2 Maven的依赖传递机制是什么？如何解决依赖冲突？
依赖传递：A依赖B，B依赖C，则A自动依赖C（传递依赖）。传递条件为scope是compile或runtime时才会传递，provided和test不会传递。

依赖冲突调解策略：
- **最短路径优先**：路径短的版本被选用（A→B→C v1.0（路径长度2）vs A→C v2.0（路径长度1），选v2.0）
- **最先声明优先**：路径长度相同时，按依赖声明顺序，先声明的胜出

解决冲突的方式：
- **exclusions**：排除不需要的传递依赖，如`<exclusions><exclusion><groupId>com.old</groupId>*</exclusion></exclusions>`
- **显式依赖声明**：直接在项目中声明需要的版本（距离最近，优先级最高）
- **dependencyManagement**：在父POM统一声明版本号，子模块引用时不写版本号

## 2.3 dependencyManagement和dependencies有什么区别？
- **dependencies**：直接在当前项目中引入依赖，子模块会继承并实际引入
- **dependencyManagement**：仅在父POM中声明依赖的版本号（version）和作用域（scope）等信息，**不会实际引入依赖**。子模块需要该依赖时，必须在自己的dependencies中声明，但可以不写version和scope（会从父POM继承）。核心作用是**统一管控依赖版本**，解决多模块项目版本不一致问题

类似地，pluginManagement也不实际执行插件，只声明插件的版本和配置，子模块声明该插件时才继承配置。

## 2.4 Maven的scope有哪些？分别说明使用场景
- **compile**（默认）：编译、测试、运行阶段都可用，会传递依赖。如spring-core
- **test**：仅测试编译和运行阶段可用，不传递。如JUnit、Mockito
- **runtime**：编译时不需要，测试和运行时需要，会传递。如JDBC驱动（编译期只依赖JDBC接口，运行时才需要具体驱动实现）
- **provided**：编译和测试时可用，运行时由容器（如Tomcat、Servlet容器）提供，不传递。如servlet-api、lombok
- **import**：仅在dependencyManagement中使用，配合`<type>pom</type>`导入外部BOM，实现依赖版本统一管理。如导入spring-boot-dependencies
- **system**（已废弃）：类似provided但需要显式指定本地jar路径`<systemPath>`，不可移植，不推荐

## 2.5 Maven聚合(modules)和继承(parent)有什么区别？
- **聚合(modules)**：父POM通过`<modules>`声明子模块列表，执行父POM的构建命令时会对所有子模块执行相同操作。聚合是"一起构建"的关系，体现在父POM中
- **继承(parent)**：子POM通过`<parent>`声明父POM坐标，继承父POM的groupId、version、properties、dependencies等配置。继承是"复用配置"的关系，体现在子POM中

两者可以同时使用：一个项目根POM既是父POM（被其他模块继承）又是聚合POM（列出所有模块），packaging必须为pom。聚合实现一键构建整个项目，继承实现统一管理版本和配置。

## 2.6 如何使用Maven Profile实现多环境配置？
在POM中定义多套profile，通过环境标识激活：
```xml
<profiles>
    <profile>
        <id>dev</id>
        <activation><activeByDefault>true</activeByDefault></activation>
        <properties><env>dev</env></properties>
    </profile>
    <profile>
        <id>prod</id>
        <properties><env>prod</env></properties>
    </profile>
</profiles>
```

配合maven-resources-plugin的filtering功能，在application.properties中使用`${env}`占位符，构建时替换为profile中定义的值。激活方式：
- 命令行：`mvn package -Pprod`
- settings.xml中设置activeProfiles
- 自动激活：根据JDK版本、操作系统等条件

与Spring Boot配合：Maven profile可将激活的环境写入spring.profiles.active，或在不同profile下编译不同application-{env}.yml文件。

## 2.7 spring-boot-maven-plugin打出的可执行jar原理是什么？
传统的Maven打包jar只包含项目自身的类文件，不包含依赖jar。spring-boot-maven-plugin打出的Fat Jar结构如下：
- `META-INF/MANIFEST.MF`：包含`Main-Class: org.springframework.boot.loader.JarLauncher`和`Start-Class: com.example.Application`
- `BOOT-INF/classes/`：项目自身的class文件
- `BOOT-INF/lib/`：所有依赖jar包（嵌套jar形式）

启动流程：`java -jar app.jar` → 执行JarLauncher.main() → 创建ClassLoader加载BOOT-INF/lib下的所有嵌套jar → 反射调用Start-Class（即应用的main方法）。核心是JarLauncher能读取嵌套jar并从其中加载类。

## 2.8 Maven私服(Nexus)的作用和工作流程是什么？
Nexus是企业内网Maven仓库管理器，作用：
- **代理外网仓库**：缓存中央仓库等远程仓库的jar，减少外网带宽消耗，加速下载
- **托管内部jar**：部署公司内部开发的jar包到私服，方便团队共享
- **稳定性和安全性**：不受外网波动影响，可控制哪些jar可用，支持安全扫描

工作流程：
- **下载流程**：开发者请求jar → 私服查找 → 有则返回，没有则从远程仓库下载后缓存再返回
- **上传流程**：`mvn deploy` → 根据distributionManagement配置推送到私服的hosted仓库

配置要点：settings.xml中配置mirror将所有仓库拦截到私服地址；配置server的账号密码用于deploy。

## 2.9 Maven的SNAPSHOT和RELEASE版本有什么区别？
- **SNAPSHOT（快照版）**：开发中的版本，不稳定，每次构建都会检查远端是否有最新版本（默认每天一次，可配置updatePolicy）。同一SNAPSHOT版本号对应不同时间的构建结果。适用于团队协作开发，依赖方始终获取最新开发版本
- **RELEASE（正式版）**：发布后的稳定版本，不会变化。发布到私服后不可修改，确保构建的稳定性和可重复性。依赖方获取后缓存在本地仓库不再更新

发布流程：开发阶段使用SNAPSHOT → 测试通过后去掉SNAPSHOT后缀发布正式版 → 版本号升级并恢复SNAPSHOT用于下个迭代

## 2.10 Gradle相比Maven有哪些优势？什么场景选择哪个？
Gradle优势：
- **构建脚本灵活性**：基于Groovy/Kotlin DSL，可以写逻辑代码，比XML灵活
- **增量构建**：跟踪任务输入输出变化，只重新构建变更部分，大型项目节省大量时间
- **构建缓存**：跨构建复用任务输出，本地和远程缓存
- **守护进程(daemon)**：后台运行JVM进程，避免每次冷启动，速度更快
- **依赖管理更细粒度**：implementation vs api区分是否传递依赖给消费者

选择建议：
- 新项目推荐**Gradle**（尤其Android官方已迁移到Gradle），性能更好，DSL更简洁
- 遗留项目、团队不熟悉或企业统一标准选**Maven**，生态更成熟，插件更丰富
- 大型多模块项目Gradle的增量构建优势更明显

## 2.11 dependency:tree命令的作用是什么？如何排查依赖冲突？
`mvn dependency:tree`展示项目的完整依赖树（包括传递依赖），通过缩进体现依赖层次，可以直观看到：
- 每个依赖的来源（哪个直接依赖传递进来的）
- 哪些依赖被省略了（通过omitted标记，说明版本被调解了）
- 冲突的版本信息

排查冲突步骤：
1. `mvn dependency:tree` 查看依赖树，找到冲突的jar版本
2. 使用`dependency:tree -Dincludes=group:artifact`过滤特定依赖的路径
3. 用exclusions排除不需要的版本，或显式声明需要的版本
4. 在IDE中使用Maven Helper插件（IDEA）可视化查看冲突并快速排除

## 2.12 settings.xml中mirror的mirrorOf配置是什么意思？
mirrorOf指定该镜像要拦截哪些仓库的请求。常见配置：
- `*`：拦截所有仓库请求（包括中央仓库和其他远程仓库），所有下载都走该镜像
- `central`：仅拦截对Maven中央仓库的请求
- `external:*`：拦截所有远程仓库但不包括localhost和file协议的本地仓库
- `*,!repo1`：拦截所有但不包括名为repo1的仓库

典型配置（配合Nexus私服）：
```xml
<mirror>
    <id>nexus</id>
    <mirrorOf>*</mirrorOf>
    <url>http://nexus.example.com/repository/maven-public/</url>
</mirror>
```
所有依赖下载全部通过Nexus私服代理。

## 2.13 maven-surefire-plugin和maven-failsafe-plugin有什么区别？
- **surefire-plugin**：单元测试插件，绑定在test阶段。命名约定为`*Test.java`、`Test*.java`，测试失败会导致构建失败。用于执行快速、隔离的单元测试
- **failsafe-plugin**：集成测试插件，绑定在verify阶段。命名约定为`*IT.java`、`IT*.java`，测试失败不会立即停止构建（fail-safe）。用于执行涉及外部资源（数据库、网络）的集成测试

分工：surefire跑单元测试（test阶段），failsafe跑集成测试（verify阶段），确保单元测试全部通过后才执行集成测试。两者可并行配置提高构建效率。

## 2.14 Maven的optional依赖有什么作用？
`<optional>true</optional>`表示该依赖不向下传递。例如：
- 一个持久层框架支持多种数据库驱动（MySQL、Oracle、PostgreSQL），依赖这些驱动时都标记为optional
- 使用该框架的项目需要根据实际使用的数据库**自己引入**对应的驱动

核心思想：框架提供支持能力但不强制传递具体实现，由最终应用自行选择。这与scope相区别：scope控制依赖在哪个classpath可用，optional控制依赖是否传递。

## 2.15 BOM(Bill of Materials)是什么？如何使用？
BOM是一个特殊的POM文件，专门用于管理一组兼容依赖的版本号。典型例子是spring-boot-dependencies和spring-cloud-dependencies。

使用方式：
```xml
<!-- 在dependencyManagement中使用import scope导入BOM -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>2.7.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

导入后，Spring Boot家族所有依赖的版本号由BOM管理，项目中引用Spring Boot组件时不需要写版本号，确保所有组件版本兼容。一个项目可以导入多个BOM，声明顺序决定版本优先级（先声明的BOM版本优先）。

## 2.16 如何配置Maven的JDK版本？为什么JDK8环境mvn compile可能报"不再支持源选项5"？
需要在pom.xml中配置maven-compiler-plugin：
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.11.0</version>
    <configuration>
        <source>1.8</source>   <!-- 源码JDK版本 -->
        <target>1.8</target>   <!-- 编译后字节码版本 -->
        <encoding>UTF-8</encoding>
    </configuration>
</plugin>
```

或者统一配置properties：
```xml
<properties>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
</properties>
```

报"不再支持源选项5"的原因：Maven默认使用JDK5编译（maven-compiler-plugin内置默认值），而JDK高版本不再支持编译为JDK5的目标字节码。解决方案是显式指定source和target为正确的版本。

推荐使用`<release>8</release>`（JDK9+）代替source/target，更准确地控制编译行为。
