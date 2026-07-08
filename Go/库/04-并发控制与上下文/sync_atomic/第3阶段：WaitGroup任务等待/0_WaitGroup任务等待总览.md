# 0. WaitGroup 任务等待总览

本阶段目标：掌握 `sync.WaitGroup` 的标准使用方式，能正确等待一组 goroutine 完成，并理解它不负责错误传播和取消。

`WaitGroup` 解决的问题是：

```text
启动了一组 goroutine，主 goroutine 如何等待它们全部结束？
```

---

## 一、基本写法

```go
var wg sync.WaitGroup

wg.Add(1)
go func() {
    defer wg.Done()
    // do work
}()

wg.Wait()
```

核心规则：

- `Add` 通常在启动 goroutine 前调用。
- `Done` 通常放在 goroutine 内部，并用 `defer`。
- `Wait` 阻塞直到计数器归零。

---

## 二、本阶段文件安排

1. `1_Add_Done_Wait基本模型.md`
2. `2_并发任务收集结果.md`
3. `3_WaitGroup常见误用.md`
4. `4_WaitGroup与errgroup.md`

---

## 三、WaitGroup 不做什么

`WaitGroup` 只负责等待，不负责：

- 限制并发数。
- 收集错误。
- 取消其他 goroutine。
- 传递结果。
- 超时控制。

这些能力通常要配合：

- channel。
- `context.Context`。
- `errgroup.Group`。
- `Mutex`。

---

## 四、本阶段达标标准

学完本阶段后，你应该能做到：

- 正确写出 `Add`、`Done`、`Wait`。
- 避免计数器为负数。
- 避免把 `Add` 放进 goroutine 内部。
- 能并发执行任务并收集结果。
- 知道什么时候应该用 `errgroup`。
