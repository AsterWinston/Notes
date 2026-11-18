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
- **审批部署**：`when: delayed` + `start_in` 做定时；配合 protected 变量与 protected environment 控制生产密钥只对授权人可见。

## 1.1 `environment` 关键字全字段

| 字段 | 类型 | 作用 | 常见误用 |
|------|------|------|---------|
| `name` | 字符串 | 环境标识，支持 `parent/child` 层级；创建后**不能改名** | 用 `$CI_COMMIT_REF_NAME` 导致非法字符（见 2.4） |
| `url` | 字符串/变量 | 环境访问地址，显示在 MR/环境/部署三处 | 忘记 `$CI_ENVIRONMENT_SLUG`，同名环境 URL 冲突 |
| `action` | `start`/`stop`/`prepare`/`verify`/`access` | 声明这一 job 对环境做什么 | 把 stop 当普通 job 手点，环境永不回收 |
| `on_stop` | job 名 | 指定"停止该环境时运行哪个 job" | deploy 与 stop 两个 job 的 `rules` 不一致 → 停止失效 |
| `auto_stop_in` | 自然语言时长 | 到期自动停止（`1 week`、`45 minutes`）；`never` 关闭 | 以为按分钟精确执行（后台 worker 每小时扫一次） |
| `deployment_tier` | `development`/`testing`/`staging`/`production`/`other` | 环境重要级，影响保护环境的分组与展示 | 依赖名字自动推断而出错 |
| `kubernetes` | `组/agent 名` | 关联 GitLab Agent，用于 Kubernetes 看板与资源管理 | 仍在写已弃用的 GitLab-managed cluster 字段 |

环境有三种状态，取决于 on-stop job 是否跑过：`available` → `stopping` → `stopped`；只有 `stopped` 的环境才能删除。

## 1.2 一个"生产级"部署 job 长什么样

```yaml
deploy-production:
  stage: deploy
  needs: [package-image]
  environment:
    name: production
    url: https://example.com
    deployment_tier: production
    kubernetes: platform/prod-agent       # 关联 Agent，K8s 看板可看该环境的工作负载
    auto_stop_in: never
  resource_group: production              # 同环境串行，禁止并发部署互踩
  rules:
    - if: $CI_COMMIT_TAG
      when: manual                        # 只由受控 Tag + 人工确认触发
  script:
    - kubectl set image deploy/app app=$IMAGE_TAG -n prod
    - ./canary.sh --percent 10
```

四个要素缺一不可：`environment`（去哪）+ `resource_group`（串行）+ `rules`（谁能触发）+ 受保护变量（用什么凭据）。少任何一项，第 15 篇的门禁都会漏。

# 2. 环境命名与层级

```yaml
environment:
  name: production/api-server        # 斜杠表示层级：parent = production
```

- 层级命名让看板按 `production` 聚合其子环境（`production/api-server`、`production/worker`）。
- 环境名是自由字符串，但**和 CI 变量作用域、保护环境、Auto DevOps 的路由约定挂钩**，定下来别乱改。

**设计考量**：环境（业务概念）≠ 分支（代码概念）。常见映射策略：

| 策略 | 规则 | 说明 |
|------|------|------|
| 分支即环境 | main→staging，release→prod | GitFlow/主干混合团队 |
| MR 即临时环境 | 每个 MR 一个 review 环境 | 评审体验最好 |
| Tag/手动即生产 | 生产只由受控 Tag 触发 | 合规审计友好 |

## 2.1 `deployment_tier` 的自动推断规则

不显式写 `deployment_tier` 时，GitLab 按环境名正则推断——**这直接决定"哪些环境算生产"，也影响 DORA 的变更失败率统计**：

| 匹配环境名的正则（Ruby） | 推断层级 |
|--------------------------|---------|
| `/(dev|review|trunk)/i` | development |
| `/(test|tst|int|ac(ce|)pt|qa|qc|control|quality)/i` | testing |
| `/(st(a|)g|mod(e|)l|pre|demo|non)/i` | staging |
| `/(pr(o|)d|live)/i` | production |
| 不匹配任何一条 | other |

踩坑点：自定义环境名如 `core-qa`、`gateway-live`、`sandbox` 会被推断成非预期层级。团队规范里应要求**生产环境显式写 `deployment_tier: production`**，不要赌正则。

