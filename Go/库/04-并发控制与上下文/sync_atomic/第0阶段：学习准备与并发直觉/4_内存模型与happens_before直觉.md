# 4. 内存模型与 happens-before 直觉

本节目标：用后端工程能理解的方式建立 Go 内存模型直觉，知道为什么同步不仅是“互斥”，还关系到“可见性”。

这一节不追求形式化证明，而是帮助你理解一个关键问题：

```text
一个 goroutine 写入的数据，另一个 goroutine 什么时候一定能看到？
```

---

## 一、并发中的可见性问题

看下面代码：

```go
var ready bool
var data string

go func() {
    data = "ok"
    ready = true
}()

for !ready {
}

fmt.Println(data)
```

直觉上，你可能认为：

```text
只要 ready 是 true，data 一定已经是 ok。
```

但在没有同步的情况下，这种推理并不可靠。

问题不只是 CPU 是否重排，也包括编译器优化、缓存可见性和 Go 内存模型允许的执行结果。

---

## 二、happens-before 是什么

你可以先把 happens-before 理解为：

```text
如果 A happens-before B，那么 A 中的写入对 B 可见。
```

并发安全代码需要建立这种顺序关系。

常见能建立 happens-before 的方式包括：

- `Mutex.Unlock` happens-before 后续成功的 `Mutex.Lock`。
- channel 发送 happens-before 对应的接收。
- `WaitGroup.Done` 与 `Wait` 的返回之间存在同步关系。
- `Once.Do` 能保证初始化结果对后续调用可见。
- atomic 操作提供对应的原子性和内存序保证。

---

## 三、锁不仅保护互斥，也保护可见性

很多初学者认为锁只是为了防止同时修改。

实际上，锁还让写入结果在后续加锁者那里可见。

例如：

```go
var mu sync.Mutex
var data string

func write() {
    mu.Lock()
    data = "ok"
    mu.Unlock()
}

func read() string {
    mu.Lock()
    defer mu.Unlock()
    return data
}
```

只要所有读写都遵守同一把锁，`read` 就能看到被正确同步过的数据。

危险写法是：

```go
func unsafeRead() string {
    return data
}
```

如果有的路径加锁，有的路径不加锁，同步规则就被破坏了。

---

## 四、channel 也能建立同步关系

```go
data := ""
done := make(chan struct{})

go func() {
    data = "ok"
    close(done)
}()

<-done
fmt.Println(data)
```

这里 `close(done)` 和 `<-done` 之间建立了同步关系。主 goroutine 在接收到关闭信号后，可以看到之前对 `data` 的写入。

这也是为什么 channel 很适合表达“事件完成”。

---

## 五、不要用 sleep 当同步

错误示例：

```go
go func() {
    data = "ok"
}()

time.Sleep(time.Second)
fmt.Println(data)
```

这段代码可能在你机器上看起来正常，但 `Sleep` 表达的是“等待一段时间”，不是“等待某个写入完成并可见”。

工程代码中应使用：

- channel。
- `WaitGroup`。
- `Mutex`。
- context。
- 其他明确同步机制。

不要用 sleep 赌调度。

---

## 六、内存模型对后端开发的启发

你不需要每天背诵内存模型条文，但必须养成一个习惯：

```text
只要多个 goroutine 共享数据，就要能说清楚它们之间的同步关系。
```

例如：

- 后台 goroutine 更新配置，HTTP goroutine 读取配置：用 `RWMutex` 或 `atomic.Value`。
- 初始化数据库连接，只初始化一次：用 `sync.Once`。
- 主 goroutine 等待 worker 结束：用 `WaitGroup`。
- 任务完成后通知调用方：用 channel 或 context。

---

## 七、本节练习

练习一：写一个用 `time.Sleep` 等待 goroutine 修改变量的例子，然后改成 channel。

练习二：写一个加锁写、加锁读的例子，然后故意加一个不加锁读路径，用 `go test -race` 检测。

练习三：用自己的话解释：

```text
锁保护的不是变量名，而是访问共享状态的规则。
```

---

## 八、本节达标标准

学完本节后，你应该能做到：

- 理解可见性是并发正确性的一部分。
- 能用直觉解释 happens-before。
- 知道锁、channel、WaitGroup、Once、atomic 都能建立某种同步关系。
- 不再用 `time.Sleep` 代替同步。
- 能检查一段后端代码是否存在未受保护的共享读写路径。
