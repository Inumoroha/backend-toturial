# 6. 综合实践：Go 完成订单事件收发

本节目标：把第 5 阶段内容串起来，用 Go 完成一条最小订单事件链路：producer 发布 `order.created`，consumer group 消费并打印处理日志。

这一节仍然是教学版，不包含完整数据库、幂等、retry、DLQ。它的目标是让 Go 和 Kafka 的基础链路跑通。可靠性会在第 6 阶段继续补齐。

---

## 一、项目结构

```text
kafka-go-lab/
  go.mod
  cmd/
    order-producer/
      main.go
    inventory-consumer/
      main.go
  internal/
    kafka/
      config.go
      message.go
      producer.go
      consumer.go
    order/
      event.go
    inventory/
      handler.go
```

---

## 二、初始化项目

```powershell
mkdir kafka-go-lab
cd kafka-go-lab
go mod init kafka-go-lab
go get github.com/twmb/franz-go/pkg/kgo
```

---

## 三、创建 Topic

进入 Kafka 容器：

```powershell
docker exec -it kafka bash
```

创建 topic：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --if-not-exists \
  --topic order.created \
  --partitions 3 \
  --replication-factor 1
```

---

## 四、定义订单事件

`internal/order/event.go`：

```go
package order

import "time"

type OrderCreatedEvent struct {
    EventID    string    `json:"event_id"`
    EventType  string    `json:"event_type"`
    Version    int       `json:"version"`
    OccurredAt time.Time `json:"occurred_at"`
    Data       OrderData `json:"data"`
}

type OrderData struct {
    OrderID     string `json:"order_id"`
    UserID      string `json:"user_id"`
    TotalAmount int64  `json:"total_amount"`
}
```

---

## 五、Producer main

`cmd/order-producer/main.go`：

```go
package main

import (
    "context"
    "encoding/json"
    "log"
    "time"

    "github.com/twmb/franz-go/pkg/kgo"
)

func main() {
    client, err := kgo.NewClient(
        kgo.SeedBrokers("localhost:9092"),
        kgo.ClientID("order-service"),
        kgo.DefaultProduceTopic("order.created"),
        kgo.RequiredAcks(kgo.AllISRAcks()),
    )
    if err != nil {
        log.Fatal(err)
    }
    defer client.Close()

    event := map[string]any{
        "event_id":    "evt_order_created_1001",
        "event_type":  "order.created",
        "version":     1,
        "occurred_at": time.Now().UTC(),
        "data": map[string]any{
            "order_id":     "order_1001",
            "user_id":      "user_88",
            "total_amount": 19800,
        },
    }

    payload, err := json.Marshal(event)
    if err != nil {
        log.Fatal(err)
    }

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    results := client.ProduceSync(ctx, &kgo.Record{
        Key:   []byte("order_1001"),
        Value: payload,
        Headers: []kgo.RecordHeader{
            {Key: "event_id", Value: []byte("evt_order_created_1001")},
            {Key: "event_type", Value: []byte("order.created")},
        },
    })
    if err := results.FirstErr(); err != nil {
        log.Fatal(err)
    }

    log.Println("order.created published")
}
```

运行：

```powershell
go run ./cmd/order-producer
```

---

## 六、Consumer main

`cmd/inventory-consumer/main.go`：

```go
package main

import (
    "context"
    "encoding/json"
    "errors"
    "log"
    "os"
    "os/signal"
    "syscall"

    "github.com/twmb/franz-go/pkg/kgo"
)

