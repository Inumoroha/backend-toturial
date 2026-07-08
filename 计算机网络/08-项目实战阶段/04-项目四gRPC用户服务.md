# 04-项目四：gRPC 用户服务

## 项目目标

实现一个简单的 gRPC 用户服务，掌握 Protobuf、gRPC Server、gRPC Client、超时、错误码和服务间调用。

## 一、功能要求

基础功能：

- 定义 UserService。
- 实现 `GetUser`。
- 实现 `CreateUser`。
- 编写 gRPC Client 调用服务。
- 为请求设置 timeout。
- 使用 gRPC status code 返回错误。

进阶功能：

- Server interceptor 记录日志。
- Client interceptor 添加 request id。
- 健康检查。
- TLS。
- 流式接口。

## 二、proto 设计

```proto
syntax = "proto3";

package user.v1;

option go_package = "example.com/grpc-user/api/user/v1;userv1";

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
}

message GetUserRequest {
  int64 id = 1;
}

message GetUserResponse {
  int64 id = 1;
  string name = 2;
  string email = 3;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
}

message CreateUserResponse {
  int64 id = 1;
}
```

## 三、服务端结构

```go
type UserServer struct {
    userv1.UnimplementedUserServiceServer
    mu    sync.Mutex
    next  int64
    users map[int64]*User
}
```

## 四、错误码

gRPC 不建议随便返回普通 error，而应该使用 status code：

```go
return nil, status.Error(codes.NotFound, "user not found")
```

常见 code：

- `InvalidArgument`
- `NotFound`
- `AlreadyExists`
- `DeadlineExceeded`
- `Unavailable`
- `Internal`

## 五、客户端超时

```go
ctx, cancel := context.WithTimeout(context.Background(), time.Second)
defer cancel()

resp, err := client.GetUser(ctx, &userv1.GetUserRequest{Id: 1})
if err != nil {
    st, ok := status.FromError(err)
    if ok {
        log.Println(st.Code(), st.Message())
    }
    return
}
```

## 六、服务端日志拦截器

```go
func loggingInterceptor(
    ctx context.Context,
    req any,
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (any, error) {
    start := time.Now()
    resp, err := handler(ctx, req)
    log.Printf("method=%s cost=%s err=%v", info.FullMethod, time.Since(start), err)
    return resp, err
}
```

注册：

```go
grpc.NewServer(grpc.UnaryInterceptor(loggingInterceptor))
```

## 七、测试方式

推荐使用：

```bash
grpcurl -plaintext localhost:9000 list
grpcurl -plaintext localhost:9000 user.v1.UserService/GetUser
```

也可以编写 Go Client 测试。

## 八、gRPC 与 HTTP JSON 对比

| 维度 | HTTP JSON | gRPC |
| --- | --- | --- |
| 接口描述 | 文档约定 | proto 强约束 |
| 数据格式 | JSON 文本 | Protobuf 二进制 |
| 浏览器直接调用 | 方便 | 不直接方便 |
| 服务间通信 | 可以 | 很适合 |
| 流式能力 | 较弱 | 原生支持 |

## 九、验收标准

必须完成：

- proto 能生成 Go 代码。
- Server 能启动并注册 UserService。
- Client 能调用 GetUser 和 CreateUser。
- 找不到用户时返回 `NotFound`。
- 客户端调用设置 timeout。
- 服务端日志记录方法名和耗时。

加分项：

- TLS。
- 健康检查。
- 一元拦截器和流式拦截器。
- Docker Compose 启动服务。

