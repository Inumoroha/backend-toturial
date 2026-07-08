# 0. Cond 条件变量总览

本阶段目标：理解 `sync.Cond` 的用途，能读懂和实现基于条件变量的阻塞队列、生产者消费者模型，并知道为什么业务代码中它不如 channel 常见。

`Cond` 解决的问题是：

```text
一些 goroutine 等待某个条件成立；
另一些 goroutine 修改状态后通知等待者。
```

---

## 一、基本模型

```go
cond := sync.NewCond(&sync.Mutex{})

cond.L.Lock()
for !condition {
    cond.Wait()
}
cond.L.Unlock()
```

通知：

```go
cond.Signal()
cond.Broadcast()
```

---

## 二、本阶段文件安排

1. `1_Wait_Signal_Broadcast模型.md`
2. `2_阻塞队列实现.md`
3. `3_Cond与channel对比.md`

---

## 三、为什么 Wait 要放在 for 里

等待条件时必须写：

```go
for !condition {
    cond.Wait()
}
```

不要写：

```go
if !condition {
    cond.Wait()
}
```

原因是 goroutine 被唤醒后，条件不一定仍然成立。可能有多个等待者同时被唤醒，也可能条件已经被其他 goroutine 消耗。

---

## 四、适合场景

适合：

- 需要多个 goroutine 等待复杂条件。
- 需要唤醒一个或全部等待者。
- 自己实现队列、连接池等底层结构。

业务代码中很多场景可以优先用 channel，因为 channel 更直接表达数据传递和关闭信号。

---

## 五、本阶段达标标准

学完本阶段后，你应该能做到：

- 正确写出 `Wait`、`Signal`、`Broadcast`。
- 知道 `Wait` 会释放锁，醒来后重新获得锁。
- 用 `Cond` 实现阻塞队列。
- 能判断某个场景更适合 `Cond` 还是 channel。
