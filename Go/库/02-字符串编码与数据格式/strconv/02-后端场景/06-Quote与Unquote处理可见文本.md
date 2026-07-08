# 06. Quote 与 Unquote 处理可见文本

本节目标：学完后，你能用 `Quote` 和 `Unquote` 让日志、迁移脚本文本更可读，并知道它不能替代安全转义。

简短引入：后端日志里经常会记录用户输入，比如用户名、文章标题、搜索词。如果输入里有换行、制表符、引号，日志会变得难读。`strconv.Quote` 可以把字符串转成 Go 字符串字面量形式，让特殊字符可见。

## 一、为什么需要它

可以理解为：`Quote` 不是加密，也不是防注入，它只是让字符串在日志或脚本里变得“看得清”。

常见场景是：

- 日志中显示包含换行的标题。
- 迁移脚本中检查旧数据里的特殊字符。
- 调试 webhook payload 里的某个字段。

```text
Quote 让文本可见，但不负责 SQL、HTML、JSON 的安全。
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
	title := "第一行\n第二行"
	quoted := strconv.Quote(title)

	fmt.Println("原始标题:")
	fmt.Println(title)
	fmt.Println("日志标题:", quoted)

	unquoted, err := strconv.Unquote(quoted)
	if err != nil {
		fmt.Println("反解析失败:", err)
		return
	}
	fmt.Println("还原后是否相等:", unquoted == title)
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
	title := "第一行\n第二行"
	quoted := strconv.Quote(title)

	fmt.Println("原始标题:")
	fmt.Println(title)
	fmt.Println("日志标题:", quoted)

	unquoted, err := strconv.Unquote(quoted)
	if err != nil {
		fmt.Println("反解析失败:", err)
		return
	}
	fmt.Println("还原后是否相等:", unquoted == title)
}
EOF
go run main.go
```

## 三、关键参数/语法/代码结构

`strconv.Quote(s)`：返回带双引号的 Go 字符串字面量。

`strconv.Unquote(s)`：把合法的 Go 字符串字面量还原成普通字符串。

还有一些变体：

- `QuoteToASCII`：把非 ASCII 字符也转义显示。
- `QuoteRune`：处理单个 rune。
- `CanBackquote`：判断字符串能否用反引号字面量表示。

初学阶段先掌握 `Quote` 和 `Unquote` 即可。

## 四、真实后端场景示例

下面模拟记录文章标题更新日志。标题可能包含换行和引号。

```go
package main

import (
	"fmt"
	"strconv"
)

func auditArticleTitle(articleID int64, oldTitle, newTitle string) string {
	return "event=article_title_update article_id=" +
		strconv.FormatInt(articleID, 10) +
		" old_title=" + strconv.Quote(oldTitle) +
		" new_title=" + strconv.Quote(newTitle)
}

func main() {
	fmt.Println(auditArticleTitle(
		1001,
		"旧标题",
		"新标题\n带换行",
	))
}
```

这样日志里能看出标题中包含 `\n`，不会把一条日志拆成多行，排查问题更方便。

## 五、注意点

`Quote` 不等于 SQL 转义：

```go
// 错误方向：不要这样拼 SQL
// sql := "SELECT * FROM articles WHERE title = " + strconv.Quote(title)
```

正确方向是参数化查询：

```go
// db.QueryContext(ctx, "SELECT * FROM articles WHERE title = ?", title)
```

`Quote` 也不等于 HTML 转义。如果要输出到 HTML 页面，应使用模板库或 HTML 转义工具。如果要生成 JSON，应使用 `encoding/json`。

## 六、常见误区

- 误区一：把 `Quote` 当成防 SQL 注入工具。  
  它生成的是 Go 字符串字面量，不是数据库安全语法。

- 误区二：把 `Unquote` 用在普通用户输入上。  
  普通输入 `"abc"` 和 Go 字面量 `"\"abc\""` 不是一回事，乱用会让接口规则变复杂。

- 误区三：日志里直接打印原始多行文本。  
  多行文本会破坏日志结构，影响检索和告警。

## 七、本节达标标准

- 能用 `Quote` 让特殊字符在日志中可见。
- 能用 `Unquote` 还原合法字面量。
- 知道 `Quote` 不能替代 SQL 参数化查询。
- 知道 JSON、HTML 应使用专门编码或转义工具。
- 能判断什么时候需要让日志字段可见化。
