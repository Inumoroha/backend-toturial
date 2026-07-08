# 8. Benchmark：评估短链接生成器性能

本节目标：学完后你能用 Go benchmark 评估热点函数性能，并避免常见的性能测试误读。

简短引入：后端项目里不是所有代码都需要 benchmark。它更适合热点路径，例如短链接生成、签名计算、JSON 编解码、权限匹配、日志字段构造。benchmark 的价值是对比修改前后，而不是拿一个数字当绝对真理。

## 一、为什么需要它

假设短链接服务每秒要生成大量短码。你优化了一段代码，但不确定是否真的更快。手动看日志不准确，压测整个服务又太重。此时可以先对核心函数写 benchmark。

```text
Benchmark 适合评估小而关键的热点代码，不适合替代完整压测。
```

完整压测还要考虑网络、数据库、缓存、限流、部署环境。benchmark 先帮你判断纯函数或小模块的成本。

## 二、基本用法

创建项目：

Windows PowerShell：

```powershell
mkdir benchdemo
cd benchdemo
go mod init example.com/benchdemo
```

Linux/macOS：

```bash
mkdir benchdemo
cd benchdemo
go mod init example.com/benchdemo
```

新建 `shortcode.go`：

```go
package benchdemo

const alphabet = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"

func EncodeBase62(n uint64) string {
	if n == 0 {
		return "0"
	}

	buf := make([]byte, 0, 11)
	for n > 0 {
		idx := n % uint64(len(alphabet))
		buf = append(buf, alphabet[idx])
		n = n / uint64(len(alphabet))
	}

	for i, j := 0, len(buf)-1; i < j; i, j = i+1, j-1 {
		buf[i], buf[j] = buf[j], buf[i]
	}
	return string(buf)
}
```

新建 `shortcode_test.go`：

```go
package benchdemo

import "testing"

func TestEncodeBase62(t *testing.T) {
	got := EncodeBase62(3844)
	if got != "100" {
		t.Fatalf("EncodeBase62() = %q, want %q", got, "100")
	}
}

func BenchmarkEncodeBase62(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_ = EncodeBase62(uint64(i + 1000000))
	}
}
```

运行 benchmark：

```powershell
go test -bench=. -benchmem
```

Linux/macOS：

```bash
go test -bench=. -benchmem
```

## 三、关键参数/语法/代码结构

benchmark 函数以 `Benchmark` 开头，参数是 `*testing.B`。

`b.N` 是 Go 自动调整的循环次数。你不需要自己决定跑多少次，Go 会尽量让结果稳定。

`-bench=.` 表示运行当前包里所有 benchmark。

`-benchmem` 会显示每次操作的内存分配次数和字节数。后端热点函数中，分配次数经常比纯耗时更值得关注。

如果你的 Go 版本支持 `b.Loop()`，也可以写成：

```go
func BenchmarkEncodeBase62Loop(b *testing.B) {
	i := uint64(1000000)
	for b.Loop() {
		_ = EncodeBase62(i)
		i++
	}
}
```

为了兼容更多项目，本教程主要使用 `b.N` 写法。

## 四、真实后端场景示例

短链接服务里，生成短码通常只是链路的一部分。完整创建短链接可能还包括：

- 校验原始 URL。
- 写数据库。
- 写缓存。
- 生成访问日志。

benchmark 不应该把这些都混在一起。可以先单独测短码编码：

```go
func BenchmarkEncodeBase62LargeID(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_ = EncodeBase62(9876543210123)
	}
}
```

如果你要比较两种实现，可以写两个 benchmark：

```go
func BenchmarkEncodeBase62SmallID(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_ = EncodeBase62(12345)
	}
}

func BenchmarkEncodeBase62LargeIDCompare(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_ = EncodeBase62(9876543210123)
	}
}
```

真实项目里，benchmark 常用于回答：“这个改动有没有明显改善？”而不是“这个函数在线上一定就是这个耗时”。

## 五、注意点

不要在 benchmark 循环里打印日志。日志 I/O 会掩盖你真正想测的代码。

不要在循环里做随机网络请求、真实数据库写入。那测到的是外部系统和网络波动。

如果函数有初始化成本，可以用 `b.ResetTimer()` 排除准备阶段：

```go
func BenchmarkWithSetup(b *testing.B) {
	data := make([]uint64, 1000)
	for i := range data {
		data[i] = uint64(i + 1000000)
	}

	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		_ = EncodeBase62(data[i%len(data)])
	}
}
```

benchmark 结果会受机器、CPU 频率、后台进程影响。重要变更最好多跑几次，或者用专门工具比较多次结果。

## 六、常见误区

- 看到一次结果就下结论：benchmark 需要多次观察，尤其是差距很小时。
- 把完整 HTTP 链路塞进 benchmark：这更接近压测，不是小模块 benchmark。
- 忽略内存分配：高频路径里频繁分配会增加 GC 压力。
- 在循环里初始化大量数据：测到的是准备成本，不是目标函数。
- 为了追求极限性能牺牲可读性：后端业务代码首先要正确、可维护，热点路径再谨慎优化。

## 七、本节达标标准

- 能写 `BenchmarkXxx(b *testing.B)`。
- 能运行 `go test -bench=. -benchmem`。
- 能解释 `b.N` 的作用。
- 能用 `b.ResetTimer()` 排除准备成本。
- 能说明 benchmark 和完整压测的区别。

