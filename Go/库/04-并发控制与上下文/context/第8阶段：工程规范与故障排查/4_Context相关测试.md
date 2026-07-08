# 4. Context 相关测试

本节目标：学习如何测试支持 context 的函数。

支持 context 的函数，至少要测试：

- 正常完成。
- 超时返回。
- 主动取消返回。
- 错误是否能被 `errors.Is` 判断。

---

## 一、被测函数

```go
func Work(ctx context.Context, cost time.Duration) error {
	select {
	case <-time.After(cost):
		return nil
	case <-ctx.Done():
		return fmt.Errorf("work stopped: %w", ctx.Err())
	}
}
```

---

## 二、测试正常完成

```go
func TestWorkSuccess(t *testing.T) {
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()

	err := Work(ctx, 10*time.Millisecond)
	if err != nil {
		t.Fatalf("expected nil, got %v", err)
	}
}
```

---

## 三、测试超时

```go
func TestWorkTimeout(t *testing.T) {
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Millisecond)
	defer cancel()

	err := Work(ctx, time.Second)
	if !errors.Is(err, context.DeadlineExceeded) {
		t.Fatalf("expected deadline exceeded, got %v", err)
	}
}
```

---

## 四、测试主动取消

```go
func TestWorkCanceled(t *testing.T) {
	ctx, cancel := context.WithCancel(context.Background())
	cancel()

	err := Work(ctx, time.Second)
	if !errors.Is(err, context.Canceled) {
		t.Fatalf("expected canceled, got %v", err)
	}
}
```

---

## 五、测试要快

测试中不要真的等几秒。

使用很短的时间：

```go
10 * time.Millisecond
```

但也不要短到在慢机器上不稳定。

一般可以结合项目情况使用 10ms、20ms、50ms。

---

## 六、本节达标标准

学完本节后，你应该能够做到：

- 为 context 函数写成功、超时、取消测试。
- 使用 `errors.Is` 判断包装错误。
- 避免测试耗时过长或不稳定。

---

## 七、测试 HTTP Handler 超时

可以用 `httptest` 测试 handler。

示例：

```go
func TestHandlerTimeout(t *testing.T) {
	h := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		ctx, cancel := context.WithTimeout(r.Context(), 10*time.Millisecond)
		defer cancel()

		select {
		case <-time.After(time.Second):
			w.WriteHeader(http.StatusOK)
		case <-ctx.Done():
			http.Error(w, "timeout", http.StatusGatewayTimeout)
		}
	})

	req := httptest.NewRequest(http.MethodGet, "/slow", nil)
	rec := httptest.NewRecorder()

	h.ServeHTTP(rec, req)

	if rec.Code != http.StatusGatewayTimeout {
		t.Fatalf("expected 504, got %d", rec.Code)
	}
}
```

这个测试不需要真的启动端口。

---

## 八、测试主动取消

可以创建一个已取消 context：

```go
ctx, cancel := context.WithCancel(context.Background())
cancel()

err := Work(ctx, time.Second)
if !errors.Is(err, context.Canceled) {
	t.Fatalf("expected canceled, got %v", err)
}
```

这种测试很快，也很稳定。

---

## 九、避免脆弱测试

不推荐：

```go
ctx, cancel := context.WithTimeout(context.Background(), time.Nanosecond)
```

太短的时间容易受调度影响。

也不推荐测试真的等待很多秒。

推荐使用：

```text
10ms 到 100ms
```

具体取值要结合 CI 环境性能。

---

## 十、测试 errgroup 取消

可以设计一个任务快速失败，另一个任务等待 ctx：

```go
func TestErrgroupCancel(t *testing.T) {
	g, ctx := errgroup.WithContext(context.Background())

	g.Go(func() error {
		return errors.New("failed")
	})

	stopped := make(chan struct{})
	g.Go(func() error {
		defer close(stopped)
		<-ctx.Done()
		return ctx.Err()
	})

	_ = g.Wait()

	select {
	case <-stopped:
	case <-time.After(time.Second):
		t.Fatal("worker did not stop")
	}
}
```

这个测试验证的是：

```text
一个任务失败后，其他任务能收到取消信号。
```

---

## 十一、测试 request id 传递

```go
func TestRequestIDContext(t *testing.T) {
	ctx := context.Background()
	ctx = WithRequestID(ctx, "req-1")

	got, ok := RequestIDFromContext(ctx)
	if !ok {
		t.Fatal("request id not found")
	}
	if got != "req-1" {
		t.Fatalf("expected req-1, got %s", got)
	}
}
```

context value 的封装函数也应该测试。

---

## 十二、本节练习

请为下面函数写三个测试：

```go
func Fetch(ctx context.Context, cost time.Duration) error
```

测试要求：

1. cost 很短时成功。
2. timeout 比 cost 短时返回 `DeadlineExceeded`。
3. ctx 已取消时返回 `Canceled`。

如果错误有包装，必须使用 `errors.Is` 判断。
