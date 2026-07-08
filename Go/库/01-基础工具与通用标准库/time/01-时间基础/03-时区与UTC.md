# 03. 时区与 UTC

本节目标：学完后，你能解释为什么后端存储时间常用 UTC，并能在展示时转换到用户所在时区。

简短引入：时区问题不是语法问题，而是线上问题。用户看到的时间、数据库存的时间、服务器日志里的时间如果没有统一规则，排查问题会非常痛苦。

## 一、为什么需要它

同一个时间点，在上海可能显示为上午 10 点，在 UTC 可能显示为凌晨 2 点。它们不是两个不同事件，只是同一时间点的不同展示方式。

真实项目中通常会：

- 数据库存 UTC，方便跨地区统一比较和排序。
- 接口展示时按用户时区转换。
- 日志中保留清晰时区，方便排查。

```text
存储时间看统一性，展示时间看用户体验。
```

## 二、基本用法

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	now := time.Now()
	utc := now.UTC()

	shanghai, err := time.LoadLocation("Asia/Shanghai")
	if err != nil {
		fmt.Println("load location:", err)
		return
	}

	fmt.Println("utc:", utc.Format(time.RFC3339))
	fmt.Println("shanghai:", utc.In(shanghai).Format("2006-01-02 15:04:05"))
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

`t.UTC()`：把时间转换成 UTC 展示。

`time.LoadLocation("Asia/Shanghai")`：加载时区。真实项目中，用户时区可能来自用户配置、租户配置或业务固定配置。

`t.In(loc)`：把同一个时间点按指定时区展示。

注意：`UTC()` 和 `In()` 通常不改变时间点本身，只改变展示时所使用的时区。

## 四、真实后端场景示例

下面模拟文章发布时间：数据库存 UTC，接口返回给上海用户时转成本地时间。

```go
package main

import (
	"fmt"
	"time"
)

type Article struct {
	ID          int64
	Title       string
	PublishedAt time.Time
}

func FormatForUser(t time.Time, timezone string) (string, error) {
	loc, err := time.LoadLocation(timezone)
	if err != nil {
		return "", fmt.Errorf("无效时区: %w", err)
	}
	return t.In(loc).Format("2006-01-02 15:04:05"), nil
}

func main() {
	article := Article{
		ID:          1,
		Title:       "Go 时间处理",
		PublishedAt: time.Date(2026, 7, 6, 2, 30, 0, 0, time.UTC),
	}

	text, err := FormatForUser(article.PublishedAt, "Asia/Shanghai")
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println(text)
}
```

## 五、注意点

不要在业务里混用“服务器本地时间”和“用户业务时间”。如果项目只服务中国用户，可以把业务时区固定为 `Asia/Shanghai`，但数据库依然建议统一 UTC。

如果接口接收的是用户输入的日期时间，例如活动开始时间，要明确这个时间属于哪个时区，再转换成 UTC 存储。

## 六、常见误区

- 认为 `UTC` 会让用户看到的时间不友好：UTC 主要用于存储，展示时可以转换。
- 服务器设置了东八区就不管时区：换服务器、容器镜像、云环境后可能不一致。
- 把没有时区的字符串直接当成绝对时间：`2026-07-06 10:00:00` 必须先知道它属于哪个时区。

## 七、本节达标标准

- 能说清楚存储时间和展示时间的区别。
- 能用 `UTC()` 和 `In()` 做时区转换。
- 能用 `LoadLocation` 加载业务时区。
- 知道数据库时间字段建议统一存 UTC。
