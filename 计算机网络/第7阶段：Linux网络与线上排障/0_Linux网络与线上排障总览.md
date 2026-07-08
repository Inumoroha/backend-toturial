# 0. Linux 网络与线上排障总览

本阶段目标：掌握 Go 后端线上网络排障常用命令和分析路径，能处理端口不通、请求超时、502/504、TIME_WAIT、CLOSE_WAIT、fd 耗尽、Docker 端口映射等问题。

---

## 一、本阶段学习顺序

```text
1_排障总流程_DNS_路由_端口_HTTP
2_ss_lsof_tcpdump_curl_dig常用命令
3_Docker网络与端口映射排障
4_文件描述符_临时端口_TIME_WAIT_CLOSE_WAIT
5_防火墙_安全组_iptables_nftables
6_线上故障案例_超时_502_504_连接泄漏
```

---

## 二、本阶段达标标准

学完本阶段后，你应该能够做到：

- 用命令确认服务是否监听端口。
- 用 curl 区分连接失败、超时、HTTP 错误。
- 用 ss 查看连接状态。
- 用 lsof 查看端口对应进程。
- 用 tcpdump 证明请求有没有到达。
- 排查 Docker 端口映射问题。
- 分析 TIME_WAIT、CLOSE_WAIT、fd 耗尽。

