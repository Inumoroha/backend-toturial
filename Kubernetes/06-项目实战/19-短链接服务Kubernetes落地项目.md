# 19. 短链接服务 Kubernetes 落地项目

本节目标：学完后你能把一个短链接 Go 后端服务按真实项目思路部署到 Kubernetes。

## 简短引入

前面每节都讲一个主题。本节把它们串起来：镜像、Deployment、Service、ConfigMap、Secret、Ingress、Job、CronJob、排障和回滚。你可以把它当作一个小型项目模板。

## 一、为什么需要它

学习 Kubernetes 最容易卡在“每个概念都听过，但不知道项目里怎么组织”。短链接服务包含后端常见要素：

- HTTP API。
- 数据库连接。
- 数据库迁移。
- 配置和密钥。
- 对外访问。
- 定时清理。
- 发布回滚。

这比只部署一个 hello world 更接近真实工作。

## 二、基本用法

建议目录：

```text
short-api/
  cmd/
    api/
      main.go
    migrate/
      main.go
  main.go
  Dockerfile
  migrations/
    001_create_short_links.sql
    001_create_short_links_down.sql
  k8s/
    namespace.yaml
    configmap.yaml
    secret.example.yaml
    deployment.yaml
    service.yaml
    ingress.yaml
    migrate-job.yaml
    cleanup-cronjob.yaml
```

创建 namespace：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: backend-dev
```

配置：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: short-api-config
  namespace: backend-dev
data:
  APP_ENV: "dev"
  LOG_LEVEL: "debug"
  HTTP_PORT: "8080"
```

密钥示例文件只作为模板，不直接提交真实密码：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: short-api-secret
  namespace: backend-dev
type: Opaque
stringData:
  DATABASE_URL: "postgres://app:password@postgres:5432/short?sslmode=disable"
  JWT_SECRET: "change-me"
```

## 三、关键参数/语法/代码结构

Deployment：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: short-api
  namespace: backend-dev
spec:
  replicas: 2
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: short-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    metadata:
      labels:
        app: short-api
    spec:
      containers:
        - name: short-api
          image: short-api:0.1.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: short-api-config
            - secretRef:
                name: short-api-secret
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8080
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
```

Service：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: short-api
  namespace: backend-dev
spec:
  selector:
    app: short-api
  ports:
    - name: http
      port: 8080
      targetPort: 8080
```

Ingress：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: short-api
  namespace: backend-dev
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

## 四、真实后端场景示例

一次完整本地部署流程：

```bash
docker build -t short-api:0.1.0 .
kind load docker-image short-api:0.1.0 --name backend-lab
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secret.example.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl rollout status deployment/short-api -n backend-dev
```

如果不用 kind，跳过 `kind load docker-image`，按你的本地集群方式加载镜像。

本地访问：

```bash
kubectl port-forward -n backend-dev svc/short-api 8080:8080
```

Windows PowerShell：

```powershell
curl.exe http://127.0.0.1:8080/healthz
```

Linux/macOS：

```bash
curl http://127.0.0.1:8080/healthz
```

迁移 Job：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: short-api-migrate-001
  namespace: backend-dev
spec:
  backoffLimit: 1
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: short-api:0.1.0
          imagePullPolicy: IfNotPresent
          command: ["/app/short-api"]
          args: ["migrate", "up"]
          envFrom:
            - secretRef:
                name: short-api-secret
```

定时清理过期短链接：

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: short-link-cleaner
  namespace: backend-dev
spec:
  schedule: "*/10 * * * *"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: cleaner
              image: short-api:0.1.0
              imagePullPolicy: IfNotPresent
              command: ["/app/short-api"]
              args: ["jobs", "clean-expired-links"]
              envFrom:
                - secretRef:
                    name: short-api-secret
