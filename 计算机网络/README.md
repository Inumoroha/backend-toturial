# Go 后端工程师计算机网络系统教程

这是一套面向 Go 后端工程师的计算机网络教程。它不是只为背面试题准备，而是按“理解原理 -> 命令实验 -> Go 代码 -> 工程排障 -> 项目实战”的顺序组织。

## 学习顺序

请按下面顺序学习：

新版教程已经按 `PostgreSQL` 目录的写法重构，优先阅读下面这种目录：

1. `第1阶段：环境搭建与网络直觉`
2. `第2阶段：网络分层与局域网基础`
3. `第3阶段：IP路由ARPICMP与NAT`
4. `第4阶段：TCP与UDP核心协议`
5. `第5阶段：DNS_HTTP_HTTPS与浏览器访问流程`
6. `第6阶段：Go网络编程实战`
7. `第7阶段：Linux网络与线上排障`
8. `第8阶段：后端网络工程能力`
9. `第9阶段：项目实战`

旧版阶段目录仍然保留，可作为补充资料：

1. `00-准备阶段`
2. `01-网络基础与分层模型`
3. `02-IP路由ARP与ICMP`
4. `03-TCP与UDP核心协议`
5. `04-DNS_HTTP_HTTPS`
6. `05-Go网络编程实战`
7. `06-Linux网络与线上排障`
8. `07-后端网络工程能力`
9. `08-项目实战阶段`

## 章节索引

### 00-准备阶段

1. `01-学习环境搭建.md`
2. `02-学习方法与实验规范.md`

### 01-网络基础与分层模型

1. `01-网络分层与数据传输全流程.md`
2. `02-MAC_IP_端口与局域网基础.md`
3. `03-OSI七层_LAN_WAN_MTU_MSS详解.md`

### 02-IP路由ARP与ICMP

1. `01-IP地址子网与路由表.md`
2. `02-ARP_ICMP_NAT实验教程.md`

### 03-TCP与UDP核心协议

1. `01-TCP从连接到可靠传输.md`
2. `02-TCP连接状态粘包与Go协议设计.md`
3. `03-UDP协议与Go实战.md`
4. `04-TCP进阶_Nagle_Keepalive_连接队列与拥塞控制.md`

### 04-DNS_HTTP_HTTPS

1. `01-DNS解析完整流程.md`
2. `02-HTTP协议与Go服务.md`
3. `03-HTTPS_TLS与证书实验.md`
4. `04-HTTP进阶_Cookie缓存_HTTP2_HTTP3.md`

### 05-Go网络编程实战

1. `01-Go_TCP编程教程.md`
2. `02-Go_HTTP客户端服务端最佳实践.md`
3. `03-Context超时连接池与可靠性.md`

### 06-Linux网络与线上排障

1. `01-Linux网络命令排障.md`
2. `02-Docker与端口网络问题排查.md`
3. `03-Go服务线上故障案例.md`
4. `04-防火墙网络命名空间与临时端口.md`

### 07-后端网络工程能力

1. `01-Nginx反向代理与负载均衡.md`
2. `02-重试限流熔断与幂等.md`
3. `03-WebSocket与gRPC入门.md`
4. `04-高性能网络编程入门.md`

### 08-项目实战阶段

1. `01-项目一TCP聊天室.md`
2. `02-项目二HTTP反向代理.md`
3. `03-项目三迷你API网关.md`
4. `04-项目四gRPC用户服务.md`
5. `05-毕业验收与复盘清单.md`
6. `06-项目落地方法_目录结构_迭代节奏.md`
7. `07-项目一TCP聊天室完整实现教程.md`
8. `08-项目二HTTP反向代理完整实现教程.md`

### 质量审查

- `教程质量审查与完善记录.md`

## PostgreSQL 风格新版教程索引

### 第1阶段：环境搭建与网络直觉

1. `0_环境搭建与网络直觉总览.md`
2. `1_安装 Go、Docker 与基础网络工具.md`
3. `2_认识本机网络信息_IP_网关_DNS_端口.md`
4. `3_使用 Docker 理解容器网络和端口映射.md`
5. `4_使用 Wireshark 与 tcpdump 做第一次抓包.md`
6. `5_启动一个 Go HTTP 服务并完成端到端访问.md`

### 第2阶段：网络分层与局域网基础

1. `0_网络分层与局域网基础总览.md`
2. `1_OSI七层与TCPIP四层模型.md`
3. `2_MAC地址_IP地址_端口号分别解决什么问题.md`
4. `3_局域网通信_交换机_路由器_网关.md`
5. `4_MTU_MSS与网络分片问题.md`

### 第3阶段：IP路由ARPICMP与NAT

1. `0_IP路由ARPICMP与NAT总览.md`
2. `1_IP地址_CIDR_私有地址与公网地址.md`
3. `2_路由表_默认网关与traceroute.md`
4. `3_ARP协议与局域网下一跳.md`
5. `4_ICMP_ping_traceroute与连通性判断.md`
6. `5_NAT原理与Docker容器访问外网.md`
7. `6_综合排障_从域名到目标IP的网络层检查.md`
8. `7_综合实验_路由_ARP_ICMP_NAT全链路观察.md`

### 第4阶段：TCP与UDP核心协议

1. `0_TCP与UDP核心协议总览.md`
2. `1_TCP连接建立_三次握手与抓包观察.md`
3. `2_TCP可靠传输_序列号_ACK_重传_窗口.md`
4. `3_TCP连接关闭_TIME_WAIT_CLOSE_WAIT.md`
5. `4_TCP字节流_粘包拆包与应用层协议设计.md`
6. `5_TCP进阶_Nagle_Keepalive_连接队列.md`
7. `6_UDP协议特点与Go_UDP_Echo实验.md`
8. `7_TCP_UDP综合对比与后端选型.md`
9. `8_TCP抓包实验_握手_挥手_状态变化.md`
10. `9_TCP长度前缀协议完整实现.md`

