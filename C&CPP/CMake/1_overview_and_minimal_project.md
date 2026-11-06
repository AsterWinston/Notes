CMake 是现代 C/C++ 项目最主流的构建系统生成器。很多初学者第一次接触它时，会被 `CMakeCache.txt`、生成器和构建类型绕晕，其实只要抓住"配置与构建分离"这一条主线，最小项目十分钟就能跑起来。本篇从定位讲起，带你写出第一个可运行的 CMake 工程。

# 1. CMake 是什么

## 1.1 定位：构建系统的生成器

CMake 本身**不编译代码**，它读取 `CMakeLists.txt`，再为当前平台生成一套原生构建文件。真正调用编译器的是 Ninja、Make、Visual Studio 或 Xcode。

```text
CMakeLists.txt ──┐
                 ├─► cmake 配置 ──► build/ 下的原生构建文件
源码与依赖 ──────┘                        │
                                          ▼
                                   cmake --build ──► 编译器 ──► 可执行文件
```

这层间接带来两个好处：

| 好处 | 说明 |
|------|------|
| 跨平台 | 同一份 CMakeLists 可在 Windows、Linux、macOS 上生成不同构建文件 |
| 跨工具链 | 换编译器、换 IDE 通常只改生成器或工具链文件，业务描述不动 |

★ **关键认知**：CMake 的产物是"构建文件"，不是二进制。因此同一个源码目录可以同时存在 `build-ninja/`、`build-vs/` 等多个构建目录，互不干扰。

## 1.2 一次配置，多平台生成

CMake 把项目描述抽象成 target、属性、依赖和用法要求（usage requirements），再由生成器翻译成具体平台能理解的规则。你在 `CMakeLists.txt` 里写的是"我要一个名为 `hello` 的可执行文件，它需要 C++17"，而不是"请调用 `g++ -std=c++17 main.cpp`"。

⚠ **关键认知**：不要把 CMake 当成高级版 Makefile 来用。手写编译命令、手拼链接参数，会丢掉依赖传播和跨平台能力，后期极难维护。

# 2. 构建流程

## 2.1 三个阶段

一次完整的 CMake 构建分为三步：

```text
1. 配置（configure）
   cmake -S . -B build
   读取 CMakeLists.txt，检查编译器，写 CMakeCache.txt 与生成文件

2. 生成（generate）
   配置阶段的一部分，输出 Ninja/Makefile/Visual Studio 工程

3. 构建（build）
   cmake --build build
   调用底层构建工具，真正编译与链接
```

`cmake -S . -B build` 中的 `-S` 指定源码目录，`-B` 指定构建目录。这两个参数从 CMake 3.13 起提供，3.20 及以上可以放心使用。

★ **关键认知**：**配置只做一次，构建可以做很多次**。修改源码后只需重新 `cmake --build`；修改 `CMakeLists.txt` 后，构建工具通常会自动触发重新配置。

## 2.2 源码内构建与源码外构建

CMake 支持两种构建方式：

| 方式 | 构建目录位置 | 是否推荐 | 原因 |
|------|--------------|----------|------|
| 源码内构建（in-source） | 与源码混在一起 | 不推荐 | 生成物污染源码树，清理困难 |
| 源码外构建（out-of-source） | 独立的 `build/` 目录 | 推荐 | 源码干净，可多配置并存，删目录即清理 |

源码外构建的典型目录：

```text
项目根目录/
├── CMakeLists.txt
├── main.cpp
└── build/                  # 生成物都在这里，可整体删除
    ├── CMakeCache.txt
    ├── CMakeFiles/
    ├── Makefile 或 build.ninja
    └── hello
```

⚠ **关键认知**：一旦在源码目录执行过 `cmake .`，就会留下 `CMakeCache.txt` 和 `CMakeFiles/`。再想切换到源码外构建时，必须先删掉这些残留，否则 CMake 会继续使用旧的缓存路径。

# 3. 最小可运行项目

## 3.1 目录结构

先建立一个只有两个源文件的工程：

```text
hello/
├── CMakeLists.txt
└── main.cpp
```

## 3.2 CMakeLists.txt

```cmake
# 声明最低 CMake 版本，3.20 起才支持本篇用到的部分特性
cmake_minimum_required(VERSION 3.20)

# 定义项目名、版本和启用的语言
project(hello
    VERSION 1.0.0
    DESCRIPTION "最小 CMake 示例"
    LANGUAGES CXX)

# 声明一个可执行目标，源文件相对当前 CMakeLists.txt 所在目录
add_executable(hello main.cpp)

# 只给 hello 这个目标启用 C++17，不影响同项目其他目标
target_compile_features(hello PRIVATE cxx_std_17)
```

`main.cpp` 保持最普通的样子：

```cpp
#include <iostream>

int main() {
    std::cout << "Hello, CMake!" << std::endl; // 用于确认程序确实运行
    return 0;
}
```

