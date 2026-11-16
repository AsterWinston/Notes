写完成 CLI 只是开始，用户拿到手的通常是一个能直接运行的二进制：不需要装 Go、不依赖系统库、版本信息一眼可见。发布环节要解决三件事——为目标平台编译、把版本写进二进制、把产物分发给用户。本篇按这条链路走一遍。

# 1. 发布一个 CLI 要解决什么

一个可交付的 CLI 发行物，至少要满足下面几点：

| 维度 | 要解决的问题 | 常见做法 |
|------|--------------|----------|
| 平台 | Linux、macOS、Windows 都能跑 | `GOOS` / `GOARCH` 交叉编译 |
| 架构 | amd64 与 arm64 都要覆盖 | 构建矩阵 |
| 依赖 | 不要求用户装运行时 | 静态链接、`CGO_ENABLED=0` |
| 版本 | 用户能报告"我装的是哪版" | `-ldflags -X` 注入 |
| 完整性 | 下载内容没被篡改 | `checksums.txt` + 签名 |
| 安装 | 一条命令完成 | Homebrew、脚本、容器镜像 |

★ **关键认知**：Go 的最大分发优势是**单个静态二进制**。只要不引入 cgo 依赖，编译产物可以直接丢进 `scratch` 容器或最小系统镜像，不需要 libc、不需要运行时。发布方案应当围绕这个优势设计，而不是照搬解释型语言的打包方式。

# 2. 交叉编译

## 2.1 GOOS 与 GOARCH

Go 编译器自带交叉编译能力，设置两个环境变量即可，无需安装目标平台的工具链：

```bash
GOOS=linux   GOARCH=amd64 go build -o notes_linux_amd64 .
GOOS=darwin  GOARCH=arm64 go build -o notes_darwin_arm64 .
GOOS=windows GOARCH=amd64 go build -o notes_windows_amd64.exe .
```

常用组合：

| GOOS | GOARCH | 目标 |
|------|--------|------|
| linux | amd64 | 绝大多数服务器 |
| linux | arm64 | ARM 服务器、树莓派 |
| darwin | arm64 | Apple Silicon |
| darwin | amd64 | Intel Mac |
| windows | amd64 | 64 位 Windows |

在 PowerShell 里语法不同，需要先设置环境变量：

```powershell
$env:GOOS = "linux"
$env:GOARCH = "amd64"
go build -o notes_linux_amd64 .
```

## 2.2 CGO 与静态链接

默认情况下，若代码或依赖使用了 cgo，产物会动态链接系统库，换一台机器可能直接报"找不到共享库"。发布版通常显式关闭 cgo：

```bash
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -trimpath -o notes .
```

- `CGO_ENABLED=0`：强制纯 Go 实现，产出静态二进制。
- `-trimpath`：去掉源码的绝对路径，避免把构建机目录泄露进产物，也便于复现构建。

★ **关键认知**：`CGO_ENABLED=0` 会改变部分标准库的实现（如 DNS 解析走纯 Go 路径）。绝大多数 CLI 没有影响，但如果依赖了依赖 cgo 的数据库驱动或图形库，就必须改用带 libc 的基础镜像，而不是强行静态编译。

## 2.3 批量构建脚本

手敲每条命令容易漏平台，写成循环更可靠：

```bash
#!/usr/bin/env bash
set -euo pipefail

VERSION="${VERSION:-$(git describe --tags --always --dirty)}"
COMMIT="$(git rev-parse --short HEAD)"
DATE="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
LDFLAGS="-s -w -X main.version=${VERSION} -X main.commit=${COMMIT} -X main.date=${DATE}"

mkdir -p dist
for target in linux/amd64 linux/arm64 darwin/amd64 darwin/arm64 windows/amd64; do
  os="${target%/*}"
  arch="${target#*/}"
  out="dist/notes_${VERSION}_${os}_${arch}"
  if [ "$os" = "windows" ]; then
    out="${out}.exe" # Windows 必须带扩展名
  fi
  echo "构建 ${os}/${arch}"
  CGO_ENABLED=0 GOOS="$os" GOARCH="$arch" \
    go build -trimpath -ldflags "$LDFLAGS" -o "$out" .
done
```

