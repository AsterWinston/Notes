
# 1. 迁移动机与策略

## 1.1 为什么要迁移

| 维度 | Maven | Gradle |
|------|-------|--------|
| 构建性能 | 顺序执行，不易并行 | 增量构建，并行执行，构建缓存 |
| 灵活性 | XML 固定生命周期 | 可编程构建逻辑，Groovy/Kotlin DSL |
| 依赖管理 | 传递依赖，版本仲裁简单 | 丰富的依赖配置，变体感知 |
| 构建逻辑复用 | parent POM 继承 | 约定插件，包含构建，版本目录 |
| IDE 支持 | 成熟稳定 | 良好，持续改进 |
| 多模块项目 | 基于 POM 的模块聚合 | 原生多项目支持 |
| Plugin 生态 | 生命周期绑定 | 丰富的插件系统和 Task 模型 |

```xml
<!-- Maven 项目的典型痛点 -->
<!-- 1. XML 膨胀 -->
<project>
    <build>
        <plugins>
            <plugin>
                <!-- 大量配置 -->
                <configuration>...</configuration>
                <executions>...</executions>
            </plugin>
        </plugins>
    </build>
    <!-- profile 导致配置复杂 -->
    <profiles>
        <profile>...</profile>
    </profiles>
</project>
```

```groovy
// Gradle 的对应方案
plugins {
    id 'java'
}

// 配置清晰，可编程
tasks.named('test') {
    useJUnitPlatform()
    maxParallelForks = Runtime.runtime.availableProcessors()
}
```

## 1.2 迁移策略选择

### 1.2.1 大爆炸迁移 (Big Bang)

```
策略：一次性将所有模块从 Maven 迁移到 Gradle
优点：完成后统一，无需维护两套构建系统
缺点：风险高，停机时间长，回归测试工作量巨大
适用：小型项目 (< 5 个模块)，构建相对简单
```

### 1.2.2 增量迁移 (Incremental)

```
策略：逐模块迁移，每次一个模块或一个功能域
优点：风险可控，可逐步验证，业务不中断
缺点：需要临时桥接机制，迁移周期长
适用：大中型项目 (> 5 个模块)，生产系统
```

```groovy
// 增量迁移的桥接方案
// 阶段1：Maven 构建产出 JAR 到本地仓库
// 阶段2：Gradle 模块声明对 Maven 产出的依赖
dependencies {
    implementation("com.example:legacy-module:1.0.0") // 仍由 Maven 构建
    implementation(project(":new-gradle-module"))       // 已迁移的模块
}

// 阶段3：完全迁移后移除 Maven 依赖
```

## 1.3 使用 gradle init 自动转换

```bash
# 在 Maven 项目根目录执行
gradle init --type java-application    # Java 应用
gradle init --type java-library        # Java 库
gradle init --type pom                 # 基于 pom.xml 生成

# 转换过程：
# 1. 读取 pom.xml
# 2. 生成 settings.gradle (多模块识别)
# 3. 生成 build.gradle (依赖和插件映射)
# 4. 生成 gradle wrapper
# 5. 可选：生成 .gitignore

# gradle init 的限制：
# - 复杂 profile 不会完全转换
# - 自定义 Maven 插件可能无法映射
# - 需要手动调整生成结果
# - resource filtering 需要重写
```

# 2. POM 到 Gradle 的结构映射

## 2.1 单模块项目映射

```xml
<!-- pom.xml -->
<project>
    <groupId>com.example</groupId>
    <artifactId>my-app</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>
    
    <dependencies>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
            <version>3.13.0</version>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.11.0</version>
                <configuration>
                    <source>17</source>
                    <target>17</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

```groovy
// build.gradle
plugins {
    id 'java'
}

group = 'com.example'
version = '1.0.0'

java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.apache.commons:commons-lang3:3.13.0'
}
```

## 2.2 多模块项目映射 (POM 聚合)

```xml
<!-- 父 POM (pom.xml) -->
<project>
    <groupId>com.example</groupId>
    <artifactId>parent</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>
    
    <modules>
        <module>core</module>
        <module>api</module>
        <module>web</module>
    </modules>
    
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>3.2.0</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

```groovy
// settings.gradle (替代 module 声明 + rootProject 定义)
rootProject.name = 'parent'

include 'core'
include 'api'
include 'web'
```

