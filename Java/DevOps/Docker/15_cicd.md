
# 1. 目标：可重复构建、可追溯发布、可快速回滚

容器化交付的理想状态：

1. **同一份源码**在 CI 上构建出**同一份镜像结果**（尽量可重复）。
2. 每个镜像都能追溯到源码版本、构建参数、依赖来源。
3. 发布是"拉镜像并运行"，回滚是"切回旧版本镜像"。

关键点：镜像是制品，不是脚本执行结果。传统部署（scp jar + ssh 重启脚本）的问题在于"部署动作"发生在目标机器上，环境状态不可控；容器化把部署收敛为"registry 里换一个 tag/digest"，目标机器只负责拉取运行，于是 CI/CD 的全部工程问题简化为三个：**制品如何产生（build）、如何证明可信（scan/sign）、如何切换（deploy）**。

# 2. 标准流水线骨架

典型阶段：

1. Test：单元测试/集成测试
2. Build Image：构建镜像
3. Scan：漏洞扫描/策略检查
4. Push：推送到 Registry
5. Deploy：部署到环境（Compose/K8s/其他）

```text
 ┌────────┐   ┌─────────────┐   ┌──────────┐   ┌────────┐   ┌──────────┐
 │ push/  │──▶│ mvn test +  │──▶│ buildx   │──▶│ trivy  │──▶│ push +   │──▶ deploy
 │ MR     │   │ 覆盖率门禁   │   │ build    │   │ scan   │   │ sign     │    (K8s/Compose)
 └────────┘   └─────────────┘   └──────────┘   └────────┘   └──────────┘
                                  cache-from/                      高危阻断
                                  cache-to 复用
```

**设计考量**：

- **扫描必须在 push 正式仓库之前**：一旦带毒镜像进了 `main` 引用的仓库，各节点随时可能拉到；先推临时 sha tag、扫描通过后再移动正式 tag，是目前主流做法。
- **build 与 test 分阶段但共享 Maven 缓存**：单测的依赖下载与镜像构建内的依赖下载用同一缓存目录/层缓存，能省一半流水线时间。
- **deploy 步骤与流水线解耦**：小型团队可以让 CI 直接 kubectl apply，规模上来后交给 GitOps（Argo CD/Flux）——CI 只改 manifest 仓库里的 digest，CD 负责收敛，发布审批与回滚策略都留在 GitOps 侧。

阶段顺序背后有成本考量：**便宜的检查放前面**（跑单测几分钟、build 要十几分钟、扫描要拉全镜像），失败越早越省 runner 时间；扫描必须在 push 到正式仓库**之前**，否则带毒制品已经分发到各节点。

# 3. GitLab CI 完整 pipeline 示例

一份可直接套用（Java + Spring Boot + buildx）的 `.gitlab-ci.yml`：

