---
kind: build_system
name: Go 多模块电商系统的原生构建与压测脚本
category: build_system
scope:
    - '**'
source_files:
    - backend/main.go
    - fronted/main.go
    - info
    - imooc
    - backend/web/assets/lib/jquery-flot/Makefile
---

## 1. 使用的构建系统/方式

仓库是一个 Go 语言项目，采用**纯原生的 `go run` / `go build` 开发模式**，没有 Makefile、Dockerfile、CI 流水线或任何自动化构建脚本。每个可执行程序都是一个独立的 `package main`：

- `backend/main.go` — 后台管理 Web 服务（Iris MVC），监听 `localhost:8080`
- `fronted/main.go` — 前端站点服务（Iris MVC），监听 `0.0.0.0:8082`
- 根目录下的 `consumer.go`、`getOne.go`、`validate.go` 等文件也是独立的可执行入口

依赖通过 Go module 管理（包路径统一以 `imooc-product/...` 引用，见 `backend/main.go`、`fronted/main.go` 中的 import），但仓库中未检出 `go.mod` / `go.sum` 文件。

## 2. 关键文件

- `backend/main.go` — 后台服务启动、模板注册、MVC 路由装配、MySQL 连接、静态资源挂载
- `fronted/main.go` — 前端服务启动、RabbitMQ 初始化、中间件挂载、静态 HTML 目录 `/html` 暴露
- `info` — 使用 `wrk` 对 `getOne` 接口进行压力测试的命令行片段（`-t80 -c200/-c2000/-c20000 -d30s --latency`）
- `imooc` — 仅包含 5 行目录结构说明（model → repositories → services → controllers → views），是分层约定而非构建配置
- `backend/web/assets/lib/jquery-flot/Makefile` — 第三方库自带的 Makefile，用于生成 minified 文件，与本项目无关

## 3. 架构与约定

- **多进程部署**：后台、前端、下单消费者等各自作为独立 Go 二进制运行，端口硬编码在源码中（8080、8082），没有统一的进程管理器或配置文件。
- **模板与静态资源内嵌**：Iris 通过 `iris.HTML(...).Reload(true)` 和 `app.StaticWeb(...)` 直接指向磁盘上的 `views/`、`assets/`、`public/` 目录，开发时启用热重载。
- **分层目录约定**：`common/`、`datamodels/`、`repositories/`、`services/`、`tool/`、`encrypt/`、`rabbitmq/` 作为共享库被各 `main` 包引用。

## 4. 约定与约束

- **无自动化构建**：仓库中没有 `Makefile`、`Dockerfile`、`.github/workflows`、`Jenkinsfile`、`build.sh` 等构建/发布脚本；编译只能手动执行 `go run ./backend/main.go` 或 `go build -o backend .`。
- **无 CI/CD**：未发现任何持续集成或持续部署配置。
- **无容器化**：未发现 Dockerfile 或 docker-compose 文件。
- **压测脚本固定**：`info` 文件记录了针对 `http://172.31.96.53:12345/getOne` 的三档并发压测命令，依赖外部工具 `wrk`。
- **端口硬编码**：后端服务地址写死为 `localhost:8080`，前端服务地址写死为 `0.0.0.0:8082`，没有环境变量注入。
- **第三方静态资源自带构建**：`backend/web/assets/lib/jquery-flot/Makefile` 由第三方库提供，用于生成压缩后的 JS/CSS，不属于本项目的构建体系。