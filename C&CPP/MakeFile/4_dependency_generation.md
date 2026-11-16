前面几篇的依赖只写了 `.cpp`，但真正决定一个 `.o` 内容的还有它包含的所有头文件。改了 `util.h` 却不重编 `util.o`，是 C/C++ 项目里最阴险的一类 bug：编译通过、行为却是旧的。让编译器自己输出依赖，是解决这个问题的标准做法。

# 1. 头文件依赖为什么会被漏掉

## 1.1 一个改动不触发重编的例子

```makefile
app: main.o util.o
	$(CXX) $^ -o $@

%.o: %.cpp
	$(CXX) $(CXXFLAGS) -c $< -o $@
```

```cpp
// util.cpp
#include "util.h"

int add(int a, int b) {
    return a + b;
}
```

现在把 `util.h` 里的函数签名从 `int add(int, int)` 改成 `long add(long, long)`，然后执行 `make`：

```text
make: 'app' is up to date.
```

Make 只看规则里写的东西：`util.o` 依赖 `util.cpp`，而 `util.cpp` 没变，所以它认为 `util.o` 是新的。`util.h` 根本不在依赖图里。

★ **关键认知**：Make 不会读源代码，它**不知道 `#include` 的存在**。依赖关系必须由外部提供，要么手写，要么让编译器分析后写进文件。这是第 1 篇"依赖驱动"模型的直接推论。

## 1.2 手工维护为什么不可行

理论上可以这样写：

```makefile
main.o: main.cpp config.h util.h
util.o: util.cpp util.h logger.h
```

但头文件会层层包含：`util.h` 可能包含 `config.h`，`config.h` 又包含 `platform.h`。一个中等规模项目里，手工维护的依赖列表几乎必然过期，而且没人愿意在每次加 `#include` 时同步改 Makefile。

# 2. 让编译器生成依赖

## 2.1 `-M` 系列选项

GCC/Clang 在预处理阶段本来就知道每个源文件包含了哪些头文件，所以可以直接把依赖输出成 Makefile 语法。

| 选项 | 输出位置 | 是否包含系统头文件 |
|------|----------|--------------------|
| `-M` | 标准输出 | 包含 |
| `-MM` | 标准输出 | 不包含 |
| `-MD` | 副作用写入 `.d` 文件 | 包含 |
| `-MMD` | 副作用写入 `.d` 文件 | 不包含 |

实际项目几乎都用 `-MMD`：系统头文件（`/usr/include` 下的）极少变化，把它们排除可以减少无谓的重编。

```bash
# 只生成依赖，不产生目标文件
g++ -MM main.cpp
```

输出：

```text
main.o: main.cpp util.h config.h
```

## 2.2 编译时顺手生成 `.d`

单独跑一遍 `-MM` 意味着预处理两次。`-MMD` 可以在正常编译的同时把依赖写到 `.d` 文件：

```bash
g++ -std=c++17 -MMD -MP -c util.cpp -o util.o
```

执行后目录里多出 `util.d`，内容类似：

```text
util.o: util.cpp util.h config.h platform.h

util.h:
config.h:
platform.h:
```

第一行是和 `.o` 对应的依赖关系，后面的空规则是 `-MP` 加的。

## 2.3 `-MP` 解决头文件被删的问题

假设 `config.h` 被删除或重命名，而某个 `.d` 文件里还留着：

```text
util.o: util.cpp util.h config.h
```

下次 `make` 会报：

```text
make: *** No rule to make target 'config.h', needed by 'util.o'.  Stop.
```

加上 `-MP` 后，编译器会为每个头文件额外生成一个空规则：

```text
config.h:
```

这告诉 Make："这个文件就算不存在，也不要报错。"这样删除头文件后，Make 只会重编受影响的文件，然后在编译阶段给出真正的错误信息。

⚠ **关键认知**：`-MP` 不是在"掩盖错误"，而是把错误从"Make 找不到文件"推迟到"编译器找不到头文件"。后者信息更准确，也更符合实际意图。工程里应当始终同时使用 `-MMD -MP`。

# 3. 在 Makefile 中接入

## 3.1 `include` 与 `-include`

```makefile
include $(DEPS)    # 文件不存在会报错
-include $(DEPS)   # 文件不存在则忽略
```

第一次构建时 `.d` 文件还不存在，必须用 `-include`。这也是几乎所有 C/C++ Makefile 模板里都写着 `-include` 的原因。

```text
第一次 make：
  .d 不存在 → -include 忽略 → 编译 → 生成 .o 和 .d
第二次 make：
  读到 .d → 头文件进入依赖图 → 改 .h 能触发重编
```

★ **关键认知**：`.d` 文件是**构建产物**，不是手写文件。它由编译器生成，由 Make 读取，`clean` 时要和 `.o` 一起删除。把它提交进 Git 是常见的错误做法。

## 3.2 完整示例

