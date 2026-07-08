# Go Context 系统学习路线图

## 目标定位

这份路线图面向想成为 Go 后端工程师的学习者。学完以后，你应该能够：

- 理解 `context.Context` 的设计目的和边界。
- 在 HTTP、RPC、数据库、缓存、消息队列等后端场景中正确传递 `context`。
- 使用 `WithCancel`、`WithTimeout`、`WithDeadline`、`WithValue` 解决真实工程问题。
- 避免常见错误，例如滥用 `context.Value`、忘记调用 `cancel`、把 `context` 存进结构体。
- 能读懂并解释标准库 `context` 的核心实现。
- 能在项目中设计超时、取消、链路追踪、请求级元数据传递方案。

---

## 学习前置要求

建议先掌握以下内容：

- Go 基础语法：函数、接口、结构体、错误处理。
- Goroutine 和 Channel 的基本使用。
- HTTP 服务基础：`net/http`、请求处理函数、中间件。
- 数据库基础：`database/sql` 或常见 ORM 的基本查询。
- 基本并发问题：阻塞、泄漏、超时、取消。

如果这些还不熟，可以先补：

- Go 并发模型：goroutine、channel、select。
- HTTP 请求生命周期。
- 后端服务中的超时控制和资源释放。

---

## 总体学习路径

```text
基础认知
  -> API 使用
  -> 并发取消
  -> 超时与 Deadline
  -> 请求链路传递
  -> 工程实践
  -> 源码理解
  -> 项目实战
```

建议学习周期：2 到 4 周。

如果每天学习 1 到 2 小时，可以按下面的四阶段推进。

---

## 第一阶段：理解 Context 是什么

### 学习目标

理解 `context` 不是普通参数，而是用来在请求链路中传递：

- 取消信号。
- 超时时间。
- 截止时间。
- 请求级元数据。

### 核心问题

你需要能回答：

- 为什么 Go 需要 `context`？
- `context.Context` 是接口还是结构体？
- 为什么函数通常把 `ctx context.Context` 放在第一个参数？
- `context.Background()` 和 `context.TODO()` 有什么区别？
- 为什么不要把 `context` 放进结构体字段？

### 必学 API

```go
context.Background()
context.TODO()
ctx.Done()
ctx.Err()
ctx.Deadline()
ctx.Value(key)
```

### 练习任务

1. 写一个函数，接收 `ctx context.Context`，打印是否有 deadline。
2. 使用 `context.Background()` 创建根 context。
3. 使用 `context.TODO()` 标记暂时还没确定来源的 context。
4. 写一个 `select`，同时监听业务 channel 和 `ctx.Done()`。

### 示例代码

```go
func work(ctx context.Context) error {
	select {
	case <-time.After(2 * time.Second):
		fmt.Println("work done")
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}
```

---

## 第二阶段：掌握取消控制

### 学习目标

掌握主动取消 goroutine 的方式，避免 goroutine 泄漏。

### 必学 API

```go
context.WithCancel(parent)
cancel()
```

### 关键理解

`WithCancel` 会基于父 context 派生一个子 context。

调用 `cancel()` 后：

- 当前 context 的 `Done()` channel 会被关闭。
- 所有派生自它的子 context 都会收到取消信号。
- 正在监听 `ctx.Done()` 的 goroutine 应该尽快退出。

### 常见场景

- 用户断开连接后停止后台任务。
- 一个任务失败后取消其他并发任务。
- 服务关闭时通知工作 goroutine 退出。

### 练习任务

1. 启动 3 个 goroutine，每个都监听同一个 `ctx.Done()`。
2. 主 goroutine 等待 1 秒后调用 `cancel()`。
3. 观察 3 个 goroutine 是否全部退出。
4. 故意不监听 `ctx.Done()`，观察 goroutine 泄漏风险。

### 示例代码

```go
func worker(ctx context.Context, id int) {
	for {
		select {
		case <-ctx.Done():
			fmt.Printf("worker %d stopped: %v\n", id, ctx.Err())
			return
		default:
			fmt.Printf("worker %d working\n", id)
			time.Sleep(300 * time.Millisecond)
		}
	}
}
```

---

## 第三阶段：掌握超时和截止时间

### 学习目标

理解后端系统中所有外部调用都应该有超时边界。

