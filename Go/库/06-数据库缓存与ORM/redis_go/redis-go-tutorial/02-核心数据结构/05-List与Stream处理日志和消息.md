# 5. List 与 Stream 处理日志和消息

本节目标：你将能用 List 做简单任务队列，用 Stream 保存可消费的业务事件。

简短引入：后端项目中，Redis 不只做缓存，也常被用来做轻量级异步处理。例如注册后发欢迎通知、记录操作日志、把耗时任务交给后台 worker。

## 一、为什么需要它

有些操作不应该阻塞用户请求：

- 用户注册后发送邮件。
- 订单创建后写审计日志。
- 文章发布后刷新搜索索引。
- 短链接访问后记录统计事件。

可以理解为，请求线程把任务放进 Redis，后台 worker 慢慢处理。

```text
Redis 队列适合轻量任务；强可靠、高吞吐、复杂重试通常应考虑 Kafka、RabbitMQ 等消息系统。
```

## 二、基本用法

List 示例：生产任务和消费任务。

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/redis/go-redis/v9"
)

func main() {
	ctx := context.Background()
	rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
	defer rdb.Close()

	queue := "queue:email"

	if err := rdb.LPush(ctx, queue, "send-welcome-email:user:1001").Err(); err != nil {
		log.Fatal(err)
	}

	task, err := rdb.BRPop(ctx, 5*time.Second, queue).Result()
	if err != nil {
		log.Fatal(err)
	}

	fmt.Println("queue:", task[0])
	fmt.Println("task:", task[1])
}
```

运行：

Windows PowerShell：

```powershell
go run .
```

Linux/macOS：

```bash
go run .
```

## 三、关键参数/语法/代码结构

`LPush` 从左侧写入。

`BRPop` 从右侧阻塞读取。没有任务时最多等待指定时间。

List 的优点是简单，缺点是确认、重试、消费组能力弱。任务被取走后，如果 worker 崩溃，就需要你自己设计补偿。

Stream 更适合保存事件日志：

```go
id, err := rdb.XAdd(ctx, &redis.XAddArgs{
	Stream: "stream:order-events",
	Values: map[string]any{
		"type":     "order_created",
		"order_id": "202607060001",
		"user_id":  "1001",
	},
}).Result()
if err != nil {
	log.Fatal(err)
}
fmt.Println("event id:", id)
```

## 四、真实后端场景示例

订单创建后写入 Stream：

```go
func PublishOrderCreated(ctx context.Context, rdb *redis.Client, orderID string, userID int64) error {
	_, err := rdb.XAdd(ctx, &redis.XAddArgs{
		Stream: "stream:order-events",
		MaxLen: 10000,
		Approx: true,
		Values: map[string]any{
			"type":     "order_created",
			"order_id": orderID,
			"user_id":  userID,
		},
	}).Result()
	return err
}
```

`MaxLen` 用来限制 Stream 长度，避免日志无限增长。`Approx: true` 表示近似裁剪，性能更友好。

真实项目中，创建订单通常要先写数据库，数据库事务提交成功后再发布事件。否则可能出现“消息发了，但订单没创建成功”的问题。

## 五、注意点

List 适合非常简单的后台任务，Stream 适合需要事件 ID、消费组、可追踪的场景。

如果任务必须保证不丢，单靠 Redis List 不够。你要设计：

- 失败重试。
- 死信队列。
- 幂等处理。
- 任务状态落库。

不要把 Redis 当无限日志仓库。Stream、List 都需要长度控制或归档策略。

## 六、常见误区

- 误区：用 List 做所有消息队列。  
  原因：确认机制和故障恢复不够完整。

- 误区：事件写入 Redis 就不写数据库日志。  
  原因：Redis 数据可能过期、被裁剪或丢失。

- 误区：后台任务不做幂等。  
  原因：重试时同一任务可能执行多次。

- 误区：不限制 Stream 长度。  
  原因：内存会持续增长。

## 七、本节达标标准

- 能用 List 写入和阻塞读取任务。
- 能用 Stream 写入业务事件。
- 能解释 List 与 Stream 的适用场景。
- 能说明 Redis 队列的可靠性边界。
- 能为事件流设置长度控制。

