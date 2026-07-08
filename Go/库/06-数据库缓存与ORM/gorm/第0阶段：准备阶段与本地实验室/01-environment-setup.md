# 01 开发环境准备

本节目标：围绕「environment setup」建立清晰的 GORM 学习目标，理解它解决的具体后端问题，能够写出可运行示例，并能通过数据库结果或 SQL 日志验证自己的代码是否正确。

## 1. 安装 Go

先确认本地是否已经安装 Go：

```bash
go version
```

如果能看到类似下面的输出，说明 Go 已经安装：

```text
go version go1.22.x windows/amd64
```

建议使用 Go 1.22 或更新版本。

## 2. 配置 Go Module

Go 后端项目一般使用 Go Module 管理依赖。创建项目时执行：

```bash
mkdir gorm-study
cd gorm-study
go mod init gorm-study
```

执行后会生成：

```text
go.mod
```

`go.mod` 用来记录项目模块名和依赖版本。以后安装 GORM、Gin、MySQL 驱动时，依赖都会记录在这里。

## 3. 安装数据库

推荐二选一：

### 方案 A：MySQL

适合大多数中文后端学习资料和企业项目。

你需要准备：

- MySQL 8.x
- 一个数据库用户
- 一个测试数据库

创建数据库示例：

```sql
CREATE DATABASE gorm_study DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### 方案 B：PostgreSQL

适合希望学习更强 SQL 能力和更规范数据库能力的同学。

创建数据库示例：

```sql
CREATE DATABASE gorm_study;
```

## 4. 安装数据库客户端

建议至少安装一个图形化工具：

- DBeaver
- DataGrip
- Navicat

图形化工具的作用不是替代 SQL，而是帮你观察：

- 表是否创建成功
- 字段类型是否符合预期
- 索引是否创建成功
- GORM 插入的数据是否正确

## 5. 准备 API 调试工具

后续工程化和项目实战阶段会写 HTTP API。建议准备：

- Apifox
- Postman
- curl

至少要能完成：

- 发送 GET 请求
- 发送 POST JSON 请求
- 设置 Authorization 请求头
- 查看响应状态码和响应体

## 6. 推荐编辑器

任选一个：

- VS Code + Go 插件
- GoLand

需要具备这些能力：

- 自动补全
- 跳转定义
- 格式化 Go 代码
- 运行测试
- 查看终端

## 7. 第一个 Go 项目检查

创建 `main.go`：

```go
package main

import "fmt"

func main() {
    fmt.Println("hello gorm")
}
```

运行：

```bash
go run .
```

如果输出：

```text
hello gorm
```

说明 Go 环境没有问题。

## 8. 常见问题

### `go` 命令找不到

说明 Go 没有加入 PATH。重新安装 Go，或者手动把 Go 的 `bin` 目录加入环境变量。

### 数据库连接不上

优先检查：

- 数据库服务是否启动
- 用户名和密码是否正确
- 端口是否正确
- 数据库名是否存在
- 防火墙是否拦截

### 中文乱码

MySQL 建库时建议使用 `utf8mb4`。连接字符串里也建议指定：

```text
charset=utf8mb4&parseTime=True&loc=Local
```

## 本节练习

- [ ] 安装 Go。
- [ ] 创建 `gorm-study` 项目。
- [ ] 初始化 `go.mod`。
- [ ] 创建 `gorm_study` 数据库。
- [ ] 用数据库客户端连接数据库。
- [ ] 运行第一个 `main.go`。
---

## 本节达标标准

学完本节后，你应该能够做到：

- 说清本节主题解决的具体后端开发问题。
- 写出本节涉及的核心 GORM 代码或 SQL 示例。
- 通过 SQL 日志、数据库客户端或测试验证结果是否正确。
- 识别本节列出的常见错误，并知道如何排查。
- 把本节知识放回博客系统或订单系统的真实场景中使用。
