# 07. errors.Join 与批量处理错误

本节目标：学完后，你能够在批量任务中收集多个错误，并用 `errors.Join` 一次性返回给调用方。

单个接口通常只返回一个错误，但后端里有很多批量场景：批量创建用户、批量发布文章、批量刷新缓存、关闭多个资源、执行多项参数校验。这个时候，遇到第一个错误就返回，可能会让调用方反复尝试；把所有错误收集起来，通常更实用。

## 一、为什么需要它

想象一个后台管理接口：批量导入 100 个用户。如果第 3 个用户失败就停止，管理员修完第 3 个再提交，又发现第 8 个失败。体验很差。

更常见的做法是：

```text
尽量处理所有项目，收集失败项，最后统一返回错误。
```

`errors.Join` 就适合这类“多个错误合成一个错误”的场景。

## 二、基本用法

复制下面代码到 `main.go`。

```go
package main

import (
    "errors"
    "fmt"
)

var ErrUsernameEmpty = errors.New("username is empty")

func validateUsername(username string) error {
    if username == "" {
        return ErrUsernameEmpty
    }

    return nil
}

func validateUsers(users []string) error {
    var errs []error

    for i, username := range users {
        if err := validateUsername(username); err != nil {
            errs = append(errs, fmt.Errorf("user[%d]: %w", i, err))
        }
    }

    return errors.Join(errs...)
}

func main() {
    err := validateUsers([]string{"alice", "", "bob", ""})
    if err != nil {
        fmt.Println("validation failed:")
        fmt.Println(err)
        fmt.Println("has empty username:", errors.Is(err, ErrUsernameEmpty))
        return
    }

    fmt.Println("all users valid")
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
validation failed:
user[1]: username is empty
user[3]: username is empty
has empty username: true
```

## 三、关键代码结构

```go
var errs []error
```

用切片收集每一项错误。

```go
errs = append(errs, fmt.Errorf("user[%d]: %w", i, err))
```

为每个错误加上位置上下文。否则你只知道有用户名为空，不知道是哪一行。

```go
return errors.Join(errs...)
```

把多个错误合成一个错误。如果 `errs` 为空，`errors.Join` 会返回 `nil`。

`errors.Is` 和 `errors.As` 仍然可以识别 join 后的错误树。

## 四、真实后端场景示例

下面模拟批量创建短链接。每个短链接都要校验 code 和目标地址。

```go
package main

import (
    "errors"
    "fmt"
    "strings"
)

var (
    ErrCodeEmpty     = errors.New("code is empty")
    ErrTargetInvalid = errors.New("target url is invalid")
)

type ShortLink struct {
    Code   string
    Target string
}

func validateShortLink(link ShortLink) error {
    var errs []error

    if link.Code == "" {
        errs = append(errs, ErrCodeEmpty)
    }
    if !strings.HasPrefix(link.Target, "https://") {
        errs = append(errs, ErrTargetInvalid)
    }

    return errors.Join(errs...)
}

func validateBatch(links []ShortLink) error {
    var errs []error

    for i, link := range links {
        if err := validateShortLink(link); err != nil {
            errs = append(errs, fmt.Errorf("short_link[%d]: %w", i, err))
        }
    }

    return errors.Join(errs...)
}

func main() {
    links := []ShortLink{
        {Code: "go", Target: "https://go.dev"},
        {Code: "", Target: "http://unsafe.example.com"},
    }

    if err := validateBatch(links); err != nil {
        fmt.Println("status: 400")
        fmt.Println("log:", err)
        fmt.Println("has invalid target:", errors.Is(err, ErrTargetInvalid))
        return
    }

    fmt.Println("status: 201")
}
```

真实项目中，批量接口的响应体可能会返回每一行的错误详情。但内部错误处理仍然可以先用 `errors.Join` 汇总，再由 handler 转成响应格式。

## 五、注意点

批量处理要先想清楚策略：

- 是遇到第一个错误就停止？
- 还是尽量处理所有数据，最后返回所有错误？
- 如果部分成功，是否需要回滚？
- 如果不能回滚，响应里是否要告诉调用方哪些成功、哪些失败？

这和业务有关。比如批量校验可以收集全部错误；批量扣库存通常要使用事务，任何一个失败都应该整体回滚。

```text
批量错误处理不仅是技术问题，还要先明确是否允许部分成功。
```

## 六、常见误区

误区一：批量任务里只返回第一个错误。

这在校验场景下会增加调用方修复成本。但在事务场景中，遇到第一个错误就回滚可能是正确选择。

误区二：收集错误但不加索引或业务标识。

```go
errs = append(errs, err)
```

这样调用方不知道哪条数据失败。应该加上 `user[3]`、`sku=xxx`、`code=abc` 这类上下文。

误区三：把 `errors.Join` 当成响应格式。

`errors.Join` 是内部错误表达，不等于 API 响应结构。真正返回给用户时，仍然要转换成稳定 JSON。

## 七、本节达标标准

- 能使用 `errors.Join` 合并多个错误。
- 能在批量处理时为每个错误增加位置或业务标识。
- 能用 `errors.Is` 判断 join 后的错误中是否包含目标错误。
- 能区分校验批量错误和事务批量错误的处理策略。
- 能说明为什么 `errors.Join` 不应该直接作为接口响应格式。
