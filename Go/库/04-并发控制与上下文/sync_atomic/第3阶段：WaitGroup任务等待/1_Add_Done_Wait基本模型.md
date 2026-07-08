# 1. Add、Done、Wait 基本模型

本节目标：掌握 `WaitGroup` 的三个核心方法，并理解计数器模型。

---

## 一、WaitGroup 的计数器

你可以把 `WaitGroup` 理解成一个并发安全计数器：

```text
Add(n)：增加 n 个未完成任务
Done()：完成 1 个任务
Wait()：等待未完成任务数变成 0
```

`Done()` 本质上等价于 `Add(-1)`。

---

## 二、标准写法

```go
func RunJobs(jobs []Job) {
    var wg sync.WaitGroup

    for _, job := range jobs {
        job := job

        wg.Add(1)
        go func() {
            defer wg.Done()
            job.Run()
        }()
    }

    wg.Wait()
}
```

注意：

- `wg.Add(1)` 在 `go func` 之前。
- `defer wg.Done()` 放在 goroutine 顶部。
- 循环变量 `job := job` 避免闭包捕获问题。

---

## 三、为什么 Add 要放在 goroutine 前

错误示例：

```go
go func() {
    wg.Add(1)
    defer wg.Done()
    doWork()
}()

wg.Wait()
```

风险是主 goroutine 可能先执行 `Wait`，此时计数器还是 0，于是直接返回。后续 goroutine 再 `Add(1)` 就破坏了等待语义。

正确做法：

```go
wg.Add(1)
go func() {
    defer wg.Done()
    doWork()
}()
```

---

## 四、Done 必须执行

如果 goroutine 中途 return 或 panic，`Done` 没有执行，`Wait` 会一直阻塞。

推荐：

```go
go func() {
    defer wg.Done()

    if err := doWork(); err != nil {
        return
    }
}()
```

如果需要处理 panic，可以额外 recover，但不要吞掉业务上必须暴露的错误。

---

## 五、等待不是取消

`Wait` 只是等待任务结束，不会主动让任务停止。

如果任务可能很久不结束，应使用 `context`：

```go
func worker(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return
        default:
            // work
        }
    }
}
```

---

## 六、本节练习

1. 启动 10 个 goroutine 打印任务编号，并用 `WaitGroup` 等待。
2. 把 `Add` 放进 goroutine 内部，观察它为什么有风险。
3. 故意少调用一次 `Done`，用 `go test -timeout=3s` 观察卡住。
4. 给 worker 加上 `context` 取消。

---

## 七、本节达标标准

学完本节后，你应该能做到：

- 正确解释 `Add`、`Done`、`Wait`。
- 知道 `Add` 应放在启动 goroutine 前。
- 知道 `WaitGroup` 只等待，不取消。
- 能避免常见计数器错误。
