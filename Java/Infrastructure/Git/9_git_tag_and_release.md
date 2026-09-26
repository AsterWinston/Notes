
# 1. 标签的本质

Git 标签是一个指向特定 commit 的**不可变引用**。与分支不同，标签创建后不会随着新的提交而移动。

```
分支：main → A → B → C → D → E (持续前进)
标签：        ↑ v1.0.0     ↑ v1.1.0 (固定不变)
```

## 1.1 标签的存储

```bash
# 轻量标签：直接存储 commit hash
cat .git/refs/tags/v1.0.0
# abc123def456...

# 附注标签：是一个 Git 对象
git cat-file -p v1.0.0
# object abc123def456...
# type commit
# tag v1.0.0
# tagger John <john@example.com> 1712345678 +0800
# Release v1.0.0
```

# 2. 标签操作详解

## 2.1 创建标签

```bash
# 轻量标签（只指向 commit，无额外信息）
git tag v1.0.0
git tag v1.0.0 abc123  # 基于特定 commit

# 附注标签（完整对象，包含元数据）
git tag -a v1.0.0 -m "Release v1.0.0"
git tag -a v1.0.0 -m "Release v1.0.0" abc123

# 带 GPG 签名的标签
git tag -s v1.0.0 -m "Release v1.0.0"     # 用默认 GPG 密钥签名
git tag -u ABCDEF1234567890 v1.0.0 -m "..." # 用指定密钥签名

# 查看 GPG 签名
git tag -v v1.0.0
```

## 2.2 查看标签

```bash
# 列出所有标签
git tag
git tag -l "v1.*"        # 匹配模式
git tag --sort=-version:refname  # 按版本排序

# 查看标签详情
git show v1.0.0          # 查看标签指向的提交差异
git log v1.0.0           # 查看标签之后的提交
git describe             # 显示最近的 tag 名称

# 检查标签是否验证通过
git tag -v v1.0.0
# object abc123...
# type commit
# tag v1.0.0
# gpg: Signature made ...
# gpg: Good signature from "John <john@example.com>"
```

## 2.3 推送与拉取标签

```bash
# 推送标签到远程
git push origin v1.0.0           # 推送单个标签
git push origin --tags           # 推送所有标签（不推荐，可能推送未准备好的标签）
git push origin refs/tags/*:refs/tags/*  # 推送所有标签（等价）

# 只推送附注标签（忽略轻量标签）
git push origin --follow-tags    # 只推送附注标签

# 拉取标签
git fetch origin --tags          # 拉取所有标签
git fetch origin v1.0.0          # 拉取特定标签
git fetch origin 'refs/tags/*:refs/tags/*'  # 显式同步所有标签
```

## 2.4 删除标签

```bash
# 删除本地标签
git tag -d v1.0.0
git tag --delete v1.0.0

# 删除远程标签
git push --delete origin v1.0.0
git push origin :refs/tags/v1.0.0  # 等价方式

# 批量删除匹配的标签
git tag -d $(git tag -l "v1.0.*")          # 删除本地匹配的标签
git push origin --delete $(git tag -l)      # 删除所有远程标签（危险！）
```

## 2.5 重命名标签

Git 不支持直接重命名标签，需要重新创建：

```bash
# 重命名标签的步骤
git tag v1.0.1 v1.0.0      # 基于旧标签创建新标签
git tag -d v1.0.0           # 删除旧标签
git push origin v1.0.1      # 推送新标签
git push --delete origin v1.0.0  # 删除远程旧标签
```

# 3. 语义化版本控制

## 3.1 SemVer 规范

语义化版本遵循 `MAJOR.MINOR.PATCH` 格式：

```
v1.4.2
│ │ │
│ │ └── PATCH：向后兼容的 bug 修复
│ └──── MINOR：向后兼容的新功能
└────── MAJOR：不兼容的 API 变更
```

**扩展格式**：

```
v1.4.2-alpha.1          # 内部测试版
v1.4.2-beta.2           # 公开测试版
v1.4.2-rc.3             # 候选发布版
v1.4.2+build.20240115   # 构建元数据
v2.0.0-alpha.1+build.1  # 组合
```

## 3.2 版本号的管理规则

```bash
# 何时增加 MAJOR：不兼容的 API 变更
# 1.0.0 → 2.0.0（重构 API，删除公共方法）

# 何时增加 MINOR：向后兼容的功能添加
# 1.0.0 → 1.1.0（添加新接口，不破坏旧的）

# 何时增加 PATCH：向后兼容的 bug 修复
# 1.0.0 → 1.0.1（修复了一个 bug，API 不变）

# 初始开发阶段
# 0.1.0, 0.2.0, ...（MAJOR=0 表示初始开发）
```

# 4. 发布管理流程

## 4.1 常规发布流程

