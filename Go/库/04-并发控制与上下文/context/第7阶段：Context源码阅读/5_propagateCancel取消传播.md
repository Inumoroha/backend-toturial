# 5. propagateCancel 取消传播

本节目标：理解父 context 取消时，子 context 如何收到通知。

取消传播是 `context` 最关键的内部机制之一。

你已经知道规则：

```text
父取消会影响子。
子取消不会影响父。
```

源码中负责建立这种关系的核心逻辑可以理解为 `propagateCancel`。

---

## 一、创建子 context 时发生什么

调用：

```go
child, cancel := context.WithCancel(parent)
```

内部不仅创建了 child。

还会把 child 和 parent 建立关系。

这样 parent 被取消时，child 也能取消。

---

## 二、如果父 context 已经取消

如果 parent 已经取消，再创建 child：

```go
parentCancel()
child, _ := context.WithCancel(parent)
```

child 会很快处于取消状态。

因为它不能比已取消的父 context 更“活着”。

---

## 三、如果父是 cancelCtx

如果父 context 本身就是可取消 context，子 context 通常会登记到父的 children 集合里。

父取消时，会遍历 children，逐个取消。

可以理解为：

```text
parent.children[child] = true
```

---

## 四、如果父不是 cancelCtx

如果父 context 不是标准可取消实现，但它的 Done channel 会关闭，内部可能需要启动一个 goroutine 监听父 Done。

当父 Done 关闭时，再取消 child。

这是为了兼容自定义 context 实现。

---

## 五、取消后为什么要移除子节点

子 context 自己取消时，通常需要从父的 children 集合里移除。

这样可以避免父 context 长期保存已经没用的子 context 引用。

这也是释放资源的一部分。

---

## 六、本节达标标准

学完本节后，你应该能够做到：

- 解释父子 context 如何建立取消关系。
- 知道父取消会遍历取消子 context。
- 知道父已取消时，新子 context 也会取消。
- 理解子 context 取消后需要从父集合移除。

---

## 七、用流程图理解传播

创建子 context 时，可以这样理解：

```text
WithCancel(parent)
  -> 创建 child cancelCtx
  -> 检查 parent 是否已经取消
  -> 如果已取消，立即取消 child
  -> 如果未取消，把 child 挂到 parent 下面
  -> 返回 child 和 cancel 函数
```

父取消时：

```text
parent cancel
  -> 关闭 parent.Done
  -> 遍历 children
  -> child cancel
  -> child 再取消自己的 children
```

这就是树状传播。

---

## 八、为什么取消传播不是双向的

如果子 context 取消会反向取消父 context，会出现很多问题。

例如：

```text
HTTP request ctx
  -> db query ctx
  -> cache query ctx
```

如果数据库子查询 300ms 超时就反向取消整个 HTTP 请求，那么 cache 查询也会被迫停止。

但业务可能希望：

```text
数据库失败后用缓存兜底。
```

所以 context 的默认规则是单向传播：

```text
父影响子。
子不影响父。
```

如果业务确实希望某个子任务失败后取消其他兄弟任务，应该在共同父层控制，例如使用 `errgroup.WithContext`。

---

## 九、运行实验：兄弟不互相影响

```go
parent := context.Background()

left, leftCancel := context.WithCancel(parent)
right, rightCancel := context.WithCancel(parent)
defer rightCancel()

leftCancel()

fmt.Println("left:", left.Err())
fmt.Println("right:", right.Err())
```

输出中：

```text
left: context canceled
right: <nil>
```

这说明兄弟 context 之间不会直接传播取消。

---

## 十、自定义 Context 的兼容

标准库允许你自己实现 `Context` 接口。

如果父 context 不是标准库内部的 `cancelCtx`，但它的 `Done()` 会关闭，子 context 仍然应该能感知。

所以源码中会有兼容逻辑：可能通过 goroutine 监听父 Done，然后取消子 context。

这部分细节第一次阅读可以粗略理解，不必深挖。

核心结论是：

```text
标准库既优化了常见父子关系，也兼容了自定义 Context。
```

---

## 十一、工程映射

当你写：

```go
ctx, cancel := context.WithTimeout(r.Context(), 300*time.Millisecond)
defer cancel()
```

你就在请求 context 下面挂了一个子 context。

如果客户端断开：

```text
r.Context() 取消 -> 子 ctx 取消
```

如果子 ctx 自己超时：

```text
子 ctx 取消 -> r.Context() 不取消
```

理解这个传播方向，才能正确设计接口级 timeout 和下游级 timeout。

---

## 十二、复习题

1. 父 context 取消时，子 context 为什么会取消？
2. 子 context 取消时，父 context 为什么不取消？
3. 兄弟 context 之间会互相取消吗？
4. 如果希望任一子任务失败后取消其他任务，应该用什么方式？
