# Go Context 面试题与参考答案

这份文档用于复习 Go 后端面试中关于 `context` 的常见问题。

适合使用方式：

```text
先自己回答。
再看参考答案。
最后尝试用自己的项目经历重新组织语言。
```

面试中不要只背 API。更重要的是能结合后端场景说明：

- 为什么需要 context。
- context 如何取消和超时。
- 如何在 HTTP、数据库、RPC、goroutine 中传递。
- 如何避免 goroutine 泄漏。
- 如何设计 timeout budget。
- 如何排查 context 相关线上问题。

---

## 一、基础概念

### 1. Go 的 context 主要解决什么问题？

`context` 主要解决请求链路中的生命周期控制和请求级元数据传递问题。

它通常用于：

- 传递取消信号。
- 传递超时或截止时间。
- 传递请求级元数据，例如 request id、trace id、认证用户。
- 控制 goroutine、数据库查询、HTTP/RPC 调用等下游任务的生命周期。

典型场景是 HTTP 请求：

```text
client -> handler -> service -> repository -> database / RPC
```

如果客户端断开连接，或者请求超时，后面的数据库查询、RPC 调用、goroutine 都应该尽快停止。`context` 就是把这个取消信号从入口一路传到下游的标准机制。

---

### 2. `context.Context` 是什么类型？

`context.Context` 是一个接口。

定义大致如下：

```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key any) any
}
```

它不是具体结构体。标准库内部有多种实现，例如：

- `emptyCtx`：根 context。
- `cancelCtx`：支持取消。
- `timerCtx`：支持超时和 deadline。
- `valueCtx`：支持 key-value。

业务函数通常只接收接口：

```go
func Do(ctx context.Context) error
```

这样不需要关心具体实现。

---

### 3. `Context` 接口四个方法分别做什么？

`Deadline()`：

返回 context 的截止时间，以及是否存在截止时间。

```go
deadline, ok := ctx.Deadline()
```

`Done()`：

返回一个只读 channel。当 context 被取消或超时时，这个 channel 会关闭。

```go
<-ctx.Done()
```

`Err()`：

返回 context 结束的原因。常见值：

```go
context.Canceled
context.DeadlineExceeded
```

`Value(key)`：

读取 context 中的请求级元数据。

---

### 4. `context.Background()` 和 `context.TODO()` 有什么区别？

它们行为上都返回一个根 context：

- 不会自动取消。
- 没有 deadline。
- 没有 value。

区别主要是语义：

`context.Background()` 表示明确的根 context，常用于：

- `main` 函数。
- 初始化。
- 测试。
- 顶层任务入口。

`context.TODO()` 表示暂时还不知道该使用哪个 context，常用于重构过渡。

例如旧代码还没接入 context，可以临时写：

```go
ctx := context.TODO()
```

但长期看，应该尽量替换成真正的上游 context。

---

### 5. 为什么不能传 `nil` context？

因为下游可能调用：

```go
ctx.Done()
ctx.Err()
ctx.Value(...)
```

如果 `ctx` 是 nil，会 panic。

如果暂时不知道传什么，至少应该传：

```go
context.TODO()
```

如果是程序入口，应该用：

```go
context.Background()
```

---

### 6. 为什么 `ctx context.Context` 通常作为函数第一个参数？

这是 Go 社区约定。

常见写法：

```go
func (s *UserService) GetUser(ctx context.Context, userID int64) (*User, error)
```

好处：

- 一眼看出函数支持取消和超时。
- 调用链风格统一。
- 和标准库、数据库库、RPC 库风格一致。
- 方便代码 review。

不推荐：

```go
func GetUser(userID int64, ctx context.Context)
```

---

### 7. context 是线程安全的吗？

`context` 通常可以被多个 goroutine 同时读取和传递。

例如多个 goroutine 同时监听：

```go
<-ctx.Done()
```

是常见用法。

但注意：

- context 本身可以并发使用。
- 放进 context 的 value 如果是可变对象，仍然需要自己保证并发安全。

例如把 map 放进 context，然后多个 goroutine 修改这个 map，就可能 data race。

---

### 8. context 是不可变的吗？

从使用者角度看，可以认为 context 是不可变链式结构。

调用：

```go
ctx2 := context.WithValue(ctx1, key, value)
```

不会修改 `ctx1`，而是基于 `ctx1` 包一层新的 context。

`WithCancel`、`WithTimeout`、`WithDeadline` 也都是基于父 context 创建子 context。

---

## 二、取消控制

### 9. `context.WithCancel` 是做什么的？

`WithCancel` 基于父 context 创建一个可取消的子 context。

```go
ctx, cancel := context.WithCancel(parent)
defer cancel()
```

调用 `cancel()` 后：

- `ctx.Done()` 会被关闭。
- `ctx.Err()` 返回 `context.Canceled`。
- 派生自这个 ctx 的子 context 也会被取消。

常见场景：

- 用户主动取消任务。
- 一个 goroutine 失败后取消其他 goroutine。
- 服务关闭时通知 worker 退出。

---

### 10. 调用 `cancel()` 会强制杀死 goroutine 吗？

不会。

context 的取消是协作式取消。

`cancel()` 只是关闭 `ctx.Done()`，发出“请停止”的信号。

