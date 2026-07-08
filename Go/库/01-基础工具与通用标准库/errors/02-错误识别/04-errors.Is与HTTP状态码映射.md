# 04. errors.Is 与 HTTP 状态码映射

本节目标：学完后，你能够用 `errors.Is` 判断业务错误，并把它们转换成合适的 HTTP 状态码。

在后端接口中，同样是 `err != nil`，含义可能完全不同。用户不存在应该返回 404，权限不足应该返回 403，参数错误应该返回 400，数据库宕机才应该返回 500。`errors.Is` 的作用，就是让你在错误被多层包装以后，仍然能判断它到底是不是某个业务错误。

## 一、为什么需要它

真实项目中，错误通常不会从 repository 直接到 handler，中间会经过 service，甚至还会经过 usecase、middleware。每一层都可能用 `%w` 包装一次。

如果你直接比较：

```go
if err == ErrUserNotFound {
    return 404
}
```

一旦错误被包装，这个判断就会失败。`errors.Is` 会沿着错误链查找目标错误，适合用于业务错误判断。

```text
需要跨层识别的业务错误，用 errors.Is 判断，不要依赖字符串或直接相等。
```

## 二、基本用法

复制下面代码到 `main.go`。

```go
package main

import (
    "errors"
    "fmt"
)

var ErrUserNotFound = errors.New("user not found")

func repoFindUser(id int64) error {
    return ErrUserNotFound
}

func serviceGetUser(id int64) error {
    if err := repoFindUser(id); err != nil {
        return fmt.Errorf("service get user: %w", err)
    }

    return nil
}

func statusCode(err error) int {
    switch {
    case errors.Is(err, ErrUserNotFound):
        return 404
    default:
        return 500
    }
}

func main() {
    err := serviceGetUser(1)
    if err != nil {
        fmt.Println("status:", statusCode(err))
        fmt.Println("log:", err)
        return
    }

    fmt.Println("status: 200")
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
status: 404
log: service get user: user not found
```

## 三、关键代码结构

`errors.Is(err, ErrUserNotFound)` 可以理解为问一句：“这个错误链里，有没有 `ErrUserNotFound`？”

它适合判断这种稳定错误：

```go
var ErrPermissionDenied = errors.New("permission denied")
var ErrOrderAlreadyPaid = errors.New("order already paid")
var ErrInventoryNotEnough = errors.New("inventory not enough")
```

在 handler 层常见写法是：

```go
func statusCode(err error) int {
    switch {
    case errors.Is(err, ErrUserNotFound):
        return 404
    case errors.Is(err, ErrPermissionDenied):
        return 403
    default:
        return 500
    }
}
```

这个函数的价值不是“少写几行 if”，而是把错误到协议响应的转换集中起来，避免每个接口各写一套不一致的判断。

## 四、真实后端场景示例

下面模拟一个文章删除接口。只有作者本人能删除文章，文章不存在返回 404，不是作者返回 403。

```go
package main

import (
    "errors"
    "fmt"
)

var (
    ErrArticleNotFound  = errors.New("article not found")
    ErrPermissionDenied = errors.New("permission denied")
)

func checkArticleOwner(articleID, userID int64) error {
    if articleID == 404 {
        return ErrArticleNotFound
    }
    if userID != 10 {
        return ErrPermissionDenied
    }

    return nil
}

func deleteArticle(articleID, userID int64) error {
    if err := checkArticleOwner(articleID, userID); err != nil {
        return fmt.Errorf("delete article %d by user %d: %w", articleID, userID, err)
    }

    return nil
}

func statusCode(err error) int {
    switch {
    case errors.Is(err, ErrArticleNotFound):
        return 404
    case errors.Is(err, ErrPermissionDenied):
        return 403
    default:
        return 500
    }
}

func main() {
    err := deleteArticle(99, 20)
    if err != nil {
        fmt.Println("status:", statusCode(err))
        fmt.Println("log:", err)
        return
    }

    fmt.Println("status: 204")
}
```

真实项目中，`statusCode` 可能会放在统一错误响应函数里，而不是散落在每个 handler 中。

## 五、注意点

`errors.Is` 适合回答“是不是某个错误”。如果你还需要拿到错误里的字段，比如哪个参数错了，就应该使用自定义错误类型和 `errors.As`。

另外，HTTP 状态码映射要保守：

- 找不到资源：404
- 没登录：401
- 登录了但没权限：403
- 参数不合法：400
- 资源冲突：409
- 服务器内部失败：500

不要因为想让前端“看得更清楚”，把内部数据库错误原样返回给用户。

## 六、常见误区

误区一：用 `err == target` 判断被包装的错误。

```go
if err == ErrUserNotFound {
    return 404
}
```

错误被 `%w` 包装后，这个判断就不可靠。

误区二：用 `err.Error()` 判断。

```go
if err.Error() == "permission denied" {
    return 403
}
```

错误文案不是稳定接口。它可能被包装，也可能为了日志可读性调整。

误区三：所有错误都返回 500。

这会让前端和调用方无法区分用户操作失败还是系统故障，也会影响监控告警判断。

## 七、本节达标标准

- 能使用 `errors.Is` 判断被包装后的业务错误。
- 能把常见业务错误映射成 HTTP 状态码。
- 能解释为什么不应该用字符串判断错误。
- 能把统一状态码映射逻辑从 handler 中抽出来。
- 能区分业务失败和系统失败。
