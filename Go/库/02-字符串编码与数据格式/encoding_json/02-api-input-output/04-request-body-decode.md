# 04. 解析请求体并做第一层校验

本节目标：学完后你能从 HTTP 请求体解析 JSON，并在进入业务逻辑前完成基础校验。

简短引入：创建用户、发布文章、提交订单这些接口，第一步通常都是读取请求体。`encoding/json` 能帮你把 JSON 请求体解码成结构体，但它不会替你判断业务是否合理。

## 一、为什么需要它

可以把请求体解析理解为“把外部输入放进后端能处理的形状”。外部输入不可信，后端必须先确认 JSON 格式正确，再确认字段是否满足业务要求。

常见场景是创建短链接：

- 用户传入原始链接。
- 后端解析 JSON。
- 校验链接不能为空。
- 生成短码。
- 写入数据库。

```text
所有来自客户端的数据，都应该先解析、再校验、再进入业务层。
```

## 二、基本用法

Windows PowerShell：

```powershell
mkdir json-request-body
cd json-request-body
go mod init json-request-body
notepad main.go
go run .
```

Linux/macOS：

```bash
mkdir json-request-body
cd json-request-body
go mod init json-request-body
nano main.go
go run .
```

`main.go`：

```go
package main

import (
	"encoding/json"
	"log"
	"net/http"
	"strings"
)

type CreateUserRequest struct {
	Name  string `json:"name"`
	Email string `json:"email"`
}

type APIResponse struct {
	Code    string `json:"code"`
	Message string `json:"message"`
}

func writeJSON(w http.ResponseWriter, status int, body APIResponse) {
	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.WriteHeader(status)
	_ = json.NewEncoder(w).Encode(body)
}

func main() {
	http.HandleFunc("/users", func(w http.ResponseWriter, r *http.Request) {
		if r.Method != http.MethodPost {
			writeJSON(w, http.StatusMethodNotAllowed, APIResponse{"METHOD_NOT_ALLOWED", "method not allowed"})
			return
		}

		var req CreateUserRequest
		if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
			writeJSON(w, http.StatusBadRequest, APIResponse{"BAD_JSON", "invalid json body"})
			return
		}

		req.Name = strings.TrimSpace(req.Name)
		req.Email = strings.TrimSpace(req.Email)
		if req.Name == "" || req.Email == "" {
			writeJSON(w, http.StatusBadRequest, APIResponse{"INVALID_ARGUMENT", "name and email are required"})
			return
		}

		writeJSON(w, http.StatusCreated, APIResponse{"OK", "user created"})
	})

	log.Println("server listening on http://localhost:8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

请求接口。

Windows PowerShell：

```powershell
Invoke-RestMethod -Method Post -Uri http://localhost:8080/users -ContentType "application/json" -Body '{"name":"Alice","email":"alice@example.com"}'
```

Linux/macOS：

```bash
curl -i -X POST http://localhost:8080/users \
  -H 'Content-Type: application/json' \
  -d '{"name":"Alice","email":"alice@example.com"}'
```

## 三、关键参数/语法/代码结构

`json.NewDecoder(r.Body).Decode(&req)` 从请求体流中读取 JSON，并写入 `req`。这里用 `Decoder`，是因为 HTTP 请求体本来就是一个流。

`&req` 必须是指针，否则解码结果无法写回变量。

`strings.TrimSpace` 是常见的第一层清洗。真实项目中，用户输入经常带有多余空格，不清洗会导致“看起来有值，实际不可用”的问题。

状态码 `400 Bad Request` 适合表示请求格式错误或参数不合法。不要把这类客户端错误返回成 `500`。

## 四、真实后端场景示例

创建短链接接口可以这样设计请求体：

```go
type CreateShortLinkRequest struct {
	OriginalURL string `json:"original_url"`
	ExpireDays  int    `json:"expire_days"`
}

func validateCreateShortLink(req CreateShortLinkRequest) error {
	if strings.TrimSpace(req.OriginalURL) == "" {
		return errors.New("original_url is required")
	}
	if req.ExpireDays < 1 || req.ExpireDays > 365 {
		return errors.New("expire_days must be between 1 and 365")
	}
	return nil
}
```

进入数据库前，应该再进入业务层生成短码，并使用参数化查询保存。

```text
JSON 校验解决输入形状问题，参数化查询解决 SQL 注入问题，它们不是同一层防线。
```

## 五、注意点

真实项目中要限制请求体大小，否则恶意客户端可以发送很大的 JSON，占用内存和连接资源。标准库里常见做法是配合 `http.MaxBytesReader`。

不要在 handler 里写太多业务逻辑。handler 适合做协议层工作，例如解析请求、返回响应；业务规则可以放到 service 函数里。

如果接口要写数据库，要明确事务边界。比如创建订单时同时写订单表和库存流水表，应该放在同一个事务中，失败时可回滚。

## 六、常见误区

误区一：只要 JSON 能解析就直接保存。这样会把空名称、负库存、非法状态写进系统。

误区二：把参数错误返回 `500`。`500` 表示服务端错误，会误导监控和调用方。

误区三：在 SQL 中拼接 JSON 字段值。用户输入必须使用参数化查询或 ORM 的安全绑定方式。

误区四：忘记限制请求体大小。小项目不明显，生产环境容易被大请求拖慢。

## 七、本节达标标准

- 能用 `json.NewDecoder(r.Body).Decode` 解析请求体。
- 能在 handler 中区分 JSON 格式错误和业务参数错误。
- 能对字符串字段做基本清洗。
- 能解释为什么客户端输入不能直接进入数据库。
- 能说出请求体大小限制的意义。

