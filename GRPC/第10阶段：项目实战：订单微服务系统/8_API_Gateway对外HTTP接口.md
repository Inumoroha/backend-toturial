# 8. API Gateway 对外 HTTP 接口

本节目标：使用 gRPC-Gateway 为订单系统提供 HTTP/JSON 入口。

项目内部服务使用 gRPC，对外入口使用 HTTP/JSON。Gateway 层负责协议转换、基础鉴权、请求日志，不负责核心订单业务。

---

## 一、对外接口设计

```text
POST /v1/orders          创建订单
GET  /v1/orders/{id}     查询订单
GET  /v1/users/{id}      查询用户
GET  /v1/products/{id}   查询商品
```

创建订单请求：

```json
{
  "userId": 1,
  "productId": 100,
  "quantity": 2,
  "idempotencyKey": "request-001"
}
```

---

## 二、proto 注解

```proto
rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse) {
  option (google.api.http) = {
    post: "/v1/orders"
    body: "*"
  };
}

rpc GetOrder(GetOrderRequest) returns (GetOrderResponse) {
  option (google.api.http) = {
    get: "/v1/orders/{id}"
  };
}
```

---

## 三、curl 测试

```bash
curl -X POST http://localhost:8080/v1/orders \
  -H 'Content-Type: application/json' \
  -d '{"userId":1,"productId":100,"quantity":2,"idempotencyKey":"request-001"}'
```

查询：

```bash
curl http://localhost:8080/v1/orders/1
```

---

## 四、trace id 传递

Gateway 收到 HTTP 请求后，可以读取或生成 trace id：

```text
HTTP Header: X-Trace-Id
-> gRPC metadata: x-trace-id
-> 下游服务日志字段 trace_id
```

这样一次外部请求进入内部多个 gRPC 服务后，日志仍然能串起来。

---

## 五、常见问题

- Gateway 层拼装复杂业务：会让入口和业务耦合。
- HTTP 字段命名和 proto JSON 命名不一致：要统一文档。
- 忘记传递 trace id：外部请求链路进入内部后断开。
- 对外错误响应直接暴露 gRPC 原始信息：需要统一格式。

---

## 六、练习任务

1. 实现 POST /v1/orders。
2. 实现 GET /v1/orders/{id}。
3. HTTP 请求进入后继续向 gRPC metadata 传 trace id。
4. 为 NotFound 设计 HTTP 404 响应。

---

## 七、完成标准

- HTTP 能成功创建订单。
- Gateway 不包含核心业务。
- 错误响应格式统一。

---

## 八、完整操作步骤：用 Gateway 暴露 HTTP/JSON

内部服务使用 gRPC，但外部调用方常常更习惯 HTTP/JSON。Gateway 的职责是协议转换，不是重写业务逻辑。

实现顺序：

1. 在 order proto 中补充 HTTP annotation。
2. 重新生成 gateway 代码。
3. 创建 `cmd/gateway/main.go`。
4. Gateway 连接 order-service。
5. 注册 HTTP handler。
6. 用 curl 调用 HTTP 接口。
7. 确认 Gateway 不直接访问 user/product/order 的内存数据。

安装依赖：

```powershell
go get github.com/grpc-ecosystem/grpc-gateway/v2/runtime
go get google.golang.org/grpc
go mod tidy
```

如果使用 buf，还需要在 `buf.gen.yaml` 中增加 gateway 插件。项目学习阶段可以先聚焦 order-service 的 HTTP 入口。

---

## 九、完整代码：Gateway main 函数

`cmd/gateway/main.go`：

```go
package main

import (
	"context"
	"log"
	"net/http"

	orderv1 "example.com/grpc-shop/gen/order/v1"
	"github.com/grpc-ecosystem/grpc-gateway/v2/runtime"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
	"google.golang.org/grpc/metadata"
)

func main() {
	ctx := context.Background()
	mux := runtime.NewServeMux(
		runtime.WithMetadata(func(ctx context.Context, r *http.Request) metadata.MD {
			traceID := r.Header.Get("X-Trace-Id")
			if traceID == "" {
				return nil
			}
			return metadata.Pairs("x-trace-id", traceID)
		}),
	)

	opts := []grpc.DialOption{grpc.WithTransportCredentials(insecure.NewCredentials())}
	if err := orderv1.RegisterOrderServiceHandlerFromEndpoint(ctx, mux, "localhost:50053", opts); err != nil {
		log.Fatalf("register order gateway: %v", err)
	}

	log.Println("gateway listening on :8080")
	if err := http.ListenAndServe(":8080", mux); err != nil {
		log.Fatalf("serve gateway: %v", err)
	}
}
```

如果你还没有生成 `RegisterOrderServiceHandlerFromEndpoint`，说明 gateway 插件还没配置。buf 生成配置需要包含类似内容：

```yaml
  - remote: buf.build/grpc-ecosystem/gateway
    out: gen
    opt:
      - paths=source_relative
```

order proto 中可以为 `CreateOrder` 增加 HTTP 映射：

```proto
import "google/api/annotations.proto";

service OrderService {
  rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse) {
    option (google.api.http) = {
      post: "/v1/orders"
      body: "*"
    };
  }

  rpc GetOrder(GetOrderRequest) returns (GetOrderResponse) {
    option (google.api.http) = {
      get: "/v1/orders/{id}"
    };
  }
}
```

---

## 十、运行命令与预期输出

启动内部服务：

```powershell
go run .\cmd\user-service
go run .\cmd\product-service
go run .\cmd\order-service
```

另开窗口启动 Gateway：

```powershell
go run .\cmd\gateway
```

预期输出：

```text
gateway listening on :8080
```

HTTP 创建订单：

```powershell
curl.exe -X POST http://localhost:8080/v1/orders -H "Content-Type: application/json" -H "X-Trace-Id: demo-trace-001" -d "{\"userId\":1,\"productId\":1,\"quantity\":1,\"requestId\":\"http-001\"}"
```

预期输出：

```json
{
  "order": {
    "id": "1",
    "userId": "1",
    "productId": "1",
    "quantity": 1,
    "totalCents": "9900",
    "status": "ORDER_STATUS_CREATED"
  }
}
```

HTTP 查询订单：

```powershell
curl.exe http://localhost:8080/v1/orders/1
```

---

## 十一、常见错误排查

- `RegisterOrderServiceHandlerFromEndpoint undefined`：没有生成 gateway 代码，检查 buf 插件。
- `google/api/annotations.proto not found`：缺少 googleapis proto import path。
- curl 请求字段不生效：检查 JSON 字段名，proto 的 `user_id` 默认 JSON 名通常是 `userId`。
- Gateway 返回 502 或连接失败：order-service 没启动，或地址不是 `localhost:50053`。
- Gateway 里写库存判断：职责越界，应交给 product-service 和 order-service。
- trace id 没进入内部日志：检查 Gateway 是否把 HTTP header 转成 metadata。

---

## 十二、教程闭环检查

本篇的完整操作步骤是：补 HTTP annotation、生成 Gateway 代码、启动内部服务、启动 Gateway、用 curl 创建和查询订单。完整代码包含 Gateway main、metadata 传递和 proto annotation 示例。运行命令包含 `go get`、`go mod tidy`、四个服务启动命令和 curl。预期输出覆盖 Gateway 启动和 HTTP 创建订单响应。常见错误排查覆盖 gateway 代码生成、annotations、JSON 字段、连接失败、职责越界和 trace id。练习任务是为 NotFound 设计统一 HTTP 错误响应。完成标准是：外部 HTTP 请求可以进入内部 gRPC 链路，Gateway 不包含核心业务。
