# 1. 依赖安全风险全景

现代 Java 应用 80% 以上的代码来自开源依赖，`pom.xml` 里一个坐标就牵出整棵传递依赖树——这意味着你的构建产物里混入了大量**你从未读过源码、也不维护**的构件。开源供应链攻击正是利用这一点：不攻击你的代码，攻击你的依赖。构建阶段是拦截这些风险的最后一道、也是成本最低的一道闸门。

供应链风险可以分成四类，治理手段各不相同：

| 风险类型 | 典型形态 | 代表事件 | 治理手段 |
|----------|---------|---------|---------|
| 已知漏洞 | 依赖带 CVE 上线 | log4shell (CVE-2021-44228) | CVE 扫描 + SBOM 持续监控 |
| 名称欺骗 | 仿冒包名诱导装错 | typosquatting、依赖混淆 | 私服白名单 + 精确坐标 |
| 主动投毒 | 版本被劫持后植入恶意代码 | event-stream、xz-utils 后门 | 锁定版本 + 校验和 + 签名 |
| 许可与合规 | 传染性许可证混入 | GPL 污染商业闭源产品 | 许可证过滤 + SCANOSS/FOSSA |

> **注意**：传递依赖是重灾区。你只声明了 `spring-boot-starter-web`，实际引入的可能是上百个 jar；安全扫描必须覆盖**整棵依赖树**（`makeAggregateBom` 而非只看直接依赖），只审计 POM 第一层的做法等于没审计。

# 2. 依赖混淆（Dependency Confusion）

**依赖混淆**是 2021 年由 Alex Birsan 公开的攻击手法，针对的恰恰是企业最信任的东西——内部构件坐标。原理：攻击者在内网 POM 里看到贵司私有构件 `com.company:billing-core` 的坐标后，在公网 Maven Central 上抢注一个**同名坐标**（groupId 用你反编译或泄露里看到的命名习惯，或者在支持 scoped 的仓库里直接同名），并故意把版本号写得更高（如 `99.9.9`）。如果构建工具的解析顺序是"取最高版本、公网仓库可达"，构建时就会**下载攻击者的公网假包而不是内网的真包**，恶意代码从此进入你的产物。

Maven 受此影响相对小（GAV 坐标里 groupId 与域名绑定，Central 对反写域名有所有权校验），但存在配置失误的暴露面：`<repositories>` 声明了公网仓库且排在私服前、SNAPSHOT 依赖没有仓库收敛、或者 `mirrorOf` 没配全。npm/PyPI 生态则是重灾区，同一套内部包名清单泄露后会被批量抢注。

**治理组合**：

- 私服统一收口：`settings.xml` 里 `mirrorOf` 覆盖 `*`，构建机禁止直连公网仓库——外网抢注的包根本进不了解析路径。
- 私服按仓库类型代理：内部 hosted 仓库优先于外部 proxy 仓库，Nexus 的 group 排序保证同名坐标先查内网。
- 内部构件使用**受控 groupId 命名空间**，并让公网仓库对该命名空间保持 404/校验失败，而不是依赖"没人注册"。
- 版本策略锁定：CI 中比对依赖清单（SBOM diff），坐标来源仓库异常即阻断。

# 3. Typosquatting 与坐标卫生

**typosquatting**（仿冒抢注）比依赖混淆更简单粗暴：注册与流行库拼写近似的包名——`commons-langn3`、`guava-lib`、`apache-commoms-codec`——等粗心的开发者打错坐标时自动中招。防范靠构建时的静态校验：

```bash
# 坐标拼写与流行库近似度检查（CI 中人工评审 + 工具结合）
mvn dependency:list | tr ':' ' ' | sort
```

**设计考量**：

- **新增依赖走评审**：坐标变更应像数据库迁移一样进 PR 审——确认 groupId/artifactId 与官方文档一字不差，确认版本是官方发布线上的版本（不存在的版本号本身就是危险信号）。
- **不要"搜到第一个就用"**：从搜索引擎/CSDN 示例里抄坐标是 typosquatting 的经典入口，应从项目官方文档取坐标。
- **私服开启"首次下载审批"**：Nexus/Artifactory 可配置策略——公网新构件首次被拉取时进入 pending 队列人工放行，兼顾效率与拦截。

# 4. CVE 扫描：OWASP Dependency-Check

对已知漏洞的批量比对，最直接的工具是把依赖树与 NVD（National Vulnerability Database）做全文匹配。OWASP Dependency-Check 是 Maven 生态的标配：

