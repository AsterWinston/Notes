CMake 学到最后，难点往往不是某个命令，而是工程习惯：什么时候用 target、目录怎么分、依赖怎么锁、出错从哪里查。本篇把前面七篇的规则收束成一套可执行的规范，并给出一份迁移检查清单。

# 1. 现代 CMake 的核心原则

## 1.1 以 target 为中心

现代 CMake 的所有配置都应尽量落到具体 target 上：

| 旧式全局命令 | 现代替代 | 原因 |
|--------------|----------|------|
| `include_directories` | `target_include_directories` | 只影响指定目标，可控制传播 |
| `link_libraries` | `target_link_libraries` | 明确依赖关系与链接顺序 |
| `add_definitions` | `target_compile_definitions` | 宏定义随 target 传播 |
| `add_compile_options` | `target_compile_options` | 避免影响无关目标 |
| `set(CMAKE_CXX_FLAGS ...)` | `target_compile_options` / `target_compile_features` | 不破坏全局编译参数 |
| `file(GLOB ...)` | 显式列出源文件 | 新增文件可被构建系统感知 |

★ **关键认知**：判断一条 CMake 命令是否现代，先看它有没有 target 参数。没有 target 参数的命令，通常意味着全局状态和隐式耦合。

## 1.2 用法要求随 target 传播

```cmake
add_library(greet STATIC src/greet.cpp)
target_include_directories(greet
    PUBLIC  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
            $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>)
target_compile_features(greet PUBLIC cxx_std_17)
target_link_libraries(greet PRIVATE Threads::Threads)
```

使用方只需要：

```cmake
target_link_libraries(app PRIVATE greet)
```

头文件路径、C++ 标准、私有依赖的链接信息都由 `greet` 自己声明。依赖的知识留在依赖内部，使用方不需要重复描述。

## 1.3 声明式而不是命令式

CMake 描述的是"要构建什么、目标之间是什么关系"，而不是"依次执行哪些编译命令"。因此：

```cmake
# 推荐：声明目标需要什么
target_link_libraries(app PRIVATE greet)

# 不推荐：手工拼路径和编译参数
target_link_libraries(app PRIVATE "${CMAKE_BINARY_DIR}/libgreet.a")
```

手拼路径会丢掉传递依赖、配置区分和跨平台差异，属于把构建系统降级成脚本。

# 2. 目录组织

## 2.1 推荐布局

```text
project/
├── CMakeLists.txt          # 顶层：项目声明与子目录入口
├── CMakePresets.json       # 共享构建预设
├── cmake/                  # 自定义模块、Config 模板、工具链
│   ├── FindXxx.cmake
│   ├── projectConfig.cmake.in
│   └── toolchains/
├── include/project/        # 公开头文件
├── src/                    # 实现与私有头文件
├── tests/                  # 测试
├── examples/               # 示例程序
└── third_party/            # 可选：内置第三方源码
```

公开头文件放在 `include/项目名/` 下，可以让使用方写 `#include <project/foo.h>`，也避免不同项目同名头文件冲突。

## 2.2 子目录与 target 边界

```text
app ──► greet ──► fmt
 │        │
 │        └──► Threads
 └──► math
```

每个逻辑单元一个 target：

| target | 类型 | 职责 |
|--------|------|------|
| `greet` | 静态库 | 问候逻辑与公开头文件 |
| `math` | 静态库 | 计算逻辑 |
| `app` | 可执行文件 | 组装入口，尽量薄 |
| `greet_tests` | 可执行文件 | 只测试 `greet` |

不要把所有源文件塞进一个巨大的 target。target 越细，依赖关系越清楚，增量构建和测试隔离也越容易。

## 2.3 测试与示例的位置

测试和示例应放在独立目录，并用选项控制：

```cmake
include(CTest)
if(BUILD_TESTING)
    add_subdirectory(tests)
endif()

option(BUILD_EXAMPLES "构建示例程序" OFF)
if(BUILD_EXAMPLES)
    add_subdirectory(examples)
endif()
```

★ **关键认知**：测试和示例也是使用方。它们应该像外部项目一样通过 target 链接库，而不是直接包含 `src/` 下的私有实现。这样才能验证公开接口是否真的可用。

# 3. 避免全局状态

## 3.1 全局命令的问题

