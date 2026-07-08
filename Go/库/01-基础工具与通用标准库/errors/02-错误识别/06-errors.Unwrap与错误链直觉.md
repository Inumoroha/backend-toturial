# 06. errors.Unwrap 与错误链直觉

本节目标：学完后，你能够理解错误包装形成的错误链，并知道什么时候需要手动使用 `errors.Unwrap`。

前面我们已经用过 `fmt.Errorf("%w", err)`、`errors.Is` 和 `errors.As`。`errors.Unwrap` 不一定是你每天都要写的 API，但它能帮助你理解错误链为什么能被识别。

## 一、为什么需要它

可以把错误链理解为一层一层包装的说明：

```text
handler create order: service create order: repository insert order: connection refused
```

每一层都说明了一个业务动作。`errors.Is` 和 `errors.As` 会顺着这条链查找目标错误。`errors.Unwrap` 则是手动拆开一层。

真实项目中，你通常不需要自己循环拆错误链。学习它主要是为了理解：为什么使用 `%w` 后，上层还能识别底层错误。

## 二、基本用法

复制下面代码到 `main.go`。

```go
package main

import (
    "errors"
    "fmt"
)

func main() {
    baseErr := errors.New("connection refused")
    repoErr := fmt.Errorf("insert order: %w", baseErr)
    serviceErr := fmt.Errorf("create order: %w", repoErr)

    fmt.Println("full:", serviceErr)
    fmt.Println("unwrap 1:", errors.Unwrap(serviceErr))
    fmt.Println("unwrap 2:", errors.Unwrap(errors.Unwrap(serviceErr)))
}
```

运行命令：

Windows PowerShell：

```powershell
go run .\main.go
```

Linux/macOS：

```bash
go run ./main.go
```

输出类似：

```text
full: create order: insert order: connection refused
unwrap 1: insert order: connection refused
unwrap 2: connection refused
```

## 三、关键代码结构

```go
fmt.Errorf("create order: %w", repoErr)
```

这会创建一个包装错误。它的文本包含外层上下文，同时内部保存了 `repoErr`。

```go
errors.Unwrap(serviceErr)
```

这只拆开一层。它不会一次性返回最底层错误。

需要注意的是，`errors.Unwrap` 只调用错误的 `Unwrap() error` 方法。对于 `errors.Join` 产生的多错误，它内部是 `Unwrap() []error`，不能被 `errors.Unwrap` 直接拆成多个错误。多错误场景应该继续使用 `errors.Is`、`errors.As` 或自己针对 `Unwrap() []error` 做处理。

## 四、真实后端场景示例

下面模拟订单创建失败。我们不建议业务代码手动拆错误链来判断，但可以打印每一层帮助理解。

```go
package main

import (
    "errors"
    "fmt"
)

var ErrInventoryNotEnough = errors.New("inventory not enough")

func reserveInventory(orderID int64) error {
    return ErrInventoryNotEnough
}

func createOrder(orderID int64) error {
    if err := reserveInventory(orderID); err != nil {
        return fmt.Errorf("create order %d: reserve inventory: %w", orderID, err)
    }

    return nil
}

func main() {
    err := createOrder(1001)
    if err != nil {
        fmt.Println("log:", err)
        fmt.Println("is inventory not enough:", errors.Is(err, ErrInventoryNotEnough))

        for current := err; current != nil; current = errors.Unwrap(current) {
            fmt.Println("chain:", current)
        }
    }
}
```

真实项目中，上面的循环更多用于学习或调试。生产代码里，业务判断优先使用 `errors.Is` 和 `errors.As`，日志通常打印完整错误即可。

## 五、注意点

`errors.Unwrap` 是理解工具，不是业务判断的首选工具。

更推荐：

- 判断固定错误：`errors.Is`
- 提取错误类型：`errors.As`
- 打印排查信息：记录完整 `err`

只有在你自己实现错误类型、调试错误链、或写错误处理工具函数时，才更可能直接使用 `Unwrap`。

```text
业务代码不要依赖手动拆链顺序，应该依赖 errors.Is 和 errors.As 表达意图。
```

## 六、常见误区

误区一：以为 `Unwrap` 会拆到最底层。

它只拆一层。如果要继续拆，需要循环。

误区二：用 `Unwrap` 代替 `errors.Is`。

手动拆链更容易写错，也会让代码依赖错误包装顺序。业务判断应该更直接。

误区三：以为 `errors.Join` 可以用 `errors.Unwrap` 拆出每一个错误。

`errors.Join` 是多错误，它的解包形式不同。普通 `errors.Unwrap` 不会返回 `[]error`。

## 七、本节达标标准

- 能说明错误链是如何通过 `%w` 形成的。
- 能使用 `errors.Unwrap` 手动拆开一层包装错误。
- 能解释为什么业务判断优先用 `errors.Is` 和 `errors.As`。
- 能知道 `errors.Unwrap` 不适合直接处理 `errors.Join` 的多错误。
- 能读懂多层包装后的错误日志。
