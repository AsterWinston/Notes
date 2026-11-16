CLI 的测试比 Web 服务更容易被忽视：反正手动敲一遍就知道了。但命令一多、参数组合一复杂，"手动敲一遍"就会变成漏测的借口。好在 CLI 的输入输出都是纯文本，只要把边界设计好，测试反而比 HTTP 更简单。本篇讲命令层怎么测、集成测试怎么跑，以及一套可长期维护的 CLI 设计习惯。

# 1. 为什么 CLI 难测

## 1.1 三个耦合点

难以测试的 CLI 通常把逻辑绑死在三处：

| 耦合点 | 典型写法 | 后果 |
|--------|----------|------|
| 进程参数 | 直接在 `RunE` 里读 `os.Args` | 测试无法构造输入 |
| 全局输出 | `fmt.Println` 写死 stdout | 无法断言输出，也无法静默 |
| 进程退出 | 业务里直接 `os.Exit(1)` | 测试进程被终止，`defer` 不执行 |
| 全局配置 | 包级 `viper.GetString` | 测试之间互相污染 |
| 真实时间 | `time.Now()` 直接用 | 断言不稳定 |

★ **关键认知**：可测试性的本质是**把"进程级副作用"收敛到最外层**。命令内部只接收参数、写传入的 `io.Writer`、返回 `error`；`os.Args`、`os.Exit`、真实时钟这些属于进程边界的东西，只在 `main` 里出现。

## 1.2 可测试的入口

把程序改造成"一个返回退出码的函数"：

```go
func main() {
	os.Exit(run(os.Args[1:], os.Stdin, os.Stdout, os.Stderr))
}

func run(args []string, stdin io.Reader, stdout, stderr io.Writer) int {
	root := NewRootCmd(&App{In: stdin, Out: stdout, Err: stderr})
	root.SetArgs(args)

	if err := root.Execute(); err != nil {
		fmt.Fprintln(stderr, "错误：", err)
		return 1
	}
	return 0
}
```

这样测试可以直接调用 `run`，传入缓冲区，断言输出与退出码，完全不需要启动子进程。

# 2. 命令层单元测试

## 2.1 用构造函数生成命令树

不要复用包级全局 `rootCmd`。全局命令的 flag 值会在多个测试之间残留，导致"单独跑通过、一起跑失败"：

```go
type App struct {
	In    io.Reader
	Out   io.Writer
	Err   io.Writer
	Store NoteStore // 接口，测试时可替换为内存实现
}

func NewRootCmd(app *App) *cobra.Command {
	root := &cobra.Command{
		Use:           "notes",
		SilenceUsage:  true,
		SilenceErrors: true, // 错误由调用方统一输出，测试只看返回值
	}
	root.SetOut(app.Out)
	root.SetErr(app.Err)
	root.AddCommand(newListCmd(app), newShowCmd(app))
	return root
}
```

每次测试调用 `NewRootCmd` 都得到一棵全新的树，flag、状态、输出流互不影响。

## 2.2 设置参数并捕获输出

Cobra 提供了 `SetArgs`、`SetOut`、`SetErr`，正好覆盖测试需要：

```go
func execute(t *testing.T, app *App, args ...string) (string, error) {
	t.Helper()
	buf := new(bytes.Buffer)
	app.Out = buf
	app.Err = buf

	cmd := NewRootCmd(app)
	cmd.SetArgs(args)
	err := cmd.Execute()
	return buf.String(), err
}
```

★ **关键认知**：`cmd.SetOut` 会沿命令树向下生效，子命令里的 `cmd.OutOrStdout()` 会取到根命令设置的 writer。因此只要在根上设一次，所有子命令的输出都能被捕获——前提是命令内部统一使用 `cmd.OutOrStdout()`，而不是 `fmt.Println`。

## 2.3 一个完整的命令测试

