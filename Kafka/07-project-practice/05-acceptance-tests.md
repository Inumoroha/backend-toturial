# 05 验收测试

本节目标：为项目设计验收测试，证明它不仅能跑通正常流程，也能处理失败和重复。

## 正常链路测试

步骤：

1. 启动 Kafka、数据库和服务。
2. 调用创建订单 API。
3. 查询订单表。
4. 查看 `order.created` 是否发布。
5. 查看库存是否扣减。
6. 查看通知是否记录。

通过标准：

- 订单创建成功。
- 事件发布成功。
- consumer 成功处理。
- offset 正常提交。
- 日志中能看到 event_id。

## 重复消费测试

步骤：

1. 准备一条固定 `event_id` 的 `order.created`。
2. 向 Kafka 写入两次。
3. 启动库存服务消费。
4. 查询库存扣减结果。

通过标准：

- 库存只扣减一次。
- 第二次消费被识别为重复。
- consumer 仍然提交 offset。

## 消费失败重试测试

步骤：

1. 让库存服务的数据库临时不可用。
2. 发送 `order.created`。
3. 观察消息是否进入 retry topic。
4. 恢复数据库。
5. 让 retry consumer 重新处理。

通过标准：

- 原消息不会无限卡住主 topic。
- retry 消息带有 attempt 和错误原因。
- 恢复后能处理成功。

## DLQ 测试

步骤：

1. 发送一条缺少 `order_id` 的坏消息。
2. consumer 解析或校验失败。
3. 查看 DLQ。

通过标准：

- 消息进入 DLQ。
- DLQ 包含原 topic、partition、offset、错误原因。
- 原消息 offset 被提交。

## 优雅退出测试

步骤：

1. 让 consumer 正在处理一条耗时消息。
2. 发送 SIGTERM。
3. 观察服务退出流程。

通过标准：

- consumer 停止拉取新消息。
- 当前消息处理完成或超时。
- offset 提交行为符合预期。
- 服务退出无 goroutine 泄漏。

## Lag 测试

步骤：

1. 批量发送 10000 条消息。
2. 暂停 consumer。
3. 查看 lag 增长。
4. 恢复 consumer。
5. 查看 lag 下降。

通过标准：

- 能观察 lag。
- consumer 恢复后 lag 能下降。
- 没有频繁 rebalance。

## 最终交付物

项目最终应该包含：

- `README.md`
- `docker-compose.yml`
- topic 创建脚本。
- 数据库 migration。
- Go 服务代码。
- 集成测试。
- 可靠性说明。
- 监控指标说明。
- 常见故障排查文档。

## 本节练习

1. 为每个验收测试写出命令或测试用例。
2. 把通过标准写进项目 README。
3. 至少自动化正常链路、重复消费、DLQ 三个测试。
4. 思考：为什么只测试正常链路远远不够？

