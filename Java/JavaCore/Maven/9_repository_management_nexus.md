
# 1. 远程仓库类型

- `releases`：正式版本仓库。
- `snapshots`：快照版本仓库。
- `pluginRepository`：插件仓库（与依赖仓库分开配置）。

```xml
<repositories>
	<repository>
		<id>central</id>
		<url>https://repo.maven.apache.org/maven2</url>
		<releases><enabled>true</enabled></releases>
		<snapshots><enabled>false</enabled></snapshots>
	</repository>
</repositories>
```

# 2. 私服基础（Nexus/Artifactory）

- 私服类型：
	- `proxy`：代理中央仓库
	- `hosted`：存放公司内部构件
	- `group`：聚合多个仓库入口
- 常见企业做法：
	- 使用 group 仓库作为统一下载入口
	- 内部构件只允许发布到 hosted

## 2.1 Nexus 三种仓库类型图解

```
用户 / CI
   │
   ▼
┌───────────────────────────┐
│     Group (聚合入口)        │
│  maven-public = group      │
│  ┌───────────────────────┐ │
│  │   proxy (代理)         │ │
│  │   - maven-central     │ │  ──► Maven Central
│  │   - maven-apache      │ │  ──► Apache Snapshots
│  │   - jcenter           │ │  ──► JCenter
│  ├───────────────────────┤ │
│  │   hosted (宿主)       │ │
│  │   - releases          │ │  ◄── mvn deploy (正式版本)
│  │   - snapshots         │ │  ◄── mvn deploy (快照版本)
│  └───────────────────────┘ │
└───────────────────────────┘
```

- **proxy**：缓存远程仓库内容，首次获取后本地不重复下载。
- **hosted**：存放企业内部构件的最终位置。
- **group**：对外暴露唯一 URL，内部转发到 proxy 和 hosted。

## 2.2 Nexus 基础搭建步骤

1. 下载 Nexus Repository Manager OSS（`sonatype/nexus3` Docker 镜像或 war 包）。
2. 启动后访问 `http://localhost:8081`，默认管理员 `admin`，初始密码在 `nexus-data/admin.password`。
3. 创建 Blob Store（存储位置）：`Settings -> Blob Stores`。
4. 创建 Repositories：
   - `maven-central`（proxy → `https://repo1.maven.org/maven2`）
   - `maven-releases`（hosted，`Disable redeploy`）
   - `maven-snapshots`（hosted，允许 redeploy）
   - `maven-public`（group，包含以上三个）
5. 配置清理策略（Cleanup Policies）：定期删除旧快照版本。
6. 配置安全域（Realm）：支持 LDAP 集成、角色与权限控制。

## 2.3 Staging 仓库

- Staging 是发布前的"暂存区"——正式版先部署到 staging，验证后再 release。
- 流程：
	1. `mvn deploy` -> staging 仓库（临时）。
	2. QA / 审核：检查构件完整性、兼容性、签名。
	3. Close（锁定 staging 仓库，禁止修改）。
	4. Release（发布到正式 releases 仓库）或 Drop（放弃）。
- 适用于严格的企业发布流程和开源 Maven Central 发布。

## 2.4 OSSRH 发布（开源）

- OSSRH（OSS Repository Hosting）是 Sonatype 提供的免费开源项目托管服务。
- 申请后获得 staging 仓库，通过前述流程发布到 Maven Central。
- 核心要求：sources.jar + javadoc.jar + GPG 签名 + 完整 POM 元信息。

# 3. 发布构件

- 通过 `distributionManagement` 配置发布地址。
- 使用 `mvn deploy` 上传构件。

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

# 4. 镜像与统一下载入口

- 推荐在 `settings.xml` 配置镜像，确保所有依赖走私服：
```xml
<mirrors>
	<mirror>
		<id>company</id>
		<mirrorOf>*</mirrorOf>
		<url>https://repo.company.com/group</url>
	</mirror>
</mirrors>
```

# 5. 仓库策略

- `releases` 禁止上传 `SNAPSHOT`。
- `snapshots` 允许覆盖更新。
- 常见策略：
	- `updatePolicy`：更新频率
	- `checksumPolicy`：校验策略

# 6. 依赖安全

- 私服可结合漏洞扫描、黑名单策略。
- 建议：
	- 固定版本，避免动态版本（如 `LATEST`）。
	- 在 CI 中执行依赖漏洞扫描。

# 7. 常见问题排查清单

- 401/403：检查 `settings.xml` 凭据与权限。
- 404：确认仓库 URL 与仓库类型（releases/snapshots）。
- 依赖下载不到：确认私服是否已代理并缓存。
- 插件下载失败：检查 `pluginRepositories` 配置。
- checksum 校验失败：删除本地缓存并重新下载。

# 8. 实战案例：私服发布与依赖下载

目标：发布公司内部构件，并通过私服下载。

发布配置：
```xml
<distributionManagement>
	<repository>
		<id>company-releases</id>
		<url>https://repo.company.com/releases</url>
	</repository>
</distributionManagement>
```

执行发布：
```bash
mvn -B -DskipTests deploy
```

客户端通过私服镜像下载依赖。
