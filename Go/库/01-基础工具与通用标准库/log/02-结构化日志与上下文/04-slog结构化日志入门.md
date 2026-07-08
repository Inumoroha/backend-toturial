# 04. slog 结构化日志入门

本节目标：学完后，你能使用 Go 标准库 `log/slog` 输出带字段的结构化日志。

## 简短引入

前面使用 `log.Println("user_id", 1001)`，人能看懂，但日志平台不一定容易检索。结构化日志会把日志拆成字段，比如 `user_id=1001`、`order_id=9527`、`level=INFO`。真实后端服务通常更需要这种格式。

## 一、为什么需要它

可以理解为，结构化日志是给机器和人一起看的日志。常见场景是：

- 在日志平台搜索某个 `request_id`。
- 统计某个接口的错误数量。
- 按 `user_id` 查一次用户操作链路。
- 按 `order_id` 排查订单状态变化。

Go 标准库从 `log/slog` 提供结构化日志能力，适合作为后端项目的基础选择。

```text
真实项目中，重要信息尽量放到字段里，不要只拼在一整段字符串里。
```

## 二、基本用法

示例代码：

```go
package main

import (
	"log/slog"
	"os"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	logger.Info("user created",
		"user_id", 1001,
		"email", "alice@example.com",
	)
}
```

运行：

```bash
go run .
```

输出类似：

```json
{"time":"2026-07-06T10:40:00+08:00","level":"INFO","msg":"user created","user_id":1001,"email":"alice@example.com"}
```

## 三、关键参数/语法/代码结构

`slog.NewJSONHandler(os.Stdout, nil)` 表示输出 JSON 格式日志到标准输出。

`logger.Info` 的第一个参数是消息，后面是一组键值对：

```go
logger.Info("user created", "user_id", 1001)
```

键建议使用稳定的英文小写加下划线，例如：

- `user_id`
- `order_id`
- `request_id`
- `duration_ms`
- `error`

除了 JSON，也可以使用文本格式：

```go
logger := slog.New(slog.NewTextHandler(os.Stdout, nil))
```

开发环境可以用 Text，生产环境通常更推荐 JSON，方便日志平台解析。

## 四、真实后端场景示例

下面模拟“创建订单”的日志。

```go
package main

import (
	"errors"
	"log/slog"
	"os"
)

func createOrder(logger *slog.Logger, userID int64, skuID int64, quantity int) error {
	logger.Info("create order started",
		"user_id", userID,
		"sku_id", skuID,
		"quantity", quantity,
	)

	if quantity <= 0 {
		return errors.New("quantity must be positive")
	}

	logger.Info("create order success",
		"user_id", userID,
		"sku_id", skuID,
		"order_id", 9527,
	)
	return nil
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	if err := createOrder(logger, 1001, 2001, 0); err != nil {
		logger.Error("create order failed",
			"user_id", 1001,
			"sku_id", 2001,
			"error", err,
		)
	}
}
```

这段日志比字符串拼接更适合检索。比如你可以在日志系统里搜索 `user_id=1001` 或 `sku_id=2001`。

## 五、注意点

字段名要稳定。今天叫 `uid`，明天叫 `user_id`，后天叫 `userID`，日志查询会变得很痛苦。

错误字段建议统一叫 `error`。这样排查和告警时更容易写规则。

不要把结构化日志理解为“什么都往字段里塞”。字段应该是排查问题需要的关键事实，不是数据库整行记录。

## 六、常见误区

误区一：把字段写进消息里。  
比如 `logger.Info("user 1001 created")` 不如 `logger.Info("user created", "user_id", 1001)` 容易检索。

误区二：字段键值数量不成对。  
`slog` 可以处理这种情况，但输出会变难看，也说明代码不够严谨。

误区三：生产环境仍输出不稳定文本。  
纯文本适合人读，但日志平台聚合和过滤更适合 JSON。

## 七、本节达标标准

- 能使用 `slog.NewJSONHandler` 输出 JSON 日志。
- 能用键值对记录 `user_id`、`order_id`、`error` 等字段。
- 知道结构化日志比字符串拼接更适合检索。
- 能解释开发环境 Text 和生产环境 JSON 的取舍。