```yaml
stages: [test, build, scan, push, deploy]

variables:
  REGISTRY: harbor.corp.com/pay
  APP: order-service
  IMAGE: $REGISTRY/$APP:$CI_COMMIT_SHORT_SHA

test:unit:
  stage: test
  image: maven:3.9-eclipse-temurin-17
  script:
    - mvn -B -Dmaven.repo.local=$CI_PROJECT_DIR/.m2 verify
  cache:
    key: maven-$CI_COMMIT_REF_SLUG
    paths: [.m2]

build:
  stage: build
  image: docker:24
  services: [docker:24-dind]
  variables:
    DOCKER_BUILDKIT: "1"
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - echo "$HARBOR_CI_PASSWORD" | docker login $REGISTRY -u ci-robot --password-stdin
  script:
    - docker buildx create --use --name ci --driver docker-container || true
    # --push 直接推 sha tag: 后续 job 在独立容器里跑, 本地镜像不共享, 必须走 registry
    - >
      docker buildx build --push
      --cache-from type=registry,ref=$REGISTRY/$APP:buildcache
      --cache-to type=registry,ref=$REGISTRY/$APP:buildcache,mode=max
      -t $IMAGE .

scan:
  stage: scan
  image: aquasec/trivy:latest
  variables:
    TRIVY_USERNAME: $HARBOR_CI_USER
    TRIVY_PASSWORD: $HARBOR_CI_PASSWORD
  script:
    - trivy image --severity HIGH,CRITICAL --exit-code 1 --ignore-unfixed $IMAGE

push:release-tag:
  stage: push
  image: docker:24
  services: [docker:24-dind]
  variables: { DOCKER_TLS_CERTDIR: "/certs" }
  rules: [ { if: '$CI_COMMIT_BRANCH == "main"' } ]
  before_script:
    - echo "$HARBOR_CI_PASSWORD" | docker login $REGISTRY -u ci-robot --password-stdin
  script:
    - docker pull $IMAGE
    - docker tag $IMAGE $REGISTRY/$APP:1.2.3-$CI_COMMIT_SHORT_SHA
    - docker push $REGISTRY/$APP:1.2.3-$CI_COMMIT_SHORT_SHA
    - cosign sign --key $COSIGN_KEY $REGISTRY/$APP:1.2.3-$CI_COMMIT_SHORT_SHA

deploy:staging:
  stage: deploy
  environment:
    name: staging
  script:
    - kubectl -n pay set image deploy/order-service
        app=$REGISTRY/$APP:1.2.3-$CI_COMMIT_SHORT_SHA
    - kubectl -n pay rollout status deploy/order-service --timeout=180s
```

# 4. Jenkins pipeline 骨架（声明式）

Jenkins 侧等价流程（`Jenkinsfile`），要点是构建机要么装 docker、要么挂 socket（安全权衡见 `13_security.md`）、要么走 dind/Kaniko：

```groovy
pipeline {
  agent any
  environment {
    REGISTRY = 'harbor.corp.com/pay'
  }
  stages {
    stage('Checkout') { steps {
      checkout scm
      script { env.IMAGE = "${env.REGISTRY}/order-service:${env.GIT_COMMIT.take(7)}" }
    }}
    stage('Test')  { steps { sh 'mvn -B verify' } }
    stage('Build') { steps {
      sh """
        docker buildx create --use --builder ci || docker buildx create --use --name ci
        docker buildx build --load \\
          --cache-from type=registry,ref=${REGISTRY}/order-service:buildcache \\
          --cache-to type=registry,ref=${REGISTRY}/order-service:buildcache,mode=max \\
          -t ${IMAGE} .
      """
    }}
    stage('Scan')  { steps { sh "trivy image --severity HIGH,CRITICAL --exit-code 1 --ignore-unfixed ${IMAGE}" } }
    stage('Push')  { steps { withCredentials([usernamePassword(credentialsId: 'harbor-ci',
                            usernameVariable: 'U', passwordVariable: 'P')]) {
      sh 'echo $P | docker login $REGISTRY -u $U --password-stdin && docker push ${IMAGE}'
    }}}
    stage('Deploy'){ steps { sh 'kubectl -n pay set image deploy/order-service app=${IMAGE} && kubectl -n pay rollout status deploy/order-service' } }
  }
}
```

> **行业实践**：GitLab CI 的 dind + registry cache、Jenkins 的"构建机常驻 docker + 缓存目录"是两种生态下的标准姿势；无论哪个平台，**CI 凭证用短效机器人账号（Harbor robot account 可设过期与只 push 指定项目），不要用个人 token**。

# 5. 版本策略：tag 规则与不可变制品

推荐 tag 策略（至少两类）：

1. 可读版本：`1.2.3`
2. 可追溯版本：`1.2.3-<git-sha>` 或单独 `sha-<git-sha>`

示例：

```bash
APP=my-app
REG=my-registry.example.com/team
VERSION=1.2.3
GIT_SHA=$(git rev-parse --short HEAD)

docker build -t $REG/$APP:$VERSION -t $REG/$APP:$VERSION-$GIT_SHA .
docker push $REG/$APP:$VERSION
docker push $REG/$APP:$VERSION-$GIT_SHA
```