goroutine 必须主动监听：

```go
select {
case <-ctx.Done():
	return ctx.Err()
default:
	doWork()
}
```

如果 goroutine 完全不检查 `ctx.Done()`，它不会因为 cancel 自动退出。

---

### 11. 为什么调用 `cancel()` 后要返回 `ctx.Err()`？

因为调用方需要知道任务停止的原因。

推荐：

```go
case <-ctx.Done():
	return ctx.Err()
```

这样调用方可以判断：

```go
errors.Is(err, context.Canceled)
errors.Is(err, context.DeadlineExceeded)
```

不推荐：

```go
case <-ctx.Done():
	return errors.New("failed")
```

这样会丢失根因。

---

### 12. 多次调用 `cancel()` 会怎样？

多次调用 `cancel()` 是安全的。

第一次调用会真正取消 context，后续调用基本没有额外效果。

这很重要，因为代码中常见：

```go
ctx, cancel := context.WithCancel(parent)
defer cancel()

if err != nil {
	cancel()
	return err
}
```

提前 cancel 和 defer cancel 可能都会执行，标准库保证这种写法安全。

---

### 13. 父 context 取消后，子 context 会怎样？

父 context 取消后，子 context 会被取消。

例如：

```go
parent, parentCancel := context.WithCancel(context.Background())
child, childCancel := context.WithCancel(parent)
defer childCancel()

parentCancel()
fmt.Println(child.Err()) // context canceled
```

取消传播方向是从父到子。

---

### 14. 子 context 取消后，父 context 会怎样？

子 context 取消不会影响父 context。

例如数据库查询子 context 超时，不会自动取消整个 HTTP 请求 context。

这是合理的，因为上层业务可能希望：

- 重试。
- 降级。
- 换一个数据源。
- 返回部分结果。

如果想让一个子任务失败后取消其他兄弟任务，通常应该使用共同父 context 的 cancel，或者使用 `errgroup.WithContext`。

---

### 15. 兄弟 context 之间会互相取消吗？

不会。

同一个父 context 派生出的两个子 context，彼此之间不会直接传播取消。

```text
parent
  -> childA
  -> childB
```

取消 `childA` 不会取消 `childB`。

如果需要“任一失败取消其他”，应该在上层创建共同的可取消 context，或者使用 `errgroup.WithContext`。

---

### 16. 如何用 context 避免 goroutine 泄漏？

关键是 goroutine 要有退出条件，并监听 `ctx.Done()`。

示例：

```go
func worker(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			return
		case job := <-jobs:
			handle(job)
		}
	}
}
```

如果 goroutine 中有发送 channel 的操作，也要考虑取消：

```go
select {
case resultCh <- result:
case <-ctx.Done():
	return
}
```

否则调用方超时不接收了，发送方可能永久阻塞。

---

### 17. goroutine 中使用 `select { default: }` 有什么风险？

如果这样写：

```go
for {
	select {
	case <-ctx.Done():
		return
	default:
	}
}
```

会导致 CPU 空转。

`default` 分支会立即执行，循环不会阻塞。

如果使用 `default`，里面应该有真实工作或等待：

```go
default:
	doWork()
	time.Sleep(100 * time.Millisecond)
```

很多场景可以不用 `default`，直接阻塞等待 job 或 ctx。

---

## 三、超时与 Deadline

### 18. `context.WithTimeout` 是做什么的？

`WithTimeout` 创建一个会在指定时间后自动取消的 context。

```go
ctx, cancel := context.WithTimeout(parent, time.Second)
defer cancel()
```

超过 1 秒后：

- `ctx.Done()` 关闭。
- `ctx.Err()` 返回 `context.DeadlineExceeded`。

常用于：

- 数据库查询超时。
- HTTP 下游请求超时。
- RPC 调用超时。
- 接口总耗时控制。

---

### 19. `context.WithDeadline` 是做什么的？

`WithDeadline` 创建一个在指定时间点自动取消的 context。

```go
deadline := time.Now().Add(time.Second)
ctx, cancel := context.WithDeadline(parent, deadline)
defer cancel()
```

`WithTimeout(parent, d)` 可以理解为：

```go
WithDeadline(parent, time.Now().Add(d))
```

日常业务中 `WithTimeout` 更常用；当你已经有明确截止时间时，用 `WithDeadline` 更合适。

---

### 20. `WithTimeout` 和 `WithDeadline` 有什么区别？

`WithTimeout` 表达持续时间：

```text
从现在开始最多等 500ms。
```

`WithDeadline` 表达具体时间点：

```text
最晚到 10:00:01 结束。
```

底层上，`WithTimeout` 本质是基于当前时间计算 deadline。

---

### 21. 为什么 `WithTimeout` 后还要 `defer cancel()`？

即使超时后 context 会自动取消，也应该调用 `cancel()`。

原因：

- 提前释放 timer 等内部资源。
- 从父 context 中解除子 context 关系。
- 形成统一习惯，避免分支提前返回时泄漏资源。

标准写法：

```go
ctx, cancel := context.WithTimeout(parent, time.Second)
defer cancel()
```

---

### 22. `context.Canceled` 和 `context.DeadlineExceeded` 有什么区别？

`context.Canceled` 表示主动取消。

例如：

```go
cancel()
```

