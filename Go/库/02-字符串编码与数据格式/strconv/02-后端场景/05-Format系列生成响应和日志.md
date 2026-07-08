# 05. Format 系列生成响应和日志

本节目标：学完后，你能用 `FormatInt`、`FormatBool`、`FormatFloat` 生成可控的日志字段、缓存 key 和简单文本输出。

简短引入：后端服务不仅要解析输入，也要把内部的数字、布尔值、小数输出成字符串。比如日志字段、Redis key、消息队列里的轻量文本字段，都可能用到 `strconv.Format...`。

## 一、为什么需要它

可以理解为：`Format` 系列负责把 Go 里的基础类型变成文本。

常见场景是：

- 生成缓存 key：`order:10001`
- 输出日志字段：`success=true`
- 展示重量：`2.50`
- 把 `int64` ID 放进字符串协议

```text
Format 系列适合生成简单文本字段；复杂 JSON 响应应交给 encoding/json。
```

## 二、基本用法

Windows PowerShell：

```powershell
@'
package main

import (
	"fmt"
	"strconv"
)

func main() {
	orderID := int64(10001)
	success := true
	amount := 19.9

	fmt.Println("cacheKey =", "order:"+strconv.FormatInt(orderID, 10))
	fmt.Println("success =", strconv.FormatBool(success))
	fmt.Println("amount =", strconv.FormatFloat(amount, 'f', 2, 64))
}
'@ | Set-Content -Encoding UTF8 main.go
go run main.go
```

Linux/macOS：

```bash
cat > main.go <<'EOF'
package main

import (
	"fmt"
	"strconv"
)

func main() {
	orderID := int64(10001)
	success := true
	amount := 19.9

	fmt.Println("cacheKey =", "order:"+strconv.FormatInt(orderID, 10))
	fmt.Println("success =", strconv.FormatBool(success))
	fmt.Println("amount =", strconv.FormatFloat(amount, 'f', 2, 64))
}
EOF
go run main.go
```

## 三、关键参数/语法/代码结构

`FormatInt(i, base)`：把有符号整数转成字符串。后端业务最常用 `base=10`。

`FormatUint(i, base)`：把无符号整数转成字符串，适合权限位、位图、短链接编码。

`FormatBool(b)`：输出 `"true"` 或 `"false"`。

`FormatFloat(f, fmt, prec, bitSize)`：

- `fmt='f'`：普通小数。
- `prec=2`：保留两位。
- `bitSize=64`：按 `float64` 处理。

## 四、真实后端场景示例

下面模拟一个订单支付日志。日志不是为了好看，而是为了线上排查。

```go
package main

import (
	"fmt"
	"strconv"
	"strings"
)

func paymentLog(orderID int64, userID int64, success bool, costMS int64) string {
	var b strings.Builder
	b.WriteString("event=payment")
	b.WriteString(" order_id=")
	b.WriteString(strconv.FormatInt(orderID, 10))
	b.WriteString(" user_id=")
	b.WriteString(strconv.FormatInt(userID, 10))
	b.WriteString(" success=")
	b.WriteString(strconv.FormatBool(success))
	b.WriteString(" cost_ms=")
	b.WriteString(strconv.FormatInt(costMS, 10))
	return b.String()
}

func main() {
	fmt.Println(paymentLog(10001, 501, true, 38))
}
```

真实项目中更常见的是使用结构化日志库，让日志字段保持类型信息。但理解 `Format` 系列有助于你看懂底层行为，也能处理轻量脚本和简单协议。

## 五、注意点

不要手写复杂 JSON：

```go
// 不建议
// body := "{\"id\":" + strconv.FormatInt(id, 10) + "}"
```

应该使用：

```go
// json.NewEncoder(w).Encode(resp)
```

原因是 JSON 里有字符串转义、空值、嵌套结构、字段标签等问题，手写很容易漏。

```text
简单字段可以 Format，结构化数据交给对应编码器。
```

## 六、常见误区

- 误区一：所有响应都用字符串拼接。  
  复杂响应应该交给 `encoding/json`，否则容易出现转义错误和字段遗漏。

- 误区二：日志只打印一句中文。  
  线上排查更需要稳定字段，比如 `order_id=... user_id=...`，方便搜索和聚合。

- 误区三：金额展示依赖 `FormatFloat` 后再参与计算。  
  格式化结果是字符串，只适合展示，不应该再用于核心账务计算。

## 七、本节达标标准

- 能用 `FormatInt` 生成 ID 字符串。
- 能用 `FormatBool` 输出布尔状态。
- 能用 `FormatFloat` 控制小数字段展示。
- 知道复杂 JSON 不应该手写拼接。
- 能写出可搜索的简单日志字段。
