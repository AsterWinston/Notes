一个 CLI 从"能跑"走向"能交付"，配置层是绕不过去的一关：开发时要指向本地数据库，CI 里要读环境变量，用户又希望能用 flag 临时覆盖。Viper 把这些来源收拢成一套读取接口，但它的优先级规则和"热加载"边界经常被误解。本篇按"来源 → 优先级 → 边界"的顺序讲清楚。

# 1. 为什么需要配置层

## 1.1 配置来源的优先级

一个成熟的 CLI 通常遵循下面的漏斗，越靠上优先级越高：

```text
viper.Set() 显式赋值              ← 最高，通常只用于测试
        │
用户显式修改的命令行 flag
        │
环境变量 NOTES_*
        │
配置文件 config.yaml
        │
SetDefault / flag 默认值          ← 最低
```

这张图要在写代码前就定下来，因为它直接决定用户遇到"为什么我的配置没生效"时该如何解释。

★ **关键认知**：配置的本质不是"读一个文件"，而是**为每个值定义来源与优先级**。只要优先级清晰，来源多并不可怕；优先级含糊，来源一多就会互相打架。

# 2. Viper 快速上手

## 2.1 安装与最小示例

```bash
go get github.com/spf13/viper@latest
```

```go
package main

import (
	"fmt"
	"strings"

	"github.com/spf13/viper"
)

func main() {
	v := viper.New() // 独立实例，避免全局状态在测试之间互相污染

	v.SetConfigName("config") // 不带扩展名，Viper 会按类型查找
	v.SetConfigType("yaml")   // 显式声明类型，便于处理无扩展名文件
	v.AddConfigPath(".")      // 当前目录
	v.AddConfigPath("$HOME/.notes")

	v.SetDefault("output", "table") // 兜底值
	v.SetEnvPrefix("notes")         // 环境变量前缀，实际读取 NOTES_*
	v.AutomaticEnv()                // 允许环境变量覆盖配置
	v.SetEnvKeyReplacer(strings.NewReplacer(".", "_")) // log.level → LOG_LEVEL

	if err := v.ReadInConfig(); err != nil {
		// 配置文件不存在是正常情况，其他错误才需要上报
		if _, ok := err.(viper.ConfigFileNotFoundError); !ok {
			panic(fmt.Errorf("读取配置失败: %w", err))
		}
	}

	fmt.Println("output =", v.GetString("output"))
}
```

## 2.2 读取配置

Viper 按类型提供读取方法，键名大小写不敏感：

```go
host := v.GetString("server.host") // 嵌套键用点号
port := v.GetInt("server.port")    // 自动做类型转换
debug := v.GetBool("debug")
tags := v.GetStringSlice("tags")   // YAML 列表读成 []string
```

⚠ **关键认知**：`GetString` 几乎不会失败——键不存在时返回空字符串，类型不对时尽力转换。所以"配置缺失"不会被自动发现。对关键配置应使用 `v.IsSet("server.host")` 显式检查，或在启动时用 `Unmarshal` + 校验统一处理。

## 2.3 Viper 怎么找配置文件

`SetConfigName` + `AddConfigPath` 的组合会按路径顺序、按支持的扩展名依次查找。找不到不会报错，`ReadInConfig` 返回 `ConfigFileNotFoundError`。

如果想完全掌控位置（比如 `--config` 指定），用 `SetConfigFile`：

```go
if path != "" {
	v.SetConfigFile(path) // 明确到文件名，忽略 SetConfigName 与搜索路径
}
```

| 方法 | 语义 | 适用场景 |
|------|------|----------|
| `SetConfigName("config")` | 给名字，按路径与扩展名搜索 | 约定优先的默认配置 |
| `SetConfigFile("/etc/notes/config.yaml")` | 给完整路径，不做搜索 | `--config` 显式指定 |

★ **关键认知**：搜索式加载适合"零配置启动"，显式路径适合"用户明确指定"。生产工具通常两者都支持：先用 `--config`，没给再走默认搜索路径。

# 3. 配置文件与结构体

## 3.1 支持的格式

