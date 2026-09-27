---
kind: error_handling
name: 基于 Iris MVC 的错误处理与错误视图体系
category: error_handling
scope:
    - '**'
source_files:
    - backend/main.go
    - fronted/main.go
    - backend/web/views/shared/error.html
    - fronted/web/views/shared/error.html
    - common/filter.go
    - common/form.go
    - common/consistent.go
    - common/ip.go
    - encrypt/aes.go
    - repositories/product_repository.go
    - services/product_service.go
    - fronted/web/controllers/user_controller.go
---

## 1. 整体方案

本仓库采用 **Iris MVC** 作为 Web 框架，错误处理围绕以下三个层次展开：
- **HTTP 层统一错误视图**：通过 `app.OnAnyErrorCode` 全局钩子捕获所有非 2xx 状态码，统一渲染 `shared/error.html`。
- **业务/工具层标准 error 返回**：各包（common、encrypt、repositories）使用 Go 原生 `error` 接口向上返回错误，调用方在控制器或中间件中决定如何呈现。
- **自定义表单解码错误类型**：`common/form.go` 内置了 `Error` 结构体，实现 `Error()`、`MarshalJSON()` 以及兼容 `github.com/pkg/errors` 的 `Cause()` 方法，用于包装表单解析过程中的字段级错误。

没有发现统一的错误码枚举、结构化错误码表或第三方错误库（如 `pkg/errors`、`sentinel`、`xerrors`）的使用；也没有 `panic/recover` 模式。错误传播遵循“函数返回值中携带 error”这一 Go 惯用方式。

## 2. 关键文件与位置

| 层级 | 文件 | 作用 |
|---|---|---|
| 应用入口 | `backend/main.go`、`fronted/main.go` | 注册 `OnAnyErrorCode` 全局错误处理器，将错误重定向到 `shared/error.html` |
| 模板 | `backend/web/views/shared/error.html`、`fronted/web/views/shared/error.html` | 统一错误展示页面，通过 `ViewData("message", ...)` 取错消息 |
| 通用拦截器 | `common/filter.go` | 自定义 `FilterHandle` 返回 `error`，并在 `Handle` 中直接 `rw.Write([]byte(err.Error()))` 输出错误字符串 |
| 表单解码 | `common/form.go` | 定义 `Error` 结构体及 `newError` 构造器，为表单解析提供带上下文的错误 |
| 一致性哈希 | `common/consistent.go` | 定义包级哨兵错误 `errEmpty = errors.New("Hash 环没有数据")`，供 `Get` 空环时复用 |
| 加密模块 | `encrypt/aes.go` | 对解密失败等场景返回 `errors.New("加密字符串错误！")` 等具体错误 |
| IP 工具 | `common/ip.go` | 获取本机 IP 失败时返回 `errors.New("获取地址异常")` |
| 仓储层 | `repositories/product_repository.go` 等 | 数据库操作失败直接返回底层 `sql` 错误，未做封装 |
| 服务层 | `services/product_service.go` 等 | 透传仓储层的 error，不做额外包装 |
| 控制器 | `fronted/web/controllers/user_controller.go` | 收到 service 错误后通过 `Redirect("/user/error")` 跳转；登录失败返回 `mvc.Response{Path: "/user/login"}` |

## 3. 架构与约定

### 3.1 HTTP 层：全局错误视图
两个 Iris 应用入口都采用相同模式：
```go
app.OnAnyErrorCode(func(ctx iris.Context) {
    ctx.ViewData("message", ctx.Values().GetStringDefault("message", "访问的页面出错！"))
    ctx.ViewLayout("")
    ctx.View("shared/error.html")
})
```
这意味着：**任何控制器或路由返回的非 2xx 状态码都会进入该回调**，并渲染同一份错误页。错误信息通过 `ctx.Values().SetString("message", ...)` 注入，再由模板读取。

### 3.2 业务/工具层：标准 error 返回
- `common/comm.go` 的 `TypeConversion` 遇到未知类型返回 `errors.New("未知的类型：" + ntype)`。
- `common/consistent.go` 定义包级变量 `errEmpty`，避免重复分配，体现“可复用的哨兵错误”约定。
- `common/ip.go` 的 `GetIntranceIp` 在找不到非回环地址时返回 `errors.New("获取地址异常")`。
- `encrypt/aes.go` 的 `PKCS7UnPadding` 在长度为 0 时返回 `errors.New("加密字符串错误！")`。

这些错误都是裸 `error`，没有统一包装，调用方自行判断。

### 3.3 表单解码错误：专用 Error 类型
`common/form.go` 内嵌了一个完整的表单解码器，其错误模型为：
```go
type Error struct { err error }
func (s *Error) Error() string { return "imooc: " + s.err.Error() }
func (s Error) MarshalJSON() ([]byte, error) { return json.Marshal(s.err.Error()) }
func (s *Error) Cause() error { return s.err } // 兼容 pkg/errors
```
内部通过 `newError(fmt.Errorf(...))` 构造，统一加上 `"imooc: "` 前缀，便于区分来源。该类型同时实现了 JSON 序列化，适合 API 场景。

### 3.4 自定义 Filter 的错误处理
`common/filter.go` 中的 `Filter.Handle` 在执行注册的 `FilterHandle` 后，如果返回 `err != nil`，会直接 `rw.Write([]byte(err.Error()))` 并提前返回，不再执行后续 web 处理函数。这是一种“短路式”错误中断。

### 3.5 仓储与服务层：透传策略
- `repositories/*_repository.go` 中数据库操作失败直接返回 `sql` 层 error（例如 `stmt.Exec` 的 err），未做领域语义包装。
- `services/*_service.go` 基本是仓储方法的薄包装，直接透传 error。
- 控制器层根据业务需要决定行为：如 `PostRegister` 中若 `AddUser` 返回 error，则 `c.Ctx.Redirect("/user/error")`。

## 4. 约定与约束

| 约定 | 说明 | 依据 |
|---|---|---|
| HTTP 错误统一走 `OnAnyErrorCode` | 所有非 2xx 响应由全局回调渲染 `shared/error.html` | `backend/main.go`、`fronted/main.go` |
| 业务错误以 `error` 返回值传递 | 各层函数签名均以 `(..., error)` 结尾 | 全仓库函数签名观察 |
| 无 panic/recover 模式 | 未发现 `defer recover()` 或 `panic` 调用 | 全仓库 grep 确认 |
| 可复用哨兵错误 | 如 `errEmpty` 使用包级 `var` 声明，避免重复分配 | `common/consistent.go` |
| 表单解码错误加前缀 | 所有解码错误经 `newError` 包装，统一带 `"imooc: "` 前缀 | `common/form.go` |
| 控制器错误跳转 `/user/error` | 注册失败时显式跳转到错误路径 | `fronted/web/controllers/user_controller.go` |
| 自定义 Filter 错误直接写 body | 不经过模板，直接输出 `err.Error()` | `common/filter.go` |

## 5. 缺失与改进点（描述性）

- 缺少统一的错误码体系（如 HTTP 状态码映射、业务错误码常量），错误多为自由文本。
- 仓储层未对 SQL 错误做领域语义包装（如区分“不存在”、“唯一冲突”），调用方难以精确分支处理。
- 未使用结构化日志记录错误上下文（仅 `log.Error(err)` 或 `fmt.Println`）。
- 自定义 Filter 的错误处理绕过模板，与其他 HTTP 错误处理方式不一致。