★ **关键认知**：`project()` 不只是起名字。它会探测编译器、设置 `PROJECT_SOURCE_DIR`、`PROJECT_BINARY_DIR` 等变量，并决定启用的语言。没有 `project()` 的工程在多数生成器下会直接报错。

## 3.3 配置、构建、运行

在 `hello/` 目录下执行：

```bash
# 配置并生成构建文件；-S 是源码目录，-B 是构建目录
cmake -S . -B build

# 调用底层构建工具编译链接
cmake --build build

# 运行产物；Windows 下换成 .\build\hello.exe
./build/hello
```

输出：

```text
Hello, CMake!
```

如果想指定生成器，在配置阶段加 `-G`：

```bash
cmake -S . -B build -G Ninja
```

⚠ **关键认知**：生成器只在**第一次配置**时生效，之后由 `CMakeCache.txt` 记住。要更换生成器，必须删除构建目录或换一个新的 `-B` 目录，不能直接对同一个构建目录再传 `-G`。

# 4. 生成器

## 4.1 常见生成器

| 生成器 | 平台 | 特点 | 适用场景 |
|--------|------|------|----------|
| `Ninja` | 全平台 | 速度快、增量构建精准 | 日常开发首选 |
| `Ninja Multi-Config` | 全平台 | 一个构建目录支持多种配置 | 需要 Debug/Release 并存 |
| `Unix Makefiles` | Linux、macOS | 兼容性最好 | 环境没有 Ninja 时 |
| `Visual Studio 17 2022` | Windows | 生成 `.sln` 工程 | 需要 VS 调试器或 IDE |
| `Xcode` | macOS | 生成 Xcode 工程 | macOS 原生开发 |

查看本机可用生成器：

```bash
cmake --help
```

输出末尾会列出 `Generators` 一节。也可以直接用：

```bash
cmake -S . -B build -G "Ninja Multi-Config"
```

## 4.2 如何选择

```text
需要 IDE 调试？
├── 是 ──► Windows 选 Visual Studio，macOS 选 Xcode
└── 否 ──► 需要 Debug/Release 同目录并存？
            ├── 是 ──► Ninja Multi-Config
            └── 否 ──► Ninja
```

★ **关键认知**：生成器选择不影响 `CMakeLists.txt` 的内容。如果你发现为了某个生成器要在脚本里写特殊分支，通常说明项目描述里混入了不该有的平台假设。

# 5. 常见错误与排查

| 现象 | 常见原因 | 处理方式 |
|------|----------|----------|
| `No CMAKE_CXX_COMPILER could be found` | 没装编译器或不在 PATH | 安装 GCC/Clang/MSVC，或在工具链文件中指定 |
| `CMake Error: The source directory ... does not appear to contain CMakeLists.txt` | `-S` 指错目录 | 确认当前目录有 `CMakeLists.txt` |
| `add_executable called with incorrect number of arguments` | 目标名或源文件为空 | 检查拼写与变量展开 |
| 修改 `CMakeLists.txt` 后行为没变 | 用错了构建目录 | 确认 `-B` 指向同一个目录 |
| 源码目录出现 `CMakeCache.txt` | 曾执行源码内构建 | 删除缓存与 `CMakeFiles/`，改用 `-B build` |
| `CMake 3.20 or higher is required` | 本机版本过低 | 升级 CMake，或按实际能力下调 `cmake_minimum_required` |

排查时优先看**第一条错误**，后面的报错往往是连锁反应。需要更详细日志时：

```bash
cmake -S . -B build --trace-expand
```

⚠ **关键认知**：`--trace` 输出量很大，只适合定位"某个变量为什么是空值"这类问题。日常排错先看 `CMakeCache.txt` 里的缓存值和 `CMakeFiles/CMakeOutput.log`。

# 6. 动手练习

1. 把 `project()` 里的 `LANGUAGES CXX` 改成 `LANGUAGES C`，观察配置阶段是否报错，并解释原因。
2. 用 `-G "Ninja Multi-Config"` 新建 `build-multi/`，分别构建 Debug 与 Release，比较产物路径差异。
3. 故意在源码目录执行一次 `cmake .`，再尝试源码外构建，记录报错并写出清理步骤。

# 7. 小结

| 概念 | 要点 |
|------|------|
| CMake 定位 | 构建系统生成器，不直接编译代码 |
| 三阶段 | 配置、生成、构建 |
| 最小工程 | `cmake_minimum_required` + `project` + `add_executable` |
| 推荐命令 | `cmake -S . -B build` 与 `cmake --build build` |
| 源码外构建 | 生成物隔离，删目录即清理 |
| 生成器 | 首次配置时用 `-G` 指定，之后由缓存记住 |

下一篇进入 target 与依赖，讲清楚为什么现代 CMake 应该"围绕目标编程"，以及 `PUBLIC`、`PRIVATE`、`INTERFACE` 如何决定头文件与编译选项的传播范围。