### 必学 API

```go
context.WithTimeout(parent, duration)
context.WithDeadline(parent, deadline)
```

### 关键理解

`WithTimeout` 是最常用的超时控制方式。

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()
```

即使 context 会自动超时，也仍然应该调用 `cancel()`，这样可以及时释放内部计时器资源。

### 常见场景

- HTTP 客户端请求超时。
- 数据库查询超时。
- Redis 请求超时。
- RPC 调用超时。
- 下游服务整体调用预算控制。

### 练习任务

1. 模拟一个耗时 3 秒的任务，用 1 秒 timeout 取消它。
2. 模拟数据库查询，使用 `QueryContext`。
3. 模拟 HTTP 请求，使用 `http.NewRequestWithContext`。
4. 比较 `WithTimeout` 和 `WithDeadline` 的使用差异。

### 示例代码：HTTP 请求超时

```go
ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
defer cancel()

req, err := http.NewRequestWithContext(ctx, http.MethodGet, "https://example.com", nil)
if err != nil {
	return err
}

resp, err := http.DefaultClient.Do(req)
if err != nil {
	return err
}
defer resp.Body.Close()
```

### 示例代码：数据库查询超时

```go
ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
defer cancel()

rows, err := db.QueryContext(ctx, "SELECT id, name FROM users WHERE status = ?", "active")
if err != nil {
	return err
}
defer rows.Close()
```

---

## 第四阶段：理解请求链路中的 Context

### 学习目标

掌握后端服务中 context 的传递方式。

一条典型链路可能是：

```text
HTTP Request
  -> Middleware
  -> Handler
  -> Service
  -> Repository
  -> Database / Redis / RPC
```

在这条链路里，`ctx` 应该从入口一路向下传递。

### 推荐函数签名

```go
func (s *UserService) GetUser(ctx context.Context, userID int64) (*User, error)
func (r *UserRepo) FindByID(ctx context.Context, userID int64) (*User, error)
```

### 不推荐写法

```go
type UserService struct {
	ctx context.Context
}
```

原因：

- context 是请求级别的，不是服务级别的。
- 容易混淆生命周期。
- 容易导致旧请求的取消信号影响新请求。

### 练习任务

1. 写一个三层结构：handler、service、repository。
2. handler 从 `r.Context()` 获取 context。
3. service 和 repository 都显式接收 `ctx context.Context`。
4. repository 中用 `QueryContext` 模拟数据库调用。

### 示例代码

```go
func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()

	user, err := h.userService.GetUser(ctx, 1001)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	json.NewEncoder(w).Encode(user)
}
```

---

## 第五阶段：谨慎使用 Context Value

### 学习目标

理解 `context.Value` 只能存放请求级元数据，不应该存放业务参数。

### 适合放入 Context 的数据

- request id。
- trace id。
- auth token 或已解析的用户身份。
- logger 中需要的请求级字段。
- 链路追踪上下文。

### 不适合放入 Context 的数据

- 函数必需的业务参数。
- 可选配置项。
- 数据库连接。
- service、repository 等依赖对象。
- 大对象或可变对象。

### 推荐 key 写法

```go
type contextKey string

const requestIDKey contextKey = "request_id"

func WithRequestID(ctx context.Context, requestID string) context.Context {
	return context.WithValue(ctx, requestIDKey, requestID)
}

func RequestIDFromContext(ctx context.Context) (string, bool) {
	v, ok := ctx.Value(requestIDKey).(string)
	return v, ok
}
```

### 练习任务

1. 写一个中间件，给每个请求生成 request id。
2. 将 request id 放入 context。
3. handler、service、repository 都能从 context 里读取 request id。
4. 封装 `WithRequestID` 和 `RequestIDFromContext`，不要到处直接调用 `ctx.Value`。

---

## 第六阶段：结合标准库和常用组件

### net/http

重点掌握：

```go
r.Context()
http.NewRequestWithContext()
```

理解：

- 客户端断开连接时，服务端 request context 会被取消。
- 服务端超时可以影响请求 context。
- 下游调用应该复用或派生当前请求的 context。

### database/sql

重点掌握：

```go
db.QueryContext()
db.ExecContext()
db.BeginTx()
stmt.QueryContext()
```

理解：

- 数据库驱动需要支持 context 才能真正取消查询。
- 超时后要正确处理 `context.DeadlineExceeded`。

### os/signal

重点掌握：

```go
signal.NotifyContext()
```

适用于服务优雅关闭。

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

<-ctx.Done()
```

