# 4. Rebalance 机制与 Go 服务退出

本节目标：理解 consumer group rebalance 为什么发生、会带来什么影响，以及 Go consumer 服务如何优雅退出以减少重复消费和长时间停顿。

Rebalance 是 Kafka consumer group 的正常机制，但频繁 rebalance 是生产事故常见信号。Go 后端工程师必须知道：哪些行为会触发 rebalance，服务退出时应该怎样处理正在消费的消息。

---

## 一、Rebalance 是什么

Rebalance 是 consumer group 重新分配 partition 的过程。

例如：

```text
原来：
consumer-A -> partition-0, partition-1
consumer-B -> partition-2

新增 consumer-C 后：
consumer-A -> partition-0
consumer-B -> partition-1
consumer-C -> partition-2
```

这个重新分配过程就是 rebalance。

---

## 二、什么时候会触发 Rebalance

常见触发原因：

- 新 consumer 加入 group。
- consumer 正常退出 group。
- consumer 崩溃或心跳超时。
- topic partition 数量变化。
- consumer 长时间不 poll。
- 网络抖动导致 coordinator 认为 consumer 不健康。

---

## 三、Rebalance 的影响

Rebalance 期间可能出现：

- 消费短暂停顿。
- partition 所属 consumer 变化。
- 正在处理的消息可能重复。
- lag 暂时升高。
- 日志中出现 join group、sync group。

正常扩容或发布时 rebalance 可以接受。

频繁无原因 rebalance 则需要排查。

---

## 四、心跳与超时直觉

Consumer 需要让 Kafka 知道自己还活着。

相关概念：

```text
heartbeat
session timeout
max poll interval
```

如果 consumer 长时间没有心跳，Kafka 会认为它挂了。

如果 consumer 长时间不 poll，Kafka 也可能认为它处理能力异常。

Go handler 如果处理一条消息特别久，就可能引发问题。

---

## 五、Go 服务为什么容易触发 Rebalance

常见原因：

- handler 调外部 HTTP 没有 timeout。
- 数据库慢查询卡住。
- 单条消息处理时间超过预期。
- goroutine 泄漏导致资源耗尽。
- CPU 打满或 GC 抖动。
- 服务发布时直接强杀进程。

所以 consumer handler 必须有：

```text
context timeout
日志
指标
错误分类
优雅退出
```

---

## 六、优雅退出为什么重要

直接 kill consumer：

```text
消息处理到一半
进程退出
offset 未提交
group 等待超时后 rebalance
其他 consumer 接管
这条消息重复消费
```

重复消费可以接受，前提是业务幂等。

但优雅退出可以减少：

- 长时间等待。
- 不必要 rebalance。
- 半处理状态。
- 日志混乱。

---

## 七、Go Consumer 优雅退出流程

推荐流程：

```text
收到 SIGTERM
取消 root context
停止拉取新消息
等待当前消息处理完成
成功则提交 offset
关闭 consumer
关闭 producer
退出进程
```

要设置最大等待时间：

```text
graceful timeout = 30s
```

如果超过时间仍未完成，只能退出，并依赖幂等处理重复消费。

---

## 八、Go 伪代码

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

done := make(chan struct{})

go func() {
    defer close(done)
    consumer.Run(ctx)
}()

select {
case <-done:
case <-ctx.Done():
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    consumer.Close(shutdownCtx)
}
```

实际封装中要保证：

- `Run` 停止拉取新消息。
- 当前 handler 有机会结束。
- producer flush。
- 日志记录退出原因。

---

## 九、Rebalance 期间 Offset 怎么办

如果消息处理成功但还没提交 offset，rebalance 后可能被另一个 consumer 重新处理。

所以：

```text
rebalance 是重复消费的重要来源之一
```

解决：

- 业务幂等。
- 尽快提交成功消息 offset。
- 避免 handler 超长。
- 优雅退出。

---

## 十、命令实验：观察 Rebalance

创建 topic：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --if-not-exists \
  --topic stage4.rebalance.demo \
  --partitions 3 \
  --replication-factor 1
```

启动第一个 consumer：

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic stage4.rebalance.demo \
  --group rebalance-demo-group
```

再启动第二个、第三个 consumer。

观察 consumer 日志和消息分配。

关闭其中一个 consumer，再查看 group：

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group rebalance-demo-group
```

---

## 十一、常见误区

### 1. Rebalance 一定是故障

不是。

