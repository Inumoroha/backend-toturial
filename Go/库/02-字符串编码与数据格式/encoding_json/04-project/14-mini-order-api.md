# 14. 小项目：订单接口中的 JSON 实战

本节目标：学完后你能写一个小型订单创建接口，把 JSON 解析、校验、响应、错误处理和生产边界串起来。

简短引入：前面的章节分别讲了 JSON 编解码、标签、请求体、错误处理、数字精度和事务意识。本节用一个订单创建接口把它们放到同一个流程里。

## 一、为什么需要它

订单接口是后端项目中很典型的 JSON 场景。它既有请求解析，也有金额、库存、状态和数据库写入边界。即使这个小项目不真的连接数据库，也要按真实项目的方式预留结构。

一个保守的订单创建流程通常是：

1. 限制请求体大小。
2. 严格解析 JSON。
3. 校验业务字段。
4. 调用业务层创建订单。
5. 在业务层用事务写订单、库存流水和日志。
6. 返回稳定 JSON 响应。

```text
handler 负责协议边界，service 负责业务规则，repository 负责数据库访问。
```

## 二、基本用法

Windows PowerShell：

```powershell
mkdir mini-order-api
cd mini-order-api
go mod init mini-order-api
notepad main.go
go run .
```

Linux/macOS：

```bash
mkdir mini-order-api
cd mini-order-api
go mod init mini-order-api
nano main.go
go run .
```

`main.go`：

```go
package main

import (
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"log"
	"net/http"
	"strings"
	"time"
)

type CreateOrderRequest struct {
	UserID     int    `json:"user_id"`
	ProductID  int    `json:"product_id"`
	Quantity   int    `json:"quantity"`
	AmountCent int64  `json:"amount_cent"`
	Remark     string `json:"remark,omitempty"`
}

type CreateOrderResponse struct {
	OrderNo    string `json:"order_no"`
	Status     string `json:"status"`
	AmountCent int64  `json:"amount_cent"`
	CreatedAt  string `json:"created_at"`
}

type APIResponse struct {
	Code    string `json:"code"`
	Message string `json:"message"`
	Data    any    `json:"data,omitempty"`
}

func main() {
	http.HandleFunc("/orders", createOrderHandler)

	log.Println("server listening on http://localhost:8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}

func createOrderHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		writeJSON(w, http.StatusMethodNotAllowed, APIResponse{"METHOD_NOT_ALLOWED", "method not allowed", nil})
		return
	}

	r.Body = http.MaxBytesReader(w, r.Body, 1<<20)
	defer r.Body.Close()

	var req CreateOrderRequest
	if err := decodeStrictJSON(r.Body, &req); err != nil {
		log.Println("decode create order:", err)
		writeJSON(w, http.StatusBadRequest, APIResponse{"BAD_JSON", "invalid json body", nil})
		return
	}

	if err := validateCreateOrder(req); err != nil {
		writeJSON(w, http.StatusBadRequest, APIResponse{"INVALID_ARGUMENT", err.Error(), nil})
		return
	}

	resp, err := createOrder(req)
	if err != nil {
		log.Println("create order:", err)
		writeJSON(w, http.StatusInternalServerError, APIResponse{"INTERNAL_ERROR", "internal server error", nil})
		return
	}

	writeJSON(w, http.StatusCreated, APIResponse{"OK", "order created", resp})
}

func decodeStrictJSON(r io.Reader, dst any) error {
	dec := json.NewDecoder(r)
	dec.DisallowUnknownFields()
	if err := dec.Decode(dst); err != nil {
		return err
	}
	if err := dec.Decode(&struct{}{}); !errors.Is(err, io.EOF) {
		return errors.New("body must contain only one json value")
	}
	return nil
}

func validateCreateOrder(req CreateOrderRequest) error {
	if req.UserID <= 0 {
		return errors.New("user_id must be positive")
	}
	if req.ProductID <= 0 {
		return errors.New("product_id must be positive")
	}
	if req.Quantity <= 0 {
		return errors.New("quantity must be positive")
	}
	if req.Quantity > 99 {
		return errors.New("quantity is too large")
	}
	if req.AmountCent <= 0 {
		return errors.New("amount_cent must be positive")
	}
	if len(strings.TrimSpace(req.Remark)) > 200 {
		return errors.New("remark is too long")
	}
	return nil
}

func createOrder(req CreateOrderRequest) (CreateOrderResponse, error) {
	// 真实项目中这里应进入 service 层：
	// 1. 查询商品和库存。
	// 2. 校验价格，不能只信任前端传入的 amount_cent。
	// 3. 开启数据库事务。
	// 4. 使用参数化查询写订单、扣库存、写库存流水。
	// 5. 提交失败时回滚。
	orderNo := fmt.Sprintf("NO%d", time.Now().UnixNano())
	return CreateOrderResponse{
		OrderNo:    orderNo,
		Status:     "created",
		AmountCent: req.AmountCent,
		CreatedAt:  time.Now().Format(time.RFC3339),
	}, nil
}

func writeJSON(w http.ResponseWriter, status int, body APIResponse) {
	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.WriteHeader(status)
	if err := json.NewEncoder(w).Encode(body); err != nil {
		log.Println("write json:", err)
	}
}
```

