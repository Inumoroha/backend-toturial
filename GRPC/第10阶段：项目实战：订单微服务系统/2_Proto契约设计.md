# 2. Proto 契约设计

本节目标：为订单系统设计可演进的 Protobuf 契约。

项目的 proto 是所有服务协作的基础。这里要把前面学过的 message、service、版本、字段编号、错误语义都用起来。

---

## 一、设计原则

- 每个服务独立 proto 包，例如 `user.v1`、`order.v1`。
- Request/Response 单独设计，不直接复用内部模型。
- enum 第一个值使用 `UNSPECIFIED`。
- 金额建议用 `int64` 表示最小货币单位，例如分。
- 所有 ID 类型要统一。
- 字段编号一旦发布，不要随便修改。

---

## 二、Order proto 示例

```proto
syntax = "proto3";

package order.v1;

option go_package = "example.com/grpc-shop/gen/order/v1;orderv1";

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_CREATED = 1;
  ORDER_STATUS_PAID = 2;
  ORDER_STATUS_CANCELED = 3;
}

message Order {
  int64 id = 1;
  int64 user_id = 2;
  int64 product_id = 3;
  int32 quantity = 4;
  int64 total_amount_cents = 5;
  OrderStatus status = 6;
}

message CreateOrderRequest {
  int64 user_id = 1;
  int64 product_id = 2;
  int32 quantity = 3;
  string idempotency_key = 4;
}

message CreateOrderResponse {
  Order order = 1;
}

message GetOrderRequest {
  int64 id = 1;
}

message GetOrderResponse {
  Order order = 1;
}

message WatchOrderStatusRequest {
  int64 order_id = 1;
}

message OrderStatusEvent {
  int64 order_id = 1;
  OrderStatus status = 2;
  string message = 3;
}

service OrderService {
  rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);
  rpc GetOrder(GetOrderRequest) returns (GetOrderResponse);
  rpc WatchOrderStatus(WatchOrderStatusRequest) returns (stream OrderStatusEvent);
}
```

---

## 三、生成代码命令

```bash
protoc --go_out=. --go_opt=paths=source_relative \
  --go-grpc_out=. --go-grpc_opt=paths=source_relative \
  api/proto/order/v1/order.proto
```

建议后续放进 Makefile：

```makefile
gen:
	protoc --go_out=. --go_opt=paths=source_relative --go-grpc_out=. --go-grpc_opt=paths=source_relative api/proto/**/*.proto
```

---

## 四、常见问题

- 金额使用 double：可能有精度问题。
- 所有服务共用一个巨大 proto 文件：后期维护困难。
- 字段没有版本规划：后续扩展容易破坏兼容性。
- 创建订单没有幂等键：客户端重试容易重复下单。

---

## 五、练习任务

1. 完成 `user.proto`。
2. 完成 `product.proto`。
3. 完成 `order.proto`。
4. 完成 `notification.proto`。
5. 生成 Go 代码并检查包名。

---

## 六、完成标准

- proto 能成功生成代码。
- 服务接口覆盖核心需求。
- 字段类型和编号设计合理。

---

## 七、完整操作步骤：按服务拆分 proto

项目实战阶段建议每个服务一个 proto 包。不要把所有 message 都放进一个巨大文件里，否则生成代码后包边界也会混乱。

先创建目录：

```powershell
mkdir .\proto\user\v1 -Force
mkdir .\proto\product\v1 -Force
mkdir .\proto\order\v1 -Force
```

准备 `buf.yaml`：

```yaml
version: v2
modules:
  - path: proto
lint:
  use:
    - STANDARD
breaking:
  use:
    - FILE
```

准备 `buf.gen.yaml`：

```yaml
version: v2
plugins:
  - remote: buf.build/protocolbuffers/go
    out: gen
    opt:
      - paths=source_relative
  - remote: buf.build/grpc/go
    out: gen
    opt:
      - paths=source_relative
```

然后按 user、product、order 的顺序写 proto。先写被依赖的服务，再写依赖它们的 order-service。

---

## 八、完整代码：三个核心 proto 文件

`proto/user/v1/user.proto`：

```proto
syntax = "proto3";

package user.v1;

option go_package = "example.com/grpc-shop/gen/user/v1;userv1";

service UserService {
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
}

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
}

message CreateUserResponse {
  User user = 1;
}

message GetUserRequest {
  int64 id = 1;
}

message GetUserResponse {
  User user = 1;
}
```

