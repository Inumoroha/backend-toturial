# 3. Schema 版本演进

本节目标：理解 Kafka 消息格式是服务之间的契约，学会设计 version、兼容新增字段和避免破坏旧 consumer。

---

## 一、为什么需要 Schema

Producer 和 consumer 是解耦的。

Producer 改字段时，consumer 可能还没升级。

如果随意把：

```json
{"order_id":"order_1"}
```

改成：

```json
{"id":"order_1"}
```

旧 consumer 可能解析失败。

---

## 二、事件 Envelope

推荐：

```json
{
  "event_id": "evt_1",
  "event_type": "order.created",
  "version": 1,
  "occurred_at": "2026-07-05T12:00:00Z",
  "data": {}
}
```

`version` 用于兼容处理。

---

## 三、安全变更

通常安全：

- 新增可选字段。
- 保留旧字段。
- 字段含义不变。
- 默认值兼容。

例如新增：

```json
"coupon_amount": 1000
```

旧 consumer 可以忽略。

---

## 四、危险变更

危险：

- 删除字段。
- 改字段类型。
- 改字段名称。
- 改字段含义。
- 把可选字段变必填。

这些都可能破坏旧 consumer。

---

## 五、JSON、Protobuf、Avro

JSON：

- 可读性好。
- 适合学习。
- 约束弱。

Protobuf：

- 类型强。
- 体积小。
- 适合 Go 项目。

Avro：

- Kafka 生态常见。
- 常和 Schema Registry 搭配。

---

## 六、Go Consumer 兼容处理

```go
switch event.Version {
case 1:
    handleV1(event)
case 2:
    handleV2(event)
default:
    return kafka.NonRetryable(errors.New("unsupported version"))
}
```

不支持的版本可以进 DLQ。

---

## 七、发布顺序

兼容发布常见顺序：

```text
先升级 consumer，让它兼容 v1 和 v2
再升级 producer，开始发送 v2
确认旧消息处理完成
最后移除 v1 兼容逻辑
```

---

## 八、本节练习

1. 设计 `order.created` v1。
2. 新增一个可选字段设计 v2。
3. 判断删除 `user_id` 是否安全。
4. 写出 consumer 兼容 v1/v2 的伪代码。

---

## 九、本节小结

- Kafka 消息格式是服务契约。
- 每条事件应带 version。
- 新增可选字段通常安全。
- 删除、改名、改类型危险。
- 生产变更要先升级 consumer，再升级 producer。

---

## 十、OrderCreated v1 示例

```json
{
  "event_id": "evt_order_created_order_1001",
  "event_type": "order.created",
  "version": 1,
  "occurred_at": "2026-07-05T12:00:00Z",
  "data": {
    "order_id": "order_1001",
    "user_id": "user_88",
    "total_amount": 19800
  }
}
```

Consumer v1 依赖：

```text
order_id
user_id
total_amount
```

---

## 十一、OrderCreated v2 示例

新增可选字段：

```json
{
  "event_id": "evt_order_created_order_1001",
  "event_type": "order.created",
  "version": 2,
  "occurred_at": "2026-07-05T12:00:00Z",
  "data": {
    "order_id": "order_1001",
    "user_id": "user_88",
    "total_amount": 19800,
    "coupon_amount": 1000
  }
}
```

这通常是兼容变更，因为旧 consumer 可以忽略 `coupon_amount`。

---

## 十二、不兼容变更示例

把：

```json
"total_amount": 19800
```

改成：

```json
"total_amount": "198.00"
```

这是危险变更，因为字段类型变了。

更好的做法：

```json
"total_amount": 19800,
"total_amount_display": "198.00"
```

保留旧字段，新增新字段。

---

## 十三、Go 解码兼容示例

```go
type EventEnvelope struct {
    EventID   string          `json:"event_id"`
    EventType string          `json:"event_type"`
    Version   int             `json:"version"`
    Data      json.RawMessage `json:"data"`
}

func Handle(ctx context.Context, payload []byte) error {
    var env EventEnvelope
    if err := json.Unmarshal(payload, &env); err != nil {
        return kafka.NonRetryable(err)
    }

    switch env.Version {
    case 1:
        return handleOrderCreatedV1(ctx, env.Data)
    case 2:
        return handleOrderCreatedV2(ctx, env.Data)
    default:
        return kafka.NonRetryable(fmt.Errorf("unsupported version %d", env.Version))
    }
}
```

