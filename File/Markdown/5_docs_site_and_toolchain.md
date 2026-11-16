当笔记从几篇 `.md` 长到几十上百篇，单靠编辑器目录树就会失去全局感：找不到入口，没有搜索，链接靠记忆。把这些文件组织成静态文档站，是让知识可浏览、可检索、可长期维护的常用做法。本篇讲清工具选型、目录组织、导航配置、构建流程和部署方式。

# 1. 为什么把文档做成站点

## 1.1 从零散文件到可浏览的站点

静态文档站的核心思路是：**Markdown 仍是唯一内容源，构建工具负责把它转成 HTML**。

```text
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Markdown    │────▶│   构建工具    │────▶│  静态 HTML   │
│  内容源       │     │  解析 + 主题  │     │  CSS / JS    │
└──────────────┘     └──────────────┘     └──────┬───────┘
                                                  ▼
                                          ┌──────────────┐
                                          │  CDN / Pages │
                                          └──────────────┘
```

带来的能力：统一导航与侧边栏、全文搜索、代码高亮、稳定 URL、低成本部署。

## 1.2 静态站点与常见替代方案

| 方案 | 优点 | 缺点 | 适合场景 |
|------|------|------|----------|
| 静态文档站 | 版本可控、部署便宜、加载快 | 需要构建流程 | 技术文档、学习笔记 |
| Wiki | 上手快、多人协作 | 结构易失控、迁移难 | 内部知识库 |
| Word/PDF | 排版自由、便于打印 | 难 diff、难检索、难协作 | 正式交付物 |
| 在线文档平台 | 实时协作、权限完善 | 数据在第三方、导出受限 | 商业团队文档 |

★ **关键认知**：静态站点的价值不在"好看"，而在**文档即代码（docs as code）**。内容进 Git，改动走 Pull Request，构建和链接检查进 CI，评审、回滚、追责都和代码一样。这套流程才是长期维护的底气。

# 2. 主流工具选型

## 2.1 VitePress

Vue 生态的静态站点生成器，基于 Vite 和 Vue 3，配置简洁，默认主题适合技术文档。

特点：启动与热更新快；默认主题自带侧边栏、搜索和深色模式；可在 Markdown 中直接使用 Vue 组件；深度定制需要懂 Vue。

## 2.2 MkDocs Material

Python 生态，配置以 YAML 为主，Material 主题成熟，搜索和导航体验好。

```yaml
site_name: Notes
theme:
  name: material
  language: zh
nav:
  - 首页: index.md
  - Markdown:
      - 概览与语法: file/markdown/1_overview_and_syntax.md
markdown_extensions:
  - tables
  - footnotes
  - pymdownx.superfences
```

特点：配置简单，对非前端团队友好；插件围绕 Python-Markdown；主题功能完整；深度定制不如前端框架灵活。

## 2.3 Docusaurus

Meta 开源，基于 React，适合"文档 + 博客 + 版本化"一体的站点。

特点：文档版本化能力强，适合多版本产品手册；博客、文档、页面可放在同一站点；配置项较多，初次上手成本略高。

## 2.4 选型对比

| 维度 | VitePress | MkDocs Material | Docusaurus |
|------|-----------|-----------------|------------|
| 语言生态 | Vue / Node | Python | React / Node |
| 配置方式 | TS/JS | YAML | JS/TS |
| 上手难度 | 低 | 低 | 中 |
| 文档版本化 | 需自行实现 | 插件支持 | 原生支持 |
| 适合场景 | 技术笔记、开源文档 | 团队文档、内部手册 | 产品文档、多版本 |

★ **关键认知**：选型的第一原则是**跟着团队的技术栈走**。团队日常写 Python，就用 MkDocs；前端是 Vue，就用 VitePress；需要多版本文档和博客，再考虑 Docusaurus。工具差距不大，维护成本才决定成败。

# 3. 目录组织

## 3.1 典型结构

```text
project/
├── docs/
│   ├── .vitepress/config.ts
│   ├── index.md
│   ├── guide/
│   │   ├── index.md
│   │   ├── getting-started.md
│   │   └── images/flow.png
│   └── reference/api.md
├── package.json
└── README.md
```

约定：`docs/` 存放全部内容；每个目录放一个 `index.md` 作为栏目首页；图片放在使用它的文档同级 `images/` 下；配置与内容分开。

## 3.2 命名与路由

| 做法 | 示例 | 说明 |
|------|------|------|
| 小写加连字符 | `getting-started.md` | URL 友好，推荐 |
| 数字前缀 | `1_overview.md` | 控制排序，本仓库采用 |
| 中文文件名 | `快速开始.md` | 可读，但 URL 需编码 |
| 大写字母 | `GettingStarted.md` | Linux 下大小写敏感，易出错 |

★ **关键认知**：文件名就是 URL 的一部分，改文件名等于改链接。发布之后重命名，必须同步更新所有引用，必要时配置重定向。命名阶段多花一分钟，能省掉后面一堆 404。

⚠ Windows 和 macOS 默认不区分大小写，Linux 区分。本地 `Images/a.png` 能显示，部署到 Linux 后 `images/a.png` 就会 404。路径一律按实际大小写书写。

## 3.3 资源目录

两种常见放法：**就近放置**，图片放在文档同级 `images/`，移动文档时一起移动；**集中放置**，统一放 `docs/public/`，适合 logo 等共用资源。

```text
docs/
├── public/logo.svg
└── guide/
    ├── setup.md
    └── images/step-1.png
```

