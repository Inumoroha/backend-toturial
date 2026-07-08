# 2. Bitmap：签到、活跃用户、布尔状态压缩

Bitmap 不是 Redis 独立的数据类型，而是基于 String 的位操作能力。

它适合保存大量“是/否”状态，比如用户是否签到、某天是否活跃、某个任务是否完成。

学完这一节后，你应该能够：

- 理解 Bitmap 的基本思想。
- 使用 `SETBIT`、`GETBIT`、`BITCOUNT`。
- 用 Bitmap 实现用户签到。
- 用 Bitmap 统计每日活跃用户。
- 设计合理的 offset。
- 知道 Bitmap 的限制和风险。

---

## 一、Bitmap 是什么

Bitmap 可以理解为一串 bit：

```text
offset: 0 1 2 3 4 5 6 7
value:  0 1 0 1 1 0 0 1
```

每个位置只保存 0 或 1。

Redis 命令：

```redis
SETBIT key offset 1
GETBIT key offset
BITCOUNT key
```

一个 bit 只占 1 位，所以非常省内存。

---

## 二、用户签到设计

需求：

```text
记录用户 2026 年 7 月每天是否签到。
```

key：

```text
sign:user:1001:202607
```

offset：

```text
日期 - 1
```

例如：

```text
7 月 1 日 -> offset 0
7 月 2 日 -> offset 1
7 月 31 日 -> offset 30
```

签到：

```redis
SETBIT sign:user:1001:202607 4 1
```

表示 7 月 5 日签到。

---

## 三、查询某天是否签到

```redis
GETBIT sign:user:1001:202607 4
```

返回：

```text
1：已签到
0：未签到
```

Go 示例：

```go
func SignKey(userID int64, now time.Time) string {
    return fmt.Sprintf("sign:user:%d:%s", userID, now.Format("200601"))
}

func SignOffset(day int) int64 {
    return int64(day - 1)
}
```

---

## 四、Go 实现签到

```go
func SignIn(ctx context.Context, rdb *redis.Client, userID int64, now time.Time) error {
    key := SignKey(userID, now)
    offset := SignOffset(now.Day())

    return rdb.SetBit(ctx, key, offset, 1).Err()
}
```

查询：

```go
func IsSigned(ctx context.Context, rdb *redis.Client, userID int64, day time.Time) (bool, error) {
    key := SignKey(userID, day)
    offset := SignOffset(day.Day())

    v, err := rdb.GetBit(ctx, key, offset).Result()
    if err != nil {
        return false, err
    }

    return v == 1, nil
}
```

---

## 五、统计本月签到天数

使用：

```redis
BITCOUNT sign:user:1001:202607
```

Go 示例：

```go
func CountMonthlySign(ctx context.Context, rdb *redis.Client, userID int64, month time.Time) (int64, error) {
    key := SignKey(userID, month)
    return rdb.BitCount(ctx, key, nil).Result()
}
```

`BITCOUNT` 会统计 key 中 bit 为 1 的数量。

---

## 六、连续签到怎么做

连续签到可以读取本月 bitmap 后在应用层计算。

Redis 命令：

```redis
GET sign:user:1001:202607
```

然后在 Go 中解析 bit。

教学阶段可以先简单循环 `GETBIT`：

```go
func CurrentStreak(ctx context.Context, rdb *redis.Client, userID int64, now time.Time) (int, error) {
    key := SignKey(userID, now)
    streak := 0

    for day := now.Day(); day >= 1; day-- {
        v, err := rdb.GetBit(ctx, key, int64(day-1)).Result()
        if err != nil {
            return 0, err
        }
        if v == 0 {
            break
        }
        streak++
    }

    return streak, nil
}
```

生产中可以减少多次 Redis 往返。

---

## 七、每日活跃用户

需求：

```text
记录某天哪些用户活跃。
```

key：

```text
active:user:20260705
```

offset：

```text
user_id
```

用户 1001 活跃：

```redis
SETBIT active:user:20260705 1001 1
```

统计当天活跃用户数：

```redis
BITCOUNT active:user:20260705
```

---

## 八、offset 不能乱用

Bitmap 最大的设计点是 offset。

如果 user_id 很大：

```text
user_id = 9000000000
```

直接作为 offset 会让 Redis 分配很大的 String 空间。

这非常危险。

更稳妥的方式：

- 确认 user_id 是紧凑递增的。
- 使用内部连续 ID。
- 按分片拆 key。
- 如果 ID 很稀疏，考虑 Set 或 HyperLogLog。

Bitmap 适合 offset 相对连续的场景。

---

## 九、按分片拆 Bitmap

如果用户量很大，可以按分片：

```text
active:user:20260705:shard:0
active:user:20260705:shard:1
```

计算：

```text
shard = user_id / 1000000
offset = user_id % 1000000
```

这样每个 key 最大只覆盖 100 万 bit。

统计总活跃时，累加每个 shard 的 `BITCOUNT`。

---

## 十、Bitmap 运算

Redis 支持位运算：

```redis
BITOP AND active:both active:user:20260705 active:user:20260706
BITOP OR active:any active:user:20260705 active:user:20260706
```

用途：

- 两天都活跃的用户数。
- 一周内任意一天活跃的用户数。
- 多条件布尔集合交并。

执行后再：

```redis
BITCOUNT active:any
```

可以得到数量。

注意：`BITOP` 会生成新 key，要设置 TTL 或及时清理。

---

## 十一、Bitmap 和 Set 的选择

| 场景 | 推荐 |
| --- | --- |
| 用户 ID 连续，保存是否活跃 | Bitmap |
| 用户 ID 稀疏，且需要成员列表 | Set |
| 只需要 UV 估算数量 | HyperLogLog |
| 要做交集并集并保留成员 | Set |

Bitmap 的优势是省内存。

代价是 offset 设计要求更高，且不方便直接列出所有成员。

---

## 十二、常见错误

### 1. 直接用很大的稀疏 ID 作为 offset

会造成巨大内存浪费。

### 2. offset 规则改来改去

旧数据会无法解释。

### 3. 用 Bitmap 后又要频繁列出所有用户

Bitmap 更适合统计状态，不适合成员列表查询。

### 4. `BITOP` 结果 key 不清理

临时结果可能长期占用内存。

### 5. 把 Bitmap 当成精确数据库

重要业务仍要考虑持久化、回源和对账。

---

## 十三、本节练习

请完成下面练习：

1. 为用户月签到设计 key。
2. 计算 7 月 5 日对应的 offset。
3. 写出签到命令。
4. 写出查询某天是否签到的命令。
5. 写出统计本月签到天数的命令。
6. 为每日活跃用户设计 key。
7. 思考 user_id 很大时能否直接作为 offset。
8. 用 `BITOP` 统计两天都活跃的用户。

---

## 十四、本节小结

这一节你学习了 Bitmap。

你需要记住：

- Bitmap 适合大量布尔状态。
- `SETBIT` 设置状态，`GETBIT` 查询状态，`BITCOUNT` 统计数量。
- 用户签到可以用月份 key + 日期 offset。
- 活跃用户可以用日期 key + 用户 offset。
- offset 必须设计稳定，避免大而稀疏。
- Bitmap 省内存，但不适合频繁列出成员。

下一节我们学习 HyperLogLog，用它做允许误差的 UV 统计。

