
# 1. 什么是 Blue Ocean

Blue Ocean 是 Jenkins 的现代化 UI 界面，重新设计了 Pipeline 的用户体验。它于 2017 年随 Jenkins 2.0 时代推出，旨在解决传统 Jenkins UI 在 Pipeline 可视化方面的不足。

核心目标：
- **Pipeline 可视化**：直观展示 Pipeline 的执行流程
- **简化操作**：减少配置步骤，提供更友好的交互
- **快速排错**：快速定位失败步骤，查看日志
- **Pipeline 编辑器**：可视化编辑 Pipeline

# 2. 安装 Blue Ocean

## 2.1 插件安装

Blue Ocean 是一组插件集合：

```
必需插件：
├── blueocean (主插件)
├── blueocean-web
├── blueocean-commons
├── blueocean-rest
├── blueocean-rest-impl
├── blueocean-pipeline-scm-api
├── blueocean-events
├── blueocean-jwt
├── blueocean-personalization
├── blueocean-dashboard
├── blueocean-pipeline-editor
└── blueocean-core-js

可选插件：
├── blueocean-bitbucket-pipeline
├── blueocean-github-pipeline
├── blueocean-git-pipeline
├── blueocean-config
├── blueocean-autofavorite
└── blueocean-display-url
```

```bash
# 通过 CLI 安装
java -jar jenkins-cli.jar -s http://localhost:8080 install-plugin blueocean

# 或直接安装并等待重启
curl -X POST \
  http://localhost:8080/pluginManager/installNecessaryPlugins \
  -H "Content-Type: application/xml" \
  -d '<install plugin="blueocean@latest"/>'
```

## 2.2 访问 Blue Ocean

```
访问方式：
1. 在 Jenkins 首页点击 "Open Blue Ocean" 链接
2. 直接访问：http://jenkins:8080/blue
3. 从任意 Pipeline 页面点击 "Blue Ocean" 按钮
```

Blue Ocean 可以与传统 UI 共存，用户可以根据偏好切换。

# 3. Blue Ocean 仪表盘

## 3.1 主界面

Blue Ocean 的主界面设计简洁：

```
┌──────────────────────────────────────────────────────┐
│ 🔍 Search Pipelines...                  [Filters]    │
├──────────────────────────────────────────────────────┤
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │ my-service-pipeline           ★ Run #142       │ │
│  │ [✓] [✓] [✓] [✓] [✓] [⋯]   5m 32s  main ↗    │ │
│  └────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────┐ │
│  │ user-service                  ★ Run #85        │ │
│  │ [✓] [✓] [✓] [✗] [⋯]        2m 10s  develop ↗ │ │
│  └────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────┐ │
│  │ deployment-pipeline           ★ Run #203       │ │
│  │ [✓] [✓] [⋯] [⋯] [⋯]        1m 45s  main ↗    │ │
│  └────────────────────────────────────────────────┘ │
│                                                      │
├──────────────────────────────────────────────────────┤
│ [Show More...]                                       │
└──────────────────────────────────────────────────────┘
```

界面元素说明：
- ★：分支状态（Favorited 的 Pipeline）
- Run #N：最近的一次构建
- 进度条：每个 stage 的实时状态（✓ 成功/✗ 失败/⋯ 运行中）
- 耗时：本次构建总耗时
- 分支名：当前分支
- ↗：打开在新窗口

## 3.2 过滤器

```
Filters:
├── Status: All/Success/Failure/Running/Aborted/Paused
├── Branch: All branches / 特定分支
├── Organization: 筛选 GitHub/GitLab 组织
└── Search: 按名称搜索 Pipeline
```

## 3.3 收藏功能

用户可以将常用的 Pipeline 添加收藏：

```
收藏 Pipeline：
1. 点击 Pipeline 右侧的 ★ 图标
2. 收藏的 Pipeline 会显示在仪表盘顶部
3. 取消收藏再次点击 ★ 即可
```

# 4. Pipeline 运行视图

## 4.1 单次运行详情

点击任意运行记录进入详情页：

