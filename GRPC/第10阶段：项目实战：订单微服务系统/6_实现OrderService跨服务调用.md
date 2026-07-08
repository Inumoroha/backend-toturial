# 6. 实现 OrderService 跨服务调用

本节目标：在订单服务中调用用户服务和商品服务，完成创建订单链路。

OrderService 是项目核心。它不应该复制用户和商品数据，而应该通过 gRPC client 调用 UserService 和 ProductService。这里开始真正练习服务间调用。

---

## 一、OrderService 依赖

```go
type orderServer struct {
    orderv1.UnimplementedOrderServiceServer
    users    userv1.UserServiceClient
    products productv1.ProductServiceClient

    mu     sync.RWMutex
    nextID int64
    orders map[int64]*orderv1.Order
}
```

OrderService 持有下游 client，并复用连接。

---

## 二、创建订单流程

```text
校验请求参数
-> 调 user-service.GetUser
-> 调 product-service.GetProduct
-> 调 product-service.DeductStock
-> 创建订单
-> 保存订单
-> 返回订单
```

每个下游调用都要设置 timeout。

---

## 三、调用下游示例

```go
userCtx, cancel := context.WithTimeout(ctx, 300*time.Millisecond)
defer cancel()

_, err := s.users.GetUser(userCtx, &userv1.GetUserRequest{Id: req.UserId})
if err != nil {
    return nil, err
}
```

扣库存：

```go
stockCtx, cancel := context.WithTimeout(ctx, 500*time.Millisecond)
defer cancel()

_, err = s.products.DeductStock(stockCtx, &productv1.DeductStockRequest{
    ProductId: req.ProductId,
    Quantity:  req.Quantity,
})
if err != nil {
    return nil, err
}
```

---

## 四、错误处理建议

下游错误不一定都原样返回。可以按规则转换：

- 用户不存在：返回 `FailedPrecondition` 或透传 `NotFound`，取决于 API 语义。
- 商品不存在：通常返回 `FailedPrecondition` 或 `NotFound`。
- 库存不足：返回 `FailedPrecondition`。
- 下游不可用：返回 `Unavailable`。
- 超时：返回 `DeadlineExceeded`。

团队要统一规范。

---

## 五、常见问题

- 下游调用复用上游 context 但没有子超时：单个下游可能耗尽全部时间。
- 订单创建失败后库存已扣：真实项目需要事务、Saga 或补偿，本学习项目先记录风险。
- OrderService 直接访问 user 内存数据：破坏服务边界。
- 每次请求新建 gRPC 连接：性能差，资源浪费。

---

## 六、练习任务

1. 实现 CreateOrder。
2. 模拟用户不存在、商品不存在、库存不足。
3. 为每个下游调用设置 timeout。
4. 写一个创建订单成功测试。

---

## 七、完成标准

- OrderService 能跨服务调用。
- 创建订单链路跑通。
- 下游错误能被正确处理。

---

## 八、完整操作步骤：把三个服务串成订单链路

OrderService 是项目核心。它自己不保存用户和商品真相，而是通过 gRPC client 调用下游服务。

实现顺序建议如下：

1. 先启动 user-service 和 product-service。
2. 在 order-service 中建立到两个下游的 gRPC 连接。
3. 在 `internal/app/order` 中实现 `Service`。
4. `CreateOrder` 内先校验参数。
5. 调用 UserService.GetUser 确认用户存在。
6. 调用 ProductService.GetProduct 获取价格。
7. 调用 ProductService.DeductStock 扣库存。
8. 创建订单并保存到内存 map。
9. 返回订单给客户端。

创建目录：

```powershell
mkdir .\internal\app\order -Force
New-Item -ItemType File .\internal\app\order\service.go -Force
```

---

## 九、完整代码：OrderService 跨服务调用

`internal/app/order/service.go`：

