# 请求路由装配（request-routing）

> 本文是 `server-api` 域下的叶子子系统文档。域级总览见 `../server-api.md`。
> 本文只展开 Express 应用如何装配全局中间件、挂载路由组与启动 HTTP/HTTPS 服务，
> 不展开各端点业务处理（分别见 auth-authz-api、workspace-chat-api 等叶子）。
>
> 源码基准：anything-llm v1.16.2，commit `128a0157`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| Express 应用与全局中间件装配 | 创建 `app=express()`，依次挂 cors、bodyParser（text/json/urlencoded，上限 3GB） | `server/index.js:50-73` |
| 开发态 HTTP 请求日志 | 仅 `NODE_ENV=development` 且 `ENABLE_HTTP_LOGGER=true` 时挂 `httpLogger`（劫持 `res.end` 记录状态码） | `server/index.js:55-64`、`server/middleware/httpLogger.js:1-19` |
| WebSocket 支持 | 非 HTTPS 模式加载 `@mintplex-labs/express-ws`；HTTPS 模式在证书加载后再挂载 | `server/index.js:78`、`server/utils/boot/index.js:48` |
| `/api` 子路由挂载 | `app.use("/api", apiRouter)`，所有业务端点统一挂在该 router 下 | `server/index.js:51,81` |
| 20+ 端点模块注册 | 依次调用各 `xxxEndpoints(apiRouter)` 把路由组注册到 apiRouter | `server/index.js:82-112` |
| 开发者 OpenAPI 端点 | `developerEndpoints(app, apiRouter)` 双参挂载，内部启用 Swagger 文档 | `server/index.js:97`、`server/endpoints/api/index.js:14-26` |
| 生产静态资源托管 | 非开发态把 `server/public` 作为静态目录，注入 `X-Frame-Options: DENY` 等安全头 | `server/index.js:114-127` |
| SPA 入口与元信息 | `/robots.txt`、`/manifest.json`、兜底 `app.use("/")` 由 `MetaGenerator` 生成 HTML | `server/index.js:129-142` |
| 开发态向量库调试路由 | 开发态暴露 `POST /api/v/:command` 直接反射调用向量库类方法 | `server/index.js:145-171` |
| 404 兜底 | `app.all("*")` 返回 404 | `server/index.js:174-176` |
| HTTP/HTTPS 启动与启动钩子 | `bootHTTP`/`bootSSL` 监听端口，启动后跑迁移、遥测、后台服务等 | `server/utils/boot/index.js:21-83` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `app`（Express 实例） | `server/index.js:50` | 全局应用对象，挂全局中间件、静态目录、404 |
| `apiRouter`（express.Router） | `server/index.js:51` | 业务路由挂载点，统一前缀 `/api` |
| `systemEndpoints(apiRouter)` 等 | `server/endpoints/system.js:80` | 各端点模块工厂函数，接收 router 并注册具体路由 |
| `developerEndpoints(app, router)` | `server/endpoints/api/index.js:14` | 唯一双参端点工厂，额外接收 app 以启用 Swagger |
| `bootHTTP(app, port)` / `bootSSL(app, port)` | `server/utils/boot/index.js:64,21` | 启动服务并执行启动后异步钩子 |
| `httpLogger({enableTimestamps})` | `server/middleware/httpLogger.js:2` | 开发态请求日志中间件工厂 |
| `catchSigTerms()` | `server/utils/boot/index.js:85` | 注册 SIGINT/SIGUSR2 退出前 flush 遥测 |

## 3. 关键调用链

**链 A：服务启动与路由装配（`server/index.js`）**
1. 进程入口先加载 dotenv、logger、SDK 超时补丁、模型定价缓存（`index.js:1-7`）。
2. 创建 `app` 与 `apiRouter`，按顺序挂 cors → bodyParser.text/json/urlencoded（`index.js:65-73`）。
3. 若 `ENABLE_HTTPS` 走 `bootSSL`，否则加载 express-ws 后由末尾 `bootHTTP` 启动（`index.js:75-79,180`）。
4. `app.use("/api", apiRouter)` 后，依次调用 20+ 端点工厂完成路由注册（`index.js:81-112`）。
5. 非开发态挂静态目录与 SPA 兜底；最后 `app.all("*")` 返回 404（`index.js:114-176`）。

**链 B：监听成功后的启动钩子（`utils/boot/index.js:67-79`）**
1. `app.listen(port, async () => {...})` 回调内按序 `migrateWebBrowsingToDefault()` → `markOnboarded()` → `setupTelemetry()` → `new CommunicationKey(true)` → `new EncryptionManager()` → `new BackgroundService().boot()` → `eagerLoadContextWindows()` → `PushNotifications.setupPushNotificationService()` → `TelegramBotService.bootIfActive()`。
2. `bootSSL` 证书读取失败时 `catch` 后自动回退 `bootHTTP`（`index.js:50-61`）。

**链 C：单次请求穿过中间件链**
1. 请求先经 cors、bodyParser 解析（`index.js:65-73`），再进入 `/api` 子路由。
2. 命中某端点模块注册的路由后，由该路由自带的守卫中间件（如 `validatedRequest`、`validApiKey`）鉴权，通过后才进入业务处理函数（见各叶子）。

