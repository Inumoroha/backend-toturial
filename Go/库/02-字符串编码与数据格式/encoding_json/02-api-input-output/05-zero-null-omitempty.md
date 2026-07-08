# 05. 区分零值、空值和未传字段

本节目标：学完后你能在更新接口中区分“字段没传”“字段传了空值”和“字段传了零值”。

简短引入：`omitempty` 看起来方便，但在后端更新接口里很容易踩坑。比如库存改成 `0`、昵称改成空字符串、手机号清空，这些动作都可能是合法业务。

## 一、为什么需要它

Go 有零值，JSON 有 `null`，HTTP 请求还可能完全不传某个字段。它们在业务上不一定相同。

以更新用户资料为例：

- 没传 `nickname`：不修改昵称。
- 传 `"nickname": ""`：把昵称改成空字符串，可能允许，也可能不允许。
- 传 `"nickname": null`：表示清空昵称，是否允许要看业务。

```text
更新接口最怕把“没传字段”误判成“传了零值”。
```

## 二、基本用法

Windows PowerShell：

```powershell
mkdir json-zero-null
cd json-zero-null
go mod init json-zero-null
notepad main.go
go run .
```

Linux/macOS：

```bash
mkdir json-zero-null
cd json-zero-null
go mod init json-zero-null
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

type UpdateInventoryRequest struct {
	Stock *int `json:"stock"`
}

func main() {
	cases := []string{
		`{}`,
		`{"stock":0}`,
		`{"stock":20}`,
	}

	for _, body := range cases {
		var req UpdateInventoryRequest
		if err := json.Unmarshal([]byte(body), &req); err != nil {
			panic(err)
		}

		if req.Stock == nil {
			fmt.Println(body, "=> stock not provided")
			continue
		}
		fmt.Println(body, "=> update stock to", *req.Stock)
	}
}
```

输出：

```text
{} => stock not provided
{"stock":0} => update stock to 0
{"stock":20} => update stock to 20
```

## 三、关键参数/语法/代码结构

`Stock *int` 使用指针表示字段是否出现。`nil` 可以理解为“没有提供值”，非 `nil` 表示客户端传了这个字段。

`0` 是 `int` 的零值，但它不一定代表没传。库存为 0、价格为 0、权限开关为 `false`，都可能是明确的业务值。

`omitempty` 主要影响编码输出，不应该盲目用来设计更新语义。

## 四、真实后端场景示例

文章更新接口通常是局部更新。客户端只传要改的字段。

```go
type UpdateArticleRequest struct {
	Title   *string `json:"title"`
	Summary *string `json:"summary"`
	Status  *string `json:"status"`
}

func applyArticlePatch(req UpdateArticleRequest) {
	if req.Title != nil {
		fmt.Println("update title to:", *req.Title)
	}
	if req.Summary != nil {
		fmt.Println("update summary to:", *req.Summary)
	}
	if req.Status != nil {
		fmt.Println("update status to:", *req.Status)
	}
}
```

真实项目中，更新数据库时不要拼接用户输入。可以根据哪些字段非 `nil` 构造白名单更新列表，并使用参数化查询。

```text
动态更新 SQL 可以动态选择字段，但字段名必须来自服务端白名单，字段值必须参数化。
```

## 五、注意点

如果你需要区分 `null` 和未传字段，普通指针还不够，因为 `{"stock":null}` 和 `{}` 都会让 `Stock` 为 `nil`。这时可以使用自定义类型记录字段是否出现，或者使用 `map[string]json.RawMessage` 做第一层检测。

初学阶段可以先掌握指针字段解决“零值和未传”的问题。等遇到 PATCH、清空字段这类需求时，再引入更明确的可选字段类型。

数据库更新要考虑事务和回滚。比如订单状态更新同时影响库存、优惠券和日志，就不能只更新一张表后直接返回成功。

## 六、常见误区

误区一：用普通 `int` 接收可选库存字段。这样 `{}` 和 `{"stock":0}` 都变成 0，业务上无法区分。

误区二：响应结构体到处使用 `omitempty`。库存 0、权限 `false`、金额 0 这类字段被省略后，会让客户端误判。

误区三：把 `null` 当成所有字段的清空指令。真实项目中清空要有明确权限和业务规则。

误区四：动态更新 SQL 时把字段名和字段值都直接拼进去。字段名要白名单，字段值要参数化。

## 七、本节达标标准

- 能解释零值、`null` 和未传字段的差异。
- 能用指针字段区分“没传”和“传了零值”。
- 能判断哪些字段不适合使用 `omitempty`。
- 能说明 PATCH 类更新接口为什么要谨慎设计字段语义。
- 能知道动态更新数据库时要使用字段白名单和参数化查询。

