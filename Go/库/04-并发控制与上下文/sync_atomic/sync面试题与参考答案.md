# Go sync 面试题与参考答案

本文整理 Go 后端面试中围绕 `sync` 包和并发同步机制常见的问题。建议不要死背答案，而是围绕四个关键词回答：

```text
共享状态
同步边界
正确性保证
工程取舍
```

---

## 一、并发基础与数据竞争

### 1. 什么是数据竞争？

数据竞争是指多个 goroutine 同时访问同一个变量，其中至少一个是写操作，并且这些访问之间没有同步保护。

典型例子：

```go
var count int

go func() { count++ }()
go func() { count++ }()
```

这里两个 goroutine 都会读写 `count`，且没有锁、channel、atomic 等同步机制，所以存在数据竞争。

---

### 2. `count++` 为什么不是并发安全的？

`count++` 看起来是一句代码，但不是一个原子操作。它大致包含三步：

```text
读取 count
计算 count + 1
写回 count
```

两个 goroutine 可能同时读取到相同旧值，然后分别写回相同新值，导致更新丢失。

---

### 3. 如何检测 Go 程序中的数据竞争？

使用 race detector：

```bash
go test -race ./...
go run -race main.go
```

它能在运行时发现当前执行路径中的数据竞争，并报告读写位置和 goroutine 创建位置。

注意：`-race` 没报错不代表绝对没有并发 bug，只能说明这次运行覆盖到的路径没有被检测出 race。

---

### 4. 普通 map 为什么不能并发读写？

Go 的普通 `map` 不是并发安全的。并发读写可能导致：

- race detector 报告数据竞争。
- 运行时报 `fatal error: concurrent map read and map write`。
- map 内部结构被破坏。

如果多个 goroutine 访问同一个 map，应使用：

- `map + Mutex`
- `map + RWMutex`
- `sync.Map`

---

### 5. 并发安全是不是只要写操作加锁就够了？

不是。只要一个 goroutine 写，另一个 goroutine 同时读，且没有同步保护，就是数据竞争。

所以如果某个变量由锁保护，那么所有读写路径都应该遵守同一把锁。

错误做法：

```go
func Set(v int) {
    mu.Lock()
    value = v
    mu.Unlock()
}

func Get() int {
    return value
}
```

`Get` 也应该加锁。

---

### 6. 什么是临界区？

临界区是指访问共享状态、并且不能被多个 goroutine 同时执行的一段代码。

例如：

```go
mu.Lock()
counter++
mu.Unlock()
```

这里 `counter++` 就是需要保护的临界区。

---

### 7. happens-before 是什么？

可以简单理解为：

```text
如果 A happens-before B，那么 A 中的写入对 B 可见。
```

Go 中常见建立 happens-before 的方式包括：

- `Mutex.Unlock` happens-before 后续成功的 `Mutex.Lock`。
- channel 发送 happens-before 对应接收。
- `WaitGroup.Done` 与 `Wait` 返回之间存在同步关系。
- `Once.Do` 完成后，初始化结果对后续调用可见。
- atomic 操作提供对应同步保证。

---

### 8. 锁只解决互斥问题吗？

不是。锁既解决互斥，也解决可见性。

如果 goroutine A 在锁内写入数据，goroutine B 后续用同一把锁读取数据，那么 B 能看到 A 的写入结果。

---

### 9. 能不能用 `time.Sleep` 等待 goroutine 完成？

不推荐。`Sleep` 只能等待一段时间，不能表达“某个任务已经完成并且结果可见”。

正确方式通常是：

- channel
- `WaitGroup`
- context
- 其他明确同步机制

---

### 10. 并发和并行有什么区别？

并发是多个任务在同一时间段内推进；并行是多个任务在同一时刻真正同时执行。

即使在单核机器上，也可能因为 goroutine 调度交错出现并发问题。

---

## 二、Mutex 面试题

### 11. `sync.Mutex` 是什么？

`sync.Mutex` 是互斥锁，用于保证同一时刻只有一个 goroutine 能进入某段临界区。

典型写法：

```go
mu.Lock()
defer mu.Unlock()
```

---

### 12. `Mutex` 保护的是变量本身吗？

更准确地说，`Mutex` 保护的是访问共享状态的规则或临界区。

如果一个字段由 `mu` 保护，那么所有访问这个字段的路径都应该使用同一把 `mu`。

---

### 13. 为什么推荐 `defer mu.Unlock()`？

因为它能保证函数返回前释放锁，避免提前 return 时忘记解锁。

尤其是代码有多个返回路径时：

```go
mu.Lock()
defer mu.Unlock()

if invalid {
    return
}
```

缺点是 `defer` 有一定开销。在极端高性能热点路径中，可以手动解锁，但业务代码通常优先保证正确性和可维护性。

---

### 14. `Mutex` 可以复制吗？

不能复制已经使用过的 `Mutex`。

