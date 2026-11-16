本篇覆盖 Maven 的高频面试题，按"主题块 → 详细分块 → 题目 → 解答"组织。四个主题块：构建基础与生命周期、依赖管理、多模块与环境管理、进阶与对比。Spring Boot 打包运行细节见 10_springboot.md，CI 中的构建实践见 33_cicd.md。

# 1. 构建基础与生命周期

## 1.1 坐标、版本与约定

### 1.1.1 Maven 是什么？构建工具是怎么演进的？

**结论**：Maven = **项目构建工具 + 依赖管理工具 + 项目对象模型（POM）**三位一体，四大核心能力：构建流程标准化（编译/测试/打包/部署）、依赖管理（自动下载并解析传递依赖）、版本管理、约定式项目结构。

**演进史（一句话版）**：

| 代际 | 工具 | 特点 | 被替代的原因 |
|------|------|------|-------------|
| 1 | Make | C/C++ 时代，手工写 target/依赖规则 | 平台绑定、规则维护噩梦 |
| 2 | Ant | Java 任务式构建，XML 写"怎么做" | **无依赖管理、无生命周期**，过程式脚本冗长 |
| 3 | Maven | "做什么"声明式：POM 描述项目，流程由生命周期内置 | 脚本表达力弱（XML 写不了逻辑）、构建性能一般 |
| 4 | Gradle | DSL 即代码 + 增量构建/构建缓存 | 学习成本、生态依赖 Gradle 插件质量（见 4.2.1） |

**约定优于配置（Convention over Configuration）**：Maven 预设目录结构——`src/main/java`（源码）、`src/main/resources`（资源）、`src/test/java` + `src/test/resources`（测试），`target` 为输出目录。项目不需要任何配置就能被识别编译；测试类命名（`*Test.java`）同样靠约定被 surefire 自动执行。代价是"不遵循约定就要显式配置覆盖"，收益是全行业项目长得一样——换项目零结构学习成本，CI 模板也由此标准化。

### 1.1.2 Maven 坐标是什么？版本号有什么规范？

**坐标（GAV）**唯一标识一个构件：

- **groupId**：组织/公司反写域名，如 `com.alibaba`；
- **artifactId**：模块名，如 `fastjson`；
- **version**：版本号，如 `1.2.83`；
- 补充维度：**packaging**（jar/war/pom/ear，默认 jar——`pom` 表示只管理不产出构件，多模块父工程专用）；**classifier**（同一 GAV 的变体，如 `sources`/`javadoc`/`jdk17` 适配包）。

GAV 同时是**仓库目录结构**：`~/.m2/repository/com/alibaba/fastjson/1.2.83/fastjson-1.2.83.jar`——依赖解析本质是按坐标拼路径找文件。

**版本规范**：`主版本.次版本.修订号[-修饰符]`（语义化版本）——主版本=不兼容 API 变更，次版本=向下兼容新功能，修订号=bugfix；修饰符惯例：`SNAPSHOT`（开发快照）、`RELEASE`/`-alpha/-beta/-rc`（里程碑）。JDK 平台版本与 Maven 版本互相约束：maven-compiler-plugin 的 `release`、Spring Boot 3.x 要求 Java 17+ 等，升级时先对 JDK 基线（见 1.2.5）。

### 1.1.3 SNAPSHOT 和 RELEASE 版本有什么区别？

| 维度 | SNAPSHOT | RELEASE |
|------|----------|---------|
| 语义 | **开发中的不稳定版本** | 正式发布的不可变版本 |
| 远端存储 | 每次 deploy 产生带时间戳的新副本（`1.0-20261115.083000-3.jar`），可反复发 | **一经发布不可覆盖**（私服 `Disable Redeploy` 强制） |
| 本地行为 | 每次构建按 `updatePolicy` 检查更新 | 下载后永久缓存，不再检查 |
| 适用 | 团队联调期（依赖方总能拿到最新） | 对外发布、生产构建 |

**updatePolicy**（仓库配置 `<snapshots><updatePolicy>`）：`daily`（默认，每天一次）/`always`/`never`/`interval:X`（X 分钟）。联调频繁改默认 daily 为 `interval:60` 或构建带 `-U` 强制检查（与 2.3.4 的失败缓存联动）。

**标准发布流程**：开发迭代用 `x.y.z-SNAPSHOT` → 测试通过 `versions:revert`/发布插件去掉后缀发 RELEASE → 立刻把主干版本号 +1 恢复 SNAPSHOT 进入下个迭代。**生产构建禁止依赖 SNAPSHOT**——构建不可重现（今天和明天拉到的"同一个版本"字节码可能不同），Sonatype 中央仓库甚至拒绝含 SNAPSHOT 依赖的构件提交。