### 第5阶段：DNS_HTTP_HTTPS与浏览器访问流程

1. `0_DNS_HTTP_HTTPS总览.md`
2. `1_DNS解析流程与dig实验.md`
3. `2_HTTP请求响应报文与curl观察.md`
4. `3_HTTP方法状态码Header与后端语义.md`
5. `4_Cookie_Session_Token与缓存控制.md`
6. `5_HTTPS_TLS证书CA与SNI.md`
7. `6_HTTP1_HTTP2_HTTP3与QUIC.md`
8. `7_Go_HTTP_Client超时连接池与请求取消.md`
9. `8_浏览器访问HTTPS网站完整复盘.md`
10. `9_HTTPS自签证书与Go服务完整实验.md`
11. `10_Go_httptrace分阶段观察DNS_TCP_TLS_HTTP耗时.md`

### 第6阶段：Go网络编程实战

1. `0_Go网络编程实战总览.md`
2. `1_net包与TCP_Echo_Server.md`
3. `2_TCP客户端_Deadline与连接生命周期.md`
4. `3_UDP_Echo_Server与Client.md`
5. `4_HTTP_Server超时配置与中间件.md`
6. `5_HTTP_Client连接池_context与错误处理.md`
7. `6_优雅关闭_信号处理与资源释放.md`
8. `7_HTTP_Server完整项目_路由_中间件_优雅关闭.md`
9. `8_HTTP_Client封装_超时_重试_连接池_测试.md`

### 第7阶段：Linux网络与线上排障

1. `0_Linux网络与线上排障总览.md`
2. `1_排障总流程_DNS_路由_端口_HTTP.md`
3. `2_ss_lsof_tcpdump_curl_dig常用命令.md`
4. `3_Docker网络与端口映射排障.md`
5. `4_文件描述符_临时端口_TIME_WAIT_CLOSE_WAIT.md`
6. `5_防火墙_安全组_iptables_nftables.md`
7. `6_线上故障案例_超时_502_504_连接泄漏.md`
8. `7_可复现故障实验_refused_timeout_CLOSEWAIT_fd耗尽.md`
9. `8_一次HTTPS访问失败的四层排障剧本.md`
10. `9_网络命名空间_veth_bridge_NAT手把手实验.md`

### 第8阶段：后端网络工程能力

1. `0_后端网络工程能力总览.md`
2. `1_Nginx反向代理与七层转发.md`
3. `2_负载均衡策略与健康检查.md`
4. `3_API网关_路由_鉴权_日志_限流.md`
5. `4_超时_重试_熔断_幂等与故障放大.md`
6. `5_WebSocket长连接与心跳.md`
7. `6_gRPC_HTTP2_Protobuf与服务间通信.md`
8. `7_压测_pprof_连接池_背压与高性能直觉.md`
9. `8_Nginx双后端负载均衡与502_504完整实验.md`
10. `9_超时重试熔断限流完整Go实验.md`
11. `10_gRPC_grpcurl_拦截器_超时完整实验.md`

### 第9阶段：项目实战

1. `0_项目实战总览.md`
2. `1_TCP聊天室项目总览与协议设计.md`
3. `2_TCP聊天室_连接管理_广播_心跳.md`
4. `3_HTTP反向代理项目总览与单上游转发.md`
5. `4_HTTP反向代理_负载均衡_超时_日志.md`
6. `5_迷你API网关_路由_限流_健康检查.md`
7. `6_gRPC用户服务_proto_server_client.md`
8. `7_项目验收_排障_复盘清单.md`
9. `8_TCP聊天室完整落地_从目录到运行.md`
10. `9_HTTP反向代理完整落地_从目录到运行.md`
11. `10_迷你API网关完整落地_配置_路由_限流.md`
12. `11_gRPC用户服务完整落地_proto生成_server_client.md`

## 文件命名规则

- 文件夹前缀 `00`、`01`、`02` 表示阶段顺序。
- 文件名前缀 `01`、`02`、`03` 表示该阶段内的学习顺序。
- 每篇教程都尽量包含：学习目标、核心概念、操作实验、Go 后端关联、练习题、验收标准。

## 推荐节奏

如果你每天学习 1 到 2 小时，建议按 10 到 12 周完成。

如果你每天学习 3 到 4 小时，可以按 6 到 8 周完成，但不要跳过实验。网络知识只看文字很容易“看懂了但不会排障”，实验是必须的。

## 学习要求

每学完一篇教程，至少做三件事：

1. 用自己的话复述核心原理。
2. 在本机执行教程里的命令或 Go 示例。
3. 写一段笔记：这个知识点在 Go 后端开发中什么时候会遇到。

## 推荐环境

- Windows + WSL2 Ubuntu，或直接使用 Linux / macOS。
- Go 1.22 或更新版本。
- Docker Desktop。
- Wireshark。
- VS Code 或 GoLand。

## 总目标

学完后你应该能做到：

- 解释一次 HTTPS 请求从域名解析到服务响应的完整过程。
- 写出 TCP、UDP、HTTP、WebSocket、gRPC 的基础 Go 程序。
- 为 Go HTTP Client 和 Server 配置合理的超时与连接池。
- 使用 `curl`、`dig`、`ss`、`lsof`、`tcpdump`、`Wireshark` 排查常见网络问题。
- 理解 Nginx、负载均衡、API 网关、限流、熔断、重试、幂等在后端系统中的位置。
- 完成 TCP 聊天室、HTTP 反向代理、迷你 API 网关、gRPC 用户服务四个项目。