## 2.2 命名的三条纪律

```text
① 层级用斜杠分组，且只有一层父级：  review/<slug>、production/<service>
② 动态部分一律用 SLUG 变量，不用 REF_NAME：  review/$CI_COMMIT_REF_SLUG
③ 名字里不放流水线/构建号：  环境是长期对象，一次性的东西放部署历史里看
```

## 2.3 非法字符与典型报错

```plaintext
This job could not be executed because it would create an environment with an invalid parameter.
```

| 诱因 | 例子 | 修复 |
|------|------|------|
| 变量未展开，名字结尾是 `/` | `environment: production/$ENVIRONMENT` | 保证变量存在（见 3.3 的 dotenv 写法），或去掉 environment |
| 分支名含非法字符 | 分支 `bug-fix!` → `review/bug-fix!` | 改用 `$CI_COMMIT_REF_SLUG`（会自动清洗非法字符） |
| `deployment_tier` 拼错 | 写成 `prod`/`Prod` | 只能是五个枚举值之一 |

> 环境**不能重命名**。要"改名"只能：停止旧环境 → 删除 → 用新名重新部署。这也意味着环境名一旦进配置中心/告警规则，改动成本很高，第 1 天就要定下来。

# 3. Review Apps：每个 MR 一个可访问的预览

Review App 把"这个改动跑起来长什么样"提前到合并之前——为每个 MR 自动部署一套临时环境，关闭/合并后自动销毁：

```yaml
review-app:
  stage: deploy
  script:
    - ./deploy.sh review-$CI_MERGE_REQUEST_IID   # 命名按 MR 唯一
  environment:
    name: review/$CI_COMMIT_REF_SLUG              # 自动按分支归组
    url: https://$CI_ENVIRONMENT_SLUG.review.example.com
    on_stop: stop-review-app                      # 指定停止时执行的 job
  rules:
    - if: $CI_MERGE_REQUEST_ID
```

- 配合 K8s/Nginx 的动态路由（按子域名路由到对应 namespace/Ingress），`$CI_ENVIRONMENT_SLUG` 提供安全的环境标识串。
- MR 页面直接出现预览链接，产品/设计不用拉分支本地起。

> **行业实践**：微服务全栈 Review App 成本高（要拉起整条依赖链），多数团队退而求其次：**前端 + BFF 层做 Review App，后端用 staging 基线**，性价比最高。

## 3.1 deploy / stop 必须成对写

停止 job 有三条硬要求，违反任一条就会出现"MR 都合了，环境还在"：

1. deploy 与 stop job 的 `rules`（或 `only/except`）**必须一致**，否则某些流水线里根本没有 stop job 可执行；
2. stop job 必须有 `environment:name` + `action: stop`，且 `when` 写在 job 级或 rules 里（用 `rules` + `when: manual` 时要配 `allow_failure: true`，否则"必须绿"的门禁被它卡住）；
3. stop job 不要在比 deploy job **更靠后的 stage 且被 needs 阻塞**，否则前序 job 失败时它永远不可运行。

```yaml
stages: [build, deploy, cleanup]

review-app:
  stage: deploy
  script: ./deploy.sh review
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    url: https://$CI_ENVIRONMENT_SLUG.review.example.com
    on_stop: stop-review-app
    auto_stop_in: 1 week            # 即使没人管，一周后自动回收
  rules:
    - if: $CI_MERGE_REQUEST_ID

stop-review-app:
  stage: cleanup
  needs: [review-app]               # 保证它可独立于失败的前序 job 运行
  variables:
    GIT_STRATEGY: none              # 分支被删后无需检出代码（否则 job 直接失败）
  script: ./teardown.sh review-$CI_MERGE_REQUEST_IID
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    action: stop
  rules:
    - if: $CI_MERGE_REQUEST_ID
      when: manual
  allow_failure: true
```

## 3.2 环境什么时候会被自动停止