## 1.2 生命周期与插件

### 1.2.1 Maven 有哪些内置生命周期？

三套相互独立的生命周期，每套由有序阶段（phase）组成：

1. **clean**：`pre-clean → clean（删 target）→ post-clean`；
2. **default**（核心，23 个阶段，常考主线）：`validate → compile → test → package → verify → install → deploy`；
3. **site**：`pre-site → site → post-site → site-deploy`（生成项目文档站点）。

**阶段执行语义**：调用某阶段 = **从头顺序执行到该阶段**——`mvn test` 实际跑了 validate+compile+test；`mvn clean install` 是**两条生命周期**：clean 生命周期执行到 clean，再切 default 生命周期从头执行到 install（两条生命周期之间没有先后约束，只是命令行写法把它们串联）。

**易错**：跳过阶段靠参数不靠"只调某阶段"——`mvn install -DskipTests` 跳过 test 阶段执行（surefire 感知参数自行不跑）；`mvn test package` 这种写法等价 `mvn package`（阶段顺序固定，重复声明不重复执行）。

### 1.2.2 生命周期、阶段和插件是什么关系？执行 mvn clean install 时发生了什么？

**关系模型**：生命周期是**抽象阶段序列**，阶段本身不干活——真正执行的是**绑定（binding）到阶段的插件目标（goal）**。default 生命周期对 `jar` 打包的默认绑定：

| 阶段 | 默认绑定的插件目标 |
|------|-------------------|
| compile | maven-compiler-plugin:compile |
| test | maven-surefire-plugin:test |
| package | maven-jar-plugin:jar |
| install | maven-install-plugin:install |
| deploy | maven-deploy-plugin:deploy |

自定义插件用 `<executions><phase>/<goal>` 显式绑定——`spring-boot-maven-plugin:repackage` 绑到 package（见 4.1.1）、`maven-failsafe-plugin` 的 integration-test+verify 双阶段绑定（1.2.4）。**同一阶段绑多个目标时，执行顺序 = POM 中插件声明顺序**（高频追问，也是"两个插件改同一样东西行为诡异"的根因）。

**mvn clean install 全程**：解析 settings.xml/pom → 建 reactor（多模块时排序）→ clean 删 target → validate 校验 POM 结构 → compile（解析依赖：本地仓库→远程下载，缺插件也要下载）→ test-compile+test → package 打成 target 下 jar → install 写入 `~/.m2/repository` 并记录 `_remote.repositories` 来源。理解"阶段是壳、插件是核"，才能解释为什么"没配任何插件的 pom 也能完成整套构建"（默认绑定）与"构建卡在下载 maven-plugins 而不是依赖"（插件也是构件，走私服镜像，见 2.3.3）。

### 1.2.3 package、install、deploy 三个阶段的区别？

| 阶段 | 产物去向 | 典型场景 |
|------|---------|---------|
| package | `target/` 目录（本地构建产物） | 提交给 CI 出包、手工验证 |
| install | 本机 `~/.m2/repository` | **本机构建多模块**：下游模块从本地仓库找上游构件 |
| deploy | 远程仓库（`distributionManagement` 指向私服/中央） | 团队共享、发布版本 |

**连环追问**：多模块里 moduleB 依赖 moduleA，单独 `mvn package` moduleB 报找不到 moduleA？——A 从没进过本地仓库；解法：在根工程 `mvn install`（reactor 一次性构建全部，模块间直接用 reactor 内构建结果，不依赖本地仓库——这也是推荐"整体构建"而非逐模块 install 的原因，见 3.1.2）或 IDE 勾选 Workspace Resolution。deploy 细节与发布流程见 4.1.2。

### 1.2.4 maven-surefire-plugin 和 maven-failsafe-plugin 有什么区别？

| 维度 | surefire | failsafe |
|------|----------|----------|
| 定位 | **单元测试**（快、隔离、无外部依赖） | **集成测试**（起服务/连库/网络） |
| 命名约定 | `*Test.java`/`Test*.java`/`*Tests.java` | `*IT.java`/`IT*.java`/`*ITCase.java` |
| 绑定阶段 | test | **integration-test**（执行）+ **verify**（判定结果） |
| 失败行为 | 立即中断构建 | IT 失败**不会让 integration-test 当场中断**——后续阶段继续跑完（保证 @After 清理/容器停服都执行），到 verify 阶段才把构建标红 |

