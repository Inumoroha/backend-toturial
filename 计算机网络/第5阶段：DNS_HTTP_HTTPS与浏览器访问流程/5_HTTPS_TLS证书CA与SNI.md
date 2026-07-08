# 5. HTTPS、TLS、证书、CA 与 SNI

本节目标：理解 HTTPS 如何提供加密、防篡改和身份认证，掌握证书、CA、证书链、SNI 的基础。

---

## 一、HTTP 的问题

HTTP 明文传输，存在：

- 窃听。
- 篡改。
- 冒充。

HTTPS = HTTP + TLS。

TLS 提供：

- 加密。
- 完整性校验。
- 身份认证。

---

## 二、对称加密与非对称加密

对称加密：

- 同一把密钥加密解密。
- 速度快。
- 密钥分发困难。

非对称加密：

- 公钥和私钥配对。
- 适合身份认证和密钥交换。
- 计算成本更高。

TLS 握手阶段协商密钥，数据传输阶段主要使用对称加密。

---

## 三、证书和 CA

证书证明：

```text
这个公钥属于这个域名。
```

CA 是受信任的证书颁发机构。

浏览器验证：

- 证书是否过期。
- 域名是否匹配。
- 证书链是否可信。
- 证书是否被吊销。

---

## 四、SNI

SNI 让客户端在 TLS 握手时告诉服务器自己要访问哪个域名。

为什么需要？

```text
同一个 IP 上可能部署多个 HTTPS 域名。
服务器需要根据域名选择证书。
```

---

## 五、curl 观察 TLS

```bash
curl -v https://example.com
```

关注：

- TLS 版本。
- 证书 subject。
- 证书 issuer。
- ALPN。
- HTTP 状态码。

查看证书：

```bash
openssl s_client -connect example.com:443 -servername example.com
```

---

## 六、Go 后端注意点

常见 TLS 问题：

- 证书过期。
- 域名不匹配。
- 缺少根证书。
- TLS 握手超时。
- 代理替换证书。

不要在生产中随意设置：

```go
InsecureSkipVerify: true
```

这会跳过证书验证，破坏 HTTPS 身份认证。

---

## 补充实验：用 openssl 看证书链和 SNI

先不带 SNI 访问：

```bash
openssl s_client -connect example.com:443
```

再带 SNI 访问：

```bash
openssl s_client -connect example.com:443 -servername example.com
```

你要关注：

```text
Certificate chain
subject
issuer
Verify return code
```

如果一个 IP 上配置了多个 HTTPS 站点，不带 `-servername` 可能拿到默认证书，而不是目标域名证书。

典型判断：

```text
Verify return code: 0 (ok)       证书链验证成功。
self-signed certificate          自签证书不被信任。
certificate has expired          证书过期。
hostname mismatch                证书域名不匹配。
```

证书排障时至少记录：

```text
访问域名：
连接 IP：
SNI：
证书 subject：
证书 issuer：
过期时间：
验证结果：
```

---

## 补充代码：Go Client 如何加载自定义 CA

内部系统可能使用公司自签 CA。不要用 `InsecureSkipVerify`，而应该把 CA 加入信任池：

```go
caPEM, err := os.ReadFile("company-ca.pem")
if err != nil {
    return err
}

pool := x509.NewCertPool()
if !pool.AppendCertsFromPEM(caPEM) {
    return errors.New("append ca failed")
}

client := &http.Client{
    Transport: &http.Transport{
        TLSClientConfig: &tls.Config{
            RootCAs: pool,
        },
    },
}
```

这段代码表达的是：

```text
我仍然验证证书。
只是额外信任公司自己的 CA。
```

这比跳过验证安全得多。

---

## 补充排障：Go 程序和浏览器证书结果不同

有时浏览器能打开网站，但 Go 程序请求失败：

```text
x509: certificate signed by unknown authority
```

常见原因：

```text
浏览器使用自己的证书信任库。
操作系统证书库没有公司 CA。
容器镜像太精简，缺少 ca-certificates。
代理软件替换了证书。
服务端漏发中间证书。
```

容器内优先检查：

```bash
cat /etc/os-release
ls /etc/ssl/certs | head
curl -v https://目标域名
```

Alpine 镜像常见修复：

```dockerfile
RUN apk add --no-cache ca-certificates
```

Debian/Ubuntu 镜像常见修复：

```dockerfile
RUN apt-get update && apt-get install -y ca-certificates
```

证书问题要修信任链，不要用跳过验证掩盖。

---

## 七、常见问题

### 1. HTTPS 是否只加密 body？

不是。HTTP 请求内容在 TLS 内部传输，Header 和 Body 都会被加密。但 IP、端口、部分握手信息仍可见。

### 2. 证书过期会怎样？

客户端验证失败，请求通常会被拒绝。

### 3. SNI 和 Host Header 一样吗？

不一样。SNI 在 TLS 握手阶段，Host Header 在 HTTP 请求阶段。

---

## 八、本节达标标准

学完本节后，你应该能够做到：

- 解释 HTTPS 解决的三个问题。
- 解释证书和 CA 的作用。
- 使用 `curl -v` 和 `openssl s_client` 查看 TLS 信息。
- 解释 SNI 为什么存在。
- 知道生产中不能随意跳过证书验证。
