本板块整理 C/C++ 语言基础与工程构建中最常用的两套工具：CMake 与 GNU Make。先从 C 和 C++ 的语法、指针、对象模型讲起，再解释"源码如何变成可执行文件、依赖如何被追踪、项目如何跨平台构建"。

# 1. 模块定位

## 1.1 本模块解决什么问题

- 初学者缺少从指针、内存、对象模型到 STL 的系统路径。
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

- 掌握 C 的类型、指针、内存管理和文件 IO。
- 掌握 C++ 的类、对象生命周期、继承多态、模板和 RAII。
- 理解目标、依赖、命令三要素。
- 掌握 CMake target、属性、依赖和安装导出。
- 掌握 Makefile 变量、模式规则、自动变量和依赖生成。
- 理解增量构建、并行构建和跨平台工具链。

## 2.2 能力目标

- 能独立编写和管理指针、内存的 C 程序。
- 能用现代 C++ 写出 RAII、移动语义和 STL 风格的代码。
- 能从零搭建一个可编译、可测试、可安装的 C/C++ 项目。
- 能定位"为什么没有重新编译""为什么链接不到库"。
- 能读懂老项目的 Makefile，并评估是否迁移到 CMake。
- 能为项目补充 CI 构建与测试步骤。

# 3. 文件清单与大纲

## 3.1 C_Basics

| 文件 | 主题 | 一句话简介 |
|------|------|-----------|
| [1_introduction_and_toolchain.md](C_Basics/1_introduction_and_toolchain.md) | 概览与工具链 | 编译链接流程、GCC/Clang 命令、最小程序 |
| [2_types_and_operators.md](C_Basics/2_types_and_operators.md) | 类型与运算符 | 基本类型、sizeof、溢出、类型转换 |
| [3_control_flow.md](C_Basics/3_control_flow.md) | 控制流 | 条件、循环、break/continue、goto 边界 |
| [4_function_and_recursion.md](C_Basics/4_function_and_recursion.md) | 函数与递归 | 声明定义、参数传递、栈帧、作用域 |
| [5_array_and_string.md](C_Basics/5_array_and_string.md) | 数组与字符串 | 数组、字符串函数、缓冲区溢出 |
| [6_pointer_and_memory.md](C_Basics/6_pointer_and_memory.md) | 指针与内存 | 指针运算、函数指针、const 指针、野指针 |
| [7_struct_enum_union.md](C_Basics/7_struct_enum_union.md) | 结构体与联合 | 对齐、位域、枚举、typedef |
| [8_dynamic_memory.md](C_Basics/8_dynamic_memory.md) | 动态内存 | malloc/free、泄漏、悬空指针、double free |
| [9_preprocessor_and_file_io.md](C_Basics/9_preprocessor_and_file_io.md) | 预处理与文件 IO | 宏、条件编译、头文件保护、文件读写 |
| [10_best_practices_and_summary.md](C_Basics/10_best_practices_and_summary.md) | 实践与总结 | 编码规范、调试工具、常见坑 |

## 3.2 Cpp_Basics

| 文件 | 主题 | 一句话简介 |
|------|------|-----------|
| [1_overview_and_toolchain.md](Cpp_Basics/1_overview_and_toolchain.md) | 概览与工具链 | 与 C 的关系、标准版本、最小程序 |
| [2_reference_and_const.md](Cpp_Basics/2_reference_and_const.md) | 引用与 const | 引用、constexpr、值类别入门 |
| [3_class_and_object.md](Cpp_Basics/3_class_and_object.md) | 类与对象 | 访问控制、成员函数、static 成员 |
| [4_constructor_destructor.md](Cpp_Basics/4_constructor_destructor.md) | 构造与析构 | 初始化列表、拷贝/移动、RAII、rule of five |
| [5_inheritance_and_polymorphism.md](Cpp_Basics/5_inheritance_and_polymorphism.md) | 继承与多态 | 虚函数、override/final、虚析构、对象切片 |
| [6_operator_overloading.md](Cpp_Basics/6_operator_overloading.md) | 运算符重载 | 成员/非成员、流运算符、比较运算符 |
| [7_template_basics.md](Cpp_Basics/7_template_basics.md) | 模板基础 | 函数模板、类模板、推导、特化 |
| [8_exception_and_raii.md](Cpp_Basics/8_exception_and_raii.md) | 异常与 RAII | 栈展开、noexcept、异常安全等级 |
| [9_namespace_and_module.md](Cpp_Basics/9_namespace_and_module.md) | 命名空间与模块 | 命名空间、头文件组织、C++20 module |
| [10_smart_pointer_and_move.md](Cpp_Basics/10_smart_pointer_and_move.md) | 智能指针与移动 | unique_ptr/shared_ptr、std::move、完美转发 |
| [11_stl_overview.md](Cpp_Basics/11_stl_overview.md) | STL 概览 | 容器、迭代器、算法、lambda、选型 |
| [12_modern_cpp_and_summary.md](Cpp_Basics/12_modern_cpp_and_summary.md) | 现代 C++ 与总结 | auto、范围 for、optional/variant、string_view |

