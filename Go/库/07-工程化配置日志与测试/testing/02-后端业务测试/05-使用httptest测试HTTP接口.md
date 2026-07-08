# 5. 使用 httptest 测试 HTTP 接口

本节目标：学完后你能用标准库 `httptest` 测试 HTTP handler 的状态码、响应体和错误分支。

简短引入：后端接口不是只测 service 就够了。HTTP handler 负责解析请求、校验方法、读取 JSON、调用 service、返回状态码和响应体。`httptest` 可以在不真正监听端口的情况下测试这些行为。

## 一、为什么需要它

真实接口经常出错在这些地方：

- 请求方法不对时没有返回 `405`。
- JSON 格式错误时返回了 `500`。
- service 报错后状态码不准确。
- 响应字段名和前端约定不一致。
- 忘记设置 `Content-Type`。

`httptest` 可以理解为：

```text
在内存里构造一次 HTTP 请求，直接调用 handler，然后检查响应。
```

它比启动真实服务更快，也更适合日常单元测试。

## 二、基本用法

创建实验项目：

Windows PowerShell：

```powershell
mkdir httpdemo
cd httpdemo
go mod init example.com/httpdemo
```

Linux/macOS：

```bash
mkdir httpdemo
cd httpdemo
go mod init example.com/httpdemo
```

新建 `handler.go`：

```go
package httpdemo

import (
	"encoding/json"
	"errors"
	"net/http"
)

type User struct {
	ID    int64  `json:"id"`
	Email string `json:"email"`
}

type UserService interface {
	Create(email string) (User, error)
}

type UserHandler struct {
	users UserService
}

func NewUserHandler(users UserService) *UserHandler {
	return &UserHandler{users: users}
}

func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
		return
	}

	var req struct {
		Email string `json:"email"`
	}
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "invalid json", http.StatusBadRequest)
		return
	}

	user, err := h.users.Create(req.Email)
	if err != nil {
		if errors.Is(err, ErrInvalidEmail) {
			http.Error(w, "invalid email", http.StatusBadRequest)
			return
		}
		http.Error(w, "internal error", http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusCreated)
	_ = json.NewEncoder(w).Encode(user)
}

var ErrInvalidEmail = errors.New("invalid email")
```

## 三、关键参数/语法/代码结构

`httptest.NewRequest` 用来构造请求，不需要真实网络端口。

`httptest.NewRecorder` 用来记录 handler 写出的状态码、响应头和响应体。

`handler.ServeHTTP` 或直接调用 handler 方法，都可以触发处理逻辑。

测试 handler 时通常检查三类东西：

- 状态码是否符合接口约定。
- 响应体是否包含约定字段。
- service 是否收到正确参数。

## 四、真实后端场景示例

新建 `handler_test.go`：

```go
package httpdemo

import (
	"bytes"
	"encoding/json"
	"errors"
	"net/http"
	"net/http/httptest"
	"testing"
)

type fakeUserService struct {
	gotEmail string
	user     User
	err      error
}

func (f *fakeUserService) Create(email string) (User, error) {
	f.gotEmail = email
	return f.user, f.err
}

func TestUserHandler_CreateUser(t *testing.T) {
	tests := []struct {
		name       string
		method     string
		body       string
		serviceErr error
		wantStatus int
		wantEmail  string
	}{
		{
			name:       "created",
			method:     http.MethodPost,
			body:       `{"email":"alice@example.com"}`,
			wantStatus: http.StatusCreated,
			wantEmail:  "alice@example.com",
		},
		{
			name:       "invalid json",
			method:     http.MethodPost,
			body:       `{bad json`,
			wantStatus: http.StatusBadRequest,
		},
		{
			name:       "invalid email",
			method:     http.MethodPost,
			body:       `{"email":"bad"}`,
			serviceErr: ErrInvalidEmail,
			wantStatus: http.StatusBadRequest,
			wantEmail:  "bad",
		},
		{
			name:       "service failed",
			method:     http.MethodPost,
			body:       `{"email":"alice@example.com"}`,
			serviceErr: errors.New("db failed"),
			wantStatus: http.StatusInternalServerError,
			wantEmail:  "alice@example.com",
		},
		{
			name:       "wrong method",
			method:     http.MethodGet,
			body:       `{}`,
			wantStatus: http.StatusMethodNotAllowed,
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			service := &fakeUserService{
				user: User{ID: 1, Email: "alice@example.com"},
				err:  tt.serviceErr,
			}
			handler := NewUserHandler(service)

			req := httptest.NewRequest(tt.method, "/users", bytes.NewBufferString(tt.body))
			rec := httptest.NewRecorder()

			handler.CreateUser(rec, req)

			if rec.Code != tt.wantStatus {
				t.Fatalf("status = %d, want %d, body = %s", rec.Code, tt.wantStatus, rec.Body.String())
			}
			if service.gotEmail != tt.wantEmail {
				t.Fatalf("service email = %q, want %q", service.gotEmail, tt.wantEmail)
			}

			if tt.wantStatus == http.StatusCreated {
				var got User
				if err := json.NewDecoder(rec.Body).Decode(&got); err != nil {
					t.Fatalf("decode response: %v", err)
				}
				if got.ID != 1 || got.Email != "alice@example.com" {
					t.Fatalf("response = %+v", got)
				}
			}
		})
	}
}
```

运行：

```powershell
go test ./...
```

Linux/macOS：

```bash
go test ./...
```

## 五、注意点

handler 测试不应该重复 service 的全部业务测试。它主要检查协议层：请求怎么进来，响应怎么出去，错误如何映射成 HTTP 状态码。

不要在 handler 里直接写数据库逻辑。否则测试 handler 时就被迫处理数据库连接，职责会混在一起。

对于鉴权接口，可以在测试请求里设置 Header：

```go
req.Header.Set("Authorization", "Bearer test-token")
```

如果项目使用中间件，可以用 `http.Handler` 链一起测试，但不要在一个测试里把所有中间件和业务都塞满。关键链路可以测，细节还是分层测。

## 六、常见误区

- 只测试状态码，不测试响应体：字段名写错也可能漏掉。
- handler 测试连真实数据库：测试慢且不稳定。
- JSON 错误返回 `500`：用户输入错误通常应该是 `400`。
- 不检查 service 收到的参数：handler 解析错字段时发现不了。
- 直接比较整段错误响应文本：标准库 `http.Error` 会带换行，过度精确会让测试脆弱。

## 七、本节达标标准

- 能用 `httptest.NewRequest` 构造请求。
- 能用 `httptest.NewRecorder` 捕获响应。
- 能测试状态码、响应体和 service 入参。
- 能说明 handler 测试和 service 测试的职责区别。
- 能为成功、非法 JSON、业务错误、服务错误写测试。

