# 2. Handler 单元测试

本节目标：为 JSON Handler 编写单元测试，覆盖成功路径和错误路径。

---

## 一、测试什么

Handler 测试重点不是测试标准库，而是测试你的 HTTP 行为：

- 状态码是否正确。
- 参数错误是否返回 `400`。
- 资源不存在是否返回 `404`。
- 创建成功是否返回 `201`。
- 响应 JSON 是否符合预期。

---

## 二、表格驱动测试

Go 测试常用表格驱动：

```go
func TestCreateTodo(t *testing.T) {
	tests := []struct {
		name       string
		body       string
		wantStatus int
	}{
		{
			name:       "created",
			body:       `{"title":"learn"}`,
			wantStatus: http.StatusCreated,
		},
		{
			name:       "missing title",
			body:       `{}`,
			wantStatus: http.StatusBadRequest,
		},
		{
			name:       "invalid json",
			body:       `{`,
			wantStatus: http.StatusBadRequest,
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			// test body
		})
	}
}
```

好处是：

- 多个场景结构统一。
- 增加测试用例很方便。
- 失败时能看到具体 case 名称。

---

## 三、测试创建 Todo

假设你有：

```go
store := newTodoStore()
```

测试：

```go
func TestCreateTodo(t *testing.T) {
	store := newTodoStore()

	req := httptest.NewRequest(http.MethodPost, "/todos", strings.NewReader(`{"title":"learn"}`))
	req.Header.Set("Content-Type", "application/json")
	rec := httptest.NewRecorder()

	store.createTodo(rec, req)

	resp := rec.Result()
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusCreated {
		t.Fatalf("status = %d, want %d", resp.StatusCode, http.StatusCreated)
	}

	var got Todo
	if err := json.NewDecoder(resp.Body).Decode(&got); err != nil {
		t.Fatalf("decode response: %v", err)
	}

	if got.ID == 0 {
		t.Fatal("id should be set")
	}
	if got.Title != "learn" {
		t.Fatalf("title = %q, want %q", got.Title, "learn")
	}
}
```

---

## 四、测试错误路径

```go
func TestCreateTodoInvalidJSON(t *testing.T) {
	store := newTodoStore()

	req := httptest.NewRequest(http.MethodPost, "/todos", strings.NewReader(`{`))
	req.Header.Set("Content-Type", "application/json")
	rec := httptest.NewRecorder()

	store.createTodo(rec, req)

	resp := rec.Result()
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusBadRequest {
		t.Fatalf("status = %d, want %d", resp.StatusCode, http.StatusBadRequest)
	}
}
```

错误路径不要省。真实项目中，很多 bug 都出现在错误分支。

---

## 五、测试路径参数

如果 Handler 使用：

```go
r.PathValue("id")
```

直接构造 Request 时不会自动设置路径参数。更简单的方式是通过 mux 测：

```go
mux := http.NewServeMux()
mux.HandleFunc("GET /todos/{id}", store.getTodo)

req := httptest.NewRequest(http.MethodGet, "/todos/1", nil)
rec := httptest.NewRecorder()

mux.ServeHTTP(rec, req)
```

这样 `ServeMux` 会负责匹配路径并设置 PathValue。

---

## 六、本节检查点

请确认你能做到：

- 为成功路径写 Handler 测试。
- 为错误 JSON 写测试。
- 为参数缺失写测试。
- 使用 mux 测试路径参数。
- 用表格驱动组织多个 case。

