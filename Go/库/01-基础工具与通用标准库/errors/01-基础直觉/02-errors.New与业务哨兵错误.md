# 02. errors.New 与业务哨兵错误

本节目标：学完后，你能够定义可被上层稳定识别的业务错误，比如用户不存在、权限不足、订单已支付。

上一节里我们直接写了 `errors.New("user not found")`。这能运行，但在真实后端项目里还不够。因为调用方经常需要判断“这是不是用户不存在”，然后返回 404。如果每次都创建一个新的错误值，判断会变得不稳定。

## 一、为什么需要它

可以把业务错误理解为服务内部约定好的信号。

用户不存在不是系统崩溃，权限不足也不是数据库坏了。它们都是业务流程中可以预期的失败。真实项目中，handler 需要根据这些失败返回不同状态码：

```text
用户不存在 -> 404
权限不足 -> 403
参数错误 -> 400
系统错误 -> 500
```

所以这些错误不能只是随手写出来的一段字符串，而应该是代码中稳定存在的错误变量。

## 二、基本用法

复制下面代码到 `main.go`。

```go
package main

import (
    "errors"
    "fmt"
)

var ErrUserNotFound = errors.New("user not found")

func findUser(id int64) error {
    if id == 404 {
        return ErrUserNotFound
    }

    return nil
}

func main() {
    err := findUser(404)
    if err != nil {
        if errors.Is(err, ErrUserNotFound) {
            fmt.Println("return HTTP 404")
            return
        }

        fmt.Println("return HTTP 500")
    }
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

输出：

```text
return HTTP 404
```

## 三、关键代码结构

```go
var ErrUserNotFound = errors.New("user not found")
```

这类定义在包级别的错误变量，常被称为 sentinel error，也就是“哨兵错误”。可以理解为一个稳定的错误标记。

```go
return ErrUserNotFound
```

当业务确实发生“用户不存在”时，返回这个固定变量。

```go
errors.Is(err, ErrUserNotFound)
```

调用方使用 `errors.Is` 判断它。现在你只需要先记住：判断标准错误时，优先使用 `errors.Is`，后面会专门讲为什么它比 `==` 更适合项目代码。

## 四、真实后端场景示例

假设我们有一个文章接口：`GET /articles/{id}`。文章不存在时应该返回 404，而不是 500。

```go
package main

import (
    "errors"
    "fmt"
)

var ErrArticleNotFound = errors.New("article not found")

func findArticle(id int64) error {
    if id == 100 {
        return ErrArticleNotFound
    }

    return nil
}

func statusFromError(err error) int {
    if errors.Is(err, ErrArticleNotFound) {
        return 404
    }

    return 500
}

func main() {
    err := findArticle(100)
    if err != nil {
        fmt.Println("status:", statusFromError(err))
        return
    }

    fmt.Println("status: 200")
}
```

真实项目中，这类错误通常放在业务包里，例如：

```text
internal/user/errors.go
internal/article/errors.go
internal/order/errors.go
```

这样 handler、service、测试代码都能复用同一个错误变量。

## 五、注意点

不是所有错误都要定义成包级变量。只有上层需要识别的错误，才适合定义成 sentinel error。

适合定义：

- `ErrUserNotFound`
- `ErrPermissionDenied`
- `ErrOrderAlreadyPaid`
- `ErrShortLinkExpired`

不适合定义：

- 临时的格式化错误
- 只会在当前函数内部处理的错误
- 带有大量动态字段的错误

```text
只有需要被上层判断的错误，才值得定义成稳定的业务错误变量。
```

## 六、常见误区

误区一：每次都创建相同文本的新错误。

```go
return errors.New("user not found")
```

如果调用方也写一个 `errors.New("user not found")` 来比较，它们不是同一个错误值。错误文本一样，不代表错误值一样。

误区二：用字符串判断错误。

```go
if err.Error() == "user not found" {
    return 404
}
```

这很脆弱。错误文案一改，判断就失效。真实项目中错误文案可能为了日志可读性调整，不应该让业务判断依赖字符串。

误区三：把所有错误都定义成全局变量。

这样会让错误列表越来越大，却没有实际价值。先从需要映射状态码、需要写测试断言、需要跨层识别的错误开始定义。

## 七、本节达标标准

- 能使用 `errors.New` 定义包级业务错误。
- 能解释什么情况下需要 sentinel error。
- 能用 `errors.Is` 判断业务错误。
- 能避免用错误字符串做业务判断。
- 能为用户、文章、订单等模块设计简单的业务错误变量。
