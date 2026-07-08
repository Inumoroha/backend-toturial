# 02 订单系统实战

本节目标：围绕「order api project」建立清晰的 GORM 学习目标，理解它解决的具体后端问题，能够写出可运行示例，并能通过数据库结果或 SQL 日志验证自己的代码是否正确。

订单系统用于练习事务、库存扣减、并发一致性和复杂业务流程。它比博客系统更接近真实商业后端。

## 1. 需求范围

第一版完成：

- 商品列表
- 商品详情
- 创建订单
- 查询订单列表
- 查询订单详情
- 取消订单
- 支付订单模拟

## 2. 模型清单

- `User`
- `Product`
- `Order`
- `OrderItem`
- `Payment`

## 3. 订单状态

建议先定义这些状态：

```text
pending    待支付
paid       已支付
cancelled  已取消
closed     已关闭
```

状态流转：

```text
pending -> paid
pending -> cancelled
paid -> closed
```

不要允许随意从任何状态跳到任何状态。

## 4. 推荐接口

### 商品

```text
GET /api/products
GET /api/products/:id
```

### 订单

```text
POST /api/orders
GET  /api/orders
GET  /api/orders/:id
POST /api/orders/:id/cancel
POST /api/orders/:id/pay
```

## 5. 下单流程

```text
1. 接收商品 ID 和数量
2. 校验商品存在
3. 校验库存充足
4. 创建订单
5. 创建订单明细
6. 扣减库存
7. 提交事务
```

任何一步失败，都回滚。

## 6. 下单 Service 结构

```go
func (s *OrderService) CreateOrder(ctx context.Context, input CreateOrderInput) (*model.Order, error) {
    var order *model.Order

    err := s.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
        // 1. 查询商品
        // 2. 计算金额
        // 3. 创建订单
        // 4. 创建订单明细
        // 5. 扣减库存
        // 6. 把创建结果赋值给 order
        return nil
    })
    if err != nil {
        return nil, err
    }

    return order, nil
}
```

事务应该在 Service 层组织。

## 7. 扣库存写法

```go
result := tx.Model(&model.Product{}).
    Where("id = ? AND stock >= ?", productID, quantity).
    Update("stock", gorm.Expr("stock - ?", quantity))

if result.Error != nil {
    return result.Error
}
if result.RowsAffected == 0 {
    return ErrStockNotEnough
}
```

这比“先查库存再直接更新”更安全。

## 8. 取消订单

取消订单要考虑是否恢复库存。

流程：

```text
1. 查询订单和明细
2. 确认订单是 pending
3. 修改订单状态为 cancelled
4. 恢复商品库存
5. 使用事务
```

如果订单已经 paid，就不能取消。

## 9. 支付订单模拟

支付流程：

```text
1. 查询订单
2. 确认订单是 pending
3. 创建支付记录
4. 修改订单状态为 paid
5. 使用事务
```

真实支付还会涉及第三方回调、幂等、防重复通知，这里先做模拟。

## 10. 幂等性思考

如果用户连续点击两次支付，会发生什么？

你需要防止：

- 重复创建支付记录。
- 重复修改订单状态。
- 重复扣库存。

基础做法：

- 状态机校验。
- 唯一业务单号。
- 更新时带状态条件。
- 检查 `RowsAffected`。

## 11. 验收清单

- [ ] 商品库存不足时不能下单。
- [ ] 下单失败时不会创建半截订单。
- [ ] 下单成功后库存正确减少。
- [ ] 订单详情能看到订单明细。
- [ ] 取消订单能恢复库存。
- [ ] 已支付订单不能取消。
- [ ] 支付订单能创建支付记录。
- [ ] 重复支付不会产生重复结果。

## 12. 加分项

- [ ] 乐观锁扣库存。
- [ ] 悲观锁扣库存。
- [ ] 订单超时自动关闭。
- [ ] 支付回调幂等处理。
- [ ] 订单号生成器。
- [ ] 集成测试覆盖事务回滚。
---

## 本节达标标准

学完本节后，你应该能够做到：

- 说清本节主题解决的具体后端开发问题。
- 写出本节涉及的核心 GORM 代码或 SQL 示例。
- 通过 SQL 日志、数据库客户端或测试验证结果是否正确。
- 识别本节列出的常见错误，并知道如何排查。
- 把本节知识放回博客系统或订单系统的真实场景中使用。
