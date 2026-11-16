CMake 的变量系统看起来像脚本语言，实际却有一套自己的作用域规则；属性又常被误当成变量使用。很多"明明设置了却不生效"的问题，根源都在这里。本篇把变量、缓存、作用域、属性和生成器表达式串起来，帮你建立一张清晰的心智地图。

# 1. 变量系统

## 1.1 普通变量

普通变量用 `set` 创建，用 `${}` 读取：

```cmake
set(MY_NAME "demo")            # 字符串
set(MY_VERSION 1.2.3)          # 字符串，不是数字
set(MY_SOURCES a.cpp b.cpp)    # 分号分隔的列表

message(STATUS "name = ${MY_NAME}")
message(STATUS "sources = ${MY_SOURCES}")
```

`set` 的第二个参数之后如果给了多个值，会组成列表，列表在内部用分号连接：`set(SOURCES a.cpp b.cpp)` 展开后是 `a.cpp;b.cpp`。

★ **关键认知**：CMake 没有真正的数值类型，所有变量本质上都是字符串。列表只是用分号连接的字符串，因此路径里带分号、值里带空格时，必须用引号包住整个变量，例如 `"${PATH_WITH_SPACE}"`，否则会被拆成多个参数。

## 1.2 缓存变量

缓存变量写入 `CMakeCache.txt`，在多次配置之间保留，通常用于给用户提供开关：

```cmake
set(MY_FEATURE "ON" CACHE BOOL "是否启用实验特性")
set(MY_INSTALL_DIR "" CACHE PATH "自定义安装目录")
set(MY_TIMEOUT "30" CACHE STRING "超时时间（秒）")
```

也可以在命令行赋值：

```bash
cmake -S . -B build -DMY_FEATURE=OFF -DMY_TIMEOUT=60
```

`option` 是布尔缓存的简写：

```cmake
option(BUILD_TESTS "是否构建测试" ON)
```

| 写法 | 是否写缓存 | 适合场景 |
|------|------------|----------|
| `set(VAR value)` | 否 | 临时变量、局部逻辑 |
| `set(VAR value CACHE BOOL "doc")` | 是 | 用户可配置选项 |
| `option(VAR "doc" ON)` | 是 | 布尔开关 |
| `set(VAR value CACHE STRING "doc" FORCE)` | 是，强制覆盖 | 脚本中强制设定，慎用 |

⚠ **关键认知**：缓存变量一旦写入，后续 `set(... CACHE ...)` 不会覆盖已有值。要让新默认值生效，用户必须删缓存或显式传 `-D`；代码里用 `FORCE` 会覆盖用户选择，通常只应出现在工具链或 CI 脚本中。

## 1.3 环境变量

环境变量用 `$ENV{}` 读取：

```cmake
message(STATUS "PATH = $ENV{PATH}")
if(DEFINED ENV{CI})
    message(STATUS "正在 CI 环境构建")
endif()
```

⚠ **关键认知**：`set(ENV{...})` 只会影响当前 CMake 进程及其子进程，不会修改父 shell 的环境，也不会持久化。它通常只用于在配置阶段调用外部工具时临时传值，不要拿它当配置存储。

# 2. 作用域

## 2.1 目录作用域

每个 `add_subdirectory` 会创建新的目录作用域。子目录继承父目录变量的副本，修改不会自动回传：

```cmake
# 根 CMakeLists.txt
set(ROOT_FLAG "on")
add_subdirectory(sub)
message(STATUS "父目录 ROOT_FLAG = ${ROOT_FLAG}")  # 仍然是 on

# sub/CMakeLists.txt
set(ROOT_FLAG "off")  # 只改变子目录作用域内的副本
```

## 2.2 函数作用域与 PARENT_SCOPE

`function` 创建独立作用域，参数和局部变量都不会泄漏出去：

```cmake
function(print_info name)
    set(LOCAL_VALUE "只在本函数内可见")
    message(STATUS "name = ${name}, local = ${LOCAL_VALUE}")
endfunction()

print_info("demo")
message(STATUS "函数外 LOCAL_VALUE = ${LOCAL_VALUE}")  # 空值
```

需要向调用方返回结果时，用 `PARENT_SCOPE`：

```cmake
function(compute_output_dir out_var)
    set(result "${CMAKE_BINARY_DIR}/output")
    set(${out_var} "${result}" PARENT_SCOPE)  # 写到父作用域
endfunction()

compute_output_dir(MY_OUTPUT_DIR)
message(STATUS "输出目录 = ${MY_OUTPUT_DIR}")
```