Viper 通过扩展名识别格式，常用的是 YAML、JSON、TOML：

```yaml
# config.yaml
server:
  host: 127.0.0.1
  port: 8080
log:
  level: info
output: table
tags:
  - go
  - cli
```

## 3.2 解析到结构体

手工 `GetString` 适合少量键；键一多就应该映射到结构体，并配合 `mapstructure` 标签：

```go
type Config struct {
	Server struct {
		Host string `mapstructure:"host"`
		Port int    `mapstructure:"port"`
	} `mapstructure:"server"`
	Log struct {
		Level string `mapstructure:"level"`
	} `mapstructure:"log"`
	Output string   `mapstructure:"output"`
	Tags   []string `mapstructure:"tags"`
}

func load(v *viper.Viper) (Config, error) {
	var cfg Config
	if err := v.Unmarshal(&cfg); err != nil {
		return Config{}, fmt.Errorf("解析配置失败: %w", err)
	}
	if cfg.Server.Port <= 0 || cfg.Server.Port > 65535 {
		return Config{}, fmt.Errorf("端口非法: %d", cfg.Server.Port)
	}
	return cfg, nil
}
```

⚠ **关键认知**：`Unmarshal` 只做映射，不做业务校验。端口范围、必填项、枚举值这些规则必须自己写，并且放在启动阶段一次性检查完，不要等到第一次请求才发现配置有问题。

# 4. 环境变量

## 4.1 自动映射

开启 `AutomaticEnv` 后，Viper 在读取某个键时会尝试同名环境变量。前缀与替换器负责把 `server.host` 变成 `NOTES_SERVER_HOST`：

```go
v.SetEnvPrefix("notes")
v.SetEnvKeyReplacer(strings.NewReplacer(".", "_", "-", "_"))
v.AutomaticEnv()
```

```bash
NOTES_SERVER_HOST=0.0.0.0 NOTES_LOG_LEVEL=debug notes list
```

## 4.2 绑定单个变量

自动映射依赖命名规则，遇到历史遗留的变量名（比如 `NOTES_HOME` 对应 `paths.home`）时，用 `BindEnv` 显式指定：

```go
_ = v.BindEnv("paths.home", "NOTES_HOME") // 两个名字不一致时显式绑定
_ = v.BindEnv("token", "NOTES_TOKEN", "CI_TOKEN") // 也支持多个候选名
```

★ **关键认知**：`AutomaticEnv` 只对"已知的键"生效得可靠——它依赖你曾经 `SetDefault`、绑定过 flag，或配置文件里出现过该键。完全动态的键名建议显式 `BindEnv`，否则容易出现"明明设了环境变量却读不到"的情况。

# 5. 与 Cobra flag 集成

## 5.1 BindPFlag

Viper 可以绑定 pflag，让配置文件为 flag 提供默认值，同时保留命令行覆盖能力：

```go
rootCmd.PersistentFlags().String("config", "", "配置文件路径")
rootCmd.PersistentFlags().String("output", "table", "输出格式")

_ = v.BindPFlag("config", rootCmd.PersistentFlags().Lookup("config"))
_ = v.BindPFlag("output", rootCmd.PersistentFlags().Lookup("output"))
```

绑定后统一从 Viper 读：

```go
output := v.GetString("output") // flag 未改则可能来自配置或环境变量
```

## 5.2 优先级的真实行为

关键细节：**只有当 flag 被用户显式修改过，绑定才会参与覆盖**。如果 flag 保持默认值，Viper 会继续向下查找环境变量和配置文件。

```text
notes --output json     → flag 已变更，输出 json（最高）
notes                   → flag 未变更，读 NOTES_OUTPUT 或 config.yaml
```

这正是我们想要的行为：flag 默认值不会把配置文件"压死"。

★ **关键认知**：不要在 `RunE` 里用 `cmd.Flags().GetString("output")` 读绑定过的 flag。那样拿到的是 flag 的默认值，绕过了 Viper 的优先级链。一旦绑定，读取入口就统一改成 `v.GetString`。

## 5.3 常见顺序错误

