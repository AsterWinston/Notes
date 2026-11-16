# 1. 为什么把镜像构建纳入 Maven

传统流程是"CI 里先 `mvn package` 产出 fat jar，再交给 docker build 打镜像"，两个工具链之间靠 shell 脚本胶水粘合。痛点很具体：多阶段 Dockerfile 把整个 fat jar 作为一层，**任何一行代码改动都会使依赖层缓存失效**，一个 200MB 的 Spring Boot jar 每次重新 COPY、重新传输；Docker daemon 的可用性成为构建环境的隐性依赖（CI runner 要挂 docker.sock 或起 dind）；jar 版本、镜像 tag、K8s manifest 三处版本号靠脚本手工同步，漏同步就部署错版本。

把镜像构建**收编进 Maven 生命周期**的价值正在于此：插件天然理解 classpath，能做依赖/资源/类的分层；tag 可以直接取 `${project.version}`，与 POM 单一事实来源对齐；`mvn deploy` 之后 `mvn jib:build` 一条命令完成"构件 + 镜像"双发布，不再需要守护进程。

> **注意**：镜像构建进 Maven 不等于"构建环境也容器化"。jib 路线是 Maven 跑在宿主机、产物直接推 Registry；而 Buildpacks/BuildKit 路线需要容器运行时可用。CI 选型时这是对环境依赖的分水岭。

# 2. jib-maven-plugin 原理与配置

Google Jib 是目前 Maven 生态做容器化的**事实标准**。它的两个反直觉设计：

- **不需要 Dockerfile，也不需要 Docker daemon**：jib 直接按 OCI 规范构建镜像层，通过 Registry HTTP API 逐层上传，构建发生在 JVM 里。
- **分层构建**：jib 把应用拆成依赖层、资源层（resources）、类层（classes）三层分别打包。改一行业务代码只有类层失效，依赖层命中 Registry 缓存秒传，这正是解决 fat jar "一层全量重传"问题的钥匙。

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
			<tags>
				<tag>latest</tag>
			</tags>
		</to>
		<container>
			<ports>
				<port>8080</port>
			</ports>
			<environment>
				<SPRING_PROFILES_ACTIVE>prod</SPRING_PROFILES_ACTIVE>
			</environment>
			<jvmFlags>
				<jvmFlag>-XX:MaxRAMPercentage=75.0</jvmFlag>
			</jvmFlags>
			<mainClass>com.example.Application</mainClass>
			<creationTime>USE_CURRENT_TIMESTAMP</creationTime>
		</container>
	</configuration>
</plugin>
```

```bash
# 构建镜像并推送到 Registry
mvn jib:build

# 构建到本地 Docker daemon（不推送）
mvn jib:dockerBuild

# 构建 tar 文件
mvn jib:buildTar
```

- 优点：无需 Dockerfile；分层缓存优化（依赖层、资源层、类层分离）；无需安装 Docker（直接推送到 Registry）；层内容与时间戳策略可通过 `<execution><creationTime>` 等配置项细控，保证构建可复现。
- 局限：分层粒度固定为"依赖/资源/类"，对 Spring Boot **repackage 后的 fat jar 无法直接分层**——正确做法是让 jib 消费原始 jar/展开 classpath（jib 在 `process-classes` 阶段工作，天然拿的是目录），或接受"Boot loader 布局整体一层"；对需要复杂构建步骤（如安装 native 库、执行初始化脚本）的场景，没有 Dockerfile 表达力，只能退 jib 的 extraDirectories 或换路线。

```bash
# Registry 凭据：先登录，jib 自动复用 ~/.docker/config.json
docker login registry.company.com
# 或在 POM 里用 <to><auth> 引用 settings.xml server 凭据
```

# 3. Jib 与 spring-boot-buildpack 对比

Spring Boot 2.3+ 通过 `spring-boot-maven-plugin` 集成 Cloud Native Buildpacks：

```xml
<plugin>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-maven-plugin</artifactId>
	<configuration>
		<image>
			<name>registry.company.com/${project.artifactId}:${project.version}</name>
			<env>
				<BP_JVM_VERSION>21</BP_JVM_VERSION>
				<BPL_JVM_HEAD_ROOM>20</BPL_JVM_HEAD_ROOM>
			</env>
			<builder>paketobuildpacks/builder-jammy-base:latest</builder>
		</image>
		<!-- 分层提示：repackage 时生成 layered jar，buildpack 据此分层拷贝 -->
		<layers>
			<enabled>true</enabled>
		</layers>
	</configuration>
