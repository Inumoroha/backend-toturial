# 3. WaitGroup、Once、Map、Pool 源码直觉

本节目标：理解几个常用同步原语的大致内部设计思路，知道从源码中提炼哪些工程结论。

---

## 一、WaitGroup

`WaitGroup` 内部核心是计数状态。

阅读重点：

- `Add` 如何修改计数器。
- 计数器变负数为什么 panic。
- `Wait` 如何等待计数器归零。
- 为什么不能复制已使用的 `WaitGroup`。

工程结论：

```text
Add 和 Done 必须严格配对；
Add 通常放在 goroutine 启动前；
WaitGroup 只等待，不传递错误。
```

---

## 二、Once

`Once` 的核心是：

```text
是否已经执行；
执行过程的互斥保护；
执行完成后的可见性保证。
```

阅读重点：

- `Do` 如何保证函数只执行一次。
- 为什么不能简单用 atomic 标记替代完整逻辑。
- panic 后为什么被认为已经执行。

工程结论：

```text
Once 适合只成功或失败一次的初始化；
需要失败重试时不要直接用 Once。
```

---

## 三、sync.Map

`sync.Map` 的内部设计比普通 map 加锁复杂。

阅读重点：

- read map 和 dirty map 的思路。
- `Load` 为什么适合读多场景。
- `Store`、`LoadOrStore` 如何处理 dirty 状态。
- `Range` 为什么不是一致快照。

工程结论：

```text
sync.Map 是为特定场景优化的结构；
普通业务 map 不要默认换成 sync.Map。
```

---

## 四、sync.Pool

`Pool` 与 P、本地池、GC 有关系。

阅读重点：

- `Get` 取不到时如何调用 `New`。
- `Put` 后对象为什么不保证长期存在。
- GC 为什么可能清理池中对象。

工程结论：

```text
Pool 是性能优化工具，不是缓存；
对象状态必须清理；
业务正确性不能依赖 Pool 命中。
```

---

## 五、本节练习

1. 阅读 `waitgroup.go`，记录 panic 条件。
2. 阅读 `once.go`，解释为什么 panic 后不重试。
3. 阅读 `map.go` 注释，整理适用场景。
4. 阅读 `pool.go` 注释，整理不能当缓存的原因。

---

## 六、本节达标标准

学完本节后，你应该能做到：

- 从源码提炼 API 使用限制。
- 理解 `sync.Map` 和 `sync.Pool` 的适用边界。
- 能把源码结论写进自己的工程规范。
