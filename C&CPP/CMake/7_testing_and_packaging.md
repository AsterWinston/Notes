构建成功不等于代码正确，能编译也不等于能交付。CTest 负责把测试纳入构建系统，CPack 负责把安装结果打成可分发的包。本篇从注册第一个测试讲起，一直走到 CI 中的自动测试与覆盖率。

# 1. 测试与打包的定位

## 1.1 为什么构建系统要管测试

如果测试只能靠手工运行，CI 就无法判断一次提交是否破坏了功能。CMake 通过 CTest 提供统一入口：

```text
CMakeLists.txt ──► 注册测试 ──► CTestTestfile.cmake
                                      │
                                      ▼
                              ctest 统一调度 ──► 通过 / 失败
```

这样本地和 CI 使用同一条命令，测试结果也可被 CI 平台直接解析。

## 1.2 CTest 与 CPack 的关系

| 工具 | 输入 | 输出 | 解决的问题 |
|------|------|------|------------|
| CTest | `add_test` 注册的测试 | 测试报告与退出码 | 验证功能是否正确 |
| CPack | `install` 规则与 CPack 变量 | `.zip`、`.tar.gz`、`.deb`、安装器等 | 生成可分发交付物 |

★ **关键认知**：CPack 不负责决定"包里有什么"，它读取的是 `install` 规则。没有安装规则，打出来的包就是空的。

# 2. CTest 基础

## 2.1 启用测试

推荐使用 `include(CTest)`，它会启用测试并定义 `BUILD_TESTING` 选项：

```cmake
include(CTest)

if(BUILD_TESTING)
    add_subdirectory(tests)
endif()
```

`BUILD_TESTING` 默认是 `ON`，发布构建时可以传 `-DBUILD_TESTING=OFF` 跳过测试目标。

## 2.2 注册测试

`add_test` 的第一个参数是测试名，`COMMAND` 后面是要执行的程序：

```cmake
add_test(NAME greet_basic
    COMMAND greet_tests --case basic)
```

需要定位构建产物时，使用生成器表达式：

```cmake
add_test(NAME app_smoke
    COMMAND $<TARGET_FILE:app>)
```

`add_test` 的 `COMMAND` 也可以直接写可执行 target 名，CMake 会解析成产物路径；显式写 `$<TARGET_FILE:app>` 更清楚，也不依赖上下文。

| 参数 | 作用 |
|------|------|
| `NAME` | 测试名，CTest 中唯一 |
| `COMMAND` | 要执行的命令与参数 |
| `WORKING_DIRECTORY` | 测试的工作目录 |
| `CONFIGURATIONS` | 只在指定构建配置下运行 |

## 2.3 运行测试

```bash
# 在构建目录中运行全部测试
ctest --test-dir build --output-on-failure

# 只运行名字匹配 greet 的测试，并行 8 个
ctest --test-dir build -R greet -j 8

# 只运行带 unit 标签的测试
ctest --test-dir build -L unit
```

| 选项 | 作用 |
|------|------|
| `--test-dir` | 指定构建目录，避免手动切换目录 |
| `--output-on-failure` | 失败时打印测试输出 |
| `-R` | 按正则筛选测试名 |
| `-L` | 按标签筛选 |
| `-j` | 并行执行测试 |
| `--timeout` | 为所有测试设置超时 |

## 2.4 测试属性与标签

```cmake
set_tests_properties(greet_basic PROPERTIES
    LABELS "unit;fast"
    TIMEOUT 10
    ENVIRONMENT "GREET_LANG=zh"
    PASS_REGULAR_EXPRESSION "Hello, CMake!")

set_tests_properties(app_smoke PROPERTIES
    LABELS "smoke"
    DEPENDS greet_basic)
```

| 属性 | 作用 |
|------|------|
| `LABELS` | 给测试打标签，配合 `ctest -L` 使用 |
| `TIMEOUT` | 单个测试的超时秒数 |
| `ENVIRONMENT` | 测试进程的环境变量 |
| `PASS_REGULAR_EXPRESSION` | 输出匹配才判定通过 |
| `WILL_FAIL` | 预期失败，用于负向测试 |

