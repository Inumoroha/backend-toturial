# 3. pprof 基础

本节目标：学会在 HTTP 服务中接入 `net/http/pprof`，用于基础性能诊断。

---

## 一、pprof 能看什么

pprof 可以帮助分析：

- CPU 使用。
- 内存分配。
- goroutine 数量和堆栈。
- 阻塞情况。
- mutex 竞争。

它是 Go 服务定位性能问题的重要工具。

---

## 二、最小接入方式

只要导入：

```go
import _ "net/http/pprof"
```

并使用默认 mux 启动服务，就会注册调试路由。

但项目中我们通常使用自定义 mux，所以更推荐单独启动调试服务。

---

## 三、单独启动 pprof 服务

```go
go func() {
	log.Println("pprof listening on :6060")
	if err := http.ListenAndServe("localhost:6060", nil); err != nil {
		log.Printf("pprof server error: %v", err)
	}
}()
```

导入：

```go
import _ "net/http/pprof"
```

访问：

```text
http://localhost:6060/debug/pprof/
```

---

## 四、常用地址

```text
/debug/pprof/
/debug/pprof/goroutine
/debug/pprof/heap
/debug/pprof/profile
```

采集 30 秒 CPU：

```bash
go tool pprof http://localhost:6060/debug/pprof/profile
```

---

## 五、生产环境注意

pprof 暴露了很多内部信息，不应该直接公开到公网。

建议：

- 只监听 localhost。
- 放在内网。
- 通过鉴权或 VPN 访问。
- 不和公网业务端口混在一起。

---

## 六、本节检查点

请确认你能回答：

- pprof 可以分析哪些问题？
- 为什么 pprof 不应该公开到公网？
- 为什么建议单独启动调试端口？

