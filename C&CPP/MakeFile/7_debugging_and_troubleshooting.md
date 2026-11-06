Makefile 的报错往往只有一行，而且经常指向"症状"而不是"病因"：`missing separator` 可能是空格，`No rule to make target` 可能是路径拼错，`up to date` 可能是时间戳或伪目标的问题。这一篇按"先观察、再定位、后修复"的顺序，把常用调试手段和高频故障串起来。

# 1. 先让 make 说话

## 1.1 `-n`：只打印，不执行

```bash
make -n
make -n clean
```

`-n`（`--just-print`）会展开所有变量、打印将要执行的命令，但不真正运行。它最适合回答："Make 到底打算做什么？"

```text
$ make -n
g++ -Iinclude -std=c++17 -Wall -MMD -MP -c src/main.cpp -o build/obj/debug/main.o
g++ -Iinclude -std=c++17 -Wall -MMD -MP -c src/util.cpp -o build/obj/debug/util.o
g++ build/obj/debug/main.o build/obj/debug/util.o -o build/bin/app
```

⚠ **关键认知**：`-n` 并不是绝对安全的"预演"。以 `+` 开头的命令、`$(MAKE)` 递归调用仍然会执行；`$(shell ...)` 在读取阶段就已经运行过。不要对包含删除、上传等副作用的 Makefile 盲目使用 `-n`。

## 1.2 `-p`：打印 Make 的数据库

```bash
make -p -f /dev/null        # 只看内置规则和变量
make -p 2>/dev/null > db.txt
```

`-p` 会输出 Make 解析后的全部内容：变量值、变量的来源、显式规则、模式规则、后缀规则。定位"某个变量为什么是这个值""哪条规则匹配上了"时非常有用。

```bash
# 查某个变量的最终值
make -p 2>/dev/null | grep '^CXXFLAGS'

# 查某个目标命中了哪条规则
make -p 2>/dev/null | grep -A3 '^build/obj/debug/main.o:'
```

在 Windows 上把 `/dev/null` 换成 `NUL`。

## 1.3 `--trace`：看清重建原因

GNU Make 4.0 起支持：

```bash
make --trace
```

输出会明确指出每个目标为什么被更新：

```text
Makefile:12: update target 'build/obj/debug/util.o' due to: src/util.cpp
g++ ... -c src/util.cpp -o build/obj/debug/util.o
```

这比盯着 `-n` 猜"为什么它又编译了"高效得多。

## 1.4 其他有用的开关

| 选项 | 作用 |
|------|------|
| `--debug=b` | 基本调试信息 |
| `--debug=v` | 详细，包含隐式规则搜索 |
| `--warn-undefined-variables` | 引用未定义变量时警告 |
| `-B` | 无条件重建所有目标 |
| `-W file` | 假设 `file` 已更新，观察传播 |
| `-o file` | 假设 `file` 很旧，不重建它 |
| `-q` | 不执行，只用退出码表示是否需要更新 |

`--debug=v` 的输出量很大，建议重定向到文件后再搜索：

```bash
make --debug=v > debug.log 2>&1
```

# 2. 调试变量

## 2.1 `$(info)`、`$(warning)`、`$(error)`

```makefile
$(info SRCS = $(SRCS))
$(warning CXXFLAGS 当前为 $(CXXFLAGS))
$(error 这里必须停止)
```

| 函数 | 输出 | 是否中断 |
|------|------|----------|
| `$(info ...)` | 标准输出 | 否 |
| `$(warning ...)` | 标准错误，带行号 | 否 |
| `$(error ...)` | 标准错误，带行号 | 是 |

它们都在**读取阶段**展开，所以放在文件顶层就能打印变量在那一刻的值。

```makefile
SRCS := $(wildcard src/*.cpp)
$(info 找到的源文件: $(SRCS))
```

如果 `SRCS` 为空，输出会直接暴露问题，而不是等到链接时报一堆 `undefined reference`。

## 2.2 看变量的"三层信息"

```makefile
$(info 值     = [$(CXX)])
$(info 来源   = $(origin CXX))
$(info 展开方式 = $(flavor CXX))
```

