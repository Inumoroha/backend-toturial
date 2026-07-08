# 01. TrimSpace 与用户输入清洗

本节目标：你能在用户注册、搜索、提交表单时，先把输入两端多余空白清理掉，再进入后续业务逻辑。

简短引入：后端收到的字符串很少是完全干净的。用户复制邮箱时可能带空格，手机端输入昵称时可能多一个换行，后台导入 CSV 时字段前后也可能有制表符。`strings.TrimSpace` 是处理这类问题的第一步。

## 一、为什么需要它

可以理解为：`TrimSpace` 负责把“用户无意带进来的外层空白”去掉。

常见场景是：

- 注册邮箱：`" alice@example.com "` 应该按 `alice@example.com` 处理。
- 搜索关键词：`"  Go 后端  "` 不应该因为两端空格导致搜索失败。
- 管理后台录入订单号：换行和空格不能参与匹配。

```text
清洗输入不是为了相信输入，而是为了让后续校验面对更稳定的数据。
```

## 二、基本用法

新建 `main.go`：

```go
package main

import (
	"fmt"
	"strings"
)

func main() {
	rawEmail := "  alice@example.com \n"
	email := strings.TrimSpace(rawEmail)

	fmt.Printf("原始值: %q\n", rawEmail)
	fmt.Printf("清洗后: %q\n", email)
}
```

Windows PowerShell：

```powershell
go run main.go
```

Linux/macOS：

```bash
go run main.go
```

这个命令的作用是快速验证清洗逻辑。真实项目中，你通常不会在 `main` 里处理注册逻辑，而是把它放进 handler 或 service 的输入转换层。

## 三、关键代码结构

`strings.TrimSpace(rawEmail)`：

- 参数是原始字符串。
- 返回值是一个新的字符串结果。
- 它只处理字符串两端的空白，不会删除中间的空格。

例如：

```go
strings.TrimSpace("  Go 后端  ") // "Go 后端"
strings.TrimSpace("Go  后端")    // "Go  后端"
```

这很重要。搜索词中间的空格可能是用户真实意图，比如 `"Go 后端"`，不能随便删除。

## 四、真实后端场景示例

下面模拟注册接口里对邮箱和昵称做基础清洗：

```go
package main

import (
	"errors"
	"fmt"
	"strings"
)

type RegisterRequest struct {
	Email string
	Name  string
}

func NormalizeRegisterRequest(req RegisterRequest) (RegisterRequest, error) {
	req.Email = strings.TrimSpace(req.Email)
	req.Name = strings.TrimSpace(req.Name)

	if req.Email == "" {
		return req, errors.New("email is required")
	}
	if req.Name == "" {
		return req, errors.New("name is required")
	}
	return req, nil
}

func main() {
	req := RegisterRequest{
		Email: " alice@example.com \n",
		Name:  " Alice ",
	}

	clean, err := NormalizeRegisterRequest(req)
	if err != nil {
		fmt.Println("注册失败:", err)
		return
	}
	fmt.Printf("可入库数据: %+v\n", clean)
}
```

真实项目中通常会先清洗，再校验，再进入数据库层。清洗不等于校验，邮箱格式、长度限制、唯一性仍然要单独处理。

## 五、注意点

`TrimSpace` 适合处理两端空白，不适合做下面这些事：

- 删除所有空格。
- 判断邮箱格式是否合法。
- 防 SQL 注入。
- 防 XSS。

```text
用户输入不能直接拼进 SQL 字符串，TrimSpace 不能替代参数化查询。
```

如果需要写入数据库，应使用参数化查询或 ORM 提供的参数绑定。清洗后的字符串仍然是不可信输入。

## 六、常见误区

误区一：以为 `TrimSpace` 会删除所有空格。

它只删除两端空白。中间空格会保留，这是正确行为。

误区二：清洗后就直接入库。

清洗只是第一步。真实项目中还要做长度、格式、权限、唯一性等校验。

误区三：搜索词清洗后不限制长度。

搜索接口如果不限制输入长度，可能造成慢查询或日志污染。常见做法是清洗后限制最大长度。

## 七、本节达标标准

- 能使用 `strings.TrimSpace` 清理用户输入两端空白。
- 能说明清洗和校验的区别。
- 能知道清洗后的输入仍然不能直接拼 SQL。
- 能把清洗逻辑放在请求进入业务逻辑之前。

