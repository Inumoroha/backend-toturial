# 4. Mutex 与 RWMutex 性能对比

本节目标：通过 benchmark 判断 `Mutex` 和 `RWMutex` 的实际表现，形成“先测量再优化”的习惯。

---

## 一、为什么需要 benchmark

很多人会默认认为：

```text
RWMutex 支持多个读者并发，所以一定比 Mutex 快。
```

这不总是成立。

原因包括：

- `RWMutex` 自身管理读者和写者的成本更高。
- 临界区很短时，额外成本可能抵消收益。
- 写操作较多时，读写锁优势下降。
- 真实业务里的锁竞争和测试想象可能不同。

---

## 二、设计三组场景

建议至少比较三种读写比例：

```text
读多写少：95% Get，5% Set
读写均衡：50% Get，50% Set
写多读少：20% Get，80% Set
```

不同场景的结论可能完全不同。

---

## 三、并发 benchmark 模板

```go
func BenchmarkCacheReadMostly(b *testing.B) {
    cache := NewCache()
    cache.Set("key", "value")

    b.ReportAllocs()
    b.RunParallel(func(pb *testing.PB) {
        i := 0
        for pb.Next() {
            if i%20 == 0 {
                cache.Set("key", "value")
            } else {
                cache.Get("key")
            }
            i++
        }
    })
}
```

你可以分别实现：

- `MutexCache`
- `RWMutexCache`

然后用相同 benchmark 对比。

---

## 四、解读时注意公平性

benchmark 要尽量公平：

- 两个实现的功能一致。
- 数据规模一致。
- 并发模式一致。
- 不要在一个版本里多做额外工作。
- 每次修改后多运行几次。

运行：

```bash
go test -bench=. -benchmem -count=5 ./...
```

`-count=5` 可以减少单次波动误导。

---

## 五、工程结论怎么写

不要只写：

```text
RWMutex 更快。
```

应该写成：

```text
在当前 benchmark 的读多写少场景下，RWMutex 版本吞吐更好；
在读写均衡场景下差距不明显；
考虑到业务缓存读取远多于写入，可以使用 RWMutex；
如果后续写入比例上升，需要重新评估。
```

这才是后端工程里的结论。

---

## 六、本节练习

1. 实现 `MutexCache` 和 `RWMutexCache`。
2. 写三组读写比例 benchmark。
3. 使用 `-count=5` 多次运行。
4. 记录结果到 README。
5. 根据结果写一段工程结论。

---

## 七、本节达标标准

学完本节后，你应该能做到：

- 用 benchmark 比较 `Mutex` 和 `RWMutex`。
- 知道 `RWMutex` 不一定更快。
- 能根据读写比例、复杂度和测试结果选择实现。
