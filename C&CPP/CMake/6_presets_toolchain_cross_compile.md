同一份代码，在开发机、CI、ARM 板子上往往需要不同的编译器、构建类型和缓存变量。每次都手敲一长串 `-D` 参数既容易出错，也无法复现。`CMakePresets.json` 与工具链文件就是用来把这些选择固化下来的。

# 1. CMakePresets 概览

## 1.1 解决什么问题

没有预设时，一次配置可能要在命令行里塞进生成器、构建类型、导出编译命令和工具链文件等一长串参数。问题在于：它们散落在文档、脚本和人的记忆里，换一台机器就要重新拼。预设把这些参数写进版本控制，用名字代替长命令：

```bash
cmake --preset arm-debug
```

## 1.2 文件结构

`CMakePresets.json` 是严格 JSON，顶层通常包含：

| 字段 | 作用 |
|------|------|
| `version` | 预设格式版本，需与 CMake 版本匹配 |
| `cmakeMinimumRequired` | 要求的最低 CMake 版本 |
| `configurePresets` | 配置阶段的预设 |
| `buildPresets` | 构建阶段的预设 |
| `testPresets` | 测试阶段的预设 |

`version` 不是 CMake 版本号，而是预设 schema 版本。schema 2 对应 CMake 3.20，schema 3 对应 3.21。以 CMake 3.20 为底线时，用 schema 2 最稳；需要 `condition` 等新字段时再升到 3。

★ **关键认知**：`CMakePresets.json` 应该提交到仓库，里面只放**可共享的构建意图**；个人路径、令牌等放在 `CMakeUserPresets.json`，并加入 `.gitignore`。

# 2. 配置预设

## 2.1 最小示例

```json
{
  "version": 2,
  "cmakeMinimumRequired": { "major": 3, "minor": 20, "patch": 0 },
  "configurePresets": [
    {
      "name": "debug",
      "displayName": "Debug",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/debug",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Debug",
        "CMAKE_EXPORT_COMPILE_COMMANDS": "ON"
      }
    }
  ]
}
```

| 变量 | 含义 |
|------|------|
| `${sourceDir}` | 源码根目录 |
| `${binaryDir}` | 当前预设的构建目录 |
| `${presetName}` | 当前预设名 |
| `${hostSystemName}` | 主机系统名，如 Windows、Linux |

使用：

```bash
cmake --preset debug
cmake --build build/debug
```

## 2.2 继承与变量

多个预设共享配置时，用 `hidden` 加 `inherits`：

```json
{
  "version": 2,
  "configurePresets": [
    {
      "name": "base",
      "hidden": true,
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/${presetName}"
    },
    {
      "name": "debug",
      "inherits": "base",
      "cacheVariables": { "CMAKE_BUILD_TYPE": "Debug" }
    }
  ]
}
```

`hidden: true` 的预设不会出现在 `--list-presets` 中，只作为模板使用。`inherits` 可以是一个字符串或数组，后面的预设会覆盖前面的同名变量。需要 Release 时再加一个继承 `base` 的预设即可。

## 2.3 条件与环境

需要按平台切换生成器时，使用 `condition`（schema 3 起）：

```json
{
  "name": "windows-ninja",
  "inherits": "base",
  "condition": { "type": "equals", "lhs": "${hostSystemName}", "rhs": "Windows" },
  "generator": "Ninja"
}
```

预设也可以设置环境变量，常用来指定包管理器根目录：

```json
{
  "name": "vcpkg",
  "inherits": "base",
  "environment": {
    "VCPKG_ROOT": "C:/vcpkg"
  },
  "cacheVariables": {
    "CMAKE_TOOLCHAIN_FILE": "$env{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake"
  }
}
```

⚠ **关键认知**：JSON 不支持注释。想解释预设用途，用 `displayName` 和 `description` 字段，不要写 `//`，否则 CMake 会直接解析失败。

# 3. 构建与测试预设

## 3.1 buildPresets

```json
{
  "version": 2,
  "buildPresets": [
    {
      "name": "debug",
      "configurePreset": "debug",
      "jobs": 8,
      "targets": ["app"]
    }
  ]
}
```

```bash
cmake --build --preset debug
```

`configurePreset` 必须指向一个存在的配置预设。`jobs` 控制并行度，`targets` 限定构建目标。

## 3.2 testPresets

```json
{
  "version": 2,
  "testPresets": [
    {
      "name": "debug",
      "configurePreset": "debug",
      "output": { "outputOnFailure": true }
    }
  ]
}
```

```bash
ctest --preset debug
```

★ **关键认知**：预设只是**参数容器**，不改变项目逻辑。构建规则仍然写在 `CMakeLists.txt` 里，预设负责选择"用哪套参数执行"。

# 4. 工具链文件

## 4.1 为什么需要

工具链文件在 `project()` 之前被读取，用来告诉 CMake 目标平台是什么、使用哪些编译器、去哪里找目标平台的库和头文件。它适合交叉编译、切换编译器、使用特殊 SDK 等场景。

## 4.2 基本内容

```cmake
# 目标系统与处理器架构
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR arm)

# 交叉编译器
set(CMAKE_C_COMPILER   arm-linux-gnueabihf-gcc)
set(CMAKE_CXX_COMPILER arm-linux-gnueabihf-g++)

# 目标平台的根文件系统
set(CMAKE_SYSROOT /opt/arm-sysroot)

# 查找规则：程序在主机找，库和头文件只在目标根下找
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)
```

