命令树的骨架搭起来之后，真正决定工具好不好用的是细节：哪些参数属于当前命令、哪些应该被所有子命令继承、位置参数错了怎么给出清晰提示、用户按 Tab 时能不能补全。本篇围绕 Cobra 的 Command 与 pflag，把这些日常最容易混淆的点一次讲清。

# 1. 命令、标志与位置参数

## 1.1 三个概念的分工

命令行输入可以拆成三类信息，Cobra 对它们的处理方式完全不同：

| 组成 | 示例 | 归属 | 说明 |
|------|------|------|------|
| 命令 | `notes list` | Command 树 | 决定由哪个 `RunE` 执行 |
| 标志（flag） | `--tag go`、`-n 10` | pflag | 有名字、有类型、可在父子间继承 |
| 位置参数（args） | `notes show 42` 中的 `42` | `RunE` 的 `args` | 无名字，顺序即语义 |

★ **关键认知**：flag 是"带名字的配置"，位置参数是"按位置约定的数据"。能用 flag 表达的可选项就不要塞进位置参数，否则用户必须记顺序，帮助信息也说不清楚。

## 1.2 一次输入如何被拆解

```text
输入：notes list --tag go -n 5 extra

1. Cobra 沿命令树匹配到 list 命令
2. pflag 解析 --tag go 与 -n 5，从参数中移除
3. 剩余 ["extra"] 作为位置参数交给 list 的 Args 校验
4. 校验通过后调用 list 的 RunE(cmd, args)
```

⚠ **关键认知**：pflag 默认允许"标志与位置参数交错"，即 `notes list extra --tag go` 也能解析。这很宽松，但也意味着用户写错顺序时不会报错。若工具要求参数顺序严格，可用 `cmd.Flags().SetInterspersed(false)` 关闭交错解析。

# 2. 子命令的组织方式

## 2.1 创建与挂载

一个子命令就是独立的 `*cobra.Command`，通过 `AddCommand` 挂到父节点：

```go
var listCmd = &cobra.Command{
	Use:     "list",
	Aliases: []string{"ls"}, // 允许 notes ls
	Short:   "列出所有笔记",
	Long:    "列出所有笔记，可按标签过滤并限制条数。",
	RunE: func(cmd *cobra.Command, args []string) error {
		tag, _ := cmd.Flags().GetString("tag")
		limit, _ := cmd.Flags().GetInt("limit")
		cmd.Printf("tag=%s limit=%d\n", tag, limit) // Printf 走命令的输出流
		return nil
	},
}

func init() {
	rootCmd.AddCommand(listCmd)
	listCmd.Flags().String("tag", "", "按标签过滤")
	listCmd.Flags().IntP("limit", "n", 20, "最多返回多少条")
}
```

`Aliases` 只影响匹配，不影响帮助里显示的主名称。`notes ls` 与 `notes list` 会进入同一个 `RunE`。

## 2.2 父命令该不该有 RunE

如果父命令只是分类容器（如 `notes config`），通常不写 `RunE`。此时直接执行 `notes config` 会打印帮助并以 0 退出。若写了 `RunE`，它就会变成一个可执行命令，容易让用户困惑"为什么 config 直接跑起来什么都没做"。

★ **关键认知**：命令树的层级应该反映用户的心智模型，而不是代码的包结构。`config get` 合理，因为"配置"是一个域、"读取"是一个动作；如果只是为了少写几个文件而随意嵌套，用户很难记住路径。

# 3. Flag 系统：局部与持久

## 3.1 定义 flag

pflag 提供成对的方法：`Xxx` 定义长选项，`XxxP` 额外接受一个短选项字母：

```go
listCmd.Flags().String("tag", "", "按标签过滤")       // 只有 --tag
listCmd.Flags().IntP("limit", "n", 20, "最多返回条数") // --limit 与 -n
listCmd.Flags().BoolP("json", "j", false, "以 JSON 输出")
```

参数顺序固定为：名称、短名、默认值、说明。说明会出现在帮助里，值得认真写。

## 3.2 持久 flag 与局部 flag

这是 Cobra 最容易混淆的一处，用一张表说清：

| 类型 | 定义方法 | 可见范围 | 典型用途 |
|------|----------|----------|----------|
| 局部 flag | `cmd.Flags()` | 仅当前命令 | `list --tag`、`add --title` |
| 持久 flag | `cmd.PersistentFlags()` | 当前命令及其所有后代 | `--config`、`--verbose` |