"failsafe"名字的含义就在这：**安全地失败**——资源清理比"第一时间报错"优先。配套参数：`-DskipTests` 两者都跳、`-DskipITs` 只跳 IT；surefire `forkCount`/`reuseForks` 多 JVM 并行跑测试提速；`<includes>/<excludes>` 定制筛选。**工程实践**：IT 往往只在 CI 的 nightly 或 `-Pit` profile 下启用（本地默认只跑 UT），与 Testcontainers 结合是主流写法（见 10_springboot.md 5.2.1）。

### 1.2.5 Maven 如何配置编译 JDK 版本？为什么会报"不再支持源选项 5"？

三种写法（优先级从低到高）：

```xml
<!-- ① properties（最常用） -->
<properties>
    <maven.compiler.release>17</maven.compiler.release>  <!-- JDK9+ 推荐 -->
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
</properties>

<!-- ② compiler 插件显式配置 -->
<plugin>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.13.0</version>
    <configuration>
        <release>17</release>
        <encoding>UTF-8</encoding>
    </configuration>
</plugin>
```

- **source/target**：老写法——source=源码语法级别，target=字节码级别；两者不配对设置（source 8 生成 target 5 字节码）会产生"用新 API 编旧字节码"的不一致风险；
- **release**（JDK9+ 编译器提供）：一条参数同时锁定语法级别 **且按对应 JDK 的平台 API 编译**——防止"用 17 编译器写出 8 运行不了/8 的 API 被误用"，新项目标准答案；
- **"源选项 5"报错**：早期 compiler-plugin 默认 source/target=1.5，高版本 JDK 移除了对 5 的支持（`Source option 5 is no longer supported. Use 8 or later.`）——**根因是没显式声明编译版本 + 插件版本过旧**，补 `release` 属性并锁插件版本即可（4.2.2）。乱码报错（unmappable character）同源忘配 encoding。
- **多 JDK 交叉构建**：`maven-toolchains-plugin` 声明"构建用 JDK X、运行目标 JDK Y"，一台机器装多版本 JDK 的团队标准方案。

# 2. 依赖管理

## 2.1 传递、调解与作用域

### 2.1.1 什么是依赖传递？传递规则是什么？

A→B→C，A 自动获得 C——**传递依赖**，由 POM 的 `<dependencies>` 元数据在解析时递归展开成依赖图。

**传递规则由 scope 决定**（核心表，记"compile 全传、test 不传"）：

| 依赖 B 对 C 的 scope | C 对 A 可见性 |
|---------------------|---------------|
| compile | compile（传递） |
| provided | **不传递** |
| runtime | runtime（传递） |
| test | **不传递** |

设计逻辑：provided 意味着"容器/上层已提供"，透传会污染下游；test 是人家自己的测试基建，没理由带出来。Maven 3.x 起 scope 矩阵严格按此执行；**依赖图不允许环**——A→B→A 解析直接报错，解法是抽公共模块断环。

### 2.1.2 Maven 的依赖调解原则是什么？如何排查和解决依赖冲突？

**同一个 GAV 在图中出现多版本，Maven 的仲裁两条规则（按序）**：

1. **最短路径优先**：A→C(2.0)（深度 1）vs A→B→C(1.0)（深度 2）→ 选 2.0；
2. **路径相同，先声明优先**：两条都是深度 2，谁在 POM 里先写 `<dependency>` 用谁的。

注意：调解**只选版本、不报错**——"选错版本"的冲突静默发生在 classpath 上，运行时才以 `NoSuchMethodError`/`ClassNotFoundException` 爆炸（面试先讲这条危害）。

**排查工具箱**：

```text
mvn dependency:tree                        # 完整依赖树，看来源与 "omitted for conflict" 
mvn dependency:tree -Dincludes=org.springframework:spring-core   # 过滤定位路径
mvn dependency:analyze                     # Used undeclared（没声明却用了）/ Unused declared（声明了没用）
IDEA Maven Helper                          # Conflicts 页可视化 + 一键 exclude
mvn dependency:copy-dependencies           # 导出实际 classpath 清单核对
```

**解决手段（按优先级）**：

1. **就近显式声明**目标版本（深度 1 永远赢，最直观）；
2. **exclusions** 掐掉不需要的那条传递路径（支持通配 `*`：`<groupId>com.old</groupId><artifactId>*</artifactId>`）；
3. **dependencyManagement 钉版本**（对所有深度生效、调解规则让位于它，见 2.2.1）——多模块治理正解；
4. 引入对方 **BOM**（同族组件版本成套，见 2.2.2）。

