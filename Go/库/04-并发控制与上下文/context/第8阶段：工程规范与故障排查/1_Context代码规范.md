# 1. Context 代码规范

本节目标：整理一套 Go 后端项目中可执行的 context 代码规范。

---

## 一、ctx 作为第一个参数

推荐：

```go
func (s *Service) Do(ctx context.Context, input Input) error
```

不推荐：

```go
func (s *Service) Do(input Input, ctx context.Context) error
```

这是 Go 社区约定，也和标准库风格一致。

---

## 二、不要传 nil context

不推荐：

```go
service.Do(nil, input)
```

推荐：

```go
service.Do(context.TODO(), input)
```

更推荐在真实调用链中传上游 ctx。

---

## 三、不要把 context 存进结构体

不推荐：

```go
type Service struct {
	ctx context.Context
}
```

推荐：

```go
func (s *Service) Do(ctx context.Context) error
```

结构体保存稳定依赖，context 作为请求级参数传递。

---

## 四、不要在下游创建 Background

不推荐：

```go
func (r *Repo) Find(id int64) error {
	ctx := context.Background()
	return r.query(ctx, id)
}
```

推荐：

```go
func (r *Repo) Find(ctx context.Context, id int64) error {
	return r.query(ctx, id)
}
```

下游创建 `Background()` 会切断上游取消和超时。

---

## 五、派生 context 后调用 cancel

推荐：

```go
ctx, cancel := context.WithTimeout(ctx, time.Second)
defer cancel()
```

适用于：

- `WithCancel`
- `WithTimeout`
- `WithDeadline`

---

## 六、Value 只存请求级元数据

不要把 context 变成隐式参数包。

业务必需参数写进函数签名。

依赖对象通过结构体注入。

---

## 七、本节达标标准

学完本节后，你应该能够做到：

- 写出团队可执行的 context 规范。
- 在代码 review 中识别 context 使用问题。
- 修正 nil ctx、结构体保存 ctx、下游 Background 等坏味道。

---

## 八、推荐团队规范模板

可以直接把下面这段放进项目开发规范：

```text
1. 所有可能阻塞、耗时、访问外部依赖的方法，必须接收 context.Context。
2. ctx 参数必须作为第一个参数，命名统一为 ctx。
3. 禁止传 nil context，不确定时使用 context.TODO，并在后续重构。
4. HTTP handler 必须优先使用 r.Context。
5. service、repository 不允许自行创建 context.Background。
6. 派生 cancel、timeout、deadline context 后必须调用 cancel。
7. context 不允许作为普通业务结构体字段保存。
8. context.Value 只允许传递请求级元数据。
9. 启动 goroutine 时必须明确退出条件，必要时监听 ctx.Done。
10. 对数据库、HTTP、RPC 等外部调用必须设置合理超时。
```

规范不需要一开始很复杂，但必须能执行。

---

## 九、代码 Review 检查点

看到一个函数时，先问：

```text
它会不会访问数据库、Redis、HTTP、RPC？
它会不会启动 goroutine？
它会不会等待 channel、锁、外部结果？
```

如果答案是会，就应该考虑接收 context。

看到一个调用链时，检查：

```text
handler 有没有从 r.Context 开始？
service 有没有继续传？
repo 有没有继续传？
外部调用有没有使用带 context 的方法？
```

---

## 十、坏味道示例：参数顺序混乱

不推荐：

```go
func GetUser(id int64, ctx context.Context) (*User, error)
```

推荐：

```go
func GetUser(ctx context.Context, id int64) (*User, error)
```

原因不是编译器要求，而是统一风格降低理解成本。

你看到第一个参数是 ctx，就知道这个函数遵守 Go 后端常见约定。

---

## 十一、坏味道示例：隐藏超时

不推荐：

```go
func (c *Client) FetchUser(id int64) (*User, error) {
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()
	return c.fetch(ctx, id)
}
```

这个函数把 timeout 写死，并切断上游 context。

更推荐：

```go
func (c *Client) FetchUser(ctx context.Context, id int64) (*User, error) {
	ctx, cancel := context.WithTimeout(ctx, time.Second)
	defer cancel()
	return c.fetch(ctx, id)
}
```

上游仍然能控制整个请求生命周期。

---

## 十二、坏味道示例：goroutine 不受控

不推荐：

```go
func Handle() {
	go func() {
		for {
			doWork()
		}
	}()
}
```

推荐：

```go
func Handle(ctx context.Context) {
	go func() {
		for {
			select {
			case <-ctx.Done():
				return
			default:
				doWork()
			}
		}
	}()
}
```

如果 goroutine 的生命周期不清楚，迟早会变成排查难题。

---

## 十三、本节练习

请找一个你以前写过的 Go demo，检查：

1. 哪些函数应该接收 ctx。
2. 有没有下游使用 `context.Background()`。
3. 有没有 `WithTimeout` 后忘记 `cancel()`。
4. 有没有把业务参数放进 context。
5. 有没有启动后无法退出的 goroutine。

把这些问题改掉，比单纯背规范更有效。
