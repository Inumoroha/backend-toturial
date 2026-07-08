# 0. Go 网络编程实战总览

本阶段目标：把前面学过的网络协议落到 Go 代码中，掌握 TCP、UDP、HTTP Server、HTTP Client、Context、超时、连接池和优雅关闭。

---

## 一、本阶段学习顺序

```text
1_net包与TCP_Echo_Server
2_TCP客户端_Deadline与连接生命周期
3_UDP_Echo_Server与Client
4_HTTP_Server超时配置与中间件
5_HTTP_Client连接池_context与错误处理
6_优雅关闭_信号处理与资源释放
```

---

## 二、本阶段达标标准

学完本阶段后，你应该能够做到：

- 写出 TCP Echo Server 和 Client。
- 写出 UDP Echo Server 和 Client。
- 写出带超时配置的 HTTP Server。
- 写出复用连接池的 HTTP Client。
- 使用 context 控制请求取消。
- 使用优雅关闭处理进程退出。

