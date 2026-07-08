# 04. Cut 与 Index 解析业务字符串

本节目标：你能从 `key=value`、`Bearer token`、`order:123` 这类业务字符串中稳定取出需要的部分。

简短引入：真实后端里经常会遇到“一个字符串里包含两段信息”的情况。以前很多人用 `Index` 找位置再切片，现在更推荐在简单分隔场景下优先使用 `strings.Cut`。

## 一、为什么需要它

可以理解为：`Cut` 负责按第一个分隔符把字符串切成“前半段”和“后半段”。

常见场景是：

- 解析配置：`timeout=3s`
- 解析权限：`article:read`
- 解析订单日志：`order_id=1001`
- 解析缓存 key：`user:42`

```text
能用 Cut 表达的简单拆分，就不要手写一堆下标运算。
```

## 二、基本用法

```go
package main

import (
	"fmt"
	"strings"
)

func main() {
	key, value, ok := strings.Cut("timeout=3s", "=")
	fmt.Println(key, value, ok)

	before, after, found := strings.Cut("user:42", ":")
	fmt.Println(before, after, found)
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

这个命令的作用是验证字符串解析结果。真实项目中，解析后通常还要把字符串转成业务类型，比如把 `"42"` 转成整数用户 ID。

## 三、关键代码结构

`strings.Cut(s, sep)` 返回三个值：

- `before`：分隔符前面的内容。
- `after`：分隔符后面的内容。
- `found`：是否真的找到了分隔符。

如果找不到分隔符，`found` 是 `false`，这时不要继续当成合法数据处理。

`Index` 适合你确实需要知道位置的场景：

```go
idx := strings.Index("article:read", ":")
```

但只是为了切成两段时，`Cut` 更清晰。

## 四、真实后端场景示例

下面解析一个权限字符串：

```go
package main

import (
	"errors"
	"fmt"
	"strings"
)

type Permission struct {
	Resource string
	Action   string
}

func ParsePermission(raw string) (Permission, error) {
	raw = strings.TrimSpace(raw)
	resource, action, ok := strings.Cut(raw, ":")
	if !ok {
		return Permission{}, errors.New("permission must be resource:action")
	}

	resource = strings.TrimSpace(resource)
	action = strings.TrimSpace(action)
	if resource == "" || action == "" {
		return Permission{}, errors.New("empty resource or action")
	}

	return Permission{Resource: resource, Action: action}, nil
}

func main() {
	p, err := ParsePermission(" article:read ")
	if err != nil {
		fmt.Println("权限错误:", err)
		return
	}
	fmt.Printf("%+v\n", p)
}
```

真实项目中，解析完成后还要检查资源和动作是否在白名单内。比如只允许 `article`、`order`、`user`，动作只允许 `read`、`write`、`delete`。

## 五、注意点

`Cut` 只按第一个分隔符切开：

```go
strings.Cut("a=b=c", "=") // "a", "b=c", true
```

如果你的业务要求只能出现一个分隔符，要额外检查 `after` 里是否还包含分隔符。

涉及数据库时要注意：

```text
解析出来的字段即使看起来合法，也不能直接拼进 SQL 条件。
```

权限资源名、排序字段、筛选字段如果要参与 SQL，应该使用白名单映射，而不是把用户传入值原样拼进去。

## 六、常见误区

误区一：忽略 `found`。

找不到分隔符时继续处理，会把非法输入当成正常输入。

误区二：手写下标切片不检查 `-1`。

`Index` 找不到会返回 `-1`，直接切片会 panic。

误区三：解析后不做白名单。

比如用户传入 `role:admin`，格式正确不代表权限合法。

## 七、本节达标标准

- 能用 `Cut` 解析 `key=value` 和 `resource:action`。
- 能正确处理找不到分隔符的情况。
- 能说出 `Cut` 和 `Index` 的适用区别。
- 能在解析后继续做业务白名单校验。

