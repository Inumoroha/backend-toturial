# 12. 处理动态 JSON 和 map 场景

本节目标：学完后你能判断什么时候使用 `map[string]any`，什么时候应该回到结构体。

简短引入：动态 JSON 很常见，例如配置、埋点属性、搜索过滤条件。但如果业务核心数据长期使用 `map[string]any`，代码会很快失去类型约束。

## 一、为什么需要它

结构体适合稳定契约，`map[string]any` 适合不稳定或半结构化数据。真实项目中，两者都需要，但边界要清楚。

适合 `map[string]any` 的场景：

- 日志附加字段。
- 埋点属性。
- 第三方回调的临时透传字段。
- 后台配置的扩展字段。

不适合长期用 `map[string]any` 的场景：

- 用户注册请求。
- 订单创建请求。
- 权限变更请求。
- 支付请求。

```text
核心业务接口优先使用结构体，动态字段才使用 map。
```

## 二、基本用法

Windows PowerShell：

```powershell
mkdir json-dynamic
cd json-dynamic
go mod init json-dynamic
notepad main.go
go run .
```

Linux/macOS：

```bash
mkdir json-dynamic
cd json-dynamic
go mod init json-dynamic
nano main.go
go run .
```

`main.go`：

```go
package main

import (
	"encoding/json"
	"fmt"
)

type AuditLog struct {
	Action string         `json:"action"`
	Actor  string         `json:"actor"`
	Extra  map[string]any `json:"extra"`
}

func main() {
	logEntry := AuditLog{
		Action: "update_role",
		Actor:  "admin-1001",
		Extra: map[string]any{
			"user_id": 1002,
			"from":    "viewer",
			"to":      "editor",
		},
	}

	data, err := json.Marshal(logEntry)
	if err != nil {
		panic(err)
	}

	fmt.Println(string(data))
}
```

## 三、关键参数/语法/代码结构

`map[string]any` 表示键是字符串，值可以是任意类型。JSON 对象解码进来后，值可能是 `string`、`float64`、`bool`、`nil`、`[]any` 或 `map[string]any`。

因为值是 `any`，使用前通常需要类型断言。断言失败会造成 panic，所以真实项目里要用安全写法。

```go
value, ok := data["name"].(string)
if !ok {
	return fmt.Errorf("name must be string")
}
```

如果动态 JSON 里有数字，默认会解码成 `float64`。涉及金额、大 ID、库存这类字段时，要么不要放进动态 map，要么使用 `Decoder.UseNumber` 后再显式转换。

动态字段也应该有边界。例如允许最多 20 个键、每个键最长 50 个字符、每个字符串值最长 500 个字符。否则日志、配置或扩展字段可能被异常请求撑得很大。

## 四、真实后端场景示例

审计日志可以使用固定字段加扩展字段。

```go
type AuditEvent struct {
	RequestID string         `json:"request_id"`
	Action    string         `json:"action"`
	ActorID   int            `json:"actor_id"`
	Extra     map[string]any `json:"extra,omitempty"`
}
```

固定字段用于查询和索引，扩展字段用于保存变化快的信息。比如权限变更日志里，`Action` 和 `ActorID` 应该是固定字段，`from_role`、`to_role` 可以放进 `Extra`。

如果审计日志要进入数据库，常见做法是固定字段单独建列，扩展字段放 JSON 列。要注意 JSON 列索引成本，不要给所有动态键都建索引。

后台配置也常见动态 JSON，但配置比日志更危险，因为配置会影响系统行为。比如短链接服务的风控配置可以长这样：

```json
{
  "max_links_per_day": 100,
  "blocked_domains": ["spam.example"],
  "enable_review": true
}
```

这类配置保存前要校验字段类型和范围。上线前最好支持预览、审核和版本回滚，避免一次错误配置影响生产流量。

## 五、注意点

动态 JSON 不代表不校验。你至少应该限制字段大小、嵌套深度、允许的键名范围，避免日志或配置被异常数据撑爆。

如果某个动态字段开始频繁参与查询，就应该考虑把它提升为正式字段，并通过迁移工具补齐历史数据。

配置类 JSON 修改要支持回滚。生产配置不要只在页面上改完就保存，最好有版本、审核和回滚记录。

对于审计日志，固定字段应该服务于查询：谁操作、操作了什么、什么时候操作、结果如何。扩展字段服务于补充上下文，不应该承担核心查询能力。

对于用户提交的动态字段，不要直接透传到第三方系统或 SQL 里。即使字段是动态的，也要在服务端做白名单或黑名单过滤。

## 六、常见误区

误区一：为了省事，所有请求都用 `map[string]any`。这样字段名写错、类型错都要到运行时才发现。

误区二：对 `any` 直接强转。类型不符会 panic，接口可能返回 500。

误区三：把动态 JSON 的所有键都建索引。索引会增加写入成本和存储成本，要按查询需求设计。

误区四：配置 JSON 没有版本和回滚。错误配置上线后很难快速恢复。

## 七、本节达标标准

- 能说出结构体和 `map[string]any` 的适用边界。
- 能安全地从 `map[string]any` 读取字段。
- 能解释动态 JSON 仍然需要校验。
- 能设计固定字段加扩展字段的审计日志结构。
- 能意识到 JSON 列索引和配置回滚的生产成本。
