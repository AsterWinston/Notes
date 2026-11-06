一个 Node 项目从"能跑"到"可维护"，包管理是绕不过去的一关。`package.json` 写错版本范围会把队友的构建搞崩，锁文件没提交会让部署结果和本地不一致，依赖装了一堆却不知道谁引入了漏洞。本篇把 npm 与 pnpm 的核心机制讲清楚。

# 1. 包管理器解决什么问题

## 1.1 依赖的三个层次

一个项目依赖的包，往往又依赖别的包，形成一棵依赖树：

```text
my-app
├── express@4.19.2
│   ├── body-parser@1.20.2
│   ├── cookie@0.6.0
│   └── ...
├── lodash@4.17.21
└── typescript@5.5.4 (devDependency)
```

包管理器负责：从注册表下载包、解析版本冲突、把包装到 `node_modules`、记录解析结果、执行生命周期脚本。

## 1.2 注册表与缓存

npm 默认从 `https://registry.npmjs.org` 下载。下载过的包会进入本地缓存，再次安装时优先命中缓存。pnpm 则使用内容寻址的全局存储，多个项目共享同一份文件。

★ **关键认知**：`node_modules` 不是"源码"，而是**可重建的产物**。它应该被 `.gitignore` 忽略，真正需要提交的是 `package.json` 和锁文件。任何人拿到这两样，都应该能装出可复现的依赖树。

# 2. package.json

## 2.1 必填字段与常用字段

```json
{
  "name": "notes-api",
  "version": "1.2.0",
  "description": "一个示例 API 服务",
  "type": "module",
  "main": "./dist/index.js",
  "exports": {
    ".": "./dist/index.js"
  },
  "engines": {
    "node": ">=20"
  },
  "scripts": {
    "dev": "node --watch src/index.js",
    "start": "node src/index.js",
    "test": "node --test"
  },
  "dependencies": {
    "express": "^4.19.2"
  },
  "devDependencies": {
    "typescript": "^5.5.4"
  }
}
```

| 字段 | 作用 |
|------|------|
| `name` | 包名，发布到 npm 时全局唯一 |
| `version` | 当前版本，遵循语义化版本 |
| `type` | `.js` 文件按 ESM 还是 CJS 解析 |
| `main` | 旧的 CommonJS 入口 |
| `exports` | 现代入口声明，可限制子路径 |
| `engines` | 声明支持的 Node 版本 |
| `scripts` | 可执行命令的别名 |
| `private` | 设为 `true` 可防止误发布 |

## 2.2 依赖类型

| 类型 | 用途 | 生产环境是否需要 |
|------|------|-----------------|
| `dependencies` | 运行时依赖，如 express | 需要 |
| `devDependencies` | 开发/构建/测试工具 | 不需要 |
| `peerDependencies` | 要求宿主项目提供的依赖 | 由宿主提供 |
| `optionalDependencies` | 装不上也不影响使用的依赖 | 可选 |
| `bundledDependencies` | 随包一起打包的依赖 | 随包发布 |

★ **关键认知**：`dependencies` 与 `devDependencies` 的分界是**运行时是否真的 require/import 它**。把构建工具放进 `dependencies` 会让生产镜像变大；把运行时库放进 `devDependencies` 则会在 `npm ci --omit=dev` 后直接启动失败。

`peerDependencies` 常见于插件：比如 `eslint-plugin-vue` 要求宿主项目自己安装 `eslint`。它不会被自动装进 `node_modules`(npm 7+ 会尝试安装，但版本由宿主决定)。

## 2.3 版本范围与语义化版本

语义化版本(SemVer)形如 `主版本.次版本.修订号`：

- **主版本**：不兼容的 API 变更；
- **次版本**：向后兼容的新功能；
- **修订号**：向后兼容的问题修复。

版本范围写法：

| 写法 | 含义 | 允许升级到 |
|------|------|-----------|
| `1.2.3` | 精确版本 | 只能 1.2.3 |
| `^1.2.3` | 兼容主版本 | `>=1.2.3 <2.0.0` |
| `~1.2.3` | 兼容次版本 | `>=1.2.3 <1.3.0` |
| `>=1.2.3` | 最低版本 | 无限上限 |
| `1.2.x` | 修订号任意 | `>=1.2.0 <1.3.0` |
| `*` | 任意版本 | 任意 |

★ **关键认知**：日常最常用的是 `^`。它允许自动获得修复和新功能，同时避免主版本破坏性变更。⚠ 但对 `0.x` 版本，`^0.2.3` 只会升到 `<0.3.0`，因为 0.x 被视为不稳定阶段；对这类包升级要更谨慎。