| 触发 | 条件 | 注意 |
|------|------|------|
| 源分支被删除或 MR 被合并 | 默认行为（历史遗留逻辑，即使没配 on_stop 也会尝试停止） | 官方正在讨论改为"仅显式配了 `on_stop` 才停"，别把它当稳定契约 |
| 使用 MR 流水线 | 配了 on_stop 时，合并/关闭自动触发 stop | 与 Merge Train 同用时，需"避免重复流水线"才会跑 stop |
| `auto_stop_in` 到期 | 后台 worker **每小时**扫一次 | 到点不精确；`access`/`prepare` action 会重置计时 |
| 手动清理 | Environments → **Clean up environments**（选截止日期） | 保护环境不会被清 |
| 卡 `stopping` 无法退出 | **Force stop**（UI 或 API `force=true`） | 不跑 on_stop，需自行清理残留资源 |

```bash
# 用 API 盘点"泄漏"的 review 环境（可用配额与成本治理的一部分）
glab api "projects/$P/deployments?status=success&per_page=100" \
  | jq -r '.[].environment' | grep '^review/' | sort -u | head
# 强制停一个卡住的环境（不跑 on_stop，残留资源需自行清理）
ENV_ID=$(glab api "projects/$P/environments?search=review/feat-x" | jq -r '.[0].id')
glab api --method POST "projects/$P/environments/$ENV_ID/stop" -f force=true
```

## 3.3 动态 URL：由部署工具生成时

托管平台（或 Terraform）常返回随机 URL，此时用 **dotenv 报告**把值回写给 `environment:url`：

```yaml
review:
  stage: deploy
  script:
    - URL=$(./deploy.sh --output-url)
    - echo "DYNAMIC_ENVIRONMENT_URL=$URL" >> deploy.env
  artifacts:
    reports:
      dotenv: deploy.env                # job 结束后 GitLab 解析它并展开 environment:url
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    url: $DYNAMIC_ENVIRONMENT_URL
    on_stop: stop_review
  rules:
    - if: $CI_MERGE_REQUEST_ID

stop_review:
  stage: deploy                         # 与 deploy 同 stage，避免 stop 不可运行
  script: ./teardown-environment
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    action: stop                        # 停止 job 里不要再设 url（它拿不到 dotenv）
  rules:
    - if: $CI_MERGE_REQUEST_ID
      when: manual
  allow_failure: true
```

**Route Maps**（Review Apps 的路由映射）则解决另一件事：从 diff 里的源文件直接跳到对应预览页面，前端团队评审体验提升明显。

## 3.4 域名与路由的落地约束

```text
通配 DNS：      *.review.example.com  → Ingress/ALB
泛证书：        *.review.example.com（Let's Encrypt DNS-01）
命名空间隔离：  review-<slug>，配合默认 NetworkPolicy
数据策略：      共用 staging 库（只读/影子库） 或 每环境独立 schema（成本高）
资源回收：      auto_stop_in + 每周盘点残留 namespace
```

**最常见的成本失控点**不是计算资源，而是"Review 环境连生产数据写操作"。必须用只读凭据或数据脱敏副本，否则临时环境会变成生产事故源。

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
| 多阶段金丝雀 | GitLab Auto DevOps 内置：pre→canary→prod 三段各带人工确认 | Auto DevOps 模板（现处于逐步退场状态，建议自建模板） |

回滚：Environments 页 → 历史部署 → **Rollback**（重新执行那条历史流水线的部署 job）。

## 4.1 `action` 的五个语义与用法

| action | 是否创建部署记录 | 典型用途 |
|--------|----------------|---------|
| `start`（默认） | 是 | 正常部署 |
| `stop` | 是（记录一次停止动作） | 回收临时环境，配合 `on_stop` |
| `prepare` | 否 | 部署前拿环境变量、准备资源；会重置 `auto_stop_in` 计时 |
| `access` | 否 | 需要环境作用域变量的非部署 job；同样重置停止计时 |
| `verify` | 否 | 只读校验/巡检，**不**重置停止计时 |

`prepare/verify/access` 的价值：把"需要生产凭据但不该记为一次部署"的 job（如迁移前检查、发布后 SMO 巡检）表达清楚，同时避开 *deployment freshness* 机制的干扰。

## 4.2 串行与"过期部署"防护

```yaml
# 同一环境禁止并发部署：后一个必须等前一个结束
deploy-staging:
  environment: { name: staging }
  resource_group: staging            # 更细可用 resource_group: production/api-server
```

