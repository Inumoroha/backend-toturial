# 2. 父子 Context 与取消传播

本节目标：理解 context 的树状结构和取消传播规则。

每次调用：

```go
context.WithCancel(parent)
context.WithTimeout(parent, d)
context.WithDeadline(parent, t)
context.WithValue(parent, key, value)
```

都会基于父 context 创建一个子 context。

因此 context 可以形成一棵树。

---

## 一、父取消会影响子

示例：

```go
parent, parentCancel := context.WithCancel(context.Background())
child, childCancel := context.WithCancel(parent)
defer childCancel()

go func() {
	<-child.Done()
	fmt.Println("child done:", child.Err())
}()

parentCancel()
time.Sleep(time.Second)
```

当父 context 取消时，子 context 也会被取消。

这是最重要的传播规则。

---

## 二、子取消不会影响父

示例：

```go
parent, parentCancel := context.WithCancel(context.Background())
defer parentCancel()

child, childCancel := context.WithCancel(parent)
childCancel()

fmt.Println("child err:", child.Err())
fmt.Println("parent err:", parent.Err())
```

输出大致是：

```text
child err: context canceled
parent err: <nil>
```

子 context 取消，不会反向取消父 context。

这很合理。

如果一个数据库查询的子超时到了，不应该自动让整个 HTTP 请求都取消。是否取消整个请求，应该由上层业务决定。

---

## 三、取消传播是单向的

可以记成：

```text
父 -> 子：会传播
子 -> 父：不会传播
兄弟之间：不会直接传播
```

例如：

```text
request ctx
  -> db ctx
  -> rpc ctx
```

请求取消时，db 和 rpc 都应该取消。

但是 db 超时，不会自动取消 rpc。

如果你希望 db 失败后取消 rpc，需要在业务层显式调用共同父 context 的 cancel，或者使用 `errgroup.WithContext`。

---

## 四、为什么要理解这个规则

真实项目中经常出现这种结构：

```go
func handler(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()

	dbCtx, cancel := context.WithTimeout(ctx, 300*time.Millisecond)
	defer cancel()

	user, err := repo.FindUser(dbCtx, id)
	_ = user
	_ = err
}
```

这里 `dbCtx` 是 `r.Context()` 的子 context。

如果客户端断开连接：

```text
r.Context() 取消
dbCtx 也取消
数据库查询应该停止
```

如果数据库 300ms 超时：

```text
dbCtx 取消
r.Context() 不会自动取消
```

这给了上层业务处理空间。

---

## 五、本节练习

请写代码验证：

1. 父 context 取消后，子 context 也取消。
2. 子 context 取消后，父 context 不取消。
3. 两个兄弟 context 中，一个取消不会影响另一个。

建议打印：

```go
fmt.Println("parent:", parent.Err())
fmt.Println("child:", child.Err())
```

---

## 六、本节达标标准

学完本节后，你应该能够做到：

- 知道 context 是树状结构。
- 解释父取消对子 context 的影响。
- 解释子取消不会影响父 context。
- 知道兄弟 context 之间不会直接传播取消。
- 能在数据库子超时场景中解释传播方向。

---

## 七、完整验证代码

```go
package main

import (
	"context"
	"fmt"
)

func main() {
	parent, parentCancel := context.WithCancel(context.Background())
	child1, child1Cancel := context.WithCancel(parent)
	child2, child2Cancel := context.WithCancel(parent)
	defer child1Cancel()
	defer child2Cancel()

	child1Cancel()
	fmt.Println("after child1 cancel")
	fmt.Println("parent:", parent.Err())
	fmt.Println("child1:", child1.Err())
	fmt.Println("child2:", child2.Err())

	parentCancel()
	fmt.Println("after parent cancel")
	fmt.Println("parent:", parent.Err())
	fmt.Println("child1:", child1.Err())
	fmt.Println("child2:", child2.Err())
}
```

这个程序同时验证：

```text
子取消不影响父。
子取消不影响兄弟。
父取消影响所有子。
```

---

## 八、真实场景：接口和数据库子超时

```go
func handler(w http.ResponseWriter, r *http.Request) {
	requestCtx, cancel := context.WithTimeout(r.Context(), time.Second)
	defer cancel()

	user, err := repo.FindByID(requestCtx, 1001)
	_ = user
	_ = err
}

func (r *Repo) FindByID(ctx context.Context, id int64) (*User, error) {
	queryCtx, cancel := context.WithTimeout(ctx, 300*time.Millisecond)
	defer cancel()

	return r.query(queryCtx, id)
}
```

如果请求 1 秒超时，queryCtx 也会取消。

如果 queryCtx 300ms 超时，请求 ctx 不会自动取消。

这给 service 机会决定是否重试、降级或返回错误。

---

## 九、常见误解

### 1. 子取消会取消父

不会。

### 2. 兄弟 context 会互相取消

不会。

### 3. WithValue 不算父子关系

它也基于父 context 包一层，所以父取消仍然会影响它。

---

## 十、本节练习

创建一个三层 context：

```text
root -> request -> db
```

分别测试：

1. cancel db。
2. cancel request。
3. cancel root。

打印每一层的 `Err()`，写出结论。
