# 9. Ingress 与后端入口

本节目标：学完后你能用 Ingress 为后端 HTTP 服务提供统一入口。

## 简短引入

Service 解决的是集群内部访问，Ingress 解决的是 HTTP 服务从集群外部进入的问题。对后端开发来说，它类似项目入口层的反向代理规则。

## 一、为什么需要它

真实项目中用户访问的是域名：

- `api.example.com/users`
- `api.example.com/orders`
- `short.example.com/aBc123`

不可能给每个服务都暴露一个随机端口。Ingress 可以按域名、路径把请求转发到不同 Service。

```text
Ingress 是规则，Ingress Controller 才是真正执行规则的组件。
```

## 二、基本用法

如果用 Minikube，可以启用 ingress：

```bash
minikube addons enable ingress
```

如果用 kind，通常需要额外安装 ingress-nginx。学习时也可以先理解 YAML，再按本地环境选择安装方式。

Ingress 示例：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: short-api
spec:
  rules:
    - host: short.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: short-api
                port:
                  number: 8080
```

应用：

```bash
kubectl apply -f ingress.yaml
kubectl get ingress
```

本地测试需要把域名指向本地入口 IP。Minikube 查看 IP：

```bash
minikube ip
```

Windows hosts 文件通常在：

```text
C:\Windows\System32\drivers\etc\hosts
```

Linux/macOS hosts 文件：

```text
/etc/hosts
```

添加类似记录：

```text
192.168.49.2 short.local
```

访问：

```bash
curl http://short.local/healthz
```

## 三、关键参数/语法/代码结构

- `host`：域名匹配。
- `path`：路径匹配。
- `pathType: Prefix`：按前缀匹配。
- `backend.service.name`：转发到哪个 Service。
- `backend.service.port.number`：转发到 Service 的哪个端口。

真实项目中，Ingress 常配合 TLS、限流、跨域、请求体大小限制、超时时间等规则。这些能力通常由具体 Ingress Controller 的注解提供。

## 四、真实后端场景示例

一个后端系统可能有两个服务：

- `user-api` 处理 `/users`
- `order-api` 处理 `/orders`

Ingress 可以这样组织：

```yaml
rules:
  - host: api.local
    http:
      paths:
        - path: /users
          pathType: Prefix
          backend:
            service:
              name: user-api
              port:
                number: 8080
        - path: /orders
          pathType: Prefix
          backend:
            service:
              name: order-api
              port:
                number: 8080
```

真实项目中也可以在应用层用网关统一路由。选择 Ingress 分路径还是应用网关，要看团队架构，不要为了 Kubernetes 而拆服务。

## 五、注意点

- Ingress 不会自动生效，必须有 Ingress Controller。
- 本地 hosts 修改只影响本机。
- TLS 证书生产环境要自动续期或有明确轮换流程。
- 不要把管理后台、调试接口随意暴露到公网。

## 六、常见错误

- 错误：创建 Ingress 后访问不了。
  检查是否安装了 Ingress Controller。
- 错误：Service 端口写错。
  Ingress 转发到的是 Service 端口，不是随便写容器端口。
- 错误：路径匹配和应用路由冲突。
  例如 Ingress 截断路径，但应用仍以完整路径匹配。

## 七、本节达标标准

- 能解释 Ingress 和 Service 的区别。
- 能写一个按域名转发的 Ingress。
- 能在本地通过 hosts 测试域名访问。
- 知道 Ingress Controller 是必要组件。

