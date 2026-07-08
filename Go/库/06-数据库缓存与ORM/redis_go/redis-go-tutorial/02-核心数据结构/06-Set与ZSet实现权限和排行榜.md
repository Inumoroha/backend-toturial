# 6. Set 与 ZSet 实现权限和排行榜

本节目标：你将能用 Set 管理用户权限集合，用 ZSet 实现热门文章排行榜。

简短引入：Set 和 ZSet 在后端业务中非常实用。Set 解决“某个元素是否属于集合”，ZSet 解决“按分数排序”。

## 一、为什么需要它

权限判断、点赞去重、黑名单、关注关系都可以用 Set 表达。

排行榜、热度列表、延迟任务时间排序可以用 ZSet 表达。

可以理解为：

- Set 关心有没有。
- ZSet 关心有没有，以及排第几。

## 二、基本用法

Set 示例：用户权限判断。

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/redis/go-redis/v9"
)

func main() {
	ctx := context.Background()
	rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
	defer rdb.Close()

	key := "user:permissions:1001"

	if err := rdb.SAdd(ctx, key, "article:read", "article:write").Err(); err != nil {
		log.Fatal(err)
	}

	ok, err := rdb.SIsMember(ctx, key, "article:write").Result()
	if err != nil {
		log.Fatal(err)
	}

	fmt.Println("can write article:", ok)
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

Set 常用命令：

- `SAdd`：加入元素。
- `SRem`：移除元素。
- `SIsMember`：判断是否存在。
- `SMembers`：读取所有元素。

ZSet 常用命令：

- `ZAdd`：加入成员和分数。
- `ZIncrBy`：增加分数。
- `ZRevRangeWithScores`：按分数从高到低读取。

ZSet 示例：

```go
key := "rank:article:hot"

_ = rdb.ZIncrBy(ctx, key, 1, "article:9001").Err()
_ = rdb.ZIncrBy(ctx, key, 3, "article:9002").Err()

items, err := rdb.ZRevRangeWithScores(ctx, key, 0, 9).Result()
if err != nil {
	log.Fatal(err)
}

for _, item := range items {
	fmt.Println(item.Member, item.Score)
}
```

## 四、真实后端场景示例

文章点赞可以用 Set 去重：

```go
func LikeArticle(ctx context.Context, rdb *redis.Client, articleID, userID int64) (bool, error) {
	likeKey := fmt.Sprintf("article:likes:%d", articleID)
	rankKey := "rank:article:hot"
	member := fmt.Sprintf("%d", userID)

	added, err := rdb.SAdd(ctx, likeKey, member).Result()
	if err != nil {
		return false, err
	}
	if added == 0 {
		return false, nil
	}

	articleMember := fmt.Sprintf("article:%d", articleID)
	if err := rdb.ZIncrBy(ctx, rankKey, 10, articleMember).Err(); err != nil {
		return false, err
	}

	return true, nil
}
```

这里 `SAdd` 返回 1 表示新点赞，返回 0 表示用户已经点过赞。只有新点赞才增加热度。

真实项目里，还要考虑数据库点赞记录。Redis 可以用于快速判断和热榜，但重要行为通常要落库，便于审计和恢复。

## 五、注意点

Set 成员过多时，`SMembers` 会一次性返回所有元素，可能造成内存和网络压力。生产中要谨慎使用，必要时用 `SScan` 分批扫描。

ZSet 排行榜需要清理旧数据。例如只保留最近一周的热度，或者定时删除低分成员。

权限缓存要有失效策略。管理员修改权限后，应该删除或刷新对应用户权限 key。

## 六、常见误区

- 误区：用 Set 保存无限增长的行为记录。  
  原因：Redis 内存有限，长期行为记录应落数据库或日志系统。

- 误区：点赞只写 Redis 不写数据库。  
  原因：Redis 更适合加速，不适合单独承担长期事实记录。

- 误区：排行榜从不清理。  
  原因：成员越积越多，查询和内存成本都会增加。

- 误区：权限变更后不删除缓存。  
  原因：用户可能继续拿到旧权限。

## 七、本节达标标准

- 能用 Set 判断权限和点赞去重。
- 能用 ZSet 实现热门文章排行。
- 能解释 Set 与 ZSet 的区别。
- 能说出 `SMembers` 的风险。
- 能设计权限缓存失效策略。

