CLI 的质量差距往往不在功能，而在失败时给人的感受：错误信息有没有说清原因、退出码能不能被脚本识别、日志会不会污染管道数据、非交互环境会不会卡住。这些都属于"输出契约"的一部分。本篇把标准输出、标准错误、退出码、日志和调试模式放到一起讲。

# 1. CLI 的三种反馈通道

## 1.1 stdout 与 stderr 的分工

命令行程序有两条输出通道，它们面向完全不同的消费者：

| 通道 | 文件描述符 | 面向 | 内容 |
|------|------------|------|------|
| 标准输出 | stdout（1） | 管道、下游程序 | 命令的结果数据 |
| 标准错误 | stderr（2） | 人、日志系统 | 错误、警告、进度、调试信息 |

```bash
notes list --json > notes.json   # 只有结果进文件
notes list --json 2> error.log   # 只有日志进文件
notes list --json | jq .         # 结果能被 jq 解析
```

★ **关键认知**：日志必须写 stderr。一旦日志混进 stdout，`notes list --json | jq` 就会因为多出几行日志而解析失败。判断标准很简单：**这段文字是"结果"还是"过程"**——结果走 stdout，过程走 stderr。

## 1.2 退出码的约定

退出码是 CLI 与 Shell、CI 之间的机器接口，比错误文本重要得多：

| 退出码 | 含义 | 典型场景 |
|--------|------|----------|
| 0 | 成功 | 命令正常完成 |
| 1 | 一般性失败 | 业务错误、未分类异常 |
| 2 | 用法错误 | 参数不合法（不少工具如此约定） |
| 126 | 无法执行 | 文件存在但无执行权限 |
| 127 | 命令不存在 | Shell 找不到可执行文件 |
| 130 | 被 SIGINT 中断 | 用户按下 Ctrl+C |

⚠ **关键认知**：只要命令失败，就必须返回非零退出码。很多脚本事故来自"打印了错误但退出码是 0"——CI 认为成功，部署继续进行。错误输出和退出码必须同时正确。

## 1.3 一次失败应该长什么样

好的错误信息通常包含三部分：**发生了什么、为什么、怎么办**。

```text
错误：无法读取配置文件 /home/alice/.notes/config.yaml
原因：文件不存在
提示：运行 notes init 生成默认配置，或用 --config 指定其他路径
```

而下面这种只把底层错误原样抛出，用户很难行动：

```text
open /home/alice/.notes/config.yaml: no such file or directory
```

# 2. 错误处理

## 2.1 用 RunE 传递错误

业务失败应该 `return error`，而不是在命令内部直接打印并退出：

```go
var showCmd = &cobra.Command{
	Use:  "show <id>",
	Args: cobra.ExactArgs(1),
	RunE: func(cmd *cobra.Command, args []string) error {
		note, err := store.Get(args[0])
		if err != nil {
			// 包装底层错误，补充"在读哪条笔记"这一层上下文
			return fmt.Errorf("读取笔记 %s 失败: %w", args[0], err)
		}
		return render(cmd.OutOrStdout(), note)
	},
}
```

★ **关键认知**：`RunE` 只负责"产生错误"，不负责"展示错误"。展示交给统一出口，这样才能保证所有命令的错误格式、输出位置和退出码一致。`%w` 保留错误链，让上层仍能用 `errors.Is` / `errors.As` 判断类型。

## 2.2 错误包装的层次

包装不是越多越好。每一层只补充这一层独有的信息：

```go
// 底层：具体操作失败
return fmt.Errorf("打开文件: %w", err)

// 中层：说明业务语义
return fmt.Errorf("加载笔记索引: %w", err)

// 顶层：给用户的行动建议
return fmt.Errorf("%w\n提示：运行 notes doctor 检查数据目录权限", err)
```

⚠ **关键认知**：不要在每一层都重复"失败了"。错误链的文本会拼成一句读不通的长句。包装时想清楚"这一层能补充什么新信息"，补充不了就原样向上返回。

