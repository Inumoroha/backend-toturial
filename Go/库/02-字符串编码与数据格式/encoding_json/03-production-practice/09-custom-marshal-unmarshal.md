# 09. 自定义 JSON 编解码

本节目标：学完后你能为业务字段实现自定义 JSON 编解码，处理状态、时间和脱敏输出。

简短引入：有些字段不能直接按 Go 默认方式输出。比如订单状态要限制枚举，手机号返回前要脱敏，时间要用项目统一格式。这时可以使用自定义编解码。

## 一、为什么需要它

默认编解码适合普通字段，但真实项目会遇到业务语义：

- 订单状态只能是 `pending`、`paid`、`cancelled`。
- 手机号响应时不能完整暴露。
- 时间格式要和客户端约定。
- 金额不能随便变成浮点数。

自定义编解码可以把这些规则收在类型附近，减少散落在各个 handler 里的重复逻辑。

```text
自定义 JSON 编解码适合稳定规则，不适合把所有业务流程都塞进去。
```

## 二、基本用法

Windows PowerShell：

```powershell
mkdir json-custom
cd json-custom
go mod init json-custom
notepad main.go
go run .
```

Linux/macOS：

```bash
mkdir json-custom
cd json-custom
go mod init json-custom
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

type OrderStatus string

const (
	OrderPending   OrderStatus = "pending"
	OrderPaid      OrderStatus = "paid"
	OrderCancelled OrderStatus = "cancelled"
)

func (s *OrderStatus) UnmarshalJSON(data []byte) error {
	var value string
	if err := json.Unmarshal(data, &value); err != nil {
		return err
	}
	switch OrderStatus(value) {
	case OrderPending, OrderPaid, OrderCancelled:
		*s = OrderStatus(value)
		return nil
	default:
		return fmt.Errorf("invalid order status: %s", value)
	}
}

type UpdateOrderRequest struct {
	Status OrderStatus `json:"status"`
}

func main() {
	var req UpdateOrderRequest
	err := json.Unmarshal([]byte(`{"status":"paid"}`), &req)
	if err != nil {
		panic(err)
	}
	fmt.Printf("%+v\n", req)
}
```

## 三、关键参数/语法/代码结构

实现 `UnmarshalJSON(data []byte) error` 的类型，可以控制自己如何从 JSON 解码。

实现 `MarshalJSON() ([]byte, error)` 的类型，可以控制自己如何编码成 JSON。

方法接收者通常要根据方向选择。解码需要修改值，常用指针接收者；编码只读取值，常用值接收者。

自定义解码常见于“字段格式本身就有业务规则”的场景，比如状态枚举、日期格式、金额字符串。它适合处理局部规则，不适合处理跨表校验。比如“订单状态必须是 paid”可以在类型里判断，但“这个订单是否属于当前用户”应该放在业务层。

自定义编码常见于响应边界，比如脱敏手机号、隐藏内部状态、统一时间格式。要注意它只在 JSON 编码时生效，不会自动影响日志、数据库保存或其他序列化方式。

## 四、真实后端场景示例

用户信息响应里，手机号可以在 JSON 输出时脱敏。

```go
type MaskedPhone string

func (p MaskedPhone) MarshalJSON() ([]byte, error) {
	value := string(p)
	if len(value) < 7 {
		return json.Marshal("")
	}
	masked := value[:3] + "****" + value[len(value)-4:]
	return json.Marshal(masked)
}

type UserProfileResponse struct {
	ID    int         `json:"id"`
	Name  string      `json:"name"`
	Phone MaskedPhone `json:"phone"`
}
```

真实项目中，脱敏应该作为响应边界的一部分，而不是依赖前端隐藏。日志里也要注意脱敏，特别是手机号、邮箱、身份证号、token。

如果项目里多个接口都返回手机号，建议统一使用响应 DTO 或统一脱敏函数，而不是每个 handler 手写一遍。这样将来脱敏规则从 `138****0000` 改成 `138******00` 时，只需要改一个地方。

对于时间字段，真实项目中常见做法是统一返回 RFC3339 字符串，例如 `2026-07-06T15:00:00+08:00`。不要每个接口自己决定格式，否则客户端会出现大量特殊处理。

## 五、注意点

自定义编解码不要做数据库操作、RPC 调用或复杂业务流程。它应该保持轻量，主要处理格式和局部规则。

错误信息要注意边界。对外响应不一定直接返回 `invalid order status: xxx`，可以转换成稳定的业务错误码。

如果枚举值已经落库，修改枚举前要考虑历史数据迁移。迁移要有脚本、回滚方案和灰度策略。

测试自定义编解码时，要覆盖合法值、非法值、空字符串、类型错误和缺失字段。尤其是枚举类字段，测试能帮助你发现“新状态上线但旧代码不认识”的问题。

如果自定义类型被多个服务共享，枚举值变更还要考虑消息队列和历史事件。旧服务消费到新枚举时，应该能明确失败、降级或进入死信队列，而不是静默吞掉。

## 六、常见误区

误区一：在 `UnmarshalJSON` 里查数据库。这样会让解析阶段变慢，也让错误边界混乱。

误区二：自定义输出手机号后，以为所有地方都脱敏了。只有经过 JSON 编码的响应会走这个逻辑，日志和数据库不会自动脱敏。

误区三：枚举值随意改名。客户端、数据库历史数据、消息队列事件都可能依赖旧值。

误区四：在自定义编解码里吞掉错误。错误应该返回给调用方，由上层决定如何响应。

## 七、本节达标标准

- 能实现简单的 `UnmarshalJSON` 校验枚举值。
- 能实现简单的 `MarshalJSON` 做响应脱敏。
- 能判断哪些规则适合放进自定义编解码。
- 能说明自定义编解码不能替代业务层校验。
- 能意识到枚举值变更涉及迁移和兼容。