**典型案**：引入 starter-X 后 spring-core 被降级 → `tree -Dincludes` 找到来源 → 上层 `dependencyManagement` 显式锁 spring-framework BOM → 复发即 CI 门禁 `dependency:analyze` + enforcer 插件（`requireUpperBoundDeps`/`banDuplicatePomDependencyVersions`，见 33_cicd.md）。

### 2.1.3 Maven 的 scope 有哪些？各自什么场景？

| scope | 编译期 | 测试期 | 运行期 | 传递 | 典型例子 |
|-------|--------|--------|--------|------|---------|
| **compile**（默认） | ✓ | ✓ | ✓ | ✓ | spring-core |
| **provided** | ✓ | ✓ | ✗（容器提供） | ✗ | servlet-api、lombok |
| **runtime** | ✗ | ✓ | ✓ | ✓ | MySQL JDBC 驱动 |
| **test** | ✗ | ✓ | ✗ | ✗ | junit、mockito |
| **import** | 仅 dependencyManagement 内，导入 BOM（2.2.2） | | | | spring-boot-dependencies |
| **system**（废弃） | 同 provided 但需 `<systemPath>` 指本地路径——不可移植，禁用 | | | | 老项目遗留 |

**三个高频追问**：

1. **runtime 为什么能跑**：JDBC 编译期只依赖 `java.sql.Driver` 接口（JDK 自带），驱动实现运行期靠 SPI/Class.forName 加载——"编译用接口、运行插实现"的依赖倒置落地；
2. **provided 的坑**：打 jar **不打包该依赖**——spring-boot Fat Jar 里 servlet-api 标 provided 是交给外置容器场景；内嵌 Tomcat 应用误把 `spring-boot-starter-tomcat` 配 provided 会导致 `java -jar` 启动缺类；lombok 标 provided 是"编译期注解处理器，运行期不需要"；
3. **test 里用 provided**：provided 在测试 classpath 可见，故"测试时容器由我提供"要显式加回。

### 2.1.4 optional 依赖是做什么的？

`<optional>true</optional>`：**我自己编译用，但不向下游传递**。

**场景**：持久层框架同时支持 MySQL/Oracle/PG 驱动——三个都声明 optional，使用框架的项目按自己实际数据库**自行引入**驱动；spring-boot 的 `configuration-processor`、MyBatis 对 cglib/javassist 的可选支持同款语义。

**与 provided 辨析**（追问必考）：

- provided = "运行时环境已经有了，别打进去"——**下游是否需要不由我控制**；
- optional = "需要就自己显式声明"——**把选择权交还下游**，且不改变本模块的 classpath；
- 两者都不传递，语义完全不同：optional 管的是**传递性**，scope 管的是**可见性阶段**。

配套治理：`maven-enforcer-plugin` 的 `dependencyConvergence` 可强制"同一依赖全图收敛到一个版本"（团队规范工具，见 2.1.2）。

## 2.2 版本统一管控

### 2.2.1 dependencyManagement 和 dependencies 有什么区别？

**一句话**：`dependencies` 真引入；`dependencyManagement` **只登记"该用什么版本"**，不引入任何东西——子模块要用时在 `<dependencies>` 声明（不写 version），版本/ scope 自动继承；**写了也不下载**，这是新手最大困惑点。

```xml
<!-- 父 POM：登记 -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.squareup.okhttp3</groupId>
            <artifactId>okhttp</artifactId>
            <version>4.12.0</version>
        </dependency>
    </dependencies>
</dependencyManagement>

<!-- 子模块：引用即生效，不写版本 -->
<dependencies>
    <dependency>
        <groupId>com.squareup.okhttp3</groupId>
        <artifactId>okhttp</artifactId>
    </dependency>
</dependencies>
```

**关键性质**：dependencyManagement 的约束**压过调解规则**——传递依赖出现多版本时，父工程登记的版本无条件胜出（对直接声明也生效），这是"根 POM 钉版本"治理冲突的机制基础（2.1.2）。配合 `<properties>` 集中版本号（`<okhttp.version>4.12.0</okhttp.version>`），改一处全局生效。

**pluginManagement** 镜像语义：只声明插件版本与 configuration，子模块声明该插件才继承——**锁全部构建插件版本的标配位置**（见 4.2.2）。另注意 `dependencyManagement` 里可以直接写本 reactor 的模块版本，防止子模块互引时版本漂移。

### 2.2.2 BOM 是什么？import 作用域怎么用？

