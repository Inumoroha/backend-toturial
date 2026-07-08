# 07. 使用 sort.Search 做阈值查找

本节目标：学完后，你能够在已排序数据中快速找到第一个满足条件的位置，例如库存阈值或费率档位。

简短引入：`sort.Search` 不是用来排序的，而是在有序数据上做查找。它常用于配置档位、库存阈值、时间窗口定位。

## 一、为什么需要它

假设系统有优惠门槛：满 100 减 10，满 300 减 40，满 500 减 80。用户订单金额是 320，我们想找到第一个大于等于 320 的门槛，或者找到不超过它的最大门槛。

数据量小时循环也能做，但 `sort.Search` 能帮你建立“有序数据查找”的习惯。

## 二、基本用法

创建 `main.go`：

```go
package main

import (
	"fmt"
	"sort"
)

func main() {
	thresholds := []int{100, 300, 500}
	amount := 320

	idx := sort.Search(len(thresholds), func(i int) bool {
		return thresholds[i] >= amount
	})

	if idx == len(thresholds) {
		fmt.Println("没有找到大于等于订单金额的门槛")
		return
	}
	fmt.Println("第一个大于等于订单金额的门槛:", thresholds[idx])
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

`sort.Search(n, f)` 会在 `[0, n)` 范围里找第一个让 `f(i)` 返回 `true` 的位置。

可以理解为：

- 前面一段都是 `false`。
- 后面一段都是 `true`。
- 它返回第一个 `true` 的下标。

```text
sort.Search 的前提是判断条件在有序数据上能形成 false 到 true 的分界。
```

## 四、真实后端场景示例

库存系统里，你想根据库存数量找到风险等级：

```go
package main

import (
	"fmt"
	"sort"
)

type StockLevel struct {
	Limit int
	Name  string
}

func main() {
	levels := []StockLevel{
		{Limit: 10, Name: "紧急"},
		{Limit: 50, Name: "偏低"},
		{Limit: 200, Name: "正常"},
	}

	stock := 42
	idx := sort.Search(len(levels), func(i int) bool {
		return stock <= levels[i].Limit
	})

	if idx == len(levels) {
		fmt.Println("库存充足")
		return
	}
	fmt.Println("库存等级:", levels[idx].Name)
}
```

这里的 `levels` 必须按 `Limit` 升序。真实项目中，配置从数据库或配置中心读出来后，建议启动时校验并排序一次。

## 五、注意点

如果数据未排序，`sort.Search` 不会报错，但结果没有业务意义。初学时可以先用 `sort.SliceIsSorted` 或手动校验。

配置变更也要谨慎。比如库存等级阈值变更，最好通过配置发布流程，有审核、回滚和日志记录，而不是随手改线上数据。

## 六、常见误区

- 误区一：在未排序数据上使用 `sort.Search`。它依赖有序性。
- 误区二：判断函数不是从 `false` 到 `true`。条件不单调时，结果不可预测。
- 误区三：忘记处理 `idx == len(slice)`。这表示没有找到满足条件的元素。

## 七、本节达标标准

- 能解释 `sort.Search` 返回的是什么。
- 能写出第一个大于等于目标值的查找。
- 能处理未找到的情况。
- 能说明为什么输入数据必须有序。

