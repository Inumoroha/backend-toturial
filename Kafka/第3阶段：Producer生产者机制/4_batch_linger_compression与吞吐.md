# 4. batch、linger、compression 与吞吐

本节目标：理解 producer 如何通过批量发送和压缩提升吞吐，知道 batch、linger、compression 参数分别影响什么，以及如何在 Go 后端中按业务类型做取舍。

Kafka 高吞吐的一个重要原因是批量。Producer 往往不是每来一条消息就立刻发一个网络请求，而是把多条消息组成 batch 再发送。这样可以减少网络请求次数，提高磁盘和网络利用率。

---

## 一、吞吐和延迟的基本取舍

调 producer 参数前，先问：

```text
我更关心吞吐，还是更关心延迟？
```

高吞吐通常意味着：

- 批量更大。
- 等待时间稍长。
- 压缩更积极。

低延迟通常意味着：

- 尽快发送。
- 批量等待少。
- 单条消息更快返回。

没有一套配置适合所有业务。

---

## 二、batch 是什么

Producer 会按 topic-partition 组织 batch。

例如：

```text
order.created partition-0 batch
order.created partition-1 batch
order.created partition-2 batch
```

同一个 batch 中包含多条消息。

batch 的好处：

- 减少网络请求。
- 减少协议开销。
- 提高压缩效果。
- 提高 broker 顺序写效率。

---

## 三、batch.size 的直觉

`batch.size` 控制 batch 的目标大小。

增大 batch size 可能带来：

- 吞吐提升。
- 压缩效果提升。
- producer 内存占用增加。
- 单条消息等待时间可能增加。

如果消息量很小，batch size 设置很大也不一定能凑满。

所以 batch size 要和实际 QPS、消息大小一起看。

---

## 四、linger.ms 的直觉

`linger.ms` 表示 producer 为了凑 batch，最多愿意等多久。

例如：

```text
linger.ms=20
```

含义：

```text
如果 batch 没满，最多等 20ms，看是否有更多消息进来
```

优点：

- 更容易形成 batch。
- 提高吞吐。
- 压缩更有效。

缺点：

- 增加端到端延迟。

对于订单创建这类接口，linger 不宜过大。

对于日志采集，linger 可以适当大一些。

---

## 五、compression.type

常见压缩算法：

```text
none
snappy
lz4
zstd
gzip
```

压缩的好处：

- 减少网络传输。
- 减少 broker 磁盘占用。
- batch 越大，压缩效果通常越好。

代价：

- producer CPU 增加。
- consumer 解压 CPU 增加。

常见选择：

- `lz4`：速度快，适合低延迟和较高吞吐。
- `zstd`：压缩率好，适合消息较大、带宽或磁盘敏感场景。
- `snappy`：老牌选择，速度和压缩率较均衡。

---

## 六、消息大小对压缩的影响

小消息：

```text
压缩收益可能有限
```

大 JSON 消息：

```text
字段重复多，压缩收益明显
```

例如订单事件：

```json
{
  "event_id": "...",
  "event_type": "order.created",
  "version": 1,
  "occurred_at": "...",
  "data": {
    "order_id": "...",
    "user_id": "...",
    "items": []
  }
}
```

字段名重复，适合压缩。

---

## 七、buffer 相关直觉

Producer 本地会有缓冲区。

如果 broker 慢、网络慢、消息产生太快，buffer 可能堆积。

可能结果：

- send 阻塞。
- send 超时。
- 返回 buffer full 错误。
- 进程内存压力增大。

Go 后端要监控 producer 发送错误和延迟。

---

## 八、不同业务的配置取舍

### 1. 订单事件

目标：

```text
可靠性优先，延迟也要可控
```

倾向：

```text
acks=all
enable.idempotence=true
linger.ms 小到中等
compression=lz4 或 zstd
```

### 2. 行为日志

目标：

```text
吞吐优先，可接受少量延迟
```

倾向：

```text
batch 更大
linger.ms 更大
compression=lz4/zstd
```

### 3. 低延迟通知事件

目标：

