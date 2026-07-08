# 2. Background 与 TODO

本节目标：理解 `context.Background()` 和 `context.TODO()` 的区别，以及什么时候使用它们。

这两个函数都会返回一个根 context：

```go
ctx := context.Background()
ctx := context.TODO()
```

它们都不会自动取消，也没有 deadline，也没有 value。

区别在语义。

---

## 一、Background

`context.Background()` 是最常用的根 context。

适合在下面场景使用：

- `main` 函数。
- 初始化代码。
- 测试代码。
- 顶层任务入口。
- 没有上游 context 的地方。

示例：

```go
func main() {
	ctx := context.Background()
	run(ctx)
}
```

`Background()` 表示：

```text
这里就是一条调用链的起点。
```

---

## 二、TODO

`context.TODO()` 表示：

```text
这里暂时还不知道应该使用哪个 context。
```

它常用于过渡阶段。

比如你正在改造旧代码：

```go
func oldFunction() error {
	ctx := context.TODO()
	return newFunction(ctx)
}
```

这表示当前代码还没完全整理好 context 来源，后续应该回头处理。

---

## 三、不要用 TODO 糊弄所有地方

`TODO()` 不是偷懒工具。

如果你明确知道这是程序入口，就用 `Background()`。

如果你在 HTTP handler 里，就用：

```go
ctx := r.Context()
```

如果你在 service 里，就接收上游传入的 `ctx`。

不推荐：

```go
func (s *UserService) GetUser(id int64) (*User, error) {
	ctx := context.TODO()
	return s.repo.FindByID(ctx, id)
}
```

推荐：

```go
func (s *UserService) GetUser(ctx context.Context, id int64) (*User, error) {
	return s.repo.FindByID(ctx, id)
}
```

---

## 四、不要传 nil context

有些人会写：

```go
doSomething(nil)
```

不要这样做。

如果函数内部调用：

```go
ctx.Done()
```

就会 panic。

如果暂时不知道传什么，至少传：

```go
context.TODO()
```

---

## 五、根 Context 不能取消

`Background()` 和 `TODO()` 本身都不能取消。

下面代码会永远等不到：

```go
ctx := context.Background()
<-ctx.Done()
```

如果你需要取消能力，要基于它派生：

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
```

如果你需要超时能力：

```go
ctx, cancel := context.WithTimeout(context.Background(), time.Second)
defer cancel()
```

---

## 六、练习

写一个函数：

```go
func wait(ctx context.Context) {
	select {
	case <-ctx.Done():
		fmt.Println("done:", ctx.Err())
	case <-time.After(time.Second):
		fmt.Println("still running")
	}
}
```

分别传入：

```go
context.Background()
context.TODO()
```

你会发现它们都不会自动取消。

---

## 七、本节达标标准

学完本节后，你应该能够做到：

- 知道 `Background()` 用于明确的根 context。
- 知道 `TODO()` 用于暂时不知道 context 来源的过渡代码。
- 不传 `nil` context。
- 知道根 context 本身不会取消。
- 知道需要取消或超时时，要用 `WithCancel` / `WithTimeout` 派生。

---

## 八、Background 的典型使用位置

适合：

```go
func main() {
	ctx := context.Background()
	if err := run(ctx); err != nil {
		log.Fatal(err)
	}
}
```

测试中：

```go
func TestService(t *testing.T) {
	ctx := context.Background()
	_ = service.Do(ctx)
}
```

初始化中：

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
_ = db.PingContext(ctx)
```

这些地方没有上游请求，所以 Background 合理。

---

## 九、TODO 的典型使用位置

TODO 常见于重构过渡：

```go
func oldEntry() {
	_ = newFunction(context.TODO())
}
```

它表达：

```text
这里未来应该传入真正的 ctx，但现在还没有整理好。
```

不要把 TODO 长期留在核心请求链路。

---

## 十、团队实践建议

可以定期搜索：

```powershell
rg "context.TODO"
```

检查每个 TODO 是否仍然合理。

如果已经能拿到上游 ctx，就替换掉。

---

## 十一、常见错误

### 1. 所有地方都用 Background

这会让取消和超时无法传播。

### 2. TODO 当作默认值长期存在

TODO 是提醒，不是最终设计。

### 3. 传 nil

nil context 会让下游调用 `ctx.Done()` 时 panic。

---

## 十二、本节练习

找出下面代码的问题：

```go
func handler(w http.ResponseWriter, r *http.Request) {
	ctx := context.Background()
	_ = service.Do(ctx)
}
```

请改成正确版本，并说明为什么。