```xml
<plugin>
	<groupId>org.owasp</groupId>
	<artifactId>dependency-check-maven</artifactId>
	<version>10.0.4</version>
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

```bash
mvn dependency-check:check
```

- 首次运行会下载 NVD 数据（需要申请 API key，否则限速下数小时），耗时极长；后续运行增量更新缓存，速度显著提升。
- `failBuildOnCVSS=7` 表示 CVSS ≥ 7.0（高危）直接**构建失败**，把安全门禁嵌进流水线而不是事后报告。
- 匹配原理是 CPE 名称模糊匹配，**误报与漏报都不少**——版本号不在 NVD 收录格式里就漏，同名不同库就误报，这是它的固有局限，需要 suppression 流程兜底。

## 4.1 漏洞抑制

`owasp-suppressions.xml`：已知误报或已确认不影响本系统的漏洞可临时抑制：

```xml
<suppressions>
	<suppress>
		<notes>经评估：该 CVE 仅在 X 功能路径触发，本系统未使用 X。评估人：张三，有效期：2026-12-31</notes>
		<cve>CVE-2024-XXXX</cve>
	</suppress>
</suppressions>
```

> **注意**：抑制必须有审批记录、责任人和**有效期**，避免漏洞被无限期"合法忽视"；suppression 文件进版本库，评审记录进工单系统，两者缺一不可。

## 4.2 versions 插件与替代扫描器

Dependency-Check 解决"已知 CVE"，`versions-maven-plugin` 解决"版本落后"——长期停留在带漏洞旧版本的依赖，升级本身就是安全动作：

```bash
# 检查依赖可升级版本（只看安全影响大的落后版本）
mvn versions:display-dependency-updates
mvn versions:display-parent-updates
```

轻量替代：Trivy（Aqua Security）无需维护 NVD 本地库，扫描速度快，适合容器化流水线：

```bash
# 扫描 pom.xml 依赖
trivy fs --scanners vuln --pkg-types pom .

# 扫描构建产物镜像
trivy image --scanners vuln my-app:latest
```

企业里也常见 Snyk、Sonatype Lifecycle（与 Nexus 原生集成，firewall 阻断已知漏洞构件下载）等商业方案——工具可以换，"构建期阻断高危"的口径不变。

# 5. SBOM：软件物料清单

**SBOM**（Software Bill of Materials）把"我用了哪些依赖、什么版本、从哪来"固化为机器可读清单，价值有三层：合规交付（客户与监管要求越来越普遍）、事件响应速度（log4shell 爆出后，有 SBOM 的平台 10 分钟定位受影响服务，没有的全网重新构建扫描）、以及供应链审计的基础数据。

CycloneDX 是目前 Java 生态的主流格式：

```xml
<plugin>
	<groupId>org.cyclonedx</groupId>
	<artifactId>cyclonedx-maven-plugin</artifactId>
	<version>2.8.0</version>
	<executions>
		<execution>
			<phase>package</phase>
			<goals><goal>makeAggregateBom</goal></goals>
		</execution>
	</executions>
</plugin>
```

```bash
mvn cyclonedx:makeAggregateBom
```

- 生成 `target/bom.json`（CycloneDX 格式，另有 `bom.xml`），聚合模块用 `makeAggregateBom` 才能包含全部传递依赖。
- SBOM 不是一次性产物：提交到 **Dependency-Track**（开源 SBOM 持续监控平台）后，新 CVE 公布时会自动比对历史版本，无需重新构建即可发现"上周还干净的包今天爆了雷"。

# 6. 版本锁定与 BOM 治理

供应链安全落到 POM 层的动作只有一句话：**可重现的解析结果**。版本漂移（今天构建和明天构建拉到不同版本）既是稳定性问题，更是安全问题——漂移的窗口就是投毒的窗口。

治理手段：

- 所有依赖版本经父 POM `dependencyManagement` 或框架 BOM 显式声明，禁止子模块自填版本：

```xml
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
```

- `maven-enforcer-plugin` 把规则固化为构建门禁：

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
					<requireUpperBoundDeps />
				</rules>
			</configuration>
		</execution>
	</executions>
</plugin>
```

- `banDynamicVersions`：禁止 `RELEASE`、`LATEST`、版本区间等动态版本。
- `requireReleaseDeps`：release 模式下禁止依赖快照版本。
- `requireUpperBoundDeps`：强制传递依赖版本不低于仲裁结果，堵住"被最近优先规则降级到带漏洞旧版"的暗坑。

> **行业实践**：BOM 由平台组统一维护、随季度评审更新，业务团队只从 BOM 取版本不自定版本；BOM 升级是**安全事件驱动**的常规变更（如 CVE 修复），走完整回归而不是"顺手升到最新"。

# 7. 镜像仓库、代理与白名单

私服（Nexus/Artifactory）在安全视角下不只是缓存加速组件，而是**供应链的边界网关**。收敛原则：构建机的解析路径必须全部经过私服，公网仓库只对私服可达，开发者与流水线都摸不到 repo1.maven.org——`settings.xml` 统一镜像：

```xml
<mirrors>
	<mirror>
		<id>company-proxy</id>
		<mirrorOf>*</mirrorOf>
		<url>https://repo.company.com/repository/maven-public</url>
	</mirror>
</mirrors>
```

在私服侧配置分层策略：