`macro` 不创建新作用域，行为更像文本替换：

```cmake
macro(set_flag)
    set(MACRO_FLAG "on")  # 直接改调用方作用域
endmacro()
```

★ **关键认知**：需要局部隔离就用 `function`，需要"返回"值就用 `PARENT_SCOPE`。`macro` 容易污染调用方变量，除了极简单的封装，优先选 `function`。

## 2.3 include 不创建作用域

`include()` 只是把另一个文件的内容读进当前位置，**不创建作用域**。被 `include` 的文件可以直接读写调用方的变量。编写可复用模块时，最好用函数包裹，避免隐式依赖。

# 3. 属性

## 3.1 什么是属性

属性是挂在**对象**上的键值对。对象可以是 target、目录、源文件、测试或全局范围。属性不是变量，不能用 `${}` 读取：

```cmake
set_target_properties(app PROPERTIES
    CXX_STANDARD 17
    CXX_STANDARD_REQUIRED ON
    OUTPUT_NAME "demo")
```

读取属性用 `get_target_property`：

```cmake
get_target_property(app_output_name app OUTPUT_NAME)
message(STATUS "输出名 = ${app_output_name}")
```

`get_target_property` 的结果会写入一个普通变量，之后才可以用 `${}` 展开。

## 3.2 目录属性与 target 属性

| 属性对象 | 设置方式 | 常见用途 |
|----------|----------|----------|
| target | `set_target_properties` | 输出名、C++ 标准、可见性 |
| directory | `set_property(DIRECTORY ...)` | 目录级默认值 |
| source file | `set_source_files_properties` | 单文件编译选项 |
| global | `set_property(GLOBAL ...)` | 全局查询表 |

目录级属性用 `set_property(DIRECTORY PROPERTY CMAKE_CXX_STANDARD 17)` 设置，会影该目录后续创建的目标。

⚠ **关键认知**：属性只对"设置之后创建或处理"的目标稳定生效。跨目录依赖顺序去设置别人的属性，会让构建结果依赖 `add_subdirectory` 的书写顺序，属于典型的脆弱设计。

# 4. 头文件、编译选项与宏

## 4.1 target_include_directories

```cmake
target_include_directories(greet
    PUBLIC  ${CMAKE_CURRENT_SOURCE_DIR}/include
    PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/src)
```

对比全局命令：

| 命令 | 作用范围 | 是否推荐 |
|------|----------|----------|
| `include_directories` | 当前目录及之后所有目标 | 不推荐 |
| `target_include_directories` | 指定 target | 推荐 |

`SYSTEM` 可以把目录标记为系统头文件目录，从而抑制第三方头文件的警告，写法是在路径前加 `SYSTEM`。

## 4.2 target_compile_options

```cmake
target_compile_options(app PRIVATE
    $<$<CXX_COMPILER_ID:MSVC>:/W4>
    $<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-Wall>
    $<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-Wextra>)
```

不要把多个选项塞进一个字符串：

```cmake
# 错误：会被当成一个包含空格的参数
target_compile_options(app PRIVATE "-Wall -Wextra")

# 正确：一个选项一个参数
target_compile_options(app PRIVATE -Wall -Wextra)
```

## 4.3 target_compile_definitions

```cmake
target_compile_definitions(app PRIVATE
    APP_VERSION="1.0.0"       # 字符串宏需要转义引号
    FEATURE_LOG=1)           # 数值宏
```

它们在 C++ 中分别相当于 `#define APP_VERSION "1.0.0"` 和 `#define FEATURE_LOG 1`。

## 4.4 target_compile_features

```cmake
target_compile_features(app PRIVATE cxx_std_17)
```

这比手写 `-std=c++17` 更可移植，CMake 会根据编译器选择正确参数。`PRIVATE` 表示只要求本目标，`PUBLIC` 表示使用方也必须满足同样的标准。

# 5. 生成器表达式

## 5.1 基本语法

生成器表达式写成 `$<...>`，**在生成阶段求值**，而不是配置阶段。因此它可以感知构建配置、编译器、平台等信息。

```cmake
target_compile_definitions(app PRIVATE
    $<$<CONFIG:Debug>:APP_DEBUG=1>)
```

这段配置只有 Debug 构建才会定义 `APP_DEBUG`。

