# 02. 用结构体标签稳定接口字段

本节目标：学完后你能用 `json` 标签控制接口字段名、忽略字段和空值输出。

简短引入：Go 结构体字段通常用大驼峰命名，例如 `UserID`；JSON 接口字段通常用小写或下划线命名，例如 `user_id`。结构体标签就是 Go 字段和 JSON 字段之间的翻译表。

## 一、为什么需要它

如果不写标签，`encoding/json` 会直接使用 Go 字段名。比如 `UserID` 会输出成 `"UserID"`。这对后端代码没问题，但前端接口通常希望字段稳定、简洁、符合约定。

结构体标签的价值是让内部代码风格和外部接口契约分开。

```text
接口字段名一旦被客户端使用，就不能随意改。
```

## 二、基本用法

Windows PowerShell：

```powershell
mkdir json-tags
cd json-tags
go mod init json-tags
notepad main.go
go run .
```

Linux/macOS：

```bash
mkdir json-tags
cd json-tags
go mod init json-tags
nano main.go
go run .
```

`main.go`：

```go
package main

import (
	"encoding/json"
	"fmt"
)

type Article struct {
	ID        int    `json:"id"`
	Title     string `json:"title"`
	AuthorID  int    `json:"author_id"`
	DraftNote string `json:"-"`
	Summary   string `json:"summary,omitempty"`
}

func main() {
	article := Article{
		ID:        10,
		Title:     "Go JSON 实战",
		AuthorID:  1001,
		DraftNote: "内部备注，不给前端",
	}

	data, err := json.Marshal(article)
	if err != nil {
		panic(err)
	}

	fmt.Println(string(data))
}
```

输出：

```text
{"id":10,"title":"Go JSON 实战","author_id":1001}
```

## 三、关键参数/语法/代码结构

`json:"id"` 表示 JSON 字段名使用 `id`。

`json:"author_id"` 表示把 Go 的 `AuthorID` 输出成 JSON 的 `author_id`。

`json:"-"` 表示这个字段完全不参与 JSON 编解码。常见场景是密码、内部备注、计算中间值。

`json:"summary,omitempty"` 表示当 `Summary` 是零值时不输出。字符串零值是 `""`，数字零值是 `0`，布尔零值是 `false`，切片和指针零值是 `nil`。

## 四、真实后端场景示例

下面是文章列表接口常见响应。列表接口通常不返回正文，只返回摘要字段，减少响应体大小。

```go
package main

import (
	"encoding/json"
	"fmt"
)

type ArticleListItem struct {
	ID        int    `json:"id"`
	Title     string `json:"title"`
	AuthorID  int    `json:"author_id"`
	Summary   string `json:"summary,omitempty"`
	CreatedAt string `json:"created_at"`
}

func main() {
	items := []ArticleListItem{
		{
			ID:        10,
			Title:     "Go JSON 实战",
			AuthorID:  1001,
			CreatedAt: "2026-07-06T15:00:00+08:00",
		},
	}

	data, err := json.Marshal(items)
	if err != nil {
		panic(err)
	}

	fmt.Println(string(data))
}
```

真实项目中，字段名应该由接口文档、前端约定或 OpenAPI 规范稳定下来。不要因为 Go 里字段改名，就顺手改 JSON 字段名。

## 五、注意点

`omitempty` 适合“没有值就不返回”的字段，但不适合所有场景。比如库存数量是 `0`，这可能是重要信息，不能因为 `omitempty` 被省略。

字段标签只影响 JSON 编解码，不会自动完成业务校验。比如 `email` 字段有标签，也不代表它一定是合法邮箱。

如果字段涉及数据库列名，不要误以为 `json` 标签会影响 SQL。数据库映射通常由 ORM 标签或 SQL 代码控制。

## 六、常见误区

误区一：给所有字段都加 `omitempty`。这样会让响应缺字段，前端难以区分“没有返回”和“值就是 0”。

误区二：使用 `json:"-"` 隐藏字段后，以为业务层也拿不到。标签只影响 JSON，结构体内部仍然有这个字段。

误区三：接口已经上线后随意改标签名。客户端会按旧字段读取，改名可能直接造成线上兼容问题。

误区四：把 `json` 标签当成数据库字段标签。JSON 是接口边界，数据库是持久化边界，两者要分清。

## 七、本节达标标准

- 能使用 `json` 标签控制输出字段名。
- 能用 `json:"-"` 防止敏感字段进入 JSON。
- 能解释 `omitempty` 的适用场景和风险。
- 能判断哪些字段不应该因为零值被省略。
- 能理解 JSON 字段名属于接口契约，不能随意改。

