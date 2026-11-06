C/C++ 项目几乎不可能完全不用第三方库。依赖从哪里来、怎么被找到、找到后怎么链接，是 CMake 工程能否稳定构建的关键。本篇围绕 `find_package` 的两种模式展开，再说明 `FetchContent` 与 vcpkg、Conan 的边界，帮你选对依赖管理方式。

# 1. 依赖管理的两条路

## 1.1 系统包与源码包

依赖可以来自两个方向：

| 来源 | 典型形式 | 优点 | 缺点 |
|------|----------|------|------|
| 预编译包 | 系统包管理器、vcpkg、Conan | 构建快，二进制可复用 | 需要匹配编译器、架构、Debug/Release |
| 源码包 | `FetchContent`、子模块、源码拷贝 | 可控、可打补丁、跨平台一致 | 配置与编译慢，依赖网络或仓库体积 |

CMake 本身不负责下载或安装依赖，它只负责**发现**依赖并提供统一的使用接口。

## 1.2 find_package 的定位

`find_package` 在已知位置查找包，并把结果暴露成**导入目标**或一组变量。使用方通过 `target_link_libraries(app PRIVATE 包::目标)` 消费，不需要关心库文件到底在哪里。

★ **关键认知**：现代 CMake 依赖消费的首选形式是**导入目标**（如 `fmt::fmt`、`ZLIB::ZLIB`、`Qt6::Core`），而不是 `Foo_INCLUDE_DIRS` 加 `Foo_LIBRARIES` 这类变量。导入目标会携带头文件路径、编译定义和传递依赖。

# 2. find_package 基础

## 2.1 基本调用

```cmake
find_package(ZLIB REQUIRED)
```

| 参数 | 含义 |
|------|------|
| `REQUIRED` | 找不到就报错停止配置 |
| `QUIET` | 找不到时不打印提示 |
| `EXACT` | 版本必须完全匹配 |
| `COMPONENTS` | 必须存在的组件 |
| `OPTIONAL_COMPONENTS` | 可选组件 |
| `CONFIG` | 只使用 Config 模式 |
| `MODULE` | 只使用 Find 模块模式 |

找不到时可以用 `if(ZLIB_FOUND)` 做降级处理，也可以加 `REQUIRED` 让配置直接失败。

## 2.2 版本与组件

```cmake
find_package(fmt 9 REQUIRED)
find_package(Qt6 6.5 REQUIRED COMPONENTS Core Widgets)
find_package(Boost 1.80 REQUIRED COMPONENTS filesystem)
```

版本通常遵循包的兼容策略。例如要求 `9` 时，可能接受 `9.x`，但不会接受 `8.x`，具体由包的版本文件决定。

组件缺失时，`Qt6_FOUND` 会是假值，可以用 `if(NOT Qt6_FOUND)` 给出更明确的提示。

## 2.3 导入目标

查找成功后，优先使用导入目标：

```cmake
find_package(fmt REQUIRED)
target_link_libraries(app PRIVATE fmt::fmt)

find_package(ZLIB REQUIRED)
target_link_libraries(app PRIVATE ZLIB::ZLIB)
```

目标名的命名空间由包作者决定，通常形如 `包名::组件名`。使用前可以用 `if(TARGET fmt::fmt)` 判断目标是否存在。

⚠ **关键认知**：`find_package` 成功不代表一定有导入目标。老包可能只提供变量，这时要么使用变量，要么在项目里自行封装一个 `IMPORTED` 目标，避免把旧式变量扩散到整个工程。

# 3. Config 模式

## 3.1 查找过程

Config 模式查找的是包自己安装的配置文件，通常位于 `<prefix>/lib/cmake/<包名>/` 下，文件名形如 `fmt-config.cmake`。`find_package(fmt)` 会到一系列前缀下寻找：

| 搜索位置 | 说明 |
|----------|------|
| `fmt_DIR` | 直接指定配置文件所在目录 |
| `CMAKE_PREFIX_PATH` | 自定义安装前缀列表 |
| 系统标准前缀 | `/usr`、`/usr/local`、`C:/Program Files` 等 |
| 环境变量 | 如 `PATH` 中的部分路径 |

找不到时最常见的报错是 `Could not find a package configuration file provided by "fmt"`，处理方式：

```bash
# 指定包配置目录
cmake -S . -B build -Dfmt_DIR="C:/libs/fmt/lib/cmake/fmt"

# 或指定安装前缀，让 CMake 自己去下面找
cmake -S . -B build -DCMAKE_PREFIX_PATH="C:/libs/fmt"
```

## 3.2 命名与变量

Config 文件通常由包作者生成，文件名可以是 `<Package>Config.cmake` 或 `<lowercase>-config.cmake`。查找成功后，CMake 会设置 `<Package>_FOUND`、`<Package>_DIR`、`<Package>_VERSION` 等变量。

## 3.3 一个典型 Config 文件

包作者生成的 Config 文件大致做这些事：

