# 0. TCP 与 UDP 核心协议总览

本阶段目标：系统理解 TCP 和 UDP，能解释 TCP 三次握手、可靠传输、四次挥手、TIME_WAIT、CLOSE_WAIT、粘包拆包，并能用 Go 写出 TCP/UDP 小程序验证。

TCP/UDP 是 Go 后端工程师最重要的网络基础之一。HTTP、gRPC、数据库连接、Redis 连接、消息队列客户端，最终都离不开传输层。

---

## 一、本阶段学习顺序

```text
1_TCP连接建立_三次握手与抓包观察
2_TCP可靠传输_序列号_ACK_重传_窗口
3_TCP连接关闭_TIME_WAIT_CLOSE_WAIT
4_TCP字节流_粘包拆包与应用层协议设计
5_TCP进阶_Nagle_Keepalive_连接队列
6_UDP协议特点与Go_UDP_Echo实验
7_TCP_UDP综合对比与后端选型
```

---

## 二、本阶段达标标准

学完本阶段后，你应该能够做到：

- 用 tcpdump 抓到 TCP 三次握手。
- 解释 TCP 为什么可靠。
- 解释 TIME_WAIT 和 CLOSE_WAIT 的原因。
- 解释 TCP 为什么会有粘包拆包。
- 设计一个长度前缀协议。
- 写出 Go TCP Echo Server 和 UDP Echo Server。
- 根据业务场景判断使用 TCP、UDP、HTTP、WebSocket 还是 gRPC。

