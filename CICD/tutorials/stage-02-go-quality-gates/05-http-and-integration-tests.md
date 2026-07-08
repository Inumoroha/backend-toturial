# 05：HTTP 测试与集成测试

## 1. 本节目标

Go 后端项目不能只测纯函数。

你还需要验证：

- HTTP 请求能否被正确解析。
- handler 是否返回正确状态码。
- JSON 响应是否符合预期。
- service 和 repository 能否协作。
- 数据库依赖是否能正常工作。

这一节先从 HTTP handler 测试开始，再理解集成测试。

## 2. HTTP handler 测试

Go 标准库提供了 `net/http/httptest`。

示例 handler：

```go
package handler

import (
	"encoding/json"
	"net/http"
)

type HealthResponse struct {
	Status string `json:"status"`
}

func Healthz(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	_ = json.NewEncoder(w).Encode(HealthResponse{Status: "ok"})
}
```

测试：

```go
package handler

import (
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"testing"
)

func TestHealthz(t *testing.T) {
	req := httptest.NewRequest(http.MethodGet, "/healthz", nil)
	rec := httptest.NewRecorder()

	Healthz(rec, req)

	if rec.Code != http.StatusOK {
		t.Fatalf("status = %d, want %d", rec.Code, http.StatusOK)
	}

	var body HealthResponse
	if err := json.NewDecoder(rec.Body).Decode(&body); err != nil {
		t.Fatalf("decode response: %v", err)
	}

	if body.Status != "ok" {
		t.Fatalf("body.status = %q, want %q", body.Status, "ok")
	}
}
```

运行：

```bash
go test ./internal/handler
```

## 3. 测试 JSON 请求

示例：

```go
reqBody := strings.NewReader(`{"title":"buy milk"}`)
req := httptest.NewRequest(http.MethodPost, "/todos", reqBody)
req.Header.Set("Content-Type", "application/json")
```

你可以测试：

- 请求 JSON 正确时返回 201。
- 缺少必填字段时返回 400。
- service 返回错误时转换为正确 HTTP 状态码。

## 4. handler 测试不要太重

handler 测试重点是 HTTP 层：

- method。
- path。
- query。
- header。
- body。
- status code。
- response body。

不要把所有数据库逻辑塞进 handler 测试。

更好的做法：

```text
handler 使用 interface 依赖 service。
测试时传 fake service。
```

## 5. Fake 依赖示例

```go
type fakeTodoService struct {
	err error
}

func (f fakeTodoService) Create(title string) error {
	return f.err
}
```

handler 接收 interface：

```go
type TodoService interface {
	Create(title string) error
}

type TodoHandler struct {
	service TodoService
}
```

这样 handler 测试不需要真实数据库。

## 6. 什么是集成测试

单元测试关注一个函数或一个模块。

集成测试关注多个模块组合后是否正常工作。

例如：

```text
handler -> service -> repository -> PostgreSQL
```

集成测试能发现单元测试发现不了的问题：

- SQL 写错。
- migration 缺失。
- 数据库字段类型不匹配。
- 事务行为不符合预期。
- 配置连接字符串错误。

## 7. 集成测试的依赖管理

常见方式：

### 方式一：本地 Docker Compose

适合初学。

```bash
docker compose up -d postgres
go test -tags=integration ./...
docker compose down
```

### 方式二：testcontainers-go

测试代码启动临时容器。

优点是自动化更好，缺点是学习成本更高。

### 方式三：CI service container

CI 平台在 job 中启动数据库服务。

适合第 3 阶段再学。

## 8. 使用 build tags 区分集成测试

集成测试可能比较慢，且需要外部依赖。

可以用 build tag 标记。

文件名：

```text
todo_repository_integration_test.go
```

文件开头：

```go
//go:build integration

package repository
```

运行普通测试：

```bash
go test ./...
```

运行集成测试：

```bash
go test -tags=integration ./...
```

## 9. 测试数据隔离

集成测试要避免互相影响。

常见策略：

- 每个测试使用独立数据库。
- 每个测试使用事务，结束后 rollback。
- 每个测试清理自己创建的数据。
- 测试表使用随机前缀或唯一 ID。

初学阶段可以先使用：

```text
每个测试前清空相关表。
```

但要记住，这种方式不适合并行测试。

## 10. 健康检查测试

你的 Go 服务应该有：

```text
GET /healthz
GET /readyz
```

区别：

- `/healthz`：进程活着即可。
- `/readyz`：依赖也准备好了，例如数据库能连接。

HTTP 测试可以先验证 `/healthz`。

集成测试可以验证 `/readyz`。

## 11. 小练习

为 `go-cicd-lab` 完成：

1. 添加 `GET /healthz` handler。
2. 使用 `httptest` 编写 handler 测试。
3. 添加 `GET /readyz` handler。
4. 暂时用 fake dependency 测试 ready 和 not ready 两种情况。
5. 执行：

```bash
go test ./...
```

进阶：

```bash
go test -tags=integration ./...
```

## 12. 本节小结

你现在应该理解：

- `httptest` 可以测试 HTTP handler。
- handler 测试重点是 HTTP 行为，不要强依赖数据库。
- 集成测试用于验证多个模块和外部依赖协作。
- 慢测试和依赖外部资源的测试可以用 build tags 区分。
- `/healthz` 和 `/readyz` 是后续部署验证的基础。