扩容、缩容、发布都会触发正常 rebalance。

### 2. Rebalance 不会导致重复

可能会。

处理成功但 offset 未提交时，可能重复消费。

### 3. Consumer 退出直接 kill 就行

不建议。

要尽量优雅退出。

### 4. 只靠调大超时解决问题

不够。

如果 handler 本身慢，应该优化业务处理或拆分任务。

---

## 十二、本节练习

1. 启动 3 个同 group consumer。
2. 关闭其中一个，观察 group 状态变化。
3. 写出 Go consumer 优雅退出步骤。
4. 解释 rebalance 为什么可能导致重复消费。
5. 思考：handler 调外部接口没有 timeout 会带来什么风险？

---

## 十三、本节小结

- Rebalance 是 consumer group 重新分配 partition。
- consumer 加入、退出、超时、partition 变化都会触发 rebalance。
- Rebalance 期间消费可能暂停，也可能带来重复消费。
- Go consumer 必须支持优雅退出。
- handler 要有 timeout，避免长时间卡住。
- 业务幂等是应对 rebalance 重复消费的根本手段。

---

## 十四、优雅退出流程

Go consumer 收到退出信号后，推荐顺序：

```text
停止拉取新消息。
等待当前 handler 完成或超时。
成功处理的消息提交 offset。
关闭 consumer 客户端。
释放数据库连接。
```

伪代码：

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

if err := consumer.Run(ctx); err != nil && !errors.Is(err, context.Canceled) {
    logger.Error("consumer stopped", "error", err)
}
```

---

## 十五、Rebalance 后为什么会重复

如果 rebalance 发生时 offset 还没提交，新 consumer 会从旧 offset 继续。

这不是 bug，而是 Kafka 为了避免丢消息做出的选择。

所以可靠 consumer 的核心仍然是：

```text
业务成功后提交 offset。
业务必须幂等。
```

---

## 十六、练习补充

1. 启动两个 consumer，观察分区分配。
2. 停止其中一个 consumer，观察 rebalance。
3. 在 handler sleep 期间停止进程，观察是否重复消费。
4. 用 processed_events 验证重复消费不会重复执行业务。

---

## 十七、优雅退出伪代码

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

err := consumer.Run(ctx)
if err != nil && !errors.Is(err, context.Canceled) {
    logger.Error("consumer stopped with error", "error", err)
}
```

`Run` 内部应该做到：

```text
停止拉取新消息。
等待当前 handler 完成或超时。
成功处理后提交 offset。
关闭 Kafka client。
释放数据库连接。
```

---

## 十八、发布时的注意事项

Kubernetes 或容器环境发布 consumer 时，要关注：

- `terminationGracePeriodSeconds` 是否足够 handler 完成。
- readiness probe 是否能在退出前摘流量。
- handler 是否支持 context 取消。
- 是否有过长事务阻塞退出。

发布不是简单杀进程。粗暴退出会增加重复消费和 rebalance 抖动。

---

## 十九、发布前检查

- [ ] handler 是否有超时。
- [ ] 是否能响应 context 取消。
- [ ] 退出前是否停止拉取新消息。
- [ ] 退出时是否关闭 Kafka client。
- [ ] 业务是否幂等。

满足这些条件，再做滚动发布会稳很多。

---

## 二十、Rebalance 观察记录

记录一次发布或重启过程：

```text
重启前 consumer 数：
重启后 consumer 数：
rebalance 耗时：
是否出现重复消费：
lag 是否短暂升高：
```

这能帮助你理解发布对消费链路的影响。

---

## 二十一、发布时观察什么

滚动发布 consumer 服务时，至少观察：

```text
rebalance 次数是否明显增加。
每次 rebalance 持续多久。
发布期间 lag 是否持续上升。
旧实例是否在超时时间内优雅退出。
是否存在处理到一半就被中断的消息。
新实例接管后是否重复处理同一批消息。
```

这些现象都要写进发布记录。Kafka consumer 的稳定性，很多时候不是由正常消费决定，而是由发布、扩容、缩容这些动态过程决定。

---

## 二十二、优雅退出日志

Go consumer 退出时，建议至少打印：

```text
收到退出信号的时间。
停止拉取新消息的时间。
正在处理的消息数量。
最后提交的 offset。
退出总耗时。
是否因为超时强制退出。
```

有了这些日志，发布后出现重复消费时，才能判断是正常接管还是退出流程不完整。