`-s -w` 去掉符号表与调试信息，能显著减小体积；代价是无法用 `dlv` 直接调试发布版，所以只应加在发布构建上。

# 3. 版本注入

## 3.1 在代码里留变量

先在 `main` 包声明可被链接器覆盖的变量：

```go
package main

var (
	version = "dev"     // 默认值，源码直接 go run 时显示 dev
	commit  = "none"    // 由构建脚本注入
	date    = "unknown" // 构建时间，UTC
)
```

构建时用 `-X` 覆盖：

```bash
go build -ldflags "-X main.version=v0.1.0 -X main.commit=abc1234" -o notes .
```

⚠ **关键认知**：`-X` 只能作用于**字符串类型的包级变量**，且包路径必须写全。如果变量在 `internal/buildinfo` 包里，就要写 `-X example.com/notes/internal/buildinfo.Version=v0.1.0`，写成 `-X buildinfo.Version` 不会生效，而且不会有任何报错——版本号会安静地停在 `dev`。

## 3.2 提供 version 子命令

用户排查问题时第一句话往往是"你装的是哪个版本"，所以版本命令要稳定、可机读：

```go
var versionCmd = &cobra.Command{
	Use:   "version",
	Short: "打印版本信息",
	Args:  cobra.NoArgs,
	RunE: func(cmd *cobra.Command, args []string) error {
		jsonOut, _ := cmd.Flags().GetBool("json")
		info := map[string]string{
			"version": version,
			"commit":  commit,
			"date":    date,
		}
		if jsonOut {
			return json.NewEncoder(cmd.OutOrStdout()).Encode(info)
		}
		fmt.Fprintf(cmd.OutOrStdout(), "notes %s (commit %s, built %s)\n", version, commit, date)
		return nil
	},
}

func init() {
	versionCmd.Flags().Bool("json", false, "以 JSON 输出")
}
```

# 4. 产物组织

一次发布应当包含二进制、校验和与说明文档，目录结构保持稳定，安装脚本才好写：

```text
dist/
├── notes_v0.1.0_linux_amd64.tar.gz
├── notes_v0.1.0_linux_arm64.tar.gz
├── notes_v0.1.0_darwin_amd64.tar.gz
├── notes_v0.1.0_darwin_arm64.tar.gz
├── notes_v0.1.0_windows_amd64.zip
└── checksums.txt
```

生成校验和：

```bash
sha256sum dist/*.tar.gz dist/*.zip > dist/checksums.txt
# macOS 自带的是 shasum
shasum -a 256 dist/*.tar.gz dist/*.zip > dist/checksums.txt
```

用户下载后可验证：

```bash
sha256sum -c checksums.txt
```

# 5. 发布到 GitHub Releases

用 `gh` 可以直接创建 Release 并上传产物：

```bash
gh release create "v${VERSION}" dist/* \
  --title "v${VERSION}" \
  --notes "本次发布内容见 CHANGELOG.md"
```

★ **关键认知**：发布产物应当**可复现且可验证**：同样的 tag 能构建出同样内容的二进制（`-trimpath`、固定 Go 版本），并且提供校验和。这比"能下载"重要得多，是供应链安全的基本要求。

# 6. Homebrew

macOS 与 Linux 用户习惯用 Homebrew 安装。官方仓库审核严格，通常先做自己的 tap：

```ruby
class Notes < Formula
  desc "一个用于管理笔记的命令行工具"
  homepage "https://github.com/you/notes"
  url "https://github.com/you/notes/archive/refs/tags/v0.1.0.tar.gz"
  sha256 "替换为实际 tarball 的 sha256"
  license "MIT"

  depends_on "go" => :build

  def install
    ldflags = "-s -w -X main.version=#{version}"
    system "go", "build", *std_go_args(ldflags: ldflags), "./cmd/notes"
  end

  test do
    assert_match version.to_s, shell_output("#{bin}/notes version")
  end
end
```

放进 `homebrew-notes` 仓库的 `Formula/` 目录后，用户执行：

```bash
brew tap you/notes
brew install notes
```

Homebrew 也可以分发预编译二进制（`url` 指向 tar.gz，`bin.install "notes"`），构建更快，但要为每种架构分别校验。

# 7. Docker

把 CLI 放进容器，常见于 CI 任务或作为其他镜像的基础工具。多阶段构建可以只保留最终二进制：

