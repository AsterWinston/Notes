
# 1. GitLab 平台概述

GitLab 是一个完整的 DevOps 平台，提供 Git 仓库管理、CI/CD、安全扫描、容器注册、监控等功能。

## 1.1 GitLab 核心组件

| 组件 | 功能 | 说明 |
|------|------|------|
| GitLab Repository | 代码托管 | 支持 Git LFS、仓库镜像 |
| GitLab CI/CD | 持续集成/部署 | 内置 CI，无需额外服务 |
| GitLab Runner | 构建执行器 | 支持 Docker/K8s/Shell |
| Container Registry | 容器镜像仓库 | 集成 Docker 镜像 |
| GitLab Pages | 静态站点 | 文档/项目站点 |
| GitLab Security | 安全扫描 | SAST/DAST/容器扫描 |
| GitLab Monitor | 监控 | APM、日志、指标 |

## 1.2 GitLab 的三种部署方式

```yaml
GitLab.com (SaaS):
  优点: 无需维护
  限制: 存储/运行时长限制
  适用: 小团队/开源项目

Self-Managed (自托管):
  优点: 完全控制、无限制
  缺点: 需要维护服务器
  适用: 企业/合规需求

GitLab Dedicated:
  优点: 专有实例、合规
  缺点: 成本高
  适用: 大型企业
```

# 2. GitLab CI/CD 深度实践

## 2.1 .gitlab-ci.yml 基础

```yaml
# .gitlab-ci.yml
stages:
  - lint
  - test
  - build
  - deploy

variables:
  NODE_VERSION: "20"
  CACHE_KEY: "$CI_COMMIT_REF_SLUG"

cache:
  key: $CACHE_KEY
  paths:
    - node_modules/

before_script:
  - node --version
  - npm ci

lint:
  stage: lint
  script:
    - npm run lint
  except:
    - tags

test:
  stage: test
  script:
    - npm run test:unit
    - npm run test:integration
  coverage: '/All files[^|]*\|[^|]*\s+([\d\.]+)/'
  artifacts:
    paths:
      - coverage/
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

build:
  stage: build
  script:
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 week

deploy:
  stage: deploy
  script:
    - npm run deploy
  environment:
    name: production
    url: https://app.example.com
  only:
    - main
  when: manual
```

## 2.2 GitLab Runner 配置

```bash
# 安装 Runner
# Linux
sudo apt-get install gitlab-runner

# macOS
brew install gitlab-runner

# Windows（使用 scoop）
scoop install gitlab-runner

# 注册 Runner
sudo gitlab-runner register \
  --url https://gitlab.com \
  --registration-token REG_TOKEN \
  --executor docker \
  --description "Docker Runner" \
  --docker-image alpine:latest \
  --docker-volumes /var/run/docker.sock:/var/run/docker.sock

# Runner 类型
# shared: 所有项目共享
# group: 组级 Runner
# specific: 项目专用
```

## 2.3 高级 CI 配置

```yaml
# 矩阵测试
.test_template: &test_template
  stage: test
  script:
    - npm run test

test-node-18:
  <<: *test_template
  image: node:18

test-node-20:
  <<: *test_template
  image: node:20

test-node-22:
  <<: *test_template
  image: node:22

# 并行测试
parallel-test:
  stage: test
  parallel: 5
  script:
    - npm run test -- --shard=$CI_NODE_INDEX/$CI_NODE_TOTAL

# 子流水线
child-pipeline:
  stage: deploy
  trigger:
    include: child-pipeline.yml
    strategy: depend  # 等待子流水线完成

# 多项目流水线
multi-project:
  stage: deploy
  trigger:
    project: mygroup/dependent-service
    branch: main
    strategy: depend
```

## 2.4 环境管理

```yaml
# 环境部署配置
deploy-review:
  stage: deploy
  script:
    - deploy review-app
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    url: https://$CI_COMMIT_REF_SLUG.example.com
    on_stop: stop-review
  only:
    - merge_requests

stop-review:
  stage: deploy
  script:
    - stop review-app
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    action: stop
  when: manual
  only:
    - merge_requests

# 手动部署
deploy-production:
  stage: deploy
  script:
    - deploy production
  environment:
    name: production
    url: https://app.example.com
  when: manual  # 需要手动触发
  only:
    - main
```

# 3. Merge Request 工作流

## 3.1 MR 生命周期

```yaml
流程:
  1. 创建功能分支
  2. 推送代码
  3. 创建 Merge Request
  4. GitLab CI 自动触发测试
  5. 代码审查
  6. 检查合并冲突
  7. 合并到目标分支

合并选项:
  - Merge Commit: 标准合并（--no-ff）
  - Squash Commit: 压缩合并（squash）
  - Fast-Forward: 快进合并

MR 设置:
  - 删除源分支: true
  - Squash commits: true
  - 允许合并: 仅通过 pipeline
  - Pipelines must succeed: true
```

## 3.2 MR 模板

