# 1. 认识 Go 测试与 testing 包

本节目标：学完后你能为一个 Go 包写出第一个可运行测试，并知道它在后端项目中的作用。

简短引入：后端项目不是写完接口就结束。用户注册、订单金额、权限判断、库存扣减这类逻辑，一旦改错就可能直接影响线上数据。Go 标准库自带 `testing` 包，配合 `go test` 可以让这些规则在每次修改后被自动验证。

## 一、为什么需要它

可以理解为，测试就是给业务规则写一组可重复执行的检查。

比如用户注册时，邮箱不能为空、密码长度不能太短、用户名不能重复。你当然可以每次手动启动服务、打开接口工具、填表单验证，但真实项目会越来越大，手动验证会漏，而且很慢。

Go 的测试有几个适合后端开发的特点：

- 不需要额外测试框架就能开始。
- 测试文件和业务代码放在同一个包附近，容易维护。
- `go test ./...` 可以一次跑完整个项目。
- 测试失败会直接告诉你哪个包、哪个函数、哪条用例失败。

```text
后端测试的第一价值是保护业务规则，而不是追求复杂工具。
```

## 二、基本用法

先创建一个最小项目。

Windows PowerShell：

```powershell
mkdir userdemo
cd userdemo
go mod init example.com/userdemo
```

Linux/macOS：

```bash
mkdir userdemo
cd userdemo
go mod init example.com/userdemo
```

新建 `user.go`：

```go
package userdemo

import "strings"

func NormalizeUsername(name string) string {
	return strings.ToLower(strings.TrimSpace(name))
}
```

新建 `user_test.go`：

```go
package userdemo

import "testing"

func TestNormalizeUsername(t *testing.T) {
	got := NormalizeUsername("  Alice  ")
	want := "alice"

	if got != want {
		t.Fatalf("NormalizeUsername() = %q, want %q", got, want)
	}
}
```

运行测试：

```powershell
go test
```

Linux/macOS 同样运行：

```bash
go test
```

看到 `PASS` 就说明测试通过。

## 三、关键参数/语法/代码结构

`*_test.go` 是 Go 测试文件的固定命名方式。`go test` 会自动识别这些文件，但普通 `go build` 不会把它们编进最终服务程序。

`func TestNormalizeUsername(t *testing.T)` 是测试函数。函数名必须以 `Test` 开头，参数必须是 `*testing.T`。

`got` 和 `want` 是 Go 测试里很常见的写法。`got` 表示实际结果，`want` 表示期望结果。这种写法简单，但很清楚。

`t.Fatalf` 表示测试失败并立即停止当前测试。常见场景是后面的检查依赖前面的结果，例如解析 JSON 失败后就没有必要继续检查字段了。

如果一个测试里有多个相互独立的检查，可以用 `t.Errorf` 记录错误并继续执行。

## 四、真实后端场景示例

现在把例子换成注册参数校验。新建 `register.go`：

```go
package userdemo

import (
	"errors"
	"strings"
)

type RegisterRequest struct {
	Email    string
	Password string
}

func ValidateRegister(req RegisterRequest) error {
	if strings.TrimSpace(req.Email) == "" {
		return errors.New("email is required")
	}
	if len(req.Password) < 8 {
		return errors.New("password is too short")
	}
	return nil
}
```

新建或追加 `register_test.go`：

```go
package userdemo

import "testing"

func TestValidateRegister(t *testing.T) {
	err := ValidateRegister(RegisterRequest{
		Email:    "alice@example.com",
		Password: "12345678",
	})

	if err != nil {
		t.Fatalf("ValidateRegister() unexpected error: %v", err)
	}
}
```

真实项目中，这类测试通常放在 service 或 domain 层，因为这些地方承载业务规则。HTTP handler 只负责把请求转成结构体，真正的规则不要散落在 handler 里。

## 五、注意点

测试应该尽量快。最基础的单元测试不应该启动 HTTP 服务、不应该连真实数据库、不应该调用第三方短信接口。

测试名称要表达业务含义。`TestValidateRegister` 比 `TestA` 好，后面用 `go test -run ValidateRegister` 可以精准运行。

错误信息要写清楚。失败时你应该能直接看出“实际值是什么，期望值是什么”。

真实项目常用命令：

```powershell
go test ./...
```

这会递归测试当前模块下所有包。它通常用于提交代码前或 CI 里。

风险是：如果你把慢集成测试也放进普通测试，`go test ./...` 会越来越慢，团队最后会不愿意跑测试。后面会单独讲如何隔离慢测试。

## 六、常见误区

- 只测试成功场景：后端 bug 很多都出在空值、非法输入、重复提交、权限不足这些失败分支。
- 测试里只写 `fmt.Println`：打印不会让测试失败，必须用 `t.Fatal` 或 `t.Error` 明确判断。
- 测试依赖本机环境：例如必须有某个本地文件、某个数据库连接。这样的测试换一台机器就可能失败。
- 测试函数命名不规范：不是 `TestXxx(t *testing.T)`，`go test` 就不会按普通测试运行。

## 七、本节达标标准

- 能创建 `*_test.go` 文件。
- 能写出 `func TestXxx(t *testing.T)` 测试函数。
- 能使用 `go test` 和 `go test ./...`。
- 能用 `got/want` 写清楚实际值和期望值。
- 能解释为什么单元测试不应该依赖真实外部服务。

