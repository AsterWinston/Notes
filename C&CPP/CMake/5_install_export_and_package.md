自己写的库能被别的项目用 `find_package` 找到，才算真正完成交付。这中间需要三样东西：安装规则、导出的 target 定义、包配置与版本文件。本篇用一个可安装的 `greet` 库，把整条链路走通，并说明每一步为什么不能省。

# 1. 为什么需要安装与导出

## 1.1 构建树与安装树

构建目录里的库路径是临时的，安装目录才是对外交付的稳定结构：

```text
构建树 build/                     安装树 install/
┌──────────────────────┐          ┌──────────────────────────────┐
│ libgreet.a           │          │ bin/                         │
│ greetTargets.cmake   │  install │ lib/libgreet.a               │
│ 源文件绝对路径        │ ───────► │ include/greet/greet.h        │
└──────────────────────┘          │ lib/cmake/greet/             │
                                  │   greetConfig.cmake          │
                                  │   greetConfigVersion.cmake   │
                                  │   greetTargets.cmake         │
                                  └──────────────────────────────┘
```

## 1.2 使用方的两种视角

| 视角 | 使用方式 | 依赖什么 |
|------|----------|----------|
| 同一个构建树 | `add_subdirectory` 或 `target_link_libraries` | 内存中的 target |
| 已安装的包 | `find_package(greet CONFIG REQUIRED)` | 安装目录里的 Config 与导出文件 |

★ **关键认知**：`install` 负责把文件放到正确位置，`export` 负责把 target 的用法要求写成文件。只有两者配合，使用方才能在不看源码的情况下正确编译和链接。

# 2. install 规则

## 2.1 安装 target

```cmake
include(GNUInstallDirs)   # 提供 CMAKE_INSTALL_LIBDIR 等标准目录变量

install(TARGETS greet
    EXPORT greetTargets
    ARCHIVE  DESTINATION ${CMAKE_INSTALL_LIBDIR}     # 静态库
    LIBRARY  DESTINATION ${CMAKE_INSTALL_LIBDIR}     # 动态库
    RUNTIME  DESTINATION ${CMAKE_INSTALL_BINDIR}     # 可执行文件与 Windows DLL
    INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
```

`EXPORT greetTargets` 把 `greet` 登记到一个导出集合中，后面 `install(EXPORT ...)` 才会生成对应的 target 文件。

| 目标类型 | 使用的关键字 | Windows 说明 |
|----------|--------------|--------------|
| 静态库 | `ARCHIVE` | `.lib` |
| 动态库 | `LIBRARY` | 导入库放 `ARCHIVE`，DLL 放 `RUNTIME` |
| 可执行文件 | `RUNTIME` | `.exe` |

## 2.2 安装头文件与资源

```cmake
# 整目录安装，结尾斜杠表示拷贝目录内容，而不是目录本身
install(DIRECTORY include/ DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
```

配置文件、许可证、脚本也用 `install(FILES ...)` 放到 `share/` 或 `etc/` 下。

## 2.3 GNUInstallDirs

`include(GNUInstallDirs)` 后可以使用标准目录变量：

| 变量 | 默认值 | 用途 |
|------|--------|------|
| `CMAKE_INSTALL_BINDIR` | `bin` | 可执行文件 |
| `CMAKE_INSTALL_LIBDIR` | `lib` 或 `lib64` | 库文件 |
| `CMAKE_INSTALL_INCLUDEDIR` | `include` | 头文件 |
| `CMAKE_INSTALL_DATADIR` | `share` | 数据文件 |
| `CMAKE_INSTALL_PREFIX` | 平台相关 | 安装根目录 |

这些是**相对路径**，最终位置由安装前缀拼接而成。不要把它们写死成绝对路径，否则包无法重定位。

★ **关键认知**：使用 `GNUInstallDirs` 的变量而不是手写 `lib`、`include`，可以让包在 `lib64` 发行版和自定义前缀下都保持正确布局。

# 3. 导出 target

## 3.1 安装导出

```cmake
install(EXPORT greetTargets
    FILE      greetTargets.cmake
    NAMESPACE greet::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/greet)
```

