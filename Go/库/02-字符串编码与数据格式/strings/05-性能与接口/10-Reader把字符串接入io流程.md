# 10. Reader 把字符串接入 io 流程

本节目标：你能用 `strings.NewReader` 把字符串变成 `io.Reader`，从而复用读取、测试、解析相关的后端代码。

简短引入：很多 Go 标准库和第三方库不直接接收字符串，而是接收 `io.Reader`。比如读取请求体、解析 CSV、上传对象、计算哈希。`strings.NewReader` 可以把字符串接入这些流程。

## 一、为什么需要它

可以理解为：`NewReader` 给字符串套上一层“可读取的数据流接口”。

常见场景是：

- 单元测试中模拟 HTTP 请求体。
- 解析一段 CSV 文本。
- 把配置字符串交给需要 reader 的解析器。
- 测试上传逻辑而不创建临时文件。

```text
Reader 让你的业务函数依赖接口，而不是依赖具体文件或网络。
```

## 二、基本用法

```go
package main

import (
	"fmt"
	"io"
	"strings"
)

func main() {
	r := strings.NewReader("hello backend")
	data, err := io.ReadAll(r)
	if err != nil {
		panic(err)
	}
	fmt.Println(string(data))
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

这个命令用于验证字符串可以被当作 reader 读取。真实项目中，这常用于测试和适配库函数。

## 三、关键代码结构

`strings.NewReader(s)` 返回 `*strings.Reader`。

它实现了多个读取相关接口，常见的是 `io.Reader`。你暂时可以理解为：凡是函数参数需要 `io.Reader`，字符串都可以用它包装后传进去。

有一个细节要记住：reader 会随着读取向后移动。读完一次后，再读通常就没有数据了。测试代码里如果要读第二次，通常重新创建一个 `strings.NewReader(raw)`，不要假设同一个 reader 可以自动回到开头。

这和真实请求体很像。HTTP 请求体读过一次后，后面的代码再读就可能读不到内容。所以中间件、日志、handler 要约定好谁负责读取请求体。

## 四、真实后端场景示例

下面解析一段 CSV 风格的用户导入数据：

```go
package main

import (
	"encoding/csv"
	"fmt"
	"strings"
)

func main() {
	raw := "email,name\nalice@example.com,Alice\nbob@example.com,Bob\n"
	reader := csv.NewReader(strings.NewReader(raw))

	records, err := reader.ReadAll()
	if err != nil {
		fmt.Println("解析失败:", err)
		return
	}

	for _, row := range records {
		fmt.Printf("%q\n", row)
	}
}
```

真实项目中，导入用户不能只解析 CSV 就入库。通常还要逐行校验、批量写入、控制事务大小、记录失败行，并保证失败后可以重试。

再看一个更贴近接口测试的例子：用字符串模拟 JSON 请求体。

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"strings"
)

type CreateUserRequest struct {
	Email string `json:"email"`
	Name  string `json:"name"`
}

func DecodeCreateUser(r io.Reader) (CreateUserRequest, error) {
	var req CreateUserRequest
	if err := json.NewDecoder(r).Decode(&req); err != nil {
		return req, err
	}
	req.Email = strings.ToLower(strings.TrimSpace(req.Email))
	req.Name = strings.TrimSpace(req.Name)
	return req, nil
}

func main() {
	body := `{"email":" Alice@Example.COM ","name":" Alice "}`
	req, err := DecodeCreateUser(strings.NewReader(body))
	if err != nil {
		fmt.Println("解析失败:", err)
		return
	}
	fmt.Printf("%+v\n", req)
}
```

这个写法的价值在于：`DecodeCreateUser` 依赖的是 `io.Reader`，所以生产环境可以传 HTTP 请求体，测试环境可以传 `strings.NewReader`。函数更容易测试，也更容易复用。

## 五、注意点

`strings.NewReader` 适合已有字符串。如果数据很大，比如几百 MB 的导入文件，不建议先整文件读成字符串再包装。真实项目中应直接使用文件 reader 或请求体 reader，避免占用过多内存。

还有一个判断标准：如果数据来源本来就是文件、网络连接或请求体，就尽量保持流式处理；如果数据本来就是一小段测试文本或配置文本，用 `strings.NewReader` 很合适。

迁移和导入类任务要保守：

- 小批量提交。
- 明确事务边界。
- 记录处理进度。
- 保留失败原因。
- 支持回滚或重跑。

## 六、常见误区

误区一：为了用 `NewReader` 把大文件全部读进内存。

这会造成内存压力。大文件应流式处理。

误区二：解析成功就直接入库。

解析只是第一步。还需要校验、去重、事务和错误处理。

误区三：测试只覆盖正常输入。

导入解析要测试空文件、缺列、多列、非法邮箱、重复数据。

误区四：不知道 reader 会被读完。

同一个 reader 被 `ReadAll` 或解码器读完后，后续再读可能是空。测试里要么重新创建 reader，要么明确使用支持 seek 的 reader 并重置位置。

## 七、本节达标标准

- 能用 `strings.NewReader` 适配需要 `io.Reader` 的函数。
- 能用它模拟 CSV 或请求体输入。
- 能说明大文件不应先整体转字符串。
- 能说出导入流程中的校验、事务和可重试要求。
- 能解释 reader 读完后不能默认再次读取。
