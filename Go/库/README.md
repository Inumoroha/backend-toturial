# Go 库学习资料分类索引

这个目录按功能整理 Go 标准库、常用第三方库和后端开发常见组件。已有资料已移动到对应分类；暂时没有内容但值得后续学习的库，已建立空目录占位。

## 01-基础工具与通用标准库

适合放语言日常开发中高频使用的基础能力。

- `fmt`: 格式化输入输出。
- `errors`: 错误创建、包装和判断。
- `log`: 标准日志。
- `flag`: 命令行参数解析。
- `regexp`: 正则表达式。
- `time`: 时间、定时器和超时。
- `bytes`, `sort`, `math`, `container`: 后续补充占位。

## 02-字符串编码与数据格式

适合放字符串处理、类型转换、序列化和模板相关内容。

- `encoding_json`: JSON 编解码。
- `strings_strconv`: 原 `stings_strconv`，已更正拼写。
- `strings`, `strconv`, `unicode`, `encoding_xml`, `encoding_csv`, `encoding_base64`, `html_template`, `text_template`: 后续补充占位。

## 03-IO文件与系统交互

适合放输入输出、文件系统、路径、压缩和嵌入资源相关内容。

- `bufio`: 带缓冲 I/O。
- `os`: 原 `os_is`，按 Go 标准库包名整理。
- `io`, `path_filepath`, `embed`, `archive_zip`, `compress_gzip`: 后续补充占位。

## 04-并发控制与上下文

适合放 goroutine 协作、同步原语、取消控制、运行时和性能相关内容。

- `context`: 上下文、取消、超时和请求链路传递。
- `sync_atomic`: 原 `sync_atmoic`，已更正拼写。
- `channels`, `runtime`, `x_sync_errgroup`: 后续补充占位。

## 05-Web网络与HTTP框架

适合放网络协议、HTTP 服务、Web 框架、RPC 和实时通信相关内容。

- `net_http`: 标准库 HTTP 服务端和客户端。
- `gin`: 原 `Gin`，统一为小写目录名。
- `net`, `net_url`, `net_rpc`, `grpc`, `websocket`, `echo`, `fiber`: 后续补充占位。

## 06-数据库缓存与ORM

适合放 SQL、ORM、数据库驱动、缓存和 NoSQL 相关内容。

- `database_sql`: 标准库数据库接口。
- `gorm`: 原 `GORM`，统一为小写目录名。
- `sqlx`, `pgx`, `redis_go`, `mongo_go_driver`: 后续补充占位。

## 07-工程化配置日志与测试

适合放测试、日志、配置、CLI、Mock、性能分析和工程实践。

- `testing`: 标准库测试。
- `log_slog`, `zap`, `zerolog`, `cobra`, `viper`, `testify`, `go_mock`, `pprof_trace`: 后续补充占位。

## 08-标准库总览与学习路线

适合放 Go 标准库全局梳理、学习路线、知识地图和复盘资料。

- `standard`: 原标准库总览资料。

## 后续放置建议

- 标准库优先按包名放入对应功能分类。
- 第三方库按主要用途放入功能分类，比如 Gin 放 Web，GORM 放数据库，Cobra 放工程化。
- 如果一个库跨多个领域，优先放到学习时最常检索的入口，并在相关目录 README 中互相引用。
