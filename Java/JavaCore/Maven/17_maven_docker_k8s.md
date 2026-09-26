
# 1. jib-maven-plugin（推荐）

- Google Jib：无需 Dockerfile、无需 Docker daemon，直接构建镜像推送到 Registry。

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
				<jvmFlag>-Xmx512m</jvmFlag>
			</jvmFlags>
			<mainClass>com.example.Application</mainClass>
		</container>
	</configuration>
</plugin>
```

执行：
```bash
# 构建镜像并推送到 Registry
mvn jib:build

# 构建到本地 Docker daemon（不推送）
mvn jib:dockerBuild

# 构建 tar 文件
mvn jib:buildTar
```

- 优点：
	- 无需 Dockerfile。
	- 分层缓存优化（依赖层、资源层、类层分离）。
	- 无需安装 Docker（直接推送到 Registry）。

# 2. dockerfile-maven-plugin（Spotify）

```xml
<plugin>
	<groupId>com.spotify</groupId>
	<artifactId>dockerfile-maven-plugin</artifactId>
	<version>1.4.13</version>
	<configuration>
		<repository>registry.company.com/${project.artifactId}</repository>
		<tag>${project.version}</tag>
		<buildArgs>
			<JAR_FILE>target/${project.build.finalName}.jar</JAR_FILE>
		</buildArgs>
	</configuration>
</plugin>
```

`Dockerfile` 示例：
```dockerfile
FROM eclipse-temurin:21-jre
ARG JAR_FILE
COPY ${JAR_FILE} /app/app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

执行：
```bash
mvn dockerfile:build
mvn dockerfile:push
```

# 3. spring-boot-maven-plugin 构建镜像

Spring Boot 2.3+ 内置镜像构建支持：

```xml
<plugin>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-maven-plugin</artifactId>
	<configuration>
		<image>
			<name>registry.company.com/${project.artifactId}:${project.version}</name>
			<env>
				<BP_JVM_VERSION>21</BP_JVM_VERSION>
			</env>
		</image>
	</configuration>
</plugin>
```

```bash
mvn spring-boot:build-image
```

- 使用 Cloud Native Buildpacks 构建，无需 Dockerfile。
- 生成的镜像包含 JVM 和依赖的全套优化。

# 4. K8s 部署流程

## 4.1 CI 中的完整流程

```yaml
# GitHub Actions 示例
- name: Build & Push Docker Image
  run: mvn jib:build -Dimage.tag=${{ github.sha }}

- name: Update K8s Deployment
  run: |
    kubectl set image deployment/order-service \
      app=registry.company.com/order-service:${{ github.sha }}
    kubectl rollout status deployment/order-service
```

## 4.2 Helm 部署

```yaml
# values.yaml
image:
  repository: registry.company.com/order-service
  tag: 1.0.0
  pullPolicy: Always
```

```bash
# CI 中更新镜像版本
mvn jib:build
helm upgrade order-service ./helm-chart \
  --set image.tag=${CI_COMMIT_TAG}
```

# 5. K8s 健康检查配合

```yaml
# deployment.yaml
spec:
  containers:
    - name: order-service
      image: registry.company.com/order-service:1.0.0
      ports:
        - containerPort: 8080
      livenessProbe:
        httpGet:
          path: /actuator/health/liveness
          port: 8080
        initialDelaySeconds: 30
        periodSeconds: 10
      readinessProbe:
        httpGet:
          path: /actuator/health/readiness
          port: 8080
        initialDelaySeconds: 10
        periodSeconds: 5
```

# 6. 版本标签策略

| 构建类型 | 镜像 Tag | 说明 |
|----------|---------|------|
| 本地开发 | `latest` / `dev` | 快速迭代 |
| PR 构建 | `pr-{number}` | 临时验证 |
| 开发分支 | `develop-{sha}` | 功能测试 |
| 正式发布 | `{version}` (如 `1.0.0`) | 生产部署 |
| 回滚 | `{version}-backup` | 紧急回退 |

# 7. 常见问题排查清单

- Jib 推送失败：检查 Registry 地址和凭据，确认网络可达。
- 镜像构建慢：检查分层配置，依赖层应尽量缓存。
- K8s 拉取镜像失败：检查 imagePullSecrets 配置。
- Spring Boot 打包镜像过大：使用 JRE 基础镜像替代 JDK，开启分层优化。
