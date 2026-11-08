# 1. Maven 集成

Maven 侧的 JUnit 5 集成靠 **Surefire 3.x**（单元测试，绑定 `test` 阶段）和 **Failsafe 3.x**（集成测试，绑定 `integration-test`/`verify` 阶段) 两个插件。Surefire 从 2.22.0 起内置 JUnit Platform provider，**classpath 上有 `junit-jupiter-engine` 就会自动探测并启用，无需手工配置 provider**；3.x 修掉了大量 Platform 相关问题（JPMS、`@Suite`、并行），新项目直接用 3.2+。

## 1.1 Surefire 单元测试

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.5</version>
    <configuration>
        <!-- 非模块化项目关掉模块路径，避免 classpath/modulepath 探测踩坑 -->
        <useModulePath>false</useModulePath>
    </configuration>
</plugin>
```

依赖侧只需 `junit-jupiter`（聚合了 api/params/engine，engine 是 `testRuntimeOnly` 性质，Maven 里用 test scope 即可）：

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.10.2</version>
    <scope>test</scope>
</dependency>
```

## 1.2 provider 选择机制

Surefire 按 classpath 探测顺序选 provider：**junit-platform-launcher/engine 在场 → JUnitPlatformProvider；只有 junit:junit 4.x → JUnit4Provider；有 junit 3.x → JUnit3Provider**。共存期（Vintage + Jupiter）走 Platform provider，由 Vintage/Jupiter 双引擎分别认领测试。两个实践要点：

- 不要手工往插件里塞 `surefire-junit47` 之类的 provider 依赖，探测错误时先检查是不是测试依赖 scope 写错导致 engine 不在 runtime classpath。
- Surefire 3.x 会**自动对齐 `junit-platform-launcher` 的版本**到项目实际使用的 Platform 版本，2.22.x 时代常见的 launcher/engine 版本冲突报错在 3.x 基本消失。

## 1.3 Failsafe 集成测试

Failsafe 与 Surefire 的关键区别：**测试失败发生在 `integration-test` 阶段，但构建要到 `verify` 阶段才失败**——保证 `post-integration-test` 有机会关闭容器/服务器。典型组合是配合 Cargo 或 Testcontainers 起停被测应用：

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-failsafe-plugin</artifactId>
    <version>3.2.5</version>
    <executions>
        <execution>
            <goals>
                <goal>integration-test</goal>   <!-- 跑 IT，失败先记录 -->
                <goal>verify</goal>              <!-- 根据记录决定构建成败 -->
            </goals>
        </execution>
    </executions>
</plugin>
```

## 1.4 测试类命名约定

| 插件 | 默认 includes | 说明 |
|------|--------------|------|
| Surefire | `**/Test*.java`、`**/*Test.java`、`**/*Tests.java`、`**/*TestCase.java` | 抽象类自动排除 |
| Failsafe | `**/IT*.java`、`**/*IT.java`、`**/*ITCase.java` | 与 Surefire 互不重叠 |

这套约定的价值在于 **`mvn test` 与 `mvn verify` 天然分层**：单测进 test 阶段秒级反馈，集成测试进 verify 阶段。需要打破约定时用 `<includes>/<excludes>` 显式配置，或用 tag 过滤（见 1.5）。

## 1.5 tag 与选择性执行

```bash
mvn test -Dgroups="fast & !flaky" -DexcludedGroups="docker"
```

```xml
<configuration>
    <groups>${test.groups}</groups>              <!-- 属性化，CI 按需覆盖 -->
    <excludedGroups>slow</excludedGroups>
</configuration>
```

# 2. Maven 并发与 fork

## 2.1 forkCount / reuseForks：进程级并行

**`forkCount` 控制启动几个 JVM 进程跑测试，`reuseForks` 控制进程是否复用**，这是 Surefire 层的并行，与 JUnit Platform 的方法级线程并行（13 篇）是两个维度：

```xml
<configuration>
    <forkCount>1C</forkCount>       <!-- 1C = 每 CPU 核 1 个进程，也可写死数字 -->
    <reuseForks>true</reuseForks>   <!-- 默认 true：进程跑完一批测试后复用 -->
</configuration>
```

组合语义：

| forkCount | reuseForks | 行为 |
|-----------|-----------|------|
| 1 | true（默认） | 单进程串行，最保守 |
| 1C | true | N 个常驻进程分抢测试类，**最常用的提速配置** |
| 1C | false | 每个测试类一个新 JVM，隔离最强、最慢 |
| 0 | - | 不 fork，测试跑在 Maven 自己的 JVM 里（argLine 失效，慎用） |

> **注意**：`reuseForks=false` 时每类一次 JVM 冷启动，大型项目慢到不可接受；但它是排查"测试间静态状态污染"的终极手段——本地用 `forkCount=1 reuseForks=false` 复现通过、CI 并行配置下失败，基本可断定是共享状态问题。`forkCount>1` 时每个进程各占一份堆内存，`argLine` 里的 `-Xmx` 要按核数乘一下，CI 容器内存不足会 OOM kill。

## 2.2 parallel/threadCount：JVM 内线程并行

```xml
<configuration>
    <parallel>classes</parallel>   <!-- methods / classes / both / classesAndMethods -->
    <threadCount>4</threadCount>
    <useUnlimitedThreads>false</useUnlimitedThreads>
