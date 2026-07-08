# 05 使用 franz-go 编写 Producer 与 Consumer

本节目标：用一个纯 Go 客户端把 Kafka 生产和消费流程写出来。这里选择 `franz-go` 作为教学主线，因为它不依赖 CGO，适合学习 Go 后端项目中的封装思路。

> 说明：不同客户端 API 细节不同，但本节关注的工程原则通用：统一封装、context 超时、手动提交、优雅退出、日志和错误分类。

## 初始化项目

```powershell
mkdir kafka-go-lab
cd kafka-go-lab
go mod init kafka-go-lab
go get github.com/twmb/franz-go/pkg/kgo
```

建议目录：

```text
kafka-go-lab/
  cmd/
    producer/
      main.go
    consumer/
      main.go
  internal/
    events/
      order.go
    kafkax/
      producer.go
      consumer.go
      message.go
```

## 事件结构

`internal/events/order.go`：

```go
package events

import "time"

type OrderCreated struct {
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

字段解释：

- `EventID` 用于 consumer 幂等。
- `EventType` 用于路由和日志。
- `Version` 用于 schema 演进。
- `OrderID` 适合作为 Kafka message key。
- 金额用整数分，不用 float。

## Producer 示例

`cmd/producer/main.go`：

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "time"

    "github.com/twmb/franz-go/pkg/kgo"
)

type OrderCreated struct {
    EventID    string    `json:"event_id"`
    EventType  string    `json:"event_type"`
    Version    int       `json:"version"`
    OccurredAt time.Time `json:"occurred_at"`
    Data       struct {
        OrderID     string `json:"order_id"`
        UserID      string `json:"user_id"`
        TotalAmount int64  `json:"total_amount"`
    } `json:"data"`
}

func main() {
    ctx := context.Background()

    client, err := kgo.NewClient(
        kgo.SeedBrokers("localhost:9092"),
        kgo.DefaultProduceTopic("order.created"),
        kgo.ClientID("order-service"),
        kgo.RequiredAcks(kgo.AllISRAcks()),
    )
    if err != nil {
        log.Fatal(err)
    }
    defer client.Close()

    event := OrderCreated{
        EventID:    "evt_order_created_1001",
        EventType:  "order.created",
        Version:    1,
        OccurredAt: time.Now().UTC(),
    }
    event.Data.OrderID = "order_1001"
    event.Data.UserID = "user_88"
    event.Data.TotalAmount = 19800

    payload, err := json.Marshal(event)
    if err != nil {
        log.Fatal(err)
    }

    sendCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    record := &kgo.Record{
        Key:   []byte(event.Data.OrderID),
        Value: payload,
        Headers: []kgo.RecordHeader{
            {Key: "event_id", Value: []byte(event.EventID)},
            {Key: "event_type", Value: []byte(event.EventType)},
            {Key: "schema_version", Value: []byte(fmt.Sprint(event.Version))},
        },
    }

    results := client.ProduceSync(sendCtx, record)
    if err := results.FirstErr(); err != nil {
        log.Fatalf("publish failed: topic=%s key=%s event_id=%s err=%v",
            "order.created", event.Data.OrderID, event.EventID, err)
    }

    log.Printf("publish success: topic=%s key=%s event_id=%s",
        "order.created", event.Data.OrderID, event.EventID)
}
```

### 这段代码要看懂什么

- producer 是长期对象，不应该每条消息创建一次。
- `context.WithTimeout` 防止发送无限等待。
- key 使用 `order_id`，保证同一订单事件进入同一 partition。
- header 放 event metadata，便于日志、trace 和排查。
- 发送失败要返回给业务或进入补偿流程。

## 创建 Topic

运行 producer 前先创建 topic：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --if-not-exists \
  --topic order.created \
  --partitions 3 \
  --replication-factor 1
```

运行：

```powershell
go run .\cmd\producer
```

验证：

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic order.created \
  --from-beginning \
  --property print.key=true \
  --property print.headers=true
```

## Consumer 示例

`cmd/consumer/main.go`：

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
    "time"

    "github.com/twmb/franz-go/pkg/kgo"
)

type OrderCreated struct {
    EventID   string `json:"event_id"`
    EventType string `json:"event_type"`
    Version   int    `json:"version"`
    Data      struct {
        OrderID     string `json:"order_id"`
        UserID      string `json:"user_id"`
        TotalAmount int64  `json:"total_amount"`
    } `json:"data"`
}

func main() {
    rootCtx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
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

    for {
        fetches := client.PollFetches(rootCtx)
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
            started := time.Now()

            if err := handleOrderCreated(rootCtx, record); err != nil {
                log.Printf("handle failed: topic=%s partition=%d offset=%d key=%s err=%v",
                    record.Topic, record.Partition, record.Offset, string(record.Key), err)
                return
            }

            if err := client.CommitRecords(rootCtx, record); err != nil {
                log.Printf("commit failed: topic=%s partition=%d offset=%d key=%s err=%v",
                    record.Topic, record.Partition, record.Offset, string(record.Key), err)
                return
            }

            log.Printf("handle success: topic=%s partition=%d offset=%d key=%s duration_ms=%d",
                record.Topic, record.Partition, record.Offset, string(record.Key),
                time.Since(started).Milliseconds())
        })
    }
}

func handleOrderCreated(ctx context.Context, record *kgo.Record) error {
    var event OrderCreated
    if err := json.Unmarshal(record.Value, &event); err != nil {
        return err
    }

    if event.EventID == "" || event.Data.OrderID == "" {
        return errors.New("invalid order.created event")
    }

    // 真实项目中这里应该：
    // 1. 开启数据库事务
    // 2. 用 event_id 做幂等去重
    // 3. 扣减库存
    // 4. 记录 processed_events
    // 5. 提交数据库事务
    log.Printf("deduct inventory: event_id=%s order_id=%s amount=%d",
        event.EventID, event.Data.OrderID, event.Data.TotalAmount)

    return nil
}
```

### 这段代码要看懂什么

- `DisableAutoCommit()` 让 offset 提交由代码控制。
- `handleOrderCreated` 成功后才 `CommitRecords`。
- commit 失败时，允许重复消费，所以 handler 必须幂等。
- 解析失败时这里暂时没有提交 offset，生产项目应该写 DLQ 成功后再提交。
- 收到退出信号后 context 会取消，poll 会返回错误，服务退出。

## 这个示例还缺什么

它是教学最小版本，生产项目还需要：

- retry topic producer。
- DLQ producer。
- 幂等去重表。
- Prometheus metrics。
- 更细的错误分类。
- handler 超时控制。
- rebalance 生命周期日志。
- 配置文件或环境变量。

这些会在后续可靠性和项目实战阶段补上。

## 本节验收

你需要完成：

1. 运行 producer，向 `order.created` 写入一条消息。
2. 运行 consumer，看到库存扣减日志。
3. 查看 `inventory-service` group 的 offset。
4. 停止 consumer 后再发送消息，观察 lag 增长。
5. 重启 consumer，观察 lag 下降。
6. 修改消息为非法 JSON，思考为什么需要 DLQ。