```groovy
// build.gradle (根项目，替代 parent POM)
allprojects {
    group = 'com.example'
    version = '1.0.0'
    
    repositories {
        mavenCentral()
    }
}

subprojects {
    apply plugin: 'java'
    
    java {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    
    dependencies {
        implementation platform('org.springframework.boot:spring-boot-dependencies:3.2.0')
        testImplementation 'org.junit.jupiter:junit-jupiter'
    }
    
    tasks.named('test') {
        useJUnitPlatform()
    }
}

// 使用约定插件替代 allprojects/subprojects (推荐方式)
// 创建 buildSrc/src/main/groovy/java-convention.gradle
// 然后各子项目：apply from: rootProject.file("gradle/java-convention.gradle")
```

## 2.3 Parent POM 继承映射

```xml
<!-- Maven parent POM 的继承 -->
<!-- child/pom.xml -->
<project>
    <parent>
        <groupId>com.example</groupId>
        <artifactId>parent</artifactId>
        <version>1.0.0</version>
    </parent>
    <artifactId>child</artifactId>
</project>
```

```groovy
// Gradle 没有 POM 继承概念，通过以下方式实现类似效果：

// 方式1：Convention Plugin (推荐)
// buildSrc/src/main/kotlin/my-java-convention.gradle.kts
plugins {
    java
}
java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}

// core/build.gradle.kts
plugins {
    id("my-java-convention")
}

// 方式2：subprojects 块 (简单场景)
// settings.gradle 根 build.gradle
subprojects {
    apply plugin: 'java'
    // 通用配置...
}

// 方式3：Included Build
// settings.gradle
includeBuild("shared-conventions")
```

# 3. 依赖作用域映射

## 3.1 Maven Scope → Gradle Configuration

| Maven Scope | Gradle Configuration | 说明 |
|------------|---------------------|------|
| `compile` (默认) | `api` / `implementation` | api: 暴露给消费者；implementation: 内部使用 |
| `runtime` | `runtimeOnly` | 仅在运行时需要 |
| `provided` | `compileOnly` | 编译时需要，不打入最终的包 |
| `test` | `testImplementation` | 仅测试编译和运行 |
| `test-runtime` | `testRuntimeOnly` | 仅测试运行时 |
| `system` | 不直接支持 | 需要手动管理本地 JAR |
| `import` (BOM) | `platform()` / `enforcedPlatform()` | BOM 依赖管理 |

```xml
<!-- Maven -->
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <!-- scope 默认为 compile -->
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <scope>provided</scope>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

```groovy
// Gradle
dependencies {
    implementation 'org.springframework:spring-context:6.1.0'    // compile
    compileOnly 'org.projectlombok:lombok:1.18.30'              // provided
    runtimeOnly 'org.postgresql:postgresql:42.7.1'              // runtime
    testImplementation 'org.junit.jupiter:junit-jupiter:5.10.1' // test
    
    annotationProcessor 'org.projectlombok:lombok:1.18.30'      // Maven 中无等价项
}
```

## 3.2 api vs implementation 的迁移决策

```groovy
// 这是 Maven 迁移中最容易出错的地方

// Maven compile scope 统一使用 implementation
// 除非该依赖的类型暴露在公共 API 中

plugins {
    id 'java-library'  // 注意：api 配置仅在 java-library 插件下可用
}

dependencies {
    // 内部实现依赖（不暴露给消费者）
    implementation 'com.google.guava:guava:33.0.0-jre'
    
    // 公共 API 依赖（暴露在方法签名、返回值中）
    api 'com.fasterxml.jackson.core:jackson-databind:2.16.0'
    
    // 判断规则：
    // 如果依赖类型出现在 public 方法签名或父类中 → api
    // 如果依赖仅在内部实现中使用 → implementation
}
```

## 3.3 Optional 依赖映射

```xml
<!-- Maven optional -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <optional>true</optional>
</dependency>
```

```groovy
// Gradle 没有直接对应的 optional
// 使用 Gradle 的 capability/feature variant 来实现类似效果

// 方式1：compileOnly (编译时需要，不打入传递依赖)
compileOnly 'org.springframework.boot:spring-boot-devtools:3.2.0'

// 方式2：特性变体 (Feature Variant) - 更精确
java {
    registerFeature("devtools") {
        usingSourceSet(sourceSets.main)
        capability("com.example", "my-lib-devtools", version)
    }
}

