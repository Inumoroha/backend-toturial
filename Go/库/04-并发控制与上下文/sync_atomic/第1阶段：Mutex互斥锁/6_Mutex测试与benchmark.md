# 6. Mutex 测试与 benchmark

本节目标：掌握 `Mutex` 代码的测试方式，能够用 race detector 和 benchmark 验证正确性与性能。

---

## 一、功能测试

功能测试验证单线程下的基本行为：

```go
func TestSafeCounterAdd(t *testing.T) {
    var c SafeCounter

    c.Add(2)
    c.Add(3)

    if got := c.Value(); got != 5 {
        t.Fatalf("got %d, want 5", got)
    }
}
```

这类测试不能覆盖并发问题，但能保证基础语义正确。

---

## 二、并发测试

```go
func TestSafeCounterConcurrent(t *testing.T) {
    var c SafeCounter
    var wg sync.WaitGroup

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            c.Add(1)
        }()
    }

    wg.Wait()

    if got := c.Value(); got != 1000 {
        t.Fatalf("got %d, want 1000", got)
    }
}
```

运行：

```bash
go test -race ./...
```

并发测试要断言最终结果，不要只看日志。

---

## 三、benchmark

普通 benchmark：

```go
func BenchmarkSafeCounterAdd(b *testing.B) {
    var c SafeCounter
    b.ReportAllocs()

    for i := 0; i < b.N; i++ {
        c.Add(1)
    }
}
```

并发 benchmark：

```go
func BenchmarkSafeCounterAddParallel(b *testing.B) {
    var c SafeCounter
    b.ReportAllocs()

    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            c.Add(1)
        }
    })
}
```

运行：

```bash
go test -bench=. -benchmem ./...
```

---

## 四、如何解读结果

重点关注：

- `ns/op`：单次操作耗时。
- `B/op`：单次操作分配字节数。
- `allocs/op`：单次操作分配次数。

对于简单 `Mutex` 计数器，通常不应该有额外分配。

如果 benchmark 中出现大量分配，可能是：

- 方法内部创建了对象。
- 使用了接口导致逃逸。
- benchmark 写法本身有问题。

---

## 五、不要只比较数字

例如 `atomic` 计数器可能比 `Mutex` 快，但它只能优雅处理简单状态。

如果你的组件需要保证：

```text
value >= 0
lastUpdated 与 value 一致
多个字段一起修改
```

那么 `Mutex` 往往比 atomic 更安全、更可维护。

性能优化的顺序应该是：

```text
先正确
再可读
再测量
最后优化
```

---

## 六、本节练习

1. 给 `SafeCounter` 写功能测试。
2. 给 `SafeCounter` 写并发测试。
3. 使用 `go test -race` 验证。
4. 写普通 benchmark 和并发 benchmark。
5. 实现 atomic 版本并对比。
6. 在 README 中总结两者适用场景。

---

## 七、本节达标标准

学完本节后，你应该能做到：

- 为加锁组件写功能测试和并发测试。
- 使用 `go test -race` 辅助发现数据竞争。
- 使用 benchmark 对比不同实现。
- 不把 benchmark 数字当成唯一设计依据。