生成的文件会定义 `greet::greet`，并携带头文件路径、编译特性和依赖关系。

构建树里也可以导出，方便不安装就直接消费：

```cmake
export(EXPORT greetTargets
    FILE      ${CMAKE_CURRENT_BINARY_DIR}/greetTargets.cmake
    NAMESPACE greet::)
```

## 3.2 命名空间

| 名称 | 出现位置 | 说明 |
|------|----------|------|
| `greet` | 本项目内部 target 名 | `target_link_libraries(app PRIVATE greet)` |
| `greet::greet` | 导出后对外的名字 | 使用方必须写这个名字 |

命名空间能避免不同项目的 target 重名。安装导出时统一加 `NAMESPACE`，使用方看到的名称就稳定了。

## 3.3 安装后的布局

```text
install/
├── include/greet/greet.h
├── lib/libgreet.a
└── lib/cmake/greet/
    ├── greetConfig.cmake
    ├── greetConfigVersion.cmake
    └── greetTargets.cmake
```

`find_package(greet)` 会到 `lib/cmake/greet/` 下寻找 `greetConfig.cmake`，因此路径必须与 `install(EXPORT ...)` 的 `DESTINATION` 保持一致。

# 4. 生成包配置文件

## 4.1 为什么不能手写 Config

一个可重定位的 Config 文件需要处理：

1. 计算包自身的安装前缀。
2. 引入导出的 target 文件。
3. 检查运行时依赖。
4. 兼容不同 CMake 版本与组件请求。

手写这些逻辑容易漏掉路径处理。CMake 提供了 `CMakePackageConfigHelpers` 来自动生成。

## 4.2 生成 Config 与版本文件

```cmake
include(CMakePackageConfigHelpers)

# 从模板生成 Config，PACKAGE_INIT 会处理安装前缀
configure_package_config_file(
    cmake/greetConfig.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/greetConfig.cmake
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/greet)

# 生成版本文件，SameMajorVersion 表示主版本相同即兼容
write_basic_package_version_file(
    ${CMAKE_CURRENT_BINARY_DIR}/greetConfigVersion.cmake
    VERSION       ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion)

```

生成后把两个文件安装到同一目录即可，完整清单见 4.4 节。`COMPATIBILITY` 的常用取值：

| 取值 | 含义 |
|------|------|
| `AnyNewerVersion` | 安装版本不低于请求版本即可 |
| `SameMajorVersion` | 主版本相同，次版本不低于请求 |
| `SameMinorVersion` | 主次版本都相同 |
| `ExactVersion` | 必须完全一致 |

## 4.3 Config 模板

`cmake/greetConfig.cmake.in`：

```cmake
@PACKAGE_INIT@

# 引入导出的 target 定义
include("${CMAKE_CURRENT_LIST_DIR}/greetTargets.cmake")

# 检查依赖，例如 greet 用到了 Threads
include(CMakeFindDependencyMacro)
find_dependency(Threads)

# 校验调用方请求的组件
check_required_components(greet)
```

`@PACKAGE_INIT@` 会在生成时替换成一段处理安装前缀的代码，不能省略。

## 4.4 完整安装段落

把前面的规则整理进 `greet/CMakeLists.txt`：

```cmake
include(GNUInstallDirs)

add_library(greet STATIC src/greet.cpp)
add_library(greet::greet ALIAS greet)   # 构建树内部也可用命名空间

target_include_directories(greet PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>)

install(TARGETS greet
    EXPORT greetTargets
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})

install(DIRECTORY include/
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

install(EXPORT greetTargets
    FILE      greetTargets.cmake
    NAMESPACE greet::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/greet)

include(CMakePackageConfigHelpers)
configure_package_config_file(
    cmake/greetConfig.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/greetConfig.cmake
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/greet)
write_basic_package_version_file(
    ${CMAKE_CURRENT_BINARY_DIR}/greetConfigVersion.cmake
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion)
install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/greetConfig.cmake
    ${CMAKE_CURRENT_BINARY_DIR}/greetConfigVersion.cmake
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/greet)
```

