> `2026/06/11 21:19:43`

在 Go 语言中，`os.Getwd()` 是 `os` 标准库中用于**获取当前工作目录**（Get Working Directory）的函数。简单来说，它告诉你：“当前这个 Go 程序是在哪个目录下被执行的？”

这在需要读取本地相对路径文件（如配置文件、模板文件、日志输出路径）时非常重要。以下是详细的用法和极其容易踩坑的注意事项：

## 基本用法

`os.Getwd()` 没有参数，返回值有两个：一个是表示当前路径的字符串，另一个是 error。

```go
package main

import (
	"fmt"
	"os"
)

func main() {
	// 获取当前工作目录
	dir, err := os.Getwd()
	if err != nil {
		fmt.Printf("获取工作目录失败: %v\n", err)
		return
	}

	fmt.Printf("当前工作目录是: %s\n", dir)
}
```

**为什么会返回 error？** 虽然很少见，但 `os.Getwd()` 确实可能失败。例如：

- 当前目录在你运行程序后，被其他终端或程序物理删除了。
- 权限问题导致操作系统拒绝返回路径。

## 核心陷阱：“当前工作目录” ≠ “程序文件所在目录”

这是 Go 新手（甚至是老手）最容易踩的坑。**`os.Getwd()` 返回的是你在终端里敲下运行命令时所在的那个目录，而不是你的 `.go` 代码文件或编译后的 `.exe`/二进制文件所在的目录。**

**场景演示：** 假设你的项目结构如下：

```
myproject/
├── config.yaml
└── cmd/
    └── main.go (里面有 os.Getwd() 和读取 config.yaml 的代码)
```

- **情况 A（正常）：** 你在终端里 `cd` 到了 `myproject/` 目录下，然后执行 `go run cmd/main.go`。 此时 `os.Getwd()` 返回的是 `/path/to/myproject`。你可以用相对路径 `./config.yaml` 顺利读到配置。
- **情况 B（报错）：** 你在终端里 `cd` 到了 `myproject/cmd/` 目录下，然后执行 `go run main.go`。 此时 `os.Getwd()` 返回的是 `/path/to/myproject/cmd`。程序会尝试去 `cmd/config.yaml` 找配置文件，导致**文件找不到**的报错。

## 进阶：如何安全地定位文件？

既然 `os.Getwd()` 依赖于用户的执行路径，十分不可控，在实际生产项目中，我们该如何安全地读取文件呢？

### 方案一：强制通过命令行参数传入绝对路径 (推荐)

不要依赖工作目录，直接让运行程序的人告诉程序配置在哪。

```bash
./myapp --config=/etc/myapp/config.yaml
```

### 方案二：获取“可执行文件”所在的真实目录

如果你希望把配置文件和编译好的二进制程序放在同一个文件夹里到处移动，你应该使用 `os.Executable()`，而不是 `os.Getwd()`。

```go
package main

import (
	"fmt"
	"os"
	"path/filepath"
)

func main() {
	// 1. 获取编译后的二进制文件本身的绝对路径
	exePath, err := os.Executable()
	if err != nil {
		panic(err)
	}

	// 2. 获取该文件所在的目录
	exeDir := filepath.Dir(exePath)

	fmt.Printf("无论你在哪里执行该程序，程序文件的真实目录始终是: %s\n", exeDir)
	
	// 然后你可以拼接路径寻找文件
	// configPath := filepath.Join(exeDir, "config.yaml")
}
```

> [!CAUTION]
>
> **⚠️ 注意：** `os.Executable()` 在使用 `go run` 命令时行为会比较特殊（因为 `go run` 会把代码编译到一个临时的临时目录去执行），它主要用于打包编译后的二进制文件（`go build`）。

## 相关兄弟函数：`os.Chdir()`

如果你想在程序运行期间**改变**当前工作目录（类似于终端里的 `cd` 命令），可以使用 `os.Chdir(dir string)`。

Go

```
// 将当前工作目录切换到 /tmp
err := os.Chdir("/tmp")
if err != nil {
    // 切换失败（例如目录不存在或无权限）
}
```

**总结：** `os.Getwd()` 简单易用，但它高度绑定于**用户执行命令时的上下文环境**。在处理相对路径时，一定要警惕执行路径变化带来的文件找不到（File Not Found）问题。