func main() {
    ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer stop()

    client, err := kgo.NewClient(
        kgo.SeedBrokers("localhost:9092"),
        kgo.ClientID("inventory-service"),
        kgo.ConsumerGroup("inventory-service"),
        kgo.ConsumeTopics("order.created"),
        kgo.DisableAutoCommit(),
    )
    if err != nil {
        log.Fatal(err)
    }
    defer client.Close()

    for ctx.Err() == nil {
        fetches := client.PollFetches(ctx)
        if errs := fetches.Errors(); len(errs) > 0 {
            for _, fetchErr := range errs {
                if errors.Is(fetchErr.Err, context.Canceled) {
                    return
                }
                log.Printf("fetch error: topic=%s partition=%d err=%v",
                    fetchErr.Topic, fetchErr.Partition, fetchErr.Err)
            }
            continue
        }

        fetches.EachRecord(func(record *kgo.Record) {
            var event map[string]any
            if err := json.Unmarshal(record.Value, &event); err != nil {
                log.Printf("decode failed: topic=%s partition=%d offset=%d err=%v",
                    record.Topic, record.Partition, record.Offset, err)
                return
            }

            log.Printf("handle order event: topic=%s partition=%d offset=%d key=%s event_id=%v",
                record.Topic, record.Partition, record.Offset, string(record.Key), event["event_id"])

            if err := client.CommitRecords(ctx, record); err != nil {
                log.Printf("commit failed: topic=%s partition=%d offset=%d err=%v",
                    record.Topic, record.Partition, record.Offset, err)
            }
        })
    }
}
```

运行：

```powershell
go run ./cmd/inventory-consumer
```

---

## 七、查看 Consumer Group

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group inventory-service
```

观察：

- `CURRENT-OFFSET`
- `LOG-END-OFFSET`
- `LAG`

---

## 八、这个实践还缺什么

当前版本只是最小链路，还缺：

- 统一 `internal/kafka` 封装。
- 业务 struct 解码。
- retry/DLQ。
- 幂等。
- 数据库。
- Prometheus。
- 更完整优雅退出。

这些会在后续阶段逐步补齐。

---

## 九、本节练习

1. 跑通 producer。
2. 跑通 consumer。
3. 查看 group lag。
4. 修改 producer 连续发送 10 条消息。
5. 停止 consumer 后发送消息，观察 lag 增长。
6. 重启 consumer，观察 lag 下降。
7. 把示例 main 拆成 `internal/kafka` 封装。

---

## 十、本节小结

- Go producer 可以用 `ProduceSync` 完成同步发送。
- Go consumer group 要关闭自动提交。
- handler 成功后手动提交 offset。
- 日志要记录 topic、partition、offset、key、event_id。
- 最小链路跑通后，再逐步加入可靠性能力。

---

## 十一、验收命令

启动 producer 后，使用命令行确认 topic 中有消息：

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic order.created \
  --from-beginning \
  --property print.key=true \
  --max-messages 1
```

启动 Go consumer 后查看 group：

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group inventory-service
```

预期：

```text
CURRENT-OFFSET 前进。
LAG 最终为 0。
```

---

## 十二、下一步改造

最小收发完成后，按顺序改造：

1. 把 producer main 中的发送逻辑移到 `internal/kafka`。
2. 把 consumer main 中的处理逻辑抽成 handler。
3. 增加错误分类。
4. 增加 retry/DLQ producer。
5. 增加 processed_events 幂等。

不要在最小链路还没跑通时就直接写复杂可靠性代码。

---

## 十三、综合实践交付物

完成后至少留下：

```text
cmd/order-producer/main.go
cmd/inventory-consumer/main.go
internal/kafka/message.go
internal/kafka/producer.go
internal/kafka/consumer.go
```

README 中写清楚启动 producer、启动 consumer、查看 lag 的命令。

---

## 十四、README 片段

```markdown
## 本地验证

1. 启动 Kafka。
2. `go run ./cmd/order-producer`
3. `go run ./cmd/inventory-consumer`
4. 使用 `kafka-consumer-groups.sh` 查看 lag。
```

综合实践必须让别人也能复现。

---

## 二十一、README 应包含的内容

把本节项目整理成 README 时，至少写清：

```text
如何启动 Kafka。
如何创建 topic。
如何设置环境变量。
如何启动 producer。
如何启动 consumer。
如何发送一条订单事件。
如何查看 consumer group lag。
如何模拟 handler 失败。
如何确认 offset 是否提交。
```

这份 README 是工程化交付的一部分。能跑通代码是一回事，能让别人稳定复现是另一回事。
