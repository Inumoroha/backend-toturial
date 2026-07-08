# 2. 项目一：errgroup 并发调用与取消

本节目标：使用 `errgroup.WithContext` 并发调用多个下游，并在失败时取消其他任务。

---

## 一、安装依赖

```powershell
go get golang.org/x/sync/errgroup
```

---

## 二、Service 聚合逻辑

```go
type ProfileService struct{}

func (s *ProfileService) GetProfile(ctx context.Context, userID int64) (*Profile, error) {
	ctx, cancel := context.WithTimeout(ctx, 800*time.Millisecond)
	defer cancel()

	g, ctx := errgroup.WithContext(ctx)

	var user User
	var orders []Order
	var wallet Wallet

	g.Go(func() error {
		var err error
		user, err = fetchUser(ctx, userID)
		return err
	})

	g.Go(func() error {
		var err error
		orders, err = fetchOrders(ctx, userID)
		return err
	})

	g.Go(func() error {
		var err error
		wallet, err = fetchWallet(ctx, userID)
		return err
	})

	if err := g.Wait(); err != nil {
		return nil, err
	}

	return &Profile{
		User:   user,
		Orders: orders,
		Wallet: wallet,
	}, nil
}
```

---

## 三、为什么这里写变量是安全的

上面三个 goroutine 分别写不同变量：

```text
user
orders
wallet
```

`g.Wait()` 返回后再读取它们。

这种写法在简单聚合场景中常见。

不要多个 goroutine 同时写同一个变量或 map。

如果要写共享 map，需要加锁或使用 channel 收集。

---

## 四、失败时取消其他任务

如果 `fetchOrders` 返回错误，`errgroup` 会取消它派生出来的 ctx。

`fetchUser` 和 `fetchWallet` 如果还没完成，并且监听了 `ctx.Done()`，就会尽快退出。

这就是协同取消。

---

## 五、Handler 调用 Service

```go
func (h *ProfileHandler) GetProfile(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()

	profile, err := h.service.GetProfile(ctx, 1001)
	if err != nil {
		http.Error(w, err.Error(), http.StatusBadGateway)
		return
	}

	_ = json.NewEncoder(w).Encode(profile)
}
```

---

## 六、本节达标标准

学完本节后，你应该能够做到：

- 使用 `errgroup.WithContext`。
- 并发执行多个下游调用。
- 在 `g.Wait()` 后组装结果。
- 理解一个任务失败后其他任务如何收到取消信号。

---

## 七、加入总超时后的调用图

```text
GetProfile(ctx)
  -> context.WithTimeout(ctx, 800ms)
  -> errgroup.WithContext(ctx)
     -> fetchUser(ctx)
     -> fetchOrders(ctx)
     -> fetchWallet(ctx)
  -> g.Wait()
  -> 组装 Profile
```

有两个取消来源：

```text
800ms 到期。
某个 goroutine 返回错误。
```

这两个来源都会让 errgroup 的 ctx 取消。

---

## 八、模拟失败实验

可以把 `fetchWallet` 改成：

```go
func fetchWallet(ctx context.Context, userID int64) (Wallet, error) {
	select {
	case <-time.After(100 * time.Millisecond):
		return Wallet{}, fmt.Errorf("wallet service unavailable")
	case <-ctx.Done():
		return Wallet{}, fmt.Errorf("fetch wallet stopped: %w", ctx.Err())
	}
}
```

然后让 `fetchOrders` 慢一点：

```go
time.After(2 * time.Second)
```

你会看到 wallet 很快失败，orders 收到取消，不会继续等满 2 秒。

---

## 九、避免共享数据竞争

示例中三个 goroutine 写不同变量：

```go
var user User
var orders []Order
var wallet Wallet
```

这种写法简单可用。

但不要这样：

```go
result := map[string]any{}

g.Go(func() error {
	result["user"] = user
	return nil
})
g.Go(func() error {
	result["orders"] = orders
	return nil
})
```

多个 goroutine 同时写 map 会 data race。

如果要共享 map，需要：

- 用 mutex。
- 用 channel 收集结果。
- 或避免共享写。

---

## 十、给每个下游设置子超时

总超时 800ms，不代表每个下游都可以占满 800ms。

可以在下游函数内部设置：

```go
func fetchOrders(ctx context.Context, userID int64) ([]Order, error) {
	ctx, cancel := context.WithTimeout(ctx, 600*time.Millisecond)
	defer cancel()

	select {
	case <-time.After(500 * time.Millisecond):
		return []Order{{ID: 1, Amount: 99}}, nil
	case <-ctx.Done():
		return nil, fmt.Errorf("fetch orders stopped: %w", ctx.Err())
	}
}
```

这样 orders 服务最多占 600ms，即使总预算还有剩余，也不会无限拖住。

---

## 十一、常见错误

### 1. g.Wait 前就读结果

必须等 `g.Wait()` 返回后再组装响应。

### 2. goroutine 中直接写 ResponseWriter

不要在多个 goroutine 中同时写 `http.ResponseWriter`。

应该 goroutine 只负责拿数据，handler 最后统一响应。

### 3. 忽略 errgroup 返回错误

```go
_ = g.Wait()
```

这样下游失败会被吞掉。

---

## 十二、本节练习

请完成三个实验：

1. 全部下游成功，接口返回 200。
2. 一个下游返回普通错误，接口返回 502。
3. 一个下游超过总超时，接口返回 504。

每个实验都观察日志里哪个任务先结束，哪个任务被取消。
