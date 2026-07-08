# 3. 网络隔离、bind、protected-mode 与 TLS 基础

Redis 安全首先是网络安全。

密码和 ACL 很重要，但如果 Redis 暴露在公网，风险仍然很高。生产 Redis 应该尽量只允许可信应用在内网访问。

学完这一节后，你应该能够：

- 理解为什么 Redis 不应该公网暴露。
- 配置 `bind` 和 `protected-mode`。
- 使用防火墙或安全组限制访问来源。
- 理解 Docker 端口映射风险。
- 知道 TLS 在 Redis 中解决什么问题。

---

## 一、禁止公网裸奔

Redis 不应该直接暴露到公网。

危险包括：

- 被扫描发现。
- 被暴力破解。
- 被执行危险命令。
- 数据被读取。
- 数据被清空。
- 服务器被进一步攻击。

即使设置了密码，也不应该把 Redis 当公网服务开放。

---

## 二、bind 配置

配置 Redis 监听地址：

```conf
bind 127.0.0.1
```

表示只监听本机。

内网部署可以：

```conf
bind 127.0.0.1 10.0.0.10
```

不要随意：

```conf
bind 0.0.0.0
```

除非你非常清楚安全组、防火墙和认证配置。

---

## 三、protected-mode

配置：

```conf
protected-mode yes
```

当 Redis 检测到不安全配置时，会限制外部访问。

这是保护机制，不应该在不理解风险的情况下关闭。

如果你为了远程连接而设置：

```conf
protected-mode no
```

同时又没有密码和网络隔离，就非常危险。

---

## 四、安全组和防火墙

云服务器应通过安全组限制：

```text
只允许应用服务器 IP 访问 Redis 6379。
```

Linux 防火墙也可以限制来源。

原则：

```text
能在网络层拒绝，就不要只依赖应用层密码。
```

Redis 安全应该是多层防护。

---

## 五、Docker 端口映射风险

开发时常写：

```yaml
ports:
  - "6379:6379"
```

这会把容器端口映射到宿主机。

如果宿主机公网可访问，Redis 可能被外部访问。

更安全的做法：

- 只在本地开发使用端口映射。
- 生产中使用内部网络。
- 不对公网开放 Redis 端口。
- 使用防火墙限制来源。

---

## 六、Docker Compose 内部访问

多个容器在同一个 Compose 网络中，可以用服务名访问：

```text
redis:6379
```

不一定需要把 Redis 端口映射到宿主机。

示例：

```yaml
services:
  app:
    image: my-app
    depends_on:
      - redis

  redis:
    image: redis:7
```

app 可以访问 `redis:6379`。

外部不能直接访问 Redis。

---

## 七、TLS 解决什么问题

TLS 用于加密客户端和 Redis 之间的通信。

它可以防止：

- 网络中间人窃听。
- 明文密码被抓包。
- 数据在传输中被读取。

适合：

- 跨主机通信。
- 跨网络边界。
- 合规要求。
- Redis 服务托管环境。

如果 Redis 和应用在同一可信内网，是否启用 TLS 要结合安全要求和性能开销。

---

## 八、TLS 基础配置概念

Redis TLS 配置通常涉及：

```conf
tls-port 6379
port 0
tls-cert-file /path/redis.crt
tls-key-file /path/redis.key
tls-ca-cert-file /path/ca.crt
```

含义：

- 启用 TLS 端口。
- 禁用明文端口。
- 配置证书、私钥和 CA。

具体证书生成和部署要结合公司的证书管理规范。

---

## 九、Go 中启用 TLS

go-redis 可以配置 TLS：

```go
rdb := redis.NewClient(&redis.Options{
    Addr:      "redis.example.internal:6379",
    Username:  "app",
    Password:  "password",
    TLSConfig: &tls.Config{
        MinVersion: tls.VersionTLS12,
    },
})
```

生产中应该配置 CA 校验。

不要为了省事关闭证书校验。

---

## 十、安全检查清单

检查：

```text
1. Redis 是否绑定公网地址？
2. 安全组是否只允许应用访问？
3. protected-mode 是否开启？
4. 是否设置认证？
5. 是否使用 ACL？
6. Docker 是否暴露了 6379？
7. 是否需要 TLS？
8. 是否有访问日志和连接监控？
```

这些检查应该纳入上线流程。

---

## 十一、常见错误

### 1. `bind 0.0.0.0` 但没有安全组

Redis 可能直接暴露公网。

### 2. 为了远程访问关闭 protected-mode

如果没有其他防护，非常危险。

### 3. Docker 生产环境暴露 6379

端口映射可能把 Redis 暴露给外部。

### 4. 密码通过明文网络传输

跨网络访问时应考虑 TLS。

### 5. 关闭 TLS 证书校验

会削弱 TLS 的安全意义。

---

## 十二、本节练习

请完成下面练习：

1. 查看 Redis 当前 `bind` 配置。
2. 查看 `protected-mode`。
3. 检查 Docker Compose 是否映射 6379。
4. 设计一条安全组规则，只允许应用服务器访问 Redis。
5. 解释 TLS 解决什么问题。
6. 在 Go redis Options 中加入 `TLSConfig`。
7. 写一份 Redis 网络安全检查清单。

---

## 十三、本节小结

这一节你学习了 Redis 网络安全。

你需要记住：

- Redis 不应该公网裸奔。
- `bind` 和 `protected-mode` 是基础保护。
- 安全组和防火墙应该限制来源。
- Docker 端口映射可能造成意外暴露。
- TLS 用于保护传输安全，跨网络访问时尤其重要。

下一节我们学习危险命令限制和配置文件管理。