// 方式3：通过 configurations 声明
configurations {
    optional
}
dependencies {
    optional 'org.springframework.boot:spring-boot-devtools:3.2.0'
}
```

# 4. 插件映射

## 4.1 常见 Maven Plugin → Gradle Plugin

| Maven Plugin | Gradle Plugin / 替代方案 |
|-------------|------------------------|
| `maven-compiler-plugin` | `java` plugin, `JavaCompile` task |
| `maven-jar-plugin` | `java` plugin, `Jar` task |
| `maven-war-plugin` | `war` plugin |
| `maven-surefire-plugin` | `java` plugin, `Test` task + `useJUnitPlatform()` |
| `maven-failsafe-plugin` | 自定义 `integrationTest` sourceSet + `Test` task |
| `maven-resources-plugin` | `processResources` task |
| `maven-assembly-plugin` | `distribution` plugin, `Zip`/`Tar` tasks |
| `maven-dependency-plugin` | `configurations`, `copy` tasks |
| `maven-release-plugin` | `maven-publish` plugin, `release` plugin |
| `maven-shade-plugin` | `gradle-shadow-plugin` (阴影 fat JAR) |
| `spotbugs-maven-plugin` | `com.github.spotbugs` plugin |
| `jacoco-maven-plugin` | `jacoco` plugin |
| `maven-checkstyle-plugin` | `checkstyle` plugin |
| `maven-pmd-plugin` | `pmd` plugin |

## 4.2 编译器配置映射

```xml
<!-- Maven -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <source>17</source>
        <target>17</target>
        <encoding>UTF-8</encoding>
        <compilerArgs>
            <arg>-Xlint:unchecked</arg>
            <arg>--add-exports</arg>
            <arg>java.base/sun.security.x509=ALL-UNNAMED</arg>
        </compilerArgs>
        <annotationProcessorPaths>
            <path>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
                <version>1.18.30</version>
            </path>
        </annotationProcessorPaths>
    </configuration>
</plugin>
```

```groovy
// Gradle
java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}

tasks.withType(JavaCompile).configureEach {
    options.encoding = 'UTF-8'
    options.compilerArgs += [
        '-Xlint:unchecked',
        '--add-exports', 'java.base/sun.security.x509=ALL-UNNAMED'
    ]
}

dependencies {
    annotationProcessor 'org.projectlombok:lombok:1.18.30'
    compileOnly 'org.projectlombok:lombok:1.18.30'
}
```

## 4.3 测试配置映射

```xml
<!-- Maven surefire -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <includes>
            <include>**/*Test.java</include>
            <include>**/*Tests.java</include>
        </includes>
        <excludes>
            <exclude>**/*IT.java</exclude>
        </excludes>
        <parallel>classes</parallel>
        <threadCount>4</threadCount>
        <systemPropertyVariables>
            <env>test</env>
        </systemPropertyVariables>
    </configuration>
</plugin>

<!-- Maven failsafe - 集成测试 -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-failsafe-plugin</artifactId>
    <configuration>
        <includes>
            <include>**/*IT.java</include>
        </includes>
    </configuration>
</plugin>
```

```groovy
// Gradle - 单元测试
tasks.named('test', Test) {
    useJUnitPlatform()
    
    filter {
        includeTestsMatching '*Test'
        includeTestsMatching '*Tests'
        excludeTestsMatching '*IT'
    }
    
    maxParallelForks = 4
    forkEvery = 1
    
    systemProperty 'env', 'test'
}

// Gradle - 集成测试 (需要手动创建 source set)
sourceSets {
    integrationTest {
        compileClasspath += sourceSets.main.output + configurations.testRuntimeClasspath
        runtimeClasspath += output + compileClasspath
    }
}

configurations {
    integrationTestImplementation.extendsFrom testImplementation
    integrationTestRuntimeOnly.extendsFrom testRuntimeOnly
}

task integrationTest(type: Test) {
    description = 'Runs integration tests'
    group = 'verification'
    
    testClassesDirs = sourceSets.integrationTest.output.classesDirs
    classpath = sourceSets.integrationTest.runtimeClasspath
    
    useJUnitPlatform()
    filter {
        includeTestsMatching '*IT'
    }
    
    shouldRunAfter test
}

check.dependsOn integrationTest
```

# 5. 属性映射

## 5.1 Maven Properties → Gradle

```xml
<!-- Maven properties -->
<properties>
    <java.version>17</java.version>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <spring.version>6.1.0</spring.version>
    <myapp.outputDir>${project.build.directory}/output</myapp.outputDir>
