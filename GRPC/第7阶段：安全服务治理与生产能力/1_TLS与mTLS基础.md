# 1. TLS 与 mTLS 基础

本节目标：理解 gRPC 安全连接的基本配置，区分 TLS 和 mTLS。

本地开发可以用 insecure 连接，但真实环境服务间通信通常要加密。TLS 保证客户端连接的是可信服务端，mTLS 进一步要求服务端也验证客户端身份。

---

## 一、TLS 与 mTLS 的区别

- TLS：客户端验证服务端证书，确认自己连接的是可信服务端。
- mTLS：服务端也验证客户端证书，双方互相确认身份。
- TLS 解决传输安全，不等同于业务权限控制。
- 业务鉴权仍然需要 token、证书身份映射、RBAC 或其他策略。

---

## 二、服务端启用 TLS

```go
creds, err := credentials.NewServerTLSFromFile("server.crt", "server.key")
if err != nil {
    log.Fatal(err)
}

s := grpc.NewServer(grpc.Creds(creds))
```

---

## 三、客户端使用 TLS

```go
creds, err := credentials.NewClientTLSFromFile("server.crt", "localhost")
if err != nil {
    log.Fatal(err)
}

conn, err := grpc.Dial(
    "localhost:50051",
    grpc.WithTransportCredentials(creds),
)
```

第二个参数 `localhost` 用于校验证书中的主机名。真实环境里应使用服务域名。

---

## 四、mTLS 的思路

mTLS 需要：

1. 服务端加载自己的证书和私钥。
2. 服务端加载 CA，用来验证客户端证书。
3. 客户端加载自己的证书和私钥。
4. 客户端加载 CA，用来验证服务端证书。

mTLS 更适合服务间调用，尤其是零信任、Service Mesh 或高安全要求的内部系统。

---

## 五、常见问题

- 以为 TLS 等于鉴权：TLS 证明连接安全，业务权限还要额外控制。
- 证书 CN/SAN 和连接域名不匹配：客户端会验证失败。
- 在生产中继续使用 insecure：服务间流量可能被窃听或篡改。
- 证书过期没有监控：到期时服务调用会突然失败。

---

## 六、练习任务

1. 将 UserService 从 insecure 改成 TLS。
2. 记录证书不匹配时的错误。
3. 思考哪些服务需要 mTLS。
4. 写下证书过期应该如何告警。

---

## 七、完成标准

- 能配置 TLS gRPC server/client。
- 能解释 TLS 和 mTLS 区别。
- 知道证书管理是生产重点。


---

## 七、完整操作步骤

本节目标是把本地 insecure gRPC 连接改成 TLS 连接，并理解 mTLS 的升级思路。

操作步骤：

1. 创建 `certs` 目录。
2. 生成本地自签名证书。
3. server 加载 `server.crt` 和 `server.key`。
4. client 使用 `server.crt` 作为信任证书连接 server。
5. 验证 TLS 调用成功。
6. 故意使用错误 server name，观察证书校验失败。
7. 理解 mTLS 需要额外验证客户端证书。

---

## 八、生成本地测试证书

在项目根目录创建目录：

```powershell
New-Item -ItemType Directory -Force certs
```

如果你的系统有 OpenSSL，可以执行：

```powershell
openssl req -x509 -newkey rsa:2048 `
  -keyout certs/server.key `
  -out certs/server.crt `
  -days 365 `
  -nodes `
  -subj "/CN=localhost" `
  -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"
```

生成后检查：

```powershell
Get-ChildItem certs
```

预期：

```text
server.crt
server.key
```

注意：现代 TLS 校验通常依赖 SAN，只有 CN 可能不够，所以命令里加了 `subjectAltName`。

---

## 九、完整代码：TLS server

```go
creds, err := credentials.NewServerTLSFromFile("certs/server.crt", "certs/server.key")
if err != nil {
    log.Fatalf("load server tls credentials: %v", err)
}

server := grpc.NewServer(
    grpc.Creds(creds),
    grpc.ChainUnaryInterceptor(
        middleware.RecoveryInterceptor,
        middleware.UnaryLoggingInterceptor,
    ),
)

userv1.RegisterUserServiceServer(server, newUserServer())
reflection.Register(server)
```

