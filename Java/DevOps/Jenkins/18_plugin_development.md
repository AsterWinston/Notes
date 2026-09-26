
# 1. 概述

Jenkins 插件的生态系统是其最大的优势之一。通过插件，可以扩展 Jenkins 的几乎所有功能：添加新的构建步骤、集成新的工具、自定义 UI 组件等。

## 1.1 插件架构

```
┌─────────────────────────────────────────────────────────┐
│                   Jenkins Core                          │
│  Extension  Point  ←  发现和加载插件                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │             插件（Plugin）                        │   │
│  │  ┌─────────┐ ┌──────────┐ ┌────────────────┐   │   │
│  │  │ Builder │ │ Trigger  │ │ Notifier       │   │   │
│  │  └─────────┘ └──────────┘ └────────────────┘   │   │
│  │  ┌─────────┐ ┌──────────┐ ┌────────────────┐   │   │
│  │  │ SCM     │ │ Publisher│ │ Step (Pipeline)│   │   │
│  │  └─────────┘ └──────────┘ └────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

## 1.2 常见的扩展点

| 扩展点 | 接口/基类 | 用途 |
|--------|-----------|------|
| Builder | `Builder` | 添加构建步骤 |
| Publisher | `Publisher` | 后构建操作（通知、归档） |
| SCM | `SCM` | 支持新的源码管理工具 |
| Trigger | `Trigger` | 添加新的构建触发方式 |
| Notifier | `Notifier` | 发送通知 |
| RootAction | `RootAction` | 在 UI 中添加顶级页面 |
| Action | `Action` | 在 Job/Run 页面添加信息 |
| Descriptor | `Descriptor` | 配置界面的描述符 |
| Step | `Step` | Pipeline 步骤 |
| GlobalConfiguration | `GlobalConfiguration` | 全局配置 |

# 2. 开发环境搭建

## 2.1 前提条件

```yaml
# 开发环境要求
- JDK 11 或 17
- Maven 3.8+
- IntelliJ IDEA（推荐）或 Eclipse
- Git
```

## 2.2 创建项目

```bash
# 使用 Maven 原型创建插件项目
mvn archetype:generate \
  -Dfilter=io.jenkins.archetypes:plugin \
  -DarchetypeGroupId=io.jenkins.archetypes \
  -DarchetypeArtifactId=plugin \
  -DarchetypeVersion=1.1.0

# 交互式填写的参数：
# groupId: com.company.jenkins
# artifactId: my-plugin
# version: 1.0.0-SNAPSHOT
# package: com.company.jenkins.plugins.myplugin
```

## 2.3 POM 配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.jenkins-ci.plugins</groupId>
        <artifactId>plugin</artifactId>
        <version>4.78</version>
        <relativePath />
    </parent>

    <groupId>com.company.jenkins</groupId>
    <artifactId>my-plugin</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>hpi</packaging>
    <name>My Jenkins Plugin</name>
    <description>描述你的插件功能</description>
    <url>https://github.com/myorg/my-plugin</url>

    <properties>
        <jenkins.version>2.440.3</jenkins.version>
        <java.level>8</java.level>
    </properties>

    <dependencies>
        <!-- Jenkins 核心依赖 -->
        <dependency>
            <groupId>org.jenkins-ci.main</groupId>
            <artifactId>jenkins-core</artifactId>
            <version>${jenkins.version}</version>
            <scope>provided</scope>
        </dependency>

        <!-- Pipeline 步骤 API -->
        <dependency>
            <groupId>org.jenkins-ci.plugins</groupId>
            <artifactId>workflow-step-api</artifactId>
            <version>2.26</version>
            <scope>provided</scope>
        </dependency>

        <!-- 测试依赖 -->
        <dependency>
            <groupId>org.jenkins-ci.plugins</groupId>
            <artifactId>plugin</artifactId>
            <version>4.78</version>
            <type>pom</type>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <repositories>
        <repository>
            <id>jenkins-releases</id>
            <url>https://repo.jenkins-ci.org/releases/</url>
        </repository>
    </repositories>
</project>
```

## 2.4 开发配置（IntelliJ IDEA）