```text
rootCmd
  PersistentFlags: --config, --verbose
  ├── list
  │     Flags: --tag, --limit
  └── config
        PersistentFlags: --file
        ├── get
        └── set
```

`notes --verbose list` 与 `notes list --verbose` 都能用；`--tag` 只有 `list` 认识；`--file` 只有 `config` 及其子命令认识。

★ **关键认知**：持久 flag 是"沿命令树向下继承"，不是"全局变量"。它定义在哪个节点，就只对该节点的子树生效。真正的全局开关应定义在根命令上，而不是随手定义在中间层。

## 3.3 读取 flag 的两种姿势

```go
// 方式一：运行时按名字读取，返回 (值, error)
tag, err := cmd.Flags().GetString("tag")
if err != nil {
	return err
}

// 方式二：定义时绑定变量，之后直接读变量
var tag string
listCmd.Flags().StringVar(&tag, "tag", "", "按标签过滤")
```

方式一更利于测试与复用，方式二写起来更短。团队里选一种并保持一致即可。⚠ 不要两种混用同一个 flag，容易出现"变量改了但 flag 值没变"的错觉。

## 3.4 布尔 flag 的陷阱

布尔 flag 不带值时默认为 `true`，因此关闭它必须显式赋值：

```bash
notes list --json          # json = true
notes list --json=false    # json = false
notes list --json false    # 错误理解！false 会被当成位置参数
```

⚠ **关键认知**：`--json false` 不会把 json 设为 false，而是把 `false` 当成一个位置参数传给 `RunE`。如果该命令使用 `cobra.NoArgs`，用户会收到"未知参数 false"的报错，这反而是一次正确的提醒。

# 4. 位置参数校验

## 4.1 内置校验器

`Args` 字段接收一个函数，Cobra 提供了一批现成的：

| 校验器 | 含义 | 适用示例 |
|--------|------|----------|
| `cobra.NoArgs` | 不允许任何位置参数 | `version`、`list` |
| `cobra.ExactArgs(n)` | 必须正好 n 个 | `show <id>` |
| `cobra.MinimumNArgs(n)` | 至少 n 个 | `add <标题>...` |
| `cobra.MaximumNArgs(n)` | 至多 n 个 | `search <词>` |
| `cobra.RangeArgs(min, max)` | 数量在区间内 | `move <id> <位置>` |
| `cobra.ArbitraryArgs` | 不限制 | 透传型命令 |

```go
var showCmd = &cobra.Command{
	Use:   "show <id>",
	Short: "查看单条笔记",
	Args:  cobra.ExactArgs(1), // 少一个或多一个都会在 RunE 之前失败
	RunE: func(cmd *cobra.Command, args []string) error {
		id := args[0]
		cmd.Printf("查看笔记 %s\n", id)
		return nil
	},
}
```

## 4.2 自定义校验与候选值

需要更复杂的规则时，直接写函数：

```go
func validateID(cmd *cobra.Command, args []string) error {
	if err := cobra.ExactArgs(1)(cmd, args); err != nil {
		return err
	}
	for _, r := range args[0] {
		if r < '0' || r > '9' { // 只允许纯数字 ID
			return fmt.Errorf("ID 必须是数字，收到 %q", args[0])
		}
	}
	return nil
}
```

如果参数取值来自固定集合，用 `ValidArgs` 配合 `cobra.OnlyValidArgs`，补全与校验可以共用同一份数据：

```go
var formatCmd = &cobra.Command{
	Use:       "format <类型>",
	ValidArgs: []string{"json", "yaml", "table"},
	Args:      cobra.OnlyValidArgs, // 只校验，不限制数量
	RunE: func(cmd *cobra.Command, args []string) error {
		cmd.Printf("输出格式：%s\n", args[0])
		return nil
	},
}
```

★ **关键认知**：`Args` 校验发生在 `RunE` 之前，失败时 `RunE` 根本不会执行。这正是把校验放在这里的价值——业务代码可以假定输入已经合法，减少防御性判断。

## 4.3 必填 flag 与互斥 flag

位置参数之外，flag 也有约束。Cobra 提供声明式方法：

```go
addCmd.Flags().String("title", "", "笔记标题")
_ = addCmd.MarkFlagRequired("title") // 未提供时自动报错

formatCmd.Flags().Bool("json", false, "JSON 输出")
formatCmd.Flags().Bool("yaml", false, "YAML 输出")
formatCmd.MarkFlagsMutuallyExclusive("json", "yaml") // 只能选一个
```

