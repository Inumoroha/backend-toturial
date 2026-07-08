# 04. readPreference 读偏好

readPreference 决定客户端从复制集哪个节点读取数据。它能分担读压力，也可能带来读到旧数据的风险。

## 1. 常见读偏好

| readPreference | 含义 |
| --- | --- |
| `primary` | 只从 primary 读，默认选择 |
| `primaryPreferred` | 优先 primary，不可用时读 secondary |
| `secondary` | 只从 secondary 读 |
| `secondaryPreferred` | 优先 secondary，不可用时读 primary |
| `nearest` | 从网络延迟较低的节点读 |

## 2. 默认 primary

默认从 primary 读。

适合：

- 强一致业务。
- 写后立即读。
- 钱包余额。
- 订单支付状态。
- 库存判断。

例如用户支付成功后立即查询订单状态，应该优先从 primary 读。

## 3. secondary 读的收益

secondary 读可以：

- 分担 primary 压力。
- 支持报表或分析查询。
- 地理分布场景下降低读延迟。

适合：

- 内容浏览。
- 后台低实时性报表。
- 历史数据查询。
- 可接受延迟的列表。

## 4. secondary 读的风险

secondary 可能复制落后。

风险：

- 刚写入的数据读不到。
- 状态看起来没更新。
- 钱包余额可能是旧值。
- 订单支付状态可能延迟。

所以不要用 secondary 读做强一致业务判断。

## 5. mongosh 设置读偏好

连接串方式：

```powershell
mongosh "mongodb://localhost:27017,localhost:27018,localhost:27019/?replicaSet=rs0&readPreference=secondary"
```

或者在 shell 中：

```javascript
db.getMongo().setReadPref("secondary")
```

查询：

```javascript
db.orders.find().limit(5)
```

## 6. secondary 读可能需要允许

老版本 shell 常见提示需要设置 secondaryOk。现代驱动通常通过 readPreference 管理。

你只需要记住：不是所有读都应该从 secondary 读。

## 7. Go 中设置读偏好

```go
clientOpts := options.Client().
    ApplyURI(uri).
    SetReadPreference(readpref.SecondaryPreferred())

client, err := mongo.Connect(clientOpts)
```

需要导入：

```go
"go.mongodb.org/mongo-driver/v2/mongo/readpref"
```

也可以对单个 collection 设置：

```go
coll := db.Collection(
    "orders",
    options.Collection().SetReadPreference(readpref.Primary()),
)
```

## 8. 场景选择

### 登录查询用户

建议 primary。

### 文章列表

可以考虑 secondaryPreferred，取决于一致性要求。

### 支付后查订单

primary。

### 后台日报

可以考虑 secondary，但最好走专门报表链路。

### 钱包余额

primary。

## 9. 本节练习

判断以下场景读偏好：

1. 用户支付后立即查订单。
2. 用户浏览公开文章列表。
3. 后台统计昨天订单总额。
4. 钱包转账前检查余额。
5. 用户查看刚发布的评论。
6. 管理员搜索历史订单。

写出：

```text
场景：
推荐 readPreference：
理由：
是否能接受旧数据：
```

## 10. 本节小结

你需要记住：

- readPreference 控制从哪个节点读。
- 默认 primary 最安全。
- secondary 读能分担压力，但可能读旧数据。
- 强一致业务不要随便读 secondary。
- 读偏好是业务一致性和性能之间的取舍。

## 11. readPreference 不是缓存开关

把读请求切到 secondary 并不等于“免费提升性能”。它会引入新的问题：

- secondary 可能落后 primary。
- 读到的数据可能不是最新。
- 长查询可能拖慢 secondary 复制。
- 备份节点和分析查询可能互相影响。
- 故障切换时读写路径会变化。

因此 readPreference 是一致性策略，不只是性能参数。

## 12. 常见模式选择

| 模式 | 含义 | 适合场景 |
| --- | --- | --- |
| `primary` | 只读 primary | 强一致、默认选择 |
| `primaryPreferred` | 优先 primary，失败时读 secondary | 可用性优先但要谨慎 |
| `secondary` | 只读 secondary | 离线分析、非核心读 |
| `secondaryPreferred` | 优先 secondary，不能用时读 primary | 可接受旧数据的查询 |
| `nearest` | 选网络延迟近的节点 | 多地域低延迟读，需谨慎一致性 |

对 Go 后端来说，默认保守选择是 `primary`。只有明确业务允许读旧，才单独给某些集合或查询设置 secondary。

## 13. Go 中按场景设置读偏好

订单核心查询：

```go
orders := db.Collection("orders", options.Collection().
    SetReadPreference(readpref.Primary()))
```

后台报表查询：

```go
reports := db.Collection("events", options.Collection().
    SetReadPreference(readpref.SecondaryPreferred()))
```

不要在全局客户端上随意设置：

```go
options.Client().SetReadPreference(readpref.SecondaryPreferred())
```

这样可能让登录、支付、订单详情等强一致接口也读到旧数据。

## 14. 复制延迟下的用户体验

假设用户刚修改昵称：

```text
T0：写入 primary 成功
T0 + 100ms：用户刷新资料页
T0 + 2s：secondary 才同步到新昵称
```

如果资料页读 secondary，用户可能看到旧昵称。对于有些业务可以接受，对于支付状态、权限变更、封禁状态就不能接受。

所以设计读偏好时要写清：

```text
这个接口是否允许读旧？
最多允许旧多久？
读旧会造成什么后果？
是否需要读 primary？
```

## 15. secondary 读也需要索引

有些团队把慢查询扔给 secondary，以为不会影响主库。这也很危险。

secondary 执行慢查询会消耗 CPU 和 IO，可能导致：

- 复制延迟增加。
- Change Streams 延迟增加。
- 故障切换后该节点成为 primary 时状态不佳。

所以 secondary 上的查询同样要：

- 有索引。
- 控制返回字段。
- 避免深分页。
- 避免在线大聚合。

## 16. 读偏好评审模板

```md
# readPreference 评审

## 接口

## 数据重要性

## 是否允许读旧

## 最大可接受延迟

## 推荐 readPreference

## 需要的索引

## 故障时表现

## 用户体验说明
```

## 17. 官方文档延伸阅读

- [Read Preference](https://www.mongodb.com/docs/manual/core/read-preference/)
