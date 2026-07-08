# 06. 处理 JSON 解析错误

本节目标：学完后你能把常见 JSON 解析错误转换成清晰的接口响应，并保留服务端日志用于排查。

简短引入：后端接口不能只写“解析失败”。客户端需要知道请求哪里错了，服务端也需要日志定位问题。但错误信息不能暴露内部实现。

## 一、为什么需要它

用户提交 JSON 时，可能出现很多问题：

- JSON 语法错了。
- 字段类型错了。
- 请求体为空。
- 请求体太大。
- 同一个请求体里塞了多份 JSON。

后端要把这些情况尽量变成稳定的 400 响应，而不是让程序 panic 或返回模糊的 500。

```text
错误处理的目标不是暴露所有细节，而是让调用方能修正请求，让服务端能定位问题。
```

## 二、基本用法

Windows PowerShell：

```powershell
mkdir json-error-handling
cd json-error-handling
go mod init json-error-handling
notepad main.go
go run .
```

Linux/macOS：

```bash
mkdir json-error-handling
cd json-error-handling
go mod init json-error-handling
nano main.go
go run .
```

`main.go`：

```go
package main

import (
	"encoding/json"
	"errors"
	"io"
	"log"
	"net/http"
)

type CreateOrderRequest struct {
	UserID int `json:"user_id"`
	Qty    int `json:"qty"`
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

func decodeJSON(r *http.Request, dst any) error {
	dec := json.NewDecoder(r.Body)
	if err := dec.Decode(dst); err != nil {
		return err
	}
	if err := dec.Decode(&struct{}{}); !errors.Is(err, io.EOF) {
		return errors.New("body must contain only one json value")
	}
	return nil
}

func main() {
	http.HandleFunc("/orders", func(w http.ResponseWriter, r *http.Request) {
		var req CreateOrderRequest
		if err := decodeJSON(r, &req); err != nil {
			log.Println("decode order request:", err)
			writeJSON(w, http.StatusBadRequest, APIResponse{"BAD_JSON", "invalid json body"})
			return
		}
		if req.UserID <= 0 || req.Qty <= 0 {
			writeJSON(w, http.StatusBadRequest, APIResponse{"INVALID_ARGUMENT", "user_id and qty must be positive"})
			return
		}
		writeJSON(w, http.StatusCreated, APIResponse{"OK", "order created"})
	})

	log.Println("server listening on http://localhost:8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

测试类型错误。

Windows PowerShell：

```powershell
Invoke-WebRequest -Method Post -Uri http://localhost:8080/orders -ContentType "application/json" -Body '{"user_id":"wrong","qty":1}'
```

Linux/macOS：

```bash
curl -i -X POST http://localhost:8080/orders \
  -H 'Content-Type: application/json' \
  -d '{"user_id":"wrong","qty":1}'
```

## 三、关键参数/语法/代码结构

`Decode(dst)` 第一次读取请求体里的 JSON。

第二次 `Decode(&struct{}{})` 用来确认请求体里没有第二个 JSON 值。比如 `{"a":1}{"b":2}` 这种输入，不应该被当成正常请求。

`errors.Is(err, io.EOF)` 表示已经读到请求体结尾，这是我们期待的结果。

服务端日志可以记录底层错误，客户端响应保持稳定。这样既方便排查，也避免把内部细节暴露出去。

## 四、真实后端场景示例

订单接口的错误可以按层次处理：

```go
func validateCreateOrder(req CreateOrderRequest) error {
	if req.UserID <= 0 {
		return errors.New("user_id must be positive")
	}
	if req.Qty <= 0 {
		return errors.New("qty must be positive")
	}
	if req.Qty > 99 {
		return errors.New("qty is too large")
	}
	return nil
}
```

真实项目中，创建订单还会涉及库存扣减。库存扣减必须放进明确事务里，失败要回滚，不能 JSON 解析成功后就分散更新多张表。

```text
创建订单这类写操作，要先校验输入，再开启事务，再写库，失败时回滚。
```

## 五、注意点

不要把所有错误都变成同一个日志级别。用户传错参数是常见情况，通常不需要打成严重错误；数据库不可用才是服务端异常。

不要返回 Go 的原始错误字符串作为公共 API 契约。标准库错误信息可能变化，而且对用户不一定友好。

如果请求体很大，应该用 `http.MaxBytesReader` 限制大小，再解码。

## 六、常见误区

误区一：忽略 `Decode` 的错误。这样空请求体、类型错误、语法错误都可能继续进入业务层。

误区二：只解码一次，不检查额外内容。攻击者或错误客户端可能传入多段 JSON。

误区三：把解析错误返回 500。JSON 错误通常是客户端请求问题，应该返回 400。

误区四：日志里记录完整请求体。请求体可能包含手机号、地址、token，生产日志要注意脱敏。

## 七、本节达标标准

- 能识别 JSON 语法错误和类型错误都应该走 400。
- 能检查请求体是否只包含一个 JSON 值。
- 能把内部错误记录到日志，同时返回稳定错误响应。
- 能解释为什么解析成功后还要做业务校验。
- 能知道订单等写操作需要事务和可回滚设计。