**BOM（Bill of Materials）**= 一个 `packaging=pom` 的特殊工程，内部只有一张 `dependencyManagement` 表：声明**一组经过兼容性验证的版本组合**。典型：`spring-boot-dependencies`（Boot 全家 + 三方几百个坐标）、`spring-cloud-dependencies`、`jackson-bom`、`guava 平台类`。

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>3.3.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

**import 语义**：把目标 POM 的 dependencyManagement **整体并入**当前工程的版本表——不需要以它为 parent，与"继承父工程"解耦（这就是 BOM 相比 parent 方案的自由度：parent 只有一个，BOM 可以叠多个）。

**多 BOM 规则**：先声明者优先（同一坐标第一个命中的版本生效）——Boot BOM 与 Cloud BOM 版本矩阵要对齐（官方有"Cloud 版本 ↔ Boot 版本"对照表，错配是 `NoSuchMethodError` 重灾区，见 15_spring_cloud.md）；**自研 BOM** 常见于平台团队：把内部 starter 的统一版本发布成 BOM，业务只引坐标。

## 2.3 仓库与私服

### 2.3.1 Maven 查找依赖的顺序是什么？

**解析顺序**：

1. **本地仓库**（`~/.m2/repository`，settings.xml `localRepository` 可改）——命中还要看 `_remote.repositories` 来源校验（构件"从哪来"记录，跨机器复制 .m2 后报"present, but unavailable"与此有关，见 2.3.4）；
2. **POM/配置声明的远程仓库**（repositories + mirror 拦截改写，见 2.3.3）——企业环境通常即私服；
3. **中央仓库**（`repo1.maven.org/maven2`，内置 super POM 声明，镜像 `mirrorOf=*` 时被接管）。

按仓库类型逐个尝试，任何一处命中即下载/缓存；全部失败则**记录失败状态不再重试**（`.lastUpdated`，见 2.3.4）。私有依赖不发布到中央，**必须先配仓库/镜像**——"新同事 clone 项目编译不过，缺公司 jar"九成是没配 settings.xml。

### 2.3.2 私服（Nexus/Artifactory）的作用和工作流程？

**三大价值**：① **代理**中央仓库，缓存公共构件——全公司共享带宽与速度，外网抖动不影响构建；② **托管**内部私有构件（团队间 jar 共享、二方库），天然访问控制；③ **安全治理**：漏洞扫描、黑白名单、禁止 SNAPSHOT 上生产等策略统一卡点。

**Nexus 仓库模型（高频追问）**：

| 类型 | 语义 | 典型 |
|------|------|------|
| proxy | 远程仓库的只读缓存代理 | maven-central |
| hosted | 托管上传的本地仓库 | maven-releases（Disable Redeploy）、maven-snapshots |
| group | 聚合多个仓库对外成**一个 URL** | maven-public = central 代理 + releases + snapshots |

**工作流程**：下载——开发者请求 → `maven-public` 聚合查询 → hosted 有则返回，proxy 成员未缓存则回源下载并留存；上传——`mvn deploy` 按 `distributionManagement` 的 `uniqueVersion`/仓库 id 把构件推入 hosted（releases 拒绝重复覆盖、snapshots 时间戳累版本，见 4.1.2）。Artifactory 语义同构（虚拟仓库/本地仓库/远程仓库）。

### 2.3.3 mirrorOf 是什么？私服认证信息放在哪里？

**mirrorOf = 镜像要拦截哪些仓库请求**（按 `<mirror>` 声明顺序第一个匹配生效）：

| 写法 | 含义 |
|------|------|
| `*` | 拦截一切仓库（包括中央和其他自定义仓库）——公司强管控 |
| `central` | 只拦中央仓库 |
| `external:*` | 所有**外部**仓库，localhost/file 协议仓库不拦 |
| `*,!releases` | 拦全部但放行 id=releases 的仓库（私服 deploy 要绕开镜像的典型写法——**mirror 不影响 deploy**，deploy 走 distributionManagement；此处 !id 多用于排除特殊代理仓库） |

```xml
<!-- settings.xml -->
<mirrors>
    <mirror>
        <id>nexus</id>
        <mirrorOf>*</mirrorOf>
        <url>https://nexus.example.com/repository/maven-public/</url>
    </mirror>
</mirrors>
<servers>
    <server>
        <id>nexus</id>                      <!-- 与 mirror/repository/server id 匹配 -->
        <username>ci-bot</username>
        <password>******</password>
    </server>
</servers>
```

