# 1. 自签名证书与 HTTPS 最小配置

本节目标：使用自签名证书让 Nginx 提供 HTTPS，并理解证书、私钥、`listen 443 ssl` 的基本含义。

学习阶段可以先用自签名证书。它不适合生产，但非常适合理解 HTTPS 配置。

---

## 一、HTTPS 配置需要什么

Nginx 开启 HTTPS 至少需要：

- 监听 443。
- 证书文件。
- 私钥文件。

配置形式：

```nginx
server {
    listen 443 ssl;
    server_name api.local;

    ssl_certificate /path/to/cert.crt;
    ssl_certificate_key /path/to/key.key;
}
```

---

## 二、生成自签名证书

创建目录：

```bash
mkdir -p ~/nginx-lab/certs
```

生成证书：

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout ~/nginx-lab/certs/api.local.key \
  -out ~/nginx-lab/certs/api.local.crt \
  -subj "/CN=api.local"
```

生成两个文件：

```text
api.local.crt  证书
api.local.key  私钥
```

私钥不能泄露。生产中私钥泄露意味着别人可能伪装你的服务。

---

## 三、配置 HTTPS server

```nginx
server {
    listen 443 ssl;
    server_name api.local;

    ssl_certificate /home/your-user/nginx-lab/certs/api.local.crt;
    ssl_certificate_key /home/your-user/nginx-lab/certs/api.local.key;

    location / {
        return 200 "hello https\n";
    }
}
```

检查：

```bash
sudo nginx -t
```

重载：

```bash
sudo systemctl reload nginx
```

---

## 四、使用 curl 验证

自签名证书不被系统信任，所以直接请求可能失败：

```bash
curl https://api.local
```

学习阶段使用：

```bash
curl -k https://api.local
```

`-k` 表示忽略证书信任校验。

期望输出：

```text
hello https
```

---

## 五、查看证书信息

```bash
openssl x509 -in ~/nginx-lab/certs/api.local.crt -noout -subject -dates
```

你可以看到：

- 证书颁发对象。
- 生效时间。
- 过期时间。

---

## 六、常见问题

### 1. 443 端口没监听

检查：

```bash
sudo ss -lntp | grep ':443'
```

### 2. Nginx 无法读取证书

检查：

```bash
sudo nginx -t
```

常见原因：

- 路径写错。
- 文件权限不允许读取。
- key 和 cert 不匹配。

### 3. 浏览器提示不安全

自签名证书正常会提示不安全。生产要使用受信任 CA 签发的证书，比如 Let's Encrypt。

---

## 七、本节练习

1. 生成自签名证书。
2. 配置 `listen 443 ssl`。
3. 用 `curl -k` 访问。
4. 用 openssl 查看证书有效期。
5. 故意写错证书路径，观察 `nginx -t` 报错。

---

## 八、本节复盘

请确认你能回答：

1. HTTPS 最少需要哪两个文件？
2. `listen 443 ssl` 表示什么？
3. 自签名证书为什么浏览器不信任？
4. `curl -k` 的作用是什么？
5. 私钥为什么不能泄露？

