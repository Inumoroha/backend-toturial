# 3. 为什么后端服务需要 Context

本节目标：从后端工程场景理解 `context` 的必要性，而不是只把它当作一个 Go API。

在 Go 后端项目里，你会经常看到这种函数签名：

```go
func (s *UserService) GetUser(ctx context.Context, userID int64) (*User, error)
```

为什么第一个参数总是 `ctx context.Context`？

答案是：后端服务中的一次请求，需要把“生命周期信息”从入口传到下游。

---

## 一、一次请求会经过很多层

典型 Web 请求链路：

```text
客户端
  -> 网关
  -> Go HTTP Handler
  -> Middleware
  -> Service
  -> Repository
  -> Database
  -> Redis
  -> 下游 HTTP / RPC 服务
```

如果客户端已经断开连接，或者网关已经超时，那么后面的数据库查询、RPC 调用、后台计算就可能没有意义了。

但是下游代码怎么知道请求已经结束？

不能靠全局变量，也不能靠每层自己猜。

Go 使用 `context.Context` 把这个信号传下去。

---

## 二、没有 Context 会怎样

假设有一个接口：

```text
GET /users/1001
```

内部执行：

```text
查询用户表。
查询订单服务。
查询积分服务。
组装响应。
```

如果订单服务卡住 10 秒，而客户端 1 秒后已经断开连接：

```text
客户端不等了。
Go 服务还在等订单服务。
数据库连接还被占着。
goroutine 还在运行。
```

并发量一高，这些无意义任务就会拖垮服务。

---

## 三、Context 传递什么

`context` 主要传递四类信息：

```text
取消信号。
超时时间。
截止时间。
请求级元数据。
```

取消信号表示：

```text
这个任务不需要继续了。
```

超时时间表示：

```text
最多等多久。
```

截止时间表示：

```text
最晚到哪个时间点必须结束。
```

请求级元数据表示：

```text
request id、trace id、用户身份等和本次请求相关的信息。
```

---

## 四、Context 不负责什么

`context` 不是万能容器。

它不应该用来：

- 保存业务参数。
- 保存数据库连接。
- 保存 service 对象。
- 控制复杂业务流程。
- 替代函数参数。
- 替代配置系统。

错误示例：

```go
ctx = context.WithValue(ctx, "user_id", userID)
userService.GetUser(ctx)
```

更推荐：

```go
userService.GetUser(ctx, userID)
```

如果一个值是函数完成业务必需的参数，就应该明确写在函数签名里。

---

## 五、Context 的传递方向

`context` 通常从上游传到下游：

```text
handler
  -> service
  -> repository
  -> database
```

不要反过来传。

也不要在底层随便创建新的 `context.Background()`，否则会切断上游取消信号。

不推荐：

```go
func (r *UserRepo) FindByID(userID int64) (*User, error) {
	ctx := context.Background()
	return r.query(ctx, userID)
}
```

推荐：

```go
func (r *UserRepo) FindByID(ctx context.Context, userID int64) (*User, error) {
	return r.query(ctx, userID)
}
```

---

## 六、Context 是协作式取消

调用 `cancel()` 并不会强行杀死 goroutine。

它只是发出信号：

```text
请停止。
```

真正停不停，要看业务代码是否监听：

```go
select {
case <-ctx.Done():
	return ctx.Err()
case result := <-resultCh:
	return result
}
```

这叫协作式取消。

所以学习 `context` 时，不能只学习创建 context，还要学习如何在函数内部尊重取消信号。

---

## 七、本节达标标准

学完本节后，你应该能够做到：

- 解释为什么后端请求需要 context。
- 说出 context 传递的四类信息。
- 知道 context 不应该传业务必需参数。
- 知道 context 应该从入口向下游传递。
- 理解 context 的取消是协作式取消。

---

## 八、真实后端例子：用户详情页

假设接口：

```text
GET /users/1001/profile
```

内部需要：

```text
查询用户表。
查询订单服务。
查询优惠券服务。
查询钱包服务。
```

如果客户端 300ms 后断开连接，而服务端还继续调用所有下游：

```text
数据库连接被占用。
HTTP 连接被占用。
goroutine 被占用。
日志里出现大量无意义错误。
```

context 的作用就是让这些下游知道：

```text
这个请求已经没必要继续了。
```

---

## 九、Context 在调用链中的位置

推荐链路：

```go
func (h *Handler) GetProfile(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	_ = h.service.GetProfile(ctx, userID)
}

func (s *Service) GetProfile(ctx context.Context, userID int64) error {
	return s.repo.FindProfile(ctx, userID)
}

func (r *Repo) FindProfile(ctx context.Context, userID int64) error {
	return r.query(ctx, userID)
}
```

context 不负责业务数据本身，但它跟着业务调用链走。

---

## 十、Context 不是线程局部变量

有些语言里会使用 thread local 保存请求信息。

Go 不推荐这种方式。

Go 的风格是显式传参：

```go
DoSomething(ctx, input)
```

这让调用关系更清楚，也更容易测试。

---

## 十一、协作式取消的例子

不响应取消：

```go
func work(ctx context.Context) error {
	time.Sleep(10 * time.Second)
	return nil
}
```

响应取消：

```go
func work(ctx context.Context) error {
	select {
	case <-time.After(10 * time.Second):
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}
```

context 不是魔法，代码必须配合它。

---

## 十二、本节练习

画出你熟悉的一个接口调用链。

例如：

```text
POST /orders
  -> handler
  -> service
  -> repository
  -> payment client
  -> message producer
```

标出：

1. context 从哪里来。
2. 哪些地方应该传 ctx。
3. 哪些地方应该设置 timeout。
4. 哪些 goroutine 需要监听取消。