---

## 十四、发布流程示例

安全升级顺序：

```text
第 1 步：consumer 支持 v1 和 v2。
第 2 步：部署所有 consumer。
第 3 步：producer 开始发送 v2。
第 4 步：观察 DLQ 和错误率。
第 5 步：确认无旧消息后，未来版本再考虑移除 v1。
```

不要先升级 producer，再让旧 consumer 被新格式打挂。

---

## 十五、Schema 评审清单

- [ ] 是否有 event version？
- [ ] 新字段是否可选？
- [ ] 是否删除了旧字段？
- [ ] 是否修改了字段类型？
- [ ] 旧 consumer 是否能忽略新字段？
- [ ] 不支持版本是否进入 DLQ？
- [ ] 是否有示例 JSON？
- [ ] 是否有兼容测试？

---

## 十四、兼容测试示例

为每个事件保存样例：

```text
testdata/order_created_v1.json
testdata/order_created_v2.json
```

consumer 测试：

```go
func TestDecodeOrderCreatedV1(t *testing.T) {
    raw := readTestdata("order_created_v1.json")
    event, err := DecodeOrderCreated(raw)
    require.NoError(t, err)
    require.Equal(t, "order.created", event.EventType)
}
```

每次改 schema，都跑这些测试，确保旧消息还能被消费。

---

## 十五、不兼容变更怎么做

如果必须删除字段或改变含义，不要直接覆盖原 topic。更稳的方式是新增版本、灰度 producer、让 consumer 同时兼容新旧版本。

---

## 十六、面试表达

```text
事件 schema 演进时，我会优先做向后兼容变更，比如新增可选字段。
如果必须做不兼容变更，会通过 version、灰度发布和兼容测试保证旧 consumer 不被突然打坏。
```

---

## 十七、兼容矩阵

设计 schema 演进时，建议维护一张表：

| Producer | Consumer | 是否兼容 | 说明 |
| --- | --- | --- | --- |
| v1 | v1 | 是 | 原始版本 |
| v2 新增可选字段 | v1 | 是 | v1 忽略未知字段 |
| v1 | v2 | 是 | v2 给新字段默认值 |
| v3 删除字段 | v1 | 否 | v1 依赖该字段 |

上线前至少要保证：

```text
新 producer 发布后，旧 consumer 不会立刻失败。
新 consumer 发布后，旧消息仍然能处理。
```

---

## 十八、Go 解码建议

Go struct 中新增字段尽量使用可选语义：

```go
type OrderData struct {
    OrderID  string  `json:"order_id"`
    UserID   string  `json:"user_id"`
    CouponID *string `json:"coupon_id,omitempty"`
}
```

如果字段可能不存在，用指针或零值策略明确处理，不要默认假设所有历史消息都有新字段。

---

## 十九、Schema 发布检查表

- [ ] 新字段是否可选。
- [ ] 旧 consumer 是否能忽略新字段。
- [ ] 新 consumer 是否能处理旧消息。
- [ ] 是否保留示例 JSON。
- [ ] 是否有兼容测试。
- [ ] 不支持版本是否进入 DLQ。

Schema 变更要当成接口变更对待，而不是随手改 struct。

---

## 二十、发布流程

推荐流程：

```text
1. 新 consumer 先兼容旧 schema。
2. 灰度发布 consumer。
3. producer 开始发送新字段。
4. 观察 DLQ 和错误率。
5. 确认稳定后再清理旧字段逻辑。
```

不要先升级 producer 再让旧 consumer 被动承受新格式。

---

## 二十一、回滚策略

如果新 schema 导致 DLQ 增长：

```text
立即暂停新 producer。
保留 DLQ 消息。
修复 consumer 兼容逻辑。
通过受控工具重放 DLQ。
```

schema 回滚也要考虑已经写入 Kafka 的消息。

---

## 二十一、Schema 变更记录模板

每次修改事件结构，都写一条变更记录：

```text
事件名称：
当前版本：
目标版本：
新增字段：
删除字段：
字段默认值：
旧 consumer 是否兼容：
新 producer 是否灰度：
DLQ 是否可能增加：
回滚策略：
```

Kafka 事件一旦写入，就不能像数据库行一样轻易修改历史数据。

因此 schema 设计要默认存在多个版本并存的窗口期。
