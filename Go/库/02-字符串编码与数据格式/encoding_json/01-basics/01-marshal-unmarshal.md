# 01. 把结构体转换成 JSON 响应

本节目标：学完后你能把后端里的用户结构体转换成 JSON，并把 JSON 请求数据解析回结构体。

简短引入：后端接口的日常工作，常常就是把数据库或业务层得到的 Go 结构体，转换成前端能理解的 JSON；也会把前端提交的 JSON，转换成 Go 结构体继续处理。

## 一、为什么需要它

可以把 JSON 理解为服务之间传递数据的通用格式。Go 代码里更适合用结构体，因为结构体有字段类型、可读性和编译期检查；网络上传输时更适合用 JSON，因为前端、网关、日志系统都能处理。

真实项目中常见流程是：

1. 查询用户。
2. 得到 `User` 结构体。
3. 转成 JSON。
4. 写入 HTTP 响应。

反过来，创建用户时常见流程是：

1. 前端提交 JSON。
2. 后端解析成 `CreateUserRequest`。
3. 校验字段。
4. 进入业务层和数据库层。

```text
JSON 解析成功，不代表业务数据合法；解析只是第一步。
```

## 二、基本用法

Windows PowerShell：

```powershell
mkdir json-basic
cd json-basic
go mod init json-basic
notepad main.go
go run .
```

Linux/macOS：

```bash
mkdir json-basic
cd json-basic
go mod init json-basic
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

type User struct {
	ID    int
	Name  string
	Email string
}

func main() {
	user := User{
		ID:    1001,
		Name:  "Alice",
		Email: "alice@example.com",
	}

	data, err := json.Marshal(user)
	if err != nil {
		panic(err)
	}
	fmt.Println(string(data))

	var decoded User
	err = json.Unmarshal(data, &decoded)
	if err != nil {
		panic(err)
	}
	fmt.Printf("%+v\n", decoded)
}
```

运行后你会看到类似输出：

```text
{"ID":1001,"Name":"Alice","Email":"alice@example.com"}
{ID:1001 Name:Alice Email:alice@example.com}
```

## 三、关键参数/语法/代码结构

`json.Marshal(user)` 的作用是把 Go 值编码为 JSON 字节切片。真实 HTTP 响应里可以直接把这个字节切片写给客户端。

`json.Unmarshal(data, &decoded)` 的作用是把 JSON 字节切片解码到 Go 变量里。这里必须传指针，因为函数需要修改 `decoded` 的内容。

结构体字段必须导出，也就是首字母大写。`encoding/json` 只能访问导出字段，所以 `Name` 可以被编码，`name` 不会被编码。

`data` 的类型是 `[]byte`。打印时用 `string(data)` 是为了让人读得懂；真实项目里写响应时通常不需要先转字符串。

## 四、真实后端场景示例

下面模拟一个用户详情响应。注意这里把数据库模型和响应模型分开了，真实项目中这样做更稳。

```go
package main

import (
	"encoding/json"
	"fmt"
)

type UserModel struct {
	ID           int
	Name         string
	Email        string
	PasswordHash string
}

type UserResponse struct {
	ID    int
	Name  string
	Email string
}

func main() {
	model := UserModel{
		ID:           1001,
		Name:         "Alice",
		Email:        "alice@example.com",
		PasswordHash: "hash-value",
	}

	resp := UserResponse{
		ID:    model.ID,
		Name:  model.Name,
		Email: model.Email,
	}

	data, err := json.Marshal(resp)
	if err != nil {
		panic(err)
	}

	fmt.Println(string(data))
}
```

这个例子里的重点不是代码长短，而是边界意识：数据库模型里有 `PasswordHash`，接口响应里不应该返回它。

## 五、注意点

`Marshal` 可能返回错误。常见原因包括结构体里有函数、通道、循环引用，或者浮点数是 `NaN`、`Inf`。后端项目中不要忽略这个错误。

`Unmarshal` 只说明 JSON 语法和基本类型能不能对上，不负责业务校验。例如年龄是 `-1`，JSON 解析可能成功，但业务上不合法。

真实项目中，JSON 解析后通常还要做：

- 必填字段校验。
- 长度限制。
- 枚举值校验。
- 权限判断。
- 数据库写入前的参数化查询。

## 六、常见误区

误区一：把 `json.Unmarshal(data, decoded)` 写成非指针。这样函数无法修改目标变量，会返回错误。

误区二：认为结构体字段小写也能输出。小写字段在包外不可见，标准库不会编码它。

误区三：直接把数据库模型返回给前端。这样很容易泄露密码哈希、内部状态、删除标记等字段。

误区四：解析成功就直接入库。真实项目里入库前还需要业务校验，并且 SQL 必须使用参数化查询。

```text
用户输入不能直接拼进 SQL 字符串。
```

## 七、本节达标标准

- 能使用 `json.Marshal` 把结构体转换成 JSON。
- 能使用 `json.Unmarshal` 把 JSON 解析到结构体。
- 能解释为什么 `Unmarshal` 需要传指针。
- 能知道结构体字段必须导出才会参与 JSON 编解码。
- 能主动区分数据库模型和接口响应模型。

