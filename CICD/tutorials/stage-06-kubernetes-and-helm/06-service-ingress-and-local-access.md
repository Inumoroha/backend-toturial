# 06：Service、Ingress 与本地访问

## 1. 本节目标

这一节学习 Kubernetes 中服务暴露方式：

- ClusterIP。
- NodePort。
- LoadBalancer。
- Ingress。
- port-forward。

## 2. ClusterIP

ClusterIP 是默认 Service 类型。

```yaml
spec:
  type: ClusterIP
```

特点：

- 只在集群内访问。
- 稳定虚拟 IP。
- 适合服务之间调用。

本地访问可以用：

```bash
kubectl -n go-cicd-lab port-forward service/go-cicd-lab 8080:80
```

## 3. NodePort

NodePort 会在每个 Node 上开放一个端口。

```yaml
spec:
  type: NodePort
```

适合学习和简单测试，不是生产 HTTP 入口首选。

查看端口：

```bash
kubectl -n go-cicd-lab get svc
```

## 4. LoadBalancer

LoadBalancer 通常在云 Kubernetes 中使用。

```yaml
spec:
  type: LoadBalancer
```

云厂商会创建外部负载均衡器。

本地 kind 默认没有云负载均衡。minikube 可以用：

```bash
minikube tunnel
```

## 5. Ingress

Ingress 用于 HTTP/HTTPS 七层入口。

它可以按 host/path 路由：

```text
api.example.com -> go-cicd-lab service
```

Ingress 需要 Ingress Controller，例如：

- ingress-nginx。
- Traefik。
- 云厂商 ingress controller。

只创建 Ingress 对象而没有 Controller，不会真正生效。

## 6. Ingress 示例

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: go-cicd-lab
  namespace: go-cicd-lab
spec:
  ingressClassName: nginx
  rules:
    - host: go-cicd-lab.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: go-cicd-lab
                port:
                  number: 80
```

本地需要配置 hosts：

```text
127.0.0.1 go-cicd-lab.local
```

具体地址取决于 kind/minikube 和 ingress controller 安装方式。

## 7. TLS

生产 Ingress 通常需要 TLS：

```yaml
tls:
  - hosts:
      - api.example.com
    secretName: api-tls
```

TLS 证书可以由：

- 手动创建 Secret。
- cert-manager 自动签发。
- 云厂商证书服务。

第 6 阶段只先理解入口模型，证书自动化可以后置。

## 8. 本地访问推荐顺序

学习阶段推荐：

1. `kubectl port-forward`。
2. minikube service。
3. 本地 Ingress Controller。

先让服务在集群内跑通，再折腾 Ingress。

## 9. 常见排查

Service 没后端：

```bash
kubectl -n go-cicd-lab get endpoints go-cicd-lab
kubectl -n go-cicd-lab describe svc go-cicd-lab
```

Ingress 不通：

```bash
kubectl -n go-cicd-lab describe ingress go-cicd-lab
kubectl get ingressclass
kubectl get pods -A | grep ingress
```

## 10. 小练习

1. 使用 ClusterIP Service。
2. 用 port-forward 访问 `/healthz`。
3. 可选：安装 ingress-nginx。
4. 创建 Ingress。
5. 配置本地 hosts。
6. 通过 host 访问服务。

## 11. 本节小结

你现在应该理解：

- ClusterIP 适合集群内部访问。
- NodePort 适合简单测试。
- LoadBalancer 通常依赖云环境。
- Ingress 是 HTTP/HTTPS 入口规则，需要 Controller。
- port-forward 是最简单的本地调试方式。