</properties>
```

```groovy
// Gradle - 多种属性映射方式

// 1. ext 块 (基础场景)
ext {
    springVersion = '6.1.0'
    outputDir = layout.buildDirectory.dir("output")
}

// 2. gradle.properties
// gradle.properties 文件:
// javaVersion=17
// springVersion=6.1.0

// 3. 版本目录 (推荐)
// gradle/libs.versions.toml
// [versions]
// spring = "6.1.0"

// 4. Kotlin DSL 委托
// val springVersion by extra("6.1.0")

// 构建时使用
tasks.withType(JavaCompile).configureEach {
    options.encoding = "UTF-8"
    sourceCompatibility = "17"
    targetCompatibility = "17"
}

dependencies {
    implementation "org.springframework:spring-context:${springVersion}"
}
```

## 5.2 Resource Filtering 映射

```xml
<!-- Maven resource filtering -->
<build>
    <resources>
        <resource>
            <directory>src/main/resources</directory>
            <filtering>true</filtering>
            <includes>
                <include>**/*.properties</include>
                <include>**/*.xml</include>
            </includes>
        </resource>
    </resources>
</build>
```

```groovy
// Gradle resource filtering
// 方式1：使用 processResources 中的 expand
tasks.named('processResources', ProcessResources) {
    filesMatching('**/*.properties') {
        expand(project.properties)
    }
    filesMatching('**/*.xml') {
        expand([
            version: project.version,
            appName: project.name
        ])
    }
}

// 方式2：使用 filter 和 ReplaceTokens (ANT filter)
import org.apache.tools.ant.filters.ReplaceTokens

tasks.named('processResources', ProcessResources) {
    filter(ReplaceTokens, tokens: [
        "app.version": project.version,
        "app.name": project.name,
        "build.timestamp": new Date().format("yyyy-MM-dd HH:mm:ss")
    ])
}

// src/main/resources/application.properties 中的占位符
// app.version=@app.version@
// app.name=@app.name@
```

# 6. Profile 映射

## 6.1 Maven Profile → Gradle 等价方案

```xml
<!-- Maven profiles -->
<profiles>
    <profile>
        <id>development</id>
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
        <properties>
            <env>dev</env>
        </properties>
        <dependencies>
            <dependency>
                <groupId>com.h2database</groupId>
                <artifactId>h2</artifactId>
                <scope>runtime</scope>
            </dependency>
        </dependencies>
    </profile>
    <profile>
        <id>production</id>
        <properties>
            <env>prod</env>
        </properties>
        <dependencies>
            <dependency>
                <groupId>org.postgresql</groupId>
                <artifactId>postgresql</artifactId>
                <scope>runtime</scope>
            </dependency>
        </dependencies>
    </profile>
</profiles>
```

```groovy
// Gradle profile 映射方案

// 方式1：使用系统属性控制
ext {
    env = System.getProperty('env', 'dev')
}

dependencies {
    if (env == 'dev') {
        runtimeOnly 'com.h2database:h2:2.2.224'
    } else {
        runtimeOnly 'org.postgresql:postgresql:42.7.1'
    }
}

// 使用: gradle build -Denv=prod

// 方式2：使用 gradle.properties 和 build variants
// gradle/dev.gradle
// gradle/prod.gradle

// build.gradle
def env = project.hasProperty('env') ? project.env : 'dev'
apply from: "gradle/${env}.gradle"

// 使用: gradle build -Penv=prod
```

```groovy
// 方式3：创建自定义 SourceSet (多 flavor 模式，类似 Android)
sourceSets {
    main {
        java {
            srcDirs = ['src/main/java', "src/${profile}/java"]
        }
    }
}

// 方式4：使用 Gradle 的配置变体 (Configuration Variants)
configurations {
    devImplementation {
        extendsFrom implementation
    }
    prodImplementation {
        extendsFrom implementation
    }
}

dependencies {
    devImplementation 'com.h2database:h2:2.2.224'
    prodImplementation 'org.postgresql:postgresql:42.7.1'
}

// 然后在运行时选择正确的 configuration
def profileConfig = project.hasProperty('profile') ?
    configurations.getByName("${profile}Implementation") :
    configurations.devImplementation
```

## 6.2 Profile 激活条件映射

```xml
<!-- Maven activation conditions -->
<activation>
    <property>
        <name>os.name</name>
        <value>Windows 10</value>
    </property>
    <jdk>[17,)</jdk>
    <file>
        <exists>src/main/prod/application.properties</exists>
    </file>
