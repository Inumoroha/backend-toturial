# 5. OrderService 创建订单与 Outbox

本节目标：实现创建订单接口，并在同一数据库事务中写入 outbox 事件。

---

## 一、接口

```http
POST /orders
Content-Type: application/json
```

```json
{
  "user_id": "user_88",
  "items": [
    {"sku_id": "sku_1", "quantity": 2, "price": 9900}
  ]
}
```

---

## 二、事务流程

```text
begin tx
insert orders
insert order_items
build order.created event
insert outbox_events
commit
```

---

## 三、伪代码

```go
func (s *Service) CreateOrder(ctx context.Context, req CreateOrderRequest) error {
    tx := s.db.BeginTx(ctx)
    defer tx.Rollback()

    orderID := newOrderID()
    eventID := "evt_order_created_" + orderID

    insertOrder(tx, orderID, req)
    insertItems(tx, orderID, req.Items)
    insertOutbox(tx, buildOutbox(eventID, orderID, req))

    return tx.Commit()
}
```

---

## 四、为什么不用直接发 Kafka

直接发会遇到：

```text
数据库成功，Kafka 失败
```

Outbox 可以重试。

---

## 五、验收

- 调用接口后 orders 有记录。
- order_items 有记录。
- outbox_events 有 PENDING 事件。
- Kafka 此时可以还没有消息，因为 worker 负责发送。

---

## 六、请求结构

`internal/order/model.go`：

```go
package order

type CreateOrderRequest struct {
    UserID string            `json:"user_id"`
    Items  []CreateOrderItem `json:"items"`
}

type CreateOrderItem struct {
    SkuID    string `json:"sku_id"`
    Quantity int    `json:"quantity"`
    Price    int64  `json:"price"`
}

type CreateOrderResponse struct {
    OrderID string `json:"order_id"`
    Status  string `json:"status"`
}
```

---

## 七、Repository 结构

`internal/order/repository.go`：

```go
package order

import "github.com/jackc/pgx/v5/pgxpool"

type Repository struct {
    pool *pgxpool.Pool
}

func NewRepository(pool *pgxpool.Pool) *Repository {
    return &Repository{pool: pool}
}
```

实际插入函数可以接收事务：

```go
func (r *Repository) CreateOrder(ctx context.Context, tx pgx.Tx, orderID string, req CreateOrderRequest, total int64) error {
    _, err := tx.Exec(ctx, `
        INSERT INTO orders(id, user_id, status, total_amount)
        VALUES ($1, $2, $3, $4)
    `, orderID, req.UserID, "CREATED", total)
    return err
}
```

---

## 八、计算金额

不要相信客户端传 total。

```go
func calculateTotal(items []CreateOrderItem) int64 {
    var total int64
    for _, item := range items {
        total += int64(item.Quantity) * item.Price
    }
    return total
}
```

校验：

```go
if req.UserID == "" {
    return error
}
if len(req.Items) == 0 {
    return error
}
```

---

## 九、Outbox 事件构造

```go
eventID := "evt_order_created_" + orderID

event := OrderCreatedEvent{
    EventID:    eventID,
    EventType:  "order.created",
    Version:    1,
    OccurredAt: time.Now().UTC(),
    Data: OrderCreatedData{
        OrderID:     orderID,
        UserID:      req.UserID,
        Items:       req.Items,
        TotalAmount: total,
    },
}
```

写 outbox：

```go
payload, _ := json.Marshal(event)

_, err := tx.Exec(ctx, `
    INSERT INTO outbox_events(id, aggregate_type, aggregate_id, event_type, topic, message_key, payload, status)
    VALUES ($1, $2, $3, $4, $5, $6, $7, 'PENDING')
`, eventID, "order", orderID, "order.created", "order.created", orderID, payload)
```

---

## 十、Handler

`internal/order/handler.go`：

```go
func (h *Handler) createOrder(w http.ResponseWriter, r *http.Request) {
    var req CreateOrderRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        writeJSON(w, http.StatusBadRequest, map[string]string{"error": "invalid json"})
        return
    }

    resp, err := h.service.CreateOrder(r.Context(), req)
    if err != nil {
        writeJSON(w, http.StatusInternalServerError, map[string]string{"error": "internal error"})
        return
    }

    writeJSON(w, http.StatusCreated, resp)
}
```

---

## 十一、为什么此时 Kafka 可以没有消息

创建订单接口只负责：

```text
写业务数据
写 outbox
```

Kafka 发布由 outbox-worker 完成。

这样即使 Kafka 临时不可用，创建订单事务仍然可以成功，事件稍后发送。

---

## 十二、本节练习

1. 实现请求结构。
2. 实现 `CreateOrder` service。
3. 在同一事务里写 orders、order_items、outbox_events。
4. 调用接口后查询 outbox。
5. 验证 Kafka 中暂时没有消息也没关系。

---

## 十三、Service 完整骨架

推荐让 `Service` 持有数据库连接和 repository：

```go
type Service struct {
    pool *pgxpool.Pool
    repo *Repository
}

func NewService(pool *pgxpool.Pool, repo *Repository) *Service {
    return &Service{pool: pool, repo: repo}
}
```