```cmake
# 引入导出的目标定义，并检查依赖
include("${CMAKE_CURRENT_LIST_DIR}/fmt-targets.cmake")
include(CMakeFindDependencyMacro)
find_dependency(Threads)

# 设置传统变量，兼容旧使用方
set(fmt_LIBRARIES fmt::fmt)
```

★ **关键认知**：Config 模式是**包作者主导**的。只要包正确导出了 target，使用方就只需要 `find_package` 加 `target_link_libraries`，不需要知道头文件路径和编译选项。

# 4. Find 模块模式

## 4.1 工作方式

Find 模块是 CMake 自带或项目自写的 `Find<Package>.cmake` 脚本。它不依赖包提供配置文件，而是先在 `CMAKE_MODULE_PATH` 中查找脚本，再去 CMake 安装目录的 `Modules/` 下查找，然后主动搜索头文件和库文件。常见内置模块有 `FindZLIB`、`FindThreads`、`FindOpenSSL`、`FindPython`。

## 4.2 自己写 Find 模块

假设有一个不在系统路径里的 `myjson` 库，可以写 `cmake/FindMyJSON.cmake`：

```cmake
# 查找头文件与库文件
find_path(MYJSON_INCLUDE_DIR NAMES myjson.h)
find_library(MYJSON_LIBRARY NAMES myjson)

# 统一处理找到与未找到的情况
include(FindPackageHandleStandardArgs)
find_package_handle_standard_args(MyJSON
    REQUIRED_VARS MYJSON_LIBRARY MYJSON_INCLUDE_DIR
    VERSION_VAR MYJSON_VERSION)

# 找到后创建导入目标，方便使用方统一写法
if(MyJSON_FOUND AND NOT TARGET MyJSON::MyJSON)
    add_library(MyJSON::MyJSON UNKNOWN IMPORTED)
    set_target_properties(MyJSON::MyJSON PROPERTIES
        IMPORTED_LOCATION "${MYJSON_LIBRARY}"
        INTERFACE_INCLUDE_DIRECTORIES "${MYJSON_INCLUDE_DIR}")
endif()

```

使用方需要先把模块目录加入搜索路径：

```cmake
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
find_package(MyJSON REQUIRED)
target_link_libraries(app PRIVATE MyJSON::MyJSON)
```

⚠ **关键认知**：`find_path` 和 `find_library` 的结果会写入缓存。第一次找错位置后，即使修正了搜索路径，也可能继续使用旧值，需要删除对应缓存项或清理构建目录。

# 5. FetchContent

## 5.1 基本用法

`FetchContent` 在配置阶段下载源码，并把它作为子项目加入当前构建：

```cmake
include(FetchContent)

FetchContent_Declare(
    fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt.git
    GIT_TAG        10.2.1          # 固定版本，避免构建结果漂移
    GIT_SHALLOW    TRUE)           # 只拉取最近历史，加快下载

FetchContent_MakeAvailable(fmt)

add_executable(app main.cpp)
target_link_libraries(app PRIVATE fmt::fmt)
```

`FetchContent_MakeAvailable` 会完成下载、加入子目录，并让 `fmt::fmt` 目标可用。如果用压缩包，则用 `URL` 指定地址，并用 `URL_HASH SHA256=...` 固定校验值。

## 5.2 固定版本与离线

| 关注点 | 做法 |
|--------|------|
| 版本可复现 | `GIT_TAG` 写标签或提交哈希，不要写分支名 |
| 下载加速 | 使用 `GIT_SHALLOW TRUE` 或直接下载发布包 |
| 内容校验 | 压缩包使用 `URL_HASH` |
| 离线构建 | 预先把源码放到缓存目录，或使用 `FETCHCONTENT_SOURCE_DIR_<NAME>` |
| 完全断网 | `FETCHCONTENT_FULLY_DISCONNECTED=ON`，要求源码已存在 |

离线时指定本地源码目录：

```bash
cmake -S . -B build \
  -DFETCHCONTENT_SOURCE_DIR_FMT="D:/deps/fmt-10.2.1"
```

⚠ **关键认知**：`GIT_TAG master` 或分支名会让构建结果随时间变化，今天能编过、明天可能失败。生产项目应固定到标签或提交哈希，并记录在锁文件或文档中。

## 5.3 与 find_package 配合

常见策略是**优先使用已安装的包，找不到再下载源码**：

```cmake
find_package(fmt 10 QUIET)
if(NOT fmt_FOUND)
    include(FetchContent)
    FetchContent_Declare(fmt
        GIT_REPOSITORY https://github.com/fmtlib/fmt.git
        GIT_TAG        10.2.1
        GIT_SHALLOW    TRUE)
    FetchContent_MakeAvailable(fmt)
endif()
target_link_libraries(app PRIVATE fmt::fmt)
```

本地开发可以复用系统包，CI 或干净环境则自动回退到源码构建。

