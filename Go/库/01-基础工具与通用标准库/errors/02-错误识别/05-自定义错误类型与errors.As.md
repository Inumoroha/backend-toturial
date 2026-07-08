# 05. 自定义错误类型与 errors.As

本节目标：学完后，你能够定义带字段的错误类型，并用 `errors.As` 在上层提取结构化信息。

有些错误只知道“是什么”还不够。比如参数校验失败时，handler 往往需要知道哪个字段错了、错在哪里。这个时候，`ErrInvalidArgument` 这种固定错误变量就不够用了，我们需要自定义错误类型。

## 一、为什么需要它

注册接口可能有多个参数：

- username 不能为空
- password 长度不能太短
- email 格式不正确

如果只返回 `errors.New("invalid argument")`，调用方不知道具体哪个字段错了。真实项目中，前端也需要把错误提示放到对应输入框旁边。

可以把自定义错误理解为“带字段的错误”。

```text
需要携带结构化信息的错误，用自定义 error 类型，再用 errors.As 提取。
```

## 二、基本用法

复制下面代码到 `main.go`。

```go
package main

import (
    "errors"
    "fmt"
)

type ValidationError struct {
    Field string
    Msg   string
}

func (e *ValidationError) Error() string {
    return e.Field + ": " + e.Msg
}

func validateUsername(username string) error {
    if username == "" {
        return &ValidationError{
            Field: "username",
            Msg:   "cannot be empty",
        }
    }

    return nil
}

func main() {
    err := validateUsername("")
    if err != nil {
        var validationErr *ValidationError
        if errors.As(err, &validationErr) {
            fmt.Println("status: 400")
            fmt.Println("field:", validationErr.Field)
            fmt.Println("message:", validationErr.Msg)
            return
        }

        fmt.Println("status: 500")
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
status: 400
field: username
message: cannot be empty
```

## 三、关键代码结构

```go
type ValidationError struct {
    Field string
    Msg   string
}
```

这个错误类型携带了字段名和错误消息。

```go
func (e *ValidationError) Error() string
```

只要实现 `Error() string`，它就是一个 `error`。

```go
var validationErr *ValidationError
errors.As(err, &validationErr)
```

`errors.As` 会沿着错误链查找第一个能赋值给 `*ValidationError` 的错误，并把它放进 `validationErr`。

如果你使用 Go 1.26 或更新版本，也可以使用 `errors.AsType` 简化写法：

```go
validationErr, ok := errors.AsType[*ValidationError](err)
if ok {
    fmt.Println(validationErr.Field)
}
```

如果项目还在 Go 1.25 或更早版本，继续使用 `errors.As`。

## 四、真实后端场景示例

下面模拟一个用户注册接口。service 负责校验参数，handler 负责把校验错误转换成 400。

```go
package main

import (
    "errors"
    "fmt"
)

type ValidationError struct {
    Field string
    Msg   string
}

func (e *ValidationError) Error() string {
    return e.Field + ": " + e.Msg
}

func register(username, password string) error {
    if username == "" {
        return &ValidationError{Field: "username", Msg: "cannot be empty"}
    }
    if len(password) < 8 {
        return &ValidationError{Field: "password", Msg: "must be at least 8 characters"}
    }

    return nil
}

func writeError(err error) {
    var validationErr *ValidationError
    if errors.As(err, &validationErr) {
        fmt.Println("status: 400")
        fmt.Printf("response: field=%s, message=%s\n", validationErr.Field, validationErr.Msg)
        return
    }

    fmt.Println("status: 500")
    fmt.Println("response: internal server error")
}

func main() {
    if err := register("bob", "123"); err != nil {
        writeError(err)
        return
    }

    fmt.Println("status: 201")
}
```

这个例子里，`ValidationError` 没有暴露数据库细节，也没有强行塞进 HTTP 状态码。它只表达一件事：某个输入字段不合法。

## 五、注意点

自定义错误类型适合携带少量、稳定、对上层有意义的信息。不要把整个请求体、完整 SQL、密码、token 放进错误里。

常见字段可以是：

- `Field`
- `Code`
- `Resource`
- `ID`
- `Reason`

如果只是判断“有没有权限”，用 `ErrPermissionDenied` 更简单。如果需要知道“哪个资源、哪个动作没有权限”，可以考虑自定义错误类型。

## 六、常见误区

误区一：所有错误都设计成复杂结构体。

这样会让代码变重。简单稳定的业务错误，用 sentinel error 就够了。

误区二：`errors.As` 的第二个参数写错。

```go
var validationErr ValidationError
errors.As(err, validationErr) // 错误写法
```

应该传入目标变量的地址：

```go
var validationErr *ValidationError
errors.As(err, &validationErr)
```

误区三：把用户可见文案和内部错误混在一起。

错误类型可以包含内部排查信息，但返回给用户时要经过 handler 转换。不要直接把所有 `err.Error()` 返回给前端。

## 七、本节达标标准

- 能定义实现 `Error() string` 的自定义错误类型。
- 能使用 `errors.As` 提取错误中的结构化字段。
- 能解释 `errors.Is` 和 `errors.As` 的适用区别。
- 能为注册、登录、权限等场景设计简单错误类型。
- 能避免把敏感信息塞进错误字段。