## 2.3 不要用 panic 表达用户错误

```go
// 错误示范：参数错误直接 panic
if len(args) == 0 {
	panic("缺少参数") // 用户会看到堆栈，退出码也不是业务语义
}

// 正确做法：返回可读错误
if len(args) == 0 {
	return errors.New("缺少参数：请提供笔记 ID")
}
```

`panic` 只适合"程序内部不变量被破坏"这类真正的 bug。用户输入错误、文件不存在、网络超时都是预期内的情况，应当走 error。

## 2.4 自定义退出码

需要区分错误类型时，定义一个携带退出码的错误类型：

```go
type ExitError struct {
	Code int
	Err  error
}

func (e *ExitError) Error() string { return e.Err.Error() }
func (e *ExitError) Unwrap() error { return e.Err }

// 用法：配置错误返回 2，便于脚本区分
return &ExitError{Code: 2, Err: fmt.Errorf("配置无效: %w", err)}
```

## 2.5 统一错误出口

把 `Execute` 的返回值集中处理，是 CLI 的标准结构：

```go
func main() {
	os.Exit(run()) // main 只做一件事：把退出码交给系统
}

func run() int {
	rootCmd.SilenceErrors = true // 阻止 Cobra 自己打印，避免重复
	rootCmd.SilenceUsage = true  // 业务错误时不要甩一大段用法

	if err := rootCmd.Execute(); err != nil {
		fmt.Fprintln(os.Stderr, "错误：", err)

		var exitErr *ExitError
		if errors.As(err, &exitErr) {
			return exitErr.Code
		}
		return 1
	}
	return 0
}
```

流程如下：

```text
RunE 返回 error
      │
      ▼
Execute() 返回给 run()
      │
      ├── errors.As 命中 *ExitError → 使用自定义码
      └── 其他错误 → 返回 1
      │
      ▼
main: os.Exit(code)
```

★ **关键认知**：不要在 `RunE` 里调用 `os.Exit`。它会立刻终止进程，跳过所有 `defer`——文件没关、临时目录没删、事务没回滚。让错误一路返回到 `main`，由最外层决定退出，资源清理才有机会执行。

# 3. 日志

## 3.1 用 log/slog 起步

Go 1.21 起标准库自带结构化日志 `log/slog`，CLI 项目不必急着引入第三方库：

```go
import "log/slog"

logger := slog.New(slog.NewTextHandler(os.Stderr, &slog.HandlerOptions{
	Level: slog.LevelInfo, // 默认只输出 Info 及以上
}))
logger.Info("开始同步", "source", "local", "count", 42)
logger.Warn("配置缺失，使用默认值", "key", "output")
logger.Error("同步失败", "err", err)
```

## 3.2 日志级别怎么选

| 级别 | 用途 | 例子 |
|------|------|------|
| Debug | 排查问题时的细节 | 请求参数、配置最终值、耗时 |
| Info | 正常但值得记录的事件 | 同步完成、生成文件 |
| Warn | 可继续但有风险 | 配置缺失、降级处理 |
| Error | 本次操作失败 | 写入失败、远端不可达 |

★ **关键认知**：日志是给"运行者"看的，命令结果是给"调用者"看的。`notes list` 的结果不应该用 `logger.Info` 输出，否则机器无法消费；反过来，调试细节也不该走 `cmd.Println`，否则会污染管道。

## 3.3 调试模式

调试模式的目标是"默认安静，需要时详细"。常见做法是根命令上的持久 flag：

```go
var debug bool

func init() {
	rootCmd.PersistentFlags().BoolVar(&debug, "debug", false, "输出调试日志")
	rootCmd.PersistentPreRun = func(cmd *cobra.Command, args []string) {
		if debug {
			slog.SetLogLoggerLevel(slog.LevelDebug) // 全局默认 logger 也放开
		}
	}
}
```

