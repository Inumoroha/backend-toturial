> `2026年6月11日`

```go
func NewScanner(r io.Reader) *Scanner {
	return &Scanner{
		r:            r,
		split:        ScanLines,
		maxTokenSize: MaxScanTokenSize,
	}
}
```

用于**按某种规则（默认是按行）读取数据流**。

类似方法: `io.Reader` 或者 `bufio.Reader` 的 `ReadLine` / `ReadString` 方法

## 基本用法: 按行读取 (默认行为)

`Scanner` 默认的分割规则是 `bufio.ScanLines`，也就是遇到换行符（`\n` 或 `\r\n`）就截断并返回一行数据。这在读取文件或标准输入时非常常用。

```go
package main

import (
	"bufio"
	"fmt"
	"strings"
)

func main() {
	// 模拟一个包含多行文本的输入源 (如文件、网络请求体等)
	input := "第一行数据\n第二行数据\n第三行数据"
	reader := strings.NewReader(input)

	// 1. 创建 Scanner 对象
	scanner := bufio.NewScanner(reader)

	// 2. 循环调用 Scan() 方法
	for scanner.Scan() {
		// 3. 使用 Text() 获取当前读取到的行 (不包含换行符)
		line := scanner.Text()
		fmt.Println("读取到:", line)
	}

	// 4. 检查是否发生错误
	// 注意：遇到 EOF (文件末尾) 时，Scan() 返回 false，但 Err() 返回 nil
	if err := scanner.Err(); err != nil {
		fmt.Printf("读取时发生错误: %v\n", err)
	}
}

// 读取到: 第一行数据
// 读取到: 第二行数据
// 读取到: 第三行数据
```

## 核心方法解析

`Scanner` 主要依赖以下四个核心方法完成工作：

- **`Scan() bool`**： 向前读取下一个 Token（默认是下一行）。如果读取成功返回 `true`；如果到达输入末尾 (EOF) 或者发生错误，返回 `false`。**这是写 `for` 循环的条件**。

- **`Text() string`**： 将最近一次 `Scan()` 读取到的数据以字符串（`string`）形式返回。它会自动去掉结尾的换行符。

- **`Bytes() []byte`**： 将最近一次 `Scan()` 读取到的数据以字节切片（`[]byte`）形式返回。

  **⚠️ 注意：** `Bytes()` 返回的底层数组是被 `Scanner` 内部复用的。如果要在下一次 `Scan()` 之后保留这些数据，必须手动 `copy`，否则数据会被覆盖。如果不想处理底层数组复用问题，直接用 `Text()`。

- **`Err() error`**： 在 `Scan()` 返回 `false` 后调用。如果是正常读取到了文件末尾，它会返回 `nil`。如果是遇到了其他真实错误（如网络断开、读取异常），它会返回具体的 `error`。

## 进阶用法：更改分割规则 (Split)

除了按行读取，`Scanner` 还内置了其他几种分割规则（Token 分割器），你甚至可以自定义规则。只需在开始 `Scan()` 之前调用 `scanner.Split()` 即可。

Go 语言内置的 Split 函数有：

1. **`bufio.ScanLines`** (默认)：按行分割。

2. **`bufio.ScanWords`**：按空格分割（自动去除连续空格），常用于统计单词。

3. **`bufio.ScanRunes`**：按 UTF-8 字符分割。

4. **`bufio.ScanBytes`**：按单个字节分割。

   示例：按单词读取

```go
package main

import (
	"bufio"
	"fmt"
	"strings"
)

func main() {
	input := "Hello   World! \n This is Go."
	scanner := bufio.NewScanner(strings.NewReader(input))

	// 切换为按“单词”分割
	scanner.Split(bufio.ScanWords)

	for scanner.Scan() {
		fmt.Printf("单词: %q\n", scanner.Text())
	}
}
// 输出:
// 单词: "Hello"
// 单词: "World!"
// 单词: "This"
// 单词: "is"
// 单词: "Go."
```

## 常见陷阱：单行数据过长 (Token Too Long)

这是使用 `bufio.NewScanner` 最容易踩的坑。`Scanner` 内部使用了一个定长的缓冲区来保存读取到的 Token（行）。默认情况下，这个缓冲区的最大容量是 **64 KB**（`bufio.MaxScanTokenSize`）。

如果你读取的文件中，**有一行的数据长度超过了 64 KB**，`scanner.Scan()` 会立即中止并返回 `false`，此时 `scanner.Err()` 会返回 `bufio.ErrTooLong`。

**解决方案：使用 `Buffer()` 方法扩大缓冲区**

在调用 `Scan()` 之前，手动分配一个更大的缓冲区：

```go
scanner := bufio.NewScanner(reader)

// 创建一个初始切片 (大小任意，比如 1024 字节)
buf := make([]byte, 1024)

// 定义允许的最大单行大小 (例如 1 MB)
const maxCapacity = 1024 * 1024 

// 设置缓冲区
scanner.Buffer(buf, maxCapacity)

for scanner.Scan() {
    // 正常读取...
}
```

## 总结建议

- **常规的文本文件读取**（日志、配置文件等），`bufio.NewScanner` 是首选，代码最优雅。
- **需要按特殊符号分割流**（如空格、特定分隔符），使用 `scanner.Split()`。
- **当确信文件可能存在极长的单行**（比如压缩过的单行巨大 JSON 文件），记得提前使用 `scanner.Buffer()` 调大缓冲区，或者改用 `bufio.Reader` 的 `ReadLine` / `ReadBytes` 来获得更底层的控制权。