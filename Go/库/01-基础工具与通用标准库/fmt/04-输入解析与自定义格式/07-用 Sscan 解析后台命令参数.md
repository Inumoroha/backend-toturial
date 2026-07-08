# 07. 用 Sscan 解析后台命令参数

本节目标：学会用 `fmt.Sscan` 从简单字符串里解析字段，适合后台小工具和学习场景。

简短引入：有些后台脚本会接收简单命令，比如 `freeze 1001 30` 表示冻结用户 1001 三十天。`Sscan` 可以把这类空格分隔的文本解析成变量。

## 一、为什么需要它

`Sscan` 可以理解为“从字符串里按空白拆出值，并转换到变量”。它适合格式很简单、字段数量固定的输入。

真实项目中常见场景：

- 内部命令演示。
- 本地批处理脚本。
- 学习输入解析。
- 简单迁移参数验证。

## 二、基本用法

```go
package main

import (
	"fmt"
)

func main() {
	line := "freeze 1001 30"

	var action string
	var userID int
	var days int

	n, err := fmt.Sscan(line, &action, &userID, &days)
	fmt.Println("n:", n, "err:", err)
	fmt.Println(action, userID, days)
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

`Sscan` 的参数必须传指针，因为它要把解析结果写回变量。

返回值 `n` 表示成功解析了几个字段。真实脚本里要检查 `n`，不能只看 `err`。

`Sscan` 默认按空白分隔，所以它不适合解析包含空格的标题、地址、备注。

## 四、真实后端场景示例

下面模拟解析一个封禁用户命令：

```go
package main

import (
	"fmt"
)

func main() {
	input := "ban 1001 spam"

	var action string
	var userID int
	var reason string

	n, err := fmt.Sscan(input, &action, &userID, &reason)
	if err != nil || n != 3 {
		fmt.Println("invalid command")
		return
	}

	fmt.Printf("action=%s user_id=%d reason=%s\n", action, userID, reason)
}
```

真实项目中，执行封禁前还要检查操作人权限，写审计日志，并确保操作可回滚。例如可以保留原状态，支持解封。

## 五、注意点

`Sscan` 不适合解析复杂命令。复杂 CLI 参数应使用 `flag` 包或更成熟的命令行库。

如果输入来自用户，不要解析后直接执行高风险操作。至少要做权限校验、参数校验、审计记录。

```text
后台命令越接近生产数据，越要保守：校验、确认、审计、可回滚。
```

一个更保守的命令处理流程通常是：

1. 先解析命令。
2. 校验动作是否在白名单内。
3. 校验 ID、数量、状态枚举是否合法。
4. 输出将要执行的计划。
5. dry-run 通过后再真正执行。

`Sscan` 只覆盖第一步。后面的校验和执行边界仍然要自己设计。比如 `ban 1001 spam` 解析成功，不代表当前操作人有封禁权限，也不代表这个用户一定存在。

## 六、常见误区

忘记传指针。`Sscan(input, action)` 无法把结果写入变量。

不检查解析字段数量。输入缺字段时，后续可能拿到零值并误操作。

用 `Sscan` 解析带空格的文章标题。它会在第一个空格处截断。

## 七、本节达标标准

- 能用 `Sscan` 解析简单空格分隔字符串。
- 能检查 `n` 和 `err`。
- 知道复杂 CLI 应考虑 `flag` 或专门库。
- 知道后台命令执行前要做权限、审计和回滚设计。
