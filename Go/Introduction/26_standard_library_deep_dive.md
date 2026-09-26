标准库是 Go 最宝贵的财富——它质量高、设计统一、文档详尽，是学习 Go 惯用法和底层实现最好的教科书。这一章深入几个核心包（io、bufio、net/http、encoding/json），看看它们如何体现 Go 的设计哲学。

# 1. io 包 —— 抽象的基石

## 1.1 核心接口

★ **关键认知**：`io` 包定义了 Go 最核心的 I/O 抽象——几个小而美的接口，是"接口应该小"哲学的典范：

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

type Closer interface {
    Close() error
}
```

★ **关键认知**：整个 Go I/O 体系都建立在这几个接口上——文件、网络连接、内存缓冲、压缩流都实现了 `Reader`/`Writer`。这种统一抽象让你能写出不依赖具体 I/O 源的通用代码。

## 1.2 组合接口

```go
type ReadWriter interface {
    Reader
    Writer
}

type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}
```

★ **关键认知**：Go 用**接口组合**而非继承来构建更复杂的接口，层次清晰、灵活复用。

# 2. bufio 包 —— 缓冲的威力

## 2.1 为什么需要缓冲

★ **关键认知**：底层 I/O（系统调用）很昂贵，逐字节读写会严重拖慢性能。`bufio` 通过**用户态缓冲区**批量读写，大幅减少系统调用次数：

```
无缓冲：读 1000 个字节 → 可能 1000 次系统调用
bufio：  读 1000 个字节 → 1~2 次系统调用（一次读满 4096 字节缓冲区）
```

## 2.2 核心类型

```go
r := bufio.NewReader(reader)      // 包装一个 Reader，加缓冲
line, _ := r.ReadString('\n')     // 按行读取
r.ReadBytes(',')                  // 按分隔符读取

w := bufio.NewWriter(writer)      // 包装一个 Writer
w.WriteString("hello")
w.Flush()                         // ★ 必须 flush 才会真正写入
```

★ **关键认知**：`bufio.Writer` 必须调用 `Flush()` 才会把缓冲的数据真正写入底层——这是最常见的坑：忘了 Flush，数据"丢失"（其实还在缓冲区）。

## 2.3 Scanner

```go
scanner := bufio.NewScanner(file)
for scanner.Scan() {              // 逐行扫描
    line := scanner.Text()
    fmt.Println(line)
}
if err := scanner.Err(); err != nil {
    // 处理错误
}
```

★ **关键认知**：`Scanner` 是逐行/逐词读取的便捷封装，内部有默认 64KB 的行长度限制（超长行会报错 `ErrTooLong`）。处理大文件时优先用 Scanner 而非 `ReadString`。

# 3. encoding/json —— 序列化的实现

## 3.1 基础用法

```go
type User struct {
    Name string `json:"name"`
    Age  int    `json:"age,omitempty"`
}

// 序列化
b, err := json.Marshal(User{Name: "tom", Age: 18})   // {"name":"tom","age":18}

// 反序列化
var u User
json.Unmarshal(b, &u)
```

## 3.2 底层实现原理

★ **关键认知**：`encoding/json` 的核心是**反射**（见反射章节）。序列化时：

```
1. 反射获取结构体的类型信息，遍历字段
2. 读取每个字段的 json tag，决定输出键名
3. 反射获取字段值，按类型序列化
4. 用 encodeState 的缓冲区拼装 JSON 字符串
```

★ **关键认知**：正是因为这个反射实现，json 序列化的性能不如手写或代码生成方案。高性能场景（如高频 API 序列化）可用 `jsoniter` 或 `easyjson`（代码生成）替代，性能可提升数倍。

## 3.3 性能优化技巧

```go
// 1. 流式：处理大 JSON 用 Decoder/Encoder 而非 Marshal（避免一次性读入内存）
dec := json.NewDecoder(reader)
var u User
dec.Decode(&u)          // 逐条读取

// 2. 复用 buffer
var buf bytes.Buffer
enc := json.NewEncoder(&buf)
enc.Encode(u)

// 3. 使用 json.RawMessage 延迟解析
type Wrapper struct {
    Data json.RawMessage `json:"data"`
}
```

# 4. net/http —— HTTP 服务的实现

## 4.1 一个最小服务器

```go
func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "hello %s", r.URL.Path)
}

http.HandleFunc("/", handler)
http.ListenAndServe(":8080", nil)
```

## 4.2 并发的实现机制

★ **关键认知**：`net/http` 的高并发来自**每连接一 goroutine**模型——每个 HTTP 请求都在独立的 goroutine 中处理：

```
连接处理流程：
1. listener 接受连接
2. 每个连接启动一个 goroutine
3. 每个请求再启动一个 goroutine（或复用连接）
4. goroutine 阻塞在 I/O 时，Go 调度器自动切换到其他 goroutine
```

★ **关键认知**：这是 Go 并发模型（轻量 goroutine + netpoll）的完美展示——无需线程池、无需异步回调，同步代码就能写出高并发服务器。每个请求 handler 都是"看起来同步"的代码，实际由调度器高效并发执行。

## 4.3 关键注意点

★ **关键认知**：使用 net/http 的几个重要细节：

```
1. 必须 defer r.Body.Close()（复用连接时释放资源）
2. 请求处理应支持 context 取消（r.Context()）
3. 生产环境应配置超时（Server 的 ReadTimeout/WriteTimeout）
4. 大文件用流式，避免一次性读入内存
```

# 5. 标准库体现的设计哲学

★ **关键认知**：从这些包可以总结出 Go 标准库的设计共性：

```
1. 小接口：io.Reader、error 都是极小接口
2. 组合优于继承：接口组合、结构体嵌入
3. 显式错误：返回值传递 error，而非异常
4. 接口与实现分离：io.Reader 统一了文件/网络/内存
5. 注重并发：net/http 天然并发
6. 零依赖：标准库几乎不依赖外部包
```

# 6. 小结

| 包 | 核心 | 要点 |
|----|------|------|
| io | Reader/Writer 接口 | 统一 I/O 抽象 |
| bufio | 缓冲区 | 减少系统调用，记得 Flush |
| encoding/json | 反射序列化 | 高性能用代码生成 |
| net/http | 每连接一 goroutine | 并发模型完美展示 |

下一篇进入测试、基准与性能分析，讲清楚 go test 的单元测试、表驱动测试、基准测试、竞态检测、模糊测试，以及 pprof 性能剖析。
