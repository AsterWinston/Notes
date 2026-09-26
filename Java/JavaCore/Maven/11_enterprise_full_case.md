
# 1. 目标与范围

- 目标：搭建可在企业落地的多模块 Maven 工程，支持私服发布与 CI 构建。
- 范围：模块划分、父 POM、依赖治理、发布配置、CI 构建命令。

# 2. 推荐模块结构

```
company-parent
├── common
├── service-api
├── service-impl
└── web
```

- `common`：通用工具与基础组件
- `service-api`：对外接口与 DTO
- `service-impl`：业务实现
- `web`：Web 层或接口层

# 3. 父 POM 统一管理

```xml
<project>
	<modelVersion>4.0.0</modelVersion>
	<groupId>com.company</groupId>
	<artifactId>company-parent</artifactId>
	<version>1.0.0</version>
	<packaging>pom</packaging>

	<modules>
		<module>common</module>
		<module>service-api</module>
		<module>service-impl</module>
		<module>web</module>
	</modules>

	<properties>
		<java.version>21</java.version>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
	</properties>

	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-dependencies</artifactId>
				<version>3.3.4</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>

	<build>
		<pluginManagement>
			<plugins>
				<plugin>
					<groupId>org.apache.maven.plugins</groupId>
					<artifactId>maven-compiler-plugin</artifactId>
					<version>3.11.0</version>
					<configuration>
						<source>${java.version}</source>
						<target>${java.version}</target>
					</configuration>
				</plugin>
			</plugins>
		</pluginManagement>
	</build>
</project>
```

# 4. 子模块 POM 示例

```xml
<project>
	<parent>
		<groupId>com.company</groupId>
		<artifactId>company-parent</artifactId>
		<version>1.0.0</version>
	</parent>
	<artifactId>service-impl</artifactId>

	<dependencies>
		<dependency>
			<groupId>com.company</groupId>
			<artifactId>service-api</artifactId>
			<version>${project.version}</version>
		</dependency>
	</dependencies>
</project>
```

# 5. 私服发布配置

```xml
<distributionManagement>
	<repository>
		<id>company-releases</id>
		<url>https://repo.company.com/releases</url>
	</repository>
	<snapshotRepository>
		<id>company-snapshots</id>
		<url>https://repo.company.com/snapshots</url>
	</snapshotRepository>
</distributionManagement>
```

`settings.xml` 凭据示例：
```xml
<servers>
	<server>
		<id>company-releases</id>
		<username>deploy</username>
		<password>******</password>
	</server>
	<server>
		<id>company-snapshots</id>
		<username>deploy</username>
		<password>******</password>
	</server>
</servers>
```

# 6. CI 构建命令模板

```bash
mvn -B -V -DskipTests clean package
mvn -B -DskipTests deploy
```

建议：
- 缓存 `~/.m2/repository`。
- 注入统一 `settings.xml`。
- 保留测试报告用于排查。

# 7. 发布流程建议

- 开发阶段使用 `SNAPSHOT`。
- 合并主分支后发布正式版本。
- 发布后打 Git Tag 并记录发布说明。

# 8. Docker 镜像构建（jib-maven-plugin）

- Google Jib 无需 Dockerfile、无需 Docker daemon，直接在 Maven 中构建镜像推送：

```xml
<plugin>
	<groupId>com.google.cloud.tools</groupId>
	<artifactId>jib-maven-plugin</artifactId>
	<version>3.4.2</version>
	<configuration>
		<from>
			<image>eclipse-temurin:21-jre</image>
		</from>
		<to>
			<image>registry.company.com/${project.artifactId}:${project.version}</image>
			<auth>
				<username>${env.REGISTRY_USER}</username>
				<password>${env.REGISTRY_PASS}</password>
			</auth>
		</to>
		<container>
			<ports>
				<port>8080</port>
			</ports>
		</container>
	</configuration>
</plugin>
```

执行：
```bash
mvn jib:build
# 或本地构建不推送：mvn jib:dockerBuild
```

- 优点：无需编写 Dockerfile、无需安装 Docker、构建速度快、分层优化。

# 9. SonarQube 代码质量集成

```xml
<properties>
	<sonar.host.url>https://sonarqube.company.com</sonar.host.url>
	<sonar.projectKey>com.company:parent</sonar.projectKey>
	<sonar.java.coveragePlugin>jacoco</sonar.java.coveragePlugin>
</properties>
```

在 CI 中执行：
```bash
mvn -B clean verify sonar:sonar
```

- JaCoCo 覆盖率报告自动上传到 SonarQube，结合质量门禁阻断低质量代码合入。

# 10. Git 分支与版本策略

| 分支 | 用途 | Maven 版本 |
|------|------|-----------|
| `develop` | 日常开发 | `SNAPSHOT` |
| `release/x.y` | 发布候选 | `RC` 或固定版本 |
| `main` | 生产代码 | 正式发布版本（Tag） |
| `hotfix/x.y.z` | 紧急修复 | 增量版本 |

发布流程：
```
develop(1.0.0-SNAPSHOT)
   │
   ├── release/1.0 (mvn versions:set -DnewVersion=1.0.0)
   │       ├── QA 验证
   │       ├── mvn deploy
   │       ├── git tag v1.0.0
   │       └── 合并到 main
   │
   └── develop 升级到 1.1.0-SNAPSHOT
```

CI 友好版本命令：
```bash
# 开发阶段
mvn -Drevision=1.0.0 -Dchangelist=-SNAPSHOT clean deploy

# 正式发布
mvn -Drevision=1.0.0 -Dchangelist= clean deploy
```

# 11. 端到端 CI/CD 流水线示例

```
[代码推送到 develop]
       │
       ▼
[CI: mvn -B clean verify]  ← 编译 + 单元测试 + 集成测试
       │
       ▼
[SonarQube 质量门禁]  ← sonar:sonar, Jacoco 覆盖率
       │
       ▼
[Docker 镜像构建]  ← mvn jib:build
       │
       ▼
[推送镜像到 Registry]
       │
       ▼
[部署到 Dev 环境]  ← Helm / kubectl apply
       │
       ▼
[自动化测试]
       │
       ▼
[部署到 Staging / Canary]
       │
       ▼
[部署到 Production]  ← Blue/Green 或滚动更新
```

# 12. Canary 与 Blue-Green 部署策略

- **Blue-Green**：维护两套完整环境（蓝/绿），切换流量。Maven 不直接负责，由 K8s Service 或 Nginx 控制。
- **Canary**：逐渐将流量切到新版本：
  1. `mvn deploy` -> 镜像 `app:v2.0`
  2. K8s 部署 `app-canary`，配置 5% 流量
  3. 监控无异常 → 逐步提高到 100%
  4. 下线旧版本

- 回滚策略：
```bash
# 回滚到上一个正式版本（基于 Git Tag）
git checkout v1.9.0
mvn -B -DskipTests deploy
```

# 13. 常见问题排查清单

- 子模块构建失败：检查模块依赖与构建顺序。
- 发布失败：确认私服权限与 `server.id` 一致。
- 依赖冲突：使用 `mvn dependency:tree -Dverbose` 排查。
- CI 构建慢：检查镜像与仓库缓存是否生效。
