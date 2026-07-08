# 01 博客系统实战

本节目标：围绕「blog api project」建立清晰的 GORM 学习目标，理解它解决的具体后端问题，能够写出可运行示例，并能通过数据库结果或 SQL 日志验证自己的代码是否正确。

博客系统用于综合练习 GORM 的基础能力：模型设计、CRUD、关联查询、分页、软删除、Repository 分层和接口开发。

## 1. 需求范围

第一版不要做太大，先完成：

- 用户注册
- 用户登录
- 创建分类
- 创建标签
- 创建文章
- 编辑文章
- 删除文章
- 文章列表
- 文章详情
- 创建评论
- 删除评论

## 2. 模型清单

- `User`
- `Category`
- `Tag`
- `Article`
- `Comment`

关系：

```text
User 1 - n Article
User 1 - n Comment
Category 1 - n Article
Article 1 - n Comment
Article n - n Tag
```

## 3. 推荐接口

### 用户

```text
POST /api/register
POST /api/login
GET  /api/me
```

### 分类

```text
POST /api/categories
GET  /api/categories
```

### 标签

```text
POST /api/tags
GET  /api/tags
```

### 文章

```text
POST   /api/articles
GET    /api/articles
GET    /api/articles/:id
PUT    /api/articles/:id
DELETE /api/articles/:id
```

### 评论

```text
POST   /api/articles/:id/comments
DELETE /api/comments/:id
```

## 4. 开发顺序

### 第一步：搭建项目

- 创建目录结构。
- 初始化 `go.mod`。
- 安装 Gin、GORM、MySQL 驱动。
- 写数据库初始化。
- 写模型和迁移。

### 第二步：用户模块

- 注册时校验邮箱唯一。
- 密码使用 hash。
- 登录成功返回 token。
- 后续接口通过用户 ID 识别当前用户。

### 第三步：分类和标签

- 分类名唯一。
- 标签名唯一。
- 创建文章时可以绑定标签。

### 第四步：文章模块

创建文章：

```text
1. 校验分类存在
2. 校验标签存在
3. 创建文章
4. 关联标签
5. 使用事务
```

文章列表：

```text
1. 支持分页
2. 支持分类筛选
3. 支持关键词搜索
4. 只返回摘要字段
5. 预加载作者和分类
```

文章详情：

```text
1. 加载作者
2. 加载分类
3. 加载标签
4. 加载评论
5. 加载评论用户
```

### 第五步：评论模块

- 登录用户可以评论文章。
- 评论支持软删除。
- 删除评论时校验作者或管理员权限。

## 5. Repository 划分

建议：

```text
UserRepository
CategoryRepository
TagRepository
ArticleRepository
CommentRepository
```

每个 Repository 只处理数据库。

## 6. Service 划分

建议：

```text
AuthService
ArticleService
CommentService
CategoryService
TagService
```

复杂业务放 Service：

- 注册
- 登录
- 创建文章并关联标签
- 编辑文章并替换标签
- 删除评论权限校验

## 7. 关键 GORM 技术点

你会用到：

- `Create`
- `First`
- `Find`
- `Where`
- `Updates`
- `Delete`
- `Preload`
- `Association("Tags").Replace`
- `Transaction`
- `Count`
- `Limit`
- `Offset`
- `Select`

## 8. 验收清单

- [ ] 注册接口能创建用户。
- [ ] 登录接口能校验密码。
- [ ] 创建文章时能关联标签。
- [ ] 编辑文章时能替换标签。
- [ ] 文章列表支持分页。
- [ ] 文章详情能返回作者、分类、标签、评论。
- [ ] 删除文章是软删除。
- [ ] 删除评论是软删除。
- [ ] SQL 日志能看到预加载查询。
- [ ] 没有明显 N+1 查询。

## 9. 加分项

- [ ] JWT 登录。
- [ ] 文章发布和草稿状态。
- [ ] 后台审核评论。
- [ ] 热门文章排行。
- [ ] 按标签查询文章。
- [ ] Repository 测试。
- [ ] Service 事务测试。
---

## 本节达标标准

学完本节后，你应该能够做到：

- 说清本节主题解决的具体后端开发问题。
- 写出本节涉及的核心 GORM 代码或 SQL 示例。
- 通过 SQL 日志、数据库客户端或测试验证结果是否正确。
- 识别本节列出的常见错误，并知道如何排查。
- 把本节知识放回博客系统或订单系统的真实场景中使用。