更稳妥的做法是自己构造 logger 并注入命令，而不是依赖全局：

```go
func newLogger(debug bool) *slog.Logger {
	level := slog.LevelInfo
	if debug {
		level = slog.LevelDebug
	}
	return slog.New(slog.NewTextHandler(os.Stderr, &slog.HandlerOptions{
		Level:     level,
		AddSource: debug, // 调试时附上文件与行号
	}))
}
```

⚠ **关键认知**：调试日志经常包含完整请求体、配置文件内容、环境变量。上线前必须确认敏感字段（token、密码、cookie）不会被打出来。宁可少打一条，也不要把密钥写进日志。

# 4. 用户提示与交互

## 4.1 交互式确认

危险操作（删除、覆盖、推送）应当先确认：

```go
func confirm(prompt string) (bool, error) {
	fmt.Fprintf(os.Stderr, "%s [y/N]: ", prompt) // 提示走 stderr，不污染结果
	reader := bufio.NewReader(os.Stdin)
	line, err := reader.ReadString('\n')
	if err != nil {
		return false, fmt.Errorf("读取确认失败: %w", err)
	}
	answer := strings.ToLower(strings.TrimSpace(line))
	return answer == "y" || answer == "yes", nil
}
```

## 4.2 非交互环境要能跳过

CI 里没有 TTY，等待输入会直接卡死。正确做法是提供 `--yes`，并且在检测不到终端时拒绝交互：

```go
import "golang.org/x/term"

func isInteractive() bool {
	return term.IsTerminal(int(os.Stdin.Fd()))
}

if !isInteractive() && !yesFlag {
	return errors.New("非交互环境请显式传入 --yes")
}
```

★ **关键认知**：任何需要人工输入的命令，都必须提供非交互路径。否则它只能手动执行，无法进入脚本和流水线，这是 CLI 可自动化能力的底线。

# 5. 常见坑

| 现象 | 原因 | 处理 |
|------|------|------|
| `jq` 解析失败 | 日志打到了 stdout | 日志统一走 stderr |
| CI 显示成功但实际失败 | 失败时仍返回 0 | 所有错误路径返回非零码 |
| 错误打印两遍 | Cobra 自动输出 + main 再输出 | 设置 `SilenceErrors` |
| 临时文件残留 | 在 `RunE` 里 `os.Exit` | 返回 error，让 main 退出 |
| 错误信息看不懂 | 只抛底层错误 | 补充上下文与行动建议 |
| CI 卡住不结束 | 无 TTY 仍在等待输入 | 提供 `--yes` 并检测终端 |
| 调试日志泄露密钥 | 未脱敏 | 打印前过滤敏感字段 |

# 6. 动手练习

1. 给 `notes show` 增加一个"笔记不存在"的错误，要求提示里包含笔记 ID 和 `notes list` 的建议，并返回退出码 1。
2. 用 `slog` 为 `notes sync` 增加 `--debug` 开关，默认只输出 Info，开启后输出每个文件的处理耗时。
3. 让删除命令在非交互环境下必须带 `--yes`，并用 `echo $?` 验证成功与失败的退出码。

# 7. 小结

| 主题 | 要点 |
|------|------|
| 输出分工 | 结果走 stdout，日志与错误走 stderr |
| 退出码 | 成功 0，失败非 0，取消 130 |
| 错误传递 | `RunE` 返回 error，`main` 统一处理 |
| 错误文本 | 发生了什么、为什么、怎么办 |
| 日志 | 结构化、分级、默认安静、调试可开 |
| 交互 | 必须提供 `--yes` 等非交互路径 |
| 退出方式 | 不在业务层 `os.Exit`，保留 defer 清理 |

下一篇进入打包与分发，讲清楚交叉编译怎么配、版本号如何注入二进制、发布产物怎么组织，以及 Homebrew、Docker 和安装脚本各自的适用场景。
