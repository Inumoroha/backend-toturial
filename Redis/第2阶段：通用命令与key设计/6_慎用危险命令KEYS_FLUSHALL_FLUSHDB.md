# 6. 慎用危险命令：KEYS *、FLUSHALL、FLUSHDB

Redis 中有一些命令非常方便，也非常危险。

这一节专门讲 `KEYS *`、`FLUSHALL`、`FLUSHDB`。学习它们不是为了经常使用，而是为了知道它们会造成什么影响，避免在错误环境中误操作。

学完这一节后，你应该能够：

- 理解为什么生产环境慎用 `KEYS *`。
- 区分 `FLUSHDB` 和 `FLUSHALL`。
- 知道这些命令可能造成的数据和性能风险。
- 在本地学习环境安全地练习这些命令。
- 建立生产环境命令操作的边界感。

---

## 一、先确认环境

这一节会涉及删除数据的命令。

请只在本地学习 Redis 中练习，不要连接公司、线上、共享测试环境执行。

你可以单独启动一个练习容器：

```bash
docker run -d --name redis-danger-demo -p 6380:6379 redis:7-alpine
```

进入这个练习容器：

```bash
docker exec -it redis-danger-demo redis-cli
```

确认端口和容器名，避免误连其他 Redis。

---

## 二、KEYS：按模式查找 key

`KEYS` 用来查找符合模式的 key。

示例：

```redis
KEYS *
```

表示查找所有 key。

```redis
KEYS user:*
```

表示查找所有以 `user:` 开头的 key。

本地准备数据：

```redis
MSET user:1 tom user:2 jack article:1 redis article:2 mysql
```

执行：

```redis
KEYS user:*
```

可能返回：

```text
1) "user:1"
2) "user:2"
```

看起来很方便，但问题也在这里：太方便了。

---

## 三、为什么 KEYS * 危险

`KEYS *` 会遍历整个 key 空间。

如果 Redis 里只有几十个 key，没有明显问题。

如果 Redis 里有几百万个 key，它可能执行很久。在执行期间，Redis 还要处理其他客户端请求，慢命令会造成请求排队。

可能影响：

- 接口响应变慢。
- Redis CPU 升高。
- 客户端超时。
- 其他正常命令被拖住。
- 大量结果返回造成网络压力。

所以生产环境不要随手执行：

```redis
KEYS *
```

更推荐：

```redis
SCAN 0 MATCH user:* COUNT 100
```

---

## 四、KEYS 能不能用

可以，但要分环境。

适合使用 `KEYS` 的情况：

- 本地学习环境。
- 临时测试 Redis。
- 确认 key 数量非常少。
- 一次性小范围调试。

不适合使用 `KEYS` 的情况：

- 生产环境。
- key 数量未知。
- 高峰期。
- 共享 Redis。
- 连接的数据库不是自己独占。

一个实用习惯是：只要你不能确定这个 Redis 里有多少 key，就不要用 `KEYS *`。

---

## 五、FLUSHDB：清空当前数据库

Redis 默认有多个逻辑数据库，默认连接的是 0 号数据库。

`FLUSHDB` 会清空当前数据库中的所有 key。

示例：

```redis
SELECT 0
MSET a 1 b 2 c 3
DBSIZE
FLUSHDB
DBSIZE
```

`DBSIZE` 用来查看当前数据库 key 数量。

执行 `FLUSHDB` 后，当前数据库所有 key 都会被删除。

这不是删除某个前缀，也不是删除某个业务模块，而是清空整个当前 DB。

---

## 六、FLUSHALL：清空所有数据库

`FLUSHALL` 比 `FLUSHDB` 更危险。

它会清空 Redis 实例中所有逻辑数据库的 key。

示例：

```redis
FLUSHALL
```

如果这个 Redis 实例被多个项目、多个环境、多个逻辑数据库共用，`FLUSHALL` 会影响全部。

