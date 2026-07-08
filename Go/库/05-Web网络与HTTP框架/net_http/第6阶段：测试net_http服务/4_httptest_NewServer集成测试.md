# 4. httptest.NewServer 集成测试

本节目标：学会启动一个测试用 HTTP 服务，并用真实 `http.Client` 调用它。

---

## 一、什么时候用 NewServer

前面测试 Handler 是直接调用：

```go
handler.ServeHTTP(rec, req)
```

这很快，也适合单元测试。

但有些场景更适合 `httptest.NewServer`：

- 测试 HTTP Client。
- 测试真实网络请求行为。
- 测试完整路由和中间件链。
- 模拟第三方 API。

---

## 二、最小示例

```go
func TestWithServer(t *testing.T) {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "ok")
	})

	server := httptest.NewServer(mux)
	defer server.Close()

	resp, err := http.Get(server.URL + "/health")
	if err != nil {
		t.Fatalf("get health: %v", err)
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		t.Fatalf("status = %d, want %d", resp.StatusCode, http.StatusOK)
	}
}
```

`server.URL` 会是一个自动分配端口的地址，例如：

```text
http://127.0.0.1:52341
```

---

## 三、测试完整 Todo 路由

```go
func TestTodoFlow(t *testing.T) {
	store := newTodoStore()
	mux := http.NewServeMux()
	mux.HandleFunc("POST /todos", store.createTodo)
	mux.HandleFunc("GET /todos/{id}", store.getTodo)

	server := httptest.NewServer(mux)
	defer server.Close()

	body := strings.NewReader(`{"title":"learn"}`)
	resp, err := http.Post(server.URL+"/todos", "application/json", body)
	if err != nil {
		t.Fatalf("create todo: %v", err)
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusCreated {
		t.Fatalf("create status = %d, want %d", resp.StatusCode, http.StatusCreated)
	}

	resp, err = http.Get(server.URL + "/todos/1")
	if err != nil {
		t.Fatalf("get todo: %v", err)
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		t.Fatalf("get status = %d, want %d", resp.StatusCode, http.StatusOK)
	}
}
```

---

## 四、为什么 defer resp.Body.Close 很重要

客户端收到响应后，必须关闭响应体：

```go
defer resp.Body.Close()
```

否则连接资源可能无法及时释放，连接复用也会受影响。

这是写 HTTP Client 时最常见的坑之一。

---

## 五、本节检查点

请确认你能做到：

- 使用 `httptest.NewServer` 启动测试服务。
- 使用 `server.URL` 构造请求地址。
- 用 `http.Client` 或 `http.Get` 调用测试服务。
- 正确关闭响应体。
- 测试一个完整接口流程。