```go
func TestListCommand(t *testing.T) {
	app := &App{Store: newMemoryStore("go", "cli", "viper")}

	out, err := execute(t, app, "list", "--limit", "2")
	if err != nil {
		t.Fatalf("执行失败: %v", err)
	}
	if !strings.Contains(out, "go") {
		t.Fatalf("输出中缺少预期内容: %q", out)
	}
	if strings.Contains(out, "viper") {
		t.Fatalf("limit 未生效，输出包含第三条: %q", out)
	}
}
```

## 2.4 表驱动测试

参数校验、错误分支适合用表驱动一次覆盖：

```go
func TestShowCommandErrors(t *testing.T) {
	tests := []struct {
		name    string
		args    []string
		wantErr string
	}{
		{name: "缺少参数", args: []string{"show"}, wantErr: "accepts 1 arg"},
		{name: "参数过多", args: []string{"show", "1", "2"}, wantErr: "accepts 1 arg"},
		{name: "笔记不存在", args: []string{"show", "404"}, wantErr: "不存在"},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			app := &App{Store: newMemoryStore()}
			_, err := execute(t, app, tt.args...)
			if err == nil {
				t.Fatal("期望报错，实际成功")
			}
			if !strings.Contains(err.Error(), tt.wantErr) {
				t.Fatalf("错误信息不含 %q：%v", tt.wantErr, err)
			}
		})
	}
}
```

⚠ **关键认知**：断言错误时不要依赖完整的英文错误文本。Cobra 的 `ExactArgs` 文案可能随版本变化，只匹配关键片段（如 `accepts 1 arg`）或改用 `errors.Is` 判断自定义哨兵错误，测试才不容易因升级而碎。

# 3. 集成测试

## 3.1 什么时候需要真实二进制

单元测试覆盖不到的地方，交给集成测试：

- 真正的进程退出码。
- Shell 补全脚本能否生成。
- 与文件系统、环境变量、配置文件的交互。
- `--help` 的实际排版。

## 3.2 只编译一次

在 `TestMain` 里把二进制构建到临时目录，所有集成测试共用：

```go
var binaryPath string

func TestMain(m *testing.M) {
	dir, err := os.MkdirTemp("", "notes-e2e")
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	defer os.RemoveAll(dir)

	binaryPath = filepath.Join(dir, "notes")
	if runtime.GOOS == "windows" {
		binaryPath += ".exe"
	}

	build := exec.Command("go", "build", "-o", binaryPath, ".")
	build.Stderr = os.Stderr
	if err := build.Run(); err != nil {
		fmt.Fprintln(os.Stderr, "构建失败:", err)
		os.Exit(1)
	}

	os.Exit(m.Run())
}
```

⚠ **关键认知**：`TestMain` 里不要调用 `defer` 之后又直接 `os.Exit` 以外的清理方式——`os.Exit` 会跳过 defer。上面的 `defer os.RemoveAll` 实际上不会执行，稳妥做法是显式在 `m.Run()` 之后清理：

```go
code := m.Run()
_ = os.RemoveAll(dir)
os.Exit(code)
```

## 3.3 断言退出码

```go
func TestShowMissingNoteExitCode(t *testing.T) {
	cmd := exec.Command(binaryPath, "show", "404")
	cmd.Env = append(os.Environ(), "NOTES_CONFIG="+filepath.Join(t.TempDir(), "none.yaml"))

	err := cmd.Run()
	var exitErr *exec.ExitError
	if !errors.As(err, &exitErr) {
		t.Fatalf("期望非零退出，实际: %v", err)
	}
	if exitErr.ExitCode() != 1 {
		t.Fatalf("期望退出码 1，实际 %d", exitErr.ExitCode())
	}
}
```

如果同时要检查错误文本，用 `CombinedOutput` 或分别接 `cmd.Stdout` / `cmd.Stderr` 缓冲区。注意 `CombinedOutput` 会打乱两条流的顺序，验证分流时不要用它。

# 4. 可测试的设计

## 4.1 依赖注入而不是全局单例

