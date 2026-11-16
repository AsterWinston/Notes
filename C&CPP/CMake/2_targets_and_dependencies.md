初学 CMake 最常见的误区，是把源文件、头文件路径和链接库当成一堆全局开关来配置。现代 CMake 的核心思路完全不同：**一切围绕 target 组织**，谁需要什么，就挂在谁身上。本篇讲清楚 target 的创建、链接，以及 `PUBLIC`、`PRIVATE`、`INTERFACE` 这三个关键字的真实含义。

# 1. 为什么需要 target

## 1.1 从文件列表到构建单元

旧式写法习惯维护全局变量：

```cmake
# 旧式写法：全局收集源文件，再加全局头文件路径
include_directories(include)
file(GLOB SOURCES "src/*.cpp")
add_executable(app ${SOURCES})
```

问题在于：`include_directories` 会影响之后所有目标，`file(GLOB)` 不会感知新增文件，依赖关系也藏在字符串里。

现代写法把每个产物声明成一个 target：

```cmake
# 现代写法：每个产物是一个 target，用法要求挂在 target 上
add_library(greet STATIC src/greet.cpp)
target_include_directories(greet PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)

add_executable(app app/main.cpp)
target_link_libraries(app PRIVATE greet)
```

★ **关键认知**：target 是 CMake 的**一等公民**。源文件、头文件搜索路径、编译选项、宏定义、链接依赖，全部可以作为 target 的属性或用法要求来管理，因此不再需要全局状态。

## 1.2 target 的三种来源

| 来源 | 创建方式 | 典型用途 |
|------|----------|----------|
| 本项目构建的 target | `add_executable`、`add_library` | 自己的可执行文件与库 |
| 外部导入的 target | `find_package`、`add_library(... IMPORTED)` | 系统库、第三方包 |
| 别名 target | `add_library(ns::name ALIAS real)` | 统一命名空间，便于消费 |

有了 target，链接时写的是**目标名**，不是文件路径。路径由 CMake 在生成阶段解析，链接顺序和传递依赖也由它负责。

# 2. 创建 target

## 2.1 add_executable

```cmake
add_executable(app
    app/main.cpp
    app/options.cpp)
```

第一个参数是目标名，后面是源文件。目标名在同一个项目中必须唯一，需要改输出名时用 `set_target_properties(app PROPERTIES OUTPUT_NAME "targets-demo")`。

## 2.2 add_library

```cmake
add_library(greet STATIC
    src/greet.cpp
    include/greet/greet.h)   # 头文件不是必需，但列出来便于 IDE 展示
```

不写类型时由 `BUILD_SHARED_LIBS` 决定：

```cmake
# 默认构建静态库；配置时传 -DBUILD_SHARED_LIBS=ON 可改为动态库
add_library(greet src/greet.cpp)
```

## 2.3 库类型选择

| 类型 | 关键字 | 产物 | 特点 |
|------|--------|------|------|
| 静态库 | `STATIC` | `.a` / `.lib` | 链接进可执行文件，部署简单 |
| 动态库 | `SHARED` | `.so` / `.dll` / `.dylib` | 运行时加载，可共享，需处理导出 |
| 模块库 | `MODULE` | `.so` / `.dll` | 运行时插件，通常不被直接链接 |
| 对象库 | `OBJECT` | 一组 `.o` | 只编译不归档，适合复用编译结果 |
| 接口库 | `INTERFACE` | 无产物 | 只承载用法要求，典型是 header-only |
| 导入库 | `IMPORTED` | 已存在的二进制 | 描述外部库，不参与构建 |

★ **关键认知**：`INTERFACE` 库没有源文件，也不能被编译。它的价值是**打包一组用法要求**，例如头文件路径、宏定义和依赖，让使用方链接它就能获得全部配置。

# 3. 链接 target

## 3.1 target_link_libraries

```cmake
target_link_libraries(app PRIVATE greet)
```

现代签名必须带 `PUBLIC`、`PRIVATE` 或 `INTERFACE` 之一。它会做两件事：

1. 把库加入 `app` 的链接命令行。
2. 把库的**传递用法要求**（头文件路径、编译定义、依赖库）传给 `app`。

## 3.2 链接顺序与传递依赖

用 target 链接时，CMake 会自动处理依赖顺序。假设：

```text
app ──► greet ──► fmt
```

只需要写：

```cmake
target_link_libraries(app PRIVATE greet)
```

`fmt` 会随着 `greet` 的 `PUBLIC` 或 `INTERFACE` 依赖自动出现在 `app` 的链接线上。

```text
最终链接顺序（简化）：
app 的对象文件 ──► libgreet.a ──► libfmt.a
```

⚠ **关键认知**：只有在直接写库文件路径、绕过 target 时，才需要手动关心 `-l` 的先后顺序。这也是"链接 target 而不是路径"的重要理由。

# 4. PUBLIC、PRIVATE、INTERFACE

## 4.1 三个关键字含义

这三个关键字描述的是**用法要求的作用范围**：

| 关键字 | 作用于当前 target | 传播给使用方 | 典型场景 |
|--------|-------------------|--------------|----------|
| `PRIVATE` | 是 | 否 | 只在实现里用到的头文件、宏、库 |
| `PUBLIC` | 是 | 是 | 公开头文件里出现的依赖 |
| `INTERFACE` | 否 | 是 | header-only 库、纯接口目标 |

以头文件路径为例：

