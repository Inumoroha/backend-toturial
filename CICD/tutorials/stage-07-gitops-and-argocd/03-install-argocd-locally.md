# 03：本地安装 Argo CD

## 1. 本节目标

这一节在本地 Kubernetes 集群安装 Argo CD。

你要完成：

- 创建 `argocd` namespace。
- 安装 Argo CD。
- 暴露 Argo CD API/UI。
- 获取初始密码。
- 安装和登录 CLI。

## 2. 创建本地集群

如果还没有集群：

```bash
kind create cluster --name go-cicd-lab
kubectl config use-context kind-go-cicd-lab
```

确认：

```bash
kubectl get nodes
```

## 3. 安装 Argo CD

Argo CD 官方 Getting Started 使用：

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

等待：

```bash
kubectl -n argocd get pods
kubectl -n argocd rollout status deployment/argocd-server
```

学习环境使用官方 install manifest 即可。

生产安装需要考虑 HA、SSO、RBAC、备份和升级策略。

## 4. 访问 Argo CD UI

本地最简单方式：

```bash
kubectl -n argocd port-forward service/argocd-server 8080:443
```

浏览器访问：

```text
https://localhost:8080
```

浏览器可能提示自签证书风险，学习环境可以继续。

## 5. 获取初始密码

新版本 Argo CD 初始 admin 密码保存在 Secret 中：

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

Windows PowerShell 可以分两步：

```powershell
$encoded = kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($encoded))
```

用户名：

```text
admin
```

登录后应尽快修改密码。

## 6. 安装 argocd CLI

官方文档提供多种安装方式。

常见：

```bash
brew install argocd
```

或从 GitHub Release 下载。

检查：

```bash
argocd version --client
```

## 7. CLI 登录

如果使用 port-forward：

```bash
argocd login localhost:8080
```

如果证书是自签的，可能需要：

```bash
argocd login localhost:8080 --insecure
```

输入：

```text
username: admin
password: <initial password>
```

## 8. 修改 admin 密码

```bash
argocd account update-password
```

学习环境也建议修改。

生产环境应禁用或严格保护 admin，并接入 SSO/RBAC。

## 9. 本地访问目标集群

Argo CD 安装在同一个集群时，Application 的 destination server 使用：

```text
https://kubernetes.default.svc
```

这是集群内部 Kubernetes API 地址。

如果要部署到外部集群，需要注册集群凭证。

## 10. 小练习

完成：

1. 安装 Argo CD。
2. port-forward UI。
3. 获取初始密码。
4. 登录 UI。
5. 安装 CLI。
6. CLI 登录。
7. 修改 admin 密码。

## 11. 本节小结

你现在应该理解：

- Argo CD 本身运行在 Kubernetes 中。
- UI/API 默认不直接暴露，需要 port-forward、LoadBalancer 或 Ingress。
- 初始 admin 密码来自 Kubernetes Secret。
- 同集群部署使用 `https://kubernetes.default.svc`。