```go
package order

import (
	"context"
	"sync"
	"time"

	orderv1 "example.com/grpc-shop/gen/order/v1"
	productv1 "example.com/grpc-shop/gen/product/v1"
	userv1 "example.com/grpc-shop/gen/user/v1"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
)

type Service struct {
	orderv1.UnimplementedOrderServiceServer

	users    userv1.UserServiceClient
	products productv1.ProductServiceClient

	mu     sync.RWMutex
	nextID int64
	orders map[int64]*orderv1.Order
}

func NewService(users userv1.UserServiceClient, products productv1.ProductServiceClient) *Service {
	return &Service{
		users:    users,
		products: products,
		nextID:   1,
		orders:   make(map[int64]*orderv1.Order),
	}
}

func (s *Service) CreateOrder(ctx context.Context, req *orderv1.CreateOrderRequest) (*orderv1.CreateOrderResponse, error) {
	if req.GetUserId() <= 0 || req.GetProductId() <= 0 || req.GetQuantity() <= 0 {
		return nil, status.Error(codes.InvalidArgument, "user_id, product_id and quantity are required")
	}

	callCtx, cancel := context.WithTimeout(ctx, 800*time.Millisecond)
	defer cancel()

	if _, err := s.users.GetUser(callCtx, &userv1.GetUserRequest{Id: req.GetUserId()}); err != nil {
		return nil, err
	}

	productResp, err := s.products.GetProduct(callCtx, &productv1.GetProductRequest{Id: req.GetProductId()})
	if err != nil {
		return nil, err
	}

	if _, err := s.products.DeductStock(callCtx, &productv1.DeductStockRequest{
		ProductId: req.GetProductId(),
		Quantity:  req.GetQuantity(),
		RequestId: req.GetRequestId(),
	}); err != nil {
		return nil, err
	}

	total := productResp.GetProduct().GetPriceCents() * int64(req.GetQuantity())

	s.mu.Lock()
	defer s.mu.Unlock()

	id := s.nextID
	s.nextID++
	order := &orderv1.Order{
		Id:         id,
		UserId:     req.GetUserId(),
		ProductId:  req.GetProductId(),
		Quantity:   req.GetQuantity(),
		TotalCents: total,
		Status:     orderv1.OrderStatus_ORDER_STATUS_CREATED,
	}
	s.orders[id] = order

	return &orderv1.CreateOrderResponse{Order: cloneOrder(order)}, nil
}

func (s *Service) GetOrder(ctx context.Context, req *orderv1.GetOrderRequest) (*orderv1.GetOrderResponse, error) {
	if req.GetId() <= 0 {
		return nil, status.Error(codes.InvalidArgument, "id must be positive")
	}

	s.mu.RLock()
	defer s.mu.RUnlock()

	order, ok := s.orders[req.GetId()]
	if !ok {
		return nil, status.Error(codes.NotFound, "order not found")
	}
	return &orderv1.GetOrderResponse{Order: cloneOrder(order)}, nil
}

func cloneOrder(o *orderv1.Order) *orderv1.Order {
	if o == nil {
		return nil
	}
	return &orderv1.Order{
		Id:         o.Id,
		UserId:     o.UserId,
		ProductId:  o.ProductId,
		Quantity:   o.Quantity,
		TotalCents: o.TotalCents,
		Status:     o.Status,
	}
}
```

在 `cmd/order-service/main.go` 中复用一个长连接，不要每次请求新建连接：

```go
userConn, err := grpc.Dial("localhost:50051", grpc.WithInsecure())
if err != nil {
	log.Fatalf("dial user-service: %v", err)
}
defer userConn.Close()

productConn, err := grpc.Dial("localhost:50052", grpc.WithInsecure())
if err != nil {
	log.Fatalf("dial product-service: %v", err)
}
defer productConn.Close()

svc := order.NewService(
	userv1.NewUserServiceClient(userConn),
	productv1.NewProductServiceClient(productConn),
)
orderv1.RegisterOrderServiceServer(grpcServer, svc)
```

---

## 十、运行命令与预期输出

分别打开三个 PowerShell 窗口：

```powershell
go run .\cmd\user-service
```

```powershell
go run .\cmd\product-service
```

```powershell
go run .\cmd\order-service
```

创建订单：

```powershell
grpcurl -plaintext -d '{\"userId\":1,\"productId\":1,\"quantity\":2,\"requestId\":\"order-001\"}' localhost:50053 order.v1.OrderService/CreateOrder
```

预期输出：

```json
{
  "order": {
    "id": "1",
    "userId": "1",
    "productId": "1",
    "quantity": 2,
    "totalCents": "19800",
    "status": "ORDER_STATUS_CREATED"
  }
}
```

查询订单：

```powershell
grpcurl -plaintext -d '{\"id\":1}' localhost:50053 order.v1.OrderService/GetOrder
```

---

## 十一、常见错误排查

- order-service 启动时报连接失败：先启动 user-service 和 product-service。
- 每次 CreateOrder 都很慢：检查是否每个请求都 Dial，下游连接应该复用。
- 用户不存在时仍然扣库存：调用顺序错误，应先确认用户和商品，再扣库存。
- 下游错误被改成 Internal：学习阶段可以透传下游错误，至少不要丢失 `NotFound` 和 `FailedPrecondition`。
- 库存扣了但订单没创建：这是跨服务一致性风险，教学项目要记录，真实项目需要 Saga、Outbox 或事务消息。

---

## 十二、教程闭环检查

本篇的完整操作步骤是：启动下游、创建 gRPC client、实现订单服务、调用用户服务、调用商品服务、扣库存、创建订单、查询订单。完整代码包含 OrderService、下游 client、超时控制、订单 map 和错误透传。运行命令包含三个服务启动命令和两条 grpcurl。预期输出覆盖创建订单成功和查询订单成功。常见错误排查覆盖启动顺序、连接复用、调用顺序、错误码和一致性风险。练习任务是模拟用户不存在、商品不存在、库存不足三种失败路径。完成标准是：创建订单链路可以真实跨服务跑通。