★ **关键认知**：`add_test` 只负责注册，不会在构建时自动运行。测试是独立阶段，CI 中应显式执行 `ctest`，并检查其退出码。

# 3. 接入测试框架

## 3.1 用 FetchContent 引入 Catch2

`tests/CMakeLists.txt`：

```cmake
include(FetchContent)

FetchContent_Declare(Catch2
    GIT_REPOSITORY https://github.com/catchorg/Catch2.git
    GIT_TAG        v3.5.2
    GIT_SHALLOW    TRUE)
FetchContent_MakeAvailable(Catch2)

add_executable(greet_tests test_greet.cpp)
target_link_libraries(greet_tests PRIVATE greet Catch2::Catch2WithMain)

# 让 CTest 自动发现 Catch2 中的 TEST_CASE
include(Catch)
catch_discover_tests(greet_tests)
```

`tests/test_greet.cpp`：

```cpp
#include <catch2/catch_test_macros.hpp>

#include "greet/greet.h"

TEST_CASE("greet 返回问候语", "[greet]") {
    REQUIRE(demo::greet("CMake") == "Hello, CMake!");  // 失败时打印表达式
}
```

`catch_discover_tests` 会在测试可执行文件构建后扫描测试用例，并为每个 `TEST_CASE` 生成一个 CTest 条目。

⚠ **关键认知**：测试可执行文件必须先能被构建，`catch_discover_tests` 才能发现用例。它通过构建后脚本完成注册，第一次配置时看不到具体测试名是正常的。GoogleTest 的接法类似，链接 `GTest::gtest_main` 后用 `gtest_discover_tests` 自动发现用例。

# 4. CPack 打包

## 4.1 基本配置

在根 `CMakeLists.txt` 末尾、所有 `install` 规则之后加入：

```cmake
set(CPACK_PACKAGE_NAME "greet")
set(CPACK_PACKAGE_VENDOR "Notes")
set(CPACK_PACKAGE_VERSION ${PROJECT_VERSION})
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "示例库")
set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE")
set(CPACK_GENERATOR "ZIP;TGZ")

include(CPack)   # 必须放在 install 规则之后
```

`include(CPack)` 会生成 `CPackConfig.cmake`，并添加 `package` 目标。

## 4.2 常用生成器

| 生成器 | 产物 | 平台 | 说明 |
|--------|------|------|------|
| `ZIP` | `.zip` | 全平台 | 简单归档，适合手动分发 |
| `TGZ` | `.tar.gz` | 全平台 | Linux 常见源码或二进制归档 |
| `DEB` | `.deb` | Linux | Debian/Ubuntu 安装包 |
| `RPM` | `.rpm` | Linux | Red Hat 系安装包 |
| `NSIS` | `.exe` 安装器 | Windows | 需要 NSIS 工具 |
| `DragNDrop` | `.dmg` | macOS | 拖拽式安装镜像 |

平台相关变量按需设置：

```cmake
set(CPACK_DEBIAN_PACKAGE_DEPENDS "libc6 (>= 2.31)")
set(CPACK_NSIS_DISPLAY_NAME "greet")
```

## 4.3 打包流程

```bash
# 先按发布配置构建并安装到打包暂存区
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
cpack --config build/CPackConfig.cmake -G ZIP

# 或者直接构建 package 目标
cmake --build build --target package
```

| 命令 | 作用 |
|------|------|
| `cpack --config build/CPackConfig.cmake` | 使用生成的配置打包 |
| `cpack -G TGZ` | 临时指定生成器 |
| `cmake --build build --target package` | 通过构建目标触发打包 |

★ **关键认知**：CPack 打包时会把 `install` 规则安装到临时暂存目录，再压缩成包。因此包内容与 `cmake --install` 的结果应当一致，任何安装缺失都会直接反映在包里。

