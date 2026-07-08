# 08. Map 与 ToValidUTF8 处理非法字符

本节目标：你能对用户昵称、文章标题、导入文本中的非法字符做保守处理，避免脏数据进入系统。

简短引入：后端服务可能收到从浏览器、脚本、老系统、CSV 文件传来的文本。多数时候它们是正常 UTF-8，但偶尔会混入控制字符或非法字节。`strings.Map` 和 `ToValidUTF8` 可以帮你做基础清理。

## 一、为什么需要它

可以理解为：这类函数用于把“不适合进入业务系统的字符”处理掉或替换掉。

常见场景是：

- 用户昵称不能包含换行和控制字符。
- 文章标题从外部导入，可能带奇怪字符。
- 日志系统不希望出现破坏展示的非法 UTF-8。

```text
字符清理要保守：宁可明确拒绝，也不要悄悄改出用户不认识的数据。
```

## 二、基本用法

```go
package main

import (
	"fmt"
	"strings"
	"unicode"
)

func main() {
	name := "Alice\nAdmin"
	clean := strings.Map(func(r rune) rune {
		if unicode.IsControl(r) {
			return -1
		}
		return r
	}, name)

	fmt.Println(clean)
	fmt.Println(strings.ToValidUTF8("hello\xffworld", "?"))
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

这个命令用于验证字符过滤效果。真实项目中，字符清理通常和长度限制、唯一性校验一起做。

## 三、关键代码结构

`strings.Map(mapping, s)` 会逐个 rune 处理字符串：

- 返回原 rune：保留。
- 返回新的 rune：替换。
- 返回 `-1`：删除。

`strings.ToValidUTF8(s, replacement)` 会把非法 UTF-8 替换成指定字符串。

初学阶段可以先理解为：`Map` 管字符规则，`ToValidUTF8` 管非法编码。

## 四、真实后端场景示例

下面处理用户昵称：

```go
package main

import (
	"errors"
	"fmt"
	"strings"
	"unicode"
	"unicode/utf8"
)

func NormalizeNickname(raw string) (string, error) {
	name := strings.TrimSpace(strings.ToValidUTF8(raw, ""))
	name = strings.Map(func(r rune) rune {
		if unicode.IsControl(r) {
			return -1
		}
		return r
	}, name)

	if name == "" {
		return "", errors.New("nickname is required")
	}
	if utf8.RuneCountInString(name) > 20 {
		return "", errors.New("nickname too long")
	}
	return name, nil
}

func main() {
	name, err := NormalizeNickname(" Alice\n ")
	if err != nil {
		fmt.Println("昵称错误:", err)
		return
	}
	fmt.Printf("%q\n", name)
}
```

真实项目中，昵称是否允许 emoji、空格、中文、特殊符号，应由产品规则决定。后端负责把规则实现清楚，并在接口错误里给出可理解的原因。

## 五、注意点

Go 的 `len(s)` 统计的是字节数，不是用户看到的字符数量。涉及中文昵称长度时，初学阶段可以先使用 `utf8.RuneCountInString` 进行基础控制。

生产系统里还要考虑数据库字段长度。数据库的 varchar 长度、索引长度和字符集设置都会影响能不能安全写入。

## 六、常见误区

误区一：用 `len` 限制中文昵称长度。

中文在 UTF-8 中通常占多个字节，`len` 可能比你想象的大。

误区二：偷偷替换用户输入但不提示。

如果清理规则会明显改变用户输入，通常应该返回错误，让用户自己修改。

误区三：把字符清理当成安全防护全部。

XSS、SQL 注入、命令注入都有各自的防护方式。字符清理只是输入治理的一部分。

## 七、本节达标标准

- 能用 `strings.Map` 删除控制字符。
- 能用 `ToValidUTF8` 处理非法 UTF-8。
- 能区分字节长度和字符数量。
- 能说明字符清理不能替代参数化查询和输出转义。

