# 03. 在 HTTP 接口中返回 JSON

本节目标：学完后你能写一个最小 HTTP 接口，把业务响应以 JSON 格式返回给客户端。

简短引入：`json.Marshal` 只是把数据变成 JSON 字节。真正的后端接口还要设置状态码、响应头，并把 JSON 写进 `http.ResponseWriter`。

## 一、为什么需要它

真实项目中，前端调用接口时不只关心响应内容，也关心响应头和状态码。比如：

- `Content-Type` 告诉客户端这是 JSON。
- `StatusCode` 告诉客户端请求是成功、参数错误还是服务异常。
- 响应体承载具体业务数据或错误信息。

```text
接口响应应该同时表达 HTTP 结果和业务结果。
```

## 二、基本用法

Windows PowerShell：

```powershell
mkdir json-http-response
cd json-http-response
go mod init json-http-response
notepad main.go
go run .
```

Linux/macOS：

```bash
mkdir json-http-response
cd json-http-response
go mod init json-http-response
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
)

type UserResponse struct {
	ID    int    `json:"id"`
	Name  string `json:"name"`
	Email string `json:"email"`
}

func main() {
	http.HandleFunc("/users/me", func(w http.ResponseWriter, r *http.Request) {
		resp := UserResponse{
			ID:    1001,
			Name:  "Alice",
			Email: "alice@example.com",
		}

		w.Header().Set("Content-Type", "application/json; charset=utf-8")
		w.WriteHeader(http.StatusOK)

		if err := json.NewEncoder(w).Encode(resp); err != nil {
			log.Println("encode response:", err)
		}
	})

	log.Println("server listening on http://localhost:8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

启动后请求接口。

Windows PowerShell：

```powershell
Invoke-RestMethod http://localhost:8080/users/me
```

Linux/macOS：

```bash
curl -i http://localhost:8080/users/me
```

## 三、关键参数/语法/代码结构

`w.Header().Set("Content-Type", "application/json; charset=utf-8")` 要在写响应体之前设置。响应一旦开始写入，很多头信息就不能再安全修改。

`w.WriteHeader(http.StatusOK)` 设置 HTTP 状态码。如果不手动设置，第一次 `Write` 时默认是 200。

`json.NewEncoder(w).Encode(resp)` 会把结构体编码后直接写入响应流，并在末尾加一个换行。对于 HTTP 响应，这是很常见的写法。

## 四、真实后端场景示例

项目中通常会封装一个统一响应函数，减少每个 handler 的重复代码。

```go
package main

import (
	"encoding/json"
	"log"
	"net/http"
)

type APIResponse struct {
	Code    string `json:"code"`
	Message string `json:"message"`
	Data    any    `json:"data,omitempty"`
}

func writeJSON(w http.ResponseWriter, status int, body APIResponse) {
	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.WriteHeader(status)
	if err := json.NewEncoder(w).Encode(body); err != nil {
		log.Println("write json:", err)
	}
}

func main() {
	http.HandleFunc("/articles/10", func(w http.ResponseWriter, r *http.Request) {
		article := map[string]any{
			"id":    10,
			"title": "Go JSON 实战",
		}

		writeJSON(w, http.StatusOK, APIResponse{
			Code:    "OK",
			Message: "success",
			Data:    article,
		})
	})

	log.Println("server listening on http://localhost:8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

真实项目中，`Code` 可以用于业务错误码，HTTP 状态码用于协议层结果。两者不要混在一起。

## 五、注意点

不要在已经写入响应体后再设置状态码。那时状态码可能已经发出，修改不会生效。

不要把内部错误原样返回给前端。比如数据库连接串、SQL 语句、堆栈信息都不应该进入响应体。

如果接口已经写入一部分响应，再发生编码错误，通常已经无法返回一个干净的错误 JSON。复杂场景可以先编码到缓冲区，再统一写出。

## 六、常见误区

误区一：忘记设置 `Content-Type`。有些客户端仍能解析，但接口行为不规范。

误区二：把所有错误都返回 200。这样监控、网关和客户端都很难判断真实状态。

误区三：直接返回 `err.Error()`。真实项目中内部错误应该写日志，对外返回稳定错误码和友好信息。

误区四：每个接口都手写一遍响应结构。重复代码多了以后，状态码和错误格式很容易不一致。

## 七、本节达标标准

- 能用 `json.NewEncoder(w).Encode` 返回 JSON 响应。
- 能正确设置 `Content-Type` 和 HTTP 状态码。
- 能写一个简单的统一响应函数。
- 能区分 HTTP 状态码和业务错误码。
- 能避免把内部错误细节直接暴露给客户端。

