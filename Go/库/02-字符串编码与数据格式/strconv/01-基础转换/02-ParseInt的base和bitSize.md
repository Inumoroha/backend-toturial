# 02. ParseInt 的 base 和 bitSize

本节目标：学完后，你能用 `ParseInt` 和 `ParseUint` 安全解析 ID、权限位、不同进制字符串，并理解越界错误。

简短引入：`Atoi` 适合简单 `int`，但后端项目里经常需要更明确的类型，比如 `int64` 用户 ID、`uint64` 权限位、十六进制迁移数据。这时应该使用 `ParseInt` 或 `ParseUint`。

## 一、为什么需要它

可以理解为：`ParseInt` 是更精细的整数解析器，你可以告诉它“按什么进制读”和“结果最多允许多大”。

常见场景是：

- 数据库 ID 使用 `BIGINT`，Go 里常用 `int64`。
- 权限位、状态位可能使用 `uint64`。
- 短链接、迁移脚本可能遇到 16 进制或 36 进制字符串。

```text
ID、权限、库存这类字段，不仅要能解析，还要明确范围。
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
	userID, err := strconv.ParseInt("922337203685477580", 10, 64)
	if err != nil {
		fmt.Println("解析 user_id 失败:", err)
		return
	}

	roleMask, err := strconv.ParseUint("15", 10, 64)
	if err != nil {
		fmt.Println("解析权限位失败:", err)
		return
	}

	fmt.Println("userID =", userID)
	fmt.Println("roleMask =", roleMask)
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
	userID, err := strconv.ParseInt("922337203685477580", 10, 64)
	if err != nil {
		fmt.Println("解析 user_id 失败:", err)
		return
	}

	roleMask, err := strconv.ParseUint("15", 10, 64)
	if err != nil {
		fmt.Println("解析权限位失败:", err)
		return
	}

	fmt.Println("userID =", userID)
	fmt.Println("roleMask =", roleMask)
}
EOF
go run main.go
```

## 三、关键参数/语法/代码结构

`strconv.ParseInt(s, base, bitSize)`：

- `s`：待解析字符串。
- `base`：进制，常用 `10`。如果传 `0`，Go 会根据前缀识别，比如 `0x10`。
- `bitSize`：结果范围，常用 `64`。如果传 `32`，超过 `int32` 范围会报错。

`strconv.ParseUint(s, base, bitSize)`：

- 用于无符号整数。
- 不能解析负数。
- 适合权限位、位图、纯非负计数类字段。

## 四、真实后端场景示例

下面是一个解析订单 ID 的函数。订单 ID 来自 path 参数，数据库字段是 `BIGINT`。

```go
package main

import (
	"fmt"
	"strconv"
)

func parseOrderID(raw string) (int64, error) {
	if raw == "" {
		return 0, fmt.Errorf("order_id 不能为空")
	}

	id, err := strconv.ParseInt(raw, 10, 64)
	if err != nil {
		return 0, fmt.Errorf("order_id 必须是 int64 范围内的十进制整数")
	}
	if id <= 0 {
		return 0, fmt.Errorf("order_id 必须大于 0")
	}
	return id, nil
}

func main() {
	id, err := parseOrderID("10000000001")
	if err != nil {
		fmt.Println("参数错误:", err)
		return
	}

	fmt.Println("准备查询订单:", id)
}
```

真实项目中，下一步查询数据库时应使用参数化查询。例如：

```go
// db.QueryContext(ctx, "SELECT id, amount FROM orders WHERE id = ?", id)
```

不要因为已经解析成数字，就改成字符串拼接 SQL。

## 五、注意点

`base=10` 是后端接口最常见选择，因为 API 参数应该清晰、稳定、容易排查。

`base=0` 虽然方便，但接口参数里一般不建议使用。因为 `"010"`、`"0x10"` 这类输入会让排查变复杂，也容易让不同客户端产生理解差异。

```text
对外接口优先使用明确的十进制，不要让客户端猜你的进制规则。
```

如果业务字段不能为负数，可以用 `ParseUint`，但很多数据库驱动和业务模型仍使用 `int64`。是否用 `uint64` 要看项目约定，不要只因为“它不能为负”就到处换成 `uint64`。

## 六、常见误区

- 误区一：`bitSize` 随便写。  
  `bitSize` 决定允许范围。库存如果最终保存到 `int32`，解析时就应该按业务范围校验，而不是解析成 `int64` 后直接写入。

- 误区二：用 `ParseUint` 解析所有非负数。  
  很多 Go 标准库和数据库接口更常用 `int64`，无脑使用 `uint64` 可能导致类型转换越来越多。

- 误区三：对外 API 使用 `base=0`。  
  这会让 `"0x10"` 被接受，增加接口不确定性。除非你的接口明确支持多进制。

## 七、本节达标标准

- 能用 `ParseInt` 解析 `int64` ID。
- 能解释 `base=10` 和 `base=0` 的区别。
- 能解释 `bitSize=32` 和 `bitSize=64` 的业务意义。
- 能为 ID、权限位、库存设置范围校验。
- 知道参数化查询仍然是数据库安全底线。