需要 import：

```go
import "google.golang.org/grpc/credentials"
```

启动后 server 仍然监听 `:50051`，但客户端不能再用 plaintext 连接。

---

## 十、完整代码：TLS client

```go
creds, err := credentials.NewClientTLSFromFile("certs/server.crt", "localhost")
if err != nil {
    log.Fatalf("load client tls credentials: %v", err)
}

conn, err := grpc.Dial(
    "localhost:50051",
    grpc.WithTransportCredentials(creds),
)
if err != nil {
    log.Fatalf("dial: %v", err)
}
defer conn.Close()
```

第二个参数 `localhost` 是 server name，必须和证书中的 SAN 匹配。

---

## 十一、运行命令

启动 TLS server：

```powershell
go run ./cmd/server
```

运行 TLS client：

```powershell
go run ./cmd/client
```

预期输出：

```text
user id=1 name=Tom email=tom@example.com
```

如果使用 grpcurl：

```powershell
grpcurl -cacert certs/server.crt localhost:50051 list
```

不要再加 `-plaintext`。

---

## 十二、错误实验：server name 不匹配

把 client 改成：

```go
credentials.NewClientTLSFromFile("certs/server.crt", "wrong-host")
```

运行后预期失败，可能看到类似：

```text
certificate is valid for localhost, not wrong-host
```

这说明 TLS 不只是加密，还会验证你连的是不是证书声明的服务端。

---

## 十三、mTLS 思路

普通 TLS：

```text
client 验证 server
```

mTLS：

```text
client 验证 server
server 验证 client
```

mTLS 服务端大致需要：

```go
tlsConfig := &tls.Config{
    ClientAuth: tls.RequireAndVerifyClientCert,
    ClientCAs:  clientCAPool,
}
creds := credentials.NewTLS(tlsConfig)
```

客户端也要加载自己的证书：

```go
cert, err := tls.LoadX509KeyPair("certs/client.crt", "certs/client.key")
```

学习阶段先跑通普通 TLS；mTLS 在服务间安全要求更高时使用。

---

## 十四、常见错误排查

### 1. 继续使用 -plaintext

TLS server 不能用：

```powershell
grpcurl -plaintext localhost:50051 list
```

应该用：

```powershell
grpcurl -cacert certs/server.crt localhost:50051 list
```

### 2. 证书不包含 SAN

错误可能是：

```text
cannot validate certificate because it doesn't contain any IP SANs
```

重新生成证书，加入 `subjectAltName`。

### 3. server name 不匹配

client 的 server name 必须匹配证书。

### 4. 生产环境使用自签名证书

生产环境应使用组织内 CA、云厂商证书、Service Mesh 或证书管理系统，并监控过期时间。

---

## 十五、练习任务

1. 生成本地 `server.crt` 和 `server.key`。
2. server 使用 TLS 启动。
3. client 使用 TLS 调用成功。
4. grpcurl 使用 `-cacert` 调用成功。
5. 故意修改 server name，观察失败。
6. 写一段话解释 TLS 和 token 鉴权的区别。

---

## 十六、完成标准

完成本节后，你应该能：

```text
生成本地测试 TLS 证书
让 gRPC server 使用 TLS
让 gRPC client 使用 TLS
用 grpcurl 调试 TLS gRPC 服务
解释 TLS 和 mTLS 的区别
知道证书 SAN、server name、CA 信任链的重要性
```

---

## 教程闭环检查

1. **完整操作步骤**：生成证书，改 server，改 client，验证成功和失败路径。
2. **完整代码**：使用正文中的 TLS server/client 示例。
3. **运行命令**：执行 openssl、go run、grpcurl -cacert。
4. **预期输出**：TLS client 能成功返回用户，错误 server name 会失败。
5. **常见错误排查**：重点检查 plaintext、SAN、server name、自签名证书信任。
6. **练习任务**：完成证书生成、TLS 调用、grpcurl 调试和错误实验。
7. **完成标准**：能把 insecure gRPC 改成 TLS gRPC。