启动后测试。

Windows PowerShell：

```powershell
Invoke-RestMethod -Method Post -Uri http://localhost:8080/orders -ContentType "application/json" -Body '{"user_id":1001,"product_id":2001,"quantity":2,"amount_cent":9900}'
```

Linux/macOS：

```bash
curl -i -X POST http://localhost:8080/orders \
  -H 'Content-Type: application/json' \
  -d '{"user_id":1001,"product_id":2001,"quantity":2,"amount_cent":9900}'
```

## 三、关键参数/语法/代码结构

`http.MaxBytesReader` 限制请求体大小。真实项目中这是防御大请求的基础措施。

`DisallowUnknownFields` 拒绝未知字段。订单接口涉及金额和库存，解析策略应该保守一些。

`AmountCent int64` 用整数分表示金额。这里仍然不能完全信任客户端金额，真实项目应根据商品价格、优惠和活动规则在服务端重新计算。

`writeJSON` 统一响应格式。这样接口成功和失败都更容易被客户端处理。

`createOrder` 在示例中是模拟业务层。真实项目中不要把数据库 SQL 全部写在 handler 里。

## 四、真实后端场景示例

如果接入数据库，service 层伪流程通常是：

```text
开始事务
查询商品并锁定库存行
校验商品状态、价格和库存
插入订单
扣减库存
插入库存流水
插入订单事件日志
提交事务
失败则回滚
```

SQL 层要坚持参数化查询：

```go
// 示例形状，省略 db 初始化。
_, err := tx.ExecContext(ctx,
	"UPDATE products SET stock = stock - ? WHERE id = ? AND stock >= ?",
	req.Quantity, req.ProductID, req.Quantity,
)
```

这个 SQL 的重点是条件更新。它比“先查库存，再无条件扣减”更能抵抗并发超卖。当然，真实项目还要检查受影响行数。

数据库结构变更要通过迁移工具完成。比如给订单表新增 `remark` 字段，应先新增可空列，再上线写入逻辑，观察稳定后再考虑约束收紧。每一步都要能回滚。

## 五、注意点

不要相信前端传来的价格。前端可以展示价格，但最终金额应由服务端根据商品、优惠、活动和用户状态计算。

不要在事务里做慢操作，例如调用外部支付接口、发送邮件、长时间 HTTP 请求。事务要短，边界要清楚。

不要把完整请求体写进生产日志。订单备注、地址、手机号都可能是敏感信息。日志要记录 request id、用户 id、错误码等可排查信息。

索引不是越多越好。订单表通常会按 `user_id`、`order_no`、`created_at` 查询，但每个索引都会增加写入成本。建索引前要看真实查询路径。

## 六、常见误区

误区一：handler 里直接写所有业务和 SQL。代码很快变得难测、难复用、难回滚。

误区二：前端传多少金额就收多少。价格和优惠必须由服务端重新计算。

误区三：扣库存不检查受影响行数。并发下可能出现库存不足仍然下单成功。

误区四：事务中夹杂外部调用。外部调用慢或失败会拖长事务，增加锁等待。

误区五：迁移订单表时直接改字段类型。生产数据迁移要分步骤、可验证、可回滚。

## 七、本节达标标准

- 能写出一个可运行的订单创建 JSON 接口。
- 能限制请求体大小并启用严格 JSON 解析。
- 能对请求字段做基础业务校验。
- 能用统一 JSON 格式返回成功和失败响应。
- 能说明订单写库为什么需要参数化查询、事务边界和可回滚迁移。

