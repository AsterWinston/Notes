本板块整理 C/C++ 工程构建中最常用的两套工具：CMake 与 GNU Make。重点不是罗列命令，而是让初学者理解"源码如何变成可执行文件、依赖如何被追踪、项目如何跨平台构建"。

# 1. 模块定位

## 1.1 本模块解决什么问题

- 手写编译命令难以维护，需要可重复的构建脚本。
- 头文件依赖、增量编译、并行构建容易出错。
- 第三方库查找、安装、导出缺乏统一方式。
- 不同平台、不同编译器的构建参数差异大。
- 老项目使用 Makefile，新项目更多使用 CMake，需要能读懂并迁移。

## 1.2 与其他板块的关系

| 板块 | 关联 |
|------|------|
| C&CPP | 语言基础、编译链接、头文件与库 |
| Java/DevOps | CI 流水线中调用 CMake/Make |
| File | 构建配置文件的格式与校验 |

# 2. 学习目标

## 2.1 知识目标

- 理解目标、依赖、命令三要素。
- 掌握 CMake target、属性、依赖和安装导出。
- 掌握 Makefile 变量、模式规则、自动变量和依赖生成。
- 理解增量构建、并行构建和跨平台工具链。

## 2.2 能力目标

- 能从零搭建一个可编译、可测试、可安装的 C/C++ 项目。
- 能定位"为什么没有重新编译""为什么链接不到库"。
- 能读懂老项目的 Makefile，并评估是否迁移到 CMake。
- 能为项目补充 CI 构建与测试步骤。

# 3. 文件清单与大纲

## 3.1 CMake

| 文件 | 主题 | 一句话简介 |
|------|------|-----------|
| [1_overview_and_minimal_project.md](CMake/1_overview_and_minimal_project.md) | 概览与最小工程 | 构建流程、CMakeLists、out-of-source 构建 |
| [2_targets_and_dependencies.md](CMake/2_targets_and_dependencies.md) | Target 与依赖 | add_executable/add_library、链接可见性 |
| [3_variables_and_properties.md](CMake/3_variables_and_properties.md) | 变量与属性 | 作用域、缓存变量、编译选项、生成器表达式 |
| [4_find_package_and_dependencies.md](CMake/4_find_package_and_dependencies.md) | 查找依赖 | find_package、FetchContent、vcpkg/Conan |
| [5_install_export_and_package.md](CMake/5_install_export_and_package.md) | 安装与导出 | install、export、package config |
| [6_presets_toolchain_cross_compile.md](CMake/6_presets_toolchain_cross_compile.md) | Preset 与交叉编译 | CMakePresets、工具链文件、平台差异 |
| [7_testing_and_packaging.md](CMake/7_testing_and_packaging.md) | 测试与打包 | CTest、CPack、CI 集成 |
| [8_best_practices_and_summary.md](CMake/8_best_practices_and_summary.md) | 实践与总结 | 现代 CMake、目录组织、排错清单 |

## 3.2 MakeFile

| 文件 | 主题 | 一句话简介 |
|------|------|-----------|
| [1_overview_and_rules.md](MakeFile/1_overview_and_rules.md) | 概览与规则 | 目标、依赖、命令与增量构建 |
| [2_variables_and_functions.md](MakeFile/2_variables_and_functions.md) | 变量与函数 | 赋值方式、自动变量、常用函数 |
| [3_pattern_rules_and_automatic_variables.md](MakeFile/3_pattern_rules_and_automatic_variables.md) | 模式规则 | 模式规则、静态模式、自动变量 |
| [4_dependency_generation.md](MakeFile/4_dependency_generation.md) | 依赖生成 | `-MMD -MP`、头文件依赖与增量编译 |
| [5_phony_and_parallel_builds.md](MakeFile/5_phony_and_parallel_builds.md) | 伪目标与并行 | `.PHONY`、`-j`、顺序依赖与错误处理 |
| [6_c_cpp_project_template.md](MakeFile/6_c_cpp_project_template.md) | 项目模板 | 目录结构、编译选项、调试与发布 |
| [7_debugging_and_troubleshooting.md](MakeFile/7_debugging_and_troubleshooting.md) | 调试与排错 | `make -n/-p`、tab 错误、循环依赖 |
| [8_best_practices_and_summary.md](MakeFile/8_best_practices_and_summary.md) | 实践与总结 | Make 与 CMake 对比、可维护性 |

# 4. 推荐阅读路径

```text
零基础构建                 现代 CMake               维护老项目
    │                         │                        │
    ▼                         ▼                        ▼
MakeFile 1 → 2            CMake 1 → 2              MakeFile 1 → 7
    │                         │                        │
    ▼                         ▼                        ▼
MakeFile 3 → 4            CMake 3 → 5              MakeFile 8
    │                         │
    ▼                         ▼
MakeFile 5 → 6            CMake 6 → 8
```

- **快速上手**：CMake 1 → 2 → 3，先能构建自己的项目。
- **依赖与分发**：CMake 4 → 5 → 7，掌握第三方库和安装包。
- **老项目维护**：MakeFile 1 → 3 → 4 → 7，先能读懂和排错。

# 5. 阅读约定

| 标记 | 含义 |
|------|------|
| `★ **关键认知**` | 本节最核心的机制或结论 |
| `⚠` | 常见坑点或易误解处 |
| 代码块 | 均标注语言，示例尽量可直接运行 |
| 命令清单 | 可照抄执行，注意替换路径和编译器 |

# 6. 小结

| 主题 | 完成标准 |
|------|----------|
| 构建模型 | 能说清目标、依赖、命令和增量构建 |
| CMake | 能写出现代 target-based 工程并安装导出 |
| Makefile | 能读懂变量、模式规则和依赖生成 |
| 排错 | 能定位未重编译、链接失败、依赖循环 |
| 工程化 | 能接入测试、打包和 CI |

建议新项目优先使用 CMake；维护老项目时先读 MakeFile 模块，再评估迁移成本。
