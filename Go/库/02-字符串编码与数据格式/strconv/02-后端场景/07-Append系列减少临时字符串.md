# 07. Append 系列减少临时字符串

本节目标：学完后，你能在字节缓冲区中使用 `AppendInt`、`AppendBool`、`AppendFloat`，减少高频路径上的临时字符串。

简短引入：大多数业务代码用 `FormatInt` 就够了。但在高频日志、协议编码、批量导出等场景中，如果你已经在操作 `[]byte`，`strconv.Append...` 可以直接把转换结果追加到字节切片里。

## 一、为什么需要它

可以理解为：`FormatInt` 会先生成字符串，`AppendInt` 可以直接写进已有的 `[]byte`。

常见场景是：

- 写入自定义访问日志。
- 批量导出 CSV 行。
- 构造 Redis 协议或内部轻量文本协议。
- HTTP 响应里手动写少量纯文本。

```text
先写清楚代码，再考虑性能；Append 系列适合已经确认的高频路径。
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
	buf := make([]byte, 0, 64)
	buf = append(buf, "user_id="...)
	buf = strconv.AppendInt(buf, 1001, 10)
	buf = append(buf, " active="...)
	buf = strconv.AppendBool(buf, true)

	fmt.Println(string(buf))
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
	buf := make([]byte, 0, 64)
	buf = append(buf, "user_id="...)
	buf = strconv.AppendInt(buf, 1001, 10)
	buf = append(buf, " active="...)
	buf = strconv.AppendBool(buf, true)

	fmt.Println(string(buf))
}
EOF
go run main.go
```

## 三、关键参数/语法/代码结构

`strconv.AppendInt(dst, i, base)`：把整数追加到 `dst` 后面。

`strconv.AppendBool(dst, b)`：追加 `true` 或 `false`。

`strconv.AppendFloat(dst, f, fmt, prec, bitSize)`：追加浮点数文本。

返回值必须接住：

```go
buf = strconv.AppendInt(buf, id, 10)
```

因为切片可能扩容，返回的新切片才是有效结果。

## 四、真实后端场景示例

下面构造一条简单访问日志。

```go
package main

import (
	"fmt"
	"strconv"
)

func accessLog(userID int64, status int, costMS int64) []byte {
	buf := make([]byte, 0, 96)
	buf = append(buf, "event=http_access user_id="...)
	buf = strconv.AppendInt(buf, userID, 10)
	buf = append(buf, " status="...)
	buf = strconv.AppendInt(buf, int64(status), 10)
	buf = append(buf, " cost_ms="...)
	buf = strconv.AppendInt(buf, costMS, 10)
	return buf
}

func main() {
	line := accessLog(1001, 200, 13)
	fmt.Println(string(line))
}
```

真实项目中，结构化日志库通常更合适。但如果你在写中间件、网关、批量导出工具，`Append` 系列会很实用。

## 五、注意点

不要为了“看起来高级”过早使用 `Append` 系列。普通 handler 里优先选择清晰代码。

选择建议：

- 拼少量字符串：`FormatInt` 或 `strings.Builder`
- 操作 `[]byte`：`AppendInt`
- JSON 响应：`encoding/json`
- 高性能日志：优先考虑成熟日志库

```text
性能优化要有场景和测量，不要用复杂代码换不确定的收益。
```

## 六、常见误区

- 误区一：忘记接收返回值。  
  `strconv.AppendInt(buf, id, 10)` 不赋回 `buf`，结果可能丢失。

- 误区二：所有拼接都改成 `Append`。  
  业务代码可读性更重要，只有高频路径才值得这样写。

- 误区三：把同一个 buffer 在多个 goroutine 里复用。  
  切片不是并发安全的。复用 buffer 要考虑生命周期和并发边界。

## 七、本节达标标准

- 能用 `AppendInt`、`AppendBool` 向 `[]byte` 追加内容。
- 知道返回的新切片必须赋值回来。
- 能判断何时用 `Append`，何时用 `Format`。
- 知道不要过早优化普通业务代码。
- 能写出一条简单的字节日志构造函数。