初始化顺序必须是：定义 flag → 创建 Viper → 绑定 flag → 读取配置。若先 `ReadInConfig` 再绑定，虽然读取时仍会动态计算，但容易在中间步骤用错来源。把初始化写成一个显式函数，顺序就固定了：

```go
func newViper(cmd *cobra.Command) (*viper.Viper, error) {
	v := viper.New()
	v.SetEnvPrefix("notes")
	v.AutomaticEnv()
	if err := v.BindPFlags(cmd.PersistentFlags()); err != nil {
		return nil, err
	}
	if err := v.ReadInConfig(); err != nil {
		var notFound viper.ConfigFileNotFoundError
		if !errors.As(err, &notFound) {
			return nil, err
		}
	}
	return v, nil
}
```

# 6. 热加载与它的边界

## 6.1 WatchConfig

Viper 提供文件监听：

```go
v.WatchConfig()
v.OnConfigChange(func(e fsnotify.Event) {
	// 回调在独立 goroutine 中执行，不能假设业务状态是静止的
	log.Printf("配置文件已变化: %s", e.Name)
})
```

回调触发时 Viper 会重新读取文件，之后 `GetString` 等调用能读到新值。

## 6.2 什么能热，什么不能

| 配置项 | 能否热加载 | 原因 |
|--------|------------|------|
| 日志级别 | 通常可以 | 读一次原子变量即可生效 |
| 输出格式 | 可以 | 每次命令执行时读取 |
| 请求超时 | 可以 | 下次创建客户端时使用 |
| 已解析的结构体 | 不会自动更新 | `Unmarshal` 是快照 |
| 监听地址与端口 | 不行 | Socket 已绑定，需重启 |
| 数据库连接串 | 通常不行 | 连接池已建立 |
| 命令行 flag | 不行 | 进程启动时已解析 |

⚠ **关键认知**：热加载不是"改文件，一切自动生效"。Viper 只保证后续 `Get` 能读到新值；已经 `Unmarshal` 出来的结构体、已经初始化的客户端、已经绑定的端口都不会自己更新。真正的热加载需要你显式设计"谁在什么时候重新读取"。

# 7. 常见坑

| 现象 | 常见原因 | 处理 |
|------|----------|------|
| 配置读不到 | 搜索路径或文件名不对 | 用 `SetConfigFile` 或打印实际路径 |
| 环境变量不生效 | 未开 `AutomaticEnv` 或未加前缀 | 检查前缀与替换器 |
| flag 总压过配置 | 用 `Flags().GetString` 读取 | 改从 Viper 读 |
| 测试之间互相影响 | 使用了包级全局 `viper` | 改用 `viper.New()` |
| 热加载后结构体没变 | `Unmarshal` 是快照 | 重新 Unmarshal 并替换快照 |
| 数字读成字符串 | 用 `GetString` 读端口 | 用 `GetInt` 或结构体解析 |
| 配置缺失无人发现 | `GetString` 返回空串 | 启动时集中校验必填项 |

# 8. 动手练习

1. 创建 `config.yaml`，写入 `output: json`，再用 `NOTES_OUTPUT=table` 启动，观察最终生效值，验证环境变量高于配置文件。
2. 给 `--output` 绑定 Viper，分别测试"不传 flag""传 `--output yaml`"两种情况，确认 flag 只在显式修改时覆盖。
3. 开启 `WatchConfig`，在程序运行期间修改日志级别，让回调打印新旧值，思考哪些业务组件需要同步刷新。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 优先级 | `Set` > 已变更 flag > 环境变量 > 配置文件 > 默认值 |
| 实例隔离 | 用 `viper.New()`，避免全局状态污染测试 |
| 结构体映射 | `mapstructure` 标签 + 启动时集中校验 |
| flag 绑定 | 绑定后统一从 Viper 读，不要再读 flag |
| 热加载 | 只保证 `Get` 读到新值，业务刷新要自己设计 |
| 并发 | 回调在独立 goroutine，快照替换要加锁 |

下一篇进入日志与错误处理，讲清楚退出码的约定、错误应该写到哪里、日志如何分级，以及调试模式怎样做到"默认安静、需要时详细"。
