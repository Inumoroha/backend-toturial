# 06. Join 生成展示文本和内部标识

本节目标：你能用 `strings.Join` 把多个安全、已校验的字符串组合成展示文本、缓存 key 或日志字段。

简短引入：后端不仅要拆字符串，也经常要把多个片段拼回去。比如把标签展示成 `"go, backend"`，把缓存 key 组合成 `"user:42:profile"`，把日志字段组合成一行可读文本。

## 一、为什么需要它

可以理解为：`Join` 用统一分隔符把切片里的多个字符串连接起来。

常见场景是：

- 展示文章标签。
- 生成缓存 key。
- 生成日志中的简短摘要。
- 组合内部业务标识。

```text
Join 适合连接已经可信或已经校验过的片段，不负责替你做安全转义。
```

## 二、基本用法

```go
package main

import (
	"fmt"
	"strings"
)

func main() {
	tags := []string{"go", "backend", "api"}
	fmt.Println(strings.Join(tags, ", "))

	keyParts := []string{"user", "42", "profile"}
	fmt.Println(strings.Join(keyParts, ":"))
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

这个命令用于验证组合结果。真实项目中，缓存 key 或日志字段通常会由固定前缀加已校验 ID 组成。

## 三、关键代码结构

`strings.Join(elems, sep)`：

- `elems` 是字符串切片。
- `sep` 是元素之间的分隔符。
- 空切片返回空字符串。

例如：

```go
strings.Join([]string{}, ",")       // ""
strings.Join([]string{"a"}, ",")    // "a"
strings.Join([]string{"a", "b"}, ",") // "a,b"
```

后端里用 `Join` 时，先想清楚你是在生成哪一种字符串：

- 展示文本：给人看，允许更友好，例如标签之间用 `", "`。
- 内部标识：给程序用，格式要稳定，例如缓存 key 使用 `":"`。
- 日志摘要：给排查问题用，字段要少而清楚，敏感信息不要出现。

这三类字符串不要混在一起。展示文本以后可能改语言、改分隔符；内部标识一旦进入缓存、消息队列或数据库，就要尽量保持稳定。

## 四、真实后端场景示例

下面生成用户资料缓存 key：

```go
package main

import (
	"fmt"
	"strconv"
	"strings"
)

func UserProfileCacheKey(userID int64) string {
	return strings.Join([]string{
		"user",
		strconv.FormatInt(userID, 10),
		"profile",
	}, ":")
}

func main() {
	fmt.Println(UserProfileCacheKey(42))
}
```

真实项目中，缓存 key 的格式要稳定。改 key 格式会造成缓存整体失效，生产环境需要提前评估影响，必要时做双读或灰度迁移。

再看一个审计日志摘要的例子。这里只拼接已经清洗过、不会泄露隐私的字段：

```go
package main

import (
	"fmt"
	"strconv"
	"strings"
)

func AuditSummary(actorID int64, action string, resource string) string {
	parts := []string{
		"actor=" + strconv.FormatInt(actorID, 10),
		"action=" + strings.ToLower(strings.TrimSpace(action)),
		"resource=" + strings.ToLower(strings.TrimSpace(resource)),
	}
	return strings.Join(parts, " ")
}

func main() {
	fmt.Println(AuditSummary(42, "Delete", "Article"))
}
```

真实项目中，审计日志通常会优先使用结构化日志字段，而不是只有一整段字符串。这里的例子是为了让你理解：`Join` 适合把多个“已确定含义的片段”组合起来。

## 五、注意点

不要用 `Join` 拼 SQL 的用户输入：

```text
可以用 Join 拼白名单字段名，不可以用 Join 拼未校验的用户输入值。
```

例如批量查询 ID 时，不建议把用户传入的 ID 字符串直接 `Join` 成 `IN (...)`。更稳妥的方式是先把每个 ID 解析成整数，再使用数据库驱动或查询构造器的参数绑定能力。

生成缓存 key 时也要注意分隔符冲突。如果某个片段来自用户输入，里面可能自带 `:`，会让 key 变得难以解析。保守做法是缓存 key 尽量由固定前缀、数字 ID、枚举值组成，避免直接使用用户昵称、标题、搜索词。

## 六、常见误区

误区一：把 `Join` 当成 SQL 安全工具。

`Join` 只负责字符串连接，不做转义、不做参数化。

误区二：缓存 key 里混入未规范化字符串。

比如邮箱没有统一小写，会导致同一个用户出现多个缓存 key。

误区三：展示文本和内部标识混用。

展示文本可以更友好，内部标识要稳定、短、可预测。

误区四：缓存 key 没有版本。

如果 key 的业务含义未来可能变化，可以加入版本片段，例如 `user:v1:42:profile`。这样迁移时更容易灰度，不必一次性清空所有缓存。

## 七、本节达标标准

- 能用 `Join` 连接标签、缓存 key、日志字段。
- 能说明 `Join` 不负责安全转义。
- 能区分展示文本和内部标识。
- 能知道缓存 key 格式变更需要评估生产影响。
- 能判断哪些片段适合进入缓存 key，哪些片段应该先映射或拒绝。