如果结构体包含 `sync.Mutex`，通常应该：

- 使用指针接收者方法。
- 用指针传递结构体。
- 不要把包含锁的结构体按值复制。

错误示例：

```go
func (c Counter) Add() {}
```

这里值接收者会复制整个 `Counter`，也复制锁。

---

### 15. Go 的 `Mutex` 是可重入锁吗？

不是。Go 的 `sync.Mutex` 不可重入。

同一个 goroutine 已经持有某把锁，再次调用 `Lock` 会把自己阻塞住，造成死锁。

---

### 16. 常见死锁原因有哪些？

常见原因：

- `Lock` 后忘记 `Unlock`。
- 提前 return 没释放锁。
- 同一个 goroutine 重复加同一把锁。
- 多把锁加锁顺序不一致。
- 持锁期间调用外部函数，外部函数又尝试获取同一把锁。
- 持锁期间等待一个永远不会发生的事件。

---

### 17. 如何避免多把锁导致的死锁？

固定加锁顺序。

例如账户转账场景，永远先锁 ID 小的账户，再锁 ID 大的账户。只要所有路径都遵守同一顺序，就能避免互相等待。

---

### 18. 持锁期间为什么不要做慢操作？

持锁期间执行慢操作会阻塞其他 goroutine，降低吞吐并增加死锁风险。

不建议在锁内执行：

- 网络请求。
- 数据库查询。
- 文件 IO。
- 长时间计算。
- 调用外部未知函数。

推荐：锁内只读取或修改共享状态，锁外执行慢操作。

---

### 19. 什么是锁粒度？

锁粒度是指一把锁保护的状态范围和临界区大小。

粗锁：

- 简单。
- 不容易漏保护。
- 并发度较低。

细锁：

- 并发度更高。
- 设计复杂。
- 更容易出现锁顺序问题。

工程上通常先用简单正确的粗锁，再根据 benchmark 或线上指标优化。

---

### 20. `Mutex` 适合哪些后端场景？

适合保护：

- 本地缓存。
- 限流计数。
- session 状态。
- 后台任务状态。
- 连接池内部状态。
- 内存队列。
- 多字段业务不变量。

---

### 21. 用 `Mutex` 实现计数器要注意什么？

所有访问计数值的路径都要加同一把锁，包括读和写：

```go
type Counter struct {
    mu    sync.Mutex
    value int64
}

func (c *Counter) Add(n int64) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value += n
}

func (c *Counter) Value() int64 {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.value
}
```

---

### 22. `Mutex` 和 channel 都能同步，怎么选择？

简单判断：

```text
共享状态保护：优先考虑 Mutex。
数据流传递或事件通知：优先考虑 channel。
```

不要为了不用锁而滥用 channel。channel 更适合表达任务、消息、信号的流动。

---

## 三、RWMutex 面试题

### 23. `sync.RWMutex` 是什么？

`RWMutex` 是读写锁。

规则：

- 多个读者可以同时持有读锁。
- 写者需要独占锁。
- 写锁和读锁不能同时持有。

---

### 24. `RLock` 和 `Lock` 有什么区别？

`RLock` 用于读共享状态，多个读者可以并发。

`Lock` 用于写共享状态，需要独占。

读取共享 map 时也要用 `RLock`，因为可能有其他 goroutine 同时写。

---

### 25. `RWMutex` 一定比 `Mutex` 快吗？

不一定。

`RWMutex` 管理读者和写者的成本比 `Mutex` 更高。如果临界区很短、读写比例不极端、写操作较多，`RWMutex` 可能不如 `Mutex`。

是否使用 `RWMutex` 应通过 benchmark 验证。

---

### 26. 什么场景适合 `RWMutex`？

适合：

- 读多写少。
- 读临界区相对频繁。
- 写操作较少。
- 共享状态主要被查询。

例如：

- 本地配置缓存。
- 路由表。
- 读多写少的本地缓存。

---

### 27. `RWMutex` 读锁期间可以写数据吗？

不可以。读锁期间只能读共享状态。

错误示例：

```go
mu.RLock()
m[key] = value
mu.RUnlock()
```

写 map 必须使用写锁。

---

### 28. 能不能从读锁升级成写锁？

不能直接在持有读锁时申请写锁，这可能导致死锁。

正确方式：

```text
释放读锁；
申请写锁；
在写锁内重新检查条件。
```

重新检查很重要，因为释放读锁到获取写锁之间，状态可能已经被其他 goroutine 改变。

---

### 29. `RWMutex` 常见误用有哪些？

常见误用：

- 读锁期间写共享状态。
- 忘记 `RUnlock`。
- 持有读锁时申请写锁。
- 为了性能盲目把 `Mutex` 换成 `RWMutex`。
- 在读锁内做慢操作或调用外部函数。

---

### 30. TTL 缓存的 `Get` 一定能用 `RLock` 吗？

不一定。

