# 2. Map 与 Set

本节目标：你能用 map 做快速查询，并用 map 模拟 set 完成去重和权限判断。

简短引入：后端项目里最常见的查找方式是“根据 ID 找对象”。用户 ID 找用户、订单号找订单、权限编码判断是否拥有权限，这些都适合用 map。

## 一、为什么需要它

切片查找通常要从头扫到尾，数据多了会慢。map 可以理解为一本按 key 编好的索引，常见场景是把数据库查出的用户列表转成 `map[int]User`，后续组装订单响应时快速补充用户信息。

## 二、基本用法

```powershell
mkdir ds-map-demo; cd ds-map-demo
go mod init ds-map-demo
notepad main.go
go run .
```

```bash
mkdir ds-map-demo && cd ds-map-demo
go mod init ds-map-demo
cat > main.go
go run .
```

`main.go`：

```go
package main

import "fmt"

type User struct {
	ID   int
	Name string
}

func main() {
	users := []User{{1, "Alice"}, {2, "Bob"}}
	byID := make(map[int]User)

	for _, user := range users {
		byID[user.ID] = user
	}

	user, ok := byID[2]
	fmt.Println(user, ok)

	permissions := map[string]bool{
		"article:create": true,
		"article:delete": false,
	}
	fmt.Println("can create:", permissions["article:create"])
}
```

## 三、关键参数/语法/代码结构

- `make(map[int]User)` 创建一个 key 为 int、value 为 User 的 map。
- `value, ok := m[key]` 可以区分 key 不存在和 value 是零值。
- Go 没有内置 set，常用 `map[string]struct{}` 或 `map[string]bool` 表示。
- map 遍历顺序不稳定，不能依赖它做固定展示。

## 四、真实后端场景示例

订单列表需要展示下单人名称。不要每个订单查一次用户，这会造成 N+1 查询。更常见的做法是批量查用户，再转 map：

```go
type Order struct {
	ID     int
	UserID int
}

func AttachUserName(orders []Order, users []User) map[int]string {
	userNameByID := make(map[int]string, len(users))
	for _, user := range users {
		userNameByID[user.ID] = user.Name
	}

	result := make(map[int]string, len(orders))
	for _, order := range orders {
		result[order.ID] = userNameByID[order.UserID]
	}
	return result
}
```

## 五、注意点

map 很适合内存索引，但生产中的唯一性仍要由数据库唯一索引兜底。比如订单号去重，内存 map 可以提前拦截重复请求，但最终必须依赖数据库唯一约束和事务。

```text
内存去重只能做加速，不能替代数据库唯一索引和幂等设计。
```

## 六、常见误区

- 误区一：并发读写普通 map。Go 的普通 map 不支持并发读写。
- 误区二：依赖 map 遍历顺序。接口返回列表要先排序。
- 误区三：只用 map 做幂等。服务重启后内存丢失，会造成重复处理。
- 误区四：忽略 `ok`，把“不存在”和“存在但值为空”混在一起。

## 七、本节达标标准

- 能用 map 按 ID 快速查找对象。
- 能用 map 模拟 set 做去重和权限判断。
- 能解释 N+1 查询为什么危险。
- 能说明内存 map 和数据库索引各自的边界。
