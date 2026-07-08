# 03. 用 Sprintf 构造业务文本

本节目标：学会用 `fmt.Sprintf` 生成字符串，并明确它不能替代安全的 SQL 参数化。

简短引入：`Printf` 是把内容写到标准输出，`Sprintf` 是把格式化结果返回成字符串。后端项目里，它常用于生成提示文本、缓存 Key、短链接路径、日志消息等。

## 一、为什么需要它

很多时候你不是马上打印，而是要把文本继续交给别的函数。例如：

- 生成缓存 Key：`user:1001:profile`
- 生成操作说明：`admin 7 disabled user 1001`
- 生成导出文件名：`orders_20260706.csv`

这时 `Sprintf` 比字符串拼接更清晰。

## 二、基本用法

```go
package main

import "fmt"

func main() {
	userID := 1001
	cacheKey := fmt.Sprintf("user:%d:profile", userID)

	fmt.Println(cacheKey)
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

`fmt.Sprintf(format, args...)` 返回一个字符串，不直接输出。

常见判断方式：

- 要输出到控制台：用 `Printf`。
- 要得到一个字符串继续使用：用 `Sprintf`。
- 要写到 `io.Writer`：用 `Fprintf`。

## 四、真实后端场景示例

下面生成一个短链接访问统计的缓存 Key：

```go
package main

import "fmt"

func main() {
	shortCode := "g7Kp2a"
	day := "20260706"

	key := fmt.Sprintf("shortlink:%s:visits:%s", shortCode, day)
	fmt.Println("redis key:", key)
}
```

真实项目中，这个 Key 可能会传给 Redis 客户端。`Sprintf` 的价值是让 Key 的结构一眼可见。

## 五、注意点

不要用 `Sprintf` 拼接 SQL 条件，尤其是包含用户输入的地方。

```text
用户输入不能直接拼进 SQL 字符串。
```

错误示例：

```go
sql := fmt.Sprintf("select * from users where name = '%s'", userInput)
```

正确方向是使用数据库驱动或 ORM 提供的参数化查询：

```go
rows, err := db.QueryContext(ctx, "select * from users where name = ?", userInput)
```

不同数据库占位符可能不同，例如 PostgreSQL 常见 `$1`。真实项目中还要配合超时控制、错误处理和索引评估。

`Sprintf` 更适合构造那些“不会被解析成代码或查询”的业务字符串。例如缓存 Key、审计摘要、文件名、短链接路径。即使如此，也要保证组成部分经过基本校验：

```go
package main

import (
	"fmt"
	"strings"
)

func buildExportFileName(module string, day string) (string, error) {
	if strings.Contains(module, "/") || strings.Contains(module, "\\") {
		return "", fmt.Errorf("invalid module name: %s", module)
	}
	return fmt.Sprintf("%s_%s.csv", module, day), nil
}

func main() {
	name, err := buildExportFileName("orders", "20260706")
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println(name)
}
```

这个例子看起来简单，但背后的习惯很重要：文件名、Key、路径片段都不要完全信任外部输入。真实项目中还会限制长度、字符集和是否覆盖已有文件。

## 六、常见误区

把 `Sprintf` 当模板引擎。复杂 HTML、邮件、报表应考虑 `html/template` 或专门模板方案。

拼 SQL。这样容易造成 SQL 注入，也会影响数据库执行计划复用。

缓存 Key 没有版本。真实项目中 Key 结构变化时，常见做法是加入版本前缀，例如 `v2:user:%d:profile`。

## 七、本节达标标准

- 能用 `Sprintf` 构造缓存 Key、文件名和业务提示。
- 能区分 `Printf` 和 `Sprintf`。
- 知道不能用 `Sprintf` 拼用户输入相关 SQL。
- 知道真实项目中查询应使用参数化。
