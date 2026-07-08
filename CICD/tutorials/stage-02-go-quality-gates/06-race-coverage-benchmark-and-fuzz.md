# 06：Race、覆盖率、Benchmark 与 Fuzz

## 1. 本节目标

这一节学习 Go 测试的进阶工具：

- race detector：发现并发数据竞争。
- coverage：查看测试覆盖率。
- benchmark：测量性能。
- fuzzing：用随机输入发现边界问题。

它们不是每个场景都要同时跑，但你需要知道它们分别解决什么问题。

## 2. Race Detector

并发是 Go 的强项，但并发代码也容易出现数据竞争。

运行：

```bash
go test -race ./...
```

如果代码中存在多个 goroutine 同时访问共享变量，并且至少一个是写操作，就可能被 race detector 报出来。

示例问题：

```go
var count int

func Inc() {
	go func() {
		count++
	}()
	go func() {
		count++
	}()
}
```

修复方向通常是：

- 使用 `sync.Mutex`。
- 使用 channel 串行化访问。
- 使用 `sync/atomic`。
- 避免共享可变状态。

## 3. race 检查适合什么时候跑

建议：

```text
PR 阶段跑 go test -race ./...
```

如果项目很大，race 很耗时，可以：

- 对核心包跑。
- 在 main 分支跑。
- 定时跑。
- 合并前手动跑。

学习项目可以直接全量跑。

## 4. 覆盖率

运行：

```bash
go test -cover ./...
```

生成覆盖率文件：

```bash
go test -coverprofile=coverage.out ./...
```

查看函数级覆盖：

```bash
go tool cover -func=coverage.out
```

生成 HTML 报告：

```bash
go tool cover -html=coverage.out
```

CI 中可以上传 `coverage.out` 作为 artifact。

## 5. 如何看待覆盖率

覆盖率不是越高越好。

高覆盖率不代表测试有效，低覆盖率也确实提示风险。

更重要的是：

- 核心业务规则是否覆盖。
- 错误路径是否覆盖。
- 边界输入是否覆盖。
- 数据库和外部依赖失败是否覆盖。
- 并发和重试逻辑是否覆盖。

初学项目可以先设目标：

```text
service 层覆盖率优先达到 70% 以上。
handler 层覆盖核心接口。
repository 层用集成测试覆盖关键路径。
```

## 6. Benchmark

Benchmark 用于测量性能。

函数格式：

```go
func BenchmarkNormalizeTitle(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_ = NormalizeTitle("  buy milk  ")
	}
}
```

运行：

```bash
go test -bench=. ./...
```

显示内存分配：

```bash
go test -bench=. -benchmem ./...
```

Benchmark 不建议作为初期 PR 必跑门禁，除非你的项目对性能非常敏感。

## 7. Fuzzing

Fuzzing 会用大量自动生成的输入测试函数，尝试发现崩溃、panic 或安全问题。

适合：

- 字符串解析。
- JSON/URL/协议解析。
- 权限规则解析。
- 自定义编码解码。
- 边界条件复杂的函数。

Fuzz 测试示例：

```go
func FuzzNormalizeTitle(f *testing.F) {
	f.Add("buy milk")
	f.Add("  buy milk  ")
	f.Add("")

	f.Fuzz(func(t *testing.T, input string) {
		got := NormalizeTitle(input)
		if strings.Contains(got, "\n") {
			t.Fatalf("title contains newline: %q", got)
		}
	})
}
```

运行：

```bash
go test -fuzz=FuzzNormalizeTitle ./internal/service
```

限制时间：

```bash
go test -fuzz=FuzzNormalizeTitle -fuzztime=30s ./internal/service
```

## 8. 哪些检查适合放进 CI

推荐：

| 检查 | PR 必跑 | main 跑 | 定时跑 | 说明 |
| --- | --- | --- | --- | --- |
| `go test ./...` | 是 | 是 | 可选 | 基础测试 |
| `go test -race ./...` | 是 | 是 | 可选 | 学习项目可全量跑 |
| coverage | 可选 | 是 | 可选 | 生成报告 |
| benchmark | 否 | 可选 | 是 | 性能趋势 |
| fuzz | 否 | 可选 | 是 | 适合重点函数 |

## 9. 小练习

执行：

```bash
go test ./...
go test -race ./...
go test -coverprofile=coverage.out ./...
go tool cover -func=coverage.out
```

然后回答：

1. 哪些包覆盖率低？
2. 低覆盖率是否影响核心业务？
3. 是否有并发代码需要 race 检查？
4. 哪个函数适合做 fuzz？

## 10. 本节小结

你现在应该理解：

- `go test -race ./...` 用于发现数据竞争。
- `coverage.out` 用于生成覆盖率报告。
- benchmark 用于性能测量。
- fuzzing 用于发现复杂输入下的崩溃和边界问题。
- 不是所有进阶检查都必须每次 PR 全量执行。

