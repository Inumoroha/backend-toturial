# 4. benchmark、pprof、trace 性能分析

本节目标：掌握并发组件常用性能分析工具，避免凭感觉优化。

---

## 一、benchmark

运行：

```bash
go test -bench=. -benchmem ./...
```

多次运行：

```bash
go test -bench=. -benchmem -count=5 ./...
```

关注：

- `ns/op`
- `B/op`
- `allocs/op`

适合比较：

- `Mutex` vs `RWMutex`
- `Mutex` vs `atomic`
- 使用 `Pool` vs 不使用 `Pool`

---

## 二、CPU pprof

生成 CPU profile：

```bash
go test -bench=. -cpuprofile=cpu.out ./...
```

查看：

```bash
go tool pprof cpu.out
```

常用命令：

```text
top
list FunctionName
web
```

CPU pprof 用于找耗时热点。

---

## 三、内存 pprof

生成内存 profile：

```bash
go test -bench=. -benchmem -memprofile=mem.out ./...
go tool pprof mem.out
```

适合分析：

- 哪些路径分配最多。
- `sync.Pool` 是否减少分配。
- 临时对象是否逃逸到堆。

---

## 四、阻塞分析

如果怀疑锁等待或 channel 阻塞，可以使用 block profile。

测试中开启：

```go
runtime.SetBlockProfileRate(1)
```

或在服务中暴露 pprof 后查看 block profile。

关注：

- 哪些锁等待时间长。
- 哪些 channel 操作阻塞。

---

## 五、trace

生成 trace：

```bash
go test -trace=trace.out ./...
go tool trace trace.out
```

trace 可以观察：

- goroutine 创建。
- goroutine 阻塞和唤醒。
- 调度行为。
- GC 和系统调用。

适合排查复杂并发行为。

---

## 六、分析顺序建议

```text
先用测试证明正确；
再用 race detector 查数据竞争；
再用 benchmark 观察是否值得优化；
再用 pprof 定位热点；
最后用 trace 看调度和阻塞细节。
```

不要一开始就上复杂工具。

---

## 七、本节练习

1. 给 `MutexCounter` 和 `AtomicCounter` 写 benchmark。
2. 生成 CPU profile。
3. 给 buffer 构建函数写 Pool/非 Pool benchmark。
4. 生成内存 profile。
5. 对一个阻塞队列测试生成 trace。

---

## 八、本节达标标准

学完本节后，你应该能做到：

- 使用 benchmark 验证性能。
- 生成并查看 CPU 和内存 pprof。
- 知道 block profile 和 trace 的用途。
- 用工具结果支持优化判断。
