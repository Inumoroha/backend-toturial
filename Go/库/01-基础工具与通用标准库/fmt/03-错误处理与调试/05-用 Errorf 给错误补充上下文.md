# 05. 用 Errorf 给错误补充上下文

本节目标：学会用 `fmt.Errorf` 包装错误，让上层既能看到业务上下文，也能判断原始错误。

简短引入：后端项目里，错误不能只写 `return err`。调用链一长，你会不知道到底是查用户失败、扣库存失败，还是提交事务失败。`fmt.Errorf` 可以给错误补一句有用的话。

## 一、为什么需要它

可以把错误上下文理解为“案发地点”。底层错误告诉你发生了什么，上下文告诉你在哪个业务动作里发生。

真实项目中常见场景：

- 查询用户失败：带上 `userID`。
- 扣库存失败：带上 `sku` 和数量。
- 提交事务失败：带上业务操作名。

## 二、基本用法

```go
package main

import (
	"errors"
	"fmt"
)

var ErrUserNotFound = errors.New("user not found")

func findUser(id int) error {
	return fmt.Errorf("find user %d: %w", id, ErrUserNotFound)
}

func main() {
	err := findUser(1001)
	fmt.Println(err)
	fmt.Println(errors.Is(err, ErrUserNotFound))
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

`%w` 用来包装错误。被包装后，可以用 `errors.Is` 或 `errors.As` 在上层判断原始错误。

`%v` 只是把错误格式化成文本，不保留可判断的错误链。

```text
需要上层判断错误类型时，用 %w；只做展示时，才考虑 %v。
```

## 四、真实后端场景示例

下面模拟订单创建时扣库存失败：

```go
package main

import (
	"errors"
	"fmt"
)

var ErrStockNotEnough = errors.New("stock not enough")

func deductStock(sku string, count int) error {
	return ErrStockNotEnough
}

func createOrder(userID int, sku string, count int) error {
	if err := deductStock(sku, count); err != nil {
		return fmt.Errorf("create order user=%d sku=%s count=%d: %w", userID, sku, count, err)
	}
	return nil
}

func main() {
	err := createOrder(1001, "BOOK-GO-001", 3)
	if errors.Is(err, ErrStockNotEnough) {
		fmt.Println("show user friendly message: 库存不足")
	}
	fmt.Println("debug:", err)
}
```

真实项目中，如果扣库存和创建订单在一个事务里，要明确事务边界：失败时回滚，成功时提交。`fmt.Errorf` 只负责解释错误，不负责事务安全。

## 五、注意点

错误上下文要有业务定位价值，但不要塞敏感信息。例如不要把用户密码、Token、完整身份证号放进错误。

包装错误时不要重复很多层无意义文本。好的上下文通常包含动作、关键 ID、关键参数。

事务中常见顺序是：开始事务、执行业务 SQL、遇错回滚、提交失败也要返回错误。每一步的错误都应该带上下文。

## 六、常见误区

用 `%v` 包装后再希望 `errors.Is` 能判断。`%v` 不会建立错误链。

错误消息只写 “failed”。这对排查帮助很小。

在错误里暴露敏感数据。错误会进入日志，日志又可能被更多人看到。

忽略提交事务的错误。提交失败同样可能导致业务不一致，必须返回并记录。

## 七、本节达标标准

- 能用 `fmt.Errorf("...: %w", err)` 包装错误。
- 能用 `errors.Is` 判断被包装的原始错误。
- 能给错误补充有用的业务上下文。
- 知道错误包装不替代事务回滚和生产日志。

