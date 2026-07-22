# 2. 创建第一个 Go Module 项目

本节目标：创建一个干净的 Go 后端学习项目，理解 `go.mod` 的作用，并能运行一个最小 Go 程序。

Gin 项目本质上也是 Go module 项目。先把 module 理解清楚，后面安装 Gin、GORM、JWT、Redis 客户端时才不会乱。

---

## 一、创建项目目录

在你喜欢的位置创建学习目录。

PowerShell：

```powershell
mkdir gin-lab
cd gin-lab
```

Linux/macOS：

```bash
mkdir gin-lab
cd gin-lab
```

建议本教程所有练习都放在这个目录中。

---

## 二、初始化 Go module

执行：

```bash
go mod init gin-lab
```

执行成功后，会生成：

```text
go.mod
```

查看内容：

```bash
type go.mod
```

Linux/macOS：

```bash
cat go.mod
```

你会看到类似：

```go
module gin-lab

go 1.22
```

含义：

- `module gin-lab`：当前项目的模块名。
- `go 1.22`：当前项目使用的 Go 语言版本。

---

## 三、模块名应该怎么取

学习项目可以直接用：

```bash
go mod init gin-lab
```

如果以后要发布到 GitHub，可以使用：

```bash
go mod init github.com/yourname/gin-lab
```

模块名会影响包的导入路径。

例如模块名是 `gin-lab`，后面内部包可能这样导入：

```go
import "gin-lab/internal/handler"
```

模块名是 `github.com/yourname/gin-lab`，导入路径就会变成：

```go
import "github.com/yourname/gin-lab/internal/handler"
```

初学阶段使用短模块名更简单。

---

## 四、创建第一个 Go 文件

创建：

```text
main.go
```

写入：

```go
package main

import "fmt"

func main() {
    fmt.Println("hello gin learning")
}
```

运行：

```bash
go run main.go
```

正常输出：

```text
hello gin learning
```

---

## 五、理解 `go run`

`go run main.go` 会做几件事：

```text
编译 main.go
运行编译后的临时程序
程序结束后清理临时文件
```

它适合开发和学习阶段。

后面部署时会使用：

```bash
go build
```

把程序编译成可执行文件。

---

## 六、整理项目当前结构

当前目录应该类似：

```text
gin-lab/
  go.mod
  main.go
```

可以执行：

```bash
Get-ChildItem
```

Linux/macOS：

```bash
ls
```

---

## 七、初始化 Git

执行：

```bash
git init
git status
```

提交当前项目：

```bash
git add .
git commit -m "init go module"
```

如果暂时还不熟 Git，至少要知道 `git status` 可以查看哪些文件被修改了。

---

## 八、常见问题

### 1. 忘记执行 `go mod init`

后面执行 `go get` 安装依赖时会遇到问题。

判断方式：项目根目录有没有 `go.mod`。

### 2. 在错误目录执行命令

如果你在桌面或上一级目录执行 `go mod init`，`go.mod` 就会生成在错误位置。

建议每次执行命令前确认当前目录：

PowerShell：

```powershell
Get-Location
```

Linux/macOS：

```bash
pwd
```

### 3. main 包写错

可执行程序入口文件必须是：

```go
package main
```

并且包含：

```go
func main() {}
```

---

## 九、本节练习

完成：

- 创建 `gin-lab`。
- 执行 `go mod init gin-lab`。
- 创建 `main.go`。
- 运行 `go run main.go`。
- 初始化 Git 并提交。

---

## 十、本节验收

你应该能够回答：

- `go.mod` 是做什么的？
- `module gin-lab` 会影响什么？
- `go run` 和 `go build` 有什么区别？
- 为什么执行命令前要确认当前目录？
- 一个可执行 Go 程序必须具备什么？


