# 1. 站点生成概述与 Site 生命周期

Maven 内置了一条独立于 clean/default 的 **site 生命周期**，目标是把项目的文档与质量报告聚合成一个静态网站：javadoc、测试报告、覆盖率、代码风格、依赖信息……全部由插件在构建时生成，`site.xml` 负责把它们编排成带统一导航和皮肤的页面。它的定位是"**项目自述门户**"——任何人拿到代码库，执行一条命令就能得到一份完整的技术档案。

```bash
mvn site
```

生成后站点位于 `target/site/`，打开 `index.html` 浏览。site 生命周期与 default 生命周期**完全独立**，执行 `mvn site` 不会触发编译打包，而多数报告插件需要上游数据（surefire-report 依赖测试结果、jacoco 依赖 agent 采集），因此"先 `mvn verify` 再 `mvn site`"才能拿到有数据的完整报告。

- 主要阶段：
	- `pre-site`：生成站点前准备
	- `site`：生成站点文档
	- `post-site`：后续处理
	- `site-deploy`：部署站点到服务器

**设计考量**：

- site 生命周期不绑定默认插件，`mvn site` 空跑只产出骨架页——报告必须来自 `<reporting>` 配置；jacoco `prepare-agent` 这类注入场景不会被"按需联动"触发，完整数据要显式先跑 `verify`。
- 报告生成是全量重算，没有增量概念；插件矩阵一大，`mvn site` 本身就是慢构建，这是它后来被现代工具链边缘化的原因之一（见第 8 节）。

# 2. 站点描述符（site.xml）

站点的导航结构、品牌元素与皮肤由 `src/site/site.xml` 定义：

```xml
<!-- src/site/site.xml -->
<project name="Order Service">
	<bannerLeft>
		<name>Order Service</name>
		<src>images/logo.png</src>
	</bannerLeft>

	<body>
		<menu name="Overview">
			<item name="Introduction" href="index.html"/>
			<item name="Getting Started" href="getting-started.html"/>
		</menu>
		<menu name="Documentation">
			<item name="API" href="apidocs/index.html"/>
		</menu>
		<menu ref="reports"/>
	</body>

	<skin>
		<groupId>org.apache.maven.skins</groupId>
		<artifactId>maven-fluido-skin</artifactId>
		<version>1.12.0</version>
	</skin>
</project>
```

- `src/site/resources/`：存放静态资源（CSS、图片、HTML）。
- `<menu ref="reports"/>`：自动插入所有生成的报告。
- `src/site/markdown/` 下的 Markdown 由 Doxia 渲染引擎转成页面并挂进导航，`index.md`（或 `index.xml`）即站点首页。

皮肤机制：site 的产出本质是"内容 + Doxia 模型"，长什么样由 skin 决定。fluido 是最后一代官方皮肤，风格停留在 2010 年代——表格密集、移动端基本不可用。这是 site 劝退新项目的表面原因，真正的问题是内容组织与更新机制，不是 CSS。

# 3. 报告插件矩阵

site 的价值不在骨架而在报告矩阵：每个质量维度对应一个报告插件，在 `<reporting>` 中声明后自动接入 `<menu ref="reports"/>`。

| 维度 | 插件 | 产出 | 前置条件 |
|------|------|------|---------|
| API 文档 | maven-javadoc-plugin | apidocs 全站 javadoc | 源码注释合规 |
| 单元测试 | maven-surefire-report-plugin | 用例通过/失败/耗时报告 | 需先执行 test |
| 覆盖率 | jacoco-maven-plugin | 行/分支覆盖热力图 | 需 prepare-agent |
| 静态检查 | maven-checkstyle-plugin | 命名/格式违规清单 | 规则文件 |
| 代码坏味道 | maven-pmd-plugin / spotbugs | 复杂度、潜在缺陷 | 规则配置 |
| 依赖信息 | maven-project-info-reports-plugin | 依赖树、许可证、SCM | 无 |
| 漏洞 | dependency-check 报告 | CVE 清单 HTML | NVD 缓存 |
| 二进制兼容 | japicmp | API 变更 diff 报告 | 上一发布版本坐标 |

