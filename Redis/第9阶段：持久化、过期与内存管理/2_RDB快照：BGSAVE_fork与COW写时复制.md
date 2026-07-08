# 2. RDB 快照：BGSAVE、fork 与 COW 写时复制

RDB 是 Redis 持久化方式之一。

它通过生成快照文件，把某个时间点的内存数据保存到磁盘。

学完这一节后，你应该能够：

- 理解 RDB 快照的工作方式。
- 区分 `SAVE` 和 `BGSAVE`。
- 理解 `fork` 和 COW 写时复制。
- 知道 RDB 的优点和缺点。
- 知道什么时候适合使用 RDB。

---

## 一、RDB 是什么

RDB 可以理解为：

```text
把 Redis 当前内存数据保存成一个二进制快照文件。
```

默认文件名通常是：

```text
dump.rdb
```

Redis 重启时，可以从 RDB 文件恢复数据。

---

## 二、生成 RDB 的方式

手动生成：

```redis
BGSAVE
```

也可以在配置中设置自动保存：

```conf
save 900 1
save 300 10
save 60 10000
```

含义：

```text
900 秒内至少 1 次写入，触发保存
300 秒内至少 10 次写入，触发保存
60 秒内至少 10000 次写入，触发保存
```

---

## 三、SAVE 和 BGSAVE

`SAVE`：

```redis
SAVE
```

会在 Redis 主进程中同步生成 RDB。

生成期间 Redis 不能处理其他命令。

生产环境不要随便使用。

`BGSAVE`：

```redis
BGSAVE
```

Redis 会 fork 一个子进程，由子进程生成 RDB。

主进程继续处理请求。

生产中更常见的是 `BGSAVE`。

---

## 四、BGSAVE 基本流程

简化流程：

```text
1. Redis 主进程 fork 出子进程。
2. 子进程读取内存数据，写入临时 RDB 文件。
3. 写入完成后，用新 RDB 文件替换旧文件。
4. 子进程退出。
5. 主进程继续处理客户端请求。
```

这就是为什么 RDB 生成期间 Redis 仍然能响应请求。

但 `fork` 本身可能造成短暂阻塞。

---

## 五、fork 是什么

`fork` 是操作系统创建子进程的机制。

Redis fork 子进程时，子进程看起来拥有和父进程一样的内存数据。

但它不会立刻复制所有内存。

操作系统使用 COW：

```text
Copy On Write，写时复制。
```

只有当父进程修改某些内存页时，相关内存页才会被复制。

---

## 六、COW 写时复制

假设 Redis 内存中有数据：

```text
A B C D
```

执行 `BGSAVE` 后：

```text
子进程看到的是快照时刻的 A B C D
主进程继续处理写请求
```

如果主进程把 A 修改成 A2：

```text
操作系统复制 A 所在内存页
主进程修改复制后的页
子进程仍看到旧 A
```

这样子进程可以生成一致的快照。

---

## 七、COW 的内存风险

`BGSAVE` 期间，如果写入很多，COW 会复制很多内存页。

这会导致：

- 内存使用量临时升高。
- 机器内存压力变大。
- 严重时触发 OOM。

所以 Redis 内存已经很满时，执行 `BGSAVE` 有风险。

生产中要关注：

```redis
INFO persistence
INFO memory
```

---

## 八、RDB 的优点

优点：

- 文件紧凑。
- 适合定期备份。
- 恢复速度通常较快。
- 对运行时写入影响相对可控。
- 可以把 RDB 文件复制到其他机器做冷备。

如果 Redis 主要作为缓存，RDB 往往足够。

因为缓存本身允许重建。

---

## 九、RDB 的缺点

缺点：

- 两次快照之间的数据可能丢失。
- `fork` 可能造成短暂阻塞。
- `BGSAVE` 期间 COW 会增加内存压力。
- 快照频率越高，磁盘和 CPU 压力越大。
- 快照频率越低，故障时丢数据越多。

这是一个典型取舍：

```text
更频繁快照 -> 更少丢数据，但更大运行压力
更少快照 -> 更小运行压力，但可能丢更多数据
```

---

## 十、查看 RDB 状态

```redis
INFO persistence
```

关注字段：

```text
rdb_bgsave_in_progress
rdb_last_save_time
rdb_last_bgsave_status
rdb_last_bgsave_time_sec
rdb_last_cow_size
```

含义：

- 是否正在执行 `BGSAVE`。
- 上次保存时间。
- 上次保存是否成功。
- 上次保存耗时。
- 上次 COW 使用内存大小。

---

## 十一、Docker 中观察 RDB

如果使用 Docker：

```bash
docker exec -it redis redis-cli
```

执行：

```redis
SET name redis
BGSAVE
INFO persistence
```

查看容器内文件：

```bash
docker exec -it redis sh
ls -lh /data
```

通常可以看到：

```text
dump.rdb
```

具体路径取决于 Redis 镜像和配置。

---

## 十二、什么时候适合 RDB

适合：

- Redis 主要做缓存。
- 数据可以从数据库重建。
- 想要定期备份。
- 可以接受分钟级数据丢失。
- 更看重恢复速度和文件紧凑。

不适合单独依赖：

- 重要业务状态。
- 不能接受最近数据丢失。
- 需要强恢复保障的系统。

重要业务通常要结合 AOF 或数据库。

---

## 十三、常见错误

### 1. 生产执行 SAVE

会阻塞 Redis 主进程。

### 2. 以为 BGSAVE 完全无影响

`fork` 和 COW 仍然有成本。

### 3. 内存很满时执行 BGSAVE

COW 可能导致内存压力进一步升高。

### 4. 快照间隔太长但又不能丢数据

RDB 可能丢失两次快照之间的数据。

### 5. 从不检查 RDB 状态

持久化失败很久可能没人知道。

---

## 十四、本节练习

请完成下面练习：

1. 执行 `BGSAVE` 生成 RDB。
2. 查看 `INFO persistence`。
3. 找到 `rdb_last_bgsave_status`。
4. 解释 `SAVE` 为什么危险。
5. 用自己的话解释 COW。
6. 思考为什么 `BGSAVE` 期间写入越多内存压力越大。
7. 判断你的 Redis 数据是否能只靠 RDB 恢复。

---

## 十五、本节小结

这一节你学习了 RDB。

你需要记住：

- RDB 是 Redis 某个时间点的数据快照。
- `SAVE` 会阻塞主进程，生产中慎用。
- `BGSAVE` 通过 fork 子进程生成快照。
- COW 写时复制让子进程看到一致快照，但会带来额外内存压力。
- RDB 文件紧凑、恢复快，但可能丢失两次快照之间的数据。

下一节我们学习 AOF：追加日志、刷盘策略和 AOF 重写。

