# 04. 用 Fprintf 写入 HTTP 响应和文件

本节目标：学会用 `fmt.Fprintf` 把格式化内容写入 `io.Writer`，理解它在 HTTP 和文件输出中的作用。

简短引入：Go 里很多输出目标都实现了 `io.Writer`，比如 HTTP 响应、文件、缓冲区。`Fprintf` 可以把同一套格式化能力写到这些目标里。

## 一、为什么需要它

后端开发不是只往控制台打印。常见场景是：

- 写 HTTP 文本响应。
- 写导出文件。
- 写迁移脚本的执行报告。
- 写测试中的内存缓冲区。

`Fprintf` 可以理解为“指定输出目的地的 `Printf`”。

## 二、基本用法

```go
package main

import (
	"fmt"
	"os"
)

func main() {
	fmt.Fprintf(os.Stdout, "service=%s status=%s\n", "user-api", "ok")
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

`fmt.Fprintf(w, format, args...)` 的第一个参数是 `io.Writer`。

它返回两个值：

```go
n, err := fmt.Fprintf(w, "hello %s", name)
```

`n` 是写入字节数，`err` 是写入错误。真实项目里，如果写文件或网络响应，错误不能随手忽略。

## 四、真实后端场景示例

下面写一个最小 HTTP 接口：

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "text/plain; charset=utf-8")
		fmt.Fprintf(w, "service=%s status=%s\n", "order-api", "ok")
	})

	log.Println("listen on :8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

运行后访问：

Windows PowerShell：

```powershell
go run .\main.go
Invoke-WebRequest http://localhost:8080/health
```

Linux/macOS：

```bash
go run ./main.go
curl http://localhost:8080/health
```

## 五、注意点

HTTP JSON 响应不要手写 `Fprintf` 拼 JSON，真实项目中应使用 `encoding/json`。手写容易漏转义，也容易让字段类型变得不稳定。

写文件时要检查错误，并在失败时让脚本可回滚。例如迁移前先备份、分批执行、记录已完成批次。

```text
迁移脚本的输出要能帮助回滚，而不是只告诉你“执行过了”。
```

如果要写一个简单的导出报告，可以先从 `os.Create` 和 `Fprintf` 开始：

```go
package main

import (
	"fmt"
	"os"
)

func main() {
	file, err := os.Create("order_report.txt")
	if err != nil {
		fmt.Println("create report:", err)
		return
	}
	defer file.Close()

	if _, err := fmt.Fprintf(file, "order_id=%s status=%s\n", "ORD-001", "paid"); err != nil {
		fmt.Println("write report:", err)
		return
	}

	fmt.Println("report generated")
}
```

真实项目中，如果导出结果会被其他系统消费，建议使用 CSV、JSON Lines 或数据库表，而不是随手定义文本格式。`Fprintf` 适合人工阅读的报告，不适合长期稳定的数据交换协议。

## 六、常见误区

忽略 `Fprintf` 的错误返回。写文件、网络连接中断时可能失败。

用 `Fprintf` 拼复杂响应。HTML 用模板，JSON 用编码器，SQL 用参数化。

在 HTTP Handler 里先写响应体再设置状态码。响应体写出后，状态码通常已经隐式变成 200。

## 七、本节达标标准

- 能说明 `Fprintf` 和 `Printf` 的区别。
- 能把格式化文本写入 `os.Stdout` 和 `http.ResponseWriter`。
- 知道写文件、网络输出时要检查错误。
- 知道 JSON、HTML、SQL 不应随意手写拼接。
