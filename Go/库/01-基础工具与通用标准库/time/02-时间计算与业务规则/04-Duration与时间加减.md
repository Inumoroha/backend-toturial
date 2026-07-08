# 04. Duration 与时间加减

本节目标：学完后，你能用 `time.Duration` 实现订单超时、库存锁定、验证码过期这类时间长度逻辑。

简短引入：后端业务经常不是只关心“现在几点”，而是关心“从某个时间点开始过了多久”。这时就需要 `time.Duration`。

## 一、为什么需要它

订单 30 分钟未支付关闭、验证码 5 分钟有效、短链接 7 天后失效，这些都不是某一个固定时间，而是一段时间长度。

可以理解为：

- `time.Time` 表示点。
- `time.Duration` 表示长度。
- `time.Time.Add(duration)` 得到新的时间点。

## 二、基本用法

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	createdAt := time.Now()
	expiredAt := createdAt.Add(30 * time.Minute)

	fmt.Println("createdAt:", createdAt.Format("15:04:05"))
	fmt.Println("expiredAt:", expiredAt.Format("15:04:05"))
	fmt.Println("ttl:", time.Until(expiredAt).Round(time.Second))
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

`30 * time.Minute` 表示 30 分钟。真实项目中比直接写数字清楚得多。

`t.Add(d)`：在某个时间点上增加一段时间。

`time.Since(t)`：从 `t` 到现在经过了多久。

`time.Until(t)`：从现在到 `t` 还剩多久。

```text
不要用裸数字表达时间长度，优先写成 5*time.Minute 或 24*time.Hour。
```

## 四、真实后端场景示例

下面模拟订单是否超时：

```go
package main

import (
	"fmt"
	"time"
)

type Order struct {
	ID        int64
	CreatedAt time.Time
	PaidAt    *time.Time
}

func IsPaymentExpired(order Order, now time.Time) bool {
	if order.PaidAt != nil {
		return false
	}
	expiredAt := order.CreatedAt.Add(30 * time.Minute)
	return !now.Before(expiredAt)
}

func main() {
	order := Order{
		ID:        1001,
		CreatedAt: time.Now().Add(-31 * time.Minute),
	}

	fmt.Println(IsPaymentExpired(order, time.Now()))
}
```

这里把 `now` 作为参数传入，是为了后面更容易测试。真实项目中，订单关闭任务和接口查询都可以复用这段判断。

## 五、注意点

`24 * time.Hour` 表示准确的 24 小时，不一定等于业务上的“明天零点”。如果你要处理“当天结束”“下个月一号”，需要用日期规则，而不是简单加小时。

配置时间长度时，推荐用字符串配置再解析，例如 `"30m"`、`"24h"`：

```go
duration, err := time.ParseDuration("30m")
```

这样比配置 `1800` 更容易看懂，也减少单位混乱。

## 六、常见误区

- 写 `30` 表示 30 分钟：Go 会把它当成 30 纳秒，业务会直接错。
- 用 `time.Since(createdAt) > 30*time.Minute` 到处判断：业务规则分散后，后期改成 15 分钟很容易漏。
- 把自然日期规则都写成小时数：会员到期、账单日、月结通常不能只靠小时数。

## 七、本节达标标准

- 能用 `time.Duration` 表示分钟、小时、天级别的时间长度。
- 能用 `Add` 计算过期时间。
- 能用 `Since`、`Until` 判断经过时间和剩余时间。
- 知道业务规则要集中封装，避免散落在各处。
