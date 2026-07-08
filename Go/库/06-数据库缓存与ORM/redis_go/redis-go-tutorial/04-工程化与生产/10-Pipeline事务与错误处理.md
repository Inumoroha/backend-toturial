# 10. Pipeline 事务与错误处理

本节目标：你将能区分 Pipeline、事务和普通命令，并写出更稳妥的 Redis 错误处理代码。

简短引入：很多初学者看到 Pipeline 和事务都会混在一起。真实项目里，它们解决的问题不同：Pipeline 主要减少网络往返，事务主要保证一组命令按顺序执行。

## 一、为什么需要它

如果一个接口要连续写多个 Redis key，比如：

- 写用户摘要缓存。
- 写用户权限集合。
- 设置过期时间。

逐条命令发送会产生多次网络往返。Pipeline 可以把命令打包发送，提高性能。

```text
Pipeline 不是事务，它主要优化网络开销。
```

## 二、基本用法

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

	pipe := rdb.Pipeline()
	nameCmd := pipe.Set(ctx, "user:1001:name", "小林", 10*time.Minute)
	permCmd := pipe.SAdd(ctx, "user:1001:permissions", "article:read", "article:write")
	pipe.Expire(ctx, "user:1001:permissions", 10*time.Minute)

	_, err := pipe.Exec(ctx)
	if err != nil {
		log.Fatal(err)
	}

	fmt.Println("set name:", nameCmd.Err())
	fmt.Println("add permissions:", permCmd.Val())
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

`Pipeline()` 创建普通流水线。命令会先排队，`Exec` 时一起发送。

`TxPipeline()` 会使用 Redis 的 `MULTI/EXEC` 包裹命令。它能保证这一组命令作为事务队列执行，但不能像关系型数据库事务一样支持复杂回滚。

示例：

```go
pipe := rdb.TxPipeline()
pipe.Set(ctx, "user:1001:name", "小林", 10*time.Minute)
pipe.SAdd(ctx, "user:1001:permissions", "article:read")
_, err := pipe.Exec(ctx)
```

真实项目中，Redis 事务能力有限。如果要处理强一致业务状态，还是要依赖数据库事务。

## 四、真实后端场景示例

用户登录后，写登录态和用户摘要：

```go
func CacheLoginSession(ctx context.Context, rdb *redis.Client, token string, userID int64) error {
	sessionKey := fmt.Sprintf("session:%s", token)
	userKey := fmt.Sprintf("user:summary:%d", userID)

	pipe := rdb.Pipeline()
	pipe.Set(ctx, sessionKey, userID, 2*time.Hour)
	pipe.HSet(ctx, userKey, map[string]any{
		"id":       userID,
		"nickname": "小林",
	})
	pipe.Expire(ctx, userKey, 30*time.Minute)

	_, err := pipe.Exec(ctx)
	return err
}
```

这里使用 Pipeline 是为了减少网络往返。即使用户摘要缓存失败，登录态是否必须失败，要根据业务判断。安全敏感场景要更保守。

## 五、注意点

错误处理要区分：

- `redis.Nil`：key 不存在。
- 网络错误：Redis 不可用、超时、连接断开。
- 数据错误：反序列化失败、类型不匹配。
- 业务错误：库存不足、权限不存在。

对于缓存读取失败，很多场景可以降级查数据库。对于分布式锁、库存扣减、登录态写入，不能随便忽略错误。

```text
能降级的是缓存加速，不是所有 Redis 依赖。
```

## 六、常见误区

- 误区：Pipeline 等于事务。  
  原因：普通 Pipeline 只减少网络往返，不保证事务语义。

- 误区：Redis 事务能像数据库一样回滚。  
  原因：Redis 事务没有传统关系型数据库那种自动回滚模型。

- 误区：`Exec` 没报错就不看单个命令结果。  
  原因：部分命令可能有自己的返回值或业务判断。

- 误区：所有 Redis 错误都忽略。  
  原因：关键链路会产生真实业务风险。

## 七、本节达标标准

- 能用 Pipeline 批量发送命令。
- 能说明 Pipeline 与 TxPipeline 的区别。
- 能判断哪些 Redis 错误可以降级。
- 能正确处理 `redis.Nil`。
- 能把 Redis 事务和数据库事务区分开。