## 3.1 Javadoc

```xml
<reporting>
	<plugins>
		<plugin>
			<groupId>org.apache.maven.plugins</groupId>
			<artifactId>maven-javadoc-plugin</artifactId>
			<version>3.6.3</version>
			<configuration>
				<source>21</source>
			</configuration>
		</plugin>
	</plugins>
</reporting>
```

javadoc 报告是 site 里使用频率最高的产出。建议配 `<doclint>all,-missing</doclint>` 分级控制注释质量，并用 `<links>` 挂上 JDK 与常用框架的在线 javadoc，让类型名可跳转——离线 javadoc 站缺外部链接时阅读体验会大幅下降。

## 3.2 测试报告

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-surefire-report-plugin</artifactId>
	<version>3.2.5</version>
</plugin>
```

surefire-report 读取 `target/surefire-reports` 下的 XML 结果渲染成 HTML：用例统计、失败堆栈、耗时排行。注意报告只反映**最后一次**执行结果，历史趋势要靠站点按版本归档（见第 5 节）。

## 3.3 JaCoCo 覆盖率报告

```xml
<plugin>
	<groupId>org.jacoco</groupId>
	<artifactId>jacoco-maven-plugin</artifactId>
	<version>0.8.12</version>
	<reportSets>
		<reportSet>
			<reports>
				<report>report</report>
			</reports>
		</reportSet>
	</reportSets>
</plugin>
```

jacoco 的 agent 注入必须配在 `<build><plugins>`（`prepare-agent` 目标绑定 `initialize` 阶段），`<reporting>` 里只负责把 exec 数据渲染成 HTML——只配 reporting 会得到一张全 0 的覆盖率报告，这是 jacoco 集成 site 的头号坑。

## 3.4 Checkstyle / PMD 报告

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-checkstyle-plugin</artifactId>
	<version>3.3.1</version>
	<configuration>
		<configLocation>google_checks.xml</configLocation>
	</configuration>
</plugin>
```

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-pmd-plugin</artifactId>
	<version>3.21.2</version>
	<configuration>
		<linkXRef>false</linkXRef>
		<minimumPriority>2</minimumPriority>
	</configuration>
</plugin>
```

checkstyle 管格式（缩进、命名、import 顺序），PMD 管坏味道（圈复杂度、超长方法、疑似 bug）。两者都有"构建期 fail"与"site 报告"两种用法——实践上建议构建期只卡增量违规，全量清单交给 site，避免老项目被历史违规卡死构建。

## 3.5 依赖分析报告

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-project-info-reports-plugin</artifactId>
	<version>3.5.0</version>
	<reportSets>
		<reportSet>
			<reports>
				<report>dependencies</report>
				<report>dependency-info</report>
				<report>index</report>
				<report>summary</report>
			</reports>
		</reportSet>
	</reportSets>
</plugin>
```

`maven-project-info-reports-plugin` 是 site 的"底座"报告——依赖清单、许可证、SCM 链接、插件列表都出自它。很多站点 `<menu ref="reports"/>` 一片空白，原因就是漏配了它（Maven 3 之后不再默认提供）。

# 4. 多模块聚合站点

父 POM 配置聚合后，`site:stage` 会把所有子模块站点拼成一个带相互链接的完整站点：

```xml
<distributionManagement>
	<site>
		<id>company-site</id>
		<url>scp://server.company.com/site/${project.artifactId}</url>
	</site>
</distributionManagement>
```

```bash
# 生成聚合站点（包含所有子模块）
mvn site site:stage
```

站点输出在 `target/staging/`，包含所有子模块的报告链接。

> **注意**：多模块场景要区分两种执行策略——报告插件默认**逐模块**执行，聚合模块要出"全项目合并报告"（如 jacoco `report-aggregate`）需单独配置聚合 goal 并汇总各子模块的 exec 数据，否则父站点覆盖率只统计父模块自身代码。子模块 URL 前缀配置不一致时，跨模块链接在 staging 与 deploy 下表现不同，上线前用 `site:stage` 验证。