## 5.2 常用表达式

| 表达式 | 含义 |
|--------|------|
| `$<CONFIG>` | 当前配置名，如 Debug、Release |
| `$<CONFIG:Debug>` | 当前是否为 Debug，返回 0 或 1 |
| `$<PLATFORM_ID:Windows>` | 是否在 Windows 平台 |
| `$<CXX_COMPILER_ID:MSVC>` | 是否使用 MSVC |
| `$<COMPILE_LANGUAGE:CXX>` | 当前是否在编译 C++ 源文件 |
| `$<TARGET_FILE:app>` | `app` 产物的完整路径 |
| `$<BUILD_INTERFACE:...>` | 仅在构建树中使用该值 |
| `$<INSTALL_INTERFACE:...>` | 仅在安装后的包中使用该值 |

## 5.3 实战示例

```cmake
add_executable(app main.cpp)

# 按配置定义宏：Debug 打开日志，Release 关闭
target_compile_definitions(app PRIVATE
    $<$<CONFIG:Debug>:APP_DEBUG=1>
    $<$<CONFIG:Release>:NDEBUG=1>)

# 按平台定义宏；$<PLATFORM_ID:...> 在生成阶段判断平台
target_compile_definitions(app PRIVATE
    $<$<PLATFORM_ID:Windows>:APP_PLATFORM_WINDOWS=1>)

# 把可执行文件路径传给自定义命令
add_custom_command(TARGET app POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E echo "生成产物: $<TARGET_FILE:app>")
```

★ **关键认知**：生成器表达式的结果不是普通变量，**不能在 `if()` 里判断**。`if($<CONFIG:Debug>)` 不会按预期工作，因为 `if` 在配置阶段执行，此时 `$<...>` 还没有求值。

# 6. 常见错误

| 现象 | 原因 | 修复 |
|------|------|------|
| 变量在子目录修改后父目录没变 | 目录作用域隔离 | 用 `PARENT_SCOPE` 或缓存变量传回 |
| `set(... CACHE ...)` 改了默认值但不生效 | 缓存中已有旧值 | 删除 `CMakeCache.txt` 或传 `-D` |
| 路径含空格时命令被拆开 | 变量展开未加引号 | 用 `"${VAR}"` |
| `if(MY_VAR)` 判断结果异常 | 变量值是 `OFF`、`0`、空等假值 | 明确用 `STREQUAL` 或 `if(DEFINED ...)` |
| 生成器表达式在 `if` 中无效 | 求值阶段不同 | 改用 `if(CMAKE_BUILD_TYPE ...)` 或拆分配置 |
| 修改 `CXX_STANDARD` 后没有重新编译 | 编译器参数未变化 | 清理构建目录，确认属性设置位置 |
| 全局 `include_directories` 影响面过大 | 使用了目录级命令 | 换成 `target_include_directories` |

⚠ **关键认知**：调试变量问题时，先加 `message(STATUS "VAR=${VAR}")` 看展开后的真实值，再确认它所在的作用域。大多数"变量为空"不是没赋值，而是读错了层级。

# 7. 动手练习

1. 写一个 `function`，接收目标名，给目标添加 `-Wall` 与 Debug 宏，并解释为什么不能用普通 `set` 修改调用方变量。
2. 用 `option` 增加 `ENABLE_LOGGING`，通过 `target_compile_definitions` 和生成器表达式控制日志代码是否编译。
3. 把 `target_include_directories` 的 `PUBLIC` 改成 `PRIVATE`，在另一个目标里包含该头文件，记录报错并说明传播规则。

# 8. 小结

| 概念 | 要点 |
|------|------|
| 普通变量 | 字符串或分号列表，用 `${}` 读取 |
| 缓存变量 | 写入 `CMakeCache.txt`，跨配置保留 |
| 环境变量 | `$ENV{}` 读取，写入不持久化 |
| 目录作用域 | 子目录继承副本，修改不回传 |
| 函数作用域 | 独立作用域，用 `PARENT_SCOPE` 返回 |
| 属性 | 挂在 target/目录/文件上，需用 get 系列读取 |
| 生成器表达式 | 生成阶段求值，可感知配置与平台 |

下一篇进入依赖查找，讲清楚 `find_package` 的 Config 模式与 Find 模块模式、导入目标如何使用，以及 `FetchContent`、vcpkg、Conan 各自适合解决什么问题。
