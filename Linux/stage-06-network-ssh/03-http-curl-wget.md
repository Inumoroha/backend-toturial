# 03：curl 与 wget 测试 HTTP

本节目标：学会用 `curl` 和 `wget` 请求网页、查看响应头、下载文件，并判断 Web 服务是否正常。

本节命令顺序：

```text
1. curl URL
2. curl -I
3. curl -v
4. curl -o
5. curl -L
6. wget URL
7. wget -O
```

请先进入练习目录：

```bash
cd ~/linux-practice/stage-06
```

---

## 1. curl：发送 HTTP 请求

请求网页：

```bash
curl http://example.com
```

请求 HTTPS：

```bash
curl https://example.com
```

如果页面内容很多，可以保存：

```bash
curl https://example.com > output/example.html
```

---

## 2. curl -I：只看响应头

查看响应头：

```bash
curl -I https://example.com
```

重点看：

| 字段 | 含义 |
| --- | --- |
| `HTTP/2 200` | 状态码 |
| `content-type` | 内容类型 |
| `server` | Web 服务器信息 |
| `location` | 重定向目标 |

常见状态码：

| 状态码 | 含义 |
| --- | --- |
| `200` | 成功 |
| `301/302` | 重定向 |
| `403` | 禁止访问 |
| `404` | 不存在 |
| `500` | 服务端错误 |

---

## 3. curl -v：查看详细过程

执行：

```bash
curl -v https://example.com
```

它会显示：

- DNS 解析结果
- 连接目标 IP
- TLS 握手信息
- 请求头
- 响应头

排查 HTTP 问题时很有用。

---

## 4. curl -o：保存到文件

保存网页：

```bash
curl -o output/example.html https://example.com
```

下载文件时指定文件名：

```bash
curl -o downloads/example.html https://example.com
```

---

## 5. curl -L：跟随重定向

有些网址会返回 301 或 302，需要跟随重定向：

```bash
curl -L http://example.com
```

只看最终响应头：

```bash
curl -IL http://example.com
```

---

## 6. wget：下载文件

下载网页：

```bash
wget https://example.com -O downloads/example.html
```

如果不指定 `-O`，`wget` 会使用 URL 中的文件名。

显示响应信息但不保存：

```bash
wget --spider https://example.com
```

---

## 7. curl 和 wget 怎么选

| 场景 | 推荐 |
| --- | --- |
| 测试接口或网页 | `curl` |
| 查看响应头 | `curl -I` |
| 查看详细连接过程 | `curl -v` |
| 简单下载文件 | `wget` |
| 脚本里调用接口 | `curl` |

---

## 8. 本节练习

执行：

```bash
curl -I https://example.com
curl -v https://example.com
curl -o output/example.html https://example.com
curl -IL http://example.com
wget https://example.com -O downloads/example-wget.html
ls -lh output downloads
```

回答：

1. `example.com` 返回的状态码是什么？
2. `curl -I` 和 `curl` 的输出有什么不同？
3. `curl -L` 解决什么问题？
4. `wget -O` 的作用是什么？

---

## 9. 本节小结

必须记住：

```bash
curl https://example.com
curl -I https://example.com
curl -v https://example.com
curl -o file.html https://example.com
curl -L http://example.com
wget https://example.com -O file.html
wget --spider https://example.com
```

完成后进入下一节：`04-ports-and-listening.md`。