如果 `Get` 只是读取，可以用 `RLock`。但如果 `Get` 发现 key 过期后要删除它，那么 `Get` 就包含写操作。

简单正确版本可以直接用写锁。更复杂的优化版本可以先 `RLock` 判断，再释放读锁、获取写锁并二次检查。

---

## 四、WaitGroup 面试题

### 31. `sync.WaitGroup` 用来做什么？

`WaitGroup` 用于等待一组 goroutine 完成。

典型写法：

```go
var wg sync.WaitGroup

wg.Add(1)
go func() {
    defer wg.Done()
    doWork()
}()

wg.Wait()
```

---

### 32. `Add`、`Done`、`Wait` 分别做什么？

- `Add(n)`：增加 n 个待完成任务。
- `Done()`：表示一个任务完成，本质是 `Add(-1)`。
- `Wait()`：阻塞直到计数器变成 0。

---

### 33. 为什么 `Add` 通常要放在启动 goroutine 前？

如果把 `Add` 放进 goroutine 内部，主 goroutine 可能先执行 `Wait`，此时计数器还是 0，于是直接返回，导致等待逻辑失效。

正确写法：

```go
wg.Add(1)
go func() {
    defer wg.Done()
}()
```

---

### 34. `WaitGroup` 计数器变成负数会怎样？

会 panic：

```text
sync: negative WaitGroup counter
```

通常是因为 `Done` 调用次数超过 `Add`。

---

### 35. 忘记调用 `Done` 会怎样？

`Wait` 会一直阻塞，可能导致程序卡住或测试超时。

推荐在 goroutine 顶部写：

```go
defer wg.Done()
```

---

### 36. `WaitGroup` 可以复制吗？

不能复制已经使用过的 `WaitGroup`。

如果需要传递，应该传指针：

```go
func run(wg *sync.WaitGroup) {}
```

更好的做法是让创建 goroutine 的函数自己管理 `Add` 和 `Done`，减少到处传 `WaitGroup`。

---

### 37. `WaitGroup` 能收集错误吗？

不能。`WaitGroup` 只负责等待任务结束，不负责错误传播。

如果需要收集错误，可以：

- 用 `Mutex` 保护错误 slice。
- 用 channel 收集结果。
- 使用 `errgroup`。

---

### 38. `WaitGroup` 能取消 goroutine 吗？

不能。`WaitGroup` 只等待，不取消。

取消通常使用 `context.Context` 或关闭 channel。

常见组合：

```text
context 通知 goroutine 退出；
WaitGroup 等待 goroutine 退出完成。
```

---

### 39. `WaitGroup` 和 `errgroup` 有什么区别？

`WaitGroup`：

- 标准库。
- 只等待。
- 不处理错误。
- 不自动取消其他任务。

`errgroup`：

- 来自 `golang.org/x/sync/errgroup`。
- goroutine 返回 error。
- `Wait` 返回第一个错误。
- `WithContext` 可以在出错时取消其他任务。

---

### 40. 如何限制一组任务的最大并发数？

`WaitGroup` 本身不限制并发。可以配合 channel 信号量：

```go
sem := make(chan struct{}, 10)

wg.Add(1)
go func() {
    defer wg.Done()

    sem <- struct{}{}
    defer func() { <-sem }()

    doWork()
}()
```

---

## 五、Once 面试题

### 41. `sync.Once` 用来做什么？

`sync.Once` 用于保证某段函数在并发环境下只执行一次。

典型场景：

- 懒加载配置。
- 初始化全局客户端。
- 初始化只读缓存。
- 单例资源初始化。

---

### 42. `Once` 的基本写法是什么？

```go
var once sync.Once

once.Do(func() {
    initResource()
})
```

无论多少 goroutine 同时调用，`Do` 里的函数只会执行一次。

---

### 43. `Once` 只保证互斥吗？

不只。`Once` 还保证初始化函数完成后的写入对后续调用者可见。

也就是说，`once.Do` 返回后，调用方能看到初始化完成的结果。

---

### 44. `Once` 中的函数 panic 后会重试吗？

不会。即使 `Do` 中的函数 panic，这次调用也会被认为已经执行过，后续 `Do` 不会再次执行该函数。

---

### 45. 初始化函数返回 error 时，`Once` 会重试吗？

不会。

常见写法：

```go
var (
    once sync.Once
    cfg  *Config
    err  error
)

func GetConfig() (*Config, error) {
    once.Do(func() {
        cfg, err = LoadConfig()
    })
    return cfg, err
}
```

如果第一次失败，后续会一直返回同一个错误。

---

### 46. 如果初始化失败后需要重试，应该用 `Once` 吗？

不适合简单使用 `Once`。

可以用 `Mutex` 自己管理状态：

```text
只有初始化成功才标记 loaded；
失败时下次允许重试。
```

---

### 47. `Once` 可以重置吗？

不建议重置已经使用的 `Once`，尤其是在并发场景中。