```bash
# 1. 从 develop 创建 release 分支
git checkout -b release/v1.2.0 develop

# 2. 在 release 分支上只做 bug 修复和文档更新
git commit -m "fix: correct typo in docs"
git commit -m "chore: update version number to 1.2.0"

# 3. 创建标签
git tag -a v1.2.0 -m "Release v1.2.0: user management feature"

# 4. 合并到 main 和 develop
git checkout main
git merge --no-ff release/v1.2.0
git tag -a v1.2.0 -m "Release v1.2.0"  # 或在 main 上重新打标签

git checkout develop
git merge --no-ff release/v1.2.0

# 5. 删除 release 分支
git branch -d release/v1.2.0

# 6. 推送
git push origin main --tags
git push origin develop
```

## 4.2 Hotfix 发布流程

```bash
# 1. 从 main 创建 hotfix 分支
git checkout -b hotfix/1.2.1 main

# 2. 修复 bug
git commit -m "fix: critical security vulnerability in auth"

# 3. 打标签
git tag -a v1.2.1 -m "Hotfix v1.2.1: security fix"

# 4. 合并回 main
git checkout main
git merge --no-ff hotfix/1.2.1
git tag -a v1.2.1 -m "Hotfix v1.2.1: security fix"

# 5. 也合并到 develop
git checkout develop
git merge --no-ff hotfix/1.2.1

# 6. 清理
git branch -d hotfix/1.2.1
git push origin main --tags
git push origin develop
```

## 4.3 自动生成 CHANGELOG

```bash
# 使用 git log 生成 changelog
git log v1.1.0..v1.2.0 --oneline --no-decorate

# 格式化的 changelog
git log v1.1.0..v1.2.0 --pretty=format:"- %s (%h)" --no-merges

# 按类型分组
echo "## Features"
git log v1.1.0..v1.2.0 --oneline --grep="^feat" --no-merges

echo "## Bug Fixes"
git log v1.1.0..v1.2.0 --oneline --grep="^fix" --no-merges

echo "## Refactoring"
git log v1.1.0..v1.2.0 --oneline --grep="^refactor" --no-merges
```

**使用 standard-version 自动生成**：

```bash
# 安装
npm install -D standard-version

# package.json
{
  "scripts": {
    "release": "standard-version",
    "release:minor": "standard-version --release-as minor",
    "release:patch": "standard-version --release-as patch",
    "release:major": "standard-version --release-as major"
  }
}

# 使用
npm run release         # 根据提交自动推断版本
npm run release:patch   # 强制发布 patch 版本

# CHANGELOG.md 会自动生成
# 同时创建 tag 并更新版本号
```

# 5. 标签的最佳实践

## 5.1 标签命名规范

```bash
# 推荐格式
v1.0.0              # 标准版本
v1.0.0-alpha.1      # Alpha 版本
v1.0.0-beta.1       # Beta 版本
v1.0.0-rc.1         # 候选发布版
v1.0.0-patch.1      # 补丁版本

# 不推荐
v1                  # 不精确
1.0.0               # 缺少 v 前缀
release-1.0.0       # 不一致
```

## 5.2 标签 vs 分支

```bash
# 标签：不可变，指向特定版本
git tag v1.0.0 abc123

# 分支：可变，持续演进
git branch feature/new-feature abc123

# 在标签上创建分支（用于 hotfix）
git checkout -b hotfix/1.0.1 v1.0.0
```

## 5.3 签名的使用场景

```bash
# 配置 GPG 密钥
gpg --full-generate-key
git config --global user.signingkey ABCDEF1234567890
git config --global commit.gpgsign true  # 自动签名所有提交

# 使用签名标签进行发布
git tag -s v1.0.0 -m "Release v1.0.0"

# 验证发布包的人身份
# 用户验证标签签名
git tag -v v1.0.0
# gpg: Good signature from "John <john@example.com>"
```

# 6. 跨仓库标签同步

```bash
# 在多个远程仓库间同步标签
git remote add github git@github.com:user/repo.git
git remote add gitlab git@gitlab.com:user/repo.git

# 推送标签到所有远程
git push github --tags
git push gitlab --tags

# 从某个远程同步标签到另一个
git fetch github --tags
git push gitlab --tags
```

# 7. 标签与 CI/CD 集成

## 7.1 GitHub Actions 自动发布

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'  # 匹配所有 v 开头的标签

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build
        run: npm run build
      - name: Create Release
        uses: softprops/action-gh-release@v1
        with:
          generate_release_notes: true
          files: |
            dist/*.zip
            dist/*.tar.gz
```

## 7.2 GitLab CI 自动发布

```yaml
# .gitlab-ci.yml
release:
  stage: deploy
  only:
    - tags
  script:
    - npm run build
    - echo "Creating release for $CI_COMMIT_TAG"
  artifacts:
    paths:
      - dist/
```
