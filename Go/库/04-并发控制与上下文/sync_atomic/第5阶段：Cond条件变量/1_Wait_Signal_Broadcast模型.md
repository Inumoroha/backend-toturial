# 1. Wait、Signal、Broadcast 模型

本节目标：掌握 `sync.Cond` 的三个核心动作：等待、唤醒一个、唤醒全部。

---

## 一、Cond 由锁创建

```go
cond := sync.NewCond(&sync.Mutex{})
```

`Cond` 必须配合锁使用。锁保护的是条件对应的共享状态。

例如：

```go
type Queue struct {
    cond  *sync.Cond
    items []string
}
```

`items` 是否为空，就是等待条件。

---

## 二、Wait 的行为

`Wait` 做了三件事：

```text
释放 cond.L；
阻塞当前 goroutine；
被唤醒后重新获得 cond.L。
```

所以调用 `Wait` 前必须先持有锁：

```go
cond.L.Lock()
for len(items) == 0 {
    cond.Wait()
}
item := items[0]
cond.L.Unlock()
```

---

## 三、Signal

`Signal` 唤醒一个等待者：

```go
cond.Signal()
```

适合场景：

- 队列新增一个元素。
- 只需要一个消费者处理。

---

## 四、Broadcast

`Broadcast` 唤醒所有等待者：

```go
cond.Broadcast()
```

适合场景：

- 状态发生全局变化。
- 服务关闭，所有等待者都应该退出。
- 配置刷新，多个等待者都需要重新检查条件。

---

## 五、为什么必须重新检查条件

即使被唤醒，也不代表条件一定成立。

例如队列里只放入一个元素，但有多个消费者被唤醒。第一个消费者拿走元素后，后面的消费者再次检查时队列已经为空。

所以必须：

```go
for len(q.items) == 0 {
    q.cond.Wait()
}
```

---

## 六、本节练习

1. 写一个等待 `ready == true` 的例子。
2. 用 `Signal` 唤醒一个等待者。
3. 用 `Broadcast` 唤醒多个等待者。
4. 把 `for` 改成 `if`，分析为什么不可靠。

---

## 七、本节达标标准

学完本节后，你应该能做到：

- 解释 `Wait` 的释放锁和重新加锁过程。
- 区分 `Signal` 和 `Broadcast`。
- 坚持用 `for` 检查等待条件。
