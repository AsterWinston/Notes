# 1. Environment：把"部署到了哪"变成平台概念

没有 Environment 时，"部署"只是某个 job 的 `script` 里一串 kubectl/ssh 命令，GitLab 不知道也不关心部署去了哪、现在跑的是哪个版本。声明 `environment:` 之后：

```yaml
deploy-staging:
  stage: deploy
  script: ./deploy.sh staging
  environment:
    name: staging                       # 环境名：Ops → Environments 里出现这个环境
    url: https://staging.example.com    # 部署后显示"访问"链接
```

价值：

- **环境视图**：Ops → Environments 页按环境聚合"当前部署的 commit/MR/流水线"，谁在什么时候把什么发到了 staging 一目了然。
- **停止环境**：`environment.action: stop` 显式下线（如 review app 合并后销毁）。
- **部署看板与历史**：每个环境保留全部部署记录，可看"上一个版本"直接发起回滚。
- **审批部署**：`when: delayed` + `start_after` 做定时；配合 protected 变量控制生产密钥只对生产环境分支可见。

# 2. 环境命名与层级

```yaml
environment:
  name: production/api-server        # 斜杠表示层级：parent = production
```

- 层级命名让看板按 `production` 聚合其子环境（`production/api-server`、`production/worker`）。
- 环境名是自由字符串，但**和 CI 变量保护、审批、Auto DevOps 的路由约定挂钩**，定下来别乱改。

**设计考量**：环境（业务概念）≠ 分支（代码概念）。常见映射策略：

| 策略 | 规则 | 说明 |
|------|------|------|
| 分支即环境 | main→staging，release→prod | GitFlow/主干混合团队 |
| MR 即临时环境 | 每个 MR 一个 review 环境 | 评审体验最好 |
| Tag/手动即生产 | 生产只由受控 Tag 触发 | 合规审计友好 |

# 3. Review Apps：每个 MR 一个可访问的预览

Review App 把"这个改动跑起来长什么样"提前到合并之前——为每个 MR 自动部署一套临时环境，关闭/合并后自动销毁：

```yaml
review-app:
  stage: deploy
  script:
    - ./deploy.sh review-$CI_MERGE_REQUEST_IID   # 命名按 MR 唯一
  environment:
    name: review/$CI_COMMIT_REF_SLUG              # 自动按分支归组
    url: https://$CI_COMMIT_REF_SLUG.review.example.com
    auto_stop_if: merged                          # 合并后自动 stop
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

- 配合 K8s/Nginx 的动态路由（按子域名路由到对应 namespace/Ingress），`$CI_ENVIRONMENT_SLUG` 提供安全的环境标识串。
- MR 页面直接出现预览链接，产品/设计不用拉分支本地起。

> **行业实践**：微服务全栈 Review App 成本高（要拉起整条依赖链），多数团队退而求其次：**前端 + BFF 层做 Review App，后端用 staging 基线**，性价比最高。

# 4. 部署策略与渐进发布

`environment` 支持 `action: start/stop/prepare/verify/access` 描述部署动作，`deployment_tier` 标注环境重要级（策略真正的执行仍靠你的脚本/工具）：

```yaml
deploy-prod-canary:
  environment:
    name: production
    action: start
    deployment_tier: production
  script: ./deploy.sh --strategy canary --percent 10   # 策略语义由脚本/CD 工具实现
```

| 策略 | 语义 | 典型实现 |
|------|------|---------|
| 滚动 | 逐实例替换 | K8s 默认 rollingUpdate |
| 蓝绿 | 两套环境瞬间切流 | 双 Deployment + Ingress 切换 |
| 金丝雀 | 按比例逐步放量 | Argo Rollouts/Flagger + Ingress mesh |
| 多阶段金丝雀 | GitLab Auto DevOps 内置：pre→canary→prod 三段各带人工确认 | Auto DevOps 模板 |

回滚：Environments 页 → 历史部署 → **Rollback**（重新执行那条历史流水线的部署 job）。

# 5. 部署审批与定时部署

```yaml
deploy-production:
  environment:
    name: production
  when: manual                  # 方案1：人工按钮
  # --- 或 ---
  start_in: 30 minutes          # 方案2：延迟排队（带计时取消窗口）
  when: delayed
  # --- 或 ---
  rules:
    - if: $CI_COMMIT_TAG
      when: on_success
```

- **manual 按钮**是持续交付（人工闸门）的标准姿势。
- `start_in` 适合"发布窗口"式部署，到点前还能取消。
- 更复杂的审批流（多级审批、变更单联动）GitLab 自身弱于 Jenkins/专业 CD 工具，常见组合是 GitLab 管 CI，**CD 交给 Argo CD/Spinnaker**，GitLab 只负责产出制品和触发。

# 6. 与外部 CD/运维工具的边界

| 方案组合 | 分工 | 适合 |
|----------|------|------|
| GitLab 全流程（本系列默认） | CI + 简单 CD（脚本/kubectl） | 中小团队、K8s 单集群 |
| GitLab CI + Argo CD | GitLab 构建推镜像改清单，Argo CD 声明式同步集群 | 云原生 GitOps（参见 Kubernetes 目录的 20_gitops 篇） |
| GitLab CI + Jenkins/Spinnaker | GitLab 托管+CI，老 CD 资产继续用 | 历史包袱重的企业 |
| GitLab CI + Ansible/Terraform | 构建产物 + 基础设施变更 | VM/混合云部署 |

> **行业实践**：**镜像/制品生产在 GitLab CI，环境同步交给声明式 CD（Argo CD）** 是 2024 年后最常见的落地形态；把"部署生产"写成 GitLab 里几百行 shell 脚本的团队，正在逐步迁移出去。

# 7. 部署指标与可观测联动

`environment` 页可配置 metrics（Prometheus 数据源）与错误追踪（Sentry），部署线会叠加 QPS/错误率曲线——每次发布是不是把错误率抬上去了，图上直接见分晓。Ultimate 的 **Deployments Board / Incident 关联**则把 MR、部署、告警串成一条时间线。

# 8. 小结

| 要点 | 一句话 |
|------|--------|
| Environment 价值 | 把不可见的 shell 部署变成有名字、有 URL、有历史、可回滚的平台对象 |
| Review App | MR 级临时环境，`auto_stop_if: merged` 自动回收 |
| 发布策略 | 蓝绿/金丝雀由脚本实现，GitLab 记录语义并提供回滚入口 |
| 审批 | manual/delayed 覆盖"人工闸门"，复杂审批让位给专业 CD |
| 边界 | CI 在 GitLab，CD 看团队规模：脚本→Argo CD→Spinnaker 渐进演化 |

环境与部署讲完，下一篇进入交付链路上的"货仓"——GitLab Container Registry 与包仓库。
