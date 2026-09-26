
# 1. Gradle 安装方式

## 1.1 JDK 要求

Gradle 运行在 JVM 上，因此必须安装 JDK。不同版本的 Gradle 对 JDK 版本有不同要求：

| Gradle 版本 | 最低 JDK | 推荐 JDK | 备注 |
|-------------|----------|----------|------|
| 7.x | JDK 8 | JDK 11+ | 可编译 Java 16 以下 |
| 8.0-8.4 | JDK 11 | JDK 17+ | Android 开发推荐 JDK 17 |
| 8.5+ | JDK 17 | JDK 21 | 支持 Java 21 特性 |

验证 JDK 安装：
```bash
java -version
# openjdk version "21.0.1" 2023-10-17 LTS
# OpenJDK Runtime Environment (build 21.0.1+12-LTS)
# OpenJDK 64-Bit Server VM (build 21.0.1+12-LTS, mixed mode, sharing)
```

Gradle 使用 `JAVA_HOME` 环境变量查找 JDK，也可以通过 `gradle.properties` 中的 `org.gradle.java.home` 属性指定：

```properties
# gradle.properties
org.gradle.java.home=/usr/lib/jvm/java-21-openjdk
```

## 1.2 方法一：SDKMAN!（推荐，Unix/MacOS/Cygwin/WSL）

SDKMAN! 是最推荐的 Gradle 安装方式，因为它可以管理多个版本并在版本之间轻松切换。

```bash
# 安装 SDKMAN!
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"

# 安装 Gradle
sdk install gradle 8.5

# 查看可用的 Gradle 版本
sdk list gradle

# 切换到其他版本
sdk use gradle 7.6.3

# 设为默认版本
sdk default gradle 8.5

# 查看当前使用的版本
sdk current gradle
# Using gradle version 8.5
```

## 1.3 方法二：Homebrew（macOS / Linux）

```bash
# 安装
brew install gradle

# 查看版本
gradle --version

# 升级
brew upgrade gradle
```

## 1.4 方法三：Chocolatey（Windows）

```powershell
# 以管理员身份运行 PowerShell
choco install gradle

# 查看版本
gradle --version

# 升级
choco upgrade gradle
```

## 1.5 方法四：手动安装（所有平台）

```bash
# 1. 从官网下载：https://gradle.org/releases/
wget https://services.gradle.org/distributions/gradle-8.5-bin.zip

# 2. 解压到目标目录
sudo unzip -d /opt/gradle gradle-8.5-bin.zip

# 3. 配置环境变量（Linux/macOS，添加到 ~/.bashrc 或 ~/.zshrc）
export GRADLE_HOME=/opt/gradle/gradle-8.5
export PATH=$GRADLE_HOME/bin:$PATH

# 4. 使配置生效
source ~/.bashrc

# 5. 验证
gradle --version
```

Windows 手动安装步骤：
```
1. 下载 gradle-8.5-bin.zip
2. 解压到 C:\gradle\gradle-8.5
3. 创建环境变量 GRADLE_HOME = C:\gradle\gradle-8.5
4. 在 PATH 中添加 %GRADLE_HOME%\bin
5. 打开新的命令行窗口，运行 gradle --version
```

## 1.6 Gradle Wrapper——推荐的项目级使用方式

在实际项目开发中，**完全不推荐全局安装 Gradle**。最佳实践是使用 Gradle Wrapper，它是项目捆绑的一个小脚本，确保每个项目使用精确的 Gradle 版本。

Wrapper 的核心文件：
```
project/
├── gradlew                    ← Unix shell 脚本
├── gradlew.bat                ← Windows 批处理脚本
├── gradle/
│   └── wrapper/
│       ├── gradle-wrapper.jar         ← Wrapper JAR（引导下载）
│       └── gradle-wrapper.properties  ← 版本配置
```

`gradle-wrapper.properties` 内容：
```properties
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-8.5-bin.zip
networkTimeout=10000
validateDistributionUrl=true
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
```

当你执行 `./gradlew build` 时：
1. Wrapper 脚本检查本地是否已缓存了指定版本的 Gradle
2. 如果没有，从 `distributionUrl` 下载
3. 使用该版本的 Gradle 执行构建