```cmake
# 公开头文件在 include/ 下，使用方也需要这个路径
target_include_directories(greet PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)

# 只在 src/ 内部使用的私有头文件，不对外暴露
target_include_directories(greet PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/src)
```

## 4.2 传播规则

```text
greet 的 PUBLIC 用法要求
        │
        ├──► 参与 greet 自身编译
        │
        └──► 传给链接 greet 的 app

greet 的 PRIVATE 用法要求
        │
        └──► 只参与 greet 自身编译，app 看不到

greet 的 INTERFACE 用法要求
        │
        └──► 不参与 greet 自身编译，只传给 app
```

★ **关键认知**：判断用哪个关键字，只看一个问题——**使用方编译时需不需要它？** 需要就是 `PUBLIC`，不需要就是 `PRIVATE`；如果当前 target 自己根本不编译（接口库），那就只能用 `INTERFACE`。反例：把头文件路径写成 `PRIVATE`，但公开头文件 `#include <greet/greet.h>`，使用方编译时会报 `fatal error: greet/greet.h: No such file or directory`。

# 5. 完整示例

## 5.1 目录结构

```text
targets-demo/
├── CMakeLists.txt
├── app/
│   ├── CMakeLists.txt
│   └── main.cpp
├── greet/
│   ├── CMakeLists.txt
│   ├── include/greet/greet.h
│   └── src/greet.cpp
```

## 5.2 根 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.20)
project(targets_demo VERSION 1.0.0 LANGUAGES CXX)

# 依次加入子目录，每个子目录维护自己的 target
add_subdirectory(greet)
add_subdirectory(app)
```

## 5.3 库的 CMakeLists.txt

`greet/CMakeLists.txt`：

```cmake
add_library(greet STATIC
    src/greet.cpp
    include/greet/greet.h)

# 公开头文件目录，使用方也需要，所以是 PUBLIC
target_include_directories(greet
    PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)

# C++17 是公开接口要求，使用方同样需要
target_compile_features(greet PUBLIC cxx_std_17)
```

`greet/greet.h`：

```cpp
#pragma once

#include <string>

namespace demo {
std::string greet(const std::string& name);
}
```

`greet/src/greet.cpp`：

```cpp
#include "greet/greet.h"

namespace demo {
std::string greet(const std::string& name) {
    return "Hello, " + name + "!"; // 简单拼接，便于验证链接结果
}
}
```

## 5.4 可执行文件的 CMakeLists.txt

`app/CMakeLists.txt`：

```cmake
add_executable(app main.cpp)

# PRIVATE：greet 只在这里使用，不需要再传给别的目标
target_link_libraries(app PRIVATE greet)
```

`app/main.cpp`：

```cpp
#include <iostream>

#include "greet/greet.h"

int main() {
    std::cout << demo::greet("CMake") << std::endl;  // 验证库函数可用
    return 0;
}
```

构建与运行：

```bash
cmake -S . -B build
cmake --build build
./build/app/app
```

⚠ **关键认知**：`app` 的 `CMakeLists.txt` 没有写任何 `include` 路径。`greet` 和 `math` 的 `PUBLIC` 头文件路径已经随 `target_link_libraries` 传播过来，这正是 target 模型的价值。

# 6. 常见错误

| 现象 | 原因 | 修复 |
|------|------|------|
| `Cannot specify link libraries for target "app" which is not built` | `target_link_libraries` 写在 `add_executable` 之前 | 先创建 target，再设置属性 |
| `The plain signature for target_link_libraries has already been used` | 混用无关键字与带关键字签名 | 全部改成带 `PUBLIC`/`PRIVATE`/`INTERFACE` 的形式 |
| `fatal error: xxx.h: No such file or directory` | 头文件路径写成 `PRIVATE`，但使用方需要 | 改成 `PUBLIC`，或让使用方自己声明 |
| `undefined reference to ...` | 只声明了库却没链接，或链接顺序错误 | 用 target 链接，不要手拼 `.a` 路径 |
| Windows 下动态库报找不到符号 | 没有导出符号 | 使用 `__declspec(dllexport)` 或生成导出头 |
| 两个库互相依赖 | 存在循环依赖 | 拆分公共部分，或改用接口库 |

⚠ **关键认知**：`target_link_libraries` 的普通签名和关键字签名不能混用。一个项目里统一使用关键字形式，能避免大量难以理解的报错。

# 7. 动手练习

1. 把 `greet` 改成 `INTERFACE` 库，把实现移到 `app` 里，观察 `target_include_directories` 应该用哪个关键字。
2. 新增一个 `math` 静态库，把 `target_link_libraries(greet PUBLIC math)` 与 `PRIVATE math` 各试一次，记录 `app` 编译结果的差异。
3. 创建一个 header-only 库，只提供 `INTERFACE` 头文件路径，并让 `app` 链接它。

# 8. 小结

| 概念 | 要点 |
|------|------|
| target | 构建单元，承载源文件、属性和依赖 |
| 库类型 | `STATIC`、`SHARED`、`MODULE`、`OBJECT`、`INTERFACE`、`IMPORTED` |
| 链接 | 链接目标名，不链接文件路径 |
| `PRIVATE` | 只用于自身编译 |
| `PUBLIC` | 自身与使用方都需要 |
| `INTERFACE` | 只传给使用方，自身不编译 |
| 判断方法 | 使用方编译时是否需要 |

下一篇进入变量与属性，讲清楚普通变量、缓存变量、目录作用域和函数作用域的区别，以及生成器表达式如何在不同配置下给出不同结果。