`context.DeadlineExceeded` 表示超时或 deadline 到达。

例如：

```go
context.WithTimeout(parent, time.Second)
```

判断时推荐：

```go
errors.Is(err, context.Canceled)
errors.Is(err, context.DeadlineExceeded)
```

---

### 23. 子 context 的 timeout 能超过父 context 吗？

不能。

如果父 context 1 秒后超时，子 context 设置 10 秒 timeout，子 context 也会在父 context 1 秒超时时被取消。

父 context 的生命周期是子 context 的上限。

这符合请求链路：

```text
上游只给你 1 秒，你不能让下游继续跑 10 秒。
```

---

### 24. 什么是 timeout budget？

timeout budget 是指为一次请求的各个环节分配时间预算。

例如接口总预算 1 秒：

```text
查询用户：200ms
查询订单：400ms
查询钱包：200ms
响应组装和预留：200ms
```

不能所有下游都随便写 1 秒。

如果是串行调用，耗时会累加。

如果是并发调用，也需要为单个下游设置子 timeout，避免某个依赖拖垮整体。

---

### 25. timeout 是性能优化吗？

不是。

timeout 是稳定性保护措施。

它不能让慢 SQL 变快，也不能修复下游服务性能问题。

它的作用是：

- 避免无限等待。
- 限制资源占用时间。
- 在异常情况下快速失败。

如果 timeout 频繁发生，应该排查根因，例如慢查询、锁等待、连接池耗尽、下游故障，而不是简单把 timeout 调大。

---

### 26. 如何判断一个 context 还有多少剩余时间？

可以使用 `Deadline()`：

```go
deadline, ok := ctx.Deadline()
if ok {
	remaining := time.Until(deadline)
	fmt.Println(remaining)
}
```

如果 `ok == false`，说明没有 deadline。

这个能力可用于判断是否值得发起新的下游调用。

---

## 四、请求链路传递

### 27. HTTP handler 中应该从哪里获取 context？

从请求对象获取：

```go
ctx := r.Context()
```

`r.Context()` 会和 HTTP 请求生命周期绑定。

当客户端断开连接、请求结束或服务端取消时，这个 context 会被取消。

---

### 28. 为什么不要在 HTTP handler 中用 `context.Background()` 替代 `r.Context()`？

因为 `context.Background()` 是根 context，不会随着请求取消。

如果 handler 中写：

```go
ctx := context.Background()
```

后续数据库查询、HTTP 下游调用就感知不到客户端断开或请求取消。

正确写法：

```go
ctx := r.Context()
```

如果要设置业务超时，应基于 `r.Context()` 派生：

```go
ctx, cancel := context.WithTimeout(r.Context(), time.Second)
defer cancel()
```

---

### 29. context 在 handler、service、repository 中应该如何传递？

推荐：

```go
func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	user, err := h.service.GetUser(ctx, userID)
}

func (s *UserService) GetUser(ctx context.Context, userID int64) (*User, error) {
	return s.repo.FindByID(ctx, userID)
}

func (r *UserRepo) FindByID(ctx context.Context, userID int64) (*User, error) {
	return r.query(ctx, userID)
}
```

原则：

```text
入口拿 ctx。
向下游传 ctx。
外部调用使用 ctx。
```

---

### 30. 为什么不要把 `*http.Request` 传到 service 层？

因为 service 层不应该依赖 HTTP 细节。

不推荐：

```go
func (s *UserService) GetUser(r *http.Request, id int64)
```

推荐：

```go
func (s *UserService) GetUser(ctx context.Context, id int64)
```

这样 service 可以被 HTTP、gRPC、CLI、消息队列消费者复用。

---

### 31. 为什么不要在 repository 中创建 `context.Background()`？

repository 是请求链路的下游，应该接收上游传来的 ctx。

不推荐：

```go
func (r *Repo) Find(id int64) (*User, error) {
	ctx := context.Background()
	return r.query(ctx, id)
}
```

这样会切断请求取消和超时。

推荐：

```go
func (r *Repo) Find(ctx context.Context, id int64) (*User, error) {
	return r.query(ctx, id)
}
```

---

### 32. 为什么不要把 context 存进结构体？

因为 context 通常是请求级生命周期，而 service、repo 等结构体通常是应用级生命周期。

不推荐：

```go
type UserService struct {
	ctx context.Context
}
```

问题：

- 生命周期混乱。
- 可能复用已经取消的旧请求 ctx。
- 函数依赖不清晰。
- 并发请求之间可能互相影响。

推荐：

```go
func (s *UserService) GetUser(ctx context.Context, id int64) (*User, error)
```

结构体保存稳定依赖，例如 repo、db、client、logger；context 作为方法参数传递。

---

### 33. 有没有可以把 context 放进结构体的例外？

少数情况下可以，例如某个后台组件或 worker group 持有一个用于组件生命周期的 root context。

但普通业务层的 handler、service、repository 不应该保存 context。

面试中可以回答：

```text
原则上不把 context 存进业务结构体。极少数生命周期管理组件可以持有 root context，但要非常明确它的生命周期和用途。
```

---

## 五、Context Value

### 34. `context.WithValue` 是做什么的？

`WithValue` 基于父 context 创建一个带 key-value 的子 context。

