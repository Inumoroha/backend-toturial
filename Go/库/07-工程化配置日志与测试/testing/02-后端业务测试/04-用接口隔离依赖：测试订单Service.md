# 4. 用接口隔离依赖：测试订单 Service

本节目标：学完后你能用接口和 fake 对象测试 service 层业务规则，而不依赖真实数据库或第三方服务。

简短引入：后端项目里 service 往往依赖 repository、库存服务、支付服务、消息队列。如果单元测试必须启动这些依赖，测试会慢、不稳定，也很难覆盖异常分支。常见做法是让 service 依赖接口，测试时传入 fake 实现。

## 一、为什么需要它

创建订单时通常要做这些事：

- 检查商品是否存在。
- 检查库存是否足够。
- 计算金额。
- 写入订单。
- 可能还要发消息或调用支付。

如果测试每次都连接真实数据库和库存系统，失败原因会变复杂：到底是业务逻辑错了，还是数据库没启动，还是网络抖动？

```text
单元测试应该优先验证本层业务逻辑，外部依赖用接口隔离。
```

## 二、基本用法

创建一个新目录实验：

Windows PowerShell：

```powershell
mkdir orderdemo
cd orderdemo
go mod init example.com/orderdemo
```

Linux/macOS：

```bash
mkdir orderdemo
cd orderdemo
go mod init example.com/orderdemo
```

新建 `service.go`：

```go
package orderdemo

import (
	"context"
	"errors"
)

type Order struct {
	UserID     int64
	SKU        string
	Quantity   int
	TotalCents int
}

type Product struct {
	SKU        string
	PriceCents int
	Stock      int
}

type ProductRepo interface {
	FindBySKU(ctx context.Context, sku string) (Product, error)
}

type OrderRepo interface {
	Create(ctx context.Context, order Order) error
}

type OrderService struct {
	products ProductRepo
	orders   OrderRepo
}

func NewOrderService(products ProductRepo, orders OrderRepo) *OrderService {
	return &OrderService{products: products, orders: orders}
}

func (s *OrderService) CreateOrder(ctx context.Context, userID int64, sku string, quantity int) error {
	if userID <= 0 {
		return errors.New("user id is required")
	}
	if sku == "" {
		return errors.New("sku is required")
	}
	if quantity <= 0 {
		return errors.New("quantity must be positive")
	}

	product, err := s.products.FindBySKU(ctx, sku)
	if err != nil {
		return err
	}
	if product.Stock < quantity {
		return errors.New("stock is not enough")
	}

	order := Order{
		UserID:     userID,
		SKU:        sku,
		Quantity:   quantity,
		TotalCents: product.PriceCents * quantity,
	}
	return s.orders.Create(ctx, order)
}
```

## 三、关键参数/语法/代码结构

`ProductRepo` 和 `OrderRepo` 是 service 需要的能力，不是具体数据库实现。真实项目中，它们可以由 MySQL、PostgreSQL 或内存实现提供。

`OrderService` 通过构造函数接收依赖。这样测试时可以传 fake，生产代码传真实 repository。

`context.Context` 是后端项目里的常见参数，用于超时、取消、链路信息传递。测试里可以用 `context.Background()`。

接口不要过大。service 只需要 `FindBySKU`，接口里就不要放一堆暂时用不到的方法。

## 四、真实后端场景示例

新建 `service_test.go`：

```go
package orderdemo

import (
	"context"
	"errors"
	"testing"
)

type fakeProductRepo struct {
	product Product
	err     error
}

func (f fakeProductRepo) FindBySKU(ctx context.Context, sku string) (Product, error) {
	return f.product, f.err
}

type fakeOrderRepo struct {
	created Order
	err     error
}

func (f *fakeOrderRepo) Create(ctx context.Context, order Order) error {
	f.created = order
	return f.err
}

func TestOrderService_CreateOrder(t *testing.T) {
	tests := []struct {
		name    string
		product Product
		repoErr error
		userID  int64
		sku     string
		qty     int
		want    Order
		wantErr bool
	}{
		{
			name:    "create order",
			product: Product{SKU: "book-1", PriceCents: 3000, Stock: 10},
			userID:  1,
			sku:     "book-1",
			qty:     2,
			want:    Order{UserID: 1, SKU: "book-1", Quantity: 2, TotalCents: 6000},
		},
		{
			name:    "stock not enough",
			product: Product{SKU: "book-1", PriceCents: 3000, Stock: 1},
			userID:  1,
			sku:     "book-1",
			qty:     2,
			wantErr: true,
		},
		{
			name:    "product repo failed",
			repoErr: errors.New("db timeout"),
			userID:  1,
			sku:     "book-1",
			qty:     1,
			wantErr: true,
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			orderRepo := &fakeOrderRepo{}
			svc := NewOrderService(
				fakeProductRepo{product: tt.product, err: tt.repoErr},
				orderRepo,
			)

			err := svc.CreateOrder(context.Background(), tt.userID, tt.sku, tt.qty)
			if (err != nil) != tt.wantErr {
				t.Fatalf("error = %v, wantErr %v", err, tt.wantErr)
			}
			if !tt.wantErr && orderRepo.created != tt.want {
				t.Fatalf("created order = %+v, want %+v", orderRepo.created, tt.want)
			}
		})
	}
}
```

运行：

```powershell
go test ./...
```

Linux/macOS：

```bash
go test ./...
```

这个测试不需要数据库，却验证了订单 service 的核心规则：参数校验、库存校验、金额计算、写入订单。

## 五、注意点

fake 不是 mock 框架的同义词。初学阶段，手写 fake 往往更清楚，也更容易理解业务依赖。

service 层测试不要关心 SQL 是否正确。SQL 是 repository 层的职责。service 只关心“查到商品后如何处理业务规则”。

接口应该由使用方定义。也就是说，`OrderService` 需要什么能力，就在 service 附近定义什么接口。不要为了复用而写一个巨大的 repository 接口。

错误处理要保守。真实项目中不要把底层数据库错误原样返回给用户，但 service 测试可以验证错误会被正确中断或包装。

## 六、常见误区

- service 里直接 new 数据库连接：测试时无法替换依赖，代码也难维护。
- 为了测试引入复杂 mock 框架：简单依赖手写 fake 更直观。
- 测试只看是否报错，不检查写入订单内容：金额算错也可能漏掉。
- fake 写得比真实代码还复杂：fake 只服务当前测试，不要模拟整个数据库。
- 接口定义过大：实现成本高，测试准备也会变麻烦。

## 七、本节达标标准

- 能为 service 依赖定义小接口。
- 能手写 fake repository。
- 能测试 service 的成功路径和失败路径。
- 能区分 service 测试和 repository 测试的边界。
- 能解释为什么单元测试不应该连接真实数据库。

