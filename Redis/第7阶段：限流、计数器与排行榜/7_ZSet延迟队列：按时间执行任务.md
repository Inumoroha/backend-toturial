# 7. ZSet 延迟队列：按时间执行任务

ZSet 不只可以做排行榜，也可以做轻量延迟队列。

核心思想是：用 score 保存任务执行时间，worker 定期取出到期任务执行。

学完这一节后，你应该能够：

- 使用 ZSet 设计延迟队列。
- 用 score 表示执行时间。
- 原子领取到期任务。
- 在 Go 中实现一个简单 worker。
- 理解 ZSet 延迟队列的可靠性边界。

---

## 一、什么是延迟队列

延迟队列表示任务在未来某个时间点执行。

例如：

```text
订单创建后 30 分钟未支付，自动取消。
```

创建订单时写入任务：

```text
task = cancel_order:1001
execute_at = now + 30min
```

到时间后 worker 执行取消逻辑。

---

## 二、为什么 ZSet 可以做延迟队列

ZSet 的 score 可以排序。

把执行时间作为 score：

```redis
ZADD delay:order:cancel 1790000000000 order:1001
```

查询到期任务：

```redis
ZRANGEBYSCORE delay:order:cancel 0 now_ms LIMIT 0 10
```

删除任务：

```redis
ZREM delay:order:cancel order:1001
```

只要 score 小于当前时间，就说明任务到期。

---

## 三、任务 member 设计

member 要唯一。

示例：

```text
order:1001
retry_email:msg_abc123
coupon_expire_notice:task_9527
```

如果同一个订单只需要一个取消任务：

```text
member = order_id
```

如果同一个业务对象可能有多次任务：

```text
member = task_id
```

任务详情可以：

- 放在 member 中，只保存简单 ID。
- 另用 Hash 保存任务详情。
- 放在数据库任务表中，Redis 只保存 task_id。

重要任务推荐数据库保存详情。

---

## 四、添加延迟任务

Go 示例：

```go
func AddDelayTask(ctx context.Context, rdb *redis.Client, queue string, member string, delay time.Duration) error {
    executeAt := time.Now().Add(delay).UnixMilli()
    return rdb.ZAdd(ctx, queue, redis.Z{
        Score:  float64(executeAt),
        Member: member,
    }).Err()
}
```

订单取消：

```go
err := AddDelayTask(ctx, rdb, "delay:order:cancel", "order:1001", 30*time.Minute)
```

---

## 五、领取任务的并发问题

多个 worker 同时查询：

```redis
ZRANGEBYSCORE delay:order:cancel 0 now LIMIT 0 1
```

可能都看到同一个任务。

如果随后都执行，就会重复处理。

所以领取任务要原子：

```text
查到期任务 + 从 ZSet 删除
```

只有删除成功的 worker 才能执行。

---

## 六、简单领取 Lua 脚本

```lua
local tasks = redis.call("ZRANGEBYSCORE", KEYS[1], 0, ARGV[1], "LIMIT", 0, ARGV[2])
if #tasks == 0 then
    return {}
end

local claimed = {}
for i, task in ipairs(tasks) do
    local removed = redis.call("ZREM", KEYS[1], task)
    if removed == 1 then
        table.insert(claimed, task)
    end
end

return claimed
```

参数：

```text
KEYS[1]：队列 key
ARGV[1]：当前时间毫秒
ARGV[2]：本次最多领取数量
```

脚本执行期间是原子的，不会有两个 worker 同时领到同一个任务。

---

## 七、Go worker 示例

```go
var claimDelayTasksScript = redis.NewScript(`
local tasks = redis.call("ZRANGEBYSCORE", KEYS[1], 0, ARGV[1], "LIMIT", 0, ARGV[2])
if #tasks == 0 then
    return {}
end

local claimed = {}
for i, task in ipairs(tasks) do
    local removed = redis.call("ZREM", KEYS[1], task)
    if removed == 1 then
        table.insert(claimed, task)
    end
end

return claimed
`)
```

```go
func ClaimDelayTasks(ctx context.Context, rdb *redis.Client, queue string, limit int64) ([]string, error) {
    now := time.Now().UnixMilli()

    vals, err := claimDelayTasksScript.Run(ctx, rdb, []string{queue}, now, limit).StringSlice()
    if err != nil {
        return nil, err
    }

    return vals, nil
}
```

