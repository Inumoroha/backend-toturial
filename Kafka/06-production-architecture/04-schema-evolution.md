# 04 Schema 与版本演进

Kafka 消息一旦被多个服务消费，消息格式就变成了服务之间的契约。随意改字段会造成线上事故。

## 为什么需要 Schema

如果 producer 改了字段名：

```json
{
  "order_id": "order_1"
}
```

改成：

```json
{
  "id": "order_1"
}
```

旧 consumer 可能直接解析失败。

## 版本字段

每条事件建议带版本：

```json
{
  "event_type": "order.created",
  "version": 1,
  "data": {}
}
```

consumer 根据 version 做兼容处理。

## 兼容性原则

安全变更：

- 新增可选字段。
- 保留旧字段。
- 给字段设置默认值。

危险变更：

- 删除字段。
- 修改字段类型。
- 修改字段含义。
- 重命名字段。

## JSON、Protobuf、Avro

### JSON

优点：

- 可读性好。
- 调试方便。
- 新手友好。

缺点：

- 类型约束弱。
- 消息较大。
- 兼容性靠自律。

### Protobuf

优点：

- 类型明确。
- 性能好。
- 适合 Go 项目。

缺点：

- 可读性不如 JSON。
- 需要维护 `.proto`。

### Avro

常与 Schema Registry 搭配，在 Kafka 生态中常见。

## Go 项目建议

学习阶段可以用 JSON。

进入生产设计时，建议：

- 定义明确 event struct。
- 每个事件带 version。
- 兼容新增字段。
- 不破坏旧 consumer。
- 关键 topic 使用 schema 管理工具。

## 本节练习

1. 为 `order.created` 写 v1 JSON。
2. 增加一个可选字段，设计 v2。
3. 思考：如果要把 `amount` 从 int 改成 string，如何兼容？
4. 写出 consumer 同时兼容 v1 和 v2 的思路。