# 5. 部署站点：site-deploy 与 Nexus

```bash
mvn site-deploy
```

- 依赖 `distributionManagement` 中的 `<site>` 配置。
- 支持协议：`scp`、`ftp`、`file`、`dav`。

传统做法是把站点推到内部 HTTP 服务器，按 `/{group}/{artifact}/{version}/` 路径归档——这正是 Maven Central 构件页面 "javadoc / References" 链接的同款机制。企业里更常见的形态是**把站点当成一种构件**推进 Nexus：把 `target/site` 打成 war/jar 后 `mvn deploy` 到 hosted 仓库（坐标如 `com.company.docs:order-service-site`），再由静态服务器解压发布——站点从此获得与构件一致的**版本化、权限与保留策略**，旧版本文档不会被误删，多版本并存可对照。

CI 中的典型串联：

```bash
mvn -B clean verify
mvn -B site site:stage
# 仅主干分支部署站点
[ "$CI_BRANCH" = "main" ] && mvn -B site-deploy
```

`site-deploy` 的 server id 要与 `settings.xml` 的 `<server><id>` 一致，凭据走加密或环境变量注入，不要把口令写进 POM。

# 6. japicmp：发布前的 API 兼容性检查

对二方库/SDK 项目，报告矩阵里最值得加的是 **japicmp**——对比当前构建产物与上一个发布版本的 jar，把公共 API 变更分成"二进制兼容/不兼容"两级，防止无意的重构把下游全部编译打断：

```xml
<plugin>
	<groupId>com.github.siom79.japicmp</groupId>
	<artifactId>japicmp-maven-plugin</artifactId>
	<version>0.21.0</version>
	<configuration>
		<oldVersion>
			<dependency>
				<groupId>com.company</groupId>
				<artifactId>order-api</artifactId>
				<version>1.0.0</version>
				<type>jar</type>
			</dependency>
		</oldVersion>
		<parameter>
			<breakBuildOnBinaryIncompatibleModifications>true</breakBuildOnBinaryIncompatibleModifications>
			<onlyBinaryIncompatible>true</onlyBinaryIncompatible>
			<semanticVersioning>TRUE</semanticVersioning>
			<ignoreMissingClasses>true</ignoreMissingClasses>
			<packageExcludes>
				<packageExclude>com.company.internal.*</packageExclude>
			</packageExcludes>
		</parameter>
	</configuration>
	<executions>
		<execution>
			<goals><goal>cmp</goal></goals>
		</execution>
	</executions>
</plugin>
```

`<parameter>` 段关键项：`breakBuildOnBinaryIncompatibleModifications=true` 让不兼容变更直接**卡构建**；`semanticVersioning=TRUE` 依据变更类型建议下一版本号（patch/minor/major），把 semver 纪律落到工具；`packageExcludes` 排除内部包，避免内部实现变更被误报为破坏性变更。

> **设计争议**：二进制兼容检查该放在 site 报告还是 CI 门禁？只出报告等于"知道断了但继续发版"，因此 japicmp 的正确姿势是报告与门禁**双用**——site 上留 diff 明细供评审，CI 里开启 breakBuild 阻断合并，发布 tag 前再人工确认一次报告。

# 7. API 文档自动化：OpenAPI 与 javadoc 的分工

site 生成的 javadoc 只解决"类和方法怎么用"，微服务时代更刚性的需求是"接口契约怎么给前端和联调方"。主流做法是让规范文件从代码里**自动长出来**，而不是手写维护：

```xml
<!-- springdoc-openapi：运行时从 Controller 注解生成 OpenAPI 3 规范 -->
<dependency>
	<groupId>org.springdoc</groupId>
	<artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
	<version>2.6.0</version>
</dependency>
```

```bash
# 启动应用后导出规范文件
curl http://localhost:8080/v3/api-docs -o target/openapi.json
```

**设计考量**：