---

## 八、轮询执行

```go
func RunDelayWorker(ctx context.Context, rdb *redis.Client, queue string, handler func(context.Context, string) error) error {
    ticker := time.NewTicker(500 * time.Millisecond)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-ticker.C:
            tasks, err := ClaimDelayTasks(ctx, rdb, queue, 10)
            if err != nil {
                log.Printf("claim delay tasks failed queue=%s err=%v", queue, err)
                continue
            }

            for _, task := range tasks {
                if err := handler(ctx, task); err != nil {
                    log.Printf("handle delay task failed task=%s err=%v", task, err)
                }
            }
        }
    }
}
```

这是教学版 worker。

生产环境要处理失败重试、并发执行、优雅退出和监控。

---

## 九、任务失败怎么办

上面的脚本领取任务时已经从 ZSet 删除。

如果 handler 执行失败，任务会丢失。

可选方案：

1. 失败后重新 `ZADD`，设置下次重试时间。
2. 把任务先移到 processing 队列，成功后确认删除。
3. 任务详情保存在数据库，Redis 只是调度索引。
4. 失败写入死信队列。

简单业务可以失败后重投：

```go
if err := handler(ctx, task); err != nil {
    _ = AddDelayTask(ctx, rdb, queue, task, time.Minute)
}
```

重要业务不要只靠这种内存式补救。

---

## 十、订单取消要幂等

延迟任务可能重复执行。

比如：

- worker 超时重试。
- 任务被重新投递。
- 订单状态已经变化。

取消订单时必须用状态条件：

```sql
UPDATE orders
SET status = 'cancelled'
WHERE id = ?
  AND status = 'pending';
```

如果订单已经支付，影响 0 行，任务应该直接结束。

延迟队列只能触发动作，不能替代业务状态机。

---

## 十一、轮询间隔和延迟精度

如果 worker 每 500ms 轮询一次，任务执行误差大约是毫秒到秒级。

Redis ZSet 延迟队列不适合极高精度定时。

适合：

- 秒级延迟。
- 分钟级延迟。
- 业务可接受少量抖动。

不适合：

- 严格毫秒级调度。
- 大规模复杂任务编排。
- 强可靠消息投递。

---

## 十二、ZSet 延迟队列和 Stream

ZSet 延迟队列：

- 适合按时间调度。
- 实现简单。
- 没有天然 ACK 和 Pending List。

Redis Stream：

- 更接近消息队列。
- 有消费者组。
- 有 ACK。
- 有 Pending List。

如果任务可靠性要求更高，后面第 8 阶段会学习 Stream。

---

## 十三、常见错误

### 1. 多 worker 非原子领取

可能重复执行同一任务。

### 2. 任务失败后直接丢失

领取后删除，失败要有重试或补偿。

### 3. 业务处理不幂等

延迟任务重复执行时会造成错误。

### 4. member 不唯一

后写任务覆盖前一个任务。

### 5. 把 ZSet 延迟队列当专业 MQ

复杂可靠消息场景应考虑 Stream、Kafka、RabbitMQ 等。

---

## 十四、本节练习

请完成下面练习：

1. 用 ZSet 设计订单取消延迟队列。
2. 写出添加 30 分钟后取消订单任务的命令。
3. 写出查询到期任务的命令。
4. 解释为什么领取任务要原子。
5. 写出领取任务 Lua 脚本。
6. 在 Go 中实现 `ClaimDelayTasks`。
7. 思考任务执行失败后如何重试。
8. 写出取消订单的幂等 SQL。

---

## 十五、本节小结

这一节你学习了 ZSet 延迟队列。

你需要记住：

- ZSet 的 score 可以表示任务执行时间。
- 到期任务可以通过 `ZRANGEBYSCORE` 查询。
- 多 worker 场景下，领取和删除要原子。
- 任务失败要有重试、补偿或数据库任务表。
- 延迟任务处理必须幂等。
- ZSet 延迟队列是轻量方案，不等于专业 MQ。

下一节我们把第 7 阶段内容串起来，做一个综合实践。