![请求路由装配架构图](request-routing-architecture.html)
![单次 API 请求生命周期时序](request-routing-sequence.html)

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `SERVER_PORT` | 缺省 `3001` | `server/index.js:76,180` |
| `ENABLE_HTTPS` | 未设置走 HTTP；设置后走 bootSSL | `server/index.js:75` |
| `HTTPS_KEY_PATH` / `HTTPS_CERT_PATH` | bootSSL 读取证书与私钥的路径 | `server/utils/boot/index.js:28-29` |
| `NODE_ENV=development` | 才启用静态/SPA 兜底的反向（即开发态不托管 public），并开放 `/v/:command` 调试路由 | `server/index.js:114,145` |
| `ENABLE_HTTP_LOGGER` | 仅开发态生效，开启请求日志 | `server/index.js:56-57` |
| `ENABLE_HTTP_LOGGER_TIMESTAMPS` | 日志是否带时间戳 | `server/index.js:61` |
| bodyParser `limit` | `3GB`（`FILE_LIMIT`） | `server/index.js:52,66-72` |

## 5. 错误与重试语义

- `bootSSL` 读证书/建 HTTPS server 抛错时，记录结构化日志并**自动回退 `bootHTTP`**（`utils/boot/index.js:50-61`），不崩溃进程。
- `app.listen(...).on("error", catchSigTerms)`：监听错误由 Node 原生 error 事件处理；`catchSigTerms` 在 SIGINT/SIGUSR2 时先 `Telemetry.flush()` 再转发信号自杀（`utils/boot/index.js:85-94`）。
- 404 兜底对所有未匹配路径统一 `sendStatus(404)`，不暴露内部错误。
- 本叶子不含业务重试逻辑；端点内的错误处理见各业务叶子。

## 6. 并发细节

- Node 单事件循环：所有 Express 中间件与路由回调在同一事件循环上串行调度；异步 I/O（DB、外部 API）以 Promise/回调让出事件循环。
- `bootHTTP`/`bootSSL` 的 listen 回调内是**顺序 await** 的启动钩子链（迁移→遥测→后台服务→推送→Telegram），任一前置 await 未完成不会并行启动后续。
- `BackgroundService().boot()`、`PushNotifications`、`TelegramBotService` 在监听成功后异步拉起，与请求处理并发运行（后台任务细节见 jobs-memory-models 与 agents-mcp 域）。
- WebSocket 通过 express-ws 挂在同一 HTTP server 上，长连接与短连接请求共用事件循环。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- Express app/apiRouter 装配、全局中间件、静态托管、404、boot 启动与启动钩子、httpLogger。

**Out-of-Scope（不在本仓库源码内）**
- 各端点业务处理逻辑（属 auth-authz-api、workspace-chat-api 等叶子）。
- 前端 SPA 构建产物（`server/public` 由 frontend 构建拷贝而来，非本叶子源码）。
- `@mintplex-labs/express-ws`、`cors`、`body-parser` 等 npm 依赖（不在本仓库源码内）。
- 反向代理/容器编排/进程管理（systemd、Docker、云平台）不在本仓库源码内。

## 8. 与相邻子系统交互

- 本叶子 → 各 API 叶子：把 `apiRouter` 传给 20+ 端点工厂，由后者注册具体路由与守卫。
- 本叶子 → server-data：启动钩子中 `markOnboarded()`、迁移类操作经 Prisma/模型层读写 SQLite（见 server-data 域）。
- 本叶子 → agents-mcp：`BackgroundService`、`TelegramBotService`、`PushNotifications` 由 boot 拉起（见 agents-mcp 域）。
- 上游：前端 SPA / 开发者 API 客户端经 HTTP/WebSocket 访问。

## 9. 语言专项适配口径（TS-JS 口径，纯 JS 并入）

- **capability seam 分组**：本叶子是典型的"装配/定义层"（Service Definition），本身不含业务 Provider/Consumer；各端点工厂是 Provider，前端与 API 客户端是 Consumer。
- **图型**：以 architecture（静态装配拓扑）+ sequence（单次请求穿过中间件链的时序）为主；本叶子无数据管道语义、无单一实体状态机，故不补 dataflow/lifecycle。
- **外部边界**：图中标 `external` 的 API 客户端、Node 内置 `http`/`https`、npm 依赖 `express`/`cors`/`express-ws`、前端静态资源均为外部边界；SQLite、后台服务在图中以 backend 节点表示并在正文说明。
- **事件驱动**：WebSocket（express-ws）与 `res.end` 劫持（httpLogger）是 JS 事件驱动热点，已在 sequence 与正文体现。
- **部署维度**：server 为独立 npm 包 `anything-llm-server`（见 server/package.json），非"单二进制"；前端构建产物拷贝进 `server/public` 后由本进程静态托管。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 装配架构图 | `request-routing-architecture.html` | architecture | showcase |
| 请求生命周期时序 | `request-routing-sequence.html` | sequence | showcase |

- 未生成 dataflow：本叶子无"数据源→处理→目的地"的数据管道语义；未生成 lifecycle：无单一实体状态机；未生成 workflow：无带泳道的多角色审批/发布流程。按资源节省原则省略。
- JSON IR 源文件位于 `json/` 目录。