`public/` 下的文件通常按原样复制到站点根目录，引用时写 `/logo.svg`。

# 4. 导航与侧边栏

## 4.1 VitePress 配置

```ts
import { defineConfig } from "vitepress";

export default defineConfig({
  themeConfig: {
    nav: [{ text: "File", link: "/file/" }],
    sidebar: {
      "/file/markdown/": [{
        text: "Markdown",
        items: [
          { text: "概览与语法", link: "/file/markdown/1_overview_and_syntax" },
          { text: "GFM 扩展", link: "/file/markdown/2_gfm_extensions" }
        ]
      }]
    }
  }
});
```

`sidebar` 的键是路径前缀，值是该前缀下显示的菜单，链接省略 `.md` 后缀。

## 4.2 MkDocs 配置

```yaml
nav:
  - 首页: index.md
  - File:
      - Markdown:
          - 概览与语法: file/markdown/1_overview_and_syntax.md
          - GFM 扩展: file/markdown/2_gfm_extensions.md
```

MkDocs 的 `nav` 是显式的树形结构，层级就是侧边栏层级。

## 4.3 自动生成与显式配置

| 方式 | 优点 | 缺点 |
|------|------|------|
| 显式配置 | 顺序可控、标题可改写 | 新增文件要手动登记 |
| 按目录自动生成 | 新增即出现 | 顺序依赖文件名，标题难定制 |

★ **关键认知**：文档站的导航顺序应该**由内容规划决定**，而不是由文件名碰巧决定。宁可多维护一份显式配置，也不要把读者丢进一个按字母排序的目录里。

# 5. 构建流程

## 5.1 开发服务器

```bash
npm run docs:dev    # VitePress，保存后自动刷新
mkdocs serve        # MkDocs，默认 8000 端口
```

开发服务器提供热更新，并会暴露死链、语法错误等问题。

## 5.2 生产构建

```bash
npm run docs:build        # 产物通常在 docs/.vitepress/dist
mkdocs build --strict     # 产物在 site/，警告视为错误
```

`--strict` 适合放进 CI。VitePress 默认也会在构建时检查死链。

## 5.3 base 与子路径

部署到 `https://example.github.io/notes/` 这类子路径时，必须配置基础路径：

```ts
export default defineConfig({
  base: "/notes/"
});
```

```yaml
site_url: https://example.github.io/notes/
```

⚠ 最常见的部署事故就是 `base` 没配或配错：首页能打开，但 CSS、图片、站内链接全部 404。本地预览正常不代表线上正常，子路径部署必须用生产构建验证。

# 6. 部署

## 6.1 GitHub Pages 与 Actions

```yaml
name: Deploy Docs

on:
  push:
    branches: [main]

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm run docs:build
      - uses: actions/upload-pages-artifact@v3
        with:
          path: docs/.vitepress/dist

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

流程是：检出代码 → 安装依赖 → 构建 → 上传产物 → 部署。

## 6.2 其他平台

| 平台 | 适合场景 | 注意点 |
|------|----------|--------|
| GitHub Pages | 开源项目、个人笔记 | 子路径要配 `base` |
| Netlify | 需要预览部署 | 设置构建命令和产物目录 |
| Vercel | 前端项目一体化 | 框架预设可能覆盖配置 |
| 自建 Nginx | 内网、可控性强 | 自行处理缓存和 HTTPS |

自建服务器时，静态文件服务要处理无后缀路由和 404 回退，否则刷新子页面会白屏。

## 6.3 缓存与回滚

- 静态资源带哈希时设置长期缓存，HTML 设为不缓存或短缓存。
- 每次部署保留上一版产物，出问题能快速回滚。
- 构建产物和源文件分开，不要提交 `dist/`、`site/` 到仓库。

# 7. 常见坑

| 现象 | 原因 | 修复 |
|------|------|------|
| 线上样式丢失 | `base` 未配置 | 设置正确的子路径 |
| 图片 404 | 路径大小写不一致 | 按实际文件名书写 |
| 站内链接失效 | 文件重命名未同步 | 全局搜索并更新引用 |
| 构建失败 | 死链或语法错误 | 本地先跑生产构建 |
| 搜索无结果 | 语言或索引配置缺失 | 配置中文分词或语言 |
| 页面 404 | 路由后缀不一致 | 统一 `cleanUrls` 配置 |
| 构建很慢 | 图片过大、插件过多 | 压缩资源、按需启用插件 |

# 8. 动手练习

1. 用 VitePress 搭一个最小站点，包含首页、一个栏目和两篇文档，配置侧边栏并在本地预览。
2. 把 `base` 改成 `/notes/`，执行生产构建，用本地静态服务器模拟子路径访问，检查资源是否 404。
3. 用 GitHub Actions 配置自动部署，故意制造一个死链，观察构建是否失败并修复。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 工具选型 | 跟随团队技术栈，优先维护成本低的 |
| 目录组织 | 内容、配置、资源分离，命名稳定 |
| 导航 | 顺序由内容规划决定，显式配置更可控 |
| 构建 | 本地开发热更新，CI 用严格模式 |
| 部署 | 子路径必须配 `base`，产物不进仓库 |
| 验证 | 生产构建加实际访问，不能只看本地 |

下一篇进入规范与 lint，讲 markdownlint 规则、标题层级、行宽、链接检查、图片管理和团队文档规范，把前面五篇的写法固化成可执行的检查。
