# 10. 用 RawMessage 处理可扩展事件

本节目标：学完后你能用 `json.RawMessage` 延迟解析不同类型的事件数据。

简短引入：消息队列、审计日志、Webhook 常常有一个共同字段表示事件类型，另一个字段保存不同结构的事件内容。`RawMessage` 适合先读类型，再决定按哪种结构解析。

## 一、为什么需要它

订单系统里可能有这些事件：

- `order_created`
- `order_paid`
- `order_cancelled`

它们都有 `type`，但 `payload` 字段结构不同。直接用一个大结构体接收，会出现很多可选字段；直接用 `map[string]any`，又会丢失类型约束。

`RawMessage` 可以理解为“先把这段 JSON 原样留着，等我知道类型后再解析”。

```text
事件类型决定 payload 结构，不要一开始就把所有事件塞进一个大结构体。
```

## 二、基本用法

Windows PowerShell：

```powershell
mkdir json-rawmessage
cd json-rawmessage
go mod init json-rawmessage
notepad main.go
go run .
```

Linux/macOS：

```bash
mkdir json-rawmessage
cd json-rawmessage
go mod init json-rawmessage
nano main.go
go run .
```

`main.go`：

```go
package main

import (
	"encoding/json"
	"fmt"
)

type Event struct {
	Type    string          `json:"type"`
	Payload json.RawMessage `json:"payload"`
}

type OrderPaidPayload struct {
	OrderNo string `json:"order_no"`
	Amount  int64  `json:"amount_cent"`
}

func main() {
	body := []byte(`{"type":"order_paid","payload":{"order_no":"NO1001","amount_cent":9900}}`)

	var event Event
	if err := json.Unmarshal(body, &event); err != nil {
		panic(err)
	}

	switch event.Type {
	case "order_paid":
		var payload OrderPaidPayload
		if err := json.Unmarshal(event.Payload, &payload); err != nil {
			panic(err)
		}
		fmt.Printf("paid: %+v\n", payload)
	default:
		fmt.Println("unknown event:", event.Type)
	}
}
```

## 三、关键参数/语法/代码结构

`json.RawMessage` 本质上是 `[]byte`，但它告诉 `encoding/json`：这段内容先保留原始 JSON。

第一层结构体只解析公共字段，例如事件类型、事件 ID、时间戳。

第二层根据 `Type` 解析 `Payload`。这样每种事件都能有自己的结构体和校验逻辑。

事件处理代码通常会拆成一个分发函数：

```go
func handleEvent(event Event) error {
	switch event.Type {
	case "order_paid":
		var payload OrderPaidPayload
		if err := json.Unmarshal(event.Payload, &payload); err != nil {
			return err
		}
		if payload.OrderNo == "" || payload.Amount <= 0 {
			return fmt.Errorf("invalid order_paid payload")
		}
		return nil
	default:
		return fmt.Errorf("unknown event type: %s", event.Type)
	}
}
```

这样做的好处是每个事件类型都有清晰入口。将来新增 `order_refunded` 时，不需要改一个巨大结构体，只要新增 payload 结构和分支处理。

## 四、真实后端场景示例

Webhook 接口常见写法：

```go
type WebhookEvent struct {
	EventID string          `json:"event_id"`
	Type    string          `json:"type"`
	Payload json.RawMessage `json:"payload"`
}

type UserDeletedPayload struct {
	UserID int `json:"user_id"`
}
```

真实项目中，处理事件前通常要做：

- 验签，确认来源可信。
- 根据 `event_id` 做幂等，避免重复消费。
- 根据 `type` 选择处理函数。
- 写处理日志，失败时可重试。

涉及数据库写入时，要用事务保护状态变更和事件记录。否则事件处理成功但日志没写，排查会很困难。

事件消费还要考虑幂等。比如同一个 `order_paid` 事件被投递两次，第二次不能重复给订单加钱、重复发货或重复扣库存。常见做法是在数据库中保存 `event_id`，并对它建立唯一索引。

```text
消息队列和 Webhook 默认都要按“可能重复投递”来设计。
```

## 五、注意点

不要因为用了 `RawMessage` 就跳过 payload 校验。第二层解析后仍然要检查必填字段和业务规则。

未知事件类型不要随便当成功处理。常见做法是记录日志、返回可重试或忽略，具体取决于事件来源和业务协议。

如果事件要长期保存，建议保存原始 JSON，同时保存解析后的关键字段，方便审计和查询。

保存原始 JSON 时要注意敏感字段。第三方回调里可能包含手机号、地址、token 或支付信息，入库和日志都要做权限控制，必要时加密或脱敏。

如果事件 payload 很大，不要给所有 payload 字段建索引。通常只把 `event_id`、`type`、`created_at`、业务主键这些字段建成普通列，用于查询和排错。

## 六、常见误区

误区一：所有事件都用 `map[string]any` 处理。短期方便，长期会让字段类型和校验散掉。

误区二：没有事件幂等。消息队列和 Webhook 都可能重复投递。

误区三：未知事件直接返回成功但不记录。后续协议升级时很难发现系统没有处理新事件。

误区四：事件处理里多张表更新没有事务。失败后可能造成订单状态和事件日志不一致。

## 七、本节达标标准

- 能使用 `json.RawMessage` 延迟解析 payload。
- 能按事件类型选择不同结构体解析。
- 能解释为什么事件处理需要幂等。
- 能知道事件写库要考虑事务和处理日志。
- 能判断什么时候不应该用一个大结构体接收所有事件。
