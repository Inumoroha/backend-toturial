# 2. cancelCtx 取消实现

本节目标：理解 `WithCancel` 背后的核心类型 `cancelCtx`。

你可以把 `cancelCtx` 理解成：

```text
一个能被取消，并且能通知子 context 的 context。
```

---

## 一、cancelCtx 大致保存什么

源码细节会随版本变化，但核心字段可以理解为：

```text
父 context。
done channel。
错误原因。
子 context 集合。
锁。
```

它需要这些信息，是因为取消时要做几件事：

- 标记自己已经取消。
- 记录取消原因。
- 关闭 done channel。
- 通知所有子 context。

---

## 二、Done channel 延迟创建

`cancelCtx` 的 done channel 通常不是创建 context 时立刻创建，而是在第一次调用 `Done()` 时再创建。

这样可以减少不必要的分配。

如果某个 context 从来没人监听 `Done()`，就没必要创建 channel。

---

## 三、cancel 做了什么

可以粗略理解为：

```text
加锁。
如果已经取消，直接返回。
记录错误原因。
关闭 done channel。
取消所有子 context。
清空子 context 集合。
解锁。
```

这也解释了为什么多次调用 cancel 是安全的。

第一次会真正取消。

后续调用基本没有额外效果。

---

## 四、为什么需要锁

context 可能被多个 goroutine 同时使用。

例如：

- 一个 goroutine 调用 `Done()`。
- 一个 goroutine 调用 `cancel()`。
- 一个 goroutine 创建子 context。

所以内部需要锁保护状态。

---

## 五、本节达标标准

学完本节后，你应该能够做到：

- 解释 `cancelCtx` 的作用。
- 知道 `Done` channel 可能延迟创建。
- 知道 cancel 会关闭 done 并通知子 context。
- 知道多次调用 cancel 是安全的。

---

## 六、用伪代码理解 cancelCtx

为了避免陷入源码细节，可以先用伪代码理解：

```go
type cancelCtx struct {
	parent   Context
	done     chan struct{}
	err      error
	children map[*cancelCtx]struct{}
	mu       sync.Mutex
}
```

真实源码不一定完全长这样，但心智模型可以先这样建立。

它要解决的问题是：

```text
我自己能被取消。
我能告诉所有孩子一起取消。
我能记录为什么取消。
我能保证并发下只取消一次。
```

---

## 七、为什么 cancel 需要幂等

幂等表示调用多次结果一样。

下面代码是合法的：

```go
ctx, cancel := context.WithCancel(context.Background())
cancel()
cancel()
```

如果 `cancel` 不是幂等的，第二次关闭 done channel 就会 panic。

Go 的 context 需要在复杂并发环境中使用，所以它必须允许多处代码安全地调用 cancel。

例如：

```go
defer cancel()

if err != nil {
	cancel()
	return err
}
```

这里错误分支可能提前 cancel，函数返回时 defer 又会 cancel 一次。这个模式很常见。

---

## 八、Done channel 为什么延迟创建

很多 context 会被传来传去，但未必真的有人监听 `Done()`。

如果每次创建 cancel context 都立刻分配 channel，会增加不必要的内存分配。

所以源码会尽量在第一次需要 Done channel 时再创建。

这是一种性能优化，但不改变外部语义。

你作为调用者只需要知道：

```text
ctx.Done() 可以安全调用。
取消后它会被关闭。
```

---

## 九、取消时 children 的意义

如果父 context 有很多子 context：

```text
request ctx
  -> db ctx
  -> http ctx
  -> cache ctx
```

父 context 取消时，必须通知所有子 context。

所以父 context 需要知道有哪些孩子。

这就是 children 集合的意义。

取消完成后，children 通常会被清理，避免继续持有无用引用。

---

## 十、源码对应工程行为

源码细节最后要回到工程行为：

```go
ctx, cancel := context.WithCancel(r.Context())
defer cancel()
```

这行代码意味着：

- 当前 ctx 是请求 ctx 的子节点。
- 请求取消时，当前 ctx 会取消。
- 你手动 cancel 当前 ctx，不会取消请求 ctx。
- 当前 ctx 取消时，它的子节点也会取消。

这就是为什么 context 能自然表达请求链路。

---

## 十一、复习题

请回答：

1. `cancelCtx` 为什么需要保存 children？
2. 为什么多次调用 cancel 不会 panic？
3. 为什么 `Done` channel 可以延迟创建？
4. 子 context 取消时，为什么最好从父 context 的 children 中移除？
