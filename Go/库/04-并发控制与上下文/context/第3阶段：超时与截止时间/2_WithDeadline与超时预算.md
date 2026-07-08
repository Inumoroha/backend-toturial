# 2. WithDeadline 与超时预算

本节目标：理解 `WithDeadline`，并学习后端接口中的超时预算思维。

`WithTimeout` 表示持续时间。

`WithDeadline` 表示具体截止时间。

```go
deadline := time.Now().Add(2 * time.Second)
ctx, cancel := context.WithDeadline(parent, deadline)
defer cancel()
```

---

## 一、WithDeadline 基础示例

```go
deadline := time.Now().Add(time.Second)
ctx, cancel := context.WithDeadline(context.Background(), deadline)
defer cancel()

select {
case <-time.After(2 * time.Second):
	fmt.Println("done")
case <-ctx.Done():
	fmt.Println("stopped:", ctx.Err())
}
```

因为 deadline 是 1 秒后，所以会输出：

```text
stopped: context deadline exceeded
```

---

## 二、WithTimeout 和 WithDeadline 的关系

可以把：

```go
context.WithTimeout(parent, 2*time.Second)
```

理解成：

```go
context.WithDeadline(parent, time.Now().Add(2*time.Second))
```

日常业务代码里，`WithTimeout` 更直观。

当你已经拿到了一个明确截止时间，或者要把 deadline 传给下游时，`WithDeadline` 更合适。

---

## 三、什么是超时预算

假设接口总超时是 1 秒。

内部要做三件事：

```text
查询用户：最多 200ms
查询订单：最多 300ms
查询钱包：最多 200ms
组装响应：预留 100ms
网络抖动：预留 200ms
```

这就是超时预算。

不是每个下游都随便设置 1 秒。

如果每个下游都设置 1 秒，三个下游串行调用时，接口可能超过 3 秒。

---

## 四、父 context 已经有 deadline 时怎么办

如果父 context 还有 500ms 就超时，你创建一个 2 秒的子 timeout：

```go
child, cancel := context.WithTimeout(parent, 2*time.Second)
defer cancel()
```

子 context 实际不会活 2 秒。

父 context 先超时，子 context 也会取消。

也就是说，子 context 的有效截止时间不会晚于父 context。

---

## 五、读取剩余时间

可以使用：

```go
deadline, ok := ctx.Deadline()
if ok {
	remaining := time.Until(deadline)
	fmt.Println("remaining:", remaining)
}
```

这在设计下游调用预算时很有用。

例如，如果请求只剩 50ms，就没必要再发起一个预计 300ms 的慢调用。

---

## 六、本节练习

请完成：

1. 创建一个 3 秒后截止的 parent context。
2. 基于 parent 创建一个 10 秒 timeout 的 child context。
3. 打印 parent 和 child 的 deadline。
4. 观察 child 的 deadline 是否会超过 parent。
5. 写出你的结论。

---

## 七、本节达标标准

学完本节后，你应该能够做到：

- 使用 `WithDeadline`。
- 解释 `WithTimeout` 和 `WithDeadline` 的区别。
- 理解子 context deadline 不会超过父 context。
- 初步设计接口总超时和下游调用超时。

---

## 八、完整示例：打印父子 Deadline

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func printDeadline(name string, ctx context.Context) {
	deadline, ok := ctx.Deadline()
	if !ok {
		fmt.Println(name, "no deadline")
		return
	}
	fmt.Println(name, "remaining:", time.Until(deadline).Round(time.Millisecond))
}

func main() {
	parent, parentCancel := context.WithTimeout(context.Background(), time.Second)
	defer parentCancel()

	child, childCancel := context.WithTimeout(parent, 5*time.Second)
	defer childCancel()

	printDeadline("parent", parent)
	printDeadline("child", child)
}
```

运行后你会发现 child 的剩余时间不会超过 parent。

---

## 九、预算设计示例

假设你有接口：

```text
GET /checkout
```

内部步骤：

```text
查用户：100ms
查购物车：200ms
锁库存：300ms
创建订单：300ms
调用支付预下单：500ms
```

如果接口总预算 1500ms，可以设计：

| 步骤 | 子 timeout |
| --- | --- |
| 查用户 | 150ms |
| 查购物车 | 250ms |
| 锁库存 | 400ms |
| 创建订单 | 400ms |
| 支付预下单 | 600ms |

注意这些步骤如果串行，不能只看单个 timeout，还要看总耗时。

---

## 十、什么时候用 Deadline

`Deadline` 适合：

- 上游已经传来明确截止时间。
- 协议里有 deadline 概念。
- 希望多个子任务共享同一个结束时间点。

例如：

```go
deadline := time.Now().Add(time.Second)
ctx, cancel := context.WithDeadline(parent, deadline)
```

如果只是“从现在起最多等多久”，优先用 `WithTimeout`。

---

## 十一、常见错误

### 1. 不看父 context 剩余时间

如果父 ctx 只剩 50ms，就不要再发起一个明显需要 500ms 的下游调用。

### 2. 每层都加很长 timeout

下游 timeout 不能突破上游预算。

### 3. deadline 时间使用本地时区做业务判断

deadline 是时间点，通常只用于控制生命周期，不应该混入复杂业务时间判断。

---

## 十二、本节练习

请写一个函数：

```go
func enoughTime(ctx context.Context, need time.Duration) bool
```

要求：

1. 如果 ctx 没有 deadline，返回 true。
2. 如果剩余时间大于 need，返回 true。
3. 否则返回 false。

然后用它判断是否值得发起一个预计 300ms 的下游调用。