```go
ctx = context.WithValue(ctx, key, value)
```

读取：

```go
value := ctx.Value(key)
```

它适合传请求级元数据，不适合传业务参数或依赖对象。

---

### 35. 什么数据适合放入 context value？

适合：

- request id。
- trace id。
- span context。
- 已认证用户。
- tenant id。
- 日志字段中的请求级信息。

这些数据通常是横切关注点，不是业务函数的核心参数。

---

### 36. 什么数据不适合放入 context value？

不适合：

- userID、orderID、page、pageSize 等业务参数。
- 数据库连接。
- repository、service、client 等依赖对象。
- 配置对象。
- 大对象。
- 可变 map。

业务必需参数应该写进函数签名。

依赖对象应该通过结构体字段注入。

---

### 37. 为什么不建议用 context 传业务参数？

因为会造成隐式依赖。

不推荐：

```go
ctx = context.WithValue(ctx, "user_id", userID)
service.GetUser(ctx)
```

调用方看不出 `GetUser` 必须依赖 userID。

推荐：

```go
service.GetUser(ctx, userID)
```

函数签名就是契约，业务必需参数应该显式表达。

---

### 38. context value 的 key 应该怎么定义？

不要直接使用 string。

推荐使用包内私有类型：

```go
type contextKey string

const requestIDKey contextKey = "request_id"
```

或者：

```go
type key struct {
	name string
}

var requestIDKey = key{name: "request_id"}
```

原因是避免和其他包的 key 冲突。

---

### 39. context value 应该如何封装？

推荐封装读写函数：

```go
func WithRequestID(ctx context.Context, requestID string) context.Context {
	return context.WithValue(ctx, requestIDKey, requestID)
}

func RequestIDFromContext(ctx context.Context) (string, bool) {
	v, ok := ctx.Value(requestIDKey).(string)
	return v, ok
}
```

业务代码不要到处写：

```go
ctx.Value(key).(string)
```

这样容易 panic，也不利于维护。

---

### 40. 认证用户适合放 context 吗？

适合，但要区分“当前认证用户”和“业务目标用户”。

当前认证用户表示：

```text
谁正在发起请求。
```

适合由认证中间件放入 context。

业务目标用户表示：

```text
要查询或操作哪个用户。
```

应该作为函数参数。

例如管理员查询用户 1001：

```text
当前认证用户：管理员 1
业务参数：用户 1001
```

---

### 41. `WithValue` 的内部查找方式是什么？

`WithValue` 是链式结构。

每次调用都会包一层 `valueCtx`。

```text
valueCtx(keyC)
  -> valueCtx(keyB)
  -> valueCtx(keyA)
  -> Background
```

调用 `Value(key)` 时，从当前层向父层逐层查找。

如果相同 key 设置多次，最外层的值会优先返回。

---

## 六、HTTP、数据库与常用组件

### 42. 如何在 HTTP 客户端请求中使用 context？

使用：

```go
req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
if err != nil {
	return err
}

resp, err := http.DefaultClient.Do(req)
if err != nil {
	return err
}
defer resp.Body.Close()
```

如果 ctx 被取消或超时，请求也会取消。

在 handler 中调用下游 HTTP 服务时，应该基于 `r.Context()` 派生子 timeout：

```go
ctx, cancel := context.WithTimeout(r.Context(), 500*time.Millisecond)
defer cancel()
```

---

### 43. `http.Client.Timeout` 和 request context timeout 有什么区别？

`http.Client.Timeout` 是 client 级别的总超时，通常作为兜底。

request context timeout 是每次请求自己的业务预算。

推荐同时使用：

```go
client := &http.Client{
	Timeout: 5 * time.Second,
}

ctx, cancel := context.WithTimeout(parent, 800*time.Millisecond)
defer cancel()
req, _ := http.NewRequestWithContext(ctx, method, url, body)
```

这样既有全局保护，也有请求级控制。

---

### 44. 数据库操作中如何使用 context？

`database/sql` 中使用带 context 的方法：

```go
db.QueryContext(ctx, sql, args...)
db.QueryRowContext(ctx, sql, args...)
db.ExecContext(ctx, sql, args...)
db.BeginTx(ctx, opts)
```

示例：

```go
ctx, cancel := context.WithTimeout(ctx, 300*time.Millisecond)
defer cancel()

row := db.QueryRowContext(ctx, `
	SELECT id, name
	FROM users
	WHERE id = ?
`, id)
```

这样数据库查询可以受请求取消和超时控制。

---

### 45. 查询多行时 context 之外还要注意什么？

要关闭 rows，并检查 `rows.Err()`。

```go
rows, err := db.QueryContext(ctx, sql)
if err != nil {
	return err
}
defer rows.Close()

for rows.Next() {
	// scan
}

if err := rows.Err(); err != nil {
	return err
}
```

如果忘记 `rows.Close()`，连接可能不能及时归还连接池。

---

### 46. 事务中如何使用 context？

开启事务时使用：

```go
tx, err := db.BeginTx(ctx, nil)
if err != nil {
	return err
}
defer tx.Rollback()
```

事务内 SQL 继续传同一个 ctx：

```go
_, err = tx.ExecContext(ctx, sql, args...)
```

成功后：

```go
return tx.Commit()
```

