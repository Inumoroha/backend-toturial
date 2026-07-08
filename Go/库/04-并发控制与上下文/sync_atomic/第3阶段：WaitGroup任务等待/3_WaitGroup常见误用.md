# 3. WaitGroup 常见误用

本节目标：识别 `WaitGroup` 的典型错误，避免任务等待逻辑在高并发下偶发失效。

---

## 一、Add 放在 goroutine 内部

错误：

```go
go func() {
    wg.Add(1)
    defer wg.Done()
    doWork()
}()
wg.Wait()
```

主 goroutine 可能在 `Add(1)` 执行前就调用 `Wait` 并返回。

正确：

```go
wg.Add(1)
go func() {
    defer wg.Done()
    doWork()
}()
```

---

## 二、Done 调用次数过多

错误：

```go
wg.Add(1)
go func() {
    defer wg.Done()
    wg.Done()
}()
```

计数器会变成负数，程序 panic：

```text
sync: negative WaitGroup counter
```

---

## 三、Done 调用次数过少

错误：

```go
wg.Add(1)
go func() {
    if somethingWrong {
        return
    }
    wg.Done()
}()
wg.Wait()
```

如果提前返回，`Done` 不执行，`Wait` 永远阻塞。

推荐：

```go
go func() {
    defer wg.Done()
    if somethingWrong {
        return
    }
}()
```

---

## 四、复制 WaitGroup

不要复制已经使用的 `WaitGroup`。

错误风险：

```go
func run(wg sync.WaitGroup) {
    wg.Done()
}
```

应该传指针：

```go
func run(wg *sync.WaitGroup) {
    wg.Done()
}
```

更好的方式是让创建 goroutine 的函数自己管理 `Add` 和 `Done`，减少把 `WaitGroup` 到处传。

---

## 五、以为 WaitGroup 会处理错误

错误心智：

```text
WaitGroup 等到了所有任务，所以任务都成功了。
```

`WaitGroup` 只说明任务结束，不说明任务成功。

错误要自己收集，或者使用 `errgroup`。

---

## 六、本节练习

1. 写一个 `Add` 在 goroutine 内部的例子。
2. 写一个计数器为负数的例子。
3. 写一个忘记 `Done` 导致测试超时的例子。
4. 写一个复制 `WaitGroup` 的例子，并解释风险。
5. 把错误收集加入批量任务执行器。

---

## 七、本节达标标准

学完本节后，你应该能做到：

- 避免 `WaitGroup` 计数器错误。
- 知道不能复制已使用的 `WaitGroup`。
- 知道等待完成不等于执行成功。
