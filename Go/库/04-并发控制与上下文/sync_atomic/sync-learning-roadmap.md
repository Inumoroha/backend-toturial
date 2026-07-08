# Go 后端工程师 sync 系统学习路线图

> 目标：从“会用 sync 包”进阶到“能在后端工程中正确设计并发安全组件”，理解常见并发 bug、性能取舍和源码实现思路。

## 0. 学习前置要求

在系统学习 `sync` 之前，建议先掌握：

- Go 基础语法：struct、interface、method、error handling
- goroutine 和 channel 的基本用法
- Go 内存模型的基本概念：并发读写、happens-before、数据竞争
- `go test`、`go test -race`、benchmark 的基本使用

推荐先能回答这些问题：

- goroutine 和 OS thread 是什么关系？
- channel 适合解决什么问题？
- 什么是 data race？
- 为什么并发 map 读写会 panic？

## 1. 总体学习路径

建议按下面顺序学习：

1. 并发基础与竞态检测
2. `sync.Mutex` 和 `sync.RWMutex`
3. `sync.WaitGroup`
4. `sync.Once`
5. `sync.Cond`
6. `sync.Map`
7. `sync.Pool`
8. `sync/atomic` 与 lock-free 思维
9. 源码阅读与性能分析
10. 后端实战项目

不要一开始就背 API。更好的方式是：先遇到问题，再学习对应工具，然后用测试验证你的理解。

## 2. 第一阶段：并发基础与数据竞争

### 学习目标

- 理解 goroutine 并发执行带来的不确定性
- 知道什么是临界区、共享变量、数据竞争
- 会使用 `go test -race` 定位并发问题

### 必学内容

- goroutine 生命周期
- 并发读写共享变量的问题
- Go race detector
- Go map 并发读写风险
- happens-before 基础概念

### 练习

1. 写一个程序，让 1000 个 goroutine 对同一个整数执行 `count++`，观察结果是否正确。
2. 使用 `go test -race` 检测上面的程序。
3. 把普通 map 放到多个 goroutine 中并发读写，观察 panic 或 race。

### 你需要掌握的判断

- 什么时候可以无锁？
- 什么时候必须加锁？
- channel 能不能替代锁？什么时候不适合？

## 3. 第二阶段：Mutex

### 学习目标

- 熟练使用 `sync.Mutex`
- 理解互斥锁保护的是“临界区”，不是某个变量本身
- 掌握锁的粒度设计

### 必学 API

```go
var mu sync.Mutex

mu.Lock()
defer mu.Unlock()
```

### 核心知识点

- 临界区保护
- `defer mu.Unlock()` 的优缺点
- 锁粒度：粗锁与细锁
- 死锁产生的常见原因
- 不要复制已经使用过的 Mutex
- 不要在持锁期间做慢操作，比如网络请求、磁盘 IO、长时间计算

### 练习

1. 实现一个并发安全计数器 `SafeCounter`。
2. 实现一个并发安全缓存 `SafeCache`，支持 `Get`、`Set`、`Delete`。
3. 故意制造一个死锁案例，然后解释为什么会死锁。

### 工程重点

后端开发中，`Mutex` 常用于保护：

- 本地缓存
- 连接状态
- 限流计数
- session 状态
- 后台任务状态

## 4. 第三阶段：RWMutex

### 学习目标

- 理解读多写少场景
- 掌握 `RLock`、`RUnlock`、`Lock`、`Unlock`
- 能判断是否真的需要 `RWMutex`

### 必学 API

```go
var mu sync.RWMutex

mu.RLock()
defer mu.RUnlock()

mu.Lock()
defer mu.Unlock()
```

### 核心知识点

- 多个 reader 可以同时持有读锁
- writer 需要独占锁
- `RWMutex` 不一定比 `Mutex` 快
- 锁竞争严重时需要 benchmark，而不是凭感觉优化

### 练习

1. 将 `SafeCache` 改造成 `RWMutex` 版本。
2. 写 benchmark 对比 `Mutex` 和 `RWMutex` 在读多写少、读写均衡、写多读少场景下的性能。
3. 分析什么时候 `RWMutex` 反而更慢。

## 5. 第四阶段：WaitGroup

### 学习目标

- 掌握等待多个 goroutine 完成的标准写法
- 避免 `Add`、`Done`、`Wait` 的常见误用

### 必学 API

```go
var wg sync.WaitGroup

wg.Add(1)
go func() {
    defer wg.Done()
    // do work
}()
wg.Wait()
```

### 核心知识点

- `Add` 应该在启动 goroutine 前调用
- `Done` 本质是 `Add(-1)`
- 计数器不能变成负数
- `WaitGroup` 只负责等待，不负责错误传播和取消
- 需要错误处理时，优先了解 `errgroup`

### 练习

1. 并发请求多个 URL，等待全部请求完成。
2. 实现一个批量任务执行器。
3. 给任务执行器加上错误收集。
4. 对比 `WaitGroup` 和 `errgroup.Group` 的适用场景。

## 6. 第五阶段：Once

### 学习目标