如果 ctx 取消或超时，事务应尽快回滚。

---

### 47. context 超时能保证数据库 SQL 一定被取消吗？

不一定完全保证，取决于数据库驱动是否支持取消，以及数据库本身的行为。

成熟驱动通常支持得比较好。

但你要理解：

```text
context 是 Go 侧取消信号。
驱动需要把它转换成数据库协议层面的取消。
```

即使底层取消不够及时，context 仍然可以让调用方停止等待。

---

### 48. `signal.NotifyContext` 是做什么的？

`signal.NotifyContext` 可以把系统信号转换成 context 取消。

常用于服务优雅关闭：

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

<-ctx.Done()
```

收到 Ctrl+C 或 SIGTERM 时，ctx 会取消。

---

### 49. 为什么 HTTP Server Shutdown 要使用新的 context？

收到退出信号后，signal ctx 已经取消。

如果直接传给：

```go
server.Shutdown(ctx)
```

可能会立刻返回。

正确做法是创建新的 shutdown context：

```go
shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

server.Shutdown(shutdownCtx)
```

它表示最多等待 5 秒完成优雅关闭。

---

### 50. 如何用 context 实现服务优雅关闭？

基本流程：

```text
1. 使用 signal.NotifyContext 监听退出信号。
2. HTTP server 停止接收新请求。
3. 正在处理的请求继续完成。
4. worker 收到 ctx.Done 后退出。
5. 使用 shutdown timeout 限制等待时间。
6. 关闭数据库、Redis、消息队列等资源。
```

代码核心：

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

<-ctx.Done()

shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

server.Shutdown(shutdownCtx)
```

---

## 七、errgroup 与并发

### 51. `errgroup.WithContext` 是做什么的？

`errgroup.WithContext` 用于多个 goroutine 协同执行。

它会返回：

```go
g, ctx := errgroup.WithContext(parent)
```

特点：

- 可以启动多个返回 error 的 goroutine。
- `g.Wait()` 等待所有 goroutine 结束。
- 任一 goroutine 返回非 nil error 时，会取消派生 ctx。
- 其他监听 ctx 的 goroutine 可以尽快退出。

---

### 52. `errgroup` 和 `sync.WaitGroup` 有什么区别？

`sync.WaitGroup`：

- 只负责等待 goroutine 结束。
- 不收集错误。
- 不自动取消其他 goroutine。

`errgroup`：

- 等待 goroutine。
- 收集第一个错误。
- 配合 context 实现任一失败取消其他任务。

如果只是等待，WaitGroup 足够。

如果多个并发任务可能失败，且失败后要取消其他任务，`errgroup` 更合适。

---

### 53. 使用 errgroup 时任务为什么还要监听 ctx？

`errgroup` 取消 ctx 只是发出信号。

任务必须监听：

```go
select {
case <-ctx.Done():
	return ctx.Err()
case result := <-work:
	return nil
}
```

如果任务内部完全不看 ctx，例如：

```go
time.Sleep(10 * time.Second)
```

即使 errgroup 取消了 ctx，任务也不会提前退出。

---

### 54. errgroup 中如何安全收集多个 goroutine 的结果？

如果每个 goroutine 写不同变量，并且 `g.Wait()` 之后再读取，通常可以接受：

```go
var user User
var orders []Order

g.Go(func() error {
	var err error
	user, err = fetchUser(ctx)
	return err
})

g.Go(func() error {
	var err error
	orders, err = fetchOrders(ctx)
	return err
})

if err := g.Wait(); err != nil {
	return err
}
```

不要多个 goroutine 同时写同一个 map 或 slice。

如果要共享写，需要加锁或用 channel 收集。

---

### 55. errgroup 如何限制并发数量？

新版 `errgroup.Group` 支持：

```go
g.SetLimit(10)
```

适合批量任务，例如 100 个用户查询，但最多同时并发 10 个。

---

## 八、源码与底层实现

### 56. `emptyCtx` 是什么？

`emptyCtx` 是根 context 的基础实现。

`context.Background()` 和 `context.TODO()` 都基于根 context。

特点：

- 没有 deadline。
- `Done()` 通常为 nil。
- `Err()` 返回 nil。
- `Value()` 返回 nil。

根 context 不会被取消。

---

### 57. 为什么根 context 的 `Done()` 可以是 nil？

因为根 context 永远不会取消。

nil channel 在 select 中永远不会就绪：

```go
select {
case <-ctx.Done():
	return ctx.Err()
case <-time.After(time.Second):
	return nil
}
```

如果 ctx 是根 context，第一个分支不会触发。

---

### 58. `cancelCtx` 大致如何实现取消？

`cancelCtx` 可以理解为保存了：

- 父 context。
- done channel。
- 取消原因 err。
- 子 context 集合。
- 锁。

取消时大致做：

```text
加锁。
如果已经取消，直接返回。
记录取消原因。
关闭 done channel。
取消所有子 context。
清理 children。
解锁。
```

这解释了为什么父取消会传播给子，以及为什么多次 cancel 安全。

---

### 59. `timerCtx` 是什么？

`timerCtx` 是支持 deadline / timeout 的 context。

可以理解为：

```text
timerCtx = cancelCtx + deadline + timer
```