实例/项目设置里的 **Deployment freshness** 有两个不同语义，常被混用：

| 设置 | 行为 | 适用 |
|------|------|------|
| Prevent outdated deployment jobs | 目标已过期时**拒绝**旧部署 job 运行 | 生产，宁可不发也不发错版本 |
| Skip outdated deployment jobs | 旧部署 job **跳过**（不报错） | 高频小步部署，允许中间版本不落地 |

`action: prepare/verify/access` 不产生部署记录，因此不会被 freshness 判定为"过期"——用它承载部署前后检查最稳。

## 4.3 镜像变量替换

```yaml
variables:
  IMAGE: $CI_REGISTRY_IMAGE
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
deploy:
  environment:
    name: staging
  script:
    - sed -i "s|%%IMAGE%%|$IMAGE|" k8s/deploy.yaml          # GitLab 不做替换，脚本自己填
    - export IMAGE_URL=$IMAGE IMAGE_VERSION=$CI_COMMIT_SHORT_SHA
    - envsubst < k8s/deploy.yaml.tmpl | kubectl apply -f -
```

`%%IMAGE%%`、`$CI_ENVIRONMENT_SLUG`、`$CI_ENVIRONMENT_URL`、`$CI_DEPLOY_FREEZE`（配合 `ref_protected` 的部署冻结窗口）是环境侧最常用的一组变量。

## 4.4 回滚的三种路径

| 路径 | 操作 | 前提 |
|------|------|------|
| 平台回滚 | Environments → 历史部署 → Rollback | 部署 job 可重复执行、镜像 tag 不可变且未被清理（第 10 篇豁免） |
| 流水线重放 | 找到旧流水线手动重跑 deploy job | 旧 ref/Tag 仍在；变量依赖当时上下文 |
| 代码回滚 | 新 MR revert → 正常门禁合并部署 | 有 DB 变更时必须配套迁移回滚脚本 |
| Auto Rollback（Ultimate） | 收到 critical 告警自动重部署最近一次成功版本 | 告警需带 `gitlab_environment_name`；三分钟内只回滚一次；正在部署时跳过 |

# 5. 部署审批与定时部署

```yaml
deploy-production:
  environment:
    name: production
  # 方案①：人工按钮
  when: manual

scheduled-rollout:
  environment:
    name: production
  # 方案②：延迟排队（到点前可取消，适合发布窗口）
  when: delayed
  start_in: 30 minutes

tag-deploy:
  environment:
    name: production
  # 方案③：只由 Tag 触发，进 stage 后自动跑
  rules:
    - if: $CI_COMMIT_TAG
      when: on_success
```

- **manual 按钮**是持续交付（人工闸门）的标准姿势。
- `start_in` 适合"发布窗口"式部署，到点前还能取消；再配 `start_after` 可做重复部署窗口（官方用法是 `when: delayed` + `start_in` + `start_after`）。
- 更复杂的审批流（多级审批、变更单联动）GitLab 自身弱于 Jenkins/专业 CD 工具，常见组合是 GitLab 管 CI，**CD 交给 Argo CD/Spinnaker**，GitLab 只负责产出制品和触发。

## 5.1 保护环境（Protected environments）

设置 → CI/CD → Protected environments，可以给环境加四道约束：

| 约束 | 含义 | 建议 |
|------|------|------|
| Allowed to deploy | 谁能部署（角色/用户/群组，含"仅 Approver"） | 生产只列 Deploy Lead 群组 |
| Approval rules | 需要几人审批 + 审批人范围（Premium/Ultimate） | 生产 1-2 人，跨团队变更加 SRE |
| Deployment rate | 每小时最大部署次数（Ultimate） | 抑制自动化脚本失控 |
| Protected variable scope | 变量只对该环境可见 | 见 5.2 |

```bash
# 复杂嵌套字段建议先 GET 一份现有保护环境，照它的结构 PUT/POST
glab api "projects/$P/protected_environments" | jq '.[0]'
glab api --method POST "projects/$P/protected_environments" \
  -f name=production \
  -f 'deploy_access_levels[][access_level]=40' \
  -f 'approval_rules[][name]=lead' -f 'approval_rules[][approvals_required]=2'
glab api "projects/$P/protected_environments" | jq -r '.[] | "\(.name)\tdeploy=\(.deploy_access_levels)\tapprovals=\(.approval_rules)"'
```