**latest 与业务版本 tag 必须分离对待**：一条常见纪律是——`<git-sha>` tag 永远不可变（对应一次唯一构建），`latest`/环境浮动标签（如 `staging-latest`）只允许指向已验证版本，由发布流程最后一步统一移动。更严格的发布：部署用 digest 锁定（避免 tag 漂移），回滚 = `set image` 回上一个 digest。

# 6. BuildKit：缓存策略与密钥挂载

## 6.1 缓存为什么在 CI 里难做

CI runner 是无状态的：容器跑完销毁，本地构建缓存随之消失。BuildKit 提供的解法是把缓存**外置成 registry 里的一个镜像**：

```bash
# 导入: 从远端缓存镜像拉层作为 --cache-from
# 导出: 把全部中间层(含依赖下载层)推入缓存镜像, mode=max 比默认的仅最终层命中率高
docker buildx build \
  --cache-from type=registry,ref=harbor.corp.com/pay/order-service:buildcache \
  --cache-to   type=registry,ref=harbor.corp.com/pay/order-service:buildcache,mode=max \
  -t $IMAGE .
```

本地开发可用 `type=local,src=/tmp/dep` 或 docker 驱动自带的本地层缓存；`type=gha`（GitHub Actions）把缓存写进 Actions cache。Dockerfile 侧要为缓存命中设计：先 COPY `pom.xml` 再 `mvn dependency:go-offline`，源码层放最后（见 `8_dockerfile_best.md`）——CI 缓存复用率高低，本质取决于 Dockerfile 分层与构建顺序。

## 6.2 并行、secret 与 provenance

BuildKit 在同一层内没有依赖的 RUN 可并行，配合 `docker buildx bake` 可一次构建 compose 定义的多服务子集。构建期需要的凭证（私有 Maven 仓库、npm token）用 `RUN --mount=type=secret` 注入，**不留层**（详见 `13_security.md` 第 7 节）。多架构与供应链：`buildx build --platform linux/amd64,linux/arm64 --push --attest type=provenance,mode=min` 一次产出多架构 manifest 并附构建证明。

CI 常见反模式：

1. `ARG/ENV` 写入密钥并在构建层留下痕迹——`docker history` 可见 ARG 值。
2. `COPY` 把证书/密钥文件塞进镜像。

原则：密钥在运行时注入（平台能力、挂载、secret 管理）；构建阶段凭证优先 BuildKit secret 机制。

# 7. Java 镜像瘦身：multi-stage 模板

CI 里的 build 阶段最终产物应当直接对接瘦身后的运行镜像——Maven+JDK 的 builder 可达 800MB+，运行层只要 JRE：

```dockerfile
# syntax=docker/dockerfile:1.6
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /src
COPY pom.xml .
RUN --mount=type=cache,target=/root/.m2 mvn -B dependency:go-offline
COPY src/ ./src/
RUN --mount=type=cache,target=/root/.m2 mvn -B package -DskipTests
# 解压 fat jar: 分层 jar 的每层可被独立缓存, 且启动无需再解析 zip
RUN cd target && java -Djarmode=layertools -jar *.jar extract --destination extracted

FROM eclipse-temurin:17-jre-alpine AS runtime
RUN addgroup -S app && adduser -S app -G app
WORKDIR /app
COPY --from=build --chown=app:app /src/target/extracted/dependencies/ ./
COPY --from=build --chown=app:app /src/target/extracted/spring-boot-loader/ ./
COPY --from=build --chown=app:app /src/target/extracted/snapshot-dependencies/ ./
COPY --from=build --chown=app:app /src/target/extracted/application/ ./
USER app
EXPOSE 8080
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

`layertools extract` 按"依赖→loader→应用代码"拆层后，**日常发版只有 application 层变化**，registry 与节点上的 dependencies 层（上百 MB）全部复用，push 与拉取都接近秒级——这是 Java 镜像 CI 提速收益最大的一招。

# 8. 发布串联：K8s 滚动发布与验证

CI 产出 digest，CD 消费 digest。典型串联（Argo Rollouts 或原生 Deployment）：

```bash
# CI 最后一步: 把 digest 写进 values, 由 CD 工具应用
DIGEST=$(docker buildx imagetools inspect $IMAGE --format '{{json .Manifest}}' | jq -r .digest)
echo "$DIGEST" > digest.txt

