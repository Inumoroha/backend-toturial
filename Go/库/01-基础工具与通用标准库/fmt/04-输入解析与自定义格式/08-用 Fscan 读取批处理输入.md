# 08. 用 Fscan 读取批处理输入

本节目标：学会用 `fmt.Fscan` 从 `io.Reader` 连续读取简单记录，并理解它在迁移和修复脚本中的边界。

简短引入：批处理脚本常常要读取一批 ID 或简单记录。`Fscan` 可以从文件、标准输入、字符串 Reader 中按空白读取字段。

## 一、为什么需要它

可以把 `Fscan` 理解为“从一个输入流里不断取下一个值”。它适合每行结构很简单的输入，例如：

```text
1001 active
1002 disabled
1003 active
```

这种形式在数据修复、迁移演练、小型导入工具中很常见。

## 二、基本用法

```go
package main

import (
	"fmt"
	"strings"
)

func main() {
	input := strings.NewReader("1001 active\n1002 disabled\n")

	for {
		var userID int
		var status string

		_, err := fmt.Fscan(input, &userID, &status)
		if err != nil {
			break
		}

		fmt.Printf("user_id=%d status=%s\n", userID, status)
	}
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

`Fscan` 的第一个参数是 `io.Reader`。这意味着输入可以来自文件、网络、标准输入或内存字符串。

循环读取时，遇到 `io.EOF` 表示正常结束，其他错误才是异常。初学阶段可以先用 `break`，真实脚本中要区分 EOF 和坏数据。

## 四、真实后端场景示例

下面模拟读取用户状态修复清单：

```go
package main

import (
	"errors"
	"fmt"
	"io"
	"strings"
)

func main() {
	input := strings.NewReader("1001 active\n1002 disabled\n")

	for {
		var userID int
		var status string

		_, err := fmt.Fscan(input, &userID, &status)
		if errors.Is(err, io.EOF) {
			break
		}
		if err != nil {
			fmt.Println("bad input:", err)
			return
		}

		fmt.Printf("prepare update user_id=%d status=%s\n", userID, status)
	}
}
```

真实项目中，真正执行数据库更新时要使用事务，分批提交，并记录每批成功范围。大批量更新还要评估索引成本和锁表风险。

## 五、注意点

`Fscan` 默认按空白分隔，不保留行结构。如果每行字段复杂，建议使用 `bufio.Scanner` 逐行读取，再解析。

迁移或修复脚本不应直接在生产上裸跑。常见流程是：备份、灰度、限速、事务边界、失败回滚、结果校验。

```text
数据迁移先保证可回滚，再追求执行速度。
```

## 六、常见误区

把 EOF 当错误报警。EOF 通常只是读完了。

不校验状态枚举。输入里出现 `actvie` 这种拼写错误时，脚本应停止或跳过并记录。

一次事务包住所有数据。数据量大时可能锁太久，真实项目通常会分批。

## 七、本节达标标准

- 能用 `Fscan` 从 `io.Reader` 读取简单记录。
- 能区分 EOF 和异常错误。
- 知道复杂行格式应逐行解析。
- 知道迁移脚本要考虑事务、索引成本、生产安全和回滚。