## 5.2 变量作用域：按环境收敛密钥

CI 变量默认对流水线所有 job 可见；一旦被攻破的测试工具混进 job，就能顺手读走生产密钥。GitLab 的做法是给变量设 **environment scope**：

| 作用域写法 | 匹配到的环境 |
|-----------|-------------|
| `*`（默认） | 全部环境 |
| `production` | 仅 production |
| `review/*` | 所有 review 子环境（前缀通配） |
| `review/feature-1` | 单个精确环境 |

```bash
glab variable set --protected --masked PROD_DB_URL "$URL"
glab api --method PUT "projects/$P/variables/PROD_DB_URL" -f environment_scope=production
```

> 注意：**不要**把环境作用域变量用在 `rules` 或 `include` 的条件判断里——流水线创建时这些变量尚未解析，会出现"条件永远不成立"的诡异现象。

## 5.3 定时与冻结窗口

| 需求 | 机制 |
|------|------|
| 周期性部署 | Schedule（Cron + 分支 + 变量）+ `$CI_PIPELINE_SOURCE == "schedule"` |
| 发布窗口 | `when: delayed` + `start_in`/`start_after` |
| 冻结变更（大促/割接） | Protected environment + Approval rules + `resource_group`，配合 CI 里的 `$CI_DEPLOY_FREEZE` 跳过部署 job |
| 依赖上游制品版本 | 跨项目流水线 + 制品变量传递（第 7 篇） |

# 6. 与外部 CD/运维工具的边界

| 方案组合 | 分工 | 适合 |
|----------|------|------|
| GitLab 全流程（本系列默认） | CI + 简单 CD（脚本/kubectl） | 中小团队、K8s 单集群 |
| GitLab CI + Argo CD | GitLab 构建推镜像改清单，Argo CD 声明式同步集群 | 云原生 GitOps（参见 Kubernetes 目录的 20_gitops 篇） |
| GitLab CI + Jenkins/Spinnaker | GitLab 托管+CI，老 CD 资产继续用 | 历史包袱重的企业 |
| GitLab CI + Ansible/Terraform | 构建产物 + 基础设施变更 | VM/混合云部署 |

> **行业实践**：**镜像/制品生产在 GitLab CI，环境同步交给声明式 CD（Argo CD）** 是近年最常见的落地形态；把"部署生产"写成 GitLab 里几百行 shell 脚本的团队，正在逐步迁移出去。

## 6.1 GitLab 与外部 CD 的三个接口点

| 接口 | 形态 | 要点 |
|------|------|------|
| 触发下游 | `trigger: project: group/app strategy: depend` | 上游失败要阻断时别用 `when: on_success` 之外的宽松条件 |
| 反向回调 | Webhook（Push/Merge request/Pipeline/Deployment events） | "Deployment events" 让外部 CD 知道 GitLab 侧部署结果 |
| 集群侧 | **GitLab Agent for Kubernetes**（pull 模式，agent 装在集群） | 比 kubeconfig 塞进变量安全；环境用 `environment:kubernetes` 关联 agent kpath |
| 云侧凭据 | OIDC（`id_tokens`）换短期云角色 | 免长期 AK/SK，见第 12 篇令牌矩阵 |

判断标准很简单：**"部署"是否需要跨环境回滚、灰度、多集群编排**。需要 → 交给声明式 CD，GitLab 只负责产出制品 + 改清单 + 记录环境；不需要 → 脚本 + `resource_group` 足够，别为了架构好看引入平台。

# 7. 部署指标与可观测联动

`environment` 页可配置 metrics（Prometheus 数据源）与错误追踪（Sentry），部署线会叠加 QPS/错误率曲线——每次发布是不是把错误率抬上去了，图上直接见分晓。Ultimate 的 **Deployments Board / Incident 关联**则把 MR、部署、告警串成一条时间线。

## 7.1 用 API 配环境指标

