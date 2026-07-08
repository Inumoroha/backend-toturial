# 4. WaitGroup 与 errgroup

本节目标：理解 `WaitGroup` 和 `errgroup` 的区别，知道什么时候应该升级到 `errgroup`。

`WaitGroup` 是标准库同步原语，`errgroup` 是 `golang.org/x/sync/errgroup` 提供的工具。

---

## 一、WaitGroup 适合什么

适合：

- 等待多个 goroutine 完成。
- 不关心错误，或错误可以自己收集。
- 不需要一个任务失败后取消其他任务。
- 想保持最少依赖。

示例：

```go
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
```

---

## 二、errgroup 适合什么

适合：

- goroutine 返回错误。
- 任何一个任务失败后，希望整体返回错误。
- 希望配合 context 取消其他任务。

示例：

```go
g, ctx := errgroup.WithContext(ctx)

for _, url := range urls {
    url := url
    g.Go(func() error {
        req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
        if err != nil {
            return err
        }
        resp, err := http.DefaultClient.Do(req)
        if err != nil {
            return err
        }
        defer resp.Body.Close()
        return nil
    })
}

if err := g.Wait(); err != nil {
    return err
}
```

---

## 三、选择建议

使用 `WaitGroup`：

```text
只需要等待完成；
错误处理很简单；
任务之间不需要取消联动。
```

使用 `errgroup`：

```text
任务会返回错误；
第一个错误就应该让整体失败；
需要 context 取消其他任务。
```

---

## 四、后端场景对比

适合 `WaitGroup`：

- 启动多个后台 worker，然后优雅关闭时等待退出。
- 并发刷新多个本地统计项，错误单独记录日志。
- 测试中等待多个 goroutine 完成。

适合 `errgroup`：

- 一个请求中并发调用多个下游服务，任意失败则请求失败。
- 批量处理文件，任意一个关键步骤失败则取消整体。
- 并发查询多个资源，并希望超时统一取消。

---

## 五、本节练习

1. 用 `WaitGroup` 并发请求多个 URL，收集错误。
2. 用 `errgroup` 重写。
3. 加入 context 超时。
4. 比较两版代码可读性。

---

## 六、本节达标标准

学完本节后，你应该能做到：

- 清楚说明 `WaitGroup` 不负责错误传播。
- 知道 `errgroup` 的适用场景。
- 能在后端请求扇出场景中选择合适工具。