# K8s 滚动发布: 新 Pod 就绪(readiness+startup probe)才逐个杀旧 Pod
kubectl -n pay set image deploy/order-service app=$REGISTRY/$APP@${DIGEST#sha256:}
kubectl -n pay rollout status deploy/order-service --timeout=300s || \
  { kubectl -n pay rollout undo deploy/order-service; exit 1; }
```

**发布纪律**：流水线里部署步骤必须"等结果"（rollout status/金丝雀指标分析），失败自动 undo；测试环境用浮动 tag 无妨，预发/生产一律 digest 部署；发布系统记录"环境→digest"映射让回滚可审计。与 `17_orchestration.md`、`18_best_practices.md` 的发布检查项衔接。

# 9. 常见问题与防线

## 9.1 镜像不可重复构建

常见原因：

1. 构建中下载"浮动版本"（例如 `latest` 或未锁定依赖）。
2. Dockerfile 里引入时间戳、随机数等。

防线：

1. 基础镜像固定到具体版本（甚至 digest）。
2. 依赖版本锁定（Maven/Gradle lockfile 等）。
3. 追求字节级可重复时再上 reproducible-builds 实践（SOURCE_DATE_EPOCH、固定构建路径），一般团队做到"输入锁定"即可。

## 9.2 构建慢

常见原因：

1. Dockerfile 层级导致缓存无法命中（先 COPY 全量源码）。
2. CI 未复用缓存（每轮从零下依赖）。

防线：

1. 调整 COPY 顺序、Spring Boot 用 layertools 分层。
2. 启用 BuildKit registry/local 缓存导入/导出。

## 9.3 CI 里"docker: Cannot connect to the Docker daemon"

GitLab 需 `services: [docker:dind]` 与 TLS 证书目录配置；Jenkins 需给 runner 用户加 docker 组；K8s 内 Jenkins agent 优先考虑 Kaniko/BuildKit 远程构建器而不是挂 docker.sock。

> **注意**：dind 模式下 build 与后续 deploy job 的本地镜像不互通，每个 job 的 image 里也没有 docker CLI 的登录态——凭证与登录要在**每个需要 docker 的 job 的 before_script 里重复执行**，这是 GitLab CI 新手最常见的"上一个 job 明明 push 成功了、下一个 job pull access denied"的根因。

# 10. 面试要点

| 问题 | 答题关键词 |
|------|------------|
| 为什么镜像是制品 | 可复现、可审计、可回滚 |
| tag 策略怎么设计 | SemVer + Git SHA；latest 单独治理；关键环境用 digest |
| 怎么加速 CI 构建 | BuildKit 缓存（cache-from/to registry, mode=max）、Dockerfile 分层、layertools |
| 密钥怎么处理 | BuildKit --mount=type=secret 不留层；运行时注入 |
| 流水线扫描放哪 | build 之后 push 之前, 高危阻断(--exit-code) |
| 发布如何串联 | CI 产 digest → CD set image → rollout status 失败自动 undo |

# 11. 小结

容器化 CI/CD 的主线是"**制品化**"：流水线从"在目标机上执行部署脚本"变成"产出一个不可变 digest 并切换它的引用"。骨架五段——test、build、scan、push、deploy，顺序体现"便宜检查前置、扫描先于分发"的成本与安全逻辑。BuildKit 带来三项能力：跨 CI 运行器的 registry 缓存（`--cache-from/--cache-to` + Dockerfile 分层设计 + Spring Boot layertools 是 Java 提速组合拳）、`--mount=type=secret` 的构建期凭证（不留层）、provenance/SBOM attest（供应链证明）。tag 纪律是"sha tag 不可变、浮动标签由发布流程最后移动、生产按 digest 部署"，发布串联则靠 `set image + rollout status + 失败 undo` 闭环。所有这些纪律值得在上线前固化成 checklist——这就是下一篇最佳实践（`18_best_practices.md`）的定位；日志与监控（`16_logging_monitoring.md`）回答"上线后怎么看见它"。