⚠ **关键认知**：`MarkFlagRequired` 只对**定义在当前命令 `Flags()` 上**的 flag 生效。如果 flag 定义在父命令的 `PersistentFlags()` 上，应使用 `MarkPersistentFlagRequired`，否则不会触发校验。

# 5. 帮助与用法文本

## 5.1 自动生成的帮助

Cobra 会根据 `Use`、`Short`、`Long`、`Example`、参数校验和 flag 定义自动生成帮助。一个完整的命令定义如下：

```go
var searchCmd = &cobra.Command{
	Use:   "search <关键词>",
	Short: "按关键词搜索笔记",
	Long: `在标题和正文中搜索关键词。

默认不区分大小写，可用 --case-sensitive 改为区分大小写。`,
	Example: `  notes search "cobra"
  notes search "viper" --case-sensitive`,
	Args: cobra.ExactArgs(1),
	RunE: func(cmd *cobra.Command, args []string) error {
		cmd.Printf("搜索：%s\n", args[0])
		return nil
	},
}
```

`Example` 里的缩进会原样保留，通常用两个空格让它在帮助里对齐。

★ **关键认知**：帮助文本是用户第一次接触工具时唯一会读的文档。`Short` 一行说清"做什么"，`Long` 补充"边界与默认行为"，`Example` 给出可直接复制的命令——这三层比写一份 README 更能提升上手率。

# 6. Shell 自动补全

## 6.1 生成补全脚本

Cobra 为有子命令的程序自动生成 `completion` 命令，支持 bash、zsh、fish、PowerShell：

```bash
# bash：临时生效
source <(notes completion bash)

# zsh：写入 fpath 中的目录
notes completion zsh > "${fpath[1]}/_notes"

# PowerShell：当前会话生效
notes completion powershell | Out-String | Invoke-Expression
```

补全不仅能补命令名和 flag 名，还能补 `ValidArgs` 中声明的候选值。

## 6.2 动态补全

候选值来自运行时（比如数据库里的标签）时，用 `RegisterFlagCompletionFunc`：

```go
_ = listCmd.RegisterFlagCompletionFunc(
	"tag",
	func(cmd *cobra.Command, args []string, toComplete string) ([]string, cobra.ShellCompDirective) {
		tags := []string{"go", "cli", "viper"} // 实际项目可从存储层读取
		return tags, cobra.ShellCompDirectiveNoFileComp
	},
)
```

⚠ **关键认知**：补全函数会在用户每次按 Tab 时执行，必须快且无副作用。不要在里面做网络请求、写文件或启动耗时初始化，否则终端会明显卡顿。返回 `ShellCompDirectiveNoFileComp` 可以阻止 Shell 继续补全文件名。

# 7. 常见坑汇总

| 现象 | 原因 | 处理 |
|------|------|------|
| 子命令读不到父级 flag | flag 被定义成了局部 flag | 改用 `PersistentFlags()` |
| `--flag false` 不生效 | 布尔 flag 的赋值方式错误 | 使用 `--flag=false` |
| 必填校验没触发 | flag 在父级持久 flag 上 | 用 `MarkPersistentFlagRequired` |
| 短选项冲突 | 父子命令用了同一字母 | 规划时集中分配短名 |
| 补全卡顿 | 补全函数做了重活 | 缓存结果或改用静态候选 |
| 帮助里命令顺序混乱 | 挂载顺序影响显示 | 保持稳定挂载顺序或使用分组 |
| 参数含空格被拆开 | Shell 分词导致 | 用户需用引号包裹 |

# 8. 动手练习

1. 给 `notes` 增加 `search <关键词>` 子命令，要求恰好一个位置参数，并支持 `--case-sensitive` 布尔 flag。
2. 把 `--verbose` 定义为根命令的持久 flag，验证 `notes --verbose list` 与 `notes list --verbose` 都能读到它。
3. 为 `list --tag` 注册动态补全，候选值来自一个字符串切片，并确认按 Tab 时不再补全文件名。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 输入三分 | 命令定路由、flag 带名字、位置参数按顺序 |
| 局部 flag | `cmd.Flags()`，只当前命令可见 |
| 持久 flag | `cmd.PersistentFlags()`，沿子树继承 |
| 参数校验 | `Args` 在 `RunE` 前执行，优先用内置校验器 |
| 帮助文本 | `Short` 一句话、`Long` 讲边界、`Example` 可复制 |
| 自动补全 | 自动生成 `completion`，动态候选要快且无副作用 |

下一篇进入配置管理，讲清楚如何用 Viper 读取配置文件与环境变量、flag 与配置的优先级如何设计，以及"热加载"到底能热到什么程度。