`WithTimeout` 和 `WithDeadline` 内部会创建类似 `timerCtx` 的结构。

timer 到期后会触发 cancel，错误原因是：

```go
context.DeadlineExceeded
```

---

### 60. `valueCtx` 是什么？

`valueCtx` 是支持 key-value 的 context。

每次 `WithValue` 都会创建一层 `valueCtx`：

```text
valueCtx(keyC)
  -> valueCtx(keyB)
  -> valueCtx(keyA)
  -> Background
```

查找时从当前层向父层查找。

这也是为什么不建议把大量业务参数放入 context。

---

### 61. context 的取消传播是如何实现的？

创建子 context 时，标准库会把父子关系建立起来。

父 context 取消时，会通知子 context。

大致流程：

```text
WithCancel(parent)
  -> 创建 child
  -> 如果 parent 已经取消，立即取消 child
  -> 否则把 child 挂到 parent children 中
```

父取消：

```text
parent cancel
  -> close parent done
  -> cancel children
```

如果父 context 不是标准库内部可取消类型，标准库可能通过 goroutine 监听父的 Done channel 来兼容。

---

### 62. `WithCancelCause` 是什么？

`WithCancelCause` 是较新 Go 版本中的 API，用于取消时携带具体原因。

普通 `cancel()` 只能让 `ctx.Err()` 返回：

```go
context.Canceled
```

`WithCancelCause` 可以设置更具体的 cause：

```go
ctx, cancel := context.WithCancelCause(parent)
cancel(errors.New("user quota exceeded"))

fmt.Println(context.Cause(ctx))
```

适合需要保留更详细取消原因的场景。

---

### 63. `context.Cause` 和 `ctx.Err()` 有什么区别？

`ctx.Err()` 返回标准取消类型：

- `context.Canceled`
- `context.DeadlineExceeded`

`context.Cause(ctx)` 可以返回更具体的取消原因。

如果没有设置特殊 cause，通常会返回和 `Err()` 相关的原因。

面试中如果没用过，可以说明：

```text
传统代码主要用 ctx.Err；新版本 Go 可以用 WithCancelCause 和 Cause 获取更具体的取消原因。
```

---

### 64. `context.WithoutCancel` 是什么？

`WithoutCancel` 用于基于父 context 创建一个不会被父取消影响的 context。

它适合少数特殊场景，例如请求结束后还要做一个很短的后台清理任务，但仍想保留一些 value。

但要谨慎使用。

因为它会切断取消传播，滥用会导致任务在请求结束后继续运行。

一般业务代码中不要随便使用。

---

## 九、工程规范与常见错误

### 65. 使用 context 的核心规范有哪些？

常见规范：

1. `ctx context.Context` 作为第一个参数。
2. 不传 nil context。
3. 不把 context 存进结构体。
4. 不在下游随意创建 `context.Background()`。
5. 派生 cancel / timeout / deadline 后调用 `cancel()`。
6. `context.Value` 只传请求级元数据。
7. goroutine 要监听 `ctx.Done()`。
8. 外部调用要设置合理 timeout。
9. 取消时返回或包装 `ctx.Err()`。

---

### 66. 常见 context 使用错误有哪些？

常见错误：

- 忘记 `defer cancel()`。
- 用 `context.Background()` 切断请求链路。
- 把 context 放进 service 结构体。
- 把业务参数塞进 context value。
- goroutine 不监听 `ctx.Done()`。
- 下游 HTTP 请求不用 `NewRequestWithContext`。
- 数据库查询不用 `QueryContext`。
- 超时后丢失 `DeadlineExceeded` 根因。
- 所有 timeout 都返回 500。

---

### 67. 如何判断一个函数是否应该接收 context？

如果函数可能：

- 阻塞。
- 耗时。
- 访问外部依赖。
- 启动 goroutine。
- 等待 channel。
- 执行数据库 / Redis / HTTP / RPC 调用。

通常应该接收 context。

纯 CPU 小函数、简单数据转换函数不一定需要 context。

---

### 68. context 是否应该传到所有函数？

不是。

context 应该传到需要生命周期控制、外部调用、阻塞等待的函数。

例如：

```go
func formatName(first, last string) string
```

这种纯函数不需要 context。

不要为了形式把 ctx 传到所有函数里。

---

### 69. 如何处理 context 相关日志？

建议区分：

```text
context.Canceled：通常 info 或 debug。
context.DeadlineExceeded：通常 warn，需要关注。
其他错误：error。
```

日志应包含：

- request id。
- trace id。
- path。
- dependency。
- timeout。
- cost。
- error。

不要只打印：

```text
context deadline exceeded
```

这无法排查。

---

### 70. 如果线上大量 `context deadline exceeded`，怎么排查？

排查顺序：

1. 确认是哪些接口超时。
2. 根据 request id 查完整日志。
3. 找出哪个下游最慢。
4. 查看数据库慢查询。
5. 查看连接池是否耗尽。
6. 查看下游服务是否故障。
7. 查看是否最近发布导致性能退化。
8. 查看流量是否上涨。
9. 判断 timeout 是否设置过短。

不要第一反应就调大 timeout。

---

### 71. 如果线上大量 `context canceled`，一定是错误吗？

不一定。

可能原因：

