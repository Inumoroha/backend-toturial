# 10. Namespace 与环境隔离

本节目标：学完后你能用 Namespace 区分不同环境或业务边界，降低误操作风险。

## 简短引入

后端项目通常有开发、测试、预发、生产环境。Kubernetes 中 Namespace 可以理解为集群里的逻辑工作区，用来隔离资源名称、权限和默认操作范围。

## 一、为什么需要它

如果所有资源都放在 `default`，很快会混乱：

- `user-api` 到底是测试还是生产？
- 删除 Service 时会不会删错环境？
- 开发同学是否能看到生产 Secret？
- CI/CD 应该部署到哪里？

Namespace 不能替代物理集群隔离，但它是最基本的逻辑隔离手段。

## 二、基本用法

创建 namespace：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: backend-dev
```

应用：

```bash
kubectl apply -f namespace.yaml
kubectl get ns
```

把资源部署到指定 namespace：

```bash
kubectl apply -n backend-dev -f deployment.yaml
kubectl get pods -n backend-dev
```

也可以在 YAML 里写：

```yaml
metadata:
  name: short-api
  namespace: backend-dev
```

## 三、关键参数/语法/代码结构

常用命令：

```bash
kubectl get all -n backend-dev
kubectl describe pod <pod-name> -n backend-dev
kubectl logs <pod-name> -n backend-dev
kubectl config set-context --current --namespace=backend-dev
```

真实项目中，建议 CI/CD 显式传 `-n`，不要依赖操作人员当前默认 namespace。

```text
生产脚本里不要依赖“当前 namespace”。部署命令应该明确写出目标 namespace。
```

## 四、真实后端场景示例

短链接服务可以有两个环境：

```text
backend-dev
backend-prod
```

同名资源在不同 namespace 中可以同时存在：

```bash
kubectl get deploy short-api -n backend-dev
kubectl get deploy short-api -n backend-prod
```

配置也分别管理：

```bash
kubectl get configmap short-api-config -n backend-dev
kubectl get configmap short-api-config -n backend-prod
```

这样可以让同一套服务配置在不同环境中独立变化。

## 五、注意点

- Namespace 是逻辑隔离，不是强安全边界。
- 跨 namespace 调用 Service 时要写清目标 namespace。
- 生产 Secret 不应该让开发 namespace 的账号读取。
- `default` namespace 适合学习，不适合长期承载正式项目。

## 六、常见错误

- 错误：资源 apply 成功，但 get 看不到。
  很可能 apply 到了另一个 namespace。
- 错误：Service 名称在 dev 能访问，prod 不能访问。
  检查调用方和被调用方是否在同一个 namespace。
- 错误：所有环境共用一个 Secret。
  环境边界会变得不清晰，轮换也困难。

## 七、本节达标标准

- 能创建 Namespace。
- 能把 Deployment、Service、ConfigMap 部署到指定 Namespace。
- 能解释 Namespace 的隔离作用和限制。
- 能避免常见的 namespace 误操作。

