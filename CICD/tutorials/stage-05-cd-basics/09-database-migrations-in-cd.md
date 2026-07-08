# 09：CD 中的数据库迁移

## 1. 本节目标

数据库迁移是 CD 中最容易翻车的部分之一。

这一节学习：

- 迁移什么时候执行。
- 为什么不能随便自动回滚数据库。
- 什么是 expand/contract。
- Go 后端如何设计兼容迁移。

## 2. 迁移和应用部署的关系

一次后端发布可能包含：

- 应用代码。
- 镜像。
- 配置。
- 数据库 schema。
- 数据修正脚本。

其中数据库变更通常最难回滚。

所以迁移必须单独设计。

## 3. 迁移执行位置

常见方式：

### 方式一：部署前执行

```text
run migration
-> deploy app
```

适合新增表、新增 nullable 字段等兼容变更。

### 方式二：应用启动时执行

```text
container start
-> app runs migration
-> app starts
```

简单，但多副本时可能冲突。单机阶段可以理解，不建议长期依赖。

### 方式三：单独 migration job

```text
build image
-> run migration job
-> deploy app
```

更清晰，Kubernetes 阶段会更常见。

## 4. Expand/Contract 模式

核心思路：

```text
先扩展 schema，让新旧代码都能运行。
再切换代码。
最后清理旧字段或旧逻辑。
```

示例：把 `name` 拆成 `first_name` 和 `last_name`。

错误做法：

```text
一次发布删除 name，新增 first_name/last_name，代码同时切换。
```

风险：回滚旧代码时 `name` 已经没了。

更安全：

1. 新增 `first_name`、`last_name`，保留 `name`。
2. 新代码双写或兼容读。
3. 数据回填。
4. 确认无旧代码依赖 `name`。
5. 后续版本删除 `name`。

## 5. 安全迁移清单

相对安全：

- 新增表。
- 新增 nullable 字段。
- 新增有默认值且不会锁表太久的字段。
- 新增索引，但要注意大表。

高风险：

- 删除字段。
- 重命名字段。
- 修改字段类型。
- 大表加锁操作。
- 大批量数据更新。
- 不可逆数据清洗。

## 6. 迁移失败怎么办

迁移失败后不要盲目继续部署。

排查：

- 当前迁移执行到哪一步。
- schema 是否部分变更。
- 数据是否被部分修改。
- 应用旧版本还能不能运行。
- 是否需要人工修复。

迁移失败通常需要比应用回滚更谨慎。

## 7. 迁移和 deploy.sh

初学阶段可以先不把迁移自动放进 `deploy.sh`。

如果要放，可以设计显式开关：

```bash
RUN_MIGRATIONS=true IMAGE=... ./deploy.sh
```

并在脚本中：

```bash
if [ "${RUN_MIGRATIONS:-false}" = "true" ]; then
  docker compose run --rm api migrate up
fi
```

前提是你的 Go 程序支持：

```bash
/server migrate up
```

## 8. 生产迁移建议

生产迁移要有：

- 迁移说明。
- 影响评估。
- 备份策略。
- 执行窗口。
- 验证方式。
- 回滚或补偿方案。
- 大表变更方案。

即使应用可以自动部署，数据库变更也常常需要更严格门禁。

## 9. 小练习

为下面变更设计迁移流程：

```text
todo 表新增 priority 字段。
```

回答：

1. 字段是否 nullable？
2. 旧代码能否忽略这个字段？
3. 新代码如何处理旧数据？
4. 是否需要回填？
5. 回滚应用时是否安全？

## 10. 本节小结

你现在应该理解：

- 数据库迁移是 CD 中的高风险动作。
- 镜像回滚不等于数据库回滚。
- expand/contract 可以降低破坏性变更风险。
- 初学阶段迁移可以手动或显式触发，不要无脑自动执行。