```yaml
# IntelliJ IDEA 配置步骤
1. 导入 Maven 项目
2. 安装 Jenkins Plugin Development 插件（可选）
3. 配置 Run Configuration:
   - Main class: Main (jenkins 内嵌启动类)
   - VM options:
     -Dhudson.model.DirectoryBrowserSupport.CSP=
     -Djenkins.install.runSetupWizard=false
   - Use classpath of module: my-plugin
4. 运行 → 启动 Jenkins 实例
5. 访问 http://localhost:8080/jenkins
```

# 3. 开发一个简单的构建步骤

## 3.1 插件主类

```java
package com.company.jenkins.plugins.myplugin;

import hudson.Plugin;
import jenkins.model.Jenkins;

/**
 * 插件主类（可选，仅在需要初始化逻辑时）
 */
public class MyPlugin extends Plugin {
    @Override
    public void start() throws Exception {
        // 插件启动时的初始化逻辑
        System.out.println("MyPlugin started");
    }

    @Override
    public void stop() throws Exception {
        // 插件停止时的清理逻辑
        System.out.println("MyPlugin stopped");
    }
}
```

## 3.2 构建步骤实现

```java
package com.company.jenkins.plugins.myplugin;

import hudson.Extension;
import hudson.Launcher;
import hudson.model.AbstractBuild;
import hudson.model.AbstractProject;
import hudson.model.BuildListener;
import hudson.tasks.BuildStepDescriptor;
import hudson.tasks.Builder;
import jenkins.tasks.SimpleBuildStep;
import net.sf.json.JSONObject;
import org.kohsuke.stapler.DataBoundConstructor;
import org.kohsuke.stapler.StaplerRequest;

import javax.annotation.Nonnull;
import java.io.IOException;
import java.io.PrintStream;

/**
 * 自定义构建步骤
 */
public class HelloWorldBuilder extends Builder implements SimpleBuildStep {

    private final String name;
    private final boolean useFrench;

    @DataBoundConstructor
    public HelloWorldBuilder(String name, boolean useFrench) {
        this.name = name;
        this.useFrench = useFrench;
    }

    public String getName() {
        return name;
    }

    public boolean isUseFrench() {
        return useFrench;
    }

    @Override
    public void perform(
            @Nonnull AbstractBuild<?, ?> build,
            @Nonnull Launcher launcher,
            @Nonnull BuildListener listener) throws InterruptedException, IOException {

        PrintStream logger = listener.getLogger();
        String greeting = useFrench ? "Bonjour" : "Hello";

        logger.println(greeting + ", " + name + "!");

        // 获取环境变量
        String buildNumber = build.getEnvironment(listener).get("BUILD_NUMBER");
        logger.println("Build number: " + buildNumber);
    }

    /**
     * 描述符（Descriptor）
     * 用于配置界面的定义
     */
    @Extension
    public static final class DescriptorImpl extends BuildStepDescriptor<Builder> {

        public DescriptorImpl() {
            load();  // 加载持久化配置
        }

        @Override
        public boolean isApplicable(Class<? extends AbstractProject> jobType) {
            return true;  // 对所有 Job 类型可用
        }

        @Override
        public String getDisplayName() {
            return "Say Hello World";  // 构建步骤的显示名称
        }

        @Override
        public boolean configure(StaplerRequest req, JSONObject formData) throws FormException {
            // 保存全局配置
            save();
            return super.configure(req, formData);
        }
    }
}
```

## 3.3 配置界面 Jelly

```xml
<!-- src/main/resources/com/company/jenkins/plugins/myplugin/HelloWorldBuilder/config.jelly -->

<?jelly escape-by-default='true'?>
<j:jelly xmlns:j="jelly:core"
         xmlns:f="/lib/form">
    <f:entry title="Name" field="name">
        <f:textbox default="World" />
    </f:entry>
    <f:entry title="Use French" field="useFrench">
        <f:checkbox default="false" />
    </f:entry>
</j:jelly>
```

## 3.4 全局配置

```java
// 全局配置示例
package com.company.jenkins.plugins.myplugin;

import hudson.Extension;
import jenkins.model.GlobalConfiguration;
import net.sf.json.JSONObject;
import org.kohsuke.stapler.DataBoundSetter;
import org.kohsuke.stapler.StaplerRequest;

@Extension
public class MyGlobalConfiguration extends GlobalConfiguration {

    private String defaultName = "World";

    public MyGlobalConfiguration() {
        load();
    }

    public String getDefaultName() {
        return defaultName;
    }

    @DataBoundSetter
    public void setDefaultName(String defaultName) {
        this.defaultName = defaultName;
        save();
    }
}
```

