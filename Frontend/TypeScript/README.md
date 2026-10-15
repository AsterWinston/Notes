# 4. TypeScript 学习笔记

TypeScript 是 JavaScript 的超集，在 JS 之上加了一套**编译期静态类型系统**。本模块共 23 篇，从类型基础讲到类型编程与工程化，只讲 TypeScript 语言本身（类型系统、语法、编译、声明、类型测试），不涉及任何前端/后端框架；示例全部内嵌正文，可直接运行。受众定位：已熟悉 JavaScript、想系统掌握 TypeScript 的开发者。

# 1. 目录

## 1.1 阶段一：基础（2-6 篇）

| 篇目 | 主题 | 一句话简介 |
|------|------|-----------|
| [1_introduction.md](1_introduction.md) | 简介与环境 | TS 定位、类型擦除、tsc 原理、可照抄的环境搭建命令 |
| [2_base_types.md](2_base_types.md) | 基础类型与类型推断 | 注解与推断、any/unknown/never 语义、as 断言、as const |
| [3_union_and_narrowing.md](3_union_and_narrowing.md) | 联合类型与类型收窄 | 收窄手段全集、类型守卫、可辨识联合、never 穷尽检查 |
| [4_function_types.md](4_function_types.md) | 函数类型 | 参数/返回值注解、重载、this 参数、回调与高阶函数 |
| [5_object_and_interface.md](5_object_and_interface.md) | 对象类型与接口 | interface、索引签名、声明合并、type vs interface 取舍 |
| [6_array_and_tuple.md](6_array_and_tuple.md) | 数组与元组 | 只读数组、元组进阶、as const 数组、数组方法签名解读 |

## 1.2 阶段二：面向对象与类型系统进阶（7-13 篇）

| 篇目 | 主题 | 一句话简介 |
|------|------|-----------|
| [7_class_and_oop.md](7_class_and_oop.md) | 类与面向对象 | 修饰符、#私有字段 vs private、参数属性、鸭子类型本质 |
| [8_enum.md](8_enum.md) | 枚举与字面量联合 | 数字/字符串/const 枚举、反向映射陷阱、as const 替代方案 |
| [9_generic_basic.md](9_generic_basic.md) | 泛型基础 | 泛型函数/类/接口、约束 extends、默认类型参数 |
| [10_conditional_types.md](10_conditional_types.md) | 条件类型与 infer | 分布式条件类型、infer 推断、手写 ReturnType、递归类型 |
| [11_mapped_and_template_types.md](11_mapped_and_template_types.md) | 映射类型与模板字面量类型 | 键重映射、属性修饰符改写、模板字面量、Record/Partial 原理 |
| [12_utility_types.md](12_utility_types.md) | 工具类型与类型编程实战 | 工具类型清单表、逐个手写实现、组合模式实战 |
| [13_type_compatibility.md](13_type_compatibility.md) | 类型兼容性与类型层级 | 协变逆变、strictFunctionTypes、类型层级全景、品牌类型 |

## 1.3 阶段三：工程化（14-18 篇）

| 篇目 | 主题 | 一句话简介 |
|------|------|-----------|
| [14_module_and_namespace.md](14_module_and_namespace.md) | 模块与声明合并 | ESM/CJS、模块解析、paths 别名、namespace、declare global |
| [15_declaration_files.md](15_declaration_files.md) | 声明文件 .d.ts 与 @types | declare 全家、手写库类型、模块增强、JSDoc 渐进加类型 |
| [16_tsconfig_deep_dive.md](16_tsconfig_deep_dive.md) | tsconfig 深入 | 编译选项逐项解析、strict 家族对照表、project references |
| [17_build_and_toolchain.md](17_build_and_toolchain.md) | 构建、运行与发布 | tsc/esbuild/swc 对比、d.ts 产出、双格式发布、命令清单 |
| [18_async_types.md](18_async_types.md) | 异步与运行时类型表达 | Promise/Awaited、Result 错误模式、类型化 fetch/事件封装 |

## 1.4 阶段四：高级专题（19-23 篇）

| 篇目 | 主题 | 一句话简介 |
|------|------|-----------|
| [19_type_programming.md](19_type_programming.md) | 类型编程实战 | 递归类型、字符串/算术类型、类型体操的工程边界 |
| [20_decorators.md](20_decorators.md) | 装饰器与元数据 | TC39 vs legacy 装饰器、执行顺序、手写 DI 与校验 |
| [21_strict_mode_and_pitfalls.md](21_strict_mode_and_pitfalls.md) | 严格模式与常见陷阱 | strict 逐开关反例、高频报错修复套路、静态类型≠运行时校验 |
| [22_type_testing.md](22_type_testing.md) | 类型测试与代码质量 | @ts-expect-error、Equal/Expect 断言、API 变更检测 |
| [23_interview_and_summary.md](23_interview_and_summary.md) | 面试要点与总结 | 高频面试题、工具类型手写清单、全系列知识地图 |

# 2. 推荐阅读路径

```
零基础入门        有 JS 基础           需要工程化           冲刺深入/面试
    │                │                    │                    │
    ▼                ▼                    ▼                    ▼
 1 → 2 → 3      1 → 2 → 5 ─┐        14 → 15 → 16        19 → 20 → 21
                4 → 6 → 7 ─┤        17 → 18              22 → 23
                           ├→ 8 → 9 → 10
                           │   11 → 12 → 13
```

- **速成路线（约 1 周）**：1 → 2 → 3 → 4 → 5 → 6 → 9 → 16，覆盖日常开发 90% 场景。
- **完整路线（约 3~4 周）**：按序号 1~23 顺读，每篇代码亲手敲一遍。
- **面试路线**：2 → 3 → 5 → 7 → 9 → 10 → 12 → 13 → 21 → 23。

# 3. 阅读约定

| 标记 | 含义 |
|------|------|
| `★ **关键认知**` | 该节最核心的机制/结论，优先记住 |
| `⚠` | 常见坑点或易误解处 |
| 代码块 | 均带逐行中文注释，可直接运行（标注"故意报错"的反例除外） |
| 命令清单 | 可照抄执行；npm 源均使用 npmmirror 国内镜像 |