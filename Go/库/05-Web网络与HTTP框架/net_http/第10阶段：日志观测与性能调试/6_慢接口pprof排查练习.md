# 6. 慢接口 pprof 排查练习

本节目标：通过一个故意变慢的接口，练习“日志发现问题 -> 压测复现 -> pprof 观察”的完整路径。

---

## 一、准备慢接口

临时加入一个 CPU 密集接口：

```go
mux.HandleFunc("GET /cpu", func(w http.ResponseWriter, r *http.Request) {
	total := 0
	for i := 0; i < 100000000; i++ {
		total += i % 10
	}
	fmt.Fprintf(w, "total=%d\n", total)
})
```

这个接口没有业务意义，只是为了制造 CPU 压力。

---

## 二、用请求日志确认慢

访问：

```bash
curl -i http://localhost:8080/cpu
```

观察请求日志中的：

```text
path=/cpu
status=200
duration_ms=...
request_id=...
```

如果你的日志能看到这些字段，说明观测基础是可用的。

---

## 三、用 hey 复现压力

```bash
hey -n 100 -c 10 http://localhost:8080/cpu
```

观察：

- 平均耗时。
- P95。
- P99。
- 是否出现 500。

---

## 四、采集 CPU pprof

确保 pprof 服务启动在：

```text
localhost:6060
```

采集：

```bash
go tool pprof http://localhost:6060/debug/pprof/profile
```

采集期间再跑一次压测，让 CPU 压力出现。

进入 pprof 后可以输入：

```text
top
```

看哪些函数最耗 CPU。

---

## 五、练习目标

完成后你应该能说清楚：

```text
哪个接口慢？
慢到什么程度？
慢请求日志怎么定位？
压测如何复现？
pprof 看到的热点函数是什么？
```

真实项目中排查会更复杂，但这条路径是非常重要的起点。