`proto/product/v1/product.proto`：

```proto
syntax = "proto3";

package product.v1;

option go_package = "example.com/grpc-shop/gen/product/v1;productv1";

service ProductService {
  rpc GetProduct(GetProductRequest) returns (GetProductResponse);
  rpc DeductStock(DeductStockRequest) returns (DeductStockResponse);
}

message Product {
  int64 id = 1;
  string name = 2;
  int64 price_cents = 3;
  int32 stock = 4;
}

message GetProductRequest {
  int64 id = 1;
}

message GetProductResponse {
  Product product = 1;
}

message DeductStockRequest {
  int64 product_id = 1;
  int32 quantity = 2;
  string request_id = 3;
}

message DeductStockResponse {
  Product product = 1;
}
```

`proto/order/v1/order.proto`：

```proto
syntax = "proto3";

package order.v1;

option go_package = "example.com/grpc-shop/gen/order/v1;orderv1";

service OrderService {
  rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);
  rpc GetOrder(GetOrderRequest) returns (GetOrderResponse);
  rpc WatchOrderStatus(WatchOrderStatusRequest) returns (stream OrderStatusEvent);
}

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_CREATED = 1;
  ORDER_STATUS_PAID = 2;
  ORDER_STATUS_SHIPPED = 3;
  ORDER_STATUS_COMPLETED = 4;
  ORDER_STATUS_CANCELED = 5;
}

message Order {
  int64 id = 1;
  int64 user_id = 2;
  int64 product_id = 3;
  int32 quantity = 4;
  int64 total_cents = 5;
  OrderStatus status = 6;
}

message CreateOrderRequest {
  int64 user_id = 1;
  int64 product_id = 2;
  int32 quantity = 3;
  string request_id = 4;
}

message CreateOrderResponse {
  Order order = 1;
}

message GetOrderRequest {
  int64 id = 1;
}

message GetOrderResponse {
  Order order = 1;
}

message WatchOrderStatusRequest {
  int64 order_id = 1;
}

message OrderStatusEvent {
  int64 order_id = 1;
  OrderStatus status = 2;
  string message = 3;
}
```

---

## 九、运行命令与预期输出

格式检查：

```powershell
buf format -w
buf lint
```

预期输出为空，表示没有 lint 错误。

生成 Go 代码：

```powershell
buf generate
```

检查生成结果：

```powershell
Get-ChildItem .\gen -Recurse -Filter *.go
```

预期能看到：

```text
gen\user\v1\user.pb.go
gen\user\v1\user_grpc.pb.go
gen\product\v1\product.pb.go
gen\product\v1\product_grpc.pb.go
gen\order\v1\order.pb.go
gen\order\v1\order_grpc.pb.go
```

如果你暂时没有安装 buf，也可以使用 `protoc`，但项目实战阶段更推荐 buf，因为它能统一 lint、生成和 breaking change 检查。

---

## 十、常见错误排查

- `go_package` 不正确：生成代码 import 路径会错，后续 Go 文件无法引用。
- enum 第一个值不是 `UNSPECIFIED`：proto3 默认零值会变得含义不清。
- 金额使用 `double`：金额要用整数最小货币单位，例如 `price_cents`。
- 请求里缺少 `request_id`：后续讨论重试和幂等时没有抓手。
- 字段编号反复修改：已经发布的字段编号不要随意复用或改变含义。
- proto 包名和 Go 包名混乱：`package order.v1` 和 `orderv1` 可以不同，但要有清晰约定。

---

## 十一、教程闭环检查

本篇的完整操作步骤是：创建 proto 目录、编写 buf 配置、分别定义 user/product/order 契约、执行 lint 和 generate。完整代码包含三个可生成的 proto 文件。运行命令包含 `buf format`、`buf lint`、`buf generate` 和生成文件检查。预期输出是 gen 目录下出现 pb.go 与 grpc.pb.go。常见错误排查覆盖 go_package、enum、金额、幂等键和字段编号。练习任务是增加 `CancelOrder` 接口但不破坏已有字段。完成标准是：proto 能生成，Go 代码能 import，服务边界与需求文档一致。