### errgroup

建议学习：

```go
golang.org/x/sync/errgroup
```

适用于多个 goroutine 协同执行，任意一个失败后取消其他任务。

---

## 第七阶段：阅读 Context 源码

### 学习目标

读懂 `context` 包的核心实现，而不是只会调用 API。

### 阅读顺序

1. `Context` 接口。
2. `emptyCtx`。
3. `cancelCtx`。
4. `timerCtx`。
5. `valueCtx`。
6. `propagateCancel`。
7. `WithCancel`。
8. `WithDeadline` / `WithTimeout`。
9. `WithValue`。

### 重点问题

- `Done()` 为什么返回 `<-chan struct{}`？
- `cancelCtx` 如何通知子 context？
- 父 context 取消时，子 context 如何被取消？
- `timerCtx` 如何实现定时取消？
- `valueCtx` 为什么是链式查找？
- `context` 的取消为什么是单向传播？

### 建议源码阅读方式

先不用一次读完全部实现。可以按 API 反推源码：

```text
我调用 WithTimeout
  -> 它创建了什么结构？
  -> 它什么时候关闭 Done channel？
  -> cancel 函数做了什么？
  -> 父 context 取消时它怎么感知？
```

---

## 第八阶段：工程规范和常见错误

### 必须养成的习惯

1. `ctx context.Context` 通常作为函数第一个参数。
2. 派生了可取消 context 后，及时 `defer cancel()`。
3. 不要传 `nil` context，不确定时用 `context.TODO()`。
4. 不要把 context 存进结构体。
5. 不要用 context 传业务必需参数。
6. 下游调用应该接收上游传来的 context。
7. goroutine 中要监听 `ctx.Done()`。
8. 对外部服务调用设置合理 timeout。

### 常见错误示例

错误：忘记 cancel。

```go
ctx, _ := context.WithTimeout(context.Background(), time.Second)
doSomething(ctx)
```

正确：

```go
ctx, cancel := context.WithTimeout(context.Background(), time.Second)
defer cancel()

doSomething(ctx)
```

错误：业务参数塞进 context。

```go
ctx = context.WithValue(ctx, "user_id", userID)
service.GetUser(ctx)
```

更推荐：

```go
service.GetUser(ctx, userID)
```

错误：忽略取消信号。

```go
func loop() {
	for {
		doWork()
	}
}
```

更推荐：

```go
func loop(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			return
		default:
			doWork()
		}
	}
}
```

---

## 第九阶段：项目实战

### 项目一：带超时控制的 HTTP 聚合服务

实现一个接口：

```text
GET /profile/{user_id}
```

内部并发调用：

- 用户服务。
- 订单服务。
- 钱包服务。

要求：

- 整个接口总超时 800ms。
- 三个下游调用并发执行。
- 任意关键调用失败时取消其他调用。
- 返回错误时区分普通错误和超时错误。
- 日志中打印 request id。

你会练到：

- `WithTimeout`。
- `errgroup.WithContext`。
- request context 传递。
- 超时错误处理。
- 日志上下文。

### 项目二：支持优雅关闭的 Worker 服务

实现一个后台任务服务：

- 主程序监听 `SIGINT` / `SIGTERM`。
- 收到退出信号后取消根 context。
- worker 停止接收新任务。
- 正在执行的任务最多等待 5 秒。
- 超时后强制退出。

你会练到：

- `signal.NotifyContext`。
- 根 context 管理。
- goroutine 退出。
- shutdown timeout。

### 项目三：数据库查询超时保护

实现一个简单用户查询接口：

```text
GET /users/{id}
```

要求：

- handler 使用 `r.Context()`。
- service 继续传递 ctx。
- repository 使用 `QueryContext`。
- 数据库查询设置 300ms 子超时。
- 超时返回明确错误。

你会练到：

- 分层传递 context。
- 为单个下游调用派生更短 timeout。
- 正确处理 `context.DeadlineExceeded`。

---

## 推荐学习顺序清单

### 第 1 周：基础和 API