如果业务需要可重置初始化，应该显式设计状态机，而不是把 `Once` 当成可重置开关。

---

### 48. `Once` 和 `atomic.Bool + CompareAndSwap` 有什么区别？

`Once` 表达“某个函数只执行一次”。

`atomic.Bool + CompareAndSwap` 表达“某个状态从 false 变成 true 的那个 goroutine 执行后续逻辑”。

如果需要读取状态，atomic bool 更直接；如果只关心一次性执行函数，`Once` 更清晰。

---

## 六、Cond 面试题

### 49. `sync.Cond` 用来做什么？

`sync.Cond` 是条件变量，用于让一组 goroutine 等待某个条件成立，并在条件变化时被唤醒。

适合实现：

- 阻塞队列。
- 生产者消费者模型。
- 简易连接池等待。
- 复杂条件等待。

---

### 50. `Cond` 为什么必须配合锁使用？

因为条件通常依赖某些共享状态，而这些共享状态需要锁保护。

例如队列是否为空，取决于 `items`，`items` 必须由锁保护。

---

### 51. `Wait` 做了什么？

`Wait` 会：

```text
释放 cond.L；
阻塞当前 goroutine；
被唤醒后重新获取 cond.L。
```

调用 `Wait` 前必须已经持有 `cond.L`。

---

### 52. 为什么 `Wait` 要放在 for 循环里，而不是 if？

因为被唤醒后条件不一定仍然成立。

可能原因：

- 多个等待者被唤醒，但资源只够一个使用。
- 其他 goroutine 抢先修改了状态。
- Broadcast 唤醒所有人后，每个人都要重新检查条件。

正确写法：

```go
for !condition {
    cond.Wait()
}
```

---

### 53. `Signal` 和 `Broadcast` 有什么区别？

`Signal` 唤醒一个等待者。

`Broadcast` 唤醒所有等待者。

新增一个队列元素通常用 `Signal`；关闭队列或连接池时通常用 `Broadcast`。

---

### 54. `Cond` 和 channel 怎么选择？

channel 更适合表达数据流和事件通知。

`Cond` 更适合多个 goroutine 等待复杂条件。

业务代码中 channel 更常见；底层队列、连接池等结构中 `Cond` 更有价值。

---

### 55. 用 `Cond` 实现阻塞队列要注意什么？

核心点：

- 用锁保护队列和关闭状态。
- `Pop` 中用 `for len(items) == 0 && !closed` 等待。
- `Push` 后 `Signal`。
- `Close` 后 `Broadcast`，唤醒所有等待者退出。

---

## 七、sync.Map 面试题

### 56. `sync.Map` 是什么？

`sync.Map` 是标准库提供的并发安全 map。

常用方法：

- `Store`
- `Load`
- `Delete`
- `LoadOrStore`
- `Range`

---

### 57. `sync.Map` 适合什么场景？

适合：

- 读多写少。
- key 相对稳定。
- 一次写入多次读取。
- 多 goroutine 访问但不想手动加锁。

不适合：

- 写入频繁。
- 需要复杂业务不变量。
- 需要类型安全。
- 需要一致快照遍历。

---

### 58. `sync.Map` 是普通 map 的万能替代品吗？

不是。

很多业务场景中，`map + RWMutex` 更清晰：

- 类型明确。
- 不需要类型断言。
- 可以维护复杂业务规则。
- 更容易控制锁范围。

---

### 59. `LoadOrStore` 的语义是什么？

```go
actual, loaded := m.LoadOrStore(key, value)
```

- 如果 key 已存在，返回已有值，`loaded == true`。
- 如果 key 不存在，存储新值，返回新值，`loaded == false`。

---

### 60. `LoadOrStore` 能防止昂贵初始化函数重复执行吗？

不一定。

如果写成：

```go
value := expensiveInit()
actual, loaded := m.LoadOrStore(key, value)
```

`expensiveInit` 在 `LoadOrStore` 前已经执行，多个 goroutine 仍然可能重复初始化。

如果要合并并发初始化请求，可以使用 `singleflight`。

---

### 61. `sync.Map.Range` 是一致快照吗？

不是。

`Range` 遍历期间其他 goroutine 可以继续 `Store` 或 `Delete`。它不保证看到某一时刻的完整一致快照。

如果业务需要一致快照，应该使用 `map + 锁` 并在锁内复制数据。

---

### 62. `sync.Map` 有什么类型安全问题？

`sync.Map` 的 key 和 value 都是 `any`，读取后通常需要类型断言：

```go
v := value.(string)
```

如果类型不对，会 panic。

可以用泛型封装改善调用侧体验，但内部仍然依赖类型断言。

---

## 八、sync.Pool 面试题

### 63. `sync.Pool` 用来做什么？

`sync.Pool` 用于复用临时对象，减少内存分配和 GC 压力。

常见复用对象：

- `bytes.Buffer`
- 临时 `[]byte`
- 编码解码对象。
- 请求处理中的临时结构体。