</plugin>
```

```bash
mvn spring-boot:build-image
```

两条路线的取舍非常清晰：

| 维度 | Jib | Buildpacks (spring-boot:build-image) |
|------|-----|--------------------------------------|
| 是否需要 Docker 环境 | **不需要**，纯 JVM 实现 | 需要本地容器运行时（daemon 或兼容 API） |
| 定制能力 | 靠 XML 配置项，复杂逻辑弱 | 靠 builder 镜像与 detect/build 阶段，表达力强 |
| 基础镜像维护 | 自己跟踪 temurin 等镜像更新 | 由 Paketo 等 builder 统一维护 OS 层与依赖修补 |
| 分层 | 依赖/资源/类三层，稳定可预期 | 按 Spring Boot 布局智能分层（含层顺序 hint） |
| 镜像体积 | 可控（自己选 slim 基础镜像） | builder 侧带调优（如自动 JRE 裁剪），通常也很小 |
| 非 JVM 构建步骤 | 难以插入 | 可通过自定义 buildpack 插桩 |
| 心智模型 | "Maven 插件产出镜像" | "平台负责从源码/产物到镜像" |

**设计考量**：企业选型的一个关键分水岭是**谁来为 OS 层安全负责**。Jib 路线上，基础镜像的 CVE 修补是你自己的事（需要定期重建、镜像扫描）；Buildpacks 路线上，Paketo/Google 这类 builder 供应商负责及时出补丁版本，你升级 builder tag 即可。金融类客户强监管"谁动了我的镜像层"时，Jib 的显式可控更受欢迎；追求零运维、纯 Spring 技术栈时 buildpacks 更省心。

# 4. 接入生命周期与多模块策略

```xml
<executions>
	<execution>
		<id>jib-build</id>
		<phase>package</phase>
		<goals><goal>build</goal></goals>
	</execution>
</executions>
```

把 jib 绑定到 `package`/`verify` 后，`mvn verify` 一步完成"测试 + 打镜像 + 推送"。多模块工程里的规则：**聚合 POM 与 library 模块不配 jib，只有可运行的 boot 模块配**；library 变更不应触发镜像重建，靠 `<skip>true</skip>` 的 profile 或直接在子模块 POM 声明来隔离。CI 中建议把镜像构建做成独立 profile（`-Pdocker`），让"跑测试"与"出镜像"可以解耦触发，避免每次 PR 构建都推送 Registry：

```bash
mvn -B verify -Pdocker -Djib.to.tags=${GIT_SHA::7}
```

# 5. 版本标签传递

跨"Maven → Registry → K8s"三段链路的版本号，最容易出的事故是三处不一致。原则只有一个：**POM 版本是唯一事实来源，其余位置全部引用它**。

```text
pom.xml (1.4.2)
   │  mvn versions:set -DnewVersion=1.4.2   ← release 流程统一改版本
   ▼
jib <to><image> ...:${project.version}       ← 镜像 tag = 1.4.2
   │  mvn jib:build
   ▼
Registry: order-service:1.4.2 (+ digest sha256:ab12...)
   │  CI 输出 image digest 到 next-version 文件
   ▼
Helm: helm upgrade --set image.tag=1.4.2     ← 或 --set image.digest=sha256:ab12...
   ▼
Deployment: image: ...@sha256:ab12...         ← 生产按 digest 固定，tag 仅为人读
```

> **行业实践**：tag 用于人读与流转，**生产部署按 digest（`@sha256:...`）固定**。同 tag 被覆盖重推是"线上跑的和我构建的不一样"的经典成因；按 digest 部署可让 K8s 审计链完全可追溯。Git SHA 只用于 PR/临时环境 tag，正式发布一律用语义化版本号。

# 6. K8s 部署串联：CI 全链路示例

```yaml
# .github/workflows/release.yml —— 主干合并后触发
jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build & Push Docker Image
        run: |
          mvn -B -ntp verify -Pdocker \
            -Djib.to.tags=${{ github.sha }}
        env:
          DOCKER_CONFIG: ${{ secrets.NEXUS_DOCKER_CONFIG }}

      - name: Update K8s Deployment
        run: |
          kubectl set image deployment/order-service \
            app=registry.company.com/order-service:${{ github.sha }} \
            --namespace=prod
          kubectl rollout status deployment/order-service \
            --namespace=prod --timeout=180s
```

```yaml
# helm values.yaml
image:
  repository: registry.company.com/order-service
  tag: 1.0.0
  pullPolicy: IfNotPresent
```

```bash
# 版本化发布：CI 中由 Helm 接管部署参数
mvn jib:build -Djib.to.tags=${CI_COMMIT_TAG}
helm upgrade order-service ./helm-chart \
  --set image.tag=${CI_COMMIT_TAG} \
  --wait --timeout 5m \
  --atomic --force-conflicts