## 2.4 scripts 与生命周期

```json
{
  "scripts": {
    "build": "tsc -p tsconfig.json",
    "dev": "node --watch src/index.js",
    "test": "node --test",
    "lint": "eslint src",
    "prestart": "npm run build",
    "start": "node dist/index.js",
    "prepare": "npm run build"
  }
}
```

执行 `npm run start` 时，npm 会先自动执行 `prestart`，再执行 `start`，最后执行 `poststart`。`prepare` 在 `npm install` 之后、`npm publish` 之前执行，适合做构建。

⚠ **关键认知**：`scripts` 里的命令默认通过 shell 执行，跨平台时 `rm -rf`、`cp` 这类命令会失败。需要跨平台时用 `rimraf`、`shx`，或直接用 Node 脚本封装。

# 3. 锁文件

## 3.1 锁文件的作用

`package.json` 里的 `^1.2.3` 只描述**范围**，不同时间安装可能解析出不同版本。锁文件记录**这一次实际安装的精确版本与完整依赖树**，保证：

- 团队成员装出同样的版本；
- CI 与生产环境可复现；
- 依赖树的解析结果不会因注册表更新而漂移。

npm 使用 `package-lock.json`，pnpm 使用 `pnpm-lock.yaml`，Yarn 使用 `yarn.lock`。

```json
{
  "name": "notes-api",
  "lockfileVersion": 3,
  "packages": {
    "node_modules/express": {
      "version": "4.19.2",
      "resolved": "https://registry.npmjs.org/express/-/express-4.19.2.tgz",
      "integrity": "sha512-..."
    }
  }
}
```

`integrity` 是内容哈希，用于校验下载的包没有被篡改。

★ **关键认知**：锁文件**必须提交到仓库**。它是团队与部署环境之间的契约。⚠ 不要手动编辑锁文件，也不要把它加进 `.gitignore`；需要更新依赖时用包管理器命令重新生成。

## 3.2 npm ci 与 npm install

| 命令 | 行为 | 适用场景 |
|------|------|---------|
| `npm install` | 按 `package.json` 解析，必要时更新锁文件 | 日常开发 |
| `npm ci` | 严格按锁文件安装，不修改锁文件 | CI、生产构建 |

`npm ci` 会先删除 `node_modules`，再按锁文件安装，速度快且结果确定。如果锁文件与 `package.json` 不一致，它会直接报错。

★ **关键认知**：CI 里应该用 `npm ci`，不要用 `npm install`。后者可能在无人察觉时升级依赖，让"本地能过、线上挂掉"成为常态。

# 4. npm 与 pnpm

## 4.1 安装结构差异

npm 采用**扁平化**的 `node_modules`：

```text
node_modules/
├── express/
├── body-parser/        <- 被提升到顶层
├── lodash/
└── ...
```

扁平化解决了旧版嵌套过深的问题，但也带来了**幽灵依赖**：你的代码可以 `require("body-parser")`，尽管你从没在 `package.json` 里声明它。

pnpm 采用**内容寻址 + 符号链接**：

```text
node_modules/
├── .pnpm/              <- 所有包的真实位置
│   ├── express@4.19.2/
│   └── body-parser@1.20.2/
├── express -> .pnpm/express@4.19.2/node_modules/express
└── lodash -> .pnpm/lodash@4.17.21/node_modules/lodash
```

只有你显式声明的依赖才会出现在 `node_modules` 顶层，未声明的包无法被访问，从而消除幽灵依赖。所有项目共享全局存储，磁盘占用更小。

## 4.2 常用命令对照

| 操作 | npm | pnpm |
|------|-----|------|
| 初始化 | `npm init -y` | `pnpm init` |
| 安装全部 | `npm install` / `npm ci` | `pnpm install --frozen-lockfile` |
| 加依赖 | `npm i express` | `pnpm add express` |
| 加开发依赖 | `npm i -D typescript` | `pnpm add -D typescript` |
| 全局安装 | `npm i -g pkg` | `pnpm add -g pkg` |
| 运行脚本 | `npm run dev` | `pnpm dev` |
| 查看过时依赖 | `npm outdated` | `pnpm outdated` |
| 审计 | `npm audit` | `pnpm audit` |

★ **关键认知**：pnpm 的优势不只是"快"和"省磁盘"，更重要的是**严格的依赖隔离**。幽灵依赖在 npm 项目里可能潜伏很久，换到 pnpm 或升级包管理器时才突然暴露。⚠ 迁移到 pnpm 时，先修掉所有未声明却直接引用的包，否则安装后立刻报 `Cannot find module`。

