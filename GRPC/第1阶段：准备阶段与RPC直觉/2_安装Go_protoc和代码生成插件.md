# 2. 安装 Go、protoc 和代码生成插件

本节目标：完成 gRPC Go 开发所需的本地工具链，并理解每个工具的职责。

gRPC Go 开发至少需要三类工具：Go 编译器、Protobuf 编译器 `protoc`、Go 代码生成插件。很多初学者卡在第一步，不是代码写错，而是工具链没有放进 PATH。

---

## 一、核心直觉

- `go` 负责构建 Go 项目。
- `protoc` 负责读取 `.proto` 文件并调用插件生成代码。
- `protoc-gen-go` 生成 Protobuf message 代码。
- `protoc-gen-go-grpc` 生成 gRPC client/server 接口代码。
- PATH 配置不正确时，`protoc` 会提示找不到插件。

---

## 二、动手步骤

1. 安装 Go，并确认 `go version` 能输出版本。
2. 安装 `protoc`，确认 `protoc --version` 能输出版本。
3. 使用 `go install` 安装两个生成插件。
4. 确认 `$(go env GOPATH)/bin` 在 PATH 中。
5. 新开一个终端重新验证命令。

---

## 三、参考代码或命令

```bash
go version
protoc --version

go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
```

PowerShell 检查插件：

```powershell
Get-Command protoc-gen-go
Get-Command protoc-gen-go-grpc
```

如果找不到：

```powershell
$env:Path += ";$(go env GOPATH)\bin"
```

---

## 四、常见问题

- `protoc` 已安装但插件找不到：通常是 GOPATH 的 bin 目录没有加入 PATH。
- 安装插件后不重开终端：旧终端可能没有读取新的 PATH。
- 混淆 `protoc-gen-go` 和 `protoc-gen-go-grpc`：前者生成消息结构，后者生成服务接口。

---

## 五、练习任务

- 在终端中分别执行 `go version`、`protoc --version`、`protoc-gen-go --version`。
- 故意移除 PATH 后观察错误，再恢复 PATH，加深理解。
- 记录你的 Go 版本、protoc 版本和插件安装位置。

---

## 六、完成标准

- 终端可以识别 `go`、`protoc`、`protoc-gen-go`、`protoc-gen-go-grpc`。
- 你知道每个工具负责什么。
- 后续生成代码不会因为基础工具链卡住。


---

## 七、Windows 上的安装与排错细节

很多教程默认 Linux/macOS 环境，但你当前在 Windows + PowerShell 下学习，所以要特别关注 PATH 和命令换行。

### 1. 检查 Go

```powershell
go version
```

预期输出类似：

```text
go version go1.22.5 windows/amd64
```

如果提示 `go : 无法将“go”项识别为 cmdlet`，说明 Go 没有安装，或者 Go 的 bin 目录没有加入 PATH。

### 2. 检查 GOPATH

```powershell
go env GOPATH
```

通常输出类似：

```text
C:\Users\你的用户名\go
```

Go 安装的命令行工具会放在：

```text
$(go env GOPATH)\bin
```

后面的 `protoc-gen-go` 和 `protoc-gen-go-grpc` 就会安装到这里。

---

## 八、安装 protoc 的验证方式

`protoc` 是 Protocol Buffers 编译器。它不是 Go 专属工具，而是读取 `.proto` 文件并调用插件生成目标语言代码。

安装完成后执行：

```powershell
protoc --version
```

预期输出类似：

```text
libprotoc 27.2
```

版本号不需要和示例完全一样，但必须能正常输出。

如果提示找不到命令，检查：

1. 是否真的安装了 protoc。
2. `protoc.exe` 所在目录是否加入 PATH。
3. 是否打开了新的 PowerShell 窗口。

---

## 九、安装 Go 插件

执行：

```powershell
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
```

安装完成后检查：

```powershell
Get-Command protoc-gen-go
Get-Command protoc-gen-go-grpc
```

也可以执行：

```powershell
protoc-gen-go --version
protoc-gen-go-grpc --version
```

如果 `Get-Command` 找不到插件，临时加入 PATH：

```powershell
$env:Path += ";$(go env GOPATH)\bin"
```

注意：这个命令只对当前 PowerShell 窗口生效。长期生效需要在系统环境变量里添加 GOPATH 的 bin 目录。

---

## 十、最小验证实验

创建一个临时目录：

```powershell
mkdir grpc-tool-check
cd grpc-tool-check
```

创建 `hello.proto`：

```proto
syntax = "proto3";

package hello.v1;

option go_package = "example.com/grpc-tool-check/hello/v1;hellov1";

message HelloRequest {
  string name = 1;
}
```

生成 Go 代码：

```powershell
protoc --go_out=. --go_opt=paths=source_relative hello.proto
```

预期结果：当前目录出现：

```text
hello.pb.go
```

如果这个文件能生成，说明 `protoc` 和 `protoc-gen-go` 已经协作成功。

---

## 十一、常见错误排查

### 错误 1：protoc-gen-go: program not found

现象：

```text
protoc-gen-go: program not found or is not executable
```

原因：`protoc` 能运行，但找不到 `protoc-gen-go` 插件。

解决：

```powershell
$env:Path += ";$(go env GOPATH)\bin"
Get-Command protoc-gen-go
```

### 错误 2：Missing go_package option

现象：

```text
unable to determine Go import path for "hello.proto"
```

原因：proto 文件里没有写 `option go_package`。

解决：在 proto 中加入：

```proto
option go_package = "example.com/your-module/path;packagename";
```

### 错误 3：PowerShell 换行符写错

PowerShell 使用反引号换行：

```powershell
protoc --go_out=. --go_opt=paths=source_relative `
  hello.proto
```

Linux/macOS 使用反斜杠：

```bash
protoc --go_out=. --go_opt=paths=source_relative \
  hello.proto
```

不要混用。

---

## 十二、完成标准

本节结束时，你应该能在任意新 PowerShell 窗口中完成：

```powershell
go version
protoc --version
protoc-gen-go --version
protoc-gen-go-grpc --version
```

并且能用一个最小 `hello.proto` 成功生成 `hello.pb.go`。
---

## 教程闭环检查

为了保证本节不是只停留在概念层面，学习时请按下面闭环完成：

1. **完整操作步骤**：先按正文顺序完成本节涉及的环境检查、文件创建、proto 编写、代码生成或运行验证。
2. **完整代码或命令**：本节如果涉及代码，请使用正文中的完整示例；如果是概念准备章节，请至少执行或记录正文给出的检查命令。
3. **运行命令**：把本节出现的关键命令实际运行一遍，例如 `go version`、`protoc --version`、`protoc ...`、`go run ...` 或 `grpcurl ...`。
4. **预期输出**：运行后对照正文中的预期输出；如果输出不同，先不要跳到下一节。
5. **常见错误排查**：遇到 PATH、go_package、端口占用、连接失败、生成文件缺失等问题时，优先按本节排错思路定位。
6. **练习任务**：完成正文中的练习，不只复制代码，要至少做一次参数或字段修改。
7. **完成标准**：能不看教程复述本节做了什么、为什么这样做、出错时从哪里查，才算真正完成。