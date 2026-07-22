# 0. 环境搭建与 Web 后端直觉总览

本阶段目标：在正式学习 Gin 之前，先把 Go 后端开发的基础环境、工具链、HTTP 请求直觉和学习工作流建立起来。

很多人学 Gin 时会直接从：

```go
r := gin.Default()
r.GET("/ping", ...)
r.Run()
```

开始。这样可以很快看到效果，但容易留下几个隐患：

- 不知道 Go module 是什么。
- 不知道服务启动后到底监听了什么端口。
- 不知道接口工具发出的请求包含哪些内容。
- 不知道 400、404、500 分别代表什么。
- 代码出了问题只会反复复制教程。

所以第一阶段不急着写复杂业务，只解决一个问题：

```text
我能在本地稳定地创建、运行、测试和排查一个 Go Web 服务。
```

---

## 一、本阶段适合什么时候学习

如果你已经会写 Go 基础语法，但没有系统写过后端服务，就应该从这里开始。

建议你至少具备：

- 知道变量、函数、结构体怎么写。
- 知道 `go run` 可以运行 Go 程序。
- 知道 JSON 大概长什么样。
- 能使用终端执行命令。

如果这些还不熟，也可以边做边查，不需要等到 Go 全部学完再开始。

---

## 二、本阶段核心内容

本阶段拆成以下小节：

1. 安装与检查 Go 开发环境。
2. 创建第一个 Go module。
3. 使用标准库启动一个 HTTP 服务。
4. 使用接口工具发送请求。
5. 理解 Web 后端的一次请求链路。
6. 建立 Git、调试和学习节奏。

学习 Gin 前先写一个标准库 HTTP 服务，是为了让你知道 Gin 到底帮你简化了什么。

---

## 三、本阶段统一练习项目

建议创建目录：

```text
gin-lab/
```

本阶段只写一个最小服务：

```text
GET /ping
GET /hello
```

目标不是功能复杂，而是完整走通：

```text
创建项目 -> 写代码 -> 启动服务 -> 发送请求 -> 查看响应 -> 查看日志 -> 修改代码 -> 再次验证
```

---

## 四、推荐学习顺序

建议按下面顺序学习：

```text
Go 环境检查
→ Go module 创建
→ 标准库 HTTP 服务
→ 接口测试工具
→ HTTP 请求链路
→ Git 与调试习惯
```

原因是：

- Go 环境不稳定，后面所有步骤都会受影响。
- module 是 Go 项目的基础。
- HTTP 服务是 Gin 的底层基础。
- 接口测试工具是后端学习每天都会用的工具。
- 调试习惯决定你能不能独立解决问题。

---

## 五、本阶段完成标准

完成本阶段后，你应该能够做到：

- 创建一个 Go module 项目。
- 写出一个标准库 HTTP 服务。
- 启动服务并访问 `GET /ping`。
- 使用接口测试工具发送 GET 请求。
- 看懂服务日志里请求方法、路径和状态码。
- 遇到端口占用、404、JSON 错误时知道先查哪里。
- 使用 Git 保存当前学习进度。

如果这些都能做到，再进入 Gin 会顺很多。

---

## 融合补充

+### 00-01 环境与学习工作流

#### 本节目标

搭好能够持续练习、调试和提交的 Go 后端环境。Gin 只是 HTTP 框架；后面的数据库、认证、测试和部署能力同样重要。先建立可重复的工作流，学习过程才不会停留在“看懂了”。

#### 准备清单

安装当前稳定版 Go，建议不低于 Go 1.22：

```bash
go version
go env GOPATH GOPROXY
```

网络环境需要时设置模块代理：

```bash
go env -w GOPROXY=https://goproxy.cn,direct
```

编辑器可选 VS Code（安装 Go、GitLens、REST Client、Docker、YAML 扩展）或 GoLand。再准备 Git 与一个接口测试工具，初学时 Postman、Apifox 或 VS Code REST Client 都可以。MySQL、Redis、Docker 可以在第 5、7、8 阶段开始使用。

创建学习仓库：

```bash
mkdir gin-learning
cd gin-learning
go mod init github.com/your-name/gin-learning
git init
git add .
git commit -m "chore: initialize gin learning project"
```

