# 5. Service 与服务发现

本节目标：学完后你能用 Service 给一组后端 Pod 提供稳定访问地址。

## 简短引入

Pod 会被重建，IP 会变化。后端项目不能把某个 Pod IP 写进配置里。Service 的作用就是给一组 Pod 一个稳定入口。

## 一、为什么需要它

真实后端项目常见服务调用：

- 订单服务调用库存服务。
- 用户服务调用权限服务。
- 网关调用文章服务。

如果直接调用 Pod IP，Pod 一重建就失效。Service 通过标签选择 Pod，并提供稳定 DNS 名称。

```text
在集群内部，服务之间优先使用 Service 名称访问，而不是 Pod IP。
```

## 二、基本用法

为 `short-api` 创建 Service：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: short-api
spec:
  type: ClusterIP
  selector:
    app: short-api
  ports:
    - name: http
      port: 8080
      targetPort: 8080
```

应用：

```bash
kubectl apply -f service.yaml
kubectl get svc short-api
```

本地调试：

```bash
kubectl port-forward svc/short-api 8080:8080
```

Windows PowerShell：

```powershell
curl.exe http://127.0.0.1:8080/healthz
```

Linux/macOS：

```bash
curl http://127.0.0.1:8080/healthz
```

## 三、关键参数/语法/代码结构

- `type: ClusterIP`：默认类型，只能集群内部访问。
- `selector`：选择后端 Pod，必须匹配 Pod 标签。
- `port`：Service 暴露的端口。
- `targetPort`：Pod 容器实际监听端口。
- `name`：端口名称，多端口 Service 建议写清楚。

常见类型：

- `ClusterIP`：内部调用，最常用。
- `NodePort`：通过节点端口暴露，学习环境可用，生产要谨慎。
- `LoadBalancer`：云厂商环境中创建负载均衡器。

## 四、真实后端场景示例

订单服务调用库存服务时，可以把库存服务地址配置成：

```text
http://inventory-api:8080
```

如果跨 namespace，例如订单服务在 `order`，库存服务在 `stock`，可以使用：

```text
http://inventory-api.stock.svc.cluster.local:8080
```

一般项目中，能用短名称就不要把完整集群域名写死到代码里。更推荐通过环境变量注入：

```yaml
env:
  - name: INVENTORY_API_BASE_URL
    value: "http://inventory-api:8080"
```

## 五、注意点

- Service 选不到 Pod 时，访问会失败或超时。
- `port` 和 `targetPort` 不一定相同，但初学阶段建议先保持一致。
- `port-forward` 适合调试，不适合生产流量。
- 生产公网入口通常通过 Ingress 或云负载均衡，不直接暴露每个后端 Service。

## 六、常见错误

- 错误：Service 存在但访问不到。
  用 `kubectl get endpoints short-api` 检查是否选到了 Pod。
- 错误：selector 写成 `app: shortapi`。
  标签不匹配时 Service 不会自动报错，但没有后端。
- 错误：把 ClusterIP 当公网地址。
  ClusterIP 只能在集群内部使用。

## 七、本节达标标准

- 能创建 ClusterIP Service。
- 能解释 `port` 和 `targetPort` 的区别。
- 能用 `port-forward` 调试 Service。
- 能排查 Service selector 选不到 Pod 的问题。

## 八、逐跳验证网络

Service 访问失败时，把链路拆成三段：应用自身、Service 后端、Service 入口。

```bash
kubectl get pods -l app=short-api --show-labels
kubectl get svc short-api -o yaml
kubectl get endpointslices -l kubernetes.io/service-name=short-api
kubectl port-forward svc/short-api 8080:8080
```

EndpointSlice 中应该出现一个或多个 Pod IP；为空时优先比较 Service 的 `selector` 与 Pod 的 `labels`。两边必须键和值都相同。

可以从集群内部验证 DNS：

```bash
kubectl run curl-test --rm -it --restart=Never --image=curlimages/curl -- \
  curl -sS http://short-api:8080/healthz
```

预期输出是 `ok`。如果镜像暂时拉取不到，不代表 Service 配置错误，可以先使用 `port-forward` 完成验证。

自测：Service 的 ClusterIP 为什么不用随着 Pod 重建而改变？因为客户端访问的是 Service 的虚拟地址，Kubernetes 会维护它到当前 Ready Pod 的后端映射。
