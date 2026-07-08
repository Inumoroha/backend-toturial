# 05 订单系统逐步落地手册

本节目标：完成一个最小订单系统，把 GORM 事务、库存扣减、状态流转和幂等思考串起来。

---

## 一、项目目标

订单系统第一版包含：

```text
商品列表
商品详情
创建订单
订单列表
订单详情
取消订单
模拟支付
```

核心不是接口数量，而是数据一致性。

---

## 二、模型设计

### Product

```go
type Product struct {
    ID        uint
    Name      string `gorm:"size:128;not null"`
    Price     int64  `gorm:"not null"`
    Stock     int    `gorm:"not null"`
    CreatedAt time.Time
    UpdatedAt time.Time
}
```

金额使用分：

```text
9900 表示 99 元
```

不要使用 `float64` 存金额。

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

### Payment

```go
type Payment struct {
    ID        uint
    OrderID   uint   `gorm:"not null;index"`
    PayNo     string `gorm:"size:64;not null;uniqueIndex"`
    Amount    int64  `gorm:"not null"`
    Status    string `gorm:"size:32;not null;index"`
    CreatedAt time.Time
    UpdatedAt time.Time
}
```

---

## 三、创建订单接口

接口：

```http
POST /api/orders
Authorization: Bearer <token>
```

请求：

```json
{
  "items": [
    {"product_id": 1, "quantity": 2},
    {"product_id": 2, "quantity": 1}
  ]
}
```

响应：

```json
{
  "id": 1,
  "order_no": "NO1783240000000",
  "total_amount": 19900,
  "status": "pending"
}
```

---

## 四、下单事务流程

```text
1. 校验订单项不为空
2. 遍历商品
3. 校验商品存在
4. 校验库存充足
5. 计算订单总价
6. 创建订单
7. 创建订单明细
8. 扣减库存
9. 提交事务
```

任何一步失败，都要回滚。

---

## 五、扣库存关键 SQL

GORM：

```go
result := tx.Model(&model.Product{}).
    Where("id = ? AND stock >= ?", item.ProductID, item.Quantity).
    Update("stock", gorm.Expr("stock - ?", item.Quantity))
```

对应 SQL：

```sql
UPDATE products
SET stock = stock - ?
WHERE id = ? AND stock >= ?;
```

这个写法的关键是：

```text
库存判断在 UPDATE 里完成
```

不要只在应用层查库存后再扣减。

---

## 六、订单状态流转

建议状态：

```text
pending    待支付
paid       已支付
cancelled  已取消
closed     已关闭
```

允许：

```text
pending -> paid
pending -> cancelled
paid -> closed
```

不允许：

```text
paid -> cancelled
cancelled -> paid
```

状态校验应该写在 Service 层。

---

## 七、取消订单

接口：

```http
POST /api/orders/:id/cancel
```

流程：

```text
1. 查询订单和明细
2. 确认订单属于当前用户
3. 确认订单状态是 pending
4. 修改订单状态为 cancelled
5. 恢复库存
6. 使用事务
```

恢复库存：

```go
err := tx.Model(&model.Product{}).
    Where("id = ?", item.ProductID).
    Update("stock", gorm.Expr("stock + ?", item.Quantity)).Error
```

---

## 八、模拟支付

接口：

```http
POST /api/orders/:id/pay
```

流程：

```text
1. 查询订单
2. 确认订单属于当前用户
3. 确认订单状态是 pending
4. 创建 payment 记录
5. 修改订单状态为 paid
6. 使用事务
```

更新订单时建议带状态条件：

```go
result := tx.Model(&model.Order{}).
    Where("id = ? AND status = ?", order.ID, "pending").
    Update("status", "paid")
```

如果 `RowsAffected == 0`，说明订单状态已经被别人改过，不能继续支付。

---

## 九、幂等性检查

重复请求是后端必须考虑的问题。

### 重复下单

普通创建订单可以允许重复，因为用户可能真的买两次。如果前端要求防重复提交，需要引入请求幂等 key。

### 重复支付

必须防止重复支付。

基本防线：

- 订单状态必须是 `pending`。
- 支付单号唯一。
- 更新订单时带 `status = pending`。
- 检查 `RowsAffected`。

---

## 十、测试场景

必须测试：

- 库存充足，下单成功。
- 库存不足，下单失败且没有订单。
- 下单成功后库存减少。
- 取消订单后库存恢复。
- 已支付订单不能取消。
- 重复支付只能成功一次。
- 支付失败时订单状态不变。

---

## 十一、常见错误

### 1. 金额用 float64

金额要用整数分，或者数据库 decimal。不要用浮点数。

### 2. 扣库存不检查 RowsAffected

库存不足时，更新可能影响 0 行。必须检查。

### 3. 取消订单不恢复库存

是否恢复库存取决于业务，但如果下单时已经扣库存，取消时通常要恢复。

### 4. 支付没有状态条件

如果不带 `status = pending`，重复支付或并发支付可能造成错误状态。

---

## 十二、最终验收清单

- [ ] 商品能创建测试数据。
- [ ] 下单使用事务。
- [ ] 库存不足能回滚。
- [ ] 订单明细和订单总价正确。
- [ ] 取消订单能恢复库存。
- [ ] 已支付订单不能取消。
- [ ] 支付订单会创建支付记录。
- [ ] 重复支付不会重复成功。
- [ ] 关键事务有测试覆盖。
---

## 本节达标标准

学完本节后，你应该能够做到：

- 说清本节主题解决的具体后端开发问题。
- 写出本节涉及的核心 GORM 代码或 SQL 示例。
- 通过 SQL 日志、数据库客户端或测试验证结果是否正确。
- 识别本节列出的常见错误，并知道如何排查。
- 把本节知识放回博客系统或订单系统的真实场景中使用。
