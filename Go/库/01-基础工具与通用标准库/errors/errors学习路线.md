# Go 标准库 errors 系统学习路线

> 目标：面向 Go 后端工程师，系统掌握 `errors` 标准库，并能在 HTTP API、数据库访问、服务分层、日志排查和业务错误建模中正确使用错误处理。

官方文档：

- [`errors` package](https://pkg.go.dev/errors)
- [`fmt.Errorf`](https://pkg.go.dev/fmt#Errorf)

## 1. 先理解 Go 的错误处理哲学

Go 的错误处理不是异常机制，而是显式返回值。

```go
result, err := doSomething()
if err != nil {
    return err
}
```

你需要先建立几个基本认识：

- `error` 是一个普通接口，不是特殊语法。
- 错误是普通值，可以被创建、返回、比较、包装和传递。
- Go 鼓励在出错位置附近显式处理错误。
- 后端服务中，错误处理的质量会直接影响日志排查、接口状态码、可观测性和系统稳定性。

`error` 接口定义如下：

```go
type error interface {
    Error() string
}
```

这意味着任何类型只要实现了 `Error() string` 方法，就实现了 `error` 接口。

学习目标：

- 能解释为什么 Go 中大量函数返回 `(..., error)`。
- 能写出最基本的 `if err != nil` 错误处理。
- 能理解错误本质上是值，而不是异常控制流。

练习：

```go
func ReadConfig(path string) (string, error) {
    if path == "" {
        return "", errors.New("path is empty")
    }

    return "config-content", nil
}
```

调用方：

```go
content, err := ReadConfig("")
if err != nil {
    fmt.Println("read config failed:", err)
    return
}

fmt.Println(content)
```

## 2. 掌握 errors.New

`errors.New` 用于创建一个只有错误信息的基础错误。

```go
err := errors.New("user not found")
```

更常见的后端写法是定义包级错误变量：

```go
var ErrUserNotFound = errors.New("user not found")
var ErrPermissionDenied = errors.New("permission denied")
```

这种错误通常叫做 sentinel error，也就是“哨兵错误”。它表示某个稳定、可识别的错误状态。

适合使用 sentinel error 的场景：

- 用户不存在
- 资源不存在
- 权限不足
- token 无效
- 业务状态不允许操作

示例：

```go
package user

import "errors"

var ErrUserNotFound = errors.New("user not found")

func FindUser(id int64) error {
    if id == 0 {
        return ErrUserNotFound
    }

    return nil
}
```

注意：

- 不要在每次返回时都 `errors.New("user not found")`，否则调用方无法稳定识别这个错误。
- 如果错误需要被上层判断，应定义成包级变量。
- 如果错误只用于临时返回，不需要上层识别，可以直接使用 `errors.New`。

练习：

- 定义 `ErrUserNotFound`。
- 写一个 `FindUser(id int64) error`。
- 当 `id == 0` 时返回 `ErrUserNotFound`。

## 3. 掌握错误包装 fmt.Errorf("%w", err)

后端开发中，不应该只把底层错误原样返回。你通常需要为错误增加上下文。

```go
if err != nil {
    return fmt.Errorf("query user by id %d: %w", id, err)
}
```

这里的 `%w` 表示包装一个错误。被包装的底层错误仍然保留在错误链中。

对比：

```go
fmt.Errorf("query user: %w", err) // 保留错误链
fmt.Errorf("query user: %v", err) // 只拼接错误文本，不保留错误链
```

推荐写法：

```go
return fmt.Errorf("create order: save order: %w", err)
```

不推荐写法：

```go
return errors.New("failed")
```

原因是 `"failed"` 太模糊，线上日志里很难定位问题。

一个典型的错误链可能长这样：

```text
handle create order: create order: save order: insert into database: connection refused
```

每一层都提供了一点上下文，排查问题时就能看出错误经过了哪些业务路径。

练习：

- 写三层函数：`Handler -> Service -> Repository`。
- `Repository` 返回原始错误。
- `Service` 使用 `%w` 包装错误。
- `Handler` 打印最终错误。

示例：

```go
func repositoryFindUser(id int64) error {
    return sql.ErrNoRows
}

func serviceGetUser(id int64) error {
    if err := repositoryFindUser(id); err != nil {
        return fmt.Errorf("get user from repository: %w", err)
    }

    return nil
}

func handlerGetUser(id int64) {
    if err := serviceGetUser(id); err != nil {
        fmt.Println("handle get user:", err)
    }
}
```

## 4. 掌握 errors.Is

`errors.Is` 用于判断一个错误或错误链中是否包含某个目标错误。

```go
if errors.Is(err, ErrUserNotFound) {
    // 返回 404
}
```

为什么不要直接比较？

```go
if err == ErrUserNotFound {
    // 只能判断未包装的错误
}
```

一旦错误被 `%w` 包装，直接比较就会失败：

```go
err := fmt.Errorf("get user: %w", ErrUserNotFound)

fmt.Println(err == ErrUserNotFound)              // false
fmt.Println(errors.Is(err, ErrUserNotFound))     // true
```

后端常见用法：

```go
switch {
case errors.Is(err, ErrUserNotFound):
    return http.StatusNotFound
case errors.Is(err, ErrPermissionDenied):
    return http.StatusForbidden
default:
    return http.StatusInternalServerError
}
```

学习目标：

- 能区分 `err == target` 和 `errors.Is(err, target)`。
- 能在错误被多层包装后仍然识别底层错误。
- 能把业务错误映射到 HTTP 状态码。

练习：

- 定义 `ErrUserNotFound`。
- 在 repository 层返回它。
- 在 service 层包装它。
- 在 handler 层用 `errors.Is` 识别它。

## 5. 掌握自定义错误类型与 errors.As

当错误需要携带结构化信息时，不要只用字符串，应该定义自定义错误类型。

示例：参数校验错误。

```go
type ValidationError struct {
    Field string
    Msg   string
}

func (e *ValidationError) Error() string {
    return e.Field + ": " + e.Msg
}
```

返回错误：

```go
func ValidateUsername(username string) error {
    if username == "" {
        return &ValidationError{
            Field: "username",
            Msg:   "cannot be empty",
        }
    }

    return nil
}
```

使用 `errors.As` 提取错误：

```go
var validationErr *ValidationError
if errors.As(err, &validationErr) {
    fmt.Println("invalid field:", validationErr.Field)
}
```

`errors.Is` 和 `errors.As` 的区别：

| 方法 | 用途 | 适合场景 |
| --- | --- | --- |
| `errors.Is` | 判断是否是某个目标错误 | `ErrUserNotFound`、`ErrInvalidToken` |
| `errors.As` | 判断并提取某种错误类型 | `*ValidationError`、`*DomainError` |

后端常见错误类型：

```go
type AppError struct {
    Code    string
    Message string
}

func (e *AppError) Error() string {
    return e.Code + ": " + e.Message
}
```

示例：

```go
var appErr *AppError
if errors.As(err, &appErr) {
    switch appErr.Code {
    case "INVALID_ARGUMENT":
        return http.StatusBadRequest
    case "CONFLICT":
        return http.StatusConflict
    }
}
```

Go 1.26 开始，标准库还提供了泛型辅助函数 `errors.AsType[E]`，可以减少 `errors.As` 的样板代码：

```go
validationErr, ok := errors.AsType[*ValidationError](err)
if ok {
    fmt.Println(validationErr.Field)
}
```

如果你的项目还在 Go 1.25 或更早版本，继续使用 `errors.As`。

练习：

- 定义 `ValidationError`。
- 校验注册参数。
- handler 层提取字段名，并返回 400。

## 6. 理解 errors.Unwrap

`errors.Unwrap` 用于拆开一层包装错误。

```go
baseErr := errors.New("database unavailable")
wrappedErr := fmt.Errorf("query user: %w", baseErr)

fmt.Println(errors.Unwrap(wrappedErr)) // database unavailable
```

注意：

- `errors.Unwrap` 只拆一层。
- 实际业务中，更多使用 `errors.Is` 和 `errors.As`。
- 学习 `Unwrap` 的主要目的，是理解错误链的工作原理。

多层包装示例：

```go
err1 := errors.New("connection refused")
err2 := fmt.Errorf("query database: %w", err1)
err3 := fmt.Errorf("get user: %w", err2)

fmt.Println(err3)
fmt.Println(errors.Unwrap(err3))
fmt.Println(errors.Unwrap(errors.Unwrap(err3)))
```

输出类似：

```text
get user: query database: connection refused
query database: connection refused
connection refused
```

练习：

- 连续包装三层错误。
- 使用 `errors.Unwrap` 一层层打印。
- 再使用 `errors.Is` 一次性判断最底层错误。

## 7. 学习 errors.Join

`errors.Join` 用于把多个错误合成一个错误。

```go
err := errors.Join(err1, err2, err3)
```

特点：

- 会忽略 `nil` 错误。
- 如果所有错误都是 `nil`，返回 `nil`。
- 拼接后的错误可以继续被 `errors.Is` 和 `errors.As` 检查。
- 适合批量任务、多个资源关闭、多项校验等场景。

示例：批量校验。

```go
func ValidateRegister(username, email string) error {
    var errs []error

    if username == "" {
        errs = append(errs, &ValidationError{
            Field: "username",
            Msg:   "cannot be empty",
        })
    }

    if email == "" {
        errs = append(errs, &ValidationError{
            Field: "email",
            Msg:   "cannot be empty",
        })
    }

    return errors.Join(errs...)
}
```

示例：关闭多个资源。

```go
func CloseAll(resources ...io.Closer) error {
    var errs []error

    for _, resource := range resources {
        if err := resource.Close(); err != nil {
            errs = append(errs, err)
        }
    }

    return errors.Join(errs...)
}
```

练习：

- 批量创建多个用户。
- 每个用户可能返回一个错误。
- 最后使用 `errors.Join` 返回所有错误。

## 8. 后端项目中的错误分层

在真实后端服务里，错误通常会经过这些层：

```text
handler/controller -> service/usecase -> repository/dao -> database/cache/rpc
```

推荐职责划分：

| 层级 | 错误处理职责 |
| --- | --- |
| repository/dao | 保留底层错误，补充数据库、缓存、RPC 等上下文 |
| service/usecase | 转换成业务错误，补充业务动作上下文 |
| handler/controller | 映射成 HTTP 状态码和用户可见响应 |
| middleware | 统一日志、trace id、panic recovery |

repository 层：

```go
func (r *UserRepository) FindByID(ctx context.Context, id int64) (*User, error) {
    user, err := r.queryUser(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("query user by id %d: %w", id, err)
    }

    return user, nil
}
```

service 层：

```go
func (s *UserService) GetProfile(ctx context.Context, id int64) (*UserProfile, error) {
    user, err := s.repo.FindByID(ctx, id)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, ErrUserNotFound
        }

        return nil, fmt.Errorf("get user profile: %w", err)
    }

    return buildProfile(user), nil
}
```

handler 层：

```go
func writeError(w http.ResponseWriter, err error) {
    switch {
    case errors.Is(err, ErrUserNotFound):
        http.Error(w, "user not found", http.StatusNotFound)
    case errors.Is(err, ErrPermissionDenied):
        http.Error(w, "permission denied", http.StatusForbidden)
    default:
        http.Error(w, "internal server error", http.StatusInternalServerError)
    }
}
```

关键原则：

- 内部错误要进日志，不要直接暴露给用户。
- 用户响应要稳定、简洁、安全。
- 错误链中要保留足够的排查上下文。
- 业务错误应该在 service 层明确建模。
- handler 层只负责协议转换，不要写复杂业务判断。

## 9. 常见业务错误设计方式

### 方式一：sentinel error

适合简单、稳定、只需要判断类型的错误。

```go
var ErrOrderNotFound = errors.New("order not found")
var ErrOrderAlreadyPaid = errors.New("order already paid")
```

优点：

- 简单直接。
- 适合 `errors.Is`。
- 容易映射状态码。

缺点：

- 不能携带字段信息。

### 方式二：自定义错误类型

适合需要携带结构化信息的错误。

```go
type ConflictError struct {
    Resource string
    ID       string
}

func (e *ConflictError) Error() string {
    return e.Resource + " conflict: " + e.ID
}
```

优点：

- 可以携带字段。
- 适合 `errors.As`。
- 更适合复杂业务。

缺点：

- 需要设计类型和字段。
- 不要滥用，否则错误体系会过度复杂。

### 方式三：统一 AppError

适合中大型服务。

```go
type AppError struct {
    Code    string
    Message string
    Cause   error
}

func (e *AppError) Error() string {
    if e.Cause == nil {
        return e.Code + ": " + e.Message
    }

    return e.Code + ": " + e.Message + ": " + e.Cause.Error()
}

func (e *AppError) Unwrap() error {
    return e.Cause
}
```

使用：

```go
return &AppError{
    Code:    "USER_NOT_FOUND",
    Message: "user not found",
    Cause:   err,
}
```

适合场景：

- 需要统一业务错误码。
- 需要对接前端或客户端。
- 需要统一错误响应格式。

## 10. 常见坑

### 坑一：使用 %v 包装错误

错误示例：

```go
return fmt.Errorf("get user: %v", err)
```

正确示例：

```go
return fmt.Errorf("get user: %w", err)
```

`%v` 只会拼接文本，`errors.Is` 和 `errors.As` 无法识别底层错误。

### 坑二：重复创建相同文本的错误

错误示例：

```go
return errors.New("user not found")
```

然后调用方：

```go
if errors.Is(err, errors.New("user not found")) {
    // 永远不应该这样写
}
```

正确示例：

```go
var ErrUserNotFound = errors.New("user not found")
```

### 坑三：错误信息没有上下文

错误示例：

```go
return err
```

如果这个错误最终出现在日志里，你可能只看到：

```text
connection refused
```

正确示例：

```go
return fmt.Errorf("create order: insert order record: %w", err)
```

日志会更有价值：

```text
create order: insert order record: connection refused
```

### 坑四：把内部错误直接返回给用户

错误示例：

```go
http.Error(w, err.Error(), http.StatusInternalServerError)
```

这可能把 SQL、表名、内部路径、RPC 地址等信息暴露出去。

推荐做法：

```go
log.Printf("request failed: %v", err)
http.Error(w, "internal server error", http.StatusInternalServerError)
```

### 坑五：错误类型设计过早复杂化

小项目里不要一开始就设计庞大的错误码体系。

推荐演进顺序：

```text
errors.New -> sentinel error -> custom error type -> AppError/error code
```

## 11. 七天学习计划

### 第 1 天：错误基础

学习内容：

- `error` 接口
- `errors.New`
- 基本 `if err != nil`
- 错误作为普通值

产出：

- 写 5 个返回 `error` 的函数。
- 能解释 `nil` error 表示什么。

### 第 2 天：sentinel error

学习内容：

- 包级错误变量
- `ErrUserNotFound` 这类业务错误
- 错误识别的基本需求

产出：

- 写一个用户查询示例。
- 用户不存在时返回固定错误变量。

### 第 3 天：错误包装

学习内容：

- `fmt.Errorf("%w", err)`
- 错误链
- 上下文信息设计

产出：

- 写 `handler -> service -> repository` 三层。
- 每层都给错误增加合适上下文。

### 第 4 天：errors.Is

学习内容：

- 判断包装后的错误
- 业务错误到 HTTP 状态码映射

产出：

- 实现 `writeError` 函数。
- 根据不同错误返回 400、403、404、500。

### 第 5 天：errors.As

学习内容：

- 自定义错误类型
- 结构化错误信息
- 参数校验错误

产出：

- 实现 `ValidationError`。
- 在 handler 层提取字段信息。

### 第 6 天：errors.Unwrap 和 errors.Join

学习内容：

- 手动拆错误链
- 多错误聚合
- 批量任务错误处理

产出：

- 写一个批量注册用户函数。
- 收集所有失败项并返回 `errors.Join`。

### 第 7 天：综合小项目

做一个迷你用户模块：

功能：

- 用户注册
- 用户查询
- 用户登录
- 参数校验
- 用户不存在
- 用户名重复
- 密码错误
- 模拟数据库错误

要求：

- repository 层保留底层错误。
- service 层转换业务错误。
- handler 层统一返回 HTTP 状态码。
- 日志中保留完整错误链。
- 用户响应不暴露内部错误。

## 12. 综合练习：用户注册错误处理

错误定义：

```go
var (
    ErrUserNotFound      = errors.New("user not found")
    ErrUsernameExists    = errors.New("username already exists")
    ErrInvalidCredential = errors.New("invalid credential")
)
```

校验错误：

```go
type ValidationError struct {
    Field string
    Msg   string
}

func (e *ValidationError) Error() string {
    return e.Field + ": " + e.Msg
}
```

service 示例：

```go
func Register(username, password string) error {
    if username == "" {
        return &ValidationError{
            Field: "username",
            Msg:   "cannot be empty",
        }
    }

    if password == "" {
        return &ValidationError{
            Field: "password",
            Msg:   "cannot be empty",
        }
    }

    if err := saveUser(username, password); err != nil {
        if errors.Is(err, ErrUsernameExists) {
            return ErrUsernameExists
        }

        return fmt.Errorf("register user: %w", err)
    }

    return nil
}
```

handler 错误映射：

```go
func statusCodeFromError(err error) int {
    var validationErr *ValidationError

    switch {
    case errors.As(err, &validationErr):
        return http.StatusBadRequest
    case errors.Is(err, ErrUserNotFound):
        return http.StatusNotFound
    case errors.Is(err, ErrUsernameExists):
        return http.StatusConflict
    case errors.Is(err, ErrInvalidCredential):
        return http.StatusUnauthorized
    default:
        return http.StatusInternalServerError
    }
}
```

## 13. 学完后的能力标准

完成这条路线后，你应该能做到：

- 知道 `error` 接口的本质。
- 能正确使用 `errors.New` 创建基础错误。
- 能设计可识别的 sentinel error。
- 能使用 `%w` 包装错误并保留错误链。
- 能用 `errors.Is` 判断底层错误。
- 能用 `errors.As` 提取自定义错误类型。
- 能理解 `errors.Unwrap` 的底层机制。
- 能用 `errors.Join` 聚合多个错误。
- 能在后端分层架构中合理传播错误。
- 能把内部错误映射成安全的 HTTP 响应。
- 能写出方便线上排查的错误上下文。

## 14. 推荐掌握顺序

最短路径：

```text
error 接口
  -> errors.New
  -> fmt.Errorf("%w")
  -> errors.Is
  -> 自定义 error
  -> errors.As
  -> errors.Unwrap
  -> errors.Join
  -> 后端错误分层
```

学习时不要只背 API，要反复问自己三个问题：

- 这个错误是否需要被上层识别？
- 这个错误是否需要携带结构化信息？
- 这个错误最终应该如何变成用户响应和日志？

只要这三个问题能回答清楚，你的 Go 错误处理就已经进入工程化阶段了。