```xml
<!-- src/main/resources/com/company/jenkins/plugins/myplugin/MyGlobalConfiguration/config.jelly -->
<?jelly escape-by-default='true'?>
<j:jelly xmlns:j="jelly:core"
         xmlns:f="/lib/form">
    <f:section title="My Plugin Configuration">
        <f:entry title="Default Name" field="defaultName">
            <f:textbox />
        </f:entry>
    </f:section>
</j:jelly>
```

# 4. Pipeline 步骤开发

## 4.1 步骤实现

```java
package com.company.jenkins.plugins.myplugin;

import hudson.Extension;
import org.jenkinsci.plugins.workflow.steps.*;
import org.kohsuke.stapler.DataBoundConstructor;
import org.kohsuke.stapler.DataBoundSetter;

import javax.annotation.Nonnull;
import java.util.*;

/**
 * Pipeline 步骤：sayHello
 */
public class SayHelloStep extends Step {

    private final String name;

    private boolean useFrench;

    @DataBoundConstructor
    public SayHelloStep(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public boolean isUseFrench() {
        return useFrench;
    }

    @DataBoundSetter
    public void setUseFrench(boolean useFrench) {
        this.useFrench = useFrench;
    }

    @Override
    public StepExecution start(StepContext context) throws Exception {
        return new Execution(context, this);
    }

    /**
     * 步骤执行逻辑
     */
    public static class Execution extends SynchronousNonBlockingStepExecution<String> {

        private static final long serialVersionUID = 1L;

        private final SayHelloStep step;

        public Execution(@Nonnull StepContext context, SayHelloStep step) {
            super(context);
            this.step = step;
        }

        @Override
        protected String run() throws Exception {
            String greeting = step.useFrench ? "Bonjour" : "Hello";
            String message = greeting + ", " + step.name + "!";

            // 输出到构建日志
            getContext().get(TaskListener.class).getLogger().println(message);

            return message;
        }
    }

    /**
     * 描述符
     */
    @Extension
    public static class DescriptorImpl extends StepDescriptor {

        @Override
        public String getFunctionName() {
            return "sayHello";  // Pipeline 中的步骤名
        }

        @Override
        public String getDisplayName() {
            return "Say Hello";
        }

        @Override
        public Set<? extends Class<?>> getRequiredContext() {
            return Collections.singleton(TaskListener.class);
        }
    }
}
```

## 4.2 Pipeline 中使用

```groovy
// Plugin 提供 sayHello 步骤
pipeline {
    agent any
    stages {
        stage('Greeting') {
            steps {
                sayHello 'World'
                sayHello name: 'Jenkins', useFrench: true
            }
        }
    }
}
```

# 5. 测试

## 5.1 单元测试

```java
package com.company.jenkins.plugins.myplugin;

import org.junit.Rule;
import org.junit.Test;
import org.jvnet.hudson.test.JenkinsRule;

public class HelloWorldBuilderTest {

    @Rule
    public JenkinsRule jenkins = new JenkinsRule();

    @Test
    public void testBuild() throws Exception {
        // 创建一个 FreeStyleJob
        FreeStyleProject project = jenkins.createFreeStyleProject("test-job");

        // 添加构建步骤
        project.getBuildersList().add(new HelloWorldBuilder("World", false));

        // 执行构建
        FreeStyleBuild build = jenkins.buildAndAssertSuccess(project);

        // 验证日志
        String log = JenkinsRule.getLog(build);
        System.out.println(log);
        assert log.contains("Hello, World!") : "Log should contain greeting";
    }
}
```

## 5.2 Pipeline 测试