- **白名单仓库**：只代理可信的远程仓库（central、显式批准的第三方仓库），未列入清单的仓库即便 POM 里声明了也拉不到——这是防依赖混淆/仿冒包最有效的架构手段。
- **许可证过滤**：禁止 GPL/AGPL 等高风险传染性许可证的依赖进入（Artifactory Xray / Sonatype Lifecycle 支持元数据级拦截）。
- **漏洞阻断**：集成防火墙插件，有高危漏洞的构件禁止下载，从"构建后扫描报警"前移到"下载即拦截"。
- **重发布保护**：hosted release 仓库开启 disable redeploy，防止已发布构件被同名版本覆盖——覆盖投放是投毒的标准路径。

**设计考量**：镜像 `mirrorOf=*` 意味着私服宕机则全体构建瘫痪，所以安全收口与可用性要一起设计——私服高可用（多副本+负载均衡）、本地仓库作为离线兜底、紧急时按审批临时放开直连并事后全量审计。

# 8. 凭据安全与构件签名

## 8.1 settings.xml 凭据加密

部署到私服、访问受保护仓库的口令不能明文躺在 `settings.xml` 里进版本库或镜像。Maven 自带加密体系（`mvn -emp` 加密主密码、`mvn -ep` 加密口令）提供最低成本的方案：

```bash
# 1. 加密主密码，输出形如 {xxx=}，写入 settings-security.xml
mvn -emp "MyMasterPassw0rd"

# 2. 用主密码加密服务器口令
mvn -ep "deployerPassw0rd"

# 3. settings.xml 中存放密文
# <server><id>releases</id><username>deployer</username>
#   <password>{o/g5cPOS4GYo26t2g+RcoTMJfDw20F78d5UeXrv455Xw=}</password>
```

```xml
<!-- ~/.m2/settings-security.xml 权限必须收紧 -->
<settingsSecurity>
	<master>{加密后的主密码}</master>
</settingsSecurity>
```

> **注意**：Maven 原生 PBE 加密算法强度偏弱（社区长期诟病，密钥可被爆破），它只解决"明文口令不落地"，不解决"口令不可恢复"。高安全要求场景应该用 CI 密钥管理（GitHub Secrets、HashiCorp Vault、K8s Secret + External Secrets）注入，或对 `settings-security.xml`/`settings.xml` 设置严格文件权限（Linux: `chmod 600`），POM 中绝不硬编码密码。

## 8.2 构件签名与校验

私服下载依赖时，Maven 会校验仓库提供的 SHA-1/MD5 校验和，防传输损坏与简单替换；开源发布（Maven Central 强制要求）还要求 **GPG 签名**，每个构件附带 `.asc` 文件：

```bash
# 发布前签名校验（maven-gpg-plugin 绑定 deploy 阶段）
mvn clean deploy   # 产物含 xxx.jar, xxx.jar.asc, xxx.pom.asc

# 消费方验证签名真实性
gpg --verify order-api-1.0.0.jar.asc order-api-1.0.0.jar
```

`maven-artifact-checker` 思路 + `pgpverify-maven-plugin` 可在构建期强制校验"依赖必须有可信 Key 的签名"，把"仓库说这是官方包"升级为"密码学证明这是官方包"——内网私服环境下这是性价比很高的加强项。

# 9. 常见问题排查清单

- OWASP 检查慢：首次需要下载 NVD 数据库（先申请 API key），后续使用缓存；CI 中可将 NVD 缓存目录纳入流水线缓存。
- 误报处理：使用 suppression 文件抑制，附评估说明与有效期。
- SBOM 不包含传递依赖：使用 `makeAggregateBom` 而非 `makeBom`。
- 私服阻断合法依赖：与安全团队沟通白名单策略，勿私自绕过镜像直连公网。
- 加密口令构建失败 "Cannot decrypt server password"：检查 `settings-security.xml` 位置与文件权限，确认加密时用的主密码与当前一致（跨机器拷贝密文容易踩这个坑）。
- 依赖下载来路不明：用 `mvn dependency:tree` + 私服访问日志核对每个坐标的实际来源仓库。

# 10. 小结

Maven 供应链安全的防线是分层的，没有单点银弹：**架构层**用私服镜像收口 + 白名单 + 重发布保护，让"包从哪来"可控；**声明层**用 BOM/父 POM 锁版本 + enforcer 禁动态版本，让"每次解析结果相同"可重现；**检测层**用 Dependency-Check/Trivy 在构建期阻断 CVSS ≥ 7 的 CVE，用 SBOM + Dependency-Track 把监控延伸到发布之后；**凭据与真实性层**用加密 settings.xml（认清其局限）、CI Secret 管理、GPG 签名校验闭环。

落地顺序建议：先做收口（镜像+白名单，一周可完成、拦截面最大），再做锁定（BOM+enforcer），最后铺检测与签名——检测类工具上线初期误报率高，直接卡门禁会引发团队反弹，先观察一个迭代再逐步收紧 `failBuildOnCVSS` 阈值。