`origin` 的常见返回值：

| 返回值 | 含义 |
|--------|------|
| `undefined` | 未定义 |
| `default` | Make 内置默认值 |
| `environment` | 来自环境变量 |
| `file` | 在 Makefile 中定义 |
| `command line` | 由 `make VAR=...` 传入 |
| `override` | 用 `override` 定义 |
| `automatic` | 自动变量 |

`flavor` 返回 `recursive` 或 `simple`，正好对应 `=` 和 `:=`。

★ **关键认知**：变量调试要看三件事——**值、来源、展开方式**。只看值往往无法解释"为什么命令行传参没生效"；加上 `origin` 就能立刻看出是不是被 `command line` 或 `environment` 覆盖了。

## 2.3 看未展开的原始值

```makefile
A = $(B)
B = hello
$(info 展开值 = $(A))       # hello
$(info 原始值 = $(value A))  # $(B)
```

`$(value VAR)` 返回变量未经展开的内容，适合确认"变量里到底存了什么文本"。

## 2.4 在规则里打印

```makefile
$(OBJDIR)/%.o: $(SRCDIR)/%.cpp
	@echo "编译 $< -> $@"
	$(CXX) ...
```

规则里的 `echo` 在**执行阶段**运行，能看到自动变量的真实值。`@` 只是不回显命令本身，不影响命令执行。

# 3. 常见故障

## 3.1 `missing separator`

```text
Makefile:5: *** missing separator.  Stop.
```

九成是命令行用了空格。定位方法：

```bash
cat -A Makefile      # Tab 显示为 ^I
```

修复：

1. 把该行开头的空格替换为一个 Tab。
2. 编辑器设置该文件使用 Tab 缩进（`.editorconfig` 里写 `indent_style = tab`）。
3. 如果团队坚持用别的字符，GNU Make 3.82+ 可以改命令前缀：

```makefile
.RECIPEPREFIX = >
all:
> echo "用 > 代替 Tab"
```

⚠ **关键认知**：`.RECIPEPREFIX` 一旦设置，**整份 Makefile** 的命令行都要用新前缀，不能和 Tab 混用。它适合特殊场景，不建议作为常规方案。

## 3.2 `No rule to make target`

```text
make: *** No rule to make target 'build/obj/util.o', needed by 'app'.  Stop.
```

按顺序排查：

1. 文件是否真实存在，文件名大小写是否一致。
2. 模式规则的目录前缀是否写对（`%.o` 与 `build/%.o` 的词干不同）。
3. 是否有 `include` 的头文件被删除，且没有 `-MP`。
4. `VPATH` 或 `-I` 是否覆盖了源文件位置。

```bash
make --debug=v 2>&1 | grep -i "no rule\|trying"
```

## 3.3 循环依赖

```makefile
a: b
	@echo a

b: a
	@echo b
```

```text
make: Circular b <- a dependency dropped.
```

Make 会丢弃形成环的那条依赖并继续，但构建结果通常不正确。

排查思路：

1. 用 `make -p` 导出依赖图，搜索目标名。
2. 检查是否把"生成物"和"输入"写反了。
3. 检查目录级依赖：`build` 依赖 `src`，`src` 又依赖 `build` 这类错误。
4. 递归 make 中父子目录互相依赖，也容易形成环。

★ **关键认知**：循环依赖不是"Make 的小毛病"，而是**依赖图本身有环**。修复方式永远是改图——引入中间目标、拆掉不必要的依赖、或把"顺序"改成顺序依赖 `|`。

## 3.4 明明改了却不重编

| 可能原因 | 验证方式 | 修复 |
|----------|----------|------|
| 头文件没进依赖 | 看 `.d` 是否存在 | `-MMD -MP` + `-include` |
| 目标比依赖新 | `ls -l --time-style=full-iso` | `make clean` 后重建 |
| 目标是伪目标却写了文件 | 检查 `.PHONY` | 加 `.PHONY` 或改名 |
| 命令没执行但提示 up to date | `make --trace` | 看它认为哪个依赖更新 |

## 3.5 每次都全量重编

