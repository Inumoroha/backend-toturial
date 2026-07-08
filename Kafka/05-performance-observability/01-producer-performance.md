# 01 Producer 性能调优

Producer 性能调优的本质是在吞吐、延迟、可靠性和资源消耗之间取舍。

## 先明确目标

调参前先回答：

- 你追求更高吞吐，还是更低延迟？
- 消息能不能丢？
- 单条消息平均大小是多少？
- 峰值 QPS 是多少？
- broker、网络、CPU 是否有瓶颈？

不同目标会得到不同配置。

## 关键参数

### batch.size

控制 producer 批次大小。

增大 batch 可能提升吞吐，因为一次请求能携带更多消息。

代价：

- 单条消息等待时间可能增加。
- 内存占用增加。

### linger.ms

控制 producer 等待更多消息组成 batch 的时间。

例如 `linger.ms=10` 表示最多等 10ms，让更多消息凑成一批。

适合高吞吐场景。

低延迟接口不要设置过大。

### compression.type

推荐优先考虑：

- `lz4`
- `zstd`
- `snappy`

压缩效果：

- 降低网络流量。
- 降低 broker 磁盘占用。
- 增加 producer 和 consumer CPU。

消息内容越大、重复字段越多，压缩收益通常越明显。

### acks

可靠性和延迟的取舍。

- `acks=all`：可靠性优先。
- `acks=1`：延迟和吞吐可能更好，但可靠性下降。
- `acks=0`：除非日志埋点等可丢场景，否则不推荐。

## 常见配置组合

### 关键业务事件

```text
acks=all
enable.idempotence=true
compression.type=zstd
linger.ms=5~20
```

适合订单、支付、库存等事件。

### 高吞吐日志

```text
acks=1 或 all
compression.type=lz4
linger.ms=20~100
batch.size 较大
```

适合行为日志、访问日志。

### 低延迟事件

```text
linger.ms=0~5
batch.size 适中
compression.type=lz4 或 none
```

适合对响应时间非常敏感的链路。

## Go 侧注意事项

- 不要每发送一条消息就创建一个 producer。
- producer 应该是长期复用的。
- 发送时设置 context 超时。
- 异步发送必须消费 delivery report。
- 服务退出前必须 close 或 flush。
- 发送失败要有日志和指标。

## 本节练习

1. 准备 1KB、10KB、100KB 三种消息。
2. 分别测试无压缩、lz4、zstd。
3. 记录吞吐、平均延迟、P95 延迟。
4. 写出你观察到的取舍。