---

### 64. `sync.Pool` 的基本用法是什么？

```go
var pool = sync.Pool{
    New: func() any {
        return new(bytes.Buffer)
    },
}

buf := pool.Get().(*bytes.Buffer)
buf.Reset()
defer pool.Put(buf)
```

---

### 65. `sync.Pool` 可以当缓存用吗？

不能。

Pool 中的对象可能在任意 GC 后被清理，不能保证 Put 进去的对象未来一定能 Get 出来。

业务缓存应该使用：

- `map + Mutex/RWMutex`
- `sync.Map`
- Redis
- 专门缓存库

---

### 66. 使用 `sync.Pool` 最容易犯什么错误？

常见错误：

- `Get` 后忘记重置对象状态。
- `Put` 后继续使用对象。
- 把包含敏感数据的对象直接放回池。
- 把 Pool 当业务缓存。
- 没有 benchmark 就引入 Pool。

---

### 67. 为什么 Put 前要清理状态？

对象下次被取出时可能仍然带着旧数据。

例如 `bytes.Buffer` 不 `Reset`，下一次写入可能拼接上次内容，产生脏数据。

---

### 68. Put 后为什么不能继续使用对象？

对象放回池后，其他 goroutine 可能取走并修改它。继续使用可能导致数据竞争或数据混乱。

---

### 69. `sync.Pool` 什么时候收益明显？

收益更可能明显：

- 对象较大。
- 创建频繁。
- 生命周期短。
- 分配发生在热点路径。
- benchmark 显示分配和 GC 压力明显。

对象很小或编译器能栈分配时，Pool 可能收益不明显，甚至增加复杂度。

---

## 九、sync/atomic 面试题

### 70. `sync/atomic` 用来做什么？

`sync/atomic` 用于对单个变量进行并发安全的原子读写或更新。

现代 Go 推荐使用类型化 atomic：

```go
var counter atomic.Int64
counter.Add(1)
value := counter.Load()
```

---

### 71. atomic 适合什么场景？

适合：

- 简单计数器。
- 成功失败数量统计。
- 布尔开关。
- 简单状态标记。
- 热路径上的简单数值状态。

---

### 72. atomic 不适合什么场景？

不适合：

- 多字段一致性。
- 复杂业务不变量。
- map、slice 等复合结构。
- 复杂状态机。
- 需要一组操作作为事务完成。

这些场景通常用 `Mutex` 更清晰。

---

### 73. atomic 和 Mutex 怎么选择？

简单判断：

```text
单个数字或布尔值，操作简单：考虑 atomic。
多个字段或复杂不变量：优先 Mutex。
```

不要为了追求无锁而写出难维护的 atomic 代码。

---

### 74. 为什么不能混用 atomic 访问和普通访问？

如果一个变量有并发访问，并且部分路径用 atomic，部分路径普通读写，普通读写路径仍然可能产生数据竞争。

同一个变量一旦使用 atomic 管理，所有并发访问都应该通过 atomic。

---

### 75. `atomic.Bool` 可以用来做什么？

可以用来表示并发安全的开关状态，例如：

- 服务是否关闭。
- 功能是否开启。
- worker 是否正在运行。

配合 `CompareAndSwap` 可以实现简单的一次性状态迁移。

---

### 76. `CompareAndSwap` 是什么？

CAS 的含义是：

```text
如果当前值等于旧值，就替换成新值并返回 true；
否则不修改并返回 false。
```

示例：

```go
if closed.CompareAndSwap(false, true) {
    // 第一个成功关闭的人执行清理逻辑
}
```

---

### 77. atomic 一定比 Mutex 快吗？

不一定。简单计数场景中 atomic 通常更轻量，但复杂业务场景里 atomic 可能需要 CAS 循环和额外状态管理，代码复杂度和 bug 风险更高。

性能应该通过 benchmark 验证，不能凭感觉。

---

## 十、源码与底层机制面试题

### 78. 如何阅读 `sync` 源码？

建议顺序：

```text
先读官方注释
再读测试
再看核心字段
最后看 runtime 关联
```

优先文件：

- `mutex.go`
- `rwmutex.go`
- `waitgroup.go`
- `once.go`
- `cond.go`
- `map.go`
- `pool.go`

---

### 79. `Mutex` 内部只是一个 bool 吗？

不是。Go 的 `Mutex` 内部有状态字段和等待者管理，并区分快路径和慢路径。

高竞争场景下还涉及等待、唤醒和饥饿模式等机制。

---

### 80. 什么是锁的快路径和慢路径？

快路径：没有竞争时，用很少的操作完成加锁或解锁。

慢路径：存在竞争时，需要进入等待、阻塞、唤醒等更复杂逻辑。

这解释了为什么无竞争锁开销不高，但高竞争锁会明显影响性能。

---

### 81. `RWMutex` 为什么比 `Mutex` 更复杂？

因为它要管理：