```
┌──────────────────────────────────────────────────────┐
│ my-service-pipeline · main · #142                    │
│  Duration: 5m 32s  ·  Commit: abc1234  ·  User: dev │
├──────────────────────────────────────────────────────┤
│                                                      │
│  ┌──────────┬────────┬──────────┬────────┬─────────┐ │
│  │  Build   │  Test  │  Scan    │ Deploy │ Notify  │ │
│  │          │        │          │        │         │ │
│  │   ✓      │   ✓    │   ✗      │   -    │    -    │ │
│  │  1m 20s  │  2m 5s │  45s     │   -    │    -    │ │
│  │          │        │          │        │         │ │
│  └──────────┴────────┴──────────┴────────┴─────────┘ │
│                                                      │
│  Failed Step: Static Analysis                        │
│  ┌──────────────────────────────────────────────┐   │
│  │ [ERROR] Found 3 checkstyle violations...     │   │
│  │ [ERROR] src/main/java/App.java:42: Line too  │   │
│  │         long (expected < 120, actual 145)    │   │
│  │ [ERROR] src/main/java/Service.java:15:       │   │
│  │         Missing Javadoc                      │   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
└──────────────────────────────────────────────────────┘
```

交互特性：
- **点击 Stage**：展开该阶段的详细日志
- **悬停**：显示步骤级别的详细信息
- **日志搜索**：支持在日志中搜索关键词
- **自动滚动**：实时构建日志自动滚动

## 4.2 步骤级细节

展开 Stage 后显示步骤级别的执行情况：

```
┌─ Scan ──────────────────────────────────────────┐
│                                                  │
│  ▶ Checkstyle                          ✓  12s   │
│  ▶ PMD                                 ✓  8s    │
│  ▶ SonarQube Analysis                  ✗  25s   │
│     ┌──────────────────────────────────┐        │
│     │ Quality Gate Failed              │        │
│     │                                   │        │
│     │ New bugs: 3                       │        │
│     │ Coverage dropped: 72% → 68%      │        │
│     │ Duplication: 5.2%                │        │
│     └──────────────────────────────────┘        │
│                                                  │
└──────────────────────────────────────────────────┘
```

# 5. Pipeline 编辑器

## 5.1 可视化编辑器

Blue Ocean Pipeline Editor 提供图形化编辑 Declarative Pipeline 的功能。

```
访问方式：
1. 打开任意 Multibranch Pipeline
2. 点击右上角 "Edit Pipeline" 按钮
3. 或在 Pipeline 运行详情页点击 "Edit"
```

编辑器功能：

```
Pipeline Editor:
├── 添加 Stage
│   ├── 命名 Stage
│   ├── 添加 Step
│   │   ├── Shell 命令
│   │   ├── Checkout
│   │   ├── DDL (数据定义)
│   │   ├── Docker 操作
│   │   ├── Maven 构建
│   │   ├── Gradle 构建
│   │   └── 自定义步骤
│   └── 添加并行 Stage
├── 拖拽排序
├── 删除/克隆 Stage
└── 生成 Jenkinsfile
```

## 5.2 编辑器生成的 Jenkinsfile

```groovy
// Blue Ocean Pipeline Editor 生成的 Jenkinsfile
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh './mvnw clean compile'
            }
        }
        stage('Test') {
            steps {
                sh './mvnw test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }
        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh './deploy.sh'
            }
        }
    }
}
```

**局限性**：
- 仅生成声明式 Pipeline（不支持脚本式）
- 不支持复杂的 Groovy 逻辑
- 不支持 `matrix`、`parallel`（嵌套）等高级功能
- 适合入门级 Pipeline，复杂场景仍需手写

# 6. Git 分支可视化

Blue Ocean 将 Git 分支和 Pipeline 运行状态整合展示：