- 掌握只执行一次初始化逻辑
- 理解并发安全的单例初始化

### 必学 API

```go
var once sync.Once

once.Do(func() {
    // init once
})
```

### 核心知识点

- `Do` 中的函数只会执行一次
- 即使多个 goroutine 同时调用，也只会有一个执行初始化逻辑
- 如果初始化函数 panic，这次调用也会被认为已经执行过
- `Once` 适合初始化资源，不适合需要重试的复杂流程

### 练习

1. 实现一个懒加载配置读取器。
2. 实现一个数据库连接单例模拟器。
3. 思考：如果初始化失败，需要重试，`sync.Once` 是否合适？

## 7. 第六阶段：Cond

### 学习目标

- 理解条件变量的用途
- 知道为什么大多数业务代码很少直接使用 `sync.Cond`
- 能读懂使用 `Cond` 的底层或高性能代码

### 必学 API

```go
cond := sync.NewCond(&sync.Mutex{})

cond.L.Lock()
for !condition {
    cond.Wait()
}
cond.L.Unlock()

cond.Signal()
cond.Broadcast()
```

### 核心知识点

- `Wait` 会释放锁，醒来后重新获取锁
- 必须用 `for` 检查条件，不能只用 `if`
- `Signal` 唤醒一个等待者
- `Broadcast` 唤醒全部等待者
- 很多场景可以用 channel 更直观地表达

### 练习

1. 实现一个阻塞队列。
2. 实现生产者消费者模型。
3. 使用 channel 再实现一版，对比可读性。

## 8. 第七阶段：Map

### 学习目标

- 理解 `sync.Map` 的适用场景
- 知道它不是普通 map 的万能替代品

### 必学 API

```go
var m sync.Map

m.Store("key", "value")
value, ok := m.Load("key")
m.Delete("key")
actual, loaded := m.LoadOrStore("key", "value")
m.Range(func(key, value any) bool {
    return true
})
```

### 核心知识点

- `sync.Map` 适合读多写少、key 相对稳定的场景
- 普通 `map + RWMutex` 在很多业务场景下更清晰
- `Range` 期间不保证看到一个完全一致的快照
- 泛型封装可以改善类型安全

### 练习

1. 用 `sync.Map` 实现一个本地缓存。
2. 用 `map + RWMutex` 实现同样功能。
3. 写 benchmark 比较两者在不同读写比例下的表现。

## 9. 第八阶段：Pool

### 学习目标

- 理解对象复用和 GC 压力
- 知道 `sync.Pool` 的生命周期和不确定性

### 必学 API

```go
var pool = sync.Pool{
    New: func() any {
        return make([]byte, 0, 1024)
    },
}

buf := pool.Get().([]byte)
buf = buf[:0]
defer pool.Put(buf)
```

### 核心知识点

- `sync.Pool` 用于临时对象复用
- Pool 中的对象可能在任意 GC 后被清理
- 不能把业务状态的正确性依赖在 Pool 上
- 放回 Pool 前要重置对象状态
- 适合复用 buffer、临时结构体、编码解码对象

### 练习

1. 用 `sync.Pool` 复用 `bytes.Buffer`。
2. 写 benchmark 对比使用 Pool 和不使用 Pool 的内存分配。
3. 故意忘记 reset buffer，观察脏数据问题。

## 10. 第九阶段：sync/atomic

### 学习目标

- 理解原子操作适合的场景
- 知道 atomic 和 mutex 的取舍
- 会使用 typed atomic 类型

### 推荐 API

```go
var counter atomic.Int64

counter.Add(1)
value := counter.Load()
counter.Store(0)
```

### 核心知识点

- atomic 适合简单数值状态
- 复杂不变量更适合用 Mutex
- atomic 代码可读性和维护成本通常更高
- 不要混用普通读写和 atomic 读写访问同一个变量

### 练习

1. 用 `atomic.Int64` 实现并发安全计数器。
2. 对比 `MutexCounter` 和 `AtomicCounter` 的 benchmark。
3. 实现一个简单的开关状态：`enabled atomic.Bool`。

## 11. 源码阅读路线

阅读源码时不要一口气啃完整包，建议按使用频率排序。

### 第一轮：读注释和测试

优先看：

- `src/sync/mutex.go`
- `src/sync/rwmutex.go`
- `src/sync/waitgroup.go`
- `src/sync/once.go`
- `src/sync/map.go`
- `src/sync/pool.go`

重点不是记住每一行，而是理解：

- 这个类型解决什么问题？
- 官方注释提醒了哪些限制？
- panic 条件有哪些？
- 哪些行为被文档保证，哪些没有保证？

### 第二轮：理解内部状态

重点关注：

- `Mutex` 的 state 字段
- 正常模式与饥饿模式
- `WaitGroup` 的计数器设计
- `Once` 如何保证只执行一次
- `sync.Map` 的 read map 和 dirty map
- `sync.Pool` 与 GC 的关系

### 第三轮：结合 runtime

进阶阅读：

- semaphore 相关实现
- goroutine park/unpark
- scheduler 与阻塞唤醒
- race detector 相关注解