**生成 Wrapper：**
```bash
# 如果已安装 Gradle
gradle wrapper --gradle-version 8.5

# 指定更具体的参数
gradle wrapper --gradle-version 8.5 --distribution-type all

# distribution-type 选项：
#   bin     - 仅运行时（最小，推荐）
#   all     - 包含源码和文档（适合 IDE）
```

## 1.7 升级 Wrapper 版本

```bash
# 方法1: 使用 wrapper task
./gradlew wrapper --gradle-version 8.6

# 方法2: 直接修改 gradle-wrapper.properties
# distributionUrl=https\://services.gradle.org/distributions/gradle-8.6-bin.zip
```

# 2. gradle init —— 快速创建项目

## 2.1 命令概述

`gradle init` 是 Gradle 提供的项目脚手架命令，可以为不同类型的项目生成初始构建文件。

```bash
gradle init
```

交互式向导会询问：
1. 项目类型（application, library, Gradle plugin, basic 等）
2. 构建脚本 DSL（Groovy 或 Kotlin）
3. 测试框架（JUnit 4, JUnit Jupiter, TestNG, Spock, Kotlin Test）
4. 项目名称和源码包名

## 2.2 可生成的项目类型

| 类型参数 | 说明 | 生成内容 |
|----------|------|----------|
| `basic` | 基本 Gradle 项目 | 仅生成 Wrapper 和 settings |
| `java-application` | Java 控制台应用 | 含主类、测试、application 插件 |
| `java-library` | Java 库 | 包含 `api()` 依赖配置 |
| `java-gradle-plugin` | Gradle 插件项目 | 插件元数据、功能测试 |
| `groovy-application` | Groovy 控制台应用 | Groovy 源码目录 |
| `groovy-library` | Groovy 库 | Groovy 库结构 |
| `kotlin-application` | Kotlin 控制台应用 | Kotlin 源码目录 |
| `kotlin-library` | Kotlin 库 | Kotlin 库结构 |
| `scala-application` | Scala 控制台应用 | Scala 源码目录 |
| `pom` | 从 Maven POM 转换 | 解析 pom.xml 生成 Gradle 项目 |

## 2.3 非交互式创建项目

```bash
# Java 库 + Kotlin DSL + JUnit Jupiter
gradle init \
  --type java-library \
  --dsl kotlin \
  --test-framework junit-jupiter \
  --project-name my-lib \
  --package com.example.mylib

# Java 应用 + Groovy DSL + JUnit 4
gradle init \
  --type java-application \
  --dsl groovy \
  --test-framework junit \
  --project-name my-app \
  --package com.example.myapp
```

## 2.4 从 Maven 迁移

```bash
# 在包含 pom.xml 的目录执行
gradle init --type pom
```

生成的 `build.gradle` 会自动转换：
- `groupId`、`artifactId`、`version`
- `dependencies` 块（Maven 依赖 → Gradle 依赖声明）
- 仓库配置
- 编译器设置转换

# 3. 第一个 build.gradle

## 3.1 Java 应用项目示例

**完整的 build.gradle.kts（Kotlin DSL）：**
```kotlin
// 插件声明
plugins {
    java
    application
}

// 项目元数据
group = "com.example"
version = "1.0.0"

// Java 兼容性设置
java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}

// 仓库配置
repositories {
    mavenCentral()
    google()
}

// 依赖声明
dependencies {
    // 核心依赖（编译和运行时都可见）
    implementation("com.google.guava:guava:32.1.3-jre")
    implementation("org.slf4j:slf4j-api:2.0.9")

    // 仅在编译时可见（运行时由容器提供）
    compileOnly("org.projectlombok:lombok:1.18.30")
    annotationProcessor("org.projectlombok:lombok:1.18.30")

    // 测试依赖
    testImplementation("org.junit.jupiter:junit-jupiter:5.10.1")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}

// 应用插件配置
application {
    mainClass.set("com.example.App")
}

// 任务配置
tasks.test {
    useJUnitPlatform()
    testLogging {
        events("passed", "skipped", "failed")
    }
}

// 自定义任务
tasks.register<Copy>("copyDependencies") {
    from(configurations.runtimeClasspath)
    into(layout.buildDirectory.dir("libs"))
}
```

