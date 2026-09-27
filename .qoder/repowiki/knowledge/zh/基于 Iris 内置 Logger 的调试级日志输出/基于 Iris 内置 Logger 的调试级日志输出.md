---
kind: logging_system
name: 基于 Iris 内置 Logger 的调试级日志输出
category: logging_system
scope:
    - '**'
source_files:
    - backend/main.go
    - fronted/main.go
    - backend/web/controllers/product_controller.go
    - backend/web/controllers/order_controller.go
    - fronted/middleware/auth.go
    - fronted/web/controllers/product_controller.go
    - fronted/web/controllers/user_controller.go
    - consumer.go
---

## 1. 使用的系统/方案

本项目没有引入第三方日志库（如 zap、logrus、slog），而是直接使用 **Iris 框架内置的 `app.Logger()`**，并通过 `SetLevel("debug")` 将日志级别设为 debug。所有 Web 服务（后台 `backend/main.go`、前端 `fronted/main.go`）均采用同一方式初始化。

此外，在 `backend/main.go` 中还存在一处对 OpenTracing 包 `github.com/opentracing/opentracing-go/log` 的误用：`log.Error(err)` 实际调用的是 OpenTracing 的 log 包而非 Iris logger，这属于一个明显的拼写/导入错误，并非真正的结构化日志实现。

## 2. 关键文件

- `backend/main.go` — 后台服务入口，设置 `app.Logger().SetLevel("debug")`
- `fronted/main.go` — 前端服务入口，同样设置 `app.Logger().SetLevel("debug")`
- `backend/web/controllers/product_controller.go` — 控制器中使用 `p.Ctx.Application().Logger().Debug(...)` 记录业务操作与错误
- `backend/web/controllers/order_controller.go` — 使用 `o.Ctx.Application().Logger().Debug(...)`
- `fronted/middleware/auth.go` — 中间件通过 `ctx.Application().Logger().Debug(...)` 记录登录状态
- `fronted/web/controllers/product_controller.go`、`user_controller.go` — 前端控制器中的 Debug/Error 日志
- `consumer.go` — 下单消费者程序，仅使用 `fmt.Println(err)` 输出错误

## 3. 架构与约定

- **无独立 logger 模块**：项目未封装统一的 logger 包或中间件；每个 controller / middleware 直接持有 `iris.Context`，通过 `ctx.Application().Logger()` 获取 Iris 内置 logger。
- **日志级别单一**：两个 Web 入口均硬编码为 `"debug"`，没有按环境切换级别的逻辑。
- **日志内容以字符串为主**：绝大多数调用是 `Logger().Debug("..." + idString)` 或 `Logger().Debug(err)`，即把错误对象或拼接后的字符串直接传入，没有定义结构化的字段键值对。
- **级别使用不统一**：大部分业务路径使用 `Debug`，仅在 `fronted/web/controllers/product_controller.go` 中出现少量 `Error` 调用（如第 62、68、83 行）。
- **非 Web 程序不使用 Iris logger**：`consumer.go` 等独立 Go 程序退化为 `fmt.Println`，说明 Iris logger 仅服务于基于 Iris MVC 的两个 Web 服务。

## 4. 约定与约束

- **Web 服务日志来源**：所有 Iris MVC 控制器和中间件应通过 `ctx.Application().Logger()` 访问日志，而不是直接打印到 stdout。
- **日志级别**：当前仓库内所有 Web 服务固定为 `debug` 级别，未见生产/测试环境差异化配置。
- **日志格式**：依赖 Iris 默认控制台输出格式，未自定义 formatter、sink 或异步写入策略。
- **结构化字段**：未发现统一的日志字段规范（如 trace_id、user_id、product_id 等结构化字段），日志以人类可读字符串为主。
- **OpenTracing 误用**：`backend/main.go:33` 中 `log.Error(err)` 来自 `opentracing-go/log`，该包提供的是 tracing span 的 field 写入接口而非通用日志器，此处应视为 bug，不应被当作结构化日志实践。

总结：本项目的 "logging system" 实质上是 **零配置的 Iris 内置 logger**，仅用于开发期 debug 输出，没有独立的日志抽象层、级别策略、结构化字段或 sink 路由。