# 6. OutboxWorker 发布 Kafka 事件

本节目标：实现 outbox worker，扫描 outbox_events 并发布 Kafka。

---

## 一、Worker 流程

```text
查询 PENDING
锁定事件
发送 Kafka
成功标记 SENT
失败更新 retry_count
循环
```

---

## 二、查询 SQL

```sql
SELECT id, topic, message_key, payload
FROM outbox_events
WHERE status = 'PENDING'
  AND (next_retry_at IS NULL OR next_retry_at <= now())
ORDER BY created_at
LIMIT 100
FOR UPDATE SKIP LOCKED;
```

---

## 三、伪代码

```go
func (w *Worker) Run(ctx context.Context) {
    for ctx.Err() == nil {
        events := w.repo.LockPending(ctx, 100)
        for _, e := range events {
            err := w.producer.Publish(ctx, e.Topic, kafka.Message{
                Key: []byte(e.MessageKey),
                Value: e.Payload,
            })
            if err != nil {
                w.repo.MarkRetry(ctx, e.ID, err)
                continue
            }
            w.repo.MarkSent(ctx, e.ID)
        }
        time.Sleep(time.Second)
    }
}
```

---

## 四、重复发送

发送 Kafka 成功但标记 SENT 失败，会重复发送。

因此 consumer 必须幂等。

---

## 五、验收

- PENDING 事件能变 SENT。
- Kafka 中能看到 order.created。
- Kafka 不可用时 retry_count 增加。

---

## 六、Repository

`internal/outbox/repository.go`：

```go
type Event struct {
    ID         string
    Topic      string
    MessageKey string
    Payload    []byte
}

type Repository struct {
    pool *pgxpool.Pool
}
```

查询 pending：

```go
func (r *Repository) LockPending(ctx context.Context, limit int) ([]Event, error) {
    rows, err := r.pool.Query(ctx, `
        SELECT id, topic, message_key, payload
        FROM outbox_events
        WHERE status = 'PENDING'
          AND (next_retry_at IS NULL OR next_retry_at <= now())
        ORDER BY created_at
        LIMIT $1
        FOR UPDATE SKIP LOCKED
    `, limit)
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    // 扫描 rows，返回 events
}
```

注意：严格来说 `FOR UPDATE` 需要在事务中使用。本项目可以先简化，进阶时再把查询和状态更新放入事务。

---

## 七、标记 SENT

```go
func (r *Repository) MarkSent(ctx context.Context, id string) error {
    _, err := r.pool.Exec(ctx, `
        UPDATE outbox_events
        SET status = 'SENT', sent_at = now()
        WHERE id = $1
    `, id)
    return err
}
```

---

## 八、标记重试

```go
func (r *Repository) MarkRetry(ctx context.Context, id string) error {
    _, err := r.pool.Exec(ctx, `
        UPDATE outbox_events
        SET retry_count = retry_count + 1,
            next_retry_at = now() + interval '1 minute'
        WHERE id = $1
    `, id)
    return err
}
```

生产中还要记录错误原因。

---

## 九、Worker 结构

```go
type Worker struct {
    repo     *Repository
    producer kafka.Producer
}

func NewWorker(repo *Repository, producer kafka.Producer) *Worker {
    return &Worker{repo: repo, producer: producer}
}
```

---

## 十、Run 方法

```go
func (w *Worker) Run(ctx context.Context) error {
    ticker := time.NewTicker(time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-ticker.C:
            if err := w.runOnce(ctx); err != nil {
                // log error
            }
        }
    }
}
```

---

## 十一、runOnce

```go
func (w *Worker) runOnce(ctx context.Context) error {
    events, err := w.repo.LockPending(ctx, 100)
    if err != nil {
        return err
    }

    for _, event := range events {
        err := w.producer.Publish(ctx, event.Topic, kafka.Message{
            Key:   []byte(event.MessageKey),
            Value: event.Payload,
        })
        if err != nil {
            _ = w.repo.MarkRetry(ctx, event.ID)
            continue
        }

        _ = w.repo.MarkSent(ctx, event.ID)
    }

    return nil
}
```

---

## 十二、验证命令

查看 outbox：

```sql
SELECT id, event_type, status, retry_count, sent_at
FROM outbox_events
ORDER BY created_at DESC;
```

查看 Kafka：

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic order.created \
  --from-beginning
