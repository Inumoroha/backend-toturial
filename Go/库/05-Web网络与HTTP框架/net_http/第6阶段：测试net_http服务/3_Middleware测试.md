# 3. Middleware 测试

本节目标：学会测试中间件。中间件是 Handler 的包装，所以测试方式仍然是构造请求、执行 Handler、检查响应。

---

## 一、测试 auth 中间件

中间件：

```go
func auth(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if r.Header.Get("Authorization") != "Bearer dev-token" {
			writeError(w, http.StatusUnauthorized, "unauthorized", "missing or invalid token")
			return
		}

		next.ServeHTTP(w, r)
	})
}
```

测试未授权：

```go
func TestAuthUnauthorized(t *testing.T) {
	nextCalled := false
	next := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		nextCalled = true
	})

	req := httptest.NewRequest(http.MethodGet, "/private", nil)
	rec := httptest.NewRecorder()

	auth(next).ServeHTTP(rec, req)

	resp := rec.Result()
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusUnauthorized {
		t.Fatalf("status = %d, want %d", resp.StatusCode, http.StatusUnauthorized)
	}
	if nextCalled {
		t.Fatal("next should not be called")
	}
}
```

关键点：鉴权失败时，必须确认 `next` 没有被调用。

---

## 二、测试授权成功

```go
func TestAuthOK(t *testing.T) {
	nextCalled := false
	next := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		nextCalled = true
		w.WriteHeader(http.StatusNoContent)
	})

	req := httptest.NewRequest(http.MethodGet, "/private", nil)
	req.Header.Set("Authorization", "Bearer dev-token")
	rec := httptest.NewRecorder()

	auth(next).ServeHTTP(rec, req)

	resp := rec.Result()
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusNoContent {
		t.Fatalf("status = %d, want %d", resp.StatusCode, http.StatusNoContent)
	}
	if !nextCalled {
		t.Fatal("next should be called")
	}
}
```

---

## 三、测试 Request ID 中间件

目标：

- 如果请求没有 `X-Request-ID`，中间件生成一个。
- 响应 Header 中有 `X-Request-ID`。
- 下游 Handler 能从 context 读到它。

测试思路：

```go
func TestRequestID(t *testing.T) {
	var gotID string
	next := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		gotID = getRequestID(r)
	})

	req := httptest.NewRequest(http.MethodGet, "/", nil)
	rec := httptest.NewRecorder()

	requestID(next).ServeHTTP(rec, req)

	resp := rec.Result()
	defer resp.Body.Close()

	headerID := resp.Header.Get("X-Request-ID")
	if headerID == "" {
		t.Fatal("missing X-Request-ID header")
	}
	if gotID == "" {
		t.Fatal("missing request id in context")
	}
}
```

---

## 四、测试 recovery

```go
func TestRecovery(t *testing.T) {
	next := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		panic("boom")
	})

	req := httptest.NewRequest(http.MethodGet, "/panic", nil)
	rec := httptest.NewRecorder()

	recovery(next).ServeHTTP(rec, req)

	resp := rec.Result()
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusInternalServerError {
		t.Fatalf("status = %d, want %d", resp.StatusCode, http.StatusInternalServerError)
	}
}
```

---

## 五、本节检查点

请确认你能做到：

- 测试中间件是否调用 next。
- 测试中间件是否提前返回。
- 测试中间件写入的 Header。
- 测试 panic 是否被 recovery 捕获。

