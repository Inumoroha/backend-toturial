# 5. 实现 ProductService 与库存扣减

本节目标：实现商品查询和库存扣减，为创建订单做准备。

ProductService 比 UserService 多一个关键点：库存扣减。即使用内存实现，也要考虑并发一致性和错误语义。

---

## 一、功能设计

- `GetProduct`：查询商品。
- `DeductStock`：扣减库存。
- 商品不存在返回 `NotFound`。
- 参数非法返回 `InvalidArgument`。
- 库存不足返回 `FailedPrecondition`。

---

## 二、内存模型

```go
type productServer struct {
    productv1.UnimplementedProductServiceServer
    mu       sync.Mutex
    products map[int64]*productv1.Product
}
```

库存扣减必须加锁，因为多个订单可能同时购买同一个商品。

---

## 三、扣减库存示例

```go
func (s *productServer) DeductStock(ctx context.Context, req *productv1.DeductStockRequest) (*productv1.DeductStockResponse, error) {
    if req.ProductId <= 0 || req.Quantity <= 0 {
        return nil, status.Error(codes.InvalidArgument, "invalid product id or quantity")
    }

    s.mu.Lock()
    defer s.mu.Unlock()

    p := s.products[req.ProductId]
    if p == nil {
        return nil, status.Error(codes.NotFound, "product not found")
    }
    if p.Stock < req.Quantity {
        return nil, status.Error(codes.FailedPrecondition, "stock not enough")
    }

    p.Stock -= req.Quantity
    return &productv1.DeductStockResponse{RemainingStock: p.Stock}, nil
}
```

---

## 四、并发一致性说明

学习项目使用互斥锁保护内存库存。真实项目通常会使用数据库事务、乐观锁、Redis 原子操作或专门库存服务。

如果 `DeductStock` 被客户端重试，可能重复扣减。生产系统通常需要：

- 幂等键。
- 订单号去重。
- 事务表。
- 补偿逻辑。

---

## 五、常见问题

- 扣库存不加锁：并发下可能超卖。
- 库存不足返回 Internal：这是业务状态，不是内部故障。
- 对 DeductStock 随意重试：可能重复扣减，需要幂等键或事务保证。
- 商品价格用 double：建议使用 int64 cents。

---

## 六、练习任务

1. 实现 GetProduct 和 DeductStock。
2. 写库存不足测试。
3. 写并发扣减测试，确认不会扣成负数。
4. 思考真实项目里如何防止重复扣库存。

---

## 七、完成标准

- ProductService 可独立启动。
- 并发扣减安全。
- 错误码表达准确。

---

## 八、完整操作步骤：实现商品查询和库存扣减

ProductService 是订单创建链路中的第二个下游。它比 UserService 多一个关键点：库存扣减必须考虑并发安全。学习项目可以用内存 map，但不能忽略锁。

建议按下面顺序实现：

1. 确认 `product.proto` 已生成 Go 代码。
2. 创建 `internal/app/product/service.go`。
3. 初始化几条商品数据。
4. 实现 `GetProduct`。
5. 实现 `DeductStock`，扣减前先校验数量和库存。
6. 为库存不足返回 `FailedPrecondition`。
7. 写并发测试，确认库存不会被扣成负数。

创建目录：

```powershell
mkdir .\internal\app\product -Force
New-Item -ItemType File .\internal\app\product\service.go -Force
```

---

## 九、完整代码：带锁的库存扣减

`internal/app/product/service.go`：