```

业务代码里要把数据库操作写得保守一点。创建短链接时不要拼 SQL 字符串，短码唯一性要交给数据库约束兜底：

```go
func CreateShortLink(ctx context.Context, db *sql.DB, code string, originalURL string) error {
	const query = `
INSERT INTO short_links (code, original_url)
VALUES ($1, $2)
ON CONFLICT (code) DO NOTHING`

	result, err := db.ExecContext(ctx, query, code, originalURL)
	if err != nil {
		return fmt.Errorf("create short link: %w", err)
	}
	rows, err := result.RowsAffected()
	if err != nil {
		return fmt.Errorf("read affected rows: %w", err)
	}
	if rows == 0 {
		return ErrCodeAlreadyExists
	}
	return nil
}
```

```text
用户输入不能直接拼进 SQL 字符串。短链接的原始 URL、短码、用户 ID 都应该通过参数化查询传入。
```

跳转统计可以放进事务里，但事务边界要小。下面示例只做“查询短链接并记录一次访问”，不要在事务里调用外部 HTTP 服务：

```go
func ResolveAndRecord(ctx context.Context, db *sql.DB, code string, ip string) (string, error) {
	tx, err := db.BeginTx(ctx, &sql.TxOptions{Isolation: sql.LevelReadCommitted})
	if err != nil {
		return "", fmt.Errorf("begin tx: %w", err)
	}
	defer tx.Rollback()

	var originalURL string
	err = tx.QueryRowContext(ctx, `
SELECT original_url
FROM short_links
WHERE code = $1`, code).Scan(&originalURL)
	if err != nil {
		return "", fmt.Errorf("query short link: %w", err)
	}

	_, err = tx.ExecContext(ctx, `
INSERT INTO short_link_visits (code, ip, visited_at)
VALUES ($1, $2, now())`, code, ip)
	if err != nil {
		return "", fmt.Errorf("record visit: %w", err)
	}

	if err := tx.Commit(); err != nil {
		return "", fmt.Errorf("commit tx: %w", err)
	}
	return originalURL, nil
}
```

迁移文件也要配套回滚思路。创建表可以这样写：

```sql
CREATE TABLE IF NOT EXISTS short_links (
    id BIGSERIAL PRIMARY KEY,
    code VARCHAR(32) NOT NULL UNIQUE,
    original_url TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE IF NOT EXISTS short_link_visits (
    id BIGSERIAL PRIMARY KEY,
    code VARCHAR(32) NOT NULL,
    ip VARCHAR(64) NOT NULL,
    visited_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS idx_short_link_visits_code_visited_at
ON short_link_visits (code, visited_at DESC);
```

回滚文件不要在生产中随手执行，尤其是会删数据的语句。学习环境可以这样理解：

```sql
DROP TABLE IF EXISTS short_link_visits;
DROP TABLE IF EXISTS short_links;
```

```text
生产回滚不等于机械执行 down SQL。涉及删表、删列、清数据时，要先确认备份、影响范围和业务是否还能兼容。
```

## 五、注意点

- `secret.example.yaml` 只能作为示例，真实 Secret 不要提交。
- 数据库迁移要先于依赖新结构的代码发布，或采用兼容发布。
- readiness probe 失败时，Service 不会把流量转给该 Pod。
- 短链接跳转接口要注意原始 URL 校验，防止开放重定向被滥用。
- 创建短码要处理唯一冲突，数据库唯一约束不能省。
- 访问统计不要在主请求里做太重的同步写入，否则跳转延迟会升高。
- 事务里不要做远程调用、文件上传或耗时计算，避免锁持有时间过长。
- 索引不是越多越好，访问统计表的索引要围绕查询条件设计，并观察写入成本。

```text
真实后端项目的 Kubernetes 落地，不是把 YAML 写完，而是让配置、发布、迁移、日志、回滚都形成闭环。
```

## 六、常见误区

- 误区：能访问 `/healthz` 就说明业务正常。
  还要验证核心接口、数据库读写、日志和配置。
- 误区：迁移和 API 同时发布就行。
  表结构不兼容时会直接造成线上错误。
- 误区：所有密钥都写在一个 Secret 里。
  权限和轮换会变困难，应按服务和用途拆分。
- 误区：本地集群成功等于生产可用。
  生产还需要监控、告警、备份、容量和权限评审。
- 误区：数据库迁移只要能执行成功就行。
  还要考虑旧代码兼容、执行时长、索引成本和失败后的处理路径。

## 七、本节达标标准

- 能组织一个后端服务的 Kubernetes 目录。
- 能完成镜像构建、部署、Service 访问。
- 能加入 ConfigMap、Secret、探针和资源限制。
- 能用 Job 执行迁移。
- 能用 CronJob 执行业务定时任务。
- 能描述一次可回滚的发布流程。
- 能写出参数化查询示例，并说明事务边界为什么要小。
