# 04 博客系统逐步落地手册

本节目标：把博客系统从空项目一步一步落地。它不是只给你功能清单，而是告诉你先写什么、后写什么、每一步如何验证。

---

## 一、项目目标

完成一个最小但结构完整的博客 API：

```text
用户注册登录
分类管理
标签管理
文章创建、编辑、列表、详情、删除
评论创建、删除
```

技术栈：

```text
Go
Gin
GORM
MySQL
JWT
```

---

## 二、第一步：初始化项目

```powershell
mkdir blog-api
cd blog-api
go mod init blog-api
go get github.com/gin-gonic/gin
go get gorm.io/gorm
go get gorm.io/driver/mysql
go get golang.org/x/crypto/bcrypt
go get github.com/golang-jwt/jwt/v5
```

创建目录：

```text
blog-api
├── cmd
│   └── server
│       └── main.go
├── internal
│   ├── config
│   ├── database
│   ├── dto
│   ├── handler
│   ├── middleware
│   ├── model
│   ├── repository
│   └── service
└── go.mod
```

---

## 三、第二步：数据库连接

先完成：

```text
internal/config/config.go
internal/database/database.go
cmd/server/main.go
```

验证标准：

```powershell
go run ./cmd/server
```

能输出：

```text
server started
database connected
```

不要急着写业务接口。先确认数据库连接稳定。

---

## 四、第三步：模型和迁移

先写模型：

```text
internal/model/user.go
internal/model/category.go
internal/model/tag.go
internal/model/article.go
internal/model/comment.go
```

然后在启动时迁移：

```go
db.AutoMigrate(
    &model.User{},
    &model.Category{},
    &model.Tag{},
    &model.Article{},
    &model.Comment{},
)
```

验证：

```sql
SHOW TABLES;
SHOW CREATE TABLE articles;
SHOW CREATE TABLE article_tags;
```

必须确认 `article_tags` 已经生成。

---

## 五、第四步：用户注册

接口：

```http
POST /api/register
```

请求：

```json
{
  "username": "tom",
  "email": "tom@example.com",
  "password": "123456"
}
```

Service 流程：

```text
1. 校验用户名、邮箱、密码非空
2. 检查邮箱是否已存在
3. bcrypt 生成密码哈希
4. 创建用户
5. 返回用户基础信息
```

验证：

```sql
SELECT id, username, email, password_hash FROM users;
```

确认：

- `password_hash` 不是明文。
- 邮箱重复注册会失败。

---

## 六、第五步：用户登录

接口：

```http
POST /api/login
```

请求：

```json
{
  "email": "tom@example.com",
  "password": "123456"
}
```

流程：

```text
1. 根据邮箱查询用户
2. bcrypt 校验密码
3. 生成 JWT
4. 返回 token
```

响应：

```json
{
  "token": "..."
}
```

验证：

- 正确密码能登录。
- 错误密码不能登录。
- 不存在的邮箱不能登录。

---

## 七、第六步：认证中间件

中间件职责：

```text
1. 读取 Authorization Header
2. 解析 Bearer token
3. 校验 JWT
4. 把 user_id 放进 gin.Context
```

后续创建文章、评论时，作者 ID 都来自中间件，不来自前端请求体。

---

## 八、第七步：分类和标签

先做简单 CRUD：

```text
POST /api/categories
GET  /api/categories
POST /api/tags
GET  /api/tags
```

验证：

- 名称不能为空。
- 名称不能重复。
- 列表能返回创建的数据。

---

## 九、第八步：创建文章

接口：

```http
POST /api/articles
Authorization: Bearer <token>
```

流程：

```text
1. 从 token 获取 user_id
2. 校验分类存在
3. 校验标签都存在
4. 创建文章
5. 绑定标签
6. 使用事务
```

验证：

```sql
SELECT * FROM articles;
SELECT * FROM article_tags;
```

如果标签 ID 不存在，文章不应该创建成功。

---

## 十、第九步：文章列表

接口：

```http
GET /api/articles?page=1&page_size=10&category_id=1&keyword=GORM
```

要求：

- 支持分页。
- 支持分类筛选。
- 支持关键词搜索。
- 只返回摘要，不返回正文。
- 返回总数。

响应结构：

```json
{
  "list": [],
  "total": 100,
  "page": 1,
  "page_size": 10
}
```

---

## 十一、第十步：文章详情

接口：

```http
GET /api/articles/:id
```

要求加载：

- 作者
- 分类
- 标签
- 评论
- 评论用户

使用：

```go
Preload("User").
Preload("Category").
Preload("Tags").
Preload("Comments.User")
```

验证 SQL 日志，确认没有 N+1。

---

## 十二、第十一步：评论

创建评论：

```http
POST /api/articles/:id/comments
```

删除评论：

```http
DELETE /api/comments/:id
```

规则：

- 登录用户才能评论。
- 评论内容不能为空。
- 删除评论要校验作者。
- 评论使用软删除。

---

## 十三、第十二步：性能检查

完成接口后，打开 SQL 日志检查：

```text
文章列表是否分页
文章列表是否查询正文
文章详情是否 N+1
评论是否一次加载太多
常用过滤字段是否有索引
```

---

## 十四、最终验收清单

- [ ] 项目能一条命令启动。
- [ ] 数据库能自动迁移。
- [ ] 用户密码不是明文。
- [ ] JWT 能保护需要登录的接口。
- [ ] 创建文章能绑定标签。
- [ ] 标签不存在时文章创建回滚。
- [ ] 文章列表支持分页和筛选。
- [ ] 文章详情加载完整关联。
- [ ] 评论可以软删除。
- [ ] SQL 日志没有明显 N+1。
- [ ] README 写清楚如何运行项目。
---

## 本节达标标准

学完本节后，你应该能够做到：

- 说清本节主题解决的具体后端开发问题。
- 写出本节涉及的核心 GORM 代码或 SQL 示例。
- 通过 SQL 日志、数据库客户端或测试验证结果是否正确。
- 识别本节列出的常见错误，并知道如何排查。
- 把本节知识放回博客系统或订单系统的真实场景中使用。