- 客户端主动取消请求。
- 用户关闭页面。
- 前端路由切换。
- 网关先超时。
- errgroup 中一个任务失败后取消其他任务。
- 服务关闭。

需要结合路径、状态码、耗时、客户端日志、网关日志判断。

---

### 72. 如何排查 goroutine 泄漏？

方法：

- 观察 goroutine 数量是否持续上涨。
- 使用 pprof 查看 goroutine dump。
- 检查是否有 goroutine 卡在 channel send / receive。
- 检查是否有外部调用没有 timeout。
- 检查 worker 是否监听 ctx。
- 检查 ticker 是否 Stop。

可以用：

```go
runtime.NumGoroutine()
```

或 pprof：

```text
/debug/pprof/goroutine?debug=2
```

---

### 73. 如何测试 context 超时和取消？

测试成功：

```go
ctx, cancel := context.WithTimeout(context.Background(), time.Second)
defer cancel()
err := Work(ctx, 10*time.Millisecond)
```

测试超时：

```go
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Millisecond)
defer cancel()
err := Work(ctx, time.Second)
if !errors.Is(err, context.DeadlineExceeded) {
	t.Fatal(err)
}
```

测试主动取消：

```go
ctx, cancel := context.WithCancel(context.Background())
cancel()
err := Work(ctx, time.Second)
if !errors.Is(err, context.Canceled) {
	t.Fatal(err)
}
```

测试时间不要设置得太极端，避免 CI 不稳定。

---

## 十、项目设计题

### 74. 设计一个带 context 的 HTTP 聚合接口

题目：

```text
GET /profile/{user_id}
```

需要并发调用：

- 用户服务。
- 订单服务。
- 钱包服务。

回答思路：

1. handler 使用 `r.Context()`。
2. service 基于 ctx 派生接口总 timeout，例如 800ms。
3. 使用 `errgroup.WithContext` 并发调用三个下游。
4. 每个下游函数接收 ctx，并设置自己的子 timeout。
5. 任一关键下游失败时，errgroup 取消其他任务。
6. 使用 request id 打日志。
7. 区分 `DeadlineExceeded`、`Canceled` 和普通下游错误。

示例核心：

```go
ctx, cancel := context.WithTimeout(r.Context(), 800*time.Millisecond)
defer cancel()

g, ctx := errgroup.WithContext(ctx)
```

---

### 75. 设计一个支持优雅关闭的 HTTP 服务

回答思路：

1. 创建 HTTP server。
2. goroutine 中启动 `ListenAndServe`。
3. 使用 `signal.NotifyContext` 监听退出信号。
4. 收到信号后创建 `shutdownCtx`。
5. 调用 `server.Shutdown(shutdownCtx)`。
6. 通知 worker 停止。
7. 关闭数据库连接池。

关键点：

```go
shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
```

不要用已经取消的 signal ctx 做 Shutdown ctx。

---

### 76. 设计一个 worker pool，要求支持取消和优雅退出

回答思路：

- worker 接收 `ctx context.Context`。
- worker 循环中 select 监听 `ctx.Done()` 和 jobs channel。
- producer 在 ctx 取消时停止发送。
- 使用 WaitGroup 等待 worker 退出。
- 单个 job 如果耗时，也要接收 ctx。

核心代码：

```go
for {
	select {
	case <-ctx.Done():
		return
	case job, ok := <-jobs:
		if !ok {
			return
		}
		handleJob(ctx, job)
	}
}
```

---

### 77. 设计数据库查询超时保护

回答思路：

1. handler 从 `r.Context()` 获取 ctx。
2. service 继续传 ctx。
3. repository 为单次查询派生子 timeout。
4. 使用 `QueryRowContext` / `QueryContext`。
5. 失败时结合 `ctx.Err()` 判断是否超时。
6. handler 将超时转换成 504 或业务错误码。

示例：

```go
ctx, cancel := context.WithTimeout(ctx, 300*time.Millisecond)
defer cancel()

row := db.QueryRowContext(ctx, sql, id)
```

---

### 78. 如何设计接口 timeout budget？

回答思路：

先看上游网关超时。

例如网关 3 秒，服务接口可以设置 2 秒，预留 1 秒给网络和网关。

然后拆分内部调用：

```text
数据库：300ms
订单服务：800ms
钱包服务：500ms
响应组装：100ms
预留：300ms
```

串行调用要看总和。

并发调用也要给单个依赖设置子 timeout。

重试也要消耗总预算，不能无限重试。

---

### 79. 如果一个非关键下游超时，是否应该取消整个请求？

不一定。

要看业务重要性。

例如 profile 接口：

- 用户基础信息是关键，失败就返回错误。
- 推荐内容是非关键，失败可以降级为空列表。

设计时可以区分关键任务和非关键任务。

关键任务可以放进 `errgroup`，失败后取消整体。

非关键任务可以单独处理错误，不一定中断整个请求。

---

### 80. 如何在微服务中传递 request id / trace id？

在当前服务内：

- request id / trace id 可以放入 context value。
- 日志从 context 中读取。

跨服务时：

- HTTP：通过 header 传递，例如 `X-Request-ID`、`traceparent`。
- gRPC：通过 metadata 传递。
- OpenTelemetry：通过 propagator 自动注入和提取 trace context。