```go
package product

import (
	"context"
	"sync"

	productv1 "example.com/grpc-shop/gen/product/v1"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
)

type Service struct {
	productv1.UnimplementedProductServiceServer

	mu       sync.Mutex
	products map[int64]*productv1.Product
}

func NewService() *Service {
	return &Service{
		products: map[int64]*productv1.Product{
			1: {Id: 1, Name: "Go gRPC Course", PriceCents: 9900, Stock: 100},
			2: {Id: 2, Name: "Backend Notebook", PriceCents: 3900, Stock: 50},
		},
	}
}

func (s *Service) GetProduct(ctx context.Context, req *productv1.GetProductRequest) (*productv1.GetProductResponse, error) {
	if req.GetId() <= 0 {
		return nil, status.Error(codes.InvalidArgument, "id must be positive")
	}

	s.mu.Lock()
	defer s.mu.Unlock()

	product, ok := s.products[req.GetId()]
	if !ok {
		return nil, status.Error(codes.NotFound, "product not found")
	}

	return &productv1.GetProductResponse{Product: cloneProduct(product)}, nil
}

func (s *Service) DeductStock(ctx context.Context, req *productv1.DeductStockRequest) (*productv1.DeductStockResponse, error) {
	if req.GetProductId() <= 0 {
		return nil, status.Error(codes.InvalidArgument, "product_id must be positive")
	}
	if req.GetQuantity() <= 0 {
		return nil, status.Error(codes.InvalidArgument, "quantity must be positive")
	}

	s.mu.Lock()
	defer s.mu.Unlock()

	product, ok := s.products[req.GetProductId()]
	if !ok {
		return nil, status.Error(codes.NotFound, "product not found")
	}
	if product.Stock < req.GetQuantity() {
		return nil, status.Error(codes.FailedPrecondition, "stock is not enough")
	}

	product.Stock -= req.GetQuantity()
	return &productv1.DeductStockResponse{Product: cloneProduct(product)}, nil
}

func cloneProduct(p *productv1.Product) *productv1.Product {
	if p == nil {
		return nil
	}
	return &productv1.Product{
		Id:         p.Id,
		Name:       p.Name,
		PriceCents: p.PriceCents,
		Stock:      p.Stock,
	}
}
```

注册服务：

```go
grpcServer := server.NewGRPCServer()
productv1.RegisterProductServiceServer(grpcServer, product.NewService())
```

---

## 十、运行命令与预期输出

启动服务：

```powershell
go run .\cmd\product-service
```

预期输出：

```text
product-service listening on :50052
```

查询商品：

```powershell
grpcurl -plaintext -d '{\"id\":1}' localhost:50052 product.v1.ProductService/GetProduct
```

预期输出：

```json
{
  "product": {
    "id": "1",
    "name": "Go gRPC Course",
    "priceCents": "9900",
    "stock": 100
  }
}
```

扣减库存：

```powershell
grpcurl -plaintext -d '{\"productId\":1,\"quantity\":2,\"requestId\":\"demo-001\"}' localhost:50052 product.v1.ProductService/DeductStock
```

预期输出里的 `stock` 应该减少：

```json
{
  "product": {
    "id": "1",
    "stock": 98
  }
}
```

---

## 十一、常见错误排查

- 库存扣成负数：扣减逻辑没有放在同一把锁内，或者先读后写之间发生并发竞争。
- 库存不足返回 `Internal`：库存不足是业务状态，应使用 `FailedPrecondition`。
- 金额字段变成小数误差：价格使用 `int64 price_cents`，不要用 float。
- 重试导致重复扣库存：学习项目先记录风险，真实项目需要幂等表、事务或库存流水。
- 查询商品时 stock 不变化：可能 clone 时返回旧对象，或者扣减请求没有打到同一个服务进程。

---

## 十二、教程闭环检查

本篇的完整操作步骤是：创建 product app、实现查询、实现扣库存、注册服务、启动服务、用 grpcurl 验证查询和扣减。完整代码包含 ProductService、锁、库存校验、错误码和 clone。运行命令包含 `go run` 与两条 `grpcurl`。预期输出覆盖商品查询和库存减少。常见错误排查覆盖并发扣减、错误码、金额类型、重复扣减和服务进程。练习任务是写并发扣减测试，并验证 100 个并发请求不会把库存扣成负数。完成标准是：ProductService 可以被 OrderService 安全调用，库存不足能被准确表达。
