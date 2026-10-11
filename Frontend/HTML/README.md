# HTML 学习笔记

HTML 是网页的骨架——用标签描述内容的**结构与语义**，浏览器据此解析出 DOM 树并渲染。本模块共 15 篇，从文档结构讲到表单、语义化与可访问性、再到资源加载与性能，是 CSS、JavaScript 乃至 Vue/React 等地基前的**第一课**。受众定位：零门槛入门或想系统补全 HTML 语义化/无障碍/性能细节的开发者。

# 1. 目录

## 1.1 阶段一：基础结构（1-5 篇）

| 篇目 | 主题 | 一句话简介 |
|------|------|-----------|
| [1_introduction.md](1_introduction.md) | 简介与标准 | HTML 定位、浏览器解析渲染、WHATWG 与 Living Standard |
| [2_document_structure.md](2_document_structure.md) | 文档结构与全局属性 | DOCTYPE、html/head/body、DOM 树、id/class/data-/lang |
| [3_meta_and_seo.md](3_meta_and_seo.md) | 元信息与 SEO | head 全家、charset/viewport、Open Graph、结构化数据 |
| [4_text_content.md](4_text_content.md) | 文本与内容标签 | 标题、段落、列表、引用、代码、语义区块 |
| [5_link_image_media.md](5_link_image_media.md) | 链接、图片与多媒体 | a/picture/audio/video/figure、响应式图片 |

## 1.2 阶段二：表单与交互（6-8 篇）

| 篇目 | 主题 | 一句话简介 |
|------|------|-----------|
| [6_form_basics.md](6_form_basics.md) | 表单基础 | form/label/input/select/textarea、提交与提交方式 |
| [7_input_types_validation.md](7_input_types_validation.md) | input 全家族与校验 | input 类型全集、required/pattern/约束校验 API |
| [8_form_advanced.md](8_form_advanced.md) | 表单进阶 | datalist/fieldset、无障碍表单、自定义控件 |

## 1.3 阶段三：语义化与可访问性（9-11 篇）

| 篇目 | 主题 | 一句话简介 |
|------|------|-----------|
| [9_semantic_tags.md](9_semantic_tags.md) | 语义化与文档大纲 | 语义化的意义、article/section/nav/aside、文档大纲 |
| [10_accessibility_aria.md](10_accessibility_aria.md) | 可访问性基础 | ARIA 初探、roles/states、键盘导航、焦点管理 |
| [11_embedding.md](11_embedding.md) | 嵌入与组合 | iframe/canvas/svg/template/slot、Web Components 边界 |

## 1.4 阶段四：工程与性能（12-15 篇）

| 篇目 | 主题 | 一句话简介 |
|------|------|-----------|
| [12_resource_loading_performance.md](12_resource_loading_performance.md) | 资源加载与性能 | preload/prefetch、defer/async、懒加载、关键渲染路径 |
| [13_templates_and_frameworks.md](13_templates_and_frameworks.md) | 模板与框架协同 | 模板引擎、HTML 模板语法、与 Vue/React 的关系 |
| [14_best_practices.md](14_best_practices.md) | 最佳实践与校验 | validator 校验、常见误区、代码风格 |
| [15_interview_and_summary.md](15_interview_and_summary.md) | 面试要点与总结 | 高频面试题、知识地图、进阶方向 |

# 2. 推荐阅读路径

```
零基础入门          有基础补漏         冲刺面试/深入
     │                  │                  │
     ▼                  ▼                  ▼
  1 → 2 → 4         1 → 3 → 6         6 → 7 → 9
  5 → 6 → 7        11 → 12            10 → 12 → 15
                ├→ 13 → 14
```

- **速成路线（约 1~2 天）**：1 → 2 → 4 → 5 → 6 → 7，覆盖常见页面结构写法。
- **完整路线**：按序号 1~15 顺读，每篇示例亲手敲一遍。
- **面试路线**：6 → 7 → 9 → 10 → 12 → 15。

# 3. 阅读约定

| 标记 | 含义 |
|------|------|
| `★ **关键认知**` | 该节最核心的机制/结论，优先记住 |
| `⚠` | 常见坑点或易误解处 |
| 代码块 | 均带逐行中文注释，可直接保存为 `.html` 在浏览器打开查看 |
| 命令清单 | 可照抄执行 |