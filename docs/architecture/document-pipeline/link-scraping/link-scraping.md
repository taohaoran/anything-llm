# 网页链接抓取（link-scraping）

> 本文是 `document-pipeline` 域下的叶子子系统文档。域级总览见 `../document-pipeline.md`。
> 本文只展开"网页 URL 抓取与 HTML→Markdown/纯文本提取"。
>
> 源码基准：anything-llm v1.16.2，commit `128a0157`，纯 JavaScript ESM。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 链接处理入口 | `processLink(link, scraperHeaders, metadata)` 校验 URL 后抓并存为文档 | `collector/processLink/index.js:15` |
| 仅取文本不落库 | `getLinkText(link, captureAs)` 供 agent 工具调用 | `collector/processLink/index.js:35` |
| 通用抓取 | `scrapeGenericUrl({link, captureAs, scraperHeaders, metadata, saveAsDocument})` | `collector/processLink/convert/generic.js` |
| HTML→Markdown | `htmlToMarkdown` 辅助 | `collector/processLink/helpers/htmlToMarkdown.js` |
| URL 校验 | `validateURL`/`validURL` | `collector/utils/url.js` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `processLink()` | `processLink/index.js:15` | 存文档模式抓链接 |
| `getLinkText()` | `processLink/index.js:35` | 不存文档，仅返回内容（agent 用） |
| `scrapeGenericUrl()` | `processLink/convert/generic.js` | 通用抓取：HTTP 拉取→HTML 清洗→按 captureAs 输出 |
| `validateURL`/`validURL` | `utils/url.js` | URL 合法性与协议校验 |

## 3. 关键调用链

1. **链接入库**：`/process-link` 路由（`collector/index.js:109`）→ `processLink(link, headers, metadata)`（`processLink/index.js:15`）→ `validateURL` + `validURL` 校验（`:17-18`）→ `scrapeGenericUrl({saveAsDocument:true})` → 抓 HTML→清洗→`writeToServerDocuments`。
2. **agent 取链接文本**：`/util/get-link`（`collector/index.js:134`）→ `getLinkText(link, captureAs)`（`processLink/index.js:35`）→ `scrapeGenericUrl({saveAsDocument:false})` 直接返回 content。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `captureAs` | `text`/`html`/`json`，决定输出格式 | `processLink/index.js:35` |
| `scraperHeaders` | 自定义请求头（反爬/鉴权） | `/process-link` 入参 |

## 5. 错误与重试语义

- URL 非法 → `{success:false, reason:"Not a valid URL."}`（`processLink/index.js:18-19,38-39`）。
- 抓取异常由路由 catch 兜底返回 success:false。
- 本层不做重试；外部网站不可达直接失败。

## 6. 并发细节

- 单次 HTTP 抓取 async；无并发池。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `collector/processLink/`、`collector/utils/url.js`

**Out-of-Scope（不在本仓库源码内）**
- 目标外部网站（被抓取对象）
- `cheerio`/`node-fetch`/`puppeteer` 等抓取 npm 包

## 8. 与相邻子系统交互

- 上游 → 本叶子：collector-server-core 的 `/process-link`、`/util/get-link`。
- 本叶子 → 下游：产出纯文本 → 写 server documents → 向量化。

## 9. 语言专项适配口径（TS/JS 口径）

- **图类型侧重**：dataflow（URL→HTTP 拉取→HTML 清洗→Markdown 管道）+ sequence。
- **外部边界**：外部网站、抓取 npm 包。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 链接抓取数据流 | `link-scraping-dataflow.html` | dataflow | showcase（一次通过） |
| 链接抓取时序 | `link-scraping-sequence.html` | sequence | showcase（初版自消息 validateURL 跨度 0px 失败，删去自消息后通过） |

JSON IR 源文件位于 `json/` 目录。