- 当前读者数量。
- 是否有写者等待。
- 写者如何阻止新读者进入。
- 写者如何等待已有读者退出。

因此 `RWMutex` 有额外管理成本。

---

### 82. `WaitGroup` 的 panic 条件有哪些？

常见 panic：

- 计数器变成负数。
- 在不合适的时机复用 `WaitGroup`，例如前一次 `Wait` 未完成时又错误调用 `Add`。

实际面试中重点回答：`Add` 和 `Done` 必须严格配对，`Add` 通常在 goroutine 启动前。

---

### 83. `Once` 为什么不能简单用一个 atomic bool 替代？

因为 `Once` 不仅要标记“是否执行过”，还要保证：

- 只有一个 goroutine 执行初始化函数。
- 其他 goroutine 等待初始化完成。
- 初始化结果对后续 goroutine 可见。
- panic 行为符合语义。

简单 atomic bool 容易在初始化未完成时让其他 goroutine 看到错误状态。

---

### 84. `sync.Map` 内部为什么适合读多写少？

`sync.Map` 内部有针对读多场景的优化，例如将读路径尽量做得更轻量，并通过 read/dirty 等结构减少锁竞争。

面试中不需要背实现细节，但要知道它是为特定并发访问模式优化的，不是普通 map 的默认替代品。

---

### 85. `sync.Pool` 和 GC 有什么关系？

Pool 中的对象可能在 GC 时被清理。

这说明：

- Pool 只是一种对象复用优化。
- 不能依赖 Pool 保存业务状态。
- Get 不到对象时应该能重新创建。

---

## 十一、性能分析与调优面试题

### 86. 如何判断 `Mutex` 和 `RWMutex` 哪个更适合？

写 benchmark，至少比较：

- 读多写少。
- 读写均衡。
- 写多读少。

同时考虑代码复杂度。不要只因为“读写锁听起来更高级”就替换。

---

### 87. 如何 benchmark 一个并发安全计数器？

可以写普通 benchmark 和并发 benchmark：

```go
func BenchmarkCounterParallel(b *testing.B) {
    var c Counter
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            c.Add(1)
        }
    })
}
```

关注：

- `ns/op`
- `B/op`
- `allocs/op`

---

### 88. 如何分析锁竞争？

可以使用：

- benchmark 对比吞吐。
- pprof mutex profile。
- block profile。
- goroutine profile。
- trace。

线上场景中还要结合请求延迟、CPU、goroutine 数量等指标。

---

### 89. `go test -race` 和 benchmark 可以一起作为性能依据吗？

不建议用 `-race` 的结果判断性能。

race detector 会插桩，显著改变程序性能。性能测试应在不带 `-race` 的情况下运行。

---

### 90. 为什么不能过早优化锁？

因为并发优化很容易引入复杂度和 bug。

正确顺序：

```text
先正确
再可读
再测试
再测量
最后优化
```

如果没有 benchmark 或线上指标证明锁是瓶颈，通常不要急着拆锁、分片或改 atomic。

---

## 十二、工程设计面试题

### 91. 如何设计一个并发安全本地缓存？

核心设计：

- 用 map 保存数据。
- 用 `Mutex` 或 `RWMutex` 保护 map。
- 所有读写路径都走同一套同步规则。
- 不直接暴露内部 map。
- 如果支持 TTL，需要处理过期删除。
- 如果有后台清理 goroutine，需要设计关闭流程。

可以回答：

```text
简单版本用 map + Mutex 保证正确；
读多写少再考虑 RWMutex；
如果 Get 发现过期要删除，简单版本直接使用写锁。
```

---

### 92. 如何实现带 TTL 的缓存？

数据结构：

```go
type Item struct {
    Value     string
    ExpiresAt time.Time
}
```

关键点：

- `Set` 写入过期时间。
- `Get` 判断是否过期，过期则删除。
- 后台 ticker 定时清理。
- 用 `WaitGroup` 等待后台 goroutine 退出。
- 用 stop channel 或 context 通知关闭。

---

### 93. TTL 缓存关闭时要注意什么？

注意：

- 关闭通知后台 goroutine。
- 等待后台 goroutine 退出。
- Close 最好幂等。
- 不要在持锁期间 `WaitGroup.Wait()`，避免后台 goroutine 也需要同一把锁而死锁。

---

### 94. 如何设计批量任务执行器？

核心点：

- `WaitGroup` 等待全部任务完成。
- channel 信号量限制最大并发。
- `Mutex` 保护错误列表。
- `atomic` 统计成功失败数量。
- `context` 处理取消和超时。

---

### 95. 如何设计一个可关闭 worker pool？

关键点：

- 任务队列用 channel。
- worker 从 channel 读取任务。
- 关闭任务队列通知 worker 退出。
- 使用 `WaitGroup` 等待所有 worker 结束。
- 用 context 传播取消。
- 避免关闭后继续提交任务导致 panic。