★ **关键认知**：`BUILD_INTERFACE` 与 `INSTALL_INTERFACE` 必须成对出现。前者让构建树使用源码目录，后者让安装后的包使用相对安装前缀的路径，缺一个就会在某一侧写出错误的绝对路径。

# 5. 使用方验证

## 5.1 安装到本地前缀

```bash
# 配置、构建并安装到临时前缀
cmake -S . -B build
cmake --build build
cmake --install build --prefix D:/local/greet
```

使用方的 `CMakeLists.txt`：

```cmake
cmake_minimum_required(VERSION 3.20)
project(consumer LANGUAGES CXX)

# 指定安装前缀；也可以设置 CMAKE_PREFIX_PATH 环境变量
set(CMAKE_PREFIX_PATH "D:/local/greet")

find_package(greet 1.0 CONFIG REQUIRED)

add_executable(app main.cpp)
target_link_libraries(app PRIVATE greet::greet)
```

```bash
cmake -S . -B build
cmake --build build
```

## 5.2 验证导出属性

安装后可以用 `get_target_property(greet::greet INTERFACE_INCLUDE_DIRECTORIES)` 检查头文件路径。如果出现源码目录的绝对路径，说明 `INSTALL_INTERFACE` 没有生效，包不可重定位。

⚠ **关键认知**：`cmake --install` 默认安装到 `CMAKE_INSTALL_PREFIX`。测试时用 `--prefix` 指向临时目录，避免污染系统目录，也方便反复验证卸载与重装。

# 6. 常见错误

| 现象 | 原因 | 修复 |
|------|------|------|
| `find_package` 找不到自己的包 | Config 没安装到搜索路径 | 设置 `CMAKE_PREFIX_PATH` 或 `<Package>_DIR` |
| 使用方报头文件找不到 | 导出属性里只有源码绝对路径 | 使用 `BUILD_INTERFACE` 与 `INSTALL_INTERFACE` |
| 安装后包只能在原机器使用 | 写死了绝对路径 | 用 `GNUInstallDirs` 相对路径与 `@PACKAGE_INIT@` |
| `greet::greet` 不存在 | 没有 `install(EXPORT ...)` 或命名空间不一致 | 检查导出集合与 `NAMESPACE` |
| 版本检查总是失败 | 没生成版本文件或版本号不一致 | 检查 `write_basic_package_version_file` |
| 动态库安装后运行时报错 | DLL 不在可执行文件搜索路径 | 安装到 `bin`，或设置运行时路径 |

⚠ **关键认知**：调试导出问题，先看安装目录里是否同时存在 `Config.cmake`、`ConfigVersion.cmake` 和 `Targets.cmake`。缺任何一个，`find_package` 都会以不同方式失败。

# 7. 动手练习

1. 把 `greet` 改成同时安装静态库和动态库，观察 Windows 下 `ARCHIVE`、`LIBRARY`、`RUNTIME` 分别装了什么。
2. 故意删掉 `greetConfigVersion.cmake`，用 `find_package(greet 1.0 CONFIG REQUIRED)` 观察报错，再补回并验证版本兼容策略。
3. 写一个消费项目，分别通过 `CMAKE_PREFIX_PATH` 和 `greet_DIR` 找到你的包，比较两种方式的适用场景。

# 8. 小结

| 概念 | 要点 |
|------|------|
| `install(TARGETS)` | 按产物类型安装库与可执行文件 |
| `GNUInstallDirs` | 提供标准相对安装目录 |
| `install(EXPORT)` | 生成对外 target 定义并加命名空间 |
| Config 文件 | 由 `configure_package_config_file` 生成，可重定位 |
| 版本文件 | 由 `write_basic_package_version_file` 生成 |
| 构建树接口 | `BUILD_INTERFACE` 与 `INSTALL_INTERFACE` 成对使用 |
| 使用方 | `find_package(包 CONFIG REQUIRED)` 加 `包::目标` |

下一篇进入预设与交叉编译，讲清楚 `CMakePresets.json` 如何固化配置、工具链文件如何切换编译器和目标平台，以及单配置与多配置生成器的差异。
