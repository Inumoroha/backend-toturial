# 03. 用 bytes.Reader 模拟请求体和文件

本节目标：学完后你能用 `bytes.Reader` 把内存数据伪装成可读取的数据源，编写更稳定的后端测试。

## 简短引入

很多 Go 函数不关心数据来自文件、网络还是内存，只要求传入一个 `io.Reader`。`bytes.Reader` 的作用就是把 `[]byte` 包装成一个可读取对象。

这在后端测试里非常实用：你不用真的创建文件，也不用启动 HTTP 服务，就能测试读取逻辑。

## 一、为什么需要它

真实项目中常见函数会这样设计：

```go
func ImportUsers(r io.Reader) error
```

这个函数可以读取上传文件，也可以读取本地文件，还可以读取测试数据。只要数据源实现了 `io.Reader`，函数就能工作。

`bytes.NewReader` 让你在测试时轻松构造输入。

## 二、基本用法

```go
package main

import (
	"bytes"
	"fmt"
	"io"
)

func readAll(r io.Reader) (string, error) {
	data, err := io.ReadAll(r)
	if err != nil {
		return "", err
	}
	return string(data), nil
}

func main() {
	r := bytes.NewReader([]byte("user_id,name\n1001,Alice\n"))

	text, err := readAll(r)
	if err != nil {
		panic(err)
	}
	fmt.Print(text)
}
```

运行方式：

Windows PowerShell：

```powershell
mkdir bytes-demo-03
cd bytes-demo-03
go mod init bytes-demo-03
notepad main.go
go run .
```

Linux/macOS：

```bash
mkdir bytes-demo-03
cd bytes-demo-03
go mod init bytes-demo-03
cat > main.go <<'EOF'
package main

import (
	"bytes"
	"fmt"
	"io"
)

func readAll(r io.Reader) (string, error) {
	data, err := io.ReadAll(r)
	if err != nil {
		return "", err
	}
	return string(data), nil
}

func main() {
	r := bytes.NewReader([]byte("user_id,name\n1001,Alice\n"))

	text, err := readAll(r)
	if err != nil {
		panic(err)
	}
	fmt.Print(text)
}
EOF
go run .
```

## 三、关键代码结构

`bytes.NewReader(data)`

创建一个从 `data` 读取的 Reader。它不会复制 `data`，所以不要在读取过程中随意修改底层切片。

`io.ReadAll(r)`

读完全部内容。适合小数据，例如测试输入、小配置、短请求体。生产上传文件不建议无限制 `ReadAll`。

`r.Seek(0, 0)`

可以把读取位置重置到开头。适合需要重复读取同一份内存数据的测试。

## 四、真实后端场景示例

假设你要测试用户导入功能，只关心 CSV 内容是否能被读取。

```go
package main

import (
	"bufio"
	"bytes"
	"fmt"
	"io"
	"strings"
)

func ImportUserNames(r io.Reader) ([]string, error) {
	scanner := bufio.NewScanner(r)
	var names []string

	firstLine := true
	for scanner.Scan() {
		line := scanner.Text()
		if firstLine {
			firstLine = false
			continue
		}

		parts := strings.Split(line, ",")
		if len(parts) != 2 {
			continue
		}
		names = append(names, parts[1])
	}

	if err := scanner.Err(); err != nil {
		return nil, err
	}
	return names, nil
}

func main() {
	data := []byte("user_id,name\n1001,Alice\n1002,Bob\n")
	names, err := ImportUserNames(bytes.NewReader(data))
	if err != nil {
		panic(err)
	}
	fmt.Println(names)
}
```

真实项目中，CSV 应优先用 `encoding/csv`。这里用 `bufio.Scanner` 是为了突出 `io.Reader` 的输入设计。

## 五、注意点

生产请求体要限制大小。

```text
外部输入必须设置大小边界。不要对用户上传内容无上限 io.ReadAll。
```

HTTP 场景常见写法是：

```go
limited := io.LimitReader(req.Body, 1<<20) // 最多读取 1 MiB
data, err := io.ReadAll(limited)
```

这不是 `bytes.Reader` 的功能，但它是后端处理字节输入时必须养成的习惯。

## 六、常见误区

误区一：为了测试真的创建临时文件。

如果函数只需要 `io.Reader`，用 `bytes.NewReader` 更简单，测试更快，也更稳定。

误区二：认为 `bytes.Reader` 会复制原始数据。

它读取的是传入切片的数据视图。读取过程中修改原始切片，可能影响结果。

误区三：生产导入也直接 `io.ReadAll`。

大文件导入应流式读取，并设计批量入库、事务边界和失败回滚策略。

## 七、本节达标标准

- 能用 `bytes.NewReader` 构造 `io.Reader`
- 能用它测试读取函数
- 知道 `io.ReadAll` 适合小数据，不适合无限制外部输入
- 能说出为什么函数参数设计成 `io.Reader` 更灵活

