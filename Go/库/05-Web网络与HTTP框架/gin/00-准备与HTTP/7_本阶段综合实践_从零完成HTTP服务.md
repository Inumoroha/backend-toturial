# 7. 本阶段综合实践：从零完成 HTTP 服务

本节目标：把第1阶段学到的 Go 环境、Go Module、标准库 HTTP、接口测试和 Git 工作流串成一次完整练习。

这不是新知识，而是第1阶段的收口任务。你要从一个空目录开始，完成一个可以运行、可以测试、可以提交的最小后端服务。

---

## 一、最终效果

完成后项目结构：

```text
hello-http/
├── go.mod
├── main.go
├── requests.http
├── README.md
└── .gitignore
```

服务提供三个接口：

```text
GET /ping
GET /hello?name=alice
GET /healthz
```

期望：

```text
/ping 返回 pong
/hello?name=alice 返回 hello alice
/healthz 返回 ok
```

---

## 二、第一步：创建项目

```bash
mkdir hello-http
cd hello-http
go mod init hello-http
```

检查：

```bash
go list -m
```

期望：

```text
hello-http
```

---

## 三、第二步：编写 main.go

创建 `main.go`：

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
)

type Response struct {
    Message string `json:"message"`
}

func main() {
    http.HandleFunc("/ping", pingHandler)
    http.HandleFunc("/hello", helloHandler)
    http.HandleFunc("/healthz", healthHandler)

    fmt.Println("server listening on :8080")
    if err := http.ListenAndServe(":8080", nil); err != nil {
        panic(err)
    }
}

func pingHandler(w http.ResponseWriter, r *http.Request) {
    writeJSON(w, http.StatusOK, Response{Message: "pong"})
}

func helloHandler(w http.ResponseWriter, r *http.Request) {
    name := r.URL.Query().Get("name")
    if name == "" {
        name = "guest"
    }

    writeJSON(w, http.StatusOK, Response{Message: "hello " + name})
}

func healthHandler(w http.ResponseWriter, r *http.Request) {
    writeJSON(w, http.StatusOK, Response{Message: "ok"})
}

func writeJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    _ = json.NewEncoder(w).Encode(data)
}
```

这里有几个重点：

- `http.HandleFunc` 注册路由。
- `r.URL.Query().Get("name")` 读取查询参数。
- `writeJSON` 统一设置 JSON 响应。
- `ListenAndServe(":8080", nil)` 启动服务。

---

## 四、第三步：启动和访问

启动：

```bash
go run main.go
```

新开一个终端访问：

```bash
curl -i http://localhost:8080/ping
curl -i "http://localhost:8080/hello?name=alice"
curl -i http://localhost:8080/healthz
```

期望：

```json
{"message":"pong"}
```

```json
{"message":"hello alice"}
```

```json
{"message":"ok"}
```

---

## 五、第四步：保存请求集合

创建 `requests.http`：

```http
### ping
GET http://localhost:8080/ping

### hello guest
GET http://localhost:8080/hello

### hello with name
GET http://localhost:8080/hello?name=alice

### health
GET http://localhost:8080/healthz

### not found
GET http://localhost:8080/not-exists
```

即使现在 404 还是标准库默认响应，也要保留这个请求。后面学 Gin 时你会把 404 改成统一 JSON。

---

## 六、第五步：编写 README

创建 `README.md`：

```md
# hello-http

第1阶段综合实践：使用 Go 标准库启动一个最小 HTTP 服务。

## 启动

```bash
go run main.go
```

## 接口

- `GET /ping`
- `GET /hello?name=alice`
- `GET /healthz`

## 测试

使用 `requests.http` 或 curl 访问接口。
```

README 不需要复杂，但要让别人知道项目怎么启动。

---

## 七、第六步：Git 保存

创建 `.gitignore`：

```gitignore
bin/
*.exe
*.log
.env
```

提交：

```bash
git init
git add .
git commit -m "complete first http service"
```

检查：

```bash
git status
```

期望：

```text
working tree clean
```

---

## 八、常见问题

### 1. `go run main.go` 后终端卡住

这是正常的。HTTP 服务正在运行。你需要新开一个终端测试接口。

### 2. 访问 `/hello?name=alice` 没有带参数

PowerShell 或某些终端中建议给 URL 加引号：

```bash
curl "http://localhost:8080/hello?name=alice"
```

### 3. 返回不是 JSON

检查是否设置：

```go
w.Header().Set("Content-Type", "application/json")
```

并使用 `json.NewEncoder(w).Encode(data)`。

---

## 九、扩展练习

完成基础版本后，继续尝试：

1. 增加 `GET /time` 返回当前时间。
2. 当 `/hello?name=` 为空时返回 `guest`。
3. 给不存在路由记录测试请求。
4. 把端口改成 `8081` 并更新 README。
5. 故意占用端口，练习排查。

---

## 十、最终验收

请确认：

- [ ] 能从空目录创建 Go Module。
- [ ] 能写出标准库 HTTP 服务。
- [ ] 能返回 JSON。
- [ ] 能读取查询参数。
- [ ] 能使用 curl 或 requests.http 测试。
- [ ] 能写 README。
- [ ] 能使用 Git 提交。
- [ ] 能解释服务运行时终端为什么不会退出。

完成后，第1阶段就不只是“环境装好了”，而是你已经亲手跑通了最小后端服务。

