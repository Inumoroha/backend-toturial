# 09. Builder 构建响应文本和 SQL 片段

本节目标：你能在循环构造较长字符串时使用 `strings.Builder`，并知道构造 SQL 片段时哪些地方必须保持安全边界。

简短引入：少量字符串拼接用 `+` 就可以；如果在循环里不断追加，`strings.Builder` 更清晰，也更适合控制性能。后端常见场景包括生成响应文本、导出内容、拼接日志摘要、构造白名单 SQL 片段。

## 一、为什么需要它

可以理解为：`Builder` 是一个专门用来逐步写字符串的容器。

常见场景是：

- 构造多行错误报告。
- 生成简单文本响应。
- 组装已校验字段的 SQL 排序片段。
- 生成导出文件中的文本块。

```text
Builder 解决的是字符串构造效率，不解决 SQL 注入。
```

## 二、基本用法

```go
package main

import (
	"fmt"
	"strings"
)

func main() {
	var b strings.Builder
	b.WriteString("用户导入结果\n")
	b.WriteString("成功: 10\n")
	b.WriteString("失败: 2\n")

	fmt.Println(b.String())
}
```

Windows PowerShell：

```powershell
go run main.go
```

Linux/macOS：

```bash
go run main.go
```

这个命令用于验证构造结果。真实项目中，`Builder` 常放在报表生成、导出、错误汇总这类逻辑里。

## 三、关键代码结构

常用方法：

- `WriteString`：追加字符串。
- `WriteByte`：追加单个字节，比如换行。
- `String`：取最终字符串。
- `Grow`：提前预估容量，减少扩容。

初学阶段不需要过度优化。只有当你在循环里构造较大字符串，或者这段逻辑处于热点路径时，再优先考虑 `Builder`。

## 四、真实后端场景示例

下面根据白名单构造排序 SQL 片段：

```go
package main

import (
	"errors"
	"fmt"
	"strings"
)

var allowedSortFields = map[string]string{
	"created_at": "created_at",
	"price":      "price",
}

func BuildOrderBy(sortField, direction string) (string, error) {
	column, ok := allowedSortFields[sortField]
	if !ok {
		return "", errors.New("invalid sort field")
	}

	direction = strings.ToUpper(strings.TrimSpace(direction))
	if direction != "ASC" && direction != "DESC" {
		return "", errors.New("invalid sort direction")
	}

	var b strings.Builder
	b.WriteString(" ORDER BY ")
	b.WriteString(column)
	b.WriteByte(' ')
	b.WriteString(direction)
	return b.String(), nil
}

func main() {
	orderBy, err := BuildOrderBy("price", "desc")
	if err != nil {
		fmt.Println("排序错误:", err)
		return
	}
	fmt.Println("SELECT * FROM orders WHERE user_id = ?"+orderBy)
}
```

这里故意只允许白名单字段。真实项目中，`user_id` 这样的值必须通过参数传入，不能拼接。

## 五、注意点

SQL 中有两类内容：

- 值：比如用户 ID、邮箱、订单号，必须参数化。
- 结构：比如排序字段、排序方向，参数化通常不支持，要用白名单映射。

```text
SQL 值走参数化，SQL 结构走白名单，二者不要混用。
```

涉及迁移脚本时，不要用程序随意拼复杂 SQL 后直接跑生产。更保守的做法是使用迁移工具，保留 up/down 脚本，先在测试环境验证，并确认可回滚。

## 六、常见误区

误区一：所有拼接都换成 `Builder`。

两三个字符串简单拼接用 `+` 更直观，不需要过度优化。

误区二：用 `Builder` 拼用户输入进 SQL。

这仍然是 SQL 注入风险，和使用什么拼接工具无关。

误区三：排序字段不做白名单。

排序字段通常无法作为普通参数绑定，必须映射到后端允许的固定列名。

## 七、本节达标标准

- 能用 `strings.Builder` 循环构造字符串。
- 能判断什么时候普通 `+` 拼接就够。
- 能说明 Builder 不提供安全能力。
- 能区分 SQL 值参数化和 SQL 结构白名单。

