# 01. error 接口与显式返回

本节目标：学完后，你能够写出最基本的 Go 错误返回代码，并理解它在后端接口中的作用。

在后端项目里，错误不是“程序崩了”的同义词。更多时候，错误表示一次业务请求没有按预期完成，比如用户参数为空、文章不存在、库存不足、数据库连接失败。Go 把这些情况当成普通返回值处理，这让错误处理变得明确，也让每一层代码都能决定自己该做什么。

## 一、为什么需要它

可以把 `error` 理解为函数给调用方的一张说明纸：这次操作失败了，原因在这里。真实项目中常见的调用链是：

```text
HTTP handler -> service -> repository -> database
```

如果 repository 查询数据库失败，它不能直接决定给用户返回什么 HTTP 状态码，因为它不知道自己被哪个接口调用。它只需要把错误返回给 service。service 再判断这是业务错误还是系统错误。handler 最后把它转换成 HTTP 响应。

这种显式返回的好处是，每一层都能看见错误，也能补充自己的上下文。

```text
错误应该沿着调用链向上返回，并在合适的层被处理。
```

## 二、基本用法

新建一个 `main.go`，复制下面代码。

```go
package main

import (
    "errors"
    "fmt"
)

func getUserName(userID int64) (string, error) {
    if userID <= 0 {
        return "", errors.New("invalid user id")
    }

    if userID == 404 {
        return "", errors.New("user not found")
    }

    return "alice", nil
}

func main() {
    name, err := getUserName(404)
    if err != nil {
        fmt.Println("get user name failed:", err)
        return
    }

    fmt.Println("user name:", name)
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
get user name failed: user not found
```

## 三、关键代码结构

`func getUserName(userID int64) (string, error)` 表示函数会返回两个值。第一个是正常结果，第二个是错误。

```go
return "", errors.New("invalid user id")
```

当函数失败时，正常结果返回零值，错误返回具体原因。

```go
return "alice", nil
```

当函数成功时，错误返回 `nil`。在 Go 里，`nil` error 表示没有错误。

```go
if err != nil {
    return
}
```

调用方必须检查错误。真实项目中，如果忽略错误，后面的代码很可能拿着空数据继续执行，最终出现更难定位的问题。

## 四、真实后端场景示例

下面模拟一个用户资料接口。handler 不直接查数据库，而是调用 service。

```go
package main

import (
    "errors"
    "fmt"
)

type User struct {
    ID   int64
    Name string
}

func findUserFromDB(id int64) (*User, error) {
    if id == 0 {
        return nil, errors.New("user id is required")
    }
    if id == 404 {
        return nil, errors.New("user not found")
    }

    return &User{ID: id, Name: "alice"}, nil
}

func getUserProfile(id int64) (*User, error) {
    user, err := findUserFromDB(id)
    if err != nil {
        return nil, err
    }

    return user, nil
}

func main() {
    user, err := getUserProfile(404)
    if err != nil {
        fmt.Println("response: failed to get user profile")
        fmt.Println("log:", err)
        return
    }

    fmt.Println(user.Name)
}
```

这里先不要急着追求完美。你只需要看到：错误从数据库访问层返回到 service，再返回到最外层。后面的章节会继续改进它，让错误更容易被识别，也更适合返回 HTTP 状态码。

## 五、注意点

`error` 是接口，真实项目中你通常不会手写这个接口，而是使用标准库或自定义类型实现它。

初学阶段可以先记住：

- 成功时返回 `nil` error。
- 失败时返回非 `nil` error。
- 调用函数后，先判断 `err != nil`。
- 不要为了省几行代码忽略错误。

如果一个函数内部能自己处理错误，比如重试一次、使用默认值、记录日志后继续执行，那么可以不向上返回。但如果它无法判断业务应该怎么处理，就应该把错误返回给调用方。

## 六、常见误区

误区一：只看正常返回值，不检查错误。

```go
user, _ := getUserProfile(404)
fmt.Println(user.Name)
```

这样可能直接触发空指针问题。后端项目中，忽略错误会让真实原因被掩盖。

误区二：错误信息太随意。

```go
return "", errors.New("bad")
```

这类错误在日志里几乎没有价值。至少要说明哪个输入或哪个动作失败了。

误区三：在底层直接决定 HTTP 响应。

repository 层不应该直接返回 404 或 500。HTTP 是接口层的协议细节，底层应该返回错误本身。

## 七、本节达标标准

- 能写出返回 `(..., error)` 的函数。
- 能用 `errors.New` 创建一个基础错误。
- 能在调用方正确判断 `err != nil`。
- 能说明为什么后端项目中错误要沿调用链返回。
- 能避免忽略错误导致的空指针或错误数据继续传播。