```dockerfile
FROM golang:1.23-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -trimpath -ldflags "-s -w" -o /out/notes ./cmd/notes

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/notes /usr/local/bin/notes
ENTRYPOINT ["notes"]
```

⚠ **关键认知**：容器里的 CLI 默认没有 TTY，也没有 HOME 目录。涉及交互确认、写配置文件、读缓存的命令必须支持 `--yes`、显式 `--config` 和可写目录参数，否则在容器里会以各种"莫名其妙"的方式失败。

# 8. 安装脚本

"一行命令安装"体验最好，但风险也最高：

```bash
curl -fsSL https://raw.githubusercontent.com/you/notes/main/install.sh | sh
```

脚本至少要处理平台识别、下载、校验、安装路径四件事：

```bash
#!/usr/bin/env bash
set -euo pipefail

VERSION="${VERSION:-v0.1.0}" # 默认固定版本，避免"今天装的和明天装的不一样"
OS="$(uname -s | tr '[:upper:]' '[:lower:]')"
ARCH="$(uname -m)"
case "$ARCH" in
  x86_64) ARCH=amd64 ;;
  aarch64 | arm64) ARCH=arm64 ;;
  *) echo "不支持的架构: $ARCH" >&2; exit 1 ;;
esac

URL="https://github.com/you/notes/releases/download/${VERSION}/notes_${VERSION}_${OS}_${ARCH}.tar.gz"
TMP="$(mktemp -d)"
trap 'rm -rf "$TMP"' EXIT

curl -fsSL "$URL" -o "$TMP/notes.tar.gz"
tar -xzf "$TMP/notes.tar.gz" -C "$TMP"
install -m 0755 "$TMP/notes" /usr/local/bin/notes
echo "已安装 notes 到 /usr/local/bin/notes"
```

★ **关键认知**：`curl | sh` 的本质是"把远端代码直接交给 Shell 执行"。如果必须提供这种安装方式，至少要做到 HTTPS、固定版本、校验和验证，并在文档里给出"先下载再检查"的手动步骤。便利性不能以放弃完整性校验为代价。

# 9. 常见坑

| 现象 | 原因 | 处理 |
|------|------|------|
| 目标机器报找不到共享库 | cgo 导致动态链接 | `CGO_ENABLED=0` 重新构建 |
| 版本号一直是 `dev` | `-X` 包路径或变量名写错 | 使用完整导入路径，确认变量是 string |
| Windows 产物无法双击 | 缺少 `.exe` 扩展名 | 构建时补上扩展名 |
| Apple Silicon 跑成 Rosetta | 下载了 amd64 包 | 按 `uname -m` 选择 arm64 |
| 容器内命令卡住 | 等待不存在的 TTY 输入 | 提供 `--yes` 并检测终端 |
| 构建机路径泄露 | 未加 `-trimpath` | 发布构建统一加 |
| 校验和与产物不匹配 | 先压缩后校验顺序错 | 对最终上传文件生成校验和 |

# 10. 动手练习

1. 写一个构建脚本，一次产出 linux/amd64、darwin/arm64、windows/amd64 三个平台的可执行文件，并用 `file` 或 `go version -m` 检查产物。
2. 给 `version` 子命令注入 `version`、`commit`、`date` 三个变量，比较 `go run . version` 与 `go build -ldflags ...` 后的输出差异。
3. 用多阶段 Dockerfile 构建一个小于 20MB 的镜像，运行 `docker run --rm <image> version` 验证入口正确。

# 11. 小结

| 主题 | 要点 |
|------|------|
| 交叉编译 | `GOOS` + `GOARCH`，一条命令一个平台 |
| 静态产物 | `CGO_ENABLED=0` + `-trimpath` |
| 版本注入 | `-ldflags "-X 完整包路径.变量=值"` |
| 产物 | 二进制 + 校验和 + 发布说明 |
| 分发 | Homebrew 面向本机用户，Docker 面向流水线 |
| 安装脚本 | 固定版本、HTTPS、校验和、可手动复核 |

下一篇进入测试与最佳实践，讲清楚命令怎么被单元测试、如何做端到端集成测试、怎样设计可测试的命令结构，并把整个 CLI 模块的关键经验收束成一份检查清单。