| 可能原因 | 验证方式 | 修复 |
|----------|----------|------|
| 目录被写成普通依赖 | 看规则里是否有 `\|` | 改成顺序依赖 |
| 依赖里包含总在变的文件 | `make --trace` | 移除或改成伪目标 |
| `$(shell)` 结果带时间戳 | 检查变量定义 | 用 `:=` 固定，或去掉 |
| 时间戳异常（未来时间） | `stat` 查看 | 修正系统时间或重新 checkout |

## 3.6 变量为空

把 `$CFLAGS` 写成 `$(CFLAGS)` 就能修复最常见的单字符展开问题。其他常见原因：

| 原因 | 例子 | 修复 |
|------|------|------|
| 单字符展开 | `$CFLAGS` | `$(CFLAGS)` |
| 定义顺序 | `A := $(B)` 在 `B` 之前 | 改用 `=` 或调整顺序 |
| 被命令行覆盖 | `make CFLAGS=` | 检查 `origin` |
| 函数参数空格 | `$(subst a, b,x)` | 去掉逗号后的空格 |
| 环境变量为空 | `export CFLAGS=` | `?=` 给默认值 |

## 3.7 并行下随机失败

```bash
make -j8      # 有时成功，有时失败
make -j1      # 一定成功
```

这说明**依赖图不完整**：某些任务实际上有先后关系，但没有写进规则。排查：

1. 用 `-j1` 确认串行稳定。
2. 逐步提高 `-j`，观察哪个目标开始竞争。
3. 检查多目标规则是否应该用 `&:`。
4. 检查是否有两个规则同时写同一个文件。

# 4. 一套可复用的排查流程

```text
记录报错 → make -n 看计划 → --trace 看原因 → make -p 查来源
        → 最小复现 → 修复 → 回归验证
```

★ **关键认知**：排错的核心是**缩小范围**。把一个几十条规则的 Makefile 删到只剩一条能复现问题的规则，问题往往自己就暴露了。不要一上来就盯着报错行改。

# 5. 高频错误速查

| 报错/现象 | 首要怀疑 | 快速验证 |
|-----------|----------|----------|
| `missing separator` | 空格代替 Tab | `cat -A Makefile` |
| `No rule to make target` | 文件名或路径错误 | `ls` 目标文件 |
| `Circular ... dependency dropped` | 依赖成环 | `make -p` 搜目标 |
| `recipe commences before first target` | 命令前没有规则头 | 检查上一行是否为空 |
| `target pattern contains no '%'` | 静态模式规则写错 | 检查 `目标: 模式: 依赖` |
| `warning: overriding recipe` | 同一目标定义了两次 | 搜索目标名 |
| `up to date` 但文件是旧的 | 时间戳或伪目标 | `make --trace` |
| 并行随机失败 | 依赖缺失 | `make -j1` 对比 |

# 6. 动手练习

1. 找一个能正常构建的 Makefile，故意制造三种错误：空格缩进、删除一个源文件、写一个循环依赖。分别用 `make -n`、`--trace`、`-p` 定位，并记录每种错误的最快识别信号。
2. 写一个变量 `A = $(B)`、`B` 后续被重新赋值，用 `$(info)`、`$(warning)`、`$(value)`、`$(origin)` 打印它在不同位置的值与来源。
3. 在一个多文件项目上分别执行 `make -j1` 和 `make -j8`，若结果不同，用 `--trace` 和依赖图找出缺失的依赖；若结果相同，尝试人为删除一条依赖再复现。

# 7. 小结

| 工具 | 回答的问题 |
|------|-----------|
| `make -n` | 打算执行哪些命令 |
| `make --trace` | 某个目标为什么被重建 |
| `make -p` | 变量从哪来、规则长什么样 |
| `$(info)` / `$(value)` / `$(origin)` | 变量此刻的值、原文、来源 |
| `make -j1` 对比 | 是否是并行竞争 |
| 最小复现 | 问题到底藏在哪条规则 |

下一篇收尾，讨论 Make 与 CMake 的取舍、可维护性与可移植性的具体做法，并给出一份可以直接照着执行的最佳实践清单。
