# 3. String 缓存用户资料

本节目标：你将能用 Redis String 缓存用户资料，并用 TTL 控制缓存生命周期。

简短引入：String 是 Redis 中最常用的数据结构。真实项目里，它经常用来缓存一段 JSON、一个计数值、一个验证码或一个短期状态。

## 一、为什么需要它

用户详情页、文章详情页、商品详情页通常读多写少。如果每次请求都查数据库，数据库压力会变大。

可以理解为，String 缓存就是把数据库查询结果序列化成一段字符串，放到 Redis 里。下次请求先读 Redis，命中就不用查数据库。

```text
缓存里的数据是副本，不是事实来源；事实来源通常仍然是数据库。
```

## 二、基本用法

创建 `main.go`：

```go
package main

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"log"
	"time"

	"github.com/redis/go-redis/v9"
)

type UserProfile struct {
	ID       int64  `json:"id"`
	Nickname string `json:"nickname"`
	Avatar   string `json:"avatar"`
}

func main() {
	ctx := context.Background()
	rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
	defer rdb.Close()

	key := "user:profile:1001"
	profile := UserProfile{ID: 1001, Nickname: "小林", Avatar: "https://example.com/a.png"}

	raw, err := json.Marshal(profile)
	if err != nil {
		log.Fatal(err)
	}

	if err := rdb.Set(ctx, key, raw, 15*time.Minute).Err(); err != nil {
		log.Fatal(err)
	}

	cached, err := rdb.Get(ctx, key).Bytes()
	if errors.Is(err, redis.Nil) {
		fmt.Println("cache miss")
		return
	}
	if err != nil {
		log.Fatal(err)
	}

	var got UserProfile
	if err := json.Unmarshal(cached, &got); err != nil {
		log.Fatal(err)
	}

	fmt.Printf("user profile: %+v\n", got)
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

`Set(ctx, key, value, expiration)` 的第四个参数是过期时间。`15*time.Minute` 表示 15 分钟后自动删除。

`Get(ctx, key).Bytes()` 适合读取 JSON 字节，再反序列化到结构体。

`redis.Nil` 表示 key 不存在。它不是系统异常，真实项目中通常叫“缓存未命中”。

Key 命名建议：

```text
业务域:对象类型:对象ID:字段或用途
```

例如：

```text
user:profile:1001
article:detail:9001
order:status:202607060001
```

## 四、真实后端场景示例

下面是一个常见的 Cache Aside 结构。为了聚焦 Redis，这里用函数模拟数据库：

```go
func GetUserProfile(ctx context.Context, rdb *redis.Client, userID int64) (UserProfile, error) {
	key := fmt.Sprintf("user:profile:%d", userID)

	cached, err := rdb.Get(ctx, key).Bytes()
	if err == nil {
		var profile UserProfile
		if err := json.Unmarshal(cached, &profile); err == nil {
			return profile, nil
		}
		_ = rdb.Del(ctx, key).Err()
	}
	if err != nil && !errors.Is(err, redis.Nil) {
		log.Printf("read redis failed: %v", err)
	}

	profile, err := queryUserProfileFromDB(ctx, userID)
	if err != nil {
		return UserProfile{}, err
	}

	raw, err := json.Marshal(profile)
	if err == nil {
		_ = rdb.Set(ctx, key, raw, 15*time.Minute).Err()
	}

	return profile, nil
}
```

真实项目中 `queryUserProfileFromDB` 应该使用参数化查询，不要拼接 SQL：

```text
用户输入不能直接拼进 SQL 字符串。
```

## 五、注意点

缓存 JSON 很方便，但也有版本问题。字段改名、类型变化、旧缓存还存在时，反序列化可能失败。保守做法是：反序列化失败就删除缓存，再回源数据库。

TTL 不建议所有 key 都一样。大量 key 同时过期，可能造成瞬时数据库压力。可以加一点随机抖动：

```go
ttl := 15*time.Minute + time.Duration(rand.Intn(120))*time.Second
```

如果用户资料刚更新，应该删除对应缓存，而不是只更新数据库。

## 六、常见误区

- 误区：缓存命中就永远可信。  
  原因：缓存可能是旧数据，尤其在更新后未删除缓存时。

- 误区：把完整用户隐私信息都放 Redis。  
  原因：缓存通常有更多访问路径，敏感字段要谨慎。

- 误区：反序列化失败直接返回 500。  
  原因：更好的方式是删除坏缓存，回源数据库恢复。

- 误区：不过期。  
  原因：没有 TTL 的缓存容易变成长期脏数据和内存负担。

## 七、本节达标标准

- 能用 String 缓存一个 Go 结构体。
- 能设置 TTL。
- 能正确处理 `redis.Nil`。
- 能解释 Cache Aside 的基本流程。
- 能说出缓存 JSON 的版本风险。