★ **关键认知**：`FetchContent` 把依赖变成当前构建的一部分，因此依赖的编译选项、警告和测试也可能被一起引入。必要时通过 `EXCLUDE_FROM_ALL`、关闭依赖的测试选项来控制影响范围。

# 6. 包管理器边界

## 6.1 vcpkg

vcpkg 是微软主导的 C/C++ 包管理器，通过工具链文件与 CMake 集成：

```bash
# 安装依赖并生成工具链
vcpkg install fmt --triplet x64-windows

# 配置时传入 vcpkg 工具链
cmake -S . -B build \
  -DCMAKE_TOOLCHAIN_FILE="C:/vcpkg/scripts/buildsystems/vcpkg.cmake"
```

使用清单模式时，在项目根目录放 `vcpkg.json` 声明依赖，之后 `find_package(fmt CONFIG REQUIRED)` 就能正常工作。

## 6.2 Conan

Conan 通过 profile 与生成器提供依赖：

```bash
# 安装依赖并生成工具链，然后用工具链配置
conan install . --output-folder=build --build=missing
cmake -S . -B build -DCMAKE_TOOLCHAIN_FILE=build/conan_toolchain.cmake
```

`CMakeDeps` 生成 `find_package` 需要的配置文件，`CMakeToolchain` 负责编译器与构建类型设置。

## 6.3 选型对比

| 方案 | 依赖形态 | 版本锁定 | 离线能力 | 适合场景 |
|------|----------|----------|----------|----------|
| 系统包管理器 | 系统预编译 | 弱 | 强 | Linux 发行版集成、服务器部署 |
| vcpkg | 源码或二进制缓存 | 清单文件 | 较强 | 跨平台桌面与服务端项目 |
| Conan | 二进制包与配方 | 锁文件、版本范围 | 强 | 大型多团队、复杂二进制矩阵 |
| FetchContent | 源码 | 提交哈希 | 需预置源码 | 依赖少、希望完全自包含 |
| Git 子模块 | 源码 | 提交哈希 | 强 | 内部库、需要改源码 |

★ **关键认知**：`find_package` 是**消费接口**，vcpkg、Conan 是**依赖供应方式**。三者不是替代关系：包管理器负责把包装好，`find_package` 负责在 CMake 里找到并使用它。

# 7. 常见错误

| 现象 | 原因 | 修复 |
|------|------|------|
| `Could not find a package configuration file` | 包未安装或前缀不在搜索路径 | 设置 `<Package>_DIR` 或 `CMAKE_PREFIX_PATH` |
| 找到了包却链接失败 | 只设置了变量，没有链接导入目标 | 改用 `包::目标`，或正确使用变量 |
| 版本不匹配 | 安装版本低于要求 | 升级包，或放宽版本约束 |
| Windows 下 Debug 链接 Release 库 | 库的配置与当前构建不一致 | 使用包管理器区分配置，或统一构建类型 |
| `FetchContent` 下载超时 | 网络受限或仓库不可达 | 配置代理，或改用本地源码与离线变量 |
| 多个依赖拉取同一库的不同版本 | 目标名冲突 | 统一版本，或在顶层集中声明 |
| vcpkg 安装了但 CMake 找不到 | 没有传工具链文件 | 配置时加 `CMAKE_TOOLCHAIN_FILE` |
| Conan 安装后仍找不到包 | 没有使用生成的工具链 | 传 `conan_toolchain.cmake` |

排查依赖问题时，打开查找日志非常有效：

```bash
cmake -S . -B build --debug-find
```

⚠ **关键认知**：不要同时用多个包管理器管理同一个依赖。两套工具链都往 `CMAKE_PREFIX_PATH` 里塞路径时，实际命中的版本可能与你以为的不同。

# 8. 动手练习

1. 用 `find_package(ZLIB REQUIRED)` 写一个解压示例，分别用导入目标和传统变量两种方式链接，比较写法差异。
2. 写一个 `FindMyJSON.cmake`，故意把库放在非标准目录，通过 `CMAKE_PREFIX_PATH` 让 CMake 找到它。
3. 用 `FetchContent` 引入 Catch2，固定到某个标签，然后断网重建，记录失败信息并设计一个离线构建方案。

# 9. 小结

| 概念 | 要点 |
|------|------|
| `find_package` | 查找包并暴露导入目标或变量 |
| Config 模式 | 包提供 `*Config.cmake`，现代包首选 |
| Find 模块 | CMake 主动搜索头文件与库，兼容老包 |
| 导入目标 | `包::目标`，携带完整用法要求 |
| `FetchContent` | 配置阶段拉源码并加入构建 |
| vcpkg/Conan | 负责依赖供应与二进制管理 |
| 可复现 | 固定版本、固定哈希、记录工具链 |

下一篇进入安装与导出，讲清楚 `install` 规则、导出 target、生成包配置文件和版本文件，让你自己的库也能被别的项目用 `find_package` 找到。
