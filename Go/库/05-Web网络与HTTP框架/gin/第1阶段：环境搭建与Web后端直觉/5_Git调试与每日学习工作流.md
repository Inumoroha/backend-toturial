# 5. Git、调试与每日学习工作流

本节目标：建立一个稳定的学习工作流。学习后端不是一次写完，而是每天小步开发、小步验证、小步提交。

---

## 一、推荐每日流程

每次学习建议按这个顺序：

```text
打开教程
→ 启动上一次代码
→ 确认接口还通
→ 写一个小功能
→ 用 requests.http 测试
→ 看终端日志
→ 记录问题
→ Git 提交
```

不要连续写两个小时才运行。代码越攒越多，错误越难定位。

---

## 二、每节提交一次 Git

查看状态：

```bash
git status
```

提交：

```bash
git add .
git commit -m "finish standard http server"
```

推荐提交信息：

```text
init go module
add standard http server
add gin ping route
add user crud routes
add response wrapper
add jwt middleware
```

提交信息不用华丽，但要能看出你完成了什么。

---

## 三、接口不通时怎么查

按顺序检查：

1. 服务是否启动。
2. 请求端口是否正确。
3. 请求路径是否正确。
4. 请求方法是否正确。
5. 请求头是否正确。
6. 请求体是否正确。
7. 终端有没有日志。
8. handler 是否真的被注册。

不要第一反应就怀疑框架。初学阶段 80% 的问题来自路径、端口、方法、JSON。

---

## 四、常见错误直觉

```text
404 Not Found
```

通常是路由路径不对。

```text
405 Method Not Allowed
```

通常是路径对了，但 HTTP 方法不对。

```text
400 Bad Request
```

通常是参数、JSON 或校验失败。

```text
401 Unauthorized
```

通常是未登录、token 缺失或 token 无效。

```text
500 Internal Server Error
```

通常是服务内部错误，例如数据库失败、空指针或未处理异常。

---

## 五、学习笔记应该记什么

不要只复制代码。建议记录：

- 今天完成了哪个接口。
- 哪个命令第一次见。
- 哪个错误卡了最久。
- 最后怎么解决的。
- 如果重写一遍，步骤是什么。

示例：

```text
问题：POST /users 返回 400。
原因：请求头没有设置 Content-Type: application/json。
解决：在 requests.http 中补上请求头。
```

这种笔记比单纯摘抄概念更有用。

---

## 六、本阶段最终练习

完成一个标准库小服务：

```text
GET /ping
GET /hello?name=alice
GET /time
```

要求：

- 每个接口返回 JSON。
- 使用 `requests.http` 测试。
- 终端能看到请求日志。
- Git 至少提交 2 次。
- 记录至少 1 个错误排查过程。

---

## 七、本节验收

你应该能够回答：

- 为什么不要写太久才运行一次？
- 接口不通时应该按什么顺序排查？
- 400、404、500 的常见原因分别是什么？
- Git 提交信息应该表达什么？
- 学习笔记记录错误有什么价值？

