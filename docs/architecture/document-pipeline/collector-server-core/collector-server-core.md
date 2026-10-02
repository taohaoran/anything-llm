# Collector 独立服务核心（collector-server-core）

> 本文是 `document-pipeline` 域下的叶子子系统文档。域级总览见 `../document-pipeline.md`。
> 本文只展开 collector 独立 Express 服务的入口、路由与任务接收编排；具体格式转换见 file-conversion、链接抓取见 link-scraping 等叶子。
>
> 源码基准：anything-llm v1.16.2，commit `128a0157`，纯 JavaScript ESM。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| Express 服务入口 | 独立进程，监听 `COLLECTOR_PORT`，启动时 `wipeCollectorStorage()` 清空临时目录 | `collector/index.js:20,216-219` |
| 中间件 | CORS、body-parser（3GB 上限）、`verifyPayloadIntegrity` 签名校验、可选 httpLogger | `collector/index.js:35-43,47` |
| `/process` 路由 | 接收已上传文件，调 `processSingleFile` 解析并返回 documents | `collector/index.js:45-73` |
| `/parse` 路由 | 仅解析不落库（parseOnly），支持 absolutePath 内部调用 | `collector/index.js:75-107` |
| `/process-link` 路由 | 接收 URL，调 `processLink` 抓取 | `collector/index.js:109-132` |
| `/util/get-link` 路由 | 仅抓链接文本不落库（agent 工具用） | `collector/index.js:134-152` |
| `/util/convert-audio-to-wav` 路由 | 音频转 16kHz 单声道 WAV | `collector/index.js:154-177` |
| `/process-raw-text` 路由 | 纯文本入库 | `collector/index.js:179-204` |
| `/accepts` 路由 | 返回支持的 MIME 列表 | `collector/index.js:208-210` |
| 扩展路由挂载 | `extensions(app)` 挂载 resync/repo/youtube 等扩展端点 | `collector/index.js:206` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| Express `app` | `collector/index.js:20` | 服务实例 |
| `getCollectorPort()` | `collector/utils/http.js` | 读取端口环境变量 |
| `verifyPayloadIntegrity` | `collector/middleware/verifyIntegrity.js` | 校验 server→collector 请求签名 |
| `reqBody(request)` | `collector/utils/http.js` | 统一解析 text/json body |
| `processSingleFile/processLink/processRawText/convertAudioToWav` | 各子目录 `index.js` | 四类处理任务的编排入口 |

## 3. 关键调用链

1. **文件上传处理**：server 端 `CollectorApi` HTTP POST `/process` → collector `verifyPayloadIntegrity`（`index.js:47`）→ `reqBody` 取 `filename/options/metadata`（`:49`）→ 路径归一化防 `..` 穿越（`:51-53`）→ `processSingleFile(targetFilename, options, metadata)`（`:58`）→ 返回 `{success, reason, documents}`。
2. **启动清理**：`app.listen` 回调里 `await wipeCollectorStorage()`（`:218`）清空 hotdir 临时文件。
3. **错误兜底**：每个路由 try/catch 失败仍返回 200 + `{success:false, reason, documents:[]}`（如 `:62-69`），由 server 端判断 success 字段。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `COLLECTOR_PORT` | 监听端口 | `index.js:22` `getCollectorPort()` |
| 文件体上限 | `3GB` | `index.js:21` |
| `WATCH_DIRECTORY` | hotdir 上传目录 | `utils/constants.js` |
| `ENABLE_HTTP_LOGGER` | 仅 development 开启 | `index.js:27` |

## 5. 错误与重试语义

- 路径穿越防护：`path.normalize(filename).replace(/^(\.\.(\/|\\|$))+/, "")`（`index.js:51-53`）；`processSingleFile` 再用 `isWithin` 校验（`processSingleFile/index.js:31-39`）。
- 路由层 catch 所有异常，返回 200 + success:false，不让 Express 默认错误页暴露堆栈。
- 服务 listen error 时注册 SIGUSR2/SIGINT 退出处理（`index.js:221-227`）。

## 6. 并发细节

- Node Express 单进程异步；请求并发由事件循环处理。
- 无 worker 池；单文件处理在 async 函数内 await。
- `extensions(app)` 在 listen 前同步挂载路由。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `collector/index.js`、`collector/middleware/`、`collector/utils/`

**Out-of-Scope（不在本仓库源码内）**
- server 主服务（另一个进程，通过 HTTP 调用 collector）
- `express`/`cors`/`body-parser` npm 包
- server 端 `CollectorApi` 封装（在 server/utils/collectorApi.js，属 server-api 域）

## 8. 与相邻子系统交互

- 上游 → 本叶子：server 主进程通过 `CollectorApi` HTTP 调用 collector 各端点（独立进程边界）。
- 本叶子 → 下游：`processSingleFile` → file-conversion 各转换器；`processLink` → link-scraping；`processRawText` → raw-text-audio；`extensions(app)` → collector-extensions-hotdir。
- 本叶子产出 documents JSON 回传 server，由 server 端向量化入库。

## 9. 语言专项适配口径（TS/JS 口径）

- **capability seam 分组**：collector 是独立 Node Express 服务进程（与 server 同仓库不同 package.json，name=anything-llm-document-collector）。
- **图类型侧重**：architecture（Express 路由+中间件+下游处理器拓扑）+ sequence（/process 请求时序）。
- **外部边界**：server 主进程（HTTP 调用方）、Node 内置 fs/path。
- **部署维度**：collector 为独立部署单元（可独立容器/进程），与 server 通过 HTTP + 文件系统交互。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| Collector 服务架构 | `collector-server-core-architecture.html` | architecture | showcase（初版 mw→app 竖边方向不合规，删去该隐式边后通过） |
| /process 请求时序 | `collector-server-core-sequence.html` | sequence | showcase（一次通过） |

JSON IR 源文件位于 `json/` 目录。