```go
type NoteStore interface {
	List(tag string, limit int) ([]Note, error)
	Get(id string) (Note, error)
}

func newListCmd(app *App) *cobra.Command {
	var tag string
	var limit int

	cmd := &cobra.Command{
		Use:   "list",
		Short: "列出笔记",
		RunE: func(cmd *cobra.Command, args []string) error {
			notes, err := app.Store.List(tag, limit) // 依赖从 App 进来
			if err != nil {
				return fmt.Errorf("列出笔记失败: %w", err)
			}
			return renderNotes(cmd.OutOrStdout(), notes)
		},
	}
	cmd.Flags().StringVar(&tag, "tag", "", "按标签过滤")
	cmd.Flags().IntVarP(&limit, "limit", "n", 20, "最多返回条数")
	return cmd
}
```

测试时注入内存实现，生产时注入真实存储。命令层不需要知道数据从哪来。

## 4.2 配置与上下文

- 配置文件路径通过 `--config` 或 `App.ConfigPath` 传入，不要写死 `$HOME`。
- 用 `cmd.Context()` 传递取消信号，测试里用 `context.WithTimeout` 模拟超时。
- 环境变量在测试中用 `t.Setenv` 设置，它会自动恢复，不会污染其他测试。

```go
func TestOutputFromEnv(t *testing.T) {
	t.Setenv("NOTES_OUTPUT", "json") // 测试结束自动还原
	// ...
}
```

# 5. 最佳实践

| 方面 | 要点 |
|------|------|
| 命令设计 | 一个命令一件事；破坏性操作要确认；支持 `--json`；能接管道 |
| 输出与错误 | 结果走 stdout，日志走 stderr；退出码稳定；错误含行动建议 |
| 结构与发布 | main 极薄；命令树用构造函数；版本用 `-X` 注入；产物带校验和 |

★ **关键认知**：最佳实践不是"多写代码"，而是**把契约固定下来**：输出流、退出码、配置优先级、版本格式。契约稳定之后，测试才有明确的断言对象，用户也才敢把工具写进脚本。

# 6. 常见坑

| 现象 | 原因 | 处理 |
|------|------|------|
| 单独跑过、一起跑失败 | 复用了全局 `rootCmd` 或全局 viper | 每次构造新命令树 |
| 测试卡住 | 命令在等 stdin | 注入空的 `io.Reader` 或提供 `--yes` |
| 退出码断言不到 | 用 `CombinedOutput` 后只看了 err | 用 `*exec.ExitError` 取码 |
| 临时文件污染仓库 | 直接写当前目录 | 用 `t.TempDir()` |
| 错误文本断言易碎 | 依赖完整英文文案 | 匹配关键片段或哨兵错误 |
| 输出断言不到 | 命令内部用了 `fmt.Println` | 统一改用 `cmd.OutOrStdout()` |
| 清理逻辑没执行 | `TestMain` 里 `os.Exit` 跳过 defer | 显式清理后再退出 |
| 测试互相影响环境变量 | 直接 `os.Setenv` | 使用 `t.Setenv` |

# 7. 动手练习

1. 为 `notes list` 写表驱动测试，覆盖"无参数""指定标签""limit 为 0"三种情况。
2. 在 `TestMain` 中编译二进制，写一个集成测试验证 `notes show 404` 的退出码为 1 且 stderr 非空。
3. 把 `NewRootCmd` 改成接收 `App` 的构造函数，注入一个内存版 `NoteStore`，让测试不依赖真实文件。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 可测试入口 | `run(args, in, out, err) int`，副作用收敛到 main |
| 命令层测试 | `SetArgs` + `SetOut` + 断言 error |
| 隔离 | 用构造函数生成新命令树，避免全局状态 |
| 集成测试 | `TestMain` 编译一次，断言退出码与输出 |
| 设计 | 依赖注入、配置与上下文隔离、`t.Setenv` |
| 契约 | 输出流、退出码、配置优先级、版本格式 |

至此 Go/CLI 系列结束：从命令树与 flag，到配置、错误、发布和测试，一条完整的工具链路已经闭环。下一步可以进入 Go/gRPC，把这套命令行能力接到真实的远程服务上。