```cmake
# 问题写法：影响之后创建的所有目标
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include)
add_definitions(-DDEBUG_LOG=1)
```

后果包括：

1. 目录顺序改变会影响哪些目标获得配置。
2. 第三方依赖也被迫接受你的宏和警告选项。
3. 删除某个依赖时，很难判断哪些全局设置还能删。
4. 同一个变量在不同子目录里被覆盖，排查困难。

## 3.2 推荐替代

| 需求 | 推荐写法 |
|------|----------|
| 头文件路径 | `target_include_directories(目标 ...)` |
| 宏定义 | `target_compile_definitions(目标 ...)` |
| 编译选项 | `target_compile_options(目标 ...)` |
| C++ 标准 | `target_compile_features(目标 ...)` |
| 链接库 | `target_link_libraries(目标 ...)` |
| 输出目录 | `set_target_properties(目标 PROPERTIES ...)` |

确实需要项目级默认值时，把默认值封装成接口库，让目标显式链接它：

```cmake
add_library(project_warnings INTERFACE)
target_compile_options(project_warnings INTERFACE
    $<$<CXX_COMPILER_ID:MSVC>:/W4>
    $<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-Wall>)

target_link_libraries(app PRIVATE project_warnings)
```

## 3.3 缓存与变量

缓存变量是全局的，使用时要克制：

1. 统一加项目前缀，例如 `MYPROJECT_BUILD_TOOLS`，避免和依赖冲突。
2. 用 `option` 表达开关，用 `CACHE STRING` 表达可配置值。
3. 不要在普通的 `CMakeLists.txt` 里用 `FORCE` 覆盖用户选择。
4. 不要把构建产物的绝对路径存入缓存，换机器就会失效。

⚠ **关键认知**：缓存变量适合"用户意图"，不适合"临时计算结果"。后者应该用普通变量，否则会污染 `CMakeCache.txt` 并导致难以复现的配置状态。

# 4. 依赖管理策略

## 4.1 选择顺序

```text
需要第三方库
├── 系统或环境已安装？ ──► find_package(CONFIG)
├── 团队使用包管理器？ ──► vcpkg / Conan
├── 依赖少且希望自包含？ ──► FetchContent
└── 内部库且需要改源码？ ──► Git 子模块 / 源码内置
```

| 方案 | 适合 | 不适合 |
|------|------|--------|
| `find_package` | 系统库、已安装 SDK | 需要精确控制源码版本 |
| vcpkg / Conan | 多平台、多配置、二进制复用 | 极简项目、无网络环境 |
| `FetchContent` | 依赖少、希望一次配置完成 | 大量依赖、频繁全量构建 |
| 子模块 / 源码内置 | 内部库、需要打补丁 | 公开第三方库的常规消费 |

## 4.2 版本固定与可复现

| 对象 | 固定方式 |
|------|----------|
| `FetchContent` | `GIT_TAG` 写标签或提交哈希，压缩包写 `URL_HASH` |
| vcpkg | `vcpkg.json` 加基线提交 |
| Conan | 锁文件或精确版本 |
| 系统包 | 在 CI 镜像或容器中固定版本 |
| CMake 本身 | 在预设中声明 `cmakeMinimumRequired` |

★ **关键认知**：可复现构建的目标不是"永远不升级"，而是"每次构建使用的依赖版本可以明确还原"。分支名和 `latest` 标签都不具备这个性质。

## 4.3 离线构建

离线环境需要提前准备三类东西：

1. 依赖源码或二进制包。
2. CMake 的查找路径或工具链文件。
3. 编译器与系统 SDK。

常用手段：

```bash
# 让 FetchContent 使用本地源码
cmake -S . -B build -DFETCHCONTENT_SOURCE_DIR_FMT="D:/deps/fmt"

# 指定包管理器生成的工具链
cmake -S . -B build -DCMAKE_TOOLCHAIN_FILE="D:/vcpkg/scripts/buildsystems/vcpkg.cmake"

# 指定自定义安装前缀
cmake -S . -B build -DCMAKE_PREFIX_PATH="D:/local/deps"
```

# 5. 排错方法

## 5.1 常用诊断命令

```bash
# 跟踪 CMake 脚本执行，定位变量来源
cmake -S . -B build --trace-expand --trace-source=CMakeLists.txt

# 查看 find_package 的查找过程
cmake -S . -B build --debug-find

# 查看详细构建命令
cmake --build build --verbose
```

