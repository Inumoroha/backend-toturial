# 7. gRPC 用户服务：proto、server、client

本节目标：实现一个最小 gRPC 用户服务，完成 proto 定义、服务端、客户端和错误码处理。

---

## 一、proto 定义

```proto
syntax = "proto3";

package user.v1;

option go_package = "example.com/grpc-user/api/user/v1;userv1";

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
}

message GetUserRequest {
  int64 id = 1;
}

message GetUserResponse {
  int64 id = 1;
  string name = 2;
}
```

---

## 二、服务端逻辑

```go
func (s *Server) GetUser(ctx context.Context, req *userv1.GetUserRequest) (*userv1.GetUserResponse, error) {
    if req.Id <= 0 {
        return nil, status.Error(codes.InvalidArgument, "invalid id")
    }

    if req.Id != 1 {
        return nil, status.Error(codes.NotFound, "user not found")
    }

    return &userv1.GetUserResponse{Id: 1, Name: "alice"}, nil
}
```

---

## 三、客户端超时

```go
ctx, cancel := context.WithTimeout(context.Background(), time.Second)
defer cancel()

resp, err := client.GetUser(ctx, &userv1.GetUserRequest{Id: 1})
```

---

## 四、错误处理

```go
st, ok := status.FromError(err)
if ok {
    log.Println(st.Code(), st.Message())
}
```

---

## 补充落地步骤：proto 先稳定再写业务

gRPC 项目推荐顺序：

```text
1. 写 proto。
2. 生成 Go 代码。
3. 实现 server 空逻辑。
4. 写 client 调通。
5. 再补参数校验和业务逻辑。
```

不要一边改 proto 一边写大量业务代码。proto 是契约，字段编号和消息结构一旦被客户端使用，就要谨慎变更。

---

## 补充验收：错误码必须可观察

至少验证：

```text
参数为空 -> InvalidArgument。
用户不存在 -> NotFound。
服务不可用 -> Unavailable。
客户端超时 -> DeadlineExceeded。
```

client 里用：

```go
st, _ := status.FromError(err)
log.Println(st.Code(), st.Message())
```

如果所有错误都变成 `Unknown` 或 `Internal`，说明服务端错误处理还不够工程化。

---

## 补充兼容性：proto 字段不要随便改编号

Protobuf 的字段编号是线上兼容的关键：

```proto
message User {
  string id = 1;
  string name = 2;
}
```

后续不能把 `name = 2` 改成别的含义。要删除字段时，也应该保留编号或标记 reserved：

```proto
reserved 2;
reserved "name";
```

这能避免旧客户端和新服务端对同一个字段编号理解不一致。

---

## 补充验收：server 和 client 必须分开终端验证

终端 1：

```bash
go run ./cmd/server
```

终端 2：

```bash
go run ./cmd/client
```

至少验证：

```text
CreateUser 成功。
GetUser 成功。
GetUser 不存在 ID 返回 NotFound。
客户端设置 deadline 后能超时退出。
```

这能证明 proto、生成代码、server 注册、client 调用和错误处理都串起来了。

---

## 五、常见问题

### 1. proto 字段编号可以随便改吗？

不建议。字段编号是 Protobuf 兼容性的关键，已发布字段要谨慎修改。

### 2. gRPC 错误能不能直接返回普通 error？

可以返回 error，但推荐使用 status code 表达明确语义。

### 3. 客户端不设置 timeout 会怎样？

请求可能长时间等待，占用 goroutine 和连接资源。

---

## 六、本节达标标准

学完本节后，你应该能够做到：

- 定义 UserService proto。
- 实现 GetUser。
- 使用 context timeout。
- 使用 gRPC status code 表达错误。
- 用客户端调用服务。