**等价的 build.gradle（Groovy DSL）：**
```groovy
plugins {
    id 'java'
    id 'application'
}

group = 'com.example'
version = '1.0.0'

java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}

repositories {
    mavenCentral()
    google()
}

dependencies {
    implementation 'com.google.guava:guava:32.1.3-jre'
    implementation 'org.slf4j:slf4j-api:2.0.9'

    compileOnly 'org.projectlombok:lombok:1.18.30'
    annotationProcessor 'org.projectlombok:lombok:1.18.30'

    testImplementation 'org.junit.jupiter:junit-jupiter:5.10.1'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

application {
    mainClass = 'com.example.App'
}

tasks.named('test') {
    useJUnitPlatform()
    testLogging {
        events 'passed', 'skipped', 'failed'
    }
}
```

## 3.2 项目目录结构约定

```
my-app/
├── build.gradle.kts              ← 构建脚本（项目级）
├── settings.gradle.kts           ← 设置脚本
├── gradle.properties             ← Gradle 属性
├── gradlew / gradlew.bat         ← Wrapper 脚本
├── gradle/
│   └── wrapper/
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradle/
│   └── libs.versions.toml        ← 版本目录（可选）
├── src/
│   ├── main/
│   │   ├── java/                 ← 生产 Java 源代码
│   │   │   └── com/example/App.java
│   │   └── resources/            ← 生产资源文件
│   │       └── application.properties
│   └── test/
│       ├── java/                 ← 测试 Java 源代码
│       │   └── com/example/AppTest.java
│       └── resources/            ← 测试资源文件
│           └── test-config.properties
└── build/                        ← 构建输出（由 Gradle 生成）
    ├── classes/                  ← 编译后的 .class 文件
    ├── libs/                     ← JAR/WAR 包
    ├── reports/                  ← 测试报告、checkstyle 等
    └── tmp/                      ← 临时文件
```

## 3.3 关键目录约定

| 目录 | 说明 | 可覆盖 |
|------|------|--------|
| `src/main/java` | 生产 Java 源文件 | `sourceSets.main.java.srcDirs` |
| `src/main/resources` | 生产资源文件 | `sourceSets.main.resources.srcDirs` |
| `src/test/java` | 测试 Java 源文件 | `sourceSets.test.java.srcDirs` |
| `src/test/resources` | 测试资源文件 | `sourceSets.test.resources.srcDirs` |
| `build` | 所有构建输出 | `layout.buildDirectory` |
| `build/classes` | 编译后的 class | 由编译任务自动管理 |
| `build/libs` | 打包产物 | jar 等任务管理 |
| `build/reports` | 报告输出 | test、checkstyle 等任务 |

# 4. Gradle 基本命令

## 4.1 gradle tasks —— 查看可用任务

```bash
# 列出所有任务（按分组显示）
./gradlew tasks

# 列出所有任务（包括内部任务）
./gradlew tasks --all

# 仅列出指定分组的任务
./gradlew tasks --group="build"
./gradlew tasks --group="verification"
```

典型输出：
```
Build tasks
-----------
assemble - Assembles the outputs of this project.
build - Assembles and tests this project.
clean - Deletes the build directory.

Build Setup tasks
-----------------
init - Initializes a new Gradle build.
wrapper - Generates Gradle wrapper files.

Documentation tasks
-------------------
javadoc - Generates Javadoc API documentation for the main source code.

Verification tasks
------------------
check - Runs all checks.
test - Runs the test suite.
```

## 4.2 gradle build —— 执行构建

```bash
# 执行完整构建（编译 + 测试 + 打包）
./gradlew build

# 仅编译，不测试
./gradlew assemble

# 仅执行测试
./gradlew test

# 清理构建产物
./gradlew clean

# 清理后重新构建
./gradlew clean build

# 并行构建（多核 CPU）
./gradlew build --parallel

# 离线模式（不检查远程仓库更新）
./gradlew build --offline

# 跳过测试
./gradlew build -x test

# 跳过多个任务
./gradlew build -x test -x checkstyleMain

# 仅构建指定子项目
./gradlew :module-a:build
```

## 4.3 详细输出和调试

```bash
# 输出信息级别（INFO）
./gradlew build --info

# 输出调试级别（DEBUG）—— 极其详细
./gradlew build --debug

# 输出堆栈跟踪（构建失败时）
./gradlew build --stacktrace

# 输出完整堆栈跟踪（包括内部异常链）
./gradlew build --full-stacktrace

# 静默模式（仅显示错误）
./gradlew build --quiet

# 完全不输出（除非有错误）
./gradlew build -q

# 生成构建扫描报告
./gradlew build --scan

# 性能分析（显示每个任务耗时）
./gradlew build --profile
# 生成 HTML 报告：build/reports/profile/

# 显示将要执行的任务列表（不实际执行）
./gradlew build --dry-run
```