```

`--atomic` 让失败自动回滚到上一 release，`--wait` 保证 rollout 就绪后流水线才算成功——这两项是"Helm 部署串进 CI"必须显式声明的，否则 push 完镜像、helm 秒返回，真实故障要等人工巡检才发现。`pullPolicy` 从 `Always` 改为 `IfNotPresent`+digest 部署，可以显著降低对 Registry 的拉取压力。

# 7. K8s 健康检查与优雅停机配合

```yaml
# deployment.yaml
spec:
  template:
    spec:
      containers:
        - name: order-service
          image: registry.company.com/order-service@sha256:ab12cd34
          ports:
            - containerPort: 8080
          resources:
            requests: { cpu: 500m, memory: 512Mi }
            limits:   { memory: 1Gi }
          startupProbe:
            httpGet: { path: /actuator/health/liveness, port: 8080 }
            failureThreshold: 30
            periodSeconds: 5
          livenessProbe:
            httpGet: { path: /actuator/health/liveness, port: 8080 }
            periodSeconds: 10
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: 8080 }
            periodSeconds: 5
          lifecycle:
            preStop:
              exec: { command: ["/bin/sh", "-c", "sleep 10"] }
```

三件套的分工：**startupProbe 给慢启动兜底**（避免 JVM 还没 ready 就被 liveness 杀掉），liveness 管"僵死重启"，readiness 管"流量摘除"。JVM 冷启动慢是容器化 Java 的头号运维问题，没有 startupProbe 时 liveness 的 `initialDelaySeconds` 很难定——定短了误杀，定长了真故障发现慢。

> **注意**：readiness 与 Service 端点摘除、滚动更新是**协同**关系。Spring Boot 需配 `management.endpoint.health.probes.enabled=true`（K8s 环境自动开启），并配合 `server.shutdown=graceful` 与 `spring.lifecycle.timeout-per-shutdown-phase=30s`，让 Pod 收到 SIGTERM 后处理完在途请求再退出；`preStop sleep` 给 kube-proxy 更新转发规则留窗口，否则滚动发布期间必然出现一波 503/连接重置。

# 8. 镜像瘦身与常见坑

| 手段 | 效果 | 说明 |
|------|------|------|
| JRE 基础镜像替代 JDK | -100~200MB | `eclipse-temurin:21-jre` / `-jre-alpine`；需要 jcmd/attach 等工具时再上 JDK |
| jib 分层保持依赖稳定 | 层复用命中 | 改代码不动依赖时，Registry 只传类层（几 MB） |
| CDS / AppCDS（21 默认开启） | 启动提速 | 镜像里预置 `appcds.jsa` 可进一步降低启动耗时 |
| `<from><image>` 锁 digest | 可复现 | `temurin:21-jre@sha256:...` 防止上游镜像漂移 |
| 排除 fat jar 直接 COPY | 体积/缓存双优 | jib 走目录模式而非 boot fat jar 模式 |

常见问题排查：

- Jib 推送失败：检查 Registry 地址和凭据，确认网络可达；私有 CA 证书需导入 JDK truststore 或配 `<allowInsecureRegistries>`（仅限测试）。
- 镜像构建慢/层不缓存：检查是否每次 `<creationTime>` 都变导致层哈希变化——用固定时间或 `USE_CURRENT_TIMESTAMP` 明确策略。
- K8s 拉取镜像失败：检查 `imagePullSecrets` 配置；私有 Registry 在节点侧要确认 DNS 与证书链。
- Spring Boot 打包镜像过大：确认基础镜像是 JRE、依赖层没把 `spring-boot-devtools` 之类的 test scope 依赖带进 compile classpath。
- buildpacks 构建报 "failed to build: docker daemon not found"：CI 环境未提供容器运行时，改用 jib 路线或挂 dind。

# 9. 小结

Maven 侧容器化的主线是：**用 jib 把"镜像"变成构建产物的一等公民**——无需 Dockerfile 与 daemon、依赖/资源/类三层各自缓存、tag 直接取 `${project.version}`。与 buildpacks 的分野在于定制权与运维责任的分配：要显式可控选 jib，要供应商托管 OS 层选 buildpacks。

串联到 K8s 后守住三条纪律：**版本单一来源**（POM 驱动 tag，生产按 digest 部署），**CI 全链路显式等待**（`helm --wait --atomic`、`kubectl rollout status`），**探照灯配齐**（startup/liveness/readiness 三探针 + graceful shutdown + preStop sleep）。做到这三条，从 `mvn verify` 到生产 Pod Ready 的链路就具备了可复现、可审计、可回滚的完整闭环。