- **分工而非替代**：javadoc 面向库消费者（SDK、starter），OpenAPI 面向 HTTP API 消费者（前端、网关、自动化测试）；二方库维护 javadoc 站点，微服务把 CI 重心放在 OpenAPI 导出。
- **契约构件**：OpenAPI 规范文件纳入版本管理，CI 导出 `openapi.json` 推送文档平台，用 `oasdiff` 比对历史版本，破坏性变更直接卡 PR；构建期不启服务可用 smart-doc 从 Javadoc 注释生成。

# 8. 趋势：site 正在被静态站与专用平台取代

2010 年前后 `mvn site` 几乎是 Java 项目文档的标准答案；2020 年后的新建项目里，完整配置 site 生命周期的已属少数。原因是文档需求被拆分给了更专业的工具，site 的每一项能力都有了更好的替代品：

| 能力 | Maven site | 现代替代 | 替代的优势 |
|------|-----------|---------|-----------|
| 叙事性文档 | src/site/markdown | Docusaurus / MkDocs Material / VitePress | 版本化、搜索、MDX 组件、移动端 |
| 构建产物托管 | 自建站点服务器 | GitHub/GitLab Pages | 零运维、PR 预览环境 |
| API 契约文档 | 无 | Swagger UI / Redoc / smart-doc / javap | 从代码自动生成、机器可读 |
| 覆盖率/测试报告 | site 页面 | Codecov / Allure / PR check 注解 | 历史趋势、增量 diff、与评审联动 |
| 质量门禁 | PMD/CPD 报告页 | SonarQube Quality Gate | 规则引擎、历史对比、阻断合并 |
| 依赖漏洞 | 无原生支持 | Dependency-Track / Snyk | 持续监控、SBOM 驱动 |

> **行业实践**：新项目叙事文档交给 Docs-as-Code（Markdown 进仓库、CI 发布静态站），质量指标交给 SonarQube 与覆盖率平台，API 文档从代码自动生成。Maven site 保留在**二方库/SDK 项目**里，主要为了产出 javadoc jar 并发布到私服供 IDE 关联——报告网站功能本身基本不再有人消费。

老项目是否要"补一个 site"，看产物给谁用：给**人**看的趋势指标别建在 site 上（sonar 已有历史曲线）；给**下游编译期**用的 javadoc/sources jar，site 生命周期反而是最顺的生成通道；纯为"看起来规范"复刻 fluid 皮肤站点属于负资产，不建议新增投入。

# 9. 常见问题排查清单

- 站点为空：确认已在 `<reporting>` 中配置报告插件，且补配了 `maven-project-info-reports-plugin`。
- 模块聚合站缺少子模块：检查父 POM 的 `<modules>` 与子模块对父 `<reporting>` 的继承。
- 覆盖率报告全 0：jacoco `prepare-agent` 必须绑在 build 阶段，`site` 前先跑 `mvn verify`。
- 报告生成失败：确认插件版本兼容 Maven 3.9.x（旧插件常见 `NoSuchMethodError`，需升级）。
- 部署失败：确认 `settings.xml` 的 `server.id` 与 `<site>` 的 `id` 一致，`scp`/`dav` 需额外部署 wagon 扩展。
- 多模块链接错乱：`site:stage` 本地相对链接与 deploy 绝对 URL 各验证一遍，确认无断链。

# 10. 小结

Maven site 是一条"把项目档案化"的流水线：site 生命周期负责编排，`site.xml` 负责结构，报告插件矩阵（javadoc、surefire-report、jacoco、checkstyle/PMD、project-info、japicmp）负责内容，`site:stage` 负责聚合验证，`site-deploy`/Nexus 负责版本化归档。掌握它的价值，一是维护存量二方库站点与 javadoc jar 发布，二是理解"构建期质量报告"这一模式本身。

同时清醒地看到：叙事文档、质量门禁、API 契约都有了更强的专业工具，site 的新增使用面在持续收窄。合理姿势是**存量维护、增量克制**——新项目用 Docs-as-Code + Sonar + OpenAPI 组合，把 Maven 文档投入聚焦在"下游要用的 javadoc/sources 构件"上。