```
┌─ Branches ──────────────────────────────────────┐
│                                                   │
│  main         ●●●●●●●●⊙○●⊙●⊙●●●                 │
│                     ✓✓✓✓✓✗✓✓✗✓✗✓✓✓              │
│                                                   │
│  develop      ●●●●●●●●○○●○●○●●●                  │
│                     ✓✓✓✓✓✓✓✗✓✓✓✓✓               │
│                                                   │
│  feature/xxx  ●                                   │
│                     ✓                             │
│                                                   │
│  release/1.0  ●●●●                                │
│                     ✓✓✓✓                          │
│                                                   │
│  ● 成功  ○ 运行中  ⊙ 等待中  ✗ 失败              │
└──────────────────────────────────────────────────┘
```

分支活动视图显示：
- 每个分支上最近的构建状态
- 分支间的相对活跃度
- 构建状态的时间趋势

# 7. Pull Request 集成

Blue Ocean 与 GitHub/GitLab 的 PR/MR 紧密集成：

```
┌─ Pull Requests (3 open) ──────────────────────┐
│                                                  │
│  #142 - Fix login bug                    dev    │
│  [✓] [✓] [✓] [✓]    3m 12s     2h ago     ↗   │
│                                                  │
│  #141 - Add search feature              dev    │
│  [✓] [✓] [✗] [⋯]    1m 45s     5h ago     ↗   │
│                                                  │
│  #140 - Update deps                     dev    │
│  [✓] [✓] [✓] [✓]    4m 20s     1d ago     ↗   │
│                                                  │
└──────────────────────────────────────────────────┘
```

特性：
- 自动显示所有打开的 PR
- 显示 PR 的最新 CI 运行状态
- 点击可直接跳转到 PR 页面
- PR 状态自动同步 Jenkins Pipeline 结果到 GitHub/GitLab

# 8. 传统 UI vs Blue Ocean

| 特性 | 传统 UI | Blue Ocean |
|------|---------|------------|
| Pipeline 可视化 | Stage View 表格 | 流程图风格 |
| 日志查看 | 全文本日志 | 按步骤折叠/展开 |
| 启动速度 | 快 | 相对慢 |
| 功能完整性 | 全部功能 | 部分功能 |
| 系统管理 | 完整 | 有限 |
| Agent 管理 | 支持 | 不支持 |
| 凭据管理 | 支持 | 不支持 |
| Pipeline 编辑 | 无 | 可视化编辑器 |
| 仪表盘 | 列表视图 | 卡片视图 |
| Git 分支展示 | 列表 | 可视化时间线 |
| PR 集成 | 有限 | 优 |
| 移动端适配 | 差 | 好 |
| 主题 | 无 | 现代化 UI |

**使用建议**：
- **Blue Ocean 优先**：查看 Pipeline 运行状态、日志、PR 时使用
- **传统 UI 辅助**：系统管理、配置修改、插件管理时使用
- 两者互不冲突，可以根据场景灵活切换

# 9. 常见问题

## 9.1 Blue Ocean 加载缓慢

```yaml
# 原因分析
可能原因：
  - 大量 Pipeline 数据
  - 浏览器缓存问题
  - 网络延迟

# 解决方案
1. 清除浏览器缓存
2. 使用 Chrome DevTools 分析网络请求
3. 减少 Pipeline 数量
4. 升级 Jenkins 版本
5. 检查 Blue Ocean 和依赖的插件版本兼容性
```

## 9.2 Blue Ocean 未显示某些 Job

```
检查清单：
1. Job 是否使用了 Pipeline 类型（传统 Freestyle 不支持）
2. 多分支 Pipeline 是否被正确扫描
3. Jenkinsfile 是否存在
4. 用户是否有权限查看该 Job
5. 是否在 Blue Ocean 过滤器中隐藏了
```

## 9.3 Pipeline Editor 无法使用

```
可能原因：
1. Pipeline 不是声明式语法（脚本式不可编辑）
2. Jenkinsfile 存在语法错误
3. blueocean-pipeline-editor 插件未安装
4. 多分支 Pipeline 的特殊分支不支持

解决办法：
1. 确认使用声明式 Pipeline
2. 手动修复 Jenkinsfile 语法
3. 安装必要插件
```