## 4.4 任务执行控制

```bash
# 强制执行（忽略 UP-TO-DATE 检查，重新执行所有任务）
./gradlew build --rerun-tasks

# 连续构建模式（监视文件变化，自动重新构建）
./gradlew build --continuous
# 等价缩写：./gradlew build -t

# 刷新依赖缓存（强制重新解析依赖）
./gradlew build --refresh-dependencies

# 配置缓存相关
./gradlew build --no-configuration-cache  # 禁用配置缓存

# 配置守护进程
./gradlew --stop       # 停止所有守护进程
./gradlew --status     # 查看守护进程状态
```

## 4.5 gradle properties —— 查看属性

```bash
# 列出项目所有属性
./gradlew properties

# 查看特定属性
./gradlew properties | rg "version"
```

典型输出：
```
Root project 'my-app'
-------------------------------------------------------------
allprojects: [root project 'my-app']
ant: org.gradle.api.internal.project.DefaultAntBuilder@...
buildDir: /path/to/my-app/build
group: com.example
name: my-app
version: 1.0.0
...
```

## 4.6 gradle help

```bash
# 查看某个任务的帮助信息
./gradlew help --task build

# 查看 Gradle 版本信息
./gradlew --version
```

# 5. Gradle Wrapper 深入理解

## 5.1 Wrapper 文件详解

**gradlew（Unix Shell 脚本）核心功能：**
```bash
#!/bin/sh
# ... 省略大部分内容 ...

# 关键变量
APP_NAME="Gradle"
APP_BASE_NAME=`basename "$0"`
DEFAULT_JVM_OPTS='"-Xmx64m" "-Xms64m"'

# 寻找 JAVA_HOME
if [ -n "$JAVA_HOME" ] ; then
    JAVACMD="$JAVA_HOME/bin/java"
fi

# 下载并解压 Gradle 发行版
# 使用 gradle-wrapper.jar 中的 Downloader 类
# 将 Gradle 安装到 GRADLE_USER_HOME/wrapper/dists/
```

**gradlew.bat（Windows 批处理脚本）：**
```batch
@rem 关键变量
set DEFAULT_JVM_OPTS="-Xmx64m" "-Xms64m"
set DIRNAME=%~dp0

@rem 查找 java.exe
if defined JAVA_HOME goto findJavaFromJavaHome
```

## 5.2 Wrapper JAR 的工作原理

`gradle-wrapper.jar` 是一个精简的引导 JAR，包含：
- `WrapperMain.java`：入口类，负责解析属性和下载
- `Downloader.java`：HTTP 下载组件
- `Unpacker.java`：ZIP 解压组件
- `Logger.java`：日志组件

工作流程：
```
./gradlew build
    │
    ├─► gradlew 脚本
    │     ├─► 查找 Java 可执行文件
    │     ├─► 构建 classpath（指向 gradle-wrapper.jar）
    │     └─► 启动 java -jar gradle-wrapper.jar
    │
    └─► WrapperMain
          ├─► 读取 gradle-wrapper.properties
          ├─► 检查 GRADLE_USER_HOME/wrapper/dists/ 缓存
          ├─► 如果未缓存：下载 distributionUrl 指定的 ZIP
          ├─► 解压到缓存目录
          └─► 启动解压后的 Gradle 进程
                └─► 传递原始参数（build）到 Gradle 主类
```

## 5.3 GRADLE_USER_HOME

`GRADLE_USER_HOME`（默认 `~/.gradle`）存储了 Gradle 的所有全局缓存：

```
~/.gradle/
├── caches/                          ← 全局缓存
│   ├── modules-2/                    ← 依赖 JAR 缓存
│   │   ├── files-2.1/               ← 下载的依赖文件
│   │   └── metadata-2.96/           ← 依赖元数据
│   ├── build-cache-1/               ← 本地构建缓存
│   ├── journal-1/                   ← 文件系统监测日志
│   └── jars-9/                      ← 工具 JAR 缓存
├── wrapper/
│   └── dists/                       ← Wrapper 下载的 Gradle 发行版
│       └── gradle-8.5-bin/
├── daemon/                          ← 守护进程元数据
│   └── 8.5/                         ← 按版本分组
├── notifications/                   ← 守护进程通知
├── jdks/                            ← 自动下载的 JDK
└── gradle.properties                ← 全局 Gradle 属性
```

