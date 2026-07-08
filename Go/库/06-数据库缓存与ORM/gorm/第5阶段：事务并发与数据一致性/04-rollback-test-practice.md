# 04 事务回滚测试实战

本节目标：不只是会写 `db.Transaction`，还要能证明它真的会回滚。你会写一个下单失败场景，确认订单、订单明细和库存不会留下错误状态。

---

## 一、为什么要测试事务

事务代码最怕“看起来用了事务，实际上没保护住业务”。

常见问题：

- 事务内部误用了外部 `db`。
- 某一步失败后没有 `return err`。
- 创建订单成功，但创建明细失败没有回滚。
- 库存不足时订单仍然留下。
- 更新库存没有检查 `RowsAffected`。

所以事务必须写测试，尤其是失败路径。

---

## 二、测试目标

我们要验证：

```text
库存不足时：
1. CreateOrder 返回错误
2. orders 表没有新增订单
3. order_items 表没有新增明细
4. products.stock 没有被扣减
```

---

## 三、测试环境选择

有两种方式：

### 方式一：使用 MySQL 测试库

最接近真实环境。建议创建单独数据库：

```sql
CREATE DATABASE gorm_study_test DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### 方式二：使用 SQLite 内存库

速度快，但和 MySQL 行为不完全一致。事务基础测试可以用，涉及锁和 SQL 方言时不要完全依赖它。

本节更推荐 MySQL 测试库。

---

## 四、测试辅助函数

创建：

```text
internal/testutil/db.go
```

示例：

```go
package testutil

import (
    "testing"

    "gorm.io/driver/mysql"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"
)

func OpenTestDB(t *testing.T) *gorm.DB {
    t.Helper()

    dsn := "root:root123456@tcp(127.0.0.1:3306)/gorm_study_test?charset=utf8mb4&parseTime=True&loc=Local"
    db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{
        Logger: logger.Default.LogMode(logger.Silent),
    })
    if err != nil {
        t.Fatal(err)
    }

    return db
}
```

---

## 五、清理表数据

测试前要清理旧数据：

```go
func resetTables(t *testing.T, db *gorm.DB) {
    t.Helper()

    if err := db.Exec("DELETE FROM order_items").Error; err != nil {
        t.Fatal(err)
    }
    if err := db.Exec("DELETE FROM orders").Error; err != nil {
        t.Fatal(err)
    }
    if err := db.Exec("DELETE FROM products").Error; err != nil {
        t.Fatal(err)
    }
}
```

删除顺序很重要：先删子表，再删父表。

---

## 六、失败回滚测试

```go
func TestCreateOrder_RollbackWhenStockNotEnough(t *testing.T) {
    db := testutil.OpenTestDB(t)

    if err := db.AutoMigrate(&Product{}, &Order{}, &OrderItem{}); err != nil {
        t.Fatal(err)
    }
    resetTables(t, db)

    product := Product{
        Name:  "GORM Book",
        Price: 9900,
        Stock: 1,
    }
    if err := db.Create(&product).Error; err != nil {
        t.Fatal(err)
    }

    err := CreateOrder(db, CreateOrderInput{
        UserID: 1,
        Items: []CreateOrderItemInput{
            {
                ProductID: product.ID,
                Quantity:  2,
            },
        },
    })
    if err == nil {
        t.Fatal("expected stock not enough error")
    }

    var orderCount int64
    if err := db.Model(&Order{}).Count(&orderCount).Error; err != nil {
        t.Fatal(err)
    }
    if orderCount != 0 {
        t.Fatalf("expected 0 orders, got %d", orderCount)
    }

    var itemCount int64
    if err := db.Model(&OrderItem{}).Count(&itemCount).Error; err != nil {
        t.Fatal(err)
    }
    if itemCount != 0 {
        t.Fatalf("expected 0 order items, got %d", itemCount)
    }

    var after Product
    if err := db.First(&after, product.ID).Error; err != nil {
        t.Fatal(err)
    }
    if after.Stock != 1 {
        t.Fatalf("expected stock 1, got %d", after.Stock)
    }
}
```

---

## 七、成功路径测试

```go
func TestCreateOrder_Success(t *testing.T) {
    db := testutil.OpenTestDB(t)

    if err := db.AutoMigrate(&Product{}, &Order{}, &OrderItem{}); err != nil {
        t.Fatal(err)
    }
    resetTables(t, db)

    product := Product{
        Name:  "GORM Book",
        Price: 9900,
        Stock: 5,
    }
    if err := db.Create(&product).Error; err != nil {
        t.Fatal(err)
    }

    err := CreateOrder(db, CreateOrderInput{
        UserID: 1,
        Items: []CreateOrderItemInput{
            {
                ProductID: product.ID,
                Quantity:  2,
            },
        },
    })
    if err != nil {
        t.Fatal(err)
    }

    var orderCount int64
    db.Model(&Order{}).Count(&orderCount)
    if orderCount != 1 {
        t.Fatalf("expected 1 order, got %d", orderCount)
    }

    var after Product
    db.First(&after, product.ID)
    if after.Stock != 3 {
        t.Fatalf("expected stock 3, got %d", after.Stock)
    }
}
```

---

## 八、故意写错验证测试价值

把事务代码中的：

```go
return fmt.Errorf("product %d stock not enough", product.ID)
```

临时改成：

```go
fmt.Errorf("product %d stock not enough", product.ID)
return nil
```

再运行测试，你应该看到失败。

这能帮助你理解：事务闭包里必须把错误 return 出去，否则 GORM 会认为事务可以提交。

---

## 九、运行测试

```powershell
go test ./...
```

如果只跑订单相关测试：

```powershell
go test ./internal/order -run TestCreateOrder
```

---

## 十、常见错误

### 1. 测试库和开发库混用

不要让测试清空开发库。测试 DSN 应该明确指向：

```text
gorm_study_test
```

### 2. 清理表顺序错误

有外键时，应先清理子表，再清理父表。

### 3. 事务内部使用外部 db

错误：

```go
db.Create(&orderItem)
```

正确：

```go
tx.Create(&orderItem)
```

### 4. 只测成功不测失败

事务最重要的是失败回滚。只测成功路径是不够的。

---

## 十一、本节达标标准

学完本节后，你应该能够做到：

- 为事务业务创建测试数据库。
- 写出事务成功路径测试。
- 写出事务失败回滚测试。
- 验证订单、订单明细、库存的一致性。
- 解释为什么事务内部必须使用 `tx`。
- 解释为什么错误必须从事务闭包中 return。