## 3.3 STL

| 文件 | 主题 | 一句话简介 |
|------|------|-----------|
| [1_overview_and_containers.md](STL/1_overview_and_containers.md) | 概览与容器 | 序列/关联/无序/适配器、复杂度与选型 |
| [2_vector_and_string.md](STL/2_vector_and_string.md) | vector 与 string | 扩容、迭代器失效、string_view、SSO |
| [3_map_set_unordered.md](STL/3_map_set_unordered.md) | map/set 与哈希 | 比较器、自定义 key、复杂度对比 |
| [4_algorithm.md](STL/4_algorithm.md) | 算法 | sort/find/transform、谓词、erase-remove |
| [5_iterator_and_ranges.md](STL/5_iterator_and_ranges.md) | 迭代器与 ranges | 迭代器分类、失效规则、C++20 ranges |
| [6_lambda_and_functional.md](STL/6_lambda_and_functional.md) | lambda 与函数对象 | 捕获、mutable、std::function、生命周期 |
| [7_optional_variant_string_view.md](STL/7_optional_variant_string_view.md) | 现代工具类型 | optional、variant、any、string_view |
| [8_selection_and_best_practices.md](STL/8_selection_and_best_practices.md) | 选型与实践 | 容器选型、性能、异常安全、总结 |

## 3.4 Concurrency

| 文件 | 主题 | 一句话简介 |
|------|------|-----------|
| [1_overview_and_thread.md](Concurrency/1_overview_and_thread.md) | 概览与线程 | thread、join/detach、生命周期、并发数 |
| [2_mutex_and_lock.md](Concurrency/2_mutex_and_lock.md) | 互斥与锁 | mutex、lock_guard、unique_lock、死锁 |
| [3_condition_variable.md](Concurrency/3_condition_variable.md) | 条件变量 | wait/notify、虚假唤醒、生产者消费者 |
| [4_atomic_and_memory_order.md](Concurrency/4_atomic_and_memory_order.md) | 原子与内存序 | atomic、CAS、memory_order、无锁边界 |
| [5_future_async.md](Concurrency/5_future_async.md) | future 与 async | promise、packaged_task、异常传递 |
| [6_thread_pool_and_queue.md](Concurrency/6_thread_pool_and_queue.md) | 线程池与队列 | 任务队列、优雅关闭、背压 |
| [7_common_bugs_and_debugging.md](Concurrency/7_common_bugs_and_debugging.md) | 常见 bug 与调试 | 数据竞争、死锁、ABA、TSan/gdb |
| [8_best_practices_and_summary.md](Concurrency/8_best_practices_and_summary.md) | 实践与总结 | 避免共享、锁顺序、性能与可维护性 |

## 3.5 CMake

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

## 3.6 MakeFile

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
零基础语言                 现代 C++                 工程构建
    │                         │                        │
    ▼                         ▼                        ▼
C 1 → 2 → 3 → 4          Cpp 1 → 2 → 3            CMake 1 → 2
    │                         │                        │
    ▼                         ▼                        ▼
C 5 → 6 → 8              Cpp 4 → 5 → 8            CMake 3 → 5
    │                         │                        │
    ▼                         ▼                        ▼
C 9 → 10                 Cpp 10 → 11 → 12         CMake 7 → 8
```

- **零基础语言**：C 1 → 2 → 3 → 4 → 5 → 6 → 8，先掌握指针和内存。
- **C++ 主线**：Cpp 1 → 2 → 3 → 4 → 5 → 8 → 10 → 11 → 12，再读 STL 1 → 8。
- **并发进阶**：Concurrency 1 → 2 → 3 → 4 → 6 → 7 → 8。
- **工程构建**：CMake 1 → 2 → 3 → 5 → 7，或 MakeFile 1 → 3 → 4 → 7。
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
| C 基础 | 能正确使用指针、数组、结构体和动态内存 |
| C++ 基础 | 能写出 RAII、继承多态和模板代码 |
| STL | 能按场景选择容器、算法和现代工具类型 |
| 并发 | 能正确使用锁、条件变量、原子和线程池 |
| 构建模型 | 能说清目标、依赖、命令和增量构建 |
| CMake | 能写出现代 target-based 工程并安装导出 |
| Makefile | 能读懂变量、模式规则和依赖生成 |
| 排错 | 能定位未重编译、链接失败、依赖循环 |
| 工程化 | 能接入测试、打包和 CI |

建议先完成 C/C++ 语言主线，再学习 CMake/MakeFile；新项目优先使用 CMake，维护老项目时先读 MakeFile 模块，再评估迁移成本。
