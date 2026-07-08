# 11. 处理数字精度和金额字段

本节目标：学完后你能避免 JSON 数字解析造成金额、订单号和大整数精度丢失。

简短引入：`encoding/json` 把数字解析到 `interface{}` 时，默认会变成 `float64`。这对金额、大订单号、大 ID 很危险，因为浮点数可能丢精度。

## 一、为什么需要它

后端项目里，数字不只是数字。它可能是：

- 金额。
- 库存。
- 订单号。
- 用户 ID。
- 分布式雪花 ID。

金额不能用浮点数随便算，大整数不能在 `float64` 里来回转换。否则可能出现少一分钱、ID 不一致这类严重问题。

```text
金额优先用整数分或定点十进制，不要用 float64 表示业务金额。
```

## 二、基本用法

Windows PowerShell：

```powershell
mkdir json-number
cd json-number
go mod init json-number
notepad main.go
go run .
```

Linux/macOS：

```bash
mkdir json-number
cd json-number
go mod init json-number
nano main.go
go run .
```

`main.go`：

```go
package main

import (
	"encoding/json"
	"fmt"
	"strings"
)

func main() {
	body := `{"order_id":9007199254740993,"amount_cent":1999}`

	dec := json.NewDecoder(strings.NewReader(body))
	dec.UseNumber()

	var data map[string]any
	if err := dec.Decode(&data); err != nil {
		panic(err)
	}

	orderID, err := data["order_id"].(json.Number).Int64()
	if err != nil {
		panic(err)
	}

	amountCent, err := data["amount_cent"].(json.Number).Int64()
	if err != nil {
		panic(err)
	}

	fmt.Println(orderID, amountCent)
}
```

## 三、关键参数/语法/代码结构

`dec.UseNumber()` 会让解码到 `interface{}` 的数字保持为 `json.Number`，而不是默认的 `float64`。

`json.Number.Int64()` 尝试把数字转成 `int64`。如果数字不是整数或超出范围，会返回错误。

如果你已经有明确结构体字段，例如 `AmountCent int64`，可以直接解码到结构体。`UseNumber` 主要用于动态 JSON 或 `map[string]any`。

真实项目中优先使用明确结构体字段。只有在字段集合不固定时，才考虑 `map[string]any` 加 `UseNumber`。结构体能让编译器帮你检查类型，也能让接口契约更清楚。

金额字段建议在接口文档里明确单位，例如 `amount_cent` 表示分。不要只写 `amount`，因为调用方不知道它是元、分，还是小数金额。

## 四、真实后端场景示例

订单创建请求可以用整数分表示金额。

```go
type CreatePaymentRequest struct {
	OrderNo    string `json:"order_no"`
	AmountCent int64  `json:"amount_cent"`
	Currency   string `json:"currency"`
}

func validatePayment(req CreatePaymentRequest) error {
	if req.OrderNo == "" {
		return fmt.Errorf("order_no is required")
	}
	if req.AmountCent <= 0 {
		return fmt.Errorf("amount_cent must be positive")
	}
	if req.Currency != "CNY" {
		return fmt.Errorf("unsupported currency")
	}
	return nil
}
```

真实项目中，金额入库要使用数据库合适类型，例如整数分字段或 `DECIMAL`。迁移金额字段时必须有数据校验脚本和回滚方案，不能直接把浮点字段改类型上线。

支付请求里还要避免让前端决定最终金额。前端传来的金额可以用于展示或二次确认，但服务端应该根据商品价格、优惠券、活动规则和用户资格重新计算。

```text
前端传入的金额不能作为最终扣款依据。
```

## 五、注意点

如果订单号只是标识，不参与数学计算，可以考虑用字符串传输。这样能避免前端 JavaScript 大整数精度问题。

库存数量可以用整数，但更新库存时要注意并发。真实项目中通常需要数据库条件更新、行锁或乐观锁，不能只在内存里判断。

金额计算涉及优惠券、退款、分账时，要把计算规则放在业务层，并用测试覆盖边界情况。

如果和外部系统对接，要确认对方的金额单位。有的接口用分，有的用元字符串，有的用小数。接入时最好在边界层转换成项目内部统一表示，业务层不要混用多个金额单位。

金额相关测试至少覆盖：1 分钱、0 元、负数、超大金额、优惠后为 0、退款金额大于支付金额。不要只测一个正常订单。

## 六、常见误区

误区一：用 `float64` 表示金额。浮点数适合近似计算，不适合财务金额。

误区二：把大 ID 解码到 `map[string]any` 后再转 `int64`。默认已经变成 `float64`，精度可能已经丢了。

误区三：订单号前端传数字。JavaScript 对大整数不安全，跨系统标识更建议用字符串。

误区四：只校验金额大于 0，不校验币种、上限和业务状态。支付类接口需要更保守的校验。

## 七、本节达标标准

- 能解释 `map[string]any` 中数字默认变成 `float64` 的风险。
- 能使用 `Decoder.UseNumber` 保留数字精度。
- 能用 `int64` 的分单位表示金额。
- 能判断大 ID 什么时候更适合作为字符串传输。
- 能说出金额字段迁移需要校验和回滚方案。
