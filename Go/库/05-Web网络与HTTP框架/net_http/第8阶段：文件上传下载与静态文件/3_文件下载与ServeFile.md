# 3. 文件下载与 ServeFile

本节目标：学会用 `http.ServeFile` 提供文件下载，并设置合适的响应 Header。

---

## 一、最小下载接口

```go
func download(w http.ResponseWriter, r *http.Request) {
	name := r.PathValue("name")
	path := filepath.Join("./uploads", filepath.Base(name))

	http.ServeFile(w, r, path)
}
```

注册：

```go
mux.HandleFunc("GET /files/{name}", download)
```

访问：

```bash
curl -i http://localhost:8080/files/avatar.png
```

---

## 二、设置下载文件名

如果希望浏览器下载而不是直接预览：

```go
w.Header().Set("Content-Disposition", `attachment; filename="avatar.png"`)
http.ServeFile(w, r, path)
```

`Content-Disposition` 常用于控制浏览器如何处理响应文件。

---

## 三、防止路径穿越

不要这样写：

```go
path := "./uploads/" + r.PathValue("name")
```

攻击者可能请求：

```text
/files/../../secret.txt
```

至少要：

```go
name := filepath.Base(r.PathValue("name"))
path := filepath.Join("./uploads", name)
```

生产项目中还要根据存储记录判断用户是否有权限访问该文件。

---

## 四、文件不存在时

`http.ServeFile` 会处理不存在文件，通常返回 `404`。

如果你想自定义 JSON 错误响应，就需要自己先检查：

```go
if _, err := os.Stat(path); err != nil {
	if os.IsNotExist(err) {
		writeError(w, http.StatusNotFound, "not_found", "file not found")
		return
	}
	writeError(w, http.StatusInternalServerError, "internal_error", "internal server error")
	return
}
```

---

## 五、本节检查点

请确认你能做到：

- 使用 `http.ServeFile` 返回文件。
- 设置 `Content-Disposition`。
- 使用 `filepath.Base` 和 `filepath.Join` 处理路径。
- 理解文件下载也需要权限和路径校验。

