# 8. 实现 HTTP 路由和响应

本节目标：实现短链接服务的 HTTP 层，包括 JSON API 和跳转接口。

---

## 一、Handler 结构

文件：`internal/httpapi/handler.go`

```go
type Handler struct {
	service *link.Service
	baseURL string
}

func NewHandler(service *link.Service, baseURL string) *Handler {
	return &Handler{service: service, baseURL: strings.TrimRight(baseURL, "/")}
}
```

`baseURL` 用来生成 `short_url`。生产环境不要临时拼 `r.Host`，应该来自配置。

---

## 二、路由注册

```go
func (h *Handler) Routes() http.Handler {
	mux := http.NewServeMux()
	mux.HandleFunc("POST /links", h.createLink)
	mux.HandleFunc("GET /links/{code}", h.getLink)
	mux.HandleFunc("GET /links", h.listLinks)
	mux.HandleFunc("GET /{code}", h.redirect)
	return mux
}
```

注意顺序：`ServeMux` 会按规则匹配，但设计上仍要避免让 `/{code}` 这种宽泛路由和管理接口混淆。真实项目中也可以把跳转域名和 API 域名分开。

---

## 三、响应结构

```go
type linkResponse struct {
	Code        string `json:"code"`
	OriginalURL string `json:"original_url"`
	ShortURL    string `json:"short_url"`
	VisitCount  int64  `json:"visit_count"`
}

func (h *Handler) toResponse(item link.Link) linkResponse {
	return linkResponse{
		Code:        item.Code,
		OriginalURL: item.OriginalURL,
		ShortURL:    h.baseURL + "/" + item.Code,
		VisitCount:  item.VisitCount,
	}
}
```

---

## 四、创建接口

```go
type createLinkRequest struct {
	OriginalURL string `json:"original_url"`
}

func (h *Handler) createLink(w http.ResponseWriter, r *http.Request) {
	r.Body = http.MaxBytesReader(w, r.Body, 1<<20)

	var req createLinkRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		writeError(w, http.StatusBadRequest, "invalid_json", "invalid json body")
		return
	}

	item, err := h.service.Create(r.Context(), req.OriginalURL)
	if err != nil {
		h.writeLinkError(w, err)
		return
	}

	writeJSON(w, http.StatusCreated, h.toResponse(item))
}
```

---

## 五、跳转接口

```go
func (h *Handler) redirect(w http.ResponseWriter, r *http.Request) {
	code := r.PathValue("code")

	item, err := h.service.GetByCode(r.Context(), code)
	if err != nil {
		h.writeLinkError(w, err)
		return
	}

	if err := h.service.IncrementVisit(r.Context(), code); err != nil {
		h.writeLinkError(w, err)
		return
	}

	http.Redirect(w, r, item.OriginalURL, http.StatusFound)
}
```

学习阶段先“计数成功再跳转”。真实高流量短链接服务可以把访问日志和计数异步化。

---

## 六、错误映射

```go
func (h *Handler) writeLinkError(w http.ResponseWriter, err error) {
	switch {
	case errors.Is(err, link.ErrInvalidURL):
		writeError(w, http.StatusBadRequest, "invalid_url", "invalid original_url")
	case errors.Is(err, link.ErrNotFound):
		writeError(w, http.StatusNotFound, "not_found", "link not found")
	default:
		writeError(w, http.StatusInternalServerError, "internal_error", "internal server error")
	}
}
```

---

## 七、本节检查点

你应该能做到：

- 创建短链接返回 `201` JSON。
- 访问短码返回 `302`。
- 详情和列表返回 JSON。
- URL 错误返回 `400`。
- 不存在短码返回 `404`。