```makefile
CXX      := g++
CXXFLAGS := -std=c++17 -Wall -Wextra -MMD -MP
SRCS     := main.cpp util.cpp order.cpp
OBJS     := $(SRCS:.cpp=.o)
DEPS     := $(OBJS:.o=.d)

app: $(OBJS)
	$(CXX) $^ -o $@

%.o: %.cpp
	$(CXX) $(CXXFLAGS) -c $< -o $@

-include $(DEPS)

.PHONY: clean
clean:
	rm -f $(OBJS) $(DEPS) app
```

验证方式：

```bash
make            # 全量编译，生成 .o 和 .d
touch util.h    # 只改头文件的时间戳
make            # 只有 util.o（以及依赖它的目标）被重编
```

## 3.3 产物放到 `build` 目录

```makefile
BUILD    := build
SRCS     := main.cpp util.cpp order.cpp
OBJS     := $(addprefix $(BUILD)/,$(SRCS:.cpp=.o))
DEPS     := $(OBJS:.o=.d)

app: $(OBJS)
	$(CXX) $^ -o $@

$(BUILD)/%.o: %.cpp
	@mkdir -p $(@D)
	$(CXX) $(CXXFLAGS) -c $< -o $@

-include $(DEPS)
```

两个细节：

1. `mkdir -p $(@D)` 保证 `build/` 存在；`$(@D)` 是目标所在目录。
2. `-MMD` 默认把 `.d` 放在目标文件旁边，所以 `build/main.o` 对应 `build/main.d`，`DEPS` 的路径自然对齐。

如果编译器版本较老，或者想完全控制位置，可以显式指定：

```makefile
$(BUILD)/%.o: %.cpp
	@mkdir -p $(@D)
	$(CXX) $(CXXFLAGS) -MMD -MF $(@:.o=.d) -c $< -o $@
```

`-MF` 指定依赖文件路径，`$(@:.o=.d)` 把目标名的 `.o` 换成 `.d`。

# 4. 增量编译的边界

## 4.1 会触发重编的变化

| 变化 | 是否重编 | 说明 |
|------|----------|------|
| 修改 `.cpp` | 是 | 直接依赖 |
| 修改被包含的 `.h` | 是 | `.d` 记录的依赖 |
| 新增 `#include` | 是 | 下次编译后 `.d` 更新 |
| 删除头文件 | 是 | 配合 `-MP` 正常报错 |
| 修改编译选项 | **否** | Make 不跟踪命令行变化 |

## 4.2 不会触发重编的变化

Make 的模型里没有"配置哈希"这个概念。以下变化都不会自动触发重编：

- 修改 `CXXFLAGS`（例如从 `-O2` 改成 `-O0`）。
- 修改 `-I` 包含路径。
- 升级编译器版本。
- 修改 `#define` 的值（除非它所在的头文件被改）。

想要覆盖这些场景，标准做法是先 `make clean` 再全量构建，或者用能跟踪配置的工具（如 CMake、Ninja 的依赖文件）。

⚠ **关键认知**：`make` 的正确性建立在"依赖图 + 时间戳"上。凡是**不影响依赖图但影响产物**的变化，它都看不见。发布构建和调试构建之间切换时，务必清理，或者把不同配置输出到不同目录。

# 5. 常见错误

| 现象 | 原因 | 修复 |
|------|------|------|
| 改 `.h` 不重编 | 没有生成/引入 `.d` | 加 `-MMD -MP` 和 `-include` |
| 首次构建报 `No rule to make target 'xxx.d'` | 用了 `include` 而非 `-include` | 改成 `-include` |
| 删除头文件后报错 | 没加 `-MP` | 加上 `-MP` |
| `.d` 被提交进仓库 | 把构建产物当源码 | 加入 `.gitignore` |
| `clean` 后残留 `.d` | 清理列表没包含 `DEPS` | `rm -f $(OBJS) $(DEPS)` |
| 换编译选项后行为不一致 | Make 不跟踪选项变化 | 清理后重建，或分目录构建 |

# 6. 动手练习

1. 给第 1 篇的 Makefile 加上 `-MMD -MP` 和 `-include $(DEPS)`，用 `touch util.h` 验证只重编相关目标，并打开 `util.d` 观察内容。
2. 故意删掉一个被包含的头文件，分别在**有** `-MP` 和**没有** `-MP` 的情况下执行 `make`，对比报错信息。
3. 把产物输出到 `build/` 目录，要求 `.o` 和 `.d` 都在 `build/` 下，`make clean` 能一次清理干净，并用 `find build` 或 `Get-ChildItem` 核对。

# 7. 小结

| 主题 | 要点 |
|------|------|
| 问题本质 | Make 不知道 `#include`，依赖要外部提供 |
| `-MMD` | 编译时顺手生成 `.d`，排除系统头 |
| `-MP` | 为头文件加空规则，删除时不误报 |
| `-include` | 首次构建 `.d` 不存在时忽略 |
| 依赖链 | `.d` 把每个头文件接入对应 `.o` 的依赖 |
| 边界 | 编译选项、编译器版本变化不会触发重编 |

下一篇进入伪目标与并行构建，讲清 `.PHONY` 为什么必须加、`make -j` 如何利用依赖图并行、顺序依赖 `|` 的用途，以及命令失败、`-` 前缀和 `.DELETE_ON_ERROR` 的行为。
