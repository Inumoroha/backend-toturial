# 3. Pool 与 GC 及性能测试

本节目标：理解 `sync.Pool` 与 GC 的关系，并用 benchmark 判断它是否真的带来收益。

---

## 一、Pool 中对象可能被 GC 清理

`sync.Pool` 的一个重要语义是：

```text
池中的对象可能在任意一次 GC 后被清理。
```

所以它不能用于保存业务状态。

错误理解：

```text
我 Put 进去一个对象，下次一定能 Get 出来。
```

正确理解：

```text
Pool 只是提供临时对象复用机会，拿不到就重新创建。
```

---

## 二、Pool 不等于缓存

缓存要求：

```text
key 对应 value；
value 在过期或删除前应该存在；
读取结果影响业务逻辑。
```

`Pool` 不满足这些要求。

如果你需要缓存，请使用：

- `map + Mutex/RWMutex`
- `sync.Map`
- Redis
- 专门的缓存库

---

## 三、benchmark 对比

无 Pool 版本：

```go
func BenchmarkBuildWithoutPool(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        var buf bytes.Buffer
        buf.WriteString("hello")
        _ = buf.String()
    }
}
```

Pool 版本：

```go
func BenchmarkBuildWithPool(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        buf := bufferPool.Get().(*bytes.Buffer)
        buf.Reset()
        buf.WriteString("hello")
        _ = buf.String()
        bufferPool.Put(buf)
    }
}
```

运行：

```bash
go test -bench=. -benchmem ./...
```

---

## 四、什么时候收益明显

收益更可能明显：

- 对象较大。
- 创建频繁。
- 生命周期短。
- 分配出现在热点路径。

收益不明显：

- 对象很小。
- 调用频率低。
- 编译器能栈分配。
- 使用 Pool 反而增加复杂度。

---

## 五、本节练习

1. 写无 Pool 和有 Pool 的 benchmark。
2. 比较 `B/op` 和 `allocs/op`。
3. 增大 buffer 内容，观察收益变化。
4. 强制触发 GC，理解 Pool 对象可能消失。

---

## 六、本节达标标准

学完本节后，你应该能做到：

- 解释 Pool 与 GC 的关系。
- 不把 Pool 当缓存。
- 用 benchmark 判断 Pool 是否值得使用。