```text
尽快投递
```

倾向：

```text
linger.ms 较小
batch 不宜过大
compression 视消息大小决定
```

---

## 九、Go 后端代码中的超时

即使 producer 内部有超时，业务代码仍建议使用 context。

```go
ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
defer cancel()

err := producer.Publish(ctx, "order.created", msg)
```

如果超时：

- 记录日志。
- 返回错误或进入 outbox 重试。
- 不要让 HTTP 请求无限等待。

---

## 十、压测时看什么

调 batch、linger、compression 时，要记录：

- 每秒消息数。
- 每秒字节数。
- 平均延迟。
- P95 延迟。
- P99 延迟。
- producer 错误数。
- CPU 使用率。
- broker bytes in。

只看平均延迟不够。

如果 P99 很高，用户仍然可能感知到卡顿。

---

## 十一、实验建议

准备三种消息大小：

```text
1KB
10KB
100KB
```

分别测试：

```text
compression=none
compression=lz4
compression=zstd
```

记录吞吐和延迟。

你会发现：

- 消息越大，压缩收益越明显。
- 压缩会消耗 CPU。
- linger 增大可能提升吞吐但增加延迟。

---

## 十二、常见误区

### 1. batch 越大越好

不是。

batch 太大可能增加延迟和内存压力。

### 2. linger 越大越好

不是。

linger 过大会让消息等待太久。

### 3. 开压缩一定更快

不一定。

压缩减少网络和磁盘，但增加 CPU。

### 4. 所有 topic 用同一套 producer 配置

不推荐。

订单事件、日志事件、监控事件的目标不同。

---

## 十三、本节练习

1. 解释 batch 和 linger 的关系。
2. 为什么 linger 会增加延迟？
3. 为什么压缩对大 JSON 消息更有效？
4. 为订单事件设计一套 producer 参数。
5. 为用户行为日志设计一套 producer 参数。
6. 写一个压测表格模板，记录吞吐、P95、P99、错误率。

---

## 十四、本节小结

- Producer 通过 batch 提高吞吐。
- `batch.size` 影响批次大小和内存使用。
- `linger.ms` 影响等待凑批时间。
- 压缩能降低网络和磁盘压力，但会消耗 CPU。
- 不同业务需要不同 producer 配置。
- 调优必须基于压测数据。
- Go 后端仍然需要 context timeout 控制发送时间。

---

## 十五、参数选择示例

订单事件：

```text
batch: 中等
linger: 5ms 左右
compression: lz4
目标：可靠性优先，兼顾吞吐
```

用户行为日志：

```text
batch: 较大
linger: 20ms-50ms
compression: zstd 或 lz4
目标：吞吐优先，允许轻微延迟
```

同样是 Kafka producer，不同业务的参数取舍完全不同。

---

## 十六、最终检查

调参前先写目标：

```text
我要提升吞吐，允许 P99 增加到 100ms。
```

没有目标就调参，很容易越调越乱。

---

## 十七、压测记录模板

```text
batch.size:
linger.ms:
compression:
message_size:
message_count:
throughput:
p95:
p99:
producer_cpu:
broker_network_in:
```

每次只改一个参数，才能判断它对吞吐或延迟的影响。

---

## 十八、最终验收

完成本节后，给出两组压测结论：

```text
低延迟配置：吞吐、P95、P99。
高吞吐配置：吞吐、P95、P99。
```

能说清差异，才算理解 batch、linger 和 compression。

---

## 二十一、吞吐调优记录模板

每次调整参数，都建议记录成表格：

| 配置 | 吞吐 | P95 | P99 | CPU | 网络流量 | 结论 |
| --- | --- | --- | --- | --- | --- | --- |
| baseline | | | | | | |
| 增大 batch | | | | | | |
| 增大 linger | | | | | | |
| 开启 compression | | | | | | |

不要只记录吞吐。Kafka 调优常见的取舍是：吞吐上升，但尾延迟或 CPU 成本也上升。

后端工程师需要给业务一个能解释的结论，而不是只说“我把参数调大了”。