这一阶段不要求一次读懂，能建立大致地图即可。

## 12. 后端实战项目

### 项目一：并发安全本地缓存

功能要求：

- `Set(key, value, ttl)`
- `Get(key)`
- `Delete(key)`
- 后台定时清理过期 key
- 支持并发访问
- 使用 `go test -race` 验证

需要用到：

- `Mutex` 或 `RWMutex`
- `WaitGroup`
- goroutine
- ticker
- context cancel

### 项目二：批量任务执行器

功能要求：

- 支持提交多个任务
- 限制最大并发数
- 等待全部任务完成
- 收集错误
- 支持 context 取消

需要用到：

- `WaitGroup`
- channel
- `Mutex` 保护错误列表
- 可选：`atomic` 统计成功/失败数量

### 项目三：简易连接池

功能要求：

- 初始化固定数量连接
- `Get` 获取连接
- `Put` 归还连接
- 支持超时获取
- 支持关闭连接池

需要用到：

- `Mutex`
- `Cond` 或 channel
- `Once` 关闭资源
- context timeout

### 项目四：高性能日志缓冲器

功能要求：

- 多 goroutine 写日志
- 内部缓冲批量刷盘
- 使用 `sync.Pool` 复用 buffer
- 支持优雅关闭

需要用到：

- `Mutex`
- `WaitGroup`
- `Pool`
- channel

## 13. 常见错误清单

学习过程中重点避开这些坑：

- 忘记 `Unlock`
- 在多个路径提前 `return` 时没有释放锁
- 持锁期间调用外部未知代码
- 持锁期间执行慢 IO
- 复制已经使用过的 `Mutex`、`WaitGroup`、`Once`
- `WaitGroup.Add` 放在 goroutine 内部
- `WaitGroup` 计数器变成负数
- 对 `sync.Map.Range` 的一致性有错误期待
- 把 `sync.Pool` 当成可靠缓存
- 以为通过一次 `go test` 就能证明没有并发 bug
- 混用 atomic 访问和普通访问

## 14. 推荐学习节奏

### 第 1 周：打基础

- 学习 goroutine、race detector、数据竞争
- 完成并发计数器练习
- 学会使用 `go test -race`

### 第 2 周：掌握常用锁

- 学习 `Mutex`、`RWMutex`
- 实现并发安全缓存
- 写 benchmark 比较不同锁策略

### 第 3 周：任务编排

- 学习 `WaitGroup`、`Once`
- 实现批量任务执行器
- 对比 `WaitGroup` 和 `errgroup`

### 第 4 周：进阶同步原语

- 学习 `Cond`、`Map`、`Pool`
- 完成阻塞队列、本地缓存、buffer pool 练习

### 第 5 周：atomic 与性能分析

- 学习 `sync/atomic`
- 写 benchmark 和 pprof
- 对比锁、channel、atomic 的适用场景

### 第 6 周：源码和项目整合

- 阅读核心源码注释
- 完成一个完整后端并发组件
- 为项目补充 race test、benchmark 和文档

## 15. 面试与工程能力检查

你可以用这些问题检验学习效果：

- `Mutex` 和 `RWMutex` 的区别是什么？
- `RWMutex` 一定比 `Mutex` 快吗？
- `WaitGroup.Add` 为什么通常要放在 goroutine 启动前？
- `sync.Once` 遇到 panic 后还能再次执行吗？
- `sync.Map` 适合哪些场景？什么时候不适合？
- `sync.Pool` 里的对象什么时候会被清理？
- atomic 和 mutex 应该如何选择？
- 如何定位一个数据竞争问题？
- 如何设计一个并发安全缓存？
- 如何优雅关闭一组后台 goroutine？

## 16. 推荐代码目录结构

建议你在当前目录下按主题建立练习：

```text
sync/
├── 01-race/
├── 02-mutex/
├── 03-rwmutex/
├── 04-waitgroup/
├── 05-once/
├── 06-cond/
├── 07-map/
├── 08-pool/
├── 09-atomic/
└── projects/
    ├── safe-cache/
    ├── task-runner/
    ├── conn-pool/
    └── log-buffer/
```

每个目录建议包含：

- `main.go`：最小可运行示例
- `*_test.go`：并发测试
- `*_bench_test.go`：性能测试
- `README.md`：记录踩坑和总结

## 17. 最终目标

学完这条路线后，你应该能做到：

- 熟练使用 `sync` 包解决后端并发问题
- 能写出通过 `go test -race` 的并发安全代码
- 能根据场景选择 `Mutex`、`RWMutex`、channel、atomic 或 `sync.Map`
- 能读懂常见并发组件的源码
- 能用 benchmark 和 pprof 验证性能判断
- 能在面试中清晰解释 Go 并发同步机制

真正的关键不是记住每个 API，而是形成一个判断框架：

> 共享状态是什么？谁会读？谁会写？并发边界在哪里？正确性靠什么保证？性能瓶颈是否被测量过？

带着这五个问题写 Go 后端并发代码，你会稳很多。