---

### 96. 如何设计简易连接池？

核心点：

- 用 slice 保存空闲连接。
- 用 `Mutex` 保护连接列表和关闭状态。
- 无连接可用时等待，可以使用 `Cond` 或 channel。
- `Put` 归还连接后唤醒等待者。
- `Close` 设置关闭状态并唤醒所有等待者。
- Close 要幂等，可以使用 `sync.Once`。

---

### 97. 为什么连接池关闭时要唤醒所有等待者？

如果有 goroutine 正在等待连接，而连接池关闭后不唤醒它们，它们可能永远阻塞。

使用 `Cond` 时通常调用 `Broadcast`。

---

### 98. 如何设计高性能日志缓冲器？

核心点：

- 多 goroutine 写日志，用 `Mutex` 保护内存 buffer 或 entries。
- 达到批量大小或定时触发 flush。
- flush 时锁内交换 slice，锁外执行 IO。
- 用 `sync.Pool` 复用 `bytes.Buffer`。
- Close 时刷出剩余日志并等待后台 goroutine 退出。

---

### 99. 为什么日志 flush 不应该持锁做 IO？

IO 可能很慢。持锁做 IO 会阻塞所有写日志的 goroutine，造成请求延迟升高。

更好的做法：

```text
锁内快速交换数据；
解锁；
锁外格式化和写出。
```

---

### 100. 如何防止 goroutine 泄漏？

原则：

- 谁启动 goroutine，谁负责让它退出。
- 后台 goroutine 要监听 context 或 stop channel。
- 关闭时用 `WaitGroup` 等待退出。
- 不要让 goroutine 永远阻塞在 channel 发送、接收或锁等待上。
- 用 pprof goroutine profile 检查泄漏。

---

## 十三、场景追问题

### 101. 面试官问：为什么不用 channel 替代锁？

可以回答：

channel 更适合表达数据流、任务分发和事件通知；锁更适合保护共享状态。

如果只是保护一个 map 或多个字段的不变量，用 `Mutex/RWMutex` 更直接、更清晰。

---

### 102. 面试官问：为什么不用 `sync.Map` 替代 `map + RWMutex`？

可以回答：

`sync.Map` 适合读多写少、key 稳定、一次写多次读的场景。但它牺牲了类型安全，`Range` 不提供一致快照，也不适合复杂不变量。

如果业务需要清晰的类型和复杂操作，`map + RWMutex` 更合适。

---

### 103. 面试官问：为什么不用 atomic 替代锁？

可以回答：

atomic 适合单个简单变量，比如计数器和布尔开关。如果涉及多个字段的一致性、map/slice、复杂状态机，`Mutex` 更容易保证正确性和可维护性。

---

### 104. 面试官问：你怎么证明这段并发代码是安全的？

可以从四层回答：

```text
第一，明确共享状态是什么。
第二，说明所有读写路径都受同一同步规则保护。
第三，写并发测试并运行 go test -race。
第四，用 benchmark 或 pprof 验证性能判断。
```

注意不要只说“我跑过没问题”。

---

### 105. 面试官问：线上出现请求卡住，你怀疑是锁问题，怎么排查？

可以回答：

- 查看 goroutine profile，看是否大量 goroutine 阻塞在 `sync.(*Mutex).Lock`。
- 查看 mutex profile 或 block profile。
- 检查是否持锁做慢 IO。
- 检查是否存在锁顺序不一致。
- 检查是否忘记解锁。
- 检查近期改动中是否引入新的共享状态。

---

### 106. 面试官问：线上 goroutine 数不断上涨，怎么排查？

可以回答：

- 用 pprof 查看 goroutine profile。
- 看 goroutine 卡在哪里，是 channel、锁、网络 IO 还是定时器。
- 检查后台 goroutine 是否有退出条件。
- 检查 context 是否传递到内部。
- 检查 channel 是否无人关闭或无人接收。
- 修复后压测观察 goroutine 数是否回落。

---

### 107. 面试官问：如果缓存读多写少，你一定会用 RWMutex 吗？

不会直接下结论。

我会先用简单正确的 `Mutex` 或 `RWMutex` 实现，再根据读写比例和 benchmark 判断。如果读临界区很短或写入不少，`RWMutex` 不一定更快。

---

### 108. 面试官问：怎么优雅关闭一组后台 goroutine？

可以回答：

- 使用 context 或 close channel 通知退出。
- 每个 goroutine 在 select 中监听退出信号。
- goroutine 退出前做必要清理。
- 使用 `WaitGroup` 等待全部退出。
- Close 设计成幂等，避免重复关闭 channel panic。

---

### 109. 面试官问：怎么避免重复关闭 channel？

可以使用：

- `sync.Once`
- `atomic.Bool + CompareAndSwap`
- `Mutex` 保护 closed 状态

例如：

```go
var once sync.Once

func Close() {
    once.Do(func() {
        close(ch)
    })
}
```

