# 1. Go 环境与测试工具

本节目标：准备学习 `sync` 所需的最小 Go 工具链，掌握后续教程会反复用到的测试、竞态检测和 benchmark 命令。

学习并发时，不建议只看代码。并发 bug 的特点是：

```text
有时出现，有时不出现；
本地小数据量不出现，上线高并发才出现；
普通测试可能通过，但 race detector 能发现问题；
功能正确不代表性能合理。
```

所以从第一天开始，就要养成“写示例、写测试、跑 race、跑 benchmark”的习惯。

---

## 一、确认 Go 环境

在终端执行：

```bash
go version
```

如果能看到类似下面的输出，说明 Go 已经安装：

```text
go version go1.22.0 windows/amd64
```

版本不需要完全一致。学习 `sync` 的核心内容时，Go 1.20 之后的版本都比较合适，因为 typed atomic 类型已经可用。

---

## 二、初始化练习模块

建议在 `Go/sync` 目录下初始化一个 module：

```bash
go mod init sync-learning
```

如果目录里已经存在 `go.mod`，就不需要重复执行。

后续每个阶段都可以把示例放在独立子目录中，例如：

```text
01-race/
02-mutex/
03-rwmutex/
```

这样执行 `go test ./...` 时，可以一次跑完全部练习。

---

## 三、最常用的测试命令

### 1. 普通测试

```bash
go test ./...
```

作用：

- 编译所有包。
- 运行所有 `*_test.go` 中的测试。
- 检查功能是否符合预期。

普通测试只能证明“这一次运行通过”，不能证明没有并发问题。

### 2. 竞态检测

```bash
go test -race ./...
```

作用：

- 在运行时检测数据竞争。
- 当两个 goroutine 对同一变量并发访问，并且至少一个是写操作，且没有同步保护时，通常会报告 race。

注意：

- `-race` 会让程序变慢，通常用于开发、测试、CI，不用于生产运行。
- `-race` 只能发现这次测试路径实际执行到的竞争。
- 没有 race 报告不等于一定没有并发 bug。

### 3. Benchmark

```bash
go test -bench=. -benchmem ./...
```

作用：

- 观察函数执行耗时。
- 观察内存分配次数和分配字节数。
- 对比 `Mutex`、`RWMutex`、`atomic`、`sync.Pool` 等方案的性能差异。

学习并发性能时，不要靠直觉下结论。很多时候 `RWMutex` 不一定比 `Mutex` 快，`atomic` 也不一定值得牺牲可读性。

---

## 四、推荐测试文件结构

以并发安全计数器为例：

```text
counter/
  counter.go
  counter_test.go
  counter_bench_test.go
```

其中：

- `counter.go` 写实现。
- `counter_test.go` 写功能测试和并发测试。
- `counter_bench_test.go` 写性能测试。

这种结构会在后续阶段反复使用。

---

## 五、一个最小测试例子

```go
package counter

import "testing"

func TestAdd(t *testing.T) {
    got := 1 + 1
    if got != 2 {
        t.Fatalf("got %d, want 2", got)
    }
}
```

运行：

```bash
go test ./...
```

如果你刚开始学 Go 后端，测试不是额外负担，而是保护你理解并发行为的工具。并发代码不写测试，很容易靠运气运行。

---

## 六、本节达标标准

学完本节后，你应该能做到：

- 确认本机 Go 环境可用。
- 初始化一个学习用 Go module。
- 知道 `go test ./...`、`go test -race ./...`、`go test -bench=. -benchmem ./...` 的作用。
- 知道 race detector 不能代替设计，只能帮助发现实际执行路径中的数据竞争。
- 能为后续每个阶段建立独立练习目录。