# 5. CI 集成

## 5.1 GitHub Actions 示例

```yaml
name: build-and-test
on: [push, pull_request]
jobs:
  linux:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug -DBUILD_TESTING=ON
      - run: cmake --build build -j 4
      - run: ctest --test-dir build --output-on-failure
```

CI 中配置、构建、测试应分成独立步骤，便于定位失败阶段；`ctest` 的退出码会决定 job 是否失败。多平台用 `strategy.matrix` 展开，依赖下载目录可以缓存。

## 5.2 覆盖率

GCC/Clang 可以用 `--coverage` 同时处理编译与链接：

```cmake
option(ENABLE_COVERAGE "启用覆盖率统计" OFF)

if(ENABLE_COVERAGE)
    add_compile_options(--coverage)
    add_link_options(--coverage)
endif()
```

生成报告：

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug -DENABLE_COVERAGE=ON
cmake --build build
ctest --test-dir build --output-on-failure

lcov --capture --directory build --output-file coverage.info
genhtml coverage.info --output-directory coverage-report
```

`lcov` 负责采集并合并 `.gcda` 数据，`genhtml` 生成 HTML 报告，Codecov 或 Coveralls 可以把报告上传到 PR 中展示。

⚠ **关键认知**：覆盖率只说明代码被执行过，不说明断言有效。不要为了数字好看而堆砌无意义的测试，重点看关键路径和边界条件是否被覆盖。

# 6. 常见错误

| 现象 | 原因 | 修复 |
|------|------|------|
| `ctest` 显示 0 个测试 | 没有 `enable_testing()` 或 `include(CTest)` | 在根目录启用测试 |
| 子目录测试未注册 | 测试目录没有被 `add_subdirectory` | 检查目录加入顺序 |
| `add_test` 找不到程序 | 用了相对路径或错误产物名 | 用 `$<TARGET_FILE:目标>` |
| 测试在错误目录运行 | 依赖相对路径的资源 | 设置 `WORKING_DIRECTORY` |
| `catch_discover_tests` 未生效 | 没有 `include(Catch)` 或未构建测试目标 | 补上模块并先构建 |
| 打出的包是空的 | 没有 `install` 规则 | 先添加安装规则再打包 |
| CPack 变量不生效 | `include(CPack)` 放在 install 之前 | 移到所有安装规则之后 |
| CI 中测试卡死 | 存在交互式等待或死循环 | 设置 `TIMEOUT`，避免交互 |
| Windows 包缺少 DLL | 运行时依赖未安装 | 用 `install(TARGETS ... RUNTIME)` 或打包工具收集 |

排查测试问题时：

```bash
# 查看测试列表与详细输出
ctest --test-dir build -N
ctest --test-dir build -V
```

# 7. 动手练习

1. 给 `greet` 增加一个 Catch2 测试，覆盖正常输入和空字符串，并用 `ctest -L unit` 只运行它。
2. 为 `app` 注册一个冒烟测试，使用 `PASS_REGULAR_EXPRESSION` 校验输出，再故意改坏输出观察失败信息。
3. 用 CPack 生成 `ZIP` 与 `TGZ` 两种包，解压后检查目录结构是否与 `cmake --install` 一致。

# 8. 小结

| 概念 | 要点 |
|------|------|
| CTest | 统一注册、筛选、并行运行测试 |
| `include(CTest)` | 启用测试并定义 `BUILD_TESTING` |
| `add_test` | 注册测试命令与参数 |
| 测试属性 | 标签、超时、环境变量、输出匹配 |
| Catch2/GoogleTest | 用 `*_discover_tests` 自动映射用例 |
| CPack | 基于 `install` 规则生成分发包 |
| CI | 配置、构建、测试分步执行并检查退出码 |
| 覆盖率 | `--coverage` 加 `lcov`/`genhtml` |

下一篇进入最佳实践与总结，把前面七篇的规则收束成一套可执行的工程规范，并给出依赖管理、目录组织、排错和迁移的检查清单。