</activation>
```

```groovy
// Gradle 中对应条件判断
def isWindows = System.getProperty('os.name').toLowerCase().contains('windows')
def jdkVersion = JavaVersion.current()
def prodFileExists = file('src/main/prod/application.properties').exists()

// 在配置阶段使用
if (isWindows && jdkVersion >= JavaVersion.VERSION_17) {
    tasks.withType(JavaCompile).configureEach {
        options.fork = true
        options.forkOptions.executable = 'javac'
    }
}

if (prodFileExists) {
    sourceSets.main.resources.srcDirs += 'src/main/prod'
}
```

# 7. 仓库映射

```xml
<!-- Maven repositories -->
<repositories>
    <repository>
        <id>central</id>
        <url>https://repo.maven.apache.org/maven2</url>
    </repository>
    <repository>
        <id>company-repo</id>
        <url>https://nexus.example.com/repository/maven-releases/</url>
        <releases>
            <enabled>true</enabled>
        </releases>
        <snapshots>
            <enabled>false</enabled>
        </snapshots>
    </repository>
</repositories>

<distributionManagement>
    <repository>
        <id>company-releases</id>
        <url>https://nexus.example.com/repository/maven-releases/</url>
    </repository>
    <snapshotRepository>
        <id>company-snapshots</id>
        <url>https://nexus.example.com/repository/maven-snapshots/</url>
    </snapshotRepository>
</distributionManagement>
```

```groovy
// Gradle repositories
repositories {
    mavenCentral()
    
    maven {
        name = 'companyRepo'
        url = uri('https://nexus.example.com/repository/maven-releases/')
        mavenContent {
            releasesOnly()
        }
    }
    
    // 带认证的私有仓库
    maven {
        name = 'privateRepo'
        url = uri('https://nexus.example.com/repository/private/')
        credentials {
            username = project.findProperty('nexusUsername') ?: System.getenv('NEXUS_USER')
            password = project.findProperty('nexusPassword') ?: System.getenv('NEXUS_PASS')
        }
    }
}

// 发布配置
publishing {
    repositories {
        maven {
            name = 'companyReleases'
            url = uri('https://nexus.example.com/repository/maven-releases/')
            credentials(PasswordCredentials)
        }
        maven {
            name = 'companySnapshots'
            url = uri('https://nexus.example.com/repository/maven-snapshots/')
            credentials(PasswordCredentials)
        }
    }
    publications {
        maven(MavenPublication) {
            from components.java
        }
    }
}
```

# 8. 构建声明周期映射

## 8.1 Maven 阶段 → Gradle 任务

```
Maven 生命周期阶段          Gradle 等价任务
─────────────────────────────────────────────
validate                  (无直接对应，可在编译前检查)
compile                   compileJava
test                      test
package                   jar / war / bootJar
verify                    check
install                   publishToMavenLocal
deploy                    publish
clean                     clean
site                      (需要自定义报告任务)
```

```groovy
// Gradle 任务依赖链模拟 Maven 生命周期
// 默认情况下 Gradle 的任务依赖图已经覆盖了 Maven 的大部分流程：
// compileJava → processResources → classes → jar → assemble
// compileTestJava → processTestResources → testClasses → test → check
```

## 8.2 构建阶段附件

```xml
<!-- Maven 阶段的多个 execution -->
<execution>
    <id>generate-sources</id>
    <phase>generate-sources</phase>
    <goals>
        <goal>generate</goal>
    </goals>
</execution>
```

```groovy
// Gradle 对应方案
tasks.register('generateSources', JavaExec) {
    // 自定义代码生成逻辑
    main = 'com.example.CodeGenerator'
    classpath = sourceSets.main.runtimeClasspath
}

// 将生成的代码附加到编译路径
sourceSets.main.java.srcDir(
    tasks.named('generateSources').map { it.outputs.files }
)

tasks.named('compileJava') {
    dependsOn generateSources
}
```

# 9. 迁移验证

## 9.1 构建产物对比

```bash
# 对比 Maven 和 Gradle 构建的 JAR 文件内容
# 1. 使用 Maven 构建
mvn clean package -DskipTests

# 2. 使用 Gradle 构建
gradle clean build -x test

