# 4. valueCtx 链式取值

本节目标：理解 `context.WithValue` 为什么是链式结构，以及取值时如何查找。

每次调用：

```go
ctx = context.WithValue(ctx, key, value)
```

都会创建一个新的 `valueCtx`，它保存：

```text
父 context。
当前 key。
当前 value。
```

---

## 一、链式结构

假设：

```go
ctx := context.Background()
ctx = context.WithValue(ctx, keyA, "a")
ctx = context.WithValue(ctx, keyB, "b")
ctx = context.WithValue(ctx, keyC, "c")
```

结构可以理解为：

```text
valueCtx(keyC)
  -> valueCtx(keyB)
  -> valueCtx(keyA)
  -> Background
```

---

## 二、Value 如何查找

当你调用：

```go
ctx.Value(keyB)
```

它会从当前这一层开始找：

```text
keyC 是否匹配？不匹配。
keyB 是否匹配？匹配，返回 value。
```

如果一直找到根 context 都没有匹配，就返回 nil。

---

## 三、为什么不要放太多 value

因为 value 是链式查找。

少量请求级元数据没有问题。

但如果你把大量业务数据都塞进去，就会：

- 查找成本增加。
- 语义混乱。
- 函数依赖隐藏。
- 调试困难。

---

## 四、相同 key 会怎样

如果后面又设置了相同 key：

```go
ctx = context.WithValue(ctx, keyA, "new-a")
```

读取时会先找到最外层的新值。

旧值还在父 context 链上，但被遮蔽了。

---

## 五、本节达标标准

学完本节后，你应该能够做到：

- 解释 `valueCtx` 的链式结构。
- 知道取值从当前层向父层查找。
- 知道相同 key 会被外层遮蔽。
- 理解为什么不应该滥用 `WithValue`。

---

## 六、用伪代码理解 valueCtx

可以把 `valueCtx` 想象成：

```go
type valueCtx struct {
	parent Context
	key    any
	value  any
}
```

每次 `WithValue` 都不是修改原来的 context，而是包一层新的 context。

这和不可变链表很像。

```go
ctx1 := context.Background()
ctx2 := context.WithValue(ctx1, keyA, "a")
ctx3 := context.WithValue(ctx2, keyB, "b")
```

`ctx1` 不会被修改。

`ctx2` 保存 keyA。

`ctx3` 保存 keyB，并指向 ctx2。

---

## 七、为什么 WithValue 不修改原 Context

如果 context 可以被原地修改，会带来并发安全和语义问题。

例如多个 goroutine 共用同一个 ctx：

```go
go use(ctx)
go use(ctx)
```

如果其中一个 goroutine 修改了 ctx 里的值，另一个 goroutine 会受到影响。

而 `WithValue` 返回新 context，可以保持原 context 不变。

---

## 八、查找成本如何理解

`Value` 是从外层向内层查找。

如果链很短，比如只有：

```text
request_id
trace_id
auth_user
```

这完全没问题。

但如果你把几十个业务参数都塞进去，链会变长，查找成本和认知成本都会增加。

更重要的是：函数签名会失真。

---

## 九、运行实验：遮蔽旧值

```go
package main

import (
	"context"
	"fmt"
)

type key string

func main() {
	const requestIDKey key = "request_id"

	ctx := context.Background()
	ctx = context.WithValue(ctx, requestIDKey, "old")
	ctx = context.WithValue(ctx, requestIDKey, "new")

	fmt.Println(ctx.Value(requestIDKey))
}
```

输出：

```text
new
```

因为最外层先匹配。

---

## 十、工程建议

使用 `WithValue` 时遵守三条规则：

1. key 使用私有自定义类型。
2. 读写通过封装函数完成。
3. 只放请求级横切元数据。

例如：

```go
func WithRequestID(ctx context.Context, requestID string) context.Context
func RequestIDFromContext(ctx context.Context) (string, bool)
```

这比直接在业务代码中写 `ctx.Value(...)` 更稳定。

---

## 十一、复习题

1. `WithValue` 会修改原 context 吗？
2. `Value` 查找顺序是什么？
3. 相同 key 多次设置时会返回哪一个？
4. 为什么业务参数不应该放进 context value？

---

## 十二、和工程规范的联系

读懂 `valueCtx` 后，你会更明白为什么 `WithValue` 不能滥用。

它不是一个可以随机读写的 map，而是一条不可变链：

```text
valueCtx -> valueCtx -> valueCtx -> emptyCtx
```

每加一个值就多包一层。

这对于 request id、trace id 很合适，因为它们数量少、含义稳定。

但对于业务参数就不合适，因为业务参数通常：

- 数量多。
- 类型多。
- 依赖强。
- 应该通过函数签名表达。

---

## 十三、调试建议

如果你怀疑 context value 没传到下游，检查：

```text
中间件是否调用了 r.WithContext(ctx)。
是否使用了正确的 key 类型。
是否在某一层重新创建了 context.Background。
是否读取时用了另一个包里的 key。
```

最常见的问题是：

```go
ctx := WithRequestID(r.Context(), requestID)
next.ServeHTTP(w, r)
```

这里忘记了：

```go
r.WithContext(ctx)
```