| 变量 | 作用 |
|------|------|
| `CMAKE_SYSTEM_NAME` | 目标操作系统，设置后进入交叉编译模式 |
| `CMAKE_SYSTEM_PROCESSOR` | 目标架构 |
| `CMAKE_C_COMPILER` | C 编译器 |
| `CMAKE_CXX_COMPILER` | C++ 编译器 |
| `CMAKE_SYSROOT` | 目标平台根文件系统 |
| `CMAKE_FIND_ROOT_PATH_MODE_*` | 控制查找时是否限制在目标根下 |

## 4.3 交叉编译示例

```bash
# 通过命令行传入工具链文件
cmake -S . -B build-arm \
  -DCMAKE_TOOLCHAIN_FILE=cmake/toolchains/arm-linux.cmake \
  -DCMAKE_BUILD_TYPE=Release

cmake --build build-arm
```

在支持该字段的 schema 3 中，也可以用配置预设的 `toolchainFile` 指向同一个文件，再用 `cmake --preset arm-release` 调用。

⚠ **关键认知**：`CMAKE_TOOLCHAIN_FILE` 必须在**第一次配置前**指定。已经配置过的构建目录会缓存编译器和系统信息，换工具链后必须删除构建目录重新配置，否则会出现主机与目标文件混链。

# 5. 构建类型与平台差异

## 5.1 单配置与多配置

| 生成器类型 | 代表 | 构建类型指定方式 |
|------------|------|------------------|
| 单配置 | Ninja、Unix Makefiles | 配置时 `CMAKE_BUILD_TYPE` |
| 多配置 | Visual Studio、Xcode、Ninja Multi-Config | 构建时 `--config` |

单配置常用取值：

| 取值 | 含义 |
|------|------|
| `Debug` | 无优化，带调试信息 |
| `Release` | 高优化，通常定义 `NDEBUG` |
| `RelWithDebInfo` | 优化并保留调试信息 |
| `MinSizeRel` | 以最小体积为目标优化 |

多配置构建：

```bash
cmake -S . -B build -G "Ninja Multi-Config"
cmake --build build --config Debug
cmake --build build --config Release
```

⚠ **关键认知**：多配置生成器会忽略 `CMAKE_BUILD_TYPE`。用 `if(CMAKE_BUILD_TYPE STREQUAL "Debug")` 判断配置，在多配置下可能永远不成立，应改用生成器表达式 `$<CONFIG:Debug>`。

## 5.2 编译器与平台差异

常见做法是用生成器表达式按编译器选择选项，例如 `$<$<CXX_COMPILER_ID:MSVC>:/W4>` 与 `$<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-Wall>`。

| 差异点 | GCC/Clang | MSVC |
|--------|-----------|------|
| 警告选项 | `-Wall -Wextra` | `/W4` |
| 优化 | `-O2`、`-O3` | `/O2` |
| 调试信息 | `-g` | `/Zi` |
| 运行时库 | 通常静态链接 libc++/libstdc++ 可配置 | `/MT`、`/MD` 需显式选择 |
| 动态库导出 | 默认可见 | 需要 `__declspec(dllexport)` 或导出头 |

★ **关键认知**：平台差异尽量用**生成器表达式和 target 属性**表达，不要把 `if(WIN32)` 散落到业务逻辑里。平台判断越集中，移植成本越低。

# 6. 常见错误

| 现象 | 原因 | 修复 |
|------|------|------|
| `--preset` 找不到预设 | 文件名或预设名拼错 | 用 `cmake --list-presets` 查看 |
| 预设版本报错 | schema 版本高于 CMake 支持 | 降低 `version` 或升级 CMake |
| 修改预设后行为没变 | 构建目录已有旧缓存 | 删除对应构建目录重新配置 |
| 多配置下 Debug 宏不生效 | 用了 `CMAKE_BUILD_TYPE` 判断 | 改用 `$<CONFIG:Debug>` |
| 交叉编译仍链接主机库 | 查找模式未限制 | 设置 `CMAKE_FIND_ROOT_PATH_MODE_*` |
| 换工具链后编译异常 | 旧构建目录缓存了主机编译器 | 删除构建目录，重新配置 |
| 预设里的环境变量为空 | 变量名拼错或未定义 | 用 `$env{NAME}` 并在环境中提供 |
| JSON 解析失败 | 写了注释或尾随逗号 | 使用严格 JSON |

排查预设问题时：

```bash
cmake --list-presets
cmake --preset debug --trace-expand
```

# 7. 动手练习

1. 写一个 `base` 隐藏预设，派生出 `debug` 与 `release`，分别用 `cmake --preset` 配置并构建。
2. 为 Windows 与 Linux 各写一个 `condition` 预设，自动选择不同的生成器。
3. 写一个 ARM Linux 工具链文件，用 `CMAKE_CROSSCOMPILING` 在 CMakeLists 中打印提示，并解释为什么交叉编译时不能直接运行测试程序。

# 8. 小结

| 概念 | 要点 |
|------|------|
| `CMakePresets.json` | 把配置、构建、测试参数固化为命名预设 |
| 继承 | `hidden` 加 `inherits` 复用公共配置 |
| 工具链文件 | 在 `project()` 前指定目标平台与编译器 |
| 交叉编译 | 设置系统名、编译器、sysroot 与查找模式 |
| 单配置 | 配置时指定 `CMAKE_BUILD_TYPE` |
| 多配置 | 构建时用 `--config`，忽略 `CMAKE_BUILD_TYPE` |
| 平台差异 | 用生成器表达式集中处理 |

下一篇进入测试与打包，讲清楚 CTest 如何注册和运行测试、CPack 如何生成安装包，以及如何把构建、测试和覆盖率接入 CI。
