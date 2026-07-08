# 5. 阶段练习：GitHub 用户查询客户端

本节目标：封装一个可靠的 HTTP Client，用来查询 GitHub 用户信息，并用 `httptest.NewServer` 测试它。

---

## 一、需求

实现：

```go
type GitHubClient struct {
	baseURL string
	client  *http.Client
}

func (c *GitHubClient) GetUser(ctx context.Context, username string) (*GitHubUser, error)
```

返回结构：

```go
type GitHubUser struct {
	Login string `json:"login"`
	ID    int64  `json:"id"`
}
```

要求：

- 使用 `http.NewRequestWithContext`。
- 设置 `Accept: application/json`。
- 设置 `User-Agent`。
- 检查非 2xx 状态码。
- 正确关闭响应体。
- 支持测试时替换 baseURL。

---

## 二、参考实现

```go
type GitHubClient struct {
	baseURL string
	client  *http.Client
}

type GitHubUser struct {
	Login string `json:"login"`
	ID    int64  `json:"id"`
}

func NewGitHubClient(baseURL string) *GitHubClient {
	return &GitHubClient{
		baseURL: strings.TrimRight(baseURL, "/"),
		client: &http.Client{
			Timeout: 5 * time.Second,
		},
	}
}

func (c *GitHubClient) GetUser(ctx context.Context, username string) (*GitHubUser, error) {
	if strings.TrimSpace(username) == "" {
		return nil, fmt.Errorf("username is required")
	}

	url := c.baseURL + "/users/" + url.PathEscape(username)
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
	if err != nil {
		return nil, err
	}

	req.Header.Set("Accept", "application/json")
	req.Header.Set("User-Agent", "net-http-learning")

	resp, err := c.client.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	if resp.StatusCode < 200 || resp.StatusCode >= 300 {
		body, _ := io.ReadAll(io.LimitReader(resp.Body, 4<<10))
		return nil, fmt.Errorf("github status %d: %s", resp.StatusCode, string(body))
	}

	var user GitHubUser
	if err := json.NewDecoder(resp.Body).Decode(&user); err != nil {
		return nil, err
	}

	return &user, nil
}
```

---

## 三、测试成功场景

```go
func TestGitHubClientGetUser(t *testing.T) {
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if r.URL.Path != "/users/alice" {
			t.Fatalf("path = %s", r.URL.Path)
		}
		if r.Header.Get("User-Agent") == "" {
			t.Fatal("missing user agent")
		}

		w.Header().Set("Content-Type", "application/json")
		fmt.Fprintln(w, `{"login":"alice","id":1}`)
	}))
	defer server.Close()

	client := NewGitHubClient(server.URL)
	user, err := client.GetUser(context.Background(), "alice")
	if err != nil {
		t.Fatalf("GetUser error: %v", err)
	}

	if user.Login != "alice" || user.ID != 1 {
		t.Fatalf("unexpected user: %+v", user)
	}
}
```

---

## 四、测试 404

```go
func TestGitHubClientGetUserNotFound(t *testing.T) {
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		http.Error(w, "not found", http.StatusNotFound)
	}))
	defer server.Close()

	client := NewGitHubClient(server.URL)
	_, err := client.GetUser(context.Background(), "missing")
	if err == nil {
		t.Fatal("expected error")
	}
}
```

---

## 五、阶段复盘

完成本阶段后，请确认你能解释：

- 为什么 Client 要复用？
- 为什么请求要带 context？
- 为什么非 2xx 要单独处理？
- 为什么测试 Client 适合用 `httptest.NewServer`？
- 为什么响应体必须关闭？

下一阶段进入文件上传、下载和静态文件服务。

