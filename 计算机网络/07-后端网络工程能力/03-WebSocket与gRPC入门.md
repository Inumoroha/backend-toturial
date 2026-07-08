# 03-WebSocket 与 gRPC 入门

## 学习目标

理解 WebSocket 和 gRPC 在后端系统中的位置，知道它们分别解决什么问题，以及与 HTTP/TCP 的关系。

## 一、WebSocket 解决什么问题

HTTP 请求响应模型适合客户端主动请求。但有些场景需要服务端主动推送：

- 即时聊天。
- 在线协作。
- 实时通知。
- 实时行情。

WebSocket 在一次 HTTP Upgrade 后建立全双工长连接。

## 二、WebSocket 与 HTTP 的关系

WebSocket 握手从 HTTP 开始：

```http
GET /ws HTTP/1.1
Upgrade: websocket
Connection: Upgrade
```

握手成功后，连接升级为 WebSocket 协议，双方可以持续互发消息。

## 三、WebSocket 后端注意点

- 连接生命周期管理。
- 心跳检测。
- 断线重连。
- 消息广播。
- 写超时。
- 单连接发送队列。
- 连接数和内存控制。

长连接会占用服务端资源，不能只考虑请求 QPS。

## 四、gRPC 解决什么问题

gRPC 常用于服务间通信。

特点：

- 基于 HTTP/2。
- 使用 Protobuf 定义接口和消息。
- 支持一元调用和流式调用。
- 自动生成客户端和服务端代码。
- 适合内部 RPC。

## 五、HTTP/2 对 gRPC 的意义

HTTP/2 提供：

- 多路复用。
- 二进制帧。
- Header 压缩。
- 流控制。

gRPC 在其上实现 RPC 语义。

## 六、一个 proto 示例

```proto
syntax = "proto3";

package user;

option go_package = "example.com/userpb";

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

## 七、WebSocket 和 gRPC 怎么选

| 场景 | 推荐 |
| --- | --- |
| 浏览器实时聊天 | WebSocket |
| 服务间调用 | gRPC |
| 对外 REST API | HTTP JSON |
| 服务端持续推送给浏览器 | WebSocket 或 SSE |
| 内部高性能强类型 RPC | gRPC |

## 八、Go 后端关联

WebSocket 常用库：

- `github.com/gorilla/websocket`
- `nhooyr.io/websocket`

gRPC 常用库：

- `google.golang.org/grpc`
- `google.golang.org/protobuf`

重点不是先背 API，而是理解：

- WebSocket 是长连接。
- gRPC 是 HTTP/2 上的 RPC。
- 二者都必须认真处理超时、取消、连接生命周期和流控。

## 九、练习题

1. WebSocket 为什么适合聊天？
2. WebSocket 和普通 HTTP 请求有什么关系？
3. gRPC 为什么适合服务间通信？
4. gRPC 和 HTTP/2 有什么关系？

## 十、验收标准

你能解释 WebSocket 与 gRPC 的核心用途，并能说明它们在 Go 后端架构中通常放在哪些位置。

