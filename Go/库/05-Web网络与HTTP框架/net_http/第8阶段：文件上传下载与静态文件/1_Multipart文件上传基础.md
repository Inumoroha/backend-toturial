# 1. Multipart 文件上传基础

本节目标：学会用 `net/http` 接收浏览器或客户端上传的文件。

---

## 一、文件上传使用什么格式

常见文件上传使用：

```text
multipart/form-data
```

curl 示例：

```bash
curl -i -X POST http://localhost:8080/upload `
  -F "file=@./avatar.png"
```

`-F` 会让 curl 使用 multipart 表单上传。

---

## 二、最小上传 Handler

```go
func upload(w http.ResponseWriter, r *http.Request) {
	if err := r.ParseMultipartForm(10 << 20); err != nil {
		http.Error(w, "invalid multipart form", http.StatusBadRequest)
		return
	}

	file, header, err := r.FormFile("file")
	if err != nil {
		http.Error(w, "missing file", http.StatusBadRequest)
		return
	}
	defer file.Close()

	fmt.Fprintf(w, "filename=%s size=%d\n", header.Filename, header.Size)
}
```

注册：

```go
mux.HandleFunc("POST /upload", upload)
```

---

## 三、ParseMultipartForm 参数是什么意思

```go
r.ParseMultipartForm(10 << 20)
```

这里的 `10 << 20` 是 10MB。

它表示表单解析时最多把多少内容放在内存中，超过部分可能写入临时文件。它不是完整的上传大小限制。

完整限制应该结合：

```go
r.Body = http.MaxBytesReader(w, r.Body, maxUploadSize)
```

下一节会详细讲。

---

## 四、保存上传文件

```go
dst, err := os.Create("./uploads/" + header.Filename)
if err != nil {
	http.Error(w, "create file failed", http.StatusInternalServerError)
	return
}
defer dst.Close()

if _, err := io.Copy(dst, file); err != nil {
	http.Error(w, "save file failed", http.StatusInternalServerError)
	return
}
```

这个写法只是入门示例，真实项目不能直接信任 `header.Filename`，否则可能有路径穿越风险。

---

## 五、本节检查点

请确认你能做到：

- 用 curl `-F` 上传文件。
- 使用 `r.ParseMultipartForm`。
- 使用 `r.FormFile("file")` 获取文件。
- 读取上传文件名和大小。
- 把文件保存到本地目录。