```yaml
# .gitlab/merge_request_templates/default.md
## 变更描述

## 关联 Issue
Closes #ISSUE_ID

## 变更类型
- [ ] feat: 新功能
- [ ] fix: Bug 修复
- [ ] refactor: 重构
- [ ] docs: 文档
- [ ] test: 测试
- [ ] chore: 其他

## 如何测试
1. 步骤一
2. 步骤二

## 检查清单
- [ ] 代码规范
- [ ] 测试覆盖
- [ ] 文档更新
- [ ] Pipeline 通过

## 截图/日志

## 备注
```

## 3.3 Code Owners 配置

```yaml
# .gitlab/CODEOWNERS
# 全局默认
* @devops-team

# 特定路径
src/auth/ @security-team
src/api/  @api-team
docs/     @docs-team

# 特定文件
Dockerfile @infra-team
.gitlab-ci.yml @ci-team
LICENSE @legal-team
```

# 4. GitLab 安全特性

## 4.1 安全扫描

```yaml
# .gitlab-ci.yml 中的安全扫描
include:
  - template: SAST.gitlab-ci.yml
  - template: DAST.gitlab-ci.yml
  - template: Secret-Detection.gitlab-ci.yml
  - template: Dependency-Scanning.gitlab-ci.yml

# SSO 扫描
sast:
  stage: security
  variables:
    SAST_EXCLUDED_PATHS: "node_modules,test,vendor"

# 密钥检测
secret_detection:
  stage: security

# 依赖扫描
dependency_scanning:
  stage: security

# 容器扫描
container_scanning:
  stage: security
  variables:
    DOCKER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```

## 4.2 合规管理

```yaml
# 合规流水线配置
compliance:
  stage: compliance
  script:
    - check-license-compliance
    - check-dependency-licenses
  artifacts:
    paths:
      - compliance-report.json

# 审计事件
# GitLab Premium 及以上版本支持
# 跟踪所有项目操作
```

# 5. GitLab 容器注册表

```yaml
# 构建和推送到 GitLab Container Registry
build-image:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:latest
  only:
    - main

# 使用预构建的镜像
deploy:
  stage: deploy
  image: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  script:
    - deploy-app
```

# 6. GitLab Pages

```yaml
pages:
  stage: deploy
  script:
    - npm ci
    - npm run build
    - mv public public-old
    - mv dist public
  artifacts:
    paths:
      - public
  only:
    - main
  environment:
    name: pages
    url: https://$CI_PROJECT_NAMESPACE.gitlab.io/$CI_PROJECT_NAME
```

# 7. GitLab Auto DevOps

Auto DevOps 提供开箱即用的 CI/CD 流水线：

```yaml
# 启用 Auto DevOps
# 项目 → Settings → CI/CD → Auto DevOps

# 或者自定义
auto-devops:
  stage: build
  variables:
    AUTO_DEVOPS_EXPLICITLY_ENABLED: "1"

# Auto DevOps 包含的阶段
# 1. Build - 构建应用
# 2. Test - 自动化测试
# 3. Code Quality - 代码质量分析
# 4. SAST - 静态安全测试
# 5. Secret Detection - 密钥检测
# 6. Dependency Scanning - 依赖扫描
# 7. Review - Review Apps
# 8. DAST - 动态安全测试
# 9. Deploy - 自动部署到 staging/production
```

# 8. GitLab API 与自动化

## 8.1 常用 API 操作

```bash
# 创建 Merge Request
curl --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  --data "source_branch=feature/login&target_branch=main&title=Add login feature" \
  "https://gitlab.com/api/v4/projects/$PROJECT_ID/merge_requests"

# 触发 Pipeline
curl --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  --data "ref=main" \
  "https://gitlab.com/api/v4/projects/$PROJECT_ID/trigger/pipeline"

# 列出 Runner
curl --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://gitlab.com/api/v4/runners"
```

## 8.2 Webhook 集成

```yaml
# 在 GitLab 项目设置中添加 Webhook
# Settings → Webhooks

事件类型:
  Push events: 推送代码
  Merge request events: MR 操作
  Pipeline events: CI 状态变化
  Tag push events: 标签推送
  Note events: 评论

常见集成:
  - Slack/钉钉/飞书通知
  - Jenkins 触发
  - Jira 同步
  - 内部部署系统
```

# 9. GitLab 与 Kubernetes 集成

```yaml
# .gitlab-ci.yml 中的 K8s 部署
deploy-k8s:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl set image deployment/app app=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - kubectl rollout status deployment/app
  environment:
    name: production
    kubernetes:
      namespace: production
  only:
    - main
```

# 10. GitLab 备份与迁移

```bash
# 自托管 GitLab 备份
sudo gitlab-backup create
sudo gitlab-backup restore BACKUP=1234567890_2024_01_15_16.7.0

# GitLab 仓库迁移
# 从 GitHub 导入
# 从其他 Git 服务导入
git clone --bare https://github.com/user/repo.git
cd repo.git
git push --mirror git@gitlab.com:user/repo.git
```