## 4.3 版本锁定与 overrides

遇到间接依赖存在漏洞、但直接依赖尚未升级时，可以用 `overrides` 强制指定版本：

```json
{
  "overrides": {
    "minimist": "^1.2.8"
  }
}
```

pnpm 对应的是 `pnpm.overrides`：

```json
{
  "pnpm": {
    "overrides": {
      "minimist": "^1.2.8"
    }
  }
}
```

⚠ **关键认知**：`overrides` 是"最后手段"，它会绕过上游的版本约束，可能引入不兼容。使用后必须跑完整测试，并记录原因，等上游修复后及时移除。

# 5. 安全审计与依赖维护

## 5.1 npm audit

```bash
npm audit
npm audit --json
npm audit fix
```

审计会对照漏洞数据库，列出受影响的包、严重级别与修复建议。`npm audit fix` 会尝试在版本范围内升级；跨主版本修复需要 `--force`，风险更高。

| 级别 | 含义 | 处理优先级 |
|------|------|-----------|
| low | 影响有限 | 可排期 |
| moderate | 特定条件可利用 | 尽快 |
| high | 可直接造成危害 | 优先 |
| critical | 远程可利用、后果严重 | 立即 |

## 5.2 依赖维护策略

- **定期升级**：每月跑一次 `npm outdated`，分批升级；
- **先修漏洞再升级功能**：安全补丁优先；
- **小步提交**：一次只升一类依赖，出问题容易定位；
- **锁文件随提交更新**：升级后立刻提交 `package-lock.json`；
- **关注 `npm audit` 之外的来源**：GitHub Dependabot、Snyk 等可作为补充。

★ **关键认知**：审计报告里的漏洞**不一定真的影响你**。要判断漏洞代码是否在你的调用路径上、是否暴露给外部输入。⚠ 但也不要因为"应该没事"就长期忽略，尤其是 `critical` 与直接处理用户输入的包。

## 5.3 供应链风险

安装一个包时，它的 `postinstall` 脚本会以你的用户权限执行。恶意包可以借此读取环境变量、上传文件。

```json
{
  "scripts": {
    "postinstall": "node ./scripts/setup.js"
  }
}
```

防护手段：

- 使用 `npm ci --ignore-scripts` 跳过安装脚本(确认项目不需要时)；
- 锁定依赖版本，避免自动引入新发布的包；
- 使用私有 registry 或依赖白名单；
- 审查新增依赖的维护者、下载量与最近提交。

# 6. 常见错误与练习

## 6.1 常见错误

**错误一：把 `node_modules` 提交进仓库**

```text
# .gitignore
node_modules/
dist/
.env
```

**错误二：生产环境装入了开发依赖**

```bash
npm ci --omit=dev   # 只装 dependencies
```

如果启动时报缺包，先检查那个包是不是被误放进了 `devDependencies`。

**错误三：锁文件与 package.json 冲突**

手动改过 `package.json` 的版本后，`npm ci` 可能报错。此时在本地跑一次 `npm install` 更新锁文件，再提交。

**错误四：忽略 engines**

```json
{
  "engines": {
    "node": ">=20"
  }
}
```

在 CI 中显式使用对应 Node 版本，否则本地 Node 22 能跑、服务器 Node 18 直接语法报错。

## 6.2 练习

1. 创建一个新项目，添加 `express` 到 `dependencies`、`vitest` 到 `devDependencies`，然后分别用 `npm ci --omit=dev` 与完整安装对比 `node_modules` 大小。

2. 解释 `^1.2.3`、`~1.2.3`、`1.2.3` 三者的区别，并说明为什么库作者通常不建议在 `dependencies` 里写 `*`。

3. 用 `npm ls <包名>` 找出某个间接依赖是被谁引入的，再尝试用 `overrides` 把它固定到安全版本。

# 7. 小结

| 概念 | 要点 |
|------|------|
| package.json | 声明依赖范围、脚本、入口与 Node 版本 |
| 依赖类型 | dependencies 运行时，devDependencies 开发期 |
| SemVer | 主版本破坏、次版本新增、修订号修复 |
| 锁文件 | 记录精确依赖树，必须提交 |
| npm ci | 严格按锁文件安装，CI 首选 |
| pnpm | 内容寻址 + 符号链接，消除幽灵依赖 |
| 审计 | 定期检查，结合调用路径判断真实风险 |

下一篇进入 `fs`、`path`、`process`，讲清楚文件读写、路径处理、环境变量与文件流，以及如何正确处理文件系统错误。
