# 01. byte slice 与 string 的边界

本节目标：学完后你能在后端代码中判断什么时候用 `string`，什么时候用 `[]byte`，并安全地在两者之间转换。

## 简短引入

后端项目里，用户提交 JSON、上传文件、网关转发请求、日志系统写入内容，本质上都可能先表现为一段字节。Go 用 `[]byte` 表示这类可变字节数据，用 `string` 表示文本。

可以先这样理解：

- `string` 更适合“已经确认是文本”的业务字段，比如用户名、文章标题、订单状态
- `[]byte` 更适合“从 I/O 来、准备写到 I/O 去、或者还没确认结构”的原始数据

## 一、为什么需要它

真实项目中常见的边界是 HTTP 请求体。

请求刚进来时，它不是一个结构体，也不是一个字符串，而是一段可以读取的字节流。你可能会把它交给 JSON 解码器，也可能先做大小限制、日志采样、签名校验。

如果一上来就把所有内容转成字符串，容易出现两个问题：

- 无意义的内存拷贝
- 把二进制内容误当文本处理

```text
只有确认数据是文本，并且确实需要文本操作时，再把 []byte 转成 string。
```

## 二、基本用法

新建临时目录运行：

Windows PowerShell：

```powershell
mkdir bytes-demo-01
cd bytes-demo-01
go mod init bytes-demo-01
notepad main.go
go run .
```

Linux/macOS：

```bash
mkdir bytes-demo-01
cd bytes-demo-01
go mod init bytes-demo-01
cat > main.go <<'EOF'
package main

import (
	"bytes"
	"fmt"
)

func main() {
	raw := []byte("  user:alice  ")

	clean := bytes.TrimSpace(raw)
	text := string(clean)

	fmt.Println(text)
	fmt.Println(bytes.HasPrefix(clean, []byte("user:")))
}
EOF
go run .
```

Windows 用户把下面内容保存到 `main.go`：

```go
package main

import (
	"bytes"
	"fmt"
)

func main() {
	raw := []byte("  user:alice  ")

	clean := bytes.TrimSpace(raw)
	text := string(clean)

	fmt.Println(text)
	fmt.Println(bytes.HasPrefix(clean, []byte("user:")))
}
```

运行结果：

```text
user:alice
true
```

## 三、关键代码结构

`raw := []byte("  user:alice  ")`

可以理解为模拟从请求体、文件或消息队列里读到的原始内容。

`bytes.TrimSpace(raw)`

去掉前后空白。常见场景是清洗用户输入、日志字段、CSV 字段。

`string(clean)`

把字节转换成文本。真实项目中通常在要打印、拼业务错误信息、放入结构体字段时才这么做。

`bytes.HasPrefix(clean, []byte("user:"))`

判断字节内容是否有指定前缀。用于解析简单协议、请求头、日志行很常见。

## 四、真实后端场景示例

假设一个内部接口接收这样的文本：

```text
  user_id=1001
```

你希望先清理空白，再判断格式。

```go
package main

import (
	"bytes"
	"errors"
	"fmt"
)

func parseUserLine(raw []byte) (string, error) {
	line := bytes.TrimSpace(raw)
	key, value, ok := bytes.Cut(line, []byte("="))
	if !ok {
		return "", errors.New("invalid user line")
	}
	if !bytes.Equal(key, []byte("user_id")) {
		return "", errors.New("unexpected key")
	}
	return string(value), nil
}

func main() {
	userID, err := parseUserLine([]byte("  user_id=1001  "))
	if err != nil {
		panic(err)
	}
	fmt.Println(userID)
}
```

这里没有直接用字符串切分，是因为入口数据本来就是 `[]byte`。我们只在最后确定要返回业务字段时才转成 `string`。

## 五、注意点

`[]byte` 是可变的，函数里改了它，调用方看到的内容也可能变。

```go
func maskFirstByte(b []byte) {
	if len(b) > 0 {
		b[0] = '*'
	}
}
```

真实项目中，如果你要长期保存一份请求体、签名原文、缓存值，不要直接持有别人传进来的切片，后面会在复制章节专门讲。

## 六、常见误区

误区一：所有请求体都先转字符串。

这样写简单，但如果请求体很大，会增加内存压力。上传文件、导入数据、图片、压缩包都不应该当普通字符串处理。

误区二：为了性能到处避免转换。

初学阶段不需要过度紧张。真正需要关注的是热点路径、大数据量、循环里的重复转换。

误区三：用 `bytes` 解析复杂格式。

JSON 用 `encoding/json`，CSV 用 `encoding/csv`，SQL 用数据库驱动参数化能力。`bytes` 适合轻量处理，不适合硬写复杂解析器。

## 七、本节达标标准

- 能说明 `string` 和 `[]byte` 的基本区别
- 能用 `bytes.TrimSpace`、`bytes.HasPrefix` 处理字节输入
- 能判断什么时候把 `[]byte` 转成 `string`
- 能避免把上传文件、二进制内容直接当字符串处理