**安全纪律**：账号密码**永不写进 pom.xml**——settings.xml 是机器级配置不进版本库；密码可 `mvn --encrypt-master-password` + `--encrypt-password` 加密为 `{cipher}` 串（master 密码放 `~/.m2/settings-security.xml`）；CI 用凭据注入而非明文。settings 两级：全局 `${maven.home}/conf/settings.xml` 与用户 `~/.m2/settings.xml`，同项**用户覆盖全局**。阿里云镜像（`maven.aliyun.com/repository/public`）是个人开发默认提速项。

### 2.3.4 依赖下载失败后，为什么本地明明有目录却报"could not resolve"？-U 是干什么的？

**失败缓存机制**：远程解析失败时 Maven 在本地目录写 `xxx.lastUpdated`（记录尝试时间与失败原因），此后**默认不再重试**——直接报 `Could not resolve dependencies ... present, but unavailable` 或反复沿用失败状态。诱因：当时私服挂了/网络抖动/密码过期/构件当时确实没发。

**处置三板斧**：

1. `mvn -U`（force update）：强制重新检查 SNAPSHOT 更新**并忽略失败缓存重试**——最高频解法；
2. 删除病灶文件：`find ~/.m2/repository -name "*.lastUpdated" -delete`（或定位到具体目录清掉整个损坏构件目录重拉）；
3. 修根因：仓库不可达/认证失效（server id 拼错、mirror 拦截漏配，2.3.3）。

**关联坑**：下载一半断网留下 **0 字节/损坏 jar**（`corrupted by another process`/`zip end block not found`）——同样删目录重拉；`_remote.repositories` 来源校验导致换机器/改镜像 id 后"present, but unavailable"——删该文件或配置 `-llr`（legacy local repository behavior）。这些排障动作在 CI 缓存恢复、私服迁移后批量出现，属于必会被问的实操题。

# 3. 多模块与环境管理

## 3.1 聚合与继承

### 3.1.1 聚合（modules）和继承（parent）有什么区别？

两个方向相反、可同时使用的机制：

| 维度 | 聚合 `<modules>` | 继承 `<parent>` |
|------|-----------------|----------------|
| 写在哪 | **父 POM** 声明子模块列表 | **子 POM** 声明父坐标 |
| 语义 | "一起构建"：在根执行 mvn 命令，reactor 对全部模块依次执行相同生命周期 | "复用配置"：子继承父的 groupId/version/properties/dependencies/dependencyManagement/pluginManagement/profiles |
| 解决的问题 | 一键构建整个工程 | 消除各模块重复的版本号/插件配置 |

```xml
<!-- 根 POM：既是聚合也是继承目标 -->
<groupId>com.example</groupId>
<artifactId>shop-parent</artifactId>
<version>1.0.0</version>
<packaging>pom</packaging>          <!-- 父工程必须 pom：不产 jar，只做管理 -->
<modules>
    <module>shop-common</module>
    <module>shop-api</module>
    <module>shop-service</module>
</modules>

<!-- 子模块 -->
<parent>
    <groupId>com.example</groupId>
    <artifactId>shop-parent</artifactId>
    <version>1.0.0</version>
    <relativePath>../pom.xml</relativePath>   <!-- 本地找父，找不到才去仓库 -->
</parent>
<artifactId>shop-api</artifactId>             <!-- groupId/version 可省略继承 -->
```

**易错**：① 只有 `<modules>` 没有 `<parent>` 也能聚合构建（reactor 生效），但配置不继承；② 只有 `<parent>` 没有 `<modules>` 也能继承（父 POM 从相对路径/仓库解析），但不一起构建；③ 工程惯例两者合用于根 POM。继承还可以"跨工程"（父 POM 单独发布到私服，多仓库共用同一 parent）——公司级父工程的标准形态。

### 3.1.2 多模块工程如何组织？构建顺序怎么确定？

**构建顺序**：reactor 读取所有模块的 `<dependency>` 建立**依赖图并拓扑排序**——shop-service 依赖 shop-api，则 api 先构建；**与 `<modules>` 的书写顺序无关**；出现环直接报错（与 2.1.1 同理）。

**标准分层模板**（Java 微服务单体仓库惯例）：

```text
shop-parent(pom)
 ├─ shop-common      # 工具/常量/统一返回体，无业务
 ├─ shop-api         # DTO/接口定义/Feign 契约（被服务方与调用方共享，见 18_feign.md）
 ├─ shop-domain      # 实体与仓储接口
 ├─ shop-service     # 业务实现
 └─ shop-web         # 启动装配、Controller
```

**治理要点**：