---

### 110. 面试官问：你怎么评价一段并发代码写得好不好？

可以从这些方面评价：

- 共享状态是否清晰。
- 同步边界是否清晰。
- 锁粒度是否合理。
- 是否有持锁慢操作。
- goroutine 生命周期是否完整。
- 关闭流程是否幂等。
- 是否有 race 测试。
- 是否有 benchmark 支撑性能选择。
- 错误和取消是否能传播。

---

## 十四、高频简答题

### 111. `Mutex` 零值可用吗？

可用。声明后无需初始化：

```go
var mu sync.Mutex
```

---

### 112. `RWMutex` 零值可用吗？

可用。

---

### 113. `WaitGroup` 零值可用吗？

可用。

---

### 114. `Once` 零值可用吗？

可用。

---

### 115. `Cond` 零值可用吗？

不直接使用零值，通常通过 `sync.NewCond(locker)` 创建。

---

### 116. `sync.Map` 零值可用吗？

可用。

```go
var m sync.Map
```

---

### 117. `sync.Pool` 零值可用吗？

可用，但如果不设置 `New`，`Get` 可能返回 `nil`。

---

### 118. `Mutex.Unlock` 未加锁时调用会怎样？

会导致运行时错误。不要对未锁定的 `Mutex` 调用 `Unlock`。

---

### 119. `RWMutex.RUnlock` 未持有读锁时调用会怎样？

会导致运行时错误。

---

### 120. `WaitGroup.Done` 本质是什么？

`Done()` 本质上是 `Add(-1)`。

---

## 十五、综合回答模板

### 121. 如果面试官让你讲讲 Go 里的 sync 包，你怎么回答？

可以这样组织：

```text
sync 包主要提供 goroutine 之间访问共享状态和协调生命周期的同步原语。

Mutex 用于互斥访问共享状态；
RWMutex 用于读多写少场景；
WaitGroup 用于等待一组 goroutine 完成；
Once 用于并发安全的一次性初始化；
Cond 用于等待复杂条件；
Map 是特定场景下的并发安全 map；
Pool 用于复用临时对象，降低分配和 GC 压力；
atomic 用于简单变量的原子操作。

实际工程里，关键不是背 API，而是先识别共享状态，再决定用锁、channel、atomic 还是其他工具，并通过 go test -race、benchmark 和 pprof 验证正确性和性能。
```

---

### 122. 如果面试官让你讲一个 sync 在项目中的使用案例，怎么回答？

可以用 TTL 缓存举例：

```text
我实现过一个本地 TTL 缓存，内部用 map 保存 key 到 value 和过期时间。
因为 HTTP 请求 goroutine 会并发 Get/Set/Delete，后台清理 goroutine 也会定时删除过期 key，所以 map 是共享状态。

简单版本我用 Mutex 保护 map，保证所有读写路径都遵守同一把锁。
Get 发现 key 过期时会删除，所以它也需要写锁。

后台清理 goroutine 通过 ticker 定时执行 DeleteExpired。
Close 时关闭 stop channel 通知后台 goroutine 退出，再用 WaitGroup 等待它结束。
我还补了并发测试，并用 go test -race 检查数据竞争。
```

---

### 123. 如果面试官追问如何优化这个 TTL 缓存，怎么回答？

可以回答：

```text
第一步先保证正确性。
如果 benchmark 显示锁竞争明显，可以考虑优化：

1. 读多写少场景下使用 RWMutex。
2. 对 Get 设计 RLock 快路径，过期删除时再升级为写锁并二次检查。
3. 使用分片锁降低单锁竞争。
4. 加入 singleflight 防止热点 key 过期时并发击穿。
5. 增加容量限制和 LRU 策略。

但每一步都需要 benchmark 或线上指标验证，不能凭感觉优化。
```

---

## 十六、自测清单

如果你能流畅回答下面问题，说明 `sync` 面试准备比较扎实：

- 什么是数据竞争？
- `count++` 为什么不安全？
- `go test -race` 能证明什么，不能证明什么？
- `Mutex` 是否可重入？
- 为什么不能复制 `Mutex`？
- `RWMutex` 一定更快吗？
- `WaitGroup.Add` 为什么放在 goroutine 前？
- `WaitGroup` 能不能收集错误？
- `Once` panic 后是否重试？
- `Cond.Wait` 为什么放在 for 里？
- `sync.Map.Range` 是不是一致快照？
- `sync.Pool` 为什么不能当缓存？
- atomic 和 Mutex 如何选择？
- 如何设计并发安全缓存？
- 如何优雅关闭后台 goroutine？
- 如何排查锁竞争？
- 如何排查 goroutine 泄漏？
- 如何用 benchmark 验证并发组件性能？

真正的面试答案不是“我知道这个 API”，而是：

```text
我知道它解决什么问题；
我知道它不适合什么场景；
我知道如何验证它是正确的；
我知道如何解释工程取舍。
```
