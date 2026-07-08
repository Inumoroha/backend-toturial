# 13. 管理接口契约和版本演进

本节目标：学完后你能设计更稳定的 JSON 请求响应，并知道字段变更时如何兼容旧客户端。

简短引入：JSON 字段不是随手起的变量名，而是客户端、服务端、测试、文档共同依赖的接口契约。后端工程师要学会让接口能演进，但不轻易破坏已有调用方。

## 一、为什么需要它

真实项目不会停在第一个版本。用户接口可能今天只有 `name`，下个月增加 `avatar_url`，再后来要废弃 `nickname`。如果没有契约意识，每次改字段都可能让客户端崩掉。

接口演进通常遵循两个方向：

- 增加字段尽量兼容。
- 删除或改名字段要谨慎迁移。

```text
已经上线的 JSON 字段名，默认当成公共契约处理。
```

## 二、基本用法

Windows PowerShell：

```powershell
mkdir json-contract
cd json-contract
go mod init json-contract
notepad main.go
go run .
```

Linux/macOS：

```bash
mkdir json-contract
cd json-contract
go mod init json-contract
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

type UserResponseV1 struct {
	ID   int    `json:"id"`
	Name string `json:"name"`
}

type UserResponseV2 struct {
	ID        int    `json:"id"`
	Name      string `json:"name"`
	AvatarURL string `json:"avatar_url,omitempty"`
}

func main() {
	resp := UserResponseV2{
		ID:        1001,
		Name:      "Alice",
		AvatarURL: "https://example.com/a.png",
	}

	data, err := json.Marshal(resp)
	if err != nil {
		panic(err)
	}

	fmt.Println(string(data))
}
```

## 三、关键参数/语法/代码结构

新增响应字段通常比较安全，因为旧客户端会忽略不认识的字段。但新增请求必填字段就不安全，因为旧客户端不会传。

字段改名本质上是删除旧字段再新增新字段。真实项目中通常要经历过渡期，比如同时支持 `nickname` 和 `display_name`，记录旧字段使用情况，确认客户端升级后再移除。

`omitempty` 可以让新字段没有值时不输出，但不能解决所有兼容问题。它只是输出策略，不是版本策略。

判断一次变更是否兼容，可以先问三个问题：

```text
旧客户端不改代码还能不能正常调用？
旧数据不迁移还能不能被新代码读取？
新服务返回的数据旧客户端能不能忽略或降级？
```

如果三个问题里有任何一个答案是否定的，就不要把它当成普通小改动处理。它需要版本、迁移或灰度计划。

## 四、真实后端场景示例

假设用户资料从 `nickname` 演进到 `display_name`，可以先兼容两者：

```go
type UpdateProfileRequest struct {
	Nickname    *string `json:"nickname"`
	DisplayName *string `json:"display_name"`
}

func normalizeDisplayName(req UpdateProfileRequest) *string {
	if req.DisplayName != nil {
		return req.DisplayName
	}
	return req.Nickname
}
```

这个阶段要在服务端日志或指标里统计旧字段使用量。等旧客户端升级完成，再移除 `nickname`。

如果字段已经落库，迁移要使用迁移工具。比如新增 `display_name` 列、回填历史数据、双写一段时间、切读新列，最后删除旧列。每一步都要能回滚。

更稳的字段改名流程通常是：

```text
新增 display_name 字段和数据库列
写入时同时写 nickname 和 display_name
读取时优先读 display_name，缺失时回退 nickname
观察旧字段使用量
通知客户端切换新字段
停止写旧字段
确认无依赖后删除旧字段和旧列
```

这个流程看起来慢，但能把线上风险拆小。真实项目中，接口契约和数据库结构经常是一起演进的，不能只改 Go 结构体标签。

## 五、注意点

接口版本可以放在 URL，例如 `/v1/users`；也可以放在 Header。小团队和内部系统常用 URL 版本，直观好排查。无论哪种方式，都要有明确的废弃计划。

不要为了省事在同一个字段里塞多种含义。比如 `status` 一会儿表示账号状态，一会儿表示审核状态，会让客户端和数据库查询都变混乱。

接口契约最好配合测试。即使没有完整 OpenAPI，也可以为关键响应写 JSON 快照测试或结构体字段测试。

关键接口建议至少做两类测试：一类测试请求 JSON 能否被正确解析，另一类测试响应 JSON 是否包含约定字段。这样改结构体标签、加 `omitempty`、调整字段类型时，测试会及时提醒你。

对外接口还应该写变更记录。记录新增了什么字段、什么时候废弃旧字段、旧字段预计什么时候删除。没有变更记录，调用方只能靠猜。

## 六、常见误区

误区一：觉得 JSON 字段名只是后端内部实现。字段名一旦被客户端使用，就是接口契约。

误区二：直接删除旧字段。老版本 App、脚本、第三方调用方可能还在使用。

误区三：新增请求必填字段不做版本处理。旧客户端请求会突然失败。

误区四：数据库迁移和接口发布没有顺序。字段还没迁移就上线读新字段，可能造成线上错误。

## 七、本节达标标准

- 能解释 JSON 字段为什么属于接口契约。
- 能判断新增响应字段和新增请求必填字段的兼容性差异。
- 能设计字段改名的过渡方案。
- 能说明数据库字段迁移要有顺序和回滚。
- 能为关键接口设计基本的契约测试思路。
