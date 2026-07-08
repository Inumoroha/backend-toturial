# 07. Replace 与 Replacer 做日志脱敏

本节目标：你能在输出日志前对手机号、邮箱、token 等敏感片段做基础脱敏，减少生产数据泄露风险。

简短引入：日志是排查问题的关键，但日志里不能随便出现用户隐私和凭证。`Replace`、`ReplaceAll`、`NewReplacer` 可以做基础字符串替换，适合处理明确的固定片段。

## 一、为什么需要它

可以理解为：替换函数负责把“不应该原样出现的内容”换成安全文本。

常见场景是：

- 把请求头里的 token 替换成 `[REDACTED]`。
- 把日志里的固定内部地址替换成占位符。
- 把迁移脚本里的环境名替换成目标环境名。

```text
日志默认按会泄露来设计，敏感字段默认不打印。
```

## 二、基本用法

```go
package main

import (
	"fmt"
	"strings"
)

func main() {
	msg := "token=abc123 token=def456"

	fmt.Println(strings.Replace(msg, "abc123", "[REDACTED]", 1))
	fmt.Println(strings.ReplaceAll(msg, "token=", "credential="))
}
```

Windows PowerShell：

```powershell
go run main.go
```

Linux/macOS：

```bash
go run main.go
```

这个命令用于观察替换次数的差异。真实项目中，脱敏通常在日志封装层做，而不是散落在每个业务函数里。

## 三、关键代码结构

`strings.Replace(s, old, new, n)`：

- `old`：要被替换的内容。
- `new`：替换后的内容。
- `n`：替换次数，`-1` 表示全部替换。

`strings.ReplaceAll(s, old, new)` 等价于替换全部。

`strings.NewReplacer` 适合一次替换多组固定内容。

替换时要特别注意 `old` 的含义：

- 如果 `old` 是明确的敏感值，例如当前请求里的 token，可以替换成 `[REDACTED]`。
- 如果 `old` 只是字段名前缀，例如 `token=`，只替换前缀并不会隐藏真实 token。
- 如果敏感内容有复杂格式，例如邮箱、手机号、身份证号，通常要先解析，再脱敏。

```text
脱敏的目标是让敏感值不出现在日志里，不是把字段名改得看起来安全。
```

## 四、真实后端场景示例

下面对已知敏感值做脱敏：

```go
package main

import (
	"fmt"
	"strings"
)

func SafeLogLine(line, password, token string) string {
	if password == "" || token == "" {
		return line
	}
	replacer := strings.NewReplacer(
		password, "[REDACTED]",
		token, "[REDACTED]",
	)
	return replacer.Replace(line)
}

func main() {
	password := "secret"
	token := "abc123"
	line := "login failed user=alice password=secret token=abc123"
	fmt.Println(SafeLogLine(line, password, token))
}
```

这个示例只适合说明固定替换。真实项目中，结构化日志更稳：不要先把敏感字段拼成字符串再脱敏，而是在记录日志时就不要写入敏感字段。

再看一个手机号脱敏的小函数。它不依赖正则，只演示最基本的业务判断：

```go
package main

import (
	"fmt"
	"strings"
)

func MaskPhone(phone string) string {
	phone = strings.TrimSpace(phone)
	if len(phone) != 11 {
		return "[INVALID_PHONE]"
	}
	return phone[:3] + "****" + phone[7:]
}

func main() {
	fmt.Println(MaskPhone(" 13812345678 "))
}
```

真实项目中，手机号可能涉及国家区号、空格、短横线等格式。是否允许这些格式，要先由业务规则决定，再写清洗和校验逻辑。

## 五、注意点

固定字符串替换不是万能脱敏。比如手机号、邮箱、身份证号通常需要更明确的解析规则，可能会用正则或专门的脱敏函数。

日志脱敏还要考虑“谁来负责”。保守建议是：

- handler 层不要打印完整请求体。
- service 层不要把密码、token、验证码放进错误信息。
- 日志封装层统一处理敏感字段。
- 生产日志采集系统设置访问权限和保留周期。

替换迁移 SQL 或配置文件时要特别谨慎：

- 先在测试环境验证。
- 保留原文件。
- 变更要可回滚。
- 不要对生产文件直接做不可逆批量替换。

## 六、常见误区

误区一：先打印完整日志，再考虑脱敏。

日志一旦进入采集系统，可能已经被多个服务保存，后补脱敏通常来不及。

误区二：以为 `ReplaceAll` 能处理所有隐私。

它只处理明确匹配到的文本。格式稍变就可能漏掉。

误区三：在业务逻辑里到处手写脱敏。

真实项目中更推荐统一日志字段规范和日志封装，减少遗漏。

## 七、本节达标标准

- 能用 `Replace` 控制替换次数。
- 能用 `ReplaceAll` 替换全部固定片段。
- 能用 `NewReplacer` 处理多组固定替换。
- 能说明日志脱敏应该前置到日志生成阶段。
- 能判断“替换字段名”和“替换敏感值”的区别。