# 3. 解压并对比
mkdir maven-jar gradle-jar
unzip maven/target/my-app-1.0.0.jar -d maven-jar/
unzip gradle/build/libs/my-app-1.0.0.jar -d gradle-jar/

diff -r maven-jar/ gradle-jar/

# 4. 对比 MANIFEST.MF
diff maven-jar/META-INF/MANIFEST.MF gradle-jar/META-INF/MANIFEST.MF
```

## 9.2 依赖树对比

```bash
# Maven 依赖树
mvn dependency:tree -DoutputFile=maven-deps.txt

# Gradle 依赖树
gradle dependencies --configuration runtimeClasspath > gradle-deps.txt

# 对比两个文件的差异
diff maven-deps.txt gradle-deps.txt
```

## 9.3 完整验证清单

```groovy
// 验证 Gradle 构建的正确性
task verifyMigration {
    dependsOn build
    
    doLast {
        // 1. 检查所有必需的任务都存在
        assert tasks.findByName('compileJava') != null
        assert tasks.findByName('test') != null
        assert tasks.findByName('jar') != null
        assert tasks.findByName('check') != null
        
        // 2. 验证 JAR 包含正确的文件
        def jarFile = tasks.jar.archiveFile.get().asFile
        assert jarFile.exists()
        
        def zip = new java.util.zip.ZipFile(jarFile)
        def entries = zip.entries().collect { it.name }
        assert entries.any { it.endsWith('.class') }
        assert entries.contains('META-INF/MANIFEST.MF')
        
        // 3. 检查资源文件被正确处理
        assert entries.any { it.startsWith('META-INF/') && it.endsWith('.properties') }
        
        // 4. 验证测试报告存在
        def testReport = file("$buildDir/reports/tests/test/index.html")
        assert testReport.exists() || !tasks.test.didWork
        
        zip.close()
    }
}
```

# 10. 常见迁移陷阱

## 10.1 作用域差异

```groovy
// 陷阱1：Maven compile scope → Gradle implementation
// Maven 中 compile 依赖默认传递给下游
// Gradle 中 implementation 不传递
// 如果下游需要该依赖，必须使用 api

// 错误示例
dependencies {
    implementation 'com.google.guava:guava:33.0.0-jre'
    // 下游项目将无法访问 Guava 的类
}

// 正确示例（如果 Guava 在公共 API 中使用）
dependencies {
    api 'com.google.guava:guava:33.0.0-jre'
}
```

## 10.2 插件行为差异

```groovy
// 陷阱2：Maven 和 Gradle 插件对 test fixture 的不同处理
// Maven: test-jar type 自动创建测试 JAR
// Gradle: 需要 java-test-fixtures 插件

plugins {
    id 'java-test-fixtures'
}
// 自动创建 testFixtures source set
// 下游通过 testFixtures(project(":lib")) 引用
```

## 10.3 资源处理差异

```groovy
// 陷阱3：资源过滤和文件编码
// Maven 默认使用项目 encoding 过滤资源
// Gradle 默认不进行过滤

// 如果 Maven 使用了 resource filtering
// Gradle 需要显式配置：
tasks.named('processResources') {
    filesMatching('**/*.properties') {
        filter {
            it.replace('${project.version}', project.version as String)
        }
    }
}
```

## 10.4 Maven Enforcer → Gradle 等价

```xml
<!-- Maven Enforcer Plugin -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-enforcer-plugin</artifactId>
    <executions>
        <execution>
            <id>enforce-versions</id>
            <goals><goal>enforce</goal></goals>
            <configuration>
                <rules>
                    <requireJavaVersion>
                        <version>[17,)</version>
                    </requireJavaVersion>
                    <dependencyConvergence/>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

```groovy
// Gradle 等价方案
// 1. Java 版本检查
assert JavaVersion.current() >= JavaVersion.VERSION_17 :
    "需要 Java 17 或更高版本，当前版本: ${JavaVersion.current()}"

// 2. 依赖收敛检查
configurations.all {
    resolutionStrategy {
        failOnVersionConflict()
        // 或手动解决
        force 'com.google.guava:guava:33.0.0-jre'
    }
}

// 3. 使用 gradle-enforcer-plugin
plugins {
    id 'org.gradle.enforcer' version '0.1'
}
enforce {
    check {
        javaVersion { min = '17' }
        dependencyConvergence { enabled = true }
        bannedDependencies {
            exclude 'commons-logging:commons-logging'
        }
    }
}
```