</configuration>
```

这是 Surefire 自己的线程池并行，**只对 JUnit 4 provider 和 Platform provider 生效方式不同**，且与 Jupiter 原生并行叠加时行为难以推理。取舍很清晰：**新项目用 Jupiter 原生并行（`junit-platform.properties`），Surefire 只负责 forkCount 进程级并行**；`parallel/threadCount` 留给无法开 Platform 并行的老项目。

# 3. Gradle 集成

## 3.1 启用 JUnit Platform

Gradle 不会自动探测引擎，**`useJUnitPlatform()` 是显式开关，漏掉则 Jupiter 测试一个都不跑且构建仍然绿**：

```groovy
// build.gradle（Groovy DSL）
dependencies {
    testImplementation platform('org.junit:junit-bom:5.10.2')
    testImplementation 'org.junit.jupiter:junit-jupiter'
}

test {
    useJUnitPlatform()            // Gradle 8.x 内置 Platform 支持，无需额外依赖
}
```

Gradle 8.x 默认使用项目 classpath 上的 `junit-platform-launcher` 版本，并会校验其与 Gradle 内置支持范围的兼容性——版本太老（< 1.3）会直接报错提示升级，比静默跳过友好。

## 3.2 测试过滤 --tests

```bash
gradle test --tests "com.example.UserServiceTest"          # 单个类
gradle test --tests "*UserService*"                        # 通配符
gradle test --tests "com.example.UserServiceTest.create*"  # 类内方法前缀匹配
gradle test --tests "*.create_order"                       # 所有类的同名方法
```

tag 过滤则写在构建脚本里：

```groovy
test {
    useJUnitPlatform {
        includeTags 'fast'
        excludeTags 'docker'
    }
}

// 更灵活：注册独立 task 分层执行
tasks.register('integrationTest', Test) {
    useJUnitPlatform { includeTags 'integration' }
    shouldRunAfter test                    // 排在单测之后
}
```

> **行业实践**：`--tests` 是本地开发的主力（改哪个类跑哪个类），但 CI 上不要用 `--tests` 做"只跑受影响的测试"，Gradle 的 build cache + up-to-date 检查已经能对未变更模块跳过测试，手工过滤反而破坏缓存命中。

## 3.3 maxParallelForks 与其他性能开关

```groovy
test {
    maxParallelForks = Runtime.runtime.availableProcessors().div(2) ?: 1  // 进程级并行
    forkEvery = 100                     // 每 100 个测试类换新 JVM，防内存泄漏累积
    maxHeapSize = '1g'
    jvmArgs '-XX:+UseParallelGC'
}
```

`maxParallelForks` 对应 Surefire 的 `forkCount`，`forkEvery` 对应 `reuseForks` 的细粒度版（0 = 永不换）。Gradle 8 还可以用 `develocity` 的 test distribution/parallel test execution 把并行下沉到方法级并分发到多 agent，大仓库场景比 `maxParallelForks` 收益更高。

# 4. JaCoCo 覆盖率集成

## 4.1 Maven

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <executions>
        <execution>
            <id>prepare-agent</id>          <!-- 注入 argLine 探针 -->
            <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
            <id>report</id>                 <!-- test 之后出报告 -->
            <phase>test</phase>
            <goals><goal>report</goal></goals>
        </execution>
        <execution>
            <id>check</id>                  <!-- 覆盖率门禁 -->
            <phase>verify</phase>
            <goals><goal>check</goal></goals>
            <configuration>
                <rules>
                    <rule>
                        <element>BUNDLE</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.60</minimum>   <!-- 行覆盖 60% 起步 -->
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

> **注意**：JaCoCo 靠改写 Surefire 的 `argLine` 属性注入 javaagent，**如果你在 Surefire 配置里自己写死了 `<argLine>`，探针会被覆盖，报告恒为空**。正确写法是用占位符拼接：`<argLine>@{argLine} -Xmx2g</argLine>`。`forkCount>1` 时每个 fork 各写一份 `jacoco.exec`，JaCoCo 0.8.x 默认按 `destFile` 追加合并，无需额外处理。

## 4.2 Gradle

```groovy
plugins {
    id 'jacoco'
}

