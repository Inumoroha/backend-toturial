# 02 订单下单事务案例

本节目标：围绕「order transaction case」建立清晰的 GORM 学习目标，理解它解决的具体后端问题，能够写出可运行示例，并能通过数据库结果或 SQL 日志验证自己的代码是否正确。

本节用订单系统练习事务。下单是后端工程师必须理解的经典场景，因为它同时涉及多表写入、库存扣减和失败回滚。

## 1. 业务流程

用户下单时：

```text
1. 校验商品是否存在
2. 校验库存是否足够
3. 创建订单
4. 创建订单明细
5. 扣减库存
6. 任一步失败则回滚
```

## 2. 模型设计

### Product

```go
type Product struct {
    ID        uint
    Name      string
    Price     int64 `gorm:"not null"`
    Stock     int   `gorm:"not null"`
    CreatedAt time.Time
    UpdatedAt time.Time
}
```

金额用 `int64` 表示“分”，不要用 `float64`。

### Order

```go
type Order struct {
    ID          uint
    UserID      uint   `gorm:"not null;index"`
    OrderNo     string `gorm:"size:64;not null;uniqueIndex"`
    TotalAmount int64  `gorm:"not null"`
    Status      string `gorm:"size:32;not null;index"`
    Items       []OrderItem
    CreatedAt   time.Time
    UpdatedAt   time.Time
}
```

### OrderItem

```go
type OrderItem struct {
    ID        uint
    OrderID   uint  `gorm:"not null;index"`
    ProductID uint  `gorm:"not null;index"`
    Price     int64 `gorm:"not null"`
    Quantity  int   `gorm:"not null"`
    CreatedAt time.Time
}
```

## 3. 下单请求结构

```go
type CreateOrderItemInput struct {
    ProductID uint
    Quantity  int
}

type CreateOrderInput struct {
    UserID uint
    Items  []CreateOrderItemInput
}
```

## 4. 事务实现

```go
func CreateOrder(db *gorm.DB, input CreateOrderInput) error {
    if len(input.Items) == 0 {
        return fmt.Errorf("order items required")
    }

    return db.Transaction(func(tx *gorm.DB) error {
        var total int64
        var orderItems []OrderItem

        for _, item := range input.Items {
            if item.Quantity <= 0 {
                return fmt.Errorf("invalid quantity")
            }

            var product Product
            if err := tx.First(&product, item.ProductID).Error; err != nil {
                return err
            }

            if product.Stock < item.Quantity {
                return fmt.Errorf("product %d stock not enough", product.ID)
            }

            total += product.Price * int64(item.Quantity)
            orderItems = append(orderItems, OrderItem{
                ProductID: product.ID,
                Price:     product.Price,
                Quantity:  item.Quantity,
            })
        }

        order := Order{
            UserID:      input.UserID,
            OrderNo:     generateOrderNo(),
            TotalAmount: total,
            Status:      "pending",
        }

        if err := tx.Create(&order).Error; err != nil {
            return err
        }

        for i := range orderItems {
            orderItems[i].OrderID = order.ID
        }

        if err := tx.Create(&orderItems).Error; err != nil {
            return err
        }

        for _, item := range input.Items {
            result := tx.Model(&Product{}).
                Where("id = ? AND stock >= ?", item.ProductID, item.Quantity).
                Update("stock", gorm.Expr("stock - ?", item.Quantity))

            if result.Error != nil {
                return result.Error
            }
            if result.RowsAffected == 0 {
                return fmt.Errorf("product %d stock not enough", item.ProductID)
            }
        }

        return nil
    })
}
```

订单号函数可以先简单写：

```go
func generateOrderNo() string {
    return fmt.Sprintf("NO%d", time.Now().UnixNano())
}
```

## 5. 为什么扣库存要带条件

```go
Where("id = ? AND stock >= ?", item.ProductID, item.Quantity)
```

这能避免库存不足时仍然扣成负数。

即使前面已经查过库存，扣减时也要再次带条件，因为并发请求可能在你查询之后修改库存。

## 6. 为什么要检查 RowsAffected

如果库存不足，更新语句不会命中任何行：

```go
if result.RowsAffected == 0 {
    return fmt.Errorf("stock not enough")
}
```

返回错误后，事务会回滚，订单和订单明细都不会保留。

## 本节练习

- [ ] 创建 `Product`、`Order`、`OrderItem` 模型。
- [ ] 插入 3 个商品。
- [ ] 实现单商品下单。
- [ ] 实现多商品下单。
- [ ] 库存不足时返回错误。
- [ ] 确认库存不足时订单没有创建成功。
- [ ] 确认订单明细失败时订单会回滚。

## 常见坑

- 只查库存，不在扣库存 SQL 里带库存条件。
- 创建订单成功后，创建明细失败却没有回滚。
- 金额用 `float64`。
- 事务内部混用了 `db` 和 `tx`。
---

## 本节达标标准

学完本节后，你应该能够做到：

- 说清本节主题解决的具体后端开发问题。
- 写出本节涉及的核心 GORM 代码或 SQL 示例。
- 通过 SQL 日志、数据库客户端或测试验证结果是否正确。
- 识别本节列出的常见错误，并知道如何排查。
- 把本节知识放回博客系统或订单系统的真实场景中使用。
