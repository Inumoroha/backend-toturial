# 2. 连接 Redis 与配置连接池

本节目标：你将能用 Go 连接 Redis，执行一次读写，并配置基本的超时和连接池参数。

简短引入：真实项目里，Redis 客户端通常是一个长期复用的组件，而不是每个请求临时创建一次。连接方式写错，会导致性能抖动、连接泄漏或请求卡死。

## 一、为什么需要它

Redis 是网络服务。Go 程序每次访问 Redis，都涉及网络连接、超时、序列化和错误处理。

可以理解为，连接池就是 Go 服务和 Redis 之间的一组可复用通道。请求来了，从池里取一个连接，用完还回去。

```text
Redis 客户端应该在服务启动时创建，在服务退出时关闭，不要在每个 HTTP 请求里反复 NewClient。
```

## 二、基本用法

创建 `main.go`：

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
	ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
	defer cancel()

	rdb := redis.NewClient(&redis.Options{
		Addr:         "localhost:6379",
		DB:           0,
		PoolSize:     10,
		MinIdleConns: 2,
		DialTimeout:  3 * time.Second,
		ReadTimeout:  2 * time.Second,
		WriteTimeout: 2 * time.Second,
	})
	defer rdb.Close()

	if err := rdb.Ping(ctx).Err(); err != nil {
		log.Fatalf("ping redis failed: %v", err)
	}

	if err := rdb.Set(ctx, "user:1:name", "小林", 10*time.Minute).Err(); err != nil {
		log.Fatalf("set cache failed: %v", err)
	}

	name, err := rdb.Get(ctx, "user:1:name").Result()
	if err != nil {
		log.Fatalf("get cache failed: %v", err)
	}

	fmt.Println("cached user name:", name)
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

`Addr` 是 Redis 地址，本地 Docker 默认是 `localhost:6379`。

`DB` 是 Redis 逻辑库编号。学习阶段可以用 0。真实项目中不要靠切 DB 来做环境隔离，更推荐不同实例、不同前缀或不同命名空间。

`PoolSize` 是最大连接数量。不是越大越好，过大会压垮 Redis，也会浪费服务资源。

`DialTimeout` 控制建立连接的最长等待时间。

`ReadTimeout` 和 `WriteTimeout` 控制读写 Redis 的最长等待时间。

`context.WithTimeout` 是请求级别的保护。真实 HTTP 服务里通常会使用请求自带的 `ctx`，再按需要加超时。

## 四、真实后端场景示例

在一个用户服务里，你通常会这样封装 Redis 客户端：

```go
package cache

import (
	"context"
	"time"

	"github.com/redis/go-redis/v9"
)

func NewRedisClient(addr string) (*redis.Client, error) {
	rdb := redis.NewClient(&redis.Options{
		Addr:         addr,
		PoolSize:     20,
		MinIdleConns: 5,
		DialTimeout:  3 * time.Second,
		ReadTimeout:  2 * time.Second,
		WriteTimeout: 2 * time.Second,
	})

	ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
	defer cancel()

	if err := rdb.Ping(ctx).Err(); err != nil {
		_ = rdb.Close()
		return nil, err
	}

	return rdb, nil
}
```

服务启动时调用一次 `NewRedisClient`，然后把 `*redis.Client` 注入到 handler、service 或 repository 中。

## 五、注意点

Redis 连接失败时，服务是否直接启动失败，要看业务性质。

常见判断：

- 如果 Redis 用于登录态、库存锁等关键链路，启动失败可以直接拒绝启动。
- 如果 Redis 只是文章详情缓存，启动失败可以降级为直接查数据库，但要记录日志和指标。

```text
缓存可以降级，不代表所有 Redis 用法都可以忽略失败。
```

## 六、常见错误

- 错误：在每个请求里创建 `redis.NewClient`。  
  结果：连接数暴涨，延迟变高，Redis 容易被打满。

- 错误：所有 Redis 操作都用 `context.Background()`。  
  结果：请求取消后 Redis 操作还在跑，排查超时会很痛苦。

- 错误：不设置超时。  
  结果：网络异常时，请求可能长时间卡住。

- 错误：把 Redis 错误全部吞掉。  
  结果：线上故障没有日志，业务表现变成随机慢或随机失败。

## 七、本节达标标准

- 能用 Go 成功 `Ping` Redis。
- 能执行 `Set` 和 `Get`。
- 能解释为什么 Redis 客户端要复用。
- 能配置基础连接池和超时。
- 能区分启动时连接检查和请求中 Redis 操作失败。