1. 版本单点：根 POM `dependencyManagement` 钉三方与**内部模块**版本（`<shop-api.version>${project.version}</shop-api.version>`），子模块零版本；
2. 内部模块互引**不写 version 时继承 parent**，写死版本会漂移；
3. **`-pl`/`-am` 增量构建**：`mvn -pl shop-api,shop-service -am package`——只构建指定模块及其上游依赖，CI 提速标配（配合 diff 检测变更模块，见 33_cicd.md）；
4. **扁平化与统一版本**：`${revision}` 占位符管理全工程版本需配 `flatten-maven-plugin`（发布时把占位符固化进 POM，否则下游解析拿到字面量 `${revision}` 报错——高频事故）；`versions:set` 批量改版本；
5. 发布纪律：库模块才 deploy，启动模块（含 main 的 web）通常只 package 出 Fat Jar 不 deploy；
6. "IDE 能编译、命令行失败"：IDE 的 workspace resolution 掩盖了模块未 install/reactor 构建差异——CI 永远在根跑 `mvn -B verify`。

## 3.2 Profile 与多环境

### 3.2.1 如何用 Maven Profile 实现多环境配置？和 Spring 的 profile 怎么配合？

**Maven Profile = 构建期开关**，按激活条件切换一组属性/依赖/插件/仓库：

```xml
<profiles>
    <profile>
        <id>dev</id>
        <activation><activeByDefault>true</activeByDefault></activation>
        <properties><env>dev</env><db.url>jdbc:mysql://dev-db:3306/app</db.url></properties>
    </profile>
    <profile>
        <id>prod</id>
        <properties><env>prod</env><db.url>jdbc:mysql://prod-db:3306/app</db.url></properties>
    </profile>
</profiles>
```

**激活途径**：命令行 `-P prod`（可多值逗号分隔）；settings.xml `<activeProfiles>`；`<activation>` 自动条件（`activeByDefault`/`property`/`jdk`/`os`）。注意 `activeByDefault` **一旦命令行指定了任何其他 profile 即失效**。

**资源过滤（filtering）接线**：

```xml
<build>
    <resources>
        <resource>
            <directory>src/main/resources</directory>
            <filtering>true</filtering>     <!-- 启用 ${} 替换 -->
        </resource>
    </resources>
</build>
```

`application.properties` 写 `spring.profiles.active=${env}`，`mvn package -Pprod` 出包即烙死 prod——这是两套 profile 体系的配合方式。

**两套 profile 辨析（高频追问）**：Maven profile 是**构建期**文本替换，产物一出包就固化；Spring profile（`spring.profiles.active`，见 10_springboot.md 3.1.2）是**运行期**装配开关。现代实践更推荐"**一次构建、到处运行**"：镜像只打一次，环境差异用环境变量/配置中心运行期注入（`SPRING_PROFILES_ACTIVE`、Nacos，见 17_nacos.md）——Maven 过滤方案会导致每个环境出不同包，审计与回滚都困难；两者别混用同一占位符机制。另注意 filtering 会破坏二进制资源（图片/证书乱码），需 exclude 或只过滤 properties/yml。

# 4. 专题与对比

## 4.1 打包与发布

### 4.1.1 spring-boot-maven-plugin 打出的可执行 Fat Jar 是什么原理？

**机制**：插件的 `repackage` 目标绑定在 **package 阶段**——maven-jar-plugin 先产出普通 jar（被改名为 `.jar.original`），repackage 在其上重建布局：

```text
META-INF/MANIFEST.MF     → Main-Class: JarLauncher（Boot 3.x 在 launcher 包）
                            Start-Class: com.example.Application
BOOT-INF/classes/        → 应用自身 class 与资源
BOOT-INF/lib/            → 全部依赖 jar（嵌套，不解压）
org/springframework/boot/loader/  → 引导代码与 LaunchedClassLoader
```

**启动**：`java -jar` 只认 MANIFEST 的 Main-Class → JarLauncher 用自定义 ClassLoader 加载 BOOT-INF/lib 嵌套 jar（突破"标准 ClassLoader 读不了 jar 内 jar"限制）→ 反射调用 Start-Class 的 main。**完整链路见 10_springboot.md 2.2.1**——本题从 Maven 视角要答出的是：谁在什么阶段做了什么（repackage@package）、两个 Class 的分工、以及 war 切换（`<layout>WAR</layout>`/starter-web 改 provided + 继承 SpringBootServletInitializer，见 10_springboot.md 2.2.2）。

