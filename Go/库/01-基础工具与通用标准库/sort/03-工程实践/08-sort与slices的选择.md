# 08. sort 与 slices 的选择

本节目标：学完后，你能够在 `sort` 包和 `slices` 包之间做保守选择，并知道新版 Go 中更推荐的写法。

简短引入：`sort` 是老牌标准库，`slices` 是 Go 泛型之后更自然的切片工具。后端项目里两者都会遇到。

## 一、为什么需要它

旧项目常见 `sort.Slice`，新项目可能看到 `slices.SortFunc`。你不需要立刻重构所有旧代码，但需要读得懂，并能在新代码中选择清晰写法。

可以简单理解：

- `sort`：稳定、传统、项目里非常常见。
- `slices`：泛型友好，很多基础类型和结构体排序更直接。

## 二、基本用法

Go 1.21 及以上可以使用 `slices`：

```go
package main

import (
	"cmp"
	"fmt"
	"slices"
)

type User struct {
	ID   int
	Name string
}

func main() {
	users := []User{
		{ID: 2, Name: "bob"},
		{ID: 1, Name: "alice"},
	}

	slices.SortFunc(users, func(a, b User) int {
		return cmp.Compare(a.ID, b.ID)
	})

	fmt.Println(users)
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

`sort.Slice` 的比较函数返回 `bool`：

```go
return users[i].ID < users[j].ID
```

`slices.SortFunc` 的比较函数返回 `int`：

```go
return cmp.Compare(a.ID, b.ID)
```

通常约定：

- 返回负数：`a` 排在 `b` 前面。
- 返回 0：两者相等。
- 返回正数：`a` 排在 `b` 后面。

## 四、真实后端场景示例

权限菜单按权重升序，同权重按 ID 升序：

```go
package main

import (
	"cmp"
	"fmt"
	"slices"
)

type Menu struct {
	ID     int
	Name   string
	Weight int
}

func main() {
	menus := []Menu{
		{ID: 3, Name: "订单", Weight: 20},
		{ID: 1, Name: "首页", Weight: 10},
		{ID: 2, Name: "用户", Weight: 20},
	}

	slices.SortFunc(menus, func(a, b Menu) int {
		if n := cmp.Compare(a.Weight, b.Weight); n != 0 {
			return n
		}
		return cmp.Compare(a.ID, b.ID)
	})

	fmt.Println(menus)
}
```

这种写法的好处是，比较两个完整对象，不需要通过 `i`、`j` 下标反复访问切片，读起来更接近业务规则。

## 五、注意点

如果团队项目 Go 版本还低于 1.21，就不能直接使用 `slices` 标准库。真实项目中要先看 `go.mod` 的 Go 版本和部署环境。

不要为了追新把整个项目里的 `sort.Slice` 全改掉。保守做法是：新代码可以用 `slices`，旧代码保持稳定，除非你有测试覆盖和明确收益。

## 六、常见误区

- 误区一：看到 `slices` 就立刻重构旧代码。没有测试时风险大于收益。
- 误区二：混淆比较函数返回值。`sort.Slice` 返回 `bool`，`slices.SortFunc` 返回 `int`。
- 误区三：忽略 Go 版本。代码能在本机跑，不代表生产构建环境支持。

## 七、本节达标标准

- 能说明 `sort.Slice` 和 `slices.SortFunc` 的区别。
- 能用 `slices.SortFunc` 排序结构体切片。
- 能根据项目 Go 版本选择工具。
- 能保持对旧项目的保守重构态度。

