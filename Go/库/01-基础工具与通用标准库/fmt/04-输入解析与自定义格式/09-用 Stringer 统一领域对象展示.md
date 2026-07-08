# 09. 用 Stringer 统一领域对象展示

本节目标：学会为业务类型实现 `String() string`，让对象在 `fmt` 输出中有统一、可控的展示。

简短引入：很多后端对象默认打印并不适合阅读。实现 `Stringer` 后，`fmt` 在遇到这个对象时会调用它的 `String` 方法。

## 一、为什么需要它

可以把 `Stringer` 理解为“这个对象如何变成一段人能读懂的文本”。常见场景是：

- 订单状态展示。
- 权限动作展示。
- 脱敏后的用户标识展示。
- 枚举值展示。

## 二、基本用法

```go
package main

import "fmt"

type OrderStatus int

const (
	OrderPending OrderStatus = iota
	OrderPaid
	OrderCanceled
)

func (s OrderStatus) String() string {
	switch s {
	case OrderPending:
		return "pending"
	case OrderPaid:
		return "paid"
	case OrderCanceled:
		return "canceled"
	default:
		return "unknown"
	}
}

func main() {
	fmt.Println("status:", OrderPaid)
}
```

Windows PowerShell：

```powershell
go run .\main.go
```

Linux/macOS：

```bash
go run ./main.go
```

## 三、关键参数/语法/代码结构

只要类型实现：

```go
String() string
```

它就满足 `fmt.Stringer`。当你使用 `%s`、`%v`、`Println` 等输出时，`fmt` 会使用这个方法生成文本。

`String` 方法里要避免再次直接格式化自己，否则可能递归。

## 四、真实后端场景示例

下面给用户显示做脱敏：

```go
package main

import "fmt"

type UserEmail string

func (e UserEmail) String() string {
	s := string(e)
	if len(s) <= 3 {
		return "***"
	}
	return s[:3] + "***"
}

func main() {
	email := UserEmail("alice@example.com")
	fmt.Println("created user:", email)
}
```

真实项目中，脱敏规则要统一，不要散落在各个 `fmt.Sprintf` 里。更正式的做法可能是封装专门的脱敏工具。

## 五、注意点

`Stringer` 会影响对象在很多地方的展示。实现前要确认这个展示是否适合作为默认展示。

不要在 `String()` 中访问数据库、调用远程服务或做复杂逻辑。它应该轻量、稳定、无副作用。

`Stringer` 和 JSON 输出要分清。`String()` 主要给开发者、日志、调试文本使用；API 返回给前端的数据应该由结构体字段和 JSON 标签控制。

例如订单状态可以实现 `String()` 方便日志阅读，但接口响应最好仍然明确返回字段：

```go
type OrderResp struct {
	ID     string `json:"id"`
	Status string `json:"status"`
}
```

这样做的好处是接口协议稳定，不会因为某个类型的 `String()` 改了文案而影响前端或其他服务。

```text
Stringer 负责对象的文本展示，不负责定义外部接口协议。
```

## 六、常见误区

在 `String()` 里写 `return fmt.Sprintf("%s", e)`，这可能递归调用自己。应先转换底层类型：`string(e)`。

把 `Stringer` 当序列化协议。API 响应仍应使用 JSON 编码和明确字段。

默认展示暴露敏感数据。用户、密钥、Token 类型尤其要谨慎。

## 七、本节达标标准

- 能为枚举或业务值对象实现 `String() string`。
- 知道 `fmt` 会自动使用 `Stringer`。
- 能避免 `String()` 中的递归格式化。
- 知道 `Stringer` 不应承担复杂业务逻辑。