- [ ] 理解 `context.Context` 接口。
- [ ] 掌握 `Background` 和 `TODO`。
- [ ] 掌握 `Done`、`Err`、`Deadline`、`Value`。
- [ ] 写 3 个取消控制 demo。
- [ ] 写 3 个超时控制 demo。

### 第 2 周：后端工程场景

- [ ] 在 HTTP handler 中使用 `r.Context()`。
- [ ] 在 service、repository 中传递 ctx。
- [ ] 使用 `QueryContext` / `ExecContext`。
- [ ] 使用 `http.NewRequestWithContext`。
- [ ] 封装 request id context 工具函数。

### 第 3 周：并发和服务治理

- [ ] 使用 `errgroup.WithContext`。
- [ ] 实现并发下游调用。
- [ ] 实现服务优雅关闭。
- [ ] 学会区分 `context.Canceled` 和 `context.DeadlineExceeded`。
- [ ] 梳理服务超时预算。

### 第 4 周：源码和项目

- [ ] 阅读 `context` 标准库源码。
- [ ] 理解 `cancelCtx`、`timerCtx`、`valueCtx`。
- [ ] 完成 HTTP 聚合服务项目。
- [ ] 完成 Worker 优雅关闭项目。
- [ ] 总结自己的 context 使用规范。

---

## 面试高频问题

1. Go 的 `context` 主要解决什么问题？
2. `context.Background()` 和 `context.TODO()` 有什么区别？
3. 为什么 `context` 通常作为函数第一个参数？
4. 为什么不要把 `context` 存在结构体里？
5. `WithCancel`、`WithTimeout`、`WithDeadline` 的区别是什么？
6. 为什么创建 timeout context 后还要调用 `cancel()`？
7. `ctx.Done()` 返回的是什么？什么时候会关闭？
8. 父 context 取消后，子 context 会怎样？
9. 子 context 取消后，父 context 会怎样？
10. `context.Value` 适合存什么？不适合存什么？
11. 如何在 HTTP 请求中传递 request id？
12. 如何用 context 控制数据库查询超时？
13. `context.Canceled` 和 `context.DeadlineExceeded` 有什么区别？
14. 如何避免 goroutine 泄漏？
15. `errgroup.WithContext` 和普通 `sync.WaitGroup` 有什么区别？

---

## 推荐代码规范

### 函数签名

```go
func DoSomething(ctx context.Context, arg Arg) error
```

### Handler 到 Service

```go
func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	_ = h.service.Do(ctx)
}
```

### Service 到 Repository

```go
func (s *Service) Do(ctx context.Context) error {
	return s.repo.Save(ctx)
}
```

### Repository 到 Database

```go
func (r *Repo) Save(ctx context.Context) error {
	_, err := r.db.ExecContext(ctx, "INSERT INTO logs(message) VALUES(?)", "hello")
	return err
}
```

### 派生更短超时

```go
func (r *Repo) Find(ctx context.Context, id int64) (*User, error) {
	queryCtx, cancel := context.WithTimeout(ctx, 300*time.Millisecond)
	defer cancel()

	row := r.db.QueryRowContext(queryCtx, "SELECT id, name FROM users WHERE id = ?", id)
	// scan result...
	return user, nil
}
```

---

## 最终能力检查

完成学习后，你可以用下面的问题检查自己：

- 我能不能解释一个 HTTP 请求的 context 生命周期？
- 我能不能写出一个不会泄漏 goroutine 的并发任务？
- 我能不能为数据库、HTTP、RPC 调用设置合理超时？
- 我能不能解释 `WithTimeout` 内部大概做了什么？
- 我能不能说清楚什么时候该用 `context.Value`，什么时候不该用？
- 我能不能设计一个完整服务的超时预算？

如果这些都能回答清楚，说明你已经可以在 Go 后端工程中比较成熟地使用 `context` 了。

---

## 后续进阶方向

学完 `context` 后，建议继续学习：

- Go 并发模式。
- `errgroup` 和并发错误处理。
- HTTP 中间件设计。
- gRPC metadata 和 deadline。
- OpenTelemetry 链路追踪。
- 服务优雅关闭。
- 超时、重试、熔断、限流。
- 数据库连接池和慢查询治理。

这些内容会和 `context` 一起构成 Go 后端工程师非常核心的一组能力。
