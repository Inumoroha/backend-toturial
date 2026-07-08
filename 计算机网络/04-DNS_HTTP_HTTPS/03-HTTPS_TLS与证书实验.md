# 03-HTTPS、TLS 与证书实验

## 学习目标

理解 HTTPS 不是一个新协议，而是 HTTP + TLS。掌握 TLS 解决的安全问题，以及证书、CA、SNI 的基本概念。

## 一、HTTP 的问题

HTTP 明文传输，存在三个主要问题：

- 窃听：中间人能看到内容。
- 篡改：中间人能修改内容。
- 冒充：你不知道对方是否真是目标服务器。

HTTPS 使用 TLS 解决这些问题。

## 二、TLS 提供什么

TLS 提供：

- 加密：防止内容被窃听。
- 完整性校验：防止内容被篡改。
- 身份认证：通过证书确认服务器身份。

## 三、对称加密与非对称加密

对称加密：

- 加密和解密使用同一把密钥。
- 速度快。
- 问题是密钥如何安全分发。

非对称加密：

- 公钥加密，私钥解密，或私钥签名，公钥验证。
- 适合身份认证和密钥交换。
- 计算成本较高。

TLS 会结合两者：握手阶段协商密钥，数据传输阶段主要使用对称加密。

## 四、证书与 CA

证书用于证明“这个公钥属于这个域名”。

CA 是受信任的证书颁发机构。浏览器和操作系统内置了一批根 CA。

验证证书时会检查：

- 证书是否过期。
- 域名是否匹配。
- 证书链是否可信。
- 证书是否被吊销。

## 五、SNI

SNI 是 TLS 握手中的扩展字段，用来告诉服务器客户端想访问哪个域名。

为什么需要 SNI？

同一个 IP 上可能部署多个 HTTPS 域名，服务器需要根据域名选择正确证书。

## 六、实验：curl 查看 TLS

```bash
curl -v https://example.com
```

关注输出：

- TLS 版本。
- 证书 subject。
- 证书 issuer。
- ALPN 协商结果。
- HTTP 状态码。

## 七、实验：openssl 查看证书

```bash
openssl s_client -connect example.com:443 -servername example.com
```

关注：

- 证书链。
- 证书有效期。
- 验证结果。

## 八、Go HTTPS Server 示例

生成自签名证书：

```bash
openssl req -x509 -newkey rsa:2048 -nodes \
  -keyout server.key \
  -out server.crt \
  -days 365 \
  -subj "/CN=localhost"
```

Go 服务：

```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "hello https")
    })

    log.Println("listen on https://localhost:8443")
    log.Fatal(http.ListenAndServeTLS(":8443", "server.crt", "server.key", nil))
}
```

访问：

```bash
curl -k -v https://localhost:8443
```

`-k` 表示跳过证书验证。生产环境不要这样做。

## 九、Go 后端关联

Go 客户端中 TLS 常见问题：

- 证书过期。
- 域名与证书不匹配。
- 缺少根证书。
- 代理或网关替换证书。
- TLS 握手超时。

自定义 Transport 时可以配置：

```go
Transport: &http.Transport{
    TLSHandshakeTimeout: 3 * time.Second,
}
```

不要为了“临时解决问题”在生产环境设置：

```go
InsecureSkipVerify: true
```

这会跳过证书验证，破坏 HTTPS 的身份认证。

## 十、练习题

1. HTTPS 解决 HTTP 的哪些安全问题？
2. TLS 为什么不全程只用非对称加密？
3. 证书链验证大致检查哪些内容？
4. SNI 解决什么问题？

## 十一、验收标准

你能用 `curl -v` 和 `openssl s_client` 查看 HTTPS 连接信息，并能解释证书、CA、TLS 握手的作用。

