# 08. 用 Encoder 和 Decoder 处理流式 JSON

本节目标：学完后你能用 `Encoder` 和 `Decoder` 处理大 JSON 或一行一条的 JSON 数据，避免一次性读入全部内容。

简短引入：普通 `Marshal` 和 `Unmarshal` 适合小对象。日志导入、订单批处理、消息文件迁移这类场景，数据可能很大，更适合用流式处理。

## 一、为什么需要它

可以把流式处理理解为“边读边处理”。如果一次性把 500MB 的 JSON 文件读进内存，服务可能变慢甚至崩溃。流式处理可以把内存压力控制在单条数据附近。

常见场景是：

- 导入历史订单。
- 读取审计日志。
- 从消息队列消费一条条 JSON 事件。
- 输出大批量报表结果。

```text
大数据量 JSON 不要默认一次性读入内存。
```

## 二、基本用法

Windows PowerShell：

```powershell
mkdir json-stream
cd json-stream
go mod init json-stream
notepad main.go
go run .
```

Linux/macOS：

```bash
mkdir json-stream
cd json-stream
go mod init json-stream
nano main.go
go run .
```

`main.go`：

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"strings"
)

type LogEntry struct {
	Level   string `json:"level"`
	Message string `json:"message"`
}

func main() {
	input := `{"level":"info","message":"order created"}
{"level":"warn","message":"stock is low"}
{"level":"info","message":"payment received"}`

	dec := json.NewDecoder(strings.NewReader(input))
	for {
		var entry LogEntry
		if err := dec.Decode(&entry); err != nil {
			if err == io.EOF {
				break
			}
			panic(err)
		}
		fmt.Printf("[%s] %s\n", entry.Level, entry.Message)
	}
}
```

这个示例使用一行一个 JSON 对象的格式，也常叫 JSON Lines。

## 三、关键参数/语法/代码结构

`json.NewDecoder(reader)` 从 `io.Reader` 读取数据，适合文件、网络连接、HTTP 请求体。

`dec.Decode(&entry)` 每次解码一个 JSON 值。对于 JSON Lines，可以循环读取，直到遇到 `io.EOF`。

`json.NewEncoder(writer).Encode(value)` 会把一个值编码到 `io.Writer`，并追加换行。常见用途是写 HTTP 响应、日志文件或导出文件。

## 四、真实后端场景示例

订单导入时，可以逐条读取订单，逐条校验，再批量写入数据库。

```go
type ImportOrder struct {
	OrderNo string `json:"order_no"`
	UserID  int    `json:"user_id"`
	Qty     int    `json:"qty"`
}

func handleOrder(order ImportOrder) error {
	if order.OrderNo == "" || order.UserID <= 0 || order.Qty <= 0 {
		return fmt.Errorf("invalid order: %s", order.OrderNo)
	}
	// 真实项目中在这里进入 service 层，使用参数化查询或批量插入。
	return nil
}
```

批量导入数据库时，不要一条数据开一个事务，也不要整个巨大文件放在一个事务里。常见做法是按批次提交，例如每 500 条一个事务。这样失败时可回滚当前批次，也不会让事务持续太久。

导入任务还应该记录进度。比如总共读取了多少行、成功多少行、失败多少行、第一条失败原因是什么。这样导入失败后，用户或运维人员才知道是数据问题、格式问题，还是数据库问题。

常见的批处理结构可以理解为：

```text
打开文件或读取请求流
循环 Decode 每条 JSON
校验单条数据
加入批次
批次满了就开启事务写入
提交成功后记录进度
失败则回滚当前批次并返回错误报告
```

## 五、注意点

`Decoder.More()` 更常用于数组读取场景。处理 JSON Lines 文件时，更直接的做法是循环 `Decode`，直到遇到 `io.EOF`。

导入接口一定要有大小限制、超时和错误报告。否则一次导入失败后，用户不知道哪一行错了，也无法安全重试。

大批量写库要关注索引成本。索引越多，写入越慢。迁移或导入前要评估是否需要临时停用某些非关键索引，完成后再重建，但这类操作必须有回滚方案。

如果导入来源是 HTTP 上传，不建议让普通接口长时间等待到全部导入完成。真实项目中通常会把文件保存后创建后台任务，接口先返回任务 ID，前端再查询任务状态。这比让一个 HTTP 请求挂几分钟更可靠。

如果导入数据会影响库存、金额或权限，要先在测试环境用一小批真实脱敏数据演练。导入脚本也要支持“只校验不写入”的 dry run 模式，降低生产风险。

## 六、常见误区

误区一：用 `io.ReadAll` 读取超大请求体。小测试能跑，生产环境可能耗尽内存。

误区二：导入文件遇到一条错就静默跳过。真实项目中应该记录错误行和原因，让数据可追踪。

误区三：批量写库不设计事务边界。失败后可能出现一半成功、一半失败的数据。

误区四：把导入当成普通请求处理，不设置超时。长时间连接容易拖垮服务。

## 七、本节达标标准

- 能用 `json.Decoder` 逐条读取 JSON 数据。
- 能用 `json.Encoder` 向响应或文件写 JSON。
- 能解释为什么大 JSON 不适合一次性读入内存。
- 能说出批量导入时事务分批和错误报告的重要性。
- 能意识到索引会影响大批量写入成本。
