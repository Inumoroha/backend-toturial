# 03. fmt.Errorf 与错误上下文

本节目标：学完后，你能够使用 `fmt.Errorf("%w", err)` 给错误增加上下文，同时保留底层错误供上层识别。

后端线上排查问题时，最怕日志里只有一句 `connection refused` 或 `record not found`。它告诉你失败了，却没告诉你哪个业务动作失败了。`fmt.Errorf("%w", err)` 的价值就在这里：它能把错误“包起来”，在不丢失原始错误的前提下补充上下文。

## 一、为什么需要它

真实项目中，一个数据库错误可能出现在很多地方：

- 查询用户资料
- 创建订单
- 扣减库存
- 生成短链接
- 写操作日志

如果日志只写底层错误，你很难判断是哪条业务路径出问题。

可以把错误包装理解为给错误贴标签：

```text
底层错误说明“发生了什么”，上层上下文说明“做什么时发生的”。
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

func repositoryFindUser(id int64) error {
    return ErrUserNotFound
}

func serviceGetProfile(id int64) error {
    if err := repositoryFindUser(id); err != nil {
        return fmt.Errorf("get user profile: %w", err)
    }

    return nil
}

func main() {
    err := serviceGetProfile(42)
    if err != nil {
        fmt.Println("log:", err)
        fmt.Println("is user not found:", errors.Is(err, ErrUserNotFound))
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

输出类似：

```text
log: get user profile: user not found
is user not found: true
```

## 三、关键参数和代码结构

```go
fmt.Errorf("get user profile: %w", err)
```

`%w` 表示包装一个错误。包装之后，错误文本会变得更清楚，同时原始错误仍然在错误链里。

```go
errors.Is(err, ErrUserNotFound)
```

虽然外层错误文本已经变成 `get user profile: user not found`，但 `errors.Is` 仍然能识别里面的 `ErrUserNotFound`。

不要把 `%w` 写成 `%v`：

```go
fmt.Errorf("get user profile: %v", err)
```

`%v` 只会拼接文本，不会保留错误链。这样上层就无法用 `errors.Is` 判断底层错误了。

## 四、真实后端场景示例

下面模拟一个短链接服务。用户访问短链接时，service 要从 repository 查询目标地址。如果短链接不存在，handler 应该返回 404；如果是数据库失败，才返回 500。

```go
package main

import (
    "errors"
    "fmt"
)

var ErrShortLinkNotFound = errors.New("short link not found")

func findShortLink(code string) (string, error) {
    if code == "missing" {
        return "", ErrShortLinkNotFound
    }

    return "https://example.com/articles/100", nil
}

func resolveShortLink(code string) (string, error) {
    targetURL, err := findShortLink(code)
    if err != nil {
        return "", fmt.Errorf("resolve short link %q: %w", code, err)
    }

    return targetURL, nil
}

func main() {
    targetURL, err := resolveShortLink("missing")
    if err != nil {
        if errors.Is(err, ErrShortLinkNotFound) {
            fmt.Println("status: 404")
            fmt.Println("log:", err)
            return
        }

        fmt.Println("status: 500")
        fmt.Println("log:", err)
        return
    }

    fmt.Println("redirect to:", targetURL)
}
```

这里的 `resolve short link "missing"` 就是上下文。线上日志看到这句话，就知道是解析短链接时失败，而不是随便哪个查询失败。

## 五、注意点

包装错误时，信息要具体，但不要泄露敏感数据。

适合写入上下文：

- 业务动作：`create order`
- 模块位置：`query user`
- 稳定标识：订单 ID、用户 ID、短链接 code

谨慎写入上下文：

- 密码
- token
- 身份证号
- 完整 SQL
- 用户隐私字段

```text
错误日志要帮助内部排查，但不能把敏感信息写进日志或返回给用户。
```

在 handler 层返回给用户的消息，通常应该比日志更简洁。

## 六、常见误区

误区一：只返回底层错误。

```go
return err
```

这样错误到最外层时缺少业务路径，排查成本很高。

误区二：使用 `%v` 导致错误链断掉。

```go
return fmt.Errorf("create order failed: %v", err)
```

上层看起来能打印错误，但不能用 `errors.Is` 或 `errors.As` 识别原始错误。

误区三：每一层都写重复废话。

```go
return fmt.Errorf("failed: %w", err)
```

`failed` 没有提供新信息。上下文应该说明具体动作，例如 `save order`、`query inventory`。

## 七、本节达标标准

- 能使用 `fmt.Errorf("%w", err)` 包装错误。
- 能解释 `%w` 和 `%v` 在错误处理中的区别。
- 能为 service、repository 中的错误补充业务上下文。
- 能避免把敏感信息写入错误文本。
- 能在包装错误后继续用 `errors.Is` 识别底层业务错误。