不要只把 trace id 放在本地 context，却不传给下游。

---

## 十一、开放式深挖题

### 81. context 为什么不提供 `Cancel()` 方法？

因为不是所有拿到 ctx 的代码都应该有权取消它。

取消权应该由创建 context 的上层持有：

```go
ctx, cancel := context.WithCancel(parent)
```

下游只接收 ctx，监听取消信号。

如果 `Context` 接口本身有 `Cancel()`，任何下游都能取消上游 context，容易破坏调用链控制权。

---

### 82. 为什么 context 取消是从父到子，而不是双向？

因为父 context 通常代表更大的生命周期，例如整个请求。

子 context 代表局部操作，例如数据库查询。

局部操作失败，不一定意味着整个请求必须取消。

例如数据库查询超时后，业务可能走缓存降级。

所以默认是：

```text
父取消影响子。
子取消不影响父。
```

如果需要子失败影响其他任务，应由业务层显式控制。

---

### 83. context 能替代 channel 吗？

不能完全替代。

context 适合传递取消、超时、请求级元数据。

channel 适合传递业务数据、任务、结果、信号。

常见组合：

```go
select {
case result := <-resultCh:
	return result, nil
case <-ctx.Done():
	return nil, ctx.Err()
}
```

context 管生命周期，channel 传业务结果。

---

### 84. context 能替代配置系统或依赖注入吗？

不能。

context 不应该保存：

- config。
- db。
- repo。
- service。
- client。

配置应该来自配置系统。

依赖应该通过结构体字段或构造函数注入。

context 只适合请求级控制信息和少量横切元数据。

---

### 85. context 使用过多会有什么问题？

如果滥用 context，可能导致：

- 函数签名不表达真实依赖。
- 业务参数隐藏在 value 中。
- 难以测试。
- value key 冲突。
- 调试困难。
- 大量无意义派生 context。

所以 context 应该用在该用的地方，不是所有函数都必须传 ctx。

---

## 十二、面试回答模板

### 86. 如何用一句话解释 context？

可以回答：

```text
context 是 Go 中用于在请求链路上传递取消信号、超时截止时间和请求级元数据的标准机制，常用于控制 goroutine、数据库查询、HTTP/RPC 调用等下游任务的生命周期。
```

---

### 87. 如何解释“context 是协作式取消”？

可以回答：

```text
调用 cancel 不会强制杀死 goroutine，它只是关闭 ctx.Done() 发出取消信号。任务本身必须在 select 中监听 ctx.Done()，收到信号后主动返回。所以 context 的取消是协作式的。
```

---

### 88. 如何解释“不要把 context 存进结构体”？

可以回答：

```text
context 通常是请求级生命周期，而 service、repository 这类结构体通常是应用级生命周期。把请求级 context 存进应用级对象会导致生命周期混乱，甚至复用已经取消的旧 context。更推荐把 context 作为方法第一个参数显式传递。
```

---

### 89. 如何解释“不要用 context 传业务参数”？

可以回答：

```text
context.Value 适合传 request id、trace id、认证用户这类请求级横切元数据，不适合传 userID、orderID 这种业务必需参数。业务参数应该出现在函数签名里，否则函数依赖会变得隐式，测试和维护都会变困难。
```

---

### 90. 如何解释“为什么 WithTimeout 后还要 cancel”？

可以回答：

```text
WithTimeout 内部会创建 timer。即使超时后会自动取消，也应该 defer cancel，这样在函数提前返回时可以及时释放 timer 等内部资源，并解除父子 context 的关联。这是 Go 中使用 timeout context 的标准习惯。
```

---

## 十三、快速复习清单

面试前可以快速过一遍：

- [ ] `Context` 接口四个方法。
- [ ] `Background` 和 `TODO` 区别。
- [ ] `WithCancel`、`WithTimeout`、`WithDeadline` 区别。
- [ ] `Canceled` 和 `DeadlineExceeded` 区别。
- [ ] 父子 context 取消传播方向。
- [ ] 为什么 `defer cancel()`。
- [ ] 为什么不把 context 存结构体。
- [ ] 为什么不传 nil context。
- [ ] 为什么不用 context 传业务参数。
- [ ] HTTP 中使用 `r.Context()`。
- [ ] HTTP client 使用 `NewRequestWithContext`。
- [ ] DB 使用 `QueryContext`。
- [ ] `signal.NotifyContext` 和优雅关闭。
- [ ] `errgroup.WithContext` 的作用。
- [ ] context 源码中的 `cancelCtx`、`timerCtx`、`valueCtx`。
- [ ] 如何排查大量 `context deadline exceeded`。
- [ ] 如何避免 goroutine 泄漏。

---

## 十四、最终建议

面试中回答 context 问题时，尽量遵循这个结构：

```text
先说概念。
再说 API。
再说工程场景。
最后说常见坑。
```

例如问 `WithTimeout`：

```text
它用于创建一个带超时的子 context。
时间到了 Done 会关闭，Err 返回 DeadlineExceeded。
常用于数据库、HTTP、RPC 调用超时控制。
使用后要 defer cancel，避免 timer 资源不能及时释放。
任务内部还要监听 ctx.Done，否则不会真正停止。
```

这种回答比只说“设置超时”更完整。
