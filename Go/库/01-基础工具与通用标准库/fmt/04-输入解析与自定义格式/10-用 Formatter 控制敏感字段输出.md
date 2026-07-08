# 10. 用 Formatter 控制敏感字段输出

本节目标：了解 `fmt.Formatter` 的适用场景，能为敏感对象设计更保守的输出方式。

简短引入：`Stringer` 只能返回一种默认文本。`Formatter` 更进一步，可以根据 `%v`、`%#v` 等格式化动词控制输出。初学阶段不需要频繁使用，但要知道它能解决什么问题。

## 一、为什么需要它

真实后端中，有些对象需要更严格的输出控制。例如 Token、手机号、内部配置。你可能希望普通输出脱敏，而调试输出也不要泄露完整值。

```text
敏感信息默认不输出，确实需要查看时也要有权限、审计和最小化范围。
```

## 二、基本用法

```go
package main

import "fmt"

type SecretToken string

func (t SecretToken) Format(s fmt.State, verb rune) {
	fmt.Fprint(s, "token(***redacted***)")
}

func main() {
	token := SecretToken("real-secret-token")
	fmt.Printf("%v\n", token)
	fmt.Printf("%#v\n", token)
}
```

Windows PowerShell：

```powershell
go run .\main.go
```

Linux/macOS：

```bash
go run ./main.go
```

## 三、关键参数/语法/代码结构

实现下面方法就是 `fmt.Formatter`：

```go
Format(state fmt.State, verb rune)
```

`state` 是输出目标，`verb` 是格式化动词。你可以根据 `verb` 决定怎么输出。

初学阶段建议只做简单、安全的输出，不要过度设计。

## 四、真实后端场景示例

下面让手机号只显示后四位：

```go
package main

import "fmt"

type Phone string

func (p Phone) Format(s fmt.State, verb rune) {
	raw := string(p)
	if len(raw) < 4 {
		fmt.Fprint(s, "***")
		return
	}
	fmt.Fprintf(s, "***%s", raw[len(raw)-4:])
}

func main() {
	phone := Phone("13800138000")
	fmt.Printf("bind phone: %v\n", phone)
}
```

真实项目中，手机号脱敏通常会放在统一工具或类型中。这样日志、审计、错误消息都能保持一致。

## 五、注意点

`Formatter` 是高级一点的接口，不要为了普通输出滥用。多数时候 `Stringer` 或显式脱敏函数已经足够。

`Format` 方法里写输出时，通常使用 `fmt.Fprint` 或 `fmt.Fprintf` 写到 `state`，不要写到标准输出。

如果确实要根据动词区分输出，建议仍然保持保守。例如无论 `%v` 还是 `%#v` 都不输出完整 Token，只在 `%T` 这类类型查看场景交给 `fmt` 默认机制会更合适。初学阶段可以采用一个简单原则：

```text
敏感类型的所有格式化输出都默认脱敏。
```

也可以把敏感类型的格式化测试写出来，防止以后改动时误泄露：

```go
package main

import "fmt"

type APIKey string

func (k APIKey) Format(s fmt.State, verb rune) {
	fmt.Fprint(s, "api_key(***redacted***)")
}

func main() {
	key := APIKey("real-api-key")
	fmt.Printf("%v\n", key)
	fmt.Printf("%#v\n", key)
	fmt.Printf("%s\n", key)
}
```

真实项目中，这类测试可以放进单元测试里，明确断言输出不包含原始密钥。

## 六、常见误区

在 `Format` 中忽略安全边界，`%#v` 输出完整密钥。调试格式也可能进入日志。

把 `Formatter` 写得过于复杂。输出方法越复杂，越容易出现递归、性能和不可预期行为。

认为脱敏就是安全。脱敏只是降低泄露风险，权限控制、日志访问控制和审计仍然重要。

## 七、本节达标标准

- 知道 `Formatter` 可以控制不同格式化动词下的输出。
- 能写一个简单脱敏类型。
- 知道 `Format` 应写入 `fmt.State`。
- 知道敏感信息输出要默认保守。