所以它是非常危险的命令。

生产环境中通常会通过 ACL、命令重命名、权限隔离等方式限制这类命令。

---

## 七、FLUSHDB 和 FLUSHALL 的区别

| 命令 | 影响范围 | 危险程度 |
| --- | --- | --- |
| `FLUSHDB` | 当前选中的数据库 | 很危险 |
| `FLUSHALL` | 当前 Redis 实例的所有数据库 | 极其危险 |

如果你不知道当前连接的是哪个 Redis、哪个 DB，不要执行它们。

如果你不知道是否有人共用这个 Redis，也不要执行它们。

---

## 八、异步清理：ASYNC

Redis 支持异步清理：

```redis
FLUSHDB ASYNC
FLUSHALL ASYNC
```

`ASYNC` 可以降低同步删除造成的阻塞影响，但它不降低数据删除的业务风险。

也就是说：

```text
ASYNC 只改变删除方式，不改变删除范围。
```

删错了还是删错了。

所以不要因为有 `ASYNC` 就放松警惕。

---

## 九、生产环境如何降低误操作风险

常见做法包括：

- 不直接开放生产 Redis 给个人电脑。
- 使用只读账号查看数据。
- 通过 ACL 禁止危险命令。
- 给不同环境使用不同 Redis 实例。
- 不让多个项目随意共用一个 Redis DB。
- 操作前确认 host、port、database、环境名。
- 批量删除前先 `SCAN` 预览 key 范围。
- 高危操作走审批或脚本化流程。

Redis 命令很快，误操作也很快。

越是简单的命令，越需要边界。

---

## 十、如果要删除某类 key，应该怎么做

不要用：

```redis
KEYS user:* 然后一次性 DEL
```

更稳妥的思路是：

1. 使用 `SCAN` 分批查找。
2. 先打印或抽样确认 key 范围。
3. 分批删除。
4. 控制每批大小。
5. 大 key 使用 `UNLINK` 而不是 `DEL`。
6. 避开业务高峰期。

学习阶段可以先理解流程。后续生产排查阶段会更详细地讲 Big Key、`UNLINK`、慢日志和清理策略。

---

## 十一、本地安全练习

请只在本地练习容器中操作。

写入数据：

```redis
MSET user:1 tom user:2 jack article:1 redis article:2 mysql
DBSIZE
```

查看 key：

```redis
KEYS *
KEYS user:*
```

清空当前数据库：

```redis
FLUSHDB
DBSIZE
```

再次写入：

```redis
MSET a 1 b 2
```

清空所有数据库：

```redis
FLUSHALL
DBSIZE
```

练习完成后，可以删除练习容器：

```bash
docker rm -f redis-danger-demo
```

注意：这是本地练习容器，删除没关系。不要对真实业务容器这么做。

---

## 十二、常见错误

### 1. 在生产环境执行 KEYS *

可能造成 Redis 阻塞和接口超时。

### 2. 以为 FLUSHDB 只删除某类 key

`FLUSHDB` 删除的是当前数据库所有 key。

### 3. 以为 FLUSHALL 只影响当前 DB

`FLUSHALL` 删除的是所有数据库。

### 4. 连接错环境

本来想清空本地 Redis，结果连到了测试或生产 Redis。

### 5. 没确认 key 范围就批量删除

前缀写错可能删掉不该删的数据。

---

## 十三、本节小结

这一节你认识了几个危险命令：

- `KEYS *`：全库扫描 key，生产环境慎用。
- `FLUSHDB`：清空当前数据库。
- `FLUSHALL`：清空整个 Redis 实例所有数据库。
- `ASYNC` 可以异步清理，但不能降低删错数据的业务风险。
- 线上查 key 更推荐 `SCAN`。
- 批量删除前必须确认范围、环境和影响。

下一节我们会进入第二阶段最重要的部分：key 命名规范、value 大小控制和 TTL 规划。
