# 1. 安装与检查 Go 开发环境

本节目标：确认本机已经具备 Go 后端开发所需的基本环境，包括 Go、终端、编辑器、接口测试工具和 Git。

学习 Gin 前，环境一定要先稳定。否则后面你遇到错误时，很难判断是代码错了、依赖错了，还是环境本身没有配置好。

---

## 一、安装 Go

建议安装 Go 1.22 或更高版本。

安装完成后，在终端执行：

```bash
go version
```

正常输出类似：

```text
go version go1.22.5 windows/amd64
```

版本号不需要和这里完全一样，只要是 Go 1.22 或更高即可。

---

## 二、检查 Go 环境变量

执行：

```bash
go env GOPATH
go env GOPROXY
go env GOOS
go env GOARCH
```

这些命令分别用于查看：

- `GOPATH`：Go 的工作区和模块缓存目录。
- `GOPROXY`：Go 下载依赖时使用的代理。
- `GOOS`：当前目标操作系统。
- `GOARCH`：当前目标 CPU 架构。

在国内网络环境下，如果下载依赖很慢，可以设置代理：

```bash
go env -w GOPROXY=https://goproxy.cn,direct
```

设置后再次检查：

```bash
go env GOPROXY
```

应该看到：

```text
https://goproxy.cn,direct
```

---

## 三、准备编辑器

推荐二选一：

- VS Code
- GoLand

如果使用 VS Code，建议安装这些插件：

- Go
- REST Client
- GitLens
- Docker
- YAML

插件不是越多越好。初学阶段重点是让下面几件事顺手：

- 代码自动补全。
- 保存时格式化。
- 可以直接运行测试。
- 可以编辑 `.http` 接口测试文件。
- 可以查看 Git 修改记录。

---

## 四、准备接口测试工具

后端开发必须频繁测试接口。推荐选择一个：

- Apifox
- Postman
- Insomnia
- VS Code REST Client

如果你还没有偏好，建议先用 VS Code REST Client，因为请求可以写成文本文件，方便和代码一起保存。

示例文件：

```text
requests.http
```

内容：

```http
### ping
GET http://localhost:8080/ping
```

后面每写一个接口，都可以把请求保存进去。

---

## 五、准备 Git

检查 Git：

```bash
git --version
```

如果能看到版本号，说明 Git 可用。

配置用户名和邮箱：

```bash
git config --global user.name "your-name"
git config --global user.email "your-email@example.com"
```

学习阶段建议每完成一个小节就提交一次：

```bash
git add .
git commit -m "finish first http server"
```

这样后面代码写坏了，可以清楚知道是哪一步引入的问题。

---

## 六、常见问题

### 1. `go` 命令不存在

现象：

```text
go: command not found
```

或 Windows 中：

```text
'go' 不是内部或外部命令
```

排查：

- Go 是否安装成功。
- Go 的安装目录是否加入 PATH。
- 终端是否需要重新打开。

### 2. 下载依赖很慢

优先检查：

```bash
go env GOPROXY
```

如果不是国内代理，可以设置：

```bash
go env -w GOPROXY=https://goproxy.cn,direct
```

### 3. 编辑器没有代码提示

检查：

- 是否安装 Go 插件。
- 当前目录是否有 `go.mod`。
- 是否打开的是项目根目录，而不是单个文件。

---

## 七、本节练习

完成下面任务：

1. 执行 `go version`。
2. 执行 `go env GOPROXY`。
3. 安装或确认编辑器插件。
4. 安装或确认接口测试工具。
5. 执行 `git --version`。

把每个命令的输出记录到学习笔记里。

---

## 八、本节验收

你应该能够回答：

- `go version` 用来确认什么？
- `GOPROXY` 是做什么的？
- 为什么后端学习必须准备接口测试工具？
- 为什么建议每个小节提交一次 Git？
- 编辑器没有提示时应该先检查什么？