**追问**：为什么 Fat Jar 不能再被别的工程当依赖引？——业务类在 BOOT-INF/classes 里，普通 ClassLoader 不可见；需要"可依赖 + 可执行"双产物时配 `<classifier>exec</classifier>` 让原始 jar 保留主坐标。

### 4.1.2 把一个库发布到私服/中央仓库，完整流程是什么？

**私服发布**：

```xml
<!-- 父 POM -->
<distributionManagement>
    <repository>
        <id>nexus-releases</id>
        <url>https://nexus.example.com/repository/maven-releases/</url>
    </repository>
    <snapshotRepository>
        <id>nexus-snapshots</id>
        <url>https://nexus.example.com/repository/maven-snapshots/</url>
    </snapshotRepository>
</distributionManagement>
```

settings.xml 配 `<server><id>nexus-releases>` 凭据（id 三者必须一致：repository/server，见 2.3.3）→ `mvn deploy`。版本含 `-SNAPSHOT` 自动走 snapshotRepository（可重复发、时间戳版本），否则走 releases（不可覆盖）。

**中央仓库（Sonatype Central Portal）**门槛更高：POM 必须齐全 `name/description/url/licenses/scm/developers`；GPG **签名**（gpg-maven-plugin）；**sources + javadoc** 两个 classifier jar（对应插件 attach）；发布到 staging 人工核对后 release（或 CI 一键）。全部自动化后这些挂在 `mvn deploy` 一条命令上——理解"deploy 远不止上传 jar"即满分。实践替代：内部构件走私服 + 版本矩阵 BOM（2.2.2），对外开源才走中央。

## 4.2 Maven vs Gradle 与版本一致性

### 4.2.1 Gradle 相比 Maven 有哪些优势？如何选型？

| 维度 | Maven | Gradle |
|------|-------|--------|
| 脚本 | XML 声明式，表达力弱（逻辑靠插件） | **Groovy/Kotlin DSL 即代码**，可写逻辑可复用 |
| 性能 | 每次全量分析、无缓存复用 | **增量构建**（跟踪任务输入输出）+ **构建缓存**（本地/远程共享）+ daemon 常驻 JVM——大工程快 2~10 倍 |
| 配置模型 | 约定极强、自由度高的小项目样板厚 | 灵活但约定弱，样板薄 |
| 依赖声明 | scope 六件套 | `implementation`（编译+运行、**不传递给消费者**）/`api`（传递）/`compileOnly`/`runtimeOnly`/`testImplementation`——**api/implementation 分离编译类路径**，升级传递依赖不引发全工程重编 |
| 生态 | 插件齐全、文档海量、企业惯性 | Android 官方绑定；插件质量参差 |
| 学习曲线 | 平缓 | DSL+生命周期+插件开发更陡 |

**选型**：Android/大型多模块/构建性能敏感 → Gradle；后端 Java 企业项目、团队统一规范、存量工程 → Maven 完全够用（瓶颈在依赖下载时上镜像比换工具划算，见 2.3.2）。**面试加分点**：implementation/api 的语义 Maven 无对应（provided 的"对消费者隐藏"变体，类似 optional 但作用于编译类路径），答出这层对比说明真懂两套模型。Boot 3.x 两栈均官方支持，构建工具选择不是架构问题，迁移成本才是。

### 4.2.2 Wrapper 是什么？为什么强调锁定插件版本？

**Wrapper = 把构建工具版本本身纳入版本管理**：仓库带 `mvnw`/`mvnw.cmd`（或 `gradlew`）脚本 + `.mvn/wrapper/maven-wrapper.properties`（distributionUrl 指定 Maven 版本，Gradle 为 `gradle/wrapper/gradle-wrapper.properties`）——任何人/CI 执行 wrapper 脚本，首次自动下载**指定版本**的发行版到 `~/.m2/wrapper`，无需预装、团队/CI 永远同一构建器。Spring Initializr 生成的项目默认带 mvnw，新标配认知。

**锁定插件版本同理**："本地能编、CI 报错"的经典根因——**不写版本的插件按 Maven 内置 super POM/仓库最新解析**，不同环境默认插件版本不一致，行为/默认参数漂移。治理：父 POM `pluginManagement` 给**全部**用到的插件钉版本（compiler/jar/surefire/failsafe/…），配合 `maven-enforcer-plugin` 的 `requirePluginVersions` 规则强制声明；依赖版本则集中 dependencyManagement/BOM（2.2.1/2.2.2）。**一句话总结本题**：可重现构建（Reproducible Build）三要素——构建器版本（wrapper）+ 插件版本（pluginManagement）+ 依赖版本（BOM/lock），缺一环就有"薛定谔的构建"。