## 5.4 Wrapper 的最佳实践

1. **始终将 Wrapper 文件提交到版本控制**
   ```bash
   git add gradlew gradlew.bat gradle/wrapper/
   git commit -m "Add Gradle Wrapper"
   ```

2. **使用绝对路径指定 distributionUrl**
   ```properties
   # 标准 URL（不需要认证）
   distributionUrl=https\://services.gradle.org/distributions/gradle-8.5-bin.zip
   ```

3. **使用 SHA-256 校验和验证**
   ```properties
   distributionSha256Sum=8b9a3206a2bf4b4e6a43a7d7e0a54b3a8e5c1d6f7b8a9c0d1e2f3a4b5c6d7e8f
   ```

4. **在 CI 中使用缓存**
   ```yaml
   # GitHub Actions 示例
   - uses: actions/cache@v3
     with:
       path: |
         ~/.gradle/caches
         ~/.gradle/wrapper
       key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
   ```

# 6. IDE 集成

## 6.1 IntelliJ IDEA

IntelliJ IDEA 对 Gradle 具有最好的集成支持：

1. **打开 Gradle 项目**：File → Open → 选择 `build.gradle(.kts)` 或项目根目录
2. **Gradle 工具窗口**：View → Tool Windows → Gradle
3. **自动导入**：Settings → Build, Execution, Deployment → Build Tools → Gradle

```properties
# gradle.properties（IDEA 专用配置）
# 使用 IDEA 自带的 JDK 进行构建
org.gradle.java.home=/path/to/jdk

# 使用 Gradle 进行构建和运行（而非 IDEA 的内部构建器）
# Settings → Build Tools → Gradle → Build and run using: Gradle
```

Gradle 工具窗口提供：
- 项目树和所有 Task 的列表
- 双击执行任何 Task
- Task 执行日志
- 依赖分析视图
- 构建刷新按钮

## 6.2 Eclipse（Buildship）

Eclipse 通过 Buildship 插件支持 Gradle：

1. Help → Eclipse Marketplace → 搜索 "Buildship Gradle"
2. File → Import → Gradle → Existing Gradle Project
3. 选择项目根目录 → Finish

Buildship 特性：
- Gradle Task 视图（Window → Show View → Gradle Tasks）
- Gradle Executions 视图（显示历史执行）
- 自动同步（Preferences → Gradle → Automatic Project Synchronization）

## 6.3 Visual Studio Code

VS Code 通过 Language Support for Java 和 Gradle for Java 扩展提供支持：

```bash
# 安装推荐扩展
code --install-extension vscjava.vscode-java-pack
code --install-extension vscjava.vscode-gradle
```

VS Code Gradle 功能：
- Gradle 项目视图（侧边栏 Gradle 图标）
- Task 执行和输出
- 依赖管理（添加/移除依赖的 GUI）
- Kotlin DSL 语法高亮和智能提示

## 6.4 IDE 同步问题排查

常见的 IDE 同步问题及解决方案：

```bash
# 问题：IDEA 无法解析依赖
# 解决：刷新 Gradle 项目 + 使缓存失效
# IDEA: File → Invalidate Caches... → Invalidate and Restart

# 问题：依赖下载失败
# 解决：检查网络 + 代理配置
# gradle.properties
systemProp.http.proxyHost=proxy.company.com
systemProp.http.proxyPort=8080
systemProp.https.proxyHost=proxy.company.com
systemProp.https.proxyPort=8443

# 问题：Kotlin DSL 没有代码补全
# 解决：确保使用 IDE 自带的 Gradle 或正确配置 JDK
```

# 7. gradle.properties 配置详解

## 7.1 属性文件的位置和优先级

Gradle 按以下优先级查找属性（高 → 低）：

```
1. 命令行 -D 参数（-Dorg.gradle.daemon=false）
2. 命令行 -P 参数（-Pversion=2.0）→ 项目属性
3. 项目根目录 gradle.properties（版本控制）
4. GRADLE_USER_HOME/gradle.properties（用户级全局）
5. 环境变量（ORG_GRADLE_* 前缀）
```

