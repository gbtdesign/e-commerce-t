---
kind: dependency_management
name: Go 依赖管理：无 go.mod 的裸源码仓库，前端静态资源直接内嵌
category: dependency_management
scope:
    - '**'
source_files:
    - common/mysql.go
    - rabbitmq/rabbitmq.go
    - backend/main.go
    - fronted/main.go
    - .gitignore
---

## 1. 使用的系统/方法

- **后端 Go 模块**：仓库根目录**不存在 `go.mod` / `go.sum`**，也没有 `vendor/` 目录（`.gitignore` 中仅有一行注释 `# vendor/`）。所有 `.go` 文件通过相对包路径引用本地子目录（如 `imooc-product/common`、`imooc-product/repositories`），说明该仓库本身是一个被其他项目以 module path `imooc-product` 引入的依赖，而不是一个独立可构建的 Go module。外部使用者需要自行提供 `go.mod` 来解析 `github.com/kataras/iris`、`github.com/go-sql-driver/mysql`、`github.com/streadway/amqp`、`github.com/opentracing/opentracing-go/log` 等第三方库。
- **前端静态资源**：没有 `package.json`、`yarn.lock`、`node_modules` 或任何前端构建工具链。Bootstrap、jQuery、Chart.js、Moment.js、Select2、Summernote、Parsley、Masonry、Raphael 等大量第三方 JS/CSS 库以**已编译/压缩后的源码形式直接内嵌**在 `backend/web/assets/lib/<lib>/` 与 `fronted/web/public/js|css|fonts/img/` 下，属于“vendored assets”模式而非 npm 依赖。

## 2. 关键文件

- `common/mysql.go` — 唯一显式 import 的 Go 第三方库：`_ "github.com/go-sql-driver/mysql"`（作为驱动注册）。
- `rabbitmq/rabbitmq.go` — 使用 `github.com/streadway/amqp` 连接 RabbitMQ，并硬编码连接串 `amqp://imoocuser:imoocuser@127.0.0.1:5672/imooc`。
- `backend/main.go`、`fronted/main.go` — 使用 `github.com/kataras/iris` 与 `github.com/kataras/iris/mvc`；同时 `backend/main.go` 还使用了 `github.com/opentracing/opentracing-go/log`。
- `backend/web/assets/lib/` — 内嵌约 30 个前端第三方库（bootstrap、jquery、chartjs、moment.js、select2、summernote、parsley、masonry、raphael、roboto 等）。
- `fronted/web/public/{js,css,fonts,img}/` — 内嵌站点所需的 Bootstrap、jQuery、Owl Carousel、Magnific Popup、Flickity 等静态资源。
- `.gitignore` — 仅忽略 `.DS_Store` 与一行注释 `# vendor/`，未声明任何依赖清单。

## 3. 架构与约定

- **Go 层**：采用多包聚合结构（`common`、`datamodels`、`repositories`、`services`、`encrypt`、`tool`、`rabbitmq` 等），由 `backend/main.go` 与 `fronted/main.go` 两个入口组装 MVC 控制器与服务层。第三方依赖全部来自 `github.com/kataras/iris` 生态及少量基础设施库（mysql driver、amqp、opentracing log）。
- **前端层**：完全绕过 npm/yarn，所有 UI 组件库以“下载后放入仓库”的方式维护，版本锁定体现在提交历史中的文件内容变更上，而非 lockfile。
- **运行时配置**：数据库连接 `root:imooc@tcp(127.0.0.1:3306)/imooc?charset=utf8` 与 RabbitMQ 地址均硬编码在源码中，不通过环境变量或配置文件注入。

## 4. 约定与约束

- **描述性约定**：
  - 所有 Go 第三方依赖通过标准 `import "github.com/..."` 引用，且仓库自身不提供 `go.mod`，依赖版本由引入方决定。
  - 所有前端第三方库以预编译资产形式内嵌到 `backend/web/assets/lib/` 与 `fronted/web/public/`，不使用包管理器。
  - 数据库与消息队列的连接信息以常量/字符串字面量硬编码在源码中。
- **实际约束（由代码体现）**：
  - 由于缺少 `go.mod`，本仓库不能单独 `go build`；必须存在一个包含 `module imooc-product` 的外部 module 才能解析其内部包路径。
  - 运行环境必须预先安装 MySQL（默认 `127.0.0.1:3306`，用户 `root`/密码 `imooc`，库 `imooc`）与 RabbitMQ（默认 `127.0.0.1:5672`，用户 `imoocuser`/密码 `imoocuser`），否则服务启动即失败。
  - 前端静态资源升级需手动替换 `backend/web/assets/lib/<lib>/` 下的对应文件，无自动化更新流程。

## 5. 适用性判断

本仓库确实存在依赖管理的痕迹（Go import 语句、内嵌的前端第三方库），但**没有现代意义上的依赖管理系统**（无 `go.mod`、无 `go.sum`、无 `package.json`、无 lockfile、无私有 registry 配置）。因此该类别在此仓库中仅以“裸源码 + 内嵌资产”的形式存在，证据较弱。

applicable: true
confidence: medium
key_files: ["common/mysql.go", "rabbitmq/rabbitmq.go", "backend/main.go", "fronted/main.go", ".gitignore"]