```

---

## 十三、本节练习

1. 实现 outbox repository。
2. 实现 worker。
3. 启动 worker 前创建一条 PENDING 事件。
4. 启动 worker 后确认 status 变 SENT。
5. 用 Kafka consumer 查看事件。

---

## 十四、事务版 LockPending

前面的 `LockPending` 为了讲清楚流程做了简化。更严谨的写法是让 worker 每一轮开启事务，在事务里锁定一批事件：

```go
func (r *Repository) LockPendingTx(ctx context.Context, tx pgx.Tx, limit int) ([]Event, error) {
    rows, err := tx.Query(ctx, `
        SELECT id, topic, message_key, payload
        FROM outbox_events
        WHERE status = 'PENDING'
          AND (next_retry_at IS NULL OR next_retry_at <= now())
        ORDER BY created_at
        LIMIT $1
        FOR UPDATE SKIP LOCKED
    `, limit)
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var events []Event
    for rows.Next() {
        var e Event
        if err := rows.Scan(&e.ID, &e.Topic, &e.MessageKey, &e.Payload); err != nil {
            return nil, err
        }
        events = append(events, e)
    }
    return events, rows.Err()
}
```

`SKIP LOCKED` 的意义是：如果同时启动多个 worker，它们不会拿到同一批 outbox 记录。

---

## 十五、runOnce 的事务边界

一种简单做法是：

```text
事务 1：锁定并读出一批 outbox。
逐条发送 Kafka。
每条发送后单独更新状态。
```

不要把“发送 Kafka”放在数据库事务里长时间占锁。Kafka 发送可能慢，长事务会影响数据库。

学习项目可以这样折中：

```go
func (w *Worker) runOnce(ctx context.Context) error {
    events, err := w.repo.FetchPending(ctx, 100)
    if err != nil {
        return err
    }

    for _, event := range events {
        if err := w.publishOne(ctx, event); err != nil {
            w.logger.Error("publish outbox failed", "event_id", event.ID, "error", err)
        }
    }
    return nil
}
```

后续进阶时，可以增加 `PROCESSING` 状态，避免 worker 崩溃时状态不清晰。

---

## 十六、publishOne 模板

```go
func (w *Worker) publishOne(ctx context.Context, event Event) error {
    err := w.producer.Publish(ctx, event.Topic, kafka.Message{
        Key:   []byte(event.MessageKey),
        Value: event.Payload,
    })
    if err != nil {
        if markErr := w.repo.MarkRetry(ctx, event.ID, err.Error()); markErr != nil {
            return fmt.Errorf("publish failed: %v; mark retry failed: %w", err, markErr)
        }
        return err
    }

    if err := w.repo.MarkSent(ctx, event.ID); err != nil {
        return fmt.Errorf("mark sent: %w", err)
    }
    return nil
}
```

注意：发送成功但 `MarkSent` 失败，会导致重复发送。这是 Outbox 模式需要 consumer 幂等的根本原因之一。

---

## 十七、重试上限

`MarkRetry` 可以增加最大次数：

```sql
UPDATE outbox_events
SET retry_count = retry_count + 1,
    next_retry_at = now() + interval '1 minute',
    status = CASE
      WHEN retry_count + 1 >= 10 THEN 'FAILED'
      ELSE 'PENDING'
    END
WHERE id = $1;
```

生产里还建议增加：

```text
last_error
locked_at
locked_by
```

学习项目可以先在日志里记录错误。

---

## 十八、常见错误

### 1. worker 查出 PENDING 后直接删除

不要删除。删除会让排查和重试都变困难。发送成功标记 `SENT`，失败保留记录。

### 2. 忽略 MarkSent 错误

忽略后你会不知道哪些事件可能重复发送。至少要记录错误日志和指标。

### 3. 没有批量限制

一次查出所有 PENDING 会导致内存和数据库压力。用 `LIMIT` 分批处理。

### 4. 多 worker 没有锁

多个 worker 可能同时发送同一事件。即使 consumer 幂等，也会浪费资源并放大重复量。

---

## 十九、指标建议

outbox-worker 至少记录：

```text
outbox_publish_total{result="success"}
outbox_publish_total{result="error"}
outbox_pending_total
outbox_oldest_pending_age_seconds
outbox_retry_total
```

其中最重要的是：

```text
outbox_oldest_pending_age_seconds
```

它能告诉你事件从业务库到 Kafka 的延迟是否越来越大。