## 7.2 常用 Gradle 内置属性

```properties
# === JVM 配置 ===
# Gradle 守护进程和 Worker 进程的 JVM 参数
org.gradle.jvmargs=-Xmx2048m -XX:MaxMetaspaceSize=512m -XX:+HeapDumpOnOutOfMemoryError

# Gradle 使用的 JDK 路径（优先级高于 JAVA_HOME）
# org.gradle.java.home=/path/to/jdk

# === 守护进程配置 ===
# 是否使用守护进程（默认 true）
org.gradle.daemon=true
# 守护进程空闲存活时间（毫秒，默认 3 小时）
org.gradle.daemon.idletimeout=10800000

# === 构建缓存配置 ===
# 启用本地构建缓存
org.gradle.caching=true

# 启用配置缓存（实验性）
org.gradle.configuration-cache=true

# === 并行和性能 ===
# 并行执行任务（多项目构建中，默认 false）
org.gradle.parallel=true

# 并行 Worker 数量（默认 CPU 核心数）
org.gradle.workers.max=4

# 配置时按需创建项目（大型多项目构建，默认 false）
org.gradle.configureondemand=true

# 启用文件系统监视（7.0+）
org.gradle.vfs.watch=true

# === 日志 ===
# 控制台输出模式：auto | plain | rich | verbose
org.gradle.console=verbose

# 日志级别：quiet | warn | lifecycle | info | debug
org.gradle.logging.level=lifecycle

# === 仓库和代理 ===
# 代理配置
# systemProp.http.proxyHost=proxy.example.com
# systemProp.http.proxyPort=8080
# systemProp.https.proxyHost=proxy.example.com
# systemProp.https.proxyPort=8443
# systemProp.http.nonProxyHosts=*.local|localhost

# === 依赖管理 ===
# 检查依赖冲突时失败
# org.gradle.dependency.verification=strict

# === 构建扫描 ===
# 自动接受使用条款以生成构建扫描
org.gradle.scans.accept=true
```

## 7.3 自定义项目属性

```properties
# 在 gradle.properties 中定义自定义属性
myAppName=MyApplication
myAppVersion=2.0.0
deployTarget=staging
maxRetryCount=3
```

在构建脚本中使用：
```kotlin
// build.gradle.kts - 读取属性
val appName: String by project
val deployTarget: String by project

println("应用名称: $appName")
println("部署目标: $deployTarget")
```

# 8. 初学者的常见问题与解决

## 8.1 依赖下载缓慢

```bash
# 配置国内镜像（阿里云 Maven 仓库）
```

```kotlin
// settings.gradle.kts（注意：这是项目级配置，放在 settings 中）
dependencyResolutionManagement {
    repositories {
        maven { url = uri("https://maven.aliyun.com/repository/public") }
        maven { url = uri("https://maven.aliyun.com/repository/central") }
        maven { url = uri("https://maven.aliyun.com/repository/google") }
        maven { url = uri("https://maven.aliyun.com/repository/gradle-plugin") }
    }
}
```

```groovy
// build.gradle
repositories {
    maven { url 'https://maven.aliyun.com/repository/public' }
    maven { url 'https://maven.aliyun.com/repository/central' }
    mavenCentral()  // 作为 fallback
}
```

## 8.2 守护进程占用内存过大

```properties
# gradle.properties
org.gradle.jvmargs=-Xmx1024m -Xms256m
# 如果构建完成后需要释放内存
org.gradle.daemon.idletimeout=300000  # 5 分钟
```

## 8.3 无法找到或加载主类

```bash
# 确保 application 插件正确配置
```

```kotlin
application {
    // Kotlin DSL 中的主类配置
    mainClass.set("com.example.MainKt")  // 注意 Kt 后缀（Kotlin 顶层函数）
}

// 或使用 application 块
application {
    mainClass.set("com.example.App")
}
```

## 8.4 Gradle 版本与 JDK 不匹配

```bash
# 错误信息示例
# "Java 21 is required to run Gradle 8.5. Your Java version is 8"

# 解决：升级 JDK 或降级 Gradle 版本
java -version
# 查看当前使用的 JDK

# 指定 JDK
# gradle.properties
# org.gradle.java.home=C:\\Program Files\\Java\\jdk-21
```
