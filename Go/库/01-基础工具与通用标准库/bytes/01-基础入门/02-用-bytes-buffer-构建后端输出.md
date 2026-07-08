# 02. 用 bytes.Buffer 构建后端输出

本节目标：学完后你能用 `bytes.Buffer` 构建响应文本、日志行和导出内容，避免在循环中粗暴拼接字符串。

## 简短引入

后端服务经常要“逐步生成一段内容”：比如订单导出、审计日志、通知消息、迁移脚本预览。`bytes.Buffer` 可以理解为一个可增长的字节容器，适合边写边生成。

## 一、为什么需要它

如果你在循环里频繁使用字符串 `+` 拼接，代码可能能跑，但不一定适合数据量变大后的场景。

`bytes.Buffer` 的优势是：

- 写入接口统一
- 可以写字符串，也可以写字节
- 最后一次性拿到 `[]byte` 或 `string`
- 很适合和 `io.Writer` 风格的代码配合

## 二、基本用法

`main.go`：

```go
package main

import (
	"bytes"
	"fmt"
)

func main() {
	var buf bytes.Buffer

	buf.WriteString("order_id,user_id,total\n")
	buf.WriteString("A1001,U7788,99.50\n")
	buf.WriteString("A1002,U8899,128.00\n")

	fmt.Print(buf.String())
}
```

运行：

Windows PowerShell：

```powershell
mkdir bytes-demo-02
cd bytes-demo-02
go mod init bytes-demo-02
notepad main.go
go run .
```

Linux/macOS：

```bash
mkdir bytes-demo-02
cd bytes-demo-02
go mod init bytes-demo-02
cat > main.go <<'EOF'
package main

import (
	"bytes"
	"fmt"
)

func main() {
	var buf bytes.Buffer

	buf.WriteString("order_id,user_id,total\n")
	buf.WriteString("A1001,U7788,99.50\n")
	buf.WriteString("A1002,U8899,128.00\n")

	fmt.Print(buf.String())
}
EOF
go run .
```

## 三、关键代码结构

`var buf bytes.Buffer`

声明一个空缓冲区。常见场景是函数内部临时生成内容。

`buf.WriteString(...)`

写入字符串。适合写固定模板、分隔符、换行符。

`buf.Write(...)`

写入 `[]byte`。适合把外部读到的字节继续拼到输出里。

`buf.String()`

拿到最终文本。适合返回 HTTP 文本响应、日志预览、命令行输出。

`buf.Bytes()`

拿到底层字节视图。后续章节会强调它的生命周期风险。

## 四、真实后端场景示例

假设后台要导出订单摘要，先生成 CSV 内容。

```go
package main

import (
	"bytes"
	"fmt"
)

type Order struct {
	ID     string
	UserID string
	Total  string
}

func buildOrderCSV(orders []Order) string {
	var buf bytes.Buffer
	buf.WriteString("order_id,user_id,total\n")

	for _, order := range orders {
		buf.WriteString(order.ID)
		buf.WriteByte(',')
		buf.WriteString(order.UserID)
		buf.WriteByte(',')
		buf.WriteString(order.Total)
		buf.WriteByte('\n')
	}

	return buf.String()
}

func main() {
	orders := []Order{
		{ID: "A1001", UserID: "U7788", Total: "99.50"},
		{ID: "A1002", UserID: "U8899", Total: "128.00"},
	}

	fmt.Print(buildOrderCSV(orders))
}
```

真实项目中，如果 CSV 字段可能包含逗号、换行、双引号，应优先使用 `encoding/csv`，而不是手写拼接。这里的例子是为了理解 `Buffer`。

```text
标准格式优先使用标准库解析和生成；bytes.Buffer 适合构建简单、可控的输出。
```

## 五、注意点

不要用 `bytes.Buffer` 拼接 SQL 并直接执行。

错误示例：

```go
sql := "select * from users where name = '" + userInput + "'"
```

保守做法：

```go
rows, err := db.QueryContext(ctx, "select * from users where name = ?", userInput)
```

`bytes.Buffer` 可以用于生成迁移脚本预览、日志内容、导出文本，但数据库执行必须走参数化查询。涉及多表写入时还要明确事务边界。

## 六、常见误区

误区一：把 `bytes.Buffer` 当全局变量复用。

全局复用容易造成并发污染。真实后端请求通常是并发处理，每个请求内部新建局部 `Buffer` 更稳。

误区二：`String()` 后继续修改缓冲区还以为结果不会变。

`String()` 返回的是当前内容的字符串结果。你应该把它当成一个输出点，不要让同一个缓冲区承载太多阶段。

误区三：为了性能所有拼接都改成 `bytes.Buffer`。

少量字符串拼接直接用 `+` 更清楚。`Buffer` 常用于循环、较大内容、需要 `io.Writer` 的场景。

## 七、本节达标标准

- 能用 `bytes.Buffer` 构建多行输出
- 能区分 `WriteString`、`Write`、`WriteByte`
- 知道简单 CSV 示例和真实 CSV 生成的区别
- 知道不能用拼接方式执行带用户输入的 SQL