CMake 还会在构建目录写入日志：

| 文件 | 用途 |
|------|------|
| `CMakeFiles/CMakeOutput.log` | 编译器探测成功记录 |
| `CMakeFiles/CMakeError.log` | 编译器探测失败原因 |
| `CMakeCache.txt` | 缓存变量与路径 |
| `compile_commands.json` | 每个源文件的实际编译命令 |

## 5.2 读错误信息

排查时按这个顺序看：

1. **第一条错误**：后面的错误常是连锁反应。
2. **目标名是否存在**：`target ... is not built` 多半是顺序或拼写问题。
3. **包是否找到**：区分"没安装"与"安装了但不在搜索路径"。
4. **链接错误**：`undefined reference` 优先检查是否链接了对应 target。

## 5.3 清理与复现

如果只改过 `CMakeLists.txt`，重新配置即可；改过工具链、编译器或缓存变量，应删除构建目录后重新配置；仍然异常时，用一个最小示例复现。

不要通过手工编辑 `CMakeCache.txt` 来"修复"问题。缓存是生成物，正确做法是删除构建目录或使用新的预设目录。

⚠ **关键认知**：CMake 的很多怪问题来自"旧构建目录 + 新配置"。在怀疑代码之前，先做一次干净构建，能排除大半环境问题。

# 6. 迁移与检查清单

从旧工程迁移到现代 CMake，可以按下面顺序推进：

| 阶段 | 动作 | 验收标准 |
|------|------|----------|
| 1 | 用 target 替换全局命令 | 不再出现 `include_directories`、`link_libraries` |
| 2 | 给每个库声明 `PUBLIC`/`PRIVATE` | 使用方只链接 target 即可编译 |
| 3 | 移除 `file(GLOB)` | 新增源文件后无需重新配置 |
| 4 | 统一 C++ 标准声明 | 用 `target_compile_features` |
| 5 | 引入预设 | 本地与 CI 使用同一套参数 |
| 6 | 补齐安装与导出 | 外部项目能 `find_package` |
| 7 | 接入 CTest | CI 自动运行测试 |
| 8 | 固定依赖版本 | 干净环境可复现构建 |

日常提交前可以快速自查：

1. `CMakeLists.txt` 是否只描述 target 与关系。
2. 是否有全局编译选项影响了第三方依赖。
3. 公开头文件路径是否用 `PUBLIC` 或 `INTERFACE` 正确传播。
4. 依赖版本是否固定。
5. 构建目录是否可以随时删除重建。

# 7. 动手练习

1. 找一个使用 `include_directories` 和 `link_libraries` 的旧工程，逐步改成 target 写法，并记录每一步解决了什么问题。
2. 给项目补上 `CMakePresets.json`、安装导出和 CTest，使外部项目能通过 `find_package` 使用你的库。
3. 故意制造一次"旧构建目录导致工具链错误"的问题，用干净构建和 `--debug-find` 定位并写出排查记录。

# 8. 小结

| 篇目 | 核心结论 |
|------|----------|
| 1 概览与最小工程 | 配置与构建分离，坚持源码外构建 |
| 2 target 与依赖 | 链接目标而不是路径，用三个关键字控制传播 |
| 3 变量与属性 | 区分变量与属性，注意作用域与求值阶段 |
| 4 依赖查找 | 优先导入目标，固定版本，选对供应方式 |
| 5 安装与导出 | `install` 加 `export` 才能被 `find_package` 消费 |
| 6 预设与交叉编译 | 用预设固化参数，用工具链切换目标平台 |
| 7 测试与打包 | CTest 管验证，CPack 管交付，CI 管执行 |
| 8 最佳实践 | 以 target 为中心，减少全局状态，保证可复现 |

现代 CMake 的工程规范可以压缩成三句话：**用 target 描述依赖，用属性表达用法要求，用预设和锁文件保证可复现**。做到这三点，项目就能在本地、CI 和不同平台上稳定构建。

CMake 系列到这里结束。下一步可以进入 Makefile 与编译链接原理，理解 CMake 生成的构建文件底层在做什么；也可以直接把这些规范应用到现有 C/C++ 项目中，从一次干净构建开始。