jacoco {
    toolVersion = '0.8.11'
}

test {
    finalizedBy jacocoTestReport        // 测试跑完自动出报告
}

jacocoTestReport {
    reports {
        xml.required = true             // CI/SonarQube 消费 xml
        html.required = true            // 本地查看
    }
}

jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                minimum = 0.60          // 与 Maven 侧一致的门禁
            }
        }
    }
}
check.dependsOn jacocoTestCoverageVerification
```

JaCoCo 与 Jupiter 原生线程并行兼容（agent 内部同步计数）；但 **并行导致覆盖率轻微抖动**（超时/竞态路径的偶发覆盖），门禁阈值应留出余量，不要贴着当前值设。

# 5. CI 中的测试分层执行

**CI 上测试不是一把梭全跑，而是按反馈速度分层**，越早的阶段失败越快、成本越低：

```
┌─────────────────────────────────────────────────┐
│ 阶段 1  编译 + 静态检查          （秒级）          │
│   compile / checkstyle / spotbugs               │
├─────────────────────────────────────────────────┤
│ 阶段 2  快速单测                 （分钟级）        │
│   mvn test -Dgroups="fast & !slow"              │
│   gradle test -PincludeTags=fast                │
│   → PR 门禁，失败即阻断                            │
├─────────────────────────────────────────────────┤
│ 阶段 3  全量单测 + 覆盖率         （合并前）        │
│   mvn verify（JaCoCo check 生效）                │
├─────────────────────────────────────────────────┤
│ 阶段 4  集成测试                 （合并后/夜间）    │
│   Failsafe *IT + Testcontainers                 │
│   gradle integrationTest                        │
│   → 需要 docker-in-docker 或 Testcontainers Cloud│
└─────────────────────────────────────────────────┘
```

分层落地的几个经验：

- **阶段 2 靠 tag 而不是文件名过滤**：文件名约定（`*Test`/`*IT`）粒度太粗，一个类里既有快测也有慢测时只能整类划归；tag 能到方法级。
- **覆盖率门禁放阶段 3 而不是阶段 2**：快测阶段覆盖率天然不满，门禁会误报。
- **集成测试依赖外部环境时优先 Testcontainers**，避免 CI runner 上手工装数据库；docker 不可用的环境用 tag `docker` 排除并另排专用 runner。
- **flaky 测试隔离**：给不稳定测试打 `@Tag("flaky")` 从阶段 2 排除、放进夜间任务观察，修好一个摘一个标签，比反复重跑掩盖问题有效。

# 6. 依赖管理建议

- **用 BOM 统一版本**：Maven 在 `dependencyManagement` 导入 `org.junit:junit-bom`，Gradle 用 `platform('org.junit:junit-bom:5.10.2')`，杜绝 jupiter-api / params / engine 版本漂移导致的 `NoSuchMethodError`。
- **Vintage 与 Jupiter 同版本**：两者都出自 junit5 仓库，BOM 管理下自动一致；手工指定版本时最容易错。
- **engine 依赖放在运行期**：Gradle 里 `testRuntimeOnly 'org.junit.jupiter:junit-jupiter-engine'`（聚合包已含）；编译期只需要 api 包，避免测试代码直接 import 引擎内部类。
- **排查多版本冲突**：`mvn dependency:tree -Dincludes=org.junit*` / `gradle dependencies --configuration testRuntimeClasspath`，常见污染源是 spring-boot-starter-test（自带 Jupiter，版本由 Spring Boot BOM 托管）与显式声明的 junit-jupiter 版本打架——**用 Spring Boot 时以 Boot 的 BOM 为准，不要显式覆盖 JUnit 版本**，除非你清楚 Platform 与 Boot 的兼容矩阵。

# 7. 小结

| 主题 | 要点 |
|------|------|
| Maven | Surefire/Failsafe 3.x 自动探测 Platform provider，无需手工配 provider |
| 命名约定 | Surefire：`*Test/*Tests/*TestCase`；Failsafe：`*IT/*ITCase`，天然分层 |
| Maven 并行 | `forkCount=1C` + `reuseForks=true` 是主力；`parallel/threadCount` 留给老项目 |
| Gradle | `useJUnitPlatform()` 必写，漏掉则静默跳过；`--tests` 本地过滤主力 |
| Gradle 并行 | `maxParallelForks` 进程级，`forkEvery` 防泄漏 |
| JaCoCo | Maven 注意 `@{argLine}` 占位符防探针被覆盖；门禁阈值留余量 |
| CI 分层 | 快测门禁（tag 过滤）→ 全量+覆盖率 → 集成测试（Testcontainers）→ 夜间 flaky 观察 |
| 版本管理 | junit-bom 统一；Spring Boot 项目以 Boot BOM 为准 |
