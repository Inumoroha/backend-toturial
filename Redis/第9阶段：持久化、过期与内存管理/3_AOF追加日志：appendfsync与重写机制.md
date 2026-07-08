# 3. AOF 追加日志：appendfsync 与重写机制

AOF 是 Redis 另一种重要持久化方式。

RDB 保存某个时间点的快照，AOF 则记录写命令日志。Redis 重启时，通过重放 AOF 文件恢复数据。

学完这一节后，你应该能够：

- 理解 AOF 的工作方式。
- 配置 `appendonly yes`。
- 区分 `appendfsync always/everysec/no`。
- 理解 AOF 文件为什么会变大。
- 理解 AOF 重写的作用。
- 知道 AOF 的优点和风险。

---

## 一、AOF 是什么

AOF 全称：

```text
Append Only File
```

含义是追加日志文件。

当 Redis 执行写命令时，会把命令追加到 AOF 文件。

例如执行：

```redis
SET name redis
INCR counter
DEL old:key
```

AOF 文件中会记录这些写操作。

Redis 重启时重新执行这些命令，恢复内存数据。

---

## 二、开启 AOF

配置：

```conf
appendonly yes
```

默认文件名通常是：

```conf
appendfilename "appendonly.aof"
```

Redis 7 之后 AOF 文件组织方式可能包含基础文件和增量文件，具体和配置有关。

学习阶段只需要理解：

```text
AOF 保存的是写命令日志。
```

---

## 三、AOF 写入流程

简化流程：

```text
1. 客户端发送写命令。
2. Redis 执行命令，修改内存。
3. Redis 把命令写入 AOF 缓冲区。
4. 根据 appendfsync 策略刷到磁盘。
```

关键问题是：

```text
什么时候真正刷盘？
```

这由 `appendfsync` 决定。

---

## 四、appendfsync always

配置：

```conf
appendfsync always
```

含义：

```text
每次写命令都尽量刷盘。
```

优点：

- 数据安全性最好。

缺点：

- 性能最差。
- 磁盘延迟会直接影响 Redis 写入。

一般不推荐普通业务使用。

---

## 五、appendfsync everysec

配置：

```conf
appendfsync everysec
```

含义：

```text
大约每秒刷盘一次。
```

优点：

- 性能和安全性比较平衡。
- 最多通常丢最近 1 秒左右数据。

这是最常见的生产选择。

---

## 六、appendfsync no

配置：

```conf
appendfsync no
```

含义：

```text
由操作系统决定什么时候刷盘。
```

优点：

- Redis 自身开销小。

缺点：

- 故障时可能丢更多数据。
- 刷盘时机不可控。

一般不适合对数据恢复有要求的业务。

---

## 七、AOF 文件为什么会变大

假设执行：

```redis
INCR counter
INCR counter
INCR counter
```

AOF 会记录多条命令。

但最终数据只是：

```text
counter = 3
```

如果长期运行，AOF 文件会越来越大。

所以需要 AOF 重写。

---

## 八、AOF 重写是什么

AOF 重写不是简单压缩原文件。

它会根据当前内存数据生成一份更短的新 AOF。

例如原来：

```redis
SET name a
SET name b
SET name c
```

重写后可以变成：

```redis
SET name c
```

最终状态一样，但日志更短。

---

## 九、触发 AOF 重写

手动触发：

```redis
BGREWRITEAOF
```

配置自动触发：

```conf
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

含义：

```text
AOF 文件比上次重写后增长 100%，且至少达到 64MB，触发重写。
```

---

## 十、AOF 重写也会 fork

和 `BGSAVE` 类似，AOF 重写也会 fork 子进程。

简化流程：

```text
1. 主进程 fork 子进程。
2. 子进程根据当前数据生成新 AOF。
3. 重写期间主进程继续处理写入。
4. 新写入会记录到增量缓冲。
5. 重写完成后合并增量。
6. 新 AOF 替换旧 AOF。
```

所以 AOF 重写也有：

- fork 成本。
- COW 内存压力。
- 磁盘 IO 压力。

---

## 十一、查看 AOF 状态

```redis
INFO persistence
```

关注字段：

```text
aof_enabled
aof_rewrite_in_progress
aof_last_rewrite_time_sec
aof_last_bgrewrite_status
aof_current_size
aof_base_size
aof_last_cow_size
```

这些字段能帮助你判断：

- AOF 是否开启。
- 是否正在重写。
- 上次重写是否成功。
- 当前 AOF 文件大小。
- COW 占用情况。

---

## 十二、AOF 的优点

优点：

- 数据安全性通常比 RDB 好。
- `everysec` 策略下性能和安全性平衡。
- 文件是命令日志，理论上更容易分析。
- 可以结合混合持久化提升恢复速度。

AOF 更适合对数据恢复要求更高的场景。

---

## 十三、AOF 的缺点

缺点：

- 文件可能更大。
- 恢复时要重放命令，可能较慢。
- 需要重写机制。
- 重写期间有 fork、COW 和 IO 压力。
- 磁盘性能会影响 Redis。

如果磁盘慢，AOF 可能带来明显延迟抖动。

---

## 十四、RDB 和 AOF 可以一起开吗

可以。

常见配置：

```conf
save 900 1
save 300 10
save 60 10000
appendonly yes
appendfsync everysec
```

Redis 重启时通常优先使用 AOF 恢复，因为它数据更新。

生产中具体策略要结合业务。

---

## 十五、常见错误

### 1. 以为 AOF 不会丢数据

`everysec` 仍可能丢最近 1 秒左右数据。

### 2. 使用 `always` 后忽略性能影响

每次写都刷盘会显著降低写入性能。

### 3. AOF 文件无限增长

没有配置或关注重写机制。

### 4. 重写时不关注内存和磁盘 IO

重写期间可能出现延迟抖动。

### 5. 磁盘空间满了才发现

AOF 文件增长必须监控。

---

## 十六、本节练习

请完成下面练习：

1. 开启 `appendonly yes`。
2. 设置 `appendfsync everysec`。
3. 写入几个 key 后查看 AOF 文件。
4. 执行 `BGREWRITEAOF`。
5. 查看 `INFO persistence` 中 AOF 相关字段。
6. 比较 `always`、`everysec`、`no` 的区别。
7. 思考你的业务能否接受丢最近 1 秒数据。

---

## 十七、本节小结

这一节你学习了 AOF。

你需要记住：

- AOF 通过追加写命令日志恢复数据。
- `appendfsync everysec` 是常见平衡选择。
- AOF 文件会增长，需要重写。
- AOF 重写也会 fork，也有 COW 和 IO 压力。
- AOF 数据安全性通常比 RDB 好，但也不是绝对不丢。

下一节我们学习混合持久化：为什么它能兼顾恢复速度和数据安全。