```bash
ENV_ID=$(glab api "projects/$P/environments?search=production" | jq -r '.[0].id')
glab api --method POST "projects/$P/environment/$ENV_ID/metrics" \
  -f metrics_url='http://prometheus.monitoring:9090' \
  -f title='Error rate' \
  -f query_gateway='sum(rate(http_requests_total{code!~"5.."}[1m]))' \
  -f query_error_rate='sum(rate(http_requests_total{code=~"5.."}[1m]))' \
  -f y_label='Requests per minute'
```

两条查询分别代表"总量"与"错误量"，部署线上叠加后能一眼看出"发布点之后错误率是否有台阶"。生产环境建议同时配 **apdex** 或 p99 延迟，只看错误率会漏掉性能退化。

## 7.2 与告警、DORA 对齐

| 联动点 | 做法 | 校验 |
|--------|------|------|
| 告警挂环境 | Alertmanager 传 `env_name`/annotation `gitlab_environment_name`，与环境名精确匹配 | 制造一条测试告警，看环境页是否出现横幅 |
| DORA 生产口径 | 只有 `deployment_tier: production` 的部署计入变更失败率 | 对比环境列表与 DORA 报告的部署数 |
| MTTR | Incident Issue 与部署时间线 `/link` | 复盘时能否 30 秒内定位"这次故障对应哪次发布" |
| 部署频率 | 每次生产部署都写 environment（不要用 `action: access` 代替真实部署） | 报告非空且与发布记录一致 |

## 7.3 环境侧排错速查

| 现象 | 根因 | 处理 |
|------|------|------|
| 环境列表看不到刚部署的环境 | job 失败/未运行，或 `environment` 被 rules 过滤掉 | 看该 job 是否真跑；`glab api projects/$P/environments` 核对 |
| 环境卡在 `stopping` | on_stop job 不可运行或 Runner 失联 | 修 job 状态或 **Force stop**（自行清理资源） |
| MR 合并后 review 环境还在 | deploy/stop 的 rules 不一致；stop 缺 `action: stop` | 按 3.1 三条硬要求核对 |
| 环境 URL 是旧值 | dotenv 未产出该变量，或 URL 非法被忽略 | job 日志确认 `deploy.env` 内容与变量名 |
| 生产部署报"权限不足" | protected environment 的 Allowed to deploy 未含执行人 | 加入允许名单，或用 Approver 角色 |
| 部署报"outdated deployment job" | 开了 Prevent outdated deployment jobs 且 ref 已落后 | 更新 MR（rebase/merge）后重跑 |
| 生产密钥取不到 | 变量非 Protected，或 environment scope 不匹配 | 设 Protected + 对应 scope（5.2） |
| DORA 部署频率为 0 | 部署 job 没写 `environment` | 补埋点（第 14 篇） |

# 8. 小结

| 要点 | 一句话 |
|------|--------|
| Environment 价值 | 把不可见的 shell 部署变成有名字、有 URL、有历史、可回滚的平台对象 |
| 命名纪律 | 层级名 + SLUG 变量 + 显式 `deployment_tier`；环境不可改名 |
| Review App | `on_stop` 成对 + rules 一致 + `GIT_STRATEGY: none` + `auto_stop_in` 兜底 |
| 发布策略 | 蓝绿/金丝雀由脚本实现，GitLab 用 `resource_group`/freshness 保证串行与防过期 |
| 审批 | manual/delayed 覆盖人工闸门；生产用 protected environment + 审批规则 + 环境作用域变量 |
| 边界 | CI 在 GitLab，CD 看团队规模：脚本→Argo CD→Spinnaker 渐进演化 |
| 联动 | 指标与告警按环境名对齐，DORA 才算得准 |

**上线自检清单**

- [ ] 每个生产部署 job 都有 `environment` + `resource_group` + 受控 `rules`
- [ ] 生产变量均为 Protected + Masked，且 environment scope 精确到环境
- [ ] review 环境有成对 stop job 与 `auto_stop_in`，并做过一次"合并即回收"验证
- [ ] `deployment_tier` 显式声明，不依赖名字正则推断
- [ ] 环境页已配错误率/延迟指标，一条测试告警能挂到正确环境
- [ ] 至少演练过一次 Rollback，确认镜像 tag 未被清理策略删除

环境与部署讲完，下一篇进入交付链路上的"货仓"——GitLab Container Registry 与包仓库。