`go.mod` 记录模块名、Go 版本和依赖。Module 模式下项目可放在任意目录，不要再依赖 GOPATH 的旧目录约定。

#### 每日学习循环

每次只推进一个可验证的小目标：

1. 写出最小可运行实现。
2. 用 curl 或接口工具发送成功与失败请求。
3. 读日志和响应，定位问题后再重构。
4. 记录一个简短笔记：新概念、遇到的错误、解决原因。
5. 提交 Git，例如 `feat: add user create endpoint`。

不要一开始设计“完美架构”。先跑通，理解请求如何流过路由、Handler 和数据层，再在第 4 阶段重构。每个阶段都保留能运行的版本，后续出错时可用 `git diff` 和 `git bisect` 回溯。

#### 基础调试顺序

| 现象 | 首先检查 |
| --- | --- |
| 程序不能启动 | `go run .` 的首个错误、端口占用、模块依赖 |
| 请求 404 | 方法、路径、`/api/v1` 前缀、路由分组 |
| JSON 绑定失败 | `Content-Type`、字段名、结构体 tag、请求体是否为空 |
| 数据库失败 | DSN、容器网络、账户、端口和迁移 |
| 鉴权失败 | `Authorization` 格式、密钥、token 过期时间 |

排错先收集事实：请求方法、URL、状态码、响应体、服务日志和依赖日志。不要仅根据浏览器的一个报错猜测问题在 Gin。

#### 本阶段验收

- `go version`、`go env GOPROXY` 能正常输出。
- 已创建 Module 并完成第一次 Git 提交。
- 能选择一种方式发送 HTTP 请求并查看状态码、Header、响应体。
- 理解“先运行、再理解、再重构”的学习顺序。

+### 00-02 Go、HTTP 与 REST 基础

#### 本节目标

在使用 Gin 前亲手启动一次标准库 HTTP 服务，理解框架替你做了什么，并建立正确的状态码与资源设计习惯。

#### 第一个 HTTP 服务

创建 `main.go`：

```go
package main

import (
    "encoding/json"
    "log"
    "net/http"
)

func health(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodGet {
        w.WriteHeader(http.StatusMethodNotAllowed)
        return
    }
    w.Header().Set("Content-Type", "application/json; charset=utf-8")
    _ = json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
}

func main() {
    http.HandleFunc("/healthz", health)
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

运行并验证：

```bash
go run .
curl -i http://localhost:8080/healthz
curl -i -X POST http://localhost:8080/healthz
```

请求包含方法、路径、查询串、Header 和可选 body；响应包含状态码、Header 和 body。Gin 后续会封装路由匹配、Context、绑定、渲染和中间件，但不会改变这个模型。

#### REST 约定

路径表达资源，用复数名词：

```text
GET    /api/v1/users          查询用户
POST   /api/v1/users          创建用户
GET    /api/v1/users/:id      查询一个用户
PATCH  /api/v1/users/:id      局部更新用户
DELETE /api/v1/users/:id      删除用户
```

避免 `/getUsers`、`/createUser` 这类动词路径。查询条件放在查询串，如 `GET /api/v1/tasks?status=todo&page=1`；资源 ID 放在路径；创建或修改内容放 JSON body。

#### 状态码选择

| 场景 | 状态码 |
| --- | --- |
| 查询、修改成功 | `200 OK` |
| 创建成功 | `201 Created` |
| 删除成功且无 body | `204 No Content` |
| JSON、字段或分页参数不合法 | `400 Bad Request` |
| 未提供或无效登录凭据 | `401 Unauthorized` |
| 已登录但没有资源权限 | `403 Forbidden` |
| 资源不存在 | `404 Not Found` |
| 邮箱等唯一资源冲突 | `409 Conflict` |
| 预料外服务错误 | `500 Internal Server Error` |

HTTP 状态码反映协议结果；后续统一响应中的业务码表示更细的错误类别。不要把所有业务失败都包装成 `200`，也不要把 SQL、token 密钥或堆栈返回给客户端。

#### 本节验收

- 能解释一次 HTTP 请求和响应分别包含什么。
- `/healthz` 对 `GET` 返回 200，对 `POST` 返回 405。
- 能为“创建用户、重复邮箱、未登录访问、资源不存在”各选出正确状态码。

