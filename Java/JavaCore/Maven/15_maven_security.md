
# 1. 依赖安全风险

- 开源依赖可能存在已知漏洞（CVE）。
- 传递依赖可能引入未预期的危险库。
- 解决方案：在构建中集成漏洞扫描，阻断高风险依赖。

# 2. OWASP Dependency-Check

## 2.1 配置

```xml
<plugin>
	<groupId>org.owasp</groupId>
	<artifactId>dependency-check-maven</artifactId>
	<version>9.2.0</version>
	<configuration>
		<failBuildOnCVSS>7</failBuildOnCVSS>
		<formats>
			<format>HTML</format>
			<format>JSON</format>
		</formats>
		<suppressionFiles>
			<suppressionFile>owasp-suppressions.xml</suppressionFile>
		</suppressionFiles>
	</configuration>
	<executions>
		<execution>
			<goals><goal>check</goal></goals>
		</execution>
	</executions>
</plugin>
```

## 2.2 执行

```bash
mvn dependency-check:check
```

- 首次运行会下载 NVD（National Vulnerability Database）数据，耗时较长。
- 后续运行使用缓存，速度显著提升。

## 2.3 漏洞抑制

`owasp-suppressions.xml`：已知误报或已确认安全的漏洞可临时抑制：

```xml
<suppressions>
	<suppress>
		<notes>经评估此漏洞不影响本系统</notes>
		<cve>CVE-2024-XXXX</cve>
	</suppress>
</suppressions>
```

- 抑制必须有审批记录，避免漏洞被忽视。

# 3. Trivy（Aqua Security）

- 支持对 Maven 依赖进行漏洞扫描：

```bash
# 扫描 pom.xml
trivy fs --scanners vuln --pkg-types pom .

# 扫描构建产物
trivy image --scanners vuln my-app:latest
```

# 4. SBOM 生成（Software Bill of Materials）

## 4.1 CycloneDX

```xml
<plugin>
	<groupId>org.cyclonedx</groupId>
	<artifactId>cyclonedx-maven-plugin</artifactId>
	<version>2.8.0</version>
	<executions>
		<execution>
			<phase>verify</phase>
			<goals><goal>makeAggregateBom</goal></goals>
		</execution>
	</executions>
</plugin>
```

```bash
mvn cyclonedx:makeAggregateBom
```

- 生成 `target/bom.json`（CycloneDX 格式）。
- SBOM 可提交到 Dependency-Track 等平台进行持续监控。

# 5. 私服安全策略

- Nexus/Artifactory 可以配置：
	- **白名单仓库**：只代理可信的远程仓库。
	- **许可证过滤**：禁止 GPL/AGPL 等高风险许可证的依赖进入。
	- **漏洞阻断**：集成防火墙插件，有高危漏洞的构件禁止下载。
- `settings.xml` 中统一镜像到公司私服，避免直连外部仓库。

# 6. 密钥与凭据安全

- POM 中不硬编码密码——使用环境变量或 `settings.xml` 加密（参见第 6 章）。
- 在 CI 中使用密钥管理服务（如 GitHub Secrets、Vault）。
- 对 `settings-security.xml` 设置严格文件权限（Linux: `chmod 600`）。

# 7. maven-enforcer-plugin 安全规则

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-enforcer-plugin</artifactId>
	<version>3.4.1</version>
	<executions>
		<execution>
			<goals><goal>enforce</goal></goals>
			<configuration>
				<rules>
					<banDynamicVersions />
					<requireReleaseDeps>
						<message>禁止使用 SNAPSHOT 依赖发布正式版本</message>
						<onlyWhenRelease>true</onlyWhenRelease>
					</requireReleaseDeps>
				</rules>
			</configuration>
		</execution>
	</executions>
</plugin>
```

- `banDynamicVersions`：禁止 `RELEASE`、`LATEST` 等动态版本。
- `requireReleaseDeps`：release 模式下禁止依赖快照版本。

# 8. 常见问题排查清单

- OWASP 检查慢：首次需要下载 NVD 数据库，后续使用缓存。
- 误报处理：使用 suppression 文件抑制，附评估说明。
- SBOM 不包含传递依赖：使用 `makeAggregateBom` 而非 `makeBom`。
- 私服阻断合法依赖：与安全团队沟通白名单策略。
