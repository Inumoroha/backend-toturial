# 9. 库存扣减与 Lua 原子操作

本节目标：你将能用 Redis Lua 脚本实现一次安全的库存检查和扣减。

简短引入：库存是 Redis 学习中非常重要的并发场景。简单的 `Get` 后再 `Set` 在并发请求下会出错，因为两个请求可能同时看到相同库存。

## 一、为什么需要它

秒杀、下单、优惠券领取都需要先判断剩余数量，再扣减。

错误写法常见是：

1. `GET stock`
2. Go 判断库存是否大于 0
3. `DECR stock`

问题是多个请求可以同时通过第 2 步，导致超卖。

```text
检查库存和扣减库存必须是一个不可拆开的操作。
```

## 二、基本用法

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/redis/go-redis/v9"
)

var decreaseStockScript = redis.NewScript(`
local stock = tonumber(redis.call("GET", KEYS[1]))
local amount = tonumber(ARGV[1])

if stock == nil then
	return -1
end

if stock < amount then
	return 0
end

return redis.call("DECRBY", KEYS[1], amount)
`)

func DecreaseStock(ctx context.Context, rdb *redis.Client, key string, amount int64) (int64, error) {
	result, err := decreaseStockScript.Run(ctx, rdb, []string{key}, amount).Int64()
	if err != nil {
		return 0, err
	}
	if result == -1 {
		return 0, fmt.Errorf("stock key not found")
	}
	if result == 0 {
		return 0, fmt.Errorf("stock not enough")
	}
	return result, nil
}

func main() {
	ctx := context.Background()
	rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
	defer rdb.Close()

	key := "stock:sku:1001"
	_ = rdb.Set(ctx, key, 10, 0).Err()

	left, err := DecreaseStock(ctx, rdb, key, 2)
	if err != nil {
		log.Fatal(err)
	}

	fmt.Println("left stock:", left)
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

`decreaseStockScript` 是库存扣减脚本。它先读取库存，再判断是否足够，最后执行 `DECRBY`。

`DecreaseStock` 是 Go 侧封装。真实项目中不要让 handler 直接到处调用 Lua 脚本，最好封装成业务含义明确的函数。

Lua 脚本在 Redis 中执行时具有原子性。这里把“读取库存、判断库存、扣减库存”放在一个脚本里，避免并发请求插队。

返回值约定：

- `-1`：库存 key 不存在。
- `0`：库存不足。
- 正数：扣减后的剩余库存。

## 四、真实后端场景示例

下单扣库存的保守流程：

1. 校验用户、商品、购买数量。
2. 在 Redis 中扣减活动库存。
3. 创建订单，写数据库事务。
4. 如果订单创建失败，需要回滚 Redis 库存。
5. 支付超时取消订单时，需要释放库存。

回滚库存示例：

```go
func RollbackStock(ctx context.Context, rdb *redis.Client, key string, amount int64) error {
	return rdb.IncrBy(ctx, key, amount).Err()
}
```

真实项目中，库存回滚要有订单状态约束。不能因为接口重试就重复加库存。

## 五、注意点

Redis 扣减库存只是并发控制的一部分。最终订单仍要写数据库，数据库也应有事务边界和唯一约束，例如同一用户同一活动只能下一单。

如果 Redis 扣减成功但数据库写入失败，要有补偿逻辑。如果补偿也失败，要记录待修复任务。

Lua 脚本要保持短小，不要在里面做复杂业务逻辑。复杂业务应该留在 Go 服务和数据库事务里。

```text
Redis 负责快速原子判断，数据库负责最终业务事实。
```

## 六、常见误区

- 误区：`GET` 后在 Go 里判断，再 `DECR`。  
  原因：并发下不是原子操作，会超卖。

- 误区：Redis 扣库存成功就算订单成功。  
  原因：订单数据库写入可能失败。

- 误区：回滚库存不做幂等。  
  原因：重复回滚会把库存加多。

- 误区：Lua 脚本写得很复杂。  
  原因：脚本阻塞 Redis 执行，复杂逻辑会影响所有请求。

## 七、本节达标标准

- 能解释库存扣减为什么需要原子性。
- 能使用 Lua 完成检查和扣减。
- 能处理库存不存在和库存不足。
- 能设计扣减失败、订单失败和库存回滚流程。
- 能说明 Redis 库存与数据库订单的边界。
