# 07. 开启严格解析，拦住未知字段

本节目标：学完后你能使用 `DisallowUnknownFields` 拦截请求中的未知字段，降低接口误用风险。

简短引入：默认情况下，`encoding/json` 会忽略结构体里没有的字段。这个行为兼容性好，但在后台管理、支付、权限等敏感接口中，过于宽松可能会掩盖客户端错误。

## 一、为什么需要它

假设权限接口只允许提交 `role`，但客户端误传了 `is_admin`。默认解析会忽略 `is_admin`，请求可能仍然成功。调用方会以为设置成功了，实际系统没有处理这个字段。

严格解析可以让后端及时告诉调用方：“你传了一个我不认识的字段。”

```text
越接近权限、资金、库存的接口，请求解析越应该保守。
```

## 二、基本用法

Windows PowerShell：

```powershell
mkdir json-strict
cd json-strict
go mod init json-strict
notepad main.go
go run .
```

Linux/macOS：

```bash
mkdir json-strict
cd json-strict
go mod init json-strict
nano main.go
go run .
```

`main.go`：

```go
package main

import (
	"encoding/json"
	"fmt"
	"strings"
)

type UpdateRoleRequest struct {
	UserID int    `json:"user_id"`
	Role   string `json:"role"`
}

func main() {
	body := `{"user_id":1001,"role":"editor","is_admin":true}`

	var req UpdateRoleRequest
	dec := json.NewDecoder(strings.NewReader(body))
	dec.DisallowUnknownFields()

	if err := dec.Decode(&req); err != nil {
		fmt.Println("decode failed:", err)
		return
	}

	fmt.Printf("%+v\n", req)
}
```

输出会提示未知字段：

```text
decode failed: json: unknown field "is_admin"
```

## 三、关键参数/语法/代码结构

`json.NewDecoder(reader)` 创建解码器。

`dec.DisallowUnknownFields()` 开启严格模式。开启后，JSON 对象里只要出现结构体没有匹配的字段，就会返回错误。

这个设置只影响当前 `Decoder`，不会全局改变 `encoding/json` 的行为。

真实接口里通常还会把严格解析和“只允许一个 JSON 值”放在一起。否则客户端传入 `{"role":"editor"}{"role":"admin"}` 这种内容时，第一次 `Decode` 可能已经成功，后面的垃圾内容被忽略。

```go
func decodeStrictJSON(r io.Reader, dst any) error {
	dec := json.NewDecoder(r)
	dec.DisallowUnknownFields()
	if err := dec.Decode(dst); err != nil {
		return err
	}
	if err := dec.Decode(&struct{}{}); !errors.Is(err, io.EOF) {
		return errors.New("body must contain only one json value")
	}
	return nil
}
```

这段代码在真实项目里的作用是把“请求格式边界”统一收紧。风险是它会让原来多传字段的客户端失败，所以最好先选择后台、内部接口、资金和权限接口使用。

## 四、真实后端场景示例

后台修改用户角色时，可以对请求体开启严格解析。

```go
func decodeStrictJSON(r io.Reader, dst any) error {
	dec := json.NewDecoder(r)
	dec.DisallowUnknownFields()
	return dec.Decode(dst)
}
```

真实项目中，角色值还需要做白名单校验：

```go
func validRole(role string) bool {
	switch role {
	case "viewer", "editor", "admin":
		return true
	default:
		return false
	}
}
```

如果角色变更要写审计日志，建议和角色更新放在同一个事务里，确保权限变化和审计记录一致。

一个更接近项目的处理顺序是：

```text
解析 JSON
拒绝未知字段
校验 role 是否在白名单
校验操作者是否有权限
开启事务
更新用户角色
写审计日志
提交事务
```

这里要注意，严格解析只解决“字段是不是我认识的”这个问题。操作者有没有权限、目标用户是否存在、角色是否允许被授予，仍然要在业务层判断。

## 五、注意点

严格解析适合内部后台、管理接口、资金接口、库存接口。对外开放 API 要谨慎，因为客户端多传字段时直接失败，可能影响兼容性。

如果你正在做接口版本迁移，可以先在日志里统计未知字段，再逐步开启严格解析。

注意 `encoding/json` 对结构体字段匹配默认是大小写不敏感的。不要把大小写差异当成安全边界。

上线严格解析时，建议按接口分批推进。先在测试环境开启，再对少量内部接口开启，观察错误日志。如果发现大量客户端误传字段，要先推动调用方修正，再扩大范围。

如果接口已经对外开放，并且历史上允许客户端多传字段，可以考虑新版本接口开启严格解析，旧版本接口保持兼容。这样比直接改旧接口更稳。

## 六、常见误区

误区一：以为默认解析会拒绝未知字段。标准库默认会忽略未知字段。

误区二：所有接口都立刻开启严格解析。老客户端可能已经多传字段，直接开启会造成兼容问题。

误区三：开启严格解析后就不做业务校验。未知字段检查只解决字段集合问题，不解决角色是否合法。

误区四：权限变更只更新用户表，不写审计日志。生产系统中权限变更通常需要可追溯。

## 七、本节达标标准

- 能使用 `Decoder.DisallowUnknownFields` 拒绝未知字段。
- 能解释默认忽略未知字段的风险。
- 能判断哪些接口更适合严格解析。
- 能说明严格解析不能替代业务校验。
- 能理解权限变更要考虑审计和事务一致性。
