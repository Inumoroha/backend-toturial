# 4. Hash 保存对象字段

本节目标：你将能用 Redis Hash 保存对象的多个字段，并判断 Hash 与 JSON String 的使用边界。

简短引入：Hash 很适合保存“对象字段”。比如用户资料里昵称、头像、等级经常单独更新，Hash 可以避免每次都读写整段 JSON。

## 一、为什么需要它

如果用户资料只有整体读写，String + JSON 很简单。但有些业务经常只改一个字段：

- 用户改昵称。
- 文章阅读数增加。
- 订单状态更新。
- 商品库存展示字段刷新。

这时 Hash 可以理解为 Redis 里的一个小型字段表。

```text
Hash 适合字段级读写，但不适合替代关系型数据库的复杂查询。
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

	key := "user:summary:1001"

	if err := rdb.HSet(ctx, key, map[string]any{
		"nickname": "小林",
		"avatar":   "https://example.com/a.png",
		"level":    3,
	}).Err(); err != nil {
		log.Fatal(err)
	}

	_ = rdb.Expire(ctx, key, 30*time.Minute).Err()

	nickname, err := rdb.HGet(ctx, key, "nickname").Result()
	if err != nil {
		log.Fatal(err)
	}

	fmt.Println("nickname:", nickname)
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

`HSet` 可以一次设置多个字段。字段值最终会按 Redis 能存储的字符串形式保存。

`HGet` 读取单个字段。

`HGetAll` 读取整个 Hash。

`HIncrBy` 可以对整数字段做原子递增，例如文章阅读数。

注意：`HSet` 本身不会自动设置过期时间，所以通常要配合 `Expire`。

## 四、真实后端场景示例

文章摘要缓存可以用 Hash：

```go
func CacheArticleSummary(ctx context.Context, rdb *redis.Client, articleID int64) error {
	key := fmt.Sprintf("article:summary:%d", articleID)

	err := rdb.HSet(ctx, key, map[string]any{
		"title":      "Go Redis 实战",
		"author_id":  1001,
		"status":     "published",
		"view_count": 0,
	}).Err()
	if err != nil {
		return err
	}

	return rdb.Expire(ctx, key, 20*time.Minute).Err()
}

func IncreaseArticleView(ctx context.Context, rdb *redis.Client, articleID int64) error {
	key := fmt.Sprintf("article:summary:%d", articleID)
	return rdb.HIncrBy(ctx, key, "view_count", 1).Err()
}
```

真实项目里，阅读数是否每次都同步数据库，要看业务要求。常见做法是 Redis 先累加，再定时批量落库。但这会引入丢失风险，需要日志、补偿或更可靠的消息机制。

## 五、注意点

Hash 不适合做复杂筛选。比如“查询所有 level=3 的用户”，不要扫描 Redis Hash 来做。这类查询应该由数据库索引完成。

如果对象字段很多，且每个字段都要频繁变更，Hash 会比 JSON String 更灵活。但如果对象经常整体读取，JSON String 更简单。

更新数据库后，如果 Redis Hash 存的是缓存副本，通常删除 key，让下次读取重新构建。

## 六、常见误区

- 误区：用 Hash 模拟数据库表。  
  原因：Redis 没有关系型查询能力，也不适合承担复杂条件检索。

- 误区：只 `HSet` 不设置过期时间。  
  原因：缓存长期存在后可能变旧，也占用内存。

- 误区：阅读数只存在 Redis。  
  原因：Redis 故障或淘汰策略可能导致数据丢失，重要数据要有落库策略。

- 误区：以为 Hash 字段可以单独设置 TTL。  
  原因：Redis TTL 是 key 级别，不是 Hash 字段级别。

## 七、本节达标标准

- 能用 Hash 保存对象字段。
- 能读取单个字段和整个 Hash。
- 能使用 `HIncrBy` 做计数。
- 能说明 Hash 与 JSON String 的取舍。
- 能知道 Hash 不适合替代数据库查询。

