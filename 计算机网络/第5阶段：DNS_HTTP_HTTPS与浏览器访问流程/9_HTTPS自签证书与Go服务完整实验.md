# 9. HTTPS 自签证书与 Go 服务完整实验

本节目标：从 0 开始生成自签证书，启动 Go HTTPS 服务，用 curl 和 openssl 观察 TLS 握手与证书信息。学完后你会更清楚 HTTPS 不是“神秘加密”，而是可以被拆成证书、TLS 握手和 HTTP 请求三个部分观察。

---

## 一、为什么要做这个实验

面试中经常问：

```text
HTTPS 和 HTTP 有什么区别？
TLS 握手大概做了什么？
证书有什么用？
为什么自签证书浏览器不信任？
```

如果只背答案，很容易讲得空。自己生成证书并启动 HTTPS 服务后，你会直观看到：

- 证书里有域名。
- 客户端会验证证书。
- 自签证书默认不被信任。
- `curl -k` 可以跳过验证，但生产不能这么做。

---

## 二、创建项目

```bash
mkdir https-lab
cd https-lab
go mod init https-lab
```

目录最终是：

```text
https-lab/
  go.mod
  main.go
  certs/
    server.crt
    server.key
```

创建证书目录：

```bash
mkdir certs
```

---

## 三、生成自签证书

执行：

```bash
openssl req -x509 -newkey rsa:2048 -nodes \
  -keyout certs/server.key \
  -out certs/server.crt \
  -days 365 \
  -subj "/CN=localhost"
```

参数解释：

- `req -x509`：生成自签名证书。
- `-newkey rsa:2048`：生成 2048 位 RSA 密钥。
- `-nodes`：私钥不加密，便于本地实验。
- `-keyout`：私钥输出路径。
- `-out`：证书输出路径。
- `-days 365`：有效期 365 天。
- `-subj "/CN=localhost"`：证书主体的 Common Name 是 localhost。

查看文件：

```bash
ls -l certs
```

你应该看到：

```text
server.crt
server.key
```

注意：`server.key` 是私钥，生产环境必须保护好，不能提交到公开仓库。

---

## 四、编写 Go HTTPS 服务

创建 `main.go`：

```go
package main

import (
    "encoding/json"
    "log"
    "net/http"
    "time"
)

type Response struct {
    Message string `json:"message"`
    Proto   string `json:"proto"`
}

func main() {
    mux := http.NewServeMux()

    mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusNoContent)
    })

    mux.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(Response{
            Message: "hello https",
            Proto:   r.Proto,
        })
    })

    srv := &http.Server{
        Addr:              ":8443",
        Handler:           mux,
        ReadHeaderTimeout: 2 * time.Second,
        ReadTimeout:       5 * time.Second,
        WriteTimeout:      10 * time.Second,
        IdleTimeout:       60 * time.Second,
    }

    log.Println("listen on https://localhost:8443")
    log.Fatal(srv.ListenAndServeTLS("certs/server.crt", "certs/server.key"))
}
```

启动：

```bash
go run .
```

---

## 五、第一次访问：证书不被信任

另一个终端执行：

```bash
curl -v https://localhost:8443/hello
```

你大概率会看到类似错误：

```text
SSL certificate problem: self-signed certificate
```

这说明 TLS 连接中客户端验证证书失败。

为什么失败？

因为这是你自己签发的证书，不在操作系统或 curl 信任的 CA 列表中。

---

## 六、跳过验证访问

本地实验可以使用：

```bash
curl -k -v https://localhost:8443/hello
```

`-k` 表示跳过证书验证。

你应该看到响应：

```json
{"message":"hello https","proto":"HTTP/1.1"}
```

注意：

```text
生产环境不要使用 -k 或 InsecureSkipVerify。
```

跳过证书验证会让 HTTPS 失去身份认证能力，容易受到中间人攻击。

---

## 七、使用 openssl 查看证书

执行：

```bash
openssl s_client -connect localhost:8443 -servername localhost
```

关注：

```text
Certificate chain
subject=CN=localhost
issuer=CN=localhost
Verify return code
```

因为是自签证书，subject 和 issuer 可能都是 localhost。

如果看到 verify error，也符合预期，因为系统不信任这个自签 CA。

---

## 八、查看证书内容

```bash
openssl x509 -in certs/server.crt -text -noout
```

关注：

- Subject。
- Issuer。
- Validity。
- Public Key。
- Signature Algorithm。

证书本质上是把公钥、域名、有效期、签发者等信息放在一起，并由签发者签名。

---

## 九、抓包观察 HTTPS

抓包：

```bash
sudo tcpdump -i any tcp port 8443 -w https-8443.pcap
```

请求：

```bash
curl -k https://localhost:8443/hello
```

用 Wireshark 打开 `https-8443.pcap`。

你能看到：

- TCP 三次握手。
- TLS ClientHello。
- TLS ServerHello。
- TLS 加密应用数据。

你看不到明文 HTTP Header 和 Body，因为它们被 TLS 加密了。

---

## 十、常见问题

### 1. 自签证书为什么不被信任？

因为它不是由系统信任的 CA 签发。客户端无法确认这个证书真的属于目标服务。

### 2. `curl -k` 是否安全？

不安全。它跳过证书验证，只适合本地临时实验。

### 3. HTTPS 是否隐藏 IP 和端口？

不隐藏。IP 和端口属于网络层和传输层，仍然可见。

### 4. SNI 在这个实验中有什么用？

`-servername localhost` 会在 TLS 握手中发送 SNI，让服务器知道客户端访问的域名。多域名共用同一 IP 时很重要。

---

## 十一、本节达标标准

学完本节后，你应该能够做到：

- 使用 openssl 生成自签证书。
- 使用 Go 启动 HTTPS 服务。
- 解释为什么自签证书默认不被信任。
- 使用 `curl -k -v` 观察 HTTPS 请求。
- 使用 `openssl s_client` 查看证书链和验证结果。
- 用抓包解释 HTTPS 能看到 TLS 握手但看不到 HTTP 明文。

