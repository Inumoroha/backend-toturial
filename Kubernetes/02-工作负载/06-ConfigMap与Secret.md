# 6. ConfigMap 与 Secret

本节目标：学完后你能把后端服务配置和敏感信息从镜像中拆出来，并注入到 Pod。

## 简短引入

同一个 Go 服务，在开发环境可能连本地 PostgreSQL，在生产环境连云数据库。镜像应该保持一致，变化的是配置。Kubernetes 中普通配置用 ConfigMap，敏感配置用 Secret。

## 一、为什么需要它

真实项目里常见配置：

- `APP_ENV`
- `HTTP_PORT`
- `LOG_LEVEL`
- `DATABASE_URL`
- `JWT_SECRET`
- 第三方支付密钥

如果这些都写进代码或镜像，就会带来两个问题：环境切换困难，敏感信息泄露风险高。

```text
配置可以进 ConfigMap，密码和令牌必须进 Secret 或外部密钥系统。
```

## 二、基本用法

创建 `configmap.yaml`：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: short-api-config
data:
  APP_ENV: "dev"
  LOG_LEVEL: "debug"
  HTTP_PORT: "8080"
```

创建 `secret.yaml`：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: short-api-secret
type: Opaque
stringData:
  DATABASE_URL: "postgres://app:password@postgres:5432/short?sslmode=disable"
  JWT_SECRET: "change-me-in-real-env"
```

在 Deployment 中引用：

```yaml
envFrom:
  - configMapRef:
      name: short-api-config
  - secretRef:
      name: short-api-secret
```

应用：

```bash
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl apply -f deployment.yaml
```

## 三、关键参数/语法/代码结构

- `data`：ConfigMap 的键值。
- `stringData`：创建 Secret 时可以直接写明文，API Server 会保存为编码后的数据。
- `envFrom`：把整个 ConfigMap 或 Secret 注入为环境变量。
- `env`：逐个注入，适合想显式控制变量名的情况。

逐个注入示例：

```yaml
env:
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: short-api-secret
        key: DATABASE_URL
```

## 四、真实后端场景示例

Go 服务读取配置：

```go
package config

import (
	"log"
	"os"
)

type Config struct {
	AppEnv      string
	LogLevel    string
	DatabaseURL string
}

func Load() Config {
	cfg := Config{
		AppEnv:      os.Getenv("APP_ENV"),
		LogLevel:    os.Getenv("LOG_LEVEL"),
		DatabaseURL: os.Getenv("DATABASE_URL"),
	}
	if cfg.DatabaseURL == "" {
		log.Fatal("DATABASE_URL is required")
	}
	return cfg
}
```

真实项目中，服务启动时尽早校验必要配置。不要等到用户请求进来才发现数据库地址为空。

## 五、注意点

- Secret 默认不是完整加密保险箱，只是 Kubernetes 的敏感数据资源。生产环境还要考虑 etcd 加密、权限控制和外部密钥管理。
- ConfigMap 更新后，环境变量不会自动刷新到已运行进程。通常需要重启 Pod。
- 不要把 Secret YAML 提交到公开仓库。
- 配置项命名要稳定，频繁改名会让部署和代码都变脆。

## 六、常见错误

- 错误：修改 ConfigMap 后服务配置没变。
  环境变量方式注入通常需要重启 Pod。
- 错误：把密码放进 ConfigMap。
  ConfigMap 更适合普通配置，不适合数据库密码和令牌。
- 错误：服务启动后才在请求里检查配置。
  配置错误应该尽早失败，方便发布阶段发现。

## 七、本节达标标准

- 能创建 ConfigMap 和 Secret。
- 能把配置注入 Deployment。
- 能在 Go 服务中读取环境变量。
- 能说明 Secret 的使用边界和风险。

## 八、验证配置是否真的注入

应用三个 YAML 后，先确认引用对象存在，再查看 Deployment 是否完成更新：

```bash
kubectl get configmap short-api-config
kubectl get secret short-api-secret
kubectl rollout status deployment/short-api
kubectl exec deployment/short-api -- printenv APP_ENV
```

预期输出为 `dev`。不要用 `printenv` 查看真实生产密码，也不要把完整 Secret 内容贴进聊天、工单或日志。

修改 `LOG_LEVEL` 后，运行：

```bash
kubectl apply -f configmap.yaml
kubectl rollout restart deployment/short-api
kubectl rollout status deployment/short-api
```

这里需要重启，是因为进程启动时读取环境变量；Kubernetes 不会修改已运行进程的环境。若以文件卷挂载 ConfigMap，文件更新行为不同，但应用仍要支持重载，初学阶段先掌握环境变量方式。

练习：从 Secret YAML 中暂时删除 `DATABASE_URL`，观察应用是否按代码设计快速失败，再恢复配置。这个实验能验证“启动时校验必要配置”是否真的生效。