```java
package com.company.jenkins.plugins.myplugin;

import org.jenkinsci.plugins.workflow.cps.CpsFlowDefinition;
import org.jenkinsci.plugins.workflow.job.WorkflowJob;
import org.jenkinsci.plugins.workflow.job.WorkflowRun;
import org.junit.Rule;
import org.junit.Test;
import org.jvnet.hudson.test.JenkinsRule;

public class SayHelloStepTest {

    @Rule
    public JenkinsRule jenkins = new JenkinsRule();

    @Test
    public void testPipelineStep() throws Exception {
        WorkflowJob job = jenkins.createProject(WorkflowJob.class, "pipeline-test");

        job.setDefinition(new CpsFlowDefinition(
            "node {\n" +
            "    sayHello 'Test'\n" +
            "    def result = sayHello name: 'French', useFrench: true\n" +
            "    echo \"Result: ${result}\"\n" +
            "}", true
        ));

        WorkflowRun run = jenkins.assertBuildStatusSuccess(job.scheduleBuild2(0));

        String log = JenkinsRule.getLog(run);
        System.out.println(log);
        assert log.contains("Hello, Test!") : "Should contain English greeting";
        assert log.contains("Bonjour, French!") : "Should contain French greeting";
    }
}
```

# 6. 国际化（i18n）

## 6.1 资源文件

```properties
# src/main/resources/com/company/jenkins/plugins/myplugin/Messages.properties
HelloWorldBuilder.DisplayName=Say Hello World
SayHelloStep.DisplayName=Say Hello
SayHelloStep.Name=Name
SayHelloStep.UseFrench=Use French

# Messages_zh_CN.properties（中文）
HelloWorldBuilder.DisplayName=问候世界
SayHelloStep.DisplayName=打招呼
SayHelloStep.Name=名称
SayHelloStep.UseFrench=使用法语
```

## 6.2 代码中使用

```java
import static com.company.jenkins.plugins.myplugin.Messages.*;

@Extension
public static final class DescriptorImpl extends BuildStepDescriptor<Builder> {
    @Override
    public String getDisplayName() {
        return HelloWorldBuilder_DisplayName();  // 从 Messages 读取
    }
}
```

# 7. 插件发布

## 7.1 准备发布

```bash
# 更新版本
mvn versions:set -DnewVersion=1.0.0

# 编译测试
mvn clean verify

# 生成 .hpi 文件
mvn package
# 输出：target/my-plugin.hpi
```

## 7.2 手动安装

```bash
# 复制到 Jenkins 插件目录
cp target/my-plugin.hpi /var/lib/jenkins/plugins/
chown jenkins:jenkins /var/lib/jenkins/plugins/my-plugin.hpi

# 重启 Jenkins
systemctl restart jenkins
# 或重新加载插件
curl -X POST http://localhost:8080/safeRestart
```

## 7.3 发布到 Jenkins 仓库

```bash
# 1. 在 Jenkins 插件站点注册账号
# https://plugins.jenkins.io/

# 2. 配置 Maven settings.xml
<settings>
  <servers>
    <server>
      <id>maven.jenkins-ci.org</id>
      <username>jenkins-username</username>
      <password>jenkins-api-token</password>
    </server>
  </servers>
</settings>

# 3. 发布
mvn release:prepare release:perform
```

# 8. 最佳实践

## 8.1 命名规范

```yaml
# 插件命名规范
artifact-id: my-plugin          # Maven artifact（小写连字符）
display-name: My Plugin         # UI 显示名称
class-name: MyPlugin            # 主类名（大驼峰）

# 步骤名
getFunctionName: myStep         # Pipeline 步骤名（小驼峰）

# 凭据 ID 命名
credentials-id: my-plugin-token
```

## 8.2 兼容性

```xml
<!-- 声明 Jenkins 最低版本 -->
<properties>
    <jenkins.version>2.440.3</jenkins.version>
    <java.level>8</java.level>
</properties>

<!-- 插件兼容性声明 -->
<extension>
    <artifactId>my-plugin</artifactId>
    <compatibleSince>1.0</compatibleSince>
    <since>1.0</since>
    <title>My Plugin</title>
    <url>https://github.com/myorg/my-plugin</url>
    <releases>https://github.com/myorg/my-plugin/releases</releases>
</extension>
```

## 8.3 文档

```markdown
<!-- README.md -->
# My Plugin

## 功能
- 提供 sayHello Pipeline 步骤
- 可自定义问候语和语言

## 使用方法

```groovy
pipeline {
    agent any
    stages {
        stage('Greeting') {
            steps {
                sayHello 'World'
            }
        }
    }
}
```

## Configuration
Manage Jenkins > Configure System > My Plugin Configuration

## Changelog
### 1.0.0
- Initial release
```