`CreateOrder` 的真实结构应该接近下面这样：

```go
func (s *Service) CreateOrder(ctx context.Context, req CreateOrderRequest) (CreateOrderResponse, error) {
    if err := req.Validate(); err != nil {
        return CreateOrderResponse{}, err
    }

    orderID := newOrderID()
    total := calculateTotal(req.Items)

    tx, err := s.pool.Begin(ctx)
    if err != nil {
        return CreateOrderResponse{}, err
    }
    defer tx.Rollback(ctx)

    if err := s.repo.InsertOrder(ctx, tx, orderID, req.UserID, total); err != nil {
        return CreateOrderResponse{}, err
    }

    if err := s.repo.InsertOrderItems(ctx, tx, orderID, req.Items); err != nil {
        return CreateOrderResponse{}, err
    }

    event, err := BuildOrderCreatedEvent(orderID, req, total)
    if err != nil {
        return CreateOrderResponse{}, err
    }

    if err := s.repo.InsertOutboxEvent(ctx, tx, event); err != nil {
        return CreateOrderResponse{}, err
    }

    if err := tx.Commit(ctx); err != nil {
        return CreateOrderResponse{}, err
    }

    return CreateOrderResponse{OrderID: orderID, Status: "CREATED"}, nil
}
```

注意：`defer tx.Rollback(ctx)` 在 commit 成功后会返回一个可忽略错误，这是常见写法。

---

## 十四、请求校验

不要让非法请求进入数据库事务：

```go
func (r CreateOrderRequest) Validate() error {
    if strings.TrimSpace(r.UserID) == "" {
        return errors.New("user_id is required")
    }
    if len(r.Items) == 0 {
        return errors.New("items is required")
    }
    for _, item := range r.Items {
        if strings.TrimSpace(item.SkuID) == "" {
            return errors.New("sku_id is required")
        }
        if item.Quantity <= 0 {
            return errors.New("quantity must be greater than 0")
        }
        if item.Price < 0 {
            return errors.New("price must be greater than or equal to 0")
        }
    }
    return nil
}
```

校验分两层：

```text
HTTP 层校验 JSON 格式。
Service 层校验业务字段。
数据库层用 NOT NULL/CHECK 做最后保护。
```

---

## 十五、插入订单明细

`order_items` 通常是一笔订单多行：

```go
func (r *Repository) InsertOrderItems(
    ctx context.Context,
    tx pgx.Tx,
    orderID string,
    items []CreateOrderItem,
) error {
    for _, item := range items {
        _, err := tx.Exec(ctx, `
            INSERT INTO order_items(order_id, sku_id, quantity, price)
            VALUES ($1, $2, $3, $4)
        `, orderID, item.SkuID, item.Quantity, item.Price)
        if err != nil {
            return err
        }
    }
    return nil
}
```

学习阶段逐行插入更直观。后续追求性能时，可以再改成 batch。

---

## 十六、Outbox 记录应该长什么样

创建订单后，`outbox_events` 应该出现类似数据：

| 字段 | 示例 |
| --- | --- |
| id | `evt_order_created_order_1001` |
| aggregate_type | `order` |
| aggregate_id | `order_1001` |
| event_type | `order.created` |
| topic | `order.created` |
| message_key | `order_1001` |
| status | `PENDING` |

验证 SQL：

```sql
SELECT id, aggregate_type, aggregate_id, event_type, topic, message_key, status
FROM outbox_events
ORDER BY created_at DESC
LIMIT 1;
```

如果这里没有记录，就不要继续查 Kafka。问题还在 order-service 的事务里。

---

## 十七、curl 验证

启动服务后请求：

```bash
curl -i -X POST http://localhost:8080/orders \
  -H "Content-Type: application/json" \
  -d '{"user_id":"user_88","items":[{"sku_id":"sku_1","quantity":2,"price":9900}]}'
```

预期响应：

```http
HTTP/1.1 201 Created
Content-Type: application/json
```

```json
{
  "order_id": "order_1001",
  "status": "CREATED"
}
```

然后查：

```sql
SELECT status, total_amount
FROM orders
WHERE id = 'order_1001';
```

预期：

```text
status = CREATED
total_amount = 19800
```

---

## 十八、常见错误

### 1. 事务里直接发送 Kafka

数据库事务无法管理 Kafka。事务回滚时，Kafka 消息不会自动撤回。

### 2. 客户端传 total_amount

客户端可以传错，也可以恶意传小。金额必须由服务端根据明细计算。

### 3. outbox 不和订单放在同一事务

如果订单 commit 成功后，单独插 outbox 失败，就又回到“订单存在但事件丢失”的问题。

### 4. event_id 每次重试都重新生成

同一个业务事件应该保持同一个 `event_id`。否则 consumer 无法根据 event_id 判断重复。

---

## 十九、面试表达

可以这样介绍 OrderService：

```text
创建订单时，我不会在 HTTP 请求里直接发 Kafka。
我会在同一个 PostgreSQL 事务里写 orders、order_items 和 outbox_events。
事务提交后，outbox-worker 异步扫描并发送 Kafka。
这样可以避免订单写库成功但消息发送失败导致的事件丢失。
```
