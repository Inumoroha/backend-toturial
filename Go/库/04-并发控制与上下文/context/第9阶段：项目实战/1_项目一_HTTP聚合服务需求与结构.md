# 1. 项目一：HTTP 聚合服务需求与结构

本节目标：设计一个使用 context 控制生命周期的 HTTP 聚合接口。

接口：

```text
GET /profile/1001
```

内部并发获取：

- 用户信息。
- 最近订单。
- 钱包余额。

---

## 一、项目目标

最终实现：

```text
GET /profile/{user_id}
```

响应示例：

```json
{
  "user": {"id": 1001, "name": "Tom"},
  "orders": [{"id": 1, "amount": 99}],
  "wallet": {"balance": 188}
}
```

---

## 二、Context 要求

要求：

- handler 从 `r.Context()` 获取 ctx。
- 整个接口总超时 800ms。
- 三个下游调用并发执行。
- 任一关键调用失败时取消其他调用。
- request id 进入日志。
- 能区分超时错误和普通错误。

---

## 三、推荐项目结构

```text
context-profile/
  go.mod
  cmd/
    api/
      main.go
  internal/
    requestctx/
      context.go
    profile/
      handler.go
      service.go
      model.go
      downstream.go
```

初学阶段也可以先放在一个 `main.go`。

但建议至少理解这些职责：

- `handler`：HTTP 输入输出。
- `service`：聚合业务。
- `downstream`：模拟下游调用。
- `requestctx`：封装 request id。

---

## 四、数据模型

```go
type User struct {
	ID   int64  `json:"id"`
	Name string `json:"name"`
}

type Order struct {
	ID     int64 `json:"id"`
	Amount int64 `json:"amount"`
}

type Wallet struct {
	Balance int64 `json:"balance"`
}

type Profile struct {
	User   User    `json:"user"`
	Orders []Order `json:"orders"`
	Wallet Wallet  `json:"wallet"`
}
```

---

## 五、模拟下游调用

```go
func fetchUser(ctx context.Context, userID int64) (User, error) {
	select {
	case <-time.After(100 * time.Millisecond):
		return User{ID: userID, Name: "Tom"}, nil
	case <-ctx.Done():
		return User{}, fmt.Errorf("fetch user stopped: %w", ctx.Err())
	}
}
```

其他下游类似。

---

## 六、本节达标标准

学完本节后，你应该能够做到：

- 说清楚聚合接口要做什么。
- 设计基本项目结构。
- 定义聚合响应模型。
- 写出一个支持 ctx 的模拟下游函数。

---

## 七、接口调用链设计

这个项目的调用链可以画成：

```text
HTTP GET /profile/1001
  -> ProfileHandler.GetProfile
  -> ProfileService.GetProfile
     -> fetchUser
     -> fetchOrders
     -> fetchWallet
  -> JSON Response
```

context 流向：

```text
r.Context()
  -> handler ctx
  -> service ctx with 800ms timeout
  -> errgroup ctx
  -> 三个下游调用
```

这个流向是项目的主线。

---

## 八、完整文件拆分建议

如果拆文件，可以这样写：

```text
internal/profile/model.go
  User
  Order
  Wallet
  Profile

internal/profile/downstream.go
  fetchUser
  fetchOrders
  fetchWallet

internal/profile/service.go
  ProfileService.GetProfile

internal/profile/handler.go
  ProfileHandler.GetProfile
```

每个文件职责单一，后续补日志和错误处理更容易。

---

## 九、下游函数完整示例

```go
func fetchOrders(ctx context.Context, userID int64) ([]Order, error) {
	select {
	case <-time.After(500 * time.Millisecond):
		return []Order{
			{ID: 1, Amount: 99},
			{ID: 2, Amount: 188},
		}, nil
	case <-ctx.Done():
		return nil, fmt.Errorf("fetch orders stopped: %w", ctx.Err())
	}
}

func fetchWallet(ctx context.Context, userID int64) (Wallet, error) {
	select {
	case <-time.After(300 * time.Millisecond):
		return Wallet{Balance: 188}, nil
	case <-ctx.Done():
		return Wallet{}, fmt.Errorf("fetch wallet stopped: %w", ctx.Err())
	}
}
```

注意每个函数都必须监听 `ctx.Done()`。

---

## 十、运行前的最小 main

```go
func main() {
	service := &ProfileService{}
	handler := &ProfileHandler{service: service}

	mux := http.NewServeMux()
	mux.HandleFunc("/profile/1001", handler.GetProfile)

	fmt.Println("listen on :8080")
	if err := http.ListenAndServe(":8080", mux); err != nil {
		fmt.Println(err)
	}
}
```

运行：

```powershell
go run ./cmd/api
```

访问：

```powershell
curl http://localhost:8080/profile/1001
```

---

## 十一、常见设计错误

### 1. 三个下游串行调用

如果串行：

```text
100ms + 500ms + 300ms = 900ms
```

已经超过 800ms 总超时。

所以这个项目应该并发调用。

### 2. 下游函数不接收 ctx

那 errgroup 取消时它们不会及时停止。

### 3. handler 直接写所有逻辑

初学可以先写在一起，但最终建议拆 service。否则 handler 会变成大杂烩。

---

## 十二、本节练习

请先不使用 errgroup，写出：

1. 数据模型。
2. 三个模拟下游函数。
3. handler 和 service 空结构。
4. main 启动 HTTP 服务。

下一节再把 service 中的并发聚合补完整。
