# 02. 用 Printf 控制业务字段格式

本节目标：学会用 `Printf` 和常用格式化动词输出后端业务字段。

简短引入：`Println` 适合快速输出，但当你希望字段顺序固定、金额保留两位、ID 按特定格式展示时，就需要 `Printf`。

## 一、为什么需要它

后端开发经常要把不同类型的数据放进一段文本里。比如订单号是字符串，库存是整数，金额是小数，权限是否开启是布尔值。`Printf` 可以让这些字段的位置和格式更明确。

```text
格式化的目的不是炫技，而是让输出稳定、可读、方便排查。
```

## 二、基本用法

```go
package main

import "fmt"

func main() {
	orderID := "ORD-001"
	amount := 99.9
	paid := true

	fmt.Printf("order=%s amount=%.2f paid=%t\n", orderID, amount, paid)
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

常用动词先掌握这些就够：

- `%s`：字符串，例如订单号、用户名。
- `%d`：十进制整数，例如库存、数量、用户 ID。
- `%f`：小数，`%.2f` 表示保留两位。
- `%t`：布尔值。
- `%v`：默认格式，适合临时调试。
- `%%`：输出百分号本身。

`Printf` 不会自动换行，所以真实项目中通常在格式字符串末尾写 `\n`。

## 四、真实后端场景示例

下面模拟库存扣减后的输出：

```go
package main

import "fmt"

func main() {
	sku := "BOOK-GO-001"
	before := 20
	deduct := 3
	after := before - deduct

	fmt.Printf("sku=%s before=%d deduct=%d after=%d\n", sku, before, deduct, after)
}
```

这类输出适合本地验证业务逻辑。上线后，通常会改成结构化日志，比如记录 `sku`、`before`、`deduct`、`after` 四个字段。

## 五、注意点

格式化动词和参数类型不匹配时，程序一般不会崩溃，但输出会出现 `%!d(string=xxx)` 这类提示。这很有用，说明格式字符串写错了。

金额在真实系统里通常不要用 `float64` 作为核心存储类型。示例用 `float64` 只是为了学习 `fmt`。生产中常见做法是用“分”为单位的整数，展示时再格式化。

如果金额用“分”存储，可以这样展示：

```go
package main

import "fmt"

func main() {
	amountCent := int64(1299)
	fmt.Printf("amount=%d.%02d\n", amountCent/100, amountCent%100)
}
```

这里 `%02d` 表示不够两位时在左侧补 `0`。常见场景是订单金额、优惠金额、退款金额的展示。注意，这只负责显示；真实项目里的金额计算、汇率、舍入规则通常要统一封装，不能散落在各个 `Printf` 里。

排查格式化问题时，可以按这个顺序看：

- 输出里有没有 `%!`，有就说明动词和参数对不上。
- 参数顺序是否和格式字符串一致。
- 字符串 ID 是否误用了 `%d`。
- 是否把展示格式误当成存储格式。

## 六、常见误区

用 `%d` 打印字符串 ID。很多订单号看起来像数字，但业务上是字符串，应使用 `%s`。

忘记换行。多个 `Printf` 输出连在一起，会让排查很难受。

认为 `%.2f` 可以解决金额精度问题。它只解决展示格式，不解决计算和存储精度。

## 七、本节达标标准

- 能使用 `%s`、`%d`、`%.2f`、`%t` 输出业务字段。
- 能识别 `%!` 开头的格式化错误。
- 知道 `Printf` 默认不换行。
- 知道展示金额和存储金额是两个问